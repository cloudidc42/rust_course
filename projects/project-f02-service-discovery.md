# Project F02: Service Discovery System

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

**Service Discovery** คือกลไกที่ช่วยให้ services ใน microservices architecture สามารถค้นหาและติดต่อกันได้โดยอัตโนมัติ โดยไม่ต้องฝัง IP address และ port ไว้ใน config แบบ hardcode — เพราะใน production ที่มี containers สร้าง/ทำลายตลอดเวลา หรือ auto-scaling เพิ่ม instance เมื่อ load สูง ที่อยู่ของ service เปลี่ยนแปลงตลอด

โปรเจคนี้สร้างระบบ service registry และ discovery ที่ทำงานในหน่วยความจำ (in-memory) เช่นเดียวกับ **HashiCorp Consul** หรือ **Netflix Eureka** แต่เขียนด้วย Rust และไม่ต้องมี network จริง เหมาะสำหรับการเรียนรู้ pattern ทั้งหมดก่อนที่จะนำไปใช้จริงในระบบ distributed

**Use case จริงในโลก production:**
- Kubernetes service mesh (สิ่งที่ kube-dns ทำให้)
- Consul service discovery ใน HashiCorp stack
- Netflix Eureka ใน Spring Cloud
- AWS Cloud Map สำหรับ ECS/EKS workloads
- Istio control plane service registry

**สิ่งที่แตกต่างจาก project ก่อนหน้า (F01 Raft KV):** F01 สนใจ *consensus* ว่าข้อมูลชุดเดียวกันจะตรงกันได้อย่างไรในหลาย node, ส่วน F02 สนใจ *routing* ว่า client จะหา server ถูกตัวได้อย่างไรเมื่อมีหลาย instance ที่เปลี่ยนแปลงตลอดเวลา

## สิ่งที่จะได้เรียนรู้

- **DashMap** — concurrent HashMap ที่ไม่ต้องใช้ `Mutex` ห่อทั้ง map, เหมาะกับ read-heavy registry
- **tokio watch channel** — สำหรับ broadcast event ไปยัง subscriber หลายตัวในแบบ last-value semantics
- **TTL-based health checking** — pattern สำหรับ detect dead services โดยใช้ heartbeat + expiry
- **Consistent hashing (ketama ring)** — virtual nodes บน hash ring สำหรับ sticky routing
- **Weighted random selection** — probability distribution สำหรับ traffic splitting
- **Extension trait pattern** — เพิ่ม behavior ให้ type ที่มีอยู่แล้วโดยไม่แก้ code เดิม
- **DNS SRV record** — format มาตรฐานสำหรับ service endpoint discovery
- **Arc<DashMap<K, V>>** — shared state ที่ scale ได้ในสภาพแวดล้อม multi-threaded

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50**: async/await, tokio runtime — ใช้สำหรับ background reaper task และ watch channel
- **Part 96–100**: `Arc`, `Mutex`, `RwLock` — shared state ระหว่าง threads
- **Part 101–105**: trait objects, extension traits — `Balancer` trait และ `DiscoveryExt`
- **Part 51–55**: `HashMap`, `BTreeMap` — ใช้สร้าง ketama ring ใน consistent hashing
- **Part 41–45**: closures, iterators — `filter`, `map` สำหรับ query engine
- **Part 86–90**: `serde` serialization — สำหรับ `ServiceInstance` JSON representation

## โครงสร้างโปรเจค (Project Layout)

```
service-disc/
├── src/
│   ├── lib.rs            # re-export ทุก module
│   ├── main.rs           # demo binary
│   ├── registry.rs       # ServiceInstance, Registry, RegistryError
│   ├── health.rs         # HealthStatus, HealthCheck, TTL logic
│   ├── discovery.rs      # ServiceQuery, DiscoveryExt trait
│   ├── watcher.rs        # ServiceEvent, Watcher, watch channel
│   ├── balancer.rs       # RoundRobinBalancer, WeightedRandomBalancer, ConsistentHashBalancer
│   └── dns.rs            # SrvRecord, DnsCache
└── Cargo.toml
```

## การออกแบบ (Architecture & Design)

### Data Flow Overview

```
Service Instance (client library)
        │
        │ register(ServiceInstance)
        ▼
   ┌─────────────────────────────────────┐
   │            Registry                  │
   │  DashMap<String, RegistryEntry>      │
   │  ┌──────────────────────────────┐   │
   │  │ RegistryEntry                │   │
   │  │  - instance: ServiceInstance │   │
   │  │  - health: HealthCheck       │   │
   │  │    - status: HealthStatus    │   │
   │  │    - last_heartbeat: Instant │   │
   │  │    - ttl: Duration           │   │
   │  └──────────────────────────────┘   │
   └────────────┬────────────────────────┘
                │                    │
                │ watch channel       │ reap_expired()
                ▼                    ▼
          Watcher/Subscribers    Background Reaper
          (ServiceEvent)         (tokio task)
                │
                ▼
     ServiceEvent::Registered
     ServiceEvent::Deregistered
     ServiceEvent::HealthChanged

Consumer (service client)
        │
        │ query(ServiceQuery { name, tag, health })
        ▼
   DiscoveryExt.query() → Vec<ServiceInstance>
        │
        ▼
   Load Balancer (Balancer trait)
   ┌─────────────────────────────────────┐
   │  RoundRobinBalancer  → AtomicUsize  │
   │  WeightedRandomBalancer → rand      │
   │  ConsistentHashBalancer → BTreeMap  │
   └─────────────────────────────────────┘
        │
        ▼
   ServiceInstance.address_port() → "10.0.0.1:8080"
        │
        ▼
   DnsCache.put() / .resolve() → SRV records
```

### ทำไมถึงเลือก DashMap แทน `RwLock<HashMap>`

ใน registry ที่มี read มากกว่า write อย่างชัดเจน (ทุก service request ต้อง lookup, แต่ register/deregister เกิดน้อยกว่า) การใช้ `RwLock<HashMap>` จะทำให้ write lock block readers ทั้งหมด DashMap แก้ปัญหานี้ด้วย **sharding** — แบ่ง internal map ออกเป็น N shard แต่ละ shard มี `RwLock` ของตัวเอง ทำให้ concurrent access ไม่ต้องรอกันทั้งหมด

```
DashMap (16 shards by default)
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ s0   │ s1   │ s2   │ s3   │ s4   │ s5   │ s6   │ s7   │
│RwLock│RwLock│RwLock│RwLock│RwLock│RwLock│RwLock│RwLock│
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
  hash(key) % 16 → shard index
```

Write ที่ key "user-1" จะ lock เฉพาะ shard ที่ hash("user-1") ชี้ ไม่ block reads ที่ shard อื่น

### ทำไมถึงใช้ `watch` channel แทน `broadcast`

| ลักษณะ | `tokio::sync::watch` | `tokio::sync::broadcast` |
|--------|---------------------|------------------------|
| เก็บ value ล่าสุด | ใช่ | ไม่ใช่ |
| Subscriber ที่ช้า | ได้ค่าล่าสุดเสมอ | อาจ miss events |
| Buffer | ไม่ต้องตั้ง | ต้องกำหนด capacity |
| Use case | "สถานะปัจจุบัน" | "ทุก event สำคัญ" |

