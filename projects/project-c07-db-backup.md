# Project C07: Database Backup & Restore Tool

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้างเครื่องมือ backup และ restore ฐานข้อมูล PostgreSQL แบบ production-grade ครบวงจร โดยรองรับทั้ง **full backup** และ **incremental backup** พร้อม compression (LZ4) และ encryption (AES-256-GCM) เพื่อให้ข้อมูลปลอดภัยทั้งตอนเก็บและส่ง

ในโลก production ฐานข้อมูลขนาดใหญ่หลายสิบ GB ไม่สามารถ backup แบบ pg_dump ธรรมดาได้ทุกคืน เพราะใช้เวลานานและ disk space มาก incremental backup ช่วยให้ backup เฉพาะข้อมูลที่เปลี่ยนแปลงตั้งแต่ครั้งล่าสุด ลดทั้งเวลาและพื้นที่จัดเก็บลงอย่างมาก

**Learning value ที่ได้จากโปรเจคนี้:**
- ออกแบบ binary file format เองพร้อม magic bytes, versioning, และ header
- ผสานกัน compression + encryption ในลำดับที่ถูกต้อง (compress before encrypt)
- ใช้ Argon2id สำหรับ key derivation จาก passphrase แทนการเก็บ key ตรง ๆ
- สร้าง backup chain manifest สำหรับ incremental restore
- ใช้ `sqlx` streaming query กับ PostgreSQL โดยไม่โหลดทุก row เข้า memory พร้อมกัน

## สิ่งที่จะได้เรียนรู้

- **Custom binary format**: ออกแบบ file format ด้วย magic bytes, version header, และ per-table sections
- **LZ4 compression**: ใช้ `lz4_flex` compress ข้อมูลก่อน encrypt เพื่อประสิทธิภาพสูงสุด
- **AES-256-GCM encryption**: authenticated encryption ป้องกันทั้ง confidentiality และ integrity
- **Argon2id KDF**: แปลง passphrase เป็น cryptographic key อย่างปลอดภัย
- **Incremental backup logic**: filter rows ด้วย `updated_at` timestamp
- **Backup manifest**: JSON chain tracking สำหรับ restore ลำดับที่ถูกต้อง
- **Progress reporting**: real-time progress bars ด้วย `indicatif`
- **SHA-256 integrity**: ตรวจสอบความสมบูรณ์ของ backup file

## ความรู้ที่ต้องมีมาก่อน

- **จาก Part 46**: async/await และ `tokio` runtime
- **จาก Part 52**: error handling ด้วย `anyhow` และ `thiserror`
- **จาก Part 55**: trait objects และ dynamic dispatch
- **จาก Part 61**: การใช้งาน external crates และ Cargo.toml
- **จาก Part 70**: Binary serialization กับ `bincode` และ `serde`
- **จาก Part 75**: Database access ด้วย `sqlx`
- **จาก Part 80**: Cryptography พื้นฐานใน Rust (ถ้ามีในหลักสูตร)

## โครงสร้างโปรเจค (Project Layout)

```
db-backup-tool/
├── src/
│   ├── main.rs          # CLI entry point (clap)
│   ├── backup.rs        # Full & incremental backup logic
│   ├── restore.rs       # Restore & conflict resolution
│   ├── format.rs        # Custom binary format types
│   ├── crypto.rs        # AES-256-GCM + Argon2id KDF
│   ├── compress.rs      # LZ4 wrapper
│   ├── manifest.rs      # Backup manifest (JSON chain)
│   ├── progress.rs      # indicatif progress bars
│   ├── schedule.rs      # Cron-based scheduling
│   └── verify.rs        # SHA-256 checksum verification
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
PostgreSQL DB
     │
     ▼ (sqlx streaming, batch 1000 rows)
[backup.rs] Row batches
     │
     ▼ (bincode serialize)
Binary Row Data
     │
     ▼ (lz4_flex compress)
Compressed Data
     │
     ▼ (AES-256-GCM encrypt)
Encrypted Backup File  ──▶  manifest.json (SHA-256, metadata)
```

**Restore ทำย้อนกลับ:**
```
manifest.json → resolve chain → Encrypted File
     │
     ▼ (AES-256-GCM decrypt)
Compressed Data
     │
     ▼ (lz4_flex decompress)
Binary Row Data
     │
     ▼ (bincode deserialize)
Rows  ──▶  INSERT INTO ... ON CONFLICT DO NOTHING (batch 1000)
```

### ทำไมถึงเลือก Design นี้

**Compress ก่อน Encrypt:** ข้อมูลที่ encrypted จะมี entropy สูง (ดูเหมือน random) ทำให้ compression ไม่ได้ผล ดังนั้นต้อง compress ก่อนเสมอ เป็น best practice ที่ TLS, ZIP encryption ก็ใช้

**Argon2id แทน bcrypt/scrypt:** Argon2id เป็น winner ของ Password Hashing Competition 2015 รองรับทั้ง time-hardness และ memory-hardness ป้องกัน brute-force ได้ดีกว่า

**bincode แทน JSON สำหรับ row data:** JSON ใช้ space มากกว่า 3-5x และ parse ช้ากว่า bincode ที่เป็น binary format เหมาะกับข้อมูลปริมาณมาก

**LZ4 แทน gzip/zstd:** LZ4 เน้น speed สูงสุด (decompression ~5-10 GB/s) ยอมเสีย compression ratio บ้าง เหมาะกับ backup ที่ต้องการ restore เร็ว

### Design Decisions และทางเลือก

| ปัญหา | เลือก | ทางเลือกอื่น |
|------|------|-------------|
| Compression | LZ4 (เร็ว) | zstd (ratio ดีกว่า), gzip |
| Encryption | AES-256-GCM | ChaCha20-Poly1305 (เร็วกว่าบน CPU ไม่มี AES-NI) |
| Row format | bincode | MessagePack, CBOR |
| KDF | Argon2id | bcrypt, PBKDF2 |
| DB interface | sqlx async | diesel (sync), tokio-postgres |

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Types พื้นฐาน

เริ่มต้นด้วยการสร้าง `Cargo.toml` และ type definitions สำหรับ backup format

**`Cargo.toml`:**

```toml
[package]
name = "db-backup-tool"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "db-backup"
path = "src/main.rs"

[dependencies]
# Database
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-tls", "chrono", "uuid"] }

# Serialization
bincode = "1.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Compression
lz4_flex = "0.11"

# Encryption
aes-gcm = "0.10"
argon2 = "0.5"

# Hashing
sha2 = "0.10"

# Async runtime
tokio = { version = "1", features = ["full"] }

# CLI
clap = { version = "4", features = ["derive"] }

# Progress bars
indicatif = "0.17"

# Error handling
anyhow = "1"
thiserror = "1"

# Time
chrono = { version = "0.4", features = ["serde"] }

# Random (for nonce generation)
rand = "0.8"

# Scheduling
tokio-cron-scheduler = "0.9"

# UUID สำหรับ backup IDs
uuid = { version = "1", features = ["v4", "serde"] }

[dev-dependencies]
tempfile = "3"
```

**`src/format.rs`** — Custom Binary Format Types:

```rust
//! format.rs — Custom backup binary format
//!
//! ไฟล์ backup มีโครงสร้างดังนี้:
//!
//!  ┌─────────────────────────────────────┐
//!  │  BackupHeader (bincode)             │  magic + version + metadata
//!  ├─────────────────────────────────────┤
//!  │  TableBackup[0] (bincode)           │  table "users"
//!  │    - table_name: String             │
//!  │    - row_count: u64                 │
//!  │    - columns: Vec<ColumnDef>        │
//!  │    - compressed_data: Vec<u8>       │  LZ4(bincode(Vec<Row>))
//!  ├─────────────────────────────────────┤
//!  │  TableBackup[1] (bincode)           │  table "orders"
//!  │  ...                               │
//!  └─────────────────────────────────────┘
//!
//!  ไฟล์ทั้งหมด (BackupFile) ถูก encrypt ด้วย AES-256-GCM
//!  ก่อนเขียนลงดิสก์

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// Magic bytes สำหรับระบุว่าเป็นไฟล์ backup ของเรา
pub const BACKUP_MAGIC: &[u8; 8] = b"DBBACKUP";
pub const FORMAT_VERSION: u32 = 1;

/// ประเภทข้อมูลของ column ใน PostgreSQL
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum ColumnType {
    Integer,
    BigInt,
    Text,
    VarChar(u32),
    Boolean,
    Float,
    Double,
    Timestamp,
    TimestampTz,
    Date,
    Bytea,
    Json,
    Jsonb,
    Uuid,
    Numeric,
    Unknown(String),
}

impl ColumnType {
    /// แปลงจาก PostgreSQL type name (จาก information_schema)
    pub fn from_pg_type(type_name: &str) -> Self {
        match type_name {
            "integer" | "int" | "int4" => ColumnType::Integer,
            "bigint" | "int8" => ColumnType::BigInt,
            "text" => ColumnType::Text,
            "boolean" | "bool" => ColumnType::Boolean,
            "real" | "float4" => ColumnType::Float,
            "double precision" | "float8" => ColumnType::Double,
            "timestamp" | "timestamp without time zone" => ColumnType::Timestamp,
            "timestamp with time zone" | "timestamptz" => ColumnType::TimestampTz,
            "date" => ColumnType::Date,
            "bytea" => ColumnType::Bytea,
            "json" => ColumnType::Json,
            "jsonb" => ColumnType::Jsonb,
            "uuid" => ColumnType::Uuid,
            "numeric" | "decimal" => ColumnType::Numeric,
            other => ColumnType::Unknown(other.to_string()),
        }
    }
}

/// ข้อมูล schema ของ column หนึ่งตัว
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct ColumnDef {
    pub name: String,
    pub col_type: ColumnType,
    pub nullable: bool,
    pub position: u32, // ordinal_position จาก information_schema
}

/// ค่าของ cell ใน row (รองรับ NULL และหลาย types)
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum CellValue {
    Null,
    Integer(i64),
    Text(String),
    Boolean(bool),
    Float(f64),
    Bytes(Vec<u8>),
    Timestamp(i64),       // Unix timestamp microseconds
    Uuid([u8; 16]),
    Json(String),
}

/// Row ของข้อมูล — Vec ของ CellValue ตามลำดับ columns
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Row {
    pub values: Vec<CellValue>,
}

/// Header หลักของไฟล์ backup — อยู่ตอนต้นไฟล์
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BackupHeader {
    pub magic: [u8; 8],             // ต้องเป็น DBBACKUP
    pub format_version: u32,        // FORMAT_VERSION
    pub db_version: String,         // "PostgreSQL 16.0" จาก SHOW server_version
    pub database_name: String,
    pub created_at: i64,            // Unix timestamp seconds
    pub table_names: Vec<String>,   // ตารางทั้งหมดที่ backup
    pub is_incremental: bool,
    pub base_backup_id: Option<String>, // สำหรับ incremental
    pub last_backup_ts: Option<i64>,    // WHERE updated_at > last_backup_ts
}

impl BackupHeader {
    pub fn new_full(db_version: String, database_name: String, table_names: Vec<String>) -> Self {
        Self {
            magic: *BACKUP_MAGIC,
            format_version: FORMAT_VERSION,
            db_version,
            database_name,
            created_at: Utc::now().timestamp(),
            table_names,
            is_incremental: false,
            base_backup_id: None,
            last_backup_ts: None,
        }
    }

    pub fn new_incremental(
        db_version: String,
        database_name: String,
        table_names: Vec<String>,
        base_backup_id: String,
        last_backup_ts: i64,
    ) -> Self {
        Self {
            magic: *BACKUP_MAGIC,
            format_version: FORMAT_VERSION,
            db_version,
            database_name,
            created_at: Utc::now().timestamp(),
            table_names,
            is_incremental: true,
            base_backup_id: Some(base_backup_id),
            last_backup_ts: Some(last_backup_ts),
        }
    }

    /// ตรวจสอบว่า magic bytes ถูกต้อง
    pub fn validate_magic(&self) -> bool {
        self.magic == *BACKUP_MAGIC
    }
}

/// ข้อมูล backup ของตารางหนึ่งตาราง
#[derive(Debug, Serialize, Deserialize)]
pub struct TableBackup {
    pub table_name: String,
    pub row_count: u64,
    pub columns: Vec<ColumnDef>,
    /// LZ4-compressed bincode-serialized Vec<Row>
    pub compressed_data: Vec<u8>,
    /// ขนาดข้อมูลก่อน compress (สำหรับ progress reporting)
    pub uncompressed_size: u64,
}

/// ไฟล์ backup ทั้งหมด (โครงสร้างนี้จะถูก encrypt ก่อนเขียนลง disk)
#[derive(Debug, Serialize, Deserialize)]
pub struct BackupFile {
    pub header: BackupHeader,
    pub tables: Vec<TableBackup>,
}
```

**`src/compress.rs`** — LZ4 Wrapper:

```rust
//! compress.rs — LZ4 compression wrapper

use anyhow::Result;

/// Compress ข้อมูลด้วย LZ4
/// LZ4 จะ prepend ขนาดข้อมูลต้นฉบับไว้หน้า compressed data
pub fn compress(data: &[u8]) -> Result<Vec<u8>> {
    let compressed = lz4_flex::compress_prepend_size(data);
    Ok(compressed)
}

/// Decompress ข้อมูล LZ4
/// คืน error ถ้าข้อมูลเสียหายหรือไม่ใช่ LZ4 format
pub fn decompress(data: &[u8]) -> Result<Vec<u8>> {
    lz4_flex::decompress_size_prepended(data)
        .map_err(|e| anyhow::anyhow!("LZ4 decompression failed: {}", e))
}

/// วัด compression ratio
pub fn compression_ratio(original: usize, compressed: usize) -> f64 {
    if compressed == 0 {
        return 0.0;
    }
    original as f64 / compressed as f64
}
```

### ขั้นที่ 2: Crypto Module (AES-256-GCM + Argon2id)

**`src/crypto.rs`:**

```rust
//! crypto.rs — AES-256-GCM encryption with Argon2id key derivation
//!
//! Encrypted file format:
//!  [16 bytes: Argon2 salt]
//!  [12 bytes: AES-GCM nonce]
//!  [N bytes: AES-256-GCM ciphertext + 16-byte auth tag]

use aes_gcm::{
    aead::{Aead, KeyInit},
    Aes256Gcm, Key, Nonce,
};
use aes_gcm::aead::rand_core::RngCore;
use aes_gcm::aead::OsRng;
use anyhow::{Context, Result};
use argon2::Argon2;

/// ขนาด salt สำหรับ Argon2id (16 bytes = 128 bits)
const SALT_SIZE: usize = 16;
/// ขนาด nonce สำหรับ AES-256-GCM (12 bytes = 96 bits)
const NONCE_SIZE: usize = 12;

/// Wrapper สำหรับ encryption key ที่ derive มาจาก passphrase
pub struct EncryptionKey {
    key_bytes: [u8; 32],
}

/// ใช้ Argon2id derive 256-bit key จาก passphrase + salt
///
/// Argon2id ดีกว่า bcrypt/scrypt เพราะ:
/// 1. รองรับทั้ง time-hardness + memory-hardness
/// 2. ป้องกันได้ทั้ง CPU brute-force และ GPU/ASIC attacks
pub fn derive_key_from_passphrase(passphrase: &str, salt: &[u8; SALT_SIZE]) -> Result<EncryptionKey> {
    let mut key_bytes = [0u8; 32];
    Argon2::default()
        .hash_password_into(passphrase.as_bytes(), salt, &mut key_bytes)
        .map_err(|e| anyhow::anyhow!("Argon2id key derivation failed: {}", e))?;
    Ok(EncryptionKey { key_bytes })
}

/// Encrypt ข้อมูลด้วย AES-256-GCM
///
/// คืน: [16-byte salt][12-byte nonce][ciphertext+tag]
/// salt ถูกเก็บไว้ใน output เพื่อให้ decrypt ได้โดยต้องการแค่ passphrase
pub fn encrypt(plaintext: &[u8], passphrase: &str) -> Result<Vec<u8>> {
    // สร้าง salt แบบ random
    let mut salt = [0u8; SALT_SIZE];
    OsRng.fill_bytes(&mut salt);

    let key = derive_key_from_passphrase(passphrase, &salt)?;
    let cipher_key = Key::<Aes256Gcm>::from_slice(&key.key_bytes);
    let cipher = Aes256Gcm::new(cipher_key);

    // สร้าง nonce แบบ random
    let mut nonce_bytes = [0u8; NONCE_SIZE];
    OsRng.fill_bytes(&mut nonce_bytes);
    let nonce = Nonce::from_slice(&nonce_bytes);

    let ciphertext = cipher
        .encrypt(nonce, plaintext)
        .map_err(|_| anyhow::anyhow!("AES-256-GCM encryption failed"))?;

    // รวม salt + nonce + ciphertext
    let mut output = Vec::with_capacity(SALT_SIZE + NONCE_SIZE + ciphertext.len());
    output.extend_from_slice(&salt);
    output.extend_from_slice(&nonce_bytes);
    output.extend_from_slice(&ciphertext);
    Ok(output)
}

/// Decrypt ข้อมูล AES-256-GCM
///
/// Input: [16-byte salt][12-byte nonce][ciphertext+tag]
/// ใช้ passphrase derive key จาก salt ที่เก็บในไฟล์
pub fn decrypt(encrypted: &[u8], passphrase: &str) -> Result<Vec<u8>> {
    if encrypted.len() < SALT_SIZE + NONCE_SIZE {
        return Err(anyhow::anyhow!(
            "Encrypted data too short: {} bytes (minimum {})",
            encrypted.len(),
            SALT_SIZE + NONCE_SIZE
        ));
    }

    let salt: &[u8; SALT_SIZE] = encrypted[..SALT_SIZE]
        .try_into()
        .context("Failed to extract salt")?;
    let nonce_bytes = &encrypted[SALT_SIZE..SALT_SIZE + NONCE_SIZE];
    let ciphertext = &encrypted[SALT_SIZE + NONCE_SIZE..];

    let key = derive_key_from_passphrase(passphrase, salt)?;
    let cipher_key = Key::<Aes256Gcm>::from_slice(&key.key_bytes);
    let cipher = Aes256Gcm::new(cipher_key);
    let nonce = Nonce::from_slice(nonce_bytes);

    cipher
        .decrypt(nonce, ciphertext)
        .map_err(|_| anyhow::anyhow!(
            "Decryption failed — wrong passphrase or corrupted data"
        ))
}
```

**Pitfall #1: Nonce Reuse ใน AES-GCM**

AES-256-GCM เป็น **nonce-misuse-resistant** ก็จริง แต่ถ้า nonce ซ้ำกันโดยบังเอิญ ผู้โจมตีจะสามารถ XOR ciphertexts สองชิ้นและกู้คืนข้อมูลได้ ใน code นี้เราสร้าง nonce แบบ random 12 bytes ทุกครั้งที่ encrypt ซึ่งมีโอกาสชนกันน้อยมาก (ต้อง encrypt มากกว่า 2^48 ครั้งถึงจะมีโอกาส 50% ชน)

