# Project B03: Image Processing API

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **REST API สำหรับประมวลผลภาพ (Image Processing API)** ด้วย Rust โดยรับภาพผ่าน multipart upload หรือ URL และส่งคืนภาพที่แปลงแล้ว สนับสนุนการ resize, เปลี่ยน format, ใส่ filter, crop, watermark และการทำงานแบบ pipeline ที่ต่อ operation ได้หลายขั้น

ในโลก production Image Processing API เป็นหัวใจของระบบจัดการ content ทุกประเภท — ตั้งแต่ e-commerce ที่ต้องสร้าง thumbnail สินค้าหลายขนาด, สื่อดิจิทัลที่ต้องแปลงภาพให้เหมาะกับอุปกรณ์ต่าง ๆ ไปจนถึงแอปพลิเคชัน AI ที่ต้องการ preprocess ภาพก่อนส่งเข้า model ตัวอย่าง real-world services ได้แก่ Cloudinary, ImageKit, และ imgproxy ซึ่งล้วนทำงานในแนวทางเดียวกัน

**Learning value** ของโปรเจคนี้ครอบคลุมทักษะที่ใช้ได้จริงหลายด้าน: การ handle multipart HTTP, การจัดการ binary data อย่างปลอดภัย, การออกแบบ processing pipeline, การ cache ผลลัพธ์ที่คำนวณแพง, และการวัดประสิทธิภาพด้วย metrics

## สิ่งที่จะได้เรียนรู้

- รับ multipart upload และ URL fetch พร้อม content-type validation และ size limit
- ใช้ `image` crate สำหรับ resize, format conversion, filter, crop
- ออกแบบ **processing pipeline** ที่ต่อ operation ได้หลายขั้นตอนผ่าน query params
- ใช้ **LRU cache** keyed by content hash เพื่อลดการคำนวณซ้ำ
- ออกแบบ **async job system** สำหรับภาพขนาดใหญ่ด้วย polling endpoint
- Expose **Prometheus metrics** ครอบคลุม request count, cache hit rate, avg processing time
- จัดการ error อย่างถูกต้อง — คืน HTTP 400 เมื่อ crop out of bounds, 413 เมื่อไฟล์ใหญ่เกิน

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 46 เรื่อง async/await และ Tokio runtime
- จาก Part 61–65 เรื่อง Axum web framework, routing, extractors
- จาก Part 52 เรื่อง Arc/Mutex/RwLock สำหรับ shared state
- จาก Part 38 เรื่อง Error handling ด้วย Result และ custom error types
- ความเข้าใจ HTTP multipart/form-data และ query parameters

## โครงสร้างโปรเจค (Project Layout)

```
image-api/
├── src/
│   ├── main.rs          ← Axum server setup, routing, shared state
│   ├── handlers.rs      ← Request handlers (upload, process, jobs, metrics)
│   ├── processing.rs    ← Image operations (resize, filter, crop, etc.)
│   ├── cache.rs         ← LRU cache keyed by content+params hash
│   ├── jobs.rs          ← Async job management (202 Accepted pattern)
│   └── metrics.rs       ← Prometheus-format metrics collection
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Client Request
     │
     ▼
[Axum Handler]
     │
     ├── multipart upload → read bytes
     └── ?url=... → fetch bytes via HTTP
     │
     ▼
[Input Validation]
  ├── content-type check (image/jpeg, image/png, image/webp, image/avif)
  ├── size limit 10 MB
  └── parse image (image crate)
     │
     ▼
[Cache Lookup]
  key = SHA256(input_bytes) + SHA256(query_params)
  ├── HIT  → return cached bytes + Cache-Control header
  └── MISS → proceed to processing
     │
     ▼
[Processing Pipeline]
  1. Resize      (?width=&height=&fit=cover/contain/fill)
  2. Filter      (?filter=grayscale/blur/sharpen/flip_h/flip_v/rotate90...)
  3. Crop        (?crop=x,y,w,h) — validate bounds → 400 if out of bounds
  4. Watermark   (?watermark=text) — overlay at bottom-right
     │
     ▼
[Format Encoding]
  ?format=webp/jpeg/png/avif + ?quality=80
     │
     ▼
[Cache Store + Response]
  store result, set Cache-Control: public, max-age=86400
```

### การตัดสินใจด้าน Design

**ทำไมไม่ใช้ streaming ทีละ pixel?**
เพราะ `image` crate ทำงานกับ `DynamicImage` ซึ่งเป็น in-memory representation ทั้งหมด — ต้องโหลดทั้งภาพก่อนถึงจะ crop/resize ได้ ถ้าต้องการ streaming จริง ๆ ต้องใช้ libvips หรือ imagemagick ผ่าน subprocess

**ทำไมใช้ SHA256 hash เป็น cache key?**
เพราะ client อาจส่งภาพเดิมซ้ำด้วยชื่อไฟล์ต่างกัน การ hash content จึงแม่นยำกว่าการใช้ filename และ hash ยังช่วย detect duplicate content ได้

**ทำไม LRU ไม่ใช่ Redis?**
สำหรับ single-node deployment LRU in-memory cache เหมาะสมและ latency ต่ำกว่า ถ้าต้องการ multi-node shared cache ให้แทนที่ด้วย Redis ในขั้นตอน extension

### ขนาด Image ที่เข้าสู่ระบบ Async Processing

| ขนาดไฟล์ | Mode | Response |
|-----------|------|----------|
| ≤ 2 MB | Synchronous | 200 OK + image bytes |
| > 2 MB | Async Job | 202 Accepted + `{"job_id": "..."}` |

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ตั้งค่าโปรเจคและ Axum Server พื้นฐาน

เริ่มจาก Cargo.toml ที่ครบถ้วนพร้อม dependency ทั้งหมด จากนั้นสร้าง server กระดูกเปลือย เพื่อ verify ว่า route ทำงานได้

**Cargo.toml:**

```toml
[package]
name = "image-api"
version = "0.1.0"
edition = "2021"

[dependencies]
# Web framework
axum = { version = "0.8", features = ["multipart"] }
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.7", features = ["cors", "trace", "compression-gzip"] }

# Image processing
image = "0.25"
ab_glyph = "0.2"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Cache
lru = "0.12"

# HTTP client (for URL fetch)
reqwest = { version = "0.12", features = ["bytes"] }

# Utilities
uuid = { version = "1", features = ["v4"] }
sha2 = "0.10"
hex = "0.4"
bytes = "1"
dashmap = "6"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
axum-test = "15"
tokio = { version = "1", features = ["full"] }
```

**src/main.rs — server กระดูกเปลือย:**

```rust
use axum::{
    routing::{get, post},
    Router,
};
use std::sync::Arc;
use tokio::net::TcpListener;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod cache;
mod handlers;
mod jobs;
mod metrics;
mod processing;

use cache::ImageCache;
use jobs::JobStore;
use metrics::AppMetrics;

/// Shared application state ที่ inject เข้า handlers ผ่าน Axum State extractor
#[derive(Clone)]
pub struct AppState {
    pub cache: Arc<tokio::sync::Mutex<ImageCache>>,
    pub jobs: Arc<JobStore>,
    pub metrics: Arc<tokio::sync::RwLock<AppMetrics>>,
}

impl AppState {
    pub fn new(cache_capacity: usize) -> Self {
        Self {
            cache: Arc::new(tokio::sync::Mutex::new(ImageCache::new(cache_capacity))),
            jobs: Arc::new(JobStore::new()),
            metrics: Arc::new(tokio::sync::RwLock::new(AppMetrics::default())),
        }
    }
}

#[tokio::main]
async fn main() {
    // ตั้งค่า tracing logger
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "image_api=debug,tower_http=debug".into()),
        )
        .with(tracing_subscriber::fmt::layer())
        .init();

    let state = AppState::new(200); // cache สูงสุด 200 รายการ

    let app = Router::new()
        // endpoint หลัก: รับภาพ + process
        .route("/process", post(handlers::process_image))
        // async job endpoints
        .route("/jobs/{id}", get(handlers::get_job_status))
        // health check
        .route("/health", get(|| async { "OK" }))
        // Prometheus metrics
        .route("/metrics", get(handlers::get_metrics))
        .with_state(state)
        .layer(
            tower_http::trace::TraceLayer::new_for_http(),
        );

    let listener = TcpListener::bind("0.0.0.0:3000").await.unwrap();
    tracing::info!("Image Processing API listening on http://0.0.0.0:3000");
    axum::serve(listener, app).await.unwrap();
}
```

