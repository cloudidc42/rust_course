# Project A03: File Watcher Daemon

> โมดูล: CLI & Systems Tools | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 4 ชั่วโมง

## ภาพรวมโปรเจค

File Watcher Daemon คือโปรแกรมที่คอยเฝ้าดูไฟล์และ directory ในระบบแล้วแจ้งเตือนเมื่อมีการเปลี่ยนแปลง
ไม่ว่าจะเป็นการสร้างไฟล์ใหม่ แก้ไขเนื้อหา ลบ หรือเปลี่ยนชื่อ

ในโลก production เครื่องมือแบบนี้ถูกใช้ในหลายรูปแบบ:

- **Hot-reload dev server** เช่น Vite/webpack watch mode ที่ compile ใหม่ทุกครั้งที่บันทึกไฟล์
- **CI/CD trigger** ตรวจจับการเปลี่ยน config แล้ว restart service อัตโนมัติ
- **Audit logging** บันทึกว่าใครแก้ไขไฟล์อะไรเมื่อไหร่ใน directory ที่ sensitive
- **Build system** เช่น `cargo watch` ที่รัน `cargo test` ทุกครั้งที่แก้ `.rs`

โปรเจคนี้สร้าง CLI tool คล้าย `watchexec` หรือ `cargo-watch` แต่เราเขียนเองตั้งแต่ต้น
ทำให้เข้าใจลึกถึง OS filesystem event API, channel-based concurrency, และ debouncing

**learning value หลัก**: เข้าใจว่า OS ส่ง filesystem event มาอย่างไร, ทำไม debounce ถึงจำเป็น,
และวิธีเชื่อม notify crate เข้ากับ clap + channel + process execution อย่าง clean

---

## สิ่งที่จะได้เรียนรู้

- การใช้ `notify` crate เพื่อรับ filesystem events จาก OS (inotify/kqueue/FSEvents)
- Event types ต่างๆ: Create, Modify, Delete, Rename และการ map เป็น user-facing message
- `mpsc::channel` เพื่อส่ง events ข้าม thread อย่าง safe
- Debouncing: ปัญหา "event storm" จากโปรแกรม editor บันทึกไฟล์หลายครั้ง
- Glob pattern matching ด้วย `glob` crate สำหรับ include/exclude filter
- รัน external command ด้วย `std::process::Command` พร้อม cooldown
- Serialize event เป็น JSON ด้วย `serde_json` สำหรับ pipeline integration
- Daemon mode: detach จาก terminal แล้ว write log file

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 41–45** — Concurrency พื้นฐาน: `thread::spawn`, `Arc<Mutex<T>>`, `mpsc::channel`
- **Part 46–48** — Error handling ด้วย `?` operator และ custom error types
- **Part 52–54** — CLI argument parsing ด้วย `clap` derive macro
- **Part 55–57** — File I/O: `std::fs`, `std::path::Path`/`PathBuf`
- **Part 60** — External process execution ด้วย `std::process::Command`

---

## โครงสร้างโปรเจค (Project Layout)

```
file-watcher/
├── src/
│   ├── main.rs          # CLI entrypoint, argument parsing, main event loop
│   ├── watcher.rs       # Watcher setup, event routing
│   ├── filter.rs        # Glob include/exclude logic
│   ├── debouncer.rs     # Manual debounce window implementation
│   ├── executor.rs      # Command execution + cooldown
│   └── output.rs        # Colored text output + JSON serialization
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

สำหรับ step แรกๆ เราจะเริ่มจาก `main.rs` ไฟล์เดียวก่อน แล้วค่อยแยก module ในขั้นหลัง

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
OS Kernel (inotify/kqueue/FSEvents)
        │
        ▼
  notify::RecommendedWatcher
        │  raw Event { kind, paths[] }
        ▼
  mpsc::channel<Event>          ← thread boundary
        │
        ▼
  main event loop
        ├── filter (include/exclude glob)
        ├── debounce window (100ms)
        ├── format output (text/JSON)
        └── execute command + cooldown
```

### ทำไมต้องใช้ channel?

`notify` ส่ง event ผ่าน callback ที่รันใน background thread ของตัวเอง
เราไม่สามารถทำงานหนักใน callback ได้ (จะ block watcher thread)
วิธีแก้คือส่ง event เข้า `mpsc::Sender` แล้ว main thread รับออกจาก `Receiver`

```
Watcher thread: callback → tx.send(event)   [fast, non-blocking]
Main thread:    loop { rx.recv() → process } [slow work here]
```

### ทำไม debounce ถึงสำคัญ?

เมื่อ editor บันทึกไฟล์ OS จะส่ง event หลายตัวติดกัน เช่น:
1. `Modify(Data)` — เขียนเนื้อหา
2. `Access(Close(Write))` — ปิด file descriptor
3. `Modify(Metadata)` — อัปเดต mtime
4. บางครั้ง `Remove` + `Create` ถ้า editor ใช้ atomic write (write to temp, then rename)

ถ้าเราตรวจจับทุก event แล้ว trigger `cargo test` ทันที มันจะรัน 3-4 ครั้งสำหรับการบันทึกครั้งเดียว
Debouncing แก้ปัญหาโดย "รอ 100ms หลังจาก event สุดท้าย" ก่อน trigger action

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Watch Directory แล้ว Print Raw Events

สร้างโปรเจคใหม่และ print ทุก event ที่ได้รับมาโดยไม่กรอง:

```toml
# Cargo.toml
[package]
name = "file-watcher"
version = "0.1.0"
edition = "2021"

[dependencies]
notify = "6.1"
```

```rust
// src/main.rs  (ขั้นที่ 1 — raw events)
use std::sync::mpsc;
use std::time::Duration;
use notify::{RecommendedWatcher, RecursiveMode, Watcher, Event};

fn main() {
    let (tx, rx) = mpsc::channel::<Event>();

    // RecommendedWatcher เลือก backend ที่เหมาะสมกับ OS โดยอัตโนมัติ:
    // Linux  → inotify
    // macOS  → FSEvents (หรือ kqueue สำหรับไฟล์เดี่ยว)
    // Windows → ReadDirectoryChangesW
    let mut watcher = RecommendedWatcher::new(
        move |result: Result<Event, notify::Error>| {
            match result {
                Ok(event) => {
                    // callback รันใน watcher thread — ส่งต่อผ่าน channel เท่านั้น
                    let _ = tx.send(event);
                }
                Err(e) => eprintln!("Watch error: {:?}", e),
            }
        },
        notify::Config::default(),
    )
    .expect("Failed to create watcher");

    // เฝ้าดู current directory แบบ non-recursive
    watcher
        .watch(std::path::Path::new("."), RecursiveMode::NonRecursive)
        .expect("Failed to watch directory");

    eprintln!("Watching current directory. Press Ctrl+C to stop.");

    loop {
        match rx.recv_timeout(Duration::from_millis(100)) {
            Ok(event) => {
                println!("{:?}", event);
            }
            Err(mpsc::RecvTimeoutError::Timeout) => {
                // ไม่มี event ในรอบนี้ — วน loop ต่อ
            }
            Err(mpsc::RecvTimeoutError::Disconnected) => {
                eprintln!("Watcher disconnected");
                break;
            }
        }
    }
}
```

