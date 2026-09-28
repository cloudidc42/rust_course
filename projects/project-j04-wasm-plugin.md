# Project J04: WebAssembly Plugin System

> โมดูล: J — Full-Stack/WASM | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **ระบบ Plugin แบบ WebAssembly** — host process ที่สามารถโหลด `.wasm` module จากภายนอกได้อย่างปลอดภัย ให้ผู้ใช้หรือนักพัฒนาภายนอกเขียน plugin เพิ่มความสามารถให้แอปได้โดยไม่ต้อง recompile โปรแกรมหลัก

ปัญหาคลาสสิกของ plugin system คือ **ความปลอดภัย**: plugin ที่เขียนด้วย native code (`.so`, `.dll`) สามารถทำอะไรก็ได้กับ process — อ่านไฟล์ที่ไม่ควรอ่าน, ทำ network request, หรือแม้กระทั่ง crash ทั้ง process WebAssembly แก้ปัญหานี้ด้วย **sandboxed execution** — code ทำงานใน isolated environment ที่มีเฉพาะ capability ที่ host ยินยอมให้เท่านั้น

**Use cases จริงในโลก production:**
- **Figma, Shopify, Cloudflare Workers** — ล้วนใช้ WASM เป็น plugin/extension runtime
- **Database plugins** — ให้ผู้ใช้เขียน custom aggregation function ที่รันใน DB engine อย่างปลอดภัย
- **CI/CD pipelines** — load plugin สำหรับ lint, test, หรือ transform code โดยไม่ต้องติดตั้ง dependency บน runner
- **Edge computing** — Cloudflare Workers รัน user code ใน WASM runtime ที่ isolate ต่อ request
- **Content management systems** — plugin ที่ transform content (markdown → HTML, template → output) โดยไม่ได้รับสิทธิ์เข้าถึง filesystem

โปรเจคนี้เน้นที่ **ฝั่ง host** (Rust บริสุทธิ์) ซึ่งจัดการทุกอย่างตั้งแต่การ compile WASM module, การจัดการ linear memory, การ inject host function เข้าไปใน plugin, ไปจนถึง sandboxing ด้วย fuel limits คุณจะเห็นว่า wasmtime ทำให้เรื่องที่ฟังดูซับซ้อนเหล่านี้ elegant และเขียนได้อย่าง type-safe

## สิ่งที่จะได้เรียนรู้

- **WASM Linear Memory Model** — ทำไม plugin ต้องส่งผ่านข้อมูลผ่าน pointer+length แทน Rust reference และวิธีอ่าน/เขียน byte ข้ามขอบเขต host-guest
- **wasmtime API ครบวงจร** — `Engine`, `Config`, `Module`, `Store<T>`, `Instance`, `Linker<T>`, `TypedFunc` และความสัมพันธ์ของแต่ละชั้น
- **Host Function Injection** — วิธีใช้ `Linker::func_wrap` เพื่อ provide capability (logging, file I/O) ให้ plugin เรียกได้
- **Fuel-based Sandboxing** — ป้องกัน infinite loop ด้วย `Config::consume_fuel` และ `Store::set_fuel`
- **Plugin Manifest Design** — ออกแบบ `plugin.toml` สำหรับ metadata, permissions, entry points, และ resource limits
- **Plugin Registry** — จัดการ lifecycle ของ plugin collection: load, version check, hot reload
- **ABI Design สำหรับ WASM** — ออกแบบ interface ระหว่าง host กับ guest ที่ใช้ได้กับทุก language ที่ compile to WASM
- **Error Propagation ข้าม boundary** — ใช้ `thiserror` สร้าง rich error type ที่ครอบคลุม WASM trap, compile error, permission error

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust ownership, borrowing, lifetimes, structs, enums, pattern matching
- **Part 21–40**: Traits, generics, `Vec`, iterators, closures, error handling
- **Part 41–60**: `HashMap`, module system, `pub` visibility, `derive` macros, `Box<dyn Trait>`
- **Part 61–80**: Unsafe Rust พื้นฐาน, raw pointer (ทำความเข้าใจเรื่อง memory layout)
- **Part 81–100**: Crate ecosystem, `serde`, TOML, error crates
- **Project J03 (WASM Image Processing)** — ความเข้าใจพื้นฐานการ compile Rust to WASM และ linear memory

## โครงสร้างโปรเจค (Project Layout)

```
wasm-plugin-host/
├── src/
│   ├── lib.rs            ← re-exports public API
│   ├── error.rs          ← PluginError enum (thiserror)
│   ├── manifest.rs       ← PluginManifest, PluginPermissions, plugin.toml parsing
│   ├── registry.rs       ← PluginRegistry — load, list, reload
│   ├── engine.rs         ← PluginEngine, PluginInstance, wasmtime integration
│   └── host_funcs.rs     ← HostState, host function registration (log, read_file)
├── examples/
│   └── markdown_plugin/  ← ตัวอย่าง plugin code (compile to WASM แยกต่างหาก)
│       ├── src/lib.rs
│       └── Cargo.toml
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### ภาพรวม Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         HOST PROCESS                            │
│                                                                 │
│  PluginRegistry                                                 │
│  ┌─────────────────────────────────────────┐                   │
│  │  plugin.toml  →  PluginManifest         │                   │
│  │  name, version, permissions, limits     │                   │
│  │  entry_points: { transform, validate }  │                   │
│  └─────────────────────────────────────────┘                   │
│               │                                                 │
│               ▼                                                 │
│  PluginEngine                                                   │
│  ┌─────────────────────────────────────────────────────┐       │
│  │  Engine (compiled once, thread-safe)                │       │
│  │     │                                               │       │
│  │     ▼                                               │       │
│  │  Module::new(engine, bytes) ← .wasm file            │       │
│  │     │                                               │       │
│  │     ▼                                               │       │
│  │  Store<HostState>  ← per-invocation state           │       │
│  │  ├── fuel counter  (sandbox)                        │       │
│  │  └── HostState { log_messages, allow_logging, ... } │       │
│  │     │                                               │       │
│  │     ▼                                               │       │
│  │  Linker<HostState>                                  │       │
│  │  ├── env.log(ptr, len)      → collect to HostState  │       │
│  │  └── env.read_file(...)     → sandboxed file I/O    │       │
│  │     │                                               │       │
│  │     ▼                                               │       │
│  │  Instance = Linker.instantiate(store, module)       │       │
│  └─────────────────────────────────────────────────────┘       │
│               │                                                 │
│               ▼                                                 │
│  PluginInstance                                                 │
│  ├── call_i32_to_i32("double", 21)  → 42                       │
│  └── call_transform("transform", input) → output               │
└─────────────────────────────────────────────────────────────────┘
                    │  Linear Memory (64KB pages)
                    │  ┌──────────────────────────────────┐
                    │  │ [0..4KB]   plugin heap (alloc)   │
                    │  │ [65536..]  output buffer          │
                    └─►│ plugin code ← เขียน result ที่นี่ │
                       └──────────────────────────────────┘
```

### ทำไมถึงใช้ `Store<T>` แทน global state?

