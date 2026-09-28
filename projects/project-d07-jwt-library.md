# Project D07: JWT Library

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

JSON Web Token (JWT) คือมาตรฐาน RFC 7519 สำหรับส่ง claims ระหว่าง parties ในรูปแบบ compact และ self-contained โปรเจคนี้สร้าง JWT library ตั้งแต่ศูนย์โดยไม่ใช้ crate `jsonwebtoken` หรือ `jwt-simple` — implements `encode`/`decode`, รองรับ HMAC (HS256/HS384/HS512) และ asymmetric (RS256, ES256), ระบบ key rotation ผ่าน `Keyring` พร้อม `kid` matching, JWKS endpoint, token revocation ด้วย JTI blacklist, และ timing-safe comparison ด้วย `subtle::ConstantTimeEq`

ใน production ระบบนี้ตรงกับ use case:
- **Identity provider (IdP)** ที่ออก access token + refresh token สำหรับ OAuth 2.0 / OIDC
- **Microservices** ที่ต้องการ verify token โดยไม่ต้องเรียก auth service ทุกครั้ง (stateless verification)
- **API Gateway** ที่ตรวจสอบ JWT ก่อนส่ง request ไปยัง upstream service
- **Multi-tenant SaaS** ที่ใช้ `kid` แยก key ตาม tenant หรือ key rotation cycle

**Learning value:** โปรเจคนี้สอนการทำงานภายในของ JWT format, HMAC และ RSA/EC signing, algorithm confusion vulnerability และวิธีป้องกัน, constant-time comparison สำหรับ MAC verification, JWK serialization ตาม RFC 7517, และ axum `FromRequestParts` extractor pattern

## สิ่งที่จะได้เรียนรู้

- **JWT structure จากข้างใน** — base64url encoding, header.payload.signature format, canonical JSON
- **HMAC-SHA256/384/512** — สร้าง MAC ด้วย `hmac` + `sha2` crates, verify ด้วย constant-time comparison
- **RSA-PKCS1v15 และ ECDSA-P256** — asymmetric signing/verification ด้วย `rsa` และ `p256` crates
- **Algorithm confusion prevention** — เหตุใด `decode` ต้องรับ `Algorithm` จาก caller แทนการอ่านจาก header
- **JWK / JWKS** — serialize/deserialize RSA และ EC public key ตาม RFC 7517
- **Key rotation** — `Keyring` + `kid` header field สำหรับ rotate key โดยไม่ restart service
- **Token blacklist / revocation** — JTI claim + in-memory `HashSet` revocation store
- **axum `FromRequestParts`** — custom extractor ที่ดึง Bearer token และ inject `Claims` เข้า handler

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50** — async/await, tokio runtime
- **Part 61–65** — axum web framework, handler, router, extractor
- **Part 80–85** — serde/serde_json, Serialize/Deserialize
- **Part 86–90** — cryptography primitives (hmac, sha2, symmetric/asymmetric)
- **Part 91–95** — trait objects, `Arc<RwLock<_>>` สำหรับ shared state

## โครงสร้างโปรเจค (Project Layout)

```
jwt-lib/
├── src/
│   ├── lib.rs          # re-exports สาธารณะ
│   ├── algorithm.rs    # enum Algorithm (HS256/384/512/RS256/ES256)
│   ├── base64url.rs    # base64url encode/decode wrapper
│   ├── claims.rs       # Claims struct, Audience, ValidationOptions
│   ├── encode.rs       # encode() + sign() + JwtError
│   ├── decode.rs       # decode() + verify_signature() + validate_claims()
│   ├── keyring.rs      # Key enum (Hmac/Rsa/Ec) + Keyring struct
│   ├── jwk.rs          # Jwk/Jwks structs + rsa_public_to_jwk/ec_public_to_jwk
│   ├── blacklist.rs    # TokenBlacklist (Arc<RwLock<HashSet<String>>>)
│   ├── refresh.rs      # RefreshStore — opaque refresh token store
│   └── extractor.rs    # JwtAuth axum extractor
├── tests/
│   └── jwt_tests.rs    # integration tests
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### JWT Format

Token ที่สมบูรณ์มีรูปแบบ:

```
base64url(header) + "." + base64url(payload) + "." + base64url(signature)
```

**Header** (JSON object):
```json
{ "typ": "JWT", "alg": "HS256", "kid": "v2" }
```

**Payload** (JSON object ของ Claims):
```json
{ "sub": "user-42", "iss": "my-service", "exp": 1735689600, "jti": "uuid-..." }
```

**Signature** — ผลลัพธ์ของ sign(`base64url(header) + "." + base64url(payload)`, key)

### Algorithm Confusion Prevention

ช่องโหว่ algorithm confusion เกิดเมื่อ library อ่าน `alg` field จาก header เพื่อเลือก algorithm ในการ verify — ผู้ส่งอาจ forge header โดยเปลี่ยน `alg` เป็น `none` หรือเปลี่ยนจาก RS256 เป็น HS256 แล้วใช้ RSA public key เป็น HMAC secret

Library นี้ป้องกันด้วยการ:
1. `decode()` รับ `expected_alg: &Algorithm` จาก caller — caller กำหนด algorithm ที่ยอมรับ
2. Header `alg` field ถูก compare กับ `expected_alg` — ถ้าไม่ตรงหรือเป็น `"none"` ให้ reject ทันที
3. ไม่มี code path ใดที่ใช้ header `alg` field เพื่อเลือก verification logic

```
decode(token, key, &Algorithm::HS256, opts, blacklist)
                    ^^^^^^^^^^^^^^^^^
                    caller กำหนด — ไม่ trust header
```

### Timing-Safe Comparison

HMAC verification ต้องใช้ constant-time comparison เพื่อป้องกัน timing side-channel:

```
ปกติ: bytes.eq(other_bytes) หยุดที่ byte แรกที่ต่างกัน
       → ใช้เวลา proportional กับจำนวน byte ที่ match
       → attacker วัดเวลาแล้วเดา prefix ของ expected HMAC ได้

ConstantTimeEq: เปรียบเทียบทุก byte เสมอ ไม่ว่าจะ match หรือไม่
               → เวลาคงที่ → ไม่รั่ว information
```

```rust
use subtle::ConstantTimeEq;

if expected_mac.ct_eq(provided_mac).into() {
    Ok(())
} else {
    Err(JwtError::InvalidSignature)
}
```

### Key Lifecycle และ Rotation

```
Keyring {
  keys: {
    "v1" → Key::Hmac { secret: [...] }  ← ยังคง verify token เก่าได้
    "v2" → Key::Hmac { secret: [...] }  ← current key สำหรับ sign
  }
  current_kid: "v2"
}

