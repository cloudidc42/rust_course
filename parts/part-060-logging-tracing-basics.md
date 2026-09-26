# Part 60: Logging และ Tracing เบื้องต้น (log, tracing crate)

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 220 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างละเอียดว่าทำไม `println!`/`eprintln!` (ที่ใช้แบบไม่เป็นทางการมาตั้งแต่ Part 1) **ไม่พอ**สำหรับ
  แอปพลิเคชันจริง — ไม่มี severity level ให้กรอง, ไม่มี context ที่มีโครงสร้าง (timestamp, module, request id),
  ปรับความละเอียด (verbosity) ตอน runtime ไม่ได้โดยไม่ compile ใหม่ — และแก้ปัญหาทั้งหมดนี้ด้วย **`log` crate**
  ซึ่งเป็น **facade** (แนวคิดเดียวกับ format-independence ของ serde ใน Part 57 แต่ใช้กับ logging) ที่แยก
  "อินเทอร์เฟซการเรียก log" ออกจาก "ตัว backend ที่ตัดสินใจว่าจะเอา log ไปทำอะไรจริง ๆ"
- เพิ่ม **`env_logger`** เข้าโปรเจกต์ (`cargo add log env_logger`) เรียก `env_logger::init()` แล้วควบคุมพฤติกรรม
  ของ log ทั้งโปรแกรม**โดยไม่ต้อง compile ใหม่เลย** ผ่าน environment variable `RUST_LOG` — พิสูจน์ด้วยการรัน
  โปรแกรมเดียวกันซ้ำหลายครั้งด้วยค่า `RUST_LOG` ต่างกัน (`info`, `debug`, `module_name=trace`) แล้วเห็น output
  จริงที่ต่างกันชัดเจนในแต่ละครั้ง
- เลือก log level ที่เหมาะสม (`error!`/`warn!`/`info!`/`debug!`/`trace!`) ได้อย่างมีเหตุผลในโค้ดจริง ไม่ใช่เดา
  สุ่ม ๆ — รู้ว่าเมื่อไหร่ควรใช้ level ไหนจากตัวอย่างระบบคลังสินค้าจำลองที่มีทั้ง error, warning ที่ recover ได้,
  event สำคัญ, และ diagnostic ละเอียดสำหรับ developer
- อธิบายปัญหาเฉพาะของ **async code** ที่ `log` แบบเดิมแก้ไม่ได้ — เมื่อหลาย task/request แข่งกันทำงานสลับกันบน
  OS thread เดียว (cooperative multitasking จาก Part 48) log line ที่พิมพ์ออกมาปนกันจะ**ไม่มีทางบอกได้ว่าเป็น
  ของ request ไหน** — และแก้ด้วย **`tracing` crate** ผ่านแนวคิด **span** ที่ `#[tracing::instrument]`
  (attribute macro แบบเดียวกับที่ Part 44 สอน) สร้างขึ้นให้อัตโนมัติ ครอบคลุมทั้งฟังก์ชันแม้ข้าม `.await`
- ใช้ **`tracing_subscriber`** (คู่เทียบของ `env_logger` ในโลก `tracing`) พร้อม `EnvFilter` ควบคุมด้วย `RUST_LOG`
  แบบเดียวกัน และเข้าใจโมเดล subscriber/layer ที่ปรับเปลี่ยนได้ (ต่อ output เป็น JSON สำหรับระบบ log aggregation
  ได้ ซึ่งเป็นพื้นฐานที่ Part 98-99 จะขยายต่อ) พร้อมเขียน **structured field** (`info!(user_id = 42, ...)`)
  แทนข้อความล้วน ๆ เพื่อให้ระบบปลายทางกรอง/ค้นหาข้อมูลได้ตรง ๆ
- ตัดสินใจได้ว่าโปรเจกต์ใหม่ควรเลือก `log` หรือ `tracing` โดยมีเหตุผลรองรับ พร้อมประยุกต์ทุกอย่างเขียนตัวอย่าง
  **request handler จำลองแบบ async ที่มีหลาย request ทำงานพร้อมกันจริง** (`tokio::spawn`) แล้ว instrument ด้วย
  `tracing` เต็มรูปแบบ พิสูจน์ด้วย output จริงว่า log จาก request ต่าง ๆ ที่สลับกันพิมพ์ออกมายังคง**ระบุที่มาได้
  ถูกต้อง 100%** ผ่าน span context

## ความรู้ที่ต้องมีมาก่อน

- **Part 46 (Async/Await เบื้องต้น)**, **Part 47 (Futures และ Executors)**, **Part 48 (Tokio: Runtime และ
  Tasks)**, **Part 49 (Tokio: I/O และ Networking)**, **Part 50 (Async Channels และ Synchronization)**: บทนี้
  พึ่งพา Part 46-50 อย่างหนักที่สุด — หัวใจของปัญหาที่ `tracing` แก้คือปัญหาที่มีอยู่**เฉพาะ**ในโลก async: การที่
  หลาย task ถูก poll สลับกันบน OS thread เดียวกัน (cooperative multitasking ที่ Part 48 อธิบายไว้ตอนสอน
  `tokio::spawn`) ทำให้ log ธรรมดาที่ไม่มีแนวคิดเรื่อง "task ปัจจุบันคือใคร" สับสนได้ง่ายมาก ถ้า Part 48 (โดย
  เฉพาะเรื่อง task, `.await`, และการที่ task หลายตัวแบ่งกันใช้ worker thread) ยังไม่แน่น กลับไปทวนก่อน เพราะบทนี้
  จะอ้างอิงกลับไปตลอดเวลาโดยไม่อธิบายกลไก async ซ้ำจากศูนย์
- **Part 12 (Result และ Error Handling เบื้องต้น)**: การตัดสินใจว่าจะ log ด้วย level ไหน (โดยเฉพาะ `error!`
  เทียบกับ `warn!`) ผูกกับความเข้าใจเรื่อง "error ที่คาดไว้แล้ว จัดการได้" เทียบกับ "error ที่ไม่ควรเกิดขึ้น"
  ที่ Part 12 (และ Part 30-31 เรื่อง `thiserror`/`anyhow`) วางกรอบไว้ — บทนี้ไม่สอนการออกแบบ error type ใหม่
  แต่จะใช้ `Result<T, E>` ที่คุณคุ้นเคยแล้วเป็นจุดตัดสินใจว่าจะ log ระดับไหน
- **Part 44 (Procedural Macros เบื้องต้น)**: `#[tracing::instrument]` คือ **attribute macro**
  (`#[proc_macro_attribute]`) ตัวเดียวกับที่ Part 44 สอนกลไกให้เขียนเอง และเป็นแนวคิดเดียวกับ `#[tokio::main]`
  ที่ Part 48 อธิบายไว้ (แปลง item หนึ่งเป็น item ใหม่ที่มีโค้ดเพิ่มเข้ามาห่อไว้) — บทนี้จะไม่อธิบายกลไก macro
  ซ้ำ แต่จะใช้ความเข้าใจนั้นอธิบายว่า `#[instrument]` "ห่อ" ฟังก์ชันของคุณด้วย span ยังไงในระดับแนวคิด
- **Part 59 (CLI Applications ด้วย clap)**: แอป CLI ที่ Part 59 สอนสร้างมักมี flag `-v`/`-vv`/`-vvv` ให้ผู้ใช้
  ปรับความละเอียดของ log ตอนรันจริง — บทนี้จะโยงกลับไปที่ pattern นี้ตรง ๆ ในหัวข้อเรื่องการปรับ verbosity ด้วย
  โค้ด (ไม่ใช่แค่ environment variable) และการที่ logging เป็นสิ่งที่ CLI tool ทุกตัวควรมีเพื่อ debug ตอนใช้งาน
  จริงในเครื่องคนอื่น
- **Part 57 (Serialization: Serde เบื้องต้น)**: บทนั้นสอนว่า `serde` เองไม่รู้จัก JSON/TOML/bincode เลย —
  มันนิยามแค่ `Serialize`/`Deserialize` trait (อินเทอร์เฟซ) แล้วปล่อยให้ crate อย่าง `serde_json`/`toml` เป็น
  ตัวจัดการ format จริง บทนี้จะดึง**แนวคิดเดียวกันเป๊ะ**มาอธิบาย `log` crate: `log` นิยามแค่ macro/อินเทอร์เฟซ
  การเรียก แต่ไม่รู้จักเลยว่า log จะไปโผล่ที่ terminal, ไฟล์, หรือระบบอื่นใด — ถ้า Part 57 ยังไม่ได้อ่าน อ่านสรุป
  สั้น ๆ นี้ก็พอเข้าใจ analogy ได้ (ไม่ต้องมีรายละเอียดของ serde มาก่อนเพื่อเข้าใจ facade pattern ของ log)

## เนื้อหา

### 60.1 ทำไม `println!`/`eprintln!` ไม่พอสำหรับแอปพลิเคชันจริง

ตั้งแต่ Part 1 คุณใช้ `println!`/`eprintln!` มาตลอดเป็นเครื่องมือหลักในการดู output — มันเรียบง่าย, compile
เร็ว, ไม่ต้องเพิ่ม dependency อะไรเลย และเหมาะมากสำหรับตัวอย่างสั้น ๆ ในหลักสูตรนี้ แต่พอถึงจุดที่คุณเขียน
แอปพลิเคชันจริงที่มีคนอื่นใช้งาน (หรือรันอยู่บน server ที่คุณไม่ได้จ้องหน้าจอตลอดเวลา) `println!` เริ่มแสดง
ข้อจำกัดที่ร้ายแรงขึ้นเรื่อย ๆ สี่ข้อ:

**1. ไม่มี severity level ให้กรอง** — `println!("กำลังเชื่อมต่อ database...")` กับ
`println!("เชื่อมต่อ database ล้มเหลว!")` มีความสำคัญต่างกันโดยสิ้นเชิงในสายตาคนอ่าน log แต่ในทางเทคนิคทั้งคู่
เป็นแค่ "การพิมพ์ข้อความ" เหมือนกันเป๊ะ ไม่มีกลไกอะไรให้คุณพูดว่า "ตอน deploy จริง ฉันอยากเห็นแค่ข้อความที่
เป็นปัญหา ไม่อยากเห็นข้อความบอกสถานะทั่วไปทุกบรรทัด" — คุณต้อง**ลบ**หรือ**comment**บรรทัด `println!` ที่ไม่
ต้องการออกเอง ซึ่งพอกลับมาต้องการดูอีกทีก็ต้องแก้โค้ดกลับมาใหม่

**2. ไม่มี context ที่มีโครงสร้าง** — `println!("เชื่อมต่อ database ล้มเหลว!")` ไม่บอกว่าเกิดขึ้น**เมื่อไหร่**
(timestamp), เกิดที่**ไฟล์/บรรทัด**ไหนของโค้ด, หรือถ้าเป็นระบบที่รับหลาย request พร้อมกัน — เกิดกับ **request
ไหน**? ข้อมูลเหล่านี้ต้องเขียนเพิ่มเข้าไปในทุกบรรทัด `println!` เองด้วยมือ (`println!("[{}] เชื่อมต่อ database
ล้มเหลว!", chrono::Local::now())`) ซึ่งรกโค้ดมากถ้าทำแบบนี้ทุกที่ที่มีการพิมพ์ log

**3. ปรับ/redirect ไม่ได้โดยไม่ compile ใหม่** — `println!` เขียนไป stdout เสมอตายตัว ถ้าต้องการให้ log ไปที่
ไฟล์แทน (ทั่วไปในระบบ production ที่ log ต้องถูกเก็บไว้อ่านย้อนหลัง) ต้องแก้โค้ดทุกจุดที่เรียก `println!` เป็น
เขียนไฟล์เอง — ไม่มีการ "สลับ backend" ได้จากภายนอกโดยไม่แก้/compile โค้ดใหม่

**4. ปรับความละเอียด (verbosity) ตอน runtime ไม่ได้** — สมมติโปรแกรมมี debugging message ละเอียดมากที่มีประโยชน์
ตอนไล่ bug แต่รกเกินไปสำหรับการใช้งานปกติ ด้วย `println!` ทางเลือกมีแค่: (ก) ลบ/comment มันออกไปเลย แล้วถ้า
ต้องการ debug อีกทีต้องแก้โค้ดใส่กลับมาใหม่ compile ใหม่ หรือ (ข) ปล่อยให้มันพิมพ์เสมอ รกไปตลอด ไม่มีทางกลาง ๆ
ที่บอกว่า "ปกติไม่ต้องโชว์ แต่ถ้าฉันตั้ง environment variable บางตัว ให้โชว์" — นี่คือสิ่งที่ทำให้ debug ปัญหาที่
เกิดเฉพาะบนเครื่อง production ยากมาก เพราะคุณไม่สามารถ "เปิด verbose mode" ได้โดยไม่ deploy โค้ดใหม่

ทั้งสี่ข้อนี้คือเหตุผลที่ระบบนิเวศ Rust (และภาษาอื่นแทบทุกภาษา) มี **logging framework** แยกออกมาต่างหาก
ไม่ใช่แค่ใช้ print statement — บทนี้จะพาไปรู้จักสองเครื่องมือหลักของ Rust ที่แก้ปัญหาทั้งสี่ข้อนี้ได้ครบ: `log`
และ `tracing`

### 60.2 `log` Crate: Facade Pattern สำหรับ Logging

#### ทำความเข้าใจ Facade Pattern ก่อนแตะโค้ด

จำได้จาก **Part 57** ไหมว่า `serde` เองไม่รู้จัก JSON เลย — มันนิยามแค่ trait `Serialize`/`Deserialize` (บอก
ว่า "ค่านี้แปลงเป็น/จากรูปแบบทั่วไปได้อย่างไร") แล้วปล่อยให้ crate อย่าง `serde_json` เป็นคนตัดสินใจว่ารูปแบบ
สุดท้ายคือ JSON จริง ๆ — ผลคือ struct หนึ่งตัวที่ `#[derive(Serialize)]` สามารถถูกแปลงเป็น JSON, TOML, YAML,
หรือ binary format ได้หมด โดยตัว struct เองไม่ต้องรู้เรื่อง format พวกนี้เลยแม้แต่นิดเดียว

**`log` crate ใช้แนวคิดเดียวกันเป๊ะ แต่กับ logging แทน serialization**:

- `log` นิยาม**macro/อินเทอร์เฟซการเรียก** — `error!()`, `warn!()`, `info!()`, `debug!()`, `trace!()` — และ
  trait `Log` ที่อธิบายว่า "backend ตัวหนึ่งต้อง implement อะไรถึงจะรับ log record ได้"
- `log` **ไม่รู้เลย**ว่า log ที่เรียกไปจะไปโผล่ที่ไหน — terminal? ไฟล์? ระบบ log แบบ centralized ผ่าน network?
  ไม่ใช่เรื่องของ `log` เลย
- **Library crate** (เช่น HTTP client, database driver ที่คนอื่นเขียนแล้วคุณเอามาใช้) สามารถเรียก
  `log::warn!("connection retry #{attempt}")` ได้อย่างอิสระ**โดยไม่ต้องสนใจ**ว่าแอปพลิเคชันที่เอา library
  ของมันไปใช้จะ handle log พวกนี้ยังไง — library แค่ "ประกาศ" ว่ามีเหตุการณ์นี้เกิดขึ้น
- **Application** (โค้ดของคุณที่เป็นจุดเริ่มต้นโปรแกรม `fn main()`) เป็นผู้**เลือก backend ที่เป็นรูปธรรม**
  (concrete logger implementation) แค่ครั้งเดียวตอนเริ่มโปรแกรม — backend ตัวนี้ (เช่น `env_logger` ที่จะเรียน
  ในหัวข้อถัดไป) implement trait `Log` จริง ๆ แล้วตัดสินใจเองว่าจะพิมพ์ log ไปไหน กรองด้วย level ไหน จัดรูปแบบ
  ยังไง

นี่คือเหตุผลเชิงลึกที่สำคัญมาก: **ระบบนิเวศทั้งหมดของ Rust (crate หลายพันตัว) ใช้ `log` เป็นภาษากลางร่วมกัน**
ได้ โดยที่ไม่มี crate ไหนต้อง "ผูก" ตัวเองเข้ากับ logging backend ตัวใดตัวหนึ่งโดยเฉพาะ — เหมือนกับที่ `serde`
ทำให้ทุก crate ที่ต้องการ serialize ข้อมูลไม่ต้องผูกตัวเองกับ JSON เพียงอย่างเดียว

#### เพิ่ม `log` เข้าโปรเจกต์

```bash
cargo add log
```

ได้ dependency เพิ่มเข้ามาใน `Cargo.toml`:

```toml
[dependencies]
log = "0.4.34"
```

macro ทั้งห้าตัวของ `log` มี syntax คล้าย `println!`/`format!` มาก (รับ format string + argument):

```rust
log::error!("เชื่อมต่อ database ล้มเหลว: {reason}", reason = "timeout");
log::warn!("คำขอใช้เวลานานผิดปกติ: {elapsed_ms}ms", elapsed_ms = 850);
log::info!("เซิร์ฟเวอร์เริ่มทำงานที่พอร์ต {port}", port = 8080);
log::debug!("ค่า config ที่โหลดมา: max_connections={max}", max = 100);
log::trace!("เข้าสู่ฟังก์ชัน handle_request()");
```

#### สิ่งสำคัญที่สุดที่ต้องเข้าใจก่อนไปต่อ: ไม่มี Logger Backend = ไม่มีอะไรเกิดขึ้นเลย

นี่คือจุดที่ผู้เริ่มต้นสับสนบ่อยที่สุด — เพราะ `log` เป็นแค่ facade **ไม่มีการ implement จริงติดมาด้วย** ถ้าคุณ
เรียก macro พวกนี้โดยไม่ได้ตั้งค่า backend ไว้เลย **มันจะทำงาน "เงียบ" สนิท ไม่มี error, ไม่มี warning, ไม่มี
อะไรปรากฏขึ้นมาเลยแม้แต่นิดเดียว**:

```rust
fn main() {
    log::error!("นี่คือ error");
    log::warn!("นี่คือ warn");
    log::info!("นี่คือ info");
    println!("จบโปรแกรมแล้ว — สังเกตว่าไม่มีบรรทัด log ใดๆ ปรากฏเลยด้านบน");
}
```

รันจริงได้ output แค่บรรทัดเดียว:

