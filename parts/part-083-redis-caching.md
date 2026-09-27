# Part 83: Caching ด้วย Redis

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างเจาะจงว่า Redis คืออะไร (in-memory key-value store ที่มี data structure หลากหลายกว่าที่คำว่า
  "key-value" ฟังดู) และทำไมมันถึงเป็นคำตอบมาตรฐานสำหรับปัญหาที่ Part 75 เกริ่นไว้ตั้งแต่หัวข้อ 75.8: **shared
  state ที่ทุก server instance เข้าถึงร่วมกันได้ ในความเร็วระดับ sub-millisecond**
- ต่อ Redis จริงด้วย crate `redis` เวอร์ชันปัจจุบันจาก crates.io ผ่าน `redis::AsyncCommands` (async API ที่ผูก
  กับ Tokio runtime ตามที่ Part 48 สอนไว้) และใช้คำสั่งพื้นฐาน `SET`/`GET`/`EXPIRE` ได้อย่างถูกต้อง
- Implement **cache-aside pattern** เต็มรูปแบบ: เช็ค Redis ก่อนเสมอ ถ้า miss ค่อยไป query PostgreSQL ผ่าน
  `PgPool` ที่ Part 70 สอนไว้ แล้ว populate cache กลับพร้อม TTL — พร้อม**วัด latency จริง**เทียบ cache hit กับ
  cache miss เพื่อพิสูจน์ว่า caching ช่วยได้จริงกี่เท่า ไม่ใช่แค่กล่าวอ้างลอย ๆ
- อธิบายและ implement **cache invalidation** สองแนวทาง (invalidate-on-write กับ TTL-only) พร้อมพิสูจน์ด้วยโค้ด
  จริงว่า TTL-only เพียงอย่างเดียวปล่อยให้เกิด **stale read** ได้จริง และ invalidate-on-write แก้ปัญหานี้อย่างไร
- **ปิดสามจุดที่ Part 74, 75, และ 78 เกริ่นไว้แล้วอย่างชัดเจนว่า "รอ Part 83"**: สร้าง Redis-backed session
  store แทน `MemoryStore` ที่ Part 75 ใช้ (พิสูจน์ว่าสอง server instance อ่าน session ของกันและกันได้จริง),
  implement idempotency-key store และ rate limiter ด้วย Redis จริงแทน `HashMap` ที่ Part 78 ใช้ชั่วคราว (พิสูจน์
  atomicity ด้วยการยิง concurrent request จริง), และสร้างระบบ revoke refresh token ที่ Part 74 sketch ไว้ให้
  ทำงานได้จริงด้วย Redis
- ใช้ data structure ของ Redis ที่ไม่ใช่ string ธรรมดา — **Sorted Set** สำหรับ leaderboard และ **Set** สำหรับ
  tag-based filtering — และอธิบายได้ว่าทำไมสอง data structure นี้ทำงานบางอย่างได้ง่ายและเร็วกว่าการพยายามทำ
  แบบเดียวกันด้วย SQL query ล้วน ๆ บน PostgreSQL
- ออกแบบระบบให้ **degrade อย่างสุภาพ (graceful degradation)** เมื่อ Redis เองล่ม ไม่ใช่ปล่อยให้ทั้งแอปพัง —
  ตัดสินใจได้ว่าเมื่อไร Redis ล่มควรเป็น hard failure เมื่อไรควร fallback ไปที่ฐานข้อมูลตรง ๆ พร้อมพิสูจน์ด้วย
  การปิด Redis จริงแล้วดูว่าแอปยังตอบสนองได้

## ความรู้ที่ต้องมีมาก่อน

- **Part 39 (Mutex, Arc และ Shared-State Concurrency) และ Part 46-50 (Async/Tokio)**: บทนี้ใช้
  `redis::AsyncCommands` ซึ่งเป็น async API ล้วน ๆ ที่ต้อง `.await` ทุกคำสั่ง — ถ้าความแตกต่างระหว่าง
  synchronous กับ asynchronous I/O และเหตุผลที่ network round-trip ควรเป็น `async fn` ยังไม่แน่น ควรกลับไปทวน
  ก่อน เพราะบทนี้จะไม่อธิบายกลไก async พื้นฐานซ้ำ
- **Part 61 (HTTP Fundamentals)** และ **Part 64 (Axum: State Management)**: `AppState` ที่เก็บทั้ง `PgPool`
  และ `redis::Client` พร้อมกันในบทนี้ ต่อยอดจาก pattern เดียวกับที่ Part 64 สอนไว้ทุกประการ
- **Part 66 (Axum: Error Handling)**: หัวข้อ 83.10 เรื่อง graceful degradation ใช้หลักการเดียวกับที่ Part 66
  สอนเรื่อง "แยก error ที่ควรบอก user ตรง ๆ ออกจาก error ที่ควร log แล้วหาทางสำรอง" — Redis ล่มไม่ควรกลายเป็น
  `500` เสมอไปถ้ามีทางสำรองที่สมเหตุสมผลกว่า
- **Part 70 (SQLx: PostgreSQL)**: บทนี้คือ**จุดที่ Redis กับ PostgreSQL มาบรรจบกันจริง** — cache-aside pattern
  ในหัวข้อ 83.3 ใช้ `PgPool` ตัวเดียวกับที่ Part 70 สอนสร้างไว้ทุกประการ (`PgPoolOptions::new().connect(...)`)
  และแนวคิด "pool ของ connection แทนการเปิดใหม่ทุกครั้ง" ที่ Part 70 สอนไว้กับ PostgreSQL จะกลับมาซ้ำอีกครั้ง
  กับ Redis ในหัวข้อ 83.10 — ถ้าไม่แน่นเรื่อง `PgPoolOptions`/`PgPool`/`sqlx::query_as!` ควรกลับไปทวนก่อน
- **Part 74 (Authentication: JWT)**: บทนี้ **implement ต่อจากที่ Part 74 sketch ไว้** ในหัวข้อ 74.11/74.17 —
  ต้องเข้าใจโครงสร้าง `Claims` (โดยเฉพาะ `sub`, `exp`, `typ`) และปัญหาเรื่อง refresh token revocation ที่ Part
  74 อธิบายไว้แล้วว่า "ทำได้แค่ sketch รอ Part 83"
- **Part 75 (Authentication: Session-based และ OAuth2)**: บทนี้ **implement ต่อจากที่ Part 75 หัวข้อ 75.8
  sketch ไว้เป็นโครงร่างเปล่า** ("`let session_store = RedisStore::new(redis_pool);` // Part 83 จะเติมให้ครบ")
  — ต้องเข้าใจว่า `SessionStore` เป็น trait ที่ `tower-sessions` ใช้ และทำไม session-based auth ถึงต้องมี
  shared store เมื่อ scale หลาย instance (ตารางเปรียบเทียบในหัวข้อ 75.2 คือจุดอ้างอิงหลัก)
- **Part 78 (RESTful API Design)**: บทนี้ **implement ต่อจาก `Arc<Mutex<HashMap<...>>>` ที่ Part 78 หัวข้อ
  78.7/78.9 ใช้ชั่วคราว** ทั้ง idempotency-key store และ rate limiter — Part 78 บอกไว้ตรง ๆ ว่า "production
  จริงต้องย้ายไป Redis (Part 83)" บทนี้คือจุดที่ทำสิ่งนั้นจริง
- **หมายเหตุเรื่องการตรวจสอบเนื้อหา**: ทุกตัวอย่างในบทนี้ผู้เขียนตั้ง Redis จริง (`redis-server`) และ
  PostgreSQL จริงไว้ในสภาพแวดล้อมทดสอบแยกนอกหลักสูตร (ไม่กระทบไฟล์ใด ๆ ในโค้สนี้) แล้ว **compile และรันจริง**
  ทุกตัวอย่าง วัด latency จริง จับ race condition จริง และ capture curl transcript จริงทั้งหมด — รวมถึงจุดที่
  เจอปัญหาความเข้ากันไม่ได้ของเวอร์ชัน crate จริงระหว่างการทดสอบ (หัวข้อกับดักข้อ 1) ซึ่งจะรายงานตรงไปตรงมา
  พร้อม error message จริง ไม่ใช่การเดา

## เนื้อหา

### 83.1 Redis คืออะไรจริง ๆ และทำไมต้องเป็นตอนนี้

#### ทวนปัญหาที่ Part 75 ทิ้งไว้: "shared session store" คืออะไรกันแน่

Part 75 หัวข้อ 75.8 อธิบายไว้แล้วว่าเมื่อระบบ session-based authentication ต้อง deploy มากกว่า 1 instance
หลัง load balancer ตัวเดียว `MemoryStore` (ที่เก็บ session ไว้ใน `HashMap` ในหน่วยความจำของ process ตัวเอง)
ใช้ไม่ได้ผลอีกต่อไป เพราะ instance A ที่สร้าง session ไม่แชร์ `HashMap` กับ instance B/C เลย — ผู้ใช้จะถูก
"logout แบบสุ่ม" ทุกครั้งที่ load balancer ส่ง request ไปตกที่ instance ที่ไม่รู้จัก session ของตัวเอง

คำตอบที่ Part 75 ทิ้งไว้คือ **shared session store** — ที่เก็บข้อมูลกลางตัวเดียวที่ทุก instance เชื่อมต่อไปหา
เวลาจะอ่าน/เขียน session แทนการเก็บไว้ในหน่วยความจำของตัวเอง แต่คำถามที่ Part 75 ไม่ได้ตอบ (เพราะยังไม่ถึงเวลา)
คือ: **ที่เก็บกลางนี้ควรเป็นอะไร?**

ตัวเลือกที่ดูเหมือนตรงไปตรงมาที่สุดคือ "ก็ใช้ PostgreSQL ที่มีอยู่แล้วสิ" (ตาราง `sessions` มี column
`session_id`, `data`, `expires_at`) — และในทางเทคนิคก็ทำได้จริง หลายระบบทำแบบนี้ แต่มีต้นทุนแอบแฝงที่สำคัญ:
**session ถูกอ่าน/เขียนบ่อยกว่า business data ทั่วไปมาก** (ทุก request ที่ต้อง auth ต้องอ่าน session อย่างน้อย
หนึ่งครั้ง) ถ้าใช้ PostgreSQL เดียวกันกับที่เก็บข้อมูลธุรกิจหลัก (books, bookings, orders) session traffic ที่
สูงมากจะแย่ง connection pool และ I/O bandwidth ไปจากงานที่สำคัญกว่า — และ PostgreSQL ถูกออกแบบมาให้ทนทาน
(durability: เขียนลง disk, WAL, ACID เต็มรูปแบบ) ซึ่งเป็นคุณสมบัติที่ **session ข้อมูลชั่วคราวที่หมดอายุใน
ไม่กี่นาทีถึงไม่กี่วันไม่ได้ต้องการเลย** — จ่ายต้นทุนของ durability เต็มรูปแบบให้กับข้อมูลที่ตั้งใจจะทิ้งไปเอง
อยู่แล้วคือการสิ้นเปลืองที่ไม่จำเป็น

นี่คือจุดที่ **Redis** เข้ามา — มันถูกออกแบบมาสำหรับ "ข้อมูลที่ต้องอ่าน/เขียนเร็วมาก มีอายุจำกัด และไม่จำเป็น
ต้องทนทานเท่า transactional data หลัก" โดยเฉพาะ

#### Redis คือ In-Memory Data Structure Store ไม่ใช่แค่ "Key-Value Store"

คำนิยามที่พบบ่อยของ Redis คือ "in-memory key-value store" ซึ่ง**ถูกแต่ไม่ครบ** — ส่วน "in-memory" ถูกต้องเป๊ะ:
Redis เก็บข้อมูลทั้งหมดไว้ใน RAM เป็นหลัก (ไม่ใช่ disk แบบ PostgreSQL) ทำให้การอ่าน/เขียนเร็วกว่าฐานข้อมูลที่
เก็บบน disk อย่างมาก เพราะไม่มีต้นทุนของ disk seek/page cache miss มาเกี่ยวข้อง แต่ส่วน "key-value" ทำให้คนคิดว่า
Redis ทำได้แค่ `SET key value` / `GET key` แบบ dictionary ธรรมดา ทั้งที่ในความจริง **value ของ Redis มีได้
หลายชนิด (data structure) ไม่ใช่แค่ string**:

| Data Structure | คำสั่งหลัก | ใช้ทำอะไรได้บ้าง |
|---|---|---|
| **String** | `SET`/`GET`/`INCR`/`EXPIRE` | ค่าเดี่ยว ๆ (cache ของ object ที่ serialize เป็น JSON, counter, flag) |
| **Hash** | `HSET`/`HGET`/`HGETALL` | เก็บ field หลายตัวภายใต้ key เดียว (เหมือน object/struct ย่อยในตัวเอง) |
| **List** | `LPUSH`/`RPUSH`/`LRANGE` | ลำดับข้อมูลที่เรียงตามที่ใส่เข้าไป (queue, recent activity log) |
| **Set** | `SADD`/`SMEMBERS`/`SINTER` | กลุ่มสมาชิกไม่ซ้ำ ไม่มีลำดับ — เช็ค membership และ set operation (union/intersect) ได้เร็ว |
| **Sorted Set** | `ZADD`/`ZRANGE`/`ZINCRBY` | เหมือน Set แต่แต่ละสมาชิกมี "score" กำกับ — เรียงลำดับตาม score ได้อัตโนมัติเสมอ |

หัวข้อ 83.9 ท้ายบทนี้จะพาไปใช้ **Sorted Set** และ **Set** จริงกับปัญหาที่ทำให้เห็นว่าทำไม data structure พวกนี้
ประหยัด logic ฝั่งแอปพลิเคชันไปได้มากเมื่อเทียบกับพยายามทำสิ่งเดียวกันด้วย SQL

#### ความเร็ว: Sub-Millisecond Latency จริง ๆ แค่ไหน

คำว่า "Redis เร็ว" ไม่ใช่คำโฆษณาลอย ๆ — มาพิสูจน์ด้วยเครื่องมือ `redis-cli --latency` ที่ยิง `PING` ซ้ำ ๆ ไปที่
Redis server แล้ววัดเวลาไปกลับจริง (รันกับ Redis จริงที่ตั้งไว้ทดสอบบทนี้):

```bash
$ redis-cli -p 6390 --latency -i 1
```

```text
min: 0, max: 1, avg: 0.17 (99 samples)
```

**เวลาเฉลี่ยของการยิง command ไปที่ Redis แล้วได้ response กลับมา คือ 0.17 มิลลิวินาที** — เทียบกับ query
PostgreSQL ทั่วไปที่ผ่าน network + query planner + disk I/O (แม้จะมี index ที่ดีมาก) ที่มักอยู่ในระดับ
1-10+ มิลลิวินาที ตัวเลขนี้ยังไม่รวม "งานที่ทำ" เลย (แค่ `PING` เปล่า ๆ) แต่ก็ให้ภาพว่า **overhead ของการคุยกับ
Redis ต่ำกว่าฐานข้อมูลบน disk อย่างมีนัยสำคัญ** — หัวข้อ 83.3 จะวัดตัวเลขจริงอีกครั้งเทียบ cache hit กับ cache
miss ที่ต้องไป query PostgreSQL จริง เพื่อให้เห็นตัวเลขที่ตรงกับสถานการณ์ใช้งานจริงมากกว่าการ `PING` เปล่า ๆ

**เหตุผลเชิงลึกที่ Redis เร็วกว่า**: (1) ข้อมูลอยู่ใน RAM ทั้งหมด ไม่มี disk I/O มาเกี่ยวข้องระหว่างอ่าน/เขียน
ปกติ (2) Redis เป็น **single-threaded event loop** สำหรับการประมวลผลคำสั่ง (ตั้งแต่ Redis 6 ขึ้นไปมี I/O
thread เสริมสำหรับอ่าน/เขียน socket แต่ตรรกะการรันคำสั่งยังเป็น single-thread) ซึ่งฟังดูขัดกับสัญชาตญาณว่า
"multi-thread ต้องเร็วกว่า" แต่ในความเป็นจริง **การไม่ต้องใช้ lock ระหว่าง thread เพื่อป้องกัน race condition
ตอนแก้ไข data structure ทำให้ throughput ต่อ core สูงมาก** และตัด overhead ของ context switching ระหว่าง
thread ออกไปทั้งหมด (3) protocol ของ Redis (RESP — REdis Serialization Protocol) เรียบง่ายมาก ออกแบบมาให้
parse เร็ว ไม่มี overhead แบบ HTTP header/JSON schema ที่ซับซ้อน

**ข้อแลกที่ต้องเข้าใจคู่กัน**: การเก็บข้อมูลใน RAM แปลว่า **ถ้า Redis process ตาย ข้อมูลอาจหายไปทั้งหมด**
ถ้าไม่ตั้งค่า persistence ไว้ (Redis รองรับ persistence สองแบบ — RDB snapshot เป็นระยะ และ AOF log ทุกคำสั่ง —
นอกสโคปเชิงลึกของบทนี้ แต่ต้องรู้ว่ามีตัวเลือกนี้อยู่) และ**ขนาดข้อมูลทั้งหมดถูกจำกัดด้วย RAM ที่มี** ไม่ใช่
ด้วย disk space แบบ PostgreSQL — สองข้อนี้คือเหตุผลที่ Redis เหมาะกับ **ข้อมูลที่เร็วสำคัญกว่าทนทาน และข้อมูล
ที่มีขนาดจำกัด/มี TTL** (cache, session, rate limit counter, idempotency key) ไม่ใช่ตัวแทนของฐานข้อมูลหลักที่
เก็บ "ความจริงเดียว" ของระบบ (source of truth) แบบ PostgreSQL — บทนี้ทั้งบทจะใช้ Redis **คู่กับ** PostgreSQL
เสมอ ไม่ใช่แทนที่

#### Redis เทียบกับ Memcached: ทำไมหลักสูตรนี้เลือก Redis

