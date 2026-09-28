# Project H07: gRPC Service Framework

> โมดูล: H — Networking/Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 18 ชั่วโมง

## ภาพรวมโปรเจค

**gRPC** (gRPC Remote Procedure Call) คือ framework สำหรับเรียกใช้ฟังก์ชันข้ามเครื่อง (remote procedure call) ที่สร้างโดย Google โดยใช้ **Protocol Buffers** เป็น interface definition language (IDL) และ **HTTP/2** เป็น transport layer gRPC เป็นรากฐานของ microservice architecture ระดับ production ในบริษัทเทคโนโลยียักษ์ใหญ่ เช่น Google, Netflix, Cloudflare, Dropbox และ Lyft

### ทำไม gRPC ถึงสำคัญ?

เปรียบเทียบกับ REST API แบบเดิม:

| คุณสมบัติ | REST/JSON | gRPC/Protobuf |
|----------|-----------|---------------|
| **Serialization** | JSON (text, slow) | Protobuf binary (3-10x เร็วกว่า) |
| **Schema** | optional (OpenAPI) | บังคับ (.proto file) |
| **Streaming** | ยาก (SSE/WebSocket) | built-in (4 รูปแบบ) |
| **Code generation** | ต้องใช้ tools เสริม | สร้างอัตโนมัติจาก .proto |
| **Type safety** | runtime เท่านั้น | compile-time ทั้ง client/server |
| **Transport** | HTTP/1.1 | HTTP/2 (multiplexing) |
| **Error codes** | HTTP status code | gRPC status codes (17 รายการ) |

gRPC รองรับ **4 รูปแบบ RPC**:
1. **Unary RPC** — request หนึ่งชิ้น, response หนึ่งชิ้น (เหมือน REST)
2. **Server Streaming** — request หนึ่งชิ้น, server ส่ง response เป็น stream
3. **Client Streaming** — client ส่ง request เป็น stream, server ตอบหนึ่งชิ้น
4. **Bidirectional Streaming** — ทั้งสองฝั่ง stream พร้อมกัน

โปรเจคนี้สร้าง **Product Inventory gRPC Service** — บริการจัดการสินค้าคงคลังที่ครบครัน ประกอบด้วย:

- **ProductService** ที่มี 4 RPCs: GetProduct, ListProducts, CreateProduct, WatchInventory (server streaming)
- **In-memory store** ด้วย `DashMap<u32, Product>` ที่ thread-safe
- **Server streaming** ด้วย `tokio_stream::wrappers::ReceiverStream`
- **Client** ที่ connect และเรียกใช้ทุก RPC รวมถึงรับ streaming response
- **Interceptors** สำหรับ authentication และ logging
- **Error handling** ที่ map domain errors ไปเป็น gRPC status codes ที่ถูกต้อง

ในโลก production แบบเดียวกันนี้ใช้กับ **Envoy proxy**, **gRPC-Gateway**, **Kubernetes gRPC health check**, และ service mesh ทุกตัว

---

## สิ่งที่จะได้เรียนรู้

- **tonic framework** — สร้าง gRPC server/client ด้วย Rust อย่างถูกต้อง
- **prost + tonic-build** — complie `.proto` file เป็น Rust struct อัตโนมัติผ่าน `build.rs`
- **Server streaming** — ส่ง stream ของ response ด้วย `ReceiverStream` + `tokio::sync::mpsc`
- **tonic interceptors** — เขียน middleware ที่ตรวจสอบ metadata header ก่อนทุก RPC
- **gRPC metadata** — อ่าน/เขียน header พิเศษใน `Request::metadata()` และ `Response::metadata_mut()`
- **DashMap concurrent store** — ใช้ `DashMap<K, V>` แทน `Mutex<HashMap>` เพื่อ performance สูงกว่า
- **tonic::Status** — สร้าง error response ด้วย status code ที่ถูกต้อง (not_found, invalid_argument, already_exists, internal, unauthenticated)
- **From trait สำหรับ error mapping** — แปลง domain error เป็น `Status` ด้วย `impl From<DomainError> for Status`

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50** — `async/await`, tokio runtime, `Future`, `tokio::spawn`
- **Part 51–60** — `Arc<T>`, `Mutex<T>`, trait objects `dyn Trait + Send + Sync`
- **Part 61–70** — Atomic types, interior mutability patterns
- **Part 21–25** — struct, enum, impl blocks, pattern matching
- **Part 31–35** — error handling ด้วย `Result`, `?` operator, custom error types
- **Part 96–110** — binary encoding, byte manipulation (เพื่อทำความเข้าใจ Protobuf internals)
- โปรเจค H06 (Protobuf Codec) — ความเข้าใจ Protobuf wire format และ `prost`

---

## โครงสร้างโปรเจค (Project Layout)

```
grpc-service/
├── proto/
│   └── product.proto          # service + message definitions
├── src/
│   ├── lib.rs                 # store, service impl, interceptors, error mapping, tests
│   ├── server.rs              # binary: start tonic server
│   └── client.rs              # binary: demo client ที่เรียกทุก RPC
├── build.rs                   # invoke tonic-build เพื่อ compile .proto
└── Cargo.toml
```

---

## การออกแบบ (Architecture & Design)

### Data Flow ของ gRPC Request

```
gRPC Client (HTTP/2)
      │
      │  Protobuf-encoded bytes over HTTP/2 stream
      ▼
┌─────────────────────────────────────────────────┐
│  tonic Server (tokio runtime)                   │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Interceptor Chain                       │   │
│  │  1. AuthInterceptor.intercept(req)       │   │
│  │     └─ check metadata["authorization"]  │   │
│  │  2. LoggingInterceptor.intercept(req)    │   │
│  │     └─ record x-grpc-method             │   │
│  └──────────────┬───────────────────────────┘   │
│                 │ Request<T> (decoded)           │
│  ┌──────────────▼───────────────────────────┐   │
│  │  ProductServiceImpl                      │   │
│  │  ├─ get_product(id) → Product            │   │
│  │  ├─ list_products(filter) → ProductList  │   │
│  │  ├─ create_product(req) → Product        │   │
│  │  └─ watch_inventory(ids) → Stream<...>   │   │
│  └──────────────┬───────────────────────────┘   │
│                 │ touches                        │
│  ┌──────────────▼───────────────────────────┐   │
│  │  ProductStore                            │   │
│  │  DashMap<u32, Product>  (thread-safe)    │   │
│  │  AtomicU32 next_id                       │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
      │
      │  Protobuf-encoded response
      ▼
gRPC Client ← decode → Rust struct
```

### ทำไมใช้ `DashMap` แทน `Mutex<HashMap>`?

`DashMap` ใช้ **sharded locking** — แบ่ง map ออกเป็น N shard แต่ละ shard มี lock ของตัวเอง ทำให้ concurrent read/write ที่ต่าง key กันไม่ต้อง contend lock เดียวกัน