```
จบโปรแกรมแล้ว — สังเกตว่าไม่มีบรรทัด log ใดๆ ปรากฏเลยด้านบน
```

สังเกตให้ดี — โปรแกรมนี้เรียก `log::error!` ซึ่งควรจะเป็น level ที่ "สำคัญที่สุด" แต่ก็**ไม่มีอะไรพิมพ์ออกมาเลย**
เหตุผลคือ: ภายใน `log` crate มี logger กลางตัวหนึ่ง (global logger, implement ผ่าน `static` + `OnceCell`-like
กลไกภายใน) ที่ทุก macro (`error!`, `warn!`, ...) จะส่ง record ไปให้ — **ถ้าไม่มีใครเรียก `log::set_logger(...)`
มาก่อน (ซึ่งปกติทำผ่านฟังก์ชัน `init()` ของ backend ที่เลือกใช้) logger กลางตัวนี้จะเป็นค่าเริ่มต้นที่ไม่ทำอะไร
เลย (a no-op logger)** — ทุก record ที่ส่งมาถูกทิ้งเงียบ ๆ ทันที นี่ไม่ใช่ bug แต่เป็น**การตัดสินใจทางออกแบบที่
ตั้งใจ**: `log` ต้องการให้แน่ใจว่า library crate ที่เรียก log macro ได้โดยไม่ panic หรือ error แม้แอปพลิเคชันที่
เอามันไปใช้จะไม่ได้สนใจ setup logging เลยก็ตาม — ผลที่ตามมาคือ **การลืม init logger backend จึงกลายเป็นกับดักที่
พบบ่อยที่สุดข้อหนึ่งของทั้งบทนี้** (จะกลับมาย้ำในหัวข้อกับดัก) เพราะไม่มี error message อะไรมาเตือนเลย

### 60.3 `env_logger`: Backend ที่ควบคุมด้วย `RUST_LOG`

ตอนนี้มาเติมส่วนที่หายไป — **backend ที่เป็นรูปธรรม** ที่รับ log record จาก `log` แล้วทำอะไรสักอย่างกับมันจริง ๆ
`env_logger` คือ backend ที่**เรียบง่ายและได้รับความนิยมสูงสุด**สำหรับงานส่วนใหญ่ — พิมพ์ log ไปที่ stderr พร้อม
timestamp, level, และ module path โดยควบคุมด้วย environment variable ชื่อ **`RUST_LOG`**

```bash
cargo add env_logger
```

```toml
[dependencies]
env_logger = "0.11.11"
```

#### ตัวอย่างจริง: ระบบจัดการคลังสินค้าจำลอง

มาเขียนแอปพลิเคชันเล็ก ๆ ที่ใช้ log ครบทั้ง 5 level ในสถานการณ์ที่สมเหตุสมผลจริง — ระบบจัดการคลังสินค้า (คำแนะนำ
จาก Style Guide ของหลักสูตรนี้ให้ใช้ตัวอย่างโลกจริงแทน `foo`/`bar`):

```rust
use std::collections::HashMap;

struct Inventory {
    stock: HashMap<String, u32>,
}

impl Inventory {
    fn new() -> Self {
        log::debug!("สร้าง Inventory ใหม่ (เริ่มต้นด้วย stock เปล่า)");
        Self { stock: HashMap::new() }
    }

    fn restock(&mut self, sku: &str, qty: u32) {
        log::trace!("restock() ถูกเรียกด้วย sku={sku} qty={qty}");
        *self.stock.entry(sku.to_string()).or_insert(0) += qty;
        log::info!("เติมสต็อก {sku} จำนวน {qty} หน่วย (รวมปัจจุบัน {})", self.stock[sku]);
    }

    fn reserve(&mut self, sku: &str, qty: u32) -> Result<(), String> {
        log::trace!("reserve() ถูกเรียกด้วย sku={sku} qty={qty}");
        let current = *self.stock.get(sku).unwrap_or(&0);
        if qty > current {
            log::warn!(
                "พยายามจอง {sku} จำนวน {qty} แต่มีสต็อกแค่ {current} — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ"
            );
            return Err(format!("สต็อกไม่พอสำหรับ {sku}"));
        }
        self.stock.insert(sku.to_string(), current - qty);
        log::info!("จองสินค้า {sku} จำนวน {qty} สำเร็จ (เหลือ {})", current - qty);
        Ok(())
    }

    fn reserve_or_panic_path(&mut self, sku: &str, qty: u32) {
        if let Err(e) = self.reserve(sku, qty) {
            // สถานการณ์นี้คือ "ความผิดพลาดที่ไม่ควรเกิดขึ้น" ในเส้นทางนี้ของโปรแกรม (caller เช็คมาก่อนแล้วว่าน่าจะพอ)
            log::error!("reserve_or_panic_path ล้มเหลวโดยไม่คาดคิด: {e}");
        }
    }
}

fn main() {
    env_logger::init();

    log::info!("เริ่มต้นระบบจัดการคลังสินค้า");

    let mut inv = Inventory::new();
    inv.restock("SKU-100", 50);
    inv.restock("SKU-200", 5);

    let _ = inv.reserve("SKU-100", 10);
    let _ = inv.reserve("SKU-200", 20); // ควรเห็น warn เพราะสต็อกไม่พอ
    inv.reserve_or_panic_path("SKU-999", 1); // sku ที่ไม่มีอยู่เลย -> reserve คืน Err -> error!

    log::info!("จบการทำงานของระบบจัดการคลังสินค้า");
}
```

**`env_logger::init()`** คือบรรทัดเดียวที่เพิ่มเข้ามาซึ่งเปลี่ยนทุกอย่าง — มันเรียก `log::set_logger(...)` ภายใน
ให้อัตโนมัติ ติดตั้ง `env_logger`'s logger เป็น backend กลางของ `log` ทั้งโปรแกรม ต้องเรียก**ครั้งเดียวตอนต้น
`main()`** เท่านั้น (เรียกซ้ำสองครั้งจะ panic เพราะ `log` อนุญาตให้ set global logger ได้แค่ครั้งเดียว)

#### รันโดยไม่ตั้ง `RUST_LOG` เลย: ค่าเริ่มต้นคือ Error เท่านั้น

```bash
cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง:

```
[2026-09-26T23:00:28Z ERROR log_demo] reserve_or_panic_path ล้มเหลวโดยไม่คาดคิด: สต็อกไม่พอสำหรับ SKU-999
```

เห็นแค่บรรทัดเดียว — **ค่าเริ่มต้นของ `env_logger` (ถ้าไม่ตั้ง `RUST_LOG` เลย) คือแสดงแค่ level `Error` เท่านั้น**
ทุก `info!`/`warn!`/`debug!`/`trace!` ที่เรียกไปถูกกรองออกหมด สังเกตรูปแบบของแต่ละบรรทัด:
`[เวลา ISO-8601 แบบ UTC] LEVEL module_path] ข้อความ` — timestamp กับ module path ที่ `println!` ไม่มีให้มา
ปรากฏขึ้นมาให้อัตโนมัติโดยไม่ต้องเขียนเพิ่มเองแม้แต่บรรทัดเดียว นี่คือคำตอบของปัญหาข้อ 2 ในหัวข้อ 60.1

#### `RUST_LOG=info`: เห็น info และสูงกว่า

```bash
RUST_LOG=info cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง:

```
[2026-09-26T23:00:35Z INFO  log_demo] เริ่มต้นระบบจัดการคลังสินค้า
[2026-09-26T23:00:35Z INFO  log_demo] เติมสต็อก SKU-100 จำนวน 50 หน่วย (รวมปัจจุบัน 50)
[2026-09-26T23:00:35Z INFO  log_demo] เติมสต็อก SKU-200 จำนวน 5 หน่วย (รวมปัจจุบัน 5)
[2026-09-26T23:00:35Z INFO  log_demo] จองสินค้า SKU-100 จำนวน 10 สำเร็จ (เหลือ 40)
[2026-09-26T23:00:35Z WARN  log_demo] พยายามจอง SKU-200 จำนวน 20 แต่มีสต็อกแค่ 5 — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ
[2026-09-26T23:00:35Z WARN  log_demo] พยายามจอง SKU-999 จำนวน 1 แต่มีสต็อกแค่ 0 — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ
[2026-09-26T23:00:35Z ERROR log_demo] reserve_or_panic_path ล้มเหลวโดยไม่คาดคิด: สต็อกไม่พอสำหรับ SKU-999
[2026-09-26T23:00:35Z INFO  log_demo] จบการทำงานของระบบจัดการคลังสินค้า
```

สังเกตว่า `debug!`/`trace!` (จาก `Inventory::new()` และ `restock`/`reserve`) ยังไม่โชว์ — `RUST_LOG=info` แปลว่า
"แสดง level `info` **และสูงกว่า**" ตามลำดับความสำคัญ **`error > warn > info > debug > trace`** (ยิ่งอยู่ทาง
ซ้ายยิ่งสำคัญมาก/verbose น้อย)

#### `RUST_LOG=debug`: เห็น debug เพิ่มขึ้นมา

```bash
RUST_LOG=debug cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง (สังเกตบรรทัด `DEBUG` ที่โผล่มาเพิ่มจากครั้งก่อน):

```
[2026-09-26T23:00:35Z INFO  log_demo] เริ่มต้นระบบจัดการคลังสินค้า
[2026-09-26T23:00:35Z DEBUG log_demo] สร้าง Inventory ใหม่ (เริ่มต้นด้วย stock เปล่า)
[2026-09-26T23:00:35Z INFO  log_demo] เติมสต็อก SKU-100 จำนวน 50 หน่วย (รวมปัจจุบัน 50)
[2026-09-26T23:00:35Z INFO  log_demo] เติมสต็อก SKU-200 จำนวน 5 หน่วย (รวมปัจจุบัน 5)
[2026-09-26T23:00:35Z INFO  log_demo] จองสินค้า SKU-100 จำนวน 10 สำเร็จ (เหลือ 40)
[2026-09-26T23:00:35Z WARN  log_demo] พยายามจอง SKU-200 จำนวน 20 แต่มีสต็อกแค่ 5 — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ
[2026-09-26T23:00:35Z WARN  log_demo] พยายามจอง SKU-999 จำนวน 1 แต่มีสต็อกแค่ 0 — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ
[2026-09-26T23:00:35Z ERROR log_demo] reserve_or_panic_path ล้มเหลวโดยไม่คาดคิด: สต็อกไม่พอสำหรับ SKU-999
[2026-09-26T23:00:35Z INFO  log_demo] จบการทำงานของระบบจัดการคลังสินค้า
```

ยัง**ไม่เห็น** `TRACE` (จาก `restock()`/`reserve()`) เพราะ `trace` อยู่ต่ำกว่า `debug` ในลำดับความสำคัญ

#### Module-Path Filtering: กรองแยกตาม Crate/Module ด้วย `RUST_LOG=module=level`

`RUST_LOG` ไม่ได้กรองได้แค่ "ทั้งโปรแกรม level เดียว" — syntax ที่ทรงพลังกว่าคือ **`module_path=level`** ซึ่งกรอง
แยกเฉพาะ module/crate นั้น ๆ (คั่นหลายเงื่อนไขด้วย comma: `RUST_LOG=my_module=debug,other_module=warn`) จุดที่
ต้องระวังให้แม่นคือ **`module_path` ในที่นี้คือชื่อ crate/binary ตามที่ compiler เห็น ไม่ใช่ชื่อ package ใน
`Cargo.toml`** — ตัวอย่างนี้ (ไฟล์อยู่ที่ `src/bin/log_demo.rs`) crate name ที่แท้จริงคือ `log_demo` (ชื่อไฟล์
`.rs` ไม่รวม extension) แม้ว่า package ใน `Cargo.toml` จะชื่อ `logging_demo` ก็ตาม — ลองใช้ชื่อ**ผิด**ก่อน:

```bash
RUST_LOG=logging_demo=trace cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง: **ไม่มี output อะไรเลย** (ว่างสนิท แม้แต่ `ERROR` ก็ไม่โผล่) เพราะ `logging_demo=trace` เป็น directive
ที่ตั้ง level ให้ module path `logging_demo` เท่านั้น ซึ่งไม่มี module ไหนของโปรแกรมนี้ตรงกับชื่อนี้เลย (crate
จริงชื่อ `log_demo`) — ผลคือ**ไม่มี directive ไหนตรงกับ module ใดของโปรแกรมเลย** จึงกลับไปใช้ค่า default ของ
`env_logger` ซึ่งก็คือไม่แสดงอะไรเลยนอกจากถ้ามี directive ทั่วไปแบบ level เดี่ยว ๆ ปนมาด้วย (ในที่นี้ไม่มี) —
ลองใช้ชื่อ**ที่ถูกต้อง**:

```bash
RUST_LOG=log_demo=trace cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง (คราวนี้เห็นครบทุก level รวม `TRACE` จาก `restock()`/`reserve()`):

```
[2026-09-26T23:00:44Z INFO  log_demo] เริ่มต้นระบบจัดการคลังสินค้า
[2026-09-26T23:00:44Z DEBUG log_demo] สร้าง Inventory ใหม่ (เริ่มต้นด้วย stock เปล่า)
[2026-09-26T23:00:44Z TRACE log_demo] restock() ถูกเรียกด้วย sku=SKU-100 qty=50
[2026-09-26T23:00:44Z INFO  log_demo] เติมสต็อก SKU-100 จำนวน 50 หน่วย (รวมปัจจุบัน 50)
[2026-09-26T23:00:44Z TRACE log_demo] restock() ถูกเรียกด้วย sku=SKU-200 qty=5
[2026-09-26T23:00:44Z INFO  log_demo] เติมสต็อก SKU-200 จำนวน 5 หน่วย (รวมปัจจุบัน 5)
[2026-09-26T23:00:44Z TRACE log_demo] reserve() ถูกเรียกด้วย sku=SKU-100 qty=10
[2026-09-26T23:00:44Z INFO  log_demo] จองสินค้า SKU-100 จำนวน 10 สำเร็จ (เหลือ 40)
[2026-09-26T23:00:44Z TRACE log_demo] reserve() ถูกเรียกด้วย sku=SKU-200 qty=20
[2026-09-26T23:00:44Z WARN  log_demo] พยายามจอง SKU-200 จำนวน 20 แต่มีสต็อกแค่ 5 — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ
[2026-09-26T23:00:44Z TRACE log_demo] reserve() ถูกเรียกด้วย sku=SKU-999 qty=1
[2026-09-26T23:00:44Z WARN  log_demo] พยายามจอง SKU-999 จำนวน 1 แต่มีสต็อกแค่ 0 — ปฏิเสธคำขอนี้แต่ระบบยังทำงานต่อได้ปกติ
[2026-09-26T23:00:44Z ERROR log_demo] reserve_or_panic_path ล้มเหลวโดยไม่คาดคิด: สต็อกไม่พอสำหรับ SKU-999
[2026-09-26T23:00:44Z INFO  log_demo] จบการทำงานของระบบจัดการคลังสินค้า
```

รูปแบบ **`RUST_LOG=my_module=debug,other_module=warn`** (คั่นด้วย comma หลาย directive) มีประโยชน์มากในระบบ
production จริงที่มีหลาย crate ทำงานร่วมกัน — เช่นอยากเห็น log ละเอียดของโค้ดตัวเองแต่ไม่อยากเห็น log ของ
dependency (เช่น HTTP client library) ที่ verbose มากจนรก:

```bash
RUST_LOG=my_app=debug,hyper=warn,sqlx=warn cargo run
```

directive นี้บอกว่า: "module `my_app` แสดงตั้งแต่ `debug` ขึ้นไป ส่วน `hyper` และ `sqlx` (dependency ที่ verbose)
แสดงแค่ `warn` ขึ้นไปพอ" — ทั้งหมดนี้ **ปรับได้จาก command line ล้วน ๆ โดยไม่ต้องแก้โค้ดหรือ compile ใหม่แม้แต่
บรรทัดเดียว** ซึ่งคือคำตอบเต็มรูปแบบของปัญหาข้อ 3 และข้อ 4 ในหัวข้อ 60.1

#### ปรับ Verbosity ด้วยโค้ดแทน Environment Variable: เชื่อมกับ CLI App จาก Part 59

`RUST_LOG` สะดวกมากตอน deploy/debug จาก terminal แต่ CLI application ที่ Part 59 สอนสร้างด้วย `clap` มักมี
รูปแบบที่ผู้ใช้คุ้นเคยกว่า: flag `-v`/`-vv`/`-vvv` ที่นับจำนวนครั้งเพื่อเพิ่มความละเอียด — `env_logger` รองรับ
การตั้งค่าแบบนี้ผ่าน `Builder` API แทนการพึ่ง `RUST_LOG` เพียงอย่างเดียว:

```rust
use log::LevelFilter;

fn verbosity_to_level(verbosity_count: u8) -> LevelFilter {
    match verbosity_count {
        0 => LevelFilter::Warn,  // ค่าเริ่มต้น: ไม่ส่ง -v เลย เห็นแค่ warn/error
        1 => LevelFilter::Info,  // -v
        2 => LevelFilter::Debug, // -vv
        _ => LevelFilter::Trace, // -vvv หรือมากกว่า
    }
}

