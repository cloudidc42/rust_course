# โมดูลโปรเจค — 100 โปรเจค Rust ที่รันได้จริง

ส่วนขยายของหลักสูตรหลัก (Parts 1–110) — สร้างโปรเจคจริงครบทุกด้านของ Rust

แต่ละโปรเจคมีเนื้อหา 800–3000+ บรรทัด พร้อมโค้ดที่ compile และรันได้จริง
ขั้นตอนการพัฒนาแบบ progressive และ tests ที่ผ่านจริงทุกตัว

> อ่านหลักสูตรหลัก (Parts 1–110) ก่อน — โปรเจคในที่นี้อ้างอิงความรู้จากทุก Part

---

## โมดูล A: CLI & Systems Tools

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| A01 | Shell Interpreter (bash subset) | ⭐⭐⭐ | ⏳ |
| A02 | HTTP Client (curl-like) | ⭐⭐ | ⏳ |
| A03 | File Watcher Daemon | ⭐⭐ | ⏳ |
| A04 | Port & Service Scanner | ⭐⭐⭐ | ⏳ |
| A05 | DNS Resolver (from scratch) | ⭐⭐⭐⭐ | ⏳ |
| A06 | Terminal Text Editor | ⭐⭐⭐ | ⏳ |
| A07 | Process Manager (PM2-like) | ⭐⭐⭐⭐ | ⏳ |
| A08 | System Monitor (htop-like) | ⭐⭐⭐ | ⏳ |
| A09 | File Deduplicator | ⭐⭐ | ⏳ |
| A10 | Secret Vault (encrypted) | ⭐⭐⭐ | ⏳ |

## โมดูล B: Web Services & APIs

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| B01 | Pastebin Service | ⭐⭐⭐ | ⏳ |
| B02 | File Upload + CDN Proxy | ⭐⭐⭐ | ⏳ |
| B03 | Image Processing API | ⭐⭐⭐ | ⏳ |
| B04 | RSS/Atom Feed Aggregator | ⭐⭐⭐ | ⏳ |
| B05 | Webhook Relay Service | ⭐⭐⭐ | ⏳ |
| B06 | OAuth2 Authorization Server | ⭐⭐⭐⭐ | ⏳ |
| B07 | Real-time Chat (WebSocket) | ⭐⭐⭐ | ⏳ |
| B08 | Email Notification Service | ⭐⭐⭐ | ⏳ |
| B09 | API Gateway with Rate Limiting | ⭐⭐⭐⭐ | ⏳ |
| B10 | Screenshot-as-a-Service | ⭐⭐⭐⭐ | ⏳ |

## โมดูล C: Data Processing & Pipelines

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| C01 | ETL Pipeline Framework | ⭐⭐⭐⭐ | ⏳ |
| C02 | Real-time Analytics Engine | ⭐⭐⭐⭐ | ⏳ |
| C03 | Time Series Database (lite) | ⭐⭐⭐⭐⭐ | ⏳ |
| C04 | Message Broker (pub/sub) | ⭐⭐⭐⭐ | ⏳ |
| C05 | Full-text Search Engine | ⭐⭐⭐⭐⭐ | ⏳ |
| C06 | CSV/Parquet Processor | ⭐⭐⭐ | ⏳ |
| C07 | Database Backup & Restore Tool | ⭐⭐⭐ | ⏳ |
| C08 | Data Migration Framework | ⭐⭐⭐⭐ | ⏳ |
| C09 | Query Language Parser | ⭐⭐⭐⭐⭐ | ⏳ |
| C10 | Schema Registry Service | ⭐⭐⭐⭐ | ⏳ |

## โมดูล D: Security & Cryptography

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| D01 | Password Manager CLI | ⭐⭐⭐ | ⏳ |
| D02 | File Encryptor/Decryptor | ⭐⭐⭐ | ⏳ |
| D03 | TLS-Terminating Proxy | ⭐⭐⭐⭐ | ⏳ |
| D04 | Certificate Manager (ACME) | ⭐⭐⭐⭐ | ⏳ |
| D05 | Audit Log System | ⭐⭐⭐ | ⏳ |
| D06 | Secrets Rotation Daemon | ⭐⭐⭐⭐ | ⏳ |
| D07 | JWT Library (from scratch) | ⭐⭐⭐ | ⏳ |
| D08 | Static Vulnerability Scanner | ⭐⭐⭐⭐⭐ | ⏳ |
| D09 | Rate-limit + WAF Middleware | ⭐⭐⭐⭐ | ⏳ |
| D10 | Zero-Knowledge Proof Demo | ⭐⭐⭐⭐⭐ | ⏳ |

## โมดูล E: Games & Graphics

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| E01 | Tetris (terminal) | ⭐⭐ | ⏳ |
| E02 | Chess Engine + UCI Protocol | ⭐⭐⭐⭐⭐ | ⏳ |
| E03 | 2D Physics Engine | ⭐⭐⭐⭐ | ⏳ |
| E04 | Ray Tracer (path tracing) | ⭐⭐⭐⭐ | ⏳ |
| E05 | Conway's Game of Life (WASM) | ⭐⭐ | ⏳ |
| E06 | Roguelike Dungeon Crawler | ⭐⭐⭐ | ⏳ |
| E07 | Procedural Map Generator | ⭐⭐⭐ | ⏳ |
| E08 | Particle System Simulator | ⭐⭐⭐⭐ | ⏳ |
| E09 | ASCII Art Renderer | ⭐⭐ | ⏳ |
| E10 | Maze Generator/Solver Visualizer | ⭐⭐ | ⏳ |

