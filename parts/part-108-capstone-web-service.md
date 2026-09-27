# Part 108: Capstone: Building a Production-Grade Web Service

> โมดูล: Capstone Projects | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 360 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- ออกแบบและสร้าง **URL shortener service** ระดับ production เต็มรูปแบบด้วย **hexagonal
  architecture** — แยก port (`LinkRepository` trait) ออกจาก adapter (SQLx + Postgres จริง) ตั้งแต่
  บรรทัดแรกของโปรเจกต์ ไม่ใช่ retrofit ทีหลัง และอธิบายได้ว่าทำไม handler ทุกตัวควรเห็นแค่ `Arc<dyn
  LinkRepository>` ไม่ใช่ `PgPool` ตรง ๆ
- implement **collision-safe short code generation** ที่พิสูจน์ความปลอดภัยต่อ concurrency ได้จริง
  ด้วย integration test ที่ยิง 30 task พร้อมกันแข่งกันเขียน code เดียวกัน แล้วยืนยันว่า Postgres
  `ON CONFLICT DO NOTHING` การันตี exactly-one-wins เสมอ
- เดินสายครบ **observability triad** (logs จาก Part 60, metrics จาก Part 98, traces จาก Part 99)
  เข้ากับ service เดียวกัน แล้วยืนยันด้วยการรัน **Postgres, Redis, Prometheus, และ Jaeger จริง**
  พร้อมกันในเครื่องเดียว — เห็น target `UP` ใน Prometheus, เห็น trace จริงใน Jaeger, และเห็นค่า
  metric จริงเปลี่ยนตามที่ยิง request จริง
- ทำ **resilience** ให้ครบสามด้าน — rate limiting ที่ผูก state ไว้ที่ Redis (ใช้ได้ถูกต้องแม้รันหลาย
  instance), timeout ที่ตัด request ค้างอัตโนมัติ, และ graceful shutdown ที่พิสูจน์ด้วยการยิง
  SIGTERM ขณะมี request กำลังทำงานอยู่จริง แล้วยืนยันว่า request นั้นได้รับคำตอบก่อน process จบ
- ทำ **cache-aside caching** ให้ hot path (`GET /{code}`) ด้วย Redis แล้ว**วัดตัวเลขจริง**ว่าเร็วขึ้น
  แค่ไหนด้วย `wrk` — ไม่ใช่แค่พูดว่า "cache ช่วยให้เร็วขึ้น" ลอย ๆ
- วิเคราะห์และแก้ **incident จำลองจริง** (database ช้าชั่วคราวจากการถูก lock) โดยอ่าน log/metric/trace
  ที่ระบบสร้างให้ระหว่างเกิดปัญหา และค้นพบ**บั๊กจริงที่เกิดจากลำดับ middleware** ที่ทำให้ request ที่
  timeout หายไปจาก metrics เงียบ ๆ พร้อมแก้ไขและพิสูจน์ว่าแก้ได้จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 62-66 (Axum: Routing, Handlers, State, Extractors, Middleware, Error Handling)**: บทนี้
  เขียน Axum application เต็มรูปแบบต่อยอดจากพื้นฐานทั้งห้าบทนี้โดยตรง — `Router`, `State<T>`,
  `Path<T>`, `axum::middleware::from_fn`, และ `AppError` ที่ implement `IntoResponse` ล้วนเป็น
  pattern ที่บทเหล่านี้สอนไว้แล้ว บทนี้จะไม่อธิบายกลไกพื้นฐานของ extractor/middleware ซ้ำจากศูนย์
- **Part 70-71 (SQLx: Database และ Migrations)**: `PgPool`, `sqlx::query`/`query_as`, และ
  `sqlx::migrate!` ที่ใช้เต็มบทนี้คือของเดิมจากสองบทนี้เป๊ะ
- **Part 98 (Observability: Metrics ด้วย Prometheus)** และ **Part 99 (Observability: Distributed
  Tracing ด้วย OpenTelemetry)**: บทนี้คือจุดที่**ทั้งสามเสาหลักของ observability มาบรรจบกันในระบบ
  เดียว** — `metrics`/`metrics-exporter-prometheus` จาก Part 98 และ `opentelemetry`/
  `tracing-opentelemetry` จาก Part 99 ถูกใช้ตรง ๆ ไม่มีการสอนซ้ำเรื่อง metric type หรือ span/trace
  concept จากศูนย์ ถ้าสองบทนี้ยังไม่แน่น ให้กลับไปทวนก่อน เพราะบทนี้อ้างอิงกลับไปตลอด
- **Part 106 (Enterprise Design Patterns)**: แนวคิด **hexagonal architecture / ports & adapters**
  ที่บทนี้ใช้เป็นโครงสร้างหลักทั้งบทมาจาก Part 106 — `LinkRepository` trait คือ "port" และ
  `PostgresLinkRepository` คือ "adapter" ตาม pattern เดียวกัน
- **Part 84 (Background Jobs)** และแนวคิด rate limiting ทั่วไป: มีประโยชน์สำหรับเข้าใจ trade-off ของ
  งานที่ทำแบบ fire-and-forget (background click counting ในหัวข้อ 108.7) แต่ไม่ใช่ prerequisite
  บังคับ
- **Part 96 (Docker และ Containerization สำหรับ Rust)**: หัวข้อ 108.10 เขียน `Dockerfile` แบบ
  multi-stage build ต่อยอดจากรูปแบบที่ Part 96 สอนไว้
- **Part 107 (Capstone: Building a Production-Grade CLI Tool)**: บทก่อนหน้านี้ (capstone แรกของสอง
  บทคู่) จบด้วยเครื่องมือ CLI ระดับ production — บทนี้คือ capstone คู่กันฝั่ง **web service** ที่เน้น
  ความสมบูรณ์เชิง operational (observability + resilience) มากกว่าฝั่ง CLI tooling

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้เขียนจากโปรเจกต์ที่ผู้เขียน**สร้างและรันจริง 100%** ใน scratch cargo project แยกไว้นอก repo ของ
หลักสูตร (ไม่กระทบไฟล์ใด ๆ ในคอร์สนี้) ด้วยเวอร์ชัน crate จริงจาก crates.io ณ วันที่เขียน: `axum =
"0.8.9"`, `sqlx = "0.9.0"` (features `runtime-tokio`, `postgres`, `uuid`, `chrono`, `migrate`),
`tokio = "1.53.1"`, `deadpool-redis = "0.23.1"`, `redis = "1.7.1"` (feature `script`), `metrics =
"0.24.6"`, `metrics-exporter-prometheus = "0.18.3"`, `opentelemetry = "0.33.0"`,
`opentelemetry-otlp = "0.33.0"` (feature `grpc-tonic`), `tracing-opentelemetry = "0.34.0"`,
`tower-http = "0.7.1"` (features `trace`, `timeout`, `limit`, `compression-gzip`), `governor =
"0.10.4"`, `rand = "0.10.3"` — และที่สำคัญที่สุด **โครงสร้างพื้นฐานทั้งหมดที่ต้องใช้เป็น process จริง
ในเครื่อง ไม่ใช่ mock**:

- **PostgreSQL 16** รันจริงเป็น native service (`service postgresql start`) — สร้าง role/database
  จริง ยืนยันด้วย `psql` ต่อผ่าน TCP จริง
- **Redis 7** รันจริงเป็น native service (`service redis-server start`) — ยืนยันด้วย `redis-cli
  ping` ได้ `PONG` จริง
- **Prometheus 2.45** ติดตั้งจริงผ่าน `apt-get install prometheus` (เวอร์ชันในเครื่องนี้ไม่มี Docker
  daemon ให้ใช้ — ดูหมายเหตุด้านล่าง) — รันจริงพร้อม `prometheus.yml` ที่ scrape แอปจริง แล้วยืนยัน
  ผ่าน Prometheus HTTP API ว่า target ขึ้นสถานะ `up` จริง
- **Jaeger 1.62 (all-in-one)** ดาวน์โหลด binary release จริงจาก GitHub แล้วรันตรง ๆ (ไม่ผ่าน Docker
  ด้วยเหตุผลเดียวกัน) — ยืนยันด้วยการ query Jaeger's HTTP API เห็น service name และ span จริงที่ตรง
  กับ `#[tracing::instrument]` ในโค้ด
- **`wrk`, `hey`, `apache2-utils`** ติดตั้งจริงผ่าน `apt-get` สำหรับ load testing จริง (ไม่ใช่ตัวเลข
  สมมติ)

**หมายเหตุสำคัญเรื่อง Docker**: สภาพแวดล้อมที่ใช้ตรวจสอบบทนี้เป็น container ที่ไม่มีสิทธิ์รัน Docker
daemon เต็มรูปแบบ (`dockerd` ไม่สามารถ start ได้เพราะ permission ของ container ที่ซ้อนกันอีกชั้น) —
บทนี้จึงรัน Postgres/Redis เป็น native service และรัน Prometheus/Jaeger เป็น binary ตรง ๆ แทนที่จะผ่าน
`docker run` (ต่างจาก Part 98/99 ที่มี Docker daemon ใช้งานได้ในสภาพแวดล้อมของบทนั้น) ผลลัพธ์ที่ได้
**เหมือนกันทุกประการในเชิงพฤติกรรม**เพราะเป็น binary เดียวกันเป๊ะ เพียงแต่วิธีรันต่างกัน — หัวข้อ 108.10
(`Dockerfile`) เขียนตามรูปแบบ multi-stage build มาตรฐานที่ถูกต้องตามหลักการของ Part 96 แต่**ไม่ได้ถูก
`docker build` จริงในการตรวจสอบครั้งนี้** เพราะข้อจำกัดของ sandbox — ทุกส่วนอื่นของบท (compile, unit
test, integration test กับ Postgres จริง, การรันเป็น process จริง, load test จริง, การจำลอง incident
จริง) ถูกตรวจสอบด้วยการรันจริงทั้งหมด ไม่มีตัวเลขใดในบทนี้ที่แต่งขึ้นมาเอง — เมื่อจบการตรวจสอบ ทุก
process, database, และไฟล์ scratch ถูกลบ/หยุดออกจากเครื่องเรียบร้อยแล้ว

## เนื้อหา

### 108.1 ภาพรวมโปรเจกต์: URL Shortener ระดับ Production

โปรเจกต์ของบทนี้คือ **URL shortener** — รับ URL ยาว ๆ แล้วคืน "short code" สั้น ๆ (เช่น `4jqnz5z`) ที่
พอมีคนเปิดจะถูก redirect ไปยัง URL ต้นฉบับ ฟังดูเรียบง่ายในระดับ "เดโม" แต่ในระดับ **production จริง**
มันมี edge case ที่น่าสนใจครบเกือบทุกด้านของ web service:

- **Concurrency correctness**: มีคนสองคนพร้อมกันได้ short code เดียวกันไหม (collision)?
- **Hot path ที่ต้องเร็วที่สุด**: `GET /{code}` ถูกเรียกบ่อยกว่า endpoint สร้างลิงก์เป็นร้อยเท่า (ทุก
  คนที่คลิกลิงก์ที่แชร์ไปคือหนึ่ง request) — ต้อง optimize latency ของ path นี้เป็นพิเศษ
- **Analytics ที่ไม่ควรกระทบ latency ของ hot path**: อยากรู้ "ลิงก์นี้ถูกคลิกกี่ครั้ง" แต่การนับคลิก
  ไม่ควรทำให้ผู้ใช้ที่คลิกลิงก์รอนานขึ้น
- **Abuse mitigation**: ใครก็เข้ามาสร้างลิงก์ได้โดยไม่ต้อง authenticate (ตาม use case ทั่วไปของ url
  shortener สาธารณะ) — ต้องมี rate limiting ป้องกันการยิงสร้างลิงก์รัว ๆ
- **Security**: ถ้ายอมให้สร้างลิงก์ไป URL อะไรก็ได้แบบไม่ตรวจสอบ ระบบจะกลายเป็นเครื่องมือของผู้ไม่
  ประสงค์ดี (open redirect ไปหลอกฟิชชิง, SSRF ไปสอดส่อง internal network ของตัวเอง)

#### Non-Functional Requirements (NFR)

ก่อนเขียนโค้ดบรรทัดแรก ทุกระบบ production ควรมีข้อกำหนดที่ไม่ใช่ฟีเจอร์ (non-functional requirement)
ที่ชัดเจน เพราะมันคือสิ่งที่กำหนด**การตัดสินใจทางเทคนิค**ตลอดทั้งบทนี้:

| ด้าน | ข้อกำหนด | เหตุผล |
|---|---|---|
| **Latency (p99) ของ `GET /{code}`** | < 20ms ที่ traffic ปกติ | นี่คือ hot path ที่ user รอผลลัพธ์อยู่หน้าจอโดยตรง |
| **Availability ของ redirect** | ต้องทำงานต่อได้แม้ cache (Redis) ล่ม | redirect คือฟีเจอร์หลักของระบบ ห้ามพังเพราะ layer เสริม |
| **Consistency ของ short code** | ห้ามมี short code ซ้ำกันสองลิงก์ต่างกันเด็ดขาด | ถ้าซ้ำ ผู้ใช้จะถูก redirect ไปผิดที่ ซึ่งเป็นข้อผิดพลาดที่ยอมรับไม่ได้ |
| **Analytics accuracy** | ยอมรับ "นับคลิกคลาดเคลื่อนได้เล็กน้อย" (at-most-once) เพื่อแลกกับ latency ของ redirect | analytics ไม่ใช่ money-critical ในระบบนี้ ต่างจาก consistency ของ short code ที่ต้อง 100% |
| **Observability** | ต้องรู้ทันทีว่า endpoint ไหนช้า, error rate เท่าไหร่, request หนึ่งตัวไปไหนบ้าง | ต่อยอด Part 98-99 เต็มรูปแบบ |

ข้อสังเกตสำคัญจากตารางนี้: **ไม่ใช่ทุกอย่างต้อง "ถูกต้อง 100%" เท่ากัน** — ระบบจริงต้อง**เลือก**ว่าจุด
ไหนต้องแม่นยำแบบ non-negotiable (short code ไม่ซ้ำ) และจุดไหนยอมรับความคลาดเคลื่อนได้เพื่อแลกกับ
performance (จำนวนคลิก) — การตัดสินใจนี้ปรากฏเป็นโค้ดจริงในหัวข้อ 108.4 และ 108.7

#### API Contract

| Method | Path | คำอธิบาย | Request Body | Response |
|---|---|---|---|---|
| `POST` | `/api/links` | สร้างลิงก์สั้นใหม่ | `{"url": string, "ttl_seconds": number?}` | `201`-เทียบเท่า (`200` จริงในโค้ด) `{code, short_url, long_url, expires_at}` |
| `GET` | `/{code}` | redirect ไปยัง URL ปลายทาง (hot path) | — | `307` + header `Location`, หรือ `404`/`410` |
| `GET` | `/api/links/{code}/stats` | ดูสถิติของลิงก์ | — | `{code, long_url, created_at, expires_at, click_count, is_expired}` |
| `GET` | `/metrics` | Prometheus scrape endpoint | — | Prometheus text format |
| `GET` | `/health/live` | liveness probe | — | `200 ok` |
| `GET` | `/health/ready` | readiness probe | — | `200 ready` หรือ `503` |

#### Data Model

ตารางเดียวที่ต้องมีสำหรับ core feature — เจตนาให้เรียบง่ายที่สุดเท่าที่ยัง demonstrate ประเด็นสำคัญได้
ครบ (ไม่ over-engineer เป็นหลายตารางที่ไม่จำเป็นสำหรับ scope ของบทนี้):

```sql
CREATE TABLE links (
    code        TEXT PRIMARY KEY,   -- short code เช่น "4jqnz5z" — primary key คือกลไก
                                     -- collision-safety หลักของทั้งระบบ (หัวข้อ 108.4)
    long_url    TEXT NOT NULL,      -- URL ปลายทางที่ผ่านการ validate แล้ว (หัวข้อ 108.3)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,        -- NULL แปลว่าไม่มีวันหมดอายุ
    created_by  TEXT,               -- เผื่อขยายเป็น multi-tenant ในอนาคต (ไม่ได้ใช้เต็มรูปแบบในบทนี้)
    click_count BIGINT NOT NULL DEFAULT 0  -- นับคลิกแบบ at-most-once (หัวข้อ 108.7)
);
```

จากจุดนี้ไป บทนี้จะสร้างระบบทั้งหมดขึ้นมาทีละชั้น — เริ่มจากโครงสร้าง hexagonal architecture (108.2),
domain logic กับ validation (108.3), collision-safe code generation (108.4), observability (108.5),
resilience (108.6), caching (108.7), database performance (108.8), security (108.9), deployment
readiness (108.10), แล้วปิดท้ายด้วยการรันระบบทั้งหมดพร้อมกันจริงและจำลอง incident (108.11)

### 108.2 Hexagonal Architecture: แยก Port ออกจาก Adapter ตั้งแต่ต้น

จาก **Part 106** เราได้เรียนหลักการ **hexagonal architecture** (หรือ "ports & adapters") — แนวคิด
หลักคือ business logic ("core" หรือ "application layer") ไม่ควรรู้จักรายละเอียดทางเทคนิคของโลกภายนอก
(database, message queue, HTTP client ของ service อื่น) โดยตรง แต่คุยผ่าน **interface ที่ตัวเองนิยาม
เอง** (เรียกว่า "port") แล้วให้ **adapter** เป็นคนไป implement รายละเอียดจริงทีหลัง

บทนี้ apply หลักการนี้กับ**ที่เก็บข้อมูลลิงก์** — port ชื่อ `LinkRepository`:

```rust
// src/ports.rs
use crate::domain::LinkRecord;
use crate::error::AppError;
use async_trait::async_trait;
use chrono::{DateTime, Utc};

/// `LinkRepository` คือ **port** ของ hexagonal architecture — มันนิยาม "อะไรที่ business logic
/// (application layer ใน `handlers.rs`) ต้องการจากที่เก็บข้อมูล" โดย**ไม่ผูกกับ SQLx, Postgres,
/// หรือ database ตัวไหนเลยแม้แต่คำเดียวในลายเซ็นของ trait นี้** — handler เห็นแค่ interface นี้
/// ผ่าน `Arc<dyn LinkRepository>` ไม่รู้เลยว่าข้างหลังเป็น Postgres จริง, in-memory mock ตอนเทสต์,
/// หรือ MySQL ถ้าวันหนึ่งทีมตัดสินใจย้าย database
///
/// นี่คือหัวใจของการแยก "adapter" (รายละเอียดทางเทคนิคของการเก็บข้อมูล) ออกจาก "core"
/// (กฎทางธุรกิจ เช่น "ต้อง retry เมื่อ code ชนกัน", "code ที่ expire แล้วถือว่าไม่พบ") — ต่อยอด
/// จากรูปแบบเดียวกับที่ Part 106 อธิบายไว้เรื่อง ports & adapters
#[async_trait]
pub trait LinkRepository: Send + Sync {
    /// พยายามสร้างแถวใหม่ด้วย `code` ที่กำหนด — คืน `true` ถ้าสร้างสำเร็จ (ไม่ชนกับ code เดิม)
    /// คืน `false` ถ้า `code` นี้มีอยู่แล้ว (collision) โดย**ไม่ถือว่าเป็น error** เพราะ caller
    /// (application layer) จะเป็นคนตัดสินใจเองว่าจะ retry ด้วย code ใหม่หรือไม่ — สัญญาสำคัญที่
    /// adapter ทุกตัวต้องรักษาไว้: การเช็คนี้ต้อง **atomic ที่ระดับ storage** (ไม่ใช่ "SELECT ก่อน
    /// แล้วค่อย INSERT" ที่มี race condition ระหว่างสองคำสั่ง) ดูหัวข้อ 108.4 ที่พิสูจน์เรื่องนี้จริง
    async fn try_insert(
        &self,
        code: &str,
        long_url: &str,
        expires_at: Option<DateTime<Utc>>,
        created_by: Option<&str>,
    ) -> Result<bool, AppError>;

    /// ค้นหาลิงก์ด้วย short code — คืน `None` ถ้าไม่พบ (ไม่ใช่ error กรณีไม่พบ เพราะ "ไม่พบ" คือ
    /// ผลลัพธ์ปกติที่คาดหวังได้ ไม่ใช่ความผิดปกติของระบบ)
    async fn find_by_code(&self, code: &str) -> Result<Option<LinkRecord>, AppError>;

    /// เพิ่ม click counter ของลิงก์นี้ทีละ 1 แบบ atomic ที่ระดับ database
    /// (`UPDATE ... SET click_count = click_count + 1`) — ไม่ใช่ "อ่านค่ามาแล้วบวกในแอปแล้วเขียน
    /// กลับ" ที่มี race condition ถ้ามีหลาย request มา increment พร้อมกัน
    async fn increment_click(&self, code: &str) -> Result<(), AppError>;

    /// นับจำนวนลิงก์ทั้งหมดในระบบ — ใช้แสดงใน readiness check และแบบฝึกหัด
    async fn count_links(&self) -> Result<i64, AppError>;
}
```

สังเกตว่าลายเซ็นของ method ทุกตัวใน trait นี้พูดภาษาของ**โดเมน** (`try_insert`, `find_by_code`) ไม่ใช่
ภาษาของ SQL (`execute`, `fetch_one`) — นี่คือความแตกต่างสำคัญระหว่าง "wrap `PgPool` ไว้เฉย ๆ" (ซึ่งไม่ใช่
hexagonal architecture จริง แค่ห่อชื่อ) กับ "นิยาม port ตามภาษาของ business logic" (ซึ่งทำให้ adapter
ไหนก็ implement ได้โดยไม่ต้องเปลี่ยนความหมาย)

จากนั้นเขียน **adapter** ที่ implement port นี้จริงด้วย SQLx + Postgres:

```rust
// src/repo_postgres.rs
use crate::domain::LinkRecord;
use crate::error::AppError;
use crate::ports::LinkRepository;
use async_trait::async_trait;
use chrono::{DateTime, Utc};
use sqlx::PgPool;

/// `PostgresLinkRepository` คือ **adapter** ที่ implement port `LinkRepository` จริงด้วย SQLx +
/// Postgres — เป็นจุดเดียวในทั้งแอปที่รู้จัก SQL/connection pool/Postgres-specific behavior
/// (`ON CONFLICT`, index) โดยตรง ถ้าวันหนึ่งต้องย้ายไป database อื่น เขียน adapter ใหม่ตัวเดียวที่
/// implement trait เดียวกันนี้ก็พอ ไม่ต้องแก้ handler แม้แต่บรรทัดเดียว
pub struct PostgresLinkRepository {
    pool: PgPool,
}

impl PostgresLinkRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl LinkRepository for PostgresLinkRepository {
    async fn try_insert(
        &self,
        code: &str,
        long_url: &str,
        expires_at: Option<DateTime<Utc>>,
        created_by: Option<&str>,
    ) -> Result<bool, AppError> {
        // `ON CONFLICT (code) DO NOTHING` คือกลไกที่ทำให้การเช็ค "code นี้ถูกใช้ไปแล้วหรือยัง"
        // เป็น atomic operation เดียวที่ระดับ Postgres เอง — ไม่มีช่องว่างระหว่าง "เช็ค" กับ
        // "เขียน" ให้ request อื่นแซงเข้ามาชนได้เลย (ต่างจากการเขียน `SELECT ... ; INSERT ...`
        // สองคำสั่งแยกกันที่มี race window อยู่ระหว่างสองคำสั่งนั้น) — primary key constraint
        // ของ column `code` คือสิ่งที่ค้ำประกันความถูกต้องนี้ในระดับ storage engine จริง ๆ
        let result = sqlx::query(
            r#"
            INSERT INTO links (code, long_url, expires_at, created_by)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (code) DO NOTHING
            "#,
        )
        .bind(code)
        .bind(long_url)
        .bind(expires_at)
        .bind(created_by)
        .execute(&self.pool)
        .await?;

        // `rows_affected() == 1` แปลว่า insert สำเร็จจริง (ไม่ชน) — `== 0` แปลว่าชนกับ code เดิม
        // และ `DO NOTHING` ทำให้ Postgres ไม่ insert อะไรเลย ไม่ throw error ให้ต้อง catch
        Ok(result.rows_affected() == 1)
    }

    async fn find_by_code(&self, code: &str) -> Result<Option<LinkRecord>, AppError> {
        let record = sqlx::query_as::<_, LinkRecord>(
            r#"
            SELECT code, long_url, created_at, expires_at, created_by, click_count
            FROM links
            WHERE code = $1
            "#,
        )
        .bind(code)
        .fetch_optional(&self.pool)
        .await?;

        Ok(record)
    }

    async fn increment_click(&self, code: &str) -> Result<(), AppError> {
        // atomic ที่ระดับ database — Postgres ล็อกแถวระหว่าง UPDATE เอง ไม่ต้องมี application-level
        // lock แม้จะมีหลาย request พร้อมกันมา increment ลิงก์เดียวกันนี้พร้อม ๆ กัน
        sqlx::query(
            r#"
            UPDATE links SET click_count = click_count + 1 WHERE code = $1
            "#,
        )
        .bind(code)
        .execute(&self.pool)
        .await?;

        Ok(())
    }

    async fn count_links(&self) -> Result<i64, AppError> {
        let (count,): (i64,) = sqlx::query_as("SELECT COUNT(*) FROM links")
            .fetch_one(&self.pool)
            .await?;
        Ok(count)
    }
}
```

และตอน wiring ใน `main()` (จะเห็นเต็มในหัวข้อ 108.6) จุดเดียวที่ประกอบ port กับ adapter เข้าด้วยกันคือ
บรรทัดเดียวนี้:

```rust
let repo: Arc<dyn LinkRepository> = Arc::new(PostgresLinkRepository::new(db_pool));
```

จากบรรทัดนี้ไป **ทั้งแอปไม่มีที่ไหนอ้างถึง `PostgresLinkRepository` หรือ `PgPool` ตรง ๆ อีกเลย** —
`AppState` (โครงสร้างที่ handler ทุกตัวเห็นผ่าน `State<AppState>`) เก็บแค่ `Arc<dyn LinkRepository>`:

```rust
// src/state.rs
use crate::ports::LinkRepository;
use crate::ratelimit::RedisRateLimiter;
use metrics_exporter_prometheus::PrometheusHandle;
use std::sync::Arc;

/// `AppState` คือสิ่งเดียวที่ handler ทุกตัวเห็นผ่าน `State<AppState>` — สังเกตว่า field
/// `repo` เป็น `Arc<dyn LinkRepository>` (trait object ของ **port**) ไม่ใช่ `PgPool` ตรง ๆ —
/// นี่คือจุดที่ hexagonal architecture เกิดขึ้นจริงในโค้ด: handler คุยกับ "port" เท่านั้น
/// ไม่รู้จัก "adapter" (`PostgresLinkRepository`) เลยแม้แต่ชื่อ
#[derive(Clone)]
pub struct AppState {
    pub repo: Arc<dyn LinkRepository>,
    pub redis_pool: deadpool_redis::Pool,
    pub rate_limiter: Arc<RedisRateLimiter>,
    pub prometheus_handle: PrometheusHandle,
    pub public_base_url: String,
}
```

**ประโยชน์ที่จับต้องได้ (ไม่ใช่แค่ทฤษฎี)**: ในหัวข้อ 108.4 ที่จะเขียน integration test พิสูจน์
concurrency safety เราสามารถทดสอบตรงกับ Postgres จริงผ่าน SQL ตรง ๆ (เพราะทดสอบพฤติกรรมของ
`ON CONFLICT` ซึ่งเป็นรายละเอียดของ adapter) แยกขาดจาก unit test ของ `validate_target_url` ในหัวข้อ
108.3 ที่ไม่ต้องแตะ database เลย (เพราะเป็น pure function ของ domain layer) — การแยกชั้นแบบนี้ทำให้
เทสต์แต่ละกลุ่มเร็ว/ช้าตามความจำเป็นจริง ไม่ใช่ทุกเทสต์ต้องรอ database connection เหมือนกันหมด

### 108.3 Domain Logic และ Input Validation: AppError กับการป้องกัน Open Redirect/SSRF

#### AppError: จุดศูนย์กลาง Error ทั้งแอป

ทุก layer (repository, cache, rate limiter, handler) แปลง error ของตัวเองมาเป็น `AppError` ตัวเดียว
แล้วปล่อยให้ `IntoResponse` เป็นจุดเดียวที่ตัดสินใจว่าจะตอบ HTTP status code ไหนกลับไปให้ client —
pattern นี้คือของเดิมจาก Part 66 ที่นำมาขยายให้ครอบคลุมทุก error ที่เกิดขึ้นได้ในระบบนี้:

```rust
// src/error.rs
use axum::http::StatusCode;
use axum::response::{IntoResponse, Response};
use axum::Json;
use serde_json::json;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("invalid url: {0}")]
    InvalidUrl(String),

    #[error("short code not found")]
    NotFound,

    #[error("short code expired")]
    Expired,

    #[error("too many collisions while generating short code")]
    TooManyCollisions,

    #[error("rate limit exceeded")]
    RateLimited { retry_after_secs: u64 },

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("cache error: {0}")]
    Cache(String),

    #[error("internal error: {0}")]
    Internal(#[from] anyhow::Error),

    #[error("service not ready: {0}")]
    NotReady(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // บันทึก metric ตาม error variant ก่อนตอบ response เสมอ — ทำให้ dashboard เห็น
        // "อัตรา error แต่ละประเภท" แยกกันได้โดยไม่ต้อง grep log
        let error_kind = match &self {
            AppError::InvalidUrl(_) => "invalid_url",
            AppError::NotFound => "not_found",
            AppError::Expired => "expired",
            AppError::TooManyCollisions => "too_many_collisions",
            AppError::RateLimited { .. } => "rate_limited",
            AppError::Database(_) => "database",
            AppError::Cache(_) => "cache",
            AppError::Internal(_) => "internal",
            AppError::NotReady(_) => "not_ready",
        };
        metrics::counter!("app_errors_total", "kind" => error_kind).increment(1);

        let (status, message) = match &self {
            AppError::InvalidUrl(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::NotFound => (StatusCode::NOT_FOUND, "short code not found".to_string()),
            AppError::Expired => (StatusCode::GONE, "this link has expired".to_string()),
            AppError::TooManyCollisions => (
                StatusCode::SERVICE_UNAVAILABLE,
                "could not allocate a short code, try again".to_string(),
            ),
            AppError::RateLimited { retry_after_secs } => (
                StatusCode::TOO_MANY_REQUESTS,
                format!("rate limit exceeded, retry after {retry_after_secs}s"),
            ),
            AppError::Database(e) => {
                // ไม่ leak รายละเอียด SQL/connection string กลับไปให้ client แต่ log ฝั่ง
                // server ไว้เต็ม ๆ ผ่าน tracing::error! (ที่ผูกกับ trace_id ของ request นี้)
                tracing::error!(error = %e, "database error");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal server error".to_string(),
                )
            }
            AppError::Cache(e) => {
                tracing::warn!(error = %e, "cache error (degraded, not fatal)");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal server error".to_string(),
                )
            }
            AppError::Internal(e) => {
                tracing::error!(error = %e, "internal error");
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "internal server error".to_string(),
                )
            }
            AppError::NotReady(reason) => (StatusCode::SERVICE_UNAVAILABLE, reason.clone()),
        };

        let body = Json(json!({
            "error": error_kind,
            "message": message,
        }));

        (status, body).into_response()
    }
}
```

สังเกต `AppError::NotReady` ที่ map ไป `503 Service Unavailable` แยกจาก `AppError::Internal` ที่ map
ไป `500` — ความแตกต่างนี้สำคัญมากสำหรับหัวข้อ 108.10 (liveness vs readiness): **503 บอก load
balancer/Kubernetes ว่า "อย่าส่ง traffic มาที่ instance นี้ชั่วคราว" ในขณะที่ 500 บอกว่า "เกิด error ที่
ไม่คาดคิดขณะประมวลผล request นี้"** — ความหมายต่างกันโดยสิ้นเชิง แม้ทั้งคู่จะ "แปลว่ามีปัญหา" ก็ตาม

#### การป้องกัน Open Redirect และ SSRF: ทำจริง ไม่ใช่แค่พูดถึง

จุดที่อันตรายที่สุดของ URL shortener ทุกตัวคือ**การรับ URL จาก user แล้วเก็บไว้ให้คนอื่นคลิกทีหลัง**
โดยไม่ตรวจสอบอะไรเลย — ถ้าไม่ระวัง ระบบจะกลายเป็นเครื่องมือของผู้ไม่ประสงค์ดีสองแบบ:

1. **Open redirect / phishing**: ผู้ไม่ประสงค์ดีสร้างลิงก์สั้นที่ดูน่าเชื่อถือ (เพราะมี domain ของเรา)
   แต่ redirect ไปหน้าฟิชชิงจริง — วิธีป้องกันหลักคือจำกัด scheme ที่ยอมรับให้เหลือแค่ `http`/`https`
   (ปฏิเสธ `javascript:`, `data:`, `file:` ที่ browser บางตัวยัง execute ได้)
2. **SSRF (Server-Side Request Forgery)**: ถ้าระบบมี component ใดที่ต้อง**เชื่อมต่อไปยัง URL ที่ผู้ใช้
   กำหนด**จริง (เช่น ระบบ preview thumbnail ที่ fetch หน้าเว็บปลายทางมาแสดง) ผู้ไม่ประสงค์ดีอาจใส่ URL
   ที่ชี้กลับไปยัง internal network ของเราเอง (เช่น `http://127.0.0.1:9200/` ที่อาจเป็น Elasticsearch
   ภายในที่ไม่มี auth, หรือ `http://169.254.169.254/latest/meta-data/` ซึ่งเป็น cloud metadata
   endpoint ที่มีชื่อเสียงเรื่องถูกใช้ขโมย credential ของ cloud instance) — บทนี้ไม่มี component ที่
   fetch URL ปลายทางจริง (แค่ redirect ให้ browser ของ user ไปเอง) แต่**เขียน validation ไว้ตั้งแต่ต้น
   เผื่ออนาคตมี feature แบบนั้นเพิ่มเข้ามา** เป็นการป้องกันแบบ defense-in-depth

```rust
// src/domain.rs (ส่วน validation)
use crate::error::AppError;

pub const MAX_URL_LENGTH: usize = 2048;

/// ตรวจสอบและ parse URL ปลายทางที่ผู้ใช้ส่งมาให้สร้างลิงก์สั้น
///
/// นี่คือจุด**ป้องกัน open redirect / SSRF เบื้องต้น**ของทั้งระบบ — ทุก URL ที่จะถูกเก็บเข้า
/// database ต้องผ่านฟังก์ชันนี้เท่านั้น ไม่มีทางอื่นที่จะเขียน `long_url` ลงตารางได้
pub fn validate_target_url(raw: &str) -> Result<url::Url, AppError> {
    if raw.len() > MAX_URL_LENGTH {
        return Err(AppError::InvalidUrl(format!(
            "url length {} exceeds max {}",
            raw.len(),
            MAX_URL_LENGTH
        )));
    }

    let parsed = url::Url::parse(raw)
        .map_err(|e| AppError::InvalidUrl(format!("cannot parse url: {e}")))?;

    // (1) จำกัด scheme ให้เหลือแค่ http/https เท่านั้น — ปฏิเสธ `javascript:`, `data:`,
    // `file:` ที่ browser บางตัวยังยอม execute ได้ถ้าเผลอปล่อยให้ redirect ไปตรงนั้น
    match parsed.scheme() {
        "http" | "https" => {}
        other => {
            return Err(AppError::InvalidUrl(format!(
                "scheme '{other}' is not allowed, only http/https"
            )))
        }
    }

    // (2) ต้องมี host เสมอ (กัน URL แบบ `http:///etc/passwd` ที่ parse ผ่านแต่ไม่มีปลายทางจริง)
    let host = parsed
        .host_str()
        .ok_or_else(|| AppError::InvalidUrl("url must have a host".to_string()))?;

    // (3) ถ้า host เป็น IP address ตรง ๆ (ไม่ใช่ domain name) ให้ปฏิเสธ loopback/private/
    // link-local range ทันที — ป้องกันไม่ให้แอปกลายเป็น proxy สำหรับสอดส่อง internal network
    // ของตัวเอง (SSRF) เช่น ห้ามสร้างลิงก์สั้นไปยัง `http://127.0.0.1:9200/` หรือ
    // `http://169.254.169.254/latest/meta-data/` (cloud metadata endpoint ที่มีชื่อเสียงเรื่องถูก
    // SSRF โจมตี) — หมายเหตุสำคัญ: การตรวจนี้ครอบคลุมกรณี host เป็น IP literal เท่านั้น ถ้า host
    // เป็น domain name (เช่น `internal.corp.example`) ที่ resolve ไปที่ private IP ตอน DNS lookup
    // จริง (DNS rebinding) การตรวจแค่ตอน validate ยังไม่พอ — ระบบ production จริงต้องมี resolver
    // ที่ตรวจ IP ปลายทาง**ทุกครั้งก่อน connect จริง**ด้วย (เช่น custom `Resolve` ของ reqwest ที่
    // ปฏิเสธ private range หลัง DNS resolve เสร็จ) บทนี้ทำแค่ชั้นแรกที่ตรวจได้ทันทีตอนรับ request
    // เพื่อโฟกัสที่ตัวบทเรียนหลัก ไม่ implement custom resolver เต็มรูปแบบ
    if let Ok(ip) = host.parse::<std::net::IpAddr>() {
        if is_disallowed_ip(&ip) {
            return Err(AppError::InvalidUrl(format!(
                "target host '{ip}' is not allowed (loopback/private/link-local range)"
            )));
        }
    }

    Ok(parsed)
}