คำถามที่มักเจอตอนเลือกเทคโนโลยี cache คือ "แล้ว Memcached ล่ะ?" — Memcached เป็น in-memory cache อีกตัวที่
เก่ากว่าและเรียบง่ายกว่า Redis มาก (เก็บได้แค่ string/binary blob ล้วน ๆ ไม่มี data structure อื่นเลย ไม่มี
persistence เลยแม้แต่แบบ RDB) — สำหรับงาน cache-aside แบบง่ายที่สุด (หัวข้อ 83.3) ทั้งสองตัวทำงานได้ใกล้เคียง
กันมาก แต่บทนี้เลือก Redis เพราะ **สามในหกงานที่บทนี้ต้องทำ (session store, idempotency-key claiming แบบ
atomic, rate limiting แบบ atomic, refresh token revocation, leaderboard, tag filtering) ต้องพึ่ง data
structure หรือ atomic primitive ที่ Memcached ไม่มีให้เลย** — `SET NX EX` แบบ atomic (หัวข้อ 83.6), Lua
script ผ่าน `EVAL` (หัวข้อ 83.7), Sorted Set (หัวข้อ 83.9), Set (หัวข้อ 83.9) ล้วนเป็นสิ่งที่ Redis มีแต่
Memcached ไม่มี — ในทางปฏิบัติ Memcached ยังถูกเลือกใช้ในระบบที่ต้องการ **แค่** cache แบบง่ายที่สุดจริง ๆ และ
ให้ความสำคัญกับความเรียบง่าย/เสถียรภาพระดับสูงสุดมากกว่าความยืดหยุ่น — แต่สำหรับระบบที่ต้องใช้ Redis เป็นทั้ง
cache และ shared state layer (ตามที่บทนี้ทั้งบทแสดงให้เห็น) Redis คือตัวเลือกที่ครอบคลุมกว่าอย่างชัดเจน

### 83.2 ติดตั้งและเชื่อมต่อจริง: crate `redis`, `AsyncCommands`, และคำสั่งพื้นฐาน

#### เลือก crate: `redis` เวอร์ชันปัจจุบัน

crate หลักสำหรับคุยกับ Redis จาก Rust มีสองตัวหลักที่ยังใช้งานจริงในปี 2026: **`redis`** (crate ดั้งเดิมที่ใช้
กันแพร่หลายที่สุด มี async API ผ่าน feature `tokio-comp`) และ **`fred`** (client รุ่นใหม่กว่าที่เขียนขึ้นมา
เพื่อ async-native ตั้งแต่ต้น รองรับ cluster/sentinel ได้ดีกว่าในหลายกรณี และเป็น client ที่
`tower-sessions-redis-store` ใช้ภายใน — จะเจอมันอีกครั้งในหัวข้อ 83.5) บทนี้เลือก **`redis`** เป็น client หลัก
สำหรับตัวอย่างทั่วไป เพราะ API ที่ตรงไปตรงมาและเอกสารที่ครบกว่าสำหรับการเริ่มต้น ตรวจสอบเวอร์ชันล่าสุดจริงจาก
crates.io ณ วันที่เขียนบทนี้ (`cargo add redis --features tokio-comp,connection-manager`) ได้ **`1.7.1`**

```toml
[dependencies]
redis = { version = "1.7.1", features = ["tokio-comp", "connection-manager"] }
tokio = { version = "1.53.1", features = ["full"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
anyhow = "1.0.104"
```

**อธิบาย feature flag**: `tokio-comp` เปิดใช้ async API ที่ผูกกับ Tokio runtime (ตามที่ Part 48 สอนไว้เรื่อง
`#[tokio::main]`) — ไม่มี feature นี้ crate `redis` จะให้แค่ synchronous API ซึ่งจะ block thread ทุกครั้งที่คุย
กับ Redis ผิดกับปรัชญา async ที่ Axum ทั้งหลักสูตรนี้ใช้มาตั้งแต่ Part 62 อย่างสิ้นเชิง `connection-manager`
เปิดใช้ `ConnectionManager` ที่จัดการ reconnect ให้อัตโนมัติถ้า connection หลุด — พฤติกรรมนี้เห็นได้จริงตอน
ทดสอบ graceful degradation ในหัวข้อ 83.10: หลังปิด Redis แล้วเปิดกลับมาใหม่ log ของแอปที่ใช้ client ตระกูลนี้
(ในกรณีของ `fred` ที่ session store ใช้ภายใน) จะแสดงความพยายาม reconnect ให้เห็นเองโดยไม่ต้อง restart แอป
ฝั่ง Rust เลย — จะกลับมาพูดถึงรายละเอียดนี้อีกครั้งในหัวข้อ 83.10

#### `SET`/`GET`/`EXPIRE` พื้นฐานผ่าน `redis::AsyncCommands`

trait **`AsyncCommands`** คือจุดเริ่มต้นของทุกคำสั่งที่ใช้ในบทนี้ — import แล้วเรียก method บน connection ได้
ตรง ๆ เหมือนเรียก method ธรรมดา (ไม่ต้องเขียน raw command string เอง แม้ว่าจะทำได้ผ่าน `redis::cmd(...)` ก็ตาม
ซึ่งจะเห็นในหัวข้อ 83.6/83.7 ตอนต้องใช้ flag ที่ยังไม่มี method สำเร็จรูปให้):

```rust
use redis::AsyncCommands;

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    // เชื่อมต่อ Redis -- คล้าย PgPoolOptions::connect() ของ Part 70 แต่เรียบง่ายกว่ามาก
    // (ไม่มี pool ในตัวอย่างนี้ก่อน -- หัวข้อ 83.10 จะเพิ่ม pool ให้)
    let client = redis::Client::open("redis://127.0.0.1:6390/")?;
    let mut conn = client.get_multiplexed_async_connection().await?;

    // SET: บันทึกค่าเข้า key -- ค่าที่ไม่มี type parameter ให้ Rust infer เอง (คืนค่า () เพราะไม่สนใจผลลัพธ์)
    let _: () = conn.set("greeting", "สวัสดี Redis").await?;

    // GET: อ่านค่าคืน -- ระบุ type ที่ต้องการ deserialize เป็น (ที่นี่คือ String)
    let value: String = conn.get("greeting").await?;
    println!("ค่าที่อ่านได้: {value}");

    // EXPIRE: ตั้ง TTL (Time To Live) ให้ key ที่มีอยู่แล้ว หน่วยเป็นวินาที
    let _: () = conn.expire("greeting", 30).await?;

    // TTL: เช็คเวลาที่เหลือก่อนหมดอายุ
    let ttl: i64 = conn.ttl("greeting").await?;
    println!("TTL ที่เหลือ: {ttl} วินาที");

    Ok(())
}
```

รันจริงผ่าน `redis-cli` ตรง ๆ (ไม่ผ่าน Rust) ให้เห็นภาพเดียวกันจากมุมมองของ command line ก่อน:

```bash
$ redis-cli -p 6390 set greeting "สวัสดี Redis"
OK
$ redis-cli -p 6390 get greeting
"สวัสดี Redis"
$ redis-cli -p 6390 expire greeting 30
(integer) 1
$ redis-cli -p 6390 ttl greeting
(integer) 30
$ redis-cli -p 6390 type greeting
string
```

**สังเกตค่าที่ `EXPIRE` คืนกลับ**: `(integer) 1` หมายถึง "ตั้ง TTL สำเร็จ" (ถ้า key ไม่มีอยู่จริงจะได้ `0`
กลับมาแทน) — และ `TYPE` ยืนยันว่า key นี้เป็นชนิด `string` (หนึ่งใน 5 data structure จากตารางหัวข้อ 83.1)

**สิ่งสำคัญที่ต้องรู้ตั้งแต่ต้น**: `SET` เพียว ๆ (ไม่มี `EX`) **ไม่มี TTL** — key จะอยู่ตลอดไปจนกว่าจะถูกลบมือ
หรือถูก `EXPIRE` ทีหลัง นี่คือกับดักข้อ 3 ท้ายบทที่จะพิสูจน์ด้วยโค้ดจริงว่าเกิดอะไรขึ้นถ้าลืมจุดนี้ — ในทางปฏิบัติ
คำสั่งที่ใช้บ่อยกว่า `SET` ตามด้วย `EXPIRE` แยกกันสองคำสั่งคือ **`SET key value EX seconds`** ที่ทำทั้งสอง
อย่างในคำสั่งเดียว (atomic ด้วย — ไม่มีช่องที่ key จะถูกสร้างแล้ว "ยังไม่มี TTL" อยู่ชั่วขณะระหว่างสองคำสั่ง):

```rust
// เทียบเท่ากับ SET greeting "..." EX 30 ในคำสั่งเดียว -- ปลอดภัยกว่า SET แล้ว EXPIRE แยกกัน
let _: () = conn.set_ex("greeting", "สวัสดี Redis", 30).await?;
```

#### Hash และ List: เมื่อไรควรใช้แทน String ธรรมดา

หัวข้อ 83.3 จะ serialize object ทั้งก้อนเป็น JSON string เดียวเก็บใน Redis string — วิธีนี้ง่ายที่สุดและ
เพียงพอสำหรับ cache-aside ทั่วไป แต่มีสถานการณ์ที่ **Hash** เหมาะกว่า: เมื่อต้องการ**อ่าน/แก้แค่บาง field**
โดยไม่ต้องดึงทั้ง object มา deserialize/serialize ใหม่ทุกครั้ง

```bash
$ redis-cli -p 6390 hset book:1:meta title "Rust in Production" author "Various Authors"
(integer) 2
$ redis-cli -p 6390 hgetall book:1:meta
title
Rust in Production
author
Various Authors
```

เทียบกับการเก็บเป็น JSON string เดียว (`book:1` เป็น `'{"title":"...","author":"..."}'`): ถ้าต้องการแก้แค่
`author` ด้วย JSON string ต้อง **`GET` ทั้งก้อน → deserialize → แก้ field → serialize ใหม่ → `SET` ทับทั้ง
ก้อน** (สี่ขั้นตอน และมีช่องให้เกิด race condition แบบเดียวกับหัวข้อ 83.7 ถ้าไม่ atomic) ในขณะที่ Hash ทำได้
ด้วยคำสั่งเดียว **`HSET book:1:meta author "New Author"`** — แก้ field เดียว ไม่แตะ field อื่นเลย และเป็น
atomic operation เดียวในตัวเอง — ข้อแลก: Hash **ไม่มี TTL ต่อ field** (TTL ตั้งได้แค่ระดับ key ทั้ง Hash เท่า
นั้น ไม่ใช่ราย field) และ query ที่ซับซ้อนกว่า "อ่าน/แก้ field เดียว" (เช่น "หา book ทุกเล่มที่ author ขึ้นต้น
ด้วย S") ทำไม่ได้เลยกับ Hash — ต้องพึ่ง PostgreSQL หรือ Set/Sorted Set เสริมแบบหัวข้อ 83.9 แทน

**List** (`LPUSH`/`RPUSH`/`LRANGE`) เหมาะกับข้อมูลที่มี**ลำดับตามเวลาที่ใส่เข้าไป** เช่น recent activity log:

```bash
$ redis-cli -p 6390 rpush recent_searches "rust book" "async programming" "redis cache"
(integer) 3
$ redis-cli -p 6390 lrange recent_searches 0 -1
rust book
async programming
redis cache
```

`RPUSH` ต่อท้าย list (เหมือน `Vec::push` ของ Rust ที่ Part 8 สอน) `LRANGE 0 -1` อ่านทั้ง list ตั้งแต่ตัวแรก
(index `0`) ถึงตัวสุดท้าย (index `-1` — Redis ใช้ index ลบเพื่อนับจากท้าย เหมือน Python มากกว่า Rust) — List
เหมาะกับ "N รายการล่าสุด" (เก็บแค่ N ตัวท้ายด้วย `LTRIM` ตัดของเก่าออกอัตโนมัติ) มากกว่าการเก็บ log ทั้งหมด
ไม่จำกัดจำนวน ซึ่งจะกลับไปเจอปัญหาเดียวกับกับดักข้อ 4 (memory โตไม่มีที่สิ้นสุด) ถ้าไม่ระวัง

หัวข้อถัดไปจะเอาพื้นฐานนี้ไปใช้กับปัญหาจริง: **cache-aside pattern** สำหรับ query ฐานข้อมูล

### 83.3 Cache-Aside Pattern เจาะลึก: Cache การ Query PostgreSQL จริง

#### แนวคิด: เช็คก่อนเสมอ Miss ค่อยไปฐานข้อมูล

**Cache-aside** (เรียกอีกชื่อว่า lazy loading) คือรูปแบบการใช้ cache ที่พบบ่อยที่สุดในระบบจริง — ตรรกะตรงไป
ตรงมามาก:

1. Request เข้ามาขอข้อมูล (เช่น หนังสือ id 1)
2. **เช็ค Redis ก่อนเสมอ** ด้วย key ที่แทนข้อมูลนั้น (เช่น `book:1`)
3. **ถ้าเจอ (cache hit)** — คืนค่าจาก Redis ทันที **ไม่แตะฐานข้อมูลเลย**
4. **ถ้าไม่เจอ (cache miss)** — query PostgreSQL จริง แล้ว **เขียนผลลัพธ์กลับเข้า Redis พร้อม TTL** ก่อนคืนค่า
   กลับไปให้ผู้เรียก (เพื่อให้ request ถัดไปที่ขอข้อมูลตัวเดียวกันเจอ cache hit)

สังเกตชื่อ "cache-**aside**": cache ไม่ใช่ตัวกลางที่ทุก write ต้องผ่านมันเสมอ (ต่างจากรูปแบบอื่นอย่าง
write-through) — แอปพลิเคชันเป็นคนตัดสินใจเองทุกครั้งว่าจะเช็ค cache ก่อนไหม จะเขียนกลับเข้า cache ตอนไหน — cache
"อยู่ข้าง ๆ" (aside) การทำงานปกติ ไม่ใช่ชั้นที่บังคับทุก operation ต้องผ่าน

#### Implement จริง: Cache Book Lookup จาก Part 70

ใช้ตาราง `books` เดียวกับที่ Part 70 สอนสร้างไว้ (`id`, `title`, `author`, `status`) — สร้าง `AppState` ที่มี
ทั้ง `PgPool` (Part 70) และ `redis::Client` (หัวข้อ 83.2) พร้อมกัน:

```rust
use redis::AsyncCommands;
use serde::{Deserialize, Serialize};
use sqlx::postgres::PgPoolOptions;
use std::time::Instant;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
struct Book {
    id: i64,
    title: String,
    author: String,
    status: String,
}

async fn get_book_cache_aside(
    redis: &redis::Client,
    pg: &sqlx::PgPool,
    book_id: i64,
) -> anyhow::Result<(Book, &'static str)> {
    let cache_key = format!("book:{book_id}");
    let mut conn = redis.get_multiplexed_async_connection().await?;

    // 1) เช็ค Redis ก่อนเสมอ
    let cached: Option<String> = conn.get(&cache_key).await?;
    if let Some(json) = cached {
        let book: Book = serde_json::from_str(&json)?;
        return Ok((book, "HIT"));
    }

    // 2) cache miss -> ไป query Postgres จริง (pattern เดียวกับ Part 70 หัวข้อ 70.5)
    let book: Book = sqlx::query_as("SELECT id, title, author, status FROM books WHERE id = $1")
        .bind(book_id)
        .fetch_one(pg)
        .await?;

    // 3) populate cache กลับเข้า Redis พร้อม TTL 60 วินาที
    let json = serde_json::to_string(&book)?;
    conn.set_ex::<_, _, ()>(&cache_key, json, 60).await?;

    Ok((book, "MISS"))
}
```

**อธิบายการออกแบบทีละจุด**:

- **`cache_key = format!("book:{book_id}")`** — ธรรมเนียมการตั้งชื่อ key ใน Redis ที่ใช้กันแพร่หลายคือ
  `{entity_type}:{id}` คั่นด้วย `:` (Redis ไม่มี "namespace" หรือ "table" แบบ SQL — ทุก key อยู่ใน "flat
  namespace" เดียวกันทั้งหมด การตั้งชื่อที่มีโครงสร้างชัดเจนแบบนี้ช่วยให้ debug ง่าย เช่นใช้
  `redis-cli --scan --pattern "book:*"` ดู key ทั้งหมดของ entity ประเภทนี้ได้)
- **serialize เป็น JSON string ก่อนเก็บ** — Redis string เก็บได้แค่ bytes/text ดิบ ๆ ไม่รู้จัก struct ของ Rust
  เลย ต้อง `serde_json::to_string`/`from_str` แปลงไปกลับเอง (เหมือนกับที่ Part 57 สอนเรื่อง serialization
  พื้นฐาน เพียงแต่ปลายทางเป็น Redis ไม่ใช่ HTTP response)
- **`set_ex` พร้อม TTL 60 วินาที** — เลือกอายุ cache ที่ "ยอมรับความเก่าได้กี่วินาที" ตามลักษณะข้อมูล (ข้อมูล
  หนังสือเปลี่ยนไม่บ่อย 60 วินาทีเป็นค่าที่สมเหตุสมผลสำหรับตัวอย่างนี้ — ระบบจริงอาจตั้งเป็นนาทีหรือชั่วโมงก็ได้
  ขึ้นกับว่าข้อมูลนั้นเปลี่ยนบ่อยแค่ไหน)
- **คืนค่า `&'static str` บอก HIT/MISS กลับมาด้วย** — ไม่ใช่ส่วนของ pattern โดยตรง แต่มีประโยชน์มากสำหรับ
  debug/measurement (และหัวข้อ 83.11 จะเอาค่านี้ไปทำเป็น HTTP header `X-Cache` จริง)

#### วัด Latency จริง: Cache Hit เร็วกว่า Miss กี่เท่า

รันจริงกับ Redis + PostgreSQL ที่ตั้งไว้ทดสอบ (ลบ cache key ก่อนเริ่มเพื่อบังคับให้รอบแรกเป็น miss จริง แล้ววัด
เวลาด้วย `std::time::Instant`):

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let redis = redis::Client::open("redis://127.0.0.1:6390/")?;
    let pg = PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://postgres:PASSWORD@127.0.0.1:5432/rust_course_demo")
        .await?;

    let mut conn = redis.get_multiplexed_async_connection().await?;
    let _: () = conn.del("book:1").await?; // บังคับให้รอบแรกเป็น cache miss จริง

    println!("=== รอบที่ 1: cache miss (ต้อง query Postgres) ===");
    let start = Instant::now();
    let (book, source) = get_book_cache_aside(&redis, &pg, 1).await?;
    let elapsed = start.elapsed();
    println!("ผลลัพธ์: {book:?}");
    println!("source: {source}, เวลาที่ใช้: {elapsed:?}");

    println!("\n=== รอบที่ 2-6: cache hit (ควรไม่แตะ Postgres เลย) ===");
    let mut hit_durations = Vec::new();
    for i in 2..=6 {
        let start = Instant::now();
        let (_book, source) = get_book_cache_aside(&redis, &pg, 1).await?;
        let elapsed = start.elapsed();
        println!("รอบที่ {i}: source={source}, เวลาที่ใช้: {elapsed:?}");
        hit_durations.push(elapsed);
    }
    Ok(())
}
```

ผลลัพธ์จริงจากการรัน (`cargo run` จริง ไม่ใช่ค่าที่แต่งขึ้น — Redis รันบน port 6390, PostgreSQL รันบน
127.0.0.1:5432 บนเครื่องทดสอบเดียวกัน):

```text
=== รอบที่ 1: cache miss (ต้อง query Postgres) ===
ผลลัพธ์: Book { id: 1, title: "Rust in Production", author: "Various Authors", status: "available" }
source: MISS, เวลาที่ใช้: 2.717962ms

