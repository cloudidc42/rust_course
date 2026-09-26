# Part 49: Tokio: I/O และ Networking (TCP/UDP)

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไม **networking คือเหตุผลตัวจริง** ที่ทำให้ async/await และ Tokio ถือกำเนิดขึ้นในโลก Rust ระดับ
  production ไม่ใช่แค่ "อีกวิธีเขียนโค้ด" — และเชื่อมโยงกลับไปที่ concept เรื่อง task และ `tokio::spawn` จาก Part 48
  ว่าทำไม "หนึ่ง connection ต่อหนึ่ง task" ถึงเป็นรูปแบบการออกแบบเซิร์ฟเวอร์ที่ใช้กันจริงในระดับ production
- ใช้ `tokio::fs::File` คู่กับ trait `AsyncReadExt`/`AsyncWriteExt` เขียนและอ่านไฟล์แบบ async ได้อย่างถูกต้อง และ
  มองเห็นว่ารูปแบบ "extension trait ที่เติม method `.read()`/`.write_all()` เข้าไปในชนิดข้อมูล" นี้จะถูกใช้ซ้ำเป๊ะ ๆ
  ตอนทำงานกับ TCP socket ในหัวข้อต่อ ๆ ไป
- สร้าง TCP server ด้วย `tokio::net::TcpListener` ที่ `.accept()` connection เข้ามาแบบ async แล้ว `tokio::spawn`
  task ใหม่แยกกันสำหรับทุก client โดยไม่ block thread หลักเลย และสร้าง TCP client ด้วย `tokio::net::TcpStream`
  ที่คุยกับ server ตัวนั้นได้จริงผ่าน localhost
- อธิบายปัญหา **message framing** ได้อย่างลึกซึ้ง (ทำไม TCP byte stream ไม่มีขอบเขตข้อความในตัวเอง, หนึ่งข้อความอาจถูก
  แยกเป็นหลาย `.read()` หรือหลายข้อความอาจถูกรวมมาใน `.read()` เดียว) และแก้ปัญหาด้วยรูปแบบ newline-delimited message
  ผ่าน `AsyncBufReadExt::lines()`
- ขยาย server ให้รองรับ**หลาย client พร้อมกันจริง ๆ** และออกแบบระบบ **broadcast ข้อความไปยังทุก client** ด้วยการเชื่อม
  ความรู้เรื่อง `Arc<Mutex<T>>` จาก Part 39 เข้ากับ channel เพื่อแก้ปัญหา shared state ข้าม task
- ใช้ `tokio::net::UdpSocket` ส่ง/รับข้อมูลแบบ connectionless ได้ และอธิบายได้ว่าเมื่อไหร่ควรเลือก UDP แทน TCP โดยเข้าใจ
  ข้อแลกเปลี่ยน (trade-off) ระหว่างความน่าเชื่อถือกับความหน่วง (latency)
- ครอบ network operation ด้วย `tokio::time::timeout()` เพื่อป้องกันโปรแกรมค้างตลอดกาลเมื่อฝั่งตรงข้ามไม่ตอบสนอง และ
  มองเห็นว่ามันคือ "syntax สะดวก" ที่สร้างจากกลไกเดียวกันกับ `select!` ที่เรียนใน Part 48

## ความรู้ที่ต้องมีมาก่อน

- **Part 48 (Tokio: Runtime และ Tasks)**: บทนี้ต่อยอดโดยตรงจาก Part 48 แบบไม่มีการทวนซ้ำตั้งแต่ต้น เราจะสมมติว่าคุณ
  รู้จัก `#[tokio::main]`, เข้าใจว่า `tokio::spawn` สร้าง task ที่เป็น "green thread" น้ำหนักเบาที่ executor สามารถรัน
  พร้อมกันได้หลายพันตัวบน thread ของ OS จำนวนน้อย, รู้จัก `.await` point คือจุดที่ task อาจถูก "สลับ" ให้ task อื่นทำงาน
  แทนตอนที่ future ยังไม่พร้อม, และรู้จัก `tokio::join!`/`tokio::select!` เบื้องต้น — เนื้อหาบทนี้คือ "เอา primitive
  เหล่านั้นไปใช้งานกับ I/O จริง" โดยตรง โดยเฉพาะ **`tokio::spawn` แบบหนึ่ง task ต่อหนึ่ง connection** ซึ่งคือ use case
  ในโลกจริงที่สำคัญที่สุดของสิ่งที่ Part 48 สอนไว้
- **Part 46 (Async/Await เบื้องต้น)** และ **Part 47 (Futures และ Executors)**: เข้าใจว่า `async fn` คืนค่าเป็นชนิดที่
  implement trait `Future`, เข้าใจว่า future ไม่ทำอะไรเลยจนกว่าจะถูก `.await` หรือ poll (เราจะเจอ pitfall ที่มาจากการ
  ลืมความจริงข้อนี้ในหัวข้อกับดักท้ายบท), และเข้าใจภาพกว้างว่า runtime/executor คือตัวที่ poll future เหล่านี้ไปเรื่อย ๆ
  จนเสร็จ
- **Part 12 (Result\<T, E\> และ Error Handling เบื้องต้น)**: network I/O ทุกชนิดใน Tokio คืนค่าเป็น `std::io::Result<T>`
  (คือ `Result<T, std::io::Error>`) เหมือนกับ `std::fs::File` ที่เรียนใน Part 12 ทุกประการ — เราจะใช้ `?` operator และ
  `match` กับ `io::Result` ตลอดทั้งบทโดยไม่อธิบายกลไก `Result` ซ้ำจากศูนย์
- **Part 14 (Collections: String และการจัดการข้อความ UTF-8)**: เนื้อหาเรื่อง byte stream และ `&[u8]` ในบทนี้ใช้ความรู้
  พื้นฐานเรื่อง `String` คือ `Vec<u8>` ที่การันตี UTF-8 จาก Part 14 โดยตรง โดยเฉพาะตอนแปลง bytes ที่รับจาก socket กลับ
  เป็นข้อความด้วย `String::from_utf8_lossy()` ที่ Part 14 อธิบายไว้แล้วว่าทำไมต้องมี `_lossy` และทำไมข้อมูลจาก
  network (ซึ่งเป็น bytes ดิบ ที่ไม่มีการันตีอะไรเลย) มีความเสี่ยงสูงกว่าข้อความที่สร้างขึ้นในโปรแกรมเองมาก
- **Part 39 (Mutex, Arc และ Shared-State Concurrency)**: หัวข้อ broadcast ในบทนี้ใช้ `Arc<Mutex<T>>` เพื่อแบ่งปัน
  รายชื่อ client ระหว่างหลาย task ตรงกับรูปแบบที่ Part 39 สอนไว้เป๊ะ ๆ เพียงแต่คราวนี้ "งาน" ที่แบ่งปันข้อมูลกันคือ
  async task ของ Tokio แทน thread ของ OS ธรรมดา — เราจะเห็นว่าความรู้เรื่อง `Arc<Mutex<T>>` ที่เรียนไว้ **ใช้ได้เหมือนกัน
  ทุกประการ** ไม่ต้องเรียนใหม่ทั้งหมด แต่มีข้อควรระวังพิเศษหนึ่งข้อที่เกี่ยวกับการถือ `MutexGuard` ข้าม `.await` ซึ่งจะ
  อธิบายในหัวข้อกับดัก
- **Part 38 (Channels)**: แนวคิด "ส่งข้อความข้าม task ผ่าน channel แทนการแชร์ memory ตรง ๆ" จาก Part 38 (ที่ใช้
  `std::sync::mpsc`) จะกลับมาอีกครั้งในหัวข้อ broadcast แต่คราวนี้ใช้เวอร์ชัน async คือ `tokio::sync::mpsc` — บทนี้จะ
  ใช้มันแบบผิวเผินพอให้ตัวอย่าง broadcast ทำงานได้จริงเท่านั้น ส่วนรายละเอียดเชิงลึกของ channel ฝั่ง Tokio ทั้งหมด
  (`mpsc`, `oneshot`, `broadcast`, `watch`, `Mutex`/`RwLock` แบบ async) จะเรียนอย่างเป็นทางการใน **Part 50**

ถ้าคุณยังไม่แน่ใจเรื่อง `tokio::spawn` ทำงานยังไงข้างใต้ หรือทำไม async task ถึง "เบา" กว่า OS thread มาก แนะนำให้
ย้อนอ่าน Part 48 อีกครั้งก่อน เพราะบทนี้จะเรียก `tokio::spawn` ซ้ำ ๆ ในทุกตัวอย่าง TCP server โดยไม่อธิบายพื้นฐานของมัน
ใหม่เลย

## เนื้อหา

### 49.1 ทำไม async ถึงถูกสร้างมาเพื่อสิ่งนี้: I/O-bound workloads และ Networking

Part 46 เริ่มต้นด้วยการอธิบาย async/await ในภาพกว้าง Part 47 เจาะลึกกลไก `Future`/executor ข้างใน และ Part 48 สอนวิธี
ใช้ Tokio runtime สร้างและจัดการ task — แต่ทั้งสามบทนั้นล้วนใช้ตัวอย่างที่ **จำลอง** งาน I/O ด้วย `tokio::time::sleep()`
เพราะโฟกัสของบทเหล่านั้นคือกลไก concurrency เอง ไม่ใช่ตัว I/O จริง คำถามที่ค้างอยู่คือ: **ในโลกจริง งานแบบไหนที่ทำให้
async คุ้มค่าคุ้มความซับซ้อนที่ต้องแลกมา?**

คำตอบสั้น ๆ คือ **network I/O** เกือบทั้งหมด ลองนึกภาพเซิร์ฟเวอร์ที่ต้องรองรับ 10,000 client เชื่อมต่อพร้อมกัน โดยแต่ละ
client ส่งข้อมูลมาเป็นครั้ง ๆ ห่างกันหลายวินาที (เช่น chat application, IoT sensor ที่ report สถานะทุก 5 วินาที, หรือ
web server ที่รอ request จาก browser) — ถ้าเซิร์ฟเวอร์ใช้สถาปัตยกรรม **"หนึ่ง OS thread ต่อหนึ่ง connection"** แบบเดิม
(ที่ Part 37 สอนไว้ตอนเรียน `std::thread`) จะต้องสร้าง OS thread 10,000 ตัว ซึ่งแต่ละตัวกิน stack memory เริ่มต้น
ประมาณ 2-8 MB (ขึ้นกับ OS และการตั้งค่า) — รวมแล้วอาจกิน RAM หลักหมื่น MB **โดยที่ thread ส่วนใหญ่ในทุกขณะเวลาไม่ได้
ทำงานอะไรเลย นอกจาก "รอ" ข้อมูลจาก network** ซึ่งเป็นการสิ้นเปลืองทรัพยากรมหาศาลเพื่อรองรับงานที่ CPU ไม่ได้ทำงานจริง
แม้แต่นิดเดียวในเวลาส่วนใหญ่

นี่คือจุดที่ async/await เข้ามาแก้ปัญหาตรงประเด็นที่สุด: task ของ Tokio (ที่ Part 48 อธิบายไว้ว่าเป็น "green thread"
ที่เบากว่า OS thread มาก มีขนาดเริ่มต้นเพียงไม่กี่ร้อย byte ไม่ใช่หลาย MB) สามารถ**รอ** ข้อมูลจาก network ได้โดยไม่กิน
OS thread จริงเลยระหว่างรอ — executor จะ "พัก" task นั้นไว้ (ผ่านกลไก `Future::poll` ที่ Part 47 อธิบายไว้) แล้วสลับไป
รัน task อื่นที่มีงานจริงให้ทำแทน บน thread ของ OS จำนวนน้อยมาก (ปกติเท่ากับจำนวน CPU core) เมื่อข้อมูลใหม่มาถึง
executor จะปลุก task ที่รอไว้ให้ทำงานต่อ — **ผลลัพธ์คือรองรับ connection หลักหมื่นถึงหลักแสนพร้อมกันได้บนเครื่องเดียว
โดยใช้ทรัพยากรเพียงเสี้ยวเดียวของสถาปัตยกรรม thread-per-connection**

นี่ไม่ใช่แค่ทฤษฎี — นี่คือเหตุผลที่ web server ระดับ production ในโลก Rust ทั้งหมด (Axum, Actix-web, Tonic สำหรับ gRPC,
รวมถึง reverse proxy อย่าง Linkerd2-proxy) สร้างอยู่บน Tokio ทั้งสิ้น เพราะภาระงานหลักของพวกมันคือ "รอรับ connection
เข้ามา รออ่านข้อมูลจาก client อ่านเสร็จก็ประมวลผลนิดหน่อยแล้วรอเขียนข้อมูลกลับ" ซึ่งเป็นงานที่ **I/O-bound** (ถูกจำกัด
ด้วยความเร็ว I/O ไม่ใช่ความเร็ว CPU) เกือบร้อยเปอร์เซ็นต์ — ตรงข้ามกับงาน **CPU-bound** (เช่น การคำนวณตัวเลขหนัก ๆ,
การเข้ารหัสวิดีโอ) ที่ async ไม่ได้ช่วยอะไรเลย เพราะ CPU ทำงานตลอดเวลาอยู่แล้ว ไม่มี "ช่วงรอ" ให้ task อื่นแซงคิว
(งานประเภทนี้ควรใช้ thread แบบ Part 37 หรือ thread pool ของ Rayon มากกว่า)

บทนี้จะสอนคุณสร้าง networking code จริงที่ใช้แพทเทิร์นเดียวกันกับที่ web framework ระดับ production ใช้ข้างใน (แม้จะใน
ขนาดที่เล็กและง่ายกว่ามาก) — เริ่มจาก async file I/O เป็นตัวอุ่นเครื่องก่อน เพราะมัน "ง่ายกว่า" networking มาก
(ไม่มีปัญหาเรื่อง connection, ไม่มีปัญหาเรื่อง framing) แต่ใช้ traitชุดเดียวกันเป๊ะ ๆ กับที่ TCP จะใช้ต่อไป

### 49.2 Async File I/O: ตัวอุ่นเครื่องก่อนเข้าเน็ตเวิร์ก

Tokio มีโมดูล `tokio::fs` ที่เป็นเวอร์ชัน async ของ `std::fs` (ที่ Part 12 แนะนำไว้ตอนพูดถึง `std::fs::File` และ
`io::Result`) — ชนิดข้อมูลหลักคือ `tokio::fs::File` ซึ่งมี method คล้าย `std::fs::File` มาก แต่ **ทุก method ที่ทำ I/O
จริงคืนค่าเป็น `Future` ที่ต้อง `.await`** เพราะการอ่าน/เขียนไฟล์ (แม้จะเร็วกว่า network มาก) ก็ยังเป็น **operation ที่
เกี่ยวข้องกับ OS/disk** ซึ่งในทางเทคนิคอาจบล็อกได้ (ดิสก์ทำงานช้ากว่า RAM เสมอ แม้จะเป็น SSD ก็ตาม)

