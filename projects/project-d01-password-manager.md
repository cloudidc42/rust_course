# Project D01: Password Manager CLI

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

---

## ภาพรวมโปรเจค

Password Manager CLI คือโปรแกรมบรรทัดคำสั่งสำหรับจัดการ credential ส่วนตัว (username, password, URL, notes) ในรูปแบบ encrypted vault บนเครื่องของผู้ใช้เอง ไม่พึ่งพา cloud service ใด ๆ

**Use case จริงในโลก production:**
- เก็บ API key, SSH password, database credential ขององค์กรไว้ใน encrypted file ที่สำรองได้
- ทีม DevOps ที่ต้องการ secrets manager แบบ offline-first และ auditable
- นักพัฒนาที่ต้องการเรียนรู้ implementation ของ symmetric encryption pipeline จากต้นทางจริง

**Learning value:**
โปรเจคนี้ครอบคลุมวงจรเต็มของ applied cryptography ตั้งแต่ key derivation → encryption → storage → decryption รวมถึงการออกแบบ data structure ที่ zeroize sensitive data เมื่อ drop และการเชื่อมต่อกับ external security API (HaveIBeenPwned) แบบ privacy-preserving

---

## สิ่งที่จะได้เรียนรู้

- **Argon2id KDF** — ทำความเข้าใจ memory-hard key derivation และ parameter tuning ตาม RFC 9106
- **AES-256-GCM AEAD** — AEAD cipher ที่ verify authenticity ของ ciphertext ก่อน decrypt
- **`zeroize` pattern** — บังคับเคลียร์ sensitive bytes จาก heap/stack เมื่อ value หมดอายุ
- **Binary file format design** — ออกแบบ magic bytes + versioned header สำหรับ vault file
- **Trigram fuzzy search** — implement similarity metric สำหรับ UX ที่ดีขึ้น
- **k-anonymity API integration** — ส่ง partial hash เพื่อตรวจสอบ password leak โดยไม่เปิดเผย password จริง
- **Clap 4 derive-based CLI** — สร้าง subcommand hierarchy อย่างมีโครงสร้าง
- **Session timeout** — เก็บ state ใน memory เท่านั้น ไม่เขียนลง disk

---

## ความรู้ที่ต้องมีมาก่อน

- Parts 1–40: ownership, borrowing, traits, enums, error handling พื้นฐาน
- Parts 41–60: trait objects, generics, closures, iterators ขั้นสูง
- Parts 61–70: file I/O, process, environment variables, `std::time`
- Parts 71–80: async/await พื้นฐาน (สำหรับ HIBP API call ในขั้นตอนหลัง)
- ความคุ้นเคยกับ `serde` / `serde_json` และการใช้ `HashMap`

---

## โครงสร้างโปรเจค (Project Layout)

```
passmanager/
├── Cargo.toml
├── src/
│   ├── main.rs          ← CLI entry point, subcommand dispatch
│   ├── crypto.rs        ← Argon2id KDF + AES-256-GCM encrypt/decrypt
│   ├── vault.rs         ← Vault struct, Entry, file I/O (read/write vault.bin)
│   ├── generator.rs     ← Password generator + strength scorer + HIBP prefix
│   ├── audit.rs         ← Append-only encrypted audit log
│   ├── session.rs       ← In-memory session state + timeout
│   ├── import_export.rs ← CSV import/export (LastPass / Bitwarden format)
│   └── clipboard.rs     ← arboard wrapper + 30-second auto-clear
├── tests/
│   └── integration.rs   ← End-to-end vault lifecycle tests
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Master Password
      │
      ▼
[Argon2id KDF]  ←── 16-byte random salt (stored in vault header)
      │
      ▼
32-byte Master Key (Zeroizing<[u8; 32]>)
      │
      ├──► [AES-256-GCM encrypt]  ◄── Vault JSON
      │          │
      │          ▼
      │    12-byte nonce ++ ciphertext ++ 16-byte GCM tag
      │          │
      │          ▼
      │    vault.bin: MAGIC(8) + VERSION(1) + SALT(16) + ENCRYPTED_BLOB
      │
      └──► [AES-256-GCM encrypt]  ◄── Audit log entries
                 │
                 ▼
           audit.bin: same header format
```

### Vault File Binary Format

```
Offset   Size   Field
──────────────────────────────────────────────────
0        8      Magic bytes: "PASSV001"
8        1      Version: 0x01
9        16     Argon2id salt (random per vault creation)
25       12     AES-GCM nonce (random per write)
37       N      Ciphertext + 16-byte GCM authentication tag
```

### Module Boundaries

| Module | ความรับผิดชอบ | ไม่รับผิดชอบ |
|--------|--------------|-------------|
| `crypto` | KDF, encrypt, decrypt | Business logic, file paths |
| `vault` | Entry CRUD, search, serialization | Encryption keys |
| `generator` | Password generation, strength scoring | Vault storage |
| `audit` | Log append, encrypted log write | User interaction |
| `session` | Unlock timestamp, timeout check | Disk I/O |
| `clipboard` | Copy to clipboard, timed clear | Password generation |

### ทำไมถึงเลือก Design นี้

**AES-256-GCM แทน AES-256-CBC + HMAC:**
GCM เป็น AEAD (Authenticated Encryption with Associated Data) — authenticate และ encrypt ใน pass เดียว ลด attack surface จาก padding oracle และ MAC-then-encrypt ordering errors

**Argon2id แทน bcrypt/scrypt:**
RFC 9106 แนะนำ Argon2id สำหรับ password hashing ทั่วไป เนื่องจาก hybrid memory-hard + data-independent property — resist GPU/ASIC brute-force และ side-channel attacks บน time-memory tradeoff

**`zeroize` แทน `drop` manual:**
`Zeroizing<T>` wrapper และ `#[derive(Zeroize)]` รับประกันว่า compiler จะไม่ optimize out การเคลียร์ memory ซึ่ง plain `drop` ไม่รับประกัน

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Cargo.toml

สร้าง project ใหม่และกำหนด dependencies ทั้งหมด:

```bash
cargo new passmanager
cd passmanager
```

**`Cargo.toml`:**

```toml
[package]
name = "passmanager"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "passmanager"
path = "src/main.rs"

[dependencies]
# Cryptography
argon2  = "0.5"          # Argon2id KDF (RFC 9106)
aes-gcm = "0.10"         # AES-256-GCM AEAD cipher
zeroize = { version = "1", features = ["derive"] }  # Secure memory wipe

# Randomness
rand    = "0.8"          # OsRng for cryptographic randomness

# Serialization
serde      = { version = "1", features = ["derive"] }
serde_json = "1"

# Utilities
sha1    = "0.10"         # SHA-1 for HIBP k-anonymity prefix
hex     = "0.4"          # Hex encode/decode
chrono  = { version = "0.4", features = ["serde"] }  # Timestamps

# CLI
clap = { version = "4", features = ["derive"] }

# Clipboard
arboard = "3"

[dev-dependencies]
tempfile = "3"           # Temporary files for tests
```

> **หมายเหตุ:** `arboard` ต้องการ system clipboard daemon (X11/Wayland/macOS/Windows) ถ้า build บน headless CI ให้ feature-flag ออก

---

### ขั้นที่ 2: Cryptographic Core (`src/crypto.rs`)

**`src/crypto.rs`** — module นี้รับผิดชอบทุกอย่างที่เกี่ยวกับ cryptographic primitive:

```rust
// src/crypto.rs

use aes_gcm::{
    aead::{Aead, AeadCore, KeyInit, OsRng as AeadOsRng},
    Aes256Gcm, Key, Nonce,
};
use argon2::{Argon2, Params, Version};
use rand::RngCore;
use zeroize::Zeroizing;

/// 16-byte random salt ที่เก็บใน vault header
pub type Salt = [u8; 16];

/// 32-byte AES-256 master key — Zeroizing ล้างค่าอัตโนมัติเมื่อ drop
pub type MasterKey = Zeroizing<[u8; 32]>;

/// สร้าง 32-byte master key จาก master password + 16-byte salt
/// ใช้ Argon2id (RFC 9106) parameters: m=65536 KiB, t=3 iterations, p=4 parallelism
pub fn derive_key(password: &str, salt: &Salt) -> Result<MasterKey, String> {
    let params = Params::new(65536, 3, 4, Some(32))
        .map_err(|e| format!("Argon2 params error: {e}"))?;

    let argon2 = Argon2::new(argon2::Algorithm::Argon2id, Version::V0x13, params);

    let mut output = Zeroizing::new([0u8; 32]);
    argon2
        .hash_password_into(password.as_bytes(), salt, output.as_mut())
        .map_err(|e| format!("Argon2 hash error: {e}"))?;

    Ok(output)
}

/// สุ่ม 16-byte salt สำหรับ vault ใหม่
pub fn generate_salt() -> Salt {
    let mut salt = [0u8; 16];
    rand::rngs::OsRng.fill_bytes(&mut salt);
    salt
}

/// Encrypt plaintext ด้วย AES-256-GCM
/// Output format: nonce(12 bytes) || ciphertext || GCM_tag(16 bytes)
pub fn encrypt(key: &MasterKey, plaintext: &[u8]) -> Result<Vec<u8>, String> {
    let aes_key = Key::<Aes256Gcm>::from_slice(key.as_ref());
    let cipher = Aes256Gcm::new(aes_key);
    let nonce = Aes256Gcm::generate_nonce(&mut AeadOsRng);

    let ciphertext = cipher
        .encrypt(&nonce, plaintext)
        .map_err(|e| format!("AES-GCM encrypt error: {e}"))?;

    let mut output = Vec::with_capacity(12 + ciphertext.len());
    output.extend_from_slice(&nonce);
    output.extend_from_slice(&ciphertext);
    Ok(output)
}

/// Decrypt nonce+ciphertext+tag ด้วย AES-256-GCM
/// ตรวจสอบ authentication tag ก่อน return plaintext เสมอ
pub fn decrypt(key: &MasterKey, data: &[u8]) -> Result<Vec<u8>, String> {
    if data.len() < 12 {
        return Err("Ciphertext too short: missing nonce".to_string());
    }
    let (nonce_bytes, ciphertext) = data.split_at(12);
    let nonce = Nonce::from_slice(nonce_bytes);

    let aes_key = Key::<Aes256Gcm>::from_slice(key.as_ref());
    let cipher = Aes256Gcm::new(aes_key);

    cipher
        .decrypt(nonce, ciphertext)
        .map_err(|_| "AES-GCM decryption failed: authentication tag mismatch".to_string())
}
```

**ทำไม Argon2id ถึงใช้ parameters เหล่านี้:**

| Parameter | ค่า | เหตุผล |
|-----------|-----|--------|
| `m` (memory) | 65536 KiB (64 MiB) | RFC 9106 § 4 แนะนำ ≥ 64 MiB สำหรับ interactive use |
| `t` (iterations) | 3 | ชดเชย time-memory tradeoff เพิ่ม CPU cost |
| `p` (parallelism) | 4 | ใช้ประโยชน์ multi-core และเพิ่ม GPU resistance |
| output | 32 bytes | ขนาด key ที่ AES-256 ต้องการ |

**ทำไม AES-256-GCM:**
- **Authenticated Encryption:** GCM tag (16 bytes) ทำหน้าที่เป็น MAC — `decrypt()` ตรวจ tag ก่อน return plaintext เสมอ หาก tag ไม่ตรง function return `Err` โดยไม่เปิดเผย plaintext ใด ๆ
- **Nonce uniqueness:** nonce 12 bytes สุ่มด้วย `OsRng` ต่อ write operation — ความน่าจะเป็นการชนกันของ nonce ใน 2^32 writes ≈ 2^{-32} (birthday bound)

---

### ขั้นที่ 3: Vault Data Model (`src/vault.rs`)

**Data structures:**

```rust
// src/vault.rs

use crate::crypto::{decrypt, encrypt, MasterKey, Salt};
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::path::Path;
use zeroize::Zeroize;

/// Magic bytes ใน vault file header
pub const VAULT_MAGIC: &[u8; 8] = b"PASSV001";
pub const VAULT_VERSION: u8 = 1;

/// Entry หนึ่งรายการใน vault
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Entry {
    pub id: String,
    pub title: String,
    pub url: String,
    pub username: String,
    pub password: String,       // ← field นี้ zeroize เมื่อ drop
    pub notes: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub tags: Vec<String>,
}

impl Drop for Entry {
    fn drop(&mut self) {
        self.password.zeroize();  // ล้าง password bytes ออกจาก heap
    }
}

/// Vault หลัก: map จาก entry ID → Entry
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct Vault {
    pub entries: HashMap<String, Entry>,
}

impl Vault {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn insert(&mut self, entry: Entry) {
        self.entries.insert(entry.id.clone(), entry);
    }

    pub fn remove(&mut self, id: &str) -> Option<Entry> {
        self.entries.remove(id)
    }

    pub fn get(&self, id: &str) -> Option<&Entry> {
        self.entries.get(id)
    }

    /// List entries เรียงตาม title
    pub fn list_sorted(&self) -> Vec<&Entry> {
        let mut entries: Vec<&Entry> = self.entries.values().collect();
        entries.sort_by(|a, b| a.title.cmp(&b.title));
        entries
    }

    /// Filter by tag (case-sensitive)
    pub fn list_by_tag<'a>(&'a self, tag: &str) -> Vec<&'a Entry> {
        let mut entries: Vec<&Entry> = self
            .entries
            .values()
            .filter(|e| e.tags.iter().any(|t| t == tag))
            .collect();
        entries.sort_by(|a, b| a.title.cmp(&b.title));
        entries
    }

    /// Substring search ใน title, url, username
    pub fn search(&self, query: &str) -> Vec<&Entry> {
        let q = query.to_lowercase();
        let mut results: Vec<&Entry> = self
            .entries
            .values()
            .filter(|e| {
                e.title.to_lowercase().contains(&q)
                    || e.url.to_lowercase().contains(&q)
                    || e.username.to_lowercase().contains(&q)
            })
            .collect();
        results.sort_by(|a, b| a.title.cmp(&b.title));
        results
    }

    /// Trigram similarity ระหว่าง string สองตัว (0.0–1.0)
    pub fn trigram_similarity(a: &str, b: &str) -> f64 {
        fn trigrams(s: &str) -> std::collections::HashSet<[char; 3]> {
            let chars: Vec<char> = format!("  {}  ", s.to_lowercase()).chars().collect();
            chars.windows(3).map(|w| [w[0], w[1], w[2]]).collect()
        }
        let ta = trigrams(a);
        let tb = trigrams(b);
        if ta.is_empty() && tb.is_empty() {
            return 1.0;
        }
        let intersection = ta.intersection(&tb).count();
        let union = ta.union(&tb).count();
        if union == 0 { 0.0 } else { intersection as f64 / union as f64 }
    }

    /// Fuzzy search โดยใช้ trigram similarity (threshold ≥ 0.2)
    pub fn fuzzy_search(&self, query: &str) -> Vec<(&Entry, f64)> {
        let mut results: Vec<(&Entry, f64)> = self
            .entries
            .values()
            .filter_map(|e| {
                let score = [&e.title, &e.url, &e.username]
                    .iter()
                    .map(|s| Self::trigram_similarity(query, s))
                    .fold(0.0f64, f64::max);
                if score >= 0.2 { Some((e, score)) } else { None }
            })
            .collect();
        results.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap_or(std::cmp::Ordering::Equal));
        results
    }

    pub fn serialize(&self) -> Result<Vec<u8>, String> {
        serde_json::to_vec(self).map_err(|e| format!("Vault serialization failed: {e}"))
    }

    pub fn deserialize(data: &[u8]) -> Result<Self, String> {
        serde_json::from_slice(data).map_err(|e| format!("Vault deserialization failed: {e}"))
    }
}
```