=== รอบที่ 2-6: cache hit (ควรไม่แตะ Postgres เลย) ===
รอบที่ 2: source=HIT, เวลาที่ใช้: 808.738µs
รอบที่ 3: source=HIT, เวลาที่ใช้: 1.119893ms
รอบที่ 4: source=HIT, เวลาที่ใช้: 755.145µs
รอบที่ 5: source=HIT, เวลาที่ใช้: 796.108µs
รอบที่ 6: source=HIT, เวลาที่ใช้: 611.243µs

เวลาเฉลี่ยตอน cache HIT (5 ครั้ง): 818.225µs
เวลาตอน cache MISS (query Postgres จริง): 2.717962ms
cache hit เร็วกว่า miss ประมาณ 3.3x
```

**อธิบายตัวเลขอย่างตรงไปตรงมา**: cache hit เร็วกว่า miss ประมาณ 3.3 เท่าในการรันครั้งนี้ — ตัวเลขนี้**รวม
overhead ของการเปิด connection ใหม่ทุกครั้ง**ทั้งสองฝั่ง (เพราะฟังก์ชัน `get_book_cache_aside` เรียก
`get_multiplexed_async_connection()` ใหม่ทุกครั้งที่ถูกเรียก ยังไม่ได้ใช้ pool ที่หัวข้อ 83.10 จะสอน) ถ้าใช้
connection pool ที่เปิดไว้ล่วงหน้า (reuse connection) ส่วนต่างที่ Redis เร็วกว่าจะยิ่งเห็นชัดขึ้นไปอีก เพราะ
overhead ของการเปิด connection ใหม่ (ที่เท่ากันทั้งสองฝั่งในการทดสอบนี้) จะถูกตัดออกไป — และในระบบจริงที่
query PostgreSQL ซับซ้อนกว่า `SELECT ... WHERE id = $1` มาก (เช่นมี `JOIN` หลายตาราง, aggregate, หรือฐานข้อมูล
มีข้อมูลหลักล้านแถว) ส่วนต่างนี้จะยิ่งมากขึ้นไปอีกหลายเท่า — **ตัวเลข 3.3x จากการทดสอบนี้คือตัวเลขขั้นต่ำแบบ
ระมัดระวัง (conservative) ไม่ใช่ตัวเลขที่ถูกเลือกมาให้ดูดี**

พิสูจน์ TTL ด้วย `redis-cli` จริง — เห็นว่า key ที่ set ไว้เมื่อกี้ยังมี TTL เหลืออยู่ตามที่ตั้งไว้:

```text
TTL ที่เหลือของ key book:1 ใน Redis ตอนนี้: 60 วินาที
```

### 83.4 Cache Invalidation: ปัญหาที่ยากที่สุดของ Caching (อธิบายอย่างตรงไปตรงมา)

มีคำกล่าวติดตลกในวงการวิศวกรรมซอฟต์แวร์ที่ว่า **"There are only two hard things in Computer Science: cache
invalidation and naming things"** (Phil Karlton) — คำกล่าวนี้ล้อเลียนความยากของปัญหานี้ แต่ก็สะท้อนความจริงที่
ต้องเข้าใจอย่างตรงไปตรงมา: **cache-aside pattern ในหัวข้อ 83.3 แก้ปัญหาเรื่อง performance ได้ แต่เปิดปัญหาใหม่
ขึ้นมาทันที — ข้อมูลใน cache กับข้อมูลจริงใน PostgreSQL อาจไม่ตรงกัน (stale)**

มีสองแนวทางหลักในการจัดการปัญหานี้ ไม่มีแนวทางไหน "ถูกเสมอ" — ขึ้นกับว่าระบบของคุณ **ยอมรับความเก่าได้แค่ไหน**

#### แนวทางที่ 1: TTL-Only (ยอมรับความเก่าในระดับหนึ่ง)

ปล่อยให้ cache หมดอายุตาม TTL ที่ตั้งไว้เท่านั้น (เช่น 60 วินาทีตามหัวข้อ 83.3) — ไม่ทำอะไรเพิ่มตอนมีการ
`UPDATE` ข้อมูลใน PostgreSQL เลย **ข้อดี**: ง่ายที่สุด ไม่ต้องเขียน invalidation logic เพิ่มที่จุด write ทุกจุด
ในระบบ **ข้อเสีย**: มี**หน้าต่างเวลา** (window) ที่ผู้ใช้อาจเห็นข้อมูลเก่าได้จริง ตั้งแต่ตอนที่ข้อมูลจริง
เปลี่ยนจนถึงตอนที่ TTL หมดอายุ

#### แนวทางที่ 2: Invalidate-on-Write (ลบ/อัปเดต Cache ทันทีที่มีการเขียน)

ทุกครั้งที่มีการ `UPDATE`/`DELETE` ข้อมูลใน PostgreSQL ให้ **ลบ (หรืออัปเดต) cache key ที่เกี่ยวข้องทันที**
ในธุรกรรมเดียวกันหรือทันทีหลังจากนั้น — request ถัดไปที่มาขอข้อมูลตัวเดียวกันจะเจอ cache miss (เพราะถูกลบไป)
แล้วไป query ข้อมูลใหม่ล่าสุดจาก PostgreSQL มา populate cache ใหม่โดยอัตโนมัติ (ใช้ตรรกะเดิมจากหัวข้อ 83.3
ไม่ต้องเขียนอะไรเพิ่ม) **ข้อดี**: ไม่มีหน้าต่างเวลาที่เห็นข้อมูลเก่าเลย (หรือมีน้อยมากในระดับ millisecond ที่
ใช้ในการลบ+query ใหม่) **ข้อเสีย**: ต้องเขียน invalidation logic ทุกจุดที่มีการเขียนข้อมูล — ถ้าลืมจุดใดจุดหนึ่ง
(เช่นมี endpoint ที่ 2 แก้ข้อมูลเดียวกันแต่ทีมลืม invalidate) จะเกิด stale read แบบเดียวกับ TTL-only แต่ครั้งนี้
**ไม่มี TTL มาช่วยจำกัดความเสียหายเลย** (ถ้าไม่มี TTL คู่ไว้ด้วย ข้อมูลเก่าอาจอยู่ตลอดไปจนกว่าจะมี write ครั้ง
ถัดไปที่บังเอิญ invalidate ถูกจุด) — ด้วยเหตุนี้ **ระบบจริงส่วนใหญ่ใช้ทั้งสองแนวทางร่วมกัน**: invalidate-on-write
เป็นหลักเพื่อความสดใหม่ (freshness) แต่ยังคง TTL ไว้เป็น safety net เผื่อกรณีที่ invalidation logic พลาดไป
จุดใดจุดหนึ่ง

#### พิสูจน์ Stale Read ด้วยโค้ดจริง แล้วแก้ด้วย Invalidate-on-Write

มาพิสูจน์ปัญหานี้ให้เห็นจริง ไม่ใช่แค่พูดในทางทฤษฎี — สถานการณ์: มี cache ของหนังสือ id 1 อยู่แล้ว (status
`available`) จากนั้น admin อัปเดต status เป็น `borrowed` ตรงใน PostgreSQL โดยตรง (เหมือน endpoint อื่นในระบบ
ที่แก้ข้อมูลแต่ "ลืม" invalidate cache):

```rust
// ==================== พิสูจน์ปัญหา stale read ====================
println!("cache ของ book:1 ตอนนี้ยังมี status เดิม (จาก query ตอนแรก)");
let cached_before: String = conn.get("book:1").await?;
let cached_book_before: Book = serde_json::from_str(&cached_before)?;
println!("cache ปัจจุบัน: {cached_book_before:?}");

println!("\nadmin แก้ status ของหนังสือเล่มนี้ใน Postgres ตรง ๆ");
sqlx::query("UPDATE books SET status = 'borrowed' WHERE id = 1")
    .execute(&pg)
    .await?;
println!("อัปเดต Postgres สำเร็จ: status ใหม่ = 'borrowed'");

println!("\nถ้าใช้ TTL-only (ไม่ invalidate cache ตอน write):");
let (book_stale, source_stale) = get_book_cache_aside(&redis, &pg, 1).await?;
println!("source={source_stale}, status ที่ได้ = '{}'", book_stale.status);
```

ผลลัพธ์จริง — **พิสูจน์ stale read ที่เกิดขึ้นจริง**:

```text
=== ส่วนที่ 2: TTL-only ปล่อยให้อ่านค่าเก่าได้จริง (stale read) ===
cache ของ book:1 ตอนนี้ยังมี status เดิม (จาก query ตอนแรก): เช็คว่า cache ว่าอะไร
cache ปัจจุบัน: Book { id: 1, title: "Rust in Production", author: "Various Authors", status: "available" }

ตอนนี้ admin แก้ status ของหนังสือเล่มนี้ใน Postgres ตรง ๆ (เหมือน UPDATE ที่ endpoint อื่นทำ)
อัปเดต Postgres สำเร็จ: status ใหม่ = 'borrowed'

ถ้าใช้ TTL-only (ไม่ invalidate cache ตอน write) request ถัดไปจะยังอ่านค่าเก่าจาก cache:
source=HIT, status ที่ได้ = 'available' <- ผิด! Postgres จริงตอนนี้คือ 'borrowed' แล้ว
(พิสูจน์แล้วว่านี่คือ stale read จริง ไม่ใช่การพูดลอย ๆ)
```

**ยืนยันชัดเจน**: request ที่ได้ `source=HIT` คืนค่า `status: "available"` **ทั้งที่ PostgreSQL จริงตอนนี้คือ
`borrowed` แล้ว** — cache ยังเก็บค่าเก่าไว้จนกว่า TTL (60 วินาทีที่ตั้งไว้) จะหมดอายุ นี่คือ stale read ที่เกิด
ขึ้นจริง ไม่ใช่ทฤษฎี

ตอนนี้แก้ด้วย invalidate-on-write — ลบ cache key ทันทีหลัง `UPDATE` สำเร็จ:

```rust
println!("=== แก้ด้วย invalidate-on-write: DEL key ทันทีหลัง UPDATE ===");
let _: () = conn.del("book:1").await?;
println!("ลบ cache key book:1 ออกจาก Redis ทันทีหลัง UPDATE สำเร็จ");

let (book_fresh, source_fresh) = get_book_cache_aside(&redis, &pg, 1).await?;
println!("source={source_fresh}, status ที่ได้ = '{}'", book_fresh.status);
```

ผลลัพธ์จริง:

```text
=== แก้ด้วย invalidate-on-write: DEL key ทันทีหลัง UPDATE ===
ลบ cache key book:1 ออกจาก Redis ทันทีหลัง UPDATE สำเร็จ
source=MISS (ต้องเป็น MISS เพราะ cache ถูกลบไปแล้ว), status ที่ได้ = 'borrowed' <- ถูกต้องแล้ว
(พิสูจน์แล้วว่า invalidate-on-write ทำให้ request ถัดไปเห็นข้อมูลล่าสุดทันที)
```

หลัง `DEL` ทันที request ถัดไปเจอ `MISS` (ตามคาด เพราะ cache ถูกลบ) แล้วไป query PostgreSQL ใหม่ ได้
`status: "borrowed"` ที่ถูกต้องกลับมา — **นี่คือรูปแบบที่ endpoint `PATCH`/`PUT` ที่แก้ไขข้อมูลควรทำเสมอในระบบ
จริง**: หลัง `UPDATE`/`DELETE` สำเร็จในฐานข้อมูล ให้ `DEL` cache key ที่เกี่ยวข้องทันทีในบรรทัดถัดไป (ไม่จำเป็น
ต้องอยู่ใน database transaction เดียวกัน เพราะ Redis ไม่ได้เป็นส่วนหนึ่งของ ACID transaction ของ PostgreSQL —
แต่ควรทำ "เร็วที่สุดเท่าที่จะทำได้" หลัง commit สำเร็จ เพื่อลดหน้าต่างเวลาที่ stale ให้เหลือน้อยที่สุด)

### 83.5 ปิดล็อกจาก Part 75: Redis-Backed Session Store ข้ามหลาย Instance จริง

ถึงเวลาทำสิ่งที่ Part 75 หัวข้อ 75.8 ทิ้งไว้เป็นแค่โครงร่างเปล่า ๆ ("`// Part 83 จะสอนการตั้งค่า Redis
connection และ error handling เต็มรูปแบบ`") ให้เป็นของจริงที่รันได้และพิสูจน์ได้

#### เลือก Store: `tower-sessions-redis-store`

Part 75 ระบุไว้แล้วว่า store ที่ `tower-sessions` แนะนำสำหรับ production คือ **`tower-sessions-redis-store`**
ซึ่งใช้ client ชื่อ **`fred`** ภายใน (ไม่ใช่ `redis` crate ที่ใช้ในหัวข้อก่อนหน้าของบทนี้ — `fred` เป็น async
Redis client อีกตัวที่ออกแบบมาให้จัดการ connection pool/reconnect ในตัวเองได้ดีมาก จึงถูกเลือกใช้ภายในของ
session store โดยเฉพาะ) ตรวจสอบเวอร์ชันปัจจุบันจริงจาก crates.io ได้ **`tower-sessions-redis-store = "0.16.0"`**

```toml
[dependencies]
tower-sessions = "0.14.0"
tower-sessions-redis-store = "0.16.0"
axum = { version = "0.8.9", features = ["macros"] }
tokio = { version = "1.53.1", features = ["full"] }
time = "0.3.55"
```

**สังเกตเลขเวอร์ชัน `tower-sessions` ที่ต่างจาก Part 75**: Part 75 ใช้ `tower-sessions = "0.15.0"` กับ
`MemoryStore` — บทนี้ระบุ **`0.14.0`** แทน นี่**ไม่ใช่การพิมพ์ผิด** แต่เป็นข้อจำกัดจริงที่ตรวจพบระหว่างทดสอบ
บทนี้ — อธิบายเต็มรูปแบบพร้อม error message จริงไว้ในหัวข้อกับดักข้อ 1 ท้ายบท สรุปสั้น ๆ ตรงนี้ก่อน: ณ วันที่
เขียนบทนี้ `tower-sessions-redis-store` เวอร์ชันล่าสุด (`0.16.0`) ยังผูกกับ `tower-sessions-core = "0.14.0"`
ตายตัว ในขณะที่ `tower-sessions = "0.15.0"` ผูกกับ `tower-sessions-core = "0.15.0"` ตายตัวเช่นกัน (ทั้งสอง
ใช้ `=` exact version ไม่ใช่ `^` range) ทำให้สองแพ็กเกจนี้ **ใช้คู่กันไม่ได้เลยในเวอร์ชันล่าสุดปัจจุบัน** ต้อง
ถอย `tower-sessions` ลงมาที่ `0.14.0` เพื่อให้ตรงกับ `tower-sessions-core` ที่ `tower-sessions-redis-store`
รองรับ — นี่คือตัวอย่างจริงของ "ecosystem ที่เคลื่อนไหวเร็ว บาง crate อัปเดตตามไม่ทัน" ที่ Part 74 ก็เจอปัญหา
คล้ายกันมาแล้วกับ `jsonwebtoken` (หัวข้อ 74.3) — ทางแก้ในทางปฏิบัติคือ **เช็ค compatibility จริงก่อน deploy
เสมอ อย่าอัปเกรด dependency ตัวใดตัวหนึ่งแบบตัวเดียวโดยไม่เช็คตัวที่ผูกกัน**

#### สร้าง `RedisStore` และแทนที่ `MemoryStore`

```rust
use axum::{response::IntoResponse, routing::get, Json, Router};
use serde::Serialize;
use time::Duration;
use tower_sessions::{Expiry, Session, SessionManagerLayer};
use tower_sessions_redis_store::{fred::prelude::*, RedisStore};

const USER_ID_KEY: &str = "user_id";
const INSTANCE_KEY: &str = "created_by_instance";

#[derive(Serialize)]
struct MeResponse {
    user_id: Option<String>,
    created_by_instance: Option<String>,
    served_by_instance: &'static str,
}

async fn login(session: Session) -> impl IntoResponse {
    session.insert(USER_ID_KEY, "user-42").await.unwrap();
    session.insert(INSTANCE_KEY, "A").await.unwrap();
    "logged in via instance A"
}

async fn me(session: Session) -> impl IntoResponse {
    let user_id: Option<String> = session.get(USER_ID_KEY).await.unwrap();
    let created_by: Option<String> = session.get(INSTANCE_KEY).await.unwrap();
    Json(MeResponse { user_id, created_by_instance: created_by, served_by_instance: "A" })
}

