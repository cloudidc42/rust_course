# Part 25: Iterators เบื้องต้น (Iterator trait, for loop desugaring)

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 220 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายนิยามจริงของ trait `Iterator` (`type Item;` และ `fn next(&mut self) -> Option<Self::Item>;`) ได้อย่าง
  ถูกต้อง และเชื่อมโยงกับแนวคิด associated type จาก Part 22 ได้ว่าทำไม `Iterator` ต้องออกแบบด้วย associated type
  ไม่ใช่ generic type parameter บน trait — นี่คือตัวอย่างที่ "เป็นธรรมชาติที่สุด" ของหลักการนั้นในทั้ง standard
  library
- อธิบายได้ว่าทำไม `next()` ต้องคืนค่าเป็น `Option<Self::Item>` โดยเชื่อมโยงตรงกับปรัชญาการออกแบบของ `Option<T>`
  จาก Part 11 — `None` แทนความหมาย "ไม่มีสมาชิกเหลือแล้ว" ได้อย่างปลอดภัยโดยไม่ต้องพึ่งค่า sentinel (เช่น `-1`)
  หรือ exception แบบภาษาอื่น
- เขียน**การ desugar ของ `for` loop ด้วยมือ** (`while let Some(x) = iter.next() { ... }` ที่ทำงานบน
  `collection.into_iter()`) และอธิบายบทบาทของ trait `IntoIterator` ที่ทำให้ `for x in collection` เขียนได้สั้น
  โดยไม่ต้องเรียก `.into_iter()` เองตรง ๆ — นี่คือการไขปริศนาของ `for` loop ที่ใช้มาตั้งแต่ Part 4 ให้กระจ่างครบ
  100%
- แยกแยะพฤติกรรมของ `.iter()`, `.iter_mut()`, และ `.into_iter()` ได้อย่างแม่นยำในเชิง ownership (คืน `&T`,
  `&mut T`, และ `T` ตามลำดับ) และอธิบายได้ว่า `for x in &v`, `for x in &mut v`, `for x in v` แต่ละแบบเรียก
  `IntoIterator` implementation คนละตัวกัน — ปิดหัวข้อที่ Part 13 ทีเซอร์ไว้ให้สมบูรณ์เต็มรูปแบบ
- เขียน **custom iterator ของตัวเองตั้งแต่ต้น** ด้วยการ implement `Iterator` (เขียนแค่ `next()`) ให้กับ struct
  ที่ออกแบบเอง แล้วพิสูจน์ได้ว่าทำไมการเขียนแค่ method เดียวนี้ทำให้ได้ method อื่นอีกกว่า 70 ตัว (`.sum()`,
  `.count()`, `.max()`, `.collect()`, ฯลฯ) มา "ฟรี" โดยไม่ต้องเขียนเพิ่มเลยแม้แต่บรรทัดเดียว
- แยกแยะ **consuming adaptor** (เช่น `.sum()`, `.count()`, `.collect()`, `.max()`, `.fold()`) กับ **iterator
  adaptor** (เช่น `.map()`, `.filter()`, `.zip()`, `.enumerate()`, `.take()`, `.skip()`, `.rev()`, `.chain()`)
  ได้อย่างชัดเจน และพิสูจน์ด้วยโค้ดจริงว่า iterator adaptor **lazy (เฉื่อยชา)** — ไม่ทำงานอะไรเลยจนกว่าจะมี
  consuming adaptor หรือ `for` loop มา "ขับเคลื่อน" มัน
- ใช้ `.collect()` ได้อย่างถูกต้องในสถานการณ์ที่ต้องรวบรวมเป็น `Vec<T>`, `String`, `HashMap<K, V>` (เชื่อม
  Part 15), และ `HashSet<T>` พร้อมอ่าน/แก้ error "type annotations needed" ตอน `.collect()` กำกวมได้ และนำทุก
  แนวคิดในบทนี้มาผสมกันเขียนโปรแกรมประมวลผล `Vec<Product>` (สไตล์เดียวกับ Part 9/13) ด้วย iterator chain จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 6 (Ownership เบื้องต้น)** และ **Part 7 (Borrowing และ References)**: หัวใจของทั้งบทนี้คือการนำกฎ
  ownership/borrowing แบบเดียวกันที่เรียนมาตั้งแต่สองบทนี้ มาอธิบายว่าทำไม `.iter()` ยืมข้อมูล (`&T`),
  `.iter_mut()` ยืมแบบแก้ไขได้ (`&mut T`), และ `.into_iter()` ยึด ownership (`T` ตรง ๆ ผ่านการ move) — ถ้ายัง
  ไม่แน่นเรื่อง move/borrow จากสองบทนี้ เนื้อหาหัวข้อ 25.6-25.7 จะเข้าใจยากมาก
- **Part 9 (Structs)**: struct `Product` ที่ใช้ในตัวอย่างท้ายบทเป็น struct สไตล์เดียวกับที่เรียนใน Part 9
  ทุกประการ (field + `impl` block + method)
- **Part 11 (Option<T> และ Null Safety)**: หัวข้อ 25.3 ผูกตรงกับปรัชญาการออกแบบของ `Option<T>` ที่เรียนมาแล้ว
  — `next()` คืน `Option<Self::Item>` ด้วยเหตุผลเดียวกันเป๊ะกับที่ `.pop()` ของ `Vec<T>` คืน `Option<T>` ใน
  Part 13 ถ้ายังไม่แน่นเรื่อง `Some`/`None`/`match`/`unwrap_or()` ควรย้อนไปทวนก่อน
- **Part 13 (Collections: Vec<T>)**: บทนี้ **ต่อยอดตรงจากหัวข้อ 13.6** ที่ทีเซอร์การวน `for n in &numbers`,
  `for n in &mut mutable_numbers`, `for s in owned_numbers` ไว้ พร้อมบอกตรง ๆ ว่ากลไกจริง (`Iterator`,
  `IntoIterator`) จะเจาะลึกใน Part 25-26 — **นี่คือ Part นั้น** เราจะอธิบายทุกอย่างที่ Part 13 ทีเซอร์ไว้ให้
  กระจ่างครบถ้วน รวมถึง error `E0382` ("moved due to this implicit call to `.into_iter()`") ที่ Part 13
  หัวข้อ 13.6 แสดงไว้แต่ยังไม่ได้อธิบายว่า `.into_iter()` คืออะไรจริง ๆ
- **Part 15 (HashMap, HashSet)**: หัวข้อ 25.12 จะ `.collect()` เป็น `HashMap<K, V>` และ `HashSet<T>` — ต้องรู้จัก
  สองชนิดข้อมูลนี้มาก่อนแล้วจาก Part 15
- **Part 19 (Traits เบื้องต้น)** และ **Part 22 (Generics ขั้นสูง)**: หัวข้อ 22.6 ของ Part 22 อธิบายไว้แล้วว่า
  ทำไม `Iterator` เลือกใช้ **associated type** (`type Item;`) ไม่ใช่ generic parameter บน trait
  (`trait Container<Item>`) พร้อมโชว์นิยามคร่าว ๆ ของ `Iterator` ไว้ล่วงหน้าและบอกว่า "จะเรียนเต็มรูปแบบใน
  Part 25-26" — บทนี้จะนำนิยามนั้นมาอธิบายทุกส่วนอย่างละเอียด ถ้ายังไม่ผ่าน Part 22 หัวข้อ 22.6 แนะนำให้ย้อนไป
  อ่านก่อน เพราะบทนี้จะไม่สอนแนวคิด "associated type คืออะไร" ซ้ำอีกรอบ (แต่จะอธิบายว่าทำไม `Iterator`
  **ต้อง**ใช้มันในเชิงเหตุผลเจาะลึกกว่าที่ Part 22 มีเวลาอธิบาย)
- **Part 24 (Closures)**: iterator adaptor แทบทุกตัวที่จะเจอในบทนี้ (`.map()`, `.filter()`, `.fold()`) รับ
  closure เป็น parameter ทั้งหมด และใช้ `Fn`/`FnMut` bound ตามหลักการเดียวกับที่ Part 24 สอนไว้ — ถ้ายังไม่คุ้น
  กับ syntax `|x| expr` และความแตกต่างของ `Fn`/`FnMut`/`FnOnce` ควรทวน Part 24 ก่อน แม้บทนี้จะไม่ลงรายละเอียด
  เรื่อง trait bound ของ closure ลึกเท่า Part 26 ก็ตาม

ถ้าคุณตามหลักสูตรมาถึงจุดนี้ คุณได้ใช้ `.iter()`, `for x in collection`, `.enumerate()`, `.chars()`, `.sum()`,
และ `.map()`/`.and_then()` (บน `Option<T>`) มาแล้วนับสิบครั้งตลอด 24 บทที่ผ่านมา โดยที่ทุกครั้งมีข้อความ
"เราจะเรียนเรื่องนี้อย่างเป็นทางการใน Part 25-26" แปะไว้เสมอ — บทนี้คือจุดที่ทุกอย่างที่ค้างไว้จะถูกอธิบายให้
กระจ่างครบถ้วน ไม่มีอะไรเป็น "มายากล" อีกต่อไป

## เนื้อหา

### 25.1 ทวนสิ่งที่ใช้มาโดยไม่รู้กลไก: รายการทีเซอร์ที่บทนี้จะไขให้หมด

ก่อนเริ่มเนื้อหาใหม่ ลองไล่ดูว่าตลอด 24 บทที่ผ่านมาเราใช้แนวคิด "iterator" แบบไม่เป็นทางการไปมากแค่ไหนแล้ว:

| ที่มา | สิ่งที่ใช้ | สถานะตอนนั้น |
|---|---|---|
| Part 4 (4.8) | `for i in 0..5`, `for score in scores` | "แปลง `scores` ให้กลายเป็น iterator อัตโนมัติผ่าน `IntoIterator`" — ทีเซอร์ไว้เฉย ๆ |
| Part 6-7 | `for x in v`, `for x in &v` | ยังไม่อธิบายว่าทำไม ownership ต่างกัน |
| Part 8 | `.iter().enumerate()` บน slice | "จะเรียนละเอียดใน Part 25-26" |
| Part 9 | เก็บ closure/สร้างรายการใหม่จาก loop | เกริ่น concept "into_iter" สั้น ๆ |
| Part 10 | pattern ที่คล้าย `.any()`/`.all()` | บอกว่าเป็น iterator method ที่จะเรียนทีหลัง |
| Part 11 | `.map()`, `.and_then()`, `.filter()` บน `Option<T>` | สอนเฉพาะบน `Option`, ยังไม่บอกว่าเป็นชื่อเดียวกับ iterator method |
| Part 13 (13.6) | `for n in &numbers`, `&mut numbers`, `numbers` | ทีเซอร์ตรง ๆ ว่า "`Iterator`, `IntoIterator` จะเจาะลึกใน Part 25-26" |
| Part 14 | `.chars()`, `.collect()` เป็น `Vec<&str>` | "จะเจาะลึกใน Part 25-26" |
| Part 18 | type parameter ของ method แยกจาก struct | อ้างถึงหลักการเดียวกับที่ iterator ใช้ |
| Part 22 (22.6) | นิยามคร่าว ๆ ของ `trait Iterator` พร้อม `type Item` | บอกตรง ๆ ว่า "จะเรียนเต็มรูปแบบใน Part 25-26" |
| Part 23 | `Fn(&Item) -> bool` แบบที่ `Iterator::filter` ใช้ | อ้างถึงล่วงหน้าเรื่อง lifetime ของ closure |
| Part 24 | ปิดท้ายบทด้วยการประกาศว่า Part 25 จะเจาะลึก `Iterator` | สัญญาไว้ตรง ๆ |

นี่ไม่ใช่เรื่องบังเอิญ — **`Iterator` เป็นหนึ่งใน trait ที่สำคัญที่สุดในภาษา Rust ทั้งหมด** มันคือกลไกที่ทำให้
`for` loop ทำงานได้, ทำให้ `.collect()`, `.sum()`, `.map()` มีอยู่จริง, และเป็นตัวอย่างที่ "สอนตัวเอง" ได้ดีที่สุด
ว่า trait system ของ Rust (ที่เรียนมาตั้งแต่ Part 19-22) ถูกออกแบบมาเพื่อรองรับ pattern แบบนี้โดยเฉพาะ — เพราะ
เหตุนี้หลักสูตรจึงจงใจ**เลื่อน**การอธิบายกลไกจริงมาไว้จนกว่าผู้เรียนจะมีพื้นฐานที่จำเป็นครบ (ownership, trait,
generics, associated type, closure) — ครบทุกอย่างแล้ว ณ จุดนี้ เราพร้อมเจาะลึกอย่างเต็มรูปแบบ

### 25.2 นิยามจริงของ trait `Iterator`: `type Item` และ `fn next(&mut self) -> Option<Self::Item>`

Part 22 หัวข้อ 22.6 เคยโชว์นิยามคร่าว ๆ ของ `Iterator` ไว้ล่วงหน้าเพื่ออธิบายเรื่อง associated type — ตอนนี้เรามา
ดูให้เต็มขึ้นอีกหน่อย (ยังไม่ครบทั้งหมด เพราะ `Iterator` มี default method มากกว่า 70 ตัว แต่นี่คือ**ส่วนที่ต้อง
ประกาศเองทั้งหมด**):

```
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // ... default method อีกกว่า 70 ตัว (sum, count, max, map, filter, ...)
    // เราจะพิสูจน์ว่ามันมาจากไหนในหัวข้อ 25.9
}
```

(นี่คือ pseudocode อธิบายโครงสร้าง ไม่ใช่โค้ดที่ compile ได้ตรง ๆ เพราะ `...` ไม่ใช่ syntax จริง — นิยามจริงจาก
`std::iter::Iterator` มีความซับซ้อนกว่านี้เล็กน้อยในรายละเอียดของแต่ละ default method แต่โครงสร้างหลักตรงกับที่
เห็นนี้ทุกประการ)

**อ่านทีละส่วน โดยเชื่อมกับสิ่งที่เรียนมาแล้ว:**

- **`type Item;`** — คือ **associated type** ตามที่เรียนใน Part 22 หัวข้อ 22.6 — บอกว่า "ทุก type ที่จะ
  implement `Iterator` ต้องเลือกชนิดข้อมูลหนึ่งชนิดมาผูกกับชื่อ `Item` — คือชนิดของค่าที่ iterator ตัวนี้จะ
  ให้ออกมาทีละตัว" เมื่อเรา implement `Iterator` ให้ `Countdown` (จะเห็นในหัวข้อ 25.8) เราจะเขียน
  `type Item = u32;` แปลว่า "สำหรับ `Countdown` โดยเฉพาะ ค่าที่วนออกมาแต่ละตัวเป็น `u32` เสมอ ไม่มีทางเป็น
  อย่างอื่น"
- **`fn next(&mut self) -> Option<Self::Item>;`** — คือ**method เดียวที่จำเป็นต้องเขียนเอง** (required method
  — ไม่มี default implementation ให้) รับ `&mut self` (ต้อง**ยืมแบบแก้ไขได้**ตัว iterator เอง เพราะการเรียก
  `next()` แต่ละครั้งต้อง "เดินหน้า" สถานะภายในของ iterator ไปทีละขั้น — เราจะเห็นเหตุผลนี้ชัดเจนตอนเขียน
  `Countdown` เอง) และคืนค่าเป็น `Option<Self::Item>` เสมอ

**คำถามสำคัญที่ Part 22 หัวข้อ 22.6 ตอบไว้แล้วบางส่วน แต่ควรทวนให้แน่นตรงนี้**: ทำไม `Iterator` ต้องใช้
associated type ไม่ใช่ `trait Iterator<Item> { fn next(&mut self) -> Option<Item>; }` แบบ generic parameter?
เหตุผลคือ **หนึ่ง type ควรเป็น "iterator ของชนิดข้อมูลเดียว" ได้แค่ทางเดียวเท่านั้น** — ถ้า `Countdown`
implement `Iterator<u32>` และ `Iterator<String>` พร้อมกันได้ (แบบที่ generic parameter อนุญาต) คำถาม "ตัวต่อไป
ของ `Countdown` คืออะไร" จะกำกวมทันทีเมื่อเขียน `for x in countdown` เพราะไม่มี context บอกว่าต้องการ `u32`
หรือ `String` associated type บังคับกฎที่เราต้องการอยู่แล้วโดยธรรมชาติ ("iterator ตัวหนึ่งให้ค่าชนิดเดียวเสมอ")
ให้กลายเป็นสิ่งที่ **compiler ตรวจสอบให้อัตโนมัติตั้งแต่ compile time** — ตรงกับที่ Part 22 พิสูจน์ด้วย error
`E0119` ไปแล้ว (implement `Iterator` ให้ type เดียวกันสองครั้งด้วย `Item` คนละชนิดจะถูกปฏิเสธทันที)