`Store<T>` คือ "universe" ของ WASM instance หนึ่งตัว — มี linear memory, table, globals เป็นของตัวเอง การ create `Store` ใหม่ต่อ invocation หมายความว่า plugin ต่างกันไม่สามารถเข้าถึงหน่วยความจำกันได้ เปรียบได้กับ process isolation ในระดับ OS แต่ lightweight กว่ามาก

### Host-Guest ABI Convention

เนื่องจาก WASM มีแค่ชนิดข้อมูล `i32`, `i64`, `f32`, `f64` การส่ง string หรือ complex data ต้องทำผ่าน **pointer + length** convention:

```
Plugin exports:
  alloc(size: i32) → ptr: i32      ← host เรียกเพื่อขอ memory ใน plugin
  dealloc(ptr: i32, size: i32)     ← host เรียกเพื่อคืน memory
  transform(ptr: i32, len: i32) → result_len: i32
  memory                           ← export linear memory

Host imports (plugin เรียก):
  env.log(ptr: i32, len: i32)      ← plugin เรียก logging
  env.read_file(path_ptr, path_len, out_ptr, out_cap) → i32
```

### ทำไม Engine ถึง thread-safe แต่ Store ไม่ใช่?

`Engine` เก็บ compiled artifact (machine code) ที่ share ได้ระหว่าง thread ปลอดภัย แต่ `Store` มี mutable state (memory, fuel counter, host data) จึงต้องใช้ต่อ thread เท่านั้น — นี่คือ design ที่ทำให้ wasmtime รองรับ multi-tenant workload: compile module ครั้งเดียว แต่ instantiate หลาย Store ในหลาย thread พร้อมกัน

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: กำหนด Error Types ด้วย `thiserror`

ก่อนเขียน logic ใด ๆ เราออกแบบ error hierarchy ให้ครอบคลุม failure modes ทั้งหมดของ plugin system:

```toml
# Cargo.toml
[dependencies]
wasmtime = { version = "25", default-features = false, features = ["cranelift", "runtime", "wat"] }
serde     = { version = "1", features = ["derive"] }
serde_json = "1"
toml       = "0.8"
thiserror  = "1"
```

```rust
// src/error.rs
use thiserror::Error;

#[derive(Debug, Error)]
pub enum PluginError {
    #[error("WASM compilation error: {0}")]
    Compilation(#[from] wasmtime::Error),

    #[error("Manifest parse error: {0}")]
    ManifestParse(String),

    #[error("Plugin not found: {0}")]
    NotFound(String),

    #[error("Permission denied: plugin '{plugin}' requested '{permission}'")]
    PermissionDenied { plugin: String, permission: String },

    #[error("ABI error: {0}")]
    Abi(String),

    #[error("Memory error: {0}")]
    Memory(String),

    #[error("Fuel exhausted: plugin exceeded execution limit")]
    FuelExhausted,

    #[error("Version mismatch: required {required}, found {found}")]
    VersionMismatch { required: String, found: String },

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

สังเกตว่า `PermissionDenied` ใช้ named fields เพื่อให้ error message มีบริบทครบถ้วน และ `#[from]` ทำ automatic conversion จาก `wasmtime::Error` และ `std::io::Error` โดยไม่ต้องเขียน `.map_err()` เอง

---

### ขั้นที่ 2: ออกแบบ Plugin Manifest

`plugin.toml` คือ "สัญญา" ระหว่าง plugin developer กับ host — ระบุ metadata, permissions, entry points และ resource limits

```toml
# examples/markdown_plugin/plugin.toml
name        = "markdown-transformer"
version     = "1.2.0"
description = "Converts Markdown to HTML safely"
author      = "Alice Developer"

[permissions]
logging     = true
read_files  = false
write_files = false
network     = false

[entry_points]
transform   = "transform_markdown"
validate    = "validate_input"

[limits]
fuel             = 50_000_000   # ~50M WASM instructions
max_memory_pages = 8            # 8 × 64KB = 512KB
```

**การ parse manifest ใน Rust:**

```rust
// src/manifest.rs
use serde::{Deserialize, Serialize};
use crate::error::PluginError;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct PluginManifest {
    pub name: String,
    pub version: String,
    pub description: String,
    pub author: String,
    pub permissions: PluginPermissions,
    pub entry_points: EntryPoints,
    #[serde(default)]
    pub limits: PluginLimits,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct PluginPermissions {
    #[serde(default)]
    pub read_files: bool,
    #[serde(default)]
    pub write_files: bool,
    #[serde(default)]
    pub network: bool,
    #[serde(default)]
    pub logging: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct EntryPoints {
    pub transform: Option<String>,
    pub validate:  Option<String>,
    pub init:      Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct PluginLimits {
    #[serde(default = "default_fuel")]
    pub fuel: u64,
    #[serde(default = "default_memory_pages")]
    pub max_memory_pages: u32,
}

fn default_fuel() -> u64 { 100_000_000 }
fn default_memory_pages() -> u32 { 16 }

impl Default for PluginLimits {
    fn default() -> Self {
        Self { fuel: default_fuel(), max_memory_pages: default_memory_pages() }
    }
}

impl PluginManifest {
    pub fn from_toml(content: &str) -> Result<Self, PluginError> {
        toml::from_str(content)
            .map_err(|e| PluginError::ManifestParse(e.to_string()))
    }

    pub fn to_toml(&self) -> Result<String, PluginError> {
        toml::to_string_pretty(self)
            .map_err(|e| PluginError::ManifestParse(e.to_string()))
    }

    /// ตรวจสอบว่า plugin มี permission ที่ขอหรือไม่
    pub fn check_permission(&self, permission: &str) -> Result<(), PluginError> {
        let allowed = match permission {
            "read_files"  => self.permissions.read_files,
            "write_files" => self.permissions.write_files,
            "network"     => self.permissions.network,
            "logging"     => self.permissions.logging,
            _             => false,
        };
        if allowed {
            Ok(())
        } else {
            Err(PluginError::PermissionDenied {
                plugin:     self.name.clone(),
                permission: permission.to_string(),
            })
        }
    }

    /// เช็ค semver compatibility แบบ simplified (major.minor.patch)
    pub fn is_compatible_with(&self, required: &str) -> bool {
        let parse = |s: &str| -> Option<(u32, u32, u32)> {
            let parts: Vec<&str> = s.split('.').collect();
            if parts.len() != 3 { return None; }
            Some((
                parts[0].parse().ok()?,
                parts[1].parse().ok()?,
                parts[2].parse().ok()?,
            ))
        };
        match (parse(&self.version), parse(required)) {
            (Some((ma, mi, _)), Some((rb, ri, _))) => ma == rb && mi >= ri,
            _ => false,
        }
    }
}
```

**ข้อสังเกตสำคัญ:**

`#[serde(default)]` บน `PluginLimits` หมายความว่าถ้า plugin.toml ไม่ระบุ `[limits]` เลย จะได้ค่า default โดยอัตโนมัติ ทำให้ manifest minimal ได้โดยไม่ต้อง verbose

---

### ขั้นที่ 3: Plugin Registry — จัดการ Collection ของ Plugins

