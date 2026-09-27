# Part 93: Full-Stack Project (2/3): สร้าง Frontend

> โมดูล: โปรเจกต์ Capstone (Full-Stack) | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- สร้างโปรเจกต์ **Leptos แบบ CSR (Client-Side Rendering) ล้วน ๆ** ด้วย `trunk` ที่แยกจาก backend ของ
  Part 92 โดยสิ้นเชิง (คนละ process, คนละ origin, คนละ repository ในโลกจริงก็ได้) และอธิบายได้ชัดว่าทำไม
  capstone นี้**ไม่ใช้** server function (`#[server]`) ของ Leptos ที่ Part 89 สอนไว้ — เพราะ server
  function ผูกกับสมมติฐานว่า frontend/backend คือ Axum process เดียวกัน ในขณะที่โจทย์จริงของทีม frontend
  คือการเรียก REST API ที่ทีม backend "ล็อก" ไว้แล้วผ่าน HTTP ธรรมดา ตรงกับสถานการณ์ทำงานจริงที่ทีม
  frontend แยกจากทีม backend
- เขียน **HTTP client module** ที่ครอบ `gloo-net` (crate เดียวกับที่ Part 88 ใช้กับ Yew) ให้เรียก
  endpoint ทั้ง 10 ตัวของ Part 92 ได้ตรงตาม contract ทุกประการ — ทุก request/response shape, ทุก query
  parameter, ทุก error shape — พร้อม type ที่ deserialize จาก JSON จริงของ backend โดยไม่ต้องเดาโครงสร้าง
- จัดการ **สถานะการ login** ด้วย Leptos signal ที่ผูกกับ context (`provide_context`/`use_context` จาก
  Part 89) ควบคู่กับการเก็บ JWT ลง `localStorage` ผ่าน `gloo-storage` (ต่อยอด Part 87 เรื่อง wasm-bindgen/
  web-sys) ให้ session รอดจาก page reload ได้จริง และตรวจสอบ token ที่เก็บไว้กับ backend จริงทุกครั้งที่
  แอปเริ่มทำงาน
- สร้างหน้า login/register/รายการหนังสือ/รายละเอียดหนังสือ/หนังสือที่ยืมอยู่/สร้างหนังสือ (admin) ด้วย
  `leptos_router` พร้อม **route guard** ที่ redirect ผู้ใช้ที่ยังไม่ login หรือไม่ใช่ admin ออกจากหน้าที่
  ไม่มีสิทธิ์เข้าโดยอัตโนมัติ
- แสดง **loading state** ด้วย `<Suspense>`/`Action::pending()` และแสดง **error จริงจาก backend** —
  ทั้ง error message รวม, field-level validation error (`400` พร้อม `fields[]`), และ error ที่มีความหมาย
  ทางธุรกิจเฉพาะ (`409` ไม่มีสำเนาว่าง, `403` ไม่ใช่ admin) — ให้ผู้ใช้อ่านเข้าใจได้จริง ไม่ใช่แค่โชว์ debug
  string ดิบ ๆ
- อธิบายและแก้ปัญหา **`Send` bound ของ `Action::new`/`Resource::new`** เมื่อ future มาจาก `gloo-net` (ซึ่ง
  ห่อ `JsFuture` ที่ผูกกับเธรดเดียวของเบราว์เซอร์) ได้ถูกต้องด้วย `Action::new_local`/`LocalResource::new`
  ตามที่ Part 89 หัวข้อ 89.6 เกริ่นไว้แล้วว่าจะเจอปัญหานี้ในแอปจริงที่ไม่ได้ใช้ server function
- **รันทั้งสองระบบพร้อมกันจริง** (backend ของ Part 92 ที่ `127.0.0.1:8092` + frontend ของบทนี้ที่
  `127.0.0.1:5173`) แล้วพิสูจน์ด้วย headless Chromium ว่า flow เต็มรูปแบบ — สมัคร → login → ดูรายการ
  หนังสือ → ยืม → ดูในหน้า "ที่ยืมอยู่" → คืน → admin สร้างหนังสือใหม่ — ทำงานจริงจาก WASM ที่ compile จาก
  โค้ดในบทนี้ ไปเรียก backend ของ Part 92 ผ่าน network request จริง ข้าม origin จริง ผ่าน CORS จริง

## ความรู้ที่ต้องมีมาก่อน

บทนี้เป็น **capstone บทที่ 2 จาก 3 บท** — มันไม่แนะนำ Leptos ใหม่ตั้งแต่ต้น (Part 89 ทำไปแล้วอย่างละเอียด)
แต่ใช้ทุกอย่างที่ Part 89 สอนไว้เป็น "ไวยากรณ์พื้นฐาน" แล้วเอามาประกอบเป็นแอปจริงที่คุยกับ backend จริงของ
Part 92 — ความรู้ที่ต้องแน่นก่อนอ่านบทนี้:

- **Part 92 (Full-Stack Project 1/3: Backend) — บทที่บทนี้ต่อยอดตรง ๆ ที่สุด**: บทนี้เรียก 10 endpoint ที่
  หัวข้อ 92.1 กำหนดไว้ **ตรงตามที่ระบุไว้ทุกประการ ไม่มีการเปลี่ยนแปลง** ทั้ง path, request/response shape,
  status code, และ error shape (`{"error":{"code","message","fields"?}}`) — ถ้าจำ contract ของ Part 92
  ไม่ชัด ต้องกลับไปอ่านหัวข้อ 92.1 ก่อน เพราะบทนี้จะไม่ทวนซ้ำทั้งตาราง (สรุปย่อไว้ในหัวข้อ 93.1 เท่านั้น)
- **Part 89 (Leptos Framework)**: signal (`signal()`/`RwSignal::new()`), `#[component]`, `view!` macro,
  `Action::new`, `Resource::new`/`LocalResource::new` + `<Suspense>`, `provide_context`/`use_context`,
  `leptos_router` (`Router`/`Routes`/`Route`/`path!`/`use_params_map`) — บทนี้ใช้ API เหล่านี้ทั้งหมดแบบ
  ที่ Part 89 สอนไว้เป๊ะ เพียงแต่เอามาแก้โจทย์ CSR ที่คุยกับ REST API แยก process แทนการเรียก server
  function ในโปรเจกต์เดียวกัน
- **Part 90 (Dioxus Framework)**: หัวข้อ 90.10 ตัดสินไว้แล้วว่าหลักสูตรนี้เลือก **Leptos** สำหรับ capstone
  (เพราะเป็น full-stack web application ที่ตรงกับจุดแข็งของ Leptos ที่สุด) — บทนี้เป็นผลจากการตัดสินใจนั้น
- **Part 88 (Yew Framework)**: บทนี้ใช้ `gloo-net` สำหรับเรียก HTTP เหมือนที่ Part 88 สอนไว้ทุกประการ
  (`Request::get`/`Request::post`, `.json()`, `.send()`) เพียงแค่เปลี่ยนจาก Yew มาเป็น Leptos ครอบด้านนอก
  — ถ้าจำเหตุผลที่ Part 88 เลือก `gloo-net` แทน `reqwest` ไม่ชัด (เพราะ WASM ในเบราว์เซอร์ต้องเรียกผ่าน
  `fetch` ของ browser เท่านั้น ไม่มีสิทธิ์เปิด raw socket) ควรทวนหัวข้อ 88.8 ก่อน
- **Part 87 (wasm-bindgen และ JavaScript Interop)**: `localStorage` ที่บทนี้ใช้เก็บ session ต่อยอดจาก
  `web-sys`/`wasm-bindgen` ที่ Part 87 สอนไว้ตรง ๆ (ผ่าน `gloo-storage` ที่ครอบ `web_sys::Storage` ให้
  เขียน/อ่านเป็น JSON ได้สะดวกขึ้น)
- **Part 78 (REST API Design)**: pagination (`page`/`per_page`) และ response shape ที่มี `data`+`meta` ที่
  บทนี้ต้อง parse และแสดงเป็น UI ปุ่ม "หน้าก่อนหน้า/หน้าถัดไป" จริง
- **Part 76 (Authorization: RBAC)**: บทนี้ตรวจ `role` จาก `UserResponse`/`/api/v1/me` เพื่อซ่อน/แสดง UI
  ของ admin ฝั่ง client — และย้ำเสมอว่า**นี่เป็นแค่ UX** การตรวจสิทธิ์จริงเกิดที่ backend เท่านั้น
- **Part 61-66 (HTTP Fundamentals ถึง Axum Error Handling)**: บทนี้ deserialize error shape ของ Part 66
  (`{"error":{"code","message"}}`) ฝั่ง client ให้ตรงกัน

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้เป็นภาคต่อของ capstone ที่ Part 92 เริ่มไว้ — เพื่อให้ Part 94 (deployment) ที่จะตามมาสามารถอ่านบทนี้
แล้วเชื่อมทุกอย่างเข้าด้วยกันได้จริง ทุกโค้ด ทุก error message ทุกผลลัพธ์ในบทนี้**มาจากการสร้างโปรเจกต์ Rust
จริงสองโปรเจกต์ (backend ของ Part 92 คัดลอกมารันจริง + frontend ของบทนี้) รันคู่กันจริงบนเครื่อง แล้ว
capture ผลลัพธ์มาทั้งหมด**:

- รัน backend ของ Part 92 จริงที่ `http://127.0.0.1:8092` ต่อกับ PostgreSQL 16 จริง (database
  `library_api_dev`) — เหมือนที่ Part 92 ทดสอบไว้ทุกประการ
- สร้างโปรเจกต์ Leptos CSR จริงด้วย `trunk`, compile เป็น `wasm32-unknown-unknown` จริง, รันด้วย
  `trunk serve --port 5173` ที่ `http://127.0.0.1:5173` (ตรงกับค่า default ของ `FRONTEND_ORIGIN` ที่ Part
  92 ตั้ง CORS ไว้รอแล้ว)
- เปิด **headless Chromium จริงผ่าน Playwright** ไปที่หน้าเว็บ แล้วขับ flow เต็มรูปแบบ — กรอกฟอร์มจริง,
  คลิกปุ่มจริง, อ่าน DOM จริง, ดัก network request/response จริงที่วิ่งระหว่าง `5173` กับ `8092` — ผลลัพธ์
  ทุกอันที่ปรากฏในบทนี้ (JSON body, HTTP status, ข้อความ error, DOM text) **คัดลอกมาจาก terminal จริง**
  ไม่มีการแต่งขึ้นเองแม้แต่จุดเดียว รวมถึง compile error จริงที่เจอระหว่างพัฒนา (ดูหัวข้อกับดักที่พบบ่อย)
- โปรเจกต์ scratch ทั้งสองที่ใช้ทดสอบถูกลบทิ้งหลังตรวจสอบเสร็จ (ไม่กระทบไฟล์ใด ๆ ในหลักสูตร) แต่**โค้ดทุก
  ไฟล์ในบทนี้คือโค้ดตัวจริงที่ compile ผ่านและรันได้ 100%** ผู้อ่านสามารถคัดลอกไปสร้างโปรเจกต์เดียวกันขึ้นมา
  ใหม่ได้ทันที (คู่กับ backend จาก Part 92)

## เนื้อหา

### 93.1 ทวนสัญญา (Contract) จาก Part 92 และขอบเขตของบทนี้

ก่อนเขียนโค้ดสักบรรทัด ต้องย้ำก่อนว่า **บทนี้ไม่มีสิทธิ์เปลี่ยน path, shape, หรือ status code ใด ๆ ที่ Part
92 กำหนดไว้แล้ว** — backend ที่หัวข้อ 92.1 ล็อกไว้คือความจริงตายตัว (source of truth) หน้าที่ของบทนี้คือ
เขียน frontend ที่**เชื่อฟัง**สัญญานั้นทั้งหมด สรุปย่อสิ่งที่ต้องใช้ตรง ๆ จากบทนั้น (รายละเอียดเต็มอยู่ใน
Part 92 หัวข้อ 92.1):

| # | Method | Path | Auth? | ใช้ที่หน้าไหนของบทนี้ |
|---|--------|------|-------|------------------------|
| 1 | GET | `/health` | ไม่ | (ใช้จริงใน Part 94 สำหรับ health check) |
| 2 | POST | `/api/v1/auth/register` | ไม่ | หน้า Register |
| 3 | POST | `/api/v1/auth/login` | ไม่ | หน้า Login |
| 4 | GET | `/api/v1/me` | ต้อง | ตรวจ session ตอน app เริ่มทำงาน |
| 5 | GET | `/api/v1/books` | ไม่ | หน้ารายการหนังสือ (pagination/filter) |
| 6 | GET | `/api/v1/books/{id}` | ไม่ | หน้ารายละเอียดหนังสือ |
| 7 | POST | `/api/v1/books` | ต้อง+admin | หน้าสร้างหนังสือ (admin เท่านั้น) |
| 8 | POST | `/api/v1/books/{id}/borrow` | ต้อง | ปุ่ม "ยืม" ที่หน้ารายการ/รายละเอียด |
| 9 | POST | `/api/v1/books/{id}/return` | ต้อง | ปุ่ม "คืน" ที่หน้า "ที่ยืมอยู่" |
| 10 | GET | `/api/v1/me/borrowed` | ต้อง | หน้า "หนังสือที่ยืมอยู่" |

Base URL คือ `http://127.0.0.1:8092` (ค่า default ของ `BIND_ADDR` ใน Part 92), การยืนยันตัวตนใช้ header
`Authorization: Bearer <access_token>`, access token มีอายุ 3600 วินาที ไม่มี refresh token ในสโคปนี้, และ
ทุก error ตอบเป็น `{"error":{"code","message","fields"?}}` — สี่ข้อนี้คือสิ่งที่ HTTP client module ในหัวข้อ
93.3 ต้องเคารพทุกจุด

#### ทำไมบทนี้ไม่ใช้ server function ของ Leptos (Part 89 หัวข้อ 89.6-89.8)

นี่คือคำถามที่ต้องตอบให้ชัดก่อนเริ่ม เพราะ Part 89 สอนว่า server function เป็น "จุดขายหลัก" ของ Leptos —
แล้วทำไมบทนี้ (ซึ่งเป็น Leptos เหมือนกัน) ไม่ใช้มัน?

server function ของ Leptos (`#[server]`) ทำงานได้เพราะมันสร้างโค้ด**สองด้าน**ให้อัตโนมัติจาก function
เดียว: ด้าน server (compile ด้วย feature `ssr` กลายเป็น handler ของ Axum จริง ๆ) และด้าน client (compile
ด้วย feature `hydrate`/`csr` กลายเป็นโค้ดที่ยิง HTTP request ไปยัง endpoint ที่สร้างขึ้นเอง) — กลไกนี้
สมมติไว้เสมอว่า **backend คือ Leptos/Axum ตัวเดียวกันกับที่จะ serve หน้าเว็บนี้** (ตามที่ Part 89 หัวข้อ
89.8 ผสาน Leptos SSR เข้ากับ Axum Router เดียวกัน)

แต่โจทย์ของ capstone นี้คือสถานการณ์ที่พบบ่อยกว่าในโลกทำงานจริง: **backend ของ Part 92 คือ Axum project
แยกที่ล็อก contract ไว้แล้ว** (utoipa สร้าง Swagger UI ของมันเอง, มี integration test ของมันเอง, deploy
แยกจาก frontend ได้) ทีม frontend (บทนี้) ไม่ได้เขียน handler ฝั่ง server เลย มีแต่ REST API ที่ต้องเรียก
ผ่าน HTTP ธรรมดาเท่านั้น — นี่คือสถานการณ์เดียวกับที่ Part 88 (Yew) เจอตอนต่อกับ Axum API แยก และเป็น
สถานการณ์ที่ `#[server]` **ใช้ไม่ได้เลย** เพราะไม่มี endpoint ให้ macro สร้างขึ้นมาคู่กัน (endpoint ทั้งหมด
ถูกสร้างไว้แล้วใน Part 92 ด้วยมือ ไม่ใช่ macro)

ดังนั้นบทนี้จึงตั้งโปรเจกต์ Leptos แบบ **CSR ล้วน ๆ** (feature `csr` เท่านั้น ไม่มี `ssr`/`hydrate`) แล้วเขียน
HTTP client module ด้วยมือด้วย `gloo-net` — เหมือนกับที่ Part 88 (Yew) ทำ เพียงแต่เปลี่ยนตัว UI framework
จาก Yew มาเป็น Leptos (signal/component/`view!` ของ Part 89) — นี่คือจุดที่ทำให้เห็นภาพชัดว่า **Leptos ไม่ใช่
"framework ที่ทำงานได้เฉพาะกับ server function"** มันใช้เป็น CSR framework ล้วน ๆ ได้เหมือนกับ Yew ทุก
ประการ (ตามที่ Part 89 หัวข้อ 89.2 "แนวทางที่ 1" เกริ่นไว้)