สำหรับ service registry การที่ subscriber "เพิ่งต่อใหม่" ต้องการรู้ว่า *ตอนนี้มี service อะไรบ้าง* มากกว่าจะต้องรู้ทุก event ที่ผ่านมา → `watch` เหมาะกว่า

### Consistent Hashing — Ketama Ring

Virtual nodes ช่วยให้การกระจายสม่ำเสมอขึ้น แทนที่จะวาง instance จริงบน ring เพียง 1 จุด เราวาง virtual nodes หลายจุด (เช่น 150 ต่อ instance):

```
          inst-0#0     inst-2#4     inst-1#2
             │             │             │
    0────────●─────────────●─────────────●─────── 2^64-1
         inst-1#0     inst-0#3     inst-2#1
             │             │             │
    ─────────●─────────────●─────────────●────
```

เมื่อต้องการ route request ด้วย key เช่น "user-42":
1. hash("user-42") → ได้ค่า H
2. หาจุดแรกบน ring ที่ >= H (ด้วย `BTreeMap::range`)
3. ถ้าหาไม่พบ wrap around ไปจุดแรก

ผลลัพธ์: key เดิมเสมอได้ instance เดิม (sticky), และเมื่อ instance หนึ่งหายไป มีแค่ keys ที่อยู่ใน segment ของมันเท่านั้นที่ต้อง remap

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Health Status และ TTL-based Health Check

เริ่มจาก module พื้นฐานที่สุด — นิยาม `HealthStatus` และ `HealthCheck` ที่ใช้ `Instant` + `Duration` ในการตรวจสอบ TTL

**`src/health.rs`**

```rust
use serde::{Deserialize, Serialize};
use std::time::{Duration, Instant};

/// สถานะสุขภาพของ service
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum HealthStatus {
    /// ทำงานปกติ
    Passing,
    /// มีปัญหาเล็กน้อยแต่ยังให้บริการได้
    Warning,
    /// ไม่สามารถให้บริการได้
    Critical,
}

impl std::fmt::Display for HealthStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            HealthStatus::Passing => write!(f, "passing"),
            HealthStatus::Warning => write!(f, "warning"),
            HealthStatus::Critical => write!(f, "critical"),
        }
    }
}

/// ข้อมูล health check สำหรับ service instance หนึ่ง
#[derive(Debug, Clone)]
pub struct HealthCheck {
    pub status: HealthStatus,
    pub last_heartbeat: Instant,
    pub ttl: Duration,
}

impl HealthCheck {
    pub fn new(ttl: Duration) -> Self {
        Self {
            status: HealthStatus::Passing,
            last_heartbeat: Instant::now(),
            ttl,
        }
    }

    /// อัปเดต heartbeat — reset นาฬิกา TTL กลับมาที่ Passing
    pub fn beat(&mut self) {
        self.last_heartbeat = Instant::now();
        self.status = HealthStatus::Passing;
    }

    /// ตรวจสอบว่า TTL หมดอายุหรือยัง
    pub fn is_expired(&self) -> bool {
        self.last_heartbeat.elapsed() > self.ttl
    }

    /// คำนวณสถานะปัจจุบัน — ถ้า TTL หมด → Critical, ไม่งั้นคืนสถานะที่เก็บไว้
    pub fn computed_status(&self) -> HealthStatus {
        if self.is_expired() {
            HealthStatus::Critical
        } else {
            self.status.clone()
        }
    }
}
```

**แนวคิดสำคัญ:** `computed_status()` แยกออกจาก `status` field เพราะ:
- `status` เป็น authoritative state ที่ถูก set โดย heartbeat หรือ active health check
- `computed_status()` เพิ่ม TTL check อีกชั้น — ถ้า TTL expired แม้ status = Passing ก็ยัง return Critical
- pattern นี้คล้ายกับ Consul TTL check ที่ "service จะตายถ้าไม่ heartbeat ภายใน TTL"

### ขั้นที่ 2: ServiceInstance และ Registry

**`src/registry.rs`** — สร้าง data model และ registry หลัก

```rust
use crate::health::{HealthCheck, HealthStatus};
use crate::watcher::{ServiceEvent, WatcherTx};
use dashmap::DashMap;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use std::time::Duration;

/// ข้อผิดพลาดของ Registry
#[derive(Debug)]
pub enum RegistryError {
    NotFound(String),
    AlreadyRegistered(String),
}

impl std::fmt::Display for RegistryError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            RegistryError::NotFound(id) =>
                write!(f, "service instance not found: {}", id),
            RegistryError::AlreadyRegistered(id) =>
                write!(f, "service instance already registered: {}", id),
        }
    }
}

impl std::error::Error for RegistryError {}

/// ข้อมูล service instance หนึ่งตัว
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ServiceInstance {
    pub id: String,         // UUID เฉพาะตัว
    pub name: String,       // ชื่อ service เช่น "user-service"
    pub address: String,    // IP หรือ hostname
    pub port: u16,          // port ที่รับฟัง
    pub tags: Vec<String>,  // ["v2", "dc-th", "canary"]
    pub metadata: HashMap<String, String>, // {"version": "2.1.0"}
    pub weight: u32,        // น้ำหนักสำหรับ weighted LB (0-100)
}

impl ServiceInstance {
    pub fn new(
        id: impl Into<String>,
        name: impl Into<String>,
        address: impl Into<String>,
        port: u16,
    ) -> Self {
        Self {
            id: id.into(),
            name: name.into(),
            address: address.into(),
            port,
            tags: Vec::new(),
            metadata: HashMap::new(),
            weight: 10,
        }
    }

    pub fn with_tags(mut self, tags: Vec<String>) -> Self {
        self.tags = tags;
        self
    }

    pub fn with_weight(mut self, weight: u32) -> Self {
        self.weight = weight;
        self
    }

    pub fn with_metadata(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.metadata.insert(key.into(), value.into());
        self
    }

    pub fn address_port(&self) -> String {
        format!("{}:{}", self.address, self.port)
    }
}

/// Entry ใน registry รวม instance + health state
struct RegistryEntry {
    instance: ServiceInstance,
    health: HealthCheck,
}

/// Service Registry หลัก — thread-safe ด้วย DashMap
pub struct Registry {
    entries: Arc<DashMap<String, RegistryEntry>>,
    default_ttl: Duration,
    watcher_tx: Option<WatcherTx>,
}

impl Registry {
    pub fn new(default_ttl: Duration) -> Self {
        Self {
            entries: Arc::new(DashMap::new()),
            default_ttl,
            watcher_tx: None,
        }
    }

    pub fn with_watcher(mut self, tx: WatcherTx) -> Self {
        self.watcher_tx = Some(tx);
        self
    }

    fn notify(&self, event: ServiceEvent) {
        if let Some(tx) = &self.watcher_tx {
            let _ = tx.send(Some(event));
        }
    }
```

**หมายเหตุ:** `Arc<DashMap<...>>` ใช้ `Arc` เพื่อให้ clone registry handle ไปให้ background task ได้ DashMap เองก็ thread-safe แล้ว แต่ต้องการ `Arc` เพื่อ shared ownership

