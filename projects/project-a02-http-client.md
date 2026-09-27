# Project A02: HTTP Client (curl-like)

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 4 ชั่วโมง

## ภาพรวมโปรเจค

เราจะสร้าง HTTP client แบบ command-line ที่ทำงานคล้าย `curl` แต่เขียนด้วย Rust ชื่อ **httpcli** โปรเจคนี้ครอบคลุมการส่ง HTTP request ทุก method (GET, POST, PUT, DELETE, PATCH), ตั้งค่า headers, ส่ง body, แสดงผล response อย่างสวยงาม, และฟีเจอร์ขั้นสูงอย่าง retry logic, progress bar, และ config file

**ทำไมถึงน่าสร้าง?** HTTP client เป็นเครื่องมือที่ developer ใช้ทุกวัน การเขียนเองทำให้เข้าใจกลไกของ HTTP อย่างลึกซึ้ง และได้ฝึกทักษะที่สำคัญมากในโลก production Rust เช่น การจัดการ error ในระบบ networked, การเขียน CLI ที่ใช้งานได้จริง, และการอ่าน/เขียน config file แบบ TOML

**Use case ในโลกจริง:**
- สคริปต์ทดสอบ API endpoint ใน CI/CD pipeline
- เครื่องมือ monitoring ที่ poll health check endpoint
- Utility สำหรับ download file พร้อม progress bar
- เครื่องมือ debug HTTP request/response โดยละเอียด

## สิ่งที่จะได้เรียนรู้

- การสร้าง CLI ที่ซับซ้อนด้วย `clap` (derive API, multiple args, flags, options)
- การส่ง HTTP request ด้วย `reqwest` ทั้งแบบ blocking และ async
- การจัดการ HTTP headers, authentication, และ redirect
- การ detect content-type และ pretty-print JSON ด้วย `serde_json`
- การแสดง progress bar ด้วย `indicatif` สำหรับ download ไฟล์ขนาดใหญ่
- Retry logic พร้อม exponential backoff สำหรับ transient error
- การอ่าน config file แบบ TOML ด้วย `dirs` และ `toml`
- Integration testing ด้วย `mockito` — mock HTTP server จริง

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 1–20: ownership, borrowing, structs, enums, error handling
- จาก Part 41–50: traits, generics, closures, iterators
- จาก Part 51–60: async/await พื้นฐาน (สำหรับทำความเข้าใจ reqwest)
- จาก Part 41: การใช้ `Result<T, E>` และ `?` operator อย่างลึกซึ้ง
- ความรู้พื้นฐานเรื่อง HTTP protocol (method, headers, status code, body)

## โครงสร้างโปรเจค (Project Layout)

```
httpcli/
├── src/
│   ├── main.rs          ← entry point + CLI parsing
│   ├── client.rs        ← HTTP client logic
│   ├── config.rs        ← config file loading
│   ├── output.rs        ← response formatting
│   └── retry.rs         ← retry with backoff
├── tests/
│   └── integration_test.rs  ← integration tests ด้วย mockito
├── Cargo.toml
└── README.md
```

สำหรับโปรเจคนี้เราจะเริ่มจาก single-file (`main.rs`) แล้วค่อย refactor เป็น module ต่าง ๆ ใน Extension exercise

## การออกแบบ (Architecture & Design)

```
┌─────────────────────────────────────────────────────────────┐
│  main()                                                      │
│   │                                                          │
│   ├─ Cli::parse()          ← clap parses argv               │
│   ├─ load_config()         ← อ่าน ~/.config/httpcli/config.toml│
│   ├─ build_client()        ← reqwest::ClientBuilder         │
│   ├─ build_headers()       ← merge config + CLI headers     │
│   │                                                          │
│   └─ for each URL:                                           │
│       ├─ execute_with_retry()  ← retry loop                 │
│       │   └─ execute_request() ← reqwest blocking call      │
│       └─ handle_response()    ← format + output             │
│           ├─ JSON mode:  serde_json::to_string_pretty()     │
│           ├─ file mode:  fs::write + indicatif progress     │
│           └─ pretty mode: detect content-type, colorize     │
└─────────────────────────────────────────────────────────────┘
```

**Design decisions:**

1. **Blocking vs Async** — เราใช้ `reqwest::blocking` เพราะ CLI tool ไม่ต้องการ concurrency สูง และ blocking API ง่ายกว่า async สำหรับการ learn โดยไม่เสียประสิทธิภาพที่จำเป็น

2. **Error handling** — ใช้ `Box<dyn Error>` ใน main เพื่อความยืดหยุ่น แต่ internal function ใช้ `Result<T, String>` หรือ `Result<T, reqwest::Error>` ตามความเหมาะสม

3. **Config merging** — Config file เป็น default, CLI args override ทั้งหมด ลำดับความสำคัญ: CLI > Environment > Config file

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: CLI Skeleton และ GET Request ง่าย ๆ

เริ่มด้วย Cargo.toml ที่มี dependency ทั้งหมดก่อน:

```toml
[package]
name = "httpcli"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "httpcli"
path = "src/main.rs"

[dependencies]
reqwest = { version = "0.12", features = ["blocking", "json", "cookies"] }
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
indicatif = "0.17"
toml = "0.8"
dirs = "5"
colored = "2"

[dev-dependencies]
mockito = "1"
```

**ทำไมถึงใช้ `features = ["blocking"]`?** reqwest 0.12 เป็น async-first คือ default API ใช้ `async fn` ต้องอยู่ใน tokio runtime ถ้าเราต้องการใช้ blocking (synchronous) ต้องเปิด feature นี้ไว้อย่างชัดเจน

```rust
// src/main.rs — ขั้นที่ 1: CLI skeleton + simple GET
use clap::Parser;

#[derive(Parser, Debug)]
#[command(
    name = "httpcli",
    version = "0.1.0",
    about = "A curl-like HTTP client written in Rust"
)]
struct Cli {
    /// URL(s) to request (รองรับหลาย URL)
    urls: Vec<String>,

    /// HTTP method (GET, POST, PUT, DELETE, PATCH)
    #[arg(short = 'X', long, default_value = "GET")]
    method: String,
}

fn main() {
    let cli = Cli::parse();

    if cli.urls.is_empty() {
        eprintln!("Error: no URL provided");
        eprintln!("Usage: httpcli <URL>");
        std::process::exit(1);
    }

    let client = reqwest::blocking::Client::new();

    for url in &cli.urls {
        match client.get(url).send() {
            Ok(resp) => {
                let body = resp.text().unwrap_or_default();
                println!("{body}");
            }
            Err(e) => {
                eprintln!("Error requesting {url}: {e}");
                std::process::exit(1);
            }
        }
    }
}
```

ทดสอบรัน:

```bash
$ cargo run -- https://httpbin.org/get
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "X-Amzn-Trace-Id": "Root=1-..."
  },
  "origin": "1.2.3.4",
  "url": "https://httpbin.org/get"
}
```

**สังเกต:** clap สร้าง `--help` ให้อัตโนมัติ:

```bash
$ cargo run -- --help
A curl-like HTTP client written in Rust

Usage: httpcli [OPTIONS] [URLS]...

Arguments:
  [URLS]...  URL(s) to request (รองรับหลาย URL)

Options:
  -X, --method <METHOD>  HTTP method (GET, POST, PUT, DELETE, PATCH) [default: GET]
  -h, --help             Print help
  -V, --version          Print version
```

---

### ขั้นที่ 2: ทุก HTTP Method + Custom Headers + Request Body

