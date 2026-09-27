# Part 107: Capstone: Building a Production-Grade CLI Tool

> โมดูล: Capstone Project | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ออกแบบ "สเปคของ CLI" (command surface: subcommand, flag, positional argument, ค่า default, exit code) **ก่อน**เขียนโค้ดสักบรรทัด แบบเดียวกับที่ทีมวิศวกรรมจริงทำก่อนเริ่ม implement เครื่องมือ command-line ที่จะมีคนอื่นใช้งานต่อ
- ประกอบร่างเครื่องมือ CLI ระดับ production เต็มรูปแบบด้วย `clap` (derive API, subcommand, global argument, `ValueEnum`) ต่อยอดจาก Part 59 ให้ลึกกว่าตัวอย่างเดี่ยว ๆ ที่เคยเห็น — รวม subcommand หลายตัวที่ใช้ argument ร่วมกันจริงในโปรแกรมเดียว
- ออกแบบ `AppError` ด้วย `thiserror` ที่แปลง error ทุกแหล่ง (I/O, `serde_json`, `toml`, business logic) ให้เป็น error message ที่ผู้ใช้แก้ไขได้ทันที พร้อม**แม็ป error แต่ละชนิดไปเป็น exit code ตามธรรมเนียม `sysexits.h`** อย่างสม่ำเสมอทั้งโปรแกรม
- เขียน parser ที่ทนทานต่อข้อมูลเสีย (malformed input) — ไฟล์ log ขนาดใหญ่ที่มีบรรทัดพังปนอยู่ 1 บรรทัดจะไม่ทำให้โปรแกรมทั้งตัวล้ม พร้อมโหมด `--strict` สำหรับ workflow ที่ต้องการความเข้มงวดกว่านั้น และแสดง progress bar ด้วย `indicatif` ระหว่างประมวลผลไฟล์ใหญ่
- รองรับทั้ง **CLI flag** และ **config file** (`~/.config/logcli/config.toml`) พร้อมลำดับความสำคัญที่ชัดเจน (CLI > config file > ค่า default ในตัวโปรแกรม) และผลิต output ได้สองแบบ — อ่านง่ายมีสีบน terminal (เคารพ `NO_COLOR` และตรวจจับ non-tty) กับ JSON สำหรับต่อท่อไปเครื่องมืออื่น
- เขียน integration test เต็มรูปแบบด้วย `assert_cmd`/`predicates` ที่รัน binary จริงแล้วตรวจ stdout/stderr/exit code เหมือนผู้ใช้จริง และเตรียม binary สำหรับแจกจ่ายจริง (release profile ที่ปรับขนาด, static linking, และ crates.io)

## ความรู้ที่ต้องมีมาก่อน

- **Part 59 (CLI Applications ด้วย clap)** — นี่คือ prerequisite ที่สำคัญที่สุดของบทนี้ บทนี้จะไม่สอน derive API ของ `clap` ซ้ำจากศูนย์ (`#[derive(Parser)]`, `#[derive(Subcommand)]`, `value_parser`, `#[arg(env = "...")]`) แต่จะใช้ความเข้าใจนั้น**ต่อยอด**ทันทีเข้าสู่โปรแกรมที่ใหญ่และสมบูรณ์กว่า ถ้า Part 59 ยังไม่แน่น (โดยเฉพาะเรื่อง type-directed parsing และ `#[derive(Subcommand)]`) ให้กลับไปทวนก่อน
- **Part 60 (Logging และ Tracing เบื้องต้น)** — บทนี้ analyze log ที่เขียนในรูปแบบ **structured logging** แบบเดียวกับที่ Part 60 สอน (`tracing_subscriber::fmt().json()` ผลิต log ออกมาเป็น JSON หนึ่ง object ต่อบรรทัด) เครื่องมือที่เราจะสร้างในบทนี้คือ "ผู้บริโภค" (consumer) ของ log รูปแบบนั้นโดยตรง — ถ้าคุณจำได้ว่า Part 60 บอกว่า `tracing` ผลิต log เป็น JSON ได้เพื่อให้ระบบ log aggregation อ่านต่อได้ นี่คือตัวอย่างจริงของ "ระบบปลายทาง" ที่ว่านั้น
- **Part 12 (Result และ Error Handling เบื้องต้น), Part 30-31 (Error Handling ขั้นสูง, thiserror/anyhow)** — `AppError` ในบทนี้คือการประยุกต์ใช้ `thiserror` เต็มรูปแบบในโปรแกรมจริงที่มี error หลายสิบชนิดจากหลายแหล่ง ถ้าเรื่อง `#[error(...)]`, `#[source]`, และ `From`/`Into` ยังไม่คุ้นมือ ควรทวน Part 30-31 ก่อน
- **Part 57-58 (Serde เบื้องต้น/ขั้นสูง)** — `#[serde(flatten)]` ที่ใช้กับ struct `LogEntry` ในบทนี้คือเทคนิคที่ Part 58 สอนไว้สำหรับรับ field ที่ไม่รู้จักล่วงหน้า (field พิเศษที่แต่ละระบบ log ใส่มาไม่เหมือนกัน)
- **Part 32-33 (Testing: Unit Tests, Integration Tests)** — หัวข้อ 107.11 ใช้แนวคิด integration test ที่ Part 33 ปูพื้นไว้ (ทดสอบ binary จากมุมมองผู้ใช้จริง ไม่ใช่เรียกฟังก์ชันภายในตรง ๆ) แต่ใช้ crate เฉพาะสำหรับทดสอบ CLI (`assert_cmd`, `predicates`) ที่ยังไม่ได้แนะนำมาก่อน
- **Part 18-19 (Generics และ Traits)** — ฟังก์ชัน `resolve<T>(...)` ที่ใช้ตัดสิน precedence ระหว่าง CLI/config/default ในหัวข้อ 107.7 เป็นตัวอย่างตรง ๆ ของ "เขียน logic ทั่วไปครั้งเดียว ใช้ได้กับหลาย type" ที่ Part 18-19 สอนไว้

## เนื้อหา

บทนี้เป็น **capstone** — ไม่มีหัวข้อทฤษฎีใหม่ที่ไม่เคยเห็น แต่เป็นการนำทุกเครื่องมือที่เรียนมาแล้วประกอบเป็นโปรแกรมเดียวที่**ใช้งานได้จริง 100%** เราจะสร้างเครื่องมือชื่อ **`logcli`** — ตัววิเคราะห์ log ที่เขียนเป็น **JSON Lines** (ไฟล์ข้อความที่แต่ละบรรทัดเป็น JSON object หนึ่งตัว รูปแบบเดียวกับที่ `tracing_subscriber::fmt().json()` จาก Part 60 ผลิตออกมา) — กรอง (`filter`) และสรุปสถิติ (`stats`) log จำนวนมากได้ พร้อมทุกคุณสมบัติที่เครื่องมือ command-line ระดับ production ต้องมี

ทุกตัวอย่างโค้ดในบทนี้**คือโค้ดจริงที่ compile ผ่านและรันจริงแล้ว** ในระหว่างเตรียมบทเรียนนี้ มีการสร้างโปรเจกต์ Cargo แยกออกมาต่างหาก เขียนโค้ดทั้งหมดที่คุณเห็นในบทนี้ compile ด้วย `cargo build --release`, สร้างไฟล์ log ตัวอย่างจริง (รวมไฟล์ขนาด ~29MB ที่มี 200,000 บรรทัดสำหรับทดสอบ progress bar), รันคำสั่งทุกคำสั่งจริงผ่าน terminal, และรัน test suite จริงจนผ่านครบ — ผลลัพธ์ที่แสดงในบทนี้ทั้งหมดคัดลอกมาจาก output จริงที่ได้ ไม่ใช่ผลลัพธ์ที่เขียนขึ้นเอง

### 107.1 กำหนดสเปคของเครื่องมือก่อนเขียนโค้ด

ก่อนเปิด editor เขียนโค้ดสักบรรทัด ทีมวิศวกรรมที่สร้างเครื่องมือ command-line ระดับ production มักเขียน "สเปค" ของ command surface ไว้ก่อนเสมอ — **เหตุผลเชิงลึก**คือ: การเปลี่ยนชื่อ flag, ลำดับ positional argument, หรือ subcommand หลังจากมีคนเริ่มใช้งานจริงแล้วเป็น **breaking change** ที่ทำให้ script/automation ของผู้ใช้พังทันที (แบบเดียวกับกับดักข้อ 4 ใน Part 59 เรื่องลำดับ field ที่กำหนดลำดับ positional argument) การคิดโครงสร้างให้ตกผลึกก่อนจึงคุ้มค่ากว่าการแก้ไขทีหลังมาก

สเปคของ `logcli` มีดังนี้:

```
logcli — วิเคราะห์ log ที่เขียนเป็น JSON Lines

USAGE:
  logcli [OPTIONS] <COMMAND>

GLOBAL OPTIONS (ใช้ได้กับทุก subcommand เพราะประกาศ global = true):
  --config <PATH>     พาธไฟล์ config (default: ~/.config/logcli/config.toml)
  --color <MODE>       auto | always | never (default: auto)
  --no-progress         ปิด progress bar

COMMANDS:
  filter    กรอง log ตาม level / คำค้น / ช่วงเวลา
  stats     สรุปสถิติ log โดย group ตาม field ที่ระบุ
  completions   พิมพ์ shell completion script ออกทาง stdout

logcli filter [OPTIONS] <FILE>
  <FILE>                พาธไฟล์ log ('-' = อ่านจาก stdin)
  --level <LEVEL>...    กรองเฉพาะ level ที่ระบุ (ใส่ได้หลายครั้ง)
  --contains <TEXT>     กรองเฉพาะบรรทัดที่ message มีคำนี้
  --since <TIME>        เฉพาะ log ตั้งแต่เวลานี้ (RFC 3339)
  --until <TIME>        เฉพาะ log ก่อนเวลานี้ (RFC 3339)
  --limit <N>           จำกัดจำนวนผลลัพธ์
  --format <FORMAT>     human | json (default: human)
  --strict              เจอบรรทัดพัง → หยุดทันที (default: ข้ามแล้วรายงานสรุป)

logcli stats [OPTIONS] <FILE>
  <FILE>                พาธไฟล์ log ('-' = อ่านจาก stdin)
  --by <FIELD>          field ที่ group (default: level)
  --format <FORMAT>     human | json (default: human)
  --strict              เจอบรรทัดพัง → หยุดทันที

logcli completions <SHELL>
  <SHELL>               bash | zsh | fish | elvish | powershell

EXIT CODES (ตามธรรมเนียม sysexits.h):
  0   สำเร็จ
  64  ผู้ใช้ใส่ argument ผิด (USAGE)
  65  ข้อมูล input ผิดรูปแบบ ในโหมด --strict (DATA_ERR)
  66  ไม่พบไฟล์ input (NO_INPUT)
  70  bug ภายในโปรแกรมเอง (SOFTWARE)
  74  I/O ล้มเหลวแบบไม่คาดคิด (IO_ERR)
  78  ไฟล์ config มีอยู่จริงแต่ parse ไม่ผ่าน (CONFIG)
```

สังเกตการตัดสินใจเชิงออกแบบที่ตกลงไว้ **ก่อน**เขียนโค้ดจริงหลายจุด ซึ่งแต่ละจุดจะมีเหตุผลอธิบายละเอียดในหัวข้อที่เกี่ยวข้อง:

1. **`<FILE>` เป็น positional argument ตัวเดียว ไม่ใช่ `--file`** — เพราะทุก subcommand ต้องมีไฟล์ input เสมอ (ไม่มี subcommand ไหนที่ทำงานได้โดยไม่มีไฟล์) การทำเป็น positional required ทำให้คำสั่งสั้นกระชับ (`logcli filter --level error app.log` อ่านง่ายกว่า `logcli filter --level error --file app.log`) และรองรับ `-` เป็นพิเศษสำหรับอ่านจาก stdin (ธรรมเนียม Unix ที่เครื่องมือ `grep`/`cat`/`jq` ทำเหมือนกันหมด ทำให้ `logcli` ต่อท่อกับเครื่องมืออื่นได้ทันที เช่น `tail -f app.log | logcli filter --level error -`)
2. **exit code ใช้ธรรมเนียม `sysexits.h`** (ไม่ใช่แค่ `0`/`1`) — เพื่อให้ script ที่เรียก `logcli` ตัดสินใจแยกแยะสาเหตุความล้มเหลวได้โดยไม่ต้อง parse ข้อความ stderr เอง (เช่น CI script อาจเช็คว่า exit code เป็น 66 แปลว่าไฟล์หาไม่พบ ให้ลอง path อื่น แต่ถ้าเป็น 65 แปลว่าไฟล์เสีย ให้แจ้งเตือนทันที) — รายละเอียดเต็มอยู่ในหัวข้อ 107.5
3. **`--strict` เป็นทางเลือก ไม่ใช่ default** — เพราะพฤติกรรม default ที่เหมาะกับ**กรณีใช้งานที่พบบ่อยที่สุด**ของเครื่องมือนี้ (ไฟล์ log ขนาดใหญ่ที่กำลัง debug incident อยู่) คือ "ทำงานต่อให้ได้มากที่สุด" ไม่ใช่ "หยุดทันทีที่เจอปัญหา" — รายละเอียดเต็มอยู่ในหัวข้อ 107.6
4. **`--format` มีแค่ `human`/`json` สองค่า ไม่ใช่ `String` ที่รับอะไรก็ได้** — ใช้ `enum` + `ValueEnum` (ตามที่กับดักข้อ 5 ใน Part 59 แนะนำไว้) เพื่อให้ `clap` validate ค่าที่ไม่ถูกต้องให้เราฟรี พร้อม error message คุณภาพสูงที่บอก `[possible values: human, json]` เอง

การเขียนสเปคแบบนี้ลงกระดาษ (หรือ comment ในโค้ด) ก่อนเริ่ม ทำให้เราเห็นภาพรวมทั้งหมดในหน้าเดียว และตรวจพบความไม่สมเหตุสมผลได้ก่อนที่จะลงทุนเวลาเขียนโค้ดจริง — เช่นถ้าตอนแรกตั้งใจให้ `--by` ของ `stats` เป็น positional argument ก็จะสังเกตได้ทันทีว่ามันจะไปชนกับ `<FILE>` ที่เป็น positional เหมือนกัน (จะสับสนว่าตัวไหนคือ field ตัวไหนคือไฟล์) จึงตัดสินใจให้ `--by` เป็น named flag ที่มี default แทน

### 107.2 Project Setup: Cargo.toml และโครงสร้างโปรเจกต์

เริ่มจากสร้างโปรเจกต์และเพิ่ม dependency ทั้งหมดที่ต้องใช้:

```bash
cargo new logcli
cd logcli
cargo add clap --features derive
cargo add clap_complete
cargo add serde --features derive
cargo add serde_json
cargo add thiserror
cargo add chrono --features serde
cargo add toml
cargo add dirs
cargo add indicatif
cargo add owo-colors
cargo add is-terminal
cargo add --dev assert_cmd
cargo add --dev predicates
cargo add --dev tempfile
```

`Cargo.toml` ที่ได้ (ปรับ `[profile.release]` เพิ่มเองสำหรับหัวข้อ 107.12):

```toml
[package]
name = "logcli"
version = "0.1.0"
edition = "2021"

[dependencies]
chrono = { version = "0.4", features = ["serde"] }
clap = { version = "4", features = ["derive"] }
clap_complete = "4"
dirs = "5"
indicatif = "0.17"
is-terminal = "0.4"
owo-colors = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
toml = "0.8"

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
tempfile = "3"

[profile.release]
strip = true
lto = true
codegen-units = 1
```

ไล่ดูว่าแต่ละ dependency ทำหน้าที่อะไร (ตัวที่เคยเห็นแล้วใน Part ก่อนหน้าจะพูดสั้น ๆ ตัวที่เจอครั้งแรกในหลักสูตรจะอธิบายละเอียดกว่า):

- **`clap` + `clap_complete`** — parse CLI arguments (Part 59) และ generate shell completion script (ครั้งแรกในหลักสูตรที่ใช้ — หัวข้อ 107.10)
- **`serde` + `serde_json`** — deserialize บรรทัด log แต่ละบรรทัดจาก JSON (Part 57-58)
- **`thiserror`** — สร้าง `AppError` (Part 30-31)
- **`chrono`** — parse/format timestamp แบบ RFC 3339 พร้อมเปรียบเทียบช่วงเวลาได้ตรงไปตรงมา (`DateTime<Utc>` implement `Ord` ให้แล้ว) — เปิด feature `serde` เพื่อให้ `#[derive(Deserialize)]` ใช้กับ field ชนิด `DateTime<Utc>` ได้ตรง ๆ โดยไม่ต้อง parse เองสองรอบ
- **`toml`** — parse ไฟล์ config (`~/.config/logcli/config.toml`) — ครั้งแรกในหลักสูตรที่ใช้ format นี้เพื่อ config file (Cargo เองก็ใช้ TOML เป็น format ของ `Cargo.toml`)
- **`dirs`** — หาพาธ config directory ที่ถูกต้องตาม OS (Linux: `~/.config`, macOS: `~/Library/Application Support`, Windows: `%APPDATA%`) โดยไม่ต้อง hardcode `$HOME` เอง ซึ่งพฤติกรรมต่างกันข้าม platform
- **`indicatif`** — progress bar/spinner ระหว่างประมวลผลไฟล์ใหญ่ (ครั้งแรกในหลักสูตร — หัวข้อ 107.6)
- **`owo-colors` + `is-terminal`** — ใส่สีลง terminal output และตรวจจับว่า output ปลายทางเป็น terminal จริงหรือถูก pipe/redirect (ครั้งแรกในหลักสูตร — หัวข้อ 107.8)
- **`assert_cmd` + `predicates` + `tempfile`** (dev-dependencies) — ทดสอบ binary จริงแบบ end-to-end (ครั้งแรกในหลักสูตร — หัวข้อ 107.11)

โครงสร้างไฟล์ที่เราจะสร้าง (แบ่งเป็นโมดูลตามความรับผิดชอบ — แนวทางเดียวกับที่ Part 34-35 สอนเรื่องการจัดโครงสร้างโปรเจกต์ Rust ที่โตขึ้น):

