# Project J07: REST API with OpenAPI Documentation

> โมดูล: J — Full-Stack & Web APIs | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

REST API ที่ดีในโลก production ไม่ได้มีแค่ code ที่ทำงานได้ — มันต้องมี **documentation ที่ถูกต้อง ครบถ้วน และ sync กับ implementation จริงเสมอ** OpenAPI (เดิมชื่อ Swagger) คือ specification มาตรฐานอุตสาหกรรมสำหรับบรรยาย REST API ที่ทีม frontend, QA, DevOps และ partner ทุกฝ่ายใช้ร่วมกัน

โปรเจคนี้สร้าง **Product Inventory REST API** ครบวงจรด้วย Rust — ตั้งแต่ออกแบบ resource, HTTP handlers, request validation, pagination จนถึง OpenAPI 3.0 spec ที่ generate อัตโนมัติจาก code annotations และ interactive API documentation ผ่าน web browser

ปัญหาที่โปรเจคนี้แก้ไข: ทีม developer มักเขียน API แล้ว documentation ตามไม่ทัน หรือ documentation เขียนแยกจาก code ทำให้ outdated เร็ว `utoipa` แก้ปัญหานี้ด้วยการ generate OpenAPI spec จาก Rust code โดยตรง ถ้า code เปลี่ยน spec ก็เปลี่ยนตาม

**Use case จริงในโลก production:**
- **Public API product** — เช่น Stripe, Twilio ที่ต้องมี interactive docs ให้ developer ทดสอบได้ทันที
- **Microservices** — แต่ละ service expose OpenAPI spec ให้ API Gateway และ service mesh อ่านได้อัตโนมัติ
- **Internal API** — ทีม frontend อ่าน spec เพื่อ generate TypeScript client อัตโนมัติด้วย tools เช่น `openapi-generator`
- **QA automation** — ทีม QA ใช้ OpenAPI spec generate test cases และ validate response schema

**Learning value:** โปรเจคนี้รวม pattern ที่ใช้ใน production API จริงทุกตัว — REST design, Axum extractors, utoipa annotations, validator integration, pagination และ API versioning

---

## สิ่งที่จะได้เรียนรู้

- **Axum router architecture** — `Router`, `State<T>`, `Json<T>`, `Path<T>`, `Query<T>` extractors, nested routes ด้วย `.nest()`
- **utoipa annotations** — `#[utoipa::path]`, `#[derive(ToSchema)]`, `#[derive(OpenApi)]` สำหรับ generate OpenAPI 3.0 spec อัตโนมัติ
- **Request validation** — `validator` crate, `#[validate]` attributes, returning 422 Unprocessable Entity พร้อม field errors
- **API versioning** — URL prefix pattern `/v1/`, `/v2/` ด้วย `.nest()` และ delegation strategy
- **Pagination pattern** — `Paginated<T>` wrapper, `PaginationMeta`, query params พร้อม clamping
- **Error type hierarchy** — `AppError` enum ที่ implement `IntoResponse`, map เป็น HTTP status codes ที่ถูกต้อง
- **Integration testing** — `tower::ServiceExt::oneshot()` สำหรับ test HTTP requests โดยไม่ต้องรัน server จริง
- **OpenAPI components** — schema definitions, security schemes, tags, info metadata

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20** — Rust fundamentals: ownership, structs, enums, traits, generics
- **Part 21–40** — Error handling ด้วย `Result`/`Option`, closures, iterators
- **Part 41–60** — `tokio` async runtime, `Arc`/`Mutex`, HTTP พื้นฐาน, `serde` JSON
- **Part 61–80** — Axum web framework, Tower middleware, HTTP extractors
- **Part 81–95** — Advanced traits, type-level programming, derive macros
- ความคุ้นเคยกับ REST API design และ HTTP status codes จะช่วยมาก

---

## โครงสร้างโปรเจค (Project Layout)

```
rest-openapi/
├── src/
│   ├── lib.rs          ← export ทุก module
│   ├── main.rs         ← entry point: start server
│   ├── error.rs        ← AppError enum + IntoResponse + ErrorResponse
│   ├── models.rs       ← Product, CreateProductRequest, UpdateProductRequest, Category
│   ├── handlers.rs     ← handler functions + ProductStore state
│   ├── router.rs       ← create_app(), ApiDoc derive, route registration
│   ├── pagination.rs   ← PaginationParams, Paginated<T>, PaginationMeta
│   ├── validation.rs   ← validate_input<T>() helper
│   └── versioning.rs   ← ProductV2, VersionInfo (v2 API types)
├── tests/
│   └── integration_test.rs  ← integration tests ด้วย tower::ServiceExt
└── Cargo.toml
```

---

## การออกแบบ (Architecture & Design)

### REST Resource Model

โปรเจคนี้ใช้ **Resource-Oriented Design** — ทุก entity เป็น resource ที่มี URL ชัดเจน และใช้ HTTP verbs มาตรฐาน:

```
Resource: Product
Base URL: /v1/products

GET    /v1/products           → list products (paginated, filterable)
POST   /v1/products           → create product
GET    /v1/products/{id}      → get product by ID
PUT    /v1/products/{id}      → update product (partial)
DELETE /v1/products/{id}      → delete product

GET    /health                → health check
GET    /redoc                 → API documentation UI
GET    /api-docs/openapi.json → OpenAPI spec (machine-readable)
```

### HTTP Status Codes ที่ใช้

| Situation | Status Code | เมื่อไหร่ |
|-----------|-------------|----------|
| GET/PUT สำเร็จ | 200 OK | ดึงหรือแก้ไข resource สำเร็จ |
| POST สำเร็จ | 201 Created | สร้าง resource ใหม่สำเร็จ |
| DELETE สำเร็จ | 204 No Content | ลบ resource สำเร็จ (ไม่มี body) |
| ไม่พบ resource | 404 Not Found | ID ที่ขอไม่มีใน store |
| Validation ผิด | 422 Unprocessable Entity | field ไม่ผ่าน validation rules |
| Server error | 500 Internal Server Error | ข้อผิดพลาดที่ไม่คาดคิด |

### ทำไมถึงใช้ `utoipa` แทนการเขียน YAML มือ

แนวทางแบบเดิมคือเขียน OpenAPI YAML แยกต่างหากแล้ว sync กับ code เอง ซึ่งมีปัญหาคือ:
- ลืม update spec เมื่อ code เปลี่ยน
- ชื่อ field ใน spec ต่างจาก struct จริง
- Type ใน spec ไม่ match กับ Rust type จริง

`utoipa` แก้ปัญหาด้วย **code-first approach** — เขียน annotation บน Rust code แล้ว derive spec อัตโนมัติ ถ้า compile ผ่าน spec ก็ถูกต้อง

### Data Flow