```rust
    /// ลงทะเบียน service instance ใหม่
    pub fn register(&self, instance: ServiceInstance) -> Result<(), RegistryError> {
        if self.entries.contains_key(&instance.id) {
            return Err(RegistryError::AlreadyRegistered(instance.id.clone()));
        }
        let id = instance.id.clone();
        let entry = RegistryEntry {
            instance: instance.clone(),
            health: HealthCheck::new(self.default_ttl),
        };
        self.entries.insert(id, entry);
        self.notify(ServiceEvent::Registered(instance));
        Ok(())
    }

    /// ยกเลิกการลงทะเบียน service instance
    pub fn deregister(&self, instance_id: &str) -> Result<(), RegistryError> {
        match self.entries.remove(instance_id) {
            Some((_, entry)) => {
                self.notify(ServiceEvent::Deregistered(entry.instance));
                Ok(())
            }
            None => Err(RegistryError::NotFound(instance_id.to_string())),
        }
    }

    /// อัปเดต heartbeat — บอกว่า instance ยังมีชีวิตอยู่
    pub fn heartbeat(&self, instance_id: &str) -> Result<(), RegistryError> {
        match self.entries.get_mut(instance_id) {
            Some(mut entry) => {
                let old_status = entry.health.status.clone();
                entry.health.beat();
                let new_status = entry.health.status.clone();
                if old_status != new_status {
                    self.notify(ServiceEvent::HealthChanged {
                        instance: entry.instance.clone(),
                        old_status,
                        new_status,
                    });
                }
                Ok(())
            }
            None => Err(RegistryError::NotFound(instance_id.to_string())),
        }
    }

    /// ลบ instances ที่ TTL หมดอายุ (เรียกโดย background reaper)
    pub fn reap_expired(&self) -> Vec<String> {
        let expired: Vec<String> = self.entries
            .iter()
            .filter(|e| e.health.is_expired())
            .map(|e| e.instance.id.clone())
            .collect();

        for id in &expired {
            if let Some((_, entry)) = self.entries.remove(id) {
                self.notify(ServiceEvent::Deregistered(entry.instance));
            }
        }
        expired
    }

    pub fn get_instance(&self, instance_id: &str) -> Option<(ServiceInstance, HealthStatus)> {
        self.entries.get(instance_id)
            .map(|e| (e.instance.clone(), e.health.computed_status()))
    }

    pub fn get_by_name(&self, name: &str) -> Vec<(ServiceInstance, HealthStatus)> {
        self.entries.iter()
            .filter(|e| e.instance.name == name)
            .map(|e| (e.instance.clone(), e.health.computed_status()))
            .collect()
    }

    pub fn all_instances(&self) -> Vec<(ServiceInstance, HealthStatus)> {
        self.entries.iter()
            .map(|e| (e.instance.clone(), e.health.computed_status()))
            .collect()
    }

    pub fn set_health_status(
        &self,
        instance_id: &str,
        status: HealthStatus,
    ) -> Result<(), RegistryError> {
        match self.entries.get_mut(instance_id) {
            Some(mut entry) => {
                let old_status = entry.health.status.clone();
                entry.health.status = status.clone();
                if old_status != status {
                    self.notify(ServiceEvent::HealthChanged {
                        instance: entry.instance.clone(),
                        old_status,
                        new_status: status,
                    });
                }
                Ok(())
            }
            None => Err(RegistryError::NotFound(instance_id.to_string())),
        }
    }

    pub fn len(&self) -> usize { self.entries.len() }
    pub fn is_empty(&self) -> bool { self.entries.is_empty() }
}
```

### ขั้นที่ 3: Watcher และ Watch Channel

**`src/watcher.rs`** — ใช้ `tokio::sync::watch` สำหรับ last-value broadcast

```rust
use crate::registry::ServiceInstance;
use crate::health::HealthStatus;
use tokio::sync::watch;

/// เหตุการณ์ที่เกิดขึ้นใน registry
#[derive(Debug, Clone)]
pub enum ServiceEvent {
    Registered(ServiceInstance),
    Deregistered(ServiceInstance),
    HealthChanged {
        instance: ServiceInstance,
        old_status: HealthStatus,
        new_status: HealthStatus,
    },
}

pub type WatcherTx = watch::Sender<Option<ServiceEvent>>;
pub type WatcherRx = watch::Receiver<Option<ServiceEvent>>;

/// Watcher สำหรับติดตามการเปลี่ยนแปลงของ registry
pub struct Watcher {
    rx: WatcherRx,
}

impl Watcher {
    /// สร้าง (WatcherTx, Watcher) คู่กัน
    pub fn new() -> (WatcherTx, Self) {
        let (tx, rx) = watch::channel(None);
        (tx, Self { rx })
    }

    /// clone receiver เพื่อส่งให้ listener หลายตัว
    pub fn subscribe(&self) -> WatcherRx {
        self.rx.clone()
    }
}
```

**ทำไมถึงใช้ `Option<ServiceEvent>`:** `watch::channel` ต้องมี initial value, เราใช้ `None` เป็นค่าเริ่มต้น และส่ง `Some(event)` เมื่อมีเหตุการณ์จริง ทำให้ subscriber ใหม่ไม่เข้าใจผิดว่ามี event เกิดขึ้นแล้ว

**การใช้งาน Watcher ใน background task:**

```rust
use tokio::time::{interval, Duration};

/// background task ที่ reap expired instances ทุก interval
pub async fn run_reaper(registry: Arc<Registry>, period: Duration) {
    let mut ticker = interval(period);
    loop {
        ticker.tick().await;
        let reaped = registry.reap_expired();
        if !reaped.is_empty() {
            eprintln!("[reaper] reaped {} expired instances: {:?}", reaped.len(), reaped);
        }
    }
}

/// ตัวอย่างการ watch events ใน async context
pub async fn watch_events(mut rx: WatcherRx) {
    loop {
        if rx.changed().await.is_err() {
            break; // sender dropped
        }
        let event = rx.borrow_and_update().clone();
        match event {
            Some(ServiceEvent::Registered(inst)) =>
                println!("[watch] + {} ({}) registered", inst.id, inst.name),
            Some(ServiceEvent::Deregistered(inst)) =>
                println!("[watch] - {} ({}) deregistered", inst.id, inst.name),
            Some(ServiceEvent::HealthChanged { instance, old_status, new_status }) =>
                println!("[watch] ~ {} health: {} → {}", instance.id, old_status, new_status),
            None => {}
        }
    }
}
```

### ขั้นที่ 4: Service Discovery และ Query Engine

**`src/discovery.rs`** — Extension trait สำหรับ query

```rust
use crate::registry::{Registry, ServiceInstance};
use crate::health::HealthStatus;

/// Query parameters สำหรับค้นหา service
#[derive(Debug, Default, Clone)]
pub struct ServiceQuery {
    pub name: Option<String>,
    pub tag: Option<String>,
    pub health: Option<HealthStatus>,
}

impl ServiceQuery {
    pub fn by_name(name: impl Into<String>) -> Self {
        Self { name: Some(name.into()), ..Default::default() }
    }

    pub fn healthy(mut self) -> Self {
        self.health = Some(HealthStatus::Passing);
        self
    }

    pub fn with_tag(mut self, tag: impl Into<String>) -> Self {
        self.tag = Some(tag.into());
        self
    }
}

/// Extension trait สำหรับ Registry ที่เพิ่ม discovery methods
pub trait DiscoveryExt {
    fn query(&self, q: &ServiceQuery) -> Vec<ServiceInstance>;
    fn healthy_instances(&self, service_name: &str) -> Vec<ServiceInstance>;
}

impl DiscoveryExt for Registry {
    fn query(&self, q: &ServiceQuery) -> Vec<ServiceInstance> {
        self.all_instances()
            .into_iter()
            .filter(|(inst, health)| {
                if let Some(name) = &q.name {
                    if &inst.name != name { return false; }
                }
                if let Some(tag) = &q.tag {
                    if !inst.tags.iter().any(|t| t == tag) { return false; }
                }
                if let Some(wanted) = &q.health {
                    if *health != *wanted { return false; }
                }
                true
            })
            .map(|(inst, _)| inst)
            .collect()
    }

    fn healthy_instances(&self, service_name: &str) -> Vec<ServiceInstance> {
        self.query(&ServiceQuery::by_name(service_name).healthy())
    }
}
```

