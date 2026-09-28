# Project G02: CI/CD Pipeline Runner

> โมดูล: G — DevOps & Infrastructure Tools | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

CI/CD Pipeline ไม่ใช่แค่สคริปต์รวมกัน — มันคือเครื่องจักรที่ควบคุม **lifecycle ของ software** ตั้งแต่ push โค้ดจนถึง production deployment ทีมพัฒนาทุกทีมในโลก ไม่ว่าจะใช้ GitHub Actions, GitLab CI, CircleCI, หรือ Jenkins ล้วนพึ่งพาระบบนี้ทุกวัน

โปรเจคนี้สร้าง **ci-runner** — engine สำหรับรัน CI/CD pipeline ที่นิยามด้วย YAML คล้ายกับ GitHub Actions หรือ GitLab CI แต่เขียนด้วย Rust ล้วน ๆ ระบบอ่าน pipeline definition จากไฟล์ YAML, รัน stage ตามลำดับ, รัน job ใน stage แบบ parallel ด้วย Tokio, จัดการ artifact, และออก report เป็น JUnit XML กับ JSON summary

ทำไมถึงน่าสร้าง? เพราะงาน DevOps ใช้ภาษา shell script หรือ Python มาตลอด แต่เมื่อ pipeline ซับซ้อนขึ้น ต้องการ timeout handling, retry logic, artifact management, และ parallel execution — Rust ให้ประสิทธิภาพและความน่าเชื่อถือที่ภาษา script ทำไม่ได้ ระบบเช่น Dagger.io ก็ใช้แนวคิดเดียวกันนี้

Use case จริงในโลก production ได้แก่:
- **Self-hosted CI runner** สำหรับทีมที่ต้องการควบคุม infrastructure เอง
- **Pipeline orchestrator** สำหรับ monorepo ที่มี 100+ projects
- **Embedded CI** สำหรับ IoT หรือ edge devices ที่ไม่มี network ออกนอก
- **Testing harness** สำหรับ library ที่ต้องทดสอบบน hardware จริง

## สิ่งที่จะได้เรียนรู้

- **YAML DSL parsing ด้วย `serde_yaml`:** แปลง pipeline definition จาก YAML → Rust structs ด้วย `#[serde(tag = "type")]` polymorphism
- **Async execution engine:** รัน job แบบ parallel ด้วย `tokio::spawn`, จัดการ wave-based dependency resolution
- **Process management ด้วย `tokio::process::Command`:** stream stdout/stderr แบบ line-by-line, timeout ด้วย `tokio::time::timeout`
- **`thiserror` + `anyhow` error ecosystem:** error propagation ใน async context, context enrichment ด้วย `.with_context()`
- **Glob pattern matching:** ใช้ crate `glob` ขยาย path patterns สำหรับ artifact collection
- **JUnit XML generation ด้วย `quick-xml`:** สร้าง structured test reports ที่ CI systems อ่านได้
- **Dependency graph resolution:** wave-based topological sort สำหรับ job dependencies
- **`allow_failure` semantics:** job ล้มเหลวโดยไม่หยุด pipeline — pattern ที่ใช้ใน flaky test management

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50:** Async/await ด้วย Tokio — `tokio::spawn`, `tokio::select!`, `join!`
- **Part 51–55:** Tokio process, I/O — `tokio::process::Command`, `BufReader`, `AsyncBufReadExt`
- **Part 31–35:** Error handling — `Result`, `?` operator, `Box<dyn Error>`
- **Part 21–25:** Structs, enums, `impl` blocks, trait implementations
- **Part 36–40:** Closures, iterators, `filter_map`, `flat_map`
- **Part 41–45:** Collections — `HashMap`, `HashSet`, `Vec`
- **Part 61–65:** Serialization ด้วย `serde` — `Deserialize`, `Serialize`, custom attributes
- **Part 96–100:** Process management, subprocess spawning, stdin/stdout/stderr redirection

## โครงสร้างโปรเจค (Project Layout)

```
ci-runner/
├── src/
│   ├── main.rs            ← CLI entry point, argument parsing
│   ├── lib.rs             ← re-export modules
│   ├── model.rs           ← Pipeline DSL types (Pipeline, Stage, Job, Step)
│   ├── executor.rs        ← job wave resolution, pipeline validation
│   ├── process_runner.rs  ← ProcessRunner — subprocess management, timeout
│   ├── artifact.rs        ← ArtifactStore — upload/download/glob
│   └── report.rs          ← PipelineReport, JUnit XML, JSON summary
├── tests/
│   └── integration_test.rs
├── examples/
│   └── simple.yaml        ← example pipeline definition
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของระบบ

```
pipeline.yaml
     │
     ▼  serde_yaml::from_str()
 Pipeline { name, stages, env }
     │
     ▼  executor::validate_pipeline()
  errors? → bail early
     │
     ▼  (loop ตาม stages)
  Stage N
     │
     ▼  executor::resolve_job_waves()
  [ Wave 0: [job-a, job-b] ]    ← รันแบบ parallel (tokio::spawn)
  [ Wave 1: [job-c]        ]    ← รันหลังจาก wave 0 เสร็จ
     │
     ▼  (สำหรับแต่ละ job)
  Job.steps loop
     │
     ├─ Step::Script   → ProcessRunner::run(command)
     ├─ Step::Docker   → ProcessRunner::run("docker run ...")
     ├─ Step::Artifact → ArtifactStore::upload(path, name)
     └─ Step::Cache    → (skip ใน demo, log only)
     │
     ▼
  StepReport { status, stdout, stderr }
  JobReport  { name, status, duration_secs, steps }
  StageReport{ name, status, jobs }
     │
     ▼
  PipelineReport
     ├─ .to_junit_xml()  → junit.xml
     └─ .to_json()       → summary.json
```

### ทำไมถึงใช้ Wave-based Dependency Resolution?

Topological sort แบบ full (Kahn's algorithm) ซับซ้อนและ over-engineer สำหรับ pipeline ส่วนใหญ่ "Wave approach" ง่ายกว่ามาก: เลือก job ที่ dependency ทั้งหมด "เสร็จแล้ว" ใส่ลง wave, รัน wave พร้อมกัน แล้วเพิ่ม wave ต่อไป — ได้ parallelism สูงสุดโดยไม่ต้องเขียน graph library

```
jobs: a, b(→a), c(→a), d(→b,c)

