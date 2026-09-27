# Part 75: Authentication: Session-based และ OAuth2

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 280 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความต่างระหว่าง **stateless authentication** (JWT ที่ Part 74 สอน) กับ **stateful
  authentication** (session-based) ได้อย่างเจาะจงในระดับ tradeoff จริง — ไม่ใช่แค่ "JWT ไม่เก็บ state,
  session เก็บ state" แบบท่องจำ แต่บอกได้ว่าทำไม session ถึง **revoke ได้ทันที** ในขณะที่ JWT ทำไม่ได้โดย
  ธรรมชาติ และทำไม session ถึงแลกมาด้วยภาระเรื่อง **shared session store** ที่ JWT ไม่มี
- ตั้งระบบ session จริงด้วย `tower-sessions` (เวอร์ชัน 0.15.0 — ตรวจสอบจริงกับ crates.io ณ วันที่เขียนบทนี้)
  บน Axum ตั้งแต่ `MemoryStore` (dev/single-instance) ไปจนถึงเข้าใจว่าทำไมต้องเปลี่ยนเป็น Redis-backed
  store ตอน production หลายเครื่อง (เชื่อมกับ Part 83)
- อ่านและตั้งค่า cookie attribute ทั้งสามตัวที่สำคัญที่สุดต่อความปลอดภัยของ session — `HttpOnly`, `Secure`,
  `SameSite` — พร้อมพิสูจน์ด้วย `curl` จริงว่าแต่ละตัวมีผลต่อพฤติกรรมจริงอย่างไร ไม่ใช่แค่ท่องนิยาม
- อธิบายได้ว่าทำไม authentication ที่ใช้ cookie ถึงเสี่ยงต่อ **CSRF** ในแบบที่ bearer token ใน
  `Authorization` header ไม่เสี่ยง และรู้จักทั้ง `SameSite` (การป้องกันหลักยุคปัจจุบัน) และ CSRF token
  แบบดั้งเดิม (defense-in-depth) ในระดับที่เพียงพอต่อการนำไปใช้จริง
- แยกแยะ **OAuth2** ออกจาก "authentication" แบบที่คนทั่วไปเข้าใจผิดได้อย่างถูกต้อง — เข้าใจว่า OAuth2 คือ
  **delegated authorization** (ให้แอปของคุณเข้าถึงข้อมูลบางส่วนของผู้ใช้บนบริการอื่นแทนผู้ใช้) และรู้ว่า
  **OpenID Connect (OIDC)** คือชั้นที่สร้างทับ OAuth2 เพื่อทำให้ "login" เป็นมาตรฐานจริง ๆ
- Implement **Authorization Code Flow** (ไม่ใช่ Implicit Flow ที่เลิกแนะนำแล้ว) เต็มรูปแบบด้วย crate
  `oauth2` เวอร์ชัน 5.0.0 กับ GitHub OAuth App จริง — รวม **PKCE** และ **state parameter** สองกลไกป้องกัน
  ที่ทำให้ flow นี้ปลอดภัยจริงในโลกที่มี attacker
- รวมทั้งสองครึ่งของบทเข้าด้วยกัน: ออกแบบระบบที่รับ login ได้ทั้งแบบ username/password (session ธรรมดา)
  และแบบ "Sign in with GitHub" (OAuth2) โดยทั้งสองทางจบลงที่ **local session เดียวกัน** — โค้ดปลายทางของ
  แอป (endpoint อื่น ๆ ทั้งหมด) ไม่ต้องรู้เลยว่าผู้ใช้ authenticate มาทางไหน

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้ใช้คำว่า cookie, header, redirect,
  status code (`200`, `302`/`303`, `400`, `401`) ตามความหมายมาตรฐานที่ Part 61 สอนไว้ตรง ๆ ไม่อธิบายซ้ำ
- **Part 62-64 (Axum พื้นฐาน, Routing/Handlers, State/Extractors)**: บทนี้พึ่งพา pattern `AppState` +
  `State<T>` extractor ที่ Part 64 สอนไว้อย่างหนัก — `tower-sessions` เพิ่ม extractor ตัวใหม่ (`Session`)
  เข้ามาอีกตัวที่ทำงานแบบเดียวกับ `State<T>` ทุกประการ (implement `FromRequestParts`) เพียงแต่ข้อมูลของมัน
  มาจาก cookie + session store แทนที่จะมาจาก `.with_state()` ตรง ๆ — ถ้า `FromRequestParts` vs
  `FromRequest` และกฎ "extractor ที่กิน body ต้องมาตัวสุดท้าย" จาก Part 64 ยังไม่แน่น ควรกลับไปทวนก่อน
- **Part 65 (Axum: Middleware ด้วย tower/tower-http)**: `SessionManagerLayer` ที่บทนี้ใช้คือ
  `tower::Layer` ตัวหนึ่ง ผูกเข้ากับ `Router` ด้วย `.layer()` แบบเดียวกับ middleware ทุกตัวที่ Part 65 สอน
  ไว้ — บทนี้จะไม่อธิบายกลไก `tower::Layer`/`Service` ซ้ำจากศูนย์
- **Part 66 (Axum: Error Handling แบบมืออาชีพ)**: error type ของ OAuth2 callback ในบทนี้ (`OAuthCallbackError`)
  ออกแบบด้วยหลักการเดียวกับ `AppError` ของ Part 66 ทุกประการ — `thiserror` + `impl IntoResponse` จุดเดียว
  + log ที่จุดกลาง (`tracing::warn!`) ก่อนแปลงเป็น response
- **Part 74 (Authentication: JWT)**: นี่คือ**จุดเทียบเคียงหลัก**ของบทนี้ — บทนี้สมมติว่าคุณเพิ่งเรียน Part
  74 มาสด ๆ: เข้าใจว่า JWT คือ token ที่ server verify ด้วยการตรวจ signature (ไม่ query database) เข้าใจ
  ปัญหาเรื่อง **revocation** ของ JWT (ต้องมี blocklist/short expiry + refresh token rotation ถึงจะ revoke
  ได้ ซึ่งเป็นภาระเพิ่มที่ Part 74 อธิบายไว้) — บทนี้ทั้งบทคือการเสนอ**ทางเลือกอีกฝั่ง**ของ spectrum เดียวกัน
- **Part 39 (Mutex, Arc และ Shared-State Concurrency)** และ **Part 46-50 (Async/Tokio)**: ตัวอย่าง
  `AppState` ในบทนี้ยังใช้ `Arc<Mutex<HashMap<...>>>` เป็น in-memory store จำลอง database เหมือนบทก่อน ๆ
  ในโมดูลนี้ — และจะเจอ**กับดักจริง**เรื่อง `MutexGuard` ค้างข้าม `.await` ในหัวข้อ 75.5 ที่ต้องใช้ความเข้าใจ
  จาก Part 39/46-50 ตรง ๆ ในการอ่านและแก้
- **สิ่งที่บทนี้จะเชื่อมไปข้างหน้า (ยังไม่ต้องรู้ตอนนี้)**: Part 76 (Authorization และ RBAC) จะสร้างต่อจาก
  "รู้ว่า user คนนี้คือใคร" (authentication — บทนี้) ไปสู่ "user คนนี้ทำอะไรได้บ้าง" (authorization) — Part
  81 (Microservices) และ Part 83 (Caching ด้วย Redis) จะกลับมาที่ปัญหา "shared session store ข้ามหลาย
  instance" ที่บทนี้เกริ่นไว้ในหัวข้อ 75.8 อย่างเต็มรูปแบบ

## เนื้อหา

### 75.1 ทวนจาก Part 74: JWT แก้ปัญหาอะไร แล้วอะไรที่ JWT แก้ไม่ได้

Part 74 สอนให้คุณสร้างระบบ authentication ที่ server **ไม่ต้องจำอะไรเลย** เกี่ยวกับ session ของผู้ใช้ —
ตอน login สำเร็จ server เซ็น JWT ที่มีข้อมูลผู้ใช้ (`sub`, `exp`, claim อื่น ๆ) ด้วย secret/private key แล้ว
ส่งกลับไปให้ client เก็บไว้เอง ทุกครั้งที่ client ยิง request มาพร้อม token นั้น server แค่ **ตรวจ
signature** (คำนวณ HMAC/RSA/EdDSA แล้วเทียบ) — ไม่ต้อง query database เลยแม้แต่ครั้งเดียวเพื่อรู้ว่า "user
นี้ login อยู่จริงไหม" นี่คือที่มาของคำว่า **stateless**: server ไม่เก็บ "สถานะการ login" ของใครไว้เลย
ความจริงทั้งหมดเกี่ยวกับ session อยู่ใน token ที่ client ถืออยู่ในมือ 100%

คุณสมบัตินี้ทรงพลังมากในหลายสถานการณ์ (Part 74 อธิบายไว้แล้ว: scale แนวนอนง่าย เพราะ server ตัวไหนก็ตรวจ
token ได้โดยไม่ต้องคุยกับตัวอื่น, เหมาะกับ mobile app/SPA ที่ไม่มี cookie แบบ browser) — แต่ Part 74 หัวข้อ
ท้าย ๆ ก็ทิ้งปัญหาที่แก้ไม่ได้ง่าย ๆ ไว้ข้อหนึ่งที่สำคัญมาก: **จะ "เตะ" user คนหนึ่งออกจากระบบทันทีได้อย่างไร**

ลองนึกภาพสถานการณ์จริงสามแบบ:

1. **ผู้ใช้กด "logout จากทุกอุปกรณ์"** เพราะสงสัยว่า token หลุด
2. **แอดมินแบน user คนหนึ่งเพราะทำผิดกฎ** ต้องการให้ user คนนั้นถูกเตะออกทันที ไม่ใช่รอ token หมดอายุเอง
3. **ตรวจพบว่า token ของ user คนหนึ่งรั่วไหล** (หลุดไปอยู่ใน log, ถูกขโมยผ่าน XSS) ต้องยกเลิก token ตัวนั้น
   ทันทีโดยไม่กระทบ token อื่นของ user คนเดียวกันที่ยังปลอดภัยอยู่

ด้วย JWT แบบ stateless ล้วน ๆ **ทำสามอย่างนี้ไม่ได้เลย** เพราะ server ไม่มี "ที่" ให้ไปบอกว่า "token ตัวนี้
ใช้ไม่ได้แล้ว" — token ที่เซ็นไปแล้วจะ valid ไปจนกว่าจะถึงเวลาที่ `exp` กำหนดไว้เสมอ ไม่ว่า server จะ "อยากจะ"
ยกเลิกมันแค่ไหนก็ตาม (Part 74 หัวข้อท้ายบทแก้ปัญหานี้ด้วยการเพิ่ม **state กลับเข้ามาบางส่วน** — blocklist
ของ token ที่ถูก revoke, หรือ refresh token ที่ตรวจสอบกับ database ได้ทุกครั้งที่ใช้ต่ออายุ access token —
สังเกตว่าทางแก้ทั้งสองทางคือการ**เพิ่ม state กลับเข้าไปในระบบ** ซึ่งเป็นสิ่งที่ JWT ตั้งใจจะเลี่ยงตั้งแต่แรก)

นี่คือจุดเริ่มต้นของบทนี้: **session-based authentication** ไม่ได้ "ล้าสมัย" หรือ "ถูก JWT แทนที่ไปแล้ว" ตาม
ที่บทความทั่วไปในอินเทอร์เน็ตชอบพูด — มันคือทางเลือกอีกฝั่งของ spectrum เดียวกัน ที่**แลก scale ง่าย ๆ ของ
stateless ไปกับ revocation ที่ทำได้ทันทีโดยธรรมชาติ** และสำหรับแอปจำนวนมาก (โดยเฉพาะเว็บแอปที่ผู้ใช้ล็อกอิน
ผ่าน browser ธรรมดา ไม่ใช่ mobile app หรือ public API) การแลกแบบนี้คุ้มค่ามาก

### 75.2 Session-based Authentication คืออะไร: แนวคิดพื้นฐาน

แนวคิดของ session-based authentication ย้อนกลับไปยุคแรก ๆ ของเว็บ (framework อย่าง Django, Rails, Express
ทุกตัวมี session middleware มาให้ default) และเรียบง่ายกว่า JWT มากในระดับแนวคิด:

1. ผู้ใช้ login สำเร็จ (ส่ง username/password ถูกต้อง)
2. **server สร้าง session ใหม่** — เป็นก้อนข้อมูล (มักเป็น key-value ธรรมดา เช่น `user_id: 42`) พร้อม
   **session ID** แบบสุ่มที่คาดเดาไม่ได้ (เช่น UUID หรือ random string 128+ bit)
3. server **เก็บก้อนข้อมูลนั้นไว้ที่ตัวเอง** (ในหน่วยความจำ, ในฐานข้อมูล, หรือใน Redis — เรียกรวม ๆ ว่า
   **session store**) โดยใช้ session ID เป็น key
4. server ส่ง session ID (ไม่ใช่ข้อมูลจริง!) กลับไปให้ browser ผ่าน **`Set-Cookie` header**
5. browser **เก็บ cookie นี้ไว้อัตโนมัติ** และแนบกลับไปกับทุก request ที่ยิงไปยัง domain เดียวกันโดยไม่ต้อง
   เขียนโค้ด JavaScript อะไรเพิ่มเลย (นี่คือความต่างสำคัญจาก JWT ที่ client ต้องจัดการเก็บ token เอง แล้ว
   แนบเข้า `Authorization` header ด้วยตัวเองทุกครั้ง)
6. ทุก request ที่เข้ามา server **อ่าน session ID จาก cookie** แล้ว**ค้นหา**ก้อนข้อมูลที่ตรงกันใน session
   store — ถ้าเจอและยังไม่หมดอายุ ก็รู้ว่า request นี้มาจาก user คนไหน

สังเกตความต่างจาก JWT ให้ชัด: สิ่งที่ client ถืออยู่ (session ID ใน cookie) **ไม่มีข้อมูลอะไรอยู่ในตัวมันเอง
เลย** มันเป็นแค่ "ใบเสร็จ" ที่ใช้ไปแลกข้อมูลจริงกับ server อีกที — ข้อมูลจริงทั้งหมด (`user_id`,
`username`, สิทธิ์ต่าง ๆ) **อยู่ที่ server เท่านั้น** นี่คือที่มาของคำว่า **stateful**: server ต้องเก็บ
"สถานะ" ของทุก session ที่ active อยู่ไว้ตลอดเวลา

ผลลัพธ์โดยตรงจากความต่างนี้คือคำตอบของปัญหาในหัวข้อ 75.1 ทันที: **การ revoke session ทำได้แค่ลบข้อมูลออกจาก
session store** — ครั้งเดียว จบ ไม่ต้องมี blocklist แยก ไม่ต้องรอ token หมดอายุ ทันทีที่ server ลบ session
ID นั้นออกจาก store, request ต่อไปที่แนบ session ID เดิมมาจะหา**ไม่เจอ**ในสโตร์ทันที → ปฏิเสธทันที

| มิติ | JWT (stateless, Part 74) | Session-based (stateful, บทนี้) |
|---|---|---|
| ข้อมูลที่ client ถือ | Token ที่มีข้อมูลจริง เซ็นด้วย signature | Session ID สุ่ม — ไม่มีข้อมูลอะไรอยู่ในตัวมันเอง |
| server ต้องเก็บอะไรไหม | ไม่ต้อง (ตรวจแค่ signature) | ต้องเก็บก้อนข้อมูล session ทุกตัวที่ active |
| การ revoke ทันที | ทำไม่ได้โดยธรรมชาติ ต้องเพิ่ม blocklist/refresh-token infra | ทำได้ทันที — ลบ record ออกจาก store บรรทัดเดียว |
| Scale แนวนอน (หลาย server instance) | ง่าย — แต่ละ instance ตรวจ signature เองได้ ไม่ต้องคุยกัน | ต้องมี **shared session store** ที่ทุก instance เข้าถึงร่วมกัน (ดูหัวข้อ 75.8) |
| ขนาดข้อมูลที่ต้องส่งทุก request | Token ทั้งก้อน (มักหลายร้อย byte ขึ้นไป) แนบทุก request | Session ID สั้น ๆ (สิบกว่า byte) — ข้อมูลจริงไม่ต้องส่งซ้ำ |
| เหมาะกับ client ประเภทไหน | mobile app, SPA, public API, ระบบ microservices ข้าม domain | เว็บแอปที่ผู้ใช้ล็อกอินผ่าน browser ตัวเดียวเป็นหลัก |
| จุดอ่อนด้าน CSRF | ต่ำ — bearer token ใน header ไม่ถูกส่งอัตโนมัติข้าม site | สูงกว่า — cookie ถูก browser ส่งอัตโนมัติข้าม site (ดูหัวข้อ 75.7) |

**นี่คือตารางที่สำคัญที่สุดในครึ่งแรกของบทนี้** — ไม่มีฝั่งไหน "ดีกว่า" อีกฝั่งโดยเด็ดขาด แต่ละแถวคือ
tradeoff จริงที่ทีมวิศวกรต้องเลือกตามบริบทของระบบตัวเอง ระบบจำนวนมากในโลกจริงถึงกับใช้**ทั้งสองแบบพร้อมกัน**
ในระบบเดียว (เว็บแอปหลักใช้ session, แต่มี public API แยกที่ให้ third-party เรียกด้วย JWT/API key) — หัวข้อ
75.20 ท้ายบทจะกลับมาคุยเรื่องการผสมสองแบบนี้อีกครั้งหลังจากคุณเห็นทั้งสองแบบเต็มรูปแบบแล้ว

### 75.3 ตั้งโปรเจกต์ด้วย `tower-sessions`

crate ที่ใช้กันจริงและยัง maintain อยู่มากที่สุดสำหรับ session บน Axum/tower คือ **`tower-sessions`** —
เขียนโดยผู้เขียนคนเดียวกับ `axum-login` (ระบบ authentication เต็มรูปแบบที่สร้างทับ `tower-sessions`) และ
เป็น `tower::Layer` ที่ทำงานร่วมกับ ecosystem `tower`/`tower-http` ที่ Part 65 สอนไว้โดยตรง — ตรวจสอบเวอร์ชัน
ล่าสุดจาก crates.io จริง ณ วันที่เขียนบทนี้ (`cargo add tower-sessions`) ได้ **`0.15.0`**

เพิ่ม dependency เข้าโปรเจกต์:

```toml
[dependencies]
axum = { version = "0.8.9", features = ["macros"] }
tokio = { version = "1.53.1", features = ["full"] }
tower-sessions = "0.15.0"
time = "0.3.55"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
tracing = "0.1.44"
tracing-subscriber = "0.3.23"
```

สังเกตว่า **ไม่ต้องเพิ่ม crate แยกสำหรับ memory store** — `tower-sessions` เปิด feature `memory-store`
(และ `axum-core` ที่ให้ `Session` extractor ทำงานกับ Axum ได้) เป็น **default feature** อยู่แล้ว ดูได้จาก
`Cargo.toml` จริงของ crate เวอร์ชัน 0.15.0:

```toml
# ตัดมาจาก Cargo.toml จริงของ tower-sessions 0.15.0
[features]
axum-core = ["tower-sessions-core/axum-core"]
default = [
    "axum-core",
    "memory-store",
]
memory-store = ["tower-sessions-memory-store"]
```

ถ้าวันหนึ่งต้องปิด default feature (เช่นตอนสลับไปใช้ Redis store จริงใน production ตาม Part 83 ที่ไม่ต้องการ
`memory-store` ติดมาด้วยเปล่า ๆ) ก็เขียน `tower-sessions = { version = "0.15.0", default-features = false,
features = ["axum-core"] }` แทน — แต่สำหรับบทนี้และ dev environment ปล่อย default ไว้เหมาะที่สุด

#### ตัวอย่างที่เล็กที่สุดที่ทำงานได้จริง

```rust
use axum::{response::IntoResponse, routing::get, Router};
use time::Duration;
use tower_sessions::{Expiry, MemoryStore, Session, SessionManagerLayer};

const COUNTER_KEY: &str = "counter";

// Session ทำงานเป็น extractor ตัวหนึ่ง -- implement FromRequestParts เหมือน State<T>/Path<T>/Query<T>
// ที่ Part 64 สอนไว้ทุกประการ (ไม่แตะ body เลย ดังนั้นวางไว้ตำแหน่งไหนก่อน body extractor ก็ได้)
async fn handler(session: Session) -> impl IntoResponse {
    let count: usize = session.get(COUNTER_KEY).await.unwrap().unwrap_or(0);
    session.insert(COUNTER_KEY, count + 1).await.unwrap();
    format!("นับได้ {} ครั้งแล้ว (session เดียวกัน)", count + 1)
}

#[tokio::main]
async fn main() {
    // ขั้นที่ 1: เลือก session store -- MemoryStore เก็บทุกอย่างใน HashMap ในหน่วยความจำของ process เดียว
    let session_store = MemoryStore::default();

    // ขั้นที่ 2: สร้าง SessionManagerLayer จาก store ตัวนั้น พร้อมตั้งค่านโยบายของ cookie/expiry
    let session_layer = SessionManagerLayer::new(session_store)
        .with_expiry(Expiry::OnInactivity(Duration::minutes(30)));

    // ขั้นที่ 3: .layer() เข้ากับ Router -- แบบเดียวกับ middleware ทุกตัวที่ Part 65 สอนไว้
    let app: Router<()> = Router::new()
        .route("/", get(handler))
        .layer(session_layer);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3175").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**อธิบายกลไกทีละส่วน:**

- **`MemoryStore::default()`** — session store ที่ง่ายที่สุดที่มีมาให้ในตัว `tower-sessions` เอง เก็บข้อมูล
  session ทั้งหมดไว้ใน `HashMap` ในหน่วยความจำของ process — **เร็วมาก ไม่มี network round-trip เลย** แต่มี
  ข้อจำกัดสำคัญสองข้อที่ต้องรู้ตั้งแต่ต้น (หัวข้อ 75.8 จะขยายเรื่องนี้เต็มรูปแบบ): (1) **ข้อมูลหายทันทีที่
  restart process** — deploy ใหม่ครั้งหนึ่ง ทุกคนที่ login อยู่ต้อง login ใหม่ (2) **ใช้ได้กับ server แค่
  instance เดียว** — ถ้า deploy หลาย instance พร้อมกัน (load balancer กระจาย request) แต่ละ instance มี
  `HashMap` ของตัวเอง ไม่แชร์กัน user คนเดียวอาจ login สำเร็จที่ instance A แต่ถูก load balancer ส่ง request
  ต่อไปยัง instance B ที่ไม่รู้จัก session นั้นเลย — **นี่คือเหตุผลที่ MemoryStore เหมาะกับ dev/testing/
  single-instance เท่านั้น**
- **`SessionManagerLayer::new(session_store)`** — สร้าง `tower::Layer` ที่ทำหน้าที่สามอย่างทุก request:
  (1) อ่าน session ID จาก cookie ที่ browser แนบมา (2) โหลดข้อมูล session จาก store ด้วย ID นั้น (3) หลัง
  handler ทำงานจบ ถ้าข้อมูล session ถูกแก้ไข (`is_modified()` เป็น true) ก็บันทึกกลับเข้า store แล้วส่ง
  `Set-Cookie` กลับไปให้ browser (ครั้งแรกที่สร้าง session ใหม่ หรือทุกครั้งที่ session ID เปลี่ยน)
- **`Expiry::OnInactivity(Duration::minutes(30))`** — นโยบาย expiry ของ session: "หมดอายุถ้าไม่มี activity
  30 นาที" — ทุกครั้งที่ session ถูกใช้งาน (`session.load()` เกิดขึ้นทุก request ที่มี cookie แนบมา) นาฬิกา
  จะถูกรีเซ็ตใหม่ ต่างจาก `Expiry::AtDateTime(...)` ที่กำหนดเวลาหมดอายุตายตัวไม่สนใจ activity และ
  `Expiry::OnSessionEnd` (ค่า default ถ้าไม่ตั้งอะไรเลย) ที่หมายถึง "session cookie" แบบดั้งเดิม — cookie
  ที่ไม่มี `Max-Age`/`Expires` เลย ทำให้ browser ลบมันทิ้งเองตอนปิด browser (ไม่ใช่ tab, ต้องปิดทั้ง browser)
- **`session: Session`** เป็น parameter ของ `handler` — เพราะ `Session` implement
  `FromRequestParts<S>` (เหมือน `State<T>`/`Path<T>` ที่ Part 64 สอน) ตัว Axum จะดึงมันออกมาจาก request
  extensions โดยอัตโนมัติ **โดยมีเงื่อนไขว่า `SessionManagerLayer` ต้องถูก `.layer()` เข้ากับ route นั้นไว้
  ก่อนแล้ว** (ถ้าลืม — ดูกับดักข้อ 2 ท้ายบทนี้ พร้อม error message จริง)
- **`session.get(...)`/`session.insert(...)`** เป็น `async fn` (ต้อง `.await`) เพราะ session store บางแบบ
  (Redis, database) ต้องมี network round-trip จริง — `MemoryStore` ไม่ต้องรอ network แต่ signature ของ
  `Session` ต้องเหมือนกันไม่ว่า store ข้างหลังจะเป็นแบบไหน (เพื่อให้สลับ store ได้โดยไม่ต้องแก้โค้ด handler
  แม้แต่บรรทัดเดียว — หลักการเดียวกับ trait object/generic ที่ Part 28 สอนไว้เรื่อง "เขียนโค้ดที่ไม่สนใจ
  รายละเอียด implementation")

### 75.4 Cookie Attributes เจาะลึก: `HttpOnly`, `Secure`, `SameSite`

ก่อนเขียนระบบ login เต็มรูปแบบ ต้องเข้าใจ cookie attribute สามตัวที่ `SessionManagerLayer` ตั้งค่าให้เป็น
ค่า default ที่ปลอดภัยอยู่แล้ว — แต่การเข้าใจว่า**ทำไม**ค่า default เป็นแบบนี้สำคัญมาก เพราะมีสถานการณ์จริงที่
ต้องปรับค่าเหล่านี้เอง (เช่น dev บน `http://` ที่หัวข้อ 75.5 จะเจอ)