### 93.2 Setup โปรเจกต์ Leptos CSR ด้วย `trunk`

ต่อยอดจาก Part 89 หัวข้อ 89.2 "แนวทางที่ 1 — CSR ล้วน ๆ" ทุกประการ (และเหมือนกับที่ Part 88 ตั้ง `trunk`
สำหรับ Yew) เพราะ CSR ของทั้งสอง framework compile เป็น `wasm32-unknown-unknown` เหมือนกัน `trunk` จึงไม่
สนใจว่าโค้ดข้างในเป็น Yew หรือ Leptos:

```bash
cargo new --lib library_frontend
cd library_frontend

cargo add leptos --features csr
cargo add leptos_router
cargo add gloo-net --features json
cargo add gloo-storage
cargo add wasm-bindgen-futures
cargo add serde --features derive
cargo add serde_json
cargo add console_error_panic_hook

rustup target add wasm32-unknown-unknown
cargo install trunk --locked
```

`Cargo.toml` ที่ได้ (เวอร์ชันที่ตรวจสอบจริงบนเครื่องที่เขียนบทนี้ — `trunk 0.21.14`, `wasm-bindgen-cli
0.2.129` เวอร์ชันเดียวกับที่ Part 88 ใช้ทุกประการ เพื่อให้ `wasm-bindgen` crate กับ CLI ตรงกันตามที่ Part 88
หัวข้อ 88.3 เตือนไว้):

```toml
# Cargo.toml
[package]
name = "library_frontend"
version = "0.1.0"
edition = "2021"

[dependencies]
leptos = { version = "0.8.21", features = ["csr"] }
leptos_router = "0.8.16"
gloo-net = { version = "0.6.0", features = ["json"] }
gloo-storage = "0.3.0"
wasm-bindgen-futures = "0.4.79"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
console_error_panic_hook = "0.1.7"
```

สังเกตว่า `leptos` เปิด**เฉพาะ** feature `csr` — ไม่มี `ssr`/`hydrate` เลย นี่คือความต่างที่ชัดเจนที่สุด
เทียบกับ Part 89 หัวข้อ 89.8 (ที่เปิดทั้งสอง feature สำหรับ full-stack SSR) เพราะโปรเจกต์นี้**ไม่มีด้าน
server ของ Leptos อยู่เลย** มันคือ SPA ล้วน ๆ ที่ compile เป็น WASM ชิ้นเดียวแล้วรันในเบราว์เซอร์ทั้งหมด

`Trunk.toml` (ตั้ง dev server ให้ตรงกับพอร์ตที่ Part 92 อนุญาตผ่าน `FRONTEND_ORIGIN` ไว้แล้ว):

```toml
# Trunk.toml
[build]
target = "index.html"

[serve]
address = "127.0.0.1"
port = 5173
```

`index.html` (เหมือนกับที่ Part 89 หัวข้อ 89.2 พิสูจน์ไว้ — `trunk` ตรวจจับ `Cargo.toml` ข้าง ๆ ไฟล์นี้ได้
อัตโนมัติ ไม่จำเป็นต้องมี `<link data-trunk rel="rust" />` เสมอไป):

```html
<!DOCTYPE html>
<html lang="th">
  <head>
    <meta charset="utf-8" />
    <title>Library Frontend</title>
  </head>
  <body></body>
</html>
```

#### โครงสร้างโปรเจกต์: แยกตามความรับผิดชอบเหมือนที่ Part 92 แยก backend

ต่อยอดหลักการแบ่ง module ของ Part 16-17 (แบ่งตาม**ความรับผิดชอบ** ไม่ใช่ขนาดไฟล์) ที่ Part 92 หัวข้อ 92.2
ใช้กับ backend — บทนี้ใช้หลักการเดียวกันกับ frontend:

```
library_frontend/
├── Cargo.toml
├── Trunk.toml
├── index.html
└── src/
    ├── main.rs              -- entry point: provide_auth_context, ตรวจ session, ประกอบ Router
    ├── api/
    │   ├── mod.rs
    │   ├── types.rs          -- struct ที่ตรงกับ shape ของ Part 92 ทุกตัว (request/response/error)
    │   └── client.rs          -- ฟังก์ชันเรียก endpoint ทั้ง 10 ตัวด้วย gloo-net
    ├── auth.rs                 -- Session, AuthContext (signal + localStorage)
    ├── components/
    │   ├── mod.rs
    │   └── nav.rs                -- แถบนำทางบนสุด แสดงสถานะ login/role
    └── pages/
        ├── mod.rs
        ├── login.rs                -- หน้า login
        ├── register.rs               -- หน้าสมัครสมาชิก
        ├── books.rs                    -- หน้ารายการหนังสือ (pagination/filter/ยืม)
        ├── book_detail.rs                -- หน้ารายละเอียดหนังสือ + ยืม
        ├── my_borrowed.rs                  -- หน้า "หนังสือที่ยืมอยู่" + คืน
        └── create_book.rs                    -- หน้าสร้างหนังสือ (admin เท่านั้น)
```

**เหตุผลของการแบ่งแบบนี้**: `api/` แยก `types.rs` (struct ล้วน ๆ ไม่มี logic) จาก `client.rs` (ฟังก์ชัน
เรียก HTTP จริง) ด้วยเหตุผลเดียวกับที่ Part 92 แยก `models/` จาก `routes/` — struct ที่แทน shape ของข้อมูล
ถูกใช้ทั้งใน `client.rs`, ใน `pages/` (สำหรับ render), และอาจถูกใช้ใน test ในอนาคต ถ้าฝัง struct ไว้ในไฟล์
เดียวกับ logic การเรียก HTTP จะทำให้ไฟล์นั้นโตขึ้นเรื่อย ๆ โดยไม่มีเหตุผล — `auth.rs` เป็นไฟล์เดียว (ไม่ใช่
module ย่อย) เพราะ logic การจัดการ session ทั้งหมด (signal + localStorage) เกี่ยวเนื่องกันแน่นมาก การแยกจะ
ทำให้ไล่โค้ดยากขึ้นโดยไม่ได้ประโยชน์ — `pages/` มีไฟล์ละหนึ่งหน้าตาม route ที่ `leptos_router` จะประกาศใน
`main.rs` หัวข้อ 93.5

### 93.3 API Client Module: ครอบ `gloo-net` ให้ตรง Contract ของ Part 92 ทุกตัว

เริ่มจาก `types.rs` — struct ทุกตัวในไฟล์นี้ต้อง**ตรงกับ shape ของ Part 92 หัวข้อ 92.1 ทุกฟิลด์** ไม่มีการ
เพิ่ม/ลด field ใด ๆ ที่ backend ไม่มี (field ที่ backend ไม่ส่งมาจะ deserialize fail ทันทีถ้า struct
ประกาศไว้แบบไม่ optional — เป็นประโยชน์เพราะ **compiler จะจับความไม่ตรงกันของ contract ให้เราทันที** แทนที่
จะปล่อยให้ bug เงียบ ๆ ไปจนถึง runtime):

```rust
// src/api/types.rs
use serde::{Deserialize, Serialize};

// ---- Request types (ส่งไปให้ backend — ต้อง Serialize) ----

#[derive(Debug, Clone, Serialize)]
pub struct RegisterRequest {
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(Debug, Clone, Serialize)]
pub struct LoginRequest {
    pub username: String,
    pub password: String,
}

#[derive(Debug, Clone, Serialize)]
pub struct CreateBookRequest {
    pub title: String,
    pub author: String,
    pub isbn: String,
    pub category: String,
    pub total_copies: i32,
}

// ---- Response types (รับมาจาก backend — ต้อง Deserialize) ----
// UserResponse ต้อง Serialize ด้วย (ไม่ใช่แค่ Deserialize) เพราะหัวข้อ 93.4 ต้องเก็บมันลง
// localStorage ผ่าน gloo-storage ซึ่งต้อง serialize ก่อนเขียนลง storage

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct UserResponse {
    pub id: i64,
    pub username: String,
    pub email: String,
    /// "member" | "admin" ตรงกับ Part 92 หัวข้อ 92.1 — เก็บเป็น String ธรรมดา ไม่ทำเป็น enum
    /// เพราะ frontend ไม่ควร "ตัดสินใจ" ว่า role มีค่าอะไรได้บ้าง หน้าที่นั้นเป็นของ backend
    /// (ตาราง users ของ Part 92 หัวข้อ 92.3 มี CHECK constraint คุมค่าไว้แล้วที่ระดับฐานข้อมูล)
    pub role: String,
    pub created_at: String,
}

#[derive(Debug, Clone, Deserialize)]
pub struct LoginResponse {
    pub access_token: String,
    pub token_type: String,
    pub expires_in: i64,
    pub user: UserResponse,
}

#[derive(Debug, Clone, Deserialize, PartialEq)]
pub struct BookResponse {
    pub id: i64,
    pub title: String,
    pub author: String,
    pub isbn: String,
    pub category: String,
    pub total_copies: i32,
    pub available_copies: i32,
    pub created_at: String,
}

#[derive(Debug, Clone, Deserialize)]
pub struct PaginationMeta {
    pub page: i64,
    pub per_page: i64,
    pub total_items: i64,
    pub total_pages: i64,
}

#[derive(Debug, Clone, Deserialize)]
pub struct BookListResponse {
    pub data: Vec<BookResponse>,
    pub meta: PaginationMeta,
}

#[derive(Debug, Clone, Deserialize, PartialEq)]
pub struct BorrowResponse {
    pub id: i64,
    pub book_id: i64,
    pub user_id: i64,
    pub borrowed_at: String,
    pub due_at: String,
    pub returned_at: Option<String>,
}

#[derive(Debug, Clone, Deserialize)]
pub struct BorrowListResponse {
    pub data: Vec<BorrowResponse>,
}

/// พารามิเตอร์ของ GET /api/v1/books — ทุกตัว optional ตาม contract ของ Part 92 หัวข้อ 92.1
#[derive(Debug, Clone, Default)]
pub struct ListBooksParams {
    pub page: Option<i64>,
    pub per_page: Option<i64>,
    pub category: Option<String>,
    pub q: Option<String>,
    pub available_only: Option<bool>,
}

// ---- Error types: ตรงกับ error shape ของ Part 92 หัวข้อ 92.4/92.1 เป๊ะ ----

/// field-level validation error หนึ่งจุด — shape ตรงกับ FieldError ของ Part 92 หัวข้อ 92.4
#[derive(Debug, Clone, Deserialize)]
pub struct ApiFieldError {
    pub field: String,
    pub message: String,
}

#[derive(Debug, Clone, Deserialize)]
pub struct ApiErrorBody {
    pub code: String,
    pub message: String,
    // "fields" ปรากฏเฉพาะตอน code == "VALIDATION_ERROR" ตาม Part 92 หัวข้อ 92.1 — #[serde(default)]
    // ทำให้ deserialize สำเร็จแม้ response ไม่มี key นี้เลย (กลายเป็น None แทน error)
    #[serde(default)]
    pub fields: Option<Vec<ApiFieldError>>,
}

#[derive(Debug, Clone, Deserialize)]
pub struct ApiErrorEnvelope {
    pub error: ApiErrorBody,
}

/// ผลลัพธ์ error ทุกแบบที่ฟังก์ชันเรียก API ในไฟล์นี้คืนกลับได้ — ครอบทั้ง error จาก backend จริง
/// (shape ตรง ApiErrorEnvelope ของ Part 92) และ error จากตัว fetch เอง (ต่อเซิร์ฟเวอร์ไม่ได้/parse ไม่ได้)
/// การแยกสามกรณีนี้ชัด ๆ สำคัญมาก — ผู้ใช้ที่เห็น "เชื่อมต่อเซิร์ฟเวอร์ไม่ได้" ควรรู้ว่าปัญหาไม่ใช่ที่ตัวเอง
/// (เช่น backend ล่ม/ปิด) ในขณะที่ผู้ใช้ที่เห็น "409 CONFLICT: ไม่มีสำเนาว่าง" ควรรู้ว่านี่คือกฎธุรกิจ
#[derive(Debug, Clone)]
pub enum ApiError {
    /// backend ตอบกลับมาพร้อม HTTP status และ error envelope ตาม shape ของ Part 92 หัวข้อ 92.1
    Api { status: u16, body: ApiErrorBody },
    /// เชื่อมต่อเซิร์ฟเวอร์ไม่ได้เลย (เซิร์ฟเวอร์ล่ม, CORS บล็อก, network error) — ไม่มี HTTP response ให้ parse
    Network(String),
    /// ได้ response กลับมาแต่ body ไม่ใช่ JSON ที่ deserialize เป็น type ที่คาดไว้ได้
    Decode(String),
}

impl ApiError {
    /// ข้อความที่ปลอดภัยและอ่านง่ายพอจะโชว์ตรง ๆ ใน UI — ไม่โชว์ debug info ดิบให้ผู้ใช้เห็น
    pub fn user_message(&self) -> String {
        match self {
            ApiError::Api { status, body } => format!("{status} {}: {}", body.code, body.message),
            ApiError::Network(msg) => format!("เชื่อมต่อเซิร์ฟเวอร์ไม่ได้: {msg}"),
            ApiError::Decode(msg) => format!("อ่านข้อมูลจากเซิร์ฟเวอร์ไม่ได้: {msg}"),
        }
    }

    /// field errors ถ้ามี (มีเฉพาะตอน code == "VALIDATION_ERROR" ตาม contract ของ Part 92)
    pub fn field_errors(&self) -> Vec<ApiFieldError> {
        match self {
            ApiError::Api { body, .. } => body.fields.clone().unwrap_or_default(),
            _ => Vec::new(),
        }
    }
}
```

**จุดที่ต้องอธิบายเพิ่ม**: `UserResponse` ประกาศ `role: String` แทนการทำ `enum Role { Member, Admin }` —
นี่เป็นการตัดสินใจที่ตั้งใจ (ไม่ใช่ความขี้เกียจ) เพราะถ้า backend เพิ่ม role ใหม่ในอนาคต (เช่น `staff` ตามที่
Part 92 หัวข้อ 92.3 กล่าวถึงเป็นความเป็นไปได้) `enum` ฝั่ง frontend ที่ปิดตายไว้แค่สองค่าจะ deserialize
**fail ทันที** สำหรับผู้ใช้ที่มี role ใหม่นั้น (`serde` ปฏิเสธ variant ที่ไม่รู้จักโดย default) ทำให้แอป
frontend พังทั้งที่ backend ไม่ได้ทำอะไรผิด — การเก็บเป็น `String` ธรรมดาแล้วเทียบด้วย `==` ตอนต้องใช้จริง
(ดูหัวข้อ 93.4) ทำให้ frontend "ทนทาน" ต่อการเปลี่ยนแปลงฝั่ง backend ได้มากกว่า โดยแลกกับการไม่มี compiler
ช่วยเช็ค typo ของชื่อ role — ข้อแลกเปลี่ยนนี้สมเหตุสมผลเพราะ role มาจาก backend เสมอ ไม่ใช่ค่าที่ frontend
สร้างขึ้นเอง

ต่อไปนี้คือ `client.rs` — ฟังก์ชันเรียก endpoint ทั้ง 10 ตัว (จริง ๆ ใช้ 9 ตัวในบทนี้ endpoint `/health`
เก็บไว้ให้ Part 94 ใช้) ทุกฟังก์ชันมีรูปแบบเดียวกัน: สร้าง `Request`, แนบ header ถ้าต้อง auth, `.send()`,
ส่งต่อให้ `read_response` แปลงเป็น `Result<T, ApiError>`:

```rust
// src/api/client.rs
use gloo_net::http::Request;

use super::types::*;

/// Base URL ของ backend Part 92 — ตรงกับที่บทนั้นกำหนดไว้ (`BIND_ADDR` default `127.0.0.1:8092`)
pub const API_BASE: &str = "http://127.0.0.1:8092";

/// อ่าน response body เป็น T ถ้า status สำเร็จ, หรือแปลงเป็น ApiError::Api ตาม error shape
/// ของ Part 92 ถ้า status ไม่สำเร็จ — ฟังก์ชันกลางที่ทุก endpoint call เรียกใช้ร่วมกัน (DRY เดียวกับ
/// หลักการที่ Part 65 อธิบายไว้เรื่อง middleware — ไม่เขียน "เช็ค status แล้ว parse" ซ้ำ 9 รอบ)
async fn read_response<T: serde::de::DeserializeOwned>(
    response: gloo_net::http::Response,
) -> Result<T, ApiError> {
    let status = response.status();
    if (200..300).contains(&status) {
        response
            .json::<T>()
            .await
            .map_err(|e| ApiError::Decode(e.to_string()))
    } else {
        let envelope = response
            .json::<ApiErrorEnvelope>()
            .await
            .map_err(|e| ApiError::Decode(format!("อ่าน error body ไม่ได้: {e}")))?;
        Err(ApiError::Api { status, body: envelope.error })
    }
}

fn map_send_err(e: gloo_net::Error) -> ApiError {
    ApiError::Network(e.to_string())
}

/// POST /api/v1/auth/register — ไม่ต้อง auth
pub async fn register(req: &RegisterRequest) -> Result<UserResponse, ApiError> {
    let response = Request::post(&format!("{API_BASE}/api/v1/auth/register"))
        .json(req)
        .map_err(|e| ApiError::Decode(e.to_string()))?
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// POST /api/v1/auth/login — ไม่ต้อง auth, คืน access_token ที่ต้องแนบใน request ที่ต้อง auth ต่อไป
pub async fn login(req: &LoginRequest) -> Result<LoginResponse, ApiError> {
    let response = Request::post(&format!("{API_BASE}/api/v1/auth/login"))
        .json(req)
        .map_err(|e| ApiError::Decode(e.to_string()))?
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// GET /api/v1/me — ต้อง auth, ใช้ตอนโหลดหน้าเว็บใหม่เพื่อยืนยันว่า token ที่มีอยู่ยังใช้ได้จริง
pub async fn me(token: &str) -> Result<UserResponse, ApiError> {
    let response = Request::get(&format!("{API_BASE}/api/v1/me"))
        .header("Authorization", &format!("Bearer {token}"))
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// GET /api/v1/books — ไม่ต้อง auth, ทุก query param optional ตาม ListBooksParams
pub async fn list_books(params: &ListBooksParams) -> Result<BookListResponse, ApiError> {
    let mut query: Vec<(&str, String)> = Vec::new();
    if let Some(page) = params.page {
        query.push(("page", page.to_string()));
    }
    if let Some(per_page) = params.per_page {
        query.push(("per_page", per_page.to_string()));
    }
    if let Some(category) = &params.category {
        if !category.is_empty() {
            query.push(("category", category.clone()));
        }
    }
    if let Some(q) = &params.q {
        if !q.is_empty() {
            query.push(("q", q.clone()));
        }
    }
    if let Some(available_only) = params.available_only {
        query.push(("available_only", available_only.to_string()));
    }

    let response = Request::get(&format!("{API_BASE}/api/v1/books"))
        .query(query)
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// GET /api/v1/books/{id} — ไม่ต้อง auth
pub async fn get_book(id: i64) -> Result<BookResponse, ApiError> {
    let response = Request::get(&format!("{API_BASE}/api/v1/books/{id}"))
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// POST /api/v1/books — ต้อง auth + admin (backend เช็คซ้ำอีกชั้นเสมอ ต่อให้ frontend ซ่อนปุ่มไปแล้ว —
/// ดูหัวข้อ 93.9 เรื่อง defense in depth)
pub async fn create_book(token: &str, req: &CreateBookRequest) -> Result<BookResponse, ApiError> {
    let response = Request::post(&format!("{API_BASE}/api/v1/books"))
        .header("Authorization", &format!("Bearer {token}"))
        .json(req)
        .map_err(|e| ApiError::Decode(e.to_string()))?
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// POST /api/v1/books/{id}/borrow — ต้อง auth
pub async fn borrow_book(token: &str, id: i64) -> Result<BorrowResponse, ApiError> {
    let response = Request::post(&format!("{API_BASE}/api/v1/books/{id}/borrow"))
        .header("Authorization", &format!("Bearer {token}"))
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// POST /api/v1/books/{id}/return — ต้อง auth
pub async fn return_book(token: &str, id: i64) -> Result<BorrowResponse, ApiError> {
    let response = Request::post(&format!("{API_BASE}/api/v1/books/{id}/return"))
        .header("Authorization", &format!("Bearer {token}"))
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}

/// GET /api/v1/me/borrowed — ต้อง auth
pub async fn my_borrowed_books(token: &str) -> Result<BorrowListResponse, ApiError> {
    let response = Request::get(&format!("{API_BASE}/api/v1/me/borrowed"))
        .header("Authorization", &format!("Bearer {token}"))
        .send()
        .await
        .map_err(map_send_err)?;
    read_response(response).await
}
```

```rust
// src/api/mod.rs
pub mod client;
pub mod types;

pub use client::*;
pub use types::*;
```

**สังเกตจุดสำคัญที่ `.query(query)` ในฟังก์ชัน `list_books`**: parameter ของ `.query()` ใน `gloo-net`
0.6.0 ต้องเป็น `IntoIterator<Item = (&str, V)>` — **คีย์ต้องเป็น `&str` ไม่ใช่ `String`** (ถ้าใส่
`Vec<(String, String)>` จะเจอ compile error `type mismatch resolving <Vec<(String, String)> as
IntoIterator>::Item == (&str, _)` ทันที) โค้ดข้างบนจึงเขียน `("page", ...)` เป็น string literal (ซึ่งเป็น
`&'static str` โดยธรรมชาติ) เข้าคู่กับ `String` ที่เป็นค่า (`.to_string()`/`.clone()`) — จุดนี้เป็นกับดักที่
เจอจริงระหว่างพัฒนาบทนี้ (รายละเอียดเต็มอยู่ในหัวข้อกับดักที่พบบ่อยข้อ 4)

### 93.4 Auth State Management: Signal + Context + `localStorage`

ต่อยอด `provide_context`/`use_context` จาก Part 89 หัวข้อ 89.5 (ที่ใช้แชร์ `CartState` ข้าม component
tree) และ `web-sys`/`wasm-bindgen` จาก Part 87 (ที่บทนี้เข้าถึงผ่าน `gloo-storage` แทนการเรียก
`web_sys::Storage` ตรง ๆ เพื่อได้ API แบบ (de)serialize JSON ให้ฟรี) — โจทย์ของหัวข้อนี้คือ **session ต้อง
อยู่ได้สองระดับพร้อมกัน**: (1) ระดับ signal สำหรับ reactivity ภายในหน้าที่เปิดอยู่ (component ไหนอ่าน
`auth.session` แล้ว signal เปลี่ยน component นั้นต้อง re-render ทันที) และ (2) ระดับ `localStorage` สำหรับ
รอดจาก page reload (ปิด browser tab แล้วเปิดใหม่ หรือกด F5 ต้องยัง login อยู่ ไม่ต้อง login ใหม่ทุกครั้ง)

```rust
// src/auth.rs
use gloo_storage::{LocalStorage, Storage};
use leptos::prelude::*;

use crate::api::UserResponse;

const STORAGE_KEY: &str = "library_session_v1";

/// Session ที่เก็บทั้งใน signal (สำหรับ reactivity ภายในหน้าเว็บที่เปิดอยู่) และใน localStorage
/// (สำหรับรอดจาก page reload) — สอง field นี้คือทั้งหมดที่หน้าอื่นต้องรู้เกี่ยวกับผู้ใช้ปัจจุบัน
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Session {
    pub access_token: String,
    pub user: UserResponse,
}

/// AuthContext ที่ provide เข้า context ตั้งแต่ root component — ทุก component ลูก
/// เรียก `use_context::<AuthContext>()` เพื่ออ่าน/แก้สถานะ login ได้โดยไม่ต้องส่ง prop ไล่ลงมาทีละชั้น
/// (ตาม pattern เดียวกับ CartState ของ Part 89 หัวข้อ 89.5 — เพียงแต่ session ของบทนี้ผูกกับ
/// localStorage เพิ่มเข้ามา ซึ่ง CartState ของ Part 89 ไม่มี เพราะไม่ต้องรอด reload)
#[derive(Debug, Clone, Copy)]
pub struct AuthContext {
    pub session: RwSignal<Option<Session>>,
}

impl AuthContext {
    /// โหลด session จาก localStorage ตอนสร้าง context ครั้งแรก (ตอน app mount / page reload) —
    /// ถ้าไม่มีอะไรเก็บไว้ (ผู้ใช้ยังไม่เคย login) หรือ parse ไม่ได้ (เช่น เวอร์ชันข้อมูลเก่าไม่ตรง
    /// STORAGE_KEY) ให้เริ่มจาก None เสมอ ไม่ panic
    pub fn new() -> Self {
        let initial = LocalStorage::get::<Session>(STORAGE_KEY).ok();
        AuthContext {
            session: RwSignal::new(initial),
        }
    }

    pub fn is_authenticated(&self) -> bool {
        self.session.get().is_some()
    }

    /// เช็ค role ฝั่ง client — ใช้เพื่อ "UX" เท่านั้น (ซ่อน/แสดงปุ่ม) ไม่ใช่ชั้นความปลอดภัยจริง
    /// (ดูหัวข้อ 93.9 — backend ของ Part 92 เช็ค role ซ้ำอีกชั้นเสมอที่ POST /api/v1/books)
    pub fn is_admin(&self) -> bool {
        self.session
            .get()
            .map(|s| s.user.role == "admin")
            .unwrap_or(false)
    }

    pub fn token(&self) -> Option<String> {
        self.session.get().map(|s| s.access_token)
    }

    /// เขียน session ทั้ง signal (ให้ UI reactive ตอบสนองทันที) และ localStorage (ให้รอด reload)
    pub fn set_session(&self, session: Session) {
        // ถ้าเขียน localStorage ไม่ได้ (เช่น private mode บล็อก storage) แอปยังใช้งานต่อได้
        // ในแท็บนี้ผ่าน signal เฉย ๆ เพียงแต่ reload แล้วจะต้อง login ใหม่ — ไม่ panic ทั้งแอป
        // เพียงเพราะ storage เขียนไม่ได้
        let _ = LocalStorage::set(STORAGE_KEY, &session);
        self.session.set(Some(session));
    }

    pub fn logout(&self) {
        LocalStorage::delete(STORAGE_KEY);
        self.session.set(None);
    }
}

pub fn provide_auth_context() {
    provide_context(AuthContext::new());
}

pub fn use_auth() -> AuthContext {
    use_context::<AuthContext>().expect("AuthContext ต้องถูก provide จาก root component ก่อนใช้ use_auth()")
}
```

(ไฟล์นี้ต้อง `use serde::{Deserialize, Serialize};` เพิ่มที่หัวไฟล์ด้วย — ตัดออกจากตัวอย่างข้างบนเพื่อความ
กระชับ แต่ต้องมีจริงตอนคัดลอกไปใช้)

**เหตุผลที่ `AuthContext` derive `Copy`**: signal handle ของ Leptos (`RwSignal<T>`) เป็น `Copy` อยู่แล้ว
(กลไกเดียวกับที่ Part 89 หัวข้อ "กับดักที่พบบ่อย" อธิบายไว้เรื่อง arena-based signal ที่ทำให้ handle เบา
มาก ไม่ต้อง `.clone()` ทุกครั้งก่อน `move` เข้า closure) — เมื่อ `AuthContext` มีแค่ field เดียวที่เป็น
`RwSignal`, การทำให้ทั้ง struct เป็น `Copy` ทำให้เขียน `let auth = use_auth();` ครั้งเดียวแล้ว `move` เข้า
closure ได้กี่ที่ก็ได้โดยไม่ต้อง clone เพิ่ม (Rust จะ copy ค่าเข้าไปให้อัตโนมัติทุกจุดที่ `move`) ตรงตาม
รูปแบบเดียวกับ `CartState` ของ Part 89 หัวข้อ 89.5 ที่ derive `Clone, Copy` ด้วยเหตุผลเดียวกันเป๊ะ

**การตรวจสอบ token กับ backend จริงตอน app เริ่มทำงาน**: การโหลด session จาก `localStorage` เพียงอย่าง
เดียวไม่พอ — access token มีอายุ 3600 วินาทีตาม Part 92 หัวข้อ 92.1 ถ้าผู้ใช้ปิดแท็บทิ้งไว้นานเกินชั่วโมง
แล้วเปิดกลับมา `localStorage` ยังเก็บ session เดิมไว้ (เพราะไม่มีอะไรมาลบทิ้งเอง) แต่ token ข้างในนั้น
**หมดอายุไปแล้ว** — ถ้าปล่อยให้ UI แสดงว่า "login อยู่" โดยไม่เช็คอะไรเลย ผู้ใช้จะเจอ `401` แบบงง ๆ ทุกครั้ง
ที่กดทำอะไรที่ต้อง auth วิธีแก้คือเรียก `GET /api/v1/me` (endpoint ที่ Part 92 หัวข้อ 92.1 อธิบายไว้ว่า
เพิ่มมาเพื่อจุดประสงค์นี้โดยเฉพาะ) ตอน root component (`App`) เริ่มทำงาน — ถ้า token ยังใช้ได้ก็ไม่มีอะไร
เกิดขึ้น ถ้าใช้ไม่ได้แล้ว (`401`) ให้ `logout()` ทันที (โค้ดเต็มอยู่ใน `main.rs` หัวข้อ 93.5)

### 93.5 Routing ด้วย `leptos_router` + Route Guard + แถบนำทาง

ต่อยอด `leptos_router` จาก Part 89 หัวข้อ 89.8 ("แอปหลายหน้าด้วย `leptos_router`") ตรง ๆ — บทนี้มี 6 route:
หน้ารายการหนังสือ (`/`), login, register, รายละเอียดหนังสือ (`/books/:id`), หนังสือที่ยืมอยู่
(`/my-borrowed`), และสร้างหนังสือ (`/admin/new-book`)

เริ่มจากแถบนำทาง — component ที่แสดงอยู่ทุกหน้า (นอก `<Routes>`) และเป็นจุดที่อ่าน `AuthContext` บ่อยที่สุด
เพื่อสลับ UI ตามสถานะ login/role:

```rust
// src/components/nav.rs
use leptos::prelude::*;
use leptos_router::components::A;

use crate::auth::use_auth;

#[component]
pub fn Nav() -> impl IntoView {
    let auth = use_auth();

    view! {
        <nav id="nav">
            <A href="/">"หนังสือ"</A>
            " | "
            <A href="/my-borrowed">"ที่ยืมอยู่"</A>
            " | "
            // <Show> ต่อยอด Part 89 — render children เฉพาะตอน `when` เป็น true, ไม่มี "flash of
            // wrong content" ตอน component เพิ่ง mount เพราะ auth.is_admin() อ่านจาก signal
            // ที่โหลดจาก localStorage มาแล้วตั้งแต่ AuthContext::new() (หัวข้อ 93.4)
            <Show when=move || auth.is_admin()>
                <A href="/admin/new-book">"เพิ่มหนังสือ (admin)"</A>
                " | "
            </Show>
            <Show
                when=move || auth.is_authenticated()
                fallback=|| view! {
                    <A href="/login">"เข้าสู่ระบบ"</A>
                    " | "
                    <A href="/register">"สมัครสมาชิก"</A>
                }
            >
                <span id="whoami">
                    {move || auth.session.get().map(|s| format!("สวัสดี {} ({})", s.user.username, s.user.role))}
                </span>
                " | "
                <button id="logout-btn" on:click=move |_| auth.logout()>"ออกจากระบบ"</button>
            </Show>
        </nav>
    }
}
```

```rust
// src/components/mod.rs
pub mod nav;

pub use nav::Nav;
```

ในบทนี้ **route guard เขียนแบบ effect ที่ redirect ทันทีเมื่อไม่ผ่านเงื่อนไข** — วางไว้ที่ต้นฟังก์ชันของ
หน้าที่ต้องป้องกัน (`MyBorrowedPage`, `CreateBookPage`) แทนการเขียน wrapper component แยก เพราะจำนวน route
ที่ต้อง guard ในบทนี้มีแค่สองหน้า ยังไม่ถึงจุดที่ต้อง abstraction เพิ่ม (ตามหลักการ "ไม่ต้อง generalize ก่อน
มี pattern ซ้ำจริง" ที่ Part 21-22 พูดถึงเรื่อง generics — เขียน concrete ก่อนจนเห็น pattern ซ้ำค่อย
สกัดเป็นของกลาง):