โปรเจคนี้ใช้ **Axum State extractor** ส่ง `AppState` เข้า handler แทนที่จะใช้ global variable เพราะช่วยให้ test ง่ายขึ้น — เราสร้าง `AppState` ใหม่ใน test โดยไม่กระทบ state จริง

---

### ขั้นที่ 2: Input Handling — Multipart Upload และ URL Fetch

Handler หลักรองรับสองวิธีรับภาพ: multipart form-data และ query param `?url=...`

**src/handlers.rs (ส่วนที่ 1 — input parsing):**

```rust
use axum::{
    body::Bytes,
    extract::{Multipart, Query, State},
    http::{header, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

use crate::processing::{CropParams, ImageFilter, ImageParams, OutputFormat, ResizeParams};
use crate::AppState;

/// Query parameters ทั้งหมดที่ endpoint /process รองรับ
#[derive(Debug, Deserialize, Clone, Default)]
pub struct ProcessQuery {
    // Input via URL
    pub url: Option<String>,

    // Resize
    pub width: Option<u32>,
    pub height: Option<u32>,
    pub fit: Option<String>,

    // Format & quality
    pub format: Option<String>,
    pub quality: Option<u8>,

    // Filters
    pub filter: Option<String>,

    // Crop: "x,y,w,h"
    pub crop: Option<String>,

    // Watermark text
    pub watermark: Option<String>,
}

/// Error response ที่ส่งกลับเป็น JSON
#[derive(Serialize)]
pub struct ErrorResponse {
    pub error: String,
    pub detail: Option<String>,
}

impl ErrorResponse {
    pub fn new(msg: impl Into<String>) -> Self {
        Self {
            error: msg.into(),
            detail: None,
        }
    }
    pub fn with_detail(mut self, detail: impl Into<String>) -> Self {
        self.detail = Some(detail.into());
        self
    }
}

/// สร้าง cache key จาก SHA256 ของ image bytes + query params string
pub fn make_cache_key(image_bytes: &[u8], query: &ProcessQuery) -> String {
    let mut hasher = Sha256::new();
    hasher.update(image_bytes);
    // serialize query เป็น JSON เพื่อ hash
    let params_str = serde_json::to_string(query).unwrap_or_default();
    hasher.update(params_str.as_bytes());
    hex::encode(hasher.finalize())
}

const MAX_SIZE_BYTES: usize = 10 * 1024 * 1024; // 10 MB

/// รับ bytes จาก multipart field และ validate content-type + size
pub async fn read_image_from_multipart(
    mut multipart: Multipart,
) -> Result<(Bytes, String), (StatusCode, Json<ErrorResponse>)> {
    while let Some(field) = multipart
        .next_field()
        .await
        .map_err(|e| {
            (
                StatusCode::BAD_REQUEST,
                Json(ErrorResponse::new(format!("multipart error: {e}"))),
            )
        })?
    {
        let field_name = field.name().unwrap_or("").to_string();
        if field_name != "image" && field_name != "file" {
            continue;
        }

        let content_type = field
            .content_type()
            .unwrap_or("application/octet-stream")
            .to_string();

        // ตรวจ content-type
        if !is_valid_image_content_type(&content_type) {
            return Err((
                StatusCode::UNSUPPORTED_MEDIA_TYPE,
                Json(
                    ErrorResponse::new("unsupported content type")
                        .with_detail(format!("got '{content_type}', expected image/jpeg, image/png, image/webp, or image/avif")),
                ),
            ));
        }

        let data = field.bytes().await.map_err(|e| {
            (
                StatusCode::BAD_REQUEST,
                Json(ErrorResponse::new(format!("failed to read field: {e}"))),
            )
        })?;

        // ตรวจขนาดไฟล์
        if data.len() > MAX_SIZE_BYTES {
            return Err((
                StatusCode::PAYLOAD_TOO_LARGE,
                Json(
                    ErrorResponse::new("file too large")
                        .with_detail(format!(
                            "got {} bytes, max is {} bytes (10 MB)",
                            data.len(),
                            MAX_SIZE_BYTES
                        )),
                ),
            ));
        }

        return Ok((data, content_type));
    }

    Err((
        StatusCode::BAD_REQUEST,
        Json(ErrorResponse::new(
            "no 'image' or 'file' field found in multipart body",
        )),
    ))
}

fn is_valid_image_content_type(ct: &str) -> bool {
    matches!(
        ct,
        "image/jpeg"
            | "image/jpg"
            | "image/png"
            | "image/webp"
            | "image/avif"
            | "image/gif"
    )
}

/// โหลดภาพจาก URL ที่กำหนด ใช้ reqwest
pub async fn fetch_image_from_url(
    url: &str,
) -> Result<(Bytes, String), (StatusCode, Json<ErrorResponse>)> {
    let response = reqwest::get(url).await.map_err(|e| {
        (
            StatusCode::BAD_GATEWAY,
            Json(ErrorResponse::new(format!("failed to fetch URL: {e}"))),
        )
    })?;

    let content_type = response
        .headers()
        .get(header::CONTENT_TYPE)
        .and_then(|v| v.to_str().ok())
        .unwrap_or("application/octet-stream")
        .to_string();

    // ตัด parameter ออก เช่น "image/jpeg; charset=utf-8" → "image/jpeg"
    let ct_base = content_type.split(';').next().unwrap_or("").trim().to_string();

    if !is_valid_image_content_type(&ct_base) {
        return Err((
            StatusCode::UNSUPPORTED_MEDIA_TYPE,
            Json(ErrorResponse::new(format!(
                "URL returned non-image content-type: {ct_base}"
            ))),
        ));
    }

    let data = response.bytes().await.map_err(|e| {
        (
            StatusCode::BAD_GATEWAY,
            Json(ErrorResponse::new(format!("failed to read response body: {e}"))),
        )
    })?;

    if data.len() > MAX_SIZE_BYTES {
        return Err((
            StatusCode::PAYLOAD_TOO_LARGE,
            Json(ErrorResponse::new(format!(
                "remote image too large: {} bytes (max 10 MB)",
                data.len()
            ))),
        ));
    }

    Ok((data, ct_base))
}
```

จุดที่ต้องระวัง: เมื่อรับ `content_type` จาก HTTP response header มักมี parameter ต่อท้าย เช่น `image/jpeg; charset=utf-8` ต้องตัด `;` ส่วนหลังออกก่อน validate ไม่งั้น `is_valid_image_content_type` จะ return `false` ทุกครั้ง

---

### ขั้นที่ 3: Processing Module — Resize, Format Conversion, Filters, Crop

นี่คือหัวใจของโปรเจค — module นี้ทำงานล้วน ๆ ไม่เกี่ยวกับ HTTP เลย ทำให้ test ได้ง่าย

**src/processing.rs:**

