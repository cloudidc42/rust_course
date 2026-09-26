# Part 59: CLI Applications ด้วย clap

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 200 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่าทำไมการ parse `std::env::args()` เองด้วยมือถึงกลายเป็นปัญหาซับซ้อนอย่างรวดเร็ว (การจัดการ `--flag value` vs `--flag=value`, short flag รวมกัน, การ validate, การ generate `--help`) และทำไม ecosystem ของ Rust จึงมี `clap` เป็นคำตอบโดยพฤตินัย (de facto standard)
- ใช้ **derive API** ของ `clap` (`#[derive(Parser)]`) เพื่อแปลง struct ธรรมดาให้กลายเป็น CLI argument parser แบบสมบูรณ์ พร้อมอธิบายเชื่อมโยงกับความรู้ proc macro จาก Part 44-45 ว่าเบื้องหลังมันทำอะไรอยู่จริง ๆ
- ใช้ attribute ของ `clap` ที่สำคัญที่สุด — `#[arg(short, long)]`, `#[arg(default_value = "...")]`, `#[arg(value_name = "...")]`, `#[command(name, version, author, about)]` — เพื่อควบคุมทั้ง behavior การ parse และหน้าตาของ `--help`/`--version`
- ออกแบบ CLI ที่มี **subcommand** หลายตัวด้วย `#[derive(Subcommand)]` บน `enum` (เชื่อมโยงกับ Part 10 เรื่อง enum และ pattern matching แบบ exhaustive) แบบเดียวกับที่ `git commit`/`git push`/`git status` ทำงานจริง
- เขียน **custom validation logic** ด้วย `value_parser` เพื่อตรวจสอบเงื่อนไขที่ซับซ้อนกว่าการ parse type ธรรมดา (เช่น ช่วงตัวเลข, รูปแบบวันที่) และเข้าใจว่า error message ที่ `clap` แสดงให้ผู้ใช้ปลายทางนั้นมีคุณภาพสูงกว่าการเขียน validation เองมากเพียงใด
- ใช้ `#[arg(env = "...")]` เพื่อรองรับการตั้งค่าผ่าน environment variable แบบ 12-factor app และผสาน `clap` เข้ากับ `serde` (Part 57-58) และ error handling (Part 12, 30-31) เพื่อสร้างเครื่องมือ command-line ระดับใช้งานจริงที่จบครบทั้ง CLI parsing, config loading, persistent storage, และ exit code ที่ถูกต้อง

## ความรู้ที่ต้องมีมาก่อน

- **Part 44 (Procedural Macros เบื้องต้น) และ Part 45 (Derive Macros ขั้นสูง)** — นี่คือ prerequisite ที่สำคัญที่สุดของบทนี้ ทั้งสอง Part สอนไว้แล้วว่า derive macro ทำงานอย่างไรเบื้องหลัง: parse struct ด้วย `syn::DeriveInput`, วิเคราะห์ field/attribute, และ generate `impl` block ด้วย `quote!{}` — `#[derive(Parser)]` ของ `clap` **คือตัวอย่างจริงของสิ่งที่ Part 44-45 สอน** ไม่ใช่กลไกใหม่ที่ต้องเรียนเพิ่ม คุณรู้อยู่แล้วว่ามันต้อง parse struct field แต่ละตัวออกมา (ชื่อ, type, attribute) แล้ว generate โค้ดที่ implement trait `Parser` ให้ — บทนี้จะไม่อธิบายกลไก proc macro ซ้ำ แต่จะอ้างอิงกลับไปที่ Part 44-45 ทุกครั้งที่พฤติกรรมของ `clap` ต้องอาศัยความเข้าใจนั้น
- **Part 57 (Serde เบื้องต้น) และ Part 58 (Serde ขั้นสูง)** — โครงสร้าง attribute ของ `clap` (เช่น `#[arg(short, long, default_value = "...")]`) มีรูปแบบทาง "ปรัชญาการออกแบบ" เหมือนกับ attribute ของ `serde` (เช่น `#[serde(rename = "...", default)]`) เพราะทั้งคู่เป็น derive macro ที่ใช้ attribute เพื่อ **ปรับแต่ง code generation ต่อ field** แบบเดียวกัน — ในหัวข้อท้ายบทเราจะผสาน `clap` (parse CLI arguments) กับ `serde` (deserialize ไฟล์ config/ข้อมูล) เข้าด้วยกันในโปรแกรมเดียว ซึ่งเป็นรูปแบบที่พบได้ในเครื่องมือ command-line ระดับ production แทบทุกตัว
- **Part 10 (Enums และ Pattern Matching)** — `#[derive(Subcommand)]` แปลง `enum` ให้กลายเป็นชุดของ subcommand โดยตรง แต่ละ variant คือ subcommand หนึ่งตัว และเมื่อ parse เสร็จแล้ว คุณจะ `match` บน enum นั้นเพื่อจัดการแต่ละคำสั่ง — Rust compiler การันตี **exhaustiveness** ของ `match` แบบเดียวกับที่ Part 10 สอนไว้ ทำให้ถ้าคุณเพิ่ม subcommand ใหม่ (เพิ่ม variant ใหม่) แล้วลืมจัดการใน `match` compiler จะฟ้อง error ทันที ไม่ใช่ bug ที่ไปโผล่ตอน runtime
- **Part 12 (Result<T, E> และ Error Handling เบื้องต้น)** — เรื่อง `?` operator, การคืนค่า `Result` จาก `main()`, และแนวคิดพื้นฐานเรื่อง exit code ของโปรแกรม (0 = สำเร็จ, ไม่ใช่ 0 = ล้มเหลว) เป็นพื้นฐานที่บทนี้ใช้ตลอดหัวข้อ 59.8 ที่ต้องเลือก exit code ให้เหมาะกับแต่ละสาเหตุความล้มเหลว (CLI arg ผิด, ไฟล์ config หาไม่พบ, ไฟล์ config รูปแบบผิด)
- **Part 32 (Testing: Unit Tests) และ Part 33 (Testing: Integration Tests)** — หัวข้อ 59.11 ใช้ `Cli::try_parse_from(...)` เขียนเป็น `#[test]` เพื่อทดสอบ logic การ parse argument ล้วน ๆ แบบเดียวกับที่ Part 32 สอนไว้ (`assert_eq!`, `#[cfg(test)] mod tests`) และกับดักข้อ 4 (ลำดับ positional argument) อ้างอิงถึงแนวคิด integration test จาก Part 33 ที่ทดสอบ binary ทั้งตัวจากมุมมองผู้ใช้จริง
- **Part 30 (Error Handling ขั้นสูง) และ Part 31 (thiserror, anyhow)** — เรื่อง custom error type, การแปลง error ด้วย `From`/`Into`, และรูปแบบการจัดการ error แบบมืออาชีพในโปรแกรมจริง จะถูกนำมาใช้ตอนที่เราต้องรวม error จากสามแหล่งต่างกัน (การ parse CLI arguments ของ `clap` เอง, การอ่านไฟล์ของ `std::fs`, และการ deserialize ของ `serde_json`) ให้ผู้ใช้เห็น error message ที่ชัดเจนและได้ exit code ที่ถูกต้องเสมอ

## เนื้อหา

บทนี้โฟกัสที่ **Derive API** ของ `clap` ตลอดทั้งบท (ตามที่อธิบายเหตุผลไว้ในหัวข้อ 59.2) และครอบคลุมสิ่งที่ CLI application ส่วนใหญ่ในโลกจริงต้องใช้: การประกาศ argument/flag พื้นฐาน, subcommand, custom validation, environment variable fallback, และการผสานกับ `serde`/error handling จนได้โปรแกรมที่ persist ข้อมูลได้จริง ส่วนหัวข้อขั้นสูงกว่านี้ที่ `clap` รองรับได้ด้วย (เช่น shell completion script ผ่าน crate เสริม `clap_complete`, การ generate man page ผ่าน `clap_mangen`, หรือ `ArgGroup` แบบซับซ้อนที่มีมากกว่าสองตัวเลือก) จะไม่ลงรายละเอียดในบทนี้ เพราะเป็นเครื่องมือเสริมที่สร้างต่อบนความเข้าใจพื้นฐานที่บทนี้ปูไว้ — เมื่อเข้าใจ Derive API หลักแล้ว การอ่าน documentation ของส่วนขยายเหล่านั้นจะไม่ใช่เรื่องยากอีกต่อไป

### 59.1 ทำไมต้องมี crate เฉพาะสำหรับ parse CLI arguments

ก่อนจะดูว่า `clap` ช่วยอะไรได้บ้าง เรามาดูก่อนว่าถ้า**ไม่ใช้มัน**แล้วเขียนโปรแกรม parse argument ด้วยมือเอง จะต้องจัดการกับอะไรบ้าง สมมติเราต้องการเขียนเครื่องมือค้นหาข้อความในไฟล์แบบง่าย ๆ ที่รับ pattern ที่จะค้นหา (positional argument, จำเป็น), และมี flag สองตัว: `-n`/`--lines` (แสดงเลขบรรทัด) กับ `-i`/`--ignore-case` (ไม่สนตัวพิมพ์เล็ก/ใหญ่)

```rust
use std::env;
use std::process::ExitCode;

/// ตัวอย่าง "เขียน parser เองด้วยมือ" โดยไม่พึ่ง clap เพื่อให้เห็นว่าปัญหาที่ต้องจัดการมีอะไรบ้าง
/// รองรับ: pattern (positional, required), --lines/-n (flag), --ignore-case/-i (flag)
/// ตั้งใจ "แค่พอทำงานได้" เพื่อโชว์ปริมาณโค้ดที่ต้องเขียนเอง ไม่ใช่ตัวอย่างระดับ production
fn main() -> ExitCode {
    let args: Vec<String> = env::args().skip(1).collect();

    let mut pattern: Option<String> = None;
    let mut show_lines = false;
    let mut ignore_case = false;

    let mut i = 0;
    while i < args.len() {
        let arg = &args[i];

        if arg == "--lines" || arg == "-n" {
            show_lines = true;
        } else if arg == "--ignore-case" || arg == "-i" {
            ignore_case = true;
        } else if arg == "-ni" || arg == "-in" {
            // ต้องดักกรณี "short flag รวมกัน" เองแบบตรง ๆ ทีละคู่ที่เป็นไปได้
            show_lines = true;
            ignore_case = true;
        } else if let Some(value) = arg.strip_prefix("--pattern=") {
            // ต้องดักกรณี --flag=value เองแยกจากกรณี --flag value
            pattern = Some(value.to_string());
        } else if arg == "--pattern" {
            i += 1;
            match args.get(i) {
                Some(v) => pattern = Some(v.clone()),
                None => {
                    eprintln!("error: --pattern ต้องมีค่าตามมาด้วย");
                    return ExitCode::from(2);
                }
            }
        } else if !arg.starts_with('-') {
            // ไม่มี prefix "--pattern" แต่เป็น positional argument ตรง ๆ
            pattern = Some(arg.clone());
        } else {
            eprintln!("error: ไม่รู้จัก argument '{arg}'");
            return ExitCode::from(2);
        }

        i += 1;
    }

    let pattern = match pattern {
        Some(p) => p,
        None => {
            // เขียน error message เองทั้งหมด ไม่มี "--help" ให้อัตโนมัติ
            eprintln!("error: ต้องระบุ pattern ที่จะค้นหา");
            eprintln!("usage: manual_parse <PATTERN> [--lines] [--ignore-case]");
            return ExitCode::from(2);
        }
    };

    println!("pattern = {pattern}, show_lines = {show_lines}, ignore_case = {ignore_case}");
    ExitCode::SUCCESS
}
```

ทดสอบจริง (compile และรันผ่านแล้ว):

```
$ manual_parse world -n -i
pattern = world, show_lines = true, ignore_case = true

$ manual_parse world -ni
pattern = world, show_lines = true, ignore_case = true

$ manual_parse --pattern=world --lines
pattern = world, show_lines = true, ignore_case = false

$ manual_parse
error: ต้องระบุ pattern ที่จะค้นหา
usage: manual_parse <PATTERN> [--lines] [--ignore-case]
```

โปรแกรมนี้**ทำงานได้จริง** แต่สังเกตปัญหาที่ซ่อนอยู่ ซึ่งจะเห็นชัดขึ้นเรื่อย ๆ ถ้าโปรแกรมมี argument มากกว่านี้:

**ปัญหาที่ 1 — คุณต้องเขียน case สำหรับทุก "รูปแบบ" ของการใส่ argument เอง** ผู้ใช้ CLI ทั่วไปคาดหวังว่าจะพิมพ์ `--pattern value`, `--pattern=value`, หรือใช้ positional ตรง ๆ ก็ได้ทั้งหมด — โค้ดข้างบนต้องมี `if`/`else if` แยกกันสำหรับแต่ละรูปแบบ และ short flag ที่รวมกันได้ (`-ni` ต้องเทียบเท่า `-n -i`) ต้องเขียนดักไว้**ล่วงหน้าทุกคู่ที่เป็นไปได้** — ถ้ามี flag แบบนี้ 5 ตัว จำนวนการรวมกันที่ต้องดักจะเพิ่มแบบ combinatorial ทันที (`-ni`, `-in`, `-nx`, `-xn`, `-nix`, ...) ซึ่งเป็นไปไม่ได้ที่จะเขียนดักครบทุกกรณีด้วยมือ

**ปัญหาที่ 2 — ไม่มี `--help` ให้เลย** ผู้ใช้โปรแกรมนี้ไม่มีทางรู้ตัวเลือกทั้งหมดนอกจากไปอ่าน source code หรือ documentation แยก ถ้าอยากได้ `--help`/`-h` ที่แสดง usage, รายชื่อ argument พร้อมคำอธิบาย, แสดง default value — ต้องเขียนเองทั้งหมด และต้องคอยอัปเดตให้ตรงกับโค้ด parsing จริงทุกครั้งที่แก้ไข (ซึ่งมักจะ**ไม่ตรงกัน**ในโปรแกรมจริงเมื่อเวลาผ่านไป เพราะไม่มีอะไรบังคับให้ทั้งสองส่วน sync กัน)

**ปัญหาที่ 3 — error message ไม่สม่ำเสมอและต้องเขียนเอง** สังเกตว่า error ของ "ไม่รู้จัก argument" กับ error ของ "pattern จำเป็นแต่ไม่ได้ใส่" ใช้ format ข้อความคนละแบบ (อันหนึ่งมี usage บอกด้วย อันหนึ่งไม่มี) เพราะเราเขียนแต่ละ error message แยกกันด้วยมือ ยิ่งโปรแกรมโตขึ้น ความไม่สม่ำเสมอนี้จะยิ่งเห็นชัด — ต่างจาก `clap` ที่ generate error message ทุกจุดด้วย format เดียวกันเสมอ (เราจะเห็นในหัวข้อถัดไป)

**ปัญหาที่ 4 — การ validate ค่าที่ซับซ้อนกว่า string ต้องเขียนเอง** ถ้าต้องการให้ argument ตัวหนึ่งเป็นตัวเลขในช่วงที่กำหนด หรือเป็นค่าจากชุดตัวเลือกที่จำกัด (เช่น `low`/`medium`/`high`) โค้ดข้างบนยังไม่ได้แตะเรื่องนี้เลย — คุณต้องเพิ่ม `.parse::<T>()` เอง จัดการ `Result` เอง เขียน error message เองสำหรับทุกกรณีที่ parse ไม่ผ่าน

**เทียบกับภาษาอื่น: ปัญหานี้ไม่ใช่เรื่องใหม่เฉพาะ Rust** ทุกภาษาที่มี ecosystem สำหรับเขียน CLI จริงจังต่างมี library แบบเดียวกับ `clap` เกิดขึ้นด้วยเหตุผลเดียวกันทั้งหมด — Python มี `argparse` (built-in ในตัวภาษาเลย เพราะปัญหานี้พบบ่อยมากจนถูกดึงเข้า standard library) ที่ให้คุณประกาศ argument ผ่าน `parser.add_argument("--lines", action="store_true")` แทนการ parse `sys.argv` เอง, Node.js มี `commander`/`yargs` ที่ให้ประกาศผ่าน method chaining คล้าย Builder API ของ `clap`, Go มี package `flag` ในตัวภาษา (จำกัดกว่า `clap` มาก ไม่รองรับ positional argument ที่ซับซ้อนหรือ subcommand ได้ดีเท่า) ที่ทำให้โปรเจกต์ Go จำนวนมากไปใช้ `cobra` (ซึ่งเป็น library ที่ `clap` ในยุคหลังได้รับแรงบันดาลใจด้าน subcommand design มาไม่น้อย) — สิ่งที่ทำให้ `clap` โดดเด่นกว่า library เทียบเท่าในภาษาอื่นคือการใช้ **derive macro** ผสานกับ **type system ของ Rust** (type-directed parsing ที่จะเห็นในหัวข้อ 59.3) ทำให้ "รูปร่างของ argument" กับ "type ที่โค้ดจะได้ใช้จริง" เป็นสิ่งเดียวกันเสมอ ไม่มีช่องว่างที่ argument ผ่านการ parse มาแล้วแต่ type ในโค้ดไม่ตรงกับที่ parser ประกาศไว้ (เช่นปัญหาที่พบได้ใน `argparse` ที่ type ของค่าที่ parse ได้เป็นแค่ `Any` ในทางปฏิบัติ ต้อง cast/assume type เอง)