**Extension Trait Pattern:** แทนที่จะเพิ่ม `query()` เข้าไปใน `Registry` struct โดยตรง การใช้ extension trait (`DiscoveryExt`) ช่วย:
1. **Separation of concerns** — registry ดูแลแค่ CRUD, discovery logic อยู่ต่างหาก
2. **Testability** — สามารถ mock `DiscoveryExt` ได้แยกต่างหาก
3. **Extensibility** — ใครก็เพิ่ม query methods ใหม่ได้โดยไม่แก้ registry code

### ขั้นที่ 5: Load Balancers

**`src/balancer.rs`** — สามรูปแบบ

#### Round-Robin Balancer

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

pub struct RoundRobinBalancer {
    counter: Arc<AtomicUsize>,
}

impl RoundRobinBalancer {
    pub fn new() -> Self {
        Self { counter: Arc::new(AtomicUsize::new(0)) }
    }
}

impl Balancer for RoundRobinBalancer {
    fn select<'a>(&self, instances: &'a [ServiceInstance]) -> Option<&'a ServiceInstance> {
        if instances.is_empty() { return None; }
        // fetch_add เป็น atomic — ไม่ต้องใช้ Mutex
        let idx = self.counter.fetch_add(1, Ordering::Relaxed) % instances.len();
        Some(&instances[idx])
    }
}
```

**`Ordering::Relaxed` เพียงพอหรือไม่?** สำหรับ counter ที่ไม่มี synchronization dependency กับ memory อื่น ใช้ `Relaxed` ได้ เราต้องการแค่ที่ counter เพิ่มขึ้น ไม่ได้ต้องการ "happens-before" guarantee ที่แน่นอน

#### Weighted Random Balancer

```rust
use rand::Rng;

pub struct WeightedRandomBalancer;

impl Balancer for WeightedRandomBalancer {
    fn select<'a>(&self, instances: &'a [ServiceInstance]) -> Option<&'a ServiceInstance> {
        if instances.is_empty() { return None; }
        let total_weight: u32 = instances.iter().map(|i| i.weight.max(1)).sum();
        let mut rng = rand::thread_rng();
        let mut pick = rng.gen_range(0..total_weight);
        for inst in instances {
            let w = inst.weight.max(1);
            if pick < w { return Some(inst); }
            pick -= w;
        }
        instances.last()
    }
}
```

**Algorithm:** "weighted reservoir sampling" แบบง่าย — สุ่มตัวเลข 0..total_weight แล้ว scan ไปเรื่อย ๆ ลดค่าลงจนถึง instance ที่ถูกเลือก เช่น weights = [70, 20, 10]:
- pick = 55 → 55 >= 70? ไม่ → pick = 55 - 70 ไม่ได้ → เลือก instance[0]  
- pick = 75 → 75 >= 70? ใช่ → pick = 5 → 5 >= 20? ไม่ → pick = 5 - 20 ไม่ได้ → เลือก instance[1]

#### Consistent Hash Balancer (Ketama Ring)

```rust
use std::collections::BTreeMap;
use std::hash::{Hash, Hasher};
use std::collections::hash_map::DefaultHasher;

pub struct ConsistentHashBalancer {
    replicas: usize, // virtual nodes ต่อ instance (แนะนำ 150)
}

impl ConsistentHashBalancer {
    pub fn new(replicas: usize) -> Self {
        Self { replicas }
    }

    fn hash_key(key: &str) -> u64 {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish()
    }

    /// เลือก instance โดยใช้ routing_key เช่น user_id หรือ session_id
    pub fn select_by_key<'a>(
        &self,
        instances: &'a [ServiceInstance],
        routing_key: &str,
    ) -> Option<&'a ServiceInstance> {
        if instances.is_empty() { return None; }

        // สร้าง ring: hash(id#replica) → instance_index
        let mut ring: BTreeMap<u64, usize> = BTreeMap::new();
        for (idx, inst) in instances.iter().enumerate() {
            for r in 0..self.replicas {
                let vnode_key = format!("{}#{}", inst.id, r);
                ring.insert(Self::hash_key(&vnode_key), idx);
            }
        }

        // หาจุดแรกบน ring ที่ >= hash(routing_key)
        let h = Self::hash_key(routing_key);
        let idx = ring.range(h..)
            .next()
            .or_else(|| ring.iter().next()) // wrap-around
            .map(|(_, &idx)| idx)
            .unwrap_or(0);

        Some(&instances[idx])
    }
}
```

**เหตุผลที่เลือก `BTreeMap`:** ต้องการ operation "หาค่าที่ >= key" (`range(h..)`) ซึ่ง `BTreeMap` รองรับใน O(log n) `HashMap` ทำไม่ได้

### ขั้นที่ 6: DNS SRV Cache

**`src/dns.rs`** — DNS-like layer สำหรับ human-readable resolution

```rust
use crate::registry::ServiceInstance;
use std::collections::HashMap;
use std::time::{Duration, Instant};

/// SRV record คล้าย DNS SRV: _service._proto.domain
#[derive(Debug, Clone)]
pub struct SrvRecord {
    /// เช่น "_http._tcp.payment-api"
    pub name: String,
    pub priority: u16,  // ค่าน้อย = สำคัญกว่า (primary)
    pub weight: u16,    // น้ำหนักใน priority เดียวกัน
    pub port: u16,
    pub target: String, // hostname/IP ของ target
}

impl SrvRecord {
    pub fn from_instance(inst: &ServiceInstance) -> Self {
        Self {
            name: format!("_http._tcp.{}", inst.name),
            priority: 10,
            weight: inst.weight as u16,
            port: inst.port,
            target: inst.address.clone(),
        }
    }
}

pub struct DnsCache {
    cache: HashMap<String, CacheEntry>,
    default_ttl: Duration,
}

struct CacheEntry {
    records: Vec<SrvRecord>,
    created_at: Instant,
    ttl: Duration,
}

impl CacheEntry {
    fn is_expired(&self) -> bool {
        self.created_at.elapsed() > self.ttl
    }
}

impl DnsCache {
    pub fn new(default_ttl: Duration) -> Self {
        Self { cache: HashMap::new(), default_ttl }
    }

    pub fn put(&mut self, service_name: &str, instances: &[ServiceInstance]) {
        let records: Vec<SrvRecord> = instances.iter()
            .map(SrvRecord::from_instance)
            .collect();
        self.cache.insert(service_name.to_string(), CacheEntry {
            records,
            created_at: Instant::now(),
            ttl: self.default_ttl,
        });
    }

    pub fn lookup(&self, service_name: &str) -> Option<&[SrvRecord]> {
        self.cache.get(service_name).and_then(|entry| {
            if entry.is_expired() { None } else { Some(entry.records.as_slice()) }
        })
    }

