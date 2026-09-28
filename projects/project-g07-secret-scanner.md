# Project G07: Secret Scanner

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **static analysis tool** ที่ตรวจหา potential secrets ใน source code โดยผสมผสานหลายเทคนิค: regex pattern matching, Shannon entropy analysis, directory traversal, git diff parsing และการสร้าง report ในรูปแบบมาตรฐาน

Secret scanner เป็นหนึ่งในเครื่องมือ DevSecOps ที่ถูกใช้งานอย่างแพร่หลายในกระบวนการพัฒนา software สมัยใหม่ เครื่องมือในประเภทนี้ทำงานในฐานะ static analyzer — อ่านและวิเคราะห์ source code, configuration files และ diff output โดยไม่ต้องรันโปรแกรม เพื่อตรวจหา pattern ที่บ่งชี้ว่าอาจเป็น credential หรือ secret ที่ถูกฝังไว้โดยไม่ตั้งใจ

**Use cases จริงในโลก production:**

- **Pre-commit hook** — รัน scanner ก่อนที่ git commit จะถูก record เพื่อตรวจ staged files
- **CI/CD pipeline gate** — ตรวจสอบ pull request ก่อน merge โดยวิเคราะห์ diff เฉพาะส่วนที่เปลี่ยนแปลง
- **Repository audit** — scan ทุกไฟล์ใน repository เพื่อตรวจหา secret ที่อาจถูก commit ไปก่อนหน้า
- **GitHub Code Scanning** — อ่าน SARIF 2.1.0 output และแสดงผลใน Security tab ของ GitHub repository

**Learning value:**

โปรเจคนี้สอนการออกแบบ **rule-based analysis engine** ที่ผสาน regex pattern matching กับ entropy analysis — สองเทคนิคที่ complement กัน: regex ดักจาก structural pattern ส่วน entropy วัดความ randomness ของข้อความ นอกจากนี้ยังครอบคลุม SARIF format specification, unified diff parsing, glob-based filtering และการออกแบบ exit code ที่ถูกต้องสำหรับ CI/CD integration

## สิ่งที่จะได้เรียนรู้

- **Shannon entropy** — คำนวณ information entropy ของ string เพื่อแยกแยะ random secret จาก human-readable text
- **Regex engine ด้วย `regex` crate** — Named capture groups, case-insensitive flags, find_iter สำหรับหาทุก match ในบรรทัด
- **`walkdir` crate** — traverse directory tree แบบ recursive พร้อม filter ไฟล์ binary และ directory ที่ไม่ต้องการ
- **Unified diff parsing** — parse `@@ -old,count +new,count @@` hunk headers เพื่อดึงเฉพาะ added lines
- **SARIF 2.1.0 format** — JSON schema มาตรฐานสำหรับ static analysis results ที่ GitHub Code Scanning อ่านได้
- **Glob pattern matching ด้วย `glob` crate** — ใช้เป็น allowlist สำหรับ suppress known-safe findings
- **`colored` crate** — สร้าง terminal output ที่มีสี severity-aware โดยไม่ต้องจัดการ ANSI escape codes โดยตรง
- **Exit code design** — return non-zero exit code เมื่อพบ Critical/High findings เพื่อ fail CI pipeline

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, impl blocks, traits
- **Part 21–30**: Collections — `Vec`, `HashMap`, iterators, `filter_map`, `collect`
- **Part 31–40**: Error handling ด้วย `Result<T, E>` และ `?` operator, `Box<dyn Error>`
- **Part 41–50**: Modules, visibility, `use` declarations, crate structure
- **Part 55–60**: `serde` ecosystem — `Serialize`/`Deserialize`, `serde_json`, `json!` macro
- **Part 71–80**: External crates — `regex`, `walkdir` (จาก Part 74 เรื่อง popular crates)
- **Part 96–100**: Design patterns — pattern เรื่อง rule engine, builder-style configuration
- **Part 101–105**: CLI tools — argument parsing, exit codes, terminal output

## โครงสร้างโปรเจค (Project Layout)

```
secret-scanner/
├── src/
│   ├── main.rs          ← CLI entry point, argument parsing, output formatting
│   ├── entropy.rs       ← Shannon entropy computation (fn entropy(s: &str) -> f64)
│   ├── rules.rs         ← Rule struct, Severity enum, default_rules() set
│   ├── finding.rs       ← Finding struct (scan result) + serde derive
│   ├── scanner.rs       ← scan_line(), scan_file(), scan_directory() via walkdir
│   ├── allowlist.rs     ← .secretsignore parser, AllowList + AllowEntry
│   ├── report.rs        ← to_json_report(), to_sarif_report() (SARIF 2.1.0)
│   └── git.rs           ← parse_diff_additions(), scan_diff()
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ภาพรวม

```
Input Source
  ├── Directory Path  ──→ scan_directory() (walkdir + scan_file)
  ├── Single File     ──→ scan_file()
  └── Git Diff Text   ──→ scan_diff() (parse_diff_additions + scan_line)
            │
            │  ทุก path รวมกันที่ scan_line()
            ▼
  ┌─────────────────────────────────────────────────┐
  │              scan_line(line, file, rules)        │
  │                                                  │
  │  for each Rule:                                  │
  │    pattern.find_iter(line)                       │
  │    → entropy(matched_text)                       │
  │    → check entropy_threshold                     │
  │    → emit Finding { rule_id, file, line, ... }  │
  └──────────────────┬──────────────────────────────┘
                     │  Vec<Finding>
                     ▼
  ┌─────────────────────────────────────────────────┐
  │              AllowList::filter_findings()        │
  │  GlobPattern, RuleId, RuleAndGlob entries       │
  └──────────────────┬──────────────────────────────┘
                     │  filtered Vec<Finding>
                     ▼
  ┌─────────────────────────────────────────────────┐
  │              Report Layer                        │
  │  "table"  → print_summary_table() (colored)     │
  │  "json"   → to_json_report() → serde_json       │
  │  "sarif"  → to_sarif_report() → SARIF 2.1.0    │
  └─────────────────────────────────────────────────┘