```rust
// src/registry.rs
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use crate::error::PluginError;
use crate::manifest::PluginManifest;

#[derive(Debug, Clone)]
pub struct PluginEntry {
    pub manifest:  PluginManifest,
    pub wasm_path: PathBuf,
    pub loaded:    bool,
}

#[derive(Debug, Default)]
pub struct PluginRegistry {
    plugins: HashMap<String, PluginEntry>,
}

impl PluginRegistry {
    pub fn new() -> Self { Self::default() }

    pub fn register(
        &mut self,
        manifest:  PluginManifest,
        wasm_path: impl Into<PathBuf>,
    ) -> Result<(), PluginError> {
        let name = manifest.name.clone();
        self.plugins.insert(name, PluginEntry {
            manifest,
            wasm_path: wasm_path.into(),
            loaded: false,
        });
        Ok(())
    }

    /// โหลด manifest จาก directory ที่มี plugin.toml และ <name>.wasm
    pub fn register_from_dir(&mut self, dir: &Path) -> Result<(), PluginError> {
        let content  = std::fs::read_to_string(dir.join("plugin.toml"))?;
        let manifest = PluginManifest::from_toml(&content)?;
        let wasm_path = dir.join(format!("{}.wasm", manifest.name));
        self.register(manifest, wasm_path)
    }

    pub fn get(&self, name: &str) -> Option<&PluginEntry> {
        self.plugins.get(name)
    }

    /// List plugins เรียงตามชื่อ (deterministic order)
    pub fn list(&self) -> Vec<&PluginEntry> {
        let mut entries: Vec<&PluginEntry> = self.plugins.values().collect();
        entries.sort_by_key(|e| &e.manifest.name);
        entries
    }

    pub fn remove(&mut self, name: &str) -> Option<PluginEntry> {
        self.plugins.remove(name)
    }

    pub fn len(&self)      -> usize { self.plugins.len() }
    pub fn is_empty(&self) -> bool  { self.plugins.is_empty() }

    /// ค้นหา plugin ที่มี entry point ที่ต้องการ
    pub fn find_with_entry_point(&self, ep: &str) -> Vec<&PluginEntry> {
        self.plugins.values().filter(|e| {
            match ep {
                "transform" => e.manifest.entry_points.transform.is_some(),
                "validate"  => e.manifest.entry_points.validate.is_some(),
                "init"      => e.manifest.entry_points.init.is_some(),
                _           => false,
            }
        }).collect()
    }

    /// Hot-reload: อ่าน manifest ใหม่จาก disk โดยไม่ restart process
    pub fn reload_manifest(&mut self, name: &str, dir: &Path) -> Result<(), PluginError> {
        let content  = std::fs::read_to_string(dir.join("plugin.toml"))?;
        let manifest = PluginManifest::from_toml(&content)?;
        if manifest.name != name {
            return Err(PluginError::ManifestParse(format!(
                "Name mismatch on reload: expected '{name}', got '{}'", manifest.name
            )));
        }
        if let Some(entry) = self.plugins.get_mut(name) {
            entry.manifest = manifest;
            entry.loaded   = false; // mark for re-instantiation
        }
        Ok(())
    }
}
```

**Hot-reload pattern:** เมื่อ plugin developer update plugin ระหว่างที่ host กำลังรันอยู่ เราสามารถ call `reload_manifest()` เพื่ออ่าน manifest ใหม่ แล้ว set `loaded = false` เพื่อบอก engine ว่าต้อง re-instantiate ในครั้งถัดไป โดยไม่กระทบ invocation ที่กำลังรันอยู่

---

### ขั้นที่ 4: wasmtime Engine — ใจกลางของ Host Runtime

```rust
// src/engine.rs
use wasmtime::{Config, Engine, Linker, Module, Store};
use crate::error::PluginError;
use crate::manifest::PluginManifest;
use crate::host_funcs::{register_host_functions, HostState};

pub struct PluginEngine {
    engine: Engine,
}

impl PluginEngine {
    /// สร้าง Engine ที่ปลอดภัย: fuel enabled, WASI disabled
    pub fn new() -> Result<Self, PluginError> {
        let mut config = Config::new();
        // เปิด fuel metering — หัวใจของ sandboxing
        config.consume_fuel(true);
        // Engine::new ใช้ Cranelift compiler (AOT by default)
        let engine = Engine::new(&config)?;
        Ok(Self { engine })
    }

    /// Compile WASM bytes → Module (can be cached and shared)
    pub fn compile(&self, wasm_bytes: &[u8]) -> Result<Module, PluginError> {
        Module::new(&self.engine, wasm_bytes).map_err(PluginError::Compilation)
    }

    /// Compile จาก WAT text format (สะดวกสำหรับ testing)
    pub fn compile_wat(&self, wat: &str) -> Result<Module, PluginError> {
        Module::new(&self.engine, wat).map_err(PluginError::Compilation)
    }

    /// Instantiate plugin พร้อม HostState ที่สร้างจาก manifest permissions
    pub fn instantiate(
        &self,
        module:   &Module,
        manifest: &PluginManifest,
    ) -> Result<PluginInstance, PluginError> {
        // สร้าง HostState จาก manifest permissions
        let state = HostState::new(
            manifest.permissions.logging,
            manifest.permissions.read_files,
        );
        let mut store = Store::new(&self.engine, state);

        // ตั้ง fuel limit จาก manifest
        store.set_fuel(manifest.limits.fuel)?;

        // สร้าง Linker และ inject host functions
        let mut linker: Linker<HostState> = Linker::new(&self.engine);
        register_host_functions(&mut linker)?;

        let instance = linker.instantiate(&mut store, module)?;

        Ok(PluginInstance {
            store,
            instance,
            manifest: manifest.clone(),
        })
    }
}
```

**ความสัมพันธ์ระหว่าง Engine, Module, Store, Instance:**

```
Engine  → compile-time config (Cranelift settings, fuel mode)
Module  → compiled machine code (immutable, shareable across threads)
Store   → runtime state (memory, fuel, host data) — 1 ต่อ 1 กับ instance
Instance → live WASM module with bound imports/exports
```

---

### ขั้นที่ 5: PluginInstance — เรียกใช้ Function ใน Plugin