```
Mutex<HashMap>                 DashMap (16 shards)
┌──────────────┐               ┌──┬──┬──┬──┐
│  Global lock │               │S0│S1│S2│..│
│ ┌──────────┐ │               ├──┼──┼──┼──┤
│ │ HashMap  │ │               │  │  │  │  │
│ └──────────┘ │               └──┴──┴──┴──┘
└──────────────┘               each shard has own lock
Thread A ──wait──► lock        Thread A → shard 3
Thread B ──wait──► wait        Thread B → shard 7 (no conflict)
```

### Server Streaming ด้วย `ReceiverStream`

```
watch_inventory RPC call
        │
        ▼
┌─────────────────────────────────┐
│ mpsc::channel::<Result<...>>(32) │
│       tx          rx            │
│       ▼           ▼             │
│  background    ReceiverStream   │
│  tokio::spawn  wrapped rx       │
│  loop(ids)     returned to      │
│  tx.send(ok)   tonic server     │
└─────────────────────────────────┘
        │
        ▼ HTTP/2 DATA frames (one per message)
gRPC Client ← stream.message().await ─► update
```

### gRPC Status Codes ที่ใช้บ่อย

| Code | Numeric | ใช้เมื่อ |
|------|---------|---------|
| `OK` | 0 | สำเร็จ |
| `NOT_FOUND` | 5 | resource ไม่พบ |
| `ALREADY_EXISTS` | 6 | resource นั้นมีอยู่แล้ว |
| `INVALID_ARGUMENT` | 3 | input ผิด format/range |
| `UNAUTHENTICATED` | 16 | token หาย/ผิด |
| `PERMISSION_DENIED` | 7 | มี token แต่ไม่มีสิทธิ์ |
| `INTERNAL` | 13 | server error ที่ไม่คาดคิด |
| `UNAVAILABLE` | 14 | server กำลัง overloaded |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: กำหนด Proto Schema

สร้างโปรเจคใหม่ก่อน:

```bash
cargo new grpc-service --lib
cd grpc-service
mkdir proto
```

ไฟล์ `proto/product.proto` — นี่คือ "contract" ระหว่าง client กับ server ทุกฝ่ายต้อง generate code จากไฟล์เดียวกัน:

```protobuf
syntax = "proto3";

package product;

// ProductService รองรับ 4 RPC patterns
service ProductService {
    // Unary RPC — ขอสินค้าด้วย ID
    rpc GetProduct(GetProductRequest) returns (Product);

    // Unary RPC — ค้นหาสินค้าด้วย filter
    rpc ListProducts(ProductFilter) returns (ProductList);

    // Unary RPC — สร้างสินค้าใหม่
    rpc CreateProduct(CreateProductRequest) returns (Product);

    // Server Streaming RPC — รับ stream ของการเปลี่ยนแปลง stock
    rpc WatchInventory(WatchInventoryRequest) returns (stream InventoryUpdate);
}

message Product {
    uint32 id    = 1;
    string name  = 2;
    double price = 3;
    uint32 stock = 4;
}

message ProductFilter {
    double min_price     = 1;  // 0.0 หมายถึงไม่กรอง
    double max_price     = 2;  // 0.0 หมายถึงไม่กรอง
    bool   in_stock_only = 3;
}

message ProductList {
    repeated Product products = 1;
}

message GetProductRequest {
    uint32 id = 1;
}

message CreateProductRequest {
    string name  = 1;
    double price = 2;
    uint32 stock = 3;
}

message WatchInventoryRequest {
    repeated uint32 product_ids = 1;
}

message InventoryUpdate {
    uint32 product_id  = 1;
    uint32 new_stock   = 2;
    string description = 3;
}
```

**หลักการตั้งชื่อ field number ใน Protobuf:**
- Field number 1–15 ใช้ 1 byte encoding (ควรใช้กับ field ที่ใช้บ่อย)
- Field number 16–2047 ใช้ 2 bytes
- ห้ามเปลี่ยน field number หลัง deploy — จะทำให้ backward compatibility พัง

---

### ขั้นที่ 2: ตั้งค่า Cargo.toml และ build.rs

เพิ่ม dependencies ใน `Cargo.toml`:

```toml
[package]
name = "grpc-service"
version = "0.1.0"
edition = "2021"

# binary สำหรับ start server
[[bin]]
name = "server"
path = "src/server.rs"

# binary สำหรับ demo client
[[bin]]
name = "client"
path = "src/client.rs"

# library หลักที่ใส่ service logic + tests
[lib]
name = "grpc_service"
path = "src/lib.rs"

[dependencies]
tonic        = { version = "0.12", features = ["transport"] }
prost        = "0.13"
tokio        = { version = "1", features = ["full"] }
tokio-stream = { version = "0.1", features = ["sync"] }
dashmap      = "6"

[build-dependencies]
tonic-build = "0.12"
```

**ทำไมแต่ละ crate?**

| Crate | บทบาท |
|-------|-------|
| `tonic` | gRPC server/client framework, interceptor, status |
| `prost` | Protobuf encoding/decoding (ถูกใช้โดย tonic) |
| `tokio` | async runtime, mpsc channel, spawn |
| `tokio-stream` | `ReceiverStream` wrapper สำหรับ server streaming |
| `dashmap` | concurrent HashMap สำหรับ in-memory store |
| `tonic-build` | compile-time .proto → Rust code generation |

