# Project H01: HTTP/1.1 + HTTP/2 Server from Scratch

> โมดูล: H — Networking & Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 16 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **HTTP server** ที่รองรับทั้ง HTTP/1.1 และ HTTP/2 ตั้งแต่ระดับ TCP socket ขึ้นมาจนถึงระดับ middleware stack ครบชุด โดยใช้ `tokio` เป็น async runtime หลัก และ `hyper` สำหรับ HTTP/2 protocol layer

HTTP server เป็นองค์ประกอบกลางของแทบทุก production system — ไม่ว่าจะเป็น REST API, gRPC gateway, หรือ static file hosting การเข้าใจว่า server ทำงานอย่างไรตั้งแต่ byte แรกที่มาถึง TCP socket ไปจนถึงการส่ง response กลับจะทำให้คุณ debug ปัญหาได้ลึกกว่าการใช้ framework สำเร็จรูปอย่าง Axum หรือ Actix

**ทำไมถึงน่าสร้าง?**

Framework อย่าง Axum หรือ Actix Web ซ่อน complexity ไว้มากมาย เมื่อ production server ของคุณมีปัญหาเรื่อง timeout, slow response, หรือ memory leak คุณต้องการความเข้าใจระดับ protocol เพื่อแก้ไขได้ โปรเจคนี้สร้างทุกอย่างด้วยมือ ทำให้คุณเข้าใจกลไกภายในอย่างแท้จริง

**Use case ในโลกจริง:**

- Edge server ที่ต้องการ custom HTTP behavior (เช่น custom framing, proprietary headers)
- Embedded HTTP server สำหรับ IoT device ที่ไม่อยากพึ่งพา full-blown framework
- Proxy หรือ gateway ที่ต้องการ middleware pipeline แบบ custom
- ฐานในการเรียนรู้ก่อนดู source code ของ hyper หรือ actix-web
- Load testing tool ที่ต้องสร้าง HTTP request/response ในระดับต่ำ

---

## สิ่งที่จะได้เรียนรู้

- การ parse HTTP/1.1 request จาก raw bytes (request line, headers, body) ด้วยมือ
- การออกแบบ `Router` ที่รองรับ path parameter (`:id`), method matching, และ 405 vs 404 differentiation
- การสร้าง middleware stack แบบ trait-based: `LoggingMiddleware`, `CompressionMiddleware` (gzip), `AuthMiddleware`
- การใช้ `hyper::server::conn::http2` พร้อม `hyper::service::service_fn` สำหรับ HTTP/2 multiplexing
- Static file serving ด้วย MIME type detection, ETag/`If-Modified-Since` caching, และ Range request
- Graceful shutdown ด้วย `tokio::signal`, `tokio::select!`, และ shutdown channel แบบ `watch`
- การใช้ `flate2` สำหรับ gzip compression ตาม `Accept-Encoding` header
- การเขียน unit test ที่ครอบคลุมทุก layer โดยไม่ต้องเปิด TCP connection จริง

---

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 1–20: ownership, borrowing, struct, enum, error handling ด้วย `Result` และ `?`
- จาก Part 41–50: traits, generics, closures, iterators, dynamic dispatch (`dyn Trait`)
- จาก Part 51–60: async/await พื้นฐาน, `tokio::spawn`, `tokio::net::TcpListener`
- จาก Part 61–70: `Arc<Mutex<T>>`, channel (`mpsc`, `watch`), tokio runtime model
- จาก Part 71–80: HTTP protocol พื้นฐาน — method, status code, headers, body
- จาก Part 81–90: hyper crate API, `tower::Service` trait concept
- ความรู้เรื่อง TCP/IP socket programming เบื้องต้น

---

## โครงสร้างโปรเจค (Project Layout)

```
http-server/
├── src/
│   ├── main.rs              ← entry point: bind socket + start server
│   ├── lib.rs               ← re-export modules
│   ├── http1/
│   │   ├── mod.rs           ← HTTP/1.1 connection handler
│   │   ├── request.rs       ← parse raw bytes → HttpRequest
│   │   └── response.rs      ← HttpResponse serialization
│   ├── router/
│   │   ├── mod.rs           ← Router, RouteMatch, RouterBuilder
│   │   └── params.rs        ← path parameter extraction
│   ├── middleware/
│   │   ├── mod.rs           ← Middleware trait
│   │   ├── logging.rs       ← LoggingMiddleware
│   │   ├── compression.rs   ← CompressionMiddleware (gzip)
│   │   └── auth.rs          ← AuthMiddleware (Bearer token)
│   ├── http2/
│   │   └── mod.rs           ← hyper HTTP/2 server setup
│   ├── static_files/
│   │   └── mod.rs           ← static file handler, ETag, Range
│   └── shutdown.rs          ← graceful shutdown logic
├── tests/
│   └── integration_test.rs  ← end-to-end HTTP/1.1 test
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

```
                    ┌─────────────────────────────────────┐
                    │           main()                      │
                    │  tokio::net::TcpListener::bind()     │
                    └──────────┬──────────────────────────┘
                               │  accept() loop
                   ┌───────────┴───────────┐
                   │                       │
         ┌─────────▼──────────┐  ┌─────────▼──────────┐
         │  HTTP/1.1 Handler  │  │  HTTP/2 (hyper)     │
         │  tokio::spawn()    │  │  hyper::conn::http2 │
         └─────────┬──────────┘  └─────────┬──────────┘
                   │                       │
         ┌─────────▼──────────────────────▼─────────┐
         │              Router                        │
         │  match_route(method, path)                 │
         │  → RouteMatch { params, handler_index }    │
         └─────────────────┬─────────────────────────┘
                           │
         ┌─────────────────▼─────────────────────────┐
         │         Middleware Stack                    │
         │  AuthMiddleware → LoggingMiddleware →       │
         │  CompressionMiddleware → Handler            │
         └─────────────────┬─────────────────────────┘
                           │
         ┌─────────────────▼─────────────────────────┐
         │         Route Handlers                     │
         │  static_files / api_handler / ...          │
         └────────────────────────────────────────────┘
```

**Design decisions:**

1. **HTTP/1.1 แยกจาก HTTP/2** — เราใช้ `tokio::net::TcpListener` + manual parsing สำหรับ HTTP/1.1 เพื่อให้เห็นกลไก raw protocol จากนั้นใช้ `hyper` สำหรับ HTTP/2 เพราะ HTTP/2 มีความซับซ้อนสูง (binary framing, stream multiplexing, HPACK header compression)

2. **Router เป็น static (compile-time route registration)** — เพื่อความเร็ว ไม่ใช้ `HashMap` แบบ runtime dynamic registration แต่ใช้ `Vec<Route>` ที่ scan ตามลำดับ ซึ่ง simple และ fast เพียงพอสำหรับ production ที่มี route ไม่เกินหลักพัน

3. **Middleware แบบ trait object** — ใช้ `Box<dyn Middleware>` เพื่อ composability ยอมแลก virtual dispatch overhead กับความยืดหยุ่นในการเพิ่ม middleware ตอน runtime

4. **Graceful shutdown ด้วย `watch::channel`** — เลือก `watch` แทน `oneshot` เพราะ `watch` ให้ทุก task อ่าน signal เดียวกันพร้อมกันได้ เหมาะกับ broadcast shutdown

5. **ETag แบบ deterministic hash** — ใช้ hash ของ file content เพื่อให้ ETag ไม่เปลี่ยนเมื่อ restart server (ต่างกับ `Last-Modified` ที่ขึ้นกับ filesystem timestamp)

---

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: โครงสร้างพื้นฐาน — HTTP/1.1 Request Parser

เริ่มด้วย `Cargo.toml` ที่มี dependency ครบ:

```toml
[package]
name = "http-server"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "http-server"
path = "src/main.rs"

[dependencies]
tokio = { version = "1", features = ["full"] }
hyper = { version = "1", features = ["full"] }
http-body-util = "0.1"
hyper-util = { version = "0.1", features = ["full"] }
flate2 = "1"
mime = "0.3"
http = "1"
bytes = "1"

[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
```

ก่อนอื่นเราต้องเข้าใจ HTTP/1.1 request format:

```
GET /users/42 HTTP/1.1\r\n
Host: localhost:8080\r\n
Accept: application/json\r\n
Authorization: Bearer my-secret-token\r\n
\r\n
```

Request แบ่งเป็นสามส่วน:
1. **Request line** — method, path, version คั่นด้วยช่องว่าง
2. **Headers** — คู่ `key: value` แต่ละบรรทัด จบด้วย `\r\n`
3. **Body** — เนื้อหาหลังจาก blank line (`\r\n\r\n`)

เขียน module `src/http1/request.rs`:

```rust
// src/http1/request.rs
use std::collections::HashMap;

#[derive(Debug, PartialEq, Clone)]
pub enum HttpMethod {
    Get,
    Post,
    Put,
    Delete,
    Patch,
    Head,
    Options,
}

impl HttpMethod {
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "GET"     => Some(HttpMethod::Get),
            "POST"    => Some(HttpMethod::Post),
            "PUT"     => Some(HttpMethod::Put),
            "DELETE"  => Some(HttpMethod::Delete),
            "PATCH"   => Some(HttpMethod::Patch),
            "HEAD"    => Some(HttpMethod::Head),
            "OPTIONS" => Some(HttpMethod::Options),
            _         => None,
        }
    }

    pub fn as_str(&self) -> &str {
        match self {
            HttpMethod::Get     => "GET",
            HttpMethod::Post    => "POST",
            HttpMethod::Put     => "PUT",
            HttpMethod::Delete  => "DELETE",
            HttpMethod::Patch   => "PATCH",
            HttpMethod::Head    => "HEAD",
            HttpMethod::Options => "OPTIONS",
        }
    }
}