```
logcli/
├── Cargo.toml
├── src/
│   ├── main.rs      — entry point, dispatch subcommand, ประกอบทุกอย่างเข้าด้วยกัน
│   ├── cli.rs        — struct/enum ของ clap ทั้งหมด (Cli, Command, *Args)
│   ├── error.rs      — AppError + exit code mapping
│   ├── model.rs      — LogEntry (โครงสร้างข้อมูลของ log แต่ละบรรทัด)
│   ├── config.rs     — FileConfig + logic การหา precedence
│   ├── reader.rs     — อ่าน/parse ไฟล์ log แบบทนทาน + progress bar
│   └── output.rs     — สี/การตรวจจับ terminal สำหรับ output
└── tests/
    └── cli.rs        — integration test เต็มรูปแบบ
```

การแบ่งแบบนี้ทำให้แต่ละไฟล์มีความรับผิดชอบเดียวที่ชัดเจน (**Single Responsibility**) — `cli.rs` ไม่รู้เรื่อง business logic เลย มีหน้าที่แค่นิยามรูปร่างของ argument, `reader.rs` ไม่รู้เรื่อง clap เลย มีหน้าที่แค่อ่านไฟล์ให้ทนทาน — ผลคือถ้าอนาคตต้องการเปลี่ยนจาก `clap` ไป argument parser ตัวอื่น (สมมติเกิดเหตุสุดวิสัย) จะแก้แค่ `cli.rs` กับจุดที่เรียกใช้ใน `main.rs` โดยไม่ต้องแตะ `reader.rs`/`model.rs` เลยแม้แต่บรรทัดเดียว

### 107.3 Data Model: `LogEntry` และการรับ field ที่ไม่รู้จักล่วงหน้าด้วย `flatten`

จุดเริ่มต้นของทุกอย่างคือ "หน้าตา" ของ log หนึ่งบรรทัดที่เราจะอ่าน สมมติ log จริงจากระบบ web service ที่ใช้ `tracing_subscriber::fmt().json()` (ตามที่ Part 60 สอนไว้) มีหน้าตาประมาณนี้:

```json
{"timestamp":"2024-01-15T10:23:01Z","level":"info","message":"request handled","endpoint":"/api/users","status":200,"duration_ms":42}
```

สังเกตว่า `timestamp`/`level`/`message` เป็น field ที่ log framework ส่วนใหญ่มีให้เสมอ แต่ `endpoint`/`status`/`duration_ms` เป็น field พิเศษที่**แต่ละระบบใส่มาไม่เหมือนกัน** — ระบบ A อาจมี `user_id`, ระบบ B อาจมี `request_id`, `trace_id` แทน เราไม่มีทาง hardcode field เหล่านี้ทุกตัวไว้ล่วงหน้าได้ในทางปฏิบัติ (และไม่ควรทำด้วย เพราะเครื่องมือจะผูกติดกับ schema ของระบบใดระบบหนึ่งไปเลย)

Part 58 สอนเทคนิคที่ตรงกับปัญหานี้เป๊ะ: **`#[serde(flatten)]`** — เก็บ field ที่ไม่รู้จักล่วงหน้าไว้ใน map แยก:

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use serde_json::Value;
use std::collections::BTreeMap;

/// หนึ่งบรรทัดของ log ในรูปแบบ JSON Lines (หนึ่ง JSON object ต่อหนึ่งบรรทัด)
/// ตรงกับรูปแบบ structured logging ที่ Part 60 สอนไว้ (เช่น `tracing_subscriber::fmt().json()`)
///
/// field `level`/`message`/`timestamp` เป็น field ที่ log จริงจากระบบ tracing/log ส่วนใหญ่มีเสมอ
/// ส่วน field อื่น ๆ ที่ไม่รู้จักล่วงหน้า (เช่น `endpoint`, `status`, `user_id`, `duration_ms`)
/// จะถูกเก็บรวมไว้ใน `fields` ผ่าน `#[serde(flatten)]` — เราไม่รู้ล่วงหน้าว่า log ของแต่ละระบบ
/// จะมี field พิเศษอะไรบ้าง ดังนั้นการ hardcode field ทุกตัวเป็นไปไม่ได้ในทางปฏิบัติ
#[derive(Debug, Clone, Deserialize, Serialize)]
pub struct LogEntry {
    pub timestamp: DateTime<Utc>,
    pub level: String,
    pub message: String,
    #[serde(flatten)]
    pub fields: BTreeMap<String, Value>,
}

impl LogEntry {
    /// เปรียบเทียบ level แบบไม่สนตัวพิมพ์เล็ก/ใหญ่ ("ERROR" กับ "error" ถือว่าตรงกัน)
    /// เพราะ logging backend แต่ละตัวไม่ได้ format level เป็นตัวพิมพ์เดียวกันเสมอไป
    pub fn level_matches(&self, wanted: &str) -> bool {
        self.level.eq_ignore_ascii_case(wanted)
    }

    /// ดึงค่า field ตามชื่อ ไม่ว่าจะเป็น field ที่ประกาศตรง ๆ (level) หรือ field ที่มาจาก flatten
    /// คืนค่าเป็น String แบบอ่านง่าย ใช้สำหรับ `stats --by <field>` ที่รับชื่อ field แบบ dynamic
    pub fn field_value(&self, name: &str) -> Option<String> {
        match name {
            "level" => Some(self.level.clone()),
            "message" => Some(self.message.clone()),
            _ => self.fields.get(name).map(value_to_group_key),
        }
    }
}

/// แปลง serde_json::Value เป็น String สำหรับใช้เป็น "key" ตอน group-by
/// สตริงเก็บ quote ไว้ตรง ๆ (ไม่ผ่าน quote) ส่วน type อื่นใช้ Display ปกติของ serde_json
fn value_to_group_key(v: &Value) -> String {
    match v {
        Value::String(s) => s.clone(),
        other => other.to_string(),
    }
}
```

ไล่ดูจุดสำคัญทีละส่วน:

- **`timestamp: DateTime<Utc>`** — เพราะเปิด feature `serde` ของ `chrono` ไว้แล้ว `chrono` implement `Deserialize` ให้ `DateTime<Utc>` มาเอง โดย parse จาก string RFC 3339 ให้อัตโนมัติ (ไม่ต้องเขียน `#[serde(deserialize_with = ...)]` เอง) — และเพราะ `DateTime<Utc>` implement `Ord` เราจึงเปรียบเทียบ `>`/`<` ระหว่าง timestamp ได้ตรง ๆ ในหัวข้อ 107.9 ตอนกรองด้วย `--since`/`--until` โดยไม่ต้องแปลงเป็น string มาเทียบกันเอง (ซึ่งเสี่ยง bug ถ้า format ไม่ตรงกันเป๊ะ)
- **`#[derive(..., Serialize)]`** — เรา derive `Serialize` ด้วย (ไม่ใช่แค่ `Deserialize`) เพราะโหมด `--format json` ในหัวข้อ 107.9 ต้อง serialize `LogEntry` กลับเป็น JSON เพื่อพิมพ์ออกไป — เชื่อมกับแนวคิด Part 57 ที่บอกว่า struct หนึ่งตัวสามารถ derive ทั้งสอง trait พร้อมกันได้ ทำให้ round-trip (อ่านเข้ามาแล้วเขียนออกไปในรูปแบบเดียวกัน) ทำได้โดยไม่ต้องเขียน struct แยกสองตัว
- **`fields: BTreeMap<String, Value>` พร้อม `#[serde(flatten)]`** — บอก serde ว่า "field ไหนที่ไม่ตรงกับ `timestamp`/`level`/`message` ให้ยัดรวมไว้ใน map นี้แทน" ใช้ `BTreeMap` (ไม่ใช่ `HashMap`) โดยตั้งใจ เพราะเราต้องการให้**ลำดับของ field คงที่**ทุกครั้งที่ print ออกมา (เชื่อมกับ Part 15 เรื่อง `BTreeMap` ที่เรียงตาม key เสมอ) — ถ้าใช้ `HashMap` ลำดับ field ใน output ของโหมด human-readable จะสุ่มไปมาทุกครั้งที่รันโปรแกรม (แม้ข้อมูล input เหมือนกันเป๊ะ) ซึ่งทำให้ diff ผลลัพธ์ระหว่างสองครั้งรันยากขึ้นโดยไม่จำเป็น — เป็นรายละเอียดเล็ก ๆ ที่ส่งผลกับ "ความน่าเชื่อถือ" ของเครื่องมือ debugging (ผู้ใช้คาดหวังว่า output เดียวกันควรมาในลำดับเดียวกันเสมอ)
- **`field_value(&self, name: &str)`** — เมธอดนี้คือสิ่งที่ทำให้ `stats --by <field ใดก็ได้>` ทำงานได้แบบ dynamic โดยไม่ต้องรู้ล่วงหน้าว่า field ชื่ออะไร — ถ้าชื่อ field ตรงกับ `level`/`message` ที่ประกาศตรง ๆ ให้ดึงจาก field นั้นตรง ๆ ไม่งั้นค้นหาใน `fields` (map ที่มาจาก flatten) — นี่คือจุดที่ "field ที่ไม่รู้จักล่วงหน้า" ของ flatten มาบรรจบกับ "field ที่รู้จักแน่ ๆ" ของ struct ทั่วไป ในฟังก์ชันเดียว

ทดสอบว่า deserialize ทำงานถูกต้องด้วยตัวอย่าง log จริง (รันผ่านแล้ว):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_extra_fields_into_flatten_map() {
        let raw = r#"{"timestamp":"2024-01-15T10:23:01Z","level":"info","message":"request handled","endpoint":"/api/users","status":200}"#;
        let entry: LogEntry = serde_json::from_str(raw).unwrap();

        assert_eq!(entry.level, "info");
        assert_eq!(entry.message, "request handled");
        assert_eq!(entry.field_value("endpoint"), Some("/api/users".to_string()));
        assert_eq!(entry.field_value("status"), Some("200".to_string()));
        assert_eq!(entry.field_value("does_not_exist"), None);
    }
}
```

### 107.4 CLI Surface ด้วย `clap`: `Cli`, `Command`, Global Arguments

ตอนนี้มาแปลงสเปคจากหัวข้อ 107.1 ให้เป็นโค้ด `clap` จริง — ทั้งหมดอยู่ใน `src/cli.rs`:

```rust
use clap::{Parser, Subcommand, ValueEnum};
use std::path::PathBuf;

/// logcli: วิเคราะห์ log ที่เขียนเป็น JSON Lines (หนึ่ง JSON object ต่อบรรทัด)
///
/// อ่าน log แล้ว filter/สรุปสถิติได้ ทั้งแบบอ่านง่ายบน terminal และแบบ JSON
/// สำหรับต่อท่อ (pipe) ไปยังเครื่องมืออื่นต่อได้ (เช่น jq)
#[derive(Parser, Debug)]
#[command(name = "logcli", version, about, long_about = None)]
pub struct Cli {
    #[command(subcommand)]
    pub command: Command,

    /// พาธของไฟล์ config (default: ~/.config/logcli/config.toml — ถ้าไม่มีไฟล์นี้ใช้ค่า default ในตัว)
    #[arg(long, global = true, value_name = "PATH")]
    pub config: Option<PathBuf>,

    /// ควบคุมการใช้สี: auto (ตรวจจาก terminal อัตโนมัติ), always, never
    #[arg(long, global = true, value_enum, value_name = "MODE")]
    pub color: Option<ColorMode>,

    /// ปิด progress bar แม้ไฟล์จะใหญ่ก็ตาม (มีผลเหมือนตั้ง progress = false ใน config file)
    #[arg(long, global = true)]
    pub no_progress: bool,
}

#[derive(Subcommand, Debug)]
pub enum Command {
    /// กรอง log ตาม level / คำค้น / ช่วงเวลา
    Filter(FilterArgs),
    /// สรุปสถิติ log โดย group ตาม field ที่ระบุ
    Stats(StatsArgs),
    /// พิมพ์ shell completion script ออกทาง stdout
    Completions(CompletionsArgs),
}

#[derive(clap::Args, Debug)]
pub struct FilterArgs {
    /// พาธของไฟล์ log (ใช้ '-' เพื่ออ่านจาก stdin แทน)
    #[arg(value_name = "FILE")]
    pub file: PathBuf,

    /// กรองเฉพาะ log level ที่ระบุ (ใส่ได้หลายตัว เช่น --level error --level warn)
    #[arg(long, value_name = "LEVEL")]
    pub level: Vec<String>,

    /// กรองเฉพาะบรรทัดที่ message มีคำนี้อยู่ (case-sensitive)
    #[arg(long, value_name = "TEXT")]
    pub contains: Option<String>,

    /// แสดงเฉพาะ log ที่เกิดตั้งแต่เวลานี้เป็นต้นไป (RFC 3339 เช่น 2024-01-15T00:00:00Z)
    #[arg(long, value_name = "TIME")]
    pub since: Option<String>,

    /// แสดงเฉพาะ log ที่เกิดก่อนเวลานี้ (RFC 3339)
    #[arg(long, value_name = "TIME")]
    pub until: Option<String>,

    /// จำกัดจำนวนผลลัพธ์สูงสุดที่แสดง
    #[arg(long, value_name = "N")]
    pub limit: Option<usize>,

    /// รูปแบบผลลัพธ์: human (อ่านง่าย มีสี) หรือ json (JSON Lines สำหรับต่อท่อ)
    #[arg(long, value_enum, value_name = "FORMAT")]
    pub format: Option<OutputFormat>,

    /// ถ้า log line ไหน parse ไม่ผ่าน ให้หยุดทำงานทั้งโปรแกรมทันที (exit code 65)
    /// แทนพฤติกรรม default ที่ข้ามบรรทัดเสียแล้วรายงานสรุปท้ายสุด
    #[arg(long)]
    pub strict: bool,
}

#[derive(clap::Args, Debug)]
pub struct StatsArgs {
    /// พาธของไฟล์ log (ใช้ '-' เพื่ออ่านจาก stdin แทน)
    #[arg(value_name = "FILE")]
    pub file: PathBuf,

    /// field ที่จะใช้ group log เข้าด้วยกัน (เช่น level, endpoint หรือชื่อ field อื่นใน log)
    #[arg(long, default_value = "level", value_name = "FIELD")]
    pub by: String,

    /// รูปแบบผลลัพธ์: human (ตารางอ่านง่าย) หรือ json (object เดียว {key: count})
    #[arg(long, value_enum, value_name = "FORMAT")]
    pub format: Option<OutputFormat>,

    /// ถ้า log line ไหน parse ไม่ผ่าน ให้หยุดทำงานทั้งโปรแกรมทันที (exit code 65)
    #[arg(long)]
    pub strict: bool,
}

#[derive(clap::Args, Debug)]
pub struct CompletionsArgs {
    /// ชื่อ shell ที่ต้องการ generate completion script ให้
    #[arg(value_enum)]
    pub shell: clap_complete::Shell,
}

#[derive(ValueEnum, Debug, Clone, Copy, PartialEq, Eq)]
pub enum OutputFormat {
    Human,
    Json,
}

#[derive(ValueEnum, Debug, Clone, Copy, PartialEq, Eq)]
pub enum ColorMode {
    Auto,
    Always,
    Never,
}
```

จุดที่ควรอธิบายเพิ่มจากที่ Part 59 ยังไม่ได้ลงรายละเอียด:

**`Command::Filter(FilterArgs)` — enum variant ที่เป็น tuple variant ห่อ struct แยก ไม่ใช่ struct variant ที่มี field ตรง ๆ** ต่างจากตัวอย่าง `taskcli` ใน Part 59 ที่ให้ field ของแต่ละ subcommand อยู่ตรงในตัว variant เอง (`Add { title: String, ... }`) บทนี้เลือกแยก argument ของแต่ละ subcommand ไปเป็น struct ของตัวเอง (`FilterArgs`, `StatsArgs`) แล้วห่อด้วย tuple variant แทน — เหตุผลคือ `FilterArgs`/`StatsArgs` มี field มากถึง 6-7 ตัว การแยกเป็น struct ทำให้**ส่งต่อ argument ทั้งกลุ่มเป็นค่าเดียว**ไปยังฟังก์ชันที่ประมวลผลได้สะดวก (`fn run_filter(args: FilterArgs, ...)`) แทนที่จะต้อง destructure field ทั้ง 7 ตัวออกมาตรงจุด `match` เหมือนตัวอย่างสั้น ๆ ใน Part 59 — ยิ่ง subcommand มี field มากเท่าไหร่ การแยก struct แบบนี้ยิ่งคุ้มค่า

**`#[arg(long, global = true)]` บน field ของ `Cli` (ไม่ใช่ของ `FilterArgs`/`StatsArgs`)** — `global = true` บอก `clap` ว่า flag นี้ใช้ได้จากทุกจุดในคำสั่ง ไม่ว่าจะเขียนก่อนหรือหลังชื่อ subcommand ก็ตาม (`logcli --color always filter ...` และ `logcli filter --color always ...` ใช้ได้ทั้งคู่) — โดยไม่ต้องประกาศ `config`/`color`/`no_progress` ซ้ำในทั้ง `FilterArgs` และ `StatsArgs` (ตรงกับหลักการ DRY ที่ Part 59.5.1 อธิบายไว้ตอนสอน `#[command(flatten)]` แต่ `global = true` เป็นวิธีที่ตรงไปตรงมากว่าสำหรับกรณีที่ argument กลุ่มนั้นอยู่ที่ระดับ `Cli` บนสุด ไม่ต้องสร้าง `Args` struct แยกเหมือนกรณีที่ต้องแชร์ระหว่าง sibling ของ subcommand เอง)

**`clap_complete::Shell` ใช้ตรงเป็น type ของ field `shell` ได้เลย** — เพราะ crate `clap_complete` เอง implement `ValueEnum` ให้ `Shell` มาแล้ว (รองรับ `bash`, `zsh`, `fish`, `elvish`, `powershell`) เราแค่ใส่ `#[arg(value_enum)]` แล้วใช้งานได้ทันที ไม่ต้องนิยาม enum ของตัวเองซ้ำ — ตัวอย่างของการที่ ecosystem ของ `clap` ขยายตัวผ่าน crate เสริมที่ผสานเข้ากับ derive API ได้แบบไร้รอยต่อ (แนวคิดเดียวกับที่ Part 59 พูดถึง `clap_complete`/`clap_mangen` ไว้สั้น ๆ ว่าเป็นเครื่องมือเสริม — บทนี้คือจุดที่เราเอามาใช้จริง)

มาดู `--help` ระดับบนสุดที่ได้จากโค้ดนี้ (รันจริงผ่าน `cargo run -- --help`):