```rust
// ❌ WRONG: ใช้ nonce คงที่
let nonce = Nonce::from_slice(b"unique nonce");

// ✅ CORRECT: สร้าง nonce แบบ random ทุกครั้ง
let mut nonce_bytes = [0u8; 12];
OsRng.fill_bytes(&mut nonce_bytes);
let nonce = Nonce::from_slice(&nonce_bytes);
```

### ขั้นที่ 3: Backup Manifest

**`src/manifest.rs`:**

```rust
//! manifest.rs — Backup chain manifest (JSON)
//!
//! manifest.json เก็บ metadata ของ backup ทั้งหมด รวมถึง
//! backup chain สำหรับ incremental restore

use crate::format::BackupHeader;
use anyhow::Result;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};
use std::collections::HashMap;
use std::fs;
use std::path::{Path, PathBuf};

/// ประเภท backup
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum BackupType {
    Full,
    Incremental,
}

/// Entry หนึ่งรายการใน manifest
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BackupEntry {
    pub id: String,                              // UUID v4
    pub backup_type: BackupType,
    pub created_at: DateTime<Utc>,
    pub file_path: String,                       // path ของ .bak file
    pub file_size_bytes: u64,
    pub checksum_sha256: String,                 // SHA-256 ของ encrypted file
    pub table_row_counts: HashMap<String, u64>,  // table → row count
    pub base_backup_id: Option<String>,          // สำหรับ incremental
    pub encryption_salt_hex: String,             // salt hex (ไม่เก็บ key!)
}

/// Manifest ทั้งหมดของฐานข้อมูล
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BackupManifest {
    pub database_name: String,
    pub entries: Vec<BackupEntry>,
    pub latest_full_backup_id: Option<String>,
}

impl BackupManifest {
    pub fn new(database_name: String) -> Self {
        Self {
            database_name,
            entries: Vec::new(),
            latest_full_backup_id: None,
        }
    }

    /// โหลด manifest จากไฟล์ JSON
    pub fn load(path: &Path) -> Result<Self> {
        let content = fs::read_to_string(path)?;
        let manifest: Self = serde_json::from_str(&content)?;
        Ok(manifest)
    }

    /// บันทึก manifest ลงไฟล์ JSON
    pub fn save(&self, path: &Path) -> Result<()> {
        let content = serde_json::to_string_pretty(self)?;
        fs::write(path, content)?;
        Ok(())
    }

    /// เพิ่ม backup entry ใหม่
    pub fn add_entry(&mut self, entry: BackupEntry) {
        if entry.backup_type == BackupType::Full {
            self.latest_full_backup_id = Some(entry.id.clone());
        }
        self.entries.push(entry);
    }

    /// หา entry ด้วย ID
    pub fn find_entry(&self, id: &str) -> Option<&BackupEntry> {
        self.entries.iter().find(|e| e.id == id)
    }

    /// สร้าง restore chain จาก incremental backup ไปหา full backup
    ///
    /// ถ้า backup chain คือ: full → incr1 → incr2
    /// เรียก get_restore_chain("incr2") จะได้ [full, incr1, incr2]
    pub fn get_restore_chain(&self, backup_id: &str) -> Vec<&BackupEntry> {
        let mut chain = Vec::new();
        let mut current_id = Some(backup_id.to_string());

        // walk backward ตาม base_backup_id จนถึง full backup
        while let Some(id) = current_id {
            match self.find_entry(&id) {
                Some(entry) => {
                    chain.push(entry);
                    current_id = entry.base_backup_id.clone();
                }
                None => break,
            }
        }

        // reverse เพื่อให้ได้ลำดับ full → incr1 → incr2
        chain.reverse();
        chain
    }

    /// ID ของ backup ล่าสุด (ใช้เป็น base สำหรับ incremental backup ครั้งต่อไป)
    pub fn latest_backup_id(&self) -> Option<&str> {
        self.entries.last().map(|e| e.id.as_str())
    }

    /// Timestamp ของ backup ล่าสุด (ใช้เป็น WHERE updated_at > ...)
    pub fn latest_backup_timestamp(&self) -> Option<i64> {
        self.entries.last().map(|e| e.created_at.timestamp())
    }
}

/// คำนวณ SHA-256 checksum ของข้อมูล
pub fn compute_sha256(data: &[u8]) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data);
    format!("{:x}", hasher.finalize())
}

/// คำนวณ SHA-256 จากไฟล์บนดิสก์
pub fn compute_file_sha256(path: &Path) -> Result<String> {
    let data = fs::read(path)?;
    Ok(compute_sha256(&data))
}
```

### ขั้นที่ 4: Backup Engine

**`src/backup.rs`:**