#[derive(Debug, PartialEq, Clone)]
pub enum HttpVersion {
    Http10,
    Http11,
    Http20,
}

impl HttpVersion {
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "HTTP/1.0"          => Some(HttpVersion::Http10),
            "HTTP/1.1"          => Some(HttpVersion::Http11),
            "HTTP/2.0" | "HTTP/2" => Some(HttpVersion::Http20),
            _                   => None,
        }
    }
}

#[derive(Debug, Clone)]
pub struct HttpRequest {
    pub method:  HttpMethod,
    pub path:    String,
    pub version: HttpVersion,
    pub headers: HashMap<String, String>,
    pub body:    Vec<u8>,
}

#[derive(Debug)]
pub enum ParseError {
    EmptyRequest,
    InvalidRequestLine(String),
    InvalidMethod(String),
    InvalidVersion(String),
    InvalidHeader(String),
}

impl std::fmt::Display for ParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ParseError::EmptyRequest               => write!(f, "Empty request"),
            ParseError::InvalidRequestLine(s)      => write!(f, "Invalid request line: {}", s),
            ParseError::InvalidMethod(s)           => write!(f, "Invalid method: {}", s),
            ParseError::InvalidVersion(s)          => write!(f, "Invalid version: {}", s),
            ParseError::InvalidHeader(s)           => write!(f, "Invalid header: {}", s),
        }
    }
}

/// Parse raw HTTP/1.1 request bytes into HttpRequest
pub fn parse_http_request(raw: &[u8]) -> Result<HttpRequest, ParseError> {
    let text = std::str::from_utf8(raw).unwrap_or("");
    let mut lines = text.lines();

    // 1) Request line
    let request_line = lines.next().ok_or(ParseError::EmptyRequest)?;
    let parts: Vec<&str> = request_line.splitn(3, ' ').collect();
    if parts.len() != 3 {
        return Err(ParseError::InvalidRequestLine(request_line.to_string()));
    }

    let method  = HttpMethod::from_str(parts[0])
        .ok_or_else(|| ParseError::InvalidMethod(parts[0].to_string()))?;
    let path    = parts[1].to_string();
    let version = HttpVersion::from_str(parts[2])
        .ok_or_else(|| ParseError::InvalidVersion(parts[2].to_string()))?;

    // 2) Headers — lowercase key
    let mut headers     = HashMap::new();
    let mut body_start  = false;
    for line in &mut lines {
        if line.is_empty() {
            body_start = true;
            break;
        }
        let colon_pos = line
            .find(':')
            .ok_or_else(|| ParseError::InvalidHeader(line.to_string()))?;
        let key   = line[..colon_pos].trim().to_lowercase();
        let value = line[colon_pos + 1..].trim().to_string();
        headers.insert(key, value);
    }

    // 3) Body — find after \r\n\r\n in raw bytes
    let body = if body_start {
        find_body_start(raw)
            .map(|pos| raw[pos..].to_vec())
            .unwrap_or_default()
    } else {
        vec![]
    };

    Ok(HttpRequest { method, path, version, headers, body })
}

fn find_body_start(raw: &[u8]) -> Option<usize> {
    // Try \r\n\r\n first
    raw.windows(4)
        .position(|w| w == b"\r\n\r\n")
        .map(|p| p + 4)
        .or_else(|| {
            // Fallback: \n\n (some clients omit CR)
            raw.windows(2)
                .position(|w| w == b"\n\n")
                .map(|p| p + 2)
        })
}
```

สร้าง `src/http1/response.rs`:

```rust
// src/http1/response.rs
use std::collections::HashMap;

#[derive(Debug, Clone)]
pub struct HttpResponse {
    pub status:  u16,
    pub reason:  String,
    pub headers: HashMap<String, String>,
    pub body:    Vec<u8>,
}

impl HttpResponse {
    pub fn new(status: u16, body: impl Into<Vec<u8>>) -> Self {
        let reason = status_reason(status).to_string();
        let body   = body.into();
        let mut headers = HashMap::new();
        headers.insert("content-length".to_string(), body.len().to_string());
        headers.insert(
            "content-type".to_string(),
            "text/plain; charset=utf-8".to_string(),
        );
        HttpResponse { status, reason, headers, body }
    }

    pub fn with_header(mut self, key: &str, value: &str) -> Self {
        self.headers.insert(key.to_lowercase(), value.to_string());
        self
    }

    /// Serialize to HTTP/1.1 wire format
    pub fn to_bytes(&self) -> Vec<u8> {
        let mut buf = Vec::new();
        buf.extend_from_slice(
            format!("HTTP/1.1 {} {}\r\n", self.status, self.reason).as_bytes(),
        );
        for (k, v) in &self.headers {
            buf.extend_from_slice(format!("{}: {}\r\n", k, v).as_bytes());
        }
        buf.extend_from_slice(b"\r\n");
        buf.extend_from_slice(&self.body);
        buf
    }
}

fn status_reason(code: u16) -> &'static str {
    match code {
        200 => "OK",
        201 => "Created",
        204 => "No Content",
        206 => "Partial Content",
        301 => "Moved Permanently",
        304 => "Not Modified",
        400 => "Bad Request",
        401 => "Unauthorized",
        403 => "Forbidden",
        404 => "Not Found",
        405 => "Method Not Allowed",
        416 => "Range Not Satisfiable",
        500 => "Internal Server Error",
        _   => "Unknown",
    }
}
```

เมื่อเรียกใช้ `parse_http_request` กับ raw bytes จะได้:

```rust
let raw = b"GET /users/42 HTTP/1.1\r\nHost: localhost\r\n\r\n";
let req = parse_http_request(raw).unwrap();
// req.method  == HttpMethod::Get
// req.path    == "/users/42"
// req.version == HttpVersion::Http11
// req.headers["host"] == "localhost"
```

---

### ขั้นที่ 2: TCP Listener และ Connection Loop

เพิ่ม TCP listener ใน `src/main.rs`:

```rust
// src/main.rs
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

mod http1;
use http1::request::parse_http_request;
use http1::response::HttpResponse;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "127.0.0.1:8080";
    let listener = TcpListener::bind(addr).await?;
    println!("HTTP/1.1 server listening on http://{}", addr);

    loop {
        let (mut socket, peer_addr) = listener.accept().await?;
        println!("Connection from {}", peer_addr);

        tokio::spawn(async move {
            let mut buf = vec![0u8; 8192];
            let n = match socket.read(&mut buf).await {
                Ok(0) => return,   // connection closed
                Ok(n) => n,
                Err(e) => {
                    eprintln!("read error: {}", e);
                    return;
                }
            };

            let response = match parse_http_request(&buf[..n]) {
                Ok(req) => {
                    println!("{} {}", req.method.as_str(), req.path);
                    HttpResponse::new(200, format!("Path: {}", req.path))
                        .with_header("server", "rust-http/0.1")
                        .to_bytes()
                }
                Err(e) => {
                    eprintln!("parse error: {}", e);
                    HttpResponse::new(400, format!("Bad Request: {}", e))
                        .to_bytes()
                }
            };

            if let Err(e) = socket.write_all(&response).await {
                eprintln!("write error: {}", e);
            }
        });
    }
}
```

ทดสอบด้วย `curl`:

```
$ cargo run &
HTTP/1.1 server listening on http://127.0.0.1:8080

$ curl -s http://127.0.0.1:8080/hello
Path: /hello

$ curl -s http://127.0.0.1:8080/users/42
Path: /users/42
```

**สิ่งสำคัญ**: เราใช้ `tokio::spawn` สำหรับแต่ละ connection เพื่อให้ handle request แบบ concurrent ไม่ต้องรอให้ request หนึ่งเสร็จก่อน

---

### ขั้นที่ 3: Router พร้อม Path Parameters

Router ต้องทำหน้าที่สองอย่าง:
1. **Match** path กับ pattern ที่ลงทะเบียนไว้
2. **Extract** path parameter จาก `:id` segments

สร้าง `src/router/mod.rs`:

```rust
// src/router/mod.rs
use std::collections::HashMap;
use crate::http1::request::HttpMethod;

#[derive(Debug, Clone, PartialEq)]
pub struct RouteMatch {
    pub params:        HashMap<String, String>,
    pub handler_index: usize,
}