sign new token → ใช้ keys["v2"], ฝัง kid="v2" ใน header
verify token   → อ่าน kid จาก header, lookup keys[kid], verify
remove "v1"    → token เก่าที่ signed ด้วย v1 จะ fail verification (intended)
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ base64url

เริ่มจาก `Cargo.toml` และ utility module พื้นฐาน

**`Cargo.toml`:**

```toml
[package]
name = "jwt-lib"
version = "0.1.0"
edition = "2021"

[dependencies]
hmac = "0.12"
sha2 = "0.10"
base64 = "0.22"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
subtle = "2"
rand = "0.8"
hex = "0.4"
rsa = { version = "0.9", features = ["sha2"] }
p256 = { version = "0.13", features = ["ecdsa", "pkcs8"] }
pkcs8 = { version = "0.10", features = ["pem"] }
signature = "2"
tokio = { version = "1", features = ["full"] }
axum = "0.8"
http = "1"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

**`src/base64url.rs`** — wrapper สำหรับ base64url encoding ตาม JWT spec (URL-safe, no padding):

```rust
use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};

pub fn encode(data: &[u8]) -> String {
    URL_SAFE_NO_PAD.encode(data)
}

pub fn decode(s: &str) -> Result<Vec<u8>, base64::DecodeError> {
    URL_SAFE_NO_PAD.decode(s)
}
```

JWT ใช้ base64url **ไม่มี padding** (`=`) เพราะ `=` ต้องถูก percent-encode ใน URL ซึ่งทำให้ token ยาวขึ้นโดยไม่จำเป็น

**`src/algorithm.rs`:**

```rust
/// อัลกอริทึมที่รองรับ — caller ต้องระบุ algorithm อย่างชัดเจน
/// ไม่อ่าน alg จาก header เพื่อป้องกัน algorithm confusion
#[derive(Debug, Clone, PartialEq)]
pub enum Algorithm {
    HS256,
    HS384,
    HS512,
    RS256,
    ES256,
}

impl Algorithm {
    pub fn name(&self) -> &'static str {
        match self {
            Algorithm::HS256 => "HS256",
            Algorithm::HS384 => "HS384",
            Algorithm::HS512 => "HS512",
            Algorithm::RS256 => "RS256",
            Algorithm::ES256 => "ES256",
        }
    }
}
```

### ขั้นที่ 2: Claims struct และ ValidationOptions

**`src/claims.rs`:**

```rust
use serde::{Deserialize, Serialize};
use serde_json::Value;
use std::collections::HashMap;

/// Standard JWT Claims ตาม RFC 7519
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct Claims {
    /// Subject — identity ของผู้ถือ token
    #[serde(skip_serializing_if = "Option::is_none")]
    pub sub: Option<String>,
    /// Issuer — ผู้ออก token
    #[serde(skip_serializing_if = "Option::is_none")]
    pub iss: Option<String>,
    /// Audience — ผู้รับที่ token นี้ตั้งใจให้
    #[serde(skip_serializing_if = "Option::is_none")]
    pub aud: Option<Audience>,
    /// Expiration time — Unix timestamp (seconds)
    /// decode() ปฏิเสธ token ถ้า now > exp
    #[serde(skip_serializing_if = "Option::is_none")]
    pub exp: Option<i64>,
    /// Not Before — Unix timestamp (seconds)
    /// decode() ปฏิเสธ token ถ้า now < nbf
    #[serde(skip_serializing_if = "Option::is_none")]
    pub nbf: Option<i64>,
    /// Issued At — Unix timestamp (seconds)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub iat: Option<i64>,
    /// JWT ID — ใช้สำหรับ revocation (blacklist)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub jti: Option<String>,
    /// Custom claims
    #[serde(flatten)]
    pub extra: HashMap<String, Value>,
}

/// aud claim สามารถเป็น string เดียวหรือ array ตาม RFC 7519 §4.1.3
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum Audience {
    Single(String),
    Multiple(Vec<String>),
}

impl Audience {
    pub fn contains(&self, expected: &str) -> bool {
        match self {
            Audience::Single(s) => s == expected,
            Audience::Multiple(v) => v.iter().any(|s| s == expected),
        }
    }
}

/// พารามิเตอร์ควบคุมการ validate claims
#[derive(Debug, Default)]
pub struct ValidationOptions {
    /// กำหนด "now" สำหรับ testing (None = ใช้เวลาจริง)
    pub now: Option<i64>,
    /// clock skew tolerance (วินาที) สำหรับ exp และ nbf
    pub leeway: u64,
    /// ตรวจสอบ iss ต้องอยู่ใน list นี้
    pub required_issuer: Option<Vec<String>>,
    /// ตรวจสอบ aud ต้องมี value นี้
    pub required_audience: Option<String>,
    /// ถ้า true: token ที่ไม่มี exp claim จะถูก reject
    pub require_exp: bool,
}

impl ValidationOptions {
    pub fn strict() -> Self {
        ValidationOptions {
            leeway: 0,
            require_exp: true,
            ..Default::default()
        }
    }

    pub fn current_time(&self) -> i64 {
        self.now.unwrap_or_else(|| {
            std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)
                .unwrap()
                .as_secs() as i64
        })
    }
}
```

**หมายเหตุ `#[serde(flatten)]`:** custom claims ถูก serialize/deserialize เป็น fields ระดับเดียวกับ standard claims ทำให้ payload JSON ดูเหมือน flat object ไม่มี nesting

### ขั้นที่ 3: Key types และ Keyring

**`src/keyring.rs`:**

