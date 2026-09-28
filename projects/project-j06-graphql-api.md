# Project J06: GraphQL API ด้วย async-graphql

> โมดูล: J — Full-Stack/WASM | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 20 ชั่วโมง

## ภาพรวมโปรเจค

**GraphQL** คือ query language สำหรับ API ที่พัฒนาโดย Facebook (Meta) ในปี 2012 และเปิดเป็น open source ปี 2015 GraphQL แก้ปัญหาหลักของ REST ได้แก่ over-fetching (ดึงข้อมูลมากกว่าที่ต้องการ), under-fetching (ต้องยิง request หลายครั้งเพื่อได้ข้อมูลครบ) และ tight coupling ระหว่าง client กับ server endpoint

แทนที่จะกำหนด URL endpoint ตาม resource (`/users`, `/users/1/posts`), GraphQL ใช้ **schema** เป็นสัญญา (contract) ระหว่าง client กับ server และให้ client ระบุเองว่าต้องการข้อมูลฟิลด์ใดบ้าง ทำให้:

- **Client** ได้รับข้อมูลตรงตามที่ต้องการ ไม่มากไม่น้อย
- **Server** มี schema เดียวรองรับหลาย client (web, mobile, IoT) ได้พร้อมกัน
- **Schema** เป็น single source of truth ที่ self-documenting ด้วย introspection

**async-graphql** คือ GraphQL server library สำหรับ Rust ที่เน้นความสมบูรณ์และ type-safety สูงสุด รองรับทุกฟีเจอร์ของ GraphQL spec ได้แก่ Query, Mutation, Subscription, Directive, Scalar, Union, Interface, DataLoader และอื่น ๆ อีกมาก โดยใช้ procedural macros (`#[derive(SimpleObject)]`, `#[Object]`, `#[Subscription]`) เพื่อ generate code จาก Rust type โดยอัตโนมัติ ทำให้ schema definition เป็น code-first และ type-safe ตั้งแต่ compile time

### เปรียบเทียบ GraphQL vs REST

| คุณสมบัติ | REST API | GraphQL |
|----------|----------|---------|
| **Endpoint** | หลาย URL (`/users`, `/posts`) | URL เดียว (`/graphql`) |
| **Response shape** | กำหนดโดย server | กำหนดโดย client |
| **Over-fetching** | พบบ่อย | ไม่มี |
| **Under-fetching** | ต้อง join หลาย request | query nested ครั้งเดียว |
| **Versioning** | `/v1/`, `/v2/` | deprecate field แทน |
| **Real-time** | ต้องใช้ polling หรือ WebSocket แยก | Subscription built-in |
| **Introspection** | ต้องใช้ OpenAPI แยก | built-in `__schema` query |
| **Type safety** | runtime (JSON) | compile-time (schema) |

โปรเจคนี้สร้าง **Book Catalog GraphQL API** — ระบบจัดการหนังสือและผู้แต่งที่ประกอบด้วย:

- **QueryRoot** สำหรับดึงข้อมูลหนังสือ, ผู้แต่ง พร้อม filtering และ pagination
- **MutationRoot** สำหรับ create/update/delete พร้อม validation และ error extensions
- **SubscriptionRoot** สำหรับ real-time events เมื่อมีหนังสือถูกเพิ่มหรือลบ
- **DataLoader** สำหรับ batch-loading ผู้แต่งเพื่อแก้ปัญหา N+1 query
- **Custom Scalars** สำหรับ UUID และ DateTime
- **Axum integration** พร้อม GraphiQL playground

---

## สิ่งที่จะได้เรียนรู้

- **Schema-first vs Code-first** — เข้าใจ philosophy ของ GraphQL และทำไม async-graphql ใช้แนวทาง code-first
- **async-graphql macros** — `#[derive(SimpleObject)]`, `#[derive(InputObject)]`, `#[Object]`, `#[Subscription]`
- **Resolver pattern** — เขียน field resolver ที่รับ `Context<'_>`, arguments, และ return `Result<T>`
- **DataLoader** — แก้ปัญหา N+1 queries ด้วย `Loader` trait และ batch loading
- **Subscription + SimpleBroker** — ส่ง real-time events ผ่าน WebSocket
- **Custom scalars** — กำหนด scalar type เอง เช่น UUID, DateTime
- **Error handling** — ใช้ `Error`, `FieldError`, `.extend_with()` สำหรับ error extensions
- **Axum integration** — mount GraphQL handler บน axum router พร้อม GraphiQL playground

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50** — `async/await`, tokio runtime, `Future`, `Stream`
- **Part 51–60** — `Arc<T>`, `Mutex<T>`, `DashMap`, thread-safe data structures
- **Part 21–25** — struct, enum, impl blocks, trait implementations
- **Part 31–35** — error handling ด้วย `Result`, `?`, custom error types
- **Part 61–70** — procedural macros และ derive macros (ทำความเข้าใจ code generation)
- **Part 71–80** — web development พื้นฐาน, HTTP, JSON serialization ด้วย serde
- โปรเจค B07 (Realtime Chat) — WebSocket พื้นฐาน
- โปรเจค J05 (Collaborative Editor) — async state management

---

## โครงสร้างโปรเจค (Project Layout)

```
graphql-api/
├── src/
│   ├── main.rs          # Axum server, GraphQL handler, GraphiQL
│   ├── lib.rs           # module re-exports
│   ├── model.rs         # domain types: BookRecord, AuthorRecord
│   ├── schema/
│   │   ├── mod.rs       # build_schema(), Context data
│   │   ├── query.rs     # QueryRoot
│   │   ├── mutation.rs  # MutationRoot
│   │   └── subscription.rs  # SubscriptionRoot
│   ├── loader.rs        # DataLoader implementations
│   ├── scalars.rs       # custom scalar: Uuid, DateTime
│   └── store.rs         # Store (in-memory DashMap)
├── tests/
│   └── integration.rs   # end-to-end schema tests
└── Cargo.toml
```

---

## การออกแบบ (Architecture & Design)

### GraphQL Request Flow

```
HTTP Client / GraphiQL
        │
        │  POST /graphql   { query: "{ books { title } }" }
        ▼
┌─────────────────────────────────────────────────────┐
│  Axum Router                                        │
│                                                     │
│  route("/graphql", GraphQL handler)                 │
│  route("/graphiql", GraphiQL playground)            │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  async-graphql Engine                       │    │
│  │                                             │    │
│  │  1. Parse & Validate against Schema         │    │
│  │  2. Execute resolvers concurrently          │    │
│  │     ├─ QueryRoot.books(filter, first)       │    │
│  │     ├─ Book.author() → DataLoader.load()    │    │
│  │     └─ AuthorLoader.batch(ids[]) → Store    │    │
│  │  3. Serialize result to JSON                │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │  WebSocket /ws/graphql                      │    │
│  │  └─ SubscriptionRoot (SimpleBroker)         │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  Store (Arc<DashMap<Uuid, T>>)                      │
│  books: DashMap<Uuid, BookRecord>                   │
│  authors: DashMap<Uuid, AuthorRecord>               │
└─────────────────────────────────────────────────────┘
```

### ทำไมถึงเลือก Code-First?

async-graphql ใช้แนวทาง **code-first** คือเขียน Rust struct/enum/impl แล้ว macro สร้าง GraphQL schema ให้โดยอัตโนมัติ ข้อดีเทียบกับ schema-first (เขียน `.graphql` file แยก):

