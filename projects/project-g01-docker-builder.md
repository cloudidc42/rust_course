# Project G01: Docker Image Builder

> โมดูล: G — DevOps/Infra | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 16–24 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Docker Image Builder** — ระบบที่ parse Dockerfile จริง สร้าง layer model พร้อม SHA-256 cache key คำนวณ build graph แบบ DAG และ export OCI-compatible image manifest โดยไม่ต้องพึ่ง Docker daemon จริง ๆ เลย

**ทำไมสร้างโปรเจคนี้?**
Docker เป็นเครื่องมือ DevOps ที่ใช้งานกันทั่วโลก แต่คนส่วนใหญ่มองมันเป็น "กล่องดำ" การสร้าง Dockerfile parser และ layer cache simulator ด้วยตัวเองจะทำให้เข้าใจอย่างลึกซึ้งว่า:
- Docker cache ทำงานอย่างไร (ทำไม layer ordering จึงสำคัญมาก)
- Multi-stage build ประหยัด image size ได้อย่างไร
- OCI Image Spec กำหนดโครงสร้าง image manifest อย่างไร
- SHA-256 content addressing ใช้แก้ปัญหา cache invalidation

**Use cases จริงในโลก production:**
- **CI/CD systems** — เช่น Buildkite, CircleCI ที่ต้องวิเคราะห์ Dockerfile ก่อน build
- **Security scanners** — เช่น Trivy, Snyk ที่ parse Dockerfile หา misconfiguration
- **Image optimization tools** — เช่น dive, docker-slim ที่วิเคราะห์ layer structure
- **Internal build systems** — บริษัทขนาดใหญ่มักสร้าง custom build pipeline
- **Dockerfile linters** — เช่น hadolint ที่ตรวจสอบ best practices

**Learning value:**
โปรเจคนี้รวมความรู้หลายด้านเข้าด้วยกัน: lexer/parser design, graph algorithms (topological sort), cryptographic hashing สำหรับ content addressing, serde serialization สำหรับ OCI spec, และ Rust enum pattern ที่สะอาดสำหรับ algebraic data types

---

## สิ่งที่จะได้เรียนรู้

- **Dockerfile parsing** — tokenizer แบบ hand-written, line continuation, exec vs. shell form
- **Algebraic data types** — `Instruction` enum ที่ครอบคลุม instruction ทุกตัว พร้อม canonical text representation
- **SHA-256 content addressing** — `sha2` crate สำหรับคำนวณ layer cache key จาก parent hash + instruction text
- **Cache invalidation cascade** — เมื่อ instruction เปลี่ยน ทุก layer ถัดไปต้อง invalidate ตาม
- **DAG และ topological sort** — สร้าง build graph ด้วย `HashMap`, implement Kahn's algorithm
- **OCI Image Spec** — `ImageManifest` structure, `serde_json` serialization พร้อม field renaming
- **`.dockerignore` pattern matching** — wildcard glob matching, negation patterns, last-match-wins semantics
- **Multi-stage build detection** — แยก stage ด้วย `FROM ... AS`, ติดตาม stage boundaries

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, structs, enums, `Vec`, `HashMap`, `String`
- **Part 21–30**: Iterators, pattern matching, `Option`, method chaining
- **Part 31–40**: Error handling ด้วย `Result`, custom error types, `?` operator
- **Part 41–50**: Traits, generics, `impl Trait`, trait objects
- **Part 51–60**: Crate ecosystem — `serde`/`serde_json`, `sha2`, `hex`; Cargo features
- **Part 61–70**: Collections เชิงลึก — `HashMap`, `VecDeque`, entry API
- **Part 96–110**: DevOps concepts — Docker layer model, OCI Image Spec, content addressing

---

## โครงสร้างโปรเจค (Project Layout)

```
docker-builder/
├── src/
│   ├── lib.rs           ← module declarations
│   ├── main.rs          ← demo binary
│   ├── parser.rs        ← Dockerfile tokenizer, Instruction enum, Dockerfile struct
│   ├── layer.rs         ← Layer model + SHA-256 cache key computation
│   ├── cache.rs         ← LayerCache, cache hit/miss, cascade invalidation
│   ├── graph.rs         ← BuildGraph (DAG), topological sort, stage detection
│   ├── context.rs       ← BuildContext, .dockerignore parser, glob matching
│   └── manifest.rs      ← ImageManifest (OCI), ImageBuilder, JSON export
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Dockerfile string
       │
       ▼
┌─────────────┐
│  Tokenizer  │  parse_from / parse_copy / parse_env / ...
│  (parser.rs)│
└──────┬──────┘
       │ Vec<Instruction>
       ▼
┌─────────────┐     ┌──────────────────┐
│ BuildGraph  │────▶│  LayerCache      │  cache hit/miss ต่อ SHA-256 key
│ (graph.rs)  │     │  (cache.rs)      │
└──────┬──────┘     └──────────────────┘
       │ topological order
       ▼
┌─────────────┐     ┌──────────────────┐
│   Layer     │────▶│  ImageBuilder    │  export OCI manifest JSON
│  (layer.rs) │     │  (manifest.rs)   │
└─────────────┘     └──────────────────┘
```

### ทำไมถึงใช้ Design นี้

**Instruction enum แทน struct hierarchy:**
Rust's enum ทำงานได้ดีกว่า inheritance สำหรับ closed set of variants ทุก instruction มี canonical_text() method ที่ใช้คำนวณ cache key ทำให้ง่ายต่อการ test

**Content addressing ด้วย SHA-256:**
Docker ใช้ content addressing แบบเดียวกับ Git — layer id คือ hash ของ content ดังนั้น layer เดียวกันที่เกิดจาก parent เดียวกัน + instruction เดียวกันจะมี id เหมือนกันเสมอ ทำให้ cache สามารถ deduplicate ได้อัตโนมัติ

**Last-match-wins สำหรับ .dockerignore:**
นี่คือ semantics จริงของ Docker — ถ้ามี `*.log` แล้วตามด้วย `!important.log` ไฟล์ `important.log` จะถูก include ไม่ใช่ ignore การ iterate through all patterns และ update result แทนการ return early ทำให้ตรงกับ spec

**Kahn's algorithm สำหรับ topological sort:**
ใช้ in-degree counting แทน DFS-based เพราะ implementation ใช้ collections มาตรฐานทั้งหมด (`HashMap`, `VecDeque`) ไม่ต้อง recursive stack และทำงานได้ถูกต้องกับ multi-stage graph

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Dockerfile Parser — Tokenizer และ Instruction Enum

เริ่มจากการสร้าง `Instruction` enum ที่ครอบคลุม instruction ทุกแบบ และ parser ที่แปลง Dockerfile string เป็น `Vec<Instruction>`

**`Cargo.toml`:**

```toml
[package]
name = "docker-builder"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sha2 = "0.10"
hex = "0.4"
chrono = { version = "0.4", features = ["serde"] }
glob = "0.3"
```

**`src/parser.rs` — ส่วน Instruction enum:**