#[tokio::main]
async fn main() {
    // เชื่อมต่อ Redis ผ่าน fred (client ที่ tower-sessions-redis-store ใช้ภายใน)
    let config = Config::from_url("redis://127.0.0.1:6390").unwrap();
    let pool = Pool::new(config, None, None, None, 6).unwrap();
    let _ = pool.connect();
    pool.wait_for_connect().await.unwrap();

    // *** จุดเดียวที่เปลี่ยนจากโครงร่างของ Part 75 หัวข้อ 75.8 ***
    // เปลี่ยนจาก MemoryStore::default() เป็น RedisStore::new(pool) -- SessionManagerLayer และ handler
    // ทั้งหมดที่เหลือ "เหมือนเดิมทุกบรรทัด" ตามที่ Part 75 บอกไว้ (SessionStore เป็น trait -- Part 28)
    let session_store = RedisStore::new(pool);
    let session_layer = SessionManagerLayer::new(session_store)
        .with_secure(false) // dev บน http:// เท่านั้น -- ดู Part 75 หัวข้อ 75.4
        .with_same_site(tower_sessions::cookie::SameSite::Lax)
        .with_expiry(Expiry::OnInactivity(Duration::minutes(30)));

    let app = Router::new()
        .route("/login", get(login))
        .route("/me", get(me))
        .layer(session_layer);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3183").await.unwrap();
    println!("instance A listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

**อธิบายส่วนที่ต่างจาก `MemoryStore`**: `Config::from_url(...)` และ `Pool::new(...)` มาจาก `fred` (re-export
ผ่าน `tower_sessions_redis_store::fred::prelude::*` เพื่อการันตีว่าเวอร์ชันของ `fred` ที่ใช้ตรงกับที่
`tower-sessions-redis-store` ต้องการเป๊ะ ไม่ต้องเพิ่ม `fred` เป็น dependency แยกเอง) `pool.connect()` เริ่ม
การเชื่อมต่อแบบ background (ไม่ block) และ `pool.wait_for_connect().await` รอจนกว่าการเชื่อมต่อครั้งแรกสำเร็จ
ก่อนเริ่มรับ request — ถ้าข้าม `wait_for_connect()` ไป request แรก ๆ อาจล้มเหลวเพราะ pool ยังเชื่อมต่อไม่เสร็จ

**จุดที่สำคัญที่สุดของหัวข้อนี้**: `RedisStore::new(pool)` ถูกส่งเข้า `SessionManagerLayer::new(...)` **แบบ
เดียวกันเป๊ะ**กับที่ `MemoryStore::default()` ถูกส่งเข้าไปใน Part 75 — และ handler (`login`, `me`) **ไม่ต้อง
แก้แม้แต่บรรทัดเดียว** เพราะ `Session` extractor ที่ handler ใช้เป็น abstraction ที่ไม่รู้จักรายละเอียดของ
store เบื้องหลังเลย (`SessionStore` เป็น trait — ตรงตามที่ Part 75 บอกไว้ล่วงหน้า และตรงตามหลักการ trait
object ที่ Part 28 สอน)

#### พิสูจน์ Multi-Instance จริง: สอง Process อ่าน Session ของกันและกันได้

นี่คือการพิสูจน์ที่เป็นเป้าหมายหลักของหัวข้อนี้ — รันสอง server instance (คนละ process, คนละ port) ที่ทั้งคู่
ชี้ไปที่ Redis ตัวเดียวกัน จำลอง instance A/B หลัง load balancer ตามภาพในหัวข้อ 75.8:

Instance A ฟังที่ `127.0.0.1:3183` (โค้ดข้างบน) และ Instance B ฟังที่ `127.0.0.1:3184` (โค้ดเดียวกันทุก
ประการ เปลี่ยนแค่ port และค่าที่ `login`/`me` ใส่ไว้เพื่อให้แยกแยะได้ว่า instance ไหนสร้าง/ตอบ session) —
รันทั้งสอง process พร้อมกัน แล้วทดสอบด้วย `curl` จริง:

```bash
$ curl -sS -i -c /tmp/mi_cookies.txt http://127.0.0.1:3183/login
```
```text
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
set-cookie: id=DAeVvVa2GRU8NMj1Ocae_Q; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
content-length: 24
date: Sun, 27 Sep 2026 02:02:56 GMT

logged in via instance A
```

login สำเร็จที่ **instance A** (port 3183) ได้ session cookie กลับมา ทดสอบเรียก `/me` ที่ instance A เดิม
ก่อนเพื่อยืนยันว่า session ใช้งานได้ปกติ:

```bash
$ curl -sS -i -b /tmp/mi_cookies.txt http://127.0.0.1:3183/me
```
```text
HTTP/1.1 200 OK
content-type: application/json
content-length: 72
date: Sun, 27 Sep 2026 02:02:56 GMT

{"user_id":"user-42","created_by_instance":"A","served_by_instance":"A"}
```

ได้ผลตามคาด — ทีนี้มาถึงจุดสำคัญที่สุด: **เอา cookie เดียวกันเป๊ะไปเรียก instance B (คนละ process, คนละ port
3184) โดยไม่ผ่าน instance A เลยแม้แต่นิดเดียว**:

```bash
$ curl -sS -i -b /tmp/mi_cookies.txt http://127.0.0.1:3184/me
```
```text
HTTP/1.1 200 OK
content-type: application/json
content-length: 72
date: Sun, 27 Sep 2026 02:02:56 GMT

{"user_id":"user-42","created_by_instance":"A","served_by_instance":"B"}
```

**ผลลัพธ์นี้คือคำตอบของทั้งหัวข้อ**: `"created_by_instance":"A"` (session ถูกสร้างจาก instance A จริง) แต่
`"served_by_instance":"B"` (request นี้ถูกตอบโดย instance B จริง — คนละ process, คนละ port, ไม่มีการแชร์
หน่วยความจำกันเลย) — instance B **อ่าน session ที่ instance A สร้างไว้ได้สำเร็จ** เพราะทั้งสอง instance
query ไปที่ Redis ตัวเดียวกัน (port 6390) เสมอ — **นี่คือพฤติกรรมที่ Part 75 หัวข้อ 75.8 อธิบายไว้ว่า
`MemoryStore` ทำไม่ได้เลย** (request ที่ไปตก instance ที่ไม่รู้จัก session จะได้ `401` เพราะ `HashMap` ของ
แต่ละ instance แยกกันสิ้นเชิง) แต่ตอนนี้พิสูจน์แล้วว่า **`RedisStore` แก้ปัญหานี้ได้จริง ไม่ใช่แค่ในทางทฤษฎี**

พิสูจน์เพิ่มเติมด้วยการดูข้อมูลจริงที่ถูกเก็บไว้ใน Redis (`redis-cli` ตรง ๆ):

```bash
$ redis-cli -p 6390 keys "*"
```
```text
DAeVvVa2GRU8NMj1Ocae_Q
```

**สังเกตว่า key ใน Redis คือ session ID ตรง ๆ (ไม่มี prefix อย่าง `session:` นำหน้า)** — นี่คือรายละเอียด
implementation ของ `tower-sessions-redis-store` เอง (เก็บ session ID เป็น key ตรง ๆ ไม่เติม namespace ให้)
ถ้าระบบของคุณมี key ประเภทอื่นปนอยู่ใน Redis instance เดียวกัน (เช่น `book:*` จากหัวข้อ 83.3) ควรพิจารณาแยก
Redis database index (Redis รองรับหลาย database แบบ `SELECT 0`-`SELECT 15` โดย default) หรือแยก Redis
instance ไปเลยสำหรับ session โดยเฉพาะ เพื่อไม่ให้ key ชนกันหรือปนกันจนสับสน

ค่าที่เก็บจริงเป็น **binary format** ไม่ใช่ JSON แบบที่หัวข้อ 83.3 ใช้ (`tower-sessions-redis-store` เลือกใช้
**MessagePack** ผ่าน crate `rmp-serde` ภายใน เพื่อความกระชับกว่า JSON):

```text
$ redis-cli -p 6390 --no-raw get "DAeVvVa2GRU8NMj1Ocae_Q"
"\x93\xc4\x10\xfd\x9e\xc69\xf5\xc84<\x15\x19\xb6V\xbd\x95\a\x0c\x82\xa7user_id\xa7user-42\xb3created_by_instance\xa1A\x99..."
```

แม้จะเป็น binary แต่ยังพอมองเห็น string `user_id`, `user-42`, `created_by_instance`, `A` ปนอยู่ในเนื้อ bytes
(MessagePack encode string เป็น readable bytes ปนกับ length-prefix byte — ไม่เหมือน JSON ที่อ่านได้ทั้งหมด
แต่ก็ไม่ใช่ binary ทึบสนิทแบบ compressed/encrypted data)

### 83.6 ปิดล็อกจาก Part 78 (ตอนที่ 1): Idempotency-Key Store ด้วย `SET NX EX` แบบ Atomic จริง

#### ทวนปัญหาที่ Part 78 ทิ้งไว้

Part 78 หัวข้อ 78.7 implement idempotency-key store ด้วย `Arc<Mutex<HashMap<String, IdempotencyRecord>>>` และ
บอกไว้ตรง ๆ ว่ามีข้อจำกัดสองข้อที่ต้องแก้ก่อนขึ้น production: **(1) ไม่มี TTL** (ข้อมูลอยู่ในหน่วยความจำตลอดไป
จนกว่า process จะ restart) และ **(2) ไม่ถูกแชร์ข้าม instance** (ถ้า deploy หลาย instance request สองครั้งที่
มี `Idempotency-Key` เดียวกันอาจไปตกที่ instance คนละตัวที่ไม่รู้จัก key ของกันและกันเลย — idempotency ใช้ไม่
ได้ผลจริงในระบบที่มีหลาย instance)

Redis แก้ทั้งสองปัญหาพร้อมกัน: **TTL แบบ built-in** และเป็น **store กลาง**ที่ทุก instance เข้าถึงร่วมกันได้ —
เหมือนกับที่หัวข้อ 83.5 แก้ปัญหา session ไปแล้ว

#### `SET key value NX EX ttl`: Primitive ที่ทำให้ "Claim" เป็น Atomic

หัวใจของการ implement idempotency-key ให้ถูกต้องคือคำสั่งเดียวนี้: **`SET key value NX EX ttl`** — สาม flag
ที่รวมกันในคำสั่งเดียว:

- **`NX`** ("only set if **N**ot e**X**ists") — `SET` จะสำเร็จ**ก็ต่อเมื่อ key นี้ยังไม่มีอยู่**เท่านั้น ถ้า
  key มีอยู่แล้ว คำสั่งนี้จะ**ไม่ทำอะไรเลย**และคืนค่า `nil` (ไม่ error แต่ก็ไม่ overwrite)
- **`EX ttl`** — ตั้งเวลาหมดอายุพร้อมกันในคำสั่งเดียว (เหมือนหัวข้อ 83.2)
- **atomicity**: Redis รันคำสั่งเดียวแบบ single-threaded (หัวข้อ 83.1 อธิบายไว้แล้ว) ทำให้ `SET ... NX EX ...`
  ทั้งหมดเป็น**การกระทำเดียวที่แบ่งแยกไม่ได้** — ไม่มีทางที่สอง request จะเจอว่า "key ยังไม่มี" พร้อมกันทั้งคู่
  แล้ว `SET` สำเร็จทั้งคู่ (ต่างจากการเขียน "เช็คก่อนด้วย `EXISTS`/`GET` แล้วค่อย `SET` แยกคำสั่ง" ที่**ไม่
  atomic** — จะพิสูจน์ผลของความผิดพลาดนี้ด้วยโค้ดจริงในหัวข้อกับดักข้อ 2)

ผลลัพธ์ของ `SET ... NX` ที่ crate `redis` คืนกลับมาคือ `Option<String>` — `Some("OK")` แปลว่า claim สำเร็จ
(เป็นคนแรก) และ `None` (nil) แปลว่ามีคน claim ไปแล้ว:

```rust
use redis::AsyncCommands;

/// พยายาม "claim" idempotency key ด้วย SET key value NX EX ttl แบบ atomic ตัวเดียว
/// คืน true = เป็นคนแรกที่ claim สำเร็จ (ควรประมวลผลจริง), false = มีคนอื่น claim ไปแล้ว (ต้อง replay)
async fn try_claim_idempotency_key(
    conn: &mut redis::aio::MultiplexedConnection,
    key: &str,
    ttl_seconds: u64,
) -> redis::RedisResult<bool> {
    let result: Option<String> = redis::cmd("SET")
        .arg(key)
        .arg("claimed")
        .arg("NX")
        .arg("EX")
        .arg(ttl_seconds)
        .query_async(conn)
        .await?;
    Ok(result.is_some())
}
```

**สังเกตว่าใช้ `redis::cmd("SET")` ตรง ๆ แทน method สำเร็จรูป** — trait `AsyncCommands` มี method
`.set_ex()`/`.set_nx()` แยกกัน แต่ ณ เวอร์ชันของ crate ที่ใช้ในบทนี้**ไม่มี method เดียวที่รวม `NX` กับ `EX`
พร้อมกันในตัว** จึงต้องประกอบคำสั่ง raw ด้วย `redis::cmd(...)` เอง (การอ่าน API ของ client library ทุกครั้ง
ก่อนใช้งานจริงสำคัญมาก อย่าสมมติว่า combination ที่ต้องการมี method สำเร็จรูปให้เสมอ)

#### พิสูจน์ Atomicity จริง: ยิง 20 Concurrent Request พร้อม Key เดียวกัน

ทดสอบพื้นฐานก่อน — claim ครั้งแรกสำเร็จ ครั้งที่สองด้วย key เดิมล้มเหลว (ตามคาด):

```text
=== ทดสอบ SET NX EX แบบ sequential (ไม่มี race) ===
claim ครั้งที่ 1: true (คาดหวัง true -- เป็นคนแรก)
claim ครั้งที่ 2 (key เดิม): false (คาดหวัง false -- ถูก claim ไปแล้ว, ต้อง replay response เดิม)
TTL ที่เหลือของ key: 300 วินาที
```

ทีนี้พิสูจน์สิ่งที่สำคัญกว่ามาก — **atomicity ภายใต้ความ concurrent จริง** ไม่ใช่แค่ sequential — ยิง 20 task
พร้อมกันจริง (`tokio::task::JoinSet`) ที่ทุกตัวพยายาม claim idempotency key **ตัวเดียวกันเป๊ะ**:

```rust
use tokio::task::JoinSet;

let race_key = "idempotency:booking:race-key-002";
let mut set = JoinSet::new();
for i in 0..20 {
    let client = client.clone();
    let key = race_key.to_string();
    set.spawn(async move {
        let mut conn = client.get_multiplexed_async_connection().await.unwrap();
        let claimed = try_claim_idempotency_key(&mut conn, &key, 300).await.unwrap();
        (i, claimed)
    });
}

let mut winners = Vec::new();
while let Some(res) = set.join_next().await {
    let (i, claimed) = res.unwrap();
    if claimed { winners.push(i); }
}
```

ผลลัพธ์จริงจากการรัน:

```text
=== พิสูจน์ atomicity: ยิง 20 concurrent request พร้อมกันด้วย idempotency key เดียวกัน ===
จำนวน request ที่ claim สำเร็จ (ควรประมวลผลจริง): 1
request ที่ claim สำเร็จคือ: [13]
จำนวน request ที่ claim ไม่สำเร็จ (ต้อง replay response เดิม): 19

ยืนยัน: จาก 20 concurrent request ที่ใช้ idempotency key เดียวกัน มีแค่ 1 request เท่านั้นที่ claim สำเร็จ
(atomic จริง ไม่มี race condition)
```

**จาก 20 task ที่ยิงพร้อมกันจริง (ไม่ใช่ทีละตัว) มีแค่ 1 ตัวเท่านั้นที่ claim สำเร็จ** (ในการรันนี้คือ task
ที่ 13 — ตัวไหนชนะขึ้นกับ timing ของ scheduler แต่**จะมีผู้ชนะแค่หนึ่งเดียวเสมอ**ไม่ว่าจะรันกี่ครั้งก็ตาม) อีก
19 ตัวที่เหลือควร**ไม่สร้าง resource ใหม่** แต่ไป replay ผลลัพธ์ของตัวที่ชนะกลับไปให้ผู้เรียกแทน — นี่คือ
พฤติกรรมที่ Part 78 ต้องการแต่ `HashMap` ธรรมดา (แม้ห่อด้วย `Mutex`) ทำได้ผ่านฝั่งเดียว **แต่ทำไม่ได้ข้าม
instance** — `SET NX EX` ของ Redis ทำได้ทั้งสองอย่างพร้อมกัน (atomic ในตัวเอง + เป็น store กลาง)

#### Handler เต็มรูปแบบ: Idempotency-Key บน `POST /bookings` ด้วย Redis

ประกอบทุกอย่างเป็น handler เต็มรูปแบบ ต่อยอดจากโครงสร้างเดียวกับ Part 78 หัวข้อ 78.7 (เก็บทั้ง hash ของ body
เพื่อตรวจจับการใช้ key ซ้ำผิดที่ และเก็บ response เต็มเพื่อ replay) เพียงแค่เปลี่ยน storage จาก
`Arc<Mutex<HashMap<...>>>` เป็น Redis:

```rust
async fn create_booking(
    State(state): State<AppState>,
    headers: HeaderMap,
    body: axum::body::Bytes,
) -> Response {
    let idem_key = headers.get("Idempotency-Key").and_then(|v| v.to_str().ok());
    let body_hash = simple_hash(&body); // FNV-1a เดียวกับ Part 78 หัวข้อ 78.7

    let payload: CreateBookingRequest = match serde_json::from_slice(&body) {
        Ok(p) => p,
        Err(_) => return bad_request_response(),
    };

    let mut conn = state.redis.get_multiplexed_async_connection().await.unwrap();

    if let Some(key) = idem_key {
        let claim_key = format!("idempotency:claim:{key}");
        let record_key = format!("idempotency:record:{key}");

        // ขั้นที่ 1: claim ด้วย SET NX EX -- atomic ระดับ Redis เดียว (TTL 24 ชั่วโมง)
        let claimed: Option<String> = redis::cmd("SET")
            .arg(&claim_key).arg(body_hash.to_string()).arg("NX").arg("EX").arg(86400)
            .query_async(&mut conn).await.unwrap_or(None);

        if claimed.is_none() {
            // มีคน claim ไปแล้ว -- เช็คว่า hash ตรงกันไหม (ตรวจจับ key ซ้ำกับ body ต่างกัน แบบ Part 78 หัวข้อ 78.7)
            let stored_hash: Option<String> = conn.get(&claim_key).await.unwrap_or(None);
            if stored_hash.as_deref() != Some(body_hash.to_string().as_str()) {
                return idempotency_conflict_response(); // 422
            }
            // รอ record ผลลัพธ์เดิม (retry เร็วมากอาจมาก่อน record ถูกเขียนโดยตัวที่ชนะ)
            for _ in 0..20 {
                if let Some(record_json) = conn.get::<_, Option<String>>(&record_key).await.unwrap_or(None) {
                    return replay_response(record_json);
                }
                tokio::time::sleep(std::time::Duration::from_millis(20)).await;
            }
        }
    }

    // สร้าง booking ใหม่จริง (path นี้วิ่งเฉพาะตอน claim สำเร็จ)
    let id: i64 = redis::cmd("INCR").arg("booking:next_id").query_async(&mut conn).await.unwrap_or(1);
    let response_body = json!({
        "data": { "id": id, "event_name": payload.event_name, "status": "confirmed" },
        "meta": { "request_id": format!("req-booking-{id}") }
    });

    if let Some(key) = idem_key {
        let record_key = format!("idempotency:record:{key}");
        let _: Result<(), _> = redis::cmd("SET").arg(&record_key).arg(response_body.to_string())
            .arg("EX").arg(86400).query_async(&mut conn).await;
    }

    (StatusCode::CREATED, Json(response_body)).into_response()
}
```

**จุดที่ต่างจาก Part 78 ที่ควรสังเกต**: แยก **claim key** (`idempotency:claim:{key}` — เก็บแค่ hash ของ body
สำหรับตรวจจับ conflict) ออกจาก **record key** (`idempotency:record:{key}` — เก็บ response เต็มสำหรับ replay)
เป็นสอง key คนละตัว **ทำไมไม่รวมเป็น key เดียว**: เพราะ `SET NX` ต้องเขียนค่า claim ให้เสร็จ**ทันที**เพื่อกัน
request อื่นที่มาพร้อมกัน แต่ response เต็มยัง**ไม่พร้อม**จนกว่าจะสร้าง booking จริงเสร็จ (ต้องรอ `INCR`
และ logic สร้าง booking ก่อน) — ถ้าพยายามยัดทุกอย่างลง key เดียวตั้งแต่ `SET NX` ครั้งแรก จะต้องรู้ response
ล่วงหน้าก่อนประมวลผลจริงซึ่งเป็นไปไม่ได้ตามลำดับเหตุการณ์จริง การแยกสอง key ทำให้ request ที่แพ้ (claim ไม่
สำเร็จ) **poll** รอ record key สั้น ๆ (สูงสุด 20 ครั้ง ครั้งละ 20ms = 400ms) จนกว่า request ที่ชนะจะเขียน
record เสร็จ — เป็นการแลก complexity เล็กน้อยเพื่อให้ atomicity ของการ claim ทำงานถูกต้อง 100%

### 83.7 ปิดล็อกจาก Part 78 (ตอนที่ 2): Rate Limiter แบบ Atomic ด้วย `INCR` และ Lua Script

#### ทำไม Atomicity สำคัญกับ Rate Limiting: พิสูจน์ Race Condition จริง

Part 78 หัวข้อ 78.9 สร้าง token bucket rate limiter แบบ in-memory ด้วย `Arc<Mutex<...>>` และบอกไว้ว่า
production ต้องมี **Redis-backed counter ที่แชร์ quota ข้ามทุก instance** — แต่มีรายละเอียดสำคัญที่ต้องเข้าใจ
ก่อนเขียนโค้ดสักบรรทัด: **การนับ request แบบ "อ่านค่าปัจจุบัน แล้วบวกหนึ่ง แล้วเขียนกลับ" ด้วยคำสั่งแยกกันสอง
คำสั่ง (`GET` แล้ว `SET`) ไม่ atomic — และจะทำให้นับผิด (under-count) จริงเมื่อมี concurrent request**

มาพิสูจน์ด้วยโค้ดจริงก่อนไปดูทางแก้ — เขียนสองเวอร์ชันเทียบกัน:

```rust
/// เวอร์ชัน "ผิด" -- อ่านค่าปัจจุบัน แล้วค่อยบวกหนึ่งแล้วเขียนกลับ (GET แล้ว SET แยกกันคนละคำสั่ง)
async fn increment_non_atomic(
    conn: &mut redis::aio::MultiplexedConnection,
    key: &str,
) -> redis::RedisResult<i64> {
    let current: Option<i64> = conn.get(key).await?;
    let current = current.unwrap_or(0);
    tokio::time::sleep(std::time::Duration::from_micros(500)).await; // จำลอง network round-trip
    let new_value = current + 1;
    let _: () = conn.set(key, new_value).await?;
    Ok(new_value)
}

/// เวอร์ชัน "ถูก" -- ใช้ INCR ตัวเดียว (atomic ระดับ Redis เอง)
async fn increment_atomic(
    conn: &mut redis::aio::MultiplexedConnection,
    key: &str,
) -> redis::RedisResult<i64> {
    conn.incr(key, 1).await
}
```

ยิงทั้งสองเวอร์ชันด้วย **50 concurrent task พร้อมกัน** เพิ่ม counter ตัวเดียวกัน แล้วดูค่าสุดท้ายที่ได้ (ถ้า
นับถูกทุกครั้ง ค่าสุดท้ายต้องเป็น 50 เป๊ะ):

```text
=== ยิง 50 concurrent request พร้อมกัน เพิ่ม counter เดียวกัน ===

--- เวอร์ชันไม่ atomic (GET แล้วค่อย SET แยกคนละคำสั่ง) ---
ค่า counter สุดท้ายที่ได้จริง: 2 (ควรจะเป็น 50 ถ้านับถูกทุกครั้ง)
!!! เกิด UNDER-COUNT จริง: หายไป 48 requests ที่ไม่ถูกนับ เพราะ 2 request อ่านค่าเดิมพร้อมกัน
    แล้วเขียนทับกัน (lost update) -- นี่คือ race condition ที่เกิดขึ้นจริง ไม่ใช่ทฤษฎี

--- เวอร์ชัน atomic (INCR คำสั่งเดียว) ---
ค่า counter สุดท้ายที่ได้จริง: 50 (ต้องเป็น 50 เป๊ะเสมอ)
ยืนยัน: INCR นับถูกครบ 50 ทุกครั้ง ไม่มี lost update เลย
```

**ผลลัพธ์รุนแรงกว่าที่คาด**: จาก 50 request พร้อมกัน เวอร์ชันไม่ atomic นับได้แค่ **2** (หายไปถึง 48 request!)
ในขณะที่เวอร์ชัน atomic ด้วย `INCR` นับได้ครบ **50 เป๊ะ** ทุกครั้งที่รัน (ทดสอบซ้ำ 3 รอบ ได้ผลเดียวกันทุกรอบ
แม้ตัวเลข under-count ของเวอร์ชันไม่ atomic จะต่างกันไปบ้างในแต่ละรอบ — 2, 2, 1 — ตามจังหวะการ schedule ของ
OS/runtime แต่**ไม่มีรอบไหนได้ 50 เลย**)

**อธิบายว่าทำไมถึงเกิด lost update**: สมมติ request A และ request B ยิงพร้อมกัน ทั้งคู่เรียก `GET key` ได้ค่า
`5` เหมือนกัน (เพราะยังไม่มีใครเขียนอะไรใหม่ ณ ขณะนั้น) จากนั้นทั้งคู่คำนวณ `5 + 1 = 6` แยกกันในหน่วยความจำของ
ตัวเอง แล้วทั้งคู่ `SET key 6` — **ผลคือ counter เป็น `6` ทั้งที่มี 2 request ผ่านเข้ามา ควรจะเป็น `7`** (ค่า
ที่ควรได้คือ `5+1+1=7` แต่กลับได้ `6` เพราะ request ตัวที่สองที่ `SET` "เขียนทับ" ผลลัพธ์ของตัวแรกไปเลย ไม่ได้
บวกต่อจากมัน) — นี่คือปัญหาคลาสสิกที่เรียกว่า **lost update** ในทฤษฎี concurrency (Part 39 อธิบายปัญหาแบบ
เดียวกันนี้ในบริบทของ shared memory ระหว่าง thread — ที่นี่ปัญหาเดียวกันเกิดข้าม network round-trip ระหว่าง
client กับ Redis แทน) **`INCR` แก้ปัญหานี้เพราะมันคือคำสั่งเดียวที่ทำ "อ่าน+บวก+เขียน" ทั้งหมดในตัวมันเองบน
ฝั่ง Redis** ไม่มีช่องว่างให้ request อื่นแทรกเข้ามาระหว่างขั้นตอนเหล่านี้เลย

#### Fixed-Window Rate Limiter จริงด้วย `INCR` + `EXPIRE` ผ่าน Lua Script (`EVAL`)

การนับ request อย่างเดียวยังไม่ใช่ rate limiter เต็มรูปแบบ — ต้องมี **window** (ช่วงเวลาที่นับ) ด้วย ปกติทำ
โดย `INCR` แล้วถ้าเป็นครั้งแรกของ window (`INCR` คืนค่า `1`) ก็ `EXPIRE` ให้ window นั้นหมดอายุอัตโนมัติ —
แต่ถ้าเขียน `INCR` แล้ว `EXPIRE` เป็น**สองคำสั่งแยกกัน** จะมีช่องโหว่เล็ก ๆ ที่ atomicity ยังไม่สมบูรณ์: ถ้า
`INCR` สำเร็จแล้ว แต่ process ล่มหรือ network หลุดก่อนเรียก `EXPIRE` — key นั้นจะ **ไม่มี TTL เลย** (นับต่อ
ไปเรื่อย ๆ ไม่มีวัน reset window) ทางแก้คือรวมทั้งสองคำสั่งเป็น **atomic operation เดียว** ด้วย **Lua script**
ที่ Redis รันผ่านคำสั่ง `EVAL`:

```rust
let script = redis::Script::new(
    r#"
    local current = redis.call('INCR', KEYS[1])
    if current == 1 then
        redis.call('EXPIRE', KEYS[1], ARGV[1])
    end
    return current
    "#,
);

let limit = 5;
let window_seconds = 10;
for i in 1..=7 {
    let count: i64 = script
        .key(window_key)
        .arg(window_seconds)
        .invoke_async(&mut conn)
        .await?;
    let allowed = count <= limit;
    println!("request {i}: count={count}/{limit} -> {}",
        if allowed { "ALLOWED (200)" } else { "REJECTED (429)" });
}
```

**ทำไม Lua script ถึง atomic**: Redis รัน**ทั้ง script ในครั้งเดียวแบบไม่มีการขัดจังหวะ** — ไม่มี request อื่น
ใดสามารถแทรกคำสั่งของตัวเองเข้ามาระหว่างที่ script นี้กำลังรัน (แม้ script จะมีหลายคำสั่งข้างในก็ตาม) เพราะ
Redis รันคำสั่งแบบ single-threaded อยู่แล้ว (หัวข้อ 83.1) — Lua script จึงเป็นเครื่องมือมาตรฐานสำหรับ "รวม
หลายคำสั่งของ Redis ให้เป็น atomic operation เดียว" เมื่อ built-in command เดี่ยว ๆ (เช่น `INCR` อย่างเดียว)
ไม่พอสำหรับตรรกะที่ต้องการ (ที่นี่คือ "INCR แล้วเช็คว่าเป็นครั้งแรกหรือไม่ ถ้าใช่ค่อย EXPIRE")

ผลลัพธ์จริงจากการรัน (limit = 5 ครั้งต่อ 10 วินาที ยิง 7 request ติดกัน):

```text
=== Fixed-window rate limiter จริงด้วย INCR + EXPIRE (atomic ผ่าน Lua script/EVAL) ===
request 1: count=1/5 -> ALLOWED (200)
request 2: count=2/5 -> ALLOWED (200)
request 3: count=3/5 -> ALLOWED (200)
request 4: count=4/5 -> ALLOWED (200)
request 5: count=5/5 -> ALLOWED (200)
request 6: count=6/5 -> REJECTED (429)
request 7: count=7/5 -> REJECTED (429)

TTL ของ window ปัจจุบัน: 10 วินาที (window จะ reset นับใหม่จาก 0 อัตโนมัติ)
```

ตรงตามที่ออกแบบไว้เป๊ะ: request ที่ 1-5 ผ่าน (`ALLOWED`) request ที่ 6-7 ถูกปฏิเสธ (`REJECTED`) — ตรงกับ
HTTP contract ที่ Part 78 หัวข้อ 78.9 ออกแบบไว้แล้ว (`429 Too Many Requests` พร้อม `Retry-After`,
`X-RateLimit-*` headers) เพียงแต่ตอนนี้ตัวนับที่อยู่หลัง contract นั้น**เป็น Redis จริงที่ atomic และแชร์ข้าม
instance ได้** ไม่ใช่ token bucket แบบ in-memory ต่อ instance อีกต่อไป

**หมายเหตุเรื่องความแม่นยำของ fixed window**: window แบบนี้มีข้อจำกัดที่ควรรู้ (ไม่ใช่ปัญหาของ atomicity แต่
เป็นข้อจำกัดของ**อัลกอริทึม**) — ถ้า request 5 ครั้งมาตอนท้ายวินาทีที่ 9 ของ window แรก แล้วอีก 5 ครั้งมาตอน
ต้นวินาทีที่ 11 (window ที่สองเริ่มแล้ว) ผู้ใช้จะยิงได้ 10 ครั้งในช่วงเวลาสั้น ๆ ราว 2 วินาที ทั้งที่ limit
คือ 5 ครั้งต่อ 10 วินาที — นี่คือพฤติกรรมที่ยอมรับได้ในระบบส่วนใหญ่ (ความเรียบง่ายของ fixed window มักคุ้มกว่า
ความแม่นยำที่เพิ่มขึ้นเล็กน้อย) แต่ถ้าต้องการความแม่นยำสูงกว่า มีอัลกอริทึม **sliding window log** ที่ใช้
Sorted Set (`ZADD` timestamp ของแต่ละ request แล้ว `ZREMRANGEBYSCORE` ตัดของเก่าที่พ้น window ออกทุกครั้ง)
ซึ่งเป็นโจทย์ของแบบฝึกหัดข้อ 3 ท้ายบทนี้

### 83.8 ปิดล็อกจาก Part 74: Refresh Token Revocation ด้วย `jti` ใน Redis

#### ทวนปัญหาที่ Part 74 Sketch ไว้

Part 74 หัวข้อ 74.11 อธิบายไว้อย่างตรงไปตรงมาว่า refresh token ที่เป็น JWT ล้วน ๆ **revoke ไม่ได้เลย** จนกว่า
จะหมดอายุเอง (7 วันในตัวอย่างของ Part 74) — และ sketch ทางแก้ไว้เป็นสามขั้นตอน: (1) เพิ่ม claim `jti` (JWT
ID) เป็นค่าสุ่มไม่ซ้ำต่อ token หนึ่งใบ (2) เก็บคู่ `(jti, revoked)` ไว้ใน Redis (3) เช็ค revocation ทุกครั้งที่
`/refresh` ถูกเรียก **หลังจาก** ตรวจสอบลายเซ็น/`exp`/`typ` ผ่านแล้ว — บทนี้ implement ทั้งสามขั้นตอนนี้เต็ม
รูปแบบด้วยของจริง

#### เพิ่ม Claim `jti` ให้ `Claims` ของ Part 74

```rust
use chrono::Utc;
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

// Claims struct เดียวกับ Part 74 ทุกประการ เพิ่มแค่ jti (JWT ID มาตรฐาน RFC 7519)
#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,
    username: String,
    role: String,
    iat: usize,
    exp: usize,
    typ: String,
    jti: String, // ค่าสุ่มไม่ซ้ำต่อ token หนึ่งใบ -- ใช้เป็น key ใน Redis สำหรับ revocation
}

const SECRET: &[u8] = b"super-secret-signing-key-for-demo-only";

fn issue_refresh_token(user_id: &str, username: &str, role: &str, ttl_seconds: usize) -> (String, String) {
    let now = Utc::now().timestamp() as usize;
    let jti = Uuid::new_v4().to_string(); // uuid crate -- ค่าสุ่ม 128 bit ไม่ซ้ำ
    let claims = Claims {
        sub: user_id.to_string(), username: username.to_string(), role: role.to_string(),
        iat: now, exp: now + ttl_seconds, typ: "refresh".to_string(), jti: jti.clone(),
    };
    let header = Header::new(Algorithm::HS256);
    let key = EncodingKey::from_secret(SECRET);
    let token = encode(&header, &claims, &key).unwrap();
    (token, jti)
}
```

#### เช็ค Revocation ด้วย `EXISTS` และ Revoke ด้วย `SET EX` ที่ TTL ตรงกับอายุที่เหลือ

```rust
#[derive(Debug)]
enum RefreshError {
    InvalidSignatureOrExpired(String),
    WrongTokenType,
    Revoked,
}

/// จุดที่ Part 74 หัวข้อ 74.11 sketch ไว้ -- ตรวจลายเซ็น/exp/typ ตามปกติก่อน แล้วค่อยเช็ค revocation
async fn try_refresh(
    conn: &mut redis::aio::MultiplexedConnection,
    refresh_token: &str,
) -> Result<Claims, RefreshError> {
    let decoding_key = DecodingKey::from_secret(SECRET);
    let validation = Validation::new(Algorithm::HS256);

    let token_data = decode::<Claims>(refresh_token, &decoding_key, &validation)
        .map_err(|e| RefreshError::InvalidSignatureOrExpired(e.to_string()))?;

    if token_data.claims.typ != "refresh" {
        return Err(RefreshError::WrongTokenType);
    }

    // เช็ค revocation กับ Redis -- EXISTS เช็คว่า jti นี้อยู่ใน revocation set ไหม
    let revoke_key = format!("revoked_jti:{}", token_data.claims.jti);
    let is_revoked: bool = conn.exists(&revoke_key).await.map_err(|_| RefreshError::Revoked)?;
    if is_revoked {
        return Err(RefreshError::Revoked);
    }

    Ok(token_data.claims)
}

/// endpoint /logout -- mark jti ว่า revoked ใน Redis พร้อม TTL เท่ากับอายุที่เหลือของ refresh token เดิม
async fn revoke_refresh_token(
    conn: &mut redis::aio::MultiplexedConnection,
    claims: &Claims,
) -> redis::RedisResult<()> {
    let now = Utc::now().timestamp() as usize;
    let remaining_ttl = claims.exp.saturating_sub(now).max(1) as u64;
    let revoke_key = format!("revoked_jti:{}", claims.jti);
    conn.set_ex::<_, _, ()>(&revoke_key, "revoked", remaining_ttl).await
}
```

**อธิบายเหตุผลที่ TTL ของ revocation record ตั้งเท่ากับ `remaining_ttl` ไม่ใช่ค่าคงที่**: นี่คือรายละเอียดที่
สำคัญมากและมักถูกมองข้าม — ถ้าตั้ง TTL ของ revocation record เป็นค่าคงที่ยาว ๆ (เช่น 30 วัน) ไม่สนใจว่า token
เดิมเหลืออายุอยู่กี่วัน จะทำให้ **revocation set ใน Redis โตขึ้นเรื่อย ๆ ไม่มีที่สิ้นสุด** สะสม record ของ
token ที่หมดอายุไปแล้วนานแล้วไว้โดยไม่จำเป็น (เพราะ token ที่หมดอายุแล้วจะถูกปฏิเสธโดย `decode()`/`exp` check
อยู่แล้วโดยไม่ต้องพึ่ง revocation list เลย) — การตั้ง TTL ให้**พอดีกับอายุที่เหลือของ token ตัวนั้นเป๊ะ** ทำให้
Redis ลบ record นี้ทิ้งให้อัตโนมัติทันทีที่ token หมดอายุไปเอง ไม่ต้องมี cleanup job แยก และ revocation set
จะไม่โตเกินกว่า "จำนวน token ที่ยัง valid อยู่และถูก revoke ไปแล้ว" ซึ่งมีขนาดจำกัดเสมอ

#### พิสูจน์ครบวงจร: Refresh สำเร็จ → Logout → Refresh ล้มเหลว

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let client = redis::Client::open("redis://127.0.0.1:6390/")?;
    let mut conn = client.get_multiplexed_async_connection().await?;

    let (refresh_token, jti) = issue_refresh_token("42", "nan", "member", 60 * 60 * 24 * 7);

    // ครั้งที่ 1: refresh ก่อน logout -- ควรสำเร็จ
    let result1 = try_refresh(&mut conn, &refresh_token).await;

    // logout: revoke jti นี้
    let claims = decode::<Claims>(&refresh_token, &DecodingKey::from_secret(SECRET), &Validation::new(Algorithm::HS256))?.claims;
    revoke_refresh_token(&mut conn, &claims).await?;

    // ครั้งที่ 2: refresh token เดิมเป๊ะ หลัง logout -- ควรล้มเหลว
    let result2 = try_refresh(&mut conn, &refresh_token).await;

    Ok(())
}
```

ผลลัพธ์จริงจากการรันเต็มรูปแบบ:

```text
=== ออก refresh token ใหม่ให้ user 'nan' ===
jti ของ refresh token นี้: 2611eb77-2989-4018-9676-826e3e52e46b
refresh token (ตัดให้สั้น): eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsInVzZXJ...

=== ครั้งที่ 1: ใช้ refresh token นี้แลก access token ใหม่ (ก่อน logout) ===
สำเร็จ: refresh ผ่าน สำหรับ user_id=42, jti=2611eb77-2989-4018-9676-826e3e52e46b

=== ผู้ใช้กด logout -- เรียก revoke_refresh_token() ===
mark jti=2611eb77-2989-4018-9676-826e3e52e46b เป็น revoked ใน Redis สำเร็จ (key: revoked_jti:2611eb77-2989-4018-9676-826e3e52e46b)
TTL ของ revocation record: ~604800 วินาที (ตรงกับอายุที่เหลือของ refresh token เดิม ~7 วัน)

=== ครั้งที่ 2: พยายามใช้ refresh token เดิม (ตัวเดียวกันเป๊ะ) แลก access token อีกครั้ง หลัง logout ===
ล้มเหลวตามที่ออกแบบไว้: Revoked <- refresh token ที่ revoke ไปแล้วใช้ต่อไม่ได้จริง

=== พิสูจน์ว่า refresh token ใบอื่นของ user เดียวกัน (jti ต่างกัน) ไม่ถูกกระทบ ===
ออก refresh token ใบใหม่ jti=9b5b796b-4953-486b-bb18-de0a65454bec (คนละใบกับที่ revoke ไปแล้ว)
สำเร็จ: refresh token ใบใหม่ (jti=9b5b796b-4953-486b-bb18-de0a65454bec) ยังใช้งานได้ปกติ ไม่ถูกกระทบจากการ revoke ใบเดิม
```

**ผลลัพธ์ยืนยันครบทุกจุดที่ Part 74 ต้องการ**: (1) refresh token ใช้งานได้ปกติก่อน logout (2) หลัง logout
refresh token **ตัวเดียวกันเป๊ะ** ใช้ต่อไม่ได้เลย — ได้ `Revoked` error ตรงตามที่ออกแบบไว้ ไม่ใช่แค่รอ `exp`
หมดอายุเอง (3) **refresh token ใบอื่นของ user คนเดียวกัน (`jti` ต่างกัน) ไม่ถูกกระทบ** — ตรงตามสถานการณ์ที่
3 ที่ Part 74 หัวข้อ 74.1 พูดถึงไว้ (ตรวจพบว่า token ใบหนึ่งรั่วไหล ต้องยกเลิกใบนั้นทันทีโดยไม่กระทบใบอื่นของ
user เดียวกันที่ยังปลอดภัยอยู่) — การใช้ `jti` เป็นหน่วยของการ revoke (ไม่ใช่ revoke ทั้ง user) คือสิ่งที่ทำให้
ความละเอียด (granularity) นี้เป็นไปได้

### 83.9 Data Structures อื่น ๆ ที่มีประโยชน์จริง: Sorted Set และ Set

#### Sorted Set: Leaderboard "งานที่ถูกจองมากที่สุดสัปดาห์นี้"

**Sorted Set** (คำสั่งหลัก `ZADD`/`ZINCRBY`/`ZRANGE`) คือ Set ที่แต่ละสมาชิกมี **score** (ตัวเลข floating
point) กำกับ และ Redis **เรียงลำดับสมาชิกตาม score ให้อัตโนมัติตลอดเวลา** — ทุกครั้งที่ score เปลี่ยน ลำดับจะ
ถูกจัดใหม่ทันทีโดยไม่ต้องสั่ง `ORDER BY` แยก นี่คือโครงสร้างข้อมูลที่เหมาะกับ **leaderboard** พอดี:

```rust
use redis::AsyncCommands;

let lb_key = "leaderboard:events:this_week";

// ทุกครั้งที่มีการจองสำเร็จ 1 ครั้ง แค่ ZINCRBY member นั้นด้วย 1
let bookings = [
    ("Rust Conf Bangkok 2026", 42), ("PyCon Bangkok 2026", 18),
    ("Golang Meetup", 7), ("JS Fest", 30), ("Data Engineering Day", 55),
];
for (event, count) in bookings {
    let _: () = conn.zincr(lb_key, event, count).await?;
}

// ZREVRANGE ดึง top-N เรียงจากมากไปน้อยพร้อม score ในคำสั่งเดียว
let top3: Vec<(String, f64)> = conn.zrevrange_withscores(lb_key, 0, 2).await?;
```

ผลลัพธ์จริง:

```text
=== Sorted Set (ZADD/ZINCRBY/ZRANGE): Leaderboard งานที่ถูกจองมากที่สุดสัปดาห์นี้ ===

Top 3 งานที่ถูกจองมากที่สุดสัปดาห์นี้:
  #1: Data Engineering Day -- 55 bookings
  #2: Rust Conf Bangkok 2026 -- 42 bookings
  #3: JS Fest -- 30 bookings

มีการจอง 'Golang Meetup' เพิ่มอีก 3 -> score ใหม่ทันที (atomic): 10
อันดับปัจจุบันของ 'Golang Meetup' (0-indexed จากบนสุด): Some(4)

leaderboard เต็มหลังอัปเดต:
  #1: Data Engineering Day -- 55
  #2: Rust Conf Bangkok 2026 -- 42
  #3: JS Fest -- 30
  #4: PyCon Bangkok 2026 -- 18
  #5: Golang Meetup -- 10
```

**เทียบกับการทำแบบเดียวกันด้วย PostgreSQL ล้วน ๆ**: ถ้าเก็บจำนวนการจองไว้ใน column ของตาราง `events` ตรง ๆ
ทุกครั้งที่ต้องแสดง leaderboard ต้องรัน `SELECT event_name, booking_count FROM events ORDER BY booking_count
DESC LIMIT 3` ซึ่งก็ทำงานได้ถูกต้องเช่นกัน — **ความต่างไม่ได้อยู่ที่ "ทำไม่ได้" แต่อยู่ที่ต้นทุนของ operation
ที่เกิดบ่อยมาก**: ถ้าหน้า leaderboard นี้ถูกเปิดดูหลายพันครั้งต่อนาที (สถานการณ์จริงของหน้า "งานฮิต" บนเว็บ
ขายตั๋ว) การ `ORDER BY` ผ่าน B-tree index ของ PostgreSQL ทุกครั้งที่มีคนเปิดหน้านั้นยังต้องเสีย disk/cache I/O
ในระดับหนึ่งเสมอ ในขณะที่ Sorted Set ของ Redis **เก็บอยู่ใน RAM ในรูปแบบที่ "เรียงแล้ว" อยู่ตลอดเวลาอยู่แล้ว**
(โครงสร้างข้อมูลภายในเป็น skip list + hash table ผสมกัน) การอ่าน top-N คือการเดินโครงสร้างที่จัดเรียงไว้แล้ว
ไม่ใช่การ sort สดทุกครั้ง — สำหรับ "ข้อมูลที่ถูกอ่านบ่อยกว่าที่ถูกเขียนมาก และต้องการ ranking แบบ real-time"
Sorted Set จึงเหมาะกว่าในเชิง performance ที่ scale สูง และยัง**ประหยัด logic ฝั่งแอปพลิเคชัน**ไปด้วย (ไม่ต้อง
เขียน SQL ที่ซับซ้อนขึ้นถ้าต้องรวม ranking จากหลายมิติ)

#### Set: Tag-Based Filtering ด้วย `SADD`/`SINTER`

**Set** (คำสั่งหลัก `SADD`/`SMEMBERS`/`SINTER`/`SISMEMBER`) คือกลุ่มสมาชิกไม่ซ้ำ ไม่มีลำดับ — เหมาะกับการทำ
**index ของแต่ละ tag** แล้วใช้ **set intersection** หาข้อมูลที่ตรงกับหลายเงื่อนไขพร้อมกัน:

```rust
// index แต่ละ tag เป็น Set ของ book id ที่มี tag นั้น
for (tag, ids) in [("rust", &[1,2,3,4][..]), ("beginner", &[2]), ("available", &[1,2,4])] {
    let key = format!("tag:{tag}");
    for id in ids { let _: () = conn.sadd(&key, id).await?; }
}

// หาหนังสือที่เป็น "rust" และ "available" พร้อมกัน -- SINTER ทำในคำสั่งเดียว
let rust_and_available: Vec<i64> = conn.sinter(("tag:rust", "tag:available")).await?;
```

ผลลัพธ์จริง:

```text
=== Set (SADD/SINTER/SMEMBERS): Tag-based filtering สำหรับหนังสือ ===

หนังสือที่มี tag 'rust': [1, 2, 3, 4]
หนังสือที่เป็นทั้ง 'rust' และ 'available' พร้อมกัน (SINTER): [1, 2, 4]
หนังสือที่เป็น 'rust' และ 'beginner' และ 'available' พร้อมกันทั้งสามเงื่อนไข: [2]

หนังสือ id=3 มี tag 'rust' หรือไม่ (SISMEMBER, O(1)): true
จำนวนหนังสือทั้งหมดที่มี tag 'rust' (SCARD, O(1)): 4
```

**เทียบกับ PostgreSQL**: การหา "หนังสือที่มีทั้ง tag A และ tag B" ด้วย SQL ล้วน ๆ ต้องมีตาราง `book_tags`
(many-to-many) แล้วเขียน query แบบ `SELECT book_id FROM book_tags WHERE tag IN ('rust','available') GROUP BY
book_id HAVING COUNT(DISTINCT tag) = 2` — ทำงานได้ถูกต้อง แต่ซับซ้อนกว่า `SINTER` มาก ทั้งในแง่ syntax และ
ในแง่ execution plan (ต้อง `GROUP BY`/`HAVING` ซึ่งมักช้ากว่า index lookup ตรง ๆ) — `SISMEMBER` และ `SCARD`
ยังเป็น **O(1)** (เวลาคงที่ไม่ขึ้นกับขนาด Set) ต่างจากการเช็ค membership ผ่าน array/list ที่ต้อง scan ทีละตัว
— สำหรับข้อมูลที่ถูกอ่านบ่อย (hot data) และมีรูปแบบการ query แบบ "intersect หลายเงื่อนไข" Set ของ Redis จึง
เป็นตัวเลือกที่คุ้มค่าในการ maintain เป็น index เสริมคู่กับ PostgreSQL (ไม่ใช่แทนที่ — PostgreSQL ยังเป็น
source of truth ของ tag แต่ละเล่มเสมอ Set ใน Redis เป็นแค่ index ที่ build ขึ้นมาเพื่อ query เร็ว และต้อง
sync ให้ตรงกับ PostgreSQL ทุกครั้งที่ tag เปลี่ยน — หลักการเดียวกับ cache invalidation ในหัวข้อ 83.4)

### 83.10 Connection Pooling และ Error Handling: เมื่อ Redis เองล่ม

#### Pooling: `deadpool-redis` เทียบกับ `PgPool` ของ Part 70

Part 70 อธิบายไว้แล้วว่าทำไมต้องมี connection pool แทนการเปิด connection ใหม่ทุกครั้งที่มี request (เปิด TCP
connection + handshake มีต้นทุนที่ไม่ควรจ่ายซ้ำทุก request) — หลักการเดียวกันนี้ใช้กับ Redis เช่นกัน แม้
connection ของ Redis จะเปิดเร็วกว่า PostgreSQL มาก (protocol เรียบง่ายกว่า ไม่มี TLS negotiation ที่ซับซ้อน
ในหลายกรณี) แต่การเปิดใหม่ทุกครั้งก็ยังเป็นต้นทุนที่ไม่จำเป็นเมื่อมี traffic สูง

**`deadpool-redis`** เป็น crate ที่ทำหน้าที่เดียวกับ `PgPoolOptions` ของ `sqlx` แต่สำหรับ Redis — ตรวจสอบ
เวอร์ชันปัจจุบันได้ **`0.23.1`**:

```toml
[dependencies]
deadpool-redis = { version = "0.23.1", features = ["rt_tokio_1"] }
```

```rust
use deadpool_redis::{Config, Runtime};
use redis::AsyncCommands;

let cfg = Config::from_url("redis://127.0.0.1:6390/");
let pool = cfg.create_pool(Some(Runtime::Tokio1))?;
// เก็บ pool ไว้ใน AppState แบบเดียวกับ PgPool ของ Part 70 -- .with_state(pool) ครั้งเดียว
// handler ทุกตัว .get() ยืม connection จาก pool ตัวนี้ ไม่เปิด connection ใหม่ทุก request
let mut conn = pool.get().await?;
let _: () = conn.set("key", "value").await?;
```

ทดสอบ pool จริงด้วย 30 concurrent task ที่ยืม connection จาก pool เดียวกันพร้อมกัน:

```text
=== ยิง 30 concurrent task ยืม connection จาก pool เดียวกันพร้อมกัน ===
ผลลัพธ์ที่ได้ครบ (แปลว่า pool จัดการ connection ให้ 30 task พร้อมกันได้จริงไม่ error เลย):
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29]