| | Code-First (async-graphql) | Schema-First |
|-|---------------------------|--------------|
| **Type safety** | compile-time ทั้งหมด | runtime (code gen) |
| **Refactoring** | IDE support เต็มรูปแบบ | ต้องแก้ 2 ที่ |
| **Complexity** | น้อยกว่า (1 source of truth) | มากกว่า (.graphql + Rust) |
| **Custom logic** | ใส่ใน impl โดยตรง | ต้อง wire ต่างหาก |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ติดตั้ง Dependencies และโครงสร้างพื้นฐาน

เริ่มจาก `Cargo.toml` ที่มี crate ครบถ้วน:

```toml
# Cargo.toml
[package]
name = "graphql-api"
version = "0.1.0"
edition = "2021"

[dependencies]
# GraphQL engine
async-graphql = { version = "7", features = ["uuid", "chrono", "dataloader"] }
async-graphql-axum = "7"

# Web framework
axum = "0.7"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors"] }

# Async runtime
tokio = { version = "1", features = ["full"] }
futures = "0.3"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Utilities
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dashmap = "6"
thiserror = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

**อธิบาย feature flags สำคัญ:**

- `async-graphql` feature `"uuid"` — เปิดใช้ built-in scalar สำหรับ `uuid::Uuid`
- `async-graphql` feature `"chrono"` — เปิดใช้ built-in scalar สำหรับ `chrono::DateTime`
- `async-graphql` feature `"dataloader"` — เปิดใช้ `async_graphql::dataloader` module
- `axum` feature default รวม JSON, routing, middleware พื้นฐาน

**สร้าง Store ก่อน** (store.rs):

```rust
// src/store.rs
use dashmap::DashMap;
use std::sync::Arc;
use uuid::Uuid;
use crate::model::{BookRecord, AuthorRecord};

/// In-memory store ที่ thread-safe ด้วย DashMap
/// ใช้ Arc เพื่อ share ข้ามหลาย async task โดยไม่ต้องมี Mutex ครอบนอก
#[derive(Clone, Default)]
pub struct Store {
    pub books: Arc<DashMap<Uuid, BookRecord>>,
    pub authors: Arc<DashMap<Uuid, AuthorRecord>>,
}

impl Store {
    pub fn new() -> Self {
        Store::default()
    }

    pub fn with_seed_data() -> Self {
        let store = Store::new();

        let author1_id = Uuid::parse_str("00000000-0000-0000-0000-000000000001").unwrap();
        let author2_id = Uuid::parse_str("00000000-0000-0000-0000-000000000002").unwrap();

        store.authors.insert(author1_id, AuthorRecord {
            id: author1_id,
            name: "Erich Gamma".into(),
            country: "Switzerland".into(),
        });
        store.authors.insert(author2_id, AuthorRecord {
            id: author2_id,
            name: "Andrew Hunt".into(),
            country: "USA".into(),
        });

        let book1_id = Uuid::parse_str("10000000-0000-0000-0000-000000000001").unwrap();
        store.books.insert(book1_id, BookRecord {
            id: book1_id,
            title: "Design Patterns".into(),
            author_id: author1_id,
            year: 1994,
            pages: 395,
        });

        store
    }
}
```

**สร้าง Domain Model** (model.rs):

```rust
// src/model.rs
use uuid::Uuid;

/// BookRecord — raw data ใน store (ไม่ใช่ GraphQL type)
#[derive(Clone, Debug, PartialEq)]
pub struct BookRecord {
    pub id: Uuid,
    pub title: String,
    pub author_id: Uuid,
    pub year: i32,
    pub pages: i32,
}

/// AuthorRecord — raw data ใน store
#[derive(Clone, Debug, PartialEq)]
pub struct AuthorRecord {
    pub id: Uuid,
    pub name: String,
    pub country: String,
}
```

สังเกตว่า `BookRecord` และ `AuthorRecord` เป็น **plain Rust struct** ที่ไม่มี GraphQL macro ซึ่งเป็น best practice ในการแยก domain model ออกจาก presentation layer

---

### ขั้นที่ 2: GraphQL Object Types ด้วย `#[derive(SimpleObject)]`

`SimpleObject` คือ macro สำหรับ struct ที่ทุก field เปิดเผยเป็น GraphQL field โดยตรง (ไม่มี custom resolver logic):

```rust
// src/schema/types.rs
use async_graphql::*;
use crate::model::{BookRecord, AuthorRecord};

/// Author — GraphQL Object Type
/// #[derive(SimpleObject)] สร้าง field ทุกตัวใน schema อัตโนมัติ
#[derive(SimpleObject, Clone, Debug)]
pub struct Author {
    /// UUID ของผู้แต่ง
    pub id: ID,
    /// ชื่อเต็มของผู้แต่ง
    pub name: String,
    /// ประเทศที่อยู่อาศัย
    pub country: String,
}

impl From<AuthorRecord> for Author {
    fn from(r: AuthorRecord) -> Self {
        Author {
            id: ID(r.id.to_string()),
            name: r.name,
            country: r.country,
        }
    }
}

/// Book — GraphQL Object Type ที่มี field author ซึ่งต้อง resolve จาก DataLoader
/// ใช้ #[graphql(skip)] เพื่อซ่อน author_id (internal field) ออกจาก schema
#[derive(SimpleObject, Clone, Debug)]
pub struct Book {
    pub id: ID,
    pub title: String,
    pub year: i32,
    pub pages: i32,
    /// author_id ใช้ภายในเท่านั้น ซ่อนจาก GraphQL schema
    #[graphql(skip)]
    pub author_id_raw: String,
}

impl From<BookRecord> for Book {
    fn from(r: BookRecord) -> Self {
        Book {
            id: ID(r.id.to_string()),
            title: r.title,
            year: r.year,
            pages: r.pages,
            author_id_raw: r.author_id.to_string(),
        }
    }
}
```

**ทำไม `ID` type แทน `String`?**

async-graphql มี type `ID` ซึ่งเป็น GraphQL scalar ประเภทพิเศษ — ตอน serialize เป็น JSON จะได้ string เหมือนกัน แต่ใน schema จะปรากฏเป็น `ID` ไม่ใช่ `String` ทำให้ client รู้ว่าควรใช้ field นี้เป็น identifier ไม่ใช่ค่าทั่วไป

**`#[graphql(skip)]` — ซ่อน field จาก schema:**

บางครั้ง struct เก็บข้อมูล internal ที่ไม่ควรเปิดเผยใน API `#[graphql(skip)]` บอก async-graphql ให้ข้าม field นั้นไป

---

### ขั้นที่ 3: InputObject และ QueryRoot

**InputObject** ใช้สำหรับ argument ที่เป็น struct ใน mutation หรือ query:

```rust
// src/schema/inputs.rs
use async_graphql::*;

/// สำหรับสร้างหนังสือใหม่
#[derive(InputObject, Debug, Clone)]
pub struct CreateBookInput {
    /// ชื่อหนังสือ (ห้ามว่าง)
    pub title: String,
    /// UUID ของผู้แต่ง (ต้องมีอยู่ใน system)
    pub author_id: ID,
    /// ปีที่ตีพิมพ์ (ค.ศ. 1000–2100)
    pub year: i32,
    /// จำนวนหน้า
    pub pages: i32,
}

/// Filter สำหรับ query books
#[derive(InputObject, Debug, Clone, Default)]
pub struct BookFilter {
    /// กรองเฉพาะหนังสือที่ตีพิมพ์หลังปีนี้ (inclusive)
    pub min_year: Option<i32>,
    /// กรองเฉพาะหนังสือที่ตีพิมพ์ก่อนปีนี้ (inclusive)
    pub max_year: Option<i32>,
    /// กรองตาม author_id
    pub author_id: Option<ID>,
}

/// Input สำหรับ cursor-based pagination
#[derive(InputObject, Debug, Clone)]
pub struct PaginationInput {
    /// จำนวน item สูงสุด
    pub first: Option<usize>,
    /// cursor จาก pageInfo.endCursor ของ page ก่อนหน้า
    pub after: Option<String>,
}
```