fn main() {
    // จำลองว่าผู้ใช้ส่ง `-vv` เข้ามาทาง CLI (ในโปรแกรมจริงค่านี้มาจาก clap ที่ Part 59 สอน)
    let verbosity_count: u8 = 2;

    env_logger::Builder::new()
        .filter_level(verbosity_to_level(verbosity_count))
        .init();

    log::error!("error level");
    log::warn!("warn level");
    log::info!("info level");
    log::debug!("debug level");
    log::trace!("trace level");
}
```

รันจริงได้ (`-vv` = `Debug`, เห็นทุก level ตั้งแต่ error ถึง debug แต่ไม่เห็น trace):

```
[2026-09-26T23:03:18Z ERROR builder_verbosity_demo] error level
[2026-09-26T23:03:18Z WARN  builder_verbosity_demo] warn level
[2026-09-26T23:03:18Z INFO  builder_verbosity_demo] info level
[2026-09-26T23:03:18Z DEBUG builder_verbosity_demo] debug level
```

`env_logger::Builder::new().filter_level(...).init()` คือรูปแบบเดียวกับ `env_logger::init()` แค่เปิดโอกาสให้
คุณกำหนด level เริ่มต้นด้วยโค้ดแทนที่จะพึ่ง `RUST_LOG` ล้วน ๆ (ทั้งสองผสมกันได้จริง เช่นให้ `RUST_LOG` override
ค่าที่มาจาก `-v` ถ้ามีการตั้งไว้ — `Builder` มีเมธอด `.parse_env("RUST_LOG")` ให้ผสมสองแหล่งนี้เข้าด้วยกันได้ใน
โปรเจกต์จริง) — นี่คือ pattern ที่ CLI tool คุณภาพดีจริง ๆ (เช่น `ripgrep`, `cargo` เอง) ใช้กันทั่วไป

### 60.4 Log Level ในเชิงลึก: เมื่อไหร่ควรใช้ Level ไหน

ตัวอย่างระบบคลังสินค้าในหัวข้อก่อนใช้ log ครบทั้ง 5 level แล้ว — มาแยกวิเคราะห์ทีละ level ว่า**ทำไม**ถึงเลือก
ใช้ level นั้นในจุดนั้น เพราะการเลือก level ผิดคือกับดักที่พบบ่อยมากในโค้ด production จริง (จะกลับมาย้ำอีกครั้ง
ในหัวข้อกับดัก):

| Level | ใช้เมื่อไหร่ | ตัวอย่างจากโค้ดคลังสินค้า | ความถี่ที่คาดหวังใน production |
|---|---|---|---|
| **`error!`** | เกิดสิ่งที่**ไม่ควรเกิดขึ้น**และ**ไม่มีการจัดการที่สมบูรณ์** — ระบบอยู่ในสถานะที่ไม่ตรงกับที่ออกแบบไว้ | `reserve_or_panic_path` เรียก `reserve()` กับ SKU ที่ไม่มีสต็อกเลย ทั้งที่ caller ควรเช็คมาก่อนแล้ว | ควรเกิด**น้อยมาก** ถ้าเกิดถี่แปลว่ามี bug หรือ assumption ที่ผิดในระบบ |
| **`warn!`** | เกิดสิ่งที่**น่าสงสัย**แต่ระบบ**recover ได้เอง**และทำงานต่อได้ตามปกติ | ลูกค้าพยายามจองสินค้าเกินสต็อกที่มี — ระบบปฏิเสธคำขอนั้นแล้วทำงานต่อ ไม่ crash | เกิดได้บ่อยกว่า error แต่ยังควรเป็น**ข้อยกเว้น** ไม่ใช่ทางเดินหลัก |
| **`info!`** | เหตุการณ์ระดับสูงที่**น่าสนใจต่อการเข้าใจภาพรวมการทำงานของระบบ** ไม่ใช่รายละเอียดปลีกย่อย | "เริ่มต้นระบบ", "เติมสต็อก X หน่วย", "จองสินค้าสำเร็จ" | ปริมาณพอดี ๆ ที่ operations team อ่านแล้วเข้าใจว่าระบบทำอะไรอยู่ |
| **`debug!`** | ข้อมูล diagnostic ละเอียดสำหรับ**developer**ที่กำลัง debug ปัญหาเฉพาะเจาะจง ไม่ใช่สำหรับดูตอนใช้งานปกติ | "สร้าง Inventory ใหม่" — developer อยากรู้ตอน trace lifecycle ของ object แต่ operations ไม่สนใจ | เปิดเฉพาะตอน debug เท่านั้น ปกติไม่เปิดใน production |
| **`trace!`** | รายละเอียดระดับ**ทุกขั้นตอน**ของการทำงาน (เช่น เข้า/ออกฟังก์ชัน, ค่า parameter ทุกตัว) verbose ที่สุด | "restock() ถูกเรียกด้วย sku=... qty=..." — เห็น parameter ของทุก call | เปิดแทบไม่เคย นอกจากไล่ bug ที่ยากมากจริง ๆ ตัวหนึ่ง |

**หลักการตัดสินใจที่ใช้ได้จริงในทางปฏิบัติ**: ถามตัวเองว่า **"ใครคือผู้อ่าน log บรรทัดนี้ และเขาต้องทำอะไรเมื่อ
เห็นมัน?"**
- ถ้าคำตอบคือ "ทีม on-call ต้องตื่นมาแก้ตอนตี 3" → `error!`
- ถ้าคำตอบคือ "ทีม operations ควรรู้ว่ามีอะไรแปลก ๆ เกิดขึ้นแต่ไม่ต้องรีบ" → `warn!`
- ถ้าคำตอบคือ "อยากรู้ภาพรวมว่าระบบทำงานถูกต้องปกติดี" → `info!`
- ถ้าคำตอบคือ "developer คนที่เขียนโค้ดนี้อยากรู้รายละเอียดตอน bug เกิด" → `debug!` หรือ `trace!`

ข้อผิดพลาดที่พบบ่อยที่สุดคือการใช้ `info!` สำหรับทุกอย่าง (ทำให้ log รกจนหา signal จริงไม่เจอ) หรือใช้ `error!`
สำหรับสถานการณ์ที่จริง ๆ เป็นแค่ business logic ปกติ (เช่น "ผู้ใช้กรอกรหัสผ่านผิด" ไม่ควรเป็น `error!` — มันเป็น
สถานการณ์ปกติที่ระบบ auth ต้อง handle ได้ ควรเป็นแค่ `warn!` หรือแม้แต่ `info!` เท่านั้น)

#### หมายเหตุขั้นสูง: ตัด `debug!`/`trace!` ออกจาก Binary ตอน Compile ด้วย Feature Flag

`log` crate มี feature พิเศษ (`release_max_level_info`, `release_max_level_warn`, ฯลฯ — เชื่อมกับความเข้าใจ
เรื่อง `[features]` จาก Part 17/35 และ Part 48 หัวข้อ 48.2) ที่ทำให้ macro ของ level ที่สูงกว่าที่กำหนด **ถูกตัด
ออกจาก binary ทั้งหมดตอน compile ใน release mode** (ไม่ใช่แค่ filter ตอน runtime) — มีประโยชน์กับโค้ดที่มี
`trace!`/`debug!` ที่ format string ซับซ้อนมากในลูปที่รันบ่อย เพราะแม้ `RUST_LOG` จะกรองไม่ให้ print ออกมาตอน
runtime แต่การสร้าง `String` จาก format string ก็ยังเกิดขึ้นก่อนที่ logger จะเช็ค level (เป็นข้อจำกัดที่มีอยู่จริง
ของ `log` — ต่างจาก `tracing` ที่จะเห็นในหัวข้อถัดไปว่าออกแบบมาให้เช็ค level ได้เร็วกว่าในหลายกรณี) — การตัดออก
ตอน compile จึงช่วยประสิทธิภาพได้จริงสำหรับโค้ดที่อ่อนไหวต่อ performance มาก ๆ

### 60.4b การ Log Error พร้อม Error Chain เต็มรูปแบบ (เชื่อมกับ Part 12 และ Part 30-31)

หัวข้อ 60.4 อธิบายว่า `error!` เหมาะกับ "สิ่งที่ไม่ควรเกิดขึ้น" — แต่พอถึงเวลาต้อง log error จริง คำถามที่ตามมา
คือ **จะ log ด้วยรูปแบบไหนถึงจะให้ข้อมูลครบที่สุดสำหรับคนที่ต้องมาไล่ debug ทีหลัง** จำได้จาก **Part 12** และ
**Part 30-31** ว่า error type ที่ดีมัก implement `std::error::Error` พร้อมเมธอด `source()` ที่ไล่กลับไปยัง
error ตัวก่อนหน้าที่เป็นสาเหตุจริง ๆ (error chain) — นี่คือตัวอย่าง error ที่มีสอง "ชั้น" ซ้อนกัน แบบเดียวกับที่
Part 30 สอนให้เขียน custom error type ที่ wrap error อื่นไว้:

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct DbConnectionError {
    host: String,
}

impl fmt::Display for DbConnectionError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "เชื่อมต่อ database ที่ {} ไม่สำเร็จ", self.host)
    }
}

impl Error for DbConnectionError {}

#[derive(Debug)]
struct OrderServiceError {
    order_id: u32,
    source: DbConnectionError,
}

impl fmt::Display for OrderServiceError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "ไม่สามารถประมวลผลคำสั่งซื้อ #{}", self.order_id)
    }
}

impl Error for OrderServiceError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        Some(&self.source)
    }
}

fn connect_db(host: &str) -> Result<(), DbConnectionError> {
    Err(DbConnectionError { host: host.to_string() })
}

fn process_order(order_id: u32) -> Result<(), OrderServiceError> {
    connect_db("db.internal").map_err(|source| OrderServiceError { order_id, source })?;
    Ok(())
}

fn main() {
    env_logger::init();

    if let Err(e) = process_order(42) {
        // log::error! ด้วย `{e}` (Display) ให้แค่ข้อความระดับบนสุด
        log::error!("ประมวลผลคำสั่งซื้อล้มเหลว: {e}");
        // log::error! ด้วย `{e:?}` (Debug) เห็นโครงสร้างเต็มรวม field ภายใน แต่ยังไม่ไล่ source() ให้อัตโนมัติ
        log::error!("รายละเอียดแบบ Debug: {e:?}");
        // ไล่ error chain เต็มรูปแบบผ่าน source() ตาม Part 30-31 สอนไว้
        let mut source = e.source();
        let mut depth = 1;
        while let Some(err) = source {
            log::error!("  สาเหตุระดับที่ {depth}: {err}");
            source = err.source();
            depth += 1;
        }
    }
}
```

รันจริงด้วย `RUST_LOG=error` ได้:

```
[2026-09-26T23:14:12Z ERROR error_chain_demo] ประมวลผลคำสั่งซื้อล้มเหลว: ไม่สามารถประมวลผลคำสั่งซื้อ #42
[2026-09-26T23:14:12Z ERROR error_chain_demo] รายละเอียดแบบ Debug: OrderServiceError { order_id: 42, source: DbConnectionError { host: "db.internal" } }
[2026-09-26T23:14:12Z ERROR error_chain_demo]   สาเหตุระดับที่ 1: เชื่อมต่อ database ที่ db.internal ไม่สำเร็จ
```

สังเกตความต่างของสามบรรทัด: **`{e}` (Display)** ให้แค่ข้อความระดับบนสุดที่มนุษย์อ่านง่ายแต่ไม่บอกสาเหตุจริง
("ไม่สามารถประมวลผลคำสั่งซื้อ #42" — แล้วทำไมล่ะ?) **`{e:?}` (Debug)** ให้โครงสร้างข้อมูลเต็มรวม field ภายใน
ทั้งหมดในก้อนเดียว มีประโยชน์มากตอน debug แต่รูปแบบขึ้นกับว่า struct นั้น derive `Debug` มายังไง ไม่มีมาตรฐาน
ที่แน่นอน และ**การไล่ `source()` เอง**ให้เห็นสาเหตุแต่ละชั้นเป็นข้อความอ่านง่ายทีละบรรทัด ชัดเจนที่สุดสำหรับคน
อ่าน log จริง — ในโค้ด production ที่ใช้ `anyhow` (Part 31) มักมี helper หรือใช้ `{:#}` (alternate Display)
ที่ `anyhow::Error` implement ไว้ให้ไล่ chain ให้อัตโนมัติแล้ว ไม่ต้องเขียน loop เองแบบนี้ — แต่หลักการเบื้องหลัง
เหมือนกันเป๊ะ: **log error ที่มี context ครบทุกชั้น ดีกว่า log แค่ข้อความสุดท้ายที่ไม่บอกอะไรเกี่ยวกับต้นตอจริง**

**หลักปฏิบัติที่แนะนำ**: log error ที่ **"ขอบของระบบ"** (error boundary — จุดที่ error จะไม่ถูก propagate
ต่อไปอีกแล้ว เช่นจุดสุดท้ายที่ HTTP handler จับ error ก่อนตอบ response กลับไป) เท่านั้น **อย่า log error
ทุกครั้งที่มันถูก propagate ผ่าน `?`** — ถ้า log ทุกชั้นที่ error เดินทางผ่าน (เช่น log ทั้งใน `connect_db`,
ทั้งใน `process_order`, ทั้งใน caller ของ `process_order`) จะได้ log ซ้ำกันสามครั้งสำหรับ error เดียวกัน ทำให้
เข้าใจผิดว่าเกิด error สามครั้งทั้งที่จริงมีแค่ครั้งเดียว — ปล่อยให้ `?` (Part 12) ส่ง error ขึ้นไปเรื่อย ๆ
จนถึงจุดที่จัดการจริง แล้ว log แค่จุดนั้นจุดเดียว

### 60.5 ปัญหาของ `log` ธรรมดาในโค้ด Async: เมื่อหลาย Task พิมพ์ Log ปนกัน

ทุกตัวอย่างที่ผ่านมาเป็นโค้ด **synchronous** ล้วน ๆ — เรียกฟังก์ชันหนึ่งจบก่อนไปเรียกฟังก์ชันถัดไป ลำดับของ log
ที่พิมพ์ออกมาจึงตรงกับลำดับการทำงานของโปรแกรมเป๊ะ ไม่มีความกำกวมเลย

แต่จำได้จาก **Part 48** ไหมว่า Tokio scheduler ทำงานแบบ **cooperative multitasking** — task หลายตัวถูก spawn
ไปพร้อมกัน (`tokio::spawn`) แล้ว **สลับกันทำงานบน worker thread เดียวกัน** ทุกครั้งที่ task หนึ่งเจอ `.await`
ที่ยังไม่พร้อม มันจะ "คืนการควบคุม" ให้ scheduler ไปทำ task อื่นก่อน แล้วกลับมาทำต่อทีหลัง — นี่คือกลไกที่ทำให้
async มีประสิทธิภาพสูง แต่มันสร้างปัญหาใหม่ที่ไม่มีในโลก synchronous: **ลำดับการทำงานของ statement ในโค้ดไม่ตรง
กับลำดับเวลาจริงที่มันถูก execute อีกต่อไป** เพราะ task ต่าง ๆ แข่งกันสลับ execute ตามจังหวะที่ I/O/timer พร้อม

มาดูปัญหานี้แบบจับต้องได้ — จำลอง "request handler" 3 ตัวที่ทำงานพร้อมกันจริงผ่าน `tokio::spawn`:

```rust
use std::time::Duration;

async fn handle_request(request_id: u32) {
    log::info!("เริ่มประมวลผล request");
    tokio::time::sleep(Duration::from_millis(30)).await;
    log::info!("อ่านข้อมูลจาก database เสร็จแล้ว");
    tokio::time::sleep(Duration::from_millis(20)).await;
    log::info!("ประมวลผลเสร็จสมบูรณ์ (request_id เดิมคือ {request_id} แต่ log ด้านบนไม่รู้เรื่องนี้เลย)");
}

#[tokio::main]
async fn main() {
    env_logger::init();
    let mut handles = Vec::new();
    for id in 1..=3 {
        handles.push(tokio::spawn(handle_request(id)));
    }
    for h in handles {
        let _ = h.await;
    }
}
```

รันจริงด้วย `RUST_LOG=info` ได้ output ที่พิสูจน์ปัญหาชัดเจนที่สุด:

```
[2026-09-26T23:00:51Z INFO  log_plain_async] เริ่มประมวลผล request
[2026-09-26T23:00:51Z INFO  log_plain_async] เริ่มประมวลผล request
[2026-09-26T23:00:51Z INFO  log_plain_async] เริ่มประมวลผล request
[2026-09-26T23:00:52Z INFO  log_plain_async] อ่านข้อมูลจาก database เสร็จแล้ว
[2026-09-26T23:00:52Z INFO  log_plain_async] อ่านข้อมูลจาก database เสร็จแล้ว
[2026-09-26T23:00:52Z INFO  log_plain_async] อ่านข้อมูลจาก database เสร็จแล้ว
[2026-09-26T23:00:52Z INFO  log_plain_async] ประมวลผลเสร็จสมบูรณ์ (request_id เดิมคือ 1 แต่ log ด้านบนไม่รู้เรื่องนี้เลย)
[2026-09-26T23:00:52Z INFO  log_plain_async] ประมวลผลเสร็จสมบูรณ์ (request_id เดิมคือ 3 แต่ log ด้านบนไม่รู้เรื่องนี้เลย)
[2026-09-26T23:00:52Z INFO  log_plain_async] ประมวลผลเสร็จสมบูรณ์ (request_id เดิมคือ 2 แต่ log ด้านบนไม่รู้เรื่องนี้เลย)
```

**นี่คือปัญหาตัวจริง**: บรรทัด `"เริ่มประมวลผล request"` ปรากฏ**สามครั้งเหมือนกันทุกตัวอักษร** — ไม่มีทางบอกได้
เลยว่าบรรทัดแรกเป็นของ request ไหน บรรทัดที่สองเป็นของ request ไหน จากข้อมูลที่มีอยู่ในบรรทัด log เอง (สังเกตว่า
ลำดับที่ "เสร็จสมบูรณ์" ออกมาคือ 1, 3, 2 — ไม่ตรงกับลำดับที่ spawn คือ 1, 2, 3 เลย เพราะ latency สุ่มเล็กน้อย
ระหว่าง task ทำให้ลำดับการ resume จริงต่างจากลำดับ spawn) ในระบบ production จริงที่รับ request จำนวนมากพร้อมกัน
สถานการณ์นี้จะยิ่งเลวร้ายกว่านี้มาก — คุณจะเห็น log หลายพันบรรทัดปนกันจากหลายร้อย request พร้อมกัน และเมื่อมี
request หนึ่งที่ error จะไม่มีทางรู้เลยว่า log บรรทัดอื่น ๆ ก่อนหน้าที่เกี่ยวข้องกับ request ที่ error นั้นคือ
บรรทัดไหนบ้าง