```
HTTP Request
     │
     ▼
[Axum Router]  →  route matching  →  middleware (CORS, tracing)
     │
     ├── /health        → health_check()
     ├── /redoc         → Redoc UI (embedded HTML)
     ├── /api-docs/...  → ApiDoc::openapi() as JSON
     │
     └── /v1/*          → [v1 nested router]
              │
              ├── GET /products          → list_products()
              │        ├── Query<PaginationParams> extraction
              │        ├── Query<ProductFilter> extraction
              │        ├── filter + sort products
              │        └── wrap in Paginated<Product>
              │
              ├── POST /products         → create_product()
              │        ├── Json<CreateProductRequest> extraction
              │        ├── validate_input(&req) → 422 if invalid
              │        ├── Product::new(req)
              │        └── store.insert()
              │
              ├── GET /products/{id}     → get_product()
              │        ├── Path<Uuid> extraction
              │        └── store.get(&id) → 404 if missing
              │
              ├── PUT /products/{id}     → update_product()
              │        ├── Path<Uuid> + Json<UpdateProductRequest>
              │        ├── validate_input(&req)
              │        └── product.apply_update(req)
              │
              └── DELETE /products/{id}  → delete_product()
                       ├── Path<Uuid> extraction
                       └── store.remove(&id) → 204
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Setup โปรเจคและกำหนด Dependencies

เริ่มจากสร้างโปรเจคและ `Cargo.toml` พร้อม dependencies ที่ต้องใช้ทั้งหมด

```bash
cargo new rest-openapi --lib
cd rest-openapi
```

**`Cargo.toml`:**
```toml
[package]
name = "rest-openapi"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "server"
path = "src/main.rs"

[lib]
name = "rest_openapi"
path = "src/lib.rs"

[dependencies]
axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
utoipa = { version = "4", features = ["axum_extras"] }
utoipa-redoc = { version = "4", features = ["axum"] }
validator = { version = "0.18", features = ["derive"] }
tower = { version = "0.4", features = ["util"] }
tower-http = { version = "0.5", features = ["cors", "trace"] }
thiserror = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }

[dev-dependencies]
tower = { version = "0.4", features = ["util"] }
http-body-util = "0.1"
tokio = { version = "1", features = ["full"] }
```

**Crates หลักและบทบาทของแต่ละตัว:**

| Crate | บทบาท |
|-------|--------|
| `axum` | HTTP web framework บน Tower |
| `utoipa` | Derive macros สำหรับ generate OpenAPI spec |
| `utoipa-redoc` | Serve ReDoc UI เป็น embedded HTML |
| `validator` | Field-level validation ด้วย derive macros |
| `tower` | Middleware abstractions + test utilities |
| `tower-http` | CORS, tracing middleware สำหรับ Axum |
| `uuid` | UUID v4 สำหรับ resource IDs |
| `chrono` | Datetime สำหรับ created_at/updated_at |
| `thiserror` | Derive macro สำหรับ error types |

**หมายเหตุเรื่อง `utoipa-swagger-ui` vs `utoipa-redoc`:**

`utoipa-swagger-ui` ต้อง download Swagger UI assets จาก GitHub ตอน build time ซึ่งอาจมีปัญหาในสภาพแวดล้อม CI/CD ที่ไม่มี internet `utoipa-redoc` embed ReDoc assets ตรงใน binary ทำให้ build ได้แม้ไม่มี internet และ binary ขนาดเล็กกว่า

---

### ขั้นที่ 2: ออกแบบ Data Models พร้อม ToSchema

สร้าง `src/models.rs` — นี่คือหัวใจของ API ทุก struct ที่จะปรากฏใน OpenAPI spec ต้อง derive `ToSchema`

**`src/models.rs`:**
```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use uuid::Uuid;
use validator::Validate;

/// หมวดหมู่สินค้า
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, ToSchema)]
#[serde(rename_all = "snake_case")]
pub enum Category {
    Electronics,
    Clothing,
    Books,
    Food,
    Other,
}

/// ข้อมูลสินค้า (Product) — response schema หลัก
#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
pub struct Product {
    /// UUID ของสินค้า
    pub id: Uuid,
    /// ชื่อสินค้า
    pub name: String,
    /// คำอธิบายสินค้า
    pub description: Option<String>,
    /// ราคาสินค้า (บาท)
    pub price: f64,
    /// จำนวนสินค้าในสต็อก
    pub stock: u32,
    /// หมวดหมู่
    pub category: Category,
    /// เวลาที่สร้าง
    pub created_at: DateTime<Utc>,
    /// เวลาที่แก้ไขล่าสุด
    pub updated_at: DateTime<Utc>,
}

/// Request body สำหรับสร้างสินค้าใหม่
#[derive(Debug, Clone, Serialize, Deserialize, ToSchema, Validate)]
pub struct CreateProductRequest {
    /// ชื่อสินค้า (2-200 ตัวอักษร)
    #[validate(length(min = 2, max = 200, message = "Name must be 2-200 characters"))]
    pub name: String,

    /// คำอธิบายสินค้า (สูงสุด 2000 ตัวอักษร)
    #[validate(length(max = 2000, message = "Description must not exceed 2000 characters"))]
    pub description: Option<String>,

    /// ราคาสินค้า (ต้องมากกว่า 0)
    #[validate(range(min = 0.01, message = "Price must be greater than 0"))]
    pub price: f64,

    /// จำนวนสินค้าในสต็อก
    pub stock: u32,

    /// หมวดหมู่สินค้า
    pub category: Category,
}

/// Request body สำหรับแก้ไขสินค้า (partial update)
#[derive(Debug, Clone, Serialize, Deserialize, ToSchema, Validate)]
pub struct UpdateProductRequest {
    #[validate(length(min = 2, max = 200, message = "Name must be 2-200 characters"))]
    pub name: Option<String>,

    #[validate(length(max = 2000, message = "Description must not exceed 2000 characters"))]
    pub description: Option<String>,

    #[validate(range(min = 0.01, message = "Price must be greater than 0"))]
    pub price: Option<f64>,

    pub stock: Option<u32>,
    pub category: Option<Category>,
}

/// Query parameters สำหรับกรองสินค้า
#[derive(Debug, Clone, Deserialize, ToSchema)]
pub struct ProductFilter {
    pub category: Option<Category>,
    pub min_price: Option<f64>,
    pub max_price: Option<f64>,
    pub search: Option<String>,
}

impl Product {
    pub fn new(req: CreateProductRequest) -> Self {
        let now = Utc::now();
        Product {
            id: Uuid::new_v4(),
            name: req.name,
            description: req.description,
            price: req.price,
            stock: req.stock,
            category: req.category,
            created_at: now,
            updated_at: now,
        }
    }

    pub fn apply_update(&mut self, req: UpdateProductRequest) {
        if let Some(name) = req.name { self.name = name; }
        if let Some(desc) = req.description { self.description = Some(desc); }
        if let Some(price) = req.price { self.price = price; }
        if let Some(stock) = req.stock { self.stock = stock; }
        if let Some(cat) = req.category { self.category = cat; }
        self.updated_at = Utc::now();
    }
}
```

**จุดสำคัญในการออกแบบ Models:**

1. **`#[derive(ToSchema)]`** บน struct และ enum ทำให้ utoipa รู้ว่าต้อง include ใน OpenAPI components