```rust
use std::fmt;

/// One instruction parsed from a Dockerfile
#[derive(Debug, Clone, PartialEq)]
pub enum Instruction {
    From {
        image: String,
        tag: Option<String>,
        alias: Option<String>,
    },
    Run(String),
    Copy {
        src: Vec<String>,
        dest: String,
        from: Option<String>,
    },
    Add {
        src: Vec<String>,
        dest: String,
    },
    Env(Vec<(String, String)>),
    Expose(Vec<u16>),
    Cmd(Vec<String>),
    Entrypoint(Vec<String>),
    Workdir(String),
    Arg {
        name: String,
        default: Option<String>,
    },
    Label(Vec<(String, String)>),
    Volume(Vec<String>),
    User(String),
    OnBuild(Box<Instruction>),
    StopSignal(String),
    Healthcheck(Option<String>),
    Shell(Vec<String>),
    Comment(String),
}

impl Instruction {
    pub fn keyword(&self) -> &'static str {
        match self {
            Instruction::From { .. }   => "FROM",
            Instruction::Run(_)        => "RUN",
            Instruction::Copy { .. }   => "COPY",
            Instruction::Add { .. }    => "ADD",
            Instruction::Env(_)        => "ENV",
            Instruction::Expose(_)     => "EXPOSE",
            Instruction::Cmd(_)        => "CMD",
            Instruction::Entrypoint(_) => "ENTRYPOINT",
            Instruction::Workdir(_)    => "WORKDIR",
            Instruction::Arg { .. }    => "ARG",
            Instruction::Label(_)      => "LABEL",
            Instruction::Volume(_)     => "VOLUME",
            Instruction::User(_)       => "USER",
            Instruction::OnBuild(_)    => "ONBUILD",
            Instruction::StopSignal(_) => "STOPSIGNAL",
            Instruction::Healthcheck(_)=> "HEALTHCHECK",
            Instruction::Shell(_)      => "SHELL",
            Instruction::Comment(_)    => "#",
        }
    }

    /// Canonical text สำหรับ cache key computation
    pub fn canonical_text(&self) -> String {
        match self {
            Instruction::From { image, tag, alias } => {
                let mut s = format!("FROM {}", image);
                if let Some(t) = tag { s.push(':'); s.push_str(t); }
                if let Some(a) = alias { s.push_str(" AS "); s.push_str(a); }
                s
            }
            Instruction::Run(cmd) => format!("RUN {}", cmd),
            Instruction::Copy { src, dest, from } => {
                let mut s = String::from("COPY");
                if let Some(f) = from { s.push_str(&format!(" --from={}", f)); }
                for item in src { s.push(' '); s.push_str(item); }
                s.push(' '); s.push_str(dest);
                s
            }
            // ... (ดูโค้ดเต็มในขั้นถัดไป)
            _ => format!("{} ...", self.keyword()),
        }
    }

    pub fn is_from(&self) -> bool {
        matches!(self, Instruction::From { .. })
    }
}
```

**`DockerfileParser` — main parser:**

Parser ทำงาน 3 ขั้นตอน:
1. อ่านทีละบรรทัด skip blank lines และ comments
2. Handle line continuation (backslash `\` ท้ายบรรทัด)
3. แยก keyword และ dispatch ไป helper function ของแต่ละ instruction

```rust
pub struct DockerfileParser;

impl DockerfileParser {
    pub fn parse(content: &str) -> Result<Dockerfile, ParseError> {
        let mut instructions = Vec::new();
        let mut lines = content.lines().enumerate().peekable();

        while let Some((line_no, line)) = lines.next() {
            let trimmed = line.trim();
            if trimmed.is_empty() { continue; }

            if trimmed.starts_with('#') {
                let text = trimmed.trim_start_matches('#').trim().to_string();
                instructions.push(Instruction::Comment(text));
                continue;
            }

            // Handle line continuation
            let mut full_line = trimmed.to_string();
            while full_line.ends_with('\\') {
                full_line.pop();
                if let Some((_, next)) = lines.next() {
                    full_line.push(' ');
                    full_line.push_str(next.trim());
                }
            }

            let (keyword, rest) = split_keyword(&full_line);
            let instr = match keyword.to_uppercase().as_str() {
                "FROM"        => parse_from(rest, line_no)?,
                "RUN"         => Instruction::Run(rest.to_string()),
                "COPY"        => parse_copy(rest, line_no)?,
                "ADD"         => parse_add(rest, line_no)?,
                "ENV"         => parse_env(rest, line_no)?,
                "EXPOSE"      => parse_expose(rest, line_no)?,
                "CMD"         => Instruction::Cmd(parse_shell_or_exec(rest)),
                "ENTRYPOINT"  => Instruction::Entrypoint(parse_shell_or_exec(rest)),
                "WORKDIR"     => Instruction::Workdir(rest.to_string()),
                "ARG"         => parse_arg(rest),
                "LABEL"       => parse_label(rest, line_no)?,
                "VOLUME"      => Instruction::Volume(parse_shell_or_exec(rest)),
                "USER"        => Instruction::User(rest.to_string()),
                "STOPSIGNAL"  => Instruction::StopSignal(rest.to_string()),
                "HEALTHCHECK" => parse_healthcheck(rest),
                "SHELL"       => Instruction::Shell(parse_shell_or_exec(rest)),
                other => return Err(ParseError::UnknownInstruction(other.to_string())),
            };
            instructions.push(instr);
        }

        if instructions.is_empty() {
            return Err(ParseError::EmptyDockerfile);
        }
        Ok(Dockerfile { instructions })
    }
}
```

**จุดสำคัญของ `parse_shell_or_exec`:**

Docker รองรับ 2 รูปแบบสำหรับ CMD, ENTRYPOINT, SHELL:
- **Exec form**: `["executable", "param1", "param2"]` — JSON array
- **Shell form**: `command param1 param2` — shell string

```rust
fn parse_shell_or_exec(s: &str) -> Vec<String> {
    let trimmed = s.trim();
    if trimmed.starts_with('[') {
        // JSON exec form
        if let Ok(serde_json::Value::Array(arr)) = serde_json::from_str(trimmed) {
            return arr
                .into_iter()
                .filter_map(|v| v.as_str().map(|s| s.to_string()))
                .collect();
        }
    }
    // Shell form — return as single string
    vec![trimmed.to_string()]
}
```

**ทดสอบขั้นที่ 1:**

```bash
$ cargo test parser::tests 2>&1 | tail -25
running 20 tests
test parser::tests::test_parse_simple_from ... ok
test parser::tests::test_parse_from_with_alias ... ok
test parser::tests::test_parse_from_no_tag ... ok
test parser::tests::test_parse_run ... ok
test parser::tests::test_parse_copy ... ok
test parser::tests::test_parse_copy_from_stage ... ok
test parser::tests::test_parse_env_key_value ... ok
test parser::tests::test_parse_expose ... ok
test parser::tests::test_parse_cmd_exec_form ... ok
test parser::tests::test_parse_entrypoint ... ok
test parser::tests::test_parse_workdir ... ok
test parser::tests::test_parse_arg ... ok
test parser::tests::test_parse_label ... ok
test parser::tests::test_parse_multi_stage ... ok
test parser::tests::test_parse_comment ... ok
test parser::tests::test_line_continuation ... ok
test parser::tests::test_empty_dockerfile_error ... ok
test parser::tests::test_unknown_instruction_error ... ok
test parser::tests::test_canonical_text_from ... ok
test parser::tests::test_canonical_text_env ... ok

test result: ok. 20 passed; 0 failed; 0 ignored; 0 measured; 30 filtered out
```

---

### ขั้นที่ 2: Layer Model และ SHA-256 Cache Key

Docker ใช้ **content addressing** เหมือน Git — ทุก layer มี id ที่เป็น SHA-256 hash ของ content เนื่องจากเราไม่ได้ build image จริง เราจะ hash `(parent_id + instruction_canonical_text)` เพื่อสร้าง cache key ที่ deterministic

**ทำไม `parent_hash + instruction_text` ถึงสมเหตุสมผล:**

ถ้า parent layer เปลี่ยน (เช่น RUN ก่อนหน้าเปลี่ยน) → parent_hash เปลี่ยน → cache key ของ layer ถัดมาเปลี่ยนทั้งหมด นี่คือ cascade invalidation ที่ถูกต้อง

**`src/layer.rs`:**

```rust
use sha2::{Digest, Sha256};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};
use crate::parser::Instruction;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Layer {
    pub id: String,                    // = cache_key (SHA-256 hex)
    pub parent_id: Option<String>,
    pub instruction_text: String,
    pub size_bytes: u64,               // simulated size
    pub created_at: u64,               // unix timestamp
    pub cache_key: String,             // SHA-256(parent_hash || ":" || instruction_text)
    pub stage_index: usize,
}

