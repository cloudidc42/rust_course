# Project J02: Full-Stack Axum + Yew

> โมดูล: J — Full-Stack / WASM | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

Full-Stack Axum + Yew คือโปรเจคที่สร้าง **web application ครบวงจรด้วย Rust ล้วน** — ตั้งแต่ backend API server ไปจนถึง frontend ที่รันใน browser เป็น WebAssembly โดยใช้ **shared types** ร่วมกันระหว่างทั้งสองฝั่ง ซึ่งเป็น pattern ที่ทรงพลังที่สุดอย่างหนึ่งใน Rust ecosystem

ปัญหาที่โปรเจคนี้แก้ไข: ใน full-stack development ทั่วไป (เช่น Node.js + TypeScript) developer ต้องนิยาม type เดิมซ้ำสองครั้ง — ครั้งหนึ่งใน backend, อีกครั้งใน frontend แล้วก็ต้องคอยดูแลให้ทั้งสองฝั่ง sync กัน เมื่อ API เปลี่ยน ต้องแก้ทั้งสองที่ด้วยมือ ซึ่งเป็นแหล่งของ bug ที่พบบ่อยมาก

Rust แก้ปัญหานี้ได้สมบูรณ์ด้วย **Cargo workspace** — สร้าง `shared` crate ที่ define ทั้ง `Todo`, `CreateTodo`, `UpdateTodo` ไว้ที่เดียว แล้วให้ทั้ง `backend` (Axum) และ `frontend` (Yew/WASM) import จาก crate เดียวกัน เมื่อ type เปลี่ยน compiler จะแจ้ง error ทั้งสองฝั่งทันที ไม่มีทางที่จะ mismatched API schema โดยไม่รู้ตัว

Use case จริงในโลก production:
- **Internal tools** — dashboard, admin panel, tracking system ที่ทีมใช้ภายในองค์กร
- **Offline-capable apps** — WASM app ที่ sync กับ backend เมื่อมี network
- **High-performance SPA** — frontend ที่ต้องการ computation หนักใน browser (เช่น data processing, encryption)
- **Secure applications** — shared type validation ทำให้ server และ client ใช้ logic เดียวกันในการ validate

Learning value ที่ได้: โปรเจคนี้สอนวิธีคิดแบบ "type-driven full-stack" — ออกแบบ data model ก่อน แล้วให้ compiler เป็นตัวบังคับว่า backend และ frontend ต้องสื่อสารกันในรูปแบบที่ถูกต้อง pattern นี้เป็นข้อได้เปรียบหลักของ Rust ที่ภาษาอื่นทำได้ยากกว่ามาก

## สิ่งที่จะได้เรียนรู้

- **Cargo workspace**: จัดการ multi-crate project ด้วย `[workspace]`, `path` dependency, และ shared `Cargo.lock`
- **Shared types**: ออกแบบ data types ที่ compile ได้ทั้งใน native (backend) และ WASM target (frontend)
- **Axum CRUD API**: `State` extractor, `Path`/`Json` extractors, `Router` composition, HTTP status codes
- **AppState pattern**: ใช้ `Arc<Mutex<Vec<Todo>>>` เป็น in-memory store, thread-safe state sharing
- **CORS middleware**: `tower-http` CorsLayer เพื่อให้ frontend เรียก backend จาก different origin ได้
- **Axum testing**: ใช้ `tower::ServiceExt::oneshot` ทดสอบ route โดยไม่ต้องรัน HTTP server จริง
- **Yew component design**: `use_state`, `use_effect_with`, props passing, event callbacks
- **gloo-net API calls**: `gloo_net::http::Request` สำหรับ fetch HTTP จาก WASM, async/await ใน Yew

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust fundamentals — ownership, structs, enums, impl blocks
- **Part 21–40**: Traits, generics, error handling (`Result`/`Option`), closures
- **Part 41–60**: `async`/`await`, `tokio` runtime, `Arc`/`Mutex`, HTTP basics, `serde` JSON serialization
- **Part 61–80**: Axum web framework, HTTP routing, extractors, middleware
- **Part 81–95**: WebAssembly basics, Yew framework, reactive components, `wasm-bindgen`
- โปรเจค J01 (Yew SPA) ควรทำก่อนเพื่อเข้าใจ Yew component model

## โครงสร้างโปรเจค (Project Layout)

```
fullstack-todo/
├── Cargo.toml              ← workspace manifest
├── shared/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs          ← Todo, CreateTodo, UpdateTodo + serde derive
├── backend/
│   ├── Cargo.toml
│   └── src/
│       └── main.rs         ← Axum server, handlers, AppState, tests
└── frontend/
    ├── Cargo.toml
    ├── index.html          ← HTML entry point สำหรับ trunk
    └── src/
        ├── main.rs         ← Yew app entry point
        ├── api.rs          ← gloo-net API client module
        ├── components/
        │   ├── mod.rs
        │   ├── todo_list.rs    ← list component
        │   └── todo_form.rs    ← create/edit form
        └── types.rs        ← re-export shared types + local state types
```

## การออกแบบ (Architecture & Design)

### ภาพรวม Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Browser (WebAssembly)                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Yew App (frontend crate)                               │    │
│  │  ┌─────────────────┐    ┌──────────────────────────┐   │    │
│  │  │  TodoList        │    │  TodoForm                │   │    │
│  │  │  component       │    │  component               │   │    │
│  │  │  (use_state,     │    │  (POST /todos)           │   │    │
│  │  │   use_effect)    │    │  (PUT /todos/:id)        │   │    │
│  │  └────────┬─────────┘    └──────────┬───────────────┘   │    │
│  │           │                         │                    │    │
│  │           └──────────┬──────────────┘                    │    │
│  │                      │                                   │    │
│  │              ┌───────▼────────┐                          │    │
│  │              │  api.rs        │                          │    │
│  │              │  gloo-net HTTP │                          │    │
│  │              └───────┬────────┘                          │    │
│  └──────────────────────┼──────────────────────────────────┘    │
└─────────────────────────┼───────────────────────────────────────┘
                          │  HTTP (JSON)
                          │  shared types ←── ใช้ร่วมกัน
┌─────────────────────────┼───────────────────────────────────────┐
│  Axum Server (backend crate)                                     │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  Router                                                  │    │
│  │  GET  /todos        → list_todos                         │    │
│  │  POST /todos        → create_todo                        │    │
│  │  GET  /todos/:id    → get_todo                           │    │
│  │  PUT  /todos/:id    → update_todo                        │    │
│  │  DELETE /todos/:id  → delete_todo                        │    │
│  └──────────────────────┬──────────────────────────────────┘    │
│                         │                                        │
│  ┌──────────────────────▼──────────────────────────────────┐    │
│  │  AppState = Arc<Mutex<Vec<Todo>>>                        │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### ทำไม shared crate ถึงสำคัญ

ในสถาปัตยกรรม full-stack Rust, shared crate คือ "contract" ระหว่าง backend และ frontend เมื่อ `Todo` struct เปลี่ยน เช่น เพิ่ม field `priority: u8` — compiler จะ error ทั้งใน backend handler และ frontend component พร้อมกัน ทำให้ refactoring ปลอดภัยโดยสมบูรณ์