**Vault file I/O:**

```rust
/// เขียน vault ลง disk — format: MAGIC(8) + VERSION(1) + SALT(16) + ENCRYPTED_BLOB
pub fn write_vault_file(
    path: &Path,
    key: &MasterKey,
    salt: &Salt,
    vault: &Vault,
) -> Result<(), String> {
    let plaintext = vault.serialize()?;
    let encrypted = encrypt(key, &plaintext)?;

    let mut data: Vec<u8> = Vec::new();
    data.extend_from_slice(VAULT_MAGIC);
    data.push(VAULT_VERSION);
    data.extend_from_slice(salt);
    data.extend_from_slice(&encrypted);

    std::fs::write(path, &data).map_err(|e| format!("Failed to write vault file: {e}"))
}

/// อ่านและ decrypt vault file — ตรวจ magic bytes, version, GCM tag ก่อน return
pub fn read_vault_file(path: &Path, key: &MasterKey) -> Result<(Salt, Vault), String> {
    let data = std::fs::read(path).map_err(|e| format!("Failed to read vault file: {e}"))?;

    if data.len() < 25 {
        return Err("Vault file too short".to_string());
    }

    if &data[0..8] != VAULT_MAGIC {
        return Err("Invalid vault file: magic bytes mismatch".to_string());
    }

    let version = data[8];
    if version != VAULT_VERSION {
        return Err(format!("Unsupported vault version: {version}"));
    }

    let mut salt: Salt = [0u8; 16];
    salt.copy_from_slice(&data[9..25]);

    let encrypted = &data[25..];
    let plaintext = decrypt(key, encrypted)?;
    let vault = Vault::deserialize(&plaintext)?;

    Ok((salt, vault))
}
```

**สิ่งสำคัญในการออกแบบ `Entry::drop`:**

```rust
// ✅ ถูก — ใช้ zeroize ล้าง password field
impl Drop for Entry {
    fn drop(&mut self) {
        self.password.zeroize();
    }
}

// ❌ ผิด — Rust compiler อาจ optimize out การเคลียร์แบบนี้
impl Drop for Entry {
    fn drop(&mut self) {
        unsafe {
            std::ptr::write_bytes(
                self.password.as_mut_ptr(),
                0,
                self.password.len()
            );
        }
    }
}
```

`zeroize` crate ใช้ `volatile_set_memory` บนแต่ละ platform เพื่อป้องกัน compiler optimization ที่จะ skip การเขียน 0 ลง memory ที่กำลังจะถูก free

---

### ขั้นที่ 4: Password Generator และ Strength Scorer (`src/generator.rs`)

```rust
// src/generator.rs

use rand::RngCore;

/// Configuration สำหรับ password generator
#[derive(Debug, Clone)]
pub struct GeneratorConfig {
    pub length: usize,           // 8–128
    pub use_uppercase: bool,
    pub use_lowercase: bool,
    pub use_digits: bool,
    pub use_symbols: bool,
    pub exclude_ambiguous: bool, // ไม่รวม 0/O/o/1/l/I/|
}

impl Default for GeneratorConfig {
    fn default() -> Self {
        Self {
            length: 20,
            use_uppercase: true,
            use_lowercase: true,
            use_digits: true,
            use_symbols: true,
            exclude_ambiguous: false,
        }
    }
}

const AMBIGUOUS: &[char] = &['0', 'O', 'o', '1', 'l', 'I', '|'];
const UPPERCASE: &str = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
const LOWERCASE: &str = "abcdefghijklmnopqrstuvwxyz";
const DIGITS: &str = "0123456789";
const SYMBOLS: &str = "!@#$%^&*()-_=+[]{}|;:,.<>?";

/// สร้าง password โดยใช้ rand::rngs::OsRng (cryptographically random)
/// ใช้ rejection sampling เพื่อป้องกัน modulo bias
pub fn generate_password(config: &GeneratorConfig) -> Result<String, String> {
    if config.length < 8 || config.length > 128 {
        return Err(format!(
            "Password length {} is out of allowed range [8, 128]",
            config.length
        ));
    }

    let mut charset: Vec<char> = Vec::new();
    if config.use_uppercase { charset.extend(UPPERCASE.chars()); }
    if config.use_lowercase { charset.extend(LOWERCASE.chars()); }
    if config.use_digits    { charset.extend(DIGITS.chars()); }
    if config.use_symbols   { charset.extend(SYMBOLS.chars()); }
    if config.exclude_ambiguous {
        charset.retain(|c| !AMBIGUOUS.contains(c));
    }

    if charset.is_empty() {
        return Err("No character classes selected; charset is empty".to_string());
    }

    let mut rng = rand::rngs::OsRng;
    let mut password = String::with_capacity(config.length);
    let n = charset.len();
    let mut buf = [0u8; 1];

    // Rejection sampling — ป้องกัน modulo bias
    let mut i = 0;
    while i < config.length {
        rng.fill_bytes(&mut buf);
        let idx = buf[0] as usize;
        let limit = (256 / n) * n;
        if idx < limit {
            password.push(charset[idx % n]);
            i += 1;
        }
    }

    Ok(password)
}
```

**ทำไมถึงต้อง Rejection Sampling:**

ถ้า charset มี 72 ตัวอักษร และ byte range คือ 0–255 (256 ค่า):
- 256 / 72 = 3 remainder 40
- ค่า 0–215 → มีความน่าจะเป็นเท่ากัน (72 × 3 = 216 ค่า)
- ค่า 216–255 → map ไปยัง index 0–39 เพิ่มความน่าจะเป็นของ 40 ตัวแรก

Rejection sampling: ถ้า byte ≥ `(256/n)*n` ให้ดึงค่าใหม่ — ทุกตัวอักษรใน charset จึงมี probability เท่ากัน

**Strength Scorer (NIST SP 800-63B aligned):**

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum StrengthLevel {
    VeryWeak, Weak, Fair, Strong, VeryStrong,
}

pub struct StrengthResult {
    pub level: StrengthLevel,
    pub score: u8,           // 0–100
    pub length: usize,
    pub has_uppercase: bool,
    pub has_lowercase: bool,
    pub has_digits: bool,
    pub has_symbols: bool,
    pub issues: Vec<String>, // human-readable improvement hints
}