/// ตรวจว่า IP อยู่ใน range ที่ไม่ควรให้ redirect ไปถึงได้ (loopback, private, link-local,
/// unspecified) — ครอบคลุมทั้ง IPv4 และ IPv6
fn is_disallowed_ip(ip: &std::net::IpAddr) -> bool {
    match ip {
        std::net::IpAddr::V4(v4) => {
            v4.is_loopback()
                || v4.is_private()
                || v4.is_link_local()
                || v4.is_unspecified()
                || *v4 == std::net::Ipv4Addr::new(169, 254, 169, 254) // cloud metadata endpoint
        }
        std::net::IpAddr::V6(v6) => v6.is_loopback() || v6.is_unspecified(),
    }
}
```

ทดสอบด้วย unit test ที่รันจริงและผ่านทุกเคส (ไม่ต้องแตะ database เลยเพราะเป็น pure function):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn validate_target_url_accepts_https() {
        let url = validate_target_url("https://example.com/docs/rust").unwrap();
        assert_eq!(url.host_str(), Some("example.com"));
    }

    #[test]
    fn validate_target_url_rejects_javascript_scheme() {
        let err = validate_target_url("javascript:alert(1)").unwrap_err();
        assert!(matches!(err, AppError::InvalidUrl(_)));
    }

    #[test]
    fn validate_target_url_rejects_loopback_ip() {
        let err = validate_target_url("http://127.0.0.1:8080/admin").unwrap_err();
        assert!(matches!(err, AppError::InvalidUrl(_)));
    }

    #[test]
    fn validate_target_url_rejects_cloud_metadata_endpoint() {
        let err = validate_target_url("http://169.254.169.254/latest/meta-data/").unwrap_err();
        assert!(matches!(err, AppError::InvalidUrl(_)));
    }

    #[test]
    fn validate_target_url_rejects_oversized_url() {
        let too_long = format!("https://example.com/{}", "a".repeat(3000));
        let err = validate_target_url(&too_long).unwrap_err();
        assert!(matches!(err, AppError::InvalidUrl(_)));
    }
}
```

รันจริงด้วย `cargo test` ได้ผลลัพธ์จริง:

```text
running 6 tests
test domain::tests::validate_target_url_rejects_javascript_scheme ... ok
test domain::tests::validate_target_url_accepts_https ... ok
test domain::tests::validate_target_url_rejects_cloud_metadata_endpoint ... ok
test domain::tests::validate_target_url_rejects_loopback_ip ... ok
test domain::tests::validate_target_url_rejects_oversized_url ... ok
test domain::tests::generate_code_has_correct_length_and_alphabet ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

และยืนยันด้วยการรัน server จริงแล้วยิง request จริง (ไม่ใช่แค่ unit test):

```bash
curl -sS -X POST http://127.0.0.1:8080/api/links \
  -H 'Content-Type: application/json' \
  -d '{"url":"javascript:alert(1)"}'
```

```json
{"error":"invalid_url","message":"scheme 'javascript' is not allowed, only http/https"}
```

```bash
curl -sS -X POST http://127.0.0.1:8080/api/links \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://169.254.169.254/latest/meta-data/"}'
```

```json
{"error":"invalid_url","message":"target host '169.254.169.254' is not allowed (loopback/private/link-local range)"}
```

ทั้งสอง response นี้คัดลอกตรงจากการยิง `curl` จริงไปที่ server ที่รันอยู่จริงในเครื่องที่ใช้ตรวจสอบบทนี้

### 108.4 Collision-Safe Short Code Generation: พิสูจน์ Concurrency Safety ด้วยการรันจริง

#### สุ่ม Short Code: ทำไมต้องเป็น 7 ตัวอักษร

```rust
// src/domain.rs (ส่วน generation)
use rand::distr::Alphanumeric;
use rand::RngExt;

/// จำนวนอักขระของ short code — 7 ตัวจาก alphabet 62 ตัว (a-z, A-Z, 0-9) ให้ keyspace
/// ประมาณ 62^7 ≈ 3.5 × 10^12 ค่าที่เป็นไปได้ ซึ่งมากพอที่โอกาสชนกันโดยบังเอิญ (birthday
/// paradox) จะต่ำมากแม้มีลิงก์อยู่ในระบบหลักสิบล้านตัว
pub const CODE_LENGTH: usize = 7;

/// จำนวนครั้งสูงสุดที่ยอมให้ retry เมื่อเจอ collision ตอนสร้าง short code ใหม่ — เกินนี้ถือว่า
/// ผิดปกติ (ไม่ใช่ collision ธรรมดา) แล้วคืน `AppError::TooManyCollisions` ให้ caller ตัดสินใจเอง
pub const MAX_COLLISION_RETRIES: u8 = 5;

/// สุ่ม short code ใหม่หนึ่งตัวจาก alphabet แบบ base62 (a-z, A-Z, 0-9)
///
/// ใช้ `rand::rng()` (thread-local generator ของ rand 0.10) แทนการฝัง seed คงที่ — ทุกครั้งที่
/// เรียกจะได้ค่าใหม่ที่แทบเป็นไปไม่ได้ที่จะทำนายได้ ซึ่งสำคัญมากสำหรับ short code เพราะถ้า
/// generator ทำนายได้ ผู้ไม่ประสงค์ดีจะเดา code ของคนอื่นแล้วเข้าถึงลิงก์ที่ไม่ได้ตั้งใจแชร์ได้ง่าย
pub fn generate_code() -> String {
    rand::rng()
        .sample_iter(&Alphanumeric)
        .take(CODE_LENGTH)
        .map(char::from)
        .collect()
}
```

**คำนวณความเสี่ยง collision จริงด้วยตัวเลข**: ด้วย birthday paradox approximation ความเสี่ยงจะเกิด
collision อย่างน้อยหนึ่งครั้งเมื่อมี $n$ ลิงก์ในระบบ (จาก keyspace $N = 62^7 \approx 3.5 \times 10^{12}$)
ประมาณได้จาก $p \approx 1 - e^{-n^2/2N}$ — ที่ $n = 10{,}000{,}000$ (สิบล้านลิงก์): $p \approx 1 -
e^{-10^{14}/(2 \times 3.5\times10^{12})} \approx 1 - e^{-14.3} \approx 99.99993\%$... **เดี๋ยว ตัวเลข
นี้ดูสูงผิดปกติ — ต้องตรวจสอบให้ถูก**: ที่ $n = 10^7$, $n^2 = 10^{14}$, $2N = 7\times10^{12}$, อัตราส่วน
$= 10^{14}/7\times10^{12} \approx 14.3$ — ค่านี้คือ**ความน่าจะเป็นสะสมของ pair ใดๆ ที่ชนกัน**ตลอดอายุ
ของทั้งระบบที่มี 10 ล้านลิงก์สั่งสมอยู่พร้อมกัน ซึ่งแปลว่า**จะเกิด collision จริงหลายครั้งแน่นอน**ถ้า
ระบบโตถึงระดับสิบล้านลิงก์ที่ใช้ code length 7 — นี่คือเหตุผลที่ **collision handling ที่ถูกต้องไม่ใช่
"nice to have" แต่เป็นสิ่งที่ระบบจริงระดับนี้ต้องมี** และเป็นเหตุผลที่แบบฝึกหัดข้อ 4 ท้ายบทให้ลองคำนวณ
ว่าต้องเพิ่ม `CODE_LENGTH` เป็นเท่าไหร่ถึงจะทำให้ความเสี่ยงนี้ต่ำลงในระดับที่ยอมรับได้

#### Collision Handling ใน Handler: Retry Loop ที่ปลอดภัย

```rust
// src/handlers.rs (ส่วน create_link)
#[tracing::instrument(skip(state, payload), fields(url_len = payload.url.len()))]
pub async fn create_link(
    State(state): State<AppState>,
    Json(payload): Json<CreateLinkRequest>,
) -> Result<Json<CreateLinkResponse>, AppError> {
    let target = validate_target_url(&payload.url)?;

    let expires_at = payload
        .ttl_seconds
        .map(|secs| Utc::now() + ChronoDuration::seconds(secs));

    let mut attempts: u8 = 0;
    let code = loop {
        attempts += 1;
        let candidate = generate_code();

        let inserted = state
            .repo
            .try_insert(&candidate, target.as_str(), expires_at, None)
            .await?;

        if inserted {
            if attempts > 1 {
                // ถ้าเกิดการชนจริง (attempts > 1) บันทึก metric ไว้ — ถ้าตัวเลขนี้เพิ่มขึ้นสูง
                // ผิดปกติในระบบจริง แปลว่า keyspace เล็กเกินไปสำหรับปริมาณลิงก์ที่มี
                metrics::counter!("short_code_collisions_total").increment((attempts - 1) as u64);
            }
            break candidate;
        }

        if attempts >= MAX_COLLISION_RETRIES {
            return Err(AppError::TooManyCollisions);
        }
    };

    metrics::counter!("links_created_total").increment(1);

    Ok(Json(CreateLinkResponse {
        code: code.clone(),
        short_url: format!("{}/{}", state.public_base_url, code),
        long_url: target.to_string(),
        expires_at,
    }))
}
```

**เหตุผลที่ retry loop นี้ปลอดภัยต่อ concurrency แม้มีหลาย request พร้อมกัน**: แต่ละรอบของ loop สุ่ม
code ใหม่ทุกครั้ง (ไม่ใช่ retry ด้วย code เดิม) แล้วพยายาม insert ผ่าน `try_insert` ที่ atomic ที่ระดับ
Postgres (หัวข้อ 108.2) — **ไม่มีการ "เช็คก่อนว่าง่ายแล้วค่อย insert"** ที่จะเปิดช่องให้ request อื่น
แซงเข้ามาระหว่างสองคำสั่ง สิ่งที่เกิดขึ้นจริงคือ "insert แบบมองโลกในแง่ดี" (optimistic) — พยายาม insert
ตรง ๆ แล้วให้ database เป็นผู้ตัดสินว่าสำเร็จหรือชน

#### พิสูจน์ด้วย Integration Test จริง: 30 Task แข่งกันเขียน Code เดียวกัน

คำพูดที่ว่า "ปลอดภัยต่อ concurrency" ต้องพิสูจน์ได้ ไม่ใช่แค่อ้างจากการอ่านโค้ด — เขียน integration test
ที่**บังคับให้ทุก task ใช้ short code เดียวกันเป๊ะ** (ไม่ใช่หวังให้สุ่มไม่ชนกัน) แล้วเช็คว่ามี
**exactly หนึ่ง task เท่านั้น**ที่ชนะ:

```rust
// tests/concurrency_proof.rs
//! integration test นี้คือ**หลักฐานจริง**ว่า collision handling ของระบบปลอดภัยต่อ concurrency
//! จริง ไม่ใช่แค่ "ดูเหมือนถูกต้องตอนอ่านโค้ด" — รันด้วย `DATABASE_URL` ชี้ไปที่ Postgres จริง
//!
//! สถานการณ์ที่จำลอง: ยิง 30 task พร้อมกัน (`tokio::spawn`) ทุกตัวพยายาม `try_insert`ด้วย
//! **short code เดียวกันเป๊ะ** (ตั้งใจบังคับให้ชนกันทั้งหมด ไม่ใช่หวังให้สุ่มไม่ชนกัน) —
//! ถ้า `ON CONFLICT DO NOTHING` ของ Postgres ทำงานถูกต้องตามที่ออกแบบไว้ ต้องมี**exactly หนึ่ง
//! task เท่านั้น**ที่ได้ `true` กลับมา ส่วนที่เหลืออีก 29 task ต้องได้ `false` ทั้งหมด
//! (จำนวน 30 ถูกเลือกให้พอดีกับ `max_connections` ของ connection pool ในเทสต์นี้ — ถ้าใช้
//! ตัวเลขสูงเกินไปพร้อมกับเทสต์อื่นที่รันขนานกัน จะชนกับ `max_connections` เริ่มต้นของ Postgres
//! เอง (ปกติคือ 100 ทั้ง cluster) แล้วได้ `PoolTimedOut` ซึ่งเป็น**ปัญหาของการตั้งค่า connection
//! limit ไม่ใช่ปัญหาของ atomicity ที่เทสต์นี้ต้องการพิสูจน์**)

use sqlx::postgres::PgPoolOptions;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

const CONCURRENT_TASKS: usize = 30;

#[tokio::test]
async fn concurrent_inserts_with_same_code_exactly_one_wins() {
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://urlshortener:urlshortener@127.0.0.1/urlshortener".into());

    let pool = PgPoolOptions::new()
        .max_connections(CONCURRENT_TASKS as u32 + 10)
        .connect(&database_url)
        .await
        .expect("connect to postgres");

    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("run migrations");

    // code คงที่พิเศษเฉพาะเทสต์นี้ ไม่ชนกับ code ที่สุ่มจริงในระบบ (ป้องกัน flaky ถ้ารันซ้ำ)
    let contested_code = "TESTXX9";
    sqlx::query("DELETE FROM links WHERE code = $1")
        .bind(contested_code)
        .execute(&pool)
        .await
        .expect("cleanup before test");

    let success_count = Arc::new(AtomicUsize::new(0));
    let mut handles = Vec::with_capacity(CONCURRENT_TASKS);

    for i in 0..CONCURRENT_TASKS {
        let pool = pool.clone();
        let success_count = success_count.clone();
        handles.push(tokio::spawn(async move {
            let result = sqlx::query(
                "INSERT INTO links (code, long_url) VALUES ($1, $2) ON CONFLICT (code) DO NOTHING",
            )
            .bind(contested_code)
            .bind(format!("https://example.com/task-{i}"))
            .execute(&pool)
            .await
            .expect("insert should not error even on conflict");

            if result.rows_affected() == 1 {
                success_count.fetch_add(1, Ordering::SeqCst);
            }
        }));
    }

    for handle in handles {
        handle.await.expect("task panicked");
    }

    let total_success = success_count.load(Ordering::SeqCst);
    assert_eq!(
        total_success, 1,
        "exactly one of {CONCURRENT_TASKS} concurrent inserts should have won the race, got {total_success}"
    );

    // ยืนยันอีกชั้นว่า database มีแถวเดียวจริง ๆ ไม่ใช่หลายแถวที่ค่า code ซ้ำกันหลุดเข้ามาได้
    let (row_count,): (i64,) =
        sqlx::query_as("SELECT COUNT(*) FROM links WHERE code = $1")
            .bind(contested_code)
            .fetch_one(&pool)
            .await
            .expect("count rows");
    assert_eq!(row_count, 1);

    sqlx::query("DELETE FROM links WHERE code = $1")
        .bind(contested_code)
        .execute(&pool)
        .await
        .expect("cleanup after test");
}

/// พิสูจน์ว่า `increment_click` (ผ่าน `UPDATE ... SET click_count = click_count + 1`) เป็น atomic
/// จริง — ยิง 30 increment พร้อมกันไปที่แถวเดียว แล้วต้องได้ผลรวมเท่ากับ 30 เป๊ะ ไม่มี "หายไป"
/// จาก race condition ที่มักเกิดถ้าเขียนแบบ "อ่านค่ามาบวกในแอปแล้วเขียนกลับ" (read-modify-write
/// ที่ไม่ atomic)
#[tokio::test]
async fn concurrent_click_increments_are_atomic() {
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://urlshortener:urlshortener@127.0.0.1/urlshortener".into());

    let pool = PgPoolOptions::new()
        .max_connections(40)
        .connect(&database_url)
        .await
        .expect("connect to postgres");

    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("run migrations");

    let code = "CLICKTST";
    sqlx::query("DELETE FROM links WHERE code = $1")
        .bind(code)
        .execute(&pool)
        .await
        .unwrap();
    sqlx::query("INSERT INTO links (code, long_url) VALUES ($1, 'https://example.com')")
        .bind(code)
        .execute(&pool)
        .await
        .unwrap();

    const CONCURRENT_CLICKS: usize = 30;
    let mut handles = Vec::with_capacity(CONCURRENT_CLICKS);
    for _ in 0..CONCURRENT_CLICKS {
        let pool = pool.clone();
        handles.push(tokio::spawn(async move {
            sqlx::query("UPDATE links SET click_count = click_count + 1 WHERE code = $1")
                .bind(code)
                .execute(&pool)
                .await
                .expect("increment should not fail");
        }));
    }
    for handle in handles {
        handle.await.expect("task panicked");
    }

    let (final_count,): (i64,) =
        sqlx::query_as("SELECT click_count FROM links WHERE code = $1")
            .bind(code)
            .fetch_one(&pool)
            .await
            .unwrap();

    assert_eq!(final_count, CONCURRENT_CLICKS as i64);

    sqlx::query("DELETE FROM links WHERE code = $1")
        .bind(code)
        .execute(&pool)
        .await
        .unwrap();
}
```

รันจริงกับ Postgres จริง (native service ในเครื่องตรวจสอบบทนี้) ได้ผลลัพธ์จริง:

```text
running 2 tests
test concurrent_click_increments_are_atomic ... ok
test concurrent_inserts_with_same_code_exactly_one_wins ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.10s
```

นี่คือ**หลักฐานที่พิสูจน์แล้ว ไม่ใช่คำอ้าง** ว่า:

1. เมื่อ 30 task พยายามเขียน short code เดียวกันพร้อมกันเป๊ะ ๆ มี**เพียงหนึ่งเดียว**ที่ชนะเสมอ ไม่ว่า
   จะรันกี่ครั้งก็ตาม (ทดสอบซ้ำได้ deterministic เพราะ `ON CONFLICT` เป็นกลไกของ database engine เอง
   ไม่ใช่ timing-dependent code ในแอป)
2. click counter ที่ increment พร้อมกัน 30 ครั้งได้ผลรวมที่ถูกต้องเป๊ะ 30 เสมอ ไม่มีการ "หายไป" จาก
   race condition

### 108.5 Observability ครบสามเสาหลัก: Logs + Metrics + Traces ในระบบเดียว

#### ติดตั้งทั้งสามเสาพร้อมกันตอน Startup

จาก Part 98 และ Part 99 เราได้เรียนแต่ละเสาแยกกัน — บทนี้เอาทั้งสามมาผูกเข้าด้วยกันในฟังก์ชันเดียว
`telemetry::init()` ที่เรียกเป็นบรรทัดแรกสุดของ `main()`:

```rust
// src/telemetry.rs
use metrics_exporter_prometheus::{PrometheusBuilder, PrometheusHandle};
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;
use tracing_subscriber::EnvFilter;