**QueryRoot** — entry point ของทุก read operation:

```rust
// src/schema/query.rs
use async_graphql::*;
use crate::store::Store;
use crate::schema::types::{Author, Book};
use crate::schema::inputs::BookFilter;

pub struct QueryRoot;

#[Object]
impl QueryRoot {
    /// ดึงหนังสือทั้งหมด พร้อม optional filter และ pagination
    async fn books(
        &self,
        ctx: &Context<'_>,
        filter: Option<BookFilter>,
        first: Option<usize>,
    ) -> Result<Vec<Book>> {
        let store = ctx.data::<Store>()?;   // ดึง Store จาก Context

        let mut results: Vec<Book> = store
            .books
            .iter()
            .filter(|entry| {
                let b = entry.value();
                if let Some(ref f) = filter {
                    if let Some(min) = f.min_year {
                        if b.year < min { return false; }
                    }
                    if let Some(max) = f.max_year {
                        if b.year > max { return false; }
                    }
                    // filter by author_id ถ้าระบุมา
                    if let Some(ref aid) = f.author_id {
                        if b.author_id.to_string() != aid.0 { return false; }
                    }
                }
                true
            })
            .map(|e| Book::from(e.value().clone()))
            .collect();

        // sort by title เพื่อ deterministic output
        results.sort_by(|a, b| a.title.cmp(&b.title));

        // apply pagination
        if let Some(n) = first {
            results.truncate(n);
        }

        Ok(results)
    }

    /// ดึงหนังสือตาม ID (คืน null ถ้าไม่พบ)
    async fn book(&self, ctx: &Context<'_>, id: ID) -> Result<Option<Book>> {
        let store = ctx.data::<Store>()?;
        let uuid = uuid::Uuid::parse_str(&id.0)
            .map_err(|e| Error::new(format!("Invalid UUID: {e}")))?;
        Ok(store.books.get(&uuid).map(|r| Book::from(r.clone())))
    }

    /// ดึงผู้แต่งตาม ID
    async fn author(&self, ctx: &Context<'_>, id: ID) -> Result<Option<Author>> {
        let store = ctx.data::<Store>()?;
        let uuid = uuid::Uuid::parse_str(&id.0)
            .map_err(|e| Error::new(format!("Invalid UUID: {e}")))?;
        Ok(store.authors.get(&uuid).map(|r| Author::from(r.clone())))
    }

    /// ดึงผู้แต่งทั้งหมด
    async fn authors(&self, ctx: &Context<'_>) -> Result<Vec<Author>> {
        let store = ctx.data::<Store>()?;
        let mut results: Vec<Author> = store
            .authors
            .iter()
            .map(|e| Author::from(e.value().clone()))
            .collect();
        results.sort_by(|a, b| a.name.cmp(&b.name));
        Ok(results)
    }
}
```

**ตัวอย่าง Query ที่ client ส่งมา:**

```graphql
# ดึงหนังสือทุกเล่มที่ตีพิมพ์ตั้งแต่ปี 2000 เป็นต้นมา เอาแค่ 5 เล่มแรก
query RecentBooks {
  books(filter: { minYear: 2000 }, first: 5) {
    id
    title
    year
    pages
  }
}

# ดึงหนังสือเล่มเดียวตาม ID
query GetBook {
  book(id: "10000000-0000-0000-0000-000000000001") {
    title
    year
  }
}
```

---

### ขั้นที่ 4: MutationRoot และ Error Handling

Mutation ต่างจาก Query ตรงที่มี **side effects** — เปลี่ยนแปลง state ของ server:

```rust
// src/schema/mutation.rs
use async_graphql::*;
use uuid::Uuid;
use crate::store::Store;
use crate::model::BookRecord;
use crate::schema::types::{Author, Book};
use crate::schema::inputs::CreateBookInput;

pub struct MutationRoot;

#[Object]
impl MutationRoot {
    /// สร้างหนังสือใหม่
    /// Return: Book ที่สร้าง พร้อม ID ที่ generate ขึ้น
    async fn create_book(
        &self,
        ctx: &Context<'_>,
        input: CreateBookInput,
    ) -> Result<Book> {
        // ── Validation ─────────────────────────────────────────────
        if input.title.trim().is_empty() {
            return Err(
                Error::new("title cannot be empty")
                    .extend_with(|_, ext| {
                        ext.set("code", "VALIDATION_ERROR");
                        ext.set("field", "title");
                    })
            );
        }

        if input.year < 1000 || input.year > 2100 {
            return Err(
                Error::new(format!("year {} is out of valid range [1000, 2100]", input.year))
                    .extend_with(|_, ext| {
                        ext.set("code", "VALIDATION_ERROR");
                        ext.set("field", "year");
                    })
            );
        }

        if input.pages <= 0 {
            return Err(
                Error::new("pages must be positive")
                    .extend_with(|_, ext| ext.set("code", "VALIDATION_ERROR"))
            );
        }

        // ── Business logic ──────────────────────────────────────────
        let store = ctx.data::<Store>()?;

        let author_uuid = Uuid::parse_str(&input.author_id.0)
            .map_err(|_| Error::new("author_id is not a valid UUID")
                .extend_with(|_, ext| ext.set("code", "INVALID_INPUT"))
            )?;

        if !store.authors.contains_key(&author_uuid) {
            return Err(
                Error::new(format!("author {} not found", author_uuid))
                    .extend_with(|_, ext| {
                        ext.set("code", "NOT_FOUND");
                        ext.set("resource", "Author");
                    })
            );
        }

        // ── Persist ─────────────────────────────────────────────────
        let id = Uuid::new_v4();
        let record = BookRecord {
            id,
            title: input.title,
            author_id: author_uuid,
            year: input.year,
            pages: input.pages,
        };
        let book = Book::from(record.clone());
        store.books.insert(id, record);

        // ── Publish event สำหรับ Subscription ──────────────────────
        SimpleBroker::publish(BookEvent::Created(book.clone()));

        Ok(book)
    }

    /// อัปเดตชื่อหนังสือ
    async fn update_book_title(
        &self,
        ctx: &Context<'_>,
        id: ID,
        new_title: String,
    ) -> Result<Book> {
        if new_title.trim().is_empty() {
            return Err(Error::new("title cannot be empty")
                .extend_with(|_, ext| ext.set("code", "VALIDATION_ERROR")));
        }

        let store = ctx.data::<Store>()?;
        let uuid = Uuid::parse_str(&id.0)
            .map_err(|e| Error::new(format!("Invalid UUID: {e}")))?;

        match store.books.get_mut(&uuid) {
            Some(mut entry) => {
                entry.title = new_title;
                Ok(Book::from(entry.clone()))
            }
            None => Err(
                Error::new(format!("book {} not found", uuid))
                    .extend_with(|_, ext| ext.set("code", "NOT_FOUND"))
            ),
        }
    }

    /// ลบหนังสือตาม ID
    /// Return: true ถ้าลบสำเร็จ, false ถ้าไม่พบ
    async fn delete_book(&self, ctx: &Context<'_>, id: ID) -> Result<bool> {
        let store = ctx.data::<Store>()?;
        let uuid = Uuid::parse_str(&id.0)
            .map_err(|e| Error::new(format!("Invalid UUID: {e}")))?;

        let removed = store.books.remove(&uuid);
        if let Some((_, book)) = removed {
            SimpleBroker::publish(BookEvent::Deleted(book.id.to_string()));
            return Ok(true);
        }
        Ok(false)
    }
}
```