**อีก method หนึ่งที่ควรรู้จักชื่อไว้ (ไม่ต้องเขียนเองก็ได้)**: `Iterator` ยังมี default method ชื่อ
`size_hint(&self) -> (usize, Option<usize>)` ที่คืน "การประมาณ" จำนวนสมาชิกที่เหลือ (ขอบล่างที่แน่นอน, ขอบบน
ถ้ารู้) — ค่า default ของมันคือ `(0, None)` (แปลว่า "ไม่รู้เลย ประมาณได้แค่ว่าอย่างน้อย 0 ตัว") แต่ iterator ที่
รู้ความยาวแน่นอนอยู่แล้ว เช่น `std::slice::Iter` (เพราะมี length เก็บไว้ตามโครงสร้างจาก Part 13) จะ override
`size_hint()` ให้คืนค่าที่แม่นยำ ประโยชน์ของมันคือ **`.collect()` ใช้ `size_hint()` เพื่อเรียก
`Vec::with_capacity()` ล่วงหน้า** (concept เดียวกับที่ Part 13 หัวข้อ 13.2 สอนไว้ว่าการจอง capacity ล่วงหน้า
ช่วยประสิทธิภาพ) ทำให้ `.collect()` เป็น `Vec<T>` จาก iterator ที่รู้ความยาวแน่นอน **ไม่ต้อง reallocate เลย
แม้แต่ครั้งเดียว** ต่างจาก iterator ที่ไม่รู้ความยาว (เช่น `.filter()` ที่ไม่รู้ล่วงหน้าว่าจะเหลือกี่ตัวจนกว่า
จะเช็คทุกตัวจริง ๆ) ซึ่งยังต้องขยาย `Vec` ไปตามธรรมชาติ นี่คืออีกตัวอย่างที่ default method ของ `Iterator`
เชื่อมโยงกับ performance ของ collection ที่เรียนมาแล้วโดยตรง ไม่ใช่แค่ความสะดวกทางไวยากรณ์เฉย ๆ

### 25.3 ทำไม `next()` ต้องคืน `Option<Self::Item>`: ผูกปรัชญาตรงกับ Part 11

นี่คือคำถามที่ควรถามให้ลึกกว่าแค่ "เพราะ std เขียนมาแบบนี้" — ทำไม**ต้อง**เป็น `Option<Self::Item>` ทำไมไม่
ออกแบบให้ `next()` คืน `Self::Item` ตรง ๆ แล้วมี method แยกอีกตัวชื่อ `has_next()` สำหรับเช็คว่าจะเรียกต่อได้
ไหม (แบบที่ Java's `Iterator<T>` ทำด้วย `hasNext()` + `next()` สองตัวแยกกัน)?

ลองนึกภาพว่า `next()` ต้องตอบคำถามหนึ่งคำถามเสมอทุกครั้งที่ถูกเรียก: **"มีสมาชิกตัวต่อไปให้ไหม?"** คำตอบมีแค่
สองแบบ: "มี และนี่คือค่านั้น" หรือ "ไม่มีแล้ว หมดแล้ว" — นี่คือรูปแบบเดียวกันเป๊ะกับที่ Part 11 สอนไว้ว่า
`Option<T>` ถูกออกแบบมาแทนความหมาย **"อาจมีค่า หรืออาจไม่มีค่า"** โดยไม่ต้องพึ่งค่า sentinel (เช่น `-1`,
`null`, ค่าว่างพิเศษ) หรือ exception

**เทียบ 3 แนวทางออกแบบที่เป็นไปได้:**

| แนวทาง | ตัวอย่างภาษา | ปัญหา |
|---|---|---|
| ใช้ sentinel value (เช่น `-1`) แทน "หมดแล้ว" | C แบบเก่า (`getchar()` คืน `EOF` ที่เป็น `-1`) | ถ้า `-1` บังเอิญเป็นค่าจริงที่ถูกต้อง (เช่น iterator ของ `i32` ที่มี `-1` อยู่จริงในข้อมูล) จะแยกไม่ออกระหว่าง "ค่าจริงคือ -1" กับ "หมดแล้ว" |
| แยก method `hasNext()`/`next()` สองตัว | Java `Iterator<T>` | ต้องเรียกสองครั้งเสมอเพื่อความปลอดภัย (`hasNext()` ก่อน `next()`) ลืมเรียก `hasNext()` ก่อนจะได้ `NoSuchElementException` ตอน runtime — compiler ไม่ช่วยเตือนอะไรเลย |
| throw exception เมื่อหมด | Python `StopIteration` (ภายใน `__next__`) | ใช้ control flow ผ่าน exception สำหรับสถานการณ์ปกติมาก (การหมด iterator ไม่ใช่ "ข้อผิดพลาด" แต่เป็นผลลัพธ์ปกติที่คาดหวังได้) ต้นทุนด้าน performance ของ exception สูงกว่า return value ธรรมดามาก |
| **คืน `Option<T>` ค่าเดียว** | **Rust** | ไม่มีปัญหาทั้งสามข้อข้างบนเลย — `Some`/`None` คือส่วนหนึ่งของ type system ที่ compiler บังคับให้จัดการทั้งสองกรณีเสมอ (เหมือนที่ `match` จาก Part 10 บังคับ exhaustiveness) และไม่มีต้นทุนของ exception เพราะเป็นแค่ enum ธรรมดาที่ compiler รู้ขนาดแน่นอนตั้งแต่ compile time |

การเลือกคืน `Option<Self::Item>` เพียงค่าเดียวยัง "รวมสองคำถามให้เป็นหนึ่ง" อย่างชาญฉลาด: ปกติเราต้องถามสอง
คำถาม ("มีสมาชิกต่อไปไหม" แล้ว "ถ้ามี ค่าคืออะไร") แต่ `next()` ตอบทั้งสองคำถามในการเรียกครั้งเดียว ไม่มีทาง
เรียก "ผิดจังหวะ" ได้เลย (ต่างจาก Java ที่เรียก `next()` โดยไม่เช็ค `hasNext()` ก่อนได้ ซึ่ง compiler ไม่ปฏิเสธ
แต่ runtime จะ throw exception) — Rust ทำให้ "การเช็คว่ามีสมาชิกต่อไปไหม" กับ "การดึงค่านั้นออกมา" เป็น
**operation เดียวที่แยกกันไม่ได้** ตรงกับหลักการของ Part 11 ที่ว่า `Option<T>` ทำให้ "ความล้มเหลว" (ในที่นี้คือ
"ไม่มีสมาชิกแล้ว") เป็นส่วนหนึ่งของ type ที่ต้องจัดการอย่างชัดเจน ไม่ใช่สิ่งที่แยกไปตรวจสอบเองต่างหาก

### 25.4 ทดลองเรียก `.next()` ด้วยมือ: มองเห็นกลไกเบื้องหลัง `for` ตรง ๆ

ก่อนจะพูดถึง `for` loop เรามาดูก่อนว่า iterator "ดิบ ๆ" ทำงานอย่างไรถ้าเราเรียก `.next()` เองทุกครั้งโดยไม่ผ่าน
`for` เลย:

```rust
fn main() {
    let v = vec![10, 20, 30];
    let mut iter = v.iter();

    println!("{:?}", iter.next());
    println!("{:?}", iter.next());
    println!("{:?}", iter.next());
    println!("{:?}", iter.next());
}
```

ผลลัพธ์:

```
Some(10)
Some(20)
Some(30)
None
```

**อธิบายทีละบรรทัด:**

- `v.iter()` สร้าง**ตัว iterator** ขึ้นมาตัวหนึ่ง (type จริงของมันคือ `std::slice::Iter<'_, i32>` — เป็น struct
  ที่ std เตรียมไว้ ซึ่งเก็บสถานะภายในไว้ว่า "ตอนนี้อยู่ตำแหน่งไหนของ `v` แล้ว") — สังเกตว่า `v.iter()` **ไม่ได้
  คืนค่าตัวเลขออกมาตรง ๆ** มันคืน**ตัว iterator** ที่ยังไม่ได้ให้ค่าอะไรเลยจนกว่าจะเรียก `.next()`
- `let mut iter` — สังเกตคำว่า **`mut` จำเป็นเสมอ** เพราะ `next(&mut self)` ต้องการยืมแบบแก้ไขได้ตัว `iter`
  เอง — ทุกครั้งที่เรียก `.next()` มันต้อง**แก้ไขสถานะภายใน** ของ iterator (เลื่อนตำแหน่งไปข้างหน้าหนึ่งช่อง)
  ถ้าไม่มี `mut` compiler จะปฏิเสธทันที (จะเห็น error จริงในหัวข้อกับดักที่ 1)
- เรียก `.next()` 4 ครั้ง: 3 ครั้งแรกได้ `Some(10)`, `Some(20)`, `Some(30)` ตามลำดับที่สมาชิกถูกเก็บใน `v` —
  ครั้งที่ 4 ได้ `None` เพราะสมาชิกหมดแล้ว **นี่คือหัวใจของทุกอย่างในบทนี้**: `for` loop, `.map()`, `.sum()`,
  ทุก method ของ iterator สุดท้ายแล้ว**ล้วนเรียก `.next()` ซ้ำ ๆ แบบนี้เบื้องหลังทั้งหมด** — ไม่มีกลไกอื่นเลย

ข้อสังเกตสำคัญ: ถ้าเรียก `.next()` ต่อไปอีกหลัง `None` ครั้งแรก (เช่นเรียกครั้งที่ 5, 6, ...) มาตรฐานของ
`Iterator` **ไม่บังคับ**ว่าต้องได้ `None` ต่อไปตลอด (เรียกว่า iterator ที่ "fused" คือรับประกันว่าจะได้ `None`
ตลอดไปหลังเจอ `None` ครั้งแรก เทียบกับ iterator ที่ไม่ fused ซึ่งในทางทฤษฎีอาจกลับมาให้ `Some` อีกได้ถ้าถูก
implement แบบพิเศษ) แต่ **iterator ทุกตัวใน std library** (`Vec::iter()`, `Range`, ฯลฯ) implement แบบ fused
เสมอ — ในทางปฏิบัติแทบไม่ต้องกังวลเรื่องนี้เลย

### 25.5 `for` loop desugars จริง ๆ เป็นอะไร: `while let Some(x) = iter.next()`

ทีนี้มาถึงคำตอบของคำถามที่ถูกทีเซอร์ไว้ตั้งแต่ Part 4: **`for` loop ที่เราใช้มาตลอด แท้จริงแล้วคือ syntax sugar
ของอะไร?**

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    println!("=== for loop ธรรมดา ===");
    for x in v1 {
        println!("for: {x}");
    }

    let v2 = vec![1, 2, 3];
    println!("=== desugar ด้วย while let ===");
    let mut iter = v2.into_iter();
    while let Some(x) = iter.next() {
        println!("desugar: {x}");
    }
}
```

ผลลัพธ์:

```
=== for loop ธรรมดา ===
for: 1
for: 2
for: 3
=== desugar ด้วย while let ===
desugar: 1
desugar: 2
desugar: 3
```

สังเกตว่าทั้งสองบล็อกให้ผลลัพธ์**เหมือนกันทุกประการ** เพราะ**มันคือโค้ดชิ้นเดียวกัน** — เพียงแค่บล็อกแรกเขียน
แบบย่อด้วย `for`, บล็อกที่สองเขียนแบบเต็มด้วยมือ นี่คือ desugaring rule ที่แท้จริงของ `for` ใน Rust (เขียนเป็น
pseudocode เพื่อให้เห็นภาพรวม):

```
for PATTERN in EXPR {
    BODY
}

// เทียบเท่ากับ:

{
    let mut iter = IntoIterator::into_iter(EXPR);
    loop {
        match iter.next() {
            Some(PATTERN) => { BODY }
            None => break,
        }
    }
}
```

(`while let Some(x) = iter.next() { BODY }` ที่เขียนไว้ในตัวอย่างข้างบนคือรูปแบบที่อ่านง่ายกว่าและให้ผลลัพธ์
เหมือนกันทุกประการกับ `loop { match ... }` ในนิยามจริง — `while let` เองก็เป็น syntax sugar ของ `loop { match }`
อีกชั้นหนึ่ง ซึ่ง Part 10 เคยแนะนำไว้แล้วตอนสอน `if let`/`while let`)

**อ่านทีละส่วน:**

1. `let mut iter = IntoIterator::into_iter(EXPR);` — สิ่งแรกที่เกิดขึ้นคือ EXPR (`v1` ในตัวอย่างเรา) ถูกส่งเข้า
   `IntoIterator::into_iter()` เพื่อ**แปลงเป็น iterator ก่อน** — นี่คือคำตอบของคำถามที่ Part 13 หัวข้อ 13.6
   ทีเซอร์ไว้ว่า "`for name in names` มีการเรียก `.into_iter()` ซ่อนอยู่เบื้องหลัง" ตอนนี้เราเห็นชัดแล้วว่า
   **มันไม่ใช่แค่ "เหมือนจะเรียก" แต่ compiler แปลงโค้ดให้เรียกจริง ๆ** ตั้งแต่ก่อนเข้า loop เลย (เราจะพูดถึง
   `IntoIterator` แบบเต็มในหัวข้อถัดไป)
2. `loop { match iter.next() { Some(PATTERN) => BODY, None => break } }` — วนซ้ำไม่มีที่สิ้นสุดจนกว่าจะเจอ
   `None` แล้ว `break` ออก — ทุกรอบเรียก `.next()` หนึ่งครั้ง ถ้าได้ `Some(value)` ก็ bind ค่าเข้ากับ pattern
   ที่เขียนไว้ (`x` ในตัวอย่างเรา) แล้วรัน `BODY`

**ทำไมเรื่องนี้สำคัญมาก**: มันแปลว่า **`for` ไม่ใช่ภาษาพิเศษที่แยกออกจาก type system** — มันเป็นแค่ทางลัด
ทางไวยากรณ์ (syntax sugar) บนกลไกเดียวกันกับที่คุณเขียน `.next()` เองได้ ทุกอย่างที่ `for` ทำได้ คุณก็เขียนด้วย
`while let` เองได้เหมือนกันทุกประการ (แค่ยาวกว่า) — เหมือนที่ closure เป็น syntax sugar บน struct + trait
(เรียนใน Part 24) และ `?` operator เป็น syntax sugar บน `match` (เรียนใน Part 12) นี่คือแนวคิดที่เกิดขึ้นซ้ำ ๆ
ในการออกแบบภาษา Rust: **ให้ syntax สั้น ๆ ที่ใช้บ่อยเป็นทางลัดของกลไกที่อธิบายได้เต็มด้วย trait ปกติ** ไม่มี
"เวทมนตร์" ที่ compiler แอบทำโดยไม่มีกฎรองรับอยู่ในระบบ trait เลย

### 25.6 trait `IntoIterator`: เหตุผลที่ `for x in collection` ใช้ได้ตรง ๆ

จากหัวข้อก่อนหน้าเราเห็นว่า `for` เรียก `IntoIterator::into_iter(EXPR)` ก่อนเสมอ — คำถามคือ `IntoIterator` คือ
อะไร? นิยามของมันคือ:

```
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;

    fn into_iter(self) -> Self::IntoIter;
}
```

**อ่านทีละส่วน:**

- **`type Item;`** — ชนิดของค่าที่จะได้ตอนวน (เหมือน `Iterator::Item`)
- **`type IntoIter: Iterator<Item = Self::Item>;`** — associated type อีกตัว บอกว่า "ตัว iterator ที่จะได้
  จากการแปลงคือ type อะไร" พร้อม bound ว่า **ต้อง implement `Iterator` ด้วย `Item` ตรงกัน** (สังเกต syntax
  `Iterator<Item = Self::Item>` — นี่คือการ bound บน associated type ที่ Part 22 หัวข้อ 22.3 สอนไว้แล้วว่า
  ต้องเขียนแบบนี้เมื่อ bound ไม่ใช่ type parameter เปล่า ๆ)
- **`fn into_iter(self) -> Self::IntoIter;`** — รับ `self` **โดยไม่มี `&`** (รับ ownership เต็ม ๆ — เชื่อม
  Part 6) แล้วคืนตัว iterator ที่พร้อมใช้งาน

พูดง่าย ๆ: **`IntoIterator` คือ "สัญญา" ว่า type นี้แปลงเป็น iterator ได้** ส่วน **`Iterator` คือ "สัญญา" ว่า
type นี้เป็น iterator อยู่แล้ว (เรียก `.next()` ได้ตรง ๆ)** — สอง trait นี้ **ไม่ใช่ตัวเดียวกัน** แม้ชื่อจะคล้าย
กันมาก และนี่คือจุดที่มือใหม่ Rust สับสนกันบ่อยที่สุด

**ข้อสังเกตสำคัญที่เชื่อมกับ Part 22 หัวข้อ 22.9 (Blanket implementation)**: standard library เขียน blanket
implementation ให้ **ทุก type ที่ implement `Iterator` ได้ `IntoIterator` มาโดยอัตโนมัติ** ด้วยหลักการง่าย ๆ ว่า
"iterator ก็แปลงเป็น iterator ของตัวเองได้เสมอ (แปลงเป็นตัวมันเอง)":

```
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;

    fn into_iter(self) -> I {
        self  // ตัวเองอยู่แล้ว ก็คืนตัวเองไปตรง ๆ
    }
}
```

นี่คือเหตุผลที่**เราไม่ต้อง implement `IntoIterator` เองทุกครั้ง**ที่ implement `Iterator` ให้ type ของเรา —
ถ้า type หนึ่งเป็น iterator อยู่แล้ว (implement `Iterator`) มันก็ "แปลงเป็น iterator" ได้ตรง ๆ อยู่แล้วโดย
อัตโนมัติ (แปลงเป็นตัวเอง) เราจะเห็นผลของกฎนี้ชัด ๆ ในหัวข้อ 25.8 ตอนเขียน `Countdown` — implement แค่
`Iterator` (ไม่แตะ `IntoIterator` เลย) แต่ `for n in Countdown::new(5)` ก็ใช้ได้ทันที

**ตารางสรุปความแตกต่างระหว่างสอง trait นี้:**

| ประเด็น | `Iterator` | `IntoIterator` |
|---|---|---|
| หน้าที่ | "ฉันเป็น iterator อยู่แล้ว เรียก `.next()` กับฉันได้ตรง ๆ" | "ฉันแปลงเป็น iterator ได้ (ผ่าน `.into_iter()`)" |
| method หลัก | `next(&mut self) -> Option<Self::Item>` | `into_iter(self) -> Self::IntoIter` |
| ใครใช้บ่อย ๆ | ตัวที่ `for` เรียก `.next()` ซ้ำ ๆ ข้างใน loop | ตัวที่ `for` เรียก **ครั้งเดียว** ก่อนเริ่ม loop เพื่อได้ `Iterator` มา |
| `Vec<T>` implement ไหม | **ไม่** (`Vec<T>` เองไม่มี `.next()`) | **มี** (3 แบบ — จะเห็นในหัวข้อ 25.7) |
| `std::slice::Iter<'_, T>` (ผลจาก `.iter()`) implement ไหม | **มี** | **มี** (ผ่าน blanket implementation ข้างบน) |