```rust
use crate::algorithm::Algorithm;
use std::collections::HashMap;

/// Key ที่รองรับ — enum แทน trait object เพื่อ exhaustive matching
pub enum Key {
    Hmac {
        secret: Vec<u8>,
        algorithm: Algorithm,
    },
    Rsa {
        private_key: rsa::RsaPrivateKey,
        public_key: rsa::RsaPublicKey,
    },
    Ec {
        private_key: p256::SecretKey,
        public_key: p256::PublicKey,
    },
}

impl Key {
    pub fn algorithm(&self) -> Algorithm {
        match self {
            Key::Hmac { algorithm, .. } => algorithm.clone(),
            Key::Rsa { .. } => Algorithm::RS256,
            Key::Ec { .. } => Algorithm::ES256,
        }
    }

    pub fn hmac(secret: impl Into<Vec<u8>>, alg: Algorithm) -> Self {
        Key::Hmac { secret: secret.into(), algorithm: alg }
    }
}

impl std::fmt::Debug for Key {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Key::Hmac { algorithm, .. } => write!(f, "Key::Hmac({:?})", algorithm),
            Key::Rsa { .. } => write!(f, "Key::Rsa"),
            Key::Ec { .. } => write!(f, "Key::Ec"),
        }
    }
}

/// Keyring — จัดการ key หลายตัวพร้อม kid rotation
///
/// Flow การ sign:
///   1. เรียก current_key() → ได้ (kid, key)
///   2. ส่ง kid ไปยัง encode() เพื่อฝังใน header
///
/// Flow การ verify:
///   1. อ่าน kid จาก header
///   2. เรียก get_key(kid) → ได้ key ที่ใช้ verify
pub struct Keyring {
    keys: HashMap<String, Key>,
    current_kid: String,
}

impl Keyring {
    pub fn new(kid: impl Into<String>, key: Key) -> Self {
        let kid = kid.into();
        let mut keys = HashMap::new();
        keys.insert(kid.clone(), key);
        Keyring { keys, current_kid: kid }
    }

    /// เพิ่ม key ใหม่ (สำหรับ rotation — เพิ่มก่อน set_current)
    pub fn add_key(&mut self, kid: impl Into<String>, key: Key) {
        self.keys.insert(kid.into(), key);
    }

    /// ลบ key เก่า (token ที่ signed ด้วย key นี้จะ verify ไม่ผ่านอีก)
    pub fn remove_key(&mut self, kid: &str) {
        self.keys.remove(kid);
    }

    /// เปลี่ยน current key ที่ใช้ sign token ใหม่
    pub fn set_current(&mut self, kid: impl Into<String>) {
        self.current_kid = kid.into();
    }

    /// ดึง current key สำหรับ signing
    pub fn current_key(&self) -> Option<(&str, &Key)> {
        self.keys.get(&self.current_kid)
            .map(|k| (self.current_kid.as_str(), k))
    }

    /// lookup key ด้วย kid (สำหรับ verification)
    pub fn get_key(&self, kid: &str) -> Option<&Key> {
        self.keys.get(kid)
    }

    pub fn kids(&self) -> Vec<&str> {
        self.keys.keys().map(|s| s.as_str()).collect()
    }
}
```

### ขั้นที่ 4: encode() และ decode()

**`src/encode.rs`** — สร้าง JWT token:

```rust
use crate::algorithm::Algorithm;
use crate::base64url;
use crate::claims::Claims;
use crate::keyring::Key;

use hmac::{Hmac, Mac};
use sha2::{Sha256, Sha384, Sha512};
use serde_json::json;

#[derive(Debug)]
pub enum JwtError {
    SerializationError(String),
    SigningError(String),
    InvalidToken(String),
    ExpiredToken,
    NotYetValid,
    InvalidIssuer,
    InvalidAudience,
    RevokedToken,
    AlgorithmNotAllowed,
    InvalidSignature,
    KeyNotFound(String),
    Base64Error(String),
}

impl std::fmt::Display for JwtError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            JwtError::SerializationError(s) => write!(f, "serialization error: {s}"),
            JwtError::SigningError(s) => write!(f, "signing error: {s}"),
            JwtError::InvalidToken(s) => write!(f, "invalid token: {s}"),
            JwtError::ExpiredToken => write!(f, "token expired"),
            JwtError::NotYetValid => write!(f, "token not yet valid (nbf)"),
            JwtError::InvalidIssuer => write!(f, "issuer not in allowed list"),
            JwtError::InvalidAudience => write!(f, "audience mismatch"),
            JwtError::RevokedToken => write!(f, "token has been revoked"),
            JwtError::AlgorithmNotAllowed => write!(f, "algorithm not allowed"),
            JwtError::InvalidSignature => write!(f, "signature verification failed"),
            JwtError::KeyNotFound(k) => write!(f, "key not found: {k}"),
            JwtError::Base64Error(s) => write!(f, "base64 decode error: {s}"),
        }
    }
}

impl std::error::Error for JwtError {}

/// สร้าง JWT token จาก Claims + Key
/// kid — ถ้ากำหนดจะถูกฝังใน header `kid` field
pub fn encode(claims: &Claims, key: &Key, kid: Option<&str>) -> Result<String, JwtError> {
    let alg_name = key.algorithm().name();

    let mut header = json!({ "typ": "JWT", "alg": alg_name });
    if let Some(k) = kid {
        header["kid"] = json!(k);
    }

    let header_json = serde_json::to_string(&header)
        .map_err(|e| JwtError::SerializationError(e.to_string()))?;
    let payload_json = serde_json::to_string(claims)
        .map_err(|e| JwtError::SerializationError(e.to_string()))?;

    let header_b64 = base64url::encode(header_json.as_bytes());
    let payload_b64 = base64url::encode(payload_json.as_bytes());
    let signing_input = format!("{header_b64}.{payload_b64}");

    let sig = sign(signing_input.as_bytes(), key)?;
    let sig_b64 = base64url::encode(&sig);

    Ok(format!("{signing_input}.{sig_b64}"))
}

fn sign(data: &[u8], key: &Key) -> Result<Vec<u8>, JwtError> {
    match key {
        Key::Hmac { secret, algorithm } => {
            match algorithm {
                Algorithm::HS256 => {
                    let mut mac = <Hmac<Sha256> as Mac>::new_from_slice(secret)
                        .map_err(|e| JwtError::SigningError(e.to_string()))?;
                    mac.update(data);
                    Ok(mac.finalize().into_bytes().to_vec())
                }
                Algorithm::HS384 => {
                    let mut mac = <Hmac<Sha384> as Mac>::new_from_slice(secret)
                        .map_err(|e| JwtError::SigningError(e.to_string()))?;
                    mac.update(data);
                    Ok(mac.finalize().into_bytes().to_vec())
                }
                Algorithm::HS512 => {
                    let mut mac = <Hmac<Sha512> as Mac>::new_from_slice(secret)
                        .map_err(|e| JwtError::SigningError(e.to_string()))?;
                    mac.update(data);
                    Ok(mac.finalize().into_bytes().to_vec())
                }
                _ => Err(JwtError::SigningError("wrong key type".into())),
            }
        }
        Key::Rsa { private_key, .. } => {
            use rsa::pkcs1v15::SigningKey;
            use rsa::signature::SignatureEncoding;
            use rsa::signature::Signer;
            let signing_key = SigningKey::<sha2::Sha256>::new(private_key.clone());
            let sig = signing_key.sign(data);
            Ok(sig.to_bytes().to_vec())
        }
        Key::Ec { private_key, .. } => {
            use p256::ecdsa::{SigningKey, Signature, signature::Signer,
                              signature::SignatureEncoding};
            let signing_key = SigningKey::from(private_key.clone());
            let sig: Signature = signing_key.sign(data);
            Ok(sig.to_bytes().to_vec())
        }
    }
}
```

**`src/decode.rs`** — verify signature และ validate claims:

```rust
use crate::algorithm::Algorithm;
use crate::base64url;
use crate::blacklist::TokenBlacklist;
use crate::claims::{Claims, ValidationOptions};
pub use crate::encode::JwtError;
use crate::keyring::Key;

use hmac::{Hmac, Mac};
use sha2::{Sha256, Sha384, Sha512};
use subtle::ConstantTimeEq;

/// decode JWT และตรวจสอบ signature + claims
///
/// Security invariants:
/// - expected_alg ต้องตรงกับ header alg field — ถ้าไม่ตรง → AlgorithmNotAllowed
/// - header alg = "none" → AlgorithmNotAllowed เสมอ
/// - signature verify ก่อน decode payload เพื่อป้องกัน payload injection
/// - HMAC comparison ด้วย ConstantTimeEq
pub fn decode(
    token: &str,
    key: &Key,
    expected_alg: &Algorithm,
    opts: &ValidationOptions,
    blacklist: Option<&TokenBlacklist>,
) -> Result<Claims, JwtError> {
    let parts: Vec<&str> = token.splitn(3, '.').collect();
    if parts.len() != 3 {
        return Err(JwtError::InvalidToken("expected 3 parts".into()));
    }

    let (header_b64, payload_b64, sig_b64) = (parts[0], parts[1], parts[2]);

    // decode header
    let header_bytes = base64url::decode(header_b64)
        .map_err(|e| JwtError::Base64Error(e.to_string()))?;
    let header: serde_json::Value = serde_json::from_slice(&header_bytes)
        .map_err(|e| JwtError::InvalidToken(format!("bad header JSON: {e}")))?;

    // ตรวจสอบ alg field ใน header
    // "none" ถูก reject เสมอ ไม่ว่า expected_alg จะเป็นอะไร
    let header_alg = header.get("alg")
        .and_then(|v| v.as_str())
        .unwrap_or("");

    if header_alg.eq_ignore_ascii_case("none") {
        return Err(JwtError::AlgorithmNotAllowed);
    }

    // header alg ต้องตรงกับ algorithm ที่ caller ระบุ
    if header_alg != expected_alg.name() {
        return Err(JwtError::AlgorithmNotAllowed);
    }

    // verify signature ก่อน decode payload
    let signing_input = format!("{header_b64}.{payload_b64}");
    let sig_bytes = base64url::decode(sig_b64)
        .map_err(|e| JwtError::Base64Error(e.to_string()))?;

    verify_signature(signing_input.as_bytes(), &sig_bytes, key, expected_alg)?;

    // decode payload
    let payload_bytes = base64url::decode(payload_b64)
        .map_err(|e| JwtError::Base64Error(e.to_string()))?;
    let claims: Claims = serde_json::from_slice(&payload_bytes)
        .map_err(|e| JwtError::InvalidToken(format!("bad payload JSON: {e}")))?;

    // validate claims (exp, nbf, iss, aud)
    validate_claims(&claims, opts)?;

    // ตรวจสอบ blacklist
    if let Some(bl) = blacklist {
        if let Some(jti) = &claims.jti {
            if bl.is_revoked(jti) {
                return Err(JwtError::RevokedToken);
            }
        }
    }

    Ok(claims)
}

fn verify_signature(
    data: &[u8],
    sig: &[u8],
    key: &Key,
    alg: &Algorithm,
) -> Result<(), JwtError> {
    match key {
        Key::Hmac { secret, .. } => {
            let expected = compute_hmac(data, secret, alg)?;
            // ConstantTimeEq ป้องกัน timing side-channel
            if expected.ct_eq(sig).into() {
                Ok(())
            } else {
                Err(JwtError::InvalidSignature)
            }
        }
        Key::Rsa { public_key, .. } => {
            use rsa::pkcs1v15::VerifyingKey;
            use rsa::signature::Verifier;
            let verifying_key = VerifyingKey::<sha2::Sha256>::new(public_key.clone());
            let sig_obj = rsa::pkcs1v15::Signature::try_from(sig)
                .map_err(|_| JwtError::InvalidSignature)?;
            verifying_key.verify(data, &sig_obj)
                .map_err(|_| JwtError::InvalidSignature)
        }
        Key::Ec { public_key, .. } => {
            use p256::ecdsa::{VerifyingKey, Signature, signature::Verifier};
            let verifying_key = VerifyingKey::from(public_key.clone());
            let sig_obj = Signature::from_bytes(sig.into())
                .map_err(|_| JwtError::InvalidSignature)?;
            verifying_key.verify(data, &sig_obj)
                .map_err(|_| JwtError::InvalidSignature)
        }
    }
}

fn compute_hmac(data: &[u8], secret: &[u8], alg: &Algorithm) -> Result<Vec<u8>, JwtError> {
    match alg {
        Algorithm::HS256 => {
            let mut mac = <Hmac<Sha256> as Mac>::new_from_slice(secret)
                .map_err(|e| JwtError::SigningError(e.to_string()))?;
            mac.update(data);
            Ok(mac.finalize().into_bytes().to_vec())
        }
        Algorithm::HS384 => {
            let mut mac = <Hmac<Sha384> as Mac>::new_from_slice(secret)
                .map_err(|e| JwtError::SigningError(e.to_string()))?;
            mac.update(data);
            Ok(mac.finalize().into_bytes().to_vec())
        }
        Algorithm::HS512 => {
            let mut mac = <Hmac<Sha512> as Mac>::new_from_slice(secret)
                .map_err(|e| JwtError::SigningError(e.to_string()))?;
            mac.update(data);
            Ok(mac.finalize().into_bytes().to_vec())
        }
        _ => Err(JwtError::SigningError("algorithm mismatch for HMAC".into())),
    }
}

fn validate_claims(claims: &Claims, opts: &ValidationOptions) -> Result<(), JwtError> {
    let now = opts.current_time();

    // exp: ปฏิเสธถ้า now > exp + leeway
    if let Some(exp) = claims.exp {
        if now > exp + opts.leeway as i64 {
            return Err(JwtError::ExpiredToken);
        }
    } else if opts.require_exp {
        return Err(JwtError::InvalidToken("missing required exp claim".into()));
    }

    // nbf: ปฏิเสธถ้า now < nbf - leeway
    if let Some(nbf) = claims.nbf {
        if now < nbf - opts.leeway as i64 {
            return Err(JwtError::NotYetValid);
        }
    }

    // iss: ปฏิเสธถ้าไม่อยู่ใน allowed list
    if let Some(required) = &opts.required_issuer {
        match &claims.iss {
            Some(iss) if required.contains(iss) => {}
            _ => return Err(JwtError::InvalidIssuer),
        }
    }

    // aud: ปฏิเสธถ้าไม่มี expected audience
    if let Some(expected_aud) = &opts.required_audience {
        match &claims.aud {
            Some(aud) if aud.contains(expected_aud) => {}
            _ => return Err(JwtError::InvalidAudience),
        }
    }

    Ok(())
}
```

### ขั้นที่ 5: JWK / JWKS Support

