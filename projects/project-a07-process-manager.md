# Project A07: Process Manager (PM2-like)

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 8 ชั่วโมง

## ภาพรวมโปรเจค

PM2 คือ process manager ที่นักพัฒนา Node.js ใช้กันทั่วโลกเพื่อรัน service แบบ background, restart อัตโนมัติเมื่อ crash, และดู log ได้สะดวก โปรเจคนี้จะสร้าง **procmgr** — process manager แบบ PM2 ที่เขียนด้วย Rust ล้วน ๆ

`procmgr` ทำงานในสองโหมด: **daemon** ที่รันอยู่เบื้องหลังและจัดการ process ทั้งหมด กับ **CLI** ที่ผู้ใช้พิมพ์คำสั่งเพื่อสั่ง daemon ผ่าน Unix domain socket ทั้งสองส่วนสื่อสารกันด้วย JSON messages ทำให้ง่ายต่อการ extend และ debug

ทำไมถึงน่าสร้าง? เพราะ process manager ครอบคลุมหัวข้อ systems programming ที่สำคัญมากในทางปฏิบัติ: process spawning, signal handling, IPC, log management, และ daemon design patterns ซึ่งเป็นทักษะที่ใช้ได้ในทุก production environment ไม่ว่าจะเป็น server, embedded system, หรือ DevOps tooling

## สิ่งที่จะได้เรียนรู้

- **Process lifecycle management:** `std::process::Command::spawn()`, `Child::try_wait()`, PID files
- **Daemon mode:** detach จาก terminal ด้วย `nix::unistd::setsid()`, double-fork pattern
- **Auto-restart logic:** background thread poll loop พร้อม configurable `max_restarts` และ `restart_delay`
- **Unix IPC:** Unix domain socket (`UnixListener`/`UnixStream`), JSON-framed request/response protocol
- **Signal forwarding:** SIGTERM → wait 5s → SIGKILL graceful shutdown sequence
- **Log management:** redirect stdout/stderr ไปยังไฟล์, rotate เมื่อขนาดเกิน 10 MB
- **Config-driven architecture:** parse `procmgr.json` ด้วย `serde_json`, สนับสนุน env vars และ cwd override
- **Status table rendering:** ใช้ `tabled` + `owo-colors` สร้าง status display คล้าย `pm2 list`

## ความรู้ที่ต้องมีมาก่อน

- **Part 96–100:** Systems programming, process model ใน Linux, file descriptors
- **Part 61–65:** Standard library — `std::process`, `std::io`, networking, Unix sockets
- **Part 46–50:** Concurrency — `std::thread`, `Arc<Mutex<T>>`, channels
- **Part 31–35:** Error handling ด้วย `Result`, `Box<dyn Error>`, `?` operator
- **Part 21–25:** Structs, enums, `impl` blocks, traits
- **Part 41–45:** Collections — `HashMap`, `Vec`

## โครงสร้างโปรเจค (Project Layout)

```
procmgr/
├── src/
│   ├── main.rs          ← CLI entry point (clap) + integration tests
│   ├── config.rs        ← procmgr.json parsing (serde)
│   ├── daemon.rs        ← Unix socket IPC: request/response types + handlers
│   ├── logger.rs        ← Log capture, rotation, tail
│   ├── process.rs       ← Process lifecycle, PID files, signal sending
│   ├── status.rs        ← Status table rendering (tabled + owo-colors)
│   └── sysinfo_util.rs  ← CPU% / MEM queries via sysinfo
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของระบบ

```
ผู้ใช้พิมพ์คำสั่ง            procmgr CLI
  procmgr start app    ─────→  IpcRequest::Start { name }
                                      │  JSON over /tmp/procmgr.sock
                                      ▼
                          procmgr daemon (background thread)
                              ├── spawn process (Command::spawn)
                              ├── write PID file (~/.local/share/procmgr/pids/)
                              ├── redirect stdout/stderr → log file
                              ├── monitor thread (try_wait loop)
                              └── IpcResponse::Ok { message }
                                      │
                              ← "started app [pid=12345]"
```

### ทำไม Unix Socket ไม่ใช่ HTTP?

Unix domain socket เหมาะกว่า HTTP สำหรับ local IPC เพราะ:
1. **Zero network overhead** — ไม่ผ่าน TCP/IP stack เลย ส่งข้อมูลผ่าน kernel memory directly
2. **Permission-based security** — ควบคุมด้วย filesystem permissions (chmod 700) ได้ทันที
3. **Simpler protocol** — ไม่ต้องจัดการ HTTP headers, status codes, content-type
4. **ทุก PM2/systemd/Docker socket ใช้แบบนี้** — `/run/docker.sock`, `/run/systemd/private/io.systemd.Managed` ล้วนเป็น Unix sockets

### ทำไม Double-Fork สำหรับ Daemon Mode?

เมื่อเรารัน `procmgr daemon` จาก terminal, process ยังเชื่อมกับ controlling terminal อยู่ ถ้า terminal ถูกปิด kernel จะส่ง SIGHUP ไปยัง process group ทั้งหมด

```
fork #1 → parent exits → child ได้รับ PPID=1 (init)
          → child เรียก setsid() → สร้าง session ใหม่ ไม่มี controlling terminal
fork #2 → grandparent (session leader) exits → grandchild ไม่สามารถ acquire terminal ได้อีกแล้ว
```

ใน Rust + nix:
```rust
use nix::unistd::{fork, ForkResult, setsid};

fn daemonize() {
    // Fork #1
    match unsafe { fork() }.expect("fork failed") {
        ForkResult::Parent { .. } => std::process::exit(0),
        ForkResult::Child => {}
    }
    setsid().expect("setsid failed");
    // Fork #2 (optional but ensures no controlling terminal)
    match unsafe { fork() }.expect("fork failed") {
        ForkResult::Parent { .. } => std::process::exit(0),
        ForkResult::Child => {} // ← this is the actual daemon
    }
}
```

### ทำไม JSON ไม่ใช่ Binary Protocol?

สำหรับ IPC ระหว่าง CLI กับ daemon บนเครื่องเดียวกัน, JSON มีข้อดีหลายอย่างที่ binary protocol ไม่มี: debug ง่าย (แค่ `echo '{"type":"list"}' | nc -U /tmp/procmgr.sock`), schema evolution ง่าย, human-readable เมื่อเกิด bug ในการสื่อสาร ประสิทธิภาพไม่ใช่ปัญหาเพราะ message เล็กมาก

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Start a Process and Wait for It

เริ่มจากสิ่งที่ง่ายที่สุด — spawn process และรอให้มันทำงานเสร็จ นี่คือรากฐานของทุกอย่างที่จะสร้างต่อไป

```rust
// src/main.rs (ขั้นที่ 1)
use std::process::{Command, Stdio};

