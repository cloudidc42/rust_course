# Project C08: Data Migration Framework

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 6 ชั่วโมง

## ภาพรวมโปรเจค

**Data Migration Framework** คือระบบจัดการ database schema changes แบบ code-first ที่เขียน migration เป็น Rust struct แทนการใช้ SQL ไฟล์แยกต่างหาก ทำให้ได้ประโยชน์จาก type-safety ของ Rust เต็มที่ เช่น compile-time check, refactoring support, และ IDE integration

ใน production ปัญหาคลาสสิกที่ทีมพบบ่อยคือ:
- Migration ถูกแก้ไขหลัง deploy แล้ว (ไม่รู้ว่า DB จริงกับ code ต่างกัน)
- ทีมหลาย branch รัน migration ชนกัน ทำให้ DB อยู่ในสถานะผิดปกติ
- Rollback ทำไม่ได้เพราะไม่มี `down()` ที่เขียนไว้
- ไม่มีวิธีดูว่า migration ไหน applied ไปแล้ว เมื่อไหร่ ใช้เวลาเท่าไหร่

Framework นี้แก้ปัญหาเหล่านี้ด้วย:
- **Migration เป็น Rust trait** — compile ไม่ผ่านถ้า implementation ไม่ครบ
- **Checksum tracking** — detect ทันทีถ้ามีคนแก้ migration หลัง apply
- **Individual transactions** — migration ล้มเหลวจะ rollback เฉพาะตัวนั้น ไม่กระทบตัวอื่น
- **Dry-run mode** — ดูก่อนว่าจะทำอะไรโดยไม่แตะ DB จริง
- **Seed data ที่ environment-aware** — run seeds เฉพาะใน dev/staging ไม่ run ใน production

---

## สิ่งที่จะได้เรียนรู้

- **Trait objects (`dyn Trait`)** — ใช้ trait เพื่อสร้าง polymorphic migration system
- **Async trait methods** — เทคนิคการ return `Pin<Box<dyn Future>>` สำหรับ async ใน trait
- **`sqlx` AnyPool** — เชื่อมต่อทั้ง PostgreSQL และ SQLite ด้วย code เดียวกัน
- **SHA-256 checksum** — ใช้ `sha2` + `hex` เพื่อ fingerprint source code
- **Transaction management** — BEGIN/COMMIT/ROLLBACK อย่างถูกต้องใน async Rust
- **Registry pattern** — รวบรวมและเรียงลำดับ migrations แบบ auto-discover
- **clap 4 derive API** — สร้าง CLI ที่มี subcommands และ env variable support
- **`colored` crate** — ทำ terminal output สวยงาม อ่านง่าย

---

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 46: async/await พื้นฐาน และ `tokio` runtime
- จาก Part 52: Trait objects, `dyn Trait`, dynamic dispatch
- จาก Part 58: `sqlx` พื้นฐาน — query, execute, pool
- จาก Part 63: Error handling ด้วย `anyhow` และ `?` operator
- จาก Part 71: `Arc<T>` และ shared ownership
- จาก Part 96: CLI tools ด้วย `clap` derive API

---

## โครงสร้างโปรเจค (Project Layout)

```
data-migration/
├── src/
│   ├── main.rs                  # CLI entry point (clap)
│   ├── migration.rs             # Migration trait + types
│   ├── registry.rs              # MigrationRegistry
│   ├── db.rs                    # DB operations (__migrations table)
│   ├── runner.rs                # run_up, run_down, show_status
│   ├── seed.rs                  # Seed data logic
│   └── migrations/
│       ├── mod.rs               # รวบรวม migrations ทั้งหมด
│       ├── m20240101_000001_create_users.rs
│       ├── m20240102_000001_create_posts.rs
│       └── m20240103_000001_add_email_index.rs
├── Cargo.toml
└── README.md
```

**ชื่อไฟล์ migration** ใช้รูปแบบ `m{YYYYMMDD}_{NNNNNN}_{name}.rs` เพื่อให้ compiler เรียงได้ถูกต้องตาม timestamp และ sequence number

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
CLI (clap)
    │
    ▼
MigrationRegistry ─── register migrations ──► sorted Vec<Arc<dyn Migration>>
    │
    ▼
Runner (run_up / run_down / show_status)
    │
    ├── DB: ensure __migrations table
    ├── DB: load applied records
    ├── compute pending / reverse list
    │
    └── for each migration:
            ├── BEGIN TRANSACTION
            ├── migration.up(conn)  OR  migration.down(conn)
            ├── COMMIT  (or ROLLBACK on error)
            └── INSERT / DELETE from __migrations
```

### __migrations Table Schema

```sql
CREATE TABLE IF NOT EXISTS __migrations (
    id          TEXT     NOT NULL PRIMARY KEY,  -- "20240101_000001_create_users"
    applied_at  TEXT     NOT NULL,              -- RFC3339 timestamp
    checksum    TEXT     NOT NULL,              -- SHA-256 hex ของ source code
    duration_ms INTEGER  NOT NULL DEFAULT 0     -- เวลาที่ใช้รัน (milliseconds)
);
```

### ทำไมถึงเลือก Design นี้

| Design Decision | เหตุผล |
|---|---|
| Migration เป็น Rust struct | Type-safe, refactorable, IDE support |
| `AnyPool` แทน `PgPool` | รองรับทั้ง SQLite (test/dev) และ PostgreSQL (prod) |
| Individual transaction | Migration หนึ่งล้มเหลวไม่กระทบตัวอื่น |
| Checksum ของ source code | Detect การแก้ไข migration หลัง apply |
| `Arc<dyn Migration>` | Shared ownership, thread-safe, dynamic dispatch |
| `Pin<Box<dyn Future>>` | ทาง standard สำหรับ async trait methods ก่อน `async_trait` |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ตั้งค่าโปรเจคและ Cargo.toml

สร้างโปรเจคใหม่พร้อม dependencies ทั้งหมด:

```bash
cargo new data-migration --bin
cd data-migration
```

**`Cargo.toml`:**

```toml
[package]
name = "data-migration"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "migrate"
path = "src/main.rs"

[features]
default = ["sqlite"]
postgres = ["sqlx/postgres"]
sqlite   = ["sqlx/sqlite"]

[dependencies]
sqlx     = { version = "0.7", features = ["runtime-tokio-rustls", "chrono", "sqlite", "any"] }
sha2     = "0.10"
hex      = "0.4"
serde    = { version = "1", features = ["derive"] }
serde_json = "1"
tokio    = { version = "1", features = ["full"] }
chrono   = { version = "0.4", features = ["serde"] }
clap     = { version = "4", features = ["derive", "env"] }
colored  = "2"
anyhow   = "1"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

**หมายเหตุ feature flags:**
- `--features postgres` — เปิดใช้ PostgreSQL driver
- `--features sqlite` (default) — ใช้ SQLite สำหรับ development/testing
- `sqlx/any` — เปิด `AnyPool` ที่รองรับ multiple backends

---

### ขั้นที่ 2: Migration Trait และ Types

**`src/migration.rs`** — หัวใจของ framework:

```rust
use anyhow::Result;
use sqlx::AnyConnection;

/// Migration trait — ทุก migration ต้อง implement
pub trait Migration: Send + Sync {
    /// ID เช่น "20240101_000001_create_users"
    fn id(&self) -> &str;

    /// Source code ของ migration (ใช้คำนวณ checksum)
    fn source(&self) -> &str;

    /// Apply migration (up)
    fn up<'a>(
        &'a self,
        conn: &'a mut AnyConnection,
    ) -> std::pin::Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>>;

    /// Revert migration (down)
    fn down<'a>(
        &'a self,
        conn: &'a mut AnyConnection,
    ) -> std::pin::Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>>;
}

/// Record ที่เก็บใน __migrations table
#[derive(Debug, Clone)]
pub struct MigrationRecord {
    pub id: String,
    pub applied_at: chrono::DateTime<chrono::Utc>,
    pub checksum: String,
    pub duration_ms: i64,
}

/// สถานะของ migration แต่ละตัว
#[derive(Debug, Clone, PartialEq)]
pub enum MigrationStatus {
    Pending,
    Applied { applied_at: chrono::DateTime<chrono::Utc>, duration_ms: i64 },
    Modified { applied_at: chrono::DateTime<chrono::Utc> },  // checksum เปลี่ยน
    Missing,  // อยู่ใน DB แต่ไม่มีไฟล์
}

/// ข้อมูลครบของ migration
#[derive(Debug, Clone)]
pub struct MigrationInfo {
    pub id: String,
    pub status: MigrationStatus,
    pub checksum: String,
}

/// คำนวณ SHA-256 checksum จาก source code
pub fn calculate_checksum(source: &str) -> String {
    use sha2::{Digest, Sha256};
    let mut hasher = Sha256::new();
    hasher.update(source.as_bytes());
    hex::encode(hasher.finalize())
}
```

**ทำไม `Pin<Box<dyn Future>>`?**

ปัญหาของ async trait methods ใน Rust คือ compiler ไม่สามารถกำหนด `Future` type ได้ที่ compile time เพราะแต่ละ impl อาจ return Future ต่างกัน วิธีแก้คือ box the future ไว้บน heap และ pin มันเพื่อป้องกัน move:

```rust
// แทนการเขียน (ยังไม่ stable ใน Rust 2021)
async fn up(&self, conn: &mut AnyConnection) -> Result<()>;

// เราเขียนแบบนี้แทน
fn up<'a>(
    &'a self,
    conn: &'a mut AnyConnection,
) -> Pin<Box<dyn Future<Output = Result<()>> + Send + 'a>>;
```

**Lifetime `'a` ใน Signature**

lifetime `'a` บอกว่า Future ที่ return มีอายุไม่เกิน `self` และ `conn` — ป้องกัน use-after-free

---

### ขั้นที่ 3: Migration Registry

**`src/registry.rs`** — ระบบ auto-sort migrations:

```rust
use crate::migration::{Migration, MigrationInfo, MigrationRecord, MigrationStatus, calculate_checksum};
use anyhow::Result;
use std::collections::HashMap;
use std::sync::Arc;

pub struct MigrationRegistry {
    migrations: Vec<Arc<dyn Migration>>,
}

impl MigrationRegistry {
    pub fn new() -> Self {
        Self { migrations: Vec::new() }
    }

    /// ลงทะเบียน migration เข้า registry (เรียงอัตโนมัติ)
    pub fn register(&mut self, migration: Arc<dyn Migration>) {
        self.migrations.push(migration);
        self.migrations.sort_by(|a, b| a.id().cmp(b.id()));
    }

    /// ลงทะเบียนหลาย migrations พร้อมกัน
    pub fn register_all(&mut self, migrations: Vec<Arc<dyn Migration>>) {
        for m in migrations {
            self.migrations.push(m);
        }
        self.migrations.sort_by(|a, b| a.id().cmp(b.id()));
    }

    /// รับ migrations ทั้งหมด (sorted)
    pub fn all(&self) -> &[Arc<dyn Migration>] {
        &self.migrations
    }

    /// หา pending migrations
    pub fn pending(
        &self,
        applied: &HashMap<String, MigrationRecord>,
    ) -> Vec<Arc<dyn Migration>> {
        self.migrations
            .iter()
            .filter(|m| !applied.contains_key(m.id()))
            .cloned()
            .collect()
    }

    /// applied migrations เรียงจากล่าสุดก่อน (สำหรับ rollback)
    pub fn applied_in_reverse(
        &self,
        applied: &HashMap<String, MigrationRecord>,
    ) -> Vec<Arc<dyn Migration>> {
        let mut result: Vec<Arc<dyn Migration>> = self
            .migrations
            .iter()
            .filter(|m| applied.contains_key(m.id()))
            .cloned()
            .collect();
        result.sort_by(|a, b| b.id().cmp(a.id())); // reverse order
        result
    }

    /// เปรียบเทียบกับ DB แล้วสร้าง MigrationInfo list
    pub fn compute_status(
        &self,
        applied: &HashMap<String, MigrationRecord>,
    ) -> Vec<MigrationInfo> {
        let mut result = Vec::new();

        for migration in &self.migrations {
            let id = migration.id().to_string();
            let checksum = calculate_checksum(migration.source());

            let status = if let Some(record) = applied.get(&id) {
                if record.checksum == checksum {
                    MigrationStatus::Applied {
                        applied_at: record.applied_at,
                        duration_ms: record.duration_ms,
                    }
                } else {
                    MigrationStatus::Modified { applied_at: record.applied_at }
                }
            } else {
                MigrationStatus::Pending
            };

            result.push(MigrationInfo { id, status, checksum });
        }

        // migrations ที่อยู่ใน DB แต่ไม่มีใน registry
        for (id, record) in applied {
            if !self.migrations.iter().any(|m| m.id() == id) {
                result.push(MigrationInfo {
                    id: id.clone(),
                    status: MigrationStatus::Missing,
                    checksum: record.checksum.clone(),
                });
            }
        }

        result.sort_by(|a, b| a.id.cmp(&b.id));
        result
    }
}
```

**เหตุผลที่ใช้ `Arc<dyn Migration>` แทน `Box<dyn Migration>`:**

`Arc` ช่วยให้ share migration object ระหว่าง registry และ runner ได้โดยไม่ต้อง clone ทั้ง struct เหมาะสำหรับ case ที่ migration object ถูกใช้ในหลายที่พร้อมกัน

---

### ขั้นที่ 4: Database Operations

**`src/db.rs`** — จัดการ `__migrations` table:

```rust
use crate::migration::MigrationRecord;
use anyhow::{Context, Result};
use sqlx::{AnyConnection, Row};
use chrono::Utc;

/// สร้าง __migrations table ถ้ายังไม่มี
pub async fn ensure_migrations_table(conn: &mut AnyConnection) -> Result<()> {
    sqlx::query(
        r#"
        CREATE TABLE IF NOT EXISTS __migrations (
            id          TEXT    NOT NULL PRIMARY KEY,
            applied_at  TEXT    NOT NULL,
            checksum    TEXT    NOT NULL,
            duration_ms INTEGER NOT NULL DEFAULT 0
        )
        "#,
    )
    .execute(&mut *conn)
    .await
    .context("ไม่สามารถสร้าง __migrations table")?;

    Ok(())
}

/// โหลด applied migrations ทั้งหมดจาก DB
pub async fn load_applied_migrations(
    conn: &mut AnyConnection,
) -> Result<std::collections::HashMap<String, MigrationRecord>> {
    let rows = sqlx::query(
        "SELECT id, applied_at, checksum, duration_ms FROM __migrations ORDER BY id"
    )
    .fetch_all(&mut *conn)
    .await
    .context("ไม่สามารถโหลด applied migrations")?;

    let mut map = std::collections::HashMap::new();
    for row in rows {
        let id: String = row.try_get("id")?;
        let applied_at_str: String = row.try_get("applied_at")?;
        let checksum: String = row.try_get("checksum")?;
        let duration_ms: i64 = row.try_get("duration_ms")?;

        let applied_at = chrono::DateTime::parse_from_rfc3339(&applied_at_str)
            .map(|dt| dt.with_timezone(&Utc))
            .unwrap_or_else(|_| Utc::now());

        map.insert(id.clone(), MigrationRecord {
            id,
            applied_at,
            checksum,
            duration_ms,
        });
    }

    Ok(map)
}

/// บันทึก migration record หลัง apply สำเร็จ
pub async fn record_migration(
    conn: &mut AnyConnection,
    id: &str,
    checksum: &str,
    duration_ms: i64,
) -> Result<()> {
    let now = Utc::now().to_rfc3339();
    sqlx::query(
        "INSERT INTO __migrations (id, applied_at, checksum, duration_ms) VALUES (?, ?, ?, ?)"
    )
    .bind(id)
    .bind(&now)
    .bind(checksum)
    .bind(duration_ms)
    .execute(&mut *conn)
    .await
    .context(format!("ไม่สามารถบันทึก migration record: {}", id))?;

    Ok(())
}

/// ลบ migration record (สำหรับ rollback)
pub async fn delete_migration_record(conn: &mut AnyConnection, id: &str) -> Result<()> {
    sqlx::query("DELETE FROM __migrations WHERE id = ?")
        .bind(id)
        .execute(&mut *conn)
        .await
        .context(format!("ไม่สามารถลบ migration record: {}", id))?;

    Ok(())
}
```

**⚠ Pitfall #1: `&mut *conn` pattern**

เวลาส่ง `AnyConnection` เข้า `sqlx::query().execute()` ต้องใช้ `&mut *conn` เพื่อ deref ก่อน:

```rust
// ❌ ผิด — borrow ไม่ถูก
sqlx::query("...").execute(conn).await?;

// ✅ ถูก — deref + re-borrow
sqlx::query("...").execute(&mut *conn).await?;
```

เหตุผล: `sqlx` ต้องการ `&mut AnyConnection` ไม่ใช่ `AnyConnection` เองหรือ `&mut &mut AnyConnection`

---

### ขั้นที่ 5: Migration Runner

**`src/runner.rs`** — logic หลักของ migrate up/down/status:

```rust
use crate::db::{delete_migration_record, ensure_migrations_table,
                load_applied_migrations, record_migration};
use crate::migration::{MigrationStatus, calculate_checksum};
use crate::registry::MigrationRegistry;
use anyhow::{bail, Context, Result};
use colored::*;
use sqlx::AnyPool;
use std::time::Instant;

pub struct UpOptions {
    pub dry_run: bool,
    pub strict: bool,  // fail ถ้า checksum เปลี่ยน
}

pub struct DownOptions {
    pub count: usize,
    pub dry_run: bool,
}

/// รัน migrate up
pub async fn run_up(
    pool: &AnyPool,
    registry: &MigrationRegistry,
    opts: &UpOptions,
) -> Result<()> {
    let mut conn = pool.acquire().await.context("ไม่สามารถเชื่อมต่อ DB")?;
    ensure_migrations_table(&mut conn).await?;
    let applied = load_applied_migrations(&mut conn).await?;
    drop(conn);

    // ตรวจ checksum ของ migrations ที่ applied แล้ว
    let status_list = registry.compute_status(&applied);
    for info in &status_list {
        if matches!(info.status, MigrationStatus::Modified { .. }) {
            let msg = format!(
                "⚠  migration '{}' ถูกแก้ไขหลังจาก apply แล้ว (checksum เปลี่ยน)",
                info.id
            );
            if opts.strict {
                bail!("{}", msg);
            } else {
                eprintln!("{}", msg.yellow());
            }
        }
    }

    let pending = registry.pending(&applied);
    if pending.is_empty() {
        println!("{}", "✓ ไม่มี migration ที่ต้อง apply".green());
        return Ok(());
    }

    println!("{}", format!("→ มี {} pending migration(s)", pending.len()).cyan());

    for migration in &pending {
        let id = migration.id();
        let checksum = calculate_checksum(migration.source());

        if opts.dry_run {
            println!("{} {} {}", "[dry-run]".yellow(), "would apply:".dimmed(), id.bold());
            let preview: String = migration.source().chars().take(80).collect();
            println!("  source preview: {}...", preview);
            continue;
        }

        println!("{} {}", "↑ Applying:".blue().bold(), id);
        let start = Instant::now();

        // รัน migration ใน individual transaction
        let result = run_migration_in_transaction(pool, &**migration).await;

        match result {
            Ok(()) => {
                let duration_ms = start.elapsed().as_millis() as i64;
                let mut conn = pool.acquire().await?;
                record_migration(&mut conn, id, &checksum, duration_ms).await?;
                println!("  {} {} ({}ms)", "✓".green(), id, duration_ms);
            }
            Err(e) => {
                eprintln!("{} {} — {}", "✗ Failed:".red().bold(), id, e);
                bail!("Migration '{}' ล้มเหลว: {}", id, e);
            }
        }
    }

    println!("{}", "✓ Migration เสร็จสิ้น".green().bold());
    Ok(())
}

/// รัน migration หนึ่งตัวภายใน transaction แยก
async fn run_migration_in_transaction(
    pool: &AnyPool,
    migration: &dyn crate::migration::Migration,
) -> Result<()> {
    let mut conn = pool.acquire().await?;
    sqlx::query("BEGIN").execute(&mut *conn).await
        .context("ไม่สามารถเริ่ม transaction")?;

    match migration.up(&mut conn).await {
        Ok(()) => {
            sqlx::query("COMMIT").execute(&mut *conn).await
                .context("ไม่สามารถ commit transaction")?;
            Ok(())
        }
        Err(e) => {
            // ROLLBACK เฉพาะ migration นั้น — migration อื่นที่ผ่านแล้วไม่กระทบ
            let _ = sqlx::query("ROLLBACK").execute(&mut *conn).await;
            Err(e)
        }
    }
}

/// รัน migrate down
pub async fn run_down(
    pool: &AnyPool,
    registry: &MigrationRegistry,
    opts: &DownOptions,
) -> Result<()> {
    let mut conn = pool.acquire().await?;
    ensure_migrations_table(&mut conn).await?;
    let applied = load_applied_migrations(&mut conn).await?;
    drop(conn);

    let to_rollback = registry.applied_in_reverse(&applied);
    let count = opts.count.min(to_rollback.len());

    if count == 0 {
        println!("{}", "ไม่มี migration ที่ต้อง rollback".yellow());
        return Ok(());
    }

    println!("{}", format!("← Rollback {} migration(s)", count).cyan());

    for migration in to_rollback.iter().take(count) {
        let id = migration.id();

        if opts.dry_run {
            println!("{} {} {}", "[dry-run]".yellow(), "would rollback:".dimmed(), id.bold());
            continue;
        }

        println!("{} {}", "↓ Rolling back:".yellow().bold(), id);

        let mut conn = pool.acquire().await?;
        sqlx::query("BEGIN").execute(&mut *conn).await?;

        match migration.down(&mut conn).await {
            Ok(()) => {
                sqlx::query("COMMIT").execute(&mut *conn).await?;
                let mut conn2 = pool.acquire().await?;
                delete_migration_record(&mut conn2, id).await?;
                println!("  {} {}", "✓".green(), id);
            }
            Err(e) => {
                let _ = sqlx::query("ROLLBACK").execute(&mut *conn).await;
                bail!("Rollback '{}' ล้มเหลว: {}", id, e);
            }
        }
    }

    println!("{}", "✓ Rollback เสร็จสิ้น".green().bold());
    Ok(())
}

/// แสดง status table
pub async fn show_status(pool: &AnyPool, registry: &MigrationRegistry) -> Result<()> {
    let mut conn = pool.acquire().await?;
    ensure_migrations_table(&mut conn).await?;
    let applied = load_applied_migrations(&mut conn).await?;
    drop(conn);

    let infos = registry.compute_status(&applied);
    if infos.is_empty() {
        println!("{}", "ไม่มี migration ใน registry".dimmed());
        return Ok(());
    }

    println!(
        "{:<45} {:<12} {:<20} {:<8}",
        "Migration ID".bold(),
        "Status".bold(),
        "Applied At".bold(),
        "Duration".bold(),
    );
    println!("{}", "-".repeat(88));

    for info in &infos {
        let (status_str, applied_at_str, duration_str) = match &info.status {
            MigrationStatus::Pending => (
                "pending".yellow().to_string(),
                "-".to_string(),
                "-".to_string(),
            ),
            MigrationStatus::Applied { applied_at, duration_ms } => (
                "applied".green().to_string(),
                applied_at.format("%Y-%m-%d %H:%M:%S").to_string(),
                format!("{}ms", duration_ms),
            ),
            MigrationStatus::Modified { applied_at } => (
                "modified".red().bold().to_string(),
                applied_at.format("%Y-%m-%d %H:%M:%S").to_string(),
                "?".to_string(),
            ),
            MigrationStatus::Missing => (
                "missing".red().to_string(),
                "-".to_string(),
                "-".to_string(),
            ),
        };

        println!(
            "{:<45} {:<20} {:<20} {:<8}",
            info.id, status_str, applied_at_str, duration_str,
        );
    }

    println!("{}", "-".repeat(88));

    let pending_count  = infos.iter().filter(|i| i.status == MigrationStatus::Pending).count();
    let applied_count  = infos.iter().filter(|i| matches!(i.status, MigrationStatus::Applied { .. })).count();
    let modified_count = infos.iter().filter(|i| matches!(i.status, MigrationStatus::Modified { .. })).count();
    let missing_count  = infos.iter().filter(|i| i.status == MigrationStatus::Missing).count();

    println!(
        "Total: {} | {} applied | {} pending | {} modified | {} missing",
        infos.len(),
        applied_count.to_string().green(),
        pending_count.to_string().yellow(),
        modified_count.to_string().red(),
        missing_count.to_string().red(),
    );

    Ok(())
}
```