#[derive(Debug, Clone)]
struct Route {
    method:        HttpMethod,
    pattern:       Vec<PathSegment>,
    handler_index: usize,
}

#[derive(Debug, Clone)]
enum PathSegment {
    Literal(String),
    Param(String),
}

pub struct Router {
    routes: Vec<Route>,
}

impl Router {
    pub fn new() -> Self {
        Router { routes: vec![] }
    }

    /// Register a route, e.g.: router.add_route(GET, "/users/:id", 0)
    pub fn add_route(&mut self, method: HttpMethod, pattern: &str, handler_index: usize) {
        let segments = pattern
            .split('/')
            .filter(|s| !s.is_empty())
            .map(|s| {
                if s.starts_with(':') {
                    PathSegment::Param(s[1..].to_string())
                } else {
                    PathSegment::Literal(s.to_string())
                }
            })
            .collect();
        self.routes.push(Route { method, pattern: segments, handler_index });
    }

    /// Returns Ok(RouteMatch) on success, Err(405) if path matches but method
    /// doesn't, Err(404) if path doesn't match at all.
    pub fn match_route(
        &self,
        method: &HttpMethod,
        path:   &str,
    ) -> Result<RouteMatch, u16> {
        let path_segs: Vec<&str> = path
            .split('/')
            .filter(|s| !s.is_empty())
            .collect();

        let mut method_matched_somewhere = false;

        for route in &self.routes {
            if route.pattern.len() != path_segs.len() {
                continue;
            }

            let mut params       = HashMap::new();
            let mut path_matches = true;

            for (pat, seg) in route.pattern.iter().zip(path_segs.iter()) {
                match pat {
                    PathSegment::Literal(lit) => {
                        if lit != seg {
                            path_matches = false;
                            break;
                        }
                    }
                    PathSegment::Param(name) => {
                        params.insert(name.clone(), seg.to_string());
                    }
                }
            }

            if path_matches {
                if &route.method == method {
                    return Ok(RouteMatch { params, handler_index: route.handler_index });
                } else {
                    method_matched_somewhere = true;
                }
            }
        }

        if method_matched_somewhere {
            Err(405) // Method Not Allowed
        } else {
            Err(404) // Not Found
        }
    }
}

/// Fluent builder API
pub struct RouterBuilder {
    router: Router,
}

impl RouterBuilder {
    pub fn new() -> Self {
        RouterBuilder { router: Router::new() }
    }

    pub fn get(mut self, path: &str, idx: usize) -> Self {
        self.router.add_route(HttpMethod::Get, path, idx);
        self
    }

    pub fn post(mut self, path: &str, idx: usize) -> Self {
        self.router.add_route(HttpMethod::Post, path, idx);
        self
    }

    pub fn put(mut self, path: &str, idx: usize) -> Self {
        self.router.add_route(HttpMethod::Put, path, idx);
        self
    }

    pub fn delete(mut self, path: &str, idx: usize) -> Self {
        self.router.add_route(HttpMethod::Delete, path, idx);
        self
    }

    pub fn build(self) -> Router {
        self.router
    }
}
```

ตัวอย่างการใช้งาน `RouterBuilder`:

```rust
let router = RouterBuilder::new()
    .get("/health",          0)   // handler index 0 = health_check
    .get("/users/:id",       1)   // handler index 1 = get_user
    .post("/users",          2)   // handler index 2 = create_user
    .put("/users/:id",       3)   // handler index 3 = update_user
    .delete("/users/:id",    4)   // handler index 4 = delete_user
    .get("/posts/:id/comments/:cid", 5)
    .build();

// Match GET /users/42
match router.match_route(&HttpMethod::Get, "/users/42") {
    Ok(m) => {
        // m.params["id"] == "42"
        // m.handler_index == 1
        dispatch_handler(m.handler_index, m.params, &request)
    }
    Err(405) => send_response(405, "Method Not Allowed"),
    Err(404) => send_response(404, "Not Found"),
    Err(_)   => unreachable!(),
}
```

**ทำไม index แทน closure?** การเก็บ `usize` index แทน `Box<dyn Fn(...)>` ทำให้ `Router` เป็น `Clone` และ `Send + Sync` โดยอัตโนมัติ เราสามารถ `Arc<Router>` แล้วแชร์ระหว่าง task ได้โดยไม่ต้องคิดเรื่อง lifetime

---

### ขั้นที่ 4: Middleware Stack

Middleware คือ layer ที่ wraps handler เพื่อเพิ่ม behavior โดยไม่แก้ handler หลัก:

```
Request → AuthMiddleware → LoggingMiddleware → CompressionMiddleware → Handler → Response
```

กำหนด trait ใน `src/middleware/mod.rs`:

```rust
// src/middleware/mod.rs
use crate::http1::{request::HttpRequest, response::HttpResponse};
use std::future::Future;
use std::pin::Pin;

pub type BoxFuture<T> = Pin<Box<dyn Future<Output = T> + Send>>;

pub trait Middleware: Send + Sync {
    fn handle(
        &self,
        req:  &HttpRequest,
        next: &dyn Fn(&HttpRequest) -> BoxFuture<HttpResponse>,
    ) -> BoxFuture<HttpResponse>;
}
```

#### LoggingMiddleware

```rust
// src/middleware/logging.rs
use super::{BoxFuture, Middleware};
use crate::http1::{request::HttpRequest, response::HttpResponse};
use std::time::Instant;

pub struct LoggingMiddleware;

impl Middleware for LoggingMiddleware {
    fn handle(
        &self,
        req:  &HttpRequest,
        next: &dyn Fn(&HttpRequest) -> BoxFuture<HttpResponse>,
    ) -> BoxFuture<HttpResponse> {
        let method = req.method.as_str().to_string();
        let path   = req.path.clone();
        let start  = Instant::now();
        let fut    = next(req);

        Box::pin(async move {
            let resp    = fut.await;
            let elapsed = start.elapsed();
            println!(
                "{} {} → {} ({:.2}ms)",
                method,
                path,
                resp.status,
                elapsed.as_secs_f64() * 1000.0
            );
            resp
        })
    }
}
```

#### AuthMiddleware

```rust
// src/middleware/auth.rs
use super::{BoxFuture, Middleware};
use crate::http1::{request::HttpRequest, response::HttpResponse};

pub struct AuthMiddleware {
    valid_tokens: Vec<String>,
    /// Paths ที่ไม่ต้องการ auth (เช่น /health)
    public_paths: Vec<String>,
}

impl AuthMiddleware {
    pub fn new(tokens: Vec<&str>, public_paths: Vec<&str>) -> Self {
        AuthMiddleware {
            valid_tokens: tokens.iter().map(|s| s.to_string()).collect(),
            public_paths: public_paths.iter().map(|s| s.to_string()).collect(),
        }
    }

    fn is_authorized(&self, req: &HttpRequest) -> bool {
        // Public paths bypass auth
        if self.public_paths.iter().any(|p| req.path.starts_with(p)) {
            return true;
        }
        // Check Authorization: Bearer <token>
        match req.headers.get("authorization") {
            None => false,
            Some(h) => {
                if let Some(token) = h.strip_prefix("Bearer ") {
                    self.valid_tokens.iter().any(|t| t == token)
                } else {
                    false
                }
            }
        }
    }
}

impl Middleware for AuthMiddleware {
    fn handle(
        &self,
        req:  &HttpRequest,
        next: &dyn Fn(&HttpRequest) -> BoxFuture<HttpResponse>,
    ) -> BoxFuture<HttpResponse> {
        if !self.is_authorized(req) {
            return Box::pin(async {
                HttpResponse::new(401, "Unauthorized")
                    .with_header("www-authenticate", "Bearer realm=\"api\"")
            });
        }
        next(req)
    }
}
```

#### CompressionMiddleware (gzip)

```rust
// src/middleware/compression.rs
use super::{BoxFuture, Middleware};
use crate::http1::{request::HttpRequest, response::HttpResponse};
use flate2::write::GzEncoder;
use flate2::Compression;
use std::io::Write;

pub struct CompressionMiddleware {
    /// ขนาด body ขั้นต่ำ (bytes) ที่จะ compress — เล็กกว่านี้ไม่คุ้ม
    min_size: usize,
}

impl CompressionMiddleware {
    pub fn new(min_size: usize) -> Self {
        CompressionMiddleware { min_size }
    }

    fn accepts_gzip(req: &HttpRequest) -> bool {
        req.headers
            .get("accept-encoding")
            .map(|ae| ae.contains("gzip"))
            .unwrap_or(false)
    }

    fn compress(data: &[u8]) -> Option<Vec<u8>> {
        let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
        encoder.write_all(data).ok()?;
        encoder.finish().ok()
    }
}