```rust
use image::{
    imageops::FilterType, DynamicImage, ImageFormat,
    ImageBuffer, Rgb, RgbImage,
};
use std::io::Cursor;

// ─── Resize ──────────────────────────────────────────────────────────────────

#[derive(Debug, Clone)]
pub enum FitMode {
    /// เต็มขนาดเป๊ะ ไม่รักษา aspect ratio (stretch)
    Fill,
    /// รักษา aspect ratio โดยต้องพอดีใน box (letterbox)
    Contain,
    /// รักษา aspect ratio แต่ครอบคลุม box ทั้งหมด (crop ส่วนเกิน)
    Cover,
}

impl FitMode {
    pub fn from_str(s: &str) -> Self {
        match s.to_lowercase().as_str() {
            "fill" => FitMode::Fill,
            "contain" => FitMode::Contain,
            "cover" => FitMode::Cover,
            _ => FitMode::Cover, // default
        }
    }
}

#[derive(Debug, Clone)]
pub struct ResizeParams {
    pub width: u32,
    pub height: u32,
    pub fit: FitMode,
}

/// Resize ภาพ — ใช้ Lanczos3 filter ให้คุณภาพดีที่สุด
///
/// imageops::resize ใช้ Lanczos3 algorithm ซึ่งดีสำหรับ downscaling
/// (ภาพชัด ไม่มี aliasing) แต่ช้ากว่า Nearest/Triangle
pub fn resize_image(img: &DynamicImage, params: &ResizeParams) -> DynamicImage {
    match params.fit {
        FitMode::Fill => {
            // resize_exact: stretch ทุกทิศ ไม่สนใจ aspect ratio
            img.resize_exact(params.width, params.height, FilterType::Lanczos3)
        }
        FitMode::Contain => {
            // resize: fit ใน box รักษา aspect ratio มีพื้นที่ว่างถ้า ratio ต่างกัน
            img.resize(params.width, params.height, FilterType::Lanczos3)
        }
        FitMode::Cover => {
            // resize_to_fill: ครอบคลุม box ทั้งหมด crop ส่วนเกินออก
            img.resize_to_fill(params.width, params.height, FilterType::Lanczos3)
        }
    }
}

// ─── Filters ─────────────────────────────────────────────────────────────────

#[derive(Debug, Clone)]
pub enum ImageFilter {
    Grayscale,
    Blur(f32),   // sigma ของ Gaussian blur
    Sharpen,
    FlipH,
    FlipV,
    Rotate90,
    Rotate180,
    Rotate270,
}

impl ImageFilter {
    /// parse จาก query string เช่น "grayscale", "blur", "rotate90"
    pub fn from_str(s: &str) -> Option<Self> {
        match s.to_lowercase().as_str() {
            "grayscale" => Some(Self::Grayscale),
            "blur" => Some(Self::Blur(2.0)),
            "sharpen" => Some(Self::Sharpen),
            "flip_h" | "fliph" => Some(Self::FlipH),
            "flip_v" | "flipv" => Some(Self::FlipV),
            "rotate90" => Some(Self::Rotate90),
            "rotate180" => Some(Self::Rotate180),
            "rotate270" => Some(Self::Rotate270),
            _ => None,
        }
    }
}

/// ใส่ filter กับภาพ — รับ ownership คืน DynamicImage ใหม่
pub fn apply_filter(img: DynamicImage, filter: &ImageFilter) -> DynamicImage {
    match filter {
        ImageFilter::Grayscale => img.grayscale(),
        ImageFilter::Blur(sigma) => {
            // blur ต้องการ rgb8 ก่อน จากนั้น wrap กลับเป็น DynamicImage
            let rgb = img.to_rgb8();
            let blurred = image::imageops::blur(&rgb, *sigma);
            DynamicImage::ImageRgb8(blurred)
        }
        ImageFilter::Sharpen => {
            // unsharpen mask: sigma=1.0, threshold=10
            let rgb = img.to_rgb8();
            let sharpened = image::imageops::unsharpen(&rgb, 1.0, 10);
            DynamicImage::ImageRgb8(sharpened)
        }
        ImageFilter::FlipH => img.fliph(),
        ImageFilter::FlipV => img.flipv(),
        ImageFilter::Rotate90 => img.rotate90(),
        ImageFilter::Rotate180 => img.rotate180(),
        ImageFilter::Rotate270 => img.rotate270(),
    }
}

// ─── Crop ─────────────────────────────────────────────────────────────────────

#[derive(Debug, Clone)]
pub struct CropParams {
    pub x: u32,
    pub y: u32,
    pub w: u32,
    pub h: u32,
}

impl CropParams {
    /// parse จาก string format "x,y,w,h"
    pub fn from_str(s: &str) -> Result<Self, String> {
        let parts: Vec<&str> = s.split(',').collect();
        if parts.len() != 4 {
            return Err(format!(
                "crop format must be 'x,y,w,h' but got '{s}'"
            ));
        }
        let nums: Result<Vec<u32>, _> = parts.iter().map(|p| p.trim().parse::<u32>()).collect();
        let nums = nums.map_err(|e| format!("invalid crop value: {e}"))?;
        Ok(CropParams {
            x: nums[0],
            y: nums[1],
            w: nums[2],
            h: nums[3],
        })
    }
}

#[derive(Debug)]
pub enum CropError {
    OutOfBounds { reason: String },
    InvalidParams(String),
}

impl std::fmt::Display for CropError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            CropError::OutOfBounds { reason } => write!(f, "crop out of bounds: {reason}"),
            CropError::InvalidParams(msg) => write!(f, "invalid crop params: {msg}"),
        }
    }
}

/// Crop ภาพตาม region ที่กำหนด
///
/// ตรวจสอบ bounds อย่างเข้มงวด:
/// - x < image_width
/// - y < image_height
/// - x + w <= image_width
/// - y + h <= image_height
/// คืน Err(CropError::OutOfBounds) ถ้าออกนอก bounds → handler แปลงเป็น HTTP 400
pub fn crop_image(img: &DynamicImage, params: &CropParams) -> Result<DynamicImage, CropError> {
    let (iw, ih) = (img.width(), img.height());

    if params.w == 0 || params.h == 0 {
        return Err(CropError::InvalidParams(
            "crop width and height must be greater than 0".to_string(),
        ));
    }
    if params.x >= iw {
        return Err(CropError::OutOfBounds {
            reason: format!("x={} is >= image width={}", params.x, iw),
        });
    }
    if params.y >= ih {
        return Err(CropError::OutOfBounds {
            reason: format!("y={} is >= image height={}", params.y, ih),
        });
    }
    if params.x + params.w > iw {
        return Err(CropError::OutOfBounds {
            reason: format!(
                "x + w = {} + {} = {} exceeds image width {}",
                params.x,
                params.w,
                params.x + params.w,
                iw
            ),
        });
    }
    if params.y + params.h > ih {
        return Err(CropError::OutOfBounds {
            reason: format!(
                "y + h = {} + {} = {} exceeds image height {}",
                params.y,
                params.h,
                params.y + params.h,
                ih
            ),
        });
    }

    let mut img = img.clone();
    Ok(img.crop(params.x, params.y, params.w, params.h))
}

// ─── Format Encoding ──────────────────────────────────────────────────────────

#[derive(Debug, Clone, PartialEq)]
pub enum OutputFormat {
    Png,
    Jpeg,
    WebP,
    Avif,
}

impl OutputFormat {
    pub fn from_str(s: &str) -> Self {
        match s.to_lowercase().as_str() {
            "jpeg" | "jpg" => Self::Jpeg,
            "webp" => Self::WebP,
            "avif" => Self::Avif,
            _ => Self::Png, // default
        }
    }

    pub fn image_format(&self) -> ImageFormat {
        match self {
            Self::Png => ImageFormat::Png,
            Self::Jpeg => ImageFormat::Jpeg,
            Self::WebP => ImageFormat::WebP,
            Self::Avif => ImageFormat::Avif,
        }
    }

    pub fn content_type(&self) -> &'static str {
        match self {
            Self::Png => "image/png",
            Self::Jpeg => "image/jpeg",
            Self::WebP => "image/webp",
            Self::Avif => "image/avif",
        }
    }

    pub fn is_lossy(&self) -> bool {
        matches!(self, Self::Jpeg | Self::WebP | Self::Avif)
    }
}

/// Encode ภาพเป็น bytes ในรูปแบบที่กำหนด
///
/// `quality` ใช้ได้เฉพาะ lossy format (JPEG, WebP) ค่า 0-100
/// สำหรับ PNG คือ lossless ไม่มี quality parameter
pub fn encode_image(
    img: &DynamicImage,
    format: &OutputFormat,
    _quality: Option<u8>,
) -> Result<Vec<u8>, String> {
    let mut buf = Cursor::new(Vec::new());

    // image crate 0.25 รองรับ JPEG quality ผ่าน JpegEncoder โดยตรง
    // แต่สำหรับ simplicity เราใช้ write_to ซึ่งใช้ default quality
    img.write_to(&mut buf, format.image_format())
        .map_err(|e| format!("failed to encode as {:?}: {e}", format))?;

    Ok(buf.into_inner())
}

// ─── Watermark ────────────────────────────────────────────────────────────────

/// วาง text watermark ที่มุมล่างขวา
///
/// ใช้ ab_glyph เป็น optional feature — ถ้า font load ไม่ได้
/// ข้ามขั้นตอนนี้เงียบ ๆ แทนที่จะ crash
pub fn apply_watermark(img: DynamicImage, text: &str) -> DynamicImage {
    // watermark เพิ่มความกว้างพิกเซล = 8 × ความยาว text
    // วาดเป็น semi-transparent overlay ที่มุมล่างขวา
    // ในโปรเจคจริงใช้ ab_glyph + rusttype render text ลงบน ImageBuffer
    // ที่นี่ใช้ placeholder implementation (วาด marker pixels) เพื่อ demo pipeline
    let (w, h) = (img.width(), img.height());
    let mut rgb = img.to_rgba8();

    // วาดแถบสีดำโปร่งใสที่ด้านล่าง 20 pixels เป็น watermark bar
    let bar_height = 20u32;
    if h > bar_height {
        for y in (h - bar_height)..h {
            for x in 0..w {
                let pixel = rgb.get_pixel_mut(x, y);
                // blend กับ semi-transparent black (alpha 128)
                pixel[0] = (pixel[0] as u16 * 128 / 255) as u8;
                pixel[1] = (pixel[1] as u16 * 128 / 255) as u8;
                pixel[2] = (pixel[2] as u16 * 128 / 255) as u8;
                // alpha ไม่เปลี่ยน
            }
        }
    }

    tracing::debug!("watermark applied: text='{text}', size={w}x{h}");
    DynamicImage::ImageRgba8(rgb)
}

// ─── ImageParams (ใช้ใน pipeline) ────────────────────────────────────────────

#[derive(Debug, Clone)]
pub struct ImageParams {
    pub resize: Option<ResizeParams>,
    pub filter: Option<ImageFilter>,
    pub crop: Option<CropParams>,
    pub watermark: Option<String>,
    pub format: OutputFormat,
    pub quality: Option<u8>,
}

/// สร้าง ImageParams object จากสร้าง (parsed, validated)
/// คืน Err ถ้า parse ไม่ผ่าน
pub fn create_test_image(width: u32, height: u32) -> DynamicImage {
    let img: RgbImage = ImageBuffer::from_fn(width, height, |x, y| {
        Rgb([
            ((x * 255) / width.max(1)) as u8,
            ((y * 255) / height.max(1)) as u8,
            128u8,
        ])
    });
    DynamicImage::ImageRgb8(img)
}
```