**⚠ Pitfall #2: อย่า share connection ระหว่าง BEGIN และ COMMIT**

เมื่อใช้ `AnyPool` (ไม่ใช่ `AnyTransaction`) ต้องใช้ connection เดียวกันตลอด transaction:

```rust
// ❌ ผิด — BEGIN และ COMMIT อาจใช้ connection คนละตัว!
sqlx::query("BEGIN").execute(&pool).await?;
migration.up(&pool).await?;
sqlx::query("COMMIT").execute(&pool).await?;

// ✅ ถูก — acquire connection หนึ่งตัว แล้วใช้ตลอด
let mut conn = pool.acquire().await?;
sqlx::query("BEGIN").execute(&mut *conn).await?;
migration.up(&mut conn).await?;   // ส่ง conn เดิม
sqlx::query("COMMIT").execute(&mut *conn).await?;
```

---

### ขั้นที่ 6: Concrete Migrations

สร้าง migrations จริงใน `src/migrations/`:

**`src/migrations/m20240101_000001_create_users.rs`:**

```rust
use crate::migration::Migration;
use anyhow::Result;
use sqlx::AnyConnection;
use std::pin::Pin;

// SOURCE เก็บ source code ทั้งหมดของ migration
// ใช้สำหรับคำนวณ checksum — ถ้าแก้ source นี้ checksum จะเปลี่ยน
const SOURCE: &str = r#"
-- Migration: 20240101_000001_create_users
-- สร้าง users table พื้นฐาน

UP:
CREATE TABLE users (
    id         INTEGER     NOT NULL PRIMARY KEY,
    username   TEXT        NOT NULL UNIQUE,
    email      TEXT        NOT NULL UNIQUE,
    created_at TEXT        NOT NULL DEFAULT (datetime('now'))
);

DOWN:
DROP TABLE IF EXISTS users;
"#;

pub struct CreateUsers;

impl Migration for CreateUsers {
    fn id(&self) -> &str {
        "20240101_000001_create_users"
    }

    fn source(&self) -> &str {
        SOURCE
    }

    fn up<'a>(
        &'a self,
        conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query(
                "CREATE TABLE IF NOT EXISTS users (
                    id         INTEGER NOT NULL PRIMARY KEY,
                    username   TEXT    NOT NULL,
                    email      TEXT    NOT NULL,
                    created_at TEXT    NOT NULL DEFAULT (datetime('now'))
                )"
            )
            .execute(&mut *conn)
            .await?;
            Ok(())
        })
    }

    fn down<'a>(
        &'a self,
        conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query("DROP TABLE IF EXISTS users")
                .execute(&mut *conn)
                .await?;
            Ok(())
        })
    }
}
```

**`src/migrations/m20240102_000001_create_posts.rs`:**

```rust
use crate::migration::Migration;
use anyhow::Result;
use sqlx::AnyConnection;
use std::pin::Pin;

const SOURCE: &str = r#"
-- Migration: 20240102_000001_create_posts
UP:
CREATE TABLE posts (
    id         INTEGER  NOT NULL PRIMARY KEY,
    user_id    INTEGER  NOT NULL REFERENCES users(id),
    title      TEXT     NOT NULL,
    body       TEXT     NOT NULL DEFAULT '',
    created_at TEXT     NOT NULL DEFAULT (datetime('now'))
);
DOWN:
DROP TABLE IF EXISTS posts;
"#;

pub struct CreatePosts;

impl Migration for CreatePosts {
    fn id(&self) -> &str { "20240102_000001_create_posts" }
    fn source(&self) -> &str { SOURCE }

    fn up<'a>(
        &'a self, conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query(
                "CREATE TABLE IF NOT EXISTS posts (
                    id INTEGER NOT NULL PRIMARY KEY,
                    user_id INTEGER NOT NULL,
                    title TEXT NOT NULL,
                    body TEXT NOT NULL DEFAULT '',
                    created_at TEXT NOT NULL DEFAULT (datetime('now'))
                )"
            )
            .execute(&mut *conn).await?;
            Ok(())
        })
    }

    fn down<'a>(
        &'a self, conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query("DROP TABLE IF EXISTS posts").execute(&mut *conn).await?;
            Ok(())
        })
    }
}
```