สร้างไฟล์ `build.rs` — รันก่อน compile ทุกครั้ง:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        .build_server(true)   // generate ProductServiceServer
        .build_client(true)   // generate ProductServiceClient
        .compile_protos(
            &["proto/product.proto"],  // source .proto files
            &["proto"],                // include directories
        )?;
    Ok(())
}
```

`tonic-build` จะ:
1. รัน `protoc` เพื่อ parse `.proto`
2. สร้างไฟล์ Rust ใน `$OUT_DIR/product.rs`
3. สร้าง trait `ProductService` ที่เราต้อง implement
4. สร้าง `ProductServiceServer<T>` wrapper
5. สร้าง `ProductServiceClient<Channel>`

include code ที่ generate ได้ใน `src/lib.rs`:

```rust
pub mod proto {
    // tonic::include_proto! macro เทียบเท่ากับ include_str! กับ OUT_DIR
    tonic::include_proto!("product");
}
```

---

### ขั้นที่ 3: In-Memory Store ด้วย DashMap

`ProductStore` เป็น thread-safe store ที่ share ได้ระหว่าง async task หลาย ๆ ตัว:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU32, Ordering};
use dashmap::DashMap;
use crate::proto::{Product, ProductFilter};

#[derive(Clone)]
pub struct ProductStore {
    pub inner:   Arc<DashMap<u32, Product>>,
    pub next_id: Arc<AtomicU32>,
}

impl ProductStore {
    pub fn new() -> Self {
        Self {
            inner:   Arc::new(DashMap::new()),
            next_id: Arc::new(AtomicU32::new(1)),
        }
    }

    /// แทรกสินค้าใหม่ — กำหนด ID อัตโนมัติ, คืนค่า Product ที่มี ID แล้ว
    pub fn insert(&self, mut product: Product) -> Product {
        let id = self.next_id.fetch_add(1, Ordering::SeqCst);
        product.id = id;
        self.inner.insert(id, product.clone());
        product
    }

    /// ดึงสินค้าด้วย ID — คืน None ถ้าไม่พบ
    pub fn get(&self, id: u32) -> Option<Product> {
        self.inner.get(&id).map(|r| r.clone())
    }

    /// filter ตาม price range และ stock status
    pub fn list_filtered(&self, filter: &ProductFilter) -> Vec<Product> {
        self.inner
            .iter()
            .filter_map(|entry| {
                let p = entry.value();
                // ถ้า min_price == 0.0 หมายถึงไม่กรอง
                if filter.min_price > 0.0 && p.price < filter.min_price {
                    return None;
                }
                // ถ้า max_price == 0.0 หมายถึงไม่กรอง
                if filter.max_price > 0.0 && p.price > filter.max_price {
                    return None;
                }
                if filter.in_stock_only && p.stock == 0 {
                    return None;
                }
                Some(p.clone())
            })
            .collect()
    }

    /// ตรวจสอบว่ามีสินค้าชื่อซ้ำหรือไม่ (case-sensitive)
    pub fn contains_name(&self, name: &str) -> bool {
        self.inner.iter().any(|entry| entry.value().name == name)
    }
}
```

**เหตุผลที่ clone ข้างใน `get()`:** `DashMap` คืน `dashmap::mapref::one::Ref<'_, K, V>` ที่ hold shard lock อยู่ เราต้อง clone ออกมาก่อนปล่อย lock ไม่งั้น borrow checker จะไม่ยอม return reference ข้าม async boundary

---

### ขั้นที่ 4: Implement ProductService Trait

`tonic-build` สร้าง trait `ProductService` มาให้ เราต้องเขียน `impl ProductService for ProductServiceImpl`:

```rust
use tonic::{Request, Response, Status};
use crate::proto::{
    product_service_server::ProductService,
    Product, ProductFilter, ProductList,
    GetProductRequest, CreateProductRequest,
};

#[derive(Clone)]
pub struct ProductServiceImpl {
    pub store: ProductStore,
}

impl ProductServiceImpl {
    pub fn new() -> Self {
        Self { store: ProductStore::new() }
    }
}

#[tonic::async_trait]
impl ProductService for ProductServiceImpl {

    // --- GetProduct ---
    async fn get_product(
        &self,
        request: Request<GetProductRequest>,
    ) -> Result<Response<Product>, Status> {
        let id = request.into_inner().id;
        match self.store.get(id) {
            Some(p) => Ok(Response::new(p)),
            None    => Err(Status::not_found(format!("product {} not found", id))),
        }
    }

    // --- ListProducts ---
    async fn list_products(
        &self,
        request: Request<ProductFilter>,
    ) -> Result<Response<ProductList>, Status> {
        let filter = request.into_inner();
        let mut products = self.store.list_filtered(&filter);
        products.sort_by_key(|p| p.id);  // deterministic order
        Ok(Response::new(ProductList { products }))
    }

    // --- CreateProduct ---
    async fn create_product(
        &self,
        request: Request<CreateProductRequest>,
    ) -> Result<Response<Product>, Status> {
        let req = request.into_inner();

        // validation
        if req.name.trim().is_empty() {
            return Err(Status::invalid_argument("product name must not be empty"));
        }
        if req.price < 0.0 {
            return Err(Status::invalid_argument("price must be non-negative"));
        }

        // duplicate check
        if self.store.contains_name(&req.name) {
            return Err(Status::already_exists(
                format!("product '{}' already exists", req.name)
            ));
        }

        let product = Product {
            id: 0,            // store จะกำหนด id
            name:  req.name,
            price: req.price,
            stock: req.stock,
        };
        let created = self.store.insert(product);
        Ok(Response::new(created))
    }

    // WatchInventory อยู่ใน Step 5
    // ...
}
```

**สังเกต `#[tonic::async_trait]`** — เนื่องจาก trait ที่มี async method ใน Rust ยังต้องใช้ macro นี้ช่วย (ปกติ async fn ใน trait ต้องการ `async-trait` crate ก่อน Rust edition 2024) tonic bundle `async_trait` มาให้ในตัว

---

### ขั้นที่ 5: Server Streaming — WatchInventory

Server streaming RPC คือฟีเจอร์ที่ทำให้ gRPC แตกต่างจาก REST อย่างชัดเจน:

```rust
use tokio::sync::mpsc;
use tokio_stream::wrappers::ReceiverStream;
use crate::proto::{WatchInventoryRequest, InventoryUpdate};

// ภายใน impl ProductService for ProductServiceImpl:
type WatchInventoryStream = ReceiverStream<Result<InventoryUpdate, Status>>;

async fn watch_inventory(
    &self,
    request: Request<WatchInventoryRequest>,
) -> Result<Response<Self::WatchInventoryStream>, Status> {
    let product_ids = request.into_inner().product_ids;

    // validation: ต้องระบุ product ID อย่างน้อยหนึ่งตัว
    if product_ids.is_empty() {
        return Err(Status::invalid_argument(
            "must provide at least one product_id"
        ));
    }

    // สร้าง channel — buffer 32 messages
    // tx: ส่งข้อมูลจาก background task
    // rx: wrap เป็น stream ส่งกลับให้ client
    let (tx, rx) = mpsc::channel::<Result<InventoryUpdate, Status>>(32);
    let store = self.store.clone();  // clone Arc pointer (cheap)

    tokio::spawn(async move {
        for &id in &product_ids {
            if let Some(p) = store.get(id) {
                let update = InventoryUpdate {
                    product_id:  id,
                    new_stock:   p.stock,
                    description: format!("initial stock for '{}'", p.name),
                };
                // ส่ง Ok(update) ผ่าน channel
                // ถ้า client disconnect แล้ว tx.send จะ return Err
                if tx.send(Ok(update)).await.is_err() {
                    break;  // client หยุดรับแล้ว — หยุด streaming
                }
            } else {
                // ส่ง error สำหรับ ID ที่ไม่พบ แล้วไปต่อ ID ถัดไป
                let err = Status::not_found(format!("product {} not found", id));
                let _ = tx.send(Err(err)).await;
            }
        }
        // เมื่อ drop tx, channel ปิด → ReceiverStream จะ complete
    });

    // ReceiverStream<Result<InventoryUpdate, Status>> คือ type ที่
    // tonic เข้าใจว่าเป็น streaming response
    Ok(Response::new(ReceiverStream::new(rx)))
}
```