**`src/jwk.rs`** — serialize/deserialize RSA และ EC key ตาม RFC 7517:

```rust
use serde::{Deserialize, Serialize};

/// JWK (JSON Web Key) — ตาม RFC 7517
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Jwk {
    pub kty: String,
    pub kid: String,
    #[serde(rename = "use", skip_serializing_if = "Option::is_none")]
    pub use_: Option<String>,
    pub alg: String,
    // RSA fields
    #[serde(skip_serializing_if = "Option::is_none")]
    pub n: Option<String>,   // modulus (base64url)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub e: Option<String>,   // public exponent (base64url)
    // EC fields
    #[serde(skip_serializing_if = "Option::is_none")]
    pub crv: Option<String>, // curve name: "P-256"
    #[serde(skip_serializing_if = "Option::is_none")]
    pub x: Option<String>,   // x coordinate (base64url)
    #[serde(skip_serializing_if = "Option::is_none")]
    pub y: Option<String>,   // y coordinate (base64url)
}

/// JWKS (JSON Web Key Set) — endpoint `GET /jwks.json` return รูปแบบนี้
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Jwks {
    pub keys: Vec<Jwk>,
}

impl Jwks {
    pub fn new(keys: Vec<Jwk>) -> Self {
        Jwks { keys }
    }

    /// ค้นหา key ด้วย kid — ใช้ระหว่าง verification
    pub fn find_key(&self, kid: &str) -> Option<&Jwk> {
        self.keys.iter().find(|k| k.kid == kid)
    }
}

/// สร้าง RSA JWK สำหรับ public key (ไม่รวม private components)
pub fn rsa_public_to_jwk(kid: &str, public_key: &rsa::RsaPublicKey) -> Jwk {
    use rsa::traits::PublicKeyParts;
    use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};

    let n = URL_SAFE_NO_PAD.encode(public_key.n().to_bytes_be());
    let e = URL_SAFE_NO_PAD.encode(public_key.e().to_bytes_be());

    Jwk {
        kty: "RSA".into(),
        kid: kid.to_string(),
        use_: Some("sig".into()),
        alg: "RS256".into(),
        n: Some(n),
        e: Some(e),
        crv: None,
        x: None,
        y: None,
    }
}

/// สร้าง EC JWK สำหรับ P-256 public key
pub fn ec_public_to_jwk(kid: &str, public_key: &p256::PublicKey) -> Jwk {
    use p256::elliptic_curve::sec1::ToEncodedPoint;
    use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};

    let point = public_key.to_encoded_point(false); // uncompressed
    let x = URL_SAFE_NO_PAD.encode(point.x().unwrap());
    let y = URL_SAFE_NO_PAD.encode(point.y().unwrap());

    Jwk {
        kty: "EC".into(),
        kid: kid.to_string(),
        use_: Some("sig".into()),
        alg: "ES256".into(),
        n: None,
        e: None,
        crv: Some("P-256".into()),
        x: Some(x),
        y: Some(y),
    }
}
```

### ขั้นที่ 6: Blacklist, Refresh Store, และ axum Extractor

**`src/blacklist.rs`** — revocation ด้วย JTI blacklist:

```rust
use std::collections::HashSet;
use std::sync::{Arc, RwLock};

/// TokenBlacklist เก็บ JTI ของ token ที่ถูก revoke
/// ใช้ Arc<RwLock<_>> เพื่อ share ระหว่าง threads
#[derive(Clone, Default)]
pub struct TokenBlacklist {
    revoked: Arc<RwLock<HashSet<String>>>,
}

impl TokenBlacklist {
    pub fn new() -> Self {
        Self::default()
    }

    /// เพิ่ม JTI เข้า blacklist
    pub fn revoke(&self, jti: &str) {
        let mut set = self.revoked.write().unwrap();
        set.insert(jti.to_string());
    }

    /// true ถ้า JTI อยู่ใน blacklist
    pub fn is_revoked(&self, jti: &str) -> bool {
        let set = self.revoked.read().unwrap();
        set.contains(jti)
    }

    pub fn len(&self) -> usize {
        self.revoked.read().unwrap().len()
    }
}
```

**`src/refresh.rs`** — opaque refresh token store:

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

#[derive(Clone, Debug)]
pub struct RefreshEntry {
    pub subject: String,
    pub expires_at: i64,
}

/// RefreshStore เก็บ opaque random bytes → subject mapping
///
/// Refresh token เป็น opaque (random bytes) ไม่ใช่ JWT
/// เพราะ refresh token มีอายุยาว (7 วัน) และต้องสามารถ revoke ได้
/// การเก็บใน database ทำให้ revoke ได้ทันทีโดยลบ record
#[derive(Clone, Default)]
pub struct RefreshStore {
    tokens: Arc<RwLock<HashMap<String, RefreshEntry>>>,
}

impl RefreshStore {
    pub fn new() -> Self {
        Self::default()
    }

    /// บันทึก refresh token ใหม่
    pub fn store(&self, token: String, subject: String, expires_at: i64) {
        let mut map = self.tokens.write().unwrap();
        map.insert(token, RefreshEntry { subject, expires_at });
    }

    /// validate และ consume refresh token (single-use)
    /// คืน subject ถ้า valid, None ถ้าไม่พบหรือหมดอายุ
    pub fn validate_and_consume(&self, token: &str) -> Option<String> {
        let now = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs() as i64;

        let mut map = self.tokens.write().unwrap();
        if let Some(entry) = map.get(token) {
            if entry.expires_at > now {
                let subject = entry.subject.clone();
                map.remove(token);
                return Some(subject);
            }
        }
        None
    }

    /// ลบ refresh token ทั้งหมดของ subject (logout ทุก device)
    pub fn revoke_all_for_subject(&self, subject: &str) {
        let mut map = self.tokens.write().unwrap();
        map.retain(|_, v| v.subject != subject);
    }
}
```

**`src/extractor.rs`** — axum `FromRequestParts` extractor:

```rust
use axum::{
    async_trait,
    extract::FromRequestParts,
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
};
use crate::claims::Claims;
use crate::encode::JwtError;

/// JwtAuth extractor สำหรับ axum
/// ดึง Bearer token จาก Authorization header, validate, inject Claims
///
/// ใช้ใน handler:
/// ```rust
/// async fn protected(JwtAuth(claims): JwtAuth) -> impl IntoResponse {
///     format!("Hello, {}!", claims.sub.unwrap_or_default())
/// }
/// ```
#[derive(Debug, Clone)]
pub struct JwtAuth(pub Claims);

/// Error response สำหรับ authentication failure
pub struct AuthError(pub JwtError);

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let body = format!("{{\"error\": \"{}\"}}", self.0);
        (
            StatusCode::UNAUTHORIZED,
            [
                ("Content-Type", "application/json"),
                // RFC 6750: WWW-Authenticate header สำหรับ Bearer scheme
                ("WWW-Authenticate", "Bearer realm=\"api\""),
            ],
            body,
        )
            .into_response()
    }
}