เพิ่ม argument สำหรับ headers, body, และ HTTP methods อื่น ๆ:

```rust
use clap::Parser;
use reqwest::blocking::Client;
use reqwest::header::{HeaderMap, HeaderName, HeaderValue, CONTENT_TYPE};
use std::str::FromStr;

#[derive(Parser, Debug)]
#[command(name = "httpcli", version = "0.1.0", about = "A curl-like HTTP client")]
struct Cli {
    urls: Vec<String>,

    /// HTTP method
    #[arg(short = 'X', long, default_value = "GET")]
    method: String,

    /// HTTP header: -H "Name: Value" (ใช้ได้หลายครั้ง)
    #[arg(short = 'H', long = "header", value_name = "HEADER")]
    headers: Vec<String>,

    /// Request body data: -d '{"key":"value"}'
    #[arg(short = 'd', long, value_name = "DATA")]
    data: Option<String>,

    /// Shorthand สำหรับ JSON: ตั้ง Content-Type + Accept เป็น application/json
    #[arg(long)]
    json: bool,
}

/// แปลง "Key: Value" string เป็น (HeaderName, HeaderValue)
fn parse_header(s: &str) -> Result<(HeaderName, HeaderValue), String> {
    let pos = s
        .find(':')
        .ok_or_else(|| format!("invalid header (no colon): {s}"))?;
    let name = s[..pos].trim();
    let value = s[pos + 1..].trim();
    let header_name = HeaderName::from_str(name)
        .map_err(|e| format!("invalid header name '{name}': {e}"))?;
    let header_value = HeaderValue::from_str(value)
        .map_err(|e| format!("invalid header value '{value}': {e}"))?;
    Ok((header_name, header_value))
}

fn build_headers(cli: &Cli) -> Result<HeaderMap, String> {
    let mut map = HeaderMap::new();

    for h in &cli.headers {
        let (name, val) = parse_header(h)?;
        map.insert(name, val);
    }

    if cli.json {
        map.insert(CONTENT_TYPE, HeaderValue::from_static("application/json"));
        map.insert(
            reqwest::header::ACCEPT,
            HeaderValue::from_static("application/json"),
        );
    }

    Ok(map)
}

fn main() {
    let cli = Cli::parse();

    if cli.urls.is_empty() {
        eprintln!("Error: no URL provided");
        std::process::exit(1);
    }

    let headers = match build_headers(&cli) {
        Ok(h) => h,
        Err(e) => {
            eprintln!("Error: {e}");
            std::process::exit(1);
        }
    };

    let method_str = cli.method.to_uppercase();
    let method = reqwest::Method::from_str(&method_str)
        .unwrap_or(reqwest::Method::GET);

    let client = Client::new();

    for url in &cli.urls {
        let mut req = client.request(method.clone(), url).headers(headers.clone());

        if let Some(ref body) = cli.data {
            req = req.body(body.clone());
        }

        match req.send() {
            Ok(resp) => {
                let body = resp.text().unwrap_or_default();
                println!("{body}");
            }
            Err(e) => {
                eprintln!("Error: {e}");
                std::process::exit(1);
            }
        }
    }
}
```

ทดสอบ POST พร้อม JSON body:

```bash
$ cargo run -- -X POST \
    -H "Content-Type: application/json" \
    -d '{"name":"rust","version":2021}' \
    https://httpbin.org/post

{
  "data": "{\"name\":\"rust\",\"version\":2021}",
  "headers": {
    "Content-Length": "30",
    "Content-Type": "application/json",
    ...
  },
  ...
}
```

ใช้ `--json` shorthand:

```bash
$ cargo run -- --json -X POST -d '{"key":"val"}' https://httpbin.org/post
# เทียบเท่ากับ -H "Content-Type: application/json" -H "Accept: application/json"
```

**ทำไม `method.clone()`?** เราวน loop หลาย URL และ `reqwest::Method` ต้อง move เข้าไปใน request builder ดังนั้นต้อง clone ทุกรอบ ถ้าลืม clone จะเจอ error:

```
error[E0382]: use of moved value: `method`
  --> src/main.rs:68:31
   |
59 |     let method = reqwest::Method::from_str(&method_str)...
   |         ------ move occurs because `method` has type `Method`, which
   |                does not implement the `Copy` trait
...
68 |         let mut req = client.request(method, url).headers(headers.clone());
   |                                      ^^^^^^ value moved here, in previous iteration of loop
```

---

### ขั้นที่ 3: Verbose Mode — แสดง Request + Response Headers

Verbose mode ช่วย debug HTTP ได้มาก เราจะแสดง headers ก่อน send request และหลัง receive response:

```rust
use colored::Colorize;

fn print_request_info(method: &str, url: &str, headers: &HeaderMap, body: Option<&str>) {
    eprintln!("{}", format!("> {} {}", method, url).bold().blue());
    for (k, v) in headers {
        eprintln!("> {}: {}", k.as_str().cyan(), v.to_str().unwrap_or("?"));
    }
    if let Some(b) = body {
        if !b.is_empty() {
            eprintln!("> [request body] {}", b);
        }
    }
    eprintln!(">");
}

fn print_response_info(resp: &reqwest::blocking::Response) {
    let status = resp.status();
    let status_line = format!(
        "< HTTP/1.1 {} {}",
        status.as_u16(),
        status.canonical_reason().unwrap_or("")
    );

    if status.is_success() {
        eprintln!("{}", status_line.bold().green());
    } else if status.is_client_error() {
        eprintln!("{}", status_line.bold().yellow());
    } else {
        eprintln!("{}", status_line.bold().red());
    }

    for (k, v) in resp.headers() {
        eprintln!("< {}: {}", k.as_str().cyan(), v.to_str().unwrap_or("?"));
    }
    eprintln!("<");
}
```

เพิ่ม `-v` flag ใน Cli struct:

```rust
/// Verbose: show request + response headers
#[arg(short = 'v', long)]
verbose: bool,
```

ตัวอย่าง output ใน verbose mode:

```bash
$ httpcli -v -X POST -H "Content-Type: application/json" \
    -d '{"test":1}' https://httpbin.org/post

> POST https://httpbin.org/post
> content-type: application/json
> [request body] {"test":1}
>
< HTTP/1.1 200 OK
< content-type: application/json
< content-length: 423
< date: Sat, 01 Jan 2025 00:00:00 GMT
<
{
  "data": "{\"test\":1}",
  ...
}
```

**ทำไมใช้ `eprintln!` แทน `println!`?** verbose headers ไปยัง stderr เพื่อให้ผู้ใช้สามารถ pipe response body ไปยัง program อื่นได้โดยไม่มี headers ปะปน เช่น `httpcli -v https://api.example.com/data | jq .`

---

### ขั้นที่ 4: JSON Pretty-Printing + Content-Type Detection

เพิ่ม flag `-L` (follow redirects) และ `--timeout`:

```rust
/// Follow redirects (ตาม Location header สูงสุด 10 ครั้ง)
#[arg(short = 'L', long)]
follow_redirects: bool,

/// Timeout in seconds
#[arg(long, default_value = "30")]
timeout: u64,
```

เพิ่ม logic ตรวจ content-type และ pretty-print JSON:

```rust
use serde_json::Value;

fn display_body(body: &str, content_type: &str) {
    if content_type.contains("application/json") {
        // พยายาม parse เป็น JSON แล้ว pretty-print
        match serde_json::from_str::<Value>(body) {
            Ok(v) => {
                let pretty = serde_json::to_string_pretty(&v)
                    .unwrap_or_else(|_| body.to_string());
                println!("{pretty}");
            }
            Err(_) => {
                // ถ้า parse ไม่ได้ แสดง raw
                println!("{body}");
            }
        }
    } else {
        println!("{body}");
    }
}
```