**กลไกการ Stream:**

```
tokio::spawn task          Channel          tonic server
       │                 (buffer=32)             │
       │──── tx.send(Ok(update1)) ──────────────►│
       │                                         │──► HTTP/2 DATA frame
       │──── tx.send(Ok(update2)) ──────────────►│
       │                                         │──► HTTP/2 DATA frame
       │  (tx dropped when loop ends)            │
       │                                     ReceiverStream::next()
       │                                         │  returns None
       │                                         │──► HTTP/2 END_STREAM
```

**การรับ streaming ฝั่ง client:**

```rust
use tokio_stream::StreamExt;

let mut stream = client
    .watch_inventory(WatchInventoryRequest { product_ids: vec![1, 2, 3] })
    .await?
    .into_inner();

// วนรับแต่ละ message จนกว่า stream จะ complete
while let Some(update) = stream.next().await {
    match update {
        Ok(inv) => println!("product {}: stock = {}", inv.product_id, inv.new_stock),
        Err(status) => eprintln!("error: {}", status.message()),
    }
}
```

---

### ขั้นที่ 6: gRPC Client

สร้าง `src/client.rs` ที่เรียกใช้ทุก RPC:

```rust
use grpc_service::{
    ProductServiceClient,
    GetProductRequest, ProductFilter, CreateProductRequest,
    WatchInventoryRequest,
};
use tonic::transport::Channel;
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // สร้าง channel เชื่อมต่อไปยัง server
    // Channel ใช้ HTTP/2 connection pooling ในตัว
    let channel = Channel::from_static("http://[::1]:50051")
        .connect()
        .await?;

    let mut client = ProductServiceClient::new(channel);

    // --- CreateProduct ---
    let resp = client.create_product(CreateProductRequest {
        name:  "Laptop".to_string(),
        price: 999.99,
        stock: 50,
    }).await?;
    let laptop = resp.into_inner();
    println!("Created: id={}, name={}, price={}", laptop.id, laptop.name, laptop.price);

    // --- GetProduct (found) ---
    let resp = client.get_product(GetProductRequest { id: laptop.id }).await?;
    let p = resp.into_inner();
    println!("Got: {:?}", p);

    // --- GetProduct (not found) — ดู error handling ---
    match client.get_product(GetProductRequest { id: 9999 }).await {
        Ok(_) => println!("unexpected success"),
        Err(status) => println!("Expected error: code={:?} msg={}", status.code(), status.message()),
    }

    // --- ListProducts ---
    let resp = client.list_products(ProductFilter {
        min_price:    0.0,
        max_price:    0.0,
        in_stock_only: false,
    }).await?;
    println!("Total products: {}", resp.into_inner().products.len());

    // --- ListProducts with filter ---
    let resp = client.list_products(ProductFilter {
        min_price:    500.0,
        max_price:    1500.0,
        in_stock_only: true,
    }).await?;
    for p in resp.into_inner().products {
        println!("  filtered: {} @ ${:.2} (stock={})", p.name, p.price, p.stock);
    }

    // --- WatchInventory (server streaming) ---
    println!("\n--- Watching inventory ---");
    let mut stream = client
        .watch_inventory(WatchInventoryRequest {
            product_ids: vec![laptop.id],
        })
        .await?
        .into_inner();

    // รับแต่ละ InventoryUpdate จนกว่า stream จะ close
    while let Some(result) = stream.next().await {
        match result {
            Ok(update) => println!(
                "inventory update: product_id={} new_stock={} ({})",
                update.product_id, update.new_stock, update.description
            ),
            Err(e) => eprintln!("stream error: {}", e),
        }
    }

    Ok(())
}
```

**ตั้งค่า Server:**

```rust
// src/server.rs
use grpc_service::{ProductServiceImpl, ProductServiceServer};
use tonic::transport::Server;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let service = ProductServiceImpl::new();

    println!("ProductService gRPC server listening on {}", addr);

    Server::builder()
        .add_service(ProductServiceServer::new(service))
        .serve(addr)
        .await?;

    Ok(())
}
```

รัน server และ client:

```bash
# Terminal 1
cargo run --bin server

# Terminal 2
cargo run --bin client
```

Output ที่คาดหวัง:

```
Created: id=1, name=Laptop, price=999.99
Got: Product { id: 1, name: "Laptop", price: 999.99, stock: 50 }
Expected error: code=NotFound msg=product 9999 not found
Total products: 1
  filtered: Laptop @ $999.99 (stock=50)

--- Watching inventory ---
inventory update: product_id=1 new_stock=50 (initial stock for 'Laptop')
```

---

### ขั้นที่ 7: Interceptors — Authentication และ Logging

Interceptor คือ middleware ที่ทำงานก่อน/หลังทุก RPC call:

#### AuthInterceptor

```rust
use tonic::{Request, Status};

/// ตรวจสอบ Bearer token ใน header "authorization"
#[derive(Clone)]
pub struct AuthInterceptor {
    pub valid_token: String,
}

impl AuthInterceptor {
    pub fn new(token: impl Into<String>) -> Self {
        Self { valid_token: token.into() }
    }

    pub fn intercept(&self, req: Request<()>) -> Result<Request<()>, Status> {
        match req.metadata().get("authorization") {
            Some(val) => {
                let expected = format!("Bearer {}", self.valid_token);
                if val.to_str().unwrap_or("") == expected {
                    Ok(req)  // pass through
                } else {
                    Err(Status::unauthenticated("invalid token"))
                }
            }
            None => Err(Status::unauthenticated("missing authorization header")),
        }
    }
}
```

#### LoggingInterceptor

```rust
use std::sync::{Arc, Mutex};

/// บันทึก method name จาก x-grpc-method metadata header
#[derive(Clone, Default)]
pub struct LoggingInterceptor {
    pub calls: Arc<Mutex<Vec<String>>>,
}

impl LoggingInterceptor {
    pub fn new() -> Self { Self::default() }

    pub fn intercept(&self, req: Request<()>) -> Result<Request<()>, Status> {
        if let Some(path) = req.metadata().get("x-grpc-method") {
            let method = path.to_str().unwrap_or("unknown").to_string();
            self.calls.lock().unwrap().push(method);
        }
        Ok(req)
    }

    pub fn logged_calls(&self) -> Vec<String> {
        self.calls.lock().unwrap().clone()
    }
}
```

#### ใช้ Interceptor กับ tonic Server

tonic รองรับ interceptor ผ่าน `tonic::service::interceptor`:

```rust
use tonic::service::interceptor;
use tonic::transport::Server;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse()?;
    let svc = ProductServiceImpl::new();
    let auth = AuthInterceptor::new("my-secret-token");

    let auth_token = auth.valid_token.clone();

    Server::builder()
        // ห่อ service ด้วย interceptor function
        .add_service(
            ProductServiceServer::with_interceptor(svc, move |req| {
                // clone interceptor เพื่อใช้ใน closure
                AuthInterceptor { valid_token: auth_token.clone() }
                    .intercept(req)
            })
        )
        .serve(addr)
        .await?;

    Ok(())
}
```

**ส่ง Token จาก Client:**

```rust
use tonic::metadata::MetadataValue;

// เพิ่ม authorization header ก่อน call
let token = MetadataValue::from_static("Bearer my-secret-token");

let resp = client
    .get_product({
        let mut req = tonic::Request::new(GetProductRequest { id: 1 });
        req.metadata_mut().insert("authorization", token);
        req
    })
    .await?;
```

---

### ขั้นที่ 8: Error Handling — Domain Error → gRPC Status

ใน production ระบบมักมี domain error ของตัวเอง แยกออกจาก transport error เราต้องแปลงระหว่างกัน:

```rust
/// Domain errors ของระบบ (ไม่รู้จัก gRPC)
#[derive(Debug)]
pub enum DomainError {
    NotFound(String),
    InvalidArgument(String),
    AlreadyExists(String),
    Internal(String),
}

/// แปลง DomainError → tonic::Status อัตโนมัติด้วย From trait
impl From<DomainError> for Status {
    fn from(err: DomainError) -> Self {
        match err {
            DomainError::NotFound(msg)        => Status::not_found(msg),
            DomainError::InvalidArgument(msg) => Status::invalid_argument(msg),
            DomainError::AlreadyExists(msg)   => Status::already_exists(msg),
            DomainError::Internal(msg)        => Status::internal(msg),
        }
    }
}

/// helper function สำหรับแปลงแบบ explicit
pub fn map_domain_error(err: DomainError) -> Status {
    err.into()
}
```

**ใช้งานใน service:**

```rust
fn fetch_from_db(id: u32) -> Result<Product, DomainError> {
    // simulate database call
    if id == 0 {
        Err(DomainError::NotFound(format!("id {} not found", id)))
    } else {
        Ok(Product { id, name: "from db".into(), price: 10.0, stock: 5 })
    }
}

async fn get_product(
    &self,
    request: Request<GetProductRequest>,
) -> Result<Response<Product>, Status> {
    let id = request.into_inner().id;

    // ใช้ ? operator กับ From impl
    let product = fetch_from_db(id).map_err(Status::from)?;
    Ok(Response::new(product))
}
```

**Status::with_details — แนบ metadata เพิ่มเติม:**

```rust
/// สร้าง Status พร้อม custom error details
pub fn status_with_message(code: tonic::Code, message: &str) -> Status {
    Status::new(code, message)
}

/// อ่าน status code จาก Status ที่ได้รับ
pub fn extract_status_code(status: &Status) -> tonic::Code {
    status.code()
}

// ใช้งาน:
let err = status_with_message(
    tonic::Code::InvalidArgument,
    "price field must be a positive number"
);
assert_eq!(extract_status_code(&err), tonic::Code::InvalidArgument);
```

---

## การทดสอบ (Testing)

### Unit Tests ครบ 21 กรณี

