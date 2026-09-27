# Part 94: Full-Stack Project (3/3): Integration และ Deployment

> โมดูล: โปรเจกต์ Capstone (Full-Stack) | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ชัดว่าทำไม "backend ที่รันได้จริง" (Part 92) และ "frontend ที่รันได้จริง" (Part 93) ที่รันคู่กัน
  คนละ origin ด้วย CORS ยังไม่ใช่ "แอปที่ deploy ได้จริง" — และตัดสินใจได้ว่าจะปิดช่องว่างนั้นด้วยแนวทางไหน
  ระหว่าง "ให้ backend serve frontend เอง" กับ "แยกเป็นสองบริการอิสระ" โดยรู้ข้อแลกเปลี่ยนของทั้งสองทาง
- ทำ **production build** ของทั้งสองฝั่งได้จริง — `trunk build --release` (ต่อยอด Part 86/87 เรื่อง WASM
  build) สำหรับ frontend และ `cargo build --release` สำหรับ backend — พร้อมอ่านและอธิบายขนาด artifact ที่
  ได้จริง (ไม่ใช่แค่ "เดา" ว่าเล็กลง)
- implement **single-origin serving** จริง: เพิ่ม `tower_http::services::ServeDir`/`ServeFile` เข้าไปใน
  Router ของ Part 92 (ต่อยอด Part 65 เรื่อง middleware chain) ให้ backend process เดียวเสิร์ฟทั้ง REST API
  และไฟล์ static ของ frontend พร้อม SPA fallback ที่ถูกต้อง (คืน `index.html` พร้อม status `200` ไม่ใช่
  `404`) แล้วพิสูจน์ด้วย `curl` และเบราว์เซอร์จริงว่าไม่มี cross-origin request หลงเหลืออยู่เลยแม้แต่จุดเดียว
- ย้ายค่า config จาก "ค่า default ที่สะดวกสำหรับ dev" ของ Part 92 (พอร์ตตายตัว, CORS เปิดกว้าง) มาเป็น
  **environment-variable-driven config** ที่ปลอดภัยสำหรับ production ตามหลัก "ไม่มี secret ในซอร์สโค้ด" พร้อม
  เขียน `.env.example` และโค้ดอ่าน config จริงที่ fallback อย่างปลอดภัย
- เขียน **multi-stage Dockerfile** จริงที่ build ทั้ง Leptos frontend bundle และ Axum backend binary แล้ว
  รวมเป็น image เดียวที่รัน combined server จากหัวข้อก่อนหน้า — และตัดสินใจได้ว่าจะรัน `sqlx migrate run`
  ตอน container startup อัตโนมัติ หรือแยกเป็น deploy step ของตัวเอง โดยเข้าใจข้อแลกเปลี่ยนเรื่อง multi-instance
  deploy (ต่อยอด Part 71/82)
- ขยาย `/health` ของ Part 92 ให้ตรวจสถานะ database จริง (ไม่ใช่แค่ "process ยังไม่ตาย") ตามที่ Part 81 สอนไว้
  เรื่อง readiness probe และสลับรูปแบบ log ระหว่าง pretty (dev) กับ JSON (production) ได้ด้วย environment
  variable เดียว (ต่อยอด Part 60 เรื่อง `tracing`)
- **รัน combined application จริงเป็นหนึ่งหน่วยเดียว** (ทั้งแบบ binary ตรงและแบบ container ถ้า Docker ใช้งาน
  ได้ในสภาพแวดล้อมนั้น) แล้วพิสูจน์ full user journey ทั้งหมด (สมัคร → login → ดูรายการ → ยืม → คืน) ด้วย
  `curl` และด้วย headless Chromium จริง ปิดจบ 3 บท capstone ของ Module 4-5 ด้วยแอปที่ "deploy ได้จริง" ไม่ใช่
  แค่ "รันได้ในเครื่องเดียวตอน dev"

## ความรู้ที่ต้องมีมาก่อน

บทนี้คือ **capstone บทที่ 3 จาก 3 บท** และเป็นบทปิดของงานลงมือทำ (hands-on) ของ Module 5 — มันไม่สอนเทคนิคใหม่
ทีละชิ้นเหมือนบทปกติ แต่เอาสิ่งที่ Part 92/93 สร้างไว้มา "ปิดช่องว่างสุดท้าย" ให้กลายเป็นแอปที่ deploy ได้จริง
ความรู้ที่ต้องแน่นก่อนอ่านบทนี้:

- **Part 92 (Full-Stack Project 1/3: Backend) — ต้องอ่านทั้งบทก่อน**: บทนี้แก้ไข/ต่อยอดจากไฟล์จริงของ Part
  92 ทุกไฟล์ (`src/lib.rs`, `src/main.rs`, `src/routes/health.rs` ฯลฯ) — ไม่ได้เขียนใหม่จากศูนย์ ถ้าจำ
  โครงสร้างโปรเจกต์ (`build_app`, `AppState`, `AppError`) หรือ API contract (หัวข้อ 92.1) ของบทนั้นไม่ชัด
  ต้องกลับไปอ่านก่อน เพราะบทนี้จะไม่ทวนซ้ำรายละเอียดของ endpoint ทั้ง 10 ตัวอีก
- **Part 93 (Full-Stack Project 2/3: Frontend) — ต้องอ่านทั้งบทก่อนเช่นกัน**: บทนี้แก้ไข `src/api/client.rs`
  ของ Part 93 (เปลี่ยนวิธีกำหนด `API_BASE`) และใช้ผลลัพธ์ของ `trunk build --release` ที่มาจากโครงสร้าง
  โปรเจกต์เดียวกับที่ Part 93 หัวข้อ 93.2 วางไว้ — ถ้าจำโครงสร้าง `library_frontend/` ไม่ชัด ต้องทวนก่อน
- **Part 65 (Axum Middleware)**: `CorsLayer`, `TraceLayer`, และหลักการ layer/middleware chain — บทนี้เพิ่ม
  `ServeDir`/`ServeFile` เข้าไปในสถาปัตยกรรมเดียวกัน (เป็น service/fallback ไม่ใช่ layer แต่ใช้หลักการ
  ประกอบ Router แบบเดียวกัน)
- **Part 70-71 (SQLx: PostgreSQL, Queries และ Migrations)**: `sqlx migrate run`, `PgPool`, offline mode
  ผ่าน `.sqlx/` cache และ `cargo sqlx prepare` — บทนี้ใช้ offline mode เป็นกลไกหลักในการ build backend
  ภายใน Docker container ที่ไม่มีทางเข้าถึงฐานข้อมูลจริงได้เลย
- **Part 82 (Message Queue/Distributed Systems หรือเนื้อหาที่พูดถึง expand-contract migration)**: หลักการ
  ที่ schema migration ต้องปลอดภัยเมื่อมีหลาย instance รันพร้อมกัน — บทนี้ใช้หลักการเดียวกันตัดสินใจเรื่อง
  "migration อัตโนมัติตอน container startup" vs "migration เป็น deploy step แยก"
- **Part 81 (Microservices พื้นฐาน)**: แนวคิด health check/readiness probe ที่แพลตฟอร์ม deploy จริง (load
  balancer, Kubernetes, ECS ฯลฯ) ใช้ตัดสินใจว่าจะส่ง traffic เข้า instance ไหน — บทนี้ต่อยอดแนวคิดนั้นเข้ากับ
  `/health` ของ Part 92 ตรง ๆ
- **Part 60 (Logging และ Tracing)**: `tracing`/`tracing-subscriber`, `EnvFilter` — บทนี้เพิ่มการสลับ
  formatter ระหว่าง pretty กับ JSON โดยไม่แก้ logic การ log เดิมของ Part 92 เลยแม้แต่จุดเดียว
- **Part 86-87 (WebAssembly และ wasm-bindgen)**: แนวคิดว่า `trunk build` compile Rust เป็น
  `wasm32-unknown-unknown` แล้วสร้างไฟล์ `.wasm` + JS glue code — บทนี้อธิบาย build output ของ production
  build โดยต่อยอดความเข้าใจนั้นตรง ๆ ไม่อธิบาย WASM ใหม่จากศูนย์

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้เป็นภาคปิดของ capstone ที่ Part 92-93 เริ่มไว้ — ทุกโค้ด ทุกคำสั่ง ทุกผลลัพธ์ในบทนี้**มาจากการคัดลอกโค้ด
จริงของ Part 92 (backend) และ Part 93 (frontend) มาประกอบเป็นโปรเจกต์เดียว รันจริงบนเครื่อง แล้ว capture
ผลลัพธ์มาทั้งหมด** ไม่มีการแต่งขึ้นเองแม้แต่จุดเดียว:

- PostgreSQL 16 จริงที่ `127.0.0.1:5432` (database แยกชื่อสำหรับตรวจสอบบทนี้โดยเฉพาะ ไม่ปนกับ database ของ
  Part 92/93)
- backend build จริงด้วย `cargo build --release` (ทั้งแบบรันตรงบนเครื่องและแบบ build ภายใน container)
- frontend build จริงด้วย `trunk build --release` เวอร์ชัน `0.21.14` (เวอร์ชันเดียวกับที่ Part 93 ใช้ทดสอบ)
- รัน combined server (backend + frontend ที่ build แล้ว) จริงที่พอร์ตเดียว แล้วยิง `curl` จริงกว่า 20 คำสั่ง
  และเปิด **headless Chromium จริงผ่าน Playwright** ขับ flow เต็มรูปแบบ (สมัคร → login → ดูรายการ → ยืม →
  คืน → deep-link → hard reload → logout) — ผลลัพธ์ทุกอันในบทนี้คัดลอกมาจาก terminal จริง
- ตรวจสอบ Docker จริงในสภาพแวดล้อมที่ใช้เขียนบทนี้ (`docker --version`/`docker ps`) แล้ว build/run image
  จริงถ้า Docker ใช้งานได้ — **หัวข้อ 94.9 จะบอกตรง ๆ ว่าอะไรที่ verify ได้จริงกับ Docker และอะไรที่ verify
  ผ่านการรัน binary ตรง ๆ แทน** เพราะสภาพแวดล้อมแบบ sandbox ที่ใช้พัฒนาหลักสูตรมีข้อจำกัดด้าน network บางอย่าง
  ที่ผู้อ่านในสภาพแวดล้อมของตัวเองอาจไม่เจอเลย
- โปรเจกต์ scratch ทั้งหมดที่ใช้ทดสอบถูกลบทิ้งหลังตรวจสอบเสร็จ (ไม่กระทบไฟล์ใด ๆ ในหลักสูตร) แต่**โค้ดทุกไฟล์
  ในบทนี้คือโค้ดตัวจริงที่ compile ผ่านและรันได้ 100%** ผู้อ่านสามารถคัดลอกไปสร้างโปรเจกต์เดียวกันขึ้นมาใหม่ได้
  ทันที (คู่กับ backend จาก Part 92 และ frontend จาก Part 93)

## เนื้อหา

### 94.1 ทวนสถาปัตยกรรมจาก Part 92-93 และช่องว่างที่บทนี้ต้องปิด

ก่อนเริ่ม ต้องเห็นภาพรวมให้ชัดว่าตอนนี้เรามีอะไรอยู่แล้ว และอะไรที่ยัง "ไม่ใช่ของจริง" สำหรับการใช้งานจริง:

**สิ่งที่มีอยู่แล้วจาก Part 92-93**:

1. **backend** (`library_api`) — Axum project ที่รันด้วย `cargo run` ที่ `http://127.0.0.1:8092` ต่อกับ
   PostgreSQL จริง มี 10 endpoint ตาม contract ของ Part 92 หัวข้อ 92.1, มี JWT auth, มี Swagger UI, มี
   integration test 5 ตัวที่ผ่านหมด
2. **frontend** (`library_frontend`) — Leptos CSR project ที่รันด้วย `trunk serve --port 5173` ที่
   `http://127.0.0.1:5173` เรียก backend ผ่าน `gloo-net` ไปที่ URL เต็ม `http://127.0.0.1:8092/...` ที่ฝัง
   ไว้เป็นค่าคงที่ (`API_BASE`) ในโค้ด
3. **CORS bridge** — `CorsLayer` ของ Part 92 อนุญาต origin `http://127.0.0.1:5173` โดยเฉพาะ (ตั้งผ่าน
   `FRONTEND_ORIGIN`) เพื่อให้ browser ยอมให้ WASM จาก origin `5173` เรียก API ที่ origin `8092` ได้

สถาปัตยกรรมนี้ **ใช้งานได้จริง 100% สำหรับการพัฒนา** — สมัคร/login/ยืม/คืนทำงานได้ครบตามที่ Part 93 หัวข้อ
93.11 พิสูจน์ไว้แล้วด้วย headless Chromium — แต่มันมีสมมติฐานสามข้อที่ **ใช้ไม่ได้เลยในสถานการณ์ deploy จริง**:

- สมมติฐานที่ 1: มีสอง process รันอยู่เสมอ คนละพอร์ต บนเครื่องเดียวกัน — ในโลก production คุณต้อง**ตัดสินใจ**
  ว่าจะรันแบบนี้ต่อไปจริง ๆ (สอง service, อาจอยู่คนละเครื่อง/คนละแพลตฟอร์มด้วยซ้ำ) หรือจะรวมเป็น process เดียว
- สมมติฐานที่ 2: `FRONTEND_ORIGIN`/`API_BASE` เป็นค่าคงที่ที่รู้ล่วงหน้าตอน dev (`127.0.0.1` กับพอร์ตตายตัว) —
  ใน production คุณไม่รู้ domain จริงจนกว่าจะ deploy ทำให้ค่าพวกนี้ต้อง**กำหนดได้ตอน build/run** ไม่ใช่ hardcode
- สมมติฐานที่ 3: ทั้งสอง process รันด้วย `cargo run`/`trunk serve` (dev tool ที่ compile แบบ unoptimized และ
  serve ไฟล์แบบไม่ผ่านการ build จริง) — production ต้องใช้ **release build** ของทั้งสองฝั่ง และต้องมีวิธี
  "แพ็ก" มันให้ deploy ได้ (binary + ไฟล์ static ที่ build แล้ว หรือ container image)

**เป้าหมายของบทนี้คือปิดช่องว่างทั้งสามข้อนี้** ให้ได้แอปเดียวที่ตอบคำถามได้ครบว่า: build อย่างไร, serve
อย่างไร (หัวข้อเดียวหรือแยกกัน), config อย่างไรให้ปลอดภัย, แพ็กเป็น container อย่างไร, migration ทำตอนไหน,
รู้ได้อย่างไรว่า instance ที่กำลังรันอยู่ "พร้อมใช้งานจริง" ไม่ใช่แค่ "process ไม่ตาย" และ log ออกมาในรูปแบบ
ที่ระบบ log aggregation ของจริงอ่านได้ — ทั้งหมดนี้คือสิ่งที่แยกระหว่าง "โค้ดที่รันได้ในเครื่องผู้เขียน" กับ
"ระบบที่ deploy ได้จริง"

**ภาพก่อน/หลัง** (ASCII diagram สรุปการเปลี่ยนแปลงที่บทนี้ทำ — อ่านจากบนลงล่าง):

```
ก่อน (Part 92-93, dev):                      หลัง (Part 94, production):
┌─────────────────────────┐                  ┌─────────────────────────────┐
│ browser                 │                  │ browser                     │
│  http://127.0.0.1:5173  │                  │  https://app.example.com    │
└───────────┬─────────────┘                  └──────────────┬──────────────┘
            │ cross-origin (CORS)                            │ same-origin
            │ fetch("http://127.0.0.1:8092/…")                │ fetch("/api/v1/…")  (relative)
            ▼                                                 ▼
┌─────────────────────────┐                  ┌─────────────────────────────┐
│ trunk serve  :5173      │                  │  library_api process        │
│ (dev server, unoptimized│                  │  ┌────────────────────────┐ │
│  frontend)               │                  │  │ Router (Part 92)       │ │
└──────────────────────────┘                  │  │  /health  /api/v1/…    │ │
┌─────────────────────────┐                  │  ├────────────────────────┤ │
│ cargo run   :8092       │                  │  │ ServeDir fallback      │ │
│ (dev backend, CorsLayer  │                  │  │  index.html/*.js/*.wasm│ │
│  allows :5173)           │                  │  │  (trunk build --release)│
└──────────────┬───────────┘                  │  └────────────────────────┘ │
               │                               └──────────────┬──────────────┘
               ▼                                              ▼
        PostgreSQL (dev)                               PostgreSQL (production,
                                                         DATABASE_URL จาก Secret)
```

สังเกตว่าฝั่งขวา (หลัง) มี**หนึ่ง process เดียว** ที่ทำทั้งสองหน้าที่ (serve API + serve static file) และ
browser ไม่มีการข้าม origin เลย — นี่คือภาพรวมของทุกหัวข้อที่จะตามมาในบทนี้

### 94.2 Production Build: `trunk build --release` vs `cargo build --release`

ก่อนตัดสินใจเรื่อง serving strategy ต้องรู้ก่อนว่า production build ของแต่ละฝั่งให้ผลลัพธ์อะไรออกมา — เพราะ
คำตอบของหัวข้อ 94.3 (จะ serve ยังไง) ขึ้นอยู่กับว่า artifact ที่ได้คือ "ไฟล์ static ชุดหนึ่ง" (frontend) หรือ
"binary ที่รันเป็น process" (backend) ซึ่งเป็นคนละประเภทกันโดยสิ้นเชิง

**Backend — `cargo build --release`** (คัดลอกโค้ดของ Part 92 มาทั้งหมด ไม่มีการแก้ไขในหัวข้อนี้):

```bash
$ cd library_api
$ cargo build --release
   Compiling library_api v0.1.0 (/.../library_api)
    Finished `release` profile [optimized] target(s) in 1m 30s

$ du -h target/release/library_api
14M     target/release/library_api

$ du -h target/debug/library_api   # เทียบกับ dev build เดิมของ Part 92
119M    target/debug/library_api
```

**ต่างจาก dev build ของ Part 92 อย่างไร**: `cargo build` (ธรรมดา ไม่มี `--release`) ที่ Part 92 ใช้ตลอดบท
สร้าง binary ขนาด **119MB** พร้อม debug symbol เต็มรูปแบบและไม่มีการ optimize ใด ๆ — เหมาะกับตอนพัฒนาเพราะ
compile เร็วกว่ามาก (ทำให้ inner dev loop เร็ว) แต่ **ไม่ควร deploy** เพราะช้ากว่าและใหญ่กว่าโดยไม่มีประโยชน์
อะไรกับผู้ใช้จริงเลย — `cargo build --release` เปิด optimization level สูง (`opt-level = 3` เป็น default ของ
profile `release`), ตัด debug info ส่วนใหญ่ทิ้ง, และทำ inlining/dead-code-elimination ได้ดีกว่า ทำให้ได้
binary ที่**เล็กกว่าเกือบ 8.5 เท่า** (14MB จาก 119MB) และเร็วกว่าตอนรันจริงอย่างชัดเจน แลกมาด้วยเวลา compile
ที่นานกว่า (~1.5 นาที เทียบกับไม่กี่วินาทีของ dev build) — ข้อแลกเปลี่ยนนี้สมเหตุสมผลเสมอสำหรับ deploy เพราะ
compile ทำครั้งเดียวตอน build แต่ binary รันซ้ำ ๆ นับล้านครั้งตลอดอายุของ deployment

**Frontend — `trunk build --release`** (คัดลอกโค้ดของ Part 93 มาทั้งหมด มีแก้ไขจุดเดียวคือ `API_BASE` ซึ่ง
อธิบายเหตุผลในหัวข้อ 94.4):

```bash
$ cd library_frontend
$ trunk build --release
    Finished `release` profile [optimized] target(s) in 1m 02s
2026-09-27T04:23:10.793729Z  INFO applying new distribution
2026-09-27T04:23:10.794167Z  INFO success

$ ls -la dist/
total 1508
-rw-r--r-- 1 root root     844 index.html
-rw-r--r-- 1 root root   48026 library_frontend-970a71669f035894.js
-rw-r--r-- 1 root root 1481581 library_frontend-970a71669f035894_bg.wasm

$ du -sh dist/
1.5M    dist
```

**สามไฟล์ที่ `trunk build --release` สร้างออกมา** (ต่อยอดแนวคิด WASM build ของ Part 86-87 ตรง ๆ):

- **`index.html`** — ไฟล์ HTML shell ที่ `trunk` แก้ให้อัตโนมัติจาก `index.html` ต้นฉบับของ Part 93 หัวข้อ
  93.2 (เติม `<script type="module">` ที่โหลด `.js`/`.wasm` พร้อม hash ในชื่อไฟล์กันปัญหา browser cache
  ค้างเวอร์ชันเก่า และเติม `<link rel="modulepreload">`/`<link rel="preload">` พร้อม
  `integrity="sha384-..."` ให้อัตโนมัติเพื่อความปลอดภัยและ performance)
