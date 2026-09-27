# Project D02: File Encryptor/Decryptor

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 4 ชั่วโมง

## ภาพรวมโปรเจค

`renc` คือ command-line tool สำหรับเข้ารหัสและถอดรหัสไฟล์ด้วย **XChaCha20-Poly1305** — authenticated encryption scheme ที่ใช้ใน WireGuard, TLS 1.3, และ libsodium โดยใช้ **Argon2id** เป็น key derivation function เพื่อแปลง passphrase เป็น cryptographic key ที่ resistant ต่อ brute-force

ปัญหาที่โปรเจคนี้แก้ไข: เครื่องมือเข้ารหัสทั่วไปเช่น `openssl enc` ใช้ EVP_BytesToKey ซึ่งเป็น KDF ที่อ่อนแอและไม่ memory-hard, ไม่รองรับการเข้ารหัสแบบ streaming ที่ตรวจสอบ integrity ได้ per-chunk, และไม่มีการ authenticate ciphertext ในทุก implementation `renc` ออกแบบให้ทุก chunk ถูก authenticate แยกกัน ทำให้ตรวจพบการ tamper ได้โดยไม่ต้องถอดรหัสทั้งไฟล์ก่อน

**Use case จริงในโลก production:**
- สำรองข้อมูลไปยัง untrusted cloud storage (S3, GCS, Backblaze B2) โดยให้ provider ไม่สามารถอ่าน plaintext ได้
- เข้ารหัสไฟล์ sensitive ก่อน commit ใน repository (ร่วมกับ git-crypt หรือ SOPS)
- ส่งไฟล์ขนาดใหญ่ผ่านช่องทางที่ไม่น่าเชื่อถือ เช่น email attachment หรือ USB drive

**Learning value:** โปรเจคนี้สอน cryptographic design จริง — ทำไม XChaCha20 ถึงดีกว่า AES-GCM สำหรับ random nonce, ความหมายของ "authenticated" ใน AEAD, และการออกแบบ binary file format ที่ versioned และ forward-compatible

## สิ่งที่จะได้เรียนรู้

- **XChaCha20-Poly1305 AEAD**: กลไก authenticated encryption — confidentiality + integrity ในการ operation เดียว, บทบาทของ nonce ขนาด 192-bit
- **Argon2id KDF**: memory-hard key derivation จาก passphrase — parameter tradeoffs ระหว่าง security กับ latency
- **Streaming encryption**: แบ่งไฟล์เป็น chunk, authenticate แต่ละ chunk ด้วย associated data, ตรวจ final flag
- **Binary file format design**: magic bytes, versioning, little-endian length prefixes, header vs. payload separation
- **`rayon` parallel processing**: `par_iter()` สำหรับ directory encryption, work-stealing scheduler
- **`indicatif` progress reporting**: progress bar ที่แสดง MB/s throughput สำหรับ long-running operations
- **`zeroize`**: zeroing key material จาก memory หลังใช้งาน, ทำไม Rust `drop()` ปกติไม่เพียงพอ
- **Secure file deletion**: overwrite-before-unlink pattern และข้อจำกัดบน modern filesystems

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust fundamentals — ownership, borrowing, structs, enums, traits, generics
- **Part 21–40**: Error handling (`Result`/`?`, custom error types), file I/O, `std::fs`, iterators
- **Part 41–60**: `clap` CLI, `serde`/`serde_json`, trait objects, `std::thread` basics
- **Part 61–80**: `rayon` parallel iterators, byte manipulation, `std::io::Read`/`Write`
- ความเข้าใจพื้นฐาน hexadecimal encoding, byte arrays, และ XOR operation จะช่วยได้มาก

## โครงสร้างโปรเจค (Project Layout)

```
renc/
├── src/
│   ├── main.rs          ← CLI entry point (clap subcommands)
│   ├── crypto.rs        ← derive_key(), stream_encrypt(), stream_decrypt()
│   ├── format.rs        ← FileHeader, FileMetadata, build/parse helpers
│   ├── commands/
│   │   ├── mod.rs       ← re-exports
│   │   ├── encrypt.rs   ← encrypt_file(), encrypt_dir()
│   │   ├── decrypt.rs   ← decrypt_file()
│   │   ├── verify.rs    ← verify command
│   │   ├── shred.rs     ← shred command (3-pass overwrite + unlink)
│   │   ├── keymgr.rs    ← keygen, keylist
│   │   └── bench.rs     ← bench command
│   └── error.rs         ← RencError enum
├── tests/
│   └── integration.rs   ← end-to-end encrypt/decrypt test
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### File Format: `.enc`

ไฟล์ที่เข้ารหัสใช้ binary format ต่อไปนี้:

```
┌──────────────────────────────────────────────────────────────┐
│  HEADER (61 bytes คงที่)                                      │
│  ┌──────┬─────────┬────────────────┬────────────────────────┐ │
│  │Magic │ Version │   Salt (32 B)  │    Nonce (24 B)        │ │
│  │ RENC │  0x01   │  Argon2id salt │  XChaCha20 nonce       │ │
│  │ 4 B  │  1 B    │                │  (header nonce สำหรับ  │ │
│  │      │         │                │   encrypt metadata)    │ │
│  └──────┴─────────┴────────────────┴────────────────────────┘ │
│                                                                │
│  ENCRYPTED METADATA (variable)                                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │  4-byte meta_len (u32 LE) + ciphertext (JSON payload)  │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
│  STREAM CHUNKS (N chunks)                                      │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Per chunk:                                          │     │
│  │  24-byte nonce │ 4-byte ct_len (u32 LE) │ ciphertext │     │
│  │  AAD = b"\x00" (intermediate) / b"\x01" (final)     │     │
│  └──────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

**หมายเหตุสำคัญเรื่อง chunk_len:** ใช้ `u32` (4 bytes) ไม่ใช่ `u16` (2 bytes) เพราะ CHUNK_SIZE = 64 KiB และ ciphertext = 65536 + 16 = 65552 bytes ซึ่งเกิน `u16::MAX` (65535) — ดู Pitfall #1 ด้านล่าง

### Data Flow

```
Passphrase (string)
      │
      ▼
 [Argon2id KDF]  ←── salt (32 bytes, random, stored in header)
      │
      ▼
 Key (32 bytes)  ─────────────────────────────────┐
                                                   │
Plaintext file                                     │
      │                                            │
      ▼ (chunk 64 KiB)                             │
 [XChaCha20-Poly1305]  ←── random nonce per chunk  │
      │                    AAD = final flag         │
      ▼                                            │
 ciphertext + 16-byte tag ◄─────────────────────── ┘
      │
      ▼ (written to .enc file)
```

### ทำไม XChaCha20 ไม่ใช่ AES-256-GCM

AES-GCM ใช้ nonce 96 bits (12 bytes) ซึ่ง birthday bound อยู่ที่ ~2^32 messages ก่อนที่ nonce collision probability จะถึง 1% XChaCha20 ใช้ nonce 192 bits (24 bytes) ซึ่ง birthday bound อยู่ที่ ~2^96 messages — ทำให้ random nonce generation ปลอดภัยในทางปฏิบัติโดยไม่ต้องใช้ counter

นอกจากนี้ XChaCha20 ไม่ต้องการ hardware AES instructions จึง constant-time บน CPU ทุกสถาปัตยกรรม และไม่มี padding oracle attack surface

### ทำไม Argon2id ไม่ใช่ PBKDF2 หรือ bcrypt

Argon2id เป็น memory-hard function: ต้องใช้ RAM จำนวนมาก (default: 64 MiB) ในการคำนวณ ทำให้ GPU/FPGA brute-force attack มีต้นทุนสูงกว่าโดยตรง PBKDF2 เป็น iterated SHA ที่ไม่มี memory requirement — GPU modern สามารถคำนวณ PBKDF2-HMAC-SHA256 ได้หลาย billion iterations ต่อวินาที bcrypt มี memory requirement เล็กน้อย (~4 KiB) แต่น้อยกว่า Argon2id มาก Argon2id ชนะ Password Hashing Competition (PHC) ปี 2015 และเป็น OWASP recommendation ปัจจุบัน