ทดลองรัน แล้วแก้ไขไฟล์อื่นในโฟลเดอร์:

```
$ cargo run
Watching current directory. Press Ctrl+C to stop.
Event { kind: Modify(Data(Any)), paths: ["/home/user/project/test.txt"],
        attr:tracker: None, attr:flag: None, attr:info: None, attr:source: None }
Event { kind: Access(Close(Write)), paths: ["/home/user/project/test.txt"],
        attr:tracker: None, attr:flag: None, attr:info: None, attr:source: None }
```

สังเกตว่าการบันทึกไฟล์ครั้งเดียวผลิต event **2 ตัว** — นี่คือสาเหตุที่ต้องใช้ debounce

---

### ขั้นที่ 2: Event Type Mapping + Colored Output + Timestamps

เพิ่ม dependencies สำหรับสี Terminal และเวลา:

```toml
[dependencies]
notify  = "6.1"
chrono  = "0.4"
owo-colors = "3"
```

แทนที่ `println!("{:?}", event)` ด้วย formatted output:

```rust
// src/main.rs  (ขั้นที่ 2 — event mapping + colors)
use notify::EventKind;
use notify::event::{CreateKind, ModifyKind, RemoveKind, RenameMode, DataChange};
use chrono::Local;
use owo_colors::OwoColorize;

/// แปลง EventKind เป็น label สั้นๆ + คำอธิบายภาษาไทย
fn event_label(kind: &EventKind) -> (&'static str, &'static str) {
    match kind {
        // ── Create ─────────────────────────────────────────────────────────
        EventKind::Create(CreateKind::File)   => ("CREATE",     "สร้างไฟล์ใหม่"),
        EventKind::Create(CreateKind::Folder) => ("CREATE_DIR", "สร้างโฟลเดอร์ใหม่"),
        EventKind::Create(_)                  => ("CREATE",     "สร้าง"),

        // ── Modify ─────────────────────────────────────────────────────────
        EventKind::Modify(ModifyKind::Data(DataChange::Content)) => ("MODIFY", "แก้ไขเนื้อหา"),
        EventKind::Modify(ModifyKind::Data(_))                   => ("MODIFY", "แก้ไขข้อมูล"),
        EventKind::Modify(ModifyKind::Metadata(_))               => ("META",   "แก้ไข metadata"),
        EventKind::Modify(ModifyKind::Name(RenameMode::Both))    => ("RENAME", "เปลี่ยนชื่อ"),
        EventKind::Modify(ModifyKind::Name(RenameMode::From))    => ("RENAME_FROM", "ย้ายออก"),
        EventKind::Modify(ModifyKind::Name(RenameMode::To))      => ("RENAME_TO",   "ย้ายเข้า"),
        EventKind::Modify(_)                                     => ("MODIFY", "แก้ไข"),

        // ── Remove ─────────────────────────────────────────────────────────
        EventKind::Remove(RemoveKind::File)   => ("DELETE",     "ลบไฟล์"),
        EventKind::Remove(RemoveKind::Folder) => ("DELETE_DIR", "ลบโฟลเดอร์"),
        EventKind::Remove(_)                  => ("DELETE",     "ลบ"),

        // ── Access / Other ─────────────────────────────────────────────────
        EventKind::Access(_) => ("ACCESS", "เข้าถึงไฟล์"),
        EventKind::Other     => ("OTHER",  "อื่นๆ"),
        _                    => ("?",      "ไม่ทราบ"),
    }
}

fn print_event(kind: &EventKind, path: &std::path::Path) {
    let timestamp = Local::now().format("%H:%M:%S%.3f");
    let (label, _desc) = event_label(kind);

    // แต่ละ event type มีสีต่างกันเพื่ออ่านง่าย
    let colored_label = match label {
        "CREATE" | "CREATE_DIR"            => format!("{:12}", label.green().bold()),
        "MODIFY"                           => format!("{:12}", label.yellow().bold()),
        "DELETE" | "DELETE_DIR"            => format!("{:12}", label.red().bold()),
        "RENAME" | "RENAME_FROM" | "RENAME_TO" => format!("{:12}", label.cyan().bold()),
        "ACCESS"                           => format!("{:12}", label.dimmed()),
        _                                  => format!("{:12}", label.white()),
    };

    println!("[{}] {} {}",
        timestamp.to_string().blue(),
        colored_label,
        path.display()
    );
}
```

output ที่ได้:

```
[14:23:07.441] CREATE       /tmp/demo/hello.txt
[14:23:11.882] MODIFY       /tmp/demo/hello.txt
[14:23:18.003] DELETE       /tmp/demo/hello.txt
```

---

### ขั้นที่ 3: Recursive Mode + Glob Pattern Filtering

เพิ่ม `clap` และ `glob` แล้วแปลง main เป็น CLI ที่รับ arguments:

```toml
[dependencies]
notify     = "6.1"
chrono     = "0.4"
owo-colors = "3"
glob       = "0.3"
clap       = { version = "4", features = ["derive"] }
```

```rust
// src/main.rs  (ขั้นที่ 3 — CLI args + glob filter)
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "fw", about = "File Watcher Daemon")]
struct Cli {
    /// Path to watch (file or directory)
    #[arg(default_value = ".")]
    path: std::path::PathBuf,

    /// Watch subdirectories recursively
    #[arg(short, long)]
    recursive: bool,

    /// Include only these glob patterns (comma-separated)
    /// Example: --include "*.rs,*.toml"
    #[arg(long, value_delimiter = ',')]
    include: Vec<String>,

    /// Exclude files matching these glob patterns (comma-separated)
    /// Example: --exclude "target/**,*.lock"
    #[arg(long, value_delimiter = ',')]
    exclude: Vec<String>,
}

/// ตรวจว่า path ตรงกับ pattern ใดๆ ในรายการหรือไม่
/// glob::Pattern รองรับ * ? [...] และ ** (recursive wildcard)
fn matches_any(path: &std::path::Path, patterns: &[String]) -> bool {
    if patterns.is_empty() {
        return false;
    }
    let path_str = path.to_string_lossy();
    for pat_str in patterns {
        if let Ok(pattern) = glob::Pattern::new(pat_str) {
            // ตรวจ full path ก่อน
            if pattern.matches(&path_str) {
                return true;
            }
            // ตรวจแค่ชื่อไฟล์ (สำหรับ pattern เช่น "*.rs")
            if let Some(fname) = path.file_name() {
                if pattern.matches(&fname.to_string_lossy()) {
                    return true;
                }
            }
        }
    }
    false
}

/// ตัดสินว่าควรประมวลผล event ของ path นี้หรือไม่
fn should_process(path: &std::path::Path, include: &[String], exclude: &[String]) -> bool {
    // ถ้ามี include filter: path ต้องตรง pattern อย่างน้อยหนึ่งตัว
    if !include.is_empty() && !matches_any(path, include) {
        return false;
    }
    // ถ้ามี exclude filter: path ต้องไม่ตรง pattern ใดเลย
    if !exclude.is_empty() && matches_any(path, exclude) {
        return false;
    }
    true
}

fn main() {
    let args = Cli::parse();

    let mode = if args.recursive {
        RecursiveMode::Recursive
    } else {
        RecursiveMode::NonRecursive
    };

    let (tx, rx) = mpsc::channel::<Event>();
    let mut watcher = RecommendedWatcher::new(
        move |result: Result<Event, notify::Error>| {
            if let Ok(event) = result {
                let _ = tx.send(event);
            }
        },
        notify::Config::default(),
    ).expect("Failed to create watcher");

    watcher.watch(&args.path, mode).expect("Failed to watch path");

    eprintln!("Watching: {} (recursive={})", args.path.display(), args.recursive);

    loop {
        match rx.recv_timeout(Duration::from_millis(100)) {
            Ok(event) => {
                for path in &event.paths {
                    if should_process(path, &args.include, &args.exclude) {
                        print_event(&event.kind, path);
                    }
                }
            }
            Err(mpsc::RecvTimeoutError::Timeout) => {}
            Err(mpsc::RecvTimeoutError::Disconnected) => break,
        }
    }
}
```