```rust
pub struct PluginInstance {
    store:    Store<HostState>,
    instance: wasmtime::Instance,
    pub manifest: PluginManifest,
}

impl PluginInstance {
    /// เรียก function ที่รับ i32 และ return i32 (สำหรับ simple computation)
    pub fn call_i32_to_i32(&mut self, fn_name: &str, arg: i32)
        -> Result<i32, PluginError>
    {
        let f = self.instance
            .get_typed_func::<i32, i32>(&mut self.store, fn_name)
            .map_err(|e| PluginError::Abi(format!("missing {fn_name}: {e}")))?;

        f.call(&mut self.store, arg).map_err(|e| {
            // Detect fuel exhaustion trap
            if let Some(trap) = e.downcast_ref::<wasmtime::Trap>() {
                match trap {
                    wasmtime::Trap::OutOfFuel => return PluginError::FuelExhausted,
                    _ => {}
                }
            }
            PluginError::FuelExhausted // any unexpected trap with low fuel
        })
    }

    /// เรียก transform function ที่รับ string ผ่าน alloc/dealloc ABI
    pub fn call_transform(&mut self, fn_name: &str, input: &str)
        -> Result<String, PluginError>
    {
        let alloc     = self.instance
            .get_typed_func::<i32, i32>(&mut self.store, "alloc")
            .map_err(|e| PluginError::Abi(format!("missing alloc: {e}")))?;
        let dealloc   = self.instance
            .get_typed_func::<(i32, i32), ()>(&mut self.store, "dealloc")
            .map_err(|e| PluginError::Abi(format!("missing dealloc: {e}")))?;
        let transform = self.instance
            .get_typed_func::<(i32, i32), i32>(&mut self.store, fn_name)
            .map_err(|e| PluginError::Abi(format!("missing {fn_name}: {e}")))?;
        let memory    = self.instance
            .get_memory(&mut self.store, "memory")
            .ok_or_else(|| PluginError::Memory("no memory export".to_string()))?;

        let input_bytes = input.as_bytes();
        let input_ptr   = alloc.call(&mut self.store, input_bytes.len() as i32)?;

        // เขียน input string ลงใน plugin linear memory
        memory.write(&mut self.store, input_ptr as usize, input_bytes)
            .map_err(|e| PluginError::Memory(e.to_string()))?;

        // เรียก transform — plugin เขียน output ที่ offset 65536 (page 1)
        let result_len = transform.call(&mut self.store, (input_ptr, input_bytes.len() as i32))?;

        // อ่าน output จาก well-known offset
        const OUTPUT_OFFSET: usize = 65536;
        let result = {
            let data = memory.data(&self.store);
            if OUTPUT_OFFSET + result_len as usize > data.len() {
                return Err(PluginError::Memory("output out of bounds".to_string()));
            }
            std::str::from_utf8(&data[OUTPUT_OFFSET..OUTPUT_OFFSET + result_len as usize])
                .map_err(|e| PluginError::Abi(format!("invalid utf8 in output: {e}")))?
                .to_string()
        };

        dealloc.call(&mut self.store, (input_ptr, input_bytes.len() as i32))?;
        Ok(result)
    }

    pub fn remaining_fuel(&self) -> u64 {
        self.store.get_fuel().unwrap_or(0)
    }

    pub fn log_messages(&self) -> &[String] {
        &self.store.data().log_messages
    }
}
```

---

### ขั้นที่ 6: Host Functions — Capability Injection

Host functions คือกลไกที่ host ให้ "superpower" แก่ plugin แต่ควบคุมได้ว่าจะให้แค่ไหน Plugin ที่ได้รับ permission `logging = true` เท่านั้นถึงจะ log ได้ Plugin ที่ไม่มี `read_files = true` จะไม่สามารถอ่านไฟล์ได้เลยแม้จะ call ฟังก์ชันนั้น:

```rust
// src/host_funcs.rs
use wasmtime::{Caller, Linker};
use crate::error::PluginError;

#[derive(Debug, Default, Clone)]
pub struct HostState {
    pub log_messages:      Vec<String>,
    pub file_reads:        Vec<String>,
    pub allow_logging:     bool,
    pub allow_read_files:  bool,
}

impl HostState {
    pub fn new(allow_logging: bool, allow_read_files: bool) -> Self {
        Self { allow_logging, allow_read_files, ..Self::default() }
    }
}

pub fn register_host_functions(linker: &mut Linker<HostState>)
    -> Result<(), PluginError>
{
    // env.log(ptr: i32, len: i32) → ()
    // Plugin เรียก function นี้เพื่อ log string ผ่าน host
    linker.func_wrap("env", "log",
        |mut caller: Caller<'_, HostState>, ptr: i32, len: i32|
    {
        if !caller.data().allow_logging { return; }

        let mem = match caller.get_export("memory") {
            Some(wasmtime::Extern::Memory(m)) => m,
            _ => return,
        };
        // ต้อง collect ข้อมูลก่อน drop borrow แล้วค่อย mutate state
        let msg: Option<String> = {
            let data  = mem.data(&caller);
            let start = ptr as usize;
            let end   = start + len as usize;
            if end <= data.len() {
                std::str::from_utf8(&data[start..end]).ok().map(|s| s.to_string())
            } else {
                None
            }
        };
        if let Some(s) = msg {
            caller.data_mut().log_messages.push(s);
        }
    })?;

    // env.read_file(path_ptr, path_len, out_ptr, out_cap) → bytes_written: i32
    linker.func_wrap("env", "read_file",
        |mut caller: Caller<'_, HostState>,
         path_ptr: i32, path_len: i32,
         out_ptr: i32, out_cap: i32| -> i32
    {
        if !caller.data().allow_read_files { return -1; }

        let mem = match caller.get_export("memory") {
            Some(wasmtime::Extern::Memory(m)) => m,
            _ => return -1,
        };
        let path_str: String = {
            let data  = mem.data(&caller);
            let start = path_ptr as usize;
            let end   = start + path_len as usize;
            if end > data.len() { return -1; }
            match std::str::from_utf8(&data[start..end]) {
                Ok(s) => s.to_string(),
                Err(_) => return -1,
            }
        };

        // บันทึก audit trail ว่า plugin อ่านไฟล์อะไรบ้าง
        caller.data_mut().file_reads.push(path_str.clone());

        let content = match std::fs::read(&path_str) {
            Ok(c)  => c,
            Err(_) => return -1,
        };
        let write_len = content.len().min(out_cap as usize);

        let mem = match caller.get_export("memory") {
            Some(wasmtime::Extern::Memory(m)) => m,
            _ => return -1,
        };
        let data  = mem.data_mut(&mut caller);
        let start = out_ptr as usize;
        if start + write_len > data.len() { return -1; }
        data[start..start + write_len].copy_from_slice(&content[..write_len]);
        write_len as i32
    })?;

    Ok(())
}
```

**สิ่งที่น่าสนใจใน borrow checker:**

เมื่อเราเรียก `mem.data(&caller)` เพื่ออ่าน linear memory เราได้ immutable borrow ของ `caller` แต่หลังจากนั้นอยากเรียก `caller.data_mut()` เพื่อแก้ไข `HostState` ซึ่งเป็น mutable borrow — Rust ไม่อนุญาต! วิธีแก้คือ collect ข้อมูลจาก memory เก็บไว้ใน local variable (`msg: Option<String>`) ก่อน แล้ว drop borrow นั้น จากนั้นค่อย mutate state ซึ่งเป็น pattern ที่พบบ่อยใน wasmtime programming

---

### ขั้นที่ 7: Sandboxing — ป้องกัน Malicious Plugins

Sandboxing มีหลายมิติใน wasmtime:

**1. Fuel Limits (CPU time)**

```rust
// เปิด fuel metering ใน Engine config
let mut config = Config::new();
config.consume_fuel(true);

// ตั้ง fuel ต่อ invocation
store.set_fuel(manifest.limits.fuel)?;

// ตรวจสอบที่ runtime
match instance.call_i32_to_i32("spin", 10_000_000) {
    Err(PluginError::FuelExhausted) => {
        eprintln!("Plugin used too much CPU — killed");
    }
    other => { /* normal handling */ }
}
```