```
$ logcli --help
logcli: วิเคราะห์ log ที่เขียนเป็น JSON Lines (หนึ่ง JSON object ต่อบรรทัด)

Usage: logcli [OPTIONS] <COMMAND>

Commands:
  filter       กรอง log ตาม level / คำค้น / ช่วงเวลา
  stats        สรุปสถิติ log โดย group ตาม field ที่ระบุ
  completions  พิมพ์ shell completion script ออกทาง stdout
  help         Print this message or the help of the given subcommand(s)

Options:
      --config <PATH>  พาธของไฟล์ config (default: ~/.config/logcli/config.toml — ถ้าไม่มีไฟล์นี้ใช้ค่า default ในตัว)
      --color <MODE>   ควบคุมการใช้สี: auto (ตรวจจาก terminal อัตโนมัติ), always, never [possible values: auto, always, never]
      --no-progress    ปิด progress bar แม้ไฟล์จะใหญ่ก็ตาม (มีผลเหมือนตั้ง progress = false ใน config file)
  -h, --help           Print help
  -V, --version        Print version
```

และ `--help` ของ `filter` (สังเกตว่า global argument อย่าง `--config`/`--color`/`--no-progress` ปรากฏปนอยู่กับ argument เฉพาะของ `filter` เองโดยอัตโนมัติ เพราะ `global = true`):

```
$ logcli filter --help
กรอง log ตาม level / คำค้น / ช่วงเวลา

Usage: logcli filter [OPTIONS] <FILE>

Arguments:
  <FILE>  พาธของไฟล์ log (ใช้ '-' เพื่ออ่านจาก stdin แทน)

Options:
      --level <LEVEL>    กรองเฉพาะ log level ที่ระบุ (ใส่ได้หลายตัว เช่น --level error --level warn)
      --contains <TEXT>  กรองเฉพาะบรรทัดที่ message มีคำนี้อยู่ (case-sensitive)
      --since <TIME>     แสดงเฉพาะ log ที่เกิดตั้งแต่เวลานี้เป็นต้นไป (RFC 3339 เช่น 2024-01-15T00:00:00Z)
      --config <PATH>    พาธของไฟล์ config (default: ~/.config/logcli/config.toml — ถ้าไม่มีไฟล์นี้ใช้ค่า default ในตัว)
      --until <TIME>     แสดงเฉพาะ log ที่เกิดก่อนเวลานี้ (RFC 3339)
      --color <MODE>     ควบคุมการใช้สี: auto (ตรวจจาก terminal อัตโนมัติ), always, never [possible values: auto, always, never]
      --limit <N>        จำกัดจำนวนผลลัพธ์สูงสุดที่แสดง
      --format <FORMAT>  รูปแบบผลลัพธ์: human (อ่านง่าย มีสี) หรือ json (JSON Lines สำหรับต่อท่อ) [possible values: human, json]
      --no-progress      ปิด progress bar แม้ไฟล์จะใหญ่ก็ตาม (มีผลเหมือนตั้ง progress = false ใน config file)
      --strict           ถ้า log line ไหน parse ไม่ผ่าน ให้หยุดทำงานทั้งโปรแกรมทันที (exit code 65) แทนพฤติกรรม default ที่ข้ามบรรทัดเสียแล้วรายงานสรุปท้ายสุด
  -h, --help             Print help
```

### 107.5 `AppError`: รวม Error จากทุกแหล่งเป็นชนิดเดียว พร้อม Exit Code ที่ถูกต้อง

โปรแกรมนี้มี error เกิดขึ้นได้จากหลายแหล่งมาก: `std::io` (เปิดไฟล์ไม่ได้), `serde_json` (parse log line ไม่ผ่าน), `toml` (parse config ไม่ผ่าน), และ business logic ของเราเอง (`--since` มาหลัง `--until`) — Part 30-31 สอนแล้วว่า `thiserror` ช่วยรวม error เหล่านี้เป็น `enum` เดียวพร้อม error message ที่มีคุณภาพ บทนี้จะใช้เต็มรูปแบบ พร้อม**เพิ่มมิติที่ Part 59 ยังไม่ได้ลงรายละเอียด**: การแม็ปแต่ละ error variant ไปเป็น **exit code ตามธรรมเนียม `sysexits.h`** (มาตรฐานเก่าจาก BSD Unix ที่นิยามความหมายของ exit code แต่ละช่วงไว้ — เครื่องมือ command-line จำนวนมากในโลกจริงยึดตามนี้ เพื่อให้ script ที่เรียกเครื่องมือเหล่านั้นตัดสินใจตามสาเหตุความล้มเหลวได้โดยไม่ต้อง parse ข้อความ):

```rust
use std::path::PathBuf;

/// Exit codes ตามธรรมเนียม sysexits.h (BSD) — Part 59 ใช้แนวทางเดียวกันนี้
/// การใช้ค่าคงที่แบบมีชื่อ (ไม่ใช่เลขลอย ๆ) ทำให้อ่าน error handling code ได้ง่ายขึ้นมาก
pub mod exit_code {
    /// ทุกอย่างสำเร็จ (ไม่ได้ถูกอ้างถึงตรง ๆ ในโค้ด เพราะ `ExitCode::SUCCESS` ของ std ทำหน้าที่นี้แทน —
    /// เก็บชื่อนี้ไว้เพื่อให้ตารางค่า exit code ในโมดูลนี้อ่านครบทุกกรณีในที่เดียว)
    #[allow(dead_code)]
    pub const OK: u8 = 0;
    /// ผู้ใช้ใส่ argument/flag ผิด (clap เองใช้ค่านี้อยู่แล้วเมื่อ parse ไม่ผ่าน)
    pub const USAGE: u8 = 64;
    /// ข้อมูล "input" ที่โปรแกรมอ่านมาผิดรูปแบบ (เช่น log line เสียในโหมด --strict)
    pub const DATA_ERR: u8 = 65;
    /// ไฟล์ input ที่ผู้ใช้ระบุไม่มีอยู่จริง หรือเปิดไม่ได้
    pub const NO_INPUT: u8 = 66;
    /// I/O ล้มเหลวแบบไม่คาดคิด (เช่น disk error, broken pipe ที่ไม่ใช่ EPIPE ปกติ)
    pub const IO_ERR: u8 = 74;
    /// ไฟล์ config มีอยู่จริงแต่เนื้อหาผิดรูปแบบ
    pub const CONFIG: u8 = 78;
    /// bug ภายในโปรแกรมเอง — ไม่ใช่ความผิดของผู้ใช้เลย
    pub const SOFTWARE: u8 = 70;
}

/// AppError คือ error type เดียวที่ `main` เห็น ไม่ว่า error จะมาจากแหล่งใด
/// (clap, I/O, serde_json, toml, หรือ business logic ของเราเอง)
///
/// หลักการออกแบบที่ตั้งใจไว้ (เชื่อมกับ Part 30/31): แต่ละ variant แบ่งออกเป็นสองกลุ่มชัดเจน
/// 1) "ผู้ใช้ทำผิด" (bad input) — ข้อความต้อง **บอกวิธีแก้** ให้ผู้ใช้แก้ไขเองได้ทันที
/// 2) "บางอย่างพังโดยไม่คาดคิด" (unexpected) — ข้อความบอกว่า "นี่ไม่ใช่ความผิดของคุณ"
///    และควรมี error chain (#[source]) ให้ไล่สาเหตุจริงได้ ไม่ปิดบังรายละเอียด
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error(
        "ไม่พบไฟล์ '{path}' — ตรวจสอบว่าพาธถูกต้องและไฟล์มีอยู่จริง (ใช้ '-' ถ้าต้องการอ่านจาก stdin)"
    )]
    FileNotFound { path: PathBuf },

    #[error("เปิดไฟล์ '{path}' ไม่สำเร็จ: {source}")]
    FileOpen {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },

    #[error(
        "log line ที่ {line_no} ในไฟล์ '{path}' ไม่ใช่ JSON ที่ถูกต้อง: {source}\n  เนื้อหาที่พังคือ: {raw}"
    )]
    MalformedLine {
        path: PathBuf,
        line_no: usize,
        raw: String,
        #[source]
        source: serde_json::Error,
    },

    #[error("ช่วงเวลาไม่ถูกต้อง: --since ({since}) ต้องมาก่อน --until ({until})")]
    InvalidTimeRange { since: String, until: String },

    #[error(
        "รูปแบบวันเวลา '{value}' ไม่ถูกต้อง — ต้องเป็น RFC 3339 เช่น 2024-01-15T00:00:00Z (รายละเอียด: {reason})"
    )]
    InvalidTimestamp { value: String, reason: String },

    #[error("ไฟล์ config '{path}' มีอยู่จริงแต่ parse ไม่ผ่าน: {source}")]
    ConfigInvalid {
        path: PathBuf,
        #[source]
        source: Box<toml::de::Error>,
    },

    #[error("อ่านไฟล์ config '{path}' ไม่สำเร็จ: {source}")]
    ConfigRead {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },

    /// I/O ที่ไม่คาดคิดระหว่างเขียนผลลัพธ์ออก (เช่น stdout ถูกปิดกลางทาง)
    #[error("เกิดข้อผิดพลาดด้าน I/O ที่ไม่คาดคิด: {source}")]
    UnexpectedIo {
        #[source]
        source: std::io::Error,
    },

    /// กรณีสุดท้ายที่ไม่ควรเกิดขึ้นเลยถ้าโค้ดถูกต้อง — เก็บไว้เป็น safety net
    /// เพื่อไม่ให้โปรแกรม panic ดิบ ๆ ออกไปถึงผู้ใช้ (เชื่อมกับ Part 30 เรื่อง invariant ภายใน)
    #[error("internal error (นี่คือ bug ของโปรแกรม โปรดรายงานพร้อมข้อความนี้): {0}")]
    Internal(String),
}

impl AppError {
    /// แปลง error แต่ละชนิดให้เป็น exit code ที่ถูกต้องตามธรรมเนียม sysexits.h
    /// นี่คือจุดเดียวในโปรแกรมทั้งหมดที่ตัดสินใจเรื่อง exit code — ไม่กระจัดกระจายอยู่หลายที่
    pub fn exit_code(&self) -> u8 {
        use exit_code::*;
        match self {
            AppError::FileNotFound { .. } => NO_INPUT,
            AppError::FileOpen { .. } => NO_INPUT,
            AppError::MalformedLine { .. } => DATA_ERR,
            AppError::InvalidTimeRange { .. } => USAGE,
            AppError::InvalidTimestamp { .. } => USAGE,
            AppError::ConfigInvalid { .. } => CONFIG,
            AppError::ConfigRead { .. } => CONFIG,
            AppError::UnexpectedIo { .. } => IO_ERR,
            AppError::Internal(_) => SOFTWARE,
        }
    }

    /// true ถ้า error นี้เป็นความผิดของผู้ใช้ (ผู้ใช้แก้ไขเองได้ด้วยการเปลี่ยน argument/ไฟล์)
    /// false ถ้าเป็นสิ่งที่ "ไม่ควรเกิด" — ใช้ตัดสินใจว่าจะพิมพ์บอกเพิ่มไหมว่า "ไม่ใช่ความผิดคุณ"
    pub fn is_user_error(&self) -> bool {
        !matches!(self, AppError::UnexpectedIo { .. } | AppError::Internal(_))
    }
}
```

จุดที่ควรสังเกต:

**`source: Box<toml::de::Error>` ใน `ConfigInvalid`** — ใส่ `Box` ห่อไว้แม้ว่า variant อื่น ๆ ที่มี `#[source]` (เช่น `FileOpen`, `ConfigRead`) ไม่ได้ห่อ `Box` เลย เหตุผลคือ **ขนาด (size) ของ `enum` ทั้งตัวถูกกำหนดโดย variant ที่ "หนัก" ที่สุด** — ถ้า `toml::de::Error` (ซึ่งเก็บรายละเอียดตำแหน่ง error หลาย field ภายใน) มีขนาดใหญ่กว่า variant อื่นอย่างมีนัยสำคัญ ทุกครั้งที่ส่ง `AppError` ผ่าน `Result<T, AppError>` (แม้ในกรณีที่เป็น `Ok(T)` ที่ไม่มี error เลย!) โปรแกรมต้องจอง stack space เท่ากับ variant ที่ใหญ่ที่สุดไว้เผื่อ — การห่อด้วย `Box` (ที่มีขนาดคงที่แค่ 8 byte บนระบบ 64-bit เพราะเป็น pointer) ทำให้ขนาดของ `AppError` ทั้ง enum ไม่ขึ้นกับขนาดของ error ภายในของ crate อื่นที่เราคุมไม่ได้ — เป็นวินัยที่ดีที่ควรฝึกไว้ทุกครั้งที่ error variant หนึ่งอาจมีขนาดต่างจากตัวอื่นมาก (clippy มี lint ชื่อ `result_large_err`/`large_enum_variant` ที่ตรวจจับกรณีนี้ให้ในบางสถานการณ์ — แม้ในโปรเจกต์นี้ยังไม่โตพอที่ clippy จะเตือน แต่การป้องกันไว้ล่วงหน้าเป็นนิสัยที่ดี)

**`is_user_error()`** — ใช้ตัดสินใจใน `main()` ว่าจะพิมพ์ข้อความเสริม "นี่ไม่ใช่ความผิดของคุณ" หรือไม่ — เป็นรายละเอียดเล็กที่ส่งผลกับ **experience ของผู้ใช้ปลายทาง** อย่างมาก: เมื่อผู้ใช้ทำอะไรผิดเอง (เช่น พิมพ์พาธไฟล์ผิด) ข้อความ error ที่ดีที่สุดคือข้อความที่บอกวิธีแก้ตรง ๆ ไม่ต้องมีอะไรเพิ่ม แต่เมื่อเกิดสิ่งที่ผู้ใช้ควบคุมไม่ได้ (เช่น disk I/O ล้มเหลวกลางทาง) ควรสื่อสารชัดเจนว่า "นี่ไม่ใช่ความผิดของคุณ" เพื่อไม่ให้ผู้ใช้เสียเวลาสงสัยว่าตัวเองทำอะไรผิด

มาดู `main.rs` ส่วนที่แปลง `AppError` ให้เป็น output ที่ผู้ใช้เห็นจริง:

```rust
fn main() -> ExitCode {
    let cli = Cli::parse();

    match run(cli) {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            eprintln!("logcli: {err}");
            if !err.is_user_error() {
                eprintln!(
                    "หมายเหตุ: นี่ไม่ใช่ความผิดของคุณ — ถ้าปัญหานี้เกิดซ้ำ โปรดรายงานพร้อมคำสั่งที่ใช้"
                );
            }
            ExitCode::from(err.exit_code())
        }
    }
}
```

สังเกตว่า `main()` คืนค่า `ExitCode` (จาก `std::process::ExitCode` ตามที่ Part 12 แนะนำไว้) **ไม่ใช่** `Result<(), AppError>` ตรง ๆ — นี่คือการตัดสินใจที่ตั้งใจมาก เพราะถ้าให้ `main()` คืน `Result<(), E>` ตรง ๆ, Rust standard library จะพิมพ์ error ด้วย **`{:?}` (Debug) ไม่ใช่ `{}` (Display)** และ**บังคับ exit code เป็น 1 เสมอ** ไม่ว่า error จะเป็นชนิดไหน — รายละเอียดนี้สำคัญมากพอที่จะเป็นกับดักข้อแรกในหัวข้อถัดไป

### 107.6 อ่านไฟล์แบบทนทาน + Progress Bar ด้วย `indicatif`

นี่คือหัวใจของ "production-grade" ในชื่อบทนี้ — log file ในโลกจริงมีขนาดหลักร้อย MB ถึงหลาย GB และ**มักมีบรรทัดเสียปนอยู่บ้างเสมอ** (process ถูก kill กลางทางขณะเขียนบรรทัดสุดท้ายยังไม่ทัน, disk เต็มชั่วขณะ, ปัญหา encoding) เครื่องมือที่ crash ทั้งตัวเพราะบรรทัดเดียวพังจะใช้งานไม่ได้จริงในสถานการณ์ที่คนอยากใช้เครื่องมือนี้มากที่สุด — ตอนกำลัง debug incident ที่ log ไฟล์มีขนาดใหญ่และอาจไม่สมบูรณ์