    pub fn evict_expired(&mut self) -> usize {
        let before = self.cache.len();
        self.cache.retain(|_, v| !v.is_expired());
        before - self.cache.len()
    }

    /// resolve service เป็น address:port เรียงตาม priority → weight
    pub fn resolve(&self, service_name: &str) -> Vec<String> {
        match self.lookup(service_name) {
            None => Vec::new(),
            Some(records) => {
                let mut sorted = records.to_vec();
                // priority น้อยก่อน, weight มากก่อนในกลุ่มเดียวกัน
                sorted.sort_by(|a, b| a.priority.cmp(&b.priority)
                    .then(b.weight.cmp(&a.weight)));
                sorted.iter()
                    .map(|r| format!("{}:{}", r.target, r.port))
                    .collect()
            }
        }
    }
}
```

**ทำไมถึงมี DNS layer:** การ resolve ชื่อ service เป็น endpoint เป็น pattern ที่ client-side library ทำอยู่แล้ว (เช่น gRPC name resolver) DnsCache ทำหน้าที่เป็น L1 cache ก่อนที่จะ query registry จริง ช่วยลด load บน registry

### ขั้นที่ 7: Main Demo และ Background Reaper

**`src/main.rs`** — รวมทุกอย่างเข้าด้วยกัน

```rust
use service_disc::*;
use std::time::Duration;

#[tokio::main]
async fn main() {
    println!("=== Service Discovery Demo ===\n");

    // 1. สร้าง Registry พร้อม Watcher
    let (tx, _watcher) = Watcher::new();
    let reg = Registry::new(Duration::from_secs(30)).with_watcher(tx);

    // 2. ลงทะเบียน services
    let instances = vec![
        ServiceInstance::new("user-1", "user-service", "10.0.0.1", 8081)
            .with_tags(vec!["v2".into(), "dc-th".into()])
            .with_weight(20),
        ServiceInstance::new("user-2", "user-service", "10.0.0.2", 8081)
            .with_tags(vec!["v2".into(), "dc-sg".into()])
            .with_weight(10),
        ServiceInstance::new("pay-1", "payment-api", "10.0.0.3", 8082)
            .with_tags(vec!["v1".into()])
            .with_weight(15),
    ];

    for inst in instances {
        reg.register(inst).unwrap();
    }
    println!("Registered {} instances", reg.len());

    // 3. Discovery — หา healthy user-service instances
    let discovered = reg.healthy_instances("user-service");
    println!("Healthy user-service instances: {}", discovered.len());
    for inst in &discovered {
        println!("  - {} @ {}", inst.id, inst.address_port());
    }

    // 4. Load Balancing — Round-Robin
    println!("\n--- Round-Robin ---");
    let rr = RoundRobinBalancer::new();
    for _ in 0..4 {
        if let Some(inst) = rr.select(&discovered) {
            println!("  → {}", inst.address_port());
        }
    }

    // 5. Consistent Hash
    println!("\n--- Consistent Hash ---");
    let ch = ConsistentHashBalancer::new(150);
    for key in &["user-42", "user-99", "user-42"] {
        if let Some(inst) = ch.select_by_key(&discovered, key) {
            println!("  key={} → {}", key, inst.address_port());
        }
    }

    // 6. DNS SRV Resolution
    println!("\n--- DNS SRV Resolution ---");
    let mut dns = DnsCache::new(Duration::from_secs(60));
    dns.put("user-service", &discovered);
    let resolved = dns.resolve("user-service");
    for addr in &resolved {
        println!("  SRV → {}", addr);
    }

    println!("\nDone.");
}
```

**Output จากการรันจริง:**

```
=== Service Discovery Demo ===

Registered 3 instances
Healthy user-service instances: 2
  - user-1 @ 10.0.0.1:8081
  - user-2 @ 10.0.0.2:8081

--- Round-Robin ---
  → 10.0.0.1:8081
  → 10.0.0.2:8081
  → 10.0.0.1:8081
  → 10.0.0.2:8081

--- Consistent Hash ---
  key=user-42 → 10.0.0.2:8081
  key=user-99 → 10.0.0.2:8081
  key=user-42 → 10.0.0.2:8081

--- DNS SRV Resolution ---
  SRV → 10.0.0.1:8081
  SRV → 10.0.0.2:8081

Done.
```

**ข้อสังเกต:**
- `user-42` และ `user-42` ได้ instance เดิม (consistent hash ทำงานถูกต้อง)
- Round-robin วนเวียน 10.0.0.1 → 10.0.0.2 → 10.0.0.1 สม่ำเสมอ
- DNS resolve เรียงตาม weight (20 มาก่อน 10)

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ทั้งหมด **25 tests** ครอบคลุมทุก module

### รัน tests ด้วย

```bash
cargo test
```

### Real `cargo test` Output

```
running 25 tests
test balancer::tests::test_round_robin_empty ... ok
test balancer::tests::test_round_robin_distribution ... ok
test balancer::tests::test_weighted_random_respects_weight ... ok
test discovery::tests::test_healthy_instances_filters_critical ... ok
test discovery::tests::test_query_by_name_and_tag ... ok
test discovery::tests::test_query_by_tag ... ok
test discovery::tests::test_query_by_name ... ok
test dns::tests::test_dns_cache_lookup ... ok
test dns::tests::test_dns_cache_miss ... ok
test dns::tests::test_dns_resolve_order ... ok
test balancer::tests::test_consistent_hash_same_key_same_instance ... ok
test health::tests::test_health_check_initial_passing ... ok
test balancer::tests::test_consistent_hash_different_keys_distribute ... ok
test registry::tests::test_deregister ... ok
test registry::tests::test_deregister_nonexistent_fails ... ok
test registry::tests::test_get_by_name ... ok
test dns::tests::test_dns_cache_expiry ... ok
test registry::tests::test_register_and_get ... ok
test registry::tests::test_register_duplicate_fails ... ok
test health::tests::test_health_check_ttl_expiry ... ok
test watcher::tests::test_watcher_receives_deregister_event ... ok
test watcher::tests::test_watcher_receives_register_event ... ok
test health::tests::test_health_check_beat_resets_timer ... ok
test registry::tests::test_ttl_expiry_reaper ... ok
test registry::tests::test_heartbeat_prevents_expiry ... ok

test result: ok. 25 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.21s

     Running unittests src/main.rs (target/debug/deps/service_disc-3acd82972a3ab35b)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests service_disc

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### รายละเอียด Tests แต่ละ Module

#### Health Tests (3 tests)

```rust
#[test]
fn test_health_check_initial_passing() {
    let hc = HealthCheck::new(Duration::from_secs(10));
    assert_eq!(hc.computed_status(), HealthStatus::Passing);
}

#[test]
fn test_health_check_beat_resets_timer() {
    let mut hc = HealthCheck::new(Duration::from_millis(200));
    thread::sleep(Duration::from_millis(210));
    // ก่อน beat ควรเป็น Critical
    assert_eq!(hc.computed_status(), HealthStatus::Critical);
    hc.beat();
    // หลัง beat ควรกลับมาเป็น Passing
    assert_eq!(hc.computed_status(), HealthStatus::Passing);
}

#[test]
fn test_health_check_ttl_expiry() {
    let hc = HealthCheck::new(Duration::from_millis(50));
    thread::sleep(Duration::from_millis(80));
    assert!(hc.is_expired());
    assert_eq!(hc.computed_status(), HealthStatus::Critical);
}
```