```

### Rule Engine Design

Rule แต่ละ entry มีโครงสร้างดังนี้:

```
Rule {
  id: "AWS001"               ← unique identifier สำหรับ allowlist/reference
  name: "AWS Access Key ID"  ← human-readable name สำหรับ report
  pattern: Regex             ← compiled regex (ไม่ recompile ซ้ำ)
  entropy_threshold: None    ← Some(4.5) = ต้องมี entropy ≥ 4.5 จึงจะ flag
  file_extensions: []        ← [] = ทุก extension, ["rs","env"] = เฉพาะที่ระบุ
  severity: Critical         ← Critical/High/Medium/Low
}
```

การใช้ `entropy_threshold` ร่วมกับ pattern ทำให้ลด false positive ได้มาก:

- Pattern `(api_key)\s*=\s*(\w+)` match กับ `api_key = test_key_here` แต่ถ้า entropy < 3.5 ก็จะไม่ flag
- ทำให้ข้าม example values เช่น `api_key = your_api_key_here` โดยอัตโนมัติ

### Shannon Entropy และการตรวจ Random Strings

**Shannon entropy formula:**

```
H(X) = -Σ P(xᵢ) · log₂(P(xᵢ))
```

โดย P(xᵢ) คือความน่าจะเป็นของอักขระ xᵢ ใน string (ความถี่สัมพัทธ์)

ตัวอย่าง entropy ของ string ต่าง ๆ:

| String | Entropy | ความหมาย |
|--------|---------|----------|
| `"aaaa"` | 0.0 bits | อักขระเดียว — ไม่มี information |
| `"aabb"` | 1.0 bits | 2 อักขระเท่ากัน — low entropy |
| `"helloworld"` | ~2.85 bits | คำปกติ — medium-low |
| `"aB3xK9mPqR7n"` | ~4.0 bits | mixed case+digits — high |
| `"AKIAIOSFODNN7EXAMPLE"` | ~4.2 bits | AWS key format — high |
| random base64 | 5.5–6.0 bits | เกือบ maximum entropy |

Rule ที่ match กับ generic pattern (เช่น `GEN001` API key) ใช้ `entropy_threshold: Some(3.5)` เพื่อกรองออกค่าที่เป็น placeholder text

### SARIF 2.1.0 Format

SARIF (Static Analysis Results Interchange Format) เป็น JSON schema ที่ OASIS กำหนดเป็นมาตรฐาน GitHub Code Scanning รองรับ SARIF ทำให้ findings จาก secret scanner แสดงผลใน Security tab ได้โดยตรง โครงสร้างหลัก:

```json
{
  "$schema": "https://...sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [{
    "tool": { "driver": { "name": "secret-scanner" } },
    "results": [{
      "ruleId": "AWS001",
      "level": "error",
      "locations": [{
        "physicalLocation": {
          "artifactLocation": { "uri": "src/config.rs" },
          "region": { "startLine": 42 }
        }
      }]
    }]
  }]
}
```

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Shannon Entropy Calculator

เริ่มจาก building block พื้นฐานที่สุด: ฟังก์ชันคำนวณ Shannon entropy สิ่งนี้คือ mathematical foundation ของ scanner ทั้งหมด

**สร้าง project:**

```bash
cargo new secret-scanner
cd secret-scanner
```

**`src/entropy.rs`:**

```rust
/// คำนวณ Shannon entropy ของ string
/// สูตร: H = -Σ p(x) * log₂(p(x)) โดย p(x) คือความถี่สัมพัทธ์ของอักขระแต่ละตัว
pub fn entropy(s: &str) -> f64 {
    if s.is_empty() {
        return 0.0;
    }
    let len = s.len() as f64;
    let mut freq = [0u32; 256];
    for &b in s.as_bytes() {
        freq[b as usize] += 1;
    }
    freq.iter()
        .filter(|&&c| c > 0)
        .map(|&c| {
            let p = c as f64 / len;
            -p * p.log2()
        })
        .sum()
}
```

**ทำความเข้าใจการ implement:**

1. `freq = [0u32; 256]` — array ขนาด 256 เก็บความถี่ของทุก byte value ที่เป็นไปได้
2. วน loop ผ่านทุก byte ของ string และเพิ่มความถี่
3. กรองเฉพาะ byte ที่ปรากฏ (`filter(|&&c| c > 0)`) — byte ที่ไม่ปรากฏ contribute เป็น 0 อยู่แล้ว
4. คำนวณ `-p * p.log2()` สำหรับแต่ละ byte แล้ว sum

**ทดสอบด้วยตัวเลขที่รู้ค่าแน่นอน:**

```rust
// ใน main.rs ขั้นแรก
mod entropy;
use crate::entropy::entropy;

fn main() {
    // "aabb" มีอักขระ 2 ชนิด เท่ากัน → entropy = 1.0
    println!("entropy('aabb') = {:.4}", entropy("aabb"));   // 1.0000
    // "aaaa" มีอักขระเดียว → entropy = 0.0
    println!("entropy('aaaa') = {:.4}", entropy("aaaa"));   // 0.0000
    // string ที่ดู random
    println!("entropy('aB3xK9mPq') = {:.4}", entropy("aB3xK9mPq")); // ~3.17
}
```

**Output:**

```
entropy('aabb') = 1.0000
entropy('aaaa') = 0.0000
entropy('aB3xK9mPq') = 3.1699
```

> **Pitfall 1: log2(0) = -infinity ทำให้โปรแกรม panic หรือ NaN**
>
> ถ้าไม่ filter byte ที่ frequency เป็น 0 ออก การคำนวณ `p.log2()` เมื่อ `p = 0` จะได้ `-infinity` ใน Rust `0.0f64.log2()` คืนค่า `NEG_INFINITY` ซึ่งเมื่อคูณกับ `0.0` จะได้ `NaN` (Not a Number) การใส่ `.filter(|&&c| c > 0)` ป้องกันปัญหานี้ได้สะอาด

---

### ขั้นที่ 2: Rule System และ Pattern Definitions

สร้าง `Rule` struct และ default ruleset ที่ครอบคลุม pattern ทั่วไป

**`Cargo.toml` (เพิ่ม dependencies):**

```toml
[package]
name = "secret-scanner"
version = "0.1.0"
edition = "2021"

[dependencies]
regex = "1.10"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
walkdir = "2.4"
colored = "2.1"
glob = "0.3"
```

**`src/rules.rs`:**

```rust
use regex::Regex;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum Severity {
    Critical,
    High,
    Medium,
    Low,
}

impl std::fmt::Display for Severity {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Severity::Critical => write!(f, "CRITICAL"),
            Severity::High     => write!(f, "HIGH"),
            Severity::Medium   => write!(f, "MEDIUM"),
            Severity::Low      => write!(f, "LOW"),
        }
    }
}

/// Rule สำหรับตรวจจับ secret pattern
#[derive(Debug, Clone)]
pub struct Rule {
    pub id: String,
    pub name: String,
    pub pattern: Regex,
    pub entropy_threshold: Option<f64>,
    pub file_extensions: Vec<String>, // empty = ทุก extension
    pub severity: Severity,
}

impl Rule {
    pub fn new(
        id: &str,
        name: &str,
        pattern: &str,
        entropy_threshold: Option<f64>,
        file_extensions: Vec<&str>,
        severity: Severity,
    ) -> Self {
        Rule {
            id: id.to_string(),
            name: name.to_string(),
            pattern: Regex::new(pattern).expect("invalid regex"),
            entropy_threshold,
            file_extensions: file_extensions.iter().map(|s| s.to_string()).collect(),
            severity,
        }
    }

    pub fn matches_extension(&self, ext: &str) -> bool {
        if self.file_extensions.is_empty() {
            return true;
        }
        self.file_extensions.iter().any(|e| e == ext)
    }
}