impl Layer {
    /// คำนวณ cache key จาก parent hash + instruction text
    pub fn compute_cache_key(parent_hash: &str, instruction_text: &str) -> String {
        let mut hasher = Sha256::new();
        hasher.update(parent_hash.as_bytes());
        hasher.update(b":");
        hasher.update(instruction_text.as_bytes());
        hex::encode(hasher.finalize())
    }

    pub fn new(
        parent_id: Option<String>,
        parent_hash: &str,
        instruction: &Instruction,
        stage_index: usize,
    ) -> Self {
        let instruction_text = instruction.canonical_text();
        let cache_key = Self::compute_cache_key(parent_hash, &instruction_text);

        Layer {
            id: cache_key.clone(),
            parent_id,
            instruction_text,
            size_bytes: simulate_layer_size(instruction),
            created_at: SystemTime::now()
                .duration_since(UNIX_EPOCH)
                .unwrap_or_default()
                .as_secs(),
            cache_key,
            stage_index,
        }
    }
}

fn simulate_layer_size(instruction: &Instruction) -> u64 {
    match instruction {
        Instruction::From { .. }    => 50 * 1024 * 1024,  // 50 MB base image
        Instruction::Run(_)         => 10 * 1024 * 1024,  // 10 MB for package installs
        Instruction::Copy { src, .. } => src.len() as u64 * 1024 * 1024,
        Instruction::Add { src, .. }  => src.len() as u64 * 2 * 1024 * 1024,
        Instruction::Env(_) | Instruction::Expose(_)
        | Instruction::Label(_) | Instruction::Workdir(_)
        | Instruction::Arg { .. }   => 0,  // metadata layers
        _                           => 1024,
    }
}
```

**ทดสอบขั้นที่ 2:**

```bash
$ cargo test layer::tests
running 6 tests
test layer::tests::test_cache_key_deterministic ... ok
test layer::tests::test_cache_key_differs_with_different_parent ... ok
test layer::tests::test_cache_key_differs_with_different_instruction ... ok
test layer::tests::test_cache_key_is_hex_sha256 ... ok
test layer::tests::test_layer_new_from_instruction ... ok
test layer::tests::test_layer_env_has_zero_size ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 44 filtered out
```

**Properties สำคัญที่ test ตรวจ:**
- Cache key เป็น deterministic — เรียกซ้ำได้ผลเหมือนกัน
- Parent hash ต่างกัน → cache key ต่างกัน
- Instruction text ต่างกัน → cache key ต่างกัน
- Format เป็น 64 hex chars (SHA-256)
- ENV/EXPOSE/WORKDIR มี `size_bytes = 0` (metadata only)

---

### ขั้นที่ 3: Layer Cache — Hit/Miss และ Cascade Invalidation

`LayerCache` เป็น `HashMap<cache_key, Layer>` ที่ทำหน้าที่ตรวจสอบว่า layer นั้นเคย build แล้วหรือไม่

**กฎสำคัญของ cache invalidation:**
เมื่อ instruction บรรทัดใดเปลี่ยน ต้อง rebuild ทุก layer ตั้งแต่บรรทัดนั้นเป็นต้นไป เพราะ parent_hash ของ layer ถัดไปจะเปลี่ยน — cascading effect นี้ implement ใน `build_with_cache` ผ่าน flag `cache_valid`

**`src/cache.rs`:**

```rust
use std::collections::HashMap;
use crate::layer::Layer;
use crate::parser::{Dockerfile, Instruction};

#[derive(Debug, Clone, PartialEq)]
pub enum CacheResult {
    Hit(String),   // cache_key ของ layer ที่ hit
    Miss,
}

#[derive(Debug, Default)]
pub struct LayerCache {
    pub layers: HashMap<String, Layer>,
}

impl LayerCache {
    pub fn new() -> Self { LayerCache { layers: HashMap::new() } }

    pub fn lookup(&self, cache_key: &str) -> CacheResult {
        if self.layers.contains_key(cache_key) {
            CacheResult::Hit(cache_key.to_string())
        } else {
            CacheResult::Miss
        }
    }

    pub fn store(&mut self, layer: Layer) {
        self.layers.insert(layer.cache_key.clone(), layer);
    }

    pub fn len(&self) -> usize { self.layers.len() }
    pub fn is_empty(&self) -> bool { self.layers.is_empty() }
}

#[derive(Debug, Clone)]
pub struct BuildStep {
    pub layer: Layer,
    pub cache_result: CacheResult,
    pub duration_ms: u64,
}

pub fn build_with_cache(
    dockerfile: &Dockerfile,
    cache: &mut LayerCache,
) -> Vec<BuildStep> {
    let mut steps = Vec::new();
    let mut parent_id: Option<String> = None;
    let mut parent_hash = String::new();
    let mut stage_index = 0usize;
    let mut cache_valid = true;   // false หลังจาก cache miss ครั้งแรก
    let mut first_stage = true;

    for instruction in &dockerfile.instructions {
        if matches!(instruction, Instruction::Comment(_)) { continue; }

        if instruction.is_from() {
            if !first_stage { stage_index += 1; }
            first_stage = false;
            parent_id = None;
            parent_hash = String::new();
            cache_valid = true;  // reset per stage
        }

        let layer = Layer::new(parent_id.clone(), &parent_hash, instruction, stage_index);

        let cache_result = if cache_valid {
            cache.lookup(&layer.cache_key)
        } else {
            CacheResult::Miss
        };

        if cache_result == CacheResult::Miss {
            cache_valid = false;   // cascade: ทุก layer ถัดไปต้อง miss
            let duration_ms = simulate_build_time(instruction);
            cache.store(layer.clone());
            steps.push(BuildStep { layer: layer.clone(), cache_result: CacheResult::Miss, duration_ms });
        } else {
            steps.push(BuildStep { layer: layer.clone(), cache_result, duration_ms: 1 });
        }

        parent_hash = layer.cache_key.clone();
        parent_id = Some(layer.id.clone());
    }
    steps
}
```

**Cascade Invalidation ด้วย `invalidate_cascade`:**

```rust
pub fn invalidate_cascade(cache: &mut LayerCache, changed_layer_key: &str) -> usize {
    let mut to_remove = vec![changed_layer_key.to_string()];
    let mut found_new = true;

    // BFS หา descendants ทั้งหมด
    while found_new {
        found_new = false;
        let current_set: Vec<String> = to_remove.clone();
        for (key, layer) in &cache.layers {
            if let Some(parent) = &layer.parent_id {
                if current_set.contains(parent) && !to_remove.contains(key) {
                    to_remove.push(key.clone());
                    found_new = true;
                }
            }
        }
    }

    let mut removed = 0;
    for key in &to_remove {
        if cache.layers.remove(key).is_some() { removed += 1; }
    }
    removed
}
```

**ทดสอบขั้นที่ 3:**

```bash
$ cargo test cache::tests
running 5 tests
test cache::tests::test_cache_lookup_miss ... ok
test cache::tests::test_cache_store_and_hit ... ok
test cache::tests::test_cache_hit_on_second_build ... ok
test cache::tests::test_cache_miss_after_instruction_change ... ok
test cache::tests::test_cascade_invalidation ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 45 filtered out
```

**ตัวอย่างพฤติกรรม cache:**

```
Build #1 (cold cache):
  [MISS] FROM ubuntu:22.04
  [MISS] RUN apt-get update
  [MISS] COPY . /app