**สรุปหัวข้อนี้:** การ parse `std::env::args()` เองด้วยมือ**ทำได้ในทางทฤษฎี** สำหรับ CLI ที่เล็กที่สุด แต่ปัญหาทั้ง 4 ข้อข้างบนจะโตขึ้นแบบไม่เป็นเส้นตรง (non-linear) เมื่อโปรแกรมมี argument/flag/subcommand มากขึ้น — นี่คือช่องว่างที่ `clap` เข้ามาเติมเต็ม: มันจัดการ**ทุกรูปแบบการใส่ argument**, **generate `--help`/`-h` ที่ sync กับโค้ด parsing เสมอ (เพราะมันคือแหล่งข้อมูลเดียวกัน)**, **generate error message ที่สม่ำเสมอและอ่านง่าย**, และ**รองรับการ validate ที่ซับซ้อน**ผ่าน type system ของ Rust เอง — ทั้งหมดนี้แบบ**declarative** คือคุณแค่**บอกว่า** argument มีอะไรบ้าง ไม่ต้องเขียน**วิธีการ parse**เอง

### 59.2 ติดตั้ง `clap`: Derive API เทียบกับ Builder API

`clap` (Command Line Argument Parser) มี API ให้เลือกใช้ 2 รูปแบบหลัก:

1. **Derive API** — ใช้ `#[derive(Parser)]` บน struct (แบบที่บทนี้จะโฟกัสทั้งบท) เขียนโค้ดน้อยที่สุด อ่านเข้าใจง่ายที่สุด เพราะโครงสร้างของ argument **คือโครงสร้างของ struct โดยตรง** เป็นวิธีที่ `clap` เอกสารทางการแนะนำให้ใช้เป็นค่าเริ่มต้นสำหรับโปรเจกต์ใหม่แทบทั้งหมด
2. **Builder API** — เรียก method เช่น `Command::new("myapp").arg(Arg::new("pattern").required(true))` แบบ imperative (สร้าง object แล้วเรียก method ต่อกันไปเรื่อย ๆ) เหมาะกับกรณีที่ต้อง**สร้าง argument แบบ dynamic ตอน runtime** (เช่น จำนวน/ชนิดของ argument ไม่รู้ล่วงหน้าตอน compile time ขึ้นกับ config ไฟล์อื่นที่โหลดมา) ซึ่งเป็นกรณีที่พบไม่บ่อยในโปรเจกต์ทั่วไป

บทนี้จะไม่ลงรายละเอียด Builder API เพราะโปรแกรม CLI ส่วนใหญ่ที่คุณจะเขียน**รู้โครงสร้าง argument ล่วงหน้าตอน compile time อยู่แล้ว** ซึ่งเป็นกรณีที่ Derive API เหมาะสมที่สุดและเป็นที่นิยมที่สุดในระบบนิเวศ (แม้แต่เครื่องมือชื่อดังอย่าง `ripgrep`, `bat`, `fd` ก็ใช้ Derive API เป็นหลัก) เพื่อให้เห็นหน้าตาของ Builder API สั้น ๆ (ไม่ลงรายละเอียดต่อ) ตัวอย่างที่เทียบเท่ากับ struct `Cli { name: String, verbose: bool }` แบบ Derive API เขียนด้วย Builder API ได้ดังนี้:

```rust
use clap::{Arg, Command};

fn main() {
    let matches = Command::new("builder_demo")
        .about("ตัวอย่าง Builder API แบบสั้น ๆ เพื่อเปรียบเทียบกับ Derive API")
        .arg(
            Arg::new("name")
                .help("ชื่อที่จะทาย")
                .required(true),
        )
        .arg(
            Arg::new("verbose")
                .short('v')
                .long("verbose")
                .help("แสดงรายละเอียดเพิ่มเติม")
                .action(clap::ArgAction::SetTrue),
        )
        .get_matches();

    let name = matches.get_one::<String>("name").unwrap();
    let verbose = matches.get_flag("verbose");
    println!("name = {name}, verbose = {verbose}");
}
```

```
$ builder_demo Alice --verbose
name = Alice, verbose = true
```

สังเกตความต่างที่ชัดเจน: Builder API ต้องเรียก `.get_one::<String>("name")` เพื่อดึงค่าออกมา ซึ่ง**ผูกชื่อ argument ด้วย string literal** (`"name"`) ที่ต้อง**ตรงกับชื่อที่ตั้งไว้ตอนสร้าง `Arg::new("name")` เป๊ะ** — ถ้าพิมพ์ผิดหรือชื่อไม่ตรงกัน compiler จะไม่ฟ้อง error เลยเพราะเป็นแค่ string ธรรมดา ปัญหาจะไปโผล่ตอน runtime เป็น `panic` จาก `.unwrap()` เท่านั้น ต่างจาก Derive API ที่ผูก argument กับ **field ของ struct โดยตรง** ทำให้พิมพ์ชื่อผิดไม่ได้เลยเพราะ compiler จะฟ้อง error ทันทีถ้า field ไม่มีอยู่จริง — นี่คือเหตุผลเชิงลึกอีกข้อที่ Derive API ได้รับความนิยมมากกว่า Builder API ในโปรเจกต์ใหม่ ๆ แทบทั้งหมด: มันย้าย error จาก runtime (string ไม่ตรงกัน) มาเป็น compile-time (field ไม่มีอยู่จริง) ตามหลักการที่ Rust ยึดถือมาตลอดหลักสูตรนี้

ติดตั้งด้วยคำสั่งเดียว:

```bash
cargo add clap --features derive
```

`--features derive` **จำเป็น** เพราะ derive macro ของ `clap` (`Parser`, `Subcommand`, `ValueEnum`, `Args`) ไม่ได้ compile เข้ามาเป็น default — เหตุผลคือ `clap_derive` (proc-macro crate ที่อยู่เบื้องหลัง `#[derive(Parser)]`) เพิ่ม dependency อย่าง `syn`/`quote`/`proc-macro2` เข้ามาด้วย ซึ่งเพิ่มเวลา compile ของโปรเจกต์ที่ไม่ได้ใช้ Derive API เลย (เช่นโปรเจกต์ที่ตั้งใจใช้ Builder API แบบ pure) — นี่คือรูปแบบเดียวกับที่ Part 44 อธิบายไว้เรื่อง `proc-macro = true` crate: `clap_derive` คือ proc-macro crate แยกที่ `clap` (library หลัก) depend on ผ่าน optional dependency แล้ว re-export ผ่าน `pub use clap_derive::Parser;` เมื่อเปิด feature `derive` เท่านั้น — pattern เดียวกับ `serde`/`serde_derive` ที่คุณเห็นมาแล้วใน Part 57

`Cargo.toml` หลังรันคำสั่งข้างบนจะมีบรรทัดประมาณนี้:

```toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

### 59.3 `#[derive(Parser)]`: จาก struct ธรรมดาสู่ CLI argument parser

จาก Part 44-45 คุณรู้อยู่แล้วว่า derive macro ทำงานโดย parse struct ทั้งก้อนเป็น `syn::DeriveInput` แล้ว generate `impl Trait for TypeName { ... }` เพิ่มเข้ามาข้าง ๆ struct เดิม (ไม่แก้ไข struct เดิม) — `#[derive(Parser)]` ก็ทำแบบเดียวกันเป๊ะ: มันอ่านชื่อ struct, อ่าน field ทุกตัว (ชื่อ, type, attribute ที่ติดอยู่, doc comment), แล้ว generate `impl clap::Parser for Cli { ... }` ที่ข้างในมี logic การสร้าง `clap::Command` ที่มี argument ตรงกับ field ทุกตัว รวมถึง generate `impl clap::FromArgMatches for Cli` เพื่อแปลงผลลัพธ์การ parse ให้กลายเป็น struct ของคุณจริง ๆ

สิ่งที่ทำให้ Derive API ของ `clap` ทรงพลังคือ **type-directed parsing** — `clap` ดู**ชนิดของ field** เพื่อตัดสินใจว่า argument ตัวนั้นควรมี behavior แบบไหนโดยอัตโนมัติ:

| ชนิดของ field | `clap` ตีความเป็น | ตัวอย่าง |
|---|---|---|
| `String`, `u32`, `f64`, ... (type ธรรมดา) | argument ที่**จำเป็นต้องมีค่า** (required) และ parse ด้วย `FromStr` ของ type นั้น | `age: u32` → ต้องใส่ `--age 25` เสมอ ถ้าใส่ `--age abc` จะได้ error parse ทันที |
| `Option<T>` | argument ที่**ไม่จำเป็น** (optional) — ถ้าไม่ใส่ จะได้ `None` โดยอัตโนมัติ ไม่ต้องเขียน default เอง | `nickname: Option<String>` → ไม่ใส่ `--nickname` ก็ไม่ error |
| `bool` | **flag แบบสวิตช์** (ไม่รับค่าตามมา) — ใส่ `--flag` แล้วได้ `true`, ไม่ใส่แล้วได้ `false` โดยอัตโนมัติ | `verbose: bool` → `--verbose` เฉย ๆ ไม่ต้องพิมพ์ `--verbose true` |
| `Vec<T>` | argument ที่รับได้**หลายค่า** (repeatable) | `tags: Vec<String>` → `--tags a --tags b` ได้ `vec!["a", "b"]` |

สังเกตว่าทั้งหมดนี้**ไม่ต้องเขียน logic การ parse เองแม้แต่บรรทัดเดียว** — เราแค่ประกาศ type ของ field ให้ตรงกับ "ความหมาย" ที่ต้องการ แล้ว `clap` เลือก behavior ที่เหมาะสมให้เองจาก type นั้น ซึ่งเป็นการใช้ type system ของ Rust ในแบบที่ Part 18-22 (Generics/Traits) ปูพื้นไว้: **type ไม่ใช่แค่ป้ายกำกับ แต่เป็นข้อมูลที่ compiler และ macro นำไปใช้ตัดสินใจ behavior ได้จริง**

มาดู**ตัวอย่างเต็ม**: เครื่องมือค้นหาข้อความในไฟล์แบบเดียวกับหัวข้อ 59.1 แต่คราวนี้เขียนด้วย `clap`:

```rust
use clap::Parser;
use std::fs;
use std::process::ExitCode;

/// minigrep: mini clone ของ grep เพื่อสอนพื้นฐาน derive(Parser)
#[derive(Parser, Debug)]
#[command(name = "minigrep", version, author, about = "ค้นหาข้อความในไฟล์ (ตัวอย่างประกอบบทเรียน clap)")]
struct Cli {
    /// คำหรือ pattern ที่ต้องการค้นหา
    pattern: String,

    /// พาธของไฟล์ที่จะค้นหาข้อความ
    #[arg(value_name = "FILE")]
    path: String,

    /// แสดงเลขบรรทัดหน้าแต่ละผลลัพธ์ที่เจอ
    #[arg(short = 'n', long = "lines")]
    show_line_numbers: bool,

    /// ค้นหาแบบไม่สนตัวพิมพ์เล็ก/ใหญ่ (case-insensitive)
    #[arg(short = 'i', long = "ignore-case")]
    ignore_case: bool,
}

fn main() -> ExitCode {
    let cli = Cli::parse();

    let content = match fs::read_to_string(&cli.path) {
        Ok(c) => c,
        Err(e) => {
            eprintln!("error: ไม่สามารถเปิดไฟล์ '{}': {e}", cli.path);
            return ExitCode::from(1);
        }
    };

    let pattern = if cli.ignore_case { cli.pattern.to_lowercase() } else { cli.pattern.clone() };
    let mut found_any = false;

    for (idx, line) in content.lines().enumerate() {
        let haystack = if cli.ignore_case { line.to_lowercase() } else { line.to_string() };
        if haystack.contains(&pattern) {
            found_any = true;
            if cli.show_line_numbers {
                println!("{}: {}", idx + 1, line);
            } else {
                println!("{line}");
            }
        }
    }

    if found_any { ExitCode::SUCCESS } else { ExitCode::from(1) }
}
```

สังเกตจุดสำคัญของโค้ดนี้ทีละส่วน:

- **`pattern: String` และ `path: String`** — ไม่มี attribute `#[arg(...)]` ติดอยู่เลย เพราะทั้งสองตัวเป็น field ธรรมดาที่มา**เรียงตามลำดับที่ประกาศในโค้ด** `clap` จะตีความว่าเป็น **positional argument** สองตัวเรียงกัน (`pattern` มาก่อน `path` เพราะประกาศก่อน) — นี่คือจุดที่ต้องระมัดระวัง (จะเห็นในหัวข้อกับดักท้ายบท): **ลำดับ field ในการประกาศ struct คือลำดับ positional argument บน command line จริง ๆ**
- **`Cli::parse()`** — เมธอดนี้**คือสิ่งที่ derive macro generate ให้** (เชื่อมกับ Part 44 เรื่อง `impl Trait for TypeName` ที่ derive macro เพิ่มเข้ามา) มันอ่าน `std::env::args()` ให้เอง (ไม่ต้องเรียก `env::args()` เองอีก), parse ตามกฎที่นิยามไว้ใน struct, และ**ถ้า parse ไม่สำเร็จ (ไม่ว่าจะขาด argument ที่จำเป็น หรือใส่ argument ที่ parse ไม่ผ่าน) มันจะพิมพ์ error message ที่จัดรูปแบบสวยงามลง stderr แล้วเรียก `std::process::exit(2)` ให้เองทันที** — โค้ดของคุณ**ไม่มีโอกาสรันต่อ**ถ้า argument ผิด นี่คือเหตุผลที่ `main()` ไม่ต้องมี logic จัดการ "parse error" เองเลยแม้แต่นิดเดียว

มา build และรันจริงเพื่อดูผลลัพธ์:

```
$ minigrep world sample.txt -n -i
1: hello world
3: another line with WORLD in caps
```

และถ้าใช้ short flag แบบรวมกัน (`-ni` แทน `-n -i` แยกกัน) — สิ่งที่ต้องเขียน `if` แยกดักเองในหัวข้อ 59.1 — `clap` รองรับให้**อัตโนมัติทันที** โดยไม่ต้องเขียนโค้ดเพิ่มเลย:

```
$ minigrep world sample.txt -ni
1: hello world
3: another line with WORLD in caps

$ minigrep world sample.txt -in
1: hello world
3: another line with WORLD in caps
```

สังเกตว่า `-ni` และ `-in` ให้ผลลัพธ์เดียวกัน (ลำดับตัวอักษรใน short flag ที่รวมกันไม่มีผลต่อความหมาย) — นี่คือ behavior มาตรฐานของ Unix-style CLI ที่ `clap` implement ให้ครบถ้วนโดยที่เราไม่ต้องคิดเรื่องนี้เลย

### 59.4 Attribute ของ `clap`: ควบคุม Parsing และหน้าตาของ `--help`

ตัวอย่างข้างบนใช้ attribute ไปแล้วหลายตัวโดยไม่ได้อธิบายละเอียด มาดูทีละตัวว่าทำหน้าที่อะไรบ้าง:

**`#[command(...)]` บน struct ระดับบนสุด** — ควบคุม**metadata ของโปรแกรมทั้งตัว** (ไม่ใช่ของ argument ตัวใดตัวหนึ่ง):

- `name = "minigrep"` — ชื่อโปรแกรมที่แสดงใน `Usage:` และ error message (ถ้าไม่ระบุ `clap` จะใช้ชื่อ crate จาก `Cargo.toml` โดยอัตโนมัติผ่าน `CARGO_PKG_NAME`)
- `version` (ไม่มีค่า แปลว่า "ใช้ค่า default") — เปิดให้มี flag `-V`/`--version` โดยอ่านเลขเวอร์ชันจาก `CARGO_PKG_VERSION` ใน `Cargo.toml` ตอน compile time ผ่าน `env!()` macro (แบบเดียวกับที่ Part 35 สอนเรื่อง Cargo metadata) — ถ้าไม่ระบุ attribute นี้เลย โปรแกรมจะ**ไม่มี** flag `--version` ให้ใช้
- `author` — ใส่ authors จาก `CARGO_PKG_AUTHORS` (ฟิลด์ `authors` ใน `Cargo.toml`) เข้าไปเป็น metadata ของโปรแกรม (ในตัวอย่างของเราไม่ได้ตั้งฟิลด์นี้ไว้ใน `Cargo.toml` จึงไม่มีผลปรากฏใน `--help` เพราะไม่มีค่าให้แสดง — สังเกตว่า `author` มีผลต่อ metadata ภายในเท่านั้น ไม่ได้ทำให้ขึ้นบรรทัด "Author:" ใน `--help` แบบมาตรฐานเสมอไป ต้องอาศัย help template เพิ่มเติมถ้าต้องการโชว์ชัด ๆ)
- `about = "..."` — คำอธิบายสั้น ๆ ของโปรแกรม แสดงเป็นบรรทัดแรกของ `--help` — ถ้าไม่ระบุ `clap` จะพยายามอ่านจาก **doc comment ของ struct** (`///` เหนือ `struct Cli { ... }`) แทนโดยอัตโนมัติ