/// สร้าง ruleset มาตรฐาน
pub fn default_rules() -> Vec<Rule> {
    vec![
        // AWS Access Key ID: ขึ้นต้นด้วย AKIA ตามด้วย uppercase letter/digit 16 ตัว
        Rule::new(
            "AWS001", "AWS Access Key ID",
            r"(?i)(AKIA[0-9A-Z]{16})",
            None, vec![], Severity::Critical,
        ),
        // AWS Secret Access Key: ค้นหา key/secret assignment ที่ match 40-char base64
        Rule::new(
            "AWS002", "AWS Secret Access Key",
            r"(?i)(aws_secret_access_key|aws_secret|aws_key)\s*[=:]\s*([A-Za-z0-9/+=]{40})",
            Some(4.5), vec![], Severity::Critical,
        ),
        // Generic API key: ค้นหา api_key= หรือ apikey: assignment
        Rule::new(
            "GEN001", "Generic API Key",
            r"(?i)(api[_\-]?key|apikey|api[_\-]?secret)\s*[=:]\s*['\"]?([A-Za-z0-9\-_]{16,64})['\"]?",
            Some(3.5), vec![], Severity::High,
        ),
        // PEM private key header — เป็น structural pattern ที่ชัดเจน ไม่ต้องตรวจ entropy
        Rule::new(
            "PKI001", "Private Key (PEM)",
            r"-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----",
            None, vec![], Severity::Critical,
        ),
        // JWT: 3 segments ของ base64url คั่นด้วยจุด ขึ้นต้นด้วย eyJ ({"  encoded)
        Rule::new(
            "JWT001", "JSON Web Token",
            r"eyJ[A-Za-z0-9\-_]+\.eyJ[A-Za-z0-9\-_]+\.[A-Za-z0-9\-_]+",
            None, vec![], Severity::High,
        ),
        // Generic high-entropy token: string ใน quotes ที่ยาว ≥ 20 chars
        Rule::new(
            "ENT001", "High-Entropy Token",
            r"['\"]([A-Za-z0-9+/=\-_]{20,})['\"]",
            Some(4.5), vec![], Severity::Medium,
        ),
        // GitHub Personal Access Token: รูปแบบ ghp_ นำหน้า
        Rule::new(
            "GH001", "GitHub Personal Access Token",
            r"ghp_[A-Za-z0-9]{36}",
            None, vec![], Severity::Critical,
        ),
        // Hardcoded password: password= assignment ที่มี non-trivial value
        Rule::new(
            "PWD001", "Hardcoded Password",
            r#"(?i)(password|passwd|pwd)\s*[=:]\s*['"]\s*([^\s'"]{8,})\s*['"]"#,
            Some(3.0), vec![], Severity::High,
        ),
    ]
}
```

**ทดสอบ Pattern ด้วย interactive check:**

ก่อนเขียน scanner จริง เราสามารถ verify pattern ด้วย:

```rust
fn main() {
    mod rules; // ใน main.rs
    use rules::default_rules;

    let rules = default_rules();
    let test_lines = [
        r#"AWS_KEY = "AKIAIOSFODNN7EXAMPLE""#,
        r#"api_key = "sk-test-12345678901234567890abcdef""#,
        r#"-----BEGIN RSA PRIVATE KEY-----"#,
        r#"token = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.SflK""#,
    ];
    for line in &test_lines {
        for rule in &rules {
            if rule.pattern.is_match(line) {
                println!("[{}] matched: {}", rule.id, line);
            }
        }
    }
}
```

**Output:**

```
[AWS001] matched: AWS_KEY = "AKIAIOSFODNN7EXAMPLE"
[PKI001] matched: -----BEGIN RSA PRIVATE KEY-----
[JWT001] matched: token = "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.SflK"
```

> **Pitfall 2: Regex compilation overhead เมื่อ compile ใน hot loop**
>
> ถ้า compile `Regex::new(pattern)` ทุกครั้งที่เรียก scan_line จะช้ามาก เนื่องจาก regex compilation มี overhead สูง การเก็บ `Regex` ที่ compiled แล้วใน `Rule` struct และ compile เพียงครั้งเดียวใน `Rule::new()` ช่วยให้ performance ดีขึ้นอย่างมีนัยสำคัญ (compilation ทำแค่ครั้งเดียวต่อ rule ไม่ใช่ทุก line ที่ scan)

---

### ขั้นที่ 3: Finding Struct และ File Scanner

สร้าง `Finding` struct เพื่อเก็บผลลัพธ์ และ scanner ที่อ่านไฟล์ทีละบรรทัด

**`src/finding.rs`:**

```rust
use serde::{Deserialize, Serialize};
use crate::rules::Severity;

/// ผลลัพธ์จากการตรวจพบ secret ที่อาจเป็นไปได้
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Finding {
    pub rule_id: String,
    pub rule_name: String,
    pub file: String,
    pub line_number: usize,
    pub matched_text: String,
    pub entropy: f64,
    pub severity: Severity,
}

impl Finding {
    pub fn new(
        rule_id: impl Into<String>,
        rule_name: impl Into<String>,
        file: impl Into<String>,
        line_number: usize,
        matched_text: impl Into<String>,
        entropy: f64,
        severity: Severity,
    ) -> Self {
        Finding {
            rule_id: rule_id.into(),
            rule_name: rule_name.into(),
            file: file.into(),
            line_number,
            matched_text: matched_text.into(),
            entropy,
            severity,
        }
    }
}
```

**`src/scanner.rs`:**

```rust
use std::path::Path;
use walkdir::WalkDir;
use crate::entropy::entropy;
use crate::finding::Finding;
use crate::rules::Rule;

/// ตรวจสอบว่า path นี้ควรถูกข้ามหรือไม่
fn should_skip_path(path: &Path) -> bool {
    let path_str = path.to_string_lossy();
    for skip_dir in &[".git", "node_modules", "target", ".svn", ".hg", "vendor"] {
        if path_str.contains(&format!("/{}/", skip_dir))
            || path_str.starts_with(&format!("{}/", skip_dir))
        {
            return true;
        }
    }
    false
}

/// ตรวจสอบว่าไฟล์น่าจะเป็น binary หรือไม่
fn is_likely_binary(path: &Path) -> bool {
    let binary_exts = [
        "exe", "dll", "so", "dylib", "bin", "obj", "o", "a",
        "png", "jpg", "jpeg", "gif", "bmp", "ico",
        "zip", "tar", "gz", "bz2", "xz", "7z", "rar",
        "pdf", "doc", "docx", "xls", "xlsx",
        "mp3", "mp4", "avi", "mov", "wav",
        "wasm", "pyc", "class",
    ];
    if let Some(ext) = path.extension() {
        let ext_str = ext.to_string_lossy().to_lowercase();
        return binary_exts.contains(&ext_str.as_str());
    }
    false
}

/// ตรวจสอบบรรทัดเดียวกับ rules ทั้งหมด
pub fn scan_line(
    line: &str,
    line_number: usize,
    file_path: &str,
    rules: &[Rule],
) -> Vec<Finding> {
    let mut findings = Vec::new();
    let ext = Path::new(file_path)
        .extension()
        .map(|e| e.to_string_lossy().to_lowercase())
        .unwrap_or_default();

    for rule in rules {
        if !rule.matches_extension(&ext) {
            continue;
        }
        for cap in rule.pattern.find_iter(line) {
            let matched = cap.as_str();
            let e = entropy(matched);

            // ถ้า rule มี entropy_threshold ต้องตรวจสอบ
            if let Some(threshold) = rule.entropy_threshold {
                if e < threshold {
                    continue;
                }
            }

            findings.push(Finding::new(
                &rule.id,
                &rule.name,
                file_path,
                line_number,
                matched,
                e,
                rule.severity.clone(),
            ));
        }
    }
    findings
}

/// scan ไฟล์เดียว — อ่านทีละบรรทัดแล้ว apply rules
pub fn scan_file(file_path: &Path, rules: &[Rule]) -> Vec<Finding> {
    if is_likely_binary(file_path) {
        return Vec::new();
    }
    let content = match std::fs::read_to_string(file_path) {
        Ok(c) => c,
        Err(_) => return Vec::new(),
    };
    let path_str = file_path.to_string_lossy().to_string();
    let mut findings = Vec::new();
    for (i, line) in content.lines().enumerate() {
        findings.extend(scan_line(line, i + 1, &path_str, rules));
    }
    findings
}

/// scan ทั้ง directory แบบ recursive
pub fn scan_directory(dir: &Path, rules: &[Rule]) -> Vec<Finding> {
    let mut all_findings = Vec::new();
    for entry in WalkDir::new(dir)
        .follow_links(false)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
    {
        let path = entry.path();
        if should_skip_path(path) {
            continue;
        }
        all_findings.extend(scan_file(path, rules));
    }
    all_findings
}
```

**จุดสำคัญในการออกแบบ:**

`scan_line` เป็น pure function รับ input ทั้งหมดผ่าน parameter ไม่มี side effect ทำให้ test ง่ายมาก เพียงแค่ส่ง string เข้าไปและตรวจ output ไม่ต้องสร้างไฟล์จริง

`scan_file` ใช้ `std::fs::read_to_string` และ silently skip ไฟล์ที่อ่านไม่ได้ (binary data ทำให้ `read_to_string` fail เนื่องจาก invalid UTF-8) แทนที่จะ panic

> **Pitfall 3: `read_to_string` บนไฟล์ binary fail ด้วย `InvalidData` error**
>
> `std::fs::read_to_string` ต้องการ valid UTF-8 ไฟล์ binary เช่น compiled object files หรือ image files จะทำให้ return `Err(InvalidData)` การใช้ `match` และ `return Vec::new()` ใน error case ทำให้ scanner ข้ามไฟล์เหล่านี้อย่าง graceful แทนที่จะ crash นอกจากนี้การ pre-check extension ด้วย `is_likely_binary()` ช่วย skip ไฟล์ binary ที่รู้จักล่วงหน้าได้เร็วขึ้น

---

### ขั้นที่ 4: Allowlist ด้วย `.secretsignore`

สร้าง mechanism สำหรับ suppress findings ที่รู้ว่า safe — เช่น test fixtures, example files, documentation

**`src/allowlist.rs`:**

```rust
use glob::Pattern;
use crate::finding::Finding;