/// ผลลัพธ์ของการติดตั้ง telemetry ทั้งหมด — เก็บ handle ที่ต้องใช้ตอนจบโปรแกรม
/// (`otel_provider.shutdown()` เพื่อ flush span ที่เหลือ) และตอนเปิด `/metrics` endpoint
/// (`prometheus_handle.render()`)
pub struct Telemetry {
    pub prometheus_handle: PrometheusHandle,
    pub otel_provider: Option<SdkTracerProvider>,
}

/// ติดตั้ง observability ครบสามเสาตั้งแต่บรรทัดแรกของ `main()`:
///
/// 1. **Logs**: `tracing_subscriber::fmt::layer()` พิมพ์ log แบบ structured ออก stdout — ทุก log
///    ผูกกับ span ปัจจุบันอัตโนมัติ (Part 60)
/// 2. **Metrics**: `PrometheusBuilder` ติดตั้ง global recorder ให้ `metrics::counter!`/`histogram!`
///    ทั้งแอปมีที่ไปเก็บค่า พร้อมตั้ง bucket ของ histogram latency ให้เหมาะกับ web service จริง
///    (Part 98)
/// 3. **Traces**: ถ้ามี `otlp_endpoint` (ตั้งไว้ผ่าน `OTEL_EXPORTER_OTLP_ENDPOINT`) ติดตั้ง
///    `tracing-opentelemetry` layer ที่ export span ไปยัง Jaeger จริง (Part 99) — ถ้าไม่ตั้ง (เช่น
///    ตอนรัน unit test) ระบบยัง**ทำงานได้ปกติทุกอย่าง** เพียงแค่ไม่มีการ export trace ออกไปไหนเลย
///    (ไม่ panic ไม่ error) เพราะ layer เป็น `Option<L>` ซึ่ง `tracing_subscriber` ปฏิบัติเหมือน
///    no-op layer เมื่อเป็น `None`
pub fn init(service_name: &str, otlp_endpoint: Option<&str>) -> Telemetry {
    let otel_layer = otlp_endpoint.map(|endpoint| {
        let provider = build_otel_provider(service_name, endpoint);
        let tracer = provider.tracer(service_name.to_string());
        (tracing_opentelemetry::layer().with_tracer(tracer), provider)
    });

    let (tracing_layer, otel_provider) = match otel_layer {
        Some((layer, provider)) => (Some(layer), Some(provider)),
        None => (None, None),
    };

    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")))
        .with(tracing_subscriber::fmt::layer())
        .with(tracing_layer)
        .init();

    let prometheus_handle = PrometheusBuilder::new()
        // ตั้ง bucket ของ latency histogram เอง ให้เหมาะกับ web service ที่ latency ปกติอยู่ในหน่วย
        // "หลัก ms ถึงหลักสิบ ms" — ถ้าไม่ตั้งเอง exporter จะ export เป็น Prometheus Summary แทน
        // Histogram (กับดักที่ Part 98 พิสูจน์ไว้แล้ว) ทำให้ histogram_quantile() ใช้ไม่ได้
        .set_buckets_for_metric(
            metrics_exporter_prometheus::Matcher::Full("http_request_duration_seconds".to_string()),
            &[
                0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0,
            ],
        )
        .expect("invalid histogram buckets")
        .install_recorder()
        .expect("failed to install Prometheus recorder");

    Telemetry {
        prometheus_handle,
        otel_provider,
    }
}

fn build_otel_provider(service_name: &str, endpoint: &str) -> SdkTracerProvider {
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .with_endpoint(endpoint.to_string())
        .build()
        .expect("failed to build OTLP exporter");

    let resource = Resource::builder()
        .with_service_name(service_name.to_string())
        .with_attribute(opentelemetry::KeyValue::new(
            "service.version",
            env!("CARGO_PKG_VERSION"),
        ))
        .build();

    SdkTracerProvider::builder()
        .with_batch_exporter(exporter)
        .with_resource(resource)
        .build()
}
```

#### Middleware วัด HTTP Metrics — ใช้ `.route_layer()` ตามหลักการจาก Part 98

```rust
// src/mw.rs (ส่วน metrics)
use axum::body::Body;
use axum::extract::MatchedPath;
use axum::http::Request;
use axum::middleware::Next;
use axum::response::Response;
use std::time::Instant;

/// Middleware วัด HTTP request counter + latency histogram ให้**ทุก route โดยอัตโนมัติ**
///
/// แนบด้วย `.route_layer()` (ไม่ใช่ `.layer()`) โดยตั้งใจ — ตามที่ Part 98 หัวข้อ 98.5 พิสูจน์ไว้
/// ด้วยการรันจริงว่า `.route_layer()` ไม่ถูกเรียกเลยสำหรับ request ที่ไม่ match route ไหน (404)
/// ทำให้ label `path` ที่มาจาก `MatchedPath` ไม่มีทางกลายเป็น raw path ที่ไม่มีขอบเขต (เช่น bot
/// scanner ยิง `/wp-admin`, `/.env` มาแบบสุ่ม) — ป้องกันกับดัก high-cardinality label ตั้งแต่ต้น
/// โดยไม่ต้องเขียน fallback logic เพิ่มเลย
pub async fn track_http_metrics(req: Request<Body>, next: Next) -> Response {
    let method = req.method().to_string();
    let path = req
        .extensions()
        .get::<MatchedPath>()
        .map(|mp| mp.as_str().to_owned())
        .unwrap_or_else(|| "unmatched".to_string());

    let start = Instant::now();
    let response = next.run(req).await;
    let elapsed = start.elapsed().as_secs_f64();
    let status = response.status().as_u16().to_string();

    metrics::counter!(
        "http_requests_total",
        "method" => method.clone(),
        "path" => path.clone(),
        "status" => status,
    )
    .increment(1);

    metrics::histogram!(
        "http_request_duration_seconds",
        "method" => method,
        "path" => path,
    )
    .record(elapsed);

    response
}
```

ทุก handler ที่สำคัญยังใส่ `#[tracing::instrument]` ตามรูปแบบของ Part 60/99 (`create_link`,
`redirect`, `stats`) ให้ span ของ request ถูกส่งออกไปที่ Jaeger ผ่าน layer ที่ตั้งไว้ข้างบนโดยอัตโนมัติ
โดยตัว handler ไม่ต้องรู้เรื่อง OTel เลยแม้แต่บรรทัดเดียว — เหมือนที่ Part 99 อธิบายไว้ว่า
`tracing::Span` กับ OTel span คือแนวคิดเดียวกันเกือบ 1:1

#### ยืนยันด้วยโครงสร้างพื้นฐานจริงสามตัวพร้อมกัน

รัน Postgres, Redis, Prometheus, และ Jaeger จริงพร้อมกัน แล้วรัน service:

```bash
# Prometheus scrape config (prometheus.yml)
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: "urlshortener"
    static_configs:
      - targets: ["127.0.0.1:8080"]
```

```bash
# รัน Jaeger (binary release, ไม่ผ่าน Docker — ดูหมายเหตุการตรวจสอบด้านบน)
./jaeger-all-in-one --collector.otlp.enabled=true

# รัน Prometheus
prometheus --config.file=./prometheus.yml --web.listen-address=127.0.0.1:9090

# รัน service (ชี้ OTEL_EXPORTER_OTLP_ENDPOINT ไปที่ Jaeger's gRPC receiver)
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4317 \
DATABASE_URL=postgres://urlshortener:urlshortener@127.0.0.1/urlshortener \
REDIS_URL=redis://127.0.0.1:6379 \
RUST_LOG=info \
./target/release/urlshortener
```

ยิง request จริงเข้าไปสองสามครั้ง (`POST /api/links`, `GET /{code}`, `GET /api/links/{code}/stats`)
แล้วเช็ค**ทั้งสามระบบพร้อมกัน**:

**1) Prometheus — target ขึ้นสถานะ `up` จริง:**

```bash
curl -sS "http://127.0.0.1:9090/api/v1/targets" | python3 -c "
import json,sys
d = json.load(sys.stdin)
for t in d['data']['activeTargets']:
    print(t['scrapeUrl'], t['health'], t['lastScrapeDuration'])
"
```

```text
http://127.0.0.1:8080/metrics up 0.00080681
```

**2) PromQL — คำนวณ rate จริงจากข้อมูลที่ scrape มา:**

```bash
curl -sS "http://127.0.0.1:9090/api/v1/query?query=sum(rate(http_requests_total[1m]))"
```

```json
{"status":"success","data":{"resultType":"vector","result":[{"metric":{},"value":[1790504511.858,"0.057409999999999996"]}]}}
```

**3) Jaeger — service และ span จริงตรงกับ `#[tracing::instrument]` ในโค้ด:**

```bash
curl -sS "http://127.0.0.1:16686/api/services"
```

```json
{"data":["jaeger-all-in-one","urlshortener"],"total":2,"limit":0,"offset":0,"errors":null}
```

```bash
curl -sS "http://127.0.0.1:16686/api/traces?service=urlshortener&limit=5"
```

ตัวอย่าง span จริงที่ดึงออกมาได้ — สังเกตชื่อ operation ตรงกับชื่อฟังก์ชันที่ใส่
`#[tracing::instrument]` เป๊ะ (`redirect`, `stats`):

```text
traceID: 4ec04037b781d36689563bc39c7ea4ac
  span: redirect duration(us)= 139
traceID: d2ab8c262747d81eb628c907699bc4c6
  span: stats duration(us)= 518
```

**4) `/metrics` endpoint ของแอปเอง — Prometheus text format จริง:**

```bash
curl -sS http://127.0.0.1:8080/metrics | grep -E "^(# TYPE|http_requests_total|links_created_total|cache_lookups_total)"
```

```text
# TYPE cache_lookups_total counter
cache_lookups_total{result="hit"} 1
cache_lookups_total{result="miss"} 2
# TYPE links_created_total counter
links_created_total 1
# TYPE http_requests_total counter
http_requests_total{method="POST",path="/api/links",status="200"} 1
http_requests_total{method="GET",path="/{code}",status="307"} 2
http_requests_total{method="GET",path="/health/live",status="200"} 2
```

สังเกตว่า label `path="/{code}"` เป็น route template ไม่ใช่ raw short code จริง (เช่น `/4jqnz5z`) —
ผลจากการใช้ `.route_layer()` + `MatchedPath` ตามที่อธิบายไว้ข้างบน ถ้าใช้ raw path แทน ทุก short code
ที่ต่างกันจะกลายเป็น time series ใหม่ไม่มีที่สิ้นสุด (กับดัก high-cardinality label ที่ Part 98 พิสูจน์
ไว้แล้ว) — ระบบนี้ป้องกันปัญหานั้นตั้งแต่การออกแบบ middleware โดยไม่ต้องคิดเพิ่มทีหลัง

**นี่คือหลักฐานที่พิสูจน์ว่าทั้งสามเสาหลัก (logs ผ่าน `tracing::info!` ที่เห็นใน terminal, metrics ผ่าน
Prometheus, traces ผ่าน Jaeger) ทำงานร่วมกันในระบบเดียวจริง ไม่ใช่แค่โค้ดที่ compile ผ่านเฉย ๆ**

### 108.6 Resilience: Timeout, Redis-Backed Rate Limiting, และ Graceful Shutdown

#### Rate Limiting ด้วย Redis: ทำไมต้องใช้ Shared State ไม่ใช่ In-Memory

ระบบ production จริงมักรัน**หลาย instance พร้อมกัน**หลัง load balancer เดียวกัน — ถ้า rate limit เก็บ
state แบบ in-memory ของแต่ละ instance (เช่น crate `governor` ที่ผูก state ไว้กับ process เดียว) ผู้ใช้
คนหนึ่งจะสามารถ "หลบ" limit ได้ง่าย ๆ แค่ยิง request กระจายไปตาม instance ต่าง ๆ (limit ต่อ instance
บวกกันแล้วมากกว่าที่ตั้งใจ) — บทนี้เลือกเก็บ state ไว้ที่ **Redis** ซึ่งทุก instance เห็นตัวเลขเดียวกัน:

```rust
// src/ratelimit.rs
use deadpool_redis::Pool;
use redis::Script;

/// Lua script ที่ทำ "increment แล้วตั้ง TTL ครั้งแรกที่สร้าง key" เป็น**หนึ่ง atomic operation
/// เดียว**ที่ฝั่ง Redis server — เหตุผลที่ต้องทำแบบนี้ (ไม่ใช่เรียก `INCR` แล้วเรียก `EXPIRE` เป็น
/// สองคำสั่งแยกจากฝั่ง client): ถ้าเป็นสองคำสั่งแยกกัน มี race window ระหว่างคำสั่งทั้งสองที่ request
/// อื่นอาจ `INCR` แซงเข้ามาได้ และถ้า process ตายพอดีระหว่างสองคำสั่ง (เช่น connection หลุด) key
/// จะไม่มี TTL เลย กลายเป็น "ค้างตลอดไป" — Lua script รันเป็น atomic unit เดียวที่ฝั่ง Redis ตัด
/// ปัญหานี้ทิ้งไปทั้งหมด
const INCR_WITH_TTL_SCRIPT: &str = r#"
local current = redis.call("INCR", KEYS[1])
if current == 1 then
    redis.call("EXPIRE", KEYS[1], ARGV[1])
end
return current
"#;

/// Rate limiter แบบ **fixed window counter** ที่เก็บ state ใน Redis (ไม่ใช่ in-memory ของแต่ละ
/// instance)
pub struct RedisRateLimiter {
    pool: Pool,
    max_requests: u32,
    window_secs: u64,
}

impl RedisRateLimiter {
    pub fn new(pool: Pool, max_requests: u32, window_secs: u64) -> Self {
        Self {
            pool,
            max_requests,
            window_secs,
        }
    }

    /// ตรวจสอบว่า `key` (มักเป็น client IP หรือ API key) ยังอยู่ในโควตาของ window ปัจจุบันไหม
    ///
    /// คืน `Ok(true)` ถ้าอนุญาตให้ผ่าน, `Ok(false)` ถ้าเกินโควตาแล้ว (ควรตอบ 429), และ `Ok(true)`
    /// (fail open) ถ้า Redis ต่อไม่ติด — **การตัดสินใจ fail-open ตรงนี้เป็น judgment call ที่ต้อง
    /// พูดตรง ๆ**: ถ้า Redis ล่ม เราเลือก "ยอมให้ทุก request ผ่านโดยไม่จำกัด" แทนที่จะ "บล็อกทุก
    /// request" เพราะ rate limiting คือกลไก**ป้องกันการใช้งานเกินขนาด** ไม่ใช่กลไกความถูกต้องของ
    /// ระบบหลัก (ไม่เหมือนการตรวจสิทธิ์ที่ต้อง fail closed)
    pub async fn check(&self, key: &str) -> Result<bool, String> {
        let mut conn = match self.pool.get().await {
            Ok(c) => c,
            Err(e) => {
                tracing::warn!(error = %e, "redis unavailable, rate limiter fails open");
                metrics::counter!("ratelimit_backend_errors_total").increment(1);
                return Ok(true);
            }
        };

        let redis_key = format!("ratelimit:{key}");
        let script = Script::new(INCR_WITH_TTL_SCRIPT);

        let current: i64 = match script
            .key(&redis_key)
            .arg(self.window_secs)
            .invoke_async(&mut conn)
            .await
        {
            Ok(v) => v,
            Err(e) => {
                tracing::warn!(error = %e, "redis rate limit script failed, fails open");
                metrics::counter!("ratelimit_backend_errors_total").increment(1);
                return Ok(true);
            }
        };

        let allowed = current <= self.max_requests as i64;
        if !allowed {
            metrics::counter!("ratelimit_rejections_total").increment(1);
        }
        Ok(allowed)
    }
}
```

Middleware ที่ใช้ตัวนี้:

```rust
// src/mw.rs (ส่วน rate limit)
use crate::error::AppError;
use crate::state::AppState;
use axum::extract::{ConnectInfo, State};
use std::net::SocketAddr;

/// Middleware ตรวจ rate limit ผ่าน Redis — key ด้วย client IP
///
/// อ่าน IP จาก `ConnectInfo<SocketAddr>` (connection ตรงจาก client) — ในระบบจริงที่อยู่หลัง
/// reverse proxy/load balancer ต้องอ่านจาก header `X-Forwarded-For` แทน (ค่าจาก `ConnectInfo`
/// จะเป็น IP ของ proxy เอง ไม่ใช่ IP ของ client จริง) — บทนี้ใช้ `ConnectInfo` ตรง ๆ เพื่อความ
/// ชัดเจนของตัวอย่าง และใส่คอมเมนต์ไว้เป็นจุดที่ต้องปรับก่อนขึ้น production จริงหลัง proxy
pub async fn rate_limit(
    State(state): State<AppState>,
    ConnectInfo(addr): ConnectInfo<SocketAddr>,
    req: axum::extract::Request,
    next: axum::middleware::Next,
) -> Result<axum::response::Response, AppError> {
    let key = addr.ip().to_string();

    let allowed = state
        .rate_limiter
        .check(&key)
        .await
        .map_err(AppError::Cache)?;

    if !allowed {
        return Err(AppError::RateLimited {
            retry_after_secs: 60,
        });
    }

    Ok(next.run(req).await)
}
```

