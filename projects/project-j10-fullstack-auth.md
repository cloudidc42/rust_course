# Project J10: Full-Stack Application with Authentication

> โมดูล: J — Full-Stack/WASM | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคสุดท้ายของหลักสูตรนี้คือการสร้าง **Full-Stack Web Application พร้อมระบบ Authentication ที่สมบูรณ์** ตั้งแต่ backend ที่เขียนด้วย Axum ไปจนถึง frontend ที่เขียนด้วย Yew ซึ่งทั้งสองส่วนสื่อสารกันผ่าน REST API ที่ป้องกันด้วย JWT (JSON Web Tokens)

ระบบ authentication คือหัวใจของ web application แทบทุกตัวในโลก production ไม่ว่าจะเป็น e-commerce, SaaS platform, หรือ internal tool ทุกระบบต้องตอบคำถามพื้นฐานสองข้อ: "คุณคือใคร?" (Authentication) และ "คุณมีสิทธิ์ทำสิ่งนี้ไหม?" (Authorization) โปรเจคนี้สอนวิธีตอบคำถามทั้งสองด้วย Rust อย่างถูกต้องและปลอดภัย

**สิ่งที่เราจะสร้าง:**
- Backend API ด้วย Axum ที่มี endpoint `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`
- Middleware สำหรับตรวจสอบ JWT token ทุก request ที่ protected
- ระบบ password hashing ด้วย Argon2 ซึ่งเป็น winner ของ Password Hashing Competition
- Refresh token rotation ที่ป้องกัน token replay attack
- Role-based access control (RBAC) ผ่าน `AuthUser` extractor
- Frontend ด้วย Yew ที่มี `AuthProvider`, `ProtectedRoute`, และ `use_context` สำหรับ auth state
- Test suite ครอบคลุม JWT generation/validation, password hashing, และ middleware behavior

**Use cases จริงในโลก production:**
- ระบบ login สำหรับ SaaS application ที่มี user roles เช่น admin, editor, viewer
- API gateway ที่ต้องตรวจสอบ token ก่อนส่ง request ไป microservices
- Multi-tenant platform ที่แต่ละ tenant มีสิทธิ์แตกต่างกัน
- Mobile/Web app ที่ต้องการ seamless token refresh โดยไม่บังคับ user login ใหม่

## สิ่งที่จะได้เรียนรู้

- **JWT internals** — โครงสร้าง header.payload.signature, HS256 vs RS256, การ encode/decode claims ด้วย `jsonwebtoken` crate
- **Argon2 password hashing** — ทำไม Argon2 ถึงดีกว่า bcrypt/scrypt, การ tune parameters, salt generation ที่ปลอดภัย
- **Axum extractors** — implement `FromRequestParts` เพื่อสร้าง `AuthUser` extractor ที่ compile-time type-safe
- **Tower middleware** — สร้าง auth middleware ด้วย `tower::Service` และ `tower-http` layers
- **Refresh token rotation** — pattern สำหรับ rotate tokens อย่างปลอดภัย ป้องกัน reuse attack
- **Role-based access control** — `require_role!` macro และ handler-level guards
- **Yew frontend auth** — `use_context`, `AuthProvider` component, `ProtectedRoute` pattern
- **Integration testing** — test auth flow ตั้งแต่ register ถึง refresh โดยไม่ต้องรัน real server

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30**: Rust basics — ownership, borrowing, structs, enums, iterators, closures
- **Part 31–50**: Error handling (`Result`, `?`), traits, generics, async/await พื้นฐาน
- **Part 51–70**: Tokio runtime, async functions, `impl Trait`, trait objects
- **Part 71–90**: Axum routing, handlers, extractors, middleware, `tower` ecosystem
- **Part 91–95**: Serde serialization, JSON API design
- **Part 96–105**: WASM และ Yew basics (จาก Project J01–J09)
- **Part 106–110**: Security concepts, cryptography fundamentals

## โครงสร้างโปรเจค (Project Layout)

```
fullstack-auth/
├── backend/
│   ├── src/
│   │   ├── main.rs              ← server startup, router setup
│   │   ├── lib.rs               ← re-export modules
│   │   ├── auth/
│   │   │   ├── mod.rs           ← auth module exports
│   │   │   ├── jwt.rs           ← JWT encode/decode, Claims struct
│   │   │   ├── password.rs      ← Argon2 hash/verify
│   │   │   ├── middleware.rs    ← AuthUser extractor, FromRequestParts
│   │   │   └── guards.rs        ← require_role! macro, role checks
│   │   ├── handlers/
│   │   │   ├── mod.rs
│   │   │   ├── register.rs      ← POST /auth/register
│   │   │   ├── login.rs         ← POST /auth/login
│   │   │   ├── refresh.rs       ← POST /auth/refresh
│   │   │   └── protected.rs     ← protected endpoint examples
│   │   ├── models/
│   │   │   ├── mod.rs
│   │   │   ├── user.rs          ← User struct, UserRole enum
│   │   │   └── token.rs         ← TokenPair, RefreshClaims
│   │   └── store/
│   │       ├── mod.rs
│   │       └── memory.rs        ← in-memory user store (DashMap)
│   ├── tests/
│   │   ├── auth_flow.rs         ← integration tests
│   │   └── jwt_tests.rs         ← JWT unit tests
│   └── Cargo.toml
├── frontend/
│   ├── src/
│   │   ├── main.rs              ← Yew app entry point
│   │   ├── context/
│   │   │   └── auth.rs          ← AuthContext, AuthProvider
│   │   ├── components/
│   │   │   ├── protected_route.rs
│   │   │   ├── login_form.rs
│   │   │   └── dashboard.rs
│   │   └── api/
│   │       └── client.rs        ← fetch wrappers สำหรับ auth endpoints
│   ├── index.html
│   └── Cargo.toml
└── Cargo.toml                   ← workspace
```

## การออกแบบ (Architecture & Design)

### Authentication Flow

```
Browser                     Axum Backend                   Memory Store
   │                              │                              │
   │  POST /auth/register         │                              │
   │  {username, password}        │                              │
   │─────────────────────────────►│                              │
   │                              │  hash_password(pw)           │
   │                              │  store.insert(user)─────────►│
   │                              │  create_token(user_id, ...)  │
   │◄─────────────────────────────│                              │
   │  {access_token, refresh_token}                              │
   │                              │                              │
   │  POST /auth/login            │                              │
   │  {username, password}        │                              │
   │─────────────────────────────►│                              │
   │                              │  find_user(username)────────►│
   │                              │◄─────────────────────────────│
   │                              │  verify_password(pw, hash)   │
   │                              │  create_access_token (15m)   │
   │                              │  create_refresh_token (7d)   │
   │                              │  store_refresh(jti, user)───►│
   │◄─────────────────────────────│                              │
   │  {access_token, refresh_token}                              │
   │                              │                              │
   │  GET /api/profile            │                              │
   │  Authorization: Bearer <AT>  │                              │
   │─────────────────────────────►│                              │
   │                              │  verify_token(AT)            │
   │                              │  extract AuthUser from claims│
   │◄─────────────────────────────│                              │
   │  {user data}                 │                              │
   │                              │                              │
   │  POST /auth/refresh          │                              │
   │  {refresh_token}             │                              │
   │─────────────────────────────►│                              │
   │                              │  verify_refresh_token(RT)────►│
   │                              │  revoke_old_token(jti)───────►│
   │                              │  issue_new_pair()             │
   │◄─────────────────────────────│                              │
   │  {new_access_token, new_refresh_token}                      │
```

### การออกแบบ JWT Claims

JWT token ประกอบด้วยสามส่วนคั่นด้วย `.`:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← Header (Base64URL encoded)
.
eyJzdWIiOiJ1c2VyLTEyMyIsImV4cCI6MTcwMDAwMDkwMCwiaWF0IjoxNzAwMDAwMDAwLCJyb2xlcyI6WyJ1c2VyIl19
.                                        ← Payload (Base64URL encoded)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c   ← Signature (HMAC-SHA256)
```

**Access Token claims:**
```json
{
  "sub": "user-123",
  "exp": 1700000900,
  "iat": 1700000000,
  "roles": ["user", "editor"],
  "type": "access"
}
```

**Refresh Token claims:**
```json
{
  "sub": "user-123",
  "exp": 1700604800,
  "iat": 1700000000,
  "jti": "550e8400-e29b-41d4-a716-446655440000",
  "type": "refresh"
}
```

`jti` (JWT ID) คือ unique identifier สำหรับแต่ละ refresh token ใช้สำหรับ revocation

### ทำไมต้องใช้ Argon2

Argon2 ชนะ Password Hashing Competition (PHC) ปี 2015 ด้วยเหตุผล:
- **Memory-hard**: ต้องใช้ RAM จำนวนมาก ทำให้ GPU/ASIC attack ทำได้ยาก
- **Configurable**: ปรับ memory, iterations, parallelism ได้ตาม hardware
- **Side-channel resistant**: Argon2i ออกแบบมาเพื่อ defend side-channel attacks

เปรียบเทียบ:
| Algorithm | Memory-hard | Side-channel | PHC Winner |
|-----------|-------------|--------------|------------|
| MD5/SHA   | ❌          | ❌           | ❌         |
| bcrypt    | ❌          | ❌           | ❌         |
| scrypt    | ✅          | ❌           | ❌         |
| Argon2    | ✅          | ✅           | ✅         |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้าง JWT — Claims, Encode, Decode

เริ่มต้นจากหัวใจของระบบคือ JWT token เราต้องเข้าใจว่า token คืออะไรและทำงานอย่างไรก่อนที่จะเพิ่ม middleware

**`backend/Cargo.toml`:**

```toml
[package]
name = "fullstack-auth-backend"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.7", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jsonwebtoken = "9"
argon2 = "0.5"
rand_core = { version = "0.6", features = ["getrandom"] }
uuid = { version = "1", features = ["v4"] }
dashmap = "5"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = "0.3"
thiserror = "1"
anyhow = "1"