2. **`#[derive(Validate)]`** บน request structs และใช้ `#[validate(...)]` attribute บน fields เพื่อกำหนด constraints

3. **Doc comments (`///`)** บน fields จะปรากฏใน OpenAPI spec เป็น field descriptions — เขียนให้ดีเพราะ frontend developer จะเห็น

4. **`UpdateProductRequest`** ใช้ `Option<T>` บนทุก field เพื่อรองรับ partial update (PATCH-style) แต่ยังคงใช้ PUT verb ตาม REST convention

5. **`#[serde(rename_all = "snake_case")]`** บน `Category` enum ทำให้ JSON ส่ง `"electronics"` แทน `"Electronics"`

---

### ขั้นที่ 3: Error Handling ที่ครบถ้วน

สร้าง `src/error.rs` — error system ที่ดีต้องแปลง domain errors เป็น HTTP response ที่มีโครงสร้างชัดเจน

**`src/error.rs`:**
```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;
use utoipa::ToSchema;

/// Error response body สำหรับ API errors ทุกชนิด
#[derive(Debug, Serialize, ToSchema)]
pub struct ErrorResponse {
    /// HTTP status code
    pub status: u16,
    /// Error message หลัก
    pub message: String,
    /// ชื่อ error type
    pub error: String,
    /// รายละเอียด validation errors (ถ้ามี)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub details: Option<Vec<FieldError>>,
}

/// Field-level validation error
#[derive(Debug, Serialize, ToSchema)]
pub struct FieldError {
    pub field: String,
    pub message: String,
}

/// Application error types
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Resource not found: {0}")]
    NotFound(String),

    #[error("Validation failed")]
    Validation(Vec<FieldError>),

    #[error("Conflict: {0}")]
    Conflict(String),

    #[error("Bad request: {0}")]
    BadRequest(String),

    #[error("Internal error: {0}")]
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, error_type, message, details) = match self {
            AppError::NotFound(msg) =>
                (StatusCode::NOT_FOUND, "NOT_FOUND", msg, None),
            AppError::Validation(errs) => (
                StatusCode::UNPROCESSABLE_ENTITY,
                "VALIDATION_ERROR",
                "Validation failed".to_string(),
                Some(errs),
            ),
            AppError::Conflict(msg) =>
                (StatusCode::CONFLICT, "CONFLICT", msg, None),
            AppError::BadRequest(msg) =>
                (StatusCode::BAD_REQUEST, "BAD_REQUEST", msg, None),
            AppError::Internal(msg) =>
                (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", msg, None),
        };

        let body = ErrorResponse {
            status: status.as_u16(),
            message,
            error: error_type.to_string(),
            details,
        };

        (status, Json(body)).into_response()
    }
}
```

**ตัวอย่าง Error Response ที่ client จะได้รับ:**

```json
// 404 Not Found
{
  "status": 404,
  "message": "Product abc123 not found",
  "error": "NOT_FOUND"
}

// 422 Unprocessable Entity (validation error)
{
  "status": 422,
  "message": "Validation failed",
  "error": "VALIDATION_ERROR",
  "details": [
    { "field": "name", "message": "Name must be 2-200 characters" },
    { "field": "price", "message": "Price must be greater than 0" }
  ]
}
```

**หลักการออกแบบ Error Responses:**

- **Machine-readable `error` field** — frontend ใช้ switch บน `error` field แทน HTTP status code
- **Human-readable `message`** — แสดงต่อ user ได้โดยตรง
- **Structured `details`** — สำหรับ validation errors ที่มีหลาย field
- **`#[serde(skip_serializing_if = "Option::is_none")]`** — ไม่ส่ง `"details": null` ในกรณีที่ไม่มี validation errors

---

### ขั้นที่ 4: Request Validation ด้วย `validator`

สร้าง `src/validation.rs` — wrapper function ที่แปลง `ValidationErrors` เป็น `AppError`

**`src/validation.rs`:**
```rust
use crate::error::{AppError, FieldError};
use validator::{Validate, ValidationErrors};

/// แปลง ValidationErrors จาก validator crate เป็น AppError::Validation
pub fn validate_input<T: Validate>(input: &T) -> Result<(), AppError> {
    input.validate().map_err(|errs| {
        let field_errors = collect_field_errors(&errs);
        AppError::Validation(field_errors)
    })
}

fn collect_field_errors(errs: &ValidationErrors) -> Vec<FieldError> {
    let mut result = Vec::new();
    for (field, errors) in errs.field_errors() {
        for err in errors {
            let message = err
                .message
                .as_ref()
                .map(|m| m.to_string())
                .unwrap_or_else(|| format!("Invalid value for field '{}'", field));
            result.push(FieldError {
                field: field.to_string(),
                message,
            });
        }
    }
    result
}
```

**Validation Attributes ที่ `validator` รองรับ:**

```rust
#[validate(length(min = 2, max = 200))]         // ความยาว string
#[validate(range(min = 0.01, max = 999999.0))]  // ช่วงตัวเลข
#[validate(email)]                               // รูปแบบ email
#[validate(url)]                                 // รูปแบบ URL
#[validate(regex = "RE_PHONE")]                  // regular expression
#[validate(contains = "@")]                      // ต้องมี substring
#[validate(must_match(other = "password"))]      // ต้อง match field อื่น
#[validate(custom = "validate_unique_username")] // custom validation function
```

---

### ขั้นที่ 5: Pagination System

สร้าง `src/pagination.rs` — generic pagination wrapper ที่ใช้ได้กับทุก resource type

**`src/pagination.rs`:**
```rust
use serde::{Deserialize, Serialize};
use utoipa::{IntoParams, ToSchema};

/// Query parameters สำหรับ pagination
#[derive(Debug, Clone, Deserialize, IntoParams)]
pub struct PaginationParams {
    /// หน้าที่ต้องการ (เริ่มต้นที่ 1)
    #[param(default = 1, minimum = 1)]
    pub page: Option<u64>,
    /// จำนวน item ต่อหน้า (1-100)
    #[param(default = 20, minimum = 1, maximum = 100)]
    pub per_page: Option<u64>,
}

impl PaginationParams {
    pub fn page(&self) -> u64 { self.page.unwrap_or(1).max(1) }
    pub fn per_page(&self) -> u64 { self.per_page.unwrap_or(20).clamp(1, 100) }
    pub fn offset(&self) -> u64 { (self.page() - 1) * self.per_page() }
}

/// Paginated response wrapper
#[derive(Debug, Serialize, ToSchema)]
pub struct Paginated<T: Serialize> {
    pub data: Vec<T>,
    pub meta: PaginationMeta,
}

/// Metadata สำหรับ pagination
#[derive(Debug, Serialize, ToSchema)]
pub struct PaginationMeta {
    pub page: u64,
    pub per_page: u64,
    pub total: u64,
    pub total_pages: u64,
    pub has_next: bool,
    pub has_prev: bool,
}

impl<T: Serialize> Paginated<T> {
    pub fn new(data: Vec<T>, page: u64, per_page: u64, total: u64) -> Self {
        let total_pages = if total == 0 {
            0
        } else {
            (total + per_page - 1) / per_page
        };
        Paginated {
            data,
            meta: PaginationMeta {
                page, per_page, total, total_pages,
                has_next: page < total_pages,
                has_prev: page > 1,
            },
        }
    }
}
```