#### เปรียบเทียบกับภาษาอื่น: "for-each protocol" ไม่ใช่แนวคิดใหม่ แต่ Rust เปิดให้เห็นชัดกว่า

แนวคิด "collection แปลงเป็นตัวที่วนได้ (iterable → iterator)" ไม่ใช่สิ่งที่ Rust คิดขึ้นมาใหม่ — ภาษาระดับสูง
ส่วนใหญ่มี protocol แบบเดียวกัน เพียงแค่ตั้งชื่อและเปิดเผยรายละเอียดให้เห็นในระดับที่ต่างกัน:

```python
# Python: __iter__() คือ IntoIterator, __next__() คือ next()
class Countdown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):        # เทียบเท่า IntoIterator::into_iter()
        return self

    def __next__(self):        # เทียบเท่า Iterator::next()
        if self.current == 0:
            raise StopIteration   # เทียบเท่า None — แต่ใช้ exception แทน enum
        self.current -= 1
        return self.current + 1

for n in Countdown(5):   # Python เรียก iter(obj) แล้ววน next(it) จนกว่าจะเจอ StopIteration
    print(n, end=" ")
```

```java
// Java: Iterable<T> คือ IntoIterator, Iterator<T> คือ Iterator (แต่แยก hasNext()/next() สองตัว)
class Countdown implements Iterable<Integer> {
    private int start;
    Countdown(int start) { this.start = start; }

    public Iterator<Integer> iterator() {   // เทียบเท่า IntoIterator::into_iter()
        return new Iterator<Integer>() {
            int current = start;
            public boolean hasNext() { return current > 0; }   // ต้องเรียกก่อน next() เสมอ
            public Integer next() { return current--; }         // ไม่คืน Option — ถ้าเรียกผิดจังหวะ throw
        };
    }
}

for (int n : new Countdown(5)) {   // Java เรียก iterator() แล้ววน hasNext()/next() คู่กัน
    System.out.print(n + " ");
}
```

```cpp
// C++: begin()/end() คือ IntoIterator (คืน iterator สองตัวแทนหนึ่งตัว — จุดเริ่มกับจุดจบ)
class Countdown {
    int start;
public:
    Countdown(int s) : start(s) {}
    struct Iter {
        int current;
        int operator*() const { return current; }        // เทียบเท่า next() unwrap แล้ว
        Iter& operator++() { --current; return *this; }  // เดินหน้าหนึ่งช่อง
        bool operator!=(const Iter& other) const { return current != other.current; }
    };
    Iter begin() { return Iter{start}; }
    Iter end()   { return Iter{0}; }   // เทียบเท่า "จุดที่ถือว่าหมดแล้ว" — เทียบด้วย != ไม่ใช่ None
};

for (int n : Countdown(5)) {   // C++ เรียก begin()/end() แล้ววนจนกว่า iterator ปัจจุบันจะ == end()
    std::cout << n << " ";
}
```

**สังเกตความแตกต่างที่สำคัญที่สุด**: ทั้งสามภาษามี protocol ที่คล้ายกันมาก (แปลง collection เป็น "ตัววนได้"
ก่อน แล้ววนดึงค่าออกมาทีละตัว) แต่ **วิธีสื่อความหมาย "หมดแล้ว" ต่างกันโดยพื้นฐาน**:

| ภาษา | สื่อความหมาย "หมดแล้ว" ด้วย | ปัญหาที่อาจเกิด |
|---|---|---|
| Python | throw `StopIteration` exception | ต้นทุนของ exception, ต้องจำไว้เองว่า `next()` อาจ throw ได้เสมอ |
| Java | `hasNext()` แยก method ให้เรียกเช็คก่อน | เรียก `next()` โดยไม่เช็ค `hasNext()` ก่อนได้ (compiler ไม่ห้าม) — ได้ `NoSuchElementException` ตอน runtime |
| C++ | เทียบ iterator ปัจจุบันกับค่า `end()` ด้วย `!=` | ถ้าลืมเช็คหรือเทียบผิด (`==` ตอนควรใช้ `!=`) เป็นได้ทั้ง infinite loop และ undefined behavior |
| **Rust** | **`Option<Self::Item>`** — `None` เป็นส่วนหนึ่งของ return type | **compiler บังคับให้จัดการทั้ง `Some`/`None`** ผ่าน `match`/`while let` — ไม่มีทาง "ลืมเช็ค" ได้เลยเพราะไวยากรณ์ไม่ยอมให้เข้าถึงค่าข้างใน `Option` โดยไม่ผ่านการจัดการทั้งสองกรณีก่อน |

Rust เป็นภาษาเดียวในตารางนี้ที่ทำให้ "การเช็คว่าหมดหรือยัง" เป็น**ส่วนหนึ่งของ type system**ไม่ใช่ระเบียบปฏิบัติ
ที่ผู้เขียนโค้ดต้องจำเอาเอง — สอดคล้องกับปรัชญาของ `Option<T>` ที่ Part 11 สอนไว้ทุกประการ: **ทำให้ข้อผิดพลาดที่
เป็นไปได้ กลายเป็นสิ่งที่ compiler ตรวจสอบให้แทนที่จะพึ่งวินัยของโปรแกรมเมอร์**

### 25.7 สามวิธีวน `Vec<T>` แบบเต็มรูปแบบ: `.iter()`, `.iter_mut()`, `.into_iter()` และ `IntoIterator` สามตัวของ `Vec<T>`

นี่คือจุดที่เราจะปิดทีเซอร์ของ Part 13 หัวข้อ 13.6 ให้สมบูรณ์ 100% — เหตุผลที่ `for n in &numbers`, `for n in
&mut mutable_numbers`, และ `for s in owned_numbers` ให้ผลต่างกันโดยสิ้นเชิงคือ **`Vec<T>` (ผ่าน `&Vec<T>`,
`&mut Vec<T>`, และ `Vec<T>` เอง) implement `IntoIterator` ไว้ถึง 3 ตัวแยกกัน** โดยแต่ละตัวให้ `IntoIter` (และ
`Item`) คนละชนิด:

| เขียน | เรียก `IntoIterator` ของ | `Item` ที่ได้ | ความหมายด้าน ownership |
|---|---|---|---|
| `for x in &v` | `impl<'a, T> IntoIterator for &'a Vec<T>` | `&'a T` | **ยืมอ่าน** — `v` ใช้ต่อได้หลัง loop |
| `for x in &mut v` | `impl<'a, T> IntoIterator for &'a mut Vec<T>` | `&'a mut T` | **ยืมแก้ไข** — แก้ไขสมาชิกได้ระหว่างวน `v` ใช้ต่อได้หลัง loop |
| `for x in v` | `impl<T> IntoIterator for Vec<T>` | `T` | **ยึด ownership** — `v` ถูก move เข้า loop ทั้งก้อน ใช้ต่อไม่ได้หลัง loop |

มาดูทั้งสามแบบทำงานพร้อมพิสูจน์ type ของตัวแปรวนในแต่ละกรณีด้วย type annotation ตรง ๆ:

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    println!("=== for n in &numbers (ยืม, ได้ &i32) ===");
    for n in &numbers {
        let n: &i32 = n; // พิสูจน์ type จริงของ n
        print!("{n} ");
    }
    println!();
    println!("numbers ใช้ต่อได้: {:?}", numbers);

    let mut mutable_numbers = vec![1, 2, 3];
    println!("=== for n in &mut mutable_numbers (ยืมแบบแก้ไขได้, ได้ &mut i32) ===");
    for n in &mut mutable_numbers {
        let n: &mut i32 = n; // พิสูจน์ type จริงของ n
        *n *= 10;
    }
    println!("{:?}", mutable_numbers);

    let owned_numbers = vec![String::from("a"), String::from("b")];
    println!("=== for s in owned_numbers (ยึด ownership, ได้ String) ===");
    for s in owned_numbers {
        let s: String = s; // พิสูจน์ type จริงของ s
        println!("owned: {s}");
    }

    // เรียก .iter() / .iter_mut() / .into_iter() ตรง ๆ (ไม่ผ่าน for) ให้เห็น type ของ .next() ชัด ๆ
    let v = vec![100, 200, 300];
    let mut it1 = v.iter();
    let first: Option<&i32> = it1.next();
    println!("v.iter().next() -> {:?}", first);

    let mut v2 = vec![100, 200, 300];
    let mut it2 = v2.iter_mut();
    let first_mut: Option<&mut i32> = it2.next();
    if let Some(x) = first_mut {
        *x += 1;
    }
    println!("v2 หลังแก้ผ่าน iter_mut: {:?}", v2);

    let v3 = vec![100, 200, 300];
    let mut it3 = v3.into_iter();
    let first_owned: Option<i32> = it3.next();
    println!("v3.into_iter().next() -> {:?}", first_owned);
}
```

ผลลัพธ์:

```
=== for n in &numbers (ยืม, ได้ &i32) ===
1 2 3 
numbers ใช้ต่อได้: [1, 2, 3]
=== for n in &mut mutable_numbers (ยืมแบบแก้ไขได้, ได้ &mut i32) ===
[10, 20, 30]
=== for s in owned_numbers (ยึด ownership, ได้ String) ===
owned: a
owned: b
v.iter().next() -> Some(100)
v2 หลังแก้ผ่าน iter_mut: [101, 200, 300]
v3.into_iter().next() -> Some(100)
```

**เชื่อมทุกอย่างให้เห็นภาพเดียว**: `for n in &numbers` **ไม่ได้**เรียก `.iter()` ตรง ๆ ในทางเทคนิค — มันเรียก
`IntoIterator::into_iter(&numbers)` ซึ่ง**ในกรณีของ `&Vec<T>` มันเทียบเท่ากับการเรียก `.iter()`** (แท้จริง
`impl<'a, T> IntoIterator for &'a Vec<T>` ก็ implement `into_iter()` ด้วยการเรียก `self.iter()` ข้างในนั่นเอง)
ในทางปฏิบัติจึงพูดได้ว่า `for x in &v` "เทียบเท่า" กับ `for x in v.iter()` — และ `for x in &mut v` เทียบเท่ากับ
`for x in v.iter_mut()`, `for x in v` เทียบเท่ากับ `for x in v.into_iter()` (จริง ๆ แล้วเทียบเท่าอย่างสมบูรณ์
100% เพราะ `into_iter()` ของ `Vec<T>` เองก็แค่คืน struct ที่เก็บ ownership ของ buffer ไว้ ทำหน้าที่เดียวกัน)

**หลักการเลือกที่ควรจำ (ทวนจาก Part 13 แต่ตอนนี้เข้าใจเหตุผลลึกกว่าเดิม)**: ใช้ `for x in &v` เป็นค่าเริ่มต้น
เสมอถ้าไม่แน่ใจ — เปลี่ยนไปใช้ `for x in &mut v` เมื่อต้องแก้ไขสมาชิกระหว่างวน และเปลี่ยนไปใช้ `for x in v`
(by value) เฉพาะเมื่อตั้งใจจริง ๆ ที่จะ "จบชีวิตของ collection ตัวนั้น" หลัง loop เพราะแต่ละตัวเลือก
`IntoIterator` implementation คนละตัวที่มีความหมายด้าน ownership ต่างกันโดยสิ้นเชิงตามตารางข้างบน ไม่ใช่แค่
"syntax ที่ต่างกันเล็กน้อย"

### 25.8 เขียน custom iterator ของตัวเอง: `Countdown`

ถึงเวลาพิสูจน์ว่าการ implement `Iterator` ให้ type ของเราเองไม่ใช่เรื่องยาก — มาสร้าง **`Countdown`**:
struct ที่นับถอยหลังจากตัวเลขที่กำหนดลงไปถึง 1 แล้วหยุด

```rust
struct Countdown {
    current: u32,
}

impl Countdown {
    fn new(start: u32) -> Countdown {
        Countdown { current: start }
    }
}

impl Iterator for Countdown {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.current == 0 {
            None
        } else {
            let value = self.current;
            self.current -= 1;
            Some(value)
        }
    }
}

fn main() {
    println!("=== วนด้วย for (Countdown implement Iterator เอง) ===");
    for n in Countdown::new(5) {
        print!("{n} ");
    }
    println!();

    let mut cd = Countdown::new(3);
    println!("เรียก next() ด้วยมือ: {:?} {:?} {:?} {:?}", cd.next(), cd.next(), cd.next(), cd.next());
}
```

ผลลัพธ์:

```
=== วนด้วย for (Countdown implement Iterator เอง) ===
5 4 3 2 1 
เรียก next() ด้วยมือ: Some(3) Some(2) Some(1) None
```

**อธิบายทีละส่วน:**

- **`struct Countdown { current: u32 }`** — struct ธรรมดา (Part 9) เก็บสถานะภายในไว้แค่ตัวเดียว: "ตัวเลข
  ปัจจุบันที่นับถอยหลังมาถึงแล้ว"
- **`impl Countdown { fn new(start: u32) -> Countdown }`** — constructor ธรรมดา ไม่เกี่ยวกับ `Iterator` เลย
  แค่สร้าง `Countdown` ตัวแรกที่ `current == start`
- **`impl Iterator for Countdown { type Item = u32; fn next(&mut self) -> Option<u32> { ... } }`** — นี่คือ
  ส่วนที่ทำให้ `Countdown` เป็น iterator จริง ๆ ตาม logic:
  - ถ้า `self.current == 0` แล้ว → **ไม่มีอะไรจะนับต่อ** → คืน `None`
  - ไม่งั้น → เก็บค่าปัจจุบันไว้ใน `value`, **ลด `self.current` ลง 1** (นี่คือเหตุผลที่ `next()` ต้องรับ
    `&mut self` — มันต้อง**แก้ไขสถานะภายใน**ทุกครั้งที่ถูกเรียก ไม่งั้นจะวนซ้ำค่าเดิมไม่มีที่สิ้นสุด) แล้วคืน
    `Some(value)`
- **`for n in Countdown::new(5)`** — ใช้งานได้ตรง ๆ ทั้งที่เราไม่ได้ implement `IntoIterator` ให้ `Countdown`
  เลย เพราะ blanket implementation ที่อธิบายไว้ในหัวข้อ 25.6 (`impl<I: Iterator> IntoIterator for I`) ให้
  `IntoIterator` มาโดยอัตโนมัติกับทุก type ที่ implement `Iterator` แล้ว

สังเกตว่า logic ของ `next()` ในตัวอย่างนี้**เหมือนกันเป๊ะ**กับ logic ของ loop เช็คเงื่อนไข + ลดค่า ที่คุณอาจ
เคยเขียนด้วยมือมาก่อนตั้งแต่ Part 4 (`while current > 0 { ...; current -= 1; }`) — ความแตกต่างคือตอนนี้ logic
นั้นถูก**บรรจุไว้ในรูปแบบมาตรฐาน**ที่ทำให้ `Countdown` ใช้งานร่วมกับทุกอย่างในระบบ iterator ของ Rust ได้ทันที
(ทั้ง `for`, `.map()`, `.sum()`, `.collect()`, ฯลฯ) โดยไม่ต้องเขียนอะไรเพิ่มอีกเลย — นี่คือประเด็นของหัวข้อ
ถัดไป

### 25.9 ได้ 70+ methods มา "ฟรี": พลังของ minimal required method + default method

ลองเรียก method ที่เราไม่ได้เขียนเองแม้แต่บรรทัดเดียวบน `Countdown`:

```rust
struct Countdown {
    current: u32,
}

impl Countdown {
    fn new(start: u32) -> Countdown {
        Countdown { current: start }
    }
}