โค้ดทดสอบทั้งหมดอยู่ใน `src/lib.rs` ภายใน `#[cfg(test)] mod tests { ... }`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tonic::{Code, metadata::MetadataValue};

    // Helper: สร้าง service instance ใหม่
    fn make_service() -> ProductServiceImpl {
        ProductServiceImpl::new()
    }

    // Helper: ใส่สินค้าเข้า store โดยตรง (bypass service validation)
    fn seed_product(svc: &ProductServiceImpl, name: &str, price: f64, stock: u32) -> u32 {
        let p = Product { id: 0, name: name.to_string(), price, stock };
        svc.store.insert(p).id
    }

    // ========== GetProduct ==========

    #[tokio::test]
    async fn test_get_product_found() {
        let svc = make_service();
        let id = seed_product(&svc, "Widget", 9.99, 10);
        let req = Request::new(GetProductRequest { id });
        let resp = svc.get_product(req).await.unwrap();
        let p = resp.into_inner();
        assert_eq!(p.name, "Widget");
        assert_eq!(p.price, 9.99);
        assert_eq!(p.stock, 10);
    }

    #[tokio::test]
    async fn test_get_product_not_found() {
        let svc = make_service();
        let req = Request::new(GetProductRequest { id: 9999 });
        let err = svc.get_product(req).await.unwrap_err();
        assert_eq!(err.code(), Code::NotFound);
        assert!(err.message().contains("9999"));
    }

    // ========== ListProducts ==========

    #[tokio::test]
    async fn test_list_products_no_filter() {
        let svc = make_service();
        seed_product(&svc, "A", 5.0, 1);
        seed_product(&svc, "B", 15.0, 0);
        let req = Request::new(ProductFilter {
            min_price: 0.0, max_price: 0.0, in_stock_only: false
        });
        let resp = svc.list_products(req).await.unwrap();
        assert_eq!(resp.into_inner().products.len(), 2);
    }

    #[tokio::test]
    async fn test_list_products_price_filter() {
        let svc = make_service();
        seed_product(&svc, "Cheap",     5.0, 1);
        seed_product(&svc, "Mid",      50.0, 1);
        seed_product(&svc, "Expensive",200.0, 1);
        let req = Request::new(ProductFilter {
            min_price: 10.0, max_price: 100.0, in_stock_only: false
        });
        let products = svc.list_products(req).await.unwrap().into_inner().products;
        assert_eq!(products.len(), 1);
        assert_eq!(products[0].name, "Mid");
    }

    #[tokio::test]
    async fn test_list_products_in_stock_only() {
        let svc = make_service();
        seed_product(&svc, "InStock",    10.0, 5);
        seed_product(&svc, "OutOfStock", 10.0, 0);
        let req = Request::new(ProductFilter {
            min_price: 0.0, max_price: 0.0, in_stock_only: true
        });
        let products = svc.list_products(req).await.unwrap().into_inner().products;
        assert_eq!(products.len(), 1);
        assert_eq!(products[0].name, "InStock");
    }

    // ========== CreateProduct ==========

    #[tokio::test]
    async fn test_create_product_success() {
        let svc = make_service();
        let req = Request::new(CreateProductRequest {
            name: "NewGadget".to_string(),
            price: 29.99,
            stock: 100,
        });
        let p = svc.create_product(req).await.unwrap().into_inner();
        assert_eq!(p.name, "NewGadget");
        assert!(p.id > 0);  // store กำหนด ID ให้
    }

    #[tokio::test]
    async fn test_create_product_duplicate_name() {
        let svc = make_service();
        seed_product(&svc, "Duplicate", 10.0, 5);
        let req = Request::new(CreateProductRequest {
            name: "Duplicate".to_string(), price: 20.0, stock: 3,
        });
        let err = svc.create_product(req).await.unwrap_err();
        assert_eq!(err.code(), Code::AlreadyExists);
    }

    #[tokio::test]
    async fn test_create_product_empty_name() {
        let svc = make_service();
        let req = Request::new(CreateProductRequest {
            name: "   ".to_string(), price: 10.0, stock: 1,
        });
        let err = svc.create_product(req).await.unwrap_err();
        assert_eq!(err.code(), Code::InvalidArgument);
    }

    #[tokio::test]
    async fn test_create_product_negative_price() {
        let svc = make_service();
        let req = Request::new(CreateProductRequest {
            name: "Negative".to_string(), price: -5.0, stock: 1,
        });
        let err = svc.create_product(req).await.unwrap_err();
        assert_eq!(err.code(), Code::InvalidArgument);
    }

    // ========== WatchInventory Streaming ==========

    #[tokio::test]
    async fn test_watch_inventory_sends_initial_stock() {
        let svc = make_service();
        let id = seed_product(&svc, "Tracked", 9.0, 42);
        let req = Request::new(WatchInventoryRequest { product_ids: vec![id] });
        let resp = svc.watch_inventory(req).await.unwrap();
        let mut stream = resp.into_inner();

        use tokio_stream::StreamExt;
        let update = stream.next().await.unwrap().unwrap();
        assert_eq!(update.product_id, id);
        assert_eq!(update.new_stock, 42);
    }

    #[tokio::test]
    async fn test_watch_inventory_accumulates_multiple() {
        let svc = make_service();
        let id1 = seed_product(&svc, "P1", 1.0, 10);
        let id2 = seed_product(&svc, "P2", 2.0, 20);
        let req = Request::new(WatchInventoryRequest { product_ids: vec![id1, id2] });

        use tokio_stream::StreamExt;
        let updates: Vec<InventoryUpdate> = svc
            .watch_inventory(req).await.unwrap()
            .into_inner()
            .filter_map(|r| r.ok())
            .collect()
            .await;

        assert_eq!(updates.len(), 2);
        let ids: Vec<u32> = updates.iter().map(|u| u.product_id).collect();
        assert!(ids.contains(&id1));
        assert!(ids.contains(&id2));
    }

    #[tokio::test]
    async fn test_watch_inventory_empty_ids_returns_error() {
        let svc = make_service();
        let req = Request::new(WatchInventoryRequest { product_ids: vec![] });
        let err = svc.watch_inventory(req).await.unwrap_err();
        assert_eq!(err.code(), Code::InvalidArgument);
    }

    // ========== AuthInterceptor ==========

    #[test]
    fn test_auth_interceptor_valid_token() {
        let auth = AuthInterceptor::new("secret-token");
        let mut req = Request::new(());
        req.metadata_mut().insert(
            "authorization",
            MetadataValue::from_static("Bearer secret-token"),
        );
        assert!(auth.intercept(req).is_ok());
    }

    #[test]
    fn test_auth_interceptor_invalid_token() {
        let auth = AuthInterceptor::new("secret-token");
        let mut req = Request::new(());
        req.metadata_mut().insert(
            "authorization",
            MetadataValue::from_static("Bearer wrong-token"),
        );
        let err = auth.intercept(req).unwrap_err();
        assert_eq!(err.code(), Code::Unauthenticated);
    }

    #[test]
    fn test_auth_interceptor_missing_header() {
        let auth = AuthInterceptor::new("secret-token");
        let req = Request::new(());
        let err = auth.intercept(req).unwrap_err();
        assert_eq!(err.code(), Code::Unauthenticated);
    }

    // ========== DomainError Mapping ==========

    #[test]
    fn test_domain_error_not_found_maps_to_status() {
        let status = map_domain_error(DomainError::NotFound("item 42".into()));
        assert_eq!(status.code(), Code::NotFound);
        assert!(status.message().contains("42"));
    }

    #[test]
    fn test_domain_error_invalid_argument_maps_to_status() {
        let status = map_domain_error(DomainError::InvalidArgument("bad field".into()));
        assert_eq!(status.code(), Code::InvalidArgument);
    }

    #[test]
    fn test_domain_error_already_exists_maps_to_status() {
        let status = map_domain_error(DomainError::AlreadyExists("dup".into()));
        assert_eq!(status.code(), Code::AlreadyExists);
    }

    #[test]
    fn test_domain_error_internal_maps_to_status() {
        let status = map_domain_error(DomainError::Internal("db crash".into()));
        assert_eq!(status.code(), Code::Internal);
    }

    #[test]
    fn test_extract_status_code() {
        let s = Status::not_found("missing");
        assert_eq!(extract_status_code(&s), Code::NotFound);
    }

    // ========== LoggingInterceptor ==========

    #[test]
    fn test_logging_interceptor_records_path() {
        let logger = LoggingInterceptor::new();
        let mut req = Request::new(());
        req.metadata_mut().insert(
            "x-grpc-method",
            MetadataValue::from_static("GetProduct"),
        );
        logger.intercept(req).unwrap();
        let calls = logger.logged_calls();
        assert_eq!(calls.len(), 1);
        assert!(calls[0].contains("GetProduct"));
    }
}
```

### ผลลัพธ์จริงจาก `cargo test`

```
running 21 tests
test tests::test_auth_interceptor_missing_header ... ok
test tests::test_auth_interceptor_valid_token ... ok
test tests::test_auth_interceptor_invalid_token ... ok
test tests::test_create_product_empty_name ... ok
test tests::test_create_product_negative_price ... ok
test tests::test_create_product_duplicate_name ... ok
test tests::test_create_product_success ... ok
test tests::test_domain_error_already_exists_maps_to_status ... ok
test tests::test_domain_error_internal_maps_to_status ... ok
test tests::test_domain_error_invalid_argument_maps_to_status ... ok
test tests::test_domain_error_not_found_maps_to_status ... ok
test tests::test_extract_status_code ... ok
test tests::test_get_product_found ... ok
test tests::test_get_product_not_found ... ok
test tests::test_list_products_in_stock_only ... ok
test tests::test_logging_interceptor_records_path ... ok
test tests::test_list_products_price_filter ... ok
test tests::test_list_products_no_filter ... ok
test tests::test_watch_inventory_empty_ids_returns_error ... ok
test tests::test_watch_inventory_sends_initial_stock ... ok
test tests::test_watch_inventory_accumulates_multiple ... ok

test result: ok. 21 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ครอบคลุม 7 กลุ่ม

| กลุ่ม | จำนวน Test | สิ่งที่ทดสอบ |
|-------|------------|-------------|
| GetProduct | 2 | found + not_found |
| ListProducts | 3 | no filter + price filter + in_stock_only |
| CreateProduct | 4 | success + duplicate + empty_name + negative_price |
| WatchInventory | 3 | initial stock + multiple products + empty IDs |
| AuthInterceptor | 3 | valid token + invalid token + missing header |
| DomainError | 5 | not_found + invalid_argument + already_exists + internal + extract code |
| LoggingInterceptor | 1 | records method name |
| **รวม** | **21** | |