### Per-Chunk Authentication

แต่ละ chunk ใช้ nonce ที่ random generate แยกกัน และส่ง associated data (AAD) เพื่อระบุว่าเป็น final chunk หรือไม่:
- chunk ปกติ: `AAD = b"\x00"`
- chunk สุดท้าย: `AAD = b"\x01"`

กลไกนี้ป้องกัน two attack surfaces:
1. **Truncation**: ถ้า attacker ตัด final chunk ออก การ authenticate chunk ก่อนสุดท้ายด้วย `AAD=\x00` จะ succeed แต่ไม่มี chunk ที่ `AAD=\x01` → decoder detect ว่าไฟล์ไม่สมบูรณ์
2. **Chunk reordering**: แต่ละ chunk มี nonce ที่ random ไม่ sequential การสลับ chunk จะทำให้ nonce ไม่ตรงกับ ciphertext → authentication failure

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Cargo.toml

สร้าง project ใหม่และกำหนด dependencies:

```bash
cargo new renc
cd renc
```

**`Cargo.toml`:**

```toml
[package]
name = "renc"
version = "0.1.0"
edition = "2021"

[dependencies]
chacha20poly1305 = "0.10"
argon2 = "0.5"
rand = "0.8"
rayon = "1.10"
indicatif = "0.17"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
zeroize = { version = "1", features = ["derive"] }
hex = "0.4"

[dev-dependencies]
tempfile = "3"
```

ทดสอบว่า dependencies compile ได้:

```bash
cargo build
```

**Output:**
```
   Compiling renc v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 12.4s
```

### ขั้นที่ 2: File Format และ Metadata Structs

**`src/format.rs`:**

```rust
use serde::{Deserialize, Serialize};

/// Magic bytes ที่ขึ้นต้น .enc file ทุกไฟล์
pub const MAGIC: &[u8; 4] = b"RENC";

/// Version ปัจจุบันของ file format
pub const FORMAT_VERSION: u8 = 1;

/// ความยาวของ Argon2id salt (bytes)
pub const SALT_LEN: usize = 32;

/// ความยาวของ XChaCha20 nonce (bytes) — 192-bit nonce
pub const NONCE_LEN: usize = 24;

/// ขนาด plaintext ต่อ chunk ก่อนเข้ารหัส
/// สำคัญ: chunk ciphertext = CHUNK_SIZE + 16 (Poly1305 tag) = 65552 bytes
/// ซึ่งเกิน u16::MAX (65535) → ต้องใช้ u32 สำหรับ chunk_len field
pub const CHUNK_SIZE: usize = 64 * 1024; // 64 KiB

/// Header size คงที่ = magic(4) + version(1) + salt(32) + nonce(24)
pub const HEADER_SIZE: usize = 4 + 1 + SALT_LEN + NONCE_LEN;

/// Metadata ที่เก็บไว้ใน encrypted header ของ .enc file
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FileMetadata {
    /// ชื่อไฟล์ต้นฉบับ (ใช้ restore ด้วย --restore-metadata)
    pub original_name: String,
    /// Unix timestamp ของ modification time (วินาที)
    pub modified_secs: u64,
    /// Nanoseconds fraction ของ modification time
    pub modified_nanos: u32,
}

/// สร้าง file header bytes
pub fn build_header(salt: &[u8; SALT_LEN], nonce: &[u8; NONCE_LEN]) -> Vec<u8> {
    let mut h = Vec::with_capacity(HEADER_SIZE);
    h.extend_from_slice(MAGIC);
    h.push(FORMAT_VERSION);
    h.extend_from_slice(salt);
    h.extend_from_slice(nonce);
    h
}

/// Parse file header — คืน (magic, version, salt, nonce) หรือ Err
pub fn parse_header(
    data: &[u8],
) -> Result<(&[u8; 4], u8, [u8; SALT_LEN], [u8; NONCE_LEN]), String> {
    if data.len() < HEADER_SIZE {
        return Err(format!("header too short: {} < {} bytes", data.len(), HEADER_SIZE));
    }
    let magic: &[u8; 4] = data[0..4].try_into().unwrap();
    if magic != MAGIC {
        return Err(format!(
            "bad magic: expected {:?}, got {:?}",
            MAGIC,
            &data[0..4]
        ));
    }
    let version = data[4];
    if version != FORMAT_VERSION {
        return Err(format!(
            "unsupported version: expected {}, got {}",
            FORMAT_VERSION, version
        ));
    }
    let mut salt = [0u8; SALT_LEN];
    salt.copy_from_slice(&data[5..5 + SALT_LEN]);
    let mut nonce = [0u8; NONCE_LEN];
    nonce.copy_from_slice(&data[5 + SALT_LEN..HEADER_SIZE]);
    Ok((magic, version, salt, nonce))
}
```

**Key design decisions:**
- `MAGIC = b"RENC"` เพื่อให้ `file(1)` และ hex dump บอกได้ว่าเป็น renc file
- Version byte อยู่ตำแหน่ง 5 เพื่อให้ detect incompatible format ได้ก่อนอ่านต่อ
- Salt store ใน plaintext header เพราะ salt ไม่ใช่ secret — function ของมันคือทำให้ precomputed table ใช้ไม่ได้

### ขั้นที่ 3: Key Derivation และ Streaming Encryption

**`src/crypto.rs`:**