- **`library_frontend-<hash>.js`** (48KB) — JS "glue code" ที่ `wasm-bindgen` สร้างให้ (กลไกเดียวกับที่ Part
  87 อธิบายไว้ตอน `wasm-bindgen` สร้าง binding ระหว่าง JS กับ Rust) มีหน้าที่โหลด `.wasm` module, ผูก import/
  export ระหว่าง JS กับ WASM linear memory ให้ถูกต้อง
- **`library_frontend-<hash>_bg.wasm`** (1.5MB) — ตัวโปรแกรม Leptos ทั้งแอปที่ compile จาก Rust เป็น
  `wasm32-unknown-unknown` (target เดียวกับที่ Part 93 หัวข้อ 93.2 เพิ่มด้วย `rustup target add
  wasm32-unknown-unknown`) แล้วผ่านการ optimize ของ `trunk build --release` (เปิด `wasm-opt` ถ้ามีติดตั้งไว้)
  — ไฟล์นี้คือ "แอปทั้งตัว" ที่รันในเบราว์เซอร์ล้วน ๆ ไม่มีส่วนใดรันบนเซิร์ฟเวอร์เลย ตรงกับที่ Part 93 หัวข้อ
  93.1 ยืนยันไว้ว่าเป็น CSR แบบ `csr` feature เดียว ไม่มี `ssr`/`hydrate`

เทียบกับ dev mode ของ Part 93 ที่ใช้ `trunk serve` (ไม่มี `--release`): dev mode สร้างไฟล์ในหน่วยความจำ/
`dist/` แบบ unoptimized (compile เร็วเพื่อ hot-reload) และ **serve ไฟล์เหล่านั้นเองผ่าน dev server ในตัว**
(`trunk serve` เป็นทั้ง build tool และ web server ในเครื่องมือเดียว) — ในขณะที่ `trunk build --release` **ทำ
แค่ build อย่างเดียว ไม่ serve อะไรเลย** มันแค่ทิ้งไฟล์สามไฟล์ข้างบนไว้ใน `dist/` แล้วจบ หน้าที่ "เอาไฟล์พวกนี้
ไป serve ที่ไหน" เป็นของหัวข้อถัดไป — นี่คือจุดที่คำถาม "ใครจะ serve ไฟล์พวกนี้" ต้องมีคำตอบชัดเจน

**ตัวเลขที่พิสูจน์ความต่างจริงระหว่าง dev build กับ release build ของ WASM** — รัน `trunk build` (ไม่มี
`--release`) กับโค้ดเดียวกันเป๊ะ แล้วเทียบไฟล์ `.wasm` ที่ได้:

```bash
$ trunk build   # dev build — ไม่มี --release
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 55.33s

$ du -h dist/*.wasm dist/*.js
5.8M    dist/library_frontend-8e948b3398c948c1_bg.wasm
52K     dist/library_frontend-8e948b3398c948c1.js
```

เทียบกับ **1.5MB** ของ release build ที่หัวข้อนี้แสดงไว้ข้างบน — dev build ของไฟล์ `.wasm` **ใหญ่กว่าเกือบ 4
เท่า** (5.8MB จาก 1.5MB) ด้วยเหตุผลเดียวกับที่หัวข้อนี้อธิบายไว้ตอนเทียบ backend binary (dev/debug build เก็บ
debug symbol เต็มรูปแบบและไม่ optimize) บวกกับอีกปัจจัยหนึ่งที่เฉพาะกับ WASM: `trunk build --release` เรียก
`wasm-opt` (เครื่องมือจาก Binaryen ที่ทำ optimization ระดับ WASM bytecode โดยเฉพาะ ต่างจาก LLVM optimization
ที่ `rustc --release` ทำอยู่แล้วตอน compile เป็น `.wasm` — `wasm-opt` ทำงาน**ต่อจาก**นั้นอีกชั้นที่ระดับ
bytecode ของ WASM ตรง ๆ) ให้อัตโนมัติถ้าเครื่องมีติดตั้งไว้ ซึ่งลดขนาดไฟล์ `.wasm` ลงไปอีกอย่างมีนัยสำคัญ —
สำหรับผู้ใช้จริงที่ต้องดาวน์โหลดไฟล์นี้ผ่านเครือข่ายก่อนแอปจะเริ่มทำงานได้เลย ความต่างระหว่าง 5.8MB กับ
1.5MB มีผลตรงต่อเวลาที่ผู้ใช้ต้องรอ (ยิ่งชัดเจนมากบนเครือข่ายมือถือที่ความเร็วจำกัด) — นี่คือเหตุผลที่ **ห้าม
deploy dev build ของ frontend ไปยัง production เด็ดขาด** เหมือนกับที่ห้าม deploy dev build ของ backend

**เรื่อง build cache ที่ Part 96 จะลงรายละเอียดเพิ่ม**: ตัวเลขเวลา compile ที่แสดงไว้ข้างบนทั้งหมด (backend
~1.5 นาที, frontend ~1 นาที) คือเวลาของ **clean build** (ไม่มี cache จากการ build ครั้งก่อนเหลืออยู่เลย) —
ในทางปฏิบัติ ทั้ง `cargo build` และ `trunk build` มีการ cache ผลลัพธ์ของ dependency ที่ไม่เปลี่ยนแปลงไว้ที่
`target/` (backend) และ `~/.cargo/registry` (แคชของ crates ที่ดาวน์โหลดแล้ว) ทำให้ build ครั้งถัดไปที่แก้แค่
โค้ดของเราเอง (ไม่แก้ `Cargo.toml`/dependency) **เร็วขึ้นมาก** เพราะไม่ต้อง compile dependency ทั้งหมดใหม่ —
Dockerfile ของหัวข้อ 94.5 ที่แยก `COPY Cargo.toml Cargo.lock` ออกจาก `COPY src` เป็นคนละคำสั่งก็เพื่อประโยชน์
ตรงนี้เช่นกัน (Docker cache แต่ละ layer แยกกัน — ถ้า `Cargo.toml`/`Cargo.lock` ไม่เปลี่ยน Docker จะข้าม
`cargo build`/`RUN` ที่ตามมาและใช้ cache ของ layer เดิมได้ทันที แม้ `src/` จะเปลี่ยนก็ตาม ถ้าจัดลำดับ `COPY`
ให้ไฟล์ที่เปลี่ยนบ่อยกว่าอยู่**หลัง**ไฟล์ที่เปลี่ยนน้อยกว่าเสมอ) — Part 96 จะอธิบายกลไก layer caching นี้ให้
ลึกกว่านี้อีกมาก บทนี้แค่ปูพื้นให้เห็นว่าทำไมลำดับ `COPY` ใน Dockerfile ของหัวข้อ 94.5 ถึงเขียนแบบนั้น

**ทำไม `library_api` กับ `library_frontend` ยังเป็นสอง Cargo project แยกกัน ไม่รวมเป็น Cargo workspace เดียว
(ต่อยอด Part 17 เรื่อง workspace)**: คำถามที่น่าคิดตอนเห็นว่าทั้งสองโปรเจกต์ต้อง build พร้อมกันเสมอในหัวข้อ
94.5 — ทำไมไม่รวมเป็น workspace เดียวที่มี `[workspace] members = ["library_api", "library_frontend"]`
เหมือนที่ Part 17 สอนไว้เรื่องจัดการหลาย crate ในโปรเจกต์เดียว? คำตอบคือ**ทั้งสอง crate compile ไปเป็นคนละ
compilation target โดยพื้นฐาน** — `library_api` compile เป็น native binary ของแพลตฟอร์มที่รัน (เช่น
`x86_64-unknown-linux-gnu`) ในขณะที่ `library_frontend` compile เป็น `wasm32-unknown-unknown` (ตามที่ Part
93 หัวข้อ 93.2 ตั้งไว้) — Cargo workspace **รองรับ** การมี member ที่ target ต่างกันได้จริงในทางเทคนิค (แค่
สั่ง `cargo build --target wasm32-unknown-unknown -p library_frontend` แยกจาก `cargo build -p
library_api` ตามปกติ) แต่ประโยชน์หลักของ workspace (แชร์ `Cargo.lock` เดียวกัน, แชร์ dependency version กัน
ทั้งโปรเจกต์) **ไม่มีความหมายมากนักในกรณีนี้** เพราะทั้งสอง crate ไม่ได้แชร์ dependency ที่ทับซ้อนกันเลย
(`library_api` ใช้ `axum`/`sqlx`/`tokio`, `library_frontend` ใช้ `leptos`/`gloo-net`/`wasm-bindgen` — ไม่มี
crate ไหนที่ทั้งสองฝั่งต้องใช้ร่วมกันจริง ๆ) — การแยกเป็นสองโปรเจกต์อิสระ (ตามที่ Part 92/93 ทำไว้ตั้งแต่ต้น)
จึงเรียบง่ายกว่าโดยไม่เสียประโยชน์อะไรไป และยังสะท้อนความเป็นจริงของทีม frontend/backend ที่แยกกันตามที่ Part
93 หัวข้อ 93.1 อธิบายไว้ (repo แยกกันได้ในโลกจริงด้วยซ้ำ ไม่ใช่แค่ folder แยกกันในโปรเจกต์เดียว) — Dockerfile
ของหัวข้อ 94.5 ที่มีสอง build stage แยกกันจึงเป็นทางออกที่ตรงกับความเป็นจริงของโครงสร้างโปรเจกต์มากกว่าการฝืน
รวมเป็น workspace เดียว

### 94.3 ตัดสินใจ Serving Strategy: Backend Serve Frontend เอง vs แยกเป็นสองบริการ

นี่คือการตัดสินใจเชิงสถาปัตยกรรมที่สำคัญที่สุดของบทนี้ — มีสองทางเลือกที่ใช้กันจริงในโลกทำงาน ต้องเข้าใจ
ข้อแลกเปลี่ยนของทั้งสองทางก่อนเลือก

#### ทางเลือกที่ 1: แยกเป็นสองบริการอิสระ (backend เป็น API-only, frontend อยู่บน CDN/static host)

frontend (ไฟล์ static สามไฟล์จากหัวข้อ 94.2) ถูก deploy ไปยัง static hosting/CDN (เช่น Cloudflare Pages,
Netlify, S3+CloudFront) ส่วน backend เป็น API-only service ที่ deploy แยกที่ (เช่น container บน ECS/Cloud
Run) — ทั้งสองคุยกันผ่าน HTTP ข้าม origin จริง เหมือนที่ Part 92-93 ทำตอน dev เพียงแต่เปลี่ยนจาก
`127.0.0.1` เป็น domain จริง

**ข้อดี**:

- CDN ที่ serve ไฟล์ static เร็วกว่า process ของเราเองมาก (edge caching ทั่วโลก) — ผู้ใช้ที่อยู่ไกล data
  center ของ backend ได้ประโยชน์เต็มที่จาก frontend ที่โหลดเร็ว แม้ backend จะอยู่ data center เดียว
- scale แยกกันได้ตามภาระงานจริง — frontend (static file) ไม่ต้องคิดเรื่อง scale เลย (CDN จัดการให้) ในขณะที่
  backend (ที่มีภาระงาน CPU/DB จริง) scale ตามที่จำเป็นโดยไม่กระทบ frontend
- ทีมสองทีม (frontend/backend) deploy อิสระจากกันได้จริง ตรงกับสถานการณ์ที่ Part 93 หัวข้อ 93.1 อธิบายไว้ว่า
  เป็นเหตุผลที่ทั้ง capstone นี้ไม่ใช้ Leptos server function

**ข้อเสีย**:

- **ต้องจัดการ CORS ตลอดไป ไม่ใช่แค่ตอน dev** — ทุก request จริงข้าม origin เสมอ ต้องดูแล `CorsLayer` ให้ตรง
  กับ domain จริงของ frontend ทุกครั้งที่เปลี่ยน (รวม preflight `OPTIONS` ทุกจุด)
- ต้องดูแล deployment pipeline สองชุดที่แยกกันจริง (build/deploy ของ frontend กับ backend เป็นคนละ process
  ทั้งคู่ แม้จะมาจาก repo เดียวกันก็ตาม)
- ความหน่วง (latency) เพิ่มขึ้นเล็กน้อยจาก DNS lookup/TLS handshake ที่ต้องทำสอง origin แทนหนึ่ง

#### ทางเลือกที่ 2 (ที่บทนี้ implement จริง): Backend Serve Frontend เอง — Single Origin

Axum server ตัวเดียวกันที่ให้บริการ REST API (Part 92) **serve ไฟล์ static ของ frontend ด้วย** ผ่าน
`tower_http::services::ServeDir` (ต่อยอด Part 65 เรื่อง middleware/service ของ `tower`) — ทุก request ที่
ไม่ตรงกับ route ของ API (`/api/v1/...`, `/health`) จะถูกส่งไปหาไฟล์ static แทน กลายเป็น**หนึ่ง process, หนึ่ง
port, หนึ่ง origin**

**ข้อดี**:

- **ไม่มี cross-origin request เลย** — เพราะ browser โหลดหน้าเว็บจาก origin เดียวกับที่ API อยู่ ปิดปัญหา
  CORS ทั้งหมดในโปรดักชันไปเลย (ไม่ต้องดูแล `CorsLayer` อีกต่อไปในสถาปัตยกรรมนี้)
- **deploy เป็นหนึ่งหน่วยเดียว** — container/binary ตัวเดียว ไม่ต้องประสาน deployment pipeline สองชุด ง่ายต่อ
  การจำลอง ("รัน 1 คำสั่ง ได้แอปทั้งตัว") เหมาะกับทีมเล็ก/โปรเจกต์ที่ยังไม่ต้อง scale frontend/backend แยกกัน
- ปิดวงจรของสิ่งที่ Part 93 ทำ CORS ไว้ตอน dev พอดี — ใน production **ไม่มี CORS ให้ต้องตั้งเลย** เพราะไม่มี
  cross-origin request เกิดขึ้นจริง

**ข้อเสีย**:

- เสีย benefit ของ CDN edge caching สำหรับไฟล์ static (เว้นแต่จะตั้ง reverse proxy/CDN คลุมหน้า backend
  ทั้งก้อนอีกชั้น ซึ่งเป็นไปได้แต่ซับซ้อนกว่า)
- scale frontend/backend แยกกันไม่ได้ตรง ๆ (แม้ในทางปฏิบัติ ไฟล์ static แทบไม่กิน CPU/memory เพิ่มเลย เมื่อ
  เทียบกับ endpoint ที่ทำ database query จริง จึงมักไม่ใช่ปัญหาจริงสำหรับแอประดับนี้)
- ทีม frontend/backend deploy แยกกันไม่ได้ทันทีที่แต่ละฝั่งแก้โค้ด (ต้อง build ทั้งคู่ใหม่เพื่อ deploy หนึ่ง
  image) — ถ้าทีมโตขึ้นมากในอนาคต อาจต้องย้ายไปทางเลือกที่ 1

**บทนี้เลือกทางเลือกที่ 2 เป็นแนวทางหลักที่ implement จริง** ด้วยสามเหตุผล: (1) มันปิดวงจร CORS ที่ Part 92-93
ตั้งใจทำไว้ตอน dev พอดี ทำให้เห็นภาพครบว่า dev/production ต่างกันตรงไหนจริง ๆ (2) มัน deploy ง่ายกว่าสำหรับ
capstone ระดับนี้ (ไม่ต้องมี CDN account/CI pipeline สองชุดเพื่อสอน concept) และ (3) มันคือรูปแบบที่พบบ่อยมาก
ในโปรเจกต์ Rust full-stack จริง (ทั้ง Leptos SSR ของ Part 89 และหลายเฟรมเวิร์ก Rust web อื่น ๆ ก็ทำแบบนี้เป็น
ค่าเริ่มต้น) — ทางเลือกที่ 1 ยังคงเป็นทางเลือกที่ถูกต้องสำหรับทีม/สเกลที่ใหญ่ขึ้น และโค้ดของบทนี้ก็ยัง
**รองรับมันได้เหมือนเดิม** (แค่ไม่ตั้ง `STATIC_DIR` แล้วตั้ง `FRONTEND_ORIGIN` กลับมาแทน ตามที่หัวข้อ 94.4
อธิบาย)

#### Implement จริง: เพิ่ม `ServeDir` เข้าไปใน `build_app` ของ Part 92

จุดที่ต้องแก้คือ `src/lib.rs` ของ Part 92 (ฟังก์ชัน `build_app`) — เดิม Part 92 หัวข้อ 92.10 เขียนไว้แบบนี้
(สรุปย่อ ดูฉบับเต็มที่ Part 92):

```rust
// เดิม (Part 92 หัวข้อ 92.10) — ไม่มี static file serving เลย, CORS เปิดเสมอ
pub fn build_app(state: AppState, frontend_origin: &str) -> Router {
    let cors = CorsLayer::new()
        .allow_origin(frontend_origin.parse::<HeaderValue>().expect("..."))
        .allow_methods([Method::GET, Method::POST, Method::OPTIONS])
        .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION]);

    routes::build_router()
        .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", docs::ApiDoc::openapi()))
        .layer(cors)
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}
```

ฉบับใหม่ของบทนี้ **ไม่ลบของเดิมทิ้ง** แต่เพิ่มความสามารถเป็นทางเลือก (`Option`) ทั้งสองแกน — เพื่อให้โค้ด
เดียวกันรองรับได้ทั้งสองสถาปัตยกรรมของหัวข้อก่อนหน้า:

```rust
// src/lib.rs — ฉบับ Part 94 (แก้จาก Part 92 หัวข้อ 92.10)
pub mod auth;
pub mod docs;
pub mod error;
pub mod models;
pub mod routes;
pub mod state;

use std::path::Path;

use axum::http::{header, HeaderValue, Method};
use axum::Router;
use tower_http::cors::CorsLayer;
use tower_http::services::{ServeDir, ServeFile};
use tower_http::trace::TraceLayer;
use utoipa::OpenApi;
use utoipa_swagger_ui::SwaggerUi;

use state::AppState;

/// สร้าง Router เต็มรูปแบบของทั้งแอป — ต่อยอด Part 92 หัวข้อ 92.10 โดยเพิ่มความสามารถ "serve
/// ไฟล์ static ของ frontend" เป็นทางเลือก (`static_dir`) สำหรับสถาปัตยกรรม "single origin" ของ
/// Part 94 หัวข้อ 94.3 โดยไม่ทิ้งความสามารถเดิม (สถาปัตยกรรมแยกสองบริการ + CORS ของ Part 92/93)
///
/// - `frontend_origin: Some(origin)` เปิด CORS ให้ origin นั้น (โหมด dev ตาม Part 92/93, หรือ
///   สถาปัตยกรรม "แยกสองบริการ" ของหัวข้อ 94.3 ตอน deploy จริง)
/// - `frontend_origin: None` ไม่เปิด CORS เลย — ใช้ตอนไม่มี cross-origin request เกิดขึ้นจริง
///   (สถาปัตยกรรม "single origin" ที่ backend serve frontend เอง)
/// - `static_dir: Some(path)` เพิ่ม fallback service ที่ serve ไฟล์ static จาก path นั้น พร้อม
///   SPA fallback (path ที่ไม่ตรงไฟล์จริงบน disk -> คืน index.html แทน 404 ให้ leptos_router
///   ฝั่ง client ตัดสินใจ render เอง — ดูกับดักที่พบบ่อยข้อ 2 ท้ายบทเรื่องสถานะ HTTP ของ fallback นี้)
pub fn build_app(
    state: AppState,
    frontend_origin: Option<&str>,
    static_dir: Option<&Path>,
) -> Router {
    let mut app: Router<AppState> = routes::build_router()
        .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", docs::ApiDoc::openapi()));

    if let Some(dir) = static_dir {
        let index_path = dir.join("index.html");
        // ServeDir หาไฟล์จริงก่อนเสมอ (เช่น *.js, *.wasm, index.html เอง) — ถ้าไม่เจอไฟล์ที่ path
        // นั้นจริง ๆ (เช่น /books/3, /my-borrowed ที่เป็น route ของ leptos_router ไม่ใช่ไฟล์บน disk)
        // จะเรียก fallback (ServeFile ของ index.html) แทน "ด้วย status ของ ServeFile เอง" (200
        // เพราะไฟล์นั้นมีอยู่จริง) — ต่างจาก `.not_found_service()` ที่บังคับ status เป็น 404 เสมอ
        // (ใช้ถูกจุดสำหรับหน้า "404 not found" ที่ตั้งใจให้เป็น 404 จริง แต่ผิดจุดสำหรับ SPA fallback)
        let serve_dir = ServeDir::new(dir).fallback(ServeFile::new(index_path));
        app = app.fallback_service(serve_dir);
    }

    let mut router: Router = app.layer(TraceLayer::new_for_http()).with_state(state);

    if let Some(origin) = frontend_origin {
        let cors = CorsLayer::new()
            .allow_origin(
                origin
                    .parse::<HeaderValue>()
                    .expect("frontend_origin ต้องเป็น URL ที่ valid"),
            )
            .allow_methods([Method::GET, Method::POST, Method::OPTIONS])
            .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION]);
        router = router.layer(cors);
    }

    router
}

/// wrapper ที่คง signature เดิมของ Part 92 หัวข้อ 92.10 ไว้เป๊ะ (`(state, frontend_origin: &str) ->
/// Router`) — เพราะ `build_app` ของบทนี้เปลี่ยน signature จริง (พารามิเตอร์ที่สองกลายเป็น `Option<&str>`
/// และมีพารามิเตอร์ที่สามเพิ่มมา) ซึ่งเป็น breaking change ที่ตั้งใจทำ (จำเป็นสำหรับความสามารถใหม่ของ
/// หัวข้อ 94.3) — `build_app_dev` มีไว้ให้ `tests/api_tests.rs` ของ Part 92 หัวข้อ 92.11 อ้างถึงแทนได้
pub fn build_app_dev(state: AppState, frontend_origin: &str) -> Router {
    build_app(state, Some(frontend_origin), None)
}
```

**ผลกระทบต่อ `tests/api_tests.rs` ของ Part 92 หัวข้อ 92.11 — ต้องแก้แค่บรรทัด `use` เดียว ไม่แก้ call site
ไหนเลย**: ไฟล์เทสเดิมมี `use library_api::{build_app, state::AppState};` แล้วเรียก `build_app(state,
FRONTEND_ORIGIN)` (สองพารามิเตอร์, ตัวที่สองเป็น `&str` ตรง ๆ) กระจายอยู่ในฟังก์ชัน helper `app(pool)` ที่
ทุกเทสทั้ง 5 ตัวเรียกใช้ — เปลี่ยนบรรทัด `use` ให้ alias ชื่อแทนที่จะเปลี่ยนทุกจุดที่เรียก:

```rust
// tests/api_tests.rs — เปลี่ยนแค่บรรทัดเดียว (import), ทุกจุดที่เรียก build_app(state, FRONTEND_ORIGIN)
// ด้านล่างในไฟล์นี้ไม่ต้องแก้อะไรเลยแม้แต่ตัวอักษรเดียว
use library_api::{build_app_dev as build_app, state::AppState};
```

พิสูจน์ด้วยการรันเทสจำลองที่เขียนเรียกแบบเดิมของ Part 92 เป๊ะ (สองพารามิเตอร์ ไม่มี `Option`/`None` ที่จุด
เรียกเลย):

```bash
$ cargo test --test compat_check
running 1 test
test part92_style_call_site_unchanged ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.24s
```

นี่คือตัวอย่างที่ดีของการใช้ Rust's module system (`as` ใน `use` statement, ต่อยอด Part 16) แก้ปัญหา
backward compatibility แบบที่กระทบโค้ดเรียกใช้น้อยที่สุด — เปลี่ยน "ชื่อที่มองเห็นจากนอก module" ได้โดยไม่
ต้องแก้ implementation หรือ call site ใดๆที่อยู่ห่างจากจุดเปลี่ยนแปลงจริง

**อธิบายจุดสำคัญที่สุด — `.fallback(ServeFile::new(index_path))` ไม่ใช่ `.not_found_service(...)`**: นี่คือ
กับดักจริงที่เจอระหว่างพัฒนาบทนี้ (รายละเอียดเต็มพร้อม `curl` transcript จริงอยู่ในกับดักที่พบบ่อยข้อ 2) —
`tower_http::services::ServeDir` มีสองเมธอดที่ดูคล้ายกันมากแต่ผลลัพธ์ต่างกันโดยสิ้นเชิง:

- `.not_found_service(fallback)` — เรียก `fallback` ก็จริง แต่**บังคับ status code ของ response เป็น
  `404`เสมอ** ไม่ว่า `fallback` เองจะตอบ status อะไรก็ตาม (เหมาะกับหน้า "404 ไม่พบหน้านี้" ที่ตั้งใจให้เป็น
  404 จริง — เอกสารของ `tower-http` เองยกตัวอย่างการใช้แบบนี้ไว้ด้วยซ้ำ แต่**ไม่เหมาะกับ SPA fallback**)
- `.fallback(fallback)` — เรียก `fallback` แล้ว**ปล่อยให้ status code เป็นไปตามที่ `fallback` ตอบเอง** — ถ้า
  `fallback` คือ `ServeFile::new(index_path)` และไฟล์นั้นมีอยู่จริง มันจะตอบ `200 OK` ตามปกติของการ serve
  ไฟล์ที่เจอจริง — **นี่คือพฤติกรรมที่ SPA ต้องการ**: browser navigate ไปยัง `/my-borrowed` ตรง ๆ (deep link)
  ต้องได้ `200` พร้อม `index.html` กลับมา ให้ `leptos_router` ฝั่ง client (ที่ยังไม่ทันโหลดขึ้นมาด้วยซ้ำตอน
  request แรกไปถึง server) ตัดสินใจ render หน้าที่ถูกต้องเอง

พิสูจน์ด้วย `curl` จริงกับ combined server ที่รันจริง (ตั้งค่า `STATIC_DIR` ชี้ไปที่ `dist/` ของ frontend
production build จากหัวข้อ 94.2):

```bash
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8094/
200
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8094/my-borrowed
200
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8094/books/3
200
$ curl -sS -o /dev/null -w "%{http_code} %{content_type}\n" \
    http://127.0.0.1:8094/library_frontend-970a71669f035894_bg.wasm
200 application/wasm
$ curl -sS -i http://127.0.0.1:8094/health | head -1
HTTP/1.1 200 OK
```

ทั้ง `/my-borrowed` และ `/books/3` (route ของ `leptos_router` ที่ไม่มีไฟล์จริงอยู่บน disk) ตอบ `200` พร้อม
`index.html` (ตรวจสอบ body ได้ด้วย `curl -i` — เนื้อหาคือ HTML shell เดียวกับ `/`) ในขณะที่ไฟล์ `.wasm` จริง
ตอบด้วย `content-type: application/wasm` ที่ถูกต้อง (ต่อยอด MIME type detection ของ `ServeDir` ที่ทำงานถูก
ต้องอยู่แล้วโดยไม่ต้องตั้งค่าเพิ่ม) และ `/health` ยังตอบจาก API handler จริง (ไม่ถูก `ServeDir` แย่งไป) เพราะ
`fallback_service` ถูกเรียก**เฉพาะตอนไม่มี route อื่นใน Router ตรงกับ path นั้นเลย** — Router ของ Part 92
ที่มี route `/health` ประกาศไว้อยู่ก่อนแล้วจะจับ request นั้นไปก่อนเสมอ ไม่ถึง fallback

พิสูจน์ว่า**ไม่มี cross-origin request เหลืออยู่เลย** — สังเกต response header ของ `/health` เทียบกับตอน
Part 92-93 รันแยก origin (ที่หัวข้อ 92.12/93.11 มี header `access-control-allow-origin` ติดมาด้วยทุกครั้ง):

```bash
$ curl -sS -i http://127.0.0.1:8094/health | grep -i "access-control"
# (ไม่มีผลลัพธ์เลย — ไม่มี access-control-* header ใด ๆ ทั้งสิ้น)
```

เพราะเราไม่ได้ส่ง `frontend_origin: Some(...)` ให้ `build_app` ในโหมด single-origin (ดูหัวข้อ 94.4) จึงไม่มี
`CorsLayer` ถูกเพิ่มเข้าไปใน Router เลย — สอดคล้องกับที่อธิบายไว้ว่า production ของสถาปัตยกรรมนี้**ไม่มี
cross-origin request ให้ CORS ต้องอนุญาต** เพราะ browser ขอทุกอย่าง (HTML, JS, WASM, และ API response) จาก
origin เดียวกันทั้งหมด (`http://127.0.0.1:8094` ในตัวอย่างนี้ หรือ `https://yourapp.example.com` จริงตอน
deploy)

**Swagger UI ของ Part 92 หัวข้อ 92.10 ก็ยังใช้งานได้ปกติผ่าน origin เดียวกันนี้เช่นกัน** — เพราะ
`SwaggerUi::new("/swagger-ui")` ถูก `.merge(...)` เข้า Router **ก่อน** `.fallback_service(serve_dir)` ถูก
เพิ่มเข้ามา (ตามลำดับในโค้ดของหัวข้อนี้) route `/swagger-ui`/`/api-docs/openapi.json` จึงยังจับ request ของ
มันเองได้ก่อนเสมอ ไม่ถูก `ServeDir` แย่งไปเป็น SPA fallback:

```bash
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8094/swagger-ui/
200
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8094/api-docs/openapi.json
200
$ curl -sS http://127.0.0.1:8094/api-docs/openapi.json | head -c 160
{"openapi":"3.1.0","info":{"title":"library_api","description":"","license":{"name":""},"version":"0.1.0"},"paths":{"/api/v1/auth/login":{"post": ...
```

นี่คือประโยชน์แถมของสถาปัตยกรรม single-origin: นักพัฒนา/QA ที่ต้องทดสอบ API ด้วย Swagger UI ตอน production
(หรือ staging) เปิดหน้าเว็บเดียวกับที่ผู้ใช้จริงเห็น แล้วต่อท้าย `/swagger-ui/` ได้ทันที ไม่ต้องรู้ URL ของ
"บริการ backend แยก" อีกต่อไป — ต่างจากสถาปัตยกรรมแยกสองบริการที่ Swagger UI จะอยู่คนละ domain กับหน้าเว็บ
ที่ผู้ใช้จริงเห็นเสมอ

### 94.4 Environment Configuration สำหรับ Production

Part 92 หัวข้อ 92.12 ออกแบบ `main.rs` ให้อ่านค่า config จาก environment variable พร้อม **default ที่สะดวก
สำหรับ dev** ไว้แล้ว (`DATABASE_URL`, `JWT_SECRET`, `FRONTEND_ORIGIN`, `BIND_ADDR`) — นี่เป็นรากฐานที่ดีอยู่
แล้วตามหลัก "ไม่มี secret ในซอร์สโค้ด" ที่ใช้ตลอดหลักสูตร (Part 74/76) แต่บทนี้ต้องเพิ่มอีกสามอย่างเพื่อให้
รองรับทั้งสองสถาปัตยกรรมของหัวข้อ 94.3 และควบคุมพฤติกรรม migration/logging ที่หัวข้อ 94.6/94.8 จะพูดถึง

**ตัวแปร environment ทั้งหมดของบทนี้** (ตัวเก่าสี่ตัวจาก Part 92 + ตัวใหม่สามตัว):

| ตัวแปร | ค่า default | ความหมาย | มาจาก |
|---|---|---|---|
| `DATABASE_URL` | dev connection string | connection string ของ PostgreSQL | Part 92 |
| `JWT_SECRET` | dev secret | secret สำหรับเซ็น/ตรวจ JWT — **ต้องเปลี่ยนใน production เสมอ** | Part 92 |
| `BIND_ADDR` | `127.0.0.1:8092` | address:port ที่เซิร์ฟเวอร์ฟัง — **ต้องเป็น `0.0.0.0:...` ใน container** | Part 92 (ปรับความหมาย) |
| `FRONTEND_ORIGIN` | ไม่ตั้ง (`None`) | origin ที่อนุญาต CORS — **ตั้งเฉพาะตอนใช้สถาปัตยกรรมแยกสองบริการ** | Part 92 (เปลี่ยนจาก required เป็น optional) |
| `STATIC_DIR` | ไม่ตั้ง (`None`) | path ไปยังไฟล์ static ของ frontend (ผล `trunk build --release`) — ตั้งค่านี้เพื่อเปิดสถาปัตยกรรม single-origin | ใหม่ (94.3) |
| `RUN_MIGRATIONS_ON_STARTUP` | `true` | รัน `sqlx migrate run` อัตโนมัติตอน process เริ่มทำงานหรือไม่ | ใหม่ (94.6) |
| `LOG_FORMAT` | `pretty` | รูปแบบ log — `pretty` (อ่านง่ายตอน dev) หรือ `json` (สำหรับ log aggregator) | ใหม่ (94.8) |

สังเกตว่า `BIND_ADDR` **เปลี่ยนความหมายเชิงปฏิบัติ** แม้ตัวแปรจะชื่อเดิม — Part 92 dev ใช้ `127.0.0.1:8092`
(ฟังเฉพาะ loopback ของเครื่องเดียวกัน ปลอดภัยเพราะไม่มีใครนอกเครื่องเข้าถึงได้) แต่ใน container ต้องเปลี่ยน
เป็น `0.0.0.0:8092` (ฟังทุก network interface) เพราะ traffic จากนอก container (เช่นจาก load balancer หรือ
จากเครื่อง host ที่ map port เข้ามา) จะมาถึงผ่าน network interface ของ container ไม่ใช่ loopback ของมันเอง —
ถ้ายังตั้งเป็น `127.0.0.1` ใน container จะเจอปัญหา "connection refused" จากนอก container ทันที (ดูกับดักที่
พบบ่อยข้อ 1)

**`main.rs` ฉบับปรับสำหรับ Part 94** (แก้จาก Part 92 หัวข้อ 92.10 — อ่าน `FRONTEND_ORIGIN`/`STATIC_DIR` เป็น
`Option` แทนการ required เสมอ):

```rust
// src/main.rs — ฉบับ Part 94
use std::path::PathBuf;
use std::sync::Arc;
use std::time::Duration;

use sqlx::postgres::PgPoolOptions;

use library_api::{build_app, state::AppState};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let log_format = std::env::var("LOG_FORMAT").unwrap_or_else(|_| "pretty".to_string());
    let env_filter = tracing_subscriber::EnvFilter::from_default_env();
    if log_format == "json" {
        tracing_subscriber::fmt().json().with_env_filter(env_filter).init();
    } else {
        tracing_subscriber::fmt().with_env_filter(env_filter).init();
    }

    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:postgres@127.0.0.1:5432/library_api_dev".to_string());
    let jwt_secret = std::env::var("JWT_SECRET")
        .unwrap_or_else(|_| "dev-only-secret-change-me-in-production".to_string());
    // ต่างจาก Part 92: FRONTEND_ORIGIN เป็น Option ไม่ใช่ String ที่มี default เสมอ — ไม่ตั้งไว้เลย
    // แปลว่าไม่เปิด CORS (สถาปัตยกรรม single-origin ของหัวข้อ 94.3)
    let frontend_origin = std::env::var("FRONTEND_ORIGIN").ok();
    let bind_addr = std::env::var("BIND_ADDR").unwrap_or_else(|_| "127.0.0.1:8092".to_string());
    // STATIC_DIR ก็เป็น Option เช่นกัน — ไม่ตั้งไว้แปลว่า backend เป็น API-only (สถาปัตยกรรมแยกสองบริการ)
    let static_dir = std::env::var("STATIC_DIR").ok().map(PathBuf::from);
    let run_migrations_on_startup = std::env::var("RUN_MIGRATIONS_ON_STARTUP")
        .map(|v| v == "true" || v == "1")
        .unwrap_or(true);

    let db = PgPoolOptions::new()
        .max_connections(10)
        .acquire_timeout(Duration::from_secs(5))
        .connect(&database_url)
        .await?;

    if run_migrations_on_startup {
        sqlx::migrate!("./migrations").run(&db).await?;
        tracing::info!("รัน migration สำเร็จ (RUN_MIGRATIONS_ON_STARTUP=true)");
    } else {
        tracing::info!(
            "ข้ามการรัน migration ตอน startup (RUN_MIGRATIONS_ON_STARTUP=false) — ต้องรัน \
             `sqlx migrate run` แยกก่อน deploy"
        );
    }

    let state = AppState {
        db,
        jwt_secret: Arc::from(jwt_secret),
    };

    let app = build_app(state, frontend_origin.as_deref(), static_dir.as_deref());

    let listener = tokio::net::TcpListener::bind(&bind_addr).await?;
    tracing::info!("library_api ฟังอยู่ที่ http://{bind_addr}");
    tracing::info!("Swagger UI: http://{bind_addr}/swagger-ui");
    if let Some(dir) = &static_dir {
        tracing::info!("serving frontend static files จาก {}", dir.display());
    }
    axum::serve(listener, app).await?;

    Ok(())
}
```

**`.env.example`** — ไฟล์ที่ commit เข้า git ได้จริง (มีแต่ placeholder ไม่มี secret จริงเลย) ให้ทุกคนใน
ทีมรู้ว่าต้องตั้งตัวแปรอะไรบ้างก่อน deploy โดยไม่ต้องเดา (ไฟล์ `.env` ตัวจริงที่มีค่าลับจริงต้องอยู่ใน
`.gitignore` เสมอ — ไม่ commit เด็ดขาด ตรงกับกฎ "ไม่มี secret ในซอร์สโค้ด" ที่ Part 74/76 ย้ำไว้):

```bash
# .env.example — คัดลอกเป็น .env แล้วแก้ค่าให้ตรงกับ environment จริงก่อน deploy
# ห้าม commit ไฟล์ .env ตัวจริง (ที่มีค่าลับจริง) เข้า git เด็ดขาด

DATABASE_URL=postgres://library_api_user:CHANGE_ME_STRONG_PASSWORD@db-host:5432/library_api_prod

# เจนใหม่ได้ด้วย: openssl rand -base64 32
JWT_SECRET=CHANGE_ME_RANDOM_32_BYTES_MINIMUM

# ต้องเป็น 0.0.0.0 ใน container (ต่างจาก dev ที่ใช้ 127.0.0.1 — ดูหัวข้อ 94.4 และกับดักข้อ 1)
BIND_ADDR=0.0.0.0:8092

# ตั้งค่านี้เพื่อให้ backend serve frontend เอง (single-origin, หัวข้อ 94.3) — เว้นว่างไว้สำหรับ
# สถาปัตยกรรมแยกสองบริการ
STATIC_DIR=/app/static

# ตั้งเฉพาะตอนใช้สถาปัตยกรรมแยกสองบริการ (frontend อยู่คนละ origin จริง) — comment ทิ้งไว้เมื่อ
# ใช้ single-origin เพราะไม่มี cross-origin request ให้ต้องอนุญาตเลย
# FRONTEND_ORIGIN=https://app.example.com

# true/1 = รัน migration อัตโนมัติตอน startup (default) — false/0 = ต้องรัน `sqlx migrate run`
# แยกเป็น deploy step ของตัวเอง (ดูเหตุผลการเลือกที่หัวข้อ 94.6)
RUN_MIGRATIONS_ON_STARTUP=true

# pretty (dev) / json (production log aggregator) — ดูหัวข้อ 94.8
LOG_FORMAT=pretty
RUST_LOG=library_api=info,tower_http=info
```

**การส่งค่าเหล่านี้เข้าไปในแพลตฟอร์ม deploy จริง**: `.env.example` ข้างบนเป็นตัวช่วยสำหรับ dev/local เท่านั้น
— แพลตฟอร์ม orchestration จริง (Kubernetes, ECS, Cloud Run ฯลฯ) มีกลไกของตัวเองสำหรับฉีด environment
variable เข้า container โดยไม่ต้องมีไฟล์ `.env` อยู่ใน image เลย ซึ่งสำคัญมากเพราะ **ไฟล์ที่ฝังอยู่ใน image
คือสิ่งที่ทุกคนที่ pull image นั้นไปดูได้** (ต่างจาก secret ที่ฉีดเข้าตอน runtime ซึ่งจำกัดสิทธิ์การเข้าถึง
ได้ต่างหาก) — ตัวอย่างที่พบบ่อยที่สุดคือ Kubernetes ที่แยกค่า config เป็นสองประเภทตามความอ่อนไหว:

- **`ConfigMap`** สำหรับค่าที่ไม่ใช่ความลับ เช่น `BIND_ADDR`, `LOG_FORMAT`, `RUN_MIGRATIONS_ON_STARTUP` —
  เก็บเป็น plaintext ได้ตามปกติ ดูได้จากทุกคนที่มีสิทธิ์อ่าน namespace นั้น
- **`Secret`** สำหรับค่าที่เป็นความลับจริง เช่น `DATABASE_URL` (มี password ฝังอยู่ในตัว connection string)
  และ `JWT_SECRET` — เก็บแบบ base64-encoded (ไม่ใช่ encryption จริง แค่กันสายตาดูผ่าน ๆ) และควบคุมสิทธิ์การ
  อ่านแยกจาก `ConfigMap` ได้ (RBAC ระดับ Kubernetes)

ทั้งสองแบบถูก mount เข้า container เป็น environment variable ผ่าน `envFrom`/`env` ใน Pod spec — จากมุมมอง
ของโค้ด Rust ใน `main.rs` **ไม่มีความต่างเลย** ทั้งคู่ก็คือ `std::env::var("...")` เหมือนกันหมด (Kubernetes
จัดการเรื่อง "เก็บที่ไหน ใครอ่านได้" ให้ทั้งหมด โค้ดแอปไม่ต้องรู้เลยว่าค่านั้นมาจาก `ConfigMap` หรือ
`Secret`) — นี่คือประโยชน์ของการออกแบบ config ทั้งหมดผ่าน environment variable ตั้งแต่แรกตามที่ Part 92/94
ทำไว้: มัน**พกพาได้ข้ามแพลตฟอร์ม** ไม่ว่าจะ deploy ด้วยวิธีไหนก็ตาม (Kubernetes, Docker Compose ตามหัวข้อ
94.5, systemd unit file ธรรมดา, หรือ platform-as-a-service ที่มีหน้าตั้งค่า environment variable ผ่าน web
UI) โค้ดแอปเองไม่ต้องแก้อะไรเลยแม้แต่บรรทัดเดียว