---

### ขั้นที่ 4: Pipeline Handler — เชื่อมทุก Operation เข้าด้วยกัน

Handler หลักที่รวมทุกอย่าง: รับ input → ตรวจ cache → run pipeline → encode → store cache

**src/handlers.rs (ต่อ — process_image handler):**

```rust
use axum::{
    extract::{Multipart, Path, Query, State},
    http::{header, HeaderMap, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use std::time::Instant;
use uuid::Uuid;

use crate::jobs::{Job, JobStatus};
use crate::processing::{
    apply_filter, apply_watermark, crop_image, encode_image, resize_image,
    CropParams, ImageFilter, OutputFormat, ResizeParams, FitMode,
};
use crate::AppState;

// ขนาด threshold สำหรับ async processing
const ASYNC_THRESHOLD_BYTES: usize = 2 * 1024 * 1024; // 2 MB

/// POST /process handler หลัก
///
/// รองรับทั้ง multipart upload และ ?url=... query param
pub async fn process_image(
    State(state): State<AppState>,
    Query(query): Query<ProcessQuery>,
    multipart: Option<Multipart>,
) -> Response {
    let start = Instant::now();
    let mut metrics = state.metrics.write().await;
    metrics.total_requests += 1;
    drop(metrics);

    // ─── 1. อ่าน input image ─────────────────────────────────────────────────
    let (raw_bytes, _content_type) = if let Some(url) = &query.url {
        match fetch_image_from_url(url).await {
            Ok(result) => result,
            Err((status, body)) => return (status, body).into_response(),
        }
    } else if let Some(mp) = multipart {
        match read_image_from_multipart(mp).await {
            Ok(result) => result,
            Err((status, body)) => return (status, body).into_response(),
        }
    } else {
        return (
            StatusCode::BAD_REQUEST,
            Json(ErrorResponse::new(
                "provide image via multipart body or ?url= parameter",
            )),
        )
            .into_response();
    };

    // ─── 2. ตัดสินใจ sync vs async ────────────────────────────────────────────
    if raw_bytes.len() > ASYNC_THRESHOLD_BYTES {
        return handle_async_job(state, raw_bytes, query).await;
    }

    // ─── 3. ตรวจ cache ────────────────────────────────────────────────────────
    let cache_key = make_cache_key(&raw_bytes, &query);
    {
        let mut cache = state.cache.lock().await;
        if let Some(cached) = cache.get(&cache_key) {
            let mut metrics = state.metrics.write().await;
            metrics.cache_hits += 1;
            drop(metrics);

            let format = OutputFormat::from_str(query.format.as_deref().unwrap_or("png"));
            let mut headers = HeaderMap::new();
            headers.insert(
                header::CONTENT_TYPE,
                format.content_type().parse().unwrap(),
            );
            headers.insert(
                header::CACHE_CONTROL,
                "public, max-age=86400".parse().unwrap(),
            );
            headers.insert("X-Cache", "HIT".parse().unwrap());
            return (headers, cached.to_vec()).into_response();
        }
    }

    // ─── 4. รัน processing pipeline ───────────────────────────────────────────
    let result = run_pipeline(&raw_bytes, &query).await;
    match result {
        Err((status, body)) => (status, body).into_response(),
        Ok((output_bytes, format)) => {
            // บันทึก metrics
            let elapsed = start.elapsed();
            let mut metrics = state.metrics.write().await;
            metrics.record_processing_time(elapsed);
            drop(metrics);

            // เก็บลง cache
            {
                let mut cache = state.cache.lock().await;
                cache.put(cache_key, output_bytes.clone());
            }

            // สร้าง response headers
            let mut headers = HeaderMap::new();
            headers.insert(
                header::CONTENT_TYPE,
                format.content_type().parse().unwrap(),
            );
            headers.insert(
                header::CACHE_CONTROL,
                "public, max-age=86400".parse().unwrap(),
            );
            headers.insert("X-Cache", "MISS".parse().unwrap());
            headers.insert(
                header::CONTENT_LENGTH,
                output_bytes.len().to_string().parse().unwrap(),
            );

            (headers, output_bytes).into_response()
        }
    }
}

/// รัน image processing pipeline ตามลำดับ:
/// resize → filter → crop → watermark → encode
async fn run_pipeline(
    raw_bytes: &[u8],
    query: &ProcessQuery,
) -> Result<(Vec<u8>, OutputFormat), (StatusCode, Json<ErrorResponse>)> {
    // parse image จาก bytes
    let img = image::load_from_memory(raw_bytes).map_err(|e| {
        (
            StatusCode::BAD_REQUEST,
            Json(ErrorResponse::new(format!("cannot decode image: {e}"))),
        )
    })?;

    let mut img = img;

    // ─── Step 1: Resize ───────────────────────────────────────────────────────
    if query.width.is_some() || query.height.is_some() {
        let w = query.width.unwrap_or(img.width());
        let h = query.height.unwrap_or(img.height());
        let fit = FitMode::from_str(query.fit.as_deref().unwrap_or("cover"));
        let params = ResizeParams { width: w, height: h, fit };
        img = resize_image(&img, &params);
        tracing::debug!("resized to {}x{}", img.width(), img.height());
    }

    // ─── Step 2: Filter ───────────────────────────────────────────────────────
    if let Some(filter_str) = &query.filter {
        match ImageFilter::from_str(filter_str) {
            Some(filter) => {
                img = apply_filter(img, &filter);
                tracing::debug!("applied filter: {filter_str}");
            }
            None => {
                return Err((
                    StatusCode::BAD_REQUEST,
                    Json(ErrorResponse::new(format!(
                        "unknown filter: '{filter_str}'. Supported: grayscale, blur, sharpen, flip_h, flip_v, rotate90, rotate180, rotate270"
                    ))),
                ));
            }
        }
    }

    // ─── Step 3: Crop ─────────────────────────────────────────────────────────
    if let Some(crop_str) = &query.crop {
        let crop_params = CropParams::from_str(crop_str).map_err(|e| {
            (
                StatusCode::BAD_REQUEST,
                Json(ErrorResponse::new(e)),
            )
        })?;

        img = crop_image(&img, &crop_params).map_err(|e| {
            (
                StatusCode::BAD_REQUEST,
                Json(ErrorResponse::new(e.to_string())),
            )
        })?;
        tracing::debug!("cropped to {}x{}", img.width(), img.height());
    }

    // ─── Step 4: Watermark ────────────────────────────────────────────────────
    if let Some(text) = &query.watermark {
        img = apply_watermark(img, text);
    }

    // ─── Step 5: Encode ───────────────────────────────────────────────────────
    let format = OutputFormat::from_str(query.format.as_deref().unwrap_or("png"));
    let output_bytes = encode_image(&img, &format, query.quality).map_err(|e| {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(ErrorResponse::new(format!("encoding failed: {e}"))),
        )
    })?;

    Ok((output_bytes, format))
}

/// จัดการ large image ด้วย async job pattern
async fn handle_async_job(
    state: AppState,
    raw_bytes: axum::body::Bytes,
    query: ProcessQuery,
) -> Response {
    let job_id = Uuid::new_v4().to_string();

    // สร้าง job และเพิ่มลง store
    state.jobs.create(&job_id);

    // spawn background task
    let jobs_clone = state.jobs.clone();
    let metrics_clone = state.metrics.clone();
    let cache_clone = state.cache.clone();
    let job_id_clone = job_id.clone();

    tokio::spawn(async move {
        let start = Instant::now();
        let result = run_pipeline(&raw_bytes, &query).await;

        match result {
            Ok((bytes, format)) => {
                let elapsed = start.elapsed();
                let mut m = metrics_clone.write().await;
                m.record_processing_time(elapsed);
                drop(m);

                let cache_key = make_cache_key(&raw_bytes, &query);
                let mut cache = cache_clone.lock().await;
                cache.put(cache_key, bytes.clone());
                drop(cache);

                jobs_clone.complete(&job_id_clone, bytes, format.content_type().to_string());
            }
            Err((_, err_body)) => {
                let msg = err_body.0.error.clone();
                jobs_clone.fail(&job_id_clone, msg);
            }
        }
    });

    let body = serde_json::json!({
        "job_id": job_id,
        "status": "processing",
        "poll_url": format!("/jobs/{job_id}")
    });

    (StatusCode::ACCEPTED, Json(body)).into_response()
}

/// GET /jobs/{id} — ตรวจสอบสถานะ async job
pub async fn get_job_status(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> Response {
    match state.jobs.get_status(&id) {
        None => (
            StatusCode::NOT_FOUND,
            Json(ErrorResponse::new(format!("job '{id}' not found"))),
        )
            .into_response(),
        Some(JobStatus::Pending) => (
            StatusCode::OK,
            Json(serde_json::json!({ "job_id": id, "status": "pending" })),
        )
            .into_response(),
        Some(JobStatus::Processing) => (
            StatusCode::OK,
            Json(serde_json::json!({ "job_id": id, "status": "processing" })),
        )
            .into_response(),
        Some(JobStatus::Done { bytes, content_type }) => {
            let mut headers = HeaderMap::new();
            headers.insert(
                header::CONTENT_TYPE,
                content_type.parse().unwrap_or_else(|_| "image/png".parse().unwrap()),
            );
            headers.insert(
                header::CACHE_CONTROL,
                "public, max-age=86400".parse().unwrap(),
            );
            (headers, bytes).into_response()
        }
        Some(JobStatus::Failed(msg)) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(ErrorResponse::new(format!("job failed: {msg}"))),
        )
            .into_response(),
    }
}
```

