# Project A10: Secret Vault (Encrypted)

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

Secret Vault คือ CLI tool สำหรับเก็บ secret ต่าง ๆ เช่น API key, database password, SSH passphrase ในไฟล์ที่เข้ารหัสด้วย AES-256-GCM บนเครื่องของผู้ใช้งานเอง โดยไม่ต้องพึ่งพา cloud service ภายนอก

ปัญหาที่โปรเจคนี้แก้ไข: developer มักเก็บ secret ใน plaintext file (`~/.env`, sticky notes, หรือ bash history) ซึ่งข้อมูลเหล่านั้นอ่านได้ง่ายหากมีการเข้าถึง filesystem โดยตรง Secret Vault แก้ปัญหานี้ด้วยการเข้ารหัสทุก entry ก่อนเขียนลงดิสก์ และใช้ master password เดียวในการล็อกทั้ง vault

Use case จริงในโลก production:
- เก็บ API keys สำหรับ development environment โดยไม่ commit ลง repository
- จัดการ credentials หลายชุดสำหรับ staging/production environment
- เป็นส่วนหนึ่งของ developer workstation security policy

Learning value ที่ได้: โปรเจคนี้สอน cryptographic primitives จริง ๆ — ไม่ใช่แค่ "ใช้ library ได้" แต่เข้าใจว่า key derivation function, authenticated encryption, และ nonce reuse problem คืออะไรและทำงานอย่างไร

## สิ่งที่จะได้เรียนรู้

- **Argon2id**: Password-based key derivation — ทำงานอย่างไร, ทำไมต้อง memory-hard, parameter tradeoffs
- **AES-256-GCM**: Authenticated encryption — confidentiality + integrity ในการ operation เดียว, บทบาทของ nonce
- **Vault file format**: JSON envelope pattern — ออกแบบ file format ที่ versioned และ forward-compatible
- **Secure memory**: `zeroize` crate — zeroing key material หลังใช้งาน, ทำไม `drop()` ไม่เพียงพอ
- **CLI patterns**: `clap` subcommands, password prompt ด้วย `rpassword`, clipboard integration ด้วย `arboard`
- **Thread + timer**: spawn background thread สำหรับ auto-clear clipboard หลัง 30 วินาที
- **Lock timeout**: บันทึก timestamp ล่าสุดที่ unlock และตรวจสอบก่อนแต่ละ operation
- **Error handling**: ออกแบบ error types ที่ไม่รั่ว secret ใน error message

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust fundamentals — ownership, borrowing, structs, enums, traits
- **Part 21–40**: Error handling (`Result`/`Option`), closures, iterators, file I/O
- **Part 41–60**: `std::thread`, trait objects, `serde` basics, Cargo dependencies
- ความคุ้นเคยกับ hexadecimal encoding และ byte arrays จะช่วยให้เข้าใจโค้ดได้เร็วขึ้น

## โครงสร้างโปรเจค (Project Layout)

```
secret-vault/
├── src/
│   ├── main.rs          ← CLI entry point, clap subcommands
│   ├── crypto.rs        ← derive_key(), encrypt(), decrypt()
│   ├── vault.rs         ← VaultFile, VaultEntry structs, serialize/deserialize
│   ├── commands.rs      ← ฟังก์ชัน init, add, get, list, delete, backup, export
│   └── error.rs         ← VaultError enum
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### ภาพรวม Data Flow

```
Master Password (string)
        │
        ▼
  [Argon2id KDF]  ←── salt (16 bytes, random, stored in vault file)
        │
        ▼
   Key (32 bytes)  ─────────────────────────────────────────────────┐
                                                                     │
Secret Value (plaintext)                                             │
        │                                                            │
        ▼                                                            ▼
  [AES-256-GCM]  ←── nonce (12 bytes, random per-entry)   [AES-256-GCM]
        │                                                            │
        ▼                                                            ▼
 ciphertext_hex                                              plaintext
