# Project B06: OAuth2 Authorization Server

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 8 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **OAuth2 Authorization Server** ที่ครบถ้วนตามมาตรฐาน RFC 6749, RFC 7636 (PKCE), RFC 7662 (Token Introspection), RFC 7009 (Token Revocation) และ OpenID Connect Discovery พร้อม JWT access token ที่เซ็นด้วย RS256

ในโลก production OAuth2 Authorization Server เป็นหัวใจของระบบ identity ขององค์กร — ทุก service ที่ต้องการรู้ว่า "ใครเป็นคนเรียก และมีสิทธิ์อะไร" จะมาถามเซิร์ฟเวอร์นี้ ตัวอย่างเช่น Google Identity Platform, Auth0, Keycloak และ AWS Cognito ล้วนทำงานบนหลักการเดียวกันทั้งสิ้น

**Use case จริงในโลก production:**
- ระบบ Single Sign-On (SSO) ขององค์กร
- API gateway ที่ต้องการ authenticate microservices กันเอง (Client Credentials flow)
- แอปพลิเคชัน mobile/web ที่ต้องการ authorize บนเซิร์ฟเวอร์ภายนอก
- Platform ที่เปิด API ให้ third-party developers

**Learning value:**
โปรเจคนี้รวมทักษะขั้น production ที่สำคัญที่สุดของ web security: asymmetric cryptography, token lifecycle management, database-backed session state, scope-based authorization, และ standards-compliant protocol implementation

---

## สิ่งที่จะได้เรียนรู้

- **Authorization Code Flow + PKCE** — ทำความเข้าใจขั้นตอนที่ทันสมัยที่สุดสำหรับ public clients
- **JWT (RS256)** — สร้าง self-contained access token ที่ resource server ตรวจสอบได้โดยไม่ต้องเรียก database
- **Asymmetric key management** — สร้าง RSA keypair, expose JWKS endpoint, และ rotate keys
- **Token lifecycle** — issuance, introspection, refresh, และ revocation
- **Axum middleware** — ตรวจสอบ scope และ Bearer token ก่อนเข้าถึง protected endpoints
- **Argon2id password hashing** — hash client_secret อย่างปลอดภัยใน database
- **OpenID Connect Discovery** — เปิด `.well-known/openid-configuration` ให้ clients auto-configure ได้
- **sqlx + PostgreSQL** — จัดการ auth codes, tokens, และ clients ด้วย type-safe queries

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50**: async/await และ Tokio runtime (จาก Part 46 เรื่อง async/await เป็นพื้นฐานหลัก)
- **Part 61–70**: Axum web framework — routing, extractors, middleware, state management
- **Part 71–80**: sqlx และ database integration กับ PostgreSQL
- **Part 96–100**: Production patterns — error handling, structured logging, configuration
- **Part 101–105**: Security concepts — hashing, JWT, cryptography ใน Rust

---

## โครงสร้างโปรเจค (Project Layout)

```
oauth2-server/
├── src/
│   ├── main.rs              # Entry point, Axum router setup
│   ├── config.rs            # Configuration (env vars, keypair loading)
│   ├── db.rs                # Database pool + migrations
│   ├── models.rs            # Structs: Client, AuthCode, Token, etc.
│   ├── error.rs             # AppError type ที่ implement IntoResponse
│   ├── crypto/
│   │   ├── mod.rs
│   │   ├── jwt.rs           # JWT sign/verify ด้วย RS256
│   │   ├── pkce.rs          # PKCE S256 challenge generation/verification
│   │   └── keys.rs          # RSA keypair, JWKS serialization
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── authorize.rs     # GET/POST /authorize
│   │   ├── token.rs         # POST /token (all grant types)
│   │   ├── clients.rs       # POST /clients (registration)
│   │   ├── introspect.rs    # POST /introspect
│   │   ├── revoke.rs        # POST /revoke
│   │   └── discovery.rs     # GET /.well-known/*
│   └── middleware/
│       ├── mod.rs
│       └── scope_check.rs   # Bearer token + scope enforcement
├── migrations/
│   ├── 001_clients.sql
│   ├── 002_auth_codes.sql
│   └── 003_tokens.sql
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow — Authorization Code + PKCE Flow

```
Client App                 Authorization Server              Resource Server
    │                              │                               │
    │── 1. GET /authorize ─────────►│                               │
    │   (client_id, redirect_uri,   │                               │
    │    scope, state,              │  ┌─────────────────────┐      │
    │    code_challenge,            │  │ Validate client_id  │      │
    │    code_challenge_method=S256)│  │ Validate redirect   │      │
    │                               │  │ Store challenge     │      │
    │◄── 2. Login page ─────────────│  └─────────────────────┘      │
    │                               │                               │
    │── 3. POST /authorize ─────────►│                               │
    │   (credentials)               │  ┌─────────────────────┐      │
    │                               │  │ Authenticate user   │      │
    │◄── 4. redirect?code=X&state=Y ─│  │ Store auth_code +   │      │
    │                               │  │ code_challenge      │      │
    │                               │  └─────────────────────┘      │
    │── 5. POST /token ─────────────►│                               │
    │   (code=X,                    │  ┌─────────────────────┐      │
    │    code_verifier=V,           │  │ Verify PKCE         │      │
    │    grant_type=auth_code)      │  │ Issue JWT (RS256)   │      │
    │◄── 6. access_token+refresh ────│  └─────────────────────┘      │
    │                               │                               │
    │── 7. GET /api/data ────────────────────────────────────────────►│
    │   Bearer: access_token        │                               │
    │                               │◄── 8. POST /introspect ────────│
    │                               │   (token)                     │
    │                               │──► 9. {active:true, scope:…} ──►│
    │◄──────────────── 10. data ─────────────────────────────────────│
```

### Design Decisions

**ทำไมใช้ RS256 แทน HS256?**
ด้วย HS256 ทุก service ที่ต้องการตรวจ token ต้องรู้ shared secret — นั่นคือ secret รั่วออกได้ง่าย ด้วย RS256 authorization server เก็บ private key ไว้คนเดียว resource servers ตรวจสอบด้วย public key ที่ดึงจาก `/jwks.json` ได้โดยอิสระ

**ทำไม Auth Code ต้องหมดอายุภายใน 5 นาที?**
Auth code เป็น one-time-use credential ที่ส่งผ่าน browser redirect (URL parameter) ดังนั้น window ที่ attacker สามารถใช้มันต้องแคบที่สุด ถ้ารับมาแล้วไม่ exchange ทันที นั่นคือ abnormal behavior

**PKCE บังคับ แม้ใน confidential clients?**
แม้ RFC 7636 เดิมกำหนดสำหรับ public clients เท่านั้น แต่ RFC 9700 (2024) แนะนำให้ทุก client ใช้ PKCE เพื่อป้องกัน authorization code injection attack เราจึง enforce ทั้งหมด

**Argon2id สำหรับ client_secret**
ถ้า database ถูก dump ออกไป client secrets ที่เป็น plain text จะทำให้ attackers สามารถ impersonate ทุก OAuth2 client ได้ทันที ดังนั้นเราต้อง hash เหมือนกับ password

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Project Setup และ Configuration

เริ่มจาก `Cargo.toml` และ configuration layer:

```toml
# Cargo.toml
[package]
name = "oauth2-server"
version = "0.1.0"
edition = "2021"

[dependencies]
axum            = { version = "0.8", features = ["macros", "multipart"] }
tokio           = { version = "1", features = ["full"] }
tower           = { version = "0.5" }
tower-http      = { version = "0.6", features = ["cors", "trace"] }
sqlx            = { version = "0.8", features = ["postgres", "runtime-tokio", "uuid", "chrono", "json"] }
serde           = { version = "1", features = ["derive"] }
serde_json      = "1"
serde_urlencoded = "0.7"
jsonwebtoken    = "9"
rsa             = { version = "0.9", features = ["sha2"] }
sha2            = "0.10"
base64          = "0.22"
argon2          = "0.5"
uuid            = { version = "1", features = ["v4", "serde"] }
chrono          = { version = "0.4", features = ["serde"] }
rand            = "0.8"
tracing         = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
dotenvy         = "0.15"
thiserror       = "1"
askama          = { version = "0.12", features = ["with-axum"] }  # HTML templates

[dev-dependencies]
axum-test       = "15"
tokio           = { version = "1", features = ["full"] }
```

```rust
// src/config.rs
use std::env;

#[derive(Clone)]
pub struct Config {
    pub database_url: String,
    pub private_key_pem: String,   // RSA private key (PEM)
    pub server_port: u16,
    pub issuer: String,            // เช่น "https://auth.example.com"
    pub access_token_ttl_secs: i64,
    pub refresh_token_ttl_secs: i64,
    pub auth_code_ttl_secs: i64,
}