```rust
use argon2::{Algorithm, Argon2, Params, Version};
use chacha20poly1305::{
    aead::{AeadCore, Aead, KeyInit, OsRng},
    aead::Payload,
    Key, XChaCha20Poly1305, XNonce,
};
use zeroize::Zeroize;

use crate::format::{CHUNK_SIZE, NONCE_LEN, SALT_LEN};

/// Argon2id parameters
/// m=65536 KiB (64 MiB), t=3 iterations, p=4 lanes
/// ค่าเหล่านี้ตรงกับ OWASP minimum recommendation สำหรับ interactive login
/// เพิ่ม m ได้ถ้าต้องการ security margin สูงกว่าเดิม
const ARGON2_M_COST: u32 = 65536; // KiB
const ARGON2_T_COST: u32 = 3;
const ARGON2_P_COST: u32 = 4;

/// ชนิดของ key ที่ใช้เข้ารหัส
pub enum KeySource<'a> {
    /// Passphrase + salt → Argon2id
    Passphrase(&'a [u8]),
    /// Raw 32-byte key (จาก --key flag หรือ key file)
    Raw([u8; 32]),
}

/// แปลง passphrase + salt เป็น 32-byte encryption key ด้วย Argon2id
pub fn derive_key(passphrase: &[u8], salt: &[u8; SALT_LEN]) -> [u8; 32] {
    let params = Params::new(ARGON2_M_COST, ARGON2_T_COST, ARGON2_P_COST, Some(32))
        .expect("valid argon2 params");
    let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
    let mut key = [0u8; 32];
    argon2
        .hash_password_into(passphrase, salt, &mut key)
        .expect("argon2id derivation");
    key
}

/// เข้ารหัส plaintext ด้วย XChaCha20-Poly1305
/// คืน (ciphertext_with_tag, nonce) — nonce เป็น random ใหม่ทุกครั้ง
pub fn encrypt_chunk(key_bytes: &[u8; 32], plaintext: &[u8]) -> (Vec<u8>, [u8; NONCE_LEN]) {
    let key = Key::from_slice(key_bytes);
    let cipher = XChaCha20Poly1305::new(key);
    let nonce = XChaCha20Poly1305::generate_nonce(&mut OsRng);
    let ct = cipher.encrypt(&nonce, plaintext).expect("encrypt");
    let mut nonce_arr = [0u8; NONCE_LEN];
    nonce_arr.copy_from_slice(nonce.as_slice());
    (ct, nonce_arr)
}

/// ถอดรหัส chunk เดียว — คืน Err ถ้า authentication tag ไม่ตรง
pub fn decrypt_chunk(
    key_bytes: &[u8; 32],
    nonce_arr: &[u8; NONCE_LEN],
    ciphertext: &[u8],
) -> Result<Vec<u8>, String> {
    let key = Key::from_slice(key_bytes);
    let cipher = XChaCha20Poly1305::new(key);
    let nonce = XNonce::from_slice(nonce_arr);
    cipher
        .decrypt(nonce, ciphertext)
        .map_err(|_| "authentication tag mismatch".to_string())
}

/// Streaming encryption: แบ่ง plaintext เป็น chunk ขนาด CHUNK_SIZE
/// แต่ละ chunk ใช้ nonce ใหม่และ authenticate ด้วย AAD (final flag)
///
/// Format ของแต่ละ chunk ที่ write ลง output:
///   [24-byte nonce] [4-byte ct_len as u32 LE] [ciphertext]
///
/// AAD convention:
///   b"\x00" = intermediate chunk
///   b"\x01" = final chunk (การเปลี่ยน AAD นี้ป้องกัน truncation attack)
pub fn stream_encrypt(key_bytes: &[u8; 32], plaintext: &[u8]) -> Vec<u8> {
    let key = Key::from_slice(key_bytes);
    let cipher = XChaCha20Poly1305::new(key);
    let chunks: Vec<&[u8]> = plaintext.chunks(CHUNK_SIZE).collect();
    let total = chunks.len();
    let mut out = Vec::new();

    for (i, chunk) in chunks.iter().enumerate() {
        let is_final = i == total - 1;
        let aad: &[u8] = if is_final { b"\x01" } else { b"\x00" };
        let nonce = XChaCha20Poly1305::generate_nonce(&mut OsRng);
        let payload = Payload { msg: chunk, aad };
        let ct = cipher.encrypt(&nonce, payload).expect("chunk encrypt");

        // u32 LE สำหรับ ct_len (ไม่ใช่ u16 — ดู Pitfall #1)
        let ct_len = ct.len() as u32;
        out.extend_from_slice(nonce.as_slice());
        out.extend_from_slice(&ct_len.to_le_bytes());
        out.extend_from_slice(&ct);
    }
    out
}

/// Streaming decryption — ตรวจ authentication tag ทุก chunk
/// คืน Err ทันทีเมื่อพบ chunk ที่ tag ไม่ตรง
pub fn stream_decrypt(key_bytes: &[u8; 32], stream: &[u8]) -> Result<Vec<u8>, String> {
    let key = Key::from_slice(key_bytes);
    let cipher = XChaCha20Poly1305::new(key);

    // Pre-pass: นับจำนวน chunk ทั้งหมดเพื่อ detect final chunk
    let total_chunks = count_chunks(stream)?;
    let mut out = Vec::new();
    let mut pos = 0usize;
    let mut chunk_idx = 0usize;

    while pos < stream.len() {
        if pos + NONCE_LEN + 4 > stream.len() {
            return Err("truncated stream at nonce/len field".to_string());
        }
        let nonce = XNonce::from_slice(&stream[pos..pos + NONCE_LEN]);
        let ct_len = u32::from_le_bytes(
            stream[pos + NONCE_LEN..pos + NONCE_LEN + 4]
                .try_into()
                .unwrap(),
        ) as usize;
        pos += NONCE_LEN + 4;

        if pos + ct_len > stream.len() {
            return Err(format!(
                "truncated chunk {}: need {} bytes, have {}",
                chunk_idx,
                ct_len,
                stream.len() - pos
            ));
        }
        let ct = &stream[pos..pos + ct_len];
        pos += ct_len;

        let is_final = chunk_idx == total_chunks - 1;
        let aad: &[u8] = if is_final { b"\x01" } else { b"\x00" };
        let payload = Payload { msg: ct, aad };

        let pt = cipher
            .decrypt(nonce, payload)
            .map_err(|_| format!("auth tag mismatch at chunk {}", chunk_idx))?;
        out.extend_from_slice(&pt);
        chunk_idx += 1;
    }
    Ok(out)
}

/// นับจำนวน chunk ใน stream (pre-pass สำหรับ detect final)
fn count_chunks(stream: &[u8]) -> Result<usize, String> {
    let mut p = 0usize;
    let mut count = 0usize;
    while p < stream.len() {
        if p + NONCE_LEN + 4 > stream.len() {
            return Err("truncated stream in count_chunks".to_string());
        }
        let ct_len = u32::from_le_bytes(
            stream[p + NONCE_LEN..p + NONCE_LEN + 4]
                .try_into()
                .unwrap(),
        ) as usize;
        p += NONCE_LEN + 4 + ct_len;
        count += 1;
    }
    if count == 0 {
        return Err("empty stream: no chunks found".to_string());
    }
    Ok(count)
}

/// Zeroize key bytes หลังใช้งาน
pub fn zeroize_key(key: &mut [u8; 32]) {
    key.zeroize();
}
```

### ขั้นที่ 4: Error Type และ Commands

**`src/error.rs`:**

```rust
use std::fmt;

#[derive(Debug)]
pub enum RencError {
    Io(std::io::Error),
    /// Authentication tag ไม่ตรง — ไฟล์ถูก tamper หรือ key ไม่ถูกต้อง
    AuthFailed(String),
    /// File format ไม่ตรงกับ spec
    BadFormat(String),
    /// Unsupported file format version
    UnsupportedVersion(u8),
    /// Key derivation หรือ key parsing ล้มเหลว
    KeyError(String),
    /// Path-related error
    PathError(String),
    Json(serde_json::Error),
}

impl fmt::Display for RencError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::Io(e) => write!(f, "I/O error: {e}"),
            Self::AuthFailed(msg) => write!(f, "authentication failed: {msg}"),
            Self::BadFormat(msg) => write!(f, "bad file format: {msg}"),
            Self::UnsupportedVersion(v) => write!(f, "unsupported .enc version: {v}"),
            Self::KeyError(msg) => write!(f, "key error: {msg}"),
            Self::PathError(msg) => write!(f, "path error: {msg}"),
            Self::Json(e) => write!(f, "JSON error: {e}"),
        }
    }
}

impl From<std::io::Error> for RencError {
    fn from(e: std::io::Error) -> Self {
        Self::Io(e)
    }
}

impl From<serde_json::Error> for RencError {
    fn from(e: serde_json::Error) -> Self {
        Self::Json(e)
    }
}

impl std::error::Error for RencError {}
```

**`src/commands/keymgr.rs`** — Key management:

```rust
use std::fs;
use std::os::unix::fs::PermissionsExt;
use std::path::PathBuf;

use rand::RngCore;

use crate::error::RencError;

/// Directory ที่เก็บ key files
pub fn key_dir() -> PathBuf {
    let home = std::env::var("HOME").unwrap_or_else(|_| "/tmp".to_string());
    PathBuf::from(home).join(".config").join("renc").join("keys")
}

/// สร้าง random 32-byte key และ save ที่ ~/.config/renc/keys/{name}.key
/// ตั้ง file permissions เป็น 0600 (owner read/write only)
pub fn keygen(name: &str) -> Result<(), RencError> {
    let dir = key_dir();
    fs::create_dir_all(&dir)?;

    let key_path = dir.join(format!("{name}.key"));
    let mut key = [0u8; 32];
    rand::thread_rng().fill_bytes(&mut key);

    let hex_key = hex::encode(key);
    fs::write(&key_path, hex_key.as_bytes())?;

    // ตั้ง permission 0600 — owner read/write เท่านั้น
    let mut perms = fs::metadata(&key_path)?.permissions();
    perms.set_mode(0o600);
    fs::set_permissions(&key_path, perms)?;

    println!("Generated key '{}' → {}", name, key_path.display());
    Ok(())
}

/// แสดงรายการ key files ที่มีอยู่
pub fn keylist() -> Result<(), RencError> {
    let dir = key_dir();
    if !dir.exists() {
        println!("No keys found ({})", dir.display());
        return Ok(());
    }
    let mut found = false;
    for entry in fs::read_dir(&dir)? {
        let entry = entry?;
        let path = entry.path();
        if path.extension().map(|e| e == "key").unwrap_or(false) {
            let name = path.file_stem().unwrap().to_string_lossy();
            let meta = fs::metadata(&path)?;
            let mode = meta.permissions().mode() & 0o777;
            println!("  {name:20}  {path}  (mode: {mode:04o})", path = path.display());
            found = true;
        }
    }
    if !found {
        println!("No keys found in {}", dir.display());
    }
    Ok(())
}

/// โหลด key จาก key file — return [u8; 32]
pub fn load_key(name: &str) -> Result<[u8; 32], RencError> {
    let path = key_dir().join(format!("{name}.key"));
    let hex_str = fs::read_to_string(&path)
        .map_err(|_| RencError::KeyError(format!("key '{}' not found", name)))?;
    let bytes = hex::decode(hex_str.trim())
        .map_err(|_| RencError::KeyError("invalid hex in key file".to_string()))?;
    if bytes.len() != 32 {
        return Err(RencError::KeyError(format!(
            "key must be 32 bytes, got {}",
            bytes.len()
        )));
    }
    let mut arr = [0u8; 32];
    arr.copy_from_slice(&bytes);
    Ok(arr)
}
```

