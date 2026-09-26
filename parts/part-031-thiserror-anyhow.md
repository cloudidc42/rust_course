# Part 31: thiserror และ anyhow ในโปรเจกต์จริง

> โมดูล: ระดับกลาง (Intermediate) | ระดับ: กลาง | เวลาโดยประมาณ: 180 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- เพิ่ม crate `thiserror` เข้าโปรเจกต์ด้วย `cargo add thiserror` และใช้ `#[derive(thiserror::Error)]` แทนการเขียน
  `impl Display`/`impl std::error::Error` ด้วยมือทั้งหมดตามที่ทำมาใน Part 30 ได้อย่างถูกต้อง พร้อมอธิบายได้ว่า
  attribute แต่ละตัว (`#[error("...")]`, `#[from]`, `#[source]`) **แทนที่โค้ดส่วนไหน**ที่เคยเขียนด้วยมือ ไม่ใช่แค่
  ท่องจำ syntax โดยไม่รู้ว่ามันทำงานอย่างไร
- ใช้ field interpolation ใน `#[error("...")]` ได้ทั้งแบบ positional (`{0}`) และแบบอ้างชื่อ field (`{field}`,
  `{source}`) และรู้ว่า `#[from]` สร้าง `impl From<T>` ให้อัตโนมัติพร้อม**บอกอัตโนมัติว่ามัน implies `#[source]`
  ด้วยเสมอ** ส่วน `#[error(transparent)]` ทำให้ variant หนึ่ง "ล่องหน" ในทั้ง `Display` และ error chain โดยส่งต่อ
  ทั้งสองอย่างไปยัง error ที่ห่อไว้ตรงๆ
- อธิบายความแตกต่างเชิงปรัชญาระหว่าง `thiserror` (สำหรับ **library** ที่ผู้เรียกต้องแยกกรณี error) กับ `anyhow`
  (สำหรับ **application/binary** ที่แค่ต้อง propagate และแสดง error ให้อ่านง่าย) ได้อย่างชัดเจน และรู้ว่าทำไม
  โปรเจกต์จริงจำนวนมากใช้**ทั้งสอง crate พร้อมกัน**ใน workspace เดียว (thiserror ใน library crate, anyhow ใน
  binary crate)
- ใช้ `anyhow::Result<T>`, มาโคร `anyhow!()`/`bail!()`/`ensure!()`, และ method `.context()`/`.with_context()`
  เพื่อแนบข้อความอธิบายให้กับ error ระหว่างที่มันเดินทางขึ้นมาตาม call stack ได้อย่างคล่องแคล่ว พร้อมพิมพ์
  error chain แบบเต็ม ("Caused by:") ด้วยการ format แค่ `{:?}` โดยไม่ต้องเขียน loop ไล่ `.source()` เองเหมือนที่
  ทำใน Part 30
- Downcast `anyhow::Error` กลับไปเป็น concrete error type ที่รู้จักด้วย `.downcast::<T>()` ในสถานการณ์ที่แม้จะใช้
  `anyhow` เป็นหลักแต่ยังต้องการ "กู้คืน" หรือตอบสนองต่างกันไปตาม error บางชนิดที่เจาะจงเป็นกรณีพิเศษ
- ตัดสินใจได้อย่างมีหลักการว่าโค้ดส่วนที่กำลังเขียนควรใช้ `thiserror`, `anyhow`, หรือ hand-rolled enum แบบ Part 30
  โดยพิจารณาจาก "ใครคือผู้เรียกโค้ดนี้ และเขาต้องแยกกรณี error หรือไม่"

## ความรู้ที่ต้องมีมาก่อน

- **Part 30 (Error Handling ขั้นสูง)**: นี่คือ**ภาคต่อโดยตรง** ของ Part 30 และเป็นความรู้ที่ต้องมีมาก่อนที่สำคัญที่สุด
  ของบทนี้ — บทนี้จะไม่สอนแนวคิดเรื่อง `std::error::Error`, `Display`, `source()`, หรือ `From`/`Into` ใหม่ตั้งแต่
  ต้นอีก เพราะ Part 30 สอนไว้ครบและลึกแล้ว สิ่งที่บทนี้ทำคือ **เอาโค้ดที่เขียนด้วยมือทั้งหมดใน Part 30** (โดยเฉพาะ
  `ConfigError`/`AppError` เวอร์ชันสมบูรณ์จากหัวข้อ 30.10 ที่มีทั้ง `impl Display`, `impl Error` พร้อม `source()`,
  และ `impl From` หลายตัว) **มาเขียนใหม่ด้วย `thiserror`** เพื่อให้เห็นบรรทัดต่อบรรทัดว่า attribute ไหนแทนที่โค้ด
  ส่วนไหน — ถ้าคุณยังไม่แน่นเรื่อง `source()`/error chain/`#[non_exhaustive]` จาก Part 30 ควรทวนก่อนเริ่มบทนี้
  เพราะกับดักและคำอธิบายในบทนี้จะอ้างอิงกลับไปที่ Part 30 ตลอดเวลาโดยไม่อธิบายซ้ำ
- **Part 12 (Result<T,E> และ Error Handling เบื้องต้น)**: บทนี้จะเทียบ `anyhow::Error` กับ `Box<dyn
  std::error::Error>` ที่ Part 12 หัวข้อ 12.7-12.8 แนะนำไว้เป็นทางลัด pragmatic สำหรับ script/`main()` — คุณต้อง
  จำได้ว่า `Box<dyn Error>` คืออะไรและทำไม Part 30 ถึงบอกว่ามันไม่เหมาะกับ library (อ่านหัวข้อ 30.1 ทวนได้)
  เพราะ `anyhow::Error` คือ "`Box<dyn Error>` เวอร์ชันที่ ergonomic กว่ามาก" ไม่ใช่แนวคิดใหม่ทั้งหมด
- **Part 17 (Packages, Crates, Workspaces)**: การเพิ่ม `thiserror`/`anyhow` เข้าโปรเจกต์ใช้กลไก dependency
  เดียวกันทุกประการกับที่ Part 17 สอนไว้ (`[dependencies]` ใน `Cargo.toml`, `cargo add`) — และหัวข้อท้ายบทนี้ที่พูด
  ถึง "library crate ใช้ thiserror, binary crate ใช้ anyhow ในคนละ crate ของ workspace เดียวกัน" ใช้แนวคิด
  **workspace** และ **path dependency** ที่ Part 17 หัวข้อ 17.5-17.7 สอนไว้ตรงๆ ถ้ายังไม่คุ้นกับโครงสร้าง
  `[workspace]` + `members` + `path = "../core"` ควรทวน Part 17 ก่อน
- **Part 2 (Cargo และโครงสร้างโปรเจกต์)**: คำสั่ง `cargo add` พื้นฐาน (ไม่ระบุ version, `--dry-run`) มาจาก Part 2
  หัวข้อ 2.7 — บทนี้จะใช้คำสั่งนี้ตรงๆ โดยไม่อธิบาย flag ซ้ำ

## เนื้อหา

### 31.1 ทวนความจำโดยย่อ: สิ่งที่ Part 30 สร้างด้วยมือทั้งหมด

ก่อนไปดู `thiserror`/`anyhow` มาทวนสั้นๆ ว่า Part 30 ลงทุนเขียนอะไรด้วยมือไปแล้วบ้าง สำหรับ `ConfigError`/
`AppError` เวอร์ชันสมบูรณ์ที่สุด (หัวข้อ 30.10):

1. `#[derive(Debug)] enum ConfigError { ... }` — นิยาม enum เอง (ต้องทำเหมือนกันไม่ว่าจะใช้ crate ไหนก็ตาม)
2. `impl fmt::Display for ConfigError { fn fmt(...) { match self { ... } } }` — เขียนข้อความสำหรับแต่ละ variant
   ด้วยมือ ผ่าน `match` ที่ต้อง exhaustive
3. `impl Error for ConfigError { fn source(&self) -> ... { match self { ... } } }` — เขียน `source()` ด้วยมือ
   อีกรอบ ผ่าน `match` ที่ต้องดูแลให้ตรงกับ `Display` (variant ไหนมี error ต้นตอ variant ไหนไม่มี)
4. `impl From<io::Error> for AppError { ... }` และ `impl From<ConfigError> for AppError { ... }` — เขียน
   `impl From` แยกกันทีละแหล่งที่มาของ error ที่ `AppError` ห่อไว้
5. ทำข้อ 1-4 **ซ้ำอีกรอบ**สำหรับ `AppError` ที่ห่อ `ConfigError` ไว้อีกชั้น

รวมแล้วสำหรับสอง enum (`ConfigError` 3 variant + `AppError` 2 variant) โค้ดส่วน `Display`/`Error`/`From` ล้วนๆ
(ไม่รวม struct `AppConfig`, ไม่รวมฟังก์ชัน parse) ยาวประมาณ **68 บรรทัด** (นับจากหัวข้อ 30.10 ตรงๆ) — และนี่คือ
สิ่งที่**ต้องทำซ้ำทุกครั้ง**ที่สร้าง error enum ใหม่ในโปรเจกต์ ไม่ว่า enum นั้นจะมีกี่ variant หรือซับซ้อนแค่ไหน

**สิ่งที่ Part 30 ทิ้งท้ายไว้ตรงๆ ในหัวข้อสุดท้าย (30.10 และบทสรุป)**: โค้ดทั้ง 68 บรรทัดนี้ **ไม่ได้มีตรรกะที่
แตกต่างกันในแต่ละโปรเจกต์เลย** — มันคือ "สูตรสำเร็จ" (boilerplate) ที่ทำตามรูปแบบเดียวกันซ้ำๆ ทุกครั้ง: "แต่ละ
variant มีข้อความของตัวเอง", "variant ที่ห่อ error ไว้ต้องคืน `Some` จาก `source()`", "แต่ละแหล่งที่มาต้องมี
`impl From`" — สิ่งที่เปลี่ยนไปในแต่ละโปรเจกต์จริงๆ มีแค่ **ข้อความ** ในแต่ละ variant เท่านั้น ส่วนโครงสร้างที่
เหลือทั้งหมดเหมือนกันเป๊ะทุกครั้ง — นี่คือสัญญาณชัดเจนว่างานแบบนี้เหมาะกับการให้ **macro** (โค้ดที่ generate โค้ด
ให้ตอน compile time) ทำแทน และนั่นคือสิ่งที่ `thiserror` ทำ

บทนี้จะไม่สอนวิธีคิดใหม่เลยแม้แต่นิดเดียว — **ทุกอย่างที่คุณเข้าใจจาก Part 30 (Display คืออะไร, source() ทำงาน
อย่างไร, From ทำอะไรให้ `?`) ยังใช้ได้เป๊ะทุกประการ** สิ่งที่เปลี่ยนคือ **ใครเป็นคนพิมพ์โค้ดที่ implement มัน** —
จากที่คุณพิมพ์เอง 68 บรรทัด จะเหลือแค่การเขียน attribute สั้นๆ ไม่กี่บรรทัดแล้วให้ macro พิมพ์ที่เหลือให้

### 31.2 เพิ่ม `thiserror` เข้าโปรเจกต์

ทำตามกลไก dependency ปกติที่ Part 2/17 สอนไว้ ใช้คำสั่ง `cargo add`:

```bash
cargo add thiserror
```

คำสั่งนี้จะแก้ `Cargo.toml` ให้อัตโนมัติ (ทดสอบจริงในเครื่องผู้เขียนบทนี้ ได้ผลลัพธ์ดังนี้):

```
    Updating crates.io index
      Adding thiserror v2.0.21 to dependencies
             Features:
             + std
```

และเพิ่มบรรทัดนี้ลงใน `[dependencies]`:

```toml
[dependencies]
thiserror = "2.0.21"
```

**ข้อสังเกตที่สำคัญ**: `thiserror` คือ crate ที่ export **derive macro** (`#[derive(thiserror::Error)]`) เท่านั้น
มันไม่มี runtime type ใหม่ๆ ให้ใช้ (ไม่มี `thiserror::SomeStruct` ให้เรียก) — สิ่งที่มันทำทั้งหมดเกิดขึ้น**ตอน
compile time**: มันอ่าน attribute ที่คุณเขียนไว้บน enum/struct แล้ว **generate โค้ด `impl Display`/`impl
std::error::Error`/`impl From` ให้เอง** ตามรูปแบบเดียวกันกับที่คุณเขียนด้วยมือใน Part 30 ทุกประการ — เพราะเป็นแค่
โค้ดที่ generate ตอน compile time (ไม่มี runtime cost ใดๆ เพิ่มเติมเลย) `thiserror` จึงเป็น dependency ที่ปลอดภัย
มากสำหรับ library แม้แต่ library ที่ระมัดระวังเรื่องขนาด binary หรือ compile time ก็ยังนิยมใช้กันทั่วไป — ต่างจาก
`anyhow` (หัวข้อ 31.8 เป็นต้นไป) ที่มี runtime type จริงๆ (`anyhow::Error`) ให้ใช้งาน

### 31.3 `#[derive(thiserror::Error)]` และ `#[error("...")]`: แทนที่ `impl Display` ทั้งก้อน

มาเริ่มจากเวอร์ชันพื้นฐานที่สุดของ `ConfigError` (จากหัวข้อ 30.3 ของ Part 30) แล้วเขียนใหม่ด้วย `thiserror`:

```rust
use std::path::PathBuf;
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("ไม่พบฟิลด์ที่จำเป็น '{0}' ในไฟล์ config")]
    MissingField(String),

    #[error("ฟิลด์ '{0}' ต้องเป็นตัวเลข แต่ค่าที่ให้มาไม่ใช่ตัวเลขที่ถูกต้อง")]
    InvalidNumber(String),

    #[error("ไม่พบไฟล์ config ที่ path: {0}")]
    FileNotFound(PathBuf),
}

fn main() {
    let errors = vec![
        ConfigError::MissingField("port".to_string()),
        ConfigError::InvalidNumber("timeout".to_string()),
        ConfigError::FileNotFound(PathBuf::from("/etc/myapp/config.toml")),
    ];

    for e in &errors {
        println!("{e}");
    }

    // exhaustiveness checking ยังทำงานเหมือนเดิมทุกประการ เพราะ ConfigError ยังเป็น enum ธรรมดา
    for e in &errors {
        match e {
            ConfigError::MissingField(_) => println!("-> ต้องแจ้งผู้ใช้ให้เติมฟิลด์"),
            ConfigError::InvalidNumber(_) => println!("-> ต้องแจ้งผู้ใช้ให้แก้ตัวเลข"),
            ConfigError::FileNotFound(_) => println!("-> ต้องสร้างไฟล์ default ให้"),
        }
    }
}
```

ผลลัพธ์ (คอมไพล์และรันจริงด้วย `cargo run` ในเครื่องผู้เขียน — เหมือนกับ Part 30 เป๊ะทุกตัวอักษร):

```
ไม่พบฟิลด์ที่จำเป็น 'port' ในไฟล์ config
ฟิลด์ 'timeout' ต้องเป็นตัวเลข แต่ค่าที่ให้มาไม่ใช่ตัวเลขที่ถูกต้อง
ไม่พบไฟล์ config ที่ path: /etc/myapp/config.toml
-> ต้องแจ้งผู้ใช้ให้เติมฟิลด์
-> ต้องแจ้งผู้ใช้ให้แก้ตัวเลข
-> ต้องสร้างไฟล์ default ให้
```

**เปรียบเทียบกับเวอร์ชันมือของ Part 30 หัวข้อ 30.3**: enum เดิมมี 46 บรรทัด (นับ `enum` + `impl Display` +
`impl Error`) ส่วนเวอร์ชันนี้มีแค่ **12 บรรทัด** (นับจาก `#[derive(Debug, Error)]` ถึงปิด `}` ของ enum) — และ
**ผลลัพธ์เหมือนกันทุกตัวอักษร** เพราะ `thiserror` generate โค้ดที่เทียบเท่ากับที่เราเขียนด้วยมือให้เองทั้งหมด

**อธิบายกลไกทีละส่วน:**

- **`#[derive(Debug, Error)]`** — สังเกตว่าเรายัง**ต้องมี `#[derive(Debug)]` เองเหมือนเดิม** — `thiserror` ไม่ได้
  generate `Debug` ให้ (เพราะ `Debug` มี `#[derive]` ของ standard library อยู่แล้ว ไม่มีเหตุผลให้ `thiserror` ทำ
  งานซ้ำ) มันจัดการแค่ครึ่งหลังของ supertrait requirement ที่ Part 30 หัวข้อ 30.2 อธิบายไว้ (`Error: Debug +
  Display`) คือ **`Display`** และตัว **`Error` เอง** (พร้อม `source()`) เท่านั้น — `Error` ที่ import มาจาก
  `thiserror::Error` ในบรรทัด `use thiserror::Error;` คือชื่อของ **derive macro** ไม่ใช่ trait `std::error::Error`
  ตรงๆ (แม้ผลลัพธ์ที่ generate ออกมาจะ `impl std::error::Error` ให้จริงๆ ก็ตาม) — ถ้าโค้ดของคุณต้องใช้ทั้งสองชื่อ
  `Error` ปนกัน (เช่นต้องอ้าง `std::error::Error` ตรงๆ เพื่อรับ `&dyn Error`) ให้เขียน `use thiserror::Error as
  ThisError;` แยกชื่อออกจากกันเพื่อไม่ให้ชนกัน (จะเห็นตัวอย่างการใช้ชื่อแยกแบบนี้ในหัวข้อถัดไป)
- **`#[error("ไม่พบฟิลด์ที่จำเป็น '{0}' ในไฟล์ config")]`** — attribute นี้เขียนอยู่**เหนือแต่ละ variant** (ไม่ใช่
  เหนือ enum ทั้งก้อน) และ**แทนที่ arm หนึ่งของ `match` ใน `impl Display` ที่เราเขียนด้วยมือ** ข้อความในเครื่องหมาย
  คำพูดคือ **format string** แบบเดียวกันกับที่ macro `format!()`/`write!()` ใช้ทุกประการ (ตามหลักการ formatting ที่
  เรียนมาตั้งแต่บทต้นๆ ของหลักสูตร) — `thiserror` จะ generate โค้ดที่เทียบเท่ากับ `write!(f, "ไม่พบฟิลด์ที่จำเป็น
  '{0}' ในไฟล์ config", self.0)` ให้อัตโนมัติ (โดย `self.0` มาจาก field แรกของ tuple variant นี้)
- **`{0}` (positional field interpolation)** — เพราะ `MissingField(String)` เป็น **tuple variant** (ไม่มีชื่อ
  field) `thiserror` จึงต้องอ้าง field ด้วย**ตำแหน่ง** เหมือนกับที่ `format!()` รองรับ argument แบบ positional
  (`{0}`, `{1}`, ...) มาตั้งแต่ Rust ยุคแรก — `{0}` แปลว่า "field ตัวที่ 0 (ตัวแรก) ของ variant นี้" ซึ่งในที่นี้คือ
  `String` ที่เก็บชื่อฟิลด์ไว้
- **exhaustiveness checking ท้ายโค้ดยังทำงานเหมือนเดิม 100%** — จุดนี้สำคัญมากและเป็นหัวใจของทั้งบทนี้: `thiserror`
  **ไม่ได้เปลี่ยนโครงสร้างของ `enum ConfigError` เลยแม้แต่นิดเดียว** มันยังเป็น enum ธรรมดาที่ compiler รู้จักทุก
  variant ครบถ้วน `match` ที่ท้ายโค้ดจึงยังบังคับ exhaustiveness เหมือนที่ Part 30 หัวข้อ 30.3 อธิบายไว้ทุกประการ —
  `thiserror` แค่ลด**งานเขียน** ไม่ได้ลด**ความสามารถ**ของ type ลงเลย

#### Field interpolation แบบอ้างชื่อ (named fields)

ถ้า variant เป็น **struct variant** (มีชื่อ field) สามารถอ้างชื่อ field ตรงๆ ในข้อความได้เลย โดยไม่ต้องนับตำแหน่ง:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("ฟิลด์ '{field}' ต้องเป็น true หรือ false แต่พบค่า '{value}'")]
    InvalidBool { field: String, value: String },
}

fn main() {
    let e = ConfigError::InvalidBool {
        field: "verbose".to_string(),
        value: "yes".to_string(),
    };
    println!("{e}");
}
```

ผลลัพธ์:

```
ฟิลด์ 'verbose' ต้องเป็น true หรือ false แต่พบค่า 'yes'
```

`{field}` และ `{value}` ในที่นี้ทำงานเหมือน **named capture ของ `format!()`** (เทียบกับ `format!("{field}")` ที่
อ่านตัวแปรชื่อ `field` จาก scope ปัจจุบันตรงๆ ตามที่เรียนมาตั้งแต่บทต้นๆ) เพียงแต่ในบริบทของ `#[error(...)]`
`thiserror` จะ generate โค้ดที่อ่านค่าจาก **field ของ `self`** (เช่น `self.field`, `self.value`) แทนตัวแปร local
— นี่คือเหตุผลที่ **struct variant อ่านง่ายกว่า tuple variant มากเมื่อมีหลาย field**: `{0}`, `{1}` บอกแค่ตำแหน่ง
ทำให้ต้องนับเลขตามลำดับที่ประกาศไว้ (เสี่ยงผิดถ้า field เยอะ) ส่วน `{field}`/`{value}` บอกความหมายตรงๆ

### 31.4 `#[from]`: แทนที่ `impl From<T>` ทั้งบล็อก

จาก Part 30 หัวข้อ 30.4/30.6 เราเขียน `impl From<ParseIntError> for ConfigError` และ `impl From<io::Error> for
AppError` ด้วยมือ เพื่อให้ `?` แปลง error ข้ามชนิดกันได้อัตโนมัติ — `thiserror` มี attribute `#[from]` ที่ทำสิ่งนี้
ให้แทน โดยเขียนไว้บน **field** ของ variant (ไม่ใช่บน variant เอง):

```rust
use thiserror::Error as ThisError;

#[derive(Debug, ThisError)]
enum ConfigError {
    #[error("io error เกิดขึ้น: {0}")]
    Io(#[from] std::io::Error),
}

// ทดสอบว่า ? แปลง io::Error เป็น ConfigError ให้อัตโนมัติจริง โดยไม่มี impl From ที่เราเขียนเองเลย
fn read_something() -> Result<String, ConfigError> {
    let content = std::fs::read_to_string("/nonexistent/path/xyz.conf")?;
    Ok(content)
}

fn main() {
    match read_something() {
        Ok(_) => println!("อ่านสำเร็จ"),
        Err(e) => println!("error: {e}"),
    }
}
```

ผลลัพธ์ (ทดสอบจริงในเครื่องที่ไม่มีไฟล์นี้อยู่ ตามที่คาดไว้):

```
error: io error เกิดขึ้น: No such file or directory (os error 2)
```

**อธิบายกลไก:**

- `Io(#[from] std::io::Error)` — attribute `#[from]` เขียนไว้**ข้างหน้า field** ตรงๆ (ในที่นี้ field ไม่มีชื่อ
  เพราะเป็น tuple variant) — `thiserror` จะ generate โค้ดที่เทียบเท่ากับ `impl From<std::io::Error> for
  ConfigError { fn from(e: std::io::Error) -> Self { ConfigError::Io(e) } }` ให้อัตโนมัติทุกประการ ตรงกับที่ Part
  30 หัวข้อ 30.6 เขียนด้วยมือเป๊ะๆ
- `std::fs::read_to_string(...)?` — เพราะมี `impl From<io::Error> for ConfigError` (ที่ `#[from]` generate ให้)
  `?` จึงแปลง `io::Error` เป็น `ConfigError::Io(...)` ให้อัตโนมัติทันที เหมือนกับที่หัวข้อ 30.6 อธิบายไว้เรื่องการ
  ห่อ error หลายแหล่งไว้ในที่เดียว
- **ข้อจำกัดที่สำคัญ**: **แต่ละ enum มีได้แค่ variant เดียวที่ `#[from]` field ชนิดเดียวกัน** — ถ้าคุณลองเขียน
  สอง variant ที่ทั้งคู่มี `#[from] std::io::Error` เหมือนกัน จะเกิด conflict ทันที (จะเห็น error จริงในหัวข้อ
  กับดักที่พบบ่อยข้อ 1) เหตุผลตรงไปตรงมา: `impl From<io::Error> for ConfigError` เขียนซ้ำสองรอบไม่ได้ ตามหลัก
  ที่ Rust ห้าม implement trait ตัวเดียวกันสองครั้งให้ type เดียวกัน (ปัญหาเดียวกันจะเกิดถ้าคุณลองเขียน
  `impl From<io::Error> for ConfigError` ด้วยมือสองครั้งเช่นกัน — `#[from]` ไม่ได้เพิ่มข้อจำกัดใหม่ มันแค่ทำให้
  เจอปัญหาเดิมได้เร็วขึ้นเพราะเขียนสั้นลง)

### 31.5 `#[source]`: แทนที่ `source()` โดยไม่ต้อง `impl From` เสมอไป

หัวข้อก่อนใช้ `#[from]` ซึ่ง **implies `#[source]` ให้อัตโนมัติเสมอ** (แปลว่าถ้าคุณใส่ `#[from]` ไว้ field นั้นจะ
ถูกคืนออกไปจาก `source()` ให้ด้วยโดยไม่ต้องเขียนอะไรเพิ่ม) แต่บางสถานการณ์คุณ **ต้องการแค่ error chain (ให้
`source()` ชี้ไปที่ field นั้น) โดยไม่ต้องการให้ `?` แปลงชนิดอัตโนมัติ** — เช่นตอนที่ error type ต้นทาง
(`ParseIntError`) ถูกใช้ใน**หลาย variant พร้อม field เพิ่มเติมอื่นด้วย** (เช่น `field: String` บอกว่ากำลัง parse
ฟิลด์ไหนอยู่ ตามข้อจำกัดที่ Part 30 หัวข้อ 30.4 ทิ้งท้ายไว้ว่า `From::from` รับ parameter เดียว รับ context เพิ่ม
ไม่ได้) — กรณีนี้ต้องสร้าง variant ด้วย `.map_err()` เอง (ไม่ใช้ `?` ตรงๆ) แต่ยังอยากให้ `source()` ทำงานถูกต้อง:

```rust
use std::error::Error;
use std::num::ParseIntError;
use thiserror::Error as ThisError;

#[derive(Debug, ThisError)]
enum ConfigError {
    #[error("ไม่พบฟิลด์ '{0}'")]
    MissingField(String),

    #[error("ฟิลด์ '{field}' ไม่ใช่ตัวเลขที่ถูกต้อง")]
    InvalidNumber {
        field: String,
        #[source]
        source: ParseIntError,
    },
}

fn parse_field(field: &str, raw: &str) -> Result<u32, ConfigError> {
    // .map_err() ใส่ field เข้าไปด้วย เพราะ ? เพียวๆ ทำไม่ได้ (From::from รับพารามิเตอร์เดียว)
    raw.parse::<u32>()
        .map_err(|source| ConfigError::InvalidNumber { field: field.to_string(), source })
}

fn print_error_chain(err: &dyn Error) {
    println!("error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        println!("caused by: {cause}");
        source = cause.source();
    }
}

fn main() {
    match parse_field("timeout_seconds", "abc") {
        Ok(t) => println!("timeout = {t}"),
        Err(e) => print_error_chain(&e),
    }
}
```

ผลลัพธ์:

```
error: ฟิลด์ 'timeout_seconds' ไม่ใช่ตัวเลขที่ถูกต้อง
caused by: invalid digit found in string
```

**อธิบายจุดสำคัญ:**

- `#[source] source: ParseIntError` — attribute นี้บอก `thiserror` ให้ generate `source()` แบบที่คืน
  `Some(&self.source)` เมื่อ `match` ไปเจอ variant นี้ (เหมือนกับ `fn source(&self) -> ... { match self {
  ConfigError::InvalidNumber { source, .. } => Some(source), ... } }` ที่ Part 30 หัวข้อ 30.5 เขียนด้วยมือเป๊ะๆ)
  — สังเกตว่า**ไม่มี `#[from]` ที่นี่** จึงไม่มี `impl From<ParseIntError> for ConfigError` ถูก generate ให้ —
  ถ้าคุณลองเขียน `raw.parse::<u32>()?` ตรงๆ ในฟังก์ชันนี้โดยไม่ใช้ `.map_err()` จะเจอ error `E0277` แบบเดียวกับที่
  Part 30 หัวข้อ 30.4 อธิบายไว้ทุกประการ (เพราะไม่มี `From<ParseIntError> for ConfigError` ให้ `?` เรียก) — นี่คือ
  ข้อแตกต่างที่ชัดเจนระหว่างสองattribute: **`#[from]` = สร้างทั้ง `From` และ `source()`, `#[source]` = สร้างแค่
  `source()` อย่างเดียว**
- **ธรรมเนียมการตั้งชื่อ field ว่า `source`** — ที่ Part 30 หัวข้อ 30.5 กล่าวไว้ว่าเป็น "ธรรมเนียมที่นิยม" ตอนนี้
  มามีความหมายจริงจังขึ้น: **ถ้า field ชื่อ `source` เป๊ะ `thiserror` จะถือว่ามันคือ error ต้นตอโดยอัตโนมัติแม้ไม่
  เขียน `#[source]` ก็ตาม** (naming convention ที่ผูกกับพฤติกรรมจริงของ macro) แต่การเขียน `#[source]` ชัดๆ ก็ยัง
  ทำงานเหมือนกัน ไม่ขึ้นกับชื่อ field เลย — บทนี้จะเขียน `#[source]` ให้เห็นชัดเจนเสมอเพื่อไม่ให้อาศัย "เดา" จากชื่อ
  ตัวแปรเพียงอย่างเดียว

### 31.6 `#[error(transparent)]`: ทำให้ Variant "ล่องหน" ในทั้ง Display และ Chain

มี pattern พิเศษอีกแบบที่ `#[error("...")]` ปกติทำไม่ได้ตรงๆ: บางครั้ง variant หนึ่งมีหน้าที่แค่ "ห่อ" error
type อื่นไว้เพื่อรวมชนิดให้ตรงกัน (ให้ `?` ใช้งานได้) **โดยไม่ต้องการเพิ่มข้อความอะไรของตัวเองเลย** — กรณีนี้ใช้
`#[error(transparent)]` ได้ ลองเทียบสองเวอร์ชันโดยตรง:

```rust
use std::error::Error;
use std::num::ParseIntError;
use thiserror::Error as ThisError;

#[derive(Debug, ThisError)]
enum ConfigError {
    #[error("ฟิลด์ '{field}' ไม่ใช่ตัวเลขที่ถูกต้อง")]
    InvalidNumber {
        field: String,
        #[source]
        source: ParseIntError,
    },
}

// เวอร์ชันที่ 1: "ไม่ transparent" — AppError เพิ่มข้อความของตัวเอง เป็นชั้นที่มองเห็นได้ใน chain
#[derive(Debug, ThisError)]
enum AppErrorOpaque {
    #[error("โหลดการตั้งค่าแอปพลิเคชันล้มเหลว: {0}")]
    Config(#[from] ConfigError),
}

// เวอร์ชันที่ 2: transparent — AppError ไม่เพิ่มข้อความใดๆ เลย Display/source ทะลุไปที่ ConfigError ตรงๆ
#[derive(Debug, ThisError)]
enum AppErrorTransparent {
    #[error(transparent)]
    Config(#[from] ConfigError),
}

fn make_config_error() -> ConfigError {
    let source = "abc".parse::<u32>().unwrap_err();
    ConfigError::InvalidNumber { field: "timeout_seconds".to_string(), source }
}

fn print_chain(err: &dyn Error) {
    println!("error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        println!("caused by: {cause}");
        source = cause.source();
    }
}

fn main() {
    let opaque: AppErrorOpaque = make_config_error().into();
    println!("--- opaque ---");
    print_chain(&opaque);

    let transparent: AppErrorTransparent = make_config_error().into();
    println!("--- transparent ---");
    print_chain(&transparent);
}
```

ผลลัพธ์ (ทดสอบจริง — สังเกตความต่างระหว่างสองเวอร์ชันให้ดี):

```
--- opaque ---
error: โหลดการตั้งค่าแอปพลิเคชันล้มเหลว: ฟิลด์ 'timeout_seconds' ไม่ใช่ตัวเลขที่ถูกต้อง
caused by: ฟิลด์ 'timeout_seconds' ไม่ใช่ตัวเลขที่ถูกต้อง
caused by: invalid digit found in string
--- transparent ---
error: ฟิลด์ 'timeout_seconds' ไม่ใช่ตัวเลขที่ถูกต้อง
caused by: invalid digit found in string
```

**อธิบายความต่างที่จับต้องได้:**

- **เวอร์ชัน opaque**: `AppErrorOpaque` เป็น**ชั้นที่มองเห็นได้ใน chain อย่างสมบูรณ์** — `Display` ของมันพิมพ์
  ข้อความของตัวเอง (`"โหลดการตั้งค่าแอปพลิเคชันล้มเหลว: ..."` ซึ่งมี `{0}` แทรกข้อความของ `ConfigError` เข้ามาด้วย
  ทำให้ข้อความซ้ำกับบรรทัด `caused by:` แรกที่ตามมา — trade-off เดียวกับที่ Part 30 หัวข้อ 30.10 สังเกตไว้เรื่อง
  ข้อความซ้ำ) และ `source()` ของมันคืน `Some(&ConfigError)` ตรงๆ ทำให้ chain มีครบ **3 ชั้น**: `AppErrorOpaque` →
  `ConfigError` → `ParseIntError`
- **เวอร์ชัน transparent**: `AppErrorTransparent` **ไม่ปรากฏใน chain เลยแม้แต่ชั้นเดียว** — `error(transparent)`
  บอก `thiserror` ว่า "**ส่งต่อทั้ง `Display` และ `source()` ไปยัง field ที่ห่อไว้ตรงๆ โดยไม่เพิ่มชั้นของตัวเอง**"
  สังเกตว่าบรรทัด `error:` พิมพ์ข้อความของ `ConfigError` ทันที (ไม่ใช่ข้อความของ `AppErrorTransparent` เพราะมันไม่มี
  ข้อความของตัวเองให้พิมพ์) และ `caused by:` มีแค่ **1 บรรทัด** (ของ `ParseIntError`) ไม่ใช่ 2 บรรทัดแบบเวอร์ชันแรก
  — เพราะ `source()` ของ `AppErrorTransparent` **ไม่คืน `Some(&ConfigError)`** แต่คืนสิ่งที่ `ConfigError.source()`
  คืนตรงๆ (คือ "ทะลุผ่าน" ชั้นของตัวเองไปเลย ไม่ใช่แค่ "เพิ่มชั้นที่ไม่มีข้อความ") — นี่คือความหมายจริงของคำว่า
  "transparent" (โปร่งใส): `AppErrorTransparent` โปร่งใสสนิททั้งใน `Display` และใน chain จนเหมือนไม่มีตัวมันอยู่เลย
- **เมื่อไรควรใช้ `#[error(transparent)]`**: ใช้เมื่อ variant มีหน้าที่แค่ "รวมชนิด" ให้ `?` ทำงานได้ (เหมือนที่
  Part 30 หัวข้อ 30.6 ออกแบบ wrapper enum) **โดยที่ error ต้นทางมีข้อความที่ดีอยู่แล้วในตัวเอง และการเพิ่มข้อความ
  ของ wrapper เข้าไปจะซ้ำซ้อนหรือไม่ให้ข้อมูลเพิ่ม** — พบบ่อยเมื่อห่อ error จาก crate ภายนอกที่มี `Display` ดี
  อยู่แล้ว (เช่น `reqwest::Error`, `serde_json::Error`) เข้าไปใน error enum ของตัวเองเพื่อรวม type แต่ไม่ต้องการ
  เขียนข้อความซ้ำ — **ถ้าต้องการให้ variant นั้นเพิ่มมุมมองของ "ชั้นตัวเอง" ในระบบ** (เช่น "โหลดการตั้งค่าล้มเหลว"
  ที่ให้ context กว้างกว่าแค่ "ตัวเลขผิด") ควรใช้ `#[error("...")]` แบบมีข้อความ (opaque) แทน เพื่อให้ chain ยังคง
  สื่อสารได้ครบทุกชั้นเหมือนที่ Part 30 ออกแบบไว้

### 31.7 เขียน `ConfigError`/`AppError` ใหม่ทั้งหมดด้วย `thiserror`: เทียบบรรทัดต่อบรรทัด

ตอนนี้มาประกอบทุก attribute ที่เรียนมา (`#[error(...)]`, `#[from]`, `#[source]`) เขียน `ConfigError`/`AppError`
เวอร์ชันสมบูรณ์จาก Part 30 หัวข้อ 30.10 ใหม่ทั้งหมด **ให้ผลลัพธ์เหมือนกันทุกตัวอักษร**:

**เวอร์ชันมือจาก Part 30 (68 บรรทัด, ไม่รวม `use`):**

```rust
#[derive(Debug)]
enum ConfigError {
    MissingField(String),
    InvalidNumber { field: String, source: ParseIntError },
    InvalidBool { field: String, value: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::MissingField(field) => write!(
                f,
                "ไม่พบฟิลด์ที่จำเป็น '{field}' กรุณาเพิ่มบรรทัด '{field}=...' ในไฟล์ config"
            ),
            ConfigError::InvalidNumber { field, source } => {
                write!(f, "ฟิลด์ '{field}' ต้องเป็นตัวเลข: {source}")
            }
            ConfigError::InvalidBool { field, value } => write!(
                f,
                "ฟิลด์ '{field}' ต้องเป็น true หรือ false แต่พบค่า '{value}'"
            ),
        }
    }
}

impl Error for ConfigError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            ConfigError::InvalidNumber { source, .. } => Some(source),
            _ => None,
        }
    }
}

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Config(ConfigError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(_) => write!(f, "ไม่สามารถอ่านไฟล์ config ได้"),
            AppError::Config(_) => write!(f, "ไฟล์ config มีข้อมูลไม่ถูกต้อง"),
        }
    }
}

impl Error for AppError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            AppError::Io(e) => Some(e),
            AppError::Config(e) => Some(e),
        }
    }
}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}
impl From<ConfigError> for AppError {
    fn from(e: ConfigError) -> Self {
        AppError::Config(e)
    }
}
```

**เวอร์ชัน `thiserror` (24 บรรทัด รวม attribute, ไม่รวม `use`):**

```rust
use std::num::ParseIntError;
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("ไม่พบฟิลด์ที่จำเป็น '{0}' กรุณาเพิ่มบรรทัด '{0}=...' ในไฟล์ config")]
    MissingField(String),

    #[error("ฟิลด์ '{field}' ต้องเป็นตัวเลข: {source}")]
    InvalidNumber {
        field: String,
        #[source]
        source: ParseIntError,
    },

    #[error("ฟิลด์ '{field}' ต้องเป็น true หรือ false แต่พบค่า '{value}'")]
    InvalidBool { field: String, value: String },
}

#[derive(Debug, Error)]
enum AppError {
    #[error("ไม่สามารถอ่านไฟล์ config ได้")]
    Io(#[from] std::io::Error),

    #[error("ไฟล์ config มีข้อมูลไม่ถูกต้อง")]
    Config(#[from] ConfigError),
}
```

**ตารางเทียบ attribute แต่ละตัวกับโค้ดที่มันแทนที่:**