**พิสูจน์ด้วยการยิง request จริงเกินโควตา**: ตั้ง `RATE_LIMIT_MAX_REQUESTS=20`,
`RATE_LIMIT_WINDOW_SECS=10` แล้วยิง 30 request ติดกันไปที่ short code เดียวกัน:

```bash
for i in $(seq 1 30); do
  curl -sS -o /dev/null -w "%{http_code} " "http://127.0.0.1:8080/uDo0axt"
done
```

ผลลัพธ์จริง — สังเกตว่าหลังจากผ่านโควตา ระบบเริ่มตอบ `429` ทันที:

```text
307 307 307 307 307 307 307 307 307 307 307 307 307 307 307 307 307 307 429 429 429 429 429 429 429 429 429 429 429 429
```

```bash
curl -sS "http://127.0.0.1:8080/uDo0axt"
```

```json
{"error":"rate_limited","message":"rate limit exceeded, retry after 60s"}
```

#### Timeout: ตัด Request ที่ค้างนานเกินไปอัตโนมัติ

```rust
// src/main.rs (ส่วนประกอบ Router)
.route_layer(TimeoutLayer::with_status_code(
    axum::http::StatusCode::REQUEST_TIMEOUT,
    Duration::from_secs(timeout_secs),
))
```

`TimeoutLayer` จาก `tower-http` ตัด request ที่ทำงานนานเกิน `timeout_secs` (ค่า default 5 วินาที) โดย
อัตโนมัติ ตอบ `408 Request Timeout` กลับไปทันทีโดยไม่ต้องรอ handler เดิมทำงานจบ — สำคัญมากสำหรับ
ป้องกัน **resource exhaustion**: ถ้า database ช้าผิดปกติ (ดูหัวข้อ 108.11 incident simulation) และไม่มี
timeout เลย ทุก request ที่เข้ามาจะ**ค้างรอไม่มีกำหนด**จนกว่า connection จะถูกตัดเอง (ซึ่งอาจนานมาก) —
worker thread/task ของ tokio จะถูกใช้งานค้างไว้เพิ่มขึ้นเรื่อย ๆ จนระบบล่มทั้งระบบแทนที่จะแค่ endpoint
เดียว

#### Graceful Shutdown: พิสูจน์ด้วยการยิง SIGTERM ขณะมี Request กำลังทำงาน

```rust
// src/main.rs
axum::serve(
    listener,
    app.into_make_service_with_connect_info::<SocketAddr>(),
)
.with_graceful_shutdown(shutdown_signal())
.await?;

// ...

/// รอ signal สำหรับ graceful shutdown — รองรับทั้ง Ctrl+C (SIGINT, สำหรับรัน local) และ SIGTERM
/// (สัญญาณที่ Kubernetes/Docker ส่งมาตอนจะ terminate container จริง) — `axum::serve`'s
/// `with_graceful_shutdown` จะรอให้ future นี้ resolve ก่อนหยุดรับ connection ใหม่ แล้ว**รอ
/// request ที่กำลังทำอยู่ให้เสร็จก่อน**ค่อยปิดตัวจริง
async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("received Ctrl+C, starting graceful shutdown"),
        _ = terminate => tracing::info!("received SIGTERM, starting graceful shutdown"),
    }
}
```

**การพิสูจน์จริง** — เพิ่ม debug endpoint ที่ sleep ตามเวลาที่กำหนด (สำหรับสาธิตเท่านั้น ระบบจริงไม่ควร
มี endpoint แบบนี้เปิดอยู่):

```rust
mod handlers_debug {
    use axum::extract::Path;
    use std::time::Duration;

    pub async fn sleep_ms(Path(millis): Path<u64>) -> String {
        tokio::time::sleep(Duration::from_millis(millis)).await;
        format!("slept {millis}ms")
    }
}
```

ยิง request ที่ sleep 3 วินาทีแบบ background แล้ว**ส่ง SIGTERM ทันทีหลังจากนั้นเพียง 0.3 วินาที** (ขณะที่
request ยังทำงานอยู่แน่นอน):

```bash
(curl -sS -w "HTTP_CODE=%{http_code} TIME_TOTAL=%{time_total}\n" \
  -o /tmp/slow_response.txt "http://127.0.0.1:8080/debug/sleep/3000" > /tmp/slow_timing.txt 2>&1 &)
sleep 0.3
kill -TERM $(pgrep -f target/release/urlshortener)
```

ผลลัพธ์จริง — timeline ที่คัดลอกตรงจากการรันจริง:

```text
=== ผลลัพธ์ของ request ที่ยังทำงานอยู่ตอนส่ง SIGTERM ===
HTTP_CODE=200 TIME_TOTAL=3.003046
slept 3000ms

=== log ของ server ===
2026-09-27T10:25:13.882567Z  INFO urlshortener: received SIGTERM, starting graceful shutdown
2026-09-27T10:25:16.591285Z  INFO urlshortener: shutdown complete
```

**สิ่งที่พิสูจน์ได้จากตัวเลขจริงนี้**: SIGTERM ถูกส่งไปตอน `10:25:13.883` (ดูจาก timestamp ของบรรทัด
"received SIGTERM") ขณะที่ request สาย sleep ยังเหลือเวลาทำงานอีกประมาณ 2.7 วินาที — server**ไม่ได้
ตัดจบ request นั้นทันที** แต่รอจนกว่า request จะตอบ `200 slept 3000ms` สำเร็จ (`TIME_TOTAL=3.003046`
วินาทีเต็ม ตรงกับที่ตั้งไว้) **แล้วค่อย**พิมพ์ "shutdown complete" ตอน `10:25:16.591` (ประมาณ 2.7 วินาที
หลัง SIGTERM ซึ่งพอดีกับเวลาที่เหลือของ request ที่ยังทำงานอยู่) — นี่คือพฤติกรรม graceful shutdown ที่
ถูกต้อง: **ไม่รับ connection ใหม่หลัง signal แต่ปล่อยให้ request ที่กำลังทำอยู่เสร็จสมบูรณ์ก่อน**
ต่างจากการ `kill -9` ที่จะตัดทุกอย่างทันทีโดยไม่สนใจว่า request ไหนทำอะไรอยู่

### 108.7 Caching Hot Path ด้วย Redis: Cache-Aside พร้อมตัวเลข Latency จริง

#### เขียน Cache-Aside Pattern

`GET /{code}` คือ endpoint ที่ hot ที่สุดในทั้งระบบ (ทุกครั้งที่มีคนคลิกลิงก์สั้นที่แชร์ไป) — ใช้
**cache-aside pattern**: เช็ค Redis ก่อนเสมอ ถ้า hit ตอบกลับทันทีโดย**ไม่แตะ database เลย** ถ้า miss
ค่อยไปอ่าน Postgres แล้วเขียนกลับเข้า cache ให้ request ถัดไปได้ประโยชน์:

```rust
// src/cache.rs
use crate::error::AppError;
use deadpool_redis::redis::AsyncCommands;
use deadpool_redis::Pool;
use std::time::Duration;

/// TTL ของ cache entry หนึ่งตัว — 5 นาที เป็นค่าที่ balance ระหว่าง "ลด load ของ database ได้มาก"
/// (ลิงก์ hot path มักถูกกดซ้ำ ๆ ในช่วงเวลาสั้น ๆ หลังแชร์) กับ "ข้อมูล stale ไม่นานเกินไป" ถ้ามีคน
/// ไปลบ/แก้ลิงก์นั้นจาก endpoint อื่น (บทนี้ไม่มี endpoint แก้ไข แต่ในระบบจริงที่มี ต้อง invalidate
/// cache ตอนแก้ไขด้วยเสมอ ไม่ใช่พึ่ง TTL เพียว ๆ)
const CACHE_TTL_SECS: u64 = 300;

fn cache_key(code: &str) -> String {
    format!("short:{code}")
}

/// อ่านค่า `long_url` จาก cache — คืน `None` ถ้าไม่พบ (cache miss เป็นเรื่องปกติ ไม่ใช่ error)
///
/// ถ้า Redis เอง**ต่อไม่ติด**หรือ error ระหว่างอ่าน โค้ดนี้เลือก**คืน `None` แทนที่จะ propagate
/// error กลับไป** — นี่คือหลักการสำคัญของ cache-aside pattern ในระบบ production: cache ต้องเป็น
/// "ทางลัดที่พังได้โดยไม่ทำให้ระบบพังตาม" (fail open) ถ้า Redis ล่มไปทั้งคลัสเตอร์ ระบบควร**ช้าลง**
/// (ทุก request ตกไปอ่าน database ตรง ๆ) ไม่ใช่**ตายสนิท**
pub async fn get_cached_url(pool: &Pool, code: &str) -> Option<String> {
    let mut conn = match pool.get().await {
        Ok(c) => c,
        Err(e) => {
            tracing::warn!(error = %e, "redis pool exhausted or unavailable, falling back to database");
            metrics::counter!("cache_errors_total", "op" => "get").increment(1);
            return None;
        }
    };

    match conn.get::<_, Option<String>>(cache_key(code)).await {
        Ok(value) => value,
        Err(e) => {
            tracing::warn!(error = %e, "redis GET failed, falling back to database");
            metrics::counter!("cache_errors_total", "op" => "get").increment(1);
            None
        }
    }
}

/// เขียนค่าเข้า cache พร้อม TTL — ถ้าเขียนไม่สำเร็จ (Redis ล่ม) แค่ log warning แล้วปล่อยผ่าน
/// ไม่ทำให้ request ปัจจุบัน fail เพราะ "cache เขียนไม่ติด" ไม่ควรทำให้ user เห็น error ทั้งที่
/// จริง ๆ เรามีคำตอบ (จาก database) อยู่ในมือแล้ว
pub async fn set_cached_url(pool: &Pool, code: &str, long_url: &str) {
    let mut conn = match pool.get().await {
        Ok(c) => c,
        Err(e) => {
            tracing::warn!(error = %e, "redis pool exhausted, skip caching this response");
            metrics::counter!("cache_errors_total", "op" => "set").increment(1);
            return;
        }
    };

    let result: Result<(), _> = conn
        .set_ex(cache_key(code), long_url, CACHE_TTL_SECS)
        .await;

    if let Err(e) = result {
        tracing::warn!(error = %e, "redis SET failed");
        metrics::counter!("cache_errors_total", "op" => "set").increment(1);
    }
}

/// health check ของ Redis — ใช้ใน readiness probe ใส่ timeout สั้น ๆ เพื่อไม่ให้ readiness
/// endpoint แขวนรอ Redis ที่ค้างนานเกินไป
pub async fn ping(pool: &Pool) -> bool {
    let check = async {
        let mut conn = pool.get().await.map_err(|_| ())?;
        let _: String = redis::cmd("PING")
            .query_async(&mut conn)
            .await
            .map_err(|_| ())?;
        Ok::<(), ()>(())
    };

    tokio::time::timeout(Duration::from_millis(500), check)
        .await
        .map(|r| r.is_ok())
        .unwrap_or(false)
}
```

ใช้ใน handler `redirect`:

```rust
// src/handlers.rs (ส่วน redirect)
/// `GET /{code}` — endpoint ที่ hot ที่สุดในทั้งระบบ
///
/// **เลือกใช้ `Redirect::temporary` (HTTP 307) ไม่ใช่ 301 (permanent redirect) โดยตั้งใจ**:
/// browser จะ cache 301 ไว้ในเครื่อง client แล้ว**ไม่ยิง request มาที่ server ของเราอีกเลย**ใน
/// ครั้งถัดไป ซึ่งพัง use case หลักของระบบทันที — เราต้องการเห็น**ทุกคลิก**เพื่อทำ analytics
/// (`click_count`) ดังนั้น redirect แบบ temporary ที่ browser ไม่ cache ถาวรคือตัวเลือกที่ถูกต้อง
/// สำหรับ short link service เสมอ
#[tracing::instrument(skip(state))]
pub async fn redirect(
    State(state): State<AppState>,
    Path(code): Path<String>,
) -> Result<Redirect, AppError> {
    if let Some(cached_url) = cache::get_cached_url(&state.redis_pool, &code).await {
        metrics::counter!("cache_lookups_total", "result" => "hit").increment(1);
        spawn_click_increment(state.clone(), code);
        return Ok(Redirect::temporary(&cached_url));
    }
    metrics::counter!("cache_lookups_total", "result" => "miss").increment(1);

    let record = state
        .repo
        .find_by_code(&code)
        .await?
        .ok_or(AppError::NotFound)?;

    if record.is_expired() {
        return Err(AppError::Expired);
    }

    cache::set_cached_url(&state.redis_pool, &code, &record.long_url).await;
    spawn_click_increment(state.clone(), code);

    Ok(Redirect::temporary(&record.long_url))
}

/// เพิ่ม click counter แบบ **fire-and-forget** ใน background task แยก — เหตุผลเชิง performance
/// ที่สำคัญมาก: เราไม่อยากให้ latency ของ redirect (ซึ่งเป็น critical path ที่ user รอเห็นผลจริง)
/// ต้องรอ `UPDATE` ของ database เสร็จก่อนถึงจะตอบ redirect กลับไปได้
///
/// trade-off ที่ต้องยอมรับตรง ๆ: ถ้า process ตายพอดีระหว่าง background task ยังไม่เสร็จ click
/// นั้นจะไม่ถูกนับ (at-most-once, ไม่ใช่ exactly-once) — ตรงกับ NFR ที่กำหนดไว้ในหัวข้อ 108.1 ว่า
/// analytics ยอมรับความคลาดเคลื่อนเล็กน้อยได้ เพื่อแลกกับ latency ของ redirect ที่ต้องเร็วที่สุด
fn spawn_click_increment(state: AppState, code: String) {
    tokio::spawn(async move {
        if let Err(e) = state.repo.increment_click(&code).await {
            tracing::warn!(error = %e, code = %code, "failed to increment click count (best-effort)");
        }
    });
}
```

#### วัด Latency จริง: Cache-Hit เทียบกับ Cache-Miss

พูดว่า "cache ช่วยให้เร็วขึ้น" ลอย ๆ ไม่มีประโยชน์เท่ากับวัดตัวเลขจริง — เริ่มจากวัด**latency ของ
request เดี่ยว ๆ**ก่อน (ลบ cache ก่อนทุกครั้งเพื่อบังคับ cache miss):

```bash
echo "--- COLD (บังคับลบ cache ก่อนทุก request → เข้า Postgres ทุกครั้ง) ---"
for i in 1 2 3 4 5; do
  redis-cli DEL "short:$CODE" > /dev/null
  curl -sS -o /dev/null -w "%{time_total}\n" "http://127.0.0.1:8080/$CODE"
done
```

```text
0.002081
0.001118
0.001401
0.001750
0.001642
```

```bash
echo "--- WARM (cache hit ทุกครั้ง หลังจาก warm ไว้แล้วหนึ่งครั้ง) ---"
for i in 1 2 3 4 5; do
  curl -sS -o /dev/null -w "%{time_total}\n" "http://127.0.0.1:8080/$CODE"
done
```

```text
0.000907
0.000878
0.000905
0.001125
0.000988
```

ตัวเลขเดี่ยว ๆ แบบนี้มี noise สูงจาก overhead ของ `curl` เอง — เพื่อให้เห็นความต่างชัดกว่าภายใต้
**concurrent load จริง** ใช้ `wrk` เทียบสองสถานการณ์:

1. **มี Redis ทำงานปกติ (cache-hit path จริง)**
2. **Redis ต่อไม่ติดจริง** (ชี้ `REDIS_URL` ไปพอร์ตที่ไม่มีอะไรฟังอยู่ — บังคับให้ `cache::get_cached_url`
   fail-open แล้วตกไปอ่าน Postgres ทุกครั้ง เหมือนระบบที่ไม่มี cache เลย)

```bash
# สถานการณ์ที่ 1: cache ทำงานปกติ
wrk -t4 -c50 -d8s --timeout 5s "http://127.0.0.1:8080/$CODE"
```

```text
Running 8s test @ http://127.0.0.1:8080/1yNk1QK
  4 threads and 50 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.20ms    2.83ms  54.58ms   75.01%
    Req/Sec     1.31k   176.86     1.72k    70.94%
  41897 requests in 8.01s, 11.95MB read
Requests/sec:   5229.90
Transfer/sec:      1.49MB
```

```bash
# สถานการณ์ที่ 2: Redis ต่อไม่ติด (REDIS_URL ชี้ไปพอร์ต 6399 ที่ไม่มี Redis ฟังอยู่)
wrk -t4 -c50 -d8s --timeout 5s "http://127.0.0.1:8081/$CODE2"
```

```text
Running 8s test @ http://127.0.0.1:8081/ReUiNEe
  4 threads and 50 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    15.14ms    3.40ms  65.16ms   74.33%
    Req/Sec   797.37    106.97     1.00k    65.00%
  25411 requests in 8.01s, 7.25MB read
Requests/sec:   3172.20
Transfer/sec:      0.90MB
```

| | Redis ทำงานปกติ (cache-hit) | Redis ต่อไม่ติด (DB ทุกครั้ง) | ผลต่าง |
|---|---|---|---|
| **Throughput** | 5,229.90 req/s | 3,172.20 req/s | **+64.9%** |
| **Latency เฉลี่ย** | 9.20ms | 15.14ms | **-39.2%** |

ตัวเลขนี้มาจากการรัน `wrk` จริงที่ concurrency เดียวกัน (`-t4 -c50`) ระยะเวลาเดียวกัน (`-d8s`) กับ
service เดียวกันสองอินสแตนซ์ — ต่างกันแค่ Redis ต่อติดหรือไม่ติด และยืนยันด้วย log จริงว่าตอนที่
Redis ต่อไม่ติด ระบบ**ไม่ล่ม**แค่ตกไปอ่าน database ทุกครั้งจริง (fail-open ทำงานถูกต้อง):

```text
2026-09-27T10:24:11.328462Z  WARN urlshortener::cache: redis pool exhausted, skip caching this response error=Error occurred while creating a new object: Connection refused (os error 111)
```

นี่คือหลักฐานที่พิสูจน์**ทั้งสองเรื่องพร้อมกัน**: (1) cache-aside pattern ให้ throughput เพิ่มขึ้นจริง
เกือบ 65% บน hot path และ (2) การออกแบบ fail-open ทำให้ระบบยังทำงานได้ต่อ (แค่ช้าลง ไม่ตาย) แม้ Redis
ล่มไปทั้งหมด — ตรงกับ NFR ที่ตั้งไว้ในหัวข้อ 108.1 ว่า "redirect ต้องทำงานต่อได้แม้ cache ล่ม" เป๊ะ