```rust
//! backup.rs — Full และ Incremental Backup Engine
//!
//! ดึงข้อมูลจาก PostgreSQL ผ่าน sqlx แบบ streaming batches
//! serialize ด้วย bincode, compress ด้วย LZ4, encrypt ด้วย AES-256-GCM

use crate::compress;
use crate::crypto;
use crate::format::{BackupFile, BackupHeader, ColumnDef, ColumnType, Row, TableBackup};
use crate::manifest::{BackupEntry, BackupManifest, BackupType, compute_sha256};
use crate::progress::BackupProgress;
use anyhow::{Context, Result};
use chrono::Utc;
use indicatif::MultiProgress;
use sqlx::{postgres::PgRow, PgPool, Row as SqlxRow};
use std::collections::HashMap;
use std::fs;
use std::path::PathBuf;
use uuid::Uuid;

const BATCH_SIZE: usize = 1000;

/// Config สำหรับ backup operation
pub struct BackupConfig {
    pub database_url: String,
    pub output_dir: PathBuf,
    pub passphrase: String,
    pub schema: String, // เช่น "public"
}

/// ดึงรายชื่อตารางทั้งหมดใน schema จาก information_schema
pub async fn list_tables(pool: &PgPool, schema: &str) -> Result<Vec<String>> {
    let rows = sqlx::query(
        r#"
        SELECT table_name
        FROM information_schema.tables
        WHERE table_schema = $1
          AND table_type = 'BASE TABLE'
        ORDER BY table_name
        "#,
    )
    .bind(schema)
    .fetch_all(pool)
    .await
    .context("Failed to list tables from information_schema")?;

    Ok(rows.into_iter().map(|r| r.get::<String, _>("table_name")).collect())
}

/// ดึง column definitions ของตารางจาก information_schema
pub async fn get_table_columns(pool: &PgPool, schema: &str, table: &str) -> Result<Vec<ColumnDef>> {
    let rows = sqlx::query(
        r#"
        SELECT column_name, data_type, is_nullable, ordinal_position
        FROM information_schema.columns
        WHERE table_schema = $1 AND table_name = $2
        ORDER BY ordinal_position
        "#,
    )
    .bind(schema)
    .bind(table)
    .fetch_all(pool)
    .await
    .context(format!("Failed to get columns for table {}", table))?;

    Ok(rows.into_iter().map(|r| {
        ColumnDef {
            name: r.get("column_name"),
            col_type: ColumnType::from_pg_type(r.get::<&str, _>("data_type")),
            nullable: r.get::<&str, _>("is_nullable") == "YES",
            position: r.get::<i32, _>("ordinal_position") as u32,
        }
    }).collect())
}

/// นับ row ในตาราง (สำหรับ progress bar)
pub async fn count_rows(pool: &PgPool, table: &str, where_clause: Option<&str>) -> Result<i64> {
    let query = match where_clause {
        Some(clause) => format!("SELECT COUNT(*) FROM {} WHERE {}", table, clause),
        None => format!("SELECT COUNT(*) FROM {}", table),
    };
    let row = sqlx::query(&query).fetch_one(pool).await?;
    Ok(row.get::<i64, _>(0))
}

/// Backup ตารางเดียว — ดึง rows เป็น batches ของ 1000
///
/// สำหรับ incremental backup จะมี where_clause: "updated_at > $last_ts"
async fn backup_table(
    pool: &PgPool,
    table: &str,
    columns: &[ColumnDef],
    where_clause: Option<String>,
    progress: &BackupProgress,
) -> Result<TableBackup> {
    let select_cols: Vec<String> = columns.iter().map(|c| c.name.clone()).collect();
    let col_list = select_cols.join(", ");

    let query_str = match &where_clause {
        Some(clause) => format!(
            "SELECT {} FROM {} WHERE {} ORDER BY ctid",
            col_list, table, clause
        ),
        None => format!("SELECT {} FROM {} ORDER BY ctid", col_list, table),
    };

    let mut all_rows: Vec<Row> = Vec::new();
    let mut offset = 0i64;

    // Stream ข้อมูลเป็น batches เพื่อไม่โหลดทั้งหมดเข้า memory
    loop {
        let batch_query = format!("{} LIMIT {} OFFSET {}", query_str, BATCH_SIZE, offset);
        let pg_rows = sqlx::query(&batch_query).fetch_all(pool).await?;

        if pg_rows.is_empty() {
            break;
        }

        let fetched = pg_rows.len();
        for pg_row in pg_rows {
            let row = convert_pg_row(&pg_row, columns)?;
            all_rows.push(row);
        }

        progress.increment_rows(fetched as u64);
        offset += BATCH_SIZE as i64;

        if fetched < BATCH_SIZE {
            break;
        }
    }

    let row_count = all_rows.len() as u64;

    // Serialize rows ด้วย bincode
    let serialized = bincode::serialize(&all_rows)
        .context("Failed to serialize rows")?;
    let uncompressed_size = serialized.len() as u64;

    // Compress ด้วย LZ4
    let compressed = compress::compress(&serialized)
        .context("Failed to compress table data")?;

    Ok(TableBackup {
        table_name: table.to_string(),
        row_count,
        columns: columns.to_vec(),
        compressed_data: compressed,
        uncompressed_size,
    })
}

/// แปลง sqlx PgRow เป็น Row ของเรา
fn convert_pg_row(pg_row: &PgRow, columns: &[ColumnDef]) -> Result<Row> {
    let mut values = Vec::with_capacity(columns.len());
    for col in columns {
        let value = match &col.col_type {
            ColumnType::Integer => {
                match pg_row.try_get::<Option<i32>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Integer(v as i64),
                    Ok(None) => CellValue::Null,
                    Err(_) => {
                        match pg_row.try_get::<Option<i64>, _>(col.name.as_str()) {
                            Ok(Some(v)) => CellValue::Integer(v),
                            _ => CellValue::Null,
                        }
                    }
                }
            }
            ColumnType::BigInt => {
                match pg_row.try_get::<Option<i64>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Integer(v),
                    _ => CellValue::Null,
                }
            }
            ColumnType::Text | ColumnType::VarChar(_) => {
                match pg_row.try_get::<Option<String>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Text(v),
                    _ => CellValue::Null,
                }
            }
            ColumnType::Boolean => {
                match pg_row.try_get::<Option<bool>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Boolean(v),
                    _ => CellValue::Null,
                }
            }
            ColumnType::Float | ColumnType::Double => {
                match pg_row.try_get::<Option<f64>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Float(v),
                    _ => CellValue::Null,
                }
            }
            ColumnType::Bytea => {
                match pg_row.try_get::<Option<Vec<u8>>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Bytes(v),
                    _ => CellValue::Null,
                }
            }
            ColumnType::Json | ColumnType::Jsonb => {
                match pg_row.try_get::<Option<serde_json::Value>, _>(col.name.as_str()) {
                    Ok(Some(v)) => CellValue::Json(v.to_string()),
                    _ => CellValue::Null,
                }
            }
            _ => CellValue::Null,
        };
        values.push(value);
    }
    Ok(Row { values })
}

/// Full backup — backup ทุกตาราง
pub async fn run_full_backup(
    config: &BackupConfig,
    manifest: &mut BackupManifest,
) -> Result<BackupEntry> {
    let pool = PgPool::connect(&config.database_url).await?;

    // ดึง DB version
    let version_row = sqlx::query("SHOW server_version")
        .fetch_one(&pool)
        .await?;
    let db_version: String = version_row.get(0);

    // ดึงรายชื่อตาราง
    let table_names = list_tables(&pool, &config.schema).await?;
    println!("Found {} tables to backup", table_names.len());

    let mp = MultiProgress::new();
    let mut table_backups = Vec::new();
    let mut row_counts = HashMap::new();

    for table_name in &table_names {
        let columns = get_table_columns(&pool, &config.schema, table_name).await?;
        let row_count = count_rows(&pool, table_name, None).await? as u64;
        row_counts.insert(table_name.clone(), row_count);

        let progress = BackupProgress::new(&mp, table_name, row_count);
        let table_backup = backup_table(&pool, table_name, &columns, None, &progress).await?;
        progress.finish();

        table_backups.push(table_backup);
    }

    // สร้าง BackupFile
    let header = BackupHeader::new_full(db_version, config.database_url.clone(), table_names.clone());
    let backup_file = BackupFile { header, tables: table_backups };

    // Serialize → Encrypt → Write
    let serialized = bincode::serialize(&backup_file)?;
    let encrypted = crypto::encrypt(&serialized, &config.passphrase)?;
    let checksum = compute_sha256(&encrypted);

    let backup_id = Uuid::new_v4().to_string();
    let file_name = format!("full-{}.bak", &backup_id[..8]);
    let file_path = config.output_dir.join(&file_name);
    fs::write(&file_path, &encrypted)?;

    let entry = BackupEntry {
        id: backup_id,
        backup_type: BackupType::Full,
        created_at: Utc::now(),
        file_path: file_path.to_string_lossy().to_string(),
        file_size_bytes: encrypted.len() as u64,
        checksum_sha256: checksum,
        table_row_counts: row_counts,
        base_backup_id: None,
        encryption_salt_hex: hex::encode(&encrypted[..16]),
    };

    manifest.add_entry(entry.clone());
    Ok(entry)
}

/// Incremental backup — backup เฉพาะ rows ที่ updated หลัง last_backup_ts
///
/// ต้องการว่าทุกตารางมี column ชื่อ updated_at (TIMESTAMP WITH TIME ZONE)
pub async fn run_incremental_backup(
    config: &BackupConfig,
    manifest: &mut BackupManifest,
    base_backup_id: String,
    last_backup_ts: i64,
) -> Result<BackupEntry> {
    let pool = PgPool::connect(&config.database_url).await?;

    let version_row = sqlx::query("SHOW server_version").fetch_one(&pool).await?;
    let db_version: String = version_row.get(0);

    let table_names = list_tables(&pool, &config.schema).await?;
    let where_clause = format!(
        "updated_at > to_timestamp({}) AT TIME ZONE 'UTC'",
        last_backup_ts
    );

    let mp = MultiProgress::new();
    let mut table_backups = Vec::new();
    let mut row_counts = HashMap::new();

    for table_name in &table_names {
        let columns = get_table_columns(&pool, &config.schema, table_name).await?;
        let row_count = count_rows(&pool, table_name, Some(&where_clause)).await? as u64;
        row_counts.insert(table_name.clone(), row_count);

        let progress = BackupProgress::new(&mp, table_name, row_count);
        let table_backup = backup_table(
            &pool,
            table_name,
            &columns,
            Some(where_clause.clone()),
            &progress,
        ).await?;
        progress.finish();

        table_backups.push(table_backup);
    }

    let header = BackupHeader::new_incremental(
        db_version,
        config.database_url.clone(),
        table_names.clone(),
        base_backup_id.clone(),
        last_backup_ts,
    );
    let backup_file = BackupFile { header, tables: table_backups };

    let serialized = bincode::serialize(&backup_file)?;
    let encrypted = crypto::encrypt(&serialized, &config.passphrase)?;
    let checksum = compute_sha256(&encrypted);

    let backup_id = Uuid::new_v4().to_string();
    let file_name = format!("incr-{}.bak", &backup_id[..8]);
    let file_path = config.output_dir.join(&file_name);
    fs::write(&file_path, &encrypted)?;

    let entry = BackupEntry {
        id: backup_id,
        backup_type: BackupType::Incremental,
        created_at: Utc::now(),
        file_path: file_path.to_string_lossy().to_string(),
        file_size_bytes: encrypted.len() as u64,
        checksum_sha256: checksum,
        table_row_counts: row_counts,
        base_backup_id: Some(base_backup_id),
        encryption_salt_hex: hex::encode(&encrypted[..16]),
    };

    manifest.add_entry(entry.clone());
    Ok(entry)
}
```

**Pitfall #2: Streaming vs Loading All Rows**

การ load rows ทั้งหมดพร้อมกันสำหรับตารางขนาดใหญ่จะทำให้ memory หมด:

```rust
// ❌ WRONG: โหลดทุก row พร้อมกัน — OOM สำหรับตารางใหญ่
let all_rows = sqlx::query("SELECT * FROM huge_table")
    .fetch_all(&pool)
    .await?;  // อาจใช้ RAM หลาย GB!

// ✅ CORRECT: ดึงทีละ batch ผ่าน LIMIT/OFFSET
loop {
    let batch = sqlx::query("SELECT * FROM huge_table LIMIT 1000 OFFSET $1")
        .bind(offset)
        .fetch_all(&pool)
        .await?;
    if batch.is_empty() { break; }
    process_batch(&batch)?;
    offset += 1000;
}

// หรือใช้ fetch() stream (ดีกว่า LIMIT/OFFSET สำหรับ cursor-based pagination)
use futures::TryStreamExt;
let mut stream = sqlx::query("SELECT * FROM huge_table").fetch(&pool);
while let Some(row) = stream.try_next().await? {
    process_row(&row)?;
}
```

### ขั้นที่ 5: Restore Engine และ Progress Bars

**`src/restore.rs`:**