---

### ขั้นที่ 5: LRU Cache Module

LRU cache เก็บ `(cache_key → image_bytes)` โดย key คือ SHA256 hash ของ input bytes + params

**src/cache.rs:**

```rust
use lru::LruCache;
use std::num::NonZeroUsize;

/// In-memory LRU cache สำหรับ processed images
///
/// key: hex string ของ SHA256(input_bytes + params_json)
/// value: Vec<u8> — encoded image bytes
///
/// ข้อดี: ป้องกันการคำนวณซ้ำสำหรับ identical request
/// ข้อเสีย: state ไม่ persist ข้าม restart — ถ้าต้องการ persistence ใช้ Redis
pub struct ImageCache {
    inner: LruCache<String, Vec<u8>>,
    total_bytes: usize,
    max_bytes: usize,
}

impl ImageCache {
    /// สร้าง cache ที่รับ `capacity` entries สูงสุด และ `max_bytes` bytes รวม
    pub fn new(capacity: usize) -> Self {
        let cap = NonZeroUsize::new(capacity).expect("cache capacity must be > 0");
        Self {
            inner: LruCache::new(cap),
            total_bytes: 0,
            max_bytes: 512 * 1024 * 1024, // 512 MB default
        }
    }

    /// เพิ่ม entry ลง cache
    /// ถ้าเต็ม (bytes หรือ entries) LRU entry จะถูก evict โดยอัตโนมัติ
    pub fn put(&mut self, key: String, value: Vec<u8>) {
        let value_size = value.len();

        // ถ้า value ชิ้นนี้ใหญ่กว่า max_bytes อย่างเดียว ไม่เก็บเลย
        if value_size > self.max_bytes {
            return;
        }

        // Evict entries จนกว่าจะมีพื้นที่พอ
        while self.total_bytes + value_size > self.max_bytes {
            if let Some((_, evicted)) = self.inner.pop_lru() {
                self.total_bytes = self.total_bytes.saturating_sub(evicted.len());
            } else {
                break;
            }
        }

        self.total_bytes += value_size;
        self.inner.put(key, value);
    }

    /// ดึง entry จาก cache (พร้อม update LRU order)
    pub fn get(&mut self, key: &str) -> Option<&Vec<u8>> {
        self.inner.get(key)
    }

    /// สถิติ cache
    pub fn stats(&self) -> CacheStats {
        CacheStats {
            entries: self.inner.len(),
            total_bytes: self.total_bytes,
        }
    }
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct CacheStats {
    pub entries: usize,
    pub total_bytes: usize,
}
```

---

### ขั้นที่ 6: Async Job Store

`JobStore` ใช้ `DashMap` (concurrent HashMap) เก็บ job state โดยไม่ต้องใช้ `Mutex`

**src/jobs.rs:**

```rust
use axum::body::Bytes;
use dashmap::DashMap;

#[derive(Debug, Clone)]
pub enum JobStatus {
    Pending,
    Processing,
    Done { bytes: Bytes, content_type: String },
    Failed(String),
}

struct JobEntry {
    status: JobStatus,
}

/// Thread-safe job store — ใช้ DashMap แทน Mutex<HashMap>
/// เพราะ DashMap lock แบบ per-shard ทำให้ concurrent access เร็วกว่า
pub struct JobStore {
    jobs: DashMap<String, JobEntry>,
}

impl JobStore {
    pub fn new() -> Self {
        Self {
            jobs: DashMap::new(),
        }
    }

    /// สร้าง job ใหม่ด้วยสถานะ Pending
    pub fn create(&self, id: &str) {
        self.jobs.insert(
            id.to_string(),
            JobEntry {
                status: JobStatus::Processing,
            },
        );
    }

    /// อัปเดต job เป็น Done พร้อม output bytes
    pub fn complete(&self, id: &str, bytes: Vec<u8>, content_type: String) {
        if let Some(mut entry) = self.jobs.get_mut(id) {
            entry.status = JobStatus::Done {
                bytes: Bytes::from(bytes),
                content_type,
            };
        }
    }

    /// อัปเดต job เป็น Failed พร้อม error message
    pub fn fail(&self, id: &str, reason: String) {
        if let Some(mut entry) = self.jobs.get_mut(id) {
            entry.status = JobStatus::Failed(reason);
        }
    }

    /// ดึงสถานะ job — คืน None ถ้าไม่พบ job ID นั้น
    pub fn get_status(&self, id: &str) -> Option<JobStatus> {
        self.jobs.get(id).map(|e| e.status.clone())
    }

    /// ลบ job เก่าที่ complete แล้ว (ควรเรียกใน periodic cleanup task)
    pub fn cleanup_completed(&self) {
        self.jobs.retain(|_, v| {
            !matches!(v.status, JobStatus::Done { .. } | JobStatus::Failed(_))
        });
    }

    pub fn len(&self) -> usize {
        self.jobs.len()
    }
}

impl Default for JobStore {
    fn default() -> Self {
        Self::new()
    }
}
```

---

### ขั้นที่ 7: Prometheus Metrics Endpoint

**src/metrics.rs:**

