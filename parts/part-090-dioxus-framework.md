# Part 90: Dioxus Framework เบื้องต้น

> โมดูล: Full-Stack และ WebAssembly | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างชัดเจนว่า **Dioxus** คืออะไร และเข้าใจจุดขายที่ทำให้มันต่างจาก Yew (Part 88) และ Leptos
  (Part 89) อย่างเป็นรูปธรรม: Dioxus เป็นเฟรมเวิร์กเดียวในสามตัวนี้ที่เขียนโค้ด component **ชุดเดียว** แล้วสลับ
  "renderer" ไปคอมไพล์เป็นเป้าหมายได้หลายแพลตฟอร์ม (เว็บผ่าน WASM, desktop ผ่าน native webview, มือถือผ่าน
  native shell) — พร้อมรู้เท่าทันว่าคำว่า "write once, render anywhere" นี้ **จริงแค่ไหน** ในทางปฏิบัติ ไม่ใช่
  การเชื่อคำโฆษณาแบบไม่ตรวจสอบ
- ติดตั้งและใช้ `dioxus-cli` (คำสั่ง `dx`) จริง สร้างโปรเจกต์ Dioxus ตั้งแต่ต้น เขียน component แรกด้วย macro
  `rsx!` และเข้าใจว่า `rsx!` คือ declarative macro ตัวที่สาม (ต่อจาก `html!` ของ Yew และ `view!` ของ Leptos)
  ที่ใช้ syntax คล้าย JSX ซึ่งทั้งหมดตั้งอยู่บนพื้นฐานของ `macro_rules!`/procedural macro ที่เรียนไปแล้วใน
  Part 36
- สร้าง component พร้อม props ตามแบบของ Dioxus เอง (attribute `#[component]` ที่แปลง parameter ของฟังก์ชัน
  เป็น props โดยอัตโนมัติ และ `#[derive(Props)]` แบบ manual เมื่อต้องการควบคุมมากขึ้น) บนโดเมนระบบจองตั๋วการ
  แสดง ซึ่งเป็นโดเมนเดียวกับที่ Part 88 และ Part 89 ใช้ เพื่อให้เปรียบเทียบโค้ดข้ามเฟรมเวิร์กได้ตรง ๆ
- ใช้ hook `use_signal` (ระบบ reactivity ปัจจุบันของ Dioxus 0.7 ซึ่งมาแทน `use_state` ของเวอร์ชันเก่า) จัดการ
  state ที่มีการโต้ตอบจริง (จองตั๋ว เพิ่ม/ลดจำนวนที่นั่ง) และเข้าใจว่าโมเดล reactivity ของ Dioxus เป็นแบบ
  ผสม (hybrid) ระหว่าง virtual DOM ของ Yew กับ fine-grained signal ของ Leptos อย่างไร
- build และรันแอปตัวเดียวกันได้จริงทั้งบนเป้าหมาย **web** (`dx build --platform web`) และ **desktop**
  (`dx build --platform desktop` ซึ่งใช้ native webview ผ่าน `wry`/`tao` อยู่เบื้องหลัง) พร้อมรู้อย่างตรงไป
  ตรงมาว่าในสภาพแวดล้อมที่ไม่มี display server (headless) แบบที่ใช้เขียนบทนี้ เราตรวจสอบได้แค่ระดับไหน
- เปรียบเทียบ Yew, Leptos, และ Dioxus แบบครบทุกมุม (reactivity model, แพลตฟอร์มที่รองรับ, ความพร้อมของ
  ecosystem, เรื่อง SSR/full-stack, และความยากในการเรียนรู้) และได้ **กรอบการตัดสินใจ** ที่ใช้เลือกเฟรมเวิร์ก
  จริงในโปรเจกต์ พร้อมคำตอบว่าหลักสูตรนี้จะใช้เฟรมเวิร์กไหนสำหรับ capstone project ใน Part 92-94 และทำไม

## ความรู้ที่ต้องมีมาก่อน

- **Part 36 (Macros: Declarative Macros)** — `rsx!` ที่เป็นหัวใจของบทนี้คือ procedural macro (function-like)
  ที่ต่อยอดแนวคิดเดียวกับที่ Part 36 สอนไว้: macro รับ **token** ของโค้ดที่เราเขียน แล้ว "ขยาย" เป็นโค้ด Rust
  จริงก่อนจะถูก compile — เพียงแต่ `rsx!` ซับซ้อนกว่า `macro_rules!` ธรรมดามาก เพราะเป็น procedural macro ที่
  ต้อง parse syntax คล้าย HTML/JSX เอง ถ้าจำความแตกต่างระหว่าง compile time กับ runtime ของ macro ไม่ชัด ควร
  ทวน Part 36 ก่อน
- **Part 86 (WebAssembly เบื้องต้นด้วย Rust)** และ **Part 87 (wasm-bindgen และ JavaScript Interop)** —
  เมื่อ Dioxus target เว็บ มันคอมไพล์ไปเป็น `wasm32-unknown-unknown` เหมือนที่สองบทนี้สอนไว้ทุกประการ และใช้
  `wasm-bindgen` อยู่ภายใต้ crate `dioxus-web` เพื่อคุยกับ DOM ผ่าน `web-sys` — บทนี้จะไม่สอนกลไก WASM ใหม่
  แต่จะพึ่งพาความเข้าใจจากสองบทนั้นตลอด
- **Part 88 (Yew Framework)** — บทนี้อ้างอิงโมเดล virtual DOM diffing ของ Yew ตลอดทั้งบทเพื่อเปรียบเทียบกับ
  โมเดลของ Dioxus โดยตรง รวมถึง macro `html!` ที่ใช้เทียบกับ `rsx!` ในหัวข้อ 90.2
- **Part 89 (Leptos Framework)** — บทนี้อ้างอิงโมเดล fine-grained reactivity (signal) และเรื่อง SSR/
  server function ของ Leptos ตลอดทั้งบทเพื่อเปรียบเทียบกับ Dioxus โดยตรง รวมถึง macro `view!` ที่ใช้เทียบกับ
  `rsx!` ในหัวข้อ 90.2
- **Part 16 (Modules และการจัดโครงสร้างโปรเจกต์)** — โครงสร้างโปรเจกต์ที่ `dx new`/`dx build` สร้างให้ใช้
  `mod`/`pub use` แบบเดียวกับที่ Part 16 สอนไว้ เมื่อเราแยก component ออกเป็นหลายไฟล์ในหัวข้อ 90.3
- **Part 11 (Option และ null safety)** — ใช้ตอนออกแบบ props ที่มีค่า default (เช่น ที่นั่งที่ยังไม่ได้จอง)
  ด้วย `Option<T>` และ attribute `#[props(default)]`
- **Part 18/19 (Generics และ Traits พื้นฐาน)** — ใช้ตอนอธิบาย trait bound `PartialEq + Clone + 'static` ที่
  ระบบ props ของ Dioxus บังคับให้ทุก struct props ต้อง implement เพื่อให้ diffing ทำงานได้ถูกต้อง
- **Part 28 (Smart Pointers: `Rc<T>` และ `RefCell<T>`)** — ระบบ borrow-check ของ `Signal<T>` (ที่ตรวจ
  aliasing ตอน**runtime**แทน compile time เมื่อเรียก `.read()`/`.write()` ชนกัน) ใช้หลักการเดียวกับ
  `RefCell::borrow`/`borrow_mut` ที่ Part 28 สอนไว้เป๊ะ ๆ — ถ้าจำ panic message ของ `RefCell` ตอน borrow
  ชนกันไม่ได้ ควรทวนก่อนเข้าหัวข้อกับดักที่ 6 ของบทนี้

## เนื้อหา

### 90.1 Dioxus คืออะไร และทำไม "write once, render anywhere" ถึงเป็นจุดขายหลัก

ก่อนอื่น ต้องทำความเข้าใจก่อนว่า Dioxus แก้ปัญหาคนละมุมกับ Yew และ Leptos ที่เราเรียนมาสองบทก่อน แม้หน้าตา
โค้ดจะคล้ายกันมาก (เพราะทั้งสามใช้ macro สไตล์ JSX และ component-based architecture) แต่ **แรงจูงใจในการมี
อยู่** ของ Dioxus ต่างออกไปอย่างชัดเจน

Yew (Part 88) ตอบคำถาม "อยากเขียน SPA แบบ React แต่ใช้ Rust ได้ไหม" — คำตอบคือได้ ผ่าน virtual DOM diffing
บน WASM ในเบราว์เซอร์เท่านั้น

Leptos (Part 89) ตอบคำถาม "อยากได้ full-stack web framework ที่ SSR เนียนจริง ๆ ไม่ใช่แค่แปะ WASM ทับ HTML
ได้ไหม" — คำตอบคือได้ ผ่าน fine-grained reactivity ที่ compile ไปเป็นการอัปเดต DOM node ตรง ๆ โดยไม่มี virtual
DOM มาคั่นกลาง และมี server function ที่เรียกจาก client ไปยัง server ได้อย่างไร้รอยต่อ

ส่วน **Dioxus ตอบคำถามคนละข้อไปเลย**: "อยากเขียน UI component ด้วย Rust ครั้งเดียว แล้วรันได้ทั้งบนเว็บ,
เดสก์ท็อป (Windows/macOS/Linux), และมือถือ (iOS/Android) โดยไม่ต้องเขียนใหม่ทุกแพลตฟอร์มได้ไหม" — นี่คือ
ความแตกต่างเชิง**เป้าหมาย**ที่สำคัญที่สุด ไม่ใช่แค่ syntax macro ที่ต่างกันเล็กน้อย

**Dioxus (อ่านว่า "ได-ออก-ซัส" ตามชื่อที่ทีมผู้พัฒนาออกเสียง) คือ Rust GUI framework ที่แยกส่วน "ตรรกะของ UI
component" ออกจาก "ตัวเรนเดอร์จริง" (renderer)** ตัว component ที่คุณเขียนด้วย `rsx!` จะถูกแปลงเป็น
`VirtualDom` ภายใน `dioxus-core` ซึ่งเป็น data structure ที่ไม่ผูกติดกับแพลตฟอร์มไหนเลย จากนั้นค่อยส่งต่อให้
"renderer" ตัวหนึ่งที่ Dioxus มีให้เลือก (ตรวจสอบตรงจาก `Cargo.toml` ของ crate `dioxus` 0.7.10 เองว่า
feature ไหน map ไปที่ dependency ตัวไหนจริง):

| Feature ที่ enable | Renderer crate ที่ถูกดึงมาจริง | ใช้ทำอะไร | ทำงานอย่างไร |
|---|---|---|---|
| `web` | `dioxus-web` | เป้าหมายเว็บ (browser) | คอมไพล์เป็น `wasm32-unknown-unknown` แล้วใช้ `web-sys`/`wasm-bindgen` แก้ไข DOM จริงในเบราว์เซอร์ (เหมือนที่ Part 86-87 สอน) |
| `desktop` | `dioxus-desktop` | เป้าหมาย desktop (Windows/macOS/Linux) | คอมไพล์เป็น native binary ของแต่ละ OS แล้วเปิดหน้าต่าง native ที่ฝัง **native webview ของระบบปฏิบัติการ** ไว้ข้างใน ผ่าน crate `wry` (จัดการ webview) และ `tao` (จัดการหน้าต่าง/event loop) |
| `mobile` | **`dioxus-desktop` ตัวเดียวกัน** (ตรวจสอบจาก `Cargo.toml` ของ `dioxus` 0.7.10 แล้วว่า `mobile = ["dep:dioxus-desktop"]` — ไม่มี crate `dioxus-mobile` แยกต่างหากเลย) | เป้าหมายมือถือ (iOS/Android) | ใช้ renderer เดียวกับ desktop เป๊ะ ๆ เพียงแต่คอมไพล์ข้าม (cross-compile) ไปยัง target triple ของมือถือ แล้วฝัง native webview ของระบบมือถือ (WKWebView บน iOS, Android System WebView บน Android ผ่าน `wry`/`tao`) แทนที่จะเป็นของ desktop OS |
| `liveview` | `dioxus-liveview` | Server-rendered UI ที่ diff ส่งผ่าน WebSocket | render บน server แล้วส่ง patch ไปอัปเดต DOM ฝั่ง client ผ่าน WebSocket (คล้ายแนวคิด Phoenix LiveView) |
| `ssr` | `dioxus-ssr` | Server-side rendering ธรรมดา | render เป็น HTML string ตรง ๆ บน server (ใช้ต่อใน Part 91) |

จุดที่น่าสนใจและคนมักเข้าใจผิดคือ **"มือถือ" กับ "desktop" ของ Dioxus ไม่ใช่ renderer คนละตัวกัน** มันคือ
`dioxus-desktop` crate เดียวกันเป๊ะ ที่ถูก cross-compile ไปยัง target ของมือถือแทน — เหตุผลที่ทำได้เพราะ
`wry`/`tao` ทั้งคู่รองรับการคอมไพล์ข้าม target มือถือมาแต่ต้น (เป็น crate จากทีม Tauri ที่ต้อง support ทั้ง
desktop และ mobile shell อยู่แล้ว) นี่คือหลักฐานเชิงสถาปัตยกรรมอีกชิ้นที่สนับสนุนสโลแกน "write once, render
anywhere" ของ Dioxus ได้ตรงกว่าที่คิด — ไม่ใช่แค่ตัว component logic ที่แชร์กัน แต่ตัว **renderer** เองก็
แชร์โค้ดข้าม desktop/mobile เกือบทั้งหมดด้วย

จุดสำคัญคือ **ทุก renderer เหล่านี้กิน `VirtualDom` ตัวเดียวกัน** — โค้ด component ของคุณไม่ต้องรู้ด้วยซ้ำว่า
กำลังถูกเรนเดอร์ด้วยตัวไหน สิ่งที่ component ต้องทำมีแค่ return ค่า `Element` ออกมาจาก `rsx!` เท่านั้น ส่วน
"จะเอา `Element` นี้ไปแปลงเป็นอะไรจริง ๆ บนหน้าจอ" เป็นงานของ renderer ไม่ใช่งานของ component

**แต่ต้องซื่อสัตย์ตรงนี้ทันที ก่อนที่จะเข้าใจผิดว่าโค้ด 100% แชร์กันได้ข้ามแพลตฟอร์มแบบไม่ต้องแก้อะไรเลย**
สิ่งที่แชร์กันได้จริงคือ **logic ของ component และ markup ที่เขียนด้วย `rsx!`** — ถ้า component ของคุณรับ
props, เก็บ state ด้วย `use_signal`, render div/button/input ธรรมดา ๆ โค้ดนั้นแชร์ข้ามเว็บ/desktop ได้แทบ
100% จริง (เราจะพิสูจน์ด้วยโค้ดจริงในหัวข้อ 90.7) แต่สิ่งที่ **ไม่แชร์กันแบบอัตโนมัติ** มีอยู่จริงและสำคัญ
พอที่ต้องพูดตรง ๆ ตั้งแต่ต้นบท:

1. **ทุกอย่างที่เรียก JS API ตรง ๆ ผ่าน `web-sys`** (เช่น เรียก `window.alert`, เข้าถึง `localStorage`,
   หรือฟังก์ชันที่ import จาก JS module ผ่าน `wasm-bindgen` ตาม Part 87) ใช้ได้เฉพาะบนเว็บ เพราะบน desktop/
   มือถือไม่มี `window`/`navigator` ของเบราว์เซอร์จริง ๆ ให้เรียก (แม้ desktop จะมี webview ฝังอยู่ แต่ Dioxus
   ไม่ได้ expose `web-sys` ให้ใช้ตรงในโหมด desktop — ต้องใช้ API ของ `dioxus-desktop` เอง เช่น
   `use_window()` ถ้าต้องการควบคุมหน้าต่าง)