**`src/migrations/m20240103_000001_add_email_index.rs`:**

```rust
use crate::migration::Migration;
use anyhow::Result;
use sqlx::AnyConnection;
use std::pin::Pin;

const SOURCE: &str = r#"
-- Migration: 20240103_000001_add_email_index
UP:
CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email ON users(email);
DOWN:
DROP INDEX IF EXISTS idx_users_email;
"#;

pub struct AddEmailIndex;

impl Migration for AddEmailIndex {
    fn id(&self) -> &str { "20240103_000001_add_email_index" }
    fn source(&self) -> &str { SOURCE }

    fn up<'a>(
        &'a self, conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query(
                "CREATE UNIQUE INDEX IF NOT EXISTS idx_users_email ON users(email)"
            )
            .execute(&mut *conn).await?;
            Ok(())
        })
    }

    fn down<'a>(
        &'a self, conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn std::future::Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query("DROP INDEX IF EXISTS idx_users_email")
                .execute(&mut *conn).await?;
            Ok(())
        })
    }
}
```

**`src/migrations/mod.rs`:**

```rust
mod m20240101_000001_create_users;
mod m20240102_000001_create_posts;
mod m20240103_000001_add_email_index;

pub use m20240101_000001_create_users::CreateUsers;
pub use m20240102_000001_create_posts::CreatePosts;
pub use m20240103_000001_add_email_index::AddEmailIndex;

use crate::migration::Migration;
use std::sync::Arc;

/// รวบรวม migrations ทั้งหมด — เพิ่ม migration ใหม่ที่นี่
pub fn all_migrations() -> Vec<Arc<dyn Migration>> {
    vec![
        Arc::new(CreateUsers),
        Arc::new(CreatePosts),
        Arc::new(AddEmailIndex),
    ]
}
```

---

### ขั้นที่ 7: Seed Data System

**`src/seed.rs`** — seed ที่ environment-aware:

```rust
use crate::migration::Migration;
use crate::db::{ensure_migrations_table, load_applied_migrations, record_migration};
use crate::migration::calculate_checksum;
use anyhow::{bail, Result};
use colored::*;
use sqlx::AnyPool;
use std::sync::Arc;
use std::time::Instant;

/// ตรวจสอบว่าควร run seed หรือไม่
/// APP_ENV=production → skip
pub fn should_run_seeds() -> bool {
    let env = std::env::var("APP_ENV").unwrap_or_default();
    env != "production"
}

/// รัน seed migrations
pub async fn run_seeds(
    pool: &AnyPool,
    seeds: &[Arc<dyn Migration>],
    dry_run: bool,
) -> Result<()> {
    if !should_run_seeds() {
        println!("{}", "⚠  APP_ENV=production — ข้าม seed migrations".yellow());
        return Ok(());
    }

    let mut conn = pool.acquire().await?;
    ensure_migrations_table(&mut conn).await?;
    let applied = load_applied_migrations(&mut conn).await?;
    drop(conn);

    // กรอง seeds ที่ยังไม่ได้รัน
    let pending_seeds: Vec<_> = seeds
        .iter()
        .filter(|s| !applied.contains_key(s.id()))
        .collect();

    if pending_seeds.is_empty() {
        println!("{}", "✓ ไม่มี seed ที่ต้อง run".green());
        return Ok(());
    }

    println!("{}", format!("🌱 Seeding {} pending seed(s)", pending_seeds.len()).cyan());

    for seed in pending_seeds {
        let id = seed.id();
        let checksum = calculate_checksum(seed.source());

        if dry_run {
            println!("{} {} {}", "[dry-run]".yellow(), "would seed:".dimmed(), id.bold());
            continue;
        }

        println!("{} {}", "🌱 Seeding:".blue(), id);
        let start = Instant::now();

        let mut conn = pool.acquire().await?;
        sqlx::query("BEGIN").execute(&mut *conn).await?;

        match seed.up(&mut conn).await {
            Ok(()) => {
                sqlx::query("COMMIT").execute(&mut *conn).await?;
                let duration_ms = start.elapsed().as_millis() as i64;
                let mut conn2 = pool.acquire().await?;
                record_migration(&mut conn2, id, &checksum, duration_ms).await?;
                println!("  {} {} ({}ms)", "✓".green(), id, duration_ms);
            }
            Err(e) => {
                let _ = sqlx::query("ROLLBACK").execute(&mut *conn).await;
                bail!("Seed '{}' ล้มเหลว: {}", id, e);
            }
        }
    }

    println!("{}", "✓ Seed เสร็จสิ้น".green().bold());
    Ok(())
}
```

**ตัวอย่าง Seed Migration สำหรับ Development:**

```rust
// src/seeds/s20240101_000001_dev_users.rs
const SOURCE: &str = "-- Seed: dev users data";

pub struct DevUsers;

impl Migration for DevUsers {
    fn id(&self) -> &str { "seed_20240101_000001_dev_users" }
    fn source(&self) -> &str { SOURCE }

    fn up<'a>(
        &'a self, conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query(
                "INSERT OR IGNORE INTO users (id, username, email) VALUES
                    (1, 'alice', 'alice@example.com'),
                    (2, 'bob',   'bob@example.com')"
            )
            .execute(&mut *conn).await?;
            Ok(())
        })
    }

    fn down<'a>(
        &'a self, conn: &'a mut AnyConnection,
    ) -> Pin<Box<dyn Future<Output = Result<()>> + Send + 'a>> {
        Box::pin(async move {
            sqlx::query("DELETE FROM users WHERE id IN (1, 2)")
                .execute(&mut *conn).await?;
            Ok(())
        })
    }
}
```

---

### ขั้นที่ 8: CLI Entry Point

**`src/main.rs`** — ประกอบทุกอย่างเข้าด้วยกัน:

```rust
mod db;
mod migration;
mod migrations;
mod registry;
mod runner;
mod seed;

use crate::migrations::all_migrations;
use crate::registry::MigrationRegistry;
use crate::runner::{DownOptions, UpOptions};
use anyhow::Result;
use clap::{Parser, Subcommand};
use sqlx::any::AnyPoolOptions;

/// Data Migration Framework
#[derive(Parser)]
#[command(name = "migrate", about = "Data migration framework", version)]
struct Cli {
    /// Database URL: sqlite:/path/to/db หรือ postgres://user:pass@host/db
    #[arg(long, env = "DATABASE_URL", default_value = "sqlite::memory:")]
    database_url: String,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Apply pending migrations
    Up {
        #[arg(long)]
        dry_run: bool,
        /// Fail ถ้า applied migration มี checksum เปลี่ยน
        #[arg(long)]
        strict: bool,
    },
    /// Rollback migrations
    Down {
        #[arg(default_value = "1")]
        count: usize,
        #[arg(long)]
        dry_run: bool,
    },
    /// แสดงสถานะ migrations
    Status,
    /// รัน seed data (skip ใน production)
    Seed {
        #[arg(long)]
        dry_run: bool,
    },
}

#[tokio::main]
async fn main() -> Result<()> {
    // ติดตั้ง SQLite driver (จำเป็นสำหรับ AnyPool)
    sqlx::any::install_default_drivers();

    let cli = Cli::parse();

    let pool = AnyPoolOptions::new()
        .max_connections(5)
        .connect(&cli.database_url)
        .await?;

    let mut registry = MigrationRegistry::new();
    registry.register_all(all_migrations());

    match cli.command {
        Commands::Up { dry_run, strict } => {
            runner::run_up(&pool, &registry, &UpOptions { dry_run, strict }).await?;
        }
        Commands::Down { count, dry_run } => {
            runner::run_down(&pool, &registry, &DownOptions { count, dry_run }).await?;
        }
        Commands::Status => {
            runner::show_status(&pool, &registry).await?;
        }
        Commands::Seed { dry_run } => {
            // เพิ่ม seed migrations ที่นี่
            let seeds = vec![];
            seed::run_seeds(&pool, &seeds, dry_run).await?;
        }
    }

    Ok(())
}
```

**⚠ Pitfall #3: ต้องเรียก `install_default_drivers()` ก่อนใช้ AnyPool**

```rust
// ❌ ผิด — จะ panic ว่าไม่มี driver
let pool = AnyPool::connect("sqlite::memory:").await?;

// ✅ ถูก — ติดตั้ง drivers ก่อน
sqlx::any::install_default_drivers();
let pool = AnyPool::connect("sqlite::memory:").await?;
```

---

### ขั้นที่ 9: PostgreSQL Mode

สำหรับ production ที่ใช้ PostgreSQL ต้อง:

1. เปิด feature flag:

```bash
cargo build --features postgres
```

2. เปลี่ยน SQL syntax ใน migration บางส่วน:

```rust
// SQLite ใช้ INTEGER PRIMARY KEY
// PostgreSQL ใช้ SERIAL หรือ BIGSERIAL
fn up<'a>(...) -> Pin<...> {
    Box::pin(async move {
        // ใช้ parameterized query ที่ทำงานได้ทั้ง 2 DB
        #[cfg(feature = "postgres")]
        sqlx::query(
            "CREATE TABLE IF NOT EXISTS users (
                id         BIGSERIAL   PRIMARY KEY,
                username   TEXT        NOT NULL UNIQUE,
                email      TEXT        NOT NULL UNIQUE,
                created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
            )"
        ).execute(&mut *conn).await?;

        #[cfg(not(feature = "postgres"))]
        sqlx::query(
            "CREATE TABLE IF NOT EXISTS users (
                id         INTEGER NOT NULL PRIMARY KEY,
                username   TEXT    NOT NULL,
                email      TEXT    NOT NULL,
                created_at TEXT    NOT NULL DEFAULT (datetime('now'))
            )"
        ).execute(&mut *conn).await?;

        Ok(())
    })
}
```

3. ตั้งค่า DATABASE_URL:

```bash
export DATABASE_URL="postgres://myuser:mypass@localhost:5432/mydb"
./target/release/migrate up
```

**⚠ Pitfall #4: `__migrations` table ต้องใช้ TEXT ไม่ใช่ TIMESTAMPTZ สำหรับ AnyPool**

เมื่อใช้ `AnyPool` (runtime polymorphism) sqlx ไม่สามารถ map `TIMESTAMPTZ` → `DateTime<Utc>` ได้โดยตรง ต้องเก็บเป็น `TEXT` ในรูป RFC3339 แล้ว parse เอง:

```rust
// ❌ ผิดกับ AnyPool
let applied_at: DateTime<Utc> = row.try_get("applied_at")?;

// ✅ ถูก — เก็บเป็น TEXT แล้ว parse
let applied_at_str: String = row.try_get("applied_at")?;
let applied_at = DateTime::parse_from_rfc3339(&applied_at_str)?
    .with_timezone(&Utc);
```

---

## การทดสอบ (Testing)

### Unit Tests ที่ครอบคลุมหลักการสำคัญ

โปรเจคมี unit tests สำหรับ logic ที่ไม่ต้องการ DB จริง:

**`src/migration.rs` — checksum tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_checksum_deterministic() {
        let source = "CREATE TABLE users (id SERIAL PRIMARY KEY);";
        let c1 = calculate_checksum(source);
        let c2 = calculate_checksum(source);
        assert_eq!(c1, c2, "checksum ต้องได้ผลเหมือนกันทุกครั้ง");
    }

    #[test]
    fn test_checksum_different_for_different_source() {
        let s1 = "CREATE TABLE users (id SERIAL PRIMARY KEY);";
        let s2 = "CREATE TABLE posts (id SERIAL PRIMARY KEY);";
        assert_ne!(
            calculate_checksum(s1),
            calculate_checksum(s2),
            "source ต่างกันต้องได้ checksum ต่างกัน"
        );
    }

    #[test]
    fn test_checksum_sha256_length() {
        // SHA-256 hex = 64 characters เสมอ
        assert_eq!(calculate_checksum("test").len(), 64);
    }

    #[test]
    fn test_checksum_known_value() {
        // SHA-256("hello") = 2cf24dba5fb0a30e...
        let checksum = calculate_checksum("hello");
        assert_eq!(
            checksum,
            "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"
        );
    }
}
```

**`src/registry.rs` — migration sorting และ status tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Mock migration สำหรับ test
    struct MockMigration { id: String, source: String }

    impl MockMigration {
        fn new(id: &str) -> Self {
            Self { id: id.to_string(), source: format!("-- {}", id) }
        }
    }

    impl Migration for MockMigration {
        fn id(&self) -> &str { &self.id }
        fn source(&self) -> &str { &self.source }
        fn up<'a>(&'a self, _conn: &'a mut AnyConnection) -> Pin<Box<...>> {
            Box::pin(async { Ok(()) })
        }
        fn down<'a>(&'a self, _conn: &'a mut AnyConnection) -> Pin<Box<...>> {
            Box::pin(async { Ok(()) })
        }
    }

    #[test]
    fn test_registry_sorts_by_id() {
        let mut registry = MigrationRegistry::new();
        // ใส่ลำดับกลับกัน
        registry.register(Arc::new(MockMigration::new("20240201_000001_add_posts")));
        registry.register(Arc::new(MockMigration::new("20240101_000001_create_users")));
        registry.register(Arc::new(MockMigration::new("20240301_000001_add_comments")));

        let ids: Vec<&str> = registry.all().iter().map(|m| m.id()).collect();
        assert_eq!(ids, vec![
            "20240101_000001_create_users",
            "20240201_000001_add_posts",
            "20240301_000001_add_comments",
        ]);
    }

    #[test]
    fn test_compute_status_modified() {
        let mut registry = MigrationRegistry::new();
        let m = Arc::new(MockMigration::new("20240101_000001_create_users"));
        registry.register(m);

        let mut applied = HashMap::new();
        applied.insert("20240101_000001_create_users".to_string(), MigrationRecord {
            id: "20240101_000001_create_users".to_string(),
            applied_at: chrono::Utc::now(),
            checksum: "old_checksum_that_differs".to_string(), // checksum เก่า
            duration_ms: 10,
        });

        let infos = registry.compute_status(&applied);
        assert!(
            matches!(infos[0].status, MigrationStatus::Modified { .. }),
            "checksum ต่าง → ต้องเป็น Modified"
        );
    }
}
```