pool status: size=8, available=8, waiting=0
```

30 task ยืม connection จาก pool ที่มีขนาดจริงแค่ **8 connection** (ค่า default ที่ผูกกับจำนวน CPU core ของ
เครื่องทดสอบ) พร้อมกันได้สำเร็จทั้งหมดโดยไม่มี error — pool จัดการคิว (queue) ให้ task ที่ต้องรอ connection
ว่างโดยอัตโนมัติ (`status.waiting` กลับมาเป็น `0` เพราะทดสอบเสร็จหมดแล้ว แต่ระหว่างรันจะมี task ที่ต้องรอคิว
ชั่วขณะถ้า 30 task มาถึงเร็วกว่าที่ 8 connection จะประมวลผลได้ทัน) — พฤติกรรมนี้เหมือนกับ `PgPool` ของ Part 70
ทุกประการในเชิงแนวคิด: **pool คือขอบเขตที่จำกัดจำนวน concurrent connection ไปยัง backend ไม่ให้เกินที่
backend รองรับได้ พร้อมคิวจัดการ request ที่มาเกินขนาด pool ให้อัตโนมัติ**

#### Persistence: ทำไมข้อมูลใน Redis "หายได้" และทำไมนั่นคือเรื่องปกติสำหรับบทนี้

ก่อนพูดถึง error handling ต้องเข้าใจข้อเท็จจริงพื้นฐานอีกข้อ: Redis **ไม่การันตี durability แบบเดียวกับ
PostgreSQL** โดย default — เช็คค่า config จริงจาก instance ที่ใช้ทดสอบบทนี้:

```bash
$ redis-cli -p 6390 config get save
save
"3600 1 300 100 60 10000"
$ redis-cli -p 6390 config get appendonly
appendonly
no
```

`save "3600 1 300 100 60 10000"` คือ **RDB snapshot policy** ค่า default — แปลว่า "snapshot ข้อมูลทั้งหมดลง
disk ถ้ามีการเปลี่ยนแปลงอย่างน้อย 1 ครั้งใน 3600 วินาที, หรืออย่างน้อย 100 ครั้งใน 300 วินาที, หรืออย่างน้อย
10000 ครั้งใน 60 วินาที" — **นี่แปลว่าถ้า Redis process ล่มกลางอากาศ (crash, kill -9, ไฟดับ) ข้อมูลที่เปลี่ยน
ไปหลัง snapshot ล่าสุดจะหายไปหมด** อาจหายไปได้มากถึงเป็นนาทีหรือชั่วโมง ขึ้นกับว่า snapshot ล่าสุดเกิดขึ้นเมื่อ
ไร ส่วน `appendonly no` แปลว่า **AOF (Append Only File — log ทุกคำสั่งเขียนแบบเดียวกับ WAL ของ PostgreSQL
ที่ Part 70 พูดถึง) ปิดอยู่** ซึ่งเป็นค่า default ของ Redis หลายการติดตั้ง — เปิด AOF ได้เพื่อลดการสูญเสียข้อมูล
ลงเหลือระดับวินาทีเดียว (หรือน้อยกว่า ขึ้นกับ `appendfsync` policy) แต่แลกกับ write throughput ที่ลดลงและ
disk I/O ที่เพิ่มขึ้น (ขัดกับเหตุผลหลักที่เลือกใช้ Redis ตั้งแต่ต้นในหัวข้อ 83.1)

**นี่คือเหตุผลเชิงลึกที่ทุกตัวอย่างในบทนี้ (session, cache, idempotency key, rate limit counter, revocation
record) เลือกใช้ Redis อย่างเหมาะสม**: ทุกอย่างที่เก็บใน Redis ในบทนี้ **สร้างใหม่ได้เสมอถ้าหายไป** — session
หายไปแค่บังคับให้ user login ใหม่ (ไม่ใช่ข้อมูลที่หายไปตลอดกาล), cache หายไปแค่กลับไป cache-aside จาก
PostgreSQL ใหม่ (หัวข้อ 83.3), idempotency record หายไปในกรณีเลวร้ายที่สุดอาจทำให้ retry ครั้งถัดไปประมวลผล
ซ้ำ (ยอมรับความเสี่ยงนี้ได้เพราะเกิดยาก และ TTL ของ record มักสั้นกว่าเวลาที่ต้อง restart Redis อยู่แล้ว) —
**ไม่มีข้อมูลไหนในบทนี้ที่เป็น "ความจริงเดียว" (source of truth) ของระบบเลย** — PostgreSQL ยังเป็น source of
truth เสมอสำหรับ books, bookings, users ตัวจริง — Redis เป็นแค่ชั้นเสริมที่เร็วกว่า ถ้าตัดสินใจเก็บข้อมูลที่
**ไม่สามารถสร้างใหม่ได้** (เช่น ยอดเงินในบัญชี, ประวัติการทำธุรกรรม) ไว้ใน Redis เพียวๆ โดยไม่มี PostgreSQL
สำรอง นั่นคือการตัดสินใจที่ต้องคิดให้รอบคอบกว่านี้มาก (อาจต้องเปิด AOF พร้อม `appendfsync always` และยอมรับ
performance ที่ลดลง หรือพิจารณาใช้ PostgreSQL ตั้งแต่ต้นแทน)

#### Error Handling: Redis ล่มควรเป็น Hard Failure หรือ Graceful Degradation?

นี่คือคำถามเชิงออกแบบที่สำคัญที่สุดของหัวข้อนี้ และคำตอบไม่ได้ตรงไปตรงมาเสมอไป — ต้องแยกตามว่า **Redis ถูก
ใช้ทำอะไรอยู่** (ตามหลักการที่ Part 66 สอนเรื่อง "แยก error ที่ควรบอก user ตรง ๆ ออกจาก error ที่ควร log แล้ว
หาทางสำรอง"):

| Redis ใช้ทำอะไร | Redis ล่มควรเป็นอะไร | เหตุผล |
|---|---|---|
| **Cache-aside (หัวข้อ 83.3)** | **Graceful degradation** — fallback ไป query PostgreSQL ตรง ๆ | ไม่มี cache ก็ยังทำงานถูกต้องได้ แค่ช้าลง ไม่ใช่ผิดเลย |
| **Session store (หัวข้อ 83.5)** | **Hard failure** (หรือ degrade เป็น "บังคับ login ใหม่") | ไม่มี session store แปลว่าไม่รู้ว่า request มาจากใครเลย — ตอบผิด (ปล่อยให้เข้าโดยไม่ auth) อันตรายกว่าปฏิเสธ |
| **Idempotency-key (หัวข้อ 83.6)** | **Hard failure สำหรับ endpoint นั้น** (503) | ถ้าตรวจ idempotency ไม่ได้ แล้วประมวลผลต่อไปเฉย ๆ อาจสร้าง resource ซ้ำ — เสี่ยงกว่าปฏิเสธชั่วคราว |
| **Rate limiter (หัวข้อ 83.7)** | ขึ้นกับนโยบาย — มักเลือก **fail-open** (ปล่อยผ่านชั่วคราว) มากกว่า fail-closed | rate limit ที่พลาดไปชั่วขณะเสียหายน้อยกว่าการปฏิเสธผู้ใช้ทุกคนเพราะ infrastructure ย่อยล่ม |

**สังเกตว่าไม่มีคำตอบเดียวที่ใช้ได้กับทุกกรณี** — นี่คือเหตุผลที่ตารางนี้สำคัญกว่าการจำกฎตายตัว: ต้องถามเสมอว่า
"ถ้า Redis ไม่ตอบตอนนี้ การ**เดา**ว่าอะไรถูกต้องปลอดภัยกว่ากัน ระหว่าง (ก) ปฏิเสธ request นี้ไปเลย กับ (ข)
ทำงานต่อโดยไม่มีข้อมูลจาก Redis" — สำหรับ cache-aside คำตอบคือ (ข) ชัดเจน (เพราะ PostgreSQL ยังมีข้อมูลจริง
ครบอยู่แล้ว Redis เป็นแค่ shortcut) แต่สำหรับ idempotency-key คำตอบคือ (ก) ชัดเจนพอกัน (เพราะถ้าประมวลผลต่อ
โดยไม่รู้ว่า request นี้เคยถูกประมวลผลไปแล้วหรือยัง อาจสร้างการจองซ้ำที่แก้คืนยากกว่าการให้ user ลองใหม่)

#### Implement Graceful Degradation จริงสำหรับ Cache-Aside

```rust
async fn get_book(State(state): State<AppState>, Path(id): Path<i64>) -> Response {
    let cache_key = format!("book:{id}");
    let mut conn = match state.redis.get_multiplexed_async_connection().await {
        Ok(c) => c,
        Err(e) => {
            // graceful degradation: Redis ล่ม -> ข้าม cache ไป query Postgres ตรง ๆ เลย ไม่ 500
            tracing::warn!("Redis connection failed, degrading to direct DB read: {e}");
            return fetch_from_db_with_header(&state.pg, id, "MISS", "degraded").await;
        }
    };
    // ... ปกติทำงานแบบ cache-aside เหมือนหัวข้อ 83.3
}
```

พิสูจน์ด้วยการ**ปิด Redis จริง** (`redis-cli -p 6390 shutdown nosave`) แล้วเรียก endpoint เดิมอีกครั้ง:

```bash
$ redis-cli -p 6390 shutdown nosave
$ curl -sS -D - http://127.0.0.1:3283/api/v1/books/2
```
```text
HTTP/1.1 200 OK
content-type: application/json
x-cache: MISS
x-cache-degraded: true
content-length: 103
date: Sun, 27 Sep 2026 02:05:04 GMT