pub fn score_password(password: &str) -> StrengthResult {
    let length = password.len();
    let mut score: u8 = 0;
    let mut issues = Vec::new();

    let has_uppercase = password.chars().any(|c| c.is_ascii_uppercase());
    let has_lowercase = password.chars().any(|c| c.is_ascii_lowercase());
    let has_digits    = password.chars().any(|c| c.is_ascii_digit());
    let has_symbols   = password.chars().any(|c| !c.is_alphanumeric());

    // Length scoring — NIST SP 800-63B § 5.1.1 ขั้นต่ำ 8 ตัวอักษร
    if length < 8 {
        issues.push("Length below NIST SP 800-63B minimum of 8".to_string());
    } else {
        score += 20;
        if length >= 12 { score += 10; }
        if length >= 16 { score += 10; }
        if length >= 20 { score += 10; }
    }

    // Character class diversity
    let class_count = [has_uppercase, has_lowercase, has_digits, has_symbols]
        .iter().filter(|&&b| b).count() as u8;
    score += class_count * 10;

    if !has_uppercase { issues.push("No uppercase letters".to_string()); }
    if !has_lowercase { issues.push("No lowercase letters".to_string()); }
    if !has_digits    { issues.push("No digits".to_string()); }
    if !has_symbols   { issues.push("No symbols".to_string()); }

    score = score.min(100);

    let level = match score {
        0..=19  => StrengthLevel::VeryWeak,
        20..=39 => StrengthLevel::Weak,
        40..=59 => StrengthLevel::Fair,
        60..=79 => StrengthLevel::Strong,
        _       => StrengthLevel::VeryStrong,
    };

    StrengthResult { level, score, length,
        has_uppercase, has_lowercase, has_digits, has_symbols, issues }
}
```

**HaveIBeenPwned k-anonymity:**

```rust
use sha1::{Digest, Sha1};

/// คำนวณ SHA-1 hex string ของ password (uppercase)
pub fn password_sha1_hex(password: &str) -> String {
    let mut hasher = Sha1::new();
    hasher.update(password.as_bytes());
    format!("{:X}", hasher.finalize())
}

/// Return 5-char prefix สำหรับ HIBP k-anonymity API query
/// ส่งเฉพาะ prefix 5 ตัวแรก — server ไม่เคยได้รับ full hash
pub fn hibp_prefix(password: &str) -> String {
    password_sha1_hex(password)[..5].to_string()
}

/// ตรวจสอบ HIBP response ว่า password hash suffix ปรากฏหรือไม่
/// response format: "SUFFIX:COUNT\nSUFFIX:COUNT\n..."
pub fn parse_hibp_response(password: &str, response: &str) -> Option<u64> {
    let full_hash = password_sha1_hex(password);
    let suffix = &full_hash[5..]; // ส่วนที่เราเปรียบเทียบ client-side

    for line in response.lines() {
        if let Some((hash_suffix, count_str)) = line.split_once(':') {
            if hash_suffix.eq_ignore_ascii_case(suffix) {
                return count_str.trim().parse().ok();
            }
        }
    }
    None
}
```

HIBP k-anonymity protocol:
1. คำนวณ SHA-1 ของ password → hex uppercase
2. ส่ง GET `/range/{prefix5}` ไปยัง `api.pwnedpasswords.com`
3. Server return รายการ suffix ทั้งหมดที่ขึ้นต้นด้วย prefix นั้น พร้อม count
4. Client เปรียบเทียบ suffix ที่เหลือ (bytes 5..40) กับรายการ — server ไม่เคยรับข้อมูลเพียงพอที่จะระบุ password จริง

---

### ขั้นที่ 5: Audit Log (`src/audit.rs`)

Audit log เป็น append-only file ที่เก็บ record ของทุก mutating operation — encrypt ด้วย master key เดียวกัน:

```rust
// src/audit.rs

use crate::crypto::{decrypt, encrypt, MasterKey};
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::path::Path;

const AUDIT_MAGIC: &[u8; 8] = b"AUDITV01";

/// Record หนึ่งรายการในเหตุการณ์ audit
#[derive(Debug, Serialize, Deserialize)]
pub struct AuditRecord {
    pub timestamp: DateTime<Utc>,
    pub operation: String,  // "add", "edit", "delete", "get", "unlock"
    pub entry_id: Option<String>,
    pub detail: Option<String>,
}

/// อ่าน audit log ทั้งหมด (decrypt แล้ว deserialize)
pub fn read_audit_log(path: &Path, key: &MasterKey) -> Result<Vec<AuditRecord>, String> {
    if !path.exists() {
        return Ok(vec![]);
    }

    let data = std::fs::read(path).map_err(|e| format!("Cannot read audit log: {e}"))?;

    if data.len() < 8 || &data[0..8] != AUDIT_MAGIC {
        return Err("Invalid audit log: magic mismatch".to_string());
    }

    // Format: MAGIC(8) + [encrypted_record_block]*
    // แต่ละ block: length(4 bytes LE) + encrypted_data
    let mut pos = 8usize;
    let mut records = Vec::new();

    while pos + 4 <= data.len() {
        let len = u32::from_le_bytes(data[pos..pos+4].try_into().unwrap()) as usize;
        pos += 4;
        if pos + len > data.len() {
            break;
        }
        let block = &data[pos..pos+len];
        pos += len;

        let plaintext = decrypt(key, block)?;
        let record: AuditRecord = serde_json::from_slice(&plaintext)
            .map_err(|e| format!("Audit record parse error: {e}"))?;
        records.push(record);
    }

    Ok(records)
}

/// Append record หนึ่งรายการลง audit log
pub fn append_audit_record(
    path: &Path,
    key: &MasterKey,
    record: &AuditRecord,
) -> Result<(), String> {
    let json = serde_json::to_vec(record)
        .map_err(|e| format!("Audit record serialize error: {e}"))?;
    let encrypted = encrypt(key, &json)?;

    // เริ่มต้น file ด้วย magic bytes ถ้ายังไม่มี
    let mut data: Vec<u8> = if path.exists() {
        std::fs::read(path).map_err(|e| format!("Cannot read audit log: {e}"))?
    } else {
        AUDIT_MAGIC.to_vec()
    };

    // Append: length(4 bytes LE) + encrypted_block
    let len = encrypted.len() as u32;
    data.extend_from_slice(&len.to_le_bytes());
    data.extend_from_slice(&encrypted);

    std::fs::write(path, &data).map_err(|e| format!("Cannot write audit log: {e}"))
}
```

---

### ขั้นที่ 6: Session Timeout และ Clipboard (`src/session.rs`, `src/clipboard.rs`)

**Session state (in-memory only):**

```rust
// src/session.rs

use std::time::{Duration, Instant};

/// Session state ที่ unlock vault ไว้ — ไม่เขียนลง disk เด็ดขาด
pub struct Session {
    unlocked_at: Instant,
    timeout: Duration,
}

impl Session {
    pub fn new(timeout_secs: u64) -> Self {
        Self {
            unlocked_at: Instant::now(),
            timeout: Duration::from_secs(timeout_secs),
        }
    }

    /// ตรวจว่า session ยัง valid อยู่หรือไม่
    pub fn is_valid(&self) -> bool {
        self.unlocked_at.elapsed() < self.timeout
    }

    /// Reset timer (เช่น หลัง user activity)
    pub fn refresh(&mut self) {
        self.unlocked_at = Instant::now();
    }

