# สารบัญหลักสูตรฉบับเต็ม (Full Curriculum)

สถานะ: ✅ เขียนแล้ว · 🚧 กำลังเขียน · ⏳ รอดำเนินการ

หมายเหตุ: จำนวน Part อาจขยายเกิน 110 หากมีหัวข้อสำคัญที่ควรเพิ่มระหว่างทาง

---

## โมดูล 0: เริ่มต้นใช้งาน Rust (Getting Started) — Part 1–5

| # | ชื่อบท | สถานะ |
|---|---|---|
| 1 | แนะนำ Rust และการติดตั้งเครื่องมือ (rustup, cargo, VS Code/rust-analyzer) | 🚧 |
| 2 | Cargo และโครงสร้างโปรเจกต์ (cargo new/build/run/check) | ✅ |
| 3 | ตัวแปร ชนิดข้อมูลพื้นฐาน และ Mutability (let, mut, shadowing, scalar/compound types) | ✅ |
| 4 | ฟังก์ชันและ Control Flow (if/else, loop, while, for, match เบื้องต้น) | ✅ |
| 5 | Comments, การจัดรูปแบบโค้ด (rustfmt) และ Clippy | ✅ |

## โมดูล 1: พื้นฐานภาษา Rust (Core Language Fundamentals) — Part 6–20

| # | ชื่อบท | สถานะ |
|---|---|---|
| 6 | Ownership เบื้องต้น (move, copy, drop) | ✅ |
| 7 | Borrowing และ References (&, &mut, borrow checker) | ✅ |
| 8 | Slices (&str, &[T]) | ✅ |
| 9 | Structs (tuple struct, unit struct, methods, impl) | ✅ |
| 10 | Enums และ Pattern Matching (match, if let, while let) | ✅ |
| 11 | Option<T> และ Null Safety | ✅ |
| 12 | Result<T,E> และ Error Handling เบื้องต้น (? operator) | ✅ |
| 13 | Collections: Vec<T> | ✅ |
| 14 | Collections: String และการจัดการข้อความ (UTF-8) | ✅ |
| 15 | Collections: HashMap, HashSet, BTreeMap/BTreeSet | ✅ |
| 16 | Modules และการจัดระเบียบโค้ด (mod, pub, use, super, crate) | ✅ |
| 17 | Packages, Crates, Workspaces | ✅ |
| 18 | Generics เบื้องต้น | ✅ |
| 19 | Traits เบื้องต้น | ✅ |
| 20 | Lifetimes เบื้องต้น | ✅ |

## โมดูล 2: ระดับกลาง (Intermediate) — Part 21–40

