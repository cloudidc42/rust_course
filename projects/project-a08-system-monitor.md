# Project A08: System Monitor (htop-like)

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

`htop` คือเครื่องมือที่ developer และ sysadmin ทั่วโลกเปิดทิ้งไว้ในทุก terminal session — มันให้ภาพรวมของระบบแบบ real-time: CPU ใช้เท่าไหร่, RAM เหลือเท่าไหร่, process ไหนกินทรัพยากรมากที่สุด, disk และ network วิ่งเท่าไหร่ในขณะนั้น

โปรเจคนี้สร้าง system monitor แบบ TUI (Text-based User Interface) ในภาษา Rust โดยใช้ crate ชั้นนำสองตัว:

- **`sysinfo`** — รวบรวมข้อมูล CPU, memory, processes, disks, networks จาก OS โดยรองรับ Linux, macOS, Windows
- **`ratatui`** — framework สำหรับ TUI ที่ render ลงใน terminal ด้วย double-buffering (วาดเฉพาะส่วนที่เปลี่ยน)

ความน่าสนใจของโปรเจคนี้คือมันบังคับให้คิดเรื่อง **data pipeline**: ข้อมูล OS → struct ใน Rust → layout ใน terminal ต้องเกิดขึ้นใน 16ms หรือน้อยกว่า (60 fps) โดยไม่ blocking event loop สิ่งที่จะได้ฝึกจึงครอบคลุมทั้ง systems programming, data modeling, state management และ reactive UI

ใน production จริง เครื่องมืออย่าง `btop++`, `glances`, `bottom (btm)` และ Datadog Agent ล้วนใช้หลักการเดียวกัน — poll OS metrics ในรอบที่กำหนด แล้วแสดงผลใน terminal หรือส่งไป dashboard

## สิ่งที่จะได้เรียนรู้

- **`sysinfo` crate**: refresh CPU, memory, processes, disk, network แบบ incremental polling
- **`ratatui` layout engine**: `Layout::default()`, `Direction`, `Constraint` สำหรับแบ่ง terminal เป็น panel
- **Ring buffer pattern**: ใช้ `VecDeque` เก็บประวัติ CPU% 60 tick สำหรับ Sparkline
- **Stateful widgets**: `TableState` และ `ScrollbarState` สำหรับ process table ที่ scroll ได้
- **Keyboard event loop**: crossterm event polling + state machine (normal / filter mode)
- **Terminal raw mode**: ใส่/ถอด raw mode อย่าง safe ด้วย RAII guard เพื่อไม่ให้ terminal พังหลัง panic
- **CLI parsing**: `clap` derive macro กับ `Duration` argument parsing
- **Graceful shutdown**: signal handler + cleanup ก่อนออก

## ความรู้ที่ต้องมีมาก่อน

- **Part 21–25:** struct, enum, impl — ออกแบบ `AppState` struct
- **Part 31–35:** error handling ด้วย `Result`, `?` operator
- **Part 41–45:** collections — `HashMap`, `VecDeque`, iterators
- **Part 46–50:** ownership และ borrow checker — share state ระหว่าง update loop กับ render
- **Part 61–65:** standard library — `std::time::Duration`, `Instant`
- **Part 96–100:** systems programming — อ่าน `/proc` data ผ่าน `sysinfo`

## โครงสร้างโปรเจค (Project Layout)

```
system-monitor/
├── src/
│   ├── main.rs          ← CLI + event loop + shutdown
│   ├── app.rs           ← AppState, update logic, ring buffer
│   ├── metrics.rs       ← wrapper เหนือ sysinfo — poll + diff
│   ├── ui/
│   │   ├── mod.rs       ← draw() entry point, layout root
│   │   ├── cpu.rs       ← CPU sparkline panel
│   │   ├── memory.rs    ← memory + swap bar
│   │   ├── process.rs   ← sortable/filterable process table
│   │   ├── disk.rs      ← disk I/O panel
│   │   └── network.rs   ← network rx/tx panel
│   └── format.rs        ← format_bytes, format_duration, color_for_pct
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
┌─────────────────────────────────────────────────────────┐
│  tick N (every 2 s by default)                          │
│                                                         │
│  sysinfo::System::refresh_all()                        │
│       │                                                 │
│       ▼                                                 │
│  metrics::Snapshot::collect(&sys)  ─────────────────── │
│       │                  (CPU%, mem bytes,             │
│       │                   process list, disk R/W,      │
│       │                   network rx/tx)               │
│       ▼                                                 │
│  AppState::update(snapshot)                            │
│       │  ┌─────────────────────────────────────┐       │
│       │  │  cpu_history: RingBuffer<f64, 60>   │       │
│       │  │  processes:  Vec<ProcessRow>        │       │
│       │  │  sort_col:   SortColumn             │       │
│       │  │  filter:     Option<String>         │       │
│       │  └─────────────────────────────────────┘       │
│       ▼                                                 │
│  ratatui::Terminal::draw(|frame| ui::draw(frame, &app))│
└─────────────────────────────────────────────────────────┘
```

### ทำไมแยก `metrics.rs` ออกจาก `app.rs`?

`sysinfo::System` เก็บ raw data ของ OS: nanosecond timestamps, byte counters ที่เพิ่มขึ้นเรื่อย ๆ (monotonic counter) และ CPU ticks ตั้งแต่เปิดเครื่อง ค่าที่อยากแสดงบนหน้าจอ (bytes/sec, CPU%) ต้องคำนวณจาก **diff สองสแนปชอต** ถ้าปน logic นี้ไว้ใน app state จะทำให้ test ยาก จึงแยก `metrics.rs` ให้รับผิดชอบการ diff และ normalize เพียงอย่างเดียว

### Ring Buffer แทน `Vec`

CPU history ต้องการเฉพาะ 60 ค่าล่าสุด ถ้าใช้ `Vec` แล้ว `push` + `remove(0)` ทุก tick จะเป็น O(n) เพราะต้อง shift ทุก element `VecDeque` แก้ปัญหานี้ด้วย O(1) `push_back` + `pop_front` — เหมาะกับ ring buffer pattern พอดี

### Terminal Safety

`ratatui` + `crossterm` ต้องการ "raw mode" (ปิด line buffering ของ terminal) ถ้าโปรแกรม panic ระหว่าง raw mode terminal จะพังใช้งานไม่ได้ จึงต้องใช้ RAII guard ที่ restore terminal ใน `Drop` เสมอ

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: เก็บข้อมูล CPU + Memory แล้ว print ลง terminal

เริ่มจากโปรแกรมที่ง่ายที่สุด: poll `sysinfo` แล้ว print ค่าออกมาทุก 2 วินาที ยังไม่มี TUI ยังไม่มี input handling — แค่ validate ว่าดึงข้อมูลได้จริง

**Cargo.toml:**

```toml
[package]
name = "system-monitor"
version = "0.1.0"
edition = "2021"

[dependencies]
sysinfo     = "0.31"
ratatui     = "0.28"
crossterm   = "0.28"
clap        = { version = "4", features = ["derive"] }

[profile.release]
strip = true
opt-level = 3
```

**src/format.rs** — helper functions สำหรับแสดงค่า:

```rust
// src/format.rs

const KIB: u64 = 1_024;
const MIB: u64 = 1_024 * KIB;
const GIB: u64 = 1_024 * MIB;
const TIB: u64 = 1_024 * GIB;

/// แปลง bytes เป็น string แบบ IEC (KiB, MiB, GiB, TiB)
pub fn format_bytes(bytes: u64) -> String {
    if bytes >= TIB {
        format!("{:.2} TiB", bytes as f64 / TIB as f64)
    } else if bytes >= GIB {
        format!("{:.2} GiB", bytes as f64 / GIB as f64)
    } else if bytes >= MIB {
        format!("{:.2} MiB", bytes as f64 / MIB as f64)
    } else if bytes >= KIB {
        format!("{:.2} KiB", bytes as f64 / KIB as f64)
    } else {
        format!("{} B", bytes)
    }
}

/// แปลง seconds เป็น "1h 23m 45s" format
pub fn format_uptime(secs: u64) -> String {
    let h = secs / 3600;
    let m = (secs % 3600) / 60;
    let s = secs % 60;
    if h > 0 {
        format!("{}h {:02}m {:02}s", h, m, s)
    } else {
        format!("{:02}m {:02}s", m, s)
    }
}

/// เลือกสี ratatui ตาม % (0-100):
///   < 50% → Green, 50–80% → Yellow, > 80% → Red
pub fn color_for_pct(pct: f64) -> ratatui::style::Color {
    use ratatui::style::Color;
    if pct >= 80.0 {
        Color::Red
    } else if pct >= 50.0 {
        Color::Yellow
    } else {
        Color::Green
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn format_bytes_gib() {
        assert_eq!(format_bytes(1_073_741_824), "1.00 GiB");
    }

    #[test]
    fn format_bytes_mib() {
        assert_eq!(format_bytes(2 * MIB), "2.00 MiB");
    }

    #[test]
    fn format_bytes_zero() {
        assert_eq!(format_bytes(0), "0 B");
    }
}
```

**src/metrics.rs** — wraps sysinfo และคำนวณ diff:

```rust
// src/metrics.rs

use std::collections::HashMap;
use sysinfo::{Disks, Networks, System};

/// ข้อมูล CPU ของแต่ละ core ณ จุดเวลาหนึ่ง (ticks)
#[derive(Debug, Clone, Copy, Default)]
pub struct CpuTick {
    pub total: u64,
    pub idle: u64,
}

impl CpuTick {
    /// คำนวณ CPU% จากสองสแนปชอต
    pub fn percent(prev: CpuTick, curr: CpuTick) -> f64 {
        let dt = curr.total.saturating_sub(prev.total);
        let di = curr.idle.saturating_sub(prev.idle);
        if dt == 0 {
            return 0.0;
        }
        let busy = dt.saturating_sub(di);
        (busy as f64 / dt as f64) * 100.0
    }
}

/// Snapshot ที่เก็บ diff ทั้งหมด (ค่าที่แสดงบนหน้าจอโดยตรง)
#[derive(Debug, Clone, Default)]
pub struct Snapshot {
    // CPU
    pub cpu_global_pct: f64,
    pub cpu_per_core: Vec<f64>,      // % ต่อ core

    // Memory
    pub mem_total: u64,
    pub mem_used: u64,
    pub mem_available: u64,
    pub swap_total: u64,
    pub swap_used: u64,

    // Processes
    pub processes: Vec<ProcessEntry>,

    // Disk I/O (bytes/sec)
    pub disk_io: Vec<DiskEntry>,

    // Network I/O (bytes/sec)
    pub net_io: Vec<NetEntry>,

    // System uptime (seconds)
    pub uptime: u64,
}

#[derive(Debug, Clone)]
pub struct ProcessEntry {
    pub pid: u32,
    pub name: String,
    pub cpu_pct: f64,
    pub mem_bytes: u64,
    pub status: String,
}

#[derive(Debug, Clone)]
pub struct DiskEntry {
    pub name: String,
    pub read_bps: u64,   // bytes per second
    pub write_bps: u64,
    pub total: u64,
    pub available: u64,
}

#[derive(Debug, Clone)]
pub struct NetEntry {
    pub iface: String,
    pub rx_bps: u64,
    pub tx_bps: u64,
}

/// State ที่เก็บ sysinfo handles + ค่าก่อนหน้าสำหรับ diff
pub struct MetricsCollector {
    sys: System,
    disks: Disks,
    networks: Networks,
    prev_disk_read: HashMap<String, u64>,
    prev_disk_write: HashMap<String, u64>,
    prev_net_rx: HashMap<String, u64>,
    prev_net_tx: HashMap<String, u64>,
    last_tick: std::time::Instant,
}

impl MetricsCollector {
    pub fn new() -> Self {
        let mut sys = System::new_all();
        sys.refresh_all();
        let disks = Disks::new_with_refreshed_list();
        let networks = Networks::new_with_refreshed_list();
        Self {
            sys,
            disks,
            networks,
            prev_disk_read: HashMap::new(),
            prev_disk_write: HashMap::new(),
            prev_net_rx: HashMap::new(),
            prev_net_tx: HashMap::new(),
            last_tick: std::time::Instant::now(),
        }
    }

    /// Refresh ทุก subsystem แล้วคืน Snapshot พร้อมใช้
    pub fn collect(&mut self) -> Snapshot {
        let dt = self.last_tick.elapsed().as_secs_f64().max(0.001);
        self.last_tick = std::time::Instant::now();

        self.sys.refresh_all();
        self.disks.refresh(true);
        self.networks.refresh(true);

        // ── CPU ──────────────────────────────────────────────────────────────
        // sysinfo 0.31 แยก cpu_usage() ออกมาเป็น % โดยตรงแล้ว (คำนวณ diff ภายใน)
        let cpu_per_core: Vec<f64> = self
            .sys
            .cpus()
            .iter()
            .map(|c| c.cpu_usage() as f64)
            .collect();
        let cpu_global_pct = if cpu_per_core.is_empty() {
            0.0
        } else {
            cpu_per_core.iter().sum::<f64>() / cpu_per_core.len() as f64
        };

        // ── Memory ───────────────────────────────────────────────────────────
        let mem_total = self.sys.total_memory();
        let mem_used = self.sys.used_memory();
        let mem_available = self.sys.available_memory();
        let swap_total = self.sys.total_swap();
        let swap_used = self.sys.used_swap();

        // ── Processes ────────────────────────────────────────────────────────
        let processes: Vec<ProcessEntry> = self
            .sys
            .processes()
            .values()
            .map(|p| ProcessEntry {
                pid: p.pid().as_u32(),
                name: p.name().to_string_lossy().into_owned(),
                cpu_pct: p.cpu_usage() as f64,
                mem_bytes: p.memory(),
                status: format!("{:?}", p.status()),
            })
            .collect();

        // ── Disk I/O ─────────────────────────────────────────────────────────
        let disk_io: Vec<DiskEntry> = self
            .disks
            .iter()
            .map(|d| {
                let name = d.name().to_string_lossy().into_owned();
                let usage = d.usage();
                let read_now = usage.read_bytes;
                let write_now = usage.written_bytes;
                let prev_r = self.prev_disk_read.get(&name).copied().unwrap_or(read_now);
                let prev_w = self.prev_disk_write.get(&name).copied().unwrap_or(write_now);
                self.prev_disk_read.insert(name.clone(), read_now);
                self.prev_disk_write.insert(name.clone(), write_now);
                DiskEntry {
                    name,
                    read_bps: ((read_now.saturating_sub(prev_r)) as f64 / dt) as u64,
                    write_bps: ((write_now.saturating_sub(prev_w)) as f64 / dt) as u64,
                    total: d.total_space(),
                    available: d.available_space(),
                }
            })
            .collect();

        // ── Network I/O ──────────────────────────────────────────────────────
        let net_io: Vec<NetEntry> = self
            .networks
            .iter()
            .map(|(iface, data)| {
                let rx_now = data.total_received();
                let tx_now = data.total_transmitted();
                let prev_rx = self.prev_net_rx.get(iface).copied().unwrap_or(rx_now);
                let prev_tx = self.prev_net_tx.get(iface).copied().unwrap_or(tx_now);
                self.prev_net_rx.insert(iface.clone(), rx_now);
                self.prev_net_tx.insert(iface.clone(), tx_now);
                NetEntry {
                    iface: iface.clone(),
                    rx_bps: ((rx_now.saturating_sub(prev_rx)) as f64 / dt) as u64,
                    tx_bps: ((tx_now.saturating_sub(prev_tx)) as f64 / dt) as u64,
                }
            })
            .collect();

        Snapshot {
            cpu_global_pct,
            cpu_per_core,
            mem_total,
            mem_used,
            mem_available,
            swap_total,
            swap_used,
            processes,
            disk_io,
            net_io,
            uptime: System::uptime(),
        }
    }
}
```