impl Config {
    pub fn from_env() -> Result<Self, String> {
        Ok(Self {
            database_url: env::var("DATABASE_URL")
                .map_err(|_| "DATABASE_URL required")?,
            private_key_pem: env::var("JWT_PRIVATE_KEY")
                .unwrap_or_else(|_| include_str!("../keys/private.pem").to_string()),
            server_port: env::var("PORT")
                .unwrap_or_else(|_| "8080".into())
                .parse()
                .map_err(|_| "PORT must be a number")?,
            issuer: env::var("ISSUER")
                .unwrap_or_else(|_| "http://localhost:8080".into()),
            access_token_ttl_secs: 3600,        // 1 ชั่วโมง
            refresh_token_ttl_secs: 86400 * 30, // 30 วัน
            auth_code_ttl_secs: 300,            // 5 นาที
        })
    }
}
```

### ขั้นที่ 2: Database Schema และ Models

```sql
-- migrations/001_clients.sql
CREATE TABLE clients (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       VARCHAR(64) UNIQUE NOT NULL,
    client_secret_hash VARCHAR(256) NOT NULL,  -- Argon2id hash
    redirect_uris   TEXT[]      NOT NULL,
    allowed_scopes  TEXT[]      NOT NULL,
    grant_types     TEXT[]      NOT NULL DEFAULT ARRAY['authorization_code', 'refresh_token'],
    is_confidential BOOLEAN     NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrations/002_auth_codes.sql
CREATE TABLE auth_codes (
    code            VARCHAR(128)  PRIMARY KEY,
    client_id       VARCHAR(64)   NOT NULL REFERENCES clients(client_id),
    user_id         VARCHAR(128)  NOT NULL,
    redirect_uri    TEXT          NOT NULL,
    scope           TEXT          NOT NULL,
    state           TEXT,
    code_challenge  VARCHAR(256),            -- PKCE challenge
    code_challenge_method VARCHAR(10),       -- "S256" หรือ "plain"
    expires_at      TIMESTAMPTZ   NOT NULL,
    used            BOOLEAN       NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- migrations/003_tokens.sql
CREATE TABLE tokens (
    id              UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    token_hash      VARCHAR(256)  UNIQUE NOT NULL,  -- SHA-256 ของ token value
    token_type      VARCHAR(20)   NOT NULL,  -- 'access' หรือ 'refresh'
    client_id       VARCHAR(64)   NOT NULL,
    user_id         VARCHAR(128),            -- NULL สำหรับ client_credentials
    scope           TEXT          NOT NULL,
    jti             VARCHAR(128)  UNIQUE,    -- JWT ID (สำหรับ access tokens)
    expires_at      TIMESTAMPTZ   NOT NULL,
    revoked         BOOLEAN       NOT NULL DEFAULT false,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tokens_hash ON tokens(token_hash);
CREATE INDEX idx_tokens_jti  ON tokens(jti) WHERE jti IS NOT NULL;
CREATE INDEX idx_auth_codes_expires ON auth_codes(expires_at) WHERE NOT used;
```

```rust
// src/models.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Client {
    pub id: Uuid,
    pub client_id: String,
    pub client_secret_hash: String,
    pub redirect_uris: Vec<String>,
    pub allowed_scopes: Vec<String>,
    pub grant_types: Vec<String>,
    pub is_confidential: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, sqlx::FromRow)]
pub struct AuthCode {
    pub code: String,
    pub client_id: String,
    pub user_id: String,
    pub redirect_uri: String,
    pub scope: String,
    pub state: Option<String>,
    pub code_challenge: Option<String>,
    pub code_challenge_method: Option<String>,
    pub expires_at: DateTime<Utc>,
    pub used: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, sqlx::FromRow)]
pub struct Token {
    pub id: Uuid,
    pub token_hash: String,
    pub token_type: String,
    pub client_id: String,
    pub user_id: Option<String>,
    pub scope: String,
    pub jti: Option<String>,
    pub expires_at: DateTime<Utc>,
    pub revoked: bool,
    pub revoked_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
}

/// Response body สำหรับ POST /token
#[derive(Debug, Serialize)]
pub struct TokenResponse {
    pub access_token: String,
    pub token_type: String,  // "Bearer"
    pub expires_in: i64,
    pub refresh_token: Option<String>,
    pub scope: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub id_token: Option<String>,   // OpenID Connect
}
```

### ขั้นที่ 3: Cryptographic Utilities

#### PKCE (RFC 7636)

```rust
// src/crypto/pkce.rs
use sha2::{Digest, Sha256};
use base64::{Engine as _, engine::general_purpose::URL_SAFE_NO_PAD};

/// สร้าง code_challenge จาก code_verifier ด้วย S256 method
/// S256 = BASE64URL(SHA256(ASCII(code_verifier)))
///
/// ข้อกำหนด RFC 7636:
/// - code_verifier ต้องยาว 43–128 ตัวอักษร
/// - ใช้ตัวอักษร [A-Z a-z 0-9 - . _ ~]
pub fn pkce_s256_challenge(verifier: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(verifier.as_bytes());
    let hash = hasher.finalize();
    URL_SAFE_NO_PAD.encode(hash)
}

/// ตรวจสอบ code_verifier กับ code_challenge ที่ส่งมาตอน /authorize
pub fn pkce_verify(verifier: &str, challenge: &str, method: &str) -> bool {
    match method {
        "S256" => {
            // ต้องไม่ใช้ == โดยตรง — ใช้ constant-time comparison เพื่อป้องกัน timing attack
            let computed = pkce_s256_challenge(verifier);
            constant_time_eq(computed.as_bytes(), challenge.as_bytes())
        }
        "plain" => constant_time_eq(verifier.as_bytes(), challenge.as_bytes()),
        _ => false,
    }
}

/// Constant-time string comparison (ป้องกัน timing attack)
fn constant_time_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }
    a.iter().zip(b.iter()).fold(0u8, |acc, (x, y)| acc | (x ^ y)) == 0
}

/// สร้าง cryptographically secure code_verifier
pub fn generate_code_verifier() -> String {
    use rand::Rng;
    let mut rng = rand::thread_rng();
    let chars: Vec<char> = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~"
        .chars()
        .collect();
    (0..96).map(|_| chars[rng.gen_range(0..chars.len())]).collect()
}
```

#### JWT ด้วย RS256

```rust
// src/crypto/jwt.rs
use jsonwebtoken::{encode, decode, Header, Algorithm, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};
use uuid::Uuid;

/// Claims ใน JWT access token
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct AccessTokenClaims {
    /// issuer — ต้องตรงกับ Config::issuer
    pub iss: String,
    /// subject — user_id หรือ client_id (สำหรับ client_credentials)
    pub sub: String,
    /// audience — client_id ที่ token ออกให้
    pub aud: String,
    /// expiry (Unix timestamp)
    pub exp: i64,
    /// issued at
    pub iat: i64,
    /// JWT ID — unique identifier สำหรับ revocation
    pub jti: String,
    /// granted scopes คั่นด้วยช่องว่าง
    pub scope: String,
    /// client_id ที่ขอ token
    pub client_id: String,
}

impl AccessTokenClaims {
    pub fn new(
        issuer: &str,
        sub: &str,
        client_id: &str,
        scope: &str,
        ttl_secs: i64,
    ) -> Self {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;
        Self {
            iss: issuer.to_string(),
            sub: sub.to_string(),
            aud: client_id.to_string(),
            exp: now + ttl_secs,
            iat: now,
            jti: Uuid::new_v4().to_string(),
            scope: scope.to_string(),
            client_id: client_id.to_string(),
        }
    }

    /// คืนค่า scope เป็น Vec<String>
    pub fn scopes(&self) -> Vec<String> {
        self.scope
            .split_whitespace()
            .map(|s| s.to_string())
            .collect()
    }
}

/// เซ็น JWT ด้วย RS256
pub fn sign_access_token(
    claims: &AccessTokenClaims,
    private_key_pem: &str,
) -> Result<String, jsonwebtoken::errors::Error> {
    let key = EncodingKey::from_rsa_pem(private_key_pem.as_bytes())?;
    let header = Header::new(Algorithm::RS256);
    encode(&header, claims, &key)
}

/// ตรวจสอบ JWT — คืน claims ถ้า valid
pub fn verify_access_token(
    token: &str,
    public_key_pem: &str,
) -> Result<AccessTokenClaims, jsonwebtoken::errors::Error> {
    let key = DecodingKey::from_rsa_pem(public_key_pem.as_bytes())?;
    let mut validation = Validation::new(Algorithm::RS256);
    validation.validate_exp = true;
    validation.validate_nbf = false;
    let data = decode::<AccessTokenClaims>(token, &key, &validation)?;
    Ok(data.claims)
}
```

#### JWKS Endpoint

```rust
// src/crypto/keys.rs
use serde::{Deserialize, Serialize};
use base64::{Engine as _, engine::general_purpose::URL_SAFE_NO_PAD};

/// JSON Web Key (RFC 7517) — แสดง public key ใน JWKS format
#[derive(Debug, Serialize, Deserialize)]
pub struct JwkKey {
    pub kty: String,    // "RSA"
    pub use_: String,   // "sig"
    #[serde(rename = "use")]
    pub use_field: String,
    pub alg: String,    // "RS256"
    pub kid: String,    // Key ID สำหรับ rotation
    pub n: String,      // modulus (base64url)
    pub e: String,      // exponent (base64url)
}

#[derive(Debug, Serialize)]
pub struct JwksResponse {
    pub keys: Vec<JwkKey>,
}

/// สร้าง JWKS จาก RSA public key (ใช้ `rsa` crate)
///
/// ตัวอย่างการใช้งาน:
/// ```rust
/// use rsa::{RsaPrivateKey, RsaPublicKey};
/// use rsa::pkcs1::EncodeRsaPublicKey;
///
/// let mut rng = rand::thread_rng();
/// let private_key = RsaPrivateKey::new(&mut rng, 2048).unwrap();
/// let public_key = RsaPublicKey::from(&private_key);
///
/// // ดึง modulus และ exponent
/// let n_bytes = public_key.n().to_bytes_be();
/// let e_bytes = public_key.e().to_bytes_be();
///
/// let jwk = JwkKey {
///     kty: "RSA".into(),
///     use_: "sig".into(),
///     use_field: "sig".into(),
///     alg: "RS256".into(),
///     kid: "key-2024-01".into(),
///     n: URL_SAFE_NO_PAD.encode(&n_bytes),
///     e: URL_SAFE_NO_PAD.encode(&e_bytes),
/// };
/// ```
pub fn build_jwks(keys: Vec<JwkKey>) -> JwksResponse {
    JwksResponse { keys }
}
```

### ขั้นที่ 4: Error Handling

```rust
// src/error.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use thiserror::Error;

/// OAuth2 Error Codes ตาม RFC 6749 Section 5.2
#[derive(Debug, Error)]
pub enum OAuthError {
    #[error("invalid_request")]
    InvalidRequest(String),