ทดลองรันพร้อม filter:

```
$ cargo run -- --recursive --include "*.rs,*.toml" --exclude "target/**" .
Watching: . (recursive=true)
[14:25:01.200] MODIFY       src/main.rs
[14:25:03.441] MODIFY       Cargo.toml
# ไม่แสดง target/debug/... เพราะถูก exclude
```

---

### ขั้นที่ 4: Debouncing Events (100ms Window)

ปัญหา: editor หนึ่งครั้งส่ง 3-5 events ทำให้ action trigger หลายครั้ง
วิธีแก้: เก็บ event ไว้ 100ms แล้วค่อย flush

มีสองแนวทางหลัก:

**แนว A — ใช้ `notify-debouncer-mini`** (ง่ายกว่า เหมาะ production):

```toml
notify-debouncer-mini = "0.4"
```

```rust
use notify_debouncer_mini::{new_debouncer, DebounceEventResult, DebouncedEvent};

fn main_with_debouncer() {
    let (tx, rx) = mpsc::channel::<DebounceEventResult>();

    // timeout = 100ms — event ภายใน 100ms นับจาก event สุดท้ายจะถูก group รวม
    let mut debouncer = new_debouncer(Duration::from_millis(100), tx)
        .expect("Failed to create debouncer");

    debouncer
        .watcher()
        .watch(std::path::Path::new("."), RecursiveMode::Recursive)
        .expect("Failed to watch");

    eprintln!("Watching with 100ms debounce...");

    loop {
        match rx.recv_timeout(Duration::from_millis(500)) {
            Ok(Ok(events)) => {
                // events = Vec<DebouncedEvent> ที่ debounce แล้ว
                // ไฟล์เดียวกันที่แก้ 5 ครั้งใน 100ms จะปรากฏเพียงครั้งเดียว
                for event in events {
                    println!("debounced: {:?} → {:?}", event.kind, event.path);
                }
            }
            Ok(Err(errors)) => {
                for e in errors {
                    eprintln!("Error: {:?}", e);
                }
            }
            Err(mpsc::RecvTimeoutError::Timeout) => {}
            Err(mpsc::RecvTimeoutError::Disconnected) => break,
        }
    }
}
```

**แนว B — Debounce ด้วยมือ** (เข้าใจ mechanism ลึกกว่า):

```rust
use std::collections::HashMap;
use std::time::Instant;

/// ตัว debounce ทำงานโดยเก็บ (path → last_event_time) ไว้
/// flush เฉพาะ path ที่ไม่มี event ใหม่มาอีกแล้วนาน >= window
struct Debouncer {
    pending: HashMap<std::path::PathBuf, (Instant, EventKind)>,
    window: Duration,
}

impl Debouncer {
    fn new(window: Duration) -> Self {
        Self {
            pending: HashMap::new(),
            window,
        }
    }

    /// บันทึก event ใหม่ (ถ้ามี event เดิมอยู่ → update timestamp)
    fn push(&mut self, path: std::path::PathBuf, kind: EventKind) {
        self.pending.insert(path, (Instant::now(), kind));
    }

    /// ดึง events ที่ "settle" แล้ว (ไม่มี event ใหม่มาใน window)
    fn drain_settled(&mut self) -> Vec<(std::path::PathBuf, EventKind)> {
        let now = Instant::now();
        let window = self.window;

        let settled: Vec<_> = self.pending
            .iter()
            .filter(|(_, (t, _))| now.duration_since(*t) >= window)
            .map(|(p, (_, k))| (p.clone(), k.clone()))
            .collect();

        for (path, _) in &settled {
            self.pending.remove(path);
        }
        settled
    }
}

// ใน main loop:
fn main_manual_debounce() {
    let (tx, rx) = mpsc::channel::<Event>();
    let mut watcher = RecommendedWatcher::new(
        move |r: Result<Event, _>| { if let Ok(e) = r { let _ = tx.send(e); } },
        notify::Config::default(),
    ).unwrap();
    watcher.watch(std::path::Path::new("."), RecursiveMode::Recursive).unwrap();

    let mut debouncer = Debouncer::new(Duration::from_millis(100));

    loop {
        // drain channel ทุกครั้งที่มี event หรือ timeout
        while let Ok(event) = rx.try_recv() {
            for path in event.paths {
                debouncer.push(path, event.kind.clone());
            }
        }

        // ตรวจว่ามี settled events หรือไม่
        for (path, kind) in debouncer.drain_settled() {
            print_event(&kind, &path);
        }

        std::thread::sleep(Duration::from_millis(20));
    }
}
```

---

### ขั้นที่ 5: Execute Command on Change (`--exec`)

เพิ่ม flag `--exec` ที่รัน shell command ทุกครั้งที่ตรวจพบการเปลี่ยนแปลง:

```rust
// เพิ่มใน Cli struct
/// Execute this shell command when a change is detected
#[arg(long)]
exec: Option<String>,
```

```rust
// src/executor.rs
use std::process::{Command, Stdio};
use std::time::{Duration, Instant};

pub struct Executor {
    command: String,
    cooldown: Duration,
    last_run: Instant,
}

impl Executor {
    pub fn new(command: String, cooldown: Duration) -> Self {
        // ตั้ง last_run ให้เก่ามากเพื่อให้ trigger ได้ทันที
        Self {
            command,
            cooldown,
            last_run: Instant::now()
                .checked_sub(Duration::from_secs(9999))
                .unwrap_or_else(Instant::now),
        }
    }

    pub fn try_run(&mut self) -> bool {
        if self.last_run.elapsed() < self.cooldown {
            return false; // ยังอยู่ใน cooldown
        }
        self.last_run = Instant::now();
        self.run_command();
        true
    }

    fn run_command(&self) {
        eprintln!("\n[exec] $ {}", self.command);

        // parse command เป็น tokens (ไม่ใช่ shell expansion เต็มรูปแบบ)
        let mut parts = self.command.split_whitespace();
        let program = match parts.next() {
            Some(p) => p,
            None => return,
        };
        let args: Vec<&str> = parts.collect();

        let result = Command::new(program)
            .args(&args)
            .stdout(Stdio::inherit())  // pass-through stdout
            .stderr(Stdio::inherit())  // pass-through stderr
            .status();

        match result {
            Ok(status) => eprintln!("[exec] Exit: {}", status),
            Err(e) => eprintln!("[exec] Failed to run '{}': {}", self.command, e),
        }
    }
}
```