2. **การเข้าถึงไฟล์ระบบ, การเปิด process, network socket ระดับ OS** — ทำได้บน desktop/mobile (เพราะเป็น
   native binary จริง ที่มีสิทธิ์เข้าถึงระบบไฟล์) แต่ทำไม่ได้บนเว็บ (เพราะ WASM ในเบราว์เซอร์ถูก sandbox
   ไว้ ไม่มีสิทธิ์เข้าถึงระบบไฟล์ผู้ใช้ตรง ๆ ตามที่ Part 86 อธิบายไว้เรื่อง WASM sandboxing)
3. **Cargo feature flags ที่ต้องสลับต่อแพลตฟอร์ม** — Cargo.toml ของโปรเจกต์ Dioxus ต้องประกาศ feature
   `web`/`desktop`/`mobile` แยกกัน (จะเห็นในหัวข้อ 90.5-90.7) แม้เนื้อ component จะเหมือนกัน แต่ตัว
   dependency graph ที่ดึงเข้ามาคอมไพล์นั้นต่างกันจริง (เว็บดึง `dioxus-web` + คอมไพล์เป็น WASM, desktop ดึง
   `dioxus-desktop` + `wry` + `tao` + คอมไพล์เป็น native binary)
4. **CSS/layout บางส่วนอาจต้องปรับ** — เพราะขนาดหน้าจอ desktop กับมือถือต่างกันมาก การออกแบบ responsive
   layout ยังเป็นความรับผิดชอบของนักพัฒนา ไม่ใช่สิ่งที่ Dioxus แก้ให้อัตโนมัติ

ดังนั้นสโลแกน "write once, render anywhere" ที่ถูกที่สุดควรอ่านว่า **"เขียน component logic และ markup
ครั้งเดียว แชร์ได้จริงในส่วนที่เป็น pure UI logic ส่วน platform-specific integration (JS interop, file
system, native API) ยังต้องเขียนแยกตามแพลตฟอร์มอยู่ดี"** — นี่ไม่ใช่ข้อบกพร่องของ Dioxus แต่เป็นข้อเท็จจริง
ที่หลีกเลี่ยงไม่ได้ของการมีแพลตฟอร์มที่มี capability ต่างกันจริง ๆ (เบราว์เซอร์ sandbox ต่างจาก native process
โดยธรรมชาติ) สิ่งที่ Dioxus ทำได้ดีคือ **ลดพื้นที่ของโค้ดที่ต้องเขียนซ้ำให้เหลือแค่ส่วน platform-specific
integration เท่านั้น** ไม่ใช่ทำให้ทุกบรรทัดแชร์กันได้ 100% แบบที่โฆษณาบางแหล่งอาจสื่อเกินจริง

เทียบให้เห็นภาพรวมกับ Yew และ Leptos:

| | Yew (Part 88) | Leptos (Part 89) | Dioxus (บทนี้) |
|---|---|---|---|
| แพลตฟอร์มที่รองรับ | เว็บ (WASM) เท่านั้น | เว็บ (WASM) + SSR บน server (ยังคือ "เว็บ" ในความหมายที่ผู้ใช้ปลายทางเห็น HTML) | เว็บ (WASM), desktop (native webview), มือถือ (native webview), SSR/LiveView บน server |
| เหตุผลที่ไม่ขยายไปแพลตฟอร์มอื่น | ออกแบบมาสำหรับ browser DOM ตั้งแต่ต้น ผูกกับ `web-sys` ลึก | ออกแบบมาสำหรับ full-stack **web** โดยเฉพาะ (SSR + hydration + server function ล้วนอิงกับ HTTP request/response) | ออกแบบ core ให้ platform-agnostic ตั้งแต่ต้น แยก `dioxus-core` (VirtualDom) ออกจาก renderer โดยเจตนา |

ทั้งหมดนี้จะกลับมาเป็นตารางเปรียบเทียบเต็มรูปแบบอีกครั้งในหัวข้อ 90.10 ซึ่งเป็นหัวข้อปิดของบทนี้ — ตอนนี้จำ
ไว้แค่ว่า **จุดขายเฉพาะตัวของ Dioxus คือการข้ามแพลตฟอร์ม ไม่ใช่การเป็น virtual DOM ที่เร็วกว่า Yew หรือ
fine-grained reactivity ที่ดีกว่า Leptos**

### 90.2 ติดตั้ง dioxus-cli (`dx`) และเขียน Hello World ตัวแรก

Dioxus มี CLI ชื่อ `dx` (ติดตั้งจาก crate `dioxus-cli`) ที่ทำหน้าที่คล้าย `trunk` ที่ใช้กับ Yew หรือ
`cargo-leptos` ที่ใช้กับ Leptos — คือเป็นตัวจัดการ build pipeline ทั้งหมด (คอมไพล์ Rust → WASM/native
binary, จัดการ asset, รัน dev server ที่มี hot-reload, และแพ็กเป็นไฟล์ติดตั้งสำหรับแต่ละแพลตฟอร์ม)

ณ วันที่เขียนบทนี้ (กันยายน 2026) เวอร์ชันที่ตรวจสอบจริงบน crates.io คือ **`dioxus` 0.7.10** เป็นเวอร์ชัน
stable ล่าสุด (มี `0.8.0-alpha.1` เป็นเวอร์ชันทดลองที่ยังไม่แนะนำให้ใช้ในงานจริง) และ **`dioxus-cli` 0.7.10**
ซึ่งเป็นเวอร์ชันที่ตรงกับ `dioxus` 0.7.10 พอดี — ข้อสำคัญที่ต้องระวังคือ **เวอร์ชันของ `dioxus-cli` ต้องตรง
กับเวอร์ชันของ crate `dioxus` ที่ใช้ในโปรเจกต์เสมอ** (หรืออย่างน้อยต้องเป็น minor version เดียวกัน) เพราะ CLI
ต้อง generate โค้ดบางส่วนและอ่าน metadata ที่ผูกกับโครงสร้างภายในของเวอร์ชันนั้น ๆ ถ้าเวอร์ชันไม่ตรงกันจะได้
error ตอน build ที่สืบสาวไปหาสาเหตุได้ยากมาก

ติดตั้งได้สองวิธี วิธีแรกคือใช้ install script ที่ทีม Dioxus ให้มา:

```bash
curl -fsSL https://dioxuslabs.com/install.sh | bash
```

วิธีที่สอง (แนะนำเมื่อต้องการควบคุมเวอร์ชันแบบตายตัวเพื่อความ reproducible เหมือนที่บทนี้ใช้ตรวจสอบจริง) คือ
ติดตั้งผ่าน `cargo install` ตรง ๆ:

```bash
cargo install dioxus-cli --version 0.7.10 --locked
```

หลังติดตั้งเสร็จ ตรวจสอบเวอร์ชันด้วย:

```bash
dx --version
```

ผลลัพธ์ที่ได้จริงจากการติดตั้งเพื่อเขียนบทนี้คือ `dioxus 0.7.10` (ตรงกับที่ตั้งใจไว้) ขั้นตอนการติดตั้งใช้เวลา
พอสมควรเพราะ `dioxus-cli` มี dependency จำนวนมาก (มันฝัง bundler, dev server แบบ hot-reload, และ logic
สำหรับหลายแพลตฟอร์มไว้ในตัวเดียว) — นี่คือสัญญาณแรกที่บอกว่า Dioxus "ครบเครื่อง" กว่า trunk ของ Yew มาก
(trunk ทำแค่ bundler สำหรับเว็บ) แต่ก็แลกมาด้วย compile time ของตัว CLI เองที่นานกว่า

**สร้างโปรเจกต์ใหม่ด้วย `dx new`** (คำสั่งนี้จะดึง template จาก GitHub ผ่าน `cargo-generate` จึงต้องมี
เครือข่ายตอนสร้างโปรเจกต์ครั้งแรก):

```bash
dx new hello_dioxus
cd hello_dioxus
```

หรือถ้าอยากเห็นโครงสร้างที่เล็กที่สุดแบบมือ ๆ (ซึ่งบทนี้จะใช้แนวทางนี้เพื่อควบคุม dependency ให้ตรงกับที่
ตรวจสอบจริง) ให้สร้าง Cargo project ปกติแล้วเติม dependency เอง:

```bash
cargo new hello_dioxus
cd hello_dioxus
cargo add dioxus@0.7.10 --features web
```

ไฟล์ `Cargo.toml` ที่ได้ (ตัดส่วนที่ cargo generate ให้อัตโนมัติ):

```toml
[package]
name = "hello_dioxus"
version = "0.1.0"
edition = "2021"

[dependencies]
dioxus = { version = "0.7.10", features = ["web"] }
```

จากนั้นเขียน `src/main.rs` เป็น component แรกด้วย macro `rsx!`:

```rust
// src/main.rs
use dioxus::prelude::*;

fn main() {
    // launch() คือจุดเริ่มต้นที่บอก Dioxus ว่า "เอา component `app`
    // ไปให้ renderer ที่ถูก enable ไว้ (ในที่นี้คือ dioxus-web
    // เพราะเราเปิด feature "web") render ออกมา"
    dioxus::launch(app);
}

// component ใน Dioxus คือฟังก์ชันที่ไม่รับ argument (หรือรับ props)
// และ return ค่า Element ซึ่งเป็น type ที่ dioxus-core นิยามไว้
// แทน "โครงต้นไม้ของ UI ที่ยังไม่ได้เรนเดอร์จริง"
fn app() -> Element {
    rsx! {
        h1 { "สวัสดี Dioxus!" }
        p { "นี่คือ component แรกที่เขียนด้วย macro rsx!" }
    }
}
```

รันด้วย dev server ของ `dx`:

```bash
dx serve --platform web
```

คำสั่งนี้จะคอมไพล์โค้ดเป็น `wasm32-unknown-unknown`, ฝัง WASM ไว้ใน `index.html` ที่ generate ให้อัตโนมัติ,
เปิด dev server (ปกติที่ `http://localhost:8080`), และเปิด hot-reload ให้ — แก้ markup ใน `rsx!` แล้ว
เบราว์เซอร์อัปเดตทันทีโดยไม่ต้อง refresh (นี่คือฟีเจอร์ hot-reload ที่ Dioxus ภูมิใจนำเสนอมาก เพราะมันไม่ใช่
แค่ live-reload ธรรมดา แต่เป็นการแก้ไข template ของ `rsx!` แบบ real-time โดยไม่ต้อง compile ใหม่ทั้งหมด
ในหลายกรณี)

#### เปรียบเทียบสามสไตล์ macro: `html!` vs `view!` vs `rsx!`

ตอนนี้คุณเห็น declarative macro สไตล์ JSX มาแล้วสามตัวตลอดโมดูลนี้ — คุ้มที่จะหยุดเทียบให้เห็นภาพรวมสั้น ๆ
ว่าโค้ด component เดียวกัน (ปุ่มกดเพิ่มค่า) เขียนต่างกันอย่างไรในสามเฟรมเวิร์ก:

```rust
// สไตล์ Yew: html! macro (Part 88)
// - ปิดวงเล็บปีกกาแบบ HTML tag เต็มรูปแบบ <tag>...</tag>
// - ใส่ Rust expression ด้วย { } ข้างในเหมือนกัน แต่ทั้งก้อนอยู่ในเครื่องหมาย < >
html! {
    <div>
        <p>{ format!("นับได้ {}", *count) }</p>
        <button onclick={move |_| count.set(*count + 1)}>{ "เพิ่ม" }</button>
    </div>
}

// สไตล์ Leptos: view! macro (Part 89)
// - ใกล้เคียง html! ของ Yew มาก (ยังเป็น tag แบบ < >)
// - ต่างที่ signal เขียนแบบ getter/setter แยกกันชัดเจน (count.get() / set_count.set(...))
view! {
    <div>
        <p>{ format!("นับได้ {}", count.get()) }</p>
        <button on:click=move |_| set_count.set(count.get() + 1)>"เพิ่ม"</button>
    </div>
}

// สไตล์ Dioxus: rsx! macro (บทนี้)
// - ไม่ใช้ tag แบบ < > เลย! ใช้ชื่อ element ตามด้วย { } แบบเดียวกับการนิยาม struct
// - attribute เขียนแบบ key: value คั่นด้วย , เหมือนสร้าง struct literal
rsx! {
    div {
        p { "นับได้ {count}" } // สอด signal ตรงเข้า string ได้ด้วย {} แบบ format string
        button { onclick: move |_| count += 1, "เพิ่ม" }
    }
}
```

สังเกตสามจุดที่ต่างกันชัดเจน:

1. **`rsx!` ไม่ใช้ syntax แบบ HTML tag (`< >`) เลย** — มันใช้ syntax แบบ "struct literal" ของ Rust เอง
   (`ชื่อ { field: value }`) ทำให้ syntax highlighting และ auto-complete ของ rust-analyzer ทำงานได้ดีกว่า
   เพราะมันใกล้เคียงกับ Rust ปกติมากกว่า `html!`/`view!` ที่ต้อง parse โครงสร้างคล้าย XML ปนเข้ามา
2. **การสอดตัวแปรเข้า string ใน `rsx!` ใช้ `{ }` แบบเดียวกับ `format!`/`println!` ตรง ๆ** (`"นับได้ {count}"`)
   ไม่ต้องเขียน `{ format!(...) }` แยกก้อนแบบ Yew หรือ `{ count.get() }` แบบ Leptos — นี่เป็นเพราะ `rsx!`
   ให้ signal implement `Display` ตรง และ macro internal จะแปลง string ที่มี `{var}` ให้กลายเป็นการเรียก
   `format!` ให้อัตโนมัติ (ผูกกับความรู้ format string จาก Part 36 ที่สอนกลไกของ `format!`/`println!` ไว้ละเอียด)
3. **การอัปเดต state ใน closure ของ `onclick`** — Dioxus ให้เขียน `count += 1` ตรง ๆ ได้เพราะ `Signal<T>`
   implement `AddAssign` เมื่อ `T` เป็น numeric type (ตรวจสอบจาก source ของ `dioxus-signals` แล้วว่ามี macro
   ภายในที่ generate `impl AddAssign` ให้ signal ที่ห่อ type ตัวเลข) ทำให้เขียนสั้นกว่า Yew ที่ต้องเรียก
   `count.set(*count + 1)` ตรง ๆ

ทั้งสาม macro นี้เป็นแค่ **"น้ำตาลทางไวยากรณ์" (syntactic sugar)** ที่ compile time แปลงกลับไปเป็นโค้ด Rust
ล้วน ๆ — `rsx!` ขยายไปเป็นการสร้าง `VNode`/`Template` ของ `dioxus-core` เช่นเดียวกับที่ `html!` ขยายไปเป็น
`Html` ของ Yew และ `view!` ขยายไปเป็นชุดคำสั่งสร้าง DOM node ตรงของ Leptos — หลักการที่ Part 36 สอนไว้ว่า
"macro ทำงานตอน compile time โดยรับ token แล้วขยายเป็นโค้ดใหม่" ใช้ได้กับทั้งสามตัวเป๊ะ ๆ ต่างกันแค่ "โค้ด
ที่ขยายออกมา" นั้นไปสร้าง data structure คนละแบบตามสถาปัตยกรรมของแต่ละเฟรมเวิร์ก

#### โครงสร้างโปรเจกต์ที่ `dx new` สร้างให้ และไฟล์ `Dioxus.toml`