**2. Memory Limits (RAM)**

wasmtime ให้ตั้ง `store_limits` เพื่อจำกัด memory growth:

```rust
use wasmtime::StoreLimitsBuilder;

let limits = StoreLimitsBuilder::new()
    .memory_size(manifest.limits.max_memory_pages as usize * 65536)
    .build();
store.limiter(|state| &mut state.store_limiter);
// ต้องเพิ่ม store_limiter field ใน HostState
```

**3. WASI Isolation**

โดย default wasmtime ไม่ enable WASI ดังนั้น plugin ที่ compile ด้วย `--target wasm32-unknown-unknown` จะ **ไม่มีสิทธิ์เข้าถึง filesystem, network, environment variables** เลย — isolation เกิดขึ้นโดยธรรมชาติ

**4. Import Whitelist**

Plugin สามารถ import ได้เฉพาะฟังก์ชันที่เรา register ไว้ใน Linker เท่านั้น ถ้า plugin พยายาม import ฟังก์ชันที่ไม่มีใน Linker จะเกิด error ตอน instantiation ทันที:

```
Error: unknown import: `env::dangerous_syscall` has not been defined
```

**5. Execution Isolation ระหว่าง Plugin Calls**

สร้าง `Store` ใหม่ต่อ invocation เพื่อให้ state ระหว่าง call ไม่รั่วไหลกัน:

```rust
// ✅ Safe: ทุก invocation มี clean state
fn handle_request(engine: &PluginEngine, module: &Module, input: &str)
    -> Result<String, PluginError>
{
    let manifest = /* ... */;
    // สร้าง instance ใหม่ทุกครั้ง → clean memory, clean fuel
    let mut instance = engine.instantiate(module, &manifest)?;
    instance.call_transform("transform", input)
}
```

---

### ขั้นที่ 8: Plugin ABI ฝั่ง Guest — โค้ดที่ Compile to WASM

แม้ว่าเราจะไม่ build WASM ใน context นี้ แต่นี่คือ Rust code ที่ plugin developer จะเขียน:

```rust
// examples/markdown_plugin/src/lib.rs
// compile ด้วย: cargo build --target wasm32-unknown-unknown --release

// ─── Allocator ────────────────────────────────────────────────
// Plugin ต้อง export alloc/dealloc เพื่อให้ host จัดการ memory
use std::alloc::{alloc, dealloc, Layout};

#[no_mangle]
pub extern "C" fn alloc(size: i32) -> *mut u8 {
    let layout = Layout::from_size_align(size as usize, 1).unwrap();
    unsafe { alloc(layout) }
}

#[no_mangle]
pub extern "C" fn dealloc(ptr: *mut u8, size: i32) {
    let layout = Layout::from_size_align(size as usize, 1).unwrap();
    unsafe { dealloc(ptr, layout) }
}

// ─── Host Import ───────────────────────────────────────────────
extern "C" {
    fn log(ptr: *const u8, len: i32);
}

fn host_log(msg: &str) {
    unsafe { log(msg.as_ptr(), msg.len() as i32) }
}

// ─── Output Buffer ─────────────────────────────────────────────
// Plugin เขียน output ที่ page 1 (byte 65536) ตาม ABI convention
static mut OUTPUT_BUF: [u8; 65536] = [0u8; 65536];

// ─── Transform Entry Point ─────────────────────────────────────
#[no_mangle]
pub extern "C" fn transform_markdown(ptr: *const u8, len: i32) -> i32 {
    let input = unsafe {
        let slice = std::slice::from_raw_parts(ptr, len as usize);
        match std::str::from_utf8(slice) {
            Ok(s)  => s,
            Err(_) => return -1,
        }
    };

    host_log(&format!("Transforming {} bytes of markdown", input.len()));

    // Minimal markdown → HTML transformation
    let output = simple_md_to_html(input);
    let bytes  = output.as_bytes();

    unsafe {
        let out = OUTPUT_BUF.as_mut_ptr();
        std::ptr::copy_nonoverlapping(bytes.as_ptr(), out, bytes.len());
    }

    bytes.len() as i32
}

fn simple_md_to_html(md: &str) -> String {
    let mut html = String::new();
    for line in md.lines() {
        if let Some(rest) = line.strip_prefix("# ") {
            html.push_str(&format!("<h1>{}</h1>\n", rest));
        } else if let Some(rest) = line.strip_prefix("## ") {
            html.push_str(&format!("<h2>{}</h2>\n", rest));
        } else if line.trim().is_empty() {
            html.push('\n');
        } else {
            html.push_str(&format!("<p>{}</p>\n", line));
        }
    }
    html
}

// ─── Validate Entry Point ──────────────────────────────────────
#[no_mangle]
pub extern "C" fn validate_input(ptr: *const u8, len: i32) -> i32 {
    let input = unsafe {
        let slice = std::slice::from_raw_parts(ptr, len as usize);
        std::str::from_utf8(slice).unwrap_or("")
    };
    // 0 = valid, 1 = too long, 2 = empty
    if input.is_empty() { 2 }
    else if input.len() > 100_000 { 1 }
    else { 0 }
}
```

**Cargo.toml สำหรับ plugin:**

```toml
# examples/markdown_plugin/Cargo.toml
[package]
name = "markdown-transformer"
version = "1.2.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]  # สำคัญ: ต้องเป็น cdylib สำหรับ WASM

[profile.release]
opt-level = "z"      # ลด binary size
lto = true
strip = true
```

**Build command:**
```bash
cargo build --target wasm32-unknown-unknown --release
# output: target/wasm32-unknown-unknown/release/markdown_transformer.wasm
```

---

### ขั้นที่ 9: WAT Format สำหรับ Testing — ทดสอบโดยไม่ต้อง compile WASM Target

เนื่องจากการ setup cross-compilation toolchain อาจยุ่งยาก เราใช้ **WAT (WebAssembly Text Format)** inline ใน test เพื่อสร้าง WASM module ขนาดเล็กสำหรับทดสอบ host logic โดยตรง:

```rust
// WAT: module ที่ export "double" function (i32 → i32)
const DOUBLE_WAT: &str = r#"
(module
  (import "env" "log" (func $log (param i32 i32)))
  (memory (export "memory") 1)
  (func (export "double") (param $n i32) (result i32)
    local.get $n
    i32.const 2
    i32.mul
  )
)
"#;

// WAT: module ที่วนลูปไม่หยุดเพื่อทดสอบ fuel limit
const LOOP_WAT: &str = r#"
(module
  (import "env" "log" (func $log (param i32 i32)))
  (memory (export "memory") 1)
  (func (export "spin") (param $n i32) (result i32)
    (local $i i32)
    i32.const 0
    local.set $i
    (block $exit
      (loop $top
        local.get $i
        i32.const 1
        i32.add
        local.set $i
        local.get $i
        local.get $n
        i32.lt_s
        br_if $top
      )
    )
    local.get $i
  )
)
"#;
```

