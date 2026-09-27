# Project A09: File Deduplicator

> โมดูล: CLI & Systems Tools | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 3 ชั่วโมง

## ภาพรวมโปรเจค

File Deduplicator คือเครื่องมือ CLI ที่สแกน directory ต้นทางแล้วหาไฟล์ที่มีเนื้อหาซ้ำกัน
(duplicate files) ด้วยวิธีที่ถูกต้องและรวดเร็ว: เปรียบเทียบด้วย cryptographic hash
แทนที่จะใช้ชื่อไฟล์หรือขนาดอย่างเดียว

ในโลก production เครื่องมือแบบนี้มีประโยชน์อย่างยิ่งในหลายสถานการณ์:

- **การจัดการไฟล์ภาพและวิดีโอ** ที่ copy-paste จาก SD card มาหลายครั้งโดยไม่ตั้งใจ
- **Backup deduplication** ลดพื้นที่ที่ backup ซ้ำซ้อนในระยะยาว
- **Developer toolchain** เช่น ลบ `.class`, `.o`, หรือ generated files ที่เหมือนกันทุก build
- **Enterprise storage audit** วิเคราะห์ว่า storage ถูกใช้ซ้ำซ้อนเท่าไหร่

โปรเจคนี้สร้าง CLI tool ที่ผสาน:

1. **pre-filter by size** — กรองไฟล์ที่ขนาดไม่เหมือนกันออกก่อน ทำให้ลดจำนวน hash ที่ต้องคำนวณ
2. **parallel hashing** ด้วย `blake3` + `rayon` — ใช้ CPU ทุก core ได้เต็มที่
3. **action modes** หลายแบบ: dry-run, delete, hardlink, move-to
4. **progress bars** แบบ multi-phase ด้วย `indicatif`
5. **JSON report** สำหรับ audit trail

**Learning value หลัก**: เข้าใจว่าทำไม pre-filter ถึงเร็วกว่า hash ทุกไฟล์,
วิธีใช้ `rayon` เพื่อ CPU parallelism, และการออกแบบ safety checks
ก่อนลบไฟล์ใน production tool

---

## สิ่งที่จะได้เรียนรู้

- **Size-based pre-filtering**: grouping ด้วย `HashMap<u64, Vec<PathBuf>>` ก่อน hash
- **Blake3 hashing**: fast cryptographic hash ที่เร็วกว่า MD5 และ SHA-256 บน modern hardware
- **Rayon parallel iteration**: `par_iter()` เพื่อ saturate CPU cores อย่างถูกต้อง
- **WalkDir**: การ walk directory อย่าง configurable รวมถึง symlink handling
- **Hardlink mechanics**: `fs::hard_link` และการตรวจสอบด้วย inode number
- **indicatif MultiProgress**: progress bar แบบหลาย phase ใน terminal
- **Safety-first file operations**: re-verify hash ก่อนลบ, keep policy, interactive mode
- **Serde JSON report**: serialize/deserialize structured data สำหรับ audit trail

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 41–45** — Concurrency พื้นฐาน: `thread::spawn`, `Arc`, `Mutex`, และ data races
- **Part 46–50** — Error handling: `Result`, `?` operator, custom error types
- **Part 52–54** — CLI argument parsing ด้วย `clap` derive macro
- **Part 55–57** — File I/O: `std::fs`, `std::path::Path` / `PathBuf`, metadata
- **Part 58–59** — Collections: `HashMap`, iterators, chaining
- **Part 60** — External crates: `walkdir`, `serde`, `rayon`

---

## โครงสร้างโปรเจค (Project Layout)

```
file-dedup/
├── src/
│   ├── main.rs          # CLI definition, entrypoint, orchestration
│   ├── walker.rs        # walk_and_group_by_size, exclude pattern logic
│   ├── hasher.rs        # hash_file, find_duplicate_groups (rayon)
│   ├── actions.rs       # delete, hardlink, move_duplicates, pick_keeper
│   ├── report.rs        # build_report, save_report, print_summary
│   └── progress.rs      # make_progress_bar helper
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

สำหรับการเรียน เราจะเขียนทุกอย่างใน `main.rs` ไฟล์เดียวก่อน
แล้วค่อย refactor เป็น module ในขั้นสุดท้าย

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Directories on disk
        │
        ▼
  WalkDir (Phase 1: Walk)
  ┌─────────────────────┐
  │  filter symlinks     │
  │  filter --exclude    │
  │  filter --min-size   │
  └────────┬────────────┘
           │  Vec<(size, PathBuf)>
           ▼
  Group by size: HashMap<u64, Vec<PathBuf>>
  (drop size groups with only 1 file — 80-95% eliminated here)
           │
           ▼
  Rayon parallel hash (Phase 2: Hash)
  ┌─────────────────────┐
  │  blake3::hash(file) │  ← all CPU cores in parallel
  └────────┬────────────┘
           │  Vec<(size, PathBuf, Hash)>
           ▼
  Group by hash: HashMap<Hash, Vec<PathBuf>>
  (drop groups with only 1 file)
           │
           ▼
  Vec<DupGroup>            ← sorted by size desc
           │
     ┌─────┼──────────────┬──────────────┐
     ▼     ▼              ▼              ▼
  DryRun  Delete       Hardlink       MoveTo
  print   rm + verify  link + verify  rename
           │
           ▼
  Report { groups, bytes_saveable, ... }
  save to duplicates.json
```

### ทำไมต้อง pre-filter by size?

ในระบบไฟล์จริง ไฟล์ส่วนใหญ่มีขนาดที่ unique ตัวอย่างเช่น:
สแกน `/usr/share/doc` บน Ubuntu ซึ่งมีไฟล์ ~50,000 ไฟล์ มักพบว่า 95%+ ของ
ไฟล์มีขนาด unique ถ้า hash ทุกไฟล์โดยตรงจะเปลืองเวลาไปกับไฟล์ที่ไม่มีทางเป็น
duplicate เลย การ group by size ก่อนทำให้เราลด hash candidate ลงได้อย่างมาก

### ทำไม Blake3?

| Algorithm | Speed (GB/s single-core) | Parallel? | Notes |
|-----------|--------------------------|-----------|-------|
| MD5       | ~0.5                     | No        | Broken cryptographically |
| SHA-256   | ~0.3                     | No        | Standard, slow |
| SHA-1     | ~0.7                     | No        | Broken |
| **Blake3**| **~6-10**                | **Yes**   | SIMD, tree-based |

Blake3 เร็วกว่า SHA-256 ถึง 20× บน modern x86 CPU ด้วย AVX-512
และรองรับ parallel hashing สำหรับไฟล์ขนาดใหญ่ด้วย

### Safety: ทำไมต้อง re-verify ก่อนลบ?

ระหว่างที่โปรแกรมกำลังรัน อาจมีกระบวนการอื่น (editor, sync tool)
แก้ไขไฟล์ระหว่าง hash phase กับ action phase ดังนั้น:

1. Phase hash: คำนวณและบันทึก hash ทั้งหมด
2. Phase action: ก่อนลบ/link ไฟล์ใด ๆ — อ่านและ hash ไฟล์นั้นใหม่อีกครั้ง
3. ถ้า hash ปัจจุบัน ≠ hash ที่บันทึกไว้ → skip + แจ้งเตือน

นี่คือ TOCTOU (Time-of-Check-Time-of-Use) protection ระดับพื้นฐาน

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Walk + Group by Size

เริ่มจาก `Cargo.toml` ที่กำหนด dependency ทั้งหมดก่อน:

```toml
[package]
name = "file-dedup"
version = "0.1.0"
edition = "2021"

[dependencies]
walkdir = "2"
blake3 = "1"
rayon = "1"
indicatif = "0.17"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
glob = "0.3"

[dev-dependencies]
tempfile = "3"
```

จากนั้นเขียน structure หลักของ CLI:

```rust
// src/main.rs — ขั้นที่ 1: Walk & group by size

use std::collections::HashMap;
use std::path::{Path, PathBuf};

use clap::{Parser, ValueEnum};
use walkdir::WalkDir;

#[derive(Parser, Debug)]
#[command(name = "file-dedup", about = "Fast file deduplicator")]
pub struct Cli {
    /// Directories to scan
    #[arg(required = true)]
    pub dirs: Vec<PathBuf>,

    /// Minimum file size to consider (e.g. 1MB, 512KB, 0)
    #[arg(long, default_value = "1")]
    pub min_size: String,

    /// Glob patterns to exclude (e.g. "*.log", ".git/**")
    #[arg(long, value_name = "PATTERN")]
    pub exclude: Vec<String>,

    /// Follow symbolic links
    #[arg(long)]
    pub follow_symlinks: bool,
}

/// แปลง string เช่น "1MB", "512KB", "4096" เป็น u64 bytes
pub fn parse_size(s: &str) -> Result<u64, String> {
    let s = s.trim();
    if s.is_empty() {
        return Ok(0);
    }
    let (num_part, unit) = if s.ends_with("KB") || s.ends_with("kb") {
        (&s[..s.len() - 2], 1024u64)
    } else if s.ends_with("MB") || s.ends_with("mb") {
        (&s[..s.len() - 2], 1024 * 1024)
    } else if s.ends_with("GB") || s.ends_with("gb") {
        (&s[..s.len() - 2], 1024 * 1024 * 1024)
    } else if s.ends_with('K') || s.ends_with('k') {
        (&s[..s.len() - 1], 1024u64)
    } else if s.ends_with('M') || s.ends_with('m') {
        (&s[..s.len() - 1], 1024 * 1024)
    } else if s.ends_with('G') || s.ends_with('g') {
        (&s[..s.len() - 1], 1024 * 1024 * 1024)
    } else {
        (s, 1u64)
    };
    let n: u64 = num_part
        .trim()
        .parse()
        .map_err(|_| format!("cannot parse size: {}", s))?;
    Ok(n * unit)
}

/// ตรวจว่า path ตรงกับ glob pattern ใด ๆ หรือไม่
fn is_excluded(path: &Path, patterns: &[String]) -> bool {
    let path_str = path.to_string_lossy();
    for pattern in patterns {
        if let Ok(pat) = glob::Pattern::new(pattern) {
            if pat.matches(&path_str) {
                return true;
            }
            if let Some(fname) = path.file_name() {
                if pat.matches(&fname.to_string_lossy()) {
                    return true;
                }
            }
        }
    }
    false
}

/// Phase 1: walk directories แล้ว group ไฟล์ตามขนาด
/// Return HashMap<file_size, Vec<PathBuf>> ที่มีเฉพาะ size ที่มีไฟล์ 2+ ชิ้น
pub fn walk_and_group_by_size(
    dirs: &[PathBuf],
    min_size: u64,
    excludes: &[String],
    follow_symlinks: bool,
) -> HashMap<u64, Vec<PathBuf>> {
    let mut by_size: HashMap<u64, Vec<PathBuf>> = HashMap::new();

    for dir in dirs {
        let walker = WalkDir::new(dir)
            .follow_links(follow_symlinks)
            .into_iter()
            .filter_map(|e| e.ok())
            .filter(|e| e.file_type().is_file());

        for entry in walker {
            let path = entry.path().to_path_buf();
            if is_excluded(&path, excludes) {
                continue;
            }
            if let Ok(meta) = entry.metadata() {
                let size = meta.len();
                if size < min_size {
                    continue;
                }
                by_size.entry(size).or_default().push(path);
            }
        }
    }

    // เก็บเฉพาะ size group ที่มีไฟล์ 2+ ชิ้น
    by_size.retain(|_, files| files.len() > 1);
    by_size
}

fn main() {
    let cli = Cli::parse();
    let min_size = parse_size(&cli.min_size).expect("invalid --min-size");

    let by_size = walk_and_group_by_size(
        &cli.dirs,
        min_size,
        &cli.exclude,
        cli.follow_symlinks,
    );

    let candidate_count: usize = by_size.values().map(|v| v.len()).sum();
    println!("Size groups with potential duplicates: {}", by_size.len());
    println!("Candidate files to hash: {}", candidate_count);
}
```

ทดสอบด้วย:

```bash
$ cargo run -- /tmp
Size groups with potential duplicates: 12
Candidate files to hash: 31
```

**สิ่งสำคัญที่ต้องเข้าใจ**: `retain` ทำงานใน-place บน `HashMap`
มัน iterate และลบ entry ที่ predicate return `false`
ซึ่งเป็น idiomatic Rust แทนที่จะ filter แล้ว collect ใหม่

---

### ขั้นที่ 2: Hash Files — หา Duplicate Hash Groups

เพิ่ม `hash_file` และ `find_duplicate_groups` โดยยังไม่ใช้ rayon ก่อน:

```rust
use std::fs;
use std::io;
use blake3::Hash;

/// อ่านไฟล์ทั้งหมดแล้ว hash ด้วย blake3
pub fn hash_file(path: &Path) -> Result<Hash, io::Error> {
    let data = fs::read(path)?;
    Ok(blake3::hash(&data))
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct FileEntry {
    pub path: String,
    pub modified: u64, // unix timestamp (seconds)
    pub inode: u64,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct DupGroup {
    pub hash: String,
    pub size: u64,
    pub files: Vec<FileEntry>,
}

/// Phase 2: hash ไฟล์ใน size groups แล้ว group ตาม hash
pub fn find_duplicate_groups(
    by_size: HashMap<u64, Vec<PathBuf>>,
) -> Vec<DupGroup> {
    use std::time::SystemTime;
    use std::os::unix::fs::MetadataExt;

    let mut by_hash: HashMap<Hash, Vec<(u64, PathBuf)>> = HashMap::new();

    for (size, paths) in by_size {
        for path in paths {
            if let Ok(hash) = hash_file(&path) {
                by_hash.entry(hash).or_default().push((size, path));
            }
        }
    }

    let mut groups: Vec<DupGroup> = by_hash
        .into_iter()
        .filter(|(_, files)| files.len() > 1)
        .map(|(hash, files)| {
            let size = files[0].0;
            let entries = files
                .into_iter()
                .map(|(_, path)| {
                    let (modified, inode) = fs::metadata(&path)
                        .map(|m| {
                            let mtime = m
                                .modified()
                                .ok()
                                .and_then(|t| {
                                    t.duration_since(SystemTime::UNIX_EPOCH).ok()
                                })
                                .map(|d| d.as_secs())
                                .unwrap_or(0);
                            (mtime, m.ino())
                        })
                        .unwrap_or((0, 0));
                    FileEntry {
                        path: path.to_string_lossy().into_owned(),
                        modified,
                        inode,
                    }
                })
                .collect();
            DupGroup {
                hash: hash.to_hex().to_string(),
                size,
                files: entries,
            }
        })
        .collect();

    // เรียงตาม size จากมากไปน้อย เพื่อแสดงผลที่ชัดเจน
    groups.sort_by(|a, b| b.size.cmp(&a.size).then(a.hash.cmp(&b.hash)));
    groups
}
```

`hash.to_hex()` ของ blake3 return `blake3::HexHash` ซึ่ง implement `Display`
และ `AsRef<str>` แต่ต้อง call `.to_string()` ถ้าต้องการเก็บเป็น `String`

---

### ขั้นที่ 3: Parallel Hashing ด้วย Rayon

sequential hashing ในขั้นที่ 2 ใช้ CPU core เดียวเท่านั้น
บน disk ที่เร็ว (NVMe SSD) bottleneck จริง ๆ คือ CPU ไม่ใช่ I/O
ดังนั้นเราใช้ `rayon` เพื่อ parallelize:

```rust
use rayon::prelude::*;

pub fn find_duplicate_groups(
    by_size: HashMap<u64, Vec<PathBuf>>,
    pb: Option<&indicatif::ProgressBar>,
) -> Vec<DupGroup> {
    use std::time::SystemTime;
    use std::os::unix::fs::MetadataExt;
    use std::sync::Mutex;

    // Flatten เป็น Vec ก่อนเพื่อใช้ par_iter
    let pairs: Vec<(u64, PathBuf)> = by_size
        .into_iter()
        .flat_map(|(size, paths)| paths.into_iter().map(move |p| (size, p)))
        .collect();

    // Parallel hash — rayon จะแบ่ง work ให้ thread pool อัตโนมัติ
    let hashed: Vec<(u64, PathBuf, Hash)> = pairs
        .into_par_iter()
        .filter_map(|(size, path)| {
            let result = hash_file(&path).ok().map(|h| (size, path, h));
            if let Some(pb) = pb {
                pb.inc(1);
            }
            result
        })
        .collect();

    // ส่วน grouping ยังคง sequential เพราะ HashMap ไม่ thread-safe
    let mut by_hash: HashMap<Hash, Vec<(u64, PathBuf)>> = HashMap::new();
    for (size, path, hash) in hashed {
        by_hash.entry(hash).or_default().push((size, path));
    }

    // ... (build DupGroup เหมือนเดิม)
    // (ละไว้ เพราะเหมือนขั้นที่ 2)
    todo!()
}
```