impl Iterator for Countdown {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.current == 0 {
            None
        } else {
            let value = self.current;
            self.current -= 1;
            Some(value)
        }
    }
}

fn main() {
    println!("sum   = {}", Countdown::new(5).sum::<u32>());
    println!("count = {}", Countdown::new(5).count());
    println!("max   = {:?}", Countdown::new(5).max());
    println!("min   = {:?}", Countdown::new(5).min());
    println!("collect เป็น Vec: {:?}", Countdown::new(5).collect::<Vec<u32>>());
    println!("nth(2) = {:?}", Countdown::new(5).nth(2));
    println!("last() = {:?}", Countdown::new(5).last());
}
```

ผลลัพธ์:

```
sum   = 15
count = 5
max   = Some(5)
min   = Some(1)
collect เป็น Vec: [5, 4, 3, 2, 1]
nth(2) = Some(3)
last() = Some(1)
```

**เราไม่ได้เขียน `.sum()`, `.count()`, `.max()`, `.min()`, `.collect()`, `.nth()`, `.last()` ให้ `Countdown`
เองแม้แต่ตัวเดียว** — ทั้งหมดนี้คือ**default method**ของ trait `Iterator` ที่ประกาศไว้ครั้งเดียวใน std library
และใช้ได้กับ**ทุก type ที่ implement `Iterator`** โดยอัตโนมัติ

**ทำไมสิ่งนี้เป็นไปได้?** เพราะ default method ทุกตัวของ `Iterator` **เขียนโดยเรียก `self.next()` เท่านั้น**
ไม่รู้จักและไม่จำเป็นต้องรู้จักรายละเอียดภายในของ `Countdown` เลยแม้แต่นิดเดียว ลองดูว่า `.count()` แบบง่าย ๆ
อาจถูกเขียนภายในประมาณนี้ (pseudocode อธิบายหลักการ ไม่ใช่โค้ดจริงจาก std เป๊ะ ๆ):

```
fn count(mut self) -> usize {
    let mut n = 0;
    while self.next().is_some() {
        n += 1;
    }
    n
}
```

สังเกตว่าฟังก์ชันนี้**ไม่รู้จัก** `Countdown` เลย มันรู้จักแค่ว่า `self` มี method ชื่อ `next()` ที่คืน
`Option<Self::Item>` ได้ — เพราะฉะนั้นมันใช้ได้กับ `Countdown`, `Vec::iter()`, `HashMap::iter()`, หรือ iterator
อะไรก็ตามที่คุณเขียนขึ้นมาเองในอนาคต **โดยไม่ต้องแก้ไข `.count()` เลยแม้แต่บรรทัดเดียว**

นี่คือรูปแบบเดียวกันกับที่ Part 19 สอนไว้เรื่อง **default method ของ trait** (method ที่มี body อยู่ในนิยาม
trait เอง ผู้ implement ไม่ต้องเขียนซ้ำ) เพียงแต่ `Iterator` นำหลักการนี้ไปใช้ **ในสเกลที่ใหญ่กว่ามาก**: มี
required method แค่ **1 ตัว** (`next()`) แต่มี default method **มากกว่า 70 ตัว** ที่สร้างจาก required method
ตัวเดียวนั้นทั้งหมด — นี่คือเหตุผลเชิง design ที่สำคัญที่สุดของทั้งบท: **ยิ่ง "ผิวสัมผัส" (surface) ของสิ่งที่
ต้อง implement เล็กเท่าไหร่ ยิ่งมีโอกาสที่ trait นั้นจะให้ "พลัง" กลับมามากเท่านั้น** เพราะทุกคนที่ implement
`Iterator` (ไม่ว่าจะเป็น std เอง หรือคุณที่เพิ่งเขียน `Countdown`) ต่างได้ผลประโยชน์เดียวกันจาก default method
ชุดเดียวกันหมด — เขียนครั้งเดียว ใช้ได้กับ type ในอนาคตที่ยังไม่มีอยู่ด้วยซ้ำตอนที่เขียน `Iterator` trait นี้
ครั้งแรก

#### เปรียบเทียบกับภาษาอื่น: "ได้ method มาฟรี" ทำได้แบบนี้ในภาษาอื่นไหม

แนวคิด "implement method เดียว แล้วได้ method อื่นมาฟรีจากมัน" มีอยู่ในภาษาอื่นด้วยเช่นกัน แต่กลไกที่ใช้รองรับ
มันต่างกันมาก และไม่ได้ให้ "พลัง" เท่ากับที่ trait ของ Rust ให้:

- **Java (ตั้งแต่ Java 8)**: `interface` มี **default method** ได้ (`default int size() { ... }`) คล้ายกับ
  default method ของ trait ใน Rust — แต่ **`Iterable<T>`/`Iterator<T>` ของ Java ไม่ได้ใช้กลไกนี้เต็มรูปแบบ**
  method อย่าง `.stream().map().filter().collect()` ต้องแปลงเป็น `Stream<T>` ก่อน (คนละ type จาก `Iterator<T>`
  เดิม) ทำให้ "ได้ของฟรี" ไม่ได้ตรง ๆ จาก `Iterator<T>` เอง ต้องผ่านชั้นแปลง type เพิ่มอีกชั้นหนึ่งเสมอ
- **Python**: `collections.abc.Iterator` (abstract base class) ให้ method อย่าง `__contains__`,
  `__length_hint__` มาฟรีถ้า implement `__iter__`/`__next__` ครบ — ใกล้เคียงกับหลักการของ Rust ในระดับหนึ่ง
  แต่ Python ใช้ **duck typing** (ตรวจสอบตอน runtime ว่ามี method ที่ต้องการหรือไม่) ไม่ใช่ trait ที่ compiler
  ตรวจสอบให้ตั้งแต่ compile time เหมือน Rust — โค้ดที่เขียนผิด (เช่น `__next__` คืนค่าผิด type) จะรู้ตัวก็ตอน
  รันจริงเท่านั้น ต่างจาก Rust ที่ compiler ปฏิเสธตั้งแต่ compile time ทันทีถ้า `next()` มี signature ไม่ตรงกับ
  ที่ `Iterator` ต้องการ
- **Go**: ไม่มี default method บน interface เลย (`interface` ของ Go เป็นแค่รายชื่อ method ที่ต้อง implement
  ครบทุกตัวเอง ไม่มีแนวคิด "default implementation") ทำให้ต้องเขียน `.Map()`, `.Filter()`, `.Sum()` ซ้ำเองทุก
  ครั้งสำหรับแต่ละ type ที่ต้องการใช้ (หรือใช้ 3rd-party library ที่จำลองด้วย generics ในเวอร์ชันใหม่ ๆ)

**สิ่งที่ทำให้ Rust ต่างจากทุกภาษาข้างบนอย่างชัดเจน**: `Iterator` เป็น trait ที่ **compiler ตรวจสอบความถูกต้อง
ของ `next()` ให้ตั้งแต่ compile time ทุกจุด** (ผ่าน associated type `Item` ที่ต้องตรงกันทั้ง 70+ default
method) **และ** default method ทั้งหมดใช้งานร่วมกับ `.map()`, `.filter()`, generic function ที่รับ
`T: Iterator` ได้โดยตรงไม่ต้องแปลง type ข้ามชั้นเหมือน Java's `Stream` — นี่คือเหตุผลที่การเขียน custom
iterator ใน Rust "คุ้มค่า" กว่าภาษาอื่นมาก: implement แค่ `next()` ตัวเดียว ได้ทุกอย่างที่ระบบ iterator ทั้งหมด
มีให้ทันที ไม่มีชั้นแปลง type แทรกกลาง ไม่มีความเสี่ยงเรื่อง runtime type error

### 25.10 Consuming Adaptor เทียบกับ Iterator Adaptor: การแบ่งประเภทที่สำคัญที่สุด

Method ของ `Iterator` ที่ได้มาฟรีทั้งหมดกว่า 70 ตัว แบ่งออกเป็นสองกลุ่มใหญ่ที่มีพฤติกรรมต่างกันโดยพื้นฐาน:

| กลุ่ม | ทำอะไร | คืนอะไร | ตัวอย่าง |
|---|---|---|---|
| **Consuming adaptor** | "ขับเคลื่อน" (drive) iterator จนกว่าจะได้คำตอบสุดท้าย — เรียก `.next()` ซ้ำ ๆ จนหมดหรือจนพอ | **ค่าสุดท้ายค่าเดียว** (ไม่ใช่ iterator อีกต่อไป) | `.sum()`, `.count()`, `.collect()`, `.max()`, `.min()`, `.fold()`, `.for_each()`, `.find()`, `.any()`, `.all()` |
| **Iterator adaptor** | "ห่อ" iterator เดิมด้วย logic เพิ่มเติม | **iterator ตัวใหม่** (ยังไม่ได้ประมวลผลอะไรจริง — ดูหัวข้อ 25.11) | `.map()`, `.filter()`, `.zip()`, `.enumerate()`, `.take()`, `.skip()`, `.rev()`, `.chain()` |

ลองดูทั้งสองกลุ่มทำงานในตัวอย่างเดียวกัน:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // iterator adaptor: .filter() และ .map() ไม่ทำอะไรทันที คืน iterator ตัวใหม่กลับมาเสมอ
    let adapted = numbers
        .iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * n);

    // consuming adaptor: .sum() "ขับเคลื่อน" iterator จนหมด แล้วคืนค่าสุดท้ายค่าเดียว
    let total: i32 = adapted.sum();
    println!("ผลรวมของกำลังสองเลขคู่: {total}");

    let count = numbers.iter().filter(|&&n| n > 5).count();
    println!("จำนวนที่มากกว่า 5: {count}");

    let max = numbers.iter().max();
    println!("ค่ามากที่สุด: {:?}", max);

    let folded = numbers.iter().fold(0, |acc, &n| acc + n);
    println!("fold รวมทั้งหมด: {folded}");
}
```

ผลลัพธ์:

```
ผลรวมของกำลังสองเลขคู่: 220
จำนวนที่มากกว่า 5: 5
ค่ามากที่สุด: Some(10)
fold รวมทั้งหมด: 55
```

**สังเกต type ของตัวแปร `adapted`**: มัน**ไม่ใช่** `Vec<i32>` หรือ `i32` — มันเป็น type ที่ซับซ้อนมาก (จริง ๆ
คือ `Map<Filter<std::slice::Iter<'_, i32>, {closure}>, {closure}>` — struct ที่ซ้อนกันหลายชั้นแทน "สายพาน"
ของการประมวลผล) เราไม่จำเป็นต้องเขียน type นี้ตรง ๆ เอง (compiler infer ให้ผ่าน `let adapted = ...` โดยไม่มี
type annotation) แต่ควรรู้ไว้ว่า **`.filter()` และ `.map()` ไม่ได้สร้างข้อมูลใหม่ทันที มันสร้าง "โครงสร้างที่
บรรยายว่าจะประมวลผลอย่างไร" เท่านั้น** — จนกว่าจะมี consuming adaptor (`.sum()` ในตัวอย่างนี้) มาเรียก
`.next()` จริง ๆ ตัวเลขจะยังไม่ถูกกรองหรือยกกำลังสองแม้แต่ตัวเดียว นี่คือหัวใจของหัวข้อถัดไป

**หลักการจำง่าย ๆ**: consuming adaptor คือ method ที่ **"จบสาย"** — เรียกแล้วไม่มี iterator เหลือให้เรียก
method ต่อได้อีก (บางตัวรับ `self` ไม่ใช่ `&mut self` ด้วยซ้ำ แปลว่ายึด ownership ของ iterator ไปเลย) ส่วน
iterator adaptor คือ method ที่ **"ต่อสาย"** — เรียกแล้วได้ iterator ตัวใหม่ที่ยัง chain ต่อ method อื่นได้อีก
เรื่อย ๆ

### 25.11 ความเฉื่อยชา (Laziness): พิสูจน์ด้วยโค้ดจริงว่า iterator adaptor ไม่ทำอะไรจนกว่าจะถูกขับเคลื่อน

นี่คือแนวคิดที่สำคัญที่สุดของหัวข้อ 25.10 — มาพิสูจน์แบบจับได้ไล่ทันด้วย side effect ที่มองเห็นได้จริง
(`println!` ข้างในตัว closure ของ `.map()`):

```rust
fn main() {
    let numbers = vec![1, 2, 3];

    println!("สร้าง iterator chain ด้วย .map() ...");
    let doubled_iter = numbers.iter().map(|n| {
        println!("  -> กำลังประมวลผล {n}");
        *n * 2
    });
    println!(".map() คืน iterator แล้ว แต่ยังไม่มีการประมวลผลเกิดขึ้นเลยสักตัว!");

    println!("เรียก .collect() เพื่อ 'ขับเคลื่อน' iterator ...");
    let doubled: Vec<i32> = doubled_iter.collect();
    println!("ผลลัพธ์: {doubled:?}");
}
```

ผลลัพธ์:

```
สร้าง iterator chain ด้วย .map() ...
.map() คืน iterator แล้ว แต่ยังไม่มีการประมวลผลเกิดขึ้นเลยสักตัว!
เรียก .collect() เพื่อ 'ขับเคลื่อน' iterator ...
  -> กำลังประมวลผล 1
  -> กำลังประมวลผล 2
  -> กำลังประมวลผล 3
ผลลัพธ์: [2, 4, 6]
```

**อ่านลำดับผลลัพธ์อย่างระมัดระวัง**: บรรทัด `.map() คืน iterator แล้ว แต่ยังไม่มีการประมวลผลเกิดขึ้นเลยสักตัว!`
ปรากฏขึ้น**ก่อน**บรรทัด `-> กำลังประมวลผล 1/2/3` ทั้งที่ในโค้ด `numbers.iter().map(|n| { println!(...); ...
})` ถูกเขียนไว้**ก่อน**บรรทัดนั้นด้วยซ้ำ! นี่คือหลักฐานที่จับต้องได้ 100% ว่า **`.map()` เพียงแค่สร้าง struct
`Map` ที่เก็บ closure ไว้เฉย ๆ — ไม่ได้เรียก closure นั้นแม้แต่ครั้งเดียว** จนกว่าจะถึงบรรทัด
`doubled_iter.collect()` ที่เริ่มเรียก `.next()` ซ้ำ ๆ (ผ่าน default implementation ของ `.collect()`) ซึ่งแต่
ละครั้งที่ `.next()` ของ `Map` ถูกเรียก มันจะไปเรียก `.next()` ของ iterator ข้างในก่อน (ในที่นี้คือ
`numbers.iter()`) แล้วเอาค่าที่ได้ไปรันผ่าน closure — **ตรงจุดนั้นเองที่ `println!` ข้างในถูกรันจริง**

**นี่คือแนวคิดที่เรียกว่า Iterator เป็น lazy (เฉื่อยชา)** — iterator adaptor ไม่ทำอะไรเลยจนกว่าจะมีคนมา "ดึง"
ค่าจากมันจริง ๆ (ผ่าน `.next()`, ไม่ว่าจะเรียกเองตรง ๆ, ผ่าน `for`, หรือผ่าน consuming adaptor) เปรียบเทียบกับ
ภาษาที่ collection method ทำงานแบบ **eager (กระตือรือร้น)**: ใน Python `[x * 2 for x in range(1_000_000) if x
% 2 == 0]` จะสร้าง **list ทั้งก้อนในหน่วยความจำทันที** ก่อนที่จะรู้ด้วยซ้ำว่าคุณต้องการใช้ผลลัพธ์กี่ตัว

**ทำไม laziness ถึงสำคัญมากด้าน performance — "Iterator Fusion":**

ลองนึกภาพ chain ที่ซับซ้อนกว่า: `v.iter().filter(pred).map(f).take(3)` ถ้า iterator ทำงานแบบ eager (สร้าง
`Vec` ใหม่ทุกขั้น) ลำดับการทำงานจะเป็น:

1. `filter(pred)` วนทั้ง `v` สร้าง `Vec` ใหม่เก็บที่ผ่านเงื่อนไข (จองหน่วยความจำใหม่ 1 ก้อน)
2. `map(f)` วนทั้ง `Vec` จากขั้นที่ 1 สร้าง `Vec` ใหม่อีกก้อน (จองหน่วยความจำใหม่อีก 1 ก้อน)
3. `take(3)` ตัดเอาแค่ 3 ตัวแรกจาก `Vec` ก้อนที่สอง

สังเกตว่าขั้นที่ 1 และ 2 **ประมวลผลสมาชิกทุกตัวของ `v` แม้ว่าสุดท้ายเราต้องการแค่ 3 ตัวแรกเท่านั้น** — สิ้นเปลือง
ทั้งเวลาและหน่วยความจำอย่างมากถ้า `v` มีสมาชิกเป็นล้านตัว