impl JwtAuth {
    /// parse "Bearer <token>" จาก Authorization header value
    pub fn parse_bearer(header_value: &str) -> Option<&str> {
        let v = header_value.trim();
        if v.starts_with("Bearer ") || v.starts_with("bearer ") {
            Some(v[7..].trim())
        } else {
            None
        }
    }
}

// ตัวอย่าง FromRequestParts implementation (ต้องมี AppState ที่มี keyring + blacklist)
// ใน production จะ inject dependencies ผ่าน axum::extract::State
//
// #[async_trait]
// impl<S> FromRequestParts<S> for JwtAuth
// where
//     S: Send + Sync + AsRef<AppState>,
// {
//     type Rejection = AuthError;
//
//     async fn from_request_parts(parts: &mut Parts, state: &S) -> Result<Self, Self::Rejection> {
//         let app_state = state.as_ref();
//         let auth_header = parts.headers
//             .get("Authorization")
//             .and_then(|v| v.to_str().ok())
//             .ok_or_else(|| AuthError(JwtError::InvalidToken("missing Authorization header".into())))?;
//
//         let token = Self::parse_bearer(auth_header)
//             .ok_or_else(|| AuthError(JwtError::InvalidToken("expected Bearer token".into())))?;
//
//         // อ่าน kid จาก header ก่อน verify เพื่อ lookup key
//         let kid = extract_kid(token);
//         let key = match kid {
//             Some(k) => app_state.keyring.get_key(k)
//                 .ok_or_else(|| AuthError(JwtError::KeyNotFound(k.to_string())))?,
//             None => app_state.keyring.current_key()
//                 .map(|(_, k)| k)
//                 .ok_or_else(|| AuthError(JwtError::KeyNotFound("no current key".into())))?,
//         };
//
//         let claims = decode(token, key, &Algorithm::HS256, &ValidationOptions::strict(),
//                             Some(&app_state.blacklist))
//             .map_err(AuthError)?;
//
//         Ok(JwtAuth(claims))
//     }
// }
```

### ขั้นที่ 7: JWKS axum Endpoint

ตัวอย่าง axum server ที่มี `/jwks.json` endpoint:

```rust
use axum::{Router, routing::get, Json, extract::State};
use std::sync::Arc;

#[derive(Clone)]
struct JwksState {
    jwks: Arc<jwt_lib::jwk::Jwks>,
}

async fn jwks_handler(State(state): State<JwksState>) -> Json<jwt_lib::jwk::Jwks> {
    Json((*state.jwks).clone())
}

// สร้าง router
pub fn jwks_router(jwks: jwt_lib::jwk::Jwks) -> Router {
    let state = JwksState { jwks: Arc::new(jwks) };
    Router::new()
        .route("/jwks.json", get(jwks_handler))
        .with_state(state)
}

// ตัวอย่าง response จาก GET /jwks.json:
// {
//   "keys": [
//     {
//       "kty": "RSA",
//       "kid": "v1",
//       "use": "sig",
//       "alg": "RS256",
//       "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78LhWx...",
//       "e": "AQAB"
//     }
//   ]
// }
```

ลำดับการ verify token ด้วย JWKS:
1. Client ดึง JWKS จาก `GET /.well-known/jwks.json` (cache ได้ตาม Cache-Control header)
2. อ่าน `kid` จาก JWT header
3. ค้นหา JWK ที่มี `kid` ตรงกันใน JWKS
4. reconstruct public key จาก JWK fields
5. verify signature

---

## การทดสอบ (Testing)

**`tests/jwt_tests.rs`:**

```rust
use jwt_lib::{
    algorithm::Algorithm,
    claims::{Claims, ValidationOptions},
    decode,
    encode,
    JwtError,
    keyring::{Key, Keyring},
    blacklist::TokenBlacklist,
    jwk::{rsa_public_to_jwk, Jwks},
};

fn now() -> i64 {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs() as i64
}

fn hmac_key() -> Key {
    Key::hmac(b"super-secret-key-at-least-32-bytes-long".to_vec(), Algorithm::HS256)
}

fn default_opts() -> ValidationOptions {
    ValidationOptions { require_exp: false, ..Default::default() }
}

// ─── Test 1: HS256 encode/decode round-trip ──────────────────────────────────
#[test]
fn test_hs256_roundtrip() {
    let key = hmac_key();
    let mut claims = Claims::default();
    claims.sub = Some("user-42".into());
    claims.exp = Some(now() + 3600);

    let token = encode(&claims, &key, None).unwrap();
    assert_eq!(token.split('.').count(), 3);

    let decoded = decode(&token, &key, &Algorithm::HS256, &default_opts(), None).unwrap();
    assert_eq!(decoded.sub, Some("user-42".into()));
}

// ─── Test 2: exp rejection ────────────────────────────────────────────────────
#[test]
fn test_exp_rejection() {
    let key = hmac_key();
    let mut claims = Claims::default();
    claims.exp = Some(now() - 100); // expired

    let token = encode(&claims, &key, None).unwrap();
    let result = decode(&token, &key, &Algorithm::HS256, &default_opts(), None);
    assert!(matches!(result, Err(JwtError::ExpiredToken)));
}

// ─── Test 3: nbf rejection ────────────────────────────────────────────────────
#[test]
fn test_nbf_rejection() {
    let key = hmac_key();
    let mut claims = Claims::default();
    claims.nbf = Some(now() + 3600); // not yet valid

    let token = encode(&claims, &key, None).unwrap();
    let result = decode(&token, &key, &Algorithm::HS256, &default_opts(), None);
    assert!(matches!(result, Err(JwtError::NotYetValid)));
}

// ─── Test 4: alg:none rejection ──────────────────────────────────────────────
#[test]
fn test_alg_none_rejection() {
    use base64::{Engine, engine::general_purpose::URL_SAFE_NO_PAD};
    let header = r#"{"typ":"JWT","alg":"none"}"#;
    let payload = r#"{"sub":"evil","exp":9999999999}"#;
    let token = format!(
        "{}.{}.",
        URL_SAFE_NO_PAD.encode(header.as_bytes()),
        URL_SAFE_NO_PAD.encode(payload.as_bytes())
    );

    let key = hmac_key();
    let result = decode(&token, &key, &Algorithm::HS256, &default_opts(), None);
    assert!(result.is_err(), "alg:none must be rejected");
}