```rust
// ตัวอย่างรูปแบบ route guard ที่ใช้ในบทนี้ (โค้ดเต็มอยู่ในหัวข้อ 93.8/93.9)
Effect::new(move |_| {
    if !auth.is_authenticated() {
        navigate("/login", Default::default());
    }
});
```

`Effect::new` (ต่อยอด Part 89 หัวข้อ 89.4) รันทันทีตอน component mount ครั้งแรก แล้ว track
`auth.is_authenticated()` เป็น dependency — ถ้าผู้ใช้กด logout ระหว่างที่ยังอยู่หน้านี้อยู่ (เช่นเปิดสอง
component พร้อมกันในหน้าเดียวกันซึ่งไม่เกิดในบทนี้ แต่หลักการเดียวกันใช้ได้ทั่วไป) effect นี้จะรันซ้ำและ
redirect ออกทันทีเช่นกัน ไม่ใช่แค่ตอน mount ครั้งแรกเท่านั้น

ประกอบทุกอย่างเข้าด้วยกันที่ `main.rs` — จุดที่ `provide_auth_context()` ถูกเรียก, ตรวจสอบ token กับ backend
จริงตามที่อธิบายไว้ในหัวข้อ 93.4, และประกาศ route ทั้ง 6 ตัว:

```rust
// src/main.rs
mod api;
mod auth;
mod components;
mod pages;

use leptos::prelude::*;
use leptos::task::spawn_local;
use leptos_router::components::{Route, Router, Routes};
use leptos_router::path;

use components::Nav;
use pages::{BookDetailPage, BooksPage, CreateBookPage, LoginPage, MyBorrowedPage, RegisterPage};

#[component]
fn App() -> impl IntoView {
    auth::provide_auth_context();
    let auth = auth::use_auth();

    // ตรวจสอบ session ที่โหลดมาจาก localStorage (ถ้ามี) กับ backend จริงตอน app เริ่มทำงาน — access
    // token มีอายุ 3600 วินาทีตาม Part 92 หัวข้อ 92.1 ถ้าปิดแท็บทิ้งไว้นานเกินไปแล้วเปิดใหม่ token ที่
    // localStorage ยังเก็บไว้อาจหมดอายุไปแล้ว การเรียก GET /api/v1/me ที่นี่คือทางเดียวที่รู้ได้แน่นอนว่า
    // token ยังใช้ได้จริงไหม (ไม่ใช่แค่เช็คว่า "มี token เก็บอยู่" เฉย ๆ)
    Effect::new(move |_| {
        if let Some(token) = auth.token() {
            spawn_local(async move {
                if api::me(&token).await.is_err() {
                    auth.logout();
                }
            });
        }
    });

    view! {
        <Router>
            <Nav />
            <main>
                <Routes fallback=|| view! { <p id="not-found">"ไม่พบหน้านี้"</p> }>
                    <Route path=path!("/") view=BooksPage />
                    <Route path=path!("/login") view=LoginPage />
                    <Route path=path!("/register") view=RegisterPage />
                    <Route path=path!("/books/:id") view=BookDetailPage />
                    <Route path=path!("/my-borrowed") view=MyBorrowedPage />
                    <Route path=path!("/admin/new-book") view=CreateBookPage />
                </Routes>
            </main>
        </Router>
    }
}

fn main() {
    console_error_panic_hook::set_once();
    leptos::mount::mount_to_body(App);
}
```

`console_error_panic_hook::set_once()` (ต่อยอด Part 87) ทำให้ panic ของ Rust ฝั่ง WASM แสดงเป็น stack
trace ที่อ่านออกใน Console ของเบราว์เซอร์ แทนที่จะเห็นแค่ `unreachable executed` ที่ไม่มีความหมายอะไรเลย —
ควรเปิดไว้เสมอตอนพัฒนา (และปิดได้ตอน production build จริงถ้าต้องการลดขนาด binary เล็กน้อย)

### 93.6 หน้า Login และ Register: ฟอร์มจริง เรียก Backend จริง

ต่อยอด event handling ของ Part 89 (`on:input`, `on:submit`, `event_target_value(&ev)`) และ `Action::new`
สำหรับผูก async operation เข้ากับ event — แต่มีจุดสำคัญที่ **ต้องใช้ `Action::new_local` ไม่ใช่
`Action::new`** เพราะ future ที่มาจาก `gloo-net` ไม่ satisfy `Send` bound (อธิบายละเอียดในหัวข้อกับดักที่
พบบ่อยข้อ 1 — Part 89 หัวข้อ 89.6 เจอปัญหาแบบเดียวกันกับ `gloo_timers` แล้วแก้ด้วย `LocalResource::new`
บทนี้เจอปัญหาเดียวกันกับ `Action` และแก้ด้วย `Action::new_local`)

```rust
// src/pages/login.rs
use leptos::prelude::*;
use leptos_router::hooks::use_navigate;

use crate::api::{self, ApiError, LoginRequest};
use crate::auth::{use_auth, Session};

#[component]
pub fn LoginPage() -> impl IntoView {
    let auth = use_auth();
    let navigate = use_navigate();

    let username = RwSignal::new(String::new());
    let password = RwSignal::new(String::new());

    // Action::new_local ผูก future เข้ากับ signal ภายในให้อัตโนมัติ (.pending()/.value()) ตาม
    // Part 89 หัวข้อ 89.6 — ไม่ต้องเขียน signal สถานะ loading เองด้วยมือแบบ spawn_local ตรง ๆ
    let login_action: Action<LoginRequest, Result<Session, ApiError>> =
        Action::new_local(move |req: &LoginRequest| {
            let req = req.clone();
            async move {
                let res = api::login(&req).await?;
                Ok(Session { access_token: res.access_token, user: res.user })
            }
        });

    Effect::new(move |_| {
        if let Some(Ok(session)) = login_action.value().get() {
            auth.set_session(session);
            navigate("/", Default::default());
        }
    });

    let on_submit = move |ev: leptos::ev::SubmitEvent| {
        ev.prevent_default();
        login_action.dispatch(LoginRequest {
            username: username.get(),
            password: password.get(),
        });
    };

    view! {
        <div id="login-page">
            <h1>"เข้าสู่ระบบ"</h1>
            <form on:submit=on_submit>
                <div>
                    <label for="login-username">"Username"</label>
                    <input
                        id="login-username"
                        type="text"
                        prop:value=username
                        on:input=move |ev| username.set(event_target_value(&ev))
                    />
                </div>
                <div>
                    <label for="login-password">"Password"</label>
                    <input
                        id="login-password"
                        type="password"
                        prop:value=password
                        on:input=move |ev| password.set(event_target_value(&ev))
                    />
                </div>
                <button id="login-submit" type="submit" disabled=move || login_action.pending().get()>
                    {move || if login_action.pending().get() { "กำลังเข้าสู่ระบบ..." } else { "เข้าสู่ระบบ" }}
                </button>
            </form>
            <div id="login-error" class="error">
                {move || match login_action.value().get() {
                    Some(Err(e)) => e.user_message(),
                    _ => String::new(),
                }}
            </div>
        </div>
    }
}
```

`RegisterRequest`/`LoginRequest` ที่ dispatch เข้า `Action` ต้อง `#[derive(Clone)]` เสมอ เพราะ
`Action::new_local` เก็บ closure ที่รับ `&I` (reference) แต่ต้องเอาไปสร้าง `async move` block ที่ own ค่า
นั้น — `let req = req.clone();` ก่อนเข้า `async move` คือ pattern มาตรฐานที่ Part 89 หัวข้อ 89.6 ก็ใช้เดียวกัน

หน้า register ซับซ้อนกว่าเล็กน้อยเพราะต้องแสดง **field-level error** (จาก `400 VALIDATION_ERROR`) แยกตาม
field ไม่ใช่โชว์ error message รวมก้อนเดียว:

```rust
// src/pages/register.rs
use leptos::prelude::*;
use leptos_router::hooks::use_navigate;

use crate::api::{self, ApiError, RegisterRequest, UserResponse};

#[component]
pub fn RegisterPage() -> impl IntoView {
    let navigate = use_navigate();

    let username = RwSignal::new(String::new());
    let email = RwSignal::new(String::new());
    let password = RwSignal::new(String::new());

    let register_action: Action<RegisterRequest, Result<UserResponse, ApiError>> =
        Action::new_local(move |req: &RegisterRequest| {
            let req = req.clone();
            async move { api::register(&req).await }
        });

    Effect::new(move |_| {
        if let Some(Ok(_)) = register_action.value().get() {
            // สมัครสำเร็จ -> ไปหน้า login ให้กรอก credential เข้าสู่ระบบเอง (endpoint register ไม่คืน token
            // ตามที่ Part 92 หัวข้อ 92.1 ออกแบบไว้ — register คืน UserResponse ธรรมดา ไม่ใช่ LoginResponse)
            navigate("/login", Default::default());
        }
    });

    let on_submit = move |ev: leptos::ev::SubmitEvent| {
        ev.prevent_default();
        register_action.dispatch(RegisterRequest {
            username: username.get(),
            email: email.get(),
            password: password.get(),
        });
    };

    // field-level error สำหรับ field หนึ่ง ๆ — คืนค่าว่างถ้าไม่มี error ของ field นั้น (ตาม
    // ApiError::field_errors() ที่คืน Vec<ApiFieldError> เปล่าถ้าไม่ใช่ VALIDATION_ERROR)
    let field_error = move |field: &'static str| -> String {
        match register_action.value().get() {
            Some(Err(e)) => e
                .field_errors()
                .into_iter()
                .find(|f| f.field == field)
                .map(|f| f.message)
                .unwrap_or_default(),
            _ => String::new(),
        }
    };

    let non_field_error = move || -> String {
        match register_action.value().get() {
            // แสดง error รวมเฉพาะตอนไม่ใช่ VALIDATION_ERROR (เช่น 409 username ซ้ำ) — ถ้าเป็น
            // VALIDATION_ERROR field_error() ข้างบนแสดงแยกให้แล้ว ไม่ต้องแสดงซ้ำสองที่
            Some(Err(e)) if e.field_errors().is_empty() => e.user_message(),
            _ => String::new(),
        }
    };

    view! {
        <div id="register-page">
            <h1>"สมัครสมาชิก"</h1>
            <form on:submit=on_submit>
                <div>
                    <label for="reg-username">"Username"</label>
                    <input
                        id="reg-username"
                        type="text"
                        prop:value=username
                        on:input=move |ev| username.set(event_target_value(&ev))
                    />
                    <span class="field-error" id="reg-username-error">{move || field_error("username")}</span>
                </div>
                <div>
                    <label for="reg-email">"Email"</label>
                    <input
                        id="reg-email"
                        type="text"
                        prop:value=email
                        on:input=move |ev| email.set(event_target_value(&ev))
                    />
                    <span class="field-error" id="reg-email-error">{move || field_error("email")}</span>
                </div>
                <div>
                    <label for="reg-password">"Password"</label>
                    <input
                        id="reg-password"
                        type="password"
                        prop:value=password
                        on:input=move |ev| password.set(event_target_value(&ev))
                    />
                    <span class="field-error" id="reg-password-error">{move || field_error("password")}</span>
                </div>
                <button id="reg-submit" type="submit" disabled=move || register_action.pending().get()>
                    {move || if register_action.pending().get() { "กำลังสมัคร..." } else { "สมัครสมาชิก" }}
                </button>
            </form>
            <div id="register-error" class="error">{non_field_error}</div>
        </div>
    }
}
```

```rust
// src/pages/mod.rs
pub mod book_detail;
pub mod books;
pub mod create_book;
pub mod login;
pub mod my_borrowed;
pub mod register;

pub use book_detail::BookDetailPage;
pub use books::BooksPage;
pub use create_book::CreateBookPage;
pub use login::LoginPage;
pub use my_borrowed::MyBorrowedPage;
pub use register::RegisterPage;
```

**พิสูจน์ด้วย headless Chromium จริง** (backend รันอยู่ที่ `8092`, frontend รันด้วย `trunk serve` ที่
`5173` — ดูหัวข้อ 93.11 สำหรับ setup เต็ม) — สมัครสมาชิกใหม่สำเร็จแล้ว redirect ไป `/login` จริง:

```
=== after register -> redirected to /login ===
http://127.0.0.1:5173/login
```

network request/response จริงที่ Playwright ดักจับได้ (จาก `page.on('response', ...)`):

```
--> POST http://127.0.0.1:8092/api/v1/auth/register
<-- 201 http://127.0.0.1:8092/api/v1/auth/register
    body={"id":3,"username":"e2euser249014","email":"e2euser249014@example.com","role":"member","created_at":"2026-09-27T03:54:40.438340Z"}
```

สมัครด้วย username เดิมซ้ำ (backend ตอบ `409` ตาม unique constraint ของ Part 92 หัวข้อ 92.3) — UI แสดง
error message ที่ `#register-error` ตรงตัวจาก backend:

```
=== duplicate register error shown in UI ===
409 CONFLICT: ขัดแย้งกับสถานะปัจจุบัน: ข้อมูลนี้ซ้ำกับที่มีอยู่แล้วในระบบ
```

ส่ง input ผิดสามจุดพร้อมกัน (username สั้นเกิน, email ผิดรูปแบบ, password สั้นเกิน) — UI แสดง field-level
error **แยกช่องตามที่ backend ระบุมาใน `fields[]`** ไม่ใช่ error รวมก้อนเดียว:

```
=== field-level validation errors shown in UI ===
{
  "username": "ต้องมี 3-32 ตัวอักษร และเป็น a-z, A-Z, 0-9 หรือ _ เท่านั้น",
  "email": "รูปแบบอีเมลไม่ถูกต้อง",
  "password": "ต้องมีอย่างน้อย 8 ตัวอักษร"
}
```

login ด้วย credential ที่ถูกต้อง — nav bar เปลี่ยนไปแสดงชื่อผู้ใช้จริงทันที (signal reactive), และ
`localStorage` มี session ที่ถูก set จริง:

```
=== after login, nav shows ===
สวัสดี e2euser249014 (member)

=== localStorage after login ===
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIzIiwidXNlcm5hbWUiOiJlMmV1c2VyMjQ5MDE0Iiwicm9sZSI6Im1lbWJlciIsImlhdCI6MTc5MDQ4MTI4NCwiZXhwIjoxNzkwNDg0ODg0fQ.P9kYlwwo-hBFUoP4ZMMhOrvwOXPvE7goiaMPt91zkj4","user":{"id":3,"username":"e2euser249014","email":"e2euser249014@example.com","role":"member","created_at":"2026-09-27T03:54:40.438340Z"}}
```

login ด้วย password ผิด — `401` จาก backend แสดงเป็นข้อความที่ `#login-error`:

```
=== wrong password login error shown in UI ===
401 UNAUTHORIZED: ไม่ได้รับอนุญาต: username หรือ password ไม่ถูกต้อง
```

### 93.7 หน้ารายการหนังสือ: Pagination, Filter, และปุ่มยืม

ต่อยอด pagination convention ของ Part 78 (`page`/`per_page`, response ที่มี `data`+`meta`) — หน้านี้เก็บ
`page`/`q`/`available_only` เป็น signal แยก แล้วให้ `list_books` refetch ทุกครั้งที่ตัวใดตัวหนึ่งเปลี่ยน
ผ่าน `LocalResource::new` (ไม่ใช่ `Resource::new` — เหตุผลเดียวกับ `Action::new_local` ในหัวข้อก่อน คือ
future ของ `gloo-net` ไม่ satisfy `Send`):