[dev-dependencies]
axum-test = "14"
tokio-test = "0.4"
```

**`backend/src/auth/jwt.rs`:**

```rust
use jsonwebtoken::{
    decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation,
};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};
use uuid::Uuid;

/// Claims สำหรับ access token (อายุสั้น ~15 นาที)
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct AccessClaims {
    /// Subject: user ID
    pub sub: String,
    /// Expiration time (Unix timestamp)
    pub exp: usize,
    /// Issued at (Unix timestamp)
    pub iat: usize,
    /// Roles ที่ user มี
    pub roles: Vec<String>,
    /// Token type
    pub token_type: String,
}

/// Claims สำหรับ refresh token (อายุยาว ~7 วัน)
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct RefreshClaims {
    pub sub: String,
    pub exp: usize,
    pub iat: usize,
    /// JWT ID — unique ID สำหรับ revocation
    pub jti: String,
    pub token_type: String,
}

/// Error type สำหรับ JWT operations
#[derive(Debug, thiserror::Error)]
pub enum JwtError {
    #[error("Token creation failed: {0}")]
    EncodingError(#[from] jsonwebtoken::errors::Error),
    #[error("Token expired")]
    Expired,
    #[error("Invalid token")]
    Invalid,
    #[error("Wrong token type: expected {expected}, got {got}")]
    WrongType { expected: String, got: String },
}

fn now_unix() -> usize {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .expect("Time went backwards")
        .as_secs() as usize
}

/// สร้าง access token อายุ `exp_secs` วินาที
pub fn create_access_token(
    user_id: &str,
    roles: Vec<String>,
    secret: &str,
    exp_secs: usize,
) -> Result<String, JwtError> {
    let now = now_unix();
    let claims = AccessClaims {
        sub: user_id.to_string(),
        exp: now + exp_secs,
        iat: now,
        roles,
        token_type: "access".to_string(),
    };
    Ok(encode(
        &Header::default(), // HS256
        &claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )?)
}

/// สร้าง refresh token อายุ `exp_secs` วินาที
pub fn create_refresh_token(
    user_id: &str,
    secret: &str,
    exp_secs: usize,
) -> Result<(String, String), JwtError> {
    let now = now_unix();
    let jti = Uuid::new_v4().to_string();
    let claims = RefreshClaims {
        sub: user_id.to_string(),
        exp: now + exp_secs,
        iat: now,
        jti: jti.clone(),
        token_type: "refresh".to_string(),
    };
    let token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )?;
    Ok((token, jti))
}

/// ตรวจสอบและ decode access token
pub fn verify_access_token(
    token: &str,
    secret: &str,
) -> Result<AccessClaims, JwtError> {
    let validation = Validation::new(Algorithm::HS256);
    let data = decode::<AccessClaims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &validation,
    )
    .map_err(|e| match e.kind() {
        jsonwebtoken::errors::ErrorKind::ExpiredSignature => JwtError::Expired,
        _ => JwtError::Invalid,
    })?;

    if data.claims.token_type != "access" {
        return Err(JwtError::WrongType {
            expected: "access".into(),
            got: data.claims.token_type,
        });
    }

    Ok(data.claims)
}

/// ตรวจสอบและ decode refresh token
pub fn verify_refresh_token(
    token: &str,
    secret: &str,
) -> Result<RefreshClaims, JwtError> {
    let validation = Validation::new(Algorithm::HS256);
    let data = decode::<RefreshClaims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &validation,
    )
    .map_err(|e| match e.kind() {
        jsonwebtoken::errors::ErrorKind::ExpiredSignature => JwtError::Expired,
        _ => JwtError::Invalid,
    })?;

    if data.claims.token_type != "refresh" {
        return Err(JwtError::WrongType {
            expected: "refresh".into(),
            got: data.claims.token_type,
        });
    }

    Ok(data.claims)
}
```

**หมายเหตุสำคัญ**: เราใช้ `Header::default()` ซึ่งหมายถึง HS256 (HMAC-SHA256) สำหรับ production ที่มีหลาย service ควรพิจารณาใช้ RS256 (RSA) แทน เพื่อให้แต่ละ service สามารถ verify token ได้โดยใช้ public key เท่านั้น โดยไม่ต้องแชร์ secret

---

### ขั้นที่ 2: Password Hashing ด้วย Argon2

Argon2 เป็น algorithm ที่ออกแบบมาให้ช้าโดยเจตนา ยิ่งช้า = ยิ่งยากสำหรับ attacker ที่จะ brute-force

**`backend/src/auth/password.rs`:**

```rust
use argon2::{
    password_hash::{PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};
use rand_core::OsRng;

/// Error type สำหรับ password operations
#[derive(Debug, thiserror::Error)]
pub enum PasswordError {
    #[error("Failed to hash password")]
    HashError,
    #[error("Invalid hash format")]
    InvalidHash,
}

/// Hash password ด้วย Argon2id (default)
///
/// ใช้ random salt ทุกครั้ง ดังนั้น password เดิมจะได้ hash ต่างกัน
pub fn hash_password(password: &str) -> Result<String, PasswordError> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();

    argon2
        .hash_password(password.as_bytes(), &salt)
        .map(|hash| hash.to_string())
        .map_err(|_| PasswordError::HashError)
}

/// ตรวจสอบว่า password ตรงกับ hash หรือไม่
///
/// ปลอดภัยจาก timing attack เพราะ argon2 ใช้ constant-time comparison
pub fn verify_password(password: &str, hash: &str) -> Result<bool, PasswordError> {
    let parsed_hash = PasswordHash::new(hash).map_err(|_| PasswordError::InvalidHash)?;

    Ok(Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok())
}

/// ตรวจสอบความแข็งแกร่งของ password (เบื้องต้น)
pub fn validate_password_strength(password: &str) -> Result<(), &'static str> {
    if password.len() < 8 {
        return Err("Password must be at least 8 characters");
    }
    if !password.chars().any(|c| c.is_uppercase()) {
        return Err("Password must contain at least one uppercase letter");
    }
    if !password.chars().any(|c| c.is_ascii_digit()) {
        return Err("Password must contain at least one digit");
    }
    Ok(())
}
```

**ทำไม salt ถึงสำคัญ?** — ถ้าไม่มี salt, attackers สามารถสร้าง Rainbow Table ล่วงหน้าได้ (precompute hash สำหรับ password ทั่วไปทุกตัว) แล้ว lookup ทันที แต่ถ้าทุก user มี salt ต่างกัน attacker ต้อง compute ใหม่สำหรับทุก hash

---

### ขั้นที่ 3: User Model และ In-Memory Store

สำหรับโปรเจคนี้เราใช้ `DashMap` เป็น in-memory store (thread-safe `HashMap`) ใน production จะเปลี่ยนเป็น PostgreSQL หรือ database อื่น

**`backend/src/models/user.rs`:**

```rust
use serde::{Deserialize, Serialize};
use std::fmt;
use uuid::Uuid;

/// Roles ที่มีในระบบ
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum UserRole {
    Admin,
    Editor,
    User,
    Viewer,
}

impl fmt::Display for UserRole {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            UserRole::Admin => write!(f, "admin"),
            UserRole::Editor => write!(f, "editor"),
            UserRole::User => write!(f, "user"),
            UserRole::Viewer => write!(f, "viewer"),
        }
    }
}

impl UserRole {
    pub fn as_str(&self) -> &'static str {
        match self {
            UserRole::Admin => "admin",
            UserRole::Editor => "editor",
            UserRole::User => "user",
            UserRole::Viewer => "viewer",
        }
    }
}

/// User entity ที่เก็บใน store
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: String,
    pub username: String,
    pub email: String,
    /// Argon2 hash ของ password (ไม่เก็บ plaintext เด็ดขาด!)
    pub password_hash: String,
    pub roles: Vec<UserRole>,
    pub created_at: u64,
}

impl User {
    pub fn new(username: String, email: String, password_hash: String) -> Self {
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs();

        User {
            id: Uuid::new_v4().to_string(),
            username,
            email,
            password_hash,
            roles: vec![UserRole::User], // default role
            created_at: now,
        }
    }