**ทำไม `par_iter()` ทำงานได้?**

`rayon` สร้าง global thread pool (ขนาดเท่ากับจำนวน logical CPU)
`into_par_iter()` แบ่ง iterator เป็น chunks ส่งให้แต่ละ thread
ทำงานพร้อมกัน แล้วรวมผลลัพธ์กลับ ไม่ต้องเขียน `thread::spawn` เอง

**ข้อควรระวัง**: closure ที่ส่งเข้า `par_iter()` ต้องเป็น `Send + Sync`
หากใช้ `Mutex<HashMap>` ภายใน closure ต้องระวัง deadlock

---

### ขั้นที่ 4: Dry-run Report

แสดงผลให้ user เห็นก่อนตัดสินใจว่าจะทำอะไร:

```rust
pub fn print_dry_run(groups: &[DupGroup]) {
    if groups.is_empty() {
        println!("No duplicates found.");
        return;
    }
    for (i, g) in groups.iter().enumerate() {
        let wasted = g.size * (g.files.len() as u64 - 1);
        println!(
            "\nGroup {} — {} bytes each, {} copies, wasted {} bytes",
            i + 1,
            g.size,
            g.files.len(),
            wasted,
        );
        println!("  Hash: {}...", &g.hash[..16]);
        for f in &g.files {
            println!("    {}", f.path);
        }
    }
    let total_wasted: u64 = groups
        .iter()
        .map(|g| g.size * (g.files.len() as u64 - 1))
        .sum();
    println!("\nTotal wasted: {} bytes ({:.2} MB)", total_wasted, total_wasted as f64 / 1_048_576.0);
}
```

ตัวอย่าง output:

```
Group 1 — 1048576 bytes each, 3 copies, wasted 2097152 bytes
  Hash: a3f8c2d1e5b9047a...
    /home/user/photos/IMG_0001.jpg
    /home/user/backup/IMG_0001.jpg
    /home/user/old-backup/IMG_0001.jpg

Group 2 — 204800 bytes each, 2 copies, wasted 204800 bytes
  Hash: 7b2e1d4f9c3a8056...
    /home/user/docs/report.pdf
    /home/user/shared/report.pdf

Total wasted: 2301952 bytes (2.20 MB)
```

---

### ขั้นที่ 5: Delete Action (Keep Newest/Oldest)

```rust
use std::io::{self, Write as IoWrite};

#[derive(clap::ValueEnum, Clone, Debug, PartialEq)]
pub enum KeepPolicy {
    Newest,
    Oldest,
}

/// เลือก index ของไฟล์ที่จะ KEEP ภายใน group
pub fn pick_keeper(group: &DupGroup, policy: &KeepPolicy) -> usize {
    match policy {
        KeepPolicy::Newest => group
            .files
            .iter()
            .enumerate()
            .max_by_key(|(_, f)| f.modified)
            .map(|(i, _)| i)
            .unwrap_or(0),
        KeepPolicy::Oldest => group
            .files
            .iter()
            .enumerate()
            .min_by_key(|(_, f)| f.modified)
            .map(|(i, _)| i)
            .unwrap_or(0),
    }
}

pub fn delete_duplicates(
    groups: &[DupGroup],
    policy: &KeepPolicy,
    interactive: bool,
    pb: Option<&indicatif::ProgressBar>,
) -> io::Result<u64> {
    let mut bytes_freed = 0u64;
    for group in groups {
        let keeper_idx = pick_keeper(group, policy);

        if interactive {
            println!("\nGroup (hash {}...):", &group.hash[..16]);
            for (i, f) in group.files.iter().enumerate() {
                let marker = if i == keeper_idx { "[KEEP]" } else { "[DEL] " };
                println!("  {} {}", marker, f.path);
            }
            print!("Proceed? [y/N] ");
            io::stdout().flush()?;
            let mut input = String::new();
            io::stdin().read_line(&mut input)?;
            if !input.trim().eq_ignore_ascii_case("y") {
                continue; // skip this group
            }
        }

        for (i, file) in group.files.iter().enumerate() {
            if i == keeper_idx {
                continue;
            }
            let path = std::path::Path::new(&file.path);

            // Safety: re-verify hash ก่อนลบ (TOCTOU protection)
            match hash_file(path) {
                Ok(current_hash) if current_hash.to_hex().as_str() == group.hash => {
                    std::fs::remove_file(path)?;
                    bytes_freed += group.size;
                    if let Some(pb) = pb {
                        pb.inc(1);
                    }
                }
                Ok(_) => {
                    eprintln!("WARNING: hash changed for {}, skipping", file.path);
                }
                Err(e) => {
                    eprintln!("WARNING: cannot read {}: {}", file.path, e);
                }
            }
        }
    }
    Ok(bytes_freed)
}
```

จุดสำคัญ: `hash_file` ถูกเรียกอีกครั้งสำหรับแต่ละไฟล์ที่จะลบ
ต้นทุนเพิ่มขึ้นเล็กน้อย แต่ป้องกันการลบไฟล์ที่ถูกแก้ไขระหว่างโปรแกรมรัน

---

### ขั้นที่ 6: Hardlink Action

Hardlink คือการทำให้หลาย path ชี้ไปยัง inode เดียวกันบน filesystem
ทำให้ไฟล์ที่ "duplicate" กันใช้พื้นที่จริง ๆ เพียงชุดเดียว แต่ทั้งสอง path
ยังคงใช้งานได้ปกติ (ต่างจาก symlink ที่ path หนึ่งอาจ broken ได้)

```rust
pub fn hardlink_duplicates(
    groups: &[DupGroup],
    policy: &KeepPolicy,
    pb: Option<&indicatif::ProgressBar>,
) -> io::Result<u64> {
    let mut bytes_saved = 0u64;
    for group in groups {
        let keeper_idx = pick_keeper(group, policy);
        let keeper_path = std::path::Path::new(&group.files[keeper_idx].path);

        for (i, file) in group.files.iter().enumerate() {
            if i == keeper_idx {
                continue;
            }
            let dup_path = std::path::Path::new(&file.path);

            match hash_file(dup_path) {
                Ok(h) if h.to_hex().as_str() == group.hash => {
                    // ลบ duplicate ก่อน แล้วสร้าง hardlink ไปหา keeper
                    std::fs::remove_file(dup_path)?;
                    std::fs::hard_link(keeper_path, dup_path)?;
                    bytes_saved += group.size;
                    if let Some(pb) = pb {
                        pb.inc(1);
                    }
                }
                Ok(_) => eprintln!("WARNING: hash mismatch for {}, skipping", file.path),
                Err(e) => eprintln!("WARNING: cannot read {}: {}", file.path, e),
            }
        }
    }
    Ok(bytes_saved)
}
```

**ข้อจำกัดของ hardlink** ที่ต้องรู้:
- ใช้ได้เฉพาะ filesystem เดียวกัน (ข้าม mount point ไม่ได้)
- ใช้ไม่ได้กับ directory
- บน Windows ต้องการ privilege พิเศษ
- เมื่อแก้ไขไฟล์ผ่าน path ใดก็ตาม จะกระทบทุก path ที่ link ถึง inode เดียวกัน

---

### ขั้นที่ 7: Progress Bars ด้วย indicatif

การใช้ `MultiProgress` ทำให้สามารถแสดง progress bar หลายอัน
แบบ interleaved ใน terminal ได้โดยไม่ทับกัน:

```rust
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};

pub fn make_progress_bar(len: u64, message: &str) -> ProgressBar {
    let pb = ProgressBar::new(len);
    pb.set_style(
        ProgressStyle::default_bar()
            .template(
                "{spinner:.cyan} {msg} [{bar:40.green/white}] {pos}/{len} ({eta})"
            )
            .unwrap()
            .progress_chars("=>-"),
    );
    pb.set_message(message.to_owned());
    pb
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ...
    let mp = MultiProgress::new();

    // Phase 1: Walk (ไม่รู้จำนวน → ใช้ spinner)
    let walk_pb = mp.add(ProgressBar::new_spinner());
    walk_pb.set_message("Walking files...");

    let by_size = walk_and_group_by_size(
        &cli.dirs, min_size, &cli.exclude, cli.follow_symlinks, Some(&walk_pb),
    );
    let candidate_count: usize = by_size.values().map(|v| v.len()).sum();
    walk_pb.finish_with_message(
        format!("Walk done — {} candidates", candidate_count)
    );

    // Phase 2: Hash (รู้จำนวนแล้ว → ใช้ bar)
    let hash_pb = mp.add(make_progress_bar(candidate_count as u64, "Hashing"));
    let groups = find_duplicate_groups(by_size, Some(&hash_pb));
    hash_pb.finish_with_message(format!("Found {} groups", groups.len()));

    // Phase 3: Action
    let dup_count: u64 = groups.iter().map(|g| g.files.len() as u64 - 1).sum();
    let action_pb = mp.add(make_progress_bar(dup_count, "Applying action"));
    // ... pass action_pb ไปให้ action function
    Ok(())
}
```

`walk_pb.inc(1)` เรียกได้จาก closure หรือ loop ได้ตลอดเพราะ `ProgressBar` implement `Clone` และ arc-wrapped ภายใน

---

### ขั้นที่ 8: JSON Report + Summary

```rust
use serde::{Deserialize, Serialize};
use std::path::Path;

#[derive(Debug, Serialize, Deserialize)]
pub struct Report {
    pub total_groups: usize,
    pub total_duplicate_files: usize,
    pub bytes_saveable: u64,
    pub groups: Vec<DupGroup>,
}

pub fn build_report(groups: &[DupGroup]) -> Report {
    let total_duplicate_files: usize = groups.iter().map(|g| g.files.len() - 1).sum();
    let bytes_saveable: u64 = groups
        .iter()
        .map(|g| g.size * (g.files.len() as u64 - 1))
        .sum();
    Report {
        total_groups: groups.len(),
        total_duplicate_files,
        bytes_saveable,
        groups: groups.to_vec(),
    }
}

pub fn save_report(report: &Report, path: &Path) -> std::io::Result<()> {
    let json = serde_json::to_string_pretty(report).expect("serialize");
    std::fs::write(path, json)
}

pub fn print_summary(report: &Report) {
    println!("\n=== Deduplication Summary ===");
    println!("Duplicate groups  : {}", report.total_groups);
    println!("Duplicate files   : {}", report.total_duplicate_files);
    println!(
        "Space saveable    : {} bytes ({:.2} MB)",
        report.bytes_saveable,
        report.bytes_saveable as f64 / 1_048_576.0
    );
}
```

ตัวอย่าง JSON output ใน `duplicates.json`:

```json
{
  "total_groups": 2,
  "total_duplicate_files": 3,
  "bytes_saveable": 2301952,
  "groups": [
    {
      "hash": "a3f8c2d1e5b9047a2f3c8e91b4d670a23f87c2d1e5b904a72f3c8e91b4d670a2",
      "size": 1048576,
      "files": [
        {
          "path": "/home/user/photos/IMG_0001.jpg",
          "modified": 1700000001,
          "inode": 1234567
        },
        {
          "path": "/home/user/backup/IMG_0001.jpg",
          "modified": 1700000500,
          "inode": 2345678
        }
      ]
    }
  ]
}
```

---

### โค้ดสมบูรณ์ (src/main.rs)

นี่คือโค้ดเต็มที่รวมทุกขั้นตอนเข้าด้วยกัน:

```rust
use std::collections::HashMap;
use std::fs;
use std::io::{self, Write as IoWrite};
use std::os::unix::fs::MetadataExt;
use std::path::{Path, PathBuf};
use std::time::SystemTime;

use blake3::Hash;
use clap::{Parser, ValueEnum};
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};
use rayon::prelude::*;
use serde::{Deserialize, Serialize};
use walkdir::WalkDir;

// ──────────────────────────────────────────────
// CLI Definition
// ──────────────────────────────────────────────

#[derive(Parser, Debug)]
#[command(
    name = "file-dedup",
    about = "Fast file deduplicator using blake3 hashing and rayon parallelism",
    version
)]
pub struct Cli {
    /// Directories to scan
    #[arg(required = true)]
    pub dirs: Vec<PathBuf>,

    /// Minimum file size to consider (e.g. 1MB, 512KB, 0)
    #[arg(long, default_value = "1")]
    pub min_size: String,

    /// Glob patterns to exclude (e.g. "*.log", ".git/**")
    #[arg(long, value_name = "PATTERN")]
    pub exclude: Vec<String>,

    /// Follow symbolic links
    #[arg(long)]
    pub follow_symlinks: bool,

    /// Which file to keep when deleting duplicates
    #[arg(long, default_value = "newest")]
    pub keep: KeepPolicy,

    /// Action to perform on duplicates
    #[arg(long, default_value = "dry-run")]
    pub action: Action,

    /// Move duplicates to this directory (used with --action move-to)
    #[arg(long, value_name = "DIR")]
    pub move_to: Option<PathBuf>,

    /// Ask before acting on each duplicate group
    #[arg(long)]
    pub interactive: bool,

    /// Write JSON report to this file
    #[arg(long, default_value = "duplicates.json")]
    pub report: PathBuf,

    /// Suppress progress bars (useful in tests / scripts)
    #[arg(long)]
    pub no_progress: bool,
}

#[derive(ValueEnum, Clone, Debug, PartialEq)]
pub enum KeepPolicy {
    Newest,
    Oldest,
}

#[derive(ValueEnum, Clone, Debug, PartialEq)]
pub enum Action {
    DryRun,
    Delete,
    Hardlink,
    MoveTo,
}

// ──────────────────────────────────────────────
// Domain types
// ──────────────────────────────────────────────

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DupGroup {
    pub hash: String,
    pub size: u64,
    pub files: Vec<FileEntry>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FileEntry {
    pub path: String,
    pub modified: u64,
    pub inode: u64,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Report {
    pub total_groups: usize,
    pub total_duplicate_files: usize,
    pub bytes_saveable: u64,
    pub groups: Vec<DupGroup>,
}

// ──────────────────────────────────────────────
// Size parsing
// ──────────────────────────────────────────────

pub fn parse_size(s: &str) -> Result<u64, String> {
    let s = s.trim();
    if s.is_empty() {
        return Ok(0);
    }
    let (num_part, unit) = if s.ends_with("KB") || s.ends_with("kb") {
        (&s[..s.len() - 2], 1024u64)
    } else if s.ends_with("MB") || s.ends_with("mb") {
        (&s[..s.len() - 2], 1024 * 1024)
    } else if s.ends_with("GB") || s.ends_with("gb") {
        (&s[..s.len() - 2], 1024 * 1024 * 1024)
    } else if s.ends_with('K') || s.ends_with('k') {
        (&s[..s.len() - 1], 1024u64)
    } else if s.ends_with('M') || s.ends_with('m') {
        (&s[..s.len() - 1], 1024 * 1024)
    } else if s.ends_with('G') || s.ends_with('g') {
        (&s[..s.len() - 1], 1024 * 1024 * 1024)
    } else {
        (s, 1u64)
    };
    let n: u64 = num_part
        .trim()
        .parse()
        .map_err(|_| format!("cannot parse size: {}", s))?;
    Ok(n * unit)
}

// ──────────────────────────────────────────────
// Walk + group by size
// ──────────────────────────────────────────────

fn is_excluded(path: &Path, patterns: &[String]) -> bool {
    let path_str = path.to_string_lossy();
    for pattern in patterns {
        if let Ok(pat) = glob::Pattern::new(pattern) {
            if pat.matches(&path_str) {
                return true;
            }
            if let Some(fname) = path.file_name() {
                if pat.matches(&fname.to_string_lossy()) {
                    return true;
                }
            }
        }
    }
    false
}

pub fn walk_and_group_by_size(
    dirs: &[PathBuf],
    min_size: u64,
    excludes: &[String],
    follow_symlinks: bool,
    pb: Option<&ProgressBar>,
) -> HashMap<u64, Vec<PathBuf>> {
    let mut by_size: HashMap<u64, Vec<PathBuf>> = HashMap::new();
    for dir in dirs {
        let walker = WalkDir::new(dir)
            .follow_links(follow_symlinks)
            .into_iter()
            .filter_map(|e| e.ok())
            .filter(|e| e.file_type().is_file());
        for entry in walker {
            let path = entry.path().to_path_buf();
            if is_excluded(&path, excludes) {
                continue;
            }
            if let Ok(meta) = entry.metadata() {
                let size = meta.len();
                if size < min_size {
                    continue;
                }
                by_size.entry(size).or_default().push(path);
                if let Some(pb) = pb {
                    pb.inc(1);
                }
            }
        }
    }
    by_size.retain(|_, files| files.len() > 1);
    by_size
}

// ──────────────────────────────────────────────
// Hash + find duplicates (parallel)
// ──────────────────────────────────────────────

pub fn hash_file(path: &Path) -> Result<Hash, io::Error> {
    let data = fs::read(path)?;
    Ok(blake3::hash(&data))
}

pub fn find_duplicate_groups(
    by_size: HashMap<u64, Vec<PathBuf>>,
    pb: Option<&ProgressBar>,
) -> Vec<DupGroup> {
    let pairs: Vec<(u64, PathBuf)> = by_size
        .into_iter()
        .flat_map(|(size, paths)| paths.into_iter().map(move |p| (size, p)))
        .collect();

    let hashed: Vec<(u64, PathBuf, Hash)> = pairs
        .into_par_iter()
        .filter_map(|(size, path)| {
            let result = hash_file(&path).ok().map(|h| (size, path, h));
            if let Some(pb) = pb {
                pb.inc(1);
            }
            result
        })
        .collect();

    let mut by_hash: HashMap<Hash, Vec<(u64, PathBuf)>> = HashMap::new();
    for (size, path, hash) in hashed {
        by_hash.entry(hash).or_default().push((size, path));
    }

    let mut groups: Vec<DupGroup> = by_hash
        .into_iter()
        .filter(|(_, files)| files.len() > 1)
        .map(|(hash, files)| {
            let size = files[0].0;
            let entries: Vec<FileEntry> = files
                .into_iter()
                .map(|(_, path)| {
                    let (modified, inode) = fs::metadata(&path)
                        .map(|m| {
                            let mtime = m
                                .modified()
                                .ok()
                                .and_then(|t| {
                                    t.duration_since(SystemTime::UNIX_EPOCH).ok()
                                })
                                .map(|d| d.as_secs())
                                .unwrap_or(0);
                            (mtime, m.ino())
                        })
                        .unwrap_or((0, 0));
                    FileEntry {
                        path: path.to_string_lossy().into_owned(),
                        modified,
                        inode,
                    }
                })
                .collect();
            DupGroup {
                hash: hash.to_hex().to_string(),
                size,
                files: entries,
            }
        })
        .collect();

    groups.sort_by(|a, b| b.size.cmp(&a.size).then(a.hash.cmp(&b.hash)));
    groups
}

// ──────────────────────────────────────────────
// Dry-run report
// ──────────────────────────────────────────────

pub fn print_dry_run(groups: &[DupGroup]) {
    if groups.is_empty() {
        println!("No duplicates found.");
        return;
    }
    for (i, g) in groups.iter().enumerate() {
        println!(
            "\nGroup {} — {} bytes each, {} copies, {} bytes wasted",
            i + 1,
            g.size,
            g.files.len(),
            g.size * (g.files.len() as u64 - 1)
        );
        for f in &g.files {
            println!("  {}", f.path);
        }
    }
}

// ──────────────────────────────────────────────
// Keep policy
// ──────────────────────────────────────────────

pub fn pick_keeper(group: &DupGroup, policy: &KeepPolicy) -> usize {
    match policy {
        KeepPolicy::Newest => group
            .files
            .iter()
            .enumerate()
            .max_by_key(|(_, f)| f.modified)
            .map(|(i, _)| i)
            .unwrap_or(0),
        KeepPolicy::Oldest => group
            .files
            .iter()
            .enumerate()
            .min_by_key(|(_, f)| f.modified)
            .map(|(i, _)| i)
            .unwrap_or(0),
    }
}

// ──────────────────────────────────────────────
// Delete action
// ──────────────────────────────────────────────

pub fn delete_duplicates(
    groups: &[DupGroup],
    policy: &KeepPolicy,
    interactive: bool,
    pb: Option<&ProgressBar>,
) -> io::Result<u64> {
    let mut bytes_freed = 0u64;
    for group in groups {
        let keeper_idx = pick_keeper(group, policy);
        if interactive {
            println!("\nDuplicate group (hash {}):", &group.hash[..16]);
            for (i, f) in group.files.iter().enumerate() {
                let marker = if i == keeper_idx { "[KEEP]" } else { "[DEL]" };
                println!("  {} {}", marker, f.path);
            }
            print!("Proceed? [y/N] ");
            io::stdout().flush()?;
            let mut input = String::new();
            io::stdin().read_line(&mut input)?;
            if !input.trim().eq_ignore_ascii_case("y") {
                continue;
            }
        }
        for (i, file) in group.files.iter().enumerate() {
            if i == keeper_idx {
                continue;
            }
            let path = Path::new(&file.path);
            match hash_file(path) {
                Ok(h) if h.to_hex().as_str() == group.hash => {
                    fs::remove_file(path)?;
                    bytes_freed += group.size;
                    if let Some(pb) = pb {
                        pb.inc(1);
                    }
                }
                Ok(_) => eprintln!("WARNING: hash mismatch for {}, skipping", file.path),
                Err(e) => eprintln!("WARNING: cannot read {}: {}", file.path, e),
            }
        }
    }
    Ok(bytes_freed)
}

// ──────────────────────────────────────────────
// Hardlink action
// ──────────────────────────────────────────────

pub fn hardlink_duplicates(
    groups: &[DupGroup],
    policy: &KeepPolicy,
    pb: Option<&ProgressBar>,
) -> io::Result<u64> {
    let mut bytes_saved = 0u64;
    for group in groups {
        let keeper_idx = pick_keeper(group, policy);
        let keeper_path = Path::new(&group.files[keeper_idx].path);
        for (i, file) in group.files.iter().enumerate() {
            if i == keeper_idx {
                continue;
            }
            let dup_path = Path::new(&file.path);
            match hash_file(dup_path) {
                Ok(h) if h.to_hex().as_str() == group.hash => {
                    fs::remove_file(dup_path)?;
                    fs::hard_link(keeper_path, dup_path)?;
                    bytes_saved += group.size;
                    if let Some(pb) = pb {
                        pb.inc(1);
                    }
                }
                Ok(_) => eprintln!("WARNING: hash mismatch for {}, skipping", file.path),
                Err(e) => eprintln!("WARNING: cannot read {}: {}", file.path, e),
            }
        }
    }
    Ok(bytes_saved)
}

// ──────────────────────────────────────────────
// Move-to action
// ──────────────────────────────────────────────

pub fn move_duplicates(
    groups: &[DupGroup],
    policy: &KeepPolicy,
    dest_dir: &Path,
    pb: Option<&ProgressBar>,
) -> io::Result<u64> {
    fs::create_dir_all(dest_dir)?;
    let mut bytes_moved = 0u64;
    for group in groups {
        let keeper_idx = pick_keeper(group, policy);
        for (i, file) in group.files.iter().enumerate() {
            if i == keeper_idx {
                continue;
            }
            let src = Path::new(&file.path);
            let fname = src.file_name().unwrap_or_default();
            // ป้องกัน name collision โดยใส่ hash prefix
            let dest_name = format!(
                "{}_{}",
                &group.hash[..8],
                fname.to_string_lossy()
            );
            let dest = dest_dir.join(dest_name);
            match hash_file(src) {
                Ok(h) if h.to_hex().as_str() == group.hash => {
                    fs::rename(src, &dest)?;
                    bytes_moved += group.size;
                    if let Some(pb) = pb {
                        pb.inc(1);
                    }
                }
                Ok(_) => eprintln!("WARNING: hash mismatch for {}, skipping", file.path),
                Err(e) => eprintln!("WARNING: cannot read {}: {}", file.path, e),
            }
        }
    }
    Ok(bytes_moved)
}

// ──────────────────────────────────────────────
// Report
// ──────────────────────────────────────────────

pub fn build_report(groups: &[DupGroup]) -> Report {
    let total_duplicate_files: usize = groups.iter().map(|g| g.files.len() - 1).sum();
    let bytes_saveable: u64 = groups
        .iter()
        .map(|g| g.size * (g.files.len() as u64 - 1))
        .sum();
    Report {
        total_groups: groups.len(),
        total_duplicate_files,
        bytes_saveable,
        groups: groups.to_vec(),
    }
}

pub fn save_report(report: &Report, path: &Path) -> io::Result<()> {
    let json = serde_json::to_string_pretty(report).expect("serialize report");
    fs::write(path, json)
}

pub fn print_summary(report: &Report) {
    println!("\n=== Deduplication Summary ===");
    println!("Duplicate groups  : {}", report.total_groups);
    println!("Duplicate files   : {}", report.total_duplicate_files);
    println!(
        "Space saveable    : {} bytes ({:.2} MB)",
        report.bytes_saveable,
        report.bytes_saveable as f64 / 1_048_576.0
    );
}

// ──────────────────────────────────────────────
// Progress bar helper
// ──────────────────────────────────────────────

pub fn make_progress_bar(len: u64, message: &str) -> ProgressBar {
    let pb = ProgressBar::new(len);
    pb.set_style(
        ProgressStyle::default_bar()
            .template(
                "{spinner:.cyan} {msg} [{bar:40.green/white}] {pos}/{len} ({eta})"
            )
            .unwrap()
            .progress_chars("=>-"),
    );
    pb.set_message(message.to_owned());
    pb
}

// ──────────────────────────────────────────────
// Main
// ──────────────────────────────────────────────

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cli = Cli::parse();
    let min_size = parse_size(&cli.min_size)?;

    let mp = MultiProgress::new();

    // Phase 1: Walk
    let walk_pb = if !cli.no_progress {
        let pb = mp.add(ProgressBar::new_spinner());
        pb.set_message("Walking files...");
        Some(pb)
    } else {
        None
    };

    let by_size = walk_and_group_by_size(
        &cli.dirs,
        min_size,
        &cli.exclude,
        cli.follow_symlinks,
        walk_pb.as_ref(),
    );

    let candidate_count: usize = by_size.values().map(|v| v.len()).sum();
    if let Some(pb) = &walk_pb {
        pb.finish_with_message(format!(
            "Walk done — {} candidates in {} size groups",
            candidate_count,
            by_size.len()
        ));
    }

    if candidate_count == 0 {
        println!("No duplicate candidates found (all files are unique by size).");
        return Ok(());
    }

    // Phase 2: Hash
    let hash_pb = if !cli.no_progress {
        Some(mp.add(make_progress_bar(candidate_count as u64, "Hashing files")))
    } else {
        None
    };

    let groups = find_duplicate_groups(by_size, hash_pb.as_ref());

    if let Some(pb) = &hash_pb {
        pb.finish_with_message(format!("Hash done — {} duplicate groups", groups.len()));
    }

    // Save report
    let report = build_report(&groups);
    save_report(&report, &cli.report)?;

    // Phase 3: Action
    let action_pb = if !cli.no_progress && !groups.is_empty() {
        let dup_count: u64 = groups.iter().map(|g| g.files.len() as u64 - 1).sum();
        Some(mp.add(make_progress_bar(dup_count, "Applying action")))
    } else {
        None
    };

    match cli.action {
        Action::DryRun => {
            print_dry_run(&groups);
        }
        Action::Delete => {
            let freed = delete_duplicates(
                &groups, &cli.keep, cli.interactive, action_pb.as_ref()
            )?;
            println!("\nDeleted duplicates. Freed {} bytes.", freed);
        }
        Action::Hardlink => {
            let saved = hardlink_duplicates(&groups, &cli.keep, action_pb.as_ref())?;
            println!("\nHardlinked duplicates. Saved {} bytes.", saved);
        }
        Action::MoveTo => {
            let dest = cli
                .move_to
                .as_deref()
                .ok_or("--move-to DIR required with --action move-to")?;
            let moved = move_duplicates(&groups, &cli.keep, dest, action_pb.as_ref())?;
            println!(
                "\nMoved {} bytes of duplicates to {}",
                moved,
                dest.display()
            );
        }
    }

    if let Some(pb) = &action_pb {
        pb.finish_with_message("Done");
    }

    print_summary(&report);
    println!("Report saved to: {}", cli.report.display());
    Ok(())
}

// ──────────────────────────────────────────────
// Tests
// ──────────────────────────────────────────────

#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;
    use std::os::unix::fs::MetadataExt;
    use tempfile::TempDir;

    fn write_file(dir: &Path, name: &str, content: &[u8]) -> PathBuf {
        let p = dir.join(name);
        fs::write(&p, content).unwrap();
        p
    }

    #[test]
    fn test_finds_one_duplicate_group() {
        let tmp = TempDir::new().unwrap();
        let dir = tmp.path();

        let content = b"hello duplicate world!";
        write_file(dir, "a.txt", content);
        write_file(dir, "b.txt", content);
        write_file(dir, "c.txt", content);
        write_file(dir, "unique1.txt", b"i am unique");
        write_file(dir, "unique2.txt", b"also unique content");

        let by_size = walk_and_group_by_size(
            &[dir.to_path_buf()], 0, &[], false, None
        );
        let groups = find_duplicate_groups(by_size, None);

        assert_eq!(groups.len(), 1, "expected exactly 1 duplicate group");
        assert_eq!(groups[0].files.len(), 3, "expected 3 files in the group");
    }

    #[test]
    fn test_min_size_filter() {
        let tmp = TempDir::new().unwrap();
        let dir = tmp.path();

        // 2 identical tiny files (10 bytes)
        let tiny = b"0123456789";
        write_file(dir, "tiny1.dat", tiny);
        write_file(dir, "tiny2.dat", tiny);

        // 2 identical larger files (100 bytes)
        let large = vec![0xABu8; 100];
        write_file(dir, "big1.dat", &large);
        write_file(dir, "big2.dat", &large);

        // min_size = 50 → tiny files (10 bytes) ควรถูก filter ออก
        let by_size = walk_and_group_by_size(
            &[dir.to_path_buf()], 50, &[], false, None
        );
        let groups = find_duplicate_groups(by_size, None);

        assert_eq!(groups.len(), 1, "only the large group should remain");
        assert_eq!(groups[0].size, 100);
    }

    #[test]
    fn test_keep_oldest_after_delete() {
        let tmp = TempDir::new().unwrap();
        let dir = tmp.path();

        let content = b"duplicate content for keep-oldest test";
        let old_path = write_file(dir, "old.txt", content);
        let new_path = write_file(dir, "new.txt", content);

        // ทดสอบ pick_keeper logic โดยตรงด้วย synthetic timestamps
        let group = DupGroup {
            hash: "aabbcc".into(),
            size: content.len() as u64,
            files: vec![
                FileEntry {
                    path: old_path.to_string_lossy().into_owned(),
                    modified: 0,    // oldest
                    inode: 1,
                },
                FileEntry {
                    path: new_path.to_string_lossy().into_owned(),
                    modified: 9999, // newest
                    inode: 2,
                },
            ],
        };

        let idx_oldest = pick_keeper(&group, &KeepPolicy::Oldest);
        assert_eq!(idx_oldest, 0, "oldest file (mtime=0) should be kept");

        let idx_newest = pick_keeper(&group, &KeepPolicy::Newest);
        assert_eq!(idx_newest, 1, "newest file (mtime=9999) should be kept");
    }

    #[test]
    fn test_hardlink_inode_match() {
        let tmp = TempDir::new().unwrap();
        let dir = tmp.path();

        let content = b"hardlink test content here";
        let file_a = write_file(dir, "a.bin", content);
        let file_b = write_file(dir, "b.bin", content);

        let by_size = walk_and_group_by_size(
            &[dir.to_path_buf()], 0, &[], false, None
        );
        let groups = find_duplicate_groups(by_size, None);

        assert_eq!(groups.len(), 1);

        let saved = hardlink_duplicates(&groups, &KeepPolicy::Newest, None).unwrap();
        assert!(saved > 0, "should report bytes saved");

        // ทั้งสอง path ต้องมี inode เดียวกัน
        let ino_a = fs::metadata(&file_a).unwrap().ino();
        let ino_b = fs::metadata(&file_b).unwrap().ino();
        assert_eq!(ino_a, ino_b, "hardlinked files must share the same inode");
    }

    #[test]
    fn test_parse_size() {
        assert_eq!(parse_size("0").unwrap(), 0);
        assert_eq!(parse_size("1024").unwrap(), 1024);
        assert_eq!(parse_size("1KB").unwrap(), 1024);
        assert_eq!(parse_size("1MB").unwrap(), 1_048_576);
        assert_eq!(parse_size("2GB").unwrap(), 2 * 1024 * 1024 * 1024);
        assert_eq!(parse_size("512kb").unwrap(), 512 * 1024);
    }

    #[test]
    fn test_report_bytes_saveable() {
        let groups = vec![DupGroup {
            hash: "abc".into(),
            size: 1000,
            files: vec![
                FileEntry { path: "a".into(), modified: 1, inode: 1 },
                FileEntry { path: "b".into(), modified: 2, inode: 2 },
                FileEntry { path: "c".into(), modified: 3, inode: 3 },
            ],
        }];
        let report = build_report(&groups);
        assert_eq!(report.total_groups, 1);
        assert_eq!(report.total_duplicate_files, 2); // 3 files - 1 keeper
        assert_eq!(report.bytes_saveable, 2000);     // 1000 * (3-1)
    }
}
```