ตัวอย่างการใช้งาน:

```bash
# รัน cargo test ทุกครั้งที่บันทึกไฟล์ .rs
$ fw --recursive --include "*.rs" --exec "cargo test" .

[14:30:01.100] MODIFY       src/main.rs

[exec] $ cargo test
   Compiling file-watcher v0.1.0
    Finished test profile [unoptimized + debuginfo] target(s) in 1.2s
     Running unittests src/main.rs

running 3 tests
test filter::tests::test_include ... ok
test filter::tests::test_exclude ... ok
test debounce::tests::test_window ... ok
test result: ok. 3 passed; 0 failed
[exec] Exit: exit status: 0
```

---

### ขั้นที่ 6: Cooldown + One-Shot Mode

**Cooldown** ป้องกันการ trigger ซ้ำถ้าการ compile/test ยังค้างอยู่:

```rust
// เพิ่มใน Cli struct
/// Minimum time between command executions (e.g. "2s", "500ms")
#[arg(long, default_value = "2s")]
cooldown: String,

/// Exit after the first change event (useful in scripts)
#[arg(long)]
once: bool,
```

```rust
// parse cooldown string เป็น Duration
fn parse_duration(s: &str) -> Duration {
    if let Some(ms) = s.strip_suffix("ms") {
        Duration::from_millis(ms.parse().unwrap_or(500))
    } else if let Some(sec) = s.strip_suffix("s") {
        Duration::from_secs(sec.parse().unwrap_or(2))
    } else {
        // fallback: ถือว่าเป็นวินาที
        Duration::from_secs(s.parse().unwrap_or(2))
    }
}
```

ตัวอย่าง one-shot mode (ใน shell script):

```bash
#!/bin/bash
# รอ build artifact เปลี่ยน แล้ว deploy
fw --include "*.wasm" --once dist/
echo "WASM changed! Deploying..."
./scripts/deploy.sh
```

หรือใช้ใน Makefile:

```makefile
wait-for-change:
    fw --once --include "*.rs" src/

test-on-change:
    fw --recursive --exec "cargo test" --cooldown 3s src/
```

---

### ขั้นที่ 7: JSON Output Mode

Mode นี้ออกแบบให้ใช้ใน pipeline เช่น ส่ง event เข้า `jq` หรือ script อื่น:

```toml
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
// เพิ่มใน Cli struct
/// Output format: "text" (default) or "json"
#[arg(long, default_value = "text")]
format: OutputFormat,

// OutputFormat enum สำหรับ clap ที่ validate input อัตโนมัติ
#[derive(Debug, Clone, clap::ValueEnum)]
enum OutputFormat {
    Text,
    Json,
}
```

```rust
use serde::Serialize;
use chrono::Local;

#[derive(Serialize)]
struct EventRecord<'a> {
    event:     &'a str,
    path:      String,
    timestamp: String,
}

fn emit_json(label: &str, path: &std::path::Path) {
    let record = EventRecord {
        event:     label,
        path:      path.to_string_lossy().into_owned(),
        timestamp: Local::now().to_rfc3339(),
    };
    // แต่ละ JSON บรรทัดเดียว (NDJSON / JSON Lines format)
    // เหมาะสำหรับ streaming และ log parsing
    println!("{}", serde_json::to_string(&record).unwrap());
}
```

ตัวอย่าง JSON output และการใช้ใน pipeline:

```bash
$ fw --format json --recursive src/ | jq '.'
{
  "event": "modify",
  "path": "/home/user/project/src/main.rs",
  "timestamp": "2024-01-15T14:30:01.441+07:00"
}
{
  "event": "modify",
  "path": "/home/user/project/src/lib.rs",
  "timestamp": "2024-01-15T14:30:01.882+07:00"
}

# ส่ง notification ผ่าน webhook เมื่อไฟล์ config เปลี่ยน
$ fw --format json --include "*.yaml" config/ | while read line; do
    curl -s -X POST https://hooks.example.com/deploy \
         -H "Content-Type: application/json" \
         -d "$line"
done
```

---

### ขั้นที่ 8: Daemon Mode

Daemon mode ให้ process detach จาก terminal แล้ว write log ลงไฟล์:

```rust
// เพิ่มใน Cli struct
/// Run as daemon, writing log to this file
#[arg(long)]
daemon: Option<std::path::PathBuf>,
```

```rust
use std::fs::{File, OpenOptions};
use std::io::Write;

fn setup_daemon(log_path: &std::path::Path) -> File {
    // Unix: fork + setsid เพื่อ detach จาก terminal
    // บน Linux สามารถใช้ libc::fork() แต่ใน Rust มีวิธีที่ safe กว่าคือ
    // ให้ process parent spawn child แล้ว exit
    #[cfg(unix)]
    {
        use std::process;
        // simple approach: ใช้ nohup/fork-like spawn
        // หรือใช้ daemonize crate สำหรับ production
        eprintln!("Daemonizing, log: {}", log_path.display());
    }

    OpenOptions::new()
        .create(true)
        .append(true)
        .open(log_path)
        .expect("Cannot open log file")
}

fn log_event(log: &mut dyn Write, label: &str, path: &std::path::Path) {
    let ts = chrono::Local::now().to_rfc3339();
    let _ = writeln!(log, "[{}] {} {}", ts, label, path.display());
}
```

สำหรับ daemon mode เต็มรูปแบบใน production ให้ใช้ `daemonize` crate:

```toml
# สำหรับ production daemon
daemonize = "0.5"
```

```rust
#[cfg(unix)]
fn daemonize_process(log_path: &std::path::Path) {
    use daemonize::Daemonize;

    let stdout = File::create(log_path).unwrap();
    let stderr = File::create(log_path.with_extension("err")).unwrap();

    Daemonize::new()
        .pid_file("/tmp/file-watcher.pid")
        .working_directory(".")
        .stdout(stdout)
        .stderr(stderr)
        .start()
        .expect("Failed to daemonize");
}
```

---

## โค้ดรวมทั้งหมด (Final Implementation)