เพราะ iterator ของ Rust เป็น lazy สิ่งที่เกิดขึ้นจริงคือ: `.next()` ถูกเรียกครั้งเดียวจาก `.take(3)` ไปยัง
`.map(f)` ไปยัง `.filter(pred)` ไปยัง `v.iter()` **สมาชิกไหลผ่านทั้ง chain ทีละตัว** — ตัวที่ 1 ของ `v` ถูกเช็ค
`filter`, ถ้าผ่านก็ผ่าน `map`, ถ้า `take` ยังไม่ครบ 3 ก็ไปดึงตัวที่ 2 ต่อ ทำแบบนี้ไปจนกว่า `take(3)` จะพอใจแล้ว
`break` ออกทันที — **ไม่มีการจอง `Vec` ตัวกลางแม้แต่ตัวเดียว ไม่มีสมาชิกที่ไม่จำเป็นถูกประมวลผลเลย** พฤติกรรมนี้
เรียกว่า **iterator fusion**: chain ของ adaptor หลายตัวถูก compiler รวม (fuse) เข้าด้วยกันเป็น loop เดียวที่มี
ประสิทธิภาพเทียบเท่ากับการเขียน loop ด้วยมือทุกประการ (Rust เรียกหลักการนี้ว่า **zero-cost abstraction** — จะ
เห็นการพิสูจน์เชิงลึกกว่านี้อีกใน Part 26 และ Part 54-56 เรื่อง performance) นี่คือเหตุผลสำคัญที่สุดที่ทำให้โค้ด
Rust ที่ใช้ iterator chain ยาว ๆ **ไม่ได้ช้ากว่าการเขียน loop เองด้วยมือเลยแม้แต่นิดเดียว** ต่างจากภาษาที่แต่ละ
ขั้นตอนสร้าง collection ตัวกลางขึ้นมาจริง ๆ

**สิ่งที่ควรระวังคู่กัน**: เพราะ iterator adaptor ไม่ทำอะไรถ้าไม่ถูกขับเคลื่อน การเขียน
`numbers.iter().map(|n| n * 2);` (ไม่มีการเก็บผลลัพธ์หรือ consume ต่อ) จะ**ไม่มีผลอะไรเลย** — compiler รุ่น
ใหม่ของ Rust จะเตือนเรื่องนี้ให้ทันทีด้วย warning (จะเห็นข้อความจริงในหัวข้อกับดักที่ 3)

### 25.12 `.collect()` เจาะลึก: ทำไมต้องบอก type ปลายทาง

`.collect()` คือ consuming adaptor ที่ใช้บ่อยที่สุดในโค้ด Rust จริง — มันรวบรวมค่าจาก iterator ทั้งหมดเข้าไปใน
collection ปลายทาง แต่มีลักษณะพิเศษที่ต่างจาก `.sum()`, `.count()`: **มันไม่รู้ว่าจะ "รวบรวมเข้าไปใน type
ไหน" จนกว่าจะมีใครบอก**

`.collect()` นิยามคร่าว ๆ (pseudocode) หน้าตาประมาณ:

```
fn collect<B: FromIterator<Self::Item>>(self) -> B
```

สังเกตว่า `B` (type ปลายทาง) เป็น **generic type parameter ของ method** (ตามหลักการ Part 18 ที่บอกว่า type
parameter ของ method แยกจาก type parameter ของ struct/trait ได้) และ `.collect()` ต้องการแค่ว่า `B` implement
trait `FromIterator<Self::Item>` — trait ที่บอกว่า "รู้วิธีสร้างตัวเองจาก iterator" (`Vec<T>`, `String`,
`HashMap<K, V>`, `HashSet<T>` ทั้งหมด implement `FromIterator` ไว้แล้วใน std)

ปัญหาคือ: ในนิพจน์ `numbers.iter().filter(...).collect()` **ไม่มีที่ไหนบอก compiler ว่า `B` คือ type อะไร**
เลยถ้าไม่มี context เพิ่มเติมช่วย — มาดู error จริงที่เกิดขึ้น:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let evens = numbers.iter().filter(|&&n| n % 2 == 0).collect();
    println!("{:?}", evens);
}
```

```
error[E0283]: type annotations needed
 --> src/main.rs:3:9
  |
3 |     let evens = numbers.iter().filter(|&&n| n % 2 == 0).collect();
  |         ^^^^^                                           ------- type must be known at this point
  |
  = note: cannot satisfy `_: FromIterator<&i32>`
note: required by a bound in `collect`
help: consider giving `evens` an explicit type
  |
3 |     let evens: Vec<_> = numbers.iter().filter(|&&n| n % 2 == 0).collect();
  |              ++++++++

error: aborting due to 1 previous error
```

**หมายเหตุสำคัญเรื่องเลข error**: หลาย ๆ แหล่งอ้างอิงของ Rust (รวมถึงเอกสารรุ่นเก่า) เรียก error กลุ่มนี้ว่า
`E0282` ("type annotations needed" แบบทั่วไป เมื่อ compiler infer type ไม่ได้เลย) แต่บน compiler รุ่นปัจจุบัน
(`rustc 1.94.1` ที่ใช้ตรวจสอบทุกตัวอย่างในบทนี้) กรณี `.collect()` กำกวมแบบนี้เจาะจงลงไปเป็น **`E0283`**
("type annotations needed" เช่นกัน แต่เกิดจากการ**เลือก trait implementation ไม่ได้** — compiler หา
`FromIterator` ที่ตรงกับสถานการณ์นี้ไม่ได้เพราะมีหลาย `B` ที่เป็นไปได้ ไม่ใช่แค่ "ไม่มีข้อมูลพอจะเดา") ทั้งสอง
error code สื่อความหมายเดียวกันในทางปฏิบัติ — **"ให้ context เพิ่มเติมกับฉันหน่อยว่าอยากได้ type ไหน"** — สิ่ง
สำคัญที่ต้องจำไม่ใช่เลข error code ที่แน่นอน (ซึ่งเปลี่ยนได้ตามเวอร์ชัน compiler) แต่คือ **ข้อความ "type
annotations needed" และวิธีแก้สองแบบที่เหมือนกันเสมอไม่ว่าจะเจอ E0282, E0283 หรือ E0284 (ตัวแปรของปัญหาเดียวกัน
ที่จะเห็นในกับดักท้ายบท)**

**สองวิธีแก้ที่ error message แนะนำไว้ตรง ๆ:**

**วิธีที่ 1 — ระบุ type บนตัวแปร:**

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let doubled: Vec<i32> = numbers.iter().map(|n| n * 2).collect();
    println!("doubled: {:?}", doubled);
}
```

**วิธีที่ 2 — turbofish บน `.collect()` เอง** (เทียบกับ turbofish ที่เรียนใน Part 18 ตอนเรียก generic function
ตรง ๆ ด้วย `::<T>`):

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let tripled = numbers.iter().map(|n| n * 3).collect::<Vec<i32>>();
    println!("tripled: {:?}", tripled);
}
```

ทั้งสองวิธีให้ผลลัพธ์เดียวกันทุกประการ — เลือกใช้แบบไหนขึ้นกับความอ่านง่ายในบริบทนั้น ๆ (วิธีที่ 1 อ่านง่ายกว่า
เมื่อชื่อตัวแปรสั้น, วิธีที่ 2 สะดวกกว่าเมื่อ chain ยาวมากจนการเขียน type ไว้หน้าตัวแปรทำให้บรรทัดแรกอ่านยาก
หรือเมื่อไม่ได้เก็บผลลัพธ์ไว้ในตัวแปรเลยแต่ส่งเข้าฟังก์ชันต่อทันที)

**`.collect()` รวบรวมเข้า type อื่นได้อีกมาก ไม่ใช่แค่ `Vec<T>`:**

```rust
use std::collections::{HashMap, HashSet};

fn main() {
    // collect เป็น String จาก iterator ของ char
    let word: String = vec!['R', 'u', 's', 't'].into_iter().collect();
    println!("word: {word}");

    // collect เป็น HashMap<K, V> จาก iterator ของ tuple (K, V) — เชื่อม Part 15
    let names = vec!["สมชาย", "สมหญิง", "มานะ"];
    let name_lengths: HashMap<&str, usize> =
        names.iter().map(|name| (*name, name.chars().count())).collect();
    let mut keys: Vec<&&str> = name_lengths.keys().collect();
    keys.sort();
    for k in keys {
        println!("{k} -> {} ตัวอักษร", name_lengths[k]);
    }

    // collect เป็น HashSet<T> เพื่อดึงค่าไม่ซ้ำ
    let with_duplicates = vec![1, 2, 2, 3, 3, 3, 4];
    let unique: HashSet<i32> = with_duplicates.into_iter().collect();
    let mut unique_sorted: Vec<i32> = unique.into_iter().collect();
    unique_sorted.sort();
    println!("unique: {:?}", unique_sorted);
}
```

ผลลัพธ์:

```
word: Rust
มานะ -> 4 ตัวอักษร
สมชาย -> 5 ตัวอักษร
สมหญิง -> 6 ตัวอักษร
unique: [1, 2, 3, 4]
```

สังเกตว่า `.collect()` เป็น `HashMap<&str, usize>` ได้เพราะ iterator ให้ค่าเป็น **tuple สองสมาชิก**
`(&str, usize)` — `HashMap<K, V>` implement `FromIterator<(K, V)>` ไว้พอดี (ตีความ tuple แต่ละตัวว่าเป็น
`(key, value)`) เช่นเดียวกับ `HashSet<T>` ที่ implement `FromIterator<T>` แล้วทิ้งค่าที่ซ้ำกันอัตโนมัติตาม
หลักการของ set ที่เรียนใน Part 15 (เก็บได้แค่ค่าที่ไม่ซ้ำกัน) — นี่คือพลังของการออกแบบผ่าน trait
`FromIterator`: `.collect()` ตัวเดียวใช้ได้กับ**ทุก type ปลายทาง**ที่ implement trait นี้ ไม่ต้องมี method
แยกชื่อ `.collect_to_vec()`, `.collect_to_hashmap()` ให้จำหลายชื่อเลย

### 25.13 Iterator Adaptor เบื้องต้นที่ใช้บ่อยที่สุด

มาดู iterator adaptor พื้นฐานที่พบบ่อยที่สุดในโค้ด Rust จริงทีละตัว (Part 26 จะเจาะลึก adaptor อื่นที่ซับซ้อน
กว่านี้อีก เช่น `.flat_map()`, `.scan()`, `.peekable()`, `.windows()`-like patterns — นี่คือแค่ชุดพื้นฐานที่
ต้องคุ้นก่อน):

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // .filter(): เก็บเฉพาะที่ผ่านเงื่อนไข
    let evens: Vec<&i32> = numbers.iter().filter(|&&n| n % 2 == 0).collect();
    println!(".filter() เลขคู่: {:?}", evens);

    // .map(): แปลงค่าทุกตัว
    let squared: Vec<i32> = numbers.iter().map(|&n| n * n).collect();
    println!(".map() กำลังสอง: {:?}", squared);

    // .enumerate(): จับคู่ index กับค่า (ทีเซอร์มาตั้งแต่ Part 8)
    let fruits = vec!["แอปเปิล", "กล้วย", "ส้ม"];
    for (i, fruit) in fruits.iter().enumerate() {
        println!(".enumerate() [{i}] = {fruit}");
    }

    // .zip(): จับคู่สอง iterator เข้าด้วยกัน หยุดที่ตัวสั้นกว่า
    let names = vec!["สมชาย", "มานะ"];
    let ages = vec![35, 40, 99]; // ตัวที่ 3 จะถูกทิ้งเพราะ names สั้นกว่า
    for (name, age) in names.iter().zip(ages.iter()) {
        println!(".zip(): {name} อายุ {age} ปี");
    }

    // .rev(): กลับลำดับ
    let reversed: Vec<&i32> = numbers.iter().rev().collect();
    println!(".rev(): {:?}", reversed);

    // .take(n): เอาแค่ n ตัวแรก
    let first_three: Vec<&i32> = numbers.iter().take(3).collect();
    println!(".take(3): {:?}", first_three);

    // .skip(n): ข้าม n ตัวแรกไป
    let skip_seven: Vec<&i32> = numbers.iter().skip(7).collect();
    println!(".skip(7): {:?}", skip_seven);

    // .chain(): ต่อ iterator สองตัวเข้าด้วยกันเป็นตัวเดียว
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let chained: Vec<i32> = a.into_iter().chain(b.into_iter()).collect();
    println!(".chain(): {:?}", chained);

    // ประกอบหลาย adaptor เข้าด้วยกันในเชนเดียว (แสดงพลังของการ chain ต่อกันได้ไม่จำกัด)
    let result: Vec<i32> = (1..=20)
        .filter(|n| n % 3 == 0)
        .map(|n| n * 10)
        .take(3)
        .collect();
    println!("chain รวม filter->map->take: {:?}", result);
}
```

ผลลัพธ์:

```
.filter() เลขคู่: [2, 4, 6, 8, 10]
.map() กำลังสอง: [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
.enumerate() [0] = แอปเปิล
.enumerate() [1] = กล้วย
.enumerate() [2] = ส้ม
.zip(): สมชาย อายุ 35 ปี
.zip(): มานะ อายุ 40 ปี
.rev(): [10, 9, 8, 7, 6, 5, 4, 3, 2, 1]
.take(3): [1, 2, 3]
.skip(7): [8, 9, 10]
.chain(): [1, 2, 3, 4, 5, 6]
chain รวม filter->map->take: [30, 60, 90]
```

**สรุป signature และความหมายของแต่ละตัวสั้น ๆ:**

| Method | รับ | ให้ Item | หยุดเมื่อไหร่ |
|---|---|---|---|
| `.filter(predicate)` | closure `Fn(&Item) -> bool` | Item เดิม เฉพาะตัวที่ `predicate` คืน `true` | เมื่อ iterator ต้นทางหมด |
| `.map(f)` | closure `FnMut(Item) -> U` | `U` (ผลลัพธ์จาก `f`) | เมื่อ iterator ต้นทางหมด |
| `.enumerate()` | ไม่รับอะไร | `(usize, Item)` — index คู่กับค่า | เมื่อ iterator ต้นทางหมด |
| `.zip(other)` | iterator อีกตัว | `(Item, OtherItem)` | เมื่อ iterator ตัวใด**สั้นกว่า**หมดก่อน |
| `.rev()` | ไม่รับอะไร (ต้องเป็น `DoubleEndedIterator`) | Item เดิม แต่ลำดับกลับหัว | เมื่อ iterator ต้นทางหมด |
| `.take(n)` | จำนวน `n` | Item เดิม | เมื่อได้ครบ `n` ตัว **หรือ**ต้นทางหมดก่อน |
| `.skip(n)` | จำนวน `n` | Item เดิม (ข้าม `n` ตัวแรกไปก่อน) | เมื่อ iterator ต้นทางหมด |
| `.chain(other)` | iterator อีกตัว | Item เดิม (ต้อง type เดียวกัน) | เมื่อทั้งสอง iterator หมด |

สังเกตว่า **ทุกตัวในตารางนี้คือ iterator adaptor** (ไม่ใช่ consuming adaptor) — เรียกแล้วได้ iterator ตัวใหม่
กลับมาเสมอ ต้องมี `.collect()`, `for`, หรือ consuming adaptor อื่นมาขับเคลื่อนจริงถึงจะเห็นผลลัพธ์ ตามหลักการ
laziness ที่เรียนไปในหัวข้อ 25.11

### 25.14 ตัวอย่างจริง: ประมวลผล `Vec<Product>` ด้วย iterator chain + custom iterator "สินค้าใกล้หมด"

ปิดท้ายเนื้อหาด้วยตัวอย่างที่รวมทุกอย่างในบทนี้เข้าด้วยกัน — ใช้ `struct Product` สไตล์เดียวกับ Part 9/13
ประมวลผลด้วย iterator chain มาตรฐาน **และ**เขียน custom iterator ของเราเอง (`LowStock`) ที่คืนเฉพาะสินค้าที่
เหลือน้อยกว่าเกณฑ์ที่กำหนด — เชื่อมทั้งสองครึ่งของบทนี้ (implement `Iterator` เอง + ใช้ built-in adaptor) ให้
เห็นภาพเดียวกัน