**ตัวอย่าง Paginated Response:**
```json
{
  "data": [
    { "id": "...", "name": "MacBook Pro 16\"", "price": 89990.0, "..." }
  ],
  "meta": {
    "page": 2,
    "per_page": 10,
    "total": 35,
    "total_pages": 4,
    "has_next": true,
    "has_prev": true
  }
}
```

**`#[derive(IntoParams)]`** บน `PaginationParams` ทำให้ utoipa รู้ว่า query parameters เหล่านี้ต้อง document ใน OpenAPI spec โดยอัตโนมัติ

---

### ขั้นที่ 6: HTTP Handlers พร้อม utoipa Annotations

สร้าง `src/handlers.rs` — handler functions ทุกตัวต้องมี `#[utoipa::path]` annotation

**`src/handlers.rs`:**
```rust
use std::sync::{Arc, Mutex};
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use uuid::Uuid;
use std::collections::HashMap;
use crate::{
    error::AppError,
    models::{Category, CreateProductRequest, Product, ProductFilter, UpdateProductRequest},
    pagination::{Paginated, PaginationParams},
    validation::validate_input,
};

/// Application state — ใช้ in-memory store แทน database
pub type ProductStore = Arc<Mutex<HashMap<Uuid, Product>>>;

/// GET /v1/products — รายการสินค้าพร้อม pagination และ filter
#[utoipa::path(
    get,
    path = "/v1/products",
    params(
        PaginationParams,
        ("category" = Option<String>, Query, description = "Filter by category"),
        ("min_price" = Option<f64>, Query, description = "Minimum price"),
        ("max_price" = Option<f64>, Query, description = "Maximum price"),
        ("search" = Option<String>, Query, description = "Search by name"),
    ),
    responses(
        (status = 200, description = "List of products"),
        (status = 500, description = "Internal error"),
    ),
    tag = "products",
)]
pub async fn list_products(
    State(store): State<ProductStore>,
    Query(pagination): Query<PaginationParams>,
    Query(filter): Query<ProductFilter>,
) -> impl IntoResponse {
    let store = store.lock().unwrap();

    let mut products: Vec<Product> = store
        .values()
        .filter(|p| {
            if let Some(ref cat) = filter.category {
                if &p.category != cat { return false; }
            }
            if let Some(min) = filter.min_price {
                if p.price < min { return false; }
            }
            if let Some(max) = filter.max_price {
                if p.price > max { return false; }
            }
            if let Some(ref search) = filter.search {
                if !p.name.to_lowercase().contains(&search.to_lowercase()) {
                    return false;
                }
            }
            true
        })
        .cloned()
        .collect();

    products.sort_by(|a, b| a.created_at.cmp(&b.created_at));

    let total = products.len() as u64;
    let page = pagination.page();
    let per_page = pagination.per_page();
    let offset = pagination.offset() as usize;

    let page_data: Vec<Product> = products
        .into_iter()
        .skip(offset)
        .take(per_page as usize)
        .collect();

    Json(Paginated::new(page_data, page, per_page, total)).into_response()
}

/// POST /v1/products — สร้างสินค้าใหม่
#[utoipa::path(
    post,
    path = "/v1/products",
    request_body = CreateProductRequest,
    responses(
        (status = 201, description = "Product created", body = Product),
        (status = 422, description = "Validation error"),
    ),
    tag = "products",
)]
pub async fn create_product(
    State(store): State<ProductStore>,
    Json(req): Json<CreateProductRequest>,
) -> Result<impl IntoResponse, AppError> {
    validate_input(&req)?;
    let product = Product::new(req);
    let mut store = store.lock().unwrap();
    store.insert(product.id, product.clone());
    Ok((StatusCode::CREATED, Json(product)))
}

/// GET /v1/products/:id — ดูสินค้าตาม ID
#[utoipa::path(
    get,
    path = "/v1/products/{id}",
    params(("id" = Uuid, Path, description = "Product UUID")),
    responses(
        (status = 200, description = "Product found", body = Product),
        (status = 404, description = "Product not found"),
    ),
    tag = "products",
)]
pub async fn get_product(
    State(store): State<ProductStore>,
    Path(id): Path<Uuid>,
) -> Result<Json<Product>, AppError> {
    let store = store.lock().unwrap();
    store
        .get(&id)
        .cloned()
        .map(Json)
        .ok_or_else(|| AppError::NotFound(format!("Product {} not found", id)))
}

/// PUT /v1/products/:id — แก้ไขสินค้า
#[utoipa::path(
    put,
    path = "/v1/products/{id}",
    params(("id" = Uuid, Path, description = "Product UUID")),
    request_body = UpdateProductRequest,
    responses(
        (status = 200, description = "Product updated", body = Product),
        (status = 404, description = "Product not found"),
        (status = 422, description = "Validation error"),
    ),
    tag = "products",
)]
pub async fn update_product(
    State(store): State<ProductStore>,
    Path(id): Path<Uuid>,
    Json(req): Json<UpdateProductRequest>,
) -> Result<Json<Product>, AppError> {
    validate_input(&req)?;
    let mut store = store.lock().unwrap();
    let product = store
        .get_mut(&id)
        .ok_or_else(|| AppError::NotFound(format!("Product {} not found", id)))?;
    product.apply_update(req);
    Ok(Json(product.clone()))
}

/// DELETE /v1/products/:id — ลบสินค้า
#[utoipa::path(
    delete,
    path = "/v1/products/{id}",
    params(("id" = Uuid, Path, description = "Product UUID")),
    responses(
        (status = 204, description = "Product deleted"),
        (status = 404, description = "Product not found"),
    ),
    tag = "products",
)]
pub async fn delete_product(
    State(store): State<ProductStore>,
    Path(id): Path<Uuid>,
) -> Result<StatusCode, AppError> {
    let mut store = store.lock().unwrap();
    if store.remove(&id).is_none() {
        return Err(AppError::NotFound(format!("Product {} not found", id)));
    }
    Ok(StatusCode::NO_CONTENT)
}

/// GET /health — Health check endpoint
#[utoipa::path(
    get,
    path = "/health",
    responses((status = 200, description = "Service is healthy")),
    tag = "system",
)]
pub async fn health_check() -> impl IntoResponse {
    Json(serde_json::json!({
        "status": "ok",
        "service": "product-api",
        "version": "1.0.0"
    }))
}
```

**โครงสร้างของ `#[utoipa::path]` annotation:**

```rust
#[utoipa::path(
    METHOD,                        // HTTP method: get, post, put, delete, patch
    path = "/v1/path/{param}",    // URL path (ต้อง match กับ route registration)
    params(
        ParamStruct,               // struct ที่ derive IntoParams
        ("name" = Type, Where, description = "..."),  // inline param
    ),
    request_body = RequestType,    // body schema (optional)
    responses(
        (status = STATUS, description = "...", body = ResponseType),
    ),
    tag = "tag-name",              // grouping ใน Swagger UI
    security(("bearer_auth" = [])) // ถ้า endpoint ต้องการ auth
)]
```