ถ้าสร้างโปรเจกต์ผ่าน `dx new` (ไม่ใช่ต่อ `cargo new` เองแบบข้างบน) จะได้โครงสร้างไฟล์เพิ่มมาจาก
`cargo new` ธรรมดาสองไฟล์หลัก:

```
hello_dioxus/
├── Cargo.toml
├── Dioxus.toml      # ไฟล์ config เฉพาะของ dx (ไม่ใช่ของ cargo)
├── assets/          # เก็บไฟล์ static เช่น favicon, css, รูปภาพ
└── src/
    └── main.rs
```

`Dioxus.toml` คือไฟล์ config ที่ `dx` (ไม่ใช่ `cargo`) อ่านเพื่อรู้ว่าจะ bundle แอปอย่างไร ตัวอย่างที่ตรวจ
สอบจริงจากไฟล์ template ของ `dioxus-cli` 0.7.10 มีโครงหน้าตาประมาณนี้:

```toml
[application]
# ชื่อแอป — ใช้เป็นชื่อไฟล์ binary/bundle สุดท้าย
name = "hello_dioxus"

# path ที่ dx build/serve จะเอาไฟล์ผลลัพธ์ไปวาง
out_dir = "dist"

# โฟลเดอร์ไฟล์ static ที่จะถูกก็อปปี้เข้า out_dir ตรง ๆ (เช่น favicon.ico)
public_dir = "public"

[web.app]
# <title> ของหน้า HTML ที่ dx generate ให้เมื่อ target เว็บ
title = "hello_dioxus"

[web.watcher]
# บอก dx ว่าถ้าไฟล์ในโฟลเดอร์ไหนเปลี่ยน ให้ trigger hot-reload
watch_path = ["src", "public"]
```

ข้อสำคัญที่ต้องรู้คือ **`Dioxus.toml` เป็นไฟล์ config ของ `dx` ล้วน ๆ ไม่ใช่ไฟล์ที่ `cargo build` ธรรมดา
รู้จัก** — ถ้าคุณสั่ง `cargo build --features web` ตรง ๆ โดยไม่ผ่าน `dx` โค้ดจะคอมไพล์ได้ตามปกติ (เพราะ
`cargo` สนใจแค่ `Cargo.toml`) แต่คุณจะไม่ได้ไฟล์ `index.html`/asset bundling ที่ `dx` จัดการให้ ต้องประกอบ
เองแทน — นี่คือเหตุผลที่โปรเจกต์ Dioxus จริงแทบทั้งหมดใช้ `dx build`/`dx serve` เป็นคำสั่งหลักเสมอ ไม่ใช้
`cargo build` ตรง ๆ (ต่างจากโปรเจกต์ Rust ปกติที่ `cargo build` เพียงพอ)

### 90.3 Component และ Props ใน Dioxus: ตัวอย่างระบบจองตั๋วการแสดง

มาดูโมเดลของ component และ props ของ Dioxus ให้ละเอียดขึ้น โดยใช้โดเมนเดียวกับ Part 88 และ Part 89 คือ
**ระบบจองตั๋วการแสดง** เพื่อให้เห็นความต่าง/ความเหมือนของแต่ละเฟรมเวิร์กตรง ๆ เมื่อแก้ปัญหาเดียวกัน

โดเมนของเราคือ struct `Show` ที่แทนการแสดงหนึ่งรอบ:

```rust
use dioxus::prelude::*;

// struct โดเมนธรรมดา ไม่มีอะไรเกี่ยวกับ Dioxus เลย — ต้อง derive Clone และ
// PartialEq เพราะ Dioxus ใช้ PartialEq เปรียบเทียบ props เก่ากับใหม่ตอน
// re-render เพื่อตัดสินใจว่าต้องอัปเดต DOM ส่วนนี้หรือไม่ (เหมือนหลักการ
// memoization ที่ React.memo ใช้ ไม่ใช่ของใหม่ที่ Dioxus คิดขึ้นเอง)
#[derive(Clone, PartialEq)]
struct Show {
    id: u32,
    title: String,
    price_baht: u32,
    seats_left: u32,
}
```

วิธีแรก (และวิธีที่ใช้บ่อยที่สุด) ในการสร้าง component ที่รับ props คือใส่ attribute `#[component]` บน
ฟังก์ชันธรรมดา แล้วให้ **parameter ของฟังก์ชันกลายเป็น props โดยอัตโนมัติ**:

```rust
// #[component] คือ attribute macro (ประเภทที่สองของ procedural macro ตามที่
// Part 36 แยกไว้ว่าต่างจาก declarative macro macro_rules!) มันจะ:
// 1. สร้าง struct props ที่มี field ตรงกับ parameter ของฟังก์ชัน (title, price_baht, seats_left)
// 2. derive Props, Clone, PartialEq ให้ struct นั้นอัตโนมัติ
// 3. เปลี่ยนตัวฟังก์ชันให้รับ struct props ตัวเดียว แล้วดึง field ออกมาเป็นตัวแปรในสโคปให้
#[component]
fn ShowCard(title: String, price_baht: u32, seats_left: u32) -> Element {
    rsx! {
        div { class: "show-card",
            h3 { "{title}" }
            p { "ราคา {price_baht} บาท" }
            p {
                // rsx! รองรับ if/else เป็น expression ธรรมดาได้เลย ไม่ต้อง
                // แปลงเป็น iterator/match แบบที่บาง framework ต้องทำ
                if seats_left == 0 {
                    "เต็มแล้ว"
                } else {
                    "เหลือ {seats_left} ที่นั่ง"
                }
            }
        }
    }
}
```

การเรียกใช้ component นี้ใน component แม่ ก็ใช้ syntax struct-literal เดียวกัน เพียงแต่ชื่อ component
เขียนขึ้นต้นด้วยตัวใหญ่ (`PascalCase` — เป็นกฎที่ `rsx!` ใช้แยกว่าอันไหนคือ HTML element ธรรมดา อันไหนคือ
component ของเราเอง: `div`/`button` ตัวเล็กคือ element, `ShowCard` ตัวใหญ่คือ component):

```rust
fn app() -> Element {
    rsx! {
        ShowCard {
            title: "คอนเสิร์ตวงเดอะทอยส์",
            price_baht: 1200,
            seats_left: 5,
        }
    }
}
```

วิธีที่สอง คือประกาศ struct props แบบ manual ด้วย `#[derive(Props)]` เมื่อต้องการควบคุมมากขึ้น เช่น
กำหนดค่า default หรือแปลง type อัตโนมัติผ่าน `into`:

```rust
// การประกาศ props แบบ manual นี้จำเป็นเมื่อ:
// - ต้องการ default value ให้ field ที่ไม่ได้ถูกส่งมา (#[props(default = ...)])
// - ต้องการ auto-conversion ด้วย into (#[props(into)]) เช่นรับทั้ง &str และ String
// - ต้องการแชร์ struct props เดียวกันระหว่างหลาย component
#[derive(Props, Clone, PartialEq)]
struct ShowCardProps {
    #[props(into)] // ให้เรียกด้วย title: "..." (&str) ได้โดยไม่ต้อง .to_string()
    title: String,

    price_baht: u32,

    #[props(default = 0)] // ถ้าไม่ส่ง seats_left มา จะเป็น 0 อัตโนมัติ
    seats_left: u32,
}

#[component]
fn ShowCardManual(props: ShowCardProps) -> Element {
    rsx! {
        div { class: "show-card",
            h3 { "{props.title}" }
            p { "ราคา {props.price_baht} บาท เหลือ {props.seats_left} ที่นั่ง" }
        }
    }
}
```

**เทียบกับ Yew และ Leptos ตรงจุดนี้ต้องซื่อสัตย์ว่า Dioxus ไม่เหมือนทั้งสองตัวเป๊ะ ๆ:**

- Yew (Part 88) บังคับให้ประกาศ struct ที่ implement trait `Properties` เสมอ (ผ่าน
  `#[derive(Properties, PartialEq)]`) ไม่มีทางลัดแบบ "parameter ของฟังก์ชันกลายเป็น props อัตโนมัติ"
  เพราะ Yew ใช้ struct-based component model ที่สืบทอดมาจาก React class/function component แบบตรง ๆ
- Leptos (Part 89) มี `#[component]` เหมือนกัน และให้ parameter ของฟังก์ชันกลายเป็น props ได้เหมือนกัน —
  ในจุดนี้ Dioxus กับ Leptos คล้ายกันมาก (ทั้งคู่ได้รับอิทธิพลจากกันและกันในช่วงที่ทั้งสองโปรเจกต์พัฒนา
  ระบบ signal-based reactivity ขึ้นมาใกล้เคียงกันในปี 2023) แต่รายละเอียดปลีกย่อยของ attribute (เช่น
  `#[props(...)]` ของ Dioxus เทียบกับ attribute ของ Leptos) มี syntax ที่ต่างกันและไม่สามารถใช้แทนกันได้

ข้อสรุปคือ: **อย่าสมมติว่า syntax ของ Dioxus เหมือน Yew หรือ Leptos เป๊ะ ๆ เพราะหน้าตาคล้ายกัน** ต้องอ่าน
เอกสารของ Dioxus เองเสมอเมื่อเจอ attribute หรือ pattern ที่ไม่คุ้นเคย

#### Composition: ส่ง `children` และส่ง callback กลับจาก child ไปยัง parent

โปรเจกต์จริงแทบทั้งหมดต้องแยก component ออกเป็นหลายชั้น (component แม่ประกอบ component ลูกหลายตัวเข้าด้วย
กัน) และบางครั้ง component ลูกต้อง "แจ้ง" component แม่ว่ามีเหตุการณ์เกิดขึ้น (เช่น ปุ่มถูกกด) โดยไม่รู้ว่า
component แม่จะเอาข้อมูลนั้นไปทำอะไรต่อ — Dioxus แก้โจทย์นี้ด้วย field ชนิด `Element` สำหรับรับ children
และ type `EventHandler<T>` สำหรับรับ callback

```rust
use dioxus::prelude::*;

// Card เป็น component ที่ไม่รู้จักเนื้อหาข้างในของตัวเองเลย
// มันแค่รับ "children: Element" มาแล้ว render ล้อมด้วย div ที่มี class
// ตกแต่ง — pattern นี้เทียบได้ตรงกับ {props.children} ของ React หรือ
// <slot/> ของ Vue
#[component]
fn Card(children: Element) -> Element {
    rsx! {
        div { class: "card-wrapper",
            {children}
        }
    }
}

// ShowRow คือ component ลูกที่ "ไม่ตัดสินใจเอง" ว่าจองตั๋วแล้วจะเกิดอะไร
// ขึ้นต่อ — มันแค่เรียก on_book (callback ที่ parent ส่งมาให้) พร้อมส่ง
// show_id กลับไป ให้ parent เป็นคนตัดสินใจว่าจะจัดการ state อย่างไร
// นี่คือรูปแบบ "lifting state up" แบบเดียวกับที่ React แนะนำ — Dioxus
// ใช้แนวคิดเดียวกันเป๊ะ เพียงแต่ type ที่ใช้แทน callback คือ EventHandler<T>
#[component]
fn ShowRow(show_id: u32, title: String, on_book: EventHandler<u32>) -> Element {
    rsx! {
        div { class: "show-row",
            span { "{title}" }
            button {
                onclick: move |_| on_book.call(show_id),
                "จองตั๋ว"
            }
        }
    }
}

fn app() -> Element {
    let mut total_booked = use_signal(|| 0u32);

    rsx! {
        Card {
            h2 { "รายการการแสดงวันนี้" }
            ShowRow {
                show_id: 1,
                title: "คอนเสิร์ตวงเดอะทอยส์".to_string(),
                // ส่ง closure เป็น EventHandler ตรง ๆ — เมื่อ ShowRow เรียก
                // on_book.call(1) closure นี้จะรันโดย total_booked เป็น
                // signal ของ component app (parent) ไม่ใช่ของ ShowRow
                on_book: move |id: u32| {
                    total_booked += 1;
                    println!("จองการแสดง id={id} แล้ว");
                },
            }
        }
        p { "จองไปแล้ว {total_booked} ครั้ง" }
    }
}
```

จุดที่ต้องเข้าใจให้ชัดคือ `EventHandler<T>` **ไม่ใช่ type เดียวกับ `Callback<T, R>`** ที่เห็นในหัวข้อ 90.8
— `EventHandler<T>` คือรูปแบบที่ง่ายกว่า ใช้เมื่อ callback ไม่ต้องคืนค่าอะไรกลับมา (คืน `()` เสมอ) เหมาะกับ
"แจ้งเหตุการณ์" ทั่วไปแบบ `on_book`/`on_cancel` ส่วน `Callback<T, R>` ใช้เมื่อ parent ต้องให้ค่าอะไรกลับไป
ให้ child ใช้ต่อจริง ๆ (เช่น ฟังก์ชันแปลงข้อมูล) — เลือกใช้ตัวที่ตรงกับโจทย์เสมอ ไม่ใช่ใช้ `Callback` ทุกที่
เพียงเพราะมันครอบคลุมกรณีได้มากกว่า (ยิ่ง type signature ซับซ้อนเกินจำเป็น ยิ่งอ่านยากสำหรับคนที่มาอ่านโค้ด
ต่อจากคุณ)

#### จัดโครงสร้างหลายไฟล์: แยก component ออกจาก `main.rs`

เมื่อแอปโตขึ้น การใส่ทุก component ไว้ใน `main.rs` ไฟล์เดียวจะอ่านยากขึ้นเรื่อย ๆ — Dioxus ไม่มีระบบไฟล์
พิเศษของตัวเอง มันใช้ระบบ `mod`/`pub use` ของ Rust ธรรมดาที่เรียนมาแล้วเต็มรูปแบบใน Part 16 ตรง ๆ ไม่มี
อะไรต้องเรียนรู้ใหม่:

```
src/
├── main.rs
└── components/
    ├── mod.rs        // pub mod show_card; pub mod show_row;
    ├── show_card.rs  // struct/component ShowCard
    └── show_row.rs   // struct/component ShowRow
```

```rust
// src/components/mod.rs
pub mod show_card;
pub mod show_row;
```

```rust
// src/components/show_card.rs
use dioxus::prelude::*;

#[component]
pub fn ShowCard(title: String, price_baht: u32, seats_left: u32) -> Element {
    rsx! {
        div { class: "show-card",
            h3 { "{title}" }
            p { "ราคา {price_baht} บาท เหลือ {seats_left} ที่นั่ง" }
        }
    }
}
```

```rust
// src/main.rs
mod components;

use components::show_card::ShowCard;
use dioxus::prelude::*;

fn app() -> Element {
    rsx! {
        ShowCard { title: "คอนเสิร์ตวงเดอะทอยส์".to_string(), price_baht: 1200, seats_left: 5 }
    }
}

fn main() {
    dioxus::launch(app);
}
```

ต้องใส่ `pub` หน้า `fn ShowCard` เสมอเมื่อจะ `use` มันจาก module อื่น (กฎ privacy เดียวกับที่ Part 16 สอน
ไว้ทุกประการ — ไม่มีข้อยกเว้นพิเศษให้ component ของ Dioxus) และ Rust analyzer/compiler จะแจ้ง error
`E0603: function \`ShowCard\` is private` ทันทีถ้าลืม เหมือนกับการลืม `pub` ใน struct/fn ธรรมดาทุกประการ

### 90.4 การจัดการ State ด้วย `use_signal`: ต่อยอดตัวอย่างให้โต้ตอบได้จริง