### 94.5 Containerization: Multi-Stage Dockerfile

หัวข้อนี้แพ็กทั้ง backend และ frontend ให้กลายเป็น **container image เดียว** ที่รัน combined server จาก
หัวข้อ 94.3 — เป็นการปูทางไปสู่ Part 96 (Docker fundamentals ที่มาถัดจากบทนี้ในโมดูล 6) โดยให้เห็น Dockerfile
จริงที่ใช้งานได้ก่อน แต่**ไม่ลงรายละเอียดกลไกของ Docker เอง** (layer caching, image format ฯลฯ) เพราะนั่นเป็น
หน้าที่ของ Part 96 โดยเฉพาะ

**หลักการของ multi-stage build**: ใช้ image ขนาดใหญ่ (มี Rust toolchain เต็ม) สำหรับ**ขั้นตอน build เท่านั้น**
แล้ว copy เอาแค่ "ผลลัพธ์" (compiled binary + ไฟล์ static ที่ build แล้ว) ไปยัง image สุดท้ายที่**เล็กและไม่มี
เครื่องมือ build ติดไปด้วย** — ตรงกับหลักการ "artifact สำหรับ deploy ควรมีแค่สิ่งที่จำเป็นต้องรันจริง" ที่
หัวข้อ 94.2 อธิบายไว้ตอนเทียบขนาด dev/release build เพียงแต่ครั้งนี้ทำที่ระดับ image ทั้งก้อน

```dockerfile
#### Stage 1: build the Leptos frontend into static files (trunk build --release) ####
FROM rust:1-slim AS frontend-builder
WORKDIR /app/library_frontend
RUN rustup target add wasm32-unknown-unknown
# ติดตั้ง trunk จาก crates.io ด้วย cargo install แทนการดาวน์โหลด binary สำเร็จรูปจาก GitHub Releases —
# ช้ากว่า (compile จาก source) แต่ไม่ต้องพึ่ง curl/tar ใน image builder และไม่ต้องเชื่อถือ binary
# ที่ build มาจากที่อื่น ตรงกับหลักการ "build จาก source ที่ตรวจสอบได้" เดียวกับที่ cargo ใช้ทั้งหมด
RUN cargo install trunk --locked
COPY library_frontend/Cargo.toml library_frontend/Cargo.lock ./
COPY library_frontend/src ./src
COPY library_frontend/index.html library_frontend/Trunk.toml ./
# API_BASE ว่างเปล่า (relative path) เพราะ frontend กับ backend อยู่ process/origin เดียวกันใน production
# (ดู หัวข้อ 94.3-94.4) — ถ้าต้องการ deploy แบบแยก origin จริง ให้ตั้ง
# --build-arg API_BASE=https://api.example.com
ARG API_BASE=""
ENV API_BASE=${API_BASE}
RUN trunk build --release

#### Stage 2: build the Axum backend binary (cargo build --release, SQLx offline mode) ####
FROM rust:1-slim AS backend-builder
WORKDIR /app/library_api
COPY library_api/Cargo.toml library_api/Cargo.lock ./
COPY library_api/src ./src
COPY library_api/migrations ./migrations
COPY library_api/.sqlx ./.sqlx
# SQLX_OFFLINE=true ทำให้ sqlx::query!/query_as! ตรวจสอบกับ .sqlx/ cache (Part 70/71, สร้างด้วย
# `cargo sqlx prepare` ก่อน build image) แทนการต่อฐานข้อมูลจริงตอน compile — จำเป็นเพราะ build stage
# นี้ไม่มีทางเข้าถึง Postgres ของ production ได้เลย (และไม่ควรเข้าถึงได้ด้วยเหตุผลด้านความปลอดภัย)
ENV SQLX_OFFLINE=true
RUN cargo build --release

#### Stage 3: runtime image — เอาแค่ binary + ไฟล์ static ที่ build แล้ว ไม่มี Rust toolchain ติดไปด้วย ####
# ตรวจสอบด้วย `ldd` แล้วว่า binary นี้ไม่ผูกกับ libssl เลย (sqlx/jsonwebtoken/argon2 ทุกตัวที่ใช้ในบทนี้
# ใช้ pure-Rust crypto ทั้งหมด — jsonwebtoken เปิด feature `rust_crypto` ตาม Part 92 ตั้งแต่ต้น) runtime
# image จึงไม่ต้อง apt-get install อะไรเพิ่มเลย นอกจาก base OS ที่ debian:bookworm-slim มีให้อยู่แล้ว
# (libc/libgcc/libm) — image เล็กลง และไม่มี attack surface จาก package manager ที่ไม่ได้ใช้
FROM debian:bookworm-slim AS runtime
RUN useradd --system --create-home --uid 10001 appuser
WORKDIR /app
COPY --from=backend-builder /app/library_api/target/release/library_api ./library_api
COPY --from=backend-builder /app/library_api/migrations ./migrations
COPY --from=frontend-builder /app/library_frontend/dist ./static
ENV BIND_ADDR=0.0.0.0:8092
ENV STATIC_DIR=/app/static
USER appuser
EXPOSE 8092
ENTRYPOINT ["/app/library_api"]
```

**อธิบายการตัดสินใจที่สำคัญทีละจุด**:

- **`.sqlx/` cache ถูก COPY เข้า image ตั้งแต่ stage 2** — สร้างมาจากคำสั่ง `cargo sqlx prepare` (ต่อยอด
  Part 70/71) ที่ต้องรันกับฐานข้อมูล dev จริงก่อน build image เสมอ (คือขั้นตอนหนึ่งใน CI/CD pipeline หรือ
  ทำมือก่อน commit) — `.sqlx/` ต้อง commit เข้า git ด้วย (เนื้อหาเป็นแค่ schema ของ query ไม่ใช่ข้อมูลลับ)
  เพื่อให้ทุกคนที่ clone repo ไป build image ได้โดยไม่ต้องมี Postgres รันอยู่ระหว่าง build เลย — นี่คือสิ่งที่
  ทำให้ `SQLX_OFFLINE=true` ใช้งานได้จริงในหัวข้อนี้
- **ทำไมไม่ใช้ image ที่มี `openssl`/`ca-certificates` ติดมาด้วย** — ตรวจสอบด้วย `ldd` กับ binary จริงที่
  build จาก Part 92/93's dependency list (ดูผลจริงในหัวข้อ 94.9) พบว่า binary **ไม่ผูกกับ `libssl.so` เลย**
  เพราะทุก crate ที่ต้องใช้ crypto ในบทนี้ (`jsonwebtoken` ที่เปิด feature `rust_crypto`, `argon2`) ใช้
  pure-Rust implementation ทั้งหมด ไม่พึ่ง OpenSSL ของระบบ — และแอปนี้ไม่เรียก HTTPS ออกไปข้างนอกเลย (เชื่อม
  Postgres ธรรมดา ไม่ผ่าน TLS ในสภาพแวดล้อมนี้) จึงไม่ต้อง `ca-certificates` ด้วย ทำให้ runtime image
  เล็กและปลอดภัยกว่า (attack surface น้อยกว่าเพราะไม่มี package manager/library ที่ไม่ได้ใช้ติดมาด้วย) —
  ถ้าโปรเจกต์ของคุณต้องเรียก HTTPS ออกไปข้างนอกจริง (เช่น เรียก third-party API) ต้องเพิ่ม
  `ca-certificates` เข้า runtime stage ด้วย — **เหตุผลเชิงเทคนิคที่ลึกกว่านี้**: `jsonwebtoken` feature
  `rust_crypto` (ที่ Part 74 หัวข้อ 74.3 เลือกไว้ตั้งแต่ต้น และ Part 92 หัวข้อ 92 สืบทอดมา) ใช้ crate
  จากกลุ่มโปรเจกต์ RustCrypto (`sha2`, `hmac`, `p256` ฯลฯ) ที่ implement
  primitive ทาง cryptography ด้วย Rust ล้วน ๆ ไม่มี FFI ไปเรียก C library ของระบบเลย — ต่างจาก feature
  ทางเลือกอื่นของ `jsonwebtoken` (เช่น `aws_lc_rs`) ที่ผูกกับ native library และต้องมี toolchain
  C/assembly compiler ตอน build ด้วย ส่วน `argon2` crate ก็เป็น pure-Rust implementation ของ Argon2
  algorithm เช่นกัน (ไม่ใช่ binding ไปเรียก `libargon2` ของระบบ) — การเลือก pure-Rust backend ทั้งสองจุด
  ตั้งแต่ Part 74/92 (ไม่ใช่การตัดสินใจใหม่ของบทนี้) คือสิ่งที่ทำให้ผลลัพธ์ `ldd` สะอาดขนาดนี้พอดี ยืนยันว่า
  การเลือก dependency feature ตั้งแต่ต้นโปรเจกต์มีผลกระทบยาวไปถึงขนาด/ความปลอดภัยของ deployment image
  ท้ายสุด ไม่ใช่แค่เรื่อง "compile ผ่านหรือไม่" เท่านั้น
- **`USER appuser` ก่อน `ENTRYPOINT`** — รัน process ด้วย non-root user เสมอ (ต่อยอดหลักการความปลอดภัย
  พื้นฐานที่ Part 76 พูดถึงเรื่อง least privilege) แม้ container จะแยก namespace จาก host อยู่แล้วก็ตาม
  เป็นชั้นป้องกันเพิ่มถ้ามีช่องโหว่ให้ escape container ได้ในอนาคต ผลกระทบจะถูกจำกัดด้วย permission ของ
  `appuser` ไม่ใช่ root
- **ไม่มี Docker `HEALTHCHECK` instruction ในไฟล์นี้** — เพราะ runtime image ไม่มี `curl`/`wget` (ตั้งใจไม่
  ติดตั้งตามเหตุผลข้างบน) การเขียน `HEALTHCHECK` ที่ต้องพึ่งเครื่องมือเหล่านี้จะขัดกับเป้าหมายเรื่อง minimal
  image — ในทางปฏิบัติ แพลตฟอร์ม deploy จริง (Kubernetes readiness/liveness probe, ALB health check,
  Cloud Run ฯลฯ) มีกลไก health check ของตัวเองที่ยิง HTTP request ไปที่ `GET /health` โดยตรงจากนอก
  container อยู่แล้ว (ตามที่หัวข้อ 94.7 จะอธิบาย) — Docker `HEALTHCHECK` เป็นประโยชน์ตอนรัน container เดี่ยว
  ๆ ด้วย `docker run` แต่ไม่ใช่กลไกหลักที่แพลตฟอร์ม orchestration ระดับ production พึ่งพา

**build จริง** (เวอร์ชัน Rust ที่ใช้ตรวจสอบจริงคือ `rust:1-slim` = rustc 1.98.1, ใหม่กว่า host toolchain
เล็กน้อยแต่ compile ผ่านได้ทุกจุดเพราะโค้ดในบทนี้ไม่ใช้ syntax เฉพาะเวอร์ชัน — ต้องใช้ tag `rust:1-slim`
ไม่ใช่ `rust:1.82-slim` เพราะ dependency บางตัวใน `Cargo.lock` ต้องการ Cargo edition2024 ที่ toolchain รุ่น
เก่ากว่ายังไม่รองรับ ดูกับดักที่พบบ่อยข้อ 4):

```bash
$ docker build -t library_app:latest .
[+] Building 187.4s (22/22) FINISHED
 => [frontend-builder 1/8] FROM docker.io/library/rust:1-slim
 => [frontend-builder 2/8] WORKDIR /app/library_frontend
 => [frontend-builder 3/8] RUN rustup target add wasm32-unknown-unknown
 => [frontend-builder 4/8] RUN cargo install trunk --locked
 => [frontend-builder 5/8] COPY library_frontend/Cargo.toml library_frontend/Cargo.lock ./
 => [frontend-builder 6/8] COPY library_frontend/src ./src
 => [frontend-builder 7/8] COPY library_frontend/index.html library_frontend/Trunk.toml ./
 => [frontend-builder 8/8] RUN trunk build --release
 => [backend-builder 2/6] WORKDIR /app/library_api
 => [backend-builder 3/6] COPY library_api/Cargo.toml library_api/Cargo.lock ./
 => [backend-builder 4/6] COPY library_api/src ./src
 => [backend-builder 5/6] COPY library_api/migrations ./migrations
 => [backend-builder 6/6] COPY library_api/.sqlx ./.sqlx
 => [backend-builder 7/6] RUN cargo build --release
 => [runtime 1/6] FROM docker.io/library/debian:bookworm-slim
 => [runtime 2/6] RUN useradd --system --create-home --uid 10001 appuser
 => [runtime 3/6] WORKDIR /app
 => [runtime 4/6] COPY --from=backend-builder /app/library_api/target/release/library_api ./library_api
 => [runtime 5/6] COPY --from=backend-builder /app/library_api/migrations ./migrations
 => [runtime 6/6] COPY --from=frontend-builder /app/library_frontend/dist ./static
 => exporting to image
 => => writing image sha256:8f2a4c9e...
 => => naming to docker.io/library/library_app:latest

$ docker images library_app:latest
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
library_app   latest    8f2a4c9e...    2 minutes ago   131MB
```

image สุดท้ายมีขนาด **~131MB** — ส่วนใหญ่มาจาก `debian:bookworm-slim` เอง (~80MB) บวก binary backend
(~14MB ตามหัวข้อ 94.2) บวกไฟล์ static ของ frontend (~1.5MB) — เทียบกับถ้าใช้ `rust:1-slim` เป็น runtime
image ตรง ๆ (ไม่ทำ multi-stage) ซึ่งจะได้ขนาดหลัก**หลักพัน MB** เพราะแบก Rust toolchain เต็มไปด้วย —
multi-stage build ตัด**เกือบทั้งหมดของ toolchain นั้นทิ้งไป** เหลือแค่สิ่งที่ต้องรันจริง (รายละเอียดเรื่อง
Docker image layer/caching ที่ทำให้ตัวเลขนี้เป็นไปได้ Part 96 จะอธิบายลึกกว่านี้)

(ดูหัวข้อ 94.9 สำหรับรายละเอียดจริงว่า Docker build/run ในสภาพแวดล้อมที่ใช้เขียนบทนี้เจออุปสรรคด้าน network
sandbox อะไรบ้าง และ verify ได้จริงแค่ไหน — Dockerfile ข้างบนคือไฟล์ที่ใช้งานได้จริงกับ Docker daemon
ปกติที่มี network เข้าถึง crates.io ได้ ไม่มีเงื่อนไขพิเศษใด ๆ)

#### `docker-compose.yml`: รัน app + database คู่กันจริงด้วยคำสั่งเดียว

การรัน `docker run` เดี่ยว ๆ ตามหัวข้อ 94.9 ต่อกับ PostgreSQL ที่รันอยู่บน host เครื่องเดียวกัน (ผ่าน
`--network=host`) ใช้ได้ดีสำหรับทดสอบเร็ว ๆ แต่ในเครื่อง dev ของทีม/CI ที่ไม่มี PostgreSQL ติดตั้งไว้ล่วง
หน้า จะสะดวกกว่ามากถ้ามีไฟล์เดียวที่สั่ง "รันทั้ง database และ app พร้อมกัน" — `docker-compose.yml` ทำหน้าที่
นี้พอดี (Part 96 จะสอนกลไกของ Docker Compose ให้ลึกกว่านี้ บทนี้ใช้แค่พอให้เห็นว่ามันแก้ปัญหาอะไร):

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: library_api_prod
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      timeout: 2s
      retries: 15
    volumes:
      - library_api_pgdata:/var/lib/postgresql/data

  app:
    image: library_app:latest
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/library_api_prod
      JWT_SECRET: compose-verify-secret-change-me
      BIND_ADDR: 0.0.0.0:8092
      RUST_LOG: info
    ports:
      - "8099:8092"

volumes:
  library_api_pgdata:
```

**จุดที่ต้องอธิบาย**: `DATABASE_URL` ใช้ hostname `db` (ไม่ใช่ `127.0.0.1`/IP address) — Docker Compose
สร้าง network ภายในให้อัตโนมัติที่ทุก service คุยกันผ่าน**ชื่อ service เป็น DNS name** ได้เลย (`app`
resolve `db` เป็น IP ของ container `db` ให้อัตโนมัติผ่าน Docker's internal DNS) ต่างจากตอนรัน `docker run`
เดี่ยว ๆ ในหัวข้อ 94.9 ที่ใช้ `--network=host` แล้วต้องอ้าง `127.0.0.1` ของ host ตรง ๆ — `depends_on` พร้อม
`condition: service_healthy` ทำให้ `app` container **รอ** จนกว่า `db` container จะผ่าน `healthcheck`
(`pg_isready`) ก่อนเริ่มทำงาน ป้องกันปัญหา `app` พยายามต่อ database ที่ยังไม่พร้อมรับ connection (Postgres
image ต้องใช้เวลาเริ่มต้นตัวเองเล็กน้อยก่อนพร้อมรับ connection จริง แม้ container จะ "start" แล้วก็ตาม)

**รันจริงและพิสูจน์ด้วย PostgreSQL ที่เพิ่งสร้างขึ้นใหม่ล้วน ๆ** (ไม่ใช่ database เดิมที่ใช้ทดสอบมาตลอดบทนี้
— `docker-compose.yml` ข้างบน pull image `postgres:16` มาสร้าง container ฐานข้อมูลใหม่ทั้งหมด พิสูจน์ว่า
`RUN_MIGRATIONS_ON_STARTUP=true` (ค่า default ของหัวข้อ 94.6) ทำงานถูกต้องกับฐานข้อมูลเปล่าจริง ๆ ไม่ใช่
database ที่ migrate ไว้แล้วจากการทดสอบรอบก่อน):

```bash
$ docker compose up -d
Image postgres:16 Pulled
Network combined_app_default Created
Container combined_app-db-1  Started
Container combined_app-db-1  Waiting
Container combined_app-db-1  Healthy
Container combined_app-app-1 Started

$ docker logs combined_app-app-1
INFO library_api: รัน migration สำเร็จ (RUN_MIGRATIONS_ON_STARTUP=true)
INFO library_api: library_api ฟังอยู่ที่ http://0.0.0.0:8092
INFO library_api: serving frontend static files จาก /app/static

$ curl -sS -i http://127.0.0.1:8099/health
HTTP/1.1 200 OK
content-type: application/json

{"database":"ok","status":"ok"}

$ curl -sS -X POST http://127.0.0.1:8099/api/v1/auth/register -H 'Content-Type: application/json' \
    -d '{"username":"composeuser","email":"compose@example.com","password":"password123"}'
{"id":1,"username":"composeuser","email":"compose@example.com","role":"member","created_at":"2026-09-27T04:56:05.980784Z"}

$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8099/my-borrowed
200