    /// คืน roles เป็น Vec<String> สำหรับ JWT claims
    pub fn role_strings(&self) -> Vec<String> {
        self.roles.iter().map(|r| r.as_str().to_string()).collect()
    }
}

/// Request body สำหรับ registration
#[derive(Debug, Deserialize)]
pub struct RegisterRequest {
    pub username: String,
    pub email: String,
    pub password: String,
}

/// Request body สำหรับ login
#[derive(Debug, Deserialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

/// Response หลัง auth สำเร็จ
#[derive(Debug, Serialize)]
pub struct AuthResponse {
    pub access_token: String,
    pub refresh_token: String,
    pub token_type: String,
    pub expires_in: u64,
}
```

**`backend/src/store/memory.rs`:**

```rust
use crate::models::user::User;
use dashmap::DashMap;
use std::sync::Arc;

/// In-memory store สำหรับ users และ refresh tokens
///
/// ใช้ DashMap เพราะรองรับ concurrent access โดยไม่ต้องใช้ Mutex
/// DashMap ใช้ sharding ภายในเพื่อลด lock contention
#[derive(Clone)]
pub struct AppState {
    /// users: username → User
    pub users: Arc<DashMap<String, User>>,
    /// valid_refresh_tokens: jti → user_id
    /// เมื่อ token ถูก revoke ให้ลบออกจาก map นี้
    pub valid_refresh_tokens: Arc<DashMap<String, String>>,
    pub jwt_secret: String,
}

impl AppState {
    pub fn new(jwt_secret: String) -> Self {
        AppState {
            users: Arc::new(DashMap::new()),
            valid_refresh_tokens: Arc::new(DashMap::new()),
            jwt_secret,
        }
    }

    /// เพิ่ม user ใหม่ — คืน Err ถ้า username ซ้ำ
    pub fn add_user(&self, user: User) -> Result<(), String> {
        if self.users.contains_key(&user.username) {
            return Err(format!("Username '{}' already taken", user.username));
        }
        self.users.insert(user.username.clone(), user);
        Ok(())
    }

    /// หา user ตาม username
    pub fn find_user(&self, username: &str) -> Option<User> {
        self.users.get(username).map(|u| u.clone())
    }

    /// เก็บ refresh token jti → user_id
    pub fn store_refresh_token(&self, jti: &str, user_id: &str) {
        self.valid_refresh_tokens
            .insert(jti.to_string(), user_id.to_string());
    }

    /// ตรวจสอบว่า refresh token ยัง valid อยู่ไหม
    pub fn is_refresh_token_valid(&self, jti: &str) -> bool {
        self.valid_refresh_tokens.contains_key(jti)
    }

    /// Revoke refresh token (ลบออกจาก store)
    pub fn revoke_refresh_token(&self, jti: &str) {
        self.valid_refresh_tokens.remove(jti);
    }
}
```

---

### ขั้นที่ 4: Register และ Login Endpoints

**`backend/src/handlers/register.rs`:**

```rust
use axum::{extract::State, http::StatusCode, Json};
use crate::{
    auth::{
        jwt::create_access_token,
        jwt::create_refresh_token,
        password::{hash_password, validate_password_strength},
    },
    models::user::{AuthResponse, RegisterRequest, User},
    store::memory::AppState,
};

/// POST /auth/register
///
/// Flow:
/// 1. Validate input (username ไม่ว่าง, password แข็งพอ)
/// 2. ตรวจสอบ username ซ้ำ
/// 3. Hash password ด้วย Argon2
/// 4. บันทึก user ใน store
/// 5. ออก JWT pair (access + refresh)
pub async fn register(
    State(state): State<AppState>,
    Json(req): Json<RegisterRequest>,
) -> Result<(StatusCode, Json<AuthResponse>), (StatusCode, Json<serde_json::Value>)> {
    // 1. Validate input
    if req.username.trim().is_empty() {
        return Err(error_response(StatusCode::BAD_REQUEST, "Username is required"));
    }
    if req.username.len() < 3 || req.username.len() > 32 {
        return Err(error_response(
            StatusCode::BAD_REQUEST,
            "Username must be 3-32 characters",
        ));
    }

    // 2. ตรวจสอบ password strength
    if let Err(msg) = validate_password_strength(&req.password) {
        return Err(error_response(StatusCode::BAD_REQUEST, msg));
    }

    // 3. Hash password
    let password_hash = hash_password(&req.password).map_err(|_| {
        error_response(StatusCode::INTERNAL_SERVER_ERROR, "Password processing failed")
    })?;

    // 4. สร้าง user และบันทึก
    let user = User::new(req.username.clone(), req.email.clone(), password_hash);
    let user_id = user.id.clone();
    let roles = user.role_strings();

    state.add_user(user).map_err(|msg| {
        error_response(StatusCode::CONFLICT, &msg)
    })?;

    // 5. ออก tokens
    issue_token_pair(&state, &user_id, roles)
}

/// POST /auth/login
pub async fn login(
    State(state): State<AppState>,
    Json(req): Json<crate::models::user::LoginRequest>,
) -> Result<(StatusCode, Json<AuthResponse>), (StatusCode, Json<serde_json::Value>)> {
    // หา user
    let user = state
        .find_user(&req.username)
        .ok_or_else(|| error_response(StatusCode::UNAUTHORIZED, "Invalid credentials"))?;

    // ตรวจสอบ password
    let ok = crate::auth::password::verify_password(&req.password, &user.password_hash)
        .map_err(|_| error_response(StatusCode::INTERNAL_SERVER_ERROR, "Verification failed"))?;

    if !ok {
        // หมายเหตุ: ใช้ error message เดียวกันกับ "user not found"
        // เพื่อป้องกัน username enumeration attack
        return Err(error_response(StatusCode::UNAUTHORIZED, "Invalid credentials"));
    }

    let user_id = user.id.clone();
    let roles = user.role_strings();
    issue_token_pair(&state, &user_id, roles)
}

fn issue_token_pair(
    state: &AppState,
    user_id: &str,
    roles: Vec<String>,
) -> Result<(StatusCode, Json<AuthResponse>), (StatusCode, Json<serde_json::Value>)> {
    const ACCESS_TTL: usize = 15 * 60; // 15 นาที
    const REFRESH_TTL: usize = 7 * 24 * 60 * 60; // 7 วัน

    let access_token = create_access_token(user_id, roles, &state.jwt_secret, ACCESS_TTL)
        .map_err(|_| error_response(StatusCode::INTERNAL_SERVER_ERROR, "Token creation failed"))?;

    let (refresh_token, jti) =
        create_refresh_token(user_id, &state.jwt_secret, REFRESH_TTL)
            .map_err(|_| error_response(StatusCode::INTERNAL_SERVER_ERROR, "Token creation failed"))?;

    // เก็บ jti ไว้สำหรับ validation ในภายหลัง
    state.store_refresh_token(&jti, user_id);

    Ok((
        StatusCode::OK,
        Json(AuthResponse {
            access_token,
            refresh_token,
            token_type: "Bearer".to_string(),
            expires_in: ACCESS_TTL as u64,
        }),
    ))
}

fn error_response(
    status: StatusCode,
    message: &str,
) -> (StatusCode, Json<serde_json::Value>) {
    (status, Json(serde_json::json!({ "error": message })))
}
```

---

### ขั้นที่ 5: AuthUser Extractor และ Protected Routes

หัวใจของ Axum auth คือ custom extractor ที่ implement `FromRequestParts` ซึ่งทำให้ handler functions รับ `AuthUser` parameter ได้โดยตรง

**`backend/src/auth/middleware.rs`:**

```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    Json,
};
use serde_json::Value;
use crate::{
    auth::jwt::{verify_access_token, AccessClaims},
    store::memory::AppState,
};

/// Authenticated user ที่ extracted จาก JWT token
///
/// ใช้เป็น parameter ใน handler เพื่อ require authentication:
/// ```rust
/// async fn my_handler(auth: AuthUser) -> impl IntoResponse {
///     format!("Hello, {}!", auth.user_id)
/// }
/// ```
#[derive(Debug, Clone)]
pub struct AuthUser {
    pub user_id: String,
    pub roles: Vec<String>,
}

impl AuthUser {
    pub fn has_role(&self, role: &str) -> bool {
        self.roles.iter().any(|r| r == role)
    }

    pub fn is_admin(&self) -> bool {
        self.has_role("admin")
    }
}

/// Implement FromRequestParts เพื่อให้ Axum สามารถ extract AuthUser
/// จาก request headers โดยอัตโนมัติ
#[async_trait]
impl FromRequestParts<AppState> for AuthUser {
    type Rejection = (StatusCode, Json<Value>);