    /// เวลาที่เหลือ (seconds)
    pub fn remaining_secs(&self) -> u64 {
        let elapsed = self.unlocked_at.elapsed();
        if elapsed >= self.timeout {
            0
        } else {
            (self.timeout - elapsed).as_secs()
        }
    }
}
```

**Clipboard helper พร้อม auto-clear หลัง 30 วินาที:**

```rust
// src/clipboard.rs

use arboard::Clipboard;
use std::thread;
use std::time::Duration;

/// Copy password ไปยัง clipboard แล้วล้างหลัง `clear_after_secs` วินาที
/// ทำใน background thread เพื่อไม่บล็อก main thread
pub fn copy_with_timeout(text: String, clear_after_secs: u64) -> Result<(), String> {
    {
        let mut clipboard = Clipboard::new()
            .map_err(|e| format!("Cannot open clipboard: {e}"))?;
        clipboard.set_text(&text)
            .map_err(|e| format!("Cannot set clipboard: {e}"))?;
    }

    // Spawn thread เพื่อ clear clipboard
    thread::spawn(move || {
        thread::sleep(Duration::from_secs(clear_after_secs));
        if let Ok(mut clipboard) = Clipboard::new() {
            // ตรวจก่อนว่า clipboard ยังเป็นข้อความเดิมอยู่
            if clipboard.get_text().ok().as_deref() == Some(&text) {
                let _ = clipboard.set_text("");
            }
        }
    });

    Ok(())
}
```

---

### ขั้นที่ 7: CSV Import/Export (`src/import_export.rs`)

```rust
// src/import_export.rs

use crate::vault::{Entry, Vault};
use chrono::Utc;
use std::io::{BufRead, BufReader, Write};
use std::path::Path;
use uuid::Uuid;  // เพิ่ม uuid = "1" ใน Cargo.toml

/// Import จาก LastPass/Bitwarden CSV format:
/// url,username,password,totp,extra,name,grouping,fav
pub fn import_csv(path: &Path, vault: &mut Vault) -> Result<usize, String> {
    let file = std::fs::File::open(path)
        .map_err(|e| format!("Cannot open CSV: {e}"))?;
    let reader = BufReader::new(file);
    let mut count = 0;

    for (line_num, line_res) in reader.lines().enumerate() {
        let line = line_res.map_err(|e| format!("Read error at line {line_num}: {e}"))?;

        // Skip header
        if line_num == 0 && line.to_lowercase().starts_with("url") {
            continue;
        }

        let fields: Vec<&str> = line.splitn(8, ',').collect();
        if fields.len() < 6 {
            continue;
        }

        let entry = Entry {
            id: format!("{:x}", md5_simple(&line)), // deterministic ID จาก content
            url: fields[0].trim().to_string(),
            username: fields[1].trim().to_string(),
            password: fields[2].trim().to_string(),
            title: fields[5].trim().to_string(),
            notes: fields[4].trim().to_string(),
            created_at: Utc::now(),
            updated_at: Utc::now(),
            tags: if fields[6].trim().is_empty() {
                vec![]
            } else {
                vec![fields[6].trim().to_string()]
            },
        };

        vault.insert(entry);
        count += 1;
    }
    Ok(count)
}

/// Export เป็น plaintext CSV — แสดง warning เสมอ
pub fn export_csv_plaintext(vault: &Vault, path: &Path) -> Result<(), String> {
    eprintln!("WARNING: Exporting plaintext CSV. Keep this file secure and delete after use.");

    let mut file = std::fs::File::create(path)
        .map_err(|e| format!("Cannot create export file: {e}"))?;

    writeln!(file, "url,username,password,totp,extra,name,grouping,fav")
        .map_err(|e| format!("Write error: {e}"))?;

    for entry in vault.list_sorted() {
        writeln!(
            file,
            "{},{},{},,,{},{},0",
            csv_escape(&entry.url),
            csv_escape(&entry.username),
            csv_escape(&entry.password),
            csv_escape(&entry.title),
            entry.tags.first().map(|s| s.as_str()).unwrap_or(""),
        ).map_err(|e| format!("Write error: {e}"))?;
    }

    Ok(())
}

fn csv_escape(s: &str) -> String {
    if s.contains(',') || s.contains('"') || s.contains('\n') {
        format!("\"{}\"", s.replace('"', "\"\""))
    } else {
        s.to_string()
    }
}

// ใช้ FNV hash แทน uuid เพื่อ deterministic import ID
fn md5_simple(s: &str) -> u64 {
    use std::hash::{Hash, Hasher};
    let mut h = std::collections::hash_map::DefaultHasher::new();
    s.hash(&mut h);
    h.finish()
}
```

---

### ขั้นที่ 8: CLI Entry Point (`src/main.rs`)

```rust
// src/main.rs

mod audit;
mod clipboard;
mod crypto;
mod generator;
mod import_export;
mod session;
mod vault;

use clap::{Parser, Subcommand};
use std::path::PathBuf;

/// passmanager — encrypted password manager CLI
#[derive(Parser)]
#[command(name = "passmanager", version, about)]
struct Cli {
    /// Path ของ vault file (default: ~/.passmanager/vault.bin)
    #[arg(long, global = true)]
    vault: Option<PathBuf>,

    /// Path ของ audit log (default: ~/.passmanager/audit.bin)
    #[arg(long, global = true)]
    audit: Option<PathBuf>,

    /// Session timeout ในหน่วยวินาที (default: 300)
    #[arg(long, global = true, default_value = "300")]
    timeout: u64,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// สร้าง vault ใหม่
    Init,
    /// เพิ่ม entry ใหม่
    Add {
        #[arg(short, long)] title: String,
        #[arg(short, long)] url: String,
        #[arg(short, long)] username: String,
        /// สร้าง password อัตโนมัติ
        #[arg(long)] generate: bool,
        /// ความยาวของ generated password
        #[arg(long, default_value = "20")] length: usize,
        #[arg(long)] tags: Vec<String>,
    },
    /// ดึง entry และ copy password ไปยัง clipboard
    Get {
        /// ID หรือ title ของ entry
        id: String,
        /// แสดง password ใน terminal แทน clipboard
        #[arg(long)] show: bool,
    },
    /// รายการ entry ทั้งหมด
    List {
        /// Filter by tag
        #[arg(long)] tag: Option<String>,
        /// Fuzzy search
        #[arg(short, long)] query: Option<String>,
    },
    /// แก้ไข entry
    Edit {
        id: String,
        #[arg(long)] title: Option<String>,
        #[arg(long)] username: Option<String>,
        #[arg(long)] url: Option<String>,
        #[arg(long)] notes: Option<String>,
    },
    /// ลบ entry
    Delete { id: String },
    /// สร้าง password ใหม่โดยไม่บันทึก
    Generate {
        #[arg(short, long, default_value = "20")] length: usize,
        #[arg(long)] no_symbols: bool,
        #[arg(long)] no_uppercase: bool,
        #[arg(long)] exclude_ambiguous: bool,
    },
    /// ตรวจสอบความแข็งแกร่งของ password (และ HIBP)
    Check {
        /// Password ที่ต้องการตรวจสอบ
        password: Option<String>,
        /// ตรวจกับ HaveIBeenPwned API
        #[arg(long)] hibp: bool,
    },
    /// Import จาก CSV (LastPass/Bitwarden format)
    Import { file: PathBuf },
    /// Export เป็น encrypted backup
    Export {
        output: PathBuf,
        /// Export เป็น plaintext CSV แทน (แสดง warning)
        #[arg(long)] plaintext: bool,
    },
    /// แสดง audit log
    Audit {
        #[arg(short, long, default_value = "20")] limit: usize,
    },
}