#### Registry Tests (6 tests)

```rust
#[test]
fn test_ttl_expiry_reaper() {
    let reg = Registry::new(Duration::from_millis(50));
    reg.register(make_instance("short-1", "svc")).unwrap();
    reg.register(make_instance("short-2", "svc")).unwrap();
    assert_eq!(reg.len(), 2);
    thread::sleep(Duration::from_millis(80));
    let reaped = reg.reap_expired();
    assert_eq!(reaped.len(), 2);
    assert_eq!(reg.len(), 0);
}

#[test]
fn test_heartbeat_prevents_expiry() {
    let reg = Registry::new(Duration::from_millis(100));
    reg.register(make_instance("keep-alive", "svc")).unwrap();
    thread::sleep(Duration::from_millis(60));
    reg.heartbeat("keep-alive").unwrap();
    thread::sleep(Duration::from_millis(60));
    // ไม่ expire เพราะ heartbeat ที่ t=60ms รีเซ็ต TTL
    // ณ จุดนี้ elapsed หลัง heartbeat = 60ms < 100ms TTL
    let reaped = reg.reap_expired();
    assert_eq!(reaped.len(), 0);
}
```

#### Discovery Tests (4 tests)

```rust
#[test]
fn test_healthy_instances_filters_critical() {
    let reg = Registry::new(Duration::from_secs(30));
    reg.register(ServiceInstance::new("h1", "db", "10.0.0.1", 5432)).unwrap();
    reg.register(ServiceInstance::new("h2", "db", "10.0.0.2", 5432)).unwrap();
    // ทำให้ h2 เป็น Critical (เช่น active health check พบ error)
    reg.set_health_status("h2", HealthStatus::Critical).unwrap();
    let healthy = reg.healthy_instances("db");
    assert_eq!(healthy.len(), 1);  // มีแค่ h1
    assert_eq!(healthy[0].id, "h1");
}
```

#### Balancer Tests (5 tests)

```rust
#[test]
fn test_round_robin_distribution() {
    let lb = RoundRobinBalancer::new();
    let instances = make_instances(3);
    let selected: Vec<String> = (0..6)
        .map(|_| lb.select(&instances).unwrap().id.clone())
        .collect();
    // 6 requests บน 3 instances → หมุนครบ 2 รอบสม่ำเสมอ
    assert_eq!(selected[0], "inst-0");
    assert_eq!(selected[1], "inst-1");
    assert_eq!(selected[2], "inst-2");
    assert_eq!(selected[3], "inst-0");
    assert_eq!(selected[4], "inst-1");
    assert_eq!(selected[5], "inst-2");
}

#[test]
fn test_weighted_random_respects_weight() {
    let lb = WeightedRandomBalancer::new();
    let instances = vec![
        ServiceInstance::new("heavy", "svc", "10.0.0.1", 8080).with_weight(90),
        ServiceInstance::new("light", "svc", "10.0.0.2", 8080).with_weight(10),
    ];
    let mut heavy_count = 0usize;
    for _ in 0..1000 {
        if lb.select(&instances).unwrap().id == "heavy" {
            heavy_count += 1;
        }
    }
    // คาดว่า heavy จะถูกเลือก ~90% ± ให้เผื่อ 10%
    assert!(heavy_count > 800, "heavy_count={}", heavy_count);
    assert!(heavy_count < 980, "heavy_count={}", heavy_count);
}

#[test]
fn test_consistent_hash_same_key_same_instance() {
    let lb = ConsistentHashBalancer::new(150);
    let instances = make_instances(5);
    let key = "user-42";
    let first = lb.select_by_key(&instances, key).unwrap().id.clone();
    // เรียกซ้ำ 10 ครั้งต้องได้ instance เดิมเสมอ
    for _ in 0..10 {
        let result = lb.select_by_key(&instances, key).unwrap().id.clone();
        assert_eq!(result, first);
    }
}

#[test]
fn test_consistent_hash_different_keys_distribute() {
    let lb = ConsistentHashBalancer::new(150);
    let instances = make_instances(5);
    let keys = ["k1", "k2", "k3", ..., "k20"]; // 20 keys
    let mut counts = HashMap::new();
    for key in &keys {
        let id = lb.select_by_key(&instances, key).unwrap().id.clone();
        *counts.entry(id).or_insert(0usize) += 1;
    }
    // ต้องมี distribution ไปอย่างน้อย 2 instances
    assert!(counts.len() >= 2);
}
```

#### DNS Tests (4 tests)

```rust
#[test]
fn test_dns_cache_expiry() {
    let mut cache = DnsCache::new(Duration::from_millis(50));
    cache.put("svc", &make_instances());
    thread::sleep(Duration::from_millis(80));
    // หลัง TTL หมด → cache miss
    assert!(cache.lookup("svc").is_none());
}

#[test]
fn test_dns_resolve_order() {
    let mut cache = DnsCache::new(Duration::from_secs(60));
    let instances = make_instances(); // weight 10, weight 5
    cache.put("postgres", &instances);
    let resolved = cache.resolve("postgres");
    // weight 10 ต้องมาก่อน weight 5
    assert!(resolved[0].contains("10.0.1.1"));
}
```

#### Watcher Tests (2 tests)

```rust
#[tokio::test]
async fn test_watcher_receives_register_event() {
    let (tx, _watcher) = Watcher::new();
    let reg = Registry::new(Duration::from_secs(30)).with_watcher(tx.clone());

    let mut rx = tx.subscribe();
    let inst = ServiceInstance::new("w-1", "svc", "127.0.0.1", 9000);
    reg.register(inst).unwrap();

    assert!(rx.has_changed().unwrap_or(false));
    let event = rx.borrow_and_update().clone();
    assert!(matches!(event, Some(ServiceEvent::Registered(_))));
}
```

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/service-disc
```

### Production Deployment Pattern

ในระบบจริง service registry มักถูก deploy เป็น highly-available cluster:

```
┌─────────────────────────────────────────────────────┐
│                   Load Balancer                      │
│                (health-check aware)                  │
└───────────┬─────────────────┬───────────┬───────────┘
            │                 │           │
     ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
     │  Registry   │   │  Registry   │   │  Registry   │
     │  Node 1     │◄──►  Node 2     │◄──►  Node 3     │
     │  (primary)  │   │  (replica)  │   │  (replica)  │
     └─────────────┘   └─────────────┘   └─────────────┘
           ▲
           │ register / heartbeat / deregister
     ┌─────┴────┐
     │ Services │
     │ (clients)│
     └──────────┘
```

สำหรับ in-memory registry นี้ ถ้าต้องการ deploy จริงให้ combine กับ Raft (Project F01) เพื่อ replicated state machine

### Docker Deployment

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/service-disc /usr/local/bin/
EXPOSE 8500
CMD ["service-disc"]
```

### Health Check Integration

```yaml
# docker-compose.yml (ถ้าเพิ่ม HTTP layer)
services:
  registry:
    image: service-disc:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8500/health"]
      interval: 10s
      timeout: 5s
      retries: 3
```

## Pitfalls และข้อผิดพลาดที่พบบ่อย

### Pitfall 1: DashMap Reference ค้างใน Long-lived Scope