```rust
struct Product {
    name: String,
    price: f64,
    quantity: u32,
}

impl Product {
    fn new(name: &str, price: f64, quantity: u32) -> Product {
        Product {
            name: name.to_string(),
            price,
            quantity,
        }
    }
}

/// custom iterator: วนผ่านสินค้าทั้งคลัง แต่คืนแค่ตัวที่ "ใกล้หมด" (quantity < threshold) เท่านั้น
struct LowStock<'a> {
    products: &'a [Product],
    threshold: u32,
    index: usize,
}

impl<'a> LowStock<'a> {
    fn new(products: &'a [Product], threshold: u32) -> LowStock<'a> {
        LowStock {
            products,
            threshold,
            index: 0,
        }
    }
}

impl<'a> Iterator for LowStock<'a> {
    type Item = &'a Product;

    fn next(&mut self) -> Option<&'a Product> {
        // วนหาตัวถัดไปที่ผ่านเงื่อนไข แล้วค่อย return — ถ้าไม่มีเหลือแล้วให้คืน None
        while self.index < self.products.len() {
            let candidate = &self.products[self.index];
            self.index += 1;
            if candidate.quantity < self.threshold {
                return Some(candidate);
            }
        }
        None
    }
}

fn main() {
    let inventory = vec![
        Product::new("ปากกา", 5.0, 100),
        Product::new("ยางลบ", 3.5, 60),
        Product::new("สมุด", 25.0, 8),
        Product::new("ไม้บรรทัด", 10.0, 3),
        Product::new("กระเป๋า", 250.0, 15),
        Product::new("เครื่องคิดเลข", 180.0, 4),
    ];

    let min_price = 10.0;
    let max_price = 200.0;

    println!("=== ขั้นที่ 1: กรองด้วย .filter() + แปลงด้วย .map() + รวมด้วย .collect() ===");
    let summaries: Vec<String> = inventory
        .iter()
        .filter(|p| p.price >= min_price && p.price <= max_price)
        .map(|p| format!("{} - {:.2} บาท ({} ชิ้น)", p.name, p.price, p.quantity))
        .collect();

    for s in &summaries {
        println!("{s}");
    }

    println!();
    println!("=== ขั้นที่ 2: ใช้ custom iterator LowStock ที่เขียนเอง ===");
    for p in LowStock::new(&inventory, 10) {
        println!("{} เหลือแค่ {} ชิ้น!", p.name, p.quantity);
    }

    println!();
    println!("=== ขั้นที่ 3: LowStock ก็ได้ method อย่าง .map()/.count()/.sum() มาฟรีเหมือนกัน ===");
    let low_stock_names: Vec<&str> = LowStock::new(&inventory, 10)
        .map(|p| p.name.as_str())
        .collect();
    println!("รายชื่อสินค้าใกล้หมด: {}", low_stock_names.join(", "));

    let low_stock_count = LowStock::new(&inventory, 10).count();
    println!("จำนวนสินค้าใกล้หมดทั้งหมด: {low_stock_count} รายการ");

    let total_low_stock_value: f64 = LowStock::new(&inventory, 10)
        .map(|p| p.price * p.quantity as f64)
        .sum();
    println!("มูลค่ารวมของสินค้าใกล้หมด: {total_low_stock_value:.2} บาท");
}
```

ผลลัพธ์:

```
=== ขั้นที่ 1: กรองด้วย .filter() + แปลงด้วย .map() + รวมด้วย .collect() ===
สมุด - 25.00 บาท (8 ชิ้น)
ไม้บรรทัด - 10.00 บาท (3 ชิ้น)
เครื่องคิดเลข - 180.00 บาท (4 ชิ้น)

=== ขั้นที่ 2: ใช้ custom iterator LowStock ที่เขียนเอง ===
สมุด เหลือแค่ 8 ชิ้น!
ไม้บรรทัด เหลือแค่ 3 ชิ้น!
เครื่องคิดเลข เหลือแค่ 4 ชิ้น!

=== ขั้นที่ 3: LowStock ก็ได้ method อย่าง .map()/.count()/.sum() มาฟรีเหมือนกัน ===
รายชื่อสินค้าใกล้หมด: สมุด, ไม้บรรทัด, เครื่องคิดเลข
จำนวนสินค้าใกล้หมดทั้งหมด: 3 รายการ
มูลค่ารวมของสินค้าใกล้หมด: 950.00 บาท
```

**อธิบายภาพรวมของโปรแกรมนี้:**

- **ขั้นที่ 1** ใช้ built-in adaptor ล้วน ๆ (`.filter()` → `.map()` → `.collect()`) เป็น pattern ที่พบบ่อยที่สุด
  ในโค้ด Rust จริง: **กรองก่อน แปลงทีหลัง รวบรวมเป็นค่าสุดท้าย** — สังเกตว่า `.filter()` รับ closure ที่รับ
  `&&Product` (เพราะ `.iter()` ให้ `&Product` แล้ว `.filter()` ห่อด้วย reference อีกชั้นตามที่เรียนในกับดักที่
  2 ท้ายบท) แต่เพราะเราเขียน `|p| p.price >= min_price && ...` (pattern `p` เดี่ยว ไม่ได้ destructure) `p`
  จึงมี type เป็น `&&Product` — ใช้งาน `.price` ได้ตรง ๆ อยู่ดีเพราะ auto-deref ของ operator `.` ที่เรียนมา
  ตั้งแต่ Part 9 ไล่ dereference ให้เองจนถึงชั้นที่หา field เจอ ไม่ต้องเขียน `p.price` เป็น `(**p).price`
- **ขั้นที่ 2** แสดง custom iterator `LowStock<'a>` ที่เขียนเอง — สังเกตว่า `next()` มี **loop ข้างในตัวเอง**
  (`while self.index < self.products.len() { ... }`) ต่างจาก `Countdown` ที่ `next()` ไม่มี loop เพราะแต่ละ
  ครั้งที่เรียกได้คำตอบทันที ในกรณีนี้ `LowStock` ต้อง **"ไล่หา" ตัวถัดไปที่ผ่านเงื่อนไข** ซึ่งอาจต้องข้ามหลาย
  ตัวก่อนจะเจอ (หรือไม่เจอเลยจนครบ ก็คืน `None`) — pattern "loop ข้างใน `next()` เพื่อข้ามตัวที่ไม่ผ่าน
  เงื่อนไข" นี้คือหลักการเดียวกันเป๊ะกับที่ `.filter()` ของ std เองต้องทำภายใน (เราเพิ่งเขียน `.filter()`
  เวอร์ชันของตัวเองแบบเจาะจงกับ `Product` โดยไม่รู้ตัว!)
- **`type Item = &'a Product;`** — สังเกตว่า `LowStock` เลือกให้ค่าเป็น **reference** (`&'a Product`) ไม่ใช่
  `Product` ตรง ๆ ตามหลักการจาก Part 13 หัวข้อ 13.12 (`low_stock` คืน `Vec<&'a Product>`) — LowStock แค่
  "ชี้บอก" ว่าตัวไหนน่าสนใจ ไม่ต้องการเป็นเจ้าของสินค้าตัวใหม่ ไม่ต้อง clone อะไรเลย lifetime `'a` ผูก
  `LowStock<'a>` เข้ากับ `inventory` ต้นทางเหมือนที่ `low_stock<'a>` ใน Part 13 ผูก `Vec<&'a Product>` ที่คืน
  ออกมา
- **ขั้นที่ 3** พิสูจน์อีกครั้งว่า `LowStock` ได้ `.map()`, `.count()`, `.sum()` มาฟรีทั้งหมด (เหมือนที่
  `Countdown` ได้ในหัวข้อ 25.9) เพราะมันก็เป็นแค่ type หนึ่งที่ implement `Iterator` — ไม่สำคัญว่า logic
  ภายใน `next()` จะซับซ้อนแค่ไหน ตราบใดที่มันคืน `Option<Self::Item>` ที่ถูกต้อง มันได้สิทธิ์ใช้ default
  method ทั้งหมดเท่ากับ iterator ตัวอื่นทุกตัวใน std library

### 25.15 รายชื่อ method ที่ได้ "ฟรี" แบ่งตามหมวดหมู่: เห็นภาพว่า 70+ methods มาจากไหนบ้าง

หัวข้อ 25.9 พิสูจน์ด้วยโค้ดจริงไปแล้วว่า implement แค่ `next()` ทำให้ได้ method อื่นมาฟรี — ตารางนี้รวบรวม
method ที่พบบ่อยที่สุดในโค้ด Rust จริงไว้เป็นหมวดหมู่ เพื่อให้เห็นภาพรวมว่า "70+ methods" ที่พูดถึงตลอดบทนี้
ครอบคลุมอะไรบ้าง (ยังไม่ครบทั้งหมด — Part 26 จะพา deep dive ตัวที่ซับซ้อนกว่านี้อีกหลายตัว):

| หมวดหมู่ | ตัวอย่าง method | ประเภท | ทำอะไร |
|---|---|---|---|
| **รวมเป็นค่าเดียว** | `.sum()`, `.product()`, `.count()`, `.max()`, `.min()`, `.fold()`, `.reduce()` | consuming | ขับเคลื่อน iterator จนหมด คืนค่าสรุปค่าเดียว |
| **ค้นหา/ตรวจสอบ** | `.find()`, `.position()`, `.any()`, `.all()`, `.last()`, `.nth(n)` | consuming | ขับเคลื่อนจนกว่าจะเจอคำตอบ (บางตัวหยุดก่อนจบถ้าเจอคำตอบแล้ว — เรียกว่า short-circuit) |
| **รวบรวมเป็น collection** | `.collect()`, `.unzip()`, `.partition()` | consuming | รวบรวมทุกสมาชิกเข้า collection ปลายทางตาม `FromIterator` |
| **แปลง/กรอง** | `.map()`, `.filter()`, `.filter_map()`, `.flat_map()`, `.flatten()` | adaptor | คืน iterator ตัวใหม่ที่ให้ค่าต่างจากเดิม |
| **จัดลำดับ/ตำแหน่ง** | `.enumerate()`, `.rev()`, `.step_by(n)` | adaptor | คืน iterator ตัวใหม่ที่จัดลำดับ/นับตำแหน่งต่างจากเดิม |
| **ตัด/ต่อ** | `.take(n)`, `.skip(n)`, `.take_while()`, `.skip_while()`, `.chain()` | adaptor | คืน iterator ตัวใหม่ที่สั้นลง/ยาวขึ้น |
| **รวมหลาย iterator** | `.zip()`, `.chain()` | adaptor | คืน iterator ตัวใหม่ที่รวมสอง iterator เข้าด้วยกัน |
| **ดูล่วงหน้า/แชร์** | `.peekable()`, `.by_ref()`, `.cloned()`, `.copied()` | adaptor | ปรับพฤติกรรมการยืม/ดูค่าของ iterator โดยไม่เปลี่ยนลำดับข้อมูล |
| **ผลข้างเคียง** | `.for_each()`, `.inspect()` | consuming/adaptor* | รันโค้ดกับทุกสมาชิก (`.for_each()` consume ทั้งหมด, `.inspect()` แค่ "แอบดู" แล้วส่งค่าเดิมผ่านต่อ) |

(`.inspect()` มีเครื่องหมาย * เพราะเป็น iterator adaptor ที่ทำงานคล้าย `.map()` แต่ไม่แปลงค่า แค่รัน closure
แอบดูค่าระหว่างทางแล้วส่งค่าเดิมต่อไป มีประโยชน์มากตอน debug chain ยาว ๆ ว่าค่า ณ จุดกลาง ๆ เป็นอย่างไร — ใช้
หลักการ laziness เดียวกับที่เรียนในหัวข้อ 25.11 ทุกประการ)

Method ทั้งหมดในตารางนี้ (และอีกกว่า 50 ตัวที่ไม่ได้แสดง) มาจากที่เดียวกัน: **default implementation ของ
trait `Iterator` ที่เขียนโดยเรียก `self.next()` (หรือ method อื่นที่สร้างจาก `next()` อีกที) เท่านั้น** — ไม่มี
method ไหนใน std library ที่ "แอบรู้" โครงสร้างภายในของ `Countdown` หรือ `LowStock` ของเราเลยแม้แต่ตัวเดียว

### 25.16 มองใต้ฝากระโปรง: `.map()` ก็แค่ struct ธรรมดาที่ implement `Iterator`

ตอนนี้เราเขียน custom iterator ของตัวเองมาแล้วสองตัว (`Countdown`, `LowStock`) — คำถามที่น่าสนใจคือ:
`.map()` ตัวจริงของ std เองก็เขียนด้วยกลไกเดียวกันหรือไม่? คำตอบคือ **ใช่ทุกประการ** ลองเขียน `.map()`
เวอร์ชันของเราเอง (`my_map`) แล้วพิสูจน์ว่าให้ผลลัพธ์เหมือน `.map()` จริงของ std เป๊ะ:

```rust
struct MyMap<I, F> {
    iter: I,
    f: F,
}

impl<I, F, B> Iterator for MyMap<I, F>
where
    I: Iterator,
    F: FnMut(I::Item) -> B,
{
    type Item = B;

    fn next(&mut self) -> Option<B> {
        match self.iter.next() {
            Some(x) => Some((self.f)(x)),
            None => None,
        }
    }
}

fn my_map<I, F, B>(iter: I, f: F) -> MyMap<I, F>
where
    I: Iterator,
    F: FnMut(I::Item) -> B,
{
    MyMap { iter, f }
}

fn main() {
    let numbers = vec![1, 2, 3];
    let doubled: Vec<i32> = my_map(numbers.into_iter(), |n| n * 2).collect();
    println!("my_map: {:?}", doubled);

    // เทียบกับ .map() จริงของ std ว่าให้ผลลัพธ์เหมือนกันทุกประการ
    let doubled_std: Vec<i32> = vec![1, 2, 3].into_iter().map(|n| n * 2).collect();
    println!("std .map(): {:?}", doubled_std);
}
```

ผลลัพธ์:

```
my_map: [2, 4, 6]
std .map(): [2, 4, 6]
```

**อธิบายทีละส่วน โดยเชื่อมกับ Part 18/22 เรื่อง generics:**

- **`struct MyMap<I, F> { iter: I, f: F }`** — เก็บ "iterator ต้นทาง" (`I`) และ "closure ที่จะใช้แปลงค่า" (`F`)
  ไว้เป็น field ธรรมดา — นี่คือคำตอบของคำถาม "closure ที่ส่งเข้า `.map()` ไปอยู่ที่ไหน" มันถูกเก็บไว้เป็น field
  ของ struct ที่ `.map()` สร้างขึ้นมา ตรงกับ mental model ของ closure ที่ Part 24 สอนไว้ (closure คือ struct
  นิรนามที่เก็บตัวแปรที่ capture มาเป็น field) — `MyMap` เพียงแค่ "เก็บ" closure นั้นไว้ในตัวมันเองอย่างชัดเจน
- **`impl<I, F, B> Iterator for MyMap<I, F> where I: Iterator, F: FnMut(I::Item) -> B`** — bound `I: Iterator`
  (ต้นทางต้องเป็น iterator) และ `F: FnMut(I::Item) -> B` (closure ต้องรับค่าจาก `I` ได้และคืน `B`) ตามหลักการ
  `where` clause จาก Part 22 หัวข้อ 22.2-22.3 — สังเกตว่า `F: FnMut` (ไม่ใช่ `Fn`) เพราะ closure ที่ส่งเข้า
  `.map()` อาจมี state ภายในที่แก้ไขได้ระหว่างเรียกซ้ำ (เช่น นับจำนวนครั้งที่ถูกเรียก) ตามหลักการ Part 24
- **`fn next(&mut self) -> Option<B> { match self.iter.next() { ... } }`** — logic คือ: **เรียก `.next()` ของ
  iterator ต้นทางก่อน** ถ้าได้ `Some(x)` ก็ส่ง `x` ผ่าน closure `self.f` แล้วคืน `Some(ผลลัพธ์)` ถ้าต้นทางหมด
  แล้ว (`None`) ก็คืน `None` ทันที **ไม่เรียก closure เลย** — สังเกตว่านี่คือเหตุผลเชิงกลไกที่ทำให้ laziness
  (หัวข้อ 25.11) เป็นจริงได้: closure ข้างใน `.map()` ถูกเรียก**ก็ต่อเมื่อ**มีคนเรียก `.next()` ของ `MyMap`
  เท่านั้น ไม่มีทางถูกเรียกล่วงหน้าได้เลยไม่ว่ากรณีใด เพราะ logic ทั้งหมดอยู่ข้างใน `next()` ซึ่งไม่มีใครเรียก
  จนกว่าจะมี consuming adaptor มาขับเคลื่อน

**ทำไมเรื่องนี้เชื่อมกับ zero-cost abstraction**: เมื่อเขียน `v.iter().filter(p).map(f).take(3)` สิ่งที่
compiler เห็นจริง ๆ คือ type ที่ซ้อนกันหลายชั้น (`Take<Map<Filter<Iter<'_, T>, P>, F>>` — อ่านจากในสุดออกมา
นอกสุด) เมื่อ compiler ทำ **monomorphization** (สร้างโค้ดเฉพาะสำหรับ concrete type ที่ใช้จริง ตามหลักการที่
เรียนใน Part 18 เรื่อง generics) ให้กับ chain ทั้งก้อนนี้ มันสามารถ **inline** การเรียก `.next()` ที่ซ้อนกัน
หลายชั้นทั้งหมดเข้าเป็น loop เดียวที่ไม่มี function call overhead เหลืออยู่เลย — นี่คือกลไกจริงที่อยู่หลังคำว่า
"iterator fusion" ที่กล่าวถึงในหัวข้อ 25.11: มันไม่ใช่ "การปรับแต่งพิเศษ" ที่ทำเฉพาะกับ `Iterator` แต่เป็นผล
ธรรมชาติของการที่ generics ใน Rust ถูก monomorphize เป็นโค้ดที่ compiler มองเห็น type ที่แน่นอนทุกชั้นและ
optimize ได้เต็มที่ (ต่างจาก dynamic dispatch ผ่าน `dyn Trait` ที่เรียนใน Part 21 ซึ่ง compiler มองไม่ทะลุผ่าน
`vtable` ได้แบบนี้)

### 25.17 Iterator ที่ไม่มีที่สิ้นสุด (Infinite Iterators): `(1..)`, `std::iter::repeat`, และเหตุผลที่ `.take()` เป็นเพื่อนที่ต้องมีเสมอ