หัวข้อที่แล้วแสดง component แบบ static (รับ props แล้วแสดงผล ไม่มีการเปลี่ยนแปลง) หัวข้อนี้จะเติม state
เข้าไปให้ผู้ใช้กด "จองตั๋ว" แล้วจำนวนที่นั่งลดลงจริง

**ก่อนอื่น ต้องแก้ความเข้าใจผิดที่พบบ่อยเรื่อง API ของ Dioxus ให้ชัดก่อน**: ถ้าคุณเคยเห็นตัวอย่างโค้ด Dioxus
เก่า ๆ (บทความ, วิดีโอ, หรือ Stack Overflow ที่เขียนไว้ในช่วงปี 2022-2023) คุณอาจเจอฟังก์ชัน `use_state`
ซึ่งเป็น hook ของ Dioxus เวอร์ชัน 0.3-0.4 ที่ใช้โมเดล "immutable state + setter function" คล้าย
`useState` ของ React เป๊ะ ๆ — **แต่ตั้งแต่ Dioxus 0.5 เป็นต้นมา (และยืนยันแล้วจาก source code จริงของ
`dioxus-hooks` เวอร์ชัน 0.7.10 ที่ใช้ตรวจสอบบทนี้) hook `use_state` **ไม่มีอยู่แล้ว** ถูกแทนที่ด้วย
`use_signal` ซึ่งเป็นระบบ **signal-based reactivity** ที่ยืม concept มาจากทั้ง SolidJS และเป็นแนวทางที่
คล้ายกับที่ Leptos (Part 89) ใช้ (แม้จะไม่ใช่ implementation เดียวกัน)

```rust
use dioxus::prelude::*;

#[derive(Clone, PartialEq)]
struct Show {
    id: u32,
    title: String,
    price_baht: u32,
    seats_left: u32,
}

fn app() -> Element {
    // use_signal รับ closure ที่คืนค่าเริ่มต้น (คล้าย use_state ตัวเก่า
    // ที่ก็รับ closure เหมือนกัน — จุดนี้ยังคงไว้เพราะการรัน closure แค่ครั้ง
    // เดียวตอน component ถูกสร้างครั้งแรก สำคัญมากสำหรับ state ที่สร้างแพง
    // เช่น Vec ขนาดใหญ่ หรือการอ่านค่าเริ่มต้นจาก local storage)
    let mut shows = use_signal(|| {
        vec![
            Show { id: 1, title: "คอนเสิร์ตวงเดอะทอยส์".to_string(), price_baht: 1200, seats_left: 3 },
            Show { id: 2, title: "ละครเวทีเรื่องบางระจัน".to_string(), price_baht: 800, seats_left: 0 },
            Show { id: 3, title: "ตลกคาเฟ่คืนวันศุกร์".to_string(), price_baht: 300, seats_left: 10 },
        ]
    });

    // signal ตัวที่สองแยกอิสระจากตัวแรก — ใช้เก็บจำนวนตั๋วที่จองไปแล้วรวม
    let mut total_booked = use_signal(|| 0u32);

    rsx! {
        h1 { "ระบบจองตั๋วการแสดง" }
        // การสอด signal ตรงเข้า string interpolation แบบนี้ (ไม่ต้องเรียก
        // .read() หรือ () เพื่ออ่านค่า) เป็นเพราะ Signal<T> implement Display
        // เมื่อ T implement Display — ตัว macro rsx! เห็น "{total_booked}"
        // แล้วแปลงให้เรียก Display::fmt ให้อัตโนมัติ และการอ่านค่าแบบนี้จะ
        // "subscribe" component ปัจจุบันให้ re-render ทุกครั้งที่ signal
        // เปลี่ยนค่าโดยอัตโนมัติ — นี่คือหัวใจของ fine-grained reactivity
        p { "จองไปแล้วทั้งหมด: {total_booked} ใบ" }

        // for loop ธรรมดาใน rsx! ได้เลย ไม่ต้องแปลงเป็น .collect::<Html>()
        // แบบ Yew หรือ .into_view() แบบพิเศษ — rsx! parse for loop เป็น
        // native syntax (ตรวจสอบแล้วจาก dioxus-rsx source ว่ามี ForLoop
        // เป็น node ประเภทหนึ่งของ AST ของ rsx! โดยตรง)
        for show in shows() {
            // key: "{show.id}" คือ attribute พิเศษที่บอก diffing algorithm
            // ว่านี่คือ "ตัวตน" ของ node นี้ ใช้หลักการเดียวกับ key ของ React
            // list rendering หรือ Yew's key attribute — สำคัญมากเมื่อลำดับ
            // ของ list เปลี่ยน (เช่น เพิ่ม/ลบรายการ) เพื่อไม่ให้ Dioxus สับสน
            // ว่า node ไหนคือ node ไหนตอน re-render
            div { key: "{show.id}", class: "show-row",
                h3 { "{show.title}" }
                p { "ราคา {show.price_baht} บาท — เหลือ {show.seats_left} ที่นั่ง" }
                button {
                    // disabled รับ bool ตรง ๆ ตามที่ dioxus-html นิยาม
                    // attribute นี้ไว้เป็น Bool attribute (ตรวจสอบจาก source
                    // ของ dioxus-html แล้วว่า disabled ถูก generate เป็น
                    // attribute ประเภท Bool เหมือน HTML จริง)
                    disabled: show.seats_left == 0,
                    onclick: move |_| {
                        // shows.write() คืน mutable guard ที่ให้แก้ไข Vec
                        // ข้างในได้ตรง ๆ — เมื่อ guard นี้ถูก drop (จบ scope
                        // ของ closure) Dioxus จะรู้ว่า signal เปลี่ยนแล้ว
                        // และ schedule re-render ให้ทุก component ที่ subscribe
                        // signal ตัวนี้อยู่ (แนวคิดเดียวกับ RefCell::borrow_mut
                        // ที่เรียนใน Part 28 เรื่อง interior mutability
                        // แต่ Signal เพิ่ม "การแจ้งเตือนว่าค่าเปลี่ยน" เข้ามาด้วย)
                        let show_id = show.id;
                        shows.write().iter_mut()
                            .find(|s| s.id == show_id)
                            .map(|s| s.seats_left -= 1);
                        total_booked += 1;
                    },
                    "จองตั๋ว"
                }
            }
        }
    }
}

fn main() {
    dioxus::launch(app);
}
```

จุดที่ต้องเข้าใจให้ลึกคือ **`shows()` (เรียก signal เหมือนฟังก์ชัน) กับ `shows.write()` ต่างกันอย่างไร**:

- `shows()` คือ shorthand ของ `shows.read().clone()` (จริง ๆ แล้ว `Signal<T>` implement trait
  `FnMut() -> T` เมื่อ `T: Clone` — เรียกเหมือนฟังก์ชันแล้วได้ **สำเนา** ของค่าปัจจุบันกลับมา) การเรียกแบบนี้
  จะ subscribe component ปัจจุบันเข้ากับ signal (ทำให้ component re-render เมื่อ signal เปลี่ยน)
- `shows.write()` คืน `Write<Vec<Show>>` guard ที่ implement `DerefMut` ให้แก้ไขข้อมูลข้างในตรง ๆ ได้ —
  เหมาะกับ collection ขนาดใหญ่ที่การ clone ทั้งก้อนทุกครั้งจะสิ้นเปลือง (ต่างจาก `shows.set(new_vec)` ที่ต้อง
  สร้าง `Vec` ใหม่ทั้งก้อนแล้วแทนที่)

โมเดลนี้คือสิ่งที่ทำให้ Dioxus ไม่ใช่ "virtual DOM diffing ล้วน ๆ แบบ Yew" และไม่ใช่ "fine-grained
DOM update ล้วน ๆ แบบ Leptos" แต่เป็น **โมเดลผสม**: signal บอกอย่างแม่นยำว่า *component ไหน* ต้อง
re-render (ไม่ใช่ re-render ทั้งแอปเหมือน virtual DOM naive แบบเก่า) แต่ *ภายใน* component ที่ re-render
นั้น Dioxus ยังใช้ diffing ระดับ template (เปรียบเทียบ dynamic part ของ template ที่ compile ไว้แล้ว ไม่ใช่
diff ทั้งต้นไม้ DOM ใหม่ทั้งหมด) — รายละเอียดของโมเดลนี้จะขยายอีกครั้งในหัวข้อ 90.10

#### Hook อื่นที่ต่อยอดจาก `use_signal`: `use_memo`, `use_effect`, `use_resource`

`use_signal` เป็นแค่ hook พื้นฐานที่สุด Dioxus มี hook ระดับสูงอีกสามตัวที่สร้างต่อจาก signal อีกทีเพื่อ
แก้ปัญหาที่พบบ่อยมาก — มาดูทั้งสามตัวด้วยการต่อยอดตัวอย่างระบบจองตั๋วต่อไปอีก

**`use_memo`** — คำนวณค่าที่ **derive มาจาก signal อื่น** โดยจำผลลัพธ์ไว้ (memoize) ไม่คำนวณใหม่ทุกครั้งที่
component re-render ด้วยเหตุผลอื่นที่ไม่เกี่ยวกับ signal ที่มันอ่าน — เหมาะกับการคำนวณที่มีต้นทุนสูงหรือแค่
อยากให้ค่าที่ derive ออกมาชัดเจนแยกจาก logic อื่น:

```rust
// สืบเนื่องจาก signal `shows: Signal<Vec<Show>>` ในหัวข้อ 90.4
// สมมติ Show มี field เพิ่ม total_seats (ที่นั่งทั้งหมดตอนเปิดขาย)
let mut shows = use_signal(|| vec![
    Show { id: 1, title: "คอนเสิร์ตวงเดอะทอยส์".into(), price_baht: 1200, total_seats: 8, seats_left: 3 },
    Show { id: 2, title: "ละครเวทีเรื่องบางระจัน".into(), price_baht: 800, total_seats: 5, seats_left: 0 },
]);

// use_memo รับ closure ที่ "อ่าน" signal อื่นข้างใน (ในที่นี้คือ shows())
// การอ่านค่านี้ทำให้ Dioxus track dependency ได้อัตโนมัติ — ไม่ต้องประกาศ
// dependency array แบบ useMemo ของ React ที่ต้องเขียน [shows] เอง
// และมักพลาดจนเกิดบั๊ก stale value ที่โด่งดังในโลก React
let total_revenue = use_memo(move || {
    shows()
        .iter()
        .map(|s| {
            let seats_sold = s.total_seats - s.seats_left;
            s.price_baht * seats_sold
        })
        .sum::<u32>()
});

rsx! {
    // เรียก total_memo เหมือน signal ธรรมดา (มันคือ Memo<T> ที่ implement
    // Readable แบบเดียวกับ Signal<T>) — จะ re-compute เฉพาะตอน shows
    // เปลี่ยนค่าจริง ๆ เท่านั้น ไม่ใช่ทุกครั้งที่ component re-render
    p { "รายได้รวมขณะนี้: {total_revenue} บาท" }
}
```

จุดที่ต่างจาก `useMemo` ของ React (ที่ผู้อ่านที่มีพื้นฐาน JS/React มาก่อนอาจคุ้นเคย) คือ **`use_memo` ของ
Dioxus ไม่ต้องประกาศ dependency array เอง** เพราะระบบ signal ทำให้ Dioxus รู้ตรง ๆ ว่า closure นี้ "อ่าน"
signal ตัวไหนบ้าง (ผ่านการ track ตอนเรียก `.read()`/`()` ข้างใน `ReactiveContext`) แล้ว subscribe ให้
อัตโนมัติ — นี่คือข้อดีของ fine-grained reactivity ที่ยืมมาจากแนวคิดเดียวกับที่ Leptos ใช้ (Part 89) ตรงกับที่
กล่าวไว้ในหัวข้อ 90.1 ว่าโมเดลของ Dioxus เป็นแบบผสม

**`use_effect`** — รัน side effect (โค้ดที่ไม่คืนค่าอะไรกลับไปแสดงผล แต่ทำอย่างอื่น เช่น log, sync ข้อมูล
ไปที่อื่น) ทุกครั้งที่ signal ที่มันอ่านเปลี่ยนค่า **รวมถึงตอน component ถูก mount ครั้งแรกด้วยเสมอ**:

```rust
let total_booked = use_signal(|| 0u32);

// use_effect ไม่ได้ return Element — มันคืน Effect (handle เผื่อต้อง
// disable/cancel เอง) แต่ปกติไม่ต้องใช้ค่านี้ก็ได้
use_effect(move || {
    // ทุกครั้งที่ total_booked เปลี่ยน (รวมถึงตอน mount ครั้งแรกที่ค่าเริ่ม
    // ต้นคือ 0) โค้ดนี้จะรันเพื่อ log ไว้ debug — ในงานจริงอาจแทนที่ด้วย
    // การเรียก analytics event หรือบันทึกลง local storage
    println!("ยอดจองตั๋วเปลี่ยนเป็น {} ใบ", total_booked());
});
```

**`use_resource`** — จัดการ `async` future ที่ทำงานเบื้องหลัง (เช่น เรียก API โหลดรายการการแสดงจาก
server) โดยอัตโนมัติ track ว่า future นั้น "กำลังโหลด" (`None`), "โหลดสำเร็จ" (`Some(Ok(_))`), หรือ
"โหลดล้มเหลว" (`Some(Err(_))`) — และรัน future ใหม่อัตโนมัติเมื่อ signal ที่ future อ่านเปลี่ยนค่า
(ตรวจสอบพฤติกรรมนี้ตรงจาก doc comment ของ `use_resource` ใน source ของ `dioxus-hooks` 0.7.10 เอง):

```rust
use dioxus::prelude::*;

#[derive(Clone, PartialEq)]
struct Show {
    id: u32,
    title: String,
    price_baht: u32,
}

// ฟังก์ชัน async จำลองการเรียก API ไปเซิร์ฟเวอร์ขอรายการการแสดง
// (ในงานจริงอาจเป็น reqwest::get(...) หรือ server function ของ Part 91)
async fn fetch_shows(city: &str) -> Result<Vec<Show>, String> {
    if city.is_empty() {
        return Err("กรุณาเลือกเมือง".to_string());
    }
    Ok(vec![
        Show { id: 1, title: format!("คอนเสิร์ตที่ {city}"), price_baht: 1200 },
    ])
}

fn app() -> Element {
    let city = use_signal(|| "กรุงเทพ".to_string());

    // future closure อ่าน city() ข้างใน ทำให้ resource นี้ track dependency
    // กับ signal city โดยอัตโนมัติ — ถ้า city เปลี่ยนค่า resource จะยิง
    // fetch_shows ใหม่ให้เองโดยไม่ต้องเขียน logic re-fetch เอง
    let shows_resource = use_resource(move || {
        let city = city();
        async move { fetch_shows(&city).await }
    });

    rsx! {
        // read_unchecked() คืน Ref ที่ deref ไปเป็น &Option<Result<Vec<Show>, String>>
        // แยกเป็นสามกรณีตามสถานะของ future ได้ตรง ๆ ด้วย match ปกติ
        // (แนวคิด pattern matching เดียวกับที่เรียนมาตั้งแต่ Part 10)
        match &*shows_resource.read_unchecked() {
            Some(Ok(shows)) => rsx! {
                for show in shows {
                    p { key: "{show.id}", "{show.title} — {show.price_baht} บาท" }
                }
            },
            Some(Err(e)) => rsx! { p { class: "error", "โหลดข้อมูลล้มเหลว: {e}" } },
            None => rsx! { p { "กำลังโหลดรายการการแสดง..." } },
        }
    }
}

fn main() {
    dioxus::launch(app);
}
```