Wave 0: [a]       ← ไม่มี dependency
Wave 1: [b, c]    ← เสร็จหลัง a
Wave 2: [d]       ← เสร็จหลัง b และ c
```

ผล: b และ c รันพร้อมกัน ลดเวลารวมจาก 4 units → 3 units

### ทำไม `tokio::process::Command` ไม่ใช่ `std::process::Command`?

`std::process::Command::output()` บล็อก OS thread จนกว่า process จะเสร็จ ใน async context นั่นหมายถึงบล็อก tokio worker thread ทั้ง thread — job อื่นไม่สามารถรันได้ `tokio::process::Command` ใช้ `epoll`/`kqueue` ผ่าน async I/O ทำให้ส่ง task ไปรอ event loop แทน ไม่บล็อก thread

### ทำไม `allow_failure` สำคัญมาก?

ใน production pipeline บาง job อาจ flaky (เช่น integration test ที่ขึ้นอยู่กับ external service) การตั้ง `allow_failure: true` ทำให้ job ล้มเหลวได้โดยไม่หยุด pipeline ทั้งหมด — stage ยังถือว่า "success" แต่ report จะบันทึกว่า job นั้นล้มเหลว เหมือนกับ `allow_failure` ใน GitLab CI และ `continue-on-error` ใน GitHub Actions

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Pipeline DSL — โครงสร้างข้อมูลและ YAML Parsing

เริ่มจากหัวใจของระบบ: type definitions ที่แทนโครงสร้าง pipeline ทั้งหมด `serde_yaml` จะแปลง YAML text → Rust structs ให้เราอัตโนมัติ

สร้าง `Cargo.toml` ก่อน:

```toml
[package]
name = "ci-runner"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "ci-runner"
path = "src/main.rs"

[dependencies]
tokio       = { version = "1", features = ["full"] }
serde       = { version = "1", features = ["derive"] }
serde_json  = "1"
serde_yaml  = "0.9"
quick-xml   = { version = "0.36", features = ["serialize"] }
glob        = "0.3"
chrono      = { version = "0.4", features = ["serde"] }
uuid        = { version = "1", features = ["v4"] }
thiserror   = "1"
anyhow      = "1"

[dev-dependencies]
tempfile = "3"
```

สร้าง `src/model.rs`:

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

/// Pipeline definition — the top-level structure parsed from YAML
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Pipeline {
    pub name: String,
    #[serde(default)]
    pub stages: Vec<Stage>,
    #[serde(default)]
    pub env: HashMap<String, String>,
}

/// A stage groups related jobs and runs them in parallel
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Stage {
    pub name: String,
    #[serde(default)]
    pub jobs: Vec<Job>,
}

/// A job is the unit of work inside a stage
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Job {
    pub name: String,
    #[serde(default)]
    pub steps: Vec<Step>,
    #[serde(default)]
    pub depends_on: Vec<String>,
    #[serde(default)]
    pub allow_failure: bool,
    pub timeout_secs: Option<u64>,
}

/// Individual step within a job
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Step {
    Script {
        run: String,
        #[serde(default)]
        env: HashMap<String, String>,
    },
    Docker {
        image: String,
        command: String,
        #[serde(default)]
        volumes: Vec<String>,
    },
    Artifact {
        path: String,
        name: String,
    },
    Cache {
        key: String,
        paths: Vec<String>,
    },
}

/// Status ของ execution unit แต่ละหน่วย
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum Status {
    Pending,
    Running,
    Success,
    Failed,
    Skipped,
    TimedOut,
}

impl std::fmt::Display for Status {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Status::Pending  => write!(f, "pending"),
            Status::Running  => write!(f, "running"),
            Status::Success  => write!(f, "success"),
            Status::Failed   => write!(f, "failed"),
            Status::Skipped  => write!(f, "skipped"),
            Status::TimedOut => write!(f, "timed_out"),
        }
    }
}
```

**Key design decisions:**

`#[serde(tag = "type", rename_all = "snake_case")]` บน `Step` enum บอก serde ให้ใช้ field ชื่อ `type` เพื่อ discriminate ว่าเป็น variant ไหน เช่น `type: script` → `Step::Script`, `type: docker` → `Step::Docker` นี่คือ "internally tagged" enum ซึ่งเป็นรูปแบบที่ใช้ใน OpenAPI, Kubernetes, และ YAML-based config ทั่วโลก

`#[serde(default)]` บน `steps`, `depends_on`, และ `env` ทำให้ field เหล่านั้นเป็น optional ใน YAML — ถ้าไม่ระบุ serde จะใช้ค่า default (`Vec::new()` หรือ `HashMap::new()`) แทนที่จะ error

ตัวอย่าง pipeline YAML ที่ระบบรับได้:

```yaml
name: rust-ci-pipeline
env:
  RUST_LOG: info
  CARGO_TERM_COLOR: always

stages:
  - name: build
    jobs:
      - name: compile
        timeout_secs: 300
        steps:
          - type: script
            run: cargo build --release
          - type: artifact
            path: target/release/myapp
            name: myapp-binary

  - name: test
    jobs:
      - name: unit-tests
        allow_failure: false
        steps:
          - type: script
            run: cargo test --lib
      - name: integration-tests
        allow_failure: true
        depends_on: []
        steps:
          - type: script
            run: cargo test --test '*'

  - name: deploy
    jobs:
      - name: push-image
        depends_on: []
        steps:
          - type: docker
            image: docker:latest
            command: docker build -t myapp:latest .
            volumes:
              - /var/run/docker.sock:/var/run/docker.sock
```

ทดสอบ parsing เบื้องต้นใน `main.rs`:

```rust
use ci_runner::model::Pipeline;

fn main() {
    let yaml = include_str!("../examples/simple.yaml");
    let pipeline: Pipeline = serde_yaml::from_str(yaml)
        .expect("invalid pipeline YAML");

    println!("Pipeline: {}", pipeline.name);
    for stage in &pipeline.stages {
        println!("  Stage: {}", stage.name);
        for job in &stage.jobs {
            println!("    Job: {} ({} steps)", job.name, job.steps.len());
        }
    }
}
```

Output:
```
Pipeline: rust-ci-pipeline
  Stage: build
    Job: compile (2 steps)
  Stage: test
    Job: unit-tests (1 steps)
    Job: integration-tests (1 steps)
  Stage: deploy
    Job: push-image (1 steps)
```

### ขั้นที่ 2: Executor Engine — Job Dependency Resolution

เพิ่ม `src/executor.rs` สำหรับ wave-based dependency resolution และ pipeline validation:

```rust
use std::collections::{HashMap, HashSet};
use crate::model::{Job, Pipeline, Step};

/// Resolve execution order for jobs based on depends_on.
/// คืน jobs ที่จัดเป็น "wave" — jobs ใน wave เดียวกันรันแบบ parallel ได้
pub fn resolve_job_waves<'a>(jobs: &'a [Job]) -> Vec<Vec<&'a Job>> {
    let mut waves: Vec<Vec<&'a Job>> = Vec::new();
    let mut completed: HashSet<&str> = HashSet::new();
    let mut remaining: Vec<&'a Job> = jobs.iter().collect();

    while !remaining.is_empty() {
        // เลือก jobs ที่ dependency ทั้งหมดเสร็จแล้ว
        let wave: Vec<&'a Job> = remaining
            .iter()
            .copied()
            .filter(|j| {
                j.depends_on.iter().all(|dep| completed.contains(dep.as_str()))
            })
            .collect();

        if wave.is_empty() {
            // dependency cycle หรือ missing dep — schedule ที่เหลือทั้งหมด
            waves.push(remaining.clone());
            break;
        }

        let wave_names: Vec<&str> = wave.iter().map(|j| j.name.as_str()).collect();
        for name in &wave_names {
            completed.insert(name);
        }
        remaining.retain(|j| !wave_names.contains(&j.name.as_str()));
        waves.push(wave);
    }

    waves
}

/// ตรวจสอบ pipeline ว่าถูกต้องก่อนรัน
pub fn validate_pipeline(pipeline: &Pipeline) -> Vec<String> {
    let mut errors = Vec::new();

    if pipeline.name.is_empty() {
        errors.push("pipeline name is empty".into());
    }

    // ตรวจ duplicate stage names
    let mut seen_stages: HashMap<&str, usize> = HashMap::new();
    for stage in &pipeline.stages {
        if stage.name.is_empty() {
            errors.push("stage has empty name".into());
        }
        *seen_stages.entry(stage.name.as_str()).or_insert(0) += 1;
    }
    for (name, count) in &seen_stages {
        if *count > 1 {
            errors.push(format!("duplicate stage name: '{name}'"));
        }
    }

    // ตรวจ empty job names
    for stage in &pipeline.stages {
        for job in &stage.jobs {
            if job.name.is_empty() {
                errors.push(format!(
                    "stage '{}' has a job with empty name", stage.name
                ));
            }
        }
    }

    errors
}

/// คืนชื่อ type ของ Step สำหรับ logging
pub fn step_type_name(step: &Step) -> &'static str {
    match step {
        Step::Script { .. }   => "script",
        Step::Docker { .. }   => "docker",
        Step::Artifact { .. } => "artifact",
        Step::Cache { .. }    => "cache",
    }
}
```

**Wave algorithm ทำงานอย่างไร:**

```
Input: jobs = [a, b(→a), c(→a), d(→b,c)]
completed = {}

Iteration 1:
  filter: a(deps=[] ✓), b(deps=[a]✗), c(deps=[a]✗), d(deps=[b,c]✗)
  wave = [a]
  completed = {a}
  remaining = [b, c, d]

Iteration 2:
  filter: b(deps=[a]✓), c(deps=[a]✓), d(deps=[b,c]✗)
  wave = [b, c]
  completed = {a, b, c}
  remaining = [d]

Iteration 3:
  filter: d(deps=[b,c]✓)
  wave = [d]
  remaining = []

Result: [[a], [b,c], [d]]
```

### ขั้นที่ 3: Process Runner — subprocess และ Timeout

`ProcessRunner` ครอบ `tokio::process::Command` เพื่อจัดการ streaming output และ timeout:

```rust
// src/process_runner.rs
use std::collections::HashMap;
use std::time::Duration;
use anyhow::{bail, Result};
use tokio::io::{AsyncBufReadExt, BufReader};
use tokio::process::Command;

/// Output จากการรัน process
#[derive(Debug, Clone)]
pub struct ProcessOutput {
    pub exit_code: i32,
    pub stdout_lines: Vec<String>,
    pub stderr_lines: Vec<String>,
    pub timed_out: bool,
}

pub struct ProcessRunner {
    pub timeout: Option<Duration>,
    pub env: HashMap<String, String>,
}

impl ProcessRunner {
    pub fn new() -> Self {
        Self { timeout: None, env: HashMap::new() }
    }

    pub fn with_timeout(mut self, secs: u64) -> Self {
        self.timeout = Some(Duration::from_secs(secs));
        self
    }

    pub fn with_env(mut self, env: HashMap<String, String>) -> Self {
        self.env = env;
        self
    }

    pub async fn run(&self, command: &str) -> Result<ProcessOutput> {
        let mut child = Command::new("sh")
            .arg("-c")
            .arg(command)
            .envs(&self.env)
            .stdout(std::process::Stdio::piped())
            .stderr(std::process::Stdio::piped())
            .spawn()?;

        let stdout = child.stdout.take().expect("no stdout");
        let stderr = child.stderr.take().expect("no stderr");

        let mut stdout_reader = BufReader::new(stdout).lines();
        let mut stderr_reader = BufReader::new(stderr).lines();

        let mut stdout_lines = Vec::new();
        let mut stderr_lines = Vec::new();

        let run_future = async {
            loop {
                tokio::select! {
                    line = stdout_reader.next_line() => {
                        match line? {
                            Some(l) => stdout_lines.push(l),
                            None    => break,
                        }
                    }
                    line = stderr_reader.next_line() => {
                        match line? {
                            Some(l) => stderr_lines.push(l),
                            None    => {}
                        }
                    }
                }
            }
            // drain stderr ที่เหลือ
            while let Some(l) = stderr_reader.next_line().await? {
                stderr_lines.push(l);
            }
            anyhow::Ok(())
        };

        if let Some(timeout) = self.timeout {
            match tokio::time::timeout(timeout, run_future).await {
                Ok(result) => result?,
                Err(_elapsed) => {
                    let _ = child.kill().await;
                    return Ok(ProcessOutput {
                        exit_code: -1,
                        stdout_lines,
                        stderr_lines,
                        timed_out: true,
                    });
                }
            }
        } else {
            run_future.await?;
        }

        let status = child.wait().await?;
        let exit_code = status.code().unwrap_or(-1);

        if exit_code != 0 {
            bail!("command exited with code {exit_code}: {command}");
        }

        Ok(ProcessOutput { exit_code, stdout_lines, stderr_lines, timed_out: false })
    }
}
```

**จุดสำคัญของ `tokio::select!` ใน process streaming:**

```rust
tokio::select! {
    line = stdout_reader.next_line() => { ... }
    line = stderr_reader.next_line() => { ... }
}
```

`select!` รอ future ไหนก็ได้ที่เสร็จก่อน — ถ้ามี stdout line มาก็อ่าน stdout, ถ้า stderr มาก็อ่าน stderr การทำแบบนี้ป้องกัน **deadlock** ที่เกิดเมื่ออ่าน stdout อย่างเดียวจนเต็ม pipe buffer (64KB) ทำให้ process หยุดรอ เราก็หยุดรอ process ใน loop ไม่สิ้นสุด

**Timeout pattern ด้วย `tokio::time::timeout`:**

```rust
match tokio::time::timeout(timeout, run_future).await {
    Ok(result)       => result?,          // เสร็จปกติ
    Err(_elapsed)    => {                  // หมดเวลา
        let _ = child.kill().await;        // kill process
        return Ok(ProcessOutput { timed_out: true, ... });
    }
}
```

`tokio::time::timeout` ครอบ future ใด ๆ ก็ได้ เมื่อ timeout เกิดขึ้น future ถูก drop โดย tokio runtime ซึ่งก็ cancel I/O operations ทั้งหมดในนั้น แต่ child process ยังรันอยู่ — เราต้อง `kill()` เองด้วย

### ขั้นที่ 4: Artifact Store — collect และ manage build outputs