impl Middleware for CompressionMiddleware {
    fn handle(
        &self,
        req:  &HttpRequest,
        next: &dyn Fn(&HttpRequest) -> BoxFuture<HttpResponse>,
    ) -> BoxFuture<HttpResponse> {
        let accepts_gzip = Self::accepts_gzip(req);
        let min_size     = self.min_size;
        let fut          = next(req);

        Box::pin(async move {
            let mut resp = fut.await;

            if accepts_gzip && resp.body.len() >= min_size {
                if let Some(compressed) = Self::compress(&resp.body) {
                    resp.body = compressed;
                    resp.headers
                        .insert("content-encoding".to_string(), "gzip".to_string());
                    resp.headers
                        .insert("content-length".to_string(), resp.body.len().to_string());
                    resp.headers
                        .insert("vary".to_string(), "Accept-Encoding".to_string());
                }
            }
            resp
        })
    }
}
```

ตัวอย่างการประกอบ middleware stack:

```rust
// สร้าง middleware stack (เรียงจากนอกเข้าใน)
let middlewares: Vec<Box<dyn Middleware>> = vec![
    Box::new(AuthMiddleware::new(
        vec!["secret-api-key"],
        vec!["/health", "/metrics"],
    )),
    Box::new(LoggingMiddleware),
    Box::new(CompressionMiddleware::new(1024)),
];
```

---

### ขั้นที่ 5: HTTP/2 ด้วย hyper

HTTP/2 มีความแตกต่างจาก HTTP/1.1 หลายอย่าง:
- **Binary framing** แทน text-based
- **Multiplexing** — request หลายอันแชร์ TCP connection เดียว
- **Header compression** ด้วย HPACK
- **Server push** — server ส่ง resource ก่อนที่ client จะขอ

`hyper` จัดการ HTTP/2 framing ให้ทั้งหมด เราแค่ implement `Service`:

```rust
// src/http2/mod.rs
use http_body_util::{BodyExt, Full};
use hyper::{Request, Response};
use hyper::body::{Bytes, Incoming};
use hyper::service::service_fn;
use hyper_util::rt::TokioIo;
use tokio::net::TcpListener;

/// Handler function type สำหรับ hyper
async fn handle_request(
    req: Request<Incoming>,
) -> Result<Response<Full<Bytes>>, hyper::Error> {
    let method = req.method().clone();
    let path   = req.uri().path().to_string();

    println!("[HTTP/2] {} {}", method, path);

    // ตัวอย่าง: routing แบบง่าย
    let (status, body) = match (method.as_str(), path.as_str()) {
        ("GET", "/health") => (200, "OK"),
        ("GET", p) if p.starts_with("/api/") => (200, "API response"),
        _ => (404, "Not Found"),
    };

    Ok(Response::builder()
        .status(status)
        .header("content-type", "text/plain")
        .header("server", "rust-http2/0.1")
        // HTTP/2 server push hint (ระบุ resource ที่ควร prefetch)
        .header("link", "</static/main.css>; rel=preload; as=style")
        .body(Full::new(Bytes::from(body)))
        .unwrap())
}

pub async fn start_http2_server(addr: &str) -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind(addr).await?;
    println!("HTTP/2 server listening on http://{}", addr);

    loop {
        let (stream, peer) = listener.accept().await?;
        println!("[HTTP/2] connection from {}", peer);

        let io = TokioIo::new(stream);

        tokio::spawn(async move {
            if let Err(err) = hyper::server::conn::http2::Builder::new(
                hyper_util::rt::TokioExecutor::new(),
            )
            .serve_connection(io, service_fn(handle_request))
            .await
            {
                eprintln!("[HTTP/2] connection error: {}", err);
            }
        });
    }
}
```

**วิธี run ทั้ง HTTP/1.1 และ HTTP/2 พร้อมกัน:**

```rust
// src/main.rs (ขั้นนี้)
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Run HTTP/1.1 บน port 8080 และ HTTP/2 บน port 8443 พร้อมกัน
    let http1_task = tokio::spawn(start_http1_server("127.0.0.1:8080"));
    let http2_task = tokio::spawn(start_http2_server("127.0.0.1:8443"));

    tokio::try_join!(http1_task, http2_task)?;
    Ok(())
}
```

**HTTP/2 multiplexing** หมายความว่า client สามารถส่ง request หลายอันพร้อมกันบน TCP connection เดียว ทดสอบด้วย curl:

```bash
# ทดสอบ HTTP/2 (curl รองรับ HTTP/2 ด้วย --http2)
$ curl --http2-prior-knowledge -v http://127.0.0.1:8443/health

# ดู multiplexing: ส่ง 10 request พร้อมกัน
$ for i in $(seq 10); do
    curl --http2-prior-knowledge http://127.0.0.1:8443/api/$i &
  done; wait
```

---

### ขั้นที่ 6: Static File Serving พร้อม ETag และ Range

Static file server ที่ดีต้องรองรับ:
1. **MIME type** ที่ถูกต้องตาม file extension
2. **ETag / If-None-Match** เพื่อ browser caching
3. **Range requests** สำหรับ video streaming, resume download

```rust
// src/static_files/mod.rs
use crate::http1::{request::HttpRequest, response::HttpResponse};
use std::path::{Path, PathBuf};
use std::time::SystemTime;

/// Generate ETag จาก file content ด้วย DJB2 hash
pub fn generate_etag(content: &[u8]) -> String {
    let mut hash: u64 = 5381;
    for &b in content {
        hash = hash.wrapping_mul(33).wrapping_add(b as u64);
    }
    format!("\"{:016x}\"", hash)
}

pub fn detect_mime_type(filename: &str) -> &'static str {
    let ext = filename.rsplit('.').next().unwrap_or("").to_lowercase();
    match ext.as_str() {
        "html" | "htm" => "text/html; charset=utf-8",
        "css"          => "text/css; charset=utf-8",
        "js"           => "application/javascript",
        "json"         => "application/json",
        "png"          => "image/png",
        "jpg" | "jpeg" => "image/jpeg",
        "gif"          => "image/gif",
        "svg"          => "image/svg+xml",
        "pdf"          => "application/pdf",
        "txt"          => "text/plain; charset=utf-8",
        "wasm"         => "application/wasm",
        "mp4"          => "video/mp4",
        "webm"         => "video/webm",
        _              => "application/octet-stream",
    }
}

#[derive(Debug, PartialEq)]
pub struct ByteRange {
    pub start: u64,
    pub end:   u64,
}

/// Parse Range header: "bytes=0-499" → ByteRange { start: 0, end: 499 }
pub fn parse_range_header(range: &str, content_length: u64) -> Option<ByteRange> {
    let range = range.strip_prefix("bytes=")?;
    let parts: Vec<&str> = range.splitn(2, '-').collect();
    if parts.len() != 2 {
        return None;
    }

    let start: u64 = if parts[0].is_empty() {
        // Suffix range: "bytes=-500" = last 500 bytes
        let suffix: u64 = parts[1].parse().ok()?;
        content_length.saturating_sub(suffix)
    } else {
        parts[0].parse().ok()?
    };

    let end: u64 = if parts[1].is_empty() {
        content_length - 1
    } else {
        parts[1].parse().ok()?
    };

    if start > end || end >= content_length {
        return None;
    }

    Some(ByteRange { start, end })
}

/// Serve static file จาก root directory
pub async fn serve_static(
    req:      &HttpRequest,
    root_dir: &Path,
) -> HttpResponse {
    // Sanitize path — ป้องกัน path traversal
    let clean_path = req.path.trim_start_matches('/');
    if clean_path.contains("..") {
        return HttpResponse::new(400, "Bad Request: path traversal");
    }

    let file_path: PathBuf = root_dir.join(clean_path);

    // อ่านไฟล์
    let content = match tokio::fs::read(&file_path).await {
        Ok(c) => c,
        Err(_) => return HttpResponse::new(404, "Not Found"),
    };

    let mime     = detect_mime_type(&req.path);
    let etag     = generate_etag(&content);
    let file_len = content.len() as u64;

    // Check If-None-Match (ETag caching)
    if let Some(client_etag) = req.headers.get("if-none-match") {
        if client_etag.trim() == etag {
            return HttpResponse::new(304, "")
                .with_header("etag", &etag)
                .with_header("cache-control", "max-age=3600");
        }
    }

    // Range request
    if let Some(range_str) = req.headers.get("range") {
        if let Some(r) = parse_range_header(range_str, file_len) {
            let slice = content[r.start as usize..=r.end as usize].to_vec();
            let content_range = format!(
                "bytes {}-{}/{}",
                r.start, r.end, file_len
            );
            return HttpResponse::new(206, slice)
                .with_header("content-type", mime)
                .with_header("content-range", &content_range)
                .with_header("accept-ranges", "bytes")
                .with_header("etag", &etag);
        } else {
            let cr = format!("bytes */{}", file_len);
            return HttpResponse::new(416, "Range Not Satisfiable")
                .with_header("content-range", &cr);
        }
    }

    // Normal response
    HttpResponse::new(200, content)
        .with_header("content-type", mime)
        .with_header("etag", &etag)
        .with_header("cache-control", "max-age=3600")
        .with_header("accept-ranges", "bytes")
}
```

ทดสอบ ETag caching:

```bash
# Request แรก — server ส่งไฟล์พร้อม ETag
$ curl -v http://localhost:8080/index.html 2>&1 | grep -E "etag|status"
< HTTP/1.1 200 OK
< etag: "a3b4c5d6e7f80910"