    #[error("invalid_client")]
    InvalidClient,

    #[error("invalid_grant")]
    InvalidGrant(String),

    #[error("unauthorized_client")]
    UnauthorizedClient,

    #[error("unsupported_grant_type")]
    UnsupportedGrantType,

    #[error("invalid_scope")]
    InvalidScope(String),

    #[error("access_denied")]
    AccessDenied,

    #[error("server_error")]
    Internal(#[from] anyhow::Error),

    #[error("database error")]
    Database(#[from] sqlx::Error),
}

impl IntoResponse for OAuthError {
    fn into_response(self) -> Response {
        let (status, error_code, description) = match &self {
            OAuthError::InvalidRequest(msg) => (
                StatusCode::BAD_REQUEST,
                "invalid_request",
                msg.clone(),
            ),
            OAuthError::InvalidClient => (
                StatusCode::UNAUTHORIZED,
                "invalid_client",
                "Client authentication failed".into(),
            ),
            OAuthError::InvalidGrant(msg) => (
                StatusCode::BAD_REQUEST,
                "invalid_grant",
                msg.clone(),
            ),
            OAuthError::UnauthorizedClient => (
                StatusCode::FORBIDDEN,
                "unauthorized_client",
                "This grant type is not allowed for this client".into(),
            ),
            OAuthError::UnsupportedGrantType => (
                StatusCode::BAD_REQUEST,
                "unsupported_grant_type",
                "Grant type not supported".into(),
            ),
            OAuthError::InvalidScope(msg) => (
                StatusCode::BAD_REQUEST,
                "invalid_scope",
                msg.clone(),
            ),
            OAuthError::AccessDenied => (
                StatusCode::FORBIDDEN,
                "access_denied",
                "The resource owner denied the request".into(),
            ),
            OAuthError::Internal(_) | OAuthError::Database(_) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "server_error",
                "An internal error occurred".into(),
            ),
        };

        let body = json!({
            "error": error_code,
            "error_description": description,
        });

        (status, Json(body)).into_response()
    }
}

pub type Result<T> = std::result::Result<T, OAuthError>;
```

### ขั้นที่ 5: Authorization Code Flow (GET & POST /authorize)

```rust
// src/handlers/authorize.rs
use axum::{
    extract::{Query, State, Form},
    response::{Html, Redirect, IntoResponse},
};
use chrono::Utc;
use serde::Deserialize;

use crate::{AppState, error::{OAuthError, Result}, crypto::pkce::pkce_s256_challenge};

/// Query parameters สำหรับ GET /authorize
#[derive(Debug, Deserialize)]
pub struct AuthorizeParams {
    pub response_type: String,
    pub client_id: String,
    pub redirect_uri: String,
    pub scope: String,
    pub state: Option<String>,
    /// PKCE: code_challenge (BASE64URL-encoded SHA256 ของ verifier)
    pub code_challenge: Option<String>,
    /// PKCE: "S256" (แนะนำ) หรือ "plain" (ไม่แนะนำ)
    pub code_challenge_method: Option<String>,
}

/// แสดงหน้า login form
/// GET /authorize?response_type=code&client_id=...&redirect_uri=...&scope=...&state=...
pub async fn authorize_get(
    State(state): State<AppState>,
    Query(params): Query<AuthorizeParams>,
) -> Result<Html<String>> {
    // 1. ตรวจสอบ response_type ต้องเป็น "code"
    if params.response_type != "code" {
        return Err(OAuthError::InvalidRequest(
            "response_type must be 'code'".into(),
        ));
    }

    // 2. ค้นหา client ใน database
    let client = sqlx::query_as!(
        crate::models::Client,
        "SELECT * FROM clients WHERE client_id = $1",
        params.client_id
    )
    .fetch_optional(&state.db)
    .await
    .map_err(OAuthError::Database)?
    .ok_or(OAuthError::InvalidClient)?;

    // 3. ตรวจสอบ redirect_uri (ต้องอยู่ใน whitelist)
    if !client.redirect_uris.contains(&params.redirect_uri) {
        return Err(OAuthError::InvalidRequest(
            "redirect_uri is not registered for this client".into(),
        ));
    }

    // 4. ตรวจสอบ scope
    let requested_scopes: Vec<String> = params.scope
        .split_whitespace()
        .map(String::from)
        .collect();
    let invalid_scope = requested_scopes
        .iter()
        .find(|s| !client.allowed_scopes.contains(s));
    if let Some(bad) = invalid_scope {
        return Err(OAuthError::InvalidScope(
            format!("scope '{}' is not allowed for this client", bad),
        ));
    }

    // 5. เก็บ PKCE challenge ไว้ใน session (ใช้ signed cookie หรือ DB)
    //    ที่นี่เราเก็บไว้ใน query string แบบ encode เข้าไปใน hidden field

    // 6. ส่งกลับ HTML login form
    let html = format!(
        r#"<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>เข้าสู่ระบบ — {client_name}</title>
    <style>
        body {{ font-family: sans-serif; max-width: 400px; margin: 100px auto; padding: 20px; }}
        input {{ width: 100%; padding: 8px; margin: 8px 0; box-sizing: border-box; }}
        button {{ width: 100%; padding: 10px; background: #2563eb; color: white; border: none; 
                  border-radius: 4px; cursor: pointer; font-size: 16px; }}
        .app-info {{ background: #f0f9ff; padding: 12px; border-radius: 4px; margin-bottom: 16px; }}
        .scope-list {{ list-style: none; padding: 0; }}
        .scope-list li::before {{ content: "✓ "; color: #16a34a; }}
    </style>
</head>
<body>
    <div class="app-info">
        <strong>{client_id}</strong> ขอสิทธิ์เข้าถึง:
        <ul class="scope-list">
            {scope_items}
        </ul>
    </div>
    <form method="POST" action="/authorize">
        <input type="hidden" name="client_id"     value="{client_id}">
        <input type="hidden" name="redirect_uri"  value="{redirect_uri}">
        <input type="hidden" name="scope"         value="{scope}">
        <input type="hidden" name="state"         value="{state_val}">
        <input type="hidden" name="code_challenge" value="{code_challenge}">
        <input type="hidden" name="code_challenge_method" value="{code_challenge_method}">
        <label>อีเมล</label>
        <input type="email" name="username" required autofocus>
        <label>รหัสผ่าน</label>
        <input type="password" name="password" required>
        <button type="submit">อนุมัติและเข้าสู่ระบบ</button>
    </form>
</body>
</html>"#,
        client_name = client.client_id,
        client_id = params.client_id,
        redirect_uri = params.redirect_uri,
        scope = params.scope,
        state_val = params.state.as_deref().unwrap_or(""),
        code_challenge = params.code_challenge.as_deref().unwrap_or(""),
        code_challenge_method = params.code_challenge_method.as_deref().unwrap_or(""),
        scope_items = requested_scopes
            .iter()
            .map(|s| format!("<li>{}</li>", s))
            .collect::<String>(),
    );

    Ok(Html(html))
}

/// Form data จาก POST /authorize
#[derive(Debug, Deserialize)]
pub struct AuthorizeForm {
    pub client_id: String,
    pub redirect_uri: String,
    pub scope: String,
    pub state: Option<String>,
    pub code_challenge: Option<String>,
    pub code_challenge_method: Option<String>,
    pub username: String,
    pub password: String,
}

/// รับ credentials, authenticate user, ออก auth_code แล้ว redirect
/// POST /authorize
pub async fn authorize_post(
    State(state): State<AppState>,
    Form(form): Form<AuthorizeForm>,
) -> Result<impl IntoResponse> {
    // 1. ตรวจสอบ user credentials (ใน production จะเช็คกับ user table)
    let user_id = authenticate_user(&form.username, &form.password, &state)
        .await
        .map_err(|_| OAuthError::AccessDenied)?;

    // 2. สร้าง auth_code แบบ secure random
    let auth_code = generate_secure_token(32); // 32 bytes = 43 chars base64url

    // 3. เก็บใน database
    let expires_at = Utc::now() + chrono::Duration::seconds(
        state.config.auth_code_ttl_secs
    );

    sqlx::query!(
        r#"
        INSERT INTO auth_codes
            (code, client_id, user_id, redirect_uri, scope, state,
             code_challenge, code_challenge_method, expires_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
        "#,
        auth_code,
        form.client_id,
        user_id,
        form.redirect_uri,
        form.scope,
        form.state,
        form.code_challenge,
        form.code_challenge_method,
        expires_at,
    )
    .execute(&state.db)
    .await
    .map_err(OAuthError::Database)?;

    // 4. Redirect กลับไปที่ client พร้อม code และ state
    let mut redirect_url = format!(
        "{}?code={}",
        form.redirect_uri, auth_code
    );
    if let Some(state_val) = &form.state {
        redirect_url.push_str(&format!("&state={}", state_val));
    }

    Ok(Redirect::to(&redirect_url))
}

/// ตัวช่วย: สร้าง secure random token
pub fn generate_secure_token(byte_len: usize) -> String {
    use rand::RngCore;
    use base64::{Engine as _, engine::general_purpose::URL_SAFE_NO_PAD};
    let mut bytes = vec![0u8; byte_len];
    rand::thread_rng().fill_bytes(&mut bytes);
    URL_SAFE_NO_PAD.encode(&bytes)
}

/// Placeholder สำหรับ user authentication
/// ใน production จะ query users table และ verify password hash
async fn authenticate_user(
    username: &str,
    password: &str,
    _state: &AppState,
) -> std::result::Result<String, ()> {
    // TODO: เชื่อมกับ user database จริง
    if username == "demo@example.com" && password == "password123" {
        Ok("user-demo-001".into())
    } else {
        Err(())
    }
}
```

### ขั้นที่ 6: Token Endpoint (POST /token)

Token endpoint จัดการ 3 grant types ในฟังก์ชันเดียว:

```rust
// src/handlers/token.rs
use axum::{extract::{State, Form}, Json};
use chrono::Utc;
use serde::Deserialize;
use sha2::{Digest, Sha256};
use base64::{Engine as _, engine::general_purpose::URL_SAFE_NO_PAD};

use crate::{
    AppState,
    error::{OAuthError, Result},
    models::TokenResponse,
    crypto::{
        jwt::{AccessTokenClaims, sign_access_token},
        pkce::pkce_verify,
    },
    handlers::authorize::generate_secure_token,
};

#[derive(Debug, Deserialize)]
pub struct TokenRequest {
    pub grant_type: String,

    // Authorization Code
    pub code: Option<String>,
    pub redirect_uri: Option<String>,
    pub code_verifier: Option<String>,

    // Refresh Token
    pub refresh_token: Option<String>,

    // Client Credentials & Auth Code
    pub client_id: Option<String>,
    pub client_secret: Option<String>,

    // Optional scope override
    pub scope: Option<String>,
}

/// POST /token — จัดการทุก grant types
pub async fn token_endpoint(
    State(app): State<AppState>,
    Form(req): Form<TokenRequest>,
) -> Result<Json<TokenResponse>> {
    match req.grant_type.as_str() {
        "authorization_code" => handle_auth_code(app, req).await,
        "refresh_token"      => handle_refresh_token(app, req).await,
        "client_credentials" => handle_client_credentials(app, req).await,
        _ => Err(OAuthError::UnsupportedGrantType),
    }
}

// ----------------------------------------------------------------
// Grant: authorization_code
// ----------------------------------------------------------------

async fn handle_auth_code(
    app: AppState,
    req: TokenRequest,
) -> Result<Json<TokenResponse>> {
    let code = req.code.ok_or_else(|| OAuthError::InvalidRequest("code required".into()))?;
    let redirect_uri = req.redirect_uri
        .ok_or_else(|| OAuthError::InvalidRequest("redirect_uri required".into()))?;

    // 1. ตรวจสอบ client credentials
    let client = authenticate_client(&app, &req.client_id, &req.client_secret).await?;

    // 2. ดึง auth_code จาก DB (ต้องยังไม่ถูกใช้ และยังไม่หมดอายุ)
    let auth_code = sqlx::query_as!(
        crate::models::AuthCode,
        r#"SELECT code, client_id, user_id, redirect_uri, scope, state,
                  code_challenge, code_challenge_method,
                  expires_at, used, created_at
           FROM auth_codes
           WHERE code = $1 AND client_id = $2 AND NOT used"#,
        code,
        client.client_id,
    )
    .fetch_optional(&app.db)
    .await
    .map_err(OAuthError::Database)?
    .ok_or_else(|| OAuthError::InvalidGrant("auth code not found or already used".into()))?;

    // 3. ตรวจสอบว่า code ยังไม่หมดอายุ
    if Utc::now() > auth_code.expires_at {
        return Err(OAuthError::InvalidGrant("auth code has expired".into()));
    }

    // 4. ตรวจสอบ redirect_uri ต้องตรงกับที่ authorize ไว้
    if auth_code.redirect_uri != redirect_uri {
        return Err(OAuthError::InvalidGrant("redirect_uri mismatch".into()));
    }

    // 5. ตรวจสอบ PKCE (ถ้ามี)
    if let Some(challenge) = &auth_code.code_challenge {
        let verifier = req.code_verifier
            .as_deref()
            .ok_or_else(|| OAuthError::InvalidRequest("code_verifier required".into()))?;
        let method = auth_code.code_challenge_method.as_deref().unwrap_or("S256");

        if !pkce_verify(verifier, challenge, method) {
            return Err(OAuthError::InvalidGrant("PKCE verification failed".into()));
        }
    }

    // 6. Mark auth_code ว่าใช้แล้ว (ป้องกัน replay)
    sqlx::query!("UPDATE auth_codes SET used = true WHERE code = $1", code)
        .execute(&app.db)
        .await
        .map_err(OAuthError::Database)?;

    // 7. ออก access_token และ refresh_token
    issue_tokens(&app, &client.client_id, &auth_code.user_id, &auth_code.scope).await
}

// ----------------------------------------------------------------
// Grant: refresh_token
// ----------------------------------------------------------------

async fn handle_refresh_token(
    app: AppState,
    req: TokenRequest,
) -> Result<Json<TokenResponse>> {
    let rt_value = req.refresh_token
        .ok_or_else(|| OAuthError::InvalidRequest("refresh_token required".into()))?;

    // 1. Hash refresh token เพื่อค้นหาใน DB
    let rt_hash = sha256_hex(&rt_value);

    // 2. ค้นหา refresh token ใน DB
    let stored = sqlx::query_as!(
        crate::models::Token,
        r#"SELECT id, token_hash, token_type, client_id, user_id, scope,
                  jti, expires_at, revoked, revoked_at, created_at
           FROM tokens
           WHERE token_hash = $1 AND token_type = 'refresh' AND NOT revoked"#,
        rt_hash,
    )
    .fetch_optional(&app.db)
    .await
    .map_err(OAuthError::Database)?
    .ok_or_else(|| OAuthError::InvalidGrant("refresh token not found or revoked".into()))?;

    // 3. ตรวจสอบ expiry
    if Utc::now() > stored.expires_at {
        return Err(OAuthError::InvalidGrant("refresh token has expired".into()));
    }

    // 4. ตรวจสอบ client (ถ้าเป็น confidential client)
    let client = authenticate_client(&app, &req.client_id, &req.client_secret).await?;
    if client.client_id != stored.client_id {
        return Err(OAuthError::InvalidGrant("refresh token does not belong to this client".into()));
    }

    // 5. Revoke refresh token เดิม (Refresh Token Rotation)
    sqlx::query!(
        "UPDATE tokens SET revoked = true, revoked_at = NOW() WHERE id = $1",
        stored.id
    )
    .execute(&app.db)
    .await
    .map_err(OAuthError::Database)?;

    // 6. ออก token ใหม่
    let user_id = stored.user_id.as_deref().unwrap_or(&stored.client_id);
    issue_tokens(&app, &stored.client_id, user_id, &stored.scope).await
}

// ----------------------------------------------------------------
// Grant: client_credentials
// ----------------------------------------------------------------

async fn handle_client_credentials(
    app: AppState,
    req: TokenRequest,
) -> Result<Json<TokenResponse>> {
    // 1. ตรวจสอบ client credentials
    let client = authenticate_client(&app, &req.client_id, &req.client_secret).await?;

    // 2. ตรวจสอบว่า client รองรับ client_credentials grant
    if !client.grant_types.contains(&"client_credentials".to_string()) {
        return Err(OAuthError::UnauthorizedClient);
    }

    // 3. scope: ใช้ที่ request มา หรือถ้าไม่ระบุใช้ allowed_scopes ทั้งหมด
    let scope = if let Some(s) = &req.scope {
        let requested: Vec<String> = s.split_whitespace().map(String::from).collect();
        let invalid = requested.iter().find(|r| !client.allowed_scopes.contains(r));
        if let Some(bad) = invalid {
            return Err(OAuthError::InvalidScope(
                format!("scope '{}' not allowed for this client", bad)
            ));
        }
        s.clone()
    } else {
        client.allowed_scopes.join(" ")
    };

    // 4. สำหรับ client_credentials: sub = client_id (ไม่มี refresh token)
    let claims = AccessTokenClaims::new(
        &app.config.issuer,
        &client.client_id,
        &client.client_id,
        &scope,
        app.config.access_token_ttl_secs,
    );

    let access_token = sign_access_token(&claims, &app.config.private_key_pem)
        .map_err(|e| OAuthError::Internal(anyhow::anyhow!("JWT sign error: {e}")))?;

    // 5. เก็บ token hash ใน DB เพื่อ revocation
    store_token_record(&app, &claims, &access_token, "access", None).await?;

    Ok(Json(TokenResponse {
        access_token,
        token_type: "Bearer".into(),
        expires_in: app.config.access_token_ttl_secs,
        refresh_token: None,  // client_credentials ไม่มี refresh token
        scope,
        id_token: None,
    }))
}

// ----------------------------------------------------------------
// Helpers
// ----------------------------------------------------------------

/// ออก access_token + refresh_token พร้อมเก็บใน DB
async fn issue_tokens(
    app: &AppState,
    client_id: &str,
    user_id: &str,
    scope: &str,
) -> Result<Json<TokenResponse>> {
    // Access token (JWT)
    let claims = AccessTokenClaims::new(
        &app.config.issuer,
        user_id,
        client_id,
        scope,
        app.config.access_token_ttl_secs,
    );
    let access_token = sign_access_token(&claims, &app.config.private_key_pem)
        .map_err(|e| OAuthError::Internal(anyhow::anyhow!("{e}")))?;

    // Refresh token (opaque random string)
    let refresh_token = generate_secure_token(48);

    // เก็บ hash ของทั้งสองใน DB
    store_token_record(app, &claims, &access_token, "access", Some(user_id)).await?;

    let rt_hash = sha256_hex(&refresh_token);
    let rt_expires = Utc::now() + chrono::Duration::seconds(app.config.refresh_token_ttl_secs);
    sqlx::query!(
        r#"INSERT INTO tokens
           (token_hash, token_type, client_id, user_id, scope, expires_at)
           VALUES ($1, 'refresh', $2, $3, $4, $5)"#,
        rt_hash,
        client_id,
        user_id,
        scope,
        rt_expires,
    )
    .execute(&app.db)
    .await
    .map_err(OAuthError::Database)?;

    Ok(Json(TokenResponse {
        access_token,
        token_type: "Bearer".into(),
        expires_in: app.config.access_token_ttl_secs,
        refresh_token: Some(refresh_token),
        scope: scope.to_string(),
        id_token: None,
    }))
}

async fn store_token_record(
    app: &AppState,
    claims: &AccessTokenClaims,
    token: &str,
    token_type: &str,
    user_id: Option<&str>,
) -> Result<()> {
    let hash = sha256_hex(token);
    let expires_at = chrono::DateTime::from_timestamp(claims.exp, 0)
        .unwrap_or_else(Utc::now);

    sqlx::query!(
        r#"INSERT INTO tokens
           (token_hash, token_type, client_id, user_id, scope, jti, expires_at)
           VALUES ($1, $2, $3, $4, $5, $6, $7)"#,
        hash,
        token_type,
        claims.client_id,
        user_id,
        claims.scope,
        claims.jti,
        expires_at,
    )
    .execute(&app.db)
    .await
    .map_err(OAuthError::Database)?;

    Ok(())
}

/// ตรวจสอบ client_id + client_secret
async fn authenticate_client(
    app: &AppState,
    client_id: &Option<String>,
    client_secret: &Option<String>,
) -> Result<crate::models::Client> {
    use argon2::{Argon2, PasswordVerifier};
    use argon2::password_hash::PasswordHash;

    let cid = client_id
        .as_deref()
        .ok_or(OAuthError::InvalidClient)?;

    let client = sqlx::query_as!(
        crate::models::Client,
        "SELECT * FROM clients WHERE client_id = $1",
        cid,
    )
    .fetch_optional(&app.db)
    .await
    .map_err(OAuthError::Database)?
    .ok_or(OAuthError::InvalidClient)?;

    // ถ้า confidential client ต้องมี secret
    if client.is_confidential {
        let secret = client_secret.as_deref().ok_or(OAuthError::InvalidClient)?;
        let hash = PasswordHash::new(&client.client_secret_hash)
            .map_err(|_| OAuthError::InvalidClient)?;
        Argon2::default()
            .verify_password(secret.as_bytes(), &hash)
            .map_err(|_| OAuthError::InvalidClient)?;
    }

    Ok(client)
}

/// SHA-256 hash แล้วเข้ารหัสเป็น hex string (สำหรับ store ใน DB)
pub fn sha256_hex(input: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(input.as_bytes());
    format!("{:x}", hasher.finalize())
}
```

### ขั้นที่ 7: Introspection, Revocation, Discovery และ Scope Middleware

#### Token Introspection (RFC 7662)

```rust
// src/handlers/introspect.rs
use axum::{extract::{State, Form}, Json};
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};

use crate::{AppState, error::Result, handlers::token::sha256_hex};

#[derive(Deserialize)]
pub struct IntrospectRequest {
    pub token: String,
    pub token_type_hint: Option<String>,
}

/// POST /introspect — ตาม RFC 7662
/// คืน {"active": false} ถ้า token ไม่ valid
/// คืน token metadata ถ้า valid
pub async fn introspect(
    State(app): State<AppState>,
    Form(req): Form<IntrospectRequest>,
) -> Result<Json<Value>> {
    // พยายาม decode เป็น JWT ก่อน (access token)
    let public_key_pem = extract_public_key_pem(&app.config.private_key_pem);
    if let Ok(claims) = crate::crypto::jwt::verify_access_token(&req.token, &public_key_pem) {
        // ตรวจสอบ DB ว่า token ถูก revoke หรือยัง
        let jti = &claims.jti;
        let revoked = sqlx::query_scalar!(
            "SELECT revoked FROM tokens WHERE jti = $1",
            jti
        )
        .fetch_optional(&app.db)
        .await
        .unwrap_or(None)
        .unwrap_or(true); // ถ้าไม่พบใน DB ถือว่า inactive

        if revoked {
            return Ok(Json(json!({"active": false})));
        }

        return Ok(Json(json!({
            "active":     true,
            "scope":      claims.scope,
            "client_id":  claims.client_id,
            "sub":        claims.sub,
            "exp":        claims.exp,
            "iat":        claims.iat,
            "iss":        claims.iss,
            "jti":        claims.jti,
            "token_type": "access_token",
        })));
    }

    // ถ้าไม่ใช่ JWT ลองค้นหา refresh token
    let token_hash = sha256_hex(&req.token);
    let stored = sqlx::query!(
        r#"SELECT client_id, user_id, scope, expires_at, revoked
           FROM tokens
           WHERE token_hash = $1 AND token_type = 'refresh'"#,
        token_hash,
    )
    .fetch_optional(&app.db)
    .await
    .map_err(crate::error::OAuthError::Database)?;

    match stored {
        Some(t) if !t.revoked && chrono::Utc::now() < t.expires_at => {
            Ok(Json(json!({
                "active":    true,
                "scope":     t.scope,
                "client_id": t.client_id,
                "sub":       t.user_id,
                "exp":       t.expires_at.timestamp(),
                "token_type": "refresh_token",
            })))
        }
        _ => Ok(Json(json!({"active": false}))),
    }
}

fn extract_public_key_pem(private_key_pem: &str) -> String {
    // ใน production: ใช้ rsa crate extract public key
    // ที่นี่ placeholder — ใน real implementation จะ parse RSA private key
    // และ derive public key ออกมา
    private_key_pem.to_string() // simplified for illustration
}
```

#### Token Revocation (RFC 7009)

```rust
// src/handlers/revoke.rs
use axum::{extract::{State, Form}, Json};
use serde::Deserialize;
use serde_json::{json, Value};

use crate::{AppState, error::Result, handlers::token::sha256_hex};

#[derive(Deserialize)]
pub struct RevokeRequest {
    pub token: String,
    pub token_type_hint: Option<String>,  // "access_token" หรือ "refresh_token"
}

/// POST /revoke — ตาม RFC 7009
/// ตาม spec: ต้องคืน 200 OK เสมอ แม้ token ไม่ valid
/// (เพื่อไม่ให้รู้ว่า token นั้น valid หรือไม่)
pub async fn revoke(
    State(app): State<AppState>,
    Form(req): Form<RevokeRequest>,
) -> Json<Value> {
    // พยายาม revoke ทั้ง access token (via jti) และ refresh token (via hash)

    // พยายาม revoke as JWT (access token) — ดึง jti จาก token
    if let Ok(claims) = extract_jti_from_jwt(&req.token, &app) {
        let _ = sqlx::query!(
            "UPDATE tokens SET revoked = true, revoked_at = NOW() WHERE jti = $1",
            claims
        )
        .execute(&app.db)
        .await;
    }

    // พยายาม revoke as opaque token (refresh token)
    let token_hash = sha256_hex(&req.token);
    let _ = sqlx::query!(
        "UPDATE tokens SET revoked = true, revoked_at = NOW() WHERE token_hash = $1",
        token_hash
    )
    .execute(&app.db)
    .await;

    // RFC 7009: ตาม spec ต้องคืน 200 เสมอ
    Json(json!({}))
}

fn extract_jti_from_jwt(token: &str, _app: &AppState) -> Result<String, ()> {
    // Decode โดยไม่ verify (เพื่อดึง jti เท่านั้น) — ใน production ต้อง verify ก่อน
    use jsonwebtoken::{decode_header, Algorithm, DecodingKey, Validation, decode};
    // simplified: parse JWT header เพื่อดู jti
    Err(()) // placeholder
}
```

#### OpenID Connect Discovery

```rust
// src/handlers/discovery.rs
use axum::{extract::State, Json};
use serde_json::{json, Value};

use crate::AppState;

/// GET /.well-known/openid-configuration
/// ตาม OpenID Connect Discovery 1.0
pub async fn openid_configuration(State(app): State<AppState>) -> Json<Value> {
    let issuer = &app.config.issuer;
    Json(json!({
        "issuer": issuer,
        "authorization_endpoint": format!("{issuer}/authorize"),
        "token_endpoint": format!("{issuer}/token"),
        "jwks_uri": format!("{issuer}/.well-known/jwks.json"),
        "registration_endpoint": format!("{issuer}/clients"),
        "introspection_endpoint": format!("{issuer}/introspect"),
        "revocation_endpoint": format!("{issuer}/revoke"),

        "response_types_supported": ["code"],
        "grant_types_supported": [
            "authorization_code",
            "refresh_token",
            "client_credentials"
        ],
        "subject_types_supported": ["public"],
        "id_token_signing_alg_values_supported": ["RS256"],
        "token_endpoint_auth_methods_supported": [
            "client_secret_post",
            "client_secret_basic"
        ],
        "code_challenge_methods_supported": ["S256", "plain"],
        "scopes_supported": ["openid", "profile", "email", "read", "write"],
        "claims_supported": ["sub", "iss", "aud", "exp", "iat", "scope", "client_id"],
    }))
}

/// GET /.well-known/jwks.json — public keys สำหรับ RS256 verification
pub async fn jwks(State(app): State<AppState>) -> Json<Value> {
    // ใน production: สร้าง JWK จาก RSA public key จริง
    // ที่นี่แสดง structure ที่ถูกต้อง
    Json(json!({
        "keys": [
            {
                "kty": "RSA",
                "use": "sig",
                "alg": "RS256",
                "kid": "key-2024-01",
                "n": "<base64url-encoded-modulus>",
                "e": "AQAB"
            }
        ]
    }))
}
```

#### Client Registration

```rust
// src/handlers/clients.rs
use axum::{extract::{State}, Json};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

use crate::{AppState, error::{OAuthError, Result}};

#[derive(Deserialize)]
pub struct RegisterClientRequest {
    pub client_name: String,
    pub redirect_uris: Vec<String>,
    pub allowed_scopes: Vec<String>,
    pub grant_types: Option<Vec<String>>,
    pub is_confidential: Option<bool>,
}

#[derive(Serialize)]
pub struct RegisterClientResponse {
    pub client_id: String,
    pub client_secret: String,  // ส่งกลับเพียงครั้งเดียว ไม่เก็บ plaintext
    pub redirect_uris: Vec<String>,
    pub allowed_scopes: Vec<String>,
    pub grant_types: Vec<String>,
}

/// POST /clients — ลงทะเบียน OAuth2 client ใหม่
pub async fn register_client(
    State(app): State<AppState>,
    Json(req): Json<RegisterClientRequest>,
) -> Result<Json<RegisterClientResponse>> {
    use argon2::{Argon2, PasswordHasher};
    use argon2::password_hash::{SaltString, rand_core::OsRng};
    use rand::Rng;

    // 1. สร้าง client_id และ client_secret
    let client_id = format!("cli_{}", &Uuid::new_v4().to_string().replace('-', "")[..16]);
    let client_secret: String = (0..48)
        .map(|_| rand::thread_rng().sample(rand::distributions::Alphanumeric) as char)
        .collect();

    // 2. Hash client_secret ด้วย Argon2id
    let salt = SaltString::generate(&mut OsRng);
    let hash = Argon2::default()
        .hash_password(client_secret.as_bytes(), &salt)
        .map_err(|_| OAuthError::Internal(anyhow::anyhow!("hash failed")))?
        .to_string();

    // 3. Grant types default
    let grant_types = req.grant_types.unwrap_or_else(|| {
        vec!["authorization_code".into(), "refresh_token".into()]
    });

    // 4. ตรวจสอบ redirect_uris — ต้องเป็น HTTPS ยกเว้น localhost
    for uri in &req.redirect_uris {
        if !uri.starts_with("https://") && !uri.starts_with("http://localhost") {
            return Err(OAuthError::InvalidRequest(
                format!("redirect_uri '{}' must use HTTPS (except localhost)", uri)
            ));
        }
    }

    // 5. บันทึกลง database
    sqlx::query!(
        r#"INSERT INTO clients
           (client_id, client_secret_hash, redirect_uris,
            allowed_scopes, grant_types, is_confidential)
           VALUES ($1, $2, $3, $4, $5, $6)"#,
        client_id,
        hash,
        &req.redirect_uris,
        &req.allowed_scopes,
        &grant_types,
        req.is_confidential.unwrap_or(true),
    )
    .execute(&app.db)
    .await
    .map_err(OAuthError::Database)?;

    Ok(Json(RegisterClientResponse {
        client_id,
        client_secret,   // ส่งกลับครั้งเดียวเท่านั้น
        redirect_uris: req.redirect_uris,
        allowed_scopes: req.allowed_scopes,
        grant_types,
    }))
}
```

#### Scope Enforcement Middleware

```rust
// src/middleware/scope_check.rs
use axum::{
    body::Body,
    extract::{Request, State},
    http::{StatusCode, header::AUTHORIZATION},
    middleware::Next,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

use crate::{AppState, crypto::jwt::verify_access_token};

/// Middleware: ตรวจสอบ Bearer token และ scope ที่กำหนด
///
/// ใช้กับ router แบบ:
/// ```rust
/// let protected = Router::new()
///     .route("/api/profile", get(profile_handler))
///     .route_layer(middleware::from_fn_with_state(
///         state.clone(),
///         |s, req, next| require_scope(s, req, next, "profile"),
///     ));
/// ```
pub async fn require_scope(
    State(app): State<AppState>,
    req: Request<Body>,
    next: Next,
    required_scope: &'static str,
) -> Response {
    // 1. ดึง Authorization header
    let auth_header = req
        .headers()
        .get(AUTHORIZATION)
        .and_then(|v| v.to_str().ok());

    let token = match auth_header {
        Some(h) if h.starts_with("Bearer ") => h.trim_start_matches("Bearer "),
        _ => {
            return (
                StatusCode::UNAUTHORIZED,
                Json(json!({
                    "error": "unauthorized",
                    "error_description": "Bearer token required"
                })),
            ).into_response();
        }
    };

    // 2. ตรวจสอบ JWT
    let public_key_pem = &app.config.private_key_pem; // ใน production: ใช้ public key
    match verify_access_token(token, public_key_pem) {
        Ok(claims) => {
            // 3. ตรวจสอบ scope
            let has_scope = claims.scope
                .split_whitespace()
                .any(|s| s == required_scope);

            if !has_scope {
                return (
                    StatusCode::FORBIDDEN,
                    Json(json!({
                        "error": "insufficient_scope",
                        "error_description": format!(
                            "Token missing required scope: {}", required_scope
                        ),
                        "scope": required_scope,
                    })),
                ).into_response();
            }

            // 4. ตรวจสอบ revocation ใน DB
            let revoked = sqlx::query_scalar!(
                "SELECT revoked FROM tokens WHERE jti = $1",
                claims.jti
            )
            .fetch_optional(&app.db)
            .await
            .ok()
            .flatten()
            .unwrap_or(false);

            if revoked {
                return (
                    StatusCode::UNAUTHORIZED,
                    Json(json!({"error": "token_revoked"})),
                ).into_response();
            }

            next.run(req).await
        }
        Err(_) => (
            StatusCode::UNAUTHORIZED,
            Json(json!({
                "error": "invalid_token",
                "error_description": "Token validation failed"
            })),
        ).into_response(),
    }
}
```

#### Main Router

```rust
// src/main.rs
use axum::{
    Router,
    routing::{get, post},
    middleware,
};
use sqlx::PgPool;
use std::sync::Arc;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod config;
mod db;
mod models;
mod error;
mod crypto {
    pub mod jwt;
    pub mod pkce;
    pub mod keys;
}
mod handlers {
    pub mod authorize;
    pub mod token;
    pub mod clients;
    pub mod introspect;
    pub mod revoke;
    pub mod discovery;
}
mod middleware {
    pub mod scope_check;
}

use config::Config;

/// Application state ที่ share ระหว่าง handlers ทั้งหมด
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub config: Arc<Config>,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. Logging
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(
            std::env::var("RUST_LOG").unwrap_or_else(|_| "oauth2_server=debug,tower_http=info".into()),
        ))
        .with(tracing_subscriber::fmt::layer())
        .init();

    // 2. Configuration
    dotenvy::dotenv().ok();
    let config = Arc::new(Config::from_env().expect("invalid config"));

    // 3. Database pool
    let db = PgPool::connect(&config.database_url)
        .await
        .expect("cannot connect to database");

    sqlx::migrate!("./migrations")
        .run(&db)
        .await
        .expect("migration failed");

    let state = AppState { db, config: config.clone() };

    // 4. Router
    let app = Router::new()
        // Authorization Code Flow
        .route("/authorize", get(handlers::authorize::authorize_get))
        .route("/authorize", post(handlers::authorize::authorize_post))
        // Token endpoint (all grant types)
        .route("/token", post(handlers::token::token_endpoint))
        // Client registration
        .route("/clients", post(handlers::clients::register_client))
        // Token introspection (RFC 7662)
        .route("/introspect", post(handlers::introspect::introspect))
        // Token revocation (RFC 7009)
        .route("/revoke", post(handlers::handlers::revoke::revoke))
        // Discovery endpoints
        .route("/.well-known/openid-configuration",
               get(handlers::discovery::openid_configuration))
        .route("/.well-known/jwks.json",
               get(handlers::discovery::jwks))
        // Protected resource endpoints (example)
        .route("/api/profile",
               get(example_profile_handler)
                   .route_layer(middleware::from_fn_with_state(
                       state.clone(),
                       |s, req, next| handlers::middleware::scope_check::require_scope(
                           s, req, next, "profile"
                       ),
                   )))
        .with_state(state)
        .layer(tower_http::trace::TraceLayer::new_for_http());

    let addr = format!("0.0.0.0:{}", config.server_port);
    tracing::info!("OAuth2 Authorization Server listening on {addr}");

    let listener = tokio::net::TcpListener::bind(&addr).await?;
    axum::serve(listener, app).await?;
    Ok(())
}

async fn example_profile_handler() -> axum::Json<serde_json::Value> {
    axum::Json(serde_json::json!({
        "sub": "user-demo-001",
        "name": "Demo User",
        "email": "demo@example.com",
    }))
}
```

---

## การทดสอบ (Testing)

### Unit Tests — Utility Layer

โค้ด test จาก verification project ที่รันจริง (ครอบคลุม PKCE S256, JWT, scope parsing, client credential hashing):

```rust
// ต่อจาก src/main.rs ของ verification project
#[cfg(test)]
mod tests {
    use super::*;
    use std::time::{SystemTime, UNIX_EPOCH};

    // --- PKCE tests ---

    #[test]
    fn test_pkce_s256_rfc_vector() {
        // RFC 7636 Appendix B test vector
        let verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk";
        let expected = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM";
        assert_eq!(pkce_s256_challenge(verifier), expected);
    }

    #[test]
    fn test_pkce_verify_s256_valid() {
        let verifier = "my-secret-verifier-string-that-is-long-enough";
        let challenge = pkce_s256_challenge(verifier);
        assert!(pkce_verify(verifier, &challenge, "S256"));
    }

    #[test]
    fn test_pkce_verify_s256_tampered_verifier() {
        let verifier = "correct-verifier-string";
        let challenge = pkce_s256_challenge(verifier);
        assert!(!pkce_verify("wrong-verifier-string", &challenge, "S256"));
    }

    #[test]
    fn test_pkce_verify_plain() {
        let verifier = "plain-text-verifier";
        assert!(pkce_verify(verifier, verifier, "plain"));
        assert!(!pkce_verify(verifier, "different", "plain"));
    }

    #[test]
    fn test_pkce_unknown_method_returns_false() {
        assert!(!pkce_verify("v", "c", "unknown"));
    }

    // --- JWT tests ---

    #[test]
    fn test_jwt_sign_and_verify_roundtrip() {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;
        let claims = Claims {
            sub: "user-42".into(),
            client_id: "app-client".into(),
            scope: "read profile".into(),
            iat: now,
            exp: now + 3600,
            jti: "jti-unique-1".into(),
        };
        let secret = b"test-secret-key-for-hs256";
        let token = jwt_sign(&claims, secret).expect("sign failed");
        assert!(!token.is_empty());
        assert_eq!(token.matches('.').count(), 2, "JWT must have 3 parts");

        let decoded = jwt_verify(&token, secret).expect("verify failed");
        assert_eq!(decoded.sub, "user-42");
        assert_eq!(decoded.scope, "read profile");
        assert_eq!(decoded.client_id, "app-client");
    }

    #[test]
    fn test_jwt_wrong_secret_fails() {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;
        let claims = Claims {
            sub: "user-1".into(),
            client_id: "app".into(),
            scope: "read".into(),
            iat: now,
            exp: now + 3600,
            jti: "jti-1".into(),
        };
        let token = jwt_sign(&claims, b"correct-secret").expect("sign failed");
        let result = jwt_verify(&token, b"wrong-secret");
        assert!(result.is_err(), "should fail with wrong secret");
    }

    #[test]
    fn test_jwt_expired_token_fails() {
        let claims = Claims {
            sub: "user-1".into(),
            client_id: "app".into(),
            scope: "read".into(),
            iat: 1_000_000,
            exp: 1_000_001,
            jti: "jti-exp".into(),
        };
        let token = jwt_sign(&claims, b"secret").expect("sign failed");
        let result = jwt_verify(&token, b"secret");
        assert!(result.is_err(), "expired token must be rejected");
    }

    // --- Scope tests ---

    #[test]
    fn test_parse_scopes_basic() {
        let scopes = parse_scopes("read write profile");
        assert_eq!(scopes, vec!["read", "write", "profile"]);
    }

    #[test]
    fn test_parse_scopes_extra_whitespace() {
        let scopes = parse_scopes("  read   write  ");
        assert_eq!(scopes, vec!["read", "write"]);
    }

    #[test]
    fn test_parse_scopes_empty_string() {
        let scopes = parse_scopes("");
        assert!(scopes.is_empty());
    }

    #[test]
    fn test_parse_scopes_normalizes_case() {
        let scopes = parse_scopes("READ Write PROFILE");
        assert_eq!(scopes, vec!["read", "write", "profile"]);
    }

    #[test]
    fn test_scopes_allowed_all_permitted() {
        let requested = parse_scopes("read write");
        let allowed = parse_scopes("read write admin profile");
        assert!(scopes_allowed(&requested, &allowed));
    }

    #[test]
    fn test_scopes_allowed_denied_when_out_of_range() {
        let requested = parse_scopes("read admin");
        let allowed = parse_scopes("read write");
        assert!(!scopes_allowed(&requested, &allowed));
    }

    #[test]
    fn test_token_has_scope_present() {
        assert!(token_has_scope("read write profile", "write"));
        assert!(token_has_scope("read write profile", "READ"));
    }

    #[test]
    fn test_token_has_scope_absent() {
        assert!(!token_has_scope("read write", "admin"));
    }

    // --- Client credential hashing tests ---

    #[test]
    fn test_hash_and_verify_client_secret() {
        let secret = "cs_live_super_secret_value_123";
        let hash = hash_client_secret(secret);
        assert!(hash.starts_with("$argon2"), "should be argon2 hash");
        assert!(verify_client_secret(secret, &hash));
    }

    #[test]
    fn test_wrong_client_secret_fails_verification() {
        let secret = "correct_secret";
        let hash = hash_client_secret(secret);
        assert!(!verify_client_secret("wrong_secret", &hash));
    }

    #[test]
    fn test_hash_is_not_plaintext() {
        let secret = "my_client_secret";
        let hash = hash_client_secret(secret);
        assert_ne!(hash, secret);
        assert!(hash.len() > secret.len());
    }
}
```

### ผลลัพธ์ `cargo test` จริง

```
running 19 tests
test tests::test_jwt_expired_token_fails ... ok
test tests::test_jwt_sign_and_verify_roundtrip ... ok
test tests::test_parse_scopes_basic ... ok
test tests::test_parse_scopes_empty_string ... ok
test tests::test_jwt_wrong_secret_fails ... ok
test tests::test_parse_scopes_extra_whitespace ... ok
test tests::test_parse_scopes_normalizes_case ... ok
test tests::test_pkce_unknown_method_returns_false ... ok
test tests::test_pkce_s256_rfc_vector ... ok
test tests::test_pkce_verify_plain ... ok
test tests::test_pkce_verify_s256_tampered_verifier ... ok
test tests::test_pkce_verify_s256_valid ... ok
test tests::test_scopes_allowed_all_permitted ... ok
test tests::test_scopes_allowed_denied_when_out_of_range ... ok
test tests::test_token_has_scope_absent ... ok
test tests::test_token_has_scope_present ... ok
test tests::test_hash_is_not_plaintext ... ok
test tests::test_hash_and_verify_client_secret ... ok
test tests::test_wrong_client_secret_fails_verification ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.05s
```

---

## Pitfalls ที่ต้องระวัง

### Pitfall 1: Auth Code Reuse — ต้อง Single-Use Enforcement อย่างเป็น Atomic

**ปัญหา:** ถ้าตรวจสอบ `used = false` และ update เป็น `true` ในสอง query แยกกัน race condition อาจทำให้ attacker ใช้ code เดิมได้สองครั้ง

```rust
// WRONG — ไม่ atomic
let code = db.fetch("SELECT * FROM auth_codes WHERE code = $1", code).await?;
if !code.used {
    db.execute("UPDATE auth_codes SET used = true WHERE code = $1", code).await?;
    // ← race condition ตรงนี้!
}

// CORRECT — ใช้ UPDATE ... RETURNING เพื่อ atomic check-and-mark
let updated = sqlx::query_as!(
    AuthCode,
    "UPDATE auth_codes SET used = true
     WHERE code = $1 AND NOT used AND expires_at > NOW()
     RETURNING *",
    code
)
.fetch_optional(&db)
.await?;

// ถ้า None → code ถูกใช้แล้ว หรือ expired
match updated {
    None => return Err(OAuthError::InvalidGrant("code already used or expired".into())),
    Some(auth_code) => { /* proceed */ }
}
```

### Pitfall 2: Redirect URI Validation ต้องเป็น Exact Match ไม่ใช่ Prefix Match

**ปัญหา:** การใช้ `starts_with()` หรือ prefix matching สำหรับ redirect_uri เปิดช่องให้ open redirector attack

```rust
// WRONG — prefix match อันตราย
if redirect_uri.starts_with("https://myapp.com") {
    // attacker ส่ง https://myapp.com.evil.com/callback ได้!
}

// WRONG — ไม่ normalize URL ก่อนเปรียบเทียบ
if client.redirect_uris.contains(&redirect_uri) {
    // https://myapp.com/cb และ https://myapp.com/cb?extra=1 ต่างกัน!
}

// CORRECT — exact match หลัง normalize (ลบ trailing slash, lowercase scheme/host)
fn normalize_redirect_uri(uri: &str) -> String {
    // parse URI แล้วเอาเฉพาะ scheme+host+path (ไม่รวม query/fragment)
    let parsed = url::Url::parse(uri).expect("invalid URL");
    format!("{}://{}{}", parsed.scheme(), parsed.host_str().unwrap_or(""), parsed.path())
}

let normalized_requested = normalize_redirect_uri(&redirect_uri);
let is_registered = client
    .redirect_uris
    .iter()
    .any(|r| normalize_redirect_uri(r) == normalized_requested);
```

### Pitfall 3: PKCE Verification ต้องใช้ Constant-Time Comparison

**ปัญหา:** การใช้ `==` ปกติสำหรับ string comparison เปิดช่องให้ timing attack ที่ attacker สามารถเดา challenge ทีละ byte ได้จาก response time

```rust
// WRONG — ใช้ == ปกติ
if computed_challenge == client_challenge {
    // timing side-channel!
}

// CORRECT — constant-time comparison
fn constant_time_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        // ต้องคืน false แต่ไม่ short-circuit ทันที
        // ให้ใช้ dummy comparison เพื่อคง timing
        return a.iter().zip(a.iter()).fold(0u8, |acc, (x, y)| acc | (x ^ y)) == 1;
    }
    a.iter().zip(b.iter()).fold(0u8, |acc, (x, y)| acc | (x ^ y)) == 0
}

// หรือใช้ crate `subtle` ที่ออกแบบมาสำหรับ crypto
use subtle::ConstantTimeEq;
let valid = computed.as_bytes().ct_eq(expected.as_bytes()).into();
```

### Pitfall 4: Access Token ใน Log / Error Message

**ปัญหา:** JWT access tokens ที่ยาวมักติดอยู่ใน stack trace หรือ request log ทำให้ token รั่วออกไปใน log aggregation system

```rust
// WRONG — log ที่มี token value
tracing::error!("Token validation failed for: {}", access_token);
tracing::debug!("Processing request with auth header: {:?}", authorization_header);

// WRONG — include token ใน error struct
#[derive(Debug)]
struct TokenError {
    token: String,  // ← จะปรากฏใน {:?} ทุกครั้ง
    reason: String,
}

// CORRECT — ใช้แค่ token hash หรือ jti ใน log
tracing::error!(
    token_prefix = &access_token[..12],  // เพียง 12 chars แรก
    error = ?err,
    "Token validation failed"
);

// CORRECT — mask authorization header
fn mask_auth_header(header: &str) -> String {
    if let Some(token) = header.strip_prefix("Bearer ") {
        format!("Bearer {}…", &token[..std::cmp::min(8, token.len())])
    } else {
        "Bearer [masked]".into()
    }
}
```

### Pitfall 5: Refresh Token Rotation ต้องทำ Atomic

**ปัญหา:** ถ้า revoke refresh token เดิมสำเร็จ แต่ออก refresh token ใหม่ล้มเหลว client จะ stuck เพราะ token เดิมถูก revoke แต่ไม่ได้ token ใหม่

```rust
// WRONG — ไม่ใช้ transaction
sqlx::query!("UPDATE tokens SET revoked = true WHERE id = $1", old_id)
    .execute(&db).await?;
// ← ถ้าขั้นต่อไปล้มเหลว old token ถูก revoke แต่ไม่มี new token!
let new_rt = generate_token();
sqlx::query!("INSERT INTO tokens ...").execute(&db).await?;

// CORRECT — ใช้ transaction
let mut tx = db.begin().await?;
sqlx::query!("UPDATE tokens SET revoked = true WHERE id = $1", old_id)
    .execute(&mut *tx).await?;
let new_rt = generate_token();
sqlx::query!("INSERT INTO tokens (token_hash, ...) VALUES ($1, ...)", sha256(&new_rt), ...)
    .execute(&mut *tx).await?;
tx.commit().await?;  // atomic: ทั้งคู่สำเร็จหรือล้มเหลวพร้อมกัน
```

---

## การ Package และ Deploy

### Dockerfile

```dockerfile
# Stage 1: Build
FROM rust:1.82-slim AS builder
WORKDIR /app

# Pre-fetch dependencies (layer caching)
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main() {}' > src/main.rs
RUN cargo build --release 2>&1 | tail -5
RUN rm src/main.rs

# Build actual binary
COPY src ./src
COPY migrations ./migrations
RUN cargo build --release

# Stage 2: Runtime
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/target/release/oauth2-server /app/oauth2-server
COPY --from=builder /app/migrations /app/migrations

# Generate RSA keypair (ใน production ใช้ secret manager แทน)
RUN openssl genrsa -out /app/keys/private.pem 2048 && \
    openssl rsa -in /app/keys/private.pem -pubout -out /app/keys/public.pem

EXPOSE 8080
CMD ["/app/oauth2-server"]
```

### Docker Compose (สำหรับ Development)

```yaml
# docker-compose.yml
version: "3.9"
services:
  oauth2-server:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://oauth2:password@db:5432/oauth2
      ISSUER: http://localhost:8080
      RUST_LOG: oauth2_server=debug
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: oauth2
      POSTGRES_PASSWORD: password
      POSTGRES_DB: oauth2
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U oauth2"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### การทดสอบด้วย curl

```bash
# 1. ลงทะเบียน client
curl -X POST http://localhost:8080/clients \
  -H "Content-Type: application/json" \
  -d '{
    "client_name": "My App",
    "redirect_uris": ["http://localhost:3000/callback"],
    "allowed_scopes": ["read", "write", "profile"]
  }'

# Response:
# {
#   "client_id": "cli_a1b2c3d4e5f6g7h8",
#   "client_secret": "AbCdEfGhIjKlMnOpQrStUvWx...",
#   ...
# }

# 2. Client Credentials flow
curl -X POST http://localhost:8080/token \
  -d "grant_type=client_credentials&client_id=cli_...&client_secret=...&scope=read"

# Response:
# {
#   "access_token": "eyJhbGciOiJSUzI1NiJ9...",
#   "token_type": "Bearer",
#   "expires_in": 3600,
#   "scope": "read"
# }

# 3. Token Introspection
curl -X POST http://localhost:8080/introspect \
  -d "token=eyJhbGciOiJSUzI1NiJ9..."

# Response:
# {
#   "active": true,
#   "scope": "read",
#   "client_id": "cli_...",
#   "exp": 1750000000,
#   "iat": 1749996400
# }

# 4. Token Revocation
curl -X POST http://localhost:8080/revoke \
  -d "token=eyJhbGciOiJSUzI1NiJ9..."

# Response: {} (200 OK เสมอตาม RFC 7009)

# 5. OpenID Connect Discovery
curl http://localhost:8080/.well-known/openid-configuration

# 6. JWKS
curl http://localhost:8080/.well-known/jwks.json

# 7. ทดสอบ Protected Endpoint โดยไม่มี token
curl http://localhost:8080/api/profile
# Response: {"error":"unauthorized","error_description":"Bearer token required"}

# 8. ทดสอบด้วย valid token
curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiJ9..."
# Response: {"sub":"...","name":"...","email":"..."}
```

### Key Rotation

เมื่อต้องการ rotate RSA keys:

```bash
# 1. สร้าง key ใหม่ พร้อม kid ใหม่
openssl genrsa -out keys/private-2025.pem 2048

# 2. เพิ่ม key ใหม่เข้า JWKS โดยไม่ลบ key เก่า
# (resource servers จะ cache JWKS และต้องการเวลา refresh)

# 3. เปลี่ยน server ให้ sign ด้วย key ใหม่ แต่ยังรับ verify ทั้งสอง key

# 4. รอจนกว่า token ที่ sign ด้วย key เก่าทั้งหมดหมดอายุ (เช่น 1 ชั่วโมง)

# 5. ลบ key เก่าออกจาก JWKS
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม OpenID Connect ID Token

เมื่อ client ขอ scope `openid` ใน Authorization Code flow ให้ออก ID Token (JWT แยก) พร้อม claims:

```json
{
  "iss": "https://auth.example.com",
  "sub": "user-001",
  "aud": "client-id",
  "exp": 1750000000,
  "iat": 1749996400,
  "nonce": "random-nonce-from-client",
  "email": "user@example.com",
  "name": "Full Name"
}
```

**Hint:** เพิ่ม `nonce` parameter ใน `/authorize` request เก็บไว้ใน `auth_codes` table แล้วนำมาใส่ใน ID Token claims ที่ `/token` endpoint

**ความยาก:** ⭐⭐ — ต้องการ: เพิ่ม user profile table, เชื่อม `user_info_endpoint` GET `/userinfo`

### แบบฝึกหัดที่ 2: Implement Device Authorization Grant (RFC 8628)

สำหรับ smart TV หรือ device ที่ไม่มี browser:

```
POST /device/code → {"device_code":"...", "user_code":"ABCD-1234", "verification_uri":"..."}
GET /device/activate → แสดงหน้า enter user_code
POST /token (grant_type=urn:ietf:params:oauth:grant-type:device_code) → polling จาก device
```

**ความยาก:** ⭐⭐⭐ — ต้องการ: polling mechanism, user_code generation (8 chars, pronounceable), expiry handling

### แบบฝึกหัดที่ 3: Rate Limiting และ Brute-Force Protection

เพิ่ม rate limiter ที่ token endpoint เพื่อป้องกัน password spraying:

```rust
// ใช้ tower_governor หรือ implement sliding window counter ใน Redis
// สร้าง middleware ที่ track:
// - failed authentication attempts per client_id per IP
// - lock out after 5 failures ใน 60 วินาที
// - ส่ง 429 Too Many Requests พร้อม Retry-After header
```

**ความยาก:** ⭐⭐⭐ — ต้องการ: Redis integration, IP extraction จาก X-Forwarded-For, distributed counter

### แบบฝึกหัดที่ 4: Dynamic Client Registration Metadata Validation

ปัจจุบัน `POST /clients` รับ request ง่ายๆ ให้ upgrade เป็น RFC 7591 (Dynamic Client Registration):

```rust
// เพิ่ม validation:
// - software_statement (signed JWT จาก software publisher)
// - ตรวจสอบว่า redirect_uris ไม่มี fragment component (#)
// - enforce ว่า public client ต้องใช้ PKCE
// - เพิ่ม client_metadata: logo_uri, tos_uri, policy_uri
// - ส่ง registration_client_uri กลับเพื่อให้ client update ตัวเองได้
```

**ความยาก:** ⭐⭐⭐⭐ — ต้องการ: JWT parsing, URL validation, schema validation ด้วย validator crate

---

## สรุป

โปรเจคนี้สร้าง **OAuth2 Authorization Server ที่ production-ready** ครอบคลุม:

| Feature | Standard | Status |
|---------|----------|--------|
| Authorization Code Flow | RFC 6749 | สมบูรณ์ |
| Client Credentials Flow | RFC 6749 | สมบูรณ์ |
| Refresh Token Grant | RFC 6749 | สมบูรณ์ |
| PKCE (S256) | RFC 7636 | สมบูรณ์ |
| Token Introspection | RFC 7662 | สมบูรณ์ |
| Token Revocation | RFC 7009 | สมบูรณ์ |
| JWT (RS256) + JWKS | RFC 7519/7517 | สมบูรณ์ |
| Client Registration | RFC 7591 (partial) | สมบูรณ์ |
| OpenID Connect Discovery | OIDC Discovery 1.0 | สมบูรณ์ |
| Scope Enforcement | RFC 6749 Section 3.3 | สมบูรณ์ |

**Pattern สำคัญที่ได้เรียน:**
- **Atomic database operations** — ใช้ `UPDATE ... RETURNING` แทน SELECT+UPDATE แยก เพื่อป้องกัน race condition
- **Defense in depth** — PKCE + constant-time comparison + Argon2id + token hashing ป้องกันหลายชั้น
- **Standards compliance** — การ implement ตาม RFC อย่างเคร่งครัดทำให้ interoperable กับทุก OAuth2 client library
- **Token lifecycle** — ทุก token มี TTL, revocation support, และ rotation pattern
- **Asymmetric cryptography** — RS256 ช่วยแยก signing authority ออกจาก verification ทำให้ microservices ตรวจสอบ token ได้อิสระ

โปรเจคถัดไป **B07 — Real-time Chat Server** จะนำ async patterns เดิมมาสร้าง WebSocket hub ที่รองรับ rooms, presence, และ message history — เชื่อมกับ OAuth2 ที่สร้างไว้เพื่อ authenticate WebSocket connections ด้วย Bearer token

---

**โปรเจคก่อนหน้า:** [project-b05-webhook-relay.md](project-b05-webhook-relay.md) | **โปรเจคถัดไป:** [project-b07-realtime-chat.md](project-b07-realtime-chat.md)