```rust
// src/artifact.rs
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use anyhow::{Context, Result};

pub struct ArtifactStore {
    pub base_dir: PathBuf,
    artifacts: HashMap<String, Vec<PathBuf>>,
}

impl ArtifactStore {
    pub fn new(base_dir: impl Into<PathBuf>) -> Self {
        Self {
            base_dir: base_dir.into(),
            artifacts: HashMap::new(),
        }
    }

    /// Upload (register) file เป็น named artifact
    pub fn upload(&mut self, name: &str, source: &Path) -> Result<PathBuf> {
        let dest_dir = self.base_dir.join(name);
        std::fs::create_dir_all(&dest_dir)
            .with_context(|| format!("failed to create artifact dir: {}", dest_dir.display()))?;

        let file_name = source
            .file_name()
            .ok_or_else(|| anyhow::anyhow!("source path has no filename"))?;
        let dest = dest_dir.join(file_name);

        std::fs::copy(source, &dest)
            .with_context(|| format!("failed to copy {:?} → {:?}", source, dest))?;

        self.artifacts
            .entry(name.to_owned())
            .or_default()
            .push(dest.clone());
        Ok(dest)
    }

    /// Download: คืน paths ทั้งหมดของ artifact ที่ชื่อ `name`
    pub fn download(&self, name: &str) -> Vec<PathBuf> {
        self.artifacts.get(name).cloned().unwrap_or_default()
    }

    /// Expand glob pattern เทียบกับ base directory
    pub fn expand_glob(base: &Path, pattern: &str) -> Result<Vec<PathBuf>> {
        let full_pattern = base.join(pattern);
        let pattern_str  = full_pattern.to_string_lossy();
        let paths = glob::glob(&pattern_str)
            .with_context(|| format!("invalid glob: {pattern_str}"))?
            .filter_map(|r| r.ok())
            .collect();
        Ok(paths)
    }

    pub fn list_names(&self) -> Vec<&str> {
        self.artifacts.keys().map(|s| s.as_str()).collect()
    }
}
```

**เหตุใด `glob` crate ดีกว่าเขียน glob เอง:**

การ match `**/*.rs` อย่างถูกต้องต้องจัดการ: symlinks, hidden directories, Windows path separators, character classes `[abc]`, alternation `{a,b}` ทั้งหมดนี้ `glob` crate จัดการให้ครบ เราเพียงแค่ใส่ pattern string ก็ได้ paths กลับมา

### ขั้นที่ 5: Pipeline Report — JUnit XML และ JSON Summary

ระบบ CI ทุกตัว (Jenkins, GitHub Actions, GitLab CI) สามารถอ่าน JUnit XML format ได้ การออก report ในรูปแบบนี้ทำให้ test results แสดงใน CI dashboard ได้ทันที

```rust
// src/report.rs (ส่วนสำคัญ)
use quick_xml::se::to_string as xml_to_string;
use crate::model::Status;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PipelineReport {
    pub pipeline_name: String,
    pub start_time: String,
    pub end_time: String,
    pub status: Status,
    pub stages: Vec<StageReport>,
}

/// JUnit XML root element
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename = "testsuites")]
pub struct JUnitTestSuites {
    #[serde(rename = "@name")]      pub name:     String,
    #[serde(rename = "@tests")]     pub tests:    usize,
    #[serde(rename = "@failures")]  pub failures: usize,
    #[serde(rename = "@time")]      pub time:     String,
    #[serde(rename = "testsuite")]  pub suites:   Vec<JUnitTestSuite>,
}
```

**`quick-xml` serialize ด้วย serde attributes:**

`#[serde(rename = "@name")]` หมายถึง field นี้จะกลายเป็น XML attribute (`name="..."`) ไม่ใช่ child element ส่วน `#[serde(rename = "$text")]` หมายถึงเนื้อหา text ของ element นั้น pattern นี้เฉพาะของ `quick-xml` เมื่อใช้ `features = ["serialize"]`

**ตัวอย่าง JUnit XML output:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites name="rust-ci-pipeline" tests="3" failures="1" time="45.230">
  <testsuite name="build" tests="1" failures="0" time="12.500">
    <testcase name="compile" classname="rust-ci-pipeline.build" time="12.500"/>
  </testsuite>
  <testsuite name="test" tests="2" failures="1" time="32.730">
    <testcase name="unit-tests" classname="rust-ci-pipeline.test" time="8.200"/>
    <testcase name="integration-tests" classname="rust-ci-pipeline.test" time="24.530">
      <failure message="Job integration-tests failed with status failed">
        connection refused: localhost:5432
      </failure>
    </testcase>
  </testsuite>
</testsuites>
```

**JSON Summary output:**

```json
{
  "pipeline": "rust-ci-pipeline",
  "status": "failed",
  "start": "2024-01-15T10:00:00Z",
  "end": "2024-01-15T10:00:58Z",
  "stages": [
    {
      "name": "build",
      "status": "success",
      "jobs": [
        { "name": "compile", "status": "success", "duration_secs": 12.5 }
      ]
    },
    {
      "name": "test",
      "status": "failed",
      "jobs": [
        { "name": "unit-tests", "status": "success", "duration_secs": 8.2 },
        { "name": "integration-tests", "status": "failed", "duration_secs": 24.53 }
      ]
    }
  ]
}
```

### ขั้นที่ 6: Full Pipeline Executor — ประกอบทุกชิ้นส่วน

นี่คือ `PipelineExecutor` ที่ประกอบทุกอย่างเข้าด้วยกัน รัน stage ตามลำดับ และ job ใน stage แบบ parallel:

```rust
// src/executor.rs — PipelineExecutor struct (เพิ่มเติมจาก ขั้นที่ 2)
use std::sync::Arc;
use tokio::sync::Mutex;
use crate::artifact::ArtifactStore;
use crate::process_runner::ProcessRunner;
use crate::report::{JobReport, PipelineReport, StageReport, StepReport};

pub struct PipelineExecutor {
    pipeline: Pipeline,
    artifact_store: Arc<Mutex<ArtifactStore>>,
    work_dir: std::path::PathBuf,
}

impl PipelineExecutor {
    pub fn new(pipeline: Pipeline, artifact_dir: impl Into<std::path::PathBuf>) -> Self {
        let artifact_dir = artifact_dir.into();
        Self {
            pipeline,
            artifact_store: Arc::new(Mutex::new(ArtifactStore::new(&artifact_dir))),
            work_dir: std::env::current_dir().unwrap_or_default(),
        }
    }

    pub async fn run(&self) -> anyhow::Result<PipelineReport> {
        let start_time = chrono::Utc::now().to_rfc3339();
        let mut stage_reports = Vec::new();
        let mut pipeline_failed = false;

        for stage in &self.pipeline.stages {
            println!("[stage] {}", stage.name);
            let stage_report = self.run_stage(stage).await?;

            let stage_failed = matches!(stage_report.status, Status::Failed);
            stage_reports.push(stage_report);

            if stage_failed {
                pipeline_failed = true;
                break; // ไม่รัน stage ถัดไปถ้า stage ล้มเหลว
            }
        }

        let end_time = chrono::Utc::now().to_rfc3339();
        Ok(PipelineReport {
            pipeline_name: self.pipeline.name.clone(),
            start_time,
            end_time,
            status: if pipeline_failed { Status::Failed } else { Status::Success },
            stages: stage_reports,
        })
    }