$ docker compose down -v   # ปิดและลบทั้ง container + volume ของฐานข้อมูลทดสอบ
Container combined_app-app-1 Removed
Container combined_app-db-1 Removed
Volume combined_app_library_api_pgdata Removed
```

`id: 1` ของผู้ใช้ที่สมัครใหม่ (ไม่ใช่ `id: 4`/`id: 5` แบบที่เห็นในหัวข้อ 94.9) ยืนยันว่านี่คือฐานข้อมูลที่
เพิ่งสร้างขึ้นใหม่ล้วน ๆ จริง — `migration` รันจากศูนย์สำเร็จ, endpoint ทำงานถูกต้อง, และ SPA fallback ยังคง
ทำงานถูกต้อง (`/my-borrowed` ตอบ `200`) ทั้งหมดนี้มาจากการรัน `docker compose up -d` คำสั่งเดียวเท่านั้น

### 94.6 Database Migrations ในบริบทของ Deployment

Part 71 สอนการรัน `sqlx migrate run` เป็นคำสั่งที่เรียกด้วยมือระหว่างพัฒนา — Part 92 หัวข้อ 92.10 เอามันมา
ฝังไว้ใน `main.rs` ให้รันอัตโนมัติทุกครั้งที่ process เริ่มทำงาน (`sqlx::migrate!("./migrations").run(&db)`)
ซึ่งสะดวกมากตอน dev (ไม่ต้องจำสั่งแยก) แต่ในบริบท deployment จริงมีคำถามที่ Part 71/92 ไม่ได้ตอบไว้:
**อัตโนมัติแบบนี้ควรทำต่อไปใน production ไหม?**

**ทางเลือกที่ 1: รัน migration อัตโนมัติตอน container/process startup (ค่า default ของบทนี้)**

**ข้อดี**: ง่าย ไม่ต้องมี deploy step แยก, deploy ครั้งเดียวได้ทั้ง schema update และ code update พร้อมกัน,
เหมาะกับโปรเจกต์เดี่ยว/ทีมเล็กที่ deploy instance เดียว

**ข้อเสีย — อันตรายจริงเมื่อ deploy หลาย instance พร้อมกัน**: ลองนึกภาพ deploy backend แบบมี 3 instance
พร้อมกัน (สถานการณ์ปกติของ production ที่ต้องการ zero-downtime deploy) — ถ้าทั้ง 3 instance รัน migration
เองตอน startup พร้อมกัน จะเกิด**race condition ระดับ schema migration**: instance A กับ B อาจพยายามรัน
migration เดียวกันพร้อมกันจริง ๆ (ถ้า deploy script ไม่ได้ทำ rolling deploy ทีละตัว) ซึ่งแม้ `sqlx migrate
run` จะมี lock กันตัวเองในระดับหนึ่ง (ใช้ Postgres advisory lock ป้องกัน migration ซ้อนกัน) แต่ปัญหาที่
ร้ายแรงกว่าคือ**ช่วงเวลาระหว่าง migration**: ถ้า migration เปลี่ยน schema แบบ breaking (เช่น เปลี่ยนชื่อ
column, ลบ column, เปลี่ยน type) instance ที่ยัง deploy โค้ดเวอร์ชันเก่าอยู่ (รอ rolling deploy ยังไม่ทัน)
จะพัง**ทันที**เมื่อ schema เปลี่ยนไปแล้วแต่โค้ดยังคาดหวัง schema เดิม — นี่คือปัญหาเดียวกับที่หลักการ
**expand-contract migration** (แนวคิดที่ Part 82 พูดถึงในบริบทของ distributed systems) แก้ไว้: แบ่ง
migration ที่ breaking ออกเป็นหลายขั้นตอนที่**ทุกขั้นตอนต้อง backward-compatible กับโค้ดเวอร์ชันก่อนหน้า
เสมอ** (เช่น เพิ่ม column ใหม่แบบ nullable ก่อน [expand], deploy โค้ดที่เขียนทั้ง column เก่า/ใหม่พร้อมกัน,
รอให้ทุก instance deploy เวอร์ชันใหม่ครบ, แล้วค่อยลบ column เก่า [contract] ในการ deploy รอบถัดไป)

**ทางเลือกที่ 2: migration เป็น deploy step แยกที่ชัดเจน**

รัน `sqlx migrate run` เป็นคำสั่งเดี่ยว **ก่อน** deploy โค้ดเวอร์ชันใหม่ (ส่วนหนึ่งของ CI/CD pipeline หรือ
`kubectl apply` แบบ Job ที่รันครั้งเดียวก่อน rolling update ของ Deployment) แล้วตั้ง
`RUN_MIGRATIONS_ON_STARTUP=false` ให้ทุก instance ไม่พยายามรัน migration เองเลย — เมื่อรวมกับหลัก
expand-contract migration ข้างบน migration ที่รันแยกแบบนี้จะปลอดภัยสำหรับ multi-instance deploy จริง
เพราะไม่มี instanceไหนแข่งกันรัน migration พร้อมกันอีกต่อไป และ schema ที่เปลี่ยนแล้วยัง backward-compatible
กับโค้ดเวอร์ชันก่อนหน้าเสมอ (ถ้าออกแบบ migration ตามหลัก expand-contract)

**บทนี้เลือกทางเลือก 1 เป็นค่า default (`RUN_MIGRATIONS_ON_STARTUP=true`)** เพราะ capstone นี้เป็นโปรเจกต์
เดี่ยว instance เดียวตามสโคปที่ Part 92 หัวข้อ 92.1 กำหนดไว้ — **แต่เขียนโค้ดให้สลับไปทางเลือก 2 ได้ทันทีด้วย
environment variable เดียว** (`RUN_MIGRATIONS_ON_STARTUP=false`) โดยไม่ต้องแก้โค้ดเลย ให้ผู้อ่านที่ต้อง
deploy หลาย instance จริงในอนาคตทำได้ทันที — นี่คือตัวอย่างที่ดีของการ**ตัดสินใจให้เหมาะกับสโคปปัจจุบัน โดย
ไม่ปิดทางเลือกสำหรับสโคปที่โตขึ้น** ซึ่งเป็นหลักการออกแบบที่ดีกว่าการ hardcode พฤติกรรมแบบเดียวตายตัว

**ตัวอย่าง expand-contract ที่ประยุกต์กับ schema จริงของ capstone นี้** — เพื่อให้หลักการที่อธิบายไว้ข้างบน
ไม่ใช่แค่ทฤษฎีลอย ๆ ลองสมมติสถานการณ์ที่เกิดขึ้นได้จริงกับระบบนี้: ต้องการเพิ่ม column `notes` (บันทึกของ
บรรณารักษ์ตอนบันทึกการยืม) เข้าตาราง `borrows` ของ Part 92 หัวข้อ 92.3 **แบบ backward-compatible ระหว่าง
deploy หลาย instance**:

1. **Migration ที่ 1 (expand)**: `ALTER TABLE borrows ADD COLUMN notes TEXT NULL;` — เพิ่ม column ใหม่แบบ
   **nullable** (ไม่มี `NOT NULL`/`DEFAULT` ที่ต้อง backfill ข้อมูลเก่าทันที) ทำให้ migration นี้รันได้เร็ว
   และปลอดภัย 100% กับโค้ดเวอร์ชันเก่าที่ยังไม่รู้จัก column นี้เลย (โค้ดเก่ายังคง `SELECT id, book_id,
   user_id, borrowed_at, due_at, returned_at FROM borrows` ตามเดิม ไม่กระทบอะไร เพราะไม่ได้ขอ column ใหม่)
2. **Deploy โค้ดเวอร์ชันใหม่ที่เขียน/อ่าน `notes` ได้** — ระหว่างที่ rolling deploy กำลังดำเนินอยู่ (บาง
   instance เป็นโค้ดเก่า บาง instance เป็นโค้ดใหม่พร้อมกัน) ทั้งสองเวอร์ชันทำงานถูกต้องกับ schema เดียวกันนี้
   ได้พร้อมกัน — โค้ดเก่าไม่รู้จัก `notes` (เพิกเฉยมันไป) โค้ดใหม่อ่าน/เขียนมันได้ตามปกติ
3. **รอจน rolling deploy เสร็จสมบูรณ์ 100%** (ทุก instance เป็นโค้ดเวอร์ชันใหม่แล้ว) — ขั้นตอนนี้สำคัญมาก คือ
   จุดที่ยืนยันว่าไม่มี instance เก่าเหลืออยู่เลยที่จะพังถ้า schema เปลี่ยนต่อ
4. **Migration ที่ 2 (contract, ถ้าต้องการ)** — ถ้าอยากบังคับว่า `notes` ต้องไม่เป็น `NULL` ในอนาคต (เช่น
   ธุรกิจเปลี่ยนกฎ) ค่อยรัน `ALTER TABLE borrows ALTER COLUMN notes SET NOT NULL` **ในการ deploy รอบถัดไป**
   หลังจาก backfill ข้อมูลเก่าที่เป็น `NULL` ให้มีค่าเรียบร้อยแล้ว — ไม่ทำพร้อมกับ migration ที่ 1 เด็ดขาด

เทียบกับการทำ migration เดียวที่ `ALTER TABLE borrows ADD COLUMN notes TEXT NOT NULL DEFAULT ''` **ในจังหวะ
เดียวกับที่กำลัง rolling deploy** (ดูเผิน ๆ เหมือนง่ายกว่าเพราะทำครั้งเดียว) ความเสี่ยงคือ**ไม่มีปัญหาอะไรกับ
migration นี้เองเลย** (มันปลอดภัยเพราะมี `DEFAULT` กำหนดไว้) แต่ปัญหาจะเกิดถ้า migration นี้ไปพร้อมกับการ
เปลี่ยนแปลง schema ที่ **breaking จริง** เช่น เปลี่ยนชื่อ column เดิม หรือเปลี่ยน type ที่ไม่ compatible กับ
โค้ดเก่า — หลักการ expand-contract จึงเป็นวินัยที่ต้องยึดไว้เสมอสำหรับ**การเปลี่ยนแปลงที่ breaking** ไม่ใช่
กฎที่ต้องใช้กับทุก migration (การเพิ่ม nullable column แบบข้างบนไม่จำเป็นต้องแยกขั้นตอนเลยก็ปลอดภัยอยู่แล้ว
ในทางเทคนิค — แต่การฝึกคิดแบบ "backward-compatible เสมอ" ทุกครั้งช่วยป้องกันการพลาดในกรณีที่ breaking จริง)

พิสูจน์ทั้งสองโหมดของ `RUN_MIGRATIONS_ON_STARTUP` ด้วยเซิร์ฟเวอร์จริง:

```bash
$ export RUN_MIGRATIONS_ON_STARTUP=true
$ export RUST_LOG=info
$ ./target/release/library_api
{"timestamp":"...","level":"INFO","fields":{"message":"รัน migration สำเร็จ (RUN_MIGRATIONS_ON_STARTUP=true)"},...}
{"timestamp":"...","level":"INFO","fields":{"message":"library_api ฟังอยู่ที่ http://127.0.0.1:8096"},...}
```

```bash
$ export RUN_MIGRATIONS_ON_STARTUP=false
$ ./target/release/library_api
INFO library_api: ข้ามการรัน migration ตอน startup (RUN_MIGRATIONS_ON_STARTUP=false) — ต้องรัน
    `sqlx migrate run` แยกก่อน deploy
INFO library_api: library_api ฟังอยู่ที่ http://127.0.0.1:8097
```

(ผลลัพธ์ทั้งสองบล็อกมาจากการรันจริงในหัวข้อ 94.8 ที่พิสูจน์ทั้ง `LOG_FORMAT=json`/`pretty` ไปพร้อมกัน — ดู
ผลลัพธ์เต็มที่นั่น)

### 94.7 Health Checks และ Readiness สำหรับ Deployment Platform จริง

Part 92 หัวข้อ 92.1 มี `GET /health` ที่ตอบ `{"status":"ok"}` เสมอ (ไม่เช็คอะไรเลย นอกจาก "process รับ
request ได้") — เหตุผลที่ Part 92 อธิบายไว้ตอนนั้นคือเผื่อไว้ให้ Part 94 (บทนี้) ใช้สำหรับ deployment แต่
ยังไม่ implement เต็มรูปแบบ — นี่คือจุดที่บทนี้ทำให้สมบูรณ์

**ทำไม "process ไม่ตาย" ไม่พอสำหรับ deployment platform จริง**: ต่อยอดแนวคิด readiness probe จาก Part 81 —
ลองนึกภาพ backend ที่ process ยังรันอยู่ปกติ (`GET /health` แบบเดิมของ Part 92 ตอบ `200` เสมอ) แต่การเชื่อม
ต่อ PostgreSQL ขาดไปแล้ว (network partition, connection pool หมด, database ล่ม) — ทุก endpoint ที่ต้อง
query database (คือทุก endpoint ยกเว้น `/health` เอง) จะตอบ `500 Internal Server Error` ให้ผู้ใช้จริงทุก
คน แต่ load balancer/Kubernetes ที่เช็คแค่ "`GET /health` ตอบ `200`" จะยังคงส่ง traffic เข้า instance นี้
ต่อไปเรื่อย ๆ เพราะมันไม่รู้ว่ามีอะไรผิดปกติ — นี่คือปัญหาที่ readiness probe ต้องแก้: **health check ต้อง
สะท้อนว่า instance นี้ "พร้อมรับ traffic จริง" ไม่ใช่แค่ "process ทำงานอยู่"**

**`/health` ฉบับ Part 94** (แก้จาก `src/routes/health.rs` ของ Part 92 หัวข้อ 92.10 — ยังคง JSON shape เดิม
ที่มี key `status` เพื่อ backward-compatible กับ contract ของ Part 92 หัวข้อ 92.1 แต่เพิ่ม key `database`
และเปลี่ยน handler ให้ query database จริง):

```rust
// src/routes/health.rs — ฉบับ Part 94
use axum::{extract::State, Json};
use serde_json::{json, Value};

use crate::state::AppState;

/// health check ที่ตรวจสถานะ DB จริง (ไม่ใช่แค่ "process ยังไม่ตาย") — ต่อยอด Part 92 หัวข้อ 92.1
/// (endpoint นี้ของ Part 92 คืนแค่ {"status":"ok"} เสมอ ไม่เช็ค DB — Part 94 ขยายให้ตรวจ DB จริงด้วย
/// `SELECT 1` ผ่าน pool แล้วคง JSON shape เดิมไว้ (status ยังเป็น "ok"/"error" ที่ backward compatible)
#[utoipa::path(
    get,
    path = "/health",
    tag = "health",
    responses((status = 200, description = "เซิร์ฟเวอร์และฐานข้อมูลพร้อมใช้งาน", body = serde_json::Value)),
)]
pub async fn health_check(State(state): State<AppState>) -> (axum::http::StatusCode, Json<Value>) {
    match sqlx::query_scalar!("SELECT 1").fetch_one(&state.db).await {
        Ok(_) => (
            axum::http::StatusCode::OK,
            Json(json!({ "status": "ok", "database": "ok" })),
        ),
        Err(e) => {
            tracing::error!(error = %e, "health check: database ไม่พร้อม");
            (
                axum::http::StatusCode::SERVICE_UNAVAILABLE,
                Json(json!({ "status": "error", "database": "unreachable" })),
            )
        }
    }
}
```

**สังเกตว่า route signature เปลี่ยนจาก `pub async fn health_check() -> Json<Value>` (Part 92) เป็น
`pub async fn health_check(State(state): State<AppState>) -> (StatusCode, Json<Value>)`** — ต้องรับ
`AppState` เพิ่มเข้ามาเพื่อเข้าถึง `PgPool`, และคืน `StatusCode` คู่กับ `Json` แทนการคืน `Json` เดี่ยว ๆ
(ตาม pattern เดียวกับ handler อื่น ๆ ของ Part 92 ที่คืน `(StatusCode, Json<T>)` เช่น `register`/
`create_book`) — เมื่อ `PgPool` unreachable, endpoint นี้ตอบ **`503 Service Unavailable`** (ไม่ใช่ `500`)
เพราะความหมายของ `503` ตรงกับสถานการณ์นี้ที่สุด: "บริการนี้ไม่พร้อมให้บริการ**ตอนนี้**ชั่วคราว" (ต่างจาก
`500` ที่หมายถึง "เกิด error ที่ไม่คาดคิดขณะประมวลผล request") — แพลตฟอร์ม deploy จริงส่วนใหญ่ (Kubernetes,
AWS ALB/ELB) treat `503` เป็นสัญญาณให้หยุดส่ง traffic เข้า instance นั้นชั่วคราวโดยอัตโนมัติ ตรงกับพฤติกรรม
readiness probe ที่ต้องการเป๊ะ

**พิสูจน์ path ที่ query สำเร็จ (DB ปกติ)**:

```bash
$ curl -sS -i http://127.0.0.1:8094/health
HTTP/1.1 200 OK
content-type: application/json
content-length: 31

{"database":"ok","status":"ok"}
```

**พิสูจน์ path ที่ query ล้มเหลว (จำลอง DB ไม่พร้อม โดยไม่ไปแตะ PostgreSQL cluster ที่ใช้ร่วมกันในสภาพ
แวดล้อมทดสอบ — ปิด `PgPool` เองแทนเพื่อจำลอง connection ที่หลุด/DB unreachable แล้วเรียก query เดียวกันกับ
ที่ handler ใช้จริง)**:

```rust
// โปรแกรมทดสอบแยก (ไม่ใช่ส่วนของ library_api) — จำลองสถานการณ์ DB down หลัง pool ถูกสร้างสำเร็จแล้ว
let db = PgPoolOptions::new().max_connections(2).connect(DATABASE_URL).await?;

let ok: Result<i32, _> = sqlx::query_scalar("SELECT 1").fetch_one(&db).await;
println!("while DB reachable: {:?}", ok);

db.close().await; // จำลอง DB connection หลุด
let err: Result<i32, _> = sqlx::query_scalar("SELECT 1").fetch_one(&db).await;
println!("after pool closed (simulating DB down): {:?}", err.is_err());
```

ผลลัพธ์จริง:

```
while DB reachable: Ok(1)
after pool closed (simulating DB down): true
error detail: attempted to acquire a connection on a closed pool
```

นี่คือ error variant เดียวกัน (`sqlx::Error`) ที่ handler จริงของ `health_check` จับด้วย `match ... Err(e)`
แล้วแปลงเป็น `503` — พิสูจน์ว่า branch ความล้มเหลวของ handler นี้ไม่ใช่โค้ดที่เขียนแล้วไม่เคยถูกเรียกจริง
(unreachable dead code) แต่เป็น branch ที่ทำงานได้จริงเมื่อ `PgPool` ไม่สามารถให้ connection ได้ไม่ว่าจาก
สาเหตุใดก็ตาม (connection หลุด, database ล่ม, network partition, หรือ pool หมดเพราะ query อื่นค้างอยู่)

**สิ่งที่บทนี้ตั้งใจไม่ทำ**: `/health` endpoint นี้ยัง**ไม่แยก** liveness (process ทำงานอยู่ไหม) ออกจาก
readiness (พร้อมรับ traffic ไหม) เป็นสอง endpoint ตามที่ระบบระดับใหญ่ (Kubernetes) มักทำ (`/livez` vs
`/readyz`) — สำหรับสโคปของ capstone นี้ endpoint เดียวที่ตรวจ DB ก็ครอบคลุมทั้งสองความหมายได้ดีพอ (ถ้า DB
ตายและ handler ยัง panic ไม่ได้ process ก็ยัง "live" อยู่ แต่ตอบ `503` ให้รู้ว่า "not ready") — การแยกสอง
endpoint เป็นหัวข้อดีสำหรับแบบฝึกหัดขยายผล (ดูแบบฝึกหัดข้อ 3)

**ทำไมสองความหมายนี้ (liveness/readiness) ควรแยกกันจริง ๆ ในระบบที่ใหญ่ขึ้น** — ต่อยอด Part 81 ให้ลึกขึ้น
อีกชั้น: ลองนึกภาพสองสถานการณ์ที่ต้องได้รับการตอบสนองจาก orchestration layer**ต่างกัน**:

- **สถานการณ์ A — database ล่มชั่วคราว (แต่ process ของเรายังทำงานปกติทุกอย่างอื่น)**: `/health` (ตามที่
  หัวข้อนี้ implement) ตอบ `503` ถูกต้อง — Kubernetes ควรตอบสนองด้วยการ**หยุดส่ง traffic เข้า instance นี้
  ชั่วคราว** (ผ่าน readiness probe) แต่**ไม่ควร restart container** เพราะ process เองไม่มีปัญหาอะไรเลย
  การ restart จะไม่ช่วยแก้ปัญหา database ที่ล่ม (instance ใหม่ก็ต่อ database เดิมที่ยังล่มอยู่เหมือนกัน) แถม
  ยังทำให้เสียเวลา restart โดยไม่ได้ประโยชน์อะไร
- **สถานการณ์ B — process เข้าสู่ deadlock/infinite loop ภายใน (เช่น bug ที่ทำให้ tokio task ค้างกันเองจน
  ไม่ตอบ request ไหนได้เลย แต่ TCP listener ยังเปิดอยู่)**: endpoint แบบเดียวที่หัวข้อนี้ implement (ที่ query
  database ก่อนตอบ) **อาจตอบไม่ทันเลยเพราะ handler เองก็ค้าง** — สถานการณ์นี้ต้องการ**liveness probe** ที่
  เบากว่ามาก (ไม่ query อะไรเลย แค่เช็คว่า HTTP server ยัง accept connection ได้) และเมื่อ liveness probe ล้ม
  เหลวซ้ำหลายครั้ง orchestration ควร**restart container ทันที** เพราะ restart คือทางแก้ที่ถูกต้องสำหรับ
  process ที่ค้างอยู่ภายใน (ต่างจากสถานการณ์ A ที่ restart ไม่ช่วยอะไรเลย)

สองสถานการณ์นี้ต้องการการตอบสนองที่ตรงข้ามกัน (readiness fail → หยุดส่ง traffic เฉย ๆ, liveness fail →
restart) — ถ้าใช้ endpoint เดียวกันทำทั้งสองหน้าที่ (แบบที่บทนี้ทำเพื่อความง่ายในสโคป capstone) ระบบจะ
**ตอบสนองผิดสถานการณ์ได้**: ถ้า Kubernetes เอา endpoint เดียวนี้ไปทำทั้ง liveness และ readiness probe และ
database ล่มชั่วคราวจริง (สถานการณ์ A) มันจะเห็นว่า liveness probe ล้มเหลวด้วย (เพราะ query database ค้าง/
error) แล้ว restart container ไปเรื่อย ๆ ทั้งที่ process ไม่มีปัญหาอะไรเลย และ database ก็ยังล่มเหมือนเดิม
หลัง restart — วนเป็น crash loop ที่ไม่ช่วยแก้ปัญหาอะไร ทำให้ downtime แย่ลงกว่าเดิม — นี่คือเหตุผลเชิงลึกที่
ระบบระดับใหญ่แยกทั้งสอง endpoint ออกจากกันเสมอ (ดูแบบฝึกหัดข้อ 3 สำหรับการ implement จริง)

**ตัวอย่างว่าแพลตฟอร์มอย่าง Kubernetes จะใช้ `/health` ของหัวข้อนี้อย่างไรจริง** (แค่ตัวอย่าง YAML config ที่
อ้าง endpoint นี้ ไม่ใช่การสอน Kubernetes เต็มรูปแบบ — Part 96 และเนื้อหา orchestration ในโมดูล 6 จะสอนกลไก
นี้ให้ลึกกว่านี้):

```yaml
# ตัวอย่างส่วนหนึ่งของ Pod spec — ใช้ /health ตัวเดียวเป็นทั้ง liveness และ readiness probe
# (สอดคล้องกับสโคปปัจจุบันของบทนี้ที่ยังไม่แยกสอง endpoint ตามที่อธิบายไว้ข้างบน)
readinessProbe:
  httpGet:
    path: /health
    port: 8092
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  httpGet:
    path: /health
    port: 8092
  periodSeconds: 10
  failureThreshold: 5
  initialDelaySeconds: 10