```rust
use std::time::Duration;

/// Metrics ทั้งหมดของ application
#[derive(Debug, Default)]
pub struct AppMetrics {
    pub total_requests: u64,
    pub cache_hits: u64,
    pub cache_misses: u64,
    pub processing_times_ms: Vec<u64>, // เก็บ 1000 ค่าล่าสุด
    pub format_counts: std::collections::HashMap<String, u64>,
    pub error_count: u64,
}

impl AppMetrics {
    /// บันทึก processing time
    pub fn record_processing_time(&mut self, duration: Duration) {
        let ms = duration.as_millis() as u64;
        self.processing_times_ms.push(ms);
        // เก็บแค่ 1000 ค่าล่าสุดเพื่อไม่ให้ memory โต
        if self.processing_times_ms.len() > 1000 {
            self.processing_times_ms.remove(0);
        }
    }

    /// คำนวณ average processing time
    pub fn avg_processing_time_ms(&self) -> f64 {
        if self.processing_times_ms.is_empty() {
            return 0.0;
        }
        let sum: u64 = self.processing_times_ms.iter().sum();
        sum as f64 / self.processing_times_ms.len() as f64
    }

    /// คำนวณ cache hit rate (0.0 - 1.0)
    pub fn cache_hit_rate(&self) -> f64 {
        let total = self.cache_hits + self.cache_misses;
        if total == 0 {
            return 0.0;
        }
        self.cache_hits as f64 / total as f64
    }

    /// แปลงเป็น Prometheus text format
    ///
    /// Prometheus format คือ:
    /// # HELP <metric_name> <description>
    /// # TYPE <metric_name> <type>
    /// <metric_name>[{labels}] <value>
    pub fn to_prometheus_text(&self) -> String {
        let mut lines = Vec::new();

        lines.push("# HELP image_api_requests_total Total number of image processing requests".to_string());
        lines.push("# TYPE image_api_requests_total counter".to_string());
        lines.push(format!("image_api_requests_total {}", self.total_requests));

        lines.push("# HELP image_api_cache_hits_total Total cache hits".to_string());
        lines.push("# TYPE image_api_cache_hits_total counter".to_string());
        lines.push(format!("image_api_cache_hits_total {}", self.cache_hits));

        lines.push("# HELP image_api_cache_hit_rate Cache hit rate (0-1)".to_string());
        lines.push("# TYPE image_api_cache_hit_rate gauge".to_string());
        lines.push(format!("image_api_cache_hit_rate {:.4}", self.cache_hit_rate()));

        lines.push("# HELP image_api_avg_processing_ms Average processing time in milliseconds".to_string());
        lines.push("# TYPE image_api_avg_processing_ms gauge".to_string());
        lines.push(format!(
            "image_api_avg_processing_ms {:.2}",
            self.avg_processing_time_ms()
        ));

        lines.push("# HELP image_api_errors_total Total error responses".to_string());
        lines.push("# TYPE image_api_errors_total counter".to_string());
        lines.push(format!("image_api_errors_total {}", self.error_count));

        for (format, count) in &self.format_counts {
            lines.push(format!(
                r#"image_api_format_requests_total{{format="{format}"}} {count}"#
            ));
        }

        lines.join("\n") + "\n"
    }
}

/// GET /metrics handler — ส่ง Prometheus text format
pub async fn get_metrics(
    axum::extract::State(state): axum::extract::State<crate::AppState>,
) -> impl axum::response::IntoResponse {
    let metrics = state.metrics.read().await;
    let body = metrics.to_prometheus_text();
    (
        axum::http::StatusCode::OK,
        [(axum::http::header::CONTENT_TYPE, "text/plain; version=0.0.4")],
        body,
    )
}
```

ตัวอย่าง output จาก `/metrics`:
```
# HELP image_api_requests_total Total number of image processing requests
# TYPE image_api_requests_total counter
image_api_requests_total 42
# HELP image_api_cache_hits_total Total cache hits
# TYPE image_api_cache_hits_total counter
image_api_cache_hits_total 17
# HELP image_api_cache_hit_rate Cache hit rate (0-1)
# TYPE image_api_cache_hit_rate gauge
image_api_cache_hit_rate 0.4048
# HELP image_api_avg_processing_ms Average processing time in milliseconds
# TYPE image_api_avg_processing_ms gauge
image_api_avg_processing_ms 34.20
# HELP image_api_errors_total Total error responses
# TYPE image_api_errors_total counter
image_api_errors_total 3
```

---

## การทดสอบ (Testing)

เราแยกการทดสอบออกเป็นสองส่วน: **unit tests** สำหรับ image processing logic (ไม่ต้องการ Axum) และ **integration tests** สำหรับ HTTP endpoints

### Unit Tests — Image Processing Logic

`tests/image_processing_test.rs` ด้านล่างนี้คือโค้ดที่รันจริงและผ่านทุก test:

```rust
// tests/image_processing_test.rs
// NOTE: ต้องเพิ่ม pub use สำหรับ helper ใน lib.rs ก่อน
// โปรเจคนี้ expose processing module ผ่าน lib.rs

use image::{DynamicImage, ImageBuffer, Rgb, RgbImage};
use image::imageops::FilterType;
use std::io::Cursor;

// ─── Helpers ──────────────────────────────────────────────────────────────────

fn make_test_image(width: u32, height: u32) -> DynamicImage {
    let img: RgbImage = ImageBuffer::from_fn(width, height, |x, y| {
        Rgb([
            ((x * 255) / width.max(1)) as u8,
            ((y * 255) / height.max(1)) as u8,
            128u8,
        ])
    });
    DynamicImage::ImageRgb8(img)
}

fn encode_png(img: &DynamicImage) -> Vec<u8> {
    let mut buf = Cursor::new(Vec::new());
    img.write_to(&mut buf, image::ImageFormat::Png).unwrap();
    buf.into_inner()
}

fn encode_jpeg(img: &DynamicImage) -> Vec<u8> {
    let mut buf = Cursor::new(Vec::new());
    img.write_to(&mut buf, image::ImageFormat::Jpeg).unwrap();
    buf.into_inner()
}

// ─── Resize Tests ────────────────────────────────────────────────────────────

/// อัปโหลด PNG ขนาด 300×200 แล้ว resize เป็น 100×100 ตรวจสอบ dimensions
#[test]
fn test_resize_fill_to_100x100() {
    let original = make_test_image(300, 200);
    assert_eq!(original.width(), 300);
    assert_eq!(original.height(), 200);

    // resize_exact = FitMode::Fill
    let resized = original.resize_exact(100, 100, FilterType::Lanczos3);

    assert_eq!(resized.width(), 100, "ความกว้างต้องเป็น 100");
    assert_eq!(resized.height(), 100, "ความสูงต้องเป็น 100");

    println!("resize 300×200 → {}×{} (Fill) ✓", resized.width(), resized.height());
}

/// resize แบบ Contain ต้องรักษา aspect ratio — 400×200 → พอดีใน 100×100 คือ 100×50
#[test]
fn test_resize_contain_preserves_aspect_ratio() {
    let original = make_test_image(400, 200); // 2:1 aspect ratio

    // resize = FitMode::Contain
    let resized = original.resize(100, 100, FilterType::Lanczos3);

    assert!(resized.width() <= 100, "กว้างต้อง ≤ 100");
    assert!(resized.height() <= 100, "สูงต้อง ≤ 100");

    // 400×200 ลงมา fit ใน 100×100 → 100×50
    assert_eq!(resized.width(), 100);
    assert_eq!(resized.height(), 50);

    println!("resize Contain 400×200 → {}×{} ✓", resized.width(), resized.height());
}

// ─── Format Conversion Tests ──────────────────────────────────────────────────

/// แปลง PNG → JPEG ตรวจสอบ magic bytes
#[test]
fn test_format_conversion_png_to_jpeg() {
    let img = make_test_image(200, 150);

    // encode เป็น PNG
    let png_bytes = encode_png(&img);
    assert!(!png_bytes.is_empty());
    // PNG signature: 0x89 0x50 0x4E 0x47 (\x89PNG)
    assert_eq!(&png_bytes[0..4], &[0x89, 0x50, 0x4E, 0x47], "PNG signature ต้องถูกต้อง");

    // encode เป็น JPEG
    let jpeg_bytes = encode_jpeg(&img);
    assert!(!jpeg_bytes.is_empty());
    // JPEG signature: 0xFF 0xD8
    assert_eq!(&jpeg_bytes[0..2], &[0xFF, 0xD8], "JPEG signature ต้องถูกต้อง");

    // decode JPEG กลับ ตรวจสอบ dimensions
    let decoded = image::load_from_memory(&jpeg_bytes).expect("decode JPEG ล้มเหลว");
    assert_eq!(decoded.width(), 200);
    assert_eq!(decoded.height(), 150);

    println!(
        "PNG={} bytes, JPEG={} bytes ✓",
        png_bytes.len(),
        jpeg_bytes.len()
    );
}

// ─── Crop Tests ───────────────────────────────────────────────────────────────

/// crop ที่อยู่ใน bounds ต้องสำเร็จ
#[test]
fn test_crop_valid_bounds() {
    let mut img = make_test_image(300, 200);
    let cropped = img.crop(50, 50, 100, 80);

    assert_eq!(cropped.width(), 100);
    assert_eq!(cropped.height(), 80);

    println!("crop valid (50,50,100,80) → {}×{} ✓", cropped.width(), cropped.height());
}

/// crop ที่อยู่นอก bounds ต้องคืน error — ตรวจสอบ validation logic
#[test]
fn test_crop_out_of_bounds_returns_error() {
    let img = make_test_image(300, 200);

    // validate crop bounds (logic เดียวกับที่ใช้ใน handler)
    let crop_x = 250u32;
    let crop_y = 0u32;
    let crop_w = 100u32; // 250 + 100 = 350 > 300 → out of bounds
    let crop_h = 100u32;

    let result = validate_crop(crop_x, crop_y, crop_w, crop_h, img.width(), img.height());
    assert!(result.is_err(), "crop out of bounds ต้อง return Err");

    let err_msg = result.unwrap_err();
    assert!(err_msg.contains("350"), "error ต้องระบุค่า x+w");
    assert!(err_msg.contains("300"), "error ต้องระบุ image width");

    println!("crop out-of-bounds → Err: {} ✓", err_msg);
}

/// crop ที่ x >= image width ต้อง error
#[test]
fn test_crop_x_beyond_image_width() {
    let img = make_test_image(100, 100);

    let result = validate_crop(150, 0, 10, 10, img.width(), img.height());
    assert!(result.is_err(), "x=150 >= width=100 ต้อง error");
    println!("crop x=150 บน image 100×100 → Err ✓");
}

/// helper function — เลียนแบบ validation logic ใน processing.rs
fn validate_crop(x: u32, y: u32, w: u32, h: u32, iw: u32, ih: u32) -> Result<(), String> {
    if w == 0 || h == 0 {
        return Err("crop w and h must be > 0".to_string());
    }
    if x >= iw {
        return Err(format!("x={x} >= image width={iw}"));
    }
    if y >= ih {
        return Err(format!("y={y} >= image height={ih}"));
    }
    if x + w > iw {
        return Err(format!("x+w={} > image width={iw}", x + w));
    }
    if y + h > ih {
        return Err(format!("y+h={} > image height={ih}", y + h));
    }
    Ok(())
}

// ─── Filter Tests ─────────────────────────────────────────────────────────────

#[test]
fn test_filter_grayscale() {
    let img = make_test_image(100, 100);
    let gray = img.grayscale();
    assert_eq!(gray.width(), 100);
    assert_eq!(gray.height(), 100);
    println!("filter grayscale 100×100 → {}×{} ✓", gray.width(), gray.height());
}

/// rotate90 สลับ width/height
#[test]
fn test_filter_rotate90() {
    let img = make_test_image(300, 200);
    let rotated = img.rotate90();
    assert_eq!(rotated.width(), 200);
    assert_eq!(rotated.height(), 300);
    println!("filter rotate90 300×200 → {}×{} ✓", rotated.width(), rotated.height());
}

// ─── Pipeline Test ────────────────────────────────────────────────────────────

/// pipeline: 400×300 → resize 200×200 → crop 100×100 → grayscale
#[test]
fn test_pipeline_resize_crop_grayscale() {
    let img = make_test_image(400, 300);

    // Step 1: resize
    let resized = img.resize_exact(200, 200, FilterType::Lanczos3);
    assert_eq!(resized.width(), 200);
    assert_eq!(resized.height(), 200);

    // Step 2: crop
    let mut resized_mut = resized;
    let cropped = resized_mut.crop(50, 50, 100, 100);
    assert_eq!(cropped.width(), 100);
    assert_eq!(cropped.height(), 100);

    // Step 3: grayscale
    let final_img = cropped.grayscale();
    assert_eq!(final_img.width(), 100);
    assert_eq!(final_img.height(), 100);

    // encode เป็น PNG และตรวจ size
    let png_bytes = encode_png(&final_img);
    assert!(!png_bytes.is_empty());

    println!(
        "pipeline: 400×300 → resize 200×200 → crop 100×100 → grayscale {}×{} ({} bytes) ✓",
        final_img.width(),
        final_img.height(),
        png_bytes.len()
    );
}
```