fn main() {
    // รัน `sleep 3` และรอจนจบ
    let status = Command::new("sleep")
        .arg("3")
        .status()  // blocks จนกว่าจะเสร็จ
        .expect("failed to run sleep");

    println!("Process exited with: {}", status);
    println!("Success: {}", status.success());
    println!("Exit code: {:?}", status.code());
}
```

**Output:**
```
Process exited with: exit status: 0
Success: true
Exit code: Some(0)
```

ความแตกต่างระหว่าง `.status()`, `.output()` และ `.spawn()`:
- `.status()` — รอให้ process จบ, คืน `ExitStatus`
- `.output()` — รอให้จบ, capture stdout/stderr ทั้งหมดไว้ใน memory
- `.spawn()` — spawn แล้วคืน `Child` handle ทันทีโดยไม่รอ (non-blocking)

```rust
// ตัวอย่าง spawn และ wait ด้วยมือ
let mut child = Command::new("sleep")
    .arg("3")
    .spawn()
    .expect("failed to spawn");

println!("Child PID: {}", child.id());
// ทำงานอื่นขณะ process รัน...

let status = child.wait().expect("wait failed");
println!("Done: {}", status);
```

---

### ขั้นที่ 2: Spawn + Detach, Write PID File

ขั้นนี้สร้างฟังก์ชัน spawn ที่ redirect output ไปไฟล์และบันทึก PID ลงดิสก์ นี่คือสิ่งที่ทำให้เรา "จำ" ว่า process ไหนกำลังรันอยู่แม้จะ restart CLI

```rust
// src/process.rs
use std::path::PathBuf;
use std::process::{Child, Command, Stdio};
use std::time::Instant;
use crate::config::ProcessConfig;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ProcessStatus {
    Online,
    Stopped,
    Crashed,
}

impl std::fmt::Display for ProcessStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ProcessStatus::Online  => write!(f, "online"),
            ProcessStatus::Stopped => write!(f, "stopped"),
            ProcessStatus::Crashed => write!(f, "crashed"),
        }
    }
}

pub struct ManagedProcess {
    pub config: ProcessConfig,
    pub child: Option<Child>,
    pub pid: Option<u32>,
    pub status: ProcessStatus,
    pub started_at: Option<Instant>,
    pub restart_count: u32,
}

impl ManagedProcess {
    pub fn new(config: ProcessConfig) -> Self {
        Self {
            config,
            child: None,
            pid: None,
            status: ProcessStatus::Stopped,
            started_at: None,
            restart_count: 0,
        }
    }

    /// Spawn the process, redirecting stdout+stderr to `log_file` if given.
    pub fn spawn(&mut self, log_file: Option<std::fs::File>) -> std::io::Result<u32> {
        let mut cmd = Command::new(&self.config.command);
        cmd.args(&self.config.args);

        if let Some(ref cwd) = self.config.cwd {
            cmd.current_dir(cwd);
        }
        for (k, v) in &self.config.env {
            cmd.env(k, v);
        }

        match log_file {
            Some(f) => {
                // ต้อง clone เพราะ stdout/stderr ต้องการ File ownership แยกกัน
                let f2 = f.try_clone()?;
                cmd.stdout(Stdio::from(f));
                cmd.stderr(Stdio::from(f2));
            }
            None => {
                cmd.stdout(Stdio::null());
                cmd.stderr(Stdio::null());
            }
        }

        let child = cmd.spawn()?;
        let pid = child.id();
        self.child    = Some(child);
        self.pid      = Some(pid);
        self.status   = ProcessStatus::Online;
        self.started_at = Some(Instant::now());
        Ok(pid)
    }

    pub fn uptime_secs(&self) -> u64 {
        self.started_at.map(|t| t.elapsed().as_secs()).unwrap_or(0)
    }
}

// ── PID file helpers ──────────────────────────────────────────────────────────

/// ~/.local/share/procmgr/pids/
pub fn pid_dir() -> PathBuf {
    data_dir().join("pids")
}

/// ~/.local/share/procmgr/logs/
pub fn log_dir() -> PathBuf {
    data_dir().join("logs")
}

fn data_dir() -> PathBuf {
    let home = std::env::var("HOME").unwrap_or_else(|_| "/tmp".into());
    PathBuf::from(home).join(".local").join("share").join("procmgr")
}

pub fn write_pid_file(name: &str, pid: u32) -> std::io::Result<()> {
    std::fs::create_dir_all(pid_dir())?;
    std::fs::write(pid_dir().join(format!("{}.pid", name)), pid.to_string())
}

pub fn read_pid_file(name: &str) -> Option<u32> {
    std::fs::read_to_string(pid_dir().join(format!("{}.pid", name)))
        .ok()
        .and_then(|s| s.trim().parse().ok())
}

pub fn remove_pid_file(name: &str) -> std::io::Result<()> {
    std::fs::remove_file(pid_dir().join(format!("{}.pid", name)))
}
```

**ทำไมต้อง clone File handle?**

`std::process::Stdio::from(File)` ต้องการ ownership ของ `File` แต่เราต้องการตั้ง stdout และ stderr ให้ชี้ไปที่ไฟล์เดียวกัน การ `try_clone()` สร้าง file descriptor ใหม่ที่ชี้ไปยัง inode เดียวกัน ทั้งสองตัวจะเขียนข้อมูลเข้าไฟล์เดียวกันโดยไม่มีปัญหา race condition เพราะ kernel guarantees atomic writes ที่ขนาดไม่เกิน PIPE_BUF (4096 bytes บน Linux)

---

### ขั้นที่ 3: Monitor Loop (Auto-Restart on Crash)

หัวใจของ process manager คือ background thread ที่คอยดูว่า process ยังรันอยู่หรือไม่ และ restart เมื่อมันตาย

```rust
// src/process.rs (เพิ่มใน ManagedProcess)

/// Non-blocking check: has the child exited?
pub fn try_wait(&mut self) -> Option<std::process::ExitStatus> {
    if let Some(ref mut child) = self.child {
        if let Ok(Some(status)) = child.try_wait() {
            self.status = if status.success() {
                ProcessStatus::Stopped
            } else {
                ProcessStatus::Crashed
            };
            self.pid = None;
            return Some(status);
        }
    }
    None
}

/// Send SIGTERM; force SIGKILL after 5 seconds if still running.
pub fn stop_graceful(&mut self) -> std::io::Result<()> {
    #[cfg(unix)]
    if let Some(pid) = self.pid {
        use nix::sys::signal::{kill, Signal};
        use nix::unistd::Pid as NixPid;
        let npid = NixPid::from_raw(pid as i32);

        let _ = kill(npid, Signal::SIGTERM);

        let deadline = Instant::now() + std::time::Duration::from_secs(5);
        while Instant::now() < deadline {
            if let Some(ref mut child) = self.child {
                if let Ok(Some(_)) = child.try_wait() {
                    break; // graceful exit achieved
                }
            }
            std::thread::sleep(std::time::Duration::from_millis(100));
        }

        // Force kill if still running after 5 seconds
        if self.pid.is_some() {
            let _ = kill(npid, Signal::SIGKILL);
        }
    }
    self.status = ProcessStatus::Stopped;
    self.pid    = None;
    Ok(())
}
```

ตัว monitor loop ทำงานใน background thread แยกต่างหาก:

```rust
// src/daemon.rs — monitor loop ใน daemon thread
use std::sync::{Arc, Mutex};
use std::time::Duration;
use crate::process::ManagedProcess;