เมธอดสำหรับอ่านและเขียนไม่ได้อยู่บน `tokio::fs::File` ตรง ๆ แต่มาจาก **extension trait** สองตัวที่สำคัญที่สุดของบทนี้
ทั้งบท:

- **`tokio::io::AsyncReadExt`**: เติม method เช่น `.read()`, `.read_exact()`, `.read_to_end()`, `.read_to_string()`
  ให้กับทุกชนิดที่ implement `AsyncRead` (ซึ่ง `tokio::fs::File` implement ไว้)
- **`tokio::io::AsyncWriteExt`**: เติม method เช่น `.write()`, `.write_all()`, `.flush()`, `.shutdown()` ให้กับทุกชนิด
  ที่ implement `AsyncWrite`

รูปแบบนี้คล้ายกับที่ Part 25/26 สอนเรื่อง `Iterator` trait มาก: `Iterator` เองมีแค่ method หลักไม่กี่ตัว
(`.next()`) แต่ method สะดวก ๆ ทั้งหมด (`.map()`, `.filter()`, `.collect()`) มาจาก blanket implementation ที่เพิ่มเข้า
มาให้ทุกชนิดที่ implement `Iterator` โดยอัตโนมัติ — `AsyncRead`/`AsyncWrite` ก็เป็นแบบเดียวกัน: trait หลักมีแค่กลไก
polling ระดับต่ำ (`poll_read`/`poll_write`) ส่วน method ที่เราเรียกใช้ตรง ๆ ในโค้ดแอปพลิเคชันมาจาก extension trait
`AsyncReadExt`/`AsyncWriteExt` ที่ **ต้อง `use` เข้ามาก่อนเสมอ** ไม่งั้น method เหล่านี้จะไม่ปรากฏให้เรียกใช้เลย
(compiler จะบอก error "method not found" ถ้าคุณลืม `use`)

มาดูตัวอย่างเขียนและอ่านไฟล์แบบ async จริง ๆ:

```rust
use tokio::fs::File;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // เขียนไฟล์แบบ async: สร้างไฟล์ใหม่ (หรือ truncate ถ้ามีอยู่แล้ว) แล้วเขียนเนื้อหาลงไป
    let mut file = File::create("/tmp/greeting.txt").await?;
    file.write_all(b"Hello from async Rust!\n").await?;
    file.write_all("สวัสดี Tokio\n".as_bytes()).await?;
    file.flush().await?;

    // อ่านไฟล์กลับแบบ async
    let mut file = File::open("/tmp/greeting.txt").await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    print!("{contents}");

    Ok(())
}
```

ผลลัพธ์จริง (รันด้วย `cargo run` โดยมี `tokio = { version = "1", features = ["full"] }` ใน `Cargo.toml`):

```
Hello from async Rust!
สวัสดี Tokio
```

สังเกตจุดสำคัญหลายจุด:

1. **`File::create()` และ `File::open()` คืนค่าเป็น `Future`** (ต่างจาก `std::fs::File::create()` ที่ทำงานทันทีแบบ
   synchronous) จึงต้อง `.await` — ทั้งคู่คืนค่าเป็น `io::Result<File>` เหมือน `std::fs` ทุกประการ (Part 12) ใช้ `?`
   ได้ตรง ๆ
2. **`.write_all(&[u8])` รับ byte slice** ไม่ใช่ `&str` ตรง ๆ — เราจึงต้องแปลง string literal ภาษาไทยด้วย
   `.as_bytes()` (ทวนความรู้จาก Part 14: `&str` ข้างใต้ก็คือ `&[u8]` ที่การันตี valid UTF-8 อยู่แล้ว การแปลงด้วย
   `.as_bytes()` ไม่มีการ copy ข้อมูลใด ๆ เลย เป็นแค่การมองข้อมูลตัวเดียวกันในมุมมองที่ต่างออกไป)
3. **`.write_all()` ต่างจาก `.write()` ตรงที่การันตีว่าเขียน "ครบทุก byte"** ที่ส่งเข้าไป ในขณะที่ `.write()` (ที่มา
   จาก trait เดียวกัน) อาจเขียนได้ไม่ครบในการเรียกครั้งเดียว (คืนค่าเป็นจำนวน byte ที่เขียนสำเร็จจริง ซึ่งอาจน้อยกว่าที่
   ขอ) — ในโค้ดแอปพลิเคชันทั่วไปที่ไม่ต้องการควบคุมระดับละเอียด **ควรใช้ `.write_all()` เป็นค่าเริ่มต้นเสมอ** เพราะมัน
   วน loop เขียนซ้ำให้จนครบให้เราโดยอัตโนมัติ (`.write()` เดี่ยว ๆ เหมาะกับกรณีพิเศษที่ต้องการควบคุม flow การเขียนเอง
   เท่านั้น ซึ่งพบน้อยมากในโค้ดทั่วไป)
4. **`.read_to_string(&mut String)` อ่านทั้งไฟล์เข้า `String` ที่มีอยู่แล้วในครั้งเดียว** คล้าย `.write_all()` ตรงที่
   การันตีว่าอ่าน "จนกว่าจะถึงจุดสิ้นสุดไฟล์ (EOF)" ให้ครบ ไม่ใช่แค่อ่านบางส่วน — และเพราะมันคืนค่าเข้า `String` มัน
   จึง**ตรวจสอบว่า bytes ที่อ่านมาเป็น valid UTF-8** ด้วย (ตามกฎ "สัญญา" ของ `String` ที่ Part 14 อธิบายไว้อย่าง
   ละเอียด) ถ้าไฟล์มี bytes ที่ไม่ใช่ UTF-8 ปนอยู่ `.read_to_string()` จะคืนค่าเป็น `Err` ทันที — ถ้าต้องการอ่านไฟล์ที่
   อาจไม่ใช่ UTF-8 (เช่นไฟล์ binary) ให้ใช้ `.read_to_end(&mut Vec<u8>)` แทน ซึ่งอ่านเป็น byte ดิบไม่ตรวจสอบ encoding
   อะไรเลย

ตารางสรุป method หลักของทั้งสอง trait ที่จะใช้ตลอดทั้งบทนี้ (ทั้งกับไฟล์และกับ TCP socket ในหัวข้อต่อไป เพราะเป็น
trait เดียวกันเป๊ะ):

| Method | มาจาก trait | ทำอะไร |
|---|---|---|
| `.read(&mut buf)` | `AsyncReadExt` | อ่านเข้า buffer **อย่างน้อย 1 byte แต่ไม่เกินขนาด buf** คืนค่าจำนวน byte ที่อ่านได้จริง (อาจน้อยกว่า `buf.len()`) |
| `.read_exact(&mut buf)` | `AsyncReadExt` | อ่านให้ **เต็ม buffer พอดี** เท่านั้น ถ้าข้อมูลไม่พอจะ error |
| `.read_to_end(&mut Vec<u8>)` | `AsyncReadExt` | อ่านทุก byte จนถึง EOF เก็บเป็น `Vec<u8>` ดิบ |
| `.read_to_string(&mut String)` | `AsyncReadExt` | เหมือน `.read_to_end()` แต่ตรวจสอบและแปลงเป็น `String` (ต้องเป็น valid UTF-8) |
| `.write(buf)` | `AsyncWriteExt` | เขียนบางส่วนของ `buf` (อาจไม่ครบ) คืนค่าจำนวน byte ที่เขียนสำเร็จ |
| `.write_all(buf)` | `AsyncWriteExt` | เขียนให้ครบทั้ง `buf` (วน loop ให้เองจนครบ) |
| `.flush()` | `AsyncWriteExt` | บอกให้ระบายข้อมูลที่อาจยัง buffer ค้างอยู่ออกไปจริง ๆ |

การอ่าน/เขียนไฟล์คือสนามฝึกที่ปลอดภัยที่สุดสำหรับ trait สองตัวนี้ เพราะ**ไม่มีความซับซ้อนเรื่อง connection หรือ
message boundary** เลย — คุณรู้แน่ ๆ ว่าไฟล์มีจุดเริ่มต้นและจุดสิ้นสุด (EOF) ที่ชัดเจน ต่างจาก TCP socket ที่เราจะเจอ
ในหัวข้อ 49.6 ว่า "จุดสิ้นสุดของข้อความ" ไม่ได้ชัดเจนแบบนั้นเลย — แต่ trait, method, และวิธีคิดเรื่อง `io::Result`
ทั้งหมดที่เพิ่งเรียนในหัวข้อนี้ **จะถูกใช้ซ้ำเป๊ะ ๆ ทุกตัวอักษร** ตอนทำงานกับ `TcpStream` ในหัวข้อถัดไป

### 49.3 TCP โดยสังเขป: client-server model และ reliable byte stream

ก่อนลงมือเขียนโค้ด ทวนความเข้าใจพื้นฐานเรื่อง TCP สั้น ๆ (บทนี้สมมติว่าคุณมีความรู้พื้นฐานเรื่อง networking มาบ้างแล้ว
จึงไม่ลงรายละเอียดเชิง protocol ลึก ๆ อย่าง three-way handshake หรือ TCP header format):

- **Client-server model**: ฝั่งหนึ่ง (**server**) เปิด port รอรับ connection ไว้ล่วงหน้า ฝั่งอีกฝั่ง (**client**)
  เป็นผู้เริ่ม connection เข้ามาที่ IP address และ port ของ server — เมื่อ connection สำเร็จ ทั้งสองฝั่งจะสื่อสารกัน
  ผ่าน **socket** ที่เปิดไว้ ส่งข้อมูลไปมาได้ทั้งสองทาง (**full-duplex**)
- **TCP (Transmission Control Protocol)** การันตี 3 อย่างที่สำคัญที่สุดสำหรับโปรแกรมเมอร์ระดับแอปพลิเคชัน:
  1. **Reliable**: ข้อมูลที่ส่งไปจะถึงปลายทางแน่นอน (ถ้า connection ไม่ขาด) — ถ้า packet หายไปกลางทาง TCP จะส่งซ้ำให้
     เองโดยอัตโนมัติ โดยที่โค้ดแอปพลิเคชันไม่ต้องรู้เรื่องนี้เลย
  2. **Ordered**: ข้อมูลจะมาถึงปลายทาง**ตามลำดับที่ส่งไปเป๊ะ** แม้ในระดับ network packet อาจมาถึงไม่ตามลำดับ (เพราะ
     เดินทางผ่านเส้นทางต่างกัน) TCP จะจัดเรียงให้ถูกต้องก่อนส่งต่อให้แอปพลิเคชัน
  3. **Byte stream**: ข้อมูลถูกมองเป็น **"ลำธารของ byte ที่ไหลต่อเนื่อง"** ไม่มีแนวคิดเรื่อง "ข้อความ" หรือ "record"
     ในระดับ protocol เลย — นี่คือจุดที่ **สำคัญที่สุด** สำหรับบทนี้ และเป็นที่มาของปัญหา framing ที่จะเจาะลึกใน
     หัวข้อ 49.6 (พูดล่วงหน้าไว้ตรงนี้ก่อน เพื่อให้จำภาพนี้ติดตัวไปตลอดบท)

สิ่งที่ TCP **ไม่การันตี** ให้เลยคือ "ขอบเขตของข้อความ" (message boundary) — ถ้าฝั่งส่งเรียก `write()` สองครั้งติดกัน
ด้วยข้อมูล `"HELLO"` แล้ว `"WORLD"` ฝั่งรับ**ไม่มีการันตีว่าจะได้ `read()` สองครั้งที่ตรงกับการ `write()` สองครั้งนั้น
เป๊ะ ๆ** — อาจได้ `read()` ครั้งเดียวที่มีทั้ง `"HELLOWORLD"` รวมกัน หรืออาจได้ `read()` หลายครั้งที่แบ่งข้อความ
`"HELLO"` เดียวออกเป็นชิ้นเล็ก ๆ ก็ได้ (ขึ้นกับขนาด buffer, ความเร็ว network, การตั้งค่า OS) — นี่ไม่ใช่ข้อบกพร่องของ
TCP แต่เป็น**การออกแบบโดยตั้งใจ**: TCP รับประกันแค่ "byte มาถึงครบถูกลำดับ" เท่านั้น ส่วน "ขอบเขตของข้อความ" เป็นเรื่อง
ที่ **protocol ระดับสูงกว่า (แอปพลิเคชันของเราเอง) ต้องกำหนดขึ้นมาเอง** — เราจะเห็นปัญหานี้แบบจับต้องได้จริงด้วยโค้ด
และผลลัพธ์จริงในหัวข้อ 49.6

### 49.4 `TcpListener` และ `TcpStream`: สร้าง Echo Server ตัวแรก

`tokio::net::TcpListener` คือชนิดข้อมูลสำหรับ **รอรับ connection เข้ามา** (ฝั่ง server) มี method หลักสองตัว:

- **`TcpListener::bind(addr)`**: ผูก (bind) listener เข้ากับ IP address และ port ที่กำหนด คืนค่าเป็น
  `io::Result<TcpListener>` — เป็น `async fn` (ต้อง `.await`) เพราะการ bind port เกี่ยวข้องกับ OS syscall ที่ในทาง
  เทคนิคอาจบล็อกได้ (แม้ในทางปฏิบัติจะเร็วมากแทบจะทันที) และคืนค่าเป็น `Result` ตรงกับหลักการที่ Part 12 สอนไว้:
  operation ที่ล้มเหลวได้ (เช่น port ถูกใช้งานอยู่แล้ว, ไม่มีสิทธิ์ bind port ต่ำกว่า 1024 บน Linux/macOS) ต้องคืนค่า
  เป็น `Result` ให้ผู้เรียกจัดการ ไม่ควร panic ทิ้งเฉย ๆ
- **`.accept()`**: **รอ (async)** จนกว่าจะมี client เชื่อมต่อเข้ามา แล้วคืนค่าเป็น `io::Result<(TcpStream, SocketAddr)>`
  — ได้ `TcpStream` สำหรับคุยกับ client ตัวนั้นโดยเฉพาะ พร้อม `SocketAddr` ที่บอก IP/port ของฝั่ง client (peer address)