ไม่ใช่ทุก iterator ที่จะคืน `None` ในที่สุด — บางตัวถูกออกแบบมาให้ **ไม่มีวันคืน `None` เลย** โดยตั้งใจ ลองดู
ตัวอย่างที่ใช้บ่อยที่สุดสองแบบ:

```rust
fn main() {
    // (1..) คือ range ที่ไม่มีจุดสิ้นสุด (unbounded range) — ไม่มีวันคืน None
    let first_five_naturals: Vec<i32> = (1..).take(5).collect();
    println!("{:?}", first_five_naturals);

    // std::iter::repeat(x) คืนค่า x ตัวเดิมไปเรื่อย ๆ ไม่มีวันคืน None
    let repeated: Vec<&str> = std::iter::repeat("Rust").take(3).collect();
    println!("{:?}", repeated);

    // ผสม adaptor เข้ากับ infinite iterator: หาเลขกำลังสองตัวแรกที่มากกว่า 100
    let first_square_over_100: Option<i32> = (1..).map(|n| n * n).find(|&n| n > 100);
    println!("{:?}", first_square_over_100);
}
```

ผลลัพธ์:

```
[1, 2, 3, 4, 5]
["Rust", "Rust", "Rust"]
Some(121)
```

**อธิบาย**: `(1..)` คือ range ที่เขียนแบบไม่มีจุดสิ้นสุด (ต่างจาก `0..5` หรือ `0..=5` ที่เรียนใน Part 4 ที่มี
จุดสิ้นสุดชัดเจน) — `.next()` ของมันคืน `Some(1)`, `Some(2)`, `Some(3)`, ... **ไปตลอดกาลโดยไม่มีวันคืน `None`**
เช่นเดียวกับ `std::iter::repeat("Rust")` ที่คืน `Some("Rust")` ซ้ำไปเรื่อย ๆ ไม่มีวันหมด

**คำถามสำคัญ**: ถ้า `(1..).take(5)` ทำงานได้ปกติ (หยุดที่ 5 ตัว) แล้ว `.collect()` กับ iterator ที่ไม่มีที่
สิ้นสุดจะเกิดอะไรขึ้นถ้า**ไม่มี** `.take()` มาจำกัดไว้ก่อน? คำตอบคือ **โปรแกรมจะวนไม่มีที่สิ้นสุด (infinite
loop) จนกว่าหน่วยความจำจะเต็มหรือถูกบังคับหยุด** เพราะ `.collect()` (และ consuming adaptor ส่วนใหญ่) เขียนด้วย
`while let Some(x) = self.next() { ... }` ภายใน (ตามหลักการหัวข้อ 25.9) — ถ้า `next()` ไม่มีวันคืน `None`
loop ข้างในก็ไม่มีวันจบตามไปด้วย โค้ดต่อไปนี้ **ห้ามรันจริงเด็ดขาด** (แสดงไว้เพื่ออธิบายแนวคิดเท่านั้น
ไม่ใช่ตัวอย่างที่ควรทดลองรัน):

```
// อันตราย: ห้ามรันจริง — โปรแกรมจะวนไม่หยุดและกิน memory จนเต็มในที่สุด
let all_of_them: Vec<i32> = (1..).collect();
```

**หลักการที่ต้องจำ**: ทุกครั้งที่ทำงานกับ iterator ที่ไม่มีจุดสิ้นสุด (ไม่ว่าจะเป็น unbounded range,
`std::iter::repeat()`, หรือ custom iterator ที่เขียนเอง — เช่น `Fibonacci` ในแบบฝึกหัดข้อ 2 ท้ายบทนี้ ที่ไม่มี
เงื่อนไขให้คืน `None` เลย) **ต้องมี adaptor ที่จำกัดจำนวนเสมอ** ก่อนจะส่งเข้า consuming adaptor — ที่พบบ่อยที่สุด
คือ `.take(n)` (จำกัดจำนวนตายตัว) หรือ `.take_while(predicate)`/`.find(predicate)` (จำกัดด้วยเงื่อนไข หยุดทันที
ที่เจอคำตอบ — เรียกว่า **short-circuit**) `.find()` ในตัวอย่างข้างบนคือตัวอย่างที่ดี: มันขับเคลื่อน `(1..)`
ไปทีละตัวจนกว่าจะเจอเลขกำลังสองที่มากกว่า 100 (คือ 121 จาก 11²) แล้ว**หยุดทันที** ไม่ไปคำนวณเลขกำลังสองของ 12,
13, 14, ... ต่ออีกเลย แม้ในทางทฤษฎีจะทำได้ไปเรื่อย ๆ ก็ตาม

### 25.18 Marker Trait เพิ่มเติม: ทำไม `.rev()` ไม่ใช้ได้กับทุก iterator

หัวข้อ 25.13 กล่าวถึง `.rev()` ไว้สั้น ๆ ว่า "ต้องเป็น `DoubleEndedIterator`" — มาดูว่าทำไมถึงมีข้อจำกัดนี้ และ
เกิดอะไรขึ้นถ้าเรียก `.rev()` กับ iterator ที่ไม่รองรับ:

```rust
struct Countdown {
    current: u32,
}

impl Iterator for Countdown {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.current == 0 {
            None
        } else {
            self.current -= 1;
            Some(self.current + 1)
        }
    }
}

fn main() {
    let cd = Countdown { current: 5 };
    for n in cd.rev() {
        print!("{n} ");
    }
}
```

```
error[E0277]: the trait bound `Countdown: DoubleEndedIterator` is not satisfied
  --> src/main.rs:20:17
   |
20 |     for n in cd.rev() {
   |                 ^^^ unsatisfied trait bound
   |
help: the trait `DoubleEndedIterator` is not implemented for `Countdown`
note: required by a bound in `rev`
```

**เหตุผลเชิงลึก**: `Iterator` ธรรมดารู้จักแค่ "ทิศทางเดียว" — เรียก `.next()` แล้วเดินหน้าไปเรื่อย ๆ จากจุด
เริ่มไปจุดจบ มันไม่รู้ (และไม่มีวิธี) ที่จะบอกได้ว่า "จุดจบอยู่ที่ไหน" เพื่อเริ่มเดินย้อนจากด้านนั้นมาด้านนี้
`.rev()` ต้องการ **`DoubleEndedIterator`** ซึ่งเป็น trait ที่ต้องการ `Iterator` เป็น**supertrait** ก่อน (ตาม
หลักการ supertraits ที่เรียนใน Part 21 หัวข้อ 21.7) แล้วเพิ่ม method อีกตัวเข้ามา:

```
trait DoubleEndedIterator: Iterator {
    fn next_back(&mut self) -> Option<Self::Item>;
}
```

**`next_back()`** คือ "next() แต่ดึงจากด้านหลังมาด้านหน้าแทน" — เพื่อ implement `DoubleEndedIterator` ให้
`Countdown` ได้ เราต้องรู้วิธี "ดึงตัวสุดท้าย" ได้โดยไม่ต้องวนจากตัวแรกไปก่อน (เช่น slice `&[T]` รู้ตำแหน่งทั้ง
`start` และ `end` อยู่แล้วในตัว ทำให้เดินจากสองฝั่งมาพบกันตรงกลางได้ — นี่คือเหตุผลที่ `std::slice::Iter`
implement `DoubleEndedIterator` ได้ตามธรรมชาติ) แต่ `Countdown` ของเราเก็บสถานะแค่ `current` ตัวเดียวและ
เดินหน้าได้ทางเดียว (ลดค่าไปเรื่อย ๆ) — **ไม่มีข้อมูลพอที่จะบอกได้ว่า "ตัวสุดท้ายคืออะไร" โดยไม่ต้องวนจนจบ
ก่อน** จึง implement `DoubleEndedIterator` ให้ `Countdown` ไม่ได้อย่างมีความหมาย (จริง ๆ กรณีนี้ทำได้เพราะ
`Countdown` รู้ค่าเริ่มต้นเสมอ แต่ยกตัวอย่างไว้เพื่ออธิบายหลักการทั่วไปที่ iterator บางแบบ เช่น iterator ที่
อ่านข้อมูลจาก network stream แบบไหลเข้ามาทีละตัวไม่รู้จบล่วงหน้า จะไม่มีทาง implement `DoubleEndedIterator`
ได้เลยในทางเทคนิค)

**Marker trait อื่นที่ควรรู้จักชื่อไว้** (Part 26 จะเจาะลึกการ implement เองถ้าจำเป็น):

| Trait | เพิ่ม method อะไร | ใช้ทำอะไร |
|---|---|---|
| `DoubleEndedIterator: Iterator` | `next_back()` | ทำให้ `.rev()`, `.rfind()`, `.rposition()` ใช้ได้ |
| `ExactSizeIterator: Iterator` | `len()` | บอกจำนวนสมาชิกที่เหลือได้แน่นอนโดยไม่ต้องวนนับ (`.len()` เร็วกว่า `.count()` เพราะไม่ต้องขับเคลื่อน iterator จริง) |
| `FusedIterator: Iterator` | (ไม่มี method เพิ่ม แค่เป็น "สัญญา") | รับประกันว่าเมื่อ `next()` คืน `None` ครั้งหนึ่งแล้ว จะคืน `None` ตลอดไป (ตามที่กล่าวถึงในหัวข้อ 25.4) |

`Vec::iter()`, `Vec::into_iter()`, และ slice iterator ทั้งหมด implement ทั้งสามตัวนี้ครบ (เพราะรู้ทั้งจุดเริ่ม
และจุดจบตั้งแต่แรกอยู่แล้ว จากโครงสร้าง pointer + length ที่เรียนใน Part 13) แต่ iterator ที่ออกแบบมาให้ "ไหล
มาทีละตัวไม่รู้จบล่วงหน้า" อย่าง `(1..)` หรือ `std::iter::repeat()` implement แค่ `Iterator` เท่านั้น (ไม่มี
`ExactSizeIterator` เพราะไม่รู้ความยาวล่วงหน้า และไม่มี `DoubleEndedIterator` เพราะไม่มี "ด้านหลัง" ให้เดินย้อน)

**ลอง implement `DoubleEndedIterator` ให้ custom iterator ของเราเองดูจริง ๆ** — คราวนี้ใช้ `Steps` ที่เก็บทั้ง
`front` (ตำแหน่งเริ่ม) และ `back` (ตำแหน่งจบ) สองค่า ทำให้รู้ "ด้านหลัง" ได้จริงโดยไม่ต้องวนจนจบก่อน:

```rust
struct Steps {
    front: u32,
    back: u32,
}

impl Steps {
    fn new(start: u32, end: u32) -> Steps {
        Steps { front: start, back: end }
    }
}

impl Iterator for Steps {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.front >= self.back {
            None
        } else {
            let value = self.front;
            self.front += 1;
            Some(value)
        }
    }
}

impl DoubleEndedIterator for Steps {
    fn next_back(&mut self) -> Option<u32> {
        if self.front >= self.back {
            None
        } else {
            self.back -= 1;
            Some(self.back)
        }
    }
}

fn main() {
    let forward: Vec<u32> = Steps::new(0, 5).collect();
    println!("forward: {:?}", forward);

    let backward: Vec<u32> = Steps::new(0, 5).rev().collect();
    println!("backward (.rev()): {:?}", backward);

    // ผสมทั้งสองทิศทาง: .next() จากหน้า สลับกับ .next_back() จากหลัง
    let mut it = Steps::new(0, 6);
    println!(
        "สลับหน้า-หลัง: {:?} {:?} {:?} {:?}",
        it.next(),
        it.next_back(),
        it.next(),
        it.next_back()
    );
}
```

ผลลัพธ์:

```
forward: [0, 1, 2, 3, 4]
backward (.rev()): [4, 3, 2, 1, 0]
สลับหน้า-หลัง: Some(0) Some(5) Some(1) Some(4)
```

**อธิบายกลไก**: `Steps` เก็บสถานะสองตัว — `front` เดินหน้าเข้ามาจากด้านซ้าย, `back` เดินหน้าเข้ามาจากด้านขวา
(ลดค่าลงเรื่อย ๆ) `next()` ใช้ `front` เพิ่มค่าไปข้างหน้า ส่วน `next_back()` ใช้ `back` ลดค่าลงมา — ทั้งสอง
method เช็คเงื่อนไขเดียวกัน (`front >= back` แปลว่า "สองด้านมาชนกันแล้ว ไม่มีสมาชิกเหลือ") ทำให้เรียกผสมกันได้
อย่างปลอดภัย (บรรทัดสุดท้ายของ `main` แสดงให้เห็นว่าเรียก `.next()` และ `.next_back()` สลับกันได้โดยไม่พังหรือ
นับสมาชิกซ้ำ) — `.rev()` ที่ std ให้มาฟรีจากการ implement `DoubleEndedIterator` เพียงเรียก `next_back()` แทน
`next()` ไปเรื่อย ๆ เท่านั้นเอง (เหมือนที่ `.rev()` ของ `Vec::iter()` ทำงานอยู่ข้างในจริง ๆ)

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เรียก `.next()` โดยไม่ประกาศตัวแปร iterator ด้วย `mut` — E0596

```rust
fn main() {
    let v = vec![1, 2, 3];
    let iter = v.iter();
    println!("{:?}", iter.next());
}
```

```
error[E0596]: cannot borrow `iter` as mutable, as it is not declared as mutable
 --> src/main.rs:4:22
  |
4 |     println!("{:?}", iter.next());
  |                      ^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
3 |     let mut iter = v.iter();
  |         +++
```

**เหตุผล**: `next(&mut self)` ต้องการยืมแบบแก้ไขได้ตัว `iter` เอง เพราะการเรียกแต่ละครั้งต้องแก้ไขสถานะภายใน
(เลื่อนตำแหน่งไปข้างหน้า) — ตัวแปรที่จะเรียก `.next()` ตรง ๆ (ไม่ผ่าน `for`) **ต้องประกาศด้วย `mut` เสมอ**
วิธีแก้ตามที่ error แนะนำตรง ๆ: เติม `mut` หน้า `let iter`

### 2. `.filter()` บน `.iter()` ต้องใช้ pattern สองชั้น (`&&n`) — ลืมแล้ว compile ไม่ผ่าน

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let evens: Vec<&i32> = numbers.iter().filter(|n| n % 2 == 0).collect();
    println!("{:?}", evens);
}
```

```
error[E0369]: cannot calculate the remainder of `&&{integer}` divided by `{integer}`
 --> src/main.rs:3:56
  |
3 |     let evens: Vec<&i32> = numbers.iter().filter(|n| n % 2 == 0).collect();
  |                                                      - ^ - {integer}
  |                                                      |
  |                                                      &&{integer}
  |
help: `%` can be used on `&{integer}` if you dereference the left-hand side
  |
3 |     let evens: Vec<&i32> = numbers.iter().filter(|n| *n % 2 == 0).collect();
  |                                                      +
```

**เหตุผลเชิงลึก**: `numbers.iter()` ให้ `Item = &i32` — สัญญาของ `.filter()` คือรับ closure แบบ
`FnMut(&Self::Item) -> bool` (สังเกต `&` เพิ่มมาอีกชั้นหน้า `Self::Item` — เพราะ `.filter()` ต้อง**ยืม**ค่าไป
เช็คเงื่อนไขโดยไม่ยึด ownership ค่านั้น ถ้าเงื่อนไขไม่ผ่านจะได้คืนค่าเดิมกลับเข้า stream ต่อได้) เมื่อ
`Self::Item` คือ `&i32` อยู่แล้ว พารามิเตอร์ของ closure จึงกลายเป็น `&&i32` (reference ซ้อน reference) การ
เขียน `|n| n % 2 == 0` ทำให้ `n` มี type เป็น `&&i32` ซึ่งไม่มี `Rem` (การหารเอาเศษ, `%`) implement ไว้ให้
โดยตรง (std implement `Rem` ให้ `i32` และ `&i32` แค่ชั้นเดียว ไม่ถึง `&&i32`)

**สามวิธีแก้ที่ใช้บ่อยที่สุด** (ทั้งหมดให้ผลเหมือนกัน เลือกอ่านง่ายที่สุดตามสถานการณ์):

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    // วิธี 1: dereference สองชั้นในเงื่อนไข
    let a: Vec<&i32> = numbers.iter().filter(|n| **n % 2 == 0).collect();

    // วิธี 2: destructure หนึ่งชั้นด้วย pattern &n (เหลือ n: &i32)
    let b: Vec<&i32> = numbers.iter().filter(|&n| n % 2 == 0).collect();

    // วิธี 3: destructure สองชั้นด้วย pattern &&n (เหลือ n: i32 ตรง ๆ)
    let c: Vec<&i32> = numbers.iter().filter(|&&n| n % 2 == 0).collect();

    println!("{:?} {:?} {:?}", a, b, c);
}
```

**หลักการเลือก**: วิธี 3 (`|&&n|`) อ่านง่ายที่สุดเมื่อ `T` เป็น `Copy` (เช่น `i32`) เพราะทำให้ `n` เป็นค่าตรง ๆ
ไม่ต้อง dereference อีกในเงื่อนไข — นี่คือ pattern ที่เจอบ่อยที่สุดในโค้ด Rust จริงเวลา `.filter()` ต่อจาก
`.iter()` ของ collection ตัวเลข