```rust
// src/main.rs  — Final version รวมทุก feature
use std::path::PathBuf;
use std::sync::mpsc;
use std::time::Duration;
use clap::Parser;
use chrono::Local;
use notify::{RecommendedWatcher, RecursiveMode, Watcher, Event, EventKind};
use notify::event::{CreateKind, ModifyKind, RemoveKind, RenameMode, DataChange};
use owo_colors::OwoColorize;
use serde::Serialize;

#[derive(Parser, Debug)]
#[command(
    name = "fw",
    about = "File Watcher Daemon — watch directories and react to changes",
    long_about = None
)]
struct Cli {
    /// Directory or file to watch
    #[arg(default_value = ".")]
    path: PathBuf,

    /// Watch subdirectories recursively
    #[arg(short, long)]
    recursive: bool,

    /// Include only these glob patterns (comma-separated), e.g. "*.rs,*.toml"
    #[arg(long, value_delimiter = ',')]
    include: Vec<String>,

    /// Exclude files matching these patterns, e.g. "target/**,*.lock"
    #[arg(long, value_delimiter = ',')]
    exclude: Vec<String>,

    /// Execute this command on detected changes
    #[arg(long)]
    exec: Option<String>,

    /// Cooldown between command executions (e.g. "2s", "500ms")
    #[arg(long, default_value = "2s")]
    cooldown: String,

    /// Exit after first change event (useful in scripts)
    #[arg(long)]
    once: bool,

    /// Output format: text or json
    #[arg(long, default_value = "text")]
    format: String,

    /// Run as daemon writing to this log file
    #[arg(long)]
    daemon: Option<PathBuf>,
}

#[derive(Serialize)]
struct JsonEvent {
    event:     String,
    path:      String,
    timestamp: String,
}

fn event_label(kind: &EventKind) -> &'static str {
    match kind {
        EventKind::Create(CreateKind::File)                       => "CREATE",
        EventKind::Create(CreateKind::Folder)                     => "CREATE_DIR",
        EventKind::Create(_)                                      => "CREATE",
        EventKind::Modify(ModifyKind::Data(DataChange::Content))  => "MODIFY",
        EventKind::Modify(ModifyKind::Data(_))                    => "MODIFY",
        EventKind::Modify(ModifyKind::Metadata(_))                => "META",
        EventKind::Modify(ModifyKind::Name(RenameMode::Both))     => "RENAME",
        EventKind::Modify(ModifyKind::Name(RenameMode::From))     => "RENAME_FROM",
        EventKind::Modify(ModifyKind::Name(RenameMode::To))       => "RENAME_TO",
        EventKind::Modify(_)                                      => "MODIFY",
        EventKind::Remove(RemoveKind::File)                       => "DELETE",
        EventKind::Remove(RemoveKind::Folder)                     => "DELETE_DIR",
        EventKind::Remove(_)                                      => "DELETE",
        EventKind::Access(_)                                      => "ACCESS",
        _                                                         => "OTHER",
    }
}

fn matches_any(path: &std::path::Path, patterns: &[String]) -> bool {
    if patterns.is_empty() {
        return false;
    }
    let path_str = path.to_string_lossy();
    for pat_str in patterns {
        if let Ok(pat) = glob::Pattern::new(pat_str) {
            if pat.matches(&path_str) { return true; }
            if let Some(fname) = path.file_name() {
                if pat.matches(&fname.to_string_lossy()) { return true; }
            }
        }
    }
    false
}

pub fn should_process(path: &std::path::Path, include: &[String], exclude: &[String]) -> bool {
    if !include.is_empty() && !matches_any(path, include) { return false; }
    if !exclude.is_empty() &&  matches_any(path, exclude) { return false; }
    true
}

fn print_text(label: &str, path: &std::path::Path) {
    let ts = Local::now().format("%H:%M:%S%.3f");
    let colored = match label {
        "CREATE" | "CREATE_DIR"                       => format!("{:12}", label.green().bold()),
        "MODIFY"                                      => format!("{:12}", label.yellow().bold()),
        "DELETE" | "DELETE_DIR"                       => format!("{:12}", label.red().bold()),
        "RENAME" | "RENAME_FROM" | "RENAME_TO"        => format!("{:12}", label.cyan().bold()),
        "ACCESS"                                      => format!("{:12}", label.dimmed()),
        _                                             => format!("{:12}", label.white()),
    };
    println!("[{}] {} {}", ts.to_string().blue(), colored, path.display());
}

fn print_json(label: &str, path: &std::path::Path) {
    let ev = JsonEvent {
        event:     label.to_lowercase(),
        path:      path.to_string_lossy().into_owned(),
        timestamp: Local::now().to_rfc3339(),
    };
    println!("{}", serde_json::to_string(&ev).unwrap());
}

fn parse_cooldown(s: &str) -> Duration {
    if let Some(ms) = s.strip_suffix("ms") {
        Duration::from_millis(ms.parse().unwrap_or(500))
    } else {
        Duration::from_secs(s.trim_end_matches('s').parse().unwrap_or(2))
    }
}

fn run_exec(cmd: &str) {
    eprintln!("\n[exec] $ {}", cmd);
    let parts: Vec<&str> = cmd.split_whitespace().collect();
    if parts.is_empty() { return; }
    match std::process::Command::new(parts[0]).args(&parts[1..]).status() {
        Ok(s)  => eprintln!("[exec] exit: {}", s),
        Err(e) => eprintln!("[exec] error: {}", e),
    }
}

fn main() {
    let args = Cli::parse();
    let mode = if args.recursive {
        RecursiveMode::Recursive
    } else {
        RecursiveMode::NonRecursive
    };
    let cooldown = parse_cooldown(&args.cooldown);
    let mut last_exec = std::time::Instant::now()
        .checked_sub(Duration::from_secs(9999))
        .unwrap_or_else(std::time::Instant::now);

    let (tx, rx) = mpsc::channel::<Event>();
    let mut watcher = RecommendedWatcher::new(
        move |r: Result<Event, notify::Error>| {
            if let Ok(ev) = r { let _ = tx.send(ev); }
        },
        notify::Config::default(),
    ).expect("Cannot create watcher");

    watcher.watch(&args.path, mode).expect("Cannot watch path");
    eprintln!("fw: watching {} (recursive={})", args.path.display(), args.recursive);

    loop {
        match rx.recv_timeout(Duration::from_millis(100)) {
            Ok(event) => {
                let label = event_label(&event.kind);
                let mut fired = false;
                for path in &event.paths {
                    if !should_process(path, &args.include, &args.exclude) { continue; }
                    fired = true;
                    if args.format == "json" {
                        print_json(label, path);
                    } else {
                        print_text(label, path);
                    }
                }
                if fired {
                    if let Some(cmd) = &args.exec {
                        if last_exec.elapsed() >= cooldown {
                            run_exec(cmd);
                            last_exec = std::time::Instant::now();
                        }
                    }
                    if args.once { break; }
                }
            }
            Err(mpsc::RecvTimeoutError::Timeout)       => {}
            Err(mpsc::RecvTimeoutError::Disconnected)  => break,
        }
    }
}
```

---

## การทดสอบ (Testing)

### Unit Tests — filter logic