**เหตุผลเชิงลึกว่าทำไม `log` แก้ปัญหานี้ไม่ได้โดยกำเนิด**: `log` ถูกออกแบบขึ้นก่อนที่ async/await จะเป็นเรื่อง
ปกติในวงการเขียนโปรแกรม — แนวคิดพื้นฐานของมันคือ "log record หนึ่งรายการ = ข้อความ + level + module path ที่
เรียก" **ไม่มีแนวคิดเรื่อง "การทำงานเชิงตรรกะปัจจุบัน" (current logical operation/task) อยู่ในโมเดลของมันเลย**
— logger กลางที่ `log` ส่ง record ไปให้เป็น global เดียวที่ใช้ร่วมกันทุก thread/task ไม่มีกลไกให้ "ผูก" ข้อมูล
เพิ่มเติม (เช่น request id) เข้ากับ record โดยอัตโนมัติตาม task ที่กำลัง execute อยู่ ณ ขณะนั้น การจะแก้ปัญหานี้
ด้วย `log` ตรง ๆ ต้องเขียน `request_id` เข้าไปในทุกข้อความด้วยมือ (`log::info!("[req={request_id}] เริ่ม
ประมวลผล request")`) ซึ่งทำได้แต่รกและเสี่ยงลืมมากถ้ามีหลายจุดในโค้ด — และยังไม่ครอบคลุมกรณีที่ `request_id`
ต้องส่งผ่านไปยังฟังก์ชันอื่น ๆ ที่ `handle_request` เรียกต่อไปอีกหลายชั้น

นี่คือปัญหาที่ **`tracing` crate** เกิดมาเพื่อแก้โดยเฉพาะ

#### เปรียบเทียบกับภาษาอื่น

แนวคิด facade pattern สำหรับ logging ไม่ใช่สิ่งที่ Rust คิดขึ้นเอง — เกือบทุกภาษาที่ใช้งานจริงระดับ production
มีสิ่งที่คล้ายกันในรูปแบบของตัวเอง เข้าใจ analogy นี้จะช่วยให้เห็นภาพว่า `log`/`env_logger` ไม่ใช่อะไรที่แปลก
ใหม่ในวงการ:

- **Python**: module `logging` ใน standard library ทำหน้าที่คล้าย `log` เป๊ะ — มี `logging.getLogger(__name__)`
  แล้วเรียก `.info()`/`.warning()`/`.error()` โดยที่ตัวโค้ดเองไม่ต้องรู้ว่า log จะไปโผล่ที่ไหน ส่วน "handler"
  (เทียบกับ backend อย่าง `env_logger`) ต้องถูกตั้งค่าแยกต่างหาก (`logging.basicConfig(...)`) — ถ้าไม่ตั้งค่า
  handler เลยก็มีค่า default ที่ทำงานได้บ้าง (ต่างจาก `log` ของ Rust ที่ default เป็น no-op สนิท)
- **Go**: standard library `log` package แบบดั้งเดิมค่อนข้าง "ตรงไปตรงมา" (ไม่มี facade แยกจาก backend มากนัก)
  แต่ Go เวอร์ชันใหม่ (1.21+) เพิ่ม `log/slog` ที่นำแนวคิด **structured logging** (เหมือน `tracing`'s
  structured field ในหัวข้อ 60.9 เป๊ะ) เข้ามาเป็นส่วนหนึ่งของ standard library โดยตรง — สะท้อนว่า structured
  logging กลายเป็นมาตรฐานอุตสาหกรรมไปแล้ว ไม่ใช่แค่แนวคิดเฉพาะของ Rust
- **JavaScript/Node.js**: ไม่มี logging facade ใน standard library เลย ระบบนิเวศใช้ third-party package อย่าง
  `winston` หรือ `pino` ตรง ๆ (มักผูกกับ backend ที่เลือกไว้แต่แรกมากกว่า Rust ที่แยก facade ออกจาก backend
  อย่างชัดเจน) — นี่คือจุดที่ facade pattern ของ Rust's `log` มีข้อดีจริง: library crate ต่าง ๆ ในระบบนิเวศ
  Rust ไม่ต้อง "เลือกฝ่าย" ว่าจะผูกกับ backend logging ตัวไหน ต่างจาก JS ecosystem ที่บาง package อาจผูกกับ
  `winston` โดยตรงจนขัดกับ choice ของแอปที่เอาไปใช้

ส่วนแนวคิด **span** ของ `tracing` มี analogy ที่ตรงที่สุดคือ **distributed tracing** ในระบบ microservice ทั่วไป
(OpenTelemetry spans, Jaeger, Zipkin) — แนวคิดเรื่อง "หน่วยงานหนึ่งชิ้นที่มีจุดเริ่ม/จบ และมี child span ซ้อนกัน
ได้" เหมือนกันเป๊ะไม่ว่าจะเป็นในโปรแกรมเดียว (`tracing` ในบทนี้) หรือข้ามหลาย service ทั้งระบบ (Part 81, Part
98-99) — นี่ไม่ใช่เรื่องบังเอิญ เพราะ `tracing` ถูกออกแบบมาให้เชื่อมต่อกับระบบ distributed tracing มาตรฐานได้
โดยตรงตั้งแต่แรก

### 60.6 `tracing` Crate: Async-Aware Structured Logging ยุคใหม่

**`tracing`** คือ crate ที่พัฒนาโดยทีม Tokio เอง ออกแบบมาให้เป็น**ซุปเปอร์เซ็ตที่ทรงพลังกว่า** `log` — ทำได้
ทุกอย่างที่ `log` ทำได้ (macro ระดับ level เหมือนกัน) แต่เพิ่มแนวคิดใหม่ที่สำคัญมากสองอย่าง: **span** (จะอธิบาย
เต็มรูปแบบในหัวข้อถัดไป) และ **structured field** (หัวข้อ 60.9) — ทั้งสองอย่างนี้คือกุญแจที่ทำให้ `tracing`
เหมาะกับโค้ด async โดยเฉพาะ

```bash
cargo add tracing
cargo add tracing-subscriber --features env-filter
```

```toml
[dependencies]
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
```

macro ระดับ event (คู่เทียบของ `log::info!` ฯลฯ) มี syntax คล้ายกันมาก แต่รองรับ **structured field** ที่จะ
อธิบายเต็มรูปแบบในหัวข้อ 60.9:

```rust
fn main() {
    tracing_subscriber::fmt::init();

    let user_id = 42;
    tracing::info!(user_id, action = "login", "user logged in");
    tracing::warn!(user_id, attempts = 3, "หลาย attempt ผิดพลาดก่อนสำเร็จ");
    tracing::debug!(cache_hit = false, "ไม่พบข้อมูลใน cache ต้อง query database");

    // เทียบกับสไตล์เดิมที่เป็นข้อความล้วนๆ ไม่มี field แยก
    println!("(เทียบ) println! แบบเดิม: user {user_id} logged in");
}
```

รันจริงด้วย `RUST_LOG=debug` ได้ (ตัด ANSI color code ที่ terminal จริงใส่มาด้วยออกเพื่อความอ่านง่าย):

```
2026-09-26T23:01:10.553588Z  INFO tracing_basic: user logged in user_id=42 action="login"
2026-09-26T23:01:10.553640Z  WARN tracing_basic: หลาย attempt ผิดพลาดก่อนสำเร็จ user_id=42 attempts=3
2026-09-26T23:01:10.553659Z DEBUG tracing_basic: ไม่พบข้อมูลใน cache ต้อง query database cache_hit=false
(เทียบ) println! แบบเดิม: user 42 logged in
```

สังเกตว่า `user_id=42` และ `action="login"` ปรากฏเป็น**ส่วนต่อท้าย**ของบรรทัดในรูปแบบ `key=value` ที่แยกจาก
ข้อความหลักอย่างชัดเจน — ต่างจาก `println!` ในบรรทัดสุดท้ายที่ `42` ถูก "ฝัง" เข้าไปในข้อความจนดึงกลับมาเป็น
ข้อมูลแยกไม่ได้อีก (รายละเอียดเรื่องนี้จะขยายต่อในหัวข้อ 60.9)

**`tracing` ไม่ได้มาแทนที่ `log` โดยสมบูรณ์ในทุกกรณี** — มันเป็น**ตัวเลือกที่ทรงพลังกว่า**สำหรับงานที่ต้องการ
ความสามารถเพิ่มเติมนี้ (หัวข้อ 60.10 จะให้แนวทางตัดสินใจแบบละเอียด)

### 60.7 Span และ `#[tracing::instrument]`: แก้ปัญหา Log ปนกันใน Async

#### Span คืออะไร

**Span** คือแนวคิดหลักที่ `log` ไม่มี — มันแทน**ช่วงเวลาการทำงาน**หนึ่งช่วง (ไม่ใช่แค่ "จุดเวลาเดียว" แบบที่
event หนึ่งตัวเป็น) ที่มี**จุดเริ่มต้นและจุดสิ้นสุดชัดเจน** และที่สำคัญที่สุดคือ **span "ครอบ" event ทุกตัวที่
เกิดขึ้นระหว่างที่มันยัง active อยู่โดยอัตโนมัติ** — ไม่ว่า event นั้นจะเกิดในฟังก์ชันเดียวกันหรือถูกเรียกต่อไป
อีกหลายชั้นก็ตาม เมื่อ event เกิดขึ้น มันจะถูก "ผนวก" เข้ากับข้อมูลของ span ที่ active อยู่ ณ ขณะนั้นโดยอัตโนมัติ
โดยที่ตัว event เองไม่ต้องรับรู้เรื่อง span เลยแม้แต่นิดเดียว

จุดที่ทำให้ span แก้ปัญหาของหัวข้อ 60.5 ได้เต็มรูปแบบคือ **span ยังคง active อยู่ได้ข้าม `.await` point** —
ต่างจากการพยายามแก้ปัญหาด้วยตัวแปร local ธรรมดา (ซึ่งถ้า task ถูก scheduler สลับไปทำ task อื่นตอน `.await`
ตัวแปร local ก็ยังอยู่ครบใน stack ของ task นั้น แต่ไม่มีกลไกอัตโนมัติให้ "แนบ" มันเข้ากับทุก log ที่เกิดขึ้น) —
span ทำสิ่งนี้ให้อัตโนมัติผ่านกลไกภายในที่ผูกกับ **`Future`** ของ task นั้นเอง (จำได้จาก Part 47 ว่า future คือ
state machine ที่ถูก poll ซ้ำ ๆ — span ผูกตัวเองเข้ากับ future นั้นตรง ๆ ผ่านทุกครั้งที่มันถูก poll)

#### `#[tracing::instrument]`: Attribute Macro ที่สร้าง Span ให้อัตโนมัติ

เขียน span มือเองได้ (`tracing::info_span!(...)` แล้ว `.enter()`) แต่วิธีที่สะดวกและใช้บ่อยที่สุดคือ
**`#[tracing::instrument]`** — attribute macro (จำได้จาก **Part 44** ว่า attribute macro รับ item หนึ่งชิ้น
แล้วคืน item ใหม่ที่มีโค้ดห่ออยู่ — แนวคิดเดียวกับ `#[tokio::main]` ที่ Part 48 อธิบายไว้) ที่ **สร้าง span ให้
ครอบทั้งฟังก์ชันโดยอัตโนมัติ** — span นั้น active ตั้งแต่ฟังก์ชันเริ่ม จนถึงฟังก์ชัน return (รวมข้าม `.await`
ทุกจุดในฟังก์ชันนั้น) โดยไม่ต้องเขียน `info_span!`/`.enter()` เองเลย

ชื่อของ span ที่สร้างขึ้นเป็นชื่อฟังก์ชันโดยอัตโนมัติ และ **parameter ของฟังก์ชันจะถูกแนบเป็น field ของ span
ให้อัตโนมัติด้วย** (ปรับแต่งได้ผ่าน `fields(...)` ในตัว attribute เอง หรือใช้ `skip(...)` ถ้าไม่ต้องการให้
parameter ตัวไหนแนบไป เช่น parameter ที่ไม่ implement `Debug` หรือมีขนาดใหญ่เกินไป)

#### ตัวอย่างใหญ่: Request Handler จำลองพร้อม Span เต็มรูปแบบ

มาแก้ตัวอย่างจากหัวข้อ 60.5 ด้วย `tracing` เต็มรูปแบบ — สังเกตว่าโครงสร้างฟังก์ชันเหมือนเดิมทุกอย่าง สิ่งที่
เพิ่มเข้ามาแค่ attribute `#[instrument]` กับเปลี่ยน `log::info!` เป็น `tracing::info!`:

```rust
use std::time::Duration;
use tracing::instrument;

#[instrument(fields(request_id = request_id))]
async fn fetch_from_db(request_id: u32) -> u64 {
    tracing::debug!("ส่ง query ไปที่ database");
    // จำลอง latency ที่ต่างกันไปตาม request เพื่อบังคับให้ execution ของแต่ละ task สลับกัน (interleave) จริง
    let latency: u64 = 15 + (request_id as u64 % 3) * 12;
    tokio::time::sleep(Duration::from_millis(latency)).await;
    tracing::info!(rows = 3, "อ่านข้อมูลจาก database เสร็จแล้ว");
    latency
}

#[instrument(fields(request_id = request_id))]
async fn process_payload(request_id: u32, rows_latency: u64) -> u64 {
    tracing::debug!("เริ่มประมวลผล payload");
    tokio::time::sleep(Duration::from_millis(5)).await;
    let result = rows_latency * 2;
    tracing::info!(result, "ประมวลผลเสร็จสมบูรณ์");
    result
}

#[instrument(fields(request_id = request_id))]
async fn handle_request(request_id: u32) {
    tracing::info!("เริ่มรับ request ใหม่");
    let latency = fetch_from_db(request_id).await;
    let result = process_payload(request_id, latency).await;
    tracing::info!(result, "ส่ง response กลับให้ client แล้ว");
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let mut handles = Vec::new();
    for id in 1..=3 {
        handles.push(tokio::spawn(handle_request(id)));
    }
    for h in handles {
        let _ = h.await;
    }
}
```

**อ่านโค้ดทีละส่วน**:

- `#[instrument(fields(request_id = request_id))]` บน `handle_request` — สร้าง span ชื่อ `handle_request` ที่
  มี field `request_id` แนบอยู่ (ค่ามาจาก parameter `request_id` ของฟังก์ชันเอง) span นี้ active ตั้งแต่เริ่ม
  ฟังก์ชันจนจบ **รวมข้าม `.await` ทั้งสองจุด** (`fetch_from_db(...).await` และ `process_payload(...).await`)
- `fetch_from_db` และ `process_payload` ก็มี `#[instrument]` ของตัวเองเช่นกัน — เมื่อถูกเรียกจากภายใน
  `handle_request` span ของมันจะเป็น **"span ลูก" (child span) ของ `handle_request`** โดยอัตโนมัติ (span
  ทำงานเป็น**stack ซ้อนกัน**ตามลำดับการเรียกฟังก์ชันจริง) ทำให้ event ที่เกิดใน `fetch_from_db` ถูกแนบทั้ง
  context ของ `handle_request` **และ** `fetch_from_db` พร้อมกัน
- `tracing::info!(rows = 3, "...")` และ `tracing::info!(result, "...")` — structured field ที่จะอธิบายเต็ม
  รูปแบบในหัวข้อ 60.9 (สังเกตว่า `result` แบบไม่มี `=` เป็น shorthand หมายถึง `result = result` คือใช้ชื่อ
  ตัวแปรเป็นชื่อ field ไปเลย)

รันจริงด้วย `RUST_LOG=info` (ตัด ANSI color code ออก):

```
2026-09-26T23:01:10.420266Z  INFO handle_request{request_id=3}: tracing_async: เริ่มรับ request ใหม่
2026-09-26T23:01:10.420273Z  INFO handle_request{request_id=2}: tracing_async: เริ่มรับ request ใหม่
2026-09-26T23:01:10.420276Z  INFO handle_request{request_id=1}: tracing_async: เริ่มรับ request ใหม่
2026-09-26T23:01:10.436550Z  INFO handle_request{request_id=3}:fetch_from_db{request_id=3}: tracing_async: อ่านข้อมูลจาก database เสร็จแล้ว rows=3
2026-09-26T23:01:10.442898Z  INFO handle_request{request_id=3}:process_payload{rows_latency=15 request_id=3}: tracing_async: ประมวลผลเสร็จสมบูรณ์ result=30
2026-09-26T23:01:10.443003Z  INFO handle_request{request_id=3}: tracing_async: ส่ง response กลับให้ client แล้ว result=30
2026-09-26T23:01:10.448156Z  INFO handle_request{request_id=1}:fetch_from_db{request_id=1}: tracing_async: อ่านข้อมูลจาก database เสร็จแล้ว rows=3
2026-09-26T23:01:10.454478Z  INFO handle_request{request_id=1}:process_payload{rows_latency=27 request_id=1}: tracing_async: ประมวลผลเสร็จสมบูรณ์ result=54
2026-09-26T23:01:10.454561Z  INFO handle_request{request_id=1}: tracing_async: ส่ง response กลับให้ client แล้ว result=54
2026-09-26T23:01:10.459833Z  INFO handle_request{request_id=2}:fetch_from_db{request_id=2}: tracing_async: อ่านข้อมูลจาก database เสร็จแล้ว rows=3
2026-09-26T23:01:10.466182Z  INFO handle_request{request_id=2}:process_payload{rows_latency=39 request_id=2}: tracing_async: ประมวลผลเสร็จสมบูรณ์ result=78
2026-09-26T23:01:10.466256Z  INFO handle_request{request_id=2}: tracing_async: ส่ง response กลับให้ client แล้ว result=78
```

**นี่คือคำตอบเต็มรูปแบบของปัญหาในหัวข้อ 60.5** — เทียบบรรทัดแรกสามบรรทัด: `handle_request{request_id=3}`,
`handle_request{request_id=2}`, `handle_request{request_id=1}` — แม้ข้อความหลัก ("เริ่มรับ request ใหม่")
จะเหมือนกันทุกตัวอักษรเหมือนในตัวอย่าง `log` ธรรมดา แต่ครั้งนี้**ทุกบรรทัดมี span context นำหน้าบอกชัดเจนว่า
เป็นของ request ไหน** — และสังเกตความลึกของ span ที่เพิ่มขึ้นเมื่อ event เกิดในฟังก์ชันที่ถูกเรียกซ้อนลงไป เช่น
`handle_request{request_id=3}:fetch_from_db{request_id=3}:` (สอง span ซ้อนกัน คั่นด้วย `:`) บอกทั้งว่า event
นี้เกิดตอนอยู่ใน `fetch_from_db` **และ**บอกด้วยว่า `fetch_from_db` ตัวนี้ถูกเรียกมาจาก `handle_request` ของ
request ไหน — ข้อมูลนี้**ไม่มีทางได้มาด้วย `log` ธรรมดาโดยไม่เขียน request_id ด้วยมือทุกจุด**