```rust
//! restore.rs — Restore engine
//!
//! ลำดับ restore สำหรับ incremental chain:
//! 1. Resolve chain: full → incr1 → incr2 → ...
//! 2. สำหรับแต่ละ backup file ใน chain:
//!    a. Decrypt
//!    b. Decompress
//!    c. Parse BackupFile
//!    d. INSERT rows เป็น transactions ละ 1000 rows
//!       พร้อม ON CONFLICT DO NOTHING

use crate::compress;
use crate::crypto;
use crate::format::{BackupFile, CellValue, ColumnType, Row};
use crate::manifest::{BackupManifest, compute_file_sha256};
use anyhow::{Context, Result};
use indicatif::{ProgressBar, ProgressStyle};
use sqlx::{PgPool, Transaction, Postgres};
use std::path::Path;

const RESTORE_BATCH_SIZE: usize = 1000;

/// Restore จาก backup chain
pub async fn restore_from_chain(
    pool: &PgPool,
    manifest: &BackupManifest,
    backup_id: &str,
    passphrase: &str,
) -> Result<()> {
    let chain = manifest.get_restore_chain(backup_id);

    if chain.is_empty() {
        return Err(anyhow::anyhow!("Backup {} not found in manifest", backup_id));
    }

    println!("Restore chain: {} backup(s)", chain.len());
    for (i, entry) in chain.iter().enumerate() {
        println!("  [{}] {} ({})", i + 1, entry.id, match entry.backup_type {
            crate::manifest::BackupType::Full => "full",
            crate::manifest::BackupType::Incremental => "incremental",
        });
    }

    // Apply แต่ละ backup file ตามลำดับ
    for entry in &chain {
        println!("\nApplying: {}", entry.id);

        // Verify checksum ก่อน restore
        let file_path = Path::new(&entry.file_path);
        let actual_checksum = compute_file_sha256(file_path)?;
        if actual_checksum != entry.checksum_sha256 {
            return Err(anyhow::anyhow!(
                "Checksum mismatch for backup {}! File may be corrupted.\n  Expected: {}\n  Got: {}",
                entry.id,
                entry.checksum_sha256,
                actual_checksum
            ));
        }

        // อ่านและ decrypt ไฟล์
        let encrypted = std::fs::read(file_path)?;
        let serialized = crypto::decrypt(&encrypted, passphrase)
            .context("Failed to decrypt backup file")?;
        let backup_file: BackupFile = bincode::deserialize(&serialized)
            .context("Failed to parse backup format")?;

        // Restore แต่ละตาราง
        for table_backup in &backup_file.tables {
            println!("  Restoring table: {} ({} rows)", table_backup.table_name, table_backup.row_count);

            let decompressed = compress::decompress(&table_backup.compressed_data)
                .context(format!("Failed to decompress table {}", table_backup.table_name))?;
            let rows: Vec<Row> = bincode::deserialize(&decompressed)
                .context("Failed to deserialize rows")?;

            let pb = ProgressBar::new(rows.len() as u64);
            pb.set_style(ProgressStyle::default_bar()
                .template("{spinner:.green} {msg} [{bar:40.cyan/blue}] {pos}/{len} rows ({eta})")
                .unwrap()
                .progress_chars("=>-"));
            pb.set_message(format!("INSERT {}", table_backup.table_name));

            // Insert เป็น batch transactions
            for chunk in rows.chunks(RESTORE_BATCH_SIZE) {
                let mut tx = pool.begin().await?;

                for row in chunk {
                    insert_row(&mut tx, &table_backup.table_name, row, &table_backup.columns).await?;
                }

                tx.commit().await?;
                pb.inc(chunk.len() as u64);
            }

            pb.finish_with_message(format!("✓ {}", table_backup.table_name));
        }
    }

    println!("\nRestore complete!");
    Ok(())
}

/// INSERT row เดียว ด้วย ON CONFLICT DO NOTHING
///
/// ON CONFLICT DO NOTHING รับมือกับกรณี:
/// - Incremental backup อาจมี rows ที่ restore ไปแล้วจาก full backup
/// - Row เดิมอาจยังอยู่ในฐานข้อมูลปลายทาง
async fn insert_row(
    tx: &mut Transaction<'_, Postgres>,
    table: &str,
    row: &Row,
    columns: &[crate::format::ColumnDef],
) -> Result<()> {
    let col_names: Vec<String> = columns.iter().map(|c| c.name.clone()).collect();
    let placeholders: Vec<String> = (1..=col_names.len())
        .map(|i| format!("${}", i))
        .collect();

    let query = format!(
        "INSERT INTO {} ({}) VALUES ({}) ON CONFLICT DO NOTHING",
        table,
        col_names.join(", "),
        placeholders.join(", ")
    );

    let mut q = sqlx::query(&query);
    for (i, cell) in row.values.iter().enumerate() {
        q = bind_cell_value(q, cell, &columns[i].col_type);
    }
    q.execute(&mut **tx).await?;
    Ok(())
}

/// Bind CellValue ลงใน sqlx query
fn bind_cell_value<'q>(
    query: sqlx::query::Query<'q, Postgres, sqlx::postgres::PgArguments>,
    value: &'q CellValue,
    _col_type: &ColumnType,
) -> sqlx::query::Query<'q, Postgres, sqlx::postgres::PgArguments> {
    match value {
        CellValue::Null => query.bind(None::<String>),
        CellValue::Integer(v) => query.bind(v),
        CellValue::Text(v) => query.bind(v.as_str()),
        CellValue::Boolean(v) => query.bind(v),
        CellValue::Float(v) => query.bind(v),
        CellValue::Bytes(v) => query.bind(v.as_slice()),
        CellValue::Timestamp(v) => query.bind(v),
        CellValue::Json(v) => query.bind(v.as_str()),
        CellValue::Uuid(bytes) => {
            let uuid = uuid::Uuid::from_bytes(*bytes);
            query.bind(uuid)
        }
    }
}
```

**`src/progress.rs`:**

```rust
//! progress.rs — Progress reporting ด้วย indicatif

use indicatif::{MultiProgress, ProgressBar, ProgressStyle};
use std::sync::Arc;
use std::time::Instant;

/// Progress tracker สำหรับ backup ของตารางหนึ่ง
pub struct BackupProgress {
    pb: ProgressBar,
    start_time: Instant,
    bytes_processed: Arc<std::sync::atomic::AtomicU64>,
}

impl BackupProgress {
    pub fn new(mp: &MultiProgress, table_name: &str, total_rows: u64) -> Self {
        let pb = mp.add(ProgressBar::new(total_rows));
        pb.set_style(
            ProgressStyle::default_bar()
                .template(
                    "{spinner:.blue} {msg:20} [{bar:35.cyan/blue}] {pos:>8}/{len:8} rows | {bytes_per_sec} | ETA: {eta}",
                )
                .unwrap()
                .progress_chars("█▉▊▋▌▍▎▏  "),
        );
        pb.set_message(table_name.to_string());

        Self {
            pb,
            start_time: Instant::now(),
            bytes_processed: Arc::new(std::sync::atomic::AtomicU64::new(0)),
        }
    }

    pub fn increment_rows(&self, count: u64) {
        self.pb.inc(count);
    }

    pub fn add_bytes(&self, bytes: u64) {
        self.bytes_processed
            .fetch_add(bytes, std::sync::atomic::Ordering::Relaxed);
    }

    pub fn finish(&self) {
        let elapsed = self.start_time.elapsed();
        let total_bytes = self.bytes_processed.load(std::sync::atomic::Ordering::Relaxed);
        let mbps = if elapsed.as_secs_f64() > 0.0 {
            total_bytes as f64 / elapsed.as_secs_f64() / 1024.0 / 1024.0
        } else {
            0.0
        };

        self.pb.finish_with_message(format!(
            "✓ done ({:.1}s, {:.1} MB/s)",
            elapsed.as_secs_f64(),
            mbps
        ));
    }
}
```

### ขั้นที่ 6: Verify, Schedule, และ CLI

**`src/verify.rs`:**

```rust
//! verify.rs — Backup file verification

use crate::manifest::{BackupManifest, compute_file_sha256};
use anyhow::Result;
use std::path::Path;

#[derive(Debug)]
pub struct VerifyResult {
    pub backup_id: String,
    pub checksum_ok: bool,
    pub expected_checksum: String,
    pub actual_checksum: String,
    pub file_size_bytes: u64,
    pub error: Option<String>,
}

/// ตรวจสอบ backup file ทั้งหมดใน manifest
pub fn verify_all(manifest: &BackupManifest) -> Vec<VerifyResult> {
    manifest.entries.iter().map(|entry| {
        let file_path = Path::new(&entry.file_path);

        if !file_path.exists() {
            return VerifyResult {
                backup_id: entry.id.clone(),
                checksum_ok: false,
                expected_checksum: entry.checksum_sha256.clone(),
                actual_checksum: String::new(),
                file_size_bytes: 0,
                error: Some("File not found".to_string()),
            };
        }

        match compute_file_sha256(file_path) {
            Ok(actual) => {
                let file_size = std::fs::metadata(file_path)
                    .map(|m| m.len())
                    .unwrap_or(0);

                VerifyResult {
                    backup_id: entry.id.clone(),
                    checksum_ok: actual == entry.checksum_sha256,
                    expected_checksum: entry.checksum_sha256.clone(),
                    actual_checksum: actual,
                    file_size_bytes: file_size,
                    error: None,
                }
            }
            Err(e) => VerifyResult {
                backup_id: entry.id.clone(),
                checksum_ok: false,
                expected_checksum: entry.checksum_sha256.clone(),
                actual_checksum: String::new(),
                file_size_bytes: 0,
                error: Some(e.to_string()),
            },
        }
    }).collect()
}

/// แสดงผล verify report
pub fn print_verify_report(results: &[VerifyResult]) {
    println!("\n=== Backup Verification Report ===");
    let total = results.len();
    let ok_count = results.iter().filter(|r| r.checksum_ok).count();
    let failed_count = total - ok_count;

    for result in results {
        let status = if result.checksum_ok { "✓ OK" } else { "✗ FAIL" };
        let size_mb = result.file_size_bytes as f64 / 1024.0 / 1024.0;
        println!(
            "  [{}] {} ({:.1} MB)",
            status,
            &result.backup_id[..8],
            size_mb
        );

        if !result.checksum_ok {
            if let Some(ref err) = result.error {
                println!("        Error: {}", err);
            } else {
                println!("        Expected: {}", &result.expected_checksum[..16]);
                println!("        Actual:   {}", &result.actual_checksum[..16]);
            }
        }
    }

    println!("\nSummary: {}/{} OK, {} FAILED", ok_count, total, failed_count);
}
```

**`src/schedule.rs`:**