ดูค่า default จริงจาก source code ของ `tower-sessions` 0.15.0:

```rust
// ตัดมาจาก src/service.rs ของ tower-sessions 0.15.0 -- ค่า default ทุกตัวของ cookie ที่มันสร้างให้
impl Default for SessionConfig<'_> {
    fn default() -> Self {
        Self {
            name: "id".into(),
            http_only: true,
            same_site: SameSite::Strict,
            expiry: None,
            secure: true,
            path: "/".into(),
            domain: None,
            always_save: false,
        }
    }
}
```

#### `HttpOnly` — ป้องกัน JavaScript อ่าน cookie

`HttpOnly: true` (default) บอก browser ว่า **"ห้าม JavaScript บนหน้าเว็บนี้เข้าถึง cookie ตัวนี้ผ่าน
`document.cookie` เด็ดขาด"** — cookie ยังถูกส่งไปกับ HTTP request ตามปกติ (นั่นคือจุดประสงค์ของมัน) แต่โค้ด
JavaScript ใดก็ตามที่รันอยู่บนหน้าเว็บ (รวมถึงโค้ดที่ attacker แฝงเข้ามาผ่าน **XSS** — cross-site scripting)
**อ่านค่ามันไม่ได้เลย**

นี่คือข้อได้เปรียบด้านความปลอดภัยที่สำคัญที่สุดข้อหนึ่งของ session cookie เทียบกับวิธีเก็บ JWT แบบที่นักพัฒนา
มือใหม่มักทำ (เก็บ JWT ใน `localStorage` แล้วอ่านด้วย JavaScript เพื่อแนบเข้า `Authorization` header เอง —
Part 74 อาจพูดถึงความเสี่ยงนี้ไว้บ้างแล้ว): ถ้าเว็บแอปมีช่องโหว่ XSS ที่ไหนสักที่ (แม้จะเล็กน้อย เช่น
comment field ที่ไม่ escape HTML ให้ถูกต้อง) attacker ที่ยิง script เข้ามาได้จะ**อ่าน JWT จาก
`localStorage` ออกไปได้ทันที** แล้วเอาไปใช้ปลอมตัวเป็น user คนนั้นจากที่ไหนก็ได้ในโลก — แต่ถ้า session
cookie ตั้ง `HttpOnly: true` ไว้ **script ตัวเดียวกันนั้นอ่าน session ID ไม่ได้เลยแม้จะรันอยู่บนหน้าเว็บ
เดียวกันจริง ๆ** (attacker ยังทำอันตรายอื่นผ่าน XSS ได้อยู่ เช่นสั่งให้ browser ยิง request ปลอมในนามของ
user แต่ **ขโมย session ID ไปใช้นอกบริบทของหน้าเว็บนั้นไม่ได้**)

#### `Secure` — ส่งผ่าน HTTPS เท่านั้น

`Secure: true` (default) บอก browser ว่า **"ส่ง cookie นี้ไปกับ request ที่เป็น HTTPS เท่านั้น — ถ้าเป็น
HTTP ธรรมดา ห้ามส่ง"** เหตุผลตรงไปตรงมา: ถ้าไม่มี attribute นี้ และมีจุดใดในระบบที่ยัง serve เนื้อหาผ่าน
`http://` อยู่ (แม้จะ redirect ไป `https://` ทันที) cookie ที่ถูกส่งไปกับ request แรกที่เป็น `http://` นั้น
**เดินทางผ่าน network แบบไม่เข้ารหัส** — ใครก็ตามที่ดักฟัง network ได้ (Wi-Fi สาธารณะที่ไม่ปลอดภัย,
ISP ที่ประสงค์ร้าย, MITM proxy) จะเห็น session ID เป็น plaintext ทันที แล้วเอาไปปลอมตัวเป็น user นั้นได้

#### `SameSite` — ตัวป้องกันหลักของ CSRF (จะขยายในหัวข้อ 75.7)

ค่า default ของ `tower-sessions` คือ `SameSite::Strict` — ค่าที่**เข้มที่สุด** ในสาม ค่าที่เป็นไปได้:

| ค่า | พฤติกรรม |
|---|---|
| `Strict` | cookie ถูกส่งเฉพาะ request ที่มาจาก**เว็บไซต์เดียวกัน**เท่านั้น ไม่ว่าจะเป็นการคลิกลิงก์จากเว็บอื่นมา หรือ redirect ข้าม site มา ก็ไม่ส่ง |
| `Lax` (ค่าที่ browser ส่วนใหญ่ใช้เป็น default ของตัวเองถ้า dev ไม่ตั้งอะไรเลย) | cookie ถูกส่งกับ **top-level navigation แบบ GET** ข้าม site ได้ (เช่นคลิกลิงก์จากเว็บอื่นมา, ถูก redirect มาจากเว็บอื่น) แต่ **ไม่ส่ง**กับ request ที่ฝังอยู่ในหน้าเว็บอื่น (เช่น `<img>`, `<iframe>`, fetch ที่เว็บอื่นยิงมาที่เว็บเรา) |
| `None` | ส่งทุกกรณี ไม่ว่าจะมาจากที่ไหน — **ต้องคู่กับ `Secure: true` เสมอ** (browser สมัยใหม่บังคับ) และเป็นค่าที่เสี่ยง CSRF สูงสุด ใช้เมื่อจำเป็นต้องแชร์ cookie ข้าม site จริง ๆ เท่านั้น (เช่น embed widget ข้าม domain) |

หัวข้อ 75.7 จะอธิบายเจาะลึกว่าทำไม `SameSite` ถึงเป็นกลไกป้องกัน CSRF ที่สำคัญที่สุดในยุคปัจจุบัน — แต่สังเกต
ไว้ก่อนตรงนี้ว่า `Strict` แม้จะปลอดภัยที่สุด แต่ก็มีผลข้างเคียงที่ต้องรู้: หัวข้อ 75.16 (OAuth2 callback) จะ
เจอสถานการณ์จริงที่ **ต้องใช้ `Lax` ไม่ใช่ `Strict`** เพราะเหตุผลเชิงเทคนิคที่อธิบายตรงนั้น

#### ปรับค่าเหล่านี้ด้วย builder methods

`SessionManagerLayer` มี builder method ให้ปรับทุกค่าที่กล่าวมา (ทั้งหมดคือ `pub fn with_*(self, ...) ->
Self` — ยึด pattern เดียวกับ builder ที่ Part หลาย ๆ บทก่อนหน้าใช้):

```rust
use time::Duration;
use tower_sessions::{cookie::SameSite, Expiry, MemoryStore, SessionManagerLayer};

let session_layer = SessionManagerLayer::new(MemoryStore::default())
    .with_name("app_session")               // ชื่อ cookie -- default คือ "id" (เตี้ยและไม่บอกอะไร
                                              // เกี่ยวกับเทคโนโลยีที่ใช้ ตามคำแนะนำของ OWASP Session
                                              // Management Cheat Sheet ที่ comment ใน source code อ้างถึง)
    .with_http_only(true)                    // default: true -- ปกติไม่ต้องเรียกซ้ำ
    .with_same_site(SameSite::Lax)            // ปรับจาก Strict (default) เป็น Lax
    .with_secure(true)                        // default: true -- production ต้องเป็น true เสมอ
    .with_expiry(Expiry::OnInactivity(Duration::minutes(30)))
    .with_path("/");                          // default: "/" -- cookie ใช้ได้ทุก path ของเว็บไซต์
```

### 75.5 ตัวอย่างเต็ม: Login / Protected Route / Logout พร้อม `curl` จริง

มาประกอบทุกอย่างเป็นระบบเล็ก ๆ ที่ทำงานได้จริง — สามarco endpoint: `POST /login` (ตรวจ username/password
แล้วสร้าง session), `GET /me` (protected route ที่อ่านข้อมูลจาก session), `POST /logout` (ทำลาย session)

```rust
use axum::{
    extract::State,
    http::StatusCode,
    response::IntoResponse,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use time::Duration;
use tower_sessions::{Expiry, MemoryStore, Session, SessionManagerLayer};

#[derive(Clone)]
struct AppState {
    // demo เท่านั้น -- เก็บ password เป็น plaintext แบบนี้ห้ามทำใน production เด็ดขาด (ต้อง hash ด้วย
    // argon2/bcrypt เสมอ -- ดูแบบฝึกหัดข้อ 1 ท้ายบทที่ให้ลองแก้จุดนี้ด้วยตัวเอง)
    users: Arc<Mutex<HashMap<String, String>>>,
}

#[derive(Deserialize)]
struct LoginRequest {
    username: String,
    password: String,
}

#[derive(Serialize)]
struct MeResponse {
    user_id: String,
    username: String,
}

const USER_ID_KEY: &str = "user_id";
const USERNAME_KEY: &str = "username";

async fn login(
    State(state): State<AppState>,
    session: Session,
    Json(body): Json<LoginRequest>,
) -> Result<StatusCode, StatusCode> {
    // สำคัญมาก: ปล่อย MutexGuard ก่อนถึง .await ตัวแรกเสมอ (ตามหลักที่ Part 64/66 เตือนไว้เรื่อง
    // Arc<Mutex<T>> ใน handler async) -- ที่นี่ทำได้ง่ายด้วยการสรุปผลเป็น bool แล้วให้ lock ตาย (drop)
    // ทันทีที่ scope ปิด ก่อนเรียก session.insert().await ด้านล่าง (ดูกับดักข้อ 3 ท้ายบทว่าเกิดอะไรขึ้นถ้า
    // ไม่ทำแบบนี้)
    let password_matches = {
        let users = state.users.lock().unwrap();
        users.get(&body.username) == Some(&body.password)
    }; // <- lock ถูก drop ตรงนี้

    if !password_matches {
        return Err(StatusCode::UNAUTHORIZED);
    }

    session
        .insert(USER_ID_KEY, format!("user-{}", body.username))
        .await
        .unwrap();
    session.insert(USERNAME_KEY, body.username).await.unwrap();

    // ป้องกัน session fixation (หัวข้อ 75.6): สร้าง session ID ใหม่ทุกครั้งที่ privilege เปลี่ยน
    // (ที่นี่คือ "เพิ่งกลายเป็น authenticated") -- เก็บข้อมูลเดิมไว้ครบ แค่เปลี่ยน ID
    session.cycle_id().await.unwrap();

    Ok(StatusCode::OK)
}

async fn me(session: Session) -> Result<Json<MeResponse>, StatusCode> {
    let user_id: Option<String> = session.get(USER_ID_KEY).await.unwrap();
    let username: Option<String> = session.get(USERNAME_KEY).await.unwrap();
    match (user_id, username) {
        (Some(user_id), Some(username)) => Ok(Json(MeResponse { user_id, username })),
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

async fn logout(session: Session) -> impl IntoResponse {
    // flush() = clear() ข้อมูลทั้งหมด + delete() ออกจาก store + เซ็ต session id เป็น None
    // ครั้งถัดไปที่ SessionManagerLayer เห็นว่าไม่มี session id เหลือ มันจะส่ง Set-Cookie ที่ลบ cookie
    // ฝั่ง browser ออกด้วย (Max-Age=0) -- ดู curl transcript ด้านล่างที่พิสูจน์เรื่องนี้จริง
    session.flush().await.unwrap();
    StatusCode::NO_CONTENT
}

#[tokio::main]
async fn main() {
    let mut users = HashMap::new();
    users.insert("nan".to_string(), "hunter2".to_string());
    let state = AppState { users: Arc::new(Mutex::new(users)) };

    let session_store = MemoryStore::default();
    let session_layer = SessionManagerLayer::new(session_store)
        .with_secure(false) // *** เฉพาะ dev บน http://127.0.0.1 เท่านั้น! production ต้อง true เสมอ ***
        .with_same_site(tower_sessions::cookie::SameSite::Lax)
        .with_expiry(Expiry::OnInactivity(Duration::minutes(30)));

    let app = Router::new()
        .route("/login", post(login))
        .route("/me", get(me))
        .route("/logout", post(logout))
        .layer(session_layer)
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3175").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

**ทำไมต้อง `.with_secure(false)` ตรงนี้**: เซิร์ฟเวอร์ตัวอย่างนี้รันบน `http://127.0.0.1` (ไม่มี TLS) —
ถ้าปล่อยค่า default (`secure: true`) ไว้ cookie ที่ส่งกลับมาจะมี attribute `Secure` ติดไปด้วย ซึ่งในทาง
เทคนิคของ HTTP cookie spec (RFC 6265bis) **`localhost`/`127.0.0.1` ถูกนับเป็น "potentially trustworthy
origin" เป็นกรณีพิเศษ** (บทหัวข้อ กับดักที่พบบ่อย ข้อ 4 จะพิสูจน์ด้วย `curl` จริงว่า cookie ที่มี `Secure`
ยังใช้งานได้ปกติบน `127.0.0.1` แม้จะเป็น `http://` — ไม่ใช่ `https://`) แต่การเขียน `.with_secure(false)`
อย่างชัดเจนตอน dev ยังเป็นธรรมเนียมที่แนะนำ เพราะทันทีที่ deploy จริงไปยัง staging/production server ที่มี
hostname จริง (ไม่ใช่ `localhost`) ข้อยกเว้นนี้จะ**ไม่มีผลอีกต่อไป** และถ้าลืมเปลี่ยนกลับเป็น `true` (หรือลืม
ลบ `.with_secure(false)` ออก) production ของคุณจะรัน session cookie โดยไม่มี `Secure` attribute เลย — ช่อง
โหว่ด้าน MITM ตามที่อธิบายไว้ในหัวข้อ 75.4

#### ทดสอบด้วย `curl` จริง — สิ่งที่ต้องรู้เรื่อง cookie jar (`-c`/`-b`)

จุดที่นักพัฒนามือใหม่สับสนบ่อยที่สุดตอนทดสอบ session ด้วย `curl`: **`curl` ไม่ทำตัวเป็น browser** —
มันไม่เก็บ cookie จาก response หนึ่งไปใช้กับ request ถัดไปโดยอัตโนมัติ ถ้าไม่บอกมันชัด ๆ ว่าต้องเก็บ/ส่ง
cookie จากไฟล์ไหน (สิ่งที่ browser ทำให้อัตโนมัติในหัวข้อ 75.2 ข้อ 5) ต้องใช้ธง `-c <file>` (**c**ookie jar
— บันทึก `Set-Cookie` ที่ได้รับลงไฟล์นี้) และ `-b <file>` (**b**iscuit/cookie — อ่าน cookie จากไฟล์นี้ส่งไป
กับ request) — มาดูผลจริงทีละขั้น (รันเซิร์ฟเวอร์ข้างบนแล้วยิงตามลำดับนี้จริง):

```bash
$ curl -sS -i http://127.0.0.1:3175/me
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:31:45 GMT
```

ยังไม่ login เลย → `401` ตามคาด ทดสอบ login ด้วย password ผิดก่อน:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3175/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"wrongpass"}'
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:31:45 GMT
```

ทีนี้ login ด้วย password ที่ถูกต้อง **พร้อม `-c` เพื่อบันทึก cookie ที่ได้กลับมา**:

```bash
$ curl -sS -i -c /tmp/cookies.txt -X POST http://127.0.0.1:3175/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"hunter2"}'
```
```
HTTP/1.1 200 OK
set-cookie: id=lyxT0_ED3SHIZ0f2SHRZkQ; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
content-length: 0
date: Sun, 27 Sep 2026 00:31:45 GMT
```

สังเกต `set-cookie` header ที่ได้จริง — ตรงตามที่ตั้งค่าไว้ทุกประการ: `HttpOnly` ติดมา (ไม่มี `Secure`
เพราะเราตั้ง `with_secure(false)` ไว้), `SameSite=Lax`, `Path=/`, `Max-Age=1800` (30 นาที × 60 วินาที ตรงกับ
`Expiry::OnInactivity(Duration::minutes(30))`) — ดู content ของไฟล์ cookie jar ที่ `curl` บันทึกไว้:

```bash
$ cat /tmp/cookies.txt
```
```
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_127.0.0.1	FALSE	/	FALSE	1790470905	id	lyxT0_ED3SHIZ0f2SHRZkQ
```

ตอนนี้มาพิสูจน์ประเด็นสำคัญที่สุดของหัวข้อนี้ — **ถ้ายิง `/me` โดย "ลืม" ส่ง cookie กลับไป (ไม่ใส่ `-b`)
แม้จะ login สำเร็จไปแล้วเมื่อกี้ ก็ยังได้ `401` เหมือนไม่เคย login เลย**:

```bash
$ curl -sS -i http://127.0.0.1:3175/me
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:31:45 GMT
```

นี่ไม่ใช่บั๊ก — เป็นพฤติกรรมที่ถูกต้อง 100% เพราะ request นี้ไม่มี cookie แนบมาเลย server จึงไม่มีทางรู้ว่ามัน
เกี่ยวข้องกับ session ที่สร้างไว้ก่อนหน้านี้ยังไงเลย (เหมือนคนละคนที่ไม่เคย login มาก่อน) — คราวนี้ใส่ `-b`
เพื่อส่ง cookie ที่บันทึกไว้กลับไปด้วย:

```bash
$ curl -sS -i -b /tmp/cookies.txt http://127.0.0.1:3175/me
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 39
date: Sun, 27 Sep 2026 00:31:45 GMT

{"user_id":"user-nan","username":"nan"}
```

ได้ `200 OK` พร้อมข้อมูล user กลับมาถูกต้อง — พิสูจน์ว่า session ทำงานได้จริงตราบใดที่ cookie ถูกส่งไปด้วย
ทดสอบ logout ต่อ (ใส่ทั้ง `-b` เพื่อส่ง session cookie ปัจจุบันไป และ `-c` เพื่อบันทึก `Set-Cookie`
ตัวใหม่ที่ server ส่งกลับมาหลัง logout ทับไฟล์เดิม):

```bash
$ curl -sS -i -b /tmp/cookies.txt -c /tmp/cookies.txt -X POST http://127.0.0.1:3175/logout
```
```
HTTP/1.1 204 No Content
set-cookie: id=; Path=/; Max-Age=0; Expires=Sat, 27 Sep 2025 00:31:45 GMT
date: Sun, 27 Sep 2026 00:31:45 GMT
```

สังเกต `set-cookie` ที่ได้: ค่า cookie ถูกล้างเป็นค่าว่าง (`id=`) พร้อม `Max-Age=0` และ `Expires` เป็นวันที่
**ในอดีต** (ปีก่อนหน้าวันปัจจุบันเป๊ะ) — นี่คือวิธีมาตรฐานที่ server สั่งให้ browser **ลบ cookie ตัวนี้ทิ้ง
ทันที** (ไม่มี "delete cookie" command แยกใน HTTP spec — ใช้ trick ตั้งวันหมดอายุเป็นอดีตแทน) ทดสอบว่า
session ตายจริงด้วย cookie jar ตัวเดิม (ที่ตอนนี้ถูกทับด้วย cookie ที่หมดอายุแล้ว):

```bash
$ curl -sS -i -b /tmp/cookies.txt http://127.0.0.1:3175/me
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:31:45 GMT
```

`401` ตามคาด — flow ทั้งหมดทำงานถูกต้องตั้งแต่ login จนถึง logout ครบวงจร (transcript ทั้งหมดข้างบนมาจากการ
รัน server และยิง `curl` จริงในสภาพแวดล้อมทดสอบของบทนี้ ไม่ใช่ผลลัพธ์ที่แต่งขึ้น)

### 75.6 Session Fixation และ `cycle_id()`

สังเกตในโค้ด `login` ข้างบนว่ามีบรรทัด `session.cycle_id().await.unwrap();` หลัง `insert` ข้อมูล user —
นี่ไม่ใช่โค้ดประดับ แต่ป้องกันการโจมตีที่เรียกว่า **session fixation** ซึ่งทำงานแบบนี้:

1. Attacker เข้าเว็บไซต์เป้าหมายเอง ได้ session ID ที่ยัง "ไม่ login" มา (เช่น `sess-AAA`)
2. Attacker หลอกให้ victim เปิดลิงก์ที่ฝัง session ID `sess-AAA` นี้เข้าไป (ถ้าเว็บไซต์รับ session ID จาก
   URL parameter ได้ — ช่องโหว่แบบเก่าที่เว็บสมัยใหม่ป้องกันด้วยการไม่รับ session ID จาก parameter เลย
   ใช้จาก cookie เท่านั้น) หรือถ้าเว็บไซต์มีช่องโหว่ที่ทำให้ set cookie ข้าม subdomain ได้
3. Victim login สำเร็จ **โดยที่ session ID ยังเป็น `sess-AAA` ตัวเดิม** (ถ้าระบบไม่เปลี่ยน ID หลัง login)
4. Attacker ที่รู้ค่า `sess-AAA` อยู่แล้ว **ใช้ ID เดิมนั้นเข้าเว็บไซต์** — ตอนนี้ session `sess-AAA` กลาย
   เป็น session ของ victim ที่ login สำเร็จแล้ว → attacker เข้าถึงบัญชี victim ได้โดยไม่ต้องรู้ password เลย

การป้องกันมาตรฐานคือ: **ทุกครั้งที่ privilege ของ session เปลี่ยนไป (โดยเฉพาะจาก "ยังไม่ login" เป็น
"login สำเร็จแล้ว") ให้สร้าง session ID ใหม่ทันที** (เก็บข้อมูลเดิมไว้ครบ) — `session.cycle_id()` ทำสิ่งนี้
ให้ตรงตัว: ลบ record เก่าออกจาก store, สร้าง ID ใหม่, บันทึกข้อมูลเดิมทั้งหมดเข้ากับ ID ใหม่ แล้วส่ง
`Set-Cookie` ที่มี ID ใหม่กลับไป — ผลคือ session ID ที่ attacker เตรียมไว้ล่วงหน้า (`sess-AAA`) จะ**ตายทันที
หลัง login สำเร็จ** ไม่มีทางใช้ปลอมตัวเป็น victim ได้อีก

จาก transcript ในหัวข้อ 75.5 สังเกตได้ว่าค่า cookie ที่ได้จาก `POST /login` (`id=lyxT0_ED3SHIZ0f2SHRZkQ`)
เป็นค่าที่**สร้างใหม่ตอนนั้นเอง** (เพราะ request `POST /login` ตัวแรกไม่มี cookie แนบมาก่อนเลย เท่ากับว่า
`cycle_id()` เปลี่ยนจาก "ไม่มี ID" เป็น "มี ID ใหม่" — พฤติกรรมเดียวกับตอนมี ID เดิมอยู่แล้วแล้วเปลี่ยนเป็น
ID ใหม่ก็เกิดขึ้นแบบเดียวกันทุกประการ)

### 75.7 CSRF: ทำไม Session/Cookie เสี่ยงในแบบที่ Bearer Token ไม่เสี่ยง

**CSRF (Cross-Site Request Forgery)** คือการโจมตีที่หลอกให้ browser ของ victim (ที่ login เว็บไซต์เป้าหมาย
อยู่แล้ว) ยิง request ที่ **victim ไม่ได้ตั้งใจส่ง** ไปยังเว็บไซต์นั้น โดยอาศัยความจริงที่ว่า **browser
แนบ cookie ไปกับ request โดยอัตโนมัติเสมอ ไม่ว่า request นั้นจะถูก "สั่ง" มาจากที่ไหน**

ลองนึกภาพสถานการณ์จริง: คุณ login เว็บธนาคารออนไลน์อยู่ (มี session cookie ของธนาคารเก็บอยู่ใน browser)
แล้วเปิด tab ใหม่ไปดูเว็บอื่นที่มี attacker แฝง HTML แบบนี้อยู่:

```html
<!-- โค้ด HTML ที่ attacker ฝังไว้ในเว็บไซต์อื่นที่ไม่เกี่ยวข้องกับธนาคารเลย -->
<img src="https://bank.example.com/transfer?to=attacker&amount=100000" style="display:none">
```

ทันทีที่ browser render หน้านี้ มันจะยิง `GET
https://bank.example.com/transfer?to=attacker&amount=100000` โดยอัตโนมัติ (เพื่อโหลดรูป) — และเพราะ
`bank.example.com` เป็น domain ที่คุณ login อยู่ **browser จะแนบ session cookie ของธนาคารไปกับ request
นี้โดยอัตโนมัติด้วย** แม้ว่า request นี้จะถูกสั่งมาจากเว็บไซต์อื่นที่ไม่เกี่ยวข้องเลยก็ตาม — ถ้าเว็บธนาคาร
เขียน endpoint `/transfer` แบบ `GET` (ผิดหลัก REST ตาม Part 61 อยู่แล้ว แต่สมมติว่าทำแบบนี้จริง) และตรวจ
สอบแค่ "session นี้ login อยู่ไหม" โดยไม่มีการป้องกัน CSRF อะไรเลย **เงินจะถูกโอนออกจริงโดยที่คุณไม่ได้กด
อะไรเลย แค่เปิดหน้าเว็บที่มี `<img>` แฝงอยู่**