```rust
// ❌ WRONG: ref จาก DashMap.get() ค้างอยู่ขณะ call method อื่น
// → deadlock เพราะ DashMap lock shard ตลอดเวลาที่ hold ref
let entry = registry.entries.get("svc-1").unwrap();
println!("{}", entry.instance.name);
registry.entries.insert("svc-2", new_entry); // ← อาจ deadlock ถ้า svc-1 และ svc-2 อยู่ shard เดียวกัน

// ✅ CORRECT: clone ออกมาก่อนแล้ว drop ref
let instance = {
    let entry = registry.entries.get("svc-1").unwrap();
    entry.instance.clone() // clone ออกก่อน drop ref
};
// ref ถูก drop แล้ว
println!("{}", instance.name);
registry.entries.insert("svc-2", new_entry); // ปลอดภัย
```

**สาเหตุ:** `DashMap::get()` คืน `Ref<K, V>` ที่ hold shard lock ตราบเท่าที่ `Ref` ยังอยู่ใน scope ถ้า code ถัดไปพยายาม write ลง shard เดียวกัน → deadlock

### Pitfall 2: watch Channel และ has_changed() Race

```rust
// ❌ WRONG: check has_changed() แต่ไม่ mark_changed() → infinite wait
loop {
    if rx.has_changed().unwrap() {
        let event = rx.borrow().clone(); // ← borrow() ไม่ mark เป็น "seen"
        handle(event);
        // has_changed() ยังเป็น true → loop forever กับ event เดิม
    }
    tokio::time::sleep(Duration::from_millis(10)).await;
}

// ✅ CORRECT: ใช้ borrow_and_update() เพื่อ mark seen
loop {
    if rx.has_changed().unwrap() {
        let event = rx.borrow_and_update().clone(); // mark seen atomically
        handle(event);
    }
    tokio::time::sleep(Duration::from_millis(10)).await;
}

// ✅ หรือใช้ changed().await แบบ idiomatic tokio
loop {
    rx.changed().await.unwrap(); // block จนกว่าจะมีค่าใหม่
    let event = rx.borrow_and_update().clone();
    handle(event);
}
```

### Pitfall 3: Consistent Hash Ring Rebuild ทุก Request

```rust
// ❌ WRONG: สร้าง ring ใหม่ทุกครั้งที่ select → O(n * replicas) ต่อ request
impl Balancer for ConsistentHashBalancer {
    fn select<'a>(&self, instances: &'a [ServiceInstance]) -> Option<&'a ServiceInstance> {
        let mut ring: BTreeMap<u64, usize> = BTreeMap::new(); // ← rebuild ทุกครั้ง
        for (idx, inst) in instances.iter().enumerate() { ... }
        ...
    }
}

// ✅ CORRECT production version: cache ring และ rebuild เมื่อ instances เปลี่ยน
pub struct ConsistentHashBalancer {
    replicas: usize,
    ring: RwLock<Option<(Vec<String>, BTreeMap<u64, usize>)>>,
    // (snapshot ของ instance ids, ring)
}

impl ConsistentHashBalancer {
    fn get_or_rebuild<'a>(&self, instances: &'a [ServiceInstance]) -> RwLockReadGuard<...> {
        // ตรวจสอบ instance ids ว่าเปลี่ยนหรือยัง ถ้าไม่ → reuse ring
        let ids: Vec<String> = instances.iter().map(|i| i.id.clone()).collect();
        {
            let guard = self.ring.read().unwrap();
            if let Some((cached_ids, _)) = guard.as_ref() {
                if cached_ids == &ids { return guard; } // reuse
            }
        }
        // rebuild
        let mut write_guard = self.ring.write().unwrap();
        *write_guard = Some((ids, build_ring(instances, self.replicas)));
        drop(write_guard);
        self.ring.read().unwrap()
    }
}
```

### Pitfall 4: TTL Race Condition กับ Reaper

```rust
// สถานการณ์: service ส่ง heartbeat พอดีขณะที่ reaper กำลัง check expiry

// Thread A (reaper):               Thread B (heartbeat):
// check: is_expired("svc-1")       <-- svc-1 just expired by 1ms
//   → true
//                                  heartbeat("svc-1") → beat() called
//                                    last_heartbeat = Instant::now()
// entries.remove("svc-1") ← ลบทิ้ง!
// → svc-1 ถูก reap ทั้งที่เพิ่ง heartbeat

// ✅ ป้องกันด้วย grace period: TTL + grace ก่อนจะ reap
pub fn reap_expired(&self) -> Vec<String> {
    let grace = Duration::from_secs(1); // เผื่อ clock skew
    let expired: Vec<String> = self.entries
        .iter()
        .filter(|e| e.health.last_heartbeat.elapsed() > e.health.ttl + grace)
        .map(|e| e.instance.id.clone())
        .collect();
    ...
}

// ✅ หรือใช้ generation counter: heartbeat เพิ่ม gen, reaper ตรวจ gen ก่อน remove
struct HealthCheck {
    ...
    generation: u64,
}

fn reap_expired(&self) -> Vec<String> {
    let candidates: Vec<(String, u64)> = self.entries
        .iter()
        .filter(|e| e.health.is_expired())
        .map(|e| (e.instance.id.clone(), e.health.generation))
        .collect();

    for (id, gen) in candidates {
        // ตรวจสอบ generation ก่อน remove — ถ้า heartbeat มา gen จะเปลี่ยน
        if let Some(entry) = self.entries.get(&id) {
            if entry.health.generation == gen { // gen ยังเหมือนเดิม → safe to remove
                drop(entry);
                self.entries.remove(&id);
            }
        }
    }
    ...
}
```

### Pitfall 5: Weighted Balancer กับ Weight = 0

```rust
// ❌ WRONG: weight=0 ทำให้ total_weight=0 → gen_range(0..0) = panic!
let total_weight: u32 = instances.iter().map(|i| i.weight).sum();
let pick = rng.gen_range(0..total_weight); // panic ถ้า total=0

// ✅ CORRECT: ใช้ .max(1) เสมอเพื่อกัน weight=0
let total_weight: u32 = instances.iter().map(|i| i.weight.max(1)).sum();
// weight=0 ถูกนับเป็น 1 แทน → ทุก instance มีโอกาสถูกเลือก

// ✅ หรือ filter instances ที่ weight=0 ออกก่อน
let active: Vec<_> = instances.iter().filter(|i| i.weight > 0).collect();
if active.is_empty() { return instances.first(); } // fallback
```

## การต่อยอด (Extensions & Exercises)

### Exercise 1: HTTP API Layer (⭐⭐)

เพิ่ม `axum` web server เพื่อทำให้ registry ใช้ผ่าน HTTP API ได้จริง คล้าย Consul API:

```
POST   /v1/agent/service/register    { id, name, address, port, tags }
DELETE /v1/agent/service/deregister/{id}
PUT    /v1/agent/check/pass/{id}     (heartbeat)
GET    /v1/health/service/{name}?passing=true
GET    /v1/catalog/services          (list all service names)
```

**Hint:** ใช้ `Arc<Registry>` เป็น State ใน axum, ใช้ `tokio::spawn` รัน reaper background task

### Exercise 2: Persistent Storage (⭐⭐⭐)

ปัจจุบัน registry อยู่ใน memory เท่านั้น — ถ้า process restart ข้อมูลหาย เพิ่ม persistence layer:
- เขียน registry snapshot ลง `sled` (embedded key-value store) ทุก N วินาที
- ตอน startup load snapshot กลับมา
- ใช้ `serde_json` serialize `ServiceInstance` เก็บใน sled