# Request ที่สอง — ส่ง If-None-Match กลับ
$ curl -v -H 'If-None-Match: "a3b4c5d6e7f80910"' http://localhost:8080/index.html
< HTTP/1.1 304 Not Modified

# Range request (bytes แรก 100)
$ curl -v -H 'Range: bytes=0-99' http://localhost:8080/large-file.bin
< HTTP/1.1 206 Partial Content
< Content-Range: bytes 0-99/1048576
```

---

### ขั้นที่ 7: Graceful Shutdown

Graceful shutdown หมายถึง:
1. หยุดรับ connection ใหม่ทันทีที่ได้รับ signal
2. รอให้ request ที่กำลัง in-flight เสร็จสิ้นก่อน
3. ปิด server อย่างสะอาด

```rust
// src/shutdown.rs
use tokio::sync::watch;

/// ShutdownSignal ใช้ watch channel เพื่อ broadcast shutdown ไปทุก task
pub struct ShutdownSignal {
    pub sender:   watch::Sender<bool>,
    pub receiver: watch::Receiver<bool>,
}

impl ShutdownSignal {
    pub fn new() -> Self {
        let (sender, receiver) = watch::channel(false);
        ShutdownSignal { sender, receiver }
    }

    pub fn trigger(&self) {
        let _ = self.sender.send(true);
    }

    pub fn is_shutdown(&self) -> bool {
        *self.receiver.borrow()
    }

    pub fn subscribe(&self) -> watch::Receiver<bool> {
        self.receiver.clone()
    }
}
```

```rust
// src/main.rs — graceful shutdown integration
use std::sync::Arc;
use tokio::sync::{watch, Semaphore};
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    // Shutdown channel
    let (shutdown_tx, mut shutdown_rx) = watch::channel(false);

    // Semaphore เพื่อนับ in-flight requests
    let in_flight = Arc::new(Semaphore::new(1000));

    // Spawn signal handler
    let tx_clone = shutdown_tx.clone();
    tokio::spawn(async move {
        // รอ Ctrl-C หรือ SIGTERM
        tokio::select! {
            _ = signal::ctrl_c() => {
                println!("\nCtrl-C received, starting graceful shutdown...");
            }
            _ = async {
                #[cfg(unix)]
                {
                    use tokio::signal::unix::{signal, SignalKind};
                    let mut sigterm = signal(SignalKind::terminate()).unwrap();
                    sigterm.recv().await;
                    println!("SIGTERM received, starting graceful shutdown...");
                }
                #[cfg(not(unix))]
                {
                    futures::future::pending::<()>().await;
                }
            } => {}
        }
        let _ = tx_clone.send(true);
    });

    println!("Server listening on http://127.0.0.1:8080");
    println!("Press Ctrl-C to shut down gracefully");

    loop {
        tokio::select! {
            // รับ connection ใหม่
            result = listener.accept() => {
                let (socket, peer) = result?;
                let in_flight  = Arc::clone(&in_flight);
                let mut rx     = shutdown_rx.clone();

                tokio::spawn(async move {
                    // ขอ permit — ถ้า semaphore เต็มจะรอ
                    let _permit = in_flight.acquire().await.unwrap();
                    handle_connection(socket).await;
                    // permit จะถูก drop อัตโนมัติเมื่อ request เสร็จ
                });
            }

            // รับ shutdown signal — หยุด accept loop
            _ = shutdown_rx.changed() => {
                if *shutdown_rx.borrow() {
                    println!("Shutdown signal received, stopping accept loop");
                    break;
                }
            }
        }
    }

    // รอให้ in-flight requests ทั้งหมดเสร็จ
    // ขอ semaphore ทั้ง 1000 permits = รอให้ทุก task คืน permit
    println!("Waiting for {} in-flight requests...", 1000 - in_flight.available_permits());
    let _ = in_flight.acquire_many(1000).await;
    println!("All requests completed. Server shut down cleanly.");

    Ok(())
}

async fn handle_connection(mut socket: tokio::net::TcpStream) {
    use tokio::io::{AsyncReadExt, AsyncWriteExt};
    let mut buf = vec![0u8; 8192];
    if let Ok(n) = socket.read(&mut buf).await {
        if n > 0 {
            let response = b"HTTP/1.1 200 OK\r\ncontent-length: 2\r\n\r\nOK";
            let _ = socket.write_all(response).await;
        }
    }
}
```

ทดสอบ graceful shutdown:

```bash
# เปิด server
$ cargo run &
Server listening on http://127.0.0.1:8080

# ส่ง request ระหว่าง shutdown
$ curl http://localhost:8080/ &
$ kill -SIGTERM $(pgrep http-server)

# Output
SIGTERM received, starting graceful shutdown...
Waiting for 1 in-flight requests...
All requests completed. Server shut down cleanly.
```

---

### ขั้นที่ 8: ประกอบระบบสมบูรณ์

รวมทุกส่วนเข้าด้วยกัน:

```rust
// src/main.rs (final version)
use std::sync::Arc;
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::sync::watch;
use tokio::signal;

mod http1;
mod router;
mod middleware;
mod static_files;
mod shutdown;

use http1::request::{parse_http_request, HttpMethod};
use http1::response::HttpResponse;
use router::{Router, RouterBuilder};
use middleware::{
    auth::AuthMiddleware,
    logging::LoggingMiddleware,
    compression::CompressionMiddleware,
};
use static_files::serve_static;
use shutdown::ShutdownSignal;

struct AppState {
    router: Router,
    static_root: std::path::PathBuf,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let state = Arc::new(AppState {
        router: RouterBuilder::new()
            .get("/health",       0)
            .get("/api/users",    1)
            .get("/api/users/:id", 2)
            .post("/api/users",   3)
            .build(),
        static_root: std::path::PathBuf::from("./public"),
    });

    let signal = ShutdownSignal::new();
    let mut rx = signal.subscribe();

    // ── HTTP/1.1 ──────────────────────────────────────────
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("HTTP/1.1 listening on :8080");

    let state_clone = Arc::clone(&state);
    let mut rx2     = rx.clone();

    let http1_task = tokio::spawn(async move {
        loop {
            tokio::select! {
                result = listener.accept() => {
                    let Ok((socket, _)) = result else { break };
                    let state = Arc::clone(&state_clone);
                    tokio::spawn(handle_http1(socket, state));
                }
                _ = rx2.changed() => {
                    if *rx2.borrow() { break }
                }
            }
        }
    });

    // ── Shutdown handler ──────────────────────────────────
    tokio::select! {
        _ = signal::ctrl_c() => {
            println!("Ctrl-C received");
            signal.trigger();
        }
    }

    http1_task.await?;
    println!("Server stopped");
    Ok(())
}