#### ทำไม bearer token (JWT ใน `Authorization` header) ไม่เสี่ยงแบบนี้

นี่คือความต่างสำคัญที่สุดระหว่างสองระบบ authentication ของบทนี้กับ Part 74: **`<img>` tag, `<form>` ธรรมดา,
หรือ resource loading อัตโนมัติใด ๆ ของ browser ไม่มีทาง "แนบ custom header" เข้าไปได้เลย** — browser แนบ
cookie ให้อัตโนมัติเพราะนั่นคือสิ่งที่ cookie ถูกออกแบบมาให้ทำ แต่ `Authorization: Bearer <jwt>` เป็น
**header ที่ต้องเขียนโค้ด JavaScript สั่งแนบเองอย่างชัดเจน** (เช่นผ่าน `fetch()`/`XMLHttpRequest` ที่กำหนด
header เอง) — เว็บไซต์ของ attacker **ไม่มีทางรู้ค่า JWT ของคุณเลย** (มันถูก `HttpOnly` หรือเก็บใน
memory/`localStorage` ของ origin คนละตัว ตาม same-origin policy) จึงไม่มีทางสั่งให้ browser แนบมันเข้า
request ปลอมได้ — **นี่คือเหตุผลที่ระบบที่ใช้ bearer token ล้วน ๆ (ไม่มี cookie เลย) แทบไม่มีความเสี่ยงจาก
CSRF โดยธรรมชาติ** (แต่ยังเสี่ยง XSS อยู่ถ้าเก็บ token ไม่ดีตามที่หัวข้อ 75.4 อธิบายไว้ — คนละปัญหากับ CSRF)

#### การป้องกัน CSRF สมัยใหม่: `SameSite` cookie attribute

จากหัวข้อ 75.4 ตาราง `SameSite` — ค่า `Lax` หรือ `Strict` แก้ปัญหาสถานการณ์ `<img>` ข้างบนได้ตรงจุดเลย:
`<img>` ที่ฝังอยู่ในเว็บไซต์อื่นคือ request แบบ **cross-site** (มาจาก origin อื่น) และไม่ใช่ top-level
navigation (ไม่ใช่การคลิกลิงก์ไปเปลี่ยนหน้าเว็บทั้งหน้า) — ทั้ง `Lax` และ `Strict` จะสั่ง browser **ไม่ส่ง**
cookie ไปกับ request แบบนี้เลย ทำให้ต่อให้ request ถูกยิงไปจริง เว็บธนาคารจะเห็นแค่ request ที่ไม่มี session
cookie แนบมา → ปฏิเสธด้วย `401` ทันที ไม่มีเงินถูกโอน

นี่คือเหตุผลที่ browser สมัยใหม่ (Chrome ตั้งแต่ปี 2020 เป็นต้นไป) เปลี่ยนค่า default ของ cookie ที่ไม่ระบุ
`SameSite` ให้เป็น `Lax` โดยอัตโนมัติ — **`SameSite` กลายเป็นการป้องกัน CSRF ขั้นพื้นฐานที่ได้มาแทบจะ "ฟรี"
ในยุคปัจจุบัน** เพียงแค่ตั้งค่าให้ถูกต้อง (ซึ่ง `tower-sessions` ตั้ง `Strict` เป็น default ให้อยู่แล้ว —
เข้มกว่าที่จำเป็นเสียอีกสำหรับ use case ทั่วไป)

#### CSRF Token แบบดั้งเดิม — defense-in-depth

ก่อนยุคที่ `SameSite` แพร่หลาย (และในระบบที่จำเป็นต้องใช้ `SameSite=None` เพราะมี legitimate cross-site
use case) การป้องกัน CSRF มาตรฐานคือ **CSRF token** (หรือเรียก "synchronizer token pattern"): server
สร้างค่าสุ่มค่าหนึ่งต่อ session แล้วฝังไว้ใน**ทุกฟอร์ม**ที่จะแก้ไขข้อมูล (เป็น hidden field หรือ header ที่
JavaScript ต้องอ่านมาแนบเอง) — endpoint ที่แก้ไขข้อมูลจะปฏิเสธ request ที่ไม่มี token นี้แนบมา หรือ token
ไม่ตรงกับที่เก็บไว้ใน session

หลักการทำงาน: attacker ที่ฝัง `<img>`/`<form>` ปลอมจากเว็บอื่น **ไม่มีทางรู้ค่า CSRF token ของ victim**
(มันถูกฝังอยู่ในหน้า HTML ที่ต้อง render จริงจาก origin ที่ถูกต้องเท่านั้น same-origin policy ห้าม
JavaScript จากเว็บอื่นอ่านเนื้อหาของหน้าเว็บ cross-origin ได้) จึงส่ง request ปลอมที่มี token ถูกต้องไม่ได้

บทนี้ไม่ implement CSRF token เต็มรูปแบบ (เพราะสำหรับ API ที่ไม่ได้ render HTML form แบบดั้งเดิม —
แบบที่ Axum + JSON API ทั้งบทนี้เป็นอยู่ — `SameSite=Lax`/`Strict` เพียงพอในทางปฏิบัติเกือบทุกกรณี) แต่ควร
**รู้จักไว้ในระดับ awareness**: ถ้าระบบของคุณจำเป็นต้องใช้ `SameSite=None` ด้วยเหตุผลทางธุรกิจ หรือต้อง
รองรับ browser เก่ามาก ๆ ที่ไม่รู้จัก `SameSite` เลย ควรเพิ่ม CSRF token เป็น defense-in-depth อีกชั้น —
crate อย่าง `axum-csrf` หรือการ implement เองด้วย double-submit cookie pattern (เก็บ token สุ่มไว้ทั้งใน
cookie ที่ไม่ใช่ `HttpOnly` และให้ JavaScript อ่านค่านั้นไปแนบเป็น header เอง แล้ว server เทียบสองค่านี้ว่า
ตรงกัน) เป็นจุดเริ่มต้นที่ควรไปศึกษาต่อถ้าระบบต้องการมัน

### 75.8 จาก `MemoryStore` ไปสู่ Multi-Instance: ทำไม Session ไม่ Scale ง่ายเท่า JWT

กลับมาที่ตารางเปรียบเทียบในหัวข้อ 75.2 แถวที่ว่า "Scale แนวนอน" — ตอนนี้มาดูรายละเอียดจริงว่าปัญหานี้หน้าตา
เป็นอย่างไร และทำไมมันสำคัญพอที่ต้องเข้าใจตั้งแต่ตอนออกแบบระบบ ไม่ใช่มาแก้ทีหลัง

สมมติระบบของคุณโตขึ้นจนต้อง deploy หลาย instance พร้อมกัน (เพื่อรองรับ traffic มากขึ้น หรือเพื่อ high
availability — ถ้า instance หนึ่งล่ม อีกตัวยังรับ request ต่อได้) หลัง load balancer หนึ่งตัว:

```
                     ┌──────────────────┐
   request ──────▶   │  Load Balancer   │
                     └─────────┬────────┘
                     ┌─────────┼────────┐
                     ▼         ▼        ▼
              ┌───────────┐ ┌───────────┐ ┌───────────┐
              │Instance A │ │Instance B │ │Instance C │
              │MemoryStore│ │MemoryStore│ │MemoryStore│  <- แต่ละตัวมี HashMap ของตัวเอง แยกกันสิ้นเชิง
              └───────────┘ └───────────┘ └───────────┘
```

ถ้ายังใช้ `MemoryStore` ผู้ใช้คนหนึ่ง login ที่ instance A สำเร็จ (session ถูกสร้างใน `HashMap` ของ A
เท่านั้น) แต่ request ถัดไปของผู้ใช้คนเดียวกันอาจถูก load balancer ส่งไปยัง instance B หรือ C (ตามอัลกอริทึม
round-robin/least-connection ที่ load balancer ใช้ — ไม่มีการันตีว่า request ต่อเนื่องจากคนเดียวกันจะไปตก
ที่ instance เดิมเสมอ) — instance B/C **ไม่มี session นั้นอยู่ใน `HashMap` ของตัวเองเลย** เพราะแต่ละ
instance รันเป็น process แยกกันคนละเครื่อง (หรือคนละ container) ไม่ได้แชร์ memory กัน → ผู้ใช้จะเห็นเหมือน
"ถูก logout" แบบสุ่ม ๆ ทุกครั้งที่ request ไปตกที่ instance ที่ไม่รู้จัก session ของตัวเอง — เป็นบั๊กที่
debug ยากมากเพราะเกิด "บางครั้ง" ไม่ใช่ทุกครั้ง (ขึ้นกับ instance ที่ load balancer เลือกให้)

เทียบกับ JWT: instance ไหนก็ตรวจสอบ signature ของ JWT ได้ด้วยตัวเอง (ตราบใดที่ทุก instance มี secret/public
key เดียวกัน) **ไม่ต้องมี "แหล่งข้อมูลกลาง" ที่ทุก instance ต้องคุยด้วย** — นี่คือข้อได้เปรียบด้าน scale
ที่หัวข้อ 75.2 พูดถึง

**ทางแก้ปัญหานี้มีสองแนวทางหลัก:**

1. **Sticky sessions** (session affinity) — ตั้งค่า load balancer ให้ "จำ" ว่า client คนนี้เคยถูกส่งไปที่
   instance ไหน แล้วส่ง request ต่อ ๆ ไปของ client คนเดียวกันไปที่ instance เดิมเสมอ (มักทำผ่าน cookie
   ที่ load balancer เองเป็นคนตั้ง แยกจาก session cookie ของแอป) — **นี่คือ workaround ที่ใช้ได้ แต่มีข้อ
   เสียสำคัญ**: เสีย load balancing ที่แท้จริงไป (instance บางตัวอาจรับ traffic หนักกว่าตัวอื่นมากถ้า
   client กลุ่มหนึ่ง "ติด" อยู่กับมัน) และถ้า instance นั้นล่มไปเลย **session ทั้งหมดที่ผูกกับมันหายไปด้วย**
   (ขัดกับเป้าหมายเรื่อง high availability ที่อยากมีหลาย instance ไว้ตั้งแต่แรก)
2. **Shared session store** — ให้ทุก instance เชื่อมต่อไปยัง**ที่เก็บข้อมูลกลางที่แชร์กัน**แทน `HashMap`
   ในหน่วยความจำของตัวเอง — instance ไหนก็ตามที่รับ request เข้ามาจะ query ไปยังที่เก็บกลางตัวเดียวกันเสมอ
   ไม่ว่า session นั้นจะถูกสร้างจาก instance ไหนก็ตาม **นี่คือทางแก้ที่ถูกต้องและใช้กันจริงใน production**

`tower-sessions` ออกแบบมาให้สลับไปใช้แนวทางที่ 2 ได้ง่ายมาก เพราะ **`SessionStore` เป็น trait** (ดูใน
หัวข้อ 75.3 ที่ `MemoryStore` implement `SessionStore` อยู่แล้ว) — สลับ store แค่เปลี่ยน type ที่ผ่านเข้า
`SessionManagerLayer::new(...)` โค้ด handler ทั้งหมด (`session.get()`/`session.insert()`/ฯลฯ) **ไม่ต้องแก้
แม้แต่บรรทัดเดียว** เพราะ `Session` extractor เป็น abstraction ที่ไม่รู้จักรายละเอียดของ store ที่อยู่
เบื้องหลังเลย (หลักการเดียวกับ trait object ที่ Part 28 สอนไว้)

Store ที่ crate แนะนำไว้ในเอกสารของตัวเองสำหรับ production ที่ใช้กันจริงคือ
**`tower-sessions-redis-store`** (ผ่าน Redis client ชื่อ `fred`) — **Part 83 (Caching ด้วย Redis)** จะสอน
Redis เต็มรูปแบบและกลับมาที่ store ตัวนี้อีกครั้งพร้อมตัวอย่างที่รันได้จริง สำหรับตอนนี้ให้จดจำแค่ภาพรวม:
เปลี่ยนจาก

```rust
let session_store = MemoryStore::default();
```

เป็น (โครงร่างที่ Part 83 จะเติมให้ครบ)

```rust
// ตัวอย่างโครงร่าง -- Part 83 จะสอนการตั้งค่า Redis connection และ error handling เต็มรูปแบบ
// let redis_pool = /* เชื่อมต่อ Redis */;
// let session_store = RedisStore::new(redis_pool);
```

แล้ว `SessionManagerLayer::new(session_store)` ที่เหลือทั้งหมด**เหมือนเดิมทุกบรรทัด** — ตอนนี้ทุก instance
ที่ deploy พร้อมกันจะ query ไปยัง Redis ตัวเดียวกัน (ที่อยู่แยกจาก instance ของแอปเอง มัก deploy เป็น
managed service เช่น AWS ElastiCache หรือ container แยก) session ที่สร้างจาก instance A จึงถูกอ่านเจอจาก
instance B/C ได้ทันที ไม่ว่า load balancer จะส่ง request ไปตกที่ไหนก็ตาม — และเรื่องนี้จะยิ่งสำคัญขึ้นไปอีก
ตอนที่ **Part 81 (Microservices Architecture)** พูดถึงระบบที่มีหลาย service แยกกัน (ไม่ใช่แค่หลาย instance
ของ service เดียว) ที่ทุก service อาจต้องรู้ว่า request หนึ่ง ๆ authenticate มาจากใคร — shared session
store (หรือทางเลือกอื่นอย่าง JWT ที่ Part 74 สอน ที่ไม่ต้องมี store กลางเลย) กลายเป็นตัวเลือกทางสถาปัตยกรรม
ที่ต้องชั่งใจอย่างจริงจังในระดับ microservices

### 75.9 สรุปครึ่งแรกของบท ก่อนไปครึ่งหลัง

ถึงจุดนี้คุณมีระบบ session-based authentication ที่ใช้งานได้จริงครบวงจร: login สร้าง session พร้อม
`cycle_id()` ป้องกัน fixation, protected route อ่านข้อมูลผ่าน `Session` extractor, logout ทำลาย session
ด้วย `flush()`, ตั้งค่า cookie attribute ที่ถูกต้อง (`HttpOnly`/`Secure`/`SameSite`) และเข้าใจ tradeoff
เรื่อง scale ที่ต้องแลกกับความสามารถ revoke ทันที

ครึ่งหลังของบทนี้เปลี่ยนโจทย์ไปคนละแบบ — ไม่ใช่ "จะเก็บ session ของ user ที่ login กับระบบเราเองยังไง"
แต่เป็น **"จะให้ user login ด้วยบัญชีที่มีอยู่แล้วบนบริการอื่น (เช่น GitHub, Google) โดยไม่ต้องสร้าง
username/password ใหม่กับเราได้ยังไง"** — นี่คือโจทย์ของ **OAuth2**

### 75.10 OAuth2 คืออะไรจริง ๆ: Delegated Authorization ไม่ใช่ Authentication

นี่คือความเข้าใจผิดที่พบบ่อยที่สุดเรื่อง OAuth2 — และต้องแก้ให้ถูกตั้งแต่ต้นก่อนเขียนโค้ดสักบรรทัด:
**OAuth2 ไม่ได้ถูกออกแบบมาเพื่อตอบคำถาม "คุณคือใคร" (authentication) เป็นหลัก** มันถูกออกแบบมาเพื่อตอบคำถาม
**"แอปนี้ได้รับอนุญาตให้เข้าถึงข้อมูล/ทำอะไรแทนคุณบนบริการอื่นได้แค่ไหน" (delegated authorization)**

ตัวอย่างที่ตรงประเด็นที่สุด: เว็บแอปจัดการรูปภาพตัวหนึ่งอยากเข้าถึง **Google Photos** ของคุณเพื่อ backup
รูปให้อัตโนมัติ — สิ่งที่เกิดขึ้นคือ: คุณ (resource owner) อนุญาตให้เว็บแอปนั้น (client) เข้าถึง**เฉพาะ
รูปภาพ**ของคุณบน Google (resource server) **โดยไม่ต้องให้ username/password ของ Google กับเว็บแอปนั้น
เลยแม้แต่นิดเดียว** — เว็บแอปได้รับ **access token** ที่จำกัดสิทธิ์เฉพาะสิ่งที่คุณอนุญาต (อ่านรูป — ไม่ใช่
สิทธิ์เต็มบัญชี Google เช่นอ่าน Gmail หรือเปลี่ยน password) และคุณสามารถ**เพิกถอน**สิทธิ์นี้ได้ทุกเมื่อจาก
หน้า setting ของ Google โดยไม่กระทบ password หลักของบัญชีเลย

สังเกตว่าโจทย์นี้**ไม่เกี่ยวกับ "login" เข้าเว็บแอปจัดการรูปเลย** — มันเกี่ยวกับ "การมอบสิทธิ์" (authorization
delegation) ล้วน ๆ

#### แล้วทำไมคนถึงใช้ OAuth2 ทำ "Sign in with X" กัน