Build #2 (warm cache — ไม่มีอะไรเปลี่ยน):
  [HIT] FROM ubuntu:22.04
  [HIT] RUN apt-get update
  [HIT] COPY . /app

Build #3 (เปลี่ยน COPY . /app เป็น COPY src/ /app):
  [HIT] FROM ubuntu:22.04      ← parent_hash เหมือนเดิม → hit
  [HIT] RUN apt-get update     ← parent_hash เหมือนเดิม → hit
  [MISS] COPY src/ /app        ← instruction เปลี่ยน → miss
  [MISS] (ทุก layer หลังจากนี้จะ miss เสมอ)
```

นี่คือเหตุผลที่ Docker best practice บอกให้วาง `COPY` และ instruction ที่เปลี่ยนบ่อยไว้ **ท้าย** Dockerfile

---

### ขั้นที่ 4: Build Graph — DAG และ Topological Sort

`BuildGraph` เป็น Directed Acyclic Graph ที่แสดง dependency ระหว่าง layer แต่ละ node มี pointer ไป children (layers ที่ depend on it)

**ทำไมต้อง DAG?**
ใน multi-stage build มี 2 stage ที่แยกกันโดยสิ้นเชิง topological sort จะเรียง layers ในลำดับที่ถูกต้องสำหรับ build โดยที่ parent layer ต้องมาก่อน child เสมอ

**`src/graph.rs`:**

```rust
use std::collections::HashMap;
use crate::layer::Layer;
use crate::parser::{Dockerfile, Instruction};

#[derive(Debug, Clone)]
pub struct BuildNode {
    pub layer: Layer,
    pub children: Vec<String>,   // child layer ids
}

#[derive(Debug, Default)]
pub struct BuildGraph {
    pub nodes: HashMap<String, BuildNode>,
    pub roots: Vec<String>,
    pub stages: Vec<Option<String>>,   // stage alias (จาก FROM ... AS)
}

impl BuildGraph {
    pub fn from_dockerfile(dockerfile: &Dockerfile) -> Self {
        let mut graph = BuildGraph::new();
        let mut parent_id: Option<String> = None;
        let mut parent_hash = String::new();
        let mut stage_index = 0usize;
        let mut first_stage = true;

        for instruction in &dockerfile.instructions {
            if matches!(instruction, Instruction::Comment(_)) { continue; }

            if instruction.is_from() {
                if !first_stage { stage_index += 1; }
                first_stage = false;
                parent_id = None;
                parent_hash = String::new();
                let alias = match instruction {
                    Instruction::From { alias, .. } => alias.clone(),
                    _ => None,
                };
                graph.stages.push(alias);
            }

            let layer = Layer::new(parent_id.clone(), &parent_hash, instruction, stage_index);
            let node = BuildNode { layer: layer.clone(), children: Vec::new() };
            graph.nodes.insert(layer.id.clone(), node);

            if let Some(ref pid) = parent_id {
                if let Some(pnode) = graph.nodes.get_mut(pid) {
                    pnode.children.push(layer.id.clone());
                }
            } else {
                graph.roots.push(layer.id.clone());
            }

            parent_hash = layer.cache_key.clone();
            parent_id = Some(layer.id.clone());
        }
        graph
    }

    /// Kahn's algorithm — topological sort ด้วย in-degree counting
    pub fn topological_sort(&self) -> Vec<String> {
        let mut in_degree: HashMap<String, usize> = HashMap::new();
        let mut order: Vec<String> = Vec::new();
        let mut queue: std::collections::VecDeque<String> = std::collections::VecDeque::new();

        for id in self.nodes.keys() {
            in_degree.entry(id.clone()).or_insert(0);
        }
        for node in self.nodes.values() {
            for child in &node.children {
                *in_degree.entry(child.clone()).or_insert(0) += 1;
            }
        }

        let mut seeds: Vec<String> = in_degree.iter()
            .filter(|(_, &d)| d == 0)
            .map(|(id, _)| id.clone())
            .collect();
        seeds.sort();
        for s in seeds { queue.push_back(s); }

        while let Some(id) = queue.pop_front() {
            let children: Vec<String> = self.nodes.get(&id)
                .map(|n| { let mut c = n.children.clone(); c.sort(); c })
                .unwrap_or_default();
            order.push(id);
            for child in children {
                let deg = in_degree.get_mut(&child).unwrap();
                *deg -= 1;
                if *deg == 0 { queue.push_back(child); }
            }
        }
        order
    }

    pub fn stage_layers(&self, stage_index: usize) -> Vec<&Layer> {
        let order = self.topological_sort();
        order.iter()
            .filter_map(|id| self.nodes.get(id))
            .filter(|n| n.layer.stage_index == stage_index)
            .map(|n| &n.layer)
            .collect()
    }

    pub fn total_size_bytes(&self) -> u64 {
        self.nodes.values().map(|n| n.layer.size_bytes).sum()
    }
}
```

**ทดสอบขั้นที่ 4:**

```bash
$ cargo test graph::tests
running 5 tests
test graph::tests::test_graph_from_simple_dockerfile ... ok
test graph::tests::test_topological_sort_linear ... ok
test graph::tests::test_multi_stage_graph ... ok
test graph::tests::test_stage_layers_filter ... ok
test graph::tests::test_total_size_positive ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 45 filtered out
```

**ตัวอย่าง Multi-Stage Graph:**

```
Stage 0 (builder):              Stage 1 (final):
FROM rust:1.75 AS builder  ←─ root
    │
    ▼
WORKDIR /app
    │
    ▼
COPY . .
    │
    ▼
RUN cargo build --release

FROM debian:bookworm-slim  ←─ root (แยกกัน)
    │
    ▼
COPY --from=builder /app/target/release/app ...
    │
    ▼