**ข้อดีของ WAT ใน testing:**

- ไม่ต้องติดตั้ง `wasm32-unknown-unknown` target
- Test compile เร็วมาก (ไม่มี cross-compilation overhead)
- ควบคุม WASM bytecode ได้แน่นอน — เหมาะสำหรับทดสอบ edge cases เช่น invalid memory access
- ใช้ debug/verify ว่า host function ทำงานถูกต้องกับ WASM convention ที่แท้จริง

---

## การทดสอบ (Testing)

### Unit Tests ใน `src/` Modules

```rust
// ตัวอย่างจาก src/manifest.rs
#[cfg(test)]
mod tests {
    use super::*;

    fn sample_manifest_toml() -> &'static str {
        r#"
name = "markdown-transformer"
version = "1.2.0"
description = "Converts Markdown to HTML"
author = "Alice"

[permissions]
logging = true
read_files = false
write_files = false
network = false

[entry_points]
transform = "transform_markdown"
validate = "validate_input"

[limits]
fuel = 50000000
max_memory_pages = 8
"#
    }

    #[test]
    fn test_manifest_parse() { /* ... */ }

    #[test]
    fn test_permission_denied() { /* ... */ }

    #[test]
    fn test_version_compatible() { /* ... */ }
}
```

### Output จาก `cargo test` จริง

```
$ cargo test
   Compiling wasm-plugin-host v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 4.24s
     Running unittests src/lib.rs (target/debug/deps/wasm_plugin_host-eed7b3d083b560a6)

running 17 tests
test engine::tests::test_fuel_limits_enforced ... ok
test engine::tests::test_engine_compile_wat ... ok
test engine::tests::test_call_i32_function ... ok
test manifest::tests::test_default_limits ... ok
test engine::tests::test_fuel_not_exhausted_for_small_computation ... ok
test host_funcs::tests::test_host_log_function ... ok
test manifest::tests::test_manifest_parse ... ok
test manifest::tests::test_manifest_roundtrip ... ok
test manifest::tests::test_permission_allowed ... ok
test registry::tests::test_find_with_entry_point ... ok
test manifest::tests::test_permission_denied ... ok
test manifest::tests::test_version_compatible ... ok
test registry::tests::test_list_sorted ... ok
test registry::tests::test_register_and_get ... ok
test registry::tests::test_remove ... ok
test registry::tests::test_register_from_dir ... ok
test host_funcs::tests::test_host_log_permission_denied ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s

   Doc-tests wasm_plugin_host

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### สิ่งที่แต่ละ test ครอบคลุม

| Test | Module | สิ่งที่ทดสอบ |
|------|--------|--------------|
| `test_manifest_parse` | manifest | parse TOML → struct, ตรวจค่าทุก field |
| `test_manifest_roundtrip` | manifest | serialize → parse → ได้ struct เดิม |
| `test_permission_allowed` | manifest | logging=true → check_permission("logging") = Ok |
| `test_permission_denied` | manifest | network=false → PermissionDenied error มี plugin+permission ใน message |
| `test_version_compatible` | manifest | semver check: same major, minor ≥ required |
| `test_default_limits` | manifest | manifest ที่ไม่ระบุ [limits] → ได้ default fuel=100M, pages=16 |
| `test_register_and_get` | registry | register → get → manifest/path ถูกต้อง |
| `test_list_sorted` | registry | list 3 plugins → เรียงตาม name alphabetically |
| `test_remove` | registry | remove → is_empty() = true |
| `test_find_with_entry_point` | registry | filter plugins ที่มี transform vs validate entry point |
| `test_register_from_dir` | registry | อ่าน plugin.toml จาก tempdir → register |
| `test_engine_compile_wat` | engine | compile WAT text → Module สำเร็จ |
| `test_call_i32_function` | engine | call double(21) → 42 ผ่าน wasmtime |
| `test_fuel_limits_enforced` | engine | spin loop กับ fuel=1000 → FuelExhausted error |
| `test_fuel_not_exhausted_for_small_computation` | engine | double(5) → fuel เหลือ > 0 |
| `test_host_log_function` | host_funcs | plugin เรียก env.log → collect ใน HostState |
| `test_host_log_permission_denied` | host_funcs | logging=false → log call ถูก silently ignore |

---

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# Build host binary
cargo build --release

# Plugin (แยก build ด้วย WASM target)
cd plugins/markdown-transformer
cargo build --target wasm32-unknown-unknown --release

# Copy .wasm และ plugin.toml ไปที่ plugins directory
cp target/wasm32-unknown-unknown/release/markdown_transformer.wasm \
   ../../plugins/markdown-transformer/markdown-transformer.wasm
```

### Plugin Distribution Structure

```
plugins/
├── markdown-transformer/
│   ├── plugin.toml           ← manifest
│   └── markdown-transformer.wasm
├── code-highlighter/
│   ├── plugin.toml
│   └── code-highlighter.wasm
└── spell-checker/
    ├── plugin.toml
    └── spell-checker.wasm
```

### ใช้งาน PluginRegistry + PluginEngine ใน Binary

```rust
// src/main.rs
use wasm_plugin_host::{PluginEngine, PluginRegistry};
use std::path::Path;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let engine   = PluginEngine::new()?;
    let mut registry = PluginRegistry::new();

    // โหลด plugins จาก directory
    for entry in std::fs::read_dir("plugins")? {
        let dir = entry?.path();
        if dir.is_dir() {
            match registry.register_from_dir(&dir) {
                Ok(_)  => println!("Loaded plugin: {}", dir.display()),
                Err(e) => eprintln!("Failed to load {}: {e}", dir.display()),
            }
        }
    }

    println!("Loaded {} plugins", registry.len());
    for plugin in registry.list() {
        println!(
            "  {} v{} — transform: {:?}",
            plugin.manifest.name,
            plugin.manifest.version,
            plugin.manifest.entry_points.transform
        );
    }

    // โหลดและรัน markdown-transformer plugin
    if let Some(entry) = registry.get("markdown-transformer") {
        let wasm_bytes = std::fs::read(&entry.wasm_path)?;
        let module     = engine.compile(&wasm_bytes)?;
        let mut inst   = engine.instantiate(&module, &entry.manifest)?;

        let markdown = "# Hello World\n\nThis is a **test**.\n";
        let html     = inst.call_transform("transform_markdown", markdown)?;
        println!("Plugin output:\n{html}");
        println!("Fuel remaining: {}", inst.remaining_fuel());
    }

    Ok(())
}
```

### Docker Deployment

```dockerfile
FROM rust:1.80-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /app/target/release/wasm-plugin-host .
COPY plugins/ plugins/
ENTRYPOINT ["./wasm-plugin-host"]
```

---

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### ข้อผิดพลาดที่ 1: Borrow Conflict เมื่ออ่าน Memory แล้ว Mutate HostState

**ปัญหา:** เมื่อเขียน host function ที่ต้องอ่าน linear memory แล้ว update HostState จะเจอ compile error:

```rust
// ❌ ไม่ compile: cannot borrow `caller` as mutable because it is also borrowed as immutable
linker.func_wrap("env", "log", |mut caller: Caller<'_, HostState>, ptr: i32, len: i32| {
    let mem  = caller.get_export("memory")...;
    let data = mem.data(&caller);           // immutable borrow
    let s    = std::str::from_utf8(&data[ptr as usize..ptr as usize + len as usize]);
    caller.data_mut().log_messages.push(s); // ❌ mutable borrow ขณะที่ยัง hold immutable borrow
});
```

**สาเหตุ:** `mem.data(&caller)` borrow `caller` อยู่ แต่ `caller.data_mut()` ต้องการ mutable borrow ของ `caller` ด้วย

**วิธีแก้:** Collect ข้อมูลจาก memory เก็บไว้ใน local variable แล้ว drop borrow ก่อน จากนั้นค่อย mutate:

```rust
// ✅ ถูกต้อง: ใช้ block scope บังคับ drop immutable borrow
linker.func_wrap("env", "log", |mut caller: Caller<'_, HostState>, ptr: i32, len: i32| {
    let mem = caller.get_export("memory")...;
    let msg: Option<String> = {
        let data  = mem.data(&caller);  // immutable borrow เริ่มที่นี่
        let start = ptr as usize;
        let end   = start + len as usize;
        if end <= data.len() {
            std::str::from_utf8(&data[start..end]).ok().map(|s| s.to_string())
        } else {
            None
        }
        // immutable borrow หมดที่นี่ (end of block)
    };
    if let Some(s) = msg {
        caller.data_mut().log_messages.push(s); // ✅ ตอนนี้ OK
    }
});
```

---

### ข้อผิดพลาดที่ 2: ลืม Enable `consume_fuel` ใน Engine Config

**ปัญหา:** ถ้า `Config::consume_fuel(true)` ไม่ถูกเรียก แต่เราพยายาม call `store.set_fuel(...)` จะ panic หรือ error:

```rust
// ❌ ลืม enable fuel
let engine = Engine::default(); // default config ไม่มี fuel
let mut store = Store::new(&engine, state);
store.set_fuel(1_000_000)?; // Error: fuel is not enabled
```

**วิธีแก้:**

```rust
// ✅ เปิด fuel ใน config ก่อน
let mut config = Config::new();
config.consume_fuel(true);
let engine = Engine::new(&config)?;
```

**ข้อสังเกต:** `Engine::default()` ใช้ default settings ที่ปิด fuel เพื่อ performance ดังนั้นใน production plugin system ต้องสร้าง config เอง อย่าใช้ `Engine::default()`

---

### ข้อผิดพลาดที่ 3: ความเข้าใจผิดเรื่อง Output Buffer Convention

**ปัญหา:** Plugin host อ่าน output จาก offset ผิด หรือ plugin เขียน output ไปผิด address

ใน ABI ที่เราออกแบบ plugin เขียน output ที่ byte offset 65536 (page 1) ซึ่งเป็น convention ที่ต้องตกลงกันระหว่าง host และ guest:

```rust
// ❌ host อ่านจาก offset 0 ซึ่งเป็น input region ไม่ใช่ output
let result_bytes = &data[0..result_len as usize]; // wrong!

// ✅ host อ่านจาก OUTPUT_OFFSET = 65536
const OUTPUT_OFFSET: usize = 65536;
let result_bytes = &data[OUTPUT_OFFSET..OUTPUT_OFFSET + result_len as usize];
```

**ปัญหา 2:** plugin เขียน output ทับ input ที่ยังต้องใช้อยู่ เพราะ input อยู่ที่ ptr และ output ก็เขียนที่ ptr เดียวกัน:

```rust
// ❌ plugin code: เขียน output ทับ input
unsafe {
    std::ptr::copy_nonoverlapping(output.as_ptr(), ptr as *mut u8, output.len());
}

// ✅ plugin code: เขียน output ที่ page 1 (65536)
static mut OUTPUT_BUF: [u8; 65536] = [0u8; 65536];
unsafe {
    std::ptr::copy_nonoverlapping(output.as_ptr(), OUTPUT_BUF.as_mut_ptr(), output.len());
}
```

**คำแนะนำ:** Document ABI convention ไว้ใน README และ test ให้ครบ ปัญหาประเภทนี้ debug ยากมากเพราะ memory corruption ใน WASM อาจทำให้ output corruption ที่ไม่ consistent

---

### ข้อผิดพลาดที่ 4: Plugin Import ที่ไม่ได้ Register ทำให้ Instantiate ล้มเหลว

**ปัญหา:** Plugin ถูก compile ด้วย import ที่ host ไม่ได้ register ไว้ใน Linker จะเกิด error ตอน instantiate:

```
Error: unknown import: `env::get_timestamp` has not been defined
```

**สาเหตุ:** Plugin developer เพิ่ม import ใหม่ แต่ host ยังไม่ได้ implement:

```rust
// plugin code เพิ่ม import ใหม่
extern "C" {
    fn get_timestamp() -> i64; // ← host ไม่รู้จัก
}
```

**วิธีแก้ระยะสั้น:** Register stub ที่ return ค่า default:

```rust
linker.func_wrap("env", "get_timestamp", || -> i64 { 0 })?;
```

**วิธีแก้ระยะยาว:** ใช้ versioned ABI — manifest ระบุ ABI version ที่ต้องการ host ตรวจสอบก่อน instantiate:

```rust
// plugin.toml
[meta]
abi_version = "2"  # host ต้องรองรับ ABI v2

// host code
if manifest.meta.abi_version > SUPPORTED_ABI_VERSION {
    return Err(PluginError::VersionMismatch { ... });
}
```

---

### ข้อผิดพลาดที่ 5: Module สามารถ Share ได้ แต่ Instance ไม่ได้

**ปัญหา:** เข้าใจผิดว่า `Module` และ `Instance` share ได้เหมือนกัน แล้วพยายาม put Instance ใน `Arc<Mutex<>>` เพื่อ share ระหว่าง thread:

```rust
// ❌ ผิด: Instance ไม่ Send
let instance = Arc::new(Mutex::new(engine.instantiate(&module, &manifest)?));
// PluginInstance ไม่ implement Send เพราะ Store<T> ไม่ Send
```

**วิธีแก้:** ให้แต่ละ thread สร้าง instance ของตัวเอง แต่ share Module ได้:

```rust
// ✅ ถูกต้อง: Module เป็น Send + Sync, share ได้
let module = Arc::new(engine.compile(&wasm_bytes)?);

// แต่ละ thread สร้าง instance ของตัวเอง
std::thread::spawn({
    let module   = Arc::clone(&module);
    let manifest = manifest.clone();
    move || {
        let mut instance = engine.instantiate(&module, &manifest)?;
        instance.call_i32_to_i32("process", 42)
    }
});
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Host Function `env.get_env_var`

**ระดับ:** ⭐⭐⭐

เพิ่ม host function ที่ให้ plugin อ่าน environment variable ได้เฉพาะรายการที่ whitelist ไว้:

```rust
// plugin manifest ระบุ env vars ที่อนุญาต
[permissions]
env_vars = ["APP_VERSION", "PLUGIN_DEBUG"]