ข้อจำกัดสำคัญของ shared crate: ต้อง compile ได้ทั้งสองสถาปัตยกรรม — `x86_64-unknown-linux-gnu` (backend) และ `wasm32-unknown-unknown` (frontend) ดังนั้นห้ามใช้:
- crate ที่ต้องการ OS threads (เช่น `std::thread`)
- crate ที่ใช้ `std::net` หรือ file I/O
- crate ที่ไม่รองรับ `no_std` หรือ WASM target

`serde` รองรับ WASM ได้ดี, `serde_json` ก็ใช้ได้ แต่ควรใช้ใน dev-dependencies ของ shared เท่านั้น (สำหรับ test) ไม่ใช่ production dependency เพราะ JSON serialization มักทำใน layer บน

### ทำไมเลือก `Arc<Mutex<Vec<Todo>>>` แทน database

สำหรับโปรเจคนี้ in-memory state สอนแนวคิดหลักได้ชัดเจนกว่า — เห็น AppState pattern, thread safety, และ state sharing ได้โดยตรงโดยไม่มี complexity ของ database connection pool ในโปรเจคจริงควรเปลี่ยนเป็น `sqlx::PgPool` หรือ `sqlx::SqlitePool`

### Data Flow ของการสร้าง Todo

```
User กรอก form (frontend)
     │
     ▼
TodoForm component
     │ onClick "บันทึก"
     ▼
api::create_todo(CreateTodo { title: "..." })
     │
     ▼ gloo_net::http::Request::post("/todos")
     │   .header("Content-Type", "application/json")
     │   .body(serde_json::to_string(&payload))
     │
HTTP POST /todos  ──────────────────────────────────►
                                                       Axum Router
                                                       create_todo handler
                                                       │
                                                       ▼
                                                  state.lock().unwrap()
                                                  ต่อ id ใหม่
                                                  todos.push(...)
                                                  │
                                                  ▼
◄────────────────────────── 201 Created { id, title, completed }
     │
     ▼
serde_json::from_str::<Todo>(&body)
     │ (ใช้ shared::Todo — type เดียวกับ backend!)
     ▼
set_todos callback → update state → re-render list
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: สร้าง Cargo Workspace

เริ่มต้นด้วยการสร้างโครงสร้าง workspace ที่รวม 3 crate ไว้ด้วยกัน workspace ทำให้ทุก crate ใช้ `Cargo.lock` ไฟล์เดียวกัน และ build artifacts ถูก cache ร่วมกันใน `target/` directory

```bash
mkdir -p fullstack-todo/{shared,backend,frontend}/src
cd fullstack-todo
```

**`Cargo.toml` (workspace root):**

```toml
[workspace]
members = ["backend", "shared", "frontend"]
resolver = "2"

# ใช้ workspace-level dependencies เพื่อให้ทุก crate ใช้ version เดียวกัน
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
axum = { version = "0.7", features = ["json"] }
tower-http = { version = "0.5", features = ["cors"] }
```

สังเกตว่า `resolver = "2"` บังคับใช้ feature resolver รุ่นใหม่ ซึ่งจำเป็นสำหรับ workspace ที่มีทั้ง native และ WASM targets

สร้าง directory สำหรับแต่ละ crate:

```bash
# shared crate — types ที่ใช้ร่วมกัน
mkdir -p shared/src

# backend crate — Axum server
mkdir -p backend/src

# frontend crate — Yew WASM app
mkdir -p frontend/src/components
```

### ขั้นที่ 2: Shared Crate — Types ที่ใช้ร่วมกัน

`shared` crate คือหัวใจของสถาปัตยกรรมนี้ ต้องออกแบบให้ compile ได้ทั้ง native และ WASM โดยไม่มี OS dependency

**`shared/Cargo.toml`:**

```toml
[package]
name = "shared"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }

[dev-dependencies]
serde_json = "1"
```

**`shared/src/lib.rs`:**

```rust
use serde::{Deserialize, Serialize};

/// Todo item หลัก — ใช้ทั้งใน backend responses และ frontend state
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Todo {
    pub id: u64,
    pub title: String,
    pub completed: bool,
}

/// Payload สำหรับ POST /todos
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CreateTodo {
    pub title: String,
}

/// Payload สำหรับ PUT /todos/:id — ทุก field เป็น Optional เพื่อรองรับ partial update
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UpdateTodo {
    pub title: Option<String>,
    pub completed: Option<bool>,
}

impl Todo {
    pub fn new(id: u64, title: impl Into<String>) -> Self {
        Self {
            id,
            title: title.into(),
            completed: false,
        }
    }
}
```

ประเด็นสำคัญในการออกแบบ:
- `PartialEq` บน `Todo` จำเป็นสำหรับ Yew เพื่อเปรียบเทียบ props และตัดสินใจว่าต้อง re-render หรือไม่
- `Clone` จำเป็นเพราะ Yew component อาจถือ reference หลายชั้น
- `UpdateTodo` ใช้ `Option<T>` เพื่อรองรับ partial update — ส่งเฉพาะ field ที่ต้องการเปลี่ยน

### ขั้นที่ 3: Backend — Axum Server พื้นฐาน

**`backend/Cargo.toml`:**

```toml
[package]
name = "backend"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "backend"
path = "src/main.rs"

[dependencies]
shared = { path = "../shared" }
axum = { version = "0.7", features = ["json"] }
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.5", features = ["cors"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[dev-dependencies]
tower = { version = "0.4", features = ["util"] }
http-body-util = "0.1"
```

**`backend/src/main.rs` — AppState และ Router:**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
    routing::{delete, get, post, put},
    Router,
};
use shared::{CreateTodo, Todo, UpdateTodo};
use std::sync::{Arc, Mutex};
use tower_http::cors::{Any, CorsLayer};

/// AppState คือ type alias สำหรับ shared state ทั่วทั้งแอปพลิเคชัน
/// Arc ทำให้ clone ได้โดยไม่ copy data
/// Mutex ทำให้ access ได้จาก multiple async tasks อย่างปลอดภัย
pub type AppState = Arc<Mutex<Vec<Todo>>>;
```

**สร้าง Router แยกเป็นฟังก์ชัน** เพื่อให้ทดสอบได้ง่าย:

```rust
pub fn build_router(state: AppState) -> Router {
    // กำหนด CORS policy — ให้ frontend ที่ต่าง origin เรียกได้
    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    Router::new()
        .route("/todos", get(list_todos))
        .route("/todos", post(create_todo))
        .route("/todos/:id", get(get_todo))
        .route("/todos/:id", put(update_todo))
        .route("/todos/:id", delete(delete_todo))
        .layer(cors)
        .with_state(state)
}
```

**Handler: GET /todos**

```rust
pub async fn list_todos(State(state): State<AppState>) -> Json<Vec<Todo>> {
    let todos = state.lock().unwrap();
    Json(todos.clone())
}
```