---

## กับดักที่ต้องระวัง (Pitfalls)

### กับดัก 1: Field Number ใน Proto ต้องไม่เปลี่ยน

```protobuf
// ❌ อันตราย — เปลี่ยน field number หลัง deploy แล้ว
message Product {
    uint32 id    = 1;
    string name  = 3;  // เคยเป็น 2, ย้ายไป 3
    double price = 2;  // เคยเป็น 3, ย้ายไป 2 ← wire format พัง!
}

// ✅ ถูกต้อง — เพิ่ม field ใหม่, อย่าเปลี่ยน field number เดิม
message Product {
    uint32 id       = 1;
    string name     = 2;
    double price    = 3;
    uint32 stock    = 4;
    string category = 5;  // field ใหม่ — ไม่กระทบ backward compatibility
}
```

กฎ: **field number คือ ID ถาวร** client เก่าที่ไม่รู้จัก field 5 จะ ignore มันไป แต่ถ้าเปลี่ยน field number จะ decode ข้อมูลผิดทันที

### กับดัก 2: `#[tonic::async_trait]` ห้ามลืม

```rust
// ❌ compile error: "async fn in trait is not yet stable"
impl ProductService for ProductServiceImpl {
    async fn get_product(...) -> ... { ... }
}

// ✅ ต้องใส่ #[tonic::async_trait]
#[tonic::async_trait]
impl ProductService for ProductServiceImpl {
    async fn get_product(...) -> ... { ... }
}
```

`tonic::async_trait` เป็น re-export ของ `async_trait::async_trait` ซึ่งแปลง async fn ใน trait เป็น `Box<dyn Future>` เพื่อให้ compiler รับได้

### กับดัก 3: DashMap Reference ต้อง Clone ก่อน Return

```rust
// ❌ compile error: reference ไม่สามารถ cross async boundary ได้
pub fn get_bad(&self, id: u32) -> Option<&Product> {
    self.inner.get(&id).map(|r| r.value())
    //                           ^^^^^^^^^^
    // Ref<'_, u32, Product> ถือ shard lock อยู่
    // ไม่สามารถ return ได้ เพราะ lifetime กับ lock ผูกกัน
}

// ✅ clone ออกก่อน drop lock
pub fn get(&self, id: u32) -> Option<Product> {
    self.inner.get(&id).map(|r| r.clone())
    //                        ^^^^^^^^^^^
    // clone Product ออกมา แล้ว Ref จะ drop → lock release
}
```

### กับดัก 4: `ReceiverStream` จะ complete เมื่อ tx ถูก drop

```rust
// ❌ ปัญหา: ลืม drop tx หลัง loop
tokio::spawn(async move {
    for id in ids {
        tx.send(Ok(update)).await.ok();
    }
    // tx ยังคงอยู่ใน scope → stream ไม่ complete!
    // client จะรอ message ต่อไปเรื่อย ๆ
    tokio::time::sleep(Duration::MAX).await;  // ← ไม่ควรทำ
});

// ✅ ถูกต้อง: tx ถูก move เข้า closure, drop เมื่อ closure จบ
tokio::spawn(async move {
    for id in ids {
        if tx.send(Ok(update)).await.is_err() {
            break;  // client disconnect แล้ว — หยุด
        }
    }
    // closure จบ → tx drop → rx ได้รับ None → stream complete
});
```

### กับดัก 5: Interceptor ใน tonic ต้องเป็น `Fn` ไม่ใช่ `FnOnce`

```rust
// ❌ ปัญหา: interceptor ถูกเรียกหลายครั้ง
// ถ้า capture value ด้วย move แต่ไม่ Copy ได้ จะ compile error
let token = "secret".to_string();
ProductServiceServer::with_interceptor(svc, move |req| {
    drop(token);  // ← error: cannot move out of `token` in FnMut context
    Ok(req)
});

// ✅ ถ้า interceptor ต้องการ state ให้ใช้ Clone หรือ Arc
let token = Arc::new("secret".to_string());
let token2 = token.clone();
ProductServiceServer::with_interceptor(svc, move |req| {
    println!("checking token: {}", token2);  // ← ใช้ reference ได้
    Ok(req)
});
```

### กับดัก 6: `protoc` ต้องติดตั้งในระบบ

`tonic-build` เรียกใช้ `protoc` binary ต้องมีอยู่ใน `$PATH`:

```bash
# Ubuntu/Debian
sudo apt-get install -y protobuf-compiler

# macOS
brew install protobuf

# ตรวจสอบ
protoc --version
# libprotoc 3.21.12
```

ถ้าไม่มี `protoc` จะ error ตอน `cargo build`:

```
error: failed to run custom build command for `grpc-service`
...
protoc: No such file or directory (os error 2)
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build ทั้ง server และ client แบบ optimized
cargo build --release

# binaries อยู่ที่:
# target/release/server
# target/release/client
```

### Docker Image

```dockerfile
# ---- Build stage ----
FROM rust:1.80-slim AS builder

# ติดตั้ง protoc สำหรับ tonic-build
RUN apt-get update && apt-get install -y protobuf-compiler

WORKDIR /app
COPY . .
RUN cargo build --release --bin server

# ---- Runtime stage ----
FROM debian:bookworm-slim

WORKDIR /app
COPY --from=builder /app/target/release/server .

EXPOSE 50051

CMD ["./server"]
```

```bash
docker build -t grpc-service:latest .
docker run -p 50051:50051 grpc-service:latest
```

### Health Check ด้วย gRPC Health Protocol

gRPC มี [standard health check protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md) — Kubernetes ใช้ตรวจ pod:

```toml
# Cargo.toml
tonic-health = "0.12"
```

```rust
use tonic_health::server::health_reporter;

let (mut health_reporter, health_svc) = health_reporter();
health_reporter
    .set_serving::<ProductServiceServer<ProductServiceImpl>>()
    .await;

Server::builder()
    .add_service(health_svc)
    .add_service(ProductServiceServer::new(svc))
    .serve(addr)
    .await?;
```

```yaml
# kubernetes readiness probe
readinessProbe:
  grpc:
    port: 50051
  initialDelaySeconds: 5
  periodSeconds: 10
```

### gRPC Reflection สำหรับ Debug

```toml
tonic-reflection = "0.12"
```

```rust
use tonic_reflection::server::Builder as ReflectionBuilder;

// register .proto file descriptor
let reflection_svc = ReflectionBuilder::configure()
    .register_encoded_file_descriptor_set(
        tonic::include_file_descriptor_set!("product_descriptor")
    )
    .build_v1()?;

Server::builder()
    .add_service(reflection_svc)
    .add_service(ProductServiceServer::new(svc))
    .serve(addr)
    .await?;
```

