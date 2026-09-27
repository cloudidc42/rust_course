# Part 89: Leptos Framework: Full-stack Rust

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายความแตกต่างเชิงสถาปัตยกรรมที่**แท้จริง**ระหว่าง Leptos กับ Yew (Part 88) ได้อย่างถูกต้อง: Yew ใช้โมเดล **Virtual DOM + diffing** (component re-run ทั้งฟังก์ชันเมื่อ state เปลี่ยน แล้วเทียบ tree ใหม่กับ tree เก่าเพื่อหา DOM mutation ที่จำเป็น) ส่วน Leptos ใช้โมเดล **fine-grained reactivity** (component function รันครั้งเดียวตอน mount เพื่อสร้าง DOM จริงและ "effect" ที่ผูกกับ signal เฉพาะจุด เมื่อ signal เปลี่ยนจะมีแค่ effect ที่ subscribe อยู่เท่านั้นที่รันใหม่ และมันเขียนเข้า DOM node เดิมตรง ๆ — **ไม่มีขั้นตอน diffing เลย**) พร้อมพิสูจน์ด้วยการทดสอบจริงในเบราว์เซอร์ว่า DOM Text node object เดิมถูกใช้ซ้ำ ไม่ใช่สร้างใหม่
- ติดตั้งโปรเจกต์ Leptos ได้ทั้งสองแนวทาง (CSR ล้วน ๆ ด้วย `wasm-pack`/`trunk`, และแบบ full-stack ด้วย `cargo-leptos`) เขียน component แรกด้วย `view!` macro ที่คอมไพล์ผ่านและรันได้จริงในเบราว์เซอร์
- ใช้ signal (`signal()`), การอ่าน (`.get()`) และการเขียน (`.set()`/`.update()`) ได้อย่างถูกต้องตาม API ปัจจุบันของ Leptos 0.8 — และรู้ว่า `create_signal()`/`create_memo()` ที่เจอในบทความ/ตัวอย่างเก่าถูก deprecate ไปแล้วเพราะเหตุผลอะไร
- สร้าง derived signal และ `Memo` ที่ recompute เฉพาะเมื่อ dependency ที่มันอ่านจริง ๆ เปลี่ยนค่า พร้อมพิสูจน์ด้วย log จริงว่า memo "เงียบ" เมื่อ signal อื่นที่ไม่เกี่ยวข้องเปลี่ยนค่า
- เขียน component ที่รับ props ได้ด้วย `#[component]` (เทียบเคียงกับ `#[derive(Properties)]` ของ Yew ใน Part 88) และเข้าใจ **server function** (`#[server]`) ซึ่งเป็นจุดขายหลักของ Leptos — ฟังก์ชันที่ compile เป็นโค้ดฝั่ง server จริง แต่เรียกจากฝั่ง client ได้เหมือนฟังก์ชัน async ธรรมดา โดย Leptos สร้างทั้ง HTTP endpoint ฝั่ง server และโค้ด fetch ฝั่ง client ให้อัตโนมัติ — พร้อมตัวอย่างที่ query/mutate ฐานข้อมูล PostgreSQL จริงผ่าน SQLx (ต่อจาก Part 70)
- อธิบายความแตกต่างระหว่าง Client-Side Rendering (CSR) กับ Server-Side Rendering (SSR) ของ Leptos ได้ รวมถึงกลไก **hydration** (การที่ WASM ฝั่ง client "จับคู่" กับ HTML ที่ server ส่งมาแล้วผูก event listener เข้าไป โดยไม่ทำลาย DOM เดิมแล้วสร้างใหม่) และรันแอป Leptos SSR จริงที่ฝังอยู่ภายใน Axum Router เดียวกับ route ธรรมดาที่เรียนมาตั้งแต่ Module 4
- อธิบายแนวคิด **islands architecture** (partial hydration — ส่งเฉพาะส่วนที่ต้อง interactive จริง ๆ เป็น WASM ไปให้ client ส่วนที่เหลือยังเป็น static HTML) ในระดับที่เพียงพอต่อการอ่านโค้ดตัวอย่างและตัดสินใจว่าแอปของตัวเองควรพิจารณาใช้หรือไม่ และสรุปเปรียบเทียบ Leptos กับ Yew (Part 88) ได้อย่างเป็นธรรม (จุดแข็ง/จุดอ่อนของทั้งคู่) โดยไม่ตัดสินว่าตัวใด "ดีที่สุด" ก่อนที่จะเห็น Dioxus ใน Part 90

## ความรู้ที่ต้องมีมาก่อน

- **Part 88 (Yew Framework: Frontend ด้วย Rust)**: บทนี้อ้างอิงโมเดล Virtual DOM ของ Yew ตลอดเวลาเพื่อเทียบกับ fine-grained reactivity ของ Leptos ถ้ายังไม่ผ่าน Part 88 ควรอ่านก่อน เพราะการเข้าใจว่า Leptos "ต่างจาก Yew อย่างไร" คือหัวใจของบทนี้
- **Part 36 (Macros: Declarative Macros)**: `view!` macro ของ Leptos (เหมือนกับ `html!` ของ Yew) เป็น procedural macro ที่แปลง syntax คล้าย JSX เป็นโค้ด Rust จริงตอน compile time — บทนี้อ้างอิงความเข้าใจพื้นฐานเรื่อง "macro ทำงานตอน compile ไม่ใช่ runtime" จาก Part 36 เพื่ออธิบายว่าทำไม `view!` ของ Leptos ไม่ได้สร้าง Virtual DOM เลยสักครั้ง
- **Part 62-66 (Axum เต็มรูปแบบ)**: หัวข้อเรื่อง Leptos SSR ภายใน Axum ใช้ `Router`, route, `State` ตรงตามที่ Part 62-66 สอนไว้ทุกประการ — บทนี้จะไม่อธิบาย mechanism พื้นฐานของ Axum ซ้ำ
- **Part 70 (Database: เชื่อมต่อ PostgreSQL ด้วย SQLx)**: ตัวอย่าง server function ที่คุยกับฐานข้อมูลใช้ `PgPool`, `sqlx::query!`/`query_as!`, และ transaction (`pool.begin()`/`commit()`) ตรงตามที่ Part 70 สอนไว้ทั้งหมด — ถ้ายังไม่แน่นเรื่อง SQLx ควรทวนก่อน เพราะบทนี้จะไม่สอน SQLx ใหม่ตั้งแต่ต้น
- **Part 48 (Tokio Runtime) และ Part 39 (Shared State)**: การผูก `PgPool` เข้ากับ context ของ server function เป็นรูปแบบหนึ่งของ shared state ข้าม async task ที่ Part 39/48 ปูพื้นไว้แล้ว
- **Part 57 (Serde เบื้องต้น)**: struct ที่ server function ส่งกลับไปให้ client ต้อง `Serialize`/`Deserialize` เพราะข้อมูลเดินทางผ่าน network จริง ๆ (ไม่ใช่แค่ function call ในหน่วยความจำเดียวกัน)
- **Part 24 (Closures)**: signal, memo, และ effect ของ Leptos ล้วนรับ closure เป็น argument (`move || ...`) ตลอดทั้งบท — บทนี้ถือว่าคุณเข้าใจเรื่อง `move`, การ capture ตัวแปร, และ `Fn`/`FnMut`/`FnOnce` จาก Part 24 มาแล้ว ไม่อธิบาย closure syntax ใหม่
- **Part 44-45 (Procedural Macros)**: `#[component]`, `#[server]`, และ `view!` ล้วนเป็น procedural macro (attribute macro และ function-like macro ตามลำดับ) — บทนี้ไม่สอนวิธีเขียน proc macro เอง แต่ใช้ความเข้าใจจาก Part 44-45 เพื่ออธิบายว่าทำไม macro เหล่านี้ถึง generate โค้ดคนละแบบได้ตามเงื่อนไข feature flag ที่ compile
- **Part 46-48 (Async/Await, Futures, Tokio Runtime)**: server function ต้องเป็น `async fn` เสมอ (หัวข้อ 89.6) และ Action/Resource ในบทนี้ทำงานบนพื้นฐาน `Future` ที่ Part 46-47 สอนไว้ — บทนี้ไม่อธิบาย `async`/`.await` พื้นฐานซ้ำ

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

Leptos เป็น ecosystem ที่ API เปลี่ยนเร็วมากในช่วงหลัง (โดยเฉพาะชื่อฟังก์ชันสร้าง signal/memo) จึงตรวจสอบทุกอย่างในบทนี้จากซอร์สโค้ดจริงของ crate ที่ resolve ได้จริงในเครื่อง ไม่ใช่จากความจำหรือบทความเก่า:

- ตั้งโปรเจกต์ scratch **หลายโปรเจกต์แยกไว้นอก repo ทั้งหมด** (ไม่กระทบไฟล์ใด ๆ ในหลักสูตร): โปรเจกต์ CSR ล้วน ๆ (ทดสอบทั้งด้วย `wasm-pack` และ `trunk`), โปรเจกต์ SSR+Axum+SQLx ที่ต่อมาขยายให้ compile ได้สอง feature (`ssr`/`hydrate`) จากซอร์สโค้ดชุดเดียวกัน, โปรเจกต์เปล่าสำหรับเทียบ "endpoint มือ" กับ server function, และสคริปต์ Playwright/Node แยกสำหรับขับเบราว์เซอร์จริง
- `cargo add leptos` บนเครื่องจริง (rustc/cargo 1.94.1) resolve ได้ **leptos 0.8.21** (ซึ่งดึง `reactive_graph 0.2.15`, `leptos_macro 0.8.18`, `tachys 0.2.19` เป็น dependency ภายใน) — เวอร์ชันนี้คือ snapshot ณ วันที่เขียนบทนี้ ถ้าคุณ `cargo add leptos` แล้วได้เลขเวอร์ชันอื่น ให้ยึดของคุณเป็นความจริงล่าสุด (ecosystem นี้ออกเวอร์ชันใหม่บ่อย)
- **อ่านซอร์สโค้ดจริงของ `reactive_graph-0.2.15/src/signal.rs` และ `computed.rs`** เพื่อยืนยันสถานะ deprecation ของ `create_signal`/`create_memo`/`create_rw_signal` แบบตรงตัวอักษร (จะยกมาเป็น quote ในหัวข้อ 89.3)
- **อ่านซอร์สโค้ดจริงของ `tachys-0.2.19/src/view/primitives.rs` และ `src/renderer/dom.rs`** เพื่อยืนยันกลไก "ไม่มี diffing" — เห็นโค้ดจริงที่เรียก `node.set_node_value(Some(text))` เขียนทับ DOM Text node เดิมตรง ๆ
- คอมไพล์และรัน **CSR ตัวจริง** ด้วย `wasm-pack build --target web` ได้ WASM bundle จริง เสิร์ฟด้วย `http-server` แล้วเปิดด้วย **headless Chromium จริงผ่าน Playwright** (`chromium.launch()`) คลิกปุ่มจริง อ่าน `textContent` จริง และที่สำคัญที่สุด — **จับ reference ของ DOM Text node object ก่อน/หลัง update แล้วเทียบด้วย `===`** เพื่อพิสูจน์ว่าเป็น node เดิมที่ไม่ได้ถูกสร้างใหม่ (ผลลัพธ์จริงที่ได้และวิธีอ่านผลจะอยู่ในหัวข้อ 89.1 และ 89.4)
- ตั้งฐานข้อมูล PostgreSQL จริงชื่อ `leptos_scratch` (ใช้ cluster เดียวกับที่ตั้งไว้ตั้งแต่ Part 70 — `pg_ctlcluster 16 main start`) สร้างตาราง `events` (โดเมนจองตั๋วงานสัมมนา ต่อเนื่องจากโดเมนที่ Part 62-70 ใช้) เขียน server function จริงสองตัว (query กับ mutation) ที่คุยกับฐานข้อมูลนี้ตรง ๆ
- คอมไพล์และรัน **SSR server จริง** (Axum + `leptos_axum` + SQLx) แล้วยิง `curl` จริงทั้ง route หน้าเว็บ (SSR HTML), route server function (query และ mutation ผ่าน `/api/...`), และ route Axum ธรรมดาที่เขียนมือ (`/healthz`) ที่อยู่ใน `Router` เดียวกัน — พิสูจน์ทั้ง SSR และ error path (จองตั๋วตอนที่นั่งเต็ม) ด้วยผลลัพธ์จริงจาก `curl`
- ขยายโปรเจกต์ SSR ให้ compile เป็น**สอง target จริง** (native binary ด้วย feature `ssr`, WASM ด้วย feature `hydrate`) แล้วเปิดหน้าเว็บด้วยเบราว์เซอร์จริงผ่าน Playwright จับ reference ของ DOM node ก่อน/หลัง hydration เทียบด้วย `===` และคลิกปุ่มที่ผูกกับ server function จริงจนเห็นแถวในฐานข้อมูล PostgreSQL เปลี่ยนค่าจริง (หัวข้อ 89.7-89.8)
- ทุก error message และผลลัพธ์ที่ยกมาในบทนี้คือสิ่งที่**รันจริงแล้วคัดลอกมา** — ถ้าคุณลองทำตามแล้วได้ผลต่างเล็กน้อย (เช่น hash suffix ของ route หรือ warning เพิ่มเติมจาก dependency version ใหม่กว่า) ให้ยึดผลจากเครื่องคุณเป็นความจริงล่าสุด

## เนื้อหา

### 89.1 Leptos คืออะไร: Fine-grained Reactivity เทียบกับ Virtual DOM ของ Yew

จาก Part 88 คุณได้เรียนรู้ Yew ซึ่งใช้โมเดลเดียวกับ React ในโลก JavaScript: **Virtual DOM + diffing** เมื่อ state ของ component เปลี่ยน (เช่นเรียก `state.set(...)` จาก `use_state`) Yew จะ**รัน component function ทั้งฟังก์ชันใหม่อีกครั้ง** function นั้นคืนค่าเป็น `Html` (ต้นไม้ของ `VNode` ที่เป็นโครงสร้างข้อมูลใน Rust ล้วน ๆ ไม่ใช่ DOM จริง) จากนั้น Yew เทียบ tree ใหม่กับ tree เก่าด้วย diffing algorithm เพื่อหาว่า "จริง ๆ แล้วอะไรเปลี่ยนบ้าง" แล้วค่อยแปลงส่วนต่างนั้นเป็นการเรียก DOM API จริงเท่าที่จำเป็น — diffing คือสิ่งที่ทำให้ Yew ไม่ต้อง re-render DOM ทั้งหน้าทุกครั้ง แต่**การรัน component function ใหม่ทุกครั้งที่ state เปลี่ยนคือขั้นตอนที่เกิดขึ้นแน่นอนเสมอ** ก่อนที่ diffing จะเริ่มทำงานด้วยซ้ำ

Leptos เลือกแนวทางที่ต่างไปโดยสิ้นเชิง เรียกว่า **fine-grained reactivity** (โมเดลเดียวกับที่ SolidJS ใช้ในโลก JavaScript ไม่ใช่โมเดลเดียวกับ React) หลักการคือ:

1. component function (เช่น `fn App() -> impl IntoView`) **รันเพียงครั้งเดียว** ตอน mount เท่านั้น — ไม่ใช่รันซ้ำทุกครั้งที่ signal เปลี่ยน
2. ตอนรันครั้งแรกนั้น `view!` macro จะ**สร้าง DOM node จริงขึ้นมาเลย** (ผ่าน `web_sys`) ไม่ใช่สร้างโครงสร้างข้อมูล Virtual DOM แบบ Yew
3. ทุกจุดใน `view!` ที่มีการอ่านค่า signal (เช่น `{count}`) macro จะสร้าง **reactive effect** ผูกกับ DOM node จุดนั้นโดยเฉพาะ — effect นี้จะ "subscribe" เข้ากับ signal ที่มันอ่านโดยอัตโนมัติ (นี่คือหัวใจของคำว่า fine-grained: การ subscribe เกิดขึ้น**ระดับ node เดี่ยว ๆ** ไม่ใช่ระดับ component)
4. เมื่อคุณเรียก setter ของ signal (`set_count.set(...)`) reactive graph จะแจ้งเตือน**เฉพาะ effect ที่ subscribe กับ signal ตัวนั้นจริง ๆ** ให้รันใหม่ effect นั้นจะเขียนค่าใหม่เข้า DOM node ที่มันผูกไว้ตรง ๆ — **ไม่มีขั้นตอนสร้าง tree ใหม่ ไม่มีขั้นตอนเทียบ tree เก่ากับใหม่ ไม่มี diffing เกิดขึ้นเลยแม้แต่ครั้งเดียว**

พูดให้เป็นรูปธรรมที่สุด: ถ้า component ของคุณมี `<p>{count}</p>` และ `<p>{other_signal}</p>` อยู่ใน component เดียวกัน เมื่อ `count` เปลี่ยนค่า **เฉพาะ text node ของ `<p>{count}</p>` เท่านั้น**ที่ถูกเขียนใหม่ — text node ของ `<p>{other_signal}</p>` จะไม่ถูกแตะต้องเลยแม้แต่การอ่าน เพราะ effect ของมันไม่ได้ subscribe กับ `count` ตั้งแต่แรก ต่างจาก Yew ที่ทั้ง component function (ซึ่งสร้าง VNode ของทั้งสอง `<p>`) จะถูกเรียกใหม่ทุกครั้ง แม้ diffing จะช่วยไม่ให้ DOM จริงถูกแก้ทั้งสอง node ก็ตาม

#### พิสูจน์ด้วยซอร์สโค้ดจริง: ไม่มี diffing แปลว่าอะไรในระดับ implementation

เพื่อไม่ให้ประโยคข้างบนเป็นแค่ "การโฆษณา" ลองไล่ดูซอร์สโค้ดจริงของ Leptos (crate ภายในชื่อ `tachys` ซึ่งเป็น rendering engine ที่ Leptos ใช้) ไฟล์ `tachys-0.2.19/src/view/primitives.rs` มี implementation ของ trait `Render` สำหรับ primitive type อย่าง `i32`/`String` (ชนิดข้อมูลที่คุณมักใส่ตรง ๆ ใน `{...}` ของ `view!`):

```rust
// จากซอร์สโค้ดจริงของ tachys-0.2.19 (ย่อให้อ่านง่ายขึ้น)
impl Render for i32 {
    type State = I32State; // เก็บ (DOM Text node, ค่าปัจจุบัน)

    fn build(self) -> Self::State {
        // เรียกครั้งเดียวตอน mount: สร้าง Text node จริงหนึ่งตัว
        let node = Rndr::create_text_node(&self.to_string());
        I32State(node, self)
    }

    fn rebuild(self, state: &mut Self::State) {
        // เรียกทุกครั้งที่ signal ที่ effect นี้ subscribe เปลี่ยนค่า
        let I32State(node, this) = state;
        if &self != this {
            Rndr::set_text(node, &self.to_string()); // เขียนทับ node เดิมตรง ๆ
            *this = self;
        }
    }
}
```

และ `Rndr::set_text` (ไฟล์ `tachys-0.2.19/src/renderer/dom.rs`) ทำสิ่งเดียวเท่านั้น:

```rust
// จากซอร์สโค้ดจริง
pub fn set_text(node: &Text, text: &str) {
    node.set_node_value(Some(text));
}
```

`node.set_node_value(...)` คือ DOM API มาตรฐาน (`Node.nodeValue`) ที่เขียนทับเนื้อหาของ text node **เดิม**โดยตรง ไม่มีการสร้าง node ใหม่ ไม่มีการเทียบ tree ใด ๆ — `build()` ถูกเรียกครั้งเดียวตอน mount เพื่อสร้าง node จริง ส่วน `rebuild()` ถูกเรียกซ้ำ ๆ โดย reactive effect ทุกครั้งที่ signal เปลี่ยน และมันแค่เขียนทับ node เดิม นี่คือหลักฐานระดับซอร์สโค้ดว่า "ไม่มี virtual DOM diffing" ไม่ใช่แค่คำพูดในเอกสารการตลาด

#### พิสูจน์ด้วยการทดสอบจริงในเบราว์เซอร์: DOM node เดิมถูกใช้ซ้ำจริง

ยังไม่พอ — เพื่อพิสูจน์ให้เห็นในระดับที่สังเกตได้จริงจากภายนอก (ไม่ใช่แค่อ่านซอร์สโค้ด) จึง build component ตัวอย่างเป็น WASM จริงด้วย `wasm-pack build --target web` เสิร์ฟด้วย `http-server` แล้วเปิดด้วย headless Chromium ผ่าน Playwright จากนั้นรันสคริปต์ที่:

1. จับ reference ของ DOM Text node object ที่อยู่ใน `<p id="count-display">` **ก่อน**คลิกปุ่ม increment
2. คลิกปุ่ม increment สามครั้ง
3. จับ reference ของ Text node object **หลัง**คลิก แล้วเทียบด้วย `===` (JavaScript reference equality — เทียบว่าเป็น object ตัวเดียวกันในหน่วยความจำหรือไม่ ไม่ใช่แค่เทียบค่า)

ผลลัพธ์จริงที่ได้ (คัดลอกจาก stdout ของสคริปต์ Playwright ที่รันจริง):

```
BEFORE count-display: Count: 0
AFTER 3 clicks count-display: Count: 3
AFTER doubled-display: Doubled: 6
DOM node identity check (fine-grained: same node object, only .data mutated): {"sameParentNode":true,"sameTextNode":true}
```

`sameTextNode: true` คือหลักฐานที่หนักแน่นที่สุด: หลังจากคลิกปุ่มสามครั้ง (เปลี่ยนค่า `count` จาก 0 เป็น 3) **DOM Text node object ที่ browser เก็บอยู่ในหน่วยความจำยังเป็นตัวเดิม** ไม่ได้ถูกทำลายแล้วสร้างใหม่ มีแค่เนื้อหาข้างในถูกเขียนทับด้วย `set_node_value` ตามที่เห็นในซอร์สโค้ดข้างบน — ถ้า Leptos ใช้โมเดล Virtual DOM แบบ Yew, การ re-render ในทางทฤษฎีอาจสร้าง VNode ใหม่แล้ว diff กลับมาเป็น text node เดิมได้เหมือนกัน (ผลลัพธ์ที่สังเกตจาก DOM อาจจะดูคล้ายกัน) — แต่ความต่างที่สำคัญคือ Leptos **ไม่มีขั้นตอนกลาง**ที่ต้องสร้าง representation ใหม่แล้วเทียบเลย มันรู้ตั้งแต่ compile time (จาก macro expansion) แล้วว่า "signal ตัวนี้ผูกกับ text node ตัวนี้โดยตรง" — งานที่ต้องทำตอน runtime จึงมีแค่ "เขียนทับ node เดิม" ไม่ใช่ "สร้าง tree ใหม่ + เทียบ + หา diff + apply diff"

#### ทำไมความแตกต่างนี้สำคัญ

ความต่างนี้ไม่ใช่แค่ทฤษฎีเชิงสถาปัตยกรรม มันมีผลจริงสองด้าน:

- **Performance ต่อการ update ครั้งหนึ่ง**: fine-grained reactivity ทำงานน้อยกว่าต่อการเปลี่ยนแปลงหนึ่งครั้งเสมอ (ไม่ต้องสร้าง tree ใหม่ ไม่ต้อง diff) แต่ต้องระวังว่านี่ไม่ได้แปลว่า Leptos "เร็วกว่า Yew เสมอ" ในทุกสถานการณ์ — ถ้าแอปมี state เปลี่ยนน้อยและ tree เล็ก ความต่างด้าน performance อาจวัดไม่ออกด้วยซ้ำ ความต่างจะเห็นชัดในแอปที่มี state เปลี่ยนบ่อยมาก ๆ (เช่น real-time dashboard) และ tree ใหญ่
- **Mental model ตอนเขียนโค้ด**: ใน Yew คุณคิดแบบ "component คือฟังก์ชันที่ map state ไปเป็น UI" (คล้าย React) ทุกครั้งที่ state เปลี่ยนคุณคิดว่า "ฟังก์ชันนี้จะถูกเรียกใหม่" ส่วนใน Leptos คุณต้องคิดแบบ "component คือโค้ด setup ที่รันครั้งเดียว แล้ว UI ที่เหลือขับเคลื่อนด้วย reactive graph ที่แยกออกจาก component lifecycle" — นี่คือเหตุผลที่ hook แบบ `use_effect` ของ Yew (ที่ผูกกับ "การ re-render ของ component") กับ `Effect::new`/closure ที่อ่าน signal ตรง ๆ ของ Leptos มี mental model ต่างกันพอสมควร แม้ syntax ของ `view!` จะดูคล้าย `html!` มากก็ตาม

#### ทำไม signal ถึง `move` เข้า closure ได้อย่างอิสระ: signal คือ "handle" ไม่ใช่ "ค่า"

สังเกตในตัวอย่างโค้ดของบทนี้ว่า signal ถูก `move` เข้า closure หลายจุดพร้อมกันได้อย่างอิสระ (เช่น `count` ถูกใช้ทั้งใน `view!` และใน closure ของ `Memo::new` พร้อมกัน) โดยไม่มี error เรื่อง ownership/borrow เกิดขึ้นเลย ทั้งที่ Part 6-7 สอนไว้ว่าปกติค่าที่ถูก `move` เข้า closure ตัวหนึ่งแล้วจะใช้ที่อื่นต่อไม่ได้ (ownership ย้ายไปแล้ว) เหตุผลอยู่ที่ตรวจสอบได้จากซอร์สโค้ดจริงของ `reactive_graph-0.2.15/src/signal/read.rs`:

```rust
// จากซอร์สโค้ดจริง — reactive_graph-0.2.15/src/signal/read.rs
pub struct ReadSignal<T, S = SyncStorage> {
    pub(crate) inner: ArenaItem<ArcReadSignal<T>, S>,
}

impl<T, S> Copy for ReadSignal<T, S> {}

impl<T, S> Clone for ReadSignal<T, S> {
    fn clone(&self) -> Self {
        *self
    }
}
```

`ReadSignal<T>` (และ `WriteSignal<T>`, `Memo<T>`, `RwSignal<T>` ด้วยหลักการเดียวกัน) **ไม่ได้เก็บค่า `T` ไว้ข้างในตัวเองโดยตรง** มันเก็บแค่ `ArenaItem<...>` ซึ่งเป็น handle ขนาดเล็ก (ภายในคือ index ชี้ไปยัง arena ที่เก็บค่าจริงอีกที คล้ายกับ pattern `slotmap`/generational-index ที่ Part 27-29 พูดถึงตอนอธิบายข้อจำกัดของ `Rc`/`RefCell`) — และที่สำคัญคือมัน `impl Copy` แบบไม่มีเงื่อนไขใด ๆ กับ `T` เลย (ไม่ต้องมี `where T: Copy`) เพราะตัว handle เองมีขนาดคงที่เล็ก ๆ ไม่ว่า `T` จะเป็น `i32` หรือ `Vec<Book>` ขนาดใหญ่ก็ตาม การ "ย้าย" signal เข้า closure จึงเป็นการ copy handle ตัวเล็ก ๆ นั้นเท่านั้น ไม่ใช่การย้ายข้อมูลจริงเลย — นี่คือเหตุผลเชิงเทคนิคที่ทำให้เขียน `move || count.get()` ได้ทุกที่โดยไม่ชนกับ borrow checker ของ Part 6-7 เลยแม้แต่ครั้งเดียวตลอดทั้งบทนี้

ข้อแลกเปลี่ยนของแนวทางนี้คือ signal ทุกตัวจะ "มีอายุ" ผูกกับ **reactive owner** ที่สร้างมัน (ปกติคือ component ที่ล้อมมันไว้) ถ้า component นั้นถูก unmount ไปแล้ว handle ที่เหลืออยู่จะกลาย เป็น "disposed" — เรียก `.get()` กับ signal ที่ owner ของมันถูก dispose ไปแล้วจะได้ค่า panic ในโหมด debug (ไม่ใช่ undefined behavior แบบ dangling pointer ในภาษาที่ไม่มี ownership system) กรณีนี้พบได้น้อยในแอปทั่วไปเพราะ Leptos จัดการ scope ของ owner ให้อัตโนมัติตาม component tree แต่ควรรู้ไว้เมื่อเขียนโค้ดที่ signal ถูกส่งออกไปไกลจาก component ที่สร้างมันมาก ๆ (เช่นเก็บไว้ใน global state ข้าม component)

### 89.2 ติดตั้งและ Setup: `cargo-leptos`, `trunk`, และ Hello World ด้วย `view!`

Leptos มีสองแนวทางหลักในการตั้งโปรเจกต์ ขึ้นกับว่าคุณต้องการแค่ CSR (client-side rendering ล้วน ๆ เหมือน Yew) หรือต้องการ full-stack (SSR + server function ซึ่งจะเห็นในหัวข้อ 89.6-89.8):

**แนวทางที่ 1 — CSR ล้วน ๆ ด้วย `wasm-pack` หรือ `trunk`**: เหมาะกับ SPA ที่ไม่ต้องการ SSR เลย ตั้งโปรเจกต์แบบ library ที่ compile เป็น WASM:

```toml
# Cargo.toml
[package]
name = "my_leptos_app"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
leptos = { version = "0.8", features = ["csr"] }
console_error_panic_hook = "0.1"
wasm-bindgen = "0.2"
```

จากนั้น build ด้วย `wasm-pack build --target web` (สร้าง `pkg/` ที่มีไฟล์ `.wasm` และ JS glue code) แล้วเขียน `index.html` ที่ import module นั้นเข้ามา หรือใช้ `trunk` (เครื่องมือที่นิยมกว่าในชุมชน Leptos/Yew เพราะจัดการ dev server + hot reload + asset bundling ให้ครบ) — วิธีตั้งค่า `trunk` เหมือนกับที่ Part 88 สอนไว้สำหรับ Yew ทุกประการ เพราะทั้งสอง framework ใช้ WASM target เดียวกันและ `trunk` ไม่สนใจว่าโค้ดข้างในเป็น Yew หรือ Leptos

ทดสอบจริงด้วยการติดตั้ง `trunk` (`cargo install trunk --locked`, ได้เวอร์ชัน `trunk 0.21.14` บนเครื่องที่ใช้เขียนบทนี้) แล้ววาง `Trunk.toml` ง่าย ๆ ไว้ที่ root ของโปรเจกต์เดียวกับ `Cargo.toml`:

```toml
# Trunk.toml
[build]
target = "index.html"
```

พร้อม `index.html` ที่**ไม่มี** `<link data-trunk rel="rust" />` เลยด้วยซ้ำ (ต่างจากที่หลายบทความเก่าบอกว่าต้องมี tag นี้เสมอ):

```html
<!DOCTYPE html>
<html lang="th">
  <head>
    <meta charset="utf-8" />
    <title>Leptos CSR Demo</title>
  </head>
  <body></body>
</html>
```

รัน `trunk build` จริง ผลลัพธ์ที่ได้ (คัดลอกจาก terminal จริง):

```
$ trunk build
2026-09-27T02:58:39.907926Z  INFO Starting trunk 0.21.14
2026-09-27T02:58:39.908269Z  INFO starting build
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.11s
2026-09-27T02:58:40.301652Z  INFO downloading wasm-bindgen version="0.2.129"
2026-09-27T02:58:40.995285Z  INFO installing wasm-bindgen
2026-09-27T02:58:41.638189Z  INFO applying new distribution
2026-09-27T02:58:41.638658Z  INFO success
```

`trunk` มองเห็น `Cargo.toml` ที่อยู่โฟลเดอร์เดียวกับ `index.html` แล้ว**เดาเอาเองโดยอัตโนมัติ**ว่านี่คือโปรเจกต์ Rust ที่ต้อง compile เป็น WASM (ไม่จำเป็นต้องมี `<link data-trunk rel="rust" />` เสมอไปถ้าโครงสร้างโฟลเดอร์ตรงตามธรรมเนียมนี้) ผลลัพธ์ที่ได้ในโฟลเดอร์ `dist/` คือ `index.html` ที่ `trunk` **แก้ไขให้เองอัตโนมัติ** โดยแทรก `<script type="module">` ที่ import ไฟล์ WASM/JS ที่ hash ชื่อไว้กันแคชเก่าเข้ามาให้ (ตัดมาบางส่วนจากไฟล์จริงที่ได้):

```html
<script type="module">
import init, * as bindings from '/leptos_csr-3e61016bf8ff84fd.js';
const wasm = await init({ module_or_path: '/leptos_csr-3e61016bf8ff84fd_bg.wasm' });
window.wasmBindings = bindings;
dispatchEvent(new CustomEvent("TrunkApplicationStarted", {detail: {wasm}}));
</script>
```

สังเกตว่าไฟล์ผลลัพธ์นี้**เหมือนกับที่ Part 88 อธิบายไว้สำหรับ Yew ทุกประการ** — เพราะขั้นตอน "compile Rust เป็น WASM แล้วสร้าง glue code JS ด้วย `wasm-bindgen`" เป็นขั้นตอนที่อยู่**นอก** Leptos/Yew โดยสิ้นเชิง มันคือ pipeline ของ Rust→WASM ทั่วไปที่ทั้งสอง framework ใช้ร่วมกัน ความต่างระหว่าง Leptos กับ Yew ทั้งหมดที่อธิบายในหัวข้อ 89.1 อยู่**ข้างใน** ไฟล์ `.wasm` นั้น ไม่ใช่ในขั้นตอน build/bundle นี้เลย

ระหว่างพัฒนา ใช้ `trunk serve` แทน `trunk build` เพื่อได้ dev server พร้อม hot-reload (rebuild อัตโนมัติเมื่อไฟล์ `.rs` เปลี่ยน แล้ว browser จะ auto-refresh ผ่าน WebSocket ที่ `trunk` ฝังสคริปต์ไว้ให้เอง) — เหมาะกับการพัฒนา component หลาย ๆ ตัวต่อเนื่อง ส่วน `wasm-pack build --target web` เหมาะกว่าตอนต้องการควบคุมทุกไฟล์ output เอง (เช่นตอนต้องเอา `.wasm`/`.js` ไปฝังในระบบ build ของ backend framework อื่นที่ไม่ใช่ `trunk`) — บทนี้ทดสอบทั้งสองวิธีจริง `wasm-pack` ใช้สำหรับตัวอย่าง signal/memo/component ในหัวข้อ 89.3-89.5 (เพื่อควบคุม `index.html` ที่ใช้กับ Playwright เอง) และ `trunk` ใช้พิสูจน์ workflow ที่ตรงกับที่ผู้เรียนจะใช้จริงในโปรเจกต์

#### ขนาดไฟล์จริงระหว่าง debug กับ release build

WASM bundle ที่ได้จาก `wasm-pack build --target web` แบบ debug (ค่า default ไม่ระบุ flag) มีขนาดใหญ่กว่าที่ควรส่งขึ้น production มาก เพราะไม่มีการ optimize ใด ๆ เลย ทดสอบวัดขนาดไฟล์จริงจากตัวอย่างในบทนี้ (รวม signal, memo, component, router เข้าด้วยกัน) เทียบ debug กับ `wasm-pack build --target web --release` (ซึ่งเรียก `wasm-opt` ต่อท้ายให้อัตโนมัติถ้าเครื่องมีติดตั้งไว้):

| Build | ขนาดไฟล์ `.wasm` จริง |
|---|---|
| `--dev` (debug, ค่า default) | 915,664 bytes (~894 KB) |
| `--release` (พร้อม `wasm-opt`) | 359,059 bytes (~350 KB) |

ขนาดลดลงมากกว่าครึ่งหนึ่งแค่จากการสลับ flag เดียว (`--release` เปิด `opt-level` ที่สูงขึ้นของ `rustc` เอง บวกกับ `wasm-opt` ที่มา post-process ไฟล์ `.wasm` อีกชั้นเพื่อตัด code ที่ไม่ได้ใช้และบีบอัดโครงสร้างให้กระชับขึ้น) ตัวเลขที่แน่นอนจะต่างไปตามจำนวน component/dependency ของแอปคุณเอง แต่หลักการเดียวกันนี้ใช้ได้เสมอ: **อย่าลืม `--release` ก่อน deploy จริง** เพราะ WASM bundle คือสิ่งที่ผู้ใช้ทุกคนต้องดาวน์โหลดก่อนหน้าจะ interactive ได้เลย (ตามที่อธิบายเรื่อง CSR ในหัวข้อ 89.7) ขนาดไฟล์ที่เล็กลงมีผลตรงต่อเวลาที่ผู้ใช้ต้องรอจริง ๆ

**แนวทางที่ 2 — Full-stack ด้วย `cargo-leptos`**: เมื่อคุณต้องการ SSR/server function (หัวข้อ 89.6 เป็นต้นไป) จำเป็นต้องมี**สอง build target ในโปรเจกต์เดียว** — เวอร์ชันที่ compile เป็น native binary (รันบน server จริง มี feature `ssr`) และเวอร์ชันที่ compile เป็น WASM (รันในเบราว์เซอร์เพื่อ hydrate, มี feature `hydrate`) `cargo-leptos` คือเครื่องมือ CLI เฉพาะของ Leptos ที่จัดการ build ทั้งสอง target นี้พร้อมกันให้ ติดตั้งด้วย:

```bash
cargo install cargo-leptos --locked
```

หลังติดตั้งจะได้คำสั่ง `cargo leptos new`, `cargo leptos build`, `cargo leptos serve` — บทนี้ลองติดตั้งจริงบนเครื่อง (`cargo install cargo-leptos --locked`) แต่ตัว crate นี้มี dependency สายที่ compile หนักมากและใช้เวลานาน (เช่น `openssl-sys`, `gix` สำหรับจัดการ git ของ template) ประกอบกับเครื่องทดสอบมี agent อื่นรันงาน build คู่ขนานอยู่พร้อมกันหลายตัว การติดตั้งจึงไม่เสร็จในเวลาที่เหมาะสม — จึงเลือกพิสูจน์เนื้อหาที่สำคัญที่สุดของบทนี้ (**server function + Axum integration**, หัวข้อ 89.6-89.8) ด้วยการสร้างโปรเจกต์ SSR แบบ manual แทน (ไม่ผ่าน `cargo leptos new` template) ซึ่งใช้แค่ `cargo build`/`cargo run` ธรรมดา ไม่ต้องพึ่ง `cargo-leptos` เลย — ข้อดีอีกอย่างคือทำให้เห็นภาพชัดว่า `leptos_axum` integration ทำงานอย่างไรโดยไม่มี "มายากล" จาก template ซ่อนอยู่ ส่วน `cargo leptos new`/`build`/`serve` ยังเป็นเครื่องมือมาตรฐานที่ใช้งานได้ตามปกติในโปรเจกต์จริง (มี hot-reload ให้ในตัว) ถ้าคุณติดตั้งสำเร็จ — รายละเอียด API ของ `leptos_axum` เต็ม ๆ อยู่ในหัวข้อ 89.8

#### Hello World ด้วย `view!` macro

ไม่ว่าจะเลือกแนวทางไหน หน้าตาโค้ด component พื้นฐานเหมือนกัน:

```rust
use leptos::prelude::*;
use leptos::mount::mount_to_body;

#[component]
fn App() -> impl IntoView {
    view! {
        <h1>"Hello, Leptos!"</h1>
        <p>"นี่คือ component แรกที่เขียนด้วย view! macro"</p>
    }
}

// เรียกครั้งเดียวตอนเริ่มโปรแกรม (สำหรับ CSR)
#[wasm_bindgen::prelude::wasm_bindgen(start)]
pub fn main() {
    console_error_panic_hook::set_once();
    mount_to_body(App);
}
```

โครงสร้างนี้คอมไพล์ผ่านและรันได้จริง (ตรวจสอบด้วย `cargo check --target wasm32-unknown-unknown` และ `wasm-pack build --target web` บนเครื่องจริง — ทั้งสองคำสั่งสำเร็จไม่มี error) สังเกต syntax ที่คล้าย Yew's `html!` มาก:

| | Yew (`html!`, Part 88) | Leptos (`view!`) |
|---|---|---|
| Tag element | `<h1>{ "Hello" }</h1>` | `<h1>"Hello"</h1>` |
| Text ธรรมดา | ต้องห่อด้วย `{ "..." }` | เขียน string literal ตรง ๆ ได้ |
| Attribute แบบ dynamic | `class={classes}` | `class=classes` |
| Event handler | `onclick={Callback::from(...)}` | `on:click=move |_| {...}` |

แม้ syntax หน้าตาคล้ายกันมาก (ทั้งคู่เป็น proc macro ที่แปลง JSX-like syntax เป็นโค้ด Rust ตอน compile time ตามหลักการ procedural macro ที่ Part 36 ปูพื้นไว้ — "แปลง token stream หนึ่งเป็น token stream อื่นตอน compile") แต่**สิ่งที่ macro ทั้งสองตัว generate ออกมาต่างกันโดยสิ้นเชิง**: `html!` ของ Yew generate โค้ดที่สร้าง `VNode` (โครงสร้างข้อมูล Rust ที่ต้องถูก diff ตอน re-render) ส่วน `view!` ของ Leptos generate โค้ดที่เรียก `Render::build()`/`rebuild()` ตามที่เห็นในหัวข้อ 89.1 — สร้าง DOM node จริงและผูก reactive effect เข้าไปตรง ๆ นี่คือตัวอย่างที่ดีว่า "syntax เหมือนกันได้ แต่ compilation strategy ต่างกันโดยสิ้นเชิง"

### 89.3 Signals: หน่วยพื้นฐานของ Reactive State

ใน Leptos ทุกอย่างที่เป็น "state ที่ต้องทำให้ UI อัปเดตตาม" ต้องเป็น **signal** ฟังก์ชันสร้าง signal พื้นฐานคือ `signal()`:

```rust
use leptos::prelude::*;

let (count, set_count) = signal(0);
```

`signal(initial_value)` คืนค่าเป็น tuple สอง element: `ReadSignal<T>` (สำหรับอ่านค่า) และ `WriteSignal<T>` (สำหรับเขียนค่า) — การแยก read/write ออกจากกันเป็น type คนละตัวเป็นการตัดสินใจเชิง type-system ที่ตั้งใจ: ฟังก์ชันที่รับ `ReadSignal<T>` เป็น parameter การันตีได้จาก signature เลยว่ามันจะไม่แก้ไขค่า (อ่านได้อย่างเดียว) ต่างจากการส่ง `&mut T` ธรรมดาที่ผู้เรียกต้องเชื่อ documentation เอาเอง

#### ทำไมไม่ใช่ `create_signal()`? — เรื่อง deprecation ที่ต้องรู้

ถ้าคุณเจอตัวอย่างเก่า (บทความ, StackOverflow, หรือ tutorial ที่เขียนไว้ตอน Leptos เวอร์ชัน 0.5/0.6) จะเจอ `create_signal()` แทน `signal()` เสมอ — เรื่องนี้สำคัญพอที่จะต้องอธิบายให้ชัด เพราะ Leptos เปลี่ยนชื่อฟังก์ชันกลุ่มนี้จริง ตรวจสอบจากซอร์สโค้ดจริงของ `reactive_graph-0.2.15/src/signal.rs` (เวอร์ชันที่ผูกมากับ leptos 0.8.21) พบว่า:

```rust
// จากซอร์สโค้ดจริง — reactive_graph-0.2.15/src/signal.rs
#[deprecated = "This function is being renamed to `signal()` to conform to \
                Rust idioms."]
pub fn create_signal<T: Send + Sync + 'static>(
    value: T,
) -> (ReadSignal<T>, WriteSignal<T>) {
    signal(value)
}
```

`create_signal` **ยังเรียกใช้ได้จริง** (มันแค่ forward ไปเรียก `signal()` ข้างใน) แต่ถูก mark ด้วย `#[deprecated]` แล้ว — ถ้าใช้จะเจอ compiler warning จริง (คัดลอกมาจาก `cargo check` จริงบนเครื่องที่เขียนบทนี้):

```
warning: use of deprecated function `leptos::prelude::create_signal`: This function is being renamed to `signal()` to conform to Rust idioms.
 --> src/lib.rs:4:31
  |
4 |     let (count, _set_count) = create_signal(0);
  |                               ^^^^^^^^^^^^^
  |
  = note: `#[warn(deprecated)]` on by default
```

เหตุผลเบื้องหลัง (ตามข้อความ deprecation) คือ "conform to Rust idioms" — สังเกตว่าใน Rust ธรรมดา ฟังก์ชันที่สร้างค่าของ type มักไม่มีคำว่า `create_` นำหน้า (เช่น `Vec::new()` ไม่ใช่ `create_vec()`, `String::from(...)` ไม่ใช่ `create_string(...)`) prefix `create_` เป็นธรรมเนียมที่หลุดมาจากตอน Leptos ยุคแรกที่ได้รับอิทธิพลจาก React hooks (`useState` มักถูกแปลเป็น `create_state` ในหลาย framework ที่ port มาจาก React) ทีม Leptos จึงค่อย ๆ เปลี่ยนชื่อฟังก์ชันกลุ่มนี้ทั้งหมดให้เข้ากับธรรมเนียม Rust มากขึ้น **บทนี้ใช้ `signal()` เป็นหลักตลอดทั้งบท** เพราะเป็น API ปัจจุบันที่ไม่มี warning

#### `.get()` และ `.set()`/`.update()`

```rust
use leptos::prelude::*;

fn main() {
    let (count, set_count) = signal(0);

    // อ่านค่า: .get() clone ค่าออกมา (ต้อง T: Clone)
    assert_eq!(count.get(), 0);

    // เขียนค่าใหม่ทั้งหมด
    set_count.set(5);
    assert_eq!(count.get(), 5);

    // แก้ไขค่าเดิม "in place" — มีประสิทธิภาพกว่าถ้า T มีขนาดใหญ่
    // (เช่น Vec ยาว ๆ) เพราะไม่ต้อง clone ค่าเก่าออกมาก่อนคำนวณค่าใหม่
    set_count.update(|n| *n += 1);
    assert_eq!(count.get(), 6);
}
```

ตัวอย่างข้างบนนี้ compile และรันได้จริงแม้ **ไม่มี DOM/เบราว์เซอร์เลย** (รันบน native target ธรรมดาผ่าน `cargo run`) — พิสูจน์ว่า reactive graph ของ Leptos เป็น library ที่แยกออกจาก DOM โดยสิ้นเชิง (`ReadSignal`/`WriteSignal` ทำงานเป็น in-memory reactive primitive ล้วน ๆ การผูกกับ DOM เกิดขึ้นเฉพาะตอนใช้ใน `view!` เท่านั้น) ผลลัพธ์จริงจากการรัน:

```
$ cargo run
   Compiling leptos_csr v0.1.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.10s
     Running `target/debug/leptos_csr`