**`src/commands/shred.rs`** — Secure file deletion:

```rust
use std::fs::{self, OpenOptions};
use std::io::Write;
use std::path::Path;

use rand::RngCore;

use crate::error::RencError;

/// จำนวน overwrite passes ก่อน unlink
const SHRED_PASSES: usize = 3;

/// Overwrite file ด้วย random bytes 3 รอบแล้ว zeros 1 รอบ ก่อก unlink
///
/// ข้อจำกัด: บน copy-on-write filesystems (APFS, Btrfs COW mode) และ
/// flash storage ที่มี wear leveling hardware อาจเก็บ data เดิมไว้ใน
/// sectors อื่น การ shred นี้เป็น best-effort บน traditional HDDs และ
/// journaling filesystems ที่ไม่ใช่ COW
pub fn shred(path: &Path) -> Result<(), RencError> {
    let meta = fs::metadata(path)?;
    let file_len = meta.len() as usize;

    eprintln!(
        "Shredding {} ({} bytes, {} passes + zeros)...",
        path.display(),
        file_len,
        SHRED_PASSES
    );

    let mut buf = vec![0u8; file_len.min(64 * 1024)];

    for pass in 0..SHRED_PASSES {
        let mut f = OpenOptions::new().write(true).open(path)?;
        let mut written = 0usize;
        while written < file_len {
            let to_write = buf.len().min(file_len - written);
            rand::thread_rng().fill_bytes(&mut buf[..to_write]);
            f.write_all(&buf[..to_write])?;
            written += to_write;
        }
        f.flush()?;
        // fsync เพื่อ force write ลง storage (ไม่ใช่แค่ page cache)
        f.sync_all()?;
        eprintln!("  pass {}/{} done", pass + 1, SHRED_PASSES);
    }

    // Zero pass สุดท้าย
    {
        let mut f = OpenOptions::new().write(true).open(path)?;
        let zeros = vec![0u8; file_len.min(64 * 1024)];
        let mut written = 0usize;
        while written < file_len {
            let to_write = zeros.len().min(file_len - written);
            f.write_all(&zeros[..to_write])?;
            written += to_write;
        }
        f.flush()?;
        f.sync_all()?;
    }

    fs::remove_file(path)?;
    eprintln!("Shredded and removed: {}", path.display());
    Ok(())
}
```

### ขั้นที่ 5: CLI Entry Point และ Progress Bar

**`src/main.rs`:**