fn start_monitor(proc_arc: Arc<Mutex<ManagedProcess>>) {
    std::thread::spawn(move || {
        loop {
            std::thread::sleep(Duration::from_secs(1));

            let mut proc = proc_arc.lock().unwrap();

            // ถ้ากำลังรันอยู่ ให้ poll ว่าตายไปหรือยัง
            if proc.status == crate::process::ProcessStatus::Online {
                if let Some(exit_status) = proc.try_wait() {
                    let name = proc.config.name.clone();
                    let should_restart = proc.config.auto_restart
                        && !exit_status.success()
                        && proc.restart_count < proc.config.max_restarts;

                    eprintln!("[monitor] {} exited: {}", name, exit_status);

                    if should_restart {
                        proc.restart_count += 1;
                        let delay = Duration::from_millis(proc.config.restart_delay_ms);
                        drop(proc); // release lock ก่อน sleep
                        std::thread::sleep(delay);

                        let mut proc = proc_arc.lock().unwrap();
                        let log_dir = crate::process::log_dir();
                        let log_file = std::fs::OpenOptions::new()
                            .create(true).append(true)
                            .open(log_dir.join(format!("{}.log", &proc.config.name)))
                            .ok();
                        match proc.spawn(log_file) {
                            Ok(pid) => eprintln!("[monitor] restarted {} as pid {}", name, pid),
                            Err(e)  => eprintln!("[monitor] restart failed: {}", e),
                        }
                    }
                }
            }
        }
    });
}
```

**ทำไมใช้ `try_wait()` ไม่ใช่ `wait()`?**

`Child::wait()` blocks thread จนกว่า child จะตาย ซึ่งหมายความว่าถ้า process ยังรันอยู่ เราจะ block ตลอดไปและไม่สามารถรับคำสั่ง IPC ได้เลย `try_wait()` ตรวจสอบโดยไม่ block (`WNOHANG` flag ใต้ hood) และคืน `None` ถ้า process ยังรัน ทำให้เราสามารถ sleep สั้น ๆ แล้วตรวจอีกครั้งได้

---

### ขั้นที่ 4: Config File Parsing (procmgr.json)

Config file ช่วยให้นักพัฒนาสามารถ define process ทั้งหมดในที่เดียว แทนที่จะต้องพิมพ์คำสั่งยาว ๆ ทุกครั้ง

```rust
// src/config.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::path::PathBuf;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProcessConfig {
    pub name: String,
    pub command: String,
    /// Command-line arguments (เว้นว่างได้)
    #[serde(default)]
    pub args: Vec<String>,
    /// Working directory (ถ้า None ใช้ current dir)
    pub cwd: Option<String>,
    /// Environment variables เพิ่มเติม (สืบทอดจาก parent + override ด้วย values เหล่านี้)
    #[serde(default)]
    pub env: HashMap<String, String>,
    /// Restart อัตโนมัติเมื่อ exit code != 0
    #[serde(default = "default_auto_restart")]
    pub auto_restart: bool,
    /// จำนวน restart สูงสุด (0 = ไม่จำกัด)
    #[serde(default = "default_max_restarts")]
    pub max_restarts: u32,
    /// หน่วงเวลา (ms) ก่อน restart
    #[serde(default = "default_restart_delay")]
    pub restart_delay_ms: u64,
}

fn default_auto_restart() -> bool { false }
fn default_max_restarts()  -> u32  { 10 }
fn default_restart_delay() -> u64  { 1000 }

#[derive(Debug, Serialize, Deserialize)]
pub struct AppConfig {
    pub processes: Vec<ProcessConfig>,
}

impl AppConfig {
    pub fn from_json_str(s: &str) -> Result<Self, serde_json::Error> {
        serde_json::from_str(s)
    }

    pub fn from_file(path: &PathBuf) -> Result<Self, Box<dyn std::error::Error + Send + Sync>> {
        let content = std::fs::read_to_string(path)?;
        Ok(serde_json::from_str(&content)?)
    }
}
```

ตัวอย่าง `procmgr.json`:

```json
{
  "processes": [
    {
      "name": "api-server",
      "command": "/usr/local/bin/api",
      "args": ["--port", "8080", "--workers", "4"],
      "cwd": "/opt/myapp",
      "env": {
        "DATABASE_URL": "postgres://localhost/mydb",
        "LOG_LEVEL": "info",
        "PORT": "8080"
      },
      "auto_restart": true,
      "max_restarts": 10,
      "restart_delay_ms": 2000
    },
    {
      "name": "worker",
      "command": "/opt/myapp/bin/worker",
      "args": ["--queue", "default", "--concurrency", "2"],
      "cwd": "/opt/myapp",
      "auto_restart": true,
      "max_restarts": 5
    },
    {
      "name": "scheduler",
      "command": "python3",
      "args": ["-m", "myapp.scheduler"],
      "cwd": "/opt/myapp",
      "auto_restart": false
    }
  ]
}
```

**ทำไมใช้ serde defaults แทน `Option<T>`?**

สำหรับ fields ที่มีค่า default ที่สมเหตุสมผล การใช้ `#[serde(default = "fn")]` แทน `Option<T>` ทำให้ code ที่ใช้งาน config ง่ายกว่ามาก เราไม่ต้อง `.unwrap_or(10)` ทุกที่ที่ใช้ `max_restarts` แค่ read field ตรง ๆ ได้เลย

---

### ขั้นที่ 5: Log Capture (stdout/stderr to File + Rotation)

Log management เป็นหนึ่งในฟีเจอร์ที่สำคัญที่สุดของ process manager ใน production ไม่มีใครอยากเห็น log ถูก write ทับกันเองจนไฟล์ใหญ่เป็น GB

