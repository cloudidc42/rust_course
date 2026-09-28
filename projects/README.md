# โมดูลโปรเจค — 100 โปรเจค Rust ที่รันได้จริง

ส่วนขยายของหลักสูตรหลัก (Parts 1–110) — สร้างโปรเจคจริงครบทุกด้านของ Rust

แต่ละโปรเจคมีเนื้อหา 1200–3000+ บรรทัด พร้อมโค้ดที่ compile และรันได้จริง
ขั้นตอนการพัฒนาแบบ progressive และ tests ที่ผ่านจริงทุกตัว

> อ่านหลักสูตรหลัก (Parts 1–110) ก่อน — โปรเจคในที่นี้อ้างอิงความรู้จากทุก Part

---

## โมดูล A: CLI & Systems Tools

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| A01 | [Shell Interpreter (bash subset)](project-a01-shell-interpreter.md) | ⭐⭐⭐ | ✅ |
| A02 | [HTTP Client (curl-like)](project-a02-http-client.md) | ⭐⭐ | ✅ |
| A03 | [File Watcher Daemon](project-a03-file-watcher.md) | ⭐⭐ | ✅ |
| A04 | [Port & Service Scanner](project-a04-port-scanner.md) | ⭐⭐⭐ | ✅ |
| A05 | [DNS Resolver (from scratch)](project-a05-dns-resolver.md) | ⭐⭐⭐⭐ | ✅ |
| A06 | [Terminal Text Editor](project-a06-text-editor.md) | ⭐⭐⭐ | ✅ |
| A07 | [Process Manager (PM2-like)](project-a07-process-manager.md) | ⭐⭐⭐⭐ | ✅ |
| A08 | [System Monitor (htop-like)](project-a08-system-monitor.md) | ⭐⭐⭐ | ✅ |
| A09 | [File Deduplicator](project-a09-file-deduplicator.md) | ⭐⭐ | ✅ |
| A10 | [Secret Vault (encrypted)](project-a10-secret-vault.md) | ⭐⭐⭐ | ✅ |

## โมดูล B: Web Services & APIs

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| B01 | [Pastebin Service](project-b01-pastebin.md) | ⭐⭐⭐ | ✅ |
| B02 | [File Upload + CDN Proxy](project-b02-file-upload.md) | ⭐⭐⭐ | ✅ |
| B03 | [Image Processing API](project-b03-image-api.md) | ⭐⭐⭐ | ✅ |
| B04 | [RSS/Atom Feed Aggregator](project-b04-rss-feed.md) | ⭐⭐⭐ | ✅ |
| B05 | [Webhook Relay Service](project-b05-webhook-relay.md) | ⭐⭐⭐ | ✅ |
| B06 | [OAuth2 Authorization Server](project-b06-oauth2.md) | ⭐⭐⭐⭐ | ✅ |
| B07 | [Real-time Chat (WebSocket)](project-b07-websocket-chat.md) | ⭐⭐⭐ | ✅ |
| B08 | [Email Notification Service](project-b08-email-service.md) | ⭐⭐⭐ | ✅ |
| B09 | [API Gateway with Rate Limiting](project-b09-api-gateway.md) | ⭐⭐⭐⭐ | ✅ |
| B10 | [Screenshot-as-a-Service](project-b10-screenshot.md) | ⭐⭐⭐⭐ | ✅ |

## โมดูล C: Data Processing & Pipelines

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| C01 | [ETL Pipeline Framework](project-c01-etl-pipeline.md) | ⭐⭐⭐⭐ | ✅ |
| C02 | [Real-time Analytics Engine](project-c02-analytics.md) | ⭐⭐⭐⭐ | ✅ |
| C03 | [Time Series Database (lite)](project-c03-time-series-db.md) | ⭐⭐⭐⭐⭐ | ✅ |
| C04 | [Message Broker (pub/sub)](project-c04-message-broker.md) | ⭐⭐⭐⭐ | ✅ |
| C05 | [Full-text Search Engine](project-c05-search-engine.md) | ⭐⭐⭐⭐⭐ | ✅ |
| C06 | [CSV/Parquet Processor](project-c06-csv-processor.md) | ⭐⭐⭐ | ✅ |
| C07 | [Database Backup & Restore Tool](project-c07-db-backup.md) | ⭐⭐⭐ | ✅ |
| C08 | [Data Migration Framework](project-c08-migration-framework.md) | ⭐⭐⭐⭐ | ✅ |
| C09 | [Query Language Parser](project-c09-query-parser.md) | ⭐⭐⭐⭐⭐ | ✅ |
| C10 | [Schema Registry Service](project-c10-schema-registry.md) | ⭐⭐⭐⭐ | ✅ |

