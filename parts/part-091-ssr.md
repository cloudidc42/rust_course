# Part 91: Server-Side Rendering (SSR) ด้วย Rust

> โมดูล: Full-Stack และ WebAssembly | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความแตกต่างที่แม่นยำระหว่าง **CSR** (Client-Side Rendering), **SSR** (Server-Side Rendering) และ
  **SSG** (Static Site Generation) ทั้งสามแนวทาง พร้อม trade-off เชิงรูปธรรมด้าน Time-to-First-Byte, SEO,
  ภาระของ server, และความซับซ้อนของ build — ไม่ใช่แค่จำนิยามลอย ๆ
- อธิบายกลไกของ **hydration** ได้ในระดับที่ลึกกว่า Part 89 มาก: รู้ว่าทำไม hydration ถึง "ยากกว่าที่ฟังดู"
  และพิสูจน์ได้จริงว่าความไม่ตรงกันระหว่าง HTML ที่ server ส่งมากับ view ที่ client คำนวณ (**hydration
  mismatch**) จะทำให้เกิดอะไรขึ้นจริง — ทั้งกรณีที่ error ออกมาดัง ๆ และกรณีที่ "เงียบ" อย่างน่ากลัวกว่า
- เปรียบเทียบความพร้อมด้าน SSR ของ Leptos, Yew และ Dioxus ได้อย่างตรงไปตรงมา โดยอ้างอิงจากซอร์สโค้ดจริงของ
  แต่ละ crate ไม่ใช่จากคำโฆษณา
- ตั้งและรัน Leptos SSR server จริงที่ query ข้อมูลจาก PostgreSQL แล้วพิสูจน์ด้วย `curl` ว่าเนื้อหาจริงอยู่ใน
  HTML ตั้งแต่ response แรก **ก่อน** JavaScript/WASM จะได้รันเลยด้วยซ้ำ
- อธิบายและพิสูจน์ **streaming SSR** ได้จริงด้วยตัวเลขที่วัดจริง (Time-To-First-Byte ต่ำ ทั้งที่ response
  ทั้งก้อนใช้เวลานานกว่านั้นมาก) และเข้าใจว่า SSR ช่วย TTFB แต่ **ไม่ได้ทำให้ Time-to-Interactive หายไป**
- ตั้งค่า `<title>`/meta description ต่างกันในแต่ละ route ด้วย `leptos_meta` และพิสูจน์ด้วย `curl` ว่า
  crawler ที่ไม่รัน JavaScript จะเห็นค่าเหล่านี้จริง
- อธิบายได้ว่าทำไม SSR ต้องมี server process รันอยู่ตลอดเวลา (ผูกกับความรู้ Axum จาก Module 4) และรู้จัก
  แนวทาง hybrid ที่ผสม SSG กับ SSR/CSR เข้าด้วยกันในระบบเดียว — ซึ่งเป็นพื้นฐานตรงที่ Full-Stack Project
  (Part 92-94) จะต่อยอด

## ความรู้ที่ต้องมีมาก่อน

- **Part 89 (Leptos Framework: Full-stack Rust)**: บทนี้ "ต่อยอด" หัวข้อ 89.7-89.8 ตรง ๆ — Part 89 แนะนำ
  CSR/SSR/hydration ในระดับภาพรวมพอให้เข้าใจตัวอย่าง Axum integration บทนี้จะลงรายละเอียดกลไกจริง พิสูจน์
  hydration mismatch ที่ Part 89 ไม่ได้ทำ และขยายไปสู่ streaming/SEO/deployment ที่ Part 89 ไม่ได้พูดถึง ถ้า
  ยังไม่แน่นเรื่อง signal, server function, หรือการผูก Leptos เข้ากับ Axum ควรทวน Part 89 ก่อน
- **Part 88 (Yew Framework) และ Part 90 (Dioxus Framework)**: หัวข้อ 91.3 เปรียบเทียบ SSR maturity ของทั้ง
  สามเฟรมเวิร์กโดยตรง จำเป็นต้องรู้จักโมเดล virtual DOM ของ Yew และแนวคิด renderer-agnostic ของ Dioxus
  มาก่อนถึงจะเข้าใจว่าทำไมความพร้อมด้าน SSR ของแต่ละตัวถึงต่างกัน
- **Part 62-66 (Axum เต็มรูปแบบ)**: SSR server ทุกตัวอย่างในบทนี้รันอยู่**ภายใน** `axum::Router` เดียวกับที่
  Part 62-66 สอนไว้ — บทนี้จะไม่อธิบาย `Router`/route/handler พื้นฐานซ้ำอีก
- **Part 70 (Database: PostgreSQL ด้วย SQLx)**: ตัวอย่างหลักของบทนี้ (หัวข้อ 91.4, 91.5, 91.9) query ข้อมูล
  หนังสือจริงจาก PostgreSQL ด้วย `sqlx::query_as!`/`query_scalar!` ตรงตามที่ Part 70 สอนไว้ทุกประการ
- **Part 36 (Macros: Declarative Macros)**: ใช้ตอนอธิบายว่า `#[cfg(feature = "ssr")]` ที่เป็นต้นเหตุของ
  hydration mismatch ในหัวข้อ 91.2 คือ conditional compilation ที่ทำงานตอน compile time เท่านั้น

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้พูดถึงกลไกที่ "ผิดง่ายและดีบักยาก" ที่สุดของ Rust full-stack web (hydration mismatch) จึงตรวจสอบทุกข้อ
อ้างด้วยการรันจริง ไม่ใช่จากความจำหรือบทความเก่า:

- ตั้งโปรเจกต์ scratch **สองโปรเจกต์แยกไว้นอก repo**: โปรเจกต์แรก (`mismatch_app`) เป็น Leptos SSR+hydrate
  แบบ manual (ไม่ผ่าน `cargo-leptos`) ที่ตั้งใจทำให้ server กับ client render ต่างกัน แล้วเปิดด้วย **headless
  Chromium จริงผ่าน Playwright** (`chromium.launch()`) เพื่อดักจับ `console.error`/`pageerror` จริงทุกบรรทัด
  โปรเจกต์ที่สอง (`books_app`) เป็น Leptos SSR ที่ query PostgreSQL จริง (cluster เดียวกับที่ตั้งไว้ตั้งแต่
  Part 70/89) มีทั้งหน้า SSR ปกติ, หน้า streaming, และหน้าที่ทำ SSG-in-process
- `cargo add leptos`/`leptos_axum`/`leptos_meta` บนเครื่องจริง (rustc/cargo 1.94.1) resolve ได้ **leptos
  0.8.21**, **leptos_axum 0.8.10**, **leptos_meta 0.8.7** (ตรงกับที่ Part 89 ตรวจสอบไว้ — เวอร์ชันไม่ขยับ
  ระหว่างสองบทนี้) พร้อม `axum 0.8.9`, `sqlx 0.9.0`, `tokio 1.53.1`, `any_spawner 0.3.0`, `wasm-bindgen
  0.2.129` — ถ้าคุณ `cargo add` แล้วได้เลขเวอร์ชันอื่น ให้ยึดของคุณเป็นความจริงล่าสุด
- **build เป็น WASM จริงด้วย feature `hydrate`** (ไม่ใช่แค่ SSR feature ฝั่ง native เหมือนที่ Part 89 ทำ) ผ่าน
  `cargo build --target wasm32-unknown-unknown --features hydrate` แล้วรัน `wasm-bindgen --target web` เอง
  (ไม่ผ่าน `cargo-leptos`/`trunk`) เพื่อได้ไฟล์ `.wasm`/`.js` จริงที่ใส่ไว้ในหน้า SSR ให้เบราว์เซอร์โหลดไป
  hydrate — นี่คือขั้นตอนที่ Part 89 บอกไว้ตรง ๆ ว่า "ข้ามไปเพราะ `cargo-leptos` ติดตั้งไม่สำเร็จในเวลาที่
  เหมาะสม" บทนี้ทำขั้นตอนนี้ให้เสร็จจริงด้วยการข้าม `cargo-leptos` ไปเลยและเรียก `wasm-bindgen` ตรง ๆ
- **ระหว่างพิสูจน์ hydration mismatch เจอบั๊กในโค้ดทดสอบของตัวเองจริง ๆ** (ไม่ใช่บั๊กของ Leptos): การ mount
  component ที่มี `<html>` ของตัวเองเข้ากับ `hydrate_body()` ตรง ๆ ทำให้ panic ทันทีแม้เนื้อหาทั้งสองฝั่งจะ
  ตรงกันเป๊ะ ต้องแก้โครงสร้างให้แยก `Shell` (ฝั่ง server เท่านั้น) ออกจาก `App` (ที่ถูก hydrate จริง) ก่อนถึง
  จะพิสูจน์ mismatch ที่ตั้งใจได้อย่างสะอาด — รายละเอียดเต็มอยู่ในหัวข้อ 91.2 และกับดักที่พบบ่อยข้อ 2
- ตั้งฐานข้อมูล PostgreSQL จริง (ใช้ database `leptos_scratch` เดียวกับ Part 89 ที่ยังรันอยู่ในเครื่อง — เพิ่ม
  ตาราง `books` เข้าไปใหม่) แล้ว query จริงผ่าน `sqlx::query_as!`/`query_scalar!` ยิง `curl` จริงทั้งหน้า
  ปกติ, หน้า streaming (วัดเวลาจริงด้วย `curl -w`), และหน้าที่ pre-render ไว้ตอน startup
- ทุก error message, ผลลัพธ์ `curl`, และ log ที่ยกมาในบทนี้คือสิ่งที่**รันจริงแล้วคัดลอกมา** — ถ้าคุณลองทำตาม
  แล้วได้ผลต่างเล็กน้อย (เช่น byte size ของ HTML หรือ hash ของไฟล์ WASM) ให้ยึดผลจากเครื่องคุณเป็นความจริง
  ล่าสุด

## เนื้อหา

### 91.1 CSR, SSR, SSG: สามแนวทาง เปรียบเทียบให้ชัดเจนที่สุด

Part 88-90 สอน Yew, Leptos, Dioxus โดยตัวอย่างส่วนใหญ่เป็น **CSR (Client-Side Rendering)** — server ส่ง
`index.html` ที่เกือบเปล่า (มีแค่ `<div id="root"></div>` หรือ `<body></body>` เปล่า ๆ) พร้อมไฟล์ `.wasm`/`.js`
มาให้ แล้ว **ทุกอย่างที่ผู้ใช้เห็นถูกสร้างขึ้นในเบราว์เซอร์ทั้งหมด** ด้วย JavaScript/WASM ที่รันหลังจากไฟล์
โหลดเสร็จ Part 89 หัวข้อ 89.7-89.8 แนะนำ **SSR (Server-Side Rendering)** ในระดับภาพรวมว่า server render
`view!` tree เป็น HTML string จริงตั้งแต่ request แรก บทนี้จะเติมแนวทางที่สามที่ยังไม่ได้พูดถึงเลยคือ **SSG
(Static Site Generation)** และเทียบทั้งสามแนวทางให้เห็นภาพชัดในตารางเดียว

**SSG คือการ render HTML ล่วงหน้าตอน build time (ไม่ใช่ตอนมี request เข้ามา)** แล้วเก็บไฟล์ `.html` ที่ได้ไว้
เป็นไฟล์ static ธรรมดา เมื่อมี request จริงเข้ามา server (หรือ CDN) แค่ส่งไฟล์ที่ render ไว้แล้วกลับไปตรง ๆ
โดยไม่ต้องรันโค้ด render ใหม่เลย ตัวอย่างที่คุ้นเคยคือเว็บไซต์ documentation ที่สร้างด้วย `mdBook` (ที่ Part 1
เคยพูดถึงตอนสอนตั้ง toolchain) — เนื้อหาถูกแปลงเป็น HTML ตอน `mdbook build` ครั้งเดียว ไม่มีการ render ซ้ำทุก
ครั้งที่มีคนเปิดหน้าเว็บ

ตารางเปรียบเทียบสามแนวทางบนแกนที่สำคัญที่สุดสี่แกน:

| แกนเปรียบเทียบ | CSR | SSR | SSG |
|---|---|---|---|
| **Time-to-First-Byte (เนื้อหาจริง)** | แย่ที่สุด — response แรกไม่มีเนื้อหาเลย ต้องรอ WASM โหลด+รัน+fetch ข้อมูลก่อนถึงเห็นอะไร | ดี — เนื้อหาจริงอยู่ใน response แรกเสมอ แต่ต้อง render (และมักต้อง query DB) ทุก request จึงมี latency ของงานนั้นบวกเข้าไป | ดีที่สุด — server แค่ส่งไฟล์ static ที่มีอยู่แล้ว ไม่มี render/query เกิดขึ้นเลยตอน request |
| **SEO / Social media crawler** | แย่ — crawler ที่ไม่รัน JS (ยังมีอยู่จริงจำนวนมาก ดูหัวข้อ 91.6) เห็นหน้าเปล่า | ดี — crawler เห็น HTML เต็มรูปแบบเหมือนผู้ใช้จริงทันที | ดีที่สุด — เหมือน SSR ทุกประการแต่รับประกันว่าเสิร์ฟเร็วกว่าเพราะไม่มี render cost |
| **ภาระของ server ต่อ request** | ต่ำมาก — server แค่เสิร์ฟไฟล์ static (มักฝากไว้บน CDN ได้เลย) | สูง — ทุก request ต้องรันโค้ด Rust render + มักต้อง query database | ต่ำมาก — เหมือน CSR (เสิร์ฟไฟล์ static) แต่ได้เนื้อหาเต็มด้วย |
| **ความซับซ้อนของ build/deploy** | ต่ำ — build ครั้งเดียวได้ไฟล์ static ชุดเดียว ไม่ต้องมี server process รันตลอด | สูงกว่า — ต้องมี server process (Axum) รันอยู่ตลอดเวลา (หัวข้อ 91.8) | สูงที่สุด — ต้องมีขั้นตอน build แยกที่ "รู้" ว่าต้อง pre-render หน้าไหนบ้าง และต้อง rebuild ใหม่ทุกครั้งที่ข้อมูลเปลี่ยน |

จากตารางนี้เห็นได้ว่า **ไม่มีแนวทางไหน "ดีที่สุด" แบบไม่มีเงื่อนไข** — SSG เร็วและถูกที่สุดแต่ใช้ได้เฉพาะเนื้อหา
ที่ไม่เปลี่ยนบ่อย (เพราะต้อง rebuild ใหม่ทุกครั้งที่ข้อมูลเปลี่ยน) SSR เหมาะกับเนื้อหาที่เปลี่ยนบ่อยและต้อง
personalize ต่อผู้ใช้ (เช่นจำนวนที่นั่งที่เหลือแบบ real-time ในโดเมนที่ Part 89 ใช้) แต่แลกมาด้วยภาระ server
ที่สูงกว่า CSR เหมาะกับแอปที่ interaction เยอะมากหลัง login แล้ว (เช่น dashboard ภายในองค์กรที่ไม่ต้องแคร์ SEO
เลย เพราะ crawler ไม่มีสิทธิ์ login เข้าไปดูอยู่แล้ว) และไม่ต้องแคร์ TTFB ของหน้าแรกมากเพราะผู้ใช้ยอมรอ loading
ได้ (คล้ายกับที่ผู้ใช้ยอมรอแอป desktop โหลด)

จุดที่บทนี้จะเน้นเป็นพิเศษคือ **SSR และ SSG ไม่ใช่ทางเลือกที่ต้องเลือกอย่างใดอย่างหนึ่งสำหรับทั้งแอป** — แอป
จริงส่วนใหญ่ผสมทั้งสามแนวทางในหน้าต่างกัน หรือแม้แต่ในหน้าเดียวกัน (หัวข้อ 91.9 จะสร้างตัวอย่างจริงที่พิสูจน์
เรื่องนี้)

#### กรอบคิดง่าย ๆ สำหรับเลือกแนวทาง: ถามสามคำถามกับแต่ละหน้า

เพื่อไม่ให้การเลือกแนวทางเป็นเรื่องนามธรรมเกินไป ลองใช้โดเมนห้องสมุด/ระบบจองตั๋วที่ใช้มาตลอดโมดูลนี้ (Part
88-90) เป็นตัวอย่าง แล้วถามสามคำถามนี้กับ**แต่ละหน้า**ในแอป (ไม่ใช่ถามครั้งเดียวกับทั้งแอป):

1. **"เนื้อหาของหน้านี้เปลี่ยนบ่อยแค่ไหน"** — ถ้าเปลี่ยนแทบไม่เคย (เช่น หน้า "เกี่ยวกับห้องสมุด", หน้า
   "นโยบายการยืม-คืน", หน้า "ติดต่อเรา") → เอียงไปทาง **SSG** เพราะไม่มีเหตุผลต้อง render ใหม่ทุก request
2. **"หน้านี้ต้อง personalize ต่อผู้ใช้แต่ละคนหรือไม่ และ SEO สำคัญไหม"** — ถ้าต้อง personalize (เช่นหน้า
   "รายการหนังสือที่ฉันยืมอยู่") หรือข้อมูลเปลี่ยนเร็วมากจน SSG ไม่ทัน (เช่น "ที่นั่งที่เหลือตอนนี้" จากโดเมน
   Part 89) แต่ยังต้องการให้ crawler เห็นเนื้อหาบางส่วน (เช่นหน้ารายละเอียดหนังสือที่อยาก index ใน Google) →
   เอียงไปทาง **SSR**