---

### ขั้นที่ 7: OpenAPI Spec Generation และ Router Setup

สร้าง `src/router.rs` — รวม routes, OpenAPI spec และ documentation UI

**`src/router.rs`:**
```rust
use axum::{routing::get, Router};
use utoipa::OpenApi;
use utoipa_redoc::{Redoc, Servable};
use crate::handlers::{
    ProductStore, create_product, delete_product,
    get_product, health_check, list_products, update_product,
};

/// OpenAPI specification definition
/// #[derive(OpenApi)] generate spec อัตโนมัติจาก annotations
#[derive(OpenApi)]
#[openapi(
    paths(
        crate::handlers::health_check,
        crate::handlers::list_products,
        crate::handlers::get_product,
        crate::handlers::create_product,
        crate::handlers::update_product,
        crate::handlers::delete_product,
    ),
    components(
        schemas(
            crate::models::Product,
            crate::models::CreateProductRequest,
            crate::models::UpdateProductRequest,
            crate::models::Category,
            crate::pagination::Paginated<crate::models::Product>,
            crate::pagination::PaginationMeta,
            crate::error::ErrorResponse,
            crate::error::FieldError,
        ),
    ),
    tags(
        (name = "products", description = "Product management API"),
        (name = "system", description = "System health and info"),
    ),
    info(
        title = "Product API",
        version = "1.0.0",
        description = "REST API with OpenAPI 3.0 documentation"
    ),
)]
pub struct ApiDoc;

/// สร้าง Axum router พร้อม documentation UI
pub fn create_app(store: ProductStore) -> Router {
    // V1 routes
    let v1_routes = Router::new()
        .route("/products", get(list_products).post(create_product))
        .route(
            "/products/:id",
            get(get_product).put(update_product).delete(delete_product),
        );

    Router::new()
        .route("/health", get(health_check))
        .nest("/v1", v1_routes)
        // ReDoc documentation UI
        .merge(Redoc::with_url("/redoc", ApiDoc::openapi()))
        // OpenAPI spec JSON endpoint
        .route(
            "/api-docs/openapi.json",
            get(|| async { axum::Json(ApiDoc::openapi()) }),
        )
        .with_state(store)
}
```

**ความสัมพันธ์ระหว่าง `paths()` และ `components()` ใน `#[openapi]`:**

- **`paths(...)`** — list ของ handler functions ที่มี `#[utoipa::path]` annotation — utoipa จะ include path ของ handlers เหล่านี้ใน spec
- **`components(schemas(...))`** — list ของ types ที่ต้องการ include ใน `#/components/schemas` section — ถ้าไม่ระบุ utoipa อาจสร้าง inline schema แทน

**ข้อควรระวัง:** path ที่ระบุใน `#[utoipa::path(path = "...")]` ต้องตรงกับ URL ที่ register ใน router จริง — utoipa ไม่ได้ check สิ่งนี้ให้อัตโนมัติ

---

### ขั้นที่ 8: API Versioning Strategy

การ version API ใน URL path เป็นวิธีที่ชัดเจนและ widely adopted เช่น Stripe (`/v1/charges`), Twilio, AWS API Gateway

**`src/versioning.rs` — V2 types:**
```rust
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

/// ข้อมูล Product แบบ v2 ที่มี field เพิ่มเติม (tags, image_url)
#[derive(Debug, Clone, Serialize, Deserialize, ToSchema)]
pub struct ProductV2 {
    pub id: uuid::Uuid,
    pub name: String,
    pub price: f64,
    /// Tags สำหรับค้นหา (เพิ่มใน v2)
    pub tags: Vec<String>,
    /// URL รูปภาพสินค้า (เพิ่มใน v2)
    pub image_url: Option<String>,
}

/// Version info response
#[derive(Debug, Serialize, Deserialize, ToSchema)]
pub struct VersionInfo {
    pub version: String,
    pub api_version: String,
    pub deprecated: bool,
    pub sunset_date: Option<String>,
}
```

**ใน `router.rs` เพิ่ม V2 routes:**
```rust
pub fn create_app_with_v2(store: ProductStore) -> Router {
    // V1 routes (ยังคงรองรับ backward compatibility)
    let v1_routes = Router::new()
        .route("/products", get(list_products).post(create_product))
        .route("/products/:id", get(get_product).put(update_product).delete(delete_product));

    // V2 routes — เพิ่ม fields ใหม่ แต่ delegate logic ไปที่ v1 handlers
    // (ในโปรเจคจริงจะมี v2 handlers ที่แยกต่างหาก)
    let v2_routes = Router::new()
        .route("/products", get(list_products_v2))
        .route("/products/:id", get(get_product_v2));

    Router::new()
        .nest("/v1", v1_routes)
        .nest("/v2", v2_routes)
        // ...
}
```

**Versioning Strategies เปรียบเทียบ:**

| Strategy | ตัวอย่าง | ข้อดี | ข้อเสีย |
|----------|---------|-------|---------|
| **URL path** (แนะนำ) | `/v1/products` | ชัดเจน, bookmarkable, proxy-friendly | URL ยาวขึ้น |
| **Query param** | `/products?version=1` | URL path เดิม | ไม่ชัดเจน, caching ยาก |
| **Header** | `API-Version: 1` | URL สะอาด | client ต้องตั้ง header ทุก request |
| **Content-Type** | `application/vnd.api+json;version=1` | มาตรฐาน MIME | ซับซ้อน, ใช้ยาก |

**Deprecation Header** — เมื่อ version เก่าจะถูกลบ ควรส่ง warning ใน response headers:
```rust
// ใน v1 handler ที่กำลัง deprecated
let mut headers = HeaderMap::new();
headers.insert("Deprecation", "true".parse().unwrap());
headers.insert("Sunset", "Sat, 31 Dec 2025 23:59:59 GMT".parse().unwrap());
headers.insert("Link", "</v2/products>; rel=\"successor-version\"".parse().unwrap());
```

---

### ขั้นที่ 9: Integration Tests ด้วย Tower ServiceExt

สร้าง `tests/integration_test.rs` — test HTTP handlers โดยตรงโดยไม่ต้องรัน server จริง