**`#[arg(...)]` บน field แต่ละตัว** — ควบคุม**พฤติกรรมของ argument ตัวนั้นโดยเฉพาะ**:

- `short` / `short = 'x'` — เปิด short flag (`-x`) ให้ field นี้ ถ้าไม่ระบุตัวอักษร `clap` จะใช้**ตัวอักษรแรกของชื่อ field** โดยอัตโนมัติ (เช่น `verbose: bool` กับ `#[arg(short)]` จะได้ `-v`)
- `long` / `long = "xxx"` — เปิด long flag (`--xxx`) ให้ field นี้ ถ้าไม่ระบุชื่อ `clap` จะแปลง**ชื่อ field จาก `snake_case` เป็น `kebab-case`** โดยอัตโนมัติ (เช่น `ignore_case: bool` กับ `#[arg(long)]` จะได้ `--ignore-case` ไม่ใช่ `--ignore_case`)
- `default_value = "..."` — ค่า default (เป็น string ที่จะถูก parse ด้วย `FromStr` ของ type field นั้นอีกที) เมื่อผู้ใช้ไม่ได้ใส่ argument ตัวนี้เลย — ทำให้ argument ตัวนี้ไม่จำเป็น (ไม่ required) โดยอัตโนมัติ แม้ type ของ field จะไม่ใช่ `Option<T>` ก็ตาม
- `value_name = "..."` — ชื่อ placeholder ที่แสดงใน `--help` (เช่น `<FILE>` แทน `<PATH>` ที่มาจากชื่อ field ตรง ๆ) — มีผลแค่กับ**การแสดงผล** ไม่มีผลต่อการ parse เลย เป็น attribute ที่มีไว้เพื่อ "คุณภาพของ documentation" ล้วน ๆ

มาดู `--help` จริงที่ `clap` generate ให้จากตัวอย่าง `minigrep` (รันจริง ตรวจสอบแล้ว):

```
$ minigrep --help
ค้นหาข้อความในไฟล์ (ตัวอย่างประกอบบทเรียน clap)

Usage: minigrep [OPTIONS] <PATTERN> <FILE>

Arguments:
  <PATTERN>  คำหรือ pattern ที่ต้องการค้นหา
  <FILE>     พาธของไฟล์ที่จะค้นหาข้อความ

Options:
  -n, --lines        แสดงเลขบรรทัดหน้าแต่ละผลลัพธ์ที่เจอ
  -i, --ignore-case  ค้นหาแบบไม่สนตัวพิมพ์เล็ก/ใหญ่ (case-insensitive)
  -h, --help         Print help
  -V, --version      Print version
```

ลองไล่ดูว่าแต่ละส่วนมาจากไหน:

- บรรทัดแรก (`ค้นหาข้อความในไฟล์...`) มาจาก `about = "..."` ใน `#[command(...)]`
- `Usage: minigrep [OPTIONS] <PATTERN> <FILE>` — `clap` **generate เอง**จากโครงสร้างของ struct ทั้งหมด: `[OPTIONS]` หมายถึงมี flag ที่ไม่จำเป็นอยู่ (`-n`, `-i`), `<PATTERN>` และ `<FILE>` คือ positional argument สองตัวเรียงตามลำดับ field ในโค้ด (สังเกตว่า `<FILE>` มาจาก `value_name = "FILE"` ที่เราตั้งไว้ ไม่ใช่ `<PATH>` ที่เป็นชื่อ field จริง)
- ส่วน `Arguments:` และ `Options:` แสดงคำอธิบายที่มาจาก **doc comment** (`///`) เหนือแต่ละ field ตรง ๆ — นี่คือเหตุผลที่ควรเขียน doc comment ให้ field ทุกตัวเสมอ เพราะมันไม่ได้มีไว้แค่เพื่อ `cargo doc` (Part 34) แต่ `clap` เอาไปใช้จริงเป็นเนื้อหาของ `--help` ที่ผู้ใช้โปรแกรมเห็นตรง ๆ
- `-h, --help` และ `-V, --version` **เพิ่มมาให้เองโดย `clap`** โดยที่เราไม่ต้องประกาศ field เพิ่มแม้แต่ตัวเดียว

และ `--version`:

```
$ minigrep --version
minigrep 0.1.0
```

**นี่คือประเด็นที่ควรเน้นย้ำ:** ทั้ง `--help` และ `--version` **ไม่มีโอกาส out-of-sync กับโค้ด parsing จริง** เพราะมันถูก generate มาจาก**แหล่งข้อมูลเดียวกัน**เสมอ (struct definition + attribute + doc comment) ต่างจากการเขียน parser เองที่ต้องคอยอัปเดต help text ด้วยมือทุกครั้งที่แก้ argument (ปัญหาที่ 2 ในหัวข้อ 59.1)

**หมายเหตุเรื่องสี:** ตัวอย่าง `--help`/error message ทั้งหมดในบทนี้แสดงเป็นข้อความล้วนไม่มีสี เพราะถูกรันผ่านการ redirect output ไปเก็บเป็นข้อความ (ซึ่งเป็นวิธีที่ใช้ตรวจสอบความถูกต้องของบทนี้ทั้งบท) แต่ในการใช้งานจริงบน terminal ที่รองรับสี `clap` (ผ่าน crate ย่อยชื่อ `anstream`/`anstyle` ที่ติดมาด้วย) จะ**ใส่สีให้ `Usage:`, ชื่อ argument, และคำว่า `error:` โดยอัตโนมัติ** เพื่อให้อ่านง่ายขึ้น โดยตรวจสอบเองว่า output ปลายทางเป็น terminal จริงหรือถูก pipe/redirect ไปที่อื่น (ถ้า pipe ไปที่อื่น จะปิดสีให้อัตโนมัติ เพื่อไม่ให้ ANSI escape code ไปปนกับข้อมูลที่โปรแกรมอื่นจะอ่านต่อ) และยังเคารพมาตรฐาน environment variable `NO_COLOR` (ถ้าตั้งไว้ จะปิดสีเสมอไม่ว่าจะเป็น terminal หรือไม่) — พฤติกรรมนี้เป็นอีกตัวอย่างของ "ส่วนที่ฟรี" ที่ `clap` จัดการให้โดยที่เราไม่ต้องคิดเรื่อง terminal detection เองเลย

**รายละเอียดเพิ่มเติมที่ควรรู้: `-h` กับ `--help` ไม่ได้แสดงผลเหมือนกันเสมอ** `clap` แยกความแตกต่างระหว่าง "short help" (`-h`) กับ "long help" (`--help`) โดยอัตโนมัติจาก doc comment: **บรรทัดแรกของ doc comment** ใช้เป็น short about (แสดงใน `-h` และในรายการ subcommand) ส่วน**ทั้งหมดของ doc comment** (รวมพารากราฟถัดไปที่เว้นบรรทัดว่างคั่น) ใช้เป็น long about (แสดงใน `--help` เท่านั้น) — เราจะเห็นตัวอย่างที่ชัดเจนของพฤติกรรมนี้ในหัวข้อ 59.9 ที่ doc comment ของ `struct Cli` มี 2 พารากราฟ

### 59.5 Subcommands ด้วย `#[derive(Subcommand)]`

เครื่องมือ command-line จำนวนมากไม่ได้มีแค่ flag/argument แบบเรียบ ๆ แต่มี**หลายคำสั่งย่อย**ภายใต้โปรแกรมเดียว เช่น `git commit`, `git push`, `git status` หรือ `cargo build`, `cargo test`, `cargo run` — แต่ละคำสั่งย่อยมี argument ของตัวเองที่ต่างกันโดยสิ้นเชิง

จาก Part 10 คุณรู้อยู่แล้วว่า **enum เหมาะกับการแทน "หนึ่งในหลายความเป็นไปได้ที่รู้ล่วงหน้าครบทุกกรณี"** — subcommand ของ CLI ก็เข้าเงื่อนไขนี้ตรงเป๊ะ: โปรแกรมหนึ่งตัวมีชุดคำสั่งย่อยที่รู้ล่วงหน้าครบถ้วนตอน compile time (ไม่ใช่สิ่งที่ผู้ใช้พิมพ์อะไรก็ได้) `clap` จึงออกแบบให้ subcommand แทนด้วย **enum ที่ derive `Subcommand`** โดยตรง — แต่ละ **variant คือ subcommand หนึ่งตัว** และ field ของแต่ละ variant คือ argument ของ subcommand นั้น (ใช้กฎ type-directed parsing แบบเดียวกับหัวข้อ 59.3 ทุกประการ)

มาดูตัวอย่างเต็ม: CLI จัดการงาน (task) แบบง่าย ที่มี 3 subcommand คือ `add`, `list`, `done`:

```rust
use clap::{Parser, Subcommand};

#[derive(Parser, Debug)]
#[command(name = "taskcli", version, about = "ตัวอย่าง subcommand ด้วย derive(Subcommand)")]
struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand, Debug)]
enum Command {
    /// เพิ่มงานใหม่เข้าไปในรายการ
    Add {
        /// ชื่อของงานที่จะเพิ่ม
        title: String,

        /// ระดับความสำคัญของงาน (low, medium, high)
        #[arg(short, long, default_value = "medium")]
        priority: String,
    },
    /// แสดงรายการงานทั้งหมด
    List {
        /// แสดงเฉพาะงานที่ยังไม่เสร็จ
        #[arg(long)]
        pending_only: bool,
    },
    /// ทำเครื่องหมายว่างานเสร็จแล้ว ตาม id
    Done {
        /// id ของงานที่ทำเสร็จแล้ว
        id: u32,
    },
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Command::Add { title, priority } => {
            println!("เพิ่มงาน: '{title}' (priority: {priority})");
        }
        Command::List { pending_only } => {
            if pending_only {
                println!("แสดงรายการงานที่ยังไม่เสร็จ");
            } else {
                println!("แสดงรายการงานทั้งหมด");
            }
        }
        Command::Done { id } => {
            println!("ทำเครื่องหมายงาน id={id} ว่าเสร็จแล้ว");
        }
    }
}
```

สังเกตจุดสำคัญ:

- **`#[command(subcommand)] command: Command`** — บอก `clap` ว่า field นี้**ไม่ใช่** argument ธรรมดา แต่เป็น**จุดที่ต้องเลือก subcommand ตัวหนึ่งจาก enum** — `clap` จะอ่านชื่อทุก variant ของ `Command` แล้วสร้างเป็นรายการ subcommand ที่ผู้ใช้ต้องเลือกหนึ่งตัว
- **ชื่อ variant `Add`/`List`/`Done` (PascalCase) กลายเป็นชื่อ subcommand `add`/`list`/`done` (kebab-case/lowercase) บน command line** — `clap` แปลงให้อัตโนมัติแบบเดียวกับที่แปลงชื่อ field เป็น `--long-flag` ในหัวข้อ 59.4
- **field ของแต่ละ variant คือ argument ของ subcommand นั้นโดยเฉพาะ** — `Add` มี `title` (positional, required) กับ `priority` (named, มี default) ส่วน `List` มีแค่ `pending_only` (flag) — แต่ละ subcommand**เป็นอิสระจากกันสมบูรณ์** ไม่มี argument ปนกัน
- **`match cli.command { ... }`** — นี่คือจุดที่เชื่อมกับ Part 10 ตรง ๆ: compiler บังคับให้ `match` ครอบคลุมทุก variant ของ `Command` (**exhaustiveness**) ถ้าเราเพิ่ม variant ใหม่เข้าไปในอนาคต (เช่น `Command::Remove { id: u32 }`) แล้วลืมเพิ่ม arm ใน `match` — **compiler จะฟ้อง error ทันทีตอน compile** ไม่ใช่ปล่อยให้เป็น bug เงียบ ๆ ที่โผล่มาตอน runtime เมื่อผู้ใช้เรียก `taskcli remove 5` แล้วไม่มีอะไรเกิดขึ้น — นี่คือข้อดีที่การออกแบบ subcommand ด้วย enum ให้มา "ฟรี" โดยที่การออกแบบด้วยวิธีอื่น (เช่น string matching เอง) ไม่มีการันตีนี้เลย

`--help` ระดับบนสุดแสดงรายการ subcommand ทั้งหมดพร้อม short about ของแต่ละตัว (มาจาก**บรรทัดแรก**ของ doc comment บน variant — ตามกฎ short/long about ที่อธิบายไว้ท้ายหัวข้อ 59.4):

```
$ taskcli --help
ตัวอย่าง subcommand ด้วย derive(Subcommand)

Usage: taskcli <COMMAND>

Commands:
  add   เพิ่มงานใหม่เข้าไปในรายการ
  list  แสดงรายการงานทั้งหมด
  done  ทำเครื่องหมายว่างานเสร็จแล้ว ตาม id
  help  Print this message or the help of the given subcommand(s)

Options:
  -h, --help     Print help
  -V, --version  Print version
```

สังเกตว่า `clap` เพิ่ม subcommand `help` ให้เองโดยอัตโนมัติด้วย (`taskcli help add` เทียบเท่ากับ `taskcli add --help`) — และแต่ละ subcommand มี `--help` ของตัวเองที่ลึกลงไปอีกชั้น:

```
$ taskcli add --help
เพิ่มงานใหม่เข้าไปในรายการ

Usage: taskcli add [OPTIONS] <TITLE>

Arguments:
  <TITLE>  ชื่อของงานที่จะเพิ่ม

Options:
  -p, --priority <PRIORITY>  ระดับความสำคัญของงาน (low, medium, high) [default: medium]
  -h, --help                 Print help
```

รันจริงแต่ละ subcommand:

```
$ taskcli add "ซื้อของ" --priority high
เพิ่มงาน: 'ซื้อของ' (priority: high)

$ taskcli list --pending-only
แสดงรายการงานที่ยังไม่เสร็จ

$ taskcli done 3
ทำเครื่องหมายงาน id=3 ว่าเสร็จแล้ว
```

และถ้าเรียก subcommand ที่ไม่มีอยู่ `clap` แจ้ง error ให้เองพร้อม suggestion ที่มีคุณภาพ:

```
$ taskcli frobnicate
error: unrecognized subcommand 'frobnicate'

Usage: taskcli <COMMAND>

For more information, try '--help'.
```

(ตัวอย่างนี้ยังไม่ persist ข้อมูลจริงลงไฟล์ — เป็นแค่การสาธิตรูปร่างของ subcommand parsing เท่านั้น เราจะเห็นเวอร์ชันที่บันทึกข้อมูลจริงในหัวข้อ 59.9)

### 59.5.1 `#[derive(Args)]` และ `#[command(flatten)]`: แชร์กลุ่ม Argument ระหว่าง Subcommand

หัวข้อ 59.9 ใช้ `global = true` เพื่อแชร์ argument ตัวเดียว (`--file`) ให้ใช้ได้ทั้งระดับบนสุดและทุก subcommand แต่ถ้าคุณมี**กลุ่มของ argument หลายตัวที่ไปด้วยกันเสมอ** (เช่น `--verbose` และ `--output` ที่หลาย subcommand ต้องมีเหมือนกันทั้งคู่) การประกาศซ้ำในทุก variant ของ enum จะทำให้โค้ดซ้ำซ้อนและเสี่ยงพลาด (เช่นลืมเพิ่มให้ variant ใหม่) `clap` มี derive macro อีกตัวชื่อ **`Args`** ที่ทำให้คุณห่อกลุ่ม argument ไว้ใน struct แยก แล้ว "flatten" (แผ่ field ของมันเข้าไปรวมกับ argument อื่นในระดับเดียวกัน) ด้วย `#[command(flatten)]`:

```rust
use clap::{Args, Parser, Subcommand};

/// argument กลุ่มที่ใช้ร่วมกันในหลาย subcommand โดยไม่ต้องประกาศซ้ำทุกที่
#[derive(Args, Debug)]
struct CommonArgs {
    /// แสดงรายละเอียดเพิ่มเติมระหว่างทำงาน
    #[arg(short, long)]
    verbose: bool,
}

#[derive(Parser, Debug)]
#[command(name = "flatten_demo")]
struct Cli {
    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand, Debug)]
enum Command {
    /// ประมวลผลไฟล์
    Process {
        file: String,

        #[command(flatten)]
        common: CommonArgs,
    },
    /// ตรวจสอบไฟล์
    Check {
        file: String,

        #[command(flatten)]
        common: CommonArgs,
    },
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        Command::Process { file, common } => {
            println!("process {file} (verbose={})", common.verbose);
        }
        Command::Check { file, common } => {
            println!("check {file} (verbose={})", common.verbose);
        }
    }
}
```

สังเกตว่า `CommonArgs` derive `Args` (ไม่ใช่ `Parser`) — **`Args` ใช้กับ struct ที่เป็น "ส่วนประกอบของ" argument parser ตัวใหญ่กว่า ไม่ใช่ตัว parser หลักที่เรียก `::parse()` ตรง ๆ** (สังเกตว่า `CommonArgs` ไม่มีเมธอด `parse()` ให้เรียก และไม่มี `#[command(name = ..., version, ...)]` เพราะไม่ใช่ "โปรแกรม" ในตัวมันเอง แค่เป็นชิ้นส่วนหนึ่งของ argument ของโปรแกรม) ส่วน `#[command(flatten)]` บน field `common: CommonArgs` บอก `clap` ว่า**อย่าตีความ `CommonArgs` เป็น argument ตัวเดียว** (ซึ่งจะผิด เพราะ `CommonArgs` ไม่ใช่ type ที่ parse จาก string ตัวเดียวได้) แต่ให้**แผ่ทุก field ข้างในออกมาปนกับ field อื่นของ `Process`/`Check` ตรง ๆ** เสมือนเขียน `verbose: bool` ไว้ในทั้งสอง variant นั้นเอง

ทดสอบจริง:

```
$ flatten_demo process report.txt --verbose
process report.txt (verbose=true)

$ flatten_demo check report.txt
check report.txt (verbose=false)
```

และ `--help` ของ `process` แสดง `-v, --verbose` ปนไปกับ argument อื่นตามปกติ ราวกับไม่มี `CommonArgs` แยกอยู่เบื้องหลังเลย:

```
$ flatten_demo process --help
ประมวลผลไฟล์

Usage: flatten_demo process [OPTIONS] <FILE>

Arguments:
  <FILE>

Options:
  -v, --verbose  แสดงรายละเอียดเพิ่มเติมระหว่างทำงาน
  -h, --help     Print help
```