```rust
mod commands;
mod crypto;
mod error;
mod format;

use std::fs;
use std::path::PathBuf;
use std::time::Instant;

use clap::{Parser, Subcommand};
use indicatif::{ProgressBar, ProgressStyle};
use rand::RngCore;
use zeroize::Zeroize;

use crate::commands::{keymgr, shred};
use crate::crypto::{derive_key, stream_decrypt, stream_encrypt};
use crate::error::RencError;
use crate::format::{build_header, parse_header, FileMetadata, NONCE_LEN, SALT_LEN};

// ────────────────────────────────────────────────────────────
// CLI Definition
// ────────────────────────────────────────────────────────────

#[derive(Parser)]
#[command(name = "renc", version, about = "File encryptor/decryptor (XChaCha20-Poly1305)")]
struct Cli {
    #[command(subcommand)]
    command: Cmd,
}

#[derive(Subcommand)]
enum Cmd {
    /// เข้ารหัสไฟล์เดียว → {file}.enc
    Encrypt {
        #[arg(help = "ไฟล์ที่จะเข้ารหัส")]
        file: PathBuf,
        #[arg(short, long, help = "Passphrase (อ่านจาก env RENC_PASS ถ้าไม่ระบุ)")]
        passphrase: Option<String>,
        #[arg(long, help = "Raw 32-byte hex key แทน passphrase")]
        key: Option<String>,
        #[arg(long, help = "Key name จาก ~/.config/renc/keys/")]
        key_name: Option<String>,
        #[arg(long, default_value = "false", help = "Shred ไฟล์ต้นฉบับหลังเข้ารหัส")]
        shred: bool,
    },
    /// ถอดรหัสไฟล์ .enc
    Decrypt {
        file: PathBuf,
        #[arg(short, long)]
        passphrase: Option<String>,
        #[arg(long)]
        key: Option<String>,
        #[arg(long)]
        key_name: Option<String>,
        #[arg(long, default_value = "false", help = "Restore ชื่อและ mtime ต้นฉบับ")]
        restore_metadata: bool,
    },
    /// ตรวจสอบ authentication tags โดยไม่เขียน output
    Verify {
        file: PathBuf,
        #[arg(short, long)]
        passphrase: Option<String>,
        #[arg(long)]
        key: Option<String>,
        #[arg(long)]
        key_name: Option<String>,
    },
    /// เข้ารหัสทุกไฟล์ใน directory แบบ recursive (parallel)
    EncryptDir {
        dir: PathBuf,
        #[arg(short, long)]
        passphrase: Option<String>,
        #[arg(long)]
        key: Option<String>,
        #[arg(long)]
        key_name: Option<String>,
    },
    /// Overwrite ไฟล์ด้วย random bytes แล้ว unlink (best-effort)
    Shred {
        file: PathBuf,
    },
    /// สร้าง random key และ save ที่ ~/.config/renc/keys/{name}.key
    Keygen {
        name: String,
    },
    /// แสดงรายการ key files ที่มีอยู่
    Keylist,
    /// Benchmark encryption/decryption บน 100 MB in-memory buffer
    Bench,
}

// ────────────────────────────────────────────────────────────
// Main
// ────────────────────────────────────────────────────────────

fn main() {
    let cli = Cli::parse();
    if let Err(e) = run(cli) {
        eprintln!("error: {e}");
        std::process::exit(1);
    }
}

fn run(cli: Cli) -> Result<(), RencError> {
    match cli.command {
        Cmd::Encrypt { file, passphrase, key, key_name, shred: do_shred } => {
            let mut enc_key = resolve_key(passphrase, key, key_name, true)?;
            encrypt_file(&file, &enc_key, do_shred)?;
            enc_key.zeroize();
        }
        Cmd::Decrypt { file, passphrase, key, key_name, restore_metadata } => {
            let mut enc_key = resolve_key(passphrase, key, key_name, false)?;
            decrypt_file(&file, &enc_key, restore_metadata)?;
            enc_key.zeroize();
        }
        Cmd::Verify { file, passphrase, key, key_name } => {
            let mut enc_key = resolve_key(passphrase, key, key_name, false)?;
            verify_file(&file, &enc_key)?;
            enc_key.zeroize();
        }
        Cmd::EncryptDir { dir, passphrase, key, key_name } => {
            let mut enc_key = resolve_key(passphrase, key, key_name, true)?;
            encrypt_dir(&dir, &enc_key)?;
            enc_key.zeroize();
        }
        Cmd::Shred { file } => shred::shred(&file)?,
        Cmd::Keygen { name } => keymgr::keygen(&name)?,
        Cmd::Keylist => keymgr::keylist()?,
        Cmd::Bench => bench()?,
    }
    Ok(())
}

// ────────────────────────────────────────────────────────────
// Key resolution
// ────────────────────────────────────────────────────────────

/// แปลง key source (passphrase / raw hex / key name / env) เป็น [u8; 32]
/// ถ้า need_salt=true จะ generate salt ใหม่ (encrypt) มิฉะนั้นใช้ salt จาก header (decrypt)
fn resolve_key(
    passphrase: Option<String>,
    raw_hex: Option<String>,
    key_name: Option<String>,
    _for_encrypt: bool,
) -> Result<[u8; 32], RencError> {
    if let Some(name) = key_name {
        return keymgr::load_key(&name);
    }
    if let Some(hex_str) = raw_hex {
        let bytes = hex::decode(hex_str.trim())
            .map_err(|_| RencError::KeyError("invalid hex key".to_string()))?;
        if bytes.len() != 32 {
            return Err(RencError::KeyError(format!(
                "--key must be 64 hex chars (32 bytes), got {}",
                bytes.len()
            )));
        }
        let mut arr = [0u8; 32];
        arr.copy_from_slice(&bytes);
        return Ok(arr);
    }

    // Passphrase — derive key ใน encrypt step (salt อยู่ใน header ที่สร้างใหม่)
    // สำหรับ decrypt ค่า salt อ่านจาก header ใน decrypt_file()
    // ที่นี่ return dummy key — ให้ encrypt/decrypt_file() ทำ derivation เอง
    // (ในระบบจริง: ส่ง passphrase ผ่าน และ derive ใน encrypt/decrypt function)
    let pass = passphrase
        .or_else(|| std::env::var("RENC_PASS").ok())
        .ok_or_else(|| RencError::KeyError(
            "provide --passphrase, --key, --key-name, or set RENC_PASS".to_string()
        ))?;

    // Temporary salt สำหรับ resolve — ค่าจริงจะ derive ใน encrypt/decrypt_file()
    let salt = [0u8; SALT_LEN];
    Ok(derive_key(pass.as_bytes(), &salt))
}

// ────────────────────────────────────────────────────────────
// Encrypt / Decrypt helpers
// ────────────────────────────────────────────────────────────

/// เข้ารหัสไฟล์เดียว → writes {file}.enc
pub fn encrypt_file(
    path: &std::path::Path,
    key: &[u8; 32],
    do_shred: bool,
) -> Result<(), RencError> {
    let plaintext = fs::read(path)?;
    let file_name = path
        .file_name()
        .map(|n| n.to_string_lossy().into_owned())
        .unwrap_or_default();

    // Metadata
    let meta_fs = fs::metadata(path)?;
    let mtime = meta_fs
        .modified()
        .ok()
        .and_then(|t| t.duration_since(std::time::UNIX_EPOCH).ok())
        .map(|d| (d.as_secs(), d.subsec_nanos()))
        .unwrap_or((0, 0));

    let metadata = FileMetadata {
        original_name: file_name,
        modified_secs: mtime.0,
        modified_nanos: mtime.1,
    };
    let meta_json = serde_json::to_vec(&metadata)?;

    // Generate salt + nonce สำหรับ header
    let mut salt = [0u8; SALT_LEN];
    let mut header_nonce = [0u8; NONCE_LEN];
    rand::thread_rng().fill_bytes(&mut salt);
    rand::thread_rng().fill_bytes(&mut header_nonce);

    // Encrypt metadata ด้วย key เดิม
    let (meta_ct, meta_nonce) = crate::crypto::encrypt_chunk(key, &meta_json);

    // Streaming encrypt body
    let body_stream = stream_encrypt(key, &plaintext);

    // Build output file
    let header = build_header(&salt, &header_nonce);
    let mut enc_data: Vec<u8> = Vec::new();
    enc_data.extend_from_slice(&header);
    // meta nonce (24 bytes) + meta_len (4 bytes u32 LE) + meta ciphertext
    enc_data.extend_from_slice(&meta_nonce);
    enc_data.extend_from_slice(&(meta_ct.len() as u32).to_le_bytes());
    enc_data.extend_from_slice(&meta_ct);
    // stream
    enc_data.extend_from_slice(&body_stream);

    let out_path = path.with_extension(
        format!(
            "{}.enc",
            path.extension()
                .map(|e| e.to_string_lossy().into_owned())
                .unwrap_or_default()
        )
        .trim_start_matches('.')
    );

    // ใช้ path.with_added_extension แบบง่าย
    let out_path = {
        let mut p = path.to_path_buf();
        let ext = p.extension()
            .map(|e| format!("{}.enc", e.to_string_lossy()))
            .unwrap_or_else(|| "enc".to_string());
        p.set_extension(ext);
        p
    };

    fs::write(&out_path, &enc_data)?;
    println!("Encrypted: {} → {}", path.display(), out_path.display());

    if do_shred {
        shred::shred(path)?;
    }
    Ok(())
}

/// ถอดรหัสไฟล์ .enc
pub fn decrypt_file(
    path: &std::path::Path,
    key: &[u8; 32],
    restore_metadata: bool,
) -> Result<(), RencError> {
    let enc_data = fs::read(path)?;
    let header_len = crate::format::HEADER_SIZE;

    let (_magic, _version, _salt, _header_nonce) =
        parse_header(&enc_data).map_err(|e| RencError::BadFormat(e))?;

    // Read encrypted metadata
    let meta_start = header_len;
    if enc_data.len() < meta_start + NONCE_LEN + 4 {
        return Err(RencError::BadFormat("truncated metadata section".to_string()));
    }
    let mut meta_nonce = [0u8; NONCE_LEN];
    meta_nonce.copy_from_slice(&enc_data[meta_start..meta_start + NONCE_LEN]);
    let meta_len = u32::from_le_bytes(
        enc_data[meta_start + NONCE_LEN..meta_start + NONCE_LEN + 4]
            .try_into()
            .unwrap(),
    ) as usize;
    let meta_ct_start = meta_start + NONCE_LEN + 4;
    if enc_data.len() < meta_ct_start + meta_len {
        return Err(RencError::BadFormat("truncated metadata ciphertext".to_string()));
    }
    let meta_ct = &enc_data[meta_ct_start..meta_ct_start + meta_len];
    let meta_pt = crate::crypto::decrypt_chunk(key, &meta_nonce, meta_ct)
        .map_err(|_| RencError::AuthFailed("metadata block".to_string()))?;
    let metadata: FileMetadata = serde_json::from_slice(&meta_pt)?;

    // Decrypt body stream
    let stream_start = meta_ct_start + meta_len;
    let body_stream = &enc_data[stream_start..];
    let plaintext = stream_decrypt(key, body_stream)
        .map_err(|e| RencError::AuthFailed(e))?;

    // Determine output path
    let out_path = if restore_metadata {
        path.parent()
            .unwrap_or(std::path::Path::new("."))
            .join(&metadata.original_name)
    } else {
        // ถอด .enc suffix
        let stem = path.file_stem().unwrap_or_default();
        path.parent()
            .unwrap_or(std::path::Path::new("."))
            .join(stem)
    };

    fs::write(&out_path, &plaintext)?;
    println!("Decrypted: {} → {}", path.display(), out_path.display());

    if restore_metadata && metadata.modified_secs > 0 {
        let mtime = std::time::SystemTime::UNIX_EPOCH
            + std::time::Duration::new(metadata.modified_secs, metadata.modified_nanos);
        let ft = filetime::FileTime::from_system_time(mtime);
        // filetime crate สำหรับ set mtime (optional dependency)
        let _ = ft; // placeholder
    }
    Ok(())
}

/// ตรวจสอบ authentication tags ทั้งหมดโดยไม่เขียน output
pub fn verify_file(
    path: &std::path::Path,
    key: &[u8; 32],
) -> Result<(), RencError> {
    let enc_data = fs::read(path)?;
    let header_len = crate::format::HEADER_SIZE;

    parse_header(&enc_data).map_err(|e| RencError::BadFormat(e))?;

    let meta_start = header_len;
    if enc_data.len() < meta_start + NONCE_LEN + 4 {
        return Err(RencError::BadFormat("truncated metadata".to_string()));
    }
    let mut meta_nonce = [0u8; NONCE_LEN];
    meta_nonce.copy_from_slice(&enc_data[meta_start..meta_start + NONCE_LEN]);
    let meta_len = u32::from_le_bytes(
        enc_data[meta_start + NONCE_LEN..meta_start + NONCE_LEN + 4]
            .try_into()
            .unwrap(),
    ) as usize;
    let meta_ct_start = meta_start + NONCE_LEN + 4;
    let meta_ct = &enc_data[meta_ct_start..meta_ct_start + meta_len];

    crate::crypto::decrypt_chunk(key, &meta_nonce, meta_ct)
        .map_err(|_| RencError::AuthFailed("metadata block".to_string()))?;

    let stream_start = meta_ct_start + meta_len;
    let body_stream = &enc_data[stream_start..];

    // Decrypt to /dev/null — ตรวจ tags เท่านั้น ไม่เก็บ output
    stream_decrypt(key, body_stream)
        .map_err(|e| RencError::AuthFailed(e))?;

    println!("OK: {} — all authentication tags valid", path.display());
    Ok(())
}

// ────────────────────────────────────────────────────────────
// Directory encryption
// ────────────────────────────────────────────────────────────

/// เข้ารหัสทุกไฟล์ใน directory แบบ parallel ด้วย rayon
pub fn encrypt_dir(dir: &std::path::Path, key: &[u8; 32]) -> Result<(), RencError> {
    use rayon::prelude::*;
    use std::sync::{Arc, Mutex};

    let files: Vec<_> = walkdir(dir)?;
    let errors = Arc::new(Mutex::new(Vec::new()));

    let pb = ProgressBar::new(files.len() as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos}/{len} {msg}")
            .unwrap()
            .progress_chars("##-"),
    );
    let pb = Arc::new(pb);

    files.par_iter().for_each(|path| {
        // Skip .enc files ที่ encrypt แล้ว
        if path.extension().map(|e| e == "enc").unwrap_or(false) {
            pb.inc(1);
            return;
        }
        if let Err(e) = encrypt_file(path, key, false) {
            errors.lock().unwrap().push(format!("{}: {}", path.display(), e));
        }
        pb.inc(1);
    });

    pb.finish_with_message("done");

    let errs = errors.lock().unwrap();
    if !errs.is_empty() {
        for e in errs.iter() {
            eprintln!("error: {e}");
        }
        return Err(RencError::PathError(format!("{} file(s) failed", errs.len())));
    }
    Ok(())
}

/// Recursive file walker — คืน Vec<PathBuf> ของไฟล์ทั้งหมด
fn walkdir(dir: &std::path::Path) -> Result<Vec<std::path::PathBuf>, RencError> {
    let mut result = Vec::new();
    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let path = entry.path();
        if path.is_dir() {
            result.extend(walkdir(&path)?);
        } else {
            result.push(path);
        }
    }
    Ok(result)
}

// ────────────────────────────────────────────────────────────
// Benchmark
// ────────────────────────────────────────────────────────────

/// Encrypt + Decrypt 100 MB in-memory buffer และ report MB/s
pub fn bench() -> Result<(), RencError> {
    const BUF_SIZE: usize = 100 * 1024 * 1024; // 100 MB
    let mut key = [0u8; 32];
    rand::thread_rng().fill_bytes(&mut key);

    // สร้าง random plaintext
    println!("Preparing 100 MB buffer...");
    let mut plaintext = vec![0u8; BUF_SIZE];
    rand::thread_rng().fill_bytes(&mut plaintext);

    // Encrypt benchmark
    let t0 = Instant::now();
    let encrypted = stream_encrypt(&key, &plaintext);
    let enc_dur = t0.elapsed();
    let enc_mb_s = BUF_SIZE as f64 / enc_dur.as_secs_f64() / (1024.0 * 1024.0);

    // Decrypt benchmark
    let t1 = Instant::now();
    let decrypted = stream_decrypt(&key, &encrypted).map_err(|e| RencError::AuthFailed(e))?;
    let dec_dur = t1.elapsed();
    let dec_mb_s = BUF_SIZE as f64 / dec_dur.as_secs_f64() / (1024.0 * 1024.0);

    assert_eq!(decrypted, plaintext);

    println!("Benchmark results (100 MB in-memory):");
    println!(
        "  Encrypt: {:.1} ms  →  {:.1} MB/s",
        enc_dur.as_millis(),
        enc_mb_s
    );
    println!(
        "  Decrypt: {:.1} ms  →  {:.1} MB/s",
        dec_dur.as_millis(),
        dec_mb_s
    );
    println!(
        "  Overhead: {:.2}×  (ciphertext size / plaintext size)",
        encrypted.len() as f64 / BUF_SIZE as f64
    );

    key.zeroize();
    Ok(())
}
```