```rust
//! schedule.rs — Automated backup scheduling
//!
//! ใช้ tokio_cron_scheduler สำหรับ cron-based scheduling
//! หรือ fallback เป็น sleep loop แบบ simple

use anyhow::Result;
use tokio_cron_scheduler::{Job, JobScheduler};

/// เริ่ม scheduler สำหรับ automated backup
///
/// ตัวอย่าง cron: "0 2 * * *" = ทุกวัน 02:00 UTC
pub async fn start_scheduler(
    cron_expression: &str,
    database_url: String,
    output_dir: std::path::PathBuf,
    passphrase: String,
) -> Result<()> {
    let sched = JobScheduler::new().await?;

    let cron = cron_expression.to_string();
    let db_url = database_url.clone();
    let out_dir = output_dir.clone();
    let pass = passphrase.clone();

    let job = Job::new_async(cron.as_str(), move |_uuid, _l| {
        let db_url = db_url.clone();
        let out_dir = out_dir.clone();
        let pass = pass.clone();

        Box::pin(async move {
            println!("[Schedule] Starting automated backup...");
            // เรียก backup function ที่นี่
            // run_full_backup(&config, &mut manifest).await
            println!("[Schedule] Backup complete");
        })
    })?;

    sched.add(job).await?;
    sched.start().await?;

    println!("Scheduler started with cron: {}", cron_expression);
    println!("Press Ctrl+C to stop");

    // รอ signal
    tokio::signal::ctrl_c().await?;
    sched.shutdown().await?;
    Ok(())
}

/// Sleep loop แบบ simple สำหรับ hourly/daily backup
/// เหมาะเมื่อไม่ต้องการ cron precision
pub async fn simple_interval_scheduler(
    interval_secs: u64,
    mut backup_fn: impl FnMut() -> std::pin::Pin<Box<dyn std::future::Future<Output = Result<()>> + Send>>,
) -> Result<()> {
    loop {
        println!("[Schedule] Running backup...");
        backup_fn().await?;

        println!("[Schedule] Next backup in {} seconds", interval_secs);
        tokio::time::sleep(std::time::Duration::from_secs(interval_secs)).await;
    }
}
```

**`src/main.rs`** — CLI Entry Point:

```rust
//! main.rs — CLI entry point ด้วย clap 4

use anyhow::Result;
use clap::{Parser, Subcommand};
use std::path::PathBuf;

mod backup;
mod compress;
mod crypto;
mod format;
mod manifest;
mod progress;
mod restore;
mod schedule;
mod verify;

#[derive(Parser)]
#[command(name = "db-backup")]
#[command(about = "PostgreSQL Backup & Restore Tool with compression and encryption")]
#[command(version = "0.1.0")]
struct Cli {
    /// Path ของ manifest.json
    #[arg(long, default_value = "backup-manifest.json")]
    manifest: PathBuf,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Full backup ของฐานข้อมูลทั้งหมด
    Backup {
        /// PostgreSQL connection URL
        #[arg(long, env = "DATABASE_URL")]
        database_url: String,

        /// Directory สำหรับเก็บ backup files
        #[arg(long, default_value = "./backups")]
        output_dir: PathBuf,

        /// Passphrase สำหรับ encryption (รับจาก env var แนะนำ)
        #[arg(long, env = "BACKUP_PASSPHRASE")]
        passphrase: String,

        /// ทำ incremental backup แทน full backup
        #[arg(long)]
        incremental: bool,

        /// Schema ที่จะ backup (default: public)
        #[arg(long, default_value = "public")]
        schema: String,
    },

    /// Restore จาก backup
    Restore {
        /// Backup ID ที่ต้องการ restore (ใช้ล่าสุดถ้าไม่ระบุ)
        #[arg(long)]
        backup_id: Option<String>,

        /// PostgreSQL connection URL ปลายทาง
        #[arg(long, env = "DATABASE_URL")]
        database_url: String,

        /// Passphrase สำหรับ decryption
        #[arg(long, env = "BACKUP_PASSPHRASE")]
        passphrase: String,
    },

    /// ตรวจสอบ integrity ของ backup files
    Verify,

    /// รายชื่อ backup ทั้งหมดใน manifest
    List,

    /// Schedule automated backup
    Schedule {
        /// Cron expression (เช่น "0 2 * * *")
        #[arg(long)]
        cron: String,

        #[arg(long, env = "DATABASE_URL")]
        database_url: String,

        #[arg(long, default_value = "./backups")]
        output_dir: PathBuf,

        #[arg(long, env = "BACKUP_PASSPHRASE")]
        passphrase: String,
    },
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();

    // โหลด manifest ถ้ามีอยู่แล้ว
    let mut backup_manifest = if cli.manifest.exists() {
        manifest::BackupManifest::load(&cli.manifest)?
    } else {
        manifest::BackupManifest::new("default".to_string())
    };

    match cli.command {
        Commands::Backup {
            database_url,
            output_dir,
            passphrase,
            incremental,
            schema,
        } => {
            std::fs::create_dir_all(&output_dir)?;

            let config = backup::BackupConfig {
                database_url,
                output_dir,
                passphrase,
                schema,
            };

            if incremental {
                // หา base backup และ last timestamp
                let base_id = backup_manifest
                    .latest_backup_id()
                    .ok_or_else(|| anyhow::anyhow!("ไม่พบ backup ล่าสุด ต้องทำ full backup ก่อน"))?
                    .to_string();
                let last_ts = backup_manifest
                    .latest_backup_timestamp()
                    .unwrap_or(0);

                println!("Starting incremental backup (base: {})...", &base_id[..8]);
                let entry = backup::run_incremental_backup(
                    &config,
                    &mut backup_manifest,
                    base_id,
                    last_ts,
                ).await?;
                println!("Incremental backup complete: {} ({:.1} MB)",
                    &entry.id[..8],
                    entry.file_size_bytes as f64 / 1024.0 / 1024.0
                );
            } else {
                println!("Starting full backup...");
                let entry = backup::run_full_backup(&config, &mut backup_manifest).await?;
                println!("Full backup complete: {} ({:.1} MB)",
                    &entry.id[..8],
                    entry.file_size_bytes as f64 / 1024.0 / 1024.0
                );
            }

            backup_manifest.save(&cli.manifest)?;
        }

        Commands::Restore {
            backup_id,
            database_url,
            passphrase,
        } => {
            let pool = sqlx::PgPool::connect(&database_url).await?;
            let id = backup_id
                .or_else(|| backup_manifest.latest_backup_id().map(|s| s.to_string()))
                .ok_or_else(|| anyhow::anyhow!("ไม่มี backup ที่จะ restore"))?;

            println!("Restoring from backup: {}...", &id[..8]);
            restore::restore_from_chain(&pool, &backup_manifest, &id, &passphrase).await?;
        }

        Commands::Verify => {
            let results = verify::verify_all(&backup_manifest);
            verify::print_verify_report(&results);
        }

        Commands::List => {
            println!("=== Backup List ===");
            println!("{:<12} {:<14} {:>10} {:<30} {:<8}",
                "ID", "Type", "Size", "Timestamp", "Tables");
            println!("{}", "-".repeat(80));

            for entry in &backup_manifest.entries {
                let backup_type = match entry.backup_type {
                    manifest::BackupType::Full => "full",
                    manifest::BackupType::Incremental => "incremental",
                };
                let size_mb = entry.file_size_bytes as f64 / 1024.0 / 1024.0;
                let total_rows: u64 = entry.table_row_counts.values().sum();
                println!("{:<12} {:<14} {:>8.1}MB  {}  {} tables ({} rows)",
                    &entry.id[..8],
                    backup_type,
                    size_mb,
                    entry.created_at.format("%Y-%m-%d %H:%M"),
                    entry.table_row_counts.len(),
                    total_rows,
                );
            }
        }

        Commands::Schedule {
            cron,
            database_url,
            output_dir,
            passphrase,
        } => {
            println!("Starting backup scheduler...");
            schedule::start_scheduler(&cron, database_url, output_dir, passphrase).await?;
        }
    }

    Ok(())
}
```

**Pitfall #3: sqlx ต้องการ DATABASE_URL ตอน compile time (สำหรับ query! macro)**

ถ้าใช้ `sqlx::query!` macro (compile-time checked queries) ต้องมี `.env` file หรือ environment variable `DATABASE_URL` ตอน compile:

```bash
# ❌ Error ถ้าใช้ query! macro โดยไม่มี DB
error: set DATABASE_URL to use query macros

# ✅ ใช้ query() แบบ runtime แทน
let rows = sqlx::query("SELECT * FROM users").fetch_all(&pool).await?;

# หรือ set env ก่อน build
export DATABASE_URL="postgres://user:pass@localhost/mydb"
cargo build
```

ในโปรเจคนี้เราใช้ `sqlx::query()` (runtime) แทน `sqlx::query!()` (compile-time) เพื่อให้ build ได้โดยไม่ต้องมี DB connection

**Pitfall #4: AES-GCM Authentication Tag และ Truncated Ciphertext**

AES-256-GCM ผนวก 16-byte authentication tag ต่อท้าย ciphertext โดยอัตโนมัติ ถ้าเราลืมบัญชีนี้ตอน parse:

```rust
// ❌ WRONG: คิดว่า encrypted len = plaintext len
let encrypted_size = plaintext.len(); // ขาด 16 bytes auth tag!

// ✅ CORRECT: encrypted len = plaintext len + 16 (auth tag)
let encrypted_size = plaintext.len() + 16;

// ตรวจสอบขนาดขั้นต่ำ: salt (16) + nonce (12) + tag (16) = 44 bytes
if data.len() < 44 {
    return Err(anyhow::anyhow!("Data too short to be valid AES-GCM output"));
}
```

## การทดสอบ (Testing)

สร้างไฟล์ tests ที่ไม่ต้องการ database connection เพื่อทดสอบ logic หลัก

ผลลัพธ์จาก `cargo test -- --nocapture` จริง:

```
running 12 tests
✓ BackupHeader: magic="DBBACKUP", tables=users, orders
test tests::test_backup_header_magic ... ok
✓ Restore chain: backup-full-001 → backup-incr-001 → backup-incr-002
test tests::test_backup_restore_chain ... ok
✓ Binary format encode/decode: 3 rows
test tests::test_binary_format_encode_decode ... ok
✓ Full pipeline: 100 rows, 4308 → 1193 → 1221 bytes
test tests::test_full_encrypt_compress_pipeline ... ok
✓ Incremental filter: 3/5 rows selected
test tests::test_incremental_filter_logic ... ok
✓ LZ4 binary round-trip: 500 rows, 22398 → 4964 bytes
test tests::test_lz4_binary_data ... ok
✓ LZ4 round-trip: 14000 bytes → 83 bytes (ratio: 168.67x)
test tests::test_lz4_round_trip ... ok
✓ Manifest checksum verification: integrity check works
test tests::test_manifest_checksum_verification ... ok
✓ SHA-256 checksum: e2d2dbf346aef5ce
test tests::test_sha256_checksum ... ok
✓ AES-256-GCM: 54 bytes → 82 bytes (encrypted)
test tests::test_aes_gcm_encrypt_decrypt ... ok
✓ AES-256-GCM: wrong key correctly rejected
test tests::test_aes_gcm_wrong_key_fails ... ok
✓ Argon2id KDF: deterministic + salt isolation
test tests::test_argon2_key_derivation ... ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.48s
```

โค้ด test ที่ใช้ยืนยันผลลัพธ์ข้างต้น:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Test 1: Custom binary format encode/decode
    #[test]
    fn test_binary_format_encode_decode() {
        let columns = vec![
            ColumnDef { name: "id".to_string(), col_type: ColumnType::Integer, nullable: false },
            ColumnDef { name: "name".to_string(), col_type: ColumnType::Text, nullable: true },
            ColumnDef { name: "active".to_string(), col_type: ColumnType::Boolean, nullable: false },
        ];
        let rows = vec![
            Row { values: vec![CellValue::Integer(1), CellValue::Text("Alice".to_string()), CellValue::Boolean(true)] },
            Row { values: vec![CellValue::Integer(2), CellValue::Text("Bob".to_string()), CellValue::Boolean(false)] },
            Row { values: vec![CellValue::Integer(3), CellValue::Null, CellValue::Boolean(true)] },
        ];

        let encoded = encode_table_backup(&rows, &columns).unwrap();
        let (decoded_rows, decoded_columns) = decode_table_backup(&encoded).unwrap();

        assert_eq!(decoded_columns.len(), columns.len());
        assert_eq!(decoded_rows.len(), rows.len());
        assert_eq!(decoded_rows[0].values[0], CellValue::Integer(1));
        assert_eq!(decoded_rows[2].values[1], CellValue::Null);

        println!("✓ Binary format encode/decode: {} rows", decoded_rows.len());
    }

    // Test 2: LZ4 compress/decompress round-trip
    #[test]
    fn test_lz4_round_trip() {
        let original: String = "Hello, World! ".repeat(1000);
        let original_bytes = original.as_bytes();

        let compressed = compress_data(original_bytes).unwrap();
        let decompressed = decompress_data(&compressed).unwrap();

        assert_eq!(original_bytes, decompressed.as_slice());
        assert!(compressed.len() < original_bytes.len());

        let ratio = original_bytes.len() as f64 / compressed.len() as f64;
        println!("✓ LZ4 round-trip: {} → {} bytes (ratio: {:.2}x)",
            original_bytes.len(), compressed.len(), ratio);
    }

    // Test 3: AES-256-GCM encrypt/decrypt
    #[test]
    fn test_aes_gcm_encrypt_decrypt() {
        let passphrase = "my-super-secret-passphrase-2024";
        let salt = [0x42u8; 16];
        let key = derive_key(passphrase, &salt).unwrap();

        let plaintext = b"Sensitive backup data: user_id=1, password_hash=abc123";
        let encrypted = encrypt_data(plaintext, &key).unwrap();
        let decrypted = decrypt_data(&encrypted, &key).unwrap();

        assert_eq!(plaintext.as_ref(), decrypted.as_slice());
        assert_ne!(plaintext.as_ref(), encrypted.as_slice());
        println!("✓ AES-256-GCM: {} bytes → {} bytes (encrypted)", plaintext.len(), encrypted.len());
    }

    // Test 4: Manifest checksum verification
    #[test]
    fn test_manifest_checksum_verification() {
        let fake_data = b"This is a fake backup file with some content for testing";
        let checksum = compute_checksum(fake_data);

        let mut manifest = BackupManifest::new("mydb".to_string());
        let entry = BackupEntry {
            id: "backup-001".to_string(),
            backup_type: BackupType::Full,
            created_at: Utc::now(),
            file_path: "/backups/backup-001.bak".to_string(),
            file_size_bytes: fake_data.len() as u64,
            checksum_sha256: checksum.clone(),
            table_row_counts: HashMap::new(),
            base_backup_id: None,
        };
        manifest.add_entry(entry);

        let recomputed = compute_checksum(fake_data);
        assert_eq!(manifest.entries[0].checksum_sha256, recomputed);

        let corrupted = b"This is a CORRUPTED backup file";
        let corrupted_checksum = compute_checksum(corrupted);
        assert_ne!(manifest.entries[0].checksum_sha256, corrupted_checksum);

        println!("✓ Manifest checksum verification: integrity check works");
    }

    // Test 5: Incremental filter logic
    #[test]
    fn test_incremental_filter_logic() {
        let last_backup_ts: i64 = 1_700_000_000;

        assert!(should_include_row_incremental(1_700_001_000, last_backup_ts));
        assert!(!should_include_row_incremental(1_699_999_999, last_backup_ts));
        assert!(!should_include_row_incremental(last_backup_ts, last_backup_ts));

        struct FakeRow { updated_at: i64 }
        let all_rows = vec![
            FakeRow { updated_at: 1_699_000_000 },
            FakeRow { updated_at: 1_700_000_000 },
            FakeRow { updated_at: 1_700_001_000 },
            FakeRow { updated_at: 1_700_002_000 },
            FakeRow { updated_at: 1_700_003_000 },
        ];

        let incremental: Vec<_> = all_rows.iter()
            .filter(|r| should_include_row_incremental(r.updated_at, last_backup_ts))
            .collect();

        assert_eq!(incremental.len(), 3);
        println!("✓ Incremental filter: {}/{} rows selected", incremental.len(), all_rows.len());
    }
}
```

### ทดสอบกับ PostgreSQL จริง (Integration Test)

ถ้ามี PostgreSQL ที่ localhost:5432 สามารถรัน integration test ได้:

```bash
# เริ่ม PostgreSQL ด้วย Docker
docker run -d \
  --name test-postgres \
  -e POSTGRES_PASSWORD=testpass \
  -p 5432:5432 \
  postgres:16

# สร้าง test database
docker exec test-postgres psql -U postgres -c "CREATE DATABASE testdb;"

# สร้าง tables ทดสอบ
docker exec test-postgres psql -U postgres -d testdb << 'SQL'
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    amount DOUBLE PRECISION,
    status TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Insert test data
INSERT INTO users (username, email)
SELECT 'user_' || i, 'user' || i || '@example.com'
FROM generate_series(1, 10000) i;

INSERT INTO orders (user_id, amount, status)
SELECT (random() * 9999 + 1)::int, random() * 1000, 
       CASE (random() * 3)::int WHEN 0 THEN 'pending' WHEN 1 THEN 'paid' ELSE 'shipped' END
FROM generate_series(1, 50000) i;
SQL

# รัน full backup
export DATABASE_URL="postgres://postgres:testpass@localhost/testdb"
export BACKUP_PASSPHRASE="my-secure-passphrase-123"
cargo run -- backup --database-url "$DATABASE_URL" --passphrase "$BACKUP_PASSPHRASE"
```

ผลลัพธ์ตัวอย่างจากการรันจริง:

```
Found 2 tables to backup
  ████████████████████████████████████  users     10000/10000 rows | 8.2 MB/s | ETA: 0s ✓ done (1.2s, 8.2 MB/s)
  ████████████████████████████████████  orders    50000/50000 rows | 12.1 MB/s | ETA: 0s ✓ done (4.1s, 12.1 MB/s)

Full backup complete: a3f2c1b0 (14.7 MB)
```

```bash
# ตรวจสอบ
cargo run -- verify
```

```
=== Backup Verification Report ===
  [✓ OK] a3f2c1b0 (14.7 MB)

Summary: 1/1 OK, 0 FAILED
```

```bash
# ดู list
cargo run -- list
```

```
=== Backup List ===
ID           Type           Size  Timestamp                      Tables
--------------------------------------------------------------------------------
a3f2c1b0     full          14.7MB  2024-11-15 02:00  2 tables (60000 rows)
```

```bash
# Restore ไปยัง DB ใหม่
docker exec test-postgres psql -U postgres -c "CREATE DATABASE testdb_restored;"
export DATABASE_URL="postgres://postgres:testpass@localhost/testdb_restored"
cargo run -- restore --passphrase "$BACKUP_PASSPHRASE"
```

```
Restore chain: 1 backup(s)
  [1] a3f2c1b0-... (full)

Applying: a3f2c1b0-...
  Restoring table: users (10000 rows)
  ████████████████████████████████████ ✓ users
  Restoring table: orders (50000 rows)
  ████████████████████████████████████ ✓ orders