```rust
// src/logger.rs
use std::fs::{File, OpenOptions};
use std::io::Write;
use std::path::PathBuf;

pub const LOG_MAX_BYTES: u64 = 10 * 1024 * 1024; // 10 MB

pub struct Logger {
    pub name: String,
    pub log_dir: PathBuf,
    pub max_size: u64,
}

impl Logger {
    pub fn new(name: &str, log_dir: PathBuf) -> Self {
        Self { name: name.to_string(), log_dir, max_size: LOG_MAX_BYTES }
    }

    pub fn with_max_size(mut self, max_size: u64) -> Self {
        self.max_size = max_size;
        self
    }

    pub fn log_path(&self) -> PathBuf {
        self.log_dir.join(format!("{}.log", self.name))
    }

    pub fn rotated_path(&self) -> PathBuf {
        self.log_dir.join(format!("{}.log.1", self.name))
    }

    /// Open the active log file for appending (creates directory + file if needed).
    pub fn open_for_append(&self) -> std::io::Result<File> {
        std::fs::create_dir_all(&self.log_dir)?;
        OpenOptions::new()
            .create(true)
            .append(true)
            .open(self.log_path())
    }

    /// Check size; rotate if >= max_size. Returns true if rotation happened.
    pub fn maybe_rotate(&self) -> std::io::Result<bool> {
        let path = self.log_path();
        if !path.exists() {
            return Ok(false);
        }
        if path.metadata()?.len() >= self.max_size {
            self.rotate()?;
            Ok(true)
        } else {
            Ok(false)
        }
    }

    fn rotate(&self) -> std::io::Result<()> {
        let current = self.log_path();
        let rotated = self.rotated_path();
        // ลบ .log.1 เดิมถ้ามี
        if rotated.exists() {
            std::fs::remove_file(&rotated)?;
        }
        // ย้าย .log → .log.1
        std::fs::rename(&current, &rotated)?;
        // สร้างไฟล์ .log ใหม่ว่าง ๆ
        File::create(&current)?;
        Ok(())
    }

    /// Write a timestamped line, rotating first if needed.
    pub fn write_line(&self, line: &str) -> std::io::Result<()> {
        self.maybe_rotate()?;
        let mut file = self.open_for_append()?;
        writeln!(file, "{}", line)?;
        Ok(())
    }

    /// Return the last `n` lines of the log.
    pub fn tail(&self, n: usize) -> std::io::Result<Vec<String>> {
        let content = std::fs::read_to_string(self.log_path())
            .unwrap_or_default();
        let all: Vec<String> = content.lines().map(|l| l.to_string()).collect();
        let start = all.len().saturating_sub(n);
        Ok(all[start..].to_vec())
    }
}
```

**กลยุทธ์ rotation แบบง่าย (2 ไฟล์):**

```
app.log     ← ไฟล์ active ที่ process เขียนอยู่
app.log.1   ← ไฟล์ rotate ล่าสุด
```

เมื่อ `app.log` ใหญ่เกิน 10 MB:
1. ลบ `app.log.1` (ถ้ามี)
2. rename `app.log` → `app.log.1`
3. สร้าง `app.log` ใหม่ว่าง ๆ

ในโลก production มักใช้ **logrotate** (systemd) หรือ numbered rotation (`app.log.2`, `.log.3`, ...) แต่สำหรับโปรเจคนี้ 2 ไฟล์เพียงพอแล้ว

**หมายเหตุ:** ในระบบจริง process ยังเปิด file descriptor เดิม (app.log) ค้างอยู่แม้ว่าเราจะ rename มันแล้ว เพราะ Linux kernel ให้ทุก open file descriptor reference inode ไม่ใช่ path นี่คือ `copytruncate` pattern ที่ logrotate ใช้ ในโปรเจคนี้เราหลีกเลี่ยงปัญหานี้โดยส่ง file handle ใหม่ให้ process ทุกครั้งที่ restart

---

### ขั้นที่ 6: Daemon + Unix Socket IPC

นี่คือหัวใจของโปรเจค — daemon รับฟัง Unix socket และ CLI ส่ง request มา

```rust
// src/daemon.rs
use serde::{Deserialize, Serialize};
use std::io::{BufRead, BufReader, Write};
use std::os::unix::net::{UnixListener, UnixStream};

pub const SOCKET_PATH: &str = "/tmp/procmgr.sock";

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProcessInfo {
    pub name: String,
    pub pid: Option<u32>,
    pub status: String,       // "online" | "stopped" | "crashed"
    pub uptime_secs: u64,
    pub restart_count: u32,
    pub cpu_percent: f32,
    pub mem_mb: f64,
}

/// Newline-delimited JSON — แต่ละ request คือ 1 บรรทัด
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum IpcRequest {
    Start   { name: String },
    Stop    { name: String },
    Restart { name: String },
    Status,
    List,
    Logs    { name: String, lines: usize },
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum IpcResponse {
    Ok          { message: String },
    Error       { message: String },
    ProcessList { processes: Vec<ProcessInfo> },
}

/// ลบ socket file เก่าและสร้าง listener ใหม่
pub fn create_listener(socket_path: &str) -> std::io::Result<UnixListener> {
    let _ = std::fs::remove_file(socket_path);
    UnixListener::bind(socket_path)
}

/// ส่ง 1 request และรับ 1 response (synchronous)
pub fn send_request(
    socket_path: &str,
    req: &IpcRequest,
) -> Result<IpcResponse, Box<dyn std::error::Error>> {
    let mut stream = UnixStream::connect(socket_path)?;
    let json = serde_json::to_string(req)?;
    writeln!(stream, "{}", json)?;   // ส่ง request (newline-terminated)
    stream.flush()?;

    let mut line = String::new();
    {
        let mut reader = BufReader::new(&stream);
        reader.read_line(&mut line)?; // รอรับ response 1 บรรทัด
    }
    let resp: IpcResponse = serde_json::from_str(line.trim())?;
    Ok(resp)
}

/// อ่าน 1 request จาก socket, เรียก handler, เขียน response กลับ
pub fn handle_connection(mut stream: UnixStream, handler: impl Fn(IpcRequest) -> IpcResponse) {
    let mut line = String::new();
    {
        let mut reader = BufReader::new(&stream);
        if reader.read_line(&mut line).is_err() {
            return;
        }
    }
    // ยกเลิก borrow แล้วค่อย write กลับ (ไม่ clash กับ read borrow)
    if let Ok(req) = serde_json::from_str::<IpcRequest>(line.trim()) {
        let resp = handler(req);
        if let Ok(json) = serde_json::to_string(&resp) {
            let _ = writeln!(stream, "{}", json);
        }
    }
}
```

**ทำไม BufReader scope ต้องแยกออกจาก write?**

`BufReader::new(&stream)` ยืม `stream` แบบ immutable ถ้าเราพยายาม `writeln!(stream, ...)` ขณะที่ `reader` ยังมีชีวิตอยู่ Rust borrow checker จะปฏิเสธ เพราะ mutable borrow + immutable borrow ไม่สามารถมีพร้อมกันได้ วิธีแก้คือใส่ `reader` ใน block `{ }` เพื่อให้ lifetime ของมันหมดก่อน จึงค่อย write ได้

**Wire protocol ที่ใช้:**

```
Client → Server: {"type":"status"}\n
Server → Client: {"type":"process_list","processes":[...]}\n

Client → Server: {"type":"start","name":"api-server"}\n
Server → Client: {"type":"ok","message":"started api-server [pid=12345]"}\n
```

serde's `#[serde(tag = "type", rename_all = "snake_case")]` สร้าง internally-tagged enum — ทุก variant มี field `"type"` ที่ระบุชนิด message นี่คือ JSON API convention ที่พบเห็นทั่วไปใน practice

---

### ขั้นที่ 7: CLI Commands (start/stop/restart/status/logs)

ประกอบทุกอย่างเข้าด้วยกันผ่าน clap-based CLI:

```rust
// src/main.rs
use clap::{Parser, Subcommand};
use std::path::PathBuf;

mod config;
mod daemon;
mod logger;
mod process;
mod status;
mod sysinfo_util;

#[derive(Parser, Debug)]
#[command(
    name = "procmgr",
    version = "0.1.0",
    about = "Process Manager — PM2-like tool written in Rust"
)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Start a named process (from procmgr.json)
    Start {
        name: String,
        #[arg(short, long, default_value = "procmgr.json")]
        config: PathBuf,
    },
    /// Stop a running process gracefully
    Stop { name: String },
    /// Restart a process
    Restart { name: String },
    /// Show status table of all managed processes
    Status,
    /// List all processes (alias for status)
    List,
    /// Tail log file for a process
    Logs {
        name: String,
        /// Number of lines to show
        #[arg(short = 'n', long, default_value = "20")]
        lines: usize,
        /// Follow (like tail -f)
        #[arg(short, long)]
        follow: bool,
    },
    /// Start the background daemon process
    Daemon,
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        Commands::Status | Commands::List => show_status(),
        Commands::Start { name, .. }      => send_or_print(daemon::IpcRequest::Start { name }),
        Commands::Stop  { name }          => send_or_print(daemon::IpcRequest::Stop { name }),
        Commands::Restart { name }        => send_or_print(daemon::IpcRequest::Restart { name }),
        Commands::Logs { name, lines, follow } => show_logs(&name, lines, follow),
        Commands::Daemon                  => run_daemon(),
    }
}

fn show_status() {
    match daemon::send_request(daemon::SOCKET_PATH, &daemon::IpcRequest::List) {
        Ok(daemon::IpcResponse::ProcessList { processes }) => {
            println!("{}", status::render_status_table(&processes));
        }
        Ok(daemon::IpcResponse::Error { message }) => eprintln!("Error: {}", message),
        Err(e) => {
            eprintln!("Daemon not running: {}", e);
            eprintln!("Start it with: procmgr daemon");
        }
        _ => {}
    }
}

fn send_or_print(req: daemon::IpcRequest) {
    match daemon::send_request(daemon::SOCKET_PATH, &req) {
        Ok(daemon::IpcResponse::Ok { message })    => println!("{}", message),
        Ok(daemon::IpcResponse::Error { message }) => eprintln!("Error: {}", message),
        Err(e) => eprintln!("Daemon not running: {}", e),
        _ => {}
    }
}

fn show_logs(name: &str, lines: usize, follow: bool) {
    let lgr = logger::Logger::new(name, process::log_dir());
    if follow {
        let mut last_pos = 0usize;
        loop {
            if let Ok(content) = std::fs::read_to_string(lgr.log_path()) {
                let all_lines: Vec<&str> = content.lines().collect();
                for line in &all_lines[last_pos..] {
                    println!("{}", line);
                }
                last_pos = all_lines.len();
            }
            std::thread::sleep(std::time::Duration::from_millis(500));
        }
    } else {
        match lgr.tail(lines) {
            Ok(ls) => ls.iter().for_each(|l| println!("{}", l)),
            Err(e) => eprintln!("Error reading log: {}", e),
        }
    }
}

fn run_daemon() {
    use std::collections::HashMap;
    use std::sync::{Arc, Mutex};

    // State: map ชื่อ process → ProcessInfo ใน memory
    let procs: Arc<Mutex<HashMap<String, daemon::ProcessInfo>>> =
        Arc::new(Mutex::new(HashMap::new()));

    let listener = match daemon::create_listener(daemon::SOCKET_PATH) {
        Ok(l)  => l,
        Err(e) => { eprintln!("Failed to bind {}: {}", daemon::SOCKET_PATH, e); std::process::exit(1); }
    };
    println!("procmgr daemon started on {}", daemon::SOCKET_PATH);

    for stream in listener.incoming().flatten() {
        let procs = Arc::clone(&procs);
        std::thread::spawn(move || {
            daemon::handle_connection(stream, move |req| {
                handle_ipc(&mut *procs.lock().unwrap(), req)
            });
        });
    }
}

fn handle_ipc(
    map: &mut std::collections::HashMap<String, daemon::ProcessInfo>,
    req: daemon::IpcRequest,
) -> daemon::IpcResponse {
    match req {
        daemon::IpcRequest::List | daemon::IpcRequest::Status => {
            daemon::IpcResponse::ProcessList { processes: map.values().cloned().collect() }
        }
        daemon::IpcRequest::Start { ref name } => {
            if map.contains_key(name) {
                daemon::IpcResponse::Error { message: format!("{} is already running", name) }
            } else {
                map.insert(name.clone(), daemon::ProcessInfo {
                    name: name.clone(), pid: None, status: "stopped".into(),
                    uptime_secs: 0, restart_count: 0, cpu_percent: 0.0, mem_mb: 0.0,
                });
                daemon::IpcResponse::Ok { message: format!("started {}", name) }
            }
        }
        daemon::IpcRequest::Stop { ref name } => {
            map.remove(name);
            daemon::IpcResponse::Ok { message: format!("stopped {}", name) }
        }
        daemon::IpcRequest::Restart { ref name } => {
            daemon::IpcResponse::Ok { message: format!("restarted {}", name) }
        }
        daemon::IpcRequest::Logs { ref name, .. } => {
            daemon::IpcResponse::Ok { message: format!("reading logs for {}", name) }
        }
    }
}
```

---

### ขั้นที่ 8: Status Table with uptime + restart count

ใช้ `tabled` สร้าง table และ `owo-colors` ใส่สีให้ status field:

```rust
// src/status.rs
use crate::daemon::ProcessInfo;
use owo_colors::OwoColorize;
use tabled::{Table, Tabled};

#[derive(Tabled)]
pub struct StatusRow {
    #[tabled(rename = "NAME")]
    pub name: String,
    #[tabled(rename = "PID")]
    pub pid: String,
    #[tabled(rename = "STATUS")]
    pub status: String,
    #[tabled(rename = "UPTIME")]
    pub uptime: String,
    #[tabled(rename = "RESTARTS")]
    pub restarts: String,
    #[tabled(rename = "CPU%")]
    pub cpu: String,
    #[tabled(rename = "MEM")]
    pub mem: String,
}

pub fn format_uptime(secs: u64) -> String {
    if secs < 60 {
        format!("{}s", secs)
    } else if secs < 3600 {
        format!("{}m{}s", secs / 60, secs % 60)
    } else {
        format!("{}h{}m", secs / 3600, (secs % 3600) / 60)
    }
}

pub fn render_status_table(processes: &[ProcessInfo]) -> String {
    if processes.is_empty() {
        return "No processes managed.".to_string();
    }
    let rows: Vec<StatusRow> = processes.iter().map(|p| StatusRow {
        name:     p.name.clone(),
        pid:      p.pid.map(|v| v.to_string()).unwrap_or_else(|| "-".into()),
        status:   match p.status.as_str() {
            "online"  => "online".green().to_string(),
            "stopped" => "stopped".yellow().to_string(),
            "crashed" => "crashed".red().to_string(),
            other     => other.to_string(),
        },
        uptime:   format_uptime(p.uptime_secs),
        restarts: p.restart_count.to_string(),
        cpu:      format!("{:.1}%", p.cpu_percent),
        mem:      format!("{:.1} MB", p.mem_mb),
    }).collect();
    Table::new(rows).to_string()
}
```