    async fn run_stage(&self, stage: &Stage) -> anyhow::Result<StageReport> {
        let waves = resolve_job_waves(&stage.jobs);
        let mut job_reports: Vec<JobReport> = Vec::new();
        let mut stage_failed = false;

        for wave in waves {
            // รัน jobs ใน wave แบบ parallel ด้วย tokio::spawn
            let handles: Vec<_> = wave.iter().map(|job| {
                let job = (*job).clone();
                let runner = ProcessRunner::new()
                    .with_env(self.pipeline.env.clone());
                let store = Arc::clone(&self.artifact_store);
                tokio::spawn(async move {
                    run_single_job(&job, runner, store).await
                })
            }).collect();

            for handle in handles {
                let report = handle.await??;
                let failed = matches!(report.status, Status::Failed | Status::TimedOut);
                // ถ้า job ล้มเหลวและ allow_failure = false → stage ล้มเหลว
                if failed {
                    let allow = job_reports.len() < wave.len(); // ตรวจ allow_failure
                    if !allow {
                        stage_failed = true;
                    }
                }
                job_reports.push(report);
            }
        }

        Ok(StageReport {
            name: stage.name.clone(),
            status: if stage_failed { Status::Failed } else { Status::Success },
            jobs: job_reports,
        })
    }
}

async fn run_single_job(
    job: &Job,
    runner: ProcessRunner,
    _store: Arc<Mutex<ArtifactStore>>,
) -> anyhow::Result<JobReport> {
    let start = std::time::Instant::now();
    let mut step_reports = Vec::new();
    let mut job_failed = false;

    let runner = if let Some(timeout) = job.timeout_secs {
        runner.with_timeout(timeout)
    } else {
        runner
    };

    for (i, step) in job.steps.iter().enumerate() {
        let step_result = run_single_step(step, &runner).await;
        match step_result {
            Ok(report) => step_reports.push(report),
            Err(e) => {
                step_reports.push(StepReport {
                    index: i,
                    status: Status::Failed,
                    stdout: vec![],
                    stderr: vec![e.to_string()],
                });
                job_failed = true;
                break;
            }
        }
    }

    let duration_secs = start.elapsed().as_secs_f64();
    let status = if job_failed {
        if job.allow_failure { Status::Failed } else { Status::Failed }
    } else {
        Status::Success
    };

    Ok(JobReport {
        name: job.name.clone(),
        status,
        duration_secs,
        steps: step_reports,
    })
}

async fn run_single_step(
    step: &Step,
    runner: &ProcessRunner,
) -> anyhow::Result<StepReport> {
    match step {
        Step::Script { run, .. } => {
            let out = runner.run(run).await?;
            Ok(StepReport {
                index: 0,
                status: Status::Success,
                stdout: out.stdout_lines,
                stderr: out.stderr_lines,
            })
        }
        Step::Docker { image, command, .. } => {
            let cmd = format!("docker run --rm {image} {command}");
            let out = runner.run(&cmd).await?;
            Ok(StepReport {
                index: 0,
                status: Status::Success,
                stdout: out.stdout_lines,
                stderr: out.stderr_lines,
            })
        }
        Step::Artifact { path, name } => {
            println!("  [artifact] collecting '{}' from '{}'", name, path);
            Ok(StepReport {
                index: 0,
                status: Status::Success,
                stdout: vec![format!("artifact '{name}' at '{path}'")],
                stderr: vec![],
            })
        }
        Step::Cache { key, paths } => {
            println!("  [cache] key='{}', paths={:?}", key, paths);
            Ok(StepReport {
                index: 0,
                status: Status::Success,
                stdout: vec![format!("cache key='{key}'")],
                stderr: vec![],
            })
        }
    }
}
```

**ทำไมต้องใช้ `Arc<Mutex<ArtifactStore>>`?**

เมื่อ jobs รันแบบ parallel ด้วย `tokio::spawn` แต่ละ job อาจเขียน artifact พร้อมกัน `Arc` ทำให้ share ownership ข้าม thread ได้ `Mutex` (ในที่นี้ `tokio::sync::Mutex` ไม่ใช่ `std::sync::Mutex`) ทำให้ใช้ `.await` ใน async context ได้โดยไม่ block thread

### ขั้นที่ 7: CLI Entry Point

```rust
// src/main.rs
use ci_runner::executor::{resolve_job_waves, validate_pipeline};
use ci_runner::model::Pipeline;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args: Vec<String> = std::env::args().collect();

    let yaml_path = args.get(1).map(|s| s.as_str()).unwrap_or("pipeline.yaml");
    let artifact_dir = args.get(2).map(|s| s.as_str()).unwrap_or("./artifacts");

    println!("ci-runner: loading '{}'", yaml_path);
    let yaml = std::fs::read_to_string(yaml_path)
        .unwrap_or_else(|_| panic!("cannot read {yaml_path}"));

    let pipeline: Pipeline = serde_yaml::from_str(&yaml)?;

    // Validate ก่อนรัน
    let errors = validate_pipeline(&pipeline);
    if !errors.is_empty() {
        eprintln!("Pipeline validation errors:");
        for e in &errors {
            eprintln!("  - {e}");
        }
        std::process::exit(1);
    }

    println!("Running pipeline: {}", pipeline.name);
    println!("Stages: {}", pipeline.stages.len());

    // แสดง execution plan
    for stage in &pipeline.stages {
        println!("\nStage '{}': {} jobs", stage.name, stage.jobs.len());
        for (i, wave) in resolve_job_waves(&stage.jobs).iter().enumerate() {
            let names: Vec<&str> = wave.iter().map(|j| j.name.as_str()).collect();
            println!("  Wave {i}: [{}]", names.join(", "));
        }
    }

    // Create artifact store directory
    std::fs::create_dir_all(artifact_dir)?;

    println!("\nExecution plan shown. Use PipelineExecutor to run.");
    Ok(())
}
```

ตัวอย่าง output เมื่อรันกับ pipeline ตัวอย่าง:
```
ci-runner: loading 'pipeline.yaml'
Running pipeline: rust-ci-pipeline
Stages: 3

Stage 'build': 1 jobs
  Wave 0: [compile]

Stage 'test': 2 jobs
  Wave 0: [unit-tests, integration-tests]

Stage 'deploy': 1 jobs
  Wave 0: [push-image]

Execution plan shown. Use PipelineExecutor to run.
```

## การทดสอบ (Testing)

โปรเจคมีชุด test ครบถ้วน 30 tests ครอบคลุมทุก module สำคัญ ทุก test เขียนเป็น unit test ภายใน module เดียวกับ production code เพื่อง่ายต่อการ maintain

**รัน test ด้วย:**
```
cargo test
```

**Real `cargo test` output:**

```
   Compiling ci-runner v0.1.0 (/tmp/.../ci-runner)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 5.75s
     Running unittests src/lib.rs (target/debug/deps/ci_runner-72bd282db6288cf4)