โครงสร้างมาตรฐานของ TCP server ทุกตัวใน Tokio คือ **loop รับ connection ไม่จบไม่สิ้น** — ทุกครั้งที่ `.accept()`
คืนค่าออกมา ให้ `tokio::spawn` task ใหม่แยกไปจัดการ connection นั้น แล้ววน loop กลับไป `.accept()` ต่อทันที (ไม่รอให้
connection ก่อนหน้าจบก่อน):

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878").await?;
    println!("Echo server listening on 127.0.0.1:7878");

    loop {
        // .accept() รอ (async) จนกว่าจะมี client ใหม่เชื่อมต่อเข้ามา
        let (mut socket, peer_addr) = listener.accept().await?;
        println!("New connection from {peer_addr}");

        // spawn task ใหม่หนึ่งตัวต่อหนึ่ง connection — connection นี้ทำงานอิสระ
        // จาก connection อื่นโดยสิ้นเชิง ไม่บล็อก loop accept() ข้างบนเลย
        tokio::spawn(async move {
            let mut buf = [0u8; 1024];
            loop {
                let n = match socket.read(&mut buf).await {
                    // read() คืนค่า 0 หมายความว่า client ปิด connection แล้ว (EOF)
                    Ok(0) => {
                        println!("{peer_addr} disconnected");
                        return;
                    }
                    Ok(n) => n,
                    Err(e) => {
                        eprintln!("read error from {peer_addr}: {e}");
                        return;
                    }
                };

                // echo กลับ: เขียน byte ที่อ่านมาได้กลับไปเป๊ะ ๆ
                if let Err(e) = socket.write_all(&buf[..n]).await {
                    eprintln!("write error to {peer_addr}: {e}");
                    return;
                }
            }
        });
    }
}
```

นี่คือจุดที่ **เชื่อมความรู้จาก Part 48 เข้ากับ networking โดยตรงที่สุดในทั้งบท**: การ `tokio::spawn` หนึ่ง task ต่อ
หนึ่ง connection คือ **canonical use case** (กรณีใช้งานต้นแบบ) ของ `tokio::spawn` ในโลกจริงทั้งหมด — ทบทวนจาก
Part 48: task ของ Tokio เบามาก การมี task เป็นพัน ๆ ตัวที่ส่วนใหญ่กำลัง "รอ" ข้อมูลจาก `.read().await` ไม่ได้เปลือง
ทรัพยากรของ OS thread จริงเลย เพราะ executor จะสลับไปทำงานอื่นตอนที่ task ไหนกำลังรออยู่ — ถ้าเราไม่ `tokio::spawn`
แต่ประมวลผลแต่ละ connection ต่อกันไปตรง ๆ ใน loop เดียว (`listener.accept().await` แล้วจัดการจนจบค่อยวนกลับไป
`accept()` ใหม่) เซิร์ฟเวอร์จะรับได้แค่ **client เดียวในเวลาเดียวกัน** — client ตัวที่สองต้องรอจน client ตัวแรกตัดการ
เชื่อมต่อก่อน ซึ่งไม่มีประโยชน์อะไรเลยสำหรับ server ที่ต้องรองรับหลายคนพร้อมกัน

สังเกตรายละเอียดสำคัญอีกจุดในโค้ด: **`Ok(0)` จาก `.read()` หมายถึง "ฝั่งตรงข้ามปิด connection แล้ว" (EOF)** ไม่ใช่
"ยังไม่มีข้อมูลมาใหม่" (ถ้ายังไม่มีข้อมูลมาใหม่ `.read().await` จะยังไม่ return เลย มันจะรอต่อไปจนกว่าจะมีข้อมูลจริง ๆ
หรือ connection ถูกปิด) — การไม่เช็ค `Ok(0)` แล้ว `break`/`return` ออกจาก loop ให้ถูกต้องเป็นสาเหตุของบัคที่พบบ่อยมาก
(จะเจาะลึกในหัวข้อกับดักท้ายบท) เพราะถ้าไม่เช็ค การอ่านค่า 0 byte ซ้ำ ๆ วนไปเรื่อย ๆ ใน loop โดยไม่หยุด จะทำให้เกิด
**busy loop กิน CPU 100%** ทันที (เพราะ `.read()` บน connection ที่ปิดแล้วจะ return `Ok(0)` ทันทีแบบไม่รอเลย ต่างจาก
ตอน connection ยังเปิดอยู่แต่ไม่มีข้อมูลใหม่ที่จะรอแบบ async จริง)

### 49.5 รันจริง: Echo Server คุยกับ Echo Client ผ่าน TCP บน localhost

มาสร้าง client ฝั่งตรงข้ามเพื่อคุยกับ server ที่เพิ่งเขียน — ในโปรเจกต์จริงมักแยกเป็นสอง binary ในโปรเจกต์เดียวกัน
(เช่น `src/bin/server.rs` และ `src/bin/client.rs`) รันแยกกันในสอง terminal:

```rust
// src/bin/client.rs
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let mut stream = TcpStream::connect("127.0.0.1:7878").await?;
    stream.write_all(b"Hello, server!").await?;

    let mut buf = [0u8; 1024];
    let n = stream.read(&mut buf).await?;
    println!("Server echoed: {}", String::from_utf8_lossy(&buf[..n]));

    Ok(())
}
```

`TcpStream::connect(addr)` เป็น method คู่กับ `TcpListener::bind()` ฝั่ง server: เป็น `async fn` ที่พยายามเชื่อมต่อไป
ยัง address ที่กำหนด คืนค่าเป็น `io::Result<TcpStream>` — ถ้า server ไม่ได้เปิดรออยู่ที่ port นั้น (หรือ port ผิด,
firewall บล็อก) จะได้ `Err` กลับมาให้จัดการ (เช่น `ConnectionRefused`)

สังเกตว่า `TcpStream` ที่ได้ทั้งฝั่ง server (จาก `.accept()`) และฝั่ง client (จาก `.connect()`) เป็น**ชนิดข้อมูล
เดียวกัน** และใช้ trait `AsyncReadExt`/`AsyncWriteExt` ชุดเดียวกันเป๊ะกับที่เรียนไปแล้วตอนอ่านเขียนไฟล์ในหัวข้อ 49.2
— นี่คือพลังของการออกแบบ trait แบบ extension ใน Rust: โค้ดที่เขียนขึ้นให้ทำงานกับ "อะไรก็ได้ที่ implement
`AsyncRead`/`AsyncWrite`" ใช้ได้กับทั้งไฟล์และ socket โดยไม่ต้องเปลี่ยนแปลงอะไรเลย

รันจริง (สอง terminal, แสดงผลลัพธ์จริงที่ได้จากการรัน `cargo run --bin server` และ `cargo run --bin client` โดยมี
`Cargo.toml` ที่ตั้ง `[[bin]]` สองตัวชี้ไปที่ไฟล์ทั้งสอง):

**Terminal 1 (server):**

```
$ cargo run --bin server
Echo server listening on 127.0.0.1:7878
New connection from 127.0.0.1:41290
127.0.0.1:41290 disconnected
```

**Terminal 2 (client):**

```
$ cargo run --bin client
Server echoed: Hello, server!
```

(หมายเหตุเรื่องวิธีตรวจสอบ: ในการเขียนบทนี้ ผู้เขียนได้ทดสอบ logic ทั้งหมดของ server และ client ด้วยการรันจริงผ่าน
TCP บน `127.0.0.1` — เพื่อความสะดวกในการจับภาพผลลัพธ์แบบอัตโนมัติ (ไม่ต้องเปิดสอง terminal พร้อมกันด้วยมือ) จึงรวม
logic ของ server ไว้เป็น background task ด้วย `tokio::spawn` ภายในโปรเซสเดียวกับ client แล้วหน่วงเวลาสั้น ๆ
(`sleep(Duration::from_millis(100))`) ก่อนให้ client เชื่อมต่อ เพื่อให้แน่ใจว่า `bind()` เสร็จก่อน `connect()` เริ่ม —
แต่ **นี่ยังคือ TCP connection จริงผ่าน loopback interface (127.0.0.1) ทุกประการ ไม่ใช่การจำลอง** ไม่มีความแตกต่างใน
เชิงเน็ตเวิร์กเลยระหว่างรันแบบนี้กับรันเป็นสอง process/terminal แยกกัน — logic ที่ทดสอบคือโค้ดเดียวกันเป๊ะกับที่แสดง
ในบทความ เพียงแค่จัดการ orchestration ของการรันต่างกันเท่านั้น)

ผลลัพธ์จริงที่จับภาพได้จากการรันแบบรวมโปรเซสข้างต้น เมื่อทดสอบด้วย client 4 ตัว (ตัวแรกทดสอบเดี่ยว ๆ อีกสามตัวยิง
พร้อมกันด้วย `tokio::join!` เพื่อพิสูจน์ concurrent handling ที่จะพูดถึงในหัวข้อ 49.8):

```
[server] listening on 127.0.0.1:17878
[server] new connection from 127.0.0.1:41290
[client 1] server echoed: Hello, server!
[server] 127.0.0.1:41290 disconnected
[server] new connection from 127.0.0.1:41300
[server] new connection from 127.0.0.1:41302
[server] new connection from 127.0.0.1:41318
[client 2] server echoed: message from client 2
[client 3] server echoed: message from client 3
[server] 127.0.0.1:41300 disconnected
[client 4] server echoed: message from client 4
```

สังเกตบรรทัดสำคัญ: **`[server] new connection from ...` ทั้งสามบรรทัดสำหรับ client 2, 3, 4 เกิดขึ้นติดกันก่อนที่
client ตัวใดจะได้รับคำตอบกลับเลย** — นี่คือหลักฐานจับต้องได้ว่า server ไม่ได้จัดการ client ทีละตัวแบบเรียงคิว แต่
`.accept()` วนกลับไปรับ connection ใหม่ได้ทันทีโดยไม่ต้องรอให้ task ของ connection ก่อนหน้าทำงานเสร็จก่อน — ตรงกับที่
อธิบายไว้ในหัวข้อ 49.4 ว่า `tokio::spawn` ทำให้ทุก connection ทำงานอิสระจากกัน

### 49.6 ปัญหาการวางกรอบข้อความ (Framing): TCP คือ byte stream ไม่มีขอบเขตข้อความ

หัวข้อ 49.3 พูดถึงข้อเท็จจริงนี้ไว้ล่วงหน้าแล้ว — ตอนนี้มาพิสูจน์ด้วยโค้ดและผลลัพธ์จริงว่ามันเกิดขึ้นได้จริงแค่ไหน
ปัญหานี้แบ่งออกเป็นสองรูปแบบที่ตรงข้ามกัน:

#### ปัญหาแบบที่ 1: ข้อความหลายก้อนถูก "รวม" มาใน `.read()` ครั้งเดียว (merging)

ถ้า server เขียนสองข้อความติดกันแบบไม่มีดีเลย์คั่นเลย:

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpListener;

async fn merge_server(addr: &str) -> std::io::Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let (mut socket, _peer) = listener.accept().await?;
    socket.write_all(b"HELLO").await?;
    socket.write_all(b"WORLD").await?;
    Ok(())
}
```

ฝั่ง client ที่อ่านด้วย `.read()` ครั้งเดียวใน buffer ใหญ่พอ:

```rust
use tokio::io::AsyncReadExt;
use tokio::net::TcpStream;

async fn merge_client(addr: &str) -> std::io::Result<()> {
    let mut stream = TcpStream::connect(addr).await?;
    let mut buf = [0u8; 1024];
    let n = stream.read(&mut buf).await?;
    println!(
        "read() หนึ่งครั้งได้ {n} bytes = {:?}",
        String::from_utf8_lossy(&buf[..n])
    );
    Ok(())
}
```

ผลลัพธ์จริงจากการรัน:

```
read() หนึ่งครั้งได้ 10 bytes = "HELLOWORLD"
```

สังเกตว่า **`.read()` เพียงครั้งเดียวได้ทั้ง `"HELLO"` และ `"WORLD"` มารวมกันเป็น `"HELLOWORLD"`** ทั้งที่ฝั่ง server
เรียก `write_all()` แยกกันสองครั้งอย่างชัดเจน — ถ้าโค้ดฝั่ง client สมมติ (ผิด ๆ) ว่า "หนึ่ง `write_all()` ฝั่งส่งเท่ากับ
หนึ่ง `read()` ฝั่งรับเสมอ" มันจะพัง เพราะ TCP stack ของ OS มีสิทธิ์รวม (coalesce) ข้อมูลที่ส่งมาใกล้ ๆ กันในเวลาเข้า
buffer เดียวกันได้เสมอ ไม่มีอะไรห้าม — นี่ไม่ใช่บัค เป็นพฤติกรรมที่ถูกต้องตามสัญญาของ TCP (แค่การันตี byte มาถึงครบ
ถูกลำดับ ไม่การันตีขอบเขตของการเรียก `write`)

#### ปัญหาแบบที่ 2: หนึ่งข้อความถูก "แยก" เป็นหลาย `.read()` (splitting)

ในทางกลับกัน ถ้าข้อความเดียวมีขนาดใหญ่กว่า buffer ที่ฝั่งรับใช้ ก็ต้อง `.read()` หลายครั้งกว่าจะได้ข้อความครบ — เรื่อง
นี้เกิดขึ้น**แน่นอนเสมอ** ไม่ต้องพึ่ง timing ของ network เลย เพราะมันเกิดจากขนาด buffer ตรง ๆ:

```rust
async fn split_server(addr: &str) -> std::io::Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let (mut socket, _peer) = listener.accept().await?;
    let big = vec![b'X'; 5000]; // ข้อความเดียว ขนาด 5000 bytes
    socket.write_all(&big).await?;
    Ok(())
}

async fn split_client(addr: &str) -> std::io::Result<()> {
    let mut stream = TcpStream::connect(addr).await?;
    let mut buf = [0u8; 1024]; // buffer เล็กกว่าข้อความมาก (1024 < 5000)
    let mut total = 0usize;
    let mut read_calls = 0u32;
    loop {
        let n = stream.read(&mut buf).await?;
        if n == 0 {
            break;
        }
        total += n;
        read_calls += 1;
        println!("read() ครั้งที่ {read_calls} ได้ {n} bytes (รวม {total})");
    }
    println!("รวมทั้งหมด {total} bytes จาก read() {read_calls} ครั้ง");
    Ok(())
}
```

ผลลัพธ์จริง:

```
read() ครั้งที่ 1 ได้ 1024 bytes (รวม 1024)
read() ครั้งที่ 2 ได้ 1024 bytes (รวม 2048)
read() ครั้งที่ 3 ได้ 1024 bytes (รวม 3072)
read() ครั้งที่ 4 ได้ 1024 bytes (รวม 4096)
read() ครั้งที่ 5 ได้ 904 bytes (รวม 5000)
รวมทั้งหมด 5000 bytes จาก read() 5 ครั้ง
```