**`tests/integration_test.rs`:**
```rust
use axum::{body::Body, http::{Request, StatusCode}};
use http_body_util::BodyExt;
use rest_openapi::{create_app, handlers::initial_store};
use serde_json::{json, Value};
use tower::ServiceExt; // for .oneshot()

/// สร้าง test app ใหม่สำหรับแต่ละ test
fn make_app() -> axum::Router {
    let store = initial_store();
    create_app(store)
}

/// Helper: ส่ง request และรับ response body เป็น JSON
async fn request_json(app: axum::Router, req: Request<Body>) -> (StatusCode, Value) {
    let response = app.oneshot(req).await.unwrap();
    let status = response.status();
    let bytes = response.into_body().collect().await.unwrap().to_bytes();
    let body: Value = serde_json::from_slice(&bytes).unwrap_or(Value::Null);
    (status, body)
}

#[tokio::test]
async fn test_health_check_returns_200() {
    let app = make_app();
    let req = Request::builder().uri("/health").body(Body::empty()).unwrap();
    let (status, body) = request_json(app, req).await;
    assert_eq!(status, StatusCode::OK);
    assert_eq!(body["status"], "ok");
}

#[tokio::test]
async fn test_create_product_returns_201() {
    let app = make_app();
    let body = json!({
        "name": "Test Laptop",
        "description": "A powerful laptop for developers",
        "price": 45000.0,
        "stock": 10,
        "category": "electronics"
    });
    let req = Request::builder()
        .method("POST")
        .uri("/v1/products")
        .header("Content-Type", "application/json")
        .body(Body::from(body.to_string()))
        .unwrap();
    let (status, resp_body) = request_json(app, req).await;
    assert_eq!(status, StatusCode::CREATED);
    assert_eq!(resp_body["name"], "Test Laptop");
    assert!(resp_body["id"].is_string());
}

#[tokio::test]
async fn test_create_product_validation_error() {
    let app = make_app();
    let body = json!({
        "name": "X",   // too short (min 2)
        "price": -50.0, // negative price
        "stock": 5,
        "category": "books"
    });
    let req = Request::builder()
        .method("POST")
        .uri("/v1/products")
        .header("Content-Type", "application/json")
        .body(Body::from(body.to_string()))
        .unwrap();
    let (status, resp_body) = request_json(app, req).await;
    assert_eq!(status, StatusCode::UNPROCESSABLE_ENTITY);
    assert_eq!(resp_body["error"], "VALIDATION_ERROR");
    assert!(resp_body["details"].is_array());
}

#[tokio::test]
async fn test_openapi_json_accessible() {
    let app = make_app();
    let req = Request::builder()
        .uri("/api-docs/openapi.json")
        .body(Body::empty())
        .unwrap();
    let (status, body) = request_json(app, req).await;
    assert_eq!(status, StatusCode::OK);
    assert_eq!(body["info"]["title"], "Product API");
    assert!(body["paths"].is_object());
}
```

**ทำไม `tower::ServiceExt::oneshot()` ดีกว่า HTTP client:**

1. **ไม่ต้องรัน server** — ไม่ต้องหา port ว่าง ไม่มี race condition
2. **Fast** — ไม่มี network stack overhead
3. **Deterministic** — ไม่มี timing issue
4. **Type-safe** — compile-time check ว่า request/response types ถูกต้อง

`.oneshot(req)` ส่ง request ผ่าน Tower service pipeline ทั้งหมด รวมทั้ง middleware เหมือนกับ request จริง

---

## การทดสอบ (Testing)

### Unit Tests

Unit tests อยู่ในแต่ละ module โดยตรง:

**ใน `src/pagination.rs`:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_pagination_params_defaults() {
        let params = PaginationParams { page: None, per_page: None };
        assert_eq!(params.page(), 1);
        assert_eq!(params.per_page(), 20);
        assert_eq!(params.offset(), 0);
    }

    #[test]
    fn test_paginated_meta_calculation() {
        let data: Vec<u32> = (1..=10).collect();
        let result = Paginated::new(data, 2, 10, 35);
        assert_eq!(result.meta.total_pages, 4);
        assert!(result.meta.has_next);
        assert!(result.meta.has_prev);
    }
}
```

**ใน `src/router.rs`:**
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::handlers::initial_store;

    #[test]
    fn test_openapi_doc_generation() {
        let doc = ApiDoc::openapi();
        assert_eq!(doc.info.title, "Product API");
        // ตรวจสอบ paths
        let paths = &doc.paths.paths;
        assert!(paths.contains_key("/v1/products"));
        assert!(paths.contains_key("/health"));
    }

    #[test]
    fn test_openapi_has_components() {
        let doc = ApiDoc::openapi();
        let components = doc.components.as_ref().expect("Should have components");
        assert!(components.schemas.contains_key("Product"));
        assert!(components.schemas.contains_key("CreateProductRequest"));
    }
}
```

### รัน Tests

```bash
cargo test
```

### ผลลัพธ์จริงจากการรัน `cargo test`:

```
running 15 tests
test pagination::tests::test_paginated_empty ... ok
test pagination::tests::test_paginated_last_page ... ok
test pagination::tests::test_paginated_meta_calculation ... ok
test pagination::tests::test_pagination_params_clamp_per_page ... ok
test pagination::tests::test_pagination_params_defaults ... ok
test pagination::tests::test_pagination_params_page2 ... ok
test validation::tests::test_description_too_long_fails_validation ... ok
test validation::tests::test_empty_name_fails_validation ... ok
test router::tests::test_openapi_doc_generation ... ok
test validation::tests::test_negative_price_fails_validation ... ok
test validation::tests::test_valid_product_passes_validation ... ok
test router::tests::test_openapi_has_components ... ok
test versioning::tests::test_version_info_serialization ... ok
test versioning::tests::test_version_info_with_sunset ... ok
test router::tests::test_app_creation ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/server-0f93e1958d90a1d9)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-2d9638375c1454bc)

running 13 tests
test test_create_product_returns_201 ... ok
test test_create_then_delete_product ... ok
test test_create_product_validation_error_name_too_short ... ok
test test_create_product_validation_error_negative_price ... ok
test test_delete_nonexistent_product_returns_404 ... ok
test test_create_then_get_product ... ok
test test_get_nonexistent_product_returns_404 ... ok
test test_health_check_returns_200 ... ok
test test_list_products_with_pagination ... ok
test test_list_products_returns_paginated_response ... ok
test test_update_product_price ... ok
test test_redoc_ui_accessible ... ok
test test_openapi_json_accessible ... ok

test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

   Doc-tests rest_openapi

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**28 tests ผ่านทั้งหมด** — 15 unit tests + 13 integration tests

---

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### 1. Path ใน `#[utoipa::path]` ไม่ตรงกับ Route Registration

**ปัญหา:** utoipa จะ generate spec ตาม path ที่ระบุใน annotation แต่ Axum route ทำงานตาม path ที่ register จริง ถ้าต่างกัน spec จะ "โกหก"

```rust
// ❌ ผิด: annotation บอก /products แต่ router nest ไว้ที่ /v1
#[utoipa::path(get, path = "/products", ...)]
pub async fn list_products(...) {}

// ใน router:
.nest("/v1", Router::new().route("/products", get(list_products)))
// → request ไปที่ /v1/products แต่ spec บอก /products
```

```rust
// ✓ ถูก: annotation ระบุ full path รวม prefix
#[utoipa::path(get, path = "/v1/products", ...)]
pub async fn list_products(...) {}
```

**วิธี verify:** ลอง `GET /api-docs/openapi.json` แล้วดูว่า paths section ตรงกับ URL ที่ใช้จริง

---