count = 5
```

(โปรแกรมทดสอบข้างบนพิมพ์ `count = 5` หลังเรียก `set_count.set(5)` — ยืนยันว่า `.get()`/`.set()` ทำงานตรงตามที่คาด)

#### ตัวอย่างโดเมนจริง: จำนวนที่นั่งที่เหลือแบบ live-update

ต่อยอดโดเมนระบบจองตั๋วที่ใช้มาตั้งแต่ Part 62 (และจะใช้ต่อในหัวข้อ server function) ลองสร้าง component แสดง "จำนวนที่นั่งที่เหลือ" ที่อัปเดตทันทีเมื่อมีคนกดจอง:

```rust
use leptos::prelude::*;

#[component]
fn SeatCounter() -> impl IntoView {
    let total_seats = 100;
    let (booked, set_booked) = signal(42_i32);

    view! {
        <div>
            <p>"ที่นั่งที่เหลือ: " {move || total_seats - booked.get()} " / " {total_seats}</p>
            <button on:click=move |_| {
                set_booked.update(|b| {
                    if *b < total_seats { *b += 1 }
                });
            }>
                "จองที่นั่ง"
            </button>
        </div>
    }
}
```

สังเกตว่า `{move || total_seats - booked.get()}` คือ **closure** ไม่ใช่ค่าตรง ๆ — นี่สำคัญมาก: ถ้าเขียน `{total_seats - booked.get()}` (ไม่มี closure) โค้ดจะคำนวณค่าครั้งเดียวตอน component สร้าง `view!` (ตอน mount) แล้วค่านั้นจะไม่อัปเดตอีกเลยแม้ `booked` จะเปลี่ยน เพราะ `view!` macro ต้องเห็น **closure ที่อ่าน signal ข้างใน** เพื่อจะรู้ว่าต้องสร้าง reactive effect ผูกกับจุดนี้ (ตรงตามหลักการที่อธิบายในหัวข้อ 89.1 — "ทุกจุดใน `view!` ที่มีการอ่านค่า signal จะถูกสร้างเป็น effect") ถ้าไม่มี closure ห่อไว้ macro จะมองว่านี่คือค่าคงที่ตอน compile/mount ธรรมดา ไม่มีการ subscribe เกิดขึ้น — นี่คือกับดักที่พบบ่อยมากสำหรับคนใหม่ (ดูหัวข้อกับดักท้ายบท)

#### `RwSignal`: รวม read และ write ไว้ในตัวเดียว

หลายครั้งการแยก `ReadSignal`/`WriteSignal` เป็นสอง handle ก็เกินความจำเป็น (เช่นตอนส่ง prop เข้า component ลูกที่ต้องทั้งอ่านและเขียนค่าเดียวกัน — ต้องส่งสอง parameter แยกกัน) `RwSignal<T>` คือ signal ที่รวมทั้งอ่านและเขียนไว้ใน handle เดียว:

```rust
use leptos::prelude::*;

let count = RwSignal::new(0);

count.set(5);           // เขียนได้ตรง ๆ ผ่าน handle เดียวกัน
assert_eq!(count.get(), 5);
count.update(|n| *n += 1);
assert_eq!(count.get(), 6);
```

`RwSignal::new(...)` คือ API ปัจจุบัน (ฟังก์ชัน `create_rw_signal()` แบบเก่าถูก deprecate ด้วยเหตุผลเดียวกับ `create_signal`/`create_memo` ตามที่ตรวจสอบได้จากซอร์สโค้ดจริงในหัวข้อก่อนหน้า) เหมาะกับกรณีที่ต้อง pass ทั้ง read+write ไปให้ component ลูก (เช่นตัวอย่าง `#[island]` ในหัวข้อ 89.9 ที่รับ `RwSignal<usize>` เป็น prop) หรือต้องเก็บ signal ไว้ใน struct/context ที่ไม่สะดวกเก็บสอง field แยกกัน — ข้อควรระวังคือ `RwSignal` เปิดช่องให้ทั้งอ่านและเขียนจากทุกที่ที่มัน handle นี้ไปถึง ต่างจากการแยก `ReadSignal`/`WriteSignal` ที่ signature ของฟังก์ชันบอกชัดเจนว่าฟังก์ชันนั้น "แก้ไขได้" หรือ "อ่านได้อย่างเดียว" — ถ้าไม่มีเหตุผลชัดเจนที่ต้องรวม ควรแยก `signal()` ไว้เป็น tuple ตามปกติเพื่อให้ type signature สื่อความหมายชัดกว่า

#### อ่านค่าแบบ "ไม่ subscribe": `.get_untracked()`

บางครั้งคุณต้องการ**อ่านค่าปัจจุบัน**ของ signal โดยที่**ไม่ต้องการให้ closure ที่กำลังรันอยู่ subscribe กับมัน** — เช่นอ่านค่าไปแค่ log/debug ครั้งเดียว โดยไม่อยากให้ effect ที่ล้อมอยู่ต้องรันซ้ำทุกครั้งที่ signal ตัวนั้นเปลี่ยน (ทั้งที่ผลลัพธ์ของ effect ไม่ได้ขึ้นกับมันจริง ๆ) ใช้ `.get_untracked()` แทน `.get()`:

```rust
let (count, set_count) = signal(0);

Effect::new(move |_| {
    // อ่าน `count` แบบ subscribe ตามปกติ — effect นี้จะรันซ้ำทุกครั้งที่ count เปลี่ยน
    let current = count.get();

    // อ่าน `other_signal` แบบไม่ subscribe — ต่อให้ other_signal เปลี่ยนค่า
    // effect นี้จะไม่ถูกกระตุ้นให้รันใหม่เพราะจุดนี้
    // (สมมติว่ามี other_signal ประกาศไว้ก่อนหน้าแล้ว)
    println!("count = {current}");
});
```

หลักการนี้สำคัญเวลา debug ว่า "ทำไม effect/memo ตัวนี้รันบ่อยเกินไป" — สาเหตุที่พบบ่อยที่สุดคือมีการเรียก `.get()` (แบบ subscribe) ทิ้งไว้ในจุดที่ตั้งใจจะ "แค่อ่านเฉย ๆ" โดยไม่รู้ตัว การไล่หา `.get()` ที่ไม่ควร subscribe แล้วเปลี่ยนเป็น `.get_untracked()` คือวิธีแก้ที่ตรงจุดที่สุด

#### `batch()`: รวมหลาย signal update ให้ trigger effect แค่ครั้งเดียว

ปกติทุกครั้งที่เรียก setter ของ signal effect ที่เกี่ยวข้องจะถูกกระตุ้นให้รันทันที ถ้าคุณเขียน setter สองตัวติดกัน effect ที่ subscribe กับทั้งคู่อาจถูกกระตุ้นสองรอบ (รอบละหนึ่ง signal) ทั้งที่ในทางตรรกะคุณต้องการให้มันรันแค่ครั้งเดียวหลังจากทั้งสองค่าอัปเดตพร้อมกัน ฟังก์ชัน `batch()` (จากซอร์สโค้ดจริงของ `reactive_graph-0.2.15/src/effect/immediate.rs`) แก้ปัญหานี้ตรง ๆ — คำอธิบายจากซอร์สโค้ดจริงบอกไว้ชัดว่า:

> "Defers any effects from running until the end of the function... this is rarely needed, but it is useful for example when multiple signals need to be updated atomically"

```rust
use leptos::prelude::*;

let (first_name, set_first_name) = signal("Ada".to_string());
let (last_name, set_last_name) = signal("Lovelace".to_string());

// ไม่มี batch: effect ที่อ่านทั้ง first_name และ last_name จะรันสองรอบ
// set_first_name.set("Grace".to_string());
// set_last_name.set("Hopper".to_string());

// มี batch: effect จะรันแค่รอบเดียวหลัง closure จบ
batch(|| {
    set_first_name.set("Grace".to_string());
    set_last_name.set("Hopper".to_string());
});
```

ตามที่เอกสารบอกไว้ตรง ๆ ว่า "rarely needed" — ในโค้ดทั่วไปไม่ต้องใช้ `batch()` เลยก็ได้ เพราะ Leptos จัดการเรื่อง batching ให้อัตโนมัติในหลายกรณีอยู่แล้ว (เช่น event handler หนึ่งตัวที่เรียก setter หลายตัวข้างในมักถูก batch ให้โดย reactive system เองอยู่แล้วในหลาย ๆ สถานการณ์) แต่ควรรู้จักไว้สำหรับกรณีที่ต้องควบคุมจังหวะการอัปเดตให้แม่นยำจริง ๆ เช่นตอน sync ข้อมูลจากหลาย field ของฟอร์มพร้อมกันในทีเดียว

### 89.4 Derived Signal และ `Memo`: คำนวณซ้ำเฉพาะเมื่อจำเป็นจริง ๆ

**Derived signal** คือรูปแบบที่ง่ายที่สุดของ "ค่าที่คำนวณจาก signal อื่น" — แค่เขียน closure ธรรมดา:

```rust
let (count, set_count) = signal(0);
let doubled = move || count.get() * 2; // นี่คือ derived signal — เป็น Fn() -> T ธรรมดา
```

`doubled` ในตัวอย่างนี้**ไม่ใช่ signal จริง ๆ** มันเป็นแค่ closure ที่คำนวณค่าใหม่ทุกครั้งที่ถูกเรียก — ถ้ามีหลายจุดใน `view!` เรียก `doubled()` มันจะคำนวณซ้ำหลายครั้งโดยไม่มี caching ใด ๆ สำหรับการคำนวณเบา ๆ (เช่น `* 2`) แบบนี้ไม่มีปัญหา แต่ถ้าการคำนวณหนัก (เช่น filter/sort list ขนาดใหญ่) การคำนวณซ้ำโดยไม่จำเป็นจะเป็นปัญหาด้าน performance

**`Memo`** แก้ปัญหานี้ตรง ๆ: มันคำนวณค่าแล้ว **cache ไว้** และจะคำนวณใหม่ก็ต่อเมื่อ signal ที่มันอ่านจริง ๆ เปลี่ยนค่าเท่านั้น (ไม่ใช่ทุกครั้งที่มีใครมาอ่านมัน) API ปัจจุบันคือ `Memo::new()`:

```rust
use leptos::prelude::*;

let (count, set_count) = signal(0);
let doubled = Memo::new(move |_| count.get() * 2);
```

เช่นเดียวกับ `create_signal`, ฟังก์ชัน `create_memo()` แบบเก่าก็ถูก deprecate แล้วเหมือนกัน ตรวจสอบจากซอร์สโค้ดจริงของ `reactive_graph-0.2.15/src/computed.rs`:

```rust
// จากซอร์สโค้ดจริง — reactive_graph-0.2.15/src/computed.rs
#[deprecated = "This function is being removed to conform to Rust idioms. \
                Please use `Memo::new()` instead."]
pub fn create_memo<T>(
    fun: impl Fn(Option<&T>) -> T + Send + Sync + 'static,
) -> Memo<T>
where
    T: PartialEq + Send + Sync + 'static,
{
    Memo::new(fun)
}
```

สังเกตแบบเดียวกับ `create_signal`: มันแค่ forward ไปเรียก `Memo::new()` แล้ว mark deprecated การเปลี่ยนแปลงนี้สอดคล้องกับทิศทางเดียวกันของทีม Leptos — ย้ายจาก "ฟังก์ชัน `create_xxx()`" ไปเป็น "associated function `Xxx::new()`" ให้ตรงกับธรรมเนียมของ Rust ทั่วไปมากขึ้น (`create_rw_signal()` ก็ถูก deprecate ไปเป็น `RwSignal::new()` ด้วยเหตุผลเดียวกัน — ถ้าคุณเห็น `create_` นำหน้าฟังก์ชันสร้าง reactive primitive ใน Leptos 0.8 ให้สงสัยว่ามันถูก deprecate ไปแล้ว)

#### พิสูจน์ว่า Memo "เงียบ" จริงเมื่อ dependency ไม่เกี่ยวข้องเปลี่ยนค่า

นี่คือจุดที่ fine-grained reactivity ให้ประโยชน์ที่จับต้องได้จริง ไม่ใช่แค่ทฤษฎี ลองเขียน component ที่มี `Memo` พร้อม log ทุกครั้งที่มันคำนวณใหม่ และมี signal อีกตัวที่ **ไม่เกี่ยวข้องกับ Memo เลย**:

```rust
use leptos::prelude::*;
use leptos::mount::mount_to_body;

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    let doubled = Memo::new(move |_| {
        leptos::logging::log!("[memo] recomputing doubled, count = {}", count.get());
        count.get() * 2
    });

    let (unrelated, set_unrelated) = signal(0);

    view! {
        <div>
            <p id="count-display">"Count: " {count}</p>
            <p id="doubled-display">"Doubled: " {doubled}</p>
            <p id="unrelated-display">"Unrelated: " {unrelated}</p>
            <button id="inc-btn" on:click=move |_| set_count.update(|n| *n += 1)>
                "increment"
            </button>
            <button id="unrelated-btn" on:click=move |_| set_unrelated.update(|n| *n += 1)>
                "bump unrelated"
            </button>
        </div>
    }
}

fn main() {
    mount_to_body(App);
}
```

build เป็น WASM จริงด้วย `wasm-pack build --target web`, เสิร์ฟด้วย `http-server`, เปิดด้วย headless Chromium ผ่าน Playwright แล้วดักจับ `console.log` ทุกบรรทัดที่เกิดขึ้นจริง ผลลัพธ์ที่ได้ (คัดลอกจาก stdout จริง):

```
--- console logs so far (memo runs) ---
  [console] [memo] recomputing doubled, count = 0
  [console] [memo] recomputing doubled, count = 1
  [console] [memo] recomputing doubled, count = 2
  [console] [memo] recomputing doubled, count = 3
AFTER unrelated clicks, unrelated-display: Unrelated: 2
New console logs emitted after clicking UNRELATED button (should be 0 memo recompute logs):
New console logs after clicking INCREMENT again (memo SHOULD recompute):
  [console] [memo] recomputing doubled, count = 4
```

อ่านผลลัพธ์นี้อย่างละเอียด: หลังคลิกปุ่ม `increment` สามครั้ง (count 0→1→2→3) log `[memo] recomputing doubled` ปรากฏสี่ครั้ง (นับรวมครั้งแรกตอน mount ที่ count ยังเป็น 0) ตรงตามที่คาด — แต่หลังจากนั้นเมื่อคลิกปุ่ม `unrelated-btn` **สองครั้ง** (ซึ่งเปลี่ยนค่า `unrelated` จาก 0 เป็น 2 ตามที่ `unrelated-display` แสดง) **ไม่มี log ใหม่เกิดขึ้นเลยแม้แต่บรรทัดเดียว** — พิสูจน์ตรง ๆ ว่า `Memo` ไม่ได้คำนวณใหม่ เพราะมันไม่ได้ subscribe กับ `unrelated` เลยตั้งแต่แรก (closure ข้างในไม่ได้เรียก `unrelated.get()`) จากนั้นเมื่อคลิก `increment` อีกครั้ง (count 3→4) log ก็กลับมาปรากฏทันที (`count = 4`) — ยืนยันว่า dependency tracking ของ Leptos **ตรงและแม่นยำ** ไม่ใช่แค่ "คำนวณใหม่ทุกครั้งที่ signal ตัวไหนก็ตามในแอปเปลี่ยน" (ซึ่งจะทำให้ Memo ไม่มีประโยชน์เลย)

นี่คือความต่างเชิงปฏิบัติจาก derived signal ธรรมดา (`move || count.get() * 2`) ที่ไม่มี caching เลย — ถ้าการคำนวณข้างใน `Memo` หนักมาก (เช่น query/filter บน list เป็นพัน element) การรู้ว่ามันจะไม่ทำงานซ้ำโดยไม่จำเป็นคือสิ่งที่ทำให้แอปไม่กระตุก และหลักฐานข้างบนพิสูจน์ว่านี่ไม่ใช่แค่ optimization ตามทฤษฎี — สังเกตได้จริงจาก log

#### `Memo` เทียบกับ `Effect::new`: "คำนวณค่า" กับ "ทำ side effect"

ง่ายที่จะสับสนระหว่าง `Memo` กับ `Effect::new` เพราะทั้งคู่ "รันซ้ำเมื่อ dependency เปลี่ยน" เหมือนกัน — ความต่างที่สำคัญคือ**เจตนา**การใช้งาน:

- **`Memo::new(...)`** ใช้เมื่อต้องการ **คำนวณค่าใหม่** จาก signal อื่น แล้วเก็บค่านั้นไว้ให้ที่อื่นอ่านต่อ (เหมือนตัวอย่าง `doubled` ในหัวข้อนี้) — มันคืนค่าที่เป็น signal อีกตัวหนึ่ง (`Memo<T>` implement trait อ่านค่าได้เหมือน `ReadSignal<T>`)
- **`Effect::new(...)`** ใช้เมื่อต้องการ **ทำอะไรบางอย่างที่ไม่ใช่การคำนวณค่าคืน** เช่น log, เขียนลง `localStorage`, sync ค่าไปยัง signal อื่นที่ไม่ได้เกี่ยวกันโดยตรง, หรือเรียก DOM API บางอย่างที่ `view!` ไม่ได้ครอบให้ — มันไม่คืนค่าอะไรที่มีประโยชน์ (คืน `()`)

```rust
use leptos::prelude::*;

let (celsius, set_celsius) = signal(0.0_f64);

// Memo: "ค่านี้คืออะไร" — คำนวณแล้วเก็บไว้ให้อ่านต่อ
let fahrenheit = Memo::new(move |_| celsius.get() * 9.0 / 5.0 + 32.0);

// Effect: "ต้องทำอะไรตามหลัง" — ไม่ได้สร้างค่าใหม่ให้ใครอ่าน แค่ react ต่อการเปลี่ยนแปลง
Effect::new(move |_| {
    leptos::logging::log!("อุณหภูมิเปลี่ยนเป็น {:.1}°C", celsius.get());
    // ตัวอย่างจริง: เขียนค่าไว้ใน localStorage ทุกครั้งที่ผู้ใช้เปลี่ยนอุณหภูมิ
});
```

กฎง่าย ๆ ที่ใช้แยกสองตัวนี้ได้เกือบทุกกรณี: **ถ้าโค้ดของคุณ `return` ค่าที่มีความหมายและมีที่อื่นเอาไปอ่านต่อ ให้ใช้ `Memo`** — **ถ้าโค้ดของคุณทำ side effect (I/O, log, เรียก API ภายนอก) และไม่มีใครสนใจ return value ให้ใช้ `Effect::new`** การใช้ `Memo` ผิดที่ (เอาไปทำ side effect ข้างในโดยไม่สนใจค่าที่คืน) จะทำงานได้จริงเพราะ Rust ไม่ได้บังคับ แต่เป็นการใช้ผิดเจตนาของ API และอ่านโค้ดยาก — ทีม Leptos ตั้งใจแยกสองอันนี้ให้สื่อความหมายที่ต่างกันตั้งแต่ชื่อ

### 89.5 Components และ Props: สร้าง `BookCard`

`#[component]` attribute macro แปลงฟังก์ชัน Rust ธรรมดาให้เป็น Leptos component — parameter ของฟังก์ชันกลายเป็น props ของ component นั้น เทียบกับ Yew (Part 88) ที่ต้องประกาศ struct แยกพร้อม `#[derive(Properties, PartialEq)]` แล้วรับ `props: BookCardProps` เป็น parameter เดียว Leptos ให้เขียน parameter ตรง ๆ แบบฟังก์ชันทั่วไป:

```rust
use leptos::prelude::*;

#[component]
fn BookCard(
    title: String,
    author: String,
    #[prop(default = false)] is_borrowed: bool,
) -> impl IntoView {
    view! {
        <div class="book-card">
            <h3>{title}</h3>
            <p>"โดย " {author}</p>
            <p>{if is_borrowed { "ถูกยืมอยู่" } else { "พร้อมให้ยืม" }}</p>
        </div>
    }
}
```

`#[prop(default = false)]` คือ attribute เฉพาะของ Leptos ที่บอกว่า prop นี้ไม่บังคับต้องส่ง (ถ้าไม่ส่งจะใช้ค่า `false`) — ความสามารถนี้ macro ทำให้โดยอัตโนมัติ (generate struct props ภายในให้เอง คุณไม่เห็นมันในโค้ด) ต่างจาก Yew ที่ optional prop ต้องประกาศเป็น `Option<T>` ตรง ๆ ใน struct props ที่คุณเขียนเอง

ใช้งาน component นี้จาก component แม่:

```rust
#[component]
fn App() -> impl IntoView {
    view! {
        <div>
            <BookCard
                title="The Rust Programming Language".to_string()
                author="Steve Klabnik".to_string()
                is_borrowed=true
            />
            <BookCard
                title="Programming Rust".to_string()
                author="Jim Blandy".to_string()
            />
        </div>
    }
}
```

ทดสอบ build เป็น WASM จริงและเปิดในเบราว์เซอร์จริง (ผ่าน Playwright) ผลลัพธ์ HTML ที่ browser render ออกมาจริง (อ่านจาก `document.body.innerHTML` จริง):

```html
<div class="book-card"><h3>The Rust Programming Language</h3><p>โดย Steve Klabnik</p><p>ถูกยืมอยู่</p></div>
<div class="book-card"><h3>Programming Rust</h3><p>โดย Jim Blandy</p><p>พร้อมให้ยืม</p></div>
```

สังเกตว่า `BookCard` ตัวที่สองไม่ได้ส่ง `is_borrowed` เลย แต่ render ออกมาเป็น "พร้อมให้ยืม" ตรงตามค่า default `false` ที่ประกาศไว้ — พิสูจน์ว่า `#[prop(default = ...)]` ทำงานถูกต้องจริง

ข้อสังเกตสำคัญ: `BookCard` ในตัวอย่างนี้รับ `title: String` แบบ owned value ธรรมดา ไม่ใช่ signal — เหมาะสำหรับข้อมูลที่ "ตั้งค่าครั้งเดียวตอนสร้าง component แล้วไม่เปลี่ยนอีก" ถ้าต้องการให้ `BookCard` reactive ตามข้อมูลที่เปลี่ยนได้ (เช่นสถานะการยืมที่อัปเดตแบบ real-time) ต้องเปลี่ยน prop type เป็น `ReadSignal<bool>` หรือ `Signal<bool>` แทน แล้วเรียก `.get()` ข้างในแทนการใช้ค่าตรง ๆ — หลักการเดียวกับหัวข้อ 89.3 ที่ต้องมี signal (ไม่ใช่ค่าธรรมดา) ถึงจะเกิด reactive effect ได้

#### เทียบเคียงกับ `BookCard` ของ Yew (Part 88) แบบตรงจุด

Part 88 สร้าง `BookCard` จาก struct `Book` (field `id`, `title`, `author`, `available`) ผ่าน `#[derive(Properties, PartialEq)]` + `#[function_component(BookCard)]` — ลองเขียน component เดียวกันด้วย struct `Book` ชุดเดียวกันเป๊ะใน Leptos เพื่อเทียบแบบไม่มีตัวแปรกวนสายตา (ทดสอบ compile และ render จริงแล้ว):

```rust
use leptos::prelude::*;

// struct ข้อมูลหนังสือ — โครงสร้างเดียวกับ Book ใน Part 88 (Yew) เป๊ะ ๆ
// สังเกตว่า Leptos ไม่ต้อง derive PartialEq เพื่อการ "ข้าม re-render" เหมือน Yew เลย
// (เพราะ fine-grained reactivity ในหัวข้อ 89.1 ไม่มีขั้นตอน re-render/diff ให้ข้ามตั้งแต่ต้น)
#[derive(Clone, Debug)]
pub struct Book {
    pub id: u32,
    pub title: String,
    pub author: String,
    pub available: bool,
}

#[component]
fn BookCard(book: Book) -> impl IntoView {
    let status_text = if book.available { "ว่าง" } else { "ถูกยืมแล้ว" };
    let status_class = if book.available { "status-available" } else { "status-borrowed" };

    view! {
        <div class="book-card">
            <h3>{book.title}</h3>
            <p>{format!("ผู้เขียน: {}", book.author)}</p>
            <span class=status_class>{status_text}</span>
        </div>
    }
}
```