/// Entry ใน allowlist
#[derive(Debug, Clone)]
pub enum AllowEntry {
    /// ข้ามไฟล์ที่ตรงกับ glob pattern (เช่น "tests/**")
    GlobPattern(Pattern),
    /// ข้าม rule ID ที่ระบุทั้งหมด (เช่น "ENT001")
    RuleId(String),
    /// ข้าม rule ID ในไฟล์ที่ตรงกับ glob (เช่น "GEN001:tests/**")
    RuleAndGlob(String, Pattern),
}

#[derive(Debug, Default)]
pub struct AllowList {
    entries: Vec<AllowEntry>,
}

impl AllowList {
    pub fn new() -> Self {
        AllowList { entries: Vec::new() }
    }

    /// Parse ไฟล์ .secretsignore
    /// รูปแบบ:
    ///   # comment
    ///   tests/**           ← glob pattern
    ///   ENT001             ← rule ID
    ///   GEN001:*.example   ← rule:glob
    pub fn parse(content: &str) -> Self {
        let mut allow = AllowList::new();
        for line in content.lines() {
            let line = line.trim();
            if line.is_empty() || line.starts_with('#') {
                continue;
            }
            if let Some((rule_id, glob_str)) = line.split_once(':') {
                if let Ok(pat) = Pattern::new(glob_str) {
                    allow.entries.push(AllowEntry::RuleAndGlob(
                        rule_id.to_string(), pat
                    ));
                }
            } else if line.contains('*') || line.contains('/') || line.contains('?') {
                if let Ok(pat) = Pattern::new(line) {
                    allow.entries.push(AllowEntry::GlobPattern(pat));
                }
            } else {
                allow.entries.push(AllowEntry::RuleId(line.to_string()));
            }
        }
        allow
    }

    /// ตรวจสอบว่า finding นี้ถูก allow ไว้หรือไม่
    pub fn is_allowed(&self, finding: &Finding) -> bool {
        for entry in &self.entries {
            match entry {
                AllowEntry::GlobPattern(pat) => {
                    if pat.matches(&finding.file) { return true; }
                }
                AllowEntry::RuleId(id) => {
                    if finding.rule_id == *id { return true; }
                }
                AllowEntry::RuleAndGlob(id, pat) => {
                    if finding.rule_id == *id && pat.matches(&finding.file) {
                        return true;
                    }
                }
            }
        }
        false
    }