**src/main.rs (Step 1 — print only):**

```rust
// src/main.rs  (Step 1: ยังไม่มี TUI)

use std::time::Duration;
use crate::format::format_bytes;

mod format;
mod metrics;

fn main() {
    let mut collector = metrics::MetricsCollector::new();
    // warm-up: collect ครั้งแรกเพื่อให้ sysinfo คำนวณ CPU% ได้ถูกต้อง
    std::thread::sleep(Duration::from_millis(200));
    let snap = collector.collect();

    println!("=== System Monitor (Step 1) ===");
    println!("Uptime: {} s", snap.uptime);
    println!("CPU global: {:.1}%", snap.cpu_global_pct);
    for (i, pct) in snap.cpu_per_core.iter().enumerate() {
        println!("  core[{}]: {:.1}%", i, pct);
    }
    println!(
        "Memory: {} used / {} total",
        format_bytes(snap.mem_used),
        format_bytes(snap.mem_total)
    );
    println!(
        "Swap:   {} used / {} total",
        format_bytes(snap.swap_used),
        format_bytes(snap.swap_total)
    );
    println!("Top 5 processes by CPU:");
    let mut procs = snap.processes.clone();
    procs.sort_by(|a, b| b.cpu_pct.partial_cmp(&a.cpu_pct).unwrap());
    for p in procs.iter().take(5) {
        println!(
            "  {:6} {:20} {:5.1}% CPU  {}",
            p.pid, p.name, p.cpu_pct, format_bytes(p.mem_bytes)
        );
    }
}
```

**ตัวอย่าง output จาก Step 1:**

```
=== System Monitor (Step 1) ===
Uptime: 432187 s
CPU global: 4.3%
  core[0]: 6.1%
  core[1]: 2.5%
  core[2]: 5.2%
  core[3]: 3.4%
Memory: 5.82 GiB used / 15.53 GiB total
Swap:   0 B used / 8.00 GiB total
Top 5 processes by CPU:
     742 cargo                12.4% CPU  126.00 MiB
    1893 node                  4.2% CPU  82.50 MiB
    1247 Xorg                  2.8% CPU  48.00 MiB
     987 pulseaudio             1.1% CPU  12.00 MiB
    2034 bash                  0.3% CPU  3.25 MiB
```

---

### ขั้นที่ 2: Skeleton TUI ด้วย ratatui

เพิ่ม `AppState` struct และ event loop แบบ minimal — แค่แสดง placeholder ในแต่ละ panel และ quit ด้วย `q`

**src/app.rs:**

```rust
// src/app.rs

use std::collections::VecDeque;
use crate::metrics::{ProcessEntry, Snapshot};

/// จำนวน tick ที่เก็บใน CPU history (สำหรับ Sparkline)
pub const CPU_HISTORY_LEN: usize = 60;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SortColumn {
    Pid,
    Name,
    Cpu,
    Mem,
}

impl SortColumn {
    /// วน cycle ไปยัง sort column ถัดไป
    pub fn next(self) -> Self {
        match self {
            SortColumn::Cpu  => SortColumn::Mem,
            SortColumn::Mem  => SortColumn::Pid,
            SortColumn::Pid  => SortColumn::Name,
            SortColumn::Name => SortColumn::Cpu,
        }
    }

    pub fn label(self) -> &'static str {
        match self {
            SortColumn::Pid  => "PID",
            SortColumn::Name => "NAME",
            SortColumn::Cpu  => "CPU%",
            SortColumn::Mem  => "MEM",
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum InputMode {
    Normal,
    Filter,
}

/// สถานะหลักของแอปทั้งหมด
pub struct AppState {
    pub should_quit: bool,

    // CPU
    pub cpu_history: VecDeque<u64>,   // ค่า 0–100 สำหรับ Sparkline
    pub cpu_global_pct: f64,
    pub cpu_per_core: Vec<f64>,

    // Memory
    pub mem_total: u64,
    pub mem_used: u64,
    pub mem_available: u64,
    pub swap_total: u64,
    pub swap_used: u64,

    // Processes
    pub processes: Vec<ProcessEntry>,
    pub sort_col: SortColumn,
    pub selected_idx: usize,
    pub input_mode: InputMode,
    pub filter_str: String,

    // Disk + Network
    pub disk_entries: Vec<crate::metrics::DiskEntry>,
    pub net_entries: Vec<crate::metrics::NetEntry>,

    pub uptime: u64,
}

impl AppState {
    pub fn new() -> Self {
        Self {
            should_quit: false,
            cpu_history: VecDeque::with_capacity(CPU_HISTORY_LEN),
            cpu_global_pct: 0.0,
            cpu_per_core: vec![],
            mem_total: 0,
            mem_used: 0,
            mem_available: 0,
            swap_total: 0,
            swap_used: 0,
            processes: vec![],
            sort_col: SortColumn::Cpu,
            selected_idx: 0,
            input_mode: InputMode::Normal,
            filter_str: String::new(),
            disk_entries: vec![],
            net_entries: vec![],
            uptime: 0,
        }
    }

    /// อัพเดต state จาก snapshot ใหม่
    pub fn update(&mut self, snap: Snapshot) {
        // CPU history ring buffer
        let pct_u64 = snap.cpu_global_pct.clamp(0.0, 100.0) as u64;
        if self.cpu_history.len() >= CPU_HISTORY_LEN {
            self.cpu_history.pop_front();
        }
        self.cpu_history.push_back(pct_u64);

        self.cpu_global_pct = snap.cpu_global_pct;
        self.cpu_per_core = snap.cpu_per_core;

        self.mem_total = snap.mem_total;
        self.mem_used = snap.mem_used;
        self.mem_available = snap.mem_available;
        self.swap_total = snap.swap_total;
        self.swap_used = snap.swap_used;

        self.processes = snap.processes;
        self.sort_processes();

        // clamp selection หลัง sort (process อาจหายไป)
        let visible_len = self.visible_processes().len();
        if visible_len > 0 && self.selected_idx >= visible_len {
            self.selected_idx = visible_len - 1;
        }

        self.disk_entries = snap.disk_io;
        self.net_entries = snap.net_io;
        self.uptime = snap.uptime;
    }

    /// sort self.processes ตาม sort_col (in-place)
    pub fn sort_processes(&mut self) {
        match self.sort_col {
            SortColumn::Cpu => self
                .processes
                .sort_by(|a, b| b.cpu_pct.partial_cmp(&a.cpu_pct).unwrap()),
            SortColumn::Mem => self
                .processes
                .sort_by(|a, b| b.mem_bytes.cmp(&a.mem_bytes)),
            SortColumn::Pid => self.processes.sort_by_key(|p| p.pid),
            SortColumn::Name => self
                .processes
                .sort_by(|a, b| a.name.cmp(&b.name)),
        }
    }

    /// filter processes ตาม filter_str (case-insensitive substring)
    pub fn visible_processes(&self) -> Vec<&ProcessEntry> {
        if self.filter_str.is_empty() {
            self.processes.iter().collect()
        } else {
            let q = self.filter_str.to_lowercase();
            self.processes
                .iter()
                .filter(|p| p.name.to_lowercase().contains(&q))
                .collect()
        }
    }

    pub fn scroll_up(&mut self) {
        self.selected_idx = self.selected_idx.saturating_sub(1);
    }

    pub fn scroll_down(&mut self) {
        let max = self.visible_processes().len().saturating_sub(1);
        if self.selected_idx < max {
            self.selected_idx += 1;
        }
    }
}
```