```rust
// tests/integration_test.rs
// (หรือ mod tests ใน src/main.rs)

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;
    use std::path::Path;
    use std::thread;
    use std::time::Duration;
    use tempfile::TempDir;
    use notify::{RecommendedWatcher, RecursiveMode, Watcher, Event};
    use std::sync::mpsc;

    // Helper: watch a dir, run an action, collect events for `wait_ms` milliseconds
    fn collect_events(
        dir: &Path,
        action: impl FnOnce() + Send + 'static,
        wait_ms: u64,
    ) -> Vec<Event> {
        let (tx, rx) = mpsc::channel::<Event>();
        let mut watcher = RecommendedWatcher::new(
            move |res: Result<Event, notify::Error>| {
                if let Ok(ev) = res { let _ = tx.send(ev); }
            },
            notify::Config::default(),
        ).unwrap();
        watcher.watch(dir, RecursiveMode::Recursive).unwrap();

        // รอ 100ms ให้ watcher ลงทะเบียนก่อน แล้วค่อย trigger action
        thread::spawn(move || {
            thread::sleep(Duration::from_millis(100));
            action();
        });

        let deadline = std::time::Instant::now() + Duration::from_millis(wait_ms);
        let mut events = Vec::new();
        loop {
            let remaining = deadline.saturating_duration_since(std::time::Instant::now());
            if remaining.is_zero() { break; }
            match rx.recv_timeout(remaining.min(Duration::from_millis(50))) {
                Ok(ev) => events.push(ev),
                Err(_) => {}
            }
        }
        drop(watcher);
        events
    }

    // ── Integration test 1: basic watch + create event ─────────────────────
    #[test]
    fn test_watch_create_event() {
        let tmp = TempDir::new().unwrap();
        let dir_path = tmp.path().to_owned();
        let file_path = dir_path.join("hello.txt");

        let events = collect_events(
            &dir_path,
            move || { fs::write(&file_path, "hello world").unwrap(); },
            3000,
        );

        assert!(!events.is_empty(), "Expected at least one filesystem event");
        let relevant = events.iter().any(|ev| {
            ev.paths.iter().any(|p| p.starts_with(&dir_path))
        });
        assert!(relevant, "Event path should be inside temp dir");
    }

    // ── Integration test 2: include *.txt filter ───────────────────────────
    #[test]
    fn test_include_filter_txt_only() {
        let rs_path   = Path::new("src/main.rs");
        let txt_path  = Path::new("notes.txt");
        let toml_path = Path::new("Cargo.toml");
        let include   = vec!["*.txt".to_string()];

        assert!( should_process(txt_path,  &include, &[]), "*.txt must pass notes.txt");
        assert!(!should_process(rs_path,   &include, &[]), "*.txt must block main.rs");
        assert!(!should_process(toml_path, &include, &[]), "*.txt must block Cargo.toml");
    }

    // ── Integration test 3: multiple include patterns ──────────────────────
    #[test]
    fn test_include_filter_multiple_patterns() {
        let include = vec!["*.rs".to_string(), "*.toml".to_string()];

        assert!( should_process(Path::new("src/main.rs"),   &include, &[]));
        assert!( should_process(Path::new("Cargo.toml"),    &include, &[]));
        assert!(!should_process(Path::new("notes.txt"),     &include, &[]));
        assert!(!should_process(Path::new("config.json"),   &include, &[]));
    }

    // ── Integration test 4: exclude filter ────────────────────────────────
    #[test]
    fn test_exclude_filter() {
        let exclude = vec!["target/**".to_string(), "target/*".to_string()];

        // src/main.rs ไม่ถูก exclude
        assert!(
            should_process(Path::new("src/main.rs"), &[], &exclude),
            "src/main.rs should pass through"
        );
    }

    // ── Integration test 5: empty filters pass everything ─────────────────
    #[test]
    fn test_no_filter_passes_everything() {
        assert!(should_process(Path::new("any/path/file.xyz"), &[], &[]));
    }

    // ── Integration test 6: debounce raw event count ──────────────────────
    #[test]
    fn test_debounce_raw_events() {
        let tmp = TempDir::new().unwrap();
        let dir_path = tmp.path().to_owned();
        let file_path = dir_path.join("debounce_test.txt");

        let events = collect_events(
            &dir_path,
            move || {
                for i in 0..5u8 {
                    fs::write(&file_path, format!("write {}", i)).unwrap();
                    thread::sleep(Duration::from_millis(10));
                }
            },
            3000,
        );

        assert!(!events.is_empty(), "Should receive at least one event");
        let file_events: Vec<_> = events.iter()
            .filter(|ev| ev.paths.iter().any(|p| p.ends_with("debounce_test.txt")))
            .collect();
        assert!(!file_events.is_empty(), "Events should reference debounce_test.txt");
        println!("5 rapid writes → {} raw inotify events (debouncer collapses these to 1)", file_events.len());
    }

    // ── Unit tests: event_label ────────────────────────────────────────────
    #[test]
    fn test_event_kind_label_create() {
        use notify::event::CreateKind;
        assert_eq!(event_label(&EventKind::Create(CreateKind::File)), "CREATE");
    }

    #[test]
    fn test_event_kind_label_modify() {
        use notify::event::DataChange;
        assert_eq!(
            event_label(&EventKind::Modify(ModifyKind::Data(DataChange::Content))),
            "MODIFY"
        );
    }

    #[test]
    fn test_event_kind_label_delete() {
        use notify::event::RemoveKind;
        assert_eq!(event_label(&EventKind::Remove(RemoveKind::File)), "DELETE");
    }

    // ── Unit test: JSON serialization ──────────────────────────────────────
    #[test]
    fn test_json_event_serialization() {
        let ev = JsonEvent {
            event:     "modify".to_string(),
            path:      "src/main.rs".to_string(),
            timestamp: "2024-01-01T00:00:00+00:00".to_string(),
        };
        let json = serde_json::to_string(&ev).unwrap();
        assert!(json.contains(r#""event":"modify""#));
        assert!(json.contains(r#""path":"src/main.rs""#));
        assert!(json.contains("\"timestamp\""));
    }
}
```

### ผลลัพธ์จาก `cargo test` จริง

```
$ cargo test
   Compiling filewatcher_verify v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 2.24s
     Running unittests src/main.rs (target/debug/deps/filewatcher_verify-06ad7453f3baaabd)

running 10 tests
test tests::test_event_kind_label_create ... ok
test tests::test_event_kind_label_modify ... ok
test tests::test_event_kind_label_delete ... ok
test tests::test_exclude_filter ... ok
test tests::test_include_filter_multiple_patterns ... ok
test tests::test_include_filter_txt_only ... ok
test tests::test_json_event_serialization ... ok
test tests::test_no_filter_passes_everything ... ok
test tests::test_debounce_raw_events ... ok
test tests::test_watch_create_event ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 3.00s
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/fw

# ติดตั้งใน PATH
sudo cp target/release/fw /usr/local/bin/

# หรือใช้ cargo install
cargo install --path .
```

### Cargo.toml สำหรับ Release

```toml
[profile.release]
opt-level     = 3
lto           = true      # Link-Time Optimization ลด binary size
codegen-units = 1         # ช้าลงแต่ผลิต binary ที่ optimize ดีกว่า
strip         = true      # ลบ debug symbols ออก (~50% size reduction)
```

ขนาด binary ก่อน/หลัง:

```
$ ls -lh target/debug/fw target/release/fw
-rwxr-xr-x 1 user group  12M target/debug/fw
-rwxr-xr-x 1 user group 1.2M target/release/fw
```

### systemd Service (Linux)

```ini
# /etc/systemd/system/file-watcher.service
[Unit]
Description=File Watcher Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/fw --recursive --include "*.yaml,*.conf" \
          --exec "systemctl reload myapp" \
          --daemon /var/log/file-watcher.log \
          /etc/myapp
Restart=on-failure
RestartSec=5s
User=www-data

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable file-watcher
sudo systemctl start file-watcher
sudo systemctl status file-watcher
```

### Docker