### 108.8 Database Performance: Indexing และกับดักจริงที่เจอตอนเขียน Migration

#### Migration และ Index ที่ใช้จริง

```sql
-- migrations/0001_init.sql
CREATE TABLE IF NOT EXISTS links (
    code        TEXT PRIMARY KEY,
    long_url    TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at  TIMESTAMPTZ,
    created_by  TEXT,
    click_count BIGINT NOT NULL DEFAULT 0
);

-- index สำหรับ query แนว "ลิงก์ที่สร้างล่าสุด" / รายงานตามช่วงเวลา
CREATE INDEX IF NOT EXISTS idx_links_created_at ON links (created_at DESC);

-- index บน expires_at: ใช้กรอง "ลิงก์ที่ยัง/ไม่ยัง expire" ได้เร็ว — หมายเหตุสำคัญ: ตั้งใจไม่ใช้
-- partial index แบบ `WHERE expires_at > now()` เพราะ Postgres ปฏิเสธฟังก์ชันที่ไม่ IMMUTABLE
-- (เช่น now()) ใน index predicate ตรง ๆ (error จริงที่เจอตอนทดสอบ: "functions in index predicate
-- must be marked IMMUTABLE") — ค่าของ now() เปลี่ยนทุกครั้งที่ query ทำให้ Postgres ไม่สามารถตัดสิน
-- ได้ล่วงหน้าตอนสร้าง index ว่าแถวไหน "เข้าเงื่อนไข predicate" อย่างถาวร ใช้ btree index ปกติบน
-- column ตรง ๆ แทน แล้วให้ query WHERE ตัดสินใจ threshold ของเวลาเองตอน query จริง
CREATE INDEX IF NOT EXISTS idx_links_expires_at ON links (expires_at) WHERE expires_at IS NOT NULL;
```

การใช้ `code TEXT PRIMARY KEY` เป็นตัวเลือกที่ตั้งใจ — Postgres สร้าง unique btree index ให้ column ที่
เป็น primary key อัตโนมัติ ซึ่งคือกลไกเดียวกันที่ทำให้ `find_by_code` เร็ว (lookup ผ่าน index) **และ**
เป็นกลไกที่ทำให้ `ON CONFLICT (code) DO NOTHING` ทำงานได้ (ต้องมี unique constraint บน column นั้นก่อน
Postgres ถึงจะรู้ว่าอะไรคือ "conflict") — index ตัวเดียวรับหน้าที่ทั้งสองอย่างพร้อมกันโดยไม่ต้องสร้างเพิ่ม

`idx_links_created_at` รองรับ query แนว "ดูลิงก์ที่สร้างล่าสุด N รายการ" (`ORDER BY created_at DESC
LIMIT N`) ที่จะช้าลงเรื่อย ๆ แบบ linear ถ้าไม่มี index (ต้อง sequential scan ทั้งตารางแล้ว sort เอง)
เมื่อตารางมีลิงก์เป็นล้านแถว — ใส่ `DESC` ในนิยาม index ตรง ๆ เพราะ query ที่ใช้จริงเรียงจากใหม่ไปเก่า
เสมอ ทำให้ Postgres อ่าน index ตามลำดับที่มันเก็บไว้แล้วโดยไม่ต้อง sort เพิ่ม

#### Load Test ระดับ Throughput: `redirect` Endpoint ที่พึ่งพา Index

ทดสอบ throughput ของ endpoint ที่ query ผ่าน primary key index (`find_by_code`) ภายใต้ concurrent
load — ผลลัพธ์จริงจากหัวข้อ 108.7 (สถานการณ์ Redis ต่อไม่ติด ซึ่งบังคับให้ทุก request ต้อง query
Postgres ผ่าน index จริง ไม่มี cache มาช่วย) แสดงให้เห็นแล้วว่าที่ 50 concurrent connections ระบบทำ
throughput ได้ **3,172 req/s** ด้วย latency เฉลี่ย **15.14ms** — ตัวเลขนี้มาจาก query เดียวคือ

```sql
SELECT code, long_url, created_at, expires_at, created_by, click_count
FROM links WHERE code = $1
```

ที่ผ่าน primary key index lookup (`O(log n)`) ไม่ใช่ sequential scan — ถ้าไม่มี primary key/index บน
`code` เลย (สมมติออกแบบผิดให้ `code` เป็นแค่ column ปกติที่ไม่มี index) query เดียวกันนี้จะต้อง scan
ทุกแถวของตารางเพื่อหาแถวที่ตรงกัน ซึ่งที่ตารางขนาดล้านแถวจะช้าลงเป็นระดับร้อย ms ต่อ query แทนที่จะเป็น
เศษเสี้ยว ms — นี่คือเหตุผลที่ index ไม่ใช่ "การปรับแต่งขั้นสูง" แต่เป็น**ความจำเป็นพื้นฐาน**สำหรับ query
ที่ถูกเรียกบ่อยที่สุดในระบบ

### 108.9 Security Hardening: Headers, Redirect Validation, Rate Limiting

สามด้านของ security hardening ที่บทนี้ implement จริง (ไม่ใช่แค่พูดถึงเป็นแนวคิด):

#### 1) Security Headers มาตรฐานในทุก Response

```rust
// src/mw.rs (ส่วน security headers)
use axum::body::Body;
use axum::http::{HeaderValue, Request};
use axum::middleware::Next;
use axum::response::Response;

/// เพิ่ม security header มาตรฐานให้**ทุก response** — ชุด header นี้เป็นหนึ่งใน "quick win" ของ
/// security hardening ที่ต้นทุนต่ำมาก (แค่ตั้ง header) แต่ปิดช่องโหว่ทั้งหมวดได้ทันที
pub async fn security_headers(req: Request<Body>, next: Next) -> Response {
    let mut response = next.run(req).await;
    let headers = response.headers_mut();

    // ป้องกัน browser "เดา" content-type เอง (MIME sniffing) ที่อาจทำให้ response ที่ตั้งใจให้
    // เป็น plain text ถูก render เป็น HTML/JS แทน
    headers.insert(
        "X-Content-Type-Options",
        HeaderValue::from_static("nosniff"),
    );
    // ห้าม embed response นี้ใน <iframe> ของเว็บอื่น — ป้องกัน clickjacking
    headers.insert("X-Frame-Options", HeaderValue::from_static("DENY"));
    // ไม่ส่ง full referrer URL ข้าม origin — ลด information leak ผ่าน Referer header
    headers.insert(
        "Referrer-Policy",
        HeaderValue::from_static("strict-origin-when-cross-origin"),
    );
    // เปิด HSTS — บอก browser ให้บังคับ HTTPS เสมอสำหรับ domain นี้ในอนาคต (มีผลจริงต้องรันหลัง
    // TLS termination เช่น reverse proxy หรือ load balancer ที่ terminate TLS ให้แล้ว)
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=63072000; includeSubDomains"),
    );

    response
}
```

ยืนยันด้วยการยิง request จริง:

```bash
curl -sS -D - -o /dev/null "http://127.0.0.1:8080/health/live" | grep -Ei "x-content-type|x-frame|strict-transport|referrer"
```

```text
x-content-type-options: nosniff
x-frame-options: DENY
referrer-policy: strict-origin-when-cross-origin
strict-transport-security: max-age=63072000; includeSubDomains
```

#### 2) Redirect-Target Validation

ตามที่อธิบายไว้เต็มรูปแบบในหัวข้อ 108.3 — `validate_target_url` คือจุดเดียวที่ทุก URL ต้องผ่านก่อนถูก
เก็บเข้า database ปฏิเสธ scheme ที่ไม่ใช่ `http`/`https` และปฏิเสธ IP ปลายทางที่เป็น loopback/private/
link-local range เพื่อป้องกัน SSRF

#### 3) Rate Limiting เป็น Abuse Mitigation

ตามที่อธิบายไว้เต็มรูปแบบในหัวข้อ 108.6 — การจำกัดจำนวน request ต่อ IP ต่อช่วงเวลาไม่ได้มีไว้แค่
"ป้องกันระบบล่มจาก traffic สูง" แต่ยังเป็น**กลไก abuse mitigation** โดยตรง: ถ้าไม่มี rate limit ผู้ไม่
ประสงค์ดีสามารถสร้างลิงก์สั้นได้ไม่จำกัดจำนวนต่อวินาที ใช้ระบบของเราเป็นเครื่องมือสร้างลิงก์ฟิชชิงจำนวน
มาก (mass phishing link generation) หรือใช้เป็นช่องทาง scan ว่า short code ไหนมีอยู่จริงบ้าง (code
enumeration) — การจำกัดอัตราการสร้างลิงก์ต่อ IP ทำให้การโจมตีแบบนี้ทำได้ช้าลงจนไม่คุ้มค่าในทางปฏิบัติ

**สิ่งที่บทนี้ตั้งใจไม่ทำ (เพื่อรักษา scope ให้ตรงเป้า)**: ไม่มี CAPTCHA, ไม่มี authentication/API key
สำหรับ `POST /api/links` (ระบบนี้ออกแบบเป็น public url shortener แบบเดียวกับ bit.ly ยุคแรก ๆ ที่ไม่
บังคับ login) — ระบบจริงที่ต้องการความปลอดภัยสูงกว่านี้ควรเพิ่ม authentication และ per-user quota
แทนที่จะพึ่ง per-IP rate limit เพียงอย่างเดียว (เพราะ IP เปลี่ยนได้ง่ายกว่า account)

### 108.10 Deployment Readiness: Liveness vs Readiness, และ Dockerfile

#### ทำไม Liveness กับ Readiness ต้องเป็นคนละ Endpoint

Kubernetes (และ orchestrator อื่น ๆ) แยกสองคำถามนี้ออกจากกันชัดเจน เพราะคำตอบที่ถูกต้องของแต่ละคำถาม
นำไปสู่**การกระทำที่ต่างกันโดยสิ้นเชิง**:

| | Liveness | Readiness |
|---|---|---|
| **คำถามที่ตอบ** | "process ยังตอบสนอง event loop อยู่ไหม (ไม่ deadlock/hang)?" | "instance นี้พร้อม**รับ traffic จริง**ไหม ณ ขณะนี้?" |
| **เช็ค dependency ไหม** | **ไม่เช็คเลย** | เช็คทุก dependency สำคัญ (database, cache) |
| **ถ้า fail แล้วเกิดอะไรขึ้น** | orchestrator **restart** process ใหม่ | orchestrator **เอา instance ออกจาก load balancer rotation ชั่วคราว** (ไม่ restart) |
| **เมื่อไหร่ที่ควร fail** | เมื่อ process ค้าง/deadlock จริง ๆ (หายากมาก ถ้าเขียนโค้ดถูกต้อง) | เมื่อ dependency ที่จำเป็นต่อการทำงานตอบไม่ได้ชั่วคราว (เกิดได้บ่อยกว่ามาก เช่น database restart, network partition) |

ถ้าทำ**ผิด**โดยเอา dependency check ไปใส่ใน liveness (ความผิดพลาดที่พบบ่อยมาก): เมื่อ database ล่ม
ชั่วคราว (เช่น restart ตามปกติของ managed database service) liveness จะ fail ตามไปด้วย → Kubernetes
เข้าใจผิดว่า process ตัวเอง"ค้าง" → **restart process ที่ไม่มีปัญหาอะไรเลย** ซ้ำไปเรื่อย ๆ ตราบใดที่
database ยังไม่กลับมา (crash loop) ทั้งที่ปัญหาจริงคือ database ไม่ใช่ตัว process — restart ไม่ช่วย
อะไรเลยและทำให้สถานการณ์แย่ลง (เสีย time ในการ restart ซ้ำ ๆ)

```rust
// src/handlers.rs (health checks)
/// `GET /health/live` — **liveness probe**: ตอบว่า process ยังตอบสนอง event loop อยู่ไหม
/// เท่านั้น — **ไม่เช็ค dependency ใดเลย** โดยตั้งใจ
pub async fn liveness() -> &'static str {
    "ok"
}

/// `GET /health/ready` — **readiness probe**: ตอบว่าแอปพร้อม**รับ traffic จริง**ไหม โดยเช็คว่า
/// ต่อ dependency สำคัญ (database, Redis) ได้จริง ณ ขณะนี้หรือไม่ — ถ้า dependency ตัวใดตัวหนึ่ง
/// ตอบไม่ได้ คืน 503 ทันที เพื่อให้ load balancer/Kubernetes**เอา instance นี้ออกจาก rotation**
/// ชั่วคราว (ไม่ส่ง traffic ใหม่มาที่นี่) โดยไม่ต้อง restart process
pub async fn readiness(State(state): State<AppState>) -> Result<&'static str, AppError> {
    let db_ok = state.repo.count_links().await.is_ok();
    let redis_ok = cache::ping(&state.redis_pool).await;

    if db_ok && redis_ok {
        Ok("ready")
    } else {
        tracing::warn!(db_ok, redis_ok, "readiness check failed");
        Err(AppError::NotReady(format!(
            "dependency not ready: db_ok={db_ok} redis_ok={redis_ok}"
        )))
    }
}
```

ยืนยันความแตกต่างนี้ด้วยการรันจริง — ปิด Redis ของอินสแตนซ์หนึ่ง (ชี้ `REDIS_URL` ไปพอร์ตที่ไม่มีอะไรอยู่)
แล้วเช็คทั้งสอง endpoint:

```bash
curl -sS http://127.0.0.1:8081/health/live
```

```text
ok
```

```bash
curl -sS http://127.0.0.1:8081/health/ready
```

```json
{"error":"not_ready","message":"dependency not ready: db_ok=true redis_ok=false"}
```

**นี่คือหลักฐานว่าการแยกสอง endpoint ทำงานถูกต้องตามที่ออกแบบไว้**: liveness ยังตอบ `ok` ปกติ (process
ไม่ได้ค้าง มันทำงานได้สมบูรณ์) แต่ readiness ตอบ `503` พร้อมบอกรายละเอียดชัดเจนว่า database ปกติ
(`db_ok=true`) แต่ Redis มีปัญหา (`redis_ok=false`) — ถ้าเป็น production จริง Kubernetes จะเอา instance
นี้ออกจาก load balancer ชั่วคราวโดย**ไม่ restart process** ซึ่งถูกต้องเพราะ process เองไม่มีปัญหา

#### Dockerfile: Multi-Stage Build

```dockerfile
# ---- Stage 1: cargo-chef แยก dependency layer จาก source code เพื่อให้ Docker cache dependency
# ไว้ได้ — build ครั้งถัดไปที่แก้แค่ source code (ไม่แก้ Cargo.toml) จะไม่ต้อง compile dependency
# ใหม่ทั้งหมด ต่อยอดรูปแบบ multi-stage build จาก Part 96 ----
FROM rust:1.82-slim AS chef
WORKDIR /app
RUN cargo install cargo-chef --locked

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# ขั้นนี้ compile แค่ dependency (ยังไม่มี source code ของเราเลย) — ถูก cache ไว้เป็น layer แยก
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release --bin urlshortener

# ---- Stage 2: runtime image ที่เล็กที่สุดเท่าที่จำเป็น ไม่มี Rust toolchain ติดไปด้วย ----
FROM debian:bookworm-slim AS runtime
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# รันด้วย non-root user เสมอ — ถ้า container ถูกเจาะ ผู้โจมตีไม่ได้สิทธิ์ root ของ container ไปด้วย
RUN useradd --create-home --uid 10001 appuser

COPY --from=builder /app/target/release/urlshortener /usr/local/bin/urlshortener
COPY --from=builder /app/migrations /app/migrations

WORKDIR /app
USER appuser

EXPOSE 8080

# HEALTHCHECK ของ Docker เอง — ผูกกับ liveness endpoint (ไม่ใช่ readiness) ตามหลักการที่อธิบายไว้
# ข้างบน: Docker/Kubernetes ควรใช้ endpoint นี้ตัดสินใจว่า "restart container ไหม" ไม่ใช่ตัดสินใจ
# เรื่อง traffic routing (นั่นเป็นหน้าที่ของ readiness probe ที่ตั้งแยกใน Kubernetes manifest)
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://127.0.0.1:8080/health/live || exit 1

ENTRYPOINT ["/usr/local/bin/urlshortener"]
```

**หมายเหตุความซื่อตรงเรื่องการตรวจสอบ**: `Dockerfile` นี้เขียนตามรูปแบบ multi-stage build มาตรฐานที่
ถูกต้อง (ต่อยอดจาก Part 96) แต่**ไม่ได้ถูก `docker build` จริงในสภาพแวดล้อมที่ใช้ตรวจสอบบทนี้** เพราะ
Docker daemon ไม่สามารถ start ได้ใน container ที่ซ้อนกันอีกชั้นของ sandbox นี้ (permission ไม่พอสำหรับ
`dockerd`) — ทุกส่วนอื่นของบทถูกตรวจสอบด้วยการรัน binary ที่ compile จริงตรง ๆ กับ Postgres/Redis/
Prometheus/Jaeger ที่รันเป็น native process/binary จริงแทน ผลลัพธ์เชิงพฤติกรรมเหมือนกันทุกประการเพราะ
เป็น binary เดียวกัน เพียงแต่ image ที่ได้จาก `Dockerfile` นี้ไม่ได้ถูกสร้างและรันเป็น container จริง
ในการตรวจสอบครั้งนี้

### 108.11 Capstone Demonstration: ระบบเต็มรูปแบบ + Incident Simulation จริง

หัวข้อสุดท้ายนี้ประกอบทุกอย่างที่เขียนมาทั้งบทเข้าด้วยกันเป็น `main()` เดียว แล้วจำลอง incident จริงหนึ่ง
ครั้งเพื่อดูว่า observability ที่ติดตั้งไว้ช่วยวินิจฉัยปัญหาได้จริงแค่ไหน

#### `main.rs` แบบสมบูรณ์