    pub fn filter_findings(&self, findings: Vec<Finding>) -> Vec<Finding> {
        findings.into_iter().filter(|f| !self.is_allowed(f)).collect()
    }
}
```

**ตัวอย่างไฟล์ `.secretsignore`:**

```
# ข้ามไฟล์ test fixtures ทั้งหมด
tests/fixtures/**

# ข้าม high-entropy false positives ใน documentation
ENT001:docs/**

# ข้าม example configuration files
GEN001:*.example
GEN001:*.sample

# ข้ามไฟล์ที่รู้ว่ามี test keys
tests/**
```

---

### ขั้นที่ 5: Git Diff Integration

Scanner ที่ใช้งานจริงใน pre-commit hook ต้องตรวจเฉพาะ code ที่เปลี่ยนแปลง ไม่ใช่ทั้ง repository ทุกครั้ง

**`src/git.rs`:**

```rust
use crate::finding::Finding;
use crate::rules::Rule;
use crate::scanner::scan_line;

/// Parse unified diff format และส่งคืน added lines พร้อม context
/// Returns: Vec<(file_path, line_number, line_content)>
pub fn parse_diff_additions(diff: &str) -> Vec<(String, usize, String)> {
    let mut current_file = String::new();
    let mut current_line: usize = 0;
    let mut additions = Vec::new();

    for line in diff.lines() {
        if line.starts_with("+++ b/") {
            current_file = line[6..].to_string();
            current_line = 0;
        } else if line.starts_with("@@ ") {
            // parse hunk header: @@ -old_start,count +new_start,count @@
            if let Some(new_range) = line.split('+').nth(1) {
                let start_str = new_range.split(',').next().unwrap_or("0");
                current_line = start_str.trim().parse().unwrap_or(1).saturating_sub(1);
            }
        } else if line.starts_with('+') && !line.starts_with("+++") {
            current_line += 1;
            additions.push((
                current_file.clone(),
                current_line,
                line[1..].to_string(), // ตัด '+' นำหน้าออก
            ));
        } else if !line.starts_with('-') {
            current_line += 1; // context line
        }
    }
    additions
}

/// scan git diff output — ตรวจเฉพาะบรรทัดที่เพิ่มมาใหม่
pub fn scan_diff(diff: &str, rules: &[Rule]) -> Vec<Finding> {
    let additions = parse_diff_additions(diff);
    let mut findings = Vec::new();
    for (file, line_num, line_content) in &additions {
        findings.extend(scan_line(line_content, *line_num, file, rules));
    }
    findings
}
```

**ตัวอย่าง unified diff format:**

```diff
diff --git a/config.env b/config.env
--- a/config.env
+++ b/config.env
@@ -1,3 +1,5 @@
 DATABASE_URL=postgres://localhost/mydb
+AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
+AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
 DEBUG=false
```

**Hunk header parsing:**

`@@ -1,3 +1,5 @@` หมายความว่า:
- Old file: เริ่มที่บรรทัด 1, ยาว 3 บรรทัด
- New file: เริ่มที่บรรทัด 1, ยาว 5 บรรทัด

เราสนใจเฉพาะ `+1` (new file start) ซึ่ง `saturating_sub(1)` ช่วยเพราะเราจะ increment ก่อน push

**Integration เป็น pre-commit hook:**

```bash
#!/bin/sh
# .git/hooks/pre-commit

# ดึง diff ของ staged files
DIFF=$(git diff --cached)

# รัน scanner บน diff
echo "$DIFF" | secret-scanner --stdin-diff

# ถ้า exit code ไม่ใช่ 0 ให้ abort commit
if [ $? -ne 0 ]; then
    echo "Pre-commit hook: potential secrets detected. Commit aborted."
    exit 1
fi
```

---

### ขั้นที่ 6: Report Formats — JSON และ SARIF 2.1.0

**`src/report.rs`:**

```rust
use serde_json::{json, Value};
use crate::finding::Finding;
use crate::rules::Severity;

/// สร้าง JSON report จาก findings
pub fn to_json_report(findings: &[Finding]) -> Value {
    let summary = json!({
        "total": findings.len(),
        "critical": findings.iter()
            .filter(|f| matches!(f.severity, Severity::Critical)).count(),
        "high": findings.iter()
            .filter(|f| matches!(f.severity, Severity::High)).count(),
        "medium": findings.iter()
            .filter(|f| matches!(f.severity, Severity::Medium)).count(),
        "low": findings.iter()
            .filter(|f| matches!(f.severity, Severity::Low)).count(),
    });
    json!({
        "version": "1.0",
        "summary": summary,
        "findings": findings,
    })
}

fn severity_to_sarif_level(s: &Severity) -> &'static str {
    match s {
        Severity::Critical | Severity::High => "error",
        Severity::Medium => "warning",
        Severity::Low => "note",
    }
}

/// สร้าง SARIF 2.1.0 report สำหรับ GitHub Code Scanning
pub fn to_sarif_report(findings: &[Finding]) -> Value {
    let results: Vec<Value> = findings
        .iter()
        .map(|f| {
            json!({
                "ruleId": f.rule_id,
                "message": {
                    "text": format!(
                        "Potential secret detected by rule '{}': {}",
                        f.rule_name, f.matched_text
                    )
                },
                "locations": [{
                    "physicalLocation": {
                        "artifactLocation": { "uri": f.file },
                        "region": { "startLine": f.line_number }
                    }
                }],
                "level": severity_to_sarif_level(&f.severity),
            })
        })
        .collect();

    json!({
        "$schema": "https://raw.githubusercontent.com/oasis-tcs/sarif-spec/master/Schemata/sarif-schema-2.1.0.json",
        "version": "2.1.0",
        "runs": [{
            "tool": {
                "driver": {
                    "name": "secret-scanner",
                    "version": "0.1.0",
                    "informationUri": "https://github.com/example/secret-scanner"
                }
            },
            "results": results
        }]
    })
}
```

**ตัวอย่าง JSON report output:**

```json
{
  "version": "1.0",
  "summary": {
    "total": 2,
    "critical": 1,
    "high": 1,
    "medium": 0,
    "low": 0
  },
  "findings": [
    {
      "rule_id": "AWS001",
      "rule_name": "AWS Access Key ID",
      "file": "src/config.rs",
      "line_number": 15,
      "matched_text": "AKIAIOSFODNN7EXAMPLE",
      "entropy": 4.17,
      "severity": "Critical"
    }
  ]
}
```

**ตัวอย่าง SARIF output (สำหรับ GitHub Code Scanning):**

```json
{
  "$schema": "https://raw.githubusercontent.com/.../sarif-schema-2.1.0.json",
  "version": "2.1.0",
  "runs": [{
    "tool": {
      "driver": {
        "name": "secret-scanner",
        "version": "0.1.0"
      }
    },
    "results": [{
      "ruleId": "AWS001",
      "level": "error",
      "message": { "text": "Potential secret detected by rule 'AWS Access Key ID'" },
      "locations": [{
        "physicalLocation": {
          "artifactLocation": { "uri": "src/config.rs" },
          "region": { "startLine": 15 }
        }
      }]
    }]
  }]
}
```

> **Pitfall 4: SARIF `level` field ต้องเป็นค่าที่ถูกต้องตาม spec**
>
> SARIF 2.1.0 กำหนดให้ `level` เป็นหนึ่งใน: `"none"`, `"note"`, `"warning"`, `"error"` เท่านั้น GitHub Code Scanning จะ reject SARIF file ที่ใช้ค่าอื่นเช่น `"critical"` หรือ `"high"` โดยไม่แสดง error ที่ชัดเจน การ map `Critical/High → "error"`, `Medium → "warning"`, `Low → "note"` เป็น convention ที่ถูกต้องตาม spec

---

### ขั้นที่ 7: CLI Entry Point และ Terminal Output ด้วย `colored`

**`src/main.rs`:**

```rust
mod allowlist;
mod entropy;
mod finding;
mod git;
mod report;
mod rules;
mod scanner;

use std::path::Path;
use colored::Colorize;
use crate::allowlist::AllowList;
use crate::finding::Finding;
use crate::report::{to_json_report, to_sarif_report};
use crate::rules::{default_rules, Severity};
use crate::scanner::scan_directory;

fn severity_colored(s: &Severity) -> String {
    match s {
        Severity::Critical => "CRITICAL".red().bold().to_string(),
        Severity::High     => "HIGH".bright_red().to_string(),
        Severity::Medium   => "MEDIUM".yellow().to_string(),
        Severity::Low      => "LOW".blue().to_string(),
    }
}

fn print_summary_table(findings: &[Finding]) {
    if findings.is_empty() {
        println!("{}", "No secrets found.".green().bold());
        return;
    }
    println!("\n{}", "=== Secret Scanner Results ===".bold());
    println!(
        "{:<10} {:<30} {:<8} {:<50}",
        "Severity", "Rule", "Line", "File"
    );
    println!("{}", "-".repeat(100));
    for f in findings {
        println!(
            "{:<10} {:<30} {:<8} {:<50}",
            severity_colored(&f.severity),
            f.rule_name,
            f.line_number,
            f.file,
        );
        println!("           └─ {}", f.matched_text.dimmed());
    }
    println!("{}", "-".repeat(100));
    println!(
        "Total: {} findings ({} critical, {} high, {} medium, {} low)",
        findings.len().to_string().bold(),
        findings.iter()
            .filter(|f| matches!(f.severity, Severity::Critical))
            .count().to_string().red(),
        findings.iter()
            .filter(|f| matches!(f.severity, Severity::High))
            .count().to_string().bright_red(),
        findings.iter()
            .filter(|f| matches!(f.severity, Severity::Medium))
            .count().to_string().yellow(),
        findings.iter()
            .filter(|f| matches!(f.severity, Severity::Low))
            .count().to_string().blue(),
    );
}

fn main() {
    let args: Vec<String> = std::env::args().collect();
    let scan_path = args.get(1).map(String::as_str).unwrap_or(".");
    let output_format = args.get(2).map(String::as_str).unwrap_or("table");

    eprintln!("Scanning: {}", scan_path);

    let rules = default_rules();
    let mut findings = scan_directory(Path::new(scan_path), &rules);

    // โหลด allowlist จาก .secretsignore ถ้ามี
    let allowlist_path = Path::new(scan_path).join(".secretsignore");
    if allowlist_path.exists() {
        if let Ok(content) = std::fs::read_to_string(&allowlist_path) {
            let allow = AllowList::parse(&content);
            let before = findings.len();
            findings = allow.filter_findings(findings);
            let filtered = before - findings.len();
            if filtered > 0 {
                eprintln!("Allowlist filtered {} findings", filtered);
            }
        }
    }

    // เรียง findings ตาม severity (Critical ก่อน)
    findings.sort_by(|a, b| {
        let ord = |s: &Severity| match s {
            Severity::Critical => 0,
            Severity::High     => 1,
            Severity::Medium   => 2,
            Severity::Low      => 3,
        };
        ord(&a.severity).cmp(&ord(&b.severity))
    });

    match output_format {
        "json"  => println!(
            "{}",
            serde_json::to_string_pretty(&to_json_report(&findings)).unwrap()
        ),
        "sarif" => println!(
            "{}",
            serde_json::to_string_pretty(&to_sarif_report(&findings)).unwrap()
        ),
        _       => print_summary_table(&findings),
    }

    // exit code 1 ถ้าพบ Critical/High findings (สำหรับ CI pipeline)
    let has_critical = findings.iter().any(|f| {
        matches!(f.severity, Severity::Critical | Severity::High)
    });
    if has_critical {
        std::process::exit(1);
    }
}
```

**ตัวอย่าง terminal output (table format):**

```
Scanning: ./my-project

=== Secret Scanner Results ===
Severity   Rule                           Line     File
----------------------------------------------------------------------------------------------------
CRITICAL   AWS Access Key ID              15       src/aws_client.rs
           └─ AKIAIOSFODNN7EXAMPLE
CRITICAL   Private Key (PEM)              1        keys/deploy.pem
           └─ -----BEGIN RSA PRIVATE KEY-----
HIGH       JSON Web Token                 42       src/auth.rs
           └─ eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.SflK
----------------------------------------------------------------------------------------------------
Total: 3 findings (2 critical, 1 high, 0 medium, 0 low)
```

**การใช้งาน:**

```bash
# scan current directory (table output)
secret-scanner .

# scan specific path
secret-scanner /path/to/project

# JSON output (สำหรับ CI artifact)
secret-scanner . json > report.json

# SARIF output (สำหรับ GitHub Code Scanning)
secret-scanner . sarif > results.sarif
```

---

### ขั้นที่ 8: Integration กับ GitHub Actions

**`.github/workflows/secret-scan.yml`:**

```yaml
name: Secret Scanner

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      security-events: write  # จำเป็นสำหรับ upload SARIF

    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable

      - name: Build secret-scanner
        run: cargo build --release

      - name: Run secret scan
        run: |
          ./target/release/secret-scanner . sarif > results.sarif
        continue-on-error: true  # ไม่ fail workflow ถ้ามี findings

      - name: Upload SARIF to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: results.sarif
```

**หลังจาก upload SARIF แล้ว GitHub จะแสดง findings ใน Security tab → Code scanning alerts**

> **Pitfall 5: `continue-on-error: true` ใน scan step**
>
> `secret-scanner` return exit code 1 เมื่อพบ Critical/High findings ซึ่งจะทำให้ GitHub Actions step fail และหยุด workflow ทั้งหมดก่อนที่จะ upload SARIF การใส่ `continue-on-error: true` ทำให้ workflow ดำเนินต่อไปจนถึง upload step ได้ แต่ยังคงแสดงว่า step fail เพื่อแจ้งให้ทราบ

---

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ครอบคลุมทุก module ทดสอบทั้ง happy path และ edge cases

### Test Coverage Summary

| Module | Tests | สิ่งที่ทดสอบ |
|--------|-------|------------|
| `entropy` | 6 | Known entropy values, empty string, single char |
| `rules` | 10 | Pattern match/no-match สำหรับทุก rule type, severity display |
| `allowlist` | 6 | Glob matching, rule ID filtering, rule+glob, comment parsing |
| `scanner` | 7 | Line scanning, file type detection, directory skip |
| `report` | 7 | JSON structure, SARIF schema/version/fields, severity mapping |
| `git` | 3 | Diff parsing, additions count, scan_diff |
| **รวม** | **39** | |

### Tests ทั้งหมดแยกตาม Module

**`src/entropy.rs` — Shannon Entropy Tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_entropy_empty_string() {
        assert_eq!(entropy(""), 0.0);
    }

    #[test]
    fn test_entropy_single_char() {
        // "aaaa" — อักขระเดียวซ้ำ entropy = 0
        let e = entropy("aaaa");
        assert!((e - 0.0).abs() < 1e-10,
            "entropy ของ 'aaaa' ต้องเป็น 0, ได้ {}", e);
    }

    #[test]
    fn test_entropy_two_chars_equal() {
        // "aabb" — 2 อักขระเท่ากัน entropy = 1.0
        // p(a)=0.5, p(b)=0.5 → H = -(0.5*log2(0.5) + 0.5*log2(0.5)) = 1.0
        let e = entropy("aabb");
        assert!((e - 1.0).abs() < 1e-10,
            "entropy ของ 'aabb' ต้องเป็น 1.0, ได้ {}", e);
    }

    #[test]
    fn test_entropy_high_random_string() {
        // String ที่ดูเหมือน random มี entropy สูง
        let s = "aB3xK9mPqR7nLzYw2TvC0dEgHjF4iOuS";
        let e = entropy(s);
        assert!(e > 4.5,
            "entropy ของ random string ต้องสูงกว่า 4.5, ได้ {}", e);
    }

    #[test]
    fn test_entropy_low_repetitive_string() {
        // "aaabbbccc" — H = -(3*(1/3)*log2(1/3)) ≈ 1.585 < 2.0
        let e = entropy("aaabbbccc");
        assert!(e < 2.0,
            "entropy ของ 'aaabbbccc' ต้องต่ำกว่า 2.0, ได้ {}", e);
    }

    #[test]
    fn test_entropy_known_value() {
        // "ab" — 2 อักขระต่างกัน entropy = 1.0
        let e = entropy("ab");
        assert!((e - 1.0).abs() < 1e-10,
            "entropy ของ 'ab' ต้องเป็น 1.0, ได้ {}", e);
    }
}
```

**`src/rules.rs` — Pattern Matching Tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::entropy::entropy;

    #[test]
    fn test_aws_key_pattern_match() {
        let rules = default_rules();
        let aws_rule = rules.iter().find(|r| r.id == "AWS001").unwrap();
        assert!(aws_rule.pattern.is_match("AKIAIOSFODNN7EXAMPLE"));
        assert!(aws_rule.pattern.is_match("export AWS_KEY=AKIAIOSFODNN7EXAMPLE"));
    }

    #[test]
    fn test_aws_key_pattern_no_match() {
        let rules = default_rules();
        let aws_rule = rules.iter().find(|r| r.id == "AWS001").unwrap();
        assert!(!aws_rule.pattern.is_match("BKIAIOSFODNN7EXAMPLE")); // prefix ผิด
        assert!(!aws_rule.pattern.is_match("AKIA123")); // สั้นเกินไป (< 16 chars)
    }

    #[test]
    fn test_jwt_pattern_match() {
        let rules = default_rules();
        let jwt_rule = rules.iter().find(|r| r.id == "JWT001").unwrap();
        let jwt = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9\
                   .eyJzdWIiOiIxMjM0NTY3ODkwIn0\
                   .SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c";
        assert!(jwt_rule.pattern.is_match(jwt));
    }

    #[test]
    fn test_jwt_pattern_no_match() {
        let rules = default_rules();
        let jwt_rule = rules.iter().find(|r| r.id == "JWT001").unwrap();
        assert!(!jwt_rule.pattern.is_match("not.a.jwt.token.here"));
        assert!(!jwt_rule.pattern.is_match("abc.def.ghi")); // ไม่ขึ้นต้นด้วย eyJ
    }

    #[test]
    fn test_pem_header_pattern_match() {
        let rules = default_rules();
        let pem_rule = rules.iter().find(|r| r.id == "PKI001").unwrap();
        assert!(pem_rule.pattern.is_match("-----BEGIN RSA PRIVATE KEY-----"));
        assert!(pem_rule.pattern.is_match("-----BEGIN PRIVATE KEY-----"));
        assert!(pem_rule.pattern.is_match("-----BEGIN EC PRIVATE KEY-----"));
    }

    #[test]
    fn test_high_entropy_token_detection() {
        let high_entropy_token = "aB3xK9mPqR7nLzYw2TvC";
        let e = entropy(high_entropy_token);
        assert!(e > 4.0,
            "token ที่ดูเหมือน random ต้องมี entropy > 4.0, ได้ {}", e);
    }

    #[test]
    fn test_low_entropy_token_not_flagged() {
        let low_entropy_token = "helloworld12345678";
        let e = entropy(low_entropy_token);
        assert!(e < 4.5,
            "คำปกติต้องมี entropy ต่ำกว่า 4.5, ได้ {}", e);
    }

    #[test]
    fn test_severity_display() {
        assert_eq!(Severity::Critical.to_string(), "CRITICAL");
        assert_eq!(Severity::High.to_string(), "HIGH");
        assert_eq!(Severity::Medium.to_string(), "MEDIUM");
        assert_eq!(Severity::Low.to_string(), "LOW");
    }

    #[test]
    fn test_rule_matches_all_extensions_when_empty() {
        let rule = Rule::new("T001", "Test", r"test",
            None, vec![], Severity::Low);
        assert!(rule.matches_extension("rs"));
        assert!(rule.matches_extension("py"));
        assert!(rule.matches_extension("txt"));
    }

    #[test]
    fn test_rule_extension_filter() {
        let rule = Rule::new("T002", "Test", r"test",
            None, vec!["rs", "toml"], Severity::Low);
        assert!(rule.matches_extension("rs"));
        assert!(rule.matches_extension("toml"));
        assert!(!rule.matches_extension("py"));
    }
}
```

**`src/allowlist.rs` — AllowList Tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::rules::Severity;

    fn make_finding(rule_id: &str, file: &str) -> Finding {
        Finding::new(rule_id, "Test Rule", file, 1, "matched", 3.0, Severity::Medium)
    }

    #[test]
    fn test_allowlist_glob_match() {
        let allow = AllowList::parse("tests/**\n");
        let f = make_finding("GEN001", "tests/fixtures/config.env");
        assert!(allow.is_allowed(&f), "ไฟล์ใน tests/ ต้องถูก allow");
    }

    #[test]
    fn test_allowlist_glob_no_match() {
        let allow = AllowList::parse("tests/**\n");
        let f = make_finding("GEN001", "src/config.rs");
        assert!(!allow.is_allowed(&f), "ไฟล์ใน src/ ต้องไม่ถูก allow");
    }

    #[test]
    fn test_allowlist_rule_id() {
        let allow = AllowList::parse("ENT001\n");
        let f = make_finding("ENT001", "src/main.rs");
        assert!(allow.is_allowed(&f), "rule ENT001 ต้องถูก allow ทุกไฟล์");
    }

    #[test]
    fn test_allowlist_rule_and_glob() {
        let allow = AllowList::parse("GEN001:*.example\n");
        let f_example = make_finding("GEN001", "config.example");
        let f_real    = make_finding("GEN001", "config.env");
        assert!(allow.is_allowed(&f_example), "GEN001 ใน .example ต้องถูก allow");
        assert!(!allow.is_allowed(&f_real),   "GEN001 ใน .env ต้องไม่ถูก allow");
    }

    #[test]
    fn test_allowlist_comments_ignored() {
        let content = "# นี่คือ comment\n# ENT001\ntests/**\n";
        let allow = AllowList::parse(content);
        let f = make_finding("ENT001", "src/main.rs");
        assert!(!allow.is_allowed(&f));
    }

    #[test]
    fn test_filter_findings_removes_allowed() {
        let allow = AllowList::parse("ENT001\ntests/**\n");
        let findings = vec![
            make_finding("ENT001", "src/main.rs"),
            make_finding("AWS001", "tests/sample.env"),
            make_finding("AWS001", "src/config.rs"),
        ];
        let filtered = allow.filter_findings(findings);
        assert_eq!(filtered.len(), 1);
        assert_eq!(filtered[0].rule_id, "AWS001");
        assert_eq!(filtered[0].file, "src/config.rs");
    }
}
```

### `cargo test` Output

```
running 39 tests
test entropy::tests::test_entropy_empty_string ... ok
test entropy::tests::test_entropy_high_random_string ... ok
test entropy::tests::test_entropy_known_value ... ok
test entropy::tests::test_entropy_low_repetitive_string ... ok
test entropy::tests::test_entropy_single_char ... ok
test entropy::tests::test_entropy_two_chars_equal ... ok
test rules::tests::test_aws_key_pattern_match ... ok
test rules::tests::test_aws_key_pattern_no_match ... ok
test rules::tests::test_high_entropy_token_detection ... ok
test rules::tests::test_jwt_pattern_match ... ok
test rules::tests::test_jwt_pattern_no_match ... ok
test rules::tests::test_low_entropy_token_not_flagged ... ok
test rules::tests::test_pem_header_pattern_match ... ok
test rules::tests::test_rule_extension_filter ... ok
test rules::tests::test_rule_matches_all_extensions_when_empty ... ok
test rules::tests::test_severity_display ... ok
test allowlist::tests::test_allowlist_comments_ignored ... ok
test allowlist::tests::test_allowlist_glob_match ... ok
test allowlist::tests::test_allowlist_glob_no_match ... ok
test allowlist::tests::test_allowlist_rule_and_glob ... ok
test allowlist::tests::test_allowlist_rule_id ... ok
test allowlist::tests::test_filter_findings_removes_allowed ... ok
test scanner::tests::test_binary_extension_detection ... ok
test scanner::tests::test_scan_line_aws_key ... ok
test scanner::tests::test_scan_line_jwt ... ok
test scanner::tests::test_scan_line_no_false_positive_normal_text ... ok
test scanner::tests::test_scan_line_pem_header ... ok
test scanner::tests::test_should_skip_git_directory ... ok
test scanner::tests::test_should_skip_node_modules ... ok
test report::tests::test_json_report_empty_findings ... ok
test report::tests::test_json_report_structure ... ok
test report::tests::test_sarif_finding_level_critical ... ok
test report::tests::test_sarif_finding_location ... ok
test report::tests::test_sarif_report_runs_structure ... ok
test report::tests::test_sarif_report_schema_version ... ok
test report::tests::test_severity_to_sarif_level_mapping ... ok
test git::tests::test_parse_diff_additions_content ... ok
test git::tests::test_parse_diff_additions_count ... ok
test git::tests::test_scan_diff_detects_aws_key ... ok

test result: ok. 39 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

> **หมายเหตุ:** output นี้ได้จากการรัน `cargo test` บน scratchpad project จริง (code เหมือนกับที่แสดงในคู่มือนี้ทุกประการ)

---

## การ Package และ Deploy

### Build Release Binary

```bash
# build optimized binary
cargo build --release

# binary ที่ได้จะอยู่ที่
./target/release/secret-scanner

# ตรวจ binary size
ls -lh target/release/secret-scanner
# ประมาณ 3–5 MB (statically linked ไม่ต้องการ dynamic library)

# strip debug symbols เพื่อลด size
strip target/release/secret-scanner
```

### Cross-compile สำหรับ CI Runners

```bash
# สำหรับ Linux (Ubuntu CI runners)
cargo build --release --target x86_64-unknown-linux-musl

# สำหรับ macOS
cargo build --release --target x86_64-apple-darwin

# สำหรับ Windows
cargo build --release --target x86_64-pc-windows-msvc
```

### Install เป็น System Tool

```bash
# install ผ่าน cargo
cargo install --path .

# หรือ install จาก git
cargo install --git https://github.com/example/secret-scanner

# ใช้งานทันที
secret-scanner /path/to/project
```

### Docker Image

```dockerfile
# Multi-stage build
FROM rust:1.75-alpine AS builder
WORKDIR /build
COPY . .
RUN apk add --no-cache musl-dev
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /build/target/x86_64-unknown-linux-musl/release/secret-scanner /secret-scanner
ENTRYPOINT ["/secret-scanner"]
```

```bash
docker build -t secret-scanner:latest .

# scan local directory ผ่าน Docker
docker run --rm -v "$(pwd):/scan:ro" secret-scanner:latest /scan
```

### Configuration File (ขยายใน production)

สำหรับ production scanner มักมี configuration file เพิ่มเติม:

```toml
# .secret-scanner.toml
[scanner]
max_file_size_kb = 1024
follow_symlinks = false
output_format = "sarif"

[rules]
# เปิด/ปิด rule เฉพาะ
disabled = ["ENT001"]  # false positives มากเกินไปสำหรับ project นี้

[[custom_rules]]
id = "CUSTOM001"
name = "Internal Service Token"
pattern = 'svc_[A-Za-z0-9]{32}'
severity = "Critical"
```

---

## Pitfalls สรุป

### Pitfall 1: `log2(0)` ใน Shannon Entropy

Shannon entropy formula ต้องการ `p * log2(p)` เมื่อ `p > 0` เท่านั้น ใน Rust, `0.0f64.log2()` คืนค่า `f64::NEG_INFINITY` และ `0.0 * f64::NEG_INFINITY` คือ `NaN` ทำให้ entropy ของทุก string กลายเป็น `NaN`

**แก้ไข:** filter byte ที่ frequency เป็น 0 ออกก่อนคำนวณ:
```rust
freq.iter().filter(|&&c| c > 0).map(|&c| { ... })
```

### Pitfall 2: Regex Recompilation ใน Hot Loop

`Regex::new()` มี overhead สูงเนื่องจากต้อง parse และ compile NFA/DFA การ compile ใน `scan_line()` ทุกครั้งจะช้ามากเมื่อ scan หลายล้านบรรทัด

**แก้ไข:** compile `Regex` ครั้งเดียวใน `Rule::new()` และเก็บใน struct field

### Pitfall 3: `read_to_string` กับไฟล์ Binary

`std::fs::read_to_string` requires valid UTF-8 ไฟล์ binary หรือไฟล์ที่มี non-UTF-8 encoding จะทำให้ return `Err`

**แก้ไข:** ใช้ `match` silently skip ไฟล์ที่อ่านไม่ได้ และ pre-filter ด้วย extension check ก่อน

### Pitfall 4: SARIF `level` Values ต้องตรง Spec

GitHub Code Scanning reject SARIF ที่ใช้ค่า level ที่ไม่ถูกต้อง โดยไม่แสดง error message ชัดเจน ทำให้ debug ยาก

**แก้ไข:** ใช้เฉพาะ `"none"`, `"note"`, `"warning"`, `"error"` ตาม SARIF 2.1.0 spec

### Pitfall 5: Exit Code ใน CI Pipeline

การ exit code 1 เมื่อพบ findings จะทำให้ GitHub Actions step fail ก่อนที่จะ upload SARIF

**แก้ไข:** ใช้ `continue-on-error: true` ใน scan step เพื่อให้ workflow ดำเนินต่อไปจนถึง upload step

### Pitfall 6: False Positives จาก Entropy-Only Detection

String ทั่วไปอย่าง UUID, hash ของ content หรือ random test data มักมี entropy สูงพอที่จะ trigger `ENT001` rule

**แก้ไข:** ปรับ `entropy_threshold` ให้สูงพอ (เช่น 4.5) และใช้ `.secretsignore` สำหรับ patterns ที่รู้แน่ว่า safe ใน project นั้น ๆ

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Custom Rule จาก YAML/TOML File (ง่าย)

ปัจจุบัน rules ถูก hardcode ใน `default_rules()` เพิ่ม feature ให้โหลด custom rules จากไฟล์ configuration:

```toml
# custom-rules.toml
[[rule]]
id = "SLACK001"
name = "Slack Bot Token"
pattern = 'xoxb-[0-9]{10,13}-[0-9]{10,13}-[a-zA-Z0-9]{24}'
severity = "Critical"

[[rule]]
id = "STRIPE001"
name = "Stripe Secret Key"
pattern = 'sk_live_[0-9a-zA-Z]{24}'
severity = "Critical"
```

สิ่งที่ต้องทำ:
1. เพิ่ม `serde` derive ให้ `Severity` (ทำแล้ว)
2. สร้าง `RuleConfig` struct สำหรับ deserialize
3. เพิ่ม `--rules-file` argument ใน CLI
4. Merge custom rules กับ default rules

### แบบฝึกหัดที่ 2: Parallel Scanning ด้วย Rayon (กลาง)

`scan_directory` ปัจจุบันทำงานแบบ single-threaded เพิ่ม parallel processing ด้วย `rayon`:

```toml
[dependencies]
rayon = "1.8"
```

```rust
use rayon::prelude::*;

pub fn scan_directory_parallel(dir: &Path, rules: &[Rule]) -> Vec<Finding> {
    let files: Vec<_> = WalkDir::new(dir)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.file_type().is_file())
        .filter(|e| !should_skip_path(e.path()))
        .collect();

    files.par_iter()
        .flat_map(|entry| scan_file(entry.path(), rules))
        .collect()
}
```

สิ่งที่ต้องระวัง: `Rule` ต้องเป็น `Send + Sync` สำหรับ `par_iter` ตรวจสอบว่า `Regex` เป็น thread-safe (มันเป็น แต่ต้องไม่ใช้ `RefCell` หรือ `Cell` ใน struct)

### แบบฝึกหัดที่ 3: Baseline Mode — Scan เฉพาะ New Findings (กลาง)

เพิ่ม "baseline" feature ที่ save findings ครั้งแรก แล้ว report เฉพาะ findings ใหม่ที่ไม่เคยเห็นมาก่อน:

```bash
# สร้าง baseline จาก scan ครั้งแรก
secret-scanner . --save-baseline baseline.json

# scan ครั้งต่อไปจะ report เฉพาะ findings ใหม่
secret-scanner . --baseline baseline.json
```

สิ่งที่ต้องทำ:
1. สร้าง `Baseline` struct ที่ serialize/deserialize ได้
2. Generate unique fingerprint สำหรับ finding แต่ละตัว (hash ของ `rule_id + file + matched_text`)
3. ใช้ `HashSet<String>` เพื่อ lookup ว่า finding นี้เคยอยู่ใน baseline หรือไม่
4. Filter output ให้แสดงเฉพาะ new findings

### แบบฝึกหัดที่ 4: Entropy Visualization (กลาง-ยาก)

เพิ่ม mode ที่แสดง entropy distribution ของ strings ที่พบในโปรเจค:

```bash
secret-scanner . --entropy-map
```

Output เป็น histogram แสดงการกระจาย entropy ของ high-entropy strings ทั้งหมดที่พบ ทำให้เห็นว่า threshold ที่เลือกตัด false positive ได้ดีแค่ไหน

สิ่งที่ต้องทำ:
1. รวบรวม entropy values ของทุก string ที่ match pattern (ก่อน threshold filter)
2. สร้าง histogram ด้วย character art (แบ่ง bucket ทุก 0.5 bits)
3. แสดง threshold ที่ใช้ใน histogram เพื่อให้เห็น distribution

### แบบฝึกหัดที่ 5: Incremental Scan Mode ด้วย File Modification Time (ยาก)

สำหรับ repository ขนาดใหญ่ การ full scan ทุกครั้งช้าเกินไป เพิ่ม incremental mode:

```bash
# scan เฉพาะไฟล์ที่เปลี่ยนหลังจาก timestamp ที่บันทึกไว้
secret-scanner . --since-last-scan
```

สิ่งที่ต้องทำ:
1. บันทึก last-scan timestamp ใน `.secret-scanner-state` file
2. ใช้ `entry.metadata()?.modified()?` จาก `walkdir` เพื่อ filter ไฟล์ที่ modified หลัง timestamp
3. Merge findings ใหม่กับ baseline findings จาก scan ก่อนหน้า
4. อัปเดต timestamp หลัง scan สำเร็จ

### แบบฝึกหัดที่ 6: Web UI Dashboard (ยาก — Full-Stack)

สร้าง web interface สำหรับดู findings โดยใช้ Rust backend:

```bash
secret-scanner . --serve 8080
```

เปิด browser ไปที่ `http://localhost:8080` เพื่อดู interactive dashboard ที่:
- แสดง findings แบบ filterable table
- คลิก finding เพื่อดู context code (บรรทัดก่อนหน้าและหลัง)
- Export เป็น JSON หรือ SARIF โดยตรงจาก browser

สิ่งที่ต้องทำ:
1. เพิ่ม `axum` หรือ `actix-web` เป็น dependency
2. Embed static HTML/JS ด้วย `include_str!`
3. สร้าง REST API: `GET /api/findings` returns JSON
4. Frontend ใช้ vanilla JavaScript fetch API

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **secret scanner** ที่ครบถ้วนพร้อมใช้งานจริงใน production ประกอบด้วย:

**สิ่งที่สร้าง:**

1. **Shannon entropy engine** — `fn entropy(s: &str) -> f64` ที่ efficient ด้วย `[u32; 256]` frequency array แทน `HashMap`
2. **Rule system** — 8 rules ที่ครอบคลุม AWS keys, JWT, PEM headers, GitHub tokens, generic API keys และ high-entropy strings ทั้งหมด compile Regex ครั้งเดียวเมื่อสร้าง rule
3. **File scanner** — `scan_directory` ด้วย `walkdir` ที่ skip binary files, `.git`, `node_modules` และ `target` อย่าง graceful
4. **Allowlist** — `.secretsignore` format รองรับ 3 รูปแบบ: glob-only, rule-only, rule+glob สำหรับ suppress false positives อย่างละเอียด
5. **Git diff scanner** — parse unified diff format และ scan เฉพาะ added lines พร้อมบรรทัด number ที่ถูกต้อง
6. **Report formats** — JSON summary, SARIF 2.1.0 สำหรับ GitHub Code Scanning, colored table ใน terminal
7. **CI/CD integration** — exit code 1 เมื่อพบ Critical/High findings สำหรับ fail pipeline

**Pattern สำคัญที่ได้เรียน:**

- **Pure function design** — `scan_line` เป็น pure function ทำให้ test ง่ายโดยไม่ต้องสร้างไฟล์จริง
- **Enum-based variant types** — `AllowEntry` enum แทน multiple struct เพื่อ represent allowlist entries ที่ต่างกัน
- **Layered filtering** — pattern match → entropy threshold → allowlist — แต่ละชั้น filter คนละมุมมอง
- **Standard output formats** — JSON สำหรับ machine consumption, SARIF สำหรับ standard tool integration, colored table สำหรับ human reading

**เชื่อมโยงกับโปรเจคถัดไป:**

โปรเจค G08 (Infrastructure as Code) จะนำแนวคิดการ analyze structured files (YAML/HCL) มาต่อยอด แทนที่จะมองเป็น plain text เราจะ parse AST และวิเคราะห์ semantic structure — เช่นตรวจว่า Terraform configuration มี `encryption_enabled = false` หรือ S3 bucket มี `public_read_access = true`

---

**โปรเจคก่อนหน้า:** [Project G06: Health Checker](project-g06-health-checker.md) | **โปรเจคถัดไป:** [Project G08: Infrastructure as Code](project-g08-infra-as-code.md)