เหตุผลคือ: ถ้าแอปของคุณขอสิทธิ์ **"อ่านข้อมูลโปรไฟล์พื้นฐาน"** (เช่น ชื่อ, อีเมล, รูป avatar) ผ่าน OAuth2
แทนที่จะขอสิทธิ์เข้าถึงข้อมูลจริงจัง ๆ อย่างรูปภาพ/ไฟล์ — ผลลัพธ์ที่ได้ (คุณรู้ว่า "คนที่กำลังคุยกับแอปตอนนี้
คือเจ้าของบัญชี GitHub ชื่อ X จริง เพราะ GitHub เพิ่งยืนยันตัวตนเขาให้") ก็ใช้**แทน**การ authenticate ได้ใน
ทางปฏิบัติ — นี่คือที่มาของปุ่ม "Sign in with Google/GitHub/Facebook" ที่เกลื่อนอินเทอร์เน็ต: **มันคือการใช้
กลไก delegated authorization ของ OAuth2 เพื่อ "ยืม" การพิสูจน์ตัวตนที่ provider ทำไว้แล้วมาใช้ ไม่ใช่กลไก
authentication ที่ OAuth2 ออกแบบมาให้ตั้งแต่ต้น** (OAuth2 spec เองก็เขียนไว้ตรง ๆ ว่ามันคือ "authorization
framework" ไม่ใช่ "authentication protocol")

การใช้แบบนี้ **ใช้งานได้จริงและปลอดภัยพอ** สำหรับกรณีทั่วไป แต่มีช่องโหว่เชิงแนวคิดที่นักพัฒนาที่ implement
"Sign in with X" ด้วยตัวเองแบบไม่รอบคอบมักพลาด (เช่น การไม่ตรวจสอบว่า access token ที่ได้มาจริง ๆ ถูกออกให้
กับ **client ID ของแอปคุณเอง** — ถ้าไม่ตรวจ อาจเกิดปัญหาที่ token ที่ออกให้แอปอื่นถูกนำมาใช้ปลอมตัวกับแอปคุณ
ได้ ในบางสถานการณ์ที่ provider ออกแบบไม่รอบคอบ) — เพื่อแก้ปัญหาความไม่สมบูรณ์นี้อย่างเป็นระบบ จึงเกิด
มาตรฐานใหม่ที่สร้างทับ OAuth2 โดยเฉพาะ

#### OpenID Connect (OIDC): ชั้นที่ทำให้ "Login" เป็นมาตรฐานจริง ๆ

**OpenID Connect (OIDC)** คือ protocol ที่สร้างทับ OAuth2 (ใช้ flow เดียวกันทุกขั้นตอนที่จะสอนในหัวข้อถัดไป)
แล้วเพิ่มสามอย่างที่ OAuth2 ดั้งเดิมไม่มี:

1. **`id_token`** — token รูปแบบ **JWT** (ใช่ — เชื่อมกับ Part 74 ตรง ๆ) ที่ provider **เซ็นเอง** และมี
   claim มาตรฐาน (`sub` = user ID ที่ provider นั้น, `iss` = ผู้ออก token, `aud` = client ID ที่ token นี้
   ออกให้, `exp`, `iat`, `nonce`) — แอปของคุณ**ตรวจสอบ signature ของ `id_token` นี้ด้วยเทคนิคเดียวกับที่
   Part 74 สอนไว้ทุกประการ** (ดึง public key ของ provider มา verify) แทนที่จะต้องเชื่อ response ดิบ ๆ จาก
   API เฉย ๆ — นี่คือกลไกที่แก้ปัญหา "ไม่ได้ตรวจว่า token ออกให้ client ID ของเราจริงไหม" อย่างเป็นระบบ
   (ตรวจ `aud` claim ตรง ๆ)
2. **scope `openid` มาตรฐาน** — ขอ scope นี้แล้วจะได้ `id_token` กลับมาด้วย (ไม่ใช่แค่ access token
   ธรรมดา) พร้อม scope ย่อยมาตรฐาน `profile`/`email` ที่ provider ทุกตัวที่รองรับ OIDC ตีความเหมือนกัน
3. **`/userinfo` endpoint มาตรฐาน** — endpoint กลางที่ provider ทุกตัวมีชื่อ/รูปแบบ response
   ใกล้เคียงกัน (ต่าง OAuth2 ดั้งเดิมที่แต่ละ provider มี endpoint โปรไฟล์ของตัวเองที่หน้าตาต่างกันหมด)

**Google, Microsoft, Auth0 รองรับ OIDC เต็มรูปแบบ — แต่ GitHub (ตัวที่บทนี้จะใช้ implement จริง) ไม่รองรับ
OIDC** GitHub OAuth App ให้แค่ OAuth2 ดั้งเดิมล้วน ๆ ไม่มี `id_token` ให้เลย ต้องเรียก REST API endpoint
ของ GitHub เอง (`GET /user`) ด้วย access token ที่ได้มาเพื่อดึงข้อมูลโปรไฟล์เอาเอง (นี่คือสิ่งที่โค้ดในหัวข้อ
75.18 จะทำ) — **ถ้าจะทำ "Sign in with Google" ด้วย OIDC จริง ๆ** (นอกเหนือจากบทนี้) ขั้นตอนจะคล้ายกันมาก
แต่ต่างตรงที่ท้ายสุดคุณจะได้ `id_token` (JWT) กลับมาโดยตรงจาก token endpoint แทนที่จะต้องยิง API เพิ่มอีก
รอบ — และการ verify signature ของ `id_token` นั้นคือการเอาความรู้เรื่อง JWT verification จาก Part 74 มาใช้
ตรง ๆ (crate อย่าง `openidconnect` สร้างทับ `oauth2` crate ตัวเดียวกันที่บทนี้ใช้ เพื่อจัดการส่วน OIDC
เพิ่มเติมนี้โดยเฉพาะ — เป็นหัวข้อขั้นสูงกว่าที่อยู่นอกเหนือ scope ของบทนี้)

**สรุปให้จำง่าย**: OAuth2 = กลไกมอบสิทธิ์ (authorization) | OIDC = OAuth2 + มาตรฐานสำหรับ "login" (identity)
บนฐาน JWT ที่คุ้นเคยจาก Part 74 | GitHub OAuth App ในบทนี้ = OAuth2 ดั้งเดิมล้วน ๆ (ไม่ใช่ OIDC) ที่เราใช้
เพื่อ "ยืม" การยืนยันตัวตนของ GitHub มาใช้เอง โดยไม่มี `id_token` ให้ — ต้องเรียก API โปรไฟล์เองเสมอ

### 75.11 บทบาทในระบบ OAuth2 (คำศัพท์ที่ต้องรู้)

ก่อน implement เต็มรูปแบบ มาทำความเข้าใจศัพท์เฉพาะสี่ตัวที่ RFC 6749 (มาตรฐาน OAuth2) ใช้ตลอดทั้งเอกสาร
เพราะจะปรากฏซ้ำ ๆ ในโค้ดและเอกสารของ provider ทุกเจ้า:

| บทบาท | คือใคร (ในบริบทของบทนี้) |
|---|---|
| **Resource Owner** | ผู้ใช้ — คนที่เป็นเจ้าของบัญชี GitHub และข้อมูลโปรไฟล์ของตัวเอง |
| **Client** | แอปของคุณ — เว็บแอป Axum ที่ต้องการเข้าถึงข้อมูลโปรไฟล์ GitHub ของผู้ใช้ |
| **Authorization Server** | GitHub — เจ้าของ endpoint `/login/oauth/authorize` และ `/login/oauth/access_token` ที่ออก token ให้ |
| **Resource Server** | GitHub เช่นกัน (ในกรณีนี้ authorization server กับ resource server เป็นบริการเดียวกัน — ไม่จำเป็นเสมอไป บางระบบแยกกันคนละ service) — เจ้าของ endpoint `api.github.com/user` ที่ข้อมูลจริงอยู่ |

### 75.12 Authorization Code Flow ทีละ Step พร้อมเหตุผล

มาถึงหัวใจของครึ่งหลังของบทนี้ — **Authorization Code Flow** (หรือเรียกเต็ม ๆ ว่า "Authorization Code
Grant") คือ flow ที่ **ถูกต้องและปลอดภัยที่สุดสำหรับเว็บแอป server-side** (แอปที่มี backend ของตัวเองที่
เก็บ secret ได้อย่างปลอดภัย — ตรงกับสถานการณ์ของ Axum app ที่บทนี้กำลังสร้าง)

```
ผู้ใช้ (browser)          แอปของคุณ (client)          GitHub (authorization server)
      │                          │                              │
      │  1. คลิก "Sign in       │                              │
      │     with GitHub"        │                              │
      ├─────────────────────────▶                              │
      │                          │  2. redirect ไปหน้า consent  │
      │                          │     ของ GitHub               │
      │◀─────────────────────────┤                              │
      │  3. browser ถูก redirect ไปที่ GitHub โดยตรง            │
      ├──────────────────────────────────────────────────────────▶
      │                          │      4. ผู้ใช้กด "Authorize" │
      │                          │         บนหน้า GitHub เอง    │
      │◀──────────────────────────────────────────────────────────┤
      │  5. GitHub redirect กลับมาที่ /callback ของแอป          │
      │     พร้อม authorization code (query parameter ?code=...) │
      ├─────────────────────────▶                              │
      │                          │  6. แอป "แลก" code เป็น     │
      │                          │     access token             │
      │                          │  ── server-to-server ──▶     │
      │                          │  ◀── access token กลับมา ── │
      │                          │  7. แอปใช้ access token      │
      │                          │     เรียก GitHub API         │
      │                          │  ── server-to-server ──▶     │
      │                          │  ◀── ข้อมูลโปรไฟล์ ──────── │
      │  8. แอปสร้าง local       │                              │
      │     session ของตัวเอง   │                              │
      │◀─────────────────────────┤                              │
```

อธิบายทีละ step และเหตุผลที่มันต้องเป็นแบบนี้:

1. **ผู้ใช้กด "Sign in with GitHub"** — เป็นแค่ลิงก์/ปุ่มธรรมดาที่ชี้ไปที่ route ของแอปคุณเอง เช่น
   `GET /login/github`
2. **แอปสร้าง authorization URL** — URL ที่ชี้ไปที่ GitHub พร้อม query parameter สำคัญ (`client_id`,
   `redirect_uri`, `scope`, `state`, และถ้าใช้ PKCE — `code_challenge`/`code_challenge_method`) แล้ว
   redirect ผู้ใช้ไปที่ URL นั้น (โค้ดในหัวข้อ 75.14-75.16 จะสร้าง URL นี้จริง)
3. **browser ไปที่ GitHub โดยตรง** — จุดสำคัญ: **แอปของคุณไม่เห็นสิ่งที่เกิดขึ้นบนหน้า GitHub เลย** ผู้ใช้
   กำลังคุยกับ GitHub โดยตรงผ่าน browser ของตัวเอง (URL bar แสดง `github.com` จริง ๆ ไม่ใช่เว็บของคุณ) —
   นี่คือจุดที่ผู้ใช้ **เห็นชัดว่ากำลังให้สิทธิ์กับ GitHub ไม่ใช่กับแอปของคุณ** ป้องกัน phishing ในระดับ
   หนึ่ง (ผู้ใช้ที่สังเกต URL bar จะรู้ว่านี่คือหน้า GitHub จริง ไม่ใช่หน้าปลอมที่แอปมุ่งร้ายสร้างเลียนแบบ
   หน้า login ของ GitHub)
4. **ผู้ใช้เห็นหน้า consent ของ GitHub** — ระบุชัดว่าแอปชื่ออะไร (ตามที่ตั้งชื่อ OAuth App ไว้) ขอสิทธิ์
   อะไรบ้าง (ตาม scope ที่ขอไปใน step 2) ผู้ใช้กด **"Authorize"** (หรือ "Cancel") — **นี่คือ step เดียวใน
   ทั้ง flow ที่ต้องมีมนุษย์กดจริง ๆ** (สำคัญมากสำหรับหัวข้อ 75.19 เรื่องขอบเขตของการทดสอบในบทนี้)
5. **GitHub redirect กลับมาที่ `redirect_uri` ที่ตั้งไว้ตอนสร้าง OAuth App** พร้อม query parameter
   `?code=xxx&state=yyy` — **`code` นี้เป็นแค่ "ใบเสร็จชั่วคราว" ที่ใช้ครั้งเดียว หมดอายุเร็ว (มักไม่กี่
   นาที) — มันยังไม่ใช่ access token** เหตุผลที่ GitHub ไม่ส่ง access token มาตรงนี้เลยคือ: **URL (รวม
   query parameter) เดินทางผ่านหลายจุดที่ไม่ปลอดภัยเท่า HTTP body** — browser history, `Referer` header ที่
   อาจรั่วไปยัง third-party script บนหน้าถัดไป, server access log ของทุก proxy ที่ request เดินทางผ่าน —
   ถ้า access token (ที่มีอายุยาวกว่าและใช้เข้าถึงข้อมูลจริงได้) หลุดไปอยู่ในที่เหล่านี้ ผลเสียจะรุนแรงกว่า
   `code` ที่ใช้ได้ครั้งเดียวและตายเร็วมาก
6. **แอป "แลก" code เป็น access token — เกิดขึ้น server-to-server เท่านั้น** นี่คือ step ที่สำคัญที่สุดของ
   ทั้ง flow ในมุมความปลอดภัย: แอป (backend ของคุณ ไม่ใช่ browser) ยิง `POST` request ไปที่ GitHub token
   endpoint โดยตรง (ไม่ผ่าน browser ของผู้ใช้เลย) พร้อมแนบ **`client_secret`** ไปด้วยเพื่อพิสูจน์ว่า "นี่คือ
   แอปที่ลงทะเบียนไว้จริง ไม่ใช่ใครก็ได้ที่ขโมย `code` ไปใช้" — **`client_secret` ไม่เคยถูกส่งไปที่ browser
   เลยตลอด flow นี้แม้แต่ครั้งเดียว** มันอยู่ในตัวแปรฝั่ง server เท่านั้น ตรงข้ามกับ Implicit Flow (ที่บทนี้
   จะอธิบายว่าทำไมไม่แนะนำในหัวข้อถัดไป) ที่ไม่มี step นี้เลย — เพราะไม่มี step นี้ Implicit Flow จึง**ไม่มี
   ทางพิสูจน์ตัวตนของ client ได้อย่างน่าเชื่อถือ**
7. **แอปใช้ access token เรียก GitHub API** — เพื่อดึงข้อมูลโปรไฟล์จริง (`GET
   https://api.github.com/user` พร้อม `Authorization: Bearer <access_token>`) — เกิดขึ้น server-to-server
   เช่นกัน
8. **แอปสร้าง local session ของตัวเอง** — นี่คือจุดที่เชื่อมกับครึ่งแรกของบทนี้: หลังได้ข้อมูลโปรไฟล์
   GitHub มาแล้ว แอปจะ upsert user record ในฐานข้อมูลของตัวเอง (ถ้ายังไม่เคยเห็น GitHub user ID นี้มาก่อน
   ก็สร้างใหม่ ถ้าเคยแล้วก็อัปเดต) แล้วสร้าง **session แบบเดียวกับที่หัวข้อ 75.5 สอน** — ผู้ใช้ตอนนี้
   "login" กับแอปของคุณสำเร็จแล้ว ด้วยกลไกเดียวกับที่ใช้กับ traditional login ทุกประการ (หัวข้อ 75.19 จะ
   ทำให้เห็นภาพนี้เต็มรูปแบบ)

#### ทำไมบทนี้ไม่สอน Implicit Flow

**Implicit Flow** เป็น flow เก่าที่ RFC 6749 นิยามไว้เพื่อรองรับแอปที่ **ไม่มี backend เก็บ secret ได้เลย**
(SPA ล้วน ๆ ที่รันแค่ JavaScript ใน browser สมัยก่อนที่ CORS/fetch จะแพร่หลาย) — มันข้าม step 6 ไปเลย โดยให้
GitHub (ในตัวอย่างสมมติ — จริง ๆ GitHub ไม่รองรับ Implicit Flow ด้วยซ้ำ) ส่ง **access token กลับมาตรงใน URL
fragment** (`#access_token=...`) หลัง redirect ทันที ไม่มีการแลก `code` เป็น token แยกขั้นตอนเลย

ปัญหาของมันตรงตามที่ step 5-6 ข้างบนอธิบายไว้แต่**รุนแรงกว่า**: (1) access token ที่มีอายุยาวและสิทธิ์เต็ม
หลุดไปอยู่ในตำแหน่งที่ไม่ปลอดภัย (URL fragment, browser history) โดยตรง ไม่ใช่แค่ `code` ที่ตายเร็ว (2) ไม่มี
การพิสูจน์ตัวตนของ client เลย (ไม่มี `client_secret` ส่งไปที่ไหนในทั้ง flow) — ใครก็ตามที่ดัก URL fragment
ได้ (แม้จะยากกว่า query parameter เพราะ fragment ไม่ถูกส่งไปกับ HTTP request ปกติ แต่ก็ยังหลุดผ่าน
JavaScript ตัวอื่นบนหน้าเดียวกัน, extension ของ browser, หรือ history ได้) เอา token นั้นไปใช้ได้ทันทีโดย
ไม่ต้องพิสูจน์อะไรเพิ่ม

**มาตรฐานอุตสาหกรรมปัจจุบัน (รวมถึง OAuth 2.1 draft ที่รวบรวม best practice ทั้งหมดของ OAuth2 เข้าด้วยกัน)
แนะนำให้เลิกใช้ Implicit Flow โดยสิ้นเชิง** — แม้แต่ SPA ล้วน ๆ ที่ไม่มี backend ก็ควรใช้ **Authorization
Code Flow + PKCE** แทน (PKCE คือกลไกที่หัวข้อ 75.15 จะอธิบาย ที่ทำให้ Authorization Code Flow ใช้ได้อย่าง
ปลอดภัยแม้ไม่มี `client_secret` เลยก็ตาม) — บทนี้จึงสอนแค่ Authorization Code Flow (+ PKCE) เท่านั้น ตรงกับ
สิ่งที่ทุก provider สมัยใหม่ (GitHub, Google, Microsoft) แนะนำให้ใช้เป็นค่าเริ่มต้น

### 75.13 เตรียม GitHub OAuth App จริง

เพื่อทดสอบโค้ดในหัวข้อถัดไปกับ GitHub จริง คุณต้องสร้าง **GitHub OAuth App** ของตัวเองก่อน (ฟรี ไม่มีค่าใช้
จ่าย ใช้ได้ทันทีที่มีบัญชี GitHub) — ขั้นตอน:

1. เข้า **https://github.com/settings/developers** (หรือ Settings → Developer settings → OAuth Apps)
2. กด **"New OAuth App"**
3. กรอกฟอร์ม:
   - **Application name**: ชื่ออะไรก็ได้ที่อธิบายแอปของคุณ (เช่น "My Rust Course Demo")
   - **Homepage URL**: `http://127.0.0.1:3178` (ตรงกับ address ที่เซิร์ฟเวอร์ dev ของบทนี้รันอยู่)
   - **Authorization callback URL**: `http://127.0.0.1:3178/callback` — **ต้องตรงกับ `redirect_uri` ที่
     โค้ดใช้เป๊ะทุกตัวอักษร** (รวม path, port, http/https) — GitHub ปฏิเสธ callback ที่ไม่ตรงกับค่านี้
     ทันที (นี่คือกลไกป้องกันสำคัญอีกชั้นหนึ่งของ OAuth2: แม้ `code` จะหลุดไปอยู่ในมือของบุคคลอื่นก็
     เอาไปแลก token ไม่ได้ถ้าไม่สามารถควบคุม callback URL ที่ลงทะเบียนไว้ได้)
4. กด **"Register application"**
5. หน้าที่ได้จะแสดง **Client ID** ทันที และมีปุ่ม **"Generate a new client secret"** — กดเพื่อได้
   **Client Secret** (แสดงให้เห็น**ครั้งเดียว** ต้อง copy เก็บไว้ทันที)

เก็บสองค่านี้ไว้เป็น environment variable (**ห้าม commit เข้า git เด็ดขาด** ตามกฎที่ style guide ของ
หลักสูตรนี้เตือนไว้เรื่องความลับ):

```bash
export GITHUB_CLIENT_ID="ค่า Client ID ที่ได้จริง"
export GITHUB_CLIENT_SECRET="ค่า Client Secret ที่ได้จริง"
```

> **หมายเหตุความโปร่งใสเรื่องการทดสอบ**: ขั้นตอนที่ 4 ของ flow ในหัวข้อ 75.12 (ผู้ใช้กด "Authorize" บนหน้า
> เว็บ GitHub จริง) **ต้องมีมนุษย์เปิด browser จริงแล้วคลิกเอง** — ไม่มีทาง automate ขั้นตอนนี้ได้ (และไม่
> ควร automate ด้วย เพราะเป็นกลไกความปลอดภัยที่ตั้งใจให้ต้องมีการยืนยันจากมนุษย์) สภาพแวดล้อมที่ใช้เขียนและ
> ตรวจสอบบทนี้**ไม่มี interactive browser** ให้ทำขั้นตอนนี้ได้ ดังนั้นส่วนที่ต้องพึ่งการคลิก "Authorize"
> จริง (ได้ `code` จริงจาก GitHub) **ไม่ได้ถูกทดสอบแบบ end-to-end ในการเขียนบทนี้** — หัวข้อ 75.17 และ
> 75.19 จะบอกไว้อย่างชัดเจนทุกจุดว่าอะไรถูกทดสอบจริง (รันเซิร์ฟเวอร์จริง ยิง `curl` จริง) กับอะไรที่ตรวจสอบ
> แค่ระดับ "โค้ด compile ผ่านและมี type/logic ถูกต้อง" (structural verification) — **คุณเองเมื่อมี
> Client ID/Secret จริงและเปิด browser ได้ ควรทดสอบ flow เต็มรูปแบบด้วยตัวเองเพื่อยืนยันว่าทำงานจริงกับ
> GitHub จริง** โค้ดในบทนี้ถูกเขียนให้ตรงตาม API จริงของ `oauth2` crate 5.0.0 ทุกประการ (ตรวจสอบจาก
> source code และตัวอย่างที่ crate เผยแพร่เองใน `examples/github_async.rs`) แต่การยืนยัน end-to-end
> เต็มรูปแบบเป็นสิ่งที่ผู้อ่านต้องทำเองในเครื่องของตัวเอง

### 75.14 ใช้ crate `oauth2`: ตั้งค่า `BasicClient`

crate ที่ใช้กันจริงและ maintain อยู่สำหรับ OAuth2 ใน Rust คือ **`oauth2`** (โดยผู้เขียนกลุ่มเดียวกับที่ดูแล
`rust-oauth2` organization) — เวอร์ชันล่าสุดจาก crates.io ณ วันที่เขียนบทนี้คือ **`5.0.0`** (เวอร์ชันนี้
เปลี่ยน API ไปมากจากเวอร์ชัน 4.x เก่า ที่บทความ/tutorial เก่า ๆ บนอินเทอร์เน็ตอาจยังสอนอยู่ — บทนี้สอนตาม
API จริงของเวอร์ชัน 5.0.0 ที่ตรวจสอบจาก source code ของ crate โดยตรง)

เพิ่ม dependency:

```toml
[dependencies]
oauth2 = "5.0.0"
```

สังเกตว่า**ไม่ต้องเพิ่ม `reqwest` เป็น dependency แยกของโปรเจกต์คุณเอง** — `oauth2` เปิด feature
`reqwest` (พร้อม `rustls-tls`) เป็น **default feature** อยู่แล้ว และ **re-export ทั้ง crate `reqwest`
ออกมาผ่าน `oauth2::reqwest`** ให้ใช้ตรง ๆ (`pub use ::reqwest;` ใน `lib.rs` ของ crate) — นี่ไม่ใช่แค่ความ
สะดวก แต่เป็นการเลี่ยงปัญหาจริงที่จะเกิดถ้าคุณเพิ่ม `reqwest` เวอร์ชันอื่นเข้ามาเอง: `oauth2` implement
trait `AsyncHttpClient` ให้กับ `reqwest::Client` **ของเวอร์ชันที่มันพึ่งพาภายใน** เท่านั้น — ถ้าโปรเจกต์คุณ
มี `reqwest` อีกเวอร์ชันหนึ่งอยู่ (เช่นเวอร์ชันล่าสุดจาก crates.io ที่อาจเป็นเลขเวอร์ชันสูงกว่าที่ `oauth2`
พึ่งพา) **สอง type นี้จะเป็นคนละ type กันในสายตาของ compiler โดยสิ้นเชิง** แม้จะมาจาก crate ชื่อเดียวกันก็
ตาม (ระบบ dependency ของ Cargo อนุญาตให้มีหลายเวอร์ชันของ crate เดียวกันอยู่ใน dependency graph พร้อมกันได้
— แต่ type จากคนละเวอร์ชันไม่ compatible กัน) — วิธีที่ปลอดภัยและตรงไปตรงมาที่สุดคือใช้
`oauth2::reqwest::Client` ตัวเดียวสำหรับทั้งการแลก token (ที่ `oauth2` ต้องใช้อยู่แล้ว) **และ**การเรียก
GitHub API ดึงโปรไฟล์ (ที่บทนี้ต้องทำเองในหัวข้อ 75.18) — เลี่ยงการมี `reqwest` สองเวอร์ชันปนกันในโปรเจกต์
เดียวไปเลย

มาสร้าง client:

```rust
use oauth2::basic::BasicClient;
use oauth2::{AuthUrl, ClientId, ClientSecret, RedirectUrl, TokenUrl};

fn build_github_client() {
    let client_id = ClientId::new(
        std::env::var("GITHUB_CLIENT_ID").expect("ต้องตั้ง GITHUB_CLIENT_ID"),
    );
    let client_secret = ClientSecret::new(
        std::env::var("GITHUB_CLIENT_SECRET").expect("ต้องตั้ง GITHUB_CLIENT_SECRET"),
    );
    let auth_url =
        AuthUrl::new("https://github.com/login/oauth/authorize".to_string()).unwrap();
    let token_url =
        TokenUrl::new("https://github.com/login/oauth/access_token".to_string()).unwrap();
    let redirect_url =
        RedirectUrl::new("http://127.0.0.1:3178/callback".to_string()).unwrap();

    let client = BasicClient::new(client_id)
        .set_client_secret(client_secret)
        .set_auth_uri(auth_url)
        .set_token_uri(token_url)
        .set_redirect_uri(redirect_url);

    // client พร้อมใช้สร้าง authorization URL และแลก code เป็น token แล้ว (หัวข้อ 75.15-75.18)
    let _ = client;
}
```

**`ClientId`/`ClientSecret`/`AuthUrl`/`TokenUrl`/`RedirectUrl` เป็น newtype wrapper รอบ `String`** —
ทำไม `oauth2` ไม่ใช้ `String` ธรรมดาตรง ๆ? เหตุผลเดียวกับที่ Part 30/74 อาจพูดถึงเรื่อง **newtype pattern
ป้องกัน "ส่งค่าผิดตำแหน่งโดยไม่ตั้งใจ"**: ถ้าทุกอย่างเป็น `String` เหมือนกัน compiler จะไม่ช่วยจับตอนที่คุณ
สลับส่ง `client_secret` เข้าตำแหน่งที่ควรเป็น `redirect_url` โดยไม่ตั้งใจเลย (ทั้งคู่เป็น `String` ที่
compiler มองว่าเหมือนกันหมด) — แต่ด้วย newtype แต่ละตัว compiler จะปฏิเสธทันทีที่ type ไม่ตรงกัน error
ประเภทนี้เกิดตั้งแต่ compile time ไม่ใช่ runtime

#### Typestate: `EndpointSet`/`EndpointNotSet`

สังเกตว่าโค้ดข้างบนเรียก `.set_auth_uri()`, `.set_token_uri()`, `.set_redirect_uri()` **ต่อกันเป็น chain**
โดยไม่มีการเก็บ `client` เป็นตัวแปรกลางแยกทีละขั้น — นี่ไม่ใช่สไตล์การเขียนที่เลือกเอง แต่เป็นผลจากการ
ออกแบบ API ของ `oauth2` 5.0.0 ที่ใช้ **typestate pattern** (แนวคิดเดียวกับที่ Part 64 แนะนำผ่าน ๆ ไว้ตอน
เขียน custom extractor) — `BasicClient` ที่จริงคือ type alias ที่มี type parameter ซ่อนอยู่ห้าตัว บอกว่า
endpoint ไหน "ถูกตั้งค่าแล้ว" (`EndpointSet`) หรือ "ยังไม่ตั้ง" (`EndpointNotSet`):

```rust
// นิยามจริงจาก source code ของ oauth2 5.0.0 (src/basic.rs)
pub type BasicClient<
    HasAuthUrl = EndpointNotSet,
    HasDeviceAuthUrl = EndpointNotSet,
    HasIntrospectionUrl = EndpointNotSet,
    HasRevocationUrl = EndpointNotSet,
    HasTokenUrl = EndpointNotSet,
> = /* ... */;
```

ทุกครั้งที่เรียก `.set_auth_uri()` มันไม่ได้ mutate `client` เดิม แต่**คืน `client` ตัวใหม่ที่มี type
parameter `HasAuthUrl` เปลี่ยนจาก `EndpointNotSet` เป็น `EndpointSet`** — ผลคือเมธอด `.authorize_url()`
(หัวข้อ 75.15) ที่ **ต้องการ `HasAuthUrl = EndpointSet`** (เพราะมันต้องรู้ auth URL เพื่อสร้าง URL ที่ถูก
ต้อง) จะถูก compiler **บล็อกไม่ให้เรียกได้เลย** ถ้ายังไม่เคยเรียก `.set_auth_uri()` มาก่อน — เป็น compile
error ไม่ใช่ runtime panic เพราะ "ลืมตั้งค่า auth URL" — เอฟเฟกต์นี้พิสูจน์ได้จริงด้วย error message จริง
(ดูกับดักที่พบบ่อยข้อ 6 ท้ายบทนี้ ที่จับ error จริงจากการพยายามเก็บ client ที่ตั้งค่าไม่ครบไว้เป็นตัวแปร
กลางที่มี type annotation ผิด)

### 75.15 PKCE: Proof Key for Code Exchange

**PKCE** (อ่านว่า "pixy" — RFC 7636) แก้ปัญหาที่เหลืออยู่ของ Authorization Code Flow แม้จะมี
`client_secret` แล้วก็ตาม: **authorization code (step 5 ในหัวข้อ 75.12) อาจถูกดักไปได้ระหว่างทาง** ก่อนที่
จะถึง step 6 (การแลก code เป็น token) — สถานการณ์คลาสสิกที่ RFC 7636 ยกตัวอย่างไว้คือแอปมือถือ: ระบบปฏิบัติ
การมือถือหลายระบบอนุญาตให้**หลายแอป**ลงทะเบียนรับ custom URL scheme เดียวกันได้ (เช่น `myapp://callback`)
— ถ้ามีแอปมุ่งร้ายลงทะเบียน scheme เดียวกันไว้ ระบบปฏิบัติการอาจส่ง redirect (ที่มี `code` แนบมา) ไปให้แอป
มุ่งร้ายนั้นเปิดรับแทนแอปที่ถูกต้อง — แอปมุ่งร้ายได้ `code` ไปแล้ว **ถ้าไม่มี PKCE มันเอา `code` นั้นไปแลก
token กับ token endpoint ได้ทันที** (มันรู้ `client_id` อยู่แล้ว เพราะเป็น public information ที่อยู่ใน
authorization URL) **โดยไม่จำเป็นต้องรู้ `client_secret` เลยด้วยซ้ำถ้าเป็น public client ที่ไม่มี secret**

**เหตุผลที่บทนี้บอกว่า "PKCE แนะนำสำหรับ server-side app ด้วย ไม่ใช่แค่ mobile/SPA เหมือนในอดีต"**: แม้แอป
Axum ของคุณจะมี `client_secret` เก็บอย่างปลอดภัยที่ backend (ทำให้สถานการณ์ custom URL scheme ข้างบนไม่เกิด
กับคุณ) แต่ `code` เองยังมีความเสี่ยงหลุดได้จากช่องทางอื่นตามที่หัวข้อ 75.12 step 5 อธิบายไว้ (browser
history, `Referer` header, access log ของ proxy/CDN ตัวกลาง) — **ถ้า `code` หลุดไปจากช่องทางเหล่านี้ โดยไม่
มี PKCE ผู้ที่ได้ `code` ไปสามารถแลกเป็น token ได้ทันที (ถ้ารู้ `client_secret` ด้วย ซึ่งบางสถานการณ์อาจรั่ว
ไปพร้อมกันได้ในระบบที่ misconfigured)** — PKCE เพิ่มการันตีอีกชั้นว่า **คนที่แลก `code` เป็น token ต้องเป็น
คนเดียวกันกับที่**สร้าง**authorization request ตั้งแต่แรก** ไม่ใช่แค่ใครก็ตามที่ครอบครอง `code` ไว้ในมือ —
นี่คือ defense-in-depth ที่ปัจจุบัน (รวมถึง OAuth 2.1 draft) แนะนำให้ใช้เป็นค่าเริ่มต้นเสมอ ไม่ว่าประเภทของ
client จะเป็นแบบไหนก็ตาม

#### วิธีทำงาน

1. **ก่อน redirect ผู้ใช้ไปที่ GitHub** แอปสร้างค่าสุ่มลับชื่อ **`code_verifier`** (สุ่มยาว ๆ) เก็บไว้
   **ที่ server เท่านั้น** (ในบทนี้คือเก็บไว้ใน session ชั่วคราว) แล้วคำนวณ **`code_challenge =
   SHA256(code_verifier)`** (แฮชทางเดียว ย้อนกลับไม่ได้) ส่ง `code_challenge` (ไม่ใช่ `code_verifier`)
   ไปเป็นส่วนหนึ่งของ authorization URL
2. **ตอนแลก `code` เป็น token (step 6)** แอปต้องส่ง **`code_verifier` ตัวจริง** (ที่เก็บไว้ตอนแรก) แนบไป
   ด้วย — GitHub (authorization server) จะคำนวณ `SHA256(code_verifier ที่ได้รับ)` แล้วเทียบกับ
   `code_challenge` ที่รับไว้ตอน step 1 — **ต้องตรงกันเท่านั้นถึงจะออก token ให้**

ผลคือ: ผู้ที่ได้ `code` ไปจากช่องทางที่หัวข้อ 75.12 กังวล (browser history, log) **ไม่มี `code_verifier`
ตัวจริง** (มันไม่เคยถูกส่งไปที่ใดนอกจาก server ของแอปคุณเลย ไม่ปรากฏใน URL หรือที่ไหนที่หลุดง่ายเลย) จึงเอา
`code` ไปแลก token ไม่ได้แม้จะมี `code` อยู่ในมือจริงก็ตาม

#### Implement ด้วย `oauth2` crate

`oauth2` 5.0.0 มี PKCE support เต็มรูปแบบในตัว (ตรวจสอบจาก source code จริง — type `PkceCodeChallenge`/
`PkceCodeVerifier` อยู่ใน `oauth2::types`) ใช้แค่สองเมธอด:

```rust
use oauth2::PkceCodeChallenge;

// (1) ก่อน redirect: สร้างคู่ challenge/verifier ด้วย SHA-256 (ตาม RFC 7636 แนะนำ -- มี
//     .new_random_plain() ให้ด้วยสำหรับ provider เก่าที่ไม่รองรับ S256 แต่ GitHub รองรับ S256 เต็มรูปแบบ
//     จึงใช้ตัวนี้เสมอ)
let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();
// pkce_challenge: ส่งไปกับ authorization URL (ปลอดภัยที่จะเปิดเผย -- เป็นแฮชทางเดียว)
// pkce_verifier: เก็บไว้ที่ server เท่านั้น (ในบทนี้คือ session) รอส่งตอนแลก token
```

`pkce_verifier.secret()` คืนค่า `&String` ตัวจริง — ใน section ถัดไปจะเห็นว่าค่านี้ต้องเก็บไว้ใน session
ระหว่าง step 2 (สร้าง URL) กับ step 6 (แลก token) ของ flow ในหัวข้อ 75.12 ซึ่งเป็นสอง HTTP request คนละ
ครั้งกัน (`GET /login/github` แล้วต่อมาคือ `GET /callback`) — session (จากครึ่งแรกของบทนี้) คือกลไกที่
เหมาะที่สุดสำหรับการ "จำ" ค่านี้ข้าม request สองครั้งนี้พอดี

### 75.16 State Parameter: CSRF Protection สำหรับ OAuth2 Redirect Dance

> **อย่าสับสน**: คำว่า "state" ในหัวข้อนี้เป็นคำศัพท์ของ OAuth2 spec เอง (query parameter ชื่อ `state`) —
> **คนละเรื่องกับ `State<AppState>` extractor ของ Axum ที่ Part 64 สอนไว้โดยสิ้นเชิง** ถึงจะบังเอิญใช้คำ
> เดียวกัน โค้ดในหัวข้อนี้จะเรียกตัวแปรที่เกี่ยวกับ OAuth2 `state` ว่า `csrf_token`/`csrf_state` เพื่อไม่ให้
> ปนกับ `State<T>` ของ Axum ในสายตา — แต่เอกสารและ query parameter จริงจะยังใช้ชื่อ `state` ตามมาตรฐาน

ปัญหาที่ `state` parameter แก้คือ **CSRF ที่เกิดกับ OAuth2 redirect dance เอง** (คนละปัญหากับ CSRF ของ
session cookie ในหัวข้อ 75.7 แต่ใช้แนวคิดป้องกันคล้ายกัน) — ปัญหานี้เรียกในเอกสาร RFC 6749 Section 10.12
ว่า "Cross-Site Request Forgery" ของ callback endpoint โดยเฉพาะ กลไกคือ: `code` ที่ callback URL ได้รับ
เป็นเพียง query parameter ธรรมดา ไม่ได้ผูกกับ browser session ใด ๆ โดยตัวมันเอง ดังนั้นถ้าไม่มีการตรวจสอบ
เพิ่มเติม ใครก็ตามที่ครอบครอง `code` ที่ยัง valid อยู่ (เช่น จากการเริ่ม flow ด้วยบัญชีของตัวเอง) สามารถส่ง
`code` นั้นให้ browser อื่นเปิด callback URL แทนได้ (`https://yourapp.com/callback?code=<code>`) และ
แอปฝั่ง server จะแลก `code` นั้นเป็น token ของบัญชีที่ผูกกับ `code` นั้นจริง แล้วสร้าง local session ให้กับ
browser ที่เปิด URL — ผลคือ local session ของ browser หนึ่งถูกผูกเข้ากับบัญชีภายนอกที่ตัวมันเองไม่ได้เป็น
คนเริ่ม flow ซึ่งขัดกับสมมติฐานพื้นฐานของระบบ auth ทุกระบบที่ว่า "หนึ่ง session ต้องผูกกับหนึ่ง flow ที่
ตัวมันเองเริ่มเท่านั้น" — `state` parameter แก้ปัญหานี้ตรงจุด: server สุ่มค่าที่คาดเดาไม่ได้ก่อน redirect
ครั้งแรก เก็บไว้ใน session ของ browser ที่เริ่ม flow เอง แล้วตรวจสอบว่าค่า `state` ที่ callback ได้รับตรงกับ
ค่าที่เก็บไว้ในเวลาที่ redirect เกิดขึ้นหรือไม่ — ถ้าไม่ตรง (หรือไม่มีค่าเก็บไว้เลย เพราะ session ของ browser
ที่เปิด callback ไม่ใช่ session เดียวกับที่เริ่ม flow) แอปต้องปฏิเสธทันทีโดยไม่แลก `code` เป็น token เด็ดขาด

### 75.17 พิสูจน์ Authorization URL Generation ด้วยการรันโค้ดจริง

หัวข้อ 75.14-75.16 ประกอบโค้ดสามชิ้น (`BasicClient`, PKCE challenge/verifier, CSRF `state`) เข้าด้วยกันแล้ว
ทีละส่วน — หัวข้อนี้เอาสามชิ้นนั้นมาต่อกันเป็นก้อนเดียวจริง แล้ว**รันจริง**เพื่อตรวจสอบว่า URL ที่โค้ดสร้าง
ออกมามีโครงสร้างถูกต้องตาม OAuth2 Authorization Code flow (RFC 6749) + PKCE (RFC 7636) ทุกจุดจริงหรือไม่ —
ไม่ใช่แค่ "อ่านโค้ดแล้วดูน่าจะถูก" แต่คือ compile ผ่านจริง รันจริง แล้วตรวจ query parameter ที่ได้ทีละตัว

```rust
// ตรวจสอบ "เชิงโครงสร้าง" (structural verification) ของโค้ดสร้าง authorization URL ด้วย oauth2 crate 5.0.0
// -- ไม่มีการเปิด browser จริง ไม่มีการยิง request ไปหา GitHub จริงในไฟล์นี้ แค่พิสูจน์ว่าโค้ด compile ผ่าน
// และ URL ที่สร้างออกมามีโครงสร้างที่ถูกต้องตาม OAuth2 Authorization Code flow + PKCE + state จริง
use oauth2::basic::BasicClient;
use oauth2::{
    AuthUrl, ClientId, ClientSecret, CsrfToken, PkceCodeChallenge, RedirectUrl, Scope, TokenUrl,
};

fn main() {
    // สร้าง BasicClient แบบ inline -- สังเกตว่าฟังก์ชัน .set_auth_uri()/.set_token_uri()/.set_redirect_uri()
    // แต่ละตัวเปลี่ยน "ชนิด" ของ client (ผ่าน typestate parameter EndpointSet/EndpointNotSet ที่ oauth2 5.x
    // ใช้ภายใน -- ดูหัวข้อ 75.14) เพื่อให้ compiler การันตีว่าถ้าเรียก .authorize_url()/.exchange_code() ได้
    // แปลว่า endpoint ที่จำเป็นถูกตั้งค่าไว้ครบแล้วจริง
    let client_id = ClientId::new("demo_client_id_1234567890".to_string());
    let client_secret = ClientSecret::new("demo_client_secret_abcdefghij".to_string());
    let auth_url =
        AuthUrl::new("https://github.com/login/oauth/authorize".to_string()).unwrap();
    let token_url =
        TokenUrl::new("https://github.com/login/oauth/access_token".to_string()).unwrap();
    let redirect_url =
        RedirectUrl::new("http://127.0.0.1:3177/callback".to_string()).unwrap();

    let client = BasicClient::new(client_id)
        .set_client_secret(client_secret)
        .set_auth_uri(auth_url)
        .set_token_uri(token_url)
        .set_redirect_uri(redirect_url);

    // PKCE (หัวข้อ 75.15): สร้างคู่ (challenge, verifier) จริงด้วย SHA-256 -- verifier ต้องเก็บไว้ฝั่ง
    // server เท่านั้น
    let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();

    // state (หัวข้อ 75.16): CSRF token สุ่มจริงสำหรับป้องกัน CSRF บน OAuth2 redirect dance
    let (auth_url, csrf_token) = client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("read:user".to_string()))
        .add_scope(Scope::new("user:email".to_string()))
        .set_pkce_challenge(pkce_challenge)
        .url();

    println!("=== Authorization URL ที่สร้างจริงจากโค้ด ===");
    println!("{auth_url}");
    println!();
    println!("=== CSRF state token (เก็บไว้ใน session ฝั่ง server) ===");
    println!("{}", csrf_token.secret());
    println!();
    println!("=== PKCE code_verifier (เก็บไว้ใน session ฝั่ง server เท่านั้น ห้ามส่งให้ browser) ===");
    println!("{}", pkce_verifier.secret());
    println!();

    // แยกส่วนของ URL ออกมาตรวจสอบทีละ query parameter ว่ามีครบตามที่ RFC 6749 + RFC 7636 กำหนด
    let parsed = oauth2::url::Url::parse(auth_url.as_str()).unwrap();
    println!("=== ตรวจ query parameters ทีละตัว ===");
    for (key, value) in parsed.query_pairs() {
        let shown = if key == "code_challenge" || key == "state" {
            format!("{value} (แสดงเต็มเพื่อตรวจสอบ)")
        } else {
            value.to_string()
        };
        println!("{key} = {shown}");
    }

    assert_eq!(parsed.host_str(), Some("github.com"));
    assert_eq!(parsed.path(), "/login/oauth/authorize");
    let has_code_challenge = parsed.query_pairs().any(|(k, _)| k == "code_challenge");
    let has_challenge_method = parsed
        .query_pairs()
        .any(|(k, v)| k == "code_challenge_method" && v == "S256");
    let has_state = parsed.query_pairs().any(|(k, _)| k == "state");
    let has_response_type_code = parsed
        .query_pairs()
        .any(|(k, v)| k == "response_type" && v == "code");
    let state_matches_token = parsed
        .query_pairs()
        .any(|(k, v)| k == "state" && v == csrf_token.secret().as_str());

    assert!(has_code_challenge);
    assert!(has_challenge_method);
    assert!(has_state);
    assert!(has_response_type_code);
    assert!(state_matches_token);

    println!();
    println!("ผ่านทุก assertion: URL ที่สร้างมี PKCE challenge, CSRF state, และใช้ response_type=code");
    println!("(Authorization Code flow) ไม่ใช่ response_type=token (Implicit flow ที่ไม่แนะนำ) จริง");
}
```

รันจริงในสภาพแวดล้อมที่ใช้ตรวจสอบบทนี้ (`cargo run`) ได้ผลลัพธ์นี้ (คัดลอกมาตรงตัวจาก stdout จริง ไม่มีการ
แต่งเติม):

```
=== Authorization URL ที่สร้างจริงจากโค้ด ===
https://github.com/login/oauth/authorize?response_type=code&client_id=demo_client_id_1234567890&state=6kgAxOoQA3HB9mCIj9PRFQ&code_challenge=1cg5S1tRe4fweuoFoUYV3Fs7QYb5S_Lgv_H1ohxpWys&code_challenge_method=S256&redirect_uri=http%3A%2F%2F127.0.0.1%3A3177%2Fcallback&scope=read%3Auser+user%3Aemail

=== CSRF state token (เก็บไว้ใน session ฝั่ง server) ===
6kgAxOoQA3HB9mCIj9PRFQ

=== PKCE code_verifier (เก็บไว้ใน session ฝั่ง server เท่านั้น ห้ามส่งให้ browser) ===
xwdpne0DVVQLhjfUCT0roRzipCkBg_IrxHjguSnfIPA

=== ตรวจ query parameters ทีละตัว ===
response_type = code
client_id = demo_client_id_1234567890
state = 6kgAxOoQA3HB9mCIj9PRFQ (แสดงเต็มเพื่อตรวจสอบ)
code_challenge = 1cg5S1tRe4fweuoFoUYV3Fs7QYb5S_Lgv_H1ohxpWys (แสดงเต็มเพื่อตรวจสอบ)
code_challenge_method = S256
redirect_uri = http://127.0.0.1:3177/callback
scope = read:user user:email

ผ่านทุก assertion: URL ที่สร้างมี PKCE challenge, CSRF state, และใช้ response_type=code
(Authorization Code flow) ไม่ใช่ response_type=token (Implicit flow ที่ไม่แนะนำ) จริง
```

ตรวจตามรายการที่ query string ต้องมีครบตาม spec: **`response_type=code`** (ยืนยันว่าเป็น Authorization
Code Flow ไม่ใช่ Implicit Flow ตามที่หัวข้อ 75.12 อธิบายไว้ว่าไม่แนะนำ), **`client_id=...`** (ตรงกับค่าที่
ส่งเข้า `ClientId::new`), **`redirect_uri=...`** (ถูก URL-encode ให้ถูกต้องโดย `oauth2`/`url` crate เอง —
`http://127.0.0.1:3177/callback` กลายเป็น `http%3A%2F%2F127.0.0.1%3A3177%2Fcallback`), **`scope=...`**
(สอง scope ที่เพิ่มด้วย `.add_scope()` ถูกต่อกันด้วยเว้นวรรค แล้ว URL-encode เป็น `+` ตามมาตรฐาน),
**`state=...`** (ตรงกับ `csrf_token.secret()` เป๊ะ — พิสูจน์ว่า state ที่ส่งไปกับ URL คือตัวเดียวกันกับที่
ฝั่ง server จะเก็บไว้เทียบตอน callback), **`code_challenge=...`** และ **`code_challenge_method=S256`**
(ยืนยันว่า PKCE ถูกเปิดใช้งานจริงด้วยวิธี SHA-256 ตามที่หัวข้อ 75.15 แนะนำ ไม่ใช่ `plain` method ที่อ่อนกว่า)

#### ขอบเขตของการตรวจสอบนี้ — และสิ่งที่ผู้อ่านต้องทำเองต่อ

สิ่งที่พิสูจน์แล้วจริงในหัวข้อนี้คือ **โค้ดสร้าง authorization URL compile ผ่าน และ URL ที่สร้างออกมามี
โครงสร้างถูกต้องครบทุก query parameter ตามที่ RFC 6749 (Authorization Code) และ RFC 7636 (PKCE) กำหนด** —
นี่คือ**structural verification** (ตรวจโครงสร้าง/รูปแบบของโค้ดและผลลัพธ์) ไม่ใช่การทดสอบ flow เต็มรูปแบบ
แบบ end-to-end เพราะ **flow ของ OAuth2 Authorization Code (หัวข้อ 75.12) มีขั้นตอนหนึ่งที่ต้องมีมนุษย์เปิด
browser จริงแล้วกดปุ่ม "Authorize" บนหน้าเว็บของ GitHub เอง** (step 4 ในหัวข้อ 75.12) — ขั้นตอนนี้เป็น
security control ที่ตั้งใจให้ automate ไม่ได้ (ถ้า automate ได้ ก็แปลว่าใครก็ตามที่ควบคุมโค้ดฝั่ง client
สามารถขอ consent แทนผู้ใช้ได้โดยไม่ต้องมีผู้ใช้ยืนยันจริง ซึ่งขัดกับจุดประสงค์ทั้งหมดของ OAuth2 consent
screen) สภาพแวดล้อมที่ใช้เขียนและตรวจสอบบทนี้ไม่มี interactive browser ให้ทำขั้นตอนนี้ได้ ดังนั้นสิ่งที่
verified จริงในบทนี้คือ: **การสร้าง URL (หัวข้อนี้) และการ handle callback ฝั่ง server (หัวข้อ 75.18-75.19)
compile ผ่าน มี logic ถูกต้อง และในกรณีของ 75.18 ยังไปถึงจุดที่ยิง network request ออกจริงด้วย** — ส่วนที่
**ไม่ได้** verified คือรอบ round-trip เต็มรูปแบบที่ต้องมีมนุษย์กด "Authorize" จริงบน GitHub แล้วได้ `code`
จริงกลับมา

ผู้อ่านที่ต้องการยืนยัน flow เต็มรูปแบบด้วยตัวเอง ทำตามขั้นตอนนี้ (ต่อจาก GitHub OAuth App ที่สร้างไว้แล้ว
ในหัวข้อ 75.13):

1. ตั้งค่า `GITHUB_CLIENT_ID`/`GITHUB_CLIENT_SECRET` เป็น environment variable ตามที่หัวข้อ 75.13 อธิบายไว้
   (ค่าจริงจาก GitHub OAuth App ของตัวเอง ไม่ใช่ค่า `demo_client_id` ที่ใช้ทดสอบใน sandbox)
2. รันเซิร์ฟเวอร์ capstone เต็มรูปแบบจากหัวข้อ 75.19 ด้วย `cargo run --bin capstone` (หรือชื่อ binary ที่
   ตั้งไว้ในโปรเจกต์ของคุณ)
3. เปิด browser จริง (Chrome/Firefox/Safari อะไรก็ได้) ไปที่ `http://127.0.0.1:3178/login/github`
4. browser จะถูก redirect ไปหน้า consent screen จริงของ GitHub (URL ที่ได้จะมีรูปแบบเหมือนที่หัวข้อนี้
   พิสูจน์แล้วว่าถูกต้องทุกประการ) — กด **"Authorize"**
5. GitHub จะ redirect กลับมาที่ `http://127.0.0.1:3178/callback?code=...&state=...` จริง — ถ้าทุกอย่าง
   ถูกต้อง (state ตรง, PKCE verifier ตรง, client_id/secret ถูกต้อง) เซิร์ฟเวอร์จะแลก `code` เป็น token
   จริง ดึงโปรไฟล์จริงจาก GitHub API แล้วสร้าง local session ให้คุณ — เปิด `/me` ต่อจะเห็นข้อมูล GitHub
   username จริงของคุณ

### 75.18 Callback Handler เต็มรูปแบบ: แลก Code เป็น Token แล้วดึงโปรไฟล์ผู้ใช้

ก่อนจะดู callback handler ต้องมี handler อีกตัวก่อน — ตัวที่**สร้าง**การ redirect ไป GitHub (นำโค้ดจาก
75.14-75.16 มาห่อเป็น Axum handler จริง):

```rust
async fn login_github(State(state): State<AppState>, session: Session) -> Response {
    // PKCE (75.15): สร้าง challenge/verifier คู่ใหม่ทุกครั้งที่มีคนกด "Sign in with GitHub"
    let (pkce_challenge, pkce_verifier) = PkceCodeChallenge::new_random_sha256();

    let (auth_url, csrf_token) = state
        .oauth_client
        .authorize_url(CsrfToken::new_random)
        .add_scope(Scope::new("read:user".to_string()))
        .set_pkce_challenge(pkce_challenge)
        .url();

    // เก็บ state (CSRF, 75.16) และ pkce_verifier (75.15) ไว้ใน session ฝั่ง server ชั่วคราว -- รอ
    // callback มาตรวจสอบ (คนละ key กับ USER_ID_KEY/USERNAME_KEY -- นี่ยังไม่ใช่ session ที่ authenticate
    // แล้ว เป็นแค่ "session ระหว่างทาง" ที่ผูกกับ flow ที่กำลังดำเนินอยู่เท่านั้น)
    session
        .insert(OAUTH_CSRF_KEY, csrf_token.secret().clone())
        .await
        .unwrap();
    session
        .insert(OAUTH_PKCE_VERIFIER_KEY, pkce_verifier.secret().clone())
        .await
        .unwrap();

    Redirect::to(auth_url.as_str()).into_response()
}
```

สังเกตว่า handler นี้คืนค่าเป็น `303 See Other` (ผ่าน `Redirect::to()` ของ Axum — ดู Part 61 เรื่อง status
code หมวด redirect) ไปยัง URL ของ GitHub ตรง ๆ browser จะติดตาม redirect นี้เองอัตโนมัติ (พฤติกรรม
มาตรฐานของทุก browser ต่อ `3xx` response) แล้วแสดงหน้า consent screen ของ GitHub ให้ผู้ใช้เห็น

#### ชนิดของ client ที่ตั้งค่าครบแล้ว: `GithubOAuthClient` type alias

`AppState` ต้องเก็บ `oauth_client` ไว้ใช้ข้ามหลาย request (สร้างครั้งเดียวตอน `main()` ไม่ใช่สร้างใหม่ทุก
ครั้งที่มี request เข้ามา) แต่ type เต็มของ `BasicClient` ที่ผ่าน `.set_auth_uri()`/`.set_token_uri()`/ฯลฯ
มาแล้ว (หัวข้อ 75.14) ยาวและมี type parameter ห้าตัวตาม typestate pattern — ตั้ง type alias ให้อ่านง่ายขึ้น:

```rust
use oauth2::{basic::BasicClient, EndpointNotSet, EndpointSet};

// alias ชนิดของ client ที่ตั้งค่า auth_uri + token_uri ครบแล้ว (ผ่าน typestate ของ oauth2 5.x)
// HasAuthUrl=EndpointSet, HasDeviceAuthUrl=EndpointNotSet, HasIntrospectionUrl=EndpointNotSet,
// HasRevocationUrl=EndpointNotSet, HasTokenUrl=EndpointSet
type GithubOAuthClient =
    BasicClient<EndpointSet, EndpointNotSet, EndpointNotSet, EndpointNotSet, EndpointSet>;
```

ลำดับ type parameter ทั้งห้าตัวตรงกับลำดับที่ประกาศไว้ใน type alias จริงของ `oauth2` (ดูหัวข้อ 75.14 ที่
คัดลอกนิยามจาก source code มา) — เขียน `EndpointSet` ตรงตำแหน่ง `HasAuthUrl` และ `HasTokenUrl` เพราะโค้ด
`main()` เรียก `.set_auth_uri()`/`.set_token_uri()` แล้วจริง ส่วน `HasDeviceAuthUrl`/`HasIntrospectionUrl`/
`HasRevocationUrl` ยังเป็น `EndpointNotSet` เพราะ flow ของบทนี้ไม่ใช้ device flow, token introspection,
หรือ token revocation — ถ้า mismatch (เช่น เขียน `EndpointNotSet` ผิดตำแหน่ง) compiler จะปฏิเสธทันทีตอน
พยายาม assign ค่าจาก `BasicClient::new(...).set_auth_uri(...).set_token_uri(...)` เข้า type alias นี้
(ดูกับดักที่พบบ่อยข้อ 5 ท้ายบทที่จับ error จริงจากสถานการณ์ใกล้เคียง)

#### `AppState` เต็มรูปแบบสำหรับทั้งสองทาง auth

```rust
#[derive(Clone)]
struct AppState {
    // ผู้ใช้แบบ traditional (username/password) -- demo เท่านั้น ห้ามเก็บ plaintext password ใน
    // production (ดูหมายเหตุใน 75.5)
    local_users: Arc<Mutex<HashMap<String, String>>>,
    // ผู้ใช้ที่เคย sign in ผ่าน GitHub มาแล้ว key = github user id (เป็น string)
    github_users: Arc<Mutex<HashMap<String, GithubProfile>>>,
    oauth_client: Arc<GithubOAuthClient>,
    http_client: Arc<oauth2::reqwest::Client>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct GithubProfile {
    id: u64,
    login: String,
}
```

`http_client` คือ `reqwest::Client` ที่ `oauth2` crate re-export มาให้เองผ่าน `oauth2::reqwest` (ไม่ต้อง
เพิ่ม `reqwest` เป็น dependency แยกใน `Cargo.toml`) — เก็บไว้เป็น `Arc` ใน `AppState` แบบเดียวกับ
`oauth_client` เพราะการสร้าง `reqwest::Client` ใหม่ทุกครั้งที่มี request เข้ามาเสียค่าใช้จ่ายโดยไม่จำเป็น
(reqwest แนะนำให้สร้าง client ครั้งเดียวแล้ว reuse — client ภายในมี connection pool ของตัวเอง)

#### Session keys เพิ่มเติมสำหรับ OAuth2 flow

```rust
const USER_ID_KEY: &str = "user_id";
const USERNAME_KEY: &str = "username";
const AUTH_METHOD_KEY: &str = "auth_method";
const OAUTH_CSRF_KEY: &str = "oauth_csrf_state";
const OAUTH_PKCE_VERIFIER_KEY: &str = "oauth_pkce_verifier";
```

สังเกตว่ามีสองกลุ่ม key ที่ความหมายต่างกันโดยสิ้นเชิงแม้จะอยู่ใน `Session` เดียวกัน: `USER_ID_KEY`/
`USERNAME_KEY`/`AUTH_METHOD_KEY` คือข้อมูลของ session ที่ **authenticate แล้ว** (มีอยู่ก็แปลว่า login
สำเร็จแล้วจริง ไม่ว่าจะทางไหน) ส่วน `OAUTH_CSRF_KEY`/`OAUTH_PKCE_VERIFIER_KEY` คือข้อมูล**ชั่วคราว**ที่มี
อยู่แค่ระหว่างที่ OAuth2 flow กำลังดำเนินอยู่ (ตั้งใน `login_github`, อ่านและลบใน `github_callback`) —
ไม่ใช่สัญญาณว่า login สำเร็จแล้ว

#### ฟังก์ชันกลาง: `establish_local_session` — จุดที่ทั้งสองทาง auth "จบลงที่เดียวกัน"

```rust
// นี่คือจุดที่ทำให้ทั้งสองทาง auth "จบลงที่เดียวกัน" -- ไม่ว่าจะมาจาก password หรือ GitHub, โค้ดส่วนนี้
// ทำงานเหมือนกันเป๊ะ: เขียน user_id/username เข้า session แล้ว cycle_id() ป้องกัน session fixation (75.6)
async fn establish_local_session(session: &Session, user_id: &str, username: &str, method: &str) {
    session.insert(USER_ID_KEY, user_id).await.unwrap();
    session.insert(USERNAME_KEY, username).await.unwrap();
    session.insert(AUTH_METHOD_KEY, method).await.unwrap();
    session.cycle_id().await.unwrap();
}
```

ฟังก์ชันนี้ไม่รู้อะไรเลยเกี่ยวกับ "มาจากไหน" — รับแค่ `user_id`/`username`/`method` (string ล้วน ๆ) แล้วเขียน
เข้า session ตามรูปแบบเดียวกันเสมอ ทั้ง `login` (traditional, ส่ง `method = "password"`) และ
`github_callback` (ส่ง `method = "github_oauth"`) เรียกฟังก์ชันนี้ตัวเดียวกัน — ผลคือ **`me`/`logout`/
protected route อื่น ๆ ทั้งหมดของแอปไม่ต้องรู้เลยว่า user คนนี้ authenticate มาทางไหน** อ่าน
`USER_ID_KEY`/`USERNAME_KEY` จาก session เหมือนกันทุกกรณี (เก็บ `auth_method` ไว้ด้วยเผื่อ UI ต้องการ
แสดงผลต่างกัน เช่น ปุ่ม "เปลี่ยนรหัสผ่าน" ที่ไม่ควรมีให้ user ที่ login ผ่าน GitHub เห็น — แต่ authorization
logic หลักไม่ต้องแยก)

#### Callback Handler เต็มรูปแบบ

```rust
#[derive(Deserialize)]
struct CallbackQuery {
    code: String,
    state: String,
}

#[derive(thiserror::Error, Debug)]
enum OAuthCallbackError {
    #[error("csrf state ไม่ตรงกัน หรือไม่พบ oauth flow ที่กำลังดำเนินอยู่ใน session")]
    CsrfMismatch,
    #[error("ไม่พบ pkce verifier ใน session (flow อาจหมดอายุ หรือเริ่มจากคนละ session)")]
    MissingPkceVerifier,
    #[error("แลก code เป็น token ไม่สำเร็จ: {0}")]
    TokenExchange(String),
    #[error("ดึงข้อมูลโปรไฟล์จาก GitHub ไม่สำเร็จ: {0}")]
    ProfileFetch(String),
}

impl IntoResponse for OAuthCallbackError {
    fn into_response(self) -> Response {
        tracing::warn!(error = %self, "github oauth callback ล้มเหลว");
        (StatusCode::BAD_REQUEST, self.to_string()).into_response()
    }
}

async fn github_callback(
    State(state): State<AppState>,
    session: Session,
    Query(query): Query<CallbackQuery>,
) -> Result<Response, OAuthCallbackError> {
    // ขั้นที่ 1: ตรวจ state (CSRF, หัวข้อ 75.16) -- ต้องตรงกับตัวที่เราสุ่มไว้ตอนสร้าง auth_url เท่านั้น
    let expected_state: Option<String> = session.get(OAUTH_CSRF_KEY).await.unwrap();
    if expected_state.as_deref() != Some(query.state.as_str()) {
        return Err(OAuthCallbackError::CsrfMismatch);
    }
    // ใช้ครั้งเดียวแล้วลบทิ้ง -- ป้องกันไม่ให้ callback URL เดิมถูกเปิดซ้ำสำเร็จเป็นครั้งที่สอง
    let _: Option<String> = session.remove(OAUTH_CSRF_KEY).await.unwrap();

    // ขั้นที่ 2: ดึง pkce_verifier (หัวข้อ 75.15) ที่เก็บไว้ตอน /login/github
    let verifier_secret: String = session
        .remove(OAUTH_PKCE_VERIFIER_KEY)
        .await
        .unwrap()
        .ok_or(OAuthCallbackError::MissingPkceVerifier)?;
    let pkce_verifier = PkceCodeVerifier::new(verifier_secret);

    // ขั้นที่ 3: แลก authorization code เป็น access token -- เกิดขึ้น "server-to-server" เท่านั้น
    // (client_secret ไม่เคยถูกส่งไปที่ browser เลยตลอด flow นี้ -- ดูหัวข้อ 75.12 step 6)
    let token_result = state
        .oauth_client
        .exchange_code(AuthorizationCode::new(query.code))
        .set_pkce_verifier(pkce_verifier)
        .request_async(state.http_client.as_ref())
        .await
        .map_err(|e| OAuthCallbackError::TokenExchange(e.to_string()))?;

    let access_token = token_result.access_token().secret();

    // ขั้นที่ 4: ใช้ access token เรียก GitHub API เพื่อดึงข้อมูลผู้ใช้จริง
    // หมายเหตุ: reqwest ตัวที่ oauth2 re-export มาไม่ได้เปิด feature "json" ไว้ (มีแค่ feature ที่ oauth2
    // เองต้องใช้ภายใน) จึงอ่าน response เป็น text แล้ว parse ด้วย serde_json ตรง ๆ แทนการเรียก .json()
    // ของ reqwest โดยตรง (ที่ต้องพึ่ง feature "json" ของ reqwest ซึ่งไม่ถูกเปิดในที่นี้)
    let response_text = state
        .http_client
        .get("https://api.github.com/user")
        .bearer_auth(access_token)
        .header("User-Agent", "rust-course-part75-demo")
        .send()
        .await
        .map_err(|e| OAuthCallbackError::ProfileFetch(e.to_string()))?
        .text()
        .await
        .map_err(|e| OAuthCallbackError::ProfileFetch(e.to_string()))?;
    let profile: GithubProfile = serde_json::from_str(&response_text)
        .map_err(|e| OAuthCallbackError::ProfileFetch(e.to_string()))?;

    // ขั้นที่ 5: upsert local user record แล้ว "establish_local_session" ตัวเดียวกับที่ /login (password)
    // ใช้ -- นี่คือจุดที่ทั้งสองทาง auth มาบรรจบกัน
    state
        .github_users
        .lock()
        .unwrap()
        .insert(profile.id.to_string(), profile.clone());

    establish_local_session(
        &session,
        &format!("github-{}", profile.id),
        &profile.login,
        "github_oauth",
    )
    .await;

    Ok(Redirect::to("/me").into_response())
}
```

อธิบายทีละขั้น: **ขั้นที่ 1-2** คือการตรวจสอบที่หัวข้อ 75.15-75.16 อธิบายไว้แล้วในทางทฤษฎี — ที่นี่คือ
implementation จริง สังเกตว่าทั้ง `state`/`pkce_verifier` ถูก **`.remove()`** (ไม่ใช่ `.get()` เฉย ๆ) ออก
จาก session ทันทีที่อ่านเสร็จ ทำให้ callback URL เดิมเปิดซ้ำครั้งที่สองไม่สำเร็จอีก (ครั้งแรก consume
ค่าที่เก็บไว้จนหมด ครั้งที่สองจะเจอ `MissingPkceVerifier` เพราะ session ไม่มีค่านั้นเหลือแล้ว) — เป็นการ
ป้องกัน replay เพิ่มอีกชั้นหนึ่งเหนือกว่าการตรวจ state เฉย ๆ **ขั้นที่ 3** คือหัวใจของ Authorization Code
flow ทั้งหมด (step 6 ในหัวข้อ 75.12): `exchange_code()` สร้าง request ที่ถูกต้องตาม RFC 6749 (`POST`
ไปยัง token endpoint พร้อม `client_id`/`client_secret`/`code`/`grant_type=authorization_code`/
`redirect_uri`) แล้ว `.set_pkce_verifier()` เติม `code_verifier` เข้าไปตาม RFC 7636 ก่อนที่ `.request_async()`
จะยิง request จริงผ่าน `http_client` ที่ส่งเข้ามา **ขั้นที่ 4** คือส่วนที่ไม่มีใน flow มาตรฐานของ OAuth2 เอง
(RFC 6749 ไม่ได้บอกว่าต้องทำอะไรกับ token — แค่บอกวิธีได้ token มา) แต่เป็นสิ่งที่ทุกแอปที่ทำ "Sign in with
GitHub" ต้องทำต่อ: ใช้ access token ที่ได้ไปเรียก GitHub REST API (`GET /user`) เพื่อรู้ว่า token นี้เป็น
ของใคร (`id`/`login`) — นี่คือเหตุผลที่หัวข้อ 75.10 บอกว่า OAuth2 เองไม่ใช่ authentication โดยตรง
(delegated authorization ล้วน ๆ) แอปต้อง**เพิ่มขั้นตอนนี้เอง**เพื่อแปลง "มี token ที่เข้าถึงข้อมูลได้" ไปสู่
"รู้ว่าเป็นใคร" **ขั้นที่ 5** คือจุดเชื่อมกับครึ่งแรกของบทนี้: เรียก `establish_local_session()` ตัวเดียวกับ
ที่ `login` (traditional) เรียก

#### ขอบเขตของการตรวจสอบส่วนนี้

โค้ด `github_callback` ข้างบน compile ผ่านและ logic ถูกต้องตามที่ตรวจสอบไว้ — ในสภาพแวดล้อมที่ใช้เขียนและ
ตรวจสอบบทนี้ ได้ทดสอบเรียก `login_github` จริง (ได้ redirect + state + PKCE challenge ที่ถูกต้องตามที่
75.17 พิสูจน์แล้ว) และทดสอบเรียก `github_callback` ด้วย `code`/`state` ปลอม (เพราะไม่มี `code` จริงจาก
GitHub ให้ใช้ — ต้องผ่านขั้นตอนกด "Authorize" ในหัวข้อ 75.17 ก่อน) ผลคือ **โค้ดเดินทางไปถึงขั้นที่ 3 จริง
— `exchange_code(...).request_async(...)` ถูกเรียกจริง ส่ง request ออกทางเน็ตเวิร์กจริง (ไม่ใช่ compile
error หรือ panic) แล้วได้ HTTP response/error กลับมาจริง** ยืนยันว่าโค้ดส่วนสร้าง request (URL, header,
body ตาม RFC 6749) ถูกต้องและเดินท่อ (network plumbing) ได้จริงจนถึงจุดนี้ — สิ่งที่**ไม่สามารถยืนยันได้**
ในสภาพแวดล้อมนี้คือ response ที่แท้จริงจากเซิร์ฟเวอร์ OAuth ของ GitHub ในกรณีที่มี `code`/`state`/PKCE
verifier ที่ถูกต้องครบทุกตัว (เพราะไม่มี `code` จริงให้ทดสอบ ตามที่อธิบายไว้ในหัวข้อ 75.13 และ 75.17) —
ผู้อ่านที่ทำตามขั้นตอนใน 75.17 ครบ (สร้าง GitHub OAuth App จริง + เปิด browser กด Authorize จริง) จะได้
เห็นขั้นที่ 3-5 ทำงานสำเร็จเต็มรูปแบบด้วยตัวเอง — ถ้า credential ที่ใช้ไม่ตรงกับที่ GitHub ลงทะเบียนไว้
(เช่น ใช้ client_id/secret ที่เป็นค่าตัวอย่าง ไม่ใช่ค่าจริง) GitHub จะตอบกลับด้วย error response ตาม
มาตรฐานของ OAuth2 token endpoint เอง (body เป็น JSON ที่มี field `error` เช่น
`incorrect_client_credentials`) ซึ่ง `exchange_code` จะแปลงเป็น `Err` ให้ `.map_err()` จับได้เหมือนกับ
error อื่น ๆ ทุกกรณี — ไม่มี branch โค้ดพิเศษที่ต้องเขียนแยกสำหรับกรณีนี้

### 75.19 Capstone: รวม Session Login และ "Sign in with GitHub" เข้าด้วยกัน

มาประกอบทุกชิ้นจากทั้งสองครึ่งของบทนี้ (75.1-75.9 และ 75.10-75.18) เข้าเป็นแอปเดียวที่รันได้จริง —
`POST /login` (traditional, ทวนจาก 75.5 แต่ปรับให้เรียก `establish_local_session` ตัวเดียวกับ OAuth2),
`GET /login/github` + `GET /callback` (OAuth2, จาก 75.18), และ `GET /me`/`POST /logout` (ใช้ร่วมกันทั้ง
สองทาง เพราะทั้งคู่จบลงที่ session รูปแบบเดียวกัน)

```rust
use axum::{
    extract::{Query, State},
    http::StatusCode,
    response::{IntoResponse, Redirect, Response},
    routing::{get, post},
    Json, Router,
};
use oauth2::{
    basic::BasicClient, AuthUrl, AuthorizationCode, ClientId, ClientSecret, CsrfToken,
    EndpointNotSet, EndpointSet, PkceCodeChallenge, PkceCodeVerifier, RedirectUrl, Scope,
    TokenResponse, TokenUrl,
};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use time::Duration;
use tower_sessions::{Expiry, MemoryStore, Session, SessionManagerLayer};

// (type alias, AppState, GithubProfile, session keys, establish_local_session,
//  login_github, CallbackQuery, OAuthCallbackError, github_callback -- ทั้งหมดจากหัวข้อ 75.18 เหมือนเดิม
//  ทุกบรรทัด ไม่ต้องเขียนซ้ำที่นี่)

// ---------- Part A: traditional session login (ปรับจาก 75.5 ให้เรียก establish_local_session) ----------

#[derive(Deserialize)]
struct LoginRequest {
    username: String,
    password: String,
}

async fn login(
    State(state): State<AppState>,
    session: Session,
    Json(body): Json<LoginRequest>,
) -> Result<StatusCode, StatusCode> {
    let password_matches = {
        let users = state.local_users.lock().unwrap();
        users.get(&body.username) == Some(&body.password)
    }; // <- lock ถูก drop ตรงนี้ ก่อนถึง .await ตัวแรก (ดูกับดักข้อ 3 ท้ายบท)

    if !password_matches {
        return Err(StatusCode::UNAUTHORIZED);
    }

    establish_local_session(
        &session,
        &format!("local-{}", body.username),
        &body.username,
        "password",
    )
    .await;
    Ok(StatusCode::OK)
}

#[derive(Serialize)]
struct MeResponse {
    user_id: String,
    username: String,
    auth_method: String,
}

async fn me(session: Session) -> Result<Json<MeResponse>, StatusCode> {
    let user_id: Option<String> = session.get(USER_ID_KEY).await.unwrap();
    let username: Option<String> = session.get(USERNAME_KEY).await.unwrap();
    let auth_method: Option<String> = session.get(AUTH_METHOD_KEY).await.unwrap();
    match (user_id, username, auth_method) {
        (Some(user_id), Some(username), Some(auth_method)) => {
            Ok(Json(MeResponse { user_id, username, auth_method }))
        }
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

async fn logout(session: Session) -> impl IntoResponse {
    session.flush().await.unwrap();
    StatusCode::NO_CONTENT
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let mut local_users = HashMap::new();
    local_users.insert("nan".to_string(), "hunter2".to_string());

    let client_id = ClientId::new(
        std::env::var("GITHUB_CLIENT_ID").unwrap_or_else(|_| "demo_client_id".to_string()),
    );
    let client_secret = ClientSecret::new(
        std::env::var("GITHUB_CLIENT_SECRET").unwrap_or_else(|_| "demo_client_secret".to_string()),
    );
    let oauth_client: GithubOAuthClient = BasicClient::new(client_id)
        .set_client_secret(client_secret)
        .set_auth_uri(AuthUrl::new("https://github.com/login/oauth/authorize".to_string()).unwrap())
        .set_token_uri(TokenUrl::new("https://github.com/login/oauth/access_token".to_string()).unwrap())
        .set_redirect_uri(
            RedirectUrl::new("http://127.0.0.1:3178/callback".to_string()).unwrap(),
        );

    let http_client = oauth2::reqwest::ClientBuilder::new()
        .redirect(oauth2::reqwest::redirect::Policy::none()) // ห้าม reqwest ติดตาม redirect เอง -- ต้อง
        // ให้โค้ดของเราเห็น response ดิบจาก token endpoint เท่านั้น (ป้องกัน open-redirect ผ่าน HTTP client
        // ที่ตั้งค่าไม่รอบคอบ ตามคำแนะนำในตัวอย่างของ oauth2 crate เอง)
        .build()
        .expect("reqwest client ควรสร้างสำเร็จ");

    let state = AppState {
        local_users: Arc::new(Mutex::new(local_users)),
        github_users: Arc::new(Mutex::new(HashMap::new())),
        oauth_client: Arc::new(oauth_client),
        http_client: Arc::new(http_client),
    };

    let session_layer = SessionManagerLayer::new(MemoryStore::default())
        .with_secure(false) // *** เฉพาะ dev บน http://127.0.0.1 -- production ต้อง true (ดู 75.4) ***
        .with_same_site(tower_sessions::cookie::SameSite::Lax) // *** ต้องเป็น Lax ไม่ใช่ Strict
        // เพราะ callback จาก GitHub เป็น cross-site navigation (ดูกับดักที่พบบ่อยข้อ 6 ท้ายบท) ***
        .with_expiry(Expiry::OnInactivity(Duration::minutes(30)));

    let app = Router::new()
        .route("/login", post(login))
        .route("/login/github", get(login_github))
        .route("/callback", get(github_callback))
        .route("/me", get(me))
        .route("/logout", post(logout))
        .layer(session_layer)
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3178")
        .await
        .unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

สังเกตสองจุดที่เปลี่ยนจาก 75.5 อย่างตั้งใจ: **(1)** `with_same_site(SameSite::Lax)` — ค่านี้**ต้อง**เป็น
`Lax` ไม่ใช่ค่า default ของ `tower-sessions` (ซึ่งคือ `Strict` — ดูกับดักที่พบบ่อยข้อ 6 ที่อธิบายว่าทำไม
`Strict` จะทำให้ OAuth2 callback พังทันที) **(2)** `oauth2::reqwest::redirect::Policy::none()` — ปิดการ
follow redirect อัตโนมัติของ `reqwest` client ที่ใช้คุยกับ token endpoint (คนละตัวกับ redirect ที่ browser
ของผู้ใช้ทำตอน `login_github`/`github_callback` คืนค่า — ตัวนั้นเป็น browser-level redirect ที่ต้องการ
ให้เกิด ส่วนตัวนี้คือ server-to-server HTTP client ที่**ไม่ต้องการ**ให้ follow redirect เองเพราะ response
ที่คาดหวังจาก token endpoint คือ JSON body ตรง ๆ ไม่ใช่ redirect)

#### curl transcript: traditional login → `/me` → `/logout` (verified แบบ end-to-end เต็มรูปแบบ)

ส่วนนี้ **ไม่พึ่งพา GitHub เลย** จึงทดสอบแบบ end-to-end ได้ครบ 100% ในสภาพแวดล้อมใดก็ตาม (รันเซิร์ฟเวอร์
ข้างบนจริงแล้วยิง `curl` จริงตามลำดับนี้):

```bash
$ curl -sS -i http://127.0.0.1:3178/me
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:54:56 GMT
```

```bash
$ curl -sS -i -X POST http://127.0.0.1:3178/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"wrongpass"}'
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:54:56 GMT
```

```bash
$ curl -sS -i -c /tmp/cap_cookies.txt -X POST http://127.0.0.1:3178/login \
    -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"hunter2"}'
```
```
HTTP/1.1 200 OK
set-cookie: id=dRb1GyTjX5T_Lh9kNRCJRQ; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
content-length: 0
date: Sun, 27 Sep 2026 00:54:56 GMT
```

```bash
$ curl -sS -i -b /tmp/cap_cookies.txt http://127.0.0.1:3178/me
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 65
date: Sun, 27 Sep 2026 00:54:56 GMT

{"user_id":"local-nan","username":"nan","auth_method":"password"}
```

สังเกต `auth_method` ในผลลัพธ์: `"password"` — ยืนยันว่า `establish_local_session` เขียนค่านี้ถูกต้องจริง
ตามทางที่ user login มา ต่อด้วย logout:

```bash
$ curl -sS -i -b /tmp/cap_cookies.txt -c /tmp/cap_cookies.txt -X POST http://127.0.0.1:3178/logout
```
```
HTTP/1.1 204 No Content
set-cookie: id=; Path=/; Max-Age=0; Expires=Sat, 27 Sep 2025 00:54:56 GMT
date: Sun, 27 Sep 2026 00:54:56 GMT
```

```bash
$ curl -sS -i -b /tmp/cap_cookies.txt http://127.0.0.1:3178/me
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
date: Sun, 27 Sep 2026 00:54:56 GMT
```

**[verified end-to-end เต็มรูปแบบ]** — ทุก request/response ข้างบนมาจากการรันเซิร์ฟเวอร์จริงและยิง `curl`
จริง ไม่มีขั้นตอนใดที่ต้องพึ่งพา GitHub หรือ browser

#### curl transcript: `GET /login/github` → 303 redirect ไปยัง GitHub จริง (verified end-to-end เฉพาะฝั่งเรา)

```bash
$ curl -sS -i -c /tmp/gh_cookies.txt http://127.0.0.1:3178/login/github
```
```
HTTP/1.1 303 See Other
location: https://github.com/login/oauth/authorize?response_type=code&client_id=demo_client_id&state=75w0hCPX9_-b05Zwr2oraA&code_challenge=NJyztsbGvly9Ny46tYVOecVi4uRGJ3z11k1e1ACnhLQ&code_challenge_method=S256&redirect_uri=http%3A%2F%2F127.0.0.1%3A3178%2Fcallback&scope=read%3Auser
set-cookie: id=VePC4x8HURlMzbd-Ktfl7g; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
content-length: 0
date: Sun, 27 Sep 2026 00:55:02 GMT
```

**[verified end-to-end เฉพาะฝั่งเซิร์ฟเวอร์ของเรา]** — `303` และ `location` header ข้างบนคือสิ่งที่
เซิร์ฟเวอร์ของเราสร้างขึ้นจริง (ตรวจแล้วว่ามี `response_type=code`, `client_id`, `state`, `code_challenge`,
`code_challenge_method=S256`, `redirect_uri`, `scope` ครบตามที่ 75.17 พิสูจน์โครงสร้างไว้) — ส่วนที่**ไม่**
verified ในบรรทัดนี้คือสิ่งที่เกิดขึ้น**หลังจาก** browser ติดตาม `location` นี้ไปจริง (หน้า consent screen
ของ GitHub เอง) เพราะ `curl` เปล่า ๆ ไม่ติดตาม redirect ไปเปิดหน้าเว็บและไม่มีมนุษย์กด "Authorize" ให้

#### curl transcript: `/callback` ที่มี `state` ไม่ตรง หรือขาดหายไป → 400 (verified end-to-end เต็มรูปแบบ)

ส่วนนี้ทดสอบได้แบบ end-to-end เต็มรูปแบบโดย**ไม่ต้อง**พึ่ง GitHub เลย เพราะเป็นการตรวจสอบที่เกิดขึ้น
**ก่อน**ที่โค้ดจะยิง request ไปหา GitHub ด้วยซ้ำ (ดูขั้นที่ 1 ใน `github_callback` หัวข้อ 75.18):

```bash
$ curl -sS -i -b /tmp/gh_cookies.txt "http://127.0.0.1:3178/callback?code=fake_code_123&state=wrong_state_value_xyz"
```
```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 146
date: Sun, 27 Sep 2026 00:55:09 GMT

csrf state ไม่ตรงกัน หรือไม่พบ oauth flow ที่กำลังดำเนินอยู่ใน session
```

ทดสอบอีกกรณี: ไม่ส่ง cookie ไปเลย (จำลอง browser คนละตัว/คนละ session จากที่เริ่ม flow):

```bash
$ curl -sS -i "http://127.0.0.1:3178/callback?code=fake_code_123&state=75w0hCPX9_-b05Zwr2oraA"
```
```
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8
content-length: 146
date: Sun, 27 Sep 2026 00:55:09 GMT

csrf state ไม่ตรงกัน หรือไม่พบ oauth flow ที่กำลังดำเนินอยู่ใน session
```

**[verified end-to-end เต็มรูปแบบ]** — ทั้งสองกรณีได้ `400 Bad Request` จริง พิสูจน์ว่ากลไก state
verification (หัวข้อ 75.16) ทำงานถูกต้องจริงในโค้ด ไม่ใช่แค่คำอธิบายเชิงทฤษฎี: กรณีแรก `state` ที่ callback
ได้รับไม่ตรงกับที่เก็บไว้ใน session (ที่ผูกกับ cookie ที่ส่งไป) กรณีที่สองไม่มี session ที่ตรงกับ flow นี้
เลย (เพราะไม่ส่ง cookie ไปเลย) — ทั้งสองกรณีแอปปฏิเสธก่อนจะพยายามแลก `code` เป็น token ด้วยซ้ำ ตรงตามที่
หัวข้อ 75.16 อธิบายไว้ว่าต้องทำ

**สรุปสถานะการ verified ของ capstone ทั้งหมด**: (a) traditional login/`/me`/`logout` — verified
end-to-end เต็มรูปแบบ (b) `/login/github` การสร้าง redirect — verified end-to-end เฉพาะฝั่งเซิร์ฟเวอร์ของ
เรา (โครงสร้าง URL ถูกต้องตามที่ 75.17 พิสูจน์) (c) `/callback` กรณี state ผิด/ขาด — verified end-to-end
เต็มรูปแบบ (d) `/callback` กรณี state/PKCE ถูกต้องครบและมี `code` จริงจาก GitHub — **ไม่ได้** verified
end-to-end ในสภาพแวดล้อมนี้ (ต้องมีมนุษย์กด "Authorize" จริงตามที่ 75.13/75.17 อธิบายไว้) แต่โค้ดในเส้นทาง
นี้ verified ในระดับ compile ผ่าน + เดินทางไปถึงจุดยิง network request จริงตามที่ 75.18 อธิบายไว้

### 75.20 เลือกใช้แบบไหน: JWT vs Session vs OAuth2

ถึงตอนนี้คุณเห็นทั้งสามกลไกเต็มรูปแบบแล้ว (JWT จาก Part 74, Session จากครึ่งแรกของบทนี้, OAuth2 จากครึ่ง
หลัง) — สิ่งสำคัญที่ต้องเข้าใจให้ชัดก่อนอื่นใด: **คำถาม "JWT vs Session" กับคำถาม "OAuth2 vs ไม่ใช้ OAuth2"
เป็นคำถามคนละมิติกัน ไม่ใช่ตัวเลือกสามตัวที่ต้องเลือกแค่หนึ่งเดียว**

**มิติที่ 1 — JWT vs Session**: คำถามนี้คือ "หลังจากรู้ตัวตนของ user แล้ว จะให้ server จดจำสถานะการ login
ยังไง" ตารางในหัวข้อ 75.2 สรุป tradeoff หลักไว้แล้ว (stateless เทียบ stateful) — สรุปเป็นคำแนะนำเชิงปฏิบัติ:

- เลือก **Session** เมื่อ: แอปเป็นเว็บแอปทั่วไปที่ผู้ใช้ล็อกอินผ่าน browser เป็นหลัก, ต้องการ revoke ทันที
  ได้ (banned user, "logout จากทุกอุปกรณ์"), ทีมพร้อมดูแล shared session store (Redis) ตอน scale หลาย
  instance, ระบบยังไม่ซับซ้อนถึงระดับ microservices ข้าม domain หลายตัว
- เลือก **JWT** เมื่อ: client เป็น mobile app หรือ SPA ที่ไม่มี cookie แบบ browser ธรรมดา, ระบบเป็น public
  API ที่ third-party เรียกตรง, ระบบเป็น microservices ที่หลาย service ต้องตรวจสอบตัวตนได้เองโดยไม่ต้องคุย
  กับ service กลางทุกครั้ง (ดูหัวข้อ 75.8 ที่เชื่อมไปยัง Part 81), ยอมรับ tradeoff เรื่อง revocation ที่
  ต้องแก้ด้วย blocklist/short-lived token ตามที่ Part 74 อธิบายไว้

**มิติที่ 2 — OAuth2 (Sign in with X) vs ไม่ใช้**: คำถามนี้คือ "user จะพิสูจน์ตัวตนกับระบบเรายังไงตั้งแต่
แรก" — username/password ที่ระบบเราเก็บเอง (ต้องดูแล hash/salt/reset-password ทั้งหมดเอง) เทียบกับให้
ผู้ใช้ login ด้วยบัญชีที่มีอยู่แล้วบนบริการอื่น (GitHub/Google/ฯลฯ ตามที่หัวข้อ 75.10 อธิบาย) โดยที่ระบบเรา
ไม่ต้องเก็บ password ของผู้ใช้เองเลย (ลดความรับผิดชอบด้านความปลอดภัยของข้อมูล credential ลงไปมาก — ถ้า
ข้อมูลของเราหลุด จะไม่มี password ของผู้ใช้ให้หลุดไปด้วย เพราะไม่ได้เก็บไว้ตั้งแต่แรก)

**คำตอบที่ถูกต้องคือ: ทั้งสองมิตินี้ผสมกันได้อิสระ** ตัวอย่างจริงที่บทนี้ประกอบให้เห็นแล้วในหัวข้อ 75.19
คือ "OAuth2 (mixin ที่ 2) + Session (มิติที่ 1)" — ใช้ GitHub เป็นตัวพิสูจน์ตัวตน แล้วจบลงที่ local session
ธรรมดา แต่ก็ทำ "OAuth2 + JWT" ได้เหมือนกัน (แลก `code` เป็น GitHub token แล้วออก JWT ของระบบเราเองแทน
session — เหมาะกับ mobile app ที่ต้องการ "Sign in with GitHub" แต่ไม่มี cookie ให้ใช้) หรือ "ไม่ใช้ OAuth2
+ JWT" (ระบบ public API ที่ยังใช้ username/password ของตัวเอง แต่ออก JWT แทน session) ก็เป็นชุดที่ถูกต้อง
เหมือนกัน — ระบบใหญ่จำนวนมากในโลกจริงถึงกับใช้**มากกว่าหนึ่งชุดพร้อมกันในระบบเดียว** เช่น เว็บแอปหลักใช้
session (สำหรับผู้ใช้ทั่วไปที่ล็อกอินผ่าน browser) แต่มี public API แยกที่ third-party integration เรียก
ด้วย JWT/API key คนละชุดสิทธิ์กันโดยสิ้นเชิง — ไม่มีกฎที่บอกว่าทั้งระบบต้องใช้กลไกเดียวทุกจุด

สิ่งที่ทำให้ระบบในหัวข้อ 75.19 ยืดหยุ่นพอที่จะผสมแบบนี้ได้คือหลักการที่ `establish_local_session` แสดงให้
เห็น: **แยก "วิธีพิสูจน์ตัวตน" (password ตรวจกับ database เอง หรือ delegate ไปให้ GitHub ตรวจ) ออกจาก
"วิธีจดจำสถานะ login หลังจากพิสูจน์ตัวตนสำเร็จแล้ว" (session หรือ JWT) ให้เป็นสองเรื่องที่ไม่ผูกติดกันตั้งแต่
ระดับโค้ด** — โค้ดส่วน "พิสูจน์ตัวตน" (ทั้ง `login` แบบ password และ `github_callback`) มีหน้าที่แค่ตอบว่า
"นี่คือ user คนไหน" แล้วส่งต่อให้จุดเดียวกัน (ในบทนี้คือ `establish_local_session` แต่ถ้าเปลี่ยนไปใช้ JWT
ก็จะเป็นฟังก์ชัน `issue_jwt(user_id, username)` แทนที่ทำหน้าที่คล้ายกัน) — ออกแบบ authentication layer
ของระบบจริงด้วยการแยกสองเรื่องนี้ตั้งแต่แรกเสมอ แม้ว่าตอนเริ่มต้นจะมีแค่ทางเข้าเดียว (เช่น password +
session อย่างเดียว) เพราะการเพิ่มทางเข้าใหม่ (OAuth2 provider อีกตัว, หรือสลับจาก session ไป JWT) ทำได้
ง่ายกว่ามากถ้าโค้ดสองส่วนนี้ไม่ได้ผูกติดกันมาตั้งแต่ต้น

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ทดสอบ session endpoint ด้วย `curl` โดยไม่ใส่ `-c`/`-b` — login "สำเร็จ" แต่ request ถัดไปไม่ authenticated

นี่คือกับดักที่พบบ่อยที่สุดตอนทดสอบระบบ session ด้วยมือ (หัวข้อ 75.5 อธิบายไว้แล้วว่าทำไม แต่ย้ำอีกครั้งใน
รูปแบบกับดักเพราะเกิดขึ้นซ้ำ ๆ จริง) — ถ้ายิง:

```bash
$ curl -sS -X POST http://127.0.0.1:3178/login -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"hunter2"}'
$ curl -sS http://127.0.0.1:3178/me
```

โดยไม่มี `-c`/`-b` เลยทั้งสองคำสั่ง — คำสั่งแรกจะได้ `200 OK` จริง (login สำเร็จจริง เซิร์ฟเวอร์สร้าง session
จริง) แต่คำสั่งที่สองจะได้:

```
{"code":401}
```

หรือ (ในระบบที่คืน body ว่าง) แค่ status `401 Unauthorized` เปล่า ๆ — **ไม่ใช่บั๊กของเซิร์ฟเวอร์** สาเหตุคือ
`curl` เป็นโปรแกรมแยกกันคนละ process ทุกครั้งที่เรียก ไม่มี "ความจำ" ระหว่างการเรียกสองครั้ง (คนละอย่างกับ
browser ที่มี cookie jar persistent อยู่ในตัวเองเสมอ) `Set-Cookie` header ที่ได้จาก request แรกถูกพิมพ์ออก
มาที่ terminal เฉย ๆ แล้วหายไป ไม่ถูกส่งกลับไปกับ request ที่สอง — ทางแก้คือใส่ `-c <file>` ตอน login (บันทึก
cookie ที่ได้) และ `-b <file>` ตอนเรียก endpoint ถัดไป (ส่ง cookie ที่บันทึกไว้กลับไป) ตามที่หัวข้อ 75.5
พิสูจน์ด้วย transcript จริงไว้ครบแล้ว

### 2. ลืม `.layer(session_layer)` — runtime error จริงจาก `tower-sessions`

ถ้าลืมเรียก `.layer(session_layer)` บน `Router` (หรือเรียกผิดตำแหน่ง เช่น เรียกหลัง `.with_state()` ในบาง
เวอร์ชันของ axum ที่ลำดับมีผล) `Session` extractor ที่ handler ขอไว้จะไม่มีอะไรให้ดึงออกมาจาก request
extensions เลย ทดสอบจริงด้วยเซิร์ฟเวอร์ที่ตั้งใจ "ลืม" `.layer()`:

```rust
async fn handler(session: Session) -> String {
    let count: Option<i32> = session.get("count").await.unwrap();
    format!("count = {:?}", count)
}

#[tokio::main]
async fn main() {
    // ตั้งใจ "ลืม" .layer(session_layer)
    let app: Router<()> = Router::new().route("/", get(handler));
    // ...
}
```

ยิง `curl` เข้าไปได้ผลลัพธ์จริงนี้:

```bash
$ curl -sS -i http://127.0.0.1:3176/
```
```
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8
content-length: 56
date: Sun, 27 Sep 2026 00:58:20 GMT

Can't extract session. Is `SessionManagerLayer` enabled?
```

ข้อความ error นี้เป็นข้อความจริงที่ `tower-sessions` ส่งกลับมาเมื่อ `Session` extractor พยายาม
`FromRequestParts` แต่ไม่พบข้อมูลที่ `SessionManagerLayer` ควรจะแทรกไว้ใน request extensions ก่อนถึง
handler — สังเกตว่า error นี้เป็น **`500`** ไม่ใช่ `401`/`400` (ยังไม่ถึงขั้นตรวจสอบสิทธิ์ด้วยซ้ำ — เป็น
ข้อผิดพลาดด้านการตั้งค่าเซิร์ฟเวอร์เอง) วิธีแก้คือตรวจว่า `.layer(session_layer)` ถูกเรียกอยู่บน `Router`
จริง และ **เรียกก่อน** route ที่ handler ใช้ `Session` extractor (หรือใช้ `.layer()` ระดับบนสุดของ
`Router` เพื่อครอบทุก route พร้อมกันแบบที่ทุกตัวอย่างในบทนี้ทำ)

### 3. ถือ `MutexGuard` ข้าม `.await` ใน login handler — compile error เรื่อง `Send`

ถ้าเขียน handler `login` โดยไม่ระวังเรื่อง scope ของ `MutexGuard` (ตามที่หมายเหตุในโค้ดหัวข้อ 75.5/75.19
เตือนไว้) เช่นเขียนแบบนี้:

```rust
async fn login_bad(
    State(state): State<AppState>,
    session: Session,
    Json(body): Json<LoginRequest>,
) -> Result<StatusCode, StatusCode> {
    let users = state.users.lock().unwrap(); // MutexGuard ถูกสร้างตรงนี้ และยังไม่ถูก drop
    if users.get(&body.username) != Some(&body.password) {
        return Err(StatusCode::UNAUTHORIZED);
    }
    session.insert("user_id", body.username.clone()).await.unwrap(); // <- .await ตัวแรก ขณะที่
    // `users` (MutexGuard) ยังอยู่ใน scope
    Ok(StatusCode::OK)
}
```

compiler จะปฏิเสธ (คัดลอก error จริงจาก `cargo build` ในสภาพแวดล้อมที่ใช้ตรวจสอบบทนี้ — ใช้
`#[axum::debug_handler]` เพื่อให้ error message อ่านง่ายขึ้นตามที่ axum แนะนำ):

```
error: future cannot be sent between threads safely
  --> src/bin/mutex_guard_across_await.rs:19:1
   |
19 | #[axum::debug_handler]
   | ^^^^^^^^^^^^^^^^^^^^^^ future returned by `login_bad` is not `Send`
   |
   = help: within `impl Future<Output = Result<StatusCode, StatusCode>>`, the trait `Send` is not
     implemented for `std::sync::MutexGuard<'_, HashMap<std::string::String, std::string::String>>`
note: future is not `Send` as this value is used across an await
  --> src/bin/mutex_guard_across_await.rs:29:54
   |
25 |     let users = state.users.lock().unwrap();
   |         ----- has type `std::sync::MutexGuard<'_, HashMap<std::string::String, std::string::String>>`
   |               which is not `Send`
...
29 |     session.insert("user_id", body.username.clone()).await.unwrap();
   |                                                      ^^^^^ await occurs here, with `users` maybe
   |                                                            used later
```

สาเหตุ (ตามที่ Part 39/46-50 อธิบายไว้): `std::sync::MutexGuard` **ไม่ implement `Send`** ตั้งใจ (เพราะ
mutex ของ OS บางแพลตฟอร์มต้อง unlock บน thread เดียวกันกับที่ lock ไว้) และ future ของ handler async ต้อง
implement `Send` เพื่อให้ tokio runtime ย้ายมันข้าม thread ในตอน scheduling ได้ (multi-thread executor
ของ tokio ที่ axum ใช้เป็นค่าเริ่มต้น) — ถ้า `MutexGuard` ยังอยู่ใน scope ตอนที่ future นั้น `.await` (คือ
suspend แล้วอาจถูก resume บน thread อื่น) ตัว future ทั้งก้อนจะกลาย "ไม่ `Send`" ไปด้วย ทางแก้ (ตามที่โค้ด
ใน 75.5/75.19 ทำไว้แล้ว): จบการใช้งาน `MutexGuard` ให้ครบใน scope ปิดของมันเอง (เช่น สรุปผลเป็น `bool`/
ค่าที่ copy ได้ แล้วให้ `{ }` block ปิด scope ทำให้ guard ถูก `drop` ก่อนถึง `.await` ตัวแรกเสมอ)

### 4. `Secure` cookie ไม่ถูกส่งกลับผ่าน HTTP ธรรมดา — ยกเว้น `localhost`/`127.0.0.1`

หัวข้อ 75.4/75.5 อธิบายไว้ว่า `localhost`/`127.0.0.1` ถูกนับเป็น "potentially trustworthy origin" เป็น
กรณีพิเศษตาม RFC 6265bis — พิสูจน์ผลต่างนี้จริงด้วย `curl` (เซิร์ฟเวอร์เดียวกัน ตั้งค่า `Secure` cookie
เป็นค่า default คือ `true` โดยไม่เรียก `.with_secure(false)`):

```bash
# ยิงตรงไปที่ 127.0.0.1 (นับเป็น localhost) ผ่าน http:// ธรรมดา
$ curl -sS -i -c /tmp/secure_cookies_localhost.txt -X POST http://127.0.0.1:3179/login \
    -H 'Content-Type: application/json' -d '{"username":"nan","password":"hunter2"}'
```
```
HTTP/1.1 200 OK
set-cookie: id=TNYzRawRcZDFfHdTxC-r8A; HttpOnly; SameSite=Strict; Secure; Path=/
```

```bash
$ cat /tmp/secure_cookies_localhost.txt
```
```
#HttpOnly_127.0.0.1	FALSE	/	TRUE	0	id	TNYzRawRcZDFfHdTxC-r8A
```

(field ที่สี่ `TRUE` คือ flag `Secure` — `curl` **บันทึกคุกกี้นี้ไว้จริง**แม้จะได้รับผ่าน `http://` ธรรมดา
เพราะ host เป็น `127.0.0.1`) ทดสอบส่งกลับ:

```bash
$ curl -sS -i -b /tmp/secure_cookies_localhost.txt http://127.0.0.1:3179/me
```
```
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 3

nan
```

ได้ผลลัพธ์ปกติ — session ทำงานได้จริงแม้ cookie มี `Secure` และการเชื่อมต่อเป็น `http://` เพราะ host เป็น
`127.0.0.1` ทีนี้ทำแบบเดียวกัน แต่เปลี่ยนแค่ hostname ที่ใช้ยิง request เป็นชื่ออื่นที่**ไม่ใช่**
`localhost`/`127.0.0.1` (ใช้ `curl --resolve` เพื่อชี้ hostname ปลอมไปยัง server ตัวเดียวกันโดยไม่ต้องแก้
DNS จริง — เซิร์ฟเวอร์ปลายทางเหมือนกันทุกประการ เปลี่ยนแค่ hostname ที่ client มองเห็น):

```bash
$ curl -sS -i --resolve myapp.example.com:3179:127.0.0.1 \
    -c /tmp/secure_cookies_fakehost.txt -X POST http://myapp.example.com:3179/login \
    -H 'Content-Type: application/json' -d '{"username":"nan","password":"hunter2"}'
```
```
HTTP/1.1 200 OK
set-cookie: id=gXvLZs_kLDaKFGuYV0qlgA; HttpOnly; SameSite=Strict; Secure; Path=/
```