เมื่อ enable reflection แล้ว สามารถใช้ tool อย่าง `grpcurl` inspect service ได้:

```bash
# list ทุก service
grpcurl -plaintext localhost:50051 list

# inspect ProductService
grpcurl -plaintext localhost:50051 describe product.ProductService

# call GetProduct
grpcurl -plaintext -d '{"id": 1}' localhost:50051 product.ProductService/GetProduct
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัด 1: เพิ่ม UpdateProduct และ DeleteProduct RPC

ขยาย `.proto` และ implement:

```protobuf
service ProductService {
    // ... existing RPCs ...
    rpc UpdateProduct(UpdateProductRequest) returns (Product);
    rpc DeleteProduct(DeleteProductRequest) returns (DeleteProductResponse);
}

message UpdateProductRequest {
    uint32 id    = 1;
    string name  = 2;  // empty = ไม่เปลี่ยน
    double price = 3;  // 0.0 = ไม่เปลี่ยน
    uint32 stock = 4;  // max_value = ไม่เปลี่ยน
}

message DeleteProductRequest {
    uint32 id = 1;
}

message DeleteProductResponse {
    bool deleted = 1;
}
```

ความท้าทาย: handle partial update — ถ้า field ใน Protobuf เป็น 0 หมายถึง "ไม่เปลี่ยน" หรือ "ตั้งค่าเป็น 0"? แนะนำให้ใช้ `optional` field หรือ `google.protobuf.FieldMask`

### แบบฝึกหัด 2: Bidirectional Streaming สำหรับ Live Chat ระหว่าง Service

```protobuf
// chat ระหว่าง services
rpc StreamUpdates(stream StockUpdate) returns (stream StockUpdate);
```

implement ด้วย `tokio::sync::broadcast::channel` เพื่อให้หลาย subscriber รับ event เดียวกัน

### แบบฝึกหัด 3: Persistent Store ด้วย SQLite

แทนที่ `DashMap` ด้วย `sqlx::SqlitePool`:

```rust
pub struct ProductStore {
    pool: sqlx::SqlitePool,
}

impl ProductStore {
    pub async fn get(&self, id: u32) -> Result<Option<Product>, sqlx::Error> {
        sqlx::query_as!(Product, "SELECT * FROM products WHERE id = ?", id)
            .fetch_optional(&self.pool)
            .await
    }
}
```

ทดสอบด้วย `sqlx::test` ที่ run isolated database ต่อ test case

### แบบฝึกหัด 4: Rate Limiting Interceptor

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};

pub struct RateLimitInterceptor {
    // client IP → (count, window_start)
    counts: Arc<Mutex<HashMap<String, (u32, Instant)>>>,
    max_per_minute: u32,
}

impl RateLimitInterceptor {
    pub fn intercept(&self, req: Request<()>) -> Result<Request<()>, Status> {
        // อ่าน x-forwarded-for หรือ x-real-ip
        // ตรวจสอบว่าเกิน max_per_minute หรือไม่
        // ถ้าเกิน return Err(Status::resource_exhausted("rate limit exceeded"))
        todo!()
    }
}
```

### แบบฝึกหัด 5: TLS สำหรับ Production

```rust
use tonic::transport::{Certificate, Identity, ServerTlsConfig};

// server
Server::builder()
    .tls_config(
        ServerTlsConfig::new()
            .identity(Identity::from_pem(&cert_pem, &key_pem))
            .client_ca_root(Certificate::from_pem(&ca_pem))
    )?
    .add_service(ProductServiceServer::new(svc))
    .serve(addr)
    .await?;

// client
let tls = ClientTlsConfig::new()
    .ca_certificate(Certificate::from_pem(&ca_pem))
    .domain_name("product.example.com");

let channel = Channel::from_static("https://product.example.com:50051")
    .tls_config(tls)?
    .connect()
    .await?;
```

สร้าง self-signed certificates ด้วย `openssl` สำหรับ local testing แล้วทดสอบ mTLS (mutual TLS) ที่ทั้ง client และ server ยืนยัน certificate ซึ่งกันและกัน

### แบบฝึกหัด 6: gRPC Gateway — expose เป็น REST API ด้วย

ใช้ `tonic-web` เพื่อให้ browser ที่ไม่รองรับ HTTP/2 gRPC เข้าถึงได้ผ่าน `grpc-web` protocol:

```toml
tonic-web = "0.12"
```

```rust
Server::builder()
    .accept_http1(true)  // รับ HTTP/1.1 สำหรับ grpc-web
    .layer(tonic_web::GrpcWebLayer::new())
    .add_service(ProductServiceServer::new(svc))
    .serve(addr)
    .await?;
```

---

## สรุป

โปรเจคนี้สร้าง gRPC service ครบฟีเจอร์ด้วย **tonic** ซึ่งเป็น framework gRPC ระดับ production สำหรับ Rust ที่ได้รับความนิยมสูงสุด สิ่งสำคัญที่ได้เรียนรู้:

**Proto-First Design:** กำหนด `.proto` schema ก่อน แล้วให้ `tonic-build` generate code อัตโนมัติ — ทำให้ client/server ไม่สามารถ mismatch ได้เลยในระดับ compile time

**Server Streaming Pattern:** `mpsc::channel` + `ReceiverStream` เป็น idiom มาตรฐานของ tonic — background task ส่งข้อมูล, stream จะ complete อัตโนมัติเมื่อ `tx` ถูก drop

**gRPC Status Codes vs HTTP Status Codes:** gRPC มี status codes ของตัวเองที่ละเอียดกว่า HTTP เช่น `NOT_FOUND`, `ALREADY_EXISTS`, `UNAUTHENTICATED`, `RESOURCE_EXHAUSTED` — ต้อง map ให้ถูกต้องเพื่อให้ client จัดการ error ได้

**From Trait สำหรับ Error Conversion:** `impl From<DomainError> for Status` ทำให้เขียน `.map_err(Status::from)?` ได้ไหลลื่น โดยไม่ต้อง match error เอง

**DashMap สำหรับ Concurrent State:** ใน async gRPC service ที่มี many concurrent requests, `DashMap` ให้ performance สูงกว่า `Mutex<HashMap>` มากสำหรับ read-heavy workload

โปรเจคถัดไป **H08: TCP Chat Server** จะนำ skill ด้าน async networking มาสร้าง real-time chat system ที่จัดการ TCP connections หลายพันตัวพร้อมกัน — ใช้ broadcast channel และ session management ที่ซับซ้อนกว่า

---

**โปรเจคก่อนหน้า:** [Project H06: Protobuf Codec](project-h06-protobuf-codec.md) | **โปรเจคถัดไป:** [Project H08: TCP Chat Server](project-h08-tcp-chat.md)