ข้อความเดียวขนาด 5000 bytes ถูกแยกเป็น **5 ครั้งของการเรียก `.read()`** (1024 × 4 + 904 = 5000 พอดี) เพราะ buffer
ฝั่งรับมีขนาดแค่ 1024 bytes — นี่คือเหตุผลที่ `.read()` เดี่ยว ๆ **ไม่เคยเป็นตัวเลือกที่ปลอดภัยสำหรับอ่าน "หนึ่ง
ข้อความที่สมบูรณ์"** เลย ไม่ว่าข้อความจะมาจาก TCP โดยตรงหรือไม่ก็ตาม เพราะ `.read()` ตามสัญญาของมันคืนแค่ **"อ่านได้
เท่าไหร่ก็เท่านั้น ไม่เกิน buffer"** — ไม่ใช่ "อ่านจนกว่าจะได้ข้อความที่สมบูรณ์หนึ่งก้อน"

คำถามที่ตามมาคือ: **แล้วเราจะรู้ได้ยังไงว่า "ข้อความหนึ่งก้อน" จบตรงไหน?** — คำตอบคือ TCP เองไม่รู้และไม่สนใจเรื่องนี้
เลย มันเป็นหน้าที่ของ **protocol ระดับแอปพลิเคชัน** ที่เราออกแบบขึ้นมาเองที่ต้องกำหนดกฎเกณฑ์บางอย่างให้ทั้งสองฝั่ง
(ทั้งฝั่งเขียนและฝั่งอ่าน) เห็นพ้องกัน วิธีที่ง่ายและใช้กันแพร่หลายที่สุดคือ **newline-delimited protocol**: ตกลงกัน
ว่า **ทุกข้อความจบด้วยตัวอักษร `\n`** ฝั่งอ่านก็แค่ "อ่านไปจนกว่าจะเจอ `\n`" แทนการ `.read()` แบบสุ่ม ๆ (protocol
ข้อความอย่างเป็นทางการอื่น ๆ เช่น HTTP/1.1 ก็ใช้แนวคิดคล้ายกัน คือแยกส่วน header ด้วย `\r\n` และมี `Content-Length`
บอกความยาวของ body ชัดเจน)

### 49.7 แก้ปัญหา Framing ด้วย Newline-Delimited Protocol (`BufReader::lines()`)

Tokio มี trait `AsyncBufReadExt` ที่เติม method `.lines()` ให้กับชนิดข้อมูลที่ implement `AsyncBufRead` — และ
`tokio::io::BufReader<R>` คือ wrapper ที่ครอบ `AsyncRead` ใด ๆ (เช่น `TcpStream`) ให้กลายเป็น `AsyncBufRead` โดยการ
เก็บ buffer ภายในของตัวเอง (คล้าย `std::io::BufReader` ที่อาจเคยเห็นตอนอ่านไฟล์แบบ synchronous) — `.lines()` คืนค่า
เป็น stream ที่มี method `.next_line()` ซึ่งอ่านไปเรื่อย ๆ (เรียก `.read()` ข้างใต้กี่ครั้งก็ได้ตามต้องการ) **จนกว่าจะ
เจอ `\n`** แล้วคืนบรรทัดที่สมบูรณ์นั้นออกมาเป็น `Option<String>` (ตัด `\n` ท้ายบรรทัดออกให้อัตโนมัติด้วย) — ถ้าสอง
ข้อความถูก merge มาในการอ่านครั้งเดียวจาก OS เหมือนในหัวข้อ 49.6 มันก็ยังแยกออกเป็นสองบรรทัดที่ถูกต้องให้เราได้ เพราะ
มันมองหาตำแหน่ง `\n` ในข้อมูลที่สะสมไว้ ไม่ใช่มองแค่ผลลัพธ์ของ `.read()` ดิบ ๆ ครั้งเดียว

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};

async fn line_server(addr: &str) -> std::io::Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let (socket, _peer) = listener.accept().await?;

    // แยก socket เป็นครึ่งอ่าน/ครึ่งเขียน เพื่อใช้ BufReader ครอบฝั่งอ่านได้อย่างอิสระ
    let (reader, mut writer) = tokio::io::split(socket);
    let mut lines = BufReader::new(reader).lines();

    while let Some(line) = lines.next_line().await? {
        println!("ได้รับข้อความสมบูรณ์ 1 บรรทัด: {line:?}");
        let reply = format!("ECHO: {line}\n");
        writer.write_all(reply.as_bytes()).await?;
    }
    Ok(())
}

async fn line_client(addr: &str) -> std::io::Result<()> {
    let mut stream = TcpStream::connect(addr).await?;
    // ส่งสามบรรทัดในการ write ครั้งเดียว (จงใจให้ merge กันแบบในหัวข้อก่อน)
    stream
        .write_all(b"first line\nsecond line\nthird line\n")
        .await?;

    let (reader, _writer) = stream.into_split();
    let mut lines = BufReader::new(reader).lines();
    let mut received = Vec::new();
    for _ in 0..3 {
        if let Some(line) = lines.next_line().await? {
            received.push(line);
        }
    }
    println!("แยกข้อความได้ถูกต้อง {} บรรทัด: {received:?}", received.len());
    Ok(())
}
```

ผลลัพธ์จริง:

```
ได้รับข้อความสมบูรณ์ 1 บรรทัด: "first line"
ได้รับข้อความสมบูรณ์ 1 บรรทัด: "second line"
ได้รับข้อความสมบูรณ์ 1 บรรทัด: "third line"
แยกข้อความได้ถูกต้อง 3 บรรทัด: ["ECHO: first line", "ECHO: second line", "ECHO: third line"]
```

แม้ client จะส่งทั้งสามบรรทัดออกไปในการเรียก `write_all()` เพียง**ครั้งเดียว** (ซึ่งมีโอกาสสูงมากที่ OS จะส่งมันไปเป็น
TCP segment เดียวและฝั่งรับได้มาในการ `.read()` ครั้งเดียวแบบรวมกันทั้งหมด เหมือนปัญหา merge ในหัวข้อก่อน) แต่
`.lines()`/`.next_line()` ก็ยังแยกออกมาเป็น 3 บรรทัดที่ถูกต้องได้อย่างสมบูรณ์แบบ — เพราะมันไม่ได้สนใจว่า `.read()`
แต่ละครั้งข้างใต้ได้ข้อมูลมาเท่าไหร่ มันแค่**สะสม byte ไว้ในบัฟเฟอร์ภายในแล้วมองหาตำแหน่ง `\n`** ไปเรื่อย ๆ จนกว่าจะ
พบ นี่คือวิธีแก้ปัญหา framing ที่ตรงประเด็นที่สุดสำหรับ protocol ที่เป็นข้อความ (text-based protocol)

ข้อสังเกตเรื่อง API สองจุด:

- **`tokio::io::split(socket)`**: แยก `TcpStream` เดียวออกเป็นครึ่งอ่าน (`ReadHalf`) กับครึ่งเขียน (`WriteHalf`) ที่
  ใช้งานพร้อมกันได้อิสระ (เช่น อ่านใน task หนึ่ง เขียนใน task อื่นพร้อมกัน) โดยข้างใต้ยังแชร์ socket จริงตัวเดียวกัน
  ผ่าน `Arc` (แนวคิดเดียวกับที่ Part 39 สอนเรื่องแบ่งปันข้อมูลด้วย `Arc` เพียงแต่คราวนี้ Tokio ทำให้เราโดยอัตโนมัติ
  ข้างใน `split()`)
- **`.into_split()`**: ทำสิ่งเดียวกันกับ `tokio::io::split()` แต่เรียกเป็น method ตรงบน `TcpStream` (บริโภค
  `self` ไปเลย) คืนค่า `OwnedReadHalf`/`OwnedWriteHalf` ที่เป็น `'static` (ไม่ยึด lifetime ของ stream เดิม) ทำให้
  ย้าย (move) เข้าไปใน `tokio::spawn` task แยกกันได้สะดวกกว่า `split()` แบบธรรมดาที่ยืม (borrow) `&mut socket` อยู่
  — เราจะใช้ `.into_split()` ในหัวขัดถัดไปตอนต้องแยกอ่าน/เขียนไปอยู่ใน task คนละตัวกันจริง ๆ

### 49.8 รองรับหลาย Client พร้อมกันจริง ๆ (Concurrent Clients)

หัวข้อ 49.5 ได้แสดงหลักฐานไปแล้วว่า server ที่ `tokio::spawn` หนึ่ง task ต่อหนึ่ง connection รองรับ 3-4 client พร้อมกัน
ได้จริงโดยไม่บล็อกกัน — มาย้ำภาพนี้ให้ชัดขึ้นด้วยการปรับ echo server ให้ใช้ newline-delimited protocol (จากหัวข้อก่อน)
ผสมกับการรองรับหลาย client พร้อมกัน ซึ่งเป็นรูปแบบที่ตรงกับสถานการณ์จริงมากกว่า echo server ดิบ ๆ:

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

async fn run_line_echo_server(addr: &str) -> std::io::Result<()> {
    let listener = TcpListener::bind(addr).await?;
    loop {
        let (socket, peer_addr) = listener.accept().await?;

        tokio::spawn(async move {
            let (reader, mut writer) = tokio::io::split(socket);
            let mut lines = BufReader::new(reader).lines();

            loop {
                match lines.next_line().await {
                    Ok(Some(line)) => {
                        println!("{peer_addr} ส่งมา: {line}");
                        let reply = format!("ECHO: {line}\n");
                        if writer.write_all(reply.as_bytes()).await.is_err() {
                            break;
                        }
                    }
                    Ok(None) => {
                        println!("{peer_addr} ปิด connection");
                        break;
                    }
                    Err(e) => {
                        eprintln!("อ่านจาก {peer_addr} ผิดพลาด: {e}");
                        break;
                    }
                }
            }
        });
    }
}
```

จุดสำคัญที่ต้องสังเกตคือ **`.next_line()` คืนค่าเป็น `io::Result<Option<String>>`** ไม่ใช่ `Option<String>` ตรง ๆ —
`Ok(None)` หมายถึง "ถึงจุดสิ้นสุดของ stream อย่างสมบูรณ์ (EOF) โดยไม่มี error" (คือ connection ปิดแบบปกติ) ส่วน
`Err(e)` หมายถึงเกิดปัญหาจริง ๆ ระหว่างอ่าน (เช่น connection ถูกตัดกลางทางแบบผิดปกติ) — การแยกสองกรณีนี้ให้ชัดคือการ
ประยุกต์ใช้หลักการจาก Part 12 อีกครั้ง: `Option` บอก "มีค่าหรือไม่มี" ส่วน `Result` บอก "สำเร็จหรือล้มเหลว" ทั้งสอง
แนวคิดผสมกันเป็น `io::Result<Option<String>>` เพื่อสื่อทั้งสามสถานะที่เป็นไปได้จริงของการอ่านบรรทัดจาก stream: **ได้
บรรทัดใหม่**, **stream จบแล้วแบบปกติ**, หรือ **เกิด error**

ถ้าคุณต้องการพิสูจน์ concurrent handling ให้เห็นชัดกว่าเดิม ลองรัน client 3 ตัวพร้อมกันด้วย `tokio::join!` (เชื่อมกับ
Part 48 ที่สอน `join!` ไว้แล้ว) โดยให้แต่ละ client ส่งข้อความและรอคำตอบ — เพราะ server สร้าง task แยกให้ทุก connection
อิสระจากกัน การที่ client ตัวหนึ่งช้า (เช่น ส่งข้อความช้า หรือประมวลผลคำตอบช้า) **จะไม่ทำให้ client ตัวอื่นต้องรอ
เลยแม้แต่นิดเดียว** — ต่างจาก server แบบ single-threaded ที่จัดการทีละ connection ซึ่งถ้า connection แรกค้าง
connection ที่สองจะไม่ได้รับการตอบสนองจนกว่า connection แรกจะเสร็จหรือหลุดไปก่อน

### 49.9 Broadcast ไปยังหลาย Client: Shared State ด้วย `Arc<Mutex<T>>`

Echo server ที่เขียนมาทั้งบทมีข้อจำกัดสำคัญหนึ่งข้อ: **แต่ละ connection คุยกับ server เท่านั้น ไม่รู้จักและคุยกับ
connection อื่นไม่ได้เลย** — ในแอปพลิเคชันแบบ chat, การแจ้งเตือนแบบ real-time, หรือ multiplayer game ง่าย ๆ เราต้องการ
ให้ **ข้อความที่ client คนหนึ่งส่งมา ถูกส่งต่อ (broadcast) ไปให้ client คนอื่น ๆ ที่กำลังเชื่อมต่ออยู่ทุกคน** — นี่คือ
ปัญหาที่ต่างออกไปจากทุกอย่างที่ทำมาในบทนี้ เพราะตอนนี้ task ของ connection หนึ่งจำเป็นต้อง**เข้าถึงข้อมูลของ
connection อื่น** (อย่างน้อยก็ต้องรู้ว่ามี connection อื่นอยู่ตัวไหนบ้าง และมีวิธีส่งข้อความไปให้มันยังไง)

นี่คือปัญหา **shared state ข้าม task** แบบเดียวกันเป๊ะกับที่ Part 39 สอนไว้ตอนพูดถึง `Arc<Mutex<T>>` เพียงแต่คราวนี้
"งาน" (task) ที่ต้องแบ่งปันข้อมูลกันเป็น async task ของ Tokio แทน OS thread ธรรมดา — ทวนจาก Part 39: `Arc<T>` (Atomic
Reference Counted) ทำให้หลาย owner ถือ reference ไปยังข้อมูลเดียวกันบน heap ได้พร้อมกันอย่างปลอดภัย (คล้าย `Rc<T>`
จาก Part 28 แต่ thread-safe) ส่วน `Mutex<T>` การันตีว่ามีแค่ผู้เข้าถึงเดียวในเวลาเดียวกันเท่านั้นที่แก้ไขข้อมูลข้างในได้
(ผ่าน `.lock()`) — รวมกันเป็น `Arc<Mutex<T>>` คือรูปแบบมาตรฐานสำหรับ "ข้อมูลก้อนเดียวที่หลาย task เข้าถึงและแก้ไขร่วม
กันได้อย่างปลอดภัย"

โครงสร้างข้อมูลที่ต้องแบ่งปันคือ **"รายชื่อของทุก client ที่เชื่อมต่ออยู่ พร้อมวิธีส่งข้อความไปให้แต่ละคน"** — วิธีส่ง
ข้อความไปให้ client แต่ละคนที่สะดวกที่สุดคือ **channel** (ทวนจาก Part 38): ให้ทุก connection มี `tokio::sync::mpsc`
channel ของตัวเอง (เวอร์ชัน async ของ `std::sync::mpsc` ที่ Part 38 สอนไว้ — รายละเอียดเชิงลึกของ `tokio::sync`
ทั้งหมดจะเรียนเต็ม ๆ ใน **Part 50** บทนี้ใช้แค่ `mpsc::unbounded_channel()` แบบผิวเผินพอให้ตัวอย่าง broadcast ทำงาน
ได้จริง) เมื่อ connection A ต้องการส่งข้อความไปให้ connection B ก็แค่ส่งข้อความผ่าน **sender ของ channel ของ B**
ที่ตัว B เก็บไว้เอง — โครงสร้างที่แบ่งปันจึงกลายเป็น `Arc<Mutex<Vec<(ClientId, Sender)>>>`: list ของ (หมายเลข client,
sender สำหรับส่งข้อความไปให้ client คนนั้น) ที่ทุก task เข้าถึงร่วมกันได้ผ่าน `Arc`

```rust
use std::sync::{Arc, Mutex};
use tokio::sync::mpsc::UnboundedSender;

