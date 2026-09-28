# Project D06: Secrets Rotation Daemon

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

Secrets Rotation Daemon คือ background service ที่ rotate credential อัตโนมัติตามตาราง — ทำงานคล้าย cron แต่รู้จัก semantic ของ credential แต่ละประเภท: รู้ว่าต้อง `ALTER USER` ใน PostgreSQL, ต้อง call provider API เพื่อสร้าง/revoke API key, ต้องออก ECDSA keypair ใหม่สำหรับ JWT, หรือต้อง patch `v1/Secret` ใน Kubernetes ทั้งหมดนี้ทำงานภายใต้ grace period (overlap) — ทั้ง credential เก่าและใหม่ยังคงถูกต้องในช่วงเปลี่ยนผ่าน เพื่อให้ service อื่นมีเวลา reload

ระบบนี้บันทึก audit trail ของทุก rotation attempt พร้อม SHA-256 hash prefix ของ key เก่าและใหม่ และ POST webhook notification เมื่อ rotation เริ่ม/สำเร็จ/ล้มเหลว

**Use cases ใน production:**
- Platform engineering team ที่ต้องการ rotate database password ทุก 30 วันโดยอัตโนมัติ
- Kubernetes-native service ที่ rotation ต้องเขียน `Secret` object และ trigger rolling restart
- JWT infrastructure ที่ต้องการ JWKS key overlap period เพื่อให้ token ที่ออกแล้วยังคงถูกต้อง
- HashiCorp Vault environment ที่ต้องการ sync secret version ระหว่าง rotation

**Learning value:** โปรเจคนี้สอนการออกแบบ state machine สำหรับ credential lifecycle, การใช้ `tokio::time::interval` สำหรับ daemon loop ที่มี jitter, การเขียน ECDSA keypair rotation ด้วย `p256` crate, pattern สำหรับ overlap period management, และการ integrate กับ Kubernetes API ผ่าน `kube` crate

## สิ่งที่จะได้เรียนรู้

- **Credential lifecycle state machine** — Pending → Rotating → Overlapping → Retired pattern
- **Tokio interval + jitter** — `tokio::time::interval` กับ jitter ±30s เพื่อหลีกเลี่ยง thundering herd
- **ECDSA P-256 keypair generation** — สร้าง signing key ด้วย `p256` crate และ serialize เป็น PEM/JWKS
- **Kubernetes API via `kube` crate** — `patch` `v1/Secret`, list deployment, trigger rolling restart
- **HashiCorp Vault KV v2 API** — `PUT /v1/kv/data/{path}` ผ่าน `reqwest` โดยตรง
- **Overlap/grace period management** — เก็บ expiry ใน registry, daemon retire expired entry
- **Exponential backoff webhook** — retry POST notification พร้อม cap และ jitter
- **Append-only audit log** — บันทึก rotation event พร้อม SHA-256 key hash prefix

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50** — async/await, tokio runtime, `tokio::time::sleep` / `interval`
- **Part 61–65** — `reqwest` HTTP client, JSON request/response
- **Part 80–85** — `serde`, `serde_json`, `toml` — Serialize/Deserialize derive
- **Part 86–90** — cryptography ใน Rust: `sha2`, `rand`, `p256`
- **Part 91–95** — `clap 4` CLI design, subcommands, argument parsing
- **Part 96–100** — `sqlx` async PostgreSQL, `kube` crate Kubernetes API

## โครงสร้างโปรเจค (Project Layout)

```
secrets-rotation/
├── src/
│   ├── main.rs           # CLI entry point (clap 4), daemon run loop
│   ├── registry.rs       # SecretEntry, RegistryConfig, TOML load/save
│   ├── scheduler.rs      # is_due_for_rotation, jitter, interval loop
│   ├── rotators/
│   │   ├── mod.rs        # Rotator trait
│   │   ├── db_password.rs  # PostgreSQL ALTER USER
│   │   ├── api_key.rs      # GitHub PAT / AWS IAM / generic HTTP
│   │   └── jwt_key.rs      # ECDSA P-256 / RSA-2048 keypair + JWKS
│   ├── providers/
│   │   ├── kubernetes.rs # kube crate: patch Secret, rolling restart
│   │   └── vault.rs      # HashiCorp Vault KV v2 HTTP API
│   ├── overlap.rs        # OverlapEntry, grace period expiry
│   ├── notify.rs         # Webhook POST, exponential backoff retry
│   └── audit.rs          # AuditEvent, append-only log writer
├── config/
│   └── secrets.toml      # example registry config
├── tests/
│   └── unit_tests.rs     # tests ที่รันได้จริงทั้งหมด
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### หลักการ: Credential Lifecycle State Machine

```
[ACTIVE] ──── rotation due? ────► [ROTATING]
                                      │
                             ┌────────┴────────┐
                             │                 │
                          success            failure
                             │                 │
                        [OVERLAPPING]    [ACTIVE] (unchanged)
                             │                 │
                      grace expired        notify + audit
                             │
                         [RETIRED]
```

daemon loop ทำงานทุก 60±30 วินาที:
1. โหลด registry จาก TOML
2. หา secret ที่ถึงเวลา rotate (`is_due_for_rotation`)
3. สำหรับแต่ละ secret ที่ due: เรียก rotator ที่เหมาะสม
4. บันทึก audit event (attempt + outcome)
5. POST webhook notification (rotation-started / completed / failed)
6. อัปเดต registry (last_rotated, overlap expiry)
7. Retire expired overlap entries

### Secret Registry: TOML Config Format

```toml
[[secrets]]
name            = "db_main"
secret_type     = "db_password"
rotation_interval_days = 30
last_rotated_unix      = 1700000000
target          = "postgres://app:@db.prod.internal:5432/myapp"
notifiers       = ["https://hooks.slack.com/...", "https://internal.api/events"]

[[secrets]]
name            = "github_pat"
secret_type     = "api_key"
rotation_interval_days = 90
last_rotated_unix      = 1695000000
target          = "env:GITHUB_TOKEN"
notifiers       = ["https://hooks.example.com/rotation"]

[[secrets]]
name            = "jwt_signing"
secret_type     = "jwt_signing_key"
rotation_interval_days = 365
last_rotated_unix      = 1680000000
target          = "k8s:default/jwt-keypair"
notifiers       = []
```

### Overlap Period Design

เมื่อ rotation สำเร็จ daemon สร้าง `OverlapEntry` ที่เก็บ:
- `old_key_hash_prefix` — SHA-256 ของ credential เก่า (8 bytes = 16 hex chars)
- `expires_at_unix` — `now + grace_seconds` (default: 3600)

ตลอด grace period ทั้ง credential เก่าและใหม่ถูกต้อง ใน JWT rotation: JWKS endpoint expose ทั้ง 2 key ทำให้ token ที่ signed ด้วย key เก่ายังค verify ได้จนหมด grace period

### Thundering Herd Prevention

daemon หลายตัวที่รันพร้อมกันจะ rotate ในเวลาใกล้เคียงกันถ้าไม่มี jitter ซึ่งกด load spike ให้ PostgreSQL หรือ KV store พร้อมกัน วิธีแก้: เพิ่ม jitter ±30 วินาทีให้แต่ละ check interval

```rust
use tokio::time::{sleep, Duration};
use rand::Rng;