```rust
use crate::error::AppError;
use crate::model::LogEntry;
use indicatif::{ProgressBar, ProgressDrawTarget, ProgressStyle};
use is_terminal::IsTerminal;
use std::io::{self, BufRead, BufReader, Read};
use std::path::Path;

/// ผลของการอ่านไฟล์ log ทั้งไฟล์: entry ที่ parse ผ่าน + จำนวนบรรทัดที่พังไปทั้งหมด
/// (เฉพาะโหมด non-strict เท่านั้นที่จะได้ `skipped > 0` — โหมด strict คืน `Err` ทันทีที่เจอบรรทัดพัง)
pub struct ReadOutcome {
    pub entries: Vec<LogEntry>,
    pub skipped: usize,
}

/// อ่านและ parse log ทั้งไฟล์ (หรือ stdin ถ้า path คือ "-")
///
/// **ทางเลือกด้าน semantics ที่ตั้งใจอธิบายไว้ตรงนี้:** เราเลือก "best-effort by default" —
/// บรรทัดที่ parse ไม่ผ่านจะถูกข้ามและรายงานเป็น warning ไปที่ stderr ทันที (พร้อมเลขบรรทัดและ
/// เนื้อหาที่พัง) แล้วโปรแกรม**ทำงานต่อ**กับบรรทัดที่เหลือ เหตุผลคือ log file ในโลกจริงมักมีขนาด
/// หลักร้อย MB ถึงหลาย GB และมักมีบรรทัดเสียปนอยู่บ้างเสมอ (เช่น process ถูก kill กลางทางขณะเขียน
/// บรรทัดสุดท้ายไม่ทัน, disk เต็มชั่วขณะ) — การให้ทั้งการวิเคราะห์ล้มเหลวเพราะบรรทัดเดียวจะทำให้เครื่องมือ
/// นี้ใช้งานไม่ได้จริงกับ log ขนาดใหญ่ในสถานการณ์ที่คนอยากใช้เครื่องมือนี้มากที่สุด (กำลัง debug incident)
///
/// แต่บาง workflow (เช่น CI ที่ตรวจสอบว่า log format ถูกต้อง 100% ก่อน deploy) ต้องการความเข้มงวดกว่านั้น
/// จึงเปิดทางเลือก `--strict` ให้เปลี่ยนพฤติกรรมเป็น "เจอบรรทัดพังบรรทัดแรก หยุดทันทีด้วย exit code 65"
pub fn read_log_entries(
    path: &Path,
    strict: bool,
    show_progress: bool,
) -> Result<ReadOutcome, AppError> {
    if path == Path::new("-") {
        let stdin = io::stdin();
        return read_from(stdin.lock(), path, strict, false);
    }

    if !path.exists() {
        return Err(AppError::FileNotFound {
            path: path.to_path_buf(),
        });
    }

    let file = std::fs::File::open(path).map_err(|source| AppError::FileOpen {
        path: path.to_path_buf(),
        source,
    })?;
    read_from(file, path, strict, show_progress)
}

fn read_from<R: Read>(
    reader: R,
    path: &Path,
    strict: bool,
    show_progress: bool,
) -> Result<ReadOutcome, AppError> {
    let buffered = BufReader::new(reader);

    let pb = if show_progress {
        let bar = ProgressBar::new_spinner();
        bar.set_draw_target(progress_draw_target());
        bar.set_style(
            ProgressStyle::with_template("{spinner:.cyan} อ่าน log... {pos} บรรทัด ({per_sec})")
                .unwrap(),
        );
        Some(bar)
    } else {
        None
    };

    let mut entries = Vec::new();
    let mut skipped = 0usize;

    for (idx, line_result) in buffered.lines().enumerate() {
        let line_no = idx + 1;
        let line = line_result.map_err(|source| AppError::UnexpectedIo { source })?;

        if let Some(bar) = &pb {
            bar.inc(1);
        }

        // บรรทัดว่างเปล่า (เช่น newline สุดท้ายของไฟล์) ไม่ถือว่าเป็นบรรทัดพัง — ข้ามเงียบ ๆ
        if line.trim().is_empty() {
            continue;
        }

        match serde_json::from_str::<LogEntry>(&line) {
            Ok(entry) => entries.push(entry),
            Err(source) => {
                if strict {
                    if let Some(bar) = &pb {
                        bar.finish_and_clear();
                    }
                    return Err(AppError::MalformedLine {
                        path: path.to_path_buf(),
                        line_no,
                        raw: truncate(&line, 120),
                        source,
                    });
                }
                eprintln!(
                    "คำเตือน: ข้ามบรรทัดที่ {line_no} ใน '{}' เพราะ parse ไม่ผ่าน: {source}",
                    path.display()
                );
                skipped += 1;
            }
        }
    }

    if let Some(bar) = &pb {
        bar.finish_and_clear();
    }

    Ok(ReadOutcome { entries, skipped })
}

/// ตัดสตริงให้สั้นลงถ้ายาวเกินไป (สำหรับโชว์เนื้อหาบรรทัดที่พังใน error message โดยไม่ยาวเกินจนอ่านลำบาก)
fn truncate(s: &str, max_len: usize) -> String {
    if s.chars().count() <= max_len {
        s.to_string()
    } else {
        let truncated: String = s.chars().take(max_len).collect();
        format!("{truncated}...")
    }
}

/// ซ่อน progress bar โดยอัตโนมัติเมื่อ stderr ไม่ใช่ terminal จริง (ถูก pipe/redirect)
/// เหตุผล: ตัวอักษรควบคุมของ progress bar (carriage return, ANSI escape) ไม่มีความหมายถ้าปลายทาง
/// เป็นไฟล์หรือโปรแกรมอื่นที่ไม่ได้ render terminal จริง จะเห็นเป็นขยะปนอยู่ในไฟล์ log/output
fn progress_draw_target() -> ProgressDrawTarget {
    if io::stderr().is_terminal() {
        ProgressDrawTarget::stderr()
    } else {
        ProgressDrawTarget::hidden()
    }
}
```

ไล่ดูการตัดสินใจสำคัญ:

**progress bar วาดไปที่ `stderr` ไม่ใช่ `stdout`** — นี่คือธรรมเนียม Unix ที่สำคัญมาก: **`stdout` มีไว้สำหรับ "ข้อมูลที่โปรแกรมผลิต"** (ผลลัพธ์ที่ต่อท่อไปเครื่องมืออื่นได้) ส่วน **`stderr` มีไว้สำหรับ "ข้อความสถานะที่มนุษย์ดู"** (progress, warning, log การทำงาน) — ถ้าเผลอวาด progress bar ไปที่ `stdout` แม้จะดูปกติตอนรันตรง ๆ บน terminal (เพราะ terminal render ANSI escape code ให้เหมือนกัน) แต่พอมีคนเอา `logcli stats --format json app.log | jq .` การผสมกันของ spinner character/ANSI escape code กับ JSON data ในสตรีมเดียวกันจะทำให้ปลายทางที่คาดหวัง byte สตรีมสะอาดพัง (ดูตัวอย่างจริงที่พังในหัวข้อกับดักข้อ 3)

**ตรวจ `is_terminal()` ก่อนวาด แม้จะวาดไปที่ `stderr` แล้วก็ตาม** — เหตุผลคือ `stderr` เองก็ถูก redirect ได้เหมือนกัน (`logcli stats app.log 2> error.log`) ถ้าวาด progress bar ไม่เช็คก่อน ไฟล์ `error.log` จะเต็มไปด้วยตัวอักษรควบคุม (`\r`, ANSI escape) นับพันบรรทัดที่ไม่มีประโยชน์เลยเมื่อเปิดไฟล์นั้นด้วย text editor ทีหลัง — `progress_draw_target()` จึงคืน `ProgressDrawTarget::hidden()` (ปิดการวาดไปเลย ไม่ใช่แค่ "วาดแบบไม่มีสี") เมื่อ `stderr` ไม่ใช่ terminal จริง

มาดูผลลัพธ์จริง — สร้างไฟล์ log ทดสอบขนาด 200,000 บรรทัด (~29MB) แล้วรัน `stats` (จับภาพผ่าน pseudo-terminal จริงเพื่อให้ progress bar แสดงผล เพราะปกติเวลารันในสภาพแวดล้อมที่ไม่มี terminal จริง `is_terminal()` จะเป็น `false` และซ่อน progress bar ไปเลยตามที่อธิบายไว้ข้างบน — ในการใช้งานจริงบน terminal ทั่วไป progress bar จะปรากฏขึ้นมาเองโดยไม่ต้องทำอะไรเพิ่ม):

```
$ logcli stats --by endpoint big.jsonl
⠁ อ่าน log... 1 บรรทัด (47,369.4972/s)
⠉ อ่าน log... 2 บรรทัด (33,147.6352/s)
⠙ อ่าน log... 3 บรรทัด (33,462.9043/s)
⠚ อ่าน log... 4 บรรทัด (34,090.3601/s)
⠒ อ่าน log... 5 บรรทัด (35,888.7986/s)
⠂ อ่าน log... 6 บรรทัด (37,369.4268/s)
   ⋮  (spinner หมุนต่อเนื่อง วาดทับบรรทัดเดียวกันด้วย \r บน terminal จริง)
⠒ อ่าน log... 7213 บรรทัด (691,248.2362/s)
⠙ อ่าน log... 37605 บรรทัด (726,342.9255/s)
⠓ อ่าน log... 74459 บรรทัด (731,010.7263/s)
⠠ อ่าน log... 108172 บรรทัด (729,430.6838/s)
⠒ อ่าน log... 142601 บรรทัด (726,770.0227/s)
⠁ อ่าน log... 174069 บรรทัด (722,642.6978/s)
endpoint                count  percent
/api/auth               40205    20.1%
/api/orders             40182    20.1%
/api/users              40079    20.0%
/api/payments           39919    20.0%
/api/search             39615    19.8%
รวมทั้งหมด: 200000 entries
```

(เฟรมที่แสดงคือตัวอย่างที่คัดมาจากการรันจริง ~25 เฟรม — บน terminal จริง แต่ละเฟรมจะวาดทับบรรทัดเดิม ไม่ใช่พิมพ์บรรทัดใหม่ต่อกันแบบนี้ เพราะ `indicatif` ใช้ ANSI escape sequence `\x1b[2K` (clear line) ตามด้วย `\r` เพื่อกลับไปวาดทับตำแหน่งเดิมทุกครั้งที่อัปเดต — สิ่งที่เห็นในตัวอย่างนี้คือการแยกแต่ละเฟรมออกจากกันเพื่ออธิบาย ไม่ใช่การพิมพ์ 25 บรรทัดจริง) เมื่อทำงานจบ `bar.finish_and_clear()` จะลบ progress bar ทิ้งไปเลย เหลือแค่ผลลัพธ์สุดท้าย (ตารางสถิติ) ให้เห็นสะอาด ๆ — สังเกตความเร็วที่เพิ่มขึ้นอย่างรวดเร็ว (จากหลักหมื่นบรรทัด/วินาทีในช่วงแรกไปจนถึงกว่า 700,000 บรรทัด/วินาที) เพราะช่วงแรกของการวัด per-second throughput ยังมีตัวอย่างน้อยเกินไปให้คำนวณค่าเฉลี่ยได้แม่นยำ

### 107.7 Config File และลำดับความสำคัญ (Precedence)

เครื่องมือ command-line ระดับ production ที่ดีไม่ควรบีบให้ผู้ใช้พิมพ์ flag เดิมซ้ำทุกครั้งที่เรียกใช้ — ถ้าผู้ใช้ต้องการปิดสีเสมอ หรือต้องการให้ output เป็น JSON เสมอ (เพราะต่อท่อเข้า script อื่นเป็นประจำ) ควรตั้งค่าไว้ **ครั้งเดียว** ในไฟล์ config แล้วไม่ต้องพิมพ์ flag ซ้ำอีก — นี่คือแนวคิดเดียวกับที่ Part 59 พูดถึง `#[arg(env = "...")]` (12-factor app) แต่ขยายไปสู่ config file แบบเต็มรูปแบบ พร้อม**ลำดับความสำคัญ**ที่ต้องชัดเจน:

```
CLI flag (ระบุตรง ๆ ตอนรัน) > config file (~/.config/logcli/config.toml) > ค่า default ในตัวโปรแกรม
```

```rust
use crate::error::AppError;
use serde::Deserialize;
use std::path::{Path, PathBuf};

/// ค่าที่โหลดมาจาก `~/.config/logcli/config.toml`
///
/// ทุก field เป็น `Option<T>` โดยตั้งใจ — ถ้าผู้ใช้ไม่ได้เขียน key นั้นไว้ในไฟล์ config
/// เราต้องรู้ว่า "ไม่ได้ตั้งค่าไว้" (None) แยกจาก "ตั้งค่าเป็นค่าที่บังเอิญตรงกับ default" (Some(default))
/// เพื่อให้ precedence CLI > config file > built-in default ทำงานถูกต้อง — ถ้าใช้ default_value
/// ธรรมดาที่นี่ เราจะแยกไม่ออกเลยว่าค่าที่ได้มาจาก config file จริง ๆ หรือมาจาก default ของ struct
#[derive(Debug, Default, Deserialize)]
pub struct FileConfig {
    pub color: Option<String>,
    pub format: Option<String>,
    pub progress: Option<bool>,
    pub strict: Option<bool>,
}

impl FileConfig {
    /// พาธ default ตามธรรมเนียม XDG: `~/.config/logcli/config.toml`
    /// ใช้ crate `dirs` (ไม่ hardcode `$HOME` เอง เพราะพฤติกรรมต่างกันระหว่าง Linux/macOS/Windows)
    pub fn default_path() -> Option<PathBuf> {
        dirs::config_dir().map(|d| d.join("logcli").join("config.toml"))
    }

    /// โหลดไฟล์ config ถ้ามีอยู่จริง — ถ้าไฟล์ไม่มีอยู่เลยถือว่า "ไม่ได้ตั้งค่าอะไรไว้" (ไม่ใช่ error)
    /// เพราะ config file เป็น**ของเสริม**เสมอ ผู้ใช้ต้องรันโปรแกรมได้แม้ไม่มีไฟล์นี้เลย (12-factor: config
    /// ควรมี default ที่สมเหตุสมผลในตัวโปรแกรมอยู่แล้ว ไฟล์ config มีไว้แค่ "override" เท่านั้น)
    /// แต่ถ้าไฟล์**มีอยู่จริง**แล้ว parse ไม่ผ่าน นั่นคือความผิดพลาดที่ต้องแจ้งผู้ใช้ตรง ๆ ไม่ใช่เงียบไว้
    pub fn load(path: &Path) -> Result<Self, AppError> {
        if !path.exists() {
            return Ok(FileConfig::default());
        }
        let content = std::fs::read_to_string(path).map_err(|source| AppError::ConfigRead {
            path: path.to_path_buf(),
            source,
        })?;
        toml::from_str(&content).map_err(|source| AppError::ConfigInvalid {
            path: path.to_path_buf(),
            source: Box::new(source),
        })
    }
}

/// ตัดสินค่าสุดท้ายระหว่าง CLI flag, config file, และ built-in default ตามลำดับความสำคัญ
/// (CLI ชนะเสมอถ้าผู้ใช้ระบุมา, ถ้าไม่ระบุค่อยดู config file, ถ้าไม่มีทั้งคู่ใช้ default)
///
/// generic function ตัวเดียวนี้ใช้ resolve ได้ทุก field เพราะ pattern การตัดสินใจเหมือนกันทุกครั้ง
/// (เชื่อมกับ Part 18-19 เรื่อง generics: เขียน logic ทั่วไปครั้งเดียว ใช้ซ้ำได้กับทุก type ที่ต้องการ)
pub fn resolve<T>(cli_value: Option<T>, config_value: Option<T>, default: T) -> T {
    cli_value.or(config_value).unwrap_or(default)
}
```

จุดออกแบบที่สำคัญที่สุดของหัวข้อนี้: **field ของ `FilterArgs`/`StatsArgs` ที่ต้องผสานกับ config file (เช่น `format`) ต้องเป็น `Option<T>` ไม่ใช่ type ที่มี `default_value`** ย้อนกลับไปดู `cli.rs` ในหัวข้อ 107.4 จะเห็นว่า `pub format: Option<OutputFormat>` (ไม่มี `default_value`) — นี่คือความจำเป็นที่มาจาก precedence logic ตรง ๆ: ถ้าเราให้ `clap` ใส่ `default_value = "human"` ไว้ที่ field นี้เลย เมื่อผู้ใช้**ไม่ได้พิมพ์ `--format` เลย** ค่าที่ `clap` คืนมาก็จะเป็น `OutputFormat::Human` อยู่ดี (ไม่ใช่ `None`) — โค้ดของเราจะแยกไม่ออกเลยว่า "`Human` มาจากที่ผู้ใช้ตั้งใจพิมพ์ `--format human` มา" หรือ "`Human` มาจาก default ของ `clap` เพราะผู้ใช้ไม่ได้พิมพ์อะไรเลย" — ถ้าแยกไม่ออก **config file ก็ไม่มีวันมีผลอะไรเลย** เพราะ `cli_value` (ที่ตอนนี้เป็น `Some(Human)` เสมอไม่ว่าผู้ใช้พิมพ์อะไรมา) จะชนะ `config_value` ทุกครั้งตาม logic ของ `resolve()` — นี่คือกับดักที่สาธิตพร้อมผลลัพธ์จริงในหัวข้อกับดักข้อ 2

การเรียกใช้ `resolve()` ใน `main.rs`:

```rust
fn resolve_format(cli_value: Option<OutputFormat>, file_config: &FileConfig) -> OutputFormat {
    let from_config = file_config.format.as_deref().and_then(|s| match s {
        "json" => Some(OutputFormat::Json),
        "human" => Some(OutputFormat::Human),
        _ => None,
    });
    config::resolve(cli_value, from_config, OutputFormat::Human)
}
```

สังเกตว่าค่าจาก config file เป็น `String` ดิบ ๆ (`"json"`/`"human"`) ที่ต้อง**แปลงเป็น `OutputFormat` เอง** — ต่างจากค่าจาก CLI ที่ `clap` แปลงให้เป็น `OutputFormat` (ผ่าน `ValueEnum`) ให้เสร็จแล้วตั้งแต่ parse (Part 59 หัวข้อ type-directed parsing) เหตุผลคือ `toml`/`serde` ไม่รู้จัก `ValueEnum` ของ `clap` เลย (เป็น trait คนละระบบกัน) เราจึงต้องเขียน mapping ของตัวเองสำหรับค่าที่มาจากไฟล์ config — ถ้าค่าใน config ไม่ตรงกับที่รู้จัก (เช่นผู้ใช้พิมพ์ `format = "yaml"` ผิด) ฟังก์ชันนี้จะคืน `None` เงียบ ๆ (ตกไปใช้ default) ซึ่งเป็นทางเลือกด้าน UX ที่ยอมรับได้สำหรับบทนี้ — ถ้าต้องการความเข้มงวดกว่านี้ ควรคืน `Result` แล้วแจ้ง error ชัดเจนว่า config file มีค่าที่ไม่รู้จัก (ทิ้งเป็นแบบฝึกหัดข้อ 4)

ทดสอบ precedence จริง — สร้างไฟล์ config ที่ตั้ง `format = "json"` และ `color = "never"`:

```bash
$ cat ~/.config/logcli/config.toml
format = "json"
color = "never"
progress = false
```

รันโดยไม่ระบุ `--format` เลย (ต้องได้ JSON ตามค่าใน config):

```
$ logcli stats --by level sample.jsonl
{
  "debug": 1,
  "error": 2,
  "info": 4,
  "warn": 2
}
```

รันอีกครั้งพร้อมระบุ `--format human` ตรง ๆ (CLI ต้องชนะ config file):

```
$ logcli stats --by level --format human sample.jsonl
level                   count  percent
info                        4    44.4%
error                       2    22.2%
warn                        2    22.2%
debug                       1    11.1%
รวมทั้งหมด: 9 entries
```

และถ้าไฟล์ config มีอยู่จริงแต่เนื้อหาผิดรูปแบบ TOML (ตั้งใจใส่ syntax ผิด):

```
$ cat bad-config.toml
this is not valid = = toml [[[

$ logcli stats --by level sample.jsonl --config bad-config.toml
logcli: ไฟล์ config 'bad-config.toml' มีอยู่จริงแต่ parse ไม่ผ่าน: TOML parse error at line 1, column 6
  |
1 | this is not valid = = toml [[[
  |      ^
key with no value, expected `=`

$ echo "exit code: $?"
exit code: 78
```

สังเกตว่า exit code เป็น **78** (`CONFIG` ตาม mapping ในหัวข้อ 107.5) ไม่ใช่ 1 ธรรมดา — script ที่เรียก `logcli` เห็น exit code นี้แล้วรู้ทันทีว่าปัญหาอยู่ที่ไฟล์ config โดยไม่ต้อง parse ข้อความ stderr เอง