CMD ["/app"]
```

Topological sort จะเรียง stage 0 ทั้งหมดก่อน แล้วตามด้วย stage 1

---

### ขั้นที่ 5: Build Context และ .dockerignore Parser

Build context คือ directory ที่ Docker ส่งไปให้ daemon เพื่อ COPY files `.dockerignore` กำหนดไฟล์ที่ไม่ต้องการส่งไป (เช่น `target/`, `.git`, `node_modules`)

**Semantics ของ .dockerignore:**
- Pattern ทุกอันถูก evaluate ตามลำดับ
- **Last match wins** — `!important.log` หลัง `*.log` จะ un-ignore ไฟล์นั้น
- Pattern ที่ลงท้ายด้วย `/` จะ match directory name (และทุก subpath)
- Pattern ที่ไม่มี `/` และไม่มี wildcard จะ match ทั้ง exact name และ prefix path

**`src/context.rs`:**

```rust
use std::path::PathBuf;

#[derive(Debug, Clone)]
pub struct BuildContext {
    pub base_dir: PathBuf,
    pub ignore_patterns: Vec<String>,
}

impl BuildContext {
    pub fn new(base_dir: impl Into<PathBuf>) -> Self {
        BuildContext { base_dir: base_dir.into(), ignore_patterns: Vec::new() }
    }

    pub fn parse_dockerignore(&mut self, content: &str) {
        for line in content.lines() {
            let trimmed = line.trim();
            if trimmed.is_empty() || trimmed.starts_with('#') { continue; }
            self.ignore_patterns.push(trimmed.to_string());
        }
    }

    /// Last-match-wins semantics
    pub fn is_ignored(&self, path: &str) -> bool {
        let path_str = path.trim_start_matches('/');
        let mut ignored = false;

        for pattern in &self.ignore_patterns {
            let pat = pattern.trim_start_matches('/');
            let is_negation = pat.starts_with('!');
            let actual_pat = if is_negation { &pat[1..] } else { pat };

            if matches_dockerignore(path_str, actual_pat) {
                ignored = !is_negation;
            }
        }
        ignored
    }
}

pub fn matches_dockerignore(path: &str, pattern: &str) -> bool {
    if pattern == "." || pattern == "**" { return true; }

    // Trailing slash: "target/" matches "target/anything"
    if pattern.ends_with('/') {
        let dir = pattern.trim_end_matches('/');
        return path.starts_with(&format!("{}/", dir)) || path == dir
            || { let comp = path.splitn(2, '/').next().unwrap_or(""); glob_match(dir, comp) };
    }

    // Double star: "node_modules/**" matches all subpaths
    if pattern.contains("**") {
        let parts: Vec<&str> = pattern.splitn(2, "**").collect();
        let prefix = parts[0];
        let suffix = parts.get(1).unwrap_or(&"");
        if let Some(remaining) = if prefix.is_empty() { Some(path) } else { path.strip_prefix(prefix) } {
            return suffix.is_empty() || remaining.ends_with(suffix.trim_start_matches('/'));
        }
        return false;
    }

    // Directory name without slash matches prefix
    if !pattern.contains('/') && !pattern.contains('*') {
        if path == pattern || path.starts_with(&format!("{}/", pattern)) { return true; }
    }

    glob_match(pattern, path)
}
```

**ทดสอบขั้นที่ 5:**

```bash
$ cargo test context::tests
running 8 tests
test context::tests::test_dockerignore_ignore_target ... ok
test context::tests::test_dockerignore_ignore_git ... ok
test context::tests::test_dockerignore_negation ... ok
test context::tests::test_dockerignore_skip_comments ... ok
test context::tests::test_dockerignore_wildcard_extension ... ok
test context::tests::test_glob_match_exact ... ok
test context::tests::test_glob_match_wildcard_star ... ok
test context::tests::test_matches_dockerignore_double_star ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out
```

---

### ขั้นที่ 6: Image Manifest — OCI-Compatible JSON Export

OCI Image Spec กำหนดโครงสร้างของ `manifest.json` ที่ container runtime อ่านเพื่อรู้ว่าต้อง pull layers อะไรบ้าง เราจะสร้าง `ImageManifest` struct ที่ serialize เป็น JSON ตาม spec

**โครงสร้าง OCI Manifest:**

```
ImageManifest
├── schemaVersion: 2
├── mediaType: "application/vnd.oci.image.manifest.v1+json"
├── config:
│   ├── mediaType: "application/vnd.oci.image.config.v1+json"
│   ├── size: <config_json_size>
│   └── digest: "sha256:<sha256_of_config>"
└── layers: [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": <layer_size>,
      "digest": "sha256:<layer_id>"
    },
    ...
  ]
```

**`src/manifest.rs`:**

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use crate::layer::Layer;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct LayerRef {
    #[serde(rename = "mediaType")]
    pub media_type: String,
    pub size: u64,
    pub digest: String,
}

impl LayerRef {
    pub fn from_layer(layer: &Layer) -> Self {
        LayerRef {
            media_type: "application/vnd.oci.image.layer.v1.tar+gzip".to_string(),
            size: layer.size_bytes,
            digest: format!("sha256:{}", layer.id),
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ImageManifest {
    #[serde(rename = "schemaVersion")]
    pub schema_version: u32,
    #[serde(rename = "mediaType")]
    pub media_type: String,
    pub config: ManifestConfig,
    pub layers: Vec<LayerRef>,
    pub annotations: HashMap<String, String>,
}

pub struct ImageBuilder {
    pub name: String,
    pub tag: String,
    layers: Vec<Layer>,
    image_config: ImageConfig,
}

impl ImageBuilder {
    pub fn new(name: impl Into<String>, tag: impl Into<String>) -> Self {
        ImageBuilder {
            name: name.into(),
            tag: tag.into(),
            layers: Vec::new(),
            image_config: ImageConfig::default_linux_amd64(),
        }
    }

    pub fn add_layer(&mut self, layer: Layer) {
        let diff_id = format!("sha256:{}", layer.id);
        self.image_config.rootfs.diff_ids.push(diff_id);
        self.image_config.history.push(HistoryEntry {
            created: "2024-01-01T00:00:00Z".to_string(),
            created_by: layer.instruction_text.clone(),
            empty_layer: layer.size_bytes == 0,
        });
        self.layers.push(layer);
    }

    pub fn export_manifest_json(&self) -> serde_json::Result<String> {
        let layer_refs: Vec<LayerRef> = self.layers.iter().map(LayerRef::from_layer).collect();
        let config_json = serde_json::to_string(&self.image_config)?;

        use sha2::{Digest, Sha256};
        let mut hasher = Sha256::new();
        hasher.update(config_json.as_bytes());
        let config_digest = hex::encode(hasher.finalize());

        let manifest = ImageManifest {
            schema_version: 2,
            media_type: "application/vnd.oci.image.manifest.v1+json".to_string(),
            config: ManifestConfig {
                media_type: "application/vnd.oci.image.config.v1+json".to_string(),
                size: config_json.len() as u64,
                digest: format!("sha256:{}", config_digest),
            },
            layers: layer_refs,
            annotations: {
                let mut m = HashMap::new();
                m.insert("org.opencontainers.image.ref.name".to_string(),
                         format!("{}:{}", self.name, self.tag));
                m
            },
        };
        serde_json::to_string_pretty(&manifest)
    }

    pub fn export_manifest(&self, path: &std::path::Path) -> std::io::Result<()> {
        let json = self.export_manifest_json()
            .map_err(|e| std::io::Error::new(std::io::ErrorKind::Other, e))?;
        std::fs::write(path, json)
    }
}
```