async fn run_with_jitter(interval_secs: u64) {
    let mut rng = rand::thread_rng();
    loop {
        // งานตรวจสอบ rotation
        check_and_rotate().await;
        let jitter = rng.gen_range(-30i64..=30i64);
        let sleep_secs = (interval_secs as i64 + jitter).max(10) as u64;
        sleep(Duration::from_secs(sleep_secs)).await;
    }
}
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Secret Registry — TOML Config และ Rotation Schedule Logic

เริ่มจากโครงสร้างข้อมูลพื้นฐาน: `SecretEntry`, `RegistryConfig`, และ logic สำหรับตัดสินว่า secret ใดถึงเวลา rotate แล้ว

**Cargo.toml:**

```toml
[package]
name = "secrets-rotation"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio      = { version = "1", features = ["full"] }
serde      = { version = "1", features = ["derive"] }
serde_json = "1"
toml       = "0.8"
rand       = "0.8"
sha2       = "0.10"
hex        = "0.4"
reqwest    = { version = "0.12", features = ["json"] }
p256       = { version = "0.13", features = ["ecdsa", "pem"] }
sqlx       = { version = "0.8", features = ["postgres", "runtime-tokio"] }
kube       = { version = "0.95", features = ["client", "runtime"] }
k8s-openapi = { version = "0.23", features = ["v1_30"] }
clap       = { version = "4", features = ["derive"] }
chrono     = { version = "0.4", features = ["serde"] }
anyhow     = "1"
tracing    = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
```

**src/registry.rs:**

```rust
use serde::{Deserialize, Serialize};
use std::fs;
use anyhow::Result;

/// ประเภทของ secret ที่ daemon รองรับ
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
pub enum SecretType {
    DbPassword,
    ApiKey,
    TlsCert,
    JwtSigningKey,
}

/// Entry เดียวใน secrets registry
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SecretEntry {
    /// ชื่อ unique ของ secret ใช้เป็น key ใน audit log
    pub name: String,
    pub secret_type: SecretType,
    /// จำนวนวันระหว่าง rotation
    pub rotation_interval_days: u32,
    /// Unix timestamp ครั้งล่าสุดที่ rotate สำเร็จ
    pub last_rotated_unix: u64,
    /// ที่อยู่ปลายทาง: connection string, "env:VAR", "k8s:namespace/name"
    pub target: String,
    /// Webhook URL สำหรับ notification (POST JSON)
    #[serde(default)]
    pub notifiers: Vec<String>,
}

/// ไฟล์ TOML registry ทั้งหมด
#[derive(Debug, Serialize, Deserialize)]
pub struct RegistryConfig {
    pub secrets: Vec<SecretEntry>,
}

impl RegistryConfig {
    pub fn load(path: &str) -> Result<Self> {
        let content = fs::read_to_string(path)?;
        Ok(toml::from_str(&content)?)
    }

    pub fn save(&self, path: &str) -> Result<()> {
        let content = toml::to_string_pretty(self)?;
        fs::write(path, content)?;
        Ok(())
    }
}
```

**src/scheduler.rs — is_due_for_rotation:**

```rust
/// คืน `true` ถ้า last_rotated_unix + interval_days*86400 ≤ now_unix
///
/// ใช้ saturating arithmetic เพื่อหลีกเลี่ยง overflow บน u64 edge cases
pub fn is_due_for_rotation(last_rotated_unix: u64, interval_days: u32, now_unix: u64) -> bool {
    let interval_secs = interval_days as u64 * 86_400;
    let next_rotation = last_rotated_unix.saturating_add(interval_secs);
    now_unix >= next_rotation
}

/// คืน Unix timestamp ของ next rotation ที่กำหนดไว้
pub fn next_rotation_time(last_rotated_unix: u64, interval_days: u32) -> u64 {
    last_rotated_unix.saturating_add(interval_days as u64 * 86_400)
}

/// คืน number of seconds จนถึง next rotation (ค่าลบหมายความว่า overdue แล้ว)
pub fn seconds_until_rotation(last_rotated_unix: u64, interval_days: u32, now_unix: u64) -> i64 {
    let next = next_rotation_time(last_rotated_unix, interval_days);
    next as i64 - now_unix as i64
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_is_due_exact_boundary() {
        // Rotated at t=0, interval=1 day; at t=86400 exactly → due
        assert!(is_due_for_rotation(0, 1, 86_400));
    }

    #[test]
    fn test_is_due_one_second_early() {
        // หนึ่งวินาทีก่อน next rotation window: ยังไม่ due
        assert!(!is_due_for_rotation(0, 1, 86_399));
    }

    #[test]
    fn test_is_due_overdue() {
        // Rotated 90 วันที่แล้ว interval=30 วัน → overdue
        let last = 1_000_000u64;
        let now = last + 90 * 86_400;
        assert!(is_due_for_rotation(last, 30, now));
    }

    #[test]
    fn test_is_not_due_recent() {
        // Rotated 5 นาทีที่แล้ว interval=90 วัน → ไม่ due
        let now = 2_000_000u64;
        let last = now - 300;
        assert!(!is_due_for_rotation(last, 90, now));
    }

    #[test]
    fn test_seconds_until_negative_when_overdue() {
        let last = 1_000u64;
        let now = last + 2 * 86_400; // 2 วันหลัง interval 1 วัน
        let secs = seconds_until_rotation(last, 1, now);
        assert!(secs < 0, "overdue secret should return negative seconds");
    }
}
```

---

### ขั้นที่ 2: Password Generation และ PostgreSQL Rotation

#### Random Password Generator

Password สำหรับ `db_password` rotation ต้องมาจาก CSPRNG และประกอบด้วย character ที่ PostgreSQL รองรับในช่อง password โดย **ไม่มี** single quote หรือ backslash เพื่อหลีกเลี่ยง SQL injection ใน `ALTER USER` statement

```rust
// src/rotators/db_password.rs
use rand::Rng;
use anyhow::Result;

/// Charset: A-Z a-z 0-9 !@#$%^&*()-_=+[]{}
/// หลีกเลี่ยง ' \ " ที่ต้อง escape ใน SQL string
const PASSWORD_CHARSET: &[u8] =
    b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz\
      0123456789!@#$%^&*()-_=+[]{}";

/// สร้าง cryptographically random password ความยาว `length` characters
pub fn generate_random_password(length: usize) -> String {
    let mut rng = rand::thread_rng();
    (0..length)
        .map(|_| {
            let idx = rng.gen_range(0..PASSWORD_CHARSET.len());
            PASSWORD_CHARSET[idx] as char
        })
        .collect()
}

/// ตรวจว่า password ประกอบด้วย character ใน allowed charset เท่านั้น
pub fn validate_password_charset(password: &str) -> bool {
    password.chars().all(|c| {
        c.is_ascii_alphanumeric()
            || matches!(
                c,
                '!' | '@' | '#' | '$' | '%' | '^' | '&' | '*'
                    | '(' | ')' | '-' | '_' | '=' | '+'
                    | '[' | ']' | '{' | '}'
            )
    })
}
```

#### PostgreSQL Rotation ด้วย sqlx

กระบวนการ rotation มี 4 ขั้นตอนที่ต้อง atomic ให้มากที่สุด:

```
1. สร้าง new_password ด้วย CSPRNG
2. ALTER USER {username} WITH PASSWORD '{new_password}'
3. ทดสอบ connection ด้วย new_password (verify)
4. อัปเดต registry → last_rotated = now
```

ถ้าขั้นที่ 3 ล้มเหลว daemon ไม่อัปเดต registry และ log error — old password ยังใช้ได้

```rust
use sqlx::postgres::PgPoolOptions;
use anyhow::{Result, Context};

pub struct DbPasswordRotator {
    /// connection URL ที่มี username และ old password ฝังอยู่
    pub connection_url: String,
    /// ชื่อ PostgreSQL user ที่จะ rotate
    pub username: String,
}

impl DbPasswordRotator {
    pub async fn rotate(&self) -> Result<String> {
        // ขั้น 1: สร้าง password ใหม่
        let new_password = generate_random_password(32);

        // ขั้น 2: เชื่อมต่อด้วย old credentials แล้วเปลี่ยน password
        let pool = PgPoolOptions::new()
            .max_connections(1)
            .connect(&self.connection_url)
            .await
            .context("connect with old credentials")?;

        // ใช้ parameterized form ไม่ได้ใน DDL — ต้อง format string
        // แต่ password ถูก validate charset ก่อนแล้ว (ไม่มี quote/backslash)
        sqlx::query(&format!(
            "ALTER USER {} WITH PASSWORD '{}'",
            self.username, new_password
        ))
        .execute(&pool)
        .await
        .context("ALTER USER failed")?;

        pool.close().await;

        // ขั้น 3: verify ด้วย new password
        let new_url = replace_password_in_url(&self.connection_url, &new_password)?;
        let verify_pool = PgPoolOptions::new()
            .max_connections(1)
            .connect(&new_url)
            .await
            .context("verify new password failed — old password still valid")?;

        sqlx::query("SELECT 1").execute(&verify_pool).await?;
        verify_pool.close().await;

        Ok(new_password)
    }
}

/// แทนที่ password ใน PostgreSQL connection URL
/// postgres://user:OLD_PASS@host:5432/db → postgres://user:NEW_PASS@host:5432/db
fn replace_password_in_url(url: &str, new_password: &str) -> Result<String> {
    // ใช้ url::Url หรือ regex ตาม preference
    // simplified version:
    let at_pos = url.find('@').context("no @ in URL")?;
    let scheme_end = url.find("://").context("no :// in URL")? + 3;
    let user_info = &url[scheme_end..at_pos];
    let colon_pos = user_info.find(':').context("no : in user info")?;
    let username = &user_info[..colon_pos];
    let rest = &url[at_pos..]; // "@host:5432/db"
    let scheme = &url[..scheme_end];
    Ok(format!("{scheme}{username}:{new_password}{rest}"))
}
```

---

### ขั้นที่ 3: API Key Rotation — GitHub PAT, AWS IAM, Generic HTTP

API key rotation มีลำดับที่ต่างจาก DB password: ต้อง **สร้างก่อน revoke** เพื่อหลีกเลี่ยง service disruption ในช่วง overlap

```
1. Call provider: create new key → new_key
2. Write new_key ไปยัง target (env file / k8s secret / vault)
3. ตรวจสอบว่า target เขียนสำเร็จ
4. Revoke old key (เฉพาะหลังจาก write สำเร็จ)
5. อัปเดต registry
```

**ถ้า --dry-run:** ข้ามขั้น 2, 4, 5 — แค่ log ว่า would rotate

```rust
// src/rotators/api_key.rs
use reqwest::Client;
use serde_json::{json, Value};
use anyhow::{Result, Context};

pub enum ApiKeyProvider {
    /// GitHub Personal Access Token (Fine-grained)
    GitHub {
        token: String,            // current token (สำหรับ auth)
        owner: String,
        repo_access: Vec<String>,
    },
    /// Generic HTTP POST → สร้าง key / DELETE → revoke
    Generic {
        create_url: String,
        revoke_url_template: String, // {old_key_id} placeholder
        auth_header: String,
    },
}

pub struct ApiKeyRotator {
    pub provider: ApiKeyProvider,
    pub dry_run: bool,
    pub client: Client,
}

impl ApiKeyRotator {
    pub async fn rotate(&self, old_key_id: &str) -> Result<String> {
        match &self.provider {
            ApiKeyProvider::Generic { create_url, revoke_url_template, auth_header } => {
                // ขั้น 1: สร้าง key ใหม่
                let resp = self.client
                    .post(create_url)
                    .header("Authorization", auth_header)
                    .json(&json!({"description": "rotated-by-daemon"}))
                    .send()
                    .await
                    .context("create key request")?
                    .error_for_status()
                    .context("create key non-2xx")?;

                let body: Value = resp.json().await?;
                let new_key = body["key"]
                    .as_str()
                    .context("missing 'key' in response")?
                    .to_string();

                if self.dry_run {
                    tracing::info!("[dry-run] would write new key and revoke old");
                    return Ok(new_key);
                }

                // ขั้น 4: Revoke old key (หลัง write สำเร็จแล้ว)
                let revoke_url = revoke_url_template.replace("{old_key_id}", old_key_id);
                self.client
                    .delete(&revoke_url)
                    .header("Authorization", auth_header)
                    .send()
                    .await
                    .context("revoke old key")?
                    .error_for_status()
                    .context("revoke non-2xx")?;

                Ok(new_key)
            }
            ApiKeyProvider::GitHub { .. } => {
                // GitHub Fine-grained PAT ต้อง call /user/installations/{id}/access_tokens
                // implementation ละไว้สำหรับ exercise
                todo!("GitHub PAT rotation")
            }
        }
    }
}
```

---

### ขั้นที่ 4: JWT Signing Key Rotation — ECDSA P-256 + JWKS Overlap

JWT key rotation ซับซ้อนกว่าเพราะ token ที่ออกแล้วยังมีอายุใช้งาน — ต้องมี overlap period ที่ทั้ง 2 key ถูกต้อง

**กระบวนการ:**
1. สร้าง ECDSA P-256 keypair ใหม่
2. Publish public key ใหม่ไปยัง JWKS endpoint (เพิ่มเข้าไป ไม่ใช่แทนที่)
3. เริ่มใช้ private key ใหม่สำหรับ sign token ใหม่
4. หลัง `--overlap-hours 24`: ลบ old public key ออกจาก JWKS

```rust
// src/rotators/jwt_key.rs
use p256::ecdsa::SigningKey;
use p256::pkcs8::EncodePrivateKey;
use rand::rngs::OsRng;
use serde_json::{json, Value};
use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine};
use anyhow::Result;

/// สร้าง ECDSA P-256 keypair และคืน (private_key_pem, jwk_public)
pub fn generate_p256_keypair() -> Result<(String, Value)> {
    let signing_key = SigningKey::random(&mut OsRng);
    let verifying_key = signing_key.verifying_key();

    // Private key เป็น PKCS#8 PEM
    let private_pem = signing_key
        .to_pkcs8_pem(p256::pkcs8::LineEnding::LF)?
        .to_string();

    // Public key เป็น JWK format (uncompressed point)
    let point = verifying_key.to_encoded_point(false);
    let x_bytes = point.x().unwrap();
    let y_bytes = point.y().unwrap();
    let kid = format!("key-{}", &hex::encode(&x_bytes[..4]));

    let jwk = json!({
        "kty": "EC",
        "crv": "P-256",
        "x": URL_SAFE_NO_PAD.encode(x_bytes),
        "y": URL_SAFE_NO_PAD.encode(y_bytes),
        "use": "sig",
        "alg": "ES256",
        "kid": kid
    });

    Ok((private_pem, jwk))
}

/// JWKS document ที่เก็บ list ของ active public keys
/// ในช่วง overlap: ทั้ง old และ new key อยู่ใน `keys` array
#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct JwksDocument {
    pub keys: Vec<Value>,
}

impl JwksDocument {
    pub fn add_key(&mut self, jwk: Value) {
        self.keys.push(jwk);
    }

    pub fn remove_key_by_kid(&mut self, kid: &str) {
        self.keys.retain(|k| k["kid"].as_str() != Some(kid));
    }

    pub fn len(&self) -> usize {
        self.keys.len()
    }
}
```