```rust
// src/pages/books.rs
use leptos::prelude::*;
use leptos_router::components::A;

use crate::api::{self, ApiError, BorrowResponse, ListBooksParams};
use crate::auth::use_auth;

#[component]
pub fn BooksPage() -> impl IntoView {
    let auth = use_auth();

    let page = RwSignal::new(1_i64);
    let q = RwSignal::new(String::new());
    let available_only = RwSignal::new(false);

    // borrow_action ผูกกับ resource ผ่าน refetch ด้านล่าง — ทุกครั้งที่ยืมสำเร็จ ต้อง refetch รายการ
    // หนังสือใหม่ เพราะ available_copies ของเล่มนั้นเปลี่ยนไปแล้วที่ backend
    let borrow_action: Action<i64, Result<BorrowResponse, ApiError>> = Action::new_local(move |id: &i64| {
        let id = *id;
        let token = auth.token();
        async move {
            let token = token.ok_or_else(|| ApiError::Network("ยังไม่ได้ login".to_string()))?;
            api::borrow_book(&token, id).await
        }
    });

    // LocalResource::new (ไม่ใช่ Resource::new) เพราะ future ของ gloo-net ห่อ JsFuture ที่ผูกกับ
    // เธรดเดียวของเบราว์เซอร์ ไม่ satisfy `Send` bound ที่ Resource::new ต้องการ (ปัญหาเดียวกับที่ Part
    // 89 หัวข้อ 89.6 เจอกับ gloo_timers — ดูกับดักที่พบบ่อยข้อ 2 ท้ายบทนี้) — LocalResource::new รับแค่
    // fetcher เดียว (ไม่มี source แยก) แต่ยัง reactive ได้เพราะทุก signal ที่ `.get()` แบบ synchronous
    // *ก่อน* เข้า `async move` (คือ page/q/available_only/version ด้านล่าง) ถูก track เป็น dependency
    // ให้ re-run อัตโนมัติทุกครั้งที่ค่าเปลี่ยน เหมือนกับ source closure ของ Resource::new ทุกประการ
    let books = LocalResource::new(move || {
        let page_val = page.get();
        let q_val = q.get();
        let available_only_val = available_only.get();
        let _version = borrow_action.version().get();
        async move {
            api::list_books(&ListBooksParams {
                page: Some(page_val),
                per_page: Some(5),
                category: None,
                q: if q_val.is_empty() { None } else { Some(q_val) },
                available_only: if available_only_val { Some(true) } else { None },
            })
            .await
        }
    });

    view! {
        <div id="books-page">
            <h1>"รายการหนังสือ"</h1>
            <div id="books-filter">
                <input
                    id="books-search"
                    type="text"
                    placeholder="ค้นหาชื่อ/ผู้เขียน"
                    on:input=move |ev| { q.set(event_target_value(&ev)); page.set(1); }
                />
                <label>
                    <input
                        id="books-available-only"
                        type="checkbox"
                        on:change=move |_| { available_only.update(|v| *v = !*v); page.set(1); }
                    />
                    " มีสำเนาว่างเท่านั้น"
                </label>
            </div>
            <Suspense fallback=move || view! { <p id="books-loading">"กำลังโหลดรายการหนังสือ..."</p> }>
                {move || {
                    books.get().map(|result| match result {
                        Err(e) => view! { <p class="error" id="books-error">{e.user_message()}</p> }.into_any(),
                        Ok(list) => view! {
                            <div>
                                <ul id="books-list">
                                    {list.data.iter().cloned().map(|book| {
                                        let book_id = book.id;
                                        let can_borrow = auth.is_authenticated() && book.available_copies > 0;
                                        view! {
                                            <li class="book-item">
                                                <A href=format!("/books/{book_id}")>
                                                    {format!("{} — {} ({}/{} ว่าง)", book.title, book.author, book.available_copies, book.total_copies)}
                                                </A>
                                                <Show when=move || can_borrow>
                                                    <button
                                                        class="borrow-btn"
                                                        on:click=move |_| { borrow_action.dispatch(book_id); }
                                                    >
                                                        "ยืม"
                                                    </button>
                                                </Show>
                                            </li>
                                        }
                                    }).collect_view()}
                                </ul>
                                <div id="books-pagination">
                                    <button
                                        id="prev-page"
                                        disabled=move || { page.get() <= 1 }
                                        on:click=move |_| page.update(|p| *p = (*p - 1).max(1))
                                    >
                                        "ก่อนหน้า"
                                    </button>
                                    <span id="page-info">
                                        {format!(" หน้า {} / {} (ทั้งหมด {} เล่ม) ", list.meta.page, list.meta.total_pages.max(1), list.meta.total_items)}
                                    </span>
                                    <button
                                        id="next-page"
                                        disabled=move || { page.get() >= list.meta.total_pages }
                                        on:click=move |_| page.update(|p| *p += 1)
                                    >
                                        "ถัดไป"
                                    </button>
                                </div>
                            </div>
                        }.into_any(),
                    })
                }}
            </Suspense>
            <div id="borrow-feedback">
                {move || match borrow_action.value().get() {
                    Some(Err(e)) => e.user_message(),
                    Some(Ok(b)) => format!("ยืมสำเร็จ! กำหนดคืนวันที่ {}", b.due_at),
                    None => String::new(),
                }}
            </div>
        </div>
    }
}
```

สังเกตว่า `disabled=move || { page.get() <= 1 }` และ `disabled=move || { page.get() >= list.meta.total_pages
}` ทั้งคู่**ต้องห่อด้วย `{}`** แม้จะเป็น closure เดียวก็ตาม — นี่ไม่ใช่สไตล์ที่เลือกเอง แต่เป็นสิ่งที่บังคับ
จริงจากการทดสอบ (รายละเอียดเต็มอยู่ในหัวข้อกับดักที่พบบ่อยข้อ 3 พร้อม DOM ที่พังจริงที่ได้จากการทดสอบตอนไม่
ห่อ `{}`)

**พิสูจน์ด้วยเซิร์ฟเวอร์คู่กันจริง** — รายการหนังสือที่ backend มีอยู่ (สร้างไว้ล่วงหน้าด้วย admin ตาม Part
92 หัวข้อ 92.8) แสดงผลถูกต้องตั้งแต่โหลดหน้าแรก:

```
=== books list (page 1) ===
The Rust Programming Language — Steve Klabnik (2/2 ว่าง)ยืม
Zero To Production In Rust — Luca Palmieri (1/1 ว่าง)ยืม
```

network request จริงที่เกิดขึ้นตอนโหลดหน้า (ไม่มี `category`/`q`/`available_only` เพราะทุกตัว default เป็น
`None` ตั้งแต่เริ่ม):

```
--> GET http://127.0.0.1:8092/api/v1/books?page=1&per_page=5
<-- 200 http://127.0.0.1:8092/api/v1/books?page=1&per_page=5
    body={"data":[{"id":1,"title":"The Rust Programming Language", ...}, {"id":2,"title":"Zero To Production In Rust", ...}],
          "meta":{"page":1,"per_page":5,"total_items":2,"total_pages":1}}
```

พิมพ์คำค้นหา "rust" ลงในกล่องค้นหา (`q=rust`) — request ใหม่ยิงออกไปพร้อม query string ที่ถูกต้อง (ทั้งสอง
เล่มมีคำว่า "Rust" อยู่ในชื่อจึงยังเห็นทั้งคู่ ซึ่งเป็นพฤติกรรมที่ถูกต้องของ substring match แบบ
case-insensitive ตาม Part 92 หัวข้อ 92.8):

```
--> GET http://127.0.0.1:8092/api/v1/books?page=1&per_page=5&q=rust
<-- 200 http://127.0.0.1:8092/api/v1/books?page=1&per_page=5&q=rust
    body={"data":[{"id":1, ...}, {"id":2, ...}], "meta":{...}}
```

กดปุ่ม "ยืม" ที่ "Zero To Production In Rust" (มี 1 สำเนา) — สำเร็จ, `available_copies` ลดจาก 1 เป็น 0
ทันที และรายการ refetch ใหม่แสดงตัวเลขที่อัปเดตแล้ว (นี่คือผลของการนับ `borrow_action.version()` เป็น
dependency ของ `LocalResource` ตามที่อธิบายไว้ในโค้ดข้างบน):

```
=== borrow feedback (list page) ===
ยืมสำเร็จ! กำหนดคืนวันที่ 2026-10-11T03:54:47.659305Z

=== books list after borrowing (available count should drop) ===
The Rust Programming Language — Steve Klabnik (2/2 ว่าง)ยืม
Zero To Production In Rust — Luca Palmieri (0/1 ว่าง)
```

สังเกตว่าปุ่ม "ยืม" ของเล่มที่สองหายไปจาก DOM โดยอัตโนมัติ (ไม่มีคำว่า "ยืม" ต่อท้ายบรรทัดที่สองอีกแล้ว) —
เพราะ `can_borrow = auth.is_authenticated() && book.available_copies > 0` กลายเป็น `false` ทันทีที่
`available_copies` เหลือ 0 และ `<Show when=move || can_borrow>` ถอดปุ่มออกจาก DOM ให้เองโดยไม่ต้องเขียน
logic เพิ่ม

### 93.8 หน้ารายละเอียดหนังสือ, ยืม/คืน, และหน้า "หนังสือที่ยืมอยู่"

หน้ารายละเอียดหนังสือดึง `id` จาก URL param ผ่าน `use_params_map()` (ต่อยอด Part 89 หัวข้อ 89.8) แล้วเรียก
`GET /api/v1/books/{id}` — ปุ่มยืมแสดงเฉพาะตอน login แล้วและมีสำเนาว่าง เหมือนกับหน้ารายการหนังสือ:

```rust
// src/pages/book_detail.rs
use leptos::prelude::*;
use leptos_router::hooks::use_params_map;

use crate::api::{self, ApiError, BorrowResponse};
use crate::auth::use_auth;

#[component]
pub fn BookDetailPage() -> impl IntoView {
    let auth = use_auth();
    let params = use_params_map();
    let book_id = move || {
        params
            .read()
            .get("id")
            .and_then(|s| s.parse::<i64>().ok())
            .unwrap_or(0)
    };

    let borrow_action: Action<i64, Result<BorrowResponse, ApiError>> = Action::new_local(move |id: &i64| {
        let id = *id;
        let token = auth.token();
        async move {
            let token = token.ok_or_else(|| ApiError::Network("ยังไม่ได้ login".to_string()))?;
            api::borrow_book(&token, id).await
        }
    });

    // LocalResource::new เหตุผลเดียวกับหน้า BooksPage — future ของ gloo-net ไม่ satisfy Send
    let book = LocalResource::new(move || {
        let id = book_id();
        let _version = borrow_action.version().get();
        async move { api::get_book(id).await }
    });

    view! {
        <div id="book-detail-page">
            <Suspense fallback=move || view! { <p id="detail-loading">"กำลังโหลด..."</p> }>
                {move || {
                    book.get().map(|result| match result {
                        Err(e) => view! { <p class="error" id="detail-error">{e.user_message()}</p> }.into_any(),
                        Ok(b) => {
                            let id = b.id;
                            let can_borrow = auth.is_authenticated() && b.available_copies > 0;
                            let must_login = !auth.is_authenticated();
                            view! {
                                <div>
                                    <h1 id="detail-title">{b.title.clone()}</h1>
                                    <p id="detail-author">"ผู้เขียน: " {b.author.clone()}</p>
                                    <p id="detail-isbn">"ISBN: " {b.isbn.clone()}</p>
                                    <p id="detail-category">"หมวด: " {b.category.clone()}</p>
                                    <p id="detail-copies">
                                        {format!("สำเนาว่าง {} จาก {} เล่ม", b.available_copies, b.total_copies)}
                                    </p>
                                    <Show when=move || can_borrow>
                                        <button id="detail-borrow-btn" on:click=move |_| { borrow_action.dispatch(id); }>
                                            "ยืมหนังสือเล่มนี้"
                                        </button>
                                    </Show>
                                    <Show when=move || must_login>
                                        <p id="detail-need-login">"ต้องเข้าสู่ระบบก่อนถึงจะยืมได้"</p>
                                    </Show>
                                </div>
                            }.into_any()
                        }
                    })
                }}
            </Suspense>
            <div id="detail-borrow-feedback">
                {move || match borrow_action.value().get() {
                    Some(Err(e)) => e.user_message(),
                    Some(Ok(b)) => format!("ยืมสำเร็จ! กำหนดคืนวันที่ {}", b.due_at),
                    None => String::new(),
                }}
            </div>
        </div>
    }
}
```

หน้า "หนังสือที่ยืมอยู่" คือหน้าแรกที่ต้องมี **route guard จริง** เพราะไม่มีความหมายเลยถ้าเปิดโดยไม่ login
— redirect ไปหน้า login ทันทีถ้า `!auth.is_authenticated()` แล้วดึงรายการจาก `GET /api/v1/me/borrowed`
พร้อมปุ่ม "คืน" ที่เรียก `POST /api/v1/books/{id}/return`:

```rust
// src/pages/my_borrowed.rs
use leptos::prelude::*;
use leptos_router::hooks::use_navigate;

use crate::api::{self, ApiError, BorrowResponse};
use crate::auth::use_auth;

#[component]
pub fn MyBorrowedPage() -> impl IntoView {
    let auth = use_auth();
    let navigate = use_navigate();

    // route guard: ถ้ายังไม่ login redirect ไปหน้า login ทันที
    Effect::new(move |_| {
        if !auth.is_authenticated() {
            navigate("/login", Default::default());
        }
    });

    let return_action: Action<i64, Result<BorrowResponse, ApiError>> = Action::new_local(move |id: &i64| {
        let id = *id;
        let token = auth.token();
        async move {
            let token = token.ok_or_else(|| ApiError::Network("ยังไม่ได้ login".to_string()))?;
            api::return_book(&token, id).await
        }
    });

    // LocalResource::new เหตุผลเดียวกับหน้า BooksPage — future ของ gloo-net ไม่ satisfy Send
    let borrowed = LocalResource::new(move || {
        let _version = return_action.version().get();
        let token = auth.token();
        async move {
            match token {
                Some(t) => api::my_borrowed_books(&t).await,
                None => Err(ApiError::Network("ยังไม่ได้ login".to_string())),
            }
        }
    });

    view! {
        <div id="my-borrowed-page">
            <h1>"หนังสือที่ยืมอยู่"</h1>
            <Suspense fallback=move || view! { <p id="borrowed-loading">"กำลังโหลด..."</p> }>
                {move || {
                    borrowed.get().map(|result| match result {
                        Err(e) => view! { <p class="error" id="borrowed-error">{e.user_message()}</p> }.into_any(),
                        Ok(list) if list.data.is_empty() => {
                            view! { <p id="borrowed-empty">"ไม่มีหนังสือที่ยืมอยู่ตอนนี้"</p> }.into_any()
                        }
                        Ok(list) => view! {
                            <ul id="borrowed-list">
                                {list.data.iter().cloned().map(|borrow| {
                                    let book_id = borrow.book_id;
                                    view! {
                                        <li class="borrowed-item">
                                            {format!("book_id={} ยืมเมื่อ {} กำหนดคืน {}", borrow.book_id, borrow.borrowed_at, borrow.due_at)}
                                            <button class="return-btn" on:click=move |_| { return_action.dispatch(book_id); }>
                                                "คืนหนังสือ"
                                            </button>
                                        </li>
                                    }
                                }).collect_view()}
                            </ul>
                        }.into_any(),
                    })
                }}
            </Suspense>
            <div id="return-feedback">
                {move || match return_action.value().get() {
                    Some(Err(e)) => e.user_message(),
                    Some(Ok(_)) => "คืนหนังสือสำเร็จ".to_string(),
                    None => String::new(),
                }}
            </div>
        </div>
    }
}
```

**พิสูจน์ flow ยืม → ดูในหน้า "ที่ยืมอยู่" → คืน เต็มรูปแบบด้วยเซิร์ฟเวอร์จริง**:

```
=== my-borrowed page content ===
หนังสือที่ยืมอยู่
book_id=2 ยืมเมื่อ 2026-09-27T03:54:47.656867Z กำหนดคืน 2026-10-11T03:54:47.659305Zคืนหนังสือ
```

กดปุ่ม "คืนหนังสือ" — network request จริงที่เกิดขึ้น:

```
--> POST http://127.0.0.1:8092/api/v1/books/2/return
<-- 200 http://127.0.0.1:8092/api/v1/books/2/return
    body={"id":1,"book_id":2,"user_id":3,"borrowed_at":"2026-09-27T03:54:47.656867Z","due_at":"2026-10-11T03:54:47.659305Z","returned_at":"2026-09-27T03:54:49.109738Z"}
--> GET http://127.0.0.1:8092/api/v1/me/borrowed
<-- 200 http://127.0.0.1:8092/api/v1/me/borrowed  body={"data":[]}
```

และ DOM หลังคืนสำเร็จเปลี่ยนไปแสดงข้อความ "ไม่มีหนังสือที่ยืมอยู่ตอนนี้" ทันที (เพราะ `return_action`
สำเร็จ → `.version()` เปลี่ยน → `LocalResource` refetch → `list.data.is_empty()` เป็น `true`):