// รายชื่อ client ทั้งหมดที่เชื่อมต่ออยู่ ณ ขณะนี้ (id, ทางส่งข้อความไปให้ client คนนั้น)
type Clients = Arc<Mutex<Vec<(u64, UnboundedSender<String>)>>>;
```

เมื่อมี client ใหม่เข้ามา: สร้าง channel คู่ `(tx, rx)` ของตัวเอง, ใส่ `tx` เข้าไปใน `Clients` ที่แบ่งปันกัน, แล้ว
spawn task แยกอีกตัวที่คอย **รับ (`rx.recv()`) ข้อความที่ถูกส่งมาจาก client คนอื่นผ่าน channel แล้วเขียนออกไปยัง
socket จริง**:

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::tcp::OwnedWriteHalf;
use tokio::sync::mpsc;

async fn spawn_writer_task(mut writer: OwnedWriteHalf, mut rx: mpsc::UnboundedReceiver<String>) {
    while let Some(msg) = rx.recv().await {
        if writer.write_all(msg.as_bytes()).await.is_err() {
            break; // client หลุดไปแล้ว หยุดพยายามเขียนต่อ
        }
    }
}
```

ส่วนฝั่งอ่าน: ทุกครั้งที่ client ส่งบรรทัดใหม่เข้ามา ให้ **lock** `Clients` แล้ววนส่งข้อความนั้นเข้า channel ของ
ทุกคน**ยกเว้นตัวเอง**:

```rust
let list = clients.lock().unwrap();
for (other_id, sender) in list.iter() {
    if *other_id != my_id {
        let _ = sender.send(broadcast_msg.clone());
    }
}
// list (และ lock) ถูกปล่อยตอนจบ scope นี้โดยอัตโนมัติ
```

จุดที่ต้อง**ระวังเป็นพิเศษ** (จะเจาะลึกในหัวข้อกับดักท้ายบท): สังเกตว่า **ในช่วงที่ถือ lock อยู่ (ตัวแปร `list`) ไม่มี
`.await` เกิดขึ้นเลยแม้แต่จุดเดียว** — `sender.send(...)` ของ `tokio::sync::mpsc::UnboundedSender` เป็น method
**แบบ synchronous ล้วน ๆ** (ไม่ใช่ `async fn`) เพราะ unbounded channel ไม่มีเพดานความจุ จึงไม่มีเหตุผลต้อง "รอ" ก่อน
ส่งได้เลย การส่งจึงเสร็จทันทีโดยไม่ block — นี่คือการออกแบบที่ตั้งใจ เพราะถ้า critical section (ช่วงที่ถือ lock) มี
`.await` แทรกอยู่ จะเกิดปัญหาร้ายแรงที่อธิบายไว้ในหัวข้อกับดัก

### 49.10 UDP: ทางเลือกที่ไม่การันตีอะไรเลย (`tokio::net::UdpSocket`)

ตรงข้ามกับ TCP ที่การันตี reliable, ordered, byte stream อย่างที่เรียนมาทั้งบท — **UDP (User Datagram Protocol)**
ไม่การันตีอะไรเลยแม้แต่ข้อเดียว:

- **Connectionless**: ไม่มีขั้นตอน "เชื่อมต่อ" ก่อนส่งข้อมูลเหมือน TCP (`.connect()`/`.accept()`) — แค่ระบุปลายทาง
  แล้วส่งข้อมูลออกไปได้เลยทันที (คล้ายส่งจดหมาย ไม่ต้อง "โทรนัด" ก่อนส่ง)
- **ไม่การันตีการมาถึง**: packet อาจหายไปกลางทางได้โดยไม่มีการแจ้งเตือนหรือส่งซ้ำให้อัตโนมัติเหมือน TCP
- **ไม่การันตีลำดับ**: packet ที่ส่งไปก่อนอาจมาถึงหลัง packet ที่ส่งทีหลังได้ (เพราะเดินทางผ่านเส้นทางต่างกันในระดับ
  network infrastructure)
- **มีขอบเขตข้อความในตัว**: ข้อดีเดียวที่ UDP ให้มากกว่า TCP ในแง่ framing คือ**หนึ่ง `send_to()` เท่ากับหนึ่ง
  `recv_from()`** เสมอ (ถ้า packet มาถึง) — UDP ส่งข้อมูลเป็นก้อน (**datagram**) แต่ละก้อนแยกจากกันชัดเจน ไม่มีปัญหา
  merge/split เหมือน TCP byte stream ที่เจอในหัวข้อ 49.6 เลย (แต่ก็มีข้อจำกัดเรื่องขนาดสูงสุดของ datagram ที่ส่งได้
  ในครั้งเดียว ซึ่งมักอยู่ราว ๆ 65,507 bytes ตามทฤษฎี แต่ในทางปฏิบัติควรเล็กกว่านี้มากเพื่อหลีกเลี่ยงการถูก
  fragmentation ในระดับ network)

`tokio::net::UdpSocket` ใช้งานผ่าน method หลักสองตัว: **`.send_to(buf, addr)`** ส่งข้อมูลไปยัง address ที่กำหนด และ
**`.recv_from(buf)`** รอรับข้อมูล คืนค่าเป็น `(จำนวน byte, address ของผู้ส่ง)`:

```rust
use tokio::net::UdpSocket;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // ผูก socket สองตัวคนละ port บน localhost (port 0 = ให้ OS สุ่ม port ว่างให้)
    let sender_sock = UdpSocket::bind("127.0.0.1:0").await?;
    let receiver_sock = UdpSocket::bind("127.0.0.1:0").await?;
    let receiver_addr = receiver_sock.local_addr()?;

    println!("sender bound at {}", sender_sock.local_addr()?);
    println!("receiver bound at {receiver_addr}");

    sender_sock.send_to(b"ping via UDP", receiver_addr).await?;

    let mut buf = [0u8; 1024];
    let (n, from) = receiver_sock.recv_from(&mut buf).await?;
    println!(
        "receiver ได้รับ {n} bytes จาก {from}: {:?}",
        String::from_utf8_lossy(&buf[..n])
    );

    // ตอบกลับ
    receiver_sock.send_to(b"pong via UDP", from).await?;
    let (n2, from2) = sender_sock.recv_from(&mut buf).await?;
    println!(
        "sender ได้รับตอบกลับ {n2} bytes จาก {from2}: {:?}",
        String::from_utf8_lossy(&buf[..n2])
    );

    Ok(())
}
```

ผลลัพธ์จริง:

```
sender bound at 127.0.0.1:39589
receiver bound at 127.0.0.1:45852
receiver ได้รับ 12 bytes จาก 127.0.0.1:39589: "ping via UDP"
sender ได้รับตอบกลับ 12 bytes จาก 127.0.0.1:45852: "pong via UDP"
```

สังเกตว่า **ไม่มีขั้นตอน `.accept()` หรือ `.connect()` เลยตลอดทั้งตัวอย่าง** — แค่ `.bind()` เพื่อจอง port แล้วส่ง/รับ
ได้ทันที และสังเกต **`UdpSocket::bind("127.0.0.1:0")`** ที่ใช้ port หมายเลข `0` ซึ่งเป็นค่าพิเศษที่บอก OS ว่า "ช่วย
เลือก port ที่ว่างให้หน่อย" (เหมาะกับตัวอย่างทดสอบที่ไม่สนใจว่า port จะเป็นเลขอะไร ตราบใดที่รู้เลขนั้นได้ทีหลังผ่าน
`.local_addr()`)

#### เมื่อไหร่ควรเลือก UDP แทน TCP

คำแนะนำเชิงปฏิบัติ (ไม่เจาะลึกทฤษฎี protocol ระดับ packet header): เลือก UDP เมื่อแอปพลิเคชันของคุณ**ทนต่อการสูญ
หายข้อมูลบางส่วนได้ และให้ความสำคัญกับความหน่วง (latency) ต่ำมากกว่าความสมบูรณ์ของข้อมูล 100%**:

| สถานการณ์ | ตัวเลือกที่เหมาะสม | เหตุผล |
|---|---|---|
| โอนไฟล์, HTTP request/response, ข้อมูลธนาคาร | **TCP** | ข้อมูลทุก byte ต้องถึงครบและถูกลำดับ ยอมแลกกับ latency ที่สูงกว่าเล็กน้อยจากกลไก retransmission |
| Video call, เกมยิงแบบ real-time (FPS/battle royale), voice chat | **UDP** | ถ้า packet หนึ่งเฟรมหายไป การรอให้ TCP ส่งซ้ำแล้วค่อยแสดงผล (ซึ่งทำให้ทุกอย่างหลังจากนั้น "สะดุด" รอ) แย่กว่าการปล่อยเฟรมนั้นข้ามไปเลยแล้วแสดงเฟรมถัดไปที่มาถึงแทน — ข้อมูลเก่าที่มาช้ากลายเป็นข้อมูลที่ไม่มีประโยชน์แล้วในสถานการณ์ real-time |
| DNS lookup | **UDP** (มี fallback เป็น TCP ถ้าข้อความใหญ่เกินไป) | ข้อความสั้นมาก การ retry ทั้งหมดง่ายกว่าที่จะเสียเวลากับ TCP handshake สำหรับ request เล็ก ๆ |
| Live streaming/broadcast ไปยังผู้ชมจำนวนมาก | มักใช้ UDP-based protocol | เข้ากับสถานการณ์ "ข้อมูลใหม่กว่ามีค่ามากกว่าข้อมูลเก่าที่มาช้า" เหมือนเกม |

โดยสรุปสั้นที่สุด: **TCP เมื่อความถูกต้องสมบูรณ์ของข้อมูลสำคัญที่สุด, UDP เมื่อความเร็ว/ความหน่วงต่ำสำคัญกว่าการรับ
ประกันว่าทุก byte จะมาถึง** — โปรเจกต์ระดับแอปพลิเคชันส่วนใหญ่ที่ไม่ใช่ real-time media หรือเกม ควรเลือก TCP เป็นค่า
เริ่มต้นเสมอ เพราะความซับซ้อนที่ต้องจัดการเอง (การจัดลำดับ packet, ตรวจจับ packet หาย, ส่งซ้ำ) เมื่อใช้ UDP นั้นสูงมาก
และ TCP จัดการให้หมดแล้วโดยที่แอปพลิเคชันไม่ต้องรับรู้เลย

### 49.11 Timeout บน Network Operation ด้วย `tokio::time::timeout()`

ปัญหาที่พบบ่อยมากในโค้ด networking จริงคือ: **ถ้าฝั่งตรงข้ามไม่ตอบสนองเลย (เครื่องค้าง, network หลุดแบบเงียบ ๆ,
โปรแกรมฝั่งนั้นค้าง) `.read().await` หรือ `.recv_from().await` ของเราจะรอ "ตลอดกาล" โดยไม่มีการแจ้งเตือนอะไรเลย**
เพราะ future ของ operation เหล่านี้จะยัง "ไม่พร้อม" (pending) ต่อไปเรื่อย ๆ จนกว่าจะมีข้อมูลมาจริง ๆ หรือ connection
ถูกปิดอย่างชัดเจน (ซึ่งถ้าฝั่งตรงข้ามค้างไปเลยโดยไม่ปิด connection อะไรจะไม่เกิดขึ้นเลย) — โปรแกรมที่ไม่มีการป้องกัน
เรื่องนี้จะมี task ค้างรอตลอดกาล กิน resource (แม้จะน้อย เพราะ task ที่ pending ไม่กิน CPU แต่ก็ยังกิน memory และทำให้
โปรแกรมดูเหมือน "แฮงค์" จากมุมมองผู้ใช้)

`tokio::time::timeout(duration, future)` คือทางออก: มันครอบ future ใดๆ ไว้ แล้วคืนค่าเป็น `Result<T, Elapsed>` — ถ้า
future ข้างในเสร็จก่อนเวลาที่กำหนดจะได้ `Ok(ค่าที่ future คืน)` แต่ถ้าเวลาหมดก่อนจะได้ `Err(Elapsed)` และ future ข้าง
ในจะถูก**ยกเลิก (cancel)** ทันที (ทวนความรู้เรื่อง cancellation ของ future จาก Part 47/48):

```rust
use tokio::io::AsyncReadExt;
use tokio::net::TcpStream;
use tokio::time::{timeout, Duration};

async fn read_with_deadline(stream: &mut TcpStream) -> std::io::Result<()> {
    let mut buf = [0u8; 64];

    match timeout(Duration::from_millis(300), stream.read(&mut buf)).await {
        Ok(Ok(0)) => println!("peer ปิด connection"),
        Ok(Ok(n)) => println!("ได้ {n} bytes"),
        Ok(Err(e)) => println!("read error: {e}"),
        Err(_elapsed) => println!("หมดเวลา! server ไม่ตอบภายใน 300ms"),
    }

    Ok(())
}
```

ผลลัพธ์จริงจากการทดสอบกับ server ที่ตั้งใจ "เงียบ" (accept connection แล้วไม่ส่งอะไรกลับมาเลย):

```
เริ่มรอ read() โดยจำกัดเวลาไว้ 300ms...
หมดเวลา! server ไม่ตอบภายใน 300ms ตามที่คาด
```

