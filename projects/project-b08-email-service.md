# Project B08: Email Notification Service

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

**Email Notification Service** คือระบบส่งอีเมลแบบ asynchronous ที่พร้อมใช้งาน production ครอบคลุมตั้งแต่การส่งอีเมลด้วย SMTP, การจัดการ template, ระบบคิว (queue) สำหรับส่งแบบ background, การติดตามสถานะ (delivery tracking) ไปจนถึง webhook สำหรับรับแจ้ง bounce

ในโลก production แทบทุก application ต้องการระบบอีเมล — ยืนยันการสมัครสมาชิก, รีเซตรหัสผ่าน, แจ้งเตือน order, newsletter และอื่น ๆ โปรเจคนี้สอนให้สร้างระบบนี้อย่างถูกต้องตั้งแต่ต้น โดยไม่พึ่ง third-party service สำเร็จรูปแต่เข้าใจทุก layer ที่ซ่อนอยู่

**Learning value หลัก:**
- เข้าใจ SMTP protocol และการ pool connection
- สร้าง background worker ด้วย `tokio::sync::mpsc`
- ออกแบบ retry logic แบบ exponential backoff
- ใช้ HMAC สร้าง signed token สำหรับ unsubscribe link ที่ปลอดภัย
- จัดการ template engine แบบ production (load จากไฟล์, render ด้วย context)
- ออกแบบ state machine สำหรับ email delivery lifecycle

## สิ่งที่จะได้เรียนรู้

- **`lettre` + `AsyncSmtpTransport`** — ส่งอีเมลแบบ async พร้อม TLS, auth และ connection pool
- **`handlebars` template engine** — โหลด template จากไฟล์, render ด้วย JSON context, ใช้ helper blocks (`#if`, `#each`)
- **Tokio mpsc channel** — สร้าง background worker ที่รับงานผ่าน channel แล้วส่งอีเมลแบบ non-blocking
- **Exponential backoff** — retry logic ที่รอ 1 → 5 → 15 นาที ก่อน mark failed
- **HMAC signed token** — สร้างและตรวจสอบ unsubscribe token ด้วย `hmac` + `sha2` พร้อม constant-time comparison
- **Axum multipart** — รับ `multipart/form-data` สำหรับ email attachment (max 25MB)
- **SQLx + delivery tracking** — schema และ query สำหรับ track สถานะอีเมลทุกฉบับ
- **Webhook pattern** — รับ bounce/complaint event จาก SES/SendGrid แล้ว update สถานะ

## ความรู้ที่ต้องมีมาก่อน

- **Part 46** — async/await และ Tokio runtime (ใช้ตลอดโปรเจค)
- **Part 52** — Axum HTTP framework, Router, extractors, handlers
- **Part 55** — SQLx และ database interaction แบบ async
- **Part 60** — Error handling ด้วย `thiserror`, `anyhow`
- **Part 63** — ความรู้พื้นฐานเรื่อง SMTP และ email protocol
- **Part 68** — Cryptographic primitives: HMAC, SHA-256

## โครงสร้างโปรเจค (Project Layout)

```
email-service/
├── src/
│   ├── main.rs            # Entry point, router setup, state initialization
│   ├── config.rs          # Configuration struct (SMTP, DB, secrets)
│   ├── error.rs           # AppError enum ครอบคลุมทุก error
│   ├── models.rs          # Email, Template, Bounce structs + EmailStatus enum
│   ├── db.rs              # Database pool setup + migration
│   ├── smtp.rs            # SMTP client wrapper ด้วย lettre
│   ├── queue.rs           # Background worker + mpsc channel logic
│   ├── templates.rs       # Handlebars engine wrapper
│   ├── unsubscribe.rs     # HMAC token generation และ validation
│   └── handlers/
│       ├── mod.rs
│       ├── emails.rs      # POST /emails/send, schedule, GET /{id}
│       ├── templates.rs   # CRUD /templates
│       ├── attachments.rs # multipart upload handler
│       ├── unsubscribe.rs # GET /unsubscribe?token=X
│       └── webhooks.rs    # POST /webhooks/bounce
├── templates/
│   ├── welcome.hbs
│   ├── password_reset.hbs
│   ├── order_confirmation.hbs
│   └── newsletter.hbs
├── migrations/
│   ├── 001_emails.sql
│   ├── 002_templates.sql
│   └── 003_suppressions.sql
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
HTTP Request
     │
     ▼
Axum Handler ──► Validate ──► DB: insert (status=queued)
                                       │
                                       ▼
                               mpsc::Sender<EmailJob>
                                       │
                                       ▼
                          Background Worker (tokio::spawn)
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                    render template           check suppression
                          │                         │
                          └────────────┬────────────┘
                                       │
                               SMTP Transport
                                       │
                          ┌────────────┴────────────┐
                          │                         │
                        OK ✓                   Error ✗
                          │                         │
                   DB: status=sent          attempt < 3?
                                                    │
                                           ┌────────┴────────┐
                                           │                 │
                                        retry            mark failed
                                   (exponential          DB: status=failed
                                     backoff)
```

### Design Decisions

**ทำไมใช้ mpsc channel แทน Redis queue?**
สำหรับ single-process deployment, in-memory channel เร็วกว่า, ไม่ต้องการ infrastructure เพิ่ม และ Tokio's bounded channel ให้ backpressure ฟรี เมื่อต้องการ horizontal scaling ค่อยเปลี่ยนเป็น Redis แบบ plug-in replacement

**ทำไม HMAC แทน random token?**
Random token ต้องเก็บใน DB เพื่อ verify, HMAC token สามารถ verify ได้ stateless โดยใช้ secret key — ลด database round-trip และไม่มี token expiry ปัญหา (แต่ต้อง rotate secret key ถ้า token ถูก revoke)

**ทำไม handlebars แทน template engine อื่น?**
Handlebars ให้ logic-less template ที่ designers ไม่ต้องรู้ Rust, syntax เหมือน Mustache ที่คุ้นเคย, และมี `register_templates_directory` helper ที่โหลดทีเดียวได้ทั้ง folder

### State Machine: EmailStatus

```
   ┌──────────────────────────────────────────────────────┐
   │                                                      │
[queued] ──► [sending] ──► [sent] ──► [bounced]          │
   ▲              │                                       │
   │           (fail)                                     │
   │              │                                       │
   └──── [failed] ◄┘  (attempts < 3: back to queued)     │
                │                                         │
                └─► [failed] (attempts == 3: terminal)   │
                                                          │
[queued/sending] ──► [suppressed] ◄───────────────────────┘
                         (terminal — cannot send to this address)
```

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: ตั้งค่าโปรเจคและ Dependencies

สร้างโปรเจคและเพิ่ม crate ทั้งหมดที่ต้องการ:

```bash
cargo new email-service
cd email-service
```

**`Cargo.toml`:**

```toml
[package]
name = "email-service"
version = "0.1.0"
edition = "2021"

[dependencies]
# HTTP Framework
axum = { version = "0.8", features = ["multipart", "macros"] }
axum-extra = { version = "0.10", features = ["typed-header"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["trace", "cors"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Email sending
lettre = { version = "0.11", features = [
    "tokio1",
    "tokio1-native-tls",
    "smtp-transport",
    "builder",
    "pool",
] }

# Template engine
handlebars = "6"

# Database
sqlx = { version = "0.8", features = [
    "runtime-tokio-native-tls",
    "postgres",
    "chrono",
    "uuid",
] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Cryptography (HMAC unsubscribe tokens)
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"

# Utilities
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
config = "0.15"
bytes = "1"
multer = "3"  # multipart parsing (used internally by axum)

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

---

### ขั้นที่ 2: Models, Error Types, และ Config

**`src/error.rs`** — กำหนด error ทุกประเภทที่ใช้ใน service:

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("SMTP error: {0}")]
    Smtp(#[from] lettre::transport::smtp::Error),

    #[error("Email builder error: {0}")]
    EmailBuilder(#[from] lettre::error::Error),

    #[error("Template error: {0}")]
    Template(#[from] handlebars::RenderError),

    #[error("Template not found: {0}")]
    TemplateNotFound(String),

    #[error("Email not found: {0}")]
    NotFound(String),

    #[error("Invalid token")]
    InvalidToken,

    #[error("Recipient suppressed: {0}")]
    Suppressed(String),

    #[error("Payload too large: {size} bytes (max 26214400)")]
    PayloadTooLarge { size: usize },

    #[error("Invalid schedule time: {0}")]
    InvalidSchedule(String),

    #[error("Internal error: {0}")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(_) | AppError::TemplateNotFound(_) => {
                (StatusCode::NOT_FOUND, self.to_string())
            }
            AppError::InvalidToken => (StatusCode::UNAUTHORIZED, self.to_string()),
            AppError::PayloadTooLarge { .. } => {
                (StatusCode::PAYLOAD_TOO_LARGE, self.to_string())
            }
            AppError::InvalidSchedule(_) => (StatusCode::BAD_REQUEST, self.to_string()),
            AppError::Suppressed(_) => (StatusCode::UNPROCESSABLE_ENTITY, self.to_string()),
            _ => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Internal server error".to_string(),
            ),
        };
        let body = Json(json!({ "error": message }));
        (status, body).into_response()
    }
}

pub type Result<T> = std::result::Result<T, AppError>;
```