{"data":{"author":"Steve Klabnik","id":2,"status":"available","title":"The Rust Programming Language"}}
```

**server ยังตอบ `200 OK` พร้อมข้อมูลที่ถูกต้องแม้ Redis จะปิดไปแล้วจริง ๆ** — header `X-Cache-Degraded: true`
บอกให้ทีม operations รู้ว่า request นี้ทำงานในโหมดสำรอง (มีประโยชน์มากสำหรับ monitoring/alerting — ถ้า header
นี้เริ่มโผล่บ่อยขึ้นเรื่อย ๆ แปลว่า Redis มีปัญหาต้องรีบไปดู แม้ผู้ใช้จะไม่รู้สึกอะไรเลยเพราะแอปยังทำงานถูกต้อง)
log ที่ฝั่ง server ก็ยืนยันตรงกัน (capture จริงจาก `tracing::warn!` ที่ยิงออกมาตอนทดสอบ):

```text
WARN capstone: Redis connection failed, degrading to direct DB read: Connection refused (os error 111)
```

นี่คือตัวอย่างที่เป็นรูปธรรมของหลักการที่ Part 66 สอนไว้: **error ที่เกิดขึ้นจริง (`Connection refused`) ไม่
ควรกลายเป็น error ที่ user เห็นเสมอไป** — ถ้ามีทางสำรองที่สมเหตุสมผล (query PostgreSQL ตรง ๆ) หน้าที่ของ
error handling ที่ดีคือ **log ให้ทีมที่ดูแลระบบรู้ + หาทางสำรองให้ user ไม่รู้สึกอะไร** ไม่ใช่ปล่อยให้ error
ลอยขึ้นไปเป็น `500 Internal Server Error` ทั้งที่มีทางแก้ไขได้ในตัว

**สิ่งที่เกิดขึ้นกับ session store (`fred`) ในสถานการณ์เดียวกัน**: การปิด Redis ระหว่างที่ `capstone` server
กำลังรันอยู่ (server ตัวเดียวกันที่ใช้ `RedisStore` สำหรับ session ในหัวข้อ 83.5/83.11 ด้วย) ทำให้ log
แสดงความพยายาม reconnect ของ `fred` pool ออกมาเองโดยอัตโนมัติ (capture จริงจากตอนทดสอบ):

```text
WARN fred::router::responses: fred-HNgJoD17NU: Broadcasting error Some(Error { details: "Connection closed.", kind: IO }) from 127.0.0.1:6390
WARN fred::router::responses: fred-lz6Vey5uD9: Broadcasting error Some(Error { details: "Connection closed.", kind: IO }) from 127.0.0.1:6390
```

**สังเกตว่านี่คือ log จาก `fred` เอง ไม่ใช่ code ที่เราเขียน** — `fred::Pool` มีตรรกะ reconnect ในตัวเองอยู่
แล้ว (คล้ายกับ `ConnectionManager` ของ crate `redis` ที่เปิดผ่าน feature `connection-manager` ในหัวข้อ 83.2)
เมื่อ Redis กลับมาออนไลน์ pool จะเชื่อมต่อใหม่ให้อัตโนมัติโดยไม่ต้อง restart แอป Rust เลย — แต่ **ระหว่างที่
กำลัง reconnect อยู่ (สักเสี้ยววินาทีถึงไม่กี่วินาทีขึ้นกับ network) session ที่ผ่าน `RedisStore` จะยังใช้งาน
ไม่ได้** เพราะ session เป็นกรณีที่ตารางเปรียบเทียบด้านบนจัดไว้ในกลุ่ม **hard failure** — และนี่คือความต่างที่
สำคัญจาก cache-aside: การมี auto-reconnect ในตัว client ไม่ได้แปลว่า "ทน Redis ล่มได้เสมอ" — มันแค่ทำให้
**กลับมาทำงานได้เร็วที่สุดหลัง Redis กลับมา** แต่ระหว่างที่ล่มอยู่ endpoint ที่ผูกกับ session จะยัง error อยู่
ตามที่ตั้งใจออกแบบไว้ ในขณะที่ endpoint cache-aside ที่ผ่าน graceful degradation ยังตอบ `200 OK` ได้ตลอดเวลา
ไม่ว่า Redis จะล่มหรือกำลัง reconnect อยู่ก็ตาม

### 83.11 Capstone: รวมทุกอย่างเข้า Axum API เดียว

ประกอบทุกหัวข้อของบทนี้เข้าเป็นระบบเดียว ต่อยอดจาก ticket-booking/library API ที่ใช้ตลอดหลักสูตร — สาม
endpoint ที่มี Redis เสริมอยู่คนละจุด: `GET /api/v1/books/{id}` (cache-aside พร้อม `X-Cache` header),
`POST /api/v1/bookings` (idempotency-key แบบ atomic), และ `POST /login` + `GET /me` (session auth ผ่าน
`RedisStore`)

```rust
use axum::{
    extract::{Path, State}, http::{HeaderMap, HeaderValue, StatusCode},
    response::{IntoResponse, Response}, routing::{get, post}, Json, Router,
};
use redis::AsyncCommands;
use serde::{Deserialize, Serialize};
use serde_json::json;
use sqlx::postgres::PgPoolOptions;
use time::Duration as TimeDuration;
use tower_sessions::{Expiry, Session, SessionManagerLayer};
use tower_sessions_redis_store::{fred::prelude::*, RedisStore};

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
struct Book { id: i64, title: String, author: String, status: String }