สังเกต **`match` ที่มีวงเล็บซ้อนกันสองชั้น: `Ok(Ok(...))`, `Ok(Err(...))`, `Err(...)`** — นี่คือรูปแบบที่พบเจอเสมอเมื่อ
ครอบ `io::Result<T>` (ผลลัพธ์จริงของ operation) ไว้ด้วย `timeout()` ที่คืน `Result<io::Result<T>, Elapsed>` (ผลลัพธ์
ของการ "แข่งกับเวลา"): **`Err(Elapsed)` ชั้นนอก** หมายถึง "หมดเวลาก่อนที่ operation จะเสร็จเลย" (ไม่รู้ด้วยซ้ำว่า
operation จะสำเร็จหรือล้มเหลว เพราะมันถูกยกเลิกไปก่อนจะรู้ผล) ส่วน **`Ok(Err(e))`** หมายถึง "operation เสร็จทันเวลา
แต่ตัว operation เองล้มเหลว" (เช่น connection ถูกรีเซ็ตจริง ๆ) — ทั้งสองกรณีต่างกันโดยพื้นฐาน และโค้ดที่ดีควรแยกจัดการ
ทั้งสองแบบให้ชัดเจน ไม่ใช่รวบมาเป็น error แบบเดียวกันหมด (เพราะ "หมดเวลา" อาจแปลว่า "ลองใหม่อีกครั้งได้" แต่
"connection reset" อาจแปลว่า "ต้องเชื่อมต่อใหม่ทั้งหมด" ซึ่งเป็นการตอบสนองที่ต่างกัน)

**เชื่อมโยงกับ Part 48**: จำได้ว่า Part 48 แนะนำ `tokio::select!` สำหรับ "แข่งกันระหว่างหลาย future เอาตัวที่เสร็จ
ก่อน" — `timeout()` แท้จริงแล้ว **คือ syntax สะดวกที่สร้างขึ้นจากกลไกเดียวกันกับ `select!` เป๊ะ ๆ** ข้างใต้
`timeout(duration, future)` ทำงานเทียบเท่ากับการเขียน `select!` แข่งกันระหว่าง future ที่คุณส่งเข้ามา กับ future ของ
`tokio::time::sleep(duration)` — ตัวไหนเสร็จก่อนก็ชนะ ถ้า `sleep` เสร็จก่อน (เวลาหมด) `timeout()` จะคืน `Err(Elapsed)`
และยกเลิก future ตัวแรกทิ้งไป — พูดอีกแบบคือ ทุกครั้งที่คุณเห็น `timeout()` ให้จำไว้ว่ามันคือ **"`select!` ระหว่าง
งานที่ต้องการ กับ ตัวจับเวลา"** ที่ Tokio เตรียม wrapper สะดวก ๆ ไว้ให้ ไม่ต้องเขียน `select!` มือเปล่าเองทุกครั้งที่
ต้องการ deadline แบบนี้

### 49.12 ตัวอย่างจริง: TCP Chat Server หลายผู้ใช้แบบสมบูรณ์

มารวมทุกอย่างที่เรียนมาทั้งบทเข้าด้วยกันเป็นตัวอย่างที่สมบูรณ์: **TCP chat server** ที่รองรับหลาย client เชื่อมต่อ
พร้อมกัน แต่ละคนพิมพ์ข้อความส่งเข้ามา (newline-delimited จากหัวข้อ 49.7) แล้ว server broadcast ข้อความนั้นไปให้ทุกคน
ที่เชื่อมต่ออยู่ (ยกเว้นผู้ส่งเอง) โดยใช้ shared state แบบ `Arc<Mutex<T>>` จากหัวข้อ 49.9:

```rust
use std::sync::{Arc, Mutex};
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::mpsc::{self, UnboundedSender};

type Clients = Arc<Mutex<Vec<(u64, UnboundedSender<String>)>>>;

async fn handle_client(
    id: u64,
    socket: TcpStream,
    peer: std::net::SocketAddr,
    clients: Clients,
) -> std::io::Result<()> {
    let (reader, mut writer) = socket.into_split();
    let (tx, mut rx) = mpsc::unbounded_channel::<String>();
    clients.lock().unwrap().push((id, tx));
    println!("client {id} ({peer}) เข้าร่วมห้อง");

    // task แยกสำหรับ "เขียน" ข้อความ broadcast ที่ส่งมาทาง channel ออกไปยัง socket จริง
    let writer_task = tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            if writer.write_all(msg.as_bytes()).await.is_err() {
                break;
            }
        }
    });

    let mut lines = BufReader::new(reader).lines();
    while let Some(line) = lines.next_line().await? {
        println!("client {id} พิมพ์: {line}");
        let broadcast_msg = format!("[client {id}]: {line}\n");

        // lock สั้น ๆ แค่ตอน iterate ส่งเข้า channel (mpsc send เป็น sync ไม่มี .await ค้างระหว่างถือ lock)
        let list = clients.lock().unwrap();
        for (other_id, sender) in list.iter() {
            if *other_id != id {
                let _ = sender.send(broadcast_msg.clone());
            }
        }
    }

    println!("client {id} ออกจากห้อง");
    clients.lock().unwrap().retain(|(cid, _)| *cid != id);
    writer_task.abort();
    Ok(())
}

async fn run_chat_server(addr: &str, clients: Clients) -> std::io::Result<()> {
    let listener = TcpListener::bind(addr).await?;
    let mut next_id = 0u64;
    loop {
        let (socket, peer) = listener.accept().await?;
        next_id += 1;
        let id = next_id;
        let clients = Arc::clone(&clients);
        tokio::spawn(async move {
            if let Err(e) = handle_client(id, socket, peer, clients).await {
                eprintln!("client {id} error: {e}");
            }
        });
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let addr = "127.0.0.1:7900";
    let clients: Clients = Arc::new(Mutex::new(Vec::new()));
    run_chat_server(addr, clients).await
}
```

ทีนี้มาจำลองสาม client (alice, bob, carol) เชื่อมต่อเข้ามาคุยกัน — โครงสร้าง client แต่ละตัวมีสองส่วนที่ทำงานคู่กัน:
**task สำหรับอ่าน** ข้อความ broadcast ที่เข้ามาแล้ว print ออกมาเรื่อย ๆ (ทำงานเป็น background ตลอดเวลาที่เชื่อมต่ออยู่)
กับ **flow หลักสำหรับพิมพ์และส่งข้อความ** ตาม script ที่กำหนดเวลาไว้:

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpStream;
use tokio::time::{sleep, Duration};

async fn simulated_client(
    addr: &str,
    label: &'static str,
    script: Vec<(u64, String)>,
) -> std::io::Result<()> {
    let stream = TcpStream::connect(addr).await?;
    let (reader, mut writer) = stream.into_split();

    // task พื้นหลัง: คอยอ่านข้อความที่ broadcast มาจาก client คนอื่น แล้ว print ออกมาเรื่อย ๆ
    tokio::spawn(async move {
        let mut lines = BufReader::new(reader).lines();
        while let Ok(Some(line)) = lines.next_line().await {
            println!("[{label} received] {line}");
        }
    });

    for (delay_ms, msg) in script {
        sleep(Duration::from_millis(delay_ms)).await;
        writer.write_all(format!("{msg}\n").as_bytes()).await?;
    }

    sleep(Duration::from_millis(500)).await; // เผื่อเวลาให้ broadcast รอบสุดท้ายมาถึง
    Ok(())
}
```

orchestration หลักที่รัน server เป็น background task แล้วปล่อยสาม client คุยกัน:

```rust
#[tokio::main]
async fn main() -> std::io::Result<()> {
    let addr = "127.0.0.1:17883";
    let clients: Clients = Arc::new(Mutex::new(Vec::new()));

    let server_clients = Arc::clone(&clients);
    tokio::spawn(async move {
        if let Err(e) = run_chat_server(addr, server_clients).await {
            eprintln!("chat_server fatal: {e}");
        }
    });
    sleep(Duration::from_millis(100)).await;

    let alice = simulated_client(
        addr,
        "alice",
        vec![(100, "hello everyone".to_string()), (600, "how are you?".to_string())],
    );
    let bob = simulated_client(addr, "bob", vec![(300, "hi alice".to_string())]);
    let carol = simulated_client(addr, "carol", vec![(450, "hey folks".to_string())]);

    let (r1, r2, r3) = tokio::join!(alice, bob, carol);
    r1?;
    r2?;
    r3?;

    Ok(())
}
```

ผลลัพธ์จริงจากการรัน (จับภาพเต็มโดยไม่ตัดทอน):

```
client 3 (127.0.0.1:45720) เข้าร่วมห้อง
client 1 (127.0.0.1:45702) เข้าร่วมห้อง
client 2 (127.0.0.1:45718) เข้าร่วมห้อง
client 1 พิมพ์: hello everyone
[carol received] [client 1]: hello everyone
[bob received] [client 1]: hello everyone
client 2 พิมพ์: hi alice
[carol received] [client 2]: hi alice
[alice received] [client 2]: hi alice
client 3 พิมพ์: hey folks
[alice received] [client 3]: hey folks
[bob received] [client 3]: hey folks
client 1 พิมพ์: how are you?
[carol received] [client 1]: how are you?
[bob received] [client 1]: how are you?
client 2 ออกจากห้อง
client 3 ออกจากห้อง
```

วิเคราะห์ผลลัพธ์ทีละจุด:

1. **"client 3 ... เข้าร่วมห้อง" ปรากฏก่อน "client 1"** ทั้งที่ในโค้ด `tokio::join!(alice, bob, carol)` เขียน
   `alice` ไว้ก่อน — นี่คือหลักฐานว่า **`tokio::join!` ไม่การันตีลำดับการทำงานจริงของ future ที่ให้มันรอ** (ตามที่
   Part 48 อธิบายไว้: `join!` แค่รอให้ **ทุกตัว**เสร็จ ไม่ได้บอกว่าใครต้องเสร็จก่อนใคร) ลำดับการ `connect()` จริง
   ขึ้นกับ scheduler ของ Tokio ว่าจะสลับไปรัน task ไหนก่อนในรอบแรก
2. **เมื่อ alice (client 1) พิมพ์ "hello everyone" ทั้ง carol และ bob ได้รับข้อความนั้นจริง** (`[carol received]`
   และ `[bob received]`) แต่ **alice เองไม่ได้รับข้อความของตัวเอง** — ตรงตามที่โค้ดตั้งใจไว้ในเงื่อนไข
   `if *other_id != id` ที่ตัดผู้ส่งออกจากรายชื่อผู้รับ broadcast (การไม่ echo ข้อความกลับไปให้ผู้พิมพ์เองเป็น
   พฤติกรรมมาตรฐานของ chat application ทั่วไป เพราะผู้พิมพ์เห็นข้อความของตัวเองบนหน้าจอตัวเองอยู่แล้วจากการพิมพ์)
3. **ทุกข้อความที่ถูกส่งไปถึงผู้รับที่ถูกต้องครบทุกคนในทุกรอบ** (bob/carol ได้ข้อความของ alice, alice/carol ได้
   ข้อความของ bob, alice/bob ได้ข้อความของ carol) — พิสูจน์ว่ากลไก broadcast ผ่าน `Arc<Mutex<Vec<...>>>` +
   `mpsc::UnboundedSender` ทำงานถูกต้องข้าม task จริง ทุกคนเชื่อมต่ออยู่บน TCP connection คนละเส้นแยกกันโดยสิ้นเชิง
4. **สังเกตว่าไม่มีบรรทัด "client 1 ออกจากห้อง" ปรากฏในผลลัพธ์** — นี่ไม่ใช่บัค แต่เป็นรายละเอียดที่น่าสนใจเรื่อง
   **graceful shutdown**: alice เป็นคนสุดท้ายที่ปิด connection (ตาม script ที่ตั้งเวลาไว้นานสุด) และมันเกิดขึ้นในช่วง
   เวลาไล่เลี่ยกับตอนที่ `tokio::join!` ใน `main()` เสร็จสมบูรณ์พอดี ทำให้โปรแกรม `main` จบและ runtime ถูกปิดตัวลง
   **ก่อนที่** task ฝั่ง server จะมีโอกาสประมวลผลและ print ข้อความ "ออกจากห้อง" ของ alice ให้เสร็จทัน — นี่คือตัวอย่าง
   จริงของ**การแข่งขันเรื่องเวลาระหว่าง task ต่าง ๆ ตอนโปรแกรมกำลังจะปิดตัว** ซึ่งเป็นประเด็นสำคัญของระบบ concurrent
   จริง (server ที่ดีในทาง production มักมีกลไก graceful shutdown ที่รอให้ task ที่กำลังทำงานอยู่จบให้เรียบร้อยก่อน
   ปิดโปรแกรมจริง ๆ ซึ่งเป็นหัวข้อที่ลึกเกินขอบเขตของบทนี้ แต่ควรรู้ไว้ว่ามันเป็นปัญหาที่มีอยู่จริง)

ตัวอย่างนี้ยังไม่สมบูรณ์แบบระดับ production (ไม่มีการ authenticate ผู้ใช้, ไม่มีชื่อผู้ใช้ที่อ่านง่าย ใช้แค่หมายเลข
id, ไม่มี rate limiting ป้องกัน spam, การ broadcast แบบ `Vec` ที่ต้อง iterate ทุกครั้งจะช้าลงเมื่อมีผู้ใช้เป็นพันคน
พร้อมกัน) แต่มันแสดงให้เห็น**แพทเทิร์นหลักที่ถูกต้อง**ของการสร้าง concurrent networking application ด้วย Tokio: หนึ่ง
task ต่อหนึ่ง connection, ใช้ channel ส่งข้อความข้าม task แทนการแชร์ memory ตรง ๆ ทุกจุด, และใช้ `Arc<Mutex<T>>` เฉพาะ
ตอนที่จำเป็นต้องแบ่งปันโครงสร้างข้อมูล "รายชื่อ" ที่ต้องอัปเดตร่วมกันจริง ๆ — Part 50 จะพาไปดู `tokio::sync::broadcast`
channel ซึ่งเป็นชนิดข้อมูลที่ Tokio สร้างมาเพื่อแก้ปัญหา "กระจายข้อความหนึ่งชุดไปยังหลายผู้รับ" นี้โดยเฉพาะ ซึ่งจะทำให้
โค้ด chat server แบบนี้เขียนได้กระชับกว่านี้อีกมาก (ไม่ต้องจัดการ `Vec` ของ sender ด้วยมือเองเลย)

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: ลืม `.await` — โค้ด compile ผ่านแต่ไม่มีอะไรเกิดขึ้นเลย

จาก Part 46/47 เราเรียนว่า `async fn` ไม่ทำอะไรเลยจนกว่า future ที่มันคืนค่าออกมาจะถูก `.await` หรือ poll — ข้อผิดพลาด
ที่พบบ่อยมากตอนเขียน networking code (โดยเฉพาะตอนรีบเขียนหรือ copy-paste โค้ดมาแก้) คือลืมใส่ `.await` ต่อท้าย method
ของ `AsyncReadExt`/`AsyncWriteExt`:

```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpStream;

async fn send_greeting(stream: &mut TcpStream) {
    stream.write_all(b"hello"); // ลืม .await!
}
```

คำเตือนจริงจาก compiler:

```
warning: unused `tokio::io::util::write_all::WriteAll` that must be used
 --> src/main.rs:5:5
  |