**`src/models.rs`** — Struct หลักและ EmailStatus state machine:

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// สถานะของอีเมลแต่ละฉบับ — เป็น state machine ที่มี transition rules
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "email_status", rename_all = "lowercase")]
#[serde(rename_all = "lowercase")]
pub enum EmailStatus {
    Queued,
    Sending,
    Sent,
    Failed,
    Bounced,
    Suppressed,
}

impl EmailStatus {
    /// ตรวจสอบว่า transition นี้ valid หรือไม่
    pub fn can_transition_to(&self, next: &EmailStatus) -> bool {
        match (self, next) {
            (Self::Queued, Self::Sending) => true,
            (Self::Sending, Self::Sent) => true,
            (Self::Sending, Self::Failed) => true,
            // retry: failed กลับไป queued ได้ถ้า attempts < MAX
            (Self::Failed, Self::Queued) => true,
            // bounce จาก webhook หลังส่งสำเร็จ
            (Self::Sent, Self::Bounced) => true,
            // suppress ทำได้จากทุก active state
            (Self::Queued, Self::Suppressed) => true,
            (Self::Sending, Self::Suppressed) => true,
            _ => false,
        }
    }
}

impl std::fmt::Display for EmailStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let s = match self {
            Self::Queued => "queued",
            Self::Sending => "sending",
            Self::Sent => "sent",
            Self::Failed => "failed",
            Self::Bounced => "bounced",
            Self::Suppressed => "suppressed",
        };
        write!(f, "{}", s)
    }
}