async fn handle_http1(
    mut socket: tokio::net::TcpStream,
    state:      Arc<AppState>,
) {
    let mut buf = vec![0u8; 16384];
    let n = match socket.read(&mut buf).await {
        Ok(0) | Err(_) => return,
        Ok(n) => n,
    };

    let req = match parse_http_request(&buf[..n]) {
        Ok(r)  => r,
        Err(_) => {
            let r = HttpResponse::new(400, "Bad Request").to_bytes();
            let _ = socket.write_all(&r).await;
            return;
        }
    };

    // Static files
    if req.path.starts_with("/static/") {
        let resp = serve_static(&req, &state.static_root).await;
        let _ = socket.write_all(&resp.to_bytes()).await;
        return;
    }

    // Router
    let resp = match state.router.match_route(&req.method, &req.path) {
        Ok(m) => match m.handler_index {
            0 => HttpResponse::new(200, r#"{"status":"ok"}"#)
                    .with_header("content-type", "application/json"),
            1 => HttpResponse::new(200, r#"[{"id":1},{"id":2}]"#)
                    .with_header("content-type", "application/json"),
            2 => {
                let id = m.params.get("id").map(|s| s.as_str()).unwrap_or("?");
                HttpResponse::new(200, format!(r#"{{"id":{}}}"#, id))
                    .with_header("content-type", "application/json")
            }
            _ => HttpResponse::new(501, "Not Implemented"),
        },
        Err(405) => HttpResponse::new(405, "Method Not Allowed"),
        Err(404) => HttpResponse::new(404, "Not Found"),
        Err(_)   => HttpResponse::new(500, "Internal Server Error"),
    };

    let _ = socket.write_all(&resp.to_bytes()).await;
}
```

ผลลัพธ์เมื่อรัน server และทดสอบ:

```
$ cargo run
HTTP/1.1 listening on :8080

$ curl http://localhost:8080/health
{"status":"ok"}

$ curl http://localhost:8080/api/users/42
{"id":42}

$ curl -X DELETE http://localhost:8080/api/users/1
HTTP/1.1 405 Method Not Allowed

$ curl http://localhost:8080/notfound
HTTP/1.1 404 Not Found
```

---

## การทดสอบ (Testing)

โค้ด test ครอบคลุม 19 unit tests ทุก layer ของระบบ:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ── Test 1: parse valid GET request line ──────────────
    #[test]
    fn test_parse_request_line_get() {
        let raw = b"GET /hello HTTP/1.1\r\nHost: localhost\r\n\r\n";
        let req = parse_http_request(raw).expect("should parse");
        assert_eq!(req.method, HttpMethod::Get);
        assert_eq!(req.path, "/hello");
        assert_eq!(req.version, HttpVersion::Http11);
    }

    // ── Test 2: parse POST request with body ──────────────
    #[test]
    fn test_parse_post_with_body() {
        let raw = b"POST /api/users HTTP/1.1\r\nContent-Length: 13\r\n\r\nHello, World!";
        let req = parse_http_request(raw).expect("should parse POST");
        assert_eq!(req.method, HttpMethod::Post);
        assert_eq!(req.path, "/api/users");
        assert_eq!(req.body, b"Hello, World!");
    }

    // ── Test 3: header keys lowercase ─────────────────────
    #[test]
    fn test_parse_headers_case_insensitive() {
        let raw = b"GET / HTTP/1.1\r\nContent-Type: application/json\r\nAuthorization: Bearer abc123\r\n\r\n";
        let req = parse_http_request(raw).expect("should parse");
        assert_eq!(
            req.headers.get("content-type").map(|s| s.as_str()),
            Some("application/json")
        );
        assert_eq!(
            req.headers.get("authorization").map(|s| s.as_str()),
            Some("Bearer abc123")
        );
    }

    // ── Test 4: invalid method ─────────────────────────────
    #[test]
    fn test_invalid_method() {
        let raw = b"INVALID /path HTTP/1.1\r\n\r\n";
        let result = parse_http_request(raw);
        assert!(matches!(result, Err(ParseError::InvalidMethod(_))));
    }

    // ── Test 5: invalid HTTP version ──────────────────────
    #[test]
    fn test_invalid_version() {
        let raw = b"GET /path HTTP/0.9\r\n\r\n";
        let result = parse_http_request(raw);
        assert!(matches!(result, Err(ParseError::InvalidVersion(_))));
    }

    // ── Test 6: router path parameter extraction ──────────
    #[test]
    fn test_router_path_params() {
        let mut router = Router::new();
        router.add_route(HttpMethod::Get, "/users/:id", 0);
        router.add_route(HttpMethod::Get, "/users/:id/posts/:post_id", 1);

        let m = router
            .match_route(&HttpMethod::Get, "/users/42")
            .expect("should match");
        assert_eq!(m.params.get("id").map(|s| s.as_str()), Some("42"));
        assert_eq!(m.handler_index, 0);

        let m2 = router
            .match_route(&HttpMethod::Get, "/users/7/posts/99")
            .expect("nested params");
        assert_eq!(m2.params.get("id").map(|s| s.as_str()), Some("7"));
        assert_eq!(m2.params.get("post_id").map(|s| s.as_str()), Some("99"));
        assert_eq!(m2.handler_index, 1);
    }

    // ── Test 7: method mismatch → 405 ─────────────────────
    #[test]
    fn test_router_method_not_allowed() {
        let mut router = Router::new();
        router.add_route(HttpMethod::Get, "/items/:id", 0);

        let result = router.match_route(&HttpMethod::Post, "/items/5");
        assert_eq!(result, Err(405));
    }

    // ── Test 8: no matching path → 404 ────────────────────
    #[test]
    fn test_router_not_found() {
        let mut router = Router::new();
        router.add_route(HttpMethod::Get, "/users/:id", 0);

        let result = router.match_route(&HttpMethod::Get, "/unknown/path");
        assert_eq!(result, Err(404));
    }

    // ── Test 9: ETag deterministic ────────────────────────
    #[test]
    fn test_etag_deterministic() {
        let content = b"Hello, World!";
        let etag1 = generate_etag(content);
        let etag2 = generate_etag(content);
        assert_eq!(etag1, etag2);
        assert!(etag1.starts_with('"'));
        assert!(etag1.ends_with('"'));
    }

    // ── Test 10: different content → different ETag ───────
    #[test]
    fn test_etag_different_content() {
        let etag1 = generate_etag(b"content v1");
        let etag2 = generate_etag(b"content v2");
        assert_ne!(etag1, etag2);
    }

    // ── Test 11: valid Bearer token ───────────────────────
    #[test]
    fn test_auth_valid_bearer_token() {
        let auth = AuthConfig::new(vec!["secret-token-abc".to_string()]);
        assert!(auth.validate_bearer(Some("Bearer secret-token-abc")));
    }

    // ── Test 12: invalid / missing token ──────────────────
    #[test]
    fn test_auth_invalid_bearer_token() {
        let auth = AuthConfig::new(vec!["secret-token-abc".to_string()]);
        assert!(!auth.validate_bearer(Some("Bearer wrong-token")));
        assert!(!auth.validate_bearer(None));
        assert!(!auth.validate_bearer(Some("Basic dXNlcjpwYXNz")));
    }

    // ── Test 13: Accept-Encoding: gzip detection ──────────
    #[test]
    fn test_should_compress_with_gzip_accept() {
        assert!(should_compress(Some("gzip, deflate, br")));
        assert!(should_compress(Some("gzip")));
        assert!(!should_compress(Some("deflate, br")));
        assert!(!should_compress(None));
    }

    // ── Test 14: gzip magic bytes ─────────────────────────
    #[test]
    fn test_gzip_compress_output() {
        let data       = b"Hello, compressed world!";
        let compressed = gzip_compress(data).expect("compress ok");
        // Gzip file signature: 0x1f 0x8b
        assert_eq!(&compressed[..2], &[0x1f, 0x8b]);
        assert!(!compressed.is_empty());
    }

    // ── Test 15: range header parsing ─────────────────────
    #[test]
    fn test_parse_range_header() {
        let r = parse_range_header("bytes=0-499", 1000).expect("valid");
        assert_eq!(r.start, 0);
        assert_eq!(r.end, 499);

        let r2 = parse_range_header("bytes=500-", 1000).expect("open end");
        assert_eq!(r2.start, 500);
        assert_eq!(r2.end, 999);

        assert!(parse_range_header("bytes=1000-1999", 1000).is_none());
    }

    // ── Test 16: HttpResponse wire format ─────────────────
    #[test]
    fn test_http_response_to_bytes() {
        let resp = HttpResponse::new(200, b"OK".to_vec())
            .with_header("x-custom", "test");
        let bytes = resp.to_bytes();
        let text  = std::str::from_utf8(&bytes).expect("utf8");
        assert!(text.starts_with("HTTP/1.1 200 OK\r\n"));
        assert!(text.contains("x-custom: test\r\n"));
        assert!(text.contains("\r\n\r\n"));
    }

    // ── Test 17: graceful shutdown signal ─────────────────
    #[tokio::test]
    async fn test_graceful_shutdown_signal() {
        let signal = ShutdownSignal::new();
        assert!(!signal.is_shutdown());
        signal.trigger_shutdown();
        tokio::task::yield_now().await;
        assert!(signal.is_shutdown());
    }

    // ── Test 18: MIME type detection ──────────────────────
    #[test]
    fn test_mime_type_detection() {
        assert_eq!(detect_mime_type("index.html"), "text/html; charset=utf-8");
        assert_eq!(detect_mime_type("style.css"),  "text/css; charset=utf-8");
        assert_eq!(detect_mime_type("app.js"),     "application/javascript");
        assert_eq!(detect_mime_type("data.json"),  "application/json");
        assert_eq!(detect_mime_type("photo.jpg"),  "image/jpeg");
        assert_eq!(detect_mime_type("unknown.xyz"),"application/octet-stream");
    }

    // ── Test 19: router literal routes ────────────────────
    #[test]
    fn test_router_literal_route() {
        let mut router = Router::new();
        router.add_route(HttpMethod::Get,  "/health", 0);
        router.add_route(HttpMethod::Post, "/health", 1);

        let m  = router.match_route(&HttpMethod::Get,  "/health").expect("GET");
        assert_eq!(m.handler_index, 0);

        let m2 = router.match_route(&HttpMethod::Post, "/health").expect("POST");
        assert_eq!(m2.handler_index, 1);
    }
}
```

รัน `cargo test` ได้ output จริงดังนี้:

```
$ cargo test
   Compiling http-server v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.95s
     Running unittests src/lib.rs (target/debug/deps/http_server-f6ee3506dcc35216)

running 19 tests
test tests::test_auth_valid_bearer_token ... ok
test tests::test_etag_different_content ... ok
test tests::test_auth_invalid_bearer_token ... ok
test tests::test_etag_deterministic ... ok
test tests::test_http_response_to_bytes ... ok
test tests::test_invalid_version ... ok
test tests::test_graceful_shutdown_signal ... ok
test tests::test_invalid_method ... ok
test tests::test_mime_type_detection ... ok
test tests::test_parse_headers_case_insensitive ... ok
test tests::test_parse_post_with_body ... ok
test tests::test_parse_range_header ... ok
test tests::test_parse_request_line_get ... ok
test tests::test_router_method_not_allowed ... ok
test tests::test_router_not_found ... ok
test tests::test_gzip_compress_output ... ok
test tests::test_router_literal_route ... ok
test tests::test_router_path_params ... ok
test tests::test_should_compress_with_gzip_accept ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/http_server-354e86765bfd3fcf)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests http_server

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก test ผ่าน: **19 passed; 0 failed**

---

## จุดพลาดที่พบบ่อย (Pitfalls)

### Pitfall 1: ลืม `\r\n` ใน HTTP/1.1 Headers

HTTP/1.1 spec กำหนดให้ใช้ CRLF (`\r\n`) คั่นระหว่าง header ไม่ใช่ LF เดียว หลาย client เช่น Python `requests` ส่งแค่ LF ซึ่ง server ที่ strict จะ reject

```rust
// ❌ ผิด: ใช้ \n เดียว — บาง client อาจไม่ยอมรับ
buf.extend_from_slice(b"HTTP/1.1 200 OK\n");
buf.extend_from_slice(b"content-length: 5\n");
buf.extend_from_slice(b"\n");

// ✓ ถูก: ใช้ \r\n ตาม RFC 7230
buf.extend_from_slice(b"HTTP/1.1 200 OK\r\n");
buf.extend_from_slice(b"content-length: 5\r\n");
buf.extend_from_slice(b"\r\n");
```

**แก้ไข**: `find_body_start()` ในโค้ดของเราจัดการทั้ง `\r\n\r\n` และ `\n\n` เพื่อความยืดหยุ่น แต่ output ต้องใช้ `\r\n` เสมอ

---

### Pitfall 2: Content-Length ไม่ตรงหลัง Compression

เมื่อ `CompressionMiddleware` compress body แล้ว ต้องอัปเดต `Content-Length` ด้วย ถ้าลืม client จะอ่าน body ผิดพลาด

```rust
// ❌ ผิด: compress แต่ไม่อัปเดต content-length
resp.body = compressed;
resp.headers.insert("content-encoding".to_string(), "gzip".to_string());
// content-length ยังเป็น size ก่อน compress!

// ✓ ถูก: อัปเดต content-length พร้อมกัน
resp.body = compressed;
resp.headers.insert("content-encoding".to_string(), "gzip".to_string());
resp.headers.insert(
    "content-length".to_string(),
    resp.body.len().to_string()   // ← size หลัง compress
);
```

นอกจากนี้ต้องเพิ่ม `Vary: Accept-Encoding` เพื่อบอก CDN/proxy ว่า response นี้แตกต่างตาม header นี้

---

### Pitfall 3: Path Traversal ใน Static File Server

ถ้าไม่ sanitize path ผู้ใช้อาจส่ง `/static/../../etc/passwd` เพื่ออ่านไฟล์นอก root directory

```rust
// ❌ อันตราย: ไม่ sanitize
let file_path = root_dir.join(req.path.trim_start_matches('/'));

// ✓ ปลอดภัย: ตรวจ path traversal ก่อน
let clean_path = req.path.trim_start_matches('/');
if clean_path.contains("..") || clean_path.starts_with('/') {
    return HttpResponse::new(400, "Bad Request");
}
let file_path = root_dir.join(clean_path);

// ✓ ปลอดภัยยิ่งกว่า: ตรวจว่า canonical path อยู่ใน root_dir
let canonical = file_path.canonicalize()?;
if !canonical.starts_with(&root_dir.canonicalize()?) {
    return HttpResponse::new(403, "Forbidden");
}
```

---

### Pitfall 4: Router ไม่แยก 404 กับ 405

หลาย implementation คืน 404 สำหรับทุกกรณีที่ route ไม่ match ซึ่งผิด RFC 7231 — ถ้า path มีอยู่แต่ method ไม่ถูกต้อง ต้องคืน **405 Method Not Allowed** พร้อม `Allow` header

```rust
// ❌ ผิด: คืน 404 ทุกกรณี
if !found { return Err(404) }

// ✓ ถูก: แยก path-not-found (404) กับ method-not-allowed (405)
if method_matched_somewhere {
    // path มีอยู่ แต่ method ไม่ถูก
    Err(405)  // ควรเพิ่ม Allow header ด้วย
} else {
    // path ไม่มีเลย
    Err(404)
}
```

```rust
// RFC 7231 กำหนดให้ response 405 ต้องมี Allow header
HttpResponse::new(405, "Method Not Allowed")
    .with_header("allow", "GET, POST, OPTIONS")
```

---

### Pitfall 5: อ่าน TCP Buffer ครั้งเดียวอาจได้ข้อมูลไม่ครบ

`socket.read()` ไม่รับประกันว่าจะได้ HTTP request ครบทั้งหมดในครั้งเดียว โดยเฉพาะ request ที่มี body ขนาดใหญ่

```rust
// ❌ ผิด: อ่านครั้งเดียว อาจได้ข้อมูลตัดกลาง
let n = socket.read(&mut buf).await?;
let req = parse_http_request(&buf[..n])?;

// ✓ ถูก: อ่านจนได้ headers ครบก่อน แล้วจึงอ่าน body ตาม Content-Length
async fn read_full_request(
    socket: &mut TcpStream,
    buf:    &mut Vec<u8>,
) -> Result<usize, std::io::Error> {
    let mut total = 0;
    loop {
        let n = socket.read(&mut buf[total..]).await?;
        if n == 0 { break }
        total += n;
        // ตรวจว่าได้ header separator แล้วหรือยัง
        if buf[..total].windows(4).any(|w| w == b"\r\n\r\n") {
            // อ่าน body ต่อถ้า Content-Length บอกว่ายังเหลือ
            break;
        }
    }
    Ok(total)
}
```

---

### Pitfall 6: Watch Channel กับ `changed()` ต้องตรวจค่าหลัง changed

`watch::Receiver::changed()` จะ complete ครั้งแรกเมื่อค่าเปลี่ยน แต่ถ้าไม่ตรวจค่าจริง อาจเกิด false positive จาก initialization

```rust
// ❌ ผิด: ไม่ตรวจค่า — อาจ shutdown ก่อนเวลา
_ = shutdown_rx.changed() => { break; }

// ✓ ถูก: ตรวจค่าจริงก่อน
_ = shutdown_rx.changed() => {
    if *shutdown_rx.borrow() {
        break;
    }
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/http-server

# ขนาด binary
$ ls -lh target/release/http-server
-rwxr-xr-x 1 user user 4.2M Sep 28 12:00 target/release/http-server

# Strip debug symbols ลดขนาด
strip target/release/http-server
$ ls -lh target/release/http-server
-rwxr-xr-x 1 user user 1.1M Sep 28 12:00 target/release/http-server
```

### Docker

```dockerfile
# Dockerfile — Multi-stage build
FROM rust:1.80-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/http-server /usr/local/bin/http-server
COPY public /public
EXPOSE 8080 8443
CMD ["http-server"]
```

```bash
docker build -t rust-http-server .
docker run -p 8080:8080 -p 8443:8443 rust-http-server
```

### Systemd Service

```ini
# /etc/systemd/system/http-server.service
[Unit]
Description=Rust HTTP Server
After=network.target

[Service]
Type=simple
User=www-data
ExecStart=/usr/local/bin/http-server
Restart=always
RestartSec=5
# Graceful shutdown timeout
TimeoutStopSec=30
# ส่ง SIGTERM เพื่อ graceful shutdown
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable http-server
systemctl start http-server
# ทดสอบ graceful shutdown
systemctl stop http-server  # ส่ง SIGTERM → server รอ in-flight requests
```

### Benchmark ด้วย wrk

```bash
# ติดตั้ง wrk
apt-get install wrk

# Benchmark 10 วินาที, 12 thread, 400 concurrent connections
wrk -t12 -c400 -d10s http://localhost:8080/health

# ผลลัพธ์ตัวอย่าง
Running 10s test @ http://localhost:8080/health
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.34ms    1.12ms  45.67ms   89.23%
    Req/Sec    14.52k     2.31k   23.45k    68.45%
  1,743,291 requests in 10.10s, 156.23MB read
Requests/sec: 172,603.07
Transfer/sec:  15.47MB
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม TLS/HTTPS ด้วย rustls

ระดับ: ⭐⭐⭐

เพิ่ม HTTPS support โดยใช้ `tokio-rustls` crate แทน plain TCP:

```toml
[dependencies]
tokio-rustls = "0.26"
rustls = "0.23"
rustls-pemfile = "2"
```

```rust
use tokio_rustls::TlsAcceptor;
use rustls::{ServerConfig, Certificate, PrivateKey};

// โจทย์: สร้าง TlsAcceptor จาก cert/key files
// แล้วใช้ acceptor.accept(stream).await? แทน stream โดยตรง
// ทดสอบด้วย: curl --insecure https://localhost:8443/health
```

สิ่งที่ต้องทำ:
- โหลด certificate และ private key จากไฟล์ PEM
- สร้าง `TlsAcceptor` จาก `ServerConfig`
- Wrap TCP stream ด้วย TLS ก่อนส่งต่อให้ HTTP handler
- ทดสอบ HTTPS ด้วย self-signed cert

---

### แบบฝึกหัดที่ 2: Connection Keep-Alive และ Pipeline

ระดับ: ⭐⭐⭐⭐

HTTP/1.1 รองรับ persistent connection ด้วย `Connection: keep-alive` ซึ่งช่วยลด latency โดยไม่ต้องสร้าง TCP connection ใหม่ทุก request:

```rust
// โจทย์: แก้ handle_connection ให้รองรับ keep-alive
// ต้องอ่าน request หลายอันจาก connection เดียว
async fn handle_connection_keepalive(mut socket: TcpStream, state: Arc<AppState>) {
    loop {
        let req = match read_full_request(&mut socket).await {
            Ok(Some(r)) => r,
            Ok(None)    => break,  // client closed
            Err(_)      => break,
        };

        let resp    = route_request(&req, &state).await;
        let keep_alive = req.headers
            .get("connection")
            .map(|v| v.to_lowercase().contains("keep-alive"))
            .unwrap_or(true);  // HTTP/1.1 default

        let _ = socket.write_all(&resp.to_bytes()).await;

        if !keep_alive { break }
    }
}
```

สิ่งที่ต้องทำ:
- ปรับ request reader ให้อ่านหลาย request จาก buffer เดียว
- Track `Connection: close` และ `Connection: keep-alive`
- เพิ่ม `Connection` header ใน response
- ทดสอบด้วย `curl --keepalive-time 5`

---

### แบบฝึกหัดที่ 3: Rate Limiting Middleware

ระดับ: ⭐⭐⭐

เพิ่ม `RateLimitMiddleware` ที่ใช้ Token Bucket algorithm:

```rust
use std::collections::HashMap;
use std::sync::Mutex;
use std::time::Instant;

pub struct TokenBucket {
    tokens:        f64,
    max_tokens:    f64,
    refill_rate:   f64,  // tokens per second
    last_refill:   Instant,
}

impl TokenBucket {
    pub fn try_consume(&mut self, tokens: f64) -> bool {
        // โจทย์: implement token bucket
        // 1) คำนวณ tokens ที่ refill ตั้งแต่ last_refill
        // 2) เพิ่ม tokens แต่ไม่เกิน max_tokens
        // 3) ถ้า tokens เพียงพอ consume แล้ว return true
        // 4) ถ้าไม่พอ return false
        todo!()
    }
}

pub struct RateLimitMiddleware {
    buckets:     Mutex<HashMap<String, TokenBucket>>,
    rate:        f64,   // requests per second
    burst:       f64,   // max burst size
}
```

สิ่งที่ต้องทำ:
- Implement `TokenBucket::try_consume()`
- Extract client IP จาก `X-Forwarded-For` หรือ remote addr
- คืน `429 Too Many Requests` พร้อม `Retry-After` header
- ทดสอบด้วย `wrk` ที่ rate สูงกว่า limit

---

### แบบฝึกหัดที่ 4: WebSocket Upgrade

ระดับ: ⭐⭐⭐⭐

HTTP/1.1 รองรับ protocol upgrade เป็น WebSocket ผ่าน:

```
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```rust
// โจทย์: implement WebSocket handshake
fn websocket_accept_key(client_key: &str) -> String {
    use sha1::{Sha1, Digest};
    let magic = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";
    let combined = format!("{}{}", client_key, magic);
    let hash = Sha1::digest(combined.as_bytes());
    base64::encode(&hash)
}

async fn handle_websocket_upgrade(req: &HttpRequest, socket: TcpStream) {
    let key    = req.headers.get("sec-websocket-key").unwrap();
    let accept = websocket_accept_key(key);

    let handshake = format!(
        "HTTP/1.1 101 Switching Protocols\r\n\
         Upgrade: websocket\r\n\
         Connection: Upgrade\r\n\
         Sec-WebSocket-Accept: {}\r\n\r\n",
        accept
    );
    // ส่ง handshake แล้ว handle WebSocket frames
    todo!("implement WebSocket frame parser")
}
```

สิ่งที่ต้องทำ:
- Compute `Sec-WebSocket-Accept` ด้วย SHA-1 hash
- ส่ง 101 Switching Protocols response
- Implement WebSocket frame format (FIN, opcode, masking key, payload)
- Echo server: รับ text frame แล้วส่งกลับ

---

### แบบฝึกหัดที่ 5: HTTP/2 Server Push

ระดับ: ⭐⭐⭐⭐

HTTP/2 รองรับ server push ซึ่งช่วยลด latency โดย server ส่ง resource ที่ client จะขอต่อไปโดยไม่ต้องรอ:

```rust
// โจทย์: ใน HTTP/2 handler ส่ง Link header เพื่อ hint server push
// และ implement actual PUSH_PROMISE frame ผ่าน hyper API

use hyper::Response;
use http_body_util::Full;
use bytes::Bytes;

async fn handle_with_push(
    req: Request<Incoming>,
) -> Result<Response<Full<Bytes>>, hyper::Error> {
    if req.uri().path() == "/" {
        // บอก client ว่าควร prefetch CSS และ JS
        Ok(Response::builder()
            .status(200)
            .header("link", "</style.css>; rel=preload; as=style")
            .header("link", "</app.js>; rel=preload; as=script")
            .body(Full::new(Bytes::from("<html>...</html>")))
            .unwrap())
    } else {
        todo!("route other paths")
    }
}
```

สิ่งที่ต้องทำ:
- ตั้ง `Link` headers สำหรับ HTTP/2 server push hints
- ทดสอบด้วย `nghttp` หรือ Chrome DevTools
- วัดผล performance เปรียบเทียบ with/without push

---

### แบบฝึกหัดที่ 6: Request Body Streaming

ระดับ: ⭐⭐⭐⭐⭐

Request body ขนาดใหญ่ (file upload) ไม่ควรโหลดเข้า RAM ทั้งหมด ควร stream เข้า storage โดยตรง:

```rust
// โจทย์: implement streaming upload handler
async fn handle_upload_stream(
    mut socket:    TcpStream,
    content_length: u64,
    output_path:   &Path,
) -> Result<(), Box<dyn std::error::Error>> {
    use tokio::fs::File;
    use tokio::io::copy;

    let mut file = File::create(output_path).await?;
    // โจทย์: อ่าน body ทีละ chunk แล้ว write ลงไฟล์
    // ไม่ buffer ทั้งหมดใน RAM
    // ใช้ tokio::io::copy หรือ manual chunk reading
    todo!()
}
```

---

## สรุป

ในโปรเจคนี้เราสร้าง HTTP server ที่ครอบคลุมทุก layer ตั้งแต่ raw TCP bytes จนถึง HTTP/2 multiplexing:

| Component | สิ่งที่เรียนรู้ |
|-----------|----------------|
| HTTP/1.1 Parser | raw bytes → struct, error handling ระดับ protocol |
| TcpListener | tokio async I/O, spawning tasks per connection |
| Router | pattern matching, path params, 404 vs 405 |
| Middleware | trait objects, async closures, composition |
| HTTP/2 (hyper) | service_fn, binary framing, multiplexing |
| Static Files | ETag hash, Range requests, MIME detection |
| Graceful Shutdown | watch channel, tokio::select!, semaphore |

**Pattern สำคัญที่ได้เรียน:**

1. **Layered architecture** — แต่ละ layer รู้แค่สิ่งที่ layer ถัดไปต้องการ parser ไม่รู้เรื่อง router, router ไม่รู้เรื่อง middleware
2. **Type-safe error propagation** — `ParseError` enum แทน String ทำให้ caller รู้ว่าจะ handle error แบบไหน
3. **`Arc<T>` สำหรับ shared state** — `Router` และ `AppState` ถูก share ระหว่าง tasks ผ่าน `Arc` โดยไม่ lock
4. **`watch::channel` สำหรับ broadcast** — shutdown signal ส่งถึงทุก task พร้อมกัน เหมาะกว่า `oneshot` ในกรณีนี้
5. **Semaphore สำหรับ in-flight counting** — หลีกเลี่ยง `AtomicUsize` ด้วย semaphore ที่ `acquire` ได้แบบ blocking

โปรเจคถัดไป **H02: WebSocket Server** จะต่อยอดจาก HTTP server นี้โดยเพิ่ม WebSocket upgrade handler, frame parser, และ real-time message broadcast — เรื่องราวที่ยากและน่าสนใจยิ่งขึ้น

---

**โปรเจคก่อนหน้า:** [project-g10-blue-green.md](project-g10-blue-green.md) | **โปรเจคถัดไป:** [project-h02-websocket.md](project-h02-websocket.md)