5 |     stream.write_all(b"hello"); // ลืม .await!
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: futures do nothing unless you `.await` or poll them
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
help: use `let _ = ...` to ignore the resulting value
  |
5 |     let _ = stream.write_all(b"hello"); // ลืม .await!
  |     +++++++
```

จุดอันตรายที่สุดของกับดักนี้คือ **มันเป็นแค่ "warning" ไม่ใช่ "error"** — โปรแกรม **compile ผ่านและรันได้ปกติ** แต่
`write_all(b"hello")` จะไม่เขียนอะไรออกไปยัง socket เลยแม้แต่ byte เดียว เพราะ future ที่สร้างขึ้นถูกสร้างแล้วก็ถูก
ทิ้ง (drop) ทันทีโดยไม่เคยถูก poll แม้แต่ครั้งเดียว — ฝั่งที่รออ่านข้อมูลอยู่จะไม่ได้รับอะไรเลยและอาจค้างรออยู่ตลอดกาล
(เว้นแต่มี `timeout()` ครอบไว้ตามหัวข้อ 49.11) วิธีป้องกันที่ดีที่สุดคือ **อย่าปิด warning ของ `#[warn(unused_must_use)]`
เด็ดขาด** และหมั่นรัน `cargo clippy` (จาก Part 5) เพื่อจับ warning เหล่านี้ให้ไวก่อนโค้ดจะถูกดันขึ้น production

### กับดักที่ 2: สมมติว่าหนึ่ง `write_all()` เท่ากับหนึ่ง `read()` เสมอ (Framing Bug)

นี่คือกับดักที่สำคัญที่สุดของทั้งบทและเป็นสาเหตุของบัคระดับ production จริงจำนวนมาก — ทวนจากหัวข้อ 49.6: TCP ไม่
การันตีขอบเขตข้อความ ลองดูตัวอย่างโปรโตคอลที่ดูเผิน ๆ เหมือนจะโอเค แต่จริง ๆ มีบัคซ่อนอยู่:

```rust
// โปรโตคอลสมมติ: ส่งความยาวข้อความ (เป็นตัวเลข ASCII) ตามด้วยตัวข้อความ เช่น "5:hello"
// เวอร์ชันนี้ "ผิด" เพราะสมมติว่า read() ครั้งเดียวจะได้ข้อความครบเสมอ
async fn broken_read_message(stream: &mut tokio::net::TcpStream) -> std::io::Result<String> {
    use tokio::io::AsyncReadExt;
    let mut buf = [0u8; 1024];
    let n = stream.read(&mut buf).await?; // อ่านครั้งเดียว แล้วสมมติว่าได้ข้อความครบ!
    Ok(String::from_utf8_lossy(&buf[..n]).to_string())
}
```

ถ้าข้อความสั้นกว่า MTU และมาถึงในการ `write_all()` เดียวจาก peer เดียว ฟังก์ชันนี้ **อาจดูเหมือนทำงานถูกต้องตอน
ทดสอบบนเครื่อง localhost** (เพราะ loopback interface มักส่งข้อมูลมาถึงในครั้งเดียวสำหรับข้อความสั้น ๆ) แต่พอขึ้น
production จริงที่ network มี latency สูงกว่า หรือข้อความมีขนาดใหญ่ขึ้น หรือแค่ peer ส่งสองข้อความติดกันเร็ว ๆ (ตาม
ที่พิสูจน์ด้วยผลลัพธ์จริงในหัวข้อ 49.6 ว่า `.read()` ได้ `"HELLOWORLD"` รวมกันมา) ฟังก์ชันนี้จะพังทันที: อาจ parse
ข้อความสองก้อนรวมกันเป็นก้อนเดียว (ทำให้ตัวเลขความยาวที่คาดหวังไม่ตรงกับความยาวจริง) หรือได้ข้อความไม่ครบแล้วเอาไป
แปลงเป็น `String`/parse เป็น JSON (ถ้าใช้ serde ในบทหลัง ๆ) จนได้ error "unexpected end of input" ทั้งที่ข้อมูลจริง
ยังไม่ได้ส่งมาไม่ครบเฉย ๆ ไม่ใช่ข้อมูลเสีย

**วิธีแก้ที่ถูกต้องเสมอ**: ใช้ framing ที่ชัดเจนอย่างที่หัวข้อ 49.7 สอนไว้ (newline-delimited ผ่าน `.lines()`) หรือถ้า
เป็น binary protocol ให้ใช้ length-prefixed framing ที่อ่าน header ความยาวให้ครบก่อนด้วย `.read_exact()` แล้วค่อยอ่าน
payload ตามความยาวนั้นให้ครบอีกที (จะฝึกเขียนจริงในแบบฝึกหัดข้อ 4 ท้ายบท) — **กฎทองคือห้าม assume ว่า `.read()`
เดี่ยว ๆ จะได้ "หนึ่งข้อความที่สมบูรณ์" เด็ดขาด ไม่ว่าในสถานการณ์ทดสอบจะดูเหมือนทำงานถูกต้องแค่ไหนก็ตาม**

### กับดักที่ 3: ลืมเช็ค `Ok(0)` จาก `.read()` — Busy Loop กิน CPU 100%

ทวนจากหัวข้อ 49.4: `.read()` คืนค่า `Ok(0)` หมายถึง peer ปิด connection แล้ว (EOF) — ถ้าลูปอ่านข้อมูลไม่เช็คกรณีนี้
และไม่ `break`/`return` ออกจาก loop จะเกิดปัญหาร้ายแรง:

```rust
// เวอร์ชันที่มีบัค: ไม่เช็ค n == 0
async fn buggy_read_loop(mut socket: tokio::net::TcpStream) {
    use tokio::io::AsyncReadExt;
    let mut buf = [0u8; 1024];
    loop {
        let n = socket.read(&mut buf).await.unwrap_or(0);
        // ไม่มีการเช็คว่า n == 0 แล้ว break ออกจาก loop!
        println!("got {n} bytes");
        // ถ้า connection ปิดไปแล้ว read() จะ return Ok(0) ทันทีแบบไม่รอ (ไม่ใช่ async รอจริง)
        // loop นี้จะวนอ่าน 0 byte ซ้ำ ๆ ไม่จบไม่สิ้น กิน CPU 100% ของ thread ที่ task นี้ถูก schedule ไปรัน
    }
}
```

ความอันตรายของกับดักนี้คือ **มันไม่ทำให้โปรแกรม crash หรือแสดง error อะไรเลย** — task นี้จะวนลูปกิน CPU เต็มไปเรื่อย ๆ
อย่างเงียบ ๆ (เพราะ `.read()` บน connection ที่ปิดแล้วจะ resolve เป็น `Ok(0)` ทันทีโดยไม่ต้องรอ ทำให้ loop หมุนเร็ว
ที่สุดเท่าที่ CPU จะทำได้) ถ้าเซิร์ฟเวอร์มี client ปิด connection บ่อย ๆ (ซึ่งเป็นเรื่องปกติมาก) และมีบัคนี้อยู่ CPU
ของเครื่อง server จะค่อย ๆ ถูกกินไปเรื่อย ๆ จนกระทบ connection อื่นทั้งหมดที่ใช้ thread pool ร่วมกัน (ทวนจาก Part 48:
worker thread ของ Tokio มีจำนวนจำกัดเท่ากับ CPU core โดย default — ถ้า task หนึ่งกิน CPU เต็มไม่ยอมปล่อยผ่าน
`.await` point ที่แท้จริง task อื่นบน worker thread เดียวกันจะได้รับผลกระทบตามไปด้วย) วิธีป้องกันคือ **เช็ค `Ok(0)`
แล้ว `break`/`return` ออกจาก loop เสมอ** อย่างที่โค้ดในหัวข้อ 49.4 ทำไว้ตั้งแต่ต้น

### กับดักที่ 4: Block Runtime ด้วยการเรียกฟังก์ชัน Synchronous ใน Async Context

ทวนจาก Part 48: worker thread ของ Tokio มีจำนวนจำกัด ถ้า task ไหนเรียกฟังก์ชันที่ **block thread จริง ๆ** (ไม่ใช่แค่
`.await` ที่ยอมให้ executor สลับงานได้) เช่น `std::thread::sleep()` หรือ I/O แบบ synchronous ของ `std::net`/`std::fs`
โดยตรง (ไม่ใช่เวอร์ชัน `tokio::` ที่เรียนในบทนี้) worker thread นั้นจะถูก **บล็อกทั้ง thread** ทำให้ task อื่น ๆ ที่ถูก
กำหนดให้รันบน thread เดียวกันไม่ได้ทำงานเลยจนกว่าการบล็อกจะจบ:

```rust
// ผิด: เผลอใช้ std::net::TcpListener (synchronous) ปนกับ async fn
async fn broken_listener() {
    let listener = std::net::TcpListener::bind("127.0.0.1:8080").unwrap();
    loop {
        let (socket, _addr) = listener.accept().unwrap(); // บล็อก thread จริง ไม่ใช่ .await!
        // socket ที่ได้ตรงนี้เป็น std::net::TcpStream (synchronous) ใช้ AsyncReadExt/AsyncWriteExt ไม่ได้เลย
        // ...
    }
}
```

โค้ดข้างบนนี้อาจดู "เหมือนทำงานได้" เมื่อทดสอบด้วย client เดียว (เพราะ block ไปเรื่อย ๆ ก่อนจะมี client ต่อเข้ามา
ก็ไม่มีใครสังเกตเห็นความแตกต่าง) แต่พอมี client หลายตัวพร้อมกัน `listener.accept().unwrap()` แบบ synchronous จะ
บล็อก worker thread ทั้งเส้นจนกว่าจะมี connection ใหม่เข้ามา ทำให้ **task อื่นทั้งหมดที่ถูก schedule ไปรันบน worker
thread เดียวกันหยุดชะงักไปด้วย** ไม่ต่างจากที่ Part 48 อธิบายไว้เรื่อง blocking call ทั่วไป — วิธีป้องกันตรงไปตรงมา
คือ **ใช้ `tokio::net::TcpListener`/`TcpStream`/`UdpSocket` เท่านั้นในโค้ด async ทั้งหมดของบทนี้ ห้ามผสมกับเวอร์ชัน
`std::net` แบบ synchronous โดยเด็ดขาด** — ทั้งสองชนิดมีชื่อคล้ายกันมาก (`TcpListener`, `TcpStream`) จึงต้องระวังเรื่อง
`use` statement ให้ import จาก `tokio::net::*` เสมอ ไม่ใช่ `std::net::*` เผลอ

### กับดักที่ 5: ถือ `std::sync::MutexGuard` ข้าม `.await` Point

หัวข้อ 49.9 เน้นไว้แล้วว่า critical section ของ `Arc<Mutex<T>>` ในตัวอย่าง broadcast **ต้องไม่มี `.await` แทรกอยู่
ข้างในเลย** — มาดูว่าถ้าฝ่าฝืนกฎนี้จะเกิดอะไรขึ้น:

```rust
use std::sync::{Arc, Mutex};
use tokio::time::{sleep, Duration};

async fn bad_increment(shared: Arc<Mutex<i32>>) {
    let mut guard = shared.lock().unwrap();
    *guard += 1;
    sleep(Duration::from_millis(10)).await; // .await ขณะยังถือ MutexGuard อยู่!
    println!("{}", *guard);
}

#[tokio::main]
async fn main() {
    let shared = Arc::new(Mutex::new(0));
    tokio::spawn(bad_increment(shared)); // spawn บังคับ future ต้อง Send
}
```

error จริงจาก compiler:

```
error: future cannot be sent between threads safely
   --> src/main.rs:14:18
    |
 14 |     tokio::spawn(bad_increment(shared)); // spawn บังคับ future ต้อง Send
    |                  ^^^^^^^^^^^^^^^^^^^^^ future returned by `bad_increment` is not `Send`
    |
    = help: within `impl Future<Output = ()>`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:7:38
    |
  5 |     let mut guard = shared.lock().unwrap();
    |         --------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
  6 |     *guard += 1;
  7 |     sleep(Duration::from_millis(10)).await; // .await ขณะยังถือ MutexGuard อยู่!
    |                                      ^^^^^ await occurs here, with `mut guard` maybe used later
note: required by a bound in `tokio::spawn`
   --> tokio-1.53.1/src/task/spawn.rs:176:21
    |
174 |     pub fn spawn<F>(future: F) -> JoinHandle<F::Output>
    |            ----- required by a bound in this function
175 |     where
176 |         F: Future + Send + 'static,
    |                     ^^^^ required by this bound in `spawn`
```

สาเหตุเชิงลึก: `std::sync::MutexGuard<'_, T>` **จงใจไม่ implement `Send`** (Part 40 อธิบายไว้ว่า `Send` หมายถึง
"ย้ายข้าม thread ได้อย่างปลอดภัย") เพราะ `MutexGuard` ผูกติดกับ mutex ระดับ OS ที่บาง platform (เช่น pthread mutex
บน Linux) มีข้อกำหนดว่า **ต้อง unlock จาก thread เดียวกันกับที่ lock ไว้เท่านั้น** — แต่ future ที่มี `.await` แทรกอยู่
ระหว่างถือ guard นั้นมีโอกาสถูก executor "ย้าย" ไปทำงานต่อบน worker thread คนละตัวกันได้ (ทวนจาก Part 48: executor
แบบ multi-thread ของ Tokio อาจสลับ thread ที่ task รันอยู่ทุกครั้งที่ task ถูก resume หลัง `.await`) ถ้ายอมให้ทำแบบนี้
ได้ อาจเกิดการ unlock mutex จาก thread ที่ผิดจริง ๆ ซึ่งเป็น undefined behavior บนบาง platform — Rust จึงป้องกันไว้
ตั้งแต่ compile time ด้วย trait bound `Send` ของ `tokio::spawn`