```bash
$ cat /tmp/secure_cookies_fakehost.txt
```
```
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

```
(ไฟล์ไม่มีบรรทัดข้อมูล cookie เลย — **`curl` ปฏิเสธที่จะบันทึกคุกกี้ที่มี `Secure` flag เมื่อได้รับผ่าน
`http://` บน host ที่ไม่ใช่ localhost**) ทดสอบเรียกซ้ำ:

```bash
$ curl -sS -i --resolve myapp.example.com:3179:127.0.0.1 \
    -b /tmp/secure_cookies_fakehost.txt http://myapp.example.com:3179/me
```
```
HTTP/1.1 401 Unauthorized
content-length: 0
```

**`401` แม้จะ login "สำเร็จ" ไปแล้วเมื่อกี้** — เพราะไม่มี cookie ให้ส่งกลับไปเลย (ไฟล์คุกกี้ว่างเปล่าตาม
ที่เห็นข้างบน) ผลที่ได้จาก `curl` ตรงกับพฤติกรรมที่ browser จริงทำเช่นกัน (ทั้งคู่ยึดตาม cookie spec
เดียวกัน) — ความหมายเชิงปฏิบัติสำหรับระบบจริง: ถ้า production deploy โดยไม่มี TLS (เสิร์ฟผ่าน `http://`
บน hostname จริง ไม่ใช่ `127.0.0.1`) แต่ session cookie ยังตั้ง `Secure: true` อยู่ (ค่า default ของ
`tower-sessions`) **cookie จะไม่ถูกเก็บ/ส่งกลับโดย client เลยแม้แต่ตัวเดียว** — ระบบจะดูเหมือน "session
ไม่เคย persist" ทุกครั้ง ซึ่ง debug ยากเพราะ error ที่เห็นคือ `401` เฉย ๆ ไม่มี error message ที่บอกสาเหตุ
ที่แท้จริงตรง ๆ (ต้องรู้เรื่อง cookie attribute นี้มาก่อนถึงจะเดาสาเหตุถูก) — ทางแก้ที่ถูกต้องคือให้
production เสิร์ฟผ่าน TLS จริงเสมอ (แล้ว `Secure: true` ทำงานถูกต้องตามที่ตั้งใจ) ไม่ใช่ปิด `Secure` ทิ้ง
เพื่อ "แก้ปัญหา" (การปิด `Secure` ทำให้เสีย protection ต่อ MITM ที่หัวข้อ 75.4 อธิบายไว้ทันที)