3. **"หน้านี้อยู่หลัง login และไม่มีใครต้องการ index มันเลยหรือไม่"** — ถ้าใช่ (เช่น หน้า admin dashboard
   จัดการสต็อกหนังสือ, หน้าตั้งค่าบัญชีผู้ใช้) → **CSR ล้วน ๆ ก็เพียงพอ** เพราะไม่มี SEO ให้แคร์ และผู้ใช้ที่
   login แล้วมักยอมรอ loading เล็กน้อยได้ (คล้ายพฤติกรรมที่ยอมรอแอป desktop เปิด)

ตารางสรุปเมื่อ apply กรอบคิดนี้กับหน้าต่าง ๆ ของระบบห้องสมุดที่บทนี้จะสร้างจริงในหัวข้อ 91.4-91.9:

| หน้า | เปลี่ยนบ่อยแค่ไหน | ต้อง personalize/SEO? | หลัง login? | แนวทางที่เลือก |
|---|---|---|---|---|
| `/about` (เกี่ยวกับห้องสมุด) | แทบไม่เปลี่ยน | ไม่ personalize, SEO มีประโยชน์ | ไม่ | **SSG** (หัวข้อ 91.9) |
| `/books` (รายการหนังสือ) | เปลี่ยนทุกครั้งที่มีคนยืม/คืน | ไม่ personalize แต่ SEO มีประโยชน์มาก (คนค้นหาชื่อหนังสือ) | ไม่ | **SSR** (หัวข้อ 91.4) |
| `/my-loans` (หนังสือที่ฉันยืมอยู่) | เปลี่ยนตามการยืม-คืนของแต่ละคน | personalize เต็มรูปแบบ ไม่มี SEO | ใช่ | **CSR** หรือ SSR ก็ได้ (ไม่มี SEO benefit แต่ personalize ทำให้ cache ยากอยู่แล้ว) |
| หน้า admin จัดการสต็อก | เปลี่ยนบ่อยมาก แต่เฉพาะ staff เท่านั้นที่เห็น | ไม่มี SEO เลย | ใช่ | **CSR ล้วน ๆ** เพียงพอ |

ข้อสังเกตสำคัญจากตารางนี้: **ตัวชี้ขาดที่แท้จริงมักไม่ใช่ "ความถี่ที่ข้อมูลเปลี่ยน" อย่างเดียว แต่คือ "ใครจะเห็น
หน้านี้และทำไม"** — หน้าที่มีแต่ผู้ใช้ที่ login แล้วเห็น ไม่มีเหตุผลต้องแคร์ SEO เลย ต่อให้ข้อมูลเปลี่ยนบ่อยแค่
ไหนก็ยังเลือก CSR ได้สบาย ๆ เพราะ "ต้นทุน" ของ SSR (ภาระ server ต่อ request) ไม่ได้แลกมาซึ่งประโยชน์ด้าน SEO
ที่ไม่มีใครได้ใช้อยู่แล้ว

### 91.2 Hydration แบบลงลึก: กลไกจริง และสิ่งที่เกิดขึ้นเมื่อมันผิดพลาด

Part 89 หัวข้อ 89.7 อธิบาย hydration ในระดับแนวคิดว่า "WASM ฝั่ง client จับคู่กับ HTML ที่มีอยู่แล้ว แล้วผูก
event listener เข้าไป โดยไม่ทำลาย DOM เดิม" คำอธิบายนี้ถูกต้องแต่ทำให้เรื่องดูง่ายเกินไป — คำถามที่ Part 89
ไม่ได้ตอบคือ **"client จะ 'จับคู่' กับ DOM เดิมได้อย่างไร ในเมื่อมันไม่ได้สร้าง DOM นั้นขึ้นมาเอง"**

คำตอบคือ: component function ฝั่ง client (ที่ compile ด้วย feature `hydrate`) **ต้องรันตัวเองเพื่อคำนวณว่า
"ถ้าฉันสร้าง view นี้เอง มันจะมีโครงสร้างหน้าตาอย่างไร"** จากนั้นมันจะเดินไล่ DOM ที่มีอยู่แล้ว (ที่ server
ส่งมา) **โดยสมมติว่า** โครงสร้างที่มันคำนวณได้ ตรงกับโครงสร้าง DOM จริงทุกจุด ถ้าสมมติฐานนี้ผิด — เช่น
component ฝั่ง client คิดว่าตำแหน่งนี้ควรเป็น text node แต่ DOM จริงกลับเป็น element — เกิด **hydration
mismatch**

นี่คือเหตุผลที่ hydration ยากกว่าที่ฟังดู: **client-side code ต้องผลิต virtual representation ที่ตรงกับ
HTML ที่ server render ออกมาแบบไม่มีที่ติเลย** ไม่ใช่แค่ "หน้าตาคล้ายกัน" — ทุก element, ทุก text node, ทุก
ลำดับต้องตรงกันเป๊ะ ต่างจาก virtual DOM diffing (Part 88) ที่ทน "ความต่าง" ได้เพราะมันคำนวณ diff ใหม่ทุกครั้ง
ไม่ได้พึ่งการเดา — hydration ไม่มีขั้นตอน diff เลย มันแค่ "เดินไปข้างหน้า" ตาม cursor แล้วหวังว่าจะเจอสิ่งที่
คาดไว้

#### พิสูจน์จากซอร์สโค้ดจริง: `hydrate()` ทำงานอย่างไรตอน element ไม่ตรงกัน