## โมดูล F: Distributed Systems

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| F01 | Key-Value Store (Raft consensus) | ⭐⭐⭐⭐⭐ | ⏳ |
| F02 | Service Discovery (Consul-like) | ⭐⭐⭐⭐⭐ | ⏳ |
| F03 | Distributed Lock Manager | ⭐⭐⭐⭐ | ⏳ |
| F04 | Circuit Breaker Library | ⭐⭐⭐⭐ | ⏳ |
| F05 | Saga Orchestrator | ⭐⭐⭐⭐⭐ | ⏳ |
| F06 | Consistent Hashing Library | ⭐⭐⭐ | ⏳ |
| F07 | Bloom Filter Service | ⭐⭐⭐ | ⏳ |
| F08 | Leader Election (ZooKeeper-like) | ⭐⭐⭐⭐⭐ | ⏳ |
| F09 | Event Sourcing + CQRS System | ⭐⭐⭐⭐⭐ | ⏳ |
| F10 | Distributed Rate Limiter | ⭐⭐⭐⭐ | ⏳ |

## โมดูล G: DevOps & Infrastructure

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| G01 | Docker Registry (OCI-compliant) | ⭐⭐⭐⭐⭐ | ⏳ |
| G02 | Kubernetes Operator | ⭐⭐⭐⭐⭐ | ⏳ |
| G03 | Infrastructure Drift Detector | ⭐⭐⭐⭐ | ⏳ |
| G04 | Deployment Pipeline Engine | ⭐⭐⭐⭐ | ⏳ |
| G05 | Blue-Green Deployer | ⭐⭐⭐⭐ | ⏳ |
| G06 | Chaos Engineering Toolkit | ⭐⭐⭐⭐ | ⏳ |
| G07 | SLO/SLA Monitor + Alerting | ⭐⭐⭐⭐ | ⏳ |
| G08 | Log Shipper (Fluentd-like) | ⭐⭐⭐⭐ | ⏳ |
| G09 | Config Management Daemon | ⭐⭐⭐⭐ | ⏳ |
| G10 | Cloud Cost Analyzer | ⭐⭐⭐ | ⏳ |

## โมดูล H: Networking & Protocols

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| H01 | HTTP/2 Server (from scratch) | ⭐⭐⭐⭐⭐ | ⏳ |
| H02 | MQTT Broker | ⭐⭐⭐⭐ | ⏳ |
| H03 | SOCKS5 Proxy | ⭐⭐⭐ | ⏳ |
| H04 | VPN Tunnel (WireGuard-like) | ⭐⭐⭐⭐⭐ | ⏳ |
| H05 | Load Balancer (L4/L7) | ⭐⭐⭐⭐ | ⏳ |
| H06 | DNS Server (authoritative) | ⭐⭐⭐⭐ | ⏳ |
| H07 | SMTP Server (receive only) | ⭐⭐⭐⭐ | ⏳ |
| H08 | gRPC Gateway (REST→gRPC) | ⭐⭐⭐⭐ | ⏳ |
| H09 | WebRTC Signaling Server | ⭐⭐⭐⭐⭐ | ⏳ |
| H10 | Reverse Proxy + Cache | ⭐⭐⭐⭐ | ⏳ |

## โมดูล I: Machine Learning & AI

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| I01 | Neural Network (from scratch) | ⭐⭐⭐⭐ | ⏳ |
| I02 | Decision Tree + Random Forest | ⭐⭐⭐⭐ | ⏳ |
| I03 | K-means / DBSCAN Clustering | ⭐⭐⭐ | ⏳ |
| I04 | NLP Tokenizer (BPE) | ⭐⭐⭐⭐ | ⏳ |
| I05 | Vector Similarity Search (HNSW) | ⭐⭐⭐⭐⭐ | ⏳ |
| I06 | LLM Inference Runtime | ⭐⭐⭐⭐⭐ | ⏳ |
| I07 | OCR Preprocessing Pipeline | ⭐⭐⭐ | ⏳ |
| I08 | Anomaly Detection Service | ⭐⭐⭐⭐ | ⏳ |
| I09 | Recommendation Engine | ⭐⭐⭐⭐ | ⏳ |
| I10 | ONNX Model Server | ⭐⭐⭐⭐⭐ | ⏳ |

## โมดูล J: Full-Stack & WASM Projects

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| J01 | Markdown Editor (WASM) | ⭐⭐⭐ | ⏳ |
| J02 | Spreadsheet App (browser) | ⭐⭐⭐⭐ | ⏳ |
| J03 | Real-time Collaborative Editor | ⭐⭐⭐⭐⭐ | ⏳ |
| J04 | Browser-based SQLite Explorer | ⭐⭐⭐ | ⏳ |
| J05 | Kanban Board (full-stack) | ⭐⭐⭐⭐ | ⏳ |
| J06 | E-commerce API (full) | ⭐⭐⭐⭐ | ⏳ |
| J07 | Blog CMS (headless) | ⭐⭐⭐⭐ | ⏳ |
| J08 | GraphQL + REST Unified API | ⭐⭐⭐⭐ | ⏳ |
| J09 | Admin Dashboard (full-stack) | ⭐⭐⭐⭐ | ⏳ |
| J10 | Real-time Multiplayer Game | ⭐⭐⭐⭐⭐ | ⏳ |

---

สถานะ: ✅ เสร็จแล้ว · 🚧 กำลังเขียน · ⏳ รอดำเนินการ

แนวทางการเขียนและมาตรฐานเนื้อหาอยู่ที่ [`STYLE_GUIDE.md`](./STYLE_GUIDE.md)