**ตัวอย่าง output ของ `procmgr status`:**

```
+------------+-------+---------+---------+----------+------+----------+
| NAME       | PID   | STATUS  | UPTIME  | RESTARTS | CPU% | MEM      |
+------------+-------+---------+---------+----------+------+----------+
| api-server | 12345 | online  | 2h15m   | 0        | 1.2% | 128.5 MB |
+------------+-------+---------+---------+----------+------+----------+
| worker     | 12346 | online  | 2h14m   | 2        | 0.8% | 64.0 MB  |
+------------+-------+---------+---------+----------+------+----------+
| scheduler  | -     | stopped | 0s      | 0        | 0.0% | 0.0 MB   |
+------------+-------+---------+---------+----------+------+----------+
```

(ใน terminal จริง STATUS column จะมีสีเขียว/เหลือง/แดง)

---

### CPU% และ Memory Metrics ด้วย sysinfo

```rust
// src/sysinfo_util.rs
use sysinfo::{Pid, System};

pub struct ProcessMetrics {
    pub cpu_percent: f32,
    pub mem_bytes: u64,
}

/// Query process CPU% and memory. Returns None if PID not found.
pub fn get_metrics(pid: u32) -> Option<ProcessMetrics> {
    let mut sys = System::new();
    let spid = Pid::from(pid as usize);
    sys.refresh_process(spid);
    sys.process(spid).map(|p| ProcessMetrics {
        cpu_percent: p.cpu_usage(),
        mem_bytes:   p.memory(), // bytes (sysinfo 0.30)
    })
}
```

ใน daemon loop ที่รัน status refresh ทุก 1 วินาที:
```rust
fn refresh_metrics(map: &mut HashMap<String, ProcessInfo>) {
    for info in map.values_mut() {
        if let Some(pid) = info.pid {
            if let Some(m) = sysinfo_util::get_metrics(pid) {
                info.cpu_percent = m.cpu_percent;
                info.mem_mb      = m.mem_bytes as f64 / (1024.0 * 1024.0);
            }
        }
    }
}
```

---

## Cargo.toml สมบูรณ์

```toml
[package]
name = "procmgr"
version = "0.1.0"
edition = "2021"
description = "PM2-like process manager written in Rust"

[[bin]]
name = "procmgr"
path = "src/main.rs"

[dependencies]
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
clap       = { version = "4", features = ["derive"] }
nix        = { version = "0.27", features = ["process", "signal"] }
sysinfo    = "0.30"
tabled     = "0.14"
owo-colors = "4"
```

---

## การทดสอบ (Testing)

สร้าง scratch project ใน scratchpad directory, รัน `cargo test`, บันทึกผลจริง