ตัวอย่าง: ก่อนและหลัง pretty-print

```bash
# ก่อน: JSON แบบ compact
{"status":"ok","data":{"id":1,"name":"Alice","score":95.5}}

# หลัง: JSON แบบ pretty
{
  "status": "ok",
  "data": {
    "id": 1,
    "name": "Alice",
    "score": 95.5
  }
}
```

สร้าง client builder ที่ตั้งค่า redirect และ timeout:

```rust
use std::time::Duration;

fn build_client(cli: &Cli) -> Result<reqwest::blocking::Client, reqwest::Error> {
    let timeout = Duration::from_secs(cli.timeout);

    let redirect_policy = if cli.follow_redirects {
        reqwest::redirect::Policy::limited(10)
    } else {
        reqwest::redirect::Policy::none()
    };

    reqwest::blocking::ClientBuilder::new()
        .timeout(timeout)
        .redirect(redirect_policy)
        .danger_accept_invalid_certs(cli.insecure)
        .build()
}
```

---

### ขั้นที่ 5: Download ไปยังไฟล์พร้อม Progress Bar

เพิ่ม `-o` flag และ progress bar ด้วย `indicatif`:

```rust
/// Save response body to file
#[arg(short = 'o', long, value_name = "FILE")]
output: Option<std::path::PathBuf>,
```

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::fs;
use std::io::Write;

fn save_to_file(
    body_bytes: &[u8],
    output_path: &std::path::Path,
) -> Result<(), Box<dyn std::error::Error>> {
    let pb = ProgressBar::new(body_bytes.len() as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template(
                "{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] \
                 {bytes}/{total_bytes} ({eta})",
            )
            .unwrap()
            .progress_chars("#>-"),
    );

    let mut file = fs::File::create(output_path)?;
    // simulate chunk writing for progress
    const CHUNK: usize = 8192;
    let mut offset = 0;
    while offset < body_bytes.len() {
        let end = (offset + CHUNK).min(body_bytes.len());
        file.write_all(&body_bytes[offset..end])?;
        pb.inc((end - offset) as u64);
        offset = end;
    }

    pb.finish_with_message(format!("Saved to {}", output_path.display()));
    Ok(())
}
```

ตัวอย่างการใช้งาน:

```bash
$ httpcli -o logo.png https://www.rust-lang.org/logos/rust-logo-512x512.png
⠋ [00:00:00] [####################>-------------------] 28KB/56KB (00:00:00)
✓ [00:00:00] [########################################] 56KB/56KB (00:00:00)
Saved to logo.png
```

**ทำไม progress bar ถึงต้องการ bytes ล่วงหน้า?** `indicatif` ต้องรู้ total size ก่อน เราได้จาก `Content-Length` header หรือจาก `resp.bytes()` หลังดาวน์โหลดเสร็จ ถ้าต้องการ progress bar แบบ real-time ต้องใช้ async reqwest และอ่าน body เป็น stream ซึ่งเราจะทำใน Extension exercise

---

### ขั้นที่ 6: Retry พร้อม Exponential Backoff

สำหรับ 5xx errors และ timeout ควร retry อัตโนมัติ pattern นี้เรียกว่า **exponential backoff**:

```rust
/// Max retry attempts for transient errors
#[arg(long, default_value = "0")]
retry: u32,
```

```rust
use std::time::{Duration, Instant};

/// Exponential backoff: รอ 200ms, 400ms, 800ms, ...
fn backoff_duration(attempt: u32) -> Duration {
    let base_ms = 200u64;
    let max_ms = 30_000u64; // สูงสุด 30 วินาที
    let ms = base_ms * 2u64.pow(attempt);
    Duration::from_millis(ms.min(max_ms))
}

fn execute_with_retry(
    client: &reqwest::blocking::Client,
    method: &reqwest::Method,
    url: &str,
    headers: &reqwest::header::HeaderMap,
    body: Option<&str>,
    verbose: bool,
    max_retries: u32,
) -> Result<reqwest::blocking::Response, String> {
    let mut attempt = 0u32;

    loop {
        let mut req = client.request(method.clone(), url).headers(headers.clone());
        if let Some(b) = body {
            req = req.body(b.to_string());
        }

        match req.send() {
            Ok(resp) => {
                let status = resp.status().as_u16();
                // 5xx = server error = retry ได้
                if status >= 500 && attempt < max_retries {
                    let wait = backoff_duration(attempt);
                    if verbose {
                        eprintln!(
                            "  [retry {}/{}] HTTP {}, waiting {}ms...",
                            attempt + 1,
                            max_retries,
                            status,
                            wait.as_millis()
                        );
                    }
                    std::thread::sleep(wait);
                    attempt += 1;
                    continue;
                }
                return Ok(resp);
            }
            Err(e) if e.is_timeout() || e.is_connect() => {
                if attempt < max_retries {
                    let wait = backoff_duration(attempt);
                    if verbose {
                        eprintln!(
                            "  [retry {}/{}] {}, waiting {}ms...",
                            attempt + 1,
                            max_retries,
                            e,
                            wait.as_millis()
                        );
                    }
                    std::thread::sleep(wait);
                    attempt += 1;
                    continue;
                }
                return Err(e.to_string());
            }
            Err(e) => return Err(e.to_string()),
        }
    }
}
```

ตัวอย่างการใช้งาน:

```bash
$ httpcli --retry 3 -v https://api.example.com/unstable-endpoint

> GET https://api.example.com/unstable-endpoint
>
< HTTP/1.1 503 Service Unavailable
<
  [retry 1/3] HTTP 503, waiting 200ms...
  [retry 2/3] HTTP 503, waiting 400ms...
  [retry 3/3] HTTP 503, waiting 800ms...
< HTTP/1.1 200 OK
<
{"status": "ok"}
```

**ทำไมต้อง exponential backoff แทน constant delay?** ถ้า retry พร้อมกัน (thundering herd) จะ overwhelm server มากขึ้น exponential backoff ลด load อย่างค่อยเป็นค่อยไปและให้เวลา server ฟื้นตัว ใน production จะเพิ่ม jitter (random offset) เพื่อป้องกัน synchronized retries

---

### ขั้นที่ 7: Config File + Basic Auth

สร้าง config file ที่ `~/.config/httpcli/config.toml`:

```toml
# ~/.config/httpcli/config.toml

# Base URL ที่ต่อท้าย URL สั้น ๆ
base_url = "https://api.myservice.com"

# Default timeout (วินาที)
timeout = 60

# Headers ที่ส่งทุก request
[default_headers]
Authorization = "Bearer my-api-token-here"
User-Agent = "httpcli/0.1.0 my-team"
Accept = "application/json"
```

```rust
use serde::Deserialize;
use std::collections::HashMap;

#[derive(Debug, Deserialize, Default)]
pub struct Config {
    #[serde(default)]
    pub default_headers: HashMap<String, String>,
    pub base_url: Option<String>,
    pub timeout: Option<u64>,
}

pub fn load_config() -> Config {
    // dirs::config_dir() = ~/.config บน Linux, ~/Library/Application Support บน macOS
    let config_path = dirs::config_dir()
        .unwrap_or_else(|| std::path::PathBuf::from("."))
        .join("httpcli")
        .join("config.toml");

    if let Ok(content) = std::fs::read_to_string(&config_path) {
        match toml::from_str::<Config>(&content) {
            Ok(c) => return c,
            Err(e) => {
                eprintln!("Warning: failed to parse config file: {e}");
            }
        }
    }

    Config::default()
}
```

Basic auth (`-u user:password`) ใช้ Base64 encode:

```rust
fn apply_basic_auth(headers: &mut HeaderMap, user_pass: &str) -> Result<(), String> {
    // Base64 encode "user:pass"
    let encoded = base64_encode(user_pass.as_bytes());
    let value = format!("Basic {encoded}");
    let header_val = reqwest::header::HeaderValue::from_str(&value)
        .map_err(|e| format!("invalid auth value: {e}"))?;
    headers.insert(reqwest::header::AUTHORIZATION, header_val);
    Ok(())
}
```

ตัวอย่าง:

```bash
# ใช้ Basic auth
$ httpcli -u admin:secret123 https://api.example.com/protected

# หรืออ่านจาก config
# config.toml มี Authorization = "Bearer token123"
$ httpcli https://api.example.com/protected
```

เพิ่ม `-k` flag (skip TLS) พร้อมคำเตือน:

```rust
/// INSECURE: skip TLS certificate verification
#[arg(short = 'k', long)]
insecure: bool,
```

```rust
if cli.insecure {
    eprintln!(
        "{}",
        "WARNING: TLS certificate verification is DISABLED. \
         Your connection may not be secure. \
         Use only in development/testing environments."
            .bold()
            .yellow()
    );
}
```

---

### ขั้นที่ 8: `--format json` Output Mode + Multiple URLs

เพิ่ม `--format` flag สำหรับ machine-readable output:

```rust
/// Output format: "pretty" (default) or "json" (structured output)
#[arg(long, default_value = "pretty")]
format: String,
```

```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
struct JsonOutput {
    status: u16,
    headers: HashMap<String, String>,
    body: String,
    duration_ms: u128,
}

fn output_json_format(
    resp: reqwest::blocking::Response,
    duration: std::time::Duration,
) -> Result<(), Box<dyn std::error::Error>> {
    let status = resp.status().as_u16();
    let resp_headers = resp.headers().clone();

    let mut headers_map = HashMap::new();
    for (k, v) in &resp_headers {
        headers_map.insert(k.to_string(), v.to_str().unwrap_or("").to_string());
    }

    let body = resp.text()?;
    let out = JsonOutput {
        status,
        headers: headers_map,
        body,
        duration_ms: duration.as_millis(),
    };

    println!("{}", serde_json::to_string_pretty(&out)?);
    Ok(())
}
```

ตัวอย่าง output:

```bash
$ httpcli --format json https://httpbin.org/status/200
{
  "status": 200,
  "headers": {
    "content-type": "text/html; charset=utf-8",
    "content-length": "0",
    "date": "Sat, 01 Jan 2025 12:00:00 GMT"
  },
  "body": "",
  "duration_ms": 243
}
```

ประโยชน์ใหญ่: สามารถ pipe ไปยัง `jq` ได้:

```bash
$ httpcli --format json https://api.example.com/health | jq '.status'
200

$ httpcli --format json https://api.example.com/health | jq '.duration_ms'
145
```

Multiple URLs พร้อม separator:

```rust
for (i, url) in cli.urls.iter().enumerate() {
    if i > 0 {
        println!("\n{}\n", "─".repeat(60));
    }

    let start = std::time::Instant::now();
    // ... execute request ...
    let duration = start.elapsed();
    // ... handle response ...
}
```

ตัวอย่าง:

```bash
$ httpcli https://httpbin.org/get https://httpbin.org/ip

{
  "args": {},
  "headers": { ... },
  "url": "https://httpbin.org/get"
}

────────────────────────────────────────────────────────────

{
  "origin": "1.2.3.4"
}
```

---

## โปรแกรมสมบูรณ์ (Complete Implementation)

ด้านล่างคือ `src/main.rs` ฉบับสมบูรณ์ที่รวมทุก feature จาก 8 ขั้นตอน:

```rust
use clap::Parser;
use colored::Colorize;
use indicatif::{ProgressBar, ProgressStyle};
use reqwest::blocking::{Client, ClientBuilder, Response};
use reqwest::header::{HeaderMap, HeaderName, HeaderValue, CONTENT_TYPE};
use serde::{Deserialize, Serialize};
use serde_json::Value;
use std::collections::HashMap;
use std::fs;
use std::io::Write;
use std::path::PathBuf;
use std::str::FromStr;
use std::time::{Duration, Instant};

// ── CLI Definition ──────────────────────────────────────────

#[derive(Parser, Debug)]
#[command(
    name = "httpcli",
    version = "0.1.0",
    about = "A curl-like HTTP client written in Rust"
)]
pub struct Cli {
    /// URL(s) to request
    pub urls: Vec<String>,