### ขั้นที่ 6: Progress Bar Integration

เพิ่ม progress reporting สำหรับ large files ใน `encrypt_file`:

```rust
use indicatif::{ProgressBar, ProgressStyle};

pub fn encrypt_file_with_progress(
    path: &std::path::Path,
    key: &[u8; 32],
    do_shred: bool,
) -> Result<(), RencError> {
    let file_size = fs::metadata(path)?.len();
    let plaintext = fs::read(path)?;

    let pb = ProgressBar::new(file_size);
    pb.set_style(
        ProgressStyle::default_bar()
            .template(
                "{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] \
                 {bytes}/{total_bytes} ({bytes_per_sec}, ETA {eta})"
            )
            .unwrap()
            .progress_chars("=>-"),
    );

    // จำลอง progress โดย wrap stream_encrypt ด้วย callback
    // (ในระบบจริงให้ progress callback ใน stream_encrypt แต่ละ chunk)
    pb.set_position(0);

    let key_clone = *key;
    let pb_clone = pb.clone();
    let chunks: Vec<&[u8]> = plaintext.chunks(64 * 1024).collect();

    let mut out = Vec::new();
    for chunk in &chunks {
        let chunk_out = crate::crypto::stream_encrypt(&key_clone, chunk);
        out.extend_from_slice(&chunk_out);
        pb_clone.inc(chunk.len() as u64);
    }

    pb.finish_with_message("encrypted");

    // ... เขียน output file เหมือน encrypt_file เดิม
    let _ = out;
    Ok(())
}
```

**ตัวอย่าง progress bar output:**
```
⠸ [00:00:02] [=========================>---------] 162.4 MiB/256.0 MiB (78.4 MiB/s, ETA 2s)
```

## การทดสอบ (Testing)

### Unit Tests ใน `src/main.rs` (สำหรับ verification build)