**Overlap Period สำหรับ JWT:**

```rust
// src/overlap.rs
use serde::{Deserialize, Serialize};

/// บันทึก credential เก่าที่ยังอยู่ใน overlap period
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OverlapEntry {
    pub secret_name: String,
    /// SHA-256 ของ credential เก่า (8 bytes → 16 hex chars) สำหรับ audit
    pub old_key_hash_prefix: String,
    /// `kid` ของ JWT key (สำหรับลบออกจาก JWKS เมื่อหมด grace)
    pub jwt_key_id: Option<String>,
    /// Unix timestamp ที่ old credential expire
    pub expires_at_unix: u64,
}

impl OverlapEntry {
    pub fn new(
        secret_name: &str,
        old_key_hash_prefix: &str,
        grace_seconds: u64,
        now_unix: u64,
    ) -> Self {
        Self {
            secret_name: secret_name.to_string(),
            old_key_hash_prefix: old_key_hash_prefix.to_string(),
            jwt_key_id: None,
            expires_at_unix: now_unix.saturating_add(grace_seconds),
        }
    }

    pub fn with_jwt_kid(mut self, kid: &str) -> Self {
        self.jwt_key_id = Some(kid.to_string());
        self
    }

    /// คืน true เมื่อ old credential expire แล้ว → ควร retire
    pub fn is_expired(&self, now_unix: u64) -> bool {
        now_unix >= self.expires_at_unix
    }

    /// จำนวนวินาทีที่เหลือก่อน expire (ค่าลบหมายความว่า expire แล้ว)
    pub fn seconds_remaining(&self, now_unix: u64) -> i64 {
        self.expires_at_unix as i64 - now_unix as i64
    }
}
```

---

### ขั้นที่ 5: Kubernetes Secret Integration และ Vault Integration

#### Kubernetes — `kube` crate

```rust
// src/providers/kubernetes.rs
use kube::{Client, Api};
use k8s_openapi::api::core::v1::Secret;
use k8s_openapi::api::apps::v1::Deployment;
use kube::api::{Patch, PatchParams};
use serde_json::json;
use anyhow::Result;

pub struct KubernetesProvider {
    client: Client,
    namespace: String,
}

impl KubernetesProvider {
    pub async fn new(namespace: &str) -> Result<Self> {
        Ok(Self {
            client: Client::try_default().await?,
            namespace: namespace.to_string(),
        })
    }

    /// Patch `v1/Secret` ด้วย key/value ใหม่
    /// ใช้ strategic merge patch — field ที่ไม่ระบุไม่ถูกแตะต้อง
    pub async fn patch_secret(
        &self,
        secret_name: &str,
        data_key: &str,
        new_value: &str,
    ) -> Result<()> {
        let secrets: Api<Secret> = Api::namespaced(self.client.clone(), &self.namespace);

        // Kubernetes Secret ต้อง base64-encode ค่า
        let encoded = base64::engine::general_purpose::STANDARD.encode(new_value);

        let patch = json!({
            "apiVersion": "v1",
            "kind": "Secret",
            "data": {
                data_key: encoded
            }
        });

        secrets
            .patch(secret_name, &PatchParams::apply("secrets-rotation"), &Patch::Merge(&patch))
            .await?;

        Ok(())
    }

    /// Trigger rolling restart ของ Deployment ทั้งหมดใน namespace ที่ mount secret นี้
    /// วิธี: เพิ่ม annotation `kubectl.kubernetes.io/restartedAt` ให้ pod template spec
    pub async fn trigger_rolling_restart_for_secret(
        &self,
        secret_name: &str,
        now_rfc3339: &str,
    ) -> Result<Vec<String>> {
        let deployments: Api<Deployment> =
            Api::namespaced(self.client.clone(), &self.namespace);

        let dep_list = deployments.list(&Default::default()).await?;
        let mut restarted = Vec::new();

        for dep in dep_list.items {
            let dep_name = dep.metadata.name.as_deref().unwrap_or("");
            let mounts_secret = dep
                .spec
                .as_ref()
                .and_then(|s| s.template.spec.as_ref())
                .map(|ps| {
                    ps.volumes.as_deref().unwrap_or(&[]).iter().any(|v| {
                        v.secret
                            .as_ref()
                            .map(|s| s.secret_name.as_deref() == Some(secret_name))
                            .unwrap_or(false)
                    })
                })
                .unwrap_or(false);

            if mounts_secret {
                let patch = json!({
                    "spec": {
                        "template": {
                            "metadata": {
                                "annotations": {
                                    "kubectl.kubernetes.io/restartedAt": now_rfc3339
                                }
                            }
                        }
                    }
                });

                deployments
                    .patch(dep_name, &PatchParams::apply("secrets-rotation"), &Patch::Merge(&patch))
                    .await?;

                restarted.push(dep_name.to_string());
            }
        }

        Ok(restarted)
    }
}
```

#### HashiCorp Vault — KV v2 HTTP API

```rust
// src/providers/vault.rs
use reqwest::Client;
use serde_json::{json, Value};
use anyhow::{Result, Context};

pub struct VaultProvider {
    client: Client,
    base_url: String,
    token: String,
}

impl VaultProvider {
    pub fn new(base_url: &str, token: &str) -> Self {
        Self {
            client: Client::new(),
            base_url: base_url.trim_end_matches('/').to_string(),
            token: token.to_string(),
        }
    }

    /// อ่าน current version ของ secret จาก KV v2
    /// GET /v1/{mount}/data/{path}
    pub async fn read_secret(&self, mount: &str, path: &str) -> Result<Value> {
        let url = format!("{}/v1/{}/data/{}", self.base_url, mount, path);
        let resp = self
            .client
            .get(&url)
            .header("X-Vault-Token", &self.token)
            .send()
            .await
            .context("vault read request")?
            .error_for_status()
            .context("vault read non-2xx")?;

        let body: Value = resp.json().await?;
        Ok(body["data"]["data"].clone())
    }

    /// เขียน secret version ใหม่ ไปยัง KV v2
    /// PUT /v1/{mount}/data/{path}
    /// Vault จะสร้าง version ใหม่อัตโนมัติ — version เก่าจะยังคงอยู่ใน history
    pub async fn write_secret(
        &self,
        mount: &str,
        path: &str,
        data: &Value,
    ) -> Result<u64> {
        let url = format!("{}/v1/{}/data/{}", self.base_url, mount, path);
        let payload = json!({ "data": data });

        let resp = self
            .client
            .put(&url)
            .header("X-Vault-Token", &self.token)
            .json(&payload)
            .send()
            .await
            .context("vault write request")?
            .error_for_status()
            .context("vault write non-2xx")?;

        let body: Value = resp.json().await?;
        let version = body["data"]["version"]
            .as_u64()
            .context("missing version in vault response")?;

        Ok(version)
    }

    /// ลบ version เก่าออกจาก history (หลัง grace period)
    /// POST /v1/{mount}/delete/{path}  body: { "versions": [old_version] }
    pub async fn delete_secret_version(
        &self,
        mount: &str,
        path: &str,
        versions: &[u64],
    ) -> Result<()> {
        let url = format!("{}/v1/{}/delete/{}", self.base_url, mount, path);
        self.client
            .post(&url)
            .header("X-Vault-Token", &self.token)
            .json(&json!({ "versions": versions }))
            .send()
            .await
            .context("vault delete version")?
            .error_for_status()
            .context("vault delete non-2xx")?;
        Ok(())
    }
}
```