ไล่ดูซอร์สโค้ดจริงของ `tachys` (rendering engine ของ Leptos ที่ Part 89 อ้างถึงตอนพิสูจน์เรื่อง "ไม่มี
diffing") ไฟล์ `tachys-0.2.19/src/hydration.rs` มีฟังก์ชันที่ถูกเรียกเมื่อ cursor เจอ node ที่ "ไม่ใช่ชนิดที่
คาดไว้":

```rust
// จากซอร์สโค้ดจริง — tachys-0.2.19/src/hydration.rs (ย่อให้อ่านง่ายขึ้น)
pub(crate) fn failed_to_cast_text_node(node: Node) -> Text {
    let hydrating = CURRENTLY_HYDRATING
        .take()
        .map(|n| n.to_string())
        .unwrap_or_else(|| "{unknown}".to_string());
    web_sys::console::error_3(
        &wasm_bindgen::JsValue::from_str(&format!(
            "A hydration error occurred while trying to hydrate an \
             element defined at {hydrating}.\n\nThe framework expected a \
             text node, but found this instead: ",
        )),
        &node,
        &wasm_bindgen::JsValue::from_str(
            "\n\nThe hydration mismatch may have occurred slightly \
             earlier, but this is the first time the framework found a \
             node of an unexpected type.",
        ),
    );
    panic!(
        "Unrecoverable hydration error. Please read the error message \
         directly above this for more details."
    );
}
```

สังเกตสามจุดสำคัญจากซอร์สโค้ดนี้: (1) มันรู้ **ตำแหน่งในซอร์สโค้ด** ที่กำลัง hydrate อยู่ (`CURRENTLY_HYDRATING`
เก็บ `Location` ที่มาจาก `#[track_caller]`-style mechanism) จึงบอกได้ว่า element ไหนในไฟล์ไหนบรรทัดไหนที่มี
ปัญหา (2) มันพิมพ์ `console.error` **ก่อน** panic เสมอ เพื่อให้ developer เห็นรายละเอียดใน DevTools ก่อนที่
โปรแกรมจะตาย (3) มันจบด้วย **`panic!`** เสมอ — hydration mismatch แบบนี้ไม่ใช่ warning ที่ทำงานต่อได้ มันคือ
"unrecoverable error" ตามคำในซอร์สโค้ดตรง ๆ

แต่มีสิ่งที่ต้องรู้ที่สำคัญกว่านั้น: `failed_to_cast_text_node`/`failed_to_cast_element` จะถูกเรียกก็ต่อเมื่อ
`cast_from()` ล้มเหลว — และไล่ดูซอร์สโค้ดจริงของ `cast_from` สำหรับ `Element` (ไฟล์ `tachys-0.2.19/src/
renderer/dom.rs`) พบว่า:

```rust
// จากซอร์สโค้ดจริง — tachys-0.2.19/src/renderer/dom.rs
impl CastFrom<Node> for Element {
    fn cast_from(node: Node) -> Option<Element> {
        node.clone().dyn_into().ok()
    }
}
```

`dyn_into::<web_sys::Element>()` ตรวจสอบแค่ว่า node นี้เป็น **Element ชนิดไหนก็ได้** (`<span>`, `<div>`,
`<p>`, ...) เทียบกับ interface กลางของ DOM เท่านั้น — **มันไม่ได้ตรวจสอบว่า tag name ตรงกับที่ view! คาดไว้
เป๊ะหรือไม่เลย** นี่คือรายละเอียดที่ทำให้เกิดพฤติกรรมสองแบบที่ต่างกันโดยสิ้นเชิงเมื่อ hydration mismatch เกิดขึ้น
ขึ้นอยู่กับว่า **ชนิดของ mismatch** คืออะไร — และนี่คือสิ่งที่บทนี้จะพิสูจน์ด้วยการรันจริงในหัวข้อถัดไป

#### ตั้งฉากทดสอบจริง: component เดียวกัน compile สองแบบ ให้ผลต่างกันตั้งใจ

เพื่อพิสูจน์เรื่องนี้ให้เห็นจริง ไม่ใช่แค่อ่านซอร์สโค้ดเดา สร้างโปรเจกต์ scratch ชื่อ `mismatch_app` ที่ compile
ได้สองแบบจากซอร์สโค้ดเดียวกัน (crate เดียว มี `[lib]` เป็น `cdylib`+`rlib` และมี `src/main.rs` เป็น binary
สำหรับฝั่ง server) `Cargo.toml`:

```toml
[package]
name = "mismatch_app"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
leptos = "0.8.21"

# --- hydrate (client / WASM) เท่านั้น ---
console_error_panic_hook = { version = "0.1", optional = true }
wasm-bindgen = { version = "0.2.129", optional = true }

# --- ssr (server / native) เท่านั้น ---
axum = { version = "0.8.9", optional = true }
tokio = { version = "1", features = ["full"], optional = true }
leptos_axum = { version = "0.8.10", optional = true }
tower-http = { version = "0.6", features = ["fs"], optional = true }
any_spawner = { version = "0.3", features = ["tokio"], optional = true }

[features]
hydrate = ["leptos/hydrate", "console_error_panic_hook", "wasm-bindgen"]
ssr = ["leptos/ssr", "axum", "tokio", "leptos_axum", "tower-http", "any_spawner"]
```

`src/lib.rs` แยก `Shell` (เอกสารทั้งก้อน ใช้ฝั่ง server เท่านั้น) ออกจาก `App` (ส่วนที่ถูก hydrate จริง) —
เหตุผลของการแยกนี้จะอธิบายในกับดักที่พบบ่อยข้อ 2 ท้ายบท:

```rust
use leptos::prelude::*;

/// `Shell` คือเปลือกเอกสารทั้งหน้า (<html>/<head>/<body> + script โหลด WASM)
/// — render ฝั่ง server เท่านั้น ไม่ถูกส่งเข้า hydrate เลย
#[component]
pub fn Shell() -> impl IntoView {
    view! {
        <html lang="th">
            <head>
                <meta charset="utf-8" />
                <title>"Hydration Mismatch Demo"</title>
            </head>
            <body>
                <App />
                <script type="module">
                    "import init from '/pkg/mismatch_app.js'; init();"
                </script>
            </body>
        </html>
    }
}

/// `App` คือ component ที่ทั้งฝั่ง server (ผ่าน Shell) และฝั่ง client
/// (ผ่าน hydrate_body(App)) เรียกใช้ "เหมือนกันทุกตัวอักษร"
#[component]
pub fn App() -> impl IntoView {
    view! {
        <h1>"Hydration Mismatch Demo"</h1>
        <p id="same">"เนื้อหานี้เหมือนกันทั้งสองฝั่ง"</p>
        <MismatchDemo />
    }
}

/// จุดที่ตั้งใจทำให้ "ไม่ตรงกัน" ระหว่าง server กับ client โดยใช้
/// #[cfg(feature = "ssr")] แยกโค้ดสองเส้นทางออกจากกัน — บั๊กแบบนี้เกิดขึ้น
/// จริงในโปรเจกต์จริงได้ง่าย ๆ เช่น โค้ดที่ตั้งใจ "ซ่อน" widget บางตัวเฉพาะ
/// ตอน render ฝั่ง server เพื่อประหยัด bandwidth โดยไม่รู้ตัวว่าจะทำให้
/// hydration พังทั้งหน้า
#[component]
fn MismatchDemo() -> impl IntoView {
    view! {
        <div id="demo">
            {
                #[cfg(feature = "ssr")]
                {
                    view! { <span id="branch">"เรนเดอร์จาก SERVER"</span> }.into_any()
                }
                #[cfg(not(feature = "ssr"))]
                {
                    "เรนเดอร์จาก CLIENT".into_any()
                }
            }
        </div>
    }
}

/// จุดเข้าฝั่ง client: โหลด WASM แล้ว hydrate เฉพาะเนื้อหาข้างใน <body>
#[cfg(feature = "hydrate")]
#[wasm_bindgen::prelude::wasm_bindgen(start)]
pub fn hydrate() {
    console_error_panic_hook::set_once();
    leptos::mount::hydrate_body(App);
}
```

จุดสำคัญของ mismatch นี้คือ: ฝั่ง server สร้าง **Element** (`<span>`) ตรงตำแหน่งนี้ ส่วนฝั่ง client (ตอน
compile ด้วย feature `hydrate` เพียว ๆ ไม่มี `ssr`) สร้าง **plain text node** (สตริงเปล่า ไม่มี element ห่อ)
ตรงตำแหน่งเดียวกัน — นี่คือความต่าง "ชนิดของ DOM node" (Element เทียบกับ Text) ไม่ใช่แค่เนื้อหาข้างในต่างกัน

build ทั้งสอง target จริง: ฝั่ง server ด้วย `cargo build --features ssr --no-default-features` (native
binary ธรรมดา) และฝั่ง client ด้วย `cargo build --target wasm32-unknown-unknown --features hydrate
--no-default-features --lib` แล้วแปลงเป็น JS glue code ด้วย `wasm-bindgen` ตรง ๆ (ไม่ผ่าน `wasm-pack`/
`trunk` เพื่อควบคุมทุกไฟล์ output เอง):

```bash
$ cargo install wasm-bindgen-cli --version 0.2.129 --offline
$ wasm-bindgen --target web --out-dir pkg --out-name mismatch_app \
    target/wasm32-unknown-unknown/debug/mismatch_app.wasm
$ ls pkg
mismatch_app.d.ts  mismatch_app.js  mismatch_app_bg.wasm  mismatch_app_bg.wasm.d.ts
```

server (`src/main.rs`) เสิร์ฟทั้งหน้า SSR และไฟล์ `pkg/` ที่ได้จากขั้นตอนข้างบนใน `Router` เดียวกัน:

```rust
#[cfg(feature = "ssr")]
#[tokio::main]
async fn main() {
    use axum::Router;
    use mismatch_app::Shell;
    use tower_http::services::ServeDir;

    // leptos_routes_with_context() ปกติจะเรียกสิ่งนี้ให้อัตโนมัติ แต่ตัวอย่าง
    // นี้ใช้ fallback ธรรมดาเพื่อความง่าย จึงต้อง init global executor ของ
    // Leptos เองก่อนเริ่ม serve request แรก (ดูกับดักที่พบบ่อยข้อ 1)
    any_spawner::Executor::init_tokio().expect("init tokio executor");

    let app = Router::new()
        .fallback(leptos_axum::render_app_to_stream(Shell))
        .nest_service("/pkg", ServeDir::new("pkg"));

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3091").await.unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}

#[cfg(not(feature = "ssr"))]
fn main() {}
```

รัน server จริงแล้ว `curl` ทันที (ก่อนเปิดเบราว์เซอร์ใด ๆ เลย) — นี่คือ HTML **จริง** ที่ server ส่งมา:

```
$ curl -sS http://127.0.0.1:3091/
<html lang="th"><head><meta charset="utf-8"><title>Hydration Mismatch Demo</title></head>
<body><h1>Hydration Mismatch Demo</h1><p id="same">เนื้อหานี้เหมือนกันทั้งสองฝั่ง</p>
<div id="demo"><span id="branch">เรนเดอร์จาก SERVER</span></div>
<script type="module">import init from '/pkg/mismatch_app.js'; init();</script>
</body></html><script nonce="...">__RESOLVED_RESOURCES=[];__SERIALIZED_ERRORS=[];...</script>
```

(จัดบรรทัดใหม่เพื่อให้อ่านง่ายขึ้น — ของจริงเป็นบรรทัดเดียว) สังเกตว่า `#branch` เป็น `<span>` ที่มีข้อความ
"เรนเดอร์จาก SERVER" ตรงตามที่ `#[cfg(feature = "ssr")]` กำหนดไว้ — นี่คือ HTML ที่ **จะถูกส่งไปให้เบราว์เซอร์
แสดงผลก่อน** WASM จะโหลดเสร็จด้วยซ้ำ

#### ผลลัพธ์จริงที่เกิดขึ้นตอนเปิดในเบราว์เซอร์: panic + console.error จริง

เปิดหน้านี้ด้วย headless Chromium ผ่าน Playwright แล้วดักจับ `console` event และ `pageerror` event ทุกตัว:

```javascript
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const consoleMsgs = [];
  page.on('console', (msg) => consoleMsgs.push(`[console.${msg.type()}] ${msg.text()}`));
  page.on('pageerror', (err) => consoleMsgs.push(`[pageerror] ${err.message}`));

  await page.goto('http://127.0.0.1:3091/', { waitUntil: 'load' });
  await page.waitForTimeout(1500);

  const branchHtml = await page.evaluate(() => document.getElementById('branch')?.outerHTML);
  console.log('--- #branch element ใน DOM หลังพยายาม hydrate ---');
  console.log(branchHtml);
  console.log('--- console/page messages ที่ดักได้จริงทั้งหมด ---');
  for (const line of consoleMsgs) console.log(line);
  await browser.close();
})();
```

ผลลัพธ์จริงที่ได้ (คัดลอกจาก stdout จริง ไม่มีการแก้ไข):

```
--- #branch element ใน DOM หลังพยายาม hydrate ---
<span id="branch">เรนเดอร์จาก SERVER</span>
--- console/page messages ที่ดักได้จริงทั้งหมด ---
[console.error] A hydration error occurred while trying to hydrate an element defined at src/lib.rs:50:10.

The framework expected a text node, but found this instead:  JSHandle@node

The hydration mismatch may have occurred slightly earlier, but this is the first time the framework found a node of an unexpected type.
[console.error] panicked at /root/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/tachys-0.2.19/src/hydration.rs:248:9:
Unrecoverable hydration error. Please read the error message directly above this for more details.
...
[pageerror] unreachable
```

ผลลัพธ์นี้ตรงกับที่คาดจากซอร์สโค้ดเป๊ะ: console.error บอกตำแหน่งไฟล์+บรรทัด (`src/lib.rs:50:10` ซึ่งคือ
ตำแหน่งของ `<div id="demo">` ใน `view!`), บอกว่า "expected a text node, but found this instead" (เพราะ
ฝั่ง client คิดว่าจะเจอ text node แต่ตำแหน่งนั้นใน DOM จริงเป็น `<span>` element) แล้ว panic จริงด้วยข้อความ
"Unrecoverable hydration error" ตามด้วย `pageerror: unreachable` — คือ WASM trap (คำสั่ง `unreachable`
ของ WebAssembly ที่ Rust panic แปลงมาเป็น) ที่ทำให้ **WASM instance นั้นตายสนิท ไม่ทำงานต่ออีกเลย**

ผลที่ผู้ใช้จะเห็นจริง ๆ คือ: **หน้าเว็บยังคงแสดงผล HTML จาก server เดิมทุกประการ** (เพราะ hydration ยังไม่ทัน
เขียนอะไรทับ DOM ก่อนจะ panic) แต่ **ทั้งหน้าไม่ตอบสนองอะไรเลยอีกต่อไป** — ปุ่มกดไม่ได้ ฟอร์มไม่ทำงาน สิ่งที่
มองเห็นด้วยตาจึงดูปกติสนิท ทำให้บั๊กแบบนี้อันตรายกว่าที่คิด เพราะไม่มี "หน้าจอสีแดงบอก error" ให้ผู้ใช้ทั่วไป
เห็นเลย — ต้องเปิด DevTools console เท่านั้นถึงจะรู้ว่าเกิดอะไรขึ้น

#### สิ่งที่น่าประหลาดใจกว่า: mismatch บางแบบไม่ error เลยแม้แต่นิดเดียว

ตอนแรกของการทดสอบบทนี้ mismatch ที่ตั้งใจทำคือสลับ `<span>` (ฝั่ง server) กับ `<div>` (ฝั่ง client) — สอง
element ต่างชื่อ tag แต่ยังเป็น **Element ทั้งคู่**:

```rust
#[cfg(not(feature = "ssr"))]
{
    view! { <div id="branch">"เรนเดอร์จาก CLIENT"</div> }.into_any()
}
```

ผลลัพธ์จริงจากการรัน Playwright แบบเดียวกันกับข้างบน:

```
--- #branch element ใน DOM หลังพยายาม hydrate ---
<span id="branch">เรนเดอร์จาก SERVER</span>
--- console/page messages ที่ดักได้จริงทั้งหมด ---
```

**ไม่มี console message เลยแม้แต่บรรทัดเดียว** — hydration "ผ่าน" แบบเงียบ ๆ ทั้งที่ server ตั้งใจ render
`<span>` แต่ client ตั้งใจ render `<div>` ตรงตำแหน่งเดียวกัน! นี่คือผลตรงตามที่พิสูจน์จากซอร์สโค้ดของ
`Element::cast_from` ข้างบน — `dyn_into::<web_sys::Element>()` เช็คแค่ "เป็น Element หรือไม่" ไม่ได้เช็ค tag
name เฉพาะเจาะจง ดังนั้น `<span>` ที่มีอยู่จริงใน DOM จึง cast ผ่านได้เสมอไม่ว่า client จะ "คาดหวัง" tag ไหน
ก็ตาม — Leptos แค่ยึด DOM node ที่มีอยู่ (ซึ่งเป็น `<span>`) มาผูก reactive effect เข้าไป โดยไม่รู้ตัวว่าจริง ๆ
แล้วมันควรจะเป็น `<div>` ตามที่ client คำนวณไว้

**นี่คือบทเรียนสำคัญที่สุดของหัวข้อนี้**: แนวคิดที่หลายคนเข้าใจว่า "hydration mismatch ใดก็ตามจะทำให้ error
ออกมาดัง ๆ เสมอ" เป็นการพูดง่ายเกินไปอย่างอันตราย — **Leptos (เวอร์ชันที่ตรวจสอบในบทนี้) ตรวจจับได้เฉพาะความ
ต่างระดับ "ชนิดของ DOM node"** (Element ↔ Text ↔ Comment) **แต่ไม่ตรวจจับความต่างระดับ "identity ของ tag"
ภายในชนิดเดียวกัน** (span ↔ div ↔ p ล้วนเป็น Element เหมือนกัน) mismatch แบบ tag-swap จึงเป็นบั๊กแบบ "เงียบ"
ที่อันตรายกว่า panic เสียอีก เพราะไม่มีสัญญาณเตือนอะไรเลยทั้งใน console และบนหน้าจอ ผลกระทบที่แท้จริงจะ
ปรากฏก็ต่อเมื่อโค้ดส่วนอื่นพึ่งพา "รู้ว่า node นี้เป็น div" อย่างเจาะจง เช่น query ผ่าน `NodeRef` แล้ว
`downcast` เป็น `HtmlDivElement` ตรง ๆ ซึ่งจะ panic คนละจุดคนละเวลาไปเลย ห่างไกลจากต้นเหตุจริงมาก

#### ทำไมกลไกนี้ถึงเป็นแบบนี้: `AnyView` และ "branch marker"

เหตุผลที่ mismatch แบบ tag-swap เงียบได้ ในขณะที่ mismatch แบบ element-vs-text panic ดัง อยู่ที่กลไกของ
`.into_any()` ที่ทั้งสอง branch ใน `MismatchDemo` ต้องเรียก (เพราะ `#[cfg(feature = "ssr")]` กับ
`#[cfg(not(...))]` แต่ละฝั่งคืนค่าเป็นคนละ concrete type กัน — `HtmlElement<Span, ...>` ไม่ใช่ type เดียวกับ
`HtmlElement<Div, ...>` หรือ `&str` — Rust ต้องมี type เดียวที่เป็นตัวแทนได้ทั้งคู่ จึงต้อง erase type ด้วย
`AnyView`) ไล่ดูซอร์สโค้ดจริงของ `tachys-0.2.19/src/view/any_view.rs` ตอน hydrate:

```rust
// จากซอร์สโค้ดจริง — tachys-0.2.19/src/view/any_view.rs (ย่อ)
fn hydrate<const FROM_SERVER: bool>(
    self,
    cursor: &Cursor,
    position: &PositionState,
) -> Self::State {
    if FROM_SERVER {
        let state = (self.hydrate_from_server)(self.value, cursor, position);
        super::close_branch_marker(position);
        state
    } else {
        panic!("hydrating AnyView from inside a ViewTemplate is not supported.")
    }
}
```

`AnyView::hydrate` เรียก `self.hydrate_from_server` ซึ่งภายในเรียก `value.into_inner::<T>().hydrate::<true>
(cursor, position)` — สังเกตว่า **`T` ที่ใช้ตรงนี้คือ type ที่ *ฝั่ง client* คำนวณไว้ (คือ `div`/plain text
ตามแต่ branch ที่ client เลือก) ไม่ใช่ type ที่ server ใช้จริง** — เพราะ Rust ไม่มีทางรู้ตอน runtime ว่า
"server render อะไรมา" มันรู้แค่ "ตัวเองควรจะ hydrate เป็น type ไหน" แล้วพยายาม cast DOM node ตรงนั้นให้เป็น
type นั้น การ cast จะ**ผ่าน**ถ้า DOM node ที่มีอยู่จริง "เป็น Element ก็ได้" (เช่น `<span>` ที่ server ส่งมา
ก็ยัง cast เป็น `Element` แบบทั่วไปผ่านได้อยู่ดี ตามที่พิสูจน์แล้วข้างบน) แต่จะ**พัง**ถ้า DOM node ที่มีอยู่จริง
ไม่ใช่ Element เลย (เช่นเป็น Text node ตามตัวอย่าง panic ที่พิสูจน์ไว้)

ส่วน `close_branch_marker` คือกลไกที่ทำให้ `AnyView`/`<Show>`/`if-else` ใน `view!` ทำงานได้ทั้งที่แต่ละ branch
อาจมีจำนวน DOM node ไม่เท่ากัน — Leptos ห่อแต่ละ dynamic branch ด้วย **comment node คู่หนึ่ง** (คล้ายกับที่
เห็นในหัวข้อ 91.5 ตอนพิสูจน์ streaming ด้วย `<!--s-1-o-->`/`<!--s-1-c-->`) เพื่อให้รู้ขอบเขตของ branch นั้น
โดยไม่ต้องพึ่งการนับจำนวน children ให้ตรงกันเป๊ะ — นี่คือเหตุผลเชิงลึกที่ทำให้ Leptos "ยอม" ให้ความยาว/ชนิด
ของ element ภายใน dynamic branch ต่างกันได้โดยไม่ error (เพราะขอบเขตถูกกำหนดด้วย marker ไม่ใช่ด้วยการนับ)
แต่ยังคง**ต้องการ** ว่าตำแหน่งแรกที่มัน cast ต้องเป็นชนิด node ที่ "cast ได้" (Element หรือ Text ตามที่โค้ด
ฝั่ง client เรียก `cast_from` เป็นชนิดไหน)

#### เทียบกับ Yew: ทำไม virtual DOM diffing ไม่มีปัญหานี้เลย

คุ้มที่จะหยุดเทียบกับ Yew (Part 88) สักครู่ เพราะโมเดล virtual DOM diffing ของ Yew **ไม่มีแนวคิด "hydration
mismatch" แบบนี้เกิดขึ้นได้ในทางเดียวกัน** เหตุผลคือ Yew's hydration (จาก `ServerRenderer::hydratable(true)`
ที่พิสูจน์การมีอยู่จริงในหัวข้อ 91.3) ทำงานด้วยการ **re-run component function ฝั่ง client เพื่อสร้าง `VNode`
tree ใหม่ทั้งอัน แล้วค่อย "จับคู่" กับ DOM ที่มีอยู่ทีละ node ผ่านกระบวนการที่คล้าย diffing** — ถ้า `VNode`
ที่ได้ไม่ตรงกับ DOM เดิม Yew มีทางเลือกที่จะ "ซ่อม" ให้ตรง (เขียน DOM ใหม่ทับให้ตรงกับที่ component ต้องการ)
ซึ่งเป็นพฤติกรรมที่มาจาก mental model เดียวกับ diffing ปกติของมันอยู่แล้ว (Part 88) ในขณะที่ Leptos (fine-
grained reactivity ตาม Part 89 หัวข้อ 89.1) **ไม่มีขั้นตอน "สร้าง tree ใหม่มาเทียบ" เลยสักครั้ง** — มันแค่เดิน
cursor ไปข้างหน้าตาม assumption เดียว ไม่มีแผนสำรองถ้า assumption นั้นผิด นี่คือ trade-off ตรงข้ามกับที่ Part
89 หัวข้อ 89.1 อธิบายไว้ว่า fine-grained reactivity "ทำงานน้อยกว่าต่อการ update หนึ่งครั้งเสมอ" — ราคาที่ต้อง
จ่ายคือความทนทานต่อ mismatch ที่น้อยกว่า diffing-based model มาก

### 91.3 ใครรองรับ SSR ได้ดีแค่ไหน: Leptos, Yew, Dioxus

Part 88-90 แต่ละบทพูดถึง SSR ของเฟรมเวิร์กตัวเองแยกกัน หัวข้อนี้จะเทียบทั้งสามให้เห็นภาพเดียวกัน โดยอ้างอิงจาก
ซอร์สโค้ดจริงของแต่ละ crate

**Leptos — ระบบ full-stack ที่ครบวงจรที่สุด**. ตามที่ Part 89 หัวข้อ 89.6-89.8 พิสูจน์ไว้แล้ว Leptos มี
`leptos_axum` ที่ผูก SSR เข้ากับ `axum::Router` ได้ตรง ๆ, มี `#[server]` macro สำหรับ server function ที่
compile เป็นคนละโค้ดกันฝั่ง client/server จากซอร์สไฟล์เดียว, มี `leptos_meta` สำหรับ SEO (หัวข้อ 91.6), และ
รองรับ **streaming SSR** ในตัว (`SsrMode::OutOfOrder` เป็นค่า default — พิสูจน์จริงในหัวข้อ 91.5) — ทั้งหมด
นี้คือ ecosystem ที่ออกแบบมาให้ "SSR คือ first-class citizen" ตั้งแต่วันแรก

**Yew — มี SSR แต่ไม่มี unified full-stack story**. Yew มี `yew::ServerRenderer` (และ `LocalServerRenderer`)
ให้ใช้จริง — ตรวจสอบจากซอร์สโค้ดจริงของ `yew-0.23.0/src/server_renderer.rs`:

```rust
// จากซอร์สโค้ดจริง — yew-0.23.0/src/server_renderer.rs (ย่อ)
impl<COMP> LocalServerRenderer<COMP>
where
    COMP: BaseComponent,
{
    /// Sets whether an the rendered result is hydratable.
    /// Defaults to `true`.
    pub fn hydratable(mut self, val: bool) -> Self { ... }

    /// Renders Yew Application.
    pub async fn render(self) -> String { ... }
}
```

นี่พิสูจน์ว่า Yew **รองรับ SSR + hydration จริง** ไม่ใช่แค่ CSR ล้วน ๆ ตามที่ Part 88 อาจทำให้เข้าใจไปได้ —
`ServerRenderer::render()` คืนค่าเป็น `String` (HTML ธรรมดา) ตรง ๆ แต่สังเกตว่ามันเป็น **bare string renderer**
เท่านั้น: ไม่มี integration กับ Axum สำเร็จรูปเหมือน `leptos_axum` (ต้องเอา `String` ที่ได้ไปห่อเป็น
`axum::response::Html` เอง), ไม่มี concept ของ server function ที่ทำให้ client เรียกโค้ด server ได้แบบไร้
รอยต่อ, และไม่มีเครื่องมือช่วย SEO/meta tag แบบ `leptos_meta` มาให้ในตัว — ถ้าจะทำ Yew SSR ให้ครบเหมือน
Leptos ต้องประกอบเองทุกชิ้น (เขียน Axum handler เอง, เขียนโค้ด fetch ข้อมูลก่อน render เอง, จัดการ meta tag
เอง) ตรงกับที่ Part 89 หัวข้อ 89.6 อธิบายไว้ว่า Yew ไม่มีแนวคิด server function เลย — SSR ของ Yew ก็เจอ
ปัญหาแบบเดียวกัน: มันคือ "เครื่องมือ render" ล้วน ๆ ไม่ใช่ "framework แบบ full-stack"

**Dioxus — SSR renderer พื้นฐาน + `dioxus-fullstack` ที่กำลังไล่ตาม Leptos**. Part 90 ตาราง renderer
(หัวข้อ 90.1) ระบุ `dioxus-ssr` เป็น renderer สำหรับ render เป็น HTML string ธรรมดา — แต่ที่สำคัญกว่าคือ
Dioxus 0.7 (เวอร์ชันที่ Part 90 ใช้) มี crate แยกชื่อ **`dioxus-fullstack`** ที่ตรวจสอบจริงพบว่ามี proc macro
attribute `server` ของตัวเอง (ซอร์สโค้ดจริงจาก `dioxus-fullstack-macro-0.7.10/src/lib.rs`):

```rust
// จากซอร์สโค้ดจริง — dioxus-fullstack-macro-0.7.10/src/lib.rs
#[proc_macro_attribute]
pub fn server(attr: proc_macro::TokenStream, mut item: TokenStream) -> TokenStream {
    // ...
}
```