#[derive(Clone)]
struct AppState { pg: sqlx::PgPool, redis: redis::Client }

// GET /api/v1/books/{id} -- cache-aside + X-Cache header (หัวข้อ 83.3 + 83.10)
async fn get_book(State(state): State<AppState>, Path(id): Path<i64>) -> Response {
    let cache_key = format!("book:{id}");
    let mut conn = match state.redis.get_multiplexed_async_connection().await {
        Ok(c) => c,
        Err(e) => {
            tracing::warn!("Redis connection failed, degrading to direct DB read: {e}");
            return fetch_from_db_with_header(&state.pg, id, "MISS", "degraded").await;
        }
    };
    let cached: Option<String> = conn.get(&cache_key).await.unwrap_or(None);
    if let Some(json_str) = cached {
        if let Ok(book) = serde_json::from_str::<Book>(&json_str) {
            let mut resp = (StatusCode::OK, Json(json!({"data": book}))).into_response();
            resp.headers_mut().insert("X-Cache", HeaderValue::from_static("HIT"));
            return resp;
        }
    }
    let resp = fetch_from_db_with_header(&state.pg, id, "MISS", "normal").await;
    if resp.status() == StatusCode::OK {
        if let Ok(book) = sqlx::query_as::<_, Book>("SELECT id, title, author, status FROM books WHERE id = $1")
            .bind(id).fetch_one(&state.pg).await
        {
            let _ = conn.set_ex::<_, _, ()>(&cache_key, serde_json::to_string(&book).unwrap(), 60).await;
        }
    }
    resp
}

// POST /login, GET /me -- session auth ผ่าน RedisStore (หัวข้อ 83.5)
async fn login(session: Session) -> impl IntoResponse {
    session.insert("user_id", "user-42").await.unwrap();
    StatusCode::OK
}
async fn me(session: Session) -> Result<Json<serde_json::Value>, StatusCode> {
    let user_id: Option<String> = session.get("user_id").await.unwrap();
    user_id.map(|uid| Json(json!({"data": {"user_id": uid}}))).ok_or(StatusCode::UNAUTHORIZED)
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pg = PgPoolOptions::new().max_connections(5)
        .connect("postgres://postgres:PASSWORD@127.0.0.1:5432/rust_course_demo").await?;
    let redis_client = redis::Client::open("redis://127.0.0.1:6390/")?;

    let config = Config::from_url("redis://127.0.0.1:6390")?;
    let pool = Pool::new(config, None, None, None, 6)?;
    let _ = pool.connect();
    pool.wait_for_connect().await?;
    let session_layer = SessionManagerLayer::new(RedisStore::new(pool))
        .with_secure(false)
        .with_same_site(tower_sessions::cookie::SameSite::Lax)
        .with_expiry(Expiry::OnInactivity(TimeDuration::minutes(30)));

    let state = AppState { pg, redis: redis_client };
    let app = Router::new()
        .route("/api/v1/books/{id}", get(get_book))
        .route("/api/v1/bookings", post(create_booking)) // จากหัวข้อ 83.6
        .route("/login", post(login))
        .route("/me", get(me))
        .layer(session_layer)
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3283").await?;
    axum::serve(listener, app).await?;
    Ok(())
}
```

#### ทดสอบเต็มรูปแบบด้วย `curl` จริง

**1) Cache-aside — MISS แล้ว HIT:**

```bash
$ curl -sS -D - http://127.0.0.1:3283/api/v1/books/2
```
```text
HTTP/1.1 200 OK
x-cache: MISS
content-length: 103

{"data":{"author":"Steve Klabnik","id":2,"status":"available","title":"The Rust Programming Language"}}
```
```bash
$ curl -sS -D - http://127.0.0.1:3283/api/v1/books/2
```
```text
HTTP/1.1 200 OK
x-cache: HIT
content-length: 103

{"data":{"author":"Steve Klabnik","id":2,"status":"available","title":"The Rust Programming Language"}}
```

**2) Idempotency-key — claim, replay, conflict:**

```bash
$ curl -sS -D - -X POST http://127.0.0.1:3283/api/v1/bookings \
    -H "Content-Type: application/json" -H "Idempotency-Key: cap-key-001" \
    -d '{"event_name":"Rust Conf Bangkok 2026"}'
```
```text
HTTP/1.1 201 Created
content-length: 114

{"data":{"event_name":"Rust Conf Bangkok 2026","id":1,"status":"confirmed"},"meta":{"request_id":"req-booking-1"}}
```
```bash
$ curl -sS -D - -X POST http://127.0.0.1:3283/api/v1/bookings \
    -H "Content-Type: application/json" -H "Idempotency-Key: cap-key-001" \
    -d '{"event_name":"Rust Conf Bangkok 2026"}'
```
```text
HTTP/1.1 201 Created
idempotent-replayed: true
content-length: 114

{"data":{"event_name":"Rust Conf Bangkok 2026","id":1,"status":"confirmed"},"meta":{"request_id":"req-booking-1"}}
```
```bash
$ curl -sS -D - -X POST http://127.0.0.1:3283/api/v1/bookings \
    -H "Content-Type: application/json" -H "Idempotency-Key: cap-key-001" \
    -d '{"event_name":"Completely Different Event"}'
```
```text
HTTP/1.1 422 Unprocessable Entity

{"error":{"code":"IDEMPOTENCY_KEY_REUSED","message":"Idempotency-Key นี้เคยถูกใช้กับ request body ที่ต่างออกไป"}}
```

**3) Session auth — ก่อน/หลัง login:**

```bash
$ curl -sS -D - http://127.0.0.1:3283/me
```
```text
HTTP/1.1 401 Unauthorized
```
```bash
$ curl -sS -D - -c /tmp/cap_cookies.txt -X POST http://127.0.0.1:3283/login
```
```text
HTTP/1.1 200 OK
set-cookie: id=zCDk4UXWKpSPOa0-iDXlUg; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
```
```bash
$ curl -sS -D - -b /tmp/cap_cookies.txt http://127.0.0.1:3283/me
```
```text
HTTP/1.1 200 OK

{"data":{"user_id":"user-42"}}
```

ทุก request ผ่านครบตามที่ออกแบบไว้ — cache แสดง `MISS`→`HIT` ถูกต้อง, idempotency claim ครั้งแรกสำเร็จ
(`201`), replay คืนผลลัพธ์เดิม (`idempotent-replayed: true`), key ซ้ำกับ body ต่างกันถูกปฏิเสธ (`422`), และ
session auth ทำงานถูกต้องทั้งก่อน/หลัง login — **นี่คือระบบเดียวที่ Redis เข้ามาเสริมสามจุดพร้อมกันโดยไม่
ชนกันเลย** เพราะแต่ละจุดใช้ Redis data structure และ key namespace ของตัวเองแยกกันชัดเจน (`book:*` สำหรับ
cache, `idempotency:*` สำหรับ idempotency, session ID ตรง ๆ สำหรับ session store)

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Version Mismatch ระหว่าง `tower-sessions` และ `tower-sessions-redis-store`

ระหว่างทดสอบบทนี้จริง การ `cargo add tower-sessions@0.15.0 tower-sessions-redis-store@0.16.0` (เวอร์ชัน
ล่าสุดของทั้งคู่ ณ วันที่เขียนบทนี้) แล้ว compile จะได้ error จริงแบบนี้:

```
error[E0277]: the trait bound `RedisStore<...>: SessionStore` is not satisfied
   --> src/bin/session_instance_b.rs:40:25
    |
 40 |     let session_layer = SessionManagerLayer::new(session_store)
    |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `SessionStore` is not implemented for `RedisStore<...>`
    |