ผลลัพธ์ HTML ที่ render จริง (จาก `sample_book` ตัวเดียวกับใน Part 88):

```html
<div class="book-card"><h3>The Rust Programming Language</h3><p>ผู้เขียน: Steve Klabnik และ Carol Nichols</p><span class="status-available">ว่าง</span></div>
```

จุดต่างที่เห็นได้ทันทีจากโค้ดที่หน้าตาใกล้เคียงกันมากที่สุดในทั้งบท: (1) Yew ต้อง `#[derive(Properties, PartialEq)]` บน struct props แยก (`BookCardProps`) ส่วน Leptos รับ `book: Book` เป็น parameter ตรง ๆ ไม่ต้องมี struct props แยกให้ macro generate ให้เอง (2) Yew ต้อง derive `PartialEq` บน `Book` เพราะ Yew ใช้มันเทียบ props เก่า/ใหม่เพื่อตัดสินใจว่าจะ "ข้าม" การ re-render component ลูกหรือไม่ (เป็น optimization ที่จำเป็นเพราะ Yew re-render ทั้ง component function ทุกครั้งที่ parent เปลี่ยน) — Leptos ไม่ต้องมี `PartialEq` เลยเพราะไม่มีแนวคิด "re-render component แล้วเทียบ" อยู่ตั้งแต่ต้น (หัวข้อ 89.1) (3) `props.book` (ต้อง `&props.book` เพราะ Yew รับ props เป็น reference) เทียบกับ `book` ตรง ๆ ใน Leptos (รับเป็น owned value) — ความต่างเล็ก ๆ นี้สะท้อนภาพใหญ่เดียวกันตลอดทั้งบท: Yew ยืมแนวคิดมาจาก React ที่ "component รับ props แล้ว render" ทุกครั้งที่มีการเปลี่ยนแปลง ส่วน Leptos treat การสร้าง component เป็น setup ที่เกิดขึ้นครั้งเดียว

#### Prop attribute ที่ใช้บ่อย: `#[prop(optional)]`, `#[prop(into)]`, และ `children`

นอกจาก `#[prop(default = ...)]` ที่เห็นข้างบน Leptos ยังมี attribute อื่นที่ใช้บ่อยเวลาออกแบบ props ให้ยืดหยุ่น ลองดูตัวอย่าง `BookCard` เวอร์ชันที่ใช้ครบทุกแบบ (ทดสอบ compile และ render จริงในเบราว์เซอร์แล้ว):

```rust
use leptos::prelude::*;

#[component]
fn BookCard(
    title: String,
    // Option<T> + #[prop(optional)]: ถ้าไม่ส่งจะเป็น None โดยไม่ต้องกำหนด default เอง
    #[prop(optional)] subtitle: Option<String>,
    // #[prop(into)]: ผู้เรียกส่งอะไรมาก็ได้ที่ Into<Signal<i32>> — ทั้ง ReadSignal<i32>
    // เดิม หรือค่า i32 literal ตรง ๆ (เพราะ Signal<T> implement From<T> ให้)
    #[prop(into)] rating: Signal<i32>,
    // children: Children คือ "ช่องรับ markup ลูก" ระหว่างเปิด-ปิด tag ของ component
    // (เทียบได้กับ props.children ของ Yew ใน Part 88)
    children: Children,
) -> impl IntoView {
    view! {
        <div class="book-card">
            <h3>{title}</h3>
            {subtitle.map(|s| view! { <p class="subtitle">{s}</p> })}
            <p>"Rating: " {move || rating.get()}</p>
            <div class="footer">{children()}</div>
        </div>
    }
}

#[component]
fn App() -> impl IntoView {
    let (stars, _set_stars) = signal(4);
    view! {
        // ส่ง signal ตรง ๆ ให้ rating — reactive ได้ถ้า stars เปลี่ยนค่าทีหลัง
        <BookCard title="The Rust Programming Language".to_string() rating=stars>
            <em>"แนะนำสำหรับผู้เริ่มต้น"</em>
        </BookCard>
        // ส่ง literal ธรรมดาให้ rating — #[prop(into)] แปลงให้เป็น Signal คงที่ให้เอง
        <BookCard
            title="Programming Rust".to_string()
            subtitle="ฉบับที่ 2".to_string()
            rating=5
        >
            <em>"เหมาะสำหรับผู้มีพื้นฐานแล้ว"</em>
        </BookCard>
    }
}
```

build เป็น WASM จริงแล้วอ่าน `document.body.innerHTML` จากเบราว์เซอร์จริง ได้ผลลัพธ์ตรงตามที่ตั้งใจทุกจุด:

```html
<div class="book-card"><h3>The Rust Programming Language</h3><!----><p>Rating: 4</p><div class="footer"><em>แนะนำสำหรับผู้เริ่มต้น</em></div></div>
<div class="book-card"><h3>Programming Rust</h3><p class="subtitle">ฉบับที่ 2</p><p>Rating: 5</p><div class="footer"><em>เหมาะสำหรับผู้มีพื้นฐานแล้ว</em></div></div>
```

สังเกตสามจุดจากผลลัพธ์จริงนี้: (1) `BookCard` ตัวแรกไม่ได้ส่ง `subtitle` — Leptos render เป็น comment node `<!---->` แทนตำแหน่งที่ `None` (นี่คือกลไกเดียวกับที่ใช้กับ conditional rendering ทั่วไปใน Leptos — comment node ทำหน้าที่เป็น "ตำแหน่งยึด" ในโครงสร้าง DOM เผื่อค่ากลายเป็น `Some` ทีหลัง โดยไม่กระทบ node ข้างเคียง) (2) `rating=stars` (ส่ง signal) กับ `rating=5` (ส่ง literal) ทำงานได้ทั้งคู่เพราะ `#[prop(into)]` เรียก `.into()` ให้อัตโนมัติ — `ReadSignal<i32>` และ `i32` ทั้งคู่ implement `Into<Signal<i32>>` (3) `children()` render เนื้อหาระหว่าง tag เปิด-ปิดของ `<BookCard>...</BookCard>` เข้าไปในตำแหน่ง `<div class="footer">` ตรงตามที่กำหนดไว้ในโค้ด — พิสูจน์ว่า pattern การส่ง markup ลูกผ่าน component ทำงานได้เหมือนกับที่ Yew ทำผ่าน `props.children` ใน Part 88 แม้ syntax รับ parameter จะต่างกัน (Leptos รับเป็น parameter `children: Children` ตรง ๆ ไม่ต้องพึ่ง struct props ที่มี field `children`)

#### แชร์ state ข้าม component โดยไม่ต้อง prop-drilling: `provide_context`/`use_context`

ส่งพารามิเตอร์ผ่าน props ตรง ๆ ใช้ได้ดีตอน component ไม่ลึกมาก แต่ถ้า state ต้องแชร์กันระหว่าง component ที่อยู่ไกลกันในโครงสร้างต้นไม้ (เช่น "จำนวนสินค้าในตะกร้า" ที่ปุ่ม "เพิ่มลงตะกร้า" กับ badge แสดงจำนวนอยู่ห่างกันคนละส่วนของหน้า) การส่งผ่าน props ทุกชั้นจะกลายเป็น **prop-drilling** ที่น่าเบื่อและแก้ยากเมื่อโครงสร้างเปลี่ยน Leptos มี `provide_context`/`use_context` แก้ปัญหานี้ — เคยเห็นมันมาแล้วในหัวข้อ 89.6 ตอนฉีด `PgPool` ให้ server function แต่จริง ๆ มันใช้งานทั่วไปได้กับ signal ฝั่ง client เช่นกัน (นี่คือกลไกเดียวกับ `ContextProvider` ของ Yew ใน Part 88 — ทั้งสอง framework มีแนวคิด "ค่าที่มองเห็นได้จาก component ลูกทุกตัวโดยไม่ต้องผ่าน props" เหมือนกัน):

```rust
use leptos::prelude::*;

#[derive(Clone, Copy)]
struct CartState {
    count: RwSignal<i32>,
}

#[component]
fn AddToCartButton() -> impl IntoView {
    // use_context หา CartState จาก component บรรพบุรุษที่ provide_context ไว้
    let cart = use_context::<CartState>().expect("CartState ต้องถูก provide ไว้ก่อน");
    view! {
        <button on:click=move |_| cart.count.update(|c| *c += 1)>
            "เพิ่มลงตะกร้า"
        </button>
    }
}

#[component]
fn CartBadge() -> impl IntoView {
    let cart = use_context::<CartState>().expect("CartState ต้องถูก provide ไว้ก่อน");
    view! { <span>"ตะกร้า: " {move || cart.count.get()}</span> }
}

#[component]
fn App() -> impl IntoView {
    let cart = CartState { count: RwSignal::new(0) };
    provide_context(cart); // component ลูกทุกตัว (ไม่ว่าจะอยู่ลึกแค่ไหน) เรียก use_context เจอ

    view! {
        <div>
            <AddToCartButton />
            <CartBadge />
        </div>
    }
}
```

`AddToCartButton` และ `CartBadge` เป็น component **พี่น้องกัน** (sibling) ไม่มีความสัมพันธ์ parent-child โดยตรง และไม่มีการส่ง prop ระหว่างกันเลย — ทดสอบจริงในเบราว์เซอร์ (คลิกปุ่มสามครั้งแล้วอ่านค่า badge) ได้ผลลัพธ์ตรงตามที่ตั้งใจ:

```
before: ตะกร้า: 0
after 3 clicks: ตะกร้า: 3
```

`use_context::<T>()` หา value จาก **component บรรพบุรุษที่ใกล้ที่สุด**ที่เคยเรียก `provide_context` ด้วย type `T` เดียวกัน (ค้นหาตาม component tree ไม่ใช่ตาม lexical scope ของโค้ด) ถ้าไม่มีบรรพบุรุษไหน provide ไว้เลยจะได้ `None` กลับมา (ในตัวอย่างนี้ `.expect(...)` จะ panic ทันทีถ้าลืม `provide_context` — เป็นความตั้งใจให้เห็น bug นี้เร็วที่สุดตอน dev ไม่ใช่เงียบ ๆ แล้ว UI พังแบบเข้าใจยาก) จุดสำคัญคือ `CartState` ในตัวอย่างนี้ห่อ `RwSignal` ไว้ข้างใน (ไม่ใช่ `i32` ตรง ๆ) เพราะ `provide_context` ส่งค่าไปแบบ **snapshot ครั้งเดียวตอนเรียก** — ถ้าใส่ `i32` ธรรมดาเข้าไป component ลูกจะได้ค่าคงที่ ณ ตอนนั้นไปตลอด ไม่มีทาง "เห็น" การเปลี่ยนแปลงในอนาคตเลย ต้องห่อด้วย signal (หรือ struct ที่มี signal ข้างใน) เสมอถ้าต้องการให้ context เป็น reactive ตามหลักการเดียวกับหัวข้อ 89.3 ที่ต้องมี signal ถึงจะเกิด reactive effect ได้

### 89.6 Server Functions: จุดขายหลักของ Leptos

ถ้า fine-grained reactivity คือความต่างเชิงสถาปัตยกรรมของ Leptos ฝั่ง client, **server function** คือความต่างเชิงสถาปัตยกรรมที่สำคัญกว่าในภาพรวม — และคือเหตุผลหลักที่ชื่อบทนี้มีคำว่า "Full-stack Rust"

Yew (Part 88) ไม่มีแนวคิดนี้เลย ถ้าคุณอยากให้ Yew component คุยกับ database คุณต้อง: (1) เขียน REST API endpoint แยกด้วย Axum เอง (Part 62-70), (2) เขียนโค้ด fetch ฝั่ง client เอง (ผ่าน `gloo-net` หรือ `reqwasm`), (3) เขียน struct สำหรับ (de)serialize request/response เอง, (4) เขียน error handling คู่ขนานสองฝั่ง (ฝั่ง server ตอบ error code, ฝั่ง client parse error code) — ทั้งหมดนี้คือ "งานเชื่อมต่อ" ที่คุณต้องทำเองทุกจุด

#### ปริมาณงานที่ต้องเขียนถ้าไม่มี server function

เพื่อให้เห็นภาพจับต้องได้ ลองเขียนโค้ดที่ทำสิ่งเดียวกับ `get_event` (query event หนึ่งตัวจากฐานข้อมูล) แบบที่ Yew (หรือ Leptos ถ้าไม่ใช้ `#[server]`) ต้องทำ — ทดสอบว่า compile ผ่านจริงทั้งสองฝั่ง:

**ฝั่ง client** (ต้องเขียนเอง เพราะไม่มี macro ช่วย):

```rust
use serde::Deserialize;

#[derive(Debug, Clone, Deserialize)]
struct EventSeats {
    id: i64,
    name: String,
    total_seats: i32,
    booked_seats: i32,
}

// ต้องเขียน struct รับผลลัพธ์เอง (ซ้ำกับ struct ฝั่ง server แทบทุกฟิลด์)
// ต้องเขียนโค้ด fetch เอง ทั้ง URL, method, error handling
async fn fetch_event_manually(event_id: i64) -> Result<EventSeats, String> {
    let url = format!("/api/manual/get_event?event_id={event_id}");
    let resp = gloo_net::http::Request::get(&url)
        .send()
        .await
        .map_err(|e| e.to_string())?;
    resp.json::<EventSeats>().await.map_err(|e| e.to_string())
}
```

**ฝั่ง server** (endpoint ของ Axum ที่ต้องเขียนแยกไว้อีกจุด — ทับซ้อนกับ struct ฝั่ง client ทุกประการ):

```rust
use axum::extract::{Query, State};
use axum::http::StatusCode;
use axum::Json;
use std::collections::HashMap;

async fn get_event_handler(
    State(pool): State<PgPool>,
    Query(params): Query<HashMap<String, String>>,
) -> Result<Json<EventSeats>, StatusCode> {
    let event_id: i64 = params
        .get("event_id")
        .and_then(|s| s.parse().ok())
        .ok_or(StatusCode::BAD_REQUEST)?;

    let row = sqlx::query_as::<_, EventSeats>(
        "SELECT id, name, total_seats, booked_seats FROM events WHERE id = $1",
    )
    .bind(event_id)
    .fetch_one(&pool)
    .await
    .map_err(|_| StatusCode::NOT_FOUND)?;

    Ok(Json(row))
}

// ต้องไปเพิ่ม route นี้เข้า Router เองอีกจุด แยกจาก route อื่น ๆ ทั้งหมด
// .route("/api/manual/get_event", get(get_event_handler))
```

ทั้งสองไฟล์นี้ทดสอบ compile ผ่านจริงแยกกัน (ฝั่ง client compile บน `wasm32-unknown-unknown` ฝั่ง server compile บน native target) — สิ่งที่ต้องสังเกตคือ **struct `EventSeats` ต้องเขียนซ้ำสองที่** (หรือแยกเป็น shared crate ที่ทั้งสองฝั่ง import — งาน setup เพิ่มอีกชั้น) **ต้องคิดชื่อ path เอง คิด encoding เอง เขียน error handling คู่ขนานเอง** และถ้าเปลี่ยน signature ของ `get_event` (เช่นเพิ่ม parameter) ต้องแก้ทั้งสามที่ให้ตรงกันเอง (struct, endpoint handler, fetch call) โดยไม่มี compiler ช่วยเตือนว่าสามจุดนี้ไม่ตรงกันแล้ว (เพราะมันเป็นโค้ดคนละไฟล์ที่เชื่อมกันด้วย string URL ไม่ใช่ type system)

เทียบกับ `#[server]` ที่กำลังจะเห็นต่อไปนี้ — เขียนแค่ **ฟังก์ชันเดียว** และ struct **ครั้งเดียว** compiler จะบังคับให้ signature ของ client/server ตรงกันเสมอ (เพราะมันมาจากไฟล์ต้นฉบับเดียวกัน) ถ้าแก้ signature ผิดที่ใดที่หนึ่ง compile error ทันทีทั้งสอง target แทนที่จะไปเจอ bug ตอน runtime ว่า client ส่ง argument ไม่ตรงกับที่ server คาดหวัง

Leptos ตัด "งานเชื่อมต่อ" นี้ออกไปด้วย macro `#[server]` หลักการคือ: **คุณเขียนฟังก์ชัน async ตัวเดียว** แล้วประกาศ `#[server]` ไว้บนหัวฟังก์ชันนั้น — Leptos จะ generate โค้ดให้สองแบบ ขึ้นกับว่า compile ด้วย feature อะไร:

- ถ้า compile ด้วย feature `ssr` (ฝั่ง server): เนื้อ function ทำงาน**ตรง ๆ ตามที่เขียน** และมันถูกลงทะเบียนเป็น HTTP endpoint จริงโดยอัตโนมัติ (ที่ path `/api/<ชื่อฟังก์ชัน>` เป็น default)
- ถ้า compile ด้วย feature `csr`/`hydrate` (ฝั่ง client/WASM): เนื้อ function ที่คุณเขียนจะ**ไม่ถูก compile เข้าไปเลย** — macro แทนที่มันด้วยโค้ดที่ทำ HTTP request ไปยัง endpoint นั้น (serialize argument เป็น request body, deserialize response กลับเป็น return type)

พูดให้กระชับ: **คุณเขียนโค้ดเหมือนมันเป็น local function call ธรรมดา แต่ compile ออกมาเป็นคนละโค้ดกันเลยขึ้นกับ target** — แนวคิดนี้คล้าย tRPC ในโลก TypeScript (เขียนฟังก์ชันฝั่ง server เรียกจาก client แบบมี type safety เต็มรูปแบบโดยไม่ต้องเขียน REST endpoint มือ) แต่ไม่เหมือนกันเป๊ะ: tRPC ทำงานผ่าน TypeScript type inference ข้าม client/server (ทั้งคู่เป็น TypeScript ภาษาเดียวกัน อยู่ใน monorepo เดียวกัน) ส่วน Leptos ทำผ่าน **compile-time code generation แบบ conditional compilation** (`#[cfg(feature = "ssr")]` ภายใน macro) — ทั้งฝั่ง client และ server มาจาก**ซอร์สโค้ด Rust ไฟล์เดียวกัน** แค่ compile ด้วย feature flag ต่างกันสองครั้ง

ยืนยันจากคำอธิบายจริงในซอร์สโค้ดของ macro (`leptos_macro-0.8.18/src/lib.rs`, doc comment เหนือฟังก์ชัน `server`):

> "Declares that a function is a server function. This means that its body will only run on the server, i.e., when the `ssr` feature on this crate is enabled. If you call a server function from the client (i.e., when the `csr` or `hydrate` features are enabled), it will instead make a network request to the server."

และมีข้อกำหนดสำคัญที่ระบุไว้ตรง ๆ ในเอกสารเดียวกัน (คัดมาเพื่อไม่ให้พลาด):

- **Server function ต้องเป็น `async`** เสมอ แม้งานข้างในจะทำงานแบบ synchronous ได้ก็ตาม — เพราะจากมุมมองฝั่ง client มันคือ network call ที่ต้อง await เสมอ
- **ต้องคืนค่าเป็น `Result<T, ServerFnError>`** เสมอ แม้งานข้างในจะไม่มีทาง fail ก็ตาม — เพราะ (de)serialization และ network call สามารถ fail ได้เสมอ
- **Server function คือ public HTTP API** — endpoint ที่ generate ขึ้นมาเรียกจาก HTTP client ตัวไหนก็ได้ ไม่ใช่แค่จาก WASM ของแอปคุณเอง ต้องระวังเรื่อง data ที่ควรอยู่ฝั่ง server เท่านั้นไม่ให้หลุดออกมาผ่าน return value
- **ห้าม generic** — เพราะแต่ละ server function สร้าง endpoint แยกกันหนึ่งอัน การ monomorphize (Part 22) ทำได้ยากในบริบทนี้

#### ตัวอย่างที่ 1 — Query: อ่านข้อมูล event จากฐานข้อมูลจริง

ต่อยอดโดเมนระบบจองตั๋วงานสัมมนา สร้างตาราง `events` จริงใน PostgreSQL (cluster เดียวกับที่ Part 70 ใช้):

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    total_seats INT NOT NULL,
    booked_seats INT NOT NULL DEFAULT 0
);

INSERT INTO events (name, total_seats, booked_seats)
VALUES ('Rust Conf 2026', 100, 42);
```

server function ที่ query ข้อมูลนี้:

```rust
use leptos::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct EventSeats {
    pub id: i64,
    pub name: String,
    pub total_seats: i32,
    pub booked_seats: i32,
}

#[server(endpoint = "get_event")]
pub async fn get_event(event_id: i64) -> Result<EventSeats, ServerFnError> {
    // use_context::<T>() ดึง value ที่ leptos_axum ผูกไว้ให้ตอน setup (หัวข้อ 89.8)
    let pool = use_context::<sqlx::PgPool>()
        .ok_or_else(|| ServerFnError::new("ไม่พบ PgPool ใน context"))?;

    let row: EventSeats = sqlx::query_as!(
        EventSeats,
        "SELECT id, name, total_seats, booked_seats FROM events WHERE id = $1",
        event_id
    )
    .fetch_one(&pool)
    .await
    .map_err(|e| ServerFnError::new(format!("query error: {e}")))?;

    Ok(row)
}
```

สังเกตว่าโค้ดข้างในคือ SQLx ธรรมดาเป๊ะตามที่ Part 70 สอนไว้ทุกประการ (`sqlx::query_as!`, `#[derive(sqlx::FromRow)]` implicit ผ่าน struct ที่มี field ตรงชื่อ column, `PgPool`) — server function ไม่ได้เปลี่ยนวิธีคุยกับฐานข้อมูลเลย มันแค่เปลี่ยน "การเรียกใช้ฟังก์ชันนี้จากฝั่ง client" ให้เป็นไปโดยอัตโนมัติ `endpoint = "get_event"` เป็น named argument ของ `#[server]` ที่ระบุ path ตรง ๆ (ถ้าไม่ระบุ Leptos จะสร้าง path จากชื่อฟังก์ชัน+hash แทน ซึ่งอ่านยากกว่าตอน debug ด้วย `curl`)

ทดสอบเรียกจริงผ่าน `curl` (จำลองว่าเป็น request ที่ client-side WASM จะส่งให้ ถ้าเรียกจาก component จริงจะเรียกผ่าน `get_event(1).await` ตรง ๆ เหมือนฟังก์ชัน async ธรรมดา แล้ว Leptos สร้างโค้ด HTTP request ที่เทียบเท่ากับ `curl` นี้ให้อัตโนมัติ):

```bash
$ curl -sS -i -X POST http://127.0.0.1:3009/api/get_event \
    -H "Content-Type: application/x-www-form-urlencoded" \
    --data "event_id=1"

HTTP/1.1 200 OK
content-type: application/json
content-length: 68

{"id":1,"name":"Rust Conf 2026","total_seats":100,"booked_seats":42}
```

(ผลลัพธ์นี้คัดลอกจริงจากการรัน SSR server ที่เขียนในหัวข้อ 89.8 พร้อมฐานข้อมูล PostgreSQL จริง — ไม่ใช่ mock)

#### ตัวอย่างที่ 2 — Mutation: จองที่นั่งแบบ atomic ด้วย transaction