    /// HTTP method (GET, POST, PUT, DELETE, PATCH)
    #[arg(short = 'X', long, default_value = "GET")]
    pub method: String,

    /// HTTP header: -H "Name: Value" (ใช้ได้หลายครั้ง)
    #[arg(short = 'H', long = "header", value_name = "HEADER")]
    pub headers: Vec<String>,

    /// Request body
    #[arg(short = 'd', long, value_name = "DATA")]
    pub data: Option<String>,

    /// Save response to file
    #[arg(short = 'o', long, value_name = "FILE")]
    pub output: Option<PathBuf>,

    /// Verbose mode: show headers
    #[arg(short = 'v', long)]
    pub verbose: bool,

    /// Follow HTTP redirects
    #[arg(short = 'L', long)]
    pub follow_redirects: bool,

    /// Request timeout (seconds)
    #[arg(long, default_value = "30")]
    pub timeout: u64,

    /// Basic auth: user:password
    #[arg(short = 'u', long, value_name = "USER:PASS")]
    pub user: Option<String>,

    /// Skip TLS verification (INSECURE)
    #[arg(short = 'k', long)]
    pub insecure: bool,

    /// Set JSON Content-Type + Accept headers
    #[arg(long)]
    pub json: bool,

    /// Output format: "pretty" or "json"
    #[arg(long, default_value = "pretty")]
    pub format: String,

    /// Cookie jar file for persisting cookies
    #[arg(long, value_name = "FILE")]
    pub cookie_jar: Option<PathBuf>,

    /// Max retry attempts for transient errors
    #[arg(long, default_value = "0")]
    pub retry: u32,
}

// ── Config File ─────────────────────────────────────────────

#[derive(Debug, Deserialize, Default)]
pub struct Config {
    #[serde(default)]
    pub default_headers: HashMap<String, String>,
    pub base_url: Option<String>,
    pub timeout: Option<u64>,
}

pub fn load_config() -> Config {
    let config_path = dirs::config_dir()
        .unwrap_or_else(|| PathBuf::from("."))
        .join("httpcli")
        .join("config.toml");

    if let Ok(content) = fs::read_to_string(&config_path) {
        toml::from_str(&content).unwrap_or_default()
    } else {
        Config::default()
    }
}

// ── JSON Output ─────────────────────────────────────────────

#[derive(Debug, Serialize)]
pub struct JsonOutput {
    pub status: u16,
    pub headers: HashMap<String, String>,
    pub body: String,
    pub duration_ms: u128,
}

// ── Header Parsing ──────────────────────────────────────────