ลองรันด้วย `RUST_LOG=debug` เพื่อเห็น field ที่ซับซ้อนขึ้นอีกขั้น — สังเกต span ของ `process_payload` ที่มีทั้ง
`rows_latency` (parameter ของฟังก์ชัน ถูกแนบอัตโนมัติเพราะไม่ได้ `skip`) และ `request_id` (ที่ระบุเพิ่มผ่าน
`fields(...)`) ปนกันอยู่ในวงเล็บเดียว:

```
2026-09-26T23:00:58.531255Z DEBUG handle_request{request_id=3}:process_payload{rows_latency=15 request_id=3}: tracing_async: เริ่มประมวลผล payload
```

นี่คือหลักฐานที่จับต้องได้ว่า `tracing` ทำ**สิ่งที่ `log` ทำไม่ได้เลยโดยกำเนิด**: ให้ log ทุกบรรทัดที่มาจาก
ฟังก์ชันเดียวกัน (โค้ดเดียวกันเป๊ะ) แต่ถูกเรียกจากหลาย "การทำงานเชิงตรรกะ" (logical operation/request) ที่
ต่างกัน สามารถระบุที่มาได้ถูกต้อง 100% แม้ execution ของทั้งสามจะสลับกันไปมาจริงบน worker thread เดียวกันตาม
กลไก cooperative multitasking ของ Part 48

### 60.8 `tracing_subscriber`: `env_logger` เทียบเท่าของฝั่ง `tracing`

`tracing` เองก็เป็น facade เหมือน `log` — มัน**ไม่ได้กำหนด**ว่า span/event จะถูกแสดงผลยังไง แค่ประกาศ trait
`Subscriber` ที่ backend ต้อง implement (คู่เทียบของ trait `Log` ของฝั่ง `log`) **`tracing_subscriber`** คือ
crate ที่ implement backend นี้ให้พร้อมใช้ — คู่เทียบของ `env_logger` ฝั่ง `tracing`

```rust
fn main() {
    tracing_subscriber::fmt::init();
    // ...
}
```

`tracing_subscriber::fmt::init()` คือรูปแบบเรียกที่สั้นที่สุด — ตั้ง subscriber ที่ format เป็น text อ่านง่าย
(แบบที่เห็นในทุกตัวอย่างก่อนหน้า) และควบคุมด้วย **`RUST_LOG`** แบบเดียวกันกับ `env_logger` เป๊ะ (เพราะ feature
`env-filter` ที่เปิดไว้ทำให้ `fmt::init()` อ่าน `RUST_LOG` โดยอัตโนมัติเช่นกัน) — นี่คือเหตุผลที่ทุกตัวอย่างใน
บทนี้สลับใช้ `RUST_LOG=info`/`RUST_LOG=debug` ได้กับทั้งฝั่ง `log`+`env_logger` และฝั่ง `tracing`+
`tracing_subscriber` โดยไม่ต้องเรียนรู้ syntax ใหม่เลย

#### `EnvFilter` แบบละเอียด: ควบคุมด้วยโค้ดเมื่อไม่พึ่ง `RUST_LOG` อย่างเดียว

ถ้าต้องการกำหนด default filter ในโค้ด (เผื่อไม่มีการตั้ง `RUST_LOG` มาก่อน คล้ายกับ `Builder::filter_level`
ฝั่ง `env_logger`) ใช้ `EnvFilter` ตรง ๆ ผ่าน builder ของ `fmt`:

```rust
use tracing_subscriber::EnvFilter;

fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")),
        )
        .init();
    // ...
}
```

`EnvFilter::try_from_default_env()` พยายามอ่าน `RUST_LOG` ก่อน — ถ้าไม่มีตั้งไว้เลย (`Err`) ให้ fallback เป็น
`"info"` แทน default เดิมของ `tracing_subscriber` (ซึ่งจริง ๆ ก็คือ `error` เหมือน `env_logger`) — pattern นี้
มีประโยชน์มากตอน deploy จริงที่อยากให้ default level (ตอนไม่ได้ตั้ง `RUST_LOG` มา) เป็นมิตรกว่าค่า default เดิม

#### โมเดล Subscriber/Layer ที่ปรับเปลี่ยนได้: เทเซอร์สู่ Part 98-99

`tracing_subscriber` ไม่ได้ให้แค่ text formatter ตัวเดียว — มันออกแบบมาเป็น**ระบบ layer ที่ต่อกันได้** (แต่ละ
layer จัดการแง่มุมหนึ่งของการประมวลผล event/span เช่น filter, format, ส่งออกไปปลายทางต่างกัน) หนึ่งในตัวเลือก
ที่สำคัญที่สุดสำหรับระบบ production คือ **output เป็น JSON** แทน text ที่มนุษย์อ่านง่าย เพราะระบบ log
aggregation (เช่น ELK stack, Grafana Loki, Datadog) ต้องการ log ที่เป็น**ข้อมูลมีโครงสร้าง** (structured data)
ที่ query/filter ได้โดยตรง ไม่ใช่ข้อความอิสระที่ต้องมา parse ด้วย regex:

```rust
use tracing::instrument;

#[instrument]
fn handle(order_id: u32) {
    tracing::info!(order_id, status = "created", "สร้างคำสั่งซื้อใหม่");
}

fn main() {
    tracing_subscriber::fmt().json().init();
    handle(777);
}
```

รันจริงได้ output เป็น JSON หนึ่งบรรทัดต่อหนึ่ง event (**เอื้อกับระบบ log aggregation อย่างมาก** — เครื่องมือ
เหล่านั้น parse JSON ได้ตรง ๆ โดยไม่ต้องเขียน regex ที่เปราะบางมาแยกข้อความ):

```
{"timestamp":"2026-09-26T23:01:10.619265Z","level":"INFO","fields":{"message":"สร้างคำสั่งซื้อใหม่","order_id":777,"status":"created"},"target":"tracing_json","span":{"order_id":777,"name":"handle"},"spans":[{"order_id":777,"name":"handle"}]}
```

สังเกตว่า field ทั้งหมด (`order_id`, `status`) รวมถึงข้อมูล span (`spans`) ถูกเก็บเป็น JSON key/value ที่ชัดเจน
— ระบบปลายทางสามารถเขียน query แบบ "หา log ทั้งหมดที่ `order_id = 777`" ได้ตรง ๆ โดยไม่ต้อง parse ข้อความเลย
นี่คือรากฐานที่ **Part 98-99** (ในโมดูล observability ของหลักสูตรนี้) จะขยายต่อเป็นระบบเต็มรูปแบบ — เชื่อมต่อ
`tracing` เข้ากับ OpenTelemetry, ส่ง trace ไปเก็บที่ระบบรวมศูนย์, ทำ distributed tracing ข้าม microservice
หลายตัว (Part 81) และสร้าง metrics dashboard จาก data ที่ structured logging แบบนี้เก็บไว้ — สิ่งที่คุณเรียน
ในบทนี้ (span, structured field, subscriber แบบ pluggable) คือ**พื้นฐานที่จำเป็นต้องเข้าใจก่อน**จะไปถึงจุดนั้น

#### ต่อ Layer หลายตัวเข้าด้วยกันด้วย `Registry`

`tracing_subscriber::fmt::init()` เป็นทางลัดที่สร้าง subscriber ตัวเดียวที่รวมทุกอย่างไว้ในเมธอดเดียว — แต่
เบื้องหลังจริง ๆ `tracing_subscriber` ให้ **`Registry`** เป็นฐาน (core subscriber ที่จัดการ span storage) แล้ว
ให้คุณ **`.with(layer)` ต่อ layer เข้าไปกี่ตัวก็ได้** แต่ละ layer รับผิดชอบแง่มุมหนึ่งอย่างอิสระจากกัน (filter
แยกจาก format แยกจาก output ปลายทาง) — ทำให้ผสมกันเป็นชุดใหม่ได้ไม่จำกัดโดยไม่ต้องเขียน subscriber ทั้งตัวเอง
ใหม่ทุกครั้ง:

```rust
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

fn main() {
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")))
        .with(tracing_subscriber::fmt::layer())
        .init();

    tracing::info!("เริ่มทำงานผ่าน layered subscriber (registry + EnvFilter layer + fmt layer)");
    tracing::debug!("บรรทัดนี้จะไม่โชว์ถ้า default filter เป็น info");
}
```

รันจริงโดยไม่ตั้ง `RUST_LOG` เลย (fallback เป็น `"info"` ตามที่กำหนดในโค้ด):

```
2026-09-26T23:15:09.118202Z  INFO layered_demo: เริ่มทำงานผ่าน layered subscriber (registry + EnvFilter layer + fmt layer)
```

รันด้วย `RUST_LOG=debug` (เห็นบรรทัด `DEBUG` เพิ่มมา):

```
2026-09-26T23:15:09.183544Z  INFO layered_demo: เริ่มทำงานผ่าน layered subscriber (registry + EnvFilter layer + fmt layer)
2026-09-26T23:15:09.183591Z DEBUG layered_demo: บรรทัดนี้จะไม่โชว์ถ้า default filter เป็น info
```

รูปแบบ `registry().with(filter).with(format_layer).init()` นี้คือสิ่งที่โค้ด production จริงส่วนใหญ่ใช้ เพราะ
เปิดทางให้เพิ่ม layer อื่นเข้ามาต่อได้ในอนาคตโดยไม่ต้องรื้อโครงสร้างเดิม — เช่นเพิ่ม layer ที่ส่ง span/event ไปยัง
ระบบ distributed tracing แบบ OpenTelemetry (`tracing-opentelemetry` crate, พื้นฐานของ Part 98-99), เพิ่ม layer
ที่เขียน log ไปไฟล์แบบ rotate รายวัน (`tracing-appender`), หรือมี layer สำหรับ JSON คู่กับ layer สำหรับ text
พร้อมกัน (เช่น เขียน JSON ไปไฟล์เพื่อเก็บถาวร พร้อมพิมพ์ text อ่านง่ายไปที่ terminal เพื่อดูสดตอน develop) — ทั้ง
หมดนี้แค่ `.with(...)` เพิ่มเข้าไปในสาย โดยไม่กระทบ layer อื่นที่มีอยู่แล้วเลย

### 60.9 Structured Field: Log เป็นข้อมูล ไม่ใช่แค่ข้อความ

กลับมาขยายความหัวข้อที่แนะนำไว้สั้น ๆ ในหัวข้อ 60.6 — **structured field** คือการแยก "ข้อมูล" ออกจาก
"ข้อความอธิบาย" อย่างชัดเจนตอน log แทนที่จะยัดทุกอย่างรวมกันเป็น string เดียว

เทียบสองแบบนี้ที่บอกเรื่องเดียวกัน:

```rust
// แบบเดิม: ข้อมูลถูก "ฝัง" เข้าไปในข้อความจนกลายเป็น string เดียวที่แยกส่วนไม่ได้อีก
println!("user {} logged in", 42);
// หรือแม้จะใช้ log/tracing แต่เขียนแบบ format string ล้วนๆ ก็มีปัญหาเดียวกัน
log::info!("user {} logged in", 42);
```

```rust
// แบบ structured field: "42" ถูกเก็บเป็น field แยกชื่อ user_id ต่างหากจากข้อความ
tracing::info!(user_id = 42, action = "login", "user logged in");
```

output ของแบบ structured (จากหัวข้อ 60.6) คือ:

```
2026-09-26T23:01:10.553588Z  INFO tracing_basic: user logged in user_id=42 action="login"
```

**ทำไมความต่างนี้สำคัญมากในทางปฏิบัติ**: สมมติระบบของคุณมี log หลายล้านบรรทัดต่อวันสะสมอยู่ในระบบ log
aggregation แล้ววันหนึ่งต้องตอบคำถาม "user_id 42 มีปัญหาอะไรบ้างเมื่อสัปดาห์ที่แล้ว" — ถ้า log เป็นข้อความล้วน ๆ
(`"user 42 logged in"`) การค้นหาต้องใช้ text search/regex ที่เปราะบางมาก (ต้องเดารูปแบบข้อความที่แม่นเป๊ะ, พลาด
ถ้ามีเลข 42 ปรากฏในที่อื่นที่ไม่ใช่ user_id เช่น "attempt 42", "port 42xxx") แต่ถ้า log เป็น structured field
(`user_id=42` แยกเป็น key-value ชัดเจน โดยเฉพาะตอน output เป็น JSON แบบหัวข้อ 60.8) ระบบปลายทางสามารถ query
ตรง ๆ แบบฐานข้อมูลได้เลย: `WHERE fields.user_id = 42` — แม่นยำ 100% ไม่มีการตีความข้อความผิดพลาด และเร็วกว่า
มากเพราะไม่ต้อง full-text scan

field แบบ shorthand (`user_id` เฉย ๆ แทน `user_id = user_id`) ที่เห็นในตัวอย่าง `result` ของหัวข้อ 60.7 เป็น
ความสะดวกที่ `tracing` มีให้ — ใช้ชื่อตัวแปรที่มีอยู่แล้วเป็นชื่อ field ไปเลยโดยไม่ต้องเขียนชื่อซ้ำสองที นี่คือ
เหตุผลที่ประโยคในหัวข้อเป้าหมายของบทนี้พูดถึง "ทำไมมันสำคัญต่อ observability infrastructure" — Part 98-99
(และ dashboard/alerting ที่สร้างทับระบบนั้น) พึ่งพา**ข้อมูลที่มีโครงสร้างชัดเจนแบบนี้เป็นวัตถุดิบตั้งต้น**
ทั้งหมด — ถ้า log เป็นข้อความอิสระล้วน ๆ ทีมที่มาสร้าง dashboard/alert ทีหลังจะต้องมา "parse" ข้อความย้อนหลัง
ซึ่งเสี่ยงพังทุกครั้งที่มีคนแก้ format ข้อความแม้แค่นิดเดียว — structured field แก้ปัญหานี้ตั้งแต่ต้นทาง

### 60.10 `log` กับ `tracing`: เลือกอันไหนสำหรับโปรเจกต์ใหม่

หลังจากเห็นทั้งสอง crate แล้ว คำถามที่ปฏิบัติได้จริงคือ "แล้วโปรเจกต์ใหม่ควรใช้ตัวไหน"

**คำแนะนำที่ใช้ได้จริงในทางปฏิบัติ**:

| สถานการณ์ | คำแนะนำ | เหตุผล |
|---|---|---|
| Application ใหม่ (โดยเฉพาะที่มี async/Tokio) | **`tracing`** | ทรงพลังกว่า, แก้ปัญหา async attribution ได้ (หัวข้อ 60.7), structured field มาให้พร้อม, เป็นมาตรฐานที่ระบบนิเวศ web framework สมัยใหม่ (Axum, Part 62-66) ใช้เป็นค่าเริ่มต้น |
| Library crate ที่คนอื่นจะนำไปใช้ | **`log`** มักยังเป็นตัวเลือกที่ดีกว่า | dependency footprint เล็กกว่ามาก (compile เร็วกว่า, dependency tree เล็กกว่า) — library ที่ต้องการแค่ "ประกาศเหตุการณ์" ไม่จำเป็นต้องดึง `tracing` ที่หนักกว่าเข้ามา และแอปพลิเคชันที่ใช้ `tracing` ก็ยังรับ log จาก library ที่ใช้ `log` ได้ผ่าน compatibility shim |
| ไม่แน่ใจ / โปรเจกต์เล็กมาก | `log` + `env_logger` ก็เพียงพอ | เรียบง่ายกว่า เรียนรู้เร็วกว่า ถ้าไม่มีปัญหา async attribution (โปรแกรม synchronous ล้วน หรือ async ที่ไม่ซับซ้อน) ก็ไม่จำเป็นต้องใช้ของที่ทรงพลังกว่าที่ต้องการ |

#### เปรียบเทียบแบบ Feature-by-Feature

| แง่มุม | `log` | `tracing` |
|---|---|---|
| Macro ระดับ event (`error!`/`warn!`/`info!`/`debug!`/`trace!`) | มี | มี (syntax เกือบเหมือนกัน) |
| แนวคิด "span" (ช่วงเวลาการทำงานที่ event ผูกเข้าไปอัตโนมัติ) | **ไม่มี** | มี — หัวใจหลักของ crate |
| Attribute macro สร้าง span อัตโนมัติ (`#[instrument]`) | ไม่มี (ไม่มีแนวคิด span ให้ instrument) | มี |
| Structured field (`info!(key = value, "msg")`) | ไม่มี (ต้องฝัง field ใน format string เอง) | มี โดยกำเนิด |
| Active ข้าม `.await` ได้ (เหมาะกับ async) | ไม่รองรับโดยกำเนิด | รองรับโดยกำเนิด — ออกแบบมาเพื่อสิ่งนี้ |
| Backend/subscriber ที่ใช้กันแพร่หลาย | `env_logger`, `fern`, `flexi_logger`, `simple_logger` | `tracing-subscriber` (แทบจะเป็นมาตรฐานเดียว) |
| Output เป็น JSON ในตัว (ไม่ต้องเขียน formatter เอง) | ต้องพึ่ง backend เฉพาะทาง (เช่น `env_logger` ไม่มีให้ในตัว) | มีให้ในตัวผ่าน `fmt().json()` |
| ต่อกับ distributed tracing (OpenTelemetry) | ไม่รองรับโดยตรง | รองรับผ่าน `tracing-opentelemetry` (Part 98-99) |
| ขนาด dependency tree ที่ดึงมาด้วย | เล็กกว่ามาก (เหมาะกับ library) | ใหญ่กว่า (คุ้มค่าสำหรับ application ที่ใช้ความสามารถเต็มที่) |
| ความนิยมในหมู่ library crate (HTTP client, driver ต่าง ๆ) | สูงมาก — เป็นตัวเลือกเริ่มต้นของ library ส่วนใหญ่ | น้อยกว่า (แต่เพิ่มขึ้นเรื่อย ๆ) |
| ความนิยมในหมู่ web framework สมัยใหม่ (Axum ที่จะเรียน Part 62) | ใช้ได้ผ่าน bridge | เป็นค่าเริ่มต้นที่แนะนำโดยตรง |