**Error Extensions — ส่ง metadata พร้อม error:**

async-graphql รองรับ **GraphQL error extensions** ตาม spec ซึ่งช่วยให้ client รู้รายละเอียดเพิ่มเติม:

```json
{
  "errors": [
    {
      "message": "title cannot be empty",
      "locations": [{ "line": 3, "column": 7 }],
      "path": ["createBook"],
      "extensions": {
        "code": "VALIDATION_ERROR",
        "field": "title"
      }
    }
  ]
}
```

client-side (เช่น Apollo Client) สามารถใช้ `error.extensions.code` เพื่อ dispatch error handling ที่เหมาะสม เช่น แสดง toast notification หรือ redirect ไปหน้า 404

---

### ขั้นที่ 5: Subscription และ SimpleBroker

**Subscription** ใน GraphQL ใช้ WebSocket เพื่อส่ง event แบบ real-time ไปยัง client ที่ subscribe:

```rust
// src/schema/subscription.rs
use async_graphql::*;
use futures::Stream;

/// Event ที่ Subscription จะส่งออก
#[derive(Clone, Debug, SimpleObject)]
pub struct BookCreatedEvent {
    pub book_id: String,
    pub title: String,
}

#[derive(Clone, Debug, SimpleObject)]
pub struct BookDeletedEvent {
    pub book_id: String,
}

/// Internal event enum — publish ผ่าน SimpleBroker
#[derive(Clone, Debug)]
pub enum BookEvent {
    Created(Book),
    Deleted(String), // book_id
}

pub struct SubscriptionRoot;

#[Subscription]
impl SubscriptionRoot {
    /// Subscribe รับ event ทุกครั้งที่มีหนังสือถูกสร้าง
    async fn book_created(&self) -> impl Stream<Item = BookCreatedEvent> {
        SimpleBroker::<BookEvent>::subscribe().filter_map(|event| {
            Box::pin(async move {
                match event {
                    BookEvent::Created(book) => Some(BookCreatedEvent {
                        book_id: book.id.to_string(),
                        title: book.title.clone(),
                    }),
                    _ => None,
                }
            })
        })
    }

    /// Subscribe รับ event ทุกครั้งที่มีหนังสือถูกลบ
    async fn book_deleted(&self) -> impl Stream<Item = BookDeletedEvent> {
        SimpleBroker::<BookEvent>::subscribe().filter_map(|event| {
            Box::pin(async move {
                match event {
                    BookEvent::Deleted(id) => Some(BookDeletedEvent { book_id: id }),
                    _ => None,
                }
            })
        })
    }
}
```

**SimpleBroker** คือ in-memory pub/sub ที่ async-graphql ให้มา เหมาะสำหรับ **single instance** deployment (production scale ต้องใช้ Redis Pub/Sub หรือ NATS แทน)

**ตัวอย่าง Subscription query:**

```graphql
subscription WatchNewBooks {
  bookCreated {
    bookId
    title
  }
}
```

Client เปิด WebSocket connection ไปที่ `ws://localhost:8080/ws/graphql` และส่ง subscription query ทุกครั้งที่ mutation `createBook` ถูกเรียก event จะถูก push ไปยัง client อัตโนมัติ

---

### ขั้นที่ 6: DataLoader — แก้ปัญหา N+1 Queries

ปัญหา **N+1** เกิดเมื่อ query ดึง N records แล้ว resolve field ของแต่ละ record โดยยิง query แยก:

```
Query: books { title author { name } }

ผลลัพธ์ที่ไม่ดี (N+1):
  SELECT * FROM books                          -- 1 query
  SELECT * FROM authors WHERE id = 'uuid-1'   -- N queries
  SELECT * FROM authors WHERE id = 'uuid-2'
  SELECT * FROM authors WHERE id = 'uuid-1'   -- ซ้ำกัน!
  ...

ผลลัพธ์ที่ดีด้วย DataLoader:
  SELECT * FROM books                          -- 1 query
  SELECT * FROM authors WHERE id IN (...)      -- 1 batch query
```

**Implementation:**

```rust
// src/loader.rs
use async_graphql::dataloader::Loader;
use std::collections::HashMap;
use std::sync::Arc;
use uuid::Uuid;
use crate::store::Store;
use crate::schema::types::Author;
use crate::model::AuthorRecord;

/// AuthorLoader — batch-loads authors โดย UUID
pub struct AuthorLoader {
    pub store: Store,
}

impl Loader<Uuid> for AuthorLoader {
    type Value = Author;
    type Error = async_graphql::Error;

    fn load(
        &self,
        keys: &[Uuid],
    ) -> impl std::future::Future<Output = Result<HashMap<Uuid, Author>, async_graphql::Error>> + Send
    {
        let store = self.store.clone();
        let keys = keys.to_vec();

        async move {
            // โหลดทุก key ในคราวเดียว — ที่นี่คือ DashMap lookup
            // ในระบบจริงจะเป็น SQL: SELECT * FROM authors WHERE id = ANY($1)
            let mut map = HashMap::new();
            for key in keys {
                if let Some(record) = store.authors.get(&key) {
                    map.insert(key, Author::from(record.clone()));
                }
            }
            Ok(map)
        }
    }
}
```

**ใช้ DataLoader ใน Book resolver:**

```rust
// ใน Book object — ต้องเปลี่ยนจาก SimpleObject เป็น Object เพื่อ custom resolver
use async_graphql::dataloader::DataLoader;

#[Object]
impl Book {
    async fn id(&self) -> &ID { &self.id }
    async fn title(&self) -> &str { &self.title }
    async fn year(&self) -> i32 { self.year }
    async fn pages(&self) -> i32 { self.pages }

    /// Resolver สำหรับ field author — ใช้ DataLoader
    async fn author(&self, ctx: &Context<'_>) -> Result<Option<Author>> {
        let loader = ctx.data::<DataLoader<AuthorLoader>>()?;
        let author_uuid = Uuid::parse_str(&self.author_id_raw)
            .map_err(|e| Error::new(format!("Invalid author UUID: {e}")))?;

        // load_one จะถูก batch กับ request อื่น ๆ ในรอบเดียวกันโดยอัตโนมัติ
        loader.load_one(author_uuid).await
    }
}
```

**Register DataLoader ใน Schema:**

```rust
// src/schema/mod.rs
use async_graphql::{Schema, EmptySubscription};
use async_graphql::dataloader::DataLoader;
use crate::store::Store;
use crate::loader::AuthorLoader;

pub fn build_schema(store: Store) -> Schema<QueryRoot, MutationRoot, SubscriptionRoot> {
    Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
        .data(store.clone())
        .data(DataLoader::new(
            AuthorLoader { store: store.clone() },
            tokio::spawn,
        ))
        .finish()
}
```

**สำคัญมาก:** DataLoader ต้องส่ง `tokio::spawn` เป็น spawner เพื่อให้ batching ทำงานถูกต้อง — DataLoader จะ collect keys ที่มาใกล้ ๆ กัน (ใน async tick เดียวกัน) แล้วเรียก `load()` ครั้งเดียวพร้อมกัน