pub fn parse_header(s: &str) -> Result<(HeaderName, HeaderValue), String> {
    let pos = s
        .find(':')
        .ok_or_else(|| format!("invalid header (no colon): {s}"))?;
    let name = s[..pos].trim();
    let value = s[pos + 1..].trim();
    let header_name = HeaderName::from_str(name)
        .map_err(|e| format!("invalid header name '{name}': {e}"))?;
    let header_value = HeaderValue::from_str(value)
        .map_err(|e| format!("invalid header value '{value}': {e}"))?;
    Ok((header_name, header_value))
}

// ── Base64 Encode (สำหรับ Basic Auth) ──────────────────────

pub fn base64_encode(input: &[u8]) -> String {
    const TABLE: &[u8] =
        b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    let mut out = String::new();
    let mut i = 0;
    while i + 2 < input.len() {
        let b0 = input[i] as usize;
        let b1 = input[i + 1] as usize;
        let b2 = input[i + 2] as usize;
        out.push(TABLE[b0 >> 2] as char);
        out.push(TABLE[((b0 & 3) << 4) | (b1 >> 4)] as char);
        out.push(TABLE[((b1 & 0xf) << 2) | (b2 >> 6)] as char);
        out.push(TABLE[b2 & 0x3f] as char);
        i += 3;
    }
    let rem = input.len() - i;
    if rem == 1 {
        let b0 = input[i] as usize;
        out.push(TABLE[b0 >> 2] as char);
        out.push(TABLE[(b0 & 3) << 4] as char);
        out.push_str("==");
    } else if rem == 2 {
        let b0 = input[i] as usize;
        let b1 = input[i + 1] as usize;
        out.push(TABLE[b0 >> 2] as char);
        out.push(TABLE[((b0 & 3) << 4) | (b1 >> 4)] as char);
        out.push(TABLE[(b1 & 0xf) << 2] as char);
        out.push('=');
    }
    out
}

// ── Client Builder ──────────────────────────────────────────

pub fn build_client(cli: &Cli) -> Result<Client, reqwest::Error> {
    let timeout = Duration::from_secs(cli.timeout);
    let redirect = if cli.follow_redirects {
        reqwest::redirect::Policy::limited(10)
    } else {
        reqwest::redirect::Policy::none()
    };

    ClientBuilder::new()
        .timeout(timeout)
        .redirect(redirect)
        .danger_accept_invalid_certs(cli.insecure)
        .build()
}

// ── Header Builder ──────────────────────────────────────────

pub fn build_headers(cli: &Cli, config: &Config) -> Result<HeaderMap, String> {
    let mut map = HeaderMap::new();

    // 1. Config defaults (ลำดับความสำคัญต่ำสุด)
    for (k, v) in &config.default_headers {
        if let (Ok(name), Ok(val)) = (
            HeaderName::from_str(k.as_str()),
            HeaderValue::from_str(v.as_str()),
        ) {
            map.insert(name, val);
        }
    }

    // 2. CLI headers (override config)
    for h in &cli.headers {
        let (name, val) = parse_header(h)?;
        map.insert(name, val);
    }

    // 3. --json shorthand
    if cli.json {
        map.insert(CONTENT_TYPE, HeaderValue::from_static("application/json"));
        map.insert(
            reqwest::header::ACCEPT,
            HeaderValue::from_static("application/json"),
        );
    }

    // 4. Basic auth
    if let Some(ref auth) = cli.user {
        let encoded = base64_encode(auth.as_bytes());
        let val = HeaderValue::from_str(&format!("Basic {encoded}"))
            .map_err(|e| e.to_string())?;
        map.insert(reqwest::header::AUTHORIZATION, val);
    }

    Ok(map)
}

// ── Retry with Exponential Backoff ──────────────────────────

fn backoff_duration(attempt: u32) -> Duration {
    let ms = 200u64 * 2u64.pow(attempt);
    Duration::from_millis(ms.min(30_000))
}

pub fn execute_with_retry(
    client: &Client,
    method: &str,
    url: &str,
    headers: &HeaderMap,
    body: Option<&str>,
    verbose: bool,
    max_retries: u32,
) -> Result<Response, String> {
    let req_method = reqwest::Method::from_str(&method.to_uppercase())
        .unwrap_or(reqwest::Method::GET);
    let mut attempt = 0u32;

    loop {
        if verbose {
            eprintln!("{}", format!("> {} {}", method.to_uppercase(), url).bold().blue());
            for (k, v) in headers {
                eprintln!("> {}: {}", k.as_str().cyan(), v.to_str().unwrap_or("?"));
            }
            if let Some(b) = body {
                eprintln!("> [body] {b}");
            }
            eprintln!(">");
        }

        let mut req = client.request(req_method.clone(), url).headers(headers.clone());
        if let Some(b) = body {
            req = req.body(b.to_string());
        }

        match req.send() {
            Ok(resp) => {
                let status = resp.status().as_u16();
                if status >= 500 && attempt < max_retries {
                    let wait = backoff_duration(attempt);
                    if verbose {
                        eprintln!(
                            "  [retry {}/{}] HTTP {}, waiting {}ms...",
                            attempt + 1, max_retries, status, wait.as_millis()
                        );
                    }
                    std::thread::sleep(wait);
                    attempt += 1;
                    continue;
                }
                return Ok(resp);
            }
            Err(e) if e.is_timeout() || e.is_connect() => {
                if attempt < max_retries {
                    let wait = backoff_duration(attempt);
                    if verbose {
                        eprintln!(
                            "  [retry {}/{}] {}, waiting {}ms...",
                            attempt + 1, max_retries, e, wait.as_millis()
                        );
                    }
                    std::thread::sleep(wait);
                    attempt += 1;
                    continue;
                }
                return Err(e.to_string());
            }
            Err(e) => return Err(e.to_string()),
        }
    }
}

// ── Response Handler ─────────────────────────────────────────

pub fn handle_response(
    resp: Response,
    cli: &Cli,
    duration: Duration,
) -> Result<(), Box<dyn std::error::Error>> {
    let status = resp.status();
    let resp_headers = resp.headers().clone();
    let content_type = resp_headers
        .get(CONTENT_TYPE)
        .and_then(|v| v.to_str().ok())
        .unwrap_or("")
        .to_string();

    if cli.verbose {
        let status_line = format!(
            "< HTTP/1.1 {} {}",
            status.as_u16(),
            status.canonical_reason().unwrap_or("")
        );
        if status.is_success() {
            eprintln!("{}", status_line.bold().green());
        } else if status.is_client_error() {
            eprintln!("{}", status_line.bold().yellow());
        } else {
            eprintln!("{}", status_line.bold().red());
        }
        for (k, v) in &resp_headers {
            eprintln!("< {}: {}", k.as_str().cyan(), v.to_str().unwrap_or("?"));
        }
        eprintln!("<");
    }

    let body_bytes = resp.bytes()?;
    let body_str = String::from_utf8_lossy(&body_bytes).into_owned();

    match cli.format.as_str() {
        "json" => {
            // Machine-readable: { status, headers, body, duration_ms }
            let mut headers_map = HashMap::new();
            for (k, v) in &resp_headers {
                headers_map.insert(k.to_string(), v.to_str().unwrap_or("").to_string());
            }
            let out = JsonOutput {
                status: status.as_u16(),
                headers: headers_map,
                body: body_str,
                duration_ms: duration.as_millis(),
            };
            println!("{}", serde_json::to_string_pretty(&out)?);
        }
        _ => {
            // pretty mode (default)
            if let Some(output_path) = &cli.output {
                // บันทึกลงไฟล์พร้อม progress bar
                let pb = ProgressBar::new(body_bytes.len() as u64);
                pb.set_style(
                    ProgressStyle::default_bar()
                        .template(
                            "{spinner:.green} [{elapsed_precise}] \
                             [{bar:40.cyan/blue}] {bytes}/{total_bytes} ({eta})",
                        )
                        .unwrap()
                        .progress_chars("#>-"),
                );
                let mut file = fs::File::create(output_path)?;
                const CHUNK: usize = 8192;
                let mut offset = 0;
                while offset < body_bytes.len() {
                    let end = (offset + CHUNK).min(body_bytes.len());
                    file.write_all(&body_bytes[offset..end])?;
                    pb.inc((end - offset) as u64);
                    offset = end;
                }
                pb.finish_with_message(format!("Saved to {}", output_path.display()));
            } else {
                // แสดงบน stdout
                if content_type.contains("application/json") {
                    match serde_json::from_str::<Value>(&body_str) {
                        Ok(v) => println!("{}", serde_json::to_string_pretty(&v)?),
                        Err(_) => println!("{body_str}"),
                    }
                } else {
                    println!("{body_str}");
                }
            }
        }
    }

    Ok(())
}