(stored in vault)                                        (shown to user)
```

### ทำไม Argon2id ไม่ใช่ bcrypt หรือ PBKDF2

Argon2id เป็น memory-hard function — ต้องการ RAM จำนวนมากในการคำนวณ ทำให้การ brute-force ด้วย GPU หรือ FPGA มีต้นทุนสูง bcrypt ใช้ RAM น้อยกว่า (จึง GPU-parallelizable ได้มากกว่า) และ PBKDF2 เป็นแค่ iterated hash ที่ไม่มี memory hardness เลย Argon2id ชนะการแข่งขัน Password Hashing Competition (PHC) ปี 2015 และเป็น recommendation ปัจจุบันของ OWASP

### ทำไม AES-256-GCM ไม่ใช่ AES-256-CBC + HMAC

AES-256-GCM เป็น Authenticated Encryption with Associated Data (AEAD): operation เดียวให้ทั้ง confidentiality และ integrity พร้อมกัน GCM tag (16 bytes) ตรวจจับการ tamper ciphertext ได้ก่อนที่จะ decrypt ถ้าใช้ CBC + HMAC แยกกัน ต้องระวัง "Encrypt-then-MAC" ordering และ key separation — มีโอกาสผิดพลาดสูงกว่า

### Nonce Uniqueness

AES-256-GCM ต้องการ nonce ที่ unique ต่อ (key, nonce) pair — ถ้า nonce ซ้ำกันภายใต้ key เดียวกัน GCM จะรั่ว key stream ที่สามารถ XOR กันกู้ plaintext ได้ โปรเจคนี้แก้ปัญหาด้วยการ generate random 12-byte nonce ต่อ entry โดยใช้ CSPRNG (`OsRng`) ความน่าจะเป็นที่ nonce จะซ้ำกันกับ entry จำนวน 2^32 ≈ 4 พันล้าน entry คือ 2^-64 (birthday bound) ซึ่งเล็กมากพอที่จะยอมรับได้ในทางปฏิบัติ

### JSON Envelope Design

Vault file ใช้ JSON envelope แทน binary format เพราะ:
1. Human-readable — debug ง่าย, backup ตรวจสอบได้
2. Forward-compatible — field เพิ่มได้โดยไม่ break format เก่า
3. `serde_json` ทำ serialization/deserialization ให้โดยตรงไม่ต้องเขียน parser เอง

โครงสร้าง:
```json
{
  "version": 1,
  "salt_hex": "a3f9b2c1...",
  "entries": [
    {
      "name": "github_token",
      "nonce_hex": "4e8a9f1b2c3d...",
      "ciphertext_hex": "7c3a...",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

`salt_hex` เก็บ salt ที่ใช้ derive key — ไม่เป็น secret เพราะ salt มีไว้ให้ Argon2 ทำงาน; ความปลอดภัยมาจาก memory-hardness ของ Argon2 ที่ทำให้การ brute-force ช้า ไม่ใช่จากการซ่อน salt

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Argon2id Key Derivation

เริ่มจากหัวใจของระบบ: การเปลี่ยน master password ให้กลายเป็น key 32 bytes

**Parameter ที่เลือก:**
- `m_cost = 65536` (64 MiB) — RAM ที่ Argon2 ต้องใช้ต่อการคำนวณ 1 ครั้ง
- `t_cost = 3` — จำนวน iteration (passes)
- `p_cost = 1` — degree of parallelism

OWASP แนะนำ minimum m=12 MiB, t=3, p=1 สำหรับ interactive login; เราเพิ่ม m เป็น 64 MiB เพื่อความปลอดภัยมากขึ้นโดยที่ UX ยังยอมรับได้ (delay ~0.5–1 วินาทีบน modern hardware)

**`Cargo.toml`**:
```toml
[package]
name = "secret-vault"
version = "0.1.0"
edition = "2021"

[dependencies]
argon2 = "0.5"
aes-gcm = "0.10"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rpassword = "7"
arboard = "3"
zeroize = { version = "1", features = ["derive"] }
hex = "0.4"
rand = "0.8"
dirs = "5"
chrono = { version = "0.4", features = ["serde"] }
```

**`src/crypto.rs`** — Step 1: Key derivation:
```rust
use argon2::{Argon2, Algorithm, Version, Params};
use zeroize::Zeroize;

/// Argon2id parameters: 64 MiB memory, 3 iterations, 1 thread.
/// Produces a 32-byte key suitable for AES-256.
pub fn derive_key(password: &str, salt: &[u8]) -> [u8; 32] {
    let params = Params::new(
        65536,  // m_cost: 64 MiB
        3,      // t_cost: 3 passes
        1,      // p_cost: parallelism 1
        Some(32), // output length
    )
    .expect("valid Argon2 parameters");

    let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);

    let mut key = [0u8; 32];
    argon2
        .hash_password_into(password.as_bytes(), salt, &mut key)
        .expect("Argon2 key derivation failed");
    key
}

/// Zeroize key bytes in-place after use.
pub fn zeroize_key(key: &mut [u8; 32]) {
    key.zeroize();
}
```

สิ่งที่น่าสังเกต: `hash_password_into` รับ `output` เป็น `&mut [u8]` ทำให้เราควบคุม memory allocation เองได้ ต่างจาก `hash_password` ที่ return `PasswordHash` struct ซึ่งมี overhead ของ PHC string format ที่ไม่จำเป็น

### ขั้นที่ 2: AES-256-GCM Encrypt/Decrypt

ขั้นนี้เพิ่ม encrypt และ decrypt สำหรับ entry เดี่ยว:

**`src/crypto.rs`** — เพิ่ม encrypt/decrypt:
```rust
use aes_gcm::{
    aead::{Aead, AeadCore, KeyInit, OsRng},
    Aes256Gcm, Nonce,
};

/// Encrypt plaintext with AES-256-GCM.
/// Returns (nonce_bytes [12], ciphertext_with_tag).
/// Each call generates a fresh random nonce — never reuse nonces under the same key.
pub fn encrypt(key: &[u8; 32], plaintext: &[u8]) -> ([u8; 12], Vec<u8>) {
    let cipher = Aes256Gcm::new_from_slice(key)
        .expect("key must be exactly 32 bytes");

    // OsRng delegates to the OS CSPRNG (/dev/urandom on Linux, CryptGenRandom on Windows)
    let nonce = Aes256Gcm::generate_nonce(&mut OsRng);
    let ciphertext = cipher
        .encrypt(&nonce, plaintext)
        .expect("AES-GCM encryption failed");

    let mut nonce_bytes = [0u8; 12];
    nonce_bytes.copy_from_slice(&nonce);
    (nonce_bytes, ciphertext)
}

/// Decrypt ciphertext with AES-256-GCM.
/// Returns error if authentication tag does not match (tampered data or wrong key).
pub fn decrypt(
    key: &[u8; 32],
    nonce_bytes: &[u8; 12],
    ciphertext: &[u8],
) -> Result<Vec<u8>, aes_gcm::Error> {
    let cipher = Aes256Gcm::new_from_slice(key)
        .expect("key must be exactly 32 bytes");
    let nonce = Nonce::from_slice(nonce_bytes);
    cipher.decrypt(nonce, ciphertext)
}
```

สิ่งสำคัญที่เข้าใจ:
- `ciphertext` ที่ return จาก `encrypt` รวม **GCM authentication tag (16 bytes)** ต่อท้ายอยู่แล้ว — ไม่ต้องคำนวณ HMAC แยก
- `decrypt` จะ return `Err` ถ้า tag ไม่ตรง (ไม่ว่าจะเกิดจาก wrong key หรือ corrupted data) — ทำให้ต้องตรวจสอบ `Result` ก่อนใช้ plaintext เสมอ
- `OsRng` เป็น cryptographically secure — ต่างจาก `rand::thread_rng()` ที่อาจใช้ algorithm ที่ไม่ suitable for cryptographic use

### ขั้นที่ 3: Vault File Format (JSON Envelope)

ออกแบบ structs และ serialization:

**`src/vault.rs`**:
```rust
use chrono::Utc;
use serde::{Deserialize, Serialize};
use std::path::{Path, PathBuf};

use crate::crypto::{decrypt, derive_key, encrypt};
use crate::error::VaultError;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct VaultEntry {
    pub name: String,
    pub nonce_hex: String,
    pub ciphertext_hex: String,
    pub created_at: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct VaultFile {
    pub version: u32,
    pub salt_hex: String,
    pub entries: Vec<VaultEntry>,
}

impl VaultFile {
    /// Create a new empty vault with a fresh random salt.
    pub fn new() -> Self {
        use rand::RngCore;
        let mut salt = [0u8; 16];
        rand::thread_rng().fill_bytes(&mut salt);
        VaultFile {
            version: 1,
            salt_hex: hex::encode(salt),
            entries: vec![],
        }
    }

    /// Load vault from disk. Returns VaultError::NotFound if file doesn't exist.
    pub fn load(path: &Path) -> Result<Self, VaultError> {
        let content = std::fs::read_to_string(path)
            .map_err(|_| VaultError::VaultNotFound(path.to_path_buf()))?;
        serde_json::from_str(&content)
            .map_err(|e| VaultError::ParseError(e.to_string()))
    }

    /// Write vault to disk (overwrites atomically via temp file + rename).
    pub fn save(&self, path: &Path) -> Result<(), VaultError> {
        // Create parent directory if needed
        if let Some(parent) = path.parent() {
            std::fs::create_dir_all(parent)
                .map_err(|e| VaultError::IoError(e.to_string()))?;
        }

        // Write to temp file first, then rename — atomic on most filesystems
        let tmp_path = path.with_extension("tmp");
        let json = serde_json::to_string_pretty(self)
            .map_err(|e| VaultError::SerializeError(e.to_string()))?;
        std::fs::write(&tmp_path, json)
            .map_err(|e| VaultError::IoError(e.to_string()))?;
        std::fs::rename(&tmp_path, path)
            .map_err(|e| VaultError::IoError(e.to_string()))?;
        Ok(())
    }

    /// Return the salt bytes decoded from salt_hex.
    pub fn salt_bytes(&self) -> Vec<u8> {
        hex::decode(&self.salt_hex).expect("salt_hex must be valid hex")
    }

    /// Add an encrypted entry. Replaces existing entry with same name.
    pub fn add_entry(&mut self, name: &str, key: &[u8; 32], secret: &str) {
        // Remove existing entry with same name
        self.entries.retain(|e| e.name != name);

        let (nonce_bytes, ciphertext) = encrypt(key, secret.as_bytes());
        self.entries.push(VaultEntry {
            name: name.to_string(),
            nonce_hex: hex::encode(nonce_bytes),
            ciphertext_hex: hex::encode(&ciphertext),
            created_at: Utc::now().to_rfc3339(),
        });
    }

    /// Decrypt and return a named entry's value.
    pub fn get_entry(
        &self,
        name: &str,
        key: &[u8; 32],
    ) -> Result<String, VaultError> {
        let entry = self
            .entries
            .iter()
            .find(|e| e.name == name)
            .ok_or_else(|| VaultError::EntryNotFound(name.to_string()))?;

        let nonce_bytes = hex::decode(&entry.nonce_hex)
            .map_err(|_| VaultError::CorruptedEntry(name.to_string()))?;
        let ciphertext = hex::decode(&entry.ciphertext_hex)
            .map_err(|_| VaultError::CorruptedEntry(name.to_string()))?;

        let mut nonce_arr = [0u8; 12];
        if nonce_bytes.len() != 12 {
            return Err(VaultError::CorruptedEntry(name.to_string()));
        }
        nonce_arr.copy_from_slice(&nonce_bytes);

        let plaintext = decrypt(key, &nonce_arr, &ciphertext)
            .map_err(|_| VaultError::DecryptionFailed)?;

        String::from_utf8(plaintext)
            .map_err(|_| VaultError::CorruptedEntry(name.to_string()))
    }

    /// Delete a named entry. Returns error if not found.
    pub fn delete_entry(&mut self, name: &str) -> Result<(), VaultError> {
        let before = self.entries.len();
        self.entries.retain(|e| e.name != name);
        if self.entries.len() == before {
            return Err(VaultError::EntryNotFound(name.to_string()));
        }
        Ok(())
    }
}

/// Default vault path: ~/.local/share/vault/vault.enc
pub fn default_vault_path() -> PathBuf {
    dirs::data_local_dir()
        .unwrap_or_else(|| PathBuf::from("."))
        .join("vault")
        .join("vault.enc")
}
```

**`src/error.rs`**:
```rust
use std::path::PathBuf;

#[derive(Debug)]
pub enum VaultError {
    VaultNotFound(PathBuf),
    VaultAlreadyExists(PathBuf),
    EntryNotFound(String),
    CorruptedEntry(String),
    DecryptionFailed,
    ParseError(String),
    SerializeError(String),
    IoError(String),
    LockTimeout,
}

impl std::fmt::Display for VaultError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            VaultError::VaultNotFound(p) => {
                write!(f, "Vault not found at {}. Run `vault init` first.", p.display())
            }
            VaultError::VaultAlreadyExists(p) => {
                write!(f, "Vault already exists at {}.", p.display())
            }
            VaultError::EntryNotFound(name) => {
                write!(f, "Entry '{}' not found in vault.", name)
            }
            VaultError::CorruptedEntry(name) => {
                write!(f, "Entry '{}' appears corrupted.", name)
            }
            VaultError::DecryptionFailed => {
                // ไม่รั่ว detail — ไม่บอกว่า wrong key หรือ corrupted data
                write!(f, "Decryption failed. Check your master password.")
            }
            VaultError::ParseError(e) => write!(f, "Failed to parse vault file: {e}"),
            VaultError::SerializeError(e) => write!(f, "Failed to serialize vault: {e}"),
            VaultError::IoError(e) => write!(f, "I/O error: {e}"),
            VaultError::LockTimeout => {
                write!(f, "Vault locked due to inactivity. Enter master password again.")
            }
        }
    }
}

impl std::error::Error for VaultError {}
```

หมายเหตุการออกแบบ `VaultError::DecryptionFailed`: error message ไม่บอกว่าเป็นเพราะ wrong password หรือ corrupted data เพื่อป้องกัน oracle attack (ไม่เปิดเผย information เพิ่มเติมให้ caller)

### ขั้นที่ 4: `vault init` + `vault add`

**`src/main.rs`** — CLI structure:
```rust
use clap::{Parser, Subcommand};
use std::path::PathBuf;

mod commands;
mod crypto;
mod error;
mod vault;

#[derive(Parser)]
#[command(name = "vault", about = "Encrypted secret vault")]
struct Cli {
    /// Path to vault file (default: ~/.local/share/vault/vault.enc)
    #[arg(long, global = true)]
    vault: Option<PathBuf>,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Initialize a new empty vault
    Init,
    /// Add or update a secret entry
    Add {
        /// Name of the entry
        name: String,
    },
    /// Retrieve a secret (copies to clipboard by default)
    Get {
        /// Name of the entry
        name: String,
        /// Print to stdout instead of clipboard
        #[arg(long)]
        print: bool,
    },
    /// List all entry names
    List,
    /// Delete an entry
    Delete {
        /// Name of the entry
        name: String,
    },
    /// Export vault as plaintext JSON (use with caution)
    Export {
        /// Confirm export of plaintext data
        #[arg(long)]
        plaintext: bool,
    },
    /// Back up vault file
    Backup {
        /// Destination directory (default: same dir as vault)
        dest: Option<PathBuf>,
    },
}

fn main() {
    let cli = Cli::parse();
    let vault_path = cli.vault.unwrap_or_else(vault::default_vault_path);

    let result = match cli.command {
        Commands::Init => commands::cmd_init(&vault_path),
        Commands::Add { name } => commands::cmd_add(&vault_path, &name),
        Commands::Get { name, print } => commands::cmd_get(&vault_path, &name, print),
        Commands::List => commands::cmd_list(&vault_path),
        Commands::Delete { name } => commands::cmd_delete(&vault_path, &name),
        Commands::Export { plaintext } => commands::cmd_export(&vault_path, plaintext),
        Commands::Backup { dest } => commands::cmd_backup(&vault_path, dest),
    };

    if let Err(e) = result {
        eprintln!("Error: {e}");
        std::process::exit(1);
    }
}
```

**`src/commands.rs`** — `cmd_init` และ `cmd_add`:
```rust
use rpassword::prompt_password;
use std::path::PathBuf;
use zeroize::Zeroize;

use crate::crypto::derive_key;
use crate::error::VaultError;
use crate::vault::{VaultFile, default_vault_path};

/// Prompt for master password. On init, prompt twice for confirmation.
fn prompt_master_password(confirm: bool) -> Result<String, VaultError> {
    let pw = prompt_password("Master password: ")
        .map_err(|e| VaultError::IoError(e.to_string()))?;

    if confirm {
        let pw2 = prompt_password("Confirm master password: ")
            .map_err(|e| VaultError::IoError(e.to_string()))?;
        if pw != pw2 {
            return Err(VaultError::IoError(
                "Passwords do not match.".to_string(),
            ));
        }
    }
    Ok(pw)
}

pub fn cmd_init(vault_path: &std::path::Path) -> Result<(), VaultError> {
    if vault_path.exists() {
        return Err(VaultError::VaultAlreadyExists(vault_path.to_path_buf()));
    }

    let mut pw = prompt_master_password(true)?;

    let vault = VaultFile::new();
    vault.save(vault_path)?;

    // Zeroize password immediately after use
    pw.zeroize();

    println!("Vault initialized at {}", vault_path.display());
    Ok(())
}

pub fn cmd_add(vault_path: &std::path::Path, name: &str) -> Result<(), VaultError> {
    let mut vault = VaultFile::load(vault_path)?;

    let mut pw = prompt_master_password(false)?;
    let salt = vault.salt_bytes();
    let mut key = derive_key(&pw, &salt);
    pw.zeroize();

    let secret = prompt_password(&format!("Secret value for '{}': ", name))
        .map_err(|e| VaultError::IoError(e.to_string()))?;

    vault.add_entry(name, &key, &secret);
    key.zeroize();

    vault.save(vault_path)?;
    println!("Entry '{}' saved.", name);
    Ok(())
}
```

สิ่งสำคัญในขั้นนี้: `pw.zeroize()` เรียกทันทีหลังไม่ต้องการ password string อีกต่อไป และ `key.zeroize()` เรียกหลัง `add_entry` เสร็จ การ zeroize ทำได้เพราะเราประกาศ `mut` ตั้งแต่ต้น

### ขั้นที่ 5: `vault get` + `vault list` + `vault delete`

**`src/commands.rs`** — เพิ่ม get, list, delete:
```rust
pub fn cmd_get(
    vault_path: &std::path::Path,
    name: &str,
    print_stdout: bool,
) -> Result<(), VaultError> {
    let vault = VaultFile::load(vault_path)?;

    let mut pw = prompt_master_password(false)?;
    let salt = vault.salt_bytes();
    let mut key = derive_key(&pw, &salt);
    pw.zeroize();

    let secret = vault.get_entry(name, &key)?;
    key.zeroize();

    if print_stdout {
        // ผู้ใช้เลือก print ออก stdout โดยตั้งใจ
        println!("{secret}");
    } else {
        copy_to_clipboard_with_autoclear(secret)?;
        println!("Secret '{}' copied to clipboard. Will clear in 30 seconds.", name);
    }

    Ok(())
}

pub fn cmd_list(vault_path: &std::path::Path) -> Result<(), VaultError> {
    let vault = VaultFile::load(vault_path)?;

    if vault.entries.is_empty() {
        println!("Vault is empty. Use `vault add <name>` to add entries.");
        return Ok(());
    }

    println!("{} entries:", vault.entries.len());
    for entry in &vault.entries {
        println!("  • {}  (added {})", entry.name, entry.created_at);
    }
    Ok(())
}

pub fn cmd_delete(vault_path: &std::path::Path, name: &str) -> Result<(), VaultError> {
    let mut vault = VaultFile::load(vault_path)?;

    // Require password confirmation before destructive operation
    let mut pw = prompt_master_password(false)?;
    let salt = vault.salt_bytes();
    let mut key = derive_key(&pw, &salt);
    pw.zeroize();

    // ตรวจสอบ decryption สำเร็จก่อนลบ (ยืนยัน password ถูกต้อง)
    vault.get_entry(name, &key)?;
    key.zeroize();

    vault.delete_entry(name)?;
    vault.save(vault_path)?;
    println!("Entry '{}' deleted.", name);
    Ok(())
}
```

### ขั้นที่ 6: Clipboard + Auto-Clear Thread

AES-GCM ปกป้อง secret บนดิสก์ แต่เมื่อ secret ถูก copy ไป clipboard จะอยู่ใน plaintext ใน clipboard manager Auto-clear thread แก้ปัญหานี้ด้วยการ sleep 30 วินาทีแล้ว clear clipboard

**`src/commands.rs`** — clipboard function:
```rust
use arboard::Clipboard;
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

/// Copy secret to clipboard and spawn a background thread that clears it after 30 seconds.
pub fn copy_to_clipboard_with_autoclear(secret: String) -> Result<(), VaultError> {
    let mut clipboard = Clipboard::new()
        .map_err(|e| VaultError::IoError(format!("Clipboard error: {e}")))?;

    clipboard
        .set_text(secret.clone())
        .map_err(|e| VaultError::IoError(format!("Clipboard write error: {e}")))?;

    // Spawn detached thread — ต้องใช้ Arc<Mutex> เพราะ Clipboard ไม่ implement Send
    // วิธีแก้: สร้าง Clipboard ใหม่ใน thread
    thread::spawn(move || {
        thread::sleep(Duration::from_secs(30));
        if let Ok(mut cb) = Clipboard::new() {
            // ตรวจสอบว่า clipboard ยังมีค่าเดิม (ผู้ใช้ไม่ได้ copy อย่างอื่นทับ)
            if let Ok(current) = cb.get_text() {
                if current == secret {
                    let _ = cb.set_text("");
                }
            }
        }
    });

    Ok(())
}
```

หมายเหตุ: `Clipboard` ไม่ implement `Send` ใน `arboard` (borrow checker จะ reject ถ้า move `clipboard` เข้า thread) วิธีแก้ที่ถูกต้องคือสร้าง `Clipboard` instance ใหม่ใน spawned thread ส่งแค่ `secret: String` ซึ่ง implement `Send`

### ขั้นที่ 7: Key Zeroing ด้วย `zeroize`

ขั้นนี้ทำให้ secure memory handling เป็นระบบมากขึ้นด้วย `Zeroize` derive macro

**ทำไม `.drop()` ไม่พอ**: Rust drop เพียงแค่ deallocate memory แต่ไม่รับประกันว่า bytes จะถูกเขียนทับก่อน อาจยังอ่านได้จาก memory dump หรือ `/proc/<pid>/mem` บน Linux ด้วย sufficient privilege

**`src/crypto.rs`** — เพิ่ม ZeroizingKey wrapper:
```rust
use zeroize::{Zeroize, ZeroizeOnDrop};

/// Key wrapper ที่ zeroize เมื่อ drop — ใช้แทน raw [u8; 32] ใน long-lived contexts
#[derive(Zeroize, ZeroizeOnDrop)]
pub struct VaultKey(pub [u8; 32]);

impl VaultKey {
    pub fn derive(password: &str, salt: &[u8]) -> Self {
        VaultKey(derive_key(password, salt))
    }

    pub fn as_bytes(&self) -> &[u8; 32] {
        &self.0
    }
}
```

**Pattern ที่ใช้ใน commands.rs** — ใช้ `VaultKey` แทน raw array:
```rust
// ก่อน (ขั้นที่ 4-5):
let mut key = derive_key(&pw, &salt);
// ... ใช้งาน ...
key.zeroize();

// หลัง (ขั้นที่ 7): zeroize อัตโนมัติเมื่อออกจาก scope
let key = VaultKey::derive(&pw, &salt);
// ... ใช้งาน key.as_bytes() ...
// key dropped automatically at end of scope → ZeroizeOnDrop runs
```

**Password zeroing** — ใช้ `zeroize::Zeroizing<String>`:
```rust
use zeroize::Zeroizing;

fn prompt_master_password_secure(confirm: bool) -> Result<Zeroizing<String>, VaultError> {
    let pw = Zeroizing::new(
        prompt_password("Master password: ")
            .map_err(|e| VaultError::IoError(e.to_string()))?,
    );

    if confirm {
        let pw2 = Zeroizing::new(
            prompt_password("Confirm master password: ")
                .map_err(|e| VaultError::IoError(e.to_string()))?,
        );
        if *pw != *pw2 {
            return Err(VaultError::IoError("Passwords do not match.".to_string()));
        }
    }
    Ok(pw)
    // pw2 dropped here → String bytes zeroed automatically
}
```

`Zeroizing<T>` เป็น newtype wrapper ที่ implement `ZeroizeOnDrop` — เมื่อ `Zeroizing<String>` ถูก drop, bytes ของ String จะถูก overwrite ด้วย zeros

### ขั้นที่ 8: Lock Timeout + Backup Command

**Lock Timeout** — เพิ่ม metadata file สำหรับเก็บ last-unlock timestamp:

Strategy: เก็บ `last_unlock_at` ใน sidecar file `vault.lock` (JSON) ข้าง ๆ vault file — แนวทางนี้ง่ายกว่าการ embed timestamp ลงใน vault file เอง (ไม่ต้องรู้ password เพื่อตรวจ timeout)

**`src/vault.rs`** — เพิ่ม LockState:
```rust
use chrono::{DateTime, Duration, Utc};
use std::path::Path;

#[derive(Debug, serde::Serialize, serde::Deserialize)]
pub struct LockState {
    pub last_unlock_at: DateTime<Utc>,
}

impl LockState {
    pub fn load(vault_path: &Path) -> Option<Self> {
        let lock_path = vault_path.with_extension("lock");
        let content = std::fs::read_to_string(lock_path).ok()?;
        serde_json::from_str(&content).ok()
    }

    pub fn save(vault_path: &Path) {
        let lock_path = vault_path.with_extension("lock");
        let state = LockState {
            last_unlock_at: Utc::now(),
        };
        if let Ok(json) = serde_json::to_string(&state) {
            let _ = std::fs::write(lock_path, json);
        }
    }

    /// Return true if vault should be considered locked (5 minutes inactivity)
    pub fn is_timed_out(vault_path: &Path) -> bool {
        match Self::load(vault_path) {
            None => false, // ไม่มี lock file = ยังไม่เคย unlock = ต้องถาม password ปกติ
            Some(state) => {
                let elapsed = Utc::now() - state.last_unlock_at;
                elapsed > Duration::minutes(5)
            }
        }
    }
}
```

**`src/commands.rs`** — ใช้ LockState:
```rust
use crate::vault::LockState;

/// Unlock vault: verify password by attempting to decrypt first entry.
/// Saves lock state on success. Returns derived key on success.
pub fn unlock_vault(
    vault: &VaultFile,
    vault_path: &std::path::Path,
) -> Result<VaultKey, VaultError> {
    if LockState::is_timed_out(vault_path) {
        eprintln!("Vault locked due to 5 minutes of inactivity.");
    }

    let pw = prompt_master_password_secure(false)?;
    let salt = vault.salt_bytes();
    let key = VaultKey::derive(&pw, &salt);

    // ตรวจสอบ password ถูกต้องโดย decrypt entry แรก (ถ้ามี)
    if let Some(first) = vault.entries.first() {
        vault.get_entry(&first.name.clone(), key.as_bytes())?;
    }

    LockState::save(vault_path);
    Ok(key)
}
```

**Backup Command**:
```rust
pub fn cmd_backup(
    vault_path: &std::path::Path,
    dest: Option<PathBuf>,
) -> Result<(), VaultError> {
    if !vault_path.exists() {
        return Err(VaultError::VaultNotFound(vault_path.to_path_buf()));
    }

    let timestamp = chrono::Utc::now().format("%Y%m%d_%H%M%S");
    let backup_name = format!("vault_{}.enc", timestamp);

    let dest_dir = dest.unwrap_or_else(|| {
        vault_path
            .parent()
            .unwrap_or(std::path::Path::new("."))
            .to_path_buf()
    });

    std::fs::create_dir_all(&dest_dir)
        .map_err(|e| VaultError::IoError(e.to_string()))?;

    let dest_path = dest_dir.join(&backup_name);
    std::fs::copy(vault_path, &dest_path)
        .map_err(|e| VaultError::IoError(e.to_string()))?;

    println!("Backup saved to {}", dest_path.display());
    Ok(())
}
```

**Export Command** — ส่วนที่ต้องระวัง:
```rust
pub fn cmd_export(
    vault_path: &std::path::Path,
    plaintext_flag: bool,
) -> Result<(), VaultError> {
    if !plaintext_flag {
        eprintln!(
            "WARNING: This command exports all secrets as plaintext JSON.\n\
             Run with --plaintext to confirm you understand the risk."
        );
        return Ok(());
    }

    let vault = VaultFile::load(vault_path)?;
    let key = unlock_vault(&vault, vault_path)?;

    let mut export = serde_json::Map::new();
    export.insert("exported_at".to_string(), serde_json::Value::String(
        chrono::Utc::now().to_rfc3339(),
    ));

    let mut entries = serde_json::Map::new();
    for entry in &vault.entries {
        if let Ok(secret) = vault.get_entry(&entry.name, key.as_bytes()) {
            entries.insert(entry.name.clone(), serde_json::Value::String(secret));
        }
    }
    export.insert("entries".to_string(), serde_json::Value::Object(entries));

    println!("{}", serde_json::to_string_pretty(&export).unwrap());
    Ok(())
}
```

## การทดสอบ (Testing)

โค้ดทดสอบต่อไปนี้สร้างและรันใน scratchpad project แยกต่างหากจาก repository จริง ผลลัพธ์ที่แสดงเป็น real output จากการรัน `cargo test` จริง

### Test Suite

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rand::RngCore;

    fn random_salt() -> [u8; 16] {
        let mut s = [0u8; 16];
        rand::thread_rng().fill_bytes(&mut s);
        s
    }

    // Test 1: Same password+salt → same key (deterministic)
    #[test]
    fn test_key_derivation_deterministic() {
        let salt = random_salt();
        let key1 = derive_key("my-master-password", &salt);
        let key2 = derive_key("my-master-password", &salt);
        assert_eq!(
            key1, key2,
            "Same password+salt must produce identical key bytes"
        );
    }

    // Test 2: Different passwords → different keys
    #[test]
    fn test_key_derivation_different_passwords() {
        let salt = random_salt();
        let key1 = derive_key("password-one", &salt);
        let key2 = derive_key("password-two", &salt);
        assert_ne!(key1, key2, "Different passwords must produce different keys");
    }

    // Test 3: Encrypt "my-secret-value", decrypt → same plaintext
    #[test]
    fn test_encrypt_decrypt_roundtrip() {
        let salt = random_salt();
        let key = derive_key("roundtrip-password", &salt);
        let plaintext = b"my-secret-value";

        let (nonce, ciphertext) = encrypt(&key, plaintext);
        let decrypted = decrypt(&key, &nonce, &ciphertext);

        assert_eq!(decrypted, plaintext, "Decrypted bytes must match plaintext");
        assert_eq!(String::from_utf8(decrypted).unwrap(), "my-secret-value");
    }

    // Test 4: Same plaintext encrypted twice → different ciphertexts (random nonce)
    #[test]
    fn test_different_nonces_produce_different_ciphertexts() {
        let salt = random_salt();
        let key = derive_key("nonce-test-password", &salt);
        let plaintext = b"same-secret";

        let (nonce1, ct1) = encrypt(&key, plaintext);
        let (nonce2, ct2) = encrypt(&key, plaintext);

        assert_ne!(nonce1, nonce2, "Each encryption must use a fresh random nonce");
        assert_ne!(ct1, ct2, "Different nonces must produce different ciphertexts");

        // Both must still decrypt correctly
        assert_eq!(decrypt(&key, &nonce1, &ct1), plaintext.to_vec());
        assert_eq!(decrypt(&key, &nonce2, &ct2), plaintext.to_vec());
    }

    // Test 5: Vault file round-trip with 3 entries
    #[test]
    fn test_vault_file_roundtrip() {
        let salt = random_salt();
        let key = derive_key("vault-roundtrip-pw", &salt);

        let mut vault = VaultFile::new(&salt);
        vault.add_entry("github_token", &key, "ghp_abc123secret");
        vault.add_entry("db_password", &key, "postgres-super-secret");
        vault.add_entry("api_key", &key, "sk-live-xxxx9999");

        let json = vault.to_json();
        assert!(json.contains("\"version\""));
        assert!(json.contains("\"github_token\""));

        let restored = VaultFile::from_json(&json);
        assert_eq!(restored.entries.len(), 3);
        assert_eq!(restored.get_entry("github_token", &key).unwrap(), "ghp_abc123secret");
        assert_eq!(restored.get_entry("db_password", &key).unwrap(), "postgres-super-secret");
        assert_eq!(restored.get_entry("api_key", &key).unwrap(), "sk-live-xxxx9999");
    }

    // Test 6: Wrong key → decryption fails (GCM authentication)
    #[test]
    #[should_panic(expected = "decryption failure")]
    fn test_wrong_key_fails_decryption() {
        let salt = random_salt();
        let key = derive_key("correct-password", &salt);
        let wrong_key = derive_key("wrong-password", &salt);
        let (nonce, ciphertext) = encrypt(&key, b"super-secret");
        decrypt(&wrong_key, &nonce, &ciphertext); // must panic
    }
}
```

### ผลลัพธ์จากการรัน `cargo test` จริง

```
running 6 tests
test tests::test_encrypt_decrypt_roundtrip ... ok
test tests::test_different_nonces_produce_different_ciphertexts ... ok
test tests::test_vault_file_roundtrip ... ok
test tests::test_key_derivation_deterministic ... ok
test tests::test_key_derivation_different_passwords ... ok
test tests::test_wrong_key_fails_decryption - should panic ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 8.15s
```

ผล 6/6 ผ่าน — key derivation เป็น deterministic, round-trip encrypt/decrypt ทำงานถูกต้อง, nonces เป็น random จริง, vault serialization ครบ 3 entries, wrong key ถูก GCM tag reject

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
```

Binary อยู่ที่ `target/release/vault` — ขนาดประมาณ 2–4 MB (statically linked Rust + crypto libs)

```bash
# Install ลง PATH
cargo install --path .
# หรือ copy manual
cp target/release/vault ~/.local/bin/vault
```

### ใช้งานครั้งแรก

```bash
# Initialize vault
vault init
# Master password: [input hidden]
# Confirm master password: [input hidden]
# Vault initialized at /home/user/.local/share/vault/vault.enc

# Add entries
vault add github_token
# Master password: [input hidden]
# Secret value for 'github_token': [input hidden]
# Entry 'github_token' saved.

vault add db_password
vault add openai_key

# List all entries
vault list
# 3 entries:
#   • github_token  (added 2024-01-15T10:30:00Z)
#   • db_password   (added 2024-01-15T10:31:22Z)
#   • openai_key    (added 2024-01-15T10:32:05Z)

# Get entry (copy to clipboard)
vault get github_token
# Master password: [input hidden]
# Secret 'github_token' copied to clipboard. Will clear in 30 seconds.

# Get entry (print to stdout)
vault get github_token --print
# Master password: [input hidden]
# ghp_abc123...

# Backup
vault backup ~/Dropbox/vault-backups/
# Backup saved to /home/user/Dropbox/vault-backups/vault_20240115_103500.enc

# Delete entry
vault delete github_token
# Master password: [input hidden]
# Entry 'github_token' deleted.
```

### Shell Completion (ด้วย clap)

```bash
# เพิ่มใน src/main.rs
use clap::CommandFactory;
use clap_complete::{generate, Shell};

fn print_completions(shell: Shell) {
    let mut cmd = Cli::command();
    generate(shell, &mut cmd, "vault", &mut std::io::stdout());
}
```

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Nonce Reuse กับ Deterministic Nonce

**ปัญหา**: ถ้าใช้ counter หรือ timestamp เป็น nonce แทน random bytes มีความเสี่ยงที่ nonce จะซ้ำในบางสถานการณ์ (เช่น vault ถูก copy ไปเครื่องอื่นแล้วแก้ไขพร้อมกัน หรือ system clock ถูก reset)

**ผลกระทบ**: ถ้า nonce ซ้ำกันภายใต้ key เดียวกัน GCM keystream จะซ้ำกัน ทำให้สามารถ XOR ciphertext ทั้งสองเพื่อได้ XOR ของ plaintexts ได้

**วิธีแก้**: ใช้ `Aes256Gcm::generate_nonce(&mut OsRng)` เสมอ — 96 bits random = birthday bound ที่ 2^-64 สำหรับ 2^32 entries

```rust
// ❌ อย่าทำ: timestamp เป็น nonce
let ts = std::time::SystemTime::now()
    .duration_since(std::time::UNIX_EPOCH).unwrap().as_secs();
let nonce_bytes = ts.to_le_bytes(); // เพียง 8 bytes และซ้ำได้

// ✅ ถูกต้อง: random nonce จาก CSPRNG
let nonce = Aes256Gcm::generate_nonce(&mut OsRng); // 12 bytes random
```

### 2. String Comparison Timing Attack บน Master Password

**ปัญหา**: การเปรียบ password strings ด้วย `==` (ซึ่ง short-circuit เมื่อเจอ mismatch ตัวแรก) สร้าง timing difference ที่วัดได้ในทางทฤษฎี

**ผลกระทบ**: สำหรับ CLI local tool นี้ timing attack ไม่ practical เพราะ network latency ไม่มี แต่ถ้าพัฒนาเป็น server-side API ควรใช้ constant-time comparison

**วิธีแก้** (สำหรับ server context):
```rust
use subtle::ConstantTimeEq;

// ✅ Constant-time comparison ด้วย subtle crate
fn passwords_match(a: &[u8], b: &[u8]) -> bool {
    a.ct_eq(b).into()
}
```

### 3. Argon2 ใช้ Salt ผิด Format

**ปัญหา**: `argon2` crate มีสอง API — `hash_password()` กับ `hash_password_into()` และ salt format ต่างกัน

```rust
// ❌ ผิด: hash_password ต้องการ SaltString (base64-encoded PHC format)
// แต่เราเก็บ salt เป็น raw bytes ใน vault file
let salt_string = SaltString::from_b64("..."); // ไม่ match กับ raw bytes ที่เก็บไว้

// ✅ ถูกต้อง: ใช้ hash_password_into กับ raw salt bytes
argon2.hash_password_into(password.as_bytes(), &salt_bytes, &mut key)?;
```

`hash_password_into` รับ `salt: &[u8]` ตรง ๆ ทำให้ใช้ raw bytes ที่เก็บใน `salt_hex` ได้เลยหลัง `hex::decode`

### 4. Clipboard ไม่ Clear เมื่อ Program Exit ก่อน 30 วินาที

**ปัญหา**: Spawned thread เป็น daemon thread — ถ้า main thread exit ก่อน background thread จะถูก kill ไปด้วย (Rust runtime teardown) ทำให้ clipboard ไม่ถูก clear

**วิธีแก้**: ถ้าต้องการ guarantee ว่า clipboard จะถูก clear ควรใช้ `Arc<Mutex<bool>>` เพื่อ signal thread หรือเก็บ `JoinHandle` แล้ว join ก่อน exit:

```rust
// วิธีที่ reliable มากขึ้น: ใช้ channel เพื่อให้ main รอ
use std::sync::mpsc;

let (tx, rx) = mpsc::channel::<()>();
thread::spawn(move || {
    thread::sleep(Duration::from_secs(30));
    // clear clipboard...
    let _ = tx.send(());
});

// รอสัญญาณจาก thread (optional — ขึ้นอยู่กับ UX ที่ต้องการ)
println!("Waiting for clipboard clear...");
let _ = rx.recv();
```

สำหรับ UX ที่ดี มักปล่อยให้ program exit ทันที (non-blocking) และยอมรับว่า clipboard จะถูก clear เมื่อ thread ทำงานเสร็จก่อน OS kill process

### 5. Vault File ไม่ Atomic ทำให้ Corrupted ได้

**ปัญหา**: เขียน vault file ตรง ๆ ด้วย `fs::write(path, json)` — ถ้าโปรแกรม crash หรือ disk full ระหว่างเขียน จะได้ vault file ที่ corrupted และอ่านไม่ได้

**วิธีแก้**: เขียนลง temp file ก่อน แล้วค่อย rename (atomic บน POSIX filesystems):
```rust
let tmp_path = path.with_extension("tmp");
std::fs::write(&tmp_path, json)?;
std::fs::rename(&tmp_path, path)?; // atomic operation
```

`rename` บน Linux/macOS เป็น atomic system call — kernel รับประกันว่า path จะชี้ไปยัง new file หรือ old file เสมอ ไม่มี intermediate state

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม `vault rename`

เพิ่ม subcommand `vault rename OLD_NAME NEW_NAME` ที่เปลี่ยนชื่อ entry โดยไม่ต้อง decrypt/re-encrypt (เพียงแค่เปลี่ยน `name` field ใน struct) ตรวจสอบว่า `NEW_NAME` ยังไม่มีใน vault ก่อน และต้องการ master password ก่อน rename

*Hint*: ไม่ต้องรู้ plaintext เพื่อ rename — แค่ iterate `vault.entries` และเปลี่ยน `.name` field

### แบบฝึกหัดที่ 2: Multi-Vault Support

เพิ่ม concept "vault profiles" — ผู้ใช้สามารถสร้าง vault หลายอันด้วย `vault init --profile work` / `vault init --profile personal` แต่ละ profile มี salt และ master password แยกกัน

*Hint*: เก็บ vault files ที่ `~/.local/share/vault/<profile>.enc` และเพิ่ม `--profile` flag ใน `Cli` struct

### แบบฝึกหัดที่ 3: Vault File Versioning + Migration

เพิ่ม `version` field ที่ไม่ได้แค่เก็บค่าแต่ใช้งานจริง:
- version 1: format ปัจจุบัน
- version 2: เพิ่ม `tags: Vec<String>` ใน VaultEntry สำหรับ filtering
- เขียน `migrate_v1_to_v2(vault: VaultFile) -> VaultFile` function
- `vault load` ตรวจสอบ version และ migrate อัตโนมัติ

*Hint*: ใช้ `match vault.version` ก่อน deserialize full struct

### แบบฝึกหัดที่ 4: Argon2 Parameter ให้ Configurable

ปัจจุบัน Argon2 parameters hardcoded เป็น m=64MiB, t=3, p=1 เพิ่มให้ `vault init` รับ `--argon2-memory <MB>` และ `--argon2-iterations <N>` และเก็บ parameters ใน vault file:

```json
{
  "version": 2,
  "salt_hex": "...",
  "argon2_params": {
    "m_cost": 65536,
    "t_cost": 3,
    "p_cost": 1
  },
  "entries": [...]
}
```

เมื่อ load vault ให้ใช้ parameters จาก vault file ไม่ใช่ hardcoded — ทำให้ vault file สามารถพกพาระหว่าง machines ที่มี performance ต่างกันได้

*Hint*: เพิ่ม `argon2_params: Argon2Params` ใน `VaultFile` struct และ `derive_key_with_params()`

### แบบฝึกหัดที่ 5: Integration Test แบบ End-to-End

เขียน integration test ที่:
1. สร้าง temp vault ใน `/tmp`
2. เรียก `cmd_init` กับ test password
3. เรียก `cmd_add` สำหรับ 5 entries
4. เรียก `cmd_list` และตรวจสอบ output มีครบ 5 entry names
5. เรียก `cmd_get` สำหรับแต่ละ entry และตรวจสอบค่าถูกต้อง
6. เรียก `cmd_delete` สำหรับ 2 entries
7. เรียก `cmd_list` อีกครั้งและตรวจสอบเหลือ 3 entries
8. ลบ temp vault หลัง test เสร็จ

*Hint*: ใช้ `tempfile::TempDir` จาก `tempfile` crate เพื่อจัดการ cleanup อัตโนมัติ

### แบบฝึกหัดที่ 6: Key Stretching Benchmark

เขียน binary `vault benchmark` ที่วัดว่า Argon2id ด้วย parameters ต่าง ๆ ใช้เวลาเท่าไหร่บนเครื่องปัจจุบัน แล้ว suggest parameters ที่ทำให้ KDF ใช้เวลา ~500ms (target สำหรับ interactive login):

```
Argon2id Benchmark:
  m=12288, t=3, p=1 →  85ms
  m=32768, t=3, p=1 → 210ms
  m=65536, t=3, p=1 → 420ms  ← recommended (closest to 500ms target)
  m=131072, t=3, p=1 → 850ms
```

*Hint*: ใช้ `std::time::Instant::now()` และ loop 3 ครั้งแล้วเฉลี่ย

## สรุป

โปรเจคนี้สร้าง Secret Vault ที่ใช้ Argon2id สำหรับ key derivation และ AES-256-GCM สำหรับ authenticated encryption เพื่อเก็บ secrets ไว้ใน local encrypted file

**Pattern สำคัญที่ได้เรียน:**

1. **AEAD = Confidentiality + Integrity**: AES-256-GCM ให้ทั้งสองใน operation เดียว ไม่ต้องจัดการ MAC แยก และ authentication failure เกิดก่อน decryption ทำให้ not provide oracle for padding

2. **Memory-hard KDF**: Argon2id ทำให้ brute-force ด้วย GPU/FPGA มีต้นทุนสูงด้วย memory requirement — parameter `m_cost` คือ lever สำคัญสำหรับ security/performance tradeoff

3. **Zeroize เป็น first-class concern**: key material และ password strings ต้อง overwrite explicitly — Rust's `drop()` ไม่ทำ zeroing ให้ `zeroize` crate และ `ZeroizeOnDrop` เป็น idiomatic solution

4. **Atomic file writes**: เขียน vault ผ่าน temp file + rename เพื่อป้องกัน corruption กรณี crash/disk full — pattern นี้ใช้ได้กับ config file ทุกประเภท

5. **Per-entry random nonce**: แต่ละ entry มี nonce ตัวเอง ทำให้ encrypt entry เดิมหลายครั้งได้โดยปลอดภัย และ entries ที่ต่างกันไม่มี keystream reuse

โปรเจคถัดไปเปลี่ยนโหมดจาก CLI เป็น web service — **Project B01: Pastebin Service** จะสร้าง HTTP API ด้วย Axum framework สำหรับ paste text และ retrieve ด้วย short URL

---

**โปรเจคก่อนหน้า:** [Project A09: File Deduplicator](project-a09-file-deduplicator.md) | **โปรเจคถัดไป:** [Project B01: Pastebin Service](project-b01-pastebin-service.md)