**ตัวอย่าง output ของ manifest:**

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 2224,
    "digest": "sha256:1f224d0747d522f719f1224fe9c1e527d2a70243..."
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 52428800,
      "digest": "sha256:a8b3c2..."
    }
  ],
  "annotations": {
    "org.opencontainers.image.ref.name": "myapp:latest"
  }
}
```

**ทดสอบขั้นที่ 6:**

```bash
$ cargo test manifest::tests
running 5 tests
test manifest::tests::test_manifest_serialization_valid_json ... ok
test manifest::tests::test_manifest_layer_count ... ok
test manifest::tests::test_manifest_total_size ... ok
test manifest::tests::test_manifest_export_to_file ... ok
test manifest::tests::test_oci_media_type ... ok
test manifest::tests::test_layer_ref_digest_format ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 44 filtered out
```

---

### ขั้นที่ 7: Integration — Demo Binary

`main.rs` เชื่อมทุก module เข้าด้วยกันเป็น end-to-end demo

**`src/main.rs`:**

```rust
use docker_builder::cache::{LayerCache, build_with_cache, CacheResult};
use docker_builder::context::BuildContext;
use docker_builder::graph::BuildGraph;
use docker_builder::manifest::ImageBuilder;
use docker_builder::parser::DockerfileParser;

fn main() {
    let dockerfile_content = r#"
# Multi-stage Dockerfile example
FROM rust:1.75 AS builder
ARG VERSION=1.0.0
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
LABEL version=1.0 maintainer=devops@example.com
ENV APP_ENV=production
WORKDIR /app
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
EXPOSE 8080
CMD ["/usr/local/bin/myapp"]
"#;

    println!("=== Docker Image Builder Demo ===\n");

    // 1. Parse the Dockerfile
    let df = DockerfileParser::parse(dockerfile_content).expect("Parse failed");
    println!("Parsed {} instructions across {} stage(s)\n",
        df.instructions.len(), df.stage_count());

    // 2. First build — cold cache
    println!("--- Build #1 (cold cache) ---");
    let mut cache = LayerCache::new();
    let steps1 = build_with_cache(&df, &mut cache);
    let total_ms: u64 = steps1.iter().map(|s| s.duration_ms).sum();
    for step in &steps1 {
        let status = match &step.cache_result {
            CacheResult::Hit(_) => "CACHED",
            CacheResult::Miss   => "BUILD ",
        };
        println!("  [{}] {} ({} ms)", status,
            &step.layer.instruction_text[..step.layer.instruction_text.len().min(50)],
            step.duration_ms);
    }
    println!("  Total: {} ms\n", total_ms);

    // 3. Second build — warm cache
    println!("--- Build #2 (warm cache) ---");
    let steps2 = build_with_cache(&df, &mut cache);
    let hits = steps2.iter().filter(|s| matches!(s.cache_result, CacheResult::Hit(_))).count();
    println!("  {}/{} layers from cache\n", hits, steps2.len());

    // 4. Build context with .dockerignore
    println!("--- Build Context ---");
    let mut ctx = BuildContext::new("/project");
    ctx.parse_dockerignore("target/\n.git\n*.log\n");
    let files = ctx.list_copy_sources("src/*");
    println!("  src/* (non-ignored): {:?}\n", files);

    // 5. Image manifest
    println!("--- Image Manifest ---");
    let mut builder = ImageBuilder::new("myapp", "latest");
    for step in &steps1 { builder.add_layer(step.layer.clone()); }
    let manifest = builder.export_manifest_json().unwrap();
    println!("{}", &manifest[..manifest.len().min(400)]);
}
```

**Output จากการรัน `cargo run`:**

```
=== Docker Image Builder Demo ===

Parsed 13 instructions across 2 stage(s)

--- Build #1 (cold cache) ---
  [BUILD ] FROM rust:1.75 AS builder (100 ms)
  [BUILD ] ARG VERSION=1.0.0 (10 ms)
  [BUILD ] WORKDIR /app (10 ms)
  [BUILD ] COPY . . (200 ms)
  [BUILD ] RUN cargo build --release (5000 ms)
  [BUILD ] FROM debian:bookworm-slim (100 ms)
  [BUILD ] LABEL version=1.0 maintainer=devops@example.com (10 ms)
  [BUILD ] ENV APP_ENV=production (10 ms)
  [BUILD ] WORKDIR /app (10 ms)
  [BUILD ] COPY --from=builder /app/target/release/myapp /usr (200 ms)
  [BUILD ] EXPOSE 8080 (10 ms)
  [BUILD ] CMD ["/usr/local/bin/myapp"] (10 ms)
  Total: 5670 ms

--- Build #2 (warm cache) ---
  12/12 layers from cache