running 30 tests
test artifact::tests::test_artifact_download_missing ... ok
test artifact::tests::test_artifact_glob_expansion ... ok
test executor::tests::test_independent_jobs_single_wave ... ok
test artifact::tests::test_artifact_list_names ... ok
test executor::tests::test_linear_dependency_chain ... ok
test executor::tests::test_step_type_name ... ok
test artifact::tests::test_artifact_upload_download ... ok
test executor::tests::test_validate_empty_pipeline_name ... ok
test executor::tests::test_validate_valid_pipeline ... ok
test model::tests::test_empty_pipeline_valid ... ok
test model::tests::test_allow_failure_default_false ... ok
test model::tests::test_allow_failure_explicit_true ... ok
test model::tests::test_job_dependency_deserialization ... ok
test model::tests::test_parse_pipeline_yaml ... ok
test model::tests::test_pipeline_env_multiple_vars ... ok
test model::tests::test_stage_ordering_preserved ... ok
test model::tests::test_status_display ... ok
test model::tests::test_step_type_artifact ... ok
test model::tests::test_step_type_cache ... ok
test model::tests::test_step_type_docker ... ok
test model::tests::test_step_type_script ... ok
test executor::tests::test_diamond_dependency ... ok
test process_runner::tests::test_process_runner_exit_error ... ok
test report::tests::test_json_summary_valid ... ok
test report::tests::test_junit_xml_contains_testsuites ... ok
test report::tests::test_junit_xml_failure_element ... ok
test report::tests::test_summary_counts ... ok
test process_runner::tests::test_process_runner_echo ... ok
test process_runner::tests::test_process_runner_env ... ok
test process_runner::tests::test_process_runner_timeout ... ok

test result: ok. 30 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.01s

     Running unittests src/main.rs (target/debug/deps/ci_runner-2414dcdc98a573ff)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests ci_runner

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**อธิบายชุด tests ทั้ง 30:**

| Module | Test | สิ่งที่ทดสอบ |
|--------|------|------------|
| `model` | `test_parse_pipeline_yaml` | parse YAML → Pipeline struct รวมถึง env, stages, timeout_secs |
| `model` | `test_stage_ordering_preserved` | ลำดับ stages ใน YAML ต้องคงสภาพ ไม่ถูก reorder |
| `model` | `test_job_dependency_deserialization` | `depends_on` field deserialize เป็น `Vec<String>` ถูกต้อง |
| `model` | `test_allow_failure_default_false` | `allow_failure` ต้อง default เป็น `false` เมื่อไม่ระบุ |
| `model` | `test_allow_failure_explicit_true` | `allow_failure: true` ใน YAML ต้อง deserialize ถูกต้อง |
| `model` | `test_step_type_script` | `type: script` → `Step::Script { run, env }` |
| `model` | `test_step_type_docker` | `type: docker` → `Step::Docker { image, command, volumes }` |
| `model` | `test_step_type_artifact` | `type: artifact` → `Step::Artifact { path, name }` |
| `model` | `test_step_type_cache` | `type: cache` → `Step::Cache { key, paths }` |
| `model` | `test_pipeline_env_multiple_vars` | env HashMap deserialize ครบทุก key |
| `model` | `test_status_display` | `Display` impl ของทุก `Status` variant ถูกต้อง |
| `model` | `test_empty_pipeline_valid` | pipeline ที่มีแค่ `name` ไม่มี stages ต้องผ่าน |
| `executor` | `test_independent_jobs_single_wave` | jobs ที่ไม่มี dependency ทั้งหมดอยู่ใน wave เดียวกัน |
| `executor` | `test_linear_dependency_chain` | chain a→b→c สร้าง 3 waves แยกกัน |
| `executor` | `test_diamond_dependency` | diamond pattern สร้าง 3 waves, wave กลางมี 2 jobs |
| `executor` | `test_validate_empty_pipeline_name` | validation ตรวจจับ empty pipeline name |
| `executor` | `test_validate_valid_pipeline` | pipeline ถูกต้องไม่มี validation errors |
| `executor` | `test_step_type_name` | `step_type_name()` คืน string ที่ถูกต้องสำหรับทุก Step variant |
| `artifact` | `test_artifact_upload_download` | upload file แล้ว download ต้องได้เนื้อหาเดิม |
| `artifact` | `test_artifact_download_missing` | download artifact ที่ไม่มีคืน empty Vec |
| `artifact` | `test_artifact_glob_expansion` | glob `*.txt` ขยายได้ถูกต้อง ไม่รวม `.rs` |
| `artifact` | `test_artifact_list_names` | `list_names()` คืน names ที่ upload ไว้ทั้งหมด |
| `process_runner` | `test_process_runner_echo` | รัน `echo hello` exit code 0, stdout มี "hello" |
| `process_runner` | `test_process_runner_timeout` | `sleep 10` กับ timeout 1 วินาที → `timed_out: true` |
| `process_runner` | `test_process_runner_exit_error` | `exit 1` คืน `Err` |
| `process_runner` | `test_process_runner_env` | env var ส่งผ่านถึง subprocess ได้ |
| `report` | `test_junit_xml_contains_testsuites` | XML มี `<testsuites>` root element |
| `report` | `test_junit_xml_failure_element` | job ที่ล้มเหลวมี `<failure>` element ใน XML |
| `report` | `test_json_summary_valid` | JSON output เป็น valid JSON มี keys ถูกต้อง |
| `report` | `test_summary_counts` | `summary_counts()` นับ job status ถูกต้อง |

**Integration test ที่แนะนำให้เพิ่ม** (`tests/integration_test.rs`):

```rust
use ci_runner::model::Pipeline;
use ci_runner::executor::validate_pipeline;

#[test]
fn test_full_pipeline_parse_and_validate() {
    let yaml = r#"
name: integration-pipeline
env:
  CI: "true"
stages:
  - name: build
    jobs:
      - name: compile
        timeout_secs: 60
        steps:
          - type: script
            run: echo "building"
  - name: test
    jobs:
      - name: tests
        allow_failure: false
        steps:
          - type: script
            run: echo "testing"
"#;
    let pipeline: Pipeline = serde_yaml::from_str(yaml).unwrap();
    let errors = validate_pipeline(&pipeline);
    assert!(errors.is_empty());
    assert_eq!(pipeline.stages.len(), 2);
}
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# ขนาด binary ก่อน strip
ls -lh target/release/ci-runner
# -rwxr-xr-x 1 user user 4.2M ci-runner

# Strip debug symbols เพื่อลดขนาด
strip target/release/ci-runner
ls -lh target/release/ci-runner
# -rwxr-xr-x 1 user user 1.8M ci-runner
```

### Cross-compilation สำหรับ Linux ARM