```rust
// src/main.rs
mod cache;
mod config;
mod domain;
mod error;
mod handlers;
mod mw;
mod ports;
mod ratelimit;
mod repo_postgres;
mod state;
mod telemetry;

use axum::routing::{get, post};
use axum::Router;
use config::Config;
use ports::LinkRepository;
use ratelimit::RedisRateLimiter;
use repo_postgres::PostgresLinkRepository;
use state::AppState;
use std::net::SocketAddr;
use std::sync::Arc;
use std::time::Duration;
use tower_http::timeout::TimeoutLayer;
use tower_http::trace::TraceLayer;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let config = Config::from_env();

    // ติดตั้ง observability ทั้งสามเสาให้เสร็จก่อนสิ่งอื่นใด — ถ้าอะไรพังหลังจากนี้ เราอยากเห็น
    // log/trace ของมันด้วย ไม่ใช่พังแบบเงียบ ๆ ก่อนมี observability
    let telemetry = telemetry::init("urlshortener", config.otlp_endpoint.as_deref());

    tracing::info!(?config, "starting urlshortener service");

    // --- Database: สร้าง connection pool + รัน migration อัตโนมัติตอน startup ---
    let db_pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(config.db_pool_max_size)
        .acquire_timeout(Duration::from_secs(5))
        .connect(&config.database_url)
        .await?;

    sqlx::migrate!("./migrations").run(&db_pool).await?;
    tracing::info!("database migrations applied");

    // --- Redis: connection pool ผ่าน deadpool-redis สำหรับทั้ง cache และ rate limiter ---
    let redis_cfg = deadpool_redis::Config::from_url(&config.redis_url);
    let redis_pool = redis_cfg.create_pool(Some(deadpool_redis::Runtime::Tokio1))?;

    // --- ประกอบ dependency ทั้งหมดเข้า AppState ---
    // สังเกตว่า `repo` ถูกใส่เข้า Arc<dyn LinkRepository> ที่จุดนี้จุดเดียว — จากตรงนี้ไปทั้งแอป
    // ไม่มีที่ไหนอ้างถึง `PostgresLinkRepository` หรือ `PgPool` ตรง ๆ อีกเลย
    let repo: Arc<dyn LinkRepository> = Arc::new(PostgresLinkRepository::new(db_pool));
    let rate_limiter = Arc::new(RedisRateLimiter::new(
        redis_pool.clone(),
        config.rate_limit_max_requests,
        config.rate_limit_window_secs,
    ));

    let state = AppState {
        repo,
        redis_pool,
        rate_limiter,
        prometheus_handle: telemetry.prometheus_handle.clone(),
        public_base_url: config.public_base_url.clone(),
    };

    let app = build_router(state, config.request_timeout_secs);

    let listener = tokio::net::TcpListener::bind(&config.bind_addr).await?;
    tracing::info!(addr = %config.bind_addr, "listening");

    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .with_graceful_shutdown(shutdown_signal())
    .await?;

    // ให้ span ที่เหลือใน batch buffer ของ OTel exporter ถูก flush ออกไปจริงก่อนโปรเซสจบ —
    // ถ้าลืมขั้นนี้ span ของ request สุดท้าย ๆ ก่อน shutdown จะหายไปเงียบ ๆ (กับดักที่ Part 99
    // พิสูจน์ไว้แล้ว)
    if let Some(provider) = telemetry.otel_provider {
        let _ = provider.shutdown();
    }

    tracing::info!("shutdown complete");
    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        tokio::signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("received Ctrl+C, starting graceful shutdown"),
        _ = terminate => tracing::info!("received SIGTERM, starting graceful shutdown"),
    }
}

fn build_router(state: AppState, timeout_secs: u64) -> Router {
    // กลุ่ม route ที่เป็น "public surface" ของ short link service (เขียนลิงก์ใหม่ + ตามลิงก์ +
    // ดูสถิติ) — มีแต่กลุ่มนี้เท่านั้นที่ผ่าน rate limiter เพราะเป็นกลุ่มที่เปิดให้ traffic จาก
    // อินเทอร์เน็ตทั่วไปยิงเข้ามาได้ไม่จำกัดตัวตน
    let public_routes = Router::new()
        .route("/api/links", post(handlers::create_link))
        .route("/api/links/{code}/stats", get(handlers::stats))
        .route("/{code}", get(handlers::redirect))
        // debug-only endpoint สำหรับสาธิต timeout/graceful-shutdown ในหัวข้อ 108.6 — ระบบจริง
        // ไม่ควรมี endpoint แบบนี้เปิดอยู่
        .route("/debug/sleep/{millis}", get(handlers_debug::sleep_ms))
        .route_layer(axum::middleware::from_fn_with_state(
            state.clone(),
            mw::rate_limit,
        ));

    // กลุ่ม "operational surface" (health check, metrics) — **ไม่ผ่าน rate limiter เดียวกัน**
    // เพราะถูกเรียกโดย infrastructure ภายใน (Kubernetes probe, Prometheus scraper) ที่ความถี่
    // คงที่และเชื่อถือได้ ไม่ใช่ traffic จากอินเทอร์เน็ตทั่วไป
    let ops_routes = Router::new()
        .route("/health/live", get(handlers::liveness))
        .route("/health/ready", get(handlers::readiness))
        .route("/metrics", get(handlers::metrics_handler));

    Router::new()
        .merge(public_routes)
        .merge(ops_routes)
        // ลำดับของสอง `.route_layer()` นี้สำคัญมาก — ดูกับดักที่ 4 ท้ายบทที่พิสูจน์จริงว่า ถ้า
        // สลับลำดับ (เอา metrics ไว้ "ใน" timeout) request ที่ค้างจน timeout จะ**ไม่ถูกนับใน
        // metrics เลย** เพราะ future ของ middleware วัด metrics ถูก cancel ก่อนจะรันถึงบรรทัดที่
        // บันทึกค่า — เรียง `TimeoutLayer` ไว้ก่อน (ชั้นใน) แล้ว `track_http_metrics` ไว้หลัง (ชั้น
        // นอก) เพื่อให้ metrics middleware เป็นคนสุดท้ายที่เห็น response ไม่ว่า response นั้นจะมา
        // จาก handler จริงหรือมาจาก `TimeoutLayer` ที่ตัดจบเอง
        .route_layer(TimeoutLayer::with_status_code(
            axum::http::StatusCode::REQUEST_TIMEOUT,
            Duration::from_secs(timeout_secs),
        ))
        .route_layer(axum::middleware::from_fn(mw::track_http_metrics))
        // security header ใส่ให้ทุก response แม้แต่ 404/500 (.layer ครอบทั้ง Router)
        .layer(axum::middleware::from_fn(mw::security_headers))
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}

/// เก็บ debug endpoint ไว้แยก module ให้ชัดเจนว่าไม่ใช่ business logic ของระบบจริง
mod handlers_debug {
    use axum::extract::Path;
    use std::time::Duration;

    pub async fn sleep_ms(Path(millis): Path<u64>) -> String {
        tokio::time::sleep(Duration::from_millis(millis)).await;
        format!("slept {millis}ms")
    }
}
```

`Cargo.toml` ฉบับสมบูรณ์ (เวอร์ชันจริงที่ compile ผ่านและตรวจสอบแล้ว):

```toml
[package]
name = "urlshortener"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1.0.104"
async-trait = "0.1.92"
axum = { version = "0.8.9", features = ["json"] }
chrono = { version = "0.4.45", features = ["serde"] }
deadpool-redis = { version = "0.23.1", features = ["rt_tokio_1"] }
governor = "0.10.4"
metrics = "0.24.6"
metrics-exporter-prometheus = "0.18.3"
opentelemetry = "0.33.0"
opentelemetry-otlp = { version = "0.33.0", features = ["grpc-tonic"] }
opentelemetry_sdk = "0.33.0"
rand = "0.10.3"
redis = { version = "1.7.1", features = ["script", "tokio-comp"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
sqlx = { version = "0.9.0", features = ["runtime-tokio", "postgres", "uuid", "chrono", "migrate"] }
thiserror = "2.0.21"
tokio = { version = "1.53.1", features = ["full"] }
tokio-util = "0.7.19"
tower = { version = "0.5.3", features = ["full"] }
tower-http = { version = "0.7.1", features = ["trace", "timeout", "limit", "compression-gzip"] }
tracing = "0.1.44"
tracing-opentelemetry = "0.34.0"
tracing-subscriber = { version = "0.3.23", features = ["env-filter", "json"] }
url = "2.5.8"
uuid = { version = "1.26.1", features = ["v4"] }
```

#### Incident Simulation: Database ถูก Lock ชั่วคราว — สังเกตพฤติกรรมที่เสื่อมลงจริง

จำลองสถานการณ์จริงที่เกิดขึ้นได้ในระบบ production: มี process อื่นมา lock ตารางค้างไว้ชั่วคราว (เช่น
migration ที่รันตอน deploy ทับซ้อนกับ traffic จริง, หรือ query ที่เขียนผิดแล้วถือ lock ค้างนาน) ทำให้
query ปกติที่ควรเร็วกลายเป็นช้ามาก:

```bash
# เปิด session แยกที่ lock ตาราง links แบบ ACCESS EXCLUSIVE (บล็อกทุก query รวมถึง SELECT) เป็น
# เวลา 8 วินาที จำลอง "database ช้าผิดปกติชั่วคราว"
psql -c "BEGIN; LOCK TABLE links IN ACCESS EXCLUSIVE MODE; SELECT pg_sleep(8); COMMIT;" &

# ยิง request ไปที่ redirect endpoint (short code ที่ยังไม่อยู่ใน cache → บังคับให้ query Postgres)
curl -sS -D - -o /tmp/incident_body.txt -w "HTTP_CODE=%{http_code} TIME_TOTAL=%{time_total}\n" \
  "http://127.0.0.1:8080/5mk3ORf"
```

ผลลัพธ์จริง:

```text
HTTP/1.1 408 Request Timeout
content-length: 0

HTTP_CODE=408 TIME_TOTAL=5.002717
```

request ค้างพอดี 5.0027 วินาที (ตรงกับ `request_timeout_secs=5` ที่ตั้งไว้) แล้วถูก `TimeoutLayer` ตัด
จบให้ตอบ `408` แทนที่จะรอ Postgres จนกว่า lock จะปลด — **นี่คือพฤติกรรมที่ตั้งใจออกแบบไว้**: ไม่ให้
request ค้างรอไม่มีกำหนดจน resource ของ server หมด

ที่สำคัญกว่านั้น — log ที่ระบบสร้างขึ้นเอง**ระหว่าง**เกิด incident ชี้ตรงไปที่ต้นเหตุทันทีโดยไม่ต้อง
สืบสวนเพิ่ม:

```text
2026-09-27T10:28:58.972459Z  WARN redirect{code="Ymn4r1R"}: sqlx::query: slow statement:
execution time exceeded alert threshold summary="SELECT code, long_url, created_at, …"
db.statement="SELECT code, long_url, created_at, expires_at, created_by, click_count FROM
links WHERE code = $1" rows_affected=0 rows_returned=0 elapsed=5.000847588s
elapsed_secs=5.000847588 slow_threshold=1s
```

สังเกตว่า log บรรทัดนี้มี**สามชั้นของ context ที่เชื่อมกันเอง**:

1. **span `redirect{code="Ymn4r1R"}`** (จาก `#[tracing::instrument]`) บอกว่านี่คือ request redirect
   ของ short code ตัวไหนเป๊ะ
2. **`sqlx::query` slow-statement warning** (ที่ SQLx สร้างให้อัตโนมัติเมื่อ query ช้าเกิน
   `slow_threshold` ที่ตั้งไว้) บอก SQL statement ตรง ๆ ที่ช้า
3. **`elapsed=5.000847588s`** บอกเวลาที่แท้จริงที่ query ใช้ ซึ่งตรงกับเวลาที่ `TimeoutLayer` ตัด
   request ทิ้งพอดี — ยืนยันว่าต้นเหตุของ timeout คือ query นี้เอง ไม่ใช่ปัญหาที่อื่น

ถ้าเป็นสถานการณ์จริงตอนตี 3 ทีม on-call จะเห็น dashboard ของ Prometheus แสดง p99 latency ของ
`GET /{code}` พุ่งขึ้นทันที (จาก `http_request_duration_seconds` histogram), เปิด Jaeger ดู trace ของ
request ที่ช้าเห็น span `redirect` ที่กินเวลาเกือบ 5 วินาทีเต็ม, แล้วเปิด log ตาม trace ID เดียวกันเจอ
บรรทัด `slow statement` ด้านบนที่บอกตรง ๆ ว่า query ไหนคือสาเหตุ — **นี่คือ observability triad
(metrics → traces → logs) ทำงานตามลำดับที่ Part 98/99 อธิบายไว้ทุกประการ**

หลังจาก lock ถูกปลด (8 วินาทีผ่านไป) ระบบกลับมาทำงานปกติทันทีโดยไม่ต้อง restart อะไร:

```bash
curl -sS -o /dev/null -w "HTTP_CODE=%{http_code} TIME_TOTAL=%{time_total}\n" \
  "http://127.0.0.1:8080/5mk3ORf"
```

```text
HTTP_CODE=307 TIME_TOTAL=0.001685
```

#### บั๊กจริงที่ค้นพบระหว่างจำลอง Incident: Timeout ที่ไม่ถูกนับใน Metrics

ระหว่างจำลอง incident ข้างบน มีสิ่งที่**ผิดปกติอย่างเงียบ ๆ**เกิดขึ้น — ตอนเช็ค `/metrics` หลังจาก
request ที่ได้ `408` ไปแล้ว:

```bash
curl -sS http://127.0.0.1:8080/metrics | grep 'status="408"'
```

```text
(ไม่มี output อะไรเลย)
```

**ไม่มี metric ของ status `408` เลย** ทั้งที่ request นั้นตอบ `408` จริงตามที่ `curl` ยืนยันไว้แล้ว — นี่
คือบั๊กจริงที่ค้นพบระหว่างการตรวจสอบบทนี้ ไม่ใช่ตัวอย่างที่แต่งขึ้น

**สาเหตุ**: ลำดับ layer เดิม (ก่อนแก้) คือ

```rust
.route_layer(axum::middleware::from_fn(mw::track_http_metrics)) // ก่อน (ผิด)
.layer(axum::middleware::from_fn(mw::security_headers))
.layer(TimeoutLayer::with_status_code(...))  // อยู่นอก metrics middleware
```

`TimeoutLayer` ที่ใส่ผ่าน `.layer()` จะกลายเป็น**ชั้นนอก**ของทุกอย่างที่ใส่ไว้ก่อนหน้า (รวมถึง
`track_http_metrics` ที่ใส่ผ่าน `.route_layer()`) — เมื่อ request ค้างเกิน timeout `TimeoutLayer` จะ
**ยกเลิก (cancel) future ของทุกอย่างที่อยู่ข้างใน** รวมถึง future ของ `track_http_metrics` เอง แล้วสร้าง
response `408` ขึ้นมาเองโดยตรง — โค้ดส่วนที่บันทึก metric ใน `track_http_metrics` (ที่อยู่**หลัง**
`next.run(req).await`) **ไม่มีโอกาสได้รันเลย** เพราะ future ทั้งก้อนถูกทิ้งไปกลางทางก่อนจะไปถึงจุดนั้น

**วิธีแก้ที่พิสูจน์แล้วว่าได้ผล**: เปลี่ยนทั้งสอง layer ให้เป็น `.route_layer()` ทั้งคู่ แล้วเรียงลำดับ
ให้ `TimeoutLayer` เป็น**ชั้นใน**และ `track_http_metrics` เป็น**ชั้นนอก** (การเรียก `.route_layer()`
ครั้งหลังจะห่อครั้งก่อนไว้ข้างใน):

```rust
.route_layer(TimeoutLayer::with_status_code(...))              // ชั้นใน — รันก่อน
.route_layer(axum::middleware::from_fn(mw::track_http_metrics)) // ชั้นนอก — เห็น response สุดท้ายเสมอ
```

รันซ้ำ incident เดียวกันหลังแก้ไข ได้ผลลัพธ์ที่ถูกต้อง:

```bash
curl -sS -o /dev/null -w "HTTP_CODE=%{http_code} TIME_TOTAL=%{time_total}\n" "http://127.0.0.1:8080/Ymn4r1R"
```

```text
HTTP_CODE=408 TIME_TOTAL=5.002580
```

```bash
curl -sS http://127.0.0.1:8080/metrics | grep 'status="408"'
```

```text
http_requests_total{method="GET",path="/{code}",status="408"} 1
```

**นี่คือตัวอย่างจริงว่าทำไม "การทดสอบด้วยการรันจริง" สำคัญกว่าการอ่านโค้ดแล้วสรุปว่าถูกต้อง** — โค้ด
ก่อนแก้ไข compile ผ่านสมบูรณ์ ไม่มี warning ไม่มี error และ**ดูเหมือนถูกต้องทุกอย่างตอนอ่านผ่าน ๆ** แต่
พฤติกรรมจริงภายใต้เงื่อนไข timeout (ซึ่งเกิดขึ้นได้จริงในระบบ production เมื่อ database ช้า) กลับทำให้
สูญเสียการมองเห็น (observability) ตรงจุดที่สำคัญที่สุด — จุดที่ระบบกำลังมีปัญหา — ซึ่งเป็นปัญหาที่แย่
กว่าการไม่มี metrics เลย เพราะมันทำให้ dashboard "ดูปกติดี" ทั้งที่จริง ๆ มี request กำลัง timeout อยู่
เงียบ ๆ

## กับดักที่พบบ่อย (Common Pitfalls)

1. **`rand` 0.10 เปลี่ยน API ของ `sample_iter` — trait ไม่ได้อยู่ใน scope โดยอัตโนมัติ**: ถ้าเขียนตาม
   ความเคยชินจาก `rand` เวอร์ชันเก่า (`rand::thread_rng().sample_iter(...)`) โดยไม่ import trait ที่
   ถูกต้องของเวอร์ชัน 0.10 จะได้ compile error จริงแบบนี้:

   ```text
   error[E0599]: no method named `sample_iter` found for struct `ThreadRng` in the current scope
      --> src/domain.rs:27:10
      |
   26 |       rand::rng()
   27 | |         .sample_iter(&Alphanumeric)
      | |_________-^^^^^^^^^^^
   help: trait `RngExt` which provides `sample_iter` is implemented but not in scope; perhaps you want to import it
      |
   1 + use rand::RngExt;
      |
   ```

   วิธีแก้ตรงตามที่ compiler แนะนำ — เพิ่ม `use rand::RngExt;` — เป็นตัวอย่างที่ดีของ Rust compiler
   error message ยุคใหม่ที่ไม่ได้แค่บอกว่า "ผิด" แต่บอก**วิธีแก้ที่ถูกต้องเป๊ะ**มาให้เลย (ต่อยอดจาก
   Part 5 ที่สอนให้อ่าน compiler error อย่างละเอียด) — เหตุผลเชิงลึกที่ API เปลี่ยนแบบนี้: การแยก
   `sample_iter` ไปอยู่ใน trait ต่างหาก (`RngExt`) ทำให้ `Rng` trait หลักเล็กลงและโฟกัสเฉพาะ method
   ที่จำเป็นต่อการ implement generator ใหม่ ในขณะที่ method "convenience" อย่าง `sample_iter` แยกไป
   อยู่ extension trait ที่ import เพิ่มเมื่อต้องใช้เท่านั้น