## โมดูล D: Security & Cryptography

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| D01 | [Password Manager CLI](project-d01-password-manager.md) | ⭐⭐⭐ | ✅ |
| D02 | [File Encryptor/Decryptor](project-d02-file-encryptor.md) | ⭐⭐⭐ | ✅ |
| D03 | [TLS-Terminating Proxy](project-d03-tls-proxy.md) | ⭐⭐⭐⭐ | ✅ |
| D04 | [Certificate Manager (ACME)](project-d04-cert-manager.md) | ⭐⭐⭐⭐ | ✅ |
| D05 | [Audit Log System](project-d05-audit-log.md) | ⭐⭐⭐ | ✅ |
| D06 | [Secrets Rotation Daemon](project-d06-secrets-rotation.md) | ⭐⭐⭐⭐ | ✅ |
| D07 | [JWT Library (from scratch)](project-d07-jwt-library.md) | ⭐⭐⭐ | ✅ |
| D08 | [Static Vulnerability Scanner](project-d08-vulnerability-scanner.md) | ⭐⭐⭐⭐⭐ | ✅ |
| D09 | [Rate-limit + WAF Middleware](project-d09-waf-middleware.md) | ⭐⭐⭐⭐ | ✅ |
| D10 | [Zero-Knowledge Proof Demo](project-d10-zkp-demo.md) | ⭐⭐⭐⭐⭐ | ✅ |