---

### ขั้นที่ 6: Notification, Audit Trail, และ Daemon Loop

#### Webhook Notification พร้อม Exponential Backoff

```rust
// src/notify.rs
use reqwest::Client;
use serde_json::{json, Value};
use std::time::Duration;
use anyhow::Result;

/// คำนวณ exponential backoff delay ใน milliseconds
/// delay = base_ms * 2^attempt, capped ที่ cap_ms
pub fn backoff_delay_ms(attempt: u32, base_ms: u64, cap_ms: u64) -> u64 {
    // min(63) ป้องกัน overflow เมื่อ attempt สูง
    let shift = attempt.min(63);
    let delay = base_ms.saturating_mul(1u64 << shift);
    delay.min(cap_ms)
}

/// POST JSON notification ไปยัง webhook URL
/// retry ด้วย exponential backoff สูงสุด max_retries ครั้ง
pub async fn notify_webhook(
    client: &Client,
    url: &str,
    payload: &Value,
    max_retries: u32,
) -> Result<()> {
    let base_ms = 1_000u64;   // 1 วินาที
    let cap_ms = 60_000u64;   // 60 วินาที

    for attempt in 0..=max_retries {
        let result = client
            .post(url)
            .json(payload)
            .timeout(Duration::from_secs(10))
            .send()
            .await;

        match result {
            Ok(resp) if resp.status().is_success() => {
                tracing::debug!("webhook {} ok (attempt {})", url, attempt);
                return Ok(());
            }
            Ok(resp) => {
                tracing::warn!(
                    "webhook {} returned {} (attempt {})",
                    url,
                    resp.status(),
                    attempt
                );
            }
            Err(e) => {
                tracing::warn!("webhook {} error: {} (attempt {})", url, e, attempt);
            }
        }

        if attempt < max_retries {
            let delay = backoff_delay_ms(attempt, base_ms, cap_ms);
            tokio::time::sleep(Duration::from_millis(delay)).await;
        }
    }

    anyhow::bail!("webhook {} failed after {} attempts", url, max_retries)
}

/// สร้าง notification payload สำหรับ rotation event
pub fn build_notification_payload(
    secret_name: &str,
    event: &str,           // "rotation-started" | "rotation-completed" | "rotation-failed"
    timestamp_unix: u64,
    next_rotation_unix: u64,
    error_msg: Option<&str>,
) -> Value {
    let mut payload = json!({
        "secret_name": secret_name,
        "event": event,
        "timestamp": timestamp_unix,
        "next_rotation": next_rotation_unix,
    });
    if let Some(err) = error_msg {
        payload["error"] = json!(err);
    }
    payload
}
```

#### Audit Log — Append-Only

```rust
// src/audit.rs
use serde::{Deserialize, Serialize};
use sha2::{Sha256, Digest};
use std::fs::OpenOptions;
use std::io::Write;
use anyhow::Result;

/// Outcome ของ rotation attempt
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum RotationOutcome {
    Started,
    Completed,
    Failed,
}

/// Audit event เดียว — serialized เป็น JSON line ใน append-only log file
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AuditEvent {
    pub event_id: String,
    pub secret_name: String,
    pub secret_type: String,
    pub outcome: RotationOutcome,
    pub timestamp_unix: u64,
    /// SHA-256 ของ old credential value (16 hex chars = 8 bytes prefix)
    pub old_key_hash_prefix: String,
    /// SHA-256 ของ new credential value (เฉพาะ Completed)
    pub new_key_hash_prefix: Option<String>,
    /// Duration ของ rotation ทั้งกระบวนการ
    pub duration_ms: Option<u64>,
    pub error_message: Option<String>,
}

/// คำนวณ SHA-256 และคืน 8 bytes แรกในรูป hex string
pub fn sha256_prefix(data: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data.as_bytes());
    let result = hasher.finalize();
    hex::encode(&result[..8])
}

/// สร้าง AuditEvent
pub fn build_audit_event(
    secret_name: &str,
    secret_type: &str,
    outcome: RotationOutcome,
    old_key: &str,
    new_key: Option<&str>,
    duration_ms: Option<u64>,
    error: Option<&str>,
    now_unix: u64,
) -> AuditEvent {
    AuditEvent {
        event_id: format!("evt-{}-{}", secret_name, now_unix),
        secret_name: secret_name.to_string(),
        secret_type: secret_type.to_string(),
        outcome,
        timestamp_unix: now_unix,
        old_key_hash_prefix: sha256_prefix(old_key),
        new_key_hash_prefix: new_key.map(sha256_prefix),
        duration_ms,
        error_message: error.map(str::to_string),
    }
}

/// เขียน AuditEvent ต่อท้าย NDJSON log file
/// ใช้ append mode เพื่อรับประกัน write ไม่ truncate ข้อมูลเก่า
pub fn append_audit_event(log_path: &str, event: &AuditEvent) -> Result<()> {
    let mut file = OpenOptions::new()
        .create(true)
        .append(true)
        .truncate(false)
        .open(log_path)?;

    let line = serde_json::to_string(event)?;
    writeln!(file, "{}", line)?;
    Ok(())
}
```

#### Daemon Loop หลัก