### 107.8 Output แบบ Human-Readable (มีสี) และ JSON (Machine-Readable)

เครื่องมือ production-grade ต้องรองรับผู้ใช้สองกลุ่ม: **มนุษย์**ที่นั่งอ่านผลลัพธ์บน terminal (ต้องการสีช่วยแยกความสำคัญ) และ**โปรแกรมอื่น**ที่ต่อท่อผลลัพธ์ไปประมวลผลต่อ (ต้องการ byte stream ที่สะอาด ไม่มี ANSI escape code ปนมา และ parse ได้แน่นอนด้วย `jq`/สคริปต์อื่น) — Part 59 พูดถึงพฤติกรรมนี้ของ `clap` เองสั้น ๆ ไว้แล้ว (ตรวจ terminal, เคารพ `NO_COLOR`) บทนี้จะเขียน logic แบบเดียวกันเอง สำหรับ output ของ**ข้อมูล**ที่เราพิมพ์ (ไม่ใช่ help text ที่ `clap` จัดการให้)

```rust
use crate::cli::ColorMode;
use is_terminal::IsTerminal;
use owo_colors::OwoColorize;

/// ตัดสินใจครั้งเดียวตอนโปรแกรมเริ่มทำงานว่า "ควรใส่สีลง stdout หรือไม่"
/// รวมทุกเงื่อนไขที่เครื่องมือ CLI ระดับ production ต้องเคารพไว้ในที่เดียว:
///
/// 1. ผู้ใช้ขอ `--color always`/`--color never` ตรง ๆ — เชื่อผู้ใช้เสมอ ไม่ต้องเดา
/// 2. มาตรฐาน `NO_COLOR` (https://no-color.org/) — ถ้าตั้ง environment variable นี้ไว้ (ไม่ว่าค่าอะไร)
///    เครื่องมือ CLI ทุกตัวที่ทำถูกต้องต้อง**ปิดสีเสมอ** ไม่ว่า terminal จะรองรับสีหรือไม่ก็ตาม
/// 3. ถ้าไม่มีทั้งสองข้อบน ให้ auto-detect จากว่า stdout เป็น terminal จริงหรือถูก pipe/redirect
///    (เชื่อมกับปัญหาเดียวกันที่ Part 59 อธิบายไว้เรื่อง `clap` เองก็ตรวจ terminal แบบนี้)
pub fn should_use_color(mode: ColorMode) -> bool {
    match mode {
        ColorMode::Always => true,
        ColorMode::Never => false,
        ColorMode::Auto => {
            if std::env::var_os("NO_COLOR").is_some() {
                return false;
            }
            std::io::stdout().is_terminal()
        }
    }
}

/// ห่อ log level ด้วยสีที่เหมาะสม (แดง=error, เหลือง=warn, เขียว=info, น้ำเงิน=debug/trace)
/// ถ้า `enabled` เป็น false คืนค่าเป็น string เปล่า ๆ ไม่มี ANSI escape code ปนมาเลย
/// (สำคัญมาก: ต้องไม่ปน escape code ลงไปในโหมดไม่มีสี เพราะโปรแกรมปลายทางที่รับผลลัพธ์ผ่าน pipe
/// อาจไม่รู้จัก ANSI code แล้วเห็นเป็นตัวอักษรแปลก ๆ ปนอยู่ในข้อความ)
pub fn paint_level(level: &str, enabled: bool) -> String {
    if !enabled {
        return level.to_string();
    }
    match level.to_uppercase().as_str() {
        "ERROR" => level.red().bold().to_string(),
        "WARN" | "WARNING" => level.yellow().bold().to_string(),
        "INFO" => level.green().to_string(),
        "DEBUG" => level.blue().to_string(),
        "TRACE" => level.dimmed().to_string(),
        _ => level.to_string(),
    }
}

/// ทำตัวเลข/ชื่อ field ให้เป็นสีเทาจาง ๆ (ใช้กับ metadata ที่ไม่ใช่ใจความหลัก เช่น timestamp)
pub fn paint_dim(text: &str, enabled: bool) -> String {
    if enabled {
        text.dimmed().to_string()
    } else {
        text.to_string()
    }
}

/// ทำตัวหนา (ใช้กับหัวตารางหรือชื่อ field ที่อยากเน้น)
pub fn paint_bold(text: &str, enabled: bool) -> String {
    if enabled {
        text.bold().to_string()
    } else {
        text.to_string()
    }
}
```

**ทำไมทุกฟังก์ชันรับ `enabled: bool` เป็น parameter แทนการเช็ค global state ข้างในฟังก์ชันเอง** — เพราะการตัดสินใจ "ควรใส่สีไหม" ควรเกิด**ครั้งเดียว**ตอนโปรแกรมเริ่มทำงาน (`should_use_color()` เรียกครั้งเดียวใน `main.rs`) แล้วส่งค่าที่ตัดสินใจแล้วผ่านลงไปเป็น parameter — ตรงข้ามกับการให้ `paint_level()` ไปเรียก `is_terminal()`/`env::var_os("NO_COLOR")` เองทุกครั้งที่ถูกเรียก (ซึ่งจะทำงานถูกต้องเหมือนกัน แต่**ทดสอบยากกว่ามาก**: unit test ของ `paint_level()` จะต้องเซ็ต/ล้าง environment variable จริงก่อนทุก test case ซึ่งเสี่ยง race condition ถ้า test รันพร้อมกันหลาย thread — Part 32 พูดถึงปัญหานี้ไว้แล้วว่า global mutable state ทำให้ test เขียนยากขึ้น) การรับ `enabled: bool` ตรง ๆ ทำให้ `paint_level("error", true)` และ `paint_level("error", false)` เป็น **pure function** ที่ทดสอบได้ตรงไปตรงมา ไม่มี side effect หรือการอ่าน state ภายนอกเลย

ทดสอบจริงว่า `--color always` ใส่ ANSI escape code เข้าไปจริง (แสดงด้วย `cat -v` เพื่อให้เห็นตัวอักษรควบคุมที่ปกติมองไม่เห็น — `^[` คืออักขระ ESC ที่เริ่มต้น ANSI escape sequence):

```
$ logcli filter --level error --color always sample.jsonl | cat -v
^[[2m2024-01-15T10:23:07+00:00^[[0m ^[[1m^[[31merror^[[39m^[[0m database connection timeout (duration_ms=3001 endpoint="/api/orders" status=500)
^[[2m2024-01-15T10:23:12+00:00^[[0m ^[[1m^[[31merror^[[39m^[[0m upstream service unavailable (duration_ms=120 endpoint="/api/payments" status=503)
```

และทดสอบว่า `NO_COLOR` ใน mode `auto` ปิดสีสำเร็จ (แม้ผลลัพธ์ถูก pipe ไปที่อื่นซึ่งจะปิดสีอยู่แล้วโดย `is_terminal()`; ทดสอบซ้อนเพื่อยืนยันว่า `NO_COLOR` เช็คก่อนถึง `is_terminal()` เสมอตามลำดับ logic ใน `should_use_color()`):

```
$ NO_COLOR=1 logcli filter --level error --color auto sample.jsonl | cat -v
2024-01-15T10:23:07+00:00 error database connection timeout (duration_ms=3001 endpoint="/api/orders" status=500)
2024-01-15T10:23:12+00:00 error upstream service unavailable (duration_ms=120 endpoint="/api/payments" status=503)
```

ไม่มี `^[` ปนมาเลยแม้แต่ตัวเดียว — ยืนยันว่า path ที่ปิดสีไม่มี ANSI escape code หลงเหลืออยู่จริง

**หมายเหตุเรื่องการตัดสินใจที่จงใจ**: สังเกตว่า `--color always` (การระบุตรง ๆ ของผู้ใช้) ยังคง**ชนะ** `NO_COLOR` ได้ ถ้าผู้ใช้ตั้ง `NO_COLOR=1` ไว้แต่ระบุ `--color always` มาด้วย โค้ดของเราจะ match เข้า `ColorMode::Always => true` ทันทีโดยไม่เช็ค `NO_COLOR` เลย (ดู `should_use_color()` — `NO_COLOR` เช็คเฉพาะกรณี `ColorMode::Auto` เท่านั้น) นี่เป็นการตีความมาตรฐาน `NO_COLOR` แบบที่ให้ **explicit user intent ชนะเสมอ** (ผู้ใช้พิมพ์ `--color always` แปลว่าตั้งใจเปิดสีจริง ๆ ในครั้งนี้ ไม่ใช่ default ที่ตั้งไว้เผื่อลืม) ซึ่งเป็นพฤติกรรมที่เครื่องมือดัง ๆ หลายตัวเลือกใช้ — แต่ผู้อ่านควรรู้ว่านี่คือ**การตีความหนึ่งแบบที่ตกลงกันเอง** ไม่ใช่กฎที่ตายตัว 100% (มาตรฐาน `NO_COLOR` เขียนไว้กว้าง ๆ ว่า "ซอฟต์แวร์ควรงดใส่สี" โดยไม่ได้ระบุชัดว่าต้องชนะ flag ที่ผู้ใช้ตั้งใจระบุมาเองหรือไม่ — ทีมที่สร้างเครื่องมือแต่ละตัวตัดสินใจกันเอง) จุดสำคัญคือ**ต้องเขียนพฤติกรรมนี้ไว้ใน `--help`/documentation ให้ชัดเจน** เพื่อไม่ให้ผู้ใช้สับสน

### 107.9 ประกอบร่าง: Dispatch และ Implement `filter`/`stats`

ตอนนี้มาประกอบทุกโมดูลเข้าด้วยกันใน `main.rs` — จุดเริ่มต้นคือ `run()` ที่โหลด config, ตัดสิน precedence, แล้ว dispatch ไปยัง handler ของแต่ละ subcommand:

```rust
mod cli;
mod config;
mod error;
mod model;
mod output;
mod reader;

use chrono::{DateTime, Utc};
use clap::{CommandFactory, Parser};
use cli::{Cli, ColorMode, Command, CompletionsArgs, FilterArgs, OutputFormat, StatsArgs};
use config::FileConfig;
use error::AppError;
use model::LogEntry;
use std::collections::BTreeMap;
use std::io::Write;
use std::process::ExitCode;

fn main() -> ExitCode {
    let cli = Cli::parse();

    match run(cli) {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            eprintln!("logcli: {err}");
            if !err.is_user_error() {
                eprintln!(
                    "หมายเหตุ: นี่ไม่ใช่ความผิดของคุณ — ถ้าปัญหานี้เกิดซ้ำ โปรดรายงานพร้อมคำสั่งที่ใช้"
                );
            }
            ExitCode::from(err.exit_code())
        }
    }
}

fn run(cli: Cli) -> Result<(), AppError> {
    let config_path = cli
        .config
        .clone()
        .or_else(FileConfig::default_path)
        .unwrap_or_else(|| "logcli-config-unresolved.toml".into());
    let file_config = FileConfig::load(&config_path)?;

    let color_mode = resolve_color_mode(cli.color, &file_config);
    let use_color = output::should_use_color(color_mode);
    let show_progress = !cli.no_progress && file_config.progress.unwrap_or(true);

    match cli.command {
        Command::Filter(args) => run_filter(args, &file_config, use_color, show_progress),
        Command::Stats(args) => run_stats(args, &file_config, use_color, show_progress),
        Command::Completions(args) => run_completions(args),
    }
}

/// ตัดสิน ColorMode สุดท้ายตาม precedence: CLI flag > config file > default (Auto)
fn resolve_color_mode(cli_value: Option<ColorMode>, file_config: &FileConfig) -> ColorMode {
    let from_config = file_config.color.as_deref().and_then(|s| match s {
        "always" => Some(ColorMode::Always),
        "never" => Some(ColorMode::Never),
        "auto" => Some(ColorMode::Auto),
        _ => None,
    });
    config::resolve(cli_value, from_config, ColorMode::Auto)
}

fn resolve_format(cli_value: Option<OutputFormat>, file_config: &FileConfig) -> OutputFormat {
    let from_config = file_config.format.as_deref().and_then(|s| match s {
        "json" => Some(OutputFormat::Json),
        "human" => Some(OutputFormat::Human),
        _ => None,
    });
    config::resolve(cli_value, from_config, OutputFormat::Human)
}

fn resolve_strict(cli_value: bool, file_config: &FileConfig) -> bool {
    if cli_value {
        return true;
    }
    file_config.strict.unwrap_or(false)
}
```

`run()` ทำสามอย่างตามลำดับก่อน dispatch ไป handler: (1) หา config path ที่จะใช้ (จาก `--config` หรือ default XDG path), (2) โหลด config file, (3) ตัดสิน `color_mode`/`show_progress` ที่เป็น "global" (ใช้ร่วมกันทุก subcommand) — ส่วน `strict`/`format` ที่เป็น per-subcommand (บาง subcommand อาจต้องการค่าต่างกัน) จะตัดสินแยกในแต่ละ handler แทน

ส่วน handler ของ `filter`:

```rust
fn run_filter(
    args: FilterArgs,
    file_config: &FileConfig,
    use_color: bool,
    show_progress: bool,
) -> Result<(), AppError> {
    let strict = resolve_strict(args.strict, file_config);
    let format = resolve_format(args.format, file_config);

    let since = parse_optional_timestamp("--since", args.since.as_deref())?;
    let until = parse_optional_timestamp("--until", args.until.as_deref())?;
    if let (Some(s), Some(u)) = (&since, &until) {
        if s > u {
            return Err(AppError::InvalidTimeRange {
                since: s.to_rfc3339(),
                until: u.to_rfc3339(),
            });
        }
    }

    let outcome = reader::read_log_entries(&args.file, strict, show_progress)?;

    let wanted_levels: Vec<String> = args.level.iter().map(|l| l.to_lowercase()).collect();

    let mut matched: Vec<&LogEntry> = outcome
        .entries
        .iter()
        .filter(|e| {
            if !wanted_levels.is_empty()
                && !wanted_levels.iter().any(|lvl| e.level_matches(lvl))
            {
                return false;
            }
            if let Some(needle) = &args.contains {
                if !e.message.contains(needle.as_str()) {
                    return false;
                }
            }
            if let Some(s) = &since {
                if e.timestamp < *s {
                    return false;
                }
            }
            if let Some(u) = &until {
                if e.timestamp > *u {
                    return false;
                }
            }
            true
        })
        .collect();

    matched.sort_by_key(|e| e.timestamp);
    if let Some(limit) = args.limit {
        matched.truncate(limit);
    }

    match format {
        OutputFormat::Human => print_filter_human(&matched, use_color),
        OutputFormat::Json => print_filter_json(&matched)?,
    }

    if outcome.skipped > 0 {
        eprintln!(
            "สรุป: ข้ามไป {} บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)",
            outcome.skipped
        );
    }

    Ok(())
}

fn print_filter_human(entries: &[&LogEntry], use_color: bool) {
    for e in entries {
        let ts = output::paint_dim(&e.timestamp.to_rfc3339(), use_color);
        let level = output::paint_level(&e.level, use_color);
        let level_padded = format!("{level:<5}");
        let extra: String = e
            .fields
            .iter()
            .map(|(k, v)| format!("{k}={v}"))
            .collect::<Vec<_>>()
            .join(" ");
        if extra.is_empty() {
            println!("{ts} {level_padded} {}", e.message);
        } else {
            println!("{ts} {level_padded} {} ({extra})", e.message);
        }
    }
}

fn print_filter_json(entries: &[&LogEntry]) -> Result<(), AppError> {
    let stdout = std::io::stdout();
    let mut handle = stdout.lock();
    for e in entries {
        let line = serde_json::to_string(e).map_err(|source| AppError::Internal(source.to_string()))?;
        writeln!(handle, "{line}").map_err(|source| AppError::UnexpectedIo { source })?;
    }
    Ok(())
}

fn parse_optional_timestamp(
    flag_name: &str,
    value: Option<&str>,
) -> Result<Option<DateTime<Utc>>, AppError> {
    match value {
        None => Ok(None),
        Some(raw) => {
            let parsed = DateTime::parse_from_rfc3339(raw)
                .map_err(|e| AppError::InvalidTimestamp {
                    value: format!("{flag_name} {raw}"),
                    reason: e.to_string(),
                })?
                .with_timezone(&Utc);
            Ok(Some(parsed))
        }
    }
}
```

และ handler ของ `stats` (สั้นกว่า เพราะ group-by ทำผ่าน `BTreeMap` ตรง ๆ):

```rust
fn run_stats(
    args: StatsArgs,
    file_config: &FileConfig,
    use_color: bool,
    show_progress: bool,
) -> Result<(), AppError> {
    let strict = resolve_strict(args.strict, file_config);
    let format = resolve_format(args.format, file_config);

    let outcome = reader::read_log_entries(&args.file, strict, show_progress)?;

    let mut counts: BTreeMap<String, usize> = BTreeMap::new();
    for entry in &outcome.entries {
        let key = entry
            .field_value(&args.by)
            .unwrap_or_else(|| "(missing)".to_string());
        *counts.entry(key).or_insert(0) += 1;
    }

    if counts.is_empty() && outcome.entries.is_empty() {
        eprintln!("คำเตือน: ไม่มี log entry ที่ parse ผ่านเลยในไฟล์นี้ — ผลสรุปจึงว่างเปล่า");
    }

    match format {
        OutputFormat::Human => print_stats_human(&args.by, &counts, use_color),
        OutputFormat::Json => print_stats_json(&counts)?,
    }

    if outcome.skipped > 0 {
        eprintln!(
            "สรุป: ข้ามไป {} บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)",
            outcome.skipped
        );
    }

    Ok(())
}

fn print_stats_human(by: &str, counts: &BTreeMap<String, usize>, use_color: bool) {
    let mut rows: Vec<(&String, &usize)> = counts.iter().collect();
    rows.sort_by(|a, b| b.1.cmp(a.1).then_with(|| a.0.cmp(b.0)));

    let total: usize = counts.values().sum();
    let header = output::paint_bold(&format!("{by:<20} {:>8} {:>8}", "count", "percent"), use_color);
    println!("{header}");
    for (key, count) in rows {
        let percent = if total == 0 {
            0.0
        } else {
            (*count as f64) * 100.0 / (total as f64)
        };
        println!("{key:<20} {count:>8} {percent:>7.1}%");
    }
    println!("{}", output::paint_dim(&format!("รวมทั้งหมด: {total} entries"), use_color));
}

fn print_stats_json(counts: &BTreeMap<String, usize>) -> Result<(), AppError> {
    let json = serde_json::to_string_pretty(counts)
        .map_err(|source| AppError::Internal(source.to_string()))?;
    println!("{json}");
    Ok(())
}
```