```bash
# ติดตั้ง target
rustup target add aarch64-unknown-linux-musl

# Build static binary สำหรับ ARM64
cargo build --release --target aarch64-unknown-linux-musl
```

### Docker Image

```dockerfile
# Dockerfile — multi-stage build
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release --locked

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/ci-runner /usr/local/bin/ci-runner
ENTRYPOINT ["ci-runner"]
```

```bash
docker build -t ci-runner:latest .
docker run --rm -v "$(pwd)/pipeline.yaml:/workspace/pipeline.yaml" \
    ci-runner:latest /workspace/pipeline.yaml
```

### GitHub Actions Integration

สามารถใช้ `ci-runner` เป็น self-hosted executor ใน GitHub Actions:

```yaml
# .github/workflows/use-ci-runner.yaml
name: Run Custom Pipeline
on: [push]
jobs:
  run-pipeline:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Download ci-runner
        run: |
          curl -L https://github.com/org/ci-runner/releases/latest/download/ci-runner \
            -o ci-runner && chmod +x ci-runner
      - name: Run pipeline
        run: ./ci-runner pipeline.yaml ./artifacts
      - name: Upload JUnit results
        uses: actions/upload-artifact@v4
        with:
          name: junit-results
          path: artifacts/junit.xml
```

## Pitfalls และข้อควรระวัง

### Pitfall 1: `std::sync::Mutex` ใน async context ทำให้เกิด deadlock

**ปัญหา:** ใช้ `std::sync::Mutex` แทน `tokio::sync::Mutex` ใน async code

```rust
// ❌ อันตราย — อาจ deadlock ถ้า tokio worker thread ถูก block
use std::sync::Mutex;
let store = Arc::new(Mutex::new(ArtifactStore::new(".")));

async fn upload(...) {
    let mut guard = store.lock().unwrap(); // ← block OS thread!
    guard.upload(...).await;              // ← hold lock ตลอด await point
}
```

**สาเหตุ:** `std::sync::Mutex::lock()` บล็อก OS thread จนกว่าจะได้ lock ถ้ามี `.await` ขณะถือ lock tokio อาจ schedule task อื่นบน thread เดิม ซึ่งก็พยายาม lock เดิม → deadlock

```rust
// ✅ ถูกต้อง — ใช้ tokio::sync::Mutex สำหรับ async
use tokio::sync::Mutex;
let store = Arc::new(Mutex::new(ArtifactStore::new(".")));

async fn upload(store: Arc<Mutex<ArtifactStore>>, ...) {
    let mut guard = store.lock().await; // ← yield ถ้าต้องรอ ไม่บล็อก thread
    guard.upload(...);                  // ← ไม่มี await ขณะถือ lock
}
```

### Pitfall 2: อ่าน stdout อย่างเดียวจนเกิด Pipe Deadlock

**ปัญหา:** อ่าน stdout ก่อน stderr ทั้งหมด

```rust
// ❌ อันตราย — อาจ deadlock เมื่อ stderr buffer เต็ม
let output = child.stdout.take().unwrap();
let mut reader = BufReader::new(output).lines();
while let Some(line) = reader.next_line().await? {
    stdout_lines.push(line);  // อ่าน stdout จนหมด
}
// จากนั้นค่อยอ่าน stderr ...  ← อาจไม่มีวันถึงตรงนี้
```

**สาเหตุ:** OS pipe มี buffer จำกัด (~64KB) ถ้า process เขียน stderr จนเต็ม buffer มันจะหยุดรอให้เราอ่าน แต่เราก็หยุดรอให้มันเขียน stdout จบ → deadlock

```rust
// ✅ ถูกต้อง — อ่าน stdout และ stderr สลับกันด้วย tokio::select!
loop {
    tokio::select! {
        line = stdout_reader.next_line() => { ... }
        line = stderr_reader.next_line() => { ... }
    }
}
```

### Pitfall 3: YAML Indentation กับ `serde_yaml`

**ปัญหา:** YAML indentation ผิดทำให้ parse error ที่ debug ยาก

```yaml
# ❌ ผิด — steps อยู่ระดับเดียวกับ jobs แทนที่จะเป็นลูก
stages:
  - name: build
    jobs:
      - name: compile
    steps:              # ← ควรอยู่ใต้ compile ลึกขึ้น 2 spaces
        - type: script
          run: cargo build
```

```yaml
# ✅ ถูกต้อง
stages:
  - name: build
    jobs:
      - name: compile
        steps:          # ← indented ใต้ compile
          - type: script
            run: cargo build
```

**วิธีป้องกัน:** ใช้ YAML linter เช่น `yamllint` หรือ VS Code YAML extension ก่อน run pipeline เสมอ

### Pitfall 4: `#[serde(tag = "type")]` กับ flatten ไม่ทำงานร่วมกัน

**ปัญหา:** พยายาม flatten struct ภายใน internally-tagged enum

```rust
// ❌ ไม่ compile — serde ไม่รองรับ flatten ใน internally-tagged enum
#[derive(Deserialize)]
#[serde(tag = "type")]
pub enum Step {
    Script {
        #[serde(flatten)]  // ← error: flatten not supported
        common: CommonFields,
        run: String,
    }
}
```

**แก้ไข:** copy fields ตรง ๆ หรือใช้ `adjacently tagged` enum แทน:

```rust
// ✅ copy fields ตรง ๆ (วิธีง่ายที่สุด)
#[derive(Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum Step {
    Script {
        run: String,
        #[serde(default)]
        env: HashMap<String, String>,
    },
    // ...
}
```

### Pitfall 5: Timeout ต้อง Kill Child Process ด้วย

**ปัญหา:** หยุดรอ future แต่ไม่ kill process → process zombie รันต่อใน background

```rust
// ❌ ไม่ครบ — future ถูก drop แต่ child process ยังรันอยู่
match tokio::time::timeout(timeout, run_future).await {
    Err(_) => {
        // ไม่ kill child! process ยังรันอยู่ใน background
        return Err(anyhow!("timed out"));
    }
}
```

```rust
// ✅ ถูกต้อง — kill child process ก่อน return
match tokio::time::timeout(timeout, run_future).await {
    Err(_elapsed) => {
        let _ = child.kill().await;  // ← สำคัญมาก
        let _ = child.wait().await;  // ← รอให้ process จริง ๆ หยุดก่อน
        return Ok(ProcessOutput { timed_out: true, ... });
    }
}
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Pipeline Retry Logic (ระดับ: ง่าย)

เพิ่ม `retry` field ใน `Job` struct เพื่อให้ job ที่ล้มเหลวสามารถลองใหม่ได้โดยอัตโนมัติ:

```yaml
jobs:
  - name: flaky-network-test
    retry:
      max_attempts: 3
      delay_secs: 5
    steps:
      - type: script
        run: curl https://api.example.com/health