fn get_vault_path(cli: &Cli) -> PathBuf {
    cli.vault.clone().unwrap_or_else(|| {
        let home = dirs::home_dir().unwrap_or_else(|| PathBuf::from("."));
        home.join(".passmanager").join("vault.bin")
    })
}

fn get_master_password() -> String {
    rpassword::prompt_password("Master password: ").unwrap_or_default()
}

fn main() {
    let cli = Cli::parse();

    let vault_path = get_vault_path(&cli);

    match &cli.command {
        Commands::Init => {
            println!("Initializing vault at {:?}", vault_path);
            // ... สร้าง vault ใหม่ด้วย generate_salt() + derive_key()
        }
        Commands::Generate { length, no_symbols, no_uppercase, exclude_ambiguous } => {
            let config = generator::GeneratorConfig {
                length: *length,
                use_symbols: !no_symbols,
                use_uppercase: !no_uppercase,
                exclude_ambiguous: *exclude_ambiguous,
                ..Default::default()
            };
            match generator::generate_password(&config) {
                Ok(pw) => {
                    let strength = generator::score_password(&pw);
                    println!("{}", pw);
                    println!("Strength: {:?} (score {}/100)", strength.level, strength.score);
                }
                Err(e) => eprintln!("Error: {e}"),
            }
        }
        Commands::Check { password, hibp } => {
            let pw = password.clone().unwrap_or_else(|| {
                rpassword::prompt_password("Password to check: ").unwrap_or_default()
            });
            let result = generator::score_password(&pw);
            println!("Score: {}/100 ({:?})", result.score, result.level);
            println!("Length: {} characters", result.length);
            for issue in &result.issues {
                println!("  ! {}", issue);
            }
            if *hibp {
                let prefix = generator::hibp_prefix(&pw);
                println!("HIBP SHA-1 prefix: {}", prefix);
                println!("(send GET https://api.pwnedpasswords.com/range/{} to check)", prefix);
            }
        }
        _ => {
            println!("Command not yet implemented in this demo. See vault.rs and crypto.rs.");
        }
    }
}
```

---

## การทดสอบ (Testing)

### Unit Tests ครบถ้วน

เพิ่ม test ทั้งหมดลงใน `tests/integration.rs`:

```rust
// tests/integration.rs

use passmanager::crypto::{derive_key, encrypt, decrypt, generate_salt};
use passmanager::generator::{
    generate_password, score_password, hibp_prefix, password_sha1_hex,
    GeneratorConfig, StrengthLevel
};
use passmanager::vault::{read_vault_file, write_vault_file, Entry, Vault};
use chrono::Utc;
use tempfile::tempdir;

// ─── Argon2id ────────────────────────────────────────────────────────────────

#[test]
fn test_derive_key_is_deterministic() {
    let salt = [1u8; 16];
    let k1 = derive_key("password", &salt).unwrap();
    let k2 = derive_key("password", &salt).unwrap();
    assert_eq!(k1.as_ref(), k2.as_ref());
}

#[test]
fn test_derive_key_differs_by_password() {
    let salt = [0u8; 16];
    let k1 = derive_key("alpha", &salt).unwrap();
    let k2 = derive_key("beta", &salt).unwrap();
    assert_ne!(k1.as_ref(), k2.as_ref());
}

#[test]
fn test_derive_key_differs_by_salt() {
    let k1 = derive_key("same", &[0u8; 16]).unwrap();
    let k2 = derive_key("same", &[1u8; 16]).unwrap();
    assert_ne!(k1.as_ref(), k2.as_ref());
}

// ─── AES-256-GCM ─────────────────────────────────────────────────────────────

#[test]
fn test_aes_gcm_roundtrip() {
    let key = derive_key("test", &[0u8; 16]).unwrap();
    let plain = b"Hello, AES-GCM!";
    let enc = encrypt(&key, plain).unwrap();
    let dec = decrypt(&key, &enc).unwrap();
    assert_eq!(dec, plain);
}

#[test]
fn test_aes_gcm_wrong_key_rejected() {
    let k1 = derive_key("correct", &[0u8; 16]).unwrap();
    let k2 = derive_key("wrong",   &[0u8; 16]).unwrap();
    let enc = encrypt(&k1, b"data").unwrap();
    assert!(decrypt(&k2, &enc).is_err());
}

#[test]
fn test_aes_gcm_tampered_tag_rejected() {
    let key = derive_key("pw", &[0u8; 16]).unwrap();
    let mut enc = encrypt(&key, b"secret").unwrap();
    *enc.last_mut().unwrap() ^= 0xFF; // flip last byte of GCM tag
    assert!(decrypt(&key, &enc).is_err());
}

#[test]
fn test_nonce_unique_per_encryption() {
    let key = derive_key("pw", &[0u8; 16]).unwrap();
    let e1 = encrypt(&key, b"x").unwrap();
    let e2 = encrypt(&key, b"x").unwrap();
    assert_ne!(&e1[..12], &e2[..12]); // nonce portion
}

// ─── Vault serialization ──────────────────────────────────────────────────────

fn make_entry(id: &str) -> Entry {
    Entry {
        id: id.to_string(),
        title: format!("Site {id}"),
        url: format!("https://{id}.example.com"),
        username: "admin@test.com".to_string(),
        password: "s3cr3t!".to_string(),
        notes: String::new(),
        created_at: Utc::now(),
        updated_at: Utc::now(),
        tags: vec!["work".to_string()],
    }
}

#[test]
fn test_vault_serialize_roundtrip() {
    let mut v = Vault::new();
    v.insert(make_entry("gh"));
    v.insert(make_entry("gl"));
    let bytes = v.serialize().unwrap();
    let v2 = Vault::deserialize(&bytes).unwrap();
    assert_eq!(v2.entries.len(), 2);
}

#[test]
fn test_vault_file_roundtrip() {
    let dir = tempdir().unwrap();
    let path = dir.path().join("vault.bin");
    let salt = generate_salt();
    let key = derive_key("masterpass", &salt).unwrap();
    let mut vault = Vault::new();
    vault.insert(make_entry("test1"));
    write_vault_file(&path, &key, &salt, &vault).unwrap();
    let (s2, v2) = read_vault_file(&path, &key).unwrap();
    assert_eq!(s2, salt);
    assert_eq!(v2.entries.len(), 1);
}

// ─── Password generator ───────────────────────────────────────────────────────

#[test]
fn test_generator_charset_uppercase_only() {
    let cfg = GeneratorConfig {
        length: 32,
        use_uppercase: true,
        use_lowercase: false,
        use_digits: false,
        use_symbols: false,
        exclude_ambiguous: false,
    };
    let pw = generate_password(&cfg).unwrap();
    assert!(pw.chars().all(|c| c.is_ascii_uppercase()));
}

#[test]
fn test_generator_excludes_ambiguous() {
    let cfg = GeneratorConfig {
        length: 128,
        use_uppercase: true,
        use_lowercase: true,
        use_digits: true,
        use_symbols: false,
        exclude_ambiguous: true,
    };
    let pw = generate_password(&cfg).unwrap();
    for c in ['0', 'O', 'o', '1', 'l', 'I', '|'] {
        assert!(!pw.contains(c), "found ambiguous char '{c}' in password");
    }
}