    async fn from_request_parts(
        parts: &mut Parts,
        state: &AppState,
    ) -> Result<Self, Self::Rejection> {
        // ดึง Authorization header
        let auth_header = parts
            .headers
            .get("Authorization")
            .and_then(|v| v.to_str().ok())
            .ok_or_else(|| {
                (
                    StatusCode::UNAUTHORIZED,
                    Json(serde_json::json!({ "error": "Missing Authorization header" })),
                )
            })?;

        // ตรวจสอบ format "Bearer <token>"
        let token = auth_header.strip_prefix("Bearer ").ok_or_else(|| {
            (
                StatusCode::UNAUTHORIZED,
                Json(serde_json::json!({ "error": "Invalid Authorization format, use Bearer <token>" })),
            )
        })?;

        // ตรวจสอบ token
        let claims: AccessClaims = verify_access_token(token, &state.jwt_secret)
            .map_err(|e| {
                let msg = match e {
                    crate::auth::jwt::JwtError::Expired => "Token expired",
                    _ => "Invalid token",
                };
                (
                    StatusCode::UNAUTHORIZED,
                    Json(serde_json::json!({ "error": msg })),
                )
            })?;

        Ok(AuthUser {
            user_id: claims.sub,
            roles: claims.roles,
        })
    }
}

/// Optional auth extractor — ไม่ต้อง auth แต่ถ้ามี token ก็จะ extract
pub struct OptionalAuthUser(pub Option<AuthUser>);

#[async_trait]
impl FromRequestParts<AppState> for OptionalAuthUser {
    type Rejection = std::convert::Infallible;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &AppState,
    ) -> Result<Self, Self::Rejection> {
        let result = AuthUser::from_request_parts(parts, state).await;
        Ok(OptionalAuthUser(result.ok()))
    }
}
```

**`backend/src/auth/guards.rs`:**

```rust
/// Macro สำหรับ role-based access control
///
/// ใช้งาน:
/// ```rust
/// async fn admin_only(auth: AuthUser) -> impl IntoResponse {
///     require_role!(auth, "admin");
///     "Admin area"
/// }
/// ```
#[macro_export]
macro_rules! require_role {
    ($auth:expr, $role:expr) => {
        if !$auth.has_role($role) {
            return (
                axum::http::StatusCode::FORBIDDEN,
                axum::Json(serde_json::json!({
                    "error": format!("Role '{}' required", $role)
                })),
            )
                .into_response();
        }
    };
}

/// Function-based role check สำหรับกรณีที่ macro ไม่เหมาะ
pub fn check_role(
    auth: &crate::auth::middleware::AuthUser,
    required_role: &str,
) -> Result<(), (axum::http::StatusCode, axum::Json<serde_json::Value>)> {
    if auth.has_role(required_role) {
        Ok(())
    } else {
        Err((
            axum::http::StatusCode::FORBIDDEN,
            axum::Json(serde_json::json!({
                "error": format!("Role '{}' required", required_role),
                "user_roles": auth.roles,
            })),
        ))
    }
}
```

**`backend/src/handlers/protected.rs`:**

```rust
use axum::{http::StatusCode, response::IntoResponse, Json};
use crate::auth::{guards::check_role, middleware::AuthUser};
use serde_json::json;

/// GET /api/profile — ต้องมี auth token (ทุก role)
pub async fn get_profile(auth: AuthUser) -> impl IntoResponse {
    Json(json!({
        "user_id": auth.user_id,
        "roles": auth.roles,
        "message": "Profile data"
    }))
}

/// GET /api/admin/users — ต้องมี role "admin"
pub async fn list_users(auth: AuthUser) -> impl IntoResponse {
    // ใช้ function-based check
    if let Err(rejection) = check_role(&auth, "admin") {
        return rejection.into_response();
    }

    Json(json!({
        "users": ["alice", "bob", "carol"],
        "message": "Admin: user list"
    }))
    .into_response()
}

/// DELETE /api/admin/users/:id — ต้องมี role "admin"
pub async fn delete_user(auth: AuthUser) -> impl IntoResponse {
    // ใช้ macro-based check
    crate::require_role!(auth, "admin");

    (StatusCode::NO_CONTENT, ()).into_response()
}

/// GET /api/editor/content — ต้องมี role "editor" หรือ "admin"
pub async fn get_editor_content(auth: AuthUser) -> impl IntoResponse {
    if !auth.has_role("editor") && !auth.has_role("admin") {
        return (
            StatusCode::FORBIDDEN,
            Json(json!({ "error": "Editor or Admin role required" })),
        )
            .into_response();
    }

    Json(json!({
        "content": "Editor content here",
        "editable": true
    }))
    .into_response()
}
```

---

### ขั้นที่ 6: Refresh Token Rotation

Refresh token rotation คือ security pattern ที่สำคัญมาก เมื่อ client ใช้ refresh token จะต้อง:
1. Invalidate token เก่า (ลบออกจาก store)
2. ออก token pair ใหม่ (access + refresh)

ถ้ามีคนขโมย refresh token และใช้ก่อน legitimate client ระบบจะตรวจพบ (token เก่าถูก revoke แล้ว) และสามารถ invalidate session ทั้งหมดได้

**`backend/src/handlers/refresh.rs`:**

```rust
use axum::{extract::State, http::StatusCode, Json};
use serde::Deserialize;
use crate::{
    auth::jwt::{create_access_token, create_refresh_token, verify_refresh_token},
    models::user::AuthResponse,
    store::memory::AppState,
};

#[derive(Debug, Deserialize)]
pub struct RefreshRequest {
    pub refresh_token: String,
}

/// POST /auth/refresh — Refresh Token Rotation
///
/// Security properties:
/// 1. Validate refresh token (signature + expiry)
/// 2. ตรวจว่า jti ยัง valid อยู่ใน store (ไม่ถูก revoke)
/// 3. Revoke token เก่าทันที (ก่อนออก token ใหม่)
/// 4. ออก token pair ใหม่และเก็บ jti ใหม่
///
/// ถ้าใครพยายาม reuse refresh token ที่ถูก rotate แล้ว
/// จะได้รับ 401 Unauthorized
pub async fn refresh_token(
    State(state): State<AppState>,
    Json(req): Json<RefreshRequest>,
) -> Result<(StatusCode, Json<AuthResponse>), (StatusCode, Json<serde_json::Value>)> {
    // 1. ตรวจสอบ signature และ expiry ของ refresh token
    let claims = verify_refresh_token(&req.refresh_token, &state.jwt_secret).map_err(|e| {
        let msg = match e {
            crate::auth::jwt::JwtError::Expired => "Refresh token expired",
            _ => "Invalid refresh token",
        };
        (
            StatusCode::UNAUTHORIZED,
            Json(serde_json::json!({ "error": msg })),
        )
    })?;

    // 2. ตรวจว่า jti ยัง valid อยู่ (ยังไม่ถูก revoke)
    if !state.is_refresh_token_valid(&claims.jti) {
        // อาจหมายความว่ามีการขโมย token — ควร log และ alert
        tracing::warn!(
            user_id = %claims.sub,
            jti = %claims.jti,
            "Attempted refresh with revoked token — possible token theft"
        );
        return Err((
            StatusCode::UNAUTHORIZED,
            Json(serde_json::json!({ "error": "Refresh token has been revoked" })),
        ));
    }

    // 3. Revoke token เก่า ทันที (ก่อนออก token ใหม่)
    //    ถ้า server crash หลังจากนี้ user ต้อง login ใหม่ (acceptable tradeoff)
    state.revoke_refresh_token(&claims.jti);

    // 4. หา user จาก store เพื่อดึง roles ล่าสุด
    //    (roles อาจเปลี่ยนไปตั้งแต่ออก token เดิม)
    let user_id = claims.sub.clone();
    let roles = find_user_roles(&state, &user_id);

    // 5. ออก token pair ใหม่
    const ACCESS_TTL: usize = 15 * 60;
    const REFRESH_TTL: usize = 7 * 24 * 60 * 60;

    let access_token = create_access_token(&user_id, roles.clone(), &state.jwt_secret, ACCESS_TTL)
        .map_err(|_| {
            (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({ "error": "Token creation failed" })),
            )
        })?;

    let (refresh_token, new_jti) =
        create_refresh_token(&user_id, &state.jwt_secret, REFRESH_TTL).map_err(|_| {
            (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({ "error": "Token creation failed" })),
            )
        })?;

    // 6. เก็บ jti ใหม่
    state.store_refresh_token(&new_jti, &user_id);

    Ok((
        StatusCode::OK,
        Json(AuthResponse {
            access_token,
            refresh_token,
            token_type: "Bearer".to_string(),
            expires_in: ACCESS_TTL as u64,
        }),
    ))
}

fn find_user_roles(state: &AppState, user_id: &str) -> Vec<String> {
    // หา user ที่มี id ตรงกัน
    // ในระบบจริงควรมี index by ID แต่ demo นี้ iterate
    for entry in state.users.iter() {
        if entry.id == user_id {
            return entry.role_strings();
        }
    }
    vec!["user".to_string()] // fallback
}
```

---

### ขั้นที่ 7: Main Router และ Server Setup

**`backend/src/main.rs`:**

```rust
mod auth;
mod handlers;
mod models;
mod store;