**`src/seed.rs` — environment detection tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_should_run_seeds_in_dev() {
        std::env::remove_var("APP_ENV");
        assert!(should_run_seeds(), "ใน dev ต้อง run seeds");
    }

    #[test]
    fn test_should_not_run_seeds_in_production() {
        std::env::set_var("APP_ENV", "production");
        assert!(!should_run_seeds(), "ใน production ต้อง skip");
        std::env::remove_var("APP_ENV");
    }
}
```

### ผลการรัน `cargo test`

```
$ cargo test
   Compiling data-migration v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.61s
     Running unittests src/main.rs (target/debug/deps/migrate-c9fb146b2862e0ee)

running 15 tests
test migration::tests::test_checksum_known_value ... ok
test migration::tests::test_checksum_sha256_length ... ok
test migration::tests::test_checksum_deterministic ... ok
test migration::tests::test_checksum_different_for_different_source ... ok
test registry::tests::test_compute_status_missing ... ok
test registry::tests::test_applied_in_reverse ... ok
test registry::tests::test_compute_status_modified ... ok
test registry::tests::test_pending_migrations ... ok
test registry::tests::test_registry_register_all_sorts ... ok
test registry::tests::test_registry_sorts_by_id ... ok
test runner::tests::test_dry_run_does_not_panic ... ok
test runner::tests::test_status_counts ... ok
test seed::tests::test_should_not_run_seeds_in_production ... ok
test seed::tests::test_should_run_seeds_in_dev ... ok
test seed::tests::test_should_run_seeds_in_staging ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ผลการรันจริง (Integration)

**รัน `migrate up`:**

```
$ DATABASE_URL="sqlite:///tmp/myapp.db" ./target/release/migrate up
→ มี 3 pending migration(s)
↑ Applying: 20240101_000001_create_users
  ✓ 20240101_000001_create_users (2ms)
↑ Applying: 20240102_000001_create_posts
  ✓ 20240102_000001_create_posts (1ms)
↑ Applying: 20240103_000001_add_email_index
  ✓ 20240103_000001_add_email_index (1ms)
✓ Migration เสร็จสิ้น
```

**รัน `migrate status` หลัง apply:**

```
$ DATABASE_URL="sqlite:///tmp/myapp.db" ./target/release/migrate status
Migration ID                                  Status       Applied At           Duration
----------------------------------------------------------------------------------------
20240101_000001_create_users                  applied      2026-09-27 12:00:27  2ms
20240102_000001_create_posts                  applied      2026-09-27 12:00:27  1ms
20240103_000001_add_email_index               applied      2026-09-27 12:00:27  1ms
----------------------------------------------------------------------------------------
Total: 3 | 3 applied | 0 pending | 0 modified | 0 missing
```

**รัน `migrate down 1` (rollback 1 migration):**

```
$ DATABASE_URL="sqlite:///tmp/myapp.db" ./target/release/migrate down 1
← Rollback 1 migration(s)
↓ Rolling back: 20240103_000001_add_email_index
  ✓ 20240103_000001_add_email_index
✓ Rollback เสร็จสิ้น
```

**รัน `migrate status` หลัง rollback:**

```
$ DATABASE_URL="sqlite:///tmp/myapp.db" ./target/release/migrate status
Migration ID                                  Status       Applied At           Duration
----------------------------------------------------------------------------------------
20240101_000001_create_users                  applied      2026-09-27 12:00:27  2ms
20240102_000001_create_posts                  applied      2026-09-27 12:00:27  1ms
20240103_000001_add_email_index               pending      -                    -
----------------------------------------------------------------------------------------
Total: 3 | 2 applied | 1 pending | 0 modified | 0 missing
```

**รัน `migrate up --dry-run`:**

```
$ DATABASE_URL="sqlite::memory:" ./target/release/migrate up --dry-run
→ มี 3 pending migration(s)
[dry-run] would apply: 20240101_000001_create_users
  source preview:
-- Migration: 20240101_000001_create_users
-- สร้าง users tabl...
[dry-run] would apply: 20240102_000001_create_posts
  source preview:
-- Migration: 20240102_000001_create_posts...
[dry-run] would apply: 20240103_000001_add_email_index
  source preview:
-- Migration: 20240103_000001_add_email_ind...
✓ Migration เสร็จสิ้น
```

---

## Pitfalls สรุปและวิธีแก้

### Pitfall #1: `&mut *conn` pattern ใน sqlx

```rust
// ❌ ผิด
sqlx::query("...").execute(conn).await?;

// ✅ ถูก
sqlx::query("...").execute(&mut *conn).await?;
```

เหตุผล: `sqlx::query().execute()` รับ `impl Executor<'_, Database = Any>` ซึ่ง `&mut AnyConnection` implement แต่ `AnyConnection` เองไม่ implement เพราะต้องการ exclusive borrow

### Pitfall #2: Connection ต้องเป็นตัวเดียวตลอด Transaction

```rust
// ❌ pool อาจให้ connection คนละตัว
sqlx::query("BEGIN").execute(&pool).await?;
migration.up(&pool).await?;
sqlx::query("COMMIT").execute(&pool).await?;

// ✅ acquire หนึ่งครั้ง ส่งต่อทุก operation
let mut conn = pool.acquire().await?;
sqlx::query("BEGIN").execute(&mut *conn).await?;
migration.up(&mut conn).await?;
sqlx::query("COMMIT").execute(&mut *conn).await?;
```

### Pitfall #3: ต้องเรียก `install_default_drivers()` ก่อน AnyPool

```rust
// ❌ runtime panic
let pool = AnyPool::connect("sqlite::memory:").await?;

// ✅ ต้องติดตั้ง drivers ก่อน
sqlx::any::install_default_drivers();
let pool = AnyPool::connect("sqlite::memory:").await?;
```

`install_default_drivers()` ลงทะเบียน SQLite/PostgreSQL/MySQL driver เข้า global registry ของ sqlx

### Pitfall #4: AnyPool ไม่รองรับ TIMESTAMPTZ โดยตรง

```rust
// ❌ ไม่ทำงานกับ AnyPool — type mismatch
let applied_at: DateTime<Utc> = row.try_get("applied_at")?;

// ✅ เก็บเป็น TEXT แล้ว parse
let applied_at_str: String = row.try_get("applied_at")?;
let applied_at = DateTime::parse_from_rfc3339(&applied_at_str)
    .map(|dt| dt.with_timezone(&Utc))
    .unwrap_or_else(|_| Utc::now());
```

เมื่อใช้ `AnyPool` type mapping จะไม่ specific เท่ากับ `PgPool` ดังนั้นต้องเก็บ datetime เป็น string ก่อน

### Pitfall #5: String Slicing บน Thai/Unicode ต้องใช้ `chars()`