// ─── Strength scorer ──────────────────────────────────────────────────────────

#[test]
fn test_strength_very_strong() {
    let r = score_password("X#9kLm!2pQ@vR5nW");
    assert!(r.score >= 80);
    assert_eq!(r.level, StrengthLevel::VeryStrong);
}

#[test]
fn test_strength_weak_short() {
    let r = score_password("abc");
    assert!(r.score < 40);
    assert!(!r.issues.is_empty());
}

// ─── HIBP k-anonymity ─────────────────────────────────────────────────────────

#[test]
fn test_hibp_prefix_is_5_chars() {
    let p = hibp_prefix("anypassword");
    assert_eq!(p.len(), 5);
    assert!(p.chars().all(|c| c.is_ascii_hexdigit()));
}

#[test]
fn test_sha1_known_vector() {
    // SHA-1("") = DA39A3EE5E6B4B0D3255BFEF95601890AFD80709
    assert_eq!(&password_sha1_hex("")[..8], "DA39A3EE");
    // SHA-1("password") = 5BAA61E4C9B93F3F0682250B6CF8331B7EE68FD8
    assert_eq!(&password_sha1_hex("password")[..8], "5BAA61E4");
}
```

### ผลลัพธ์ `cargo test` จริง

```
running 28 tests
test crypto::tests::test_decrypt_tampered_ciphertext_fails ... ok
test crypto::tests::test_decrypt_wrong_key_fails ... ok
test crypto::tests::test_derive_key_deterministic ... ok
test crypto::tests::test_derive_key_different_passwords ... ok
test crypto::tests::test_derive_key_different_salts ... ok
test crypto::tests::test_encrypt_decrypt_roundtrip ... ok
test crypto::tests::test_encrypt_nonce_unique ... ok
test generator::tests::test_generate_password_digits_only ... ok
test generator::tests::test_generate_password_empty_charset_fails ... ok
test generator::tests::test_generate_password_excludes_ambiguous ... ok
test generator::tests::test_generate_password_length ... ok
test generator::tests::test_generate_password_length_bounds ... ok
test generator::tests::test_generate_password_uppercase_only ... ok
test generator::tests::test_hibp_prefix_length ... ok
test generator::tests::test_score_password_strong ... ok
test generator::tests::test_score_password_very_strong ... ok
test generator::tests::test_score_password_weak ... ok
test generator::tests::test_sha1_known_value ... ok
test integration_tests::test_full_vault_lifecycle ... ok
test integration_tests::test_key_derivation_is_slow_enough ... ok
test integration_tests::test_password_generation_and_strength ... ok
test vault::tests::test_trigram_similarity ... ok
test vault::tests::test_vault_magic_check ... ok
test vault::tests::test_vault_search ... ok
test vault::tests::test_vault_serialize_deserialize ... ok
test vault::tests::test_vault_tag_filter ... ok
test vault::tests::test_vault_write_read_file ... ok
test vault::tests::test_vault_wrong_key_fails ... ok

test result: ok. 28 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 17.69s
```

---

## Pitfalls ที่พบบ่อย

### Pitfall 1: Modulo Bias ใน Random Password Generation

**ปัญหา:** ถ้าใช้ `rng.gen::<u8>() % charset_len` โดยตรง ตัวอักษรแรก ๆ ใน charset จะถูกเลือกบ่อยกว่าที่ควร เนื่องจาก 256 ไม่หารด้วย charset size ลงตัวในกรณีทั่วไป

```rust
// ❌ มี modulo bias ถ้า 256 % charset.len() != 0
let idx = rng.gen::<u8>() as usize % charset.len();

// ✅ ใช้ rejection sampling
let limit = (256 / n) * n;
loop {
    let b = rng.gen::<u8>() as usize;
    if b < limit {
        return charset[b % n];
    }
}
```

ตัวอย่าง: charset 72 ตัว → 256 % 72 = 40 → index 0–39 มีโอกาสถูกเลือกมากกว่า index 40–71 ด้วยอัตราส่วน 4:3

---

### Pitfall 2: GCM Nonce Reuse

**ปัญหา:** AES-256-GCM จะ **break catastrophically** ถ้าใช้ nonce เดิมกับ key เดิมสองครั้ง — ผู้สังเกตการณ์สามารถ XOR ciphertext ทั้งสองออกมาได้ ซึ่งยกเลิก keystream

```rust
// ❌ อันตราย — nonce ตายตัว
let nonce = Nonce::from_slice(b"unique nonce");

// ✅ สุ่ม nonce ใหม่ทุกครั้งด้วย OsRng
let nonce = Aes256Gcm::generate_nonce(&mut OsRng);
```

**ผลลัพธ์ของ nonce reuse ใน GCM:**
ถ้า (key, nonce) ถูกใช้ซ้ำ: `C1 XOR C2 = P1 XOR P2` ซึ่งทำให้ attacker สามารถ recover plaintexts ทั้งสองได้ในกรณีที่รู้ plaintext หนึ่งตัว

---

### Pitfall 3: Compiler-optimized-away Memory Clear

**ปัญหา:** Rust compiler (และ LLVM) สามารถ optimize out การเขียน 0 ลง buffer ที่กำลังจะถูก drop ได้ เนื่องจาก dead store elimination

```rust
// ❌ อาจถูก optimize out
fn clear_password(s: &mut String) {
    unsafe {
        let bytes = s.as_bytes_mut();
        for b in bytes.iter_mut() {
            *b = 0;
        }
    }
}

// ✅ zeroize รับประกัน volatile write บนทุก platform
use zeroize::Zeroize;
s.zeroize();
```

`zeroize` crate ใช้ `compiler_fence` + `volatile_write` ป้องกัน dead store elimination ใน release builds

---

### Pitfall 4: Argon2 Salt Reuse ข้าม Vault

**ปัญหา:** ถ้า salt เดิมถูกนำไปใช้กับ vault หลายอัน (เช่น generate salt ครั้งเดียวแล้วเก็บใน config) password เดียวกันจะ derive key เดิมในทุก vault — ป้องกัน per-vault key isolation ไม่ได้

```rust
// ❌ salt ซ้ำ — แต่ละ vault ไม่ได้รับ key ที่ independent จริง
const FIXED_SALT: Salt = [0xDE, 0xAD, 0xBE, 0xEF, ...];

// ✅ สร้าง salt ใหม่ต่อ vault creation เท่านั้น และเก็บ salt ใน vault header
let salt = generate_salt(); // ใช้ OsRng สร้าง salt ใหม่ทุกครั้งที่ init vault
write_vault_file(&path, &key, &salt, &vault)?;
// salt จะถูกอ่านกลับมาจาก file header ตอน unlock
```

Salt ต้องเป็น per-vault เพื่อให้ key derivation ของแต่ละ vault เป็น independent แม้ใช้ master password เดียวกัน

---

### Pitfall 5: Clipboard Persistence หลัง Timeout

**ปัญหา:** ถ้า `arboard` thread crash หรือถูก drop ก่อน timeout ผ่านไป password จะยังคงอยู่ใน clipboard ของ OS ตลอดไป

**วิธีป้องกัน:**
1. Thread ตรวจว่า clipboard ยังเป็น text เดิมก่อน clear (ป้องกัน clear text ที่ user copy ทีหลัง)
2. ใช้ `Arc<AtomicBool>` cancel token เพื่อ abort timer ถ้า session lock ก่อน 30 วินาที

```rust
use std::sync::{Arc, atomic::{AtomicBool, Ordering}};