use axum::{
    routing::{delete, get, post},
    Router,
};
use store::memory::AppState;
use tower_http::{
    cors::{Any, CorsLayer},
    trace::TraceLayer,
};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    // ตั้งค่า tracing
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer())
        .init();

    let secret = std::env::var("JWT_SECRET")
        .unwrap_or_else(|_| "dev-secret-change-in-production".to_string());

    let state = AppState::new(secret);

    let app = build_router(state);

    let addr = "0.0.0.0:3000";
    tracing::info!("Server listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

pub fn build_router(state: AppState) -> Router {
    // Auth routes (ไม่ต้อง authenticate)
    let auth_routes = Router::new()
        .route("/register", post(handlers::register::register))
        .route("/login", post(handlers::register::login))
        .route("/refresh", post(handlers::refresh::refresh_token));

    // API routes (ต้อง authenticate ทุก request)
    let api_routes = Router::new()
        .route("/profile", get(handlers::protected::get_profile))
        .route("/editor/content", get(handlers::protected::get_editor_content));

    // Admin routes (ต้องมี role "admin")
    let admin_routes = Router::new()
        .route("/users", get(handlers::protected::list_users))
        .route("/users/:id", delete(handlers::protected::delete_user));

    Router::new()
        .nest("/auth", auth_routes)
        .nest("/api", api_routes)
        .nest("/api/admin", admin_routes)
        .layer(
            CorsLayer::new()
                .allow_origin(Any)
                .allow_methods(Any)
                .allow_headers(Any),
        )
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}
```

---

### ขั้นที่ 8: Frontend ด้วย Yew — AuthContext และ ProtectedRoute

**`frontend/Cargo.toml`:**

```toml
[package]
name = "fullstack-auth-frontend"
version = "0.1.0"
edition = "2021"

[dependencies]
yew = { version = "0.21", features = ["csr"] }
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = ["Window", "Storage", "Request", "RequestInit", "RequestMode", "Response", "Headers"] }
gloo-net = { version = "0.5", features = ["http"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[lib]
crate-type = ["cdylib", "rlib"]
```

**`frontend/src/context/auth.rs`:**

```rust
use serde::{Deserialize, Serialize};
use std::rc::Rc;
use yew::prelude::*;

/// Auth state ที่ share ผ่าน Context
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct AuthState {
    pub is_authenticated: bool,
    pub user_id: Option<String>,
    pub roles: Vec<String>,
    pub access_token: Option<String>,
    pub refresh_token: Option<String>,
}

impl AuthState {
    pub fn logged_out() -> Self {
        AuthState {
            is_authenticated: false,
            user_id: None,
            roles: vec![],
            access_token: None,
            refresh_token: None,
        }
    }

    pub fn has_role(&self, role: &str) -> bool {
        self.roles.iter().any(|r| r == role)
    }
}

/// Actions ที่สามารถทำกับ auth state
pub enum AuthAction {
    Login {
        user_id: String,
        roles: Vec<String>,
        access_token: String,
        refresh_token: String,
    },
    Logout,
    UpdateTokens {
        access_token: String,
        refresh_token: String,
    },
}

impl Reducible for AuthState {
    type Action = AuthAction;

    fn reduce(self: Rc<Self>, action: Self::Action) -> Rc<Self> {
        match action {
            AuthAction::Login {
                user_id,
                roles,
                access_token,
                refresh_token,
            } => Rc::new(AuthState {
                is_authenticated: true,
                user_id: Some(user_id),
                roles,
                access_token: Some(access_token),
                refresh_token: Some(refresh_token),
            }),
            AuthAction::Logout => Rc::new(AuthState::logged_out()),
            AuthAction::UpdateTokens {
                access_token,
                refresh_token,
            } => Rc::new(AuthState {
                access_token: Some(access_token),
                refresh_token: Some(refresh_token),
                ..(*self).clone()
            }),
        }
    }
}

pub type AuthContext = UseReducerHandle<AuthState>;

/// Props สำหรับ AuthProvider
#[derive(Properties, PartialEq)]
pub struct AuthProviderProps {
    pub children: Children,
}

/// AuthProvider — wrap ส่วน app ที่ต้องการ auth context
///
/// ใช้งาน:
/// ```
/// html! {
///     <AuthProvider>
///         <App />
///     </AuthProvider>
/// }
/// ```
#[function_component(AuthProvider)]
pub fn auth_provider(props: &AuthProviderProps) -> Html {
    let auth = use_reducer(AuthState::logged_out);

    html! {
        <ContextProvider<AuthContext> context={auth}>
            { for props.children.iter() }
        </ContextProvider<AuthContext>>
    }
}
```

**`frontend/src/components/protected_route.rs`:**

```rust
use yew::prelude::*;
use crate::context::auth::AuthContext;

/// Props สำหรับ ProtectedRoute
#[derive(Properties, PartialEq)]
pub struct ProtectedRouteProps {
    pub children: Children,
    /// Role ที่ต้องการ (ถ้าไม่ระบุ = ต้องแค่ login)
    #[prop_or_default]
    pub required_role: Option<String>,
}

/// ProtectedRoute — แสดง children เฉพาะเมื่อ user authenticated
///
/// ถ้าไม่ได้ login จะแสดง redirect ไปหน้า login
/// ถ้า login แล้วแต่ไม่มี role ที่ required จะแสดง 403
#[function_component(ProtectedRoute)]
pub fn protected_route(props: &ProtectedRouteProps) -> Html {
    let auth = use_context::<AuthContext>().expect("AuthContext not found");

    // ตรวจสอบว่า authenticated หรือยัง
    if !auth.is_authenticated {
        return html! {
            <div class="auth-required">
                <h2>{ "กรุณาเข้าสู่ระบบ" }</h2>
                <p>{ "คุณต้องเข้าสู่ระบบก่อนเข้าถึงหน้านี้" }</p>
                <a href="/login">{ "ไปหน้า Login" }</a>
            </div>
        };
    }

    // ตรวจสอบ role ถ้ามีการระบุ
    if let Some(ref required_role) = props.required_role {
        if !auth.has_role(required_role) {
            return html! {
                <div class="forbidden">
                    <h2>{ "ไม่มีสิทธิ์เข้าถึง" }</h2>
                    <p>{ format!("ต้องการ role: {}", required_role) }</p>
                </div>
            };
        }
    }

    // Authenticated และมี role ที่ถูกต้อง
    html! {
        { for props.children.iter() }
    }
}
```

**`frontend/src/components/login_form.rs`:**

```rust
use gloo_net::http::Request;
use serde::{Deserialize, Serialize};
use yew::prelude::*;
use crate::context::auth::{AuthAction, AuthContext};

#[derive(Debug, Serialize)]
struct LoginRequest {
    username: String,
    password: String,
}

#[derive(Debug, Deserialize)]
struct AuthResponse {
    access_token: String,
    refresh_token: String,
}

/// Login form component
#[function_component(LoginForm)]
pub fn login_form() -> Html {
    let auth = use_context::<AuthContext>().expect("AuthContext not found");
    let username = use_state(String::new);
    let password = use_state(String::new);
    let error = use_state(|| None::<String>);
    let loading = use_state(|| false);

    let on_submit = {
        let auth = auth.clone();
        let username = username.clone();
        let password = password.clone();
        let error = error.clone();
        let loading = loading.clone();

        Callback::from(move |e: SubmitEvent| {
            e.prevent_default();

            let auth = auth.clone();
            let username_val = (*username).clone();
            let password_val = (*password).clone();
            let error = error.clone();
            let loading = loading.clone();

            loading.set(true);
            error.set(None);

            wasm_bindgen_futures::spawn_local(async move {
                let body = LoginRequest {
                    username: username_val.clone(),
                    password: password_val,
                };

                let result = Request::post("http://localhost:3000/auth/login")
                    .json(&body)
                    .unwrap()
                    .send()
                    .await;

                loading.set(false);

                match result {
                    Ok(resp) if resp.ok() => {
                        if let Ok(data) = resp.json::<AuthResponse>().await {
                            // Parse roles จาก JWT (simplified — production ควรใช้ endpoint ที่คืน user info)
                            auth.dispatch(AuthAction::Login {
                                user_id: username_val,
                                roles: vec!["user".to_string()],
                                access_token: data.access_token,
                                refresh_token: data.refresh_token,
                            });
                        }
                    }
                    Ok(resp) => {
                        error.set(Some(format!("Login failed: {}", resp.status())));
                    }
                    Err(e) => {
                        error.set(Some(format!("Network error: {}", e)));
                    }
                }
            });
        })
    };

    html! {
        <div class="login-form">
            <h1>{ "เข้าสู่ระบบ" }</h1>
            if let Some(err) = (*error).clone() {
                <div class="error-banner">{ err }</div>
            }
            <form onsubmit={on_submit}>
                <div class="field">
                    <label>{ "ชื่อผู้ใช้" }</label>
                    <input
                        type="text"
                        value={(*username).clone()}
                        oninput={Callback::from({
                            let username = username.clone();
                            move |e: InputEvent| {
                                let input = e.target_unchecked_into::<web_sys::HtmlInputElement>();
                                username.set(input.value());
                            }
                        })}
                    />
                </div>
                <div class="field">
                    <label>{ "รหัสผ่าน" }</label>
                    <input
                        type="password"
                        value={(*password).clone()}
                        oninput={Callback::from({
                            let password = password.clone();
                            move |e: InputEvent| {
                                let input = e.target_unchecked_into::<web_sys::HtmlInputElement>();
                                password.set(input.value());
                            }
                        })}
                    />
                </div>
                <button type="submit" disabled={*loading}>
                    if *loading { "กำลังเข้าสู่ระบบ..." } else { "เข้าสู่ระบบ" }
                </button>
            </form>
        </div>
    }
}
```

**`frontend/src/main.rs`:**

```rust
mod api;
mod components;
mod context;

use components::{
    login_form::LoginForm,
    protected_route::ProtectedRoute,
};
use context::auth::AuthProvider;
use yew::prelude::*;

#[function_component(Dashboard)]
fn dashboard() -> Html {
    let auth = use_context::<context::auth::AuthContext>()
        .expect("AuthContext not found");

    let on_logout = {
        let auth = auth.clone();
        Callback::from(move |_| {
            auth.dispatch(context::auth::AuthAction::Logout);
        })
    };

    html! {
        <div class="dashboard">
            <h1>{ "Dashboard" }</h1>
            <p>{ format!("ยินดีต้อนรับ, {}!", auth.user_id.as_deref().unwrap_or("Unknown")) }</p>
            <p>{ format!("Roles: {:?}", auth.roles) }</p>
            <button onclick={on_logout}>{ "ออกจากระบบ" }</button>
        </div>
    }
}

#[function_component(App)]
fn app() -> Html {
    let auth = use_context::<context::auth::AuthContext>()
        .expect("AuthContext not found");

    html! {
        <div class="app">
            if auth.is_authenticated {
                // Protected content
                <ProtectedRoute>
                    <Dashboard />
                </ProtectedRoute>
                // Admin-only content
                <ProtectedRoute required_role="admin">
                    <div class="admin-panel">
                        <h2>{ "Admin Panel" }</h2>
                        <p>{ "เฉพาะ admin เท่านั้น" }</p>
                    </div>
                </ProtectedRoute>
            } else {
                <LoginForm />
            }
        </div>
    }
}

fn main() {
    yew::Renderer::<AuthProvider<App>>::new().render();
}
```

---

### ขั้นที่ 9: การทดสอบ (Testing)

ต่อไปนี้เป็น tests ที่รันได้จริง เราสร้าง test project ใน scratchpad directory เพื่อตรวจสอบ JWT และ password logic:

**Test code ที่ใช้ (ส่วนสำคัญ):**

```rust
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use argon2::{Argon2, PasswordHash, PasswordHasher, PasswordVerifier};
use argon2::password_hash::SaltString;
use rand_core::OsRng;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, PartialEq)]
pub struct Claims {
    pub sub: String,
    pub exp: usize,
    pub iat: usize,
    pub roles: Vec<String>,
}