### 5. `oauth2` 5.0.0 typestate builder: `EndpointSet`/`EndpointNotSet` mismatch

ถ้าลืมเรียก endpoint ที่จำเป็นตัวใดตัวหนึ่งก่อนเรียกเมธอดที่ต้องใช้มัน (เช่น ลืม `.set_token_uri()` แล้ว
พยายามเรียก `.exchange_code()`) compiler จะปฏิเสธทันทีตามที่หัวข้อ 75.14 อธิบายกลไก typestate ไว้ — ทดสอบ
จริง:

```rust
let client_id = ClientId::new("demo".to_string());
let auth_url = AuthUrl::new("https://github.com/login/oauth/authorize".to_string()).unwrap();

// ตั้งใจ "ลืม" เรียก .set_token_uri() -- BasicClient ตัวนี้จึงมี HasTokenUrl = EndpointNotSet อยู่
let client = BasicClient::new(client_id).set_auth_uri(auth_url);

// .authorize_url() ต้องการแค่ HasAuthUrl = EndpointSet เท่านั้น เรียกได้ปกติ
let (url, _csrf) = client.authorize_url(CsrfToken::new_random).url();

// แต่ .exchange_code() ต้องการ HasTokenUrl = EndpointSet ด้วย -- ยังไม่ได้ตั้ง จึง compile ไม่ผ่าน
let _ = client.exchange_code(oauth2::AuthorizationCode::new("fake".to_string()));
```