**ทำไมสิ่งนี้ถึงมีประโยชน์ในทางปฏิบัติ:** สมมติคุณมี 8 subcommand ที่ทุกตัวต้องมี `--verbose`, `--quiet`, `--output` เหมือนกันหมด — ถ้าไม่ใช้ `flatten` คุณต้องประกาศ field ทั้ง 3 ตัวซ้ำใน 8 variant (24 บรรทัดของ attribute ที่ต้องคอยให้ตรงกัน) และถ้าวันหนึ่งต้องเพิ่ม argument ตัวที่ 4 เข้าไปในกลุ่มนี้ ต้องแก้ทั้ง 8 ที่พร้อมกัน — เสี่ยงพลาดแบบเดียวกับปัญหา "โค้ดซ้ำ" ทั่วไปที่ Part 52-53 (Design Patterns) เตือนไว้เรื่อง DRY (Don't Repeat Yourself) เมื่อใช้ `Args` + `flatten` คุณแก้ที่ `CommonArgs` ที่เดียว ทุก subcommand ที่ flatten มันเข้าไปได้ผลลัพธ์ตรงกันโดยอัตโนมัติเสมอ — เป็นอีกตัวอย่างที่แสดงว่าการออกแบบ CLI ด้วย `clap` แปลงปัญหา "เขียนโค้ด parsing ซ้ำ ๆ" ให้กลายเป็นปัญหา "ออกแบบโครงสร้างข้อมูล (struct/enum) ให้ดี" ซึ่งเป็นปัญหาที่ Rust ถนัดจัดการอยู่แล้ว

### 59.6 Validation ด้วย `value_parser`: ตรวจสอบมากกว่าแค่ type

Type-directed parsing ในหัวข้อ 59.3 ครอบคลุมแค่ "แปลง string เป็น type ที่ต้องการได้ไหม" (เช่น `"25"` แปลงเป็น `u32` ได้) แต่บ่อยครั้งเราต้องการ validate **เงื่อนไขที่ลึกกว่านั้น** เช่น ตัวเลขต้องอยู่ในช่วงที่กำหนด หรือ string ต้องอยู่ในรูปแบบเฉพาะ (เช่นวันที่) — `clap` เปิดช่องให้ทำแบบนี้ผ่าน `#[arg(value_parser = ...)]` ซึ่งรับ**ฟังก์ชันใด ๆ ที่มี signature `Fn(&str) -> Result<T, E>` โดยที่ `E: Display`**

```rust
use clap::Parser;

/// ตรวจสอบพอร์ตว่าอยู่ในช่วง 1..=65535 และ parse วันที่แบบ YYYY-MM-DD เอง
fn parse_port(s: &str) -> Result<u16, String> {
    let port: u32 = s.parse().map_err(|_| format!("'{s}' ไม่ใช่ตัวเลข"))?;
    if port == 0 || port > 65535 {
        return Err(format!("พอร์ตต้องอยู่ในช่วง 1-65535 แต่ได้ {port}"));
    }
    Ok(port as u16)
}

#[derive(Debug, Clone)]
struct SimpleDate {
    year: u32,
    month: u32,
    day: u32,
}

fn parse_date(s: &str) -> Result<SimpleDate, String> {
    let parts: Vec<&str> = s.split('-').collect();
    if parts.len() != 3 {
        return Err(format!("รูปแบบวันที่ต้องเป็น YYYY-MM-DD แต่ได้ '{s}'"));
    }
    let year: u32 = parts[0].parse().map_err(|_| format!("ปี '{}' ไม่ใช่ตัวเลข", parts[0]))?;
    let month: u32 = parts[1].parse().map_err(|_| format!("เดือน '{}' ไม่ใช่ตัวเลข", parts[1]))?;
    let day: u32 = parts[2].parse().map_err(|_| format!("วัน '{}' ไม่ใช่ตัวเลข", parts[2]))?;
    if !(1..=12).contains(&month) {
        return Err(format!("เดือนต้องอยู่ในช่วง 1-12 แต่ได้ {month}"));
    }
    if !(1..=31).contains(&day) {
        return Err(format!("วันต้องอยู่ในช่วง 1-31 แต่ได้ {day}"));
    }
    Ok(SimpleDate { year, month, day })
}

#[derive(Parser, Debug)]
#[command(name = "validate_demo", about = "สาธิต value_parser สำหรับ validation ที่ซับซ้อนกว่า type ธรรมดา")]
struct Cli {
    /// หมายเลขพอร์ตที่จะเปิดฟัง (1-65535)
    #[arg(short, long, value_parser = parse_port)]
    port: u16,

    /// วันที่เริ่มงาน รูปแบบ YYYY-MM-DD
    #[arg(long, value_parser = parse_date)]
    start_date: SimpleDate,
}

fn main() {
    let cli = Cli::parse();
    println!("port = {}", cli.port);
    println!("start_date = {}-{:02}-{:02}", cli.start_date.year, cli.start_date.month, cli.start_date.day);
}
```

สังเกตว่า `parse_port` คืน `Result<u16, String>` ธรรมดา — ไม่ต้องมี trait พิเศษอะไรเพิ่มเติมเลย (เชื่อมกับ Part 12 เรื่อง `Result<T, E>` ตรง ๆ) `clap` เรียกฟังก์ชันนี้ให้เองทุกครั้งที่ต้อง parse argument `--port`, ถ้าได้ `Ok(port)` มันเก็บค่าไว้ในตัวแปรตามปกติ, ถ้าได้ `Err(msg)` มันเอา `msg` ไปประกอบเป็น error message ที่ format สวยงามให้ทันที — และที่สำคัญคือ `parse_date` แสดงให้เห็นว่า `value_parser` ไม่ได้จำกัดแค่ "validate ตัวเลข" แต่ใช้สร้าง **type ที่ซับซ้อนกว่า string/number ธรรมดา** (`SimpleDate`) จาก input ดิบได้เลยในขั้นตอนเดียว โดยที่ field `start_date` ใน struct มี type เป็น `SimpleDate` ตรง ๆ (ไม่ใช่ `String` ที่ต้อง parse เพิ่มทีหลังใน `main()`)

ทดสอบกรณีถูกต้อง:

```
$ validate_demo --port 8080 --start-date 2026-01-15
port = 8080
start_date = 2026-01-15
```

ทดสอบกรณีพอร์ตเกินช่วง — สังเกต error message ที่ `clap` ประกอบให้ (ตรวจสอบจริง):

```
$ validate_demo --port 999999 --start-date 2026-01-15
error: invalid value '999999' for '--port <PORT>': พอร์ตต้องอยู่ในช่วง 1-65535 แต่ได้ 999999

For more information, try '--help'.
```

ทดสอบกรณีรูปแบบวันที่ผิด:

```
$ validate_demo --port 8080 --start-date 2026/01/15
error: invalid value '2026/01/15' for '--start-date <START_DATE>': รูปแบบวันที่ต้องเป็น YYYY-MM-DD แต่ได้ '2026/01/15'

For more information, try '--help'.
```

และกรณีเดือนเกินช่วง (validation ที่ลึกกว่าแค่ "แปลงเป็นตัวเลขได้ไหม"):

```
$ validate_demo --port 8080 --start-date 2026-13-15
error: invalid value '2026-13-15' for '--start-date <START_DATE>': เดือนต้องอยู่ในช่วง 1-12 แต่ได้ 13

For more information, try '--help'.
```

สังเกตรูปแบบที่ `clap` ใช้เสมอ: `error: invalid value '<ค่าที่ผู้ใช้ใส่>' for '<ชื่อ argument>': <error message ที่ฟังก์ชัน value_parser ของเราคืนมา>` — นี่คือสิ่งที่ทำให้ **error message ของ `clap` สม่ำเสมอกันทั้งโปรแกรม** แม้ logic การ validate จะต่างกันโดยสิ้นเชิงในแต่ละ argument (เทียบกับปัญหาที่ 3 ในหัวข้อ 59.1 ที่การเขียน error message เองมักไม่สม่ำเสมอ) ผู้ใช้เห็นทันทีว่า**ค่าไหน**ที่ผิดและ**ทำไม**ผิด โดยไม่ต้องเดาว่า error นี้มาจาก argument ตัวไหน — คุณภาพระดับนี้ยากที่จะทำได้ด้วยมือแบบสม่ำเสมอทุกจุดในโปรแกรมขนาดใหญ่

### 59.6.1 `conflicts_with`: บอกว่า Argument สองตัวใช้พร้อมกันไม่ได้

Validation ในหัวข้อ 59.6 ตรวจสอบ**ค่าของ argument ตัวเดียว** แต่บางครั้งปัญหาคือ**ความสัมพันธ์ระหว่าง argument สองตัว** เช่น `--verbose` (แสดงรายละเอียดมากขึ้น) กับ `--quiet` (ไม่แสดงอะไรเลย) เป็นสองสิ่งที่ขัดแย้งกันเชิงความหมาย ไม่ควรใส่มาพร้อมกัน — `clap` มี attribute `conflicts_with` สำหรับประกาศความสัมพันธ์แบบนี้โดยตรง ไม่ต้องเขียน `if cli.verbose && cli.quiet { ... }` เช็คเองใน `main()`:

```rust
use clap::Parser;

/// สาธิต conflicts_with: --quiet และ --verbose ใช้พร้อมกันไม่ได้
#[derive(Parser, Debug)]
#[command(name = "conflict_demo")]
struct Cli {
    /// แสดงรายละเอียดมากขึ้น
    #[arg(short, long, conflicts_with = "quiet")]
    verbose: bool,

    /// ไม่แสดงข้อความใด ๆ เลย
    #[arg(short, long)]
    quiet: bool,
}

fn main() {
    let cli = Cli::parse();
    println!("{cli:?}");
}
```

สังเกตว่า `conflicts_with = "quiet"` อ้างอิงถึง field อีกตัวด้วย**ชื่อ field เป็น string** (`"quiet"` ต้องตรงกับชื่อ field `quiet` เป๊ะ) — นี่เป็นจุดเดียวใน Derive API ที่ยังพึ่ง string matching อยู่ (คล้ายกับ Builder API ในหัวข้อ 59.2) เพราะ `clap` ต้องอ้างอิง argument อื่นที่**ยังไม่รู้จักตอน macro expand ของ field ปัจจุบัน** แต่ต่างจาก Builder API ตรงที่ `clap_derive` **ตรวจสอบชื่อนี้ให้ตอน build `Command`** (ผ่าน `debug_assert!` แบบเดียวกับกับดักข้อ 3) ถ้าพิมพ์ผิดจะได้ panic ชัดเจนตอน debug build ทันที ไม่ใช่ silent bug

ทดสอบกรณีใส่ปกติ (ใส่แค่ตัวเดียว ไม่มีปัญหา):

```
$ conflict_demo --verbose
Cli { verbose: true, quiet: false }
```

ทดสอบกรณีใส่ทั้งสองตัวพร้อมกัน — `clap` ปฏิเสธให้เองก่อนโค้ดของเราจะได้รันด้วยซ้ำ:

```
$ conflict_demo --verbose --quiet
error: the argument '--verbose' cannot be used with '--quiet'

Usage: conflict_demo --verbose

For more information, try '--help'.
```

เทียบกับการเช็คเองใน `main()` (`if cli.verbose && cli.quiet { eprintln!(...); return ExitCode::from(2); }`) ซึ่งก็ทำงานได้เหมือนกัน แต่ `conflicts_with` มีข้อดีสามอย่าง: (1) error message ที่ได้ตรงรูปแบบเดียวกับ error อื่น ๆ ของ `clap` โดยอัตโนมัติ ไม่ต้องเขียนเอง (2) ปรากฏใน `--help` แบบ metadata ภายใน ทำให้ tool อื่นที่ generate documentation จาก `clap::Command` (เช่น generate man page) รู้จักความสัมพันธ์นี้ได้ด้วย ไม่ใช่แค่ logic ที่ซ่อนอยู่ใน `main()` และ (3) ความสัมพันธ์แบบนี้ถูกตรวจสอบ**ก่อน**โค้ด `main()` ของเราจะได้รันด้วยซ้ำ ทำให้ไม่มีทางลืมเช็คในบางจุดของโปรแกรมที่ซับซ้อนขึ้น

### 59.7 Environment Variable Fallback ด้วย `#[arg(env = "...")]`

แนวคิด **12-factor app** (มาตรฐานการออกแบบแอปพลิเคชันที่ deploy บน cloud อย่างแพร่หลาย) แนะนำให้แยก**ค่า config ที่เปลี่ยนตามสภาพแวดล้อม** (เช่น API key, connection string, feature flag) ออกจากตัวโค้ด และตั้งค่าผ่าน **environment variable** แทนการ hardcode หรือใส่ผ่าน argument ทุกครั้ง — แต่ในทางปฏิบัติ นักพัฒนามักต้องการทั้งสองทาง: **ตั้งผ่าน environment variable เป็นค่า default สำหรับใช้งานประจำ แต่ override ผ่าน CLI flag ได้เวลาทดสอบหรือกรณีพิเศษ**

`clap` รองรับ pattern นี้ตรง ๆ ผ่าน `#[arg(env = "...")]`:

```rust
use clap::Parser;

/// สาธิต #[arg(env = "...")]: CLI flag มาก่อน, ถ้าไม่ใส่จะ fallback ไปอ่าน environment variable
#[derive(Parser, Debug)]
#[command(name = "env_demo")]
struct Cli {
    /// API key สำหรับเชื่อมต่อบริการ (ใส่ตรง ๆ หรือกำหนดผ่าน env var APP_API_KEY ก็ได้)
    #[arg(long, env = "APP_API_KEY")]
    api_key: String,

    /// ระดับ verbosity อ่านจาก env var APP_VERBOSE ได้ ถ้าไม่ใส่จะใช้ default
    #[arg(long, env = "APP_VERBOSE", default_value_t = 0)]
    verbose: u8,
}

fn main() {
    let cli = Cli::parse();
    println!("api_key = {}", cli.api_key);
    println!("verbose = {}", cli.verbose);
}
```

หมายเหตุ: `default_value_t = 0` ต่างจาก `default_value = "0"` ที่เคยเห็นในหัวข้อก่อน — `default_value_t` รับ**ค่า Rust literal ตรง ๆ** ที่ต้อง implement `Clone` (ในที่นี้คือ `0u8`) แทนการรับเป็น string แล้วให้ `clap` parse อีกที ทั้งสองแบบให้ผลลัพธ์เหมือนกันในกรณีนี้ แต่ `default_value_t` ตรวจสอบความถูกต้องของ type ได้ตั้งแต่ compile time (ถ้าใส่ค่าที่ type ไม่ตรงจะเจอ compile error ทันที ไม่ใช่ runtime panic ตอน parse default value)

**ลำดับความสำคัญ (priority) ที่ `clap` ใช้คือ: ค่าจาก CLI flag > ค่าจาก environment variable > `default_value`/`default_value_t`** — ลองดู `--help` ที่แสดง priority นี้ให้ผู้ใช้เห็นชัดเจน:

```
$ env_demo --help
สาธิต #[arg(env = "...")]: CLI flag มาก่อน, ถ้าไม่ใส่จะ fallback ไปอ่าน environment variable

Usage: env_demo [OPTIONS] --api-key <API_KEY>

Options:
      --api-key <API_KEY>  API key สำหรับเชื่อมต่อบริการ (ใส่ตรง ๆ หรือกำหนดผ่าน env var APP_API_KEY ก็ได้) [env: APP_API_KEY=]
      --verbose <VERBOSE>  ระดับ verbosity อ่านจาก env var APP_VERBOSE ได้ ถ้าไม่ใส่จะใช้ default [env: APP_VERBOSE=] [default: 0]
  -h, --help               Print help
```

สังเกต `[env: APP_API_KEY=]` — `clap` แสดงชื่อ environment variable ที่เกี่ยวข้องพร้อมค่าปัจจุบันของมัน (ในที่นี้ไม่มีค่าเพราะเรารัน `--help` โดยไม่ได้ตั้ง environment variable ไว้ก่อน) นี่คือรายละเอียดเล็ก ๆ ที่มีประโยชน์มาก: ผู้ใช้ที่งงว่า "ทำไม flag นี้ไม่ required แต่ error บอกว่า required" จะเห็นทันทีจาก `--help` ว่ามันรับค่าจาก environment variable ได้ด้วย

ทดสอบกรณีไม่ใส่ flag และไม่ตั้ง environment variable — ยัง error เหมือนเป็น required argument ตามปกติ (เพราะไม่มีค่าจากทางไหนเลย):

```
$ env_demo
error: the following required arguments were not provided:
  --api-key <API_KEY>

Usage: env_demo --api-key <API_KEY>

For more information, try '--help'.
```

ทดสอบกรณีตั้ง environment variable ไว้ ไม่ต้องใส่ flag เลย:

```
$ APP_API_KEY=secret123 APP_VERBOSE=2 env_demo
api_key = secret123
verbose = 2
```

และทดสอบว่า **CLI flag override environment variable จริง** (ตั้ง environment variable ไว้ แต่ยังใส่ flag ทับ):

```
$ APP_API_KEY=secret123 env_demo --api-key fromcli
api_key = fromcli
verbose = 0
```

ผลลัพธ์ยืนยันลำดับความสำคัญที่กล่าวไว้ตอนต้น: `--api-key fromcli` ที่ใส่ตรง ๆ ชนะค่าจาก `APP_API_KEY=secret123` เสมอ

### 59.8 ผสาน `clap` + `serde` + Error Handling: โหลด Config File อย่างถูกต้อง

เครื่องมือ command-line ระดับใช้งานจริงมักต้องอ่าน**ไฟล์ config** เพิ่มเติมจาก argument ธรรมดา (เพราะการตั้งค่าที่ซับซ้อนหรือมีจำนวนมาก ใส่ผ่าน CLI flag ล้วน ๆ ไม่สะดวก) รูปแบบที่พบบ่อยที่สุดคือ: **`clap` parse argument `--config <path>` แล้วโปรแกรมอ่านไฟล์ที่พาธนั้นด้วย `std::fs`, deserialize ด้วย `serde` (Part 57), และต้องจัดการทุกจุดที่อาจล้มเหลวอย่างเหมาะสม** พร้อม **exit code ที่บอกสาเหตุของความล้มเหลวต่างกัน** (เชื่อมกับ Part 12 เรื่อง exit code และ Part 30-31 เรื่อง error handling ระดับมืออาชีพ)

จุดที่อาจล้มเหลวมีอย่างน้อย 3 จุดที่**ต่างกันโดยธรรมชาติ** และควรได้ exit code ต่างกัน:

1. **CLI argument ผิด** (เช่นลืมใส่ `--config`) — `clap` จัดการให้เองแล้ว (exit code 2 ตามที่เห็นมาตลอดบทนี้)
2. **ไฟล์ config หาไม่พบ** — เป็น I/O error จาก `std::fs`
3. **ไฟล์ config มีอยู่แต่รูปแบบไม่ถูกต้อง** (JSON ผิด syntax หรือ field ผิด type) — เป็น deserialize error จาก `serde_json`

```rust
use clap::Parser;
use serde::Deserialize;
use std::fs;
use std::process::ExitCode;

// exit code ตามธรรมเนียมของ sysexits.h ที่ Part 12 เคยพูดถึง
const EXIT_CONFIG_NOT_FOUND: u8 = 66; // EX_NOINPUT
const EXIT_CONFIG_INVALID: u8 = 78; // EX_CONFIG

/// โครงสร้าง config ที่อ่านจากไฟล์ JSON ด้วย serde (จาก Part 57)
#[derive(Debug, Deserialize)]
struct AppConfig {
    host: String,
    port: u16,
    #[serde(default)]
    debug: bool,
}

/// สาธิตการผสาน clap (parse CLI) + serde (parse config file) + exit code ที่เหมาะสมกับแต่ละความล้มเหลว
#[derive(Parser, Debug)]
#[command(name = "config_demo")]
struct Cli {
    /// พาธของไฟล์ config รูปแบบ JSON
    #[arg(long, value_name = "PATH")]
    config: String,
}

fn main() -> ExitCode {
    let cli = Cli::parse();

    let raw = match fs::read_to_string(&cli.config) {
        Ok(text) => text,
        Err(e) => {
            eprintln!("error: ไม่พบไฟล์ config ที่ '{}': {e}", cli.config);
            return ExitCode::from(EXIT_CONFIG_NOT_FOUND);
        }
    };

    let config: AppConfig = match serde_json::from_str(&raw) {
        Ok(c) => c,
        Err(e) => {
            eprintln!("error: ไฟล์ config '{}' มีรูปแบบไม่ถูกต้อง: {e}", cli.config);
            return ExitCode::from(EXIT_CONFIG_INVALID);
        }
    };

    println!("โหลด config สำเร็จ: host={}, port={}, debug={}", config.host, config.port, config.debug);
    ExitCode::SUCCESS
}
```

สังเกตว่าเราเลือกใช้ **`std::process::ExitCode`** เป็น return type ของ `main()` แทนการเรียก `std::process::exit()` ตรง ๆ — นี่คือแนวทางที่แนะนำใน Rust สมัยใหม่ (Part 12 ปูพื้นเรื่องนี้ไว้): `ExitCode` ให้ compiler รู้ตั้งแต่ signature ของ `main()` เลยว่าฟังก์ชันนี้จบด้วยการกำหนด exit code แบบใดแบบหนึ่งเสมอ (ไม่ใช่ side-effect ที่ซ่อนอยู่กลางฟังก์ชันแบบ `process::exit()` ที่ตัด control flow ทันทีโดยไม่ผ่าน destructor ของตัวแปรที่เหลืออยู่ใน scope) — สำหรับโปรแกรมเล็ก ๆ แบบนี้ความต่างไม่มีผลกระทบมาก แต่เป็นนิสัยที่ดีที่ควรฝึกไว้

ทดสอบทั้ง 4 กรณี (สร้างไฟล์ config จริงและรันจริงตรวจสอบแล้ว):

**กรณีสำเร็จ:**

```
$ config_demo --config good_config.json
โหลด config สำเร็จ: host=127.0.0.1, port=8080, debug=true
```

(โดย `good_config.json` มีเนื้อหา `{ "host": "127.0.0.1", "port": 8080, "debug": true }`)

**กรณีไฟล์หาไม่พบ (exit code 66):**

```
$ config_demo --config missing.json
error: ไม่พบไฟล์ config ที่ 'missing.json': No such file or directory (os error 2)
$ echo $?
66
```

**กรณีไฟล์มีอยู่แต่รูปแบบผิด (exit code 78)** — ในตัวอย่างนี้ `port` เป็น string `"not-a-number"` ทั้งที่ `AppConfig` ประกาศว่าเป็น `u16`:

```
$ config_demo --config bad_config.json
error: ไฟล์ config 'bad_config.json' มีรูปแบบไม่ถูกต้อง: invalid type: string "not-a-number", expected u16 at line 1 column 45
$ echo $?
78
```

**กรณีลืมใส่ `--config` เลย (exit code 2 จาก `clap` เอง ไม่ต้องเขียนโค้ดจัดการ):**

```
$ config_demo
error: the following required arguments were not provided:
  --config <PATH>

Usage: config_demo --config <PATH>

For more information, try '--help'.
$ echo $?
2
```

สังเกตว่า error message ของ `serde_json` (`invalid type: string "not-a-number", expected u16 at line 1 column 45`) มา**จาก `serde` โดยตรง** ไม่ใช่สิ่งที่เราเขียนเอง — นี่คือตัวอย่างที่ดีของการที่ Part 57-58 (serde) และบทนี้ (clap) **ทำงานร่วมกันโดยที่แต่ละ crate รับผิดชอบ error message ในโดเมนของตัวเอง**: `clap` รับผิดชอบความถูกต้องของ**รูปร่าง command line**, `serde` รับผิดชอบความถูกต้องของ**รูปร่างข้อมูล** และตัวโปรแกรมของเรารับผิดชอบแค่การ**เลือก exit code ที่สื่อความหมาย**และห่อ error message ทั้งสองแหล่งให้ผู้ใช้อ่านเข้าใจว่าเกิดอะไรขึ้นที่ไหน — เป็นการแบ่งงานที่ชัดเจนตามแนวคิด separation of concerns ที่ Part 30-31 พูดถึง

### 59.9 โปรเจกต์เต็ม: Todo List CLI พร้อม Persistent Storage

มาถึงหัวข้อสุดท้ายที่รวมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน: **CLI จัดการ todo list** ที่มี subcommand `add`/`list`/`complete`/`remove`, บันทึกข้อมูลลงไฟล์ JSON ด้วย `serde` (คงอยู่ข้ามการรันโปรแกรมแต่ละครั้ง ไม่ใช่แค่ใน memory แบบหัวข้อ 59.5), มี error handling ครบทุกจุด, และ `--help` ที่สมบูรณ์

```rust
use clap::{Parser, Subcommand, ValueEnum};
use serde::{Deserialize, Serialize};
use std::fs;
use std::path::PathBuf;
use std::process::ExitCode;

const EXIT_STORAGE_ERROR: u8 = 1;
const EXIT_NOT_FOUND: u8 = 3;

/// todo: CLI จัดการรายการสิ่งที่ต้องทำ พร้อมบันทึกข้อมูลลงไฟล์ JSON
///
/// ตัวอย่างการรวมความรู้จาก Part 44-45 (derive macro), Part 57-58 (serde),
/// Part 10 (enum/match), และ Part 12/30-31 (error handling + exit code)
#[derive(Parser, Debug)]
#[command(name = "todo", version, author = "Rust Course", about = "จัดการรายการสิ่งที่ต้องทำแบบง่าย")]
struct Cli {
    /// พาธของไฟล์เก็บข้อมูล (default: todo.json ในโฟลเดอร์ปัจจุบัน)
    #[arg(long, env = "TODO_FILE", default_value = "todo.json", global = true)]
    file: PathBuf,

    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand, Debug)]
enum Command {
    /// เพิ่มงานใหม่
    Add {
        /// ข้อความอธิบายงาน
        title: String,

        /// ระดับความสำคัญของงาน
        #[arg(short, long, value_enum, default_value_t = Priority::Medium)]
        priority: Priority,
    },
    /// แสดงรายการงานทั้งหมด (หรือเฉพาะที่ยังไม่เสร็จด้วย --pending)
    List {
        /// แสดงเฉพาะงานที่ยังไม่เสร็จ
        #[arg(long)]
        pending: bool,
    },
    /// ทำเครื่องหมายว่างานเสร็จแล้ว
    Complete {
        /// id ของงานที่ต้องการทำเครื่องหมาย
        id: u32,
    },
    /// ลบงานออกจากรายการ
    Remove {
        /// id ของงานที่ต้องการลบ
        id: u32,
    },
}

#[derive(ValueEnum, Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "lowercase")]
enum Priority {
    Low,
    Medium,
    High,
}

impl std::fmt::Display for Priority {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let s = match self {
            Priority::Low => "low",
            Priority::Medium => "medium",
            Priority::High => "high",
        };
        write!(f, "{s}")
    }
}

#[derive(Debug, Serialize, Deserialize)]
struct Task {
    id: u32,
    title: String,
    priority: Priority,
    done: bool,
}

#[derive(Debug, Serialize, Deserialize)]
struct TaskList {
    next_id: u32,
    tasks: Vec<Task>,
}

impl Default for TaskList {
    fn default() -> Self {
        TaskList { next_id: 1, tasks: Vec::new() }
    }
}

fn load(path: &PathBuf) -> Result<TaskList, String> {
    match fs::read_to_string(path) {
        Ok(text) => serde_json::from_str(&text)
            .map_err(|e| format!("ไฟล์ '{}' มีรูปแบบไม่ถูกต้อง: {e}", path.display())),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(TaskList::default()),
        Err(e) => Err(format!("ไม่สามารถอ่านไฟล์ '{}': {e}", path.display())),
    }
}

fn save(path: &PathBuf, list: &TaskList) -> Result<(), String> {
    let json = serde_json::to_string_pretty(list)
        .map_err(|e| format!("ไม่สามารถแปลงข้อมูลเป็น JSON: {e}"))?;
    fs::write(path, json).map_err(|e| format!("ไม่สามารถเขียนไฟล์ '{}': {e}", path.display()))
}

fn main() -> ExitCode {
    let cli = Cli::parse();

    let mut list = match load(&cli.file) {
        Ok(l) => l,
        Err(e) => {
            eprintln!("error: {e}");
            return ExitCode::from(EXIT_STORAGE_ERROR);
        }
    };

    match cli.command {
        Command::Add { title, priority } => {
            let id = list.next_id;
            list.next_id += 1;
            list.tasks.push(Task { id, title: title.clone(), priority, done: false });
            if let Err(e) = save(&cli.file, &list) {
                eprintln!("error: {e}");
                return ExitCode::from(EXIT_STORAGE_ERROR);
            }
            println!("เพิ่มงาน #{id}: \"{title}\" (priority: {priority})");
        }
        Command::List { pending } => {
            let tasks: Vec<&Task> = list.tasks.iter().filter(|t| !pending || !t.done).collect();
            if tasks.is_empty() {
                println!("ไม่มีงานในรายการ");
            } else {
                for t in tasks {
                    let mark = if t.done { "[x]" } else { "[ ]" };
                    println!("{mark} #{} {} (priority: {})", t.id, t.title, t.priority);
                }
            }
        }
        Command::Complete { id } => {
            match list.tasks.iter_mut().find(|t| t.id == id) {
                Some(t) => {
                    t.done = true;
                    if let Err(e) = save(&cli.file, &list) {
                        eprintln!("error: {e}");
                        return ExitCode::from(EXIT_STORAGE_ERROR);
                    }
                    println!("ทำเครื่องหมายงาน #{id} ว่าเสร็จแล้ว");
                }
                None => {
                    eprintln!("error: ไม่พบงาน id {id}");
                    return ExitCode::from(EXIT_NOT_FOUND);
                }
            }
        }
        Command::Remove { id } => {
            let before = list.tasks.len();
            list.tasks.retain(|t| t.id != id);
            if list.tasks.len() == before {
                eprintln!("error: ไม่พบงาน id {id}");
                return ExitCode::from(EXIT_NOT_FOUND);
            }
            if let Err(e) = save(&cli.file, &list) {
                eprintln!("error: {e}");
                return ExitCode::from(EXIT_STORAGE_ERROR);
            }
            println!("ลบงาน #{id} แล้ว");
        }
    }

    ExitCode::SUCCESS
}
```

มาไล่ดูจุดที่รวมความรู้จากหลาย Part เข้าด้วยกัน:

- **`#[arg(..., global = true)]` บน field `file`** — `global = true` ทำให้ argument `--file` ใช้ได้ทั้งใน**ระดับบนสุด** (`todo --file x.json add ...`) และ**ใน subcommand ทุกตัว** (`todo add ... --file x.json`) โดยไม่ต้องประกาศซ้ำในทุก variant ของ `Command` — เป็นเทคนิคสำคัญที่ช่วยลดโค้ดซ้ำสำหรับ argument ที่ควรใช้ร่วมกันได้ทุกคำสั่ง (เช่น `--verbose`, `--config`, `--file` แบบนี้)
- **`#[derive(ValueEnum)]` บน `enum Priority`** — เป็น derive macro อีกตัวของ `clap` (นอกจาก `Parser`/`Subcommand`) ที่ทำให้ enum ตัวหนึ่งใช้เป็น**ชุดตัวเลือกที่จำกัด**สำหรับ argument ได้โดยตรง (`value_enum` ใน `#[arg(...)]`) — ผู้ใช้พิมพ์ `--priority high` ได้ (ตัวพิมพ์เล็กของชื่อ variant) และถ้าพิมพ์ค่าที่ไม่อยู่ใน enum จะได้ error พร้อมรายการตัวเลือกที่ถูกต้องทั้งหมดให้ทันที (ดูตัวอย่าง error ด้านล่าง) สังเกตว่า `Priority` derive ทั้ง `ValueEnum` (สำหรับ `clap`) **และ** `Serialize`/`Deserialize` (สำหรับ `serde`) **บน struct/enum เดียวกัน** — นี่คือการ compose derive macro จากสอง crate ที่ Part 44 อธิบายไว้ว่าทำได้อย่างเป็นธรรมชาติ เพราะแต่ละตัว generate `impl Trait` ที่แยกจากกันโดยสมบูรณ์
- **`TaskList` และ `Task` derive ทั้ง `Serialize` และ `Deserialize`** — ตรงกับสิ่งที่ Part 57 สอนไว้ทุกประการ: struct ที่ derive ทั้งสอง trait นี้แปลงเป็น/จาก JSON ได้โดยอัตโนมัติผ่าน `serde_json::to_string_pretty`/`serde_json::from_str`
- **`load()`/`save()` คืน `Result<T, String>`** — รวม error จากทั้ง `std::fs` (I/O) และ `serde_json` (deserialize) เข้าเป็น error type เดียวกันด้วย `.map_err(...)` (เทคนิคจาก Part 30 เรื่องการแปลง error ให้เป็น type เดียวกันเมื่อต้องรวมหลายแหล่ง) — ในโปรเจกต์ขนาดใหญ่กว่านี้ ควรพิจารณาใช้ custom error enum กับ `thiserror` (Part 31) แทน `String` ตรง ๆ เพื่อให้ผู้เรียกสามารถ pattern-match แยกแยะสาเหตุของ error ได้ แต่สำหรับ CLI ขนาดนี้ที่ error ทุกจุดจบด้วยการพิมพ์ข้อความแล้ว exit เท่านั้น `String` เพียงพอและลดความซับซ้อนของโค้ดลงได้มาก
- **exit code แยกตามสาเหตุ**: `EXIT_STORAGE_ERROR = 1` (อ่าน/เขียนไฟล์ล้มเหลว) และ `EXIT_NOT_FOUND = 3` (หา id ที่ระบุไม่เจอ) — ต่างจาก exit code `2` ที่ `clap` ใช้เองเมื่อ CLI argument ผิด (เราไม่ต้องกำหนดค่านี้เอง เพราะ `clap` เลือกให้แล้วก่อนโค้ดของเราจะได้รันด้วยซ้ำ)

มาดู `--help` ระดับบนสุดที่สมบูรณ์ (สังเกตว่า doc comment ของ `struct Cli` มี **2 พารากราฟ** คั่นด้วยบรรทัดว่าง — นี่คือจุดที่แสดงความต่างระหว่าง short help กับ long help ที่พูดถึงไว้ท้ายหัวข้อ 59.4 ให้เห็นชัดที่สุด):

```
$ todo -h
จัดการรายการสิ่งที่ต้องทำแบบง่าย

Usage: todo [OPTIONS] <COMMAND>

Commands:
  add       เพิ่มงานใหม่
  list      แสดงรายการงานทั้งหมด (หรือเฉพาะที่ยังไม่เสร็จด้วย --pending)
  complete  ทำเครื่องหมายว่างานเสร็จแล้ว
  remove    ลบงานออกจากรายการ
  help      Print this message or the help of the given subcommand(s)

Options:
      --file <FILE>  พาธของไฟล์เก็บข้อมูล (default: todo.json ในโฟลเดอร์ปัจจุบัน) [env: TODO_FILE=] [default: todo.json]
  -h, --help         Print help (see more with '--help')
  -V, --version      Print version
```

```
$ todo --help
todo: CLI จัดการรายการสิ่งที่ต้องทำ พร้อมบันทึกข้อมูลลงไฟล์ JSON

ตัวอย่างการรวมความรู้จาก Part 44-45 (derive macro), Part 57-58 (serde), Part 10 (enum/match), และ Part 12/30-31 (error handling + exit code)

Usage: todo [OPTIONS] <COMMAND>

Commands:
  add       เพิ่มงานใหม่
  list      แสดงรายการงานทั้งหมด (หรือเฉพาะที่ยังไม่เสร็จด้วย --pending)
  complete  ทำเครื่องหมายว่างานเสร็จแล้ว
  remove    ลบงานออกจากรายการ
  help      Print this message or the help of the given subcommand(s)

Options:
      --file <FILE>
          พาธของไฟล์เก็บข้อมูล (default: todo.json ในโฟลเดอร์ปัจจุบัน)
          
          [env: TODO_FILE=]
          [default: todo.json]

  -h, --help
          Print help (see a summary with '-h')

  -V, --version
          Print version
```

สังเกตความต่างที่ชัดเจน: `-h` ใช้**บรรทัดแรก**ของ doc comment (`about = "จัดการรายการสิ่งที่ต้องทำแบบง่าย"` ที่เราตั้งไว้ตรง ๆ ผ่าน attribute จริง ๆ ก็มีผลเหนือ doc comment เสมอถ้าตั้งไว้) และแสดง option แบบบรรทัดเดียวกระชับ ในขณะที่ `--help` ใช้**ทั้งสองพารากราฟ**ของ doc comment และแสดงรายละเอียดของแต่ละ option แบบหลายบรรทัด (รวม `[env: ...]` และ `[default: ...]` แยกบรรทัดให้อ่านง่ายขึ้น) — `clap` เลือก format ให้เหมาะกับ**ปริมาณเนื้อหา**โดยอัตโนมัติ นี่คือสิ่งที่ทำให้ประโยคในเป้าหมายบทนี้ที่ว่า "clap generate help text ที่ขัดเกลามาแล้ว" เป็นจริงในระดับรายละเอียดที่คุณไม่ต้องคิดเรื่อง formatting เองเลย

มาดู `--help` ของแต่ละ subcommand และการรันจริงแบบครบ end-to-end (สร้างไฟล์ใหม่ ทดสอบทุก subcommand ตรวจสอบแล้วจริง):

```
$ todo add --help
เพิ่มงานใหม่

Usage: todo add [OPTIONS] <TITLE>

Arguments:
  <TITLE>  ข้อความอธิบายงาน

Options:
      --file <FILE>          พาธของไฟล์เก็บข้อมูล (default: todo.json ในโฟลเดอร์ปัจจุบัน) [env: TODO_FILE=] [default: todo.json]
  -p, --priority <PRIORITY>  ระดับความสำคัญของงาน [default: medium] [possible values: low, medium, high]
  -h, --help                 Print help
```

สังเกต `[possible values: low, medium, high]` ที่ `clap` เพิ่มให้เองจาก `#[derive(ValueEnum)]` — ผู้ใช้เห็นตัวเลือกที่ถูกต้องทั้งหมดโดยไม่ต้องไปเปิดอ่าน source code

รันจริงทีละคำสั่งบนไฟล์ใหม่ (`demo_todo.json` ที่ยังไม่มีอยู่มาก่อน):

```
$ todo --file demo_todo.json add "ซื้อนม"
เพิ่มงาน #1: "ซื้อนม" (priority: medium)

$ todo --file demo_todo.json add "ทำความสะอาดบ้าน" --priority low
เพิ่มงาน #2: "ทำความสะอาดบ้าน" (priority: low)

$ todo --file demo_todo.json add "ส่งรายงาน" --priority high
เพิ่มงาน #3: "ส่งรายงาน" (priority: high)

$ todo --file demo_todo.json list
[ ] #1 ซื้อนม (priority: medium)
[ ] #2 ทำความสะอาดบ้าน (priority: low)
[ ] #3 ส่งรายงาน (priority: high)

$ todo --file demo_todo.json complete 1
ทำเครื่องหมายงาน #1 ว่าเสร็จแล้ว

$ todo --file demo_todo.json list --pending
[ ] #2 ทำความสะอาดบ้าน (priority: low)
[ ] #3 ส่งรายงาน (priority: high)

$ todo --file demo_todo.json remove 2
ลบงาน #2 แล้ว

$ todo --file demo_todo.json list
[x] #1 ซื้อนม (priority: medium)
[ ] #3 ส่งรายงาน (priority: high)
```

สังเกตว่า `list --pending` กรองงาน `#1` ที่เสร็จแล้วออกไป และ `remove 2` ลบงาน `#2` ออกจากไฟล์จริง (ไม่ใช่แค่ใน memory) — ลองดูไฟล์ `demo_todo.json` หลังการรันทั้งหมด (แสดงจริงจากผลลัพธ์ `serde_json::to_string_pretty`):

```json
{
  "next_id": 4,
  "tasks": [
    {
      "id": 1,
      "title": "ซื้อนม",
      "priority": "medium",
      "done": true
    },
    {
      "id": 3,
      "title": "ส่งรายงาน",
      "priority": "high",
      "done": false
    }
  ]
}
```

สังเกตว่า `priority` ถูกเก็บเป็น `"medium"`/`"high"` (lowercase) ตามที่ `#[serde(rename_all = "lowercase")]` กำหนดไว้บน `enum Priority` — ตรงกับรูปแบบที่ `--priority` รับจาก CLI (ผ่าน `ValueEnum`) พอดี ทำให้ค่าที่ผู้ใช้พิมพ์บน command line กับค่าที่เก็บในไฟล์ JSON**มีหน้าตาเดียวกัน** ไม่ต้องแปลงกลับไปกลับมาให้สับสน

และทดสอบกรณี id ไม่มีอยู่จริง (exit code 3 ตามที่ตั้งไว้):

```
$ todo --file demo_todo.json complete 999
error: ไม่พบงาน id 999
$ echo $?
3
```

โปรเจกต์นี้แสดงให้เห็นภาพรวมของบทนี้ครบทุกส่วน: **derive macro** (`Parser`, `Subcommand`, `ValueEnum`) จาก Part 44-45 ที่ทำให้เขียน struct/enum ธรรมดาแล้วได้ CLI parser เต็มรูปแบบ, **serde** (`Serialize`, `Deserialize`) จาก Part 57-58 ที่ทำให้ persist ข้อมูลลง JSON ได้ในไม่กี่บรรทัด, **enum/match แบบ exhaustive** จาก Part 10 ที่การันตีว่าทุก subcommand ถูกจัดการ, และ **error handling พร้อม exit code ที่สื่อความหมาย** จาก Part 12/30-31 — ทั้งหมดนี้ประกอบกันเป็นเครื่องมือ command-line ที่ใช้งานได้จริง ไม่ใช่แค่ตัวอย่างสอนทฤษฎีลอย ๆ

### 59.10 ตารางสรุป: Attribute และ Type ที่ใช้บ่อยที่สุดของ `clap`

ก่อนเข้าหัวข้อกับดัก มาสรุปภาพรวมของสิ่งที่เรียนมาทั้งบทเป็นตารางอ้างอิงเดียว (quick reference) เพื่อให้กลับมาเปิดดูได้ง่ายเวลาเขียน CLI จริงในอนาคต โดยไม่ต้องไล่อ่านทั้งบทซ้ำ:

**Derive macro ของ `clap` ทั้ง 4 ตัว:**

| Derive macro | ใช้กับ | หน้าที่ |
|---|---|---|
| `Parser` | struct ระดับบนสุดที่เป็น "โปรแกรมทั้งตัว" | generate `impl Parser` ให้เรียก `Cli::parse()` ได้ตรง ๆ — ต้องมีตัวเดียวต่อโปรแกรม (หรือต่อ binary ถ้าเป็น workspace ที่มีหลาย binary) |
| `Subcommand` | enum ที่ตัวแทน "หนึ่งในหลายคำสั่งย่อย" | แต่ละ variant กลายเป็น subcommand หนึ่งตัว, field ของ variant คือ argument ของ subcommand นั้น |
| `Args` | struct ที่เป็น "กลุ่มของ argument" ที่ไม่ใช่ทั้งโปรแกรม | ใช้คู่กับ `#[command(flatten)]` เพื่อแชร์กลุ่ม argument ระหว่างหลาย subcommand/หลายจุด |
| `ValueEnum` | enum ที่ตัวแทน "ชุดตัวเลือกที่จำกัด" สำหรับ argument ตัวเดียว | ใช้คู่กับ `#[arg(value_enum)]` — ได้ validation อัตโนมัติพร้อม `[possible values: ...]` ใน `--help` |

**Type ของ field และความหมายที่ `clap` ตีความให้ (type-directed parsing จากหัวข้อ 59.3):**

| Type ของ field | ความหมาย |
|---|---|
| `String`, `u32`, `f64`, `PathBuf`, ... | argument ที่จำเป็น (required), parse ด้วย `FromStr` |
| `Option<T>` | argument ที่ไม่จำเป็น, ไม่ใส่ = `None` |
| `bool` | flag แบบสวิตช์ (ไม่รับค่าตามมา), ไม่ใส่ = `false` |
| `Vec<T>` | argument ที่รับได้หลายค่า (ใส่ flag ซ้ำได้หลายครั้ง) |
| `enum` ที่ derive `ValueEnum` + `#[arg(value_enum)]` | argument ที่ค่าต้องอยู่ในชุดตัวเลือกที่จำกัด |

**Attribute ที่ใช้บ่อยที่สุด:**

| Attribute | ระดับที่ใช้ | หน้าที่ |
|---|---|---|
| `#[command(name = "...")]` | struct ระดับบนสุด | ชื่อโปรแกรมที่แสดงใน `Usage:`/error message |
| `#[command(version)]` | struct ระดับบนสุด | เปิด `-V`/`--version` โดยอ่านค่าจาก `CARGO_PKG_VERSION` |
| `#[command(author)]` / `#[command(author = "...")]` | struct ระดับบนสุด | ตั้งค่า metadata ผู้เขียน (มีผลจำกัดต่อ `--help` มาตรฐาน — ดูกับดักข้อ 6) |
| `#[command(about = "...")]` | struct ระดับบนสุด/variant ของ subcommand | คำอธิบายสั้น ๆ ที่แสดงใน `--help`/รายการ subcommand — ถ้าไม่ระบุจะอ่านจาก doc comment แทน |
| `#[command(subcommand)]` | field ที่ type เป็น enum ที่ derive `Subcommand` | บอกว่า field นี้คือจุดเลือก subcommand |
| `#[command(flatten)]` | field ที่ type เป็น struct ที่ derive `Args` | แผ่ field ของ struct นั้นเข้ามาปนกับ argument อื่นในระดับเดียวกัน |
| `#[arg(short)]` / `#[arg(short = 'x')]` | field ของ argument | เปิด short flag — ถ้าไม่ระบุตัวอักษร ใช้ตัวแรกของชื่อ field |
| `#[arg(long)]` / `#[arg(long = "xxx")]` | field ของ argument | เปิด long flag — ถ้าไม่ระบุชื่อ แปลง `snake_case` เป็น `kebab-case` ให้ |
| `#[arg(default_value = "...")]` | field ของ argument | ค่า default แบบ string (parse อีกทีด้วย `FromStr` ของ type field) |
| `#[arg(default_value_t = ...)]` | field ของ argument | ค่า default แบบ Rust literal ตรง ๆ (ตรวจ type ได้ตอน compile time) |
| `#[arg(value_name = "...")]` | field ของ argument | ชื่อ placeholder ที่แสดงใน `--help` (ไม่มีผลต่อการ parse) |
| `#[arg(value_parser = fn_name)]` | field ของ argument | custom validation/parsing logic ที่ซับซ้อนกว่า `FromStr` ธรรมดา |
| `#[arg(value_parser = [..])]` | field ของ argument ที่เป็น `String` | จำกัดค่าที่รับได้ให้อยู่ในรายการ string ที่กำหนด โดยไม่ต้องเขียนฟังก์ชันเอง |
| `#[arg(value_enum)]` | field ที่ type เป็น enum ที่ derive `ValueEnum` | ใช้ enum นั้นเป็นชุดตัวเลือกของ argument นี้ |
| `#[arg(env = "...")]` | field ของ argument | fallback ไปอ่าน environment variable ถ้าไม่ได้ใส่ค่าผ่าน CLI (CLI ชนะเสมอถ้าใส่ทั้งคู่) |
| `#[arg(global = true)]` | field ของ argument ในระดับบนสุด | ให้ argument นี้ใช้ได้ทั้งระดับบนสุดและในทุก subcommand โดยไม่ต้องประกาศซ้ำ |

**สิ่งที่ `clap` generate ให้เสมอโดยไม่ต้องประกาศเอง** (ตราบใดที่มี `#[derive(Parser)]`/`#[derive(Subcommand)]`): `-h`/`--help`, `-V`/`--version` (ถ้าเปิด `#[command(version)]`), subcommand `help` (ถ้ามี subcommand), exit code `2` เมื่อ parse ล้มเหลว, และ error message ที่จัดรูปแบบสม่ำเสมอทุกจุด — นี่คือ "ส่วนที่ฟรี" ที่คุณได้มาโดยไม่ต้องเขียนโค้ดสักบรรทัด เทียบกับปัญหาทั้ง 4 ข้อในหัวข้อ 59.1 ที่ต้องเขียนเองทั้งหมดถ้าไม่ใช้ `clap`

### 59.11 ทดสอบ CLI ด้วย `Cli::try_parse_from` โดยไม่ต้อง spawn process จริง

กับดักข้อ 4 (ลำดับ field กำหนดลำดับ positional argument) แนะนำให้เขียน integration test ที่รัน binary จริง แต่การรัน binary จริงทุกครั้ง (ผ่าน `std::process::Command` หรือ crate อย่าง `assert_cmd`) มีต้นทุน: ต้อง compile binary ให้เสร็จก่อน, ต้อง spawn process ใหม่ทุก test case (ช้ากว่าการเรียกฟังก์ชันธรรมดามาก), และทดสอบแค่ "ผลลัพธ์สุดท้ายที่พิมพ์ออกมา" ไม่ได้ทดสอบ "ค่าที่ parse ได้จริง" ตรง ๆ

`clap` มีทางเลือกที่เบากว่ามากสำหรับทดสอบ **เฉพาะส่วนการ parse** โดยไม่ต้อง spawn process เลย: **`Cli::try_parse_from(...)`** — รับ iterator ของ string (จำลอง argument ที่ผู้ใช้จะพิมพ์ รวม argument ตัวที่ 0 ที่ปกติเป็นชื่อโปรแกรม) แล้วคืน `Result<Cli, clap::Error>` ตรง ๆ แทนการ panic/exit เหมือน `Cli::parse()` — เหมาะกับการเขียน **unit test** (Part 32) ที่ทดสอบ logic การ parse ล้วน ๆ แยกจาก logic ของโปรแกรม:

```rust
use clap::Parser;

#[derive(Parser, Debug, PartialEq)]
#[command(name = "test_parse_demo")]
struct Cli {
    #[arg(short, long)]
    verbose: bool,

    name: String,
}

fn main() {
    let cli = Cli::parse();
    println!("{cli:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_name_and_verbose_flag() {
        let cli = Cli::try_parse_from(["test_parse_demo", "--verbose", "Alice"]).unwrap();
        assert_eq!(cli, Cli { verbose: true, name: "Alice".to_string() });
    }

    #[test]
    fn missing_required_name_is_an_error() {
        let result = Cli::try_parse_from(["test_parse_demo", "--verbose"]);
        assert!(result.is_err());
    }
}
```

รันด้วย `cargo test` แล้วผ่านทั้งสอง test (ตรวจสอบจริง):

```
running 2 tests
test tests::parses_name_and_verbose_flag ... ok
test tests::missing_required_name_is_an_error ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

สังเกตจุดสำคัญ: **`#[derive(PartialEq)]` บน `Cli`** จำเป็นสำหรับ `assert_eq!` เพราะเราต้องเทียบค่า `Cli` ทั้งก้อนกับค่าที่คาดหวังตรง ๆ (Part 9/19 สอนไว้แล้วว่า `assert_eq!` ต้องการ `PartialEq` และ `Debug` เพื่อเทียบค่าและพิมพ์ error ถ้าไม่ตรงกัน) — และ test ที่สองยืนยันว่าเมื่อขาด argument ที่จำเป็น (`name`) `try_parse_from` คืน `Err` แทนการ panic ทำให้เราทดสอบ "กรณี argument ผิด" ได้โดยไม่ต้อง catch panic หรือ spawn process แยกเพื่อเช็ค exit code เลย

**เมื่อไหร่ควรใช้ `try_parse_from` (unit test) เทียบกับรัน binary จริง (integration test ตาม Part 33):** ใช้ `try_parse_from` เพื่อทดสอบว่า "โครงสร้าง argument ที่ประกาศไว้ parse ค่าที่คาดหวังได้ถูกต้องไหม" (ครอบคลุมกับดักข้อ 4 เรื่องลำดับ positional ได้ตรงจุดและเร็วกว่ามาก เพราะไม่ต้อง compile/spawn binary ใหม่ทุกครั้ง) แต่ยังควรมี integration test ที่รัน binary จริงอย่างน้อยสำหรับ **user journey หลัก** (เช่น "เพิ่มงานแล้ว list ออกมาต้องเห็นงานนั้น" ในตัวอย่าง todo CLI) เพราะ `try_parse_from` ทดสอบแค่ชั้นการ parse ไม่ได้ทดสอบว่าโปรแกรมทำงานถูกต้องครบทั้ง pipeline (อ่านไฟล์ เขียนไฟล์ พิมพ์ผลลัพธ์) จริง ๆ

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืมเปิด feature `derive` — ได้ compile error ที่ชี้ไปผิดจุดในตอนแรก**

```toml
[dependencies]
clap = "4"   # ลืม features = ["derive"]
```

```rust
use clap::Parser;

#[derive(Parser, Debug)]
struct Cli {
    name: String,
}

fn main() {
    let cli = Cli::parse();
    println!("{}", cli.name);
}
```

Error จริงที่ได้ (ตรวจสอบแล้ว):

```
error: cannot find derive macro `Parser` in this scope
 --> src/main.rs:3:10
  |
3 | #[derive(Parser, Debug)]
  |          ^^^^^^
  |
note: `Parser` is imported here, but it is only a trait, without a derive macro
 --> src/main.rs:1:5
  |
1 | use clap::Parser;
  |     ^^^^^^^^^^^^

error[E0599]: no function or associated item named `parse` found for struct `Cli` in the current scope
 --> src/main.rs:9:20
  |
9 |     let cli = Cli::parse();
  |                    ^^^^^ function or associated item not found in `Cli`
  |
  = help: items from traits can only be used if the trait is implemented and in scope
  = note: the following traits define an item `parse`, perhaps you need to implement one of them:
          candidate #1: `Parser`
          candidate #2: `TypedValueParser`
```

**สาเหตุ:** เมื่อไม่เปิด feature `derive`, `clap` จะ export แค่ **trait** `Parser` (ที่นิยาม method `parse()` ไว้เป็น signature เปล่า ๆ) แต่ไม่ export **derive macro** ที่ generate `impl Parser for Cli` ให้ — compiler เห็น `use clap::Parser;` ว่า import ได้จริง (มันมีอยู่ในรูปแบบ trait) จึงไม่ error ตรง `use` แต่พอเจอ `#[derive(Parser)]` มันหา derive macro ชื่อนี้ไม่เจอในเนมสเปซของ macro (คนละเนมสเปซจาก trait ตามที่ Part 44 อธิบายเรื่องเนมสเปซของ macro ไว้) จึง error ตรงนั้น และเมื่อไม่มี `impl Parser for Cli` ที่มาจาก derive มา `Cli` ก็ไม่มี method `parse()` ให้เรียกจริง ๆ ตามข้อความ error ที่สอง **วิธีแก้:** เปิด feature ให้ถูกต้องด้วย `cargo add clap --features derive` หรือแก้ `Cargo.toml` ให้เป็น `clap = { version = "4", features = ["derive"] }` — สังเกตว่านี่เป็นกับดักที่มือใหม่เจอบ่อยเป็นอันดับต้น ๆ เพราะข้อความ error แรกดูเหมือนเป็นปัญหาการ `use` ผิด แต่จริง ๆ แล้วเป็นปัญหา feature flag ของ Cargo

**2. field ชนิด `bool` เป็น "switch" ไม่ใช่ "flag ที่รับค่า" — พิมพ์ `--flag true` ไม่ได้**

มือใหม่ที่คุ้นกับภาษาอื่น (เช่น argparse ของ Python ที่บางครั้งใช้ `--flag=true`) มักคาดหวังว่า `bool` flag ของ `clap` จะรับค่าตามมาได้ด้วย:

```
$ minigrep world sample.txt --lines true
error: unexpected argument 'true' found

Usage: minigrep [OPTIONS] <PATTERN> <FILE>

For more information, try '--help'.
```

หรือแม้จะลองใช้ `=` ก็ยัง error เพราะ `bool` field ไม่ถูกออกแบบให้รับ value เลย:

```
$ minigrep world sample.txt --lines=true
error: unexpected value 'true' for '--lines' found; no more were expected

Usage: minigrep <PATTERN|FILE|--lines|--ignore-case>

For more information, try '--help'.
```

**สาเหตุ:** ตามตารางใน 59.3 field ชนิด `bool` ถูก `clap` ตีความเป็น **switch แบบ presence/absence** (ใส่ = `true`, ไม่ใส่ = `false`) ไม่ใช่ argument ที่ "รับค่าตามมาแล้ว parse เป็น `bool`" — เมื่อ `clap` เห็น `true` ตามหลัง `--lines` มันตีความ `true` เป็น**อีก argument หนึ่ง**ที่ไม่รู้จัก ไม่ใช่ค่าของ `--lines` **วิธีแก้:** ใช้ `--lines` เฉย ๆ ถ้าต้องการเปิด และไม่ใส่เลยถ้าต้องการปิด (`minigrep world sample.txt --lines`) ถ้าต้องการ argument ที่รับค่า `true`/`false` แบบชัดเจนจริง ๆ (เช่นกรณีต้องส่งผ่าน script ที่ generate ค่ามาแบบ dynamic) ให้เปลี่ยน type เป็น `Option<bool>` พร้อม `#[arg(long, value_name = "BOOL")]` หรือกำหนด `action = ArgAction::Set` เอง (นอกเหนือจาก scope ของบทนี้ แต่ควรรู้ว่าทำได้)

**3. `debug_assert!` ของ `clap` ดักการชนกันของ short flag ได้แค่ใน debug build — release build ไม่แจ้งเตือนเลย**

```rust
use clap::Parser;

#[derive(Parser, Debug)]
struct Cli {
    #[arg(short, long)]
    verbose: bool,

    #[arg(short, long)]
    version_flag: bool, // ตั้งใจให้ derive short letter ชนกับ verbose (ทั้งคู่ขึ้นต้นด้วย v)
}

fn main() {
    let cli = Cli::parse();
    println!("{cli:?}");
}
```

ใน **debug build** (`cargo run` / `cargo build` ปกติ) จะได้ panic ที่ชี้ปัญหาชัดเจนทันทีตอนรัน:

```
thread 'main' panicked at .../clap_builder-4.6.7/src/builder/debug_asserts.rs:125:17:
Command <ชื่อโปรแกรม>: Short option names must be unique for each argument, but '-v' is in use by both 'verbose' and 'version_flag'
```

แต่ใน **release build** (`cargo build --release`) การตรวจสอบนี้ (ที่ implement ด้วย `debug_assert!` ภายใน `clap` เอง) **ถูกตัดออกไปเลยเพราะ `debug_assert!` ไม่ทำงานเมื่อ compile แบบ optimize** — โปรแกรม compile ผ่านและรันได้แบบไม่มี panic แต่ผลลัพธ์ผิดเงียบ ๆ:

```
$ ./target/release/dup_short --verbose
Cli { verbose: true, version_flag: false }

$ ./target/release/dup_short -v
Cli { verbose: true, version_flag: false }
```

สังเกตว่า `-v` ตั้งค่าให้ `verbose` เป็น `true` เสมอ ไม่ว่าผู้ใช้ตั้งใจจะหมายถึง argument ตัวไหน (`version_flag` แทบเป็น dead code ในทางปฏิบัติ เพราะไม่มีทาง trigger มันผ่าน short flag ได้อีกเลย) **สาเหตุ:** `clap` ใช้ `debug_assert!` (ไม่ใช่ `assert!`) สำหรับตรวจสอบความถูกต้องเชิงโครงสร้างของ `Command` ที่ derive macro generate ให้ เพื่อไม่ให้เสียเวลา runtime ตรวจสอบซ้ำใน release build ที่ควรจะผ่านการทดสอบมาแล้ว — แต่นี่หมายความว่า **การทดสอบ CLI ของคุณด้วย `cargo run`/`cargo test` เพียงอย่างเดียวไม่พอที่จะจับปัญหานี้ได้ก่อน ship จริงด้วย `--release`** **วิธีแก้:** รัน `cargo build --release` แล้วทดสอบ (หรืออย่างน้อยรัน `cargo check --release`) เป็นส่วนหนึ่งของ CI/CD เสมอ ไม่ใช่ทดสอบแค่ debug build และตรวจทาน short flag ทุกตัวด้วยตาก่อน merge เมื่อเพิ่ม argument ใหม่ — ปัญหานี้ยิ่งสำคัญเมื่อ struct มี field จำนวนมากที่ short flag ถูกกำหนดแบบ implicit (ไม่ระบุตัวอักษรเอง อาศัยตัวอักษรแรกของชื่อ field) เพราะโอกาสชนกันโดยไม่ตั้งใจจะสูงขึ้นเมื่อ field เพิ่มขึ้น

**4. ลำดับ field ใน struct คือลำดับ positional argument จริง — สลับ field ทำให้ CLI เปลี่ยนพฤติกรรมแบบเงียบ ๆ**

กลับไปดูตัวอย่าง `minigrep` ในหัวข้อ 59.3:

```rust
#[derive(Parser, Debug)]
struct Cli {
    pattern: String,  // positional ตัวที่ 1
    path: String,      // positional ตัวที่ 2
    // ...
}
```

`minigrep world sample.txt` ทำงานถูกต้องเพราะ `pattern = "world"`, `path = "sample.txt"` ตามลำดับการประกาศ แต่ถ้ามีคนมา refactor แล้วสลับลำดับ field เพื่อ "จัดกลุ่มให้อ่านง่ายขึ้น" โดยไม่ทันคิดว่ามีผลต่อ CLI:

```rust
#[derive(Parser, Debug)]
struct Cli {
    path: String,       // ตอนนี้กลายเป็น positional ตัวที่ 1
    pattern: String,    // ตอนนี้กลายเป็น positional ตัวที่ 2
    // ...
}
```

โค้ด**compile ผ่านปกติ** (ไม่มี error หรือ warning ใด ๆ เลย เพราะ compiler ไม่มีทางรู้ว่าผู้ใช้ command line คาดหวังลำดับไหน) แต่ทุก script/alias ที่เคยเรียก `minigrep world sample.txt` จะพัง**เงียบ ๆ** เพราะตอนนี้ `path = "world"` และ `pattern = "sample.txt"` สลับความหมายกันโดยสมบูรณ์ — โปรแกรมจะพยายามเปิดไฟล์ชื่อ `"world"` (ซึ่งไม่มีอยู่) แล้ว error แบบ "หาไฟล์ไม่เจอ" ที่ดูเหมือนเป็นปัญหาคนละเรื่องกับสาเหตุจริง

**สาเหตุ:** `clap` ไม่มีทางรู้ "ความหมาย" ของ positional argument นอกจากลำดับที่ประกาศใน struct เพราะ positional argument ไม่มีชื่อ (`--xxx`) ให้ผู้ใช้ระบุอย่างชัดเจนบน command line — มันจึงต้องอาศัย**ลำดับ**เป็นสัญญาเดียวที่มี **วิธีแก้/ป้องกัน:** (ก) เขียน integration test (Part 33) ที่รัน binary จริงด้วย argument ในรูปแบบที่ผู้ใช้จะพิมพ์จริง แล้วยืนยันผลลัพธ์ ไม่ใช่แค่ unit test ที่เรียกฟังก์ชันภายในตรง ๆ ซึ่งจะไม่จับปัญหาการสลับลำดับ field แบบนี้ได้เลย (ข) ถ้า argument มีมากกว่า 1-2 ตัวและเสี่ยงสับสน ควรพิจารณาออกแบบให้เป็น**named argument** (`--pattern`/`--path`) แทน positional เพื่อไม่ต้องพึ่งลำดับเลย โดยเฉพาะถ้า type ของทั้งสอง field เหมือนกัน (เช่น `String` ทั้งคู่แบบตัวอย่างนี้) เพราะ compiler ก็ช่วยจับความสับสนแบบนี้ไม่ได้เช่นกัน (ต่าง type กันอย่างน้อยจะ parse ไม่ผ่านถ้าใส่ผิดตำแหน่ง แต่ type เดียวกันจะ parse ผ่านเสมอไม่ว่าลำดับไหน)

**5. ระบุ `value_enum` แต่ใส่ค่าไม่อยู่ใน enum — error message ที่ `clap` ให้มามีคุณภาพสูงกว่าการ validate เองมาก**

```
$ todo add "งานทดสอบ" --priority urgent
error: invalid value 'urgent' for '--priority <PRIORITY>'
  [possible values: low, medium, high]

For more information, try '--help'.
```

สังเกตว่า `clap` ไม่ได้บอกแค่ "ค่าไม่ถูกต้อง" แต่**แสดงตัวเลือกที่ถูกต้องทั้งหมดให้เลย** เทียบกับถ้าเราต้อง validate เองด้วยมือ (เช่นใน hypotheticalตัวอย่างหัวข้อ 59.1 ที่ยังไม่ได้แตะเรื่อง enum เลย) เราต้องเขียนโค้ดสร้างรายการ `possible values` เอง คอยอัปเดตให้ตรงกับ enum จริงทุกครั้งที่เพิ่ม variant ใหม่ — ในขณะที่ `clap` ทำสิ่งนี้ให้อัตโนมัติจาก `#[derive(ValueEnum)]` ตรง ๆ นี่ไม่ใช่ "ข้อผิดพลาด" ที่ต้องแก้ แต่เป็นข้อสังเกตสำคัญที่ควรจำ: **เมื่อมีชุดค่าที่จำกัดไว้ล่วงหน้า ให้ใช้ `enum` + `ValueEnum` เสมอ แทนการรับเป็น `String` แล้ว validate เองด้วย `if`/`match`** เพราะจะได้ error message คุณภาพนี้แบบไม่ต้องเขียนเพิ่มเลย และยังได้ประโยชน์ของ enum แบบ Part 10 (exhaustive match, ไม่มีทาง "ลืม" validate) มาด้วยในตัว

**6. `#[command(author)]` ไม่แสดงผลถ้า `Cargo.toml` ไม่มีฟิลด์ `authors` — ไม่ error แต่เงียบไปเฉย ๆ**

ถ้า `Cargo.toml` ของโปรเจกต์ไม่มีฟิลด์ `authors = [...]` (พบได้บ่อยในโปรเจกต์ใหม่ที่สร้างด้วย `cargo new` เวอร์ชันหลัง ๆ ซึ่งไม่ได้เติมฟิลด์นี้ให้อัตโนมัติเสมอไป) การใส่ `#[command(author)]` จะ**ไม่ error อะไรเลย** แต่ก็ไม่มีผลลัพธ์ปรากฏชัดเจนใน `--help` ด้วย (ตรวจสอบแล้วจากตัวอย่างในบทนี้ทุกตัวที่ `Cargo.toml` ไม่มีฟิลด์ `authors` — `--help`/`--version` ไม่มีชื่อผู้เขียนโชว์เลย) **สาเหตุ:** `author` (ไม่ใส่ค่า) ดึงข้อมูลจาก `env!("CARGO_PKG_AUTHORS")` ที่ Cargo เตรียมไว้ตอน compile time จากฟิลด์ `authors` ใน `Cargo.toml` — ถ้าฟิลด์นั้นไม่มีหรือเป็นค่าว่าง ตัวแปร environment นี้ก็จะเป็น string เปล่า และไม่มีอะไรให้แสดง **วิธีแก้:** ถ้าต้องการให้ชื่อผู้เขียนปรากฏจริง ต้องเติมฟิลด์ `authors = ["ชื่อของคุณ <email>"]` ใน `Cargo.toml` เอง หรือใส่ค่าตรง ๆ ผ่าน `#[command(author = "ชื่อที่ต้องการ")]` แทนการใช้ค่า default — และควรทราบว่าถึงจะตั้งค่าถูก `author` ก็มักไม่ปรากฏในหน้าตา `--help` มาตรฐานของ `clap` เว้นแต่ตั้ง help template เองเพิ่มเติม (ต่าง จาก `about`/`version` ที่ปรากฏชัดเจนโดย default) — จึงไม่ใช่ attribute ที่มีประโยชน์มากนักถ้าไม่ได้ปรับแต่ง template เพิ่ม เมื่อเทียบกับ `version`/`about`/`long_about` ที่ให้ผลลัพธ์ที่มองเห็นได้ทันที

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** ขยาย `minigrep` จากหัวข้อ 59.3 ให้มี argument เพิ่มอีกหนึ่งตัวคือ `--count`/`-c` (type `bool`) — เมื่อใส่ flag นี้ ให้โปรแกรมพิมพ์**จำนวนบรรทัดที่เจอ pattern** เท่านั้น (ไม่ต้องพิมพ์เนื้อหาบรรทัด) แทนพฤติกรรมปกติ ทดสอบว่า `minigrep world sample.txt --count` พิมพ์ตัวเลขจำนวนบรรทัดที่มีคำว่า "world" ถูกต้อง และยืนยันว่า `--help` แสดง `-c, --count` พร้อมคำอธิบายที่มาจาก doc comment ที่คุณเขียนไว้
   (hint: เพิ่ม field `count: bool` พร้อม `#[arg(short = 'c', long)]` และ doc comment เหนือมัน แก้ logic ใน `main()` ให้เก็บจำนวนที่เจอไว้ในตัวแปร แล้วแยก `if cli.count { println!("{n}") } else { ... }` ก่อนพิมพ์ผลลัพธ์แบบเดิม)

2. **[กลาง]** เขียน subcommand CLI ใหม่ชื่อ `notecli` ที่มี 2 subcommand: `create` (รับ `title: String` และ `#[arg(long)] tags: Vec<String>` ที่รับได้หลายค่า) และ `search` (รับ `keyword: String` เป็น positional argument) ทั้งสอง subcommand ให้ทำงานแบบ in-memory ก่อน (ยังไม่ต้อง persist ลงไฟล์) โดย `create` พิมพ์ชื่อ note และรายการ tags ทั้งหมดที่ใส่มา ส่วน `search` พิมพ์แค่ข้อความยืนยันว่าค้นหาคำนั้น ทดสอบว่า `notecli create "แผนงาน Q1" --tags work --tags urgent` เก็บ tags ได้ทั้งสองค่าใน `Vec<String>` และยืนยันด้วย `--help` ว่า `clap` แสดง `--tags <TAGS>` ถูกต้องสำหรับ argument ที่รับได้หลายค่า
   (hint: field ชนิด `Vec<String>` ตามตารางในหัวข้อ 59.3 รองรับการใส่ flag เดิมซ้ำหลายครั้งได้เองโดยอัตโนมัติ ไม่ต้องเขียน logic รวมค่าเอง — ลองสังเกตว่า `--help` ของ argument ชนิดนี้มีข้อความพิเศษต่างจาก argument ปกติหรือไม่)

3. **[ยาก]** เพิ่ม `value_parser` แบบ custom ให้ `todo add` (จากหัวข้อ 59.9) ที่ validate ว่า `title` **ต้องไม่เป็นค่าว่างและต้องมีความยาวไม่เกิน 100 ตัวอักษร** (ถ้าฝ่าเงื่อนไขใดเงื่อนไขหนึ่ง ให้ `value_parser` คืน `Err` พร้อมข้อความที่บอกสาเหตุชัดเจน) ทดสอบทั้ง 3 กรณี: title ปกติ (ผ่าน), title เป็นค่าว่าง `""` (error พร้อมข้อความที่คุณกำหนด), และ title ยาวเกิน 100 ตัวอักษร (error พร้อมข้อความที่คุณกำหนด) — ตรวจสอบว่า error message ที่ `clap` แสดงมีรูปแบบ `invalid value '...' for '<TITLE>': <ข้อความของคุณ>` ตรงตามรูปแบบที่เห็นในหัวข้อ 59.6
   (hint: เขียนฟังก์ชัน `fn parse_title(s: &str) -> Result<String, String>` แยกออกมา แล้วใส่ `#[arg(value_parser = parse_title)]` บน field `title` — เพราะ field เดิมเป็น `String` อยู่แล้ว การเปลี่ยนมาใช้ `value_parser` แบบ custom ไม่ต้องเปลี่ยน type ของ field เลย เพราะ `parse_title` คืน `String` เหมือนเดิม แค่มี logic validate เพิ่มก่อนคืนค่า)

4. **[ยาก/ประยุกต์ใช้งานจริง]** ต่อยอดจาก todo CLI ในหัวข้อ 59.9 ให้เพิ่ม subcommand ใหม่ชื่อ `export` ที่รับ `#[arg(long, value_name = "PATH")] output: PathBuf` (จำเป็นต้องระบุ) และ `#[arg(long, env = "TODO_EXPORT_FORMAT", default_value = "json", value_parser = ["json", "csv"])] format: String` — subcommand นี้อ่านข้อมูลจากไฟล์ `--file` เดิม (ที่เป็น global argument อยู่แล้ว) แล้วเขียนออกเป็นไฟล์ใหม่ตาม `--output` โดยถ้า `format = "json"` ให้เขียนแบบเดิม (ใช้ `serde_json::to_string_pretty`) แต่ถ้า `format = "csv"` ให้เขียนเองแบบ manual โดยไม่ใช้ crate `csv` (แค่ join field ด้วย comma ทีละแถว พร้อม header) ทดสอบทั้งสอง format ให้ได้ไฟล์ผลลัพธ์ที่ถูกต้อง และทดสอบว่าถ้าผู้ใช้ใส่ `--format xml` (ค่าที่ไม่อยู่ใน `["json", "csv"]`) จะได้ error ที่ `clap` แจ้งมาให้เองก่อนโค้ดของคุณแม้แต่รันด้วยซ้ำ — วัดผลว่า exit code ของแต่ละกรณี (สำเร็จ, format ผิด, `--file` เดิมหาไม่พบ) ตรงตามที่ควรจะเป็นหรือไม่ เทียบกับ exit code ที่ตั้งไว้ในหัวข้อ 59.9
   (hint: `value_parser = ["json", "csv"]` เป็นรูปแบบพิเศษของ `clap` ที่รับ array ของ string literal ตรง ๆ แล้วมันสร้าง validator ที่เช็คว่าค่าที่ใส่มาต้องอยู่ในรายการนี้เท่านั้นให้อัตโนมัติ โดยไม่ต้องเขียนฟังก์ชัน `parse_xxx` เองแบบหัวข้อ 59.6 เลย เหมาะกับกรณีที่ตัวเลือกเป็น string ธรรมดา ไม่ต้องแปลงเป็น type อื่น — ต่างจาก `enum` + `ValueEnum` ที่เหมาะกับกรณีอยากได้ type ที่ปลอดภัยกว่า `String` ไปใช้ต่อในโค้ด)

## สรุป

บทนี้เริ่มต้นด้วยคำถามเดียวกับที่ Part 44 ตั้งไว้ตอนเริ่มเรียน proc macro: "ทำไมไม่เขียนเองด้วยมือ" — เราได้เห็นด้วยโค้ดจริงว่าการ parse `std::env::args()` เองต้องจัดการรูปแบบการใส่ argument ที่หลากหลาย (`--flag value`, `--flag=value`, short flag รวมกัน), ต้องเขียน `--help` เอง, ต้องเขียน error message เองแบบไม่สม่ำเสมอ, และต้อง validate เองทุกเงื่อนไข — ทั้งหมดนี้เป็นงานที่โตแบบไม่เป็นเส้นตรงเมื่อโปรแกรมซับซ้อนขึ้น `clap` เข้ามาเติมเต็มช่องว่างนี้ด้วยแนวทาง **declarative**: เราแค่ประกาศโครงสร้างของ argument ผ่าน struct/enum และ attribute แล้วปล่อยให้ derive macro (`#[derive(Parser)]`, `#[derive(Subcommand)]`, `#[derive(ValueEnum)]`) generate ทุกอย่างที่เหลือให้ — และนี่คือจุดที่ Part 44-45 ที่คุณเรียนมาก่อนหน้านี้แสดงคุณค่าจริง: คุณไม่ได้เห็น `#[derive(Parser)]` เป็นเวทมนตร์ที่ทำงานไม่รู้จักที่มา แต่รู้ว่าเบื้องหลังมันคือ proc macro ที่ parse struct ด้วย `syn`, วิเคราะห์ field/attribute/doc comment, แล้ว generate `impl Parser` ด้วย `quote!{}` — กลไกเดียวกับที่คุณเขียนเองมาแล้วใน Part 44-45 ทุกประการ เพียงแต่ครั้งนี้มีคนอื่นเขียนมาให้แล้วอย่างสมบูรณ์และผ่านการทดสอบมานับล้านครั้งจากผู้ใช้ทั่วโลก

เราได้เห็น type-directed parsing ที่ทำให้ type ของ field (`String`, `Option<T>`, `bool`, `Vec<T>`, `enum` ที่ derive `ValueEnum`) กำหนด behavior การ parse โดยอัตโนมัติ, attribute อย่าง `short`/`long`/`default_value`/`value_name`/`env` ที่ปรับแต่งพฤติกรรมและหน้าตาของ `--help` แบบละเอียด, subcommand ผ่าน `#[derive(Subcommand)]` บน enum ที่เชื่อมกับ Part 10 เรื่อง exhaustive matching ตรง ๆ, `value_parser` แบบ custom สำหรับ validation ที่ซับซ้อนกว่า type ธรรมดา พร้อม error message ที่มีคุณภาพสูงกว่าที่เราจะเขียนเองได้อย่างสม่ำเสมอ, และ `#[arg(env = "...")]` สำหรับรองรับแนวคิด 12-factor app ที่ CLI flag override environment variable ได้เสมอ ปิดท้ายด้วยโปรเจกต์เต็ม (Todo CLI) ที่รวมทุกอย่างเข้าด้วยกันพร้อม `serde` (Part 57-58) สำหรับ persistent storage และ exit code ที่สื่อความหมายตามแนวคิดจาก Part 12/30-31

ที่สำคัญไม่แพ้กัน เราได้เห็นกับดักที่มาจากธรรมชาติของ derive macro โดยตรง: `debug_assert!` ที่ดักการชนกันของ short flag ทำงานแค่ใน debug build (กับดักข้อ 3) ซึ่งเป็นเหตุผลที่ควรทดสอบ release build ก่อน ship จริงเสมอ และลำดับ field ใน struct ที่กำหนดลำดับ positional argument โดยตรง (กับดักข้อ 4) ซึ่งเป็นรายละเอียดที่ compiler ไม่มีทางเตือนได้เลยเพราะเป็นเรื่องของ "ความหมายที่ผู้ใช้คาดหวัง" ไม่ใช่ความถูกต้องของโค้ด

หลักสูตรตอนนี้มาถึงจุดที่คุณมีเครื่องมือครบสำหรับสร้าง CLI application ระดับใช้งานจริงได้แล้ว: parse argument แบบแข็งแรง (`clap`), จัดการ error อย่างเหมาะสม (Part 12/30-31), persist ข้อมูล (`serde`, Part 57-58) แต่ยังขาดชิ้นสำคัญอีกชิ้นหนึ่งที่โปรแกรมระดับ production ทุกตัวต้องมี: **การบันทึก log ว่าโปรแกรมทำอะไรไปบ้าง** เพื่อ debug ปัญหาที่เกิดขึ้นตอนใช้งานจริง (ที่คุณนั่ง `println!` ไล่ดูเองไม่ได้อีกต่อไปเมื่อโปรแกรม deploy อยู่บนเครื่องอื่น) ใน **Part 60: Logging และ Tracing เบื้องต้น** เราจะเรียน crate `log` (มาตรฐานพื้นฐานของ ecosystem) และ `tracing` (เครื่องมือรุ่นใหม่ที่รองรับ structured logging และ async context ได้ดีกว่า) ซึ่งเป็นเครื่องมือถัดไปที่จะทำให้ CLI/application ของคุณพร้อมสำหรับการ debug และ monitor ในโลกจริง — และคุณจะได้เห็นว่า `tracing` เองก็มี attribute macro อย่าง `#[instrument]` ที่ใช้เทคนิคเดียวกับที่ Part 44 สอนไว้เช่นกัน

---

**Part ก่อนหน้า:** [Serde ขั้นสูง](part-058-serde-advanced.md) | **Part ถัดไป:** [Logging และ Tracing เบื้องต้น](part-060-logging-tracing-basics.md)