// JWT tests
#[test]
fn test_jwt_encode_decode_roundtrip() {
    let token = create_token("user-123", vec!["admin".into()], SECRET, 900);
    let claims = verify_token(&token, SECRET).expect("Token should be valid");
    assert_eq!(claims.sub, "user-123");
    assert!(claims.roles.contains(&"admin".to_string()));
}

#[test]
fn test_jwt_wrong_secret_fails() {
    let token = create_token("bob", vec![], SECRET, 900);
    let result = verify_token(&token, "wrong_secret");
    assert!(result.is_err(), "Verification with wrong secret should fail");
}

#[test]
fn test_jwt_tampered_token_fails() {
    let token = create_token("carol", vec!["user".into()], SECRET, 900);
    let mut tampered = token.clone();
    let last = tampered.pop().unwrap();
    let replacement = if last == 'a' { 'b' } else { 'a' };
    tampered.push(replacement);
    let result = verify_token(&tampered, SECRET);
    assert!(result.is_err(), "Tampered token should fail verification");
}

// Password tests
#[test]
fn test_password_hash_and_verify() {
    let hash = hash_password("correct_horse_battery_staple");
    assert!(verify_password("correct_horse_battery_staple", &hash));
}

#[test]
fn test_same_password_produces_different_hashes() {
    let h1 = hash_password("password123");
    let h2 = hash_password("password123");
    assert_ne!(h1, h2, "Same password should produce different hashes (random salt)");
}
```

**ผลการรัน `cargo test` จริง:**

```
   Compiling auth_tests v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 2.74s
     Running unittests src/main.rs (target/debug/deps/auth_tests-dba930437f6597d5)

running 13 tests
test tests::test_admin_role_present ... ok
test tests::test_extract_subject_from_token ... ok
test tests::test_jwt_contains_correct_claims ... ok
test tests::test_jwt_encode_decode_roundtrip ... ok
test tests::test_empty_roles_claim ... ok
test tests::test_jwt_tampered_token_fails ... ok
test tests::test_jwt_iat_and_exp_fields ... ok
test tests::test_jwt_wrong_secret_fails ... ok
test tests::test_jwt_structure_has_three_parts ... ok
test tests::test_argon2_hash_format ... ok
test tests::test_wrong_password_fails_verify ... ok
test tests::test_same_password_produces_different_hashes ... ok
test tests::test_password_hash_and_verify ... ok

test result: ok. 13 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 4.01s
```

Tests ครอบคลุม:

| Test | สิ่งที่ทดสอบ |
|------|-------------|
| `test_jwt_encode_decode_roundtrip` | JWT สร้างและ decode ได้ถูกต้อง |
| `test_jwt_contains_correct_claims` | Claims มีค่า sub และ roles ถูกต้อง |
| `test_jwt_wrong_secret_fails` | Token ด้วย secret ผิดต้อง fail |
| `test_jwt_tampered_token_fails` | Token ที่ถูกแก้ไขต้อง fail |
| `test_jwt_structure_has_three_parts` | JWT ต้องมี 3 ส่วน คั่นด้วย `.` |
| `test_jwt_iat_and_exp_fields` | `exp = iat + exp_secs` |
| `test_password_hash_and_verify` | Hash แล้ว verify ได้ถูกต้อง |
| `test_wrong_password_fails_verify` | Password ผิดต้อง fail |
| `test_same_password_produces_different_hashes` | Random salt ทำให้ hash ต่างกัน |
| `test_argon2_hash_format` | Hash ขึ้นต้นด้วย `$argon2` |
| `test_extract_subject_from_token` | ดึง `sub` จาก token ได้ |
| `test_empty_roles_claim` | `roles: []` ทำงานได้ |
| `test_admin_role_present` | Role "admin" อยู่ใน claims |

---

## การทดสอบแบบ Integration

สำหรับ backend API tests ใช้ `axum-test` crate:

**`backend/tests/auth_flow.rs`:**

```rust
use axum_test::TestServer;
use serde_json::json;

// สร้าง test server จาก router ของเรา
async fn create_test_server() -> TestServer {
    let state = fullstack_auth_backend::store::memory::AppState::new(
        "test-secret-key".to_string()
    );
    let app = fullstack_auth_backend::build_router(state);
    TestServer::new(app).unwrap()
}

#[tokio::test]
async fn test_register_and_login_flow() {
    let server = create_test_server().await;

    // Register
    let register_res = server
        .post("/auth/register")
        .json(&json!({
            "username": "testuser",
            "email": "test@example.com",
            "password": "SecurePass1"
        }))
        .await;

    register_res.assert_status_ok();
    let body: serde_json::Value = register_res.json();
    assert!(body["access_token"].is_string());
    assert!(body["refresh_token"].is_string());

    // Login
    let login_res = server
        .post("/auth/login")
        .json(&json!({
            "username": "testuser",
            "password": "SecurePass1"
        }))
        .await;

    login_res.assert_status_ok();
}

#[tokio::test]
async fn test_duplicate_username_rejected() {
    let server = create_test_server().await;

    let body = json!({
        "username": "duplicate",
        "email": "dup@example.com",
        "password": "SecurePass1"
    });

    // First registration should succeed
    server.post("/auth/register").json(&body).await.assert_status_ok();

    // Second registration should fail with 409 Conflict
    let res = server.post("/auth/register").json(&body).await;
    res.assert_status(axum::http::StatusCode::CONFLICT);
}

#[tokio::test]
async fn test_protected_route_requires_token() {
    let server = create_test_server().await;

    // ไม่มี token
    let res = server.get("/api/profile").await;
    res.assert_status(axum::http::StatusCode::UNAUTHORIZED);
}