note: there are multiple different versions of crate `tower_sessions_core` in the dependency graph
    |
109 | pub trait SessionStore: Debug + Send + Sync + 'static {
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ this is the expected trait
   ::: tower-sessions-core-0.14.0/src/session_store.rs:109:1
    |
109 | pub trait SessionStore: Debug + Send + Sync + 'static {
    | ----------------------------------------------------- this is the found trait
```

**เหตุผล**: `tower-sessions = "0.15.0"` ผูกกับ `tower-sessions-core = "=0.15.0"` (exact pin) ในขณะที่
`tower-sessions-redis-store = "0.16.0"` ผูกกับ `tower-sessions-core = "=0.14.0"` (exact pin เช่นกัน แต่คนละ
เวอร์ชัน) — Cargo ลง `tower-sessions-core` **สองเวอร์ชันพร้อมกัน**ในต้นไม้ dependency (0.14.0 และ 0.15.0) และ
เพราะ `SessionStore` trait ถูก define อยู่ใน `tower-sessions-core` **trait จาก 0.14.0 กับ trait จาก 0.15.0
ถือเป็นคนละ trait กันในสายตาของ compiler แม้จะหน้าตาเหมือนกันทุกตัวอักษร** — `RedisStore` implement trait
ตัวที่มาจาก 0.14.0 แต่ `SessionManagerLayer` (จาก `tower-sessions` 0.15.0) ต้องการ trait ตัวที่มาจาก 0.15.0

**วิธีแก้**: ถอย `tower-sessions` ลงมาที่ `"0.14.0"` (ให้ตรงกับ `tower-sessions-core` ที่
`tower-sessions-redis-store` เวอร์ชันล่าสุดรองรับ) — `cargo build` ผ่านสำเร็จทันทีหลังแก้:

```toml
tower-sessions = "0.14.0"           # ไม่ใช่ 0.15.0 ที่ Part 75 ใช้กับ MemoryStore
tower-sessions-redis-store = "0.16.0"
```

**บทเรียนที่กว้างกว่าปัญหานี้**: เมื่อสอง crate ผูกกับ dependency ร่วมกันด้วย **exact version pin** (`=x.y.z`
ไม่ใช่ `^x.y.z`) การอัปเดตตัวหนึ่งโดยไม่เช็คว่าตัวอื่นตามทันหรือยัง**เสี่ยงชนกันได้เสมอ** — ก่อน deploy ด้วย
เวอร์ชันล่าสุดของทุก dependency ควรรัน `cargo build` ในสภาพแวดล้อมทดสอบก่อนจริง ๆ อย่าสมมติว่า "เวอร์ชันล่าสุด
ของทุกตัวต้องเข้ากันได้เสมอ" — ecosystem ที่เคลื่อนไหวเร็วแบบ Rust/Tokio มักมี crate ที่อัปเดตตามกันไม่ทัน
เป็นระยะ (Part 74 หัวข้อ 74.3 ก็เจอปัญหาคล้ายกันมาแล้วกับ `jsonwebtoken` ที่ต้องเลือก crypto backend เอง)

### 2. ลืมใช้ `NX` ตอน Claim Idempotency Key — Race Condition ที่ทำให้สร้าง Resource ซ้ำ

ถ้าเขียนตรรกะการ claim idempotency key แบบ "เช็คก่อนด้วย `EXISTS`/`GET` แล้วค่อย `SET` แยกคนละคำสั่ง" (ลืมใส่
`NX`) แทนการใช้ `SET key value NX EX ttl` แบบ atomic ในหัวข้อ 83.6 — ทดสอบยิง 10 concurrent request พร้อมกัน
ด้วย idempotency key เดียวกันไปที่โค้ดแบบผิดนี้:

```rust
async fn naive_claim(conn: &mut redis::aio::MultiplexedConnection, key: &str) -> bool {
    let exists: bool = conn.exists(key).await.unwrap();
    if exists { return false; }
    let _: () = conn.set(key, "claimed").await.unwrap(); // ลืม NX!
    true
}
```

ผลลัพธ์จริง:

```text
=== กับดัก 1: ลืมใช้ NX ตอน claim idempotency key -- GET แล้วค่อย SET แยกคนละคำสั่ง ===

จาก 10 concurrent request ที่ใช้ idempotency key เดียวกัน (ไม่มี NX):
request ที่คิดว่าตัวเอง 'เป็นคนแรก' และควรสร้าง resource ใหม่: [2, 5, 6, 7, 4, 9, 1, 3, 8, 0]
!!! เกิดบั๊กจริง: 10 request คิดว่าตัวเองเป็นคนแรกพร้อมกัน -> สร้าง resource ซ้ำ 10 รายการ
    ทั้งที่ idempotency key เดียวกันเป๊ะ -- นี่คือผลของการไม่ atomic
```

**ทั้ง 10 request คิดว่าตัวเองเป็น "คนแรก" พร้อมกันหมด** — เพราะทุกตัวเรียก `EXISTS` (ได้ `false` เพราะยังไม่มี
ใครเขียนอะไรเลย ณ ขณะที่เช็ค) **ก่อน**ที่ใครจะเรียก `SET` เสร็จ — ถ้าเป็น endpoint สร้างการจองจริง นี่แปลว่า
ผู้ใช้คนเดียวที่กด "จอง" ครั้งเดียว (แต่ client เผลอยิง request ซ้ำ เช่นจาก double-click หรือ network retry)
จะได้ **10 การจองที่แยกกัน** ทั้งที่ตั้งใจจะจองแค่ครั้งเดียว — วิธีแก้คือใช้ `SET NX` ตามหัวข้อ 83.6 เสมอ ไม่ใช่
แยก "เช็ค" กับ "เขียน" เป็นสองคำสั่ง ไม่ว่าจะดูเหมือน "เร็วกว่าถ้าเขียนแบบตรงไปตรงมา" แค่ไหนก็ตาม

### 3. `WRONGTYPE Operation against a key holding the wrong kind of value`

Redis ผูก **type ของ value เข้ากับ key ตั้งแต่ตอนสร้าง** (string, hash, list, set, sorted set — ตามตาราง
หัวข้อ 83.1) และจะปฏิเสธคำสั่งที่ผิดชนิดทันที — ถ้าเผลอเรียกคำสั่งของ data structure ผิดกับ key ที่มีอยู่แล้ว
เป็นอีกชนิด:

```rust
let str_key = "pitfall:wrongtype:demo";
conn.set(str_key, "this is a plain string").await?; // สร้างเป็น string

let result: redis::RedisResult<i64> = conn.lpush(str_key, "item").await; // LPUSH คือคำสั่งของ List!
```

error จริงที่ได้:

```
error จริงจาก Redis: "WRONGTYPE": Operation against a key holding the wrong kind of value
```

**สาเหตุที่พบบ่อยในทางปฏิบัติ**: การใช้ naming convention ของ key ที่ไม่รัดกุมพอ (เช่นสอง feature ในทีมต่างคน
ต่างเลือกใช้ prefix `user:42` แต่ feature หนึ่งเก็บเป็น string (ข้อมูล profile แบบ JSON) อีก feature เก็บเป็น
Set (รายการ permission) โดยไม่รู้ตัวว่าชนกัน) — วิธีป้องกัน: **ตั้งชื่อ key ให้บอกทั้ง entity และวัตถุประสงค์
ชัดเจนเสมอ** (เช่น `user:42:profile` กับ `user:42:permissions` แยกกันชัดเจน ไม่ใช่ `user:42` เฉย ๆ ที่สอง
ทีมอาจตีความคนละแบบ) — error นี้เกิดตอน**เรียกคำสั่งจริง**ไม่ใช่ตอน compile เลย (เพราะ Rust type system ไม่รู้
เลยว่า key ไหนควรเป็นชนิดไหนใน Redis — ทุกอย่างเป็น `RedisResult<T>` ทั่วไปที่ error ได้เสมอ) จึงต้องมี
integration test ที่รันกับ Redis จริงเพื่อจับปัญหานี้ ไม่สามารถพึ่ง compiler จับให้ได้เลย

### 4. ลืมตั้ง TTL — Memory เต็มไม่มีวันสิ้นสุด

`SET` เพียว ๆ **ไม่มี TTL โดย default** — ถ้า endpoint ใดสร้าง key ด้วย input ที่ไม่ซ้ำกันเรื่อย ๆ (เช่นใช้
`request_id`, `session_id` แบบสุ่ม, หรือ user input เป็นส่วนหนึ่งของ key) โดยลืมใส่ `EX`/`PX`:

```rust
let no_ttl_key = "pitfall:no_ttl:demo";
conn.set(no_ttl_key, "forgot the TTL").await?;
let ttl: i64 = conn.ttl(no_ttl_key).await?;
```

ผลลัพธ์จริง:

```text
SET pitfall:no_ttl:demo โดยไม่ใส่ EX/PX -- TTL ที่ได้กลับมา: -1 (-1 แปลว่า 'ไม่มีวันหมดอายุ')
```

**`-1` คือค่าที่ `TTL` คืนเมื่อ key มีอยู่จริงแต่ไม่มีวันหมดอายุ** (ต่างจาก `-2` ที่คืนเมื่อ key ไม่มีอยู่เลย)
— key แบบนี้จะอยู่ใน RAM ของ Redis **ตลอดไป**จนกว่าจะถูกลบด้วยมือ หรือ process จะถูก restart — ถ้าเป็น key
ที่ถูกสร้างซ้ำ ๆ ด้วยค่าที่ไม่ซ้ำกันตลอดเวลา (เช่น idempotency key ที่ไม่มี TTL, หรือ cache key ที่ผูกกับ
timestamp) memory ของ Redis จะโตขึ้นเรื่อย ๆ ไม่มีที่สิ้นสุดจนกว่าจะเต็ม RAM ทั้งหมดของเครื่อง — ตอนนั้น Redis
จะเริ่ม evict key ตาม policy ที่ตั้งไว้ (`maxmemory-policy` เช่น `allkeys-lru` — ลบ key ที่ไม่ถูกใช้นานสุด
ก่อน) หรือถ้าไม่ตั้ง policy ไว้เลย (`noeviction` เป็นค่า default) **จะเริ่มปฏิเสธคำสั่งเขียนใหม่ทั้งหมด**
ทำให้ระบบพังทั้งระบบ ไม่ใช่แค่ feature ที่ลืม TTL — บทเรียนคือ: **ทุก key ที่สร้างขึ้นมาควรถามตัวเองเสมอว่า
"key นี้ควรอยู่นานแค่ไหน" และตั้ง TTL ให้ตรงกับคำตอบนั้นเสมอ** ไม่มี key ไหนใน Redis ที่ "ไม่ต้องมี TTL" จริง ๆ
นอกจาก key ที่ตั้งใจให้เป็นข้อมูลถาวรจริง ๆ ซึ่งควรพิจารณาว่าเหมาะกับ PostgreSQL มากกว่าตั้งแต่แรก

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม cache-aside ให้กับ endpoint `GET /api/v1/events/{id}` (สมมติว่ามีตาราง `events` แบบ
   เดียวกับ `books`) โดยใช้ TTL ที่ต่างจาก book (เช่น 5 นาทีแทน 60 วินาที เพราะข้อมูล event เปลี่ยนน้อยกว่า
   status ของหนังสือ) — *hint*: โครงสร้างฟังก์ชันเหมือนหัวข้อ 83.3 ทุกประการ เปลี่ยนแค่ SQL query และ TTL
   ที่ส่งเข้า `set_ex`

2. **(กลาง)** เขียน middleware (`axum::middleware::from_fn`) ที่ครอบ rate limiting logic จากหัวข้อ 83.7
   ให้ใช้ได้กับทุก route โดยไม่ต้องเขียนโค้ดซ้ำในแต่ละ handler ตามที่ Part 65 สอนเรื่อง middleware ไว้ — key
   ของ rate limit ควรแยกตาม IP address หรือ user_id (จาก session/JWT) ไม่ใช่ key คงที่ตัวเดียวสำหรับทุกคน —
   *hint*: ดึง IP จาก `ConnectInfo<SocketAddr>` extractor ของ Axum แล้วผูกเข้ากับ Lua script จากหัวข้อ 83.7

3. **(ยาก)** implement **sliding window log** rate limiter ที่แม่นยำกว่า fixed window ในหัวข้อ 83.7 ด้วย
   Sorted Set — ทุกครั้งที่มี request ให้ `ZADD` timestamp ปัจจุบัน (เป็น score) เข้า key ของ user นั้น แล้ว
   `ZREMRANGEBYSCORE` ตัด entry ที่เก่ากว่า window ออกก่อนนับด้วย `ZCARD` — ทั้งสามคำสั่งต้องรวมเป็น Lua
   script เดียวเพื่อ atomicity เหมือนหัวข้อ 83.7 — *hint*: score ของ `ZADD` ควรเป็น timestamp ระดับ
   millisecond บวกด้วยค่าสุ่มเล็ก ๆ (หรือใช้ Redis Stream ID) เพื่อไม่ให้ score ชนกันถ้ามีสอง request มาถึง
   ในเวลาเดียวกันเป๊ะ

4. **(ยาก/ประยุกต์)** แก้ปัญหา **cache stampede** (thundering herd) — สถานการณ์: cache key ของข้อมูลที่ถูก
   อ่านหนักมากหมดอายุพร้อมกัน ทำให้ request จำนวนมากพร้อมกันเจอ cache miss พร้อมกันหมด แล้วยิง query
   PostgreSQL ตัวเดียวกันซ้ำ ๆ พร้อมกันหลายสิบครั้งโดยไม่จำเป็น (ทั้งที่ query เดียวก็พอสำหรับทุก request ที่
   รอผลลัพธ์เดียวกัน) — implement "distributed lock" ด้วย `SET NX EX` (แบบเดียวกับหัวข้อ 83.6) ที่ทำให้มีแค่
   **หนึ่ง** request เท่านั้นที่ไป query PostgreSQL จริงตอน cache miss ส่วน request อื่นที่มาพร้อมกันให้รอสั้น
   ๆ แล้วอ่านจาก cache ที่ request แรกเพิ่ง populate เสร็จ — *hint*: ใช้ pattern เดียวกับ "claim แล้ว poll รอ
   record" ที่ handler `create_booking` ในหัวข้อ 83.6 ใช้ทุกประการ เพียงแค่เปลี่ยนจาก "claim การสร้าง
   booking" เป็น "claim การ regenerate cache"

## สรุป

บทนี้เอา Redis มาแก้ปัญหาที่สามบทก่อนหน้าทิ้งไว้เป็นโครงร่างที่ยังไม่สมบูรณ์ให้เสร็จสมบูรณ์จริง — ไม่ใช่แค่
สอน Redis เป็นเทคโนโลยีใหม่ที่แยกออกมาต่างหาก: **Part 75** ทิ้งปัญหา "shared session store ข้ามหลาย instance"
ไว้ บทนี้แก้ด้วย `tower-sessions-redis-store` และพิสูจน์ด้วยสอง server instance จริงที่อ่าน session ของกันและ
กันได้ **Part 78** ทิ้ง idempotency-key store และ rate limiter แบบ in-memory ที่ "ไม่ scale ข้าม instance"
ไว้ บทนี้แก้ด้วย `SET NX EX` (atomic idempotency claiming) และ `INCR`/Lua script (atomic rate limiting)
พร้อมพิสูจน์ race condition จริงทั้งที่มีและไม่มี atomicity **Part 74** ทิ้งปัญหา "refresh token revoke ไม่ได้"
ไว้ บทนี้แก้ด้วย `jti` claim ผูกกับ Redis revocation record ที่มี TTL พอดีกับอายุที่เหลือของ token

นอกเหนือจากการปิดสามจุดนี้ บทนี้ยังสอน **cache-aside pattern** เต็มรูปแบบพร้อมวัด latency จริง (cache hit
เร็วกว่า miss หลายเท่าตัว), **cache invalidation** สองแนวทางพร้อมพิสูจน์ stale read จริง, data structure ของ
Redis ที่ไม่ใช่ string ธรรมดา (**Sorted Set** สำหรับ leaderboard, **Set** สำหรับ tag filtering), และหลักการ
**graceful degradation** เมื่อ Redis เองล่ม — พร้อมยืนยันด้วยการปิด Redis จริงแล้วดูว่าระบบยังตอบสนองได้

หลักการที่สำคัญที่สุดที่ควรจำจากบทนี้คือ **atomicity** — ทุกครั้งที่มีมากกว่าหนึ่ง request เข้ามาพร้อมกันและ
ต้องแก้ไขข้อมูลตัวเดียวกัน (claim idempotency key, เพิ่ม counter ของ rate limiter, สร้าง session ใหม่) การ
แยกตรรกะเป็นหลายคำสั่ง (เช็คก่อน แล้วเขียน) เปิดช่องให้เกิด race condition เสมอ ไม่ว่าจะดูเหมือนไม่มีปัญหา
ตอนทดสอบแบบ sequential ก็ตาม — คำตอบของ Redis คือ atomic primitive (`SET NX`, `INCR`) หรือ Lua script
(`EVAL`) ที่รวมหลายขั้นตอนให้เป็นการกระทำเดียวที่แบ่งแยกไม่ได้บนฝั่ง server เสมอ

Part ถัดไป (**Part 84: Background Jobs และ Task Queues**) จะพาไปดูอีกด้านของการทำงานแบบ asynchronous ในระดับ
ระบบ — งานที่ไม่ควรทำใน request-response cycle ตรง ๆ (เช่นส่งอีเมล, ประมวลผลไฟล์ขนาดใหญ่, สร้างรายงาน) ควรถูก
"queue" ไว้ให้ worker แยกไปทำทีหลัง — และ Redis ที่เพิ่งเรียนไปในบทนี้ (โดยเฉพาะ List และ Sorted Set) จะ
กลับมาเป็นหนึ่งในตัวเลือกของ backend สำหรับ queue เหล่านั้นด้วย

---

**Part ก่อนหน้า:** [Message Queues: RabbitMQ/Kafka Integration](part-082-message-queues.md) | **Part ถัดไป:** [Background Jobs และ Task Queues](part-084-background-jobs.md)