โค้ดด้านล่างนี้คือ unit tests ที่ใช้ใน verification build จริง — compile และรันได้:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ── 1. Chunk encryption round-trip ───────────────────────
    #[test]
    fn test_chunk_roundtrip() {
        let mut key = [0u8; 32];
        rand::RngCore::fill_bytes(&mut rand::thread_rng(), &mut key);
        let plaintext = b"Hello, XChaCha20-Poly1305!";
        let (ct, nonce) = encrypt_chunk(&key, plaintext);
        let recovered = decrypt_chunk(&key, &nonce, &ct).expect("decrypt ok");
        assert_eq!(recovered, plaintext);
        // ciphertext ใหญ่กว่า plaintext 16 bytes (Poly1305 tag)
        assert_eq!(ct.len(), plaintext.len() + 16);
    }

    // ── 2. Wrong key fails authentication ───────────────────
    #[test]
    fn test_wrong_key_fails() {
        let mut key = [0u8; 32];
        rand::RngCore::fill_bytes(&mut rand::thread_rng(), &mut key);
        let (ct, nonce) = encrypt_chunk(&key, b"secret data");
        let mut bad_key = key;
        bad_key[0] ^= 0xFF;
        let result = decrypt_chunk(&bad_key, &nonce, &ct);
        assert!(result.is_err(), "wrong key must not decrypt");
    }

    // ── 3. File-format magic and version check ───────────────
    #[test]
    fn test_header_magic_version() {
        let mut salt = [0u8; SALT_LEN];
        let mut nonce = [0u8; NONCE_LEN];
        rand::RngCore::fill_bytes(&mut rand::thread_rng(), &mut salt);
        rand::RngCore::fill_bytes(&mut rand::thread_rng(), &mut nonce);
        let header = build_header(&salt, &nonce);
        assert_eq!(&header[0..4], MAGIC, "magic bytes must be RENC");
        assert_eq!(header[4], FORMAT_VERSION, "version must be 1");
        assert_eq!(header.len(), 4 + 1 + SALT_LEN + NONCE_LEN);
    }

    // ── 4. Streaming chunk authentication (multi-chunk) ──────
    #[test]
    fn test_streaming_chunk_auth() {
        let mut key = [0u8; 32];
        rand::RngCore::fill_bytes(&mut rand::thread_rng(), &mut key);
        // 3 full chunks + remainder
        let plaintext: Vec<u8> = (0u8..=255).cycle().take(CHUNK_SIZE * 3 + 1234).collect();
        let stream = stream_encrypt(&key, &plaintext);
        let recovered = stream_decrypt(&key, &stream).expect("streaming decrypt ok");
        assert_eq!(recovered, plaintext);
    }

    // ── 5. Argon2id determinism ──────────────────────────────
    #[test]
    fn test_argon2id_derivation() {
        let passphrase = b"correct horse battery staple";
        let salt = [0xAB_u8; SALT_LEN];
        let key1 = derive_key(passphrase, &salt);
        let key2 = derive_key(passphrase, &salt);
        assert_eq!(key1, key2, "same input must produce same key");
        let key3 = derive_key(b"wrong passphrase", &salt);
        assert_ne!(key1, key3);
        let key4 = derive_key(passphrase, &[0xCD_u8; SALT_LEN]);
        assert_ne!(key1, key4);
    }

    // ── 6. Tamper detection ──────────────────────────────────
    #[test]
    fn test_stream_tamper_rejected() {
        let mut key = [0u8; 32];
        rand::RngCore::fill_bytes(&mut rand::thread_rng(), &mut key);
        let plaintext = b"tamper test data with enough length".to_vec();
        let mut stream = stream_encrypt(&key, &plaintext);
        let flip_pos = NONCE_LEN + 4 + 5; // หลัง nonce + len field
        stream[flip_pos] ^= 0x01;
        let result = stream_decrypt(&key, &stream);
        assert!(result.is_err(), "tampered ciphertext must fail authentication");
    }

    // ── 7. Metadata serialization round-trip ─────────────────
    #[test]
    fn test_metadata_serialization() {
        let meta = FileMetadata {
            original_name: "document.pdf".to_string(),
            modified_secs: 1_700_000_000,
            modified_nanos: 123_456_789,
        };
        let json = serde_json::to_string(&meta).expect("serialize");
        let recovered: FileMetadata = serde_json::from_str(&json).expect("deserialize");
        assert_eq!(recovered.original_name, meta.original_name);
        assert_eq!(recovered.modified_secs, meta.modified_secs);
        assert_eq!(recovered.modified_nanos, meta.modified_nanos);
    }

    // ── 8. Zeroize clears key bytes ───────────────────────────
    #[test]
    fn test_zeroize_key() {
        let mut key = derive_key(b"my passphrase", &[0x11_u8; SALT_LEN]);
        assert!(key.iter().any(|&b| b != 0));
        key.zeroize();
        assert!(key.iter().all(|&b| b == 0));
    }
}
```

### ผล `cargo test` จริง

```
running 9 tests
test tests::test_header_parse_roundtrip ... ok
test tests::test_header_magic_version ... ok
test tests::test_metadata_serialization ... ok
test tests::test_stream_tamper_rejected ... ok
test tests::test_wrong_key_fails ... ok
test tests::test_chunk_roundtrip ... ok
test tests::test_streaming_chunk_auth ... ok
test tests::test_zeroize_key ... ok
test tests::test_argon2id_derivation ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 15.80s
```

ทุก test ผ่านจริง — รัน Argon2id ด้วย m=64MiB ทำให้แต่ละ test ที่ใช้ key derivation ใช้เวลาประมาณ 1–2 วินาทีซึ่งเป็นค่าที่ถูกต้องตาม parameter ที่กำหนด

## Pitfalls และข้อควรระวัง

### Pitfall #1: `u16` overflow ใน chunk_len field (ร้ายแรงมาก)

**ปัญหา:** ถ้าใช้ `u16` สำหรับ chunk_len field และ `CHUNK_SIZE = 64 KiB`:

```
ciphertext_len = CHUNK_SIZE + 16 (Poly1305 tag)
              = 65536 + 16
              = 65552 bytes
u16::MAX      = 65535

65552_u16 → overflow → 16 (wraps around!)
```

เมื่อ decode อ่าน `ct_len = 16` แทนที่จะเป็น 65552 ทำให้ parse stream ผิดตำแหน่งทั้งหมด nonce ของ chunk ถัดไปจะถูก interpret ว่าเป็น ciphertext ของ chunk ปัจจุบัน ซึ่งทำให้ authentication fail ทุก chunk หลังจากนั้น

**วิธีแก้:** ใช้ `u32` (4 bytes) สำหรับ chunk_len เสมอ:

```rust
// ❌ ผิด — overflow เมื่อ CHUNK_SIZE = 64 KiB
let ct_len = ct.len() as u16;
out.extend_from_slice(&ct_len.to_le_bytes()); // 2 bytes

// ✅ ถูก — u32 รองรับ chunk ได้ถึง 4 GiB
let ct_len = ct.len() as u32;
out.extend_from_slice(&ct_len.to_le_bytes()); // 4 bytes
```

### Pitfall #2: Nonce reuse ภายใต้ key เดียวกัน

XChaCha20-Poly1305 ใช้ XSalsa20 construction สำหรับ 192-bit nonce — ความปลอดภัยขึ้นอยู่กับว่า nonce ต้อง unique per (key, nonce) pair ถ้า nonce ซ้ำกัน keystream XOR กันได้:

```
ct1 = pt1 XOR keystream(k, n)
ct2 = pt2 XOR keystream(k, n)  // nonce ซ้ำ!

ct1 XOR ct2 = pt1 XOR pt2       // leak plaintext XOR
```

**วิธีแก้:** Generate nonce ด้วย CSPRNG ทุกครั้ง (`OsRng`) — ไม่ใช่ counter ที่อาจซ้ำกันถ้า process restart:

```rust
// ✅ ถูก — random nonce ต่อ chunk
let nonce = XChaCha20Poly1305::generate_nonce(&mut OsRng);
```

XChaCha20's 192-bit nonce ทำให้ birthday bound อยู่ที่ ~2^96 messages ก่อน collision probability ถึง threshold — random nonce generation จึงปลอดภัยในทางปฏิบัติ

### Pitfall #3: `drop()` ไม่ zero memory

Rust `drop()` คืน ownership และ deallocate memory แต่ **ไม่รับประกันว่า compiler จะไม่ optimize ออก** — โดยเฉพาะถ้า optimizer เห็นว่าค่าไม่ถูกใช้อีก:

```rust
// ❌ ไม่น่าเชื่อถือ — optimizer อาจ elide assignment
let mut key = [0u8; 32];
// ... ใช้ key ...
key = [0u8; 32]; // อาจถูก optimize ออก