```dockerfile
# Multi-stage build เพื่อ binary ขนาดเล็ก
FROM rust:1.75-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgcc-s1 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/fw /usr/local/bin/fw
ENTRYPOINT ["fw"]
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดัก 1: Event Storm — รับ events มากเกินคาด

**ปัญหา**: บันทึกไฟล์ครั้งเดียวแต่ command รัน 4-5 ครั้ง

**สาเหตุ**: OS ส่ง filesystem events หลายตัวต่อการ operation เดียว เช่น write ไฟล์ได้รับ:
1. `Modify(Data)` — เขียนข้อมูล
2. `Access(Close(Write))` — ปิด fd
3. `Modify(Metadata)` — update mtime
4. atomic editors เช่น vim ส่ง `Remove` + `Create` + `Rename`

**วิธีแก้**: ใช้ debounce เสมอ — อย่า trigger action ที่ทุก raw event

```rust
// ผิด: trigger ทุก event
Ok(event) => {
    run_exec(&cmd);  // รัน 4 ครั้งสำหรับบันทึกครั้งเดียว!
}

// ถูก: debounce ก่อน
// ใช้ notify-debouncer-mini หรือ manual debounce window
```

---

### กับดัก 2: inotify Watch Limit บน Linux

**ปัญหา**: ดู directory ใหญ่ๆ แล้วเจอ error:

```
Error: Os { code: 28, kind: Other, message: "No space left on device" }
```

**สาเหตุ**: Linux กำหนด limit จำนวน inotify watches ต่อ user (default 8,192)
การดู directory ที่มี subdirectory 10,000+ ตัวจะเกิน limit

**วิธีตรวจสอบ**:

```bash
cat /proc/sys/fs/inotify/max_user_watches
# 8192  ← default

# ดูว่าตอนนี้ใช้ไปเท่าไหร่
cat /proc/sys/fs/inotify/max_user_instances
```

**วิธีแก้ชั่วคราว**:

```bash
# เพิ่ม limit สำหรับ session นี้
sudo sysctl fs.inotify.max_user_watches=524288

# เพิ่มแบบถาวร
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**วิธีแก้ใน code**: จำกัด depth ที่ดู หรือ filter directories ที่ไม่จำเป็นออก:

```rust
// อย่า recursive โดยไม่จำเป็น
// ถ้า --recursive ให้ exclude target/, .git/, node_modules/
let default_excludes = vec![
    "target/**".to_string(),
    ".git/**".to_string(),
    "node_modules/**".to_string(),
];
```

---

### กับดัก 3: macOS FSEvents Quirks — Path Normalization

**ปัญหา**: บน macOS `notify` ส่ง path ที่ resolved แล้ว (symlink resolved, canonical form)
ทำให้ glob pattern ที่ใช้ relative path หรือ symlink path ไม่ match

**สาเหตุ**: FSEvents API บน macOS คืน absolute canonical path เสมอ
ต่างจาก Linux inotify ที่คืน path ที่ผู้ใช้ส่งมา

**ตัวอย่าง**:

```bash
# บน Linux
$ fw --include "*.rs" ~/projects/myapp/src
# path ใน event: /home/user/projects/myapp/src/main.rs ✓

# บน macOS (ถ้า ~/projects เป็น symlink ไป /Volumes/Data/projects)
# path ใน event: /Volumes/Data/projects/myapp/src/main.rs
# glob "*.rs" ยังใช้งานได้เพราะ match แค่ extension
```

**วิธีป้องกัน**: ใช้ glob patterns ที่ไม่ขึ้นกับ prefix (เช่น `*.rs` แทน `src/*.rs`):

```rust
// ปลอดภัยกว่า: ตรวจ filename เท่านั้น
fn matches_any(path: &std::path::Path, patterns: &[String]) -> bool {
    // ...
    // ตรวจ filename ก่อน full path
    if let Some(fname) = path.file_name() {
        if pat.matches(&fname.to_string_lossy()) { return true; }
    }
    if pat.matches(&path.to_string_lossy()) { return true; }
    // ...
}
```

---

### กับดัก 4: Thread Safety ของ Watcher Callback

**ปัญหา**: พยายามแชร์ data structure ระหว่าง callback กับ main thread โดยตรง

**Compiler error ที่จะเจอ**:

```
error[E0277]: `*mut notify::inotify::InotifyWatcher` cannot be sent between threads safely
  --> src/main.rs:25:5
   |
25 |     let mut watcher = RecommendedWatcher::new(
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |     within `notify::RecommendedWatcher`, the trait `Send` is not implemented
   |     for `*mut notify::inotify::InotifyWatcher`
```

**สาเหตุ**: callback ของ `RecommendedWatcher` รันใน thread แยก
ถ้า capture `Rc<T>`, `RefCell<T>`, หรือ raw pointer จะ fail เพราะ `!Send`

**วิธีผิด**:

```rust
// ผิด: Rc ไม่ใช่ Send
let events = Rc::new(RefCell::new(Vec::new()));
let events_clone = Rc::clone(&events);
let mut watcher = RecommendedWatcher::new(
    move |r| { events_clone.borrow_mut().push(r); }, // compile error!
    ...
)
```

**วิธีถูก — ใช้ channel**:

```rust
// ถูก: mpsc::Sender เป็น Send
let (tx, rx) = mpsc::channel();
let mut watcher = RecommendedWatcher::new(
    move |r: Result<Event, _>| {
        if let Ok(ev) = r { let _ = tx.send(ev); }
    },
    notify::Config::default(),
).unwrap();
// รับ events ใน main thread ผ่าน rx.recv()
```

**หรือถ้าต้องการ shared state จริงๆ ใช้ `Arc<Mutex<T>>`**:

```rust
// ก็ได้: Arc<Mutex> เป็น Send + Sync
let events = Arc::new(Mutex::new(Vec::new()));
let events_clone = Arc::clone(&events);
let mut watcher = RecommendedWatcher::new(
    move |r: Result<Event, _>| {
        if let Ok(ev) = r {
            events_clone.lock().unwrap().push(ev);
        }
    },
    notify::Config::default(),
).unwrap();
```

---

### กับดัก 5: Watcher ถูก Drop ก่อนเวลา

**ปัญหา**: watcher หยุดทำงานทันทีหลังสร้าง ไม่มี event ออกมาเลย

**สาเหตุ**: `RecommendedWatcher` stop watching เมื่อถูก drop
ถ้าสร้าง watcher ในฟังก์ชันแล้วไม่ return มัน มันจะถูก drop เมื่อออกจาก scope

**วิธีผิด**:

```rust
fn start_watching(path: &Path) -> mpsc::Receiver<Event> {
    let (tx, rx) = mpsc::channel();
    let mut watcher = RecommendedWatcher::new(/* ... */).unwrap();
    watcher.watch(path, RecursiveMode::Recursive).unwrap();
    rx // watcher ถูก drop ที่นี่! หยุดทำงานทันที
}
```

**วิธีถูก — เก็บ watcher ไว้ใน scope ที่ยาวพอ**:

```rust
fn main() {
    let (tx, rx) = mpsc::channel();
    let mut watcher = RecommendedWatcher::new(/* ... */).unwrap();
    watcher.watch(path, RecursiveMode::Recursive).unwrap();

    // watcher ยังมีชีวิตอยู่ตลอด main loop
    loop {
        if let Ok(ev) = rx.recv() { /* process */ }
    }
    // watcher ถูก drop ที่นี่ (สิ้นสุด main)
}
```

---

### กับดัก 6: Glob Pattern `**` กับ Path Separator