ตารางนี้ตอกย้ำสิ่งที่หัวข้อก่อนสรุปไว้: **ไม่มีตัวเลือกที่ "ดีกว่าเสมอในทุกกรณี"** — `tracing` ทรงพลังกว่าในแทบ
ทุกมิติ **ยกเว้น** ขนาด dependency ซึ่งเป็นเหตุผลเดียวที่หนักแน่นพอที่จะทำให้ library crate จำนวนมากยังเลือก
`log` อยู่ต่อไป (หลักการเดียวกับที่ Part 17/35 และ Part 48 หัวข้อ 48.2 สอนเรื่อง feature flag — "จ่ายเฉพาะที่ใช้"
คือหลักการที่ library crate ควรยึดถือ เพราะไม่รู้ว่าแอปที่เอาไปใช้จะต้องการความสามารถระดับ `tracing` หรือไม่)

**ประเด็นสำคัญที่ทำให้การเลือกนี้ไม่ใช่ "เลือกอันใดอันหนึ่งแล้วอีกอันหายไปเลย"**: crate อย่าง `tracing-log`
มีไว้เป็น **compatibility shim** ที่เชื่อมสองระบบเข้าด้วยกัน — แอปพลิเคชันของคุณใช้ `tracing`/`tracing_subscriber`
เต็มรูปแบบได้ แล้วยัง**รับ log record จาก dependency crate ที่เขียนด้วย `log` ธรรมดา**เข้ามาแสดงในระบบเดียวกัน
ได้ ไม่ต้องรัน logger สองระบบซ้อนกัน — และมีจุดที่น่าสนใจมากที่พิสูจน์ด้วยการทดลองจริง: **`tracing_subscriber`
เปิด feature `tracing-log` เป็น default feature อยู่แล้ว** ซึ่งหมายความว่า `tracing_subscriber::fmt::init()`
เพียงอย่างเดียว**จับ `log::info!`/`log::warn!`/... จาก dependency ได้อัตโนมัติโดยไม่ต้องเรียก
`tracing_log::LogTracer::init()` เองเลย**:

```rust
// สาธิตว่า tracing_subscriber::fmt (default features มี "tracing-log" ติดมาด้วย) จับ log::info!/warn!/...
// จาก dependency crate ที่ใช้ log ธรรมดาได้เองโดยอัตโนมัติ ไม่ต้องเปิด bridge เองซ้ำ
fn some_dependency_that_only_uses_log() {
    // สมมติว่าฟังก์ชันนี้อยู่ใน dependency crate ภายนอกที่เขียนด้วย `log` ธรรมดา (ไม่รู้จัก tracing เลย)
    log::info!("นี่คือ log บรรทัดที่มาจาก dependency crate สมมติที่ใช้ log ธรรมดา");
    log::warn!("dependency crate เตือนบางอย่าง");
}

fn main() {
    // ไม่ต้องเรียก tracing_log::LogTracer::init() เอง เพราะ tracing_subscriber::fmt::init()
    // เปิด feature "tracing-log" (default feature) มาให้อยู่แล้ว จึง subscribe log::Record โดยอัตโนมัติ
    tracing_subscriber::fmt::init();

    tracing::info!("นี่คือ log บรรทัดจาก tracing โดยตรงในแอปของเราเอง");
    some_dependency_that_only_uses_log();
}
```

รันจริงด้วย `RUST_LOG=info` (ตัด ANSI ออก):

```
2026-09-26T23:01:31.433275Z  INFO bridge_demo: นี่คือ log บรรทัดจาก tracing โดยตรงในแอปของเราเอง
2026-09-26T23:01:31.433315Z  INFO bridge_demo: นี่คือ log บรรทัดที่มาจาก dependency crate สมมติที่ใช้ log ธรรมดา
2026-09-26T23:01:31.433325Z  WARN bridge_demo: dependency crate เตือนบางอย่าง
```

log จากทั้งสองระบบ (`tracing::info!` ที่เขียนเอง และ `log::info!`/`log::warn!` ที่จำลองว่ามาจาก dependency
crate ภายนอก) ถูกแสดงผลผ่าน subscriber ตัวเดียวกันได้อย่างไร้รอยต่อ — พิสูจน์คำแนะนำที่ว่า "เลือก `tracing`
สำหรับแอป ไม่ต้องกลัวว่า dependency ที่ใช้ `log` จะหายไปจาก log stream" (ถ้าเรียก
`tracing_log::LogTracer::init()` ซ้ำเข้าไปอีกทีทั้งที่ `fmt::init()` ติดตั้ง bridge ให้อยู่แล้ว จะเกิด conflict
— รายละเอียดเต็มอยู่ในหัวข้อกับดักที่ 6)

### 60.11 ตัวอย่างใหญ่ปิดท้าย: วิเคราะห์ Request Handler แบบ Async เต็มรูปแบบ

ตัวอย่างในหัวข้อ 60.7 คือตัวอย่างหลักของบทนี้ที่รวมทุกอย่างเข้าด้วยกัน — มาวิเคราะห์ภาพรวมทั้งระบบอีกครั้งเพื่อ
ให้เห็นว่าทุกส่วนที่เรียนมาทำงานประกอบกันอย่างไรในสถานการณ์ที่ใกล้เคียงกับระบบจริงที่สุด:

**โครงสร้าง pipeline**: `handle_request` (span ระดับบนสุด, มี field `request_id`) เรียก `fetch_from_db` (span
ลูก, จำลอง I/O-bound work ด้วย `tokio::time::sleep` — latency ต่างกันไปตาม `request_id` เพื่อบังคับให้ทั้งสาม
request แข่งกัน resume ไม่พร้อมกัน เหมือนสถานการณ์จริงที่ database query แต่ละครั้งใช้เวลาไม่เท่ากัน) แล้วต่อด้วย
`process_payload` (span ลูกอีกตัว, จำลอง CPU-light processing)

**สิ่งที่พิสูจน์ได้จาก output จริง**:

1. **สาม request ทำงานคู่ขนานจริง ไม่ใช่ทีละตัว** — สังเกตว่า `handle_request{request_id=3}` เริ่ม
   `fetch_from_db` ก่อน request 1 และ 2 (เพราะ latency ที่คำนวณจาก `request_id % 3` ทำให้ request 3 มี latency
   สั้นที่สุด) นี่คือพฤติกรรมที่ตรงกับที่ Part 48 สอนไว้เรื่อง `tokio::spawn` — ทั้งสาม task ถูก schedule ให้
   ทำงานสลับกันตามความพร้อมจริง ไม่ใช่ตามลำดับที่เขียนโค้ด
2. **ทุก log บรรทัดระบุที่มาได้ถูกต้อง 100%** แม้ execution ของสาม task จะสลับกันไปมา — นี่คือสิ่งที่ปัญหาใน
   หัวข้อ 60.5 พิสูจน์ไว้ว่า `log` ธรรมดาทำไม่ได้เลย
3. **Field ซ้อนกันตามลำดับการเรียกฟังก์ชันจริง** — `process_payload{rows_latency=15 request_id=3}` มีทั้ง field
   ของตัวเอง (`rows_latency`) และ field ที่ระบุเพิ่มจาก `request_id` (ทั้งสองมาจาก parameter ของฟังก์ชันตัวเอง
   ไม่ใช่จาก parent span — แต่ parent span `handle_request{request_id=3}` ก็ยังปรากฏเป็น prefix อยู่ ทำให้เห็น
   บริบทครบทั้งสองระดับ)
4. **โค้ดจริงที่เพิ่มขึ้นจากเวอร์ชัน `log` ธรรมดามีแค่**: เปลี่ยน `log::` เป็น `tracing::` และเพิ่ม
   `#[instrument(...)]` เหนือฟังก์ชัน async ทุกตัวที่ต้องการให้เป็นจุดอ้างอิงของ log ที่เกิดข้างใน — ไม่ต้อง
   เขียน `request_id` ซ้ำในทุกข้อความ log เหมือนที่ต้องทำถ้าจะแก้ปัญหาด้วย `log` ธรรมดา

นี่คือเหตุผลที่หัวข้อ 60.10 แนะนำ `tracing` สำหรับแอปพลิเคชันใหม่ที่มี async เป็นหัวใจหลัก — ต้นทุนที่เพิ่มขึ้น
(เรียนรู้แนวคิด span, เพิ่ม attribute หนึ่งบรรทัดต่อฟังก์ชัน) แลกมาด้วยความสามารถในการ debug ระบบที่ซับซ้อนได้
ง่ายขึ้นมหาศาลเมื่อระบบมีขนาดใหญ่ขึ้นและรับ concurrent request จำนวนมากขึ้นในระดับ production จริง

### 60.12 กรณีศึกษา: ใช้ Span ไล่หา Request ที่ทำงานช้าผิดปกติ

มาปิดเนื้อหาก่อน checklist ด้วยกรณีศึกษาที่จำลองสถานการณ์ debug จริง เพื่อให้เห็นว่าทุกอย่างที่เรียนมาในบทนี้
ประกอบกันแก้ปัญหาจริงได้อย่างไร — สมมติสถานการณ์: ระบบ production ของคุณรับหลายร้อย request พร้อมกันทุกวินาที
ทีม operations รายงานว่า "บาง request ตอบกลับช้าผิดปกติ แต่ไม่รู้ว่า request ไหน หรือช้าที่ขั้นตอนไหน"

**ถ้าใช้ `println!`/`log` ธรรมดา**: สิ่งที่ทำได้คือเพิ่ม `println!`/`log::info!` ในทุกจุดที่คิดว่าอาจช้า แล้ว
deploy ใหม่ รอให้ปัญหาเกิดซ้ำ (ซึ่งอาจใช้เวลาเป็นชั่วโมงหรือเป็นวันถ้าปัญหาเกิดไม่บ่อย) แล้วมานั่งไถ log หลาย
พันบรรทัดที่ปนกันจากทุก request พร้อมกัน (ปัญหาเดียวกับที่พิสูจน์ไว้ในหัวข้อ 60.5) หาบรรทัดที่ timestamp ห่างกัน
ผิดปกติ **โดยไม่รู้ด้วยซ้ำว่าสองบรรทัดที่ timestamp ห่างกันนั้นเป็นของ request เดียวกันหรือคนละ request** —
กระบวนการนี้ใช้เวลานานและมีโอกาสสรุปผิดสูงมาก

**ถ้า instrument ด้วย `tracing` ไว้ล่วงหน้าแบบตัวอย่างในหัวข้อ 60.7/60.11**: กลับไปดู log ที่เก็บไว้ (ถ้า
output เป็น JSON ตามหัวข้อ 60.8 จะยิ่งสะดวก) แล้ว **filter ด้วย `request_id`** ของ request ที่ผู้ใช้รายงานว่าช้า
(สมมติได้ request_id มาจาก response header หรือจาก error report ของผู้ใช้) จะได้ log เฉพาะของ request นั้น
เรียงตามเวลาชัดเจน ไม่ปนกับ request อื่นเลย — ดูจากตัวอย่าง output จริงในหัวข้อ 60.7 (สมมติว่านี่คือ request
ที่ถูกรายงานว่าช้า):

```
2026-09-26T23:01:10.420276Z  INFO handle_request{request_id=1}: tracing_async: เริ่มรับ request ใหม่
2026-09-26T23:01:10.448156Z  INFO handle_request{request_id=1}:fetch_from_db{request_id=1}: tracing_async: อ่านข้อมูลจาก database เสร็จแล้ว rows=3
2026-09-26T23:01:10.454478Z  INFO handle_request{request_id=1}:process_payload{rows_latency=27 request_id=1}: tracing_async: ประมวลผลเสร็จสมบูรณ์ result=54
2026-09-26T23:01:10.454561Z  INFO handle_request{request_id=1}: tracing_async: ส่ง response กลับให้ client แล้ว result=54
```

แค่เทียบ timestamp ระหว่างบรรทัด สามารถคำนวณได้ทันทีว่า **`fetch_from_db` ใช้เวลาประมาณ 27.9ms** (จาก
23:01:10.420276 ถึง 23:01:10.448156) ในขณะที่ **`process_payload` ใช้เวลาแค่ 6.3ms** — ถ้า pattern นี้เกิดขึ้น
ซ้ำ ๆ ในหลาย request ที่ถูกรายงานว่าช้า สามารถสรุปได้ทันทีว่า **`fetch_from_db` (การ query database) คือจุดที่
เป็นคอขวดจริง** ไม่ใช่ `process_payload` — ทั้งหมดนี้ได้มาจาก log ที่เก็บไว้ตามปกติ **โดยไม่ต้อง deploy โค้ดใหม่
สักบรรทัดเดียว** และไม่ต้องรอให้ปัญหาเกิดซ้ำเพื่อใส่ log เพิ่ม เพราะ instrumentation ถูกเตรียมไว้ล่วงหน้าแล้ว
ตั้งแต่ตอนเขียนโค้ด

**บทเรียนที่ได้จากกรณีศึกษานี้**: การลงทุนใส่ `#[instrument]` และ structured field ตั้งแต่ตอนเขียนโค้ดครั้งแรก
(ต้นทุนแค่ไม่กี่บรรทัดต่อฟังก์ชัน) ให้ผลตอบแทนมหาศาลตอนต้อง debug ปัญหา production จริงที่ซับซ้อน — นี่คือ
เหตุผลเชิงปฏิบัติที่หนักแน่นที่สุดที่ทำให้ทีม engineering จริงจังกับการเขียน observability เข้าไปใน**ทุก**
service ตั้งแต่วันแรก ไม่ใช่ผัดวันไปทำ "ทีหลังตอนมีเวลา" — ถึงตอนที่จำเป็นต้องใช้จริง (ปัญหา production ที่
กดดันเรื่องเวลา) มักไม่มีเวลาเหลือให้กลับไปเพิ่ม instrumentation ใหม่ทันเวลาอีกแล้ว

### 60.13 พิสูจน์ด้วยตัวเลขจริง: ต้นทุนของ Log ที่ถูก Filter ออกน้อยแค่ไหน (เชื่อมกับ Part 54-55)

คำถามที่มักเกิดขึ้นตอนพิจารณาใส่ `tracing::trace!()`/`debug!()` จำนวนมากในโค้ด hot path (โค้ดที่รันบ่อยมาก เช่น
ในลูปประมวลผลข้อมูลจำนวนมาก) คือ "การเรียก macro พวกนี้ที่ถูก filter ออกไปเลย (ไม่ได้ print อะไรจริง) มี
overhead มากแค่ไหน" — จำได้จาก **Part 54 (Performance Optimization และ Benchmarking)** ว่าหลักการที่ถูกต้องคือ
**วัดจริง ไม่เดา** มาพิสูจน์กันตรง ๆ ด้วยการเรียก `tracing::trace!()` หนึ่งล้านครั้งโดยตั้ง filter ไว้ที่ `info`
(ทำให้ `trace!` ทุกตัวถูกกรองออกก่อนจะไป format หรือ print อะไรเลย) เทียบกับลูปเปล่าที่ไม่มี `tracing` แม้แต่นิด
เดียว:

```rust
use std::time::Instant;

fn main() {
    // ตั้ง filter ไว้ที่ info เท่านั้น -> trace! ทุกตัวถูกกรองออกก่อนจะไป format/print อะไรเลย
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::new("info"))
        .init();

    const N: u32 = 1_000_000;

    let start = Instant::now();
    for i in 0..N {
        tracing::trace!(iteration = i, "ทำงานรอบที่ {i} (ถูกกรองออกเพราะ level ต่ำกว่า info)");
    }
    let with_filtered_calls = start.elapsed();

    let start2 = Instant::now();
    for i in 0..N {
        std::hint::black_box(i);
    }
    let without_calls = start2.elapsed();

    println!("เรียก tracing::trace!() ที่ถูก filter ออก {N} ครั้ง: {with_filtered_calls:?}");
    println!("loop เปล่าไม่มี tracing เลย {N} ครั้ง: {without_calls:?}");
}
```

รันจริงด้วย `cargo build --release` (สำคัญมาก — ต้องวัดใน release mode ตามหลักการของ Part 54 เพราะ debug build
มี overhead อื่นปนมาที่ไม่สะท้อนความเป็นจริง) พร้อม `RUST_LOG=info`:

```
เรียก tracing::trace!() ที่ถูก filter ออก 1000000 ครั้ง: 328.741µs
loop เปล่าไม่มี tracing เลย 1000000 ครั้ง: 322.635µs
```

ผลลัพธ์จริง: **หนึ่งล้านครั้งของ `trace!()` ที่ถูก filter ออกใช้เวลาเพิ่มขึ้นจากลูปเปล่าเพียง ~6 microseconds
รวม** (คิดเป็นเวลาเพิ่มขึ้นเฉลี่ยน้อยกว่า 1 นาโนวินาทีต่อการเรียกหนึ่งครั้ง) — นี่คือหลักฐานที่จับต้องได้ว่า
**การเช็ค level ของ `tracing` ก่อนตัดสินใจว่าจะ format/ประมวลผล field หรือไม่นั้นเร็วมากจนวัดผลกระทบแทบไม่ได้
ในทางปฏิบัติ** เหตุผลเชิงลึกคือ `tracing` ออกแบบ macro ให้เช็ค "callsite นี้ level ไหน, ปัจจุบัน enable ไหม"
ผ่าน metadata ที่ cache ไว้ตั้งแต่ compile time (ไม่ต้อง lock หรือ query อะไรที่ช้าตอน runtime) ก่อนจะไปแตะ
argument หรือ field เลยแม้แต่นิดเดียวถ้าผลเช็คคือ "ไม่ enable" — **ข้อสรุปเชิงปฏิบัติ**: ไม่ต้องกังวลเรื่อง
performance จนถึงขั้นไม่กล้าใส่ `trace!`/`debug!` ในโค้ดที่รันบ่อย ตราบใดที่ level เหล่านั้นถูก filter ออกใน
production จริง (ผ่าน `RUST_LOG`/`EnvFilter` ที่เหมาะสมตามหัวข้อ 60.4) — แต่ก็ควร**วัดจริงในโค้ดของตัวเอง**เสมอ
ถ้าสงสัย ไม่ใช่เชื่อตัวเลขจากบทเรียนนี้ไปตรง ๆ (ตัวเลขจะต่างกันไปตามเครื่องและ workload จริง — หลักการสำคัญกว่า
ตัวเลข)

### 60.14 Checklist ก่อนนำ Logging ไปใช้จริงใน Production

ก่อนปิดเนื้อหาบทนี้ มารวบตาราง checklist เชิงปฏิบัติที่สรุปทุกหลักการที่เรียนมาไว้ในที่เดียว — ใช้เป็น
เกณฑ์ตรวจสอบก่อน deploy ระบบที่มี logging/tracing จริง:

| รายการตรวจสอบ | เหตุผล (อ้างอิงหัวข้อ) |
|---|---|
| มีการเรียก `env_logger::init()` หรือ `tracing_subscriber::fmt::init()` (หรือเทียบเท่า) **ครั้งเดียว** ที่ต้นโปรแกรม | ไม่ทำแล้ว log หายไปเงียบ ๆ (60.2, กับดัก 1) เรียกซ้ำสองครั้งแล้ว panic (กับดัก 1 ส่วนขยาย) |
| ตั้ง default log level (fallback เมื่อไม่มี `RUST_LOG`) เป็นค่าที่เหมาะกับ production จริง ไม่ใช่ปล่อยตามค่า default ของ library (`error` เท่านั้น) | ค่า default ของทั้ง `env_logger`/`EnvFilter` ค่อนข้าง "เงียบ" เกินไปสำหรับใช้งานจริงที่อยากเห็น `info` เป็นอย่างน้อย (60.8) |
| เลือก level ตามเกณฑ์ "ใครอ่าน แล้วต้องทำอะไร" ไม่ใช่ตามสัญชาตญาณ | ป้องกัน alert fatigue จาก `error!` ที่ใช้ผิด และ log ท่วมจาก `info!` ที่ใช้เกิน (60.4, กับดัก 5) |
| ฟังก์ชัน async ที่เป็น "หน่วยงานเชิงตรรกะ" (request, job, transaction) มี `#[instrument]` ครอบ | ไม่มี span จะ debug ระบบที่รับ concurrent request จำนวนมากไม่ได้เลยว่า log แต่ละบรรทัดเป็นของอะไร (60.7, กับดัก 4) |
| log error ที่ **"ขอบของระบบ"** จุดเดียว ไม่ log ซ้ำหลายชั้นตามที่ error ถูก propagate ผ่าน `?` | ป้องกันความเข้าใจผิดว่า error เดียวเกิดขึ้นหลายครั้ง (60.4b) |
| ถ้าระบบมีระบบ log aggregation ปลายทาง (ELK, Loki, Datadog ฯลฯ) ใช้ output แบบ JSON (`fmt().json()`) แทน text | ให้ระบบปลายทาง query ด้วย structured field ได้ตรง ๆ ไม่ต้อง parse ข้อความอิสระ (60.8, 60.9) |
| ไม่เรียก `tracing_log::LogTracer::init()` เองถ้าใช้ `tracing_subscriber::fmt` อยู่แล้ว | default feature `tracing-log` ติดตั้ง bridge ให้แล้ว เรียกซ้ำจะ panic (60.10, กับดัก 6) |
| ทดสอบ `RUST_LOG` หลายค่าจริงก่อน deploy (`error`, `info`, `debug`) ไม่ใช่เชื่อว่า code ที่เขียนถูกต้องแน่นอน | syntax ผิดบางกรณี fail แบบเงียบสนิท ไม่มี error เตือน (60.3, กับดัก 2-3) |

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืม Init Logger Backend — Log หายไปเงียบ ๆ โดยไม่มี Error ใดๆ

นี่คือกับดักที่พบบ่อยที่สุดของทั้งบทนี้ อธิบายละเอียดแล้วในหัวข้อ 60.2 — สรุปสั้น ๆ ที่นี่พร้อมหลักฐานจริง:

```rust
fn main() {
    log::error!("นี่คือ error");
    log::warn!("นี่คือ warn");
    log::info!("นี่คือ info");
    println!("จบโปรแกรมแล้ว — สังเกตว่าไม่มีบรรทัด log ใดๆ ปรากฏเลยด้านบน");
}
```

รันจริงได้ output แค่บรรทัดเดียวคือ `println!` เท่านั้น — **ไม่มี error, ไม่มี warning จาก compiler, ไม่มีอะไร
เตือนเลยว่ามีปัญหา** เพราะทั้ง `log` และ `tracing` ถูกออกแบบมาให้ library crate เรียก macro ได้อย่างปลอดภัยแม้
ไม่มีใคร init backend เลย (ค่าเริ่มต้นคือ no-op logger) **วิธีแก้**: ตรวจสอบเสมอว่ามีการเรียก `env_logger::init()`
(ฝั่ง `log`) หรือ `tracing_subscriber::fmt::init()` (ฝั่ง `tracing`) **ครั้งเดียวตอนต้นของ `main()`** — ถ้า
โปรแกรมรันแล้วไม่มี log อะไรออกมาเลยแม้จะตั้ง `RUST_LOG=trace` ไว้ นี่คือสิ่งแรกที่ควรเช็คก่อนสงสัยอย่างอื่น

**ส่วนขยาย — ปัญหาฝั่งตรงข้าม: เรียก `init()` ซ้ำสองครั้ง**: ถ้าโค้ดมีจุดที่เผลอเรียก `env_logger::init()`
สองครั้ง (เช่น เรียกใน helper function ที่ถูกเรียกใช้จากหลายที่ในโปรแกรม โดยแต่ละที่ไม่รู้ว่าอีกที่หนึ่งก็เรียก
เหมือนกัน) จะได้ผลตรงข้ามกับกับดักข้างบนคือ **panic ทันที** เพราะ `log` อนุญาตให้ set global logger ได้แค่
ครั้งเดียวในโปรแกรมทั้งชีวิต:

```rust
fn main() {
    env_logger::init();
    env_logger::init(); // เรียกซ้ำสองครั้งโดยไม่ตั้งใจ (เช่น เรียกใน helper function ที่ถูกเรียกจากหลายที่)
    log::info!("บรรทัดนี้จะไม่มีวันได้เห็น เพราะ panic เกิดก่อนหน้านี้แล้ว");
}
```

รันจริงได้ panic ทันที:

```
thread 'main' panicked at .../env_logger-0.11.11/src/logger.rs:901:16:
env_logger::init should not be called after logger initialized: SetLoggerError(())
```

**วิธีแก้**: ให้แน่ใจว่ามีจุดเรียก `env_logger::init()`/`tracing_subscriber::fmt::init()` **จุดเดียวเท่านั้นใน
ทั้งโปรแกรม** (ปกติคือบรรทัดแรก ๆ ของ `fn main()`) — ถ้าจำเป็นต้องเรียกจาก helper function ที่อาจถูกเรียกซ้ำ
(เช่นใน test suite ที่แต่ละ `#[test]` เรียก setup function เดียวกัน) ให้ใช้ `env_logger::try_init()` แทน (คืน
`Result` ให้จัดการเองแทนที่จะ panic) หรือห่อด้วย `std::sync::Once`/`std::sync::OnceLock` เพื่อให้แน่ใจว่า init
เกิดขึ้นแค่ครั้งเดียวจริง ๆ แม้จะถูกเรียกจากหลายจุด

### 2. RUST_LOG Syntax ผิด — บางกรณี Silent Failure, บางกรณีมี Error Message ชัดเจน

การพิมพ์ `RUST_LOG` ผิด syntax มีพฤติกรรมที่**ต่างกันระหว่าง `env_logger` กับ `tracing_subscriber`'s
`EnvFilter`** ซึ่งเป็นเรื่องที่ต้องรู้ไว้เพื่อไม่ให้เสียเวลา debug ผิดทาง

**ฝั่ง `env_logger`** — ใช้ `:` แทน `=` (ผิด separator):