จุดที่ควรสังเกต: **`rows.sort_by(|a, b| b.1.cmp(a.1).then_with(|| a.0.cmp(b.0)))`** — เรียงจากจำนวนมากไปน้อย (`b.1.cmp(a.1)` สลับด้านเพื่อกลับทิศทาง) แต่ถ้าจำนวนเท่ากัน (`.then_with(...)`) ให้เรียงตามชื่อ key แทน (`a.0.cmp(b.0)` ทิศทางปกติ) — การมี tie-breaker แบบนี้สำคัญมากสำหรับ**ความสม่ำเสมอของผลลัพธ์**: ถ้าไม่มี tie-breaker ลำดับของ key ที่มีจำนวนเท่ากันจะขึ้นกับลำดับที่ `BTreeMap::iter()` คืนมา (ซึ่งจริง ๆ ก็คือเรียงตาม key อยู่แล้วเพราะเป็น `BTreeMap` — แต่การเขียน tie-breaker ชัดเจนทำให้โค้ดไม่พึ่งพา "ผลข้างเคียง" ของโครงสร้างข้อมูลที่อาจเปลี่ยนได้ในอนาคตถ้าเปลี่ยนไปใช้ `HashMap` แทน)

ผลลัพธ์จริงของทั้งสอง subcommand (ไฟล์ `sample.jsonl` มี 10 บรรทัด โดยบรรทัดที่ 6 เป็นข้อความธรรมดาที่ไม่ใช่ JSON เพื่อทดสอบการข้ามบรรทัดพัง):

```
$ logcli filter --level error sample.jsonl
คำเตือน: ข้ามบรรทัดที่ 6 ใน 'sample.jsonl' เพราะ parse ไม่ผ่าน: expected ident at line 1 column 2
2024-01-15T10:23:07+00:00 error database connection timeout (duration_ms=3001 endpoint="/api/orders" status=500)
2024-01-15T10:23:12+00:00 error upstream service unavailable (duration_ms=120 endpoint="/api/payments" status=503)
สรุป: ข้ามไป 1 บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)

$ logcli filter --level error --format json sample.jsonl
คำเตือน: ข้ามบรรทัดที่ 6 ใน 'sample.jsonl' เพราะ parse ไม่ผ่าน: expected ident at line 1 column 2
{"timestamp":"2024-01-15T10:23:07Z","level":"error","message":"database connection timeout","duration_ms":3001,"endpoint":"/api/orders","status":500}
{"timestamp":"2024-01-15T10:23:12Z","level":"error","message":"upstream service unavailable","duration_ms":120,"endpoint":"/api/payments","status":503}
สรุป: ข้ามไป 1 บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)

$ logcli stats --by endpoint sample.jsonl
คำเตือน: ข้ามบรรทัดที่ 6 ใน 'sample.jsonl' เพราะ parse ไม่ผ่าน: expected ident at line 1 column 2
endpoint                count  percent
/api/users                  4    44.4%
/api/orders                 3    33.3%
/api/payments               2    22.2%
รวมทั้งหมด: 9 entries
สรุป: ข้ามไป 1 บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)
```

สังเกตว่า `filter --format json` พิมพ์ออกมาเป็น **JSON Lines** (หนึ่ง object ต่อบรรทัด ไม่ใช่ array ห่อรวม `[...]`) — เลือกรูปแบบนี้เพราะต่อท่อไปยังเครื่องมือที่อ่าน JSON Lines ได้ทันที (เช่น `jq -c` ประมวลผลทีละบรรทัด หรือ `logcli filter ... | logcli filter ...` ต่อท่อ `logcli` เข้ากับตัวเองอีกชั้นได้ในทางทฤษฎีถ้าจะขยายให้อ่าน stdin ที่เป็น JSON Lines อยู่แล้ว) ต่างจาก `stats --format json` ที่พิมพ์เป็น **JSON object เดียว** (`{key: count}`) เพราะความหมายของผลลัพธ์คือ "สรุปหนึ่งชุด" ไม่ใช่ "รายการของ record หลายตัว" — การเลือกรูปแบบ JSON ให้ตรงกับ**ความหมายของข้อมูล** (ไม่ใช่ใช้รูปแบบเดียวกันทุกที่เพราะขี้เกียจคิด) เป็นรายละเอียดที่ทำให้เครื่องมือใช้งานร่วมกับเครื่องมืออื่นได้ลื่นไหลกว่า

### 107.10 Shell Completion ด้วย `clap_complete`

เครื่องมือ CLI ระดับ production แทบทุกตัวรองรับ **tab completion** — ผู้ใช้พิมพ์ `logcli fil<Tab>` แล้วได้ `logcli filter` เติมให้อัตโนมัติ, พิมพ์ `logcli filter --le<Tab>` แล้วได้ `--level` เติมให้ — สิ่งนี้ทำได้เพราะ shell (bash/zsh/fish/...) อ่าน "completion script" ที่บอกว่าโปรแกรมนี้มี subcommand/flag อะไรบ้าง crate `clap_complete` generate script เหล่านี้ให้เราได้ฟรีจากโครงสร้าง `Cli` ที่มีอยู่แล้ว (แหล่งข้อมูลเดียวกับที่ generate `--help` — ไม่มีโอกาส out-of-sync แบบเดียวกับที่ Part 59 อธิบายไว้เรื่อง `--help`)

```rust
fn run_completions(args: CompletionsArgs) -> Result<(), AppError> {
    let mut cmd = Cli::command();
    let name = cmd.get_name().to_string();
    clap_complete::generate(args.shell, &mut cmd, name, &mut std::io::stdout());
    Ok(())
}
```

`Cli::command()` มาจาก `use clap::CommandFactory;` — เมธอดนี้ (ที่ `#[derive(Parser)]` generate ให้เราโดยอัตโนมัติ ตามที่ Part 59 อธิบายไว้ว่า derive macro generate `impl` หลายตัวไม่ใช่แค่ `Parser` เดียว) คืน `clap::Command` แบบ builder-style ที่บรรยายโครงสร้างของ CLI ทั้งหมด (เหมือนกับที่ Part 59.2 แสดง Builder API ไว้เปรียบเทียบ — ที่จริง Derive API ก็ compile ลงไปเป็น `Command` แบบเดียวกันนี้เบื้องหลัง) `clap_complete::generate()` รับ `Command` ตัวนี้ไปวิเคราะห์แล้ว generate script ตาม syntax ของ shell ที่ระบุ

ผลลัพธ์จริงสำหรับ bash (แสดง 15 บรรทัดแรก):

```
$ logcli completions bash
_logcli() {
    local i cur prev opts cmd
    COMPREPLY=()
    if [[ "${BASH_VERSINFO[0]}" -ge 4 ]]; then
        cur="$2"
    else
        cur="${COMP_WORDS[COMP_CWORD]}"
    fi
    prev="$3"
    cmd=""
    opts=""

    for i in "${COMP_WORDS[@]:0:COMP_CWORD}"
    do
        case "${cmd},${i}" in
```

และสำหรับ zsh (10 บรรทัดแรก):

```
$ logcli completions zsh
#compdef logcli

autoload -U is-at-least

_logcli() {
    typeset -A opt_args
    typeset -a _arguments_options
    local ret=1

    if is-at-least 5.2; then
```

และ fish (8 บรรทัดแรก — สังเกตว่า syntax ต่างจาก bash/zsh สิ้นเชิง เพราะ fish ใช้ shell language ของตัวเองที่ไม่เหมือนใคร แต่ `clap_complete` แปลง `Command` เดียวกันให้ตรงกับ syntax เฉพาะของ shell แต่ละตัวให้เราโดยอัตโนมัติ):

```
$ logcli completions fish
# Print an optspec for argparse to handle cmd's options that are independent of any subcommand.
function __fish_logcli_global_optspecs
    string join \n config= color= no-progress h/help V/version
end

function __fish_logcli_needs_command
    # Figure out if the current invocation already has a command.
    set -l cmd (commandline -opc)
```

การติดตั้งจริงสำหรับผู้ใช้ (ตัวอย่าง bash — เพิ่มเข้า `~/.bashrc` หรือใช้ครั้งเดียวต่อ session):

```bash
# เพิ่มบรรทัดนี้ใน ~/.bashrc เพื่อให้ทำงานทุกครั้งที่เปิด shell ใหม่
echo 'source <(logcli completions bash)' >> ~/.bashrc

# หรือสำหรับ zsh (~/.zshrc)
echo 'source <(logcli completions zsh)' >> ~/.zshrc

# หรือสำหรับ fish (fish มีโฟลเดอร์ completion เฉพาะ ไม่ต้อง source เอง)
logcli completions fish > ~/.config/fish/completions/logcli.fish
```

`<SHELL>` ของ `logcli completions` ใช้ type `clap_complete::Shell` ตรง ๆ (ตามที่อธิบายไว้ในหัวข้อ 107.4) ทำให้ `clap` validate ชื่อ shell ให้เราฟรี — ถ้าพิมพ์ผิด:

```
$ logcli completions powrshell
error: invalid value 'powrshell' for '<SHELL>'
  [possible values: bash, elvish, fish, powershell, zsh]

  tip: a similar value exists: 'powershell'

For more information, try '--help'.
```

สังเกตว่า `clap` ยังแนะนำค่าที่ใกล้เคียง (`tip: a similar value exists: 'powershell'`) ให้อีกด้วย โดยที่เราไม่ได้เขียนโค้ดตรวจ "ค่าที่ใกล้เคียง" เองเลยแม้แต่บรรทัดเดียว — นี่คือกลไก suggestion ที่ `clap` มีในตัวสำหรับ `ValueEnum` ทุกตัว (ที่กับดักข้อ 5 ของ Part 59 พูดถึงสั้น ๆ ไว้แล้วว่า error message คุณภาพสูงกว่าที่เขียนเองได้)

### 107.11 การทดสอบ CLI แบบเต็มรูปแบบด้วย `assert_cmd` และ `predicates`

Part 33 สอนแนวคิด integration test ไว้แล้ว: ทดสอบ binary จากมุมมองผู้ใช้จริง ไม่ใช่เรียกฟังก์ชันภายในตรง ๆ — สำหรับเครื่องมือ CLI แนวคิดนี้สำคัญเป็นพิเศษ เพราะ**สิ่งที่ต้องทดสอบจริง ๆ คือ "พฤติกรรมของ binary ที่ compile แล้ว"** (stdout, stderr, exit code) ไม่ใช่แค่ "ฟังก์ชันข้างในคืนค่าถูกไหม" — กับดักข้อ 4 ของ Part 59 สาธิตไว้แล้วว่าการสลับลำดับ positional argument ทำให้โปรแกรม compile ผ่านปกติแต่พฤติกรรมจริงพังเงียบ ๆ ซึ่งมีแต่ integration test ที่รัน binary จริงเท่านั้นที่จับปัญหานี้ได้

crate `assert_cmd` ให้ API สำหรับรัน binary ของ crate ตัวเอง (`Command::cargo_bin("logcli")`) แล้ว assert ผลลัพธ์แบบ fluent style ผสานกับ `predicates` ที่ให้ตัวช่วยตรวจสอบ string (`contains`, `not`, ฯลฯ) แบบอ่านง่าย:

```rust
//! Integration test เต็มรูปแบบ: รัน binary จริงผ่าน `assert_cmd` แล้วตรวจ stdout/stderr/exit code
//! ตรงตามที่ผู้ใช้จริงจะเห็น — ไม่ใช่การเรียกฟังก์ชันภายในตรง ๆ แบบ unit test

use assert_cmd::Command;
use predicates::prelude::*;
use std::io::Write;
use tempfile::NamedTempFile;

/// สร้างไฟล์ log ตัวอย่างชั่วคราวที่มีทั้งบรรทัดดีและบรรทัดพัง 1 บรรทัด
/// คืนค่า `NamedTempFile` ไว้ (ต้องถือ handle ไว้ ไม่งั้นไฟล์จะถูกลบทิ้งก่อนใช้งาน)
fn sample_log_file() -> NamedTempFile {
    let mut file = NamedTempFile::new().expect("สร้าง temp file ไม่สำเร็จ");
    writeln!(
        file,
        r#"{{"timestamp":"2024-01-15T10:23:01Z","level":"info","message":"request handled","endpoint":"/api/users"}}"#
    )
    .unwrap();
    writeln!(
        file,
        r#"{{"timestamp":"2024-01-15T10:23:07Z","level":"error","message":"database connection timeout","endpoint":"/api/orders"}}"#
    )
    .unwrap();
    writeln!(file, "this line is not json at all").unwrap();
    writeln!(
        file,
        r#"{{"timestamp":"2024-01-15T10:23:12Z","level":"error","message":"upstream unavailable","endpoint":"/api/payments"}}"#
    )
    .unwrap();
    file
}

#[test]
fn filter_by_level_prints_only_matching_entries() {
    let log = sample_log_file();

    Command::cargo_bin("logcli")
        .unwrap()
        .args(["filter", "--level", "error", "--no-progress"])
        .arg(log.path())
        .assert()
        .success()
        .stdout(predicate::str::contains("database connection timeout"))
        .stdout(predicate::str::contains("upstream unavailable"))
        .stdout(predicate::str::contains("request handled").not());
}

#[test]
fn filter_json_format_outputs_valid_json_lines() {
    let log = sample_log_file();

    let output = Command::cargo_bin("logcli")
        .unwrap()
        .args(["filter", "--level", "error", "--format", "json", "--no-progress"])
        .arg(log.path())
        .assert()
        .success()
        .get_output()
        .stdout
        .clone();

    let text = String::from_utf8(output).unwrap();
    let lines: Vec<&str> = text.lines().collect();
    assert_eq!(lines.len(), 2, "ต้องมี error 2 entry เท่านั้น");
    for line in lines {
        let parsed: serde_json::Value = serde_json::from_str(line).expect("ต้อง parse เป็น JSON ได้");
        assert_eq!(parsed["level"], "error");
    }
}

#[test]
fn malformed_line_is_skipped_by_default_with_warning_on_stderr() {
    let log = sample_log_file();

    Command::cargo_bin("logcli")
        .unwrap()
        .args(["stats", "--by", "level", "--no-progress"])
        .arg(log.path())
        .assert()
        .success()
        .stderr(predicate::str::contains("ข้ามบรรทัดที่ 3"))
        .stderr(predicate::str::contains("ข้ามไป 1 บรรทัด"));
}

#[test]
fn strict_mode_fails_fast_on_first_malformed_line_with_exit_code_65() {
    let log = sample_log_file();

    Command::cargo_bin("logcli")
        .unwrap()
        .args(["filter", "--level", "error", "--strict", "--no-progress"])
        .arg(log.path())
        .assert()
        .failure()
        .code(65)
        .stderr(predicate::str::contains("ไม่ใช่ JSON ที่ถูกต้อง"));
}

#[test]
fn missing_input_file_exits_with_code_66_and_actionable_message() {
    Command::cargo_bin("logcli")
        .unwrap()
        .args(["filter", "--level", "error", "does-not-exist-anywhere.jsonl"])
        .assert()
        .failure()
        .code(66)
        .stderr(predicate::str::contains("ไม่พบไฟล์"));
}

#[test]
fn invalid_since_timestamp_exits_with_usage_code_64() {
    let log = sample_log_file();

    Command::cargo_bin("logcli")
        .unwrap()
        .args(["filter", "--since", "not-a-real-date", "--no-progress"])
        .arg(log.path())
        .assert()
        .failure()
        .code(64)
        .stderr(predicate::str::contains("RFC 3339"));
}

#[test]
fn since_after_until_is_rejected_as_invalid_range() {
    let log = sample_log_file();

    Command::cargo_bin("logcli")
        .unwrap()
        .args([
            "filter",
            "--since",
            "2024-01-15T23:00:00Z",
            "--until",
            "2024-01-15T00:00:00Z",
            "--no-progress",
        ])
        .arg(log.path())
        .assert()
        .failure()
        .stderr(predicate::str::contains("ต้องมาก่อน"));
}

#[test]
fn stats_by_endpoint_counts_groups_correctly() {
    let log = sample_log_file();

    Command::cargo_bin("logcli")
        .unwrap()
        .args(["stats", "--by", "endpoint", "--format", "json", "--no-progress"])
        .arg(log.path())
        .assert()
        .success()
        .stdout(predicate::str::contains(r#""/api/orders": 1"#))
        .stdout(predicate::str::contains(r#""/api/payments": 1"#))
        .stdout(predicate::str::contains(r#""/api/users": 1"#));
}

#[test]
fn missing_subcommand_shows_usage_and_exits_nonzero() {
    Command::cargo_bin("logcli")
        .unwrap()
        .assert()
        .failure()
        .stderr(predicate::str::contains("Usage: logcli"));
}

#[test]
fn version_flag_prints_version_and_exits_zero() {
    Command::cargo_bin("logcli")
        .unwrap()
        .arg("--version")
        .assert()
        .success()
        .stdout(predicate::str::contains("logcli"));
}

#[test]
fn completions_bash_generates_nonempty_script() {
    Command::cargo_bin("logcli")
        .unwrap()
        .args(["completions", "bash"])
        .assert()
        .success()
        .stdout(predicate::str::contains("_logcli()"));
}

#[test]
fn reading_from_stdin_with_dash_works() {
    Command::cargo_bin("logcli")
        .unwrap()
        .args(["filter", "--level", "info", "--no-progress", "-"])
        .write_stdin(
            r#"{"timestamp":"2024-01-15T10:23:01Z","level":"info","message":"from stdin"}"#,
        )
        .assert()
        .success()
        .stdout(predicate::str::contains("from stdin"));
}
```

ไล่ดูรูปแบบ (pattern) ที่ทดสอบครอบคลุมทุกมิติของโปรแกรม:

- **พฤติกรรมปกติ** (`filter_by_level_prints_only_matching_entries`, `stats_by_endpoint_counts_groups_correctly`) — ตรวจว่าผลลัพธ์ตรงตามที่คาด และที่ไม่ควรอยู่ก็ไม่อยู่ (`.not()` — สำคัญเท่ากับการเช็คว่า "มี" เพราะการทดสอบแค่ "ผลลัพธ์ที่ถูกมีอยู่" ไม่ได้การันตีว่า "ผลลัพธ์ที่ผิดไม่ได้หลุดมาด้วย")
- **ความทนทานต่อข้อมูลเสีย** (`malformed_line_is_skipped_by_default_with_warning_on_stderr`, `strict_mode_fails_fast_on_first_malformed_line_with_exit_code_65`) — ทดสอบทั้งสองโหมดของหัวข้อ 107.6 พร้อมกัน ยืนยันว่า default (ข้าม+เตือน) กับ `--strict` (หยุดทันที) ให้ผลต่างกันจริงตามสเปค
- **Exit code ที่ถูกต้อง** (`missing_input_file_exits_with_code_66...`, `invalid_since_timestamp_exits_with_usage_code_64`) — ทดสอบตรงว่า `AppError::exit_code()` แม็ปถูกจริง ไม่ใช่แค่ทดสอบว่า "fail" เฉย ๆ (การเรียก `.code(66)` เจาะจงเลขที่ต้องการ ไม่ใช่แค่ `.failure()` ที่ยอมรับ exit code ใดก็ได้ที่ไม่ใช่ 0)
- **infrastructure ของเครื่องมือเอง** (`version_flag_prints_version_and_exits_zero`, `completions_bash_generates_nonempty_script`, `missing_subcommand_shows_usage_and_exits_nonzero`) — ทดสอบว่าสิ่งที่ `clap` ให้มาโดยอัตโนมัติ (version, help, subcommand ที่ derive ไว้) ยังทำงานถูกต้องอยู่ (ป้องกัน regression ถ้ามีคนแก้ `#[command(...)]` โดยไม่ทันคิดว่าจะกระทบ)
- **การรับ input จาก stdin** (`reading_from_stdin_with_dash_works`) — ใช้ `write_stdin()` ของ `assert_cmd` ส่งข้อมูลเข้า stdin ของ process ที่รันขึ้นมาจริง ทดสอบ path `-` ที่กล่าวถึงในหัวข้อ 107.1 ว่าเป็นธรรมเนียม Unix สำคัญ

รัน test suite จริง (ผลลัพธ์จริงจากการรัน `cargo test`):

```
$ cargo test
   Compiling logcli v0.1.0 (/path/to/logcli)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 10.95s
     Running unittests src/main.rs (target/debug/deps/logcli-25c9fa936a6985c9)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/cli.rs (target/debug/deps/cli-fb3be8cdfccb9273)

running 12 tests
test filter_by_level_prints_only_matching_entries ... ok
test filter_json_format_outputs_valid_json_lines ... ok
test missing_input_file_exits_with_code_66_and_actionable_message ... ok
test completions_bash_generates_nonempty_script ... ok
test malformed_line_is_skipped_by_default_with_warning_on_stderr ... ok
test invalid_since_timestamp_exits_with_usage_code_64 ... ok
test missing_subcommand_shows_usage_and_exits_nonzero ... ok
test stats_by_endpoint_counts_groups_correctly ... ok
test reading_from_stdin_with_dash_works ... ok
test since_after_until_is_rejected_as_invalid_range ... ok
test strict_mode_fails_fast_on_first_malformed_line_with_exit_code_65 ... ok
test version_flag_prints_version_and_exits_zero ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
```

ทั้ง 12 test case ผ่านหมด ในเวลารวมไม่ถึง 30 มิลลิวินาที (ไม่รวมเวลา compile) — สังเกตว่าแต่ละ test สร้าง process ของ `logcli` ขึ้นมาจริงแล้วปิดไป (`Command::cargo_bin` ทำเช่นนี้ทุกครั้งที่เรียก) ซึ่งช้ากว่าการเรียกฟังก์ชันตรง ๆ ในหน่วยความจำเดียวกันมาก แต่**คุ้มค่า**เพราะได้ความมั่นใจระดับ "นี่คือพฤติกรรมที่ผู้ใช้จริงจะเห็น" แบบ 100% ไม่มีช่องว่างระหว่าง "โค้ดที่ทดสอบ" กับ "โค้ดที่ deploy จริง" เลย

### 107.12 Packaging และการเตรียม Binary สำหรับแจกจ่าย

โปรแกรมที่เขียนเสร็จแล้วต้อง**แจกจ่ายได้จริง** — ผู้ใช้ปลายทางไม่ได้ต้องการ source code หรือ `cargo run` พวกเขาต้องการ **binary ไฟล์เดียว**ที่ดาวน์โหลดมาแล้วรันได้ทันที มาดูสามเรื่องที่สำคัญ: ขนาด binary, static linking, และการเผยแพร่ผ่าน crates.io

**1) ปรับ `[profile.release]` เพื่อลดขนาด binary**

ค่า default ของ `cargo build --release` ให้ผลลัพธ์ที่เร็วกว่า debug build มาก แต่ยังไม่ได้ปรับให้ขนาดเล็กที่สุด ลองดูขนาดจริงก่อนและหลังปรับ:

```
$ cargo build --release
$ ls -lh target/release/logcli
-rwxr-xr-x 2 user user 2.1M  target/release/logcli
$ file target/release/logcli
target/release/logcli: ELF 64-bit LSB pie executable, x86-64, ... dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, ... not stripped
```

Binary ที่ได้มีขนาด **2.1MB** และยัง**ไม่ strip** (มี debug symbol ติดอยู่เต็มไปหมด ที่ไม่มีประโยชน์กับผู้ใช้ปลายทางเลย เพราะไม่มีการ debug ผ่าน `gdb` บนเครื่องผู้ใช้) เพิ่ม profile ต่อไปนี้ใน `Cargo.toml`:

```toml
[profile.release]
strip = true        # ลบ debug symbol ออกจาก binary หลัง compile
lto = true          # Link-Time Optimization — ให้ compiler มองเห็นทั้งโปรแกรมพร้อมกันตอน optimize
codegen-units = 1   # compile เป็นหน่วยเดียว (ช้าลงตอน compile แต่ optimize ได้ดีกว่า)
```

Build ใหม่อีกครั้ง:

```
$ cargo build --release
$ ls -lh target/release/logcli
-rwxr-xr-x 2 user user 1.4M  target/release/logcli
```

ขนาดลดจาก **2.1MB เหลือ 1.4MB** (ลดลงประมาณ 33%) — `strip = true` ลบ debug symbol ที่ไม่จำเป็นออก, `lto = true` ให้ compiler ตัด dead code ข้าม crate boundary ได้ดีขึ้น (เช่นฟังก์ชันของ `clap`/`chrono` ที่โปรแกรมเราไม่ได้เรียกใช้เลยจะถูกตัดทิ้งอย่างละเอียดกว่า optimize แบบ crate-by-crate ปกติ), `codegen-units = 1` แลกเวลา compile ที่นานขึ้นกับการที่ LLVM มองเห็นโค้ดทั้งโปรแกรมพร้อมกันตอน optimize (ปกติ Cargo แบ่งงาน compile เป็นหลาย "หน่วย" ทำงานพร้อมกันหลาย thread เพื่อความเร็ว แต่แลกกับการ optimize ที่ทำได้แค่ในแต่ละหน่วยแยกกัน)

**2) Static Linking (เมื่อจำเป็น)**

ตรวจสอบว่า binary ที่ได้ผูก (link) กับ library ระบบตัวไหนบ้าง:

```
$ ldd target/release/logcli
	linux-vdso.so.1 (0x00007f431f7a5000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f431f5ca000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f431f4e1000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f431f200000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f431f7a7000)
```

Binary นี้ผูกกับ `glibc` (ผ่าน `libc.so.6`) แบบ dynamic — แปลว่าเครื่องที่จะรัน binary นี้ **ต้องมี** `glibc` เวอร์ชันที่ตรงหรือใหม่กว่าที่ compile ไว้ ถ้าแจกจ่าย binary นี้ไปยังระบบที่ใช้ `glibc` คนละเวอร์ชัน (พบบ่อยเมื่อแจกจ่ายข้าม distro Linux ที่ต่างกันมาก เช่น compile บน Ubuntu ล่าสุดแล้วเอาไปรันบน distro ที่ค่อนข้างเก่ากว่า) อาจได้ error แบบ `version 'GLIBC_2.XX' not found` — วิธีแก้ที่ใช้กันทั่วไปคือ compile ด้วย **musl target** ที่ผูก C library เข้าไปใน binary แบบ static ทั้งหมด (ไม่พึ่งพา `glibc` ของระบบปลายทางเลย):

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
```

Binary ที่ได้จาก target นี้จะรันได้บนแทบทุก Linux distro โดยไม่ต้องสนใจเวอร์ชัน `glibc` ของเครื่องปลายทางเลย (แลกกับขนาด binary ที่ใหญ่ขึ้นเล็กน้อยเพราะต้องรวม libc เข้ามาด้วย) — เหมาะมากสำหรับแจกจ่ายผ่าน Docker image แบบ `FROM scratch` หรือ `FROM alpine` ที่ไม่มี `glibc` ติดตั้งมาให้เลยด้วยซ้ำ

**3) เผยแพร่ผ่าน crates.io**

ถ้าต้องการให้ผู้ใช้ติดตั้งผ่าน `cargo install logcli` ได้ตรง ๆ โดยไม่ต้อง clone repository เอง ต้องเผยแพร่ package ไปที่ crates.io — ก่อนเผยแพร่ควรเพิ่ม metadata ที่ขาดใน `Cargo.toml` (crates.io บังคับต้องมี `description`/`license` เป็นอย่างน้อย):

```toml
[package]
name = "logcli"
version = "0.1.0"
edition = "2021"
description = "วิเคราะห์และ filter log ที่เขียนเป็น JSON Lines"
license = "MIT"
repository = "https://github.com/yourname/logcli"
```

ตรวจสอบว่า package พร้อมเผยแพร่ (ทำ dry-run ก่อนเสมอ ไม่กระทบอะไรจริงบน crates.io):

```bash
cargo publish --dry-run
```

และเผยแพร่จริง (ต้อง `cargo login` ด้วย API token จาก crates.io ก่อนครั้งแรก):

```bash
cargo publish
```

หลังจากนั้นใครก็ติดตั้งได้ทันทีด้วย `cargo install logcli` โดยไม่ต้องมี source code เลย — **ข้อควรระวัง**: เวอร์ชันที่เผยแพร่ไปแล้วบน crates.io **ลบไม่ได้** (ทำได้แค่ `cargo yank` ที่ซ่อนจากการติดตั้งใหม่ แต่ยังดาวน์โหลดได้สำหรับคนที่ pin เวอร์ชันนั้นไว้อยู่แล้ว) จึงควรทดสอบให้ครบถ้วน (โดยเฉพาะรัน test suite เต็มรูปแบบจากหัวข้อ 107.11) ก่อน `cargo publish` เสมอ

### 107.13 Walkthrough แบบครบวงจร

มาดูภาพรวมทั้งหมดของ `logcli` ผ่าน session เดียวที่ใช้ทุกคุณสมบัติต่อกัน (รันจริงด้วย release binary หลัง build เสร็จ):

```
$ logcli --version
logcli 0.1.0

$ logcli filter --level warn --level error sample.jsonl
คำเตือน: ข้ามบรรทัดที่ 6 ใน 'sample.jsonl' เพราะ parse ไม่ผ่าน: expected ident at line 1 column 2
2024-01-15T10:23:05+00:00 warn  slow query detected (duration_ms=812 endpoint="/api/orders" status=200)
2024-01-15T10:23:07+00:00 error database connection timeout (duration_ms=3001 endpoint="/api/orders" status=500)
2024-01-15T10:23:12+00:00 error upstream service unavailable (duration_ms=120 endpoint="/api/payments" status=503)
2024-01-15T10:23:22+00:00 warn  rate limit approaching (duration_ms=15 endpoint="/api/users" status=200)
สรุป: ข้ามไป 1 บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)

$ logcli stats --by endpoint --format json sample.jsonl
คำเตือน: ข้ามบรรทัดที่ 6 ใน 'sample.jsonl' เพราะ parse ไม่ผ่าน: expected ident at line 1 column 2
{
  "/api/orders": 3,
  "/api/payments": 2,
  "/api/users": 4
}
สรุป: ข้ามไป 1 บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)

$ logcli filter --level info --contains handled --limit 3 sample.jsonl
คำเตือน: ข้ามบรรทัดที่ 6 ใน 'sample.jsonl' เพราะ parse ไม่ผ่าน: expected ident at line 1 column 2
2024-01-15T10:23:01+00:00 info  request handled (duration_ms=42 endpoint="/api/users" status=200)
2024-01-15T10:23:02+00:00 info  request handled (duration_ms=58 endpoint="/api/orders" status=200)
2024-01-15T10:23:09+00:00 info  request handled (duration_ms=39 endpoint="/api/users" status=200)
สรุป: ข้ามไป 1 บรรทัดที่ parse ไม่ผ่าน (ดูรายละเอียดคำเตือนด้านบน)

$ logcli filter --strict --level info sample.jsonl
logcli: log line ที่ 6 ในไฟล์ 'sample.jsonl' ไม่ใช่ JSON ที่ถูกต้อง: expected ident at line 1 column 2
  เนื้อหาที่พังคือ: this line is not json at all
$ echo "exit=$?"
exit=65

$ logcli stats --by level nope.jsonl
logcli: ไม่พบไฟล์ 'nope.jsonl' — ตรวจสอบว่าพาธถูกต้องและไฟล์มีอยู่จริง (ใช้ '-' ถ้าต้องการอ่านจาก stdin)
$ echo "exit=$?"
exit=66
```

Session นี้แสดงให้เห็นครบทุกมิติ: filter หลาย level พร้อมกัน, group-by แบบ JSON, ผสม `--contains`/`--limit`, พฤติกรรม best-effort ที่ข้ามบรรทัดพังพร้อมสรุปท้าย, โหมด `--strict` ที่หยุดทันทีด้วย exit code 65, และ error เรื่องไฟล์หาไม่พบด้วย exit code 66 — ทุก exit code ตรงตามตารางในหัวข้อ 107.1 เป๊ะ

## กับดักที่พบบ่อย (Common Pitfalls)

**1. `fn main() -> Result<(), Box<dyn Error>>` ทำให้ error message กลายเป็น Debug format และ exit code ตายตัวที่ 1 เสมอ**

มือใหม่ที่คุ้นกับการเขียน `?` ใน `main()` ตรง ๆ (ตามที่ Part 12 สอนไว้สำหรับโปรแกรมเล็ก) มักเขียนแบบนี้เมื่อโปรแกรมโตขึ้น:

```rust
use std::error::Error;
use std::fmt;

#[derive(Debug)]
struct NotFound;
impl fmt::Display for NotFound {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "ไม่พบไฟล์ที่ระบุ")
    }
}
impl Error for NotFound {}

fn main() -> Result<(), Box<dyn Error>> {
    Err(Box::new(NotFound))
}
```

ทดสอบจริง (compile และรันผ่านแล้ว):

```
$ naive_demo
Error: NotFound
$ echo "exit code: $?"
exit code: 1
```

สังเกตสองปัญหาที่เกิดพร้อมกัน: **(ก) ข้อความที่เห็นคือ `NotFound` (จาก `#[derive(Debug)]`) ไม่ใช่ `ไม่พบไฟล์ที่ระบุ` (จาก `impl Display`) ที่เราตั้งใจเขียนไว้ให้ผู้ใช้อ่าน** — เพราะ `std::process::Termination` (trait ที่ `main()` ใช้ตัดสินว่าจะพิมพ์/exit อย่างไรตอนคืน `Result`) เรียก `eprintln!("Error: {:?}", err)` เสมอ คือใช้ **Debug** ไม่ใช่ **Display** ไม่ว่าเราจะ implement `Display` ไว้สวยงามแค่ไหนก็ตาม **(ข) exit code เป็น 1 เสมอ** ไม่ว่า error จะเป็นชนิดไหน (ไฟล์หาไม่พบ, argument ผิด, I/O ล้มเหลว ทุกอย่างได้ 1 เหมือนกันหมด) — script ที่เรียกโปรแกรมนี้แยกแยะสาเหตุความล้มเหลวจาก exit code ไม่ได้เลย **วิธีแก้:** ให้ `main()` คืน `ExitCode` (จาก `std::process::ExitCode`) แล้วจัดการ `Result` ภายในเอง (พิมพ์ error ด้วย `{}`/Display เอง แล้วแปลงเป็น exit code ที่ต้องการเอง) แบบเดียวกับที่ `logcli` ทำในหัวข้อ 107.5 — วิธีนี้ `main()` ไม่มี `?` ตรง ๆ อีกต่อไป แต่ได้ควบคุมทั้ง message format และ exit code เต็มที่

**2. ใช้ `default_value` บน field ของ `clap` ที่ต้องผสานกับ config file — precedence พังเงียบ ๆ โดยไม่มี error หรือ warning ใด ๆ**

สมมติเผลอเขียน field `format` ของ `StatsArgs` (ในหัวข้อ 107.4) แบบนี้แทน (ให้ `clap` ใส่ default ไว้ในตัว ดูเหมือนสะดวกกว่า):

```rust
#[arg(long, value_enum, value_name = "FORMAT", default_value = "human")]
pub format: OutputFormat,   // ไม่ใช่ Option<OutputFormat> อีกต่อไป
```

โค้ด**compile ผ่านปกติ** ไม่มี warning อะไรเลย แต่ทดสอบจริงกับ config file ที่ตั้ง `format = "json"` ไว้:

```
$ cat ~/.config/logcli/config.toml
format = "json"

$ logcli stats --by level sample.jsonl
level                   count  percent
info                        4    44.4%
error                       2    22.2%
warn                        2    22.2%
debug                       1    11.1%
รวมทั้งหมด: 9 entries
```

ผลลัพธ์ออกมาเป็น **human table** ไม่ใช่ JSON ตามที่ตั้งไว้ในไฟล์ config เลย — config file ถูก**เมิน**อย่างเงียบ ๆ **สาเหตุ:** เมื่อ field เป็น `OutputFormat` ตรง ๆ (ไม่ใช่ `Option<OutputFormat>`) พร้อม `default_value`, `clap` จะใส่ค่า `Human` ให้เสมอไม่ว่าผู้ใช้พิมพ์ `--format` มาหรือไม่ — โค้ดของเราจึงแยกไม่ออกเลยว่า `cli_value` ที่ได้มา (`Human`) มาจาก "ผู้ใช้ตั้งใจพิมพ์ `--format human`" หรือ "ผู้ใช้ไม่ได้พิมพ์อะไรเลย `clap` เติม default มาเอง" — logic `resolve(cli_value, config_value, default)` จาก 107.7 จะเลือก `cli_value` (`Some(Human)`) ก่อน `config_value` เสมอ ทำให้ config file ไม่มีผลอะไรเลยไม่ว่าจะตั้งค่าอะไรไว้ **วิธีแก้:** field ของ `clap` ที่ต้องรองรับ precedence จาก config file **ต้องเป็น `Option<T>` โดยไม่มี `default_value`เสมอ** — ปล่อยให้ `None` แปลว่า "ผู้ใช้ไม่ได้ระบุ" อย่างแท้จริง แล้วค่อยตัดสิน default จริง ๆ ในโค้ดของเราเองหลังผสานกับ config file แล้ว (ตามที่ `logcli` ทำอยู่)