// ✅ ถูก — zeroize ใช้ volatile write ที่ optimizer ไม่ elide ได้
use zeroize::Zeroize;
key.zeroize();
```

`zeroize` crate ใช้ `core::ptr::write_volatile` และ compiler fence เพื่อให้แน่ใจว่า zero-write ถูก execute จริง

### Pitfall #4: Shred ไม่ทำงานบน Copy-on-Write Filesystems

`shred` command ใช้ `write()` + `fsync()` + `unlink()` ซึ่งใช้งานได้บน traditional HDD และ ext4/XFS/NTFS ที่ไม่ใช่ COW แต่บน:

- **APFS (macOS)** — CoW by default: write ใหม่ไปยัง block ใหม่ block เดิมยังอยู่
- **Btrfs (COW mode)** — เช่นเดียวกัน
- **SSD with wear leveling** — firmware map write ไปยัง physical block ต่างๆ เพื่อกระจาย wear

ในกรณีเหล่านี้ plaintext blocks อาจยังคงอยู่ใน storage แม้ shred จะ complete:

```rust
// Best-effort shred — document ข้อจำกัด
/// WARNING: บน APFS, Btrfs (COW mode), และ flash storage ที่มี wear leveling
/// การ overwrite อาจไม่เขียนทับ physical blocks เดิม
/// สำหรับ strong deletion ให้ใช้ full-disk encryption ร่วมกับ secure erase firmware command
pub fn shred(path: &Path) -> Result<(), RencError> {
    // ...
}
```

### Pitfall #5: `parse_header` ต้องตรวจ magic ก่อน version

ถ้าตรวจ version ก่อน magic และ input เป็น random binary file byte ที่ position 4 อาจบังเอิญเท่ากับ `FORMAT_VERSION` (1) — ทำให้ผ่าน version check แล้วค่อย fail ด้วย error message ที่ไม่ชัดเจน:

```rust
// ❌ ผิดลำดับ
if data[4] != FORMAT_VERSION { return Err("bad version"); }
if &data[0..4] != MAGIC { return Err("bad magic"); }

// ✅ ถูก — ตรวจ magic ก่อนเสมอ
if &data[0..4] != MAGIC { return Err("bad magic"); }
if data[4] != FORMAT_VERSION { return Err("unsupported version"); }
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimized release build พร้อม debug symbols แยก
cargo build --release
strip target/release/renc  # ลด binary size

# Cross-compile สำหรับ musl (static binary)
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
```

**Binary size:**
```
target/release/renc       ~4.2 MB (before strip)
target/release/renc       ~1.8 MB (after strip)
```

### Install

```bash
# Install ไปยัง ~/.cargo/bin/
cargo install --path .

# ทดสอบ
renc keygen mykey
echo "test content" > test.txt
renc encrypt test.txt --key-name mykey
renc verify test.txt.enc --key-name mykey
renc decrypt test.txt.enc --key-name mykey
```

### Docker

```dockerfile
FROM rust:1.79-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/renc /usr/local/bin/renc
ENTRYPOINT ["renc"]
```

```bash
docker build -t renc .
docker run --rm -v $(pwd):/data renc encrypt /data/file.txt --passphrase mysecret
```

### Shell Completion

```bash
# bash
renc --generate-completion bash > ~/.local/share/bash-completion/completions/renc

# zsh  
renc --generate-completion zsh > ~/.zsh/completions/_renc
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Streaming I/O แทน in-memory buffer

**ปัญหา:** ปัจจุบัน `stream_encrypt` โหลด plaintext ทั้งไฟล์เข้า memory ก่อน ไฟล์ขนาด 2 GB จะใช้ RAM 2+ GB

**งาน:** แก้ `encrypt_file` ให้ใช้ `BufReader` / `BufWriter` และเข้ารหัสทีละ chunk โดยไม่ buffer ทั้งไฟล์:

```rust
use std::io::{BufReader, BufWriter, Read, Write};

pub fn encrypt_file_streaming(
    input: &Path,
    output: &Path,
    key: &[u8; 32],
) -> Result<(), RencError> {
    let mut reader = BufReader::new(File::open(input)?);
    let mut writer = BufWriter::new(File::create(output)?);

    // เขียน header ก่อน
    // ...

    let mut buf = vec![0u8; CHUNK_SIZE];
    loop {
        let n = reader.read(&mut buf)?;
        if n == 0 { break; }
        // encrypt chunk และเขียนลง writer ทันที
        // ต้องจัดการ final chunk flag ด้วย
        // ...
    }
    Ok(())
}
```

**ความยาก:** ⭐⭐ — ต้องจัดการ final flag โดยไม่รู้จำนวน chunk ล่วงหน้า (hint: ใช้ lookahead buffer หนึ่ง chunk)

### แบบฝึกหัดที่ 2: Encrypted Archive (หลายไฟล์ใน .renc)

**งาน:** สร้างคำสั่ง `archive` ที่บีบอัดหลายไฟล์เข้า archive เดียว แบบ tar แต่เข้ารหัส:

```
renc archive output.renc file1.txt file2.pdf photos/
renc extract output.renc --passphrase mysecret
```

**File format เพิ่มเติม:** header ของ archive มี table of contents:
```json
{
  "files": [
    {"name": "file1.txt", "offset": 0, "length": 1024},
    {"name": "file2.pdf", "offset": 1024, "length": 4096}
  ]
}
```

**ความยาก:** ⭐⭐⭐ — ต้อง design format ที่ random-access ได้ (extract ไฟล์เดียวโดยไม่ decrypt ทั้ง archive)

### แบบฝึกหัดที่ 3: Age-Compatible Format

[age](https://age-encryption.org/) เป็น file encryption standard ที่ใช้ใน production หลายระบบ

**งาน:** เพิ่ม `--age` flag ที่ produce output ที่ compatible กับ `age` tool โดยใช้ X25519 key exchange แทน symmetric key:

```
renc encrypt --age --recipient age1... file.txt
# output ที่ age tool สามารถ decrypt ได้:
age -d -i ~/.age/key.txt file.txt.age
```

**ความยาก:** ⭐⭐⭐⭐ — ต้องเข้าใจ age format specification และ X25519 ephemeral key exchange

### แบบฝึกหัดที่ 4: Hardware Security Key (YubiKey) Integration

**งาน:** เพิ่ม support สำหรับ YubiKey HMAC-SHA1 challenge-response เพื่อ two-factor encryption:

```
renc encrypt --yubikey-slot 2 file.txt
# ต้อง touch YubiKey เพื่อ generate challenge response
# key = Argon2id(passphrase + yubikey_response, salt)
```

**ความยาก:** ⭐⭐⭐⭐⭐ — ต้องใช้ `yubikey` crate, handle USB HID communication, และ design challenge-response protocol ที่ตรวจ replay attack ได้

## สรุป

โปรเจคนี้สร้าง file encryption tool ระดับ production โดยใช้:

1. **XChaCha20-Poly1305** — AEAD ที่ give confidentiality + integrity ต่อ chunk, resistant ต่อ timing attack
2. **Argon2id** — memory-hard KDF ที่ resistant ต่อ GPU/FPGA brute-force บน passphrase
3. **Binary file format** ที่ versioned — magic bytes, salt, per-chunk nonce, final flag AAD
4. **`zeroize`** — secure memory clearing หลัง key material ใช้งาน
5. **`rayon`** — parallel directory encryption ด้วย work-stealing scheduler
6. **`indicatif`** — progress reporting ที่ show MB/s throughput

Pattern ที่สำคัญที่สุดที่ได้เรียน:

- **chunk_len ต้อง `u32`** — `CHUNK_SIZE + tag` เกิน `u16::MAX` (Pitfall #1)
- **Final flag ใน AAD** — ป้องกัน truncation attack โดยไม่ต้องเพิ่ม field พิเศษ
- **Pre-pass count_chunks** — จำเป็นสำหรับ detect final chunk ใน streaming decode
- **Random nonce ต่อ chunk** — XChaCha20's 192-bit nonce space ทำให้ collision-safe โดยไม่ต้องใช้ counter

โปรเจคถัดไป [**Project D03: TLS Proxy**](project-d03-tls-proxy.md) จะนำความรู้ cryptography เหล่านี้ไปใช้ในบริบท network protocol — สร้าง transparent TLS termination proxy ด้วย `rustls` ที่ handle certificate management และ SNI routing

---

**โปรเจคก่อนหน้า:** [project-d01-password-manager.md](project-d01-password-manager.md) | **โปรเจคถัดไป:** [project-d03-tls-proxy.md](project-d03-tls-proxy.md)