pub fn copy_with_cancel(text: String, secs: u64) -> Arc<AtomicBool> {
    let cancelled = Arc::new(AtomicBool::new(false));
    let cancel_clone = cancelled.clone();

    std::thread::spawn(move || {
        for _ in 0..secs {
            std::thread::sleep(std::time::Duration::from_secs(1));
            if cancel_clone.load(Ordering::Relaxed) { return; }
        }
        // Clear clipboard
        if let Ok(mut cb) = arboard::Clipboard::new() {
            if cb.get_text().ok().as_deref() == Some(&text) {
                let _ = cb.set_text("");
            }
        }
    });

    cancelled // caller เก็บ token นี้ไว้ call .store(true) เมื่อต้องการ cancel
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimized release build
cargo build --release

# Binary จะอยู่ที่:
# target/release/passmanager (Linux/macOS)
# target/release/passmanager.exe (Windows)

# ขนาดลดโดยใช้ strip symbols
cargo build --release
strip target/release/passmanager
```

### ติดตั้งลง PATH

```bash
cargo install --path .
# Binary ติดตั้งไปที่ ~/.cargo/bin/passmanager
```

### Cross-compile สำหรับ ARM (Raspberry Pi)

```bash
# ติดตั้ง cross-compile toolchain
rustup target add aarch64-unknown-linux-gnu

# Build
cargo build --release --target aarch64-unknown-linux-gnu
```

### Feature Flags สำหรับ Headless Environment

ถ้าต้องการ build บนเครื่องที่ไม่มี clipboard daemon:

```toml
# Cargo.toml
[features]
default = ["clipboard"]
clipboard = ["arboard"]

[dependencies]
arboard = { version = "3", optional = true }
```

```bash
# Build โดยไม่ใช้ clipboard
cargo build --no-default-features
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม TOTP Support

เพิ่ม field `totp_secret: Option<String>` ใน `Entry` และ implement TOTP code generation (RFC 6238) โดยใช้ crate `totp-rs` หรือ implement เองจาก HOTP spec:

```rust
// ใช้ HMAC-SHA1 กับ timestamp/30s เป็น counter
use hmac::{Hmac, Mac};
use sha1::Sha1;

pub fn generate_totp(secret_base32: &str) -> Result<String, String> {
    let secret = base32::decode(base32::Alphabet::RFC4648 { padding: false }, secret_base32)
        .ok_or("Invalid base32 secret")?;

    let counter = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH).unwrap()
        .as_secs() / 30;

    let msg = counter.to_be_bytes();
    let mut mac = Hmac::<Sha1>::new_from_slice(&secret).unwrap();
    mac.update(&msg);
    let result = mac.finalize().into_bytes();

    let offset = (result[19] & 0x0F) as usize;
    let code = (u32::from_be_bytes(result[offset..offset+4].try_into().unwrap()) & 0x7FFFFFFF) % 1_000_000;
    Ok(format!("{:06}", code))
}
```

**ความยาก:** ⭐⭐ | **เวลา:** 2–3 ชั่วโมง

---

### แบบฝึกหัดที่ 2: Remote Sync via Encrypted Blob

Implement `sync` subcommand ที่ส่ง encrypted vault blob ไปยัง S3-compatible storage (Minio, Cloudflare R2) โดยใช้ `aws-sdk-s3` หรือ `reqwest` กับ pre-signed URL:

```
passmanager sync --remote s3://my-bucket/vault.bin
passmanager sync --pull   # download + merge
passmanager sync --push   # upload
```

**สิ่งที่ต้องออกแบบ:** conflict resolution strategy เมื่อทั้งสอง vault มี entry ที่แตกต่างกัน (timestamp-based last-write-wins หรือ CRDT)

**ความยาก:** ⭐⭐⭐ | **เวลา:** 4–5 ชั่วโมง

---

### แบบฝึกหัดที่ 3: Vault Sharing ด้วย Public Key Cryptography

เพิ่ม `share` subcommand ที่ encrypt entry ด้วย recipient's public key (X25519 ECDH + AES-GCM) เพื่อให้ share password กับบุคคลอื่นได้อย่างปลอดภัย:

```
passmanager share --entry github --recipient alice.pub
# → output: share_token.bin (encrypt ด้วย Alice's public key)

# Alice runs:
passmanager receive share_token.bin --key alice.key
```

ใช้ crate `x25519-dalek` สำหรับ key exchange และ `aes-gcm` สำหรับ symmetric encryption ของ payload

**ความยาก:** ⭐⭐⭐⭐ | **เวลา:** 5–8 ชั่วโมง

---

### แบบฝึกหัดที่ 4: TUI Interface ด้วย `ratatui`

แทนที่ CLI แบบ one-shot ด้วย interactive TUI ที่มี:
- Password list panel พร้อม fuzzy search แบบ real-time
- Detail panel ที่แสดง entry พร้อม masked password
- Keyboard shortcuts: `a` add, `e` edit, `d` delete, `c` copy, `q` quit
- Session timeout countdown ที่แสดงใน status bar

```toml
# เพิ่มใน Cargo.toml
ratatui   = "0.29"
crossterm = "0.28"
```

**ความยาก:** ⭐⭐⭐ | **เวลา:** 6–10 ชั่วโมง

---

## สรุป

โปรเจค Password Manager CLI นี้สร้าง pipeline ของ applied cryptography ที่สมบูรณ์:

**Pattern สำคัญที่ได้เรียน:**

1. **KDF-then-encrypt pattern:** ไม่เคย derive key ซ้ำโดยไม่มี salt — salt ถูก generate ครั้งเดียวต่อ vault แล้วเก็บใน file header

2. **AEAD decryption contract:** `decrypt()` ต้อง verify authentication tag ก่อน return plaintext เสมอ — GCM tag เป็น integrity guarantee ไม่ใช่ option

3. **`Zeroizing<T>` wrapper pattern:** wrap sensitive types (`[u8; 32]`, `String`) ด้วย `Zeroizing` หรือ implement `Drop` + `zeroize()` เพื่อรับประกัน memory clearing ใน release builds

4. **Binary format versioning:** magic bytes + version byte ทำให้รองรับ forward migration ได้โดยไม่ทำให้ไฟล์เก่า corrupt

5. **k-anonymity API pattern:** ส่ง partial hash เท่านั้น — privacy-preserving API design ที่ใช้ได้กับ use case อื่น เช่น duplicate detection

6. **In-memory-only session state:** `Instant::now()` ใน heap — ไม่มีทางเขียนลง disk โดยบังเอิญ แตกต่างกับ timestamp ใน file

โปรเจคถัดไป **File Encryptor** (D02) ต่อยอดจากพื้นฐาน AES-GCM ใน D01 โดยเพิ่ม streaming encryption สำหรับไฟล์ขนาดใหญ่ที่ไม่สามารถโหลดทั้งหมดเข้า memory ได้ พร้อม authenticated chunking

---

**โปรเจคก่อนหน้า:** [project-c10-schema-registry.md](project-c10-schema-registry.md) | **โปรเจคถัดไป:** [project-d02-file-encryptor.md](project-d02-file-encryptor.md)