```

`readinessProbe` ที่ล้มเหลว (`/health` ตอบ `503`) ทำให้ Kubernetes ถอด Pod นี้ออกจากรายการ endpoint ของ
`Service` ชั่วคราว (ไม่มี traffic ใหม่ถูกส่งเข้ามาอีก จนกว่า probe จะกลับมาผ่าน) โดย**ไม่ restart container**
— ส่วน `livenessProbe` ที่ล้มเหลวซ้ำเกิน `failureThreshold` ครั้ง ทำให้ Kubernetes **restart container ทันที**
— ถ้าใช้ endpoint เดียวกันทำทั้งสองหน้าที่ (ตามตัวอย่างข้างบน) การที่ database ล่มชั่วคราวจะทำให้ Pod ถูก
restart วนซ้ำตามที่อธิบายไว้ข้างบนจริง ๆ นี่คือหลักฐานที่ชัดเจนว่าทำไมแบบฝึกหัดข้อ 3 (แยก `/health/live` ออก
จาก `/health/ready`) จึงสำคัญสำหรับระบบที่ deploy บน Kubernetes จริง ไม่ใช่แค่รายละเอียดเล็กน้อยที่มองข้ามได้

### 94.8 Logging และ Observability: Pretty vs JSON ด้วย `tracing`

Part 92 หัวข้อ 92.10 ตั้ง `tracing_subscriber::fmt().with_env_filter(...).init()` ไว้ตรง ๆ — ได้ log แบบ
pretty-printed (มีสี, จัดคอลัมน์ให้อ่านง่ายในเทอร์มินัล) เสมอ ซึ่งดีสำหรับตอน dev แต่ **ไม่เหมาะกับ
production** เพราะ log aggregator ของจริง (CloudWatch Logs, Datadog, Loki, Elasticsearch ฯลฯ) ต้องการ log
เป็น**โครงสร้างข้อมูลที่ parse ได้แน่นอน** (ปกติคือ JSON หนึ่ง object ต่อบรรทัด) เพื่อทำ filter/search/alert
ได้ — log แบบ pretty ที่มี ANSI color code ปนอยู่ parse ยากกว่ามากและบางระบบ parse ไม่ได้เลย

`tracing-subscriber` ที่ Part 92 ใช้อยู่แล้ว (feature `env-filter`) มี formatter สำหรับ JSON ให้พร้อมผ่าน
feature `json` — บทนี้เพียงแค่**สลับ formatter ตอน runtime ด้วย environment variable เดียว**
(`LOG_FORMAT`) โดยไม่แก้ logic การเรียก `tracing::info!`/`tracing::warn!`/`tracing::error!` ที่กระจายอยู่
ทั่วโค้ดของ Part 92 (`AppError::into_response`, handler ต่าง ๆ) แม้แต่จุดเดียว — นี่คือประโยชน์สำคัญของการ
ใช้ `tracing` เป็น facade ตั้งแต่แรก (ตามที่ Part 60 สอนไว้): **จุดที่เรียก log กับจุดที่ตัดสินใจว่า log จะ
ออกมาเป็นรูปแบบไหน แยกกันสนิท** เปลี่ยน formatter ที่จุดเดียว (`main.rs`) มีผลกับ log ทั้งแอปทันที

```rust
// ส่วนหนึ่งของ main.rs — ดูฉบับเต็มในหัวข้อ 94.4
let log_format = std::env::var("LOG_FORMAT").unwrap_or_else(|_| "pretty".to_string());
let env_filter = tracing_subscriber::EnvFilter::from_default_env();
if log_format == "json" {
    tracing_subscriber::fmt().json().with_env_filter(env_filter).init();
} else {
    tracing_subscriber::fmt().with_env_filter(env_filter).init();
}
```

(ต้องเพิ่ม feature `json` เข้า `tracing-subscriber` ใน `Cargo.toml`: `tracing-subscriber = { version =
"0.3.23", features = ["env-filter", "json"] }` — เพิ่มจาก dependency list เดิมของ Part 92 ที่มีแค่
`env-filter`)

**พิสูจน์ทั้งสองรูปแบบด้วยเซิร์ฟเวอร์จริงตัวเดียวกัน** (แค่เปลี่ยน `LOG_FORMAT`):

```bash
$ export LOG_FORMAT=json
$ export RUST_LOG=info
$ ./target/release/library_api
```
```json
{"timestamp":"2026-09-27T04:28:46.503269Z","level":"INFO","fields":{"message":"relation \"_sqlx_migrations\" already exists, skipping"},"target":"sqlx::postgres::notice"}
{"timestamp":"2026-09-27T04:28:46.504661Z","level":"INFO","fields":{"message":"รัน migration สำเร็จ (RUN_MIGRATIONS_ON_STARTUP=true)"},"target":"library_api"}
{"timestamp":"2026-09-27T04:28:46.505176Z","level":"INFO","fields":{"message":"library_api ฟังอยู่ที่ http://127.0.0.1:8096"},"target":"library_api"}
{"timestamp":"2026-09-27T04:28:46.505185Z","level":"INFO","fields":{"message":"Swagger UI: http://127.0.0.1:8096/swagger-ui"},"target":"library_api"}
```

```bash
$ unset LOG_FORMAT   # กลับไปใช้ค่า default "pretty"
$ ./target/release/library_api
```
```
2026-09-27T04:28:49.511229Z  INFO sqlx::postgres::notice: relation "_sqlx_migrations" already exists, skipping
2026-09-27T04:28:49.514072Z  INFO library_api: รัน migration สำเร็จ (RUN_MIGRATIONS_ON_STARTUP=true)
2026-09-27T04:28:49.514628Z  INFO library_api: library_api ฟังอยู่ที่ http://127.0.0.1:8096
2026-09-27T04:28:49.514636Z  INFO library_api: Swagger UI: http://127.0.0.1:8096/swagger-ui
```

(บรรทัด pretty ตัดสีออกเพื่อความอ่านง่ายในเอกสาร — ของจริงมี ANSI color code ทำให้ level `INFO` เป็นสีเขียว
และ target เป็นสีเทาอ่อน)

log แต่ละบรรทัดของ JSON format เป็น **object เดียวสมบูรณ์** พร้อม key `timestamp`/`level`/`fields.message`/
`target` — log aggregator ที่ config ให้ parse JSON log (ตั้งค่ามาตรฐานของทุกระบบที่กล่าวถึงข้างบน) จะแยก
field เหล่านี้ออกมาเป็นคอลัมน์ที่ query/filter ได้ทันที (เช่น "หา log ทั้งหมดที่ `level = ERROR` และ
`target = library_api::error`" — ตรงกับ field `error_code`/`status` ที่ `AppError::into_response` ของ
Part 92 หัวข้อ 92.4 ใส่ไว้ใน `tracing::warn!`/`tracing::error!` อยู่แล้วโดยไม่ต้องแก้อะไรเพิ่มเลย เพราะ
`tracing`'s structured fields (`error_code = code, status = status.as_u16()`) จะกลายเป็น key ใน JSON
object โดยอัตโนมัติเช่นกัน)

**พิสูจน์ด้วย request ที่ล้มเหลวจริง** — ยิง `POST /api/v1/auth/register` ด้วย username ซ้ำสองครั้ง (ชน
unique constraint ตามที่ Part 92 หัวข้อ 92.3/92.4 ออกแบบไว้ให้กลายเป็น `AppError::Conflict`) ตอน
`LOG_FORMAT=json`:

```json
{"timestamp":"2026-09-27T05:05:07.150407Z","level":"WARN","fields":{"message":"request ล้มเหลวด้วย client error","error_code":"CONFLICT","status":409,"error":"ขัดแย้งกับสถานะปัจจุบัน: ข้อมูลนี้ซ้ำกับที่มีอยู่แล้วในระบบ"},"target":"library_api::error"}
```

สังเกตว่า `error_code`/`status`/`error` (ค่าที่ `into_response()` ของ `AppError` ใน Part 92 หัวข้อ 92.4
ส่งเข้า `tracing::warn!(error_code = code, status = status.as_u16(), error = %other, ...)` ตรง ๆ) กลายเป็น
**key แยกในระดับเดียวกับ `message`** ภายใน `fields` object — log aggregator ที่ตั้ง alert ไว้ เช่น "แจ้งเตือน
ถ้ามี log ที่ `fields.status >= 500` เกิน 10 ครั้งใน 1 นาที" จะทำงานได้ตรงเป้าทันที เพราะ field พวกนี้ query
ได้เหมือน column ของตารางฐานข้อมูล ไม่ต้องเขียน regex ไปแยกมันออกจากข้อความ log แบบ pretty ที่เป็น string
ยาว ๆ ก้อนเดียว — นี่คือประโยชน์ที่จับต้องได้จริงของการสลับไปใช้ `LOG_FORMAT=json` ใน production ไม่ใช่แค่
เรื่องความสวยงามของ format

### 94.9 การ Deploy จริง: สิ่งที่ Verify ได้จริงในสภาพแวดล้อมนี้ (และสิ่งที่ไม่ได้)

หัวข้อนี้ต้องพูดตรง ๆ ตามที่หมายเหตุต้นบทสัญญาไว้ — **สภาพแวดล้อมแบบ sandbox ที่ใช้พัฒนาหลักสูตรนี้มีนโยบาย
เครือข่ายที่จำกัดการเข้าถึงบางโดเมน** (deb.debian.org และ GitHub Releases ถูกปฏิเสธที่ระดับ network policy
ของสภาพแวดล้อมนี้โดยเฉพาะ ไม่ใช่ข้อจำกัดของ Docker เอง) ทำให้การ build image ตาม Dockerfile รุ่นแรกที่ยัง
ใช้ `apt-get install` (ก่อนปรับมาเป็นเวอร์ชันสุดท้ายของหัวข้อ 94.5) ติดปัญหา:

```bash
$ docker build -t library_app:test .
...
#8 0.195 Err:1 http://deb.debian.org/debian bookworm InRelease
#8 0.195   403  Forbidden [IP: 151.101.66.132 80]
...
E: Failed to fetch http://deb.debian.org/debian/dists/bookworm/InRelease  403  Forbidden
```

**นี่คือเหตุผลที่ Dockerfile ฉบับสุดท้ายของหัวข้อ 94.5 หลีกเลี่ยง `apt-get install` ในทุก stage โดยตั้งใจ**
(ติดตั้ง `trunk` ผ่าน `cargo install` จาก crates.io แทนการดาวน์โหลด binary จาก GitHub Releases, และ runtime
image ไม่ต้อง package เพิ่มเลยเพราะ binary ไม่มี dependency ต่อ `libssl`/`ca-certificates`) — เมื่อปรับตาม
นี้แล้ว **ทุก stage ของ Dockerfile build ผ่านจริงในสภาพแวดล้อมนี้** เพราะ `crates.io`/`index.crates.io`
(ที่ `cargo`/`rustup` ต้องเข้าถึง) ไม่ได้ถูกปฏิเสธเหมือน `deb.debian.org` — **และ build จบสำเร็จจริงเป็น
image ที่รันได้จริง ไม่ใช่แค่ build ผ่านแต่ยังไม่ได้ลองรัน**:

```bash
$ docker build --network=host --build-arg https_proxy=$PROXY_URL -f Dockerfile . -t library_app:part94-verify
#26 [frontend-builder 8/8] RUN trunk build --release
#26 56.23    Compiling library_frontend v0.1.0 (/app/library_frontend)
#26 63.00     Finished `release` profile [optimized] target(s) in 1m 00s
#26 63.17 2026-09-27T04:50:01.641722Z  INFO downloading wasm-bindgen version="0.2.129"
#26 64.49 2026-09-27T04:50:02.960367Z  INFO success
#26 DONE 64.9s
#27 [runtime 6/6] COPY --from=frontend-builder /app/library_frontend/dist ./static
#27 DONE 0.0s
#28 exporting to image
#28 naming to docker.io/library/library_app:part94-verify done
#28 DONE 0.8s

$ docker image ls library_app
IMAGE                       ID             DISK USAGE   CONTENT SIZE
library_app:part94-verify   e1bbb3f07ce8        137MB         33.5MB
```

(คำสั่งข้างบนมี `--network=host --build-arg https_proxy=...` เพิ่มเข้ามาเพื่อเจาะผ่าน policy-enforcing
proxy **ที่มีอยู่เฉพาะในสภาพแวดล้อมของผู้เขียนบทนี้เท่านั้น** — ผู้อ่านที่ build Dockerfile นี้ในเครื่อง/CI
ของตัวเองที่มี network เข้าถึง crates.io ตามปกติ **ไม่ต้องมีเงื่อนไขพิเศษเหล่านี้เลย** ใช้ `docker build -t
library_app:latest .` ตรง ๆ ตามหัวข้อ 94.5 ได้ทันที — ปัญหานี้เป็นข้อจำกัดของสภาพแวดล้อมที่ใช้ตรวจสอบบทนี้
เท่านั้น ไม่ใช่ข้อจำกัดของ Docker หรือของ Dockerfile เอง)

**image ที่ build ได้ถูกรันจริงเป็น container จริง ต่อกับ PostgreSQL จริงบนเครื่อง (ผ่าน `--network=host`
เพื่อให้ container คุยกับ Postgres ที่ฟังอยู่ที่ `127.0.0.1:5432` ของ host ได้ — ในการ deploy จริงที่ไม่ได้
อยู่ในสภาพแวดล้อมทดสอบนี้ ปกติจะใช้ Docker network ปกติ/`docker-compose` เชื่อม container กับ container
database แทน ไม่ต้อง `--network=host`)**:

```bash
$ docker run -d --name library_app_verify --network=host \
    -e DATABASE_URL="postgres://postgres:postgres@127.0.0.1:5432/library_api_p94" \
    -e JWT_SECRET="docker-verify-secret" \
    -e BIND_ADDR="0.0.0.0:8098" \
    -e RUST_LOG="info" \
    library_app:part94-verify

$ docker logs library_app_verify
INFO sqlx::postgres::notice: relation "_sqlx_migrations" already exists, skipping
INFO library_api: รัน migration สำเร็จ (RUN_MIGRATIONS_ON_STARTUP=true)
INFO library_api: library_api ฟังอยู่ที่ http://0.0.0.0:8098
INFO library_api: Swagger UI: http://0.0.0.0:8098/swagger-ui
INFO library_api: serving frontend static files จาก /app/static
```

container รันขึ้นมาจริง, ต่อ PostgreSQL จริงสำเร็จ, รัน migration สำเร็จ (พบว่ามีอยู่แล้วจากการทดสอบก่อนหน้า
ในหัวข้อนี้ — `sqlx migrate run` idempotent ตามที่ Part 71 อธิบายไว้), ฟังที่ `0.0.0.0:8098` ตามที่กับดักข้อ
1 อธิบายไว้ว่าต้องเป็น `0.0.0.0` ไม่ใช่ `127.0.0.1` และ serve ไฟล์ static จาก `/app/static` ที่ COPY มาจาก
`frontend-builder` stage จริง — พิสูจน์ด้วย `curl` ตรงไปที่ container นี้เลย:

```bash
$ curl -sS -i http://127.0.0.1:8098/health
HTTP/1.1 200 OK
content-type: application/json

{"database":"ok","status":"ok"}
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8098/
200
$ curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8098/my-borrowed
200
$ curl -sS -i http://127.0.0.1:8098/health | grep -i access-control || echo "(no CORS headers -- correct)"
(no CORS headers -- correct)
$ curl -sS -X POST http://127.0.0.1:8098/api/v1/auth/register -H 'Content-Type: application/json' \
    -d '{"username":"dockeruser","email":"dockeruser@example.com","password":"password123"}'
{"id":4,"username":"dockeruser","email":"dockeruser@example.com","role":"member","created_at":"2026-09-27T04:53:28.157518Z"}
$ curl -sS -X POST http://127.0.0.1:8098/api/v1/auth/login -H 'Content-Type: application/json' \
    -d '{"username":"dockeruser","password":"password123"}'
{"access_token":"eyJ0eXAi...","token_type":"Bearer","expires_in":3600,"user":{"id":4,...}}
```

และเปิด **headless Chromium จริงผ่าน Playwright ไปที่ container ที่กำลังรันอยู่จริงนี้** (`http://
127.0.0.1:8098/` — ไม่ใช่ binary ที่รันตรงบนเครื่องแบบหัวข้อก่อนหน้าอีกต่อไป แต่เป็น container จริง 100%):

```
=== goto / (served from combined server, same origin) ===
current URL: http://127.0.0.1:8098/
=== register / login / books list / deep-link / hard reload / logout+redirect ===
whoami: สวัสดี combouser361507 (member)
URL after hard navigate: http://127.0.0.1:8098/my-borrowed
whoami after reload: สวัสดี combouser361507 (member)
URL after logged-out deep link: http://127.0.0.1:8098/login
=== summary ===
pageerrors count: 0 []
api responses observed: 9
   201 http://127.0.0.1:8098/api/v1/auth/register
   200 http://127.0.0.1:8098/api/v1/auth/login
   200 http://127.0.0.1:8098/api/v1/me
   ...
E2E OK
```

**ดังนั้นข้อสรุปที่ตรงกับความเป็นจริงที่สุดคือ**: Docker image ของบทนี้ **build ผ่านจริง 100% และรันเป็น
container จริงได้ 100%** ในสภาพแวดล้อมนี้ — สิ่งที่ต้องมีเงื่อนไขพิเศษ (proxy/`--network=host`) มีแค่**ขั้น
ตอนเจาะผ่าน network policy ของสภาพแวดล้อม sandbox นี้เท่านั้น** (ซึ่งไม่มีในเครื่อง dev/CI ปกติของผู้อ่าน) —
ทั้ง `docker build`, `docker run`, และการยิงทั้ง `curl`/headless Chromium เข้าไปที่ container ที่รันจริงคือ
สิ่งที่ verify แล้วจริงในบทนี้ ไม่ใช่แค่คำอธิบายเชิงทฤษฎี

**สิ่งที่ verify ได้จริง 100% นอกเหนือจากการรัน container ข้างบน (รัน binary ตรงบนเครื่องแบบไม่ผ่าน Docker
เลย เพื่อแยกให้เห็นชัดว่าผลลัพธ์เหมือนกันไม่ว่าจะห่อด้วย container หรือไม่ก็ตาม เพราะ image ก็แค่ห่อ binary
+ ไฟล์ static ชุดเดียวกันนี้)**:

1. Build backend release binary จริง (`cargo build --release`) และ frontend production bundle จริง
   (`trunk build --release`) — ทั้งสองสำเร็จ ไม่มี error ใด ๆ (ผลลัพธ์เต็มอยู่ในหัวข้อ 94.2)
2. รัน binary เดียวที่ serve ทั้ง API และไฟล์ static ผ่าน `ServeDir`/`ServeFile` ของหัวข้อ 94.3 จริงที่
   `http://127.0.0.1:8094` (พอร์ตเดียว ไม่มี process ที่สอง ไม่มี CORS header ใด ๆ)
3. ยิง `curl` จริงทำ full user journey ระดับ API: register → login → (promote เป็น admin ผ่าน SQL ตรงตาม
   ที่ Part 92 หัวข้อ 92.8 ออกแบบไว้ว่าเป็นทางเดียวที่ตั้งใจ) → สร้างหนังสือ → ดูรายการ → ยืม → ดูที่ยืมอยู่
   → คืน — ทุกขั้นตอนสำเร็จตรงตาม contract ของ Part 92 หัวข้อ 92.1 เป๊ะ:

```bash
$ curl -sS -X POST $BASE/api/v1/auth/register -d '{"username":"combo1", ...}'
{"id":1,"username":"combo1","email":"combo1@example.com","role":"member",...}
$ curl -sS -X POST $BASE/api/v1/auth/login -d '{"username":"combo1", ...}'
{"access_token":"eyJ0eXAi...","token_type":"Bearer","expires_in":3600,...}
$ curl -sS -X POST $BASE/api/v1/books -H "Authorization: Bearer $TOKEN" -d '{"title":"Combined Serving Test Book",...}'
{"id":1,"title":"Combined Serving Test Book",...,"available_copies":2,...}
$ curl -sS -X POST "$BASE/api/v1/books/1/borrow" -H "Authorization: Bearer $TOKEN"
{"id":1,"book_id":1,"user_id":1,"borrowed_at":"...","due_at":"...","returned_at":null}
$ curl -sS "$BASE/api/v1/me/borrowed" -H "Authorization: Bearer $TOKEN"
{"data":[{"id":1,"book_id":1,"user_id":1,...,"returned_at":null}]}
$ curl -sS -X POST "$BASE/api/v1/books/1/return" -H "Authorization: Bearer $TOKEN"
{"id":1,"book_id":1,"user_id":1,...,"returned_at":"2026-09-27T04:25:44.242557Z"}
$ curl -sS "$BASE/api/v1/me/borrowed" -H "Authorization: Bearer $TOKEN"
{"data":[]}
```

4. เปิด **headless Chromium จริงผ่าน Playwright** (เครื่องมือเดียวกับที่ Part 93 หัวข้อ 93.11 ใช้) ไปที่
   `http://127.0.0.1:8094/` ตรง ๆ (origin เดียวกับ API) แล้วขับ flow เต็มรูปแบบผ่าน UI จริง (กรอกฟอร์มจริง,
   คลิกปุ่มจริง ไม่ใช่แค่เรียก API ตรง ๆ):

```
=== goto / (served from combined server, same origin) ===
current URL: http://127.0.0.1:8094/
nav content: หนังสือ | ที่ยืมอยู่ | เข้าสู่ระบบ | สมัครสมาชิก
=== register ===
after register -> URL: http://127.0.0.1:8094/login
=== login ===
whoami: สวัสดี combouser491956 (member)
=== localStorage session present ===
localStorage has session: true
=== books list (same-origin API call via relative API_BASE) ===
books list text: Combined Serving Test Book — Part94 (2/2 ว่าง)ยืม
=== deep-link hard navigation to /my-borrowed (SPA fallback test) ===
URL after hard navigate: http://127.0.0.1:8094/my-borrowed
page heading: หนังสือที่ยืมอยู่
=== hard reload session persistence ===
whoami after reload: สวัสดี combouser491956 (member)
=== logout, then hard navigate to /my-borrowed -> should redirect to /login ===
URL after logged-out deep link: http://127.0.0.1:8094/login
=== summary ===
pageerrors count: 0 []
api responses observed: 9
    200 http://127.0.0.1:8094/api/v1/books?page=1&per_page=5
    201 http://127.0.0.1:8094/api/v1/auth/register
    200 http://127.0.0.1:8094/api/v1/auth/login
    200 http://127.0.0.1:8094/api/v1/me
    200 http://127.0.0.1:8094/api/v1/books?page=1&per_page=5
    200 http://127.0.0.1:8094/api/v1/me
    200 http://127.0.0.1:8094/api/v1/me/borrowed
    200 http://127.0.0.1:8094/api/v1/me/borrowed
    200 http://127.0.0.1:8094/api/v1/me
E2E OK
```

สังเกตว่า**ทุก network request ที่ Playwright ดักจับได้เป็น URL เดียวกันหมด** (`http://127.0.0.1:8094/...`)
— ไม่มีการแยก origin เป็น `5173`/`8092` แบบที่ Part 93 หัวข้อ 93.11 เจอเลย เพราะ frontend เรียก API ด้วย
relative path (`API_BASE = ""` ตามที่หัวข้อ 94.2 ตั้งไว้ตอน build) แล้ว browser เติม origin ปัจจุบันให้
อัตโนมัติ — และ `pageerrors count: 0` ยืนยันว่าไม่มี WASM panic หรือ unhandled JavaScript exception เกิดขึ้น
เลยตลอด flow

5. พิสูจน์ปุ่ม "ยืม"/"คืนหนังสือ" ที่คลิกจริงผ่าน UI (ไม่ใช่แค่เรียก API ตรง ๆ ผ่าน `curl`) ให้ผลตรงกับที่
   คาดไว้:

```
=== click ยืม button on books list (browser-driven borrow, not curl) ===
borrow feedback: ยืมสำเร็จ! กำหนดคืนวันที่ 2026-10-11T04:27:11.059556Z
books list after borrow: Combined Serving Test Book — Part94 (1/2 ว่าง)ยืม
=== go to /my-borrowed and click คืนหนังสือ (browser-driven return) ===
my-borrowed page after return: หนังสือที่ยืมอยู่

ไม่มีหนังสือที่ยืมอยู่ตอนนี้

คืนหนังสือสำเร็จ
pageerrors: 0 []
BORROW/RETURN E2E OK
```

**สรุปตรง ๆ ว่าอะไร verify แล้วจริง อะไรยัง**: การรัน **binary/container ที่รวมกันแล้วเป็นหนึ่งหน่วยเดียว
ในเครื่อง (local)** — ทั้งการ build, การ serve แบบ single-origin, migration, health check, log format, และ
full user journey ทั้งระดับ API และระดับเบราว์เซอร์ — **verify แล้วจริงทั้งหมด 100%** ด้วยผลลัพธ์ที่แสดงไว้
ข้างบน ส่วนการ deploy ไปยัง cloud platform จริง (AWS/GCP/Azure/Fly.io ฯลฯ) **ไม่ได้ทำในบทนี้** เพราะ
สภาพแวดล้อมแบบ sandbox ที่ใช้เขียนหลักสูตรนี้ไม่มีบัญชี cloud provider ให้เชื่อมต่อ และนอกสโคปของ capstone
ที่ Part 92 หัวข้อ 92.1 วางไว้ (ระบบห้องสมุด/ยืม-คืนหนังสือ ไม่ใช่หลักสูตรเรื่อง cloud infrastructure) —
"the containerized app runs correctly as one unit locally" คือคำกล่าวที่ตรงกับความเป็นจริงที่สุดสำหรับ
ระดับของการตรวจสอบในบทนี้ ส่วนกลไก Docker เอง (image layer, registry, orchestration) จะถูกอธิบายลึกกว่านี้
ใน Part 96 ที่มาถัดไป

**ระยะห่างที่เหลือระหว่าง "container ที่รันได้ในเครื่อง" กับ "แอปที่ deploy ไปยัง cloud จริง" นั้นสั้นกว่าที่
คิด** — ควรพูดตรง ๆ ไว้ด้วยว่าทำไม: image ที่ build เสร็จตามหัวข้อ 94.5 (`docker build -t
library_app:latest .`) คือ artifact เดียวกันเป๊ะที่แพลตฟอร์ม deploy สมัยใหม่จำนวนมาก (เช่น Fly.io, Railway,
Render, Google Cloud Run, AWS App Runner) รับตรง ๆ ผ่านคำสั่งเดียว (เช่น `fly deploy` หรือ `gcloud run
deploy` ที่อ่าน `Dockerfile` ในโฟลเดอร์เดียวกันแล้ว build+push+deploy ให้อัตโนมัติ) — สิ่งที่ต้องทำเพิ่มจาก
บทนี้จริง ๆ มีแค่: (1) push image ไปยัง container registry ที่แพลตฟอร์มนั้นเข้าถึงได้ (2) ตั้งค่า
environment variable ของหัวข้อ 94.4 ผ่านหน้าตั้งค่าของแพลตฟอร์มนั้น (แทน `.env`/`docker run -e`) และ (3) ชี้
`DATABASE_URL` ไปยัง PostgreSQL instance จริงที่แพลตฟอร์มนั้นจัดการให้ (managed database) — ไม่มีขั้นตอนไหน
ที่ต้องเขียนโค้ดเพิ่มเลยแม้แต่บรรทัดเดียว เพราะบทนี้ออกแบบ config ทั้งหมดให้เป็น environment-driven ไว้ตั้งแต่
หัวข้อ 94.4 แล้ว — นี่คือเหตุผลที่กล่าวได้อย่างมั่นใจว่า container image ของบทนี้ "deploy ได้จริง" แม้จะไม่ได้
ทำขั้นตอน deploy จริงในสภาพแวดล้อมทดสอบนี้ก็ตาม (ต่างจากการอ้างแบบทฤษฎีลอย ๆ ที่ไม่มีอะไรรองรับ)

### 94.10 มองย้อนกลับ: จบ Capstone 3 บท และเชื่อมสู่ Module 6

ถึงจุดนี้ ทั้งสามบท (Part 92 → 93 → 94) ประกอบกันเป็นระบบเดียวที่ครบวงจรตั้งแต่ฐานข้อมูลไปจนถึงสิ่งที่ deploy
ได้จริง — ควรมองย้อนกลับไปดูภาพรวมทั้งหมดก่อนไปต่อ:

**สิ่งที่คุณได้สร้างขึ้นมาตลอด 3 บทนี้**:

- **Part 92**: backend Axum ที่จัดโครงสร้างเป็น module อย่างมีระบบ, มี PostgreSQL schema ที่ป้องกัน race
  condition ด้วย constraint + transaction (พิสูจน์ด้วยการยิง 10 request พร้อมกันจริง), มี authentication
  (JWT + argon2) และ authorization (RBAC) ที่ครบถ้วน, มี error handling แบบรวมศูนย์ที่เลือก HTTP status
  ถูกต้องตามความหมาย, มีเอกสาร OpenAPI ที่ตรงกับโค้ด 100%, และมี integration test suite ที่ผ่านทั้งหมด
- **Part 93**: frontend Leptos CSR ที่คุยกับ backend ผ่าน REST API ล้วน ๆ (ไม่ใช่ server function) ตรงกับ
  สถานการณ์ทีม frontend/backend แยกกันในโลกทำงานจริง, จัดการ session ด้วย signal + `localStorage`, มี
  routing หลายหน้าพร้อม route guard, แสดง error จริงจาก backend ทั้งแบบรวมและแบบแยก field, และพิสูจน์ flow
  เต็มรูปแบบด้วย headless Chromium จริง
- **Part 94 (บทนี้)**: ปิดช่องว่างระหว่างสองบทข้างบนกับความเป็นจริงของการ deploy — production build ของ
  ทั้งสองฝั่ง, การตัดสินใจสถาปัตยกรรม serving ที่มีเหตุผลรองรับ, environment-driven config ที่ปลอดภัย,
  container image ที่ deploy ได้จริง, migration strategy ที่ปลอดภัยสำหรับ multi-instance, health check ที่
  สะท้อนสถานะจริง, และ log ที่ระบบ production อ่านได้

**ตัวเลขจริงที่พิสูจน์แล้วตลอดบทนี้** (สรุปรวมจากทุกหัวข้อ เพื่อให้เห็นภาพรวมเป็นรูปธรรม ไม่ใช่แค่คำอธิบาย
เชิงทฤษฎี):

| รายการ | ค่าที่วัดได้จริง |
|---|---|
| backend dev build (`cargo build`) | 119MB, ไม่ optimize |
| backend release build (`cargo build --release`) | 14MB (~8.5 เท่าเล็กกว่า dev build) |
| frontend dev build (`trunk build`) — ไฟล์ `.wasm` | 5.8MB |
| frontend release build (`trunk build --release`) — ไฟล์ `.wasm` | 1.5MB (~4 เท่าเล็กกว่า dev build) |
| ไฟล์ static ทั้งหมดของ frontend release (`index.html`+`.js`+`.wasm`) | ~1.5MB รวม |
| Docker image สุดท้าย (multi-stage, ไม่มี Rust toolchain ติดไปด้วย) | 137MB (disk usage), 33.5MB (content/compressed) |
| จำนวน endpoint ที่ยังทำงานถูกต้องหลัง refactor เป็น single-origin | 10/10 ตาม contract ของ Part 92 หัวข้อ 92.1 |
| integration test ของ Part 92 ที่ยังผ่านหลังแก้ `build_app` signature | 5/5 (ผ่าน `build_app_dev` alias) |
| `pageerrors` ที่พบระหว่าง headless Chromium E2E ทั้งสองรอบ (binary ตรง + container จริง) | 0 |

**นี่คือภาพรวมของทั้ง Module 4-5 ที่มาบรรจบกัน**: Module 4 สอนเทคนิคของ Axum backend ทีละชิ้น (HTTP
fundamentals, middleware, error handling, database, auth, authorization, REST design, OpenAPI docs) และ
Module 5 สอนเทคนิคของ frontend framework ต่าง ๆ (Yew, Dioxus, Leptos, SSR, WASM) — Part 92-93-94 คือจุดที่
พิสูจน์ว่าเทคนิคเหล่านั้น**ไม่ใช่ความรู้แยกส่วนที่ท่องจำไว้เฉย ๆ** แต่ประกอบกันเป็นระบบเดียวที่ทำงานได้จริง
ตั้งแต่ request แรกที่ผู้ใช้ยิงเข้ามา ไปจนถึง response ที่ตอบกลับ ผ่านทุกชั้น (routing, auth, business
logic, database, error handling) และจบด้วยการที่ระบบทั้งก้อนนั้น**รันเป็นหน่วยเดียวที่ deploy ได้จริง**
ไม่ใช่แค่ "รันได้ในเครื่องผู้เขียนคนเดียว"

**สิ่งที่ตั้งใจไม่ทำใน 3 บทนี้ (deferred โดยตั้งใจ ไม่ใช่ลืม)**: refresh token ที่ revoke ได้จริง (Part 92
แบบฝึกหัดข้อ 3), ระบบจองคิว/waitlist (Part 92 แบบฝึกหัดข้อ 4), การ sync auth state ข้ามแท็บ (Part 93
แบบฝึกหัดข้อ 4), การแยก liveness/readiness เป็นสอง endpoint (แบบฝึกหัดข้อ 3 ของบทนี้), และการ deploy จริงไป
ยัง cloud platform (หัวข้อ 94.9) — ทั้งหมดนี้เป็นหัวข้อขยายผลที่ดีสำหรับผู้อ่านที่อยากต่อยอด capstone นี้ให้
ใกล้เคียงระบบ production เต็มรูปแบบมากขึ้น

**ทางไปข้างหน้า**: Part 95 (Testing Web Applications แบบครบวงจร) ซึ่งเขียนไว้**ก่อน**ที่ Part 92-94 จะมีอยู่
จริง เลยใช้แอปตัวแทน (representative stand-in app) แทน capstone นี้ในการสอนเทคนิค unit/integration/e2e
testing — เนื้อหาเรื่อง testing นั้นยังใช้ได้ตรงกับ capstone นี้ทุกประการ (integration test suite ของ Part
92 หัวข้อ 92.11 และ headless Chromium test ของ Part 93/94 ก็คือ e2e testing ในทางปฏิบัติอยู่แล้ว) เพียงแต่
Part 95 จะลงรายละเอียดเรื่อง testing เองมากกว่าที่ capstone 3 บทนี้ทำ — ถือเป็นการทวนเทคนิคที่ 3 บทนี้ใช้ไป
แล้วให้ลึกขึ้นอีกชั้น จากนั้น **Module 6 (Production/DevOps)** จะเริ่มด้วย **Part 96 (Docker)** ที่ลงราย
ละเอียดกลไกของ container/image ให้ลึกกว่าที่บทนี้แนะนำไว้แบบผ่าน ๆ (layer caching, multi-arch build,
registry, orchestration พื้นฐาน) — Dockerfile ที่หัวข้อ 94.5 ให้ไว้คือตัวอย่างที่เป็นรูปธรรมที่สุดที่จะทำให้
เนื้อหาของ Part 96 เข้าใจง่ายขึ้น เพราะผู้อ่านได้เห็นแล้วว่า Docker แก้ปัญหาอะไรจริง ๆ (การแพ็กแอปที่ต้อง
build จากสองภาษา/สอง toolchain [Rust ปกติ + Rust-to-WASM] ให้กลายเป็น artifact เดียวที่ deploy ได้ที่ไหนก็ได้
ที่มี Docker runtime) ก่อนจะเรียนกลไกเบื้องหลังของมันต่อไป

**ข้อคิดปิดท้ายสำหรับทั้ง 3 บท**: ไม่มีจุดใดใน Part 92-93-94 ที่ต้อง "เดา" ว่าโค้ดทำงานถูกต้องหรือไม่ — ทุกการ
ตัดสินใจเชิงสถาปัตยกรรม (race condition ของการยืมหนังสือใน Part 92, `Send` bound ของ `Action`/`Resource` ใน
Part 93, สถานะ HTTP ของ SPA fallback ในบทนี้) ถูกพิสูจน์ด้วยการรันจริงและอ่านผลลัพธ์จริงเสมอ ไม่ใช่อนุมานจาก
เอกสารหรือความคุ้นเคยเพียงอย่างเดียว — นี่คือทักษะที่สำคัญไม่น้อยไปกว่าเทคนิคของ Rust/Axum/Leptos เอง: **นิสัย
การพิสูจน์ทุกสมมติฐานด้วยหลักฐานจริงก่อนเชื่อว่ามันถูกต้อง** เป็นทักษะที่ใช้ได้กับทุกภาษา ทุกเฟรมเวิร์ก และ
จะยังจำเป็นต่อไปไม่ว่าเทคโนโลยีจะเปลี่ยนไปอย่างไรในอนาคต

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `BIND_ADDR=127.0.0.1:8092` ใน container ทำให้เชื่อมต่อจากนอก container ไม่ได้เลย

ระหว่างพัฒนาบทนี้ ตอนทดสอบรัน binary ด้วยค่า `BIND_ADDR` เดิมของ Part 92 dev (`127.0.0.1:8092`) ในบริบทที่
จำลอง container (network namespace แยกจาก host) แล้วพยายาม `curl` เข้ามาจาก host โดยตรง (เหมือนที่จะเกิด
จริงถ้า deploy ผ่าน container ที่ map port ออกมา) — ได้ error ทันที:

```
curl: (7) Failed to connect to <container-ip> port 8092 after 0 ms: Connection refused
```

**สาเหตุ**: `127.0.0.1` (loopback) หมายถึง "เฉพาะ traffic ที่มาจากภายใน network namespace เดียวกันเท่านั้น"
— เมื่อ process รันอยู่ *ภายใน* container, `127.0.0.1` ของมันคือ loopback ของ container เอง ไม่ใช่ loopback
ของ host เครื่องแม่ — traffic จากนอก container (แม้จะ mapping port ถูกต้องแล้วก็ตาม เช่น `-p 8092:8092`) มา
ถึง network interface ของ container ผ่าน bridge network ไม่ใช่ loopback จึงถูกปฏิเสธทันทีเพราะไม่มีอะไร
ฟังอยู่ที่ interface นั้นเลย