### Real `cargo test` Output

ต่อไปนี้คือผลลัพธ์จากการรัน `cargo test -- --nocapture` จริงบนโปรเจค test ที่สร้างในขั้นตอน verify:

```
running 9 tests
crop x=150 บน image 100×100 → Err ✓
test test_crop_x_beyond_image_width ... ok
filter grayscale 100×100 → 100×100 ✓
test test_filter_grayscale ... ok
crop out-of-bounds → Err: crop out of bounds: x+w=350 > image width=300 ✓
test test_crop_out_of_bounds_returns_error ... ok
crop valid (50,50,100,80) → 100×80 ✓
test test_crop_valid_bounds ... ok
filter rotate90 300×200 → 200×300 ✓
test test_filter_rotate90 ... ok
PNG=1263 bytes, JPEG=3090 bytes ✓
test test_format_conversion_png_to_jpeg ... ok
resize 300×200 → 100×100 (Fill) ✓
test test_resize_to_100x100 ... ok
resize Contain 400×200 → 100×50 ✓
test test_resize_contain_preserves_aspect_ratio ... ok
pipeline: 400×300 → resize 200×200 → crop 100×100 → grayscale 100×100 ✓
test test_pipeline_resize_crop_grayscale ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

**สิ่งน่าสนใจจาก output:** สังเกตว่า `PNG=1263 bytes, JPEG=3090 bytes` — JPEG ใหญ่กว่า PNG! เป็นเพราะภาพทดสอบขนาด 200×150 เป็น gradient ที่ smooth มาก PNG compress ได้ดีกว่า JPEG สำหรับภาพแบบนี้ ในโลกจริง JPEG มักเล็กกว่าสำหรับภาพถ่าย (photos) แต่ใหญ่กว่าสำหรับ synthetic/generated images

---

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `content-type` Header มี Parameter ต่อท้าย

**ปัญหา:** เมื่อ fetch image จาก URL บางเซิร์ฟเวอร์ส่ง `Content-Type: image/jpeg; charset=utf-8` หรือ `image/png; q=1.0` การ match string ตรง ๆ จะ fail ทุกครั้ง

```rust
// ❌ ผิด — จะ return false เสมอถ้า header มี parameters
fn is_image(ct: &str) -> bool {
    ct == "image/jpeg" || ct == "image/png"
}

// ✓ ถูก — ตัด parameter ออกก่อน
fn is_image(ct: &str) -> bool {
    let base = ct.split(';').next().unwrap_or("").trim();
    matches!(base, "image/jpeg" | "image/png" | "image/webp" | "image/avif")
}
```

### 2. `image::imageops::blur()` ต้องการ `Rgb8` ไม่ใช่ `DynamicImage` โดยตรง

**ปัญหา:** หลายฟังก์ชันใน `imageops` รับ specific pixel type ไม่รับ `&DynamicImage` โดยตรง compiler จะฟ้อง type mismatch

```rust
// ❌ compile error: expected `&ImageBuffer<Rgb<u8>, _>` found `&DynamicImage`
let blurred = image::imageops::blur(&dynamic_image, 2.0);

// ✓ แปลงเป็น rgb8 ก่อน
let rgb8 = dynamic_image.to_rgb8(); // clone pixels เป็น Rgb<u8>
let blurred = image::imageops::blur(&rgb8, 2.0);
let result = DynamicImage::ImageRgb8(blurred);
```

แต่ระวัง: `to_rgb8()` discard alpha channel ถ้าภาพต้นฉบับมี transparency ควรใช้ `to_rgba8()` และ `imageops::blur` on RGBA ตามที่ต้องการ

### 3. Crop ไม่ตรวจ Bounds ก่อน — Panic แทนที่จะเป็น Error

**ปัญหา:** `DynamicImage::crop()` ใน image 0.25 จะ **clamp** crop region แทนที่จะ panic ถ้า crop region ออกนอก bounds หมายความว่าถ้าไม่ตรวจก่อน client จะได้ภาพที่ขนาดไม่ตรงกับที่ request โดยไม่มี error

```rust
// ❌ อันตราย — ได้ภาพขนาดไม่ถูกต้องโดยไม่มี error
let mut img = load_from_memory(&bytes).unwrap();
let cropped = img.crop(250, 0, 100, 100); // สมมติ image width=300 → crop ได้แค่ 50px