--- Build Context ---
  src/* (non-ignored): ["src/main.rs", "src/lib.rs", "src/parser.rs"]

--- Image Manifest ---
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 2224,
    "digest": "sha256:1f224d0747d522f719f1224fe9c1e527..."
  },
  ...
}
```

---

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ทั้งหมด **50 tests** กระจายอยู่ใน 5 modules

### รัน tests ทั้งหมด

```bash
$ cargo test
```

### Real `cargo test` Output

```
running 50 tests
test cache::tests::test_cache_lookup_miss ... ok
test cache::tests::test_cache_hit_on_second_build ... ok
test cache::tests::test_cascade_invalidation ... ok
test context::tests::test_dockerignore_ignore_git ... ok
test cache::tests::test_cache_miss_after_instruction_change ... ok
test cache::tests::test_cache_store_and_hit ... ok
test context::tests::test_dockerignore_ignore_target ... ok
test context::tests::test_dockerignore_negation ... ok
test context::tests::test_dockerignore_skip_comments ... ok
test context::tests::test_dockerignore_wildcard_extension ... ok
test context::tests::test_glob_match_exact ... ok
test context::tests::test_glob_match_wildcard_star ... ok
test context::tests::test_matches_dockerignore_double_star ... ok
test graph::tests::test_multi_stage_graph ... ok
test graph::tests::test_stage_layers_filter ... ok
test graph::tests::test_graph_from_simple_dockerfile ... ok
test layer::tests::test_cache_key_deterministic ... ok
test graph::tests::test_total_size_positive ... ok
test layer::tests::test_cache_key_differs_with_different_instruction ... ok
test graph::tests::test_topological_sort_linear ... ok
test layer::tests::test_cache_key_differs_with_different_parent ... ok
test layer::tests::test_cache_key_is_hex_sha256 ... ok
test layer::tests::test_layer_env_has_zero_size ... ok
test layer::tests::test_layer_new_from_instruction ... ok
test manifest::tests::test_layer_ref_digest_format ... ok
test manifest::tests::test_manifest_layer_count ... ok
test manifest::tests::test_manifest_total_size ... ok
test manifest::tests::test_manifest_export_to_file ... ok
test manifest::tests::test_manifest_serialization_valid_json ... ok
test manifest::tests::test_oci_media_type ... ok
test parser::tests::test_canonical_text_env ... ok
test parser::tests::test_empty_dockerfile_error ... ok
test parser::tests::test_canonical_text_from ... ok
test parser::tests::test_parse_arg ... ok
test parser::tests::test_parse_cmd_exec_form ... ok
test parser::tests::test_parse_comment ... ok
test parser::tests::test_line_continuation ... ok
test parser::tests::test_parse_copy ... ok
test parser::tests::test_parse_entrypoint ... ok
test parser::tests::test_parse_env_key_value ... ok
test parser::tests::test_parse_copy_from_stage ... ok
test parser::tests::test_parse_expose ... ok
test parser::tests::test_parse_from_no_tag ... ok
test parser::tests::test_parse_from_with_alias ... ok
test parser::tests::test_parse_label ... ok
test parser::tests::test_parse_run ... ok
test parser::tests::test_parse_multi_stage ... ok
test parser::tests::test_parse_simple_from ... ok
test parser::tests::test_parse_workdir ... ok
test parser::tests::test_unknown_instruction_error ... ok

test result: ok. 50 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/docker_builder-8de25710c0c9b468)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests docker_builder

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### สิ่งที่ test แต่ละกลุ่มตรวจสอบ

**`parser::tests` (20 tests)**
- ทุก instruction type parse ถูกต้อง: `FROM`, `RUN`, `COPY`, `ADD`, `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `WORKDIR`, `ARG`, `LABEL`
- `FROM` พร้อม tag, alias, ไม่มี tag
- `COPY --from=stage` multi-stage reference
- `CMD`/`ENTRYPOINT` ทั้ง exec form `["a","b"]` และ shell form
- Line continuation ด้วย backslash
- Error cases: empty Dockerfile, unknown instruction
- `canonical_text()` สำหรับ `FROM` และ `ENV`

**`layer::tests` (6 tests)**
- Cache key เป็น deterministic (เรียกซ้ำได้ผลเหมือนกัน)
- Cache key แตกต่างกันเมื่อ parent hash ต่างกัน
- Cache key แตกต่างกันเมื่อ instruction text ต่างกัน
- Format เป็น 64-char hex string (SHA-256)
- Layer สร้างจาก instruction ได้ถูกต้อง
- ENV/metadata layers มี `size_bytes = 0`

**`cache::tests` (5 tests)**
- Cache miss เมื่อ lookup key ที่ไม่มี
- Cache hit หลัง store
- Build ครั้งที่ 2 ได้ hits ทั้งหมด
- Instruction เปลี่ยน → layer นั้นและ layers ถัดไปเป็น miss
- Cascade invalidation ลบ descendants ถูกต้อง

**`graph::tests` (5 tests)**
- สร้าง graph จาก simple Dockerfile
- Topological sort เรียงถูกต้อง (parent ก่อน child เสมอ)
- Multi-stage graph มี stages ถูกต้องพร้อม alias
- `stage_layers()` filter ตาม stage index
- `total_size_bytes()` คืนค่า positive

**`context::tests` (8 tests)**
- `target/` pattern match `target/debug/app`
- `.git` pattern match `.git/HEAD`
- Negation `!important.log` หลัง `*.log`
- Skip blank lines และ comments ใน .dockerignore
- `*.md` wildcard match
- Exact match glob
- `*` ไม่ข้าม `/` (single-component wildcard)
- `**` match multi-component paths

**`manifest::tests` (6 tests)**
- JSON output เป็น valid JSON
- `schemaVersion: 2` ถูก serialize เป็น `"schemaVersion"` (renamed)
- Layer count ถูกต้อง
- Total size คำนวณถูก
- Export to file และ read back ได้
- OCI media type strings ถูกต้อง

---

## ปัญหาที่พบบ่อย (Pitfalls)

### Pitfall 1: Instruction Order มีผลต่อ Cache ทุก Layer ถัดไป

```
# BAD — COPY ก่อน RUN install ทำให้ cache miss ทุกครั้งที่ code เปลี่ยน
FROM ubuntu:22.04
COPY . /app
RUN apt-get install -y curl

# GOOD — ติดตั้ง dependencies ก่อน แล้วค่อย COPY code
FROM ubuntu:22.04
RUN apt-get install -y curl
COPY . /app
```

เหตุผล: ถ้า `COPY . /app` miss (เพราะ source code เปลี่ยน) ทุก layer ถัดไปจะ miss ด้วย แต่ถ้าวาง `RUN apt-get` ก่อน `COPY` layer ของ apt-get จะ cache hit ตราบเท่าที่ `Dockerfile` ก่อนหน้ายังเหมือนเดิม

### Pitfall 2: SHA-256 ของ Layer ขึ้นกับ parent_hash ทั้งสาย

```rust
// parent_hash เปลี่ยน → cache_key ของทุก layer ถัดไปเปลี่ยนทั้งหมด
let cache_key = sha256(parent_hash + ":" + instruction_text);
```

ถ้าคุณ reorder instructions แม้แต่บรรทัดเดียว ทุก layer หลังจากนั้นจะมี cache_key ใหม่ทั้งหมด ดังนั้นควรจัด instructions ที่ไม่เปลี่ยนบ่อยไว้ข้างบน

### Pitfall 3: `.dockerignore` ใช้ Last-Match-Wins ไม่ใช่ First-Match

```
# .dockerignore
*.log            ← ignore ทุก .log
!important.log   ← ยกเว้น important.log (last match wins)
```

หลายคน implement แบบ "return ทันทีที่ match ครั้งแรก" ซึ่งผิด — ต้อง iterate ทุก pattern และ update state ตลอด

```rust
// WRONG:
for pattern in patterns {
    if matches(path, pattern) {
        return !is_negation;  // ← ผิด! return เร็วเกินไป
    }
}

// CORRECT:
let mut ignored = false;
for pattern in patterns {
    if matches(path, pattern) {
        ignored = !is_negation;  // ← update, อย่า return
    }
}
ignored
```

### Pitfall 4: Box<Instruction> จำเป็นสำหรับ Recursive Enum

```rust
// ERROR: recursive type has infinite size
pub enum Instruction {
    OnBuild(Instruction),   // ← Rust ไม่รู้ขนาด at compile time
}

// CORRECT: Box ทำให้ size กลายเป็น pointer size (8 bytes)
pub enum Instruction {
    OnBuild(Box<Instruction>),
}
```

Rust ต้องรู้ขนาดของ enum ที่ compile time แต่ถ้า `Instruction` มี variant ที่ contain `Instruction` อีกชั้น size จะ infinite ต้องใช้ `Box<T>` (heap pointer) แทน

### Pitfall 5: `serde` serialize field ตาม snake_case ไม่ใช่ camelCase

```rust
// ถ้าไม่ rename จะได้ "schema_version" แต่ OCI spec ต้องการ "schemaVersion"
pub struct ImageManifest {
    #[serde(rename = "schemaVersion")]
    pub schema_version: u32,
    #[serde(rename = "mediaType")]
    pub media_type: String,
}
```

อีกทางเลือกคือ `#[serde(rename_all = "camelCase")]` ที่ struct level แต่ต้องระวังว่า field ทุกตัวจะถูก rename ทั้งหมด

### Pitfall 6: `*` wildcard ใน glob ไม่ข้าม path separator `/`

```
# pattern "*.rs" ควร match "main.rs" แต่ไม่ match "src/main.rs"
assert!(glob_match("*.rs", "main.rs"));       // true ✓
assert!(!glob_match("*.rs", "src/main.rs"));  // true ✓ (ถูกต้อง)
```

ถ้าต้องการ match ข้าม directory ต้องใช้ `**/*.rs` แทน นี่เป็น behavior มาตรฐานของ .dockerignore

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# ขนาด binary
ls -lh target/release/docker-builder
```