// ── Main ─────────────────────────────────────────────────────

fn main() {
    let cli = Cli::parse();
    let config = load_config();

    if cli.insecure {
        eprintln!(
            "{}",
            "WARNING: TLS certificate verification is DISABLED. \
             Use only in dev/test environments."
                .bold()
                .yellow()
        );
    }

    let client = match build_client(&cli) {
        Ok(c) => c,
        Err(e) => {
            eprintln!("Error building client: {e}");
            std::process::exit(1);
        }
    };

    let headers = match build_headers(&cli, &config) {
        Ok(h) => h,
        Err(e) => {
            eprintln!("Error parsing headers: {e}");
            std::process::exit(1);
        }
    };

    if cli.urls.is_empty() {
        eprintln!("Error: no URL provided. Run with --help for usage.");
        std::process::exit(1);
    }

    for (i, url) in cli.urls.iter().enumerate() {
        if i > 0 {
            println!("\n{}\n", "─".repeat(60));
        }

        let start = Instant::now();
        let result = execute_with_retry(
            &client,
            &cli.method,
            url,
            &headers,
            cli.data.as_deref(),
            cli.verbose,
            cli.retry,
        );
        let duration = start.elapsed();

        match result {
            Ok(resp) => {
                if let Err(e) = handle_response(resp, &cli, duration) {
                    eprintln!("Error handling response: {e}");
                    std::process::exit(1);
                }
            }
            Err(e) => {
                eprintln!("Request failed for {url}: {e}");
                std::process::exit(1);
            }
        }
    }
}

// ── Unit Tests ───────────────────────────────────────────────

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_header_valid() {
        let (name, val) = parse_header("Content-Type: application/json").unwrap();
        assert_eq!(name.as_str(), "content-type");
        assert_eq!(val.to_str().unwrap(), "application/json");
    }

    #[test]
    fn test_parse_header_with_spaces() {
        let (name, val) = parse_header("X-Custom-Header:  my-value  ").unwrap();
        assert_eq!(name.as_str(), "x-custom-header");
        assert_eq!(val.to_str().unwrap(), "my-value");
    }

    #[test]
    fn test_parse_header_missing_colon() {
        let result = parse_header("NoColonHere");
        assert!(result.is_err());
        assert!(result.unwrap_err().contains("no colon"));
    }

    #[test]
    fn test_base64_encode() {
        assert_eq!(base64_encode(b"user:pass"), "dXNlcjpwYXNz");
        assert_eq!(base64_encode(b"admin:secret123"), "YWRtaW46c2VjcmV0MTIz");
        assert_eq!(base64_encode(b""), "");
    }

    #[test]
    fn test_config_default() {
        let c = Config::default();
        assert!(c.default_headers.is_empty());
        assert!(c.base_url.is_none());
        assert!(c.timeout.is_none());
    }
}
```

---

## การทดสอบ (Testing)

### Unit Tests

Unit tests ทดสอบ pure functions ที่ไม่ต้องการ network:

```rust
#[test]
fn test_parse_header_valid() {
    let (name, val) = parse_header("Content-Type: application/json").unwrap();
    assert_eq!(name.as_str(), "content-type");
    assert_eq!(val.to_str().unwrap(), "application/json");
}

#[test]
fn test_parse_header_with_spaces() {
    let (name, val) = parse_header("X-Custom-Header:  my-value  ").unwrap();
    assert_eq!(name.as_str(), "x-custom-header");
    assert_eq!(val.to_str().unwrap(), "my-value");
}

#[test]
fn test_parse_header_missing_colon() {
    let result = parse_header("NoColonHere");
    assert!(result.is_err());
    assert!(result.unwrap_err().contains("no colon"));
}

#[test]
fn test_base64_encode() {
    assert_eq!(base64_encode(b"user:pass"), "dXNlcjpwYXNz");
    assert_eq!(base64_encode(b"admin:secret123"), "YWRtaW46c2VjcmV0MTIz");
    assert_eq!(base64_encode(b""), "");
}
```

### Integration Tests (ด้วย mockito)

`tests/integration_test.rs` ใช้ `mockito` สร้าง mock HTTP server จริงใน memory:

```rust
use mockito::Server;
use std::process::Command;

fn bin_path() -> std::path::PathBuf {
    let mut p = std::env::current_exe().unwrap();
    p.pop();
    if p.ends_with("deps") {
        p.pop();
    }
    p.push("httpcli");
    p
}

#[test]
fn test_get_request_prints_body() {
    let mut server = Server::new();
    let mock = server
        .mock("GET", "/hello")
        .with_status(200)
        .with_header("content-type", "text/plain")
        .with_body("Hello from mock server!")
        .create();

    let url = format!("{}/hello", server.url());
    let output = Command::new(bin_path())
        .arg(&url)
        .output()
        .expect("failed to run httpcli binary");

    let stdout = String::from_utf8_lossy(&output.stdout);
    assert!(output.status.success());
    assert!(stdout.contains("Hello from mock server!"));
    mock.assert();
}