#[tokio::test]
async fn test_refresh_token_rotation() {
    let server = create_test_server().await;

    // Register เพื่อได้ token pair
    let reg_res = server
        .post("/auth/register")
        .json(&json!({
            "username": "rotateuser",
            "email": "rotate@example.com",
            "password": "SecurePass1"
        }))
        .await;

    let tokens: serde_json::Value = reg_res.json();
    let refresh_token = tokens["refresh_token"].as_str().unwrap();

    // ใช้ refresh token ครั้งแรก — ควรสำเร็จ
    let refresh_res = server
        .post("/auth/refresh")
        .json(&json!({ "refresh_token": refresh_token }))
        .await;

    refresh_res.assert_status_ok();
    let new_tokens: serde_json::Value = refresh_res.json();
    let new_refresh = new_tokens["refresh_token"].as_str().unwrap();

    // ตรวจสอบว่า token ใหม่ไม่เหมือนเก่า (rotation)
    assert_ne!(refresh_token, new_refresh);

    // ลอง reuse refresh token เก่า — ควร fail
    let reuse_res = server
        .post("/auth/refresh")
        .json(&json!({ "refresh_token": refresh_token }))
        .await;

    reuse_res.assert_status(axum::http::StatusCode::UNAUTHORIZED);
}
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### ข้อผิดพลาดที่ 1: เก็บ JWT Secret ใน Code

```rust
// ❌ ผิด — อย่าทำแบบนี้เด็ดขาด
const JWT_SECRET: &str = "my-super-secret-key-hardcoded";

fn create_token(user_id: &str) -> String {
    // ...
    EncodingKey::from_secret(JWT_SECRET.as_bytes())
}
```

```rust
// ✅ ถูก — อ่านจาก environment variable
fn main() {
    let secret = std::env::var("JWT_SECRET")
        .expect("JWT_SECRET environment variable must be set");
    // ...
}
```

**เหตุผล**: Secret ที่ hardcode จะถูกเห็นใน:
- Source code (ถ้า push ขึ้น Git)
- Binary file (ด้วย `strings` command)
- Docker image layers

ใช้ tool อย่าง `cargo-geiger` หรือ `trufflehog` เพื่อ scan หา hardcoded secrets

---

### ข้อผิดพลาดที่ 2: ไม่ตรวจสอบ Token Type

```rust
// ❌ ผิด — ใช้ Claims เดียวกันสำหรับทั้ง access และ refresh token
async fn protected_handler(token: String) -> Result<...> {
    let claims = verify_token(&token, SECRET)?;
    // ถ้า user ส่ง refresh token มาแทน access token จะผ่านไปได้!
}
```

```rust
// ✅ ถูก — แยก Claims struct และตรวจสอบ token_type
async fn protected_handler(token: String) -> Result<...> {
    let claims = verify_access_token(&token, SECRET)?;
    // verify_access_token จะ return Err ถ้า token_type != "access"
}
```

**เหตุผล**: Refresh token มีอายุนาน 7 วัน ถ้าไม่ตรวจสอบ type attacker อาจใช้ refresh token ที่ขโมยมาเพื่อเข้า protected API ได้นานกว่า 15 นาที

---

### ข้อผิดพลาดที่ 3: Username Enumeration Attack

```rust
// ❌ ผิด — error message บอกข้อมูลมากเกินไป
pub async fn login(req: LoginRequest) -> Result<...> {
    let user = find_user(&req.username)
        .ok_or(Error::UserNotFound("Username not found"))?;  // ❌ บอกว่า username ไม่มีอยู่
    if !verify_password(&req.password, &user.hash)? {
        return Err(Error::WrongPassword("Wrong password"));  // ❌ บอกว่า password ผิด
    }
}
```

```rust
// ✅ ถูก — ใช้ error message เดียวกันทั้งสองกรณี
pub async fn login(req: LoginRequest) -> Result<...> {
    let user = find_user(&req.username)
        .ok_or(Error::Unauthorized("Invalid credentials"))?;  // ✅ ไม่บอกว่าใครผิด
    if !verify_password(&req.password, &user.hash)? {
        return Err(Error::Unauthorized("Invalid credentials"));  // ✅ เหมือนกัน
    }
}
```

**เหตุผล**: ถ้า error message บอกว่า "Username not found" vs "Wrong password" attacker สามารถ enumerate ได้ว่า username ไหนมีอยู่ในระบบ แล้วเน้น brute-force เฉพาะ username ที่มีอยู่จริง

---

### ข้อผิดพลาดที่ 4: ไม่ Revoke Refresh Token เมื่อ User Logout

```rust
// ❌ ผิด — logout แค่ clear token ที่ client
pub async fn logout(auth: AuthUser) -> impl IntoResponse {
    // Client ลบ token ออกจาก localStorage
    // แต่ refresh token ยังใช้งานได้บน server!
    StatusCode::OK
}
```

```rust
// ✅ ถูก — revoke refresh token บน server ด้วย
pub async fn logout(
    State(state): State<AppState>,
    Json(req): Json<LogoutRequest>,  // รับ refresh_token จาก client
) -> impl IntoResponse {
    // ตรวจสอบ refresh token เพื่อดึง jti
    if let Ok(claims) = verify_refresh_token(&req.refresh_token, &state.jwt_secret) {
        state.revoke_refresh_token(&claims.jti);
    }
    StatusCode::OK
}
```

**เหตุผล**: ถ้าไม่ revoke บน server คนที่ขโมย refresh token ไปก็ยังใช้ได้จนกว่า token จะหมดอายุเอง (7 วัน)

---

### ข้อผิดพลาดที่ 5: ใช้ bcrypt แทน Argon2 ในปี 2024

```rust
// ❌ ไม่แนะนำ — bcrypt ถูก design ในปี 1999
// ไม่ memory-hard, GPU attack ทำได้ง่ายกว่า
use bcrypt::{hash, verify, DEFAULT_COST};
let hashed = hash(password, DEFAULT_COST)?;
```

```rust
// ✅ แนะนำ — Argon2id เป็น best practice ปัจจุบัน
use argon2::{Argon2, PasswordHasher};
let argon2 = Argon2::default(); // Argon2id variant
let hash = argon2.hash_password(password.as_bytes(), &salt)?;
```

**เหตุผล**: bcrypt มี maximum password length ที่ 72 bytes และไม่ memory-hard Argon2 ชนะ PHC และ NIST แนะนำให้ใช้

---

### ข้อผิดพลาดที่ 6: ไม่ validate exp ใน Token

```rust
// ❌ ผิด — disable exp validation โดยไม่มีเหตุผล
let mut validation = Validation::new(Algorithm::HS256);
validation.validate_exp = false; // ❌ อย่าทำแบบนี้ใน production!
```

```rust
// ✅ ถูก — ปล่อยให้ validate_exp เป็น true (default)
let validation = Validation::new(Algorithm::HS256);
// validate_exp = true โดย default
let data = decode::<Claims>(&token, &key, &validation)?;
// ถ้า token หมดอายุจะได้ Err(ExpiredSignature) อัตโนมัติ
```

**หมายเหตุ**: การ disable `validate_exp = false` มีประโยชน์ใน unit tests เท่านั้น (เพื่อใช้ token ที่สร้างด้วย timestamp คงที่)

---

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# Backend
cd backend
JWT_SECRET=$(openssl rand -base64 32) cargo build --release

# Frontend (ต้องติดตั้ง wasm-pack ก่อน)
cd frontend
wasm-pack build --target web --out-dir ../dist/wasm

# หรือใช้ Trunk
cd frontend
trunk build --release
```

### Docker Setup

**`Dockerfile` สำหรับ Backend:**

```dockerfile
# Build stage
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY backend/ ./backend/
RUN cargo build --release -p fullstack-auth-backend

# Runtime stage
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/fullstack-auth-backend /usr/local/bin/
EXPOSE 3000
CMD ["fullstack-auth-backend"]
```

**`docker-compose.yml`:**

```yaml
version: '3.8'
services:
  backend:
    build: .
    ports:
      - "3000:3000"
    environment:
      - JWT_SECRET=${JWT_SECRET}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    image: nginx:alpine
    volumes:
      - ./dist:/usr/share/nginx/html:ro
    ports:
      - "8080:80"
    depends_on:
      - backend
```

### Environment Variables

```bash
# .env (อย่า commit ไฟล์นี้!)
JWT_SECRET=your-256-bit-secret-here-generate-with-openssl-rand-base64-32
RUST_LOG=info
PORT=3000
```

---

## การต่อยอด — แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม RS256 (Asymmetric JWT)

เปลี่ยน backend จาก HS256 (shared secret) เป็น RS256 (public/private key pair) โดย:

1. สร้าง RSA key pair ด้วย `openssl genrsa -out private.pem 2048`
2. Extract public key: `openssl rsa -in private.pem -pubout -out public.pem`
3. แก้ `create_access_token` ให้ใช้ `EncodingKey::from_rsa_pem`
4. แก้ `verify_access_token` ให้ใช้ `DecodingKey::from_rsa_pem` (public key เท่านั้น)

**ประโยชน์**: microservices อื่นสามารถ verify token ได้โดยใช้แค่ public key ไม่ต้องแชร์ secret

```rust
// ตัวอย่าง RS256 setup
use jsonwebtoken::{Algorithm, Header};