**วิธีแก้**: ให้ lock อยู่ในช่วงสั้น ๆ ที่ไม่มี `.await` เลย (เหมือนที่ตัวอย่าง chat server ในหัวข้อ 49.9/49.12 ทำ
— lock, iterate ส่งเข้า channel แบบ sync ล้วน ๆ, แล้วปล่อย lock ก่อนจะมี `.await` ใด ๆ) หรือถ้าจำเป็นต้อง `.await`
จริง ๆ ระหว่างถือ lock (เช่นต้องเขียนไฟล์ขณะถือ state) ให้เปลี่ยนไปใช้ `tokio::sync::Mutex` (เวอร์ชัน async ของ
Tokio เอง ที่จะเรียนโดยละเอียดใน Part 50) ซึ่ง implement `Send` ให้ guard ของมันได้อย่างปลอดภัย เพราะมันไม่ได้ผูกกับ
OS mutex ระดับต่ำแบบเดียวกัน — แต่ข้อแลกเปลี่ยนคือ `tokio::sync::Mutex` ช้ากว่า `std::sync::Mutex` เล็กน้อยเสมอ
(เพราะมันมี state machine ของตัวเองสำหรับจัดการ task ที่รอ) ดังนั้น **หลักปฏิบัติที่ดีคือใช้ `std::sync::Mutex`
เป็นค่าเริ่มต้นเสมอเมื่อ critical section สั้นและไม่มี `.await` และเปลี่ยนไปใช้ `tokio::sync::Mutex` เฉพาะเมื่อ
จำเป็นต้อง `.await` จริง ๆ ระหว่างถือ lock เท่านั้น**

### กับดักที่ 6: ใช้ `?` ผิดที่ใน Accept Loop จนเซิร์ฟเวอร์ทั้งตัวล้มเพราะ Client เดียว

ข้อผิดพลาดเชิง error-handling ที่พบบ่อยคือ ปล่อยให้ error ของ**การจัดการ client รายตัว** ทำให้ **loop `.accept()`
หลักของเซิร์ฟเวอร์ทั้งตัวหยุดทำงาน**:

```rust
// ผิด: ใช้ ? กับ error ที่เกิดจากการจัดการ client รายตัว ไม่ใช่ error ของ accept() เอง
async fn broken_server(addr: &str) -> std::io::Result<()> {
    use tokio::io::{AsyncReadExt, AsyncWriteExt};
    use tokio::net::TcpListener;

    let listener = TcpListener::bind(addr).await?;
    loop {
        let (mut socket, _peer) = listener.accept().await?;
        let mut buf = [0u8; 1024];
        let n = socket.read(&mut buf).await?; // ถ้า client ตัดการเชื่อมต่อกะทันหัน error นี้จะ ? ออกจาก loop ทั้งหมด!
        socket.write_all(&buf[..n]).await?;
    }
}
```

ปัญหาคือ `socket.read(&mut buf).await?` ใน loop นี้**ไม่ได้อยู่ใน `tokio::spawn` แยกต่างหาก** — มันรันอยู่ใน task
เดียวกันกับ loop `.accept()` หลัก (เพราะไม่มีการ spawn) ถ้า client ตัวใดตัวหนึ่งตัดการเชื่อมต่อกะทันหันจนเกิด
`io::Error` (เช่น `ConnectionReset`) `?` operator จะส่ง error นั้นออกจากทั้งฟังก์ชัน `broken_server` ทันที — **ทำให้
เซิร์ฟเวอร์ทั้งตัวหยุดรับ connection ใหม่ไปเลย เพราะ client เพียงคนเดียวมีปัญหา** ซึ่งเป็นพฤติกรรมที่ยอมรับไม่ได้เลย
สำหรับ production server (client หนึ่งคนไม่ควรมีอำนาจ "ปิด" บริการทั้งระบบได้)

**วิธีแก้ที่ถูกต้อง** คือสิ่งที่โค้ดในหัวข้อ 49.4/49.8/49.12 ทำมาตลอดทั้งบท: จัดการ error ของ**การอ่าน/เขียนแต่ละ
connection ภายใน `tokio::spawn` task ของมันเอง** ด้วย `match`/`if let Err(...)` แล้ว `return`/`break` ออกจาก**เฉพาะ
task นั้น** โดยไม่กระทบ loop `.accept()` หลักเลย ส่วน `?` ที่ใช้กับ `listener.accept()` เองยังสมควรใช้ได้ เพราะ error
จากการ `accept()` มักหมายถึงปัญหาระดับ OS/ระบบจริง ๆ (เช่น หมด file descriptor) ที่ทำให้เซิร์ฟเวอร์ทำงานต่อไม่ได้อยู่
แล้ว ต่างจาก error ของการอ่าน/เขียนกับ client รายตัวที่เป็นเรื่องปกติที่เกิดขึ้นได้ทุกวันและไม่ควรกระทบ client คนอื่น
เลย — หลักการนี้คือการนำ "แนวคิดเรื่อง scope ของ error" จาก Part 12/30 มาประยุกต์กับ concurrency: **error ที่เกิดกับ
งานหนึ่งงาน ควรจำกัดผลกระทบไว้แค่งานนั้น ไม่ควรลามไปงานอื่นที่ไม่เกี่ยวข้องกัน**

## แบบฝึกหัด (Exercises)

1. **(ง่าย) Echo Server ที่แปลงเป็นตัวพิมพ์ใหญ่**: แก้ไข echo server จากหัวข้อ 49.4 ให้แปลงข้อความเป็นตัวพิมพ์ใหญ่
   ก่อน echo กลับ (เช่น client ส่ง `"hello"` ต้องได้รับ `"HELLO"` กลับมา) — ใช้ `.to_ascii_uppercase()` กับ
   `&buf[..n]` ที่เป็น `&[u8]` โดยตรง (ไม่ต้องแปลงเป็น `String` ก่อนก็ได้) *Hint*: ทวนจาก Part 14 ว่าทำไมควรใช้
   `.to_ascii_uppercase()` แทน `.to_uppercase()` ธรรมดาถ้าข้อมูลอาจมีอักขระที่ไม่ใช่ ASCII ปนอยู่ — ลองคิดว่าถ้า client
   ส่งคำภาษาไทยเข้ามา จะเกิดอะไรขึ้นถ้าคุณ uppercase ทีละ byte แบบ `&[u8]` ตรง ๆ โดยไม่ผ่านการตรวจสอบ UTF-8 boundary
   ก่อน (เชื่อมกับความรู้เรื่อง byte boundary จาก Part 8/14)

2. **(กลาง) เพิ่มคำสั่ง `/quit` และ Idle Timeout**: ขยาย line-based server จากหัวข้อ 49.7/49.8 ให้ (ก) รองรับ
   บรรทัดพิเศษ `/quit` — เมื่อ client ส่งบรรทัดนี้มา ให้ server ตอบ `"Goodbye!\n"` แล้วปิด connection อย่างสุภาพ
   (ไม่ใช่แค่รอให้ client ปิดเอง) และ (ข) ครอบการเรียก `.next_line()` ด้วย `tokio::time::timeout()` จากหัวข้อ 49.11
   — ถ้า client ไม่ส่งอะไรมาเลยเกิน 10 วินาที ให้ server ปิด connection นั้นทิ้งโดยอัตโนมัติ (ป้องกัน connection ที่
   "แขวน" ค้างไว้เฉย ๆ กินทรัพยากรเซิร์ฟเวอร์โดยไม่จำเป็น) *Hint*: โครง `match timeout(dur, lines.next_line()).await`
   จะได้ผลลัพธ์สามชั้นแบบเดียวกับตัวอย่างในหัวข้อ 49.11 (`Ok(Ok(Some(line)))`, `Ok(Ok(None))`, `Ok(Err(e))`,
   `Err(_elapsed)`) ลองไล่ให้ครบทุกกรณีด้วย `match`

3. **(ยาก) Chat Server แบบมีชื่อผู้ใช้และคำสั่ง `/list`**: ขยาย chat server จากหัวข้อ 49.12 ให้ (ก) เมื่อ client
   เชื่อมต่อเข้ามาใหม่ ให้ส่งชื่อผู้ใช้เป็นบรรทัดแรกก่อนอะไรทั้งหมด (server ต้องอ่านบรรทัดแรกนั้นแยกจาก loop หลัก
   เพื่อเก็บไว้เป็นชื่อ) แทนการใช้แค่หมายเลข `id` ตัวเลข และ (ข) เพิ่ม `Vec<(u64, String, UnboundedSender<String>)>`
   เก็บชื่อผู้ใช้ควบคู่ไปด้วย แล้วให้ข้อความ broadcast แสดงชื่อจริงแทนหมายเลข (เช่น `"[alice]: hello"` แทน
   `"[client 1]: hello"`) และ (ค) เพิ่มคำสั่งพิเศษ `/list` ที่ตอบกลับเฉพาะผู้ส่งคำสั่งนั้น (ไม่ broadcast) ด้วยรายชื่อ
   ผู้ใช้ที่เชื่อมต่ออยู่ทั้งหมด ณ ขณะนั้น (เช่น `"Online: alice, bob, carol\n"`) *Hint*: การตอบกลับเฉพาะผู้ส่งคำสั่ง
   (ไม่ broadcast) ทำได้ง่าย ๆ ด้วยการเขียนตรงไปยัง `writer` ของ connection นั้นเองโดยไม่ต้องผ่าน channel เลย
   เพราะ writer half ของ connection นั้นยังอยู่ในมือของ task เดียวกันกับที่กำลังอ่านคำสั่งอยู่

4. **(ยาก/ประยุกต์ใช้งานจริง) Length-Prefixed Binary Framing**: ปัญหาของ newline-delimited protocol (หัวข้อ 49.7)
   คือมันใช้ไม่ได้กับข้อมูล binary ที่อาจมี byte `\n` (0x0A) ปนอยู่ในเนื้อข้อมูลจริง (เช่น รูปภาพ, ไฟล์เสียง, ข้อมูล
   ที่ serialize มาแล้ว) เพราะจะถูกตีความผิดว่าเป็นจุดจบข้อความ — ให้ออกแบบ framing แบบใหม่ที่ใช้ **4-byte big-endian
   `u32`** บอกความยาวของ payload ที่ตามมา (แทนการหาตัวคั่น) แล้วเขียนฟังก์ชัน `write_frame(writer, payload: &[u8])`
   และ `read_frame(reader) -> io::Result<Vec<u8>>` ที่ใช้ `.write_all()`/`.read_exact()` (ทวนจากหัวข้อ 49.2 ตาราง
   สรุป method) ให้ทำงานถูกต้องไม่ว่า payload จะมี byte อะไรปนอยู่ก็ตาม แล้วทดสอบส่งข้อมูล binary ที่มี byte `0x0A`
   ปนอยู่ตรงกลางจริง ๆ เพื่อพิสูจน์ว่า framing แบบนี้ไม่มีปัญหาเหมือน newline-delimited *Hint*: ใช้
   `u32::to_be_bytes()` แปลงความยาวเป็น 4 bytes ก่อนเขียน และ `u32::from_be_bytes([u8; 4])` แปลงกลับตอนอ่าน —
   `read_exact(&mut [0u8; 4])` การันตีว่าจะได้ header ครบ 4 bytes เสมอก่อนจะรู้ว่าต้องอ่าน payload อีกกี่ byte ต่อ
   (ต่างจาก `.read()` เดี่ยว ๆ ที่ไม่การันตีแบบนั้นตามที่กับดักที่ 2 อธิบายไว้) ลองคิดต่อว่า length-prefixed framing
   แบบนี้มีข้อดี/ข้อเสียอะไรเทียบกับ newline-delimited ในแง่ความง่ายต่อการ debug ด้วยตาเปล่า (เช่นการใช้
   `nc`/`telnet` ทดสอบ) กับความสามารถในการรองรับข้อมูล binary

## สรุป

บทนี้พาคุณจากการอุ่นเครื่องด้วย async file I/O ไปสู่แกนกลางของเหตุผลที่ Tokio และ async/await ถือกำเนิดขึ้นในโลก
Rust จริง ๆ: **networking** — เราเรียนรู้ว่า `AsyncReadExt`/`AsyncWriteExt` เป็น extension trait ชุดเดียวที่ใช้ได้ทั้ง
กับไฟล์และ TCP socket, เห็นว่า `tokio::spawn` หนึ่ง task ต่อหนึ่ง connection (ที่ Part 48 สอนไว้) คือแพทเทิร์นต้นแบบ
ของ concurrent server ทุกตัวในโลกจริง, พิสูจน์ด้วยโค้ดและผลลัพธ์จริงว่า TCP เป็น byte stream ที่ไม่มีขอบเขตข้อความ
ในตัวเอง (ปัญหา framing ที่ทั้ง merge และ split เกิดขึ้นได้จริง ไม่ใช่แค่ทฤษฎี) และแก้ปัญหานั้นด้วย newline-delimited
protocol ผ่าน `BufReader`/`AsyncBufReadExt::lines()`, ขยายไปสู่การรองรับหลาย client พร้อมกันจริง และออกแบบระบบ
broadcast ข้อความข้าม task ด้วยการผสาน `Arc<Mutex<T>>` จาก Part 39 เข้ากับ channel, แตะ UDP โดยสังเขปเพื่อให้เห็น
ข้อแลกเปลี่ยนระหว่างความน่าเชื่อถือกับ latency, และปิดท้ายด้วย `tokio::time::timeout()` ที่เป็นทางลัดสะดวกของกลไก
`select!` จาก Part 48 สำหรับป้องกัน operation ที่อาจค้างตลอดกาล

ตัวอย่าง TCP chat server หลายผู้ใช้ในหัวข้อ 49.12 คือการรวมทุกแนวคิดของบทนี้เข้าด้วยกันเป็นแอปพลิเคชันที่ทำงานได้จริง
บน localhost — แต่มันยังใช้ `Vec<(u64, Sender)>` ที่ต้อง lock/iterate ด้วยมือทุกครั้งที่ broadcast ซึ่งเป็นวิธีที่
ตรงไปตรงมาที่สุดแต่ไม่ใช่วิธีที่สะดวกหรือมีประสิทธิภาพที่สุด — **Part 50: Async Channels และ Synchronization
(tokio::sync)** จะพาคุณไปรู้จัก primitive ของ Tokio เองอย่างเป็นทางการทั้งชุด: `tokio::sync::mpsc` แบบเต็มรูปแบบ
(ที่บทนี้แค่หยิบมาใช้แบบผิวเผิน), `tokio::sync::oneshot` สำหรับส่งค่าเดียวข้าม task, **`tokio::sync::broadcast`**
ที่ถูกออกแบบมาเพื่อโจทย์ "กระจายข้อความหนึ่งชุดไปยังผู้รับหลายคน" โดยเฉพาะ (จะทำให้เขียน chat server แบบบทนี้ได้
กระชับกว่ามาก), `tokio::sync::watch` สำหรับกระจายค่าสถานะล่าสุด, และ `tokio::sync::Mutex`/`RwLock` แบบ async ที่
กับดักที่ 5 ของบทนี้แนะไว้ล่วงหน้าแล้วว่าเมื่อไหร่ควรใช้แทน `std::sync::Mutex` ธรรมดา

---

**Part ก่อนหน้า:** [Tokio: Runtime และ Tasks](part-048-tokio-runtime.md) | **Part ถัดไป:** [Async Channels และ Synchronization](part-050-tokio-sync.md)