```toml
[dependencies]
sled = "0.34"
```

```rust
pub struct PersistentRegistry {
    inner: Registry,
    db: sled::Db,
}

impl PersistentRegistry {
    pub fn load_from_disk(path: &str, ttl: Duration) -> Self {
        let db = sled::open(path).unwrap();
        let reg = Registry::new(ttl);
        for item in db.iter() {
            let (_, v) = item.unwrap();
            if let Ok(inst) = serde_json::from_slice::<ServiceInstance>(&v) {
                let _ = reg.register(inst);
            }
        }
        Self { inner: reg, db }
    }
}
```

### Exercise 3: Active Health Checks (⭐⭐⭐)

TTL-based health check เป็น passive — รอให้ service บอกเอง เพิ่ม active HTTP health check ที่ registry probe สอบถาม service เอง:

```rust
#[derive(Clone)]
pub struct HttpHealthChecker {
    client: reqwest::Client,
    interval: Duration,
}

impl HttpHealthChecker {
    pub async fn run_checks(&self, registry: Arc<Registry>) {
        let mut ticker = interval(self.interval);
        loop {
            ticker.tick().await;
            let instances = registry.all_instances();
            for (inst, _) in instances {
                let url = format!("http://{}/health", inst.address_port());
                let status = match self.client.get(&url).timeout(Duration::from_secs(2)).send().await {
                    Ok(r) if r.status().is_success() => HealthStatus::Passing,
                    Ok(_) => HealthStatus::Warning,
                    Err(_) => HealthStatus::Critical,
                };
                let _ = registry.set_health_status(&inst.id, status);
            }
        }
    }
}
```

**Hint:** ใช้ `reqwest` crate, เพิ่ม `health_check_url` field ใน `ServiceInstance`

### Exercise 4: Service Mesh Topology (⭐⭐⭐⭐)

สร้าง topology graph ว่า service ไหน depends on service ไหน:

```rust
pub struct TopologyRegistry {
    registry: Registry,
    /// service A → Vec<service B> (A calls B)
    dependencies: DashMap<String, Vec<String>>,
}

impl TopologyRegistry {
    pub fn declare_dependency(&self, from: &str, to: &str) { ... }

    /// ค้นหา downstream services ที่ได้รับผลกระทบ ถ้า `service` ล้ม
    pub fn impact_analysis(&self, service: &str) -> Vec<String> { ... }

    /// ตรวจหา circular dependencies
    pub fn detect_cycles(&self) -> Vec<Vec<String>> { ... }
}
```

**Hint:** ใช้ DFS/BFS บน adjacency list, สำหรับ cycle detection ใช้ Tarjan's algorithm

### Exercise 5: Namespace และ Multi-tenancy (⭐⭐⭐)

ใน production มักต้องการแยก services ของแต่ละ team ออกจากกัน:

```rust
pub struct NamespacedRegistry {
    // namespace → Registry
    namespaces: DashMap<String, Registry>,
}

impl NamespacedRegistry {
    /// register ภายใต้ namespace เช่น "payments", "auth", "infra"
    pub fn register(&self, ns: &str, instance: ServiceInstance) -> Result<(), RegistryError> {
        self.namespaces.entry(ns.to_string())
            .or_insert_with(|| Registry::new(Duration::from_secs(30)))
            .register(instance)
    }

    /// query ข้าม namespace (เช่น infra services ที่ทุก namespace ใช้ได้)
    pub fn query_global(&self, q: &ServiceQuery) -> Vec<ServiceInstance> { ... }
}
```

### Exercise 6: Metrics และ Observability (⭐⭐)

เพิ่ม Prometheus metrics สำหรับ production monitoring:

```rust
pub struct RegistryMetrics {
    /// จำนวน registered instances แยกตาม service name
    registered_total: HashMap<String, u64>,
    /// จำนวน heartbeats ที่ได้รับต่อวินาที
    heartbeat_rate: f64,
    /// จำนวน instances ที่ถูก reap ทั้งหมด
    reaped_total: u64,
    /// distribution ของ health status
    health_distribution: HashMap<String, u64>, // "passing"|"warning"|"critical" → count
}

/// format เป็น Prometheus text exposition format
fn render_metrics(metrics: &RegistryMetrics) -> String {
    format!(
        "# HELP registry_registered_total Total registered instances\n\
         # TYPE registry_registered_total gauge\n\
         registry_registered_total {}\n",
        metrics.registered_total.values().sum::<u64>()
    )
}
```

**Hint:** ใช้ `Arc<Mutex<RegistryMetrics>>` เก็บ metrics ใน registry, expose ผ่าน `/metrics` endpoint

## สรุป

โปรเจคนี้สร้าง **Service Discovery System** ที่ครอบคลุมทุก concern ของระบบ distributed service registry:

| Component | เทคนิคที่ใช้ | Rust Concept ที่ได้ |
|-----------|------------|-------------------|
| Registry | `Arc<DashMap<K, V>>` | Concurrent shared state, lock-free reads |
| Health Check | `Instant` + `Duration` | Time-based expiry, TTL pattern |
| Watch/Events | `tokio::sync::watch` | Last-value semantics, async notification |
| Discovery | Extension trait | Trait-based API extension |
| Round-Robin | `AtomicUsize` | Lock-free atomic counter |
| Weighted Random | Probability sampling | Statistical distribution |
| Consistent Hash | `BTreeMap` range query | Ketama ring, virtual nodes |
| DNS Cache | `HashMap` + TTL | Cache pattern, TTL eviction |

**Pattern สำคัญ 4 ข้อที่ได้จากโปรเจคนี้:**

1. **TTL Heartbeat Pattern** — service ต้องพิสูจน์ว่า "ยังมีชีวิต" ด้วยการส่ง heartbeat ภายใน TTL แทนที่จะรอให้ TCP connection drop เป็น signal ทำให้ detect "zombie service" (process ยังรันแต่ไม่ respond) ได้

2. **Sharded Concurrent Map** — `DashMap` เป็นตัวอย่างของ sharding pattern ที่ scale read throughput โดยไม่ lock ทั้ง map เทคนิคเดียวกันใช้ใน Redis Cluster, Cassandra partition, และ database connection pooling

3. **Consistent Hashing with Virtual Nodes** — แก้ปัญหา uneven distribution ของ simple modulo hashing (เช่น `hash(key) % n`) และ massive remapping เมื่อ n เปลี่ยน virtual nodes เพิ่มจุดบน ring ให้ distribution สม่ำเสมอ

4. **Extension Trait Pattern** — แยก concerns ออกจากกัน registry รู้จัก CRUD, discovery รู้จัก query, balancer รู้จัก selection — ทำให้แต่ละ module ทดสอบและ extend ได้แยกอิสระ

**เชื่อมโยงไปโปรเจคถัดไป:** Project F03 — Distributed Lock จะนำ `tokio::sync` primitives ที่เรียนในโปรเจคนี้ไปต่อยอด แต่แทนที่จะเป็น single-process lock เราจะสร้าง distributed lock ที่ต้องทำงานข้ามหลาย processes ด้วย lease-based locking และ fencing token

---

**โปรเจคก่อนหน้า:** [Project F01: Raft KV Store](project-f01-raft-kv.md) | **โปรเจคถัดไป:** [Project F03: Distributed Lock](project-f03-dist-lock.md)