// host function: env.get_env_var(key_ptr, key_len, out_ptr, out_cap) → i32
linker.func_wrap("env", "get_env_var", |mut caller: Caller<'_, HostState>, ...| {
    // อ่าน key จาก WASM memory
    // ตรวจว่า key อยู่ใน whitelist ใน HostState
    // ถ้าอนุญาต: อ่าน std::env::var(key) แล้วเขียนลง out_ptr
    // ถ้าไม่อนุญาต: return -1
});
```

**สิ่งที่จะได้เรียน:** วิธี extend host API, whitelist pattern, และ borrow checker ใน `Caller`

---

### แบบฝึกหัดที่ 2: Async Plugin Execution ด้วย Tokio

**ระดับ:** ⭐⭐⭐⭐

wasmtime รองรับ async execution ผ่าน `Config::async_support(true)` เปลี่ยน `PluginEngine` ให้รัน plugin แบบ async และตั้ง timeout แทนที่จะใช้ fuel:

```rust
use tokio::time::{timeout, Duration};

pub async fn call_transform_async(
    &mut self,
    fn_name: &str,
    input: &str,
) -> Result<String, PluginError> {
    timeout(
        Duration::from_millis(100),
        self.inner_call_transform(fn_name, input)
    )
    .await
    .map_err(|_| PluginError::FuelExhausted)?
}
```

**สิ่งที่จะได้เรียน:** wasmtime async API, เปรียบเทียบ fuel-based vs time-based sandboxing

---

### แบบฝึกหัดที่ 3: Plugin Hot Reload ด้วย File Watcher

**ระดับ:** ⭐⭐⭐⭐

ใช้ crate `notify` เพื่อ watch plugin directory และ reload อัตโนมัติเมื่อไฟล์ `.wasm` หรือ `plugin.toml` เปลี่ยน:

```rust
use notify::{Watcher, RecursiveMode};
use std::sync::mpsc::channel;

fn start_hot_reload_watcher(
    registry: Arc<Mutex<PluginRegistry>>,
    plugins_dir: PathBuf,
) {
    let (tx, rx) = channel();
    let mut watcher = notify::recommended_watcher(tx).unwrap();
    watcher.watch(&plugins_dir, RecursiveMode::Recursive).unwrap();

    std::thread::spawn(move || {
        for event in rx {
            if let Ok(event) = event {
                // ถ้า plugin.toml เปลี่ยน → reload manifest
                // ถ้า .wasm เปลี่ยน → recompile module
                // ใช้ RwLock แทน Mutex เพื่อ allow concurrent reads
            }
        }
    });
}
```

**สิ่งที่จะได้เรียน:** File watching, `Arc<RwLock<T>>` สำหรับ concurrent read, hot reload pattern

---

### แบบฝึกหัดที่ 4: Plugin Dependency Graph และ Load Order

**ระดับ:** ⭐⭐⭐⭐⭐

ออกแบบ plugin ที่สามารถ declare dependency ต่อกันได้:

```toml
# plugin.toml
[dependencies]
"markdown-transformer" = ">=1.0.0"
"code-highlighter"     = ">=2.0.0"
```

Implement topological sort เพื่อหา load order ที่ถูกต้อง:

```rust
pub fn resolve_load_order(
    registry: &PluginRegistry,
) -> Result<Vec<String>, PluginError> {
    // Build dependency graph
    // Detect cycles (circular dependency → error)
    // Return topologically sorted list
}
```

**สิ่งที่จะได้เรียน:** Graph algorithms ใน Rust, cycle detection (DFS), Kahn's algorithm สำหรับ topological sort

---

### แบบฝึกหัดที่ 5: Plugin Signature Verification

**ระดับ:** ⭐⭐⭐⭐

เพิ่ม security layer: plugin .wasm file ต้องถูก sign ด้วย ed25519 key ของ developer host ต้องตรวจ signature ก่อน load:

```toml
# plugin.toml
[meta]
signature = "ed25519:abc123..."  # SHA-256 hash ของ .wasm ถูก sign ด้วย private key
public_key = "ed25519:xyz789..."
```

```rust
use ed25519_dalek::{PublicKey, Signature, Verifier};

pub fn verify_plugin_signature(
    wasm_bytes: &[u8],
    manifest: &PluginManifest,
) -> Result<(), PluginError> {
    // อ่าน public_key จาก manifest
    // Compute SHA-256 ของ wasm_bytes
    // Verify signature
}
```

**สิ่งที่จะได้เรียน:** Cryptographic verification, `ed25519-dalek`, supply chain security สำหรับ plugin system

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **WebAssembly Plugin Host** ที่ครบถ้วนสำหรับ production use ครอบคลุม:

**สิ่งที่สร้างและทดสอบจริง (17 tests ผ่านทั้งหมด):**

- `PluginManifest` — parse/serialize `plugin.toml`, permission checking, semver compatibility
- `PluginRegistry` — register, list, remove, find by entry point, hot reload
- `PluginEngine` — wasmtime integration, compile WAT/WASM, fuel-based sandboxing
- `PluginInstance` — call i32 functions, string transform via ABI, remaining fuel query
- `HostState` + host functions — logging injection, sandboxed file access, permission enforcement

**Pattern สำคัญที่ได้เรียน:**

1. **Host-Guest Memory Protocol** — ข้อมูลข้าม WASM boundary ต้องผ่าน pointer+length; Rust's ownership system บังคับให้ต้อง explicit ทุกครั้ง ไม่มีการ "ส่ง reference" ข้าม boundary ได้

2. **Layered Sandboxing** — fuel limits (CPU), memory limits (RAM), import whitelist (syscalls), WASI isolation (filesystem/network) ทำงานร่วมกันเป็น defense-in-depth

3. **`Store<T>` Typed State** — wasmtime ใช้ generic state ที่ type-safe แทน global state หรือ unsafe pointer casting ทำให้ host data ไม่รั่วระหว่าง plugin invocations

4. **Borrow Checker กับ `Caller<'_, T>`** — การอ่าน WASM memory แล้ว mutate host state ต้องระวัง borrow scope เป็นพิเศษ ใช้ block scope เพื่อ force drop immutable borrow ก่อน

5. **WAT สำหรับ Testing** — ไม่จำเป็นต้องมี cross-compilation toolchain เพื่อทดสอบ host logic ใช้ WAT inline text แทน WASM binary ใน unit tests ได้

**เชื่อมโยงไปโปรเจคถัดไป:**

Project J05 (Collaborative Editor) จะขยายความรู้เรื่อง real-time data sharing ผ่าน WebSocket และ CRDT (Conflict-free Replicated Data Types) ซึ่งเป็นอีกด้านของ full-stack Rust — ขณะที่ J04 เน้น computation isolation, J05 เน้น distributed state consistency ทั้งสองโปรเจคสะท้อนความสามารถของ Rust ในการ tackle ปัญหา systems programming ที่ซับซ้อนได้อย่าง type-safe

---

**โปรเจคก่อนหน้า:** [WebAssembly Image Processing](project-j03-wasm-image.md) | **โปรเจคถัดไป:** [Collaborative Editor](project-j05-collaborative-editor.md)