server function สำหรับ "จอง" คือจุดที่แสดงพลังของการรวม server function + transaction (Part 70) ได้ชัดที่สุด เพราะการจองต้องเป็น atomic operation (เช็คว่ายังมีที่นั่งเหลือ + เพิ่มจำนวนที่จอง ต้องสำเร็จทั้งคู่หรือไม่สำเร็จเลย เหมือนตัวอย่างยืมหนังสือใน Part 70):

```rust
#[server(endpoint = "book_seat")]
pub async fn book_seat(event_id: i64) -> Result<i32, ServerFnError> {
    let pool = use_context::<sqlx::PgPool>()
        .ok_or_else(|| ServerFnError::new("ไม่พบ PgPool ใน context"))?;

    let mut tx = pool
        .begin()
        .await
        .map_err(|e| ServerFnError::new(format!("begin tx error: {e}")))?;

    // FOR UPDATE ล็อกแถวนี้ไว้จนกว่า transaction จะจบ ป้องกัน race condition
    // ระหว่าง request ที่มาพร้อมกัน (เช่นสองคนกดจองที่นั่งสุดท้ายพร้อมกัน)
    let row = sqlx::query!(
        "SELECT total_seats, booked_seats FROM events WHERE id = $1 FOR UPDATE",
        event_id
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(|e| ServerFnError::new(format!("select error: {e}")))?;

    if row.booked_seats >= row.total_seats {
        return Err(ServerFnError::new("ที่นั่งเต็มแล้ว"));
    }

    let updated = sqlx::query!(
        "UPDATE events SET booked_seats = booked_seats + 1 WHERE id = $1 RETURNING booked_seats",
        event_id
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(|e| ServerFnError::new(format!("update error: {e}")))?;

    tx.commit()
        .await
        .map_err(|e| ServerFnError::new(format!("commit error: {e}")))?;

    Ok(updated.booked_seats)
}
```

ทดสอบเรียกจริงตอนที่นั่งยังไม่เต็ม (`booked_seats = 42` จาก 100):

```bash
$ curl -sS -i -X POST http://127.0.0.1:3009/api/book_seat \
    -H "Content-Type: application/x-www-form-urlencoded" --data "event_id=1"

HTTP/1.1 200 OK
content-type: application/json
content-length: 2

43
```

ตรวจสอบในฐานข้อมูลจริงหลังเรียก:

```sql
SELECT booked_seats FROM events WHERE id=1;
 booked_seats
--------------
           43
(1 row)
```

ค่าเปลี่ยนจาก 42 เป็น 43 จริง — transaction commit สำเร็จ จากนั้นทดสอบ error path โดย `UPDATE events SET booked_seats = 100` (ให้เต็มพอดี) แล้วเรียก `book_seat` อีกครั้ง:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3009/api/book_seat \
    -H "Content-Type: application/x-www-form-urlencoded" --data "event_id=1"

HTTP/1.1 500 Internal Server Error
serverfnerror: /api/book_seat
content-type: text/plain
content-length: 57

ServerError|ที่นั่งเต็มแล้ว
```

ผลลัพธ์นี้ยืนยันสองเรื่อง: (1) `return Err(ServerFnError::new("ที่นั่งเต็มแล้ว"))` แปลงเป็น HTTP 500 พร้อม body ที่มี prefix `ServerError|` ตามด้วยข้อความ error ของเรา — นี่คือ format ที่ `leptos_axum` ใช้ตอน serialize `ServerFnError` กลับไปให้ client (ฝั่ง client ที่เรียกผ่าน `.await` จริง จะได้ `Err(ServerFnError::ServerError("ที่นั่งเต็มแล้ว".to_string()))` กลับมาโดยตรง ไม่ต้อง parse string เอง เพราะโค้ด client-side ที่ macro generate ให้จัดการ deserialize ให้แล้ว) และ (2) transaction ทำงานถูกต้อง — ไม่มีการ `UPDATE` เกิดขึ้นเมื่อเงื่อนไข `booked_seats >= total_seats` เป็นจริง เพราะ `return Err(...)` เกิดขึ้น**ก่อน**เรียก `UPDATE` และ transaction ที่ไม่เคย `commit()` จะ rollback อัตโนมัติตอน `tx` ถูก drop (หลักการเดียวกับที่ Part 70 อธิบายเรื่อง RAII ของ `Transaction`)

#### ดึงข้อมูลตอน component โหลด: `Resource`/`LocalResource` + `<Suspense>`

`Action` (ที่จะเห็นในหัวข้อถัดไป) เหมาะกับการ "trigger" การเรียก server function จาก event เช่นคลิกปุ่ม — แต่สำหรับ query ที่ต้องโหลดทันทีที่ component ปรากฏ (เช่นโหลดรายการหนังสือทั้งหมดตอนเปิดหน้า) รูปแบบมาตรฐานคือ **`Resource`** จับคู่กับ **`<Suspense>`** `Resource::new(source, fetcher)` รับสอง closure: `source` (signal ที่ถ้าเปลี่ยนค่าจะ trigger การ fetch ใหม่ — ใส่ `|| ()` ถ้าต้องการ fetch แค่ครั้งเดียวตอน mount) และ `fetcher` (closure async ที่ทำการดึงข้อมูลจริง มักเป็นการเรียก server function ตรง ๆ)

ทดสอบด้วยตัวอย่างจริง (ใช้ `LocalResource::new` แทน `Resource::new` เพราะ future ที่ทดสอบไม่ satisfy `Send` bound — รายละเอียดเรื่องนี้อยู่ในกับดักท้ายบท ในแอปจริงที่ future มาจาก server function ผ่าน HTTP client ที่เป็น `Send` ได้ตามปกติ มักใช้ `Resource::new` ตรง ๆ ได้เลยไม่ต้องพึ่ง `LocalResource`):

```rust
use leptos::prelude::*;

// จำลอง network delay 300ms เหมือน server function จริงที่ต้องรอ round-trip
// (ในแอปจริงฟังก์ชันนี้จะถูกแทนด้วยการเรียก get_book_count() ซึ่งเป็น server function จริง)
async fn fetch_book_count() -> i32 {
    gloo_timers::future::TimeoutFuture::new(300).await;
    42
}

#[component]
fn BookCountDisplay() -> impl IntoView {
    let book_count = LocalResource::new(fetch_book_count);

    view! {
        // <Suspense> คือ "ตัวจับ" resource ที่ยังโหลดไม่เสร็จทุกตัวที่อยู่ข้างใน —
        // ระหว่างที่ยังไม่เสร็จ (ไม่ว่าจะมี resource กี่ตัวก็ตาม) มันแสดง fallback แทน
        <Suspense fallback=move || view! { <p id="loading">"กำลังโหลด..."</p> }>
            <p id="result">"จำนวนหนังสือ: " {move || book_count.get()}</p>
        </Suspense>
    }
}
```

build และรันจริงในเบราว์เซอร์ (จำลอง network delay 300ms ด้วย `gloo_timers::future::TimeoutFuture` แทนการต่อ server function จริง เพื่อให้เห็นสถานะ "กำลังโหลด" ได้ชัดในการทดสอบ) — ดักจับ DOM ทั้งช่วงก่อนและหลัง resource resolve เสร็จ ผลลัพธ์จริงที่ได้:

```
immediately after load:
<p id="loading">กำลังโหลด...</p>

after 600ms:
<p id="result">จำนวนหนังสือ: 42</p>
```

พิสูจน์ตรงตามที่ตั้งใจ: ทันทีที่ component mount (ก่อนที่ `fetch_book_count()` จะ resolve) `<Suspense>` แสดง `fallback` (ข้อความ "กำลังโหลด...") ให้เห็นทันที แทนที่จะปล่อยให้หน้าจอค้างเปล่า ๆ จนกว่าข้อมูลมาถึง — พอ resource resolve เสร็จ (หลัง 300ms ในตัวอย่างนี้) `<Suspense>` สลับไปแสดง children จริงเองโดยอัตโนมัติ (ค่า `42` ที่ `book_count.get()` คืนมา) โดยที่คุณไม่ต้องเขียน `if`/`match` เช็คสถานะ loading เองเลยแม้แต่จุดเดียว — `<Suspense>` จัดการ "มี resource ตัวไหนในลูกของมันที่ยังไม่เสร็จหรือไม่" ให้อัตโนมัติ (แม้จะมีหลาย `Resource` ซ้อนกันหลายตัวอยู่ข้างในก็ตาม มันจะรอให้ครบทุกตัวก่อนเลิกแสดง fallback)

รูปแบบนี้คือคำตอบให้กับ hint ของแบบฝึกหัดข้อ 4 ท้ายบท (`Resource::new(|| (), |_| get_all_books())`) — ในแอปจริงที่ต่อกับ server function จริง (ซึ่ง future ของมันเป็น `Send` ได้ตามปกติเพราะไม่ได้พึ่ง JS API ที่ผูกกับ thread เดียวแบบ `gloo_timers`) จะใช้ `Resource::new` ตรง ๆ แทน `LocalResource::new` ได้เลย

#### เรียกจาก component จริง

ในโค้ด component ฝั่ง client คุณเรียก server function เหมือนฟังก์ชัน async ธรรมดา ไม่ต้องเขียนโค้ด fetch เอง — ตัวอย่างการผูกกับปุ่มจองตั๋ว โดยใช้ `Action` (ตัวช่วยของ Leptos สำหรับผูก async operation ที่ trigger จาก event เข้ากับ reactive state):

```rust
#[component]
fn BookingButton(event_id: i64) -> impl IntoView {
    let book_action = Action::new(move |_: &()| book_seat(event_id));

    view! {
        <button on:click=move |_| { book_action.dispatch(()); }>
            "จองที่นั่ง"
        </button>
        {move || match book_action.value().get() {
            Some(Ok(new_count)) => format!("จองสำเร็จ ตอนนี้จองไปแล้ว {new_count} ที่"),
            Some(Err(e)) => format!("จองไม่สำเร็จ: {e}"),
            None => "ยังไม่ได้กดจอง".to_string(),
        }}
    }
}
```

`book_action.dispatch(())` เรียก `book_seat(event_id)` — ซึ่งถ้า compile ด้วย feature `hydrate`/`csr` จะกลายเป็นโค้ดที่ยิง HTTP POST ไปยัง `/api/book_seat` ตามที่พิสูจน์ด้วย `curl` ข้างบน โดยที่**คุณไม่ได้เขียนโค้ด fetch สักบรรทัดเดียว** — นี่คือสิ่งที่ทำให้ server function เป็นจุดขายที่แท้จริงของ Leptos ไม่ใช่แค่ syntax sugar: มันลดงาน "เชื่อมต่อ client-server" ที่ปกติต้องเขียนคู่กันสองฝั่ง (endpoint + fetch call) ให้เหลือแค่ฝั่งเดียว

#### ตัวเลือกการ encode: ทำไม `curl` ข้างบนต้องส่งเป็น `x-www-form-urlencoded`

สังเกตว่าการทดสอบด้วย `curl` ข้างบนส่ง argument ผ่าน `Content-Type: application/x-www-form-urlencoded` (เช่น `--data "event_id=1"`) ไม่ใช่ JSON — นี่ไม่ใช่เรื่องบังเอิญ ตามเอกสารของ macro `#[server]` (หัวข้อ "Server Function Encodings" ที่ตรวจสอบมาจากซอร์สโค้ดจริงของ `leptos_macro-0.8.18/src/lib.rs`) ค่า default ของ `input` คือ `PostUrl` (POST request แบบ URL-encoded body — เหมือนที่ HTML `<form>` ธรรมดาส่ง) และค่า default ของ `output` คือ `Json` ตัวเลือกอื่นที่ระบุได้ผ่าน named argument `input`/`output` ของ `#[server(...)]`:

| Encoding | HTTP Method | Request body | เหมาะกับ |
|---|---|---|---|
| `PostUrl` (default สำหรับ `input`) | `POST` | URL-encoded | ค่า default ทั่วไป ใช้ได้กับ argument ส่วนใหญ่ |
| `GetUrl` | `GET` | URL-encoded (เป็น query string) | server function ที่เป็น query ล้วน ๆ ไม่มี side effect — ทำให้ browser/CDN cache ได้ (ต่างจาก `POST` ที่ไม่ถูก cache ตามธรรมชาติของ HTTP) |
| `Cbor` | `POST` | CBOR (binary format กระชับกว่า JSON) | payload ขนาดใหญ่ที่อยากลด bandwidth |
| `Json` (default สำหรับ `output`) | - | JSON | ค่า default ของ response แทบทุกกรณี |

ตัวอย่างการเปลี่ยน `get_event` (ซึ่งเป็น query ล้วน ๆ ไม่มี side effect) ให้ใช้ `GetUrl` แทน `PostUrl` — ทำให้เรียกด้วย HTTP GET ธรรมดาได้ (เปิดทางให้ browser cache ผลลัพธ์ได้ถ้าต้องการ ต่างจาก `book_seat` ที่เป็น mutation และควรคงเป็น `POST` เพราะมี side effect เสมอ ตามหลักการ HTTP method semantics ที่ Part 61 สอนไว้):

```rust
use leptos::server_fn::codec::GetUrl;

#[server(endpoint = "get_event", input = GetUrl)]
pub async fn get_event(event_id: i64) -> Result<EventSeats, ServerFnError> {
    // เนื้อโค้ดเหมือนเดิมทุกอย่าง — เปลี่ยนแค่ transport
    // ...
}
```

หลังเปลี่ยน `curl` ที่ใช้ทดสอบก็ต้องเปลี่ยนตามให้ตรง (`GET` พร้อม query string แทน `POST` พร้อม body) — ทดสอบจริงแล้ว compile ผ่านและได้ผลลัพธ์ถูกต้องเหมือนเดิม:

```bash
$ curl -sS "http://127.0.0.1:3009/api/get_event?event_id=1"
{"id":1,"name":"Rust Conf 2026","total_seats":100,"booked_seats":42}
```

#### ทดสอบ server function โดยตรงแบบ unit test — ไม่ต้องมี HTTP request จริง

เพราะ server function ก็คือ async function ธรรมดาเมื่อ compile ด้วย feature `ssr` (ตามที่อธิบายไว้ตั้งแต่ต้นหัวข้อ) คุณสามารถเรียกมันตรง ๆ ใน `#[tokio::test]` ได้เลยโดยไม่ต้องเปิด HTTP server จริงหรือยิง `curl` เทียบกับ integration test แบบ Part 32-33 — สิ่งเดียวที่ต้องทำเพิ่มคือ **สร้าง reactive owner และ provide context เอง** (ปกติ `leptos_axum` เป็นคนทำให้อัตโนมัติตอน handle request จริงตามที่ตั้งไว้ใน `.leptos_routes_with_context(...)` ในหัวข้อ 89.8):

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_get_event_directly() {
        let pool = PgPoolOptions::new()
            .max_connections(2)
            .connect("postgres://postgres:postgres@localhost/leptos_scratch")
            .await
            .expect("connect");

        // server function เรียกตรง ๆ ได้โดยไม่ต้องมี HTTP request จริง
        // ต้อง provide_context เองเพราะปกติ leptos_axum เป็นคนทำให้ตอน handle request จริง
        let owner = leptos::prelude::Owner::new();
        owner.set();
        provide_context(pool);

        let result = get_event(1).await;
        assert!(result.is_ok());
        let event = result.unwrap();
        assert_eq!(event.name, "Rust Conf 2026");
    }
}
```

ทดสอบรันจริงด้วย `cargo test` (เชื่อมต่อฐานข้อมูล `leptos_scratch` เดียวกับที่ใช้ทดสอบ `curl` ในหัวข้อนี้) ได้ผลลัพธ์จริง:

```
running 1 test
test tests::test_get_event_directly ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.04s
```

การทดสอบแบบนี้เร็วกว่าการเปิด server จริงแล้วยิง `curl`/HTTP client มาก (ไม่มี network round-trip, ไม่ต้อง serialize/deserialize ข้าม HTTP) เหมาะสำหรับ unit test ที่ต้องการตรวจสอบ business logic ข้างในเท่านั้น ส่วนการทดสอบผ่าน HTTP จริง (แบบที่ทำด้วย `curl` ตลอดหัวข้อนี้) ยังจำเป็นสำหรับ integration test ที่ต้องพิสูจน์ว่า encoding/routing ทำงานถูกต้องครบทั้ง pipeline ด้วย — ทั้งสองระดับเสริมกัน ไม่ใช่แทนกัน

#### `ServerFnError` มีอะไรมากกว่าที่เห็น

`ServerFnError::new(...)` ที่ใช้มาตลอดหัวข้อนี้เป็นแค่การสร้าง custom error message string — แต่ `ServerFnError<E>` เป็น enum ที่มีหลาย variant ครอบคลุม failure mode ที่พบบ่อยของ "การเรียกฟังก์ชันข้าม network" (ต่างจาก error ของฟังก์ชันธรรมดาที่ Part 12 สอน ซึ่งมักมีสาเหตุจำกัดกว่า) เช่น error ตอน serialize/deserialize argument ล้มเหลว, error ตอน network request ล้มเหลว (client หลุดการเชื่อมต่อกลางทาง), หรือ error ตอนหา endpoint ไม่เจอ (เผลอเรียก path ผิด) — ทั้งหมดนี้ Leptos ห่อให้เป็น `ServerFnError` เดียวกันเพื่อให้โค้ดฝั่ง client จัดการ error แบบเดียวกันได้ไม่ว่าจะ fail ที่ขั้นไหนของ pipeline (ตามที่ระบุไว้ใน doc comment ของ macro: "arguments need to be serialized... and the return type must be serialized... this means that the set of valid server function argument and return types is a subset of all possible Rust types") — ในทางปฏิบัติ สำหรับ error ทางธุรกิจของแอปคุณเอง (เช่น "ที่นั่งเต็มแล้ว" ในตัวอย่างนี้) การใช้ `ServerFnError::new(...)` เพียงพอแล้ว ส่วน variant อื่น ๆ ส่วนใหญ่ Leptos สร้างให้อัตโนมัติเมื่อเกิด failure ระดับ transport ที่ไม่ใช่ business logic ของคุณ

ทดสอบจริงด้วยการยิง request ที่ไม่ส่ง `event_id` ไปให้ `get_event` (ลืมใส่ argument ที่ต้องมี) เทียบกับยิงไปที่ path ที่ไม่มีอยู่จริง เพื่อดู variant อื่นของ error ที่ Leptos สร้างให้อัตโนมัติ (ไม่ใช่ business error ที่เราเขียนเอง):

```bash
$ curl -sS -i -X POST http://127.0.0.1:3009/api/get_event \
    -H "Content-Type: application/x-www-form-urlencoded" --data ""
HTTP/1.1 500 Internal Server Error
serverfnerror: /api/get_event
content-type: text/plain

Args|missing field `event_id`

$ curl -sS -i http://127.0.0.1:3009/api/nonexistent_fn
HTTP/1.1 404 Not Found
```

`Args|missing field` คือ error ที่เกิดจากขั้น**deserialize argument** (ตรงกับ variant `ServerFnError::Args` — ต่างจาก `ServerFnError::ServerError` ที่ error ทางธุรกิจของเราใช้ ซึ่งจะขึ้น prefix `ServerError|` ตามที่เห็นในตัวอย่าง "ที่นั่งเต็มแล้ว" ก่อนหน้านี้) เกิดขึ้น**ก่อน**ที่โค้ดข้างในฟังก์ชัน `get_event` จะได้รันด้วยซ้ำ เพราะ Leptos ต้อง deserialize argument ให้สำเร็จก่อนเสมอ — พิสูจน์ว่า `ServerFnError` มีหลาย "ชั้น" ของความล้มเหลวจริง ไม่ใช่แค่ error message เดียวที่เราเขียนเอง ส่วน path ที่ไม่มีอยู่จริงได้ 404 ธรรมดาจาก Axum เลย (ไม่ผ่านชั้น server function เลยด้วยซ้ำ เพราะ router ของ Axum เองเป็นคนตอบก่อนที่จะถึงชั้นของ server function)

### 89.7 Rendering Modes: CSR, SSR, และ Hydration

Leptos รองรับ rendering mode หลักสามแบบ ควบคุมด้วย feature flag:

- **`csr`** (Client-Side Rendering): เหมือนโมเดลของ Yew ทั้งหมด — server ส่ง `index.html` เปล่า ๆ พร้อมไฟล์ WASM มาให้ browser แล้วทุกอย่างสร้างขึ้นฝั่ง client ล้วน ๆ ข้อเสียคือผู้ใช้เห็นหน้าขาวจนกว่า WASM จะโหลดและรันเสร็จ (อาจช้าถ้าเน็ตไม่ดีหรือ WASM bundle ใหญ่) และ search engine crawler ที่ไม่รัน JS/WASM จะเห็นหน้าเปล่า
- **`ssr`** (Server-Side Rendering): server render `view!` tree เป็น**HTML string จริง**ตั้งแต่ request แรก (ตามที่เห็นผลลัพธ์จริงจาก `curl http://127.0.0.1:3009/` ในหัวข้อ 89.8) ผู้ใช้เห็นเนื้อหาทันทีที่ HTML มาถึง ไม่ต้องรอ WASM โหลดเลย ดีต่อ perceived performance และ SEO
- **`hydrate`**: ใช้คู่กับ `ssr` เสมอ — เป็น build target ฝั่ง client ที่ทำหน้าที่ **hydration**

#### Hydration คืออะไรกันแน่

Hydration คือกลไกที่ทำให้ SSR ต่างจาก "แค่ render HTML แล้วจบ" — หลังจาก server ส่ง HTML จริงมาให้ (และผู้ใช้เห็นเนื้อหาแล้วทันที) ไฟล์ WASM (จาก build target `hydrate`) จะโหลดตามมาทีหลัง เมื่อโหลดเสร็จมันจะ:

1. **เดินผ่าน DOM tree ที่มีอยู่แล้ว** (ที่ server ส่งมาเป็น HTML) จับคู่กับโครงสร้างที่ `view!` macro กำหนดไว้ (โดยอ้างจากตำแหน่ง/ลำดับของ node ให้ตรงกับตอน render ฝั่ง server)
2. **ผูก event listener** (`on:click` และอื่น ๆ) เข้ากับ DOM node ที่มีอยู่แล้วเหล่านั้น
3. **เชื่อม reactive graph** (signal, effect) เข้ากับ node เดิม เพื่อให้ node เหล่านั้นกลาย เป็น "reactive" ต่อจากนี้

จุดสำคัญที่สุดคือ **hydration ไม่ทำลาย DOM เดิมแล้วสร้างใหม่** — มันแค่ "จับมือ" กับ DOM ที่มีอยู่แล้วให้กลายเป็น interactive ผู้ใช้จะไม่เห็นการกระพริบหรือ flash ของหน้าจอเลยระหว่างขั้นตอนนี้ (ต่างจากแนวทางเดิมสมัย React ก่อนมี hydration ที่บางครั้ง client ต้อง re-render ทับ HTML ที่ server ส่งมาทั้งหมด) นี่คือเหตุผลที่คำว่า "hydrate" (เติมน้ำ) ถูกเลือกใช้เป็นคำเปรียบเปรย: HTML ที่ server ส่งมาเป็นเหมือน "โครงแห้ง" (มีรูปร่างครบแต่ยังไม่มีชีวิต ยังกดปุ่มไม่ได้) ส่วน WASM ที่มา hydrate คือการ "เติมชีวิต" ให้โครงเดิมโดยไม่เปลี่ยนรูปร่างมันเลย