```bash
RUST_LOG='log_demo:debug' cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง: **ไม่มี output อะไรเลยแม้แต่บรรทัดเดียว** (ไม่มีแม้แต่ `ERROR` ที่ควรจะเป็น default) ไม่มี error
message ใด ๆ ปรากฏออกมาทั้ง stdout และ stderr — `env_logger` parse directive ที่ผิด syntax แล้ว**ล้มเหลวแบบ
เงียบ** ทำให้ทั้ง filter กลายเป็นไม่แสดงอะไรเลย นี่คือกรณีที่อันตรายที่สุดเพราะไม่มีสัญญาณเตือนอะไรมาบอกว่า
`RUST_LOG` ที่ตั้งไว้พิมพ์ผิด

**ฝั่ง `tracing_subscriber`'s `EnvFilter`** — ใช้ space แทน comma คั่นหลาย directive:

```bash
RUST_LOG='tracing_async=debug tracing_async::fetch_from_db=trace' cargo run --quiet --bin tracing_async
```

ผลลัพธ์จริง — คราวนี้**มี error message ชัดเจนพิมพ์ออกมาทาง stderr**:

```
ignoring `tracing_async=debug tracing_async::fetch_from_db=trace`: error parsing level filter: expected one of "off", "error", "warn", "info", "debug", "trace", or a number 0-5
```

`EnvFilter` เห็นว่า directive ทั้งก้อนนี้ parse ไม่ผ่าน (เพราะไม่มี comma คั่น มันจึงพยายามตีความทั้งก้อนเป็น
`target=level` เดียว ซึ่ง "level" ที่ได้กลายเป็นข้อความยาวที่ไม่ตรงกับ level ใดเลย) จึง**เตือนแล้วข้าม directive
นี้ไปเลย** (fallback ไปใช้ default filter ซึ่งในกรณีนี้คือไม่แสดงอะไรเพราะโปรแกรมไม่มี `error!` ให้เห็น) — ดีกว่า
`env_logger` ตรงที่**มี** error message บอกให้รู้ว่า `RUST_LOG` มีปัญหา syntax แต่ก็ยังไม่ crash โปรแกรม (ยังคง
รันต่อไปด้วย filter ที่ fallback)

**วิธีแก้ทั้งสองกรณี**: จำ syntax ที่ถูกต้องให้แม่น — `module_path=level` (equals sign, ไม่ใช่ colon) และ
คั่นหลาย directive ด้วย **comma** เท่านั้น (`module1=level1,module2=level2`) ถ้า log ไม่ออกมาตามที่คาดหวังทั้งที่
init logger ถูกต้องแล้ว (กับดักที่ 1 ไม่ใช่สาเหตุ) ให้ตรวจสอบ syntax ของ `RUST_LOG` เป็นลำดับถัดไป และสังเกตว่า
ฝั่ง `tracing`'s `EnvFilter` มักจะให้ error message ที่ช่วย debug ได้ดีกว่าฝั่ง `env_logger` ในกรณีนี้

### 3. เข้าใจผิดเรื่อง Module Path สำหรับ `src/bin/*.rs` — ชื่อ Crate ไม่ใช่ชื่อ Package

อธิบายละเอียดแล้วในหัวข้อ 60.3 — สรุปสั้น ๆ พร้อมหลักฐาน: โปรเจกต์ที่ package ชื่อ `logging_demo` ใน
`Cargo.toml` แต่มีไฟล์ `src/bin/log_demo.rs` — crate name ที่แท้จริงของ binary นี้คือ **`log_demo`** (ชื่อไฟล์)
ไม่ใช่ `logging_demo` (ชื่อ package) ลองใช้ชื่อผิด:

```bash
RUST_LOG=logging_demo=trace cargo run --quiet --bin log_demo
```

ผลลัพธ์จริง: **ไม่มี output อะไรเลย** เพราะ directive `logging_demo=trace` ไม่ตรงกับ module path ใดในโปรแกรมนี้
เลย (module path จริงทั้งหมดขึ้นต้นด้วย `log_demo`) **วิธีแก้**: เช็คชื่อ crate จริงจาก error message ของ log
ที่แสดงผลออกมาก่อนหน้านี้แล้ว (สังเกตคำในวงเล็บเหลี่ยม `[... log_demo]` ของ `env_logger` หรือคำหลัง target ของ
`tracing_subscriber` เช่น `tracing_async:`) นั่นคือชื่อ module path ที่ต้องใช้ใน `RUST_LOG` ตรง ๆ — ถ้าโปรเจกต์
มีทั้ง `[lib]` และหลาย `[[bin]]`/`src/bin/*.rs` ให้ระวังว่าแต่ละ binary มีชื่อ crate ของตัวเองต่างจาก package
name เสมอถ้าตั้งชื่อไฟล์ไม่ตรงกับชื่อ package (โปรเจกต์ที่มีแค่ `src/main.rs` เดียว ไม่มีปัญหานี้ เพราะ crate
name ของมันจะเท่ากับชื่อ package ตรง ๆ)

### 4. ไม่ใส่ `#[instrument]` — Log ในโค้ด Async ยัง Attribute ไม่ได้เหมือนเดิม

อธิบายละเอียดแล้วในหัวข้อ 60.5 และ 60.7 — สรุปสั้น ๆ ที่นี่: การเปลี่ยนจาก `log::info!` เป็น `tracing::info!`
เพียงอย่างเดียว**ไม่ได้แก้ปัญหา attribution อัตโนมัติ** ถ้าไม่มี `#[instrument]` ครอบฟังก์ชัน async ที่เกี่ยวข้อง
ด้วย — `tracing::info!` ที่เรียกนอก span ใด ๆ ก็ยังพิมพ์ log เปล่า ๆ ไม่มี context นำหน้าเหมือนเดิม (จะเห็นแค่
`target: message` ธรรมดาแบบเดียวกับ `log` ทุกประการ ไม่มีวงเล็บ `handle_request{request_id=...}` นำหน้าเลย)
**วิธีแก้**: ทุกฟังก์ชัน async ที่เป็น "หน่วยงานเชิงตรรกะ" ที่อยากให้ log ข้างในระบุที่มาได้ (เช่น request
handler, background job หนึ่งงาน, การประมวลผล item หนึ่งตัวใน batch) ควรมี `#[tracing::instrument]` ครอบไว้ —
ยิ่งฟังก์ชันนั้นมี `.await` อยู่ข้างในและถูกเรียกพร้อมกันหลายครั้งผ่าน `tokio::spawn` (เหมือนตัวอย่างหลักของ
บทนี้) ยิ่งจำเป็นต้องมี `#[instrument]` มากเท่านั้น — ถ้าลืมแค่ฟังก์ชันเดียวในหลายฟังก์ชันที่ซ้อนกัน (เช่น ลืม
`#[instrument]` บน `fetch_from_db` แต่มีบน `handle_request`) log จาก `fetch_from_db` ก็ยังจะได้ context ของ
`handle_request` มา (เพราะ span เป็น stack ที่สืบทอดกัน) แต่จะไม่มีข้อมูลเฉพาะของ `fetch_from_db` เอง (เช่น
ไม่มี field ใด ๆ ที่ตั้งใจแนบไว้ที่ระดับนั้น)

### 5. เลือก Log Level ผิด — Error ที่ไม่ใช่ Error จริง หรือ Info ที่ท่วมจน Signal หาไม่เจอ

อธิบายแนวทางเลือก level ไว้ละเอียดแล้วในหัวข้อ 60.4 — กับดักที่พบจริงบ่อยสองรูปแบบ:

**(ก) ใช้ `error!` กับสถานการณ์ที่จริง ๆ เป็น business logic ปกติ** — เช่น "ผู้ใช้กรอกรหัสผ่านผิด",
"สินค้าหมดสต็อกตอนลูกค้าสั่งซื้อ" เหล่านี้เป็น**ทางเดินปกติที่ระบบต้อง handle ได้อยู่แล้ว** ไม่ใช่ข้อผิดพลาดของ
ระบบ — ถ้า log เป็น `error!` ทุกครั้งที่เกิดเหตุการณ์เหล่านี้ (ซึ่งเกิดบ่อยมากในระบบจริงที่มีผู้ใช้จำนวนมาก)
ระบบ alert ที่ต่อกับ `error!` (เช่นส่ง notification ให้ทีม on-call ตอนมี error) จะแจ้งเตือนถี่จนทีมเริ่ม
เพิกเฉย alert (alert fatigue) — พอเกิด error ที่**จริง**สำคัญ (เช่น database connection pool หมด) ก็จะถูก
กลบไปในกองของ alert ที่ไม่สำคัญจำนวนมาก **วิธีแก้**: สงวน `error!` ไว้สำหรับสถานการณ์ที่ระบบอยู่ในสถานะที่ไม่
ตรงกับ design จริง ๆ เท่านั้น ใช้ `warn!` หรือแม้แต่ `info!` สำหรับ business logic ปกติที่แค่ "ไม่เป็นไปตามที่
ต้องการของผู้ใช้" แต่ระบบยัง handle ได้ตามที่ออกแบบไว้

**(ข) ใช้ `info!` สำหรับทุกอย่างจนรกเกินจะอ่าน** — ถ้าทุกฟังก์ชันมี `info!` สองสามบรรทัด production log จะ
ท่วมด้วยรายละเอียดที่ไม่มีใครต้องการเห็นตอนดูภาพรวมระบบ ทำให้เวลาต้อง debug ปัญหาจริง ต้องไถผ่าน log ที่ไม่
เกี่ยวข้องจำนวนมากกว่าจะเจอบรรทัดที่สำคัญ **วิธีแก้**: ใช้เกณฑ์จากหัวข้อ 60.4 อย่างเคร่งครัด — ถ้าข้อมูลนั้นมี
ประโยชน์แค่ตอน debug เฉพาะเจาะจง ให้เป็น `debug!`/`trace!` (ที่ปิดไว้ตามปกติ เปิดเฉพาะตอนต้องการ) ไม่ใช่
`info!` ที่เปิดอยู่เสมอใน production

### 6. เรียก `tracing_log::LogTracer::init()` ซ้ำทั้งที่ `tracing_subscriber::fmt` มี Bridge ให้อยู่แล้ว

อธิบายพื้นฐานไว้แล้วในหัวข้อ 60.10 — นี่คือกับดักที่พบจริงเมื่อทดลอง: `tracing_subscriber` เปิด feature
`tracing-log` เป็น **default feature** ทำให้ `tracing_subscriber::fmt::init()` จับ `log::` macro record จาก
dependency ได้อัตโนมัติอยู่แล้วโดยไม่ต้องทำอะไรเพิ่ม — ถ้าเผลอไปเรียก `tracing_log::LogTracer::init()` **เอง
ก่อน** `tracing_subscriber::fmt::init()` (เข้าใจว่าต้อง "เปิด bridge" เองตามที่เคยอ่านจาก tutorial เก่า) จะเกิด
การพยายาม set global `log` logger **สองครั้ง** ซึ่ง `log` crate อนุญาตให้ set ได้แค่ครั้งเดียวเท่านั้น — ผลคือ
**panic** ทันที:

```
thread 'main' (6920) panicked at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tracing-subscriber-0.3.23/src/fmt/mod.rs:1263:16:
Unable to install global subscriber: SetLoggerError(())
```

ข้อความ **`SetLoggerError(())`** บอกตรง ๆ ว่ามีความพยายาม set global logger ของ `log` crate เป็นครั้งที่สอง
(ครั้งแรกมาจาก `LogTracer::init()` ที่เรียกเอง ครั้งที่สองมาจาก `tracing_subscriber::fmt::init()` ที่พยายาม
ติดตั้ง bridge ของตัวเองซ้ำอีกที เพราะ default feature `tracing-log` ทำให้มันพยายามทำสิ่งเดียวกันอยู่แล้ว)
**วิธีแก้**: ถ้าใช้ `tracing_subscriber::fmt::init()` (หรือ `fmt().init()` แบบ builder) เฉย ๆ **ไม่ต้องเรียก
`tracing_log::LogTracer::init()` เองเลย** เพราะ bridge ถูกติดตั้งให้อัตโนมัติอยู่แล้ว — `tracing_log::LogTracer`
มีประโยชน์จริงเฉพาะกรณีที่คุณใช้ subscriber แบบ custom ที่**ไม่ได้ผ่าน** `tracing_subscriber::fmt` (เช่นเขียน
`Subscriber` ของตัวเอง หรือปิด default feature `tracing-log` ไปแล้วด้วยเหตุผลเฉพาะทาง) — ก่อนเพิ่มโค้ด
bridge เองเสมอ ให้เช็คก่อนว่า backend ที่ใช้อยู่มี bridge ติดมาให้แล้วหรือยัง (ในกรณีของ `tracing_subscriber::fmt`
คือ**มีให้แล้วเสมอ**ตราบใดที่ไม่ได้ปิด default feature)

### 7. `#[instrument]` แนบทุก Parameter เป็น Field โดยอัตโนมัติ — รวมถึงข้อมูลลับด้วย!

นี่คือกับดักด้าน**ความปลอดภัย**ที่อันตรายมาก และเกิดง่ายมากถ้าไม่รู้ล่วงหน้า — `#[tracing::instrument]` (หัวข้อ
60.7) แนบ **ทุก parameter ของฟังก์ชันที่ implement `Debug`** เป็น field ของ span โดยอัตโนมัติ **โดยไม่แยกแยะเลย
ว่า parameter ตัวไหนเป็นข้อมูลอ่อนไหว**:

```rust
use tracing::instrument;

// อันตราย: #[instrument] แนบทุก parameter เป็น field โดยอัตโนมัติ รวมถึง `password` ด้วย!
#[instrument]
fn login(username: &str, password: &str) -> bool {
    tracing::info!("ตรวจสอบรหัสผ่าน");
    password == "correct-password"
}

#[instrument(skip(password))]
fn login_safe(username: &str, password: &str) -> bool {
    tracing::info!("ตรวจสอบรหัสผ่าน");
    password == "correct-password"
}

fn main() {
    tracing_subscriber::fmt::init();
    login("alice", "super-secret-123");
    login_safe("alice", "super-secret-123");
}
```

รันจริงด้วย `RUST_LOG=info` (ตัด ANSI ออก) — เทียบสองบรรทัดนี้ให้ดี:

```
2026-09-26T23:19:08.330460Z  INFO login{username="alice" password="super-secret-123"}: instrument_leak_demo: ตรวจสอบรหัสผ่าน
2026-09-26T23:19:08.330542Z  INFO login_safe{username="alice"}: instrument_leak_demo: ตรวจสอบรหัสผ่าน
```

บรรทัดแรก **`password="super-secret-123"` รั่วไปอยู่ใน log เต็ม ๆ** ทุกครั้งที่ฟังก์ชัน `login` ถูกเรียก — ถ้า
ระบบ log aggregation ปลายทางเก็บ log พวกนี้ไว้ (ซึ่งเป็นเรื่องปกติของระบบ production) รหัสผ่านของผู้ใช้จริงจะ
ถูกเก็บไว้เป็น plain text ในระบบ log ที่อาจมีคนเข้าถึงได้มากกว่าระบบ database จริงเสียอีก — เป็นช่องโหว่ความ
ปลอดภัยที่ร้ายแรงมาก และสามารถเกิดขึ้นได้ง่าย ๆ กับ parameter อื่นที่อ่อนไหวเช่นกัน (API token, credit card
number, personal identifiable information อื่น ๆ)

**วิธีแก้**: ใช้ **`skip(...)`** ใน `#[instrument(skip(password))]` (เห็นในบรรทัดที่สอง `login_safe`) เพื่อ
บอกให้ macro **ไม่**แนบ parameter ตัวนั้นเป็น field เลย — ผลคือ span ของ `login_safe` มีแค่ `username` ไม่มี
`password` ติดไปด้วยแม้แต่นิดเดียว **กฎปฏิบัติที่ควรยึดถือเสมอ**: ทุกครั้งที่ใส่ `#[instrument]` บนฟังก์ชันที่
รับ parameter ที่เป็นข้อมูลอ่อนไหว (password, token, secret, ข้อมูลส่วนบุคคล) ให้ `skip(...)` พารามิเตอร์นั้น
**เสมอ** โดยไม่มีข้อยกเว้น — ถ้าฟังก์ชันมี parameter หลายตัวที่ไม่อยาก log เลย ใช้ `skip_all` แล้วเลือกเฉพาะ
field ที่ต้องการผ่าน `fields(...)` แทน (ปลอดภัยกว่าเพราะเป็นแบบ "allowlist" ไม่ใช่ "denylist" ที่เสี่ยงลืม
parameter ใหม่ที่เพิ่มเข้ามาทีหลัง)

## แบบฝึกหัด (Exercises)

1. **(ง่าย) เพิ่ม `log`+`env_logger` เข้าโปรแกรมที่มีอยู่**: หยิบโปรแกรมเล็ก ๆ ที่คุณเคยเขียนในบทก่อน ๆ (เช่น
   ระบบจัดการงาน to-do หรือเครื่องคิดเลขจาก Part ต้น ๆ) มาเพิ่ม `cargo add log env_logger` แล้วแทนที่
   `println!`/`eprintln!` ที่ใช้บอกสถานะทั้งหมดด้วย log macro ที่ level เหมาะสม (ใช้เกณฑ์จากหัวข้อ 60.4)
   ทดสอบรันด้วย `RUST_LOG` สามค่าต่างกัน (`error`, `info`, `debug`) แล้วบันทึกว่า output ต่างกันอย่างไรในแต่
   ละครั้ง
   *(hint: อย่าลืม `env_logger::init()` เป็นบรรทัดแรกสุดใน `main()` — ถ้า log ไม่ออกมาเลย นี่คือสิ่งแรกที่
   ต้องเช็คตามกับดักที่ 1)*

2. **(กลาง) ย้ายจาก `log` ไปเป็น `tracing` พร้อม Structured Field**: เขียนฟังก์ชันจำลอง "ระบบยืนยันตัวตน"
   (`fn login(username: &str, password_correct: bool) -> Result<u32, String>`) ที่คืน user_id ถ้า
   `password_correct` เป็นจริง หรือ `Err` ถ้าไม่ใช่ — เขียนสองเวอร์ชัน เวอร์ชันแรกใช้ `log` ธรรมดาแบบ format
   string ล้วน ๆ เวอร์ชันที่สองใช้ `tracing` กับ structured field (`user_id`, `username`, `success`) เปรียบเทียบ
   output ของทั้งสองเวอร์ชันตอน `RUST_LOG=info` แล้วอธิบายว่าทำไม output จากเวอร์ชัน `tracing` ถึง "ค้นหาได้ง่าย
   กว่า" ถ้าต้อง grep หา log ทั้งหมดของ user_id หนึ่งตัวโดยเฉพาะ
   *(hint: ลองรันด้วย `tracing_subscriber::fmt().json().init()` แทน `fmt::init()` ธรรมดา แล้วดูว่า field
   ปรากฏเป็น JSON key ที่ query ตรงได้อย่างไร ตามตัวอย่างหัวข้อ 60.8)*

3. **(ยาก) `#[instrument]` กับ Concurrent Task พิสูจน์การ Attribute ที่ถูกต้อง**: เขียนโปรแกรม async ที่จำลอง
   "background job processor" — spawn 4 task พร้อมกันผ่าน `tokio::spawn` แต่ละ task ประมวลผล "งาน" หนึ่งชิ้น
   (มี job_id ต่างกัน) ผ่าน 3 ขั้นตอนย่อย (แต่ละขั้นตอนเป็นฟังก์ชัน async ของตัวเองที่มี `tokio::time::sleep`
   จำลอง latency ต่างกันในแต่ละ task) ใส่ `#[instrument(fields(job_id = job_id))]` ครอบทั้งฟังก์ชันระดับบนสุด
   และฟังก์ชันขั้นตอนย่อยทุกตัว รันด้วย `RUST_LOG=debug` แล้วพิสูจน์ (เขียนคำอธิบายกำกับ output ที่ได้) ว่า
   ทุกบรรทัด log สามารถระบุได้ถูกต้อง 100% ว่าเป็นของ job ไหน แม้ execution จะสลับกันไปมาจริงเพราะ latency
   ต่างกัน
   *(hint: ใช้โครงสร้างเดียวกับตัวอย่างหลักในหัวข้อ 60.7/60.11 เป็นต้นแบบ แค่เปลี่ยนจาก "request" เป็น "job"
   และเพิ่มขั้นตอนจาก 2 เป็น 3 ขั้น)*

4. **(ยากมาก / ประยุกต์ใช้งานจริง) ระบบจำลองเต็มรูปแบบพร้อม JSON Output และ Log-to-Tracing Bridge**: สร้าง
   โปรแกรมจำลอง "API Gateway" เล็ก ๆ ที่รับ 5 request พร้อมกัน (`tokio::spawn`) แต่ละ request ผ่าน 3 ชั้น:
   (ก) validate input (instrumented, structured field บอกผลว่า valid/invalid), (ข) เรียก "downstream service"
   จำลอง (instrumented, มี field ของ latency และ status code จำลอง) และ (ค) log สรุปผลลัพธ์สุดท้าย — เขียน
   ฟังก์ชันเสริมหนึ่งตัวที่**ใช้ `log::` macro ธรรมดาเท่านั้น** (ไม่ใช้ `tracing` เลย จำลองว่าเป็น dependency
   crate ภายนอก) ให้ทำงานอยู่ในทุก request ด้วย ตั้ง subscriber เป็น `tracing_subscriber::fmt().json().init()`
   แล้วยืนยันว่า log จากทั้งฟังก์ชันที่ใช้ `tracing` (มี span context ครบ) และฟังก์ชันที่ใช้ `log` ธรรมดา (ไม่มี
   span context เพราะไม่ได้ instrument) ทั้งคู่ปรากฏใน JSON output stream เดียวกันได้ อธิบายในคำตอบว่าทำไม
   ฟังก์ชันที่ใช้ `log` ธรรมดาถึงไม่มี field `spans` ติดมาด้วยทั้งที่มันถูกเรียกจากภายใน span ของ request นั้น ๆ
   *(hint: คำตอบของคำถามสุดท้ายเกี่ยวกับสาเหตุที่ `log` record ไม่มี span context อยู่ในเนื้อหาหัวข้อ 60.5 —
   `log` ไม่มีแนวคิดเรื่อง "current span" อยู่ในโมเดลของมันเลยตั้งแต่ต้น การที่ tracing-log bridge ทำได้แค่
   "แปลง record เป็น event ของ tracing" แต่ไม่สามารถguess span context ที่ log call เดิมไม่ได้ตั้งใจส่งมาให้)*

## สรุป

บทนี้พาคุณจากข้อจำกัดของ `println!`/`eprintln!` ที่ใช้มาตั้งแต่ Part 1 ไปสู่ระบบ logging/observability ที่ใช้
งานได้จริงในระดับ production — `log` crate สอนแนวคิด **facade pattern** (เชื่อมกับ Part 57) ที่แยกอินเทอร์เฟซ
การเรียกออกจาก backend ที่เลือกได้อิสระ ควบคุมได้เต็มที่ผ่าน `RUST_LOG` โดยไม่ต้อง compile ใหม่ (ผ่าน
`env_logger`) — จากนั้นบทนี้ชี้ให้เห็นข้อจำกัดที่ `log` มีโดยกำเนิดในโลก async (เชื่อมกับ Part 46-50 อย่างเข้มข้น):
เมื่อหลาย task สลับกันทำงานบน OS thread เดียวตามกลไก cooperative multitasking log ธรรมดาไม่มีทางบอกได้ว่าบรรทัด
ไหนเป็นของ task ไหน — พิสูจน์ด้วย output จริงที่มีบรรทัดซ้ำกันเป๊ะสามครั้งจากสาม request ที่ไม่มีทางแยกแยะได้เลย
`tracing` crate แก้ปัญหานี้ด้วย **span** ที่ `#[tracing::instrument]` (attribute macro เชื่อมกับ Part 44) สร้าง
ให้ครอบทั้งฟังก์ชันข้าม `.await` โดยอัตโนมัติ พิสูจน์ด้วย output จริงที่ทุกบรรทัดมี `request_id` นำหน้าถูกต้อง
100% แม้ execution จะสลับกันไปมาจริง — ปิดท้ายด้วย **structured field** ที่ทำให้ log เป็นข้อมูลที่ query ได้
(ไม่ใช่แค่ข้อความ) ซึ่งเป็นวัตถุดิบตั้งต้นของระบบ observability ที่ Part 98-99 จะขยายต่อ และ compatibility
shim ที่ทำให้เลือก `tracing` สำหรับแอปได้โดยไม่ทิ้ง dependency ที่ใช้ `log` ไปข้างหลัง

บทนี้คือ**บทปิดของโมดูล 3 (ระดับสูง / Advanced, Part 41-60)** — ควรมองย้อนดูเส้นทางทั้งหมดของโมดูลนี้ก่อนไปต่อ
โมดูล 4:

- **Part 41-45 (Unsafe/FFI/Macros)**: เปิดโมดูลด้วยการดึงม่านของสิ่งที่ Rust "ป้องกัน" ให้เห็นเบื้องหลังจริง —
  `unsafe` เบื้องต้น, raw pointer และ memory layout, การเชื่อมกับ C ผ่าน FFI, และ procedural macro ทั้งแบบ
  attribute และ derive ที่ทำให้เข้าใจว่าโค้ดที่ดูเหมือน "มนตร์ดำ" (เช่น `#[tokio::main]`, `#[derive(Debug)]`,
  และ `#[instrument]` ในบทนี้เอง) จริง ๆ แล้วเป็นแค่การ generate โค้ดที่เขียนเองได้อยู่แล้ว
- **Part 46-50 (Async/Tokio)**: จากความเข้าใจเรื่อง `Future`/`Poll`/`Waker` ไปสู่การใช้งาน Tokio runtime จริง —
  `tokio::spawn`, I/O/networking, และ synchronization แบบ async — รากฐานที่บทนี้ (โดยเฉพาะปัญหา log
  attribution ในหัวข้อ 60.5-60.7) พึ่งพาอย่างเข้มข้นที่สุดในบรรดา Part ทั้งหมดของโมดูล 3
- **Part 51-53 (Atomics/Design Patterns)**: ขยายความเข้าใจ concurrency ไปถึงระดับ lock-free และวางกรอบการ
  ออกแบบโค้ดที่ maintain ได้ในระยะยาวด้วย pattern ที่พิสูจน์แล้วว่าใช้ได้จริงในภาษา Rust
- **Part 54-55 (Performance/Profiling)**: สอนวิธี**วัด**ก่อนที่จะปรับให้เร็วขึ้น — ทักษะเดียวกันนี้คือสิ่งที่
  ใช้ตรวจสอบว่าการเพิ่ม logging/tracing เข้าไปในระบบจริงไม่ได้สร้าง overhead ที่กระทบ performance เกินจำเป็น
- **Part 56-59 (Memory/Serde/CLI)**: ปิดท้ายด้วยทักษะที่ทุกแอปพลิเคชันจริงต้องมี — serialization (Part 57-58)
  ที่ผูก analogy โดยตรงกับ facade pattern ของ `log` ในบทนี้ และการสร้าง CLI application (Part 59) ที่บทนี้
  โยงกลับไปด้วยรูปแบบ `-v`/`-vv`/`-vvv` ควบคุม verbosity
- **Part 60 (บทนี้)**: ปิดโมดูลด้วยเครื่องมือที่ทำให้ทุกอย่างที่เรียนมา**สังเกตได้จริงตอนรัน** — เพราะไม่ว่าโค้ด
  จะถูกต้องแค่ไหนตาม type system ของ Rust หรือเร็วแค่ไหนตาม benchmark ที่ Part 54 สอน ถ้าไม่มีทางรู้ว่าระบบจริง
  ทำอะไรอยู่ตอนรันบน production ก็ไม่มีทาง debug ปัญหาที่เกิดขึ้นจริงได้เลย

**โมดูล 4 (การพัฒนาเว็บแอปพลิเคชัน / Web Development, Part 61-85)** เริ่มต้นที่ **Part 61 (HTTP Fundamentals
และ REST API Concepts)** — จะปรับพื้นฐานเรื่อง HTTP protocol และหลักการออกแบบ REST API ก่อนเข้าสู่ **Part 62
(แนะนำ Axum Framework)** ที่จะสร้าง web server จริงตัวแรกของหลักสูตร โดยตรงบน Tokio runtime ที่ Part 48-50 สอน
ไว้แล้ว — และที่สำคัญที่สุดสำหรับเนื้อหาบทนี้: **web server ทุกตัวที่จะสร้างตลอดโมดูล 4 ต้องมี logging/tracing
ที่ดีเพื่อ debug ปัญหาการรับ concurrent request จำนวนมากในระดับ production** — `tracing` และ `#[instrument]`
ที่เรียนในบทนี้จะกลับมาเป็นเครื่องมือหลักที่ใช้ตลอดทั้งโมดูล 4 โดยเฉพาะตอน middleware (Part 65) ที่ต้อง log
ทุก HTTP request ที่เข้ามาพร้อม request ID ที่ไม่ปนกันแม้ระบบรับหลายพัน request พร้อมกันจริง

---

**Part ก่อนหน้า:** [CLI Applications ด้วย clap](part-059-clap-cli.md) | **Part ถัดไป:** [HTTP Fundamentals และ REST API Concepts](part-061-http-fundamentals.md)