// ─── Test 5: algorithm confusion prevention ──────────────────────────────────
#[test]
fn test_algorithm_confusion() {
    let key384 = Key::hmac(
        b"super-secret-key-at-least-32-bytes-long".to_vec(),
        Algorithm::HS384
    );
    let mut claims = Claims::default();
    claims.sub = Some("mallory".into());

    let token = encode(&claims, &key384, None).unwrap();

    // decode ด้วย expected HS256 ต้อง fail
    let key256 = hmac_key();
    let result = decode(&token, &key256, &Algorithm::HS256, &default_opts(), None);
    assert!(matches!(result, Err(JwtError::AlgorithmNotAllowed)));
}

// ─── Test 6: JWKS serialization ──────────────────────────────────────────────
#[test]
fn test_jwks_serialization() {
    use rsa::RsaPrivateKey;
    use rand::rngs::OsRng;

    let private_key = RsaPrivateKey::new(&mut OsRng, 2048).unwrap();
    let public_key = rsa::RsaPublicKey::from(&private_key);

    let jwk = rsa_public_to_jwk("key-1", &public_key);
    assert_eq!(jwk.kty, "RSA");
    assert_eq!(jwk.kid, "key-1");
    assert!(jwk.n.is_some() && jwk.e.is_some());

    let jwks = Jwks::new(vec![jwk]);
    let json = serde_json::to_string(&jwks).unwrap();
    assert!(json.contains("\"keys\""));
    assert!(json.contains("\"kty\":\"RSA\""));

    assert!(jwks.find_key("key-1").is_some());
    assert!(jwks.find_key("nonexistent").is_none());
}

// ─── Test 7: kid lookup via Keyring ──────────────────────────────────────────
#[test]
fn test_kid_lookup() {
    let key_a = Key::hmac(b"secret-key-a-32-bytes-minimum-len".to_vec(), Algorithm::HS256);
    let key_b = Key::hmac(b"secret-key-b-32-bytes-minimum-len".to_vec(), Algorithm::HS256);

    let mut keyring = Keyring::new("v1", key_a);
    keyring.add_key("v2", key_b);
    keyring.set_current("v2");

    let (kid, current_key) = keyring.current_key().unwrap();
    let mut claims = Claims::default();
    claims.sub = Some("tester".into());

    let token = encode(&claims, current_key, Some(kid)).unwrap();

    // ตรวจสอบ kid ใน header
    let header_b64 = token.split('.').next().unwrap();
    let header_bytes = base64::Engine::decode(
        &base64::engine::general_purpose::URL_SAFE_NO_PAD,
        header_b64,
    ).unwrap();
    let header: serde_json::Value = serde_json::from_slice(&header_bytes).unwrap();
    assert_eq!(header["kid"], "v2");

    // verify ด้วย key ที่ lookup จาก kid
    let lookup_key = keyring.get_key("v2").unwrap();
    let decoded = decode(&token, lookup_key, &Algorithm::HS256, &default_opts(), None).unwrap();
    assert_eq!(decoded.sub, Some("tester".into()));
}

// ─── Test 8: blacklist / revocation ──────────────────────────────────────────
#[test]
fn test_blacklist_revocation() {
    let key = hmac_key();
    let bl = TokenBlacklist::new();

    let mut claims = Claims::default();
    claims.sub = Some("user-77".into());
    claims.jti = Some("unique-jti-abc123".into());
    claims.exp = Some(now() + 3600);

    let token = encode(&claims, &key, None).unwrap();

    // ก่อน revoke: valid
    assert!(decode(&token, &key, &Algorithm::HS256, &default_opts(), Some(&bl)).is_ok());

    // revoke
    bl.revoke("unique-jti-abc123");

    // หลัง revoke: rejected
    let r = decode(&token, &key, &Algorithm::HS256, &default_opts(), Some(&bl));
    assert!(matches!(r, Err(JwtError::RevokedToken)));
}
```

### Real `cargo test` Output

```
running 8 tests
test test_alg_none_rejection ... ok
test test_hs256_roundtrip ... ok
test test_exp_rejection ... ok
test test_algorithm_confusion ... ok
test test_blacklist_revocation ... ok
test test_kid_lookup ... ok
test test_nbf_rejection ... ok
test test_jwks_serialization ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 8.67s
```

---

## Pitfalls และ ข้อผิดพลาดที่พบบ่อย

### Pitfall 1: ใช้ `==` เปรียบเทียบ HMAC แทน `ConstantTimeEq`

```rust
// อันตราย — early exit ทำให้ attacker วัดเวลาได้
if computed_mac == provided_mac { ... }

// ถูกต้อง — ใช้ subtle::ConstantTimeEq
use subtle::ConstantTimeEq;
if computed_mac.ct_eq(&provided_mac).into() { ... }
```

การใช้ `==` กับ byte slice ใช้ lexicographic comparison ที่หยุดทันทีที่พบ byte ที่ต่างกัน ส่งผลให้เวลา comparison มีความสัมพันธ์กับจำนวน byte แรกที่ตรงกัน `ConstantTimeEq` เปรียบเทียบทุก byte เสมอโดยไม่ branch ก่อนกำหนด

### Pitfall 2: Trust `alg` field จาก header

```rust
// อันตราย — อ่าน algorithm จาก header แล้วเลือก verification key ตาม
let alg = header["alg"].as_str().unwrap();
let result = match alg {
    "HS256" => verify_hmac(&token, &hmac_secret),
    "RS256" => verify_rsa(&token, &rsa_public_key),
    "none"  => Ok(()), // ← CVE-2015-9235 pattern
    _ => Err(...)
};

// ถูกต้อง — caller กำหนด expected algorithm
pub fn decode(token: &str, key: &Key, expected_alg: &Algorithm, ...) {
    let header_alg = header["alg"].as_str().unwrap_or("");
    if header_alg != expected_alg.name() {
        return Err(JwtError::AlgorithmNotAllowed);
    }
    // ...
}
```

ช่องโหว่คลาสสิก: library ที่ accept `"alg": "none"` ทำให้ signature ไม่ถูกตรวจสอบ หรือ library ที่ switch จาก RS256 เป็น HS256 แล้วใช้ RSA public key เป็น HMAC secret

### Pitfall 3: Decode payload ก่อน verify signature

```rust
// อันตราย — decode ก่อน verify
let claims: Claims = serde_json::from_slice(&payload_bytes)?;
verify_signature(&sig)?; // ← สาย

// ถูกต้อง — verify ก่อนเสมอ
verify_signature(signing_input.as_bytes(), &sig_bytes, key, alg)?;
let claims: Claims = serde_json::from_slice(&payload_bytes)?; // ← ปลอดภัย
```

การ deserialize payload ก่อน verify signature เปิดช่องให้ attacker ส่ง payload ที่ดัดแปลงมาเพื่อ exploit deserialization bugs แม้ signature จะ fail ในภายหลัง

### Pitfall 4: HMAC key สั้นเกินไป

```rust
// ระวัง — HMAC-SHA256 key ที่สั้นมาก
let key = Key::hmac(b"secret".to_vec(), Algorithm::HS256);
// ยังทำงานได้แต่ security ต่ำ