และ `dioxus-fullstack` เองก็ `pub use axum::{body, extract, response, routing};` ตรง ๆ — แสดงว่า Dioxus
0.7 กำลังเดินตามแนวทางเดียวกับ Leptos: มี server function attribute macro ของตัวเอง และผูกกับ Axum โดยตรง
ไม่ใช่แค่ "render เป็น string เฉย ๆ" แบบที่ Part 90 อาจให้ภาพไว้ (เพราะตอนที่ Part 90 เขียน โฟกัสอยู่ที่ web/
desktop/mobile renderer มากกว่า SSR) **ข้อสังเกตสำคัญ**: บทนี้ตรวจสอบความมีอยู่ของ API เหล่านี้ **จากการอ่าน
ซอร์สโค้ดจริงเท่านั้น ไม่ได้ compile และรัน Dioxus fullstack server จริงในสภาพแวดล้อมนี้** (ต่างจาก Leptos ที่
บทนี้ compile และรันจริงทุกตัวอย่าง) เพราะเวลาที่มีอยู่จำกัดอยู่ที่การพิสูจน์ hydration mismatch ของ Leptos ให้
ลึกที่สุดเป็นหลัก — ถ้าต้องเลือกใช้ Dioxus fullstack จริงในโปรเจกต์ ควรอ่านเอกสารและลองรันเองก่อนเชื่อคำอธิบาย
นี้ 100%

สรุปเป็นตารางเทียบ SSR maturity สามเฟรมเวิร์ก:

| | Leptos | Yew | Dioxus |
|---|---|---|---|
| SSR renderer พื้นฐาน | มี (`leptos_axum::render_app_to_stream` ฯลฯ) | มี (`yew::ServerRenderer`) | มี (`dioxus-ssr` / `dioxus-fullstack`) |
| Hydration | มี พร้อม fine-grained reactivity รองรับ (หัวข้อ 91.2) | มี (`hydratable(true)` เป็นค่า default) | มี |
| Server function (เขียนฟังก์ชันเดียว เรียกได้ทั้งสองฝั่ง) | มี (`#[server]`, Part 89 หัวข้อ 89.6) เป็นจุดขายหลักมาตั้งแต่แรก | **ไม่มี** — ต้องเขียน REST endpoint + fetch call เองคู่กัน | มี (`#[server]` ผ่าน `dioxus-fullstack`, verified จาก source เท่านั้นในบทนี้) |
| Axum integration สำเร็จรูป | มี (`leptos_axum`) แน่นและเป็นทางการที่สุด | ไม่มี — ต้องประกอบ `Html<String>` เอง | มี (`dioxus-fullstack` re-export `axum` ตรง ๆ) |
| Streaming SSR สำเร็จรูป | มี เป็นค่า default (`SsrMode::OutOfOrder`, หัวข้อ 91.5) | ไม่มีสำเร็จรูป (ต้องประกอบ stream เอง) | ไม่ได้ตรวจสอบในบทนี้ |
| SEO helper (`<Title>`/`<Meta>`) | มี (`leptos_meta`, หัวข้อ 91.6) | ไม่มีในตัว | ไม่ได้ตรวจสอบในบทนี้ |

ข้อสรุปที่ตรงไปตรงมาที่สุด: **ถ้าเป้าหมายคือ SSR แบบเต็มรูปแบบ (server function + streaming + SEO helper
ในระบบเดียว) Leptos ยังคือตัวเลือกที่พร้อมที่สุดและมีเอกสาร/ตัวอย่างมากที่สุด ณ ตอนที่เขียนบทนี้** — Yew
เหมาะกับกรณีที่ต้องการ SSR แบบพื้นฐาน (แค่ให้ crawler เห็น HTML) โดยไม่ต้องการ server function หรือ streaming
เลย ส่วน Dioxus กำลังพัฒนา `dioxus-fullstack` ให้ตามทันอย่างรวดเร็ว แต่การันตีความพร้อมระดับ production ควร
ตรวจสอบเอกสารและ community adoption ล่าสุดด้วยตัวเองก่อนใช้งานจริง เพราะระบบนี้ค่อนข้างใหม่กว่า `leptos_axum`
มาก นี่คือเหตุผลที่ Full-Stack Project (Part 92-94) จะเลือกใช้ Leptos เป็นหลัก

#### ภาพประกอบ: หน้าตาของการต่อ Yew SSR เข้ากับ Axum เองทั้งหมด

เพื่อให้เห็นภาพว่า "ไม่มี unified full-stack story" แปลว่าต้องทำอะไรเพิ่มเองบ้าง ลองเขียนภาพประกอบสั้น ๆ ว่า
ถ้าจะเอา Yew's `ServerRenderer` มาต่อกับ Axum ต้องทำอย่างไร (โค้ดนี้เป็น**ภาพประกอบเพื่อความเข้าใจเท่านั้น
ไม่ได้ compile/รันจริงในบทนี้** — ต่างจากตัวอย่าง Leptos ทุกตัวที่บทนี้ compile และรันจริงทั้งหมด เพราะบทนี้
เลือกใช้เวลาที่มีอยู่จำกัดไปกับการพิสูจน์ hydration mismatch ของ Leptos ให้ลึกที่สุดเป็นหลัก):

```rust
// ภาพประกอบเท่านั้น — ไม่ได้ compile/รันจริงในบทนี้
async fn yew_ssr_handler() -> axum::response::Html<String> {
    // ต้องเขียน handler เองทุกจุด ไม่มี .leptos_routes_with_context() ให้ใช้
    let html = yew::ServerRenderer::<App>::new().render().await;

    // ต้องประกอบ <html>/<head>/<script> เองด้วย string formatting ธรรมดา
    // (ไม่มี <MetaTags/> หรือ shell mechanism สำเร็จรูปแบบ leptos_axum)
    let full_page = format!(
        "<!DOCTYPE html><html><head><title>My Yew App</title></head><body>{html}\
         <script type=\"module\">import init from '/pkg/app.js'; init();</script>\
         </body></html>"
    );
    axum::response::Html(full_page)
}

// ถ้าต้องการให้ client เรียกข้อมูลจาก server (เทียบเท่า server function
// ของ Leptos) ต้องเขียน endpoint แยกเองอีกตัว แล้วเขียนโค้ด fetch ฝั่ง
// client เองอีกชุด (ผ่าน gloo-net ตามที่ Part 88 หัวข้อ 88.x แนะนำ) —
// ไม่มีกลไกที่ compile ฟังก์ชันเดียวให้เป็นทั้ง endpoint และ fetch call
// อัตโนมัติแบบ #[server] ของ Leptos เลย
```

เทียบจำนวนงานที่ต้องทำเอง: การประกอบ HTML shell, การเขียน route เชื่อมกับ Axum, และการเขียน fetch call คู่กับ
endpoint ล้วนเป็นสิ่งที่ `leptos_axum`/`#[server]` ทำให้อัตโนมัติในตัวอย่างของบทนี้ (หัวข้อ 91.4) ทั้งหมด — นี่
คือความต่างเชิง "ปริมาณโค้ดที่ต้องเขียนเอง" ที่จับต้องได้ ไม่ใช่แค่ความรู้สึกว่า "framework หนึ่งดูครบกว่า"

### 91.4 ตัวอย่างจริง: หน้ารายการหนังสือ SSR ด้วย Leptos + Axum + SQLx

ตอนนี้มาสร้างตัวอย่างที่จับต้องได้: หน้าเว็บที่ query รายการหนังสือจริงจาก PostgreSQL แล้ว render เป็น HTML
เต็มรูปแบบ พิสูจน์ด้วย `curl` ว่าเนื้อหาจริงอยู่ในไฟล์ตั้งแต่ response แรก โดเมนนี้คือห้องสมุดต่อยอดจาก Part 70
(ที่สอน SQLx เบื้องต้นด้วยระบบยืม-คืนหนังสือ) และ Part 89 (ที่ใช้ฐานข้อมูล `leptos_scratch` เดียวกัน)

สร้างตาราง `books` ในฐานข้อมูลเดิม:

```sql
CREATE TABLE books (
    id BIGSERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    author TEXT NOT NULL,
    total_copies INT NOT NULL,
    available_copies INT NOT NULL
);

INSERT INTO books (title, author, total_copies, available_copies) VALUES
    ('The Rust Programming Language', 'Steve Klabnik & Carol Nichols', 5, 2),
    ('Programming Rust', 'Jim Blandy & Jason Orendorff', 3, 3),
    ('Zero To Production In Rust', 'Luca Palmieri', 4, 0),
    ('Rust for Rustaceans', 'Jon Gjengset', 2, 1);
```

โปรเจกต์นี้ไม่ต้องมี hydrate/WASM เลย (เพื่อโฟกัสที่ SSR/database/streaming เป็นหลัก — เรื่อง hydration
พิสูจน์ครบแล้วในหัวข้อ 91.2) `Cargo.toml` จึงใช้ feature `ssr` อย่างเดียว:

```toml
[package]
name = "books_app"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8.9"
leptos = { version = "0.8.21", features = ["ssr"] }
leptos_axum = "0.8.10"
leptos_meta = "0.8.7"
any_spawner = { version = "0.3", features = ["tokio"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.9.0", default-features = false, features = ["runtime-tokio", "postgres", "macros"] }
tokio = { version = "1", features = ["full"] }
```

หัวใจของหัวข้อนี้คือ **การันตีว่าข้อมูลอยู่ใน HTML ก้อนแรกจริง ๆ ไม่ใช่ถูกฉีดเข้ามาทีหลังด้วย JavaScript** —
เทคนิคที่ใช้คือ query ข้อมูลจาก PostgreSQL ให้เสร็จ **ก่อน** เรียก Leptos renderer ด้วยซ้ำ แล้วส่งผลลัพธ์เข้าไป
ผ่าน `provide_context()` component จึงอ่านค่าแบบ synchronous ตรง ๆ ไม่ต้องพึ่ง `Resource`/`Suspense` เลย
(ต่างจากหัวข้อ 91.5 ที่จะใช้ `Suspense` ตั้งใจเพื่อพิสูจน์ streaming):

```rust
use axum::extract::State;
use axum::http::Request;
use axum::body::Body;
use axum::response::{IntoResponse, Response};
use leptos::prelude::*;
use leptos_meta::*;
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

#[derive(Clone, Debug, Serialize, Deserialize)]
struct Book {
    id: i64,
    title: String,
    author: String,
    total_copies: i32,
    available_copies: i32,
}

async fn fetch_books(pool: &PgPool) -> Vec<Book> {
    sqlx::query_as!(
        Book,
        "SELECT id, title, author, total_copies, available_copies \
         FROM books ORDER BY id"
    )
    .fetch_all(pool)
    .await
    .expect("query books")
}

#[component]
fn BooksShell() -> impl IntoView {
    provide_meta_context();
    let books = use_context::<Vec<Book>>().unwrap_or_default();

    view! {
        <html lang="th">
            <head>
                <meta charset="utf-8" />
                <Title text="รายการหนังสือ - ห้องสมุด Rust Course" />
                <Meta
                    name="description"
                    content="รายชื่อหนังสือ Rust ทั้งหมดในห้องสมุด พร้อมจำนวนที่ยืมได้ตอนนี้"
                />
                <MetaTags />
            </head>
            <body>
                <h1>"รายการหนังสือ (SSR ต่อ request จาก PostgreSQL จริง)"</h1>
                <ul id="book-list">
                    {books
                        .into_iter()
                        .map(|b| view! {
                            <li>
                                <strong>{b.title}</strong> " โดย " {b.author} " — เหลือ "
                                {b.available_copies} "/" {b.total_copies} " เล่ม"
                            </li>
                        })
                        .collect_view()}
                </ul>
            </body>
        </html>
    }
}

// handler เอง (ไม่ผ่าน leptos_routes_with_context) เพราะต้อง query DB
// "ก่อน" เรียก renderer แล้วส่งผลผ่าน provide_context
async fn books_handler(State(pool): State<PgPool>, req: Request<Body>) -> Response {
    let books = fetch_books(&pool).await;
    let handler = leptos_axum::render_app_to_stream_with_context(
        move || provide_context(books.clone()),
        BooksShell,
    );
    handler(req).await.into_response()
}
```

สังเกตสองจุดในโค้ดนี้ที่ควรอธิบายเพิ่ม: (1) `books.into_iter().map(|b| view! {...}).collect_view()` —
`collect_view()` คือเมธอดที่ Leptos เติมให้กับ `Iterator` ใด ๆ ที่ item เป็น `impl IntoView` ผ่าน trait
extension (คล้ายกับที่ `Iterator::collect()` ปกติต้องมี target type ที่ implement `FromIterator` — Part 26
สอนไว้แล้วว่า `collect()` ทำงานอย่างไรกับ type ปลายทางต่าง ๆ กัน) `collect_view()` ทำหน้าที่แปลง iterator
ของ view หลาย ๆ ตัวให้กลายเป็น view เดียวที่ render ต่อกันเป็นลิสต์ — ถ้าไม่มีเมธอดนี้ต้อง handle การรวม
`Vec<impl IntoView>` เป็น view เดียวเอง (2) `use_context::<Vec<Book>>()` ในการอ่านค่าคืนรูปแบบเดียวกันกับที่
Part 89 หัวข้อ 89.6 ใช้อ่าน `PgPool` ออกจาก context ของ server function — `provide_context`/`use_context`
เป็น mechanism กลางของ Leptos ที่ไม่ได้ผูกกับ SSR หรือ server function อย่างใดอย่างหนึ่งเท่านั้น มันคือวิธี
"ส่งค่าจากที่หนึ่งลงไปให้ลูกหลานในต้นไม้ component อ่านได้ โดยไม่ต้องส่งผ่าน prop ทุกชั้น" (เทียบเคียงได้กับ
`Context` API ของ React ถ้าคุณคุ้นเคยกับ ecosystem นั้น) ในกรณีนี้ context ถูก provide ที่ระดับ handler
(นอกต้นไม้ component เลยด้วยซ้ำ) แล้ว component (`BooksShell`) อ่านออกมาใช้ตรง ๆ

รัน server จริง (`cargo run` ธรรมดา ไม่ต้องพึ่ง `cargo-leptos` เลย เพราะ route นี้ไม่มี hydrate/WASM ให้จัดการ)
แล้ว `curl` ทันที — นี่คือ HTML **ดิบ** ที่ได้จริง ก่อนที่ JavaScript ตัวไหนจะได้รันแม้แต่บรรทัดเดียว (เพราะ
`curl` ไม่รัน JavaScript เลย):

```
$ curl -sS http://127.0.0.1:3092/books
<html lang="th"><head><meta charset="utf-8"><!--HEAD--><meta name="description"
content="รายชื่อหนังสือ Rust ทั้งหมดในห้องสมุด พร้อมจำนวนที่ยืมได้ตอนนี้ — เรนเดอร์เป็น HTML
เต็มรูปแบบตั้งแต่ request แรกด้วย Leptos SSR"><title>รายการหนังสือ - ห้องสมุด Rust
Course</title></head><body><h1>รายการหนังสือ (SSR ต่อ request จาก PostgreSQL จริง)</h1>
<ul id="book-list"><li><strong>The Rust Programming Language</strong> โดย <!>Steve Klabnik &amp;
Carol Nichols<!> — เหลือ <!>2<!>/<!>5<!> เล่ม</li><li><strong>Programming Rust</strong> โดย
<!>Jim Blandy &amp; Jason Orendorff<!> — เหลือ <!>3<!>/<!>3<!> เล่ม</li><li><strong>Zero To
Production In Rust</strong> โดย <!>Luca Palmieri<!> — เหลือ <!>0<!>/<!>4<!> เล่ม</li>
<li><strong>Rust for Rustaceans</strong> โดย <!>Jon Gjengset<!> — เหลือ <!>1<!>/<!>2<!> เล่ม</li>
<!></ul></body></html><script nonce="...">__RESOLVED_RESOURCES=[];...</script>
```

(จัดบรรทัดใหม่เพื่อให้อ่านง่าย — ของจริงเป็นบรรทัดเดียวยาว) นี่คือหลักฐานที่หนักแน่นที่สุด: รายชื่อหนังสือ
ทั้งสี่เล่ม, ผู้แต่ง, และจำนวนที่เหลือ **อยู่ใน HTML ตัวจริงทุกตัวอักษร** — ไม่มี `<div id="root"></div>`
เปล่า ๆ ที่รอ JS มาเติมแบบที่ CSR จะเป็น (`<!>` ที่เห็นแทรกอยู่คือ comment marker ที่ Leptos ใช้เป็น
"ตำแหน่งยึด" สำหรับ reactive text — ปรากฏได้แม้ใน SSR output เพราะจุดเหล่านั้นเป็นตำแหน่งที่**อาจจะ**เปลี่ยน
ค่าได้ถ้าเป็น dynamic content แต่ในกรณีนี้เป็นค่าคงที่จาก DB ที่ไม่ได้เปลี่ยนอีกแล้วหลัง render)

#### ประกอบทุกชิ้นเข้าด้วยกัน: `main()` เต็มรูปแบบ

โค้ดที่เห็นข้างบนคือแค่ component กับ handler ของ route `/books` — `main()` ที่ผูกทุกอย่างเข้ากับ
`axum::Router` (รวม route `/healthz` ธรรมดาที่ไม่เกี่ยวกับ Leptos เลย เพื่อยืนยันอีกครั้งตามที่ Part 89
หัวข้อ 89.8 พิสูจน์ไว้ว่า route ของ Leptos กับ route ธรรมดาอยู่ใน `Router` เดียวกันได้สบาย ๆ) มีดังนี้:

```rust
use axum::routing::get;
use axum::Router;
use sqlx::postgres::PgPoolOptions;

#[tokio::main]
async fn main() {
    // เหมือนกับกับดักที่พบบ่อยข้อ 1 ท้ายบท — ต้อง init เองเพราะไม่ได้ผ่าน
    // leptos_routes_with_context()
    any_spawner::Executor::init_tokio().expect("init tokio executor");

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://postgres:postgres@localhost/leptos_scratch")
        .await
        .expect("connect to postgres");

    let app = Router::new()
        .route("/books", get(books_handler))
        .route(
            "/healthz",
            get(|| async { axum::Json(serde_json::json!({"status": "ok"})) }),
        )
        .with_state(pool);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3092").await.unwrap();
    println!("listening on 127.0.0.1:3092");
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

พิสูจน์ว่า route ธรรมดา (`/healthz`) ยังทำงานได้ปกติในระบบเดียวกัน:

```
$ curl -sS http://127.0.0.1:3092/healthz
{"status":"ok"}
```

### 91.5 Streaming SSR: ส่ง HTML เป็นชิ้น ๆ ไม่ต้องรอทั้งหน้า

ตัวอย่างในหัวข้อ 91.4 ใช้ query ที่เร็ว (SELECT ธรรมดาจากตารางเล็ก) — แต่ในระบบจริง บางส่วนของหน้าอาจต้องรอ
ข้อมูลที่ query ช้า (เช่น เรียก external API, aggregate ข้อมูลหนัก, หรือ query ที่ query planner ยังไม่ดี)
ถ้า SSR ต้อง**รอทุกส่วนของหน้าให้พร้อมก่อนส่ง response แม้แต่ byte แรก** ผู้ใช้จะเห็นหน้าขาวจนกว่าส่วนที่ช้า
ที่สุดจะเสร็จ — ซึ่งแย่กว่า CSR เสียอีกในบางกรณี (เพราะ CSR อย่างน้อยเห็น loading skeleton ทันที)

**Streaming SSR แก้ปัญหานี้** ด้วยการส่ง HTML เป็นชิ้น ๆ (chunk) ผ่าน HTTP chunked transfer-encoding: ส่วนที่
render เร็ว (fast shell) ถูกส่งออกไปทันที ส่วนที่ต้องรอข้อมูลช้าจะถูกแทนที่ด้วย fallback ชั่วคราวก่อน แล้ว
ส่ง chunk เพิ่มเติมที่มีเนื้อหาจริงตามมาทีหลัง **ทั้งหมดนี้ยังเป็น HTTP response เดียวกัน** ไม่ใช่หลาย request

Leptos รองรับ streaming SSR แบบ **out-of-order** เป็นค่า default อยู่แล้ว — ตรวจสอบจริงจากซอร์สโค้ดของ
`leptos_axum-0.8.10/src/lib.rs` ในฟังก์ชัน `leptos_routes_with_context` พบว่า `SsrMode::OutOfOrder` (ค่า
default ของ route ที่ไม่ได้ระบุ mode) ใช้ `render_app_to_stream_with_context` ตรง ๆ — ฟังก์ชันตัวเดียวกันที่
หัวข้อ 91.4 เรียกตรง ๆ อยู่แล้ว! นั่นแปลว่า **ตัวอย่างในหัวข้อ 91.4 ก็ใช้ streaming SSR อยู่แล้วโดยไม่รู้ตัว**
เพียงแต่มันเร็วมากจนสังเกตไม่เห็นความต่าง (query เร็วมากจนทุกอย่างพร้อมในเวลาไล่เลี่ยกัน)

เพื่อให้เห็นความต่างชัด ๆ สร้าง route ใหม่ที่จำลอง query ช้าด้วย `tokio::time::sleep` แล้วห่อด้วย
`<Suspense>`:

```rust
use std::time::Duration;

#[component]
fn SlowBooksShell(pool: PgPool) -> impl IntoView {
    let books = Resource::new(
        || (),
        move |_| {
            let pool = pool.clone();
            async move {
                // จำลอง query ที่ใช้เวลานาน 2 วินาที
                tokio::time::sleep(Duration::from_secs(2)).await;
                fetch_books(&pool).await
            }
        },
    );

    view! {
        <html lang="th">
            <head><meta charset="utf-8" /><title>"Streaming SSR Demo"</title></head>
            <body>
                <h1>"ส่วนนี้มาถึงทันที (ไม่ต้องรอ DB ที่ช้า)"</h1>
                <Suspense fallback=|| view! { <p id="loading">"กำลังโหลดรายการหนังสือ (query ช้า)..."</p> }>
                    <ul id="slow-book-list">
                        {move || books.get().map(|list| {
                            list.into_iter().map(|b| view! { <li>{b.title}</li> }).collect_view()
                        })}
                    </ul>
                </Suspense>
            </body>
        </html>
    }
}
```

`<Suspense>` คือ component ของ Leptos ที่บอกว่า "รอ `Resource` ข้างในให้พร้อมก่อน แสดง `fallback` ระหว่างรอ"
— ส่วนที่อยู่**นอก** `<Suspense>` (คือ `<h1>`) ไม่ต้องรออะไรเลย ถูก render และส่งออกไปทันที

วัดเวลาจริงด้วย `curl -w` เทียบระหว่างหน้าปกติ (91.4) กับหน้านี้:

```
$ curl -sS -o /dev/null -w "TTFB=%{time_starttransfer}s  total=%{time_total}s\n" \
    http://127.0.0.1:3092/books
TTFB=0.002133s  total=0.002179s

$ curl -sS -o /dev/null -w "TTFB=%{time_starttransfer}s  total=%{time_total}s\n" \
    http://127.0.0.1:3092/slow-books
TTFB=0.001034s  total=2.004417s
```

ตัวเลขนี้คือหลักฐานที่ชัดที่สุดของ streaming SSR: **TTFB (Time-To-First-Byte) ของหน้าที่มีข้อมูลช้าอยู่
ที่ 0.001 วินาที — เร็วเท่ากับหน้าปกติเลย** ทั้งที่ response ทั้งก้อนใช้เวลารวม 2.004 วินาที (รอ
`tokio::time::sleep(2s)` ให้ครบก่อน) — เบราว์เซอร์ (หรือ crawler) จะเริ่มเห็นและ render `<h1>` **ทันที**
โดยไม่ต้องรอ 2 วินาทีนั้นเลย

ดู raw response เต็ม ๆ เพื่อเข้าใจกลไกจริงที่อยู่เบื้องหลัง:

```
$ curl -sS http://127.0.0.1:3092/slow-books
<html lang="th"><head>...<title>Streaming SSR Demo</title></head><body>
<h1>ส่วนนี้มาถึงทันที (ไม่ต้องรอ DB ที่ช้า)</h1>
<p>ด้านล่างนี้จะ 'ค้าง' อยู่ที่ fallback ก่อน แล้วค่อยเปลี่ยนเป็นรายการจริงหลังจาก 2 วินาที</p>
<!--s-1-o--><p id="loading">กำลังโหลดรายการหนังสือ (query ช้า)...</p><!--s-1-c-->
</body></html>
<template id="1-f"><ul id="slow-book-list"><li>The Rust Programming Language</li>
<li>Programming Rust</li><li>Zero To Production In Rust</li>
<li>Rust for Rustaceans</li><li>Rust Atomics and Locks</li><!></ul></template>
<script nonce="...">(function() {
  let id = "1-";
  let open, close;
  let walker = document.createTreeWalker(document.body, NodeFilter.SHOW_COMMENT);
  while (walker.nextNode()) {
    if (walker.currentNode.textContent == `s-${id}o`) open = walker.currentNode;
    else if (walker.currentNode.textContent == `s-${id}c`) close = walker.currentNode;
  }
  let range = new Range();
  range.setStartBefore(open); range.setEndBefore(close);
  range.deleteContents();
  let tpl = document.getElementById(`${id}f`);
  close.parentNode.insertBefore(tpl.content.cloneNode(true), close);
  close.remove();
})()</script>
<script nonce="...">__RESOLVED_RESOURCES[0] = "[{\"id\":1,\"title\":\"The Rust Programming...
```

(จัดบรรทัดใหม่เพื่อให้อ่านง่าย) กลไกที่ Leptos ใช้จริงคือ: (1) ส่วน `<h1>`/`<p>` ถูกส่งออกไปในชิ้นแรกทันที
พร้อมกับ `fallback` ของ `<Suspense>` ที่ถูกครอบด้วย HTML comment สองตัว (`<!--s-1-o-->` เปิด, `<!--s-1-c-->`
ปิด) เป็น "ตำแหน่งยึด" (2) เมื่อ `Resource` เสร็จ (หลัง 2 วินาที) ชิ้นที่สองจะถูกส่งตามมา: เนื้อหาจริงถูกห่อ
ไว้ใน `<template>` tag (ซึ่ง browser ไม่แสดงผล `<template>` ตรง ๆ) พร้อม JavaScript สั้น ๆ ที่ (3) เดินหา
comment marker คู่ที่ตรงกัน ลบเนื้อหาเดิม (fallback) ระหว่างมันออก แล้วย้ายเนื้อหาจาก `<template>` มาแทนที่
— ทั้งหมดนี้เกิดขึ้น**ใน HTTP response เดียวกัน** ไม่ใช่การยิง request ใหม่ นี่คือเหตุผลที่ต้องมี JavaScript
(WASM) ทำงานอยู่ฝั่ง client เพื่อ "ย้าย" เนื้อหาจาก `<template>` ไปแทนที่ fallback — ถ้าไม่มี JS เลย (เช่น
crawler บางตัวที่ไม่รัน JS) จะเห็นแค่ fallback แช่นิ่งอยู่ (แต่เนื้อหาที่แท้จริงก็ยังอยู่ในไฟล์ HTML ที่ได้รับ
ในรูป `<template>`/JSON ที่ serialize ไว้ ถ้า crawler ฉลาดพอจะ parse ได้อยู่ดี)

ยังมีอีกจุดที่ควรอธิบายจาก raw response ข้างบน: บรรทัดสุดท้าย `__RESOLVED_RESOURCES[0] = "[{\"id\":1,...`
คือค่าที่ `Resource` คำนวณได้ (รายการหนังสือทั้งหมด) ที่ถูก **serialize เป็น JSON string แล้วฝังไว้ใน HTML
เลย** — เหตุผลที่ต้องทำแบบนี้คือ: ถ้าแอปตัวนี้มี hydrate WASM จริงด้วย (ตัวอย่างนี้ตัดส่วน hydrate ออกเพื่อ
โฟกัสที่ streaming ล้วน ๆ ตามที่บอกไว้ตอนต้นหัวข้อ 91.4) ตอน hydrate ฝั่ง client ที่มี `Resource` ตัวเดียวกันนี้
**จะไม่ยิง HTTP request ไปขอข้อมูลซ้ำอีกรอบ** — มันจะอ่านค่าที่ server serialize ไว้ใน `__RESOLVED_RESOURCES`
ตรงนี้แทน (คล้ายกับที่ Part 89 หัวข้อ 89.6 อธิบายเรื่อง server function ที่ serialize ค่าไปมาระหว่าง client/
server ผ่าน `Serialize`/`Deserialize`) นี่คือ optimization ที่สำคัญมาก: ไม่มี "double fetch" (fetch ครั้งแรก
ตอน server render, fetch ครั้งที่สองตอน client hydrate) เกิดขึ้นเลย — ข้อมูลถูกส่งมาครั้งเดียวพร้อมกับ HTML
เอง

#### `SsrMode` ห้าแบบ: streaming ไม่ใช่ตัวเลือกเดียว

`OutOfOrder` (ค่า default ที่ตัวอย่างข้างบนใช้อยู่) เป็นแค่หนึ่งในห้าโหมดที่ `leptos_router`/`leptos_axum`
รองรับ ไล่ดูจริงจากซอร์สโค้ดของ `leptos_axum-0.8.10/src/lib.rs` (ส่วนที่ match กับ `listing.mode()` ตอน
register route) พบว่ามีทางเลือกครบดังนี้:

| `SsrMode` | ฟังก์ชันที่เรียกจริง | พฤติกรรม |
|---|---|---|
| `OutOfOrder` (default) | `render_app_to_stream_with_context` | ส่ง shell พร้อม fallback ก่อนทันที แต่ละ `<Suspense>` ที่ resolve เสร็จจะถูกส่งเป็น chunk ตามมา **ไม่เรียงตามลำดับที่ปรากฏใน `view!`** — อันที่พร้อมก่อนได้ส่งก่อน (ตามที่แบบฝึกหัดข้อ 3 ให้ลองพิสูจน์) |
| `InOrder` | `render_app_to_stream_in_order_with_context` | เหมือน OutOfOrder แต่ **รอให้ `<Suspense>` resolve ตามลำดับที่ปรากฏใน view!** เท่านั้น (Suspense ตัวที่สองต้องรอตัวแรกเสร็จก่อนเสมอ แม้จะพร้อมก่อนก็ตาม) — ใช้เมื่อลำดับการแสดงผลสำคัญกว่าความเร็ว |
| `PartiallyBlocked` | `render_app_to_stream_with_context_and_replace_blocks` | คล้าย OutOfOrder แต่ "block" การส่ง shell ไว้จนกว่า Suspense ที่ระบุไว้เป็นพิเศษจะพร้อม — ใช้เมื่อมีเนื้อหาสำคัญที่ไม่อยากให้ผู้ใช้เห็น fallback เลยแม้แต่แว๊บเดียว |
| `Async` | `render_app_async_with_context` | **รอทุกอย่างพร้อมก่อนส่ง response แม้แต่ byte แรก** — ไม่มี streaming เลย เหมือนย้อนกลับไปพฤติกรรมแบบ SSR ดั้งเดิมที่สุด เหมาะกับ crawler ที่ไม่รัน JS แน่ ๆ (ไม่ต้องกังวลว่า fallback จะถูก index ผิด ๆ) |
| `Static(_)` | `handle_static_route` | ต้อง compile ด้วย feature `default` (ไม่ใช่ WASM32 server target) — pre-render หน้าไว้แบบ SSG จริง ๆ ในระดับ route (ใกล้เคียงกับที่หัวข้อ 91.9 ทำเองด้วยมือ แต่นี่คือ built-in mechanism ของ Leptos เอง) |

ตัวอย่างในบทนี้ (หัวข้อ 91.4-91.5) ไม่ได้ตั้ง `SsrMode` เจาะจง เพราะเรียก `render_app_to_stream`/
`render_app_to_stream_with_context` ตรง ๆ โดยไม่ผ่าน `leptos_routes_with_context()` (route listing) — แต่
รู้ไว้ว่าถ้าใช้ `leptos_routes_with_context` แบบมี route list (เหมือนที่ Part 89 หัวข้อ 89.8 ทำ) จะเลือก
`SsrMode` ได้ต่อ route ผ่าน attribute ของ `<Route>` ใน `leptos_router` — เป็นการตัดสินใจระดับ **รายหน้า**
ได้ละเอียดกว่าที่คิด ไม่ใช่ต้องเลือกโหมดเดียวให้ทั้งแอป

### 91.6 SEO และ Meta Tags: ทำไม SSR สำคัญกับ Crawler

เหตุผลหนึ่งที่ SSR ถูกพูดถึงคู่กับ SEO เสมอคือ: **search engine crawler และ social media bot จำนวนมาก
ไม่รัน JavaScript** (หรือรันแบบมีข้อจำกัดมาก) Googlebot สมัยใหม่รัน JavaScript ได้ในระดับหนึ่งก็จริง แต่ crawler
อื่น ๆ อีกมาก (Facebook/Twitter/LinkedIn link preview bot, crawler ของ search engine เล็กกว่า, เครื่องมือ
monitoring, screen reader บางตัว) **อ่านแค่ HTML ดิบที่ได้จาก response แรกเท่านั้น** ถ้าแอปเป็น CSR ล้วน ๆ
สิ่งที่ crawler เหล่านี้เห็นคือ `<div id="root"></div>` เปล่า ๆ — ไม่มี title ที่สื่อความหมาย ไม่มี description
ไม่มีเนื้อหาให้ index เลย

Leptos มี crate แยกชื่อ `leptos_meta` สำหรับจัดการ `<title>`/`<meta>` แบบ **reactive และ per-route** — component
`<Title>` และ `<Meta>` สามารถวางไว้ที่ไหนก็ได้ในต้นไม้ component (ไม่ต้องอยู่ใน `<head>` โดยตรง) แล้ว Leptos
จะ "ย้าย" มันไปวางใน `<head>` ให้เองตอน render จริง ผ่าน `<MetaTags/>` marker ที่ต้องวางไว้ใน `<head>` ครั้งเดียว

หัวข้อ 91.4 ได้แสดงตัวอย่างนี้ไปแล้วบางส่วน มาดูให้ชัดว่า route ต่างกันให้ meta tag ต่างกันจริง โดยเทียบ
`/books` กับ `/about` (ที่จะสร้างในหัวข้อ 91.9):