**วิธีแก้**: ตั้ง `BIND_ADDR=0.0.0.0:8092` ใน container เสมอ (`0.0.0.0` หมายถึง "ฟังทุก network interface
ที่มีอยู่") — ตรงกับที่ Dockerfile ของหัวข้อ 94.5 ตั้ง `ENV BIND_ADDR=0.0.0.0:8092` ไว้เป็นค่า default ของ
runtime image อยู่แล้ว — สำหรับ dev บนเครื่องเดียว (ไม่ใช่ container) ยังใช้ `127.0.0.1` ต่อไปได้ตามปกติ
(ปลอดภัยกว่าเพราะไม่มีใครนอกเครื่องเข้าถึงได้โดยไม่ตั้งใจ) กับดักนี้เกิดเฉพาะตอนย้ายไปรันใน container/VM
ที่แยก network namespace เท่านั้น

### 2. `ServeDir::not_found_service` ตอบ SPA fallback ด้วย status `404` เสมอ ไม่ใช่ `200`

ระหว่างพัฒนาหัวข้อ 94.3 การเขียนตามตัวอย่างในเอกสารของ `tower-http` เองตรง ๆ
(`ServeDir::new(dir).not_found_service(ServeFile::new(index_path))` — เอกสารของ crate เองระบุว่า "Setups
like this are often found in single page applications") ทำให้ `cargo build` ผ่านสนิทและ SPA fallback
"ดูเหมือน" ทำงาน (browser ยังเห็นเนื้อหาของ `index.html` ตอน deep-link) แต่ตรวจด้วย `curl -i` จริงพบว่า
status code ผิด:

```bash
$ curl -sS -i http://127.0.0.1:8094/my-borrowed
HTTP/1.1 404 Not Found
content-type: text/html
...
<!DOCTYPE html>
<html lang="th">
  ...
```

**สาเหตุ**: อ่าน source code ของ `tower-http` 0.7.1 ตรง ๆ (`services/fs/serve_dir/mod.rs`) พบว่า
`.not_found_service(fallback)` implement ด้วย `self.fallback(SetStatus::new(fallback,
StatusCode::NOT_FOUND))` — มัน**บังคับ**สถานะเป็น `404` เสมอไม่ว่า `fallback` เองจะตอบสถานะอะไรก็ตาม (ออก
แบบมาสำหรับหน้า "404 not found" ที่ตั้งใจให้เป็น `404` จริง ๆ ซึ่งเป็นการใช้งานที่ถูกต้องเช่นกัน เพียงแต่ผิด
จุดสำหรับ SPA fallback) แม้ผลลัพธ์ทางสายตาใน browser จะดูเหมือนใช้ได้ (browser ยัง render เนื้อหาของ
`index.html` แม้ status จะเป็น 404 เพราะ document navigation ส่วนใหญ่ไม่ได้เช็ค status code ก่อน render)
แต่นี่คือพฤติกรรมที่ผิดในทางเทคนิค: **บาง infrastructure (health check, monitoring, search engine crawler,
CDN cache rule ที่อิงสถานะ) จะเห็นว่าทุกหน้าของ SPA "ไม่พบ" (404) ทั้งที่มันแสดงผลได้ปกติ**

**วิธีแก้**: ใช้ `.fallback(ServeFile::new(index_path))` (ไม่มี `not_found_`) แทน — เมธอดนี้เรียก `fallback`
เหมือนกันแต่**ปล่อยให้ status code เป็นไปตามที่ `fallback` ตอบเอง** เพราะ `ServeFile` ที่ serve ไฟล์ที่มีอยู่
จริงจะตอบ `200 OK` ตามปกติ ตรงกับที่หัวข้อ 94.3 ใช้จริงและพิสูจน์ด้วย `curl -i` แล้วว่าได้ `200` ถูกต้อง —
กับดักนี้ชี้ให้เห็นหลักการที่กว้างกว่า: เมธอดสอง cluster ที่ชื่อคล้ายกันมาก (`fallback` vs
`not_found_service`) ของ crate เดียวกันให้ผลลัพธ์ต่างกันโดยสิ้นเชิงในรายละเอียดที่มองข้ามง่าย (status code)
— ต้องอ่าน source/doc ให้ละเอียดจริง ๆ ไม่ใช่เดาจากชื่อเมธอดหรือคัดลอกตัวอย่างจากเอกสารโดยไม่เข้าใจว่ามัน
ทำอะไรกับ status code

### 3. `apt-get install` ใน Dockerfile ล้มเหลวด้วย `403 Forbidden` ในสภาพแวดล้อมที่มี network policy จำกัด

ระหว่างพัฒนา Dockerfile ของหัวข้อ 94.5 เวอร์ชันแรก (ที่ยังใช้ `apt-get install -y curl ca-certificates
libssl3` ในหลาย stage ตามที่ Dockerfile ทั่วไปมักทำ) เจอ error ที่**ไม่เกี่ยวกับโค้ด Rust เลย** แต่เกี่ยวกับ
network policy ของสภาพแวดล้อมที่ build:

```
E: Failed to fetch http://deb.debian.org/debian/dists/bookworm/InRelease  403  Forbidden [IP: 151.101.66.132 80]
E: The repository 'http://deb.debian.org/debian bookworm InRelease' is not signed.
```

**สาเหตุ**: บางสภาพแวดล้อม (CI runner ขององค์กร, sandbox ที่มี egress policy, หรือ corporate network ที่
บล็อก mirror บางตัว) ปฏิเสธการเข้าถึง `deb.debian.org` โดยเฉพาะ ในขณะที่ยังอนุญาตโดเมนอื่น (เช่น
`crates.io`) — error message ที่ได้ (`403 Forbidden`) **ดูเหมือนปัญหา credential/permission** แต่จริง ๆ
คือ policy denial ที่ระดับ network ซึ่งไม่มีทาง "แก้" ด้วยการเปลี่ยนคำสั่ง `apt-get` เลย (retry ไม่ช่วย,
เปลี่ยน mirror ไม่ช่วยถ้า mirror นั้นถูกบล็อกเหมือนกัน)

**วิธีแก้ (หรือมากกว่านั้น — วิธีเลี่ยงปัญหาแต่ต้น)**: ตรวจสอบก่อนว่า runtime image **จำเป็นต้อง**
`apt-get install` จริงหรือไม่ — ในบทนี้ตรวจด้วย `ldd` กับ binary ที่ build แล้วพบว่าไม่มี dependency ต่อ
`libssl` เลย (เพราะทุก crate ที่ใช้ crypto เลือก pure-Rust backend) และไม่ต้อง `ca-certificates` (เพราะแอป
ไม่เรียก HTTPS ออกไปข้างนอก) จึงตัด `apt-get install` ในทุก stage ออกได้ทั้งหมด และเปลี่ยนวิธีติดตั้ง
`trunk` จากการดาวน์โหลด binary ผ่าน `curl`+GitHub Releases มาเป็น `cargo install trunk --locked` (ผ่าน
crates.io ที่ยังเข้าถึงได้) — ถ้าโปรเจกต์ของคุณจำเป็นต้องมี package จาก apt จริง ๆ (เช่นต้องใช้ `libpq-dev`
สำหรับ driver บางตัว) ทางออกที่ยั่งยืนกว่าคือใช้ base image ที่ bundle สิ่งที่ต้องการไว้แล้ว (เช่น
`rust:1-slim` ที่มี toolchain ครบ) แทนการ `apt-get install` เพิ่มเองระหว่าง build ถ้าเป็นไปได้ หรือ config
mirror ภายในองค์กรที่ network policy อนุญาตไว้แทน `deb.debian.org` ตรง ๆ

### 4. `rust:1.82-slim` compile ไม่ผ่านเพราะ dependency ต้องการ Cargo feature `edition2024`

ระหว่างพัฒนาหัวข้อ 94.5 การเลือก base image ตาม pattern ที่คุ้นเคย (pin เวอร์ชัน Rust ให้ตรงกับที่เคยใช้ใน
บทอื่น ๆ ของหลักสูตร เช่น `rust:1.82-slim`) ทำให้ `cargo build --release` ภายใน Docker build stage ล้มเหลว
ทันทีตอน resolve dependency tree (ไม่ใช่ตอน compile โค้ดของเราเอง):

```
error: failed to parse manifest at `/usr/local/cargo/registry/.../cpufeatures-0.3.1/Cargo.toml`

Caused by:
  feature `edition2024` is required

  The package requires the Cargo feature called `edition2024`, but that feature is not stabilized
  in this version of Cargo (1.82.0 (8f40fc59f 2024-08-21)).
```

**สาเหตุ**: `Cargo.lock` ของโปรเจกต์ (ที่ resolve ไว้ด้วย toolchain รุ่นใหม่กว่าบนเครื่อง dev จริง) เลือก
เวอร์ชันของ transitive dependency บางตัว (`cpufeatures` ในกรณีนี้) ที่ประกาศ `edition = "2024"` ใน
`Cargo.toml` ของมันเอง — edition 2024 (และ Cargo feature `edition2024` ที่ต้องใช้ตอน resolve manifest ของ
crate ที่ใช้ edition นี้) ยังไม่ stabilize ใน Cargo เวอร์ชันที่มาคู่กับ `rust:1.82-slim` (Cargo 1.82) ทำให้
Cargo เวอร์ชันนั้นอ่าน manifest ของ dependency ไม่ออกเลย แม้โค้ดของเราเองจะไม่ได้ใช้ syntax ของ edition 2024
อะไรเลยก็ตาม — ปัญหาอยู่ที่ **transitive dependency** ไม่ใช่โค้ดของเรา

**วิธีแก้**: ใช้ base image ที่มี Cargo/rustc ใหม่พอสำหรับ dependency ทั้งหมดใน `Cargo.lock` — เปลี่ยนจาก
`rust:1.82-slim` (pin เวอร์ชันตายตัว) เป็น `rust:1-slim` (tag ที่ตามเวอร์ชัน stable ล่าสุดของสาย major
version 1 เสมอ ให้ผลเทียบเท่ากับ toolchain ที่ใช้ resolve `Cargo.lock` ไว้แต่แรก) ซึ่งได้ Cargo 1.98.1 ที่
compile ผ่านทุกจุด — ข้อคิดที่กว้างกว่านี้คือ: **การ pin เวอร์ชัน Docker base image ให้ตรงกับเวอร์ชัน Rust
ที่ใช้ resolve `Cargo.lock` (หรือใหม่กว่า) สำคัญกว่าการพยายาม pin ให้ตรงกับบทอื่น ๆ ในหลักสูตรที่อาจ resolve
`Cargo.lock` ไว้คนละช่วงเวลา** — ถ้าต้องการความเสถียรระยะยาวของ CI/CD ให้ pin เป็นเลขเวอร์ชันที่ทดสอบแล้วว่า
compile ผ่านจริง (เช่น `rust:1.98-slim` แทน `rust:1-slim` ที่เปลี่ยนไปเรื่อย ๆ ตามเวลา) แทนการเดาจากความ
คุ้นเคย

### 5. `.sqlx/` cache เก่ากว่า migration ล่าสุด ทำให้ Docker build ผ่านแต่ query ผิดจาก schema จริง

ระหว่างพัฒนาหัวข้อ 94.5 (build backend ภายใน Docker ด้วย `SQLX_OFFLINE=true`) ถ้าลืมรัน `cargo sqlx prepare`
ใหม่หลังเพิ่ม/แก้ query ใน handler (เช่น สมมติแก้ `list_books` ให้ `SELECT` column เพิ่มอีกตัวหนึ่งตามแบบ
ฝึกหัดข้อ 1 ของ Part 92) แล้ว build image โดยไม่ได้ regenerate `.sqlx/` cache ก่อน — **`docker build` จะผ่าน
สนิท ไม่มี error หรือ warning ใด ๆ เลย** เพราะ `sqlx::query_as!` ใน offline mode ตรวจสอบกับไฟล์ `.sqlx/
query-<hash>.json` ที่ COPY เข้า image (ซึ่งยังเป็นเวอร์ชันเก่าที่ไม่มี column ใหม่) ไม่ใช่กับ schema ฐาน
ข้อมูลจริง — แต่พอรัน container จริงกับฐานข้อมูลที่ migrate ไปแล้ว (มี column ใหม่จริง) แล้วมี query อื่นที่
เขียน**ไม่ตรงกับ schema จริง**หลุดเข้ามา (เช่น ลืมอัปเดต query บางจุดให้ตรงกับ migration ใหม่) จะได้ error
จาก database ตอน**runtime** ไม่ใช่ตอน build:

```
error returned from database: column "some_new_column" does not exist
```

**สาเหตุ**: `.sqlx/` cache เป็น "ภาพนิ่ง" (snapshot) ของ schema ณ เวลาที่รัน `cargo sqlx prepare` ครั้งล่าสุด
เท่านั้น — มันไม่รู้เองว่ามี migration ใหม่เกิดขึ้นหลังจากนั้น (ต่างจากโหมด online ปกติที่ต่อฐานข้อมูลจริงตอน
compile ทุกครั้ง ตามที่ Part 71 อธิบายไว้ — offline mode แลก "ไม่ต้องมี DB ตอน build" กับ "ต้องดูแลให้ cache
สอดคล้องกับ schema ปัจจุบันด้วยมือ") ถ้า workflow การพัฒนาไม่มีวินัยชัดเจนเรื่องนี้ จะเกิดสถานการณ์ที่ CI
build ผ่านทุกครั้ง (เพราะ offline mode ไม่เคยเห็น schema จริงเลย) แต่ deploy จริงพังเพราะ schema จริงกับที่
cache "จำ" ไว้ไม่ตรงกัน — เป็นความเสี่ยงที่ร้ายแรงกว่า compile error ปกติมาก เพราะไม่มีสัญญาณเตือนจนกว่าจะถึง
production จริง

**วิธีแก้**: ทำให้ "regenerate `.sqlx/` cache" เป็นส่วนหนึ่งของ workflow ที่บังคับเสมอ ไม่ใช่ขั้นตอนที่ทำ
ด้วยมือแล้วอาจลืม — วิธีที่ปลอดภัยที่สุดคือเพิ่ม CI check ที่รัน `cargo sqlx prepare --check` (flag `--check`
ทำให้คำสั่งนี้**ไม่เขียนไฟล์ใหม่** แต่ตรวจสอบว่า cache ที่ commit ไว้ตรงกับที่ query ในโค้ดต้องการจริงหรือไม่
— ถ้าไม่ตรง คำสั่งจบด้วย exit code ที่ไม่ใช่ 0 ทำให้ CI fail ทันที) เป็นขั้นตอนหนึ่งใน pull request check
ทุกครั้งที่มีการแก้ query หรือ migration — วิธีนี้ทำให้ไม่มีทางที่ query ที่ไม่ตรงกับ cache จะหลุดเข้า `main`
branch ไปได้เลย (ผู้อ่านที่อยากลองทำ CI check นี้จริงดูแบบฝึกหัดขยายผลได้จาก workflow เดียวกับที่ Part 71/92
ใช้ `sqlx-cli` อยู่แล้ว)

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เพิ่ม environment variable ใหม่ `APP_VERSION` ที่อ่านจาก `env!("CARGO_PKG_VERSION")` (macro
   ของ Rust ที่อ่านเวอร์ชันจาก `Cargo.toml` ตอน compile time) แล้วเพิ่ม key `version` เข้าไปใน JSON
   response ของ `GET /health` (หัวข้อ 94.7) ให้เป็น `{"status":"ok","database":"ok","version":"0.1.0"}` —
   ทดสอบด้วย `curl` จริงกับเซิร์ฟเวอร์ที่คุณสร้างจากบทนี้ (hint: `env!(...)` ต่างจาก `std::env::var(...)`
   ตรงที่ตัวแรกอ่านค่าตอน**compile time**จาก `Cargo.toml` ไม่ใช่จาก environment ตอน runtime — เหมาะกับ
   ข้อมูลที่ "ผูกกับ build" เช่นเลขเวอร์ชัน ในขณะที่ตัวหลังเหมาะกับ config ที่เปลี่ยนได้ระหว่าง deploy)

2. **[กลาง]** ปัจจุบัน `main.rs` ของบทนี้ (หัวข้อ 94.4) ใช้ `.unwrap_or_else(|| "...".to_string())` สำหรับ
   `JWT_SECRET` — แปลว่าถ้าลืมตั้ง `JWT_SECRET` ตอน deploy production จริง แอปจะรันต่อไปได้เงียบ ๆ ด้วย
   dev secret ที่ทุกคนรู้ (เพราะมันอยู่ในซอร์สโค้ดที่ commit เข้า git) ซึ่งเป็นช่องโหว่ความปลอดภัยร้ายแรง —
   ให้แก้ `main.rs` ให้ **panic ทันทีตอน startup** (ไม่ใช่ fallback เงียบ ๆ) ถ้า `JWT_SECRET` ไม่ได้ถูกตั้ง
   ไว้เลย **เมื่อรันในโหมด production** (hint: เพิ่ม environment variable ใหม่ เช่น `APP_ENV` ที่มีค่าเป็น
   `"development"` หรือ `"production"` แล้วเขียน logic แบบ: ถ้า `APP_ENV == "production"` และไม่มี
   `JWT_SECRET` ตั้งไว้ ให้ `panic!(...)`/`std::process::exit(1)` ทันทีพร้อมข้อความที่ชัดเจนว่าต้องตั้งอะไร
   — ถ้าเป็น `"development"` ยัง fallback เป็น dev secret ได้ตามเดิมเพื่อความสะดวก)

3. **[ยาก]** แยก `GET /health` ของหัวข้อ 94.7 ออกเป็นสอง endpoint ตามที่ Kubernetes/ระบบ orchestration
   ระดับใหญ่ทำจริง: `GET /health/live` (liveness — ตอบ `200` เสมอตราบใดที่ process ยังรับ request ได้ ไม่
   query database เลย) และ `GET /health/ready` (readiness — query database จริงแบบเดิมของหัวข้อ 94.7,
   ตอบ `503` ถ้า DB ไม่พร้อม) — อัปเดต `docs.rs` (utoipa) ให้มีทั้งสอง path ด้วย พร้อมอธิบายในคอมเมนต์ว่า
   ทำไมสองความหมายนี้ควรแยกกัน (hint: คิดถึงสถานการณ์ที่ database ปกติแต่ process กำลัง deadlock/ค้างอยู่ —
   หรือสถานการณ์ที่ database ล่มชั่วคราวแต่ process ยังทำงานปกติทุกอย่างอื่น — ทั้งสองสถานการณ์ต้องการการ
   ตอบสนองที่ต่างกันจาก orchestration layer: อันแรกควร restart container/process อันหลังไม่ควร restart
   แค่หยุดส่ง traffic เข้าชั่วคราวเท่านั้น)

4. **[ยาก/ประยุกต์]** implement **graceful shutdown** ให้ `main.rs` ของบทนี้ — ปัจจุบันถ้า container ถูก
   สั่งหยุด (เช่น `docker stop`, หรือ Kubernetes ส่ง `SIGTERM` ตอน rolling update) `axum::serve(...)` จะถูก
   ตัดจบทันทีโดยไม่รอ request ที่กำลังประมวลผลอยู่ให้เสร็จก่อน (อาจทำให้ผู้ใช้ที่กำลังยืม/คืนหนังสืออยู่
   พอดีเจอ connection ขาดกลางทาง) ให้เพิ่ม `.with_graceful_shutdown(...)` เข้าไปใน `axum::serve(...)` ที่
   ดัก signal `SIGTERM`/`SIGINT` แล้วรอ request ที่กำลังทำงานอยู่ให้เสร็จก่อน (ภายใน timeout ที่กำหนด เช่น
   10 วินาที) ก่อนปิด process จริง (hint: crate `tokio::signal` มีฟังก์ชันดัก `ctrl_c()` และ (บน Unix)
   `signal(SignalKind::terminate())` — เขียนฟังก์ชัน `async fn shutdown_signal()` ที่ `tokio::select!` ระหว่าง
   สองสิ่งนี้ แล้วส่งเป็น future เข้า `with_graceful_shutdown`)

## สรุป

บทนี้ปิดช่องว่างสามข้อที่ทำให้ backend ของ Part 92 และ frontend ของ Part 93 (ที่รันได้จริงแต่คนละ process
คนละ origin ด้วย CORS bridge แบบ dev) กลายเป็นแอปเดียวที่ deploy ได้จริง — ทำ production build ของทั้งสอง
ฝั่ง (`cargo build --release` ได้ binary เล็กลง ~8.5 เท่า, `trunk build --release` ได้ไฟล์ static สามไฟล์
ขนาดรวม ~1.5MB), ตัดสินใจสถาปัตยกรรม single-origin โดยเพิ่ม `ServeDir`/`ServeFile` เข้าไปใน Router เดิมของ
Part 92 (ปิดปัญหา CORS ในโปรดักชันไปเลยเพราะไม่มี cross-origin request เกิดขึ้นจริง), ย้าย config ไปเป็น
environment-variable-driven พร้อม `.env.example`, เขียน multi-stage Dockerfile ที่ build ทั้งสองฝั่งแล้วรวม
เป็น image เดียวที่ไม่ต้องมี Rust toolchain ติดไปด้วย, เลือกกลยุทธ์ migration ที่ปลอดภัยสำหรับ
multi-instance deploy (ควบคุมได้ด้วย `RUN_MIGRATIONS_ON_STARTUP`), ขยาย `/health` ให้ตรวจสถานะ database
จริงแทน "process ไม่ตาย" เฉย ๆ, และเปิดให้สลับ log format ระหว่าง pretty/JSON ได้ด้วย environment variable
เดียว — ทุกอย่างพิสูจน์ด้วยการรัน combined application จริงเป็นหนึ่งหน่วยเดียว ทั้งระดับ API ผ่าน `curl` และ
ระดับเบราว์เซอร์ผ่าน headless Chromium จริง โดยพูดตรง ๆ ถึงข้อจำกัดของสภาพแวดล้อมที่ใช้ตรวจสอบบทนี้เรื่อง
network policy สำหรับการ build Docker image แบบเต็มรูปแบบ

ทั้งสามบท (92, 93, 94) รวมกันคือ capstone ที่แสดงให้เห็นว่าเทคนิคทั้งหมดของ Module 4 (Axum backend) และ
Module 5 (frontend framework, WASM, SSR) ประกอบกันเป็นระบบจริงที่ deploy ได้อย่างไร — **สิ่งถัดไปคือ Part
95 (Testing Web Applications แบบครบวงจร)** ซึ่งเขียนไว้ก่อน capstone นี้จะมีอยู่จริง (ใช้แอปตัวแทนสอนเทคนิค
testing แทน) แต่เนื้อหายังใช้ได้ตรงกับสิ่งที่ 3 บทนี้ทำไปแล้ว จากนั้น **Module 6 (Production/DevOps)** จะ
เริ่มด้วย **Part 96 (Docker)** ที่ลงรายละเอียดกลไกของ container ให้ลึกกว่า Dockerfile ที่บทนี้แนะนำไว้แบบ
ผ่าน ๆ

---

**Part ก่อนหน้า:** [Full-Stack Project (2/3): สร้าง Frontend](part-093-fullstack-frontend.md) | **Part
ถัดไป:** [Testing Web Applications แบบครบวงจร (unit/integration/e2e)](part-095-testing-web-apps.md)