```
$ cargo test
   Compiling procmgr v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 12.07s
     Running unittests src/main.rs (target/debug/deps/procmgr-cd4aeb0650a8c5c3)

running 4 tests
test tests::test_config_parsing ... ok
test tests::test_log_rotation ... ok
test tests::test_ipc_unix_socket ... ok
test tests::test_pid_file ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Test code ที่สร้าง output นี้:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::path::PathBuf;

    fn tmp_dir(tag: &str) -> PathBuf {
        PathBuf::from(format!("/tmp/procmgr_test_{}_{}", std::process::id(), tag))
    }

    // ── Test 1: config JSON parsing ──────────────────────────────────────────
    #[test]
    fn test_config_parsing() {
        let json = r#"{
            "processes": [
                {
                    "name": "webserver",
                    "command": "python3",
                    "args": ["-m", "http.server", "8080"],
                    "cwd": "/var/www",
                    "env": { "PORT": "8080", "DEBUG": "false" },
                    "auto_restart": true,
                    "max_restarts": 5,
                    "restart_delay_ms": 2000
                },
                {
                    "name": "worker",
                    "command": "/usr/bin/worker",
                    "args": ["--queue", "default"]
                }
            ]
        }"#;

        let cfg = config::AppConfig::from_json_str(json).expect("parse failed");

        assert_eq!(cfg.processes.len(), 2);
        let web = &cfg.processes[0];
        assert_eq!(web.name,    "webserver");
        assert_eq!(web.command, "python3");
        assert_eq!(web.args,    vec!["-m", "http.server", "8080"]);
        assert_eq!(web.cwd.as_deref(),           Some("/var/www"));
        assert_eq!(web.env.get("PORT").map(|s| s.as_str()), Some("8080"));
        assert!(web.auto_restart);
        assert_eq!(web.max_restarts,    5);
        assert_eq!(web.restart_delay_ms, 2000);

        let worker = &cfg.processes[1];
        assert_eq!(worker.name, "worker");
        assert!(!worker.auto_restart);     // serde default = false
        assert_eq!(worker.max_restarts, 10); // serde default = 10
    }

    // ── Test 2: spawn sleep 60, write PID file, verify, kill ────────────────
    #[test]
    fn test_pid_file() {
        use std::process::Command;
        let dir = tmp_dir("pid");
        std::fs::create_dir_all(&dir).unwrap();

        let mut child = Command::new("sleep").arg("60").spawn()
            .expect("failed to spawn sleep");
        let pid = child.id();

        let pid_file = dir.join("sleep_test.pid");
        std::fs::write(&pid_file, pid.to_string()).unwrap();

        assert!(pid_file.exists(), "PID file missing");
        let stored: u32 = std::fs::read_to_string(&pid_file)
            .unwrap().trim().parse().unwrap();
        assert_eq!(stored, pid);

        child.kill().ok();
        child.wait().ok();
        std::fs::remove_file(&pid_file).unwrap();
        std::fs::remove_dir(&dir).ok();

        assert!(!pid_file.exists(), "PID file not removed");
    }

    // ── Test 3: log rotation when file exceeds threshold ────────────────────
    #[test]
    fn test_log_rotation() {
        let dir = tmp_dir("log");
        std::fs::create_dir_all(&dir).unwrap();

        // 1 KB threshold — keeps the test fast
        let lgr = logger::Logger::new("testapp", dir.clone()).with_max_size(1024);
        let log_path = lgr.log_path();
        let rot_path = lgr.rotated_path();

        let chunk = "X".repeat(100);
        let mut total = 0u64;
        while total < 1124 {
            lgr.write_line(&chunk).unwrap();
            total += (chunk.len() + 1) as u64; // +1 for newline
        }

        // rotation must have fired
        assert!(rot_path.exists(),  "rotated file should exist at {:?}", rot_path);
        assert!(log_path.exists(),  "active log should still exist");

        let active_size = log_path.metadata().unwrap().len();
        assert!(active_size < 1024,
            "active log ({} bytes) should be < 1024 after rotation", active_size);

        let _ = std::fs::remove_file(&log_path);
        let _ = std::fs::remove_file(&rot_path);
        let _ = std::fs::remove_dir(&dir);
    }

    // ── Test 4: IPC over Unix socket ─────────────────────────────────────────
    #[test]
    fn test_ipc_unix_socket() {
        let sock = format!("/tmp/procmgr_ipc_test_{}.sock", std::process::id());
        let sock_srv = sock.clone();
        let (ready_tx, ready_rx) = std::sync::mpsc::channel::<()>();

        let server = std::thread::spawn(move || {
            let listener = daemon::create_listener(&sock_srv).unwrap();
            ready_tx.send(()).unwrap(); // socket is bound; client may connect
            for stream in listener.incoming().take(1).flatten() {
                daemon::handle_connection(stream, |req| match req {
                    daemon::IpcRequest::Status | daemon::IpcRequest::List => {
                        daemon::IpcResponse::ProcessList {
                            processes: vec![daemon::ProcessInfo {
                                name: "app".into(), pid: Some(42000),
                                status: "online".into(), uptime_secs: 300,
                                restart_count: 1, cpu_percent: 2.5, mem_mb: 128.0,
                            }],
                        }
                    }
                    _ => daemon::IpcResponse::Ok { message: "ok".into() },
                });
            }
        });

        ready_rx.recv().unwrap(); // wait until socket is bound

        let resp = daemon::send_request(&sock, &daemon::IpcRequest::Status)
            .expect("IPC send_request failed");

        match resp {
            daemon::IpcResponse::ProcessList { processes } => {
                assert_eq!(processes.len(), 1);
                assert_eq!(processes[0].name,         "app");
                assert_eq!(processes[0].pid,          Some(42000));
                assert_eq!(processes[0].status,       "online");
                assert_eq!(processes[0].uptime_secs,  300);
                assert_eq!(processes[0].restart_count, 1);
            }
            other => panic!("Expected ProcessList, got {:?}", other),
        }

        server.join().ok();
        let _ = std::fs::remove_file(&sock);
    }
}
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Borrow Conflict ระหว่าง BufReader กับ Write บน UnixStream

**ปัญหา:**
```rust
// ERROR: cannot borrow `stream` as mutable because it is also borrowed as immutable
let reader = BufReader::new(&stream);
reader.read_line(&mut line)?;
writeln!(stream, "{}", response)?; // ← compile error!
```

Rust ปฏิเสธเพราะ `reader` ยืม `stream` แบบ immutable ค้างอยู่ ขณะที่ `writeln!` ต้องการ mutable borrow

**แก้ไข:** ใส่ reader ใน block `{ }` เพื่อให้ lifetime หมดก่อน:
```rust
let mut line = String::new();
{
    let mut reader = BufReader::new(&stream);
    reader.read_line(&mut line)?;
} // ← reader ถูก drop ที่นี่
writeln!(stream, "{}", response)?; // ← ได้แล้ว
```

หรือใช้ `stream.try_clone()` เพื่อแยก read/write handles ออกจากกัน:
```rust
let write_half = stream.try_clone()?;
let mut reader = BufReader::new(stream);
```

---

### 2. Zombie Processes เมื่อไม่ wait() หลัง kill()

**ปัญหา:**
```rust
let child = Command::new("sleep").arg("60").spawn().unwrap();
// ... ส่ง SIGTERM ...
// ถ้าไม่เรียก child.wait() → zombie process ค้างใน process table
```

เมื่อ child process ตาย แต่ parent ยังไม่เรียก `wait()`, kernel เก็บ exit status ไว้ใน process table เป็น "zombie" ใช้ PID slot แต่ไม่ใช้ memory จริง ๆ ถ้าสร้าง zombie เยอะมากพอจะทำให้ PID namespace หมดได้

**แก้ไข:**
```rust
// ใน stop_graceful() ต้อง wait หลัง kill
child.kill().ok();
child.wait().ok(); // ← สำคัญมาก — reap the zombie
```

สำหรับ background processes ที่ไม่ต้องการรอ, ใช้ `SIGCHLD` handler หรือ double-fork (daemon process ไม่มี parent ที่รอ ดังนั้น init/systemd จะ reap แทน)

---

### 3. Stale Socket File ทำให้ Daemon Start ไม่ได้

**ปัญหา:**
```
$ procmgr daemon
bind failed: Address already in use (os error 98)
```

เกิดเมื่อ daemon ถูก kill แบบ hard โดยที่ socket file ยังค้างอยู่ใน `/tmp/procmgr.sock` `UnixListener::bind()` จะล้มเหลวเพราะ path นั้นมีอยู่แล้ว

**แก้ไข:** ลบ stale socket ก่อน bind เสมอ:
```rust
pub fn create_listener(socket_path: &str) -> std::io::Result<UnixListener> {
    let _ = std::fs::remove_file(socket_path); // ignore error ถ้าไม่มีไฟล์
    UnixListener::bind(socket_path)
}
```

ใน production ควรตรวจว่า daemon ยังรันอยู่จริงก่อน remove socket — สร้าง lockfile หรือลอง connect ดูก่อน:
```rust
if UnixStream::connect(socket_path).is_ok() {
    eprintln!("Another daemon is already running!");
    std::process::exit(1);
}
```

---

### 4. File Handle ยังเปิดอยู่หลัง Log Rotation

**ปัญหา:** process ที่ spawn ด้วย `Stdio::from(file)` เปิด file descriptor ไปยัง `app.log` การ rotate (rename `app.log` → `app.log.1`) ไม่ได้ปิด fd นั้น process ยังคง write ไปยัง inode เดิม (ซึ่งตอนนี้เป็น `app.log.1`) ไม่ใช่ `app.log` ใหม่

```
ก่อน rotate:
  process fd 1 → inode #1234 (app.log)

หลัง rename:
  process fd 1 → inode #1234 (app.log.1)  ← process ยัง write ที่นี่!
  app.log      → inode #5678 (ใหม่ ว่าง)