#[test]
fn test_post_with_data_and_header() {
    let mut server = Server::new();
    let mock = server
        .mock("POST", "/echo")
        .with_status(201)
        .with_header("content-type", "application/json")
        .with_body(r#"{"received":true}"#)
        .match_header("content-type", "application/json")
        .match_body(r#"{"key":"val"}"#)
        .create();

    let url = format!("{}/echo", server.url());
    let output = Command::new(bin_path())
        .args([
            "-X", "POST",
            "-H", "Content-Type: application/json",
            "-d", r#"{"key":"val"}"#,
            &url,
        ])
        .output()
        .expect("failed to run httpcli binary");

    let stdout = String::from_utf8_lossy(&output.stdout);
    assert!(output.status.success());
    // JSON should be pretty-printed because content-type is application/json
    assert!(stdout.contains("received"));
    mock.assert();
}

#[test]
fn test_format_json_output() {
    let mut server = Server::new();
    let mock = server
        .mock("GET", "/status")
        .with_status(200)
        .with_header("content-type", "text/plain")
        .with_body("OK")
        .create();

    let url = format!("{}/status", server.url());
    let output = Command::new(bin_path())
        .args(["--format", "json", &url])
        .output()
        .expect("failed to run httpcli binary");

    let stdout = String::from_utf8_lossy(&output.stdout);
    assert!(output.status.success());

    // Must be valid JSON
    let parsed: serde_json::Value = serde_json::from_str(&stdout)
        .expect(&format!("Output is not valid JSON: {stdout}"));

    assert_eq!(parsed["status"].as_u64().unwrap(), 200);
    assert!(parsed["body"].as_str().unwrap().contains("OK"));
    assert!(parsed["duration_ms"].as_u64().is_some());
    mock.assert();
}

#[test]
fn test_multiple_urls() {
    let mut server = Server::new();
    let mock1 = server
        .mock("GET", "/one")
        .with_status(200)
        .with_body("response_one")
        .create();
    let mock2 = server
        .mock("GET", "/two")
        .with_status(200)
        .with_body("response_two")
        .create();

    let url1 = format!("{}/one", server.url());
    let url2 = format!("{}/two", server.url());
    let output = Command::new(bin_path())
        .args([&url1, &url2])
        .output()
        .expect("failed to run httpcli binary");

    let stdout = String::from_utf8_lossy(&output.stdout);
    assert!(output.status.success());
    assert!(stdout.contains("response_one"));
    assert!(stdout.contains("response_two"));
    // ต้องมี separator ระหว่าง response
    assert!(stdout.contains("─"));
    mock1.assert();
    mock2.assert();
}
```

### ผลลัพธ์ `cargo test` จริง

```
running 5 tests
test tests::test_base64_encode ... ok
test tests::test_parse_header_missing_colon ... ok
test tests::test_config_default ... ok
test tests::test_parse_header_with_spaces ... ok
test tests::test_parse_header_valid ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-bc5ba44c8ec4ff8e)

running 4 tests
test test_get_request_prints_body ... ok
test test_multiple_urls ... ok
test test_format_json_output ... ok
test test_post_with_data_and_header ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `reqwest::Method` ไม่ implement `Copy` — ต้อง `.clone()` ใน loop

**ปัญหา:**

```rust
let method = reqwest::Method::GET;
for url in &urls {
    let req = client.request(method, url); // ERROR: method moved on first iteration
}
```

**Error message:**

```
error[E0382]: use of moved value: `method`
  --> src/main.rs:42:31
   |
38 |     let method = reqwest::Method::GET;
   |         ------ move occurs because `method` has type `Method`,
   |                which does not implement the `Copy` trait
...
42 |         let req = client.request(method, url);
   |                                  ^^^^^^ value moved here,
   |                                         in previous iteration of loop
```

**วิธีแก้:**

```rust
let method = reqwest::Method::GET;
for url in &urls {
    let req = client.request(method.clone(), url); // clone ทุกรอบ
}
```

**ทำไม `Method` ไม่ implement `Copy`?** เพราะ `Method` อาจเป็น custom method ที่เก็บ `String` ภายใน และ `String` ไม่ implement `Copy`

---

### 2. Response ถูก consume แล้วไม่สามารถอ่านซ้ำได้

**ปัญหา:**

```rust
let resp = client.get(url).send()?;

// อ่าน status แล้วก็โอเค
let status = resp.status();

// ลองอ่าน headers ก็โอเค
let headers = resp.headers().clone();

// ลองอ่าน body — ERROR! resp ถูก consume ไปแล้วเมื่อกี้? ไม่
let body = resp.text()?; // OK ถ้ายังไม่ consume

// แต่ถ้าเรียก .text() แล้วเรียก .bytes() อีกครั้ง — ERROR!
let body_str = resp.text()?;
let body_bytes = resp.bytes()?; // ERROR: resp ถูก consume ไปแล้ว
```

**Error message:**

```
error[E0382]: use of moved value: `resp`
   --> src/main.rs:55:26
    |
54 |     let body_str = resp.text()?;
    |                   ---- `resp` moved due to this method call
55 |     let body_bytes = resp.bytes()?;
    |                     ^^^^ value used here after move
```

**วิธีแก้:** อ่านเป็น `bytes()` ครั้งเดียวก่อน แล้วค่อย convert:

```rust
let body_bytes = resp.bytes()?;
let body_str = String::from_utf8_lossy(&body_bytes).into_owned();
// ตอนนี้มีทั้ง bytes และ string ใช้ได้ทั้งคู่
```

---

### 3. ลืม `.clone()` ที่ `HeaderMap` ทำให้ borrow error

**ปัญหา:**

```rust
let headers = build_headers(&cli);

for url in &urls {
    let req = client.get(url).headers(headers); // headers ถูก move
    // ... รอบถัดไป headers หายไปแล้ว!
}
```

**วิธีแก้:**

```rust
for url in &urls {
    let req = client.get(url).headers(headers.clone()); // clone ทุกรอบ
}
```

`HeaderMap` implement `Clone` แต่ไม่ implement `Copy` เพราะมี `HeaderValue` ภายในที่อาจเก็บ heap data

---

### 4. `danger_accept_invalid_certs` ใน Production คือหายนะ

**ปัญหา:** หลายคน copy-paste `-k` flag จาก dev/test environment เข้า production script โดยไม่ตั้งใจ

```rust
// อย่าทำแบบนี้ใน production!
ClientBuilder::new()
    .danger_accept_invalid_certs(true) // <=== ปิด TLS verification ทั้งหมด
    .build()
```

**ผลกระทบ:** Man-in-the-middle attack เป็นไปได้ 100% ผู้โจมตีสามารถดักข้อมูลและปลอมแปลง response ได้

**วิธีป้องกัน:**
1. ตั้งค่า default เป็น `false` (ซึ่งเป็น default ของ reqwest อยู่แล้ว)
2. แสดง warning ด้วยสีเหลือง/แดงทุกครั้งที่ใช้ `-k`
3. Log ไว้ใน audit trail
4. อย่าใส่ไว้ใน config file — ควรบังคับให้ pass ทุกครั้งผ่าน CLI เท่านั้น

```rust
if cli.insecure {
    // แสดง warning ที่ชัดเจน ปฏิเสธใน production ไม่ได้
    eprintln!(
        "{}",
        "SECURITY WARNING: TLS verification DISABLED. \
         Connection may be intercepted."
            .bold()
            .red()
    );
    // อาจเพิ่ม env check:
    if std::env::var("HTTPCLI_ALLOW_INSECURE").is_err() {
        eprintln!("Set HTTPCLI_ALLOW_INSECURE=1 to confirm you understand the risk");
        std::process::exit(1);
    }
}
```

---

### 5. `toml::from_str` panic ถ้า TOML syntax ผิด

**ปัญหา:**

```rust
// ถ้า config.toml มี syntax ผิด เช่น "key = " (ไม่มี value)
let config: Config = toml::from_str(&content).unwrap(); // PANIC!
```

**วิธีแก้:** ใช้ `unwrap_or_default()` หรือ log แล้ว fallback:

```rust
let config = toml::from_str::<Config>(&content)
    .unwrap_or_else(|e| {
        eprintln!("Warning: config file parse error: {e}. Using defaults.");
        Config::default()
    });
```

---

### 6. `HeaderValue::from_str` ล้มเหลวสำหรับ Unicode headers

**ปัญหา:** HTTP header values ต้องเป็น ASCII เท่านั้น

```rust
let val = HeaderValue::from_str("ค่า header ภาษาไทย"); // ERROR!
```

**Error message:**

```
called `Result::unwrap()` on an `Err` value: InvalidHeaderValue
```

**วิธีแก้:** ต้อง encode เป็น Base64 หรือ percent-encode:

```rust
// Option 1: ใช้ ASCII เท่านั้น
let val = HeaderValue::from_static("ascii-only-value");

// Option 2: encode เป็น base64
let encoded = base64_encode("ค่าภาษาไทย".as_bytes());
let val = HeaderValue::from_str(&encoded)?;
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/httpcli

# ขนาดประมาณ 4-8 MB (ขึ้นกับ OS)
ls -lh target/release/httpcli
-rwxr-xr-x 1 user user 5.2M Jan 1 00:00 target/release/httpcli
```

### Strip Symbols เพื่อลดขนาด

```bash
# strip symbols ออก
strip target/release/httpcli

# หรือตั้งใน Cargo.toml
[profile.release]
strip = true
opt-level = "z"    # optimize for size
lto = true         # link-time optimization
codegen-units = 1  # maximize optimization
```

### Cross-compile สำหรับ Linux จาก macOS

```bash
# ติดตั้ง target
rustup target add x86_64-unknown-linux-musl

# Build static binary (ไม่ต้อง glibc)
cargo build --release --target x86_64-unknown-linux-musl

# Binary รันได้ทุก Linux ไม่ต้อง install dependencies
scp target/x86_64-unknown-linux-musl/release/httpcli server:/usr/local/bin/
```

### Install ไปยัง System

```bash
# ติดตั้งจาก source
cargo install --path .

# หรือ install จาก crates.io (ถ้า publish แล้ว)
cargo install httpcli

# uninstall
cargo uninstall httpcli
```

### Docker Image (Multi-stage Build)

```dockerfile
# Dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/httpcli /httpcli
ENTRYPOINT ["/httpcli"]
```

```bash
docker build -t httpcli .
docker run --rm httpcli https://httpbin.org/get
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: Async + Streaming Download พร้อม Real-time Progress Bar (ระดับกลาง)

**โจทย์:** เปลี่ยนจาก `reqwest::blocking` เป็น `reqwest` async แล้วอ่าน body เป็น stream เพื่อแสดง progress bar แบบ real-time ขณะ download

**Hint:**

```rust
use tokio::io::AsyncWriteExt;
use futures_util::StreamExt;

// เปลี่ยน async fn main()
#[tokio::main]
async fn main() {
    let resp = reqwest::get(url).await?;
    let total = resp.content_length();

    let pb = ProgressBar::new(total.unwrap_or(0));
    let mut stream = resp.bytes_stream();
    let mut file = tokio::fs::File::create(path).await?;

    while let Some(chunk) = stream.next().await {
        let bytes = chunk?;
        file.write_all(&bytes).await?;
        pb.inc(bytes.len() as u64);
    }
    pb.finish();
}
```

**สิ่งที่จะได้เรียน:** async/await, Stream trait, `futures_util`, tokio file I/O

---

### Exercise 2: Cookie Jar Persistence (ระดับกลาง)

**โจทย์:** Implement `--cookie-jar file.txt` ที่บันทึก/โหลด cookies ระหว่าง requests จาก Netscape cookie format

**Hint:**

Cookie file format:
```
# Netscape HTTP Cookie File
.example.com    TRUE    /    FALSE    0    session_id    abc123
.example.com    TRUE    /    FALSE    0    user_pref    dark_mode
```

```rust
// อ่าน cookie จากไฟล์
fn load_cookies(path: &Path) -> Vec<(String, String, String)> {
    // return vec of (domain, name, value)
    todo!()
}

// บันทึก cookies หลัง request
fn save_cookies(path: &Path, resp: &Response) {
    // extract Set-Cookie headers and append to file
    todo!()
}
```

**สิ่งที่จะได้เรียน:** HTTP cookie protocol, file I/O, string parsing

---

### Exercise 3: HTTP/2 + Connection Pooling Benchmark (ระดับสูง)

**โจทย์:** เพิ่ม `--http2` flag และทดสอบ throughput ด้วย `--benchmark N` ที่ส่ง N requests แบบ concurrent แล้วรายงานสถิติ (min/max/avg/p99 latency)

**Hint:**

```rust
use std::sync::Arc;
use tokio::task::JoinSet;

async fn benchmark(url: &str, count: usize) {
    let client = Arc::new(reqwest::Client::builder()
        .http2_prior_knowledge()
        .build()?);

    let mut set = JoinSet::new();
    let mut latencies = Vec::new();

    for _ in 0..count {
        let c = client.clone();
        let u = url.to_string();
        set.spawn(async move {
            let start = Instant::now();
            let _ = c.get(&u).send().await;
            start.elapsed()
        });
    }

    while let Some(Ok(lat)) = set.join_next().await {
        latencies.push(lat);
    }

    // คำนวณสถิติ
    latencies.sort();
    println!("min: {:?}", latencies.first());
    println!("p99: {:?}", latencies[latencies.len() * 99 / 100]);
    println!("max: {:?}", latencies.last());
}
```

**สิ่งที่จะได้เรียน:** HTTP/2, async concurrency, JoinSet, statistics

---

### Exercise 4: HAR File Export (ระดับสูง)

**โจทย์:** เพิ่ม `--har output.har` ที่ export HTTP Archive (HAR) format ซึ่ง Chrome DevTools, Postman และเครื่องมืออื่น ๆ อ่านได้ HAR format เป็น JSON ตาม spec: https://w3c.github.io/web-performance/specs/HAR/Overview.html

**Hint:**

```rust
#[derive(Serialize)]
struct HarLog {
    version: &'static str,
    creator: HarCreator,
    entries: Vec<HarEntry>,
}

#[derive(Serialize)]
struct HarEntry {
    started_date_time: String,  // ISO 8601
    time: f64,                   // ms
    request: HarRequest,
    response: HarResponse,
}

// implement Serialize สำหรับทุก struct แล้ว serde_json::to_writer_pretty(file, &log)
```

**สิ่งที่จะได้เรียน:** complex serde Serialize, industry-standard file formats, timestamp handling

---

## สรุป

ในโปรเจคนี้เราได้สร้าง HTTP client ที่ใช้งานได้จริง ครอบคลุม:

**Pattern สำคัญที่ได้เรียน:**

1. **Builder Pattern** — `ClientBuilder`, `RequestBuilder` ของ reqwest ใช้ builder pattern อย่างสง่างาม เราเห็นว่า Rust chain methods ด้วย `?` propagation ได้อย่างราบรื่น

2. **Layered Configuration** — config file เป็น default, CLI override ทีหลัง เป็น pattern ที่ใช้กันทั่วไปใน real tools (git config, cargo config, etc.)

3. **Error Propagation** — ใช้ `Result<T, E>` ในทุก function และ `?` operator ทำให้ error path ชัดเจน ไม่มี panic ซ่อนอยู่

4. **Integration Testing** — mockito ช่วยให้เราทดสอบ HTTP behavior จริงโดยไม่ต้องพึ่ง external network ซึ่งสำคัญมากใน CI/CD

5. **Progressive Enhancement** — เริ่มจาก GET ง่าย ๆ แล้วค่อยเพิ่ม feature ทีละขั้น วิธีนี้ทำให้เราสามารถ verify แต่ละ step ก่อนเดินหน้า

**โปรเจคถัดไป (A03 — File Watcher Daemon)** จะใช้ทักษะที่คล้ายกันแต่เน้นเรื่อง OS-level filesystem events, `inotify` บน Linux, background threads, และ channel-based message passing ซึ่งเป็นอีกมุมหนึ่งของ systems programming ใน Rust

---

**โปรเจคก่อนหน้า:** [Shell Interpreter](project-a01-shell-interpreter.md) | **โปรเจคถัดไป:** [File Watcher Daemon](project-a03-file-watcher.md)