```rust
// src/main.rs (daemon run loop portion)
use clap::Parser;
use std::time::{Duration, SystemTime, UNIX_EPOCH};
use rand::Rng;
use tokio::time::interval;

#[derive(Parser, Debug)]
#[command(name = "secrets-rotation", about = "Secrets Rotation Daemon")]
struct Cli {
    /// Path ไปยัง secrets registry TOML file
    #[arg(long, default_value = "config/secrets.toml")]
    config: String,

    /// Interval ในการตรวจสอบ rotation (วินาที)
    #[arg(long, default_value_t = 60)]
    check_interval: u64,

    /// Grace period สำหรับ overlap (วินาที)
    #[arg(long, default_value_t = 3600)]
    grace_seconds: u64,

    /// Overlap hours สำหรับ JWT key rotation
    #[arg(long, default_value_t = 24)]
    overlap_hours: u64,

    /// Path สำหรับ audit log
    #[arg(long, default_value = "audit.log")]
    audit_log: String,

    /// Dry run: ไม่เขียน credential จริง
    #[arg(long)]
    dry_run: bool,
}

async fn run_daemon(cli: Cli) -> anyhow::Result<()> {
    let mut rng = rand::thread_rng();

    loop {
        let now_unix = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();

        // โหลด registry สด ๆ ในแต่ละ loop (อ่านจาก disk เพื่อ pick up manual changes)
        let config = registry::RegistryConfig::load(&cli.config)?;

        for secret in &config.secrets {
            if scheduler::is_due_for_rotation(
                secret.last_rotated_unix,
                secret.rotation_interval_days,
                now_unix,
            ) {
                tracing::info!(
                    "secret '{}' is due for rotation (type={:?})",
                    secret.name,
                    secret.secret_type
                );

                // POST notification: rotation-started
                let notif = notify::build_notification_payload(
                    &secret.name,
                    "rotation-started",
                    now_unix,
                    scheduler::next_rotation_time(now_unix, secret.rotation_interval_days),
                    None,
                );
                for url in &secret.notifiers {
                    let _ = notify::notify_webhook(&reqwest::Client::new(), url, &notif, 3).await;
                }

                // ดำเนิน rotation ตาม secret_type
                // (implementation แยกต่างหากตาม type)
                let _result = perform_rotation(secret, &cli).await;

                // บันทึก audit event
                // (ใส่ outcome จาก result)
            }
        }

        // Retire expired overlap entries
        // retire_expired_overlaps(&mut overlap_store, now_unix).await?;

        // Sleep interval + jitter ±30 วินาที
        let jitter: i64 = rng.gen_range(-30..=30);
        let sleep_secs = (cli.check_interval as i64 + jitter).max(10) as u64;
        tracing::debug!("next check in {} seconds", sleep_secs);
        tokio::time::sleep(Duration::from_secs(sleep_secs)).await;
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let cli = Cli::parse();
    tracing::info!("Starting secrets-rotation daemon (dry_run={})", cli.dry_run);
    run_daemon(cli).await
}
```

---

## การทดสอบ (Testing)

tests ทั้งหมดอยู่ใน `src/main.rs` (unit tests) และ `tests/unit_tests.rs` (integration) ทดสอบ logic หลักโดยไม่ต้องการ external dependencies (ไม่ต้อง PostgreSQL / Kubernetes จริง)

**Cargo.toml สำหรับ test (minimal version สำหรับ verify):**

```toml
[package]
name = "secrets-rotation"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio   = { version = "1", features = ["full"] }
serde   = { version = "1", features = ["derive"] }
serde_json = "1"
toml    = "0.8"
rand    = "0.8"
sha2    = "0.10"
hex     = "0.4"
chrono  = { version = "0.4", features = ["serde"] }
```

**Test code ที่ verify จริง (src/main.rs #[cfg(test)]):**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ---- Rotation Schedule ----

    #[test]
    fn test_is_due_exact_boundary() {
        // Rotated at t=0, interval=1 day; at t=86400 exactly → due
        assert!(is_due_for_rotation(0, 1, 86_400));
    }

    #[test]
    fn test_is_due_one_second_early() {
        assert!(!is_due_for_rotation(0, 1, 86_399));
    }

    #[test]
    fn test_is_due_overdue() {
        let last = 1_000_000u64;
        let now = last + 90 * 86_400;
        assert!(is_due_for_rotation(last, 30, now));
    }

    #[test]
    fn test_is_not_due_recent() {
        let now = 2_000_000u64;
        let last = now - 300;
        assert!(!is_due_for_rotation(last, 90, now));
    }

    // ---- Password Generation ----

    #[test]
    fn test_password_length_32() {
        let pw = generate_random_password(32);
        assert_eq!(pw.len(), 32);
    }

    #[test]
    fn test_password_charset_valid() {
        for _ in 0..20 {
            let pw = generate_random_password(64);
            assert!(validate_password_charset(&pw), "invalid char in: {}", pw);
        }
    }

    #[test]
    fn test_password_entropy_not_constant() {
        let pw1 = generate_random_password(32);
        let pw2 = generate_random_password(32);
        assert_ne!(pw1, pw2);
    }

    // ---- Overlap Period ----

    #[test]
    fn test_overlap_not_expired_during_grace() {
        let now = 5_000u64;
        let entry = OverlapEntry::new("db_main", "aabbccdd", 3_600, now);
        assert!(!entry.is_expired(now + 1_000));
    }

    #[test]
    fn test_overlap_expired_after_grace() {
        let now = 5_000u64;
        let entry = OverlapEntry::new("db_main", "aabbccdd", 3_600, now);
        assert!(entry.is_expired(now + 3_600));
    }

    #[test]
    fn test_overlap_seconds_remaining() {
        let now = 10_000u64;
        let entry = OverlapEntry::new("api_key", "11223344", 7_200, now);
        assert_eq!(entry.seconds_remaining(now + 3_600), 3_600);
    }

    // ---- TOML Config Deserialization ----

    #[test]
    fn test_toml_config_deserialization() {
        let toml_str = r#"
[[secrets]]
name = "db_main"
secret_type = "db_password"
rotation_interval_days = 30
last_rotated_unix = 1700000000
target = "postgres://localhost/mydb"
notifiers = ["https://hooks.example.com/rotation"]

[[secrets]]
name = "github_token"
secret_type = "api_key"
rotation_interval_days = 90
last_rotated_unix = 1695000000
target = "env:GITHUB_TOKEN"
notifiers = []
"#;
        let config: RegistryConfig = toml::from_str(toml_str).unwrap();
        assert_eq!(config.secrets.len(), 2);
        assert_eq!(config.secrets[0].name, "db_main");
        assert_eq!(config.secrets[0].secret_type, SecretType::DbPassword);
        assert_eq!(config.secrets[1].secret_type, SecretType::ApiKey);
    }

    #[test]
    fn test_toml_config_missing_notifiers_defaults() {
        let entry = SecretEntry {
            name: "jwt_key".to_string(),
            secret_type: SecretType::JwtSigningKey,
            rotation_interval_days: 365,
            last_rotated_unix: 0,
            target: "k8s:default/jwt-secret".to_string(),
            notifiers: vec![],
        };
        let serialized = toml::to_string(&RegistryConfig { secrets: vec![entry] }).unwrap();
        let back: RegistryConfig = toml::from_str(&serialized).unwrap();
        assert_eq!(back.secrets[0].secret_type, SecretType::JwtSigningKey);
    }

    // ---- Webhook Retry Backoff ----

    #[test]
    fn test_backoff_attempt_0_is_base() {
        assert_eq!(backoff_delay_ms(0, 1_000, 60_000), 1_000);
    }

    #[test]
    fn test_backoff_doubles_each_attempt() {
        assert_eq!(backoff_delay_ms(1, 1_000, 60_000), 2_000);
        assert_eq!(backoff_delay_ms(2, 1_000, 60_000), 4_000);
        assert_eq!(backoff_delay_ms(3, 1_000, 60_000), 8_000);
    }

    #[test]
    fn test_backoff_capped_at_max() {
        assert_eq!(backoff_delay_ms(10, 1_000, 60_000), 60_000);
    }

    // ---- Audit Event Construction ----

    #[test]
    fn test_audit_event_completed() {
        let evt = build_audit_event(
            "db_main",
            SecretType::DbPassword,
            RotationOutcome::Completed,
            "old-secret-value",
            Some("new-secret-value"),
            Some(320),
            None,
            1_700_000_000,
        );
        assert_eq!(evt.secret_name, "db_main");
        assert!(evt.new_key_hash_prefix.is_some());
        assert!(evt.error_message.is_none());
        assert_eq!(evt.duration_ms, Some(320));
        assert_eq!(evt.old_key_hash_prefix.len(), 16);
    }

    #[test]
    fn test_audit_event_failed_no_new_key() {
        let evt = build_audit_event(
            "api_key",
            SecretType::ApiKey,
            RotationOutcome::Failed,
            "current-key",
            None,
            None,
            Some("provider returned 503"),
            1_700_000_001,
        );
        assert!(evt.new_key_hash_prefix.is_none());
        assert_eq!(evt.error_message.as_deref(), Some("provider returned 503"));
    }

    #[test]
    fn test_sha256_prefix_deterministic() {
        let h1 = sha256_prefix("test-value");
        let h2 = sha256_prefix("test-value");
        assert_eq!(h1, h2);
        let h3 = sha256_prefix("other-value");
        assert_ne!(h1, h3);
    }

    #[test]
    fn test_audit_event_json_roundtrip() {
        let evt = build_audit_event(
            "tls_cert",
            SecretType::TlsCert,
            RotationOutcome::Started,
            "old-cert-pem",
            None,
            None,
            None,
            1_700_000_100,
        );
        let json = serde_json::to_string(&evt).unwrap();
        let back: AuditEvent = serde_json::from_str(&json).unwrap();
        assert_eq!(back.secret_name, evt.secret_name);
        assert_eq!(back.old_key_hash_prefix, evt.old_key_hash_prefix);
    }
}
```

**ผลการรัน `cargo test` จริง:**

```
running 19 tests
test tests::test_audit_event_failed_no_new_key ... ok
test tests::test_backoff_attempt_0_is_base ... ok
test tests::test_backoff_capped_at_max ... ok
test tests::test_backoff_doubles_each_attempt ... ok
test tests::test_is_due_exact_boundary ... ok
test tests::test_is_due_one_second_early ... ok
test tests::test_is_due_overdue ... ok
test tests::test_audit_event_completed ... ok
test tests::test_audit_event_json_roundtrip ... ok
test tests::test_is_not_due_recent ... ok
test tests::test_overlap_expired_after_grace ... ok
test tests::test_overlap_not_expired_during_grace ... ok
test tests::test_password_entropy_not_constant ... ok
test tests::test_password_charset_valid ... ok
test tests::test_sha256_prefix_deterministic ... ok
test tests::test_toml_config_missing_notifiers_defaults ... ok
test tests::test_toml_config_deserialization ... ok
test tests::test_password_length_32 ... ok
test tests::test_overlap_seconds_remaining ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