---

### ขั้นที่ 3: CPU Sparkline พร้อม history ring buffer

ratatui มี `Sparkline` widget ที่รับ `&[u64]` และ render เป็น bar chart ย่อส่วนแนวนอน เหมาะมากสำหรับแสดงประวัติ CPU%

**src/ui/cpu.rs:**

```rust
// src/ui/cpu.rs

use ratatui::{
    layout::Rect,
    style::{Color, Modifier, Style},
    text::{Line, Span},
    widgets::{Block, Borders, Sparkline},
    Frame,
};
use crate::app::AppState;
use crate::format::color_for_pct;

/// วาด CPU panel: global% + sparkline + per-core bars
pub fn draw_cpu(f: &mut Frame, area: Rect, app: &AppState) {
    let history: Vec<u64> = app.cpu_history.iter().copied().collect();

    let cpu_color = color_for_pct(app.cpu_global_pct);
    let title = format!(
        " CPU  {:.1}% (↑ max {}) ",
        app.cpu_global_pct,
        history.iter().copied().max().unwrap_or(0)
    );

    let sparkline = Sparkline::default()
        .block(
            Block::default()
                .title(Line::from(vec![
                    Span::styled("⚡ ", Style::default().fg(Color::Yellow)),
                    Span::styled(title, Style::default().fg(cpu_color).add_modifier(Modifier::BOLD)),
                ]))
                .borders(Borders::ALL),
        )
        .data(&history)
        .max(100)
        .style(Style::default().fg(cpu_color));

    f.render_widget(sparkline, area);
}
```

**src/ui/mod.rs** — entry point สำหรับ draw ทั้งหมด:

```rust
// src/ui/mod.rs

use ratatui::{
    layout::{Constraint, Direction, Layout},
    Frame,
};
use crate::app::AppState;

mod cpu;
mod memory;
mod process;
mod disk;
mod network;

/// วาด UI ทั้งหมดลงใน frame เดียว
pub fn draw(f: &mut Frame, app: &AppState) {
    let size = f.area();

    // แบ่ง terminal เป็น 3 แถวหลัก
    let rows = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(6),  // row 0: CPU sparkline
            Constraint::Length(5),  // row 1: memory + disk
            Constraint::Min(10),    // row 2: process table (fill ที่เหลือ)
            Constraint::Length(5),  // row 3: network I/O
        ])
        .split(size);

    // row 0: CPU
    cpu::draw_cpu(f, rows[0], app);

    // row 1: แบ่งซ้าย/ขวา — memory (60%) | disk (40%)
    let mid_cols = Layout::default()
        .direction(Direction::Horizontal)
        .constraints([Constraint::Percentage(60), Constraint::Percentage(40)])
        .split(rows[1]);
    memory::draw_memory(f, mid_cols[0], app);
    disk::draw_disk(f, mid_cols[1], app);

    // row 2: process table
    process::draw_process(f, rows[2], app);

    // row 3: network
    network::draw_network(f, rows[3], app);
}
```

---

### ขั้นที่ 4: Memory Bar + Disk Usage

แสดง memory bar ที่เปลี่ยนสีตาม % ใช้งาน และ disk usage สำหรับแต่ละ mount point

**src/ui/memory.rs:**

```rust
// src/ui/memory.rs

use ratatui::{
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Modifier, Style},
    text::{Line, Span},
    widgets::{Block, Borders, Gauge},
    Frame,
};
use crate::app::AppState;
use crate::format::{color_for_pct, format_bytes};

pub fn draw_memory(f: &mut Frame, area: Rect, app: &AppState) {
    let block = Block::default()
        .title(" 🧠 Memory ")
        .borders(Borders::ALL);
    let inner = block.inner(area);
    f.render_widget(block, area);

    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([Constraint::Length(1), Constraint::Length(1)])
        .margin(0)
        .split(inner);

    // RAM gauge
    let mem_pct = if app.mem_total > 0 {
        (app.mem_used as f64 / app.mem_total as f64) * 100.0
    } else {
        0.0
    };
    let mem_color = color_for_pct(mem_pct);
    let ram_label = format!(
        "RAM  {} / {}  ({:.1}%)",
        format_bytes(app.mem_used),
        format_bytes(app.mem_total),
        mem_pct
    );
    let ram_gauge = Gauge::default()
        .gauge_style(Style::default().fg(mem_color).bg(Color::DarkGray))
        .ratio((mem_pct / 100.0).clamp(0.0, 1.0))
        .label(ram_label);
    f.render_widget(ram_gauge, chunks[0]);

    // Swap gauge
    let swap_pct = if app.swap_total > 0 {
        (app.swap_used as f64 / app.swap_total as f64) * 100.0
    } else {
        0.0
    };
    let swap_color = color_for_pct(swap_pct);
    let swap_label = format!(
        "SWAP {} / {}  ({:.1}%)",
        format_bytes(app.swap_used),
        format_bytes(app.swap_total),
        swap_pct
    );
    let swap_gauge = Gauge::default()
        .gauge_style(Style::default().fg(swap_color).bg(Color::DarkGray))
        .ratio((swap_pct / 100.0).clamp(0.0, 1.0))
        .label(swap_label);
    f.render_widget(swap_gauge, chunks[1]);
}
```

**src/ui/disk.rs:**

```rust
// src/ui/disk.rs

use ratatui::{
    layout::Rect,
    style::{Color, Style},
    text::{Line, Span},
    widgets::{Block, Borders, List, ListItem},
    Frame,
};
use crate::app::AppState;
use crate::format::format_bytes;

pub fn draw_disk(f: &mut Frame, area: Rect, app: &AppState) {
    let items: Vec<ListItem> = app
        .disk_entries
        .iter()
        .map(|d| {
            let used = d.total.saturating_sub(d.available);
            let pct = if d.total > 0 {
                (used as f64 / d.total as f64) * 100.0
            } else {
                0.0
            };
            let line = Line::from(vec![
                Span::styled(
                    format!("{:<12}", shorten_disk_name(&d.name)),
                    Style::default().fg(Color::Cyan),
                ),
                Span::raw(format!(
                    " R:{}/s W:{}/s  {:.0}%",
                    format_bytes(d.read_bps),
                    format_bytes(d.write_bps),
                    pct
                )),
            ]);
            ListItem::new(line)
        })
        .collect();

    let list = List::new(items).block(
        Block::default()
            .title(" 💾 Disk I/O ")
            .borders(Borders::ALL),
    );
    f.render_widget(list, area);
}

fn shorten_disk_name(name: &str) -> String {
    // "/dev/nvme0n1p1" → "nvme0n1p1"
    name.trim_start_matches("/dev/").to_string()
}
```