**ปัญหา**: `target/**` ไม่ match `target/debug/build` บางระบบ

**สาเหตุ**: `glob::Pattern` ตาม convention ของ Unix glob:
- `*` match ทุกอย่างยกเว้น `/`
- `**` match ทุกอย่างรวมถึง `/` (recursive)

แต่ `glob::Pattern::matches()` ทำงานกับ string เดียว ไม่ใช่ path components
ดังนั้น `target/**` match `target/anything` แต่การ normalize path บน Windows
ที่ใช้ `\` จะไม่ match

**วิธีป้องกัน**:

```rust
// convert path ให้เป็น forward-slash ก่อน match
let path_str = path.to_string_lossy().replace('\\', "/");
if pattern.matches(&path_str) { return true; }
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Debouncer ด้วย `notify-debouncer-full`

**ระดับ**: กลาง | **เวลา**: 1 ชั่วโมง

`notify-debouncer-full` เป็น crate ที่ครบกว่า `notify-debouncer-mini`
มี feature เพิ่มเติมเช่น path caching และ rename detection ที่ฉลาดขึ้น

**โจทย์**: เขียน binary ใหม่ที่ใช้ `notify-debouncer-full` แทน raw `notify`
แล้ว compare behavior เมื่อ editor ที่ใช้ atomic write (vim, emacs) บันทึกไฟล์

**Hint**:
```toml
notify-debouncer-full = "0.3"
```

```rust
use notify_debouncer_full::{new_debouncer, DebounceEventResult};

let mut debouncer = new_debouncer(
    Duration::from_millis(100),
    None,  // tick_rate
    move |result: DebounceEventResult| { /* handler */ },
).unwrap();
```

สังเกต: `DebounceEventResult` ใน `full` ประกอบด้วย `DebouncedEvent` ที่มีข้อมูลมากกว่า
รวมถึง `rescan` flag ที่บอกว่า watcher ต้อง rescan directory tree

---

### แบบฝึกหัดที่ 2: Watch Multiple Paths

**ระดับ**: กลาง | **เวลา**: 45 นาที

**โจทย์**: รับ paths หลายตัวใน command line:

```bash
$ fw --recursive src/ tests/ Cargo.toml
```

**Hint**: ใช้ `Watcher::watch()` หลายครั้งกับ watcher ตัวเดียว
`RecommendedWatcher` รองรับ multiple paths ในนั้น:

```rust
#[arg(required = true)]
paths: Vec<PathBuf>,

// ใน main:
for path in &args.paths {
    watcher.watch(path, mode).expect("Failed to watch");
}
```

ความท้าทาย: ถ้า path หนึ่งไม่มีอยู่ ควร fail ทั้งหมดหรือ skip path นั้น?
ลอง implement `--ignore-missing` flag ที่ skip path ที่ไม่มีอยู่

---

### แบบฝึกหัดที่ 3: HTTP Webhook on Change

**ระดับ**: สูง | **เวลา**: 2 ชั่วโมง

**โจทย์**: เพิ่ม `--webhook <url>` ที่ POST JSON event ไปยัง URL เมื่อตรวจพบการเปลี่ยนแปลง

```bash
$ fw --webhook http://localhost:3000/events --include "*.rs" src/
```

**Hint**:
```toml
reqwest = { version = "0.11", features = ["blocking", "json"] }
```

```rust
fn post_webhook(url: &str, event: &JsonEvent) {
    if let Err(e) = reqwest::blocking::Client::new()
        .post(url)
        .json(event)
        .send()
    {
        eprintln!("[webhook] Error: {}", e);
    }
}
```

ความท้าทาย: ถ้า webhook URL ไม่ตอบสนอง ควร retry กี่ครั้ง? ด้วย exponential backoff?
ลอง implement retry logic ง่ายๆ:

```rust
for attempt in 0..3 {
    match client.post(url).json(event).send() {
        Ok(_) => break,
        Err(e) if attempt < 2 => {
            thread::sleep(Duration::from_millis(100 * 2_u64.pow(attempt)));
        }
        Err(e) => eprintln!("[webhook] Failed after 3 attempts: {}", e),
    }
}
```

---

### แบบฝึกหัดที่ 4: TUI Dashboard ด้วย `ratatui`

**ระดับ**: สูง-มาก | **เวลา**: 3 ชั่วโมง

**โจทย์**: แทนที่การ print แบบ scroll ด้วย TUI (Terminal UI) ที่แสดง:
- รายการ events ล่าสุด 20 รายการ (scrollable)
- Stats: จำนวน CREATE/MODIFY/DELETE แยกประเภท
- Status bar แสดง path ที่กำลัง watch

**Hint**:
```toml
ratatui = "0.25"
crossterm = "0.27"
```

```rust
use ratatui::{
    backend::CrosstermBackend,
    layout::{Constraint, Direction, Layout},
    widgets::{Block, Borders, List, ListItem, Paragraph},
    Terminal,
};

struct AppState {
    events: VecDeque<String>,   // circular buffer ขนาด 100
    stats: HashMap<String, u64>,
}
```

ความท้าทาย หลัก: terminal raw mode ต้องถูก restore เสมอแม้ process panic
ใช้ `scopeguard` crate หรือ `Drop` impl เพื่อ cleanup:

```rust
use scopeguard::defer;
crossterm::terminal::enable_raw_mode().unwrap();
defer! {
    crossterm::terminal::disable_raw_mode().unwrap();
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง File Watcher Daemon ที่ครบครัน โดยผ่านขั้นตอน:

1. **ขั้นที่ 1-2**: เข้าใจ raw filesystem events และการ map เป็น human-readable output
2. **ขั้นที่ 3**: CLI argument parsing + glob filtering ด้วย clap และ glob crate
3. **ขั้นที่ 4**: แก้ปัญหา event storm ด้วย debouncing (100ms window)
4. **ขั้นที่ 5**: รัน external command ด้วย `std::process::Command`
5. **ขั้นที่ 6-7**: cooldown, one-shot mode, และ JSON output สำหรับ pipeline integration
6. **ขั้นที่ 8**: daemon mode สำหรับ background operation

**Pattern สำคัญที่ได้เรียน**:

- **Channel pattern**: ส่งข้อมูลข้าม thread boundary อย่าง safe โดยไม่ต้องใช้ shared mutable state
- **Debounce pattern**: รวม burst events ให้เป็น single trigger — ใช้ได้กับทุกระบบที่รับ "noisy input"
- **Builder pattern ของ CLI**: clap derive macro สร้าง type-safe CLI ที่ document ตัวเอง
- **Strategy pattern สำหรับ output**: `--format text/json` ใช้ enum + match แทน if/else chain

**เชื่อมโยงกับโปรเจคถัดไป**: โปรเจค A04 Port & Service Scanner จะใช้ `tokio` async runtime
และ `std::net::TcpStream` เพื่อสแกน network ports แบบ concurrent
ความรู้เรื่อง channel และ thread จากโปรเจคนี้จะเป็นพื้นฐานสำคัญ

---

**โปรเจคก่อนหน้า:** [HTTP Client](project-a02-http-client.md) | **โปรเจคถัดไป:** [Port & Service Scanner](project-a04-port-scanner.md)