**3. วาด progress bar ไปที่ `stdout` แทน `stderr` — ข้อมูลที่ต่อท่อไปยังเครื่องมืออื่นเสี่ยงมีตัวอักษรควบคุมปนมา**

สมมติเผลอเขียน `progress_draw_target()` (หัวข้อ 107.6) แบบนี้ (ลืมแยก UI ออกจาก data stream):

```rust
fn progress_draw_target() -> ProgressDrawTarget {
    ProgressDrawTarget::stdout()   // ผิด! ควรเป็น stderr
}
```

ทดสอบด้วยการรันในสภาพแวดล้อมที่ `stdout` ของโปรแกรมถูกต่อเข้ากับ terminal จริงโดยตรง (จำลองด้วย pseudo-terminal เพื่อบังคับให้ `is_terminal()` เป็น `true`) แล้วดู raw byte ที่ถูกส่งไปจริง:

```
$ logcli stats --by level --format json big.jsonl   # stdout ต่อกับ terminal จริง
```

raw bytes ที่ระบบปลายทางได้รับจริง (ตัดมาบางส่วน แสดงด้วย `repr()` ของ Python เพื่อให้เห็นตัวอักษรควบคุมที่ปกติมองไม่เห็น):

```
b'\x1b[36m\xe2\xa0\x81\x1b[0m \xe0\xb8\xad\xe0\xb9\x88\xe0\xb8\xb2\xe0\xb8\x99 log... 1 \xe0\xb8\x9a...\r\x1b[2K...'
b'...196569 \xe0\xb8\x9a\xe0\xb8\xa3\xe0\xb8\xa3\xe0\xb8\x97\xe0\xb8\xb1\xe0\xb8\x94 (130,565.1967/s)                                      \r\x1b[2K{\r\n  "debug": 33329,\r\n  "error": 33196,\r\n  "info": 100142,\r\n  "warn": 33333\r\n}\r\n'
```

สังเกตบรรทัดสุดท้าย: **สปินเนอร์และ ANSI escape code (`\x1b[2K`, `\r`) อยู่ใน**สตรีมเดียวกัน**กับ JSON payload จริง (`{... "debug": 33329, ...}`) เชื่อมต่อกันไม่มีขอบเขตแยก** — ถ้าอะไรก็ตามที่อ่าน raw bytes จาก `stdout` โดยไม่ผ่านการ render ผ่าน terminal จริง (เช่น เครื่องมือ capture output แบบ pseudo-terminal, บาง CI runner ที่จัดสรร pty ให้ทุกคำสั่ง, หรือ `script`/`tee` ที่ทำงานหลัง pty) จะได้ byte stream ที่ปนตัวอักษรควบคุมเข้ากับข้อมูลจริง ทำให้ parse JSON ไม่ผ่าน **สาเหตุ:** `stdout` มีไว้สำหรับ "ข้อมูลที่โปรแกรมผลิต" ส่วน `stderr` มีไว้สำหรับ "สถานะที่มนุษย์ดู" — การผสมสองอย่างนี้เข้าสตรีมเดียวกันทำลายสมมติฐานพื้นฐานที่เครื่องมือ Unix ทุกตัวยึดถือ (`some_producer | some_consumer` ต้องได้ byte stream ที่สะอาดจาก `stdout` เสมอ) **วิธีแก้:** วาด progress/สถานะไปที่ `stderr` เสมอ (ตามที่ `logcli` ทำจริงในหัวข้อ 107.6) — แม้ในกรณีทดสอบเบื้องต้นที่ redirect ไปไฟล์ปกติหรือ pipe ธรรมดา indicatif เองจะ auto-detect ว่าไม่ใช่ terminal แล้วซ่อน progress ให้อยู่ดี (ซึ่งทำให้ bug นี้**ดูเหมือนไม่มีปัญหา**ในการทดสอบทั่วไป) แต่การแยกสตรีมตามธรรมเนียม Unix ตั้งแต่ต้นคือสิ่งที่ป้องกัน edge case ที่การ auto-detect นั้นช่วยไม่ได้ (เช่นเมื่อ `stdout` ของโปรแกรมถูกส่งผ่าน pty จริง ๆ) — อย่าพึ่งพา "โชคดีที่ library ป้องกันให้" แทนการออกแบบสตรีมให้ถูกต้องตั้งแต่ต้น

**4. field ที่จำเป็น (`message`) หายไปจากบรรทัด log — เห็น error `missing field` แต่ไม่รู้ว่าควรจัดการยังไง**

ทดสอบส่งบรรทัด log ที่ไม่มี field `message` (บางระบบ logging อาจลืมส่ง field นี้มา หรือส่งมาผิด schema):

```
$ echo '{"timestamp":"2024-01-15T10:23:01Z","level":"info"}' > missing_field.jsonl
$ logcli filter --level info --strict --no-progress missing_field.jsonl
logcli: log line ที่ 1 ในไฟล์ 'missing_field.jsonl' ไม่ใช่ JSON ที่ถูกต้อง: missing field `message` at line 1 column 51
  เนื้อหาที่พังคือ: {"timestamp":"2024-01-15T10:23:01Z","level":"info"}
$ echo "exit=$?"
exit=65
```

**สาเหตุ:** `LogEntry` (หัวข้อ 107.3) นิยาม `message: String` เป็น field ที่ **required** ตรง ๆ (ไม่ใช่ `Option<String>`) — เมื่อ `serde_json` deserialize บรรทัดที่ไม่มี key `message` เลย จะ error ทันทีตาม design ของ `serde` (Part 57 สอนไว้ว่า field ที่ไม่ใช่ `Option<T>` ถือว่า required เสมอ ไม่มี field ไหนเป็น optional โดยปริยาย) — ในโหมด non-strict บรรทัดนี้จะถูกข้ามไปเงียบ ๆ (เหมือนบรรทัดพังแบบอื่น) แต่ในโหมด `--strict` จะเห็น error ตรง ๆ แบบนี้ **นี่ไม่ใช่ bug** — เป็นพฤติกรรมที่ถูกต้องแล้วของการ validate schema แบบเข้มงวด แต่เป็นจุดที่ควร**ตัดสินใจอย่างมีสติ**ว่า field ไหนของ `LogEntry` ควร required จริง ๆ (`timestamp`, `level` อาจสมเหตุสมผลที่จะ required เสมอ) กับ field ไหนที่ระบบ logging บางระบบอาจไม่ส่งมา (เช่น `message` บางระบบอาจไม่มี ถ้าใช้ structured field ล้วน ๆ) — ถ้าต้องการรองรับกรณีนี้ ควรเปลี่ยนเป็น `message: Option<String>` แล้วจัดการค่า `None` ตอน print (เช่นแสดงเป็นค่าว่างหรือ `"(no message)"`) แทนการปล่อยให้ error ทั้งบรรทัด — ทางเลือกด้าน schema design แบบนี้ควรทำโดยรู้ตัวเสมอ ไม่ใช่ปล่อยให้เป็นผลข้างเคียงของการลืมเขียน `Option<T>`

**5. debug build กับ release build ให้ throughput ต่างกันมาก — ทดสอบ performance บน debug build แล้วสรุปผิด**

ทดสอบอ่านไฟล์ log ขนาด 200,000 บรรทัดด้วย debug build เทียบกับ release build:

```
$ cargo build          # debug (ไม่มี --release)
$ time ./target/debug/logcli stats --by level --no-progress big.jsonl > /dev/null
```

debug build ไม่มี optimization ใด ๆ เลย (แต่ละ bounds check, การ clone `String` ทุกครั้งที่ deserialize, ฯลฯ ทำงานแบบตรงไปตรงมาไม่มีการ inline/vectorize) ทำให้ประมวลผลไฟล์ขนาดใหญ่ **ช้ากว่า release build อย่างมีนัยสำคัญ** (จากที่วัดจริงในหัวข้อ 107.6 release build ประมวลผลได้เกิน 700,000 บรรทัด/วินาทีหลัง warm-up) **สาเหตุ:** `cargo build`/`cargo run` (ไม่มี `--release`) ใช้ `[profile.dev]` ที่ปิด optimization ไว้โดย default (`opt-level = 0`) เพื่อให้ **เวลา compile เร็วที่สุด** สำหรับ loop แก้โค้ด-รัน-แก้โค้ดระหว่างพัฒนา — ไม่ได้ออกแบบมาเพื่อความเร็วตอนรัน **วิธีแก้:** วัด performance และทดสอบพฤติกรรมกับไฟล์ขนาดใหญ่จริงด้วย `cargo build --release` เท่านั้น อย่าสรุปผล benchmark จาก debug build เด็ดขาด (เชื่อมกับกับดักข้อ 3 ของ Part 59 ที่เตือนไว้แล้วว่า debug build กับ release build ให้ผลต่างกันได้ในมิติอื่นด้วย เช่น `debug_assert!` ที่ทำงานเฉพาะ debug build — บทนี้เพิ่มมิติ "ความเร็ว" เข้าไปอีกมิติที่ต้องระวังเวลาเทียบผลระหว่างสองโหมดนี้)

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เพิ่ม subcommand ใหม่ชื่อ `validate` ที่รับ `<FILE>` เพียงตัวเดียว (ไม่มี option อื่น) — อ่านไฟล์ log ทั้งไฟล์ (ใช้ `reader::read_log_entries` เดิม โดยส่ง `strict = false`) แล้วพิมพ์สรุปว่า "อ่านได้ทั้งหมด N บรรทัด, ข้ามไป M บรรทัดที่ parse ไม่ผ่าน" ถ้า `M > 0` ให้ exit code เป็น 65 (เหมือนโหมด strict ของ subcommand อื่น) แม้จะไม่ได้ throw `AppError::MalformedLine` ตรง ๆ (hint: ใช้ `ReadOutcome.skipped` ที่มีอยู่แล้วมาตัดสินใน `run_validate()` เองว่าจะคืน `Ok(())` หรือ `Err(...)` — ไม่จำเป็นต้องเพิ่ม variant ใหม่ใน `AppError` ก็ได้ ถ้าเลือกคืน error ธรรมดาพร้อมข้อความสรุป)

2. **[กลาง]** เพิ่ม field `top: Option<usize>` ให้ `StatsArgs` (`#[arg(long, value_name = "N")]`) — เมื่อระบุ `--top 3` ให้ `stats` แสดงเฉพาะ 3 กลุ่มที่มีจำนวนมากที่สุด (หลังจากเรียงแล้ว) ทั้งในโหมด human และ JSON ทดสอบว่า `logcli stats --by endpoint --top 2 sample.jsonl` แสดงแค่ 2 endpoint ที่มี count สูงสุด และเขียน integration test เพิ่มใน `tests/cli.rs` ยืนยันพฤติกรรมนี้ (hint: ใช้ `rows.truncate(top)` หลังจาก `sort_by` ใน `print_stats_human` — สำหรับโหมด JSON ต้องแปลง `BTreeMap` เป็น `Vec` เรียงแล้ว truncate ก่อน serialize เพราะ `BTreeMap` เองไม่มีแนวคิด "ลำดับตามจำนวน" ในตัว มีแต่ลำดับตาม key)

3. **[ยาก]** ปัจจุบัน `field_value("by")` ใน `run_stats` คืน `"(missing)"` เงียบ ๆ ถ้า field ที่ระบุใน `--by` ไม่มีอยู่ใน log entry เลย (ไม่ error ให้ผู้ใช้รู้ตัวว่าอาจพิมพ์ชื่อ field ผิด) ให้ปรับพฤติกรรม: ถ้า**ทุก entry**ในไฟล์ไม่มี field ที่ระบุเลยแม้แต่ตัวเดียว (คือได้ `"(missing)"` 100% ของ entry ทั้งหมด) ให้เพิ่ม `AppError` variant ใหม่ (เช่น `AppError::FieldNeverFound { field: String }`) ที่บอกผู้ใช้ว่า field นี้ไม่ปรากฏในไฟล์เลยสักครั้ง พร้อม exit code ที่เหมาะสม (ควรเป็น USAGE = 64 เพราะเป็นความผิดของผู้ใช้ที่พิมพ์ชื่อ field ผิด) แต่ถ้า field นั้นพบใน**บาง**entry (ไม่ใช่ทั้งหมด) ให้ยังคงพฤติกรรมเดิม (`"(missing)"` สำหรับ entry ที่ไม่มี) เพราะนั่นอาจเป็นข้อมูลที่ถูกต้องแล้ว (hint: นับจำนวน entry ที่ `field_value()` คืน `None` เทียบกับจำนวน entry ทั้งหมดก่อนตัดสินใจ error หรือไม่ — ต้องรอให้อ่านไฟล์ทั้งไฟล์เสร็จก่อนถึงจะรู้ว่า field นั้น "ไม่พบเลยสักครั้ง" จริงหรือไม่)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ปัจจุบันถ้าไฟล์ config มีค่า `format` ที่ไม่รู้จัก (เช่น `format = "yaml"` ที่ไม่ใช่ `"human"`/`"json"`) ฟังก์ชัน `resolve_format()` จะคืน `None` เงียบ ๆ (ตกไปใช้ default `human`) โดยไม่แจ้งผู้ใช้เลยว่า config file มีค่าที่พิมพ์ผิด ให้ปรับปรุงระบบ config ทั้งหมดให้ validate ค่าตอนโหลด (ใน `FileConfig::load()` หรือจุดที่เหมาะสม) — ถ้า `format`/`color` ในไฟล์ config มีค่าที่ไม่รู้จัก ให้คืน `AppError` ใหม่ (เช่น `AppError::InvalidConfigValue { key: String, value: String, allowed: Vec<String> }`) ที่บอกชัดเจนว่า key ไหนผิด ค่าที่ใส่มาคืออะไร และค่าที่ใช้ได้มีอะไรบ้าง (ในรูปแบบเดียวกับที่ `clap` แสดง `[possible values: ...]` ให้กับ `ValueEnum`) ทดสอบด้วยไฟล์ config ที่มี `format = "yaml"` แล้วยืนยันว่าได้ error message ที่ระบุค่าที่ใช้ได้ครบถ้วน พร้อม exit code 78 (CONFIG) เหมือน error อื่นของ config file (hint: เขียนฟังก์ชัน helper ที่รับ `key: &str`, `value: &str`, `allowed: &[&str]` คืน `Result<T, AppError>` ใช้ซ้ำได้ทั้ง `format` และ `color` แทนการเขียน validation logic ซ้ำสองที่ — เชื่อมกับแนวคิด DRY ที่ Part 59.5.1 พูดถึงไว้)

## สรุป

บทนี้ประกอบร่างทุกเครื่องมือที่หลักสูตรสอนมาตลอด — `clap` (Part 59), error handling ด้วย `thiserror` (Part 30-31), `serde`/`flatten` (Part 57-58), structured logging ที่เป็น "ข้อมูลต้นทาง" ของเครื่องมือนี้ (Part 60), generics (Part 18-19), และ integration testing (Part 32-33) — ให้กลายเป็นเครื่องมือ command-line เดียวที่**ใช้งานได้จริง 100%** ไม่ใช่แค่ตัวอย่างสอนแนวคิดแยกส่วน จุดที่สำคัญที่สุดที่บทนี้เพิ่มเข้ามาเหนือ Part 59 คือมิติที่ทำให้เครื่องมือ "production-grade" จริง ๆ: การแม็ป error ไปเป็น **exit code ตามธรรมเนียม `sysexits.h`** ที่ script อื่นพึ่งพาได้, ความทนทานต่อข้อมูลเสียที่ไม่ทำให้โปรแกรมทั้งตัวล้มเพราะบรรทัดเดียวพัง (พร้อมทางเลือก `--strict` สำหรับ workflow ที่ต้องการความเข้มงวดกว่า), progress bar ที่แยกออกจาก data stream อย่างถูกต้องตามธรรมเนียม Unix, config file ที่ผสานกับ CLI flag ด้วย precedence ที่ชัดเจน (และกับดักที่แอบซ่อนอยู่ถ้าใช้ `default_value` ผิดที่), output สองรูปแบบที่เคารพทั้งมนุษย์ (สี, `NO_COLOR`) และเครื่องจักร (JSON ที่ parse ได้แน่นอน), shell completion ที่ generate จากแหล่งข้อมูลเดียวกับ `--help` เสมอ, integration test ที่รัน binary จริงแทนการเรียกฟังก์ชันภายใน, และการเตรียม binary สำหรับแจกจ่ายจริง (ลดขนาด, static linking, เผยแพร่ผ่าน crates.io)

ทุกตัวอย่างในบทนี้ compile และรันจริงแล้วระหว่างเตรียมบทเรียน (รวมการทดสอบกับไฟล์ log ขนาด ~29MB ที่มี 200,000 บรรทัด) และ test suite ทั้ง 12 test case ผ่านครบ — นี่คือมาตรฐานที่เครื่องมือ CLI ระดับ production ทุกตัวต้องผ่านก่อนถึงมือผู้ใช้จริง ในบทถัดไป **Part 108: Capstone: Building a Production-Grade Web Service** เราจะย้ายจากโลกของ command-line ไปสู่โลกของ**เครือข่าย** — สร้าง web service ระดับ production เต็มรูปแบบที่ต้องจัดการกับความซับซ้อนอีกชุดหนึ่งที่ CLI tool ไม่ต้องเจอ (concurrent request หลายพันตัวพร้อมกัน, database connection pooling, HTTP status code ที่ถูกต้อง, middleware, graceful shutdown) โดยยังคงใช้หลักการพื้นฐานเดียวกันที่บทนี้ตอกย้ำไว้: error handling ที่ชัดเจน, การทดสอบที่ครอบคลุมพฤติกรรมจริง, และการออกแบบที่คำนึงถึงผู้ใช้ปลายทางเป็นหลัก

---

**Part ก่อนหน้า:** [Rust Design Patterns สำหรับ Enterprise Applications](part-106-enterprise-design-patterns.md) | **Part ถัดไป:** [Capstone: Building a Production-Grade Web Service](part-108-capstone-web-service.md)