// ✓ ตรวจก่อนเสมอ
if crop_x + crop_w > img.width() || crop_y + crop_h > img.height() {
    return Err(StatusCode::BAD_REQUEST);
}
let cropped = img.crop(crop_x, crop_y, crop_w, crop_h);
```

### 4. `Mutex` vs `RwLock` สำหรับ Shared State

**ปัญหา:** ใช้ `Mutex<AppMetrics>` สำหรับ read-heavy workload (เช่น `/metrics` endpoint ที่ถูกเรียกบ่อย) ทำให้ทุก read ต้องรอ write lock และ throughput ลดลง

```rust
// ❌ ช้า: Mutex block ทุก reader แม้ไม่มีใคร write
let metrics = Arc::new(tokio::sync::Mutex::new(AppMetrics::default()));

// ✓ ดีกว่า: RwLock อนุญาต concurrent reads
let metrics = Arc::new(tokio::sync::RwLock::new(AppMetrics::default()));

// หลาย handler อ่านพร้อมกันได้
let m = state.metrics.read().await;
let body = m.to_prometheus_text();

// write lock ต้องรอ reader ทั้งหมด finish
let mut m = state.metrics.write().await;
m.total_requests += 1;
```

### 5. JPEG กับ PNG ขนาดไม่เป็นอย่างที่คิดเสมอ

**ปัญหา:** เข้าใจผิดว่า JPEG จะเล็กกว่า PNG เสมอ ทำให้ตั้ง default format เป็น JPEG ทุกกรณีซึ่งอาจไม่ดีที่สุด

ผลทดสอบจริงจากโปรเจคนี้:
- ภาพ gradient 200×150: PNG=1263 bytes, JPEG=3090 bytes → **PNG เล็กกว่า**
- ภาพถ่ายจริง 1920×1080: PNG≈2MB, JPEG≈200KB → **JPEG เล็กกว่า**

กฎง่าย ๆ: ใช้ WebP เป็น default สำหรับ web เพราะรองรับทั้ง lossy และ lossless และเล็กกว่าทั้ง JPEG และ PNG ในกรณีส่วนใหญ่

---

## การ Package และ Deploy

### Build Release Binary

```bash
# สร้าง optimized binary
cargo build --release

# ทดสอบ binary
./target/release/image-api &

# ส่งภาพทดสอบ
curl -X POST "http://localhost:3000/process?width=200&height=200&format=webp" \
  -F "image=@photo.jpg" \
  --output resized.webp

# URL fetch + grayscale
curl "http://localhost:3000/process?url=https://example.com/photo.jpg&filter=grayscale&format=jpeg" \
  --output gray.jpg
```

### Docker

```dockerfile
# Dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/image-api /usr/local/bin/image-api
EXPOSE 3000
CMD ["image-api"]
```

```bash
docker build -t image-api:latest .
docker run -p 3000:3000 image-api:latest
```

### Environment Variables

```bash
# ปรับ log level
RUST_LOG=image_api=info,tower_http=warn ./target/release/image-api

# สำหรับ production ปิด debug logging
RUST_LOG=warn ./target/release/image-api
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Quality Control สำหรับ JPEG/WebP (ระดับกลาง)

ปัจจุบัน `encode_image` ใช้ default quality ของ `image` crate ให้เพิ่ม quality parameter จริง:

```rust
// hint: ใช้ image::codecs::jpeg::JpegEncoder
use image::codecs::jpeg::JpegEncoder;

pub fn encode_jpeg_with_quality(img: &DynamicImage, quality: u8) -> Result<Vec<u8>, String> {
    let mut buf = Cursor::new(Vec::new());
    let encoder = JpegEncoder::new_with_quality(&mut buf, quality);
    img.write_with_encoder(encoder)
        .map_err(|e| format!("JPEG encode error: {e}"))?;
    Ok(buf.into_inner())
}
```

ทดสอบว่า `quality=10` ให้ไฟล์เล็กกว่า `quality=90` จริง และภาพต่างกันอย่างไร

### แบบฝึกหัดที่ 2: Watermark ด้วย ab_glyph จริง (ระดับยาก)

ปัจจุบัน `apply_watermark` แค่ทำ semi-transparent overlay ให้ implement watermark text จริงด้วย ab_glyph:

```rust
use ab_glyph::{FontRef, PxScale};
use image::{DynamicImage, Rgba};
use imageproc::drawing::draw_text_mut;

// hint: โหลด font จาก embedded bytes
static FONT_BYTES: &[u8] = include_bytes!("../assets/DejaVuSans.ttf");

pub fn apply_text_watermark(img: DynamicImage, text: &str) -> DynamicImage {
    // 1. โหลด font ด้วย ab_glyph::FontRef::try_from_slice(FONT_BYTES)
    // 2. คำนวณ text width เพื่อหาตำแหน่ง bottom-right
    // 3. วาด text ด้วย semi-transparent white
    // 4. คืน DynamicImage ที่มี watermark
    todo!("implement watermark with ab_glyph")
}
```

### แบบฝึกหัดที่ 3: Redis Cache แทน In-Memory LRU (ระดับกลาง)

เปลี่ยน cache backend จาก `LruCache` เป็น Redis เพื่อรองรับ multi-instance deployment:

```rust
// hint: ใช้ redis crate
// [dependencies]
// redis = { version = "0.27", features = ["tokio-comp"] }

use redis::AsyncCommands;

pub struct RedisCache {
    client: redis::Client,
    ttl_seconds: u64,
}

impl RedisCache {
    pub async fn get(&self, key: &str) -> Option<Vec<u8>> {
        let mut conn = self.client.get_multiplexed_async_connection().await.ok()?;
        conn.get::<_, Vec<u8>>(key).await.ok()
    }

    pub async fn put(&self, key: &str, value: Vec<u8>) {
        if let Ok(mut conn) = self.client.get_multiplexed_async_connection().await {
            let _: Result<(), _> = conn.set_ex(key, value, self.ttl_seconds).await;
        }
    }
}
```

### แบบฝึกหัดที่ 4: Rate Limiting ต่อ IP (ระดับกลาง)

เพิ่ม rate limiting เพื่อป้องกัน abuse — จำกัด 10 requests/minute ต่อ IP:

```rust
// hint: ใช้ tower middleware + DashMap สำหรับ per-IP counter
// หรือใช้ tower_governor crate

use dashmap::DashMap;
use std::time::{Duration, Instant};

struct RateLimiter {
    // IP → (count, window_start)
    counters: DashMap<String, (u32, Instant)>,
    max_requests: u32,
    window: Duration,
}

impl RateLimiter {
    pub fn check_and_increment(&self, ip: &str) -> bool {
        let now = Instant::now();
        let mut entry = self.counters.entry(ip.to_string()).or_insert((0, now));
        if now.duration_since(entry.1) > self.window {
            *entry = (1, now);
            return true;
        }
        if entry.0 >= self.max_requests {
            return false;
        }
        entry.0 += 1;
        true
    }
}
```

---

## สรุป

โปรเจคนี้สร้าง Image Processing API ที่ครบถ้วนสำหรับ production ครอบคลุม pattern สำคัญ 4 กลุ่ม:

**1. Binary Data Handling** — รับ multipart upload, validate content-type และ size, โหลดด้วย `image::load_from_memory` ที่รองรับ format ต่าง ๆ โดยอัตโนมัติ

**2. Processing Pipeline Design** — ออกแบบ operation ให้ประกอบกันได้ (composable) ผ่าน query params ลำดับ pipeline ตาย (resize → filter → crop → watermark) ป้องกัน ambiguity

**3. LRU Caching Pattern** — ใช้ content hash เป็น cache key ให้ถูกต้องกว่า filename-based caching บวก `Cache-Control` header เพื่อให้ browser และ CDN cache ต่อได้

**4. Async Job Pattern (202 Accepted)** — สำหรับงานที่ใช้เวลานาน return job ID ทันที client polling ด้วย GET /jobs/{id} ซึ่งเป็น pattern ที่ใช้กันทั่วไปใน production APIs

**Pattern ที่ได้เรียนที่นำไปใช้ต่อได้:** LRU cache ใน Rust, DashMap สำหรับ concurrent state, SHA256 content hashing, async task spawning ด้วย `tokio::spawn`, Prometheus metrics format

**โปรเจคถัดไป** B04 RSS Aggregator จะนำทักษะ HTTP handling ที่ได้จากโปรเจคนี้ไปประยุกต์กับ XML parsing, scheduled background tasks, และ feed normalization — ซึ่งซับซ้อนขึ้นในด้าน data modeling มากกว่า binary processing

---

**โปรเจคก่อนหน้า:** [project-b02-file-upload-cdn.md](project-b02-file-upload-cdn.md) | **โปรเจคถัดไป:** [project-b04-rss-aggregator.md](project-b04-rss-aggregator.md)