---

## การทดสอบ (Testing)

### รัน Tests จริง

สร้าง scratch project ใน scratchpad แล้วรัน:

```bash
$ cargo test 2>&1
```

**Output จริงจากการรัน:**

```
   Compiling getrandom v0.4.3
   Compiling rustix v1.1.5
   Compiling bitflags v2.13.2
   Compiling linux-raw-sys v0.12.1
   Compiling fastrand v2.5.0
   Compiling tempfile v3.27.0
   Compiling file_dedup v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 2.72s
     Running unittests src/main.rs (target/debug/deps/file_dedup-62c5a08602cc7c7b)

running 6 tests
test tests::test_keep_oldest_after_delete ... ok
test tests::test_finds_one_duplicate_group ... ok
test tests::test_min_size_filter ... ok
test tests::test_hardlink_inode_match ... ok
test tests::test_parse_size ... ok
test tests::test_report_bytes_saveable ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### อธิบาย Tests แต่ละตัว

**test_finds_one_duplicate_group**: สร้าง 3 ไฟล์ที่มีเนื้อหาเหมือนกัน (`a.txt`, `b.txt`, `c.txt`) และ 2 ไฟล์ unique ใน `TempDir` แล้วตรวจว่า dedup engine หา group เดียวที่มี 3 ไฟล์

**test_min_size_filter**: สร้างไฟล์เล็ก (10 bytes) และไฟล์ใหญ่ (100 bytes) ที่แต่ละขนาดมีสำเนา 2 ชุด แล้วตรวจว่า `--min-size 50` filter ไฟล์เล็กออกได้

**test_keep_oldest_after_delete**: ทดสอบ `pick_keeper` logic โดยตรงด้วย synthetic `FileEntry` ที่มี `modified` timestamps ต่างกัน ตรวจว่า Oldest policy เลือก index ที่ถูกต้อง

**test_hardlink_inode_match**: รัน hardlink action บนไฟล์จริงใน TempDir แล้วตรวจว่า inode ของทั้งสอง path เหมือนกันหลังจาก hardlink ด้วย `MetadataExt::ino()`

**test_parse_size**: ทดสอบ size string parsing ครอบคลุม KB, MB, GB, lowercase, และตัวเลขล้วน

**test_report_bytes_saveable**: ตรวจ calculation ของ `bytes_saveable = size * (files - 1)` และ `total_duplicate_files = sum(files - 1)`

### Integration Test ตัวอย่าง

```rust
// tests/integration_test.rs
use std::fs;
use std::path::Path;
use tempfile::TempDir;