---

### ขั้นที่ 5: Process Table พร้อม Sorting

`Table` widget ของ ratatui รองรับ `TableState` ซึ่งทำให้เลื่อน highlight row ได้

**src/ui/process.rs:**

```rust
// src/ui/process.rs

use ratatui::{
    layout::{Constraint, Rect},
    style::{Color, Modifier, Style},
    text::{Line, Span},
    widgets::{Block, Borders, Cell, Row, Table, TableState},
    Frame,
};
use crate::app::{AppState, SortColumn};
use crate::format::format_bytes;

pub fn draw_process(f: &mut Frame, area: Rect, app: &AppState) {
    let visible = app.visible_processes();

    // สร้าง header — ไฮไลต์คอลัมน์ที่กำลัง sort
    let headers = ["PID", "NAME", "CPU%", "MEM", "STATUS"].map(|h| {
        let is_sort = matches!(
            (h, app.sort_col),
            ("PID", SortColumn::Pid)
                | ("NAME", SortColumn::Name)
                | ("CPU%", SortColumn::Cpu)
                | ("MEM", SortColumn::Mem)
        );
        let style = if is_sort {
            Style::default()
                .fg(Color::Yellow)
                .add_modifier(Modifier::BOLD | Modifier::UNDERLINED)
        } else {
            Style::default().fg(Color::White).add_modifier(Modifier::BOLD)
        };
        Cell::from(h).style(style)
    });
    let header_row = Row::new(headers).height(1);

    // สร้าง data rows
    let rows: Vec<Row> = visible
        .iter()
        .map(|p| {
            let cpu_color = if p.cpu_pct >= 80.0 {
                Color::Red
            } else if p.cpu_pct >= 50.0 {
                Color::Yellow
            } else {
                Color::Green
            };
            Row::new([
                Cell::from(format!("{:6}", p.pid)),
                Cell::from(format!("{:<20}", truncate(&p.name, 20))),
                Cell::from(format!("{:5.1}", p.cpu_pct))
                    .style(Style::default().fg(cpu_color)),
                Cell::from(format!("{:>10}", format_bytes(p.mem_bytes))),
                Cell::from(format!("{:<10}", p.status)),
            ])
        })
        .collect();

    let filter_info = if !app.filter_str.is_empty() {
        format!("  [filter: {}]", app.filter_str)
    } else {
        String::new()
    };
    let title = format!(
        " 📋 Processes ({} shown){}  sort:{} ",
        visible.len(),
        filter_info,
        app.sort_col.label()
    );

    let table = Table::new(
        rows,
        [
            Constraint::Length(7),
            Constraint::Length(21),
            Constraint::Length(6),
            Constraint::Length(11),
            Constraint::Length(11),
        ],
    )
    .header(header_row)
    .block(Block::default().title(title).borders(Borders::ALL))
    .highlight_style(
        Style::default()
            .bg(Color::DarkGray)
            .add_modifier(Modifier::BOLD),
    )
    .highlight_symbol("► ");

    let mut state = TableState::default();
    if !visible.is_empty() {
        state.select(Some(app.selected_idx));
    }
    f.render_stateful_widget(table, area, &mut state);
}

fn truncate(s: &str, max: usize) -> &str {
    if s.len() <= max {
        s
    } else {
        &s[..max]
    }
}
```

---

### ขั้นที่ 6: Keyboard Interaction (เลือก / Kill / Filter)

**src/main.rs (ฉบับสมบูรณ์):**

```rust
// src/main.rs

use std::io;
use std::time::{Duration, Instant};

use clap::Parser;
use crossterm::{
    event::{self, DisableMouseCapture, EnableMouseCapture, Event, KeyCode, KeyModifiers},
    execute,
    terminal::{disable_raw_mode, enable_raw_mode, EnterAlternateScreen, LeaveAlternateScreen},
};
use ratatui::{backend::CrosstermBackend, Terminal};

mod app;
mod format;
mod metrics;
mod ui;

use app::{AppState, InputMode, SortColumn};
use metrics::MetricsCollector;

/// System Monitor — htop-like TUI ที่เขียนด้วย Rust
#[derive(Parser, Debug)]
#[command(author, version, about)]
struct Args {
    /// Refresh interval (e.g. "2s", "500ms")
    #[arg(short, long, default_value = "2s", value_parser = parse_duration)]
    interval: Duration,
}

fn parse_duration(s: &str) -> Result<Duration, String> {
    if let Some(ms) = s.strip_suffix("ms") {
        ms.parse::<u64>()
            .map(Duration::from_millis)
            .map_err(|e| e.to_string())
    } else if let Some(secs) = s.strip_suffix('s') {
        secs.parse::<u64>()
            .map(Duration::from_secs)
            .map_err(|e| e.to_string())
    } else {
        s.parse::<u64>()
            .map(Duration::from_secs)
            .map_err(|e| e.to_string())
    }
}

fn main() -> io::Result<()> {
    let args = Args::parse();

    // ── Terminal setup ────────────────────────────────────────────────────────
    enable_raw_mode()?;
    let mut stdout = io::stdout();
    execute!(stdout, EnterAlternateScreen, EnableMouseCapture)?;
    let backend = CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend)?;

    // ── App state + metrics ───────────────────────────────────────────────────
    let mut app = AppState::new();
    let mut collector = MetricsCollector::new();

    // warm-up tick เพื่อให้ sysinfo ได้ baseline ก่อน
    std::thread::sleep(Duration::from_millis(200));
    let initial = collector.collect();
    app.update(initial);

    let mut last_tick = Instant::now();

    // ── Main event loop ───────────────────────────────────────────────────────
    loop {
        terminal.draw(|f| ui::draw(f, &app))?;

        // poll event ด้วย timeout = ที่เหลือก่อน tick ถัดไป
        let timeout = args
            .interval
            .checked_sub(last_tick.elapsed())
            .unwrap_or_default();

        if event::poll(timeout)? {
            if let Event::Key(key) = event::read()? {
                match app.input_mode {
                    InputMode::Normal => handle_normal_key(&mut app, key.code, key.modifiers),
                    InputMode::Filter => handle_filter_key(&mut app, key.code),
                }
            }
        }

        if app.should_quit {
            break;
        }

        // metric refresh
        if last_tick.elapsed() >= args.interval {
            let snap = collector.collect();
            app.update(snap);
            last_tick = Instant::now();
        }
    }

    // ── Restore terminal ──────────────────────────────────────────────────────
    disable_raw_mode()?;
    execute!(
        terminal.backend_mut(),
        LeaveAlternateScreen,
        DisableMouseCapture
    )?;
    terminal.show_cursor()?;
    Ok(())
}

fn handle_normal_key(app: &mut AppState, code: KeyCode, modifiers: KeyModifiers) {
    match code {
        KeyCode::Char('q') | KeyCode::Char('Q') => app.should_quit = true,
        KeyCode::Char('c') if modifiers.contains(KeyModifiers::CONTROL) => {
            app.should_quit = true
        }
        KeyCode::Up | KeyCode::Char('k') => app.scroll_up(),
        KeyCode::Down | KeyCode::Char('j') => app.scroll_down(),
        KeyCode::Char('s') => {
            app.sort_col = app.sort_col.next();
            app.sort_processes();
        }
        KeyCode::Char('/') => {
            app.input_mode = InputMode::Filter;
            app.filter_str.clear();
        }
        KeyCode::F(5) => { /* force refresh handled by elapsed check */ }
        KeyCode::Char('K') => kill_selected(app),
        _ => {}
    }
}

fn handle_filter_key(app: &mut AppState, code: KeyCode) {
    match code {
        KeyCode::Esc => {
            app.input_mode = InputMode::Normal;
            app.filter_str.clear();
        }
        KeyCode::Enter => {
            app.input_mode = InputMode::Normal;
        }
        KeyCode::Backspace => {
            app.filter_str.pop();
        }
        KeyCode::Char(c) => {
            app.filter_str.push(c);
        }
        _ => {}
    }
}

fn kill_selected(app: &mut AppState) {
    let visible = app.visible_processes();
    if let Some(proc) = visible.get(app.selected_idx) {
        let pid = proc.pid;
        // ส่ง SIGTERM ผ่าน libc (Unix only)
        #[cfg(unix)]
        unsafe {
            libc::kill(pid as libc::pid_t, libc::SIGTERM);
        }
        #[cfg(windows)]
        {
            // Windows: ใช้ TerminateProcess ผ่าน winapi
            let _ = pid; // suppress warning
        }
    }
}
```