| `thiserror` attribute | แทนที่โค้ดมือส่วนไหนใน Part 30 |
|---|---|
| `#[derive(Debug, Error)]` เหนือ enum | `impl fmt::Display for X { ... }` ทั้งก้อน + `impl Error for X { ... }` ทั้งก้อน |
| `#[error("...")]` เหนือแต่ละ variant | หนึ่ง match arm ใน `impl Display` |
| `#[source]` เหนือ field | โค้ดใน `source()` ที่คืน `Some(&field)` สำหรับ variant นั้น (variant ที่ไม่มี attribute นี้จะคืน `None` โดย default เหมือนที่ Part 30 อธิบายไว้เรื่อง default implementation) |
| `#[from]` เหนือ field | `impl From<FieldType> for X { ... }` ทั้งก้อน (**และ** ครอบคลุม `#[source]` ให้ในตัวด้วยอัตโนมัติ) |

จำนวนบรรทัดลดลงจาก 68 เหลือ 24 บรรทัด (ลดลงมากกว่าครึ่ง) **โดยที่พฤติกรรมทุกอย่างเหมือนกันทุกประการ**: `match`
แยกกรณี error ยังทำงานได้เต็มรูปแบบเหมือนเดิม (เพราะยังเป็น enum ธรรมดา), `source()` ยังคืนค่าถูกต้องตาม variant,
`?` ยังแปลง `io::Error`/`ConfigError` เป็น `AppError` ได้อัตโนมัติทุกจุดในโปรแกรม — ตัวอย่างแบบเต็มที่พิสูจน์เรื่อง
นี้ (พร้อม error chain ที่พิมพ์ผลเหมือนกับ Part 30 หัวข้อ 30.10 เป๊ะทุกตัวอักษร) จะแสดงในหัวข้อ 31.15

### 31.8 `anyhow`: ปรัชญาที่แตกต่างโดยสิ้นเชิงจาก `thiserror`

ทุกอย่างที่เรียนมาจนถึงตอนนี้ (`thiserror` รวมทั้งเวอร์ชันมือจาก Part 30) มีจุดร่วมกันอย่างหนึ่ง: **มันคือการสร้าง
error TYPE ของตัวเอง** — enum ที่มีโครงสร้างชัดเจน ให้ผู้เรียก `match` แยกกรณีได้ ตามที่ Part 30 หัวข้อ 30.1
อธิบายไว้ว่าเหมาะกับ **library** ที่ผู้เรียกต้องตัดสินใจต่างกันไปตามสาเหตุของความล้มเหลว