fn write_file(dir: &Path, name: &str, content: &[u8]) {
    fs::write(dir.join(name), content).unwrap();
}

#[test]
fn test_full_pipeline_delete() {
    let src = TempDir::new().unwrap();
    let content = b"integration test content -- same in all three files";

    write_file(src.path(), "dup1.txt", content);
    write_file(src.path(), "dup2.txt", content);
    write_file(src.path(), "dup3.txt", content);
    write_file(src.path(), "unique.txt", b"only one copy");

    // ตรวจว่ามี 4 ไฟล์ก่อน
    let before: Vec<_> = fs::read_dir(src.path()).unwrap().collect();
    assert_eq!(before.len(), 4);

    // รัน full pipeline ด้วย delete action
    use file_dedup::{
        walk_and_group_by_size, find_duplicate_groups,
        delete_duplicates, KeepPolicy,
    };

    let by_size = walk_and_group_by_size(
        &[src.path().to_path_buf()], 0, &[], false, None,
    );
    let groups = find_duplicate_groups(by_size, None);
    assert_eq!(groups.len(), 1);

    delete_duplicates(&groups, &KeepPolicy::Newest, false, None).unwrap();

    // หลัง delete ควรเหลือ 2 ไฟล์ (1 keeper + 1 unique)
    let after: Vec<_> = fs::read_dir(src.path()).unwrap().collect();
    assert_eq!(after.len(), 2);
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/file-dedup

# ตรวจขนาด (ควรอยู่ราว 3-5 MB)
ls -lh ./target/release/file-dedup
```

### ใช้งานจริง

```bash
# Dry-run สแกน home directory (ข้าม .git และ node_modules)
./file-dedup ~/Documents --exclude "*.git" --exclude "node_modules/**" \
    --min-size 1KB

# ลบ duplicates โดย keep newest
./file-dedup ~/Downloads --action delete --keep newest

# แทนที่ duplicate ด้วย hardlinks (ประหยัด space แต่ทุก path ยังใช้ได้)
./file-dedup ~/Photos --action hardlink

# ย้าย duplicates ไปยัง holding directory เพื่อ review ก่อนลบ
./file-dedup ~/backup --action move-to --move-to ~/duplicate-holding

# Interactive mode + dry-run
./file-dedup ~/workspace --interactive
```

### Cross-compile สำหรับ Linux Server

```bash
# ติดตั้ง cross-compilation target
rustup target add x86_64-unknown-linux-musl

# Build static binary (ไม่ต้องพึ่ง libc บน target)
cargo build --release --target x86_64-unknown-linux-musl

# อยู่ที่
./target/x86_64-unknown-linux-musl/release/file-dedup
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดักที่ 1: ลืม `retain` หลัง group by size

ถ้าไม่ filter size group ที่มีไฟล์แค่ชิ้นเดียวออก จะ hash ไฟล์ที่ไม่มีทางซ้ำ:

```rust
// ผิด: hash ทุกไฟล์โดยไม่ filter
let by_size = walk_all_files(); // HashMap<u64, Vec<PathBuf>>
// ← ถ้าไม่ retain จะ hash ไฟล์ที่ unique by size ด้วย ช้าโดยไม่จำเป็น

// ถูก: filter ก่อน
by_size.retain(|_, files| files.len() > 1);
```

บน directory ขนาดใหญ่ การ filter นี้อาจลดงาน hash ลงได้ถึง 95%

### กับดักที่ 2: Hash ซ้ำกันได้ถ้า Path String เหมือนกัน

ถ้าใช้ `HashMap<String, Vec<PathBuf>>` แทน `HashMap<Hash, Vec<PathBuf>>`
อาจเกิด false duplicate ถ้าคำนวณ hash เป็น string แล้ว convert กลับไปกลับมา:

```rust
// อันตราย: ใช้ hex string เป็น key ทำให้ตรวจ collision ยาก
let mut by_hash: HashMap<String, Vec<PathBuf>> = HashMap::new();
// ถ้าพิมพ์ผิด: "a3f8..." กับ "A3F8..." จะเป็น key ต่างกัน

// ถูก: ใช้ blake3::Hash โดยตรง (มี PartialEq/Hash ให้แล้ว)
let mut by_hash: HashMap<blake3::Hash, Vec<PathBuf>> = HashMap::new();
```

### กับดักที่ 3: Hardlink ข้าม Filesystem ไม่ได้

```rust
// นี่จะ panic หรือ return Err ถ้า src และ dst อยู่คนละ mount point
fs::hard_link("/home/user/file.txt", "/mnt/backup/file.txt")?;
// Error: os error 18 (EXDEV: Invalid cross-device link)
```

ต้องตรวจ filesystem ก่อน หรือ fallback เป็น copy + delete แทน:

```rust
match fs::hard_link(&keeper, &dup) {
    Ok(_) => { /* สำเร็จ */ }
    Err(e) if e.raw_os_error() == Some(18) => {
        // EXDEV: ข้าม filesystem — fallback เป็น copy
        eprintln!("Cross-device: copying instead of hardlinking");
        fs::copy(&keeper, &dup)?;
        // NOTE: ขนาดไฟล์จะไม่ลดลงในกรณีนี้
    }
    Err(e) => return Err(e),
}
```

### กับดักที่ 4: WalkDir ไม่ข้าม symlink โดย default

`WalkDir::new(dir)` โดย default จะ **ไม่** follow symlinks
ถ้า directory มี symlink ไปยังไฟล์ข้างนอก และไม่ได้ call `.follow_links(true)`
ไฟล์นั้นจะถูก skip ไปเงียบ ๆ ซึ่งอาจทำให้ผลลัพธ์ไม่ครบ:

```rust
// ถ้าต้องการ follow symlinks:
WalkDir::new(dir).follow_links(follow_symlinks)
// แต่ต้องระวัง symlink loop: WalkDir จัดการให้อัตโนมัติ
// ด้วย cycle detection ผ่าน inode tracking
```

ระวัง: ถ้า `follow_links(true)` อาจนับไฟล์เดิมหลายครั้ง
ถ้ามี symlink หลายอันชี้ไปที่ไฟล์เดียวกัน

### กับดักที่ 5: `par_iter` บน `HashMap` ต้องระวัง Data Race

`HashMap` ไม่ implement `Send` โดยตรงสำหรับ mutation:

```rust
// ผิด: ไม่สามารถ mutate HashMap จาก parallel closure โดยตรง
let mut by_hash = HashMap::new();
pairs.par_iter().for_each(|(size, path, hash)| {
    by_hash.entry(hash).or_default().push((size, path)); // ERROR: cannot borrow mutably
});

// ถูก: collect ก่อน แล้วค่อย group แบบ sequential
let hashed: Vec<_> = pairs.into_par_iter().map(|...| ...).collect();
let mut by_hash = HashMap::new();
for (size, path, hash) in hashed { // sequential grouping
    by_hash.entry(hash).or_default().push((size, path));
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Streaming Hash สำหรับไฟล์ขนาดใหญ่

โค้ดปัจจุบัน `fs::read(path)` อ่านไฟล์ทั้งหมดเข้า memory ก่อน hash
ถ้าไฟล์ขนาด 4GB จะใช้ RAM 4GB ให้แก้เป็น streaming:

```rust
use blake3::Hasher;
use std::io::{BufReader, Read};

pub fn hash_file_streaming(path: &Path) -> io::Result<Hash> {
    let file = std::fs::File::open(path)?;
    let mut reader = BufReader::with_capacity(64 * 1024, file); // 64KB buffer
    let mut hasher = Hasher::new();
    let mut buf = [0u8; 64 * 1024];
    loop {
        let n = reader.read(&mut buf)?;
        if n == 0 {
            break;
        }
        hasher.update(&buf[..n]);
    }
    Ok(hasher.finalize())
}
```

**Hint**: เปรียบเทียบ throughput ระหว่าง `fs::read` กับ streaming ด้วยไฟล์ 1GB

### แบบฝึกหัดที่ 2: Two-Phase Hashing (Partial Hash First)

สำหรับไฟล์ขนาดใหญ่ หาก size เหมือนกัน ให้ hash แค่ 64KB แรกก่อน
ถ้า partial hash ต่างกัน → ไม่ต้อง hash ทั้งไฟล์เลย:

```rust
pub enum HashResult {
    Unique,                // ไม่มีทางซ้ำ (partial hash ต่างกัน)
    PossibleDup(Hash),     // partial hash เหมือน → ต้องทำ full hash
    ConfirmedDup(Hash),    // full hash เหมือน
}

pub fn two_phase_hash(candidates: Vec<PathBuf>) -> Vec<DupGroup> {
    // Phase A: hash แค่ 4KB แรก
    let partial: HashMap<[u8; 32], Vec<PathBuf>> = ...;

    // Phase B: สำหรับ group ที่ partial hash เหมือนกัน → full hash
    // ...
}
```

### แบบฝึกหัดที่ 3: Cross-Filesystem Hardlink Fallback

แก้ `hardlink_duplicates` ให้ detect `EXDEV` error (errno 18) และ fallback
เป็น copy-then-delete แบบ atomic เพื่อรองรับ duplicate ที่อยู่ต่าง partition

### แบบฝึกหัดที่ 4: Database Cache ของ Hash Results

สร้าง local SQLite database (ด้วย `rusqlite` crate) ที่ cache
`(path, size, mtime) → blake3_hash` เพื่อไม่ต้อง re-hash ไฟล์ที่ไม่ได้เปลี่ยน
ระหว่าง run ครั้งที่สองบน directory เดิม

```toml
[dependencies]
rusqlite = { version = "0.31", features = ["bundled"] }
```

### แบบฝึกหัดที่ 5: TUI Mode ด้วย ratatui

แทนที่จะแสดง list แบบ text ให้สร้าง TUI interactive mode
ที่ user สามารถ browse duplicate groups, เลือกว่าจะ keep ไฟล์ไหน,
และกด Enter เพื่อยืนยัน ด้วย `ratatui` crate

### แบบฝึกหัดที่ 6: Watch Mode (ป้องกัน Duplicate ใหม่)

ใช้ `notify` crate เพื่อ watch directory และแจ้งเตือนทันทีเมื่อมีไฟล์ใหม่
ที่ hash ตรงกับไฟล์ที่มีอยู่แล้ว เป็นการป้องกัน duplicate แบบ real-time

---

## สรุป

ใน Project A09 เราสร้าง file deduplication tool ที่:

1. **Size pre-filter**: `HashMap<u64, Vec<PathBuf>>` กรองไฟล์ unique by size ออก 80-95%
2. **Blake3 parallel hash**: `rayon::par_iter()` + `blake3::hash()` ใช้ทุก CPU core
3. **Safety-first deletion**: re-verify hash ก่อน remove/link ทุกครั้ง
4. **Multiple action modes**: dry-run, delete, hardlink, move-to ตามความต้องการ
5. **Audit trail**: `serde_json` report บันทึก duplicates.json ทุกครั้งที่รัน
6. **Multi-phase progress**: `indicatif::MultiProgress` แยก bar ต่อ phase

**Pattern สำคัญที่ได้เรียน**:
- `HashMap::retain()` สำหรับ in-place filtering
- `into_par_iter()` + `collect()` pattern สำหรับ parallel computation
- TOCTOU protection: check-then-act ด้วย hash re-verification
- `std::fs::hard_link()` และการตรวจ inode equality
- `indicatif::MultiProgress` สำหรับ multi-phase CLI progress

โปรเจคถัดไป [A10 — Secret Vault](project-a10-secret-vault.md) จะนำทักษะ
file I/O และ hashing ที่ได้เรียนในโปรเจคนี้ไปประยุกต์กับ encryption:
เก็บข้อมูล sensitive ด้วย AES-GCM, key derivation ด้วย Argon2,
และ secure memory zeroing

---

**โปรเจคก่อนหน้า:** [A08 — System Monitor](project-a08-system-monitor.md) | **โปรเจคถัดไป:** [A10 — Secret Vault](project-a10-secret-vault.md)