การที่ SSR + hydration ทำงานร่วมกันได้ดีเป็นเรื่องที่ต้องอาศัยความร่วมมือจาก fine-grained reactivity โดยตรง — เพราะ Leptos รู้อยู่แล้วตั้งแต่ compile time ว่า signal ตัวไหนผูกกับ node ไหน (ตามที่พิสูจน์ในหัวข้อ 89.1) การ hydrate จึงทำได้แค่ "หา node ที่ตรงตำแหน่งแล้วผูก effect เข้าไป" โดยไม่ต้องสร้าง representation ใหม่มาเทียบกับ DOM เดิมก่อน (ถ้าใช้โมเดล Virtual DOM การ hydrate มักซับซ้อนกว่านี้ เพราะต้องมีขั้นตอน reconcile ต้นไม้ virtual กับ DOM จริงที่ server ส่งมา)

#### พิสูจน์ hydration แบบเต็ม pipeline: SSR ส่ง HTML จริง แล้ว WASM มา "จับมือ" กับ node เดิมจริง

คำอธิบายข้างบนพิสูจน์ได้จริงแบบครบ pipeline (ไม่ใช่แค่ทฤษฎี) — ใช้โครงสร้างโปรเจกต์ Axum + `leptos_axum` + SQLx แบบเดียวกับที่หัวข้อ 89.8 จะอธิบายเต็ม ๆ ต่อไป (routing, `.leptos_routes_with_context`, server function ที่คุยกับฐานข้อมูล) แต่เพิ่มการ compile เป็น**สอง target จากซอร์สโค้ดชุดเดียวกัน**: build เป็น native binary ด้วย feature `ssr` (`cargo build --features ssr`) สำหรับรัน Axum server และ build เป็น WASM ด้วย feature `hydrate` (`wasm-pack build --target web --features hydrate`) สำหรับให้ browser โหลด — component `App` (มี signal `count` และปุ่ม increment เหมือนหัวข้อ 89.3-89.4) เขียนไว้**ที่เดียว**ใน `lib.rs` ใช้ร่วมกันทั้งสอง target โดยไม่ต้องเขียนซ้ำ (ต่างจากตัวอย่างในหัวข้อ 89.8 ที่ตั้งใจให้เรียบง่ายด้วยการ build แค่ target `ssr` อย่างเดียวเพื่อโฟกัสที่ server function ก่อน แล้วค่อยเพิ่ม `hydrate` เข้ามาที่นี่)

โครงสร้าง `Cargo.toml` ที่ทำให้ compile ได้สองแบบจาก dependency set คนละชุด (`sqlx`/`axum`/`tokio` ต้องเป็น optional dependency ที่ผูกกับ feature `ssr` เท่านั้น ไม่อย่างนั้น `wasm-pack` จะพยายาม compile `sqlx` ลง `wasm32-unknown-unknown` ซึ่งจะพังเพราะ driver ของ PostgreSQL ต้องพึ่ง native networking):

```toml
[package]
name = "leptos_ssr"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[[bin]]
name = "leptos_ssr"
required-features = ["ssr"]

[dependencies]
leptos = "0.8.21"
serde = { version = "1", features = ["derive"] }
wasm-bindgen = "0.2"
console_error_panic_hook = "0.1"

axum = { version = "0.8.9", optional = true }
leptos_axum = { version = "0.8.10", optional = true }
tokio = { version = "1", features = ["full"], optional = true }
sqlx = { version = "0.9.0", default-features = false, features = ["runtime-tokio", "postgres", "macros"], optional = true }
tower-http = { version = "0.6", features = ["fs"], optional = true }

[features]
hydrate = ["leptos/hydrate"]
ssr = ["leptos/ssr", "dep:leptos_axum", "dep:axum", "dep:tokio", "dep:sqlx", "dep:tower-http"]
```

`lib.rs` มี `App` component ตัวเดียวที่ใช้ทั้งสอง target บวกจุดเข้าฝั่ง hydrate ที่ cfg ไว้ให้ compile เฉพาะเมื่อเปิด feature `hydrate` เท่านั้น:

```rust
#[component]
pub fn App() -> impl IntoView {
    let (count, set_count) = signal(0);
    view! {
        <h1>"Rust Conf 2026"</h1>
        <p id="count-display">"Count: " {count}</p>
        <button id="inc-btn" on:click=move |_| set_count.update(|n| *n += 1)>
            "increment"
        </button>
    }
}

// จุดเข้าฝั่ง client — คอมไพล์เฉพาะตอน build ด้วย --features hydrate เท่านั้น
#[cfg(feature = "hydrate")]
#[wasm_bindgen::prelude::wasm_bindgen(start)]
pub fn hydrate() {
    console_error_panic_hook::set_once();
    // hydrate_body ต่างจาก mount_to_body (หัวข้อ 89.2) ตรงที่มันไม่สร้าง DOM
    // ใหม่ทั้งหมด — มันเดินเข้าไป "จับคู่" กับ DOM ที่มีอยู่แล้วจาก SSR
    leptos::mount::hydrate_body(App);
}
```

`main.rs` (compile เฉพาะ target `ssr` เพราะ `required-features = ["ssr"]` ใน `Cargo.toml`) เพิ่มสองอย่างจากตัวอย่างในหัวข้อ 89.8: `<HydrationScripts>`/`<AutoReload>` ใน shell, และ `.nest_service("/pkg", ServeDir::new("pkg"))` เพื่อเสิร์ฟไฟล์ที่ `wasm-pack` สร้างไว้เป็น static file:

```rust
use leptos_ssr::App;

fn shell(options: LeptosOptions) -> impl IntoView {
    view! {
        <!DOCTYPE html>
        <html lang="th">
            <head>
                <meta charset="utf-8" />
                <title>"Leptos SSR + Hydrate Demo"</title>
                // สองตัวนี้คือคนที่ generate <script>/<link> ที่โหลด WASM
                // ให้อัตโนมัติ — ไม่ต้องเขียน <script> เองเลย
                <leptos::hydration::AutoReload options=options.clone() />
                <leptos::hydration::HydrationScripts options=options.clone() />
            </head>
            <body>
                <App />
            </body>
        </html>
    }
}

// ในฟังก์ชัน main (ส่วนที่เหลือเหมือนหัวข้อ 89.8 ทุกประการ) เพิ่มแค่บรรทัดนี้
// ลงใน Router เพื่อเสิร์ฟไฟล์ WASM/JS จากโฟลเดอร์ pkg/ ที่ wasm-pack สร้างไว้:
// .nest_service("/pkg", tower_http::services::ServeDir::new("pkg"))
```

build ทั้งสอง target จริง (`cargo build --no-default-features --features ssr` ได้ native binary, `wasm-pack build --target web --no-default-features --features hydrate` ได้ `pkg/leptos_ssr.js` + `pkg/leptos_ssr_bg.wasm` — ชื่อไฟล์ตรงกับที่ `<HydrationScripts>` คาดหวังไว้พอดีเพราะทั้งคู่ยึด `output_name` เดียวกันจาก `LeptosOptions`) ผลลัพธ์จริงจาก `curl http://127.0.0.1:3009/` (server ที่ compile ด้วย feature `ssr`) แสดง HTML ที่มีทั้งเนื้อหาที่ render จริง**และ**สคริปต์ hydration ที่ Leptos ฝังมาให้อัตโนมัติผ่าน component `<HydrationScripts>`:

```html
<body>
  <h1>Rust Conf 2026</h1>
  <p>หน้านี้ถูก render เป็น HTML จริงบนฝั่ง server ก่อน แล้วค่อย hydrate ให้กดปุ่มได้</p>
  <p id="count-display">Count: <!>0</p>
  <button id="inc-btn">increment</button>
</body>
```

```html
<script type="module" nonce="...">
(function (root, pkg_path, output_name, wasm_output_name) {
	import(`${root}/${pkg_path}/${output_name}.js`)
		.then(mod => {
			mod.default({module_or_path: `${root}/${pkg_path}/${wasm_output_name}.wasm`}).then(() => {
				mod.hydrate();
			});
		})
})
("", "pkg", "leptos_ssr", "leptos_ssr_bg");
</script>
```

สังเกตว่า `<p id="count-display">Count: 0</p>` เป็น **HTML จริงที่มีค่า `0` อยู่แล้ว** ตั้งแต่ response แรกที่ server ตอบมา (ไม่ใช่ placeholder เปล่า ๆ) และสคริปต์ที่แนบมาทำหน้าที่ตรงตามชื่อ `mod.hydrate()` — import ไฟล์ JS/WASM ที่ build จาก feature `hydrate` มา แล้วเรียกฟังก์ชัน `hydrate()` (ตรงกับ `#[wasm_bindgen(start)] pub fn hydrate() { leptos::mount::hydrate_body(App); }` ที่เขียนไว้ใน `lib.rs`) ไม่ใช่ `mount_to_body()` แบบที่ใช้ในตัวอย่าง CSR ล้วน ๆ ของหัวข้อ 89.2-89.5 — `hydrate_body` คือฟังก์ชันที่ทำหน้าที่ "จับคู่กับ DOM ที่มีอยู่แล้ว" แทนการสร้าง DOM ใหม่ทั้งหมดที่ `mount_to_body` ทำ

เปิดหน้านี้ด้วยเบราว์เซอร์จริงผ่าน Playwright แล้วทดสอบสองเรื่อง: (1) กดปุ่ม increment ได้จริงหลัง WASM โหลดเสร็จ และ (2) **DOM node ของ `<h1>` ที่จับ reference ไว้ตั้งแต่ก่อน WASM โหลดเสร็จ (ตอนที่หน้ายังเป็น HTML ดิบจาก server) ยังเป็น object ตัวเดิมหลัง hydrate เสร็จแล้ว** — ผลลัพธ์จริงที่ได้:

```
after load, count-display: Count: 0
after 3 clicks, count-display: Count: 3
page errors: []

h1 node identity preserved through hydration: true
```

`h1 node identity preserved through hydration: true` คือหลักฐานตรงจุดที่สุดของคำอธิบายในหัวข้อนี้ทั้งหมด: WASM ที่โหลดมาไม่ได้ทำลาย `<h1>` ที่ server ส่งมาแล้วสร้างใหม่ — มันแค่เดินเข้าไป "จับมือ" กับ node เดิมที่มีอยู่แล้วในหน้า (เหมือนที่อธิบายไว้ข้างบนทุกคำ) และหลังจากนั้นปุ่ม `increment` ก็ใช้งานได้จริงเพราะ reactive graph ถูกผูกเข้ากับ node เดิมเรียบร้อยแล้ว ไม่มี error ใดๆเกิดขึ้นระหว่างกระบวนการนี้เลย (`page errors: []`) — นี่คือ SSR + hydration ที่ทำงานจริงแบบครบวงจร ไม่ใช่แค่คำอธิบายเชิงทฤษฎี

#### ข้อแลกเปลี่ยนของแต่ละ mode

ไม่มี rendering mode ใด "ดีที่สุด" แบบสัมบูรณ์ — แต่ละแบบแลกอะไรกับอะไรต่างกัน:

| มิติ | CSR | SSR + Hydration | Islands (89.9) |
|---|---|---|---|
| เวลาที่ผู้ใช้เห็นเนื้อหาแรก | ช้าสุด (ต้องรอ WASM ทั้งก้อนโหลด+รันก่อน) | เร็ว (HTML มาตั้งแต่ response แรก) | เร็ว (เหมือน SSR) |
| ขนาด WASM ที่ต้องส่งไปเบราว์เซอร์ | ทั้งแอป | ทั้งแอป (เพื่อ hydrate ทุกส่วน) | เฉพาะส่วนที่เป็น `#[island]` เท่านั้น |
| ความซับซ้อนของ infrastructure | ต่ำสุด (เป็น static file ก็พอ, ขึ้น CDN ได้ตรง ๆ) | สูงขึ้น (ต้องมี server รันตลอดเวลา, มี state ฝั่ง server) | สูงสุด (ต้องแยก build pipeline ระหว่าง server component กับ island) |
| SEO / เนื้อหาที่ crawler เห็น | แย่ (เห็นหน้าเปล่าถ้า crawler ไม่รัน JS) | ดี (เห็น HTML เต็ม ๆ ทันที) | ดี (เหมือน SSR) |
| เหมาะกับ | Dashboard/แอปภายในที่ผู้ใช้ login แล้วเท่านั้น ไม่สน SEO | เว็บสาธารณะทั่วไปที่สน perceived performance และ SEO | เว็บที่มีเนื้อหา static เยอะมากแต่มี interactive widget เป็นจุด ๆ (เช่น blog ที่มีปุ่มโหวตไม่กี่ปุ่ม) |

#### Streaming SSR: ไม่ต้องรอ resource ที่ช้าที่สุดก่อนส่ง HTML

รายละเอียดอีกชั้นที่ควรรู้ (ยังอยู่ในระดับแนวคิด — จะลงลึกกว่านี้ใน Part 91): SSR ของ Leptos ไม่ได้ "render ทั้งหน้าเป็น string เดียวเสร็จสมบูรณ์ก่อนค่อยส่ง" เสมอไป ตรวจสอบจากซอร์สโค้ดจริงของ `leptos_axum-0.8.10/src/lib.rs` พบว่าฟังก์ชันหลักที่ใช้ (ซึ่งเป็นสิ่งที่ `.leptos_routes_with_context(...)` เรียกใช้ภายใน) มีชื่อว่า `render_app_to_stream` และคำอธิบายจริงจากซอร์สโค้ดบอกไว้ตรง ๆ ว่า:

> "Returns an Axum Handler that listens for a `GET` request and tries to route it... serving an **HTML stream** of your application."

และ return type ภายในคือ `PinnedHtmlStream` (`Pin<Box<dyn Stream<Item = io::Result<Bytes>> + Send>>`) — เป็น **stream ของ byte chunk** ไม่ใช่ `String` ก้อนเดียว หมายความว่าถ้าหน้าเว็บของคุณมีส่วนที่ต้องรอ async resource ช้า (เช่น query ฐานข้อมูลที่ใช้เวลานาน ห่อด้วย `<Suspense>`) ส่วนอื่นของหน้าที่พร้อมแล้วสามารถถูกส่งไปให้ browser ได้ก่อน โดยที่ส่วนที่ยังรออยู่จะตามมาทีหลังในสตรีมเดียวกัน — ผู้ใช้เห็น "โครงหน้าเว็บ" เร็วขึ้นแทนที่จะต้องรอ resource ที่ช้าที่สุดของทั้งหน้าก่อนเห็นอะไรเลยแม้แต่ส่วนที่ไม่เกี่ยวข้อง นี่คือเหตุผลที่ SSR ของ Leptos ไม่ใช่แค่ "print HTML string แล้วส่ง" แบบง่าย ๆ

หัวข้อนี้เป็นการแนะนำแนวคิดเท่านั้น — **Part 91 จะลงรายละเอียด SSR แบบข้าม framework** (เปรียบเทียบวิธีที่ Yew, Leptos และ Dioxus (Part 90) ทำ SSR/hydration ต่างกันอย่างไร รวมถึงเรื่อง streaming SSR และ progressive hydration ที่ซับซ้อนกว่านี้) บทนี้ให้แค่ภาพรวมพอที่จะเข้าใจตัวอย่าง Axum integration ในหัวขัดต่อไป

### 89.8 การผสาน Leptos SSR เข้ากับ Axum จริง

ข้อเท็จจริงที่สำคัญมากคือ: **Leptos SSR ไม่ใช่ web server ของตัวเอง** มันคือ crate `leptos_axum` (หรือ `leptos_actix` ถ้าใช้ Actix-web) ที่เพิ่ม route ลงใน `axum::Router` ที่คุณสร้างเอง — หมายความว่า Axum ตัวเดียวกันที่คุณเรียนมาตั้งแต่ Module 4 (Part 62-66) สามารถให้บริการทั้ง Leptos SSR route และ route ธรรมดาที่คุณเขียนมือ (เช่น health check, webhook, หรือ API เดิมที่ไม่เกี่ยวกับ Leptos) **ใน `Router` เดียวกัน**

ตัวอย่างนี้ทดสอบด้วยการเขียนโปรเจกต์ manual (ไม่ผ่าน `cargo leptos new` เพื่อให้เห็นทุกจุดต่อชัด ๆ) — `Cargo.toml`:

```toml
[package]
name = "leptos_ssr"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8.9"
leptos = { version = "0.8.21", features = ["ssr"] }
leptos_axum = "0.8.10"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.9.0", default-features = false, features = ["runtime-tokio", "postgres", "macros"] }
tokio = { version = "1", features = ["full"] }
```

`main.rs` (รวม server function จากหัวข้อ 89.6 เข้ากับ setup ของ Axum):

```rust
use leptos::prelude::*;

// ... EventSeats struct, get_event(), book_seat() จากหัวข้อ 89.6 อยู่ที่นี่ ...

#[component]
pub fn App() -> impl IntoView {
    view! {
        <html lang="th">
            <head>
                <meta charset="utf-8" />
                <title>"Leptos SSR Demo"</title>
            </head>
            <body>
                <h1>"Rust Conf 2026"</h1>
                <p>"หน้านี้ถูก render เป็น HTML จริงบนฝั่ง server"</p>
            </body>
        </html>
    }
}

fn shell() -> impl IntoView {
    view! { <App /> }
}

#[tokio::main]
async fn main() {
    use axum::routing::get;
    use axum::{Json, Router};
    use leptos_axum::{generate_route_list, LeptosRoutes};
    use sqlx::postgres::PgPoolOptions;

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://postgres:postgres@localhost/leptos_scratch")
        .await
        .expect("connect to postgres");

    let leptos_options = leptos::config::LeptosOptions::builder()
        .output_name("leptos_ssr")
        .site_addr("127.0.0.1:3009".parse::<std::net::SocketAddr>().unwrap())
        .build();

    // generate_route_list สแกน view tree ของ App เพื่อหาว่าต้องสร้าง route อะไรบ้าง
    // (รองรับ leptos_router ถ้ามีหลายหน้า — ตัวอย่างนี้มีหน้าเดียวเพื่อความชัดเจน)
    let routes = generate_route_list(App);

    let pool_for_ctx = pool.clone();
    let app = Router::new()
        // .leptos_routes_with_context ผูก route ของ Leptos (ทั้งหน้า SSR และ
        // server function endpoint อย่าง /api/get_event, /api/book_seat)
        // เข้ากับ Router — closure ที่สองคือจุดที่ "ฉีด" PgPool เข้า context
        // ให้ server function เรียก use_context::<PgPool>() เจอ
        .leptos_routes_with_context(
            &leptos_options,
            routes,
            move || provide_context(pool_for_ctx.clone()),
            shell,
        )
        // route ธรรมดาที่เขียนมือ ตรงตามที่ Part 62-66 สอน — อยู่ใน Router
        // เดียวกับ route ของ Leptos เป๊ะ ๆ ไม่ต้องแยก server
        .route(
            "/healthz",
            get(|| async { Json(serde_json::json!({"status": "ok"})) }),
        )
        .with_state(leptos_options);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3009")
        .await
        .unwrap();
    axum::serve(listener, app.into_make_service()).await.unwrap();
}
```

คอมไพล์และรันจริง (`cargo build` สำเร็จ ไม่มี error หลังแก้ปัญหา type inference เล็กน้อยของ `sqlx::query!` — ต้องระบุ type annotation ให้ column ที่ macro infer ไม่ได้เอง) จากนั้นทดสอบด้วย `curl` จริงสามจุด:

**1) route หน้าเว็บ SSR (`GET /`)** — พิสูจน์ว่า server ส่ง HTML จริงมา ไม่ใช่หน้าเปล่า:

```bash
$ curl -sS http://127.0.0.1:3009/
<html lang="th"><head><meta charset="utf-8"><title>Leptos SSR Demo</title></head>
<body><h1>Rust Conf 2026</h1><p>หน้านี้ถูก render เป็น HTML จริงบนฝั่ง server</p></body></html>
<script nonce="...">__RESOLVED_RESOURCES=[];__SERIALIZED_ERRORS=[];...</script>
```

สังเกต `<h1>Rust Conf 2026</h1>` และเนื้อหาภาษาไทยทั้งหมดอยู่ใน HTML ที่ตอบกลับมาโดยตรง — นี่คือ SSR จริง (เทียบกับ CSR ที่ `curl` จะเห็นแค่ `<div id="app"></div>` เปล่า ๆ กับ `<script>` ที่ยังไม่รัน) ส่วน `<script>` ที่มี `__RESOLVED_RESOURCES` คือ metadata ที่ Leptos ฝังไว้สำหรับขั้นตอน hydration (หัวข้อ 89.7) ให้ WASM ที่โหลดตามมาทีหลังรู้ว่า resource ไหน resolve ไปแล้วบ้างตั้งแต่ตอน render ฝั่ง server

**2) route server function (`POST /api/get_event`, `/api/book_seat`)** — ผลลัพธ์ตรงตามที่แสดงเต็ม ๆ ในหัวข้อ 89.6

**3) route ธรรมดาที่เขียนมือ (`GET /healthz`)** — พิสูจน์ว่า route ของ Axum ปกติอยู่ร่วมกับ route ของ Leptos ได้ใน `Router` เดียวกันจริง:

```bash
$ curl -sS http://127.0.0.1:3009/healthz
{"status":"ok"}
```

นี่คือข้อเท็จจริงที่ควรค่าแก่การเน้นย้ำ: การเรียนรู้ Axum อย่างละเอียดตั้งแต่ Module 4 **ไม่ใช่ความรู้ที่ต้องทิ้งไปตอนมาใช้ Leptos SSR** — `.route()`, middleware ผ่าน `tower`, `State<T>` extractor, และทุกอย่างที่ Part 62-66 สอนยังใช้งานได้ตรง ๆ ในโปรเจกต์เดียวกับ Leptos ส่วนที่ Leptos เพิ่มเข้ามาคือ `.leptos_routes_with_context(...)` ซึ่งก็เป็นแค่ method หนึ่งบน `Router` ที่ผูก route เพิ่มเข้าไป ไม่ต่างจาก `.route(...)` ธรรมดาในหลักการ

#### สิ่งที่ `leptos_axum` ผูก context ให้อัตโนมัติเสมอ

`.leptos_routes_with_context(...)` ที่ใช้ในตัวอย่างข้างบนนี้ generate handler ที่ภายในเรียก `render_app_to_stream` (ฟังก์ชันที่พูดถึงในหัวข้อ 89.7 เรื่อง streaming) — และตรวจสอบจากซอร์สโค้ดจริงของ `leptos_axum-0.8.10` (doc comment ของฟังก์ชันนั้น) พบว่านอกจาก `PgPool` ที่เราฉีดเข้าไปเองผ่าน closure ที่สอง Leptos **ผูก context บางอย่างให้อัตโนมัติเสมอ** ทุก request โดยไม่ต้องทำอะไรเพิ่ม:

> "This function always provides context values including the following types: `Parts`, `ResponseOptions`, `ServerMetaContext`"

หมายความว่าใน server function หรือ component ฝั่ง server คุณเรียก `use_context::<leptos_axum::ResponseOptions>()` ได้เลยเพื่อควบคุม HTTP response ที่กำลังจะส่งกลับ (เช่นตั้ง status code หรือ header เพิ่มเติมจากภายใน server function — มีประโยชน์มากตอนต้องคืน 404/403 จาก logic ข้างในแทนการ throw error ธรรมดา) โดยไม่ต้องผ่าน `provide_context` มือเองเหมือนที่ทำกับ `PgPool` — Leptos จัดการ "สิ่งที่เกี่ยวกับ HTTP request/response ดิบ ๆ" ให้ทุกครั้งอัตโนมัติ ส่วน "สิ่งที่เป็นของแอปคุณเอง" (เช่น connection pool, config เฉพาะแอป) คุณต้อง `provide_context` เพิ่มเองผ่าน closure ที่สองของ `.leptos_routes_with_context(...)` ตามที่ทำในตัวอย่างข้างบน