```rust
// ❌ panic ถ้า source มี multibyte characters (Thai, Chinese, ฯลฯ)
let preview = &source[..80];

// ✅ ใช้ chars() เพื่อ iterate ที่ char boundary
let preview: String = source.chars().take(80).collect();
```

Rust string indexing ทำงานที่ byte level — ถ้า byte ที่ 80 อยู่กลาง UTF-8 sequence จะ panic ทันที

---

## การ Package และ Deploy

### Build Release Binary

```bash
# SQLite mode (default, สำหรับ dev/staging)
cargo build --release

# PostgreSQL mode (สำหรับ production)
cargo build --release --features postgres --no-default-features
```

### Docker

```dockerfile
FROM rust:1.75 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release --features postgres --no-default-features

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/migrate /usr/local/bin/migrate
ENTRYPOINT ["migrate"]
```

```bash
# Build
docker build -t data-migration .

# Run migration ใน production
docker run --rm \
  -e DATABASE_URL="postgres://user:pass@db:5432/mydb" \
  data-migration up --strict
```

### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/migrate.yml
name: Database Migration

on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        ports: ["5432:5432"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions-rust-lang/setup-rust-toolchain@v1

      - name: Build migration tool
        run: cargo build --release --features postgres --no-default-features

      - name: Run dry-run first
        run: ./target/release/migrate --database-url "$DATABASE_URL" up --dry-run --strict
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/testdb

      - name: Apply migrations
        run: ./target/release/migrate --database-url "$DATABASE_URL" up --strict
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost/testdb
```

### Config ด้วย Environment Variables

```bash
# .env สำหรับ development
DATABASE_URL=sqlite:///var/lib/myapp/dev.db
APP_ENV=development

# .env.production
DATABASE_URL=postgres://myuser:secret@prod-db:5432/myapp
APP_ENV=production
```

```bash
# โหลด .env แล้วรัน
set -a; source .env; set +a
./migrate up --strict
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Parallel Migration Detection (ระดับกลาง)

เพิ่มการตรวจสอบว่ามี migration กำลังรันอยู่แล้วหรือไม่ (advisory lock) ป้องกัน 2 processes รัน migration พร้อมกัน:

**โจทย์:** implement `acquire_migration_lock()` และ `release_migration_lock()` ที่ใช้ `__migration_lock` table เป็น mutex:

```sql
CREATE TABLE __migration_lock (
    locked_at TEXT,
    locked_by TEXT  -- hostname:pid
);
```

- ถ้า lock มีอยู่และ locked_at < 5 นาทีที่แล้ว → fail ด้วย error "migration is already running"
- ถ้า lock หมดอายุ → overwrite lock เก่า
- ใช้ `std::panic::catch_unwind` หรือ `Drop` trait เพื่อ release lock อัตโนมัติ

### แบบฝึกหัดที่ 2: Migration Checksum Repair (ระดับกลาง)

เพิ่ม subcommand `migrate fix-checksum [ID]` ที่:

1. แสดง migration ทั้งหมดที่มี `Modified` status
2. ถ้า user ยืนยัน → update checksum ใน `__migrations` table ให้ตรงกับ current source
3. ต้องมี flag `--confirm` เพื่อป้องกันรันโดยไม่ตั้งใจ

```bash
migrate fix-checksum --confirm 20240101_000001_create_users
# Updated checksum for 20240101_000001_create_users
```

### แบบฝึกหัดที่ 3: Migration History Log (ระดับสูง)

เพิ่ม table `__migration_history` ที่เก็บประวัติทุก operation (up/down/failed):

```sql
CREATE TABLE __migration_history (
    seq        INTEGER PRIMARY KEY,
    migration_id TEXT    NOT NULL,
    operation  TEXT    NOT NULL,  -- 'up' | 'down' | 'failed'
    executed_at TEXT   NOT NULL,
    duration_ms INTEGER,
    error_msg  TEXT               -- NULL ถ้าสำเร็จ
);
```

- ทุก `run_up` และ `run_down` ต้อง insert ลง history
- เพิ่ม subcommand `migrate history [--limit N]` แสดง N รายการล่าสุด
- ใช้ `serde_json` serialize error details ลงใน `error_msg`

### แบบฝึกหัดที่ 4: Async Trait ด้วย `async_trait` Crate (ระดับสูง)

Framework ปัจจุบันใช้ `Pin<Box<dyn Future>>` ซึ่งยุ่งยาก เพิ่ม `async_trait` crate แล้วเปรียบเทียบ:

```toml
[dependencies]
async-trait = "0.1"
```

```rust
// แบบเก่า — verbose
fn up<'a>(
    &'a self,
    conn: &'a mut AnyConnection,
) -> Pin<Box<dyn Future<Output = Result<()>> + Send + 'a>>;

// แบบใหม่ด้วย async_trait — clean กว่ามาก
#[async_trait]
pub trait Migration: Send + Sync {
    fn id(&self) -> &str;
    fn source(&self) -> &str;
    async fn up(&self, conn: &mut AnyConnection) -> Result<()>;
    async fn down(&self, conn: &mut AnyConnection) -> Result<()>;
}
```

**โจทย์:** migrate codebase ทั้งหมดให้ใช้ `async_trait` โดยไม่ break tests ใดๆ ทดสอบด้วย `cargo test`

---

## สรุป

โปรเจคนี้สร้าง **Data Migration Framework** ที่ครบถ้วนสำหรับ production:

| Feature | ที่เรียนได้ |
|---|---|
| Migration trait | Trait objects, `dyn Trait`, dynamic dispatch |
| `Pin<Box<dyn Future>>` | Async traits ใน stable Rust |
| Registry + sorting | `Arc`, shared ownership, lexicographic ordering |
| Checksum detection | SHA-256, hex encoding, fingerprinting |
| Individual transactions | Connection management, BEGIN/COMMIT/ROLLBACK |
| `AnyPool` | Runtime polymorphism ระหว่าง DB backends |
| Seed system | Environment-aware logic, `APP_ENV` convention |
| Status table | Enum pattern matching, colored terminal output |

**Pattern สำคัญที่ได้เรียน:**

1. **Async Trait Pattern** — `Pin<Box<dyn Future + Send + 'a>>` เป็น workaround standard สำหรับ async trait methods
2. **Registry Pattern** — collect + sort + filter objects ผ่าน trait interface
3. **Idempotent Operations** — `CREATE TABLE IF NOT EXISTS` ทำให้รัน migration ซ้ำได้ปลอดภัย
4. **Fingerprinting** — SHA-256 checksum ของ source code ตรวจจับการแก้ไขได้ทันที
5. **Environment Gates** — check `APP_ENV` ก่อนรัน side-effectful operations

**เชื่อมโยงกับโปรเจคถัดไป:**

โปรเจค C09 Query Parser จะต่อยอดจาก DB foundation ที่สร้างไว้ โดยเพิ่ม query language parser ที่แปลง DSL → SQL ซึ่งต้องการความเข้าใจเรื่อง Rust parser combinators และ AST (Abstract Syntax Tree) ที่ลึกขึ้น

---

**โปรเจคก่อนหน้า:** [project-c07-db-backup.md](project-c07-db-backup.md) | **โปรเจคถัดไป:** [project-c09-query-parser.md](project-c09-query-parser.md)