```
$ curl -sS http://127.0.0.1:3092/books | grep -oE '<title>[^<]*</title>|content="[^"]*"'
<title>รายการหนังสือ - ห้องสมุด Rust Course</title>
content="รายชื่อหนังสือ Rust ทั้งหมดในห้องสมุด พร้อมจำนวนที่ยืมได้ตอนนี้ — เรนเดอร์เป็น HTML
เต็มรูปแบบตั้งแต่ request แรกด้วย Leptos SSR"

$ curl -sS http://127.0.0.1:3092/about | grep -oE '<title>[^<]*</title>|content="[^"]*"'
<title>เกี่ยวกับห้องสมุด</title>
content="ข้อมูลทั่วไปเกี่ยวกับห้องสมุด Rust Course — หน้านี้ถูก pre-render ไว้ล่วงหน้าเพียงครั้งเดียว"
```

สอง route คืนค่า `<title>`/`<meta name="description">` **ต่างกันจริง** ตรงตามที่กำหนดไว้ในโค้ดของแต่ละ
component และเห็นได้แม้จาก `curl` ตรง ๆ (ซึ่งไม่รัน JS) — นี่คือสิ่งที่ social media bot/search engine crawler
จะเห็นเป๊ะ ๆ เช่นเดียวกัน ต่างจาก CSR ที่ทุก route จะคืน `<title>` เดียวกัน (title ของ `index.html` ตัวเดียว
ที่ตั้งไว้ตอน build) เพราะ routing ทั้งหมดเกิดขึ้นฝั่ง client หลังจาก JS โหลดเสร็จแล้ว

ข้อควรระวังเชิงปฏิบัติ: `leptos_meta` จัดการแค่ `<title>`/`<meta>`/`<link>`/`<style>`/`<script>` ที่อยู่ใน
`<head>` — มันไม่ได้ทำให้แอป "SEO-friendly" แบบสมบูรณ์อัตโนมัติ (ยังต้องคิดเรื่อง semantic HTML, `alt` ของ
รูปภาพ, structured data แบบ JSON-LD ถ้าต้องการ rich snippet, sitemap.xml ฯลฯ เอง) — สิ่งที่มันแก้ได้ตรง ๆ
คือปัญหาที่ CSR แก้ไม่ได้เลยคือ "crawler ที่ไม่รัน JS เห็นหน้าเปล่า"

#### กลไกจริงเบื้องหลัง `<MetaTags/>`: ไม่ใช่ DOM patch แต่เป็น "string surgery"

สังเกตจาก raw HTML ที่ `curl` ได้ในหัวข้อ 91.4 ว่ามี comment แปลก ๆ โผล่มา: `<!--HEAD-->` — นี่ไม่ใช่ noise
แต่คือกลไกจริงของ `<MetaTags/>` ไล่ดูซอร์สโค้ดจริงของ `leptos_meta-0.8.7/src/lib.rs` พบว่า `<MetaTags/>`
render ออกมาเป็น **string literal ธรรมดา** ตรง ๆ:

```rust
// จากซอร์สโค้ดจริง — leptos_meta-0.8.7/src/lib.rs
fn to_html_with_buf(self, buf: &mut String, ...) {
    buf.push_str("<!--HEAD-->");
}
```

ส่วน `<Title>`/`<Meta>` ที่ถูกวางไว้ *ที่ไหนก็ได้* ในต้นไม้ component (ไม่จำเป็นต้องอยู่ใน `<head>` เลย — สังเกต
ในหัวข้อ 91.4 ที่วางไว้ข้าง ๆ `<meta charset="utf-8">` ใน `<head>` ก็จริง แต่หลักการเดียวกันนี้ใช้ได้แม้วางไว้
ลึกใน component tree) จะแค่ "ลงทะเบียน" ตัวเองไว้ใน `MetaContext` แทนที่จะ render HTML ตรงตำแหน่งนั้นทันที
จากนั้นก่อนที่ chunk แรกของ response จะถูกส่งออกไปจริง ๆ leptos_meta จะทำ **string surgery** บน HTML buffer
ที่ render ไว้แล้ว — หา `<!--HEAD-->` marker ตำแหน่งไหนก็ตาม แล้วแทรก `<title>`/`<meta>` ที่รวบรวมมาได้ทั้งหมด
เข้าไป**ตรงจุดนั้น**ก่อนส่ง (ตรวจสอบจริงจากซอร์สโค้ด):

```rust
// จากซอร์สโค้ดจริง — leptos_meta-0.8.7/src/lib.rs (ย่อ)
let marker_loc = first_chunk
    .find("<!--HEAD-->")
    .map(|pos| pos + "<!--HEAD-->".len())
    .unwrap_or_else(|| first_chunk.find("</head>").unwrap_or(head_loc));
let (before_marker, after_marker) = first_chunk.split_at_mut(marker_loc);
buf.push_str(before_marker);
buf.push_str(&meta_buf);   // <meta> ทั้งหมดที่ลงทะเบียนไว้จากทุกที่ในต้นไม้ component
if let Some(title) = title {
    buf.push_str("<title>");
    buf.push_str(&title);
    buf.push_str("</title>");
}
```

นี่คือเหตุผลเชิงกลไกที่ทำให้ `<Title>`/`<Meta>` "ยืดหยุ่น" ได้มากกว่าที่คิด — component ลูกที่อยู่ลึกมาก (เช่น
component แสดงรายละเอียดหนังสือที่ซ่อนอยู่หลาย layer) สามารถกำหนด `<Title>` ของทั้งหน้าได้เลย โดยไม่ต้องส่ง
ค่ากลับขึ้นไปให้ component แม่แล้วให้แม่เป็นคนวางใน `<head>` เอง — มันคือ pattern ที่คล้าย "context ที่เขียน
ได้จากทุกที่" มากกว่า "prop ที่ต้องส่งขึ้นไปข้างบน"

### 91.7 Performance จริง: TTFB เทียบกับ TTI

หัวข้อนี้จะเจาะประเด็นที่ถูกพูดง่ายเกินไปบ่อยที่สุดในบทความเกี่ยวกับ SSR: **"SSR เร็วกว่า CSR"** — ประโยคนี้
ถูกครึ่งเดียว ต้องแยกให้ชัดระหว่างสอง metric ที่คนละเรื่องกันโดยสิ้นเชิง:

- **Time-To-First-Byte (TTFB)**: เวลาตั้งแต่ browser ส่ง request จนได้รับ byte แรกของ response กลับมา —
  วัด "server ตอบสนองเร็วแค่ไหน" เท่านั้น ไม่เกี่ยวกับว่าเนื้อหานั้นมี "อะไร" อยู่ในนั้น
- **Time-To-Interactive (TTI)**: เวลาตั้งแต่ผู้ใช้เริ่มโหลดหน้า จนกระทั่งหน้านั้น **ตอบสนองต่อ interaction
  ได้จริง** (กดปุ่มได้, กรอกฟอร์มได้) — วัด "ผู้ใช้ใช้งานแอปได้จริงเมื่อไหร่"

**SSR ช่วย TTFB ในความหมายที่ว่า HTML ที่ได้กลับมามีเนื้อหาจริงทันที** (พิสูจน์แล้วในหัวข้อ 91.4) — ผู้ใช้
เห็น**อะไรบางอย่าง**บนหน้าจอเร็วขึ้นมาก เทียบกับ CSR ที่ต้องรอ WASM bundle โหลด+parse+execute+fetch ข้อมูล
ก่อนถึงเห็นเนื้อหาจริง วัดจริงด้วย `curl` เทียบระหว่างหน้า SSR (`/books`) กับหน้า CSR-shell เปล่า (`/csr-shell`
— จำลอง HTML ที่แอป CSR ล้วน ๆ แบบ Part 88-89 จะส่งมา):

```
$ curl -sS -o /dev/null -w "TTFB=%{time_starttransfer}s total=%{time_total}s size=%{size_download} bytes\n" \
    http://127.0.0.1:3092/csr-shell
TTFB=0.000908s total=0.000988s size=182 bytes

$ curl -sS -o /dev/null -w "TTFB=%{time_starttransfer}s total=%{time_total}s size=%{size_download} bytes\n" \
    http://127.0.0.1:3092/books
TTFB=0.002513s total=0.002601s size=1541 bytes
```

ต้องอ่านตัวเลขนี้อย่างซื่อสัตย์: **บน localhost ที่ไม่มี network latency เลย TTFB ของทั้งสองหน้าต่างกันแค่
ไม่ถึง 2 มิลลิวินาที** — ตัวเลขระดับนี้ไม่มีความหมายในโลกจริงเลย เพราะ network latency จริงบนอินเทอร์เน็ต
(หลายสิบถึงหลายร้อยมิลลิวินาที) จะครอบงำความต่างเล็กน้อยนี้จนไม่มีนัยสำคัญ **สิ่งที่ต่างกันจริงและสำคัญกว่า
มากคือ `size` และเนื้อหาข้างใน**: `/csr-shell` ส่งมาแค่ 182 bytes ที่ไม่มีข้อมูลหนังสือแม้แต่ตัวเดียว (มีแค่
`<div id="root"></div>` เปล่า ๆ) ส่วน `/books` ส่งมา 1541 bytes ที่มีรายชื่อหนังสือครบทุกเล่มจริง — **TTFB
ที่ใกล้เคียงกันไม่ได้แปลว่า "ประสบการณ์ผู้ใช้เหมือนกัน"** เพราะ TTFB วัดแค่ "byte แรกมาถึงเมื่อไหร่" ไม่ได้
วัดว่า "เนื้อหาที่มีความหมายปรากฏบนหน้าจอเมื่อไหร่" (metric ที่ตรงประเด็นกว่าคือ **First Contentful Paint**
ซึ่ง SSR จะชนะ CSR ขาดลอยเสมอ เพราะเนื้อหาที่ "มีความหมาย" (มีตัวหนังสือ) มาถึงพร้อม TTFB เลย ในขณะที่ CSR ต้อง
รอขั้นตอนดาวน์โหลด+parse+execute WASM ก่อน ซึ่งมักกินเวลาหลักร้อยมิลลิวินาทีถึงวินาทีในเครื่องช้าหรือเน็ตช้า)

ส่วนที่ **SSR ไม่ได้แก้เลย** คือ **Time-to-Interactive** — แม้ HTML จะมาถึงพร้อมเนื้อหาเต็มตั้งแต่ต้น
หน้านั้นจะ "กดปุ่มได้จริง" ก็ต่อเมื่อ hydration เสร็จสมบูรณ์ (หัวข้อ 91.2) ซึ่งยังต้องดาวน์โหลด, parse, และ
execute ไฟล์ WASM เหมือนเดิม — **ปุ่มบนหน้า SSR ทุกปุ่มกดไม่ได้เลยจนกว่า hydration จะเสร็จ** เหมือนกับ CSR
ทุกประการในแง่นี้ ความต่างมีแค่ว่า **ผู้ใช้เห็นเนื้อหา (แต่กดไม่ได้) ระหว่างที่รอ** เทียบกับ CSR ที่**ไม่เห็น
อะไรเลยและกดไม่ได้**ระหว่างรอ — ทั้งสองกรณี WASM bundle ขนาดเท่าเดิมต้องถูกดาวน์โหลดและ execute เหมือนกันหมด
ระยะเวลารอ TTI จึงใกล้เคียงกันมาก (แถมยิ่งซับซ้อนกว่าเดิมนิดหน่อยฝั่ง SSR เพราะต้องมีขั้นตอน hydration ที่
ตรวจสอบ DOM เดิมด้วย ไม่ใช่แค่สร้าง DOM ใหม่แบบ CSR)