#### อ้างอิง field สำคัญของ `LeptosOptions`

ตัวอย่างข้างบนสร้าง `LeptosOptions` ด้วย `.builder()` ตรง ๆ (สร้างมือ ไม่ผ่าน `cargo-leptos`) เพื่อความง่ายในการควบคุม แต่ในโปรเจกต์จริงที่ใช้ `cargo leptos new` scaffolding ค่าพวกนี้จะมาจาก section `[package.metadata.leptos]` ใน `Cargo.toml` แทน (`leptos_config` ออกแบบให้ deserialize จาก TOML section นี้โดยตรง ตามที่ระบุไว้ในซอร์สโค้ดจริงว่า struct นี้ "shares keys with cargo-leptos, to allow for easy interoperability") ตรวจสอบจากซอร์สโค้ดจริงของ `leptos_config-0.8.10/src/lib.rs` field ที่ใช้บ่อยที่สุดมีดังนี้ (ชื่อ key ใน TOML เป็น kebab-case ตามที่ `#[serde(rename_all = "kebab-case")]` กำหนด):

| Field (Rust) | Key ใน `Cargo.toml` | ความหมาย |
|---|---|---|
| `output_name` | `output-name` | ชื่อไฟล์ WASM/JS ที่ `wasm-bindgen` จะสร้าง (ต้องตรงกับชื่อที่ build จริงสร้างออกมา) |
| `site_root` | `site-root` | โฟลเดอร์ปลายทางที่ `cargo-leptos` เอาไฟล์ที่ build เสร็จแล้วไปวาง (default `.`) |
| `site_pkg_dir` | `site-pkg-dir` | โฟลเดอร์ย่อยที่เก็บไฟล์ WASM/JS จาก `wasm-bindgen` (default `pkg` — ตรงกับ convention ที่เห็นตอนใช้ `wasm-pack` เองในหัวข้อ 89.2) |
| `site_addr` | `site-addr` | address:port ที่ server ฟัง (default `127.0.0.1:3000`) |
| `env` | `env` | แยก dev/production (มีผลต่อ error message ที่ leptos แสดงเวลา panic — production ซ่อนรายละเอียดที่ dev โชว์เต็ม) |
| `reload_port` | `reload-port` | port ของ WebSocket ที่ใช้ทำ hot-reload ตอน `cargo leptos watch` (คนละกลไกกับ reactive signal — นี่คือ dev-tool ล้วน ๆ) |

ตัวอย่าง `[package.metadata.leptos]` ใน `Cargo.toml` ของโปรเจกต์ที่ผ่าน `cargo leptos new` (โครงสร้างตามที่ field ข้างบนกำหนด):

```toml
[package.metadata.leptos]
output-name = "my_leptos_app"
site-root = "target/site"
site-pkg-dir = "pkg"
site-addr = "127.0.0.1:3000"
env = "DEV"
```

#### แอปหลายหน้าด้วย `leptos_router`

ตัวอย่างในหัวข้อ 89.8 มีแค่หน้าเดียว (`generate_route_list(App)` จึงสร้าง route แค่ตัวเดียวคือ `/`) แอปจริงมักมีหลายหน้าและต้องมี URL แยกกัน (เช่น `/books` แสดงรายการ, `/books/:id` แสดงรายละเอียด) — `leptos_router` (crate แยกที่ผูกกับ Leptos โดยเฉพาะ) จัดการเรื่องนี้ ทดสอบเขียนจริงและ build เป็น WASM รันในเบราว์เซอร์จริง:

```rust
use leptos::prelude::*;
use leptos_router::components::*;
use leptos_router::path;
use leptos_router::hooks::use_params_map;

#[component]
fn BookList() -> impl IntoView {
    view! { <p>"รายการหนังสือทั้งหมด"</p> }
}

#[component]
fn BookDetail() -> impl IntoView {
    // use_params_map() อ่าน dynamic segment จาก URL ปัจจุบัน (":id" ในตัวอย่างนี้)
    let params = use_params_map();
    let id = move || params.read().get("id").unwrap_or_default();
    view! { <p>"หนังสือ id = " {id}</p> }
}

#[component]
fn App() -> impl IntoView {
    view! {
        // <Router> ครอบทั้งแอปเพื่อเปิดใช้ client-side navigation
        <Router>
            <main>
                // <Routes> นิยาม route ทั้งหมดพร้อม fallback สำหรับ path ที่ไม่ match
                <Routes fallback=|| "ไม่พบหน้านี้">
                    <Route path=path!("/books") view=BookList />
                    <Route path=path!("/books/:id") view=BookDetail />
                </Routes>
            </main>
        </Router>
    }
}
```

`path!("/books/:id")` คือ macro ที่ประกาศ path pattern พร้อม dynamic segment (`:id`) — ตรงกับ concept เดียวกับ path parameter ที่ Part 62-63 สอนไว้สำหรับ Axum (`/books/{id}`) เพียงแต่คนละ syntax เพราะเป็น router คนละตัวที่ทำงานคนละฝั่ง (ฝั่งนี้คือ client-side router ที่ทำงานใน WASM ไม่เกี่ยวกับ Axum router ฝั่ง server เลย — ทั้งสองระบบแค่บังเอิญมีแนวคิด "path parameter" คล้ายกัน) build เป็น WASM จริงแล้วเปิดสอง URL ด้วยเบราว์เซอร์จริง (ผ่าน static server ที่ทำ SPA fallback ให้ ส่ง `index.html` เดียวกันไม่ว่า path ไหน แล้วให้ `leptos_router` เป็นคนตัดสินใจว่าจะ render component ไหนจาก URL ที่เห็นฝั่ง client) ได้ผลลัพธ์จริงตรงตามที่ตั้งใจ:

```
at /books: รายการหนังสือทั้งหมด
at /books/42: หนังสือ id = 42
```

ข้อควรระวังเชิง infrastructure ที่พบระหว่างทดสอบจริง (เจอ error จริงตอนแรก): เว็บเซิร์ฟเวอร์ที่เสิร์ฟไฟล์ static ต้อง **ส่ง `index.html` กลับมาสำหรับทุก path ที่ไม่ตรงกับไฟล์จริง** (เรียกว่า SPA fallback) เพราะ `leptos_router` ใช้ HTML5 History API (`pushState`) ควบคุม URL โดยไม่ reload หน้าจริง — ถ้าผู้ใช้กด refresh หรือพิมพ์ URL `/books/42` ตรง ๆ ใน address bar โดยที่ server ไม่มี SPA fallback server จะตอบ `404 Not Found` เพราะไม่มีไฟล์ชื่อ `books/42` อยู่จริงในระบบไฟล์ (ระหว่างทดสอบเจอ error `net::ERR_CONNECTION_REFUSED`/`404` จริงจนกว่าจะเพิ่ม fallback ให้เว็บเซิร์ฟเวอร์ที่ใช้ทดสอบ) — นี่ไม่ใช่ปัญหาเฉพาะ Leptos แต่เป็นข้อกำหนดของ client-side routing ทุกแบบ (Yew ที่ใช้ `yew-router` ใน Part 88 ก็ต้องการ SPA fallback แบบเดียวกัน) ในโปรเจกต์ SSR (หัวข้อ 89.8) ปัญหานี้หายไปเองเพราะทุก path ที่ `generate_route_list` รู้จักจะมี handler ฝั่ง Axum ตอบให้ตรง ๆ อยู่แล้ว ไม่ต้องพึ่ง fallback แบบ static file server

#### บทพิสูจน์ครบวงจร: จากปุ่มที่ hydrate แล้ว ไปจนถึงแถวในฐานข้อมูลจริง

ตอนนี้มีทุกส่วนพร้อมแล้ว (SSR ส่ง HTML จริงจากหัวข้อนี้, hydration ที่จับมือกับ DOM เดิมจากหัวข้อ 89.7, server function ที่คุยกับฐานข้อมูลจากหัวข้อ 89.6) ลองรวมทั้งหมดเข้าด้วยกันเป็นครั้งเดียวเพื่อพิสูจน์คำกล่าวที่สำคัญที่สุดของบทนี้แบบครบวงจรจริง ๆ: **"เขียนฟังก์ชันเดียว เรียกจาก component เหมือนฟังก์ชัน local แต่จริง ๆ มันคุยกับฐานข้อมูลข้ามเครือข่ายให้เสร็จ"**

เพิ่มปุ่ม "จองที่นั่ง" ลงใน `App` (component เดียวกันที่ใช้ทั้ง SSR และ hydrate จากหัวข้อ 89.7) ผูกกับ `book_seat` (server function ตัวเดียวกับหัวข้อ 89.6) ผ่าน `Action`:

```rust
#[component]
pub fn App() -> impl IntoView {
    let (count, set_count) = signal(0);

    // Action ผูกกับ server function จริง (book_seat) — เรียกจาก event handler
    // เหมือนฟังก์ชัน async ธรรมดา ทั้งที่จริง ๆ มันยิง HTTP ไปเซิร์ฟเวอร์
    let book_action = Action::new(|_: &()| book_seat(1));

    view! {
        <h1>"Rust Conf 2026"</h1>
        <p id="count-display">"Count: " {count}</p>
        <button id="inc-btn" on:click=move |_| set_count.update(|n| *n += 1)>"increment"</button>
        <button id="book-btn" on:click=move |_| { book_action.dispatch(()); }>"จองที่นั่ง"</button>
        <p id="book-result">
            {move || match book_action.value().get() {
                Some(Ok(n)) => format!("จองสำเร็จ ตอนนี้จองไปแล้ว {n} ที่"),
                Some(Err(e)) => format!("จองไม่สำเร็จ: {e}"),
                None => "ยังไม่ได้กดจอง".to_string(),
            }}
        </p>
    }
}
```

build ทั้งสอง target อีกครั้ง (`cargo build --features ssr` และ `wasm-pack build --features hydrate`) รัน server แล้วเปิดหน้าด้วย Playwright จริง คลิกปุ่ม "จองที่นั่ง" หนึ่งครั้ง แล้วอ่านทั้งข้อความบนหน้าเว็บและแถวจริงในฐานข้อมูล:

```
before click: ยังไม่ได้กดจอง
after click: จองสำเร็จ ตอนนี้จองไปแล้ว 43 ที่
```

```sql
SELECT booked_seats FROM events WHERE id=1;
 booked_seats
--------------
           43
(1 row)
```

เส้นทางที่เกิดขึ้นจริงตอนคลิกปุ่มนี้ครบทุกขั้น: (1) เบราว์เซอร์ (WASM ที่ hydrate ไว้แล้ว) เรียก `book_action.dispatch(())` (2) โค้ดฝั่ง client ของ `book_seat` (คนละโค้ดกับที่เราเขียน — macro generate ให้ ตามที่อธิบายในหัวข้อ 89.6) ยิง HTTP POST ไปที่ `/api/book_seat` (3) Axum route ที่ `.leptos_routes_with_context` สร้างไว้รับ request นี้ (4) เรียก `book_seat` เวอร์ชันจริงฝั่ง server (โค้ดเดียวกันที่เราเขียนไว้ใน `lib.rs` แต่ compile ด้วย feature `ssr`) ซึ่งเปิด transaction จริงกับ PostgreSQL, ล็อกแถวด้วย `FOR UPDATE`, เช็คที่นั่งว่าง, `UPDATE`, และ `COMMIT` (5) ผลลัพธ์ (`43`) เดินทางกลับมาเป็น HTTP response (6) โค้ด client deserialize กลับมาเป็น `Result<i32, ServerFnError>` แล้วเขียนใส่ `book_action.value()` (signal ภายในของ `Action`) (7) `{move || match book_action.value().get() {...}}` ที่อ่าน signal นั้นถูกกระตุ้นให้รันใหม่ (fine-grained reactivity จากหัวข้อ 89.1) แล้วเขียนข้อความใหม่ลง DOM node เดิม —**ทั้งเจ็ดขั้นตอนนี้เกิดขึ้นจากการเขียนโค้ดแค่สองบรรทัดในฝั่ง component** (`Action::new(...)` กับ `.dispatch(())`) โดยไม่มีการเขียน fetch, ไม่มีการเขียน JSON serialize/deserialize มือ, ไม่มีการเขียน HTTP route แยกสำหรับ endpoint นี้เลยแม้แต่บรรทัดเดียว — นี่คือสิ่งที่หัวข้อ 89.6 อธิบายไว้ว่าเป็น "จุดขายหลักของ Leptos" ตอนนี้พิสูจน์แล้วว่าทำงานได้จริงครบทั้ง pipeline ตั้งแต่ปลายนิ้วผู้ใช้ไปจนถึงแถวในฐานข้อมูล

### 89.9 Islands Architecture: Partial Hydration (แนวคิดขั้นสูง)

หัวข้อ SSR ในหัวข้อ 89.7-89.8 มีข้อจำกัดหนึ่งที่ต้องพูดตรง ๆ: แม้ SSR จะทำให้ HTML แรกมาเร็ว แต่ **WASM bundle ทั้งก้อนยังต้องถูกส่งไปให้ client เพื่อ hydrate ทั้งหน้า** แม้หน้านั้นจะมีส่วน interactive อยู่นิดเดียว (เช่นปุ่มกดเดียวท่ามกลางเนื้อหา static เป็นพันบรรทัด) นี่คือปัญหาที่ **islands architecture** ถูกออกแบบมาแก้ — แนวคิดคือ: มีแค่ "island" (ส่วนที่ต้อง interactive จริง ๆ) เท่านั้นที่ compile เป็น WASM แล้วส่งไปให้ client ส่วนที่เหลือของหน้ายังคงเป็น static HTML ล้วน ๆ ไม่มี JS/WASM ห่อหุ้มเลย

Leptos รองรับแนวคิดนี้ผ่าน attribute macro `#[island]` ตรวจสอบจากเอกสารจริงในซอร์สโค้ด (`leptos_macro-0.8.18/src/lib.rs`) พบตัวอย่างที่ทีม Leptos เขียนไว้เอง (ปรับเล็กน้อยให้ตรงกับโดเมนของบทนี้):

```rust
use leptos::prelude::*;

#[component]
pub fn App() -> impl IntoView {
    // ฟังก์ชันนี้รันได้เฉพาะฝั่ง server เท่านั้น (จะ panic ถ้ารันในเบราว์เซอร์)
    // เพราะ App ไม่ใช่ island — มันเป็น "server component" ธรรมดา
    let file = std::fs::read_to_string("./data/seat_count.txt").unwrap();
    let seats: usize = file.trim().parse().unwrap();

    view! {
        <p>"จำนวนที่นั่งเริ่มต้นอ่านจากไฟล์บน server"</p>
        // ทั้ง <BookingIsland> จะถูก compile เป็น WASM ส่งไปให้ client
        // ส่วน <p> ข้างบนยังเป็น static HTML ล้วน ๆ ไม่มี WASM ห่อ
        <BookingIsland value=seats />
    }
}

#[island]
pub fn BookingIsland(
    #[prop(into)] value: RwSignal<usize>,
) -> impl IntoView {
    view! {
        <button on:click=move |_| value.update(|n| *n += 1)>
            "จอง (" {value} " ที่นั่งแล้ว)"
        </button>
    }
}
```

หลักการสำคัญตามที่เอกสารระบุไว้ตรง ๆ: **มีแค่โค้ดที่อยู่ข้างใน `#[island]` เท่านั้นที่ถูก compile เป็น WASM** ส่วนที่เหลือของ `App` (รวมถึง `std::fs::read_to_string` ที่จะ panic แน่นอนถ้ารันในเบราว์เซอร์) ไม่ถูกส่งไปฝั่ง client เลย — Props ที่ส่งจาก server component เข้าไปให้ island (`value=seats`) ถูก serialize มาเป็นค่าเริ่มต้นให้ island ใช้ตอนสร้าง ทำให้ island ยังรับข้อมูลจาก server ได้โดยไม่ต้องยิง server function เพิ่ม

ข้อจำกัดที่เอกสารเดียวกันระบุไว้ตรง ๆ (ไม่ปิดบัง): `children` ที่ส่งเข้า island จะถูกมองเป็น "opaque" ทั้งหมด (ถูกมัดรวมเป็น element เดียวชื่อ `<leptos-children>` ใน HTML) — คุณ**ไม่สามารถ iterate หรือแสดงแบบมีเงื่อนไข**ด้วย `<Show>`/control flow ปกติกับ children ของ island ได้ ถ้าต้องการซ่อน/แสดงแบบมีเงื่อนไขต้องใช้ CSS (`display: none`) เพราะ children ไม่ได้ถูก serialize เป็นข้อมูลจริง มันถูกส่งไปเป็น HTML ดิบเท่านั้น — ถ้า HTML นั้นไม่ปรากฏใน DOM เลย (ไม่ใช่แค่ซ่อนด้วย CSS) มันจะไม่ถูกส่งไปให้ client เลยด้วยซ้ำ

**บทนี้ให้แค่ความเข้าใจระดับแนวคิดสำหรับ islands** เพราะการตั้งโปรเจกต์ให้ mix `#[component]` (server-only) กับ `#[island]` (client+server) ในโปรเจกต์เดียวต้องพึ่งพา build pipeline ของ `cargo-leptos` แบบเต็มรูปแบบ ซึ่งซับซ้อนกว่าสโคปของบทนี้ที่โฟกัสที่ server function เป็นหลัก — ยืนยันได้ว่านี่เป็น feature จริงที่มีอยู่ใน crate (ไม่ใช่แค่แผนในอนาคต) จาก list feature ที่ `cargo add leptos` แสดงจริงตอนติดตั้งในหัวข้อ 89.2 ซึ่งมีทั้ง `islands` และ `islands-router` อยู่ในนั้นด้วย สิ่งที่ควรจำจากหัวข้อนี้คือ **islands คือคำตอบของ Leptos ต่อคำถาม "SSR ทำให้ HTML แรกมาเร็ว แต่จะลด WASM ที่ต้องส่งไปทั้งหน้าได้อย่างไร"** และมันคือทิศทางที่ full-stack framework รุ่นใหม่หลายตัว (ไม่จำกัดแค่ในโลก Rust) กำลังเดินไปทางเดียวกัน

### 89.10 Leptos vs Yew: เปรียบเทียบตรงไปตรงมา

ทั้ง Leptos และ Yew แก้ปัญหาเดียวกัน (เขียน frontend ด้วย Rust+WASM) แต่เลือกจุดยืนต่างกันมากในทุกมิติสำคัญ — สรุปเป็นตารางเพื่อให้เห็นภาพรวมก่อนไปเรียน Dioxus (Part 90) ซึ่งจะเป็นตัวเลือกที่สามที่ต้องเทียบด้วย (ตารางสรุปขั้นสุดท้ายทั้งสามตัวจะรอไว้หลัง Part 90 ตามแนวทางเดียวกับที่ Part 69 รอเทียบ Axum/Actix-web/Rocket ให้ครบสามตัวก่อนสรุป):

| มิติ | Yew (Part 88) | Leptos (บทนี้) |
|---|---|---|
| Rendering model | Virtual DOM + diffing (คล้าย React) | Fine-grained reactivity ผ่าน signal (คล้าย SolidJS) — ไม่มี diffing |
| การ re-render | รัน component function ใหม่ทั้งฟังก์ชันทุกครั้งที่ state เปลี่ยน | component function รันครั้งเดียวตอน mount — มีแค่ effect ที่เกี่ยวข้องรันซ้ำ |
| Syntax เขียน view | `html!` macro (คล้าย JSX) | `view!` macro (คล้าย JSX เช่นกัน แต่ compile เป็นโค้ดคนละแบบ) |
| State/props | `use_state`, `#[derive(Properties, PartialEq)]` | `signal()`, `#[component]` รับ parameter ตรง ๆ |
| Full-stack story | ไม่มีในตัว — ต้องเขียน REST API + fetch เอง (Part 62-70 style) | `#[server]` macro — เขียนฟังก์ชันเดียว ได้ทั้ง endpoint และ client call อัตโนมัติ |
| SSR maturity | มี SSR แต่ค่อนข้างพื้นฐาน ecosystem รอบ SSR ไม่ใหญ่เท่า | SSR + hydration + islands เป็นจุดสนใจหลักของทีมพัฒนา ecosystem รอบนี้ใหญ่กว่า |
| ความเก่า/ความอิ่มตัวของ ecosystem | เก่ากว่า (เริ่มก่อน) — คู่มือ/ตัวอย่าง third-party มากกว่า | ใหม่กว่า — API เปลี่ยนบ่อยกว่า (ตามที่เห็นเรื่อง `create_signal`→`signal()`) เอกสาร third-party น้อยกว่า |
| Learning curve | ใกล้เคียง React มากกว่า (คนที่มาจาก React เข้าใจเร็ว) | ใกล้เคียง SolidJS มากกว่า — mental model "component รันครั้งเดียว" ต้องปรับตัวถ้าคุ้นเคยกับ React มาก่อน |
| ความเสี่ยงเรื่อง breaking change | ต่ำกว่า (API เสถียรกว่า) | สูงกว่า (เพิ่งเจอ deprecation ของฟังก์ชันพื้นฐานอย่าง `create_signal` เมื่อไม่นาน) |
| การจัดการ error ข้าม client-server | ต้องออกแบบเองทั้งหมด (error code ฝั่ง server, parse error ฝั่ง client เอง) | มี `ServerFnError` กลางให้ครอบคลุม failure mode ของ "การเรียกข้าม network" โดยอัตโนมัติ (หัวข้อ 89.6) |
| การทดสอบ logic ฝั่ง "API" | ต้องทดสอบผ่าน HTTP client จริงเสมอ (endpoint คือของจริงที่แยกจากโค้ด Rust ธรรมดา) | เรียก server function ตรง ๆ ใน `#[tokio::test]` ได้เหมือนฟังก์ชันธรรมดา (หัวข้อ 89.6) เพราะมันเป็น async fn จริง ๆ ตอน compile ด้วย feature `ssr` |

ข้อสรุปที่ให้ได้ ณ จุดนี้ (ยังไม่ใช่ข้อสรุปสุดท้าย เพราะยังไม่ได้เห็น Dioxus): ถ้าโปรเจกต์ของคุณต้องการ **full-stack story ที่แน่นและ SSR/server function เป็นหัวใจของแอป** (เช่นแอปที่ต้อง query ฐานข้อมูลจากหลายหน้าตลอดเวลา และอยากลดงาน "เขียน API แยกจาก UI") Leptos ให้ประโยชน์ที่จับต้องได้จริงตามที่พิสูจน์ในหัวข้อ 89.6-89.8 แต่ต้องแลกกับ mental model ที่ต่างจาก React/Yew มากกว่า และ API ที่ยังเปลี่ยนได้เร็วกว่า ถ้าโปรเจกต์เป็น SPA ล้วน ๆ ที่ไม่ต้องพึ่ง SSR/server function เลย ความต่างระหว่างสองตัวจะแคบลงมาก และ Yew ที่เสถียรกว่าอาจเป็นตัวเลือกที่ปลอดภัยกว่าในเชิงความเสี่ยงระยะยาว — Part 90 จะนำ Dioxus เข้ามาเป็นตัวเลือกที่สาม (framework ที่พยายามผสมข้อดีของทั้งสองแนวทาง พร้อมเรื่อง cross-platform ที่ Yew/Leptos ไม่ได้โฟกัส) ก่อนที่บทเปรียบเทียบสุดท้ายจะสรุปทั้งสามตัวเข้าด้วยกัน

#### สรุป API ที่เจอในบทนี้ (Quick Reference)