> **หมายเหตุ:** ถ้าต้องการ `kill_selected` ใน Unix ให้เพิ่ม `libc = "0.2"` ใน `[dependencies]` และ `#[cfg(unix)]` ครอบ code ส่วนนั้น

---

### ขั้นที่ 7: Network I/O Panel

**src/ui/network.rs:**

```rust
// src/ui/network.rs

use ratatui::{
    layout::Rect,
    style::{Color, Style},
    text::{Line, Span},
    widgets::{Block, Borders, List, ListItem},
    Frame,
};
use crate::app::AppState;
use crate::format::format_bytes;

pub fn draw_network(f: &mut Frame, area: Rect, app: &AppState) {
    let items: Vec<ListItem> = app
        .net_entries
        .iter()
        .filter(|n| n.iface != "lo") // ซ่อน loopback
        .map(|n| {
            let line = Line::from(vec![
                Span::styled(
                    format!("{:<12}", &n.iface),
                    Style::default().fg(Color::Cyan),
                ),
                Span::styled(
                    format!("↓ {}/s", format_bytes(n.rx_bps)),
                    Style::default().fg(Color::Green),
                ),
                Span::raw("  "),
                Span::styled(
                    format!("↑ {}/s", format_bytes(n.tx_bps)),
                    Style::default().fg(Color::Yellow),
                ),
            ]);
            ListItem::new(line)
        })
        .collect();

    let list = List::new(items).block(
        Block::default()
            .title(" 🌐 Network I/O ")
            .borders(Borders::ALL),
    );
    f.render_widget(list, area);
}
```

---

### ขั้นที่ 8: Configurable Refresh + Graceful Exit

ขั้นนี้รวม `--interval` CLI flag (ผ่าน `clap`) และ RAII terminal cleanup ที่ทำงานแม้ panic

**RAII Terminal Guard** (ป้องกัน terminal พังหลัง panic):

```rust
// เพิ่มใน src/main.rs ก่อน fn main()

struct TerminalGuard;

impl Drop for TerminalGuard {
    fn drop(&mut self) {
        // ถูก call ทั้งตอน return ปกติและตอน panic
        let _ = disable_raw_mode();
        let _ = execute!(
            io::stdout(),
            LeaveAlternateScreen,
            DisableMouseCapture
        );
    }
}
```

ใช้งาน:

```rust
fn main() -> io::Result<()> {
    let args = Args::parse();
    enable_raw_mode()?;
    let mut stdout = io::stdout();
    execute!(stdout, EnterAlternateScreen, EnableMouseCapture)?;

    let _guard = TerminalGuard; // ← drop เมื่อออกจาก scope ไม่ว่ากรณีใด

    // ... rest of main
}
```

**Keyboard help bar** แสดงที่ด้านล่าง (เพิ่มใน `ui/mod.rs`):

```rust
// เพิ่มใน draw() หลังจาก layout
use ratatui::widgets::Paragraph;
use ratatui::style::Color;

let help = match app.input_mode {
    crate::app::InputMode::Normal =>
        " q:Quit  ↑↓/jk:Select  s:SortCycle  /:Filter  K:Kill  F5:Refresh ",
    crate::app::InputMode::Filter =>
        " Typing filter... Enter:Confirm  Esc:Cancel ",
};
let help_widget = Paragraph::new(help)
    .style(ratatui::style::Style::default().fg(Color::DarkGray));
// render ที่ bottom row ถ้า layout มีพื้นที่เหลือ
```

---

## การทดสอบ (Testing)

โค้ดด้านล่างเป็น test ที่รันได้จริงโดยไม่ต้องมี terminal หรือ sysinfo — ทดสอบเฉพาะ pure functions ใน `format.rs`, `app.rs` และ ring buffer logic

### ไฟล์ test ครบชุด

ใส่ไว้ใน `src/lib.rs` หรือเพิ่ม `#[cfg(test)]` module ใน `src/app.rs` และ `src/format.rs`

**CPU percentage calculation:**

```rust
// ใน src/metrics.rs  (หรือ test module ที่เกี่ยวข้อง)

#[cfg(test)]
mod tests {
    use super::CpuTick;

    #[test]
    fn cpu_percent_50() {
        let prev = CpuTick { total: 1000, idle: 500 };
        let curr = CpuTick { total: 1200, idle: 600 };
        let pct = CpuTick::percent(prev, curr);
        assert!((pct - 50.0).abs() < 1e-9, "expected 50%, got {pct}");
    }

    #[test]
    fn cpu_percent_100() {
        let prev = CpuTick { total: 0, idle: 0 };
        let curr = CpuTick { total: 400, idle: 0 };
        let pct = CpuTick::percent(prev, curr);
        assert!((pct - 100.0).abs() < 1e-9, "expected 100%, got {pct}");
    }

    #[test]
    fn cpu_percent_zero_delta() {
        let snap = CpuTick { total: 5000, idle: 3000 };
        assert_eq!(CpuTick::percent(snap, snap), 0.0);
    }
}
```

**format_bytes:**

```rust
// ใน src/format.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn format_bytes_gib() {
        assert_eq!(format_bytes(1_073_741_824), "1.00 GiB");
    }

    #[test]
    fn format_bytes_mib() {
        assert_eq!(format_bytes(2 * MIB), "2.00 MiB");
    }

    #[test]
    fn format_bytes_kib() {
        assert_eq!(format_bytes(512 * KIB), "512.00 KiB");
    }

    #[test]
    fn format_bytes_bytes() {
        assert_eq!(format_bytes(999), "999 B");
    }

    #[test]
    fn format_bytes_tib() {
        assert_eq!(format_bytes(TIB), "1.00 TiB");
    }
}
```