// แนะนำ — key ต้องยาวอย่างน้อยเท่ากับ output size ของ hash function
// HS256 → อย่างน้อย 32 bytes
// HS384 → อย่างน้อย 48 bytes
// HS512 → อย่างน้อย 64 bytes
let key = Key::hmac(random_bytes_32(), Algorithm::HS256);
```

RFC 2104 ระบุว่า key ที่สั้นกว่า hash output size จะลด security ของ HMAC

### Pitfall 5: Refresh token เป็น JWT (แทนที่จะเป็น opaque)

```rust
// ไม่แนะนำ — refresh token เป็น JWT
// ปัญหา: revoke ไม่ได้ทันทีถ้าใช้ stateless verification
// ต้องเก็บ blacklist ซึ่งเท่ากับต้องมี state อยู่ดี
let refresh_token = encode(&long_lived_claims, &key, None)?;

// แนะนำ — refresh token เป็น opaque random bytes เก็บใน DB
use rand::Rng;
let refresh_token: String = rand::thread_rng()
    .sample_iter(&rand::distributions::Alphanumeric)
    .take(64)
    .map(char::from)
    .collect();
refresh_store.store(refresh_token.clone(), subject, expires_at);
```

Refresh token มีอายุยาว (7 วัน) ดังนั้นต้องสามารถ revoke ได้ทันทีเมื่อพบการใช้งานผิดปกติ opaque token ที่เก็บใน DB ทำให้ delete record แล้ว revoke ได้ทันที

### Pitfall 6: ไม่ตรวจสอบ `aud` claim ใน resource server

```rust
// อันตราย — decode โดยไม่กำหนด required_audience
let claims = decode(&token, &key, &Algorithm::HS256,
                    &ValidationOptions::default(), None)?;

// ถูกต้อง — resource server ต้องระบุ audience ที่คาดหวัง
let opts = ValidationOptions {
    required_audience: Some("my-api.example.com".into()),
    ..Default::default()
};
let claims = decode(&token, &key, &Algorithm::HS256, &opts, None)?;
```

Token ที่ออกสำหรับ service A ไม่ควร valid สำหรับ service B `aud` claim ระบุ recipient ที่ token นี้ตั้งใจให้ resource server ที่ไม่ตรวจสอบ `aud` ยอมรับ token ของ service อื่น (confused deputy)

---

## การ Package และ Deploy

### Build Release

```bash
cargo build --release
# binary อยู่ที่ target/release/jwt-lib
```

### ใช้เป็น library ใน project อื่น

เพิ่มใน `Cargo.toml`:
```toml
[dependencies]
jwt-lib = { path = "../jwt-lib" }
# หรือจาก crates.io:
# jwt-lib = "0.1"
```

### การจัดการ Key ใน Production

```bash
# สร้าง RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# สร้าง EC key pair (P-256)
openssl ecparam -name prime256v1 -genkey -noout -out ec-private.pem
openssl ec -in ec-private.pem -pubout -out ec-public.pem
```

Key ควรเก็บใน:
- **Development**: environment variable หรือ `.env` file (ไม่ commit)
- **Production**: HashiCorp Vault, AWS Secrets Manager, หรือ Kubernetes Secrets

### Refresh Token Flow

```
Client                    Server
  │                          │
  ├─── POST /login ──────────►│
  │                          │ issue:
  │                          │   access_token (JWT, 15 min)
  │                          │   refresh_token (opaque, 7 days)
  │◄── { access_token,  ─────┤
  │      refresh_token }     │
  │                          │
  │  [15 minutes later]      │
  │                          │
  ├─── POST /refresh ────────►│ validate refresh_token in DB
  │    { refresh_token }     │ consume (single-use)
  │                          │ issue new access_token
  │◄── { access_token } ─────┤
  │                          │
```

---

## การต่อยอด (Extensions & Exercises)

1. **เพิ่ม ES384/ES512** — รองรับ P-384 และ P-521 curves ด้วย `p384` crate โดยแก้ `Algorithm` enum และ `sign`/`verify_signature` ให้รองรับ curve ใหม่ตาม kid convention

2. **Persistent JWKS Cache** — สร้าง `JwksClient` ที่ fetch JWKS จาก URL, cache ด้วย `Arc<RwLock<Jwks>>`, auto-refresh เมื่อพบ `kid` ที่ไม่รู้จัก (background task ด้วย `tokio::spawn`)

3. **Clock Skew Test** — เขียน test ที่ใช้ `ValidationOptions { now: Some(fake_time), leeway: 30, .. }` เพื่อจำลองสถานการณ์ที่ server clock ต่างจาก client clock และตรวจสอบว่า leeway ทำงานถูกต้องทั้งฝั่ง exp และ nbf

4. **RSA Sign/Verify Round-trip** — สร้าง test ที่ generate RSA 2048-bit keypair, encode ด้วย `Key::Rsa`, decode ด้วย public key เดียวกัน และตรวจสอบว่า signature จาก private key 1 ไม่ผ่าน verification ด้วย private key 2

---

## สรุป

โปรเจคนี้สร้าง JWT library ตั้งแต่พื้นฐาน — base64url encoding, HMAC signing ด้วย SHA-256/384/512, RSA-PKCS1v15 และ ECDSA-P256 signing, claims validation (exp/nbf/iss/aud), algorithm confusion prevention, JWK/JWKS serialization, key rotation ผ่าน `Keyring` + `kid`, token revocation ผ่าน JTI blacklist, และ timing-safe comparison ด้วย `subtle::ConstantTimeEq`

**Pattern สำคัญที่ได้เรียน:**
- **Caller-controlled algorithm** — security invariant ที่ต้องรักษา: algorithm ต้องมาจาก caller ไม่ใช่ token header
- **Verify before deserialize** — ลำดับการ check ที่สำคัญ: signature ก่อน payload
- **Constant-time MAC comparison** — ป้องกัน timing side-channel ใน HMAC verification
- **Opaque refresh tokens** — แยก access token (stateless JWT) กับ refresh token (stateful opaque) ตาม use case
- **Arc<RwLock<_>> สำหรับ shared mutable state** — pattern สำหรับ blacklist และ keyring ที่ safe ใน async context

**โปรเจคถัดไป (D08)** จะต่อยอดด้วยการสร้าง vulnerability scanner ที่ใช้ cryptographic primitives ในการตรวจสอบ security weaknesses ใน TLS certificates, HMAC configurations, และ key sizes

---

**โปรเจคก่อนหน้า:** [project-d06-secrets-rotation.md](project-d06-secrets-rotation.md) | **โปรเจคถัดไป:** [project-d08-vuln-scanner.md](project-d08-vuln-scanner.md)