## โมดูล E: Games & Graphics

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| E01 | [Tetris (terminal)](project-e01-tetris.md) | ⭐⭐ | ✅ |
| E02 | [Chess Engine + UCI Protocol](project-e02-chess-engine.md) | ⭐⭐⭐⭐⭐ | ✅ |
| E03 | [2D Physics Engine](project-e03-physics-engine.md) | ⭐⭐⭐⭐ | ✅ |
| E04 | [Ray Tracer (path tracing)](project-e04-ray-tracer.md) | ⭐⭐⭐⭐ | ✅ |
| E05 | [Conway's Game of Life (WASM)](project-e05-game-of-life.md) | ⭐⭐ | ✅ |
| E06 | [Roguelike Dungeon Crawler](project-e06-roguelike.md) | ⭐⭐⭐ | ✅ |
| E07 | [Procedural Map Generator](project-e07-map-generator.md) | ⭐⭐⭐ | ✅ |
| E08 | [Particle System Simulator](project-e08-particle-system.md) | ⭐⭐⭐⭐ | ✅ |
| E09 | [ASCII Art Renderer](project-e09-ascii-art.md) | ⭐⭐ | ✅ |
| E10 | [Maze Generator/Solver Visualizer](project-e10-maze-generator.md) | ⭐⭐ | ✅ |

## โมดูล F: Distributed Systems

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| F01 | [Key-Value Store (Raft Consensus)](project-f01-raft-kv.md) | ⭐⭐⭐⭐⭐ | ✅ |
| F02 | [Service Discovery (Consul-like)](project-f02-service-discovery.md) | ⭐⭐⭐⭐⭐ | ✅ |
| F03 | [Distributed Lock Manager](project-f03-dist-lock.md) | ⭐⭐⭐⭐ | ✅ |
| F04 | [Circuit Breaker Library](project-f04-circuit-breaker.md) | ⭐⭐⭐⭐ | ✅ |
| F05 | [Saga Orchestrator](project-f05-saga.md) | ⭐⭐⭐⭐⭐ | ✅ |
| F06 | [Event Sourcing + CQRS System](project-f06-event-sourcing.md) | ⭐⭐⭐⭐⭐ | ✅ |
| F07 | [CRDT Data Structures](project-f07-crdts.md) | ⭐⭐⭐⭐ | ✅ |
| F08 | [Consistent Hashing Library](project-f08-consistent-hash.md) | ⭐⭐⭐ | ✅ |
| F09 | [Distributed Tracing System](project-f09-dist-tracing.md) | ⭐⭐⭐⭐ | ✅ |
| F10 | [Distributed Message Queue](project-f10-message-queue.md) | ⭐⭐⭐⭐ | ✅ |

## โมดูล G: DevOps & Infrastructure

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| G01 | [Docker Image Builder (OCI)](project-g01-docker-builder.md) | ⭐⭐⭐⭐⭐ | ✅ |
| G02 | [CI/CD Pipeline Runner](project-g02-ci-runner.md) | ⭐⭐⭐⭐ | ✅ |
| G03 | [Log Aggregator & Query Engine](project-g03-log-aggregator.md) | ⭐⭐⭐⭐ | ✅ |
| G04 | [Metrics Collector (Prometheus-like)](project-g04-metrics-collector.md) | ⭐⭐⭐⭐ | ✅ |
| G05 | [Config Manager with Hot-Reload](project-g05-config-manager.md) | ⭐⭐⭐⭐ | ✅ |
| G06 | [Health Check Service](project-g06-health-checker.md) | ⭐⭐⭐ | ✅ |
| G07 | [Secret Scanner (SAST)](project-g07-secret-scanner.md) | ⭐⭐⭐⭐ | ✅ |
| G08 | [Infrastructure as Code Engine](project-g08-infra-as-code.md) | ⭐⭐⭐⭐⭐ | ✅ |
| G09 | [Distributed Rate Limiter](project-g09-rate-limiter.md) | ⭐⭐⭐⭐ | ✅ |
| G10 | [Blue-Green Deployment Controller](project-g10-blue-green.md) | ⭐⭐⭐⭐ | ✅ |

## โมดูล H: Networking & Protocols

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| H01 | [HTTP/1.1 + HTTP/2 Server (from scratch)](project-h01-http-server.md) | ⭐⭐⭐⭐⭐ | ✅ |
| H02 | [WebSocket Server (RFC 6455)](project-h02-websocket.md) | ⭐⭐⭐⭐ | ✅ |
| H03 | [DNS Stub Resolver (RFC 1035)](project-h03-dns-resolver.md) | ⭐⭐⭐⭐ | ✅ |
| H04 | [MQTT Broker (QoS 0/1)](project-h04-mqtt-broker.md) | ⭐⭐⭐⭐ | ✅ |
| H05 | [Load Balancer (L4/L7)](project-h05-load-balancer.md) | ⭐⭐⭐⭐ | ✅ |
| H06 | [Protocol Buffer Codec (from scratch)](project-h06-protobuf-codec.md) | ⭐⭐⭐⭐ | ✅ |
| H07 | [gRPC Service Framework (tonic)](project-h07-grpc-service.md) | ⭐⭐⭐⭐ | ✅ |
| H08 | [TCP Chat Server](project-h08-tcp-chat.md) | ⭐⭐⭐ | ✅ |
| H09 | [QUIC/UDP Reliable Transport](project-h09-quic-transport.md) | ⭐⭐⭐⭐⭐ | ✅ |
| H10 | [Network Proxy (SOCKS5 + HTTP CONNECT)](project-h10-network-proxy.md) | ⭐⭐⭐⭐ | ✅ |

## โมดูล I: Machine Learning & AI

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| I01 | [Linear Regression from Scratch](project-i01-linear-regression.md) | ⭐⭐⭐ | ✅ |
| I02 | [Neural Network from Scratch](project-i02-neural-network.md) | ⭐⭐⭐⭐ | ✅ |
| I03 | [Decision Tree & Random Forest](project-i03-decision-tree.md) | ⭐⭐⭐⭐ | ✅ |
| I04 | [Clustering Algorithms (k-means, DBSCAN)](project-i04-clustering.md) | ⭐⭐⭐ | ✅ |
| I05 | [NLP Tokenizer & BPE](project-i05-nlp-tokenizer.md) | ⭐⭐⭐⭐ | ✅ |
| I06 | [Recommender System (Collaborative Filtering)](project-i06-recommender.md) | ⭐⭐⭐⭐ | ✅ |
| I07 | [Time Series Forecasting](project-i07-time-series.md) | ⭐⭐⭐⭐ | ✅ |
| I08 | [Genetic Algorithm](project-i08-genetic-algorithm.md) | ⭐⭐⭐ | ✅ |
| I09 | [Q-Learning / Reinforcement Learning](project-i09-q-learning.md) | ⭐⭐⭐⭐ | ✅ |
| I10 | [Graph Neural Network (GCN)](project-i10-graph-nn.md) | ⭐⭐⭐⭐⭐ | ✅ |

## โมดูล J: Full-Stack & WASM Projects

| ID | โปรเจค | ความยาก | สถานะ |
|---|---|---|---|
| J01 | [Yew SPA Frontend](project-j01-yew-spa.md) | ⭐⭐⭐ | ✅ |
| J02 | [Full-Stack Axum + Yew](project-j02-fullstack-axum.md) | ⭐⭐⭐⭐ | ✅ |
| J03 | [WASM Image Processor](project-j03-wasm-image.md) | ⭐⭐⭐ | ✅ |
| J04 | [WebAssembly Plugin System (wasmtime)](project-j04-wasm-plugin.md) | ⭐⭐⭐⭐ | ✅ |
| J05 | [Real-Time Collaborative Editor (OT)](project-j05-collaborative-editor.md) | ⭐⭐⭐⭐⭐ | ✅ |
| J06 | [GraphQL API (async-graphql)](project-j06-graphql-api.md) | ⭐⭐⭐⭐ | ✅ |
| J07 | [REST API with OpenAPI (utoipa)](project-j07-rest-openapi.md) | ⭐⭐⭐ | ✅ |
| J08 | [Server-Sent Events (SSE) Server](project-j08-sse-server.md) | ⭐⭐⭐ | ✅ |
| J09 | [WASM with wasm-bindgen + npm](project-j09-wasm-bindgen.md) | ⭐⭐⭐⭐ | ✅ |
| J10 | [Full-Stack App with Authentication (JWT + Axum + Yew)](project-j10-fullstack-auth.md) | ⭐⭐⭐⭐⭐ | ✅ |

---

สถานะ: ✅ เสร็จแล้ว · 🚧 กำลังเขียน · ⏳ รอดำเนินการ

แนวทางการเขียนและมาตรฐานเนื้อหาอยู่ที่ [`STYLE_GUIDE.md`](./STYLE_GUIDE.md)