| # | ชื่อบท | สถานะ |
|---|---|---|
| 21 | Traits ขั้นสูง (default methods, trait objects, dyn Trait) | ✅ |
| 22 | Generics ขั้นสูง (trait bounds, where clauses, PhantomData) | ✅ |
| 23 | Lifetimes ขั้นสูง (lifetime elision, higher-ranked, structs with lifetimes) | ✅ |
| 24 | Closures (Fn, FnMut, FnOnce, capturing) | ✅ |
| 25 | Iterators เบื้องต้น (Iterator trait, for loop desugaring) | ✅ |
| 26 | Iterators ขั้นสูง (adapters, custom iterators, performance) | ✅ |
| 27 | Smart Pointers: Box<T> | ✅ |
| 28 | Smart Pointers: Rc<T> และ RefCell<T> | ✅ |
| 29 | Smart Pointers: Weak<T>, Cow<T>, interior mutability patterns | ✅ |
| 30 | Error Handling ขั้นสูง (custom error types, From/Into, error chains) | ✅ |
| 31 | thiserror และ anyhow ในโปรเจกต์จริง | ✅ |
| 32 | Testing: Unit Tests (#[test], assert!, mocking เบื้องต้น) | ⏳ |
| 33 | Testing: Integration Tests และการจัดระเบียบ test suite | ✅ |
| 34 | Documentation: rustdoc, doc tests, docs.rs | ✅ |
| 35 | Cargo ขั้นสูง (features, profiles, workspaces จริง, cargo.lock) | ⏳ |
| 36 | Macros: Declarative Macros (macro_rules!) | ⏳ |
| 37 | Threads พื้นฐาน (std::thread, join, move closures) | ⏳ |
| 38 | Channels (mpsc, message passing concurrency) | ⏳ |
| 39 | Mutex, Arc และ Shared-State Concurrency | ⏳ |
| 40 | Send, Sync และความปลอดภัยของ Concurrency | ⏳ |

## โมดูล 3: ระดับสูง (Advanced) — Part 41–60

| # | ชื่อบท | สถานะ |
|---|---|---|
| 41 | Unsafe Rust เบื้องต้น | ⏳ |
| 42 | Raw Pointers และ Memory Layout | ⏳ |
| 43 | FFI: การเชื่อมต่อกับ C (extern "C", bindgen) | ⏳ |
| 44 | Procedural Macros เบื้องต้น | ⏳ |
| 45 | Procedural Macros: Derive Macros ขั้นสูง | ⏳ |
| 46 | Async/Await เบื้องต้น | ⏳ |
| 47 | Futures และ Executors (วิธีทำงานภายใน) | ⏳ |
| 48 | Tokio: Runtime และ Tasks | ⏳ |
| 49 | Tokio: I/O และ Networking (TCP/UDP) | ⏳ |
| 50 | Async Channels และ Synchronization (tokio::sync) | ⏳ |
| 51 | Atomics และ Lock-free Programming | ⏳ |
| 52 | Design Patterns ใน Rust (Builder, Strategy, Observer) | ⏳ |
| 53 | Design Patterns ใน Rust (Newtype, Typestate, RAII) | ⏳ |
| 54 | Performance Optimization และ Benchmarking (criterion) | ⏳ |
| 55 | Profiling Rust Applications (flamegraph, perf) | ⏳ |
| 56 | Memory Management ขั้นสูงและ Zero-cost Abstractions | ⏳ |
| 57 | Serialization: Serde เบื้องต้น | ⏳ |
| 58 | Serde ขั้นสูง (custom Serialize/Deserialize) | ⏳ |
| 59 | CLI Applications ด้วย clap | ⏳ |
| 60 | Logging และ Tracing เบื้องต้น (log, tracing crate) | ⏳ |

## โมดูล 4: การพัฒนาเว็บแอปพลิเคชัน (Web Development) — Part 61–85

| # | ชื่อบท | สถานะ |
|---|---|---|
| 61 | HTTP Fundamentals และ REST API Concepts | ⏳ |
| 62 | แนะนำ Axum Framework | ⏳ |
| 63 | Axum: Routing และ Handlers | ⏳ |
| 64 | Axum: State Management และ Extractors | ⏳ |
| 65 | Axum: Middleware (tower, tower-http) | ⏳ |
| 66 | Axum: Error Handling แบบมืออาชีพ | ⏳ |
| 67 | แนะนำ Actix-web Framework | ⏳ |
| 68 | Actix-web: Routing, Handlers, Middleware | ⏳ |
| 69 | เปรียบเทียบ Axum vs Actix-web vs Rocket | ⏳ |
| 70 | Database: เชื่อมต่อ PostgreSQL ด้วย SQLx | ⏳ |
| 71 | SQLx: Queries, Migrations, Connection Pooling | ⏳ |
| 72 | Diesel ORM เบื้องต้น | ⏳ |
| 73 | SeaORM เบื้องต้น | ⏳ |
| 74 | Authentication: JWT | ⏳ |
| 75 | Authentication: Session-based และ OAuth2 | ⏳ |
| 76 | Authorization และ RBAC | ⏳ |
| 77 | WebSockets ด้วย Axum | ⏳ |
| 78 | RESTful API Design Best Practices | ⏳ |
| 79 | GraphQL ด้วย async-graphql | ⏳ |
| 80 | gRPC ด้วย Tonic | ⏳ |
| 81 | Microservices Architecture ด้วย Rust | ⏳ |
| 82 | Message Queues: RabbitMQ/Kafka Integration | ⏳ |
| 83 | Caching ด้วย Redis | ⏳ |
| 84 | Background Jobs และ Task Queues | ⏳ |
| 85 | API Documentation ด้วย OpenAPI/Swagger (utoipa) | ⏳ |

## โมดูล 5: Full-Stack และ WebAssembly — Part 86–95

| # | ชื่อบท | สถานะ |
|---|---|---|
| 86 | WebAssembly เบื้องต้นด้วย Rust | ⏳ |
| 87 | wasm-bindgen และ JavaScript Interop | ⏳ |
| 88 | Yew Framework: Frontend ด้วย Rust | ⏳ |
| 89 | Leptos Framework: Full-stack Rust | ⏳ |
| 90 | Dioxus Framework เบื้องต้น | ⏳ |
| 91 | Server-Side Rendering (SSR) ด้วย Rust | ⏳ |
| 92 | Full-Stack Project (1/3): ออกแบบและสร้าง Backend | ⏳ |
| 93 | Full-Stack Project (2/3): สร้าง Frontend | ⏳ |
| 94 | Full-Stack Project (3/3): Integration และ Deployment | ⏳ |
| 95 | Testing Web Applications แบบครบวงจร (unit/integration/e2e) | ⏳ |

## โมดูล 6: Production, DevOps และระดับมืออาชีพ — Part 96–110

| # | ชื่อบท | สถานะ |
|---|---|---|
| 96 | Docker และ Containerization สำหรับ Rust | ⏳ |
| 97 | CI/CD Pipeline ด้วย GitHub Actions | ⏳ |
| 98 | Observability: Metrics ด้วย Prometheus | ⏳ |
| 99 | Observability: Distributed Tracing ด้วย OpenTelemetry | ⏳ |
| 100 | Security Best Practices ใน Rust | ⏳ |
| 101 | Deployment: Cloud Platforms (AWS/GCP/Fly.io) | ⏳ |
| 102 | Embedded Rust เบื้องต้น | ⏳ |
| 103 | Game Development ด้วย Bevy Engine เบื้องต้น | ⏳ |
| 104 | Blockchain และ Smart Contracts ด้วย Rust | ⏳ |
| 105 | Contributing to Open Source Rust Projects | ⏳ |
| 106 | Rust Design Patterns สำหรับ Enterprise Applications | ⏳ |
| 107 | Capstone: Building a Production-Grade CLI Tool | ⏳ |
| 108 | Capstone: Building a Production-Grade Web Service | ⏳ |
| 109 | Code Review, Refactoring และ Clean Code ใน Rust | ⏳ |
| 110 | เตรียมตัวสัมภาษณ์งาน Rust Developer และแนวทางอาชีพ | ⏳ |

---

หลังจบ Part 110 หลักสูตรจะพิจารณาเพิ่มโมดูลเสริม (เช่น Rust สำหรับ Data Engineering, Rust สำหรับ AI/ML inference,
Rust compiler internals) ตามความเหมาะสม เพื่อให้ครอบคลุมระดับ "World-Class" อย่างแท้จริง