ก่อนไปหัวข้อกับดักและแบบฝึกหัด สรุป API หลักที่บทนี้ใช้ไว้ในตารางเดียว เพื่อกลับมาเปิดดูได้เร็วตอนเขียนโค้ดจริง:

| API | มาจากไหน | ใช้ทำอะไร | หัวข้อ |
|---|---|---|---|
| `signal(v)` | `leptos::prelude` | สร้าง `(ReadSignal, WriteSignal)` คู่หนึ่ง | 89.3 |
| `.get()` / `.set()` / `.update()` | trait ของ `ReadSignal`/`WriteSignal` | อ่าน / เขียนทั้งค่า / แก้ไข in-place | 89.3 |
| `RwSignal::new(v)` | `leptos::prelude` | signal ที่รวม read+write ไว้ handle เดียว | 89.3 |
| `.get_untracked()` | trait `GetUntracked` | อ่านค่าโดยไม่ subscribe | 89.3 |
| `batch(\|\| { ... })` | `leptos::prelude` | รวมหลาย signal update ให้ trigger effect ครั้งเดียว | 89.3 |
| `Memo::new(f)` | `leptos::prelude` | ค่าที่ cache ไว้ recompute เฉพาะตอน dependency เปลี่ยน | 89.4 |
| `Effect::new(f)` | `leptos::prelude` | รัน side effect ตอน dependency เปลี่ยน (ไม่คืนค่าที่มีความหมาย) | 89.4 |
| `#[component]` | `leptos_macro` (re-export) | ประกาศฟังก์ชันเป็น Leptos component | 89.5 |
| `#[prop(optional/default/into)]` | `leptos_macro` | ปรับพฤติกรรมของ prop แต่ละตัว | 89.5 |
| `provide_context` / `use_context` | `leptos::prelude` | แชร์ค่าข้าม component tree โดยไม่ต้องผ่าน props | 89.5, 89.6, 89.8 |
| `#[server(...)]` | `leptos::prelude`/`leptos_macro` | ประกาศ server function — compile ต่างกันตาม feature | 89.6 |
| `ServerFnError` | `leptos::prelude` | error type กลางสำหรับ server function | 89.6 |
| `Action::new(f)` | `leptos::prelude` | trigger async call (มักเป็น server function) จาก event | 89.6, 89.8 |
| `Resource::new` / `LocalResource::new` | `leptos::prelude` | fetch ข้อมูลตอน component mount | 89.6 |
| `<Suspense fallback=...>` | `leptos::prelude` | แสดง fallback ระหว่าง resource ข้างในยังโหลดไม่เสร็จ | 89.6 |
| `mount_to_body` | `leptos::mount` | mount แอปแบบ CSR (สร้าง DOM ใหม่ทั้งหมด) | 89.2 |
| `hydrate_body` | `leptos::mount` | mount แอปแบบ hydrate (จับคู่กับ DOM เดิมจาก SSR) | 89.7 |
| `generate_route_list` / `LeptosRoutes` | `leptos_axum` | สแกน route จาก view tree แล้วผูกเข้า Axum `Router` | 89.8 |
| `<AutoReload>` / `<HydrationScripts>` | `leptos::hydration` | ฝัง script hot-reload/hydration ลงใน HTML shell | 89.7, 89.8 |
| `Router` / `Routes` / `Route` / `path!` | `leptos_router` | client-side routing หลายหน้าใน SPA | 89.8 |

## กับดักที่พบบ่อย (Common Pitfalls)

**1. เขียน `{count.get() * 2}` ตรง ๆ ใน `view!` โดยไม่ห่อด้วย closure — ค่าไม่อัปเดตเลย**

```rust
// ผิด — คำนวณครั้งเดียวตอน mount แล้วไม่อัปเดตอีก
view! { <p>{count.get() * 2}</p> }
```

ปุ่มกดเปลี่ยน `count` แล้ว UI ไม่ขยับ ไม่มี error ใด ๆ ตอน compile หรือ runtime (เพราะ syntax ถูกต้องทุกอย่าง) แต่ผลลัพธ์ผิดเงียบ ๆ — สาเหตุตามที่อธิบายในหัวข้อ 89.3: `view!` macro ต้องเห็น **closure ที่อ่าน signal ข้างใน** เพื่อสร้าง reactive effect ผูกกับจุดนั้น ถ้าเขียนแบบ `{count.get() * 2}` มันคือ **ค่า `i32` ที่คำนวณเสร็จแล้วครั้งเดียว** ก่อนที่ macro จะเห็น ไม่มี signal tracking เกิดขึ้น วิธีแก้คือห่อด้วย closure เสมอเมื่อค่าต้องเปลี่ยนตาม signal:

```rust
// ถูก
view! { <p>{move || count.get() * 2}</p> }
```

**2. ใช้ `create_signal`/`create_memo` แล้วเจอ warning จำนวนมากตอน build**

```
warning: use of deprecated function `leptos::prelude::create_signal`: This function is being renamed to `signal()` to conform to Rust idioms.
 --> src/lib.rs:4:31
  |
4 |     let (count, _set_count) = create_signal(0);
  |                               ^^^^^^^^^^^^^
```

โค้ดยัง compile และรันได้ปกติ (deprecation ไม่ใช่ error) แต่ตัวอย่าง/บทความเก่าจำนวนมากบนอินเทอร์เน็ตยังใช้ `create_signal`/`create_memo`/`create_rw_signal` อยู่ ทำให้สับสนว่า "อันไหนคือของปัจจุบัน" วิธีแก้คือเช็คซอร์สโค้ดจริงของ crate ที่ resolve ในเครื่องตัวเอง (`~/.cargo/registry/src/.../reactive_graph-*/src/signal.rs`) เสมอเมื่อไม่แน่ใจ แทนที่จะเชื่อบทความเก่า — และใช้ `signal()`, `Memo::new()`, `RwSignal::new()` เป็น API หลักตามที่บทนี้สอน

**3. ลืมว่า server function คือ HTTP endpoint จริง — เผลอส่งข้อมูลลับกลับไปในผลลัพธ์**

```rust
#[server]
pub async fn get_user_profile(user_id: i64) -> Result<UserProfile, ServerFnError> {
    let user = sqlx::query_as!(UserProfile, "SELECT * FROM users WHERE id = $1", user_id)
        .fetch_one(&pool).await?;
    Ok(user) // ถ้า UserProfile มี field password_hash อยู่ด้วย — รั่วออกไปเป็น JSON ทันที!
}
```

เพราะ `#[server]` แปลงฟังก์ชันเป็น HTTP endpoint จริงตามที่อธิบายในหัวข้อ 89.6 (`SELECT *` แล้ว map เข้า struct ที่มี field ที่ไม่ควรออกไปสู่ client) ทุก field ของ return type ที่ `Serialize` ได้จะถูกส่งกลับไปเป็น JSON จริง ไม่มีการกรองอัตโนมัติ ต่างจาก private field ในโค้ด Rust ทั่วไปที่ compiler ป้องกันการเข้าถึงข้ามโมดูล — วิธีแก้คือสร้าง struct แยกสำหรับ "สิ่งที่ปลอดภัยจะส่งออกไป" (DTO pattern) เสมอ อย่าใช้ struct ที่ map ตรงจากตารางฐานข้อมูลเป็น return type ของ server function โดยไม่ตรวจสอบ field ก่อน

**4. เรียก server function ตรง ๆ นอก reactive context แล้วสงสัยว่าทำไม UI ไม่รู้ผล**

```rust
// เรียกตรง ๆ แบบนี้ได้ผลลัพธ์จริง แต่ UI ไม่มีทางรู้ว่ามันเสร็จแล้ว
spawn_local(async move {
    let _ = book_seat(1).await;
    // ไม่มี signal ไหนถูกอัปเดต — UI ไม่ re-render จุดไหนเลย
});
```

server function เป็นแค่ async function ธรรมดา — มันไม่ผูกกับ reactive graph เองโดยอัตโนมัติ ถ้าคุณ `.await` มันแล้วไม่เอาผลลัพธ์ไปเขียนใส่ signal ไหนเลย UI จะไม่มีทางรู้ว่า operation เสร็จแล้ว วิธีที่ถูกต้อง (ตามหัวข้อ 89.6) คือใช้ `Action::new(...)` ที่ผูกผลลัพธ์เข้ากับ signal ภายในให้อัตโนมัติ (`action.value()` เป็น signal ที่อัปเดตเองเมื่อ action เสร็จ) หรือถ้าเรียกตรง ๆ ด้วย `spawn_local` ต้องเขียน signal ที่เกี่ยวข้องด้วยมือหลัง `.await` เสร็จ:

```rust
let (result, set_result) = signal(None);
spawn_local(async move {
    let r = book_seat(1).await;
    set_result.set(Some(r)); // ตอนนี้ UI ที่อ่าน result จะอัปเดตจริง
});
```

**5. Cargo.toml ระบุเวอร์ชัน `leptos` และ `leptos_axum`/`leptos_router` ไม่ตรงกัน — compile error ที่อ่านแล้วงงว่าเกิดอะไรขึ้น**

ระบุ `leptos = "0.7"` คู่กับ `leptos_axum = "0.8.10"` ในโปรเจกต์เดียวกัน (เผลอทำตามตัวอย่างเก่าที่ pin เวอร์ชันไว้ไม่ตรงกับที่เขียนในบทนี้) จะได้ compile error ที่ดูไม่เกี่ยวกับ "เวอร์ชันไม่ตรงกัน" เลยในตอนแรก — ทดสอบจริงด้วยการปรับ `Cargo.toml` ให้เวอร์ชันไม่ตรงกันแล้ว `cargo build` ได้ error จริงตามนี้:

```
error[E0308]: mismatched types
   --> src/main.rs:128:21
    |
128 |         .with_state(leptos_options);
    |          ---------- ^^^^^^^^^^^^^^ expected `leptos_config::LeptosOptions`, found `LeptosOptions`
    |
note: there are multiple different versions of crate `leptos_config` in the dependency graph
```

สาเหตุคือ `leptos = "0.7"` ดึง `leptos_config` เวอร์ชัน `0.7.8` เข้ามา ในขณะที่ `leptos_axum = "0.8.10"` ต้องการ `leptos_config` เวอร์ชัน `0.8.x` — Cargo ยอมให้ทั้งสองเวอร์ชันอยู่ใน dependency graph เดียวกันได้ (เพราะเป็น crate คนละ semver major) แต่ `LeptosOptions` ของทั้งสองเวอร์ชันเป็น**type คนละตัวกัน**ในมุมมองของ type system แม้จะหน้าตาเหมือนกันทุกอย่างก็ตาม — วิธีแก้คือตรวจสอบให้ `leptos`, `leptos_axum`, `leptos_router`, `leptos_meta` ทุกตัวในโปรเจกต์เดียวกันใช้เวอร์ชัน**สายเดียวกันเสมอ** (เช่นทั้งหมด `"0.8"`) วิธีที่ปลอดภัยที่สุดคือปล่อยให้ `cargo add` เลือกเวอร์ชันล่าสุดให้ทุกตัวพร้อมกันในครั้งเดียว แทนการ pin เวอร์ชันแต่ละตัวด้วยมือแยกกัน

**6. `Resource::new(...)` ไม่ compile เพราะ future ไม่ใช่ `Send` — โดยเฉพาะตอนทดสอบด้วย library ที่ผูกกับ JS**

```
error: future cannot be sent between threads safely
   |
12 |     let book_count = Resource::new(|| (), |_| fetch_book_count());
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ future returned by `fetch_book_count` is not `Send`
   |
note: future is not `Send` as it awaits another future which is not `Send`
   |
 6 |     gloo_timers::future::TimeoutFuture::new(300).await;
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here on type `TimeoutFuture`, which is not `Send`
```

(error นี้คือของจริงที่เจอตอนทดสอบตัวอย่าง `Resource`/`Suspense` ในหัวข้อ 89.6 ก่อนแก้ไข) `Resource::new` กำหนด bound ว่า future ที่ fetcher คืนมาต้อง `Send` (เพราะออกแบบมาให้ทำงานได้ทั้งฝั่ง server ที่เป็น multi-thread runtime อย่าง Tokio ด้วย) แต่หลาย JS API ที่ผูกผ่าน `wasm-bindgen` (เช่น `gloo_timers::future::TimeoutFuture` ที่ผูกกับ browser's `setTimeout`) ไม่ใช่ `Send` เพราะ WASM ในเบราว์เซอร์เป็น single-threaded และ JS value ที่ห่อมาไม่มีการันตีเรื่อง thread safety แบบ Rust ต้องการ — วิธีแก้คือใช้ **`LocalResource::new(...)`** แทน (bound เดียวกันแต่ไม่ต้องการ `Send`) เหมาะสำหรับ resource ที่ fetcher เรียก JS API ตรง ๆ ฝั่ง client ส่วน resource ที่ fetcher เป็นการเรียก server function ผ่าน HTTP (ซึ่งเป็น `Send` ได้ตามปกติ) ยังใช้ `Resource::new` ตรง ๆ ได้ไม่มีปัญหา

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** สร้าง component `TemperatureConverter` ที่มี signal `celsius: f64` (ตั้งต้นที่ 0.0) พร้อม input field ที่ผูกกับ signal นั้น (`on:input`) และแสดงค่า Fahrenheit ที่คำนวณจาก `celsius` ผ่าน closure (`{move || celsius.get() * 9.0 / 5.0 + 32.0}`) — Hint: `event_target_value(&ev)` คือฟังก์ชันช่วยของ Leptos สำหรับอ่านค่าจาก `<input>` event แล้ว parse เป็น `f64` ด้วย `.parse().unwrap_or(0.0)`

2. **(กลาง)** เพิ่ม `Memo` ชื่อ `is_freezing` ที่คำนวณจาก `celsius` (คืนค่า `true` ถ้า `celsius.get() <= 0.0`) ใส่ log ข้างในเหมือนตัวอย่างหัวข้อ 89.4 แล้วทดสอบ (ด้วยมือในเบราว์เซอร์จริง เปิด DevTools Console) ว่าถ้าเพิ่ม signal อีกตัวที่ไม่เกี่ยวข้อง (เช่น `city_name: String`) แล้วเปลี่ยนค่ามัน `is_freezing` ไม่ควร recompute — Hint: ทำตามรูปแบบการทดสอบใน 89.4 ทุกประการ แค่เปลี่ยนโดเมน

3. **(ยาก)** เขียน server function คู่ `#[server]` สองตัวสำหรับระบบยืมหนังสือ (ต่อยอด Part 70): `get_book(book_id: i64) -> Result<Book, ServerFnError>` (query) และ `borrow_book(book_id: i64) -> Result<i32, ServerFnError>` (mutation ที่ใช้ transaction ลด `available_copies` ทีละ 1 พร้อม insert แถวใน `borrow_records` — ต้องสำเร็จทั้งคู่หรือ rollback ทั้งคู่) ทดสอบด้วย `curl` จริงทั้ง success path และ error path (ยืมตอนที่ `available_copies = 0`) — Hint: โครงสร้างตารางและ pattern transaction อ้างจาก Part 70 ได้ตรง ๆ เปลี่ยนแค่ชนิดของ error message ที่ห่อด้วย `ServerFnError::new(...)`

4. **(ยาก/ประยุกต์)** สร้างหน้า SSR เต็มรูปแบบที่มี route `/books` แสดงรายการหนังสือทั้งหมด (query ผ่าน server function ตอน component โหลดด้วย `Resource::new`) ผสานกับ route ธรรมดาของ Axum ที่ชื่อ `/api/stats` (คืนค่า `{"total_books": N}` แบบ JSON ธรรมดาไม่ผ่าน Leptos) ใน `Router` เดียวกันตามที่หัวข้อ 89.8 สอน แล้วพิสูจน์ด้วย `curl` สองคำสั่งว่าทั้งสอง route ตอบถูกต้องจากเซิร์ฟเวอร์ตัวเดียวกัน — Hint: `Resource::new(|| (), |_| get_all_books())` คือรูปแบบมาตรฐานสำหรับ fetch ข้อมูลตอน component mount ใน Leptos (คล้าย `useEffect` + fetch ของ React แต่ผูกกับ reactive graph โดยตรง)

5. **(ยากมาก/ประยุกต์เต็มรูปแบบ)** ทำตามหัวข้อ 89.7-89.8 ให้ครบ: ตั้งโปรเจกต์ที่ compile ได้ทั้ง feature `ssr` (native binary, รัน Axum server) และ feature `hydrate` (WASM ผ่าน `wasm-pack build --target web --features hydrate`) จากซอร์สโค้ด `App` component ชุดเดียวกัน แล้วเปิดหน้าเว็บด้วยเบราว์เซอร์จริง (Chromium headless ผ่าน Playwright หรือเบราว์เซอร์ปกติก็ได้) เขียนสคริปต์ทดสอบที่ (ก) จับ DOM node reference ของ element หนึ่งตัว **ก่อน** WASM โหลดเสร็จ (ข) รอให้ hydrate เสร็จแล้วคลิกปุ่มที่ผูกกับ signal (ค) เทียบ node reference เดิมกับที่อ่านได้หลัง hydrate ด้วย `===` — ถ้าทำถูก ค่าที่ได้ต้องเป็น `true` เสมอ (ตามที่พิสูจน์ไว้จริงในหัวข้อ 89.7) — Hint: จุดที่มักพลาดคือลืมเสิร์ฟไฟล์ `pkg/*.js`/`pkg/*.wasm` เป็น static file จาก Axum (ต้องมี route หรือ `ServeDir` ชี้ไปที่โฟลเดอร์ `pkg/` ให้ตรงกับ path ที่ `<HydrationScripts>` generate ไว้ใน HTML)

## สรุป

**เช็คลิสต์ก่อนขึ้น production** (รวมข้อควรระวังจากทั้งบทที่มักถูกมองข้ามตอน deploy จริง แต่ละข้ออ้างอิงหัวข้อที่อธิบายรายละเอียดไว้แล้ว):

- [ ] build ด้วย `--release` ทั้งฝั่ง `ssr` (native binary) และ `hydrate`/`csr` (WASM) เสมอ — วัดจริงในหัวข้อ 89.2 ว่าขนาด `.wasm` ต่างกันมากกว่าครึ่งหนึ่งระหว่าง debug กับ release
- [ ] ตรวจให้ `leptos`, `leptos_axum`, `leptos_router`, `leptos_meta` (ถ้าใช้) เป็นเวอร์ชัน**สายเดียวกันเสมอ** — ไม่ตรงกันจะได้ compile error ที่อ่านยากตามที่พิสูจน์ไว้ในกับดักข้อ 5
- [ ] เว็บเซิร์ฟเวอร์/reverse proxy ที่วางไว้หน้า static asset ต้องมี **SPA fallback** ถ้าใช้ `leptos_router` แบบ CSR ล้วน ๆ (ไม่จำเป็นถ้าเป็น SSR เพราะ Axum ตอบทุก route ที่รู้จักเองอยู่แล้ว — ตามที่อธิบายในหัวข้อ 89.8)
- [ ] path ของไฟล์ WASM/JS ที่ `<HydrationScripts>` generate ต้องตรงกับที่ web server เสิร์ฟจริง (`site_pkg_dir`/`output_name` ต้องสอดคล้องกับผลลัพธ์ของ `wasm-bindgen`) — พิสูจน์การต่อกันถูกต้องในหัวข้อ 89.7 ด้วยการรันจริง
- [ ] ตรวจสอบ struct ที่ server function คืนค่าว่าไม่มี field ที่ควรอยู่ฝั่ง server เท่านั้นหลุดออกไป (เช่น password hash) ตามที่เตือนไว้ในกับดักข้อ 3 — เพราะ server function คือ public HTTP endpoint จริง ไม่มีการกรองอัตโนมัติ
- [ ] ถ้าใช้ transaction ในการ mutate ข้อมูล (เช่น `book_seat`) ตรวจให้แน่ใจว่า error path ทุกจุด `return Err(...)` เกิด**ก่อน** `tx.commit()` เสมอ เพื่อให้ rollback อัตโนมัติทำงานถูกต้องตามที่ Part 70 สอนไว้

บทนี้พาไปรู้จัก Leptos ในฐานะ framework ที่ตั้งใจต่างจาก Yew (Part 88) ในระดับสถาปัตยกรรม ไม่ใช่แค่ syntax: **fine-grained reactivity** ทำให้ signal ที่เปลี่ยนค่าไปอัปเดตเฉพาะ DOM node ที่เกี่ยวข้องโดยตรง ไม่มีขั้นตอน diffing เลย (พิสูจน์แล้วด้วยทั้งซอร์สโค้ดจริงของ `tachys` และการทดสอบ DOM node identity จริงในเบราว์เซอร์) API ปัจจุบัน (`signal()`, `Memo::new()`) มาแทน `create_signal()`/`create_memo()` รุ่นเก่าที่ deprecate ไปแล้วเพื่อให้สอดคล้องกับธรรมเนียม Rust มากขึ้น และที่สำคัญที่สุด — **server function** (`#[server]`) คือจุดขายที่ทำให้ Leptos เป็น "full-stack" อย่างแท้จริง: เขียนฟังก์ชันเดียวที่คุยกับฐานข้อมูลผ่าน SQLx (Part 70) ตรง ๆ แล้ว Leptos generate ทั้ง HTTP endpoint ฝั่ง server และโค้ด fetch ฝั่ง client ให้อัตโนมัติ ทั้งหมดนี้รันได้จริงภายใน `axum::Router` เดียวกับที่เรียนมาตั้งแต่ Module 4 — เห็นได้จากตัวอย่าง SSR+Axum+SQLx ที่ทดสอบด้วย `curl` จริงทั้ง query, mutation, และ error path และพิสูจน์ครบวงจรที่สุดในหัวข้อ 89.8 ที่คลิกปุ่มบนหน้าที่ hydrate แล้วในเบราว์เซอร์จริง แล้วเห็นแถวในฐานข้อมูล PostgreSQL เปลี่ยนค่าจริงตามไปด้วย โดยไม่มีการเขียนโค้ด fetch/JSON มือแม้แต่บรรทัดเดียว

หัวข้อ SSR/hydration ในบทนี้ก็ไม่ใช่แค่คำอธิบายเชิงทฤษฎีเช่นกัน — พิสูจน์แล้วด้วยการ build โปรเจกต์เดียวกันเป็นสอง target จริง (`ssr` สำหรับ native binary, `hydrate` สำหรับ WASM) แล้วเทียบ DOM node reference ก่อน/หลัง hydration ด้วย `===` จริงในเบราว์เซอร์ ยืนยันว่า WASM ที่โหลดมาไม่ได้ทำลาย HTML ที่ server ส่งมาแล้วสร้างใหม่ แต่ "จับมือ" กับ node เดิมให้กลายเป็น interactive เท่านั้น — แต่นี่เป็นเพียงการแนะนำแนวคิด — **Part 90 (Dioxus Framework เบื้องต้น)** จะพาไปรู้จักตัวเลือกที่สามในโลก Rust+WASM frontend ซึ่งพยายามผสานจุดแข็งของทั้งสองแนวทางเข้าด้วยกัน พร้อมมุมมองเรื่อง cross-platform ที่ Yew และ Leptos ไม่ได้โฟกัสเป็นหลัก ก่อนที่จะสรุปเปรียบเทียบทั้งสาม framework เข้าด้วยกันในบทถัดไปหลังจากนั้น

---

**Part ก่อนหน้า:** [Yew Framework: Frontend ด้วย Rust](part-088-yew-framework.md) | **Part ถัดไป:** [Dioxus Framework เบื้องต้น](part-090-dioxus-framework.md)