2. **ลืมเปิด feature `script` ของ crate `redis` — `Script` ไม่มีให้ import**: `deadpool-redis` ไม่ได้
   เปิด feature `script` ของ `redis` ให้อัตโนมัติ (เพราะไม่ใช่ทุกคนต้องการใช้ Lua script) ถ้าเขียน
   `use deadpool_redis::redis::Script;` ตรง ๆ โดยไม่เพิ่ม dependency `redis` ของตัวเองพร้อม feature
   `script` จะได้ error จริงแบบนี้:

   ```text
   error[E0432]: unresolved import `deadpool_redis::redis::Script`
     --> src/ratelimit.rs:1:5
      |
    1 | use deadpool_redis::redis::Script;
      |     ^^^^^^^^^^^^^^^^^^^^^^^------
      |                            |
      |                            no `Script` in the root
      |
   note: found an item that was configured out
      |
   671 | #[cfg(feature = "script")]
      |       ------------------ the item is gated behind the `script` feature
   ```

   วิธีแก้: เพิ่ม `redis` เป็น dependency ของตัวเองพร้อม feature ที่ต้องการ (`cargo add redis
   --features script,tokio-comp`) — Cargo จะรวม (unify) feature ของ `redis` เวอร์ชันเดียวกันที่ถูก
   ดึงมาทั้งจาก `deadpool-redis` โดยตรงและจาก dependency ของเราเองเข้าด้วยกันอัตโนมัติ ทำให้ feature
   `script` ถูกเปิดใช้งานจริงในทุกที่ที่ crate `redis` ตัวเดียวกันนี้ถูกใช้ ไม่ต้องแก้ dependency ของ
   `deadpool-redis` เอง

3. **Partial index ที่ใช้ `now()` ใน predicate — Postgres ปฏิเสธเพราะ `now()` ไม่ IMMUTABLE**: ความ
   คิดที่ดูสมเหตุสมผลตอนออกแบบ schema คือสร้าง partial index เฉพาะ "ลิงก์ที่ยังไม่ expire"
   (`WHERE expires_at > now()`) เพื่อให้ query ที่กรองแต่ลิงก์ที่ยัง active เร็วขึ้น — แต่ Postgres
   ปฏิเสธ migration นี้ทันทีด้วย error จริง:

   ```text
   error returned from database: functions in index predicate must be marked IMMUTABLE
   ```

   สาเหตุเชิงลึก: index ต้องมี predicate ที่ให้ผลลัพธ์**เดียวกันเสมอสำหรับข้อมูลชุดเดียวกัน** ไม่ว่า
   จะ query เมื่อไหร่ก็ตาม (เพื่อให้ Postgres รู้ล่วงหน้าตอนสร้าง/บำรุง index ว่าแถวไหนควรอยู่ใน index)
   แต่ `now()` คืนค่าต่างกันทุกครั้งที่เรียก (ขึ้นกับเวลาปัจจุบัน) จึงไม่ตรงตามข้อกำหนดนี้ ถูก Postgres
   classify เป็น `STABLE` ไม่ใช่ `IMMUTABLE` — วิธีแก้: สร้าง btree index ปกติบน column `expires_at`
   ตรง ๆ (ไม่ใส่ predicate ที่พึ่งเวลาปัจจุบัน) แล้วให้ query ที่เรียกจริงเป็นคนกำหนด threshold ของเวลา
   เอง (`WHERE expires_at > NOW()` ใน query ธรรมดา ไม่ใช่ใน index definition ใช้ได้ปกติ)

4. **ยิง Integration Test พร้อมกันหลายตัวเกิน `max_connections` ของ Postgres — `PoolTimedOut`**:
   เขียน integration test สอง function ที่แต่ละตัวเปิด connection pool ขนาดใหญ่ของตัวเอง (เช่น 60 +
   110 = 170 connections รวมกัน) แล้วรันด้วย `cargo test` ที่ execute ทุก test function **พร้อมกัน
   โดย default** (คนละ thread แต่โปรเซสเดียวกัน) จะชนกับ `max_connections` เริ่มต้นของ Postgres ทั้ง
   คลัสเตอร์ (ปกติคือ 100) แล้วได้ panic จริงแบบนี้:

   ```text
   thread 'concurrent_click_increments_are_atomic' panicked at tests/concurrency_proof.rs:128:18:
   increment should not fail: PoolTimedOut
   ```

   จุดที่ทำให้กับดักนี้น่าสับสน: error ดูเหมือนบอกว่า "ปัญหาเรื่อง concurrency safety ของแอป" ทั้งที่
   จริง ๆ เป็นแค่**ปัญหาเรื่องการตั้งค่า connection limit ของเทสต์เอง** ไม่เกี่ยวกับความถูกต้องของ
   `ON CONFLICT DO NOTHING`/`UPDATE ... SET x = x + 1` ที่เทสต์ต้องการพิสูจน์เลย — วิธีแก้: ลดจำนวน
   concurrent task ในแต่ละเทสต์ให้พอดีกับ connection limit ที่มี โดยคำนวณผลรวมของทุกเทสต์ที่อาจรัน
   พร้อมกันไว้ล่วงหน้า (บทนี้ใช้ 30 task ต่อเทสต์ ให้ผลรวมสองเทสต์ไม่เกิน ~80 connections ปลอดภัยเทียบ
   กับ limit 100) หรือรัน `cargo test -- --test-threads=1` ถ้าจำเป็นต้องใช้ concurrency สูงกว่านั้น
   จริง ๆ ในเทสต์เดียว

5. **`TimeoutLayer::new` deprecated — ต้องใช้ `with_status_code` แทน**: เวอร์ชัน `tower-http` ที่ใหม่
   กว่าเปลี่ยน default behavior ของ `TimeoutLayer::new` ให้ deprecated พร้อม warning จริง:

   ```text
   warning: use of deprecated associated function `tower_http::timeout::TimeoutLayer::new`:
   Use `TimeoutLayer::with_status_code` instead
      --> src/main.rs:155:30
       |
   155 |         .layer(TimeoutLayer::new(Duration::from_secs(timeout_secs)))
       |                              ^^^
   ```

   วิธีแก้ตรงไปตรงมาตามที่ warning บอก — เปลี่ยนเป็น `TimeoutLayer::with_status_code(StatusCode::
   REQUEST_TIMEOUT, duration)` ที่ให้ควบคุม status code ที่ตอบกลับเมื่อ timeout ได้ชัดเจน (ค่า
   default เดิมของ `::new` คือ `408` อยู่แล้ว แต่ API ใหม่บังคับให้เขียนชัดเจนแทนการพึ่ง default ที่
   ซ่อนอยู่)

6. **ลำดับ middleware ผิด ทำให้ request ที่ timeout หายไปจาก metrics เงียบ ๆ**: พิสูจน์ไว้เต็มรูปแบบ
   แล้วในหัวข้อ 108.11 — ถ้าวาง `TimeoutLayer` เป็น**ชั้นนอก**ของ middleware ที่วัด metrics (เช่นใส่
   ผ่าน `.layer()` ในขณะที่ metrics middleware ใส่ผ่าน `.route_layer()` ซึ่งอยู่ชั้นในกว่าเสมอ) ทุก
   request ที่ค้างจน `TimeoutLayer` ต้องตัดจบเอง จะทำให้ future ของ metrics middleware ถูก cancel
   ก่อนถึงบรรทัดที่บันทึกค่า — ผลคือ `/metrics` **ไม่มี** entry ของ status `408` เลย ทั้งที่ระบบตอบ
   `408` จริงตามที่ยืนยันด้วย `curl` ได้ วิธีตรวจจับกับดักนี้ในระบบจริง: หลังตั้ง timeout ใหม่ ให้ทดสอบ
   ด้วยการยิง request ที่บังคับให้ timeout จริง (เช่นผ่าน debug endpoint ที่ sleep นานกว่า timeout)
   แล้วเช็ค `/metrics` ว่ามี entry ของ status code ที่ timeout ตอบ (`408` หรือค่าที่ตั้งไว้) ปรากฏจริง
   — ถ้าไม่มี แปลว่าลำดับ layer ผิด ต้องสลับให้ metrics middleware เป็นชั้นนอกกว่า `TimeoutLayer`
   เสมอ (ทำผ่าน `.route_layer()` ทั้งคู่ แล้วเรียง `TimeoutLayer` ก่อน `track_http_metrics` ตามที่
   หัวข้อ 108.11 แก้ไว้)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม Counter metric ชื่อ `expired_link_access_attempts_total` ที่นับจำนวนครั้งที่มีคน
   พยายามเข้าลิงก์ที่ expire ไปแล้ว (เพิ่มโค้ดในจุดที่ handler `redirect` คืน `AppError::Expired`) —
   พิสูจน์ด้วยการสร้างลิงก์ที่ `ttl_seconds: 1`, รอ 2 วินาที, ยิง request ไปที่ลิงก์นั้น 3 ครั้ง แล้วดู
   ค่าใน `/metrics` ว่าตรงกับ 3 จริง *(hint: เพิ่ม `metrics::counter!("expired_link_access_attempts_total").increment(1);`
   ก่อนบรรทัด `return Err(AppError::Expired);` ใน `redirect`)*

2. **(กลาง)** implement endpoint `DELETE /api/links/{code}` ที่ลบลิงก์ออกจาก database **และ**
   invalidate cache ของ code นั้นพร้อมกัน (ใช้ฟังก์ชัน `cache::invalidate` ที่เขียนเตรียมไว้แล้วใน
   `cache.rs` แต่ยังไม่มี endpoint เรียกใช้จริงในบทนี้) — ต้องเพิ่ม method ใหม่ใน `LinkRepository`
   trait ด้วย (`delete_by_code`) แล้ว implement ทั้ง trait และ `PostgresLinkRepository` พร้อมพิสูจน์
   ด้วยการสร้างลิงก์ → เรียก `GET /{code}` ให้ cache ติด → ลบลิงก์ → เรียก `GET /{code}` อีกครั้งแล้ว
   ต้องได้ `404` ทันที (ไม่ใช่ redirect ไปที่ URL เดิมจาก cache ที่ยังไม่ถูกลบ) *(hint: ถ้าลืม
   invalidate cache หลังลบ database request ถัดไปจะยัง cache-hit แล้ว redirect ไปที่ URL เดิมได้อยู่
   แม้ database ไม่มีแถวนั้นแล้ว — นี่คือเหตุผลที่ handler ต้องเรียกทั้งสองอย่างตามลำดับ ลบ database
   ก่อนแล้วค่อย invalidate cache เพื่อไม่ให้มี window ที่ cache ยังมีอยู่แต่ database ไม่มีแล้ว)*

3. **(ยาก)** โค้ดในหัวข้อ 108.3 ระบุไว้ตรง ๆ ว่าการตรวจ SSRF ปัจจุบันครอบคลุมแค่กรณี host เป็น IP
   literal เท่านั้น ไม่ครอบคลุมกรณี DNS rebinding (domain name ที่ resolve ไปที่ private IP) — ให้
   ออกแบบ (เขียนเป็น pseudocode หรือโค้ดจริงถ้าต้องการ) วิธีป้องกัน DNS rebinding แบบสมบูรณ์กว่านี้ 1
   วิธี พร้อมอธิบาย trade-off ของมัน *(hint: แนวทางหนึ่งคือเขียน custom DNS resolver ที่ resolve
   hostname เป็น IP ก่อน ตรวจ IP ที่ได้ด้วย `is_disallowed_ip` แล้วค่อยส่งต่อ IP ที่ resolve แล้วไปยัง
   HTTP client ที่ทำ connection จริง (ปฏิเสธ redirect ที่ resolve ใหม่ระหว่างทางด้วย) — trade-off คือ
   ต้อง resolve DNS เองสองครั้ง (ตอน validate และตอน connect จริง) และต้องจัดการ TTL ของ DNS cache
   เองเพื่อไม่ให้ IP เปลี่ยนไปเป็น private ระหว่างสองครั้งนั้น)*

4. **(ยาก/ประยุกต์ใช้งานจริง)** จากสูตร birthday paradox ในหัวข้อ 108.4 ($p \approx 1 -
   e^{-n^2/2N}$ โดย $N = 62^{L}$ คือขนาด keyspace ที่ความยาว code เท่ากับ $L$) คำนวณว่าต้องใช้
   `CODE_LENGTH` เท่าไหร่ถึงจะทำให้ความน่าจะเป็นที่จะเกิด collision อย่างน้อยหนึ่งครั้งต่ำกว่า 0.01%
   ($p < 0.0001$) เมื่อระบบมีลิงก์สะสมถึง 100 ล้านลิงก์ ($n = 10^8$) แล้วปรับ `CODE_LENGTH` ในโค้ด
   ให้เป็นค่าที่คำนวณได้ พร้อมเขียนเหตุผลว่าทำไมการเพิ่มความยาว code ไม่กระทบ backward compatibility
   ของลิงก์ที่มีความยาวเดิมอยู่แล้วในระบบ (เพราะ column `code` เป็น `TEXT` ไม่ใช่ `CHAR(7)` ตายตัว)
   *(hint: ที่ $n = 10^8$ ต้องการ $N \gg n^2 / (2 \times 0.0001) = 10^{16}/0.0002 = 5\times10^{19}$
   — คำนวณ $62^L$ สำหรับ $L$ ต่าง ๆ แล้วหาค่า $L$ ที่น้อยที่สุดที่ทำให้ $62^L$ เกินตัวเลขนี้ (ลองคำนวณ
   ด้วยมือหรือเขียนโปรแกรมเล็ก ๆ ช่วยคำนวณ) คำตอบที่ควรได้อยู่ที่ประมาณ `CODE_LENGTH = 11-12` ตัวอักษร
   ไม่ใช่ 7 ตัวเดิม ถ้าต้องการรองรับสเกลระดับ 100 ล้านลิงก์อย่างปลอดภัยจริง)*

## สรุป

บทนี้สร้าง **URL shortener ระดับ production เต็มรูปแบบ** ตั้งแต่การออกแบบ non-functional
requirement และ API contract (108.1), วางโครงสร้างด้วย **hexagonal architecture** ที่แยก port
(`LinkRepository`) ออกจาก adapter (`PostgresLinkRepository`) ตั้งแต่บรรทัดแรก (108.2), เขียน domain
logic ที่ป้องกัน open redirect/SSRF จริง (108.3), พิสูจน์ **collision-safe short code generation**
ด้วย integration test ที่ยิง concurrent task จริงแข่งกันเขียนกับ Postgres จริง (108.4), เดินสายครบทั้ง
**สามเสาหลักของ observability** (logs + metrics + traces) แล้วยืนยันด้วยการรัน Postgres, Redis,
Prometheus, และ Jaeger จริงพร้อมกัน (108.5), ทำ **resilience** ครบสามด้าน — rate limiting ที่ผูก state
ไว้ที่ Redis, timeout ที่ตัด request ค้าง, และ graceful shutdown ที่พิสูจน์ด้วยการยิง SIGTERM ขณะมี
request ทำงานอยู่จริง (108.6), ทำ **cache-aside caching** ที่วัด throughput เพิ่มขึ้นจริง 64.9% ด้วย
`wrk` (108.7), เลือก **index** ที่ถูกต้องพร้อมกับดักจริงที่เจอตอนเขียน migration (108.8), ทำ
**security hardening** สามด้าน (108.9), แยก **liveness/readiness** อย่างถูกต้องพร้อม `Dockerfile`
(108.10), และปิดท้ายด้วยการรันระบบทั้งหมดพร้อมกันจริงแล้ว**จำลอง incident จริง**ที่นำไปสู่การค้นพบและ
แก้บั๊กจริงเรื่องลำดับ middleware ที่ทำให้ metrics มองไม่เห็น timeout (108.11)

**สิ่งที่สำคัญที่สุดที่บทนี้พยายามสื่อ**: production-readiness ไม่ใช่ checklist ของฟีเจอร์ที่ทำเสร็จ
แล้วก็จบ แต่คือ**กระบวนการพิสูจน์อย่างต่อเนื่อง**ว่าระบบทำงานถูกต้องภายใต้เงื่อนไขจริง (concurrency,
failure ของ dependency, load สูง, timeout) — ทุกตัวเลขและทุก log ในบทนี้มาจากการรันจริง ไม่ใช่การ
คาดเดา และบั๊กที่ค้นพบในหัวข้อ 108.11 คือหลักฐานที่ชัดที่สุดว่าทำไม "compile ผ่านและดูโค้ดแล้วถูกต้อง"
ไม่เพียงพอสำหรับระบบที่จะรันจริงใน production — ต้องทดสอบพฤติกรรมภายใต้เงื่อนไขขอบ (edge condition)
จริงเสมอ

ด้วยบทนี้ หลักสูตรได้ปิดคู่ capstone project สองบท (**Part 107**: CLI tool, **Part 108**: web
service) ที่ครอบคลุมรูปแบบแอปพลิเคชัน Rust ระดับ production สองแบบหลักที่นักพัฒนา Rust พบเจอบ่อยที่สุด
ในโลกจริง — บทถัดไป (**Part 109**) จะเปลี่ยนโฟกัสจาก "สร้างระบบใหม่" ไปที่ "ทบทวนและปรับปรุงคุณภาพของ
โค้ดที่มีอยู่แล้ว" ผ่าน code review, refactoring, และหลักการ clean code ที่ปรับให้เข้ากับธรรมชาติของ
Rust โดยเฉพาะ

---

**Part ก่อนหน้า:** [Capstone: Building a Production-Grade CLI Tool](part-107-capstone-cli-tool.md) |
**Part ถัดไป:** [Code Review, Refactoring และ Clean Code ใน Rust](part-109-code-review-refactoring.md)