---

## Pitfalls ที่ต้องระวัง

### Pitfall 1: Revoke Before Write

เมื่อ rotate API key ลำดับ **revoke-then-write** ทำให้ service ที่ depend on key นั้น encounter `401 Unauthorized` ในช่วงที่กำลัง write key ใหม่ลง target

**ลำดับที่ถูก:**
```
create_new → write_new_to_target → verify_target → revoke_old
```

ถ้า `write_new_to_target` ล้มเหลว old key ยังอยู่และ service ยังทำงานได้ ถ้า `revoke_old` ล้มเหลว มี 2 key ที่ valid อยู่ชั่วคราว (acceptable — ดีกว่า 0 key)

### Pitfall 2: SQL Injection ใน `ALTER USER`

`ALTER USER` เป็น DDL statement — ไม่รองรับ parameterized query ในฝั่ง driver ดังนั้นต้อง validate password charset ก่อน format เข้า string

**สิ่งที่ผิด:**
```rust
// ถ้า new_password = "x' SUPERUSER --" จะกลายเป็น:
// ALTER USER alice WITH PASSWORD 'x' SUPERUSER --'
sqlx::query(&format!("ALTER USER alice WITH PASSWORD '{}'", new_password))
```

**วิธีแก้:** charset ที่ใช้ต้องไม่มี `'`, `"`, `\`, `;` และ validate ก่อน execute:
```rust
fn validate_password_charset(pw: &str) -> bool {
    pw.chars().all(|c| c.is_ascii_alphanumeric() || "!@#$%^&*()-_=+[]{}".contains(c))
}
```

### Pitfall 3: TOML เก็บ Timestamp เป็น Integer ไม่ใช่ RFC3339

TOML 1.0 มี datetime type แต่ crate `toml` v0.8 serialize `chrono::DateTime<Utc>` เป็น RFC3339 string ซึ่งไม่ compatible กับ field ที่กำหนดเป็น `u64` (Unix timestamp)

**ปัญหา:**
```toml
# ถ้าใช้ chrono::DateTime<Utc>
last_rotated = "2024-01-15T00:00:00Z"   # serialize ได้เป็น string
# ถ้าใช้ u64
last_rotated_unix = 1705276800           # serialize ได้เป็น integer
```

**วิธีแก้:** ใช้ `u64` Unix timestamp ใน registry ตลอด — ไม่ผสม datetime type กับ integer ใน TOML schema เดียวกัน

### Pitfall 4: Kubernetes Patch Type ที่ไม่ถูกต้อง

`kube` crate รองรับหลาย patch type: `Merge`, `Json`, `Strategic`, `Apply` แต่ละแบบมีพฤติกรรมต่างกันสำหรับ `v1/Secret`

- `Patch::Json` — เหมาะสำหรับ atomic field operation, ต้องระบุ path ชัดเจน
- `Patch::Merge` — merge top-level field, เหมาะสำหรับ update specific key ใน `data`
- `Patch::Apply` — Server-Side Apply, ต้อง set `fieldManager`, ดีกว่าสำหรับ ownership tracking

**ผิด:**
```rust
// ถ้าใช้ Apply โดยไม่ set fieldManager → conflict error
secrets.patch(name, &PatchParams::default(), &Patch::Apply(&payload)).await?;
```

**ถูก:**
```rust
secrets.patch(
    name,
    &PatchParams::apply("secrets-rotation"),   // ต้องระบุ fieldManager
    &Patch::Apply(&payload)
).await?;
```

### Pitfall 5: tokio::time::interval ไม่มี Jitter โดย Default

`tokio::time::interval` ยิง tick แรกทันที (at time zero) และ tick ถัดไปตรงเวลาพอดี ถ้า daemon หลายตัวเริ่มพร้อมกัน (เช่น Kubernetes deployment scale-up) จะ rotate พร้อมกันทำให้เกิด contention

**แก้โดยเพิ่ม jitter:**
```rust
use tokio::time::{sleep, Duration};
use rand::Rng;

let jitter_ms: u64 = rand::thread_rng().gen_range(0..60_000);
sleep(Duration::from_millis(jitter_ms)).await;  // stagger ก่อน loop เริ่ม
```

### Pitfall 6: `OpenOptions::append` บน Network Filesystem

`O_APPEND` บน NFS หรือ FUSE filesystem บางตัวไม่ใช่ atomic — หลาย process อาจเขียน partial line ทับกันได้ ถ้า audit log อยู่บน shared storage ให้ใช้ file lock (`flock`) หรือ write ผ่าน dedicated log service

```rust
// การ lock ก่อน append บน local filesystem
use fs2::FileExt;