error จริงจาก `cargo build`:

```
error[E0599]: no method named `exchange_code` found for struct `Client<StandardErrorResponse<...>, ..., ..., ..., ..., ...>` in the current scope
  --> src/bin/typestate_missing_endpoint.rs:16:20
   |
16 |     let _ = client.exchange_code(oauth2::AuthorizationCode::new("fake".to_string()));
   |                    ^^^^^^^^^^^^^ method not found in `Client<StandardErrorResponse<...>, ..., ..., ..., ..., ...>`
   |
   = note: the method was found for
           - `oauth2::Client<TE, TR, TIR, RT, TRE, HasAuthUrl, HasDeviceAuthUrl, HasIntrospectionUrl, HasRevocationUrl, EndpointMaybeSet>`
           - `oauth2::Client<TE, TR, TIR, RT, TRE, HasAuthUrl, HasDeviceAuthUrl, HasIntrospectionUrl, HasRevocationUrl, EndpointSet>`
```

สังเกตว่า error ไม่ได้บอกตรง ๆ ว่า "ลืมเรียก `.set_token_uri()`" (เพราะ compiler มองเห็นแค่ระดับ type ไม่รู้
เจตนา) แต่บอกว่า **`exchange_code` มีอยู่จริงสำหรับ `Client` ที่ position `HasTokenUrl` เป็น `EndpointSet`
หรือ `EndpointMaybeSet` เท่านั้น** — เมื่อเห็น error รูปแบบนี้ (เมธอดของ `oauth2::Client` ที่ "หาไม่พบ" ทั้ง
ที่เห็นอยู่ใน docs) ให้ตรวจสอบก่อนว่าเรียก `.set_*_uri()` ที่เมธอดนั้นต้องการครบหรือยัง — วิธีป้องกันที่ดี
ที่สุดคือตั้ง type alias อย่างชัดเจนแบบที่หัวข้อ 75.18 ทำ (`type GithubOAuthClient = BasicClient<...>`)
เพราะจะทำให้ error เกิดเร็วขึ้น (ตอน assign เข้าตัวแปรที่มี type annotation) และอ่านง่ายกว่า error ที่เกิด
จากการเรียกเมธอดที่ต้องการ endpoint ที่ยังไม่ตั้ง

### 6. `SameSite=Strict` ทำให้ OAuth2 callback พังทันที — ต้องใช้ `Lax`

`tower-sessions` มีค่า default ของ `SameSite` เป็น **`Strict`** (พิสูจน์ได้จาก `set-cookie` header ที่
ปรากฏจริงในกับดักข้อ 4 ข้างบน: `SameSite=Strict; Secure` — เซิร์ฟเวอร์นั้นไม่ได้เรียก `.with_same_site()`
เลย ค่าที่เห็นคือ default ของ crate เอง) — ถ้าปล่อยค่า default นี้ไว้กับระบบที่มี OAuth2 login (หัวข้อ
75.10-75.19) **ระบบจะพังตรง callback endpoint ทันที** ด้วยเหตุผลทางเทคนิคดังนี้:

`SameSite` cookie attribute ควบคุมว่า browser จะแนบ cookie ไปกับ request ข้าม "site" หรือไม่ (นิยาม
"site" ตาม spec คือ scheme + registrable domain — ไม่รวม subdomain/port) ค่า `Strict` คือระดับเข้มที่สุด:
**cookie จะไม่ถูกแนบไปเลยกับ request ใด ๆ ที่มาจาก navigation ข้าม site แม้จะเป็น top-level navigation
ก็ตาม** (คนละแบบกับ `Lax` ที่ยังยอมให้แนบ cookie กับ top-level `GET` navigation ข้าม site ได้ เช่น การกด
ลิงก์หรือ redirect) — การที่ `github.com` redirect กลับมาที่ `https://yourapp.com/callback` (ขั้นที่ 6
ของ flow ในหัวข้อ 75.12) **คือ cross-site top-level navigation ตามนิยามนี้เป๊ะ**: browser เพิ่งอยู่ที่
`github.com` แล้วถูกสั่งให้ไปที่ `yourapp.com` — เป็นคนละ site กัน

ผลคือ: ถ้า session cookie ที่ `login_github` ตั้งไว้ (เก็บ `state`/`pkce_verifier` — หัวข้อ 75.18) มี
`SameSite=Strict` **browser จะไม่แนบ cookie นั้นไปกับ request ที่ GitHub redirect กลับมาที่ `/callback`
เลย** — ผลคือ `github_callback` handler จะอ่าน `OAUTH_CSRF_KEY` จาก session ไม่เจอ (เพราะ request ที่มา
ถึงไม่มี cookie แนบมา ทำให้ `tower-sessions` มองว่าเป็น session ใหม่ที่ว่างเปล่า) แล้วปฏิเสธด้วย
`CsrfMismatch` (หัวข้อ 75.18) **ทุกครั้ง ไม่มีข้อยกเว้น** แม้ผู้ใช้จะกด "Authorize" ถูกต้องสมบูรณ์ก็ตาม —
นี่คือเหตุผลที่โค้ดในหัวข้อ 75.19 เรียก `.with_same_site(tower_sessions::cookie::SameSite::Lax)` **อย่าง
ตั้งใจ** ไม่ใช่ปล่อยค่า default ไว้: `Lax` ยังคงป้องกัน cross-site `POST` (ที่ CSRF แบบดั้งเดิมกังวล — ดู
หัวข้อ 75.7) ได้เหมือนเดิม เพียงแต่ยอมให้ cross-site top-level `GET` navigation (แบบที่ OAuth2 callback
เป็น) แนบ cookie ไปด้วยได้ — เป็นระดับที่พอดีสำหรับระบบที่มี OAuth2 login (ไม่ใช่ `None` ที่จะปิดการป้องกัน
CSRF ไปเลยทั้งหมด)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** แก้ระบบ session-based login ในหัวข้อ 75.5 ให้ hash password ด้วย `argon2` แทนการเก็บ
   plaintext (ตามที่หมายเหตุในโค้ดเตือนไว้ตลอดบท) — เพิ่ม `argon2` เป็น dependency, เขียนฟังก์ชัน
   `hash_password`/`verify_password`, แล้วปรับ `login` ให้เรียก `verify_password` แทนการเทียบ `String`
   ตรง ๆ *hint*: `argon2::Argon2::default().hash_password(...)`/`.verify_password(...)` ใช้คู่กับ
   `password_hash::SaltString::generate(&mut OsRng)` สำหรับ salt แบบสุ่มใหม่ทุกครั้งที่ hash — เก็บทั้ง
   hash string (มี salt+parameter ฝังอยู่ในตัวมันเองตามรูปแบบ PHC string) ไว้แทน password ดิบใน
   `HashMap`

2. **(กลาง)** เพิ่ม endpoint `GET /login/google` + `GET /callback/google` ใน capstone ของหัวข้อ 75.19
   เพื่อรองรับ "Sign in with Google" เพิ่มอีกหนึ่งทาง ควบคู่กับ GitHub ที่มีอยู่แล้ว โดยไม่แก้
   `establish_local_session` เลย (ใช้ตัวเดิม ส่ง `method = "google_oauth"`) *hint*: Google OAuth2 มี
   `auth_uri`/`token_uri` คนละ URL จาก GitHub (ค้นหาจาก Google OAuth2 documentation) แต่โครงสร้างโค้ด
   ฝั่ง Rust (PKCE, state, `exchange_code`) เหมือนกันทุกประการ ต่างกันแค่ endpoint URL และ scope ที่ขอ
   — ถ้าต้องการดึง email ด้วย ต้อง decode ID token แบบ OIDC (หัวข้อ 75.10 พูดถึงไว้ผ่าน ๆ) ไม่ใช่แค่
   access token เฉย ๆ แบบ GitHub

3. **(ยาก)** ปรับ `SessionManagerLayer` ในหัวข้อ 75.8/75.19 ให้ใช้ `RedisStore` แทน `MemoryStore` จริง
   (ตั้ง Redis ด้วย `docker run -p 6379:6379 redis` หรือเทียบเท่า) แล้วรัน capstone สอง instance พร้อมกัน
   บนสองพอร์ต (เช่น `3178`/`3180`) พิสูจน์ด้วย `curl` ว่า login ที่ instance หนึ่งแล้วเรียก `/me` ที่อีก
   instance ด้วย cookie เดิม**สำเร็จ** (คนละกับพฤติกรรมของ `MemoryStore` ที่หัวข้อ 75.8 อธิบายไว้ว่าจะ
   ล้มเหลว) *hint*: ใช้ crate `tower-sessions-redis-store` กับ `fred` เป็น Redis client ตามที่หัวข้อ
   75.8 ระบุไว้ — ทั้งสอง instance ต้องชี้ไปที่ Redis ตัวเดียวกัน (connection string เดียวกัน) และโค้ด
   handler ทั้งหมดไม่ต้องแก้เลยแม้แต่บรรทัดเดียว (ตามที่หัวข้อ 75.8 อธิบายไว้ว่า `SessionStore` เป็น
   trait — เปลี่ยนแค่ตัวที่ผ่านเข้า `SessionManagerLayer::new(...)`)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ออกแบบและ implement ระบบที่ endpoint `POST /login` (traditional) และ
   `GET /callback` (OAuth2) **ทั้งคู่ออก JWT แทน session** (ไม่ใช่ `tower-sessions` เลย) โดยยังคง
   pattern การแยก "พิสูจน์ตัวตน" ออกจาก "จดจำสถานะ" ตามที่หัวข้อ 75.20 อธิบายไว้ (เขียนฟังก์ชัน
   `issue_jwt(user_id, username, auth_method)` แทนที่ `establish_local_session` แล้วให้ทั้งสอง handler
   เรียกฟังก์ชันนี้แทน) — protected route (`/me`) ต้องอ่าน JWT จาก `Authorization: Bearer` header (ตาม
   ที่ Part 74 สอน) ไม่ใช่จาก cookie *hint*: ทบทวนโครงสร้าง claims (`sub`/`exp`/claim อื่น ๆ) และการเซ็น/
   ตรวจสอบ JWT จาก Part 74 — ส่วนที่ต้องคิดเพิ่มเองคือ: `state`/`pkce_verifier` ของ OAuth2 flow (หัวข้อ
   75.15-75.16) ยังต้องเก็บไว้ที่ไหนสักที่ระหว่าง `/login/github` กับ `/callback` (สอง request คนละครั้ง)
   — ถ้าไม่มี session แล้วจะเก็บค่าชั่วคราวนี้ยังไง? (คำตอบที่เป็นไปได้หนึ่งทาง: signed short-lived cookie
   แยกเฉพาะสำหรับ flow นี้ ที่ไม่ใช่ session เต็มรูปแบบ)

## สรุป

บทนี้ครอบคลุมสองครึ่งของ authentication ฝั่งเว็บที่เป็นคู่ตรงข้ามกับ JWT ของ Part 74 บนมิติ
stateless/stateful — **ครึ่งแรก (75.1-75.9)**: session-based authentication ด้วย `tower-sessions` ตั้งแต่
แนวคิดพื้นฐาน (session ID เป็นแค่ "ใบเสร็จ" ที่ข้อมูลจริงอยู่ที่ server), cookie attribute สามตัวที่สำคัญ
ต่อความปลอดภัย (`HttpOnly`/`Secure`/`SameSite`) พร้อมพิสูจน์ด้วย `curl` จริงทุกจุด, session fixation และ
`cycle_id()`, CSRF ที่ session เสี่ยงกว่า bearer token และการป้องกันทั้งสองชั้น (`SameSite` + CSRF token
แบบดั้งเดิม), ไปจนถึงข้อจำกัดเรื่อง scale ที่ต้องแก้ด้วย shared session store (เชื่อมไป Part 83) —
**ครึ่งหลัง (75.10-75.19)**: OAuth2 Authorization Code Flow เต็มรูปแบบด้วย crate `oauth2` 5.0.0 กับ GitHub
OAuth App จริง เข้าใจว่า OAuth2 คือ delegated authorization ไม่ใช่ authentication โดยตรง (OIDC คือชั้นที่
เติมให้เป็น authentication มาตรฐาน), typestate pattern ของ `BasicClient`, PKCE และ `state` parameter สอง
กลไกป้องกันที่จำเป็นสำหรับ flow นี้ ไปจนถึงการประกอบ callback handler เต็มรูปแบบและรวมทั้งสองครึ่งเข้าเป็น
แอปเดียวที่จบลงที่ local session รูปแบบเดียวกันไม่ว่า user จะ login มาทางไหน — และปิดท้ายด้วยหลักการสำคัญ
ที่สุดของบท (75.20): **แยก "วิธีพิสูจน์ตัวตน" ออกจาก "วิธีจดจำสถานะ login"** ให้เป็นสองเรื่องที่ไม่ผูกติด
กันตั้งแต่ระดับโค้ด เพราะทั้งสองมิติ (JWT/Session และ OAuth2/password) เลือกผสมกันได้อิสระตามความต้องการ
ของระบบจริง

ตอนนี้คุณรู้แล้วว่า **user คนนี้คือใคร** (authentication — ทั้ง Part 74 และบทนี้) แต่ยังไม่มีกลไกใดตอบคำถาม
ที่สำคัญไม่แพ้กัน: **user คนนี้ทำอะไรได้บ้าง** — user ทั่วไปกับ admin ควรเข้าถึง endpoint ต่างกัน,
resource ของ user คนหนึ่งไม่ควรให้ user อีกคนแก้ไขได้แม้จะ authenticate สำเร็จเหมือนกัน — **Part 76
(Authorization และ RBAC)** จะสร้างต่อจากรากฐานที่บทนี้และ Part 74 วางไว้ ไปสู่การออกแบบระบบสิทธิ์
(role-based access control) เต็มรูปแบบ

---

**Part ก่อนหน้า:** [Authentication: JWT](part-074-jwt-authentication.md) | **Part ถัดไป:** [Authorization และ RBAC](part-076-authorization-rbac.md)