### 3. สร้าง iterator adaptor แล้วไม่เก็บ/ไม่ consume — เตือนว่า "iterators are lazy"

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    numbers.iter().map(|n| n * 2);
    println!("จบโปรแกรม");
}
```

```
warning: unused `Map` that must be used
 --> src/main.rs:3:5
  |
3 |     numbers.iter().map(|n| n * 2);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: iterators are lazy and do nothing unless consumed
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
help: use `let _ = ...` to ignore the resulting value
  |
3 |     let _ = numbers.iter().map(|n| n * 2);
  |     +++++++
```

**เหตุผล**: ตามหลักการ laziness จากหัวข้อ 25.11 — บรรทัด `numbers.iter().map(|n| n * 2);` **ไม่ทำอะไรเลย**
มันสร้าง struct `Map` ขึ้นมาแล้วทิ้งไปทันทีโดยไม่มีใครเรียก `.next()` เลยแม้แต่ครั้งเดียว compiler รู้ว่านี่
มักเป็นความผิดพลาดของผู้เขียนโค้ด (ตั้งใจจะทำอะไรสักอย่างกับข้อมูล แต่ลืมต่อ `.collect()`/`for` เข้าไป) จึง
ทำเครื่องหมาย type ของ adaptor เหล่านี้ด้วย `#[must_use]` (เรียนแนวคิด attribute ไว้บ้างแล้วตั้งแต่
`#[derive(Debug)]` ใน Part 9) เพื่อเตือนตอน compile — ข้อความ `note: iterators are lazy and do nothing
unless consumed` คือคำอธิบายที่ตรงประเด็นที่สุด **ยืนยันสิ่งที่หัวข้อ 25.11 พิสูจน์ไว้ด้วยโค้ดจริง**

**วิธีแก้**: ถ้าตั้งใจทิ้งผลลัพธ์จริง ๆ (แทบไม่มีเหตุผลให้ทำแบบนี้ในทางปฏิบัติ) ใช้ `let _ = ...` ตามที่แนะนำ
แต่ในกรณีทั่วไป **นี่คือสัญญาณว่าลืมต่อ `.collect()` หรือ `for` เข้าไปจริง ๆ** ควรกลับไปดูว่าตั้งใจจะทำอะไรกับ
ผลลัพธ์ที่ควรได้

### 4. ลืม dereference ใน `.map()` ทำให้ type ไม่ตรงกับที่ `.collect()` ต้องการ — E0277

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let doubled: Vec<i32> = numbers.iter().map(|n| n).collect();
    println!("{:?}", doubled);
}
```

```
error[E0277]: a value of type `Vec<i32>` cannot be built from an iterator over elements of type `&{integer}`
 --> src/main.rs:3:55
  |
3 |     let doubled: Vec<i32> = numbers.iter().map(|n| n).collect();
  |                                                       ^^^^^^^ value of type `Vec<i32>` cannot be built from `std::iter::Iterator<Item=&{integer}>`
  |
help: the trait `FromIterator<&{integer}>` is not implemented for `Vec<i32>`
      but trait `FromIterator<i32>` is implemented for it
  = help: for that trait implementation, expected `i32`, found `&{integer}`
note: the method call chain might not have had the expected associated types
 --> src/main.rs:3:37
  |
2 |     let numbers = vec![1, 2, 3];
  |                   ------------- this expression has type `Vec<{integer}>`
3 |     let doubled: Vec<i32> = numbers.iter().map(|n| n).collect();
  |                                     ^^^^^^ ---------- `Iterator::Item` remains `&{integer}` here
  |                                     |
  |                                     `Iterator::Item` is `&{integer}` here
```

**เหตุผล**: `.map(|n| n)` แค่คืนค่าเดิมกลับไปโดยไม่ได้แปลงอะไรเลย — เพราะ `numbers.iter()` ให้ `Item = &i32`
closure `|n| n` จึงคืน `&i32` เหมือนเดิม แต่ตัวแปร `doubled` ระบุ type ไว้เป็น `Vec<i32>` (ค่าตรง ๆ ไม่ใช่
reference) — `Vec<i32>` implement `FromIterator<i32>` แต่**ไม่** implement `FromIterator<&i32>` ทำให้
`.collect()` หา implementation ที่ตรงกันไม่ได้ error message บอกตรงจุดชัดเจนมากว่า `Iterator::Item remains
&{integer} here` — ชี้ตรงไปที่ `.map()` ว่าเป็นจุดที่ type ยังไม่ถูกแปลง

**วิธีแก้**: เติม dereference ใน closure ให้คืนค่าจริงแทน reference:

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let doubled: Vec<i32> = numbers.iter().map(|n| *n).collect();
    println!("{:?}", doubled);
}
```

(หรือใช้ `numbers.into_iter()` ถ้าไม่ต้องใช้ `numbers` ต่อหลังจากนี้ จะได้ `Item = i32` ตรง ๆ โดยไม่ต้อง
dereference เลย)

### 5. เรียก `.into_iter()` ตรง ๆ แล้วพยายามใช้ตัวแปรเดิมต่อ — E0382

```rust
fn main() {
    let names = vec![String::from("A"), String::from("B")];
    let mut iter = names.into_iter();
    println!("{:?}", iter.next());
    println!("{}", names.len());
}
```

```
error[E0382]: borrow of moved value: `names`
 --> src/main.rs:5:20
  |
2 |     let names = vec![String::from("A"), String::from("B")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     let mut iter = names.into_iter();
  |                          ----------- `names` moved due to this method call
4 |     println!("{:?}", iter.next());
5 |     println!("{}", names.len());
  |                    ^^^^^ value borrowed here after move
  |
note: `into_iter` takes ownership of the receiver `self`, which moves `names`
help: you can `clone` the value and consume it, but this might not be your desired behavior
  |
3 |     let mut iter = names.clone().into_iter();
  |                         ++++++++
```

**เหตุผล**: ตอนนี้เรารู้แล้วจากหัวข้อ 25.6 ว่า `.into_iter()` รับ `self` แบบไม่มี `&` เลย (`fn into_iter(self)
-> Self::IntoIter`) — เท่ากับว่าเรียก `.into_iter()` ตรง ๆ กับ `names` ก็เท่ากับ **move `names` เข้าไปเป็นของ
`iter` ทั้งก้อน** ตามกฎ ownership จาก Part 6 ทันที ข้อความ error ที่ว่า `` `into_iter` takes ownership of the
receiver `self`, which moves `names` `` คือคำอธิบายที่ตรงประเด็นที่สุด และเหมือนกันในหลักการกับ error ที่
Part 13 หัวข้อ 13.6 แสดงไว้ตอน `for name in names` (ซึ่งก็เรียก `.into_iter()` เบื้องหลังเหมือนกันทุกประการ
ตามที่หัวข้อ 25.5 พิสูจน์ไว้) — **เขียน `.into_iter()` ตรง ๆ เอง ก็เจอปัญหาเดียวกันกับที่ `for` ทำให้เกิดขึ้น
โดยไม่รู้ตัว เพราะมันคือ method เดียวกันเป๊ะ**

**วิธีแก้**: ถ้ายังต้องใช้ `names` ต่อหลังจากนี้ ให้เปลี่ยนไปใช้ `.iter()` (ยืมแทนยึด ownership):

```rust
fn main() {
    let names = vec![String::from("A"), String::from("B")];
    let mut iter = names.iter();
    println!("{:?}", iter.next());
    println!("{}", names.len()); // ใช้ names ต่อได้ปกติ
}
```

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เขียนฟังก์ชัน `fn even_squares(numbers: &[i32]) -> Vec<i32>` ที่ใช้ iterator chain
   (`.iter().filter(...).map(...).collect()`) กรองเอาแต่เลขคู่จาก `numbers` แล้วยกกำลังสองแต่ละตัว คืนเป็น
   `Vec<i32>` ทดสอบด้วย `even_squares(&[1, 2, 3, 4, 5, 6])` ควรได้ `[4, 16, 36]` (hint: ต้อง dereference ใน
   `.filter()` และ `.map()` ตามหลักการกับดักที่ 2 และ 4 ของบทนี้ — ลองเขียนโดยใช้ pattern `|&n|` ทั้งสองที่)

2. **[กลาง]** เขียน custom iterator ชื่อ `Fibonacci` ที่เก็บสถานะภายในเป็นตัวเลขสองตัว (เช่น `current` และ
   `next`) แล้ว implement `Iterator` ให้มัน (`type Item = u64;`) โดย `next()` คืนค่า Fibonacci ตัวถัดไปไป
   เรื่อย ๆ ไม่มีที่สิ้นสุด (`Some` เสมอ ไม่มี `None`) ทดสอบด้วย `Fibonacci::new().take(10).collect::<Vec<u64>>()`
   ควรได้ `[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]` (hint: ระวังอย่าใช้ `for x in Fibonacci::new() { ... }` แบบไม่มี
   `.take()` เด็ดขาด — เพราะ iterator นี้ไม่มีวันคืน `None` โปรแกรมจะวนไม่หยุด ต้องมี consuming adaptor ที่
   จำกัดจำนวนเสมอสำหรับ iterator แบบ infinite เช่นนี้)

3. **[ยาก]** โค้ดต่อไปนี้ compile ไม่ผ่าน:
   ```
   fn parse_numbers(input: &[&str]) -> Vec<i32> {
       input.iter().map(|s| s.parse().unwrap()).collect()
   }
   ```
   ลอง compile ดูจริง ๆ แล้วอ่าน error ที่ได้ (เป็นตัวแปรของปัญหา "type annotations needed" ที่เรียนในหัวข้อ
   25.12 แต่เกิดจาก `.parse()` แทน `.collect()` โดยตรง) แก้ไขได้ 2 วิธี: (ก) ระบุ return type ให้ชัดเจนกว่านี้
   ผ่าน turbofish บน `.parse::<i32>()`, หรือ (ข) เปลี่ยนวิธีเขียนให้ compiler infer ได้จาก return type ของ
   ฟังก์ชันเอง ลองทำทั้งสองวิธีแล้วสังเกตว่า error message พูดถึง "generic parameter ที่ยังหาค่าไม่ได้" ต่างจาก
   E0283 ของ `.collect()` อย่างไร (hint: `.parse()` ก็เป็น generic method ที่มี type parameter ของ method เอง
   เหมือน `.collect()` — ปัญหาเดียวกันเกิดขึ้นได้กับ method อื่นที่ไม่ใช่ `.collect()` ด้วย ไม่ใช่ปัญหาเฉพาะ
   `.collect()` เท่านั้น)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ขยายตัวอย่างระบบสินค้าคงคลังในหัวข้อ 25.14: เขียน custom iterator ชื่อ
   `PriceRange<'a>` ที่รับ `&'a [Product]` พร้อม `min: f64` และ `max: f64` แล้วคืนเฉพาะสินค้าที่ราคาอยู่ใน
   ช่วงนั้น (`type Item = &'a Product;` เหมือน `LowStock`) จากนั้นเขียนฟังก์ชัน
   `fn total_value_in_range(products: &[Product], min: f64, max: f64) -> f64` ที่ใช้
   `PriceRange::new(products, min, max)` ร่วมกับ `.map()` และ `.sum()` (ที่ได้มาฟรีจากการ implement
   `Iterator`) คำนวณมูลค่ารวม (`price * quantity`) ของสินค้าทั้งหมดที่อยู่ในช่วงราคานั้น โดย**ไม่ต้อง**เขียน
   loop ธรรมดาเลยแม้แต่บรรทัดเดียวในฟังก์ชันนี้ — ทดสอบกับข้อมูลจากหัวข้อ 25.14 แล้วเทียบผลลัพธ์กับการคำนวณ
   ด้วยมือว่าตรงกัน (hint: `PriceRange` กับ `LowStock` มีโครงสร้างเกือบเหมือนกันทุกประการ ต่างกันแค่เงื่อนไข
   ข้างใน `next()` — ลองสังเกตว่า pattern "struct เก็บ slice + index + เงื่อนไข, `next()` วน `while` หา
   ตัวถัดไปที่ผ่านเงื่อนไข" คือ template ที่ใช้ซ้ำได้กับ custom iterator แบบ "filter เฉพาะทาง" ได้เกือบทุกแบบ)

## สรุป

บทนี้ไขทุกทีเซอร์เรื่อง iterator ที่หลักสูตรค้างไว้ตั้งแต่ Part 4 ให้กระจ่างครบถ้วน เริ่มจากนิยามจริงของ trait
`Iterator` (`type Item;` + `fn next(&mut self) -> Option<Self::Item>;`) และเชื่อมกับ Part 22 ว่าทำไมต้องใช้
associated type ไม่ใช่ generic parameter — เพราะ "iterator หนึ่งตัวต้องให้ค่าชนิดเดียวเสมอ" จากนั้นอธิบายว่า
ทำไม `next()` คืน `Option<Self::Item>` ด้วยปรัชญาเดียวกับ `Option<T>` ที่เรียนใน Part 11: ไม่ต้องมี sentinel
value, ไม่ต้องมี exception, แค่ enum ธรรมดาที่ compiler บังคับให้จัดการทั้งสองกรณี

จุดที่สำคัญที่สุดของบทคือการ desugar `for` loop ให้เห็นว่าแท้จริงมันคือ `while let Some(x) =
IntoIterator::into_iter(collection).next()` วนซ้ำ — ไม่มีเวทมนตร์ซ่อนอยู่เลย พร้อมอธิบาย `IntoIterator` (trait
ที่แปลง type เป็น iterator ได้) แยกจาก `Iterator` (trait ที่ตัวมันเองเป็น iterator อยู่แล้ว) ให้ชัดเจน และปิด
ทีเซอร์ของ Part 13 อย่างสมบูรณ์: `for x in &v`, `for x in &mut v`, `for x in v` แต่ละแบบเรียก `IntoIterator`
implementation คนละตัวของ `Vec<T>` ให้ `Item` เป็น `&T`, `&mut T`, `T` ตามลำดับ

ครึ่งหลังของบทพาไปเขียน custom iterator ของตัวเอง (`Countdown`, `LowStock`, `Steps`) พิสูจน์ว่าการ implement
แค่ `next()` ตัวเดียวทำให้ได้ default method กว่า 70 ตัวมาฟรี (`.sum()`, `.count()`, `.max()`, `.collect()`,
ฯลฯ) แล้วแบ่งประเภท method เหล่านั้นเป็น **consuming adaptor** (ขับเคลื่อน iterator จนได้ค่าสุดท้าย) กับ
**iterator adaptor** (ห่อ iterator เดิมด้วย logic ใหม่ คืน iterator ตัวใหม่ที่ยัง chain ต่อได้) พร้อมพิสูจน์
ด้วยโค้ดจริงว่า iterator adaptor **lazy** — ไม่ทำงานอะไรเลยจนกว่าจะถูกขับเคลื่อน ซึ่งเป็นกลไกที่ทำให้ chain
ยาว ๆ มีประสิทธิภาพเทียบเท่า loop มือเปล่าผ่าน iterator fusion (พิสูจน์เพิ่มด้วยการเขียน `.map()` เวอร์ชัน
ของตัวเอง `MyMap` ให้เห็นว่ามันก็แค่ struct ธรรมดาที่เก็บ closure ไว้เป็น field) ปิดท้ายด้วย `.collect()`
แบบเจาะลึก (ทำไมต้องบอก type ปลายทาง, สองวิธีแก้ error "type annotations needed") และ adaptor พื้นฐาน
(`.filter()`, `.map()`, `.enumerate()`, `.zip()`, `.rev()`, `.take()`, `.skip()`, `.chain()`) ก่อนขยายผลไปสู่
iterator ที่ไม่มีที่สิ้นสุด (`(1..)`, `std::iter::repeat()`) กับข้อควรระวังเรื่อง `.take()`/`.find()` ที่ต้องมี
เสมอ และปิดเนื้อหาด้วย marker trait เพิ่มเติม (`DoubleEndedIterator`, `ExactSizeIterator`, `FusedIterator`)
พร้อม implement `DoubleEndedIterator` ให้ custom iterator ของตัวเองจริง ๆ (`Steps`) ก่อนนำทุกแนวคิดมาผสมกันใน
ตัวอย่างระบบสินค้าคงคลังท้ายบท

ใน **Part 26** เราจะเจาะลึก iterator ต่อไปอีกขั้น: adaptor ที่ซับซ้อนกว่านี้ (`.flat_map()`, `.scan()`,
`.peekable()`, `.by_ref()`, `.step_by()`), เขียน custom iterator ที่มี state ซับซ้อนกว่านี้ (รวม generic
iterator ที่ทำงานกับหลาย `Item` type), และวิเคราะห์ performance ของ iterator เทียบกับ loop มือเปล่าอย่าง
ละเอียดกว่านี้ด้วยเครื่องมือ benchmark จริง — ทุกอย่างที่เรียนในบทนี้ (นิยาม `Iterator`/`IntoIterator`,
laziness, consuming vs iterator adaptor, marker trait) จะเป็นฐานที่ Part 26 ต่อยอดโดยตรงทุกจุด

---

**Part ก่อนหน้า:** [Closures](part-024-closures.md) | **Part ถัดไป:** [Iterators ขั้นสูง](part-026-iterators-advanced.md)