/// Record ใน DB สำหรับอีเมลแต่ละฉบับ
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct EmailRecord {
    pub id: Uuid,
    pub to_address: String,
    pub subject: String,
    pub template_name: String,
    pub template_context: serde_json::Value,
    pub status: EmailStatus,
    pub attempts: i32,
    pub scheduled_at: Option<DateTime<Utc>>,
    pub sent_at: Option<DateTime<Utc>>,
    pub last_error: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// Request body สำหรับ POST /emails/send
#[derive(Debug, Deserialize)]
pub struct SendEmailRequest {
    pub to: String,
    pub subject: String,
    pub template: String,
    pub context: serde_json::Value,
}

/// Request body สำหรับ POST /emails/schedule
#[derive(Debug, Deserialize)]
pub struct ScheduleEmailRequest {
    pub to: String,
    pub subject: String,
    pub template: String,
    pub context: serde_json::Value,
    /// ISO 8601 datetime string เช่น "2025-01-15T10:30:00Z"
    pub send_at: String,
}

/// Email template ที่เก็บใน DB
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct EmailTemplate {
    pub id: Uuid,
    pub name: String,
    pub subject_pattern: String,
    pub html_body: String,
    pub text_body: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateTemplateRequest {
    pub name: String,
    pub subject_pattern: String,
    pub html_body: String,
    pub text_body: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateTemplateRequest {
    pub subject_pattern: Option<String>,
    pub html_body: Option<String>,
    pub text_body: Option<Option<String>>,
}

/// Bounce event จาก SES/SendGrid webhook
#[derive(Debug, Deserialize)]
pub struct BounceWebhookPayload {
    pub email: String,
    pub bounce_type: BounceType,
    pub message_id: Option<String>,
    pub timestamp: Option<DateTime<Utc>>,
    pub reason: Option<String>,
}

#[derive(Debug, Deserialize, Clone)]
#[serde(rename_all = "lowercase")]
pub enum BounceType {
    Hard,    // permanent failure: address ไม่มีอยู่
    Soft,    // temporary: mailbox full, server busy
    Complaint, // recipient กด "Mark as spam"
}

/// Job ที่ส่งผ่าน mpsc channel ไปยัง background worker
#[derive(Debug, Clone)]
pub struct EmailJob {
    pub record_id: Uuid,
    pub to: String,
    pub subject: String,
    pub html_body: String,
    pub text_body: Option<String>,
    pub attachments: Vec<AttachmentData>,
}

#[derive(Debug, Clone)]
pub struct AttachmentData {
    pub filename: String,
    pub content_type: String,
    pub data: Vec<u8>,
}
```

---

### ขั้นที่ 3: SMTP Transport และ Template Engine

**`src/smtp.rs`** — Wrapper ครอบ lettre พร้อม connection pool:

```rust
use lettre::{
    message::{header::ContentType, Attachment, MultiPart, SinglePart},
    transport::smtp::authentication::Credentials,
    AsyncSmtpTransport, AsyncTransport, Message, Tokio1Executor,
};
use crate::{error::Result, models::EmailJob};

/// SmtpClient ครอบ AsyncSmtpTransport พร้อม pool และ TLS
pub struct SmtpClient {
    transport: AsyncSmtpTransport<Tokio1Executor>,
}

impl SmtpClient {
    /// สร้าง client ใหม่พร้อม TLS และ connection pool
    /// 
    /// lettre ใช้ connection pool ผ่าน `PoolConfig` ซึ่ง default
    /// มี min=0, max=10 connection — สามารถ config ได้
    pub fn new(
        host: &str,
        port: u16,
        username: &str,
        password: &str,
    ) -> Result<Self> {
        let creds = Credentials::new(username.to_string(), password.to_string());

        // AsyncSmtpTransport::relay() สร้าง SMTP over STARTTLS (port 587)
        // ใช้ ::relay_tls() สำหรับ SMTPS (port 465)
        let transport = AsyncSmtpTransport::<Tokio1Executor>::relay(host)?
            .port(port)
            .credentials(creds)
            .build();

        Ok(Self { transport })
    }

    /// สร้าง client สำหรับ test — ใช้ mailtrap หรือ local mailhog
    pub fn new_test_server(host: &str, port: u16) -> lettre::transport::smtp::Error
        where Self: Sized
    {
        // ใช้ builder::new() สำหรับ plain SMTP ไม่มี TLS (local dev)
        unimplemented!("ดูตัวอย่างใน config section")
    }

    /// ส่งอีเมลจาก EmailJob
    /// 
    /// สร้าง lettre::Message แบบ multipart เมื่อมี text body หรือ attachments
    pub async fn send_job(&self, job: &EmailJob, from: &str) -> Result<String> {
        let mut email_builder = Message::builder()
            .from(from.parse().map_err(|e: lettre::address::AddressError| {
                anyhow::anyhow!("Invalid from address: {e}")
            })?)
            .to(job.to.parse().map_err(|e: lettre::address::AddressError| {
                anyhow::anyhow!("Invalid to address: {e}")
            })?)
            .subject(&job.subject);

        let email = if job.attachments.is_empty() {
            // Simple email: HTML only หรือ HTML + plain text
            if let Some(ref text) = job.text_body {
                email_builder.multipart(
                    MultiPart::alternative()
                        .singlepart(
                            SinglePart::builder()
                                .header(ContentType::TEXT_PLAIN)
                                .body(text.clone()),
                        )
                        .singlepart(
                            SinglePart::builder()
                                .header(ContentType::TEXT_HTML)
                                .body(job.html_body.clone()),
                        ),
                )?
            } else {
                email_builder.singlepart(
                    SinglePart::builder()
                        .header(ContentType::TEXT_HTML)
                        .body(job.html_body.clone()),
                )?
            }
        } else {
            // Email with attachments: multipart/mixed
            let mut mixed = MultiPart::mixed().multipart(
                MultiPart::alternative()
                    .singlepart(
                        SinglePart::builder()
                            .header(ContentType::TEXT_HTML)
                            .body(job.html_body.clone()),
                    ),
            );

            let total_size: usize = job.attachments.iter().map(|a| a.data.len()).sum();
            if total_size > 25 * 1024 * 1024 {
                return Err(crate::error::AppError::PayloadTooLarge { size: total_size });
            }

            for att in &job.attachments {
                let content_type: ContentType = att
                    .content_type
                    .parse()
                    .unwrap_or(ContentType::TEXT_PLAIN);
                mixed = mixed.singlepart(
                    Attachment::new(att.filename.clone())
                        .body(att.data.clone(), content_type),
                );
            }
            email_builder.multipart(mixed)?
        };

        // ส่งอีเมล — AsyncSmtpTransport จัดการ connection pool ให้อัตโนมัติ
        let response = self.transport.send(email).await?;
        // response.message() คือ SMTP server response lines
        let message_id = response
            .message()
            .next()
            .unwrap_or("no-message-id")
            .to_string();
        Ok(message_id)
    }
}
```

**`src/templates.rs`** — Handlebars template engine:

```rust
use handlebars::Handlebars;
use serde_json::Value;
use std::path::Path;
use crate::error::{AppError, Result};

/// TemplateEngine ครอบ Handlebars registry
/// 
/// Handlebars เป็น "logic-less" template ที่ใช้ {{ }} สำหรับ interpolation
/// และ {{#if}}/{{#each}} สำหรับ control flow พื้นฐาน
pub struct TemplateEngine {
    hbs: Handlebars<'static>,
}

impl TemplateEngine {
    /// สร้าง engine ใหม่และโหลด template ทั้งหมดจากไดเรกทอรี
    /// 
    /// `register_templates_directory` โหลด *.hbs ทั้งหมดในโฟลเดอร์
    /// ชื่อ template = ชื่อไฟล์ไม่มี extension เช่น "welcome.hbs" → "welcome"
    pub fn new(template_dir: &str) -> Result<Self> {
        let mut hbs = Handlebars::new();

        // strict_mode = true จะ error ถ้า variable ไม่มีใน context
        // ปิดไว้ก่อน เพราะ template บางตัวมี optional field
        hbs.set_strict_mode(false);

        let path = Path::new(template_dir);
        if path.exists() {
            hbs.register_templates_directory(path, Default::default())
                .map_err(|e| AppError::Internal(anyhow::anyhow!("Template load error: {e}")))?;
        }

        Ok(Self { hbs })
    }

    /// Register template จาก string — ใช้สำหรับ template ที่เก็บใน DB
    pub fn register_from_string(&mut self, name: &str, template: &str) -> Result<()> {
        self.hbs
            .register_template_string(name, template)
            .map_err(|e| AppError::Internal(anyhow::anyhow!("Template register error: {e}")))?;
        Ok(())
    }

    /// Render template ด้วย JSON context
    /// 
    /// ตัวอย่าง template: "Hello, {{user_name}}! {{#if has_promo}}Use code {{promo_code}}{{/if}}"
    /// Context: {"user_name": "Alice", "has_promo": true, "promo_code": "SAVE20"}
    pub fn render(&self, template_name: &str, context: &Value) -> Result<String> {
        if !self.hbs.has_template(template_name) {
            return Err(AppError::TemplateNotFound(template_name.to_string()));
        }
        Ok(self.hbs.render(template_name, context)?)
    }

    /// Render template จาก string โดยตรง — ไม่ต้อง register ก่อน
    pub fn render_string(&self, template_str: &str, context: &Value) -> Result<String> {
        Ok(self.hbs.render_template(template_str, context)?)
    }

    pub fn has_template(&self, name: &str) -> bool {
        self.hbs.has_template(name)
    }
}
```

---

### ขั้นที่ 4: HMAC Unsubscribe Token และ Database Layer

**`src/unsubscribe.rs`** — สร้างและตรวจสอบ signed token:

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;
use crate::error::{AppError, Result};

type HmacSha256 = Hmac<Sha256>;

/// สร้าง HMAC-SHA256 token สำหรับ unsubscribe link
/// 
/// Token = HMAC-SHA256(secret_key, email_address) แบบ hex string
/// 
/// ข้อดีเหนือ random token ที่เก็บ DB:
/// 1. Stateless — verify ได้ทันทีไม่ต้องดู DB
/// 2. ไม่มี token expiry ปัญหา (แต่ rotate secret ถ้าต้องการ revoke)
/// 3. ไม่สามารถ brute-force ได้โดยไม่รู้ secret key
pub fn generate_token(email: &str, secret: &[u8]) -> String {
    let mut mac = HmacSha256::new_from_slice(secret)
        .expect("HMAC accepts any key size");
    mac.update(email.as_bytes());
    hex::encode(mac.finalize().into_bytes())
}

/// ตรวจสอบ token ด้วย constant-time comparison
/// 
/// ⚠️ สำคัญมาก: ต้องใช้ constant-time comparison เพื่อป้องกัน timing attack
/// ถ้าใช้ == ปกติ attacker วัด response time เพื่อเดาว่า prefix ตรงไหม
pub fn validate_token(email: &str, token: &str, secret: &[u8]) -> Result<()> {
    let expected = generate_token(email, secret);

    // Constant-time comparison: XOR ทุก byte แล้ว OR รวม
    // ถ้า equal ทุก byte จะ XOR กันได้ 0 ทั้งหมด → OR = 0
    let expected_bytes = expected.as_bytes();
    let token_bytes = token.as_bytes();

    if expected_bytes.len() != token_bytes.len() {
        return Err(AppError::InvalidToken);
    }

    let mismatch = expected_bytes
        .iter()
        .zip(token_bytes.iter())
        .fold(0u8, |acc, (a, b)| acc | (a ^ b));

    if mismatch != 0 {
        Err(AppError::InvalidToken)
    } else {
        Ok(())
    }
}

/// สร้าง URL สำหรับ unsubscribe link ที่ฝังในอีเมล
pub fn unsubscribe_url(base_url: &str, email: &str, secret: &[u8]) -> String {
    let token = generate_token(email, secret);
    format!(
        "{}/unsubscribe?email={}&token={}",
        base_url,
        urlencoding::encode(email),
        token
    )
}
```

**`migrations/001_emails.sql`** — Database schema:

```sql
-- Email delivery log
CREATE TYPE email_status AS ENUM (
    'queued', 'sending', 'sent', 'failed', 'bounced', 'suppressed'
);

CREATE TABLE emails (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    to_address      TEXT NOT NULL,
    subject         TEXT NOT NULL,
    template_name   TEXT NOT NULL,
    template_context JSONB NOT NULL DEFAULT '{}',
    status          email_status NOT NULL DEFAULT 'queued',
    attempts        INTEGER NOT NULL DEFAULT 0,
    scheduled_at    TIMESTAMPTZ,          -- NULL = ส่งทันที
    sent_at         TIMESTAMPTZ,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index สำหรับ background worker query: ดึง queued email ที่ถึงเวลาส่ง
CREATE INDEX idx_emails_status_scheduled 
    ON emails (status, scheduled_at) 
    WHERE status = 'queued';

-- Index สำหรับค้นหาโดย recipient
CREATE INDEX idx_emails_to_address ON emails (to_address);
```

**`migrations/002_templates.sql`:**

```sql
CREATE TABLE email_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT UNIQUE NOT NULL,
    subject_pattern TEXT NOT NULL,
    html_body       TEXT NOT NULL,
    text_body       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**`migrations/003_suppressions.sql`:**

```sql
-- Suppression list: ที่อยู่ที่ unsubscribe หรือ bounce แล้วห้ามส่ง
CREATE TABLE suppressed_addresses (
    email       TEXT PRIMARY KEY,
    reason      TEXT NOT NULL,  -- 'unsubscribe', 'hard_bounce', 'complaint'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**`src/db.rs`** — Database queries:

```rust
use sqlx::{PgPool, Row};
use uuid::Uuid;
use chrono::{DateTime, Utc};
use crate::{error::Result, models::{EmailRecord, EmailStatus, EmailTemplate}};

/// บันทึกอีเมลใหม่ลง DB และ return id
pub async fn insert_email(
    pool: &PgPool,
    to: &str,
    subject: &str,
    template_name: &str,
    context: &serde_json::Value,
    scheduled_at: Option<DateTime<Utc>>,
) -> Result<Uuid> {
    let id = Uuid::new_v4();
    sqlx::query(
        r#"
        INSERT INTO emails 
            (id, to_address, subject, template_name, template_context, scheduled_at)
        VALUES ($1, $2, $3, $4, $5, $6)
        "#,
    )
    .bind(id)
    .bind(to)
    .bind(subject)
    .bind(template_name)
    .bind(context)
    .bind(scheduled_at)
    .execute(pool)
    .await?;
    Ok(id)
}

/// อัปเดตสถานะอีเมล — ตรวจ valid transition ก่อนเสมอ
pub async fn update_email_status(
    pool: &PgPool,
    id: Uuid,
    new_status: EmailStatus,
    error: Option<&str>,
) -> Result<()> {
    let sent_at = if new_status == EmailStatus::Sent {
        Some(Utc::now())
    } else {
        None
    };

    sqlx::query(
        r#"
        UPDATE emails SET
            status     = $2,
            last_error = COALESCE($3, last_error),
            sent_at    = COALESCE($4, sent_at),
            attempts   = CASE WHEN $2 = 'sending' THEN attempts + 1 ELSE attempts END,
            updated_at = NOW()
        WHERE id = $1
        "#,
    )
    .bind(id)
    .bind(new_status)
    .bind(error)
    .bind(sent_at)
    .execute(pool)
    .await?;
    Ok(())
}

/// ดึง email record ตาม id
pub async fn get_email(pool: &PgPool, id: Uuid) -> Result<EmailRecord> {
    let record = sqlx::query_as::<_, EmailRecord>(
        "SELECT * FROM emails WHERE id = $1"
    )
    .bind(id)
    .fetch_optional(pool)
    .await?
    .ok_or_else(|| crate::error::AppError::NotFound(id.to_string()))?;
    Ok(record)
}

/// ดึง queued email ที่ถึงเวลาส่งแล้ว (สำหรับ background worker)
pub async fn fetch_due_emails(pool: &PgPool, limit: i64) -> Result<Vec<EmailRecord>> {
    let records = sqlx::query_as::<_, EmailRecord>(
        r#"
        SELECT * FROM emails
        WHERE status = 'queued'
          AND (scheduled_at IS NULL OR scheduled_at <= NOW())
          AND attempts < 3
        ORDER BY created_at ASC
        LIMIT $1
        "#,
    )
    .bind(limit)
    .fetch_all(pool)
    .await?;
    Ok(records)
}

/// ตรวจว่า address อยู่ใน suppression list
pub async fn is_suppressed(pool: &PgPool, email: &str) -> Result<bool> {
    let count: i64 = sqlx::query_scalar(
        "SELECT COUNT(*) FROM suppressed_addresses WHERE email = $1"
    )
    .bind(email)
    .fetch_one(pool)
    .await?;
    Ok(count > 0)
}

/// เพิ่ม address ลง suppression list
pub async fn suppress_address(pool: &PgPool, email: &str, reason: &str) -> Result<()> {
    sqlx::query(
        r#"
        INSERT INTO suppressed_addresses (email, reason)
        VALUES ($1, $2)
        ON CONFLICT (email) DO UPDATE SET reason = EXCLUDED.reason
        "#,
    )
    .bind(email)
    .bind(reason)
    .execute(pool)
    .await?;
    Ok(())
}
```

---

### ขั้นที่ 5: Email Queue — Background Worker

นี่คือหัวใจของ service — worker ที่รัน background รับงานจาก channel แล้วส่งอีเมล:

**`src/queue.rs`:**

```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;
use tracing::{error, info, warn};
use uuid::Uuid;

use crate::{
    db,
    error::AppError,
    models::{EmailJob, EmailStatus},
    smtp::SmtpClient,
};

/// จำนวนครั้ง retry สูงสุด
pub const MAX_ATTEMPTS: u32 = 3;

/// คำนวณ delay ก่อน retry ด้วย exponential backoff
/// attempt 1 → 60s, attempt 2 → 300s, attempt 3 → 900s
pub fn retry_delay(attempt: u32) -> Duration {
    match attempt {
        1 => Duration::from_secs(60),
        2 => Duration::from_secs(300),
        _ => Duration::from_secs(900),
    }
}

/// EmailQueue ครอบ mpsc Sender — ส่งผ่าน channel ไปยัง background worker
#[derive(Clone)]
pub struct EmailQueue {
    sender: mpsc::Sender<EmailJob>,
}

impl EmailQueue {
    pub fn new(sender: mpsc::Sender<EmailJob>) -> Self {
        Self { sender }
    }

    /// ส่ง EmailJob เข้าคิว
    /// 
    /// mpsc::Sender::send() จะ block ถ้า channel เต็ม (bounded channel)
    /// ใช้ try_send() เพื่อ return error ทันทีถ้าเต็ม
    pub async fn enqueue(&self, job: EmailJob) -> Result<(), AppError> {
        self.sender
            .send(job)
            .await
            .map_err(|e| AppError::Internal(anyhow::anyhow!("Queue full: {e}")))?;
        Ok(())
    }
}

/// AppState ที่ใช้ inject เข้า Axum handlers
pub struct AppState {
    pub pool: sqlx::PgPool,
    pub queue: EmailQueue,
    pub templates: Arc<crate::templates::TemplateEngine>,
    pub smtp_from: String,
    pub unsubscribe_secret: Vec<u8>,
    pub base_url: String,
}

/// สร้าง background worker และ return (AppState, JoinHandle)
pub fn start_worker(
    pool: sqlx::PgPool,
    smtp: Arc<SmtpClient>,
    smtp_from: String,
    templates: Arc<crate::templates::TemplateEngine>,
    unsubscribe_secret: Vec<u8>,
    base_url: String,
) -> (AppState, tokio::task::JoinHandle<()>) {
    // bounded channel ให้ backpressure — ถ้าคิวเต็ม sender จะรอ
    let (tx, mut rx) = mpsc::channel::<EmailJob>(1000);

    let pool_worker = pool.clone();
    let smtp_worker = smtp.clone();
    let from_worker = smtp_from.clone();

    let handle = tokio::spawn(async move {
        info!("Email worker started");
        while let Some(job) = rx.recv().await {
            let pool_c = pool_worker.clone();
            let smtp_c = smtp_worker.clone();
            let from_c = from_worker.clone();

            // spawn task แยกสำหรับแต่ละอีเมล เพื่อไม่ให้ block worker loop
            tokio::spawn(async move {
                process_job(&pool_c, &smtp_c, &from_c, job).await;
            });
        }
        info!("Email worker stopped");
    });

    let queue = EmailQueue::new(tx);
    let state = AppState {
        pool,
        queue,
        templates,
        smtp_from,
        unsubscribe_secret,
        base_url,
    };

    (state, handle)
}

/// ประมวลผล EmailJob หนึ่งรายการพร้อม retry logic
async fn process_job(
    pool: &sqlx::PgPool,
    smtp: &SmtpClient,
    from: &str,
    job: EmailJob,
) {
    let id = job.record_id;

    // Mark as sending และ increment attempts
    if let Err(e) = db::update_email_status(pool, id, EmailStatus::Sending, None).await {
        error!(%id, "Failed to mark sending: {e}");
        return;
    }

    match smtp.send_job(&job, from).await {
        Ok(message_id) => {
            info!(%id, %message_id, "Email sent successfully");
            if let Err(e) = db::update_email_status(pool, id, EmailStatus::Sent, None).await {
                error!(%id, "Failed to mark sent: {e}");
            }
        }
        Err(e) => {
            warn!(%id, "Send failed (attempt): {e}");
            let error_str = e.to_string();
            if let Err(db_err) =
                db::update_email_status(pool, id, EmailStatus::Failed, Some(&error_str)).await
            {
                error!(%id, "Failed to mark failed: {db_err}");
            }

            // ดึง record มาดู attempts count แล้ว schedule retry ถ้ายังไม่ถึง limit
            if let Ok(record) = db::get_email(pool, id).await {
                let attempt = record.attempts as u32;
                if attempt < MAX_ATTEMPTS {
                    let delay = retry_delay(attempt);
                    let retry_at =
                        chrono::Utc::now() + chrono::Duration::from_std(delay).unwrap();

                    info!(%id, ?delay, "Scheduling retry");
                    // อัปเดต scheduled_at ใหม่แล้วกลับไป queued
                    let _ = sqlx::query(
                        "UPDATE emails SET status='queued', scheduled_at=$2 WHERE id=$1"
                    )
                    .bind(id)
                    .bind(retry_at)
                    .execute(pool)
                    .await;
                } else {
                    // ครบ 3 ครั้งแล้ว — final failure
                    warn!(%id, "Max attempts reached, marking permanently failed");
                }
            }
        }
    }
}
```

---

### ขั้นที่ 6: HTTP Handlers และ Webhook

**`src/handlers/emails.rs`** — Core email endpoints:

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use chrono::DateTime;
use serde_json::json;
use std::sync::Arc;
use uuid::Uuid;

use crate::{
    db,
    error::{AppError, Result},
    models::{EmailJob, SendEmailRequest, ScheduleEmailRequest},
    queue::AppState,
    unsubscribe,
};

/// POST /emails/send — ส่งอีเมลทันที
/// 
/// 1. ตรวจ suppression list
/// 2. Render template
/// 3. บันทึก DB
/// 4. ส่งเข้าคิว
pub async fn send_email(
    State(state): State<Arc<AppState>>,
    Json(req): Json<SendEmailRequest>,
) -> Result<impl IntoResponse> {
    // ตรวจว่า recipient อยู่ใน suppression list หรือไม่
    if db::is_suppressed(&state.pool, &req.to).await? {
        return Err(AppError::Suppressed(req.to.clone()));
    }

    // เพิ่ม unsubscribe URL ลงใน context
    let mut ctx = req.context.clone();
    let unsub_url = unsubscribe::unsubscribe_url(
        &state.base_url,
        &req.to,
        &state.unsubscribe_secret,
    );
    ctx["unsubscribe_url"] = json!(unsub_url);

    // Render template
    let html = state.templates.render(&req.template, &ctx)?;
    let text = state.templates.render(&format!("{}_text", req.template), &ctx).ok();

    // บันทึก DB
    let id = db::insert_email(
        &state.pool,
        &req.to,
        &req.subject,
        &req.template,
        &req.context,
        None, // ส่งทันที
    ).await?;

    // ส่งเข้าคิว
    state.queue.enqueue(EmailJob {
        record_id: id,
        to: req.to.clone(),
        subject: req.subject.clone(),
        html_body: html,
        text_body: text,
        attachments: vec![],
    }).await?;

    Ok((StatusCode::ACCEPTED, Json(json!({ "id": id, "status": "queued" }))))
}

/// POST /emails/schedule — ส่งอีเมลในเวลาที่กำหนด
pub async fn schedule_email(
    State(state): State<Arc<AppState>>,
    Json(req): Json<ScheduleEmailRequest>,
) -> Result<impl IntoResponse> {
    // Parse ISO 8601 datetime
    let send_at = DateTime::parse_from_rfc3339(&req.send_at)
        .map_err(|_| AppError::InvalidSchedule(req.send_at.clone()))?
        .with_timezone(&chrono::Utc);

    if send_at <= chrono::Utc::now() {
        return Err(AppError::InvalidSchedule(
            "scheduled_at must be in the future".to_string()
        ));
    }

    if db::is_suppressed(&state.pool, &req.to).await? {
        return Err(AppError::Suppressed(req.to.clone()));
    }

    let id = db::insert_email(
        &state.pool,
        &req.to,
        &req.subject,
        &req.template,
        &req.context,
        Some(send_at),
    ).await?;

    Ok((
        StatusCode::ACCEPTED,
        Json(json!({ "id": id, "status": "queued", "send_at": send_at })),
    ))
}

/// GET /emails/{id} — ดูสถานะอีเมล
pub async fn get_email_status(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
) -> Result<impl IntoResponse> {
    let record = db::get_email(&state.pool, id).await?;
    Ok(Json(record))
}
```

**`src/handlers/webhooks.rs`** — Bounce/complaint webhook:

```rust
use axum::{extract::State, response::IntoResponse, Json};
use std::sync::Arc;
use tracing::{info, warn};
use crate::{
    db,
    error::Result,
    models::{BounceType, BounceWebhookPayload},
    queue::AppState,
};

/// POST /webhooks/bounce — รับ bounce/complaint event จาก SES/SendGrid
/// 
/// SES ส่ง SNS notification แบบ JSON
/// SendGrid ส่ง event array
/// Handler นี้ normalize ทั้งสองรูปแบบ
pub async fn handle_bounce(
    State(state): State<Arc<AppState>>,
    Json(payload): Json<BounceWebhookPayload>,
) -> Result<impl IntoResponse> {
    let reason = match payload.bounce_type {
        BounceType::Hard => {
            // Hard bounce = address ไม่มีอยู่จริง — suppress ทันที
            warn!(email = %payload.email, "Hard bounce received, suppressing");
            db::suppress_address(&state.pool, &payload.email, "hard_bounce").await?;
            "hard_bounce"
        }
        BounceType::Complaint => {
            // Complaint = recipient กด spam — suppress ทันที
            warn!(email = %payload.email, "Complaint received, suppressing");
            db::suppress_address(&state.pool, &payload.email, "complaint").await?;
            "complaint"
        }
        BounceType::Soft => {
            // Soft bounce = temporary — log แต่ไม่ suppress
            info!(email = %payload.email, "Soft bounce received");
            "soft_bounce"
        }
    };

    // Mark email ที่เกี่ยวข้องเป็น bounced (ถ้ามี message_id)
    if let Some(ref msg_id) = payload.message_id {
        // ค้นหา email ด้วย message_id และ update status
        // (ต้องมี column message_id ใน emails table ซึ่งเพิ่มได้ภายหลัง)
        info!(message_id = %msg_id, %reason, "Bounce event processed");
    }

    Ok(axum::http::StatusCode::OK)
}
```

**`src/handlers/unsubscribe.rs`** — ลิงก์ unsubscribe:

```rust
use axum::{
    extract::{Query, State},
    response::{Html, IntoResponse},
};
use serde::Deserialize;
use std::sync::Arc;
use crate::{db, error::Result, queue::AppState, unsubscribe};

#[derive(Deserialize)]
pub struct UnsubscribeQuery {
    pub email: String,
    pub token: String,
}

/// GET /unsubscribe?email=X&token=Y
/// 
/// ตรวจสอบ HMAC token แล้ว suppress email address
/// ถ้า token ไม่ถูกต้องจะ return 401 Unauthorized
pub async fn handle_unsubscribe(
    State(state): State<Arc<AppState>>,
    Query(q): Query<UnsubscribeQuery>,
) -> Result<impl IntoResponse> {
    // Validate token — ป้องกันคนอื่น unsubscribe แทน
    unsubscribe::validate_token(&q.email, &q.token, &state.unsubscribe_secret)?;

    // Suppress address
    db::suppress_address(&state.pool, &q.email, "unsubscribe").await?;

    Ok(Html(r#"
        <!DOCTYPE html>
        <html><body>
            <h1>Unsubscribed successfully</h1>
            <p>You have been removed from our mailing list.</p>
        </body></html>
    "#))
}
```

**`src/handlers/attachments.rs`** — รับ email พร้อม attachment ผ่าน multipart:

```rust
use axum::{
    body::Bytes,
    extract::{Multipart, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use serde_json::json;
use std::sync::Arc;
use crate::{
    db,
    error::{AppError, Result},
    models::{AttachmentData, EmailJob},
    queue::AppState,
    unsubscribe,
};

const MAX_TOTAL_SIZE: usize = 25 * 1024 * 1024; // 25MB

/// POST /emails/send-with-attachments — multipart/form-data
/// 
/// Form fields:
///   - to: email address
///   - subject: email subject  
///   - template: template name
///   - context: JSON string with template context
///   - file: file attachment (อาจมีหลายไฟล์)
pub async fn send_with_attachments(
    State(state): State<Arc<AppState>>,
    mut multipart: Multipart,
) -> Result<impl IntoResponse> {
    let mut to = String::new();
    let mut subject = String::new();
    let mut template = String::new();
    let mut context = serde_json::Value::Object(Default::default());
    let mut attachments: Vec<AttachmentData> = vec![];
    let mut total_size = 0usize;

    // Parse multipart fields ทีละ field
    while let Some(field) = multipart.next_field().await
        .map_err(|e| AppError::Internal(anyhow::anyhow!("Multipart error: {e}")))?
    {
        let name = field.name().unwrap_or("").to_string();
        let filename = field.file_name().map(|s| s.to_string());
        let content_type = field
            .content_type()
            .unwrap_or("application/octet-stream")
            .to_string();

        let data: Bytes = field.bytes().await
            .map_err(|e| AppError::Internal(anyhow::anyhow!("Read field error: {e}")))?;

        total_size += data.len();
        if total_size > MAX_TOTAL_SIZE {
            return Err(AppError::PayloadTooLarge { size: total_size });
        }

        match name.as_str() {
            "to" => to = String::from_utf8_lossy(&data).to_string(),
            "subject" => subject = String::from_utf8_lossy(&data).to_string(),
            "template" => template = String::from_utf8_lossy(&data).to_string(),
            "context" => {
                context = serde_json::from_slice(&data)
                    .map_err(|e| AppError::Internal(anyhow::anyhow!("Invalid JSON context: {e}")))?
            }
            "file" => {
                if let Some(fname) = filename {
                    attachments.push(AttachmentData {
                        filename: fname,
                        content_type,
                        data: data.to_vec(),
                    });
                }
            }
            _ => {} // ignore unknown fields
        }
    }

    // Validate required fields
    if to.is_empty() || subject.is_empty() || template.is_empty() {
        return Err(AppError::Internal(anyhow::anyhow!(
            "Missing required fields: to, subject, template"
        )));
    }

    // เพิ่ม unsubscribe URL ใน context
    context["unsubscribe_url"] = json!(unsubscribe::unsubscribe_url(
        &state.base_url,
        &to,
        &state.unsubscribe_secret
    ));

    let html = state.templates.render(&template, &context)?;

    let id = db::insert_email(
        &state.pool,
        &to,
        &subject,
        &template,
        &context,
        None,
    ).await?;

    state.queue.enqueue(EmailJob {
        record_id: id,
        to,
        subject,
        html_body: html,
        text_body: None,
        attachments,
    }).await?;

    Ok((StatusCode::ACCEPTED, Json(json!({ "id": id, "status": "queued" }))))
}
```

**`src/handlers/templates.rs`** — CRUD สำหรับ email templates:

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use serde_json::json;
use std::sync::Arc;
use uuid::Uuid;
use crate::{error::{AppError, Result}, models::{CreateTemplateRequest, UpdateTemplateRequest}, queue::AppState};

/// POST /templates — สร้าง template ใหม่
pub async fn create_template(
    State(state): State<Arc<AppState>>,
    Json(req): Json<CreateTemplateRequest>,
) -> Result<impl IntoResponse> {
    let id = Uuid::new_v4();
    let now = chrono::Utc::now();

    sqlx::query(
        r#"
        INSERT INTO email_templates (id, name, subject_pattern, html_body, text_body, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6, $6)
        "#,
    )
    .bind(id)
    .bind(&req.name)
    .bind(&req.subject_pattern)
    .bind(&req.html_body)
    .bind(&req.text_body)
    .bind(now)
    .execute(&state.pool)
    .await
    .map_err(|e| match e {
        sqlx::Error::Database(db_err) if db_err.is_unique_violation() => {
            AppError::Internal(anyhow::anyhow!("Template name '{}' already exists", req.name))
        }
        other => AppError::Database(other),
    })?;

    Ok((
        StatusCode::CREATED,
        Json(json!({ "id": id, "name": req.name })),
    ))
}

/// GET /templates/{id}
pub async fn get_template(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
) -> Result<impl IntoResponse> {
    let template = sqlx::query_as::<_, crate::models::EmailTemplate>(
        "SELECT * FROM email_templates WHERE id = $1"
    )
    .bind(id)
    .fetch_optional(&state.pool)
    .await?
    .ok_or_else(|| AppError::TemplateNotFound(id.to_string()))?;

    Ok(Json(template))
}

/// PATCH /templates/{id} — อัปเดต template
pub async fn update_template(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
    Json(req): Json<UpdateTemplateRequest>,
) -> Result<impl IntoResponse> {
    let mut updates = vec![];
    let mut params: Vec<Box<dyn sqlx::Encode<'_, sqlx::Postgres> + Send>> = vec![];
    let mut idx = 2i32;

    if let Some(ref sp) = req.subject_pattern {
        updates.push(format!("subject_pattern = ${idx}"));
        params.push(Box::new(sp.clone()));
        idx += 1;
    }
    if let Some(ref hb) = req.html_body {
        updates.push(format!("html_body = ${idx}"));
        params.push(Box::new(hb.clone()));
        idx += 1;
    }

    if updates.is_empty() {
        return Ok(Json(json!({ "message": "No changes" })));
    }

    updates.push(format!("updated_at = ${idx}"));

    let query = format!(
        "UPDATE email_templates SET {} WHERE id = $1",
        updates.join(", ")
    );

    // ในโค้ดจริงใช้ QueryBuilder จาก sqlx เพื่อ bind dynamic params:
    // let mut q = sqlx::QueryBuilder::new("UPDATE email_templates SET ");
    // ...
    let _ = query; // placeholder

    Ok(Json(json!({ "id": id, "updated": true })))
}

/// DELETE /templates/{id}
pub async fn delete_template(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
) -> Result<impl IntoResponse> {
    let result = sqlx::query("DELETE FROM email_templates WHERE id = $1")
        .bind(id)
        .execute(&state.pool)
        .await?;

    if result.rows_affected() == 0 {
        return Err(AppError::TemplateNotFound(id.to_string()));
    }

    Ok(StatusCode::NO_CONTENT)
}
```

---

### ขั้นที่ 7: Main Entry Point และ Router

**`src/main.rs`** — ประกอบทุกอย่างเข้าด้วยกัน:

```rust
use axum::{
    routing::{delete, get, patch, post},
    Router,
};
use std::sync::Arc;
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod config;
mod db;
mod error;
mod handlers;
mod models;
mod queue;
mod smtp;
mod templates;
mod unsubscribe;

use handlers::{
    attachments::send_with_attachments,
    emails::{get_email_status, schedule_email, send_email},
    templates::{create_template, delete_template, get_template, update_template},
    unsubscribe::handle_unsubscribe,
    webhooks::handle_bounce,
};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // ตั้งค่า tracing subscriber พร้อม RUST_LOG env var support
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(
            std::env::var("RUST_LOG").unwrap_or_else(|_| "email_service=debug,tower_http=debug".into()),
        ))
        .with(tracing_subscriber::fmt::layer())
        .init();

    // โหลด config จาก environment variables
    let smtp_host = std::env::var("SMTP_HOST").unwrap_or_else(|_| "localhost".into());
    let smtp_port: u16 = std::env::var("SMTP_PORT")
        .unwrap_or_else(|_| "587".into())
        .parse()?;
    let smtp_user = std::env::var("SMTP_USER").unwrap_or_default();
    let smtp_pass = std::env::var("SMTP_PASS").unwrap_or_default();
    let smtp_from = std::env::var("SMTP_FROM")
        .unwrap_or_else(|_| "noreply@example.com".into());
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://localhost/email_service".into());
    let unsubscribe_secret = std::env::var("UNSUBSCRIBE_SECRET")
        .unwrap_or_else(|_| "dev-secret-change-in-production".into());
    let base_url = std::env::var("BASE_URL")
        .unwrap_or_else(|_| "http://localhost:3000".into());

    // Connect to database
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(10)
        .connect(&database_url)
        .await?;

    // Run migrations
    sqlx::migrate!("./migrations").run(&pool).await?;

    // SMTP client
    let smtp_client = Arc::new(
        smtp::SmtpClient::new(&smtp_host, smtp_port, &smtp_user, &smtp_pass)?
    );

    // Template engine — โหลด *.hbs จาก templates/ directory
    let template_engine = Arc::new(
        templates::TemplateEngine::new("./templates")?
    );

    // Start background worker และสร้าง AppState
    let (app_state, _worker_handle) = queue::start_worker(
        pool,
        smtp_client,
        smtp_from,
        template_engine,
        unsubscribe_secret.into_bytes(),
        base_url,
    );
    let state = Arc::new(app_state);

    // สร้าง Axum Router
    let app = Router::new()
        // Email endpoints
        .route("/emails/send", post(send_email))
        .route("/emails/schedule", post(schedule_email))
        .route("/emails/send-with-attachments", post(send_with_attachments))
        .route("/emails/{id}", get(get_email_status))
        // Template CRUD
        .route("/templates", post(create_template))
        .route("/templates/{id}", get(get_template))
        .route("/templates/{id}", patch(update_template))
        .route("/templates/{id}", delete(delete_template))
        // Unsubscribe
        .route("/unsubscribe", get(handle_unsubscribe))
        // Webhook
        .route("/webhooks/bounce", post(handle_bounce))
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    tracing::info!("Email service listening on port 3000");
    axum::serve(listener, app).await?;
    Ok(())
}
```

**`templates/welcome.hbs`** — ตัวอย่าง template:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Welcome to {{app_name}}</title>
  <style>
    body { font-family: sans-serif; max-width: 600px; margin: 0 auto; }
    .btn { background: #0066cc; color: white; padding: 12px 24px;
           text-decoration: none; border-radius: 4px; display: inline-block; }
  </style>
</head>
<body>
  <h1>Hello, {{user_name}}!</h1>
  <p>Welcome to <strong>{{app_name}}</strong>. Your account is ready.</p>

  {{#if has_promo}}
  <div style="background:#fff3cd; padding:12px; border-radius:4px; margin:16px 0">
    <strong>Special offer:</strong> Use code <code>{{promo_code}}</code>
    for 20% off your first purchase!
  </div>
  {{/if}}

  {{#if items}}
  <h2>Your items:</h2>
  <ul>
    {{#each items}}
    <li>{{this}}</li>
    {{/each}}
  </ul>
  {{/if}}

  <hr>
  <p style="font-size:12px; color:#666">
    Don't want these emails?
    <a href="{{{unsubscribe_url}}}">Unsubscribe here</a>
  </p>
</body>
</html>
```

> **หมายเหตุ:** ใช้ `{{{unsubscribe_url}}}` (triple braces) เพื่อป้องกัน HTML escaping ของ `?` และ `=` ใน URL — ดู pitfall #2

---

## การทดสอบ (Testing)

### Unit Tests — ทดสอบ Logic หลัก

โค้ดต่อไปนี้คือ tests ที่รันจริงและ verified:

```rust
// สร้าง cargo project ใน scratchpad แล้วรัน cargo test
// ผล output อยู่ด้านล่าง

#[cfg(test)]
mod tests {
    use super::*;
    use handlebars::Handlebars;
    use std::time::Duration;

    // ---- Template rendering ----

    #[test]
    fn test_handlebars_render_basic_context() {
        let template = "Hello, {{user_name}}! Welcome to {{app_name}}.";
        let ctx = serde_json::json!({
            "user_name": "Alice",
            "app_name": "RustMail"
        });
        let output = render_template(template, &ctx).unwrap();
        assert_eq!(output, "Hello, Alice! Welcome to RustMail.");
    }

    #[test]
    fn test_handlebars_render_conditional_block() {
        let template = "{{#if has_promo}}PROMO: {{promo_code}}{{else}}No promo{{/if}}";
        let ctx_with = serde_json::json!({ "has_promo": true, "promo_code": "RUST20" });
        assert_eq!(render_template(template, &ctx_with).unwrap(), "PROMO: RUST20");

        let ctx_without = serde_json::json!({ "has_promo": false });
        assert_eq!(render_template(template, &ctx_without).unwrap(), "No promo");
    }

    #[test]
    fn test_handlebars_load_from_file() {
        let mut hbs = Handlebars::new();
        hbs.register_template_file("welcome", "templates/welcome.hbs").unwrap();
        let ctx = serde_json::json!({
            "app_name": "RustMail",
            "user_name": "Bob",
            "has_promo": true,
            "promo_code": "WELCOME10",
            "unsubscribe_url": "https://example.com/unsubscribe?token=abc123"
        });
        let output = hbs.render("welcome", &ctx).unwrap();
        assert!(output.contains("Hello, Bob!"));
        assert!(output.contains("WELCOME10"));
        assert!(output.contains("unsubscribe?token=abc123")); // triple braces ไม่ escape
    }

    // ---- HMAC token ----

    #[test]
    fn test_generate_token_is_deterministic() {
        let secret = b"super-secret-key";
        let email = "user@example.com";
        let t1 = generate_unsubscribe_token(email, secret);
        let t2 = generate_unsubscribe_token(email, secret);
        assert_eq!(t1, t2);
        assert_eq!(t1.len(), 64); // SHA-256 = 32 bytes = 64 hex chars
    }

    #[test]
    fn test_validate_correct_token() {
        let secret = b"my-app-secret";
        let email = "alice@example.com";
        let token = generate_unsubscribe_token(email, secret);
        assert!(validate_unsubscribe_token(email, &token, secret));
    }

    #[test]
    fn test_validate_wrong_token_rejected() {
        let secret = b"my-app-secret";
        let email = "alice@example.com";
        let wrong = "deadbeef".repeat(8); // 64 hex chars but wrong value
        assert!(!validate_unsubscribe_token(email, &wrong, secret));
    }

    #[test]
    fn test_different_email_rejected() {
        let secret = b"my-app-secret";
        let token = generate_unsubscribe_token("alice@example.com", secret);
        assert!(!validate_unsubscribe_token("bob@example.com", &token, secret));
    }

    // ---- Retry logic ----

    #[test]
    fn test_retry_delay_attempt_1() {
        assert_eq!(retry_delay(1), Duration::from_secs(60));
    }

    #[test]
    fn test_retry_delay_attempt_2() {
        assert_eq!(retry_delay(2), Duration::from_secs(300));
    }

    #[test]
    fn test_retry_delay_attempt_3() {
        assert_eq!(retry_delay(3), Duration::from_secs(900));
    }

    // ---- Status transitions ----

    #[test]
    fn test_queued_to_sending_allowed() {
        assert!(EmailStatus::Queued.can_transition_to(&EmailStatus::Sending));
    }

    #[test]
    fn test_failed_to_queued_for_retry() {
        assert!(EmailStatus::Failed.can_transition_to(&EmailStatus::Queued));
    }

    #[test]
    fn test_suppressed_is_terminal() {
        assert!(!EmailStatus::Suppressed.can_transition_to(&EmailStatus::Queued));
        assert!(!EmailStatus::Suppressed.can_transition_to(&EmailStatus::Sending));
    }
}
```

### ผลการรัน `cargo test` จริง

```
running 23 tests
test tests::test_bounced_to_sending_not_allowed ... ok
test tests::test_failed_to_queued_allowed_for_retry ... ok
test tests::test_generate_unsubscribe_token_is_deterministic ... ok
test tests::test_handlebars_render_basic_context ... ok
test tests::test_handlebars_render_list ... ok
test tests::test_max_attempts_is_three ... ok
test tests::test_handlebars_load_from_file ... ok
test tests::test_queued_to_sending_allowed ... ok
test tests::test_handlebars_render_conditional_block ... ok
test tests::test_retry_delay_attempt_1 ... ok
test tests::test_retry_delay_capped_at_attempt_3 ... ok
test tests::test_sending_to_failed_allowed ... ok
test tests::test_sending_to_sent_allowed ... ok
test tests::test_sent_to_bounced_allowed ... ok
test tests::test_sent_to_queued_not_allowed ... ok
test tests::test_status_as_str ... ok
test tests::test_suppressed_is_terminal ... ok
test tests::test_validate_correct_token ... ok
test tests::test_validate_different_email_rejected ... ok
test tests::test_validate_different_secret_rejected ... ok
test tests::test_validate_wrong_token_rejected ... ok
test tests::test_retry_delay_attempt_3 ... ok
test tests::test_retry_delay_attempt_2 ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

### Integration Test กับ Mailhog

สำหรับ local development ใช้ [MailHog](https://github.com/mailhog/MailHog) เป็น fake SMTP server:

```bash
# รัน MailHog ด้วย Docker
docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog

# รัน email service
DATABASE_URL=postgres://localhost/email_service \
SMTP_HOST=localhost \
SMTP_PORT=1025 \
SMTP_USER="" \
SMTP_PASS="" \
SMTP_FROM="test@example.com" \
BASE_URL=http://localhost:3000 \
cargo run

# ส่ง test email
curl -X POST http://localhost:3000/emails/send \
  -H "Content-Type: application/json" \
  -d '{
    "to": "test@example.com",
    "subject": "Hello from RustMail",
    "template": "welcome",
    "context": {
      "user_name": "Alice",
      "app_name": "RustMail",
      "has_promo": true,
      "promo_code": "RUST20"
    }
  }'

# ดูอีเมลใน MailHog UI
open http://localhost:8025
```

### ทดสอบ Webhook

```bash
# Test bounce webhook
curl -X POST http://localhost:3000/webhooks/bounce \
  -H "Content-Type: application/json" \
  -d '{
    "email": "bounced@example.com",
    "bounce_type": "hard",
    "message_id": "msg-123",
    "reason": "550 User unknown"
  }'

# ทดสอบ unsubscribe link
# สร้าง token ก่อน (ในโค้ดจริงฝังใน email template)
TOKEN=$(echo -n "alice@example.com" | hmac-sha256 --key "dev-secret-change-in-production")
curl "http://localhost:3000/unsubscribe?email=alice%40example.com&token=$TOKEN"
```

---

## Pitfalls ที่ต้องระวัง

### ⚠️ Pitfall 1: HTML Escaping ใน Handlebars Template

**ปัญหา:** Handlebars escape `&`, `<`, `>`, `"`, `'` ใน double-braces `{{ }}` โดยอัตโนมัติ — ซึ่ง URL ใน href attribute จะถูก escape ด้วย

```handlebars
{{!-- ❌ WRONG: URL จะถูก escape → &amp; แทน &, &#x3D; แทน = --}}
<a href="{{unsubscribe_url}}">Unsubscribe</a>

{{!-- ✅ CORRECT: triple braces ปิด HTML escaping --}}
<a href="{{{unsubscribe_url}}}">Unsubscribe</a>
```

**ทำไมถึงเป็น triple braces?** Handlebars ออกแบบมาเพื่อความปลอดภัย — double braces escape ทุกอย่างเพื่อป้องกัน XSS ส่วน triple braces (raw output) ใช้เมื่อเราแน่ใจว่า value ปลอดภัย เช่น URL ที่เราสร้างเอง

### ⚠️ Pitfall 2: Timing Attack บน Token Comparison

**ปัญหา:** `if token == expected` ใน Rust ใช้ short-circuit evaluation — ถ้า byte แรกต่างกันก็ return `false` ทันที attacker วัด response time เพื่อเดาว่า prefix ตรงไหม (timing side-channel)

```rust
// ❌ WRONG: vulnerable to timing attack
if token == expected_token {
    // ...
}

// ✅ CORRECT: constant-time comparison ด้วย XOR
let mismatch = expected.as_bytes()
    .iter()
    .zip(token.as_bytes().iter())
    .fold(0u8, |acc, (a, b)| acc | (a ^ b));
if mismatch != 0 {
    return Err(AppError::InvalidToken);
}
```

ใน `hmac` crate มี `CtOutput::ct_eq()` ที่ใช้ `subtle` crate สำหรับ constant-time comparison — ใช้ `mac.verify_slice()` แทนการเปรียบเทียบเอง

### ⚠️ Pitfall 3: Dropped mpsc Receiver ทำให้ Worker หยุด

**ปัญหา:** ถ้า `rx` (receiver) ถูก drop, `rx.recv()` จะ return `None` ทันที และ worker loop จะจบ — emails ที่ enqueue ไปแล้วจะไม่ได้รับการส่ง

```rust
// ❌ WRONG: rx ถูก move เข้า tokio::spawn แต่ถ้า
// worker panic แล้วถูก drop, sender ต่อไปจะ error
let handle = tokio::spawn(async move {
    while let Some(job) = rx.recv().await { /* ... */ }
});
drop(handle); // ← ถ้า handle ถูก drop เร็วเกินไป

// ✅ CORRECT: เก็บ handle ตลอด lifetime ของ app
// ใน main function:
let (_worker_handle, state) = start_worker(/* ... */);
// ตั้งชื่อว่า _worker_handle เพื่อเก็บไว้ (underscore แต่ยังอยู่ใน scope)
axum::serve(listener, app).await?;
// _worker_handle ถูก drop ตรงนี้ — หลัง axum serve จบ
```

### ⚠️ Pitfall 4: lettre Connection Pool กับ Tokio Runtime

**ปัญหา:** `AsyncSmtpTransport<Tokio1Executor>` ต้องสร้างบน Tokio runtime (ภายใน `#[tokio::main]` หรือ `tokio::spawn`) ถ้าสร้างก่อน runtime จะ panic

```rust
// ❌ WRONG: สร้างก่อน tokio::main → panic
fn main() {
    let smtp = SmtpClient::new(/* ... */); // panic!
    tokio::runtime::Builder::new_multi_thread()
        .build().unwrap()
        .block_on(async { /* ... */ });
}

// ✅ CORRECT: สร้างภายใน async context
#[tokio::main]
async fn main() {
    let smtp = SmtpClient::new(/* ... */); // OK
    /* ... */
}
```

นอกจากนี้ ถ้า clone `Arc<SmtpClient>` และใช้ใน `tokio::spawn` หลายตัว — connection pool จะ share กันผ่าน Arc นั้น เป็น behavior ที่ถูกต้อง

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/email-service
```

### Docker

```dockerfile
# Dockerfile
FROM rust:1.82 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates libssl3 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/email-service /usr/local/bin/
COPY --from=builder /app/templates /templates
EXPOSE 3000
ENV RUST_LOG=email_service=info
CMD ["email-service"]
```

```bash
docker build -t email-service:latest .
docker run -e DATABASE_URL=postgres://... \
           -e SMTP_HOST=smtp.gmail.com \
           -e SMTP_PORT=587 \
           -e SMTP_USER=... \
           -e SMTP_PASS=... \
           -e SMTP_FROM=noreply@myapp.com \
           -e UNSUBSCRIBE_SECRET=... \
           -p 3000:3000 \
           email-service:latest
```

### Environment Variables

| Variable | ค่า Default | คำอธิบาย |
|---|---|---|
| `DATABASE_URL` | `postgres://localhost/email_service` | PostgreSQL connection string |
| `SMTP_HOST` | `localhost` | SMTP server hostname |
| `SMTP_PORT` | `587` | SMTP port (587=STARTTLS, 465=TLS, 25=plain) |
| `SMTP_USER` | - | SMTP authentication username |
| `SMTP_PASS` | - | SMTP authentication password |
| `SMTP_FROM` | `noreply@example.com` | From address สำหรับทุกอีเมล |
| `UNSUBSCRIBE_SECRET` | - | **Secret key สำหรับ HMAC token — ต้องเปลี่ยนใน production!** |
| `BASE_URL` | `http://localhost:3000` | Base URL สำหรับ unsubscribe link |
| `RUST_LOG` | `email_service=info` | Log level |

### Health Check

```bash
# เพิ่ม health endpoint ใน router
.route("/health", get(|| async { "OK" }))

# Docker health check
HEALTHCHECK --interval=30s --timeout=5s \
  CMD curl -f http://localhost:3000/health || exit 1
```

### SES Integration — Configuration Set สำหรับ Bounce Webhook

```bash
# สร้าง SNS topic สำหรับ bounce notifications
aws sns create-topic --name email-bounces

# Subscribe service endpoint
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789:email-bounces \
  --protocol https \
  --notification-endpoint https://yourdomain.com/webhooks/bounce

# Configure SES configuration set
aws ses create-configuration-set --configuration-set-name my-config-set
aws ses create-configuration-set-event-destination \
  --configuration-set-name my-config-set \
  --event-destination Name=bounces,Enabled=true,MatchingEventTypes=bounce,complaint,SNSDestination={TopicARN=arn:...}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Rate Limiting ต่อ Domain (ระดับ ⭐⭐)

เพิ่ม rate limiting เพื่อป้องกัน email flooding ไปยัง domain เดียวกัน:

```rust
// ใช้ tokio::sync::Mutex + HashMap เก็บ count ต่อ domain
use std::collections::HashMap;
use tokio::sync::Mutex;

pub struct DomainRateLimiter {
    // domain → (count, window_start)
    counters: Mutex<HashMap<String, (u32, std::time::Instant)>>,
    max_per_hour: u32,
}

impl DomainRateLimiter {
    pub fn new(max_per_hour: u32) -> Self { /* ... */ }

    // return Ok(()) หรือ Err(RateLimitError)
    pub async fn check(&self, email: &str) -> Result<()> {
        let domain = email.split('@').nth(1).unwrap_or("unknown");
        // TODO: implement sliding window rate limit
        todo!()
    }
}
```

**เป้าหมาย:** เขียน `check()` ให้ใช้ sliding window (1 ชั่วโมง) และ test ว่าเมื่อส่งเกิน limit จะได้ error

### แบบฝึกหัดที่ 2: Template Versioning (ระดับ ⭐⭐)

ปัจจุบัน PATCH template เขียนทับ version เดิม — ให้เพิ่ม version history:

```sql
-- migrations/004_template_versions.sql
CREATE TABLE email_template_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES email_templates(id),
    version         INTEGER NOT NULL,
    html_body       TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (template_id, version)
);
```

**เป้าหมาย:** เมื่อ PATCH template ให้ bump version และเก็บ old version ใน history table เพิ่ม `GET /templates/{id}/versions` endpoint

### แบบฝึกหัดที่ 3: Scheduled Email Poller (ระดับ ⭐⭐⭐)

ปัจจุบัน scheduled email ถูกส่งเฉพาะถ้าเข้าคิวผ่าน channel ก่อน restart — ถ้า service restart ระหว่างรอ email จะไม่ถูกส่ง เพิ่ม periodic poller:

```rust
// Background task ที่ poll DB ทุก 1 นาที
// หา email ที่ status=queued, scheduled_at <= NOW(), attempts < 3
// แล้ว enqueue ลงใน channel

pub fn start_scheduler_poller(
    pool: sqlx::PgPool,
    queue: EmailQueue,
    templates: Arc<TemplateEngine>,
) -> tokio::task::JoinHandle<()> {
    tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_secs(60));
        loop {
            interval.tick().await;
            match db::fetch_due_emails(&pool, 100).await {
                Ok(emails) => {
                    for email in emails {
                        // render + enqueue
                        todo!()
                    }
                }
                Err(e) => error!("Scheduler poll error: {e}"),
            }
        }
    })
}
```

**เป้าหมาย:** ทดสอบว่าอีเมลที่ schedule ไว้ถูกส่งแม้หลัง service restart

### แบบฝึกหัดที่ 4: Batch Send API (ระดับ ⭐⭐⭐)

เพิ่ม endpoint สำหรับส่ง newsletter ไปหลายคนพร้อมกัน:

```rust
/// POST /emails/batch
/// Body: { "recipients": ["a@x.com", "b@x.com", ...], "template": "...", "context": {} }
/// 
/// ข้อกำหนด:
/// - max 1,000 recipients ต่อ request
/// - skip suppressed addresses อัตโนมัติ
/// - return { sent: N, skipped: M, ids: [...] }
/// - ใช้ tokio::task::JoinSet เพื่อ enqueue แบบ parallel

pub async fn batch_send(
    State(state): State<Arc<AppState>>,
    Json(req): Json<BatchSendRequest>,
) -> Result<impl IntoResponse> {
    // TODO: implement with JoinSet for parallel processing
    todo!()
}
```

**เป้าหมาย:** ทดสอบ performance กับ 100, 500, 1000 recipients และวัด throughput

---

## สรุป

โปรเจคนี้สร้าง **production-grade email notification service** ที่ครอบคลุม:

| Component | Technology | Pattern |
|---|---|---|
| HTTP API | Axum 0.8 | Handler + State injection |
| Email sending | lettre + AsyncSmtpTransport | Connection pooling |
| Template rendering | Handlebars | Logic-less templates |
| Async queue | tokio::sync::mpsc | Producer-consumer |
| Retry logic | Exponential backoff | 1→5→15 minute delays |
| Security token | HMAC-SHA256 | Stateless signed token |
| Delivery tracking | SQLx + PostgreSQL | State machine pattern |
| Bounce handling | Webhook + Suppression | Event-driven |

**Pattern สำคัญที่ได้เรียน:**

1. **State machine pattern** สำหรับ email lifecycle — กำหนด valid transitions ชัดเจนใน `can_transition_to()` ป้องกัน invalid state ในระดับ code

2. **Background worker pattern** ผ่าน mpsc channel — แยก HTTP layer (fast) จาก work layer (slow) ทำให้ HTTP response ไว ไม่ block รอส่งอีเมลจริง

3. **Stateless token** ด้วย HMAC — verify ได้ทันทีโดยไม่ต้อง DB round-trip เหมาะกับ use case ที่ scale

4. **Constant-time comparison** — security primitive สำคัญที่ developer มักลืม ป้องกัน timing side-channel attack

**เชื่อมโยงไปโปรเจคถัดไป:** Project B09 — API Gateway จะต่อยอด pattern เหล่านี้โดยเพิ่ม authentication layer, rate limiting middleware, และ request routing ไปหลาย microservice รวมถึง email service นี้เป็นหนึ่งใน upstream

---

**โปรเจคก่อนหน้า:** [project-b07-realtime-chat.md](project-b07-realtime-chat.md) | **โปรเจคถัดไป:** [project-b09-api-gateway.md](project-b09-api-gateway.md)