ถ้าเคยใช้เครื่องมือวัด performance อย่าง Lighthouse หรือ Chrome DevTools' Performance panel จะคุ้นกับชื่อ
metric มาตรฐานที่ Google เรียกว่า **Core Web Vitals** — **Largest Contentful Paint (LCP)** คือตัวที่ตรงกับ
สิ่งที่ SSR ช่วยได้ตรง ๆ (เวลาที่ "ก้อนเนื้อหาที่ใหญ่ที่สุดที่มองเห็นได้" ปรากฏบนหน้าจอ) ส่วน **Total Blocking
Time (TBT)**/**Interaction to Next Paint (INP)** คือตัวที่สะท้อนปัญหาฝั่ง TTI/hydration cost ที่ SSR ช่วย
ไม่ได้เลย — เวลาเปิดโปรเจกต์จริงแล้ววัดด้วยเครื่องมือเหล่านี้ ควรอ่านผลแยกสอง metric นี้ให้ชัดเสมอ ไม่ใช่ดูแค่
ตัวเลขคะแนนรวม (Performance score) ตัวเดียว เพราะคะแนนรวมของ SSR app ที่มี WASM bundle ใหญ่มาก อาจ**ดูดีด้าน
LCP แต่แย่ด้าน TBT** พร้อมกันได้ในหน้าเดียว — ซึ่งเป็นสัญญาณตรงตามที่บทนี้อธิบายไว้ทุกประการ

สรุปให้แม่นยำที่สุด: **SSR ปรับปรุง "เวลาที่ผู้ใช้เห็นเนื้อหาที่มีความหมาย" (คล้าย First Contentful Paint)
แต่ไม่ได้ลดเวลาที่ต้องใช้ในการดาวน์โหลด/parse/execute WASM bundle เพื่อให้หน้าโต้ตอบได้ (Time-to-Interactive)
เลยแม้แต่นิดเดียว** — ทั้งสองอย่างเป็นปัญหาคนละเรื่อง ต้องแก้คนละวิธี (SSR แก้เรื่องแรก, การลดขนาด WASM
bundle/code splitting แก้เรื่องหลัง ซึ่งเป็นหัวข้อที่ Part 86-87 แนะนำไว้บ้างแล้วเรื่อง `wasm-opt`)

#### ตัวเลขจริงของ "ต้นทุน TTI": ไฟล์ WASM ที่ต้องดาวน์โหลดมีขนาดเท่าไหร่

เพื่อให้ "ต้นทุนของ TTI" ไม่ใช่แค่คำพูดลอย ๆ กลับไปดูไฟล์ WASM ที่ build จริงจากตัวอย่าง hydration mismatch
ในหัวข้อ 91.2:

```
$ ls -la pkg/
-rw-r--r-- 1 root root   29684 mismatch_app.js
-rw-r--r-- 1 root root  695421 mismatch_app_bg.wasm
```

**695 KB** สำหรับ component ที่มีแค่ `<h1>`, `<p>` สองอัน, และเงื่อนไข `#[cfg]` ง่าย ๆ — นี่คือ **debug build
ที่ไม่ได้ optimize เลย** (ไม่มี `wasm-opt`, ไม่มี `--release`) ตัวเลขนี้ยังไม่รวม reactive runtime ของ Leptos
เองที่ผูกมาด้วย (`reactive_graph`, `tachys`) แอปจริงที่มี component จำนวนมากกว่านี้และไม่ได้ optimize ขนาด
WASM เลย อาจมีขนาดหลาย MB ได้ง่าย ๆ — เบราว์เซอร์ต้อง**ดาวน์โหลดไฟล์ขนาดนี้ทั้งไฟล์, parse เป็น internal
representation, แล้ว execute เพื่อเริ่ม hydration** ก่อนหน้าเว็บจะกดอะไรได้เลย ไม่ว่าจะ SSR หรือ CSR ก็ตาม
ต้องแบกต้นทุนนี้เท่ากันเป๊ะ — นี่คือตัวเลขที่ทำให้ "SSR เร็วกว่า CSR" เป็นประโยคที่พูดง่ายเกินไปอย่างเป็น
รูปธรรม: ตัวเลข TTFB ที่ต่างกันแค่ไม่ถึง 2 มิลลิวินาที (หัวข้อก่อนหน้า) เทียบกับต้นทุน TTI ที่วัดเป็นร้อย ๆ
กิโลไบต์ที่ต้องดาวน์โหลด/parse/execute เท่ากันทั้งสองแนวทาง — ถ้าจะปรับปรุง TTI จริง ๆ ต้องไปแก้ที่ขนาด WASM
bundle (ผ่าน `wasm-opt -Oz`, code splitting, หรือลด dependency) ไม่ใช่ไปแก้ที่ SSR/CSR

### 91.8 Deployment: SSR ต้องมี Server รันอยู่ตลอดเวลา

ความต่างเชิง deployment ที่สำคัญที่สุดระหว่าง CSR/SSG กับ SSR คือ: **แอป CSR/SSG ล้วน ๆ คือไฟล์ static ชุดหนึ่ง
(`.html`/`.js`/`.wasm`) ที่วางไว้บน CDN หรือ static file host (เช่น GitHub Pages, S3+CloudFront, Netlify)
ได้เลยโดยไม่ต้องมีโค้ด server ฝั่งเราเองรันอยู่เลย** — แต่ **แอป SSR ต้องมี process ที่รันโค้ด Rust ของเรา
อยู่ตลอดเวลา** เพื่อรอรับ request แล้ว render HTML ใหม่ทุกครั้ง

ตามที่ Part 89 หัวข้อ 89.8 และบทนี้หัวข้อ 91.4-91.5 พิสูจน์ไว้ตลอด: **Leptos SSR ไม่ใช่ web server ของตัวเอง
มันคือชุด handler ที่เสียบเข้า `axum::Router` เดียวกับที่ Module 4 (Part 62-66) สอนไว้ทุกประการ** — นั่น
แปลว่าการ deploy Leptos SSR app คือการ deploy **Axum server ธรรมดา** ตัวหนึ่ง ที่ต้องมี:

- **process ที่รันตลอดเวลา** (ไม่ใช่ serverless function ที่ตื่นมาทำงานแค่ตอนมี request แบบ AWS Lambda ทั่วไป
  — แม้จะมีแนวทาง deploy Leptos บน serverless ได้เหมือนกัน แต่ซับซ้อนกว่าการรัน process ปกติมาก เพราะต้อง
  cold-start ทุกครั้งซึ่งขัดกับจุดแข็งของ SSR เรื่อง TTFB โดยตรง)
- **การเชื่อมต่อฐานข้อมูล** ที่ต้องคง `PgPool` ไว้ตลอดอายุของ process (เหมือนที่ Part 89 หัวข้อ 89.8 ทำ) ไม่ใช่
  เปิด-ปิด connection ใหม่ทุก request
- **ทรัพยากร CPU/memory ที่เพียงพอสำหรับ render ทุก request** — ต่างจาก static file ที่ CDN เสิร์ฟได้โดยไม่
  ต้องใช้ CPU ของเราเลย (แค่ bandwidth) การ render HTML ทุก request ใช้ CPU จริงของ process เรา ถ้า traffic
  สูงมากต้อง scale server ให้พอ (horizontal scaling ด้วยหลาย instance หลัง load balancer — เรื่องนี้จะกลับมา
  ในบทที่พูดถึง production deployment)

นี่คือเหตุผลที่ **Part 96 (Docker และการ Deploy จริง)** จะเป็นบทที่สอนวิธี package Axum/Leptos SSR server
ของเราให้เป็น container image ที่รันได้จริงบน production — เพราะ "process ที่รันตลอดเวลา" ต้องมีสภาพแวดล้อม
ที่เสถียร reproducible และ scale ได้ ซึ่งคือปัญหาที่ container/orchestration เข้ามาแก้ ถ้าคุณจำได้ว่า Part 89
หัวข้อ 89.8 รัน SSR server ด้วย `cargo run` ตรง ๆ บนเครื่อง dev — Part 96 จะสอนว่าขั้นตอนถัดไปจากนั้น (build
release binary, สร้าง Docker image ขนาดเล็กด้วย multi-stage build, ตั้งค่า environment variable สำหรับ
connection string ของฐานข้อมูล production) ต้องทำอย่างไรถึงจะเอา process เดียวกันนี้ไปรันบน server จริงได้
อย่างปลอดภัยและเชื่อถือได้

#### ภาพประกอบ: หน้าตาคร่าว ๆ ของสิ่งที่ Part 96 จะลงรายละเอียด

เพื่อให้เห็นภาพว่า "process ที่รันตลอดเวลา" แปลว่าต้องเตรียมอะไรบ้างในทางปฏิบัติ (รายละเอียดเต็มอยู่ใน Part
96 บทนี้ให้แค่ภาพร่างกว้าง ๆ — Dockerfile นี้เป็น**ภาพประกอบเพื่อความเข้าใจเท่านั้น ไม่ได้ build/รันจริงใน
บทนี้**):

```dockerfile
# ภาพประกอบเท่านั้น — รายละเอียดจริง (multi-stage build ที่ compile ทั้ง
# native binary และ WASM bundle, การจัดการ secret, ฯลฯ) อยู่ใน Part 96
FROM rust:1.94 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release --features ssr

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/books_app /usr/local/bin/books_app
# DATABASE_URL ต้องมาจาก environment variable ตอน deploy จริง ไม่ใช่ hardcode
# ในอิมเมจ (เทียบกับกับดักที่พบบ่อยข้อ 5 ที่พูดถึงตอน compile time)
ENV DATABASE_URL=""
EXPOSE 3092
CMD ["books_app"]
```

สังเกตความต่างที่สำคัญจากไฟล์ static ล้วน ๆ: **image นี้ยังต้องมี process `books_app` รันอยู่ข้างในตลอดเวลา
หลัง container start** — ต่างจาก CSR/SSG ที่ image (ถ้าจะทำเป็น container เลย) แค่ต้องมี web server ธรรมดา
ที่สุด (เช่น `nginx`) คอยเสิร์ฟไฟล์ static เฉย ๆ ไม่ต้องรันโค้ด Rust ของเราเลยด้วยซ้ำ — หรือแม้แต่ไม่ต้องมี
container เลยก็ได้ถ้าใช้ static host อย่าง GitHub Pages ตรง ๆ ความต่างนี้สะท้อนอยู่ใน "ความซับซ้อนของ
deploy" ที่ตารางหัวข้อ 91.1 ระบุไว้ตั้งแต่ต้นบท และเป็นเหตุผลที่ต้องมี reverse proxy (เช่น `nginx`/`Caddy`
วางไว้หน้า Axum) จัดการ TLS/load balancing ในระดับ production จริง ซึ่ง Part 96 จะลงรายละเอียดเต็มรูปแบบ

### 91.9 แนวทาง Hybrid: ผสม SSG กับ SSR/CSR ในระบบเดียว

หัวข้อ 91.1 บอกไว้แล้วว่าแอปจริงไม่ต้องเลือกแนวทางเดียวสำหรับทั้งแอป — ตัวอย่างที่ชัดเจนที่สุดคือ **เนื้อหาที่
เปลี่ยนไม่บ่อย** (เช่น หน้า "เกี่ยวกับเรา", หน้ารายละเอียดหนังสือที่ข้อมูล title/author ไม่เปลี่ยนบ่อย) ควรใช้
แนวทางแบบ SSG (render ครั้งเดียว เสิร์ฟซ้ำได้เรื่อย ๆ) ส่วน **เนื้อหาที่เปลี่ยนตลอดเวลา** (เช่น จำนวนที่นั่ง
ที่เหลือแบบ real-time ในโดเมนที่ Part 89 ใช้ หรือจำนวนหนังสือที่ยืมได้ตอนนี้) ต้องใช้ SSR (render ใหม่ทุก
request) หรือ CSR (fetch ใหม่ฝั่ง client เป็นระยะ)

มาสร้างตัวอย่างที่พิสูจน์แนวคิดนี้จริง — เพิ่ม route `/about` เข้าไปในโปรเจกต์ `books_app` จากหัวขัด 91.4 ที่
**render ด้วย Leptos renderer ตัวเดียวกัน แต่เรียกแค่ครั้งเดียวตอน server เริ่มทำงาน** แล้ว cache HTML
string ที่ได้ไว้ตอบทุก request ต่อจากนั้น — จำลองพฤติกรรมของ SSG (ปกติ SSG ทำตอน build time ด้วยเครื่องมือ
build แยกต่างหาก แต่หลักการ "render ครั้งเดียว เสิร์ฟซ้ำ" เหมือนกันเป๊ะ แค่ทำในโปรเซสเดียวกับ server แทน):

```rust
#[component]
fn AboutShell(total_books_at_startup: i64) -> impl IntoView {
    view! {
        <html lang="th">
            <head>
                <meta charset="utf-8" />
                <title>"เกี่ยวกับห้องสมุด"</title>
                <meta name="description" content="ข้อมูลทั่วไปเกี่ยวกับห้องสมุด — หน้านี้ถูก pre-render ไว้ล่วงหน้าเพียงครั้งเดียว" />
            </head>
            <body>
                <h1>"เกี่ยวกับห้องสมุด Rust Course"</h1>
                <p>
                    "จำนวนหนังสือทั้งหมดในห้องสมุด (ณ เวลาที่ server เริ่มทำงาน): "
                    {total_books_at_startup} " เล่ม"
                </p>
                <p>
                    "ตัวเลขนี้ถูก \"แช่แข็ง\" ไว้ตอน server start — ต่างจากหน้า /books "
                    "ที่ query ฐานข้อมูลใหม่ทุกครั้งที่มี request เข้ามา"
                </p>
            </body>
        </html>
    }
}
```

ใน `main()` เรียก render ครั้งเดียวตอน startup โดยส่ง `Request` จำลองเข้าไปให้ handler function ตัวเดียวกัน
กับที่ใช้ตอบ request จริง (พิสูจน์ว่าเป็น renderer ตัวเดียวกันเป๊ะ ไม่ใช่เครื่องมือคนละชุด):

```rust
let total_books_at_startup: i64 = sqlx::query_scalar!("SELECT count(*) FROM books")
    .fetch_one(&pool)
    .await
    .expect("count books")
    .unwrap_or(0);

let about_handler_fn = leptos_axum::render_app_to_stream(
    move || view! { <AboutShell total_books_at_startup /> },
);
let dummy_req = Request::builder().method("GET").uri("/about").body(Body::empty()).unwrap();
let about_response = about_handler_fn(dummy_req).await;
let about_bytes = axum::body::to_bytes(about_response.into_body(), usize::MAX).await.unwrap();
let about_html: String = String::from_utf8(about_bytes.to_vec()).unwrap();

let app = Router::new()
    .route("/books", get(books_handler))
    .route("/about", get(move || { let html = about_html.clone(); async move { Html(html) } }))
    // ...
    .with_state(pool);
```

พิสูจน์ความ hybrid นี้ด้วยการทดลองจริง: เพิ่มหนังสือใหม่เข้า database **ระหว่างที่ server ยังรันอยู่** แล้ว
เทียบผลจากทั้งสอง route:

```
$ psql "postgres://postgres:postgres@localhost/leptos_scratch" \
    -c "INSERT INTO books (title, author, total_copies, available_copies) \
        VALUES ('Rust Atomics and Locks', 'Mara Bos', 2, 2);"
INSERT 0 1

$ curl -sS http://127.0.0.1:3092/books | grep -o "Rust Atomics and Locks"
Rust Atomics and Locks

$ curl -sS http://127.0.0.1:3092/about | grep -oP '(?<=server เริ่มทำงาน\): <!>)\d+'
4
```

ผลลัพธ์นี้พิสูจน์ hybrid pattern ได้ตรง ๆ: **`/books` เห็นหนังสือเล่มใหม่ทันทีในการ request ครั้งต่อไป**
เพราะมัน query database สดใหม่ทุกครั้ง (SSR แท้ ๆ) ส่วน **`/about` ยังแสดง "4" เล่มเหมือนเดิม** (ไม่รวมเล่ม
ใหม่ที่เพิ่งเพิ่ม) เพราะมันถูก render ครั้งเดียวตอน server start และ cache ไว้ (พฤติกรรมแบบ SSG) — ทั้งสอง
route รันอยู่ใน **Axum process เดียวกัน ใช้ renderer function เดียวกัน (`render_app_to_stream`)** แต่ให้
พฤติกรรมต่างกันโดยสิ้นเชิงตามที่เราตั้งใจออกแบบไว้ นี่คือตัวอย่างที่เป็นรูปธรรมที่สุดว่า "SSG กับ SSR ไม่ใช่
ของที่ต้องเลือกแยกกันทั้งแอป" — เลือกได้เป็นรายหน้า หรือแม้แต่รายส่วนของหน้าเดียวกัน (ผสมกับเทคนิค streaming
จากหัวข้อ 91.5 ก็ได้ เช่น หน้ารายละเอียดหนังสือที่ title/author render แบบ static แต่ availability ห่อด้วย
`<Suspense>` fetch สดทุกครั้ง)

ต้องซื่อสัตย์ว่าเทคนิคที่ใช้ใน `/about` ("render ครั้งเดียวตอน process start แล้ว cache ไว้ในหน่วยความจำ")
**ไม่ใช่ SSG แบบเดียวกับที่ `mdBook` ทำ** เป๊ะ ๆ (ที่หัวข้อ 91.1 พูดถึงไปแล้ว) — `mdBook build` render ไฟล์
`.html` ออกมาเป็น**ไฟล์จริงบนดิสก์** ตอน build time (แยกขั้นตอนจาก process ที่เสิร์ฟ request โดยสิ้นเชิง
อาจไม่มี process รันอยู่เลยตอนเสิร์ฟจริงถ้าใช้ static host) ในขณะที่เทคนิคของบทนี้ render เก็บไว้เป็น `String`
**ในหน่วยความจำของ process เดียวกันที่เสิร์ฟ SSR route อื่น ๆ ด้วย** — ถ้า process ตายแล้วเริ่มใหม่ ต้อง
render ใหม่ทุกครั้ง (ต่างจากไฟล์ `.html` ของ `mdBook` ที่อยู่ถาวรบนดิสก์ไม่ว่า process จะเปิด-ปิดกี่รอบ)
บทนี้เลือกสาธิตด้วยวิธีนี้เพราะมันแสดงหลักการ "render ครั้งเดียว เสิร์ฟซ้ำ" ได้ชัดโดยไม่ต้องแยกขั้นตอน build
ออกจาก Axum process เลย — เหมาะกับการเรียนรู้แนวคิด แต่ในระบบ production จริงที่ต้องการ SSG แท้ ๆ (เช่น
เอกสารที่ deploy บน CDN แยกจาก backend เลย) ควรแยก build step ออกมาต่างหากจริง ๆ ไม่ใช่ฝากไว้ใน `main()`
ของ server แบบนี้

ข้อควรระวังของแนวทางนี้ในโลกจริง: cache ที่ทำแบบ manual นี้ **ไม่มีวัน invalidate เองอัตโนมัติ** — ถ้าข้อมูล
ที่ใช้สร้างหน้า "เกี่ยวกับ" เปลี่ยนจริง ๆ (เช่นเปลี่ยนชื่อห้องสมุด) วิธีเดียวที่จะอัปเดตคือ **restart server**
ระบบ SSG ระดับ production จริง (เช่น Next.js ISR, หรือ build pipeline ที่ trigger rebuild อัตโนมัติเมื่อ
เนื้อหาเปลี่ยนใน CMS) มีกลไก invalidation ที่ซับซ้อนกว่านี้มาก — ตัวอย่างในบทนี้ทำให้เห็นแค่หลักการพื้นฐาน
("render ครั้งเดียว cache ไว้") ไม่ใช่ระบบ SSG แบบสมบูรณ์สำหรับ production

#### ขยายแนวคิด: cache invalidation แบบ time-based (พื้นฐานของ ISR)

ตัวอย่างในหัวข้อนี้ cache ค่า `about_html` ไว้แบบ "ตายตัว" (ต้อง restart server ถึงจะอัปเดต) — ขั้นถัดไปที่
ใกล้เคียงกับ **Incremental Static Regeneration (ISR)** ของ framework อื่น ๆ ในโลก JavaScript คือการทำให้
cache **หมดอายุอัตโนมัติตามเวลา** โดยไม่ต้อง restart process เลย แนวคิดคร่าว ๆ (ให้รายละเอียดเต็มไว้เป็น
แบบฝึกหัดข้อ 4 ท้ายบท) คือเปลี่ยนจาก `String` ตายตัว เป็น `Arc<RwLock<String>>` ที่มี background task
(`tokio::spawn`) คอย re-render ทับค่าเดิมเป็นระยะ:

```rust
// แนวคิดคร่าว ๆ (รายละเอียดเต็มเป็นแบบฝึกหัดข้อ 4)
let about_html: Arc<RwLock<String>> = Arc::new(RwLock::new(render_about(&pool).await));

let about_html_bg = about_html.clone();
let pool_bg = pool.clone();
tokio::spawn(async move {
    loop {
        tokio::time::sleep(Duration::from_secs(60)).await;
        let fresh = render_about(&pool_bg).await;
        *about_html_bg.write().await = fresh;
    }
});
```

ทุก request ที่เข้ามาที่ `/about` จะอ่านค่าล่าสุดจาก `RwLock` (เร็วมาก แค่ clone `String` ที่มีอยู่แล้ว ไม่ต้อง
query database เลย) ในขณะที่ background task เป็นตัวเดียวที่ query database และ re-render จริง ทุก ๆ 60
วินาที — ผู้ใช้จะไม่มีวัน "รอ" การ render เลยแม้แต่ request เดียว (ต่างจาก SSR เต็มรูปแบบที่ทุก request ต้อง
รอ render) แต่ข้อมูลก็ไม่ได้ "ค้าง" ตลอดไปแบบตัวอย่างเดิม (ต่างจาก SSG แท้ ๆ ที่ต้อง rebuild ใหม่ทั้งระบบ)
— นี่คือจุดกึ่งกลางที่แท้จริงระหว่าง SSG กับ SSR ในทางปฏิบัติ และเป็นเหตุผลที่ real-world framework หลายตัว
เลือกทำ ISR แทนที่จะให้เลือกแค่ SSG หรือ SSR เพียวๆ

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม init global executor เมื่อไม่ได้ผ่าน `leptos_routes_with_context`**

ถ้าเขียน Axum handler ที่เรียก `leptos_axum::render_app_to_stream(...)` ตรง ๆ (แบบหัวข้อ 91.2/91.4) โดยไม่ผ่าน
`.leptos_routes_with_context()` (ซึ่งปกติจะ init executor ให้อัตโนมัติภายใน) ต้องเรียก
`any_spawner::Executor::init_tokio()` เองก่อนเริ่ม serve request แรก ไม่งั้นจะพังทันทีที่มี request เข้ามา
ด้วย panic จริงแบบนี้ (คัดลอกจากการรันจริง):

```
thread 'tokio-rt-worker' panicked at .../any_spawner-0.3.0/src/lib.rs:146:9:
At .../any_spawner-0.3.0/src/lib.rs:146:9, tried to spawn a Future with
Executor::spawn() before a global executor was initialized.
```

วิธีแก้คือเรียก `any_spawner::Executor::init_tokio().expect("init tokio executor");` ครั้งเดียวตอนต้นของ
`main()` ก่อนสร้าง `Router` เลย (ตามที่บทนี้ทำในทุกตัวอย่าง)

**2. ให้ component ที่มี `<html>` ของตัวเองไปเป็น target ของ `hydrate_body()` ตรง ๆ**

`leptos::mount::hydrate_body(f)` เริ่ม cursor ที่ `<body>` element ที่มีอยู่แล้ว แล้วเดินไล่ **children ของ
body** — ถ้าคุณส่ง component ที่ประกาศ `<html>`/`<head>`/`<body>` ของตัวเองเข้าไปให้ `hydrate_body` ตรง ๆ
(ไม่ได้แยก Shell/App ออกจากกันแบบหัวข้อ 91.2) hydration จะพัง **แม้เนื้อหาทั้งสองฝั่งจะตรงกันเป๊ะทุกตัวอักษร
ก็ตาม** เพราะ cursor คาดหวังว่าจะเจอ content ของ body ตรง ๆ แต่กลับเจอ `<html>` element ทั้งก้อน ตัวอย่าง
error จริงที่เกิดขึ้น (ก่อนแก้เป็นโครงสร้าง Shell/App):

```
[console.error] panicked at .../tachys-0.2.19/src/html/mod.rs:217:14:
called `Option::unwrap()` on a `None` value
[pageerror] unreachable
```

วิธีแก้: แยก component ออกเป็นสองชั้นเสมอเมื่อทำ SSR+hydrate — `Shell` (มี `<html>`/`<head>`/`<body>`, render
ฝั่ง server เท่านั้น ไม่ถูก hydrate) กับ component ภายใน (เช่น `App`, เป็นเนื้อหาที่อยู่*ข้างใน* `<body>`
เท่านั้น และเป็นตัวที่ถูกส่งเข้า `hydrate_body()` จริง) — รูปแบบนี้ตรงกับ pattern มาตรฐานที่ `cargo-leptos`
สร้างให้อัตโนมัติเมื่อสร้างโปรเจกต์ผ่าน template (`shell()` function แยกจาก `App` component) ซึ่ง Part 89
หัวข้อ 89.8 ใช้อยู่แล้วโดยไม่รู้ตัวว่าทำไมต้องแยกแบบนั้น

**3. เข้าใจผิดว่า hydration mismatch ทุกแบบจะ error ออกมาดัง ๆ เสมอ**

ตามที่พิสูจน์ในหัวข้อ 91.2: ความต่างระดับ **"ชนิดของ DOM node"** (Element ↔ Text ↔ Comment) จะทำให้ Leptos
panic พร้อม console.error ชัดเจน แต่ความต่างระดับ **"identity ของ tag ภายในชนิดเดียวกัน"** (เช่น `<span>`
ฝั่ง server แต่ `<div>` ฝั่ง client) จะผ่านไปแบบ**เงียบสนิท ไม่มี warning ไม่มี error เลย** เพราะ
`Element::cast_from()` เช็คแค่ "เป็น Element หรือไม่" ไม่เช็ค tag name เจาะจง — ถ้าโค้ดของคุณมี logic ที่
`#[cfg(feature = "ssr")]` แยกกันระหว่าง element คนละชนิด (เช่นเปลี่ยน `<button>` เป็น `<a>` เพื่อ SEO บาง
กรณี) ควรตรวจสอบด้วยตาจริงในเบราว์เซอร์เสมอ ไม่ใช่แค่เชื่อว่า "ถ้ามันไม่ error แปลว่าโอเค"

**4. `AddrInUse` จาก server เก่าที่ยังรันค้างอยู่ตอนพัฒนา**

เวลาแก้โค้ดแล้ว restart server บ่อย ๆ ระหว่าง dev (เช่นสลับ mismatch ในหัวข้อ 91.2 กลับไปกลับมา) มักลืมว่า
process เก่ายังรันอยู่ (เพราะ panic เกิดขึ้นใน tokio task ไม่ใช่ main thread — main thread บล็อกอยู่ที่
`axum::serve()` เฉย ๆ ไม่ได้ crash ตามไปด้วย) แล้ว `cargo run` ตัวใหม่จะพังด้วย:

```
thread 'main' panicked at src/main.rs:21:10:
called `Result::unwrap()` on an `Err` value: Os { code: 98, kind: AddrInUse,
message: "Address already in use" }
```

วิธีแก้: `pkill -f "target/debug/<ชื่อโปรเจกต์>"` เพื่อฆ่า process เก่าก่อนรันใหม่ทุกครั้งที่ทดสอบ hydration
mismatch หรือเปลี่ยนพอร์ตให้ไม่ชนกันระหว่างพัฒนา

**5. ลืมตั้ง `DATABASE_URL` ตอน compile โค้ดที่ใช้ `sqlx::query_as!`/`query_scalar!`**

ตามที่ Part 70 สอนไว้ (และย้ำอีกครั้งใน Part 89) macro ของ SQLx ต้อง**เชื่อมต่อฐานข้อมูลจริงตอน compile time**
เพื่อตรวจสอบ SQL — ในบทนี้ที่ผสม SSR เข้ากับ SQLx (หัวข้อ 91.4/91.9) ถ้าลืม `export DATABASE_URL=...` ก่อน
`cargo build` จะได้ error จริงแบบนี้:

```
error: set `DATABASE_URL` to use query macros online, or run `cargo sqlx prepare`
  to update the query cache
  --> src/main.rs:24:5
   |
24 | /     sqlx::query_as!(
   ...
error[E0282]: type annotations needed
  --> src/main.rs:24:5
```

สังเกตว่า error ตัวที่สอง (`E0282: type annotations needed`) เป็นผลข้างเคียงจากตัวแรก — เพราะ macro ที่ควร
จะ expand เป็นโค้ดที่มี type ชัดเจน (จาก schema ของฐานข้อมูลจริง) กลับ expand ไม่สมบูรณ์เมื่อต่อฐานข้อมูล
ไม่ได้ ทำให้ compiler อนุมาน type ต่อไปไม่ได้ วิธีแก้คือตั้ง `DATABASE_URL` ให้ตรงกับฐานข้อมูลจริงก่อน build
เสมอ หรือใช้ `cargo sqlx prepare` เพื่อสร้าง query cache แบบ offline (ไม่ต้องต่อฐานข้อมูลตอน build ในเครื่อง
อื่น เช่นตอน build บน CI/CD — Part 96 จะพูดถึงเรื่องนี้อีกครั้งตอนตั้ง Docker build pipeline)

**6. ลืมเรียก `provide_meta_context()` ก่อนใช้ `<Title>`/`<Meta>`**

ตัวอย่างในหัวข้อ 91.4 เรียก `provide_meta_context()` เป็นบรรทัดแรกในทุก component ที่ใช้ `<Title>`/`<Meta>`
เสมอ — ถ้าลืมเรียกไม่ได้แปลว่า compile ไม่ผ่านหรือ panic ทันที (เพราะ `use_head()` ที่ `<Title>`/`<Meta>`
เรียกใช้ภายในมี fallback สร้าง `MetaContext` ใหม่ให้เองถ้ายังไม่มี) แต่จะได้ debug warning จริงแบบนี้ (จาก
ซอร์สโค้ดของ `leptos_meta-0.8.7/src/lib.rs`):

```
use_head() is being called without a MetaContext being provided. We'll
automatically create and provide one, but if this is being called in a
child route it may cause bugs. To be safe, you should provide_meta_context()
somewhere in the root of the app.
```

ปัญหาจริงที่ตามมาไม่ใช่ error ทันที แต่คือ **ถ้ามีมากกว่าหนึ่ง component เรียก `use_head()` แบบไม่มี
`MetaContext` ที่ provide ไว้ร่วมกัน แต่ละ component จะได้ `MetaContext` คนละตัว** — `<Title>` จาก component
หนึ่งจะไม่เห็น `<Meta>` จากอีก component เลย ทำให้ meta tag บางตัวหายไปจาก `<head>` แบบหาสาเหตุยาก วิธีแก้คือ
เรียก `provide_meta_context()` ที่ root component ของแอปเพียงครั้งเดียว (ไม่ใช่ในทุก component ลูกที่ใช้
`<Title>`/`<Meta>`) เพื่อการันตีว่าทุกจุดใน tree ใช้ `MetaContext` ตัวเดียวกัน

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม route `/books/:id` ในโปรเจกต์ `books_app` (หัวข้อ 91.4) ที่แสดงรายละเอียดหนังสือเล่มเดียว
   ตาม `id` ที่ส่งมาใน URL พร้อมตั้ง `<Title>`/`<Meta name="description">` ให้เปลี่ยนตามชื่อหนังสือเล่มนั้น
   (เช่น title = "{ชื่อหนังสือ} - ห้องสมุด Rust Course") พิสูจน์ด้วย `curl` ว่าหนังสือคนละเล่มให้ `<title>`
   คนละค่ากันจริง — *Hint*: ใช้ `axum::extract::Path<i64>` ดึง `id` จาก URL แล้ว query แค่แถวเดียวด้วย
   `WHERE id = $1` ก่อนส่งเข้า `provide_context()` เหมือนหัวข้อ 91.4 ระวังกรณี `id` ที่ไม่มีในฐานข้อมูล —
   ควรตอบ HTTP 404 กลับไปแทนที่จะ panic ตอน `.fetch_one()` ล้มเหลว

2. **(กลาง)** สร้าง hydration mismatch แบบใหม่ที่ต่างจากหัวข้อ 91.2 โดยตั้งใจให้ฝั่ง client render **list ที่
   มีจำนวน item ต่างจากฝั่ง server** (เช่น server render หนังสือ 4 เล่ม แต่ client คำนวณแค่ 3 เล่มเพราะเงื่อนไข
   `#[cfg]` ที่กรองบางเล่มออก) เปิดด้วย Playwright แล้วดูว่า console.error ที่ได้ต่างจากหัวข้อ 91.2 อย่างไร
   (เป็น "element/text mismatch" แบบเดียวกัน หรือเป็นคนละแบบ) — *Hint*: ความยาวของ list ที่ต่างกันจะทำให้
   cursor เดินไปเจอ node ที่ "ไม่มีอยู่จริง" (เกิน bound ของ children ที่ DOM มี) ลองดูว่า error message พูด
   ถึง "expected... but found" อะไร และตำแหน่งไฟล์/บรรทัดที่มันรายงานตรงกับตำแหน่งจริงของ `<li>` ที่มีปัญหา
   หรือไม่ — บันทึกผลที่ได้เทียบกับสมมติฐานของคุณก่อนลงมือทำด้วย

3. **(ยาก)** ทำ streaming SSR ในหัวข้อ 91.5 ให้มี **สอง `<Suspense>` ที่ resolve ไม่พร้อมกัน** (เช่น
   `Resource` แรก sleep 1 วินาที, `Resource` ที่สอง sleep 3 วินาที) แล้ววัดด้วย `curl` (หรือเขียนสคริปต์อ่าน
   response แบบ streaming ทีละ chunk พร้อม timestamp) ว่า chunk ของ Suspense แรกมาถึงก่อน chunk ของ Suspense
   ที่สองจริงหรือไม่ (out-of-order streaming ควรส่งอันที่พร้อมก่อนออกไปก่อน ไม่ต้องรอเรียงตามลำดับที่ปรากฏใน
   `view!`) — *Hint*: `curl --no-buffer` ช่วยให้เห็น chunk ทยอยมาแบบเรียลไทม์ใน terminal แทนที่จะรอทั้งก้อน

4. **(ประยุกต์ใช้งานจริง)** ขยายตัวอย่าง hybrid ในหัวข้อ 91.9 ให้ cache ของ `/about` **invalidate อัตโนมัติ**
   ทุก 60 วินาที (ไม่ต้อง restart server) โดยใช้ `tokio::spawn` รัน background task ที่ re-render หน้านั้นซ้ำ
   ทุก 60 วินาทีแล้วเขียนค่าใหม่ทับ `Arc<RwLock<String>>` (หรือ `ArcSwap` ถ้าต้องการ lock-free) ที่ route
   handler อ่านอยู่ — นี่คือรูปแบบที่ใกล้เคียงกับ **Incremental Static Regeneration** ของ framework อื่น ๆ
   ในโลก JavaScript — *Hint*: ทบทวน Part 39 (Shared State) เรื่อง `Arc<RwLock<T>>` ข้าม task และต้องระมัดระวัง
   ไม่ให้ background task hold write lock ค้างนานเกินไปจนบล็อก request ที่กำลังอ่านอยู่

## สรุป

บทนี้เจาะลึกกลไกที่ Part 89 เพียงแค่แนะนำภาพรวมไว้ ทำให้เห็นสามแนวทาง CSR/SSR/SSG พร้อม trade-off ที่ชัดเจน
บนสี่แกน (TTFB, SEO, ภาระ server, ความซับซ้อนของ build) และพิสูจน์กลไก hydration ในระดับที่ลึกกว่าที่บทความ
ทั่วไปอธิบายไว้มาก — โดยเฉพาะข้อเท็จจริงที่น่าประหลาดใจว่า **hydration mismatch บางแบบ (สลับ tag ของ element)
ผ่านไปแบบเงียบสนิทโดยไม่มี error เลย** ในขณะที่บางแบบ (สลับชนิดของ node) จะ panic ทันทีพร้อม console.error
ที่ชัดเจน — ทั้งสองพฤติกรรมพิสูจน์ด้วยการรันจริงใน headless Chromium ไม่ใช่การคาดเดา

เราเปรียบเทียบความพร้อมด้าน SSR ของ Leptos, Yew, และ Dioxus อย่างตรงไปตรงมาจากซอร์สโค้ดจริง สร้าง SSR server
จริงที่ query PostgreSQL แล้วพิสูจน์ด้วย `curl` ว่าเนื้อหาอยู่ในไฟล์ HTML ตั้งแต่ response แรก ขยายไปสู่
streaming SSR ที่วัดผลจริงด้วยตัวเลข TTFB, ตั้ง SEO meta tag ต่อ route ที่พิสูจน์ได้จาก crawler ที่ไม่รัน JS,
และแยกความแตกต่างระหว่าง TTFB กับ Time-to-Interactive ให้แม่นยำ (SSR ช่วยอย่างแรกแต่ไม่ได้ทำให้อย่างหลัง
หายไป) ปิดท้ายด้วยแนวทาง hybrid ที่ผสม SSG กับ SSR ในโปรเซสเดียวกัน ซึ่งพิสูจน์ด้วยการทดลองจริงว่าสอง route
ในระบบเดียวให้พฤติกรรมต่างกันได้ตามที่ออกแบบ

**SSR foundation ที่บทนี้สร้างไว้ — Axum Router ที่ผูกกับ Leptos renderer, การ query database ก่อน render,
streaming SSR, SEO meta tags, และรูปแบบ hybrid SSG/SSR — คือรากฐานตรงที่ Full-Stack Project สามส่วน (Part
92-94) จะนำไปต่อยอดเป็นแอปพลิเคชันที่สมบูรณ์**: Part 92 จะออกแบบและสร้าง backend เต็มรูปแบบ (schema
ฐานข้อมูล, authentication, business logic) Part 93 จะสร้าง frontend ที่ผูกกับ backend นั้นด้วยเทคนิค SSR/
server function ที่บทนี้ปูพื้นไว้ และ Part 94 จะเติมส่วนที่เหลือให้เป็นระบบสมบูรณ์พร้อม deploy — ซึ่ง Part 96
(Docker และการ Deploy จริง) จะสอนว่าต้องเอา Axum/Leptos SSR process ที่เราสร้างและทดสอบไว้ตลอดบทนี้ไป
package และรันบน production อย่างไรให้ปลอดภัยและ scale ได้จริง

ก่อนไปต่อ ควรจำหลักการสามข้อที่บทนี้พิสูจน์ด้วยการรันจริงทั้งหมด ไม่ใช่แค่คำอธิบายเชิงทฤษฎี: (1) hydration
mismatch ไม่ใช่ "error เสมอ" — บางแบบเงียบสนิท บางแบบ panic ดัง ๆ ขึ้นอยู่กับว่าต่างกันที่ "ชนิดของ node" หรือ
"identity ของ tag ภายในชนิดเดียวกัน" (2) SSR ปรับปรุงสิ่งที่ผู้ใช้*เห็น*ได้เร็วขึ้น แต่ไม่ได้ทำให้สิ่งที่ผู้ใช้
*กดได้*เร็วขึ้นเลยแม้แต่นิดเดียว เพราะต้นทุนของ WASM bundle ยังเท่าเดิมไม่ว่าจะ SSR หรือ CSR (3) SSG/SSR/CSR
เป็นตัวเลือกที่ตัดสินใจได้ **ระดับรายหน้า** ไม่ใช่ระดับทั้งแอป — คำถามที่ควรถามเสมอคือ "ใครจะเห็นหน้านี้ ทำไม
และเปลี่ยนบ่อยแค่ไหน" ไม่ใช่ "framework นี้ควรใช้ SSR หรือ CSR"

จำสามข้อนี้ไว้ให้แม่น เพราะ Part 92-94 จะอ้างอิงกลับมาที่หลักการเหล่านี้ตลอดทั้งสามบท

---

**Part ก่อนหน้า:** [Dioxus Framework เบื้องต้น](part-090-dioxus-framework.md) | **Part ถัดไป:** [Full-Stack Project (1/3): ออกแบบและสร้าง Backend](part-092-fullstack-backend.md)