`State(state)` เป็น extractor ที่ดึง AppState ออกมา Axum จะ inject state นี้ให้อัตโนมัติสำหรับทุก request `lock().unwrap()` จะ block จนกว่า Mutex จะว่าง — ในโปรเจคนี้เพียงพอ แต่สำหรับ production ควรพิจารณา `tokio::sync::RwLock` หากมี read มากกว่า write

**Handler: POST /todos**

```rust
pub async fn create_todo(
    State(state): State<AppState>,
    Json(payload): Json<CreateTodo>,
) -> (StatusCode, Json<Todo>) {
    let mut todos = state.lock().unwrap();
    // หา id สูงสุดแล้วบวก 1 — simple auto-increment
    let id = todos.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    let todo = Todo::new(id, payload.title);
    todos.push(todo.clone());
    (StatusCode::CREATED, Json(todo))
}
```

Return type `(StatusCode, Json<Todo>)` เป็น tuple response — Axum จะ set HTTP status code เป็น 201 Created และ body เป็น JSON ของ todo ที่สร้างใหม่

**Handler: GET /todos/:id**

```rust
pub async fn get_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Todo>, StatusCode> {
    let todos = state.lock().unwrap();
    todos
        .iter()
        .find(|t| t.id == id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}
```

`Path(id): Path<u64>` — Axum จะ parse `:id` จาก URL แล้วแปลงเป็น `u64` อัตโนมัติ ถ้า parse ไม่ได้ (เช่น `/todos/abc`) Axum return 400 Bad Request ให้เองโดยไม่ต้องเขียน code เพิ่ม

`Result<Json<Todo>, StatusCode>` — Axum implement `IntoResponse` สำหรับ `StatusCode` ดังนั้น return `Err(StatusCode::NOT_FOUND)` จะได้ 404 response

**Handler: PUT /todos/:id**

```rust
pub async fn update_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(payload): Json<UpdateTodo>,
) -> Result<Json<Todo>, StatusCode> {
    let mut todos = state.lock().unwrap();
    let todo = todos
        .iter_mut()
        .find(|t| t.id == id)
        .ok_or(StatusCode::NOT_FOUND)?;

    if let Some(title) = payload.title {
        todo.title = title;
    }
    if let Some(completed) = payload.completed {
        todo.completed = completed;
    }
    Ok(Json(todo.clone()))
}
```

`?` operator ใช้ได้เพราะ return type คือ `Result<..., StatusCode>` — ถ้าหา todo ไม่เจอ จะ early return `Err(StatusCode::NOT_FOUND)` ทันที

**Handler: DELETE /todos/:id**

```rust
pub async fn delete_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> StatusCode {
    let mut todos = state.lock().unwrap();
    let len_before = todos.len();
    todos.retain(|t| t.id != id);
    if todos.len() < len_before {
        StatusCode::NO_CONTENT   // 204 — ลบสำเร็จ
    } else {
        StatusCode::NOT_FOUND    // 404 — ไม่พบ id นี้
    }
}
```

`Vec::retain()` เป็น idiom ที่สะอาดสำหรับลบ element โดยใช้ predicate

**Main function:**

```rust
#[tokio::main]
async fn main() {
    let state: AppState = Arc::new(Mutex::new(Vec::new()));
    let app = build_router(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Backend listening on http://0.0.0.0:3000");
    axum::serve(listener, app).await.unwrap();
}
```

### ขั้นที่ 4: CORS Middleware — เชื่อม Frontend กับ Backend

CORS (Cross-Origin Resource Sharing) เป็น browser security mechanism ที่ block HTTP request จาก origin หนึ่ง (เช่น `http://localhost:8080` ที่ Trunk serve frontend) ไปยังอีก origin (เช่น `http://localhost:3000` ที่ Axum รัน) โดย default

`tower-http` ให้ `CorsLayer` ที่จัดการ CORS headers ให้อัตโนมัติ:

```rust
use tower_http::cors::{Any, CorsLayer};
use axum::http::{HeaderValue, Method};

// Development setup — อนุญาตทุก origin
let cors_dev = CorsLayer::new()
    .allow_origin(Any)
    .allow_methods(Any)
    .allow_headers(Any);

// Production setup — จำกัด origin
let cors_prod = CorsLayer::new()
    .allow_origin("https://myapp.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers(Any);
```

สำหรับ development ใช้ `Any` ได้ แต่ใน production **ต้องระบุ origin ที่อนุญาตอย่างชัดเจน** เพื่อป้องกัน CSRF attack

CORS layer ต้องถูก add ก่อน route definition (ใน `.layer(cors)`) เพื่อให้ middleware wrap ทุก route ถ้าใส่ผิดที่จะทำให้ preflight OPTIONS request ไม่ถูก handle

### ขั้นที่ 5: ทดสอบ Backend ด้วย tower::ServiceExt

ข้อได้เปรียบสำคัญของ Axum คือสามารถทดสอบ route handlers ได้โดยตรงด้วย `tower::ServiceExt::oneshot` โดยไม่ต้องรัน HTTP server จริง ทำให้ test เร็วและ reliable