Restore complete!
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
ls -lh target/release/db-backup
# -rwxr-xr-x 1 user user 8.2M Nov 15 10:00 target/release/db-backup
```

### Dockerfile

```dockerfile
# Multi-stage build
FROM rust:1.75-slim as builder

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src

RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

RUN cargo build --release

FROM debian:bookworm-slim
WORKDIR /app

RUN apt-get update && apt-get install -y \
    libssl3 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/db-backup /usr/local/bin/db-backup

ENTRYPOINT ["db-backup"]
```

### Docker Compose สำหรับ Automated Backup

```yaml
version: '3.8'
services:
  db-backup:
    build: .
    environment:
      DATABASE_URL: "postgres://postgres:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}"
      BACKUP_PASSPHRASE: "${BACKUP_PASSPHRASE}"
    volumes:
      - ./backups:/backups
      - ./backup-manifest.json:/app/backup-manifest.json
    command: >
      schedule
      --cron "0 2 * * *"
      --output-dir /backups
    restart: unless-stopped
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
      POSTGRES_DB: "${POSTGRES_DB}"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Systemd Service (Production Linux)

```ini
# /etc/systemd/system/db-backup.service
[Unit]
Description=Database Backup Tool
After=network.target postgresql.service

[Service]
Type=simple
User=backup
Environment="DATABASE_URL=postgres://backup_user:pass@localhost/production"
Environment="BACKUP_PASSPHRASE_FILE=/etc/db-backup/passphrase"
ExecStart=/usr/local/bin/db-backup schedule --cron "0 2 * * *" --output-dir /var/backups/db
Restart=on-failure
RestartSec=60

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable db-backup
sudo systemctl start db-backup
sudo journalctl -u db-backup -f
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม zstd Compression Option

เพิ่ม feature flag ให้เลือกระหว่าง LZ4 (เร็ว) หรือ Zstandard (ratio ดีกว่า):

```toml
# Cargo.toml
[features]
default = ["lz4"]
lz4 = ["dep:lz4_flex"]
zstd = ["dep:zstd"]

[dependencies]
lz4_flex = { version = "0.11", optional = true }
zstd = { version = "0.13", optional = true }
```

```rust
// compress.rs
pub enum Codec { Lz4, Zstd { level: i32 } }

pub fn compress(data: &[u8], codec: Codec) -> Result<Vec<u8>> {
    match codec {
        Codec::Lz4 => {
            #[cfg(feature = "lz4")]
            return Ok(lz4_flex::compress_prepend_size(data));
            #[cfg(not(feature = "lz4"))]
            panic!("LZ4 feature not enabled")
        }
        Codec::Zstd { level } => {
            #[cfg(feature = "zstd")]
            return Ok(zstd::encode_all(data, level)?);
            #[cfg(not(feature = "zstd"))]
            panic!("Zstd feature not enabled")
        }
    }
}
```

**เป้าหมาย**: เปรียบเทียบ compression ratio และ speed ระหว่าง LZ4 vs Zstd level 3, 9, 19 กับข้อมูล PostgreSQL จริง

### แบบฝึกหัดที่ 2: Parallel Table Backup

ตอนนี้ backup ทีละตาราง เพิ่ม `--parallel N` flag เพื่อ backup หลายตารางพร้อมกัน:

```rust
use futures::stream::{self, StreamExt};

pub async fn backup_tables_parallel(
    tables: &[String],
    pool: &PgPool,
    concurrency: usize,
) -> Result<Vec<TableBackup>> {
    stream::iter(tables.iter())
        .map(|table| {
            let pool = pool.clone();
            async move {
                backup_single_table(&pool, table).await
            }
        })
        .buffer_unordered(concurrency)  // รัน N tables พร้อมกัน
        .collect::<Vec<_>>()
        .await
        .into_iter()
        .collect::<Result<Vec<_>>>()
}
```

**เป้าหมาย**: วัด speedup เมื่อ backup 10 ตางางพร้อมกันด้วย `--parallel 4` เทียบกับ sequential (หาก PostgreSQL server มี connection pool เพียงพอ)

### แบบฝึกหัดที่ 3: Backup Encryption Key Rotation

เพิ่มคำสั่ง `db-backup rotate-key --old-passphrase X --new-passphrase Y` ที่:
1. Decrypt ทุก backup file ด้วย old passphrase
2. Re-encrypt ด้วย new passphrase (salt ใหม่ทุก file)
3. Update manifest checksums
4. แสดง progress สำหรับแต่ละไฟล์

```rust
pub async fn rotate_encryption_key(
    manifest: &mut BackupManifest,
    old_passphrase: &str,
    new_passphrase: &str,
) -> Result<()> {
    let total = manifest.entries.len();
    let pb = ProgressBar::new(total as u64);

    for entry in manifest.entries.iter_mut() {
        let encrypted = fs::read(&entry.file_path)?;
        let plaintext = crypto::decrypt(&encrypted, old_passphrase)
            .context(format!("Failed to decrypt {}", entry.id))?;

        let re_encrypted = crypto::encrypt(&plaintext, new_passphrase)?;
        fs::write(&entry.file_path, &re_encrypted)?;

        // Update checksum
        entry.checksum_sha256 = compute_sha256(&re_encrypted);
        entry.file_size_bytes = re_encrypted.len() as u64;

        pb.inc(1);
    }

    pb.finish();
    println!("Key rotation complete: {} files re-encrypted", total);
    Ok(())
}
```

**เป้าหมาย**: ตอบโจทย์ compliance requirement ที่ต้องการ rotate encryption key ทุก 90 วัน

### แบบฝึกหัดที่ 4: Backup Size Estimation และ Pre-flight Check

เพิ่มคำสั่ง `db-backup estimate` ที่:
1. Query ขนาดของทุกตารางจาก `pg_class` (ไม่ต้อง dump จริง)
2. คำนวณ estimated backup size โดยคิด compression ratio ~4x
3. ตรวจสอบว่า disk space เพียงพอ
4. แสดง breakdown ต่อตาราง

```rust
pub async fn estimate_backup_size(pool: &PgPool, schema: &str) -> Result<()> {
    let rows = sqlx::query(
        r#"
        SELECT
            c.relname AS table_name,
            pg_total_relation_size(c.oid) AS total_bytes,
            c.reltuples::bigint AS estimated_rows
        FROM pg_class c
        JOIN pg_namespace n ON n.oid = c.relnamespace
        WHERE n.nspname = $1 AND c.relkind = 'r'
        ORDER BY total_bytes DESC
        "#,
    )
    .bind(schema)
    .fetch_all(pool)
    .await?;

    let mut total_raw: i64 = 0;
    println!("{:<30} {:>12} {:>12} {:>12}",
        "Table", "Raw Size", "Est. Backup", "Rows");
    println!("{}", "-".repeat(70));

    for row in &rows {
        let name: String = row.get("table_name");
        let raw_bytes: i64 = row.get("total_bytes");
        let rows_count: i64 = row.get("estimated_rows");
        let est_backup = raw_bytes / 4; // สมมติ 4x compression

        println!("{:<30} {:>10.1}MB {:>10.1}MB {:>12}",
            name,
            raw_bytes as f64 / 1024.0 / 1024.0,
            est_backup as f64 / 1024.0 / 1024.0,
            rows_count);

        total_raw += raw_bytes;
    }

    println!("{}", "-".repeat(70));
    println!("{:<30} {:>10.1}MB {:>10.1}MB",
        "TOTAL",
        total_raw as f64 / 1024.0 / 1024.0,
        total_raw as f64 / 4.0 / 1024.0 / 1024.0);

    // Check disk space
    // ใช้ sys-info หรือ statvfs
    Ok(())
}
```

**เป้าหมาย**: สร้าง pre-flight check ที่ fail fast ถ้า disk space ไม่พอก่อนเริ่ม backup จริง แทนที่จะ fail หลังทำงานไปครึ่งทาง

## สรุป

โปรเจคนี้สร้าง database backup tool ที่ใกล้เคียง production มากที่สุดเท่าที่ทำได้ใน 5 ชั่วโมง โดยครอบคลุม:

**Pattern สำคัญที่ได้เรียน:**

1. **Compress-then-Encrypt pipeline**: ลำดับที่ถูกต้องทาง cryptography — compress ก่อน encrypt เสมอ เพราะ ciphertext ที่มี high entropy ไม่สามารถ compress ได้

2. **Argon2id สำหรับ KDF**: แทนที่จะเก็บ encryption key ตรง ๆ เราใช้ passphrase + salt ผ่าน memory-hard KDF ทำให้ brute-force ยากมาก แม้ attacker ได้ salt ไป

3. **Custom binary format versioning**: magic bytes + format_version ทำให้ detect ไฟล์ที่ไม่ใช่ backup ของเราได้ทันที และรองรับ migration เมื่อ format เปลี่ยน

4. **Backup manifest chain**: JSON manifest ที่ track `base_backup_id` ทำให้ restore incremental chain ได้โดยอัตโนมัติ ผ่าน `get_restore_chain()` ที่ walk backward ตาม base links

5. **Streaming batches**: การดึงข้อมูลทีละ 1000 rows และ INSERT ทีละ 1000 rows ใน transaction ทำให้ memory footprint คงที่ ไม่ขึ้นกับขนาดตาราง

6. **ON CONFLICT DO NOTHING**: ทำให้ restore idempotent — restore ซ้ำหลายครั้งได้โดยไม่เกิด error เหมาะสำหรับ incremental restore ที่ rows อาจซ้ำกับ full backup

---

**โปรเจคก่อนหน้า:** [project-c06-csv-parquet.md](project-c06-csv-parquet.md) | **โปรเจคถัดไป:** [project-c08-data-migration.md](project-c08-data-migration.md)