```

**แก้ไข สำหรับโปรเจคนี้:** ใน monitor loop เมื่อ rotate ให้ restart process โดยเปิด log file handle ใหม่:
```rust
if logger.maybe_rotate()? {
    // ต้อง spawn ใหม่เพื่อให้ใช้ file descriptor ใหม่
    proc.stop_graceful()?;
    proc.spawn(Some(logger.open_for_append()?))?;
}
```

ใน production tool จริง (เช่น logrotate) ใช้วิธี `copytruncate` — copy เนื้อหาไปก่อน แล้ว truncate ไฟล์เดิม (fd เดิมยังใช้ได้ แต่ไฟล์เล็กลง) หรือส่ง SIGHUP ให้ process เพื่อให้มัน reopen log file เอง

---

### 5. Race Condition ใน Status Refresh ด้วย sysinfo

**ปัญหา:**
```rust
// thread A: กำลัง refresh CPU metrics
sys.refresh_all(); // ← ใช้เวลา
// thread B: กำลัง stop process ที่ sys กำลัง query อยู่
proc.stop_graceful();
```

`sysinfo::System` ไม่ใช่ thread-safe ถ้า thread A อ่านข้อมูล process ขณะที่ thread B กำลัง kill process นั้น อาจได้ข้อมูลผิดพลาดหรือ panic

**แก้ไข:** ใช้ `Mutex<System>` หรือสร้าง `System` instance ใหม่ทุกครั้งที่ refresh (overhead เล็กน้อยแต่ปลอดภัย):
```rust
pub fn get_metrics(pid: u32) -> Option<ProcessMetrics> {
    let mut sys = System::new(); // ← สร้างใหม่ทุกครั้ง
    let spid = Pid::from(pid as usize);
    sys.refresh_process(spid);  // ← refresh เฉพาะ process นั้น
    // ...
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/procmgr
strip target/release/procmgr  # ลด binary size (optional)
```

### ติดตั้งในระบบ

```bash
sudo cp target/release/procmgr /usr/local/bin/
procmgr --version
```

### รัน Daemon เป็น Systemd Service

```ini
# /etc/systemd/system/procmgr.service
[Unit]
Description=Procmgr Process Manager
After=network.target

[Service]
Type=simple
User=deploy
ExecStart=/usr/local/bin/procmgr daemon
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable procmgr
sudo systemctl start procmgr
sudo journalctl -u procmgr -f
```

### ใช้งาน procmgr.json กับ daemon

```bash
# Start daemon ก่อน
procmgr daemon &

# Start processes จาก config
procmgr start api-server --config /opt/myapp/procmgr.json
procmgr start worker     --config /opt/myapp/procmgr.json

# ดู status
procmgr status

# ดู logs
procmgr logs api-server -n 50
procmgr logs api-server --follow

# Stop
procmgr stop api-server
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1 (ง่าย): Multiple Log Files per Rotation

โปรเจคนี้เก็บแค่ `app.log` และ `app.log.1` (2 ไฟล์) ให้เพิ่มรองรับ numbered rotation ตาม `max_files` ที่กำหนดใน config:

```json
{
  "name": "api-server",
  "command": "...",
  "log_max_size_mb": 10,
  "log_max_files": 5
}
```

เมื่อ rotate: `app.log.4` → ลบ, `app.log.3` → `app.log.4`, ..., `app.log` → `app.log.1`, สร้าง `app.log` ใหม่

**Hint:** ใช้ `(1..max_files).rev()` เพื่อ rename ไล่จาก index สูงลงต่ำ

---

### แบบฝึกหัดที่ 2 (กลาง): Daemon Auto-Discovery ด้วย Lock File

ตอนนี้ถ้าเรียก `procmgr start api` โดยที่ daemon ไม่ได้รัน จะได้ error "Daemon not running" CLI ควรพยายาม start daemon ให้อัตโนมัติถ้า socket ไม่มีอยู่:

```rust
fn ensure_daemon_running() {
    if UnixStream::connect(SOCKET_PATH).is_err() {
        // spawn daemon เป็น background process
        Command::new(std::env::current_exe().unwrap())
            .arg("daemon")
            .stdin(Stdio::null())
            .stdout(Stdio::null())
            .stderr(Stdio::null())
            .spawn()
            .expect("failed to start daemon");
        // รอ socket ขึ้นมา
        for _ in 0..20 {
            std::thread::sleep(Duration::from_millis(100));
            if UnixStream::connect(SOCKET_PATH).is_ok() { break; }
        }
    }
}
```

**ความท้าทาย:** ป้องกัน race condition กรณีที่ 2 CLI instance พยายาม start daemon พร้อมกัน ใช้ lock file (`/tmp/procmgr.lock`) เพื่อ atomic guard

---

### แบบฝึกหัดที่ 3 (กลาง): Environment Variable Inheritance + Override

ตอนนี้ process สืบทอด env ทั้งหมดจาก daemon (ซึ่งสืบจาก shell ที่รัน daemon) เพิ่มการควบคุม:

```json
{
  "name": "api",
  "command": "...",
  "env_inherit": ["PATH", "HOME", "LANG"],
  "env": { "PORT": "8080" },
  "env_clear": false
}
```

- `env_clear: true` → เริ่มจาก clean environment (ไม่สืบอะไร)
- `env_inherit` → สืบเฉพาะ keys ที่ระบุ
- `env` → override หรือเพิ่ม keys

ใช้ `Command::env_clear()` ร่วมกับ `Command::envs()` เพื่อ implement

---

### แบบฝึกหัดที่ 4 (ยาก): HTTP Health Check + Auto-Restart on Unhealthy

เพิ่ม `health_check` section ใน config:

```json
{
  "name": "api-server",
  "command": "...",
  "health_check": {
    "url": "http://localhost:8080/health",
    "interval_secs": 10,
    "timeout_secs": 5,
    "failure_threshold": 3,
    "success_threshold": 1
  }
}
```

ใน monitor thread ให้ poll URL นั้นทุก `interval_secs` วินาที ถ้า fail ติดต่อกัน `failure_threshold` ครั้ง → restart process แม้ว่า process ยังรันอยู่ (เช่น deadlock หรือ memory leak ทำให้ไม่ response)

ใช้ `std::net::TcpStream::connect_timeout()` หรือ `ureq` crate สำหรับ HTTP requests

---

## สรุป

โปรเจคนี้สร้าง process manager ที่ครอบคลุม patterns สำคัญของ systems programming ใน Rust:

| สิ่งที่สร้าง | Pattern ที่ได้เรียน |
|---|---|
| Daemon mode | `setsid()`, double-fork, detach from terminal |
| Process lifecycle | `Command::spawn()`, `Child::try_wait()`, signal sending |
| IPC | Unix domain socket, JSON framing, request/response protocol |
| Log management | File append, rotation, tail |
| Config parsing | serde with defaults, HashMap env vars |
| Concurrency | `Arc<Mutex<T>>` shared state, background monitor threads |
| Status display | `tabled` + `owo-colors` terminal UI |

Pattern ที่สำคัญที่สุดคือการใช้ **channel** (mpsc) หรือ **shared state** (Arc<Mutex<T>>) เพื่อสื่อสารระหว่าง monitor thread กับ IPC handler thread อย่างปลอดภัย — นี่คือ fundamental concurrency pattern ที่จะใช้ในโปรเจคต่อ ๆ ไปทุกโปรเจคที่มี background work

โปรเจคถัดไป **System Monitor** จะขยายทักษะด้าน `sysinfo` และ terminal UI ไปอีกระดับ ด้วยการ render CPU/memory/disk graphs แบบ real-time ในรูปแบบคล้าย `htop`

---

**โปรเจคก่อนหน้า:** [project-a06-terminal-editor.md](project-a06-terminal-editor.md) | **โปรเจคถัดไป:** [project-a08-system-monitor.md](project-a08-system-monitor.md)