pub fn create_access_token_rs256(
    user_id: &str,
    roles: Vec<String>,
    private_key_pem: &[u8],
    exp_secs: usize,
) -> Result<String, JwtError> {
    let header = Header::new(Algorithm::RS256);
    let key = EncodingKey::from_rsa_pem(private_key_pem)?;
    // ...
}
```

### แบบฝึกหัดที่ 2: เพิ่ม Rate Limiting บน Auth Endpoints

ป้องกัน brute-force attack โดยเพิ่ม rate limiting บน `/auth/login`:

1. เพิ่ม `tower_governor` crate (rate limiting middleware)
2. Limit: สูงสุด 5 requests ต่อ minute per IP
3. หลังจาก 5 ครั้งผิด ให้ lock account 15 นาที
4. เพิ่ม test สำหรับ rate limiting behavior

```toml
[dependencies]
tower_governor = "0.3"
```

```rust
use tower_governor::{governor::GovernorConfigBuilder, GovernorLayer};

let governor_conf = GovernorConfigBuilder::default()
    .per_minute(5)
    .burst_size(5)
    .finish()
    .unwrap();

let app = Router::new()
    .route("/auth/login", post(login))
    .layer(GovernorLayer { config: Arc::new(governor_conf) });
```

### แบบฝึกหัดที่ 3: เพิ่ม PostgreSQL แทน In-Memory Store

แทน `DashMap` ด้วย PostgreSQL โดยใช้ `sqlx`:

1. เพิ่ม `sqlx` crate พร้อม `postgres` feature
2. สร้าง migration SQL:
   ```sql
   CREATE TABLE users (
     id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
     username VARCHAR(32) UNIQUE NOT NULL,
     email VARCHAR(255) UNIQUE NOT NULL,
     password_hash TEXT NOT NULL,
     roles TEXT[] NOT NULL DEFAULT '{user}',
     created_at TIMESTAMPTZ DEFAULT NOW()
   );

   CREATE TABLE refresh_tokens (
     jti UUID PRIMARY KEY,
     user_id UUID NOT NULL REFERENCES users(id),
     expires_at TIMESTAMPTZ NOT NULL,
     created_at TIMESTAMPTZ DEFAULT NOW()
   );
   ```
3. แทนที่ `AppState.users` ด้วย `PgPool`
4. เพิ่ม integration test ด้วย `testcontainers` crate (PostgreSQL ใน Docker)

### แบบฝึกหัดที่ 4: Two-Factor Authentication (TOTP)

เพิ่ม TOTP (Time-based One-Time Password) เช่นที่ Google Authenticator ใช้:

1. เพิ่ม `totp-rs` crate
2. เพิ่ม endpoint `POST /auth/2fa/setup` — generate TOTP secret และ QR code URI
3. เพิ่ม endpoint `POST /auth/2fa/verify` — ตรวจสอบ 6-digit code
4. แก้ login flow: หลัง password ถูกต้อง ถ้า user มี 2FA enabled ต้องใส่ TOTP code ด้วย

```toml
[dependencies]
totp-rs = { version = "5", features = ["gen_secret"] }
```

```rust
use totp_rs::{Algorithm, Secret, TOTP};

pub fn generate_totp_secret() -> String {
    let secret = Secret::generate_secret();
    secret.to_encoded().to_string()
}

pub fn verify_totp(secret: &str, code: &str) -> bool {
    let totp = TOTP::new(
        Algorithm::SHA1, 6, 1, 30,
        Secret::Encoded(secret.to_string()).to_bytes().unwrap(),
    ).unwrap();
    totp.check_current(code).unwrap_or(false)
}
```

### แบบฝึกหัดที่ 5: OAuth2 / Social Login

เพิ่ม "Login with GitHub" โดยใช้ OAuth2 Authorization Code flow:

1. Register GitHub OAuth App ที่ github.com/settings/developers
2. เพิ่ม `oauth2` crate
3. สร้าง endpoint `GET /auth/github` — redirect ไป GitHub
4. สร้าง endpoint `GET /auth/github/callback` — รับ code, exchange เป็น access_token, ดึง user info
5. สร้างหรืออัปเดต user record จาก GitHub profile

### แบบฝึกหัดที่ 6: Audit Log

เพิ่ม security audit log สำหรับทุก auth event:

1. สร้าง `AuditEvent` struct: `{event_type, user_id, ip_address, timestamp, success}`
2. Log ทุก: login attempt (success/fail), logout, refresh, password change
3. เพิ่ม endpoint `GET /api/admin/audit-log` สำหรับ admin ดู log
4. ส่ง suspicious event (เช่น multiple failed login) ไปยัง notification system

---

## สรุป

ในโปรเจคนี้เราได้สร้าง full-stack authentication system ที่สมบูรณ์ตั้งแต่ต้นจนจบ ครอบคลุม:

**ด้าน Security:**
- JWT token ด้วย HS256, แยก access/refresh token อย่างชัดเจน
- Argon2 password hashing ที่ memory-hard และ side-channel resistant
- Refresh token rotation ที่ป้องกัน reuse/replay attack
- Role-based access control ผ่าน `AuthUser` extractor ที่ compile-time safe

**ด้าน Architecture:**
- `FromRequestParts` trait ของ Axum ทำให้ authentication เป็น type-safe และ composable
- `DashMap` สำหรับ concurrent in-memory state โดยไม่มี lock contention
- Tower middleware layers สำหรับ CORS และ tracing

**ด้าน Frontend:**
- Yew `use_reducer` สำหรับ auth state management แบบ Flux pattern
- `ContextProvider` และ `use_context` สำหรับ share auth state ทั่วทั้ง component tree
- `ProtectedRoute` component ที่ reusable และรองรับ role-based protection

Pattern เหล่านี้สามารถนำไปใช้ใน production application ได้จริง เพียงแต่ต้อง:
1. เปลี่ยน in-memory store เป็น database จริง (PostgreSQL + sqlx)
2. ใช้ RS256 แทน HS256 สำหรับ distributed systems
3. เพิ่ม rate limiting, account lockout, และ audit logging
4. ตั้งค่า HTTPS (TLS) บน production server เสมอ

---

## บทสรุปของการเดินทาง — ครบ 100 โปรเจค

ถึงตอนนี้คุณได้เดินทางผ่านโปรเจคทั้ง 100 โปรเจคของหลักสูตร Rust ครบสมบูรณ์แล้ว จากโปรเจคแรกที่สร้าง Shell Interpreter ด้วย Rust พื้นฐาน ไปจนถึงโปรเจคสุดท้ายที่เป็น Full-Stack Application พร้อม Authentication คุณได้ผ่านการสร้างระบบที่หลากหลาย ตั้งแต่ CLI tools ที่รันบน terminal, Web Services ที่รองรับ traffic จำนวนมาก, Data Processing pipelines ที่ประมวลผล dataset ขนาดใหญ่, Security tools ที่ปกป้องข้อมูล, Games และ Graphics ที่ใช้ GPU, Distributed Systems ที่ทำงานข้าม nodes, DevOps infrastructure ที่ automate deployment, Networking protocols ตั้งแต่ TCP ถึง QUIC, ML/AI systems ที่ implement algorithm ด้วยมือ และในที่สุดคือ Full-Stack WASM applications ที่รัน Rust ทั้งบน server และบน browser

สิ่งที่คุณได้เรียนรู้ตลอดการเดินทางนี้ไม่ใช่แค่ syntax หรือ API ของ Rust แต่คือ **วิธีคิดแบบ Systems Programmer** — ความเข้าใจว่าโปรแกรมทำงานอย่างไรในระดับ hardware, ทำไม ownership system ถึงทำให้ memory safety และ concurrency safety เป็นไปได้โดยไม่มี garbage collector, และทำไม Rust จึงกลายเป็นภาษาที่ Linux kernel, Android, Windows, และ Firefox ต่างก็เลือกใช้

หลักสูตรนี้ออกแบบมาให้ทุกโปรเจคมี "real output จากการรันจริง" เพราะโปรแกรมที่ดีคือโปรแกรมที่รันได้ ไม่ใช่แค่โปรแกรมที่อ่านสวย ขอให้นำทักษะที่ได้ไปสร้างสิ่งที่มีความหมาย — ไม่ว่าจะเป็น open-source library ที่ผู้คนจะใช้, ระบบที่แก้ปัญหาจริงของธุรกิจคุณ, หรือการมีส่วนร่วมใน Rust ecosystem ที่กำลังเติบโตอยู่ทุกวัน

**ยินดีด้วยที่สำเร็จหลักสูตร Rust 100 โปรเจค** ขอให้โค้ดทุกบรรทัดที่คุณเขียนต่อจากนี้ไป compile ผ่านในครั้งแรก และ `cargo test` ทุก test ผ่านตลอดไป

```
The Rust Programming Language
"The compiler is your friend."
```

---

**โปรเจคก่อนหน้า:** [project-j09-wasm-bindgen.md](project-j09-wasm-bindgen.md) | **โปรเจคถัดไป:** (โปรเจคนี้คือโปรเจคสุดท้าย — ลำดับที่ 100 ของหลักสูตร)