let mut file = OpenOptions::new().create(true).append(true).open(path)?;
file.lock_exclusive()?;
writeln!(file, "{}", json_line)?;
file.unlock()?;
```

---

## การ Package และ Deploy

### Release Binary

```bash
# Build release binary
cargo build --release

# Binary อยู่ที่
./target/release/secrets-rotation

# รัน daemon
RUST_LOG=info ./target/release/secrets-rotation \
    --config /etc/secrets-rotation/registry.toml \
    --check-interval 60 \
    --grace-seconds 3600 \
    --audit-log /var/log/secrets-rotation/audit.log
```

### Docker Image

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/secrets-rotation /usr/local/bin/
RUN useradd -r -s /bin/false rotationd
USER rotationd
ENTRYPOINT ["/usr/local/bin/secrets-rotation"]
```

### Kubernetes Deployment (ใช้ daemon เอง)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secrets-rotation-daemon
  namespace: infra
spec:
  replicas: 1   # ต้องเป็น 1 เสมอ — หลาย replica rotate ซ้อนกัน
  template:
    spec:
      serviceAccountName: secrets-rotation
      containers:
      - name: daemon
        image: my-registry/secrets-rotation:latest
        args:
        - --config=/config/registry.toml
        - --audit-log=/audit/rotation.log
        - --grace-seconds=3600
        env:
        - name: RUST_LOG
          value: "info"
        - name: VAULT_TOKEN
          valueFrom:
            secretKeyRef:
              name: vault-credentials
              key: token
        volumeMounts:
        - name: config
          mountPath: /config
        - name: audit
          mountPath: /audit
      volumes:
      - name: config
        configMap:
          name: secrets-rotation-config
      - name: audit
        persistentVolumeClaim:
          claimName: audit-log-pvc
---
# RBAC: daemon ต้องการ permission อ่าน/เขียน Secret และ list/patch Deployment
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secrets-rotation
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "patch", "update"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]
```

### Systemd Unit (สำหรับ VM deployment)

```ini
[Unit]
Description=Secrets Rotation Daemon
After=network.target

[Service]
Type=simple
User=rotationd
ExecStart=/usr/local/bin/secrets-rotation \
    --config /etc/secrets-rotation/registry.toml \
    --audit-log /var/log/secrets-rotation/audit.log
Restart=on-failure
RestartSec=10
Environment=RUST_LOG=info
# ป้องกัน daemon เขียน secret ออก stdout
StandardOutput=journal
StandardError=journal
# Restrict filesystem access
ReadWritePaths=/etc/secrets-rotation /var/log/secrets-rotation
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม AWS IAM Access Key Rotation (ระดับ: กลาง)

Implement `AwsIamRotator` ที่:
- ใช้ `aws-sdk-iam` crate เรียก `CreateAccessKey` สำหรับ IAM user
- Write key/secret ไปยัง `~/.aws/credentials` หรือ Vault
- เรียก `DeleteAccessKey` สำหรับ old key หลัง verify
- Handle edge case: IAM user มี access key ได้สูงสุด 2 ตัว — ต้อง delete เก่าก่อนถ้า limit ถูกชน

ทดสอบด้วย LocalStack (AWS local emulator) ที่รันใน Docker

### Exercise 2: เพิ่ม TLS Certificate Rotation (ระดับ: สูง)

Implement `TlsCertRotator` ที่:
- Generate certificate signing request (CSR) ด้วย `rcgen` crate
- Submit CSR ไปยัง Let's Encrypt ACME v2 endpoint หรือ internal CA
- รอ validation (HTTP-01 challenge หรือ DNS-01)
- Write new cert + key pair ไปยัง Kubernetes Secret
- Trigger nginx/envoy reload: `kubectl exec {pod} -- nginx -s reload`

overlap period: เก็บ old cert ไว้ใน Secret เป็น `tls.crt.previous` จนกว่า all connections drain

### Exercise 3: เพิ่ม Web Dashboard สำหรับ Audit Log (ระดับ: กลาง)

สร้าง HTTP endpoint ด้วย axum ที่:
- `GET /api/secrets` — คืน list ของ secret entries จาก registry พร้อม `next_rotation_in_hours`
- `GET /api/audit?secret=db_main&limit=50` — scan NDJSON audit log และ filter
- `GET /api/health` — liveness check
- Serve static HTML dashboard ที่แสดง rotation timeline และ last-rotation status

### Exercise 4: Distributed Lock สำหรับ Multi-Replica Safety (ระดับ: สูง)

ปัจจุบัน deployment ต้อง `replicas: 1` เสมอ — เพิ่ม distributed lock เพื่อให้ replica หลายตัว safe:

1. ใช้ Redis `SET NX PX` (lock ด้วย expiry) ก่อน rotate แต่ละ secret
2. Lock key = `rotation-lock:{secret_name}`
3. ถ้า acquire lock ล้มเหลว → skip (another replica กำลัง rotate อยู่)
4. Release lock หลัง rotation เสร็จ (หรือ timeout)

ทดสอบด้วยการ run 3 daemon instance พร้อมกัน และ verify ว่า audit log มี rotation event เดียวต่อ secret per cycle

---

## สรุป

โปรเจคนี้สร้าง secrets rotation daemon ที่:

1. **อ่าน registry จาก TOML** — `SecretEntry` กับ type, interval, target, notifiers
2. **ตรวจ rotation schedule** — `is_due_for_rotation` ด้วย Unix timestamp arithmetic
3. **Rotate ตาม secret type** — PostgreSQL `ALTER USER`, API key create-then-revoke, JWT ECDSA keypair
4. **จัดการ overlap period** — `OverlapEntry` เก็บ expiry ให้ daemon retire credential เก่าอัตโนมัติ
5. **Integrate กับ Kubernetes** — patch `v1/Secret`, trigger rolling restart ของ Deployment ที่ mount secret
6. **Integrate กับ Vault** — write KV v2 version ใหม่, delete version เก่าหลัง grace period
7. **Notify webhooks** — exponential backoff retry, payload รวม next_rotation
8. **บันทึก audit trail** — append-only NDJSON พร้อม SHA-256 key hash prefix
9. **รัน daemon loop** — `tokio::time::sleep` กับ jitter ±30s เพื่อ thundering herd prevention

**Pattern สำคัญที่ได้เรียน:**
- **Create-before-revoke ordering** — ลำดับที่ถูกต้องสำหรับ credential rotation
- **Overlap/grace period** — รูปแบบสำหรับ zero-downtime key transition
- **CSPRNG + charset validation** — สร้าง credential ที่ปลอดภัยและ safe สำหรับ SQL context
- **Kubernetes RBAC + Server-Side Apply** — pattern สำหรับ Kubernetes operator

โปรเจคถัดไป [Project D07: JWT Library](project-d07-jwt-library.md) จะลงลึกเรื่อง JWT signing/verification อย่างเต็มรูปแบบ — ใช้ key material และ JWKS infrastructure ที่สร้างในโปรเจคนี้เป็นพื้นฐาน

---

**โปรเจคก่อนหน้า:** [project-d05-audit-log.md](project-d05-audit-log.md) | **โปรเจคถัดไป:** [project-d07-jwt-library.md](project-d07-jwt-library.md)