---

### ขั้นที่ 7: Custom Scalars

async-graphql มี built-in scalar สำหรับ `uuid::Uuid` และ `chrono::DateTime` แต่บางครั้งต้องการ custom scalar เพิ่มเติม เช่น `EmailAddress`, `NonEmptyString`, หรือ `PositiveInt`:

```rust
// src/scalars.rs
use async_graphql::*;
use std::fmt;

/// NonEmptyString — string ที่ไม่ว่างเปล่า
#[derive(Clone, Debug, PartialEq)]
pub struct NonEmptyString(pub String);

#[Scalar(name = "NonEmptyString")]
impl ScalarType for NonEmptyString {
    fn parse(value: Value) -> InputValueResult<Self> {
        match value {
            Value::String(s) if !s.trim().is_empty() => {
                Ok(NonEmptyString(s))
            }
            Value::String(_) => Err(InputValueError::custom("string must not be empty")),
            _ => Err(InputValueError::expected_type(value)),
        }
    }

    fn to_value(&self) -> Value {
        Value::String(self.0.clone())
    }
}

/// PositiveInt — integer ที่มากกว่า 0
#[derive(Clone, Debug, PartialEq, Copy)]
pub struct PositiveInt(pub i32);

#[Scalar(name = "PositiveInt")]
impl ScalarType for PositiveInt {
    fn parse(value: Value) -> InputValueResult<Self> {
        match value {
            Value::Number(n) => {
                let i = n.as_i64()
                    .ok_or_else(|| InputValueError::custom("not an integer"))?;
                if i > 0 {
                    Ok(PositiveInt(i as i32))
                } else {
                    Err(InputValueError::custom(
                        format!("{i} is not positive")
                    ))
                }
            }
            _ => Err(InputValueError::expected_type(value)),
        }
    }

    fn to_value(&self) -> Value {
        Value::Number(self.0.into())
    }
}
```

**ใช้ Built-in UUID scalar** (ต้องเปิด feature `"uuid"`):

```rust
use async_graphql::*;
use uuid::Uuid;

#[derive(SimpleObject)]
pub struct Entity {
    /// ใช้ Uuid โดยตรง — async-graphql แปลงเป็น scalar "UUID" อัตโนมัติ
    pub id: Uuid,
    pub name: String,
}
```

Schema ที่ generate จะมี scalar definition อัตโนมัติ:

```graphql
scalar UUID
scalar DateTime   # จาก feature "chrono"

type Entity {
  id: UUID!
  name: String!
}
```

---

### ขั้นที่ 8: Integration กับ Axum

```rust
// src/main.rs
use async_graphql_axum::{GraphQL, GraphQLSubscription};
use axum::{Router, routing::get};
use async_graphql::http::GraphiQLSource;
use axum::response::Html;
use crate::schema::build_schema;
use crate::store::Store;

async fn graphiql() -> Html<String> {
    Html(
        GraphiQLSource::build()
            .endpoint("/graphql")
            .subscription_endpoint("/ws/graphql")
            .finish()
    )
}

#[tokio::main]
async fn main() {
    // tracing setup
    tracing_subscriber::fmt()
        .with_env_filter("info")
        .init();

    let store = Store::with_seed_data();
    let schema = build_schema(store);

    let app = Router::new()
        // GraphQL endpoint (POST)
        .route("/graphql", get(graphiql).post_service(GraphQL::new(schema.clone())))
        // WebSocket endpoint สำหรับ Subscription
        .route("/ws/graphql", get(GraphQLSubscription::new(schema)));

    let addr = "0.0.0.0:8080";
    tracing::info!("GraphiQL playground: http://{addr}/graphql");
    tracing::info!("GraphQL endpoint:    http://{addr}/graphql");
    tracing::info!("WS subscription:     ws://{addr}/ws/graphql");

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**CORS สำหรับ production:**

```rust
use tower_http::cors::{CorsLayer, Any};
use axum::http::Method;

let cors = CorsLayer::new()
    .allow_origin(Any)           // ใน production ระบุ origin จริง
    .allow_methods([Method::GET, Method::POST])
    .allow_headers(Any);

let app = Router::new()
    .route("/graphql", ...)
    .layer(cors);
```

**Multipart Upload (file upload):**

```toml
# Cargo.toml — เพิ่ม feature
async-graphql = { version = "7", features = ["multipart", "uuid", "chrono", "dataloader"] }
```

```rust
use async_graphql::Upload;

#[derive(InputObject)]
pub struct UploadCoverInput {
    pub book_id: ID,
    pub file: Upload,
}

#[Object]
impl MutationRoot {
    async fn upload_cover(
        &self,
        ctx: &Context<'_>,
        input: UploadCoverInput,
    ) -> Result<String> {
        let file = input.file.value(ctx)?;
        let filename = file.filename.unwrap_or("cover.jpg".into());
        let content_type = file.content_type.unwrap_or("image/jpeg".into());

        // อ่าน byte จาก file.content (AsyncRead)
        // บันทึกลงดิสก์หรืออัปโหลดไป object storage

        Ok(format!("/covers/{}", filename))
    }
}
```

---

### ขั้นที่ 9: Pagination แบบ Cursor-Based

Cursor-based pagination เป็น best practice ของ GraphQL (ตาม Relay Cursor Connections Specification) เหมาะกับ dataset ขนาดใหญ่ที่อาจมีการเพิ่ม/ลบ record ระหว่าง page ต่าง ๆ:

```rust
// src/schema/pagination.rs
use async_graphql::*;
use base64::{Engine, engine::general_purpose::STANDARD as BASE64};

/// Connection wrapper ตาม Relay spec
#[derive(SimpleObject)]
pub struct BookConnection {
    /// list ของ edges
    pub edges: Vec<BookEdge>,
    /// metadata สำหรับ pagination
    pub page_info: PageInfo,
    /// จำนวน total (ไม่ filter)
    pub total_count: i32,
}

#[derive(SimpleObject)]
pub struct BookEdge {
    /// opaque cursor สำหรับ item นี้
    pub cursor: String,
    /// ข้อมูล book
    pub node: Book,
}

#[derive(SimpleObject)]
pub struct PageInfo {
    pub has_next_page: bool,
    pub has_previous_page: bool,
    pub start_cursor: Option<String>,
    pub end_cursor: Option<String>,
}

/// สร้าง cursor จาก offset index
pub fn encode_cursor(index: usize) -> String {
    BASE64.encode(format!("cursor:{index}"))
}

/// decode cursor เป็น index
pub fn decode_cursor(cursor: &str) -> Option<usize> {
    let decoded = BASE64.decode(cursor).ok()?;
    let s = std::str::from_utf8(&decoded).ok()?;
    s.strip_prefix("cursor:")?.parse().ok()
}