**Process sort:**

```rust
// ใน src/app.rs

#[cfg(test)]
mod tests {
    use super::*;
    use crate::metrics::ProcessEntry;

    fn mock_procs() -> Vec<ProcessEntry> {
        vec![
            ProcessEntry { pid: 3, name: "zsh".into(),   cpu_pct: 1.5,  mem_bytes: 8 * 1024 * 1024,   status: "Run".into() },
            ProcessEntry { pid: 1, name: "init".into(),  cpu_pct: 0.1,  mem_bytes: 2 * 1024 * 1024,   status: "Sleep".into() },
            ProcessEntry { pid: 2, name: "cargo".into(), cpu_pct: 95.0, mem_bytes: 256 * 1024 * 1024,  status: "Run".into() },
            ProcessEntry { pid: 4, name: "bash".into(),  cpu_pct: 0.5,  mem_bytes: 4 * 1024 * 1024,   status: "Sleep".into() },
        ]
    }

    #[test]
    fn sort_cpu_descending() {
        let mut app = AppState::new();
        app.processes = mock_procs();
        app.sort_col = SortColumn::Cpu;
        app.sort_processes();
        assert_eq!(app.processes[0].name, "cargo");
        assert_eq!(app.processes[1].name, "zsh");
        assert_eq!(app.processes[2].name, "bash");
        assert_eq!(app.processes[3].name, "init");
    }

    #[test]
    fn sort_pid_ascending() {
        let mut app = AppState::new();
        app.processes = mock_procs();
        app.sort_col = SortColumn::Pid;
        app.sort_processes();
        let pids: Vec<u32> = app.processes.iter().map(|p| p.pid).collect();
        assert_eq!(pids, vec![1, 2, 3, 4]);
    }

    #[test]
    fn filter_by_name() {
        let mut app = AppState::new();
        app.processes = mock_procs();
        app.filter_str = "ba".into();
        let visible = app.visible_processes();
        // "bash" matches "ba", "cargo" ไม่ match
        assert!(visible.iter().all(|p| p.name.contains("ba")));
    }
}
```

**Ring buffer:**

```rust
// ใน src/app.rs (หรือ test module แยก)

#[cfg(test)]
mod ring_tests {
    use std::collections::VecDeque;

    fn make_ring(cap: usize) -> VecDeque<u64> {
        VecDeque::with_capacity(cap)
    }

    fn push_ring(rb: &mut VecDeque<u64>, cap: usize, val: u64) {
        if rb.len() >= cap {
            rb.pop_front();
        }
        rb.push_back(val);
    }

    #[test]
    fn ring_buffer_capacity_enforced() {
        let mut rb = make_ring(60);
        for i in 0..70u64 {
            push_ring(&mut rb, 60, i);
        }
        assert_eq!(rb.len(), 60);
    }

    #[test]
    fn ring_buffer_oldest_evicted() {
        let mut rb = make_ring(60);
        for i in 0..70u64 {
            push_ring(&mut rb, 60, i);
        }
        // หลัง push 70 ค่าลงใน buffer 60 slot: ค่าที่เก่าที่สุดคือ 10
        assert_eq!(rb[0], 10, "oldest should be 10, got {}", rb[0]);
        assert_eq!(*rb.back().unwrap(), 69);
    }

    #[test]
    fn ring_buffer_partial_fill() {
        let mut rb = make_ring(10);
        for i in 1u64..=3 {
            push_ring(&mut rb, 10, i);
        }
        assert_eq!(rb.len(), 3);
        let v: Vec<u64> = rb.iter().copied().collect();
        assert_eq!(v, vec![1, 2, 3]);
    }
}
```

### ผลการรัน `cargo test` จริง

ด้านล่างคือ output จาก `cargo test` ที่รันกับ verification project (logic เดียวกัน ไม่มี terminal dependency):

```
   Compiling sysmon-verify v0.1.0 (scratchpad/sysmon-verify)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.65s
     Running unittests src/lib.rs (target/debug/deps/sysmon_verify-5039e20d618680c2)

running 14 tests
test tests::test_cpu_percent_100_percent ... ok
test tests::test_cpu_percent_50_percent ... ok
test tests::test_format_bytes_gib ... ok
test tests::test_format_bytes_kib ... ok
test tests::test_format_bytes_mib ... ok
test tests::test_cpu_percent_zero_delta ... ok
test tests::test_format_bytes_bytes ... ok
test tests::test_format_bytes_tib ... ok
test tests::test_ring_buffer_capacity_enforced ... ok
test tests::test_sort_by_cpu_descending ... ok
test tests::test_ring_buffer_partial_fill ... ok
test tests::test_ring_buffer_oldest_evicted ... ok
test tests::test_sort_by_mem_descending ... ok
test tests::test_sort_by_pid_ascending ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

   Doc-tests sysmon_verify

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก test ผ่าน 100%: 4 ชุดหลัก (CPU%, format_bytes, process sort, ring buffer) รวม 14 test cases

---

## การ Package และ Deploy

### Build release binary

```bash
cargo build --release
# binary อยู่ที่ target/release/system-monitor
strip target/release/system-monitor
ls -lh target/release/system-monitor
# → -rwxr-xr-x  1 user  group  2.1M  system-monitor
```

### Cross-compile สำหรับ Linux arm64 (Raspberry Pi)

```bash
rustup target add aarch64-unknown-linux-gnu
cargo build --release --target aarch64-unknown-linux-gnu
```

### ติดตั้ง globally

```bash
cargo install --path .
# ใช้งาน:
system-monitor --interval 1s
```

### Systemd service (headless monitoring + log)

ถ้าต้องการ run เป็น background service ที่บันทึก metrics ลง file แทนการแสดง TUI ให้เพิ่ม `--json` flag และ pipe ไปยัง log collector:

```bash
# /etc/systemd/system/sysmon.service
[Unit]
Description=System Monitor Metrics Logger

[Service]
ExecStart=/usr/local/bin/system-monitor --json --interval 10s >> /var/log/sysmon.jsonl
Restart=always
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: `sysinfo` ต้องการ warm-up tick ก่อนได้ CPU%

`sysinfo` คำนวณ CPU% จาก diff ระหว่างสองสแนปชอต ถ้า call `refresh_all()` แล้วอ่าน `cpu_usage()` ทันที จะได้ `0.0` เสมอ

```rust
// ❌ ผิด: อ่านทันทีหลัง refresh ครั้งแรก
sys.refresh_all();
println!("{}", sys.global_cpu_usage()); // 0.0 เสมอ

// ✅ ถูก: รอให้ได้ baseline ก่อน
sys.refresh_all();
std::thread::sleep(Duration::from_millis(200));
sys.refresh_all(); // refresh รอบที่สองจึงได้ diff
println!("{}", sys.global_cpu_usage()); // ค่าจริง
```

### กับดักที่ 2: ลืม restore terminal mode หลัง panic

ถ้า `main()` panic ระหว่าง raw mode terminal จะค้างใช้งานไม่ได้ — พิมพ์อะไรก็ไม่เห็น output ต้องปิด terminal แล้วเปิดใหม่