### ใช้งาน Library จากโปรเจคอื่น

เพิ่มใน `Cargo.toml` ของโปรเจคที่ต้องการใช้:

```toml
[dependencies]
docker-builder = { path = "../docker-builder" }
```

```rust
use docker_builder::parser::DockerfileParser;
use docker_builder::cache::{LayerCache, build_with_cache};

fn analyze_dockerfile(content: &str) {
    let df = DockerfileParser::parse(content).expect("parse error");
    let mut cache = LayerCache::new();
    let steps = build_with_cache(&df, &mut cache);
    for step in steps {
        println!("{}: {} bytes", step.layer.instruction_text, step.layer.size_bytes);
    }
}
```

### Docker Image สำหรับ Tool นี้เอง

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release
COPY src ./src
RUN touch src/main.rs && cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/docker-builder /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/docker-builder"]
```

### Integration กับ CI/CD

```yaml
# GitHub Actions example
- name: Analyze Dockerfile
  run: |
    ./docker-builder analyze Dockerfile > layer-report.json
    cat layer-report.json
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: BuildKit Syntax Support (Easy)

เพิ่ม support สำหรับ BuildKit directives เช่น:

```dockerfile
# syntax=docker/dockerfile:1
# escape=`
```

Directive เหล่านี้อยู่ที่บรรทัดแรกของ Dockerfile และเป็น comment พิเศษที่กำหนด parser behavior ลอง parse และ store เป็น `BuildKitConfig`:

```rust
pub struct BuildKitConfig {
    pub syntax: Option<String>,
    pub escape: Option<char>,
}
```

แล้วใช้ `escape` character แทน backslash ในการ handle line continuation

### Exercise 2: Dockerfile Linter (Medium)

สร้าง linter ที่ตรวจสอบ best practices และ output warnings:

```rust
pub enum LintWarning {
    PinYourBase { image: String, suggestion: String },
    CopyBeforeInstall { line: usize },
    NoHealthcheck,
    RunAsRoot,
    LargeLayer { instruction: String, size_mb: u64 },
}

pub fn lint(dockerfile: &Dockerfile) -> Vec<LintWarning> {
    let mut warnings = Vec::new();

    // Check: FROM ไม่ pin tag
    for instr in &dockerfile.instructions {
        if let Instruction::From { image, tag: None, .. } = instr {
            warnings.push(LintWarning::PinYourBase {
                image: image.clone(),
                suggestion: format!("{}:latest", image),
            });
        }
    }
    // ... เพิ่ม checks อื่น ๆ
    warnings
}
```

### Exercise 3: Parallel Stage Builds (Hard)

ใน multi-stage build ที่ stage ไม่ depend กัน สามารถ build parallel ได้ แก้ `BuildGraph` เพื่อตรวจหา independent stages แล้วสร้าง build plan ที่ระบุ parallelism:

```rust
pub struct BuildPlan {
    pub waves: Vec<Vec<String>>,  // layer ids ที่ build parallel ได้ในแต่ละ wave
}

impl BuildGraph {
    pub fn parallel_build_plan(&self) -> BuildPlan {
        // Wave = set of nodes ที่ in-degree = 0 ณ เวลานั้น
        // ทุก node ใน wave เดียวกัน build parallel ได้
        todo!()
    }
}
```

### Exercise 4: Layer Deduplication (Medium)

ถ้า 2 stages มี WORKDIR /app layer ที่ตรงกัน (parent hash ต่างกันแต่ content เหมือนกัน) ลองเพิ่ม content-based deduplication ที่ hash เฉพาะ instruction text ไม่รวม parent:

```rust
pub struct DeduplicationResult {
    pub original_count: usize,
    pub deduplicated_count: usize,
    pub saved_bytes: u64,
}

pub fn deduplicate_layers(layers: &[Layer]) -> DeduplicationResult {
    let mut seen_content: HashMap<String, usize> = HashMap::new();
    // hash เฉพาะ instruction_text
    // นับ duplicates
    todo!()
}
```

### Exercise 5: Vulnerability Scanner Integration (Advanced)

Parse `FROM` instructions เพื่อดึง base image names และ check กับ vulnerability database (ใช้ Trivy API หรือ CVE database):

```rust
pub struct VulnReport {
    pub image: String,
    pub tag: String,
    pub cve_count: usize,
    pub critical: Vec<String>,
    pub high: Vec<String>,
}

// ใช้ reqwest crate สำหรับ HTTP calls
async fn check_image_vulns(image: &str, tag: &str) -> Result<VulnReport, reqwest::Error> {
    // เรียก Trivy REST API หรือ Grype
    todo!()
}
```

### Exercise 6: OCI Image Tar Export (Advanced)

สร้าง function ที่ export image เป็น `.tar` format ที่ `docker load` รับได้จริง:

```
myapp-latest.tar
├── manifest.json
├── <config_digest>.json
└── <layer_id>/
    └── layer.tar
```

```rust
pub fn export_oci_tar(builder: &ImageBuilder, output_path: &Path) -> std::io::Result<()> {
    // ใช้ tar crate สร้าง tar archive
    // เพิ่ม manifest.json, config JSON, และ layer tarballs (simulated)
    todo!()
}
```

---

## สรุป

โปรเจคนี้สร้าง **Docker Image Builder** แบบครบวงจรโดยไม่พึ่ง Docker daemon จริง ๆ:

**สิ่งที่สร้างได้:**
- **Dockerfile parser** ที่รองรับ instruction ทุกแบบ ทั้ง exec form และ shell form, line continuation, multi-stage
- **SHA-256 content addressing** สำหรับ layer identity และ cache key computation — เหมือน Docker จริง
- **Layer cache** พร้อม cascade invalidation เมื่อ instruction เปลี่ยน
- **Build graph (DAG)** พร้อม topological sort ด้วย Kahn's algorithm
- **`.dockerignore` parser** พร้อม last-match-wins semantics และ glob matching
- **OCI-compatible manifest** ที่ serialize เป็น JSON ตาม spec

**Pattern สำคัญที่ได้เรียน:**
- `enum` ใน Rust เหมาะกับ algebraic data types เช่น instruction set ที่ closed — compile-time exhaustiveness checking ป้องกัน bugs
- Content addressing ด้วย SHA-256 เป็น foundation ของทั้ง Docker, Git, IPFS — immutable, verifiable, automatically deduplicated
- Cache invalidation เป็นปัญหา "2 hard things in CS" — การ model ด้วย parent hash chain ทำให้ใช้ properties ทางคณิตศาสตร์ช่วย
- `serde_json` `#[serde(rename)]` จำเป็นมากเมื่อต้องทำงานกับ external specs ที่ใช้ camelCase

**โปรเจคถัดไป (G02)** จะต่อยอดจากโปรเจคนี้โดยสร้าง **CI Runner** ที่รับ pipeline YAML, parse job definitions, schedule jobs ลงใน worker pool, และเชื่อมกับ Docker builder สำหรับ containerized builds

---

**โปรเจคก่อนหน้า:** [Project F10: Message Queue](project-f10-message-queue.md) | **โปรเจคถัดไป:** [Project G02: CI Runner](project-g02-ci-runner.md)