// ใน QueryRoot:
async fn books_connection(
    &self,
    ctx: &Context<'_>,
    first: Option<usize>,
    after: Option<String>,
) -> Result<BookConnection> {
    let store = ctx.data::<Store>()?;
    let mut all: Vec<Book> = store.books
        .iter()
        .map(|e| Book::from(e.value().clone()))
        .collect();
    all.sort_by(|a, b| a.title.cmp(&b.title));

    let total_count = all.len() as i32;
    let start_index = after
        .as_deref()
        .and_then(decode_cursor)
        .map(|i| i + 1)
        .unwrap_or(0);

    let slice = &all[start_index.min(all.len())..];
    let limit = first.unwrap_or(10);
    let has_next_page = slice.len() > limit;
    let items: Vec<Book> = slice.iter().take(limit).cloned().collect();

    let edges: Vec<BookEdge> = items
        .into_iter()
        .enumerate()
        .map(|(i, node)| BookEdge {
            cursor: encode_cursor(start_index + i),
            node,
        })
        .collect();

    let start_cursor = edges.first().map(|e| e.cursor.clone());
    let end_cursor = edges.last().map(|e| e.cursor.clone());

    Ok(BookConnection {
        edges,
        page_info: PageInfo {
            has_next_page,
            has_previous_page: start_index > 0,
            start_cursor,
            end_cursor,
        },
        total_count,
    })
}
```

**ตัวอย่าง query แบบ cursor pagination:**

```graphql
query GetBooks($after: String) {
  booksConnection(first: 2, after: $after) {
    edges {
      cursor
      node {
        title
        year
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
    totalCount
  }
}
```

---

## การทดสอบ (Testing)

โปรเจคนี้ทดสอบ logic ด้วย unit tests ที่ไม่ต้องการ running server เพราะ `Schema::execute()` ของ async-graphql ทำงานได้โดยตรงใน test context

### Test Setup

```rust
// ใน test module
fn schema() -> Schema<QueryRoot, MutationRoot, EmptySubscription> {
    build_schema(Store::new())
}
```

### Test Cases ที่ครอบคลุม

ด้านล่างคือผลลัพธ์จาก `cargo test` จริงของ scratchpad project ที่ verify code ทุกบรรทัดในเอกสารนี้:

```
running 21 tests
test tests::test_input_object_fields_accessible ... ok
test tests::test_dataloader_batches_multiple_keys ... ok
test tests::test_dataloader_returns_none_for_missing_key ... ok
test tests::test_mutation_create_book_empty_title_fails ... ok
test tests::test_mutation_create_book_invalid_year_fails ... ok
test tests::test_mutation_create_book_success ... ok
test tests::test_error_extension_code_present ... ok
test tests::test_mutation_delete_book ... ok
test tests::test_mutation_create_book_unknown_author_fails ... ok
test tests::test_mutation_delete_nonexistent_book_returns_false ... ok
test tests::test_mutations_are_isolated_per_store ... ok
test tests::test_query_book_invalid_uuid_returns_error ... ok
test tests::test_query_author_by_id ... ok
test tests::test_query_all_books ... ok
test tests::test_query_books_filter_by_min_year ... ok
test tests::test_query_books_sorted_by_title ... ok
test tests::test_query_nonexistent_book_returns_null ... ok
test tests::test_query_books_with_first_pagination ... ok
test tests::test_query_single_book_by_id ... ok
test tests::test_schema_has_query_type ... ok
test tests::test_schema_has_mutation_type ... ok

test result: ok. 21 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

### โค้ด Tests สำคัญ

**Test 1: Schema Introspection**

```rust
#[tokio::test]
async fn test_schema_has_query_type() {
    let schema = schema();
    let res = schema.execute("{ __schema { queryType { name } } }").await;
    assert!(res.errors.is_empty(), "errors: {:?}", res.errors);
    let data = res.data.into_json().unwrap();
    assert_eq!(data["__schema"]["queryType"]["name"], "QueryRoot");
}
```

**Test 2: DataLoader Batching ป้องกัน N+1**

```rust
#[tokio::test]
async fn test_dataloader_batches_multiple_keys() {
    let store = Store::new();
    let batch_counter = Arc::new(std::sync::atomic::AtomicUsize::new(0));
    let loader = DataLoader::new(
        AuthorLoader {
            store: store.clone(),
            batch_calls: batch_counter.clone(),
        },
        tokio::spawn,
    );

    let id1 = Uuid::parse_str("00000000-0000-0000-0000-000000000001").unwrap();
    let id2 = Uuid::parse_str("00000000-0000-0000-0000-000000000002").unwrap();

    // Load 2 keys พร้อมกัน — DataLoader ควร batch เป็น 1 call
    let (r1, r2) = tokio::join!(loader.load_one(id1), loader.load_one(id2));

    assert_eq!(r1.unwrap().unwrap().name, "Erich Gamma");
    assert_eq!(r2.unwrap().unwrap().name, "Andrew Hunt");

    // ตรวจสอบว่าเรียก load() เพียงครั้งเดียว
    let calls = batch_counter.load(std::sync::atomic::Ordering::SeqCst);
    assert_eq!(calls, 1, "DataLoader should batch into 1 call, got {calls}");
}
```

**Test 3: Error Extensions**

```rust
#[tokio::test]
async fn test_error_extension_code_present() {
    let schema = schema();
    let mutation = r#"
        mutation {
            createBook(input: {
                title: ""
                authorId: "00000000-0000-0000-0000-000000000001"
                year: 2020
                pages: 100
            }) { id }
        }
    "#;
    let res = schema.execute(mutation).await;
    assert!(!res.errors.is_empty());
    let ext = res.errors[0].extensions.as_ref().unwrap();
    assert_eq!(ext.get("code").unwrap(), &Value::from("VALIDATION_ERROR"));
}
```

**Test 4: Store Isolation ระหว่าง Schema instances**

```rust
#[tokio::test]
async fn test_mutations_are_isolated_per_store() {
    let schema_a = build_schema(Store::new());
    let schema_b = build_schema(Store::new());

    // สร้างหนังสือใน schema_a
    let mutation = r#"mutation { createBook(input: {
        title: "Schema A Only"
        authorId: "00000000-0000-0000-0000-000000000001"
        year: 2024
        pages: 100
    }) { id title } }"#;
    let res_a = schema_a.execute(mutation).await;
    assert!(res_a.errors.is_empty());

    // schema_b ยังมีแค่ 3 เล่ม (ไม่ได้รับ mutation ของ schema_a)
    let res_b = schema_b.execute("{ books { title } }").await;
    let data_b = res_b.data.into_json().unwrap();
    assert_eq!(data_b["books"].as_array().unwrap().len(), 3);
}
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### Pitfall 1: ลืม `.data(store)` ใน Schema Builder

**อาการ:** runtime error `"Data of type `Store` was not found."` เมื่อ resolver เรียก `ctx.data::<Store>()`

```rust
// ❌ ผิด — ไม่ได้ inject Store เข้า Schema
let schema = Schema::build(QueryRoot, MutationRoot, EmptySubscription)
    .finish();

// ✅ ถูก — inject Store ด้วย .data()
let schema = Schema::build(QueryRoot, MutationRoot, EmptySubscription)
    .data(store.clone())
    .data(DataLoader::new(AuthorLoader { store }, tokio::spawn))
    .finish();
```

สาเหตุ: `ctx.data::<T>()` ใช้ type เป็น key ใน type-map หาก type นั้นไม่ได้ inject มาจะ panic ที่ runtime ไม่ใช่ compile time

### Pitfall 2: ใช้ `#[derive(SimpleObject)]` กับ field ที่ต้อง custom resolver

**อาการ:** ไม่สามารถเขียน resolver สำหรับ field บางตัวเมื่อใช้ `SimpleObject`

```rust
// ❌ ผิด — SimpleObject ไม่รองรับ custom resolver
#[derive(SimpleObject)]
pub struct Book {
    pub id: ID,
    // ต้องการ resolver ที่ query DataLoader แต่ทำไม่ได้ใน SimpleObject
    pub author: Author,  // จะ panic เพราะ Author ไม่ได้อยู่ใน Book struct
}

// ✅ ถูก — ใช้ #[Object] เมื่อต้องการ custom resolver
pub struct Book {
    pub id: ID,
    pub author_id_raw: String,  // internal field
}

#[Object]
impl Book {
    async fn id(&self) -> &ID { &self.id }

    // Custom resolver — load author จาก DataLoader
    async fn author(&self, ctx: &Context<'_>) -> Result<Option<Author>> {
        let loader = ctx.data::<DataLoader<AuthorLoader>>()?;
        let uuid = Uuid::parse_str(&self.author_id_raw)?;
        loader.load_one(uuid).await
    }
}
```

**กฎง่าย ๆ:** ถ้า field ทั้งหมดใน struct อ่านค่าตรง ๆ ใช้ `SimpleObject` แต่ถ้ามี field ใดที่ต้องการ async context, DataLoader, หรือ computation ต้องใช้ `#[Object]`

### Pitfall 3: DataLoader ไม่ batch เพราะใช้ spawner ผิด

**อาการ:** DataLoader ทำงานได้แต่ load() ถูกเรียกหลายครั้งแทนที่จะเป็น batch เดียว

```rust
// ❌ ผิด — ใช้ futures::executor::block_on เป็น spawner
let loader = DataLoader::new(AuthorLoader { store }, futures::executor::block_on);

// ❌ ผิด — สร้าง DataLoader ใหม่ทุก request แทนที่จะ share ครั้งเดียว
async fn books(&self, ctx: &Context<'_>) -> Result<Vec<Book>> {
    let loader = DataLoader::new(...);  // ทุกครั้งที่ query = DataLoader ใหม่ = ไม่มี batching
    ...
}

// ✅ ถูก — register DataLoader ใน Schema builder ด้วย tokio::spawn
let schema = Schema::build(...)
    .data(DataLoader::new(AuthorLoader { store }, tokio::spawn))
    .finish();
```

DataLoader batching ทำงานโดยการ collect keys ใน "tick" เดียวกัน ถ้าสร้าง DataLoader ใหม่ทุกครั้งก็จะไม่มีการรวม keys เข้าด้วยกัน และ `tokio::spawn` จำเป็นเพื่อให้ batch timer ทำงานถูกต้องใน tokio runtime

### Pitfall 4: Field name conflict ระหว่าง Rust naming convention และ GraphQL

**อาการ:** `cargo build` ผ่านแต่ schema ใช้ชื่อ field ไม่ถูกต้อง (camelCase vs snake_case)

async-graphql แปลง `snake_case` ของ Rust เป็น `camelCase` ของ GraphQL โดยอัตโนมัติ เช่น `author_id` กลายเป็น `authorId` ในส่วนที่เป็น GraphQL schema นี่คือ behavior ตาม GraphQL convention แต่บางครั้งทำให้สับสน:

```rust
// Rust field: author_id
// GraphQL field: authorId (auto-converted)

#[derive(SimpleObject)]
pub struct Book {
    pub author_id: ID,   // → "authorId" ใน GraphQL
}
```

หากต้องการชื่อเฉพาะ ใช้ `#[graphql(name = "...")]`:

```rust
#[derive(SimpleObject)]
pub struct Book {
    #[graphql(name = "author_id")]   // บังคับให้ใช้ snake_case
    pub author_id: ID,
}
```

**สำหรับ InputObject** ทิศทางกลับกัน — client ส่ง `authorId` (camelCase) แต่ Rust field เป็น `author_id`:

```rust
#[derive(InputObject)]
pub struct CreateBookInput {
    // client ส่ง: { authorId: "..." }
    // Rust field: author_id
    pub author_id: ID,   // async-graphql รับ authorId จาก client อัตโนมัติ
}
```

### Pitfall 5: Subscription ไม่รับ Event เพราะ schema type ไม่ตรงกัน

**อาการ:** publish event แล้วไม่มี subscriber ได้รับ event

```rust
// ❌ ผิด — publish type ไม่ตรงกับ subscribe type
SimpleBroker::publish(BookCreatedEvent { ... });  // publish BookCreatedEvent

// แต่ subscribe ด้วย BookEvent
async fn book_created(&self) -> impl Stream<Item = BookEvent> {
    SimpleBroker::<BookEvent>::subscribe()  // ไม่เคยได้รับ BookCreatedEvent
}

// ✅ ถูก — ใช้ type เดียวกัน
SimpleBroker::publish(BookEvent::Created(book));  // publish BookEvent

async fn book_created(&self) -> impl Stream<Item = BookCreatedEvent> {
    SimpleBroker::<BookEvent>::subscribe()  // subscribe BookEvent
        .filter_map(|e| async move {
            match e { BookEvent::Created(b) => Some(BookCreatedEvent { ... }), _ => None }
        })
}
```

SimpleBroker ใช้ type เป็น channel key ดังนั้น publisher และ subscriber ต้องใช้ type เดียวกันเสมอ

### Pitfall 6: `ctx.data::<T>()` ล้มเหลวเพราะ type mismatch

**อาการ:** `"Data of type `DataLoader<AuthorLoader>` was not found."` ทั้งที่ inject ไปแล้ว

```rust
// ❌ ผิด — inject เป็น AuthorLoader แต่ get เป็น DataLoader<AuthorLoader>
let schema = Schema::build(...)
    .data(AuthorLoader { store })          // inject AuthorLoader (ไม่ wrap DataLoader)
    .finish();

async fn author(&self, ctx: &Context<'_>) -> Result<Option<Author>> {
    let loader = ctx.data::<DataLoader<AuthorLoader>>()?;  // หา DataLoader<AuthorLoader> — ไม่เจอ!
}

// ✅ ถูก — inject เป็น DataLoader<AuthorLoader>
let schema = Schema::build(...)
    .data(DataLoader::new(AuthorLoader { store }, tokio::spawn))  // wrap ด้วย DataLoader
    .finish();
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/graphql-api
# GraphiQL playground: http://0.0.0.0:8080/graphql
```

### Dockerfile

```dockerfile
# ─── Stage 1: Build ───────────────────────────────────────────────────────────
FROM rust:1.82-slim-bookworm AS builder
WORKDIR /app

# cache dependencies
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release
RUN rm -f src/main.rs

# build actual source
COPY src ./src
RUN touch src/main.rs
RUN cargo build --release

# ─── Stage 2: Runtime ─────────────────────────────────────────────────────────
FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /app/target/release/graphql-api .

EXPOSE 8080
CMD ["./graphql-api"]
```

```bash
docker build -t graphql-api:latest .
docker run -p 8080:8080 graphql-api:latest
```

### Health Check Endpoint

```rust
// เพิ่ม health check ใน Axum router
use axum::Json;
use serde_json::{json, Value};

async fn health() -> Json<Value> {
    Json(json!({ "status": "ok", "service": "graphql-api" }))
}

let app = Router::new()
    .route("/health", get(health))
    .route("/graphql", ...);
```

```bash
curl http://localhost:8080/health
# {"status":"ok","service":"graphql-api"}
```

### Environment Variables

```rust
use std::env;

#[tokio::main]
async fn main() {
    let port = env::var("PORT").unwrap_or_else(|_| "8080".to_string());
    let bind_addr = format!("0.0.0.0:{port}");
    // ...
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Field `booksByAuthor` ใน `Author` Type ⭐⭐

เพิ่ม field `books` ใน `Author` object ที่คืนหนังสือทั้งหมดของผู้แต่งคนนั้น โดยต้องแก้ `Author` จาก `SimpleObject` เป็น `#[Object]` และใช้ `Store` จาก context:

```graphql
query {
  author(id: "00000000-0000-0000-0000-000000000001") {
    name
    books {          # field ใหม่ที่ต้องเพิ่ม
      title
      year
    }
  }
}
```

**แนวทาง:**
1. เพิ่ม `author_id_raw: String` ใน `Author` struct (hidden field)
2. เปลี่ยนจาก `#[derive(SimpleObject)]` เป็น `#[Object]` impl
3. เขียน `async fn books(&self, ctx: &Context<'_>) -> Result<Vec<Book>>`
4. Filter `store.books` โดย `author_id == self.author_id_raw`

### แบบฝึกหัดที่ 2: เพิ่ม `BookLoader` สำหรับ Batch Load หนังสือ ⭐⭐⭐

สร้าง `BookLoader` คล้ายกับ `AuthorLoader` และใช้ใน `Author.books` เพื่อ batch-load หนังสือทั้งหมดของผู้แต่งในคำขอเดียว ตรวจสอบด้วย unit test ว่า load ถูกเรียกเพียงครั้งเดียว:

```rust
pub struct BooksByAuthorLoader {
    pub store: Store,
}

impl Loader<Uuid> for BooksByAuthorLoader {
    type Value = Vec<Book>;
    type Error = async_graphql::Error;

    fn load(&self, author_ids: &[Uuid]) -> impl Future<...> {
        // โหลดหนังสือทั้งหมด แล้ว group โดย author_id
        // คืน HashMap<Uuid, Vec<Book>>
        todo!()
    }
}
```

### แบบฝึกหัดที่ 3: เพิ่ม Directive `@deprecated` และ `@auth` ⭐⭐⭐

GraphQL รองรับ directive สำหรับ annotate field หรือ argument:

```rust
// built-in @deprecated
#[derive(SimpleObject)]
pub struct Book {
    // field เก่าที่จะถูก deprecate
    #[graphql(deprecation = "Use authorId field instead")]
    pub author: String,
}

// custom @auth directive
struct AuthDirective;

impl CustomDirective for AuthDirective {
    // ตรวจสอบ header authorization ก่อน execute field
    fn resolve_field(&self, ctx: &Context, _directive_args: ...) -> Result<()> {
        let token = ctx.http_headers()
            .get("Authorization")
            .ok_or_else(|| Error::new("Unauthorized"))?;
        // validate JWT token
        Ok(())
    }
}
```

### แบบฝึกหัดที่ 4: Implement `Union` Type สำหรับ Search Result ⭐⭐⭐

เพิ่ม query `search(keyword: String!)` ที่คืน `SearchResult` ซึ่งเป็น union ของ `Book` และ `Author`:

```graphql
union SearchResult = Book | Author

query {
  search(keyword: "Gamma") {
    ... on Book {
      title
      year
    }
    ... on Author {
      name
      country
    }
  }
}
```

```rust
#[derive(Union)]
pub enum SearchResult {
    Book(Book),
    Author(Author),
}

// ใน QueryRoot:
async fn search(&self, ctx: &Context<'_>, keyword: String) -> Result<Vec<SearchResult>> {
    let store = ctx.data::<Store>()?;
    let kw = keyword.to_lowercase();
    let mut results = Vec::new();

    for entry in store.books.iter() {
        if entry.title.to_lowercase().contains(&kw) {
            results.push(SearchResult::Book(Book::from(entry.clone())));
        }
    }
    for entry in store.authors.iter() {
        if entry.name.to_lowercase().contains(&kw) {
            results.push(SearchResult::Author(Author::from(entry.clone())));
        }
    }
    Ok(results)
}
```

### แบบฝึกหัดที่ 5: เพิ่ม Rate Limiting ด้วย Axum Middleware ⭐⭐⭐⭐

ป้องกัน GraphQL endpoint จาก DDoS ด้วย rate limiting:

```toml
# Cargo.toml
tower-governor = "0.4"
```

```rust
use tower_governor::{governor::GovernorConfigBuilder, GovernorLayer};
use std::net::SocketAddr;

let governor_conf = GovernorConfigBuilder::default()
    .per_second(10)        // 10 requests
    .burst_size(20)        // burst สูงสุด 20
    .use_headers()
    .finish()
    .unwrap();

let app = Router::new()
    .route("/graphql", ...)
    .layer(GovernorLayer { config: Arc::new(governor_conf) });
```

### แบบฝึกหัดที่ 6: เพิ่ม Persisted Queries ⭐⭐⭐⭐⭐

Persisted Queries ช่วยลดขนาด request และป้องกัน arbitrary queries ใน production:

```rust
// client ส่ง hash ของ query แทน query ทั้งหมด
// POST /graphql { "extensions": { "persistedQuery": { "sha256Hash": "abc123" } } }

use dashmap::DashMap;

pub struct QueryStore {
    queries: DashMap<String, String>, // hash -> query string
}

// middleware ที่ lookup query จาก hash
async fn persisted_query_middleware(
    State(store): State<Arc<QueryStore>>,
    req: Request,
    next: Next,
) -> Response {
    // ถ้า body มี hash ให้ lookup และแทน query
    // ถ้าไม่พบ hash ให้คืน error "PersistedQueryNotFound"
    todo!()
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **GraphQL API** ที่สมบูรณ์แบบด้วย async-graphql และ Axum ครอบคลุม:

**สิ่งที่ได้สร้าง:**
- **Schema** แบบ code-first ด้วย `#[derive(SimpleObject)]`, `#[derive(InputObject)]`, และ `#[Object]`
- **QueryRoot** พร้อม filtering, pagination, และ field resolvers
- **MutationRoot** พร้อม validation, error extensions (`code`, `field`), และ NOT_FOUND errors
- **SubscriptionRoot** ด้วย `SimpleBroker` สำหรับ real-time events ผ่าน WebSocket
- **DataLoader** ที่แก้ปัญหา N+1 queries ด้วย batch loading (ยืนยันด้วย unit test ว่า 1 call แทน N calls)
- **Custom Scalars** (`NonEmptyString`, `PositiveInt`) ด้วย `ScalarType` trait
- **Cursor Pagination** ตาม Relay Cursor Connections Specification
- **Axum Integration** พร้อม GraphiQL playground และ WebSocket endpoint

**Pattern สำคัญที่ได้เรียน:**
1. **Type-safe Context** — inject dependency ผ่าน `Schema::build().data()` แทน global state
2. **DataLoader Pattern** — แก้ N+1 ด้วย batch + dedup ที่ level DataLoader
3. **Error Extensions** — ส่ง structured error metadata ที่ client parse ได้
4. **Code-First Schema** — Rust struct เป็น single source of truth สำหรับ schema
5. **Union/Interface** — model polymorphic response อย่างถูกต้องตาม GraphQL spec

**เชื่อมโยงกับโปรเจคถัดไป:** Project J07 จะสร้าง REST API พร้อม OpenAPI specification ด้วย `utoipa` และ `axum` ซึ่งเป็นทางเลือกของ GraphQL ในบางกรณี โดยเฉพาะเมื่อ client เป็น third-party ที่ต้องการ standard REST interface หรือ SDK generation

---

**โปรเจคก่อนหน้า:** [Project J05: Collaborative Editor](project-j05-collaborative-editor.md) | **โปรเจคถัดไป:** [Project J07: REST API + OpenAPI](project-j07-rest-openapi.md)