### 2. Derive Order ของ Validation Macros

**ปัญหา:** `#[derive(Validate)]` ต้องมาหลัง `#[derive(Deserialize)]` และ field attributes ต้องอยู่บนฟิลด์ถูกต้อง

```rust
// ❌ ผิด: validate attribute บน type ที่ไม่รองรับ
#[derive(Serialize, Deserialize, Validate)]
pub struct Request {
    #[validate(range(min = 0.01))]
    pub name: String,  // String ไม่มี range constraint
}
```

```rust
// ✓ ถูก: ใช้ validator ที่เหมาะกับ type
#[derive(Serialize, Deserialize, Validate)]
pub struct Request {
    #[validate(length(min = 2, max = 200, message = "Name must be 2-200 characters"))]
    pub name: String,  // length เหมาะกับ String

    #[validate(range(min = 0.01, message = "Price must be greater than 0"))]
    pub price: f64,    // range เหมาะกับ numeric types
}
```

---

### 3. `ProductStore` ที่แชร์กันใน Integration Tests

**ปัญหา:** ถ้า tests ใช้ store เดิมกัน state ที่เปลี่ยนในหนึ่ง test จะ affect test อื่น

```rust
// ❌ ผิด: share app เดียวกันระหว่าง tests (ใน tokio::test concurrent)
static APP: Lazy<Router> = Lazy::new(|| create_app(initial_store()));

#[tokio::test]
async fn test_a() { /* create product */ }

#[tokio::test]
async fn test_b() { /* count products — อาจได้จำนวนผิดถ้า test_a รันก่อน */ }
```

```rust
// ✓ ถูก: สร้าง app ใหม่สำหรับแต่ละ test ด้วย fresh store
fn make_app() -> Router {
    create_app(initial_store())  // store ใหม่ทุกครั้ง
}

#[tokio::test]
async fn test_a() {
    let app = make_app();  // isolated store
    // ...
}
```

---

### 4. Forgetting `Servable` Trait Import

**ปัญหา:** `utoipa_redoc::Redoc::with_url()` จะ compile ไม่ได้ถ้าไม่ import `Servable` trait

```rust
// ❌ ผิด: ลืม import Servable
use utoipa_redoc::Redoc;

// Error: no method named `with_url` found for struct `Redoc`
Router::new().merge(Redoc::with_url("/redoc", ApiDoc::openapi()))
```

```rust
// ✓ ถูก: import Servable ด้วย
use utoipa_redoc::{Redoc, Servable};

Router::new().merge(Redoc::with_url("/redoc", ApiDoc::openapi()))
```

---

### 5. `#[derive(OpenApi)]` ต้องอ้างอิง handlers ด้วย full path

**ปัญหา:** เมื่อ `ApiDoc` อยู่ใน module ต่างจาก handlers, utoipa macro ต้องการ full path เพื่อหา `__path_*` structs ที่ generate

```rust
// ❌ ผิด: ถ้า handlers อยู่ใน module อื่น
#[derive(OpenApi)]
#[openapi(paths(list_products, create_product))]  // compile error
pub struct ApiDoc;
```

```rust
// ✓ ถูก: ใช้ full path
#[derive(OpenApi)]
#[openapi(paths(
    crate::handlers::list_products,
    crate::handlers::create_product,
))]
pub struct ApiDoc;
```

---

### 6. Response Type ใน `responses()` Annotation

**ปัญหา:** ลืมระบุ `body = Type` ทำให้ OpenAPI spec ไม่มี response schema

```rust
// ❌ ขาด body — spec จะ generate response ว่าง
#[utoipa::path(
    get, path = "/v1/products/{id}",
    responses(
        (status = 200, description = "Product found"),  // ไม่มี body schema
    )
)]
```

```rust
// ✓ ระบุ body type
#[utoipa::path(
    get, path = "/v1/products/{id}",
    responses(
        (status = 200, description = "Product found", body = Product),
        (status = 404, description = "Not found", body = ErrorResponse),
    )
)]
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/server
```

### Dockerfile

```dockerfile
FROM rust:1.82-alpine AS builder

RUN apk add --no-cache musl-dev

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
# Cache dependencies layer
RUN mkdir -p src && echo "fn main(){}" > src/main.rs
RUN cargo build --release 2>/dev/null || true

COPY src ./src
RUN touch src/main.rs && cargo build --release

# Runtime image
FROM alpine:latest
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/target/release/server /usr/local/bin/server
EXPOSE 3000
CMD ["server"]
```

### Docker Build และ Run

```bash
docker build -t product-api .
docker run -p 3000:3000 product-api
```

### Environment Variables สำหรับ Production

```bash
# กำหนด port และ log level
PORT=8080 RUST_LOG=info ./target/release/server

# ใช้ environment variable ใน main.rs
let port = std::env::var("PORT")
    .unwrap_or_else(|_| "3000".to_string())
    .parse::<u16>()
    .expect("PORT must be a number");
let addr = format!("0.0.0.0:{}", port);
let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
```

### Health Check สำหรับ Kubernetes

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 2
  periodSeconds: 5
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Authentication ด้วย Bearer Token

เพิ่ม JWT authentication โดยใช้ Tower middleware:

```rust
// ใน Cargo.toml เพิ่ม:
// jsonwebtoken = "9"

use axum::{
    extract::Request,
    http::{HeaderMap, StatusCode},
    middleware::Next,
    response::Response,
};

pub async fn auth_middleware(
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = headers
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // validate JWT token
    validate_jwt(token).map_err(|_| StatusCode::UNAUTHORIZED)?;

    Ok(next.run(request).await)
}

// ใน router เพิ่ม layer:
let protected_routes = Router::new()
    .route("/products", post(create_product))
    .route("/products/:id", put(update_product).delete(delete_product))
    .layer(axum::middleware::from_fn(auth_middleware));
```

เพิ่ม security scheme ใน OpenAPI spec:
```rust
#[derive(OpenApi)]
#[openapi(
    // ...
    modifiers(&SecurityAddon),
)]
pub struct ApiDoc;

struct SecurityAddon;
impl utoipa::Modify for SecurityAddon {
    fn modify(&self, openapi: &mut utoipa::openapi::OpenApi) {
        if let Some(components) = openapi.components.as_mut() {
            components.add_security_scheme(
                "bearer_auth",
                utoipa::openapi::security::SecurityScheme::Http(
                    utoipa::openapi::security::HttpBuilder::new()
                        .scheme(utoipa::openapi::security::HttpAuthScheme::Bearer)
                        .bearer_format("JWT")
                        .build(),
                ),
            );
        }
    }
}
```

---

### แบบฝึกหัดที่ 2: เพิ่ม Database Integration ด้วย SQLx

แทนที่ in-memory store ด้วย PostgreSQL:

```rust
// Cargo.toml เพิ่ม:
// sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "uuid", "chrono"] }

use sqlx::{PgPool, postgres::PgPoolOptions};

// AppState ใหม่
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
}

// Handler ใหม่
pub async fn create_product_db(
    State(state): State<AppState>,
    Json(req): Json<CreateProductRequest>,
) -> Result<impl IntoResponse, AppError> {
    validate_input(&req)?;
    
    let product = sqlx::query_as!(
        Product,
        r#"INSERT INTO products (id, name, description, price, stock, category, created_at, updated_at)
           VALUES ($1, $2, $3, $4, $5, $6, NOW(), NOW())
           RETURNING *"#,
        Uuid::new_v4(),
        req.name,
        req.description,
        req.price,
        req.stock as i32,
        req.category.to_string(),
    )
    .fetch_one(&state.db)
    .await
    .map_err(|e| AppError::Internal(e.to_string()))?;
    
    Ok((StatusCode::CREATED, Json(product)))
}
```

---

### แบบฝึกหัดที่ 3: เพิ่ม Rate Limiting Middleware

ป้องกัน API abuse ด้วย sliding window rate limiter:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::Instant;
use axum::extract::ConnectInfo;
use std::net::SocketAddr;

#[derive(Clone)]
pub struct RateLimiter {
    requests: Arc<Mutex<HashMap<String, Vec<Instant>>>>,
    max_requests: usize,
    window_secs: u64,
}

impl RateLimiter {
    pub fn new(max_requests: usize, window_secs: u64) -> Self {
        RateLimiter {
            requests: Arc::new(Mutex::new(HashMap::new())),
            max_requests,
            window_secs,
        }
    }

    pub fn check_rate(&self, client_ip: &str) -> bool {
        let mut requests = self.requests.lock().unwrap();
        let now = Instant::now();
        let window = std::time::Duration::from_secs(self.window_secs);

        let timestamps = requests.entry(client_ip.to_string()).or_default();
        // ลบ timestamps ที่เก่าเกิน window
        timestamps.retain(|&t| now.duration_since(t) < window);

        if timestamps.len() < self.max_requests {
            timestamps.push(now);
            true
        } else {
            false
        }
    }
}

// ใน router เพิ่ม layer:
let limiter = RateLimiter::new(100, 60); // 100 requests per minute
Router::new()
    .route("/v1/products", post(create_product))
    .layer(axum::middleware::from_fn_with_state(
        limiter,
        rate_limit_middleware,
    ))
```

---

### แบบฝึกหัดที่ 4: เพิ่ม HATEOAS Links

HATEOAS (Hypermedia as the Engine of Application State) เป็น REST constraint ที่ advanced — response บอก client ว่า action ถัดไปที่ทำได้มีอะไรบ้าง:

```rust
#[derive(Debug, Serialize, ToSchema)]
pub struct ProductWithLinks {
    #[serde(flatten)]
    pub product: Product,
    /// Links ที่เกี่ยวข้องกับ resource นี้
    pub _links: ResourceLinks,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct ResourceLinks {
    pub self_: Link,
    pub update: Option<Link>,
    pub delete: Option<Link>,
    pub collection: Link,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct Link {
    pub href: String,
    pub method: String,
}

impl ProductWithLinks {
    pub fn new(product: Product, base_url: &str) -> Self {
        let id = product.id;
        ProductWithLinks {
            product,
            _links: ResourceLinks {
                self_: Link {
                    href: format!("{}/v1/products/{}", base_url, id),
                    method: "GET".to_string(),
                },
                update: Some(Link {
                    href: format!("{}/v1/products/{}", base_url, id),
                    method: "PUT".to_string(),
                }),
                delete: Some(Link {
                    href: format!("{}/v1/products/{}", base_url, id),
                    method: "DELETE".to_string(),
                }),
                collection: Link {
                    href: format!("{}/v1/products", base_url),
                    method: "GET".to_string(),
                },
            },
        }
    }
}
```

---

### แบบฝึกหัดที่ 5: Webhook Notifications

เพิ่ม webhook system ให้ client subscribe และรับแจ้งเตือนเมื่อสินค้าเปลี่ยนแปลง:

```rust
use tokio::sync::broadcast;

#[derive(Clone, Debug, Serialize)]
pub struct ProductEvent {
    pub event_type: String,  // "product.created", "product.updated", "product.deleted"
    pub product_id: Uuid,
    pub timestamp: DateTime<Utc>,
}

// ใน AppState เพิ่ม:
pub struct AppState {
    pub store: ProductStore,
    pub event_sender: broadcast::Sender<ProductEvent>,
}

// ใน create_product handler:
let event = ProductEvent {
    event_type: "product.created".to_string(),
    product_id: product.id,
    timestamp: Utc::now(),
};
let _ = state.event_sender.send(event); // ส่ง event ให้ทุก subscriber

// Webhook delivery task:
tokio::spawn(async move {
    let mut receiver = event_sender.subscribe();
    while let Ok(event) = receiver.recv().await {
        for url in &webhook_urls {
            if let Err(e) = deliver_webhook(url, &event).await {
                eprintln!("Webhook delivery failed: {}", e);
            }
        }
    }
});
```

---

## สรุป

โปรเจคนี้ครอบคลุม pattern สำคัญของการสร้าง production-ready REST API ด้วย Rust:

**สิ่งที่สร้างสำเร็จ:**
1. **REST API** ครบวงจรด้วย Axum — CRUD operations, HTTP status codes ที่ถูกต้อง, type-safe extractors
2. **OpenAPI 3.0 spec** ที่ generate อัตโนมัติจาก code annotations — ไม่ต้อง maintain YAML แยก
3. **Request validation** ด้วย `validator` crate — field-level rules, structured error responses
4. **Pagination** แบบ generic — ใช้ได้กับทุก resource type
5. **API versioning** ด้วย URL prefix — backward compatible, deprecation path ชัดเจน
6. **Integration tests** ด้วย Tower ServiceExt — test HTTP layer โดยไม่ต้องรัน server

**Patterns สำคัญที่ได้เรียน:**

| Pattern | Implementation | ใช้ที่ไหน |
|---------|---------------|----------|
| Code-first OpenAPI | `#[utoipa::path]` + `#[derive(ToSchema)]` | ทุก handler + model |
| Type-safe extractors | `Path<Uuid>`, `Query<T>`, `Json<T>` | handler parameters |
| Error type hierarchy | `AppError: IntoResponse` | error.rs |
| Generic pagination | `Paginated<T>` wrapper | list endpoints |
| Validation middleware | `validate_input<T: Validate>()` | POST/PUT handlers |
| In-process testing | `.oneshot()` ของ Tower service | integration_test.rs |

**การเชื่อมโยงกับโปรเจคถัดไป:**

โปรเจคถัดไป (J08: SSE Server) จะเพิ่ม **real-time streaming** ให้กับ API นี้ — client subscribe รับ events เมื่อสินค้าถูกสร้าง อัปเดต หรือลบ โดยใช้ Server-Sent Events (SSE) ซึ่งเป็น pattern ที่ใช้ใน dashboards, live feeds และ notification systems

---

**โปรเจคก่อนหน้า:** [Project J06: GraphQL API](project-j06-graphql-api.md) | **โปรเจคถัดไป:** [Project J08: SSE Server](project-j08-sse-server.md)