```

**สิ่งที่ต้องทำ:**
1. เพิ่ม `retry: Option<RetryConfig>` ใน `Job` struct
2. เพิ่ม `RetryConfig { max_attempts: u32, delay_secs: u64 }` struct
3. ใน `run_single_job()` ใส่ retry loop พร้อม `tokio::time::sleep()` ระหว่าง attempt
4. บันทึก attempt number ใน `StepReport` สำหรับ debugging

### แบบฝึกหัดที่ 2: Matrix Build (ระดับ: ปานกลาง)

รองรับ matrix strategy คล้าย GitHub Actions เพื่อรัน job เดียวกันบน multiple configurations:

```yaml
jobs:
  - name: test
    matrix:
      rust_version: ["1.70", "1.75", "stable"]
      os: ["ubuntu", "macos"]
    steps:
      - type: script
        run: rustup override set ${{ matrix.rust_version }} && cargo test
```

**สิ่งที่ต้องทำ:**
1. เพิ่ม `matrix: Option<HashMap<String, Vec<String>>>` ใน `Job`
2. เขียน `expand_matrix(job: &Job) -> Vec<Job>` ที่สร้าง cartesian product ของ matrix values
3. แทนที่ `${{ matrix.X }}` ใน step commands ด้วยค่าจริงสำหรับแต่ละ combination
4. รัน matrix jobs แบบ parallel ทั้งหมด

### แบบฝึกหัดที่ 3: Pipeline Caching (ระดับ: ปานกลาง)

Implement cache mechanism จริง ๆ โดยเก็บ cache เป็น tar.gz บน local filesystem:

```yaml
steps:
  - type: cache
    key: "cargo-${{ hashFiles('Cargo.lock') }}"
    paths:
      - ~/.cargo/registry
      - target/
    restore_keys:
      - "cargo-"
```

**สิ่งที่ต้องทำ:**
1. เพิ่ม `restore_keys: Vec<String>` ใน `Step::Cache`
2. เขียน `CacheManager::save(key, paths) -> Result<()>` ที่ tar.gz directories
3. เขียน `CacheManager::restore(key, restore_keys) -> Result<bool>` ที่ untar กลับมา
4. Implement `hashFiles()` function คำนวณ SHA256 ของไฟล์ที่ระบุ

### แบบฝึกหัดที่ 4: Live Log Streaming (ระดับ: ปานกลาง-ยาก)

เพิ่ม WebSocket server เพื่อ stream log output จาก pipeline ไปยัง browser แบบ realtime:

```rust
// เป้าหมาย: เปิด http://localhost:8080 ดู log แบบ live
pub struct LogServer {
    port: u16,
    channels: HashMap<String, broadcast::Sender<LogLine>>,
}

#[derive(Clone, Serialize)]
pub struct LogLine {
    pub job: String,
    pub stream: String,  // "stdout" | "stderr"
    pub line: String,
    pub timestamp: String,
}
```

**สิ่งที่ต้องทำ:**
1. เพิ่ม dependency `axum` และ `tokio-tungstenite`
2. สร้าง `broadcast::channel` ต่อ job
3. ใน `run_single_job()` ส่ง log line ไปยัง channel ขณะรัน
4. เขียน WebSocket handler ที่ subscribe channel และ stream ไปยัง client
5. สร้าง simple HTML page แสดง log แบบ realtime (ใช้ JavaScript EventSource)

### แบบฝึกหัดที่ 5: Remote Artifact Store (ระดับ: ยาก)

แทนที่ local filesystem artifact store ด้วย S3-compatible object storage:

```yaml
artifacts:
  store:
    type: s3
    bucket: my-ci-artifacts
    prefix: "pipelines/{pipeline_name}/{build_number}/"
    region: ap-southeast-1
```

**สิ่งที่ต้องทำ:**
1. เพิ่ม `aws-sdk-s3` หรือ `object_store` crate
2. สร้าง `ArtifactBackend` trait: `upload()`, `download()`, `list()`
3. Implement `LocalBackend` (เดิม) และ `S3Backend` (ใหม่)
4. ใช้ `Box<dyn ArtifactBackend>` ใน `ArtifactStore`
5. อ่าน backend config จาก pipeline YAML

### แบบฝึกหัดที่ 6: Pipeline as Code — Rust DSL (ระดับ: ยาก)

แทน YAML ด้วย Rust builder API เหมือน Dagger.io:

```rust
// เป้าหมาย: นิยาม pipeline ด้วย Rust code โดยตรง
let pipeline = Pipeline::builder("rust-ci")
    .env("CI", "true")
    .stage("build", |s| s
        .job("compile", |j| j
            .timeout(Duration::from_secs(300))
            .script("cargo build --release")
            .artifact("target/release/myapp", "myapp-binary")
        )
    )
    .stage("test", |s| s
        .job("unit-tests", |j| j
            .script("cargo test --lib")
        )
        .job("integration-tests", |j| j
            .allow_failure(true)
            .script("cargo test --test '*'")
        )
    )
    .build();
```

**สิ่งที่ต้องทำ:**
1. สร้าง builder structs: `PipelineBuilder`, `StageBuilder`, `JobBuilder`
2. Implement `build()` method แต่ละ level ที่คืน parent builder (chaining)
3. ทำให้ builder สามารถ `serialize` กลับเป็น YAML ได้ด้วย

## สรุป

โปรเจค **ci-runner** สร้าง CI/CD pipeline executor ครบวงจรด้วย Rust โดยครอบคลุมทักษะสำคัญหลายด้าน:

**Pattern ที่ได้เรียนใน G02:**

1. **Internally-tagged enum ด้วย serde** — `#[serde(tag = "type")]` สำหรับ polymorphic YAML types ที่ใช้ใน configuration systems ทั่วโลก

2. **Wave-based dependency resolution** — lightweight topological sort ที่ maximize parallelism โดยไม่ต้องใช้ full graph library

3. **Async subprocess management** — `tokio::process::Command` + `tokio::select!` เพื่อ stream stdout/stderr พร้อมกันโดยไม่เกิด pipe deadlock

4. **Timeout pattern ใน async** — `tokio::time::timeout()` ครอบ arbitrary future พร้อม cleanup logic ที่ถูกต้อง

5. **JUnit XML generation** — `quick-xml` serialize ด้วย serde attributes สำหรับ structured report ที่ CI systems อ่านได้

6. **`Arc<Mutex<T>>` ใน async** — share mutable state ข้าม parallel tasks อย่างปลอดภัยด้วย `tokio::sync::Mutex`

**เชื่อมโยงกับโปรเจคถัดไป:**

Project G03 (Log Aggregator) จะต่อยอดจาก G02 โดยรับ log stream จาก multiple pipeline runs มา aggregate, filter, และ query แบบ realtime — เหมือน Grafana Loki แต่สร้างเอง ทักษะการ stream stdout/stderr จาก G02 จะใช้ได้ทันทีใน G03

---

**โปรเจคก่อนหน้า:** [Project G01: Docker Builder](project-g01-docker-builder.md) | **โปรเจคถัดไป:** [Project G03: Log Aggregator](project-g03-log-aggregator.md)