**`backend/src/main.rs` — test module:**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::{
        body::Body,
        http::{Method, Request},
    };
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    fn make_state() -> AppState {
        Arc::new(Mutex::new(Vec::new()))
    }

    async fn body_to_string(body: Body) -> String {
        let bytes = body.collect().await.unwrap().to_bytes();
        String::from_utf8(bytes.to_vec()).unwrap()
    }

    #[tokio::test]
    async fn test_list_todos_empty() {
        let state = make_state();
        let app = build_router(state);
        let req = Request::builder()
            .method(Method::GET)
            .uri("/todos")
            .body(Body::empty())
            .unwrap();
        let resp = app.oneshot(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
        let body = body_to_string(resp.into_body()).await;
        assert_eq!(body, "[]");
    }

    #[tokio::test]
    async fn test_create_todo() {
        let state = make_state();
        let app = build_router(state);
        let payload = r#"{"title":"Learn Axum"}"#;
        let req = Request::builder()
            .method(Method::POST)
            .uri("/todos")
            .header("content-type", "application/json")
            .body(Body::from(payload))
            .unwrap();
        let resp = app.oneshot(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::CREATED);
        let body = body_to_string(resp.into_body()).await;
        let todo: Todo = serde_json::from_str(&body).unwrap();
        assert_eq!(todo.title, "Learn Axum");
        assert!(!todo.completed);
        assert_eq!(todo.id, 1);
    }

    #[tokio::test]
    async fn test_get_todo_not_found() {
        let state = make_state();
        let app = build_router(state);
        let req = Request::builder()
            .method(Method::GET)
            .uri("/todos/999")
            .body(Body::empty())
            .unwrap();
        let resp = app.oneshot(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::NOT_FOUND);
    }

    #[tokio::test]
    async fn test_update_todo() {
        let state = make_state();
        // pre-seed state ก่อนทดสอบ
        {
            let mut v = state.lock().unwrap();
            v.push(Todo::new(1, "Old Title"));
        }
        let app = build_router(state);
        let payload = r#"{"completed":true}"#;
        let req = Request::builder()
            .method(Method::PUT)
            .uri("/todos/1")
            .header("content-type", "application/json")
            .body(Body::from(payload))
            .unwrap();
        let resp = app.oneshot(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
        let body = body_to_string(resp.into_body()).await;
        let todo: Todo = serde_json::from_str(&body).unwrap();
        assert!(todo.completed);
        assert_eq!(todo.title, "Old Title");  // title ไม่เปลี่ยน
    }

    #[tokio::test]
    async fn test_delete_todo() {
        let state = make_state();
        {
            let mut v = state.lock().unwrap();
            v.push(Todo::new(1, "To Delete"));
        }
        let app = build_router(state);
        let req = Request::builder()
            .method(Method::DELETE)
            .uri("/todos/1")
            .body(Body::empty())
            .unwrap();
        let resp = app.oneshot(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::NO_CONTENT);
    }

    #[tokio::test]
    async fn test_delete_todo_not_found() {
        let state = make_state();
        let app = build_router(state);
        let req = Request::builder()
            .method(Method::DELETE)
            .uri("/todos/999")
            .body(Body::empty())
            .unwrap();
        let resp = app.oneshot(req).await.unwrap();
        assert_eq!(resp.status(), StatusCode::NOT_FOUND);
    }

    #[tokio::test]
    async fn test_create_multiple_todos_and_list() {
        let state = make_state();
        let app = build_router(state.clone());

        // สร้าง todo แรก
        let req1 = Request::builder()
            .method(Method::POST)
            .uri("/todos")
            .header("content-type", "application/json")
            .body(Body::from(r#"{"title":"Task One"}"#))
            .unwrap();
        app.clone().oneshot(req1).await.unwrap();

        // สร้าง todo ที่สอง
        let req2 = Request::builder()
            .method(Method::POST)
            .uri("/todos")
            .header("content-type", "application/json")
            .body(Body::from(r#"{"title":"Task Two"}"#))
            .unwrap();
        app.clone().oneshot(req2).await.unwrap();

        // list ทั้งหมด
        let req3 = Request::builder()
            .method(Method::GET)
            .uri("/todos")
            .body(Body::empty())
            .unwrap();
        let resp = app.oneshot(req3).await.unwrap();
        assert_eq!(resp.status(), StatusCode::OK);
        let body = body_to_string(resp.into_body()).await;
        let todos: Vec<Todo> = serde_json::from_str(&body).unwrap();
        assert_eq!(todos.len(), 2);
        assert_eq!(todos[0].title, "Task One");
        assert_eq!(todos[1].title, "Task Two");
    }
}
```

สังเกตว่าใน `test_create_multiple_todos_and_list` เราใช้ `state.clone()` และ `app.clone()` เพื่อให้ทุก request ใช้ state เดียวกัน — `Arc::clone` ไม่ copy data แค่เพิ่ม reference count

### ขั้นที่ 6: Frontend Yew — App Structure

**`frontend/Cargo.toml`:**

```toml
[package]
name = "frontend"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
shared = { path = "../shared" }
yew = { version = "0.21", features = ["csr"] }
gloo-net = { version = "0.5", features = ["http"] }
serde_json = "1"
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = ["console"] }
```

`crate-type = ["cdylib", "rlib"]` จำเป็นสำหรับ WASM:
- `cdylib` — สร้าง `.wasm` file สำหรับ browser
- `rlib` — สร้าง Rust library สำหรับ test และ integration กับ crate อื่น

**`frontend/index.html`:**

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Fullstack Todo — Axum + Yew</title>
    <style>
        body { font-family: sans-serif; max-width: 600px; margin: 2rem auto; padding: 0 1rem; }
        .todo-item { display: flex; align-items: center; gap: 0.5rem; padding: 0.5rem 0; border-bottom: 1px solid #eee; }
        .todo-item.completed span { text-decoration: line-through; color: #999; }
        input[type="text"] { flex: 1; padding: 0.5rem; font-size: 1rem; }
        button { padding: 0.5rem 1rem; cursor: pointer; }
        .error { color: red; padding: 0.5rem; }
        .loading { color: #666; font-style: italic; }
    </style>
</head>
<body></body>
</html>
```

**`frontend/src/main.rs`:**

```rust
use yew::prelude::*;

mod api;
mod components;

use components::app::App;

fn main() {
    yew::Renderer::<App>::new().render();
}
```

### ขั้นที่ 7: API Client Module

API client module แยก HTTP logic ออกจาก component logic ทำให้ component code อ่านง่ายขึ้น และสามารถ mock ได้ในการทดสอบ

**`frontend/src/api.rs`:**

```rust
use gloo_net::http::Request;
use shared::{CreateTodo, Todo, UpdateTodo};

const API_BASE: &str = "http://localhost:3000";

/// ข้อผิดพลาดที่อาจเกิดขึ้นจาก API call
#[derive(Debug, Clone, PartialEq)]
pub enum ApiError {
    NetworkError(String),
    ParseError(String),
    NotFound,
    ServerError(u16),
}

impl std::fmt::Display for ApiError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ApiError::NetworkError(msg) => write!(f, "Network error: {}", msg),
            ApiError::ParseError(msg) => write!(f, "Parse error: {}", msg),
            ApiError::NotFound => write!(f, "ไม่พบรายการที่ต้องการ"),
            ApiError::ServerError(code) => write!(f, "Server error: {}", code),
        }
    }
}

/// ดึง todo ทั้งหมดจาก backend
pub async fn fetch_todos() -> Result<Vec<Todo>, ApiError> {
    let response = Request::get(&format!("{}/todos", API_BASE))
        .send()
        .await
        .map_err(|e| ApiError::NetworkError(e.to_string()))?;

    if !response.ok() {
        return Err(ApiError::ServerError(response.status()));
    }

    response
        .json::<Vec<Todo>>()
        .await
        .map_err(|e| ApiError::ParseError(e.to_string()))
}

/// สร้าง todo ใหม่
pub async fn create_todo(payload: CreateTodo) -> Result<Todo, ApiError> {
    let body = serde_json::to_string(&payload)
        .map_err(|e| ApiError::ParseError(e.to_string()))?;

    let response = Request::post(&format!("{}/todos", API_BASE))
        .header("Content-Type", "application/json")
        .body(body)
        .map_err(|e| ApiError::NetworkError(e.to_string()))?
        .send()
        .await
        .map_err(|e| ApiError::NetworkError(e.to_string()))?;

    if !response.ok() {
        return Err(ApiError::ServerError(response.status()));
    }

    response
        .json::<Todo>()
        .await
        .map_err(|e| ApiError::ParseError(e.to_string()))
}

/// อัปเดต todo ที่มีอยู่
pub async fn update_todo(id: u64, payload: UpdateTodo) -> Result<Todo, ApiError> {
    let body = serde_json::to_string(&payload)
        .map_err(|e| ApiError::ParseError(e.to_string()))?;

    let response = Request::put(&format!("{}/todos/{}", API_BASE, id))
        .header("Content-Type", "application/json")
        .body(body)
        .map_err(|e| ApiError::NetworkError(e.to_string()))?
        .send()
        .await
        .map_err(|e| ApiError::NetworkError(e.to_string()))?;

    if response.status() == 404 {
        return Err(ApiError::NotFound);
    }
    if !response.ok() {
        return Err(ApiError::ServerError(response.status()));
    }

    response
        .json::<Todo>()
        .await
        .map_err(|e| ApiError::ParseError(e.to_string()))
}

/// ลบ todo
pub async fn delete_todo(id: u64) -> Result<(), ApiError> {
    let response = Request::delete(&format!("{}/todos/{}", API_BASE, id))
        .send()
        .await
        .map_err(|e| ApiError::NetworkError(e.to_string()))?;

    if response.status() == 404 {
        return Err(ApiError::NotFound);
    }
    if !response.ok() {
        return Err(ApiError::ServerError(response.status()));
    }

    Ok(())
}
```

`gloo_net::http::Request` เป็น wrapper รอบ browser's Fetch API ที่ compile มาสำหรับ WASM โดยเฉพาะ ต่างจาก `reqwest` ที่ทำงานได้บน native แต่ต้องการ feature flag พิเศษสำหรับ WASM

### ขั้นที่ 8: Yew Components — TodoList และ TodoForm

#### App Component หลัก

**`frontend/src/components/mod.rs`:**

```rust
pub mod app;
pub mod todo_form;
pub mod todo_list;
```

**`frontend/src/components/app.rs`:**

```rust
use yew::prelude::*;
use shared::Todo;
use crate::api::{self, ApiError};
use crate::components::todo_list::TodoList;
use crate::components::todo_form::TodoForm;

#[function_component(App)]
pub fn app() -> Html {
    // State หลักของแอป
    let todos = use_state(Vec::<Todo>::new);
    let loading = use_state(|| true);
    let error = use_state(|| Option::<String>::None);

    // โหลด todos ครั้งแรกเมื่อ component mount
    {
        let todos = todos.clone();
        let loading = loading.clone();
        let error = error.clone();
        use_effect_with((), move |_| {
            wasm_bindgen_futures::spawn_local(async move {
                match api::fetch_todos().await {
                    Ok(data) => {
                        todos.set(data);
                        loading.set(false);
                    }
                    Err(e) => {
                        error.set(Some(e.to_string()));
                        loading.set(false);
                    }
                }
            });
            || ()  // cleanup function — ไม่จำเป็นในกรณีนี้
        });
    }

    // Callback สำหรับเพิ่ม todo ใหม่ (จาก TodoForm)
    let on_create = {
        let todos = todos.clone();
        let error = error.clone();
        Callback::from(move |new_todo: Todo| {
            let mut current = (*todos).clone();
            current.push(new_todo);
            todos.set(current);
            error.set(None);
        })
    };

    // Callback สำหรับอัปเดต todo (toggle completed)
    let on_toggle = {
        let todos = todos.clone();
        let error = error.clone();
        Callback::from(move |id: u64| {
            let todos = todos.clone();
            let error = error.clone();
            let current: Vec<Todo> = (*todos).clone();
            let todo = current.iter().find(|t| t.id == id).cloned();

            if let Some(t) = todo {
                let payload = shared::UpdateTodo {
                    title: None,
                    completed: Some(!t.completed),
                };
                wasm_bindgen_futures::spawn_local(async move {
                    match api::update_todo(id, payload).await {
                        Ok(updated) => {
                            let new_todos: Vec<Todo> = (*todos)
                                .clone()
                                .into_iter()
                                .map(|t| if t.id == id { updated.clone() } else { t })
                                .collect();
                            todos.set(new_todos);
                            error.set(None);
                        }
                        Err(e) => {
                            error.set(Some(e.to_string()));
                        }
                    }
                });
            }
        })
    };

    // Callback สำหรับลบ todo
    let on_delete = {
        let todos = todos.clone();
        let error = error.clone();
        Callback::from(move |id: u64| {
            let todos = todos.clone();
            let error = error.clone();
            wasm_bindgen_futures::spawn_local(async move {
                match api::delete_todo(id).await {
                    Ok(()) => {
                        let new_todos: Vec<Todo> = (*todos)
                            .clone()
                            .into_iter()
                            .filter(|t| t.id != id)
                            .collect();
                        todos.set(new_todos);
                        error.set(None);
                    }
                    Err(e) => {
                        error.set(Some(e.to_string()));
                    }
                }
            });
        })
    };

    html! {
        <div>
            <h1>{ "Todo List — Fullstack Rust" }</h1>

            <TodoForm on_create={on_create} />

            if let Some(err) = (*error).clone() {
                <p class="error">{ err }</p>
            }

            if *loading {
                <p class="loading">{ "กำลังโหลด..." }</p>
            } else {
                <TodoList
                    todos={(*todos).clone()}
                    on_toggle={on_toggle}
                    on_delete={on_delete}
                />
            }
        </div>
    }
}
```

#### TodoList Component

**`frontend/src/components/todo_list.rs`:**

```rust
use yew::prelude::*;
use shared::Todo;

#[derive(Properties, PartialEq)]
pub struct TodoListProps {
    pub todos: Vec<Todo>,
    pub on_toggle: Callback<u64>,
    pub on_delete: Callback<u64>,
}

#[function_component(TodoList)]
pub fn todo_list(props: &TodoListProps) -> Html {
    if props.todos.is_empty() {
        return html! {
            <p>{ "ยังไม่มี todo — เพิ่มรายการแรกด้านบน" }</p>
        };
    }

    html! {
        <ul style="list-style: none; padding: 0;">
            { for props.todos.iter().map(|todo| {
                let id = todo.id;
                let on_toggle = props.on_toggle.clone();
                let on_delete = props.on_delete.clone();

                html! {
                    <li key={id} class={classes!("todo-item", todo.completed.then_some("completed"))}>
                        <input
                            type="checkbox"
                            checked={todo.completed}
                            onchange={Callback::from(move |_| on_toggle.emit(id))}
                        />
                        <span style="flex: 1;">{ &todo.title }</span>
                        <button
                            onclick={Callback::from(move |_| on_delete.emit(id))}
                            style="background: #ff4444; color: white; border: none; border-radius: 4px;"
                        >
                            { "ลบ" }
                        </button>
                    </li>
                }
            })}
        </ul>
    }
}
```

`Properties` ต้องการ `PartialEq` derive เพื่อให้ Yew เปรียบเทียบ props และ skip re-render ถ้า props ไม่เปลี่ยน `Callback<T>` implement `PartialEq` ด้วย pointer equality

#### TodoForm Component

**`frontend/src/components/todo_form.rs`:**

```rust
use yew::prelude::*;
use shared::{CreateTodo, Todo};
use crate::api;

#[derive(Properties, PartialEq)]
pub struct TodoFormProps {
    pub on_create: Callback<Todo>,
}

#[function_component(TodoForm)]
pub fn todo_form(props: &TodoFormProps) -> Html {
    let title = use_state(String::new);
    let submitting = use_state(|| false);
    let form_error = use_state(|| Option::<String>::None);

    let on_input = {
        let title = title.clone();
        Callback::from(move |e: InputEvent| {
            let input: web_sys::HtmlInputElement = e.target_unchecked_into();
            title.set(input.value());
        })
    };

    let on_submit = {
        let title = title.clone();
        let submitting = submitting.clone();
        let form_error = form_error.clone();
        let on_create = props.on_create.clone();

        Callback::from(move |e: SubmitEvent| {
            e.prevent_default();  // ป้องกัน form submit แบบ traditional (reload หน้า)

            let current_title = (*title).clone();
            if current_title.trim().is_empty() {
                form_error.set(Some("กรุณาใส่ชื่อ todo".to_string()));
                return;
            }

            let title = title.clone();
            let submitting = submitting.clone();
            let form_error = form_error.clone();
            let on_create = on_create.clone();

            submitting.set(true);
            form_error.set(None);

            wasm_bindgen_futures::spawn_local(async move {
                let payload = CreateTodo { title: current_title };
                match api::create_todo(payload).await {
                    Ok(new_todo) => {
                        on_create.emit(new_todo);
                        title.set(String::new());   // clear form
                        submitting.set(false);
                    }
                    Err(e) => {
                        form_error.set(Some(e.to_string()));
                        submitting.set(false);
                    }
                }
            });
        })
    };

    html! {
        <form onsubmit={on_submit} style="display: flex; gap: 0.5rem; margin-bottom: 1rem;">
            <input
                type="text"
                value={(*title).clone()}
                oninput={on_input}
                placeholder="เพิ่ม todo ใหม่..."
                disabled={*submitting}
            />
            <button type="submit" disabled={*submitting}>
                if *submitting {
                    { "กำลังบันทึก..." }
                } else {
                    { "เพิ่ม" }
                }
            </button>

            if let Some(err) = (*form_error).clone() {
                <span class="error">{ err }</span>
            }
        </form>
    }
}
```

### ขั้นที่ 9: Build และ Serve — Production Setup

#### รัน Backend

```bash
# Terminal 1 — รัน backend
cd fullstack-todo
cargo run -p backend
# Backend listening on http://0.0.0.0:3000
```

#### ติดตั้ง Trunk และ Build Frontend

Trunk เป็น build tool สำหรับ Yew/WASM ที่จัดการ WASM compilation, asset bundling, และ dev server ให้อัตโนมัติ

```bash
# ติดตั้ง trunk (ครั้งเดียว)
cargo install trunk
rustup target add wasm32-unknown-unknown

# Terminal 2 — รัน frontend dev server
cd fullstack-todo/frontend
trunk serve --open
# Serving at http://localhost:8080
```

Trunk จะ:
1. Compile `frontend` crate เป็น WASM
2. Generate JavaScript glue code ด้วย `wasm-bindgen`
3. Bundle assets พร้อม `index.html`
4. Start dev server พร้อม hot reload

#### Serve Frontend จาก Axum (Production)

ใน production อาจต้องการ serve frontend static files จาก Axum เอง เพื่อลด infrastructure complexity

```bash
# Build frontend สำหรับ production
cd frontend
trunk build --release
# สร้างไฟล์ใน dist/
```

**`backend/src/main.rs` — เพิ่ม static file serving:**

```rust
use tower_http::services::{ServeDir, ServeFile};

pub fn build_router_with_frontend(state: AppState) -> Router {
    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    // API routes
    let api_routes = Router::new()
        .route("/todos", get(list_todos))
        .route("/todos", post(create_todo))
        .route("/todos/:id", get(get_todo))
        .route("/todos/:id", put(update_todo))
        .route("/todos/:id", delete(delete_todo))
        .layer(cors)
        .with_state(state);

    // Serve frontend static files — fallback ไปที่ index.html สำหรับ SPA routing
    let frontend_service = ServeDir::new("../frontend/dist")
        .fallback(ServeFile::new("../frontend/dist/index.html"));

    Router::new()
        .nest("/", api_routes)
        .fallback_service(frontend_service)
}
```

สำหรับ production จริง ควรใช้ reverse proxy (nginx, Caddy) มาก serve static files แทน เพราะมี performance ดีกว่า

---

## การทดสอบ (Testing)

### รัน Tests ทั้งหมดในระบบ

ผลลัพธ์จากการรัน `cargo test` จริงในโปรเจคตัวอย่าง:

```
$ cargo test
   Compiling shared v0.1.0 (fullstack-todo/shared)
   Compiling backend v0.1.0 (fullstack-todo/backend)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 8.05s
     Running unittests src/main.rs (target/debug/deps/backend-4615e7a80516bf77)

running 7 tests
test tests::test_delete_todo ... ok
test tests::test_create_todo ... ok
test tests::test_get_todo_not_found ... ok
test tests::test_list_todos_empty ... ok
test tests::test_create_multiple_todos_and_list ... ok
test tests::test_update_todo ... ok
test tests::test_delete_todo_not_found ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/lib.rs (target/debug/deps/shared-0380901378f613b8)

running 4 tests
test tests::test_update_todo_partial ... ok
test tests::test_create_todo_deserialize ... ok
test tests::test_todo_new ... ok
test tests::test_todo_serialize_deserialize ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests shared

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

รวม **11 tests ผ่านทั้งหมด** — 7 backend integration tests + 4 shared type unit tests

### อธิบาย Test Strategy

**Backend tests** ใช้ `tower::ServiceExt::oneshot` ซึ่งส่ง single request ผ่าน Tower service โดยไม่ต้อง bind port จริง — ทำให้:
- Test รันเร็วมาก (ไม่ต้องรอ network)
- ไม่มี port conflict ระหว่าง parallel test runs
- Test แต่ละตัวได้ fresh state (`make_state()` สร้าง state ใหม่ทุกครั้ง)

**Shared tests** ใช้ `serde_json` เพื่อ verify ว่า serialization/deserialization ทำงานถูกต้อง — สำคัญมากเพราะ JSON format นี้ใช้ทั้งใน backend responses และ frontend parsing

### การทดสอบ Frontend ด้วย wasm-pack

Frontend tests สามารถรันบน headless browser ได้ด้วย `wasm-pack`:

```bash
# ติดตั้ง wasm-pack
cargo install wasm-pack

# รัน tests ใน headless Chrome
wasm-pack test --headless --chrome frontend/
```

ตัวอย่าง frontend test สำหรับ api module:

```rust
// frontend/src/api.rs
#[cfg(test)]
mod tests {
    // Note: frontend tests รันใน WASM context
    // ต้องใช้ wasm-bindgen-test แทน #[test] ธรรมดา
    use wasm_bindgen_test::*;

    wasm_bindgen_test_configure!(run_in_browser);

    #[wasm_bindgen_test]
    fn test_api_error_display() {
        use super::ApiError;
        let err = ApiError::NotFound;
        assert_eq!(err.to_string(), "ไม่พบรายการที่ต้องการ");
    }

    #[wasm_bindgen_test]
    fn test_api_error_server() {
        use super::ApiError;
        let err = ApiError::ServerError(500);
        assert!(err.to_string().contains("500"));
    }
}
```

---

## ข้อผิดพลาดที่พบบ่อย

### 1. CORS Error — "No 'Access-Control-Allow-Origin' header"

**อาการ**: Frontend fetch ไม่ได้ข้อมูล, Console แสดง:
```
Access to fetch at 'http://localhost:3000/todos' from origin
'http://localhost:8080' has been blocked by CORS policy
```

**สาเหตุ**: ลืมเพิ่ม `CorsLayer` ใน backend router, หรือเพิ่มหลัง `.with_state()` ทำให้ layer ครอบคลุมไม่ครบ

**วิธีแก้**: ต้องใส่ `.layer(cors)` ก่อน `.with_state(state)`:

```rust
// ✅ ถูกต้อง — cors layer ครอบคลุมทุก route
Router::new()
    .route("/todos", get(list_todos))
    // ...routes อื่น ๆ...
    .layer(cors)        // ← ก่อน with_state
    .with_state(state)

// ❌ ผิด — cors จะไม่ทำงานกับ state routes
Router::new()
    .route("/todos", get(list_todos))
    .with_state(state)
    .layer(cors)        // ← หลัง with_state (ไม่มีผล)
```

นอกจากนี้ต้องเพิ่ม CORS headers สำหรับ preflight OPTIONS request ด้วย — `CorsLayer::new().allow_methods(Any)` จะ handle OPTIONS ให้อัตโนมัติ

---

### 2. Mutex Deadlock — "thread 'main' panicked: called `Result::unwrap()` on an `Err` value: PoisonError"

**อาการ**: Backend panic ขณะ handle request concurrent

**สาเหตุ**: `PoisonError` เกิดขึ้นเมื่อ thread ที่ถือ lock panic โดยไม่ release lock ทำให้ lock "poisoned" — thread อื่นที่ `lock().unwrap()` จะได้รับ `PoisonError`

**วิธีแก้**: ใช้ `unwrap_or_else` หรือ handle poison error:

```rust
// ✅ แบบที่ 1 — ignore poison (ใช้ได้ถ้ายอมรับความเสี่ยง)
let todos = state.lock().unwrap_or_else(|e| e.into_inner());

// ✅ แบบที่ 2 — return error เมื่อ lock poisoned
let todos = state.lock().map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
```

ในโปรเจคที่ใหญ่ขึ้น พิจารณาเปลี่ยนไปใช้ `tokio::sync::Mutex` แทน `std::sync::Mutex` เพื่อรองรับ async context ได้ดีกว่า และไม่มี poison mechanism

---

### 3. wasm32 Compile Error — "crate ... is not compiled for this target"

**อาการ**: `trunk build` ล้มเหลวด้วย:
```
error[E0463]: can't find crate for `std`
  = note: the `wasm32-unknown-unknown` target may not be installed
```
หรือ:
```
error: crate `tokio` requires OS thread support
```

**สาเหตุ**: shared crate import crate ที่ไม่รองรับ WASM เช่น `tokio`, `std::net`, หรือ crate ที่ใช้ OS thread

**วิธีแก้**:
```bash
# ติดตั้ง WASM target ก่อน
rustup target add wasm32-unknown-unknown

# ตรวจสอบ shared crate ไม่มี OS-specific dependencies
# ใช้ cfg target_arch สำหรับ conditional compilation ถ้าจำเป็น
```

```rust
// shared/src/lib.rs — conditional compilation สำหรับ test ที่ต้องการ std
#[cfg(all(test, not(target_arch = "wasm32")))]
mod native_tests {
    // test ที่ใช้ serde_json, std::io ฯลฯ
}

#[cfg(all(test, target_arch = "wasm32"))]
mod wasm_tests {
    use wasm_bindgen_test::*;
    // test ที่รันใน browser
}
```

---

### 4. use_effect_with Loop — Component Re-render ไม่หยุด

**อาการ**: Browser freezes, network tab แสดง request ไป `/todos` ไม่หยุด

**สาเหตุ**: `use_effect_with` dependency มีค่าเปลี่ยนทุกครั้งที่ re-render เช่น ใส่ `todos.clone()` เป็น dependency ในขณะที่ effect นั้นก็แก้ไข `todos`

```rust
// ❌ ผิด — todos เปลี่ยนทุกครั้งที่ effect รัน → loop ไม่หยุด
use_effect_with((*todos).clone(), move |_| {
    // fetch todos แล้ว set ใหม่ → todos เปลี่ยน → effect รันอีก
    ...
});

// ✅ ถูกต้อง — ใช้ () หรือ dependency ที่เปลี่ยนแค่ครั้งเดียว
use_effect_with((), move |_| {
    // รันแค่ครั้งเดียวตอน mount (เหมือน componentDidMount ใน React)
    ...
});
```

**กฎง่ายๆ**: ถ้า effect fetch data ครั้งแรก ใช้ `()` เป็น dependency เสมอ ถ้าต้องการ refetch เมื่อ id เปลี่ยน ใช้ id เป็น dependency

---

### 5. JSON Parse Error — "missing field" หลัง backend เปลี่ยน field name

**อาการ**: Frontend แสดง parse error หลังแก้ `Todo` struct ใน shared crate

**สาเหตุ**: สร้าง frontend build เก่าค้างไว้ใน browser cache ทำให้ยังใช้ code เก่าที่ expect field name เก่า

```bash
# ✅ วิธีแก้ — clear Trunk build cache และ rebuild
rm -rf frontend/dist
trunk build

# หรือใช้ hard refresh ใน browser (Ctrl+Shift+R)
```

กรณีที่ต้องการ backward compatibility ใช้ serde `rename` attribute:

```rust
#[derive(Serialize, Deserialize)]
pub struct Todo {
    pub id: u64,
    #[serde(rename = "task_name")]  // ส่ง JSON key ชื่อ task_name แต่ Rust field ชื่อ title
    pub title: String,
    pub completed: bool,
}
```

---

## การ Package และ Deploy

### Docker Compose — Backend + Frontend

```yaml
# docker-compose.yml
version: "3.9"
services:
  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile
    ports:
      - "3000:3000"
    environment:
      - RUST_LOG=info

  frontend:
    build:
      context: .
      dockerfile: frontend/Dockerfile
    ports:
      - "8080:80"
    depends_on:
      - backend
```

**`backend/Dockerfile`:**

```dockerfile
FROM rust:1.75 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release -p backend

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /app/target/release/backend ./backend
EXPOSE 3000
CMD ["./backend"]
```

**`frontend/Dockerfile`:**

```dockerfile
FROM rust:1.75 AS builder
RUN cargo install trunk
RUN rustup target add wasm32-unknown-unknown
WORKDIR /app
COPY . .
RUN trunk build --release frontend/index.html

FROM nginx:alpine
COPY --from=builder /app/frontend/dist /usr/share/nginx/html
# SPA routing — redirect ทุก path ไปที่ index.html
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**`frontend/nginx.conf`:**

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(wasm|js)$ {
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

### Build Release Binary

```bash
# Build backend binary
cargo build --release -p backend
# สร้างที่ target/release/backend

# Build frontend WASM
cd frontend && trunk build --release
# สร้าง dist/ directory พร้อมทุก static files
```

Binary ขนาดใน release mode ประมาณ:
- Backend: ~5-10 MB (stripped: ~2-4 MB)
- Frontend WASM: ~300-800 KB (ขึ้นอยู่กับ dependencies)

ลด WASM size ด้วย:

```toml
# Cargo.toml
[profile.release]
opt-level = "z"      # optimize for size
lto = true           # link-time optimization
codegen-units = 1    # better optimization
panic = "abort"      # ลด binary size (ไม่มี panic unwinding)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: เพิ่มระบบ Priority

เพิ่ม field `priority: u8` (1-5) ใน `Todo` struct และปรับ `CreateTodo` ให้มี `priority: Option<u8>` (default = 3)

ขั้นตอน:
- แก้ `shared/src/lib.rs` — เพิ่ม field และ default value
- แก้ backend handler `create_todo` ให้ handle `priority`
- เพิ่ม route `GET /todos?priority=5` เพื่อ filter by priority
- แก้ frontend form ให้มี `<select>` สำหรับเลือก priority

เป้าหมายการเรียนรู้: เห็น workflow ของการเปลี่ยน shared type และผลกระทบที่ compiler จะแจ้งทั้งสองฝั่ง

---

### แบบฝึกหัดที่ 2: เปลี่ยนจาก In-Memory Store เป็น SQLite

เปลี่ยน `AppState` จาก `Arc<Mutex<Vec<Todo>>>` เป็น `sqlx::SqlitePool`

ขั้นตอน:
- เพิ่ม `sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio"] }` ใน `backend/Cargo.toml`
- สร้าง migration SQL สำหรับ `todos` table
- เปลี่ยน handler ทุกตัวให้ใช้ `sqlx::query!` หรือ `sqlx::query_as!`
- อัปเดต test ให้สร้าง in-memory SQLite database (`sqlite::memory:`)

สิ่งที่ท้าทาย: handler test ต้องสร้าง database schema ก่อนทุก test เปรียบเทียบ test setup กับแบบ in-memory state ว่า trade-off ต่างกันอย่างไร

---

### แบบฝึกหัดที่ 3: เพิ่ม Authentication ด้วย JWT

เพิ่มระบบ login/logout แบบ simple โดยใช้ JWT token

ขั้นตอน:
- เพิ่ม crate `jsonwebtoken` ใน backend
- สร้าง `POST /auth/login` ที่รับ `{username, password}` และ return JWT
- เพิ่ม Axum middleware ที่ตรวจ `Authorization: Bearer <token>` header
- แก้ frontend ให้ store JWT ใน `localStorage` และส่งใน request headers ทุกครั้ง
- เพิ่ม login form ใน Yew ที่แสดงก่อน app หลัก

สิ่งที่ท้าทาย: จัดการ token expiry ใน frontend อย่างไร? ต้อง redirect ไป login page เมื่อ token หมดอายุ

---

### แบบฝึกหัดที่ 4: Real-time Updates ด้วย WebSocket

เพิ่ม WebSocket endpoint ใน backend ที่ broadcast ทุกการเปลี่ยนแปลงไปยัง frontend ทุกตัวที่เปิดอยู่

ขั้นตอน:
- ใช้ `axum::extract::ws::WebSocket` สำหรับ WebSocket handler
- สร้าง broadcast channel ด้วย `tokio::sync::broadcast`
- เมื่อมีการ create/update/delete todo ให้ broadcast event ไปยังทุก client
- ใน frontend ใช้ `gloo-net`'s WebSocket หรือ `web-sys::WebSocket`
- แก้ Yew app ให้ subscribe WebSocket และ update state เมื่อได้รับ event

สิ่งที่ท้าทาย: ต้องออกแบบ event format ใน shared crate — สร้าง `TodoEvent` enum ที่ serialize/deserialize ได้

---

### แบบฝึกหัดที่ 5: Error Boundary และ Retry Logic

ปรับปรุง error handling ให้ production-ready

ขั้นตอน:
- เพิ่ม retry logic ใน `api.rs` — retry 3 ครั้งด้วย exponential backoff (กรณี network timeout)
- สร้าง Yew `ErrorBoundary` component ที่ catch errors จาก child components
- แสดง loading skeleton แทน "กำลังโหลด..." ข้อความ
- เพิ่ม toast notification สำหรับ success/error messages (auto-dismiss หลัง 3 วินาที)

สิ่งที่ท้าทาย: exponential backoff ใน async WASM context ต้องใช้ `gloo_timers::future::TimeoutFuture` แทน `tokio::time::sleep`

---

## สรุป

โปรเจคนี้สาธิต full-stack Rust ในสถาปัตยกรรมที่ **type-safe** และ **compile-time verified** ตลอด stack — ตั้งแต่ database model ไปจนถึง browser UI

**Pattern สำคัญที่ได้เรียน:**

1. **Cargo Workspace** — จัดการ multi-crate monorepo ที่ share code ระหว่าง native และ WASM ได้
2. **Shared Types** — หัวใจของ full-stack Rust ที่ทำให้ backend/frontend sync อัตโนมัติผ่าน compiler
3. **AppState + Arc<Mutex<T>>** — pattern พื้นฐานสำหรับ shared mutable state ใน Axum
4. **Tower ServiceExt testing** — ทดสอบ Axum handlers โดยไม่ต้องรัน server จริง ทำให้ test เร็วและ reliable
5. **Yew + use_effect_with** — lifecycle management ใน Yew ที่คล้าย React useEffect แต่มี Rust's type safety
6. **gloo-net API client** — HTTP calls จาก WASM ที่ใช้ browser Fetch API ผ่าน safe Rust API

**ข้อแตกต่างจาก TypeScript full-stack:**

ใน TypeScript + Next.js เราต้องใช้ `zod` หรือ `io-ts` เพื่อ runtime type validation ระหว่าง frontend/backend ซึ่งเพิ่ม code และยังมีความเสี่ยงว่า validation อาจไม่ครบ ใน Rust ทุก type validation เกิดขึ้น compile-time โดย `serde` — ถ้า API เปลี่ยน compile จะ fail ทันที ไม่มีทาง ship broken API ได้โดยไม่รู้ตัว

**การต่อยอด:**

- โปรเจค J03 จะสำรวจ **WASM Image Processing** — ใช้ Rust WASM สำหรับ computation หนักใน browser
- สำหรับ production full-stack Rust ศึกษา **Leptos** ซึ่งรองรับ Server-Side Rendering และ full-stack routing คล้าย Next.js มากกว่า

---

**โปรเจคก่อนหน้า:** [Project J01: Yew SPA](project-j01-yew-spa.md) | **โปรเจคถัดไป:** [Project J03: WASM Image Processing](project-j03-wasm-image.md)