```
=== my-borrowed page after return ===
หนังสือที่ยืมอยู่

ไม่มีหนังสือที่ยืมอยู่ตอนนี้

คืนหนังสือสำเร็จ
```

**ทดสอบ route guard จริง**: ล็อกเอาต์แล้วพยายามเข้า `/my-borrowed` ตรง ๆ (พิมพ์ URL เอง ไม่ใช่คลิกลิงก์
จากในแอป) — ระบบ redirect ไปหน้า login ทันทีโดยไม่แสดงเนื้อหาของหน้านั้นแม้แต่แว้บเดียว:

```
=== logged-out user visiting /my-borrowed -> current URL (route guard) ===
http://127.0.0.1:5173/login
```

### 93.9 Admin-Only UI: หน้าสร้างหนังสือ + การตรวจ Role ฝั่ง Client

ต่อยอด RBAC จาก Part 76 — หน้านี้ตรวจ `auth.is_admin()` (ซึ่งอ่าน `role` จาก `UserResponse` ที่ได้มาจาก
`LoginResponse`/`GET /api/v1/me` โดยตรง ไม่ได้ decode จาก JWT payload เอง) เพื่อ redirect คนที่ไม่ใช่ admin
ออกจากหน้านี้ทันที **แต่ต้องย้ำเสมอว่านี่คือ UX เท่านั้น** — การตรวจสิทธิ์จริงเกิดที่ backend
(`current_user.is_admin()` ใน `create_book` handler ของ Part 92 หัวข้อ 92.8) การซ่อน/redirect ฝั่ง
frontend เป็นแค่การทำให้ผู้ใช้ทั่วไปไม่เห็น UI ที่ใช้ไม่ได้ ไม่ใช่ชั้นความปลอดภัย เพราะ **ใครก็เปิด
DevTools ยิง `curl`/`fetch` ตรงไปที่ backend แนบ token ของตัวเองได้เสมอ ไม่ผ่าน UI นี้เลยก็ได้** — นี่คือ
หลักการ **defense in depth** เดียวกับที่ Part 76 และ Part 92 หัวข้อ 92.8 ย้ำไว้ตลอด: การเช็คฝั่ง client มี
ไว้เพื่อ "ประสบการณ์ใช้งานที่ดี" ส่วนการเช็คฝั่ง server มีไว้เพื่อ "ความปลอดภัยจริง" สองอย่างนี้ไม่ใช่สิ่ง
เดียวกันและต้องมีทั้งคู่เสมอ

```rust
// src/pages/create_book.rs
use leptos::prelude::*;
use leptos_router::hooks::use_navigate;

use crate::api::{self, ApiError, BookResponse, CreateBookRequest};
use crate::auth::use_auth;

#[component]
pub fn CreateBookPage() -> impl IntoView {
    let auth = use_auth();
    let navigate = use_navigate();

    // route guard สองชั้น: ต้อง login และต้องเป็น admin — ถ้าไม่ผ่านชั้นใดชั้นหนึ่ง redirect ออกทันที
    // (การซ่อนปุ่มในหน้า Nav เป็นแค่ UX เท่านั้น backend ยังเช็คซ้ำเสมอตาม Part 92 หัวข้อ 92.8)
    Effect::new(move |_| {
        if !auth.is_authenticated() {
            navigate("/login", Default::default());
        } else if !auth.is_admin() {
            navigate("/", Default::default());
        }
    });

    let title = RwSignal::new(String::new());
    let author = RwSignal::new(String::new());
    let isbn = RwSignal::new(String::new());
    let category = RwSignal::new(String::new());
    let total_copies = RwSignal::new(1_i32);

    let create_action: Action<CreateBookRequest, Result<BookResponse, ApiError>> =
        Action::new_local(move |req: &CreateBookRequest| {
            let req = req.clone();
            let token = auth.token();
            async move {
                let token = token.ok_or_else(|| ApiError::Network("ยังไม่ได้ login".to_string()))?;
                api::create_book(&token, &req).await
            }
        });

    let field_error = move |field: &'static str| -> String {
        match create_action.value().get() {
            Some(Err(e)) => e
                .field_errors()
                .into_iter()
                .find(|f| f.field == field)
                .map(|f| f.message)
                .unwrap_or_default(),
            _ => String::new(),
        }
    };

    let non_field_error = move || -> String {
        match create_action.value().get() {
            Some(Err(e)) if e.field_errors().is_empty() => e.user_message(),
            _ => String::new(),
        }
    };

    let on_submit = move |ev: leptos::ev::SubmitEvent| {
        ev.prevent_default();
        create_action.dispatch(CreateBookRequest {
            title: title.get(),
            author: author.get(),
            isbn: isbn.get(),
            category: category.get(),
            total_copies: total_copies.get(),
        });
    };

    view! {
        <div id="create-book-page">
            <h1>"เพิ่มหนังสือใหม่ (admin เท่านั้น)"</h1>
            <form on:submit=on_submit>
                <div>
                    <label for="cb-title">"ชื่อหนังสือ"</label>
                    <input id="cb-title" type="text" prop:value=title
                        on:input=move |ev| title.set(event_target_value(&ev)) />
                    <span class="field-error" id="cb-title-error">{move || field_error("title")}</span>
                </div>
                <div>
                    <label for="cb-author">"ผู้เขียน"</label>
                    <input id="cb-author" type="text" prop:value=author
                        on:input=move |ev| author.set(event_target_value(&ev)) />
                    <span class="field-error" id="cb-author-error">{move || field_error("author")}</span>
                </div>
                <div>
                    <label for="cb-isbn">"ISBN (10 หรือ 13 หลัก)"</label>
                    <input id="cb-isbn" type="text" prop:value=isbn
                        on:input=move |ev| isbn.set(event_target_value(&ev)) />
                    <span class="field-error" id="cb-isbn-error">{move || field_error("isbn")}</span>
                </div>
                <div>
                    <label for="cb-category">"หมวด"</label>
                    <input id="cb-category" type="text" prop:value=category
                        on:input=move |ev| category.set(event_target_value(&ev)) />
                    <span class="field-error" id="cb-category-error">{move || field_error("category")}</span>
                </div>
                <div>
                    <label for="cb-total-copies">"จำนวนสำเนา"</label>
                    <input id="cb-total-copies" type="number" prop:value=move || total_copies.get().to_string()
                        on:input=move |ev| total_copies.set(event_target_value(&ev).parse().unwrap_or(0)) />
                    <span class="field-error" id="cb-total-copies-error">{move || field_error("total_copies")}</span>
                </div>
                <button id="cb-submit" type="submit" disabled=move || create_action.pending().get()>
                    {move || if create_action.pending().get() { "กำลังบันทึก..." } else { "สร้างหนังสือ" }}
                </button>
            </form>
            <div id="create-book-error" class="error">{non_field_error}</div>
            <div id="create-book-success">
                {move || match create_action.value().get() {
                    Some(Ok(b)) => format!("สร้างหนังสือ '{}' สำเร็จ (id={})", b.title, b.id),
                    _ => String::new(),
                }}
            </div>
        </div>
    }
}
```

**พิสูจน์ด้วยเซิร์ฟเวอร์จริง — สาม flow**: (1) admin สร้างหนังสือสำเร็จ, (2) isbn ซ้ำได้ `409`, (3) ส่ง
input ผิดห้าจุดพร้อมกันได้ field-level error ครบทุกจุด:

```
=== admin login, nav shows ===
สวัสดี nan (admin)

=== admin create-book success message ===
สร้างหนังสือ 'The Pragmatic Programmer' สำเร็จ (id=3)

=== duplicate isbn create-book error ===
409 CONFLICT: ขัดแย้งกับสถานะปัจจุบัน: ข้อมูลนี้ซ้ำกับที่มีอยู่แล้วในระบบ

=== create-book field-level validation errors ===
{
  "title": "ต้องไม่เป็นค่าว่าง",
  "author": "ต้องไม่เป็นค่าว่าง",
  "isbn": "ต้องเป็นตัวเลข 10 หรือ 13 หลัก",
  "category": "ต้องไม่เป็นค่าว่าง",
  "total_copies": "ต้องมากกว่า 0"
}
```

network request ของ flow แรก (สังเกต `available_copies == total_copies` เพราะเป็นหนังสือใหม่ ยังไม่มีใคร
ยืม — ตรงกับที่ Part 92 หัวข้อ 92.8 ออกแบบ SQL ไว้ให้สองค่านี้เท่ากันเสมอตอนสร้าง):

```
--> POST http://127.0.0.1:8092/api/v1/books
<-- 201 http://127.0.0.1:8092/api/v1/books
    body={"id":3,"title":"The Pragmatic Programmer","author":"Dave Thomas","isbn":"9002490141111",
          "category":"programming","total_copies":3,"available_copies":3,"created_at":"2026-09-27T03:54:54.675172Z"}
```

**ทดสอบ route guard ของหน้านี้ด้วย member ธรรมดา** (ไม่ใช่ admin) — พยายามเข้า `/admin/new-book` ตรง ๆ ถูก
redirect ออกทันทีไปหน้าแรก:

```
=== non-admin visiting /admin/new-book -> current URL ===
http://127.0.0.1:5173/
```

**พิสูจน์ว่า backend เช็คซ้ำจริง แม้ frontend จะซ่อน UI ไปแล้ว**: สมมติผู้ใช้ member เปิด DevTools แล้วยิง
`fetch` ตรงไปที่ backend เอง (ข้าม UI ทั้งหมด) แนบ token ของ member — backend ตอบ `403` เสมอไม่ว่า
frontend จะทำอะไรก็ตาม (พิสูจน์ไว้แล้วในเทส `curl` ของ Part 92 หัวข้อ 92.8 — ยกมาซ้ำที่นี่เพื่อย้ำ):

```bash
$ curl -sS -i -X POST http://127.0.0.1:8092/api/v1/books \
  -H "Authorization: Bearer <token ของ member ธรรมดา>" -H 'Content-Type: application/json' \
  -d '{"title":"x","author":"y","isbn":"1234567890123","category":"z","total_copies":1}'
```
```
HTTP/1.1 403 Forbidden
{"error":{"code":"FORBIDDEN","message":"ไม่มีสิทธิ์ทำรายการนี้: ต้องเป็น admin เท่านั้นที่สร้างหนังสือได้"}}
```

นี่คือเหตุผลที่ `create_book` ใน `client.rs` (หัวข้อ 93.3) ไม่ต้องมี logic เช็ค role ก่อนยิง request เลย —
ถึงมันจะไม่เช็ค (หรือมีบั๊กที่ปล่อยให้ member กดปุ่มได้ด้วยวิธีใดวิธีหนึ่ง) backend ก็ปฏิเสธให้อยู่ดี ความ
ปลอดภัยของระบบไม่เคยขึ้นอยู่กับว่า frontend เขียนถูกหรือผิด

### 93.10 Loading States และ Error Handling UX

บทนี้ใช้ **สองกลไก loading ที่ต่างกัน** ตามประเภทของการดึงข้อมูล ต่อยอด Part 89 หัวข้อ 89.6:

1. **`<Suspense fallback=...>`** สำหรับข้อมูลที่โหลดตอน component mount (รายการหนังสือ, รายละเอียดหนังสือ,
   รายการที่ยืมอยู่) — ผ่าน `LocalResource`/`Resource` ที่ผูกกับ `<Suspense>` โดยอัตโนมัติ ไม่ต้องเขียน
   `if`/`match` เช็ค "กำลังโหลดอยู่ไหม" เองเลยแม้แต่จุดเดียว (ตามที่ Part 89 พิสูจน์ไว้แล้วด้วยตัวอย่าง
   `BookCountDisplay`)
2. **`Action::pending()`** สำหรับ mutation ที่ trigger จาก event ของผู้ใช้ (login, register, สร้าง
   หนังสือ) — ใช้ปิด/เปลี่ยนข้อความปุ่มระหว่างรอ response กันผู้ใช้กดซ้ำหลายครั้งจนยิง request ซ้อนกัน
   (ทุกปุ่ม submit ในบทนี้เขียน `disabled=move || xxx_action.pending().get()` ไว้เสมอ)

ส่วน error handling ใช้ `ApiError` (หัวข้อ 93.3) เป็นจุดศูนย์กลางเดียว — ทุกหน้าเรียก `.user_message()` เพื่อ
แสดง error รวม และ `.field_errors()` เพื่อแสดง error แยกตาม field เมื่อเป็น `VALIDATION_ERROR` การรวม
logic นี้ไว้ที่ type เดียว (แทนการ format string ซ้ำในทุกหน้า) ทำให้ถ้าต้องเปลี่ยนวิธีแสดง error ในอนาคต
(เช่น เพิ่ม i18n) แก้ที่จุดเดียวพอ — ตรงตามหลักการ DRY เดียวกับที่ Part 92 หัวข้อ 92.4 รวม `AppError` ไว้
จุดเดียวฝั่ง backend

ตารางสรุปว่าแต่ละสถานการณ์แสดง UI อย่างไร (ทุกแถวพิสูจน์ด้วยการรันจริงในหัวข้อก่อนหน้าแล้ว):

| สถานการณ์ | HTTP Status | `ApiError` variant | UI แสดงอะไร |
|---|---|---|---|
| username ซ้ำตอน register | 409 | `Api{code:"CONFLICT",..}` | ข้อความรวมที่ `#register-error` |
| input ผิดหลายจุดตอน register | 400 | `Api{code:"VALIDATION_ERROR",fields:[..]}` | error แยกช่องใต้แต่ละ input |
| password ผิดตอน login | 401 | `Api{code:"UNAUTHORIZED",..}` | ข้อความรวมที่ `#login-error` |
| ไม่มีสำเนาว่างตอนยืม | 409 | `Api{code:"CONFLICT",..}` | ข้อความที่ `#borrow-feedback`/`#detail-borrow-feedback` |
| member พยายามสร้างหนังสือ (ถ้าหลุด UI guard มาได้) | 403 | `Api{code:"FORBIDDEN",..}` | ข้อความที่ `#create-book-error` |
| backend ปิด/เชื่อมต่อไม่ได้ | - | `Network(..)` | "เชื่อมต่อเซิร์ฟเวอร์ไม่ได้: ..." |

### 93.11 พิสูจน์ End-to-End: รันสองเซิร์ฟเวอร์คู่กันจริง + Headless Chromium

หัวข้อนี้คือขั้นตอนที่ยืนยันว่าทุกโค้ดในบทนี้ **ทำงานได้จริงเป็นระบบเดียว** ไม่ใช่แค่ compile ผ่านแยกกัน —
ตั้งใจแยกหัวข้อนี้ไว้ต่างหากเพื่อให้เห็นภาพรวมของ workflow ทั้งหมดตั้งแต่ต้นจนจบ

**ขั้นที่ 1 — รัน backend ของ Part 92** (คัดลอกโค้ดทั้งหมดจาก Part 92 มาเป็นโปรเจกต์แยก):

```bash
$ export DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/library_api_dev"
$ sqlx migrate run
Applied 1/migrate create users (7.929216ms)
Applied 2/migrate create books (6.911648ms)
Applied 3/migrate create borrows (8.11964ms)

$ export JWT_SECRET="dev-only-secret-change-me-in-production"
$ export FRONTEND_ORIGIN="http://127.0.0.1:5173"
$ export BIND_ADDR="127.0.0.1:8092"
$ cargo run
...
library_api ฟังอยู่ที่ http://127.0.0.1:8092
Swagger UI: http://127.0.0.1:8092/swagger-ui
```

ยืนยันว่าเซิร์ฟเวอร์พร้อมจริง:

```bash
$ curl -sS -i http://127.0.0.1:8092/health
HTTP/1.1 200 OK
content-type: application/json
access-control-allow-origin: http://127.0.0.1:5173
content-length: 15

{"status":"ok"}
```

สังเกต header `access-control-allow-origin: http://127.0.0.1:5173` ที่ปรากฏแม้กับ `/health` (endpoint ที่
ไม่ต้อง auth) — เพราะ `CorsLayer` ของ Part 92 หัวข้อ 92.10 ครอบทั้ง Router ไม่ใช่แค่ endpoint ที่ต้อง auth
เท่านั้น

**ขั้นที่ 2 — build และรัน frontend ของบทนี้**:

```bash
$ trunk serve --port 5173 --address 127.0.0.1
2026-09-27T03:56:00.669747Z  INFO Starting trunk 0.21.14
2026-09-27T03:56:00.710620Z  INFO starting build
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.12s
2026-09-27T03:56:02.261077Z  INFO applying new distribution
2026-09-27T03:56:02.262920Z  INFO success
2026-09-27T03:56:02.262920Z  INFO server listening at:
2026-09-27T03:56:02.263075Z  INFO     http://127.0.0.1:5173/
```

ยืนยันว่า `trunk serve` มี **SPA fallback ในตัว** อยู่แล้ว — ยิง request ตรงไปที่ sub-route (ไม่ใช่แค่ `/`)
ก็ได้ HTML shell กลับมา (ให้ `leptos_router` ฝั่ง client ทำ routing ต่อเอง ไม่ใช่ 404 จาก dev server):

```bash
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:5173/
200
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:5173/my-borrowed
200
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:5173/admin/new-book
200
```

(นี่เป็นจุดที่ต้องระวังถ้าจะทดสอบ production build (`trunk build` แล้ว serve ไฟล์ใน `dist/` ด้วย static
file server ธรรมดา) — ดูกับดักที่พบบ่อยข้อ 5 ท้ายบทนี้ ซึ่งเป็นเรื่องที่ Part 94 ต้องจัดการตอน deploy จริง)

**ขั้นที่ 3 — ขับทั้ง flow ด้วย headless Chromium ผ่าน Playwright** สคริปต์เปิดเบราว์เซอร์จริง (ไม่มีหน้าจอ)
แล้วทำตามลำดับ: สมัครสมาชิก → สมัครซ้ำ (409) → สมัครด้วย input ผิด (400) → login → login ผิด (401) → ดู
รายการหนังสือ → ค้นหา → ยืม → ดูหน้าที่ยืมอยู่ → คืน → ทดสอบ route guard ตอน logout → login เป็น admin →
สร้างหนังสือ → สร้างซ้ำ (409 isbn) → สร้างด้วย input ผิด (400) → ยืมซ้ำเล่มเดิม (409) ทั้งหมดในสคริปต์เดียว
รันจบใน**ไม่ถึง 20 วินาที** สำหรับ 15 ขั้นตอน (ดูผลลัพธ์ทีละขั้นตอนที่ยกมาใน log จริงกระจายอยู่ในหัวข้อ
93.6-93.9 ข้างบนแล้ว) สรุปสถิติจาก run จริงครั้งหนึ่ง:

```
console messages captured during whole run:
- 12 คำเตือน "The `integrity` attribute is currently ignored ..." (คำเตือนของ Chromium เรื่อง
  <link rel="preload"> ที่ trunk แทรกให้ ไม่ใช่บั๊กของโค้ดเรา — ไม่กระทบการทำงาน)
- 5 [console:error] "Failed to load resource: the server responded with a status of 4xx" —
  ทุกอันคือ error ที่ "ตั้งใจให้เกิด" ในสคริปต์ทดสอบ (409/400/401 ตามลำดับ) ไม่ใช่บั๊ก
- 0 [pageerror] — ไม่มี panic หรือ unhandled JS exception เกิดขึ้นเลยตลอดทั้ง flow
```

`console:error` ที่ Chromium log ให้ทุกครั้งที่ `fetch` ได้ status `>= 400` เป็นพฤติกรรมปกติของเบราว์เซอร์
(ไม่ใช่สัญญาณว่าโค้ดเรามีบั๊ก) — จุดที่ต้องเช็คจริง ๆ คือ **`[pageerror]` ต้องเป็นศูนย์เสมอ** เพราะนั่นหมายถึง
panic ของ WASM หรือ unhandled JS exception ซึ่งจะพังทั้งแอป ไม่ใช่แค่ request เดียวที่ล้มเหลว

**ขั้นที่ 4 — ทดสอบ session persistence ข้าม hard reload** (สำคัญเพราะพิสูจน์ว่า `localStorage` ทำงานจริง
ไม่ใช่แค่ signal ในหน่วยความจำที่หายไปทุกครั้งที่ WASM re-initialize):

```
logged in, whoami = สวัสดี nan (admin)
after hard reload, whoami = สวัสดี nan (admin)
book detail page: The Rust Programming Language
ผู้เขียน: Steve Klabnik
...
สำเนาว่าง 2 จาก 2 เล่ม
ยืมหนังสือเล่มนี้
```

`page.reload()` ของ Playwright คือการโหลดเอกสารใหม่ทั้งหมด (WASM module ถูก re-initialize จากศูนย์ ไม่ใช่
แค่ re-render component) — การที่ `whoami` ยังแสดงชื่อผู้ใช้เดิมได้หลัง reload พิสูจน์ว่า
`AuthContext::new()` (หัวข้อ 93.4) โหลด session จาก `localStorage` จริงตอน WASM เริ่มทำงานใหม่ทุกครั้ง ไม่
ใช่แค่ตอนแอปยังไม่ได้ reload

**ขั้นที่ 5 — ทดสอบว่า token ที่หมดอายุ/ไม่ถูกต้องถูกล้างออกอัตโนมัติจริง**: เขียน session ปลอมที่มี
`access_token: "expired.fake.token"` ลง `localStorage` ตรง ๆ ผ่าน `page.evaluate()` (จำลองสถานการณ์ปิด
แท็บทิ้งไว้นานเกิน 1 ชั่วโมงตามอายุ token ของ Part 92) แล้ว reload:

```
whoami element present after invalid-token reload? false
localStorage after invalid-token check: null
nav content: หนังสือ | ที่ยืมอยู่ | เข้าสู่ระบบ | สมัครสมาชิก
```

นี่คือผลของ `Effect::new` ใน `main.rs` (หัวข้อ 93.5) ที่เรียก `GET /api/v1/me` ตอน app เริ่มทำงาน — backend
ตอบ `401` เพราะ token ปลอมนั้น decode ไม่ผ่านเลย (`jsonwebtoken::decode` ล้มเหลวตั้งแต่ signature ไม่ตรง)
ทำให้ `api::me(&token).await.is_err()` เป็น `true` แล้ว `auth.logout()` ถูกเรียกทันที ล้าง `localStorage`
และ signal กลับเป็น `None` — nav bar จึงกลับไปแสดงลิงก์ "เข้าสู่ระบบ"/"สมัครสมาชิก" ให้เห็นถูกต้อง ไม่มี
การค้างแสดงสถานะ "login อยู่" ที่ไม่จริงเลย

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `Action::new` ปฏิเสธ future ของ `gloo-net` เพราะไม่ satisfy `Send`

ระหว่างพัฒนาหน้า login/register (หัวข้อ 93.6) การเขียนตามรูปแบบที่ Part 89 หัวข้อ 89.6 สอนไว้ตรง ๆ
(`Action::new(move |req: &LoginRequest| { ... async move { api::login(&req).await } ... })`) เจอ compile
error ยาวทันที (ตัดมาเฉพาะส่วนสำคัญ):

```
error[E0277]: `Rc<RefCell<leptos::web_sys::js_sys::futures::Inner>>` cannot be sent between threads safely
   --> src/pages/login.rs:18:9
    |
 18 | /         Action::new(move |req: &LoginRequest| {
    | |__________^ `Rc<RefCell<...>>` cannot be sent between threads safely
    |
    = help: within `{async block@src/pages/login.rs:20:13: 20:23}`, the trait `Send` is not implemented
      for `Rc<RefCell<leptos::web_sys::js_sys::futures::Inner>>`
note: required because it appears within the type `JsFuture`
note: required by a bound in `leptos::prelude::Action::<I, O>::new`
    |
697 |         Fu: Future<Output = O> + Send + 'static,
    |                                  ^^^^ required by this bound in `Action::<I, O>::new`
```

**สาเหตุ**: `gloo-net`'s `Request::send()` ห่อ `wasm_bindgen_futures::JsFuture` ซึ่งข้างในเก็บ
`Rc<RefCell<...>>` (ไม่ใช่ `Arc<Mutex<...>>`) เพราะ WASM ในเบราว์เซอร์รันบนเธรดเดียวเสมอ ไม่มีเหตุผลต้องเสีย
ค่าใช้จ่ายของ atomic/lock ข้าม thread — แต่ `Action::new` (ตัวเดียวกับที่ Part 89 หัวข้อ 89.6 ใช้กับ server
function) ต้องการ `Future: Send` เพราะมันถูกออกแบบให้ใช้ได้ทั้งฝั่ง `ssr` (ที่รันบน tokio multi-thread
runtime จริง ต้อง `Send`) และฝั่ง `csr`/`hydrate` — เมื่อ future ของเราไม่ `Send` เพราะพึ่ง `gloo-net`
ตรง ๆ (ไม่ได้มาจาก server function ที่ compile คนละแบบตาม feature) จึงชนกับ bound นี้ทันที

**วิธีแก้**: ใช้ `Action::new_local` แทน — เป็นฟังก์ชันที่ `reactive_graph` (crate ที่ Leptos สร้าง
`Action` อยู่บน) เตรียมไว้ให้เฉพาะสำหรับสถานการณ์นี้พอดี (ไม่ต้องการ `Send` แต่ panic ถ้าถูกเข้าถึงจากเธรด
อื่นที่ไม่ใช่เธรดที่สร้างมัน — ซึ่งไม่มีปัญหาเลยสำหรับ CSR ในเบราว์เซอร์ที่มีเธรดเดียวอยู่แล้ว):

```rust
let login_action: Action<LoginRequest, Result<Session, ApiError>> =
    Action::new_local(move |req: &LoginRequest| {  // เปลี่ยนจาก Action::new
        let req = req.clone();
        async move { /* ... */ }
    });
```

Part 89 หัวข้อ 89.6 เคยเจอปัญหาแบบเดียวกันนี้มาก่อนแล้วกับ `Resource::new`/`gloo_timers` และแก้ด้วย
`LocalResource::new` — บทนี้คือหลักฐานว่าปัญหาเดียวกันเกิดกับ `Action` ทุกครั้งที่ future มาจาก
`gloo-net`/`web-sys` ตรง ๆ (ไม่ผ่าน server function) ซึ่งเป็นสถานการณ์ปกติของ CSR app ที่คุยกับ REST API
แยก — ไม่ใช่กรณีพิเศษที่พบยาก

### 2. `Resource::new` ต้องการให้ผลลัพธ์เป็น `Serialize`/`Deserialize` แม้เป็นแอป CSR ล้วน ๆ

ก่อนจะเปลี่ยนไปใช้ `LocalResource::new` ตามหัวข้อ 93.7-93.8 การลองใช้ `Resource::new` ตรง ๆ (แก้ปัญหา
`Send` ด้วยการยอมรับว่า `borrow_action.dispatch` คืนค่าไม่ใช่ `()` ไปแล้ว) ยังเจอ compile error อีกชุดที่ดู
ไม่เกี่ยวกับ `Send` เลย:

```
error[E0277]: the trait bound `api::types::ApiError: serde::Serialize` is not satisfied
   --> src/pages/my_borrowed.rs:28:20
    |
 28 |       let borrowed = Resource::new(
    |  ____________________^
    |
    = note: required for `Result<api::types::BorrowListResponse, api::types::ApiError>` to implement `Serialize`
    = note: required for `JsonSerdeCodec` to implement `Encoder<Result<...>>`
note: required by a bound in `leptos::prelude::Resource::<T>::new`
    |
1030 |     JsonSerdeCodec: Encoder<T> + Decoder<T>,
    |                     ^^^^^^^^^^ required by this bound in `Resource::<T>::new`
```

**สาเหตุ**: `Resource::new` ออกแบบมาให้ทำงานได้ทั้งฝั่ง SSR (ต้อง serialize ผลลัพธ์ฝังลง HTML ให้ WASM
ฝั่ง client อ่านตอน hydrate ตามที่ Part 89 หัวข้อ 89.7 อธิบายกลไก hydration ไว้) และฝั่ง CSR — เพราะฉะนั้น
มันจึงเรียกร้อง `T: Serialize + Deserialize` เสมอ **ไม่ว่าโปรเจกต์จะเปิด feature `ssr` จริงหรือไม่ก็ตาม**
เพราะ bound ถูกกำหนดไว้ที่ตัว type เดียวกันทั้งสอง mode `ApiError` ของบทนี้ไม่ได้ derive `Serialize`
(ตั้งใจไม่ทำเพราะไม่มีเหตุผลต้อง serialize error ไปที่ไหน) จึงชน bound นี้

**วิธีแก้**: `LocalResource::new` (ตัวเดียวกับที่แก้ปัญหา `Send` ในหัวข้อก่อน) ไม่มี bound นี้เลย เพราะมัน
ประกาศไว้ตรง ๆ ว่า "loads its data locally on the client" (จากคอมเมนต์ในซอร์สโค้ดจริงของ
`leptos_server-0.8.8/src/local_resource.rs`) — ไม่มีด้าน SSR ให้ serialize ข้าม เพราะฉะนั้นไม่ต้องมี bound
`Serialize` เลยแม้แต่นิดเดียว วิธีแก้นี้เข้ากันได้ดีกับกับดักข้อ 1 พอดี เพราะทั้งสองปัญหาชี้ไปทางเดียวกัน:
**สำหรับ CSR app ที่คุยกับ REST API ผ่าน `gloo-net` ตรง ๆ ให้ใช้ `Action::new_local`/`LocalResource::new`
เป็นค่าเริ่มต้นเสมอ ไม่ใช่ `Action::new`/`Resource::new`** (สองตัวหลังนี้ออกแบบมาสำหรับ server function
ของ Leptos ที่ future เป็น `Send` ได้ตามปกติเพราะมาจาก `reqwest`/`sqlx` ฝั่ง server จริง)

### 3. เครื่องหมาย `>=` ในค่า attribute ที่ไม่ห่อด้วย `{}` ทำให้ `view!` macro parse ผิด

ระหว่างพัฒนาปุ่ม "ถัดไป" ของ pagination (หัวข้อ 93.7) การเขียน `disabled=move || page.get() >=
list.meta.total_pages` แบบไม่ห่อวงเล็บปีกกา (ตามสไตล์ปุ่ม "ก่อนหน้า" ที่ใช้ `<=` แล้วทำงานถูกต้องดี) ทำให้
`trunk build` compile ผ่านสนิท **ไม่มี error หรือ warning ใด ๆ เลย** แต่ DOM จริงที่ได้ตอนรันกลับพังแบบ
ประหลาด — ปุ่มแสดงข้อความที่ไม่ควรมี:

```html
<button id="next-page" disabled="1">= list.meta.total_pages on : click = move | _ | page.update(|p| *p += 1) &gt;ถัดไป</button>
```

**สาเหตุ**: `view!` macro ของ Leptos (สร้างจาก `rstml`) parse ค่า attribute ที่เขียนแบบ bare Rust
expression (ไม่ห่อ `{}`) ด้วยการอ่าน token ไปจนกว่าจะเจอสิ่งที่ตีความได้ว่าเป็นจุดจบของ attribute — เมื่อ
เจอ `>` เดี่ยว ๆ ใน `>=` มันตีความผิดว่าเป็นจุดปิด tag (เหมือน HTML/JSX ที่ `>` ปิด opening tag) ทำให้
expression `page.get() >= list.meta.total_pages` ถูกตัดครึ่งกลาง `>` แล้วเนื้อความที่เหลือ (`=
list.meta.total_pages`) กลายเป็น text content ที่รั่วเข้าไปปนกับ attribute ตัวถัดไปและ children ของ tag —
compile ผ่านเพราะ macro ยังสร้างโค้ด Rust ที่ valid ได้ (แค่ผลลัพธ์ทางความหมายผิดจากที่ตั้งใจ) เป็นกับดักที่
ร้ายกาจเพราะไม่มีสัญญาณเตือนใด ๆ ตอน compile เลย ต้องเห็น DOM จริงถึงจะรู้ตัว

**วิธีแก้**: ห่อ expression ที่มี `>`/`>=` (หรือ operator อื่นที่มีโอกาสตีความผิดเป็นส่วนของ tag) ด้วย `{}`
เสมอ:

```rust
// ผิด — compile ผ่านแต่ DOM พัง
disabled=move || page.get() >= list.meta.total_pages

// ถูก — ห่อด้วย {} บอก macro ชัด ๆ ว่านี่คือ Rust expression block ทั้งก้อน ไม่ใช่ markup
disabled=move || { page.get() >= list.meta.total_pages }
```