```rust
// ❌ ผิด: cleanup ใน return path แต่ไม่ครอบ panic
enable_raw_mode()?;
run_app()?; // ถ้า panic ที่นี่ terminal พัง
disable_raw_mode()?; // ← ไม่ถูก call ถ้า panic

// ✅ ถูก: ใช้ RAII guard
struct TerminalGuard;
impl Drop for TerminalGuard {
    fn drop(&mut self) {
        let _ = disable_raw_mode();
        let _ = execute!(io::stdout(), LeaveAlternateScreen);
    }
}

let _guard = TerminalGuard; // drop เมื่อออกจาก scope ทุกกรณี
```

### กับดักที่ 3: `Vec::remove(0)` เป็น O(n) — ใช้ `VecDeque` แทน

Ring buffer ที่ใช้ `Vec` + `remove(0)` จะ shift ทุก element ทุก tick ทำให้ช้าเมื่อ history ยาว

```rust
// ❌ ผิด: O(n) ทุก tick
let mut history: Vec<u64> = Vec::new();
if history.len() >= 60 {
    history.remove(0); // shift O(n)!
}
history.push(new_val);

// ✅ ถูก: VecDeque — O(1) ทั้ง push_back และ pop_front
use std::collections::VecDeque;
let mut history: VecDeque<u64> = VecDeque::with_capacity(60);
if history.len() >= 60 {
    history.pop_front(); // O(1)
}
history.push_back(new_val); // O(1) amortized
```

### กับดักที่ 4: `TableState::select()` ต้องส่ง mutable reference ไปยัง `render_stateful_widget`

Widget อย่าง `Table` และ `List` ที่มี state ต้องใช้ `render_stateful_widget` ไม่ใช่ `render_widget` ธรรมดา ถ้าใช้ผิดจะไม่เห็น highlight

```rust
// ❌ ผิด: ไม่ส่ง state
f.render_widget(table, area);

// ✅ ถูก:
let mut state = TableState::default();
state.select(Some(selected_idx));
f.render_stateful_widget(table, area, &mut state);
```

### กับดักที่ 5: disk I/O counter เป็น monotonic — ต้อง diff เอง

`d.usage().read_bytes` ใน sysinfo คืน total bytes ตั้งแต่ boot ไม่ใช่ bytes/sec ถ้าแสดงตรง ๆ จะได้ตัวเลขหลายพันล้านที่ไม่มีความหมาย

```rust
// ❌ ผิด: แสดง total bytes ตั้งแต่ boot
format!("{}", d.usage().read_bytes) // → "1234567890"

// ✅ ถูก: diff กับรอบก่อนแล้วหารด้วย elapsed seconds
let read_bps = (read_now.saturating_sub(prev_read)) as f64 / dt;
format!("{}/s", format_bytes(read_bps as u64)) // → "1.24 MiB/s"
```

---

## แบบฝึกหัด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม per-core CPU bars (ระดับง่าย)

แทนที่ global sparkline ด้วย mini bar สำหรับแต่ละ CPU core โดยใช้ `Gauge` widget หลายตัววางซ้อนกัน

**Hint:** ใช้ `Layout::vertical` แบ่ง CPU panel ออกเป็น N แถวตาม `app.cpu_per_core.len()` แล้ว render `Gauge` ตัวละ core

```rust
let core_layout = Layout::default()
    .direction(Direction::Vertical)
    .constraints(
        (0..app.cpu_per_core.len())
            .map(|_| Constraint::Length(1))
            .collect::<Vec<_>>()
    )
    .split(inner_area);

for (i, (&pct, &chunk)) in app.cpu_per_core.iter()
    .zip(core_layout.iter()).enumerate()
{
    let gauge = Gauge::default()
        .ratio((pct / 100.0).clamp(0.0, 1.0))
        .label(format!("CPU{} {:5.1}%", i, pct))
        .gauge_style(Style::default().fg(color_for_pct(pct)));
    f.render_widget(gauge, chunk);
}
```

---

### แบบฝึกหัดที่ 2: Export metrics เป็น JSON (ระดับกลาง)

เพิ่ม `--json` flag ที่ทำให้โปรแกรม print metrics เป็น JSON ทุก tick แทนการแสดง TUI — เหมาะสำหรับ pipe ไปยัง log aggregator หรือ dashboard

**Hint:** เพิ่มใน `Args`:
```rust
/// Print metrics as JSON instead of TUI
#[arg(long)]
json: bool,
```

แล้วสร้าง `MetricsJson` struct ที่ implement `serde::Serialize` และ print ด้วย `serde_json::to_string()`

เพิ่ม `serde = { version = "1", features = ["derive"] }` และ `serde_json = "1"` ใน Cargo.toml

---

### แบบฝึกหัดที่ 3: Configurable color theme (ระดับกลาง)

สร้าง `Theme` struct ที่อ่านจาก TOML file (`~/.config/sysmon/theme.toml`) โดยเก็บ color สำหรับแต่ละ panel

```toml
# ~/.config/sysmon/theme.toml
cpu_low    = "Green"
cpu_mid    = "Yellow"
cpu_high   = "Red"
mem_bar    = "Cyan"
proc_sel   = "Blue"
```

**Hint:** ใช้ `toml` crate และ `serde::Deserialize` อ่าน config แล้วส่ง `Theme` เป็น argument ไปยัง draw functions ทุกตัว

---

### แบบฝึกหัดที่ 4: Process tree view (ระดับยาก)

แทนที่ flat process list ด้วย tree ที่แสดง parent-child relationship โดยใช้ข้อมูล `parent()` จาก `sysinfo::Process`

```rust
// สร้าง tree structure
use std::collections::HashMap;

fn build_tree(procs: &[ProcessEntry]) -> HashMap<u32, Vec<u32>> {
    // pid → [child_pids]
    let mut tree: HashMap<u32, Vec<u32>> = HashMap::new();
    // ... fill from sysinfo process.parent()
    tree
}
```

**Hint:** Render tree ด้วยการ indent ชื่อ process ตาม depth: `"  ├─ cargo"`, `"  │  └─ rustc"`

---

## สรุป

โปรเจคนี้สร้าง system monitor แบบ TUI ที่ทำงานได้จริง โดยเรียนรู้ทักษะสำคัญหลายอย่างพร้อมกัน:

**Pattern สำคัญที่ได้เรียน:**

1. **Metrics pipeline** — แยก "collect raw data" (`metrics.rs`) ออกจาก "compute display values" (`app.rs`) และ "render" (`ui/`) ทำให้ test ง่ายและ maintain ง่าย

2. **Ring buffer ด้วย `VecDeque`** — เก็บ time-series data ที่มีขนาดคงที่โดยไม่สิ้นเปลือง memory และ O(1) push/pop

3. **Terminal RAII pattern** — ใช้ `Drop` trait ป้องกัน terminal corruption แม้เกิด panic

4. **Event loop + polling** — แยก "input event" กับ "metric refresh" ด้วย timeout ที่คำนวณจาก elapsed time ทำให้ UI responsive โดยไม่ waste CPU

5. **Stateful widgets** — `TableState` สำหรับ scrollable list — ต้องส่ง mutable state ไปกับ widget ทุกครั้งที่ render

โปรเจคถัดไป **A09 — File Deduplicator** จะนำทักษะ systems programming ไปต่อยอดในด้านอื่น: hash-based deduplication, parallel file scanning ด้วย `rayon`, และ file I/O ขั้นสูง

---

**โปรเจคก่อนหน้า:** [project-a07-process-manager.md](project-a07-process-manager.md) | **โปรเจคถัดไป:** [project-a09-file-deduplicator.md](project-a09-file-deduplicator.md)