`anyhow` เดินเส้นทางที่**ตรงข้ามกันโดยสิ้นเชิง**: มันให้ **type เดียว** ชื่อ `anyhow::Error` ที่เป็น **catch-all
error type** สำหรับ**ทุกความล้มเหลวในโปรแกรม** โดยไม่แคร์ว่าต้นทางเป็น error ชนิดไหน — แนวคิดนี้**คล้ายกับ
`Box<dyn std::error::Error>` ที่ Part 12 หัวข้อ 12.7-12.8 แนะนำไว้เป๊ะๆ** (ทั้งคู่คือ "type เดียวที่รับ error ได้
ทุกชนิดที่ implement `Error`") แต่ `anyhow::Error` **ergonomic กว่ามาก** ในรายละเอียดที่จะเห็นตลอดหัวข้อถัดไป
(สร้าง ad-hoc error ได้ง่าย, แนบ context ได้, พิมพ์ chain แบบเต็มได้โดยไม่ต้องเขียน loop เอง)

**คำถามที่สำคัญที่สุดของบทนี้คือ: แล้วเมื่อไรควรใช้ `thiserror` เมื่อไรควรใช้ `anyhow`?** คำตอบคือหลักการเดียวกับ
ตารางในหัวข้อ 30.1 ของ Part 30 ขยายออกไป:

| | `thiserror` | `anyhow` |
|---|---|---|
| สร้าง type ใหม่ไหม | ใช่ — enum ของตัวเองที่ compiler รู้จักครบ | ไม่ — ใช้ `anyhow::Error` type เดียวสำหรับทุก error |
| ผู้เรียก `match` แยกกรณีได้ไหม | ได้เต็มรูปแบบ (เหมือน hand-rolled enum) | ไม่ได้ตรงๆ (ต้อง downcast — หัวข้อ 31.12) |
| เหมาะกับ | **library** — โค้ดที่คนอื่น (หรือ module อื่น) จะเรียกใช้ และต้องตัดสินใจต่างกันไปตามสาเหตุ | **application/binary** — `main()`, CLI tool, handler ระดับบนสุดที่แค่ต้อง propagate error ขึ้นมาแล้วแสดงให้อ่านง่าย |
| ตัวอย่างในบทนี้ | `ConfigError`/`AppError` ในฐานะ public API ของ `config_core` | `main()` ที่เรียก `config_core` แล้วแค่ต้อง "ทำงานให้จบ หรือพิมพ์ error ที่อ่านง่ายแล้วจบโปรแกรม" |

**หลักที่จำง่ายที่สุด**: ถามตัวเองว่า **"ผู้เรียกฟังก์ชันนี้จำเป็นต้องเขียน `match`/`if let` แยกพฤติกรรมไปตาม
สาเหตุของ error หรือไม่?"** — ถ้า**ใช่** (เช่น "ถ้า field หาย ให้ใช้ค่า default แต่ถ้า parse ผิดให้หยุดโปรแกรม")
ต้องใช้ type ที่มีโครงสร้าง (`thiserror` หรือ hand-rolled enum) ถ้า**ไม่ใช่** (แค่ต้องการให้ error ลอยขึ้นมาถึง
จุดที่ log/แสดงผล/จบโปรแกรม) `anyhow` คือตัวเลือกที่เขียนน้อยที่สุดและอ่านง่ายที่สุด — หัวข้อ 31.13 จะรวบรวม
framework การตัดสินใจนี้ให้ครบถ้วนกว่านี้อีกครั้ง

### 31.9 เพิ่ม `anyhow` เข้าโปรเจกต์

```bash
cargo add anyhow
```

ผลลัพธ์จริง:

```
      Adding anyhow v1.0.104 to dependencies
             Features:
             + std
             - backtrace
```

```toml
[dependencies]
anyhow = "1.0.104"
```

### 31.10 `anyhow::Result<T>` และ `?` ที่แปลงชนิดใดๆ ได้อัตโนมัติ — ไม่ต้องมี `impl From` เลย

`anyhow` export type alias ชื่อ `Result<T>` (แปลว่า `Result<T, anyhow::Error>`) ให้ใช้แทน `std::result::Result`
เต็มรูปแบบ:

```rust
use anyhow::{anyhow, Result};

// anyhow::Result<u16> คือ shorthand ของ Result<u16, anyhow::Error>
fn parse_port(raw: &str) -> Result<u16> {
    let value: u32 = raw.parse::<u32>()?; // ParseIntError -> anyhow::Error อัตโนมัติ
    let port: u16 = value.try_into()?; // TryFromIntError -> anyhow::Error อัตโนมัติเหมือนกัน (คนละชนิด error!)
    Ok(port)
}

fn validate_port(port: u16) -> Result<u16> {
    if port == 0 {
        return Err(anyhow!("port ต้องไม่เป็น 0"));
    }
    Ok(port)
}

fn main() {
    println!("{:?}", parse_port("8080"));
    println!("{:?}", parse_port("abc"));
    println!("{:?}", parse_port("999999"));
    println!("{:?}", validate_port(0));
}
```

ผลลัพธ์ (ทดสอบจริง, ปิด backtrace ด้วย `RUST_BACKTRACE=0` เพื่อให้อ่านง่าย):

```
Ok(8080)
Err(invalid digit found in string)
Err(out of range integral type conversion attempted)
Err(port ต้องไม่เป็น 0)
```

**นี่คือจุดที่ต่างจาก Part 30 อย่างสิ้นเชิง**: ฟังก์ชัน `parse_port` เจอ error **สองชนิดที่ไม่เกี่ยวข้องกันเลย**
(`ParseIntError` จาก `.parse()` และ `TryFromIntError` จาก `.try_into()`) แล้ว `?` แปลงทั้งคู่เป็น `anyhow::Error`
ได้ทันที **โดยไม่มี `impl From` สักบรรทัดเดียวในโค้ดนี้เลย** — เทียบกับ Part 30 ที่ทุกครั้งที่เจอ error ชนิดใหม่
เข้ามาในฟังก์ชัน ต้องเขียน `impl From<NewErrorType> for MyError` เพิ่มอีกหนึ่งบล็อก (หัวข้อ 30.6) หรือ Part 30
หัวข้อ 31.4 ที่ยังต้องเขียน `#[from]` เพิ่มทีละ field อยู่ดี

**ทำไม `?` ทำแบบนี้ได้**: `anyhow` เขียน **blanket implementation** ไว้ล่วงหน้า (แนวคิดเดียวกับ blanket impl ของ
`Into<T>` ที่ Part 30 หัวข้อ 30.8b อธิบายไว้ และเหมือนกับที่ standard library เขียน `impl<E: Error> From<E> for
Box<dyn Error>` ให้ตามที่ Part 30 หัวข้อ 30.1 อธิบายไว้) ประมาณนี้:

```
impl<E> From<E> for anyhow::Error
where
    E: std::error::Error + Send + Sync + 'static,
{
    // ...
}
```

พูดเป็นภาษาคน: **error ชนิดใดก็ตามที่ implement `std::error::Error` (บวกเงื่อนไขเรื่อง thread-safety
`Send + Sync` และไม่มี borrowed reference `'static` ซึ่ง error ทั่วไปเกือบทั้งหมดมีอยู่แล้วโดยธรรมชาติ) จะแปลง
เป็น `anyhow::Error` ได้ทันทีโดยไม่ต้องเขียน `impl From` เพิ่มเติมเลยแม้แต่บรรทัดเดียว** — นี่คือความต่างที่
สำคัญที่สุดระหว่าง `anyhow::Error` กับ custom error enum: **enum ของคุณต้องมีคน (คุณ หรือ `thiserror`) เขียน
`impl From` ให้ครบทุกแหล่งที่มา แต่ `anyhow::Error` เขียน `impl From` ครอบคลุม "ทุกแหล่งที่มาที่มีอยู่ในจักรวาล"
ไว้ล่วงหน้าแล้วในบรรทัดเดียว** — ราคาที่ต้องจ่ายคือ**สูญเสียข้อมูลชนิดที่แน่นอนไปเหมือนกับ `Box<dyn Error>`**
ตามที่ Part 30 หัวข้อ 30.1 วิเคราะห์ไว้ (ผู้เรียกไม่รู้จาก signature ว่า error อาจเป็นชนิดไหนได้บ้าง) — ซึ่งเป็น
ราคาที่**ยอมรับได้เต็มที่**สำหรับโค้ดระดับ application ตามหลักการในหัวข้อ 31.8

### 31.11 `anyhow!()`: สร้าง Error แบบ Ad-Hoc

จากตัวอย่างก่อนหน้า `anyhow!("port ต้องไม่เป็น 0")` สร้าง `anyhow::Error` ขึ้นมาจาก **format string ตรงๆ** โดย
ไม่ต้องมี struct/enum ของตัวเองเลย — ทำงานเหมือน `format!()` ทุกประการแต่ผลลัพธ์เป็น error object พร้อมใช้กับ
`Err(...)`/`return`/`?` ได้ทันที:

```rust
use anyhow::anyhow;

fn check_balance(balance: i64) -> anyhow::Result<()> {
    if balance < 0 {
        return Err(anyhow!("ยอดเงินติดลบ: {balance} (ควรไม่เกิด ตรวจสอบ logic การหักบัญชี)"));
    }
    Ok(())
}

fn main() {
    println!("{:?}", check_balance(-500));
    println!("{:?}", check_balance(1000));
}
```

ผลลัพธ์:

```
Err(ยอดเงินติดลบ: -500 (ควรไม่เกิด ตรวจสอบ logic การหักบัญชี))
Ok(())
```

**เมื่อไรควรใช้ `anyhow!()` เทียบกับ error type ที่มีโครงสร้าง**: `anyhow!()` เหมาะกับสถานการณ์ที่เป็น "จุดจบ"
ของ error (invariant ที่ไม่ควรเกิดขึ้นเลย, การตรวจสอบแบบ ad-hoc ที่ทำครั้งเดียวในโค้ดทั้งโปรแกรม) มากกว่าสถานการณ์
ที่ผู้เรียกจะต้องแยกแยะสาเหตุ — ถ้าคุณพบว่ากำลังเขียน `anyhow!("...")` ข้อความคล้ายๆ กันหลายที่ในโค้ด และมีคน
พยายามเช็ค `.to_string().contains(...)` เพื่อแยกแยะประเภทของมันในภายหลัง นั่นคือสัญญาณว่าควรเปลี่ยนไปใช้ enum
ที่มีโครงสร้าง (`thiserror`) แทน — string message ไม่ควรใช้เป็นกลไกแยกกรณี error เหมือนกับที่ Part 12 หัวข้อ
12.3 เตือนไว้เรื่อง error ที่เป็น string ล้วนๆ

### 31.12 `.context()` และ `.with_context()`: แนบข้อความระหว่าง Error เดินทางขึ้น Call Stack

นี่คือฟีเจอร์ที่ทรงพลังที่สุดของ `anyhow` และเป็นสิ่งที่ **ไม่มีอะไรเทียบเท่าใน `thiserror` หรือ hand-rolled enum
เลย** — ลองนึกภาพระบบที่มี 3 ชั้น (เหมือนที่ Part 30 หัวข้อ 30.5 อธิบายไว้เรื่อง application → service → database)
แล้วแต่ละชั้นอยากเพิ่ม "มุมมองของตัวเอง" เข้าไปในข้อความ error โดยไม่ต้องประกาศ type ใหม่เลย:

```rust
use anyhow::{Context, Result};
use std::collections::HashMap;

// ชั้นที่ 3 (ล่างสุด): อ่านไฟล์ดิบๆ
fn read_raw_file(path: &str) -> Result<String> {
    std::fs::read_to_string(path).with_context(|| format!("อ่านไฟล์ '{path}' ไม่สำเร็จ"))
}

// ชั้นที่ 2: parse เนื้อหาเป็น key=value แล้วดึงฟิลด์ที่ต้องการ
fn parse_port_field(content: &str) -> Result<u16> {
    let map: HashMap<&str, &str> = content
        .lines()
        .filter_map(|line| line.split_once('='))
        .collect();

    let raw = map
        .get("port")
        .context("ไม่พบฟิลด์ 'port' ในไฟล์ config")?;

    raw.parse::<u16>()
        .with_context(|| format!("ฟิลด์ 'port' มีค่า '{raw}' ซึ่งไม่ใช่ตัวเลขที่ถูกต้อง"))
}

// ชั้นที่ 1 (บนสุด): ประกอบทุกอย่างเข้าด้วยกัน เพิ่ม context ระดับแอปพลิเคชัน
fn load_server_port(path: &str) -> Result<u16> {
    let content = read_raw_file(path).context("โหลดการตั้งค่าเซิร์ฟเวอร์ล้มเหลว")?;
    let port = parse_port_field(&content).context("การตั้งค่าเซิร์ฟเวอร์มีข้อมูลไม่ถูกต้อง")?;
    Ok(port)
}

fn main() {
    match load_server_port("/nonexistent/server.conf") {
        Ok(port) => println!("port = {port}"),
        Err(e) => {
            println!("--- แบบสั้น (Display) ---");
            println!("{e}");
            println!("--- แบบเต็ม (Debug, {{:?}}) ---");
            println!("{e:?}");
        }
    }
}
```

ผลลัพธ์ (ทดสอบจริง — ไม่มีไฟล์ `/nonexistent/server.conf` อยู่จริงในเครื่อง):

```
--- แบบสั้น (Display) ---
โหลดการตั้งค่าเซิร์ฟเวอร์ล้มเหลว
--- แบบเต็ม (Debug, {:?}) ---
โหลดการตั้งค่าเซิร์ฟเวอร์ล้มเหลว

Caused by:
    0: อ่านไฟล์ '/nonexistent/server.conf' ไม่สำเร็จ
    1: No such file or directory (os error 2)
```

และถ้าไฟล์อ่านได้แต่ฟิลด์ `port` มีค่าไม่ใช่ตัวเลข (ทดสอบ `parse_port_field("port=abc")` ตรงๆ):

```
--- แบบสั้น (Display) ---
ฟิลด์ 'port' มีค่า 'abc' ซึ่งไม่ใช่ตัวเลขที่ถูกต้อง
--- แบบเต็ม (Debug, {:?}) ---
ฟิลด์ 'port' มีค่า 'abc' ซึ่งไม่ใช่ตัวเลขที่ถูกต้อง

Caused by:
    invalid digit found in string
```

**อธิบายกลไกทีละส่วน เทียบกับสิ่งที่ Part 30 ทำด้วยมือ:**

- **`.context("...")` และ `.with_context(|| ...)`** — ทั้งสองทำสิ่งเดียวกัน (ห่อ error เดิมไว้พร้อมข้อความใหม่
  แล้วคืน `anyhow::Error` ตัวใหม่ที่มี error เดิมเป็น "source" ของมัน) ต่างกันแค่ **`.context()` รับ string/value
  ตรงๆ** (evaluate ทันทีไม่ว่า error จะเกิดหรือไม่) ส่วน **`.with_context(|| ...)` รับ closure** (evaluate เฉพาะ
  ตอนที่เป็น `Err` เท่านั้น) — หลักปฏิบัติ: ใช้ `.context()` กับ string literal คงที่ (เร็วกว่า ไม่มี allocation
  พิเศษ) ใช้ `.with_context(|| format!(...))` เมื่อข้อความต้องคำนวณจากตัวแปร (เพราะ `format!()` จะไม่ถูกเรียกเลย
  ถ้าไม่มี error เกิดขึ้น ประหยัดกว่าการเรียก `format!()` ทุกครั้งแม้ตอนที่ไม่มี error)
- **ทุกครั้งที่เรียก `.context()`/`.with_context()` คือการเพิ่ม "ชั้น" ใหม่เข้าไปใน chain โดยอัตโนมัติ** — เทียบ
  กับ Part 30 หัวข้อ 30.5 ที่ต้องออกแบบ enum variant ใหม่ + เขียน `source()` + เขียน `impl From` ทุกครั้งที่ต้อง
  เพิ่มชั้นใหม่ ในขณะที่ `anyhow` แค่เรียก method หนึ่งตัวตรงจุดที่ต้องการเพิ่ม context เท่านั้น — chain ทั้งหมด
  ในตัวอย่างนี้มี 3 ชั้น (`"โหลดการตั้งค่าเซิร์ฟเวอร์ล้มเหลว"` → `"อ่านไฟล์ ... ไม่สำเร็จ"` → `"No such file or
  directory"`) โดยไม่มี type ใหม่ถูกประกาศเลยแม้แต่ตัวเดียวตลอดทั้งตัวอย่าง
- **`{e}` (Display) พิมพ์แค่ context ชั้นบนสุด** — เหมือนกับที่ Part 30 ออกแบบให้ `Display` ของแต่ละชั้นพิมพ์แค่
  "มุมมองของตัวเอง" (หัวข้อ 30.8 กฎข้อ 4) — `anyhow::Error` ทำสิ่งนี้ให้อัตโนมัติโดยไม่ต้องออกแบบเอง
- **`{e:?}` (Debug) พิมพ์ chain แบบเต็มพร้อมคำว่า `Caused by:`** — นี่คือสิ่งที่แทนที่ฟังก์ชัน `print_error_chain`
  ที่ Part 30 หัวข้อ 30.5 เขียนด้วยมือ (`while let Some(cause) = source { println!("caused by: {cause}"); source
  = cause.source(); }`) **ทั้งหมด** — `anyhow::Error` implement `Debug` เองให้ไล่ `source()` ทั้ง chain แล้ว
  format ออกมาเป็นข้อความที่อ่านง่ายพร้อม numbering (`0:`, `1:`, ...) เมื่อ chain มีมากกว่า 1 ชั้น (ถ้ามีแค่ชั้น
  เดียวจะไม่มี numbering ตามที่เห็นในตัวอย่างที่สอง) — คุณได้ฟังก์ชัน `print_error_chain` แบบสมบูรณ์**มาโดย
  อัตโนมัติ** แค่ format ด้วย `{:?}` เท่านั้น ไม่ต้องเขียน loop เองเลย
- **`fn main() -> anyhow::Result<()>`** — ถ้าเปลี่ยน `main()` ให้คืน `anyhow::Result<()>` แล้วใช้ `?` ตรงๆ (ไม่ต้อง
  `match`) Rust จะพิมพ์ `Error: ` ตามด้วยผลลัพธ์ของ `{:?}` ให้อัตโนมัติเมื่อโปรแกรมจบด้วย `Err` (ผ่าน trait
  `Termination` ที่ standard library implement ให้ `Result<T, E: Debug>`) พร้อม exit code 1 — ทดสอบจริง:

```rust
use anyhow::{Context, Result};

fn load() -> Result<String> {
    std::fs::read_to_string("/no/such/file.conf").context("โหลดไฟล์ล้มเหลว")
}

fn main() -> Result<()> {
    let content = load()?;
    println!("{content}");
    Ok(())
}
```

  ผลลัพธ์เมื่อรัน (exit code เป็น 1 ด้วย):

  ```
  Error: โหลดไฟล์ล้มเหลว

  Caused by:
      No such file or directory (os error 2)
  ```

  รูปแบบ `fn main() -> anyhow::Result<()> { ...; Ok(()) }` เป็น pattern ที่พบมากที่สุดในโปรแกรม Rust ระดับ
  application ที่ใช้ `anyhow` — มันทำให้ `main()` เขียนแบบเดียวกับฟังก์ชันอื่นทั้งโปรแกรม (ใช้ `?` ได้ตรงๆ) โดยไม่
  ต้อง `match` ที่จุดสุดท้ายเองเลย

### 31.13 `bail!()` และ `ensure!()`: Early Return แบบสั้นกระชับ

`anyhow` มีมาโครสองตัวที่ช่วยลดโค้ด `if ... { return Err(...) }` ให้สั้นลง:

```rust
use anyhow::{bail, ensure, Result};

fn withdraw(balance: u64, amount: u64) -> Result<u64> {
    // ensure!(condition, "message") = ถ้า condition เป็น false ให้ return Err(anyhow!("message")) ทันที
    ensure!(amount > 0, "จำนวนเงินที่ถอนต้องมากกว่า 0 (ได้รับ {amount})");

    if amount > balance {
        // bail!("message") = return Err(anyhow!("message")) ทันที (เหมือน ensure! แต่ไม่มีเงื่อนไขในตัวมันเอง)
        bail!("ยอดเงินไม่พอ: มี {balance} แต่ต้องการถอน {amount}");
    }

    Ok(balance - amount)
}

fn main() {
    println!("{:?}", withdraw(1000, 300));
    println!("{:?}", withdraw(1000, 0));
    println!("{:?}", withdraw(1000, 5000));
}
```

ผลลัพธ์:

```
Ok(700)
Err(จำนวนเงินที่ถอนต้องมากกว่า 0 (ได้รับ 0))
Err(ยอดเงินไม่พอ: มี 1000 แต่ต้องการถอน 5000)
```

**เปรียบเทียบให้ชัด**: `ensure!(amount > 0, "...")` เทียบเท่ากับ:

```rust
if !(amount > 0) {
    return Err(anyhow::anyhow!("จำนวนเงินที่ถอนต้องมากกว่า 0 (ได้รับ {amount})"));
}
```

ส่วน `bail!("...")` เทียบเท่ากับ:

```rust
return Err(anyhow::anyhow!("ยอดเงินไม่พอ: มี {balance} แต่ต้องการถอน {amount}"));
```

ทั้งสองมาโครนี้**ไม่ได้เพิ่มความสามารถใหม่**เลย (แค่ `if` + `return Err(anyhow!(...))` ที่เขียนสั้นลง) แต่ช่วยให้
โค้ดที่มีการตรวจสอบ precondition หลายจุด (นิยมเรียกว่า "guard clause") อ่านเป็นรายการเงื่อนไขที่ชัดเจน ไม่ต้อง
มี `if { return }` พันกันหลายชั้นให้ตาลาย — `ensure!` เหมาะกับการเช็ค**เงื่อนไข** (invariant ที่ควรเป็นจริง)
ส่วน `bail!` เหมาะกับจุดที่ **รู้แน่ชัดแล้วว่าต้องล้มเหลว** (ไม่มีเงื่อนไขให้เช็คอีก เพราะเดินทางมาถึงจุดนี้ได้ก็
แปลว่าล้มเหลวแน่นอน เช่นอยู่ใน `match` arm ที่เป็นกรณีผิดพลาดโดยตรง)

### 31.14 Downcasting: กู้คืน Concrete Error Type จาก `anyhow::Error`

หัวข้อ 31.8 บอกไว้ว่า `anyhow::Error` "ไม่ให้ผู้เรียก `match` แยกกรณีได้ตรงๆ" — แต่นั่นไม่ได้แปลว่า **ทำไม่ได้
เลย** เหมือนกับที่ Part 30 หัวข้อ 30.1 แสดงให้เห็นว่า `Box<dyn Error>` ยังมี `.downcast_ref::<T>()` ให้ใช้
`anyhow::Error` ก็มี `.downcast::<T>()` ให้เช่นกัน (และ `.downcast_ref::<T>()`/`.downcast_mut::<T>()` แบบไม่ยึด
ownership ด้วย) — ใช้ในสถานการณ์ที่ **ส่วนใหญ่ของโค้ดใช้ `anyhow` เป็นหลัก แต่มี error บางชนิดเจาะจงที่ต้องการ
"กู้คืน" หรือตอบสนองเป็นพิเศษ**:

```rust
use anyhow::{Context, Result};
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("ไม่พบฟิลด์ '{0}'")]
    MissingField(String),
    #[error("ฟิลด์ '{0}' ไม่ใช่ตัวเลข")]
    InvalidNumber(String),
}

fn load_port(raw: Option<&str>) -> Result<u16> {
    let raw = raw.ok_or_else(|| ConfigError::MissingField("port".to_string()))?;
    let port: u16 = raw
        .parse()
        .map_err(|_| ConfigError::InvalidNumber("port".to_string()))?;
    Ok(port)
}

fn load_with_default(raw: Option<&str>) -> Result<u16> {
    match load_port(raw).context("โหลด port ล้มเหลว") {
        Ok(port) => Ok(port),
        Err(err) => {
            // ไล่ดูว่า "ต้นตอที่แท้จริง" เป็น ConfigError::MissingField หรือไม่
            // ถ้าใช่ ให้ใช้ค่า default แทนแล้ว "กู้คืน" จากความล้มเหลวได้ตรงจุด
            match err.downcast::<ConfigError>() {
                Ok(ConfigError::MissingField(_)) => {
                    println!("-> ไม่พบ port ในไฟล์ ใช้ค่า default 8080 แทน");
                    Ok(8080)
                }
                Ok(other) => Err(other.into()),
                Err(original) => Err(original), // ไม่ใช่ ConfigError เลย ส่งต่อ error เดิม
            }
        }
    }
}

fn main() -> Result<()> {
    println!("port1 = {}", load_with_default(None)?);
    println!("port2 = {}", load_with_default(Some("8080"))?);

    match load_with_default(Some("abc")) {
        Ok(p) => println!("port3 = {p}"),
        Err(e) => println!("port3 error: {e:?}"),
    }

    Ok(())
}
```

ผลลัพธ์ (ทดสอบจริง):

```
-> ไม่พบ port ในไฟล์ ใช้ค่า default 8080 แทน
port1 = 8080
port2 = 8080
port3 error: ฟิลด์ 'port' ไม่ใช่ตัวเลข
```

**อธิบายกลไก:**

- **`err.downcast::<ConfigError>()`** — method นี้**ยึด ownership** ของ `err` แล้วคืน `Result<ConfigError,
  anyhow::Error>`: **`Ok(config_error)`** ถ้า error ที่ห่ออยู่ภายในเป็น `ConfigError` จริงๆ (concrete type ตรง
  เป๊ะ) หรือ **`Err(err)`** คืน `anyhow::Error` **ตัวเดิม** กลับมาถ้าไม่ตรงชนิด (ไม่ได้ทำลายข้อมูลไปไหน แค่บอกว่า
  "เดาผิด") — สังเกตว่า `.context()` ที่เพิ่มไว้ก่อนหน้า (`"โหลด port ล้มเหลว"`) **ไม่ทำให้ downcast เจอ
  `ConfigError` ไม่ได้** เพราะ `anyhow::Error` เก็บ chain ทั้งหมดไว้ภายใน `.downcast::<T>()` จะไล่หา **ที่ใดก็ได้
  ในทั้ง chain** ที่ตรงกับ `T` ไม่ใช่แค่ชั้นบนสุดเท่านั้น
- **`Ok(ConfigError::MissingField(_))`** — เพราะ `downcast` คืน concrete enum เต็มรูปแบบ (ไม่ใช่ trait object)
  เราจึง `match` แยก variant ต่อได้อีกชั้นตามปกติ (`MissingField` ปะทะ `InvalidNumber`) — ได้ความสามารถแยกกรณี
  กลับมาเหมือนตอนใช้ hand-rolled enum ตรงๆ เฉพาะจุดที่ต้องการ
- **`Err(other) => Err(other.into())`** — ถ้า downcast สำเร็จแต่เป็น variant อื่นที่ไม่ต้องการจัดการพิเศษ ต้อง
  แปลง `ConfigError` กลับเป็น `anyhow::Error` ด้วย `.into()` (ใช้ blanket impl เดียวกับหัวข้อ 31.10) ก่อนคืนกลับ
  ไป เพื่อให้ signature ของฟังก์ชันยังตรงกัน (`anyhow::Result<u16>`)
- **`Err(original) => Err(original)`** — ถ้า downcast ล้มเหลวเพราะ error ไม่ใช่ `ConfigError` เลย (เช่นมันมาจาก
  แหล่งอื่นในระบบ) จะได้ `anyhow::Error` **ตัวเดิม** กลับมาตรงๆ (ไม่ต้อง `.into()` เพราะมันเป็น `anyhow::Error`
  อยู่แล้ว) — ส่งต่อ error เดิมขึ้นไปโดยไม่มีข้อมูลสูญหาย

**ข้อคิดสำคัญที่ควรสรุปจากหัวข้อนี้**: การ downcast ควรเป็น**ทางเลือกสำรอง** ไม่ใช่กลไกหลักของการออกแบบ — ถ้า
คุณพบว่าโค้ดของคุณ downcast บ่อยมากในหลายจุด นั่นคือสัญญาณว่าจริงๆ แล้วผู้เรียกต้องการ **type ที่มีโครงสร้าง**
(กลับไปใช้ `thiserror` หรือ hand-rolled enum ตรงๆ) มากกว่าการพึ่ง `anyhow::Error` แล้วเดาชนิดย้อนหลัง — downcast
มีประโยชน์สูงสุดในจุด**พิเศษเฉพาะจุดเดียว** (เหมือนตัวอย่างนี้ที่มีแค่จุดเดียวที่ต้องการ "กู้คืน" จาก
`MissingField`) ไม่ใช่กลไกหลักของทั้งระบบ

### 31.15 Decision Framework: เมื่อไรควรใช้อะไร

รวบรวมทุกอย่างที่เรียนมาในบทนี้และ Part 30 เป็นแนวทางตัดสินใจเดียว:

```
กำลังออกแบบ error type ให้ฟังก์ชัน/module ใหม่?
│
├─ ผู้เรียกต้องแยกพฤติกรรมไปตามสาเหตุของ error หรือไม่?
│  (เช่น "ถ้า NotFound ให้สร้างใหม่ ถ้า PermissionDenied ให้แจ้งเตือน")
│
├─ ใช่ → นี่คือ "library-style" error
│   │
│   ├─ ยอมรับ dependency เพิ่ม (thiserror) ได้ไหม?
│   │  ├─ ได้ → ใช้ thiserror (หัวข้อ 31.2-31.7)
│   │  └─ ไม่ได้ (เช่น crate ที่ห้ามมี dependency เลย) → hand-rolled enum แบบ Part 30
│   │
│   └─ ผลลัพธ์: enum ที่มี match แบบ exhaustive, #[non_exhaustive] ถ้าเป็น public API
│
└─ ไม่ใช่ → นี่คือ "application-style" error (แค่ต้อง propagate + แสดงผล)
    │
    └─ ใช้ anyhow::Result<T> + .context()/.with_context() (หัวข้อ 31.8-31.13)
       ผลลัพธ์: โค้ดสั้น อ่านง่าย พิมพ์ chain เต็มด้วย {:?} โดยไม่ต้องออกแบบ type เอง
```

**ข้อเท็จจริงที่สำคัญที่สุดของหัวข้อนี้**: โปรเจกต์จริงส่วนใหญ่ **ไม่ได้เลือกอย่างใดอย่างหนึ่งแล้วใช้ทั้งโปรเจกต์**
— pattern ที่พบมากที่สุดใน Rust ecosystem จริง (สังเกตได้จาก crate ที่มีชื่อเสียงจำนวนมากบน crates.io) คือ:

> **library crate ในโปรเจกต์ใช้ `thiserror` สำหรับ public error type ของมัน ส่วน binary/CLI crate ที่เรียกใช้
> library นั้นใช้ `anyhow` ใน `main()` และฟังก์ชันระดับบน** — ทั้งสองอยู่ใน **workspace เดียวกัน** ตามโครงสร้างที่
> Part 17 หัวข้อ 17.5-17.7 สอนไว้

เหตุผลที่ pattern นี้สมเหตุสมผล: **`config_core` (library) ไม่รู้ล่วงหน้าว่าใครจะมาเรียกใช้มัน** — อาจเป็น CLI
tool, web server, หรือ library อื่นที่ต้องการ `match` แยกกรณี error ของมันเองต่อ ดังนั้น `config_core` **ต้อง**
คืน type ที่มีโครงสร้างชัดเจน (`thiserror`) เพื่อไม่ปิดโอกาสผู้เรียกในอนาคต — แต่ `config_cli` (binary) **รู้
แน่นอนแล้ว**ว่าตัวเองคือจุดสุดท้าย (ไม่มีใครเรียกมันต่ออีก) หน้าที่ของมันแค่ "ทำงานให้จบ หรือแสดง error ให้อ่าน
ง่ายที่สุดแล้วจบโปรแกรม" — `anyhow` คือเครื่องมือที่ตรงกับหน้าที่นี้พอดี

**โครงร่างของ workspace แบบสองก้อนนี้** (ทดสอบ compile และรันจริงแล้ว — ให้ผลลัพธ์เหมือนกับตัวอย่างเต็มในหัวข้อ
31.16 ทุกประการ เพราะเป็นโค้ดชุดเดียวกัน แค่แยกไฟล์กันจริงๆ):

```toml
# config_workspace/Cargo.toml (root — ไม่มี [package] เลย ตามที่ Part 17 หัวข้อ 17.6 สอนไว้)
[workspace]
resolver = "2"
members = ["config_core", "config_cli"]
```

```toml
# config_workspace/config_core/Cargo.toml
[package]
name = "config_core"
version = "0.1.0"
edition = "2021"

[dependencies]
thiserror = "2"
```

```toml
# config_workspace/config_cli/Cargo.toml
[package]
name = "config_cli"
version = "0.1.0"
edition = "2021"

[dependencies]
config_core = { path = "../config_core" } # path dependency ตามที่ Part 17 หัวข้อ 17.7 สอนไว้
anyhow = "1"
```

```rust
// config_workspace/config_core/src/lib.rs (ตัดมาแค่ส่วน error type — โค้ดเต็มดูหัวข้อ 31.16)
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("ไม่พบฟิลด์ที่จำเป็น '{0}'")]
    MissingField(String),
    // ... variant อื่นๆ
}

#[derive(Debug, Error)]
pub enum AppError {
    #[error("ไม่สามารถอ่านไฟล์ config ได้")]
    Io(#[from] std::io::Error),
    #[error("ไฟล์ config มีข้อมูลไม่ถูกต้อง")]
    Config(#[from] ConfigError),
}
```

```rust
// config_workspace/config_cli/src/main.rs
use anyhow::{Context, Result};
use config_core::load_app_config;

fn main() -> Result<()> {
    let path = "/nonexistent/app.conf";
    let config = load_app_config(path)
        .with_context(|| format!("โหลด config จาก '{path}' ไม่สำเร็จ"))?;
    println!("{config:?}");
    Ok(())
}
```

สังเกตว่า **`config_cli` ไม่ต้องเขียน `impl From<ConfigError> for anyhow::Error` หรือ `impl From<AppError> for
anyhow::Error` เลยแม้แต่บรรทัดเดียว** — เพราะ `AppError` (ที่ `thiserror` generate `impl std::error::Error` ให้
เรียบร้อยแล้ว) เข้าเงื่อนไข blanket impl ของ `anyhow::Error` ที่อธิบายไว้ในหัวข้อ 31.10 โดยอัตโนมัติ — นี่คือจุดที่
**`thiserror` และ `anyhow` ทำงานเข้ากันได้อย่างไร้รอยต่อ**: `thiserror` ทำให้ error type ของ library "ถูกต้อง
ตามหลักการ" ( implement `std::error::Error` เต็มรูปแบบ) และความถูกต้องนั้นเองคือสิ่งที่ทำให้ `anyhow` ที่ฝั่ง
binary รับมันไปใช้ได้ทันทีโดยไม่ต้องเขียนอะไรเพิ่ม

### 31.16 ตัวอย่างจริงแบบเต็ม: Config File Loader ครั้งที่สาม — `thiserror` + `anyhow` ทำงานร่วมกัน

มาประกอบทุกอย่างเข้าด้วยกันเป็นครั้งสุดท้าย โดยใช้โดเมนเดิมจาก Part 30 หัวข้อ 30.10 เป๊ะๆ (parser สำหรับไฟล์
config รูปแบบ `key=value` ที่ล้มเหลวได้ 3 แบบ: I/O error, ขาดฟิลด์, ค่าฟิลด์ผิด) — คราวนี้ `ConfigError`/
`AppError` ใช้ `thiserror` (เสมือนเป็น `config_core`) และ `main()` ใช้ `anyhow` พร้อม `.context()` ทุกจุด (เสมือน
เป็น `config_cli`) เขียนไว้ในไฟล์เดียวกันเพื่อให้ทดสอบและอ่านง่าย (แนบ comment กำกับไว้ว่าส่วนไหนเทียบเท่ากับ
crate ไหนในโครงสร้าง workspace ของหัวข้อ 31.15):

```rust
// ============================================================
// config_core: "library crate" (จำลองในไฟล์เดียวด้วย mod) — ใช้ thiserror
// เพราะผู้เรียก (main ด้านล่าง) ต้องแยกกรณี error ได้
// ============================================================
mod config_core {
    use std::collections::HashMap;
    use std::num::ParseIntError;
    use thiserror::Error;

    #[derive(Debug)]
    pub struct AppConfig {
        pub port: u16,
        pub timeout_seconds: u32,
        pub verbose: bool,
    }

    #[derive(Debug, Error)]
    pub enum ConfigError {
        #[error("ไม่พบฟิลด์ที่จำเป็น '{0}' กรุณาเพิ่มบรรทัด '{0}=...' ในไฟล์ config")]
        MissingField(String),

        #[error("ฟิลด์ '{field}' ต้องเป็นตัวเลข: {source}")]
        InvalidNumber {
            field: String,
            #[source]
            source: ParseIntError,
        },

        #[error("ฟิลด์ '{field}' ต้องเป็น true หรือ false แต่พบค่า '{value}'")]
        InvalidBool { field: String, value: String },
    }

    #[derive(Debug, Error)]
    pub enum AppError {
        #[error("ไม่สามารถอ่านไฟล์ config ได้")]
        Io(#[from] std::io::Error),

        #[error("ไฟล์ config มีข้อมูลไม่ถูกต้อง")]
        Config(#[from] ConfigError),
    }

    fn get_field<'a>(map: &'a HashMap<String, String>, field: &str) -> Result<&'a str, ConfigError> {
        map.get(field)
            .map(|s| s.as_str())
            .ok_or_else(|| ConfigError::MissingField(field.to_string()))
    }

    fn parse_lines(content: &str) -> HashMap<String, String> {
        let mut map = HashMap::new();
        for line in content.lines() {
            let line = line.trim();
            if line.is_empty() {
                continue;
            }
            if let Some((key, value)) = line.split_once('=') {
                map.insert(key.trim().to_string(), value.trim().to_string());
            }
        }
        map
    }

    pub fn parse_config(content: &str) -> Result<AppConfig, ConfigError> {
        let map = parse_lines(content);

        let port_raw = get_field(&map, "port")?;
        let port: u16 = port_raw.parse().map_err(|source| ConfigError::InvalidNumber {
            field: "port".to_string(),
            source,
        })?;

        let timeout_raw = get_field(&map, "timeout_seconds")?;
        let timeout_seconds: u32 =
            timeout_raw.parse().map_err(|source| ConfigError::InvalidNumber {
                field: "timeout_seconds".to_string(),
                source,
            })?;

        let verbose_raw = get_field(&map, "verbose")?;
        let verbose = match verbose_raw {
            "true" => true,
            "false" => false,
            other => {
                return Err(ConfigError::InvalidBool {
                    field: "verbose".to_string(),
                    value: other.to_string(),
                })
            }
        };

        Ok(AppConfig { port, timeout_seconds, verbose })
    }

    pub fn load_app_config(path: &str) -> Result<AppConfig, AppError> {
        let content = std::fs::read_to_string(path)?; // io::Error -> AppError ผ่าน #[from]
        let config = parse_config(&content)?; // ConfigError -> AppError ผ่าน #[from]
        Ok(config)
    }
}

// ============================================================
// main(): "application crate" — ใช้ anyhow เพราะแค่ต้อง propagate
// และแสดง error ให้อ่านง่าย ไม่ต้อง match แยกกรณีในชั้นนี้
// ============================================================
use anyhow::{Context, Result};
use config_core::{load_app_config, parse_config};

fn run(path: &str) -> Result<()> {
    let config = load_app_config(path)
        .with_context(|| format!("โหลด config จาก '{path}' ไม่สำเร็จ"))?;
    println!("โหลดสำเร็จ: {config:?}");
    Ok(())
}

fn main() {
    println!("=== กรณีที่ 1: ไฟล์ไม่มีอยู่จริง ===");
    if let Err(e) = run("/nonexistent/path/app.conf") {
        println!("{e:?}");
    }

    println!();
    println!("=== กรณีที่ 2-4: เนื้อหาไฟล์รูปแบบต่างๆ (ทดสอบผ่าน parse_config ตรงๆ) ===");
    let cases = [
        "port=8080\ntimeout_seconds=30\nverbose=true",
        "port=8080\ntimeout_seconds=30",
        "port=abc\ntimeout_seconds=30\nverbose=true",
        "port=8080\ntimeout_seconds=30\nverbose=yes",
    ];

    for case in cases {
        match parse_config(case).context("parse_config ล้มเหลว") {
            Ok(cfg) => println!("โหลดสำเร็จ: {cfg:?}"),
            Err(e) => println!("{e:?}"),
        }
    }
}
```

ผลลัพธ์ (ทดสอบจริงด้วย `cargo run` ครบทุกกรณี):

```
=== กรณีที่ 1: ไฟล์ไม่มีอยู่จริง ===
โหลด config จาก '/nonexistent/path/app.conf' ไม่สำเร็จ

Caused by:
    0: ไม่สามารถอ่านไฟล์ config ได้
    1: No such file or directory (os error 2)

=== กรณีที่ 2-4: เนื้อหาไฟล์รูปแบบต่างๆ (ทดสอบผ่าน parse_config ตรงๆ) ===
โหลดสำเร็จ: AppConfig { port: 8080, timeout_seconds: 30, verbose: true }
parse_config ล้มเหลว

Caused by:
    ไม่พบฟิลด์ที่จำเป็น 'verbose' กรุณาเพิ่มบรรทัด 'verbose=...' ในไฟล์ config
parse_config ล้มเหลว

Caused by:
    0: ฟิลด์ 'port' ต้องเป็นตัวเลข: invalid digit found in string
    1: invalid digit found in string
parse_config ล้มเหลว

Caused by:
    ฟิลด์ 'verbose' ต้องเป็น true หรือ false แต่พบค่า 'yes'
```

**เทียบกับผลลัพธ์ของ Part 30 หัวข้อ 30.10 (ที่ทำทุกอย่างด้วยมือ) โดยตรง**: โครงสร้างของ chain เหมือนกันทุก
ประการ (กรณีที่ 1 มี 2 ชั้น, กรณีที่ 3 มี 3 ชั้นซ้อนกันจาก `parse_config` เอง — สังเกตว่าตัวอย่างนี้เพิ่มชั้น
`.context("parse_config ล้มเหลว")` เข้าไปอีกชั้นบนสุดจาก `anyhow` ทำให้ chain ยาวกว่า Part 30 อีกหนึ่งชั้น
เพราะ Part 30 เรียก `parse_config` ตรงๆ ไม่ผ่าน `anyhow` — นี่คือ "ต้นทุน" เล็กๆ ที่ต้องรู้: การเติม `.context()`
ทุกจุดจะเพิ่มชั้นใน chain แม้บางครั้งชั้นนั้นจะดูซ้ำซ้อนกับ `Display` ของ error เดิม — ควรพิจารณาว่าจุดไหนควร
เพิ่ม context จุดไหนไม่ควร ไม่ใช่เติมทุกจุดโดยอัตโนมัติ) — **สิ่งที่ต่างจาก Part 30 อย่างสิ้นเชิงคือปริมาณโค้ด**:
`config_core` ทั้งโมดูล (error type 2 ตัว + ฟังก์ชัน parse ทั้งหมด) สั้นกว่าเวอร์ชันมือของ Part 30 อย่างเห็นได้ชัด
เฉพาะส่วน error type (24 บรรทัดเทียบกับ 68 บรรทัดตามที่วัดไว้ในหัวข้อ 31.7) และ `main()` **ไม่ต้องเขียน
`print_error_chain` เองเลยแม้แต่บรรทัดเดียว** — แค่ format ด้วย `{:?}` ก็ได้ chain แบบเต็มมาให้ทันที

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `#[from]` ซ้ำสองครั้งกับ Error Type เดียวกันในสอง Variant (E0119)

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum ConfigError {
    #[error("io error 1: {0}")]
    IoA(#[from] std::io::Error),

    #[error("io error 2: {0}")]
    IoB(#[from] std::io::Error),
}

fn main() {}
```

Error ที่ได้ (ทดสอบจริง):

```
error[E0119]: conflicting implementations of trait `From<std::io::Error>` for type `ConfigError`
 --> src/main.rs:9:11
  |
6 |     IoA(#[from] std::io::Error),
  |           ---- first implementation here
...
9 |     IoB(#[from] std::io::Error),
  |           ^^^^ conflicting implementation for `ConfigError`
```

**สาเหตุ**: ตามหัวข้อ 31.4 `#[from]` generate `impl From<T> for ConfigError` ให้ — ถ้าสอง variant ต่างมี
`#[from]` field ชนิดเดียวกัน (`std::io::Error` ทั้งคู่) `thiserror` จะพยายาม generate `impl From<std::io::Error>
for ConfigError` **สองครั้ง** ซึ่งขัดกับกฎของ Rust ที่ห้าม implement trait ตัวเดียวกันให้ type เดียวกันซ้ำสองรอบ
(กฎเดียวกันจะขัดแย้งถ้าคุณเขียน `impl From` ด้วยมือซ้ำสองบล็อกเช่นกัน — `#[from]` ไม่ได้เพิ่มข้อจำกัดใหม่ แค่ทำให้
เจอ error นี้ได้เร็วขึ้นเพราะเขียนสั้นลงจนอาจมองข้ามได้ง่าย) **วิธีแก้**: ให้แค่ variant เดียวมี `#[from]` สำหรับ
error type นั้น ส่วน variant อื่นที่ต้องการ error ชนิดเดียวกันแต่บริบทต่างกัน ให้ใช้ `#[source]` เฉยๆ (ไม่มี
`#[from]`) แล้วสร้างด้วย `.map_err()` ตรงจุดที่ต้องการแยกบริบทแทน (ตามที่หัวข้อ 31.5 สาธิตไว้)

### 2. `#[derive(thiserror::Error)]` โดยไม่มี `#[derive(Debug)]` — Supertrait Bound เดิมจาก Part 30 กลับมาอีกครั้ง (E0277)

```rust
use thiserror::Error;

#[derive(Error)] // ลืม Debug!
enum ConfigError {
    #[error("ไม่พบฟิลด์ '{0}'")]
    MissingField(String),
}

fn main() {}
```

Error ที่ได้ (ทดสอบจริง):

```
error[E0277]: `ConfigError` doesn't implement `Debug`
 --> src/main.rs:4:6
  |
3 | #[derive(Error)]
  |          ----- in this derive macro expansion
4 | enum ConfigError {
  |      ^^^^^^^^^^^ the trait `Debug` is not implemented for `ConfigError`
  |
  = note: add `#[derive(Debug)]` to `ConfigError` or manually `impl Debug for ConfigError`
note: required by a bound in `std::error::Error`
help: consider annotating `ConfigError` with `#[derive(Debug)]`
```

**สาเหตุ**: `#[derive(thiserror::Error)]` generate `impl std::error::Error for ConfigError` ให้ ซึ่งยังต้องผ่าน
supertrait requirement เดียวกันกับที่ Part 30 หัวข้อ 30.2 อธิบายไว้ (`Error: Debug + Display`) — `thiserror`
generate `Display` ให้จาก `#[error(...)]` แต่ **ไม่ generate `Debug` ให้** (เพราะ `Debug` มี `#[derive]` ของ
standard library อยู่แล้ว ไม่มีเหตุผลให้ทำงานซ้ำ) การใช้ `thiserror` **ไม่ได้ยกเลิกกฎ supertrait ของ Part 30
เลยแม้แต่นิดเดียว** มันแค่ช่วยเขียนฝั่ง `Display`/`Error` ให้เท่านั้น **วิธีแก้**: เติม `#[derive(Debug, Error)]`
เสมอ (สังเกตลำดับ: `Debug` ต้องมาก่อนหรือหลัง `Error` ก็ได้ใน `#[derive(...)]` เดียวกัน แต่ต้องมีทั้งคู่)

### 3. เรียก `.context()`/`.with_context()` โดยไม่ `use anyhow::Context` (E0599)

```rust
fn read_file() -> anyhow::Result<String> {
    let content = std::fs::read_to_string("app.conf")
        .context("อ่านไฟล์ app.conf ไม่สำเร็จ")?;
    Ok(content)
}

fn main() {
    println!("{:?}", read_file());
}
```

Error ที่ได้ (ทดสอบจริง):

```
error[E0599]: no method named `context` found for enum `Result<T, E>` in the current scope
 --> src/main.rs:3:10
  |
2 |       let content = std::fs::read_to_string("app.conf")
  |  ___________________-
3 | |         .context("อ่านไฟล์ app.conf ไม่สำเร็จ")?;
  | |_________-^^^^^^^
  |
  = help: items from traits can only be used if the trait is in scope
help: trait `Context` which provides `context` is implemented but not in scope; perhaps you want to import it
  |
1 + use anyhow::Context;
  |
```

**สาเหตุ**: `.context()`/`.with_context()` **ไม่ใช่ method ของ `Result<T, E>` โดยตรง** — มันเป็น method ของ
**trait `anyhow::Context`** ที่ `anyhow` implement ไว้ให้ `Result<T, E>` ทุกชนิด (`E: std::error::Error + Send +
Sync + 'static`) และ `Option<T>` — ตามหลักการที่ Rust บังคับว่า **method จาก trait จะเรียกได้ก็ต่อเมื่อ trait
นั้นอยู่ใน scope** (`use` ไว้แล้ว) เท่านั้น เหมือนกับที่ต้อง `use std::io::Read;` ก่อนเรียก `.read()` บน
`TcpStream` ตามหลักการที่เรียนมาตั้งแต่บทเรื่อง trait — compiler บอกวิธีแก้ไว้ตรงๆ ในบรรทัด help ท้ายสุด
**วิธีแก้**: เติม `use anyhow::Context;` ไว้ที่หัวไฟล์เสมอเมื่อจะใช้ `.context()`/`.with_context()` (มักเขียนรวม
กับ `Result` เป็น `use anyhow::{Context, Result};` ในบรรทัดเดียวตามที่ตัวอย่างในบทนี้ทำ)

### 4. ใช้ `?` กับ `anyhow::Error` ในฟังก์ชันที่คืน Error Type ของตัวเอง (E0277)

```rust
#[derive(Debug)]
struct MyOwnResultType;

fn load() -> Result<u32, MyOwnResultType> {
    let raw = "abc";
    let n: u32 = raw.parse().map_err(|e| anyhow::anyhow!("parse failed: {e}"))?;
    Ok(n)
}

fn main() {
    println!("{:?}", load());
}
```

Error ที่ได้ (ทดสอบจริง):

```
error[E0277]: `?` couldn't convert the error to `MyOwnResultType`
 --> src/main.rs:7:79
  |
4 | fn load() -> Result<u32, MyOwnResultType> {
  |              ---------------------------- expected `MyOwnResultType` because of this
...
7 |     let n: u32 = raw.parse().map_err(|e| anyhow::anyhow!("parse failed: {e}"))?;
  |                      ------- this has type `Result<_, _>`                     ^ the trait `From<anyhow::Error>` is not implemented for `MyOwnResultType`
  |
note: `MyOwnResultType` needs to implement `From<anyhow::Error>`
  = note: the question mark operation (`?`) implicitly performs a conversion on the error value using the `From` trait
```

**สาเหตุ**: กฎของ `?` ที่ Part 30 หัวข้อ 30.4 อธิบายไว้ (compiler ต้องหา `impl From<ErrorTypeต้นทาง> for
ErrorTypeปลายทาง` เสมอ) **ยังใช้กับ `anyhow::Error` เหมือนกับ error type อื่นๆ ทุกประการ** — `anyhow::Error`
ไม่ใช่ "ข้อยกเว้น" ของกฎนี้ มันแค่มี blanket `From` ที่ครอบคลุมกว้างมาก (หัวข้อ 31.10) แต่นั่นคือ `impl From<E> for
anyhow::Error` (ทิศทางเข้า `anyhow::Error`) **ไม่ใช่ทิศทางออกจาก `anyhow::Error`ไปเป็น type อื่น** — ถ้าฟังก์ชัน
คืน error type ของตัวเอง (`MyOwnResultType`) ที่ไม่มี `impl From<anyhow::Error>` ให้ `?` ก็ยังพังเหมือนสถานการณ์
ปกติทุกประการ **วิธีแก้**: ให้ฟังก์ชันที่ใช้ `anyhow` ภายในคืน `anyhow::Result<T>` ตรงๆ (แนวทางที่ถูกต้องตามหลัก
การหัวข้อ 31.8 — ผสมปรัชญาทั้งสองในฟังก์ชันเดียวกันมักเป็นสัญญาณว่าออกแบบผิดที่) หรือถ้าจำเป็นต้องคืน
`MyOwnResultType` จริงๆ ให้แปลงด้วย `.map_err(|e| MyOwnResultType::from_anyhow(e))` (เขียน conversion เองแบบ
เจาะจง) แทนที่จะพึ่ง `?` ตรงๆ

### 5. `#[error(transparent)]` กับ Variant ที่มีมากกว่า 1 Field

```rust
use thiserror::Error;

#[derive(Debug, Error)]
enum AppError {
    #[error(transparent)]
    Config(std::io::Error, String),
}

fn main() {}
```

Error ที่ได้ (ทดสอบจริง):

```
error: #[error(transparent)] requires exactly one field
 --> src/main.rs:5:5
  |
5 | /     #[error(transparent)]
6 | |     Config(std::io::Error, String),
  | |__________________________________^
```

**สาเหตุ**: ตามหัวข้อ 31.6 `#[error(transparent)]` มีหน้าที่ "ส่งต่อ `Display`/`source()` ไปยัง field ที่ห่อไว้
ตรงๆ" — การส่งต่อแบบนี้ทำได้ก็ต่อเมื่อ**มี field เดียวให้ส่งต่อไปหา** ถ้ามีสอง field ขึ้นไป `thiserror` ไม่มีทาง
รู้ว่าจะ "ส่งต่อ" ไปที่ field ไหน (ไม่มีความกำกวมแบบ `#[error("...")]` ที่ระบุตำแหน่ง/ชื่อ field ชัดเจนได้)
**วิธีแก้**: ถ้า variant นั้นต้องมีข้อมูลมากกว่า 1 field จริงๆ ให้เปลี่ยนไปใช้ `#[error("...")]` แบบมีข้อความ
ปกติแทน (แล้วอ้าง field ที่ต้องการด้วย `{0}`/`{1}` หรือชื่อ field) — `#[error(transparent)]` เหมาะกับ
variant ที่เป็น "wrapper บริสุทธิ์" ของ error เดียวเท่านั้นตามที่อธิบายไว้ในหัวข้อ 31.6

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เอา `ConfigError` เวอร์ชันพื้นฐานจากหัวข้อ 31.3 (`MissingField`, `InvalidNumber`, `FileNotFound`)
   มาเพิ่ม variant ใหม่ `EmptyValue(String)` เหมือนกับที่ Part 30 แบบฝึกหัดข้อ 1 ให้ทำ (สำหรับฟิลด์ที่มีอยู่ใน
   ไฟล์แต่ค่าเป็น string ว่าง) — เขียนแค่ `#[error("...")]` บรรทัดเดียวสำหรับ variant ใหม่ (ตามหลักการเขียน
   `Display` ที่ดีจาก Part 30 หัวข้อ 30.8) แล้วเทียบดูว่าคุณต้องแก้โค้ดกี่บรรทัดเทียบกับตอนที่ทำแบบฝึกหัดข้อนี้
   ด้วยมือใน Part 30 (hint: เพราะยังไม่ใส่ `#[non_exhaustive]` การเพิ่ม variant จะทำให้ `match` เดิมที่ไม่มี
   `_ =>` compile ไม่ผ่านเหมือนเดิมทุกประการ — `thiserror` ไม่เปลี่ยนพฤติกรรมนี้เลย)

2. **[กลาง]** ต่อยอด `ConfigError`/`AppError` จากหัวข้อ 31.7 ให้เพิ่ม variant ที่สามใน `AppError` ชื่อ
   `Env(#[from] std::env::VarError)` (สำหรับความล้มเหลวจากการอ่าน environment variable) จากนั้นเขียนฟังก์ชัน
   `load_port_from_env(key: &str) -> Result<u16, AppError>` ที่เรียก `std::env::var(key)?` แล้ว `.parse()?`
   (ต้องเพิ่ม variant ที่ห่อ `ParseIntError` ด้วยถ้ายังไม่มี หรือ reuse `ConfigError::InvalidNumber` ที่มีอยู่
   ผ่าน `.map_err()`) ทดสอบด้วยทั้งกรณี environment variable ไม่มีอยู่ และกรณีมีอยู่แต่ค่าไม่ใช่ตัวเลข พร้อมพิมพ์
   error chain ด้วย `{:?}` ถ้าใช้ `anyhow::Error` ห่อผลลัพธ์อีกชั้นในฟังก์ชันที่เรียกใช้ (hint: สังเกตว่า
   `#[from]` ทำงานกับ error type จาก standard library อย่าง `VarError` ได้เหมือนกับ `io::Error` ทุกประการ ไม่มี
   ข้อจำกัดพิเศษอะไรเพิ่ม)

3. **[ยาก]** สร้างระบบอ่านไฟล์สต๊อกสินค้าแบบเดียวกับ Part 30 แบบฝึกหัดข้อ 3 (บรรทัด `sku,quantity` คั่นด้วย
   comma) ใหม่ทั้งหมด แต่คราวนี้แบ่งงานตามหลักการหัวข้อ 31.15: เขียน `inventory_core` (จำลองเป็น `mod` เดียวใน
   ไฟล์เดียวก็ได้ เหมือนหัวข้อ 31.16) ที่มี `InventoryError` เขียนด้วย `thiserror` ห่อ error ได้อย่างน้อย 2
   แหล่ง (I/O + validation ของตัวเอง เช่น `sku` ว่าง, `quantity` parse ไม่ได้หรือติดลบ) จากนั้นเขียน `main()`
   ที่ใช้ `anyhow` เรียก `inventory_core` พร้อม `.context()` อย่างน้อย 2 ชั้น ทดสอบด้วยอย่างน้อย 3 กรณีที่ล้มเหลว
   คนละแบบ แล้วเขียนสรุปเทียบสั้นๆ (3-5 บรรทัด) ว่าโค้ดชุดนี้สั้นลงกว่าเวอร์ชันที่ทำด้วยมือทั้งหมดใน Part 30 มาก
   น้อยแค่ไหน (hint: นับบรรทัดเฉพาะส่วน error type/`Display`/`Error`/`From` เหมือนที่หัวข้อ 31.7 ทำเพื่อเทียบให้
   ยุติธรรม อย่านับฟังก์ชัน parse ที่เหมือนกันทั้งสองเวอร์ชัน)

4. **[ยาก/ประยุกต์ใช้งานจริง]** สร้าง workspace จริง 2 package ตามโครงร่างในหัวข้อ 31.15 (ไม่ใช่จำลองในไฟล์
   เดียวแบบหัวข้อ 31.16) คือ `<ชื่อโดเมนของคุณ>_core` (ใช้ `thiserror`) และ `<ชื่อโดเมนของคุณ>_cli` (ใช้
   `anyhow`, depend on core ผ่าน path dependency) แล้วเพิ่มความสามารถ downcast แบบหัวข้อ 31.14 ใน `main()` ของ
   `_cli`: เมื่อเจอ error variant ที่เจาะจงหนึ่งตัว (เช่น "ไม่พบไฟล์ config") ให้ "กู้คืน" ด้วยการสร้างไฟล์
   default ให้อัตโนมัติแล้วลองโหลดใหม่อีกครั้ง ส่วน error variant อื่นให้พิมพ์ chain เต็มด้วย `{:?}` แล้วจบ
   โปรแกรมด้วย exit code ที่ไม่ใช่ 0 (hint: ใช้ `std::process::exit(1)` หลัง print หรือคืน `Err(...)` จาก
   `fn main() -> anyhow::Result<()>` ตรงๆ ตามที่หัวข้อ 31.12 อธิบายพฤติกรรมของ `main()` ที่คืน `anyhow::Result`
   ไว้ — ทดสอบให้แน่ใจว่า `cargo build` ที่ root workspace compile ทั้งสอง package พร้อมกันสำเร็จ ตามกลไก
   Cargo.lock ร่วมกันที่ Part 17 หัวข้อ 17.8 อธิบายไว้)

## สรุป

การเดินทางเรื่อง error handling ของหลักสูตรนี้เริ่มจาก **Part 12** ที่วาง foundation ไว้ (`Result<T, E>` ในฐานะ
enum ธรรมดา, `?` operator, `Box<dyn Error>` ในฐานะทางลัด pragmatic) ต่อด้วย **Part 30** ที่พาไปลึกที่สุดด้วยการ
**เขียนทุกอย่างด้วยมือ** — custom error enum, `impl Display`, `impl std::error::Error` พร้อม `source()`,
`impl From` หลายตัว, wrapper enum ที่ห่อ error จากหลายแหล่ง, `#[non_exhaustive]` — เพื่อให้เข้าใจอย่างถ่องแท้ว่า
**กลไกทุกชิ้นทำงานอย่างไรและทำไมต้องมีแต่ละชิ้น** และ **Part 31 (บทนี้)** ปิดวงจรด้วยการแสดงให้เห็นว่า
`thiserror`/`anyhow` **ไม่ได้เปลี่ยนกลไกพื้นฐานเหล่านั้นเลยแม้แต่นิดเดียว** มันแค่ **generate โค้ดที่เหมือนกับที่
เขียนด้วยมือใน Part 30 ให้อัตโนมัติจาก attribute สั้นๆ** (`thiserror`) หรือให้ **type สำเร็จรูปที่ ergonomic
กว่า `Box<dyn Error>`** สำหรับโค้ดระดับ application (`anyhow`)

สิ่งที่ควรติดตัวไปจากบทนี้มากที่สุดคือ**กรอบการตัดสินใจ**ในหัวข้อ 31.15: **`thiserror` สำหรับ library ที่ผู้
เรียกต้องแยกกรณี error, `anyhow` สำหรับ application/binary ที่แค่ต้อง propagate และแสดงผลให้อ่านง่าย** — และ
ความจริงที่ว่าโปรเจกต์จริงจำนวนมาก**ใช้ทั้งสองพร้อมกัน**ใน workspace เดียวกันได้อย่างไร้รอยต่อ เพราะ error type
ที่ `thiserror` generate ให้นั้น "ถูกต้องตามหลักการ" ของ `std::error::Error` เต็มรูปแบบ (ตามที่ Part 30 สอนไว้)
จนทำให้ `anyhow` ที่ฝั่ง binary รับมันไปใช้ได้ทันทีโดยไม่ต้องเขียน conversion อะไรเพิ่มเลย

จากนี้ไปเมื่อคุณต้องออกแบบ error handling ในโปรเจกต์จริง คุณมีเครื่องมือครบทั้งสามระดับ: **hand-rolled enum**
(เมื่อไม่อยากมี dependency เพิ่มหรือต้องการควบคุมทุกรายละเอียดเอง), **`thiserror`** (เมื่อต้องการโครงสร้างแบบ
hand-rolled แต่เขียนสั้นกว่ามาก), และ **`anyhow`** (เมื่อแค่ต้องการให้ error ลอยขึ้นมาถึงจุดแสดงผลอย่างสวยงาม) —
และที่สำคัญกว่าเครื่องมือทั้งสามคือ **ความเข้าใจว่าทำไมแต่ละตัวถึงเหมาะกับสถานการณ์ต่างกัน** ซึ่งเป็นสิ่งที่ไม่มี
crate ไหนสอนให้ได้ นอกจากการเข้าใจกลไกพื้นฐานที่ Part 30 ปูทางไว้อย่างละเอียดที่สุดแล้วเท่านั้น

---

**Part ก่อนหน้า:** [Error Handling ขั้นสูง](part-030-error-handling-advanced.md) | **Part ถัดไป:** [Testing: Unit Tests](part-032-testing-unit-tests.md)