หลังแก้ ปุ่มแสดงถูกต้อง (`disabled=""` ตอนอยู่หน้าสุดท้าย, ไม่มี `disabled` ตอนยังมีหน้าถัดไป) ตรงตามที่
ตั้งใจ — ข้อสรุปที่ปลอดภัยที่สุดคือ**ห่อ closure ของทุก attribute ที่มี comparison operator ด้วย `{}` เป็น
นิสัย** ไม่ต้องรอให้เจอปัญหาก่อนค่อยแก้เฉพาะจุด

### 4. `gloo-net`'s `.query()` ต้องการคีย์เป็น `&str` ไม่ใช่ `String`

ตอนเขียน `list_books` ในหัวข้อ 93.3 การสร้าง `Vec<(String, String)>` แล้วส่งเข้า `.query(...)` ตรง ๆ (เพราะ
คีย์บางตัวอยากสร้างแบบ dynamic) เจอ compile error:

```
error[E0271]: type mismatch resolving `<Vec<(String, String)> as IntoIterator>::Item == (&str, _)`
  --> src/api/client.rs:88:16
   |
88 |         .query(query)
   |          ----- ^^^^^ expected `(&str, _)`, found `(std::string::String, std::string::String)`
   |
note: required by a bound in `RequestBuilder::query`
   |
112 |         T: IntoIterator<Item = (&'a str, V)>,
   |                         ^^^^^^^^^^^^^^^^^^^ required by this bound in `RequestBuilder::query`
```

**สาเหตุ**: `gloo-net` 0.6.0 กำหนด signature ของ `.query()` ให้รับ `IntoIterator<Item = (&'a str, V)>` —
คีย์ต้องเป็น `&str` เสมอ (เพราะชื่อ query parameter เป็นค่าคงที่ที่รู้ตอน compile time อยู่แล้วเสมอ ไม่มี
เหตุผลต้องเป็น owned `String`) ในขณะที่ **ค่า** (`V`) ยืดหยุ่นกว่า (generic ที่ constraint แค่
`AsRef<str>`-like ในทางปฏิบัติ) เพราะค่ามาจาก runtime จริง (ผลจาก `.to_string()` ของตัวเลข หรือค่าจาก
signal)

**วิธีแก้**: ใช้ string literal เป็นคีย์เสมอ (ซึ่งเป็น `&'static str` โดยธรรมชาติ) เข้าคู่กับ `String` ที่
เป็นค่า:

```rust
// ผิด
let mut query: Vec<(String, String)> = Vec::new();
query.push(("page".into(), page.to_string()));

// ถูก
let mut query: Vec<(&str, String)> = Vec::new();
query.push(("page", page.to_string()));
```

กับดักนี้ชี้ให้เห็นหลักการที่กว้างกว่าเรื่อง `gloo-net` เพียงอย่างเดียว: เวลาเจอ compile error ประเภท "type
mismatch resolving ... == (X, Y)" ให้อ่าน **bound ที่ note บอกไว้** เสมอ (ในเคสนี้คือ
`RequestBuilder::query`'s `where T: IntoIterator<Item = (&'a str, V)>`) แทนที่จะเดาว่า API ควรรับอะไร —
เอกสาร/signature จริงของ crate เวอร์ชันที่ใช้อยู่คือความจริงเดียวที่เชื่อได้ ไม่ใช่ความจำจาก crate อื่นหรือ
เวอร์ชันอื่นที่คุ้นเคย

### 5. Static file server ธรรมดา (ไม่มี SPA fallback) ทำให้ route ของ `leptos_router` ได้ `404` เมื่อ hard navigate

ระหว่างตรวจสอบเนื้อหาบทนี้ หลังจาก `trunk build` แล้วเสิร์ฟโฟลเดอร์ `dist/` ด้วย static file server
ธรรมดา (`http-server` แบบ default ไม่เปิด SPA fallback) — เปิดหน้าแรก (`/`) ได้ปกติ แต่พอ Playwright สั่ง
`page.goto('http://127.0.0.1:5173/register')` ตรง ๆ (จำลองผู้ใช้พิมพ์ URL เอง หรือ bookmark ไว้) กลับได้
`404` จาก **เว็บเซิร์ฟเวอร์เอง** (ไม่ใช่จากแอป Leptos เลย เพราะ WASM ยังไม่ได้โหลดขึ้นมาด้วยซ้ำ):

```
[2026-09-27T03:51:54.779Z]  "GET /register" "Mozilla/5.0 (X11; Linux x86_64) ... HeadlessChrome/141.0..."
[2026-09-27T03:51:54.783Z]  "GET /register" Error (404): "Not found"
```

**สาเหตุ**: `leptos_router` (เหมือน SPA router ทุกตัวไม่ว่าจะเป็น React Router, Vue Router) ทำ routing
**ฝั่ง client เท่านั้น** — มันคือ JavaScript/WASM ที่อ่าน `window.location` แล้วตัดสินใจ render component
ไหน ทั้งหมดนี้เกิด**หลังจาก**หน้าเว็บโหลดสำเร็จแล้ว แต่การ hard navigate ไปที่ `/register` ตรง ๆ (ไม่ผ่าน
การคลิก `<A>` ภายในแอปที่กำลังรันอยู่) คือการขอไฟล์ที่ path `/register` จาก**เว็บเซิร์ฟเวอร์**โดยตรง — ถ้า
เว็บเซิร์ฟเวอร์ไม่มีไฟล์ที่ path นั้นจริง (มีแต่ `index.html`, `*.js`, `*.wasm` ที่ root) มันก็ตอบ `404`
ตามธรรมชาติของ static file server ทั่วไป โดยไม่รู้จัก "route ของแอป" เลย

**วิธีแก้**: เว็บเซิร์ฟเวอร์ที่ serve ไฟล์ static ของ SPA ต้องตั้งค่า **SPA fallback** (ทุก path ที่ไม่ตรง
กับไฟล์จริงให้ตอบ `index.html` กลับไปแทน `404` แล้วให้ router ฝั่ง client จัดการเอง) — `trunk serve` (dev
server ที่ใช้ตลอดบทนี้) มี fallback นี้อยู่แล้ว**โดยอัตโนมัติ** ไม่ต้องตั้งอะไรเพิ่ม (พิสูจน์ไว้แล้วในหัวข้อ
93.11) แต่ production static server ธรรมดาไม่มี — ต้องตั้งเองเสมอ (เช่น `serve -s dist`, หรือ nginx
`try_files $uri /index.html;`) นี่คือประเด็นสำคัญที่ **Part 94 (deployment) ต้องจัดการตอน deploy จริง**
เพราะ production build จริงจะไม่ใช้ `trunk serve` (ที่มีไว้สำหรับ dev เท่านั้น) แต่ใช้ `trunk build`
(สร้าง `dist/` แบบ optimize) แล้ว serve ด้วยเว็บเซิร์ฟเวอร์ตัวจริงที่ต้องตั้ง SPA fallback เองให้ถูกต้อง

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เพิ่ม `<select>` dropdown ที่หน้ารายการหนังสือ (`BooksPage`) ให้เลือก `category` ได้ (ตอนนี้
   หัวข้อ 93.7 ยังไม่ได้ทำ UI ให้กรอก `category` แม้ `ListBooksParams`/`list_books` จะรองรับ parameter นี้
   อยู่แล้ว) — ทดสอบด้วย headless Chromium จริงกับ backend ของ Part 92 ที่มีหนังสือหลาย category ว่า
   filter ทำงานถูกต้อง (hint: เพิ่ม `RwSignal<String>` อีกตัวสำหรับ category ที่เลือก แล้วเพิ่มมันเข้าไป
   ในรายการ signal ที่ `LocalResource::new`'s closure `.get()` เหมือน `page`/`q`/`available_only`)

2. **[กลาง]** ตอนนี้ถ้า access token หมดอายุ**ระหว่าง**ที่ผู้ใช้กำลังใช้แอปอยู่ (ไม่ใช่แค่ตอนเปิดแอปใหม่ตาม
   ที่หัวข้อ 93.11 ขั้นที่ 5 ทดสอบไว้) เช่นกดปุ่ม "ยืม" ตอน token หมดอายุไปแล้วพอดี ผู้ใช้จะเห็นแค่ข้อความ
   `401 UNAUTHORIZED: ...` ที่ `#borrow-feedback` โดยไม่มีอะไรเกิดขึ้นต่อ — ให้แก้ `ApiError` และจุดที่เรียก
   `borrow_book`/`return_book`/`create_book` ทุกจุด ให้ตรวจว่าถ้า error เป็น `Api{status: 401, ..}` ให้
   เรียก `auth.logout()` อัตโนมัติแล้ว redirect ไปหน้า login (hint: อาจเขียนเป็นฟังก์ชันช่วยกลาง เช่น
   `fn handle_api_error(auth: AuthContext, navigate: impl Fn(&str), err: &ApiError)` ที่ทุกหน้าเรียกใช้
   ร่วมกันหลัง action ล้มเหลว แทนการเขียน `if let ApiError::Api{status:401,..}` ซ้ำในทุกไฟล์)

3. **[ยาก]** เขียนฟังก์ชัน decode JWT payload ฝั่ง client (แยก JWT ด้วย `.` แล้ว base64-decode ส่วนที่สอง
   เป็น JSON — **ไม่ต้อง verify signature** เพราะ frontend ไม่มี `jwt_secret` อยู่แล้วและไม่ควรมีด้วย เป้า
   คือแค่อ่าน `exp` claim เพื่อรู้ว่า token จะหมดอายุเมื่อไหร่) แล้วแสดง countdown "token จะหมดอายุใน X
   นาที" ที่ nav bar โดยไม่ต้องเรียก `/api/v1/me` ซ้ำเพื่อรู้เวลาหมดอายุ (hint: `Claims` struct ของ Part 92
   หัวข้อ 92.5 มี field `exp: usize` เป็น Unix timestamp — ต้องหา crate สำหรับ base64 decode เอง เช่น
   `base64` crate เพิ่มเข้ามาใน `Cargo.toml`, ระวังเรื่อง URL-safe base64 variant ที่ JWT ใช้ ต่างจาก
   standard base64 เล็กน้อย)

4. **[ยาก/ประยุกต์]** ตอนนี้ถ้าเปิดแอปนี้สองแท็บพร้อมกัน แล้ว logout จากแท็บหนึ่ง อีกแท็บจะยังแสดงว่า login
   อยู่จนกว่าจะ reload เอง (เพราะ signal ของแต่ละแท็บเป็นอิสระจากกัน แม้ `localStorage` จะถูกล้างไปแล้วก็
   ตาม) — ให้ implement การ sync สถานะ auth ข้ามแท็บโดยฟัง [`storage` event](https://developer.mozilla.org/en-US/docs/Web/API/Window/storage_event)
   ของเบราว์เซอร์ (event นี้ยิงให้ทุกแท็บอื่นที่เปิด origin เดียวกันเมื่อ `localStorage` ถูกเปลี่ยนจากแท็บ
   ใดแท็บหนึ่ง — ยกเว้นแท็บที่เป็นคนเปลี่ยนเอง) แล้วอัปเดต `auth.session` signal ให้ตรงกันทุกแท็บโดย
   อัตโนมัติ (hint: ต้องใช้ `web-sys`/`wasm-bindgen` ตรง ๆ ต่อยอด Part 87 เพื่อ `add_event_listener` บน
   `window` object สำหรับ event `"storage"` เพราะ `gloo-events`/`leptos` ไม่มี wrapper สำเร็จรูปสำหรับ
   event นี้โดยเฉพาะ — ต้องระวังเรื่อง closure lifetime ที่ต้อง `.forget()` ไม่ให้ callback ถูก drop ก่อน
   ที่ event จะยิงจริง ตามที่ Part 87 อธิบายไว้เรื่อง `Closure::wrap`)

## สรุป

บทนี้สร้าง frontend ที่สมบูรณ์และรันได้จริง 100% สำหรับระบบห้องสมุด/ยืม-คืนหนังสือ ด้วย **Leptos แบบ CSR
ล้วน ๆ** ที่คุยกับ backend ของ Part 92 ผ่าน `gloo-net` ตรงตาม API contract ทุกประการ — ครอบคลุมการจัดการ
session (signal + `localStorage` ผ่าน `gloo-storage`), routing หลายหน้าพร้อม route guard ด้วย
`leptos_router`, ฟอร์ม login/register/สร้างหนังสือที่แสดง error จริงจาก backend ทั้งแบบรวมและแบบแยก field,
หน้ารายการ/รายละเอียดหนังสือพร้อม pagination/filter/ยืม, หน้า "ที่ยืมอยู่" พร้อมคืน, และ admin-only UI ที่
ย้ำหลักการ defense in depth ว่าการเช็คสิทธิ์ฝั่ง client เป็นแค่ UX ไม่ใช่ความปลอดภัยจริง — ทุก flow ถูก
พิสูจน์ด้วยการรัน backend และ frontend คู่กันจริงแล้วขับด้วย headless Chromium ผ่าน Playwright จริง ไม่มี
การแต่งผลลัพธ์ขึ้นเองแม้แต่จุดเดียว รวมถึง compile error จริงห้าจุดที่เจอระหว่างพัฒนา (`Send` bound ของ
`Action`/`Resource`, การ parse attribute ของ `view!` macro, และ signature ของ `gloo-net`'s `.query()`)

**สิ่งที่ Part 94 (Full-Stack Project 3/3: Integration และ Deployment) ต้องใช้จากบทนี้**:

- **โครงสร้างโปรเจกต์ frontend**: `library_frontend/` แบบ CSR ที่ build ด้วย `trunk build` ได้ static
  asset ชุดหนึ่ง (`index.html` + `*.js` + `*.wasm`) ที่ deploy ไปที่ไหนก็ได้ที่ serve static file ได้
  (ไม่ต้องมี Rust runtime ฝั่ง server เลยสำหรับส่วนนี้ ต่างจาก backend ของ Part 92 ที่ต้องมี)
- **ข้อกำหนดเรื่อง SPA fallback**: web server ที่ serve `dist/` ของ frontend ต้องตั้งค่าให้ทุก path ที่ไม่
  ตรงกับไฟล์จริง fallback ไปที่ `index.html` (ดูกับดักที่พบบ่อยข้อ 5) ไม่อย่างนั้น route ของ
  `leptos_router` ที่ไม่ใช่ `/` จะได้ `404` ทันทีที่ผู้ใช้ reload หน้าหรือเข้าผ่าน bookmark/ลิงก์ตรง
- **ค่า `API_BASE`/`FRONTEND_ORIGIN` ที่ต้องตรงกันสองฝั่งเสมอ**: `client.rs` hardcode
  `http://127.0.0.1:8092` ไว้สำหรับ dev — ตอน deploy จริง Part 94 ต้องเปลี่ยนค่านี้ให้ตรงกับ URL จริงของ
  backend (และตั้ง `FRONTEND_ORIGIN` ของ backend ให้ตรงกับ URL จริงของ frontend เช่นกัน ไม่อย่างนั้น CORS
  จะบล็อกทุก request ทันทีเหมือนที่ Part 92 หัวข้อ 92.10 อธิบายกลไกไว้)
- **การตรวจสอบ session ผ่าน `GET /api/v1/me`**: กลไกในหัวข้อ 93.4/93.5 ที่ตรวจ token กับ backend จริงตอน
  app เริ่มทำงาน คือสิ่งที่ทำให้ frontend "ทนทาน" ต่อการที่ access token หมดอายุระหว่างที่ไม่ได้ใช้งาน
  Part 94 ควรตรวจสอบว่ากลไกนี้ยังทำงานถูกต้องหลัง deploy จริงด้วย (เช่นทดสอบด้วย token ที่หมดอายุจริงข้าม
  environment จริง)

ทั้งสามบท (92, 93, 94) ประกอบกันเป็น capstone เดียวที่แสดงให้เห็นว่าทุกเทคนิคที่เรียนมาตลอด Module 4-5 ของ
หลักสูตรนี้ทำงานร่วมกันเป็นระบบจริงได้อย่างไร — จาก database schema ที่ปลอดภัยจาก race condition ไปจนถึง
UI ที่ผู้ใช้จริงจับต้องได้ และท้ายที่สุดคือการนำระบบทั้งหมดออกไปให้ใช้งานได้จริงใน Part 94

---

**Part ก่อนหน้า:** [Full-Stack Project (1/3): ออกแบบและสร้าง Backend](part-092-fullstack-backend.md) | **Part ถัดไป:** [Full-Stack Project (3/3): Integration และ Deployment](part-094-fullstack-integration.md)