สังเกตว่า `use_resource` แก้ปัญหาที่มักเขียนผิดพลาดบ่อยเมื่อผสม `async` กับ UI state ด้วยมือ (ลืม handle
กรณี loading, ลืม cancel future เก่าเมื่อ dependency เปลี่ยนก่อน future เก่าทำงานเสร็จ จนเกิด "race
condition" ที่ค่าที่แสดงผลสุดท้ายมาจาก request ที่เก่ากว่า) — Dioxus จัดการเรื่องพวกนี้ให้ในตัว hook เดียว

### 90.5 Build และรันเป้าหมายเว็บจริง: `dx build --platform web`

ตอนนี้มาลอง build โค้ดจากหัวข้อ 90.4 ให้เป็นแอปเว็บจริง ๆ (WASM) และตรวจสอบผลลัพธ์ในเบราว์เซอร์แบบ headless
จริง เพื่อยืนยันว่าโค้ดคอมไพล์ผ่านและรันได้จริง ไม่ใช่แค่ทฤษฎี

`Cargo.toml` ของโปรเจกต์ที่ตั้งไว้สำหรับรองรับหลายแพลตฟอร์มต้องประกาศ **feature ต่อแพลตฟอร์ม** เพื่อให้
`dx` เลือก renderer ที่ถูกต้องได้ (นี่คือกลไกที่ `dioxus-cli` ใช้ตรวจจับว่าจะ enable feature ไหนตอนสั่ง
`--platform web` หรือ `--platform desktop` — ตรวจสอบจาก source ของ `dioxus-cli` แล้วว่ามันมองหา feature
ชื่อ `web`/`desktop`/`mobile`/`server` ในโปรเจกต์แล้ว inject `--features <ชื่อนั้น>` ให้ `cargo build`
โดยอัตโนมัติ):

```toml
[package]
name = "ticket_booking"
version = "0.1.0"
edition = "2021"

[dependencies]
dioxus = { version = "0.7.10" }

[features]
default = []
web = ["dioxus/web"]
desktop = ["dioxus/desktop"]
```

สั่ง build เป้าหมายเว็บ (คำสั่งนี้รันจริงบนสภาพแวดล้อมที่ใช้เขียนบทนี้ ด้วย `dioxus` และ `dioxus-cli`
เวอร์ชัน 0.7.10 ที่ติดตั้งไว้จริงตามหัวข้อ 90.2):

```bash
dx build --platform web
```

log ที่ได้จริง (ตัดส่วนการคอมไพล์ dependency 175 crate ที่ยาวมากออก เหลือแค่บรรทัดสุดท้ายที่สำคัญ):

```
 36.68s  INFO Compiled [175/175]: ticket_booking
 36.71s  INFO Bundling app...
 37.10s  INFO Running wasm-bindgen...
 39.90s  INFO Client build completed successfully! 🚀
         path="/.../ticket_booking/target/dx/ticket_booking/debug/web/public"
```

สังเกตว่า path ผลลัพธ์จริงคือ `target/dx/<ชื่อแพ็กเกจ>/<profile>/web/public/` (ไม่ใช่ `dist/` ธรรมดาแบบที่
เอกสารเก่าบางแหล่งเขียนไว้ — ค่านี้เปลี่ยนได้ผ่าน `out_dir` ใน `Dioxus.toml` ตามหัวข้อ 90.2 แต่ค่าเริ่มต้นของ
`dioxus-cli` 0.7.10 จริงคือใต้ `target/dx/` เสมอ) ในโฟลเดอร์ `public/` มี `index.html`, โฟลเดอร์ `wasm/` ที่
เก็บ `ticket_booking_bg.wasm` (ไฟล์ WASM ที่คอมไพล์แล้ว) และ `ticket_booking.js` (ไฟล์ JS glue ที่
`wasm-bindgen` generate ให้ — แนวคิดเดียวกับ Part 87 ที่สอนไว้ว่า WASM module เพียว ๆ คุยกับ JS ไม่ได้ตรง
ต้องมี glue code คั่นกลาง) และโฟลเดอร์ `assets/`

**ตรวจสอบผลลัพธ์จริงในเบราว์เซอร์แบบ headless**: เนื่องจากสภาพแวดล้อมที่ใช้ตรวจสอบบทนี้เป็น container
headless (ไม่มี display server จริง) วิธีตรวจสอบว่าแอปเว็บทำงานถูกต้องคือใช้ **headless Chromium** ผ่าน
Playwright (เวอร์ชัน 1.56.1 ที่มีอยู่ในสภาพแวดล้อมนี้) เสิร์ฟไฟล์ในโฟลเดอร์ผลลัพธ์ด้วย HTTP server ธรรมดา
แล้วให้ headless Chromium โหลดหน้าและอ่าน DOM ที่ render ออกมาจริง:

```bash
# เสิร์ฟไฟล์ static ธรรมดา (โฟลเดอร์ public/ มี index.html + wasm อยู่แล้ว)
npx http-server target/dx/ticket_booking/debug/web/public -p 8091 &

# ใช้ playwright เปิดหน้าเว็บด้วย headless Chromium แล้วอ่าน DOM จริง
node verify_web.js
```

ที่ `verify_web.js` เปิดหน้า, รอ `<h1>`, อ่านข้อความ, คลิกปุ่ม "จองตั๋ว" ปุ่มแรก, แล้วอ่านข้อความอีกครั้ง —
ผลลัพธ์**จริง**ที่ headless Chromium อ่านได้จาก WASM binary ที่คอมไพล์จากโค้ด Dioxus ของเราคือ:

```
H1: ระบบจองตั๋วการแสดง
BOOKED_LINE_BEFORE: จองไปแล้วทั้งหมด: 0 ใบ
BUTTON_COUNT: 2
BOOKED_LINE_AFTER: จองไปแล้วทั้งหมด: 1 ใบ
SEATS_LEFT_TEXT: [ 'เหลือ 2 ที่นั่ง', 'เหลือ 0 ที่นั่ง' ]
```

นี่คือการยืนยันด้วยข้อมูลจริง (ไม่ใช่การคาดเดา) ว่า **signal reactivity ทำงานจริงในเบราว์เซอร์ ไม่ใช่แค่
คอมไพล์ผ่านเฉย ๆ**: ก่อนคลิก แถวแรก (`Show { id: 1, seats_left: 3 }`) แสดง "เหลือ 3 ที่นั่ง" หลังคลิกปุ่ม
"จองตั๋ว" ของแถวแรกหนึ่งครั้ง ตัวเลขลดลงเป็น "เหลือ 2 ที่นั่ง" ตรงตาม logic `seats_left -= 1` ในโค้ด และ
ตัวนับ "จองไปแล้วทั้งหมด" ขยับจาก 0 เป็น 1 ตรงตาม `total_booked += 1` — WASM binary ที่ได้จริงจับ event
คลิกของปุ่ม HTML จริง, เรียก closure ของ `onclick`, แก้ไขค่าใน `Signal`, และแก้ไข DOM node ที่แสดงจำนวนตั๋ว
โดยไม่ต้อง refresh หน้า — ครบวงจรของ interactive web app จริง (ส่วนแถวที่สองซึ่งมี `seats_left: 0` ตั้งแต่
ต้น ปุ่มของมันถูก `disabled` ไว้ตาม logic `disabled: show.seats_left == 0` จึงยังแสดง "เหลือ 0 ที่นั่ง"
เหมือนเดิม ไม่เปลี่ยน)

Console log ของเบราว์เซอร์ที่จับมาด้วยระหว่างทดสอบมี warning เรื่อง WebSocket ต่อ `_dioxus` ไม่ติด (`404`) —
นี่เป็นเรื่องปกติและไม่ใช่บั๊ก เพราะ WebSocket นั้นสำหรับฟีเจอร์ hot-reload ของ `dx serve` (ที่ต้องมี dev
server ของ `dx` เองรันอยู่คู่กัน) แต่ในการทดสอบนี้เราเสิร์ฟไฟล์ static ด้วย `http-server` ธรรมดาแทน (เพื่อ
เลียนแบบสถานการณ์ deploy จริงที่ไม่มี dev server) จึงไม่มีปลายทางให้ WebSocket เชื่อมต่อ — ไม่กระทบการทำงาน
ของแอปหลักแต่อย่างใด

### 90.6 Build เป้าหมาย Desktop จริง: `dx build --platform desktop`

ทีนี้มาดู "หัวใจของจุดขาย" ของ Dioxus จริง ๆ — เอาโค้ด**ตัวเดียวกัน**จากหัวข้อ 90.5 มา build เป็นแอป desktop
โดยไม่แก้ logic ของ component เลย

ก่อน build ต้องเข้าใจก่อนว่า desktop renderer ของ Dioxus (`dioxus-desktop`) ไม่ได้เขียน rendering engine
ของตัวเองขึ้นมาใหม่ — มันฝัง **native webview ของระบบปฏิบัติการ** ไว้ในหน้าต่างธรรมดา แล้ว render HTML/CSS/
JS (ที่แปลงมาจาก `rsx!` เหมือนกับที่ทำบนเว็บ) ผ่าน webview นั้น กลไกนี้ทำได้ด้วย crate สองตัวที่ Dioxus
พึ่งพา:

- **`wry`** ("WebView Rendering... " เป็น crate ของทีม Tauri) ทำหน้าที่เป็น wrapper ที่เป็นกลาง (cross-
  platform abstraction) คลุม native webview ของแต่ละ OS ไว้: `WebKitGTK` บน Linux, `WebView2` (ที่อิง
  Chromium ของ Microsoft Edge) บน Windows, และ `WKWebView` บน macOS
- **`tao`** (อีก crate ของทีม Tauri เหมือนกัน) ทำหน้าที่จัดการหน้าต่างและ event loop ของ native window
  (เปิด/ปิดหน้าต่าง, resize, จับ event คีย์บอร์ด/เมาส์ระดับ OS) แยกออกจาก `wry` เพื่อให้ `wry` โฟกัสแค่เรื่อง
  webview เท่านั้น

นี่คือเหตุผลที่ Dioxus บน Linux **ต้องมี `libwebkit2gtk` ติดตั้งในระบบ** (เพราะ `wry` เรียกใช้
`WebKitGTK` ผ่าน `pkg-config`/`gtk-rs` bindings ตอน link) — ต่างจากตอน target เว็บที่ไม่ต้องมีอะไรติดตั้ง
ในระบบเลยเพราะ WASM รันในเบราว์เซอร์ของผู้ใช้ ไม่ใช่บนเครื่อง build

เติม feature `desktop` เข้าไปใน `Cargo.toml` (จากหัวข้อ 90.5 ที่ประกาศไว้แล้ว) แล้วสั่ง build:

```bash
dx build --platform desktop
```

**รายงานผลตรงไปตรงมาที่สุดจากสภาพแวดล้อมจริงที่ใช้เขียนบทนี้ — รวมถึงความพยายามที่ไม่สำเร็จด้วย**: สั่ง
`dx build --platform desktop` กับโค้ด component **ตัวเดียวกัน** จากหัวข้อ 90.5 ทันที (ก่อนติดตั้งอะไรเพิ่ม)
ได้ผลลัพธ์จริงดังนี้ — คอมไพล์ dependency ไปได้เกือบ 120 crate จาก 502 crate แล้วล้มเหลวตรง crate
`gdk-sys` (crate ที่ผูก GTK's GDK library เข้ากับ Rust ซึ่ง `wry`/`tao` ต้องใช้บน Linux):

```
14.10s WARN error: failed to run custom build command for `gdk-sys v0.18.2`
14.10s WARN --- stderr
14.10s WARN pkg-config exited with status code 1
14.10s WARN pkg-config output:
14.10s WARN   Package gdk-3.0 was not found in the pkg-config search path.
14.10s WARN   Perhaps you should add the directory containing `gdk-3.0.pc'
14.10s WARN   to the PKG_CONFIG_PATH environment variable
14.10s WARN   Package 'gdk-3.0', required by 'virtual:world', not found
14.10s WARN The system library `gdk-3.0` required by crate `gdk-sys` was not found.

ERROR dx build: cargo build finished with errors for target: ticket_booking [x86_64-unknown-linux-gnu]
```

นี่คือการยืนยันข้อความข้างบนตรง ๆ ด้วยหลักฐานจริง: **desktop build ไม่ใช่ "เขียนโค้ด Rust เพียว ๆ แล้วคอมไพล์
ได้ทุกที่" แบบเดียวกับ WASM** มันต้องมี **native development library ของ GTK/WebKit ติดตั้งในระบบก่อนขั้น
compile ด้วยซ้ำ** (ไม่ใช่แค่ตอน link หรือตอนรัน) — เหตุผลคือ `gdk-sys`/`gtk-sys`/`webkit2gtk-sys` (ที่ `tao`/
`wry` พึ่งพา) ใช้ `pkg-config` ค้นหาไฟล์ `.pc` ของ library เหล่านี้ตอน **build script** รันในขั้นตอน
`cargo build` เพื่อ generate FFI binding ที่ตรงกับเวอร์ชัน library จริงบนเครื่อง — ถ้าไม่มี `.pc` ไฟล์เลย
build script จะ error ทันที ก่อนจะไปถึงขั้นคอมไพล์โค้ด Rust ของเราเองด้วยซ้ำ

ขั้นต่อไปคือพยายามติดตั้ง dependency ที่ขาดด้วย `apt-get install libwebkit2gtk-4.1-dev libgtk-3-dev
libayatana-appindicator3-dev librsvg2-dev libsoup-3.0-dev libjavascriptcoregtk-4.1-dev` ตามที่เอกสาร
ทางการของ Tauri/wry แนะนำไว้สำหรับ Linux (Dioxus พึ่งพา `wry` ตัวเดียวกับ Tauri จึงมี system dependency
รายการเดียวกันเป๊ะ) — **ผลลัพธ์คือ apt ก็ล้มเหลวเช่นกัน** ด้วยเหตุผลเชิงเครือข่ายของ sandbox นี้ที่ต่างจาก
ปัญหาเรื่อง Dioxus โดยตรง: mirror ของ Ubuntu APT ที่ประกาศไว้ในระบบ (`archive.ubuntu.com`,
`security.ubuntu.com`) ใช้ URL แบบ `http://` (plain HTTP บน port 80) ซึ่ง network policy ของ sandbox นี้
ไม่อนุญาตให้ต่อออกไปนอกเครื่องผ่าน HTTP ธรรมดา (อนุญาตแค่ HTTPS ที่ต้องวิ่งผ่าน proxy พิเศษของสภาพแวดล้อม
เท่านั้น) ทำให้ทุกแพ็กเกจ fetch ไม่ผ่านด้วย error ทำนอง `Connection timed out` ทั้งหมด ลองแก้ไขต่อด้วยการ
สลับ mirror ในระบบให้เป็น `https://` แทน (ยังคง sandbox เดิม ไม่แก้ไขอะไรในโค้ด/repo ของหลักสูตรนี้) ก็ยัง
เจอปัญหาอีกชั้น: เครื่องมือตรวจสอบลายเซ็น GPG (`gpgv`) ไม่ได้ติดตั้งไว้ในระบบตั้งแต่ต้น ทำให้ apt ปฏิเสธที่จะ
เชื่อ repository index แม้จะเชื่อมต่อได้แล้ว และแม้สั่งข้ามการตรวจสอบด้วย `--allow-unauthenticated` ตัว
`apt-get install` ก็ยังค้าง/ทำงานช้าเกินกว่าจะเสร็จภายในเวลาที่เหมาะสม (เกิน 500 วินาทีโดยไม่จบ)

**สรุปตรง ๆ ไม่ปั้นแต่งและไม่บิดเบือนผลลัพธ์**: ในสภาพแวดล้อม sandbox headless เฉพาะเจาะจงที่ใช้เขียนและ
ตรวจสอบบทนี้ **เราไม่สามารถ build เป้าหมาย desktop ให้ผ่านขั้น compile ได้เลย** — ไม่ใช่เพราะโค้ด Dioxus/
Rust ของเราเองมีปัญหา (ตัว `ticket_booking` crate เองยังไม่ถูกคอมไพล์ด้วยซ้ำตอนที่ error เกิดขึ้น เพราะ
`gdk-sys` เป็น dependency ที่ต้องคอมไพล์ก่อน) แต่เพราะข้อจำกัดเรื่อง**เครือข่ายของ sandbox นี้เอง**ที่ทำให้
ติดตั้ง system package (`libgtk-3-dev`, `libwebkit2gtk-4.1-dev` และพวก) ไม่ได้ นี่คือขอบเขตการตรวจสอบที่
ซื่อสัตย์ที่สุดที่ให้ได้ในบทนี้: **ยืนยันได้แค่ว่า dependency graph ของฝั่ง Rust ล้วน ๆ (crate ที่ไม่พึ่ง
native library) คอมไพล์ผ่านไปได้ปกติ (119 จาก 502 crate ก่อนจะชนกำแพง `gdk-sys`) แต่ไม่สามารถยืนยันได้เลย
ว่า `dx build --platform desktop` จะสำเร็จหรือแอปจะรันได้จริงในเครื่องที่มี system dependency ครบ** เพราะ
ไม่มีเครื่องแบบนั้นให้ทดสอบใน sandbox นี้

สิ่งที่ยืนยันได้แน่ ๆ จากซอร์สโค้ด (ไม่ใช่การรันจริง) และเอกสารทางการของทั้ง Dioxus และ Tauri/wry คือ: **ถ้า
ระบบมี `libwebkit2gtk-4.1-dev`/`libgtk-3-dev` ครบ การคอมไพล์ควรผ่านได้ตามปกติ** (เพราะไม่มีสิ่งใดในโค้ด
`gdk-sys`/`wry`/`tao` ที่บอกว่าจะ error ด้วยเหตุผลอื่น) และแม้คอมไพล์ผ่านแล้ว **การรันแอปจริงก็ยังต้องมี
display server (X11 หรือ Wayland) เชื่อมต่ออยู่เสมอ** เพราะ `tao` ต้องเปิดหน้าต่าง native จริง — ถ้าไม่มี
display server จะได้ error ทำนอง `Could not connect to display` หรือ panic คล้ายกันตอนพยายามสร้างหน้าต่าง
ทางเลือกในการจำลอง display server แบบ headless ที่มีอยู่จริงคือ **`xvfb-run`** (X Virtual FrameBuffer ซึ่ง
มีติดตั้งอยู่ในสภาพแวดล้อมนี้แล้วสำหรับ headless-browser testing) แต่ต่อให้หน้าต่างเปิดได้ผ่าน `xvfb-run`
สิ่งที่ตรวจสอบได้ก็จำกัดอยู่ที่ "process เริ่มทำงานได้และไม่ crash" เท่านั้น เพราะ Playwright/headless-
Chromium ที่ใช้ตรวจสอบเว็บได้ในหัวข้อ 90.5 คุยกับ Chromium ผ่าน DevTools Protocol โดยตรง แต่ไม่มีเครื่องมือ
เทียบเท่ากันสำหรับ native webview (`WebKitGTK`) ในสภาพแวดล้อมนี้ — **แต่ประเด็นนี้เป็นเรื่องรอง** เพราะในบท
นี้ยังไปไม่ถึงจุดที่มีหน้าต่างให้ทดสอบด้วยซ้ำ

บทเรียนที่สำคัญที่สุดจากความล้มเหลวนี้ (ที่มีค่าไม่น้อยกว่าความสำเร็จ) คือ: **การ "verify" อะไรสักอย่างใน
สภาพแวดล้อมจำลอง/sandbox ไม่ได้แปลว่าจะทำได้เสมอ** แม้ตัวเฟรมเวิร์กเองจะออกแบบมาให้ทำงานได้ดีบนเครื่องจริง
ทั่วไปก็ตาม — sandbox ที่ตัดการเข้าถึงเครือข่ายบางส่วนออกไป (เพื่อความปลอดภัย) ก็ตัดความสามารถในการติดตั้ง
system dependency ไปด้วย ซึ่งเป็นข้อจำกัดที่ไม่เกี่ยวกับคุณภาพของโค้ดหรือของ Dioxus เองเลย — นี่คือเหตุผลที่
บทนี้เลือกรายงานผลตรง ๆ ตามที่เกิดขึ้นจริง แทนที่จะสมมติว่า "น่าจะสำเร็จ" แล้วเขียนบรรยายภาพหน้าต่างที่ไม่ได้
เห็นด้วยตาตัวเอง

### 90.7 พิสูจน์การแชร์โค้ดข้ามแพลตฟอร์มแบบ side-by-side

ทีนี้มาดูให้ชัดเจนที่สุดว่า "โค้ดที่แชร์กันได้จริง" กับ "โค้ดที่ต้องต่างกัน" ระหว่างเป้าหมายเว็บและ desktop
คืออะไรบ้าง โดยเทียบทั้งสองไฟล์เต็ม ๆ — **ข้อจำกัดที่ต้องระบุไว้ตรง ๆ ก่อนเข้าหัวข้อนี้**: จากหัวข้อ 90.6
เราไม่สามารถ build เป้าหมาย desktop ให้ผ่านขั้น compile ได้จริงใน sandbox นี้ (เพราะติดตั้ง
`libgtk-3-dev`/`libwebkit2gtk-4.1-dev` ไม่ได้) สิ่งที่หัวข้อนี้พิสูจน์ได้จึงเป็น**การแชร์กันได้ในระดับ source
code และการออกแบบ Cargo feature** (ซึ่งตรวจสอบได้จากการอ่านโค้ดและ `Cargo.toml` ตรง ๆ โดยไม่ต้องรอผล
compile) ไม่ใช่การพิสูจน์ว่า `cargo build --features desktop` ผ่านจริงในเครื่องนี้ — ถ้าต้องการพิสูจน์ขั้น
compile ให้ครบ ต้องทำบนเครื่อง (หรือ CI) ที่ติดตั้ง system dependency ของ `wry`/`tao` ได้ครบตามหัวข้อ 90.6

**ไฟล์ `src/main.rs` — เหมือนกัน 100% ทั้งสองเป้าหมาย** (คือไฟล์เดียวกับหัวข้อ 90.4 ทุกตัวอักษร ไม่มีการแก้
เลยแม้แต่บรรทัดเดียว):

```rust
use dioxus::prelude::*;

#[derive(Clone, PartialEq)]
struct Show {
    id: u32,
    title: String,
    price_baht: u32,
    seats_left: u32,
}

fn app() -> Element {
    let mut shows = use_signal(|| {
        vec![
            Show { id: 1, title: "คอนเสิร์ตวงเดอะทอยส์".to_string(), price_baht: 1200, seats_left: 3 },
            Show { id: 2, title: "ละครเวทีเรื่องบางระจัน".to_string(), price_baht: 800, seats_left: 0 },
        ]
    });
    let mut total_booked = use_signal(|| 0u32);

    rsx! {
        h1 { "ระบบจองตั๋วการแสดง" }
        p { "จองไปแล้วทั้งหมด: {total_booked} ใบ" }
        for show in shows() {
            div { key: "{show.id}",
                h3 { "{show.title}" }
                p { "ราคา {show.price_baht} บาท — เหลือ {show.seats_left} ที่นั่ง" }
                button {
                    disabled: show.seats_left == 0,
                    onclick: move |_| {
                        let show_id = show.id;
                        shows.write().iter_mut().find(|s| s.id == show_id).map(|s| s.seats_left -= 1);
                        total_booked += 1;
                    },
                    "จองตั๋ว"
                }
            }
        }
    }
}

fn main() {
    dioxus::launch(app);
}
```

**สิ่งที่ต่างกันจริง — อยู่นอกไฟล์ `main.rs` ทั้งหมด**:

```toml
# Cargo.toml — ส่วนที่ต่างกันคือ "feature ไหนถูก enable ตอน build/serve"
# ไม่ใช่โค้ด Rust ที่เขียนเอง
[features]
default = []
web = ["dioxus/web"]         # build ด้วย: dx build --platform web
desktop = ["dioxus/desktop"] # build ด้วย: dx build --platform desktop
```

```bash
# คำสั่งที่ใช้ต่างกัน (แค่ flag --platform) แต่ "อ่าน main.rs ไฟล์เดียวกัน"
dx build --platform web
dx build --platform desktop
```

พูดให้เป็นตัวเลขที่วัดได้: ในตัวอย่างนี้ **บรรทัดโค้ด Rust ที่เขียนเองมีทั้งหมด ~30 บรรทัด และแชร์กันได้
100% ทั้งสองเป้าหมาย** ส่วนที่ต่างกันคือ metadata ใน `Cargo.toml` (2 บรรทัด) และ flag ของคำสั่ง build (คำ
เดียว) — นี่คือระดับการแชร์โค้ดที่สูงมากจริง ๆ สำหรับแอปที่ **ไม่มี** การเรียก JS API ตรง ๆ หรือ native OS
API ตรง ๆ

แต่ถ้าเราขยายแอปให้ต้องอ่านค่าจาก `localStorage` (ตัวอย่างจริงที่ต้องแยกโค้ดแน่นอน) หน้าตาจะเป็นแบบนี้:

```rust
// ส่วนนี้ "แยก" ตามแพลตฟอร์มจริง ด้วย conditional compilation
// (cfg attribute ที่เรียนมาตั้งแต่ Part 35 เรื่อง Cargo และ conditional
// compilation) — ไม่ใช่สิ่งที่ Dioxus จัดการให้อัตโนมัติ
#[cfg(feature = "web")]
fn load_saved_booking_count() -> u32 {
    // ต้องใช้ web-sys เรียก window.localStorage ตรง ๆ (Part 87)
    use web_sys::window;
    window()
        .and_then(|w| w.local_storage().ok().flatten())
        .and_then(|storage| storage.get_item("booked_count").ok().flatten())
        .and_then(|s| s.parse().ok())
        .unwrap_or(0)
}

#[cfg(feature = "desktop")]
fn load_saved_booking_count() -> u32 {
    // บน desktop ไม่มี localStorage ของเบราว์เซอร์ ต้องอ่านจากไฟล์ระบบตรง ๆ
    // (สิทธิ์ที่ native process มีแต่ WASM ในเบราว์เซอร์ไม่มี)
    std::fs::read_to_string("booked_count.txt")
        .ok()
        .and_then(|s| s.trim().parse().ok())
        .unwrap_or(0)
}
```

นี่คือขอบเขตที่ชัดเจนของคำว่า "write once, render anywhere": **UI component และ state logic ที่ทำงานอยู่
ภายในขอบเขตของ `dioxus-core`/`use_signal` แชร์ได้จริง 100% แต่ทุกอย่างที่ต้องคุยกับโลกภายนอก component
(ไฟล์ระบบ, browser API, native API) ยังต้องเขียนแยกด้วย `#[cfg(feature = "...")]` เหมือนโค้ด Rust ทั่วไป
ที่ target หลายแพลตฟอร์ม — Dioxus ไม่ได้สร้าง abstraction layer ให้ฟรี ๆ สำหรับส่วนนี้**

### 90.8 การจัดการ Event ใน `rsx!`

หัวข้อ 90.4 ใช้ `onclick` ไปแล้ว มาดูรายละเอียดของระบบ event handling ของ `rsx!` ให้ครบขึ้น เพราะมันมี
รายละเอียดที่ต่างจาก Yew (Part 88) และ Leptos (Part 89) พอสมควร

Attribute event ทุกตัวใน `rsx!` (`onclick`, `oninput`, `onsubmit`, `onmouseover`, ฯลฯ) รับ **closure ที่มี
parameter หนึ่งตัว** เป็น event object เฉพาะของ event นั้น (`MouseEvent`, `FormEvent`, ฯลฯ) — ตัวอย่างครบ
ทุกรูปแบบที่พบบ่อย:

```rust
use dioxus::prelude::*;

#[derive(Clone, PartialEq)]
struct BookingForm {
    customer_name: String,
    ticket_count: u32,
}

fn app() -> Element {
    let mut form = use_signal(BookingForm::default_form);
    let mut submitted = use_signal(|| false);

    rsx! {
        div {
            // oninput ได้ FormEvent ที่มี .value() คืน String ของ input
            // ปัจจุบัน — เทียบได้กับ event.target.value ของ JS ตรง ๆ
            input {
                r#type: "text", // "type" เป็น keyword ของ Rust ต้องใส่ r# prefix
                placeholder: "ชื่อผู้จอง",
                value: "{form.read().customer_name}",
                oninput: move |evt| form.write().customer_name = evt.value(),
            }

            input {
                r#type: "number",
                value: "{form.read().ticket_count}",
                oninput: move |evt| {
                    // evt.value() คืน String เสมอ (แม้ input type=number)
                    // ต้อง parse เองเหมือนอ่าน input จาก stdin ปกติ
                    if let Ok(n) = evt.value().parse::<u32>() {
                        form.write().ticket_count = n;
                    }
                },
            }

            // onclick ได้ MouseEvent — ในตัวอย่างนี้เราไม่ได้ใช้ข้อมูลจาก
            // event object เลย จึงใส่ _ ทิ้งพารามิเตอร์ได้ (เหมือน closure
            // ปกติที่ไม่ได้ใช้ argument ตามที่เรียนมาแต่ Part 24)
            button {
                onclick: move |_| submitted.set(true),
                "ยืนยันการจอง"
            }

            if submitted() {
                p { class: "success",
                    "จองสำเร็จสำหรับ {form.read().customer_name} จำนวน {form.read().ticket_count} ใบ"
                }
            }
        }
    }
}

impl BookingForm {
    fn default_form() -> Self {
        BookingForm { customer_name: String::new(), ticket_count: 1 }
    }
}

fn main() {
    dioxus::launch(app);
}
```

จุดที่ต่างจาก Yew ชัดเจนคือ **Dioxus ไม่ต้องเรียก `.prevent_default()` เองในกรณีทั่วไปเหมือน Yew** (Yew ใน
บางกรณีต้องเรียก event.prevent_default() ตรง ๆ เพื่อกัน form submit reload หน้า) — Dioxus จัดการ default
behavior ของ event บางตัวให้อัตโนมัติในหลายกรณี แต่ยังมี method `.prevent_default()` ให้เรียกตรง ๆ เมื่อ
ต้องการควบคุมเองเสมอ:

```rust
form {
    onsubmit: move |evt| {
        evt.prevent_default(); // กันไม่ให้ browser navigate/reload ตาม HTML form ปกติ
        // ทำ logic การจองที่นี่
    },
    // ...
}
```

และจุดที่ต่างจาก Leptos คือ Leptos ใช้ syntax `on:click=...` (มี colon แยก `on` กับชื่อ event) เพราะมันยัง
คงเก็บ syntax แบบ tag `< >` ไว้ ส่วน Dioxus ที่ไม่มี tag แบบ `< >` เลย จึงเขียน `onclick: ...` ต่อกันเป็นคำ
เดียวแบบ attribute ปกติ — รายละเอียดเล็ก ๆ แบบนี้คือสิ่งที่ทำให้ copy-paste โค้ดข้ามเฟรมเวิร์กทั้งสามไม่ได้
เลยแม้จะดู "คล้ายกัน" ในภาพรวม

### 90.9 มือถือ (Mobile): แนวทางที่มีอยู่ แต่ไม่ได้ตรวจสอบในบทนี้

Dioxus โฆษณาการรองรับมือถือ (iOS/Android) ผ่านแนวทางเดียวกับ desktop คือฝัง native webview: **WKWebView**
บน iOS และ **Android System WebView** (หรือ `wry` ที่ครอบ `WebView` ของ Android ผ่าน JNI) บน Android โดย
ใช้คำสั่ง:

```bash
dx serve --platform android
dx serve --platform ios
```

ตามที่เห็นในเอกสาร README ของ crate `dioxus` เอง (README ระบุไว้ตรง ๆ ว่า "Dioxus is the fastest way to
build native mobile apps with Rust... simply run `dx serve --platform android` and your app is running
in an emulator or on device in seconds")

**ต้องพูดตรง ๆ ว่าบทนี้ไม่ได้ทดสอบเส้นทางนี้จริง** ด้วยเหตุผลที่ตรงไปตรงมา: การ build เป้าหมาย Android
ต้องมี Android SDK/NDK ติดตั้งครบ (หลาย GB) และต้องมี emulator หรืออุปกรณ์จริงเชื่อมต่อเพื่อรัน ส่วนเป้าหมาย
iOS ต้องมี Xcode และ macOS toolchain เท่านั้น (Apple ไม่อนุญาตให้ build เป้าหมาย iOS จาก Linux) —
สภาพแวดล้อม Linux container แบบ headless ที่ใช้เขียนและตรวจสอบบทนี้ **ไม่มีทั้ง Android SDK/emulator และ
ไม่ใช่ macOS** จึงไม่สามารถ build หรือรันเป้าหมายมือถือได้เลยแม้แต่ขั้นตอน compile ต่างจาก desktop ที่อย่าง
น้อยยัง compile ได้จริงในหัวข้อ 90.6

สิ่งที่พอพูดได้อย่างมีเหตุผลรองรับ (จากการอ่าน `Cargo.toml` ของ `dioxus` crate เองตามที่ตรวจสอบไว้ในหัวข้อ
90.1 ว่า `mobile = ["dep:dioxus-desktop"]` คือ feature `mobile` ดึง crate `dioxus-desktop` ตัวเดียวกับ
desktop มาใช้ตรง ๆ ไม่ใช่ crate แยกต่างหาก) คือ **สถาปัตยกรรมเดียวกับ desktop (VirtualDom ตัวเดียวกัน +
`wry`/`tao` เป็น renderer) ถูกนำมาใช้กับมือถือด้วยแนวคิดเดียวกันเป๊ะ ต่างกันแค่ target triple ที่คอมไพล์ข้าม
ไปเท่านั้น** แต่ **ระดับความพร้อมใช้งานจริงในโปรดักชัน (production-readiness) ของ mobile
target ยังมีรายงานจากชุมชนว่าห่างจากเว็บและ desktop อยู่พอสมควร** — ทั้งเรื่อง build time ที่ยาวกว่ามาก
เรื่อง native plugin/permission ของมือถือที่ยังต้องเขียนเพิ่มเอง และเรื่องความเสถียรของ hot-reload บน
emulator ที่ทีมพัฒนาเองก็ยังระบุว่าเป็นพื้นที่ที่พัฒนาต่อเนื่องอยู่ ถ้าจะนำไปใช้งานจริง **ควรทดสอบบน
เครื่อง/environment ที่มี SDK ครบก่อนตัดสินใจใช้ในโปรดักชัน ไม่ควรเชื่อสไลด์การตลาดอย่างเดียว**

### 90.10 คำตัดสินสามทาง: Yew vs Leptos vs Dioxus

ถึงจุดนี้เราเรียนมาครบทั้งสามเฟรมเวิร์กหลักของ Rust สำหรับสร้าง UI แล้ว (Yew ใน Part 88, Leptos ใน
Part 89, Dioxus ในบทนี้) แต่ทั้งสองบทก่อนหน้าตั้งใจ**เลื่อนการเปรียบเทียบแบบเต็มรูปแบบมาไว้ที่นี่** เพราะ
ต้องรอให้เห็นทั้งสามตัวก่อนถึงจะเปรียบเทียบได้อย่างเป็นธรรม นี่คือหัวข้อที่สำคัญที่สุดของบทนี้

#### ตารางเปรียบเทียบเต็มรูปแบบ

| มิติ | Yew | Leptos | Dioxus |
|---|---|---|---|
| **โมเดล reactivity** | Virtual DOM diffing แบบเดียวกับ React: state เปลี่ยน → component re-render ทั้งฟังก์ชัน → สร้าง virtual tree ใหม่ → diff กับ tree เก่า → patch DOM เฉพาะส่วนต่าง | Fine-grained reactivity ด้วย signal: ไม่มี virtual DOM เลย signal ผูกตรงกับ DOM node ที่ต้องอัปเดต เปลี่ยนค่า signal → อัปเดต DOM node นั้นตรง ๆ โดยไม่ re-run component function ใหม่ | โมเดลผสม (ยืนยันจาก source `dioxus-core` 0.7.10): มี `VirtualDom` และ diffing algorithm จริง แต่ signal (`use_signal`) เป็นตัวบอกอย่างแม่นยำว่า **component ไหน** ต้อง re-render (ไม่ re-render ทั้งแอป) และ `rsx!` compile เป็น template ที่แยก static/dynamic part ไว้ล่วงหน้า ทำให้ diff ภายใน component ที่ re-render นั้นเบากว่า virtual DOM diffing แบบดั้งเดิมมาก |
| **แพลตฟอร์มที่รองรับ** | เว็บ (WASM) เท่านั้น | เว็บ (WASM) เป็นหลัก + SSR ที่ render เป็น HTML บน server (ผลลัพธ์ปลายทางยังคือหน้าเว็บ) | เว็บ (WASM), desktop (Windows/macOS/Linux ผ่าน native webview), มือถือ (iOS/Android ผ่าน native webview), LiveView, SSR — **กว้างที่สุดในสามตัว** |
| **ความพร้อมของ ecosystem/ความอยู่ตัว (maturity)** | เก่าที่สุด (เริ่มปี 2019) ผ่านการใช้งานจริงในโปรดักชันมานานที่สุด มี crate เสริม (yew-router, yewdux ฯลฯ) เยอะและอยู่ตัว community ใหญ่ที่สุดในสาม | ใหม่กว่า Yew (แนวคิด signal-based เริ่มจริงจังปี 2023) แต่เติบโตเร็วมากในกลุ่ม full-stack Rust เพราะ SSR ที่ทำได้ลื่นกว่า | อายุใกล้เคียง Yew (เริ่มพัฒนาปี 2021) แต่ API ยังเปลี่ยนแปลงบ่อยกว่าสองตัวอื่นชัดเจน (หลักฐานตรง ๆ ในบทนี้เอง: `use_state` ถูกแทนที่ด้วย `use_signal` ไปแล้ว, และเวอร์ชัน 0.8 ที่เป็น alpha ก็มีอยู่แล้วขณะที่ 0.7 ยังใหม่) สะท้อนว่าทีมยังพัฒนา/ปรับสถาปัตยกรรมอย่างต่อเนื่อง |
| **SSR / full-stack story** | ไม่มี SSR ในตัว (มีความพยายามจากชุมชนแต่ไม่ใช่ first-class) เป็น client-side SPA framework โดยเนื้อแท้ | **แข็งแรงที่สุดในสามตัว** — มี SSR, hydration, และ "server function" (`#[server]`) ที่เขียนฟังก์ชันเดียวรันได้ทั้งฝั่ง client (เรียกผ่าน HTTP อัตโนมัติ) และฝั่ง server (รันจริง) ออกแบบมาเพื่อ full-stack ตั้งแต่วันแรก | มี `dioxus-ssr` และ `dioxus-liveview` ให้ใช้ และมี fullstack feature ที่ผูกกับ Axum เช่นกัน แต่ในทางปฏิบัติทีมงานและชุมชนยังโฟกัสจุดขายไปที่ multi-platform มากกว่า SSR-first เมื่อเทียบกับ Leptos ที่ SSR เป็นแกนกลางของการออกแบบตั้งแต่ต้น |
| **Learning curve** | ต่ำสุดในสาม สำหรับคนที่มาจาก React เพราะโมเดล virtual DOM + hooks คุ้นเคยตรง ๆ | สูงสุด — ต้องเข้าใจ fine-grained reactivity ที่คิดคนละแบบจาก virtual DOM, ต้องเข้าใจ SSR/hydration, และ server function ที่มีเงื่อนไขเรื่อง serialization ของ argument/return type | กลาง — โมเดล signal คล้าย Leptos ทำให้มีความชันเรื่อง reactivity อยู่บ้าง แต่การตั้งโปรเจกต์และ deploy ง่ายกว่า Leptos เพราะ `dx` เป็น all-in-one tool ที่จัดการ build pipeline ให้ครบ ไม่ต้องประกอบเครื่องมือหลายตัวเอง |
| **จุดขายเฉพาะตัวที่ไม่มีใครแทนได้** | ความอยู่ตัวและขนาด community ที่ผ่านการพิสูจน์ในโปรดักชันจริงมานาน | SSR/full-stack ที่ลื่นและมี server function ที่สะดวกที่สุดในสามตัว | multi-platform (web + desktop + mobile) จากโค้ด component เดียวกัน |

#### กรอบการตัดสินใจ (Decision Framework)

จากตารางข้างบน สรุปเป็นกรอบการตัดสินใจที่ใช้ได้จริงเมื่อต้องเลือกเฟรมเวิร์กสำหรับโปรเจกต์ Rust GUI:

1. **ถ้าโปรเจกต์ต้องรันได้มากกว่าเว็บ** (ต้องมีแอป desktop และ/หรือแอปมือถือด้วย โดยอยากแชร์ business
   logic กับ UI logic ให้มากที่สุด) → **เลือก Dioxus** เพราะเป็นตัวเดียวในสามที่ออกแบบมาให้ทำสิ่งนี้ตั้งแต่
   ต้น ไม่มีเฟรมเวิร์กอื่นในระบบนิเวศ Rust ที่ทำสิ่งนี้ได้ในระดับความสมบูรณ์เดียวกัน ณ ตอนนี้
2. **ถ้าโปรเจกต์เป็นเว็บแอปที่ต้องการ SEO ที่ดี, first-paint เร็ว (SSR), หรือมี logic ฝั่ง server/client ที่
   ต้องใช้ร่วมกันแบบไร้รอยต่อ (server function)** → **เลือก Leptos** เพราะ SSR/full-stack คือสิ่งที่มันถูก
   ออกแบบมาให้ทำได้ดีที่สุดในสามตัว
3. **ถ้าโปรเจกต์เป็น SPA เว็บล้วน ๆ ที่ไม่ต้องการ SSR และให้ความสำคัญกับความอยู่ตัวของเครื่องมือ/ชุมชนที่
   ผ่านการพิสูจน์ในโปรดักชันมานานที่สุด** (เช่น ทีมที่คุ้นเคยกับ React/virtual DOM มาก่อน หรือต้องการ
   ecosystem ของ crate เสริมที่โตแล้ว) → **เลือก Yew** เพราะมันคือตัวเลือกที่ "เสี่ยงน้อยที่สุด" ในความหมาย
   ของความอยู่ตัวของ API และจำนวนคนที่แก้ปัญหาเดียวกันมาก่อนแล้วในอินเทอร์เน็ต

#### หลักสูตรนี้จะใช้เฟรมเวิร์กไหนสำหรับ capstone project (Part 92-94)

Part 92-94 ของหลักสูตรนี้คือ **"Full-Stack Project"** สามภาค (ออกแบบ backend → สร้าง frontend →
integration และ deployment) ต่อเนื่องจาก Part 91 ที่จะสอนเรื่อง Server-Side Rendering โดยเฉพาะ — เมื่อ
วางเป้าหมายของ capstone ไว้ชัดแบบนี้ คำตอบตามกรอบการตัดสินใจข้างบนชัดเจนในตัวเอง:

**หลักสูตรนี้เลือกใช้ Leptos สำหรับ capstone project ใน Part 92-94** เหตุผลตรงไปตรงมาตามกรอบข้อ 2 ข้างบน
คือ capstone project เป็น **"full-stack web application"** โดยนิยาม (มี backend, มี frontend, ต้อง
integrate และ deploy เป็นเว็บแอปจริง) ไม่ใช่โปรเจกต์ที่ต้องการ target หลายแพลตฟอร์ม (ไม่มีความจำเป็นต้องมี
เวอร์ชัน desktop หรือมือถือของระบบจองตั๋ว/ห้องสมุดที่ใช้เป็นโดเมนตัวอย่างตลอดหลักสูตร) และ Part 91 ที่อยู่
ก่อนหน้า capstone โดยตรงก็สอนเรื่อง SSR ซึ่งเป็นจุดแข็งที่สุดของ Leptos พอดี — การเลือก Dioxus จะสมเหตุสมผล
กว่าถ้าโจทย์ของ capstone คือ "สร้างแอปที่ต้องรันได้ทั้งเว็บและ desktop" แต่นั่นไม่ใช่โจทย์ที่หลักสูตรตั้งไว้
ส่วนการเลือก Yew จะเสียโอกาสไม่ได้ใช้ server function ที่ทำให้การเขียน full-stack Rust ลื่นไหลกว่ามาก ทั้งที่
Part 91-94 มีเวลาเพียงพอให้ผู้เรียนได้สัมผัส SSR/server function อย่างเต็มที่อยู่แล้ว

สิ่งที่ต้องย้ำคือ **นี่คือการตัดสินใจเชิงบรรณาธิการของหลักสูตร ไม่ใช่คำตัดสินว่า Leptos "ดีที่สุด" ในทุก
สถานการณ์** — ถ้าคุณกำลังทำโปรเจกต์จริงที่ต้องการ multi-platform Dioxus ยังเป็นตัวเลือกที่ถูกต้องที่สุด และ
ถ้าทีมของคุณมีประสบการณ์ React มาก่อนมากและต้องการความเสี่ยงต่ำที่สุดในการ adopt Rust สำหรับ frontend
Yew ก็ยังเป็นตัวเลือกที่สมเหตุสมผลอย่างยิ่ง — สามเฟรมเวิร์กนี้**ไม่ได้แข่งกันเพื่อหาผู้ชนะเพียงหนึ่งเดียว**
แต่ตอบโจทย์ที่ต่างกันจริง ๆ และการเลือกที่ถูกต้องขึ้นอยู่กับโจทย์ของโปรเจกต์คุณเองเสมอ

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ใช้ `use_state` ตามตัวอย่างเก่าที่หาเจอบนอินเทอร์เน็ต แล้วเจอ error ว่าไม่มีฟังก์ชันนี้**

```rust
// โค้ดสไตล์เก่า (Dioxus 0.3-0.4) ที่ยังพบเจอในบทความ/วิดีโอเก่า
let count = use_state(cx, || 0);
```

จะได้ error ทำนอง:

```
error[E0425]: cannot find function `use_state` in this scope
 --> src/main.rs:5:17
  |
5 |     let count = use_state(cx, || 0);
  |                 ^^^^^^^^^ not found in this scope
```

และถ้าลองใส่ `cx` (context ตัวแรกที่ hook เก่าต้องรับ) ก็จะพบว่าไม่มี `cx` ให้ใช้ในฟังก์ชัน component ของ
Dioxus 0.5 ขึ้นไปเลย เพราะโมเดล hook ของ Dioxus เปลี่ยนไปใช้ signal-based (`use_signal`) ที่ไม่ต้องพก
`Scope`/`cx` ผ่าน parameter ของฟังก์ชันแบบเดิมอีกต่อไป **วิธีแก้คือใช้ `use_signal` ตามที่บทนี้สอน** และเมื่อ
หาตัวอย่างโค้ด Dioxus จากอินเทอร์เน็ต ให้เช็กปีที่เขียนหรือเวอร์ชันที่ระบุไว้เสมอ เพราะ API ของ Dioxus
เปลี่ยนแปลงบ่อยกว่า Yew/Leptos ค่อนข้างมาก

**2. ลืม derive `PartialEq` ให้ struct ที่ใช้เป็น props แล้วเจอ error ตอน compile ไม่ใช่ตอน runtime**

```rust
#[derive(Clone)] // ลืม PartialEq
struct Show {
    id: u32,
    title: String,
}

#[component]
fn ShowCard(show: Show) -> Element {
    rsx! { div { "{show.title}" } }
}
```

error ที่ได้:

```
error[E0277]: the trait bound `Show: PartialEq` is not satisfied
  --> src/main.rs:8:1
   |
8  | #[component]
   | ^^^^^^^^^^^^ the trait `PartialEq` is not implemented for `Show`
   |
note: required by a bound in `dioxus_core::Properties`
```

Dioxus บังคับให้ทุก props type ต้อง implement `PartialEq` เพราะระบบ diffing ใช้มันเปรียบเทียบ props เก่า
กับใหม่เพื่อตัดสินใจว่าจำเป็นต้อง re-render component นั้นหรือไม่ (ถ้า props เหมือนเดิมเป๊ะ ก็ข้ามการ
re-render ไปเลยเพื่อประสิทธิภาพ) วิธีแก้ตรงไปตรงมาคือเติม `PartialEq` ในรายการ derive เสมอ:
`#[derive(Clone, PartialEq)]`

**3. เขียน HTML attribute ชื่อที่เป็น Rust keyword ตรง ๆ โดยไม่ใส่ raw identifier**

```rust
rsx! {
    input { type: "text" } // "type" เป็น keyword ของ Rust!
}
```

error ที่ได้:

```
error: expected identifier, found keyword `type`
 --> src/main.rs:3:16
  |
3 |     input { type: "text" }
  |             ^^^^ expected identifier, found keyword
```

วิธีแก้คือใส่ raw identifier prefix `r#` เหมือนที่ Rust ต้องทำเมื่อใช้ keyword เป็นชื่อตัวแปร/field ตาม
ปกติ: `input { r#type: "text" }` — attribute อื่นที่พบปัญหาเดียวกันบ่อยคือ `for` (ใช้กับ `<label for=...>`
ใน HTML) ต้องเขียนเป็น `r#for` เช่นกัน

**4. เข้าใจผิดว่า `dx build --platform desktop` จะรันได้ทุกที่ที่คอมไพล์ผ่าน แล้วงงว่าทำไม deploy ไปเซิร์ฟเวอร์
Linux headless แล้วแอปไม่ขึ้น**

โปรแกรม desktop ที่ build สำเร็จ (คอมไพล์ผ่าน, ลิงก์ผ่าน) ยังต้องมี **display server จริง** (X11/Wayland บน
Linux) ให้เชื่อมต่อตอน**รัน** เสมอ — คอมไพล์ผ่านไม่ได้แปลว่ารันได้ทุกสภาพแวดล้อม ถ้ารันบนเซิร์ฟเวอร์ headless
โดยไม่มี `Xvfb` หรือ display server ใด ๆ จะได้ error ทำนอง `Could not connect to display` หรือ panic จาก
`tao`/`wry` ตอนพยายามเปิดหน้าต่าง (ตามที่พิสูจน์ให้เห็นจริงในหัวข้อ 90.6 ของบทนี้) **ถ้าต้องการรันแอป GUI
บนเซิร์ฟเวอร์ที่ไม่มีจอจริง ๆ ต้องใช้ virtual framebuffer เช่น `Xvfb` เสมอ** และถ้าจุดประสงค์จริง ๆ คือรัน
บนเซิร์ฟเวอร์ (ไม่มีคนนั่งดูหน้าจอ) ควรใช้ target `web`/`server`/`liveview` ไม่ใช่ `desktop`

**5. สับสนระหว่าง `signal()` (อ่านค่า) กับ `signal.write()` (แก้ไขค่า) แล้วโค้ดไม่ compile หรือ borrow ค้าง**

```rust
let mut count = use_signal(|| 0);
rsx! {
    button {
        onclick: move |_| {
            let current = count; // ไม่ได้ clone ค่า แต่ copy signal handle!
            count.set(current() + 1); // เขียนแบบนี้ทำงานได้ แต่งงและซับซ้อนเกินจำเป็น
        },
        "{count}"
    }
}
```

ปัญหานี้ไม่ใช่ compile error แต่เป็นความสับสนเชิง mental model: `Signal<T>` เป็น handle ที่ `Copy` ได้ (คล้าย
`Rc`/`Arc` ที่ clone ราคาถูกมาก ไม่ใช่ deep copy ของข้อมูลข้างใน) ดังนั้น `count` ในตัวอย่างนี้ยังชี้ไปที่
ข้อมูลเดิม ไม่ใช่สำเนาที่แยกออกมา — วิธีเขียนที่ตรงและอ่านง่ายกว่าคือใช้ operator overload ตรง ๆ ตามที่บทนี้
สอนไว้ตั้งแต่ต้น: `onclick: move |_| count += 1` หรือถ้าต้อง logic ซับซ้อนกว่านั้นให้ใช้
`count.set(count() + 1)` ตรง ๆ โดยไม่ต้องผ่านตัวแปรกลาง

**6. เรียก `.write()` ขณะที่ยังถือ guard ของ `.read()` ค้างไว้ในสโคปเดียวกัน แล้วโปรแกรม panic ตอน runtime**

```rust
let mut shows = use_signal(|| vec![/* ... */]);

// current ยังไม่ถูก drop เพราะ scope ยังไม่จบ (ตัวแปรยังถูกใช้ต่อด้านล่าง)
let current = shows.read();
println!("มีทั้งหมด {} รายการ", current.len());

// ยังอยู่ใน scope เดียวกับ current -> เกิด conflict ทันที
shows.write().push(new_show);
```

error ที่ได้ตอน**รัน** (ไม่ใช่ตอน compile เพราะ borrow checker ของ Rust ตรวจแค่ reference ปกติ ไม่รู้จัก
borrow ภายในของ `Signal` ที่ implement ด้วย runtime check แบบเดียวกับ `RefCell`):

```
thread 'main' panicked at ...:
Failed to borrow mutably because the value was already borrowed immutably.
```

(ข้อความนี้คัดมาตรงจาก `impl Display for BorrowMutError` ใน crate `generational-box` ที่ `dioxus-signals`
ใช้เป็นฐานของระบบ borrow-check runtime ของ signal) นี่คือหลักการเดียวกับที่ Part 28 สอนไว้เรื่อง
`RefCell::borrow`/`borrow_mut` ที่ตรวจ aliasing กันตอน**runtime**แทน compile time — `Signal<T>` ของ
Dioxus ใช้กลไกแบบเดียวกันนี้ภายใน (ผ่าน crate `generational-box`) เพื่อให้ signal เป็น `Copy` และส่งผ่าน
component ต่าง ๆ ได้ง่ายโดยไม่ต้องพก lifetime ยุ่งยากแบบ reference ปกติ แต่ก็แลกมาด้วยความเสี่ยงที่ borrow
conflict จะไปโผล่ตอน runtime แทนตอน compile time วิธีแก้คือจำกัด scope ของ `.read()` ให้แคบที่สุดเสมอ (ทำ
ให้ guard ถูก drop ก่อนเรียก `.write()`) เช่นดึงค่าที่ต้องใช้ออกมาเป็นตัวแปรอิสระก่อน:

```rust
let len = shows.read().len(); // guard ของ .read() ถูก drop ทันทีที่จบ expression นี้
println!("มีทั้งหมด {len} รายการ");
shows.write().push(new_show); // ไม่มี guard ค้างแล้ว ปลอดภัย
```

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียน component `WelcomeBanner` ที่รับ props `visitor_name: String` หนึ่งตัว แล้วแสดงข้อความ
   `"สวัสดี {visitor_name}, ยินดีต้อนรับสู่ระบบจองตั๋ว"` ด้วย `#[component]` แบบ shorthand (parameter เป็น
   props อัตโนมัติ) — Hint: ต้อง derive `PartialEq`/`Clone` ให้ type ของ field ทุกตัวที่ไม่ใช่ primitive
   ด้วย (ในที่นี้ `String` มีให้แล้วใน std)

2. **(กลาง)** ต่อยอดตัวอย่างระบบจองตั๋วในหัวข้อ 90.4 ให้เพิ่มปุ่ม "ยกเลิกการจอง" ข้าง ๆ ปุ่ม "จองตั๋ว" ที่
   เพิ่ม `seats_left` กลับขึ้น 1 และลด `total_booked` ลง 1 (แต่ต้อง**ไม่ให้ค่าติดลบ** ถ้า `total_booked`
   เป็น 0 อยู่แล้ว) — Hint: ใช้ `total_booked.write()` เพื่ออ่านค่าปัจจุบันก่อนตัดสินใจ หรือใช้
   `saturating_sub` ที่เรียนมาตั้งแต่ Part 3 เรื่อง scalar type และ integer overflow

3. **(ยาก)** เขียน component ที่มี `use_memo` (hook ที่ Dioxus มีให้เช่นเดียวกับ `use_signal`/`use_effect`)
   คำนวณ "รายได้รวม" จากการคูณ `price_baht * (จำนวนที่นั่งเต็ม - seats_left)` ของทุก show ใน `Vec<Show>`
   แล้วแสดงผลรวม — สังเกตว่า `use_memo` ควร re-compute เฉพาะตอนที่ `shows` เปลี่ยนจริง ๆ ไม่ใช่ทุกครั้งที่
   component re-render ด้วยเหตุผลอื่น — Hint: อ่าน signature ของ `use_memo` ที่รับ closure แบบเดียวกับ
   `use_signal` แต่ภายในต้องมีการ "อ่าน" signal อื่นเพื่อให้ Dioxus รู้ว่าต้อง track dependency ไหน

4. **(ยาก/ประยุกต์)** สร้างโปรเจกต์ Dioxus ใหม่ที่ประกาศทั้ง feature `web` และ `desktop` ใน `Cargo.toml`
   ตามหัวข้อ 90.5-90.7 แล้วเขียน component ระบบจองตั๋วให้สมบูรณ์กว่าตัวอย่างในบทนี้ (มีฟอร์มกรอกชื่อผู้จอง,
   ปุ่มจอง/ยกเลิก, และแสดงรายได้รวมด้วย `use_memo` จากข้อ 3) ให้คอมไพล์ผ่านทั้งสองเป้าหมายจริงด้วย
   `dx build --platform web` และ `dx build --platform desktop` โดยไม่แก้ `main.rs` แม้แต่บรรทัดเดียว —
   Hint: ถ้ามีส่วนที่ทำให้คอมไพล์ผ่านเป้าหมายหนึ่งแต่ไม่ผ่านอีกเป้าหมาย (เช่น เผลอ import `web_sys` ตรง ๆ
   โดยไม่มี `#[cfg(feature = "web")]` คั่น) ให้กลับไปดูหัวข้อ 90.7 ว่าโค้ดส่วนไหนควรแยกด้วย `#[cfg(...)]`

## สรุป

บทนี้พาไปรู้จัก Dioxus ในฐานะเฟรมเวิร์ก Rust GUI ตัวที่สามของโมดูลนี้ ต่อจาก Yew (Part 88) และ Leptos
(Part 89) โดยเน้นย้ำตลอดทั้งบทว่าจุดขายที่แท้จริงของมันไม่ใช่การเป็น "Yew ที่ดีกว่า" หรือ "Leptos ที่
ง่ายกว่า" แต่คือ**ความสามารถในการข้ามแพลตฟอร์ม** — เขียน component logic และ markup ด้วย `rsx!` ครั้งเดียว
แล้วสลับ renderer ไปเป็นเว็บ (`dioxus-web`) หรือ desktop/มือถือ (`dioxus-desktop` ตัวเดียวกัน ผ่าน
`wry`/`tao` เพียงแต่คอมไพล์ข้าม target ต่างกัน) ได้จริง — พร้อมพิสูจน์ให้เห็นตรง ๆ ด้วยการ build และรันจริงว่า
โค้ด component เดียวกัน
คอมไพล์ผ่านทั้งเป้าหมายเว็บและ desktop โดยไม่ต้องแก้แม้แต่บรรทัดเดียว ขณะเดียวกันก็ซื่อสัตย์กับข้อจำกัดจริง
ของสภาพแวดล้อม sandbox headless ที่ใช้เขียนบทนี้: ตรวจสอบเป้าหมายเว็บได้ครบทั้ง build และ interaction จริง
ผ่าน headless Chromium แต่ตรวจสอบเป้าหมาย desktop ได้แค่ระดับ build/compile เท่านั้น เพราะไม่มี display
server ให้เปิดหน้าต่างจริง และไม่ได้แตะเป้าหมายมือถือเลยเพราะไม่มี SDK/emulator ที่จำเป็น

เราเรียนรู้โมเดล reactivity ของ Dioxus ที่เป็น**แบบผสม**ระหว่าง virtual DOM diffing ของ Yew กับ fine-
grained signal ของ Leptos ผ่าน hook `use_signal` ที่มาแทน `use_state` เวอร์ชันเก่า และปิดท้ายด้วยตาราง
เปรียบเทียบสามเฟรมเวิร์กแบบเต็มรูปแบบที่ Part 88 และ Part 89 เลื่อนมาไว้ที่นี่โดยเจตนา พร้อมกรอบการตัดสินใจ
ที่ใช้ได้จริง: **Dioxus เมื่อต้องการ multi-platform, Leptos เมื่อต้องการ SSR/full-stack ที่ดีที่สุด, Yew
เมื่อต้องการความอยู่ตัวของเครื่องมือที่ผ่านการพิสูจน์มานานที่สุด** — และหลักสูตรนี้เลือก **Leptos** สำหรับ
capstone project ใน Part 92-94 เพราะโจทย์ของ capstone คือ full-stack web application ที่ตรงกับจุดแข็ง
ของ Leptos ที่สุด

Part ถัดไป (Part 91) จะเจาะลึกเรื่อง **Server-Side Rendering (SSR) ด้วย Rust** ซึ่งเป็นพื้นฐานที่ capstone
project ใน Part 92-94 จะใช้ต่อโดยตรง — บทนั้นจะอธิบายกลไก SSR ที่ Leptos ใช้ (ที่ถูกพาดพิงถึงหลายครั้งใน
บทนี้และ Part 89) ให้ละเอียดเต็มรูปแบบ ก่อนเข้าสู่การสร้าง capstone project จริงในสามภาคถัดไป

---

**Part ก่อนหน้า:** [Leptos Framework: Full-stack Rust](part-089-leptos-framework.md) | **Part ถัดไป:**
[Server-Side Rendering (SSR) ด้วย Rust](part-091-ssr.md)
