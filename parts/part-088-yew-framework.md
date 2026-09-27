# Part 88: Yew Framework: Frontend ด้วย Rust

> โมดูล: Full-Stack และ WebAssembly | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างชัดเจนว่า **Yew** คือ layer ไหนของ frontend stack ใน Rust — มันไม่ได้แทนที่
  `wasm-bindgen`/`web-sys` ที่ Part 87 สอนไว้ แต่**สร้างอยู่บน**สิ่งเหล่านั้น เพื่อให้คุณเขียน UI แบบ
  declarative/component-based คล้าย React ได้ แทนที่จะต้อง `document.create_element()` เองทุกจุดแบบที่
  Part 87 ทำ — เข้าใจ "ทำไม" framework แบบนี้ต้องมีอยู่ ไม่ใช่แค่ "ทำอย่างไร"
- ติดตั้งเครื่องมือที่จำเป็นสำหรับพัฒนาแอป Yew ด้วย `trunk` (build tool มาตรฐานของ Yew) และเขียน component
  "Hello World" ตัวแรกด้วย macro `html!` ได้จริง แล้ว build+serve ออกมาเป็นหน้าเว็บที่รันได้จริงในเบราว์เซอร์
- สร้าง component ที่รับ **props** ผ่าน `#[derive(Properties, PartialEq)]` และ `#[function_component]`
  ตามแนวทางที่ทั้งวงการ frontend สมัยใหม่ใช้ (component + props คือหน่วยพื้นฐานของ React/Vue/Svelte/Yew
  ทั้งหมด) — สร้าง component จริงสำหรับระบบห้องสมุด (`BookCard`) ที่ render ข้อมูลหนังสือได้จริง
- จัดการ **state ภายใน component** ด้วย hook `use_state` และทำงานกับ side effect ด้วย `use_effect`/
  `use_effect_with` — สร้างระบบค้นหา/กรองรายการหนังสือ (search/filter) ที่ re-render เมื่อ state เปลี่ยนได้จริง
- เชื่อมต่อ event ของ DOM (`onclick`, `oninput`) เข้ากับ state ผ่าน `Callback<T>` — และส่งต่อ callback ลงไปเป็น
  prop เพื่อให้ **child component สื่อสารกลับขึ้นไปยัง parent** ได้ (pattern "lifting state up") รวมถึงรู้จัก
  `use_context` สำหรับกรณีที่ต้นไม้ component ลึกเกินกว่าจะส่ง prop ทีละชั้น
- ดึงข้อมูลจริงจาก REST API ที่เขียนด้วย Axum (โมดูล 4 ที่คุณเรียนมาแล้ว) ผ่าน `gloo-net` ภายใน
  `use_effect_with`, และสร้างแอปหลายหน้า (list + detail) ด้วย `yew-router` — พร้อมเข้าใจตำแหน่งของ Yew เทียบ
  กับ `wasm-bindgen` ดิบ ๆ และ framework อื่นที่กำลังจะเรียนต่อไป (Leptos ใน Part 89, Dioxus ใน Part 90)

## ความรู้ที่ต้องมีมาก่อน

- **Part 87 (wasm-bindgen และ JavaScript Interop)**: นี่คือ Part ที่บทนี้พึ่งพาโดยตรงที่สุด — Yew ไม่ได้เขียน
  binding เข้า JavaScript engine ขึ้นมาใหม่เอง มันใช้ `wasm-bindgen` และ `web-sys` (ตัวเดียวกับที่ Part 87
  สอน) เป็นรากฐานทั้งหมด ทุกครั้งที่ Yew "แก้ DOM" เบื้องหลังคือการเรียก `web_sys::Element::set_attribute()`,
  `Node::append_child()` ฯลฯ ที่ Part 87 สอนให้คุณเรียกเองมาแล้ว ถ้ายังไม่ผ่าน Part 87 บทนี้จะเหมือน "มายากล"
  เพราะจะไม่รู้ว่า Yew ซ่อนงานอะไรไว้ข้างหลัง — โดยเฉพาะเรื่อง `#[wasm_bindgen(start)]`, การ compile เป็น
  `wasm32-unknown-unknown` target, และวิธีที่ WASM module คุยกับ JS glue code
- **Part 86 (WebAssembly เบื้องต้นด้วย Rust)**: ความเข้าใจพื้นฐานว่า WASM คืออะไร ทำไม browser รันมันได้ และ
  ทำไม Rust ต้อง compile ไปเป็น target พิเศษ (`wasm32-unknown-unknown`) แทนที่จะรันแบบเดียวกับโปรแกรม CLI
  ทั่วไปที่ผ่านมาทั้งหมดในคอร์สนี้
- **Part 36 (Macros: Declarative Macros)** และ **Part 44-45 (Proc Macros)**: macro `html!` ที่เป็นหัวใจของ
  Yew นั้นหน้าตาคล้าย JSX ของ React แต่มันคือ **procedural macro** ตัวจริง (คล้ายกับที่ Part 44-45 สอน ไม่ใช่
  `macro_rules!` แบบ Part 36) ที่ประมวลผลตอน compile time เป็น Rust code ธรรมดา — ถ้าเข้าใจว่า macro ใน Rust
  ทำงานยังไง (ไม่ใช่ template engine ที่ทำงานตอน runtime) จะเข้าใจว่าทำไม `html!` ให้ compiler error ที่ชี้
  ตำแหน่งผิดใน markup ได้แม่นยำ และทำไมมันไม่ใช่ "string templating" แบบที่ภาษาอื่นทำ
- **Part 25-26 (Iterators พื้นฐานและขั้นสูง)**: การ render list ของ component (เช่น รายการหนังสือทั้งหมด) ใน
  Yew ใช้ `.iter().map(...)` ธรรมดาเป๊ะ ๆ ไม่มี syntax พิเศษเพิ่ม — ถ้า `map`, `collect::<Html>()`, และ
  iterator chain ยังไม่คล่อง หัวข้อ 88.6 จะอ่านยาก
- **Part 57-58 (Serde เบื้องต้นและขั้นสูง)**: ข้อมูลหนังสือที่ดึงจาก API จะถูก deserialize ด้วย
  `#[derive(Deserialize)]` แบบเดียวกับที่ Part 57 สอนตรง ๆ ไม่มีอะไรพิเศษเพิ่มจาก Yew
- **Part 62-66 (Axum พื้นฐานถึง Error Handling)**: หัวข้อ 88.8 (data fetching) จะสร้างแอป Yew ที่ยิง request
  ไปยัง REST API ที่เขียนด้วย Axum จริง — ใช้ pattern `Router`, `Json<T>`, `State` ตรงตามที่โมดูล 4 สอนมาทั้งหมด
  ถ้าลืมพื้นฐาน Axum ให้ย้อนไปทวน Part 62-64 ก่อน
- **Part 24 (Closures)**: `Callback<T>` ของ Yew และ callback ที่ส่งเข้า hook ต่าง ๆ ล้วนสร้างจาก closure — ความ
  เข้าใจเรื่อง `move ||`, การ capture ตัวแปร, และทำไมต้อง `.clone()` ก่อน `move` เข้า closure จะถูกใช้ซ้ำ ๆ
  ตลอดบทนี้

## เนื้อหา

### 88.1 ทำไมต้องมี Yew ทั้งที่มี wasm-bindgen อยู่แล้ว

Part 87 สอนให้คุณคุยกับ DOM ได้ตรง ๆ ผ่าน `web-sys`: หา element ด้วย `document.get_element_by_id()`, สร้าง
element ใหม่ด้วย `document.create_element()`, ตั้งค่า attribute/text ด้วย `set_attribute()`/`set_text_content()`,
ผูก event ด้วย `add_event_listener_with_callback()` แล้วห่อ closure ด้วย `Closure::wrap()` ก่อนส่งเข้า
JavaScript ผ่าน `.as_ref().unchecked_ref()` วิธีนี้**ทำงานได้จริง 100%** — ไม่มีอะไรผิด — แต่ลองนึกภาพแอปที่มี
ปฏิสัมพันธ์มากกว่า "ปุ่มกดหนึ่งปุ่ม" เยอะ ๆ เช่น รายการหนังสือ 50 เล่มที่กรองด้วยช่องค้นหา แต่ละเล่มมีปุ่ม "ยืม"
ของตัวเอง แล้วต้องอัปเดตสถานะ "ว่าง/ถูกยืมแล้ว" ของทุกเล่มพร้อมกัน — โค้ดแบบ Part 87 จะกลายเป็นการไล่จัดการ
DOM node เป็นสิบ ๆ จุดด้วยมือ ทุกจุดต้องจำเองว่า "ตอนนี้ DOM หน้าตาเป็นยังไง" แล้วคำนวณเองว่า "ต้องแก้ node
ไหนบ้าง" — นี่คือปัญหาเดียวกันเป๊ะกับที่ทำให้วงการ JavaScript สร้าง React/Vue/Svelte ขึ้นมาในปี 2013-2016:
เมื่อแอปโตขึ้น การจัดการ DOM แบบ **imperative** (บอกทีละคำสั่งว่า "ทำอะไร") กลายเป็นเรื่องซับซ้อนเกินจัดการ
เพราะ state กับ DOM สามารถ "หลุดจากกัน" ได้ง่าย (คุณแก้ state แล้วลืมแก้ DOM ให้ตรง หรือแก้ DOM ผิดจุด)

**Yew** คือคำตอบของ Rust ต่อปัญหานี้ — มันคือ **component-based UI framework** ที่ยืม mental model มาจาก
React ตรง ๆ (ทีมผู้พัฒนา Yew พูดถึงเรื่องนี้อย่างเปิดเผยในเอกสารของโครงการเอง): คุณอธิบาย UI แบบ
**declarative** ("หน้าตาควรเป็นแบบนี้ ถ้า state เป็นแบบนี้") แทนที่จะบอกทีละคำสั่งว่า "ทำอะไรกับ DOM" แล้ว Yew
จะรับผิดชอบคำนวณเองว่า DOM ปัจจุบันต่างจาก DOM ที่ควรจะเป็นตรงไหน แล้วแก้เฉพาะจุดที่ต่างเท่านั้น (กระบวนการนี้
เรียกว่า **virtual DOM diffing** — สร้าง "แผนผัง" DOM จำลองในหน่วยความจำสองชุด เทียบกันบิตต่อบิต แล้วสั่งแก้
DOM จริงเฉพาะส่วนต่าง) กระบวนการนี้ไม่ใช่เวทมนตร์ — มันคือโค้ด Rust ที่เขียนอยู่ *เหนือ* `web-sys` เป๊ะ ๆ
สิ่งที่ Yew ทำคือเขียนโค้ด imperative แบบ Part 87 ให้คุณโดยอัตโนมัติ จากคำอธิบาย declarative ที่คุณเขียน

#### เทียบให้เห็นภาพ: UI เดียวกัน เขียนสองแบบ

ลองสร้าง UI เล็ก ๆ ที่ทุกคนคุ้นเคย: ปุ่มนับจำนวนคลิก (counter) พร้อมข้อความแสดงจำนวนครั้งที่กด นี่คือแบบที่
Part 87 สอน (imperative, ใช้ `web-sys` ตรง ๆ):

```rust
// แบบ Part 87: imperative DOM manipulation ด้วย wasm-bindgen/web-sys ตรง ๆ
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{Document, Element, HtmlElement};
use std::cell::Cell;
use std::rc::Rc;

#[wasm_bindgen(start)]
pub fn run() -> Result<(), JsValue> {
    let document: Document = web_sys::window().unwrap().document().unwrap();
    let body = document.body().unwrap();

    // 1. สร้าง element ที่แสดงจำนวนครั้ง — ต้องสร้างเองทีละ node
    let display: Element = document.create_element("p")?;
    display.set_text_content(Some("จำนวนครั้งที่กด: 0"));
    body.append_child(&display)?;

    // 2. สร้างปุ่ม — ต้องสร้างเองอีกทีละ node
    let button: HtmlElement = document
        .create_element("button")?
        .dyn_into::<HtmlElement>()?;
    button.set_text_content(Some("กด!"));
    body.append_child(&button)?;

    // 3. state ต้องเก็บเองด้วย Rc<Cell<T>> เพราะ closure จะถูกเรียกซ้ำหลายครั้ง
    let count = Rc::new(Cell::new(0_i32));
    let count_clone = count.clone();
    let display_clone = display.clone();

    // 4. เขียน closure ที่ "รู้เอง" ว่าต้องแก้ DOM node ไหนตอนไหน — นี่คือส่วนที่ปวดหัวขึ้นเรื่อย ๆ
    //    เมื่อ UI มีหลาย element ที่ต้องอัปเดตพร้อมกัน
    let on_click = Closure::<dyn FnMut()>::new(move || {
        let new_value = count_clone.get() + 1;
        count_clone.set(new_value);
        // ต้องคำนวณ string ใหม่เอง แล้วยัดเข้า DOM เอง ทุกครั้งที่ state เปลี่ยน
        display_clone.set_text_content(Some(&format!("จำนวนครั้งที่กด: {new_value}")));
    });
    button.add_event_listener_with_callback("click", on_click.as_ref().unchecked_ref())?;
    on_click.forget(); // ทวนจาก Part 87: ต้อง .forget() ไม่ให้ closure ถูก drop ก่อนเวลา

    Ok(())
}
```

โค้ดนี้ **ถูกต้องทุกประการ** และเป็นสิ่งที่ Part 87 สอนให้เขียนได้ แต่สังเกตปัญหา 3 จุด: (1) state (`count`) กับ
DOM (`display`) เป็นคนละตัวแปรที่ต้อง sync มือทุกครั้ง — ถ้าคุณเพิ่ม element ที่สองที่ต้องแสดงค่าเดียวกัน (เช่น
badge เล็ก ๆ มุมขวาบน) ต้องมาแก้ closure นี้เพิ่ม logic sync อีกจุด (2) `Rc<Cell<...>>` และ `.clone()` ของทุก
element ที่ closure ต้องแก้ กลายเป็น boilerplate ที่โตตามจำนวน element เชิงเส้น (3) โครงสร้างของ UI (ลำดับ
node, nesting) กระจายอยู่ในโค้ดหลายบรรทัดที่ไม่ได้เรียงต่อกันเป็นภาพเดียว ต้องอ่านทั้งฟังก์ชันเพื่อจะรู้ว่า
"สุดท้ายหน้าตา DOM เป็นยังไง"

เทียบกับแบบเดียวกันที่เขียนด้วย Yew:

```rust
use yew::prelude::*;

#[function_component(Counter)]
fn counter() -> Html {
    // state ประกาศเป็นค่าเดียว ไม่ต้อง Rc<Cell<>> เอง — Yew จัดการ interior mutability ให้
    let count = use_state(|| 0_i32);

    // callback แค่ "บอกว่า state ควรเปลี่ยนเป็นอะไร" — ไม่ต้องรู้ว่า DOM node ไหนต้องแก้
    let onclick = {
        let count = count.clone();
        Callback::from(move |_| count.set(*count + 1))
    };

    // โครงสร้าง UI ทั้งหมดอยู่รวมกันเป็นภาพเดียว อ่านแล้วเห็นทันทีว่าหน้าตาเป็นยังไง
    html! {
        <div>
            <p>{ format!("จำนวนครั้งที่กด: {}", *count) }</p>
            <button {onclick}>{ "กด!" }</button>
        </div>
    }
}

fn main() {
    yew::Renderer::<Counter>::new().render();
}
```

โค้ดสองชุดนี้ **compile ไปเป็น WASM ที่ทำสิ่งเดียวกันในเบราว์เซอร์ทุกประการ** — ไม่มีเวทมนตร์ ไม่มีการข้าม
`web-sys` ไปทางลัด Yew เพียงแค่**เขียนโค้ด imperative แบบชุดแรกให้คุณโดยอัตโนมัติ** จากคำอธิบาย declarative
ในชุดที่สอง เวลา `count.set(*count + 1)` ถูกเรียก สิ่งที่เกิดขึ้นเบื้องหลังคือ Yew เรียก `counter()` ใหม่อีก
รอบ ได้ `Html` tree ใหม่ (แทนค่า `count` ใหม่) แล้วเทียบกับ `Html` tree เก่า (virtual DOM diffing) พบว่ามีแค่
text content ของ `<p>` ที่เปลี่ยน จึงเรียก `web_sys::Node::set_node_value()` (ฟังก์ชันระดับเดียวกับที่ Part 87
สอน) แก้เฉพาะจุดนั้นจุดเดียว ไม่ได้ทำลาย DOM ทั้งต้นแล้วสร้างใหม่ทั้งหมด

#### ตารางเทียบความรับผิดชอบระหว่างสองแนวทาง

| ประเด็น | wasm-bindgen/web-sys ตรง ๆ (Part 87) | Yew (บทนี้) |
|---|---|---|
| วิธีอธิบาย UI | Imperative — บอกทีละคำสั่งว่าทำอะไรกับ DOM | Declarative — บอกว่า UI ควรมีหน้าตาแบบไหนถ้า state เป็นแบบนี้ |
| การ sync state กับ DOM | คุณทำเอง (ต้องจำเองว่า DOM จุดไหน sync กับ state ตัวไหน) | Yew ทำให้ผ่าน virtual DOM diffing อัตโนมัติ |
| หน่วยจัดระเบียบโค้ด | ฟังก์ชัน/closure ทั่วไป | Component (function ที่ return `Html`) + props ที่ type-safe |
| การ reuse UI ส่วนย่อย | ต้องเขียนฟังก์ชันสร้าง element เองแล้วต่อกันมือ | Component ซ้อน component ได้ตรง ๆ เหมือน React |
| ระดับควบคุม DOM | เต็มที่ 100% (เข้าถึง `web-sys` ทุก API) | ผ่าน abstraction ของ Yew (แต่ยังเรียก `web-sys` ตรงได้เมื่อจำเป็น) |
| ค่าใช้จ่ายที่เพิ่ม | ไม่มี — บาง (thin) ที่สุดเท่าที่จะเป็นไปได้ | ต้อง diff virtual DOM ทุกครั้งที่ re-render (ค่าใช้จ่ายเล็กน้อย) |
| เหมาะกับ | widget เล็ก ๆ ฝังใน HTML/JS ที่มีอยู่แล้ว, ปะ WASM module เข้าโปรเจกต์เดิม | สร้างแอปทั้งหน้า (SPA) ที่มี state/ปฏิสัมพันธ์ซับซ้อน |

จุดสำคัญที่ต้องเข้าใจให้แน่นคือ **Yew ไม่ได้ "แทนที่" Part 87** — มันคือชั้นบนของสิ่งที่ Part 87 สอน พิสูจน์ได้
ด้วยคำสั่งเดียวกับที่ Part 62 (Axum) ใช้พิสูจน์ว่า Axum สร้างอยู่บน `hyper`/`tower`:

```bash
cargo tree -p yew
```

ตัดมาเฉพาะส่วนที่เกี่ยวข้อง (dependency tree เต็มยาวกว่านี้มาก):

```
yew v0.23.0
├── wasm-bindgen v0.2.129
├── wasm-bindgen-futures v0.4.79
│   └── wasm-bindgen v0.2.129 (*)
├── web-sys v0.3.106
│   └── wasm-bindgen v0.2.129 (*)
├── js-sys v0.3.106
│   └── wasm-bindgen v0.2.129 (*)
├── gloo-utils v0.2.0
│   ├── wasm-bindgen v0.2.129 (*)
│   └── web-sys v0.3.106 (*)
```

เห็น `wasm-bindgen`, `web-sys`, และ `js-sys` เป็น dependency ตรงของ `yew` เป๊ะ — Part 87 สอนไว้ครบทุกตัว
(สังเกตด้วยว่า `gloo-utils` ที่ Yew พึ่งพาก็คือ crate ตัวเดียวกันในตระกูล `gloo` ที่หัวข้อ 88.8 จะใช้ตรง ๆ
ผ่าน `gloo-net` — Yew กับ `gloo` มาจาก ecosystem เดียวกัน คนละเลเยอร์)

### 88.2 ติดตั้งเครื่องมือ: trunk และ Hello World แรก

การ compile โปรเจกต์ Yew ให้กลายเป็นเว็บแอปที่รันได้จริงต้องมีขั้นตอนมากกว่า `cargo build` ธรรมดา — ต้อง
(1) compile ไปเป็น target `wasm32-unknown-unknown` (ทวนจาก Part 86-87) (2) รัน `wasm-bindgen` CLI เพื่อสร้าง
JS glue code ที่โหลด `.wasm` file (3) สร้าง `index.html` ที่โหลด glue code นั้น และ (4) เสิร์ฟไฟล์ทั้งหมดผ่าน
HTTP server (เบราว์เซอร์ไม่ยอมโหลด `.wasm` จาก `file://` โดยตรงด้วยเหตุผลด้าน security) ทำทุกขั้นตอนนี้ด้วยมือ
ทุกครั้งที่แก้โค้ดเป็นงานที่น่าเบื่อมาก — **`trunk`** คือ build tool ที่ทีม Yew แนะนำเป็นมาตรฐาน (ทำหน้าที่
คล้าย `webpack`/`vite` ในโลก JavaScript แต่เฉพาะทางสำหรับ Rust/WASM) มันรวมทั้ง 4 ขั้นตอนข้างต้นเข้าเป็นคำสั่ง
เดียว และยังมี dev server ที่ hot-reload ให้เองเวลาแก้โค้ด

ติดตั้ง `trunk` และ target `wasm32-unknown-unknown` (ทำครั้งเดียวต่อเครื่อง):

```bash
rustup target add wasm32-unknown-unknown
cargo install trunk --locked
```

จากนั้นสร้างโปรเจกต์และเพิ่ม `yew`:

```bash
cargo new hello-yew
cd hello-yew
cargo add yew --features csr
```

flag `--features csr` สำคัญ — Yew รองรับหลายโหมดการ render: **CSR** (Client-Side Rendering — render ทั้งหมด
ในเบราว์เซอร์ด้วย WASM ซึ่งเป็นโหมดที่บทนี้ใช้ทั้งบท), **SSR** (Server-Side Rendering — render เป็น HTML บน
เซิร์ฟเวอร์ก่อนส่งไปเบราว์เซอร์ ซึ่ง Part 91 จะพูดถึง) และ **hydration** (โหมดผสมทั้งสอง) ถ้าไม่เปิด feature
ที่ต้องการ `Renderer` (ตัวที่ mount component ลง DOM) จะไม่ถูก compile เข้ามา

`Cargo.toml` ที่ได้ (เวอร์ชันคือเวอร์ชันล่าสุดบน crates.io ณ เวลาที่เขียนบทนี้ — `cargo add` จะดึงเวอร์ชัน
ล่าสุดให้เสมอ ไม่ต้องพิมพ์เลขเวอร์ชันตามนี้เป๊ะ):

```toml
[package]
name = "hello-yew"
version = "0.1.0"
edition = "2021"

[dependencies]
yew = { version = "0.23.0", features = ["csr"] }
```

โปรเจกต์ที่ build ด้วย `trunk` ต้องมี `index.html` ที่ root ของโปรเจกต์ (ข้าง ๆ `Cargo.toml`) — นี่คือจุดที่
ต่างจากโปรเจกต์ Rust ปกติที่ไม่มี HTML entry point เลย:

```html
<!doctype html>
<html lang="th">
  <head>
    <meta charset="utf-8" />
    <title>Hello Yew</title>
  </head>
  <body></body>
</html>
```

สังเกตว่า `<body>` ว่างเปล่า — `trunk` จะ inject `<script>` ที่โหลด `.wasm` module ให้อัตโนมัติตอน build
(มันหา `<link data-trunk rel="rust" />` ถ้ามี หรือใส่ script tag ปกติให้เองถ้าไม่ระบุ) เนื้อหาทั้งหมดในหน้า
จะถูกสร้างโดย WASM module ตอน runtime ผ่าน component ที่คุณเขียน

ตัว component แรก ใน `src/main.rs`:

```rust
use yew::prelude::*;

// #[function_component] คือ attribute macro ที่แปลงฟังก์ชันธรรมดาที่ return Html
// ให้กลายเป็น "component" ที่ Yew รู้จัก — ชื่อในวงเล็บ (App) คือชื่อ type ของ component นี้
// (ธรรมเนียมของ Yew: ตั้งชื่อ component เป็น PascalCase เหมือน React component)
#[function_component(App)]
fn app() -> Html {
    // html! คือ procedural macro (แบบที่ Part 44-45 สอน ไม่ใช่ macro_rules! แบบ Part 36)
    // มันประมวลผลตอน compile time แปลง markup ที่หน้าตาคล้าย HTML/JSX ให้กลายเป็น Rust code
    // ธรรมดาที่สร้างค่า Html (struct ภายในของ Yew ที่แทน virtual DOM node หนึ่งต้น)
    html! {
        <div>
            <h1>{ "สวัสดี Yew!" }</h1>
            <p>{ "นี่คือ component แรกของเราที่รันจริงในเบราว์เซอร์ผ่าน WebAssembly" }</p>
        </div>
    }
}

fn main() {
    // Renderer::<App>::new() สร้างตัว render ที่ผูกกับ component type App
    // .render() คือจุดเริ่ม mount: มัน mount ลง <body> ทั้งก้อนโดย default
    // (ถ้าต้องการ mount ลง element เฉพาะ ใช้ Renderer::<App>::with_root(...) แทน)
    yew::Renderer::<App>::new().render();
}
```

รันด้วย:

```bash
trunk serve
```

`trunk serve` จะ compile โปรเจกต์ไปเป็น `wasm32-unknown-unknown`, รัน `wasm-bindgen` ให้อัตโนมัติ (ดาวน์โหลด
`wasm-bindgen-cli` เวอร์ชันที่ตรงกับ `wasm-bindgen` ใน `Cargo.lock` ให้เองถ้ายังไม่มี — จุดนี้สำคัญมาก เพราะ
`wasm-bindgen` crate (ที่ใช้ตอน compile) กับ `wasm-bindgen-cli` (ที่ใช้ตอน post-process) **ต้องเป็นเวอร์ชัน
เดียวกันเป๊ะ** ไม่งั้นจะได้ error ที่งงมากตอน runtime — Part 87 อาจพูดถึงเรื่องนี้ไปแล้วถ้าคุณใช้ `wasm-pack`)
แล้วเปิด dev server ที่ `http://127.0.0.1:8080` (ค่า default) พร้อม hot-reload ทุกครั้งที่แก้ไฟล์ `.rs`

ถ้าต้องการ build เป็นไฟล์ static (ไม่รัน dev server) เพื่อเอาไปเสิร์ฟด้วย server อื่น ใช้:

```bash
trunk build --release
```

ผลลัพธ์จะอยู่ใน `dist/` — โฟลเดอร์นี้มี `index.html` ที่ trunk แก้ไขให้แล้ว, ไฟล์ `.wasm`, และไฟล์ `.js` glue
code ที่ `wasm-bindgen` สร้าง — เอาทั้งโฟลเดอร์นี้ไปวางบน static file server ไหนก็ได้ (nginx, S3+CloudFront,
GitHub Pages ฯลฯ) ก็รันได้ทันที ไม่ต้องมี Rust toolchain อยู่บนเซิร์ฟเวอร์นั้นเลย — นี่คือข้อดีสำคัญของ Client
-Side Rendering: ผลลัพธ์สุดท้ายคือไฟล์ static ล้วน ๆ

**ผลจากการรันจริง**: เมื่อเปิดหน้าเว็บที่ได้ใน headless Chromium (ใช้ตรวจสอบผลลัพธ์ของบทนี้เอง — ดูหัวข้อ
"วิธีตรวจสอบ" ท้ายบท) และอ่านค่า `document.body.innerHTML` ออกมา จะได้ผลลัพธ์จริง:

```html
<div>
  <h1>สวัสดี Yew!</h1>
  <p>นี่คือ component แรกของเราที่รันจริงในเบราว์เซอร์ผ่าน WebAssembly</p>
</div>
```

สังเกตว่า `<div>`, `<h1>`, `<p>` ที่ปรากฏใน DOM จริง ตรงกับ markup ใน `html! { ... }` เป๊ะทุกตัวอักษร — นี่คือ
สิ่งที่พิสูจน์ว่า `html!` macro ไม่ใช่ string ที่ถูก interpret ตอน runtime แต่เป็น Rust code ที่ compile แล้ว
เรียก `web-sys` (`create_element`, `append_child`, `set_text_content`) ตรง ๆ เบื้องหลัง — output ที่เห็นคือ
DOM tree จริงที่ browser วาดออกมา ไม่ใช่ text ที่ copy วางมา

#### ขนาดไฟล์ `.wasm` จริง: ทำไม `--release` สำคัญมากสำหรับ WASM

Hello World ข้างบนดูเล็กมาก แต่ไฟล์ `.wasm` ที่ได้ไม่ได้เล็กตามสัดส่วนเสมอไป เพราะทุกอย่างที่ `yew` ต้องใช้
(virtual DOM diffing engine, hook system, `wasm-bindgen` runtime) ถูก compile รวมเข้าไปในไฟล์เดียวกันหมด — ลอง
เทียบขนาดไฟล์จริงระหว่าง `trunk build` (dev, ไม่ optimize) กับ `trunk build --release` (optimize เต็มที่) ของ
Hello World ตัวเดียวกันเป๊ะ (วัดจริงด้วย `ls -la dist/*.wasm` บนเครื่องที่เขียนบทนี้):

| Build mode | ขนาดไฟล์ `.wasm` ดิบ | ขนาดหลัง gzip (แบบที่ HTTP server ส่งจริงเวลามี compression) |
|---|---|---|
| `trunk build` (dev) | 1,067,851 bytes (~1.02 MiB) | — (ไม่มีใครส่ง dev build ให้ production) |
| `trunk build --release` | 284,440 bytes (~278 KiB) | 95,977 bytes (~94 KiB) |

ต่างกันเกือบ **4 เท่า** ระหว่าง dev กับ release ทั้ง ๆ ที่เป็นโค้ดชุดเดียวกันเป๊ะ — เหตุผลคือ dev build เก็บ
debug symbol และไม่ทำ dead-code elimination/inlining อย่างเข้มข้น (เหมือนกับที่ Part 54 สอนเรื่อง
`--release` ของ native binary ทั่วไป แต่ผลต่างสำหรับ WASM มักเห็นชัดกว่า เพราะ browser ต้อง download ไฟล์นี้
ผ่านเครือข่ายทุกครั้งที่ผู้ใช้เปิดหน้าเว็บครั้งแรก ขนาดไฟล์จึงกระทบ **เวลาโหลดหน้าเว็บจริง** ไม่ใช่แค่เวลา
compile) กฎง่าย ๆ ที่ต้องจำ: **อย่า deploy `.wasm` จาก `trunk build` (dev) ขึ้น production เด็ดขาด** ใช้
`trunk build --release` เสมอ — ต่างจากงาน backend (Part 62-85) ที่บางทีปล่อย dev build ไปทดสอบใน staging ยัง
พอไหว แต่กับ WASM ฝั่ง frontend ผลต่างของขนาดไฟล์กระทบผู้ใช้จริงตรง ๆ ทุกครั้งที่โหลดหน้า

(ตัวเลขข้างบนคือของ Hello World ที่มีแค่ `yew` เป็น dependency — แอปจริงที่เพิ่ม `yew-router`, `gloo-net`
ฯลฯ เข้าไปจะมีขนาดมากกว่านี้ตามด้วย แต่สัดส่วนที่ dev ใหญ่กว่า release หลายเท่ายังคงจริงเสมอ)

### 88.3 Component และ Props: สร้าง `BookCard`

Hello World ข้างบนเป็น component ที่ไม่รับข้อมูลจากภายนอกเลย — ในแอปจริงคุณต้องส่งข้อมูลเข้า component จาก
"ข้างนอก" (จาก component แม่) แบบเดียวกับที่ function ธรรมดารับ parameter — ใน Yew เรียกสิ่งนี้ว่า **props**
(props ย่อจาก "properties" — คำเดียวกับที่ React ใช้ เพราะ mental model เหมือนกันตรง ๆ)

มาสร้าง component `BookCard` สำหรับระบบห้องสมุด (domain เดียวกับที่คอร์สนี้ใช้ซ้ำในหลาย Part) ที่รับข้อมูล
หนังสือหนึ่งเล่มมา render:

```rust
use yew::prelude::*;

// struct ข้อมูลหนังสือ — โครงสร้างข้อมูลธรรมดา ไม่มีอะไรพิเศษของ Yew เลย
// PartialEq/Clone จำเป็น เพราะ props ทุกตัวต้อง Clone ได้ (Yew clone props ทุกครั้งที่ re-render
// เพื่อส่งให้ component ใหม่) และ PartialEq (จะอธิบายต่อว่าทำไมจำเป็นด้านล่าง)
#[derive(Clone, PartialEq, Debug)]
pub struct Book {
    pub id: u32,
    pub title: String,
    pub author: String,
    pub available: bool,
}

// struct props ของ BookCard — ต้อง derive สองตัวนี้เสมอสำหรับ function component:
// - Properties: มาจาก yew (macro นี้ generate โค้ดที่ทำให้ struct นี้ใช้เป็น props ได้)
// - PartialEq: Yew ใช้เทียบ props เก่ากับ props ใหม่ทุกครั้งที่ component แม่ re-render
//   ถ้า props "เหมือนเดิม" (PartialEq เท่ากัน) Yew จะ "ข้าม" การ re-render component ลูกนี้ไปเลย
//   (นี่คือ optimization อัตโนมัติที่มาจากการ derive PartialEq — ไม่ต้องเขียนโค้ดพิเศษอะไรเพิ่ม)
#[derive(Properties, PartialEq)]
pub struct BookCardProps {
    pub book: Book,
}

#[function_component(BookCard)]
fn book_card(props: &BookCardProps) -> Html {
    let book = &props.book;
    let status_text = if book.available { "ว่าง" } else { "ถูกยืมแล้ว" };
    let status_class = if book.available { "status-available" } else { "status-borrowed" };

    html! {
        <div class="book-card">
            <h3>{ &book.title }</h3>
            <p>{ format!("ผู้เขียน: {}", book.author) }</p>
            <span class={status_class}>{ status_text }</span>
        </div>
    }
}

#[function_component(App)]
fn app() -> Html {
    let sample_book = Book {
        id: 1,
        title: "The Rust Programming Language".to_string(),
        author: "Steve Klabnik และ Carol Nichols".to_string(),
        available: true,
    };

    html! {
        // เวลาส่ง component เป็น child ของ html! ใช้ syntax <ComponentName prop_name={value} />
        // สังเกตว่านี่คือ syntax เดียวกันกับ HTML attribute ธรรมดา แต่ค่าที่ใส่ใน {} คือ
        // Rust expression จริง ไม่ใช่ string เสมอไป (ตรงนี้คือจุดที่ html! ต่างจาก HTML ตรง ๆ)
        <BookCard book={sample_book} />
    }
}

fn main() {
    yew::Renderer::<App>::new().render();
}
```

ไล่ทีละส่วนที่สำคัญ:

- **`#[derive(Properties, PartialEq)]`** — `Properties` เป็น derive macro ที่มาจาก crate `yew` เอง (ทำงาน
  แบบเดียวกับ derive macro ที่ Part 45 สอน — มันสร้าง `impl` ให้ struct นี้ implement trait `Properties` ของ
  Yew ให้อัตโนมัติ) สิ่งที่ทำให้เกิดขึ้นคือ struct นี้สามารถถูกใช้เป็น "ชนิดของ prop" ของ function component
  ได้ — ถ้าลืม derive ตัวนี้ compiler จะบอกตรง ๆ ว่า `the trait bound BookCardProps: Properties is not
  satisfied`
- **`&BookCardProps`** — function component รับ props เป็น **reference** เสมอ (`&Props` ไม่ใช่ `Props`) เพราะ
  Yew เป็นเจ้าของ props อยู่แล้วภายใน (มันต้องเก็บ props ไว้เทียบกับ props รอบถัดไป) — ฟังก์ชัน component จึง
  แค่ "ยืมดู" ผ่าน reference ทวนจาก Part 7 (Borrowing) ตรง ๆ ว่าทำไม borrow ถึงเพียงพอในกรณีนี้: เราอ่านข้อมูล
  เพื่อสร้าง `Html` เท่านั้น ไม่ได้ต้องการ ownership
- **`<BookCard book={sample_book} />`** — วิธีเรียกใช้ component ลูกจาก component แม่ ค่าใน `{}` เป็น Rust
  expression จริง ในตัวอย่างนี้คือตัวแปร `sample_book` ที่ moved เข้าไปเป็นค่า `book` ของ props (ทวนจาก Part 6
  ว่าทำไม move นี้ไม่มีปัญหา — `sample_book` เป็น local variable ที่ไม่ได้ใช้ต่อหลังจากนี้)

**ผลจากการรันจริง**: build ตัวอย่างนี้ด้วย `trunk build` แล้วเปิดในเบราว์เซอร์จริง อ่าน `document.body.
innerHTML` ได้ผลลัพธ์จริง:

```html
<div class="book-card">
  <h3>The Rust Programming Language</h3>
  <p>ผู้เขียน: Steve Klabnik และ Carol Nichols</p>
  <span class="status-available">ว่าง</span>
</div>
```

จะเห็นว่าข้อมูลจาก `Book` struct (title, author, สถานะ) ไหลจาก component แม่ (`App`) ไปยัง component ลูก
(`BookCard`) ผ่าน props แล้ว render ออกมาเป็น DOM จริงครบทุกจุด — ตรงกับที่คาดไว้จากโค้ดทุกประการ

#### macro `classes!`: กำหนด CSS class แบบมีเงื่อนไข

ตัวอย่าง `BookCard` ข้างบนใช้ `class="book-card"` เป็น string literal ตรง ๆ ซึ่งพอสำหรับ class คงที่ แต่บ่อยครั้ง
class ต้อง**เปลี่ยนตามเงื่อนไข** (เช่น เพิ่ม class `"highlighted"` เมื่อผู้ใช้เลือกการ์ดนั้นอยู่) — เขียน
`format!("book-card {}", if selected { "highlighted" } else { "" })` เองได้ แต่จะเหลือ space เกินตอนไม่มีเงื่อน
ไขจริง (`"book-card "` มี space ท้าย) และอ่านยากขึ้นเรื่อย ๆ เมื่อเงื่อนไขมีหลายตัว — Yew มี macro `classes!`
ที่แก้ปัญหานี้ให้ตรง ๆ:

```rust
use yew::prelude::*;

#[function_component(Greeting)]
fn greeting() -> Html {
    // use_state(|| false) — state แบบ bool ธรรมดา เก็บว่า "ไฮไลต์อยู่หรือไม่"
    let highlighted = use_state(|| false);

    let onclick = {
        let highlighted = highlighted.clone();
        Callback::from(move |_| highlighted.set(!*highlighted))
    };

    // classes! รับ argument ได้หลายแบบผสมกัน: &str ธรรมดา, Option<&str> (None แปลว่า "ไม่เอา class นี้"),
    // หรือ Vec<String> — ผลลัพธ์คือ Classes ที่ Yew join ด้วย space ให้ถูกต้องเสมอ ไม่มี space เกิน
    let classes = classes!(
        "greeting",
        highlighted.then(|| "greeting--highlighted"), // Option<&str>: Some เมื่อ highlighted เป็น true
    );

    html! {
        <div>
            <p class={classes}>{ "สวัสดี, Rustacean!" }</p>
            <button {onclick}>{ "toggle" }</button>
        </div>
    }
}
```

**ผลจากการรันจริง**: สถานะเริ่มต้น (`highlighted = false`) `document.body.innerHTML`:

```html
<div><p class="greeting">สวัสดี, Rustacean!</p><button>toggle</button></div>
```

คลิกปุ่ม `toggle` ครั้งแรก (`highlighted` กลายเป็น `true`):

```html
<div><p class="greeting greeting--highlighted">สวัสดี, Rustacean!</p><button>toggle</button></div>
```

คลิกอีกครั้ง (`highlighted` กลับเป็น `false`) — class `"greeting--highlighted"` หายไปเอง กลับไปเป็น
`class="greeting"` เหมือนตอนแรกเป๊ะ — `classes!` คำนวณ string ที่ join กันถูกต้องให้ทุกครั้งโดยไม่ต้องมี space
เกินหรือขาดเลย ไม่ว่าจะมีเงื่อนไขกี่ตัวผสมกัน (ในโปรเจกต์ที่ใช้ CSS framework แบบ utility-class เช่น Tailwind
ที่ต้องผสม class เป็นสิบตัวตามเงื่อนไข `classes!` จะมีประโยชน์ชัดเจนมากขึ้นไปอีก)

### 88.4 State ด้วย Hooks: `use_state` และ `use_effect`

Component ที่เห็นมาจนถึงตอนนี้เป็น component แบบ "รับข้อมูลจากข้างนอก แล้วแสดงผล" ล้วน ๆ — ไม่มี "ความจำ" ของ
ตัวเอง (เรียกว่า **stateless**) แต่ UI จริงส่วนใหญ่ต้องมี state ภายในตัวมันเอง เช่น ข้อความในช่องค้นหาที่ผู้ใช้
พิมพ์อยู่ — Yew ให้ **hooks** สำหรับเรื่องนี้ (คำและ mental model นี้ก็ยืมมาจาก React ตรง ๆ อีกครั้ง — React
เปิดตัว hooks ในปี 2019 และเป็นแนวคิดที่แพร่หลายไปในหลาย framework ของหลายภาษา)

#### `use_state`: ความจำที่อยู่รอดข้าม re-render

```rust
use yew::prelude::*;

#[function_component(Counter)]
fn counter() -> Html {
    // use_state(|| initial_value) — closure จะถูกเรียกแค่ครั้งแรกที่ component นี้ mount เท่านั้น
    // ครั้งต่อไปที่ component re-render ค่าเดิมจะถูกดึงกลับมา ไม่ถูกสร้างใหม่
    // ชนิดของ count คือ UseStateHandle<i32> — smart pointer ที่ deref เป็น &i32 ได้
    let count = use_state(|| 0_i32);

    html! {
        <p>{ format!("นับได้: {}", *count) }</p>
    }
}
```

`use_state(|| 0)` คืนค่าชนิด `UseStateHandle<T>` — ไม่ใช่ `T` ตรง ๆ เหตุผลคือ `UseStateHandle<T>` ต้องทำสอง
อย่างพร้อมกัน: (1) ให้ *อ่าน* ค่าปัจจุบันได้ (ผ่าน `Deref` ไปเป็น `&T` — นี่คือเหตุผลที่เขียน `*count` เพื่อ
ได้ `i32` ออกมา คล้ายกับที่ Part 27-28 สอนเรื่อง `Deref` ของ `Box`/`Rc`) และ (2) ให้ *เขียน* ค่าใหม่ผ่าน
`.set(new_value)` ซึ่งไม่ได้แค่เปลี่ยนตัวแปรธรรมดา — มันบอก Yew ว่า "component นี้ต้อง re-render" ด้วย
(triggering re-render คือสิ่งที่ทำให้ `use_state` ต่างจากตัวแปร `let mut` ธรรมดาโดยสิ้นเชิง: `let mut x = 0;
x += 1;` ไม่มีทางบอก Yew ให้วาดหน้าจอใหม่ได้เลย)

`UseStateHandle<T>` implement `Clone` เสมอ (ไม่ว่า `T` จะ `Clone` หรือไม่ — มันเป็น handle บาง ๆ ที่ภายใน
เป็น `Rc` ชี้ไปยังค่าจริง ทวนจาก Part 28 เรื่อง `Rc` ว่าทำไม clone ของ `Rc` "ถูก" มาก — เป็นแค่ increment
reference count ไม่ใช่ copy ข้อมูลทั้งก้อน) นี่คือเหตุผลที่โค้ดในหัวข้อ 88.1 เขียน `let count = count.clone();`
ก่อน `move` เข้า closure ของ `onclick` ได้อย่างไม่แพง

#### ตัวอย่างที่มีความหมายกว่า: ระบบค้นหา/กรองรายการหนังสือ

```rust
use yew::prelude::*;

#[derive(Clone, PartialEq, Debug)]
struct Book {
    id: u32,
    title: String,
    author: String,
}

fn sample_books() -> Vec<Book> {
    vec![
        Book { id: 1, title: "The Rust Programming Language".into(), author: "Steve Klabnik".into() },
        Book { id: 2, title: "Programming Rust".into(), author: "Jim Blandy".into() },
        Book { id: 3, title: "Zero To Production In Rust".into(), author: "Luca Palmieri".into() },
        Book { id: 4, title: "Rust for Rustaceans".into(), author: "Jon Gjengset".into() },
    ]
}

#[function_component(BookSearch)]
fn book_search() -> Html {
    // state ที่เก็บข้อความในช่องค้นหา — เริ่มต้นเป็น string ว่าง
    let query = use_state(String::new);
    let books = use_state(sample_books);

    // กรองรายการหนังสือด้วย query ปัจจุบัน — ทวนจาก Part 25-26: .iter().filter().cloned().collect()
    // คำนวณใหม่ทุกครั้งที่ component นี้ re-render (คือทุกครั้งที่ query หรือ books เปลี่ยน)
    let query_lower = query.to_lowercase();
    let filtered: Vec<Book> = books
        .iter()
        .filter(|b| {
            b.title.to_lowercase().contains(&query_lower)
                || b.author.to_lowercase().contains(&query_lower)
        })
        .cloned()
        .collect();

    html! {
        <div>
            <p>{ format!("พบ {} เล่มจากทั้งหมด {} เล่ม", filtered.len(), books.len()) }</p>
            <ul>
                { for filtered.iter().map(|book| html! {
                    <li key={book.id}>{ format!("{} — {}", book.title, book.author) }</li>
                }) }
            </ul>
        </div>
    }
}
```

ตัวอย่างนี้ยังไม่มีช่อง input ที่แก้ `query` ได้ (จะเพิ่มในหัวข้อ 88.5 เรื่อง event handling) แต่แสดงให้เห็น
แล้วว่า `use_state` ทำให้การ derive ค่าที่ขึ้นกับ state (`filtered`) เขียนเป็น expression ธรรมดาได้เลย ไม่ต้อง
มี logic แยกไว้คอย sync กับ DOM แบบ Part 87 — ทุกครั้งที่ `query` เปลี่ยน Yew จะเรียก `book_search()` ใหม่ทั้ง
ฟังก์ชัน คำนวณ `filtered` ใหม่ แล้ว diff กับ `Html` เก่า

#### `use_effect_with`: side effect ที่ผูกกับการเปลี่ยนแปลงของค่าที่ระบุ

`use_state` จัดการ "ความจำ" แต่ยังมีอีกกรณีที่พบบ่อย: การรันโค้ดบางอย่าง**ตอบสนองต่อการเปลี่ยนแปลง**ของค่าบาง
ตัว โดยไม่ได้เกิดจาก event ของ DOM ตรง ๆ (เช่น fetch ข้อมูลใหม่ทุกครั้งที่ id ของหนังสือที่กำลังดูเปลี่ยน หรือ
set ค่า document title ให้ตรงกับหนังสือที่กำลังดู) — Yew มี `use_effect_with` สำหรับกรณีนี้:

```rust
use yew::prelude::*;
use web_sys::window;

#[derive(Properties, PartialEq)]
struct PageTitleProps {
    title: String,
}

#[function_component(PageTitleSetter)]
fn page_title_setter(props: &PageTitleProps) -> Html {
    let title = props.title.clone();

    // use_effect_with(deps, callback) — callback จะถูกเรียกหลัง Yew วาด DOM เสร็จแล้ว
    // (คือ "หลังจาก" render ไม่ใช่ "ระหว่าง" render — สำคัญมากเพราะ effect หลายตัวแก้ browser API
    // ที่ไม่ปลอดภัยถ้าทำระหว่างที่ virtual DOM ยังไม่นิ่ง) callback จะถูกเรียกใหม่อีกครั้งก็ต่อเมื่อ
    // ค่าใน deps (ที่นี่คือ title) เปลี่ยนไปจากรอบก่อนเท่านั้น — เทียบ PartialEq ให้อัตโนมัติเหมือนกับ
    // ที่ React เทียบ dependency array ของ useEffect
    use_effect_with(title.clone(), move |title: &String| {
        if let Some(doc) = window().and_then(|w| w.document()) {
            doc.set_title(&format!("ห้องสมุด — {title}"));
        }
        // ทวนจาก Part 87 เรื่อง web-sys: window()/document() คืนค่า Option เพราะอาจไม่มี document
        // จริง ๆ ในบริบทบางอย่าง (เช่น Web Worker) — Yew ไม่ได้ทำให้ Option หายไป มันแค่ให้จุดที่
        // เรียก web-sys อยู่ในที่ที่ "ปลอดภัยเรื่องเวลา" (หลัง render) เท่านั้น
    });

    html! { <h1>{ &title }</h1> }
}
```

`use_effect_with` มีเวอร์ชันที่ไม่ระบุ dependency เลย (`use_effect`, รันทุกครั้งที่ re-render — ใช้น้อยกว่า
เพราะมักทำงานถี่เกินจำเป็น) และเวอร์ชันที่ใช้ tuple ว่าง `()` เป็น deps (รันครั้งเดียวตอน mount เท่านั้น —
pattern ที่จะใช้ในหัวข้อ 88.8 สำหรับการ fetch ข้อมูลตอนเปิดหน้า) closure สามารถ return closure อีกชั้นเป็น
**cleanup function** ที่ Yew จะเรียกก่อน effect รอบถัดไปจะรัน หรือก่อน component ถูก unmount — ใช้บ่อยสำหรับ
ยกเลิก timer หรือ event listener ที่ effect สร้างไว้ (ทวนแนวคิดเดียวกับ `Drop` trait ของ Part 27 แต่ผูกกับ
"รอบของ effect" แทน "อายุของตัวแปร")

#### เมื่อ `use_state` ไม่พอ: `use_reducer` สำหรับ state ที่มีหลายวิธีเปลี่ยนแปลง

`use_state` เหมาะกับ state ที่ "แก้ตรง ๆ" ได้ง่าย (ตัวเลข, string, `Vec` ที่แทนที่ทั้งก้อน) แต่ถ้า state หนึ่งตัว
มี**วิธีเปลี่ยนแปลงหลายแบบ** ที่ต้องแยกความชัดเจน (เช่น ตะกร้าสินค้าที่ "เพิ่ม" กับ "ลบ" ทำงานต่างกัน) การเขียน
`Callback::from(move |_| state.set(...))` แยกทุกจุดจะเริ่มซ้ำซ้อนและกระจายตรรกะการแก้ state ไปทั่วทั้งไฟล์ —
Yew มี **`use_reducer`** (ยืม pattern มาจาก `useReducer` ของ React ซึ่งก็ยืมมาจาก Redux อีกที) ที่รวบรวม "ทุก
วิธีที่ state จะเปลี่ยนได้" ไว้เป็นฟังก์ชันเดียว แยกออกจากจุดที่ *เรียกใช้* การเปลี่ยนแปลงนั้นชัดเจน:

```rust
use std::rc::Rc;
use yew::prelude::*;

#[derive(Clone, PartialEq, Debug)]
struct CartState {
    count: u32,
}

// enum ที่แทน "การกระทำ" ทั้งหมดที่เป็นไปได้กับ state นี้ — ทวนจาก Part 10 เรื่อง enum/pattern matching
enum CartAction {
    Add,
    Remove,
}

// Reducible คือ trait ของ Yew ที่บอกว่า "จาก state เดิม + action หนึ่งตัว จะได้ state ใหม่ยังไง"
// รับ self เป็น Rc<Self> (ไม่ใช่ &self หรือ self ตรง ๆ) เพราะ use_reducer เก็บ state ไว้เป็น Rc ภายใน
// เพื่อให้ clone ถูกแบบเดียวกับ UseStateHandle
impl Reducible for CartState {
    type Action = CartAction;

    fn reduce(self: Rc<Self>, action: Self::Action) -> Rc<Self> {
        match action {
            CartAction::Add => Rc::new(CartState { count: self.count + 1 }),
            CartAction::Remove => Rc::new(CartState { count: self.count.saturating_sub(1) }),
        }
    }
}

#[function_component(Cart)]
fn cart() -> Html {
    let state = use_reducer(|| CartState { count: 0 });

    // .dispatch(action) คือจุดเดียวที่ "เรียกใช้" การเปลี่ยนแปลง — ไม่มีจุดไหนใน component เขียน
    // state.count = ... ตรง ๆ เลย ตรรกะการคำนวณค่าใหม่ทั้งหมดถูกรวมไว้ที่ reduce() เพียงจุดเดียว
    let add = {
        let state = state.clone();
        Callback::from(move |_| state.dispatch(CartAction::Add))
    };
    let remove = {
        let state = state.clone();
        Callback::from(move |_| state.dispatch(CartAction::Remove))
    };

    html! {
        <div>
            <p>{ format!("จำนวนในตะกร้า: {}", state.count) }</p>
            <button onclick={add}>{ "+" }</button>
            <button onclick={remove}>{ "-" }</button>
        </div>
    }
}

fn main() {
    yew::Renderer::<Cart>::new().render();
}
```

**ผลจากการรันจริง**: คลิกปุ่ม `+` สามครั้งติดกัน แล้วอ่าน `document.body.innerHTML`:

```html
<div><p>จำนวนในตะกร้า: 3</p><button>+</button><button>-</button></div>
```

คลิกปุ่ม `-` อีกครั้ง:

```html
<div><p>จำนวนในตะกร้า: 2</p><button>+</button><button>-</button></div>
```

ข้อดีของ pattern นี้เห็นชัดเมื่อ state ซับซ้อนขึ้น: ถ้าอีกสองสัปดาห์ต้องเพิ่ม action `Clear` (ล้างตะกร้าทั้งหมด)
สิ่งที่ต้องแก้คือเพิ่ม variant ใน `CartAction` กับเพิ่ม arm ใน `match` ของ `reduce()` — ไม่ต้องไปตามหาว่ามีจุด
ไหนใน component ที่แก้ `state.count` มือแบบกระจัดกระจายบ้าง เพราะไม่มีจุดแบบนั้นอยู่แล้วตั้งแต่ต้น

### 88.5 Event Handling: `Callback<T>` ผูก `onclick`/`oninput` เข้ากับ state

มาต่อยอดตัวอย่าง `BookSearch` จากหัวข้อ 88.4 ให้ช่องค้นหาใช้งานได้จริง — ต้องผูก event `oninput` ของ
`<input>` เข้ากับการอัปเดต `query` state — โค้ดนี้ต้องใช้ type `HtmlInputElement` จาก crate `web-sys` ตรง ๆ
(ตัวเดียวกับที่ Part 87 สอน) ซึ่ง**ไม่ได้ถูก re-export มาให้ผ่าน `yew::prelude::*` โดยอัตโนมัติ** ต้องเพิ่ม
`web-sys` เป็น dependency ของโปรเจกต์เองพร้อมเปิด feature ของ type ที่จะใช้อย่างเจาะจง (ทวนจาก Part 87 ว่า
`web-sys` ออกแบบให้ทุก type ของ Web API อยู่หลัง feature flag ของตัวเอง เพื่อไม่ให้ compile เข้ามาทั้งหมดโดยไม่
จำเป็น):

```bash
cargo add web-sys --features HtmlInputElement
```

```rust
use web_sys::HtmlInputElement;
use yew::prelude::*;

#[derive(Clone, PartialEq, Debug)]
struct Book {
    id: u32,
    title: String,
    author: String,
}

fn sample_books() -> Vec<Book> {
    vec![
        Book { id: 1, title: "The Rust Programming Language".into(), author: "Steve Klabnik".into() },
        Book { id: 2, title: "Programming Rust".into(), author: "Jim Blandy".into() },
        Book { id: 3, title: "Zero To Production In Rust".into(), author: "Luca Palmieri".into() },
        Book { id: 4, title: "Rust for Rustaceans".into(), author: "Jon Gjengset".into() },
    ]
}

#[function_component(BookSearch)]
fn book_search() -> Html {
    let query = use_state(String::new);
    let books = use_state(sample_books);

    // Callback<InputEvent> — Callback<T> คือ "closure ที่ clone ได้และส่งเป็น prop ได้"
    // ตัวมันเองก็เก็บอยู่ใน Rc ภายใน (คล้าย UseStateHandle) เพื่อให้ clone ได้ถูก ๆ
    let oninput = {
        let query = query.clone();
        Callback::from(move |event: InputEvent| {
            // event ที่ได้จาก DOM เป็น web_sys::InputEvent ตรง ๆ (ชนิดเดียวกับ Part 87 สอน)
            // .target() คืน Option<EventTarget> ต้อง dyn_into แปลงให้เป็น HtmlInputElement
            // ก่อนจะอ่าน .value() ได้ — นี่คือจุดที่เห็นชัดว่า Yew ไม่ได้ "ปิดกั้น" web-sys เลย
            // มันแค่ให้คุณเขียน event handler แบบ declarative แต่ตัว event เองยังเป็นของจริง 100%
            if let Some(input) = event.target_dyn_into::<HtmlInputElement>() {
                query.set(input.value());
            }
        })
    };

    let query_lower = query.to_lowercase();
    let filtered: Vec<Book> = books
        .iter()
        .filter(|b| {
            b.title.to_lowercase().contains(&query_lower)
                || b.author.to_lowercase().contains(&query_lower)
        })
        .cloned()
        .collect();

    html! {
        <div>
            <input
                type="text"
                placeholder="ค้นหาชื่อหนังสือหรือผู้เขียน..."
                value={(*query).clone()}
                {oninput}
            />
            <p>{ format!("พบ {} เล่มจากทั้งหมด {} เล่ม", filtered.len(), books.len()) }</p>
            <ul>
                { for filtered.iter().map(|book| html! {
                    <li key={book.id}>{ format!("{} — {}", book.title, book.author) }</li>
                }) }
            </ul>
        </div>
    }
}

fn main() {
    yew::Renderer::<BookSearch>::new().render();
}
```

จุดที่ต้องเข้าใจให้แน่นในตัวอย่างนี้:

- **`event.target_dyn_into::<HtmlInputElement>()`** — เมธอดสะดวกของ `web-sys` (มาจาก trait
  `TargetCast` ที่ Yew re-export) ที่ทำสิ่งเดียวกับ `event.target().unwrap().dyn_into::<HtmlInputElement>()
  .unwrap()` ของ Part 87 แต่คืนเป็น `Option` ให้เขียน `if let` จัดการ error ได้สะดวกกว่า — เบื้องหลังยังเป็น
  `wasm_bindgen::JsCast::dyn_into` ตัวเดิมเป๊ะ
- **`value={(*query).clone()}`** — สังเกตว่า `<input>` เขียน `value` ด้วยค่าจาก state ตรง ๆ (ไม่ใช่ปล่อยให้
  DOM จำค่าของตัวเอง) นี่คือ pattern ที่เรียกว่า **controlled input** (คำนี้ก็มาจาก React อีกครั้ง) — React
  /Yew "เป็นเจ้าของ" ค่าของ input ผ่าน state เสมอ ไม่ใช่ให้ DOM เก็บค่าไว้เอง ข้อดีคือ state กับสิ่งที่แสดงบน
  จอ **ไม่มีทางหลุดจากกันได้เลย** ต่างจากโค้ด Part 87 ที่ต้องคอย sync มือระหว่าง `<input>` element กับตัวแปร
  ที่เก็บค่าไว้แยกกัน
- **`{oninput}`** — shorthand ของ `oninput={oninput}` เมื่อชื่อตัวแปรตรงกับชื่อ attribute เป๊ะ (คล้ายกับ
  object shorthand ของ JavaScript แต่ตรงนี้เป็น syntax พิเศษของ `html!` macro เอง ไม่ใช่ feature ของ Rust
  ทั่วไป)

**ผลจากการรันจริง**: หลัง build ด้วย `trunk build` และเปิดในเบราว์เซอร์จริง พิมพ์คำว่า `"rust"` ลงในช่องค้นหา
(จำลองการพิมพ์จริงผ่าน browser automation ไม่ใช่แค่แก้ state ด้วยโค้ด) แล้วอ่าน `document.body.innerHTML`:

```html
<div>
  <input type="text" placeholder="ค้นหาชื่อหนังสือหรือผู้เขียน...">
  <p>พบ 4 เล่มจากทั้งหมด 4 เล่ม</p>
  <ul>
    <li>The Rust Programming Language — Steve Klabnik</li>
    <li>Programming Rust — Jim Blandy</li>
    <li>Zero To Production In Rust — Luca Palmieri</li>
    <li>Rust for Rustaceans — Jon Gjengset</li>
  </ul>
</div>
```

สังเกตสิ่งที่ **ไม่** ปรากฏ: `<input>` ไม่มี `value="rust"` ติดอยู่ใน `innerHTML` เลย แม้ผู้ใช้พิมพ์ "rust" ไป
แล้วจริง ๆ — นี่ไม่ใช่บั๊ก แต่เป็นพฤติกรรมมาตรฐานของ DOM: ค่า `value` ของ `<input>` เป็น **IDL property**
(ตั้งผ่าน `element.value = ...` ซึ่งเป็นสิ่งที่ Yew ทำเบื้องหลังให้ตรง ๆ ผ่าน `web-sys`) ไม่ใช่ **HTML attribute**
(ที่จะสะท้อนออกมาใน `innerHTML`/`outerHTML`) ต่างหาก — ทวนจาก Part 87: การอ่าน "ค่าปัจจุบัน" ของ input ต้องอ่าน
ผ่าน property (`document.querySelector('input').value` ซึ่งได้ `"rust"` จริง) ไม่ใช่ผ่าน attribute หรือ
`innerHTML` — สิ่งที่ยืนยันได้จริงคือผลของการกรองที่ถูกต้อง (`พบ 4 เล่ม` เพราะทุกเล่มมีคำว่า "rust" ในชื่อ) ซึ่ง
พิสูจน์ว่า `query.set(input.value())` ใน `oninput` callback ทำงานจริงและ trigger re-render จริง

แล้วพิมพ์ต่อเป็น `"rust for"` ผลลัพธ์กรองเหลือเล่มเดียวตามคาด:

```html
<div>
  <input type="text" placeholder="ค้นหาชื่อหนังสือหรือผู้เขียน...">
  <p>พบ 1 เล่มจากทั้งหมด 4 เล่ม</p>
  <ul>
    <li>Rust for Rustaceans — Jon Gjengset</li>
  </ul>
</div>
```

state เปลี่ยน → component re-render → virtual DOM diff → เฉพาะส่วนของ `<ul>` ที่ต่างถูกแก้จริงในเบราว์เซอร์
ทั้งหมดนี้เกิดขึ้นโดยไม่มีโค้ดเส้นไหนใน `book_search()` พูดถึง DOM node ตรง ๆ เลยแม้แต่บรรทัดเดียว

### 88.6 List และ Key: render `Vec<Book>` เป็นรายการ component

หัวข้อก่อนหน้าใช้ `.iter().map(...)` render รายการเป็น `<li>` ธรรมดาไปแล้ว — ทีนี้ลอง render เป็นรายการของ
`BookCard` component (จากหัวข้อ 88.3) แทน เพื่อ reuse component ที่มีอยู่:

```rust
use yew::prelude::*;

#[derive(Clone, PartialEq, Debug)]
struct Book {
    id: u32,
    title: String,
    author: String,
    available: bool,
}

#[derive(Properties, PartialEq)]
struct BookCardProps {
    book: Book,
}

#[function_component(BookCard)]
fn book_card(props: &BookCardProps) -> Html {
    let book = &props.book;
    let status = if book.available { "ว่าง" } else { "ถูกยืมแล้ว" };
    html! {
        <div class="book-card">
            <h3>{ &book.title }</h3>
            <span>{ status }</span>
        </div>
    }
}

fn sample_books() -> Vec<Book> {
    vec![
        Book { id: 1, title: "The Rust Programming Language".into(), author: "Steve Klabnik".into(), available: true },
        Book { id: 2, title: "Programming Rust".into(), author: "Jim Blandy".into(), available: false },
        Book { id: 3, title: "Zero To Production In Rust".into(), author: "Luca Palmieri".into(), available: true },
    ]
}

#[function_component(BookList)]
fn book_list() -> Html {
    let books = sample_books();

    html! {
        <div class="book-list">
            // .iter().map(...) ธรรมดาเป๊ะ ๆ ตามที่ Part 25-26 สอน — html! macro รองรับ { for ... }
            // สำหรับ expression ที่เป็น iterator ของ Html โดยเฉพาะ (เขียน { for iter } ไม่ใช่
            // { iter.collect::<Html>() } ตรง ๆ แม้ทั้งสองแบบทำงานเหมือนกันเป๊ะ — { for } เป็น
            // syntax sugar ที่ html! macro มอบให้)
            { for books.iter().map(|book| html! {
                // key={book.id} สำคัญมากเวลา list มีการ เพิ่ม/ลบ/สลับลำดับ — มันบอก Yew ว่า
                // "node นี้แทนหนังสือเล่มไหน" ข้ามรอบ re-render แต่ละรอบ ทำให้ virtual DOM diffing
                // จับคู่ node เก่ากับใหม่ได้ถูกตัว แทนที่จะจับคู่ตาม "ตำแหน่งในลิสต์" เฉย ๆ
                // (ปัญหาเดียวกันกับที่ React ต้องใช้ key prop — ถ้าไม่มี key แล้ว list ถูกเรียงลำดับ
                // ใหม่ Yew อาจ "จำผิดตัว" ว่า node ไหนคือหนังสือเล่มไหน ทำให้ animation/focus/scroll
                // position เพี้ยนได้ แม้ข้อมูลจะถูกต้อง)
                <BookCard key={book.id} book={book.clone()} />
            }) }
        </div>
    }
}

fn main() {
    yew::Renderer::<BookList>::new().render();
}
```

**ผลจากการรันจริง**: `document.body.innerHTML` ที่ได้จาก build+run จริง:

```html
<div class="book-list">
  <div class="book-card"><h3>The Rust Programming Language</h3><span>ว่าง</span></div>
  <div class="book-card"><h3>Programming Rust</h3><span>ถูกยืมแล้ว</span></div>
  <div class="book-card"><h3>Zero To Production In Rust</h3><span>ว่าง</span></div>
</div>
```

3 เล่มจาก `sample_books()` ถูก render เป็น `BookCard` 3 ก้อนตามลำดับใน `Vec` เป๊ะ — `book.clone()` จำเป็นตรงนี้
เพราะ `book` ใน closure ของ `.map()` เป็น `&Book` (ยืมมาจาก `books.iter()`) แต่ props ของ `BookCard` ต้องการ
`Book` แบบมี ownership เต็ม ๆ (props ถูก Yew เก็บไว้ใช้เทียบรอบถัดไป จะยืมจาก `Vec` เดิมที่อาจถูก drop ไปแล้ว
ไม่ได้) — ทวนจาก Part 6-7 ตรง ๆ ว่าทำไม compiler บังคับให้ clone ในจุดนี้: lifetime ของ `books` (ตัวแปร local
ใน `book_list()`) สั้นกว่า lifetime ของ props ที่ Yew ต้องเก็บไว้ข้าม re-render

**ข้อควรระวัง**: ลองลบ `key={book.id}` ออกจากตัวอย่างข้างบนดู — โค้ดยัง **compile ผ่านและ render ผลลัพธ์ที่
หน้าตาเหมือนเดิมทุกประการ** (ตรวจสอบจริงแล้ว — ไม่มี error, ไม่มี warning ใน console เลยด้วยซ้ำ) เพราะ Yew
(ต่างจาก React ที่จะพิมพ์ `Warning: Each child in a list should have a unique "key" prop` ใน console ทันที)
**ไม่เตือนเวลาลืม `key` เลย** — มันจะ diff ตาม "ตำแหน่งในลิสต์" เงียบ ๆ แทน ผลกระทบของการลืม `key` จะไม่เห็นจาก
ข้อมูลที่ render ถูกต้องหรือไม่ (ข้อมูลจะยังถูกต้องเสมอในตัวอย่างนี้ เพราะ list ไม่มีการเพิ่ม/ลบ/สลับลำดับ) แต่
จะเห็นผลเมื่อ list มีการเรียงลำดับใหม่ระหว่าง re-render จริง ๆ (เช่น ผู้ใช้กด sort หรือลบเล่มกลางลิสต์) — ตอน
นั้น Yew อาจ "จับคู่ node เก่า/ใหม่ผิดตัว" ทำให้ state ภายใน component ลูก (ถ้ามี เช่น scroll position หรือ
input ที่ focus อยู่) เพี้ยนไปติดกับ node ผิดใบ แม้ข้อมูลที่แสดงจะยังถูกต้อง — นี่คือเหตุผลที่ควรติดตั้งวินัย
**ใส่ `key` ทุกครั้งที่ render list ด้วย `.map()`** เป็นธรรมเนียม ไม่ต้องรอให้เห็นบั๊กจริงก่อนแล้วค่อยแก้

### 88.7 Component Communication: callback เป็น prop, lifting state up, และ `use_context`

ตัวอย่างที่ผ่านมาส่งข้อมูล**ลง**จาก parent ไปยัง child เท่านั้น (ทางเดียว) — แต่ในแอปจริง เหตุการณ์มักเกิดที่
child แล้วต้องแจ้ง parent ให้ทำอะไรบางอย่าง เช่น ปุ่ม "ยืม" ที่อยู่ใน `BookCard` (child) แต่ข้อมูล "รายการ
หนังสือทั้งหมด" ถูกเก็บอยู่ที่ `BookList` (parent) — `BookCard` เอง**ไม่มีสิทธิ์แก้ state ของ parent ได้ตรง ๆ**
(Rust ownership ไม่อนุญาตให้ child "เอื้อม" ไปแก้ state ของ parent ข้ามขอบเขตของ component แบบพลการอยู่แล้ว)
วิธีแก้คือ pattern ที่เรียกว่า **lifting state up**: parent เป็นเจ้าของ state และส่ง **callback** ลงไปเป็น
prop ให้ child เรียกกลับขึ้นมาเวลามีเหตุการณ์เกิดขึ้น — child ไม่ได้แก้ state เอง มันแค่ "บอก" parent ว่าเกิด
อะไรขึ้น แล้วให้ parent เป็นคนตัดสินใจว่าจะแก้ state ยังไง

```rust
use yew::prelude::*;

#[derive(Clone, PartialEq, Debug)]
struct Book {
    id: u32,
    title: String,
    available: bool,
}

#[derive(Properties, PartialEq)]
struct BookCardProps {
    book: Book,
    // Callback<u32> ที่ parent ส่งลงมา — child จะเรียกมันตอนกดปุ่ม โดยส่ง id ของหนังสือเล่มนี้กลับไป
    on_borrow: Callback<u32>,
}

#[function_component(BookCard)]
fn book_card(props: &BookCardProps) -> Html {
    let book = props.book.clone();
    let on_borrow = props.on_borrow.clone();

    // onclick ของ BookCard "ไม่ได้" แก้ state ของตัวเอง (BookCard ไม่มี state เรื่องความว่าง/ไม่ว่างเลย)
    // มันแค่ .emit(id) เรียก callback ที่ parent ยื่นมาให้ — แปลว่า "ฉันถูกกด แจ้ง parent ด้วย"
    let onclick = Callback::from(move |_| on_borrow.emit(book.id));

    html! {
        <div class="book-card">
            <h3>{ &props.book.title }</h3>
            if props.book.available {
                <button {onclick}>{ "ยืมเล่มนี้" }</button>
            } else {
                <span>{ "ถูกยืมแล้ว" }</span>
            }
        </div>
    }
}

#[function_component(BookList)]
fn book_list() -> Html {
    // state ของรายการหนังสือทั้งหมดอยู่ที่ parent (BookList) จุดเดียว — นี่คือ "single source of truth"
    // (คำที่ React ใช้เรียก pattern นี้ตรง ๆ) BookCard ไม่มี state ของตัวเองเลย มันเป็นแค่ "presentational
    // component" ที่รับข้อมูลมาแสดง แล้วรายงานเหตุการณ์กลับขึ้นไปเท่านั้น
    let books = use_state(|| {
        vec![
            Book { id: 1, title: "The Rust Programming Language".into(), available: true },
            Book { id: 2, title: "Programming Rust".into(), available: true },
        ]
    });

    let on_borrow = {
        let books = books.clone();
        Callback::from(move |id: u32| {
            // แก้ state ของ BookList โดยสร้าง Vec ใหม่ (immutable update — ทวนจาก Part 25-26
            // เรื่อง iterator ที่ไม่แก้ค่าเดิม แต่สร้างค่าใหม่ผ่าน .map())
            let updated: Vec<Book> = books
                .iter()
                .map(|b| {
                    if b.id == id {
                        Book { available: false, ..b.clone() }
                    } else {
                        b.clone()
                    }
                })
                .collect();
            books.set(updated);
        })
    };

    html! {
        <div class="book-list">
            { for books.iter().map(|book| {
                html! {
                    <BookCard
                        key={book.id}
                        book={book.clone()}
                        on_borrow={on_borrow.clone()}
                    />
                }
            }) }
        </div>
    }
}

fn main() {
    yew::Renderer::<BookList>::new().render();
}
```

**ผลจากการรันจริง**: ก่อนกดปุ่ม `document.body.innerHTML`:

```html
<div class="book-list">
  <div class="book-card"><h3>The Rust Programming Language</h3><button>ยืมเล่มนี้</button></div>
  <div class="book-card"><h3>Programming Rust</h3><button>ยืมเล่มนี้</button></div>
</div>
```

หลัง**คลิกปุ่ม "ยืมเล่มนี้" ของเล่มแรกจริง** (ผ่าน headless browser automation) `document.body.innerHTML`
เปลี่ยนเป็น:

```html
<div class="book-list">
  <div class="book-card"><h3>The Rust Programming Language</h3><span>ถูกยืมแล้ว</span></div>
  <div class="book-card"><h3>Programming Rust</h3><button>ยืมเล่มนี้</button></div>
</div>
```

สังเกตว่าเล่มแรกเปลี่ยนจาก `<button>` เป็น `<span>ถูกยืมแล้ว</span>` ส่วนเล่มที่สองยังเป็น `<button>` เดิม —
เหตุการณ์ (คลิก) เกิดขึ้นที่ `BookCard` (child) แต่ state ที่เปลี่ยนจริงอยู่ที่ `BookList` (parent) การไหลของ
ข้อมูลเป็นวงกลม: **props ไหลลง** (`book`, `on_borrow`) → **event ไหลขึ้น** (ผ่าน `.emit()`) → parent ตัดสินใจ
แก้ state → **props ใหม่ไหลลง** อีกรอบ (data flow ทางเดียวเสมอ ไม่มีทางที่ child จะแก้ state ของ parent ข้าม
ขอบเขตแบบไม่ผ่าน callback ได้เลย — ตรงกับหลักการ ownership ของ Rust ที่ Part 6-7 สอนไว้ตรง ๆ)

#### เมื่อ prop drilling เริ่มเจ็บ: `use_context`

pattern ข้างบนใช้ได้ดีเมื่อต้นไม้ component ลึกไม่กี่ชั้น แต่ถ้า `BookCard` ถูกฝังอยู่ลึก 4-5 ชั้น (เช่น
`App` → `Page` → `Sidebar` → `BookList` → `BookCard`) การส่ง `on_borrow` (หรือ state อื่นที่หลายจุดในต้นไม้
ต้องใช้ เช่น "ผู้ใช้ที่ login อยู่ตอนนี้" หรือ "ธีมสี dark/light") ผ่านทุกชั้นเป็น prop จะกลายเป็นปัญหาที่
เรียกว่า **prop drilling** — ทุก component ระดับกลางต้องรับ prop นั้นมาแค่เพื่อ "ส่งต่อ" ไม่ได้ใช้เองเลย

Yew มี `use_context` แก้ปัญหานี้ — สร้าง context หนึ่งจุด แล้ว component *ไหนก็ได้* ในต้นไม้ที่อยู่ใต้
`ContextProvider` นั้นดึงค่าออกมาใช้ตรง ๆ ได้โดยไม่ต้องผ่าน prop ทีละชั้น:

```rust
use std::rc::Rc;
use yew::prelude::*;

// context ที่เก็บ Callback สำหรับยืมหนังสือ — ห่อด้วย Rc เพราะ context ต้อง Clone + PartialEq
// และการ clone Callback ทั้งก้อนซ้ำ ๆ ในทุกชั้นของต้นไม้จะแพงกว่าการ clone Rc ที่ชี้ไปยังก้อนเดียวกัน
#[derive(Clone, PartialEq)]
struct BorrowContext {
    on_borrow: Callback<u32>,
}

#[function_component(BookCardDeep)]
fn book_card_deep(props: &BookCardProps) -> Html {
    // use_context::<T>() คืน Option<T> — None ถ้าไม่มี ContextProvider<T> อยู่เหนือ component นี้
    // ไม่ต้องรับ on_borrow มาเป็น prop เลย ไม่ว่า BookCardDeep จะถูกฝังลึกกี่ชั้นก็ตาม
    let ctx = use_context::<BorrowContext>().expect("ต้องมี BorrowContext provider อยู่เหนือ component นี้");
    let book_id = props.book.id;
    let onclick = Callback::from(move |_| ctx.on_borrow.emit(book_id));

    html! {
        <div class="book-card">
            <h3>{ &props.book.title }</h3>
            <button {onclick}>{ "ยืมเล่มนี้" }</button>
        </div>
    }
}

#[function_component(App)]
fn app() -> Html {
    let on_borrow = Callback::from(|id: u32| web_sys::console::log_1(&format!("ยืมเล่ม id={id}").into()));
    let ctx = BorrowContext { on_borrow };

    html! {
        // ContextProvider<T> ทำให้ context value เข้าถึงได้จาก use_context::<T>() ของ
        // component *ลูกทุกตัว* ที่อยู่ใต้มัน ไม่จำกัดความลึก
        <ContextProvider<BorrowContext> context={ctx}>
            <div>
                // สมมติว่า Sidebar/Page อยู่ตรงนี้กี่ชั้นก็ได้ — BookCardDeep ที่อยู่ลึกแค่ไหน
                // ก็ยังเรียก use_context ดึง on_borrow ออกมาได้ตรง ๆ โดยไม่ต้องผ่าน prop สักตัว
                <BookCardDeep book={Book { id: 1, title: "Rust for Rustaceans".into(), available: true }} />
            </div>
        </ContextProvider<BorrowContext>>
    }
}
```

**ข้อควรระวังเรื่องการเลือกใช้**: `use_context` ไม่ใช่ "ทางแก้ทุกกรณี" — มันเหมาะกับค่าที่เป็น global-ish จริง
ๆ ในต้นไม้ (theme, ข้อมูล user ที่ login, callback ระดับแอป) ถ้าใช้ context สำหรับข้อมูลที่จริง ๆ ควรเป็น
prop ธรรมดา (เช่น `title` ของ `BookCard` เอง) จะทำให้ component นั้น**ทดสอบยากขึ้น** (ต้องมี provider ครอบ
ก่อนถึงจะ render ได้) และ**อ่านยากขึ้น** (มองจาก signature ของ component ไม่รู้แล้วว่ามันต้องการอะไรจากข้าง
นอกบ้าง เพราะ dependency ซ่อนอยู่ใน `use_context` ข้างในฟังก์ชัน) — หลักทั่วไปที่ทีม Yew และ React แนะนำ
ตรงกัน: **ใช้ prop ก่อนเสมอ ใช้ context เมื่อ prop drilling เจ็บจริง ๆ เท่านั้น**

### 88.8 Fetching Data: เชื่อม Yew เข้ากับ REST API ที่เขียนด้วย Axum

ทุกตัวอย่างที่ผ่านมาใช้ข้อมูล mock ที่ฝังอยู่ในโค้ด Rust ตรง ๆ — แอปจริงต้องดึงข้อมูลจากเซิร์ฟเวอร์ นี่คือจุดที่
เชื่อมกลับไปยัง**โมดูล 4 ทั้งโมดูล** (Part 62-85) ตรง ๆ: ถ้าคุณทำ Tasks API หรือ Books API ด้วย Axum ไว้ตาม
Part 62-64 แล้ว Yew frontend ที่จะเขียนในหัวข้อนี้สามารถยิง request ไปยัง API ตัวนั้นได้เลยทันที ไม่ต้องแก้
โค้ดฝั่งเซิร์ฟเวอร์อะไรเพิ่ม (ยกเว้นเรื่อง CORS ซึ่งจะอธิบายด้านล่าง)

crate ที่ใช้คือ **`gloo-net`** — ส่วนหนึ่งของตระกูล `gloo` (ที่หัวข้อ 88.1 พิสูจน์ไปแล้วว่า Yew เองก็พึ่งพา
`gloo-utils` อยู่) `gloo-net` ให้ HTTP client ที่ใช้งานง่ายบน `web-sys`'s `fetch` API (คือ `window.fetch()`
ของเบราว์เซอร์ตัวเดียวกับที่ JavaScript ใช้ — ไม่ใช่ HTTP client ที่ทำ TCP connection เองแบบ `reqwest` บน
เซิร์ฟเวอร์ เพราะใน browser sandbox โค้ด WASM **ไม่มีสิทธิ์เปิด raw socket เองได้เลย** ทุก network request
ต้องผ่าน browser API เท่านั้น) นี่คือเหตุผลที่ `gloo-net` เป็นตัวเลือกที่ idiomatic กว่า `reqwest` สำหรับงาน
frontend ใน WASM ในปัจจุบัน — `reqwest` รองรับ target `wasm32-unknown-unknown` ผ่าน feature พิเศษก็จริง แต่
ภายในมันก็ยังต้องเรียก `fetch` ผ่าน `web-sys` อยู่ดี (มันไม่มีทางเลือกอื่นในเบราว์เซอร์) การใช้ `gloo-net`
ตรง ๆ จึงบางกว่า ไม่มี abstraction layer ซ้อนที่ออกแบบมาสำหรับ native HTTP client เป็นหลัก และเป็น crate ที่
ทีม Yew core เขียน/ดูแลเองในตระกูล `gloo` เดียวกัน — เอกสารและตัวอย่างของ Yew ทางการเลือก `gloo-net` เป็น
ค่าเริ่มต้นในทุกตัวอย่างที่เกี่ยวกับ networking

ติดตั้ง:

```bash
cargo add gloo-net --features json
cargo add serde --features derive
cargo add wasm-bindgen-futures
```

(`wasm-bindgen-futures` คือ crate ที่ Part 87/46-47 แนะนำไปแล้วสำหรับรัน `Future` บน event loop ของเบราว์เซอร์
ผ่าน `spawn_local` — โค้ดด้านล่างจะใช้ตรง ๆ อีกครั้งตอนเรียก fetch แบบ async ข้าง `use_effect_with`)

**ฝั่งเซิร์ฟเวอร์** — Axum API ขั้นต่ำที่สุดสำหรับตัวอย่างนี้ (โครงสร้างเดียวกับที่ Part 62 สอน เพิ่มแค่
`tower-http::cors::CorsLayer` ที่ Part 65 แนะนำไว้):

```rust
// เซิร์ฟเวอร์ Axum — สมมติว่านี่คือส่วนหนึ่งของ Books API ที่คุณสร้างไว้แล้วในโมดูล 4
use axum::{routing::get, Json, Router};
use serde::Serialize;
use tower_http::cors::CorsLayer;

#[derive(Clone, Serialize)]
struct Book {
    id: u32,
    title: String,
    author: String,
}

async fn list_books() -> Json<Vec<Book>> {
    Json(vec![
        Book { id: 1, title: "The Rust Programming Language".into(), author: "Steve Klabnik".into() },
        Book { id: 2, title: "Programming Rust".into(), author: "Jim Blandy".into() },
    ])
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/api/books", get(list_books))
        // CorsLayer::permissive() อนุญาตทุก origin — ใช้ได้เฉพาะตอนพัฒนา (dev) เท่านั้น
        // เหตุผลที่ต้องมี: เว็บเบราว์เซอร์บังคับ same-origin policy — trunk serve รันที่ port 8080
        // แต่ Axum server รันที่ port อื่น (เช่น 3000) เบราว์เซอร์เห็นว่าเป็นคนละ "origin"
        // (scheme+host+port ต้องตรงกันหมดถึงจะนับเป็น origin เดียวกัน) จึงบล็อก fetch request
        // ข้าม origin โดย default ด้วยเหตุผลด้าน security — CORS header คือสิ่งที่ฝั่งเซิร์ฟเวอร์
        // ต้องส่งกลับมาเพื่อ "อนุญาต" การข้าม origin นี้อย่างชัดเจน
        .layer(CorsLayer::permissive());

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**ฝั่ง Yew** — component ที่ fetch จาก endpoint นี้ตอน mount:

```rust
use gloo_net::http::Request;
use serde::Deserialize;
use yew::prelude::*;

#[derive(Clone, PartialEq, Deserialize, Debug)]
struct Book {
    id: u32,
    title: String,
    author: String,
}

#[function_component(BookListRemote)]
fn book_list_remote() -> Html {
    // state สามชั้นที่ต้องมี: กำลังโหลดอยู่ไหม / โหลดสำเร็จแล้วได้อะไร / โหลดพลาดไหม
    // ใช้ Option<Vec<Book>> แทนสถานะ "ยังไม่มีข้อมูล" กับ Result สำหรับ error message
    let books: UseStateHandle<Option<Result<Vec<Book>, String>>> = use_state(|| None);

    {
        let books = books.clone();
        // deps เป็น () แปลว่า effect นี้รันแค่ครั้งเดียวตอน component mount เท่านั้น
        // (เทียบเท่า useEffect(() => {...}, []) ของ React ที่มี dependency array ว่าง)
        use_effect_with((), move |_| {
            // wasm_bindgen_futures::spawn_local คือฟังก์ชันที่ Part 87/Part 46-47 สอนไว้แล้ว
            // สำหรับรัน Future บน JS event loop ของเบราว์เซอร์ (ไม่มี Tokio runtime ในเบราว์เซอร์
            // — WASM ฝั่ง client ใช้ event loop ของเบราว์เซอร์เอง ไม่ใช่ #[tokio::main])
            wasm_bindgen_futures::spawn_local(async move {
                let result = Request::get("http://127.0.0.1:3000/api/books")
                    .send()
                    .await;

                let parsed = match result {
                    Ok(response) => response
                        .json::<Vec<Book>>()
                        .await
                        .map_err(|e| format!("แปลง JSON ไม่ได้: {e}")),
                    Err(e) => Err(format!("เชื่อมต่อเซิร์ฟเวอร์ไม่ได้: {e}")),
                };
                books.set(Some(parsed));
            });
            // ไม่ return cleanup function เพราะไม่มีอะไรต้องเก็บกวาดหลัง fetch เสร็จ
            || ()
        });
    }

    match &*books {
        None => html! { <p>{ "กำลังโหลดข้อมูล..." }</p> },
        Some(Err(err)) => html! { <p class="error">{ format!("เกิดข้อผิดพลาด: {err}") }</p> },
        Some(Ok(list)) => html! {
            <ul>
                { for list.iter().map(|book| html! {
                    <li key={book.id}>{ format!("{} — {}", book.title, book.author) }</li>
                }) }
            </ul>
        },
    }
}

fn main() {
    yew::Renderer::<BookListRemote>::new().render();
}
```

**ผลจากการรันจริง**: ยืนยันด้วยการตั้งเซิร์ฟเวอร์ Axum จริงที่ `127.0.0.1:3000` (ใช้ Cargo.toml ที่มี `axum`,
`tokio`, `tower-http` ตามโมดูล 4) พร้อมกับ build แอป Yew ด้านบนด้วย `trunk build` แล้วเสิร์ฟที่
`127.0.0.1:8181` แล้วเปิด headless browser ไปที่หน้า Yew พร้อมสุ่มอ่าน `document.body.innerHTML` ทุก ๆ 20
มิลลิวินาที — เห็นลำดับสถานะจริงตามที่คาด:

```
t=0ms:  (ยังโหลด WASM module อยู่ — <body> ยังไม่มีอะไรจาก component เลย)
t=20ms: <p>กำลังโหลดข้อมูล...</p>
t=40ms: <ul><li>The Rust Programming Language — Steve Klabnik</li><li>Programming Rust — Jim Blandy</li></ul>
```

จับได้ชัดว่า state ไล่จาก `None` (แสดง "กำลังโหลดข้อมูล...") ไปเป็น `Some(Ok(...))` (แสดงรายการ) ภายในเวลาไม่ถึง
50 มิลลิวินาทีสำหรับ request ไปยัง `localhost` — และ `document.body.innerHTML` สุดท้ายคือ:

```html
<ul>
  <li>The Rust Programming Language — Steve Klabnik</li>
  <li>Programming Rust — Jim Blandy</li>
</ul>
```

ตรงกับข้อมูลที่ endpoint `/api/books` ของ Axum server ส่งกลับมาทุกประการ (ทดสอบแยกด้วย `curl
http://127.0.0.1:3000/api/books` ได้ JSON `[{"id":1,"title":"The Rust Programming Language","author":"Steve
Klabnik"},{"id":2,"title":"Programming Rust","author":"Jim Blandy"}]` ตรงกับที่ `Book` struct ฝั่ง Yew
deserialize ออกมาได้พอดี) นี่คือหลักฐานที่ชัดที่สุดว่า **Yew frontend คุยกับ Axum backend ที่คุณสร้างไว้แล้ว
ตั้งแต่โมดูล 4 ได้ตรง ๆ โดยไม่ต้องแก้อะไรฝั่งเซิร์ฟเวอร์เลย** (ยกเว้นเพิ่ม CORS layer ตอน dev)

ถ้าลอง**ปิดเซิร์ฟเวอร์ Axum จริง** (kill process ทิ้ง) แล้ว reload หน้า Yew จะเห็น branch ของ error ทำงานจริง
เช่นกัน — `document.body.innerHTML` ที่ได้:

```html
<p class="error">เกิดข้อผิดพลาด: เชื่อมต่อเซิร์ฟเวอร์ไม่ได้: TypeError: Failed to fetch</p>
```

พร้อม console error ที่เบราว์เซอร์พิมพ์แยกออกมาอีกบรรทัด (จับได้จริงจาก console ของ headless browser):

```
[console.error] Failed to load resource: net::ERR_CONNECTION_REFUSED
```

ข้อความ error ฝั่งใน (`TypeError: Failed to fetch`) มาจาก JS `fetch` API ตัวจริงที่ Chromium คืนมา ผ่านชั้นของ
`gloo-net` ตรง ๆ (ข้อความอาจต่างกันเล็กน้อยระหว่างเบราว์เซอร์ — Firefox คืนข้อความว่า `NetworkError when
attempting to fetch resource.` แทน แต่เป็น `TypeError` ตัวเดียวกันในทั้งสองเบราว์เซอร์) ยืนยันอีกครั้งว่า
`gloo-net` ไม่ได้ "ปลอมแปลง" พฤติกรรมของ `fetch` มันแค่ห่อ API ที่มีอยู่แล้วให้เขียนเป็น `async`/`await` สไตล์
Rust ได้สะดวกขึ้นเท่านั้น

### 88.9 Routing หลายหน้าด้วย `yew-router`

แอปที่ผ่านมาทั้งบทเป็น **single page** — มีแค่ view เดียว แอปจริงมักต้องมีหลายหน้า (list ↔ detail, home ↔
about ฯลฯ) โดยที่ URL ในแถบ address bar เปลี่ยนตามจริง (กด back/forward ของเบราว์เซอร์แล้วต้องกลับหน้าที่ถูก
ต้อง, refresh หน้าแล้วต้องอยู่หน้าเดิม) — **`yew-router`** คือ crate แยกที่ทีม Yew ดูแลสำหรับงานนี้ (แยกจาก
`yew` core เพราะไม่ใช่ทุกแอปต้องการ routing — SPA ที่มีหน้าเดียวจริง ๆ ไม่ต้องพึ่ง crate นี้เลย)

```bash
cargo add yew-router
```

นิยาม route ทั้งหมดของแอปด้วย enum ที่ derive `Routable`:

```rust
use yew::prelude::*;
use yew_router::prelude::*;

// enum ที่แทนทุกหน้าของแอป — derive Routable คือ macro ของ yew-router (คล้าย Properties ของ yew เอง
// เป็น procedural macro ที่ generate การแปลง enum variant ↔ URL path ให้อัตโนมัติ)
#[derive(Clone, Routable, PartialEq)]
enum Route {
    #[at("/")]
    BookListPage,
    // {id} คือ path parameter — จับคู่กับ segment ของ URL แล้วส่งเป็น field ของ variant
    #[at("/books/:id")]
    BookDetailPage { id: u32 },
    #[not_found]
    #[at("/404")]
    NotFound,
}

// ฟังก์ชันที่ map จาก Route ไปเป็น Html ของหน้านั้น — เรียกว่า "switch function"
fn switch(route: Route) -> Html {
    match route {
        Route::BookListPage => html! { <BookListPage /> },
        Route::BookDetailPage { id } => html! { <BookDetailPage id={id} /> },
        Route::NotFound => html! { <h1>{ "404 — ไม่พบหน้านี้" }</h1> },
    }
}

#[derive(Clone, PartialEq, Debug)]
struct Book {
    id: u32,
    title: String,
}

fn sample_books() -> Vec<Book> {
    vec![
        Book { id: 1, title: "The Rust Programming Language".into() },
        Book { id: 2, title: "Programming Rust".into() },
    ]
}

#[function_component(BookListPage)]
fn book_list_page() -> Html {
    let books = sample_books();
    html! {
        <div>
            <h1>{ "รายการหนังสือทั้งหมด" }</h1>
            <ul>
                { for books.iter().map(|book| html! {
                    <li key={book.id}>
                        // <Link<Route>> คือ component ของ yew-router ที่ render <a> ตัวจริง
                        // แต่แทนที่จะ reload หน้าทั้งหน้าแบบ <a href> ธรรมดา มันดัก event คลิก
                        // แล้วเปลี่ยน route ผ่าน History API ของเบราว์เซอร์ (pushState) เอง —
                        // ทำให้ SPA รู้สึกเร็วเหมือนแอปเดียว ไม่มีการ reload หน้าเว็บทั้งหน้าเลย
                        <Link<Route> to={Route::BookDetailPage { id: book.id }}>
                            { &book.title }
                        </Link<Route>>
                    </li>
                }) }
            </ul>
        </div>
    }
}

#[derive(Properties, PartialEq)]
struct BookDetailPageProps {
    id: u32,
}

#[function_component(BookDetailPage)]
fn book_detail_page(props: &BookDetailPageProps) -> Html {
    let books = sample_books();
    let book = books.iter().find(|b| b.id == props.id);

    html! {
        <div>
            { match book {
                Some(b) => html! {
                    <>
                        <h1>{ &b.title }</h1>
                        <Link<Route> to={Route::BookListPage}>{ "← กลับไปหน้ารายการ" }</Link<Route>>
                    </>
                },
                None => html! { <p>{ "ไม่พบหนังสือเล่มนี้" }</p> },
            } }
        </div>
    }
}

#[function_component(App)]
fn app() -> Html {
    html! {
        // BrowserRouter ใช้ HTML5 History API (pushState/popState) จัดการ URL จริงในแถบ address bar
        // (มี HashRouter เป็นทางเลือกที่ใช้ URL fragment #/path แทน — ใช้ตอนไม่มีสิทธิ์ตั้งค่า
        // เซิร์ฟเวอร์ให้ fallback ทุก path มาที่ index.html เดียวกัน)
        <BrowserRouter>
            // Switch<Route> คือจุดที่จับคู่ URL ปัจจุบันกับ Route enum แล้วเรียก switch function
            <Switch<Route> render={switch} />
        </BrowserRouter>
    }
}

fn main() {
    yew::Renderer::<App>::new().render();
}
```

`trunk serve` มี dev server ที่ตั้ง fallback มาที่ `index.html` ให้อัตโนมัติสำหรับทุก path ที่ไม่ตรงกับไฟล์
static จริง (จำเป็นสำหรับ `BrowserRouter` — ถ้า refresh หน้าที่ `/books/2` ตรง ๆ เบราว์เซอร์จะขอไฟล์ที่ path
นั้นจากเซิร์ฟเวอร์ ถ้าเซิร์ฟเวอร์ไม่ fallback มาที่ `index.html` ก็จะได้ 404 จริงจากเซิร์ฟเวอร์ก่อนที่ WASM
จะได้โอกาสรันเสียอีก) เวลา deploy ไป production ต้องตั้งค่า static file server ปลายทาง (nginx, Netlify,
Vercel ฯลฯ) ให้ทำ fallback แบบเดียวกันด้วยตัวเอง — นี่คือ operational detail ที่มักถูกมองข้ามตอน deploy SPA
ที่มี client-side routing ครั้งแรก

**ผลจากการรันจริง**: เปิดหน้าแรก (`/`) ได้ `document.body.innerHTML`:

```html
<div>
  <h1>รายการหนังสือทั้งหมด</h1>
  <ul>
    <li><a href="/books/1">The Rust Programming Language</a></li>
    <li><a href="/books/2">Programming Rust</a></li>
  </ul>
</div>
```

คลิก link "The Rust Programming Language" จริง (ผ่าน headless browser automation คลิก `<a>` แท็กนั้นตรง ๆ) —
URL ในเบราว์เซอร์เปลี่ยนเป็น `http://127.0.0.1:8080/books/1` (ยืนยันด้วย `window.location.pathname` ที่อ่าน
ได้จริงเป็น `/books/1`) **โดยไม่มี network request ไปขอหน้าใหม่จากเซิร์ฟเวอร์เลย** (ตรวจสอบว่าไม่มี full page
reload เกิดขึ้นจริงผ่าน network log ของ headless browser) และ `document.body.innerHTML` เปลี่ยนเป็น:

```html
<div>
  <h1>The Rust Programming Language</h1>
  <a href="/">← กลับไปหน้ารายการ</a>
</div>
```

นี่คือพฤติกรรมของ SPA routing ตัวจริง: URL เปลี่ยน, DOM เปลี่ยนตาม route, แต่ไม่มีการ reload หน้าเว็บทั้งหน้า
— ทั้งหมดเกิดขึ้นภายใน WASM module เดียวที่โหลดมาแค่ครั้งเดียวตอนเปิดหน้าแรก

### 88.10 Yew เทียบกับ wasm-bindgen ดิบ ๆ, Leptos, และ Dioxus: มุมมองที่ตรงไปตรงมา

ก่อนปิดบท ควรวางตำแหน่งของ Yew ให้ชัดเจนเทียบกับสิ่งที่เรียนมาแล้วและสิ่งที่กำลังจะเรียนต่อ — **โดยไม่รีบ
ตัดสินว่า framework ไหน "ดีที่สุด"** เพราะการตัดสินแบบนั้นจะยังไม่เป็นธรรมจนกว่าคุณจะได้เห็น Leptos (Part 89)
และ Dioxus (Part 90) ก่อน บทนี้ให้แค่ข้อเท็จจริงที่สังเกตได้แล้วตอนนี้:

**Yew เทียบกับ wasm-bindgen/web-sys ดิบ ๆ (Part 87)**: หัวข้อ 88.1 แสดงให้เห็นแล้วว่า Yew ไม่ได้แข่งกับ Part
87 — มันสร้างอยู่บนสิ่งที่ Part 87 สอน ให้เลือกใช้ตามขนาดของงาน: ถ้าแค่ต้องการปะ interactivity เล็ก ๆ เข้า
หน้า HTML ที่มีอยู่แล้ว (เช่น widget คำนวณราคา หรือ animation เล็ก ๆ) `wasm-bindgen`/`web-sys` ตรง ๆ แบบ Part
87 อาจเบากว่าและตรงเป้ากว่า แต่ถ้าจะสร้างแอปทั้งหน้าที่มี state/component ซับซ้อนหลายสิบตัว การจัดการ DOM มือ
จะกลายเป็นภาระที่โตเร็วกว่าความซับซ้อนของแอปจริง (ไม่เป็นเส้นตรง — โตแบบ exponential เมื่อ component ต้อง
sync กันหลายตัว) framework แบบ Yew จึงคุ้มค่ากว่าอย่างชัดเจนตั้งแต่ขนาดกลางขึ้นไป

**จุดยืนของ Yew ใน ecosystem**: Yew เป็น framework ตัวแรกที่ทำ component-based UI สำหรับ Rust/WASM ให้เป็นที่
รู้จักอย่างกว้างขวาง (เริ่มโครงการมาตั้งแต่ปี 2018 — เก่าแก่ที่สุดในสามตัวที่คอร์สนี้จะสอน) ทำให้มันมี
ecosystem ที่โตที่สุดในบรรดา Rust frontend framework ทั้งหมด ณ ตอนนี้: จำนวน crate เสริม (`yew-router`,
`yew-hooks`, `yewdux` สำหรับ state management แบบ global), จำนวนตัวอย่าง/tutorial ในโลกจริง, และจำนวนโปรเจกต์
production ที่ใช้งานจริงมากที่สุดเมื่อเทียบกับ Leptos และ Dioxus — นี่คือข้อได้เปรียบที่ชัดเจนและเป็นรูปธรรม
ของ Yew ที่ควรให้เครดิตตรง ๆ ไม่ว่าจะชอบสถาปัตยกรรมภายในของมันหรือไม่ก็ตาม

**สถาปัตยกรรมภายในที่ต่างจากสองตัวถัดไป**: กลไกที่ Yew ใช้อัปเดต UI คือ **virtual DOM diffing** (อธิบายไป
แล้วในหัวข้อ 88.1) — ทุกครั้งที่ state เปลี่ยน component ทั้งฟังก์ชันถูกเรียกใหม่ ได้ `Html` tree ใหม่ทั้งก้อน
มาเทียบกับก้อนเก่า นี่คือ pattern เดียวกับที่ React ใช้ (และ Vue 2, Preact) — Part 89 จะสอน **Leptos** ซึ่งใช้
กลไกที่เรียกว่า **fine-grained reactivity** (คล้าย SolidJS ในโลก JavaScript) ที่ *ไม่* re-run ฟังก์ชัน
component ทั้งก้อนเวลา state เปลี่ยน แต่ผูก "signal" ตรงเข้ากับ DOM node ที่เกี่ยวข้องเป๊ะ ๆ ตั้งแต่ตอน setup
ครั้งเดียว — ต่างเชิงปรัชญาการออกแบบอย่างมีนัยสำคัญจาก Yew ส่วน Part 90 จะสอน **Dioxus** ที่ใช้ virtual DOM
คล้าย Yew แต่ออกแบบ API ให้ใกล้เคียง React มากขึ้นอีกขั้น (รวมถึงรองรับ native desktop/mobile app ผ่าน
renderer ตัวเดียวกัน ไม่ใช่แค่ web) — ทั้งสามกลไกนี้มี trade-off คนละแบบในเรื่อง performance, ergonomics, และ
ขนาดของ `.wasm` bundle ที่ได้ ซึ่งจะประเมินให้เห็นภาพชัดเจนกว่านี้อีกครั้งหลังจากที่คุณได้ลงมือเขียนโค้ดจริง
กับทั้งสามตัวแล้วในบทถัดไป

ตารางสรุปสิ่งที่**สังเกตได้แล้วตอนนี้** (ยังไม่ใช่การตัดสิน — แค่ข้อเท็จจริงเชิงสถาปัตยกรรมและ ecosystem ที่
เห็นแล้วจากบทนี้):

| ประเด็น | wasm-bindgen ดิบ (Part 87) | Yew (บทนี้) |
|---|---|---|
| ปีเริ่มโครงการ | (ไม่ใช่ framework — เป็น binding layer) | 2018 |
| กลไกอัปเดต UI | ไม่มี — คุณคุม DOM เอง 100% | Virtual DOM diffing |
| Component model | ไม่มีในตัว (เขียนฟังก์ชันเองได้ แต่ไม่มี convention กลาง) | function component + hooks (แบบ React) |
| Routing | ต้องเขียนเอง (จับ `popstate` event เอง) | `yew-router` (official) |
| ขนาด ecosystem เสริม | เล็ก (เป็น primitive layer เอง) | ใหญ่ที่สุดในสาม Rust framework |
| เหมาะกับ | widget เล็ก ๆ, ปะ WASM เข้าโปรเจกต์เดิม | SPA ขนาดกลางถึงใหญ่ |
| ขนาด `.wasm` (release, gzip) ของ Hello World | ขึ้นกับโค้ดที่เขียนเอง — อาจเล็กกว่ามากถ้าเขียนบางมาก | ~94 KiB (วัดจริงในหัวข้อ 88.2 — รวม runtime ของ Yew ทั้งชุดแล้ว) |

ตัวเลข ~94 KiB ที่วัดได้จริงในหัวข้อ 88.2 คือ**ต้นทุนคงที่**ของการเลือกใช้ Yew (หรือ framework ที่ทำงานบน
แนวคิดเดียวกัน) — ไม่ว่าแอปจะเล็กหรือใหญ่แค่ไหน runtime ของ virtual DOM diffing/hooks system ต้องถูกส่งไปให้
เบราว์เซอร์ทุกครั้งที่โหลดหน้าเว็บครั้งแรก (แม้ browser cache ไว้ให้ครั้งถัดไปก็ตาม) เทียบกับ `wasm-bindgen`
ดิบที่ Part 87 สอน ซึ่งไม่มี "runtime" ส่วนกลางแบบนี้เลย — ขนาดไฟล์ขึ้นกับโค้ดที่คุณเขียนเองตรง ๆ เท่านั้น จุด
นี้เป็นส่วนหนึ่งของ trade-off ที่ต้องชั่งน้ำหนักตอนเลือก framework จริง (ตัวเลขที่แม่นยำของ Leptos และ Dioxus
สำหรับงานเดียวกันจะเห็นได้ใน Part 89-90 เพื่อเทียบกันตรง ๆ)

การเปรียบเทียบแบบสามทาง (Yew vs Leptos vs Dioxus) เต็มรูปแบบจะรอไปถึงหลัง Part 90 — เมื่อคุณได้ลงมือเขียน
โค้ดจริงกับทั้งสามตัวและเห็น trade-off ด้วยตัวเองแล้ว การตัดสินตอนนี้ (ก่อนเห็นอีกสองตัว) จะไม่เป็นธรรมต่อ
framework ที่ยังไม่ได้เรียน

### 88.11 ทดสอบ Component ของ Yew: จุดที่เชื่อมกับ Part 32-33

บทนี้ยังไม่ได้พูดถึงการเขียน automated test สำหรับ component เลย — คอร์สนี้แนะนำ unit test และ integration
test มาแล้วเต็มรูปแบบใน Part 32-33 แต่ test เหล่านั้นรันบน target `x86_64` ปกติ (หรือ target ของเครื่องที่ใช้
พัฒนา) ไม่ใช่ `wasm32-unknown-unknown` — การทดสอบ component ของ Yew โดยเฉพาะ (เช่น "component นี้ render
`<button>` ถูกไหม", "คลิกแล้ว state เปลี่ยนจริงไหม") ต้องรันบน WASM จริง เพราะพึ่งพา `web-sys`/DOM API ที่ไม่มี
อยู่ใน target ปกติ — เครื่องมือสำหรับงานนี้คือ `wasm-bindgen-test` (มาจากตระกูล `wasm-bindgen` ที่ Part 87 สอน
ตรง ๆ) ซึ่งให้ attribute macro `#[wasm_bindgen_test]` แทนที่ `#[test]` ปกติของ Part 32 แล้วรันผ่าน
`wasm-bindgen-test-runner` ที่เปิด headless browser จริงขึ้นมา execute ทุก test case (แนวคิดเดียวกันกับที่บทนี้
เองใช้ตรวจสอบทุกตัวอย่างที่เห็นมาทั้งบท — เพียงแต่บทนี้ใช้ Playwright ควบคุม browser จากภายนอก ในขณะที่
`wasm-bindgen-test` ทำสิ่งเดียวกันแต่ผนวกเข้ากับ `cargo test` โดยตรง) — Yew เองมี crate เสริมชื่อ `yew::
functional::test` (และ helper อื่นในบางเวอร์ชัน) สำหรับ render component ลง DOM จำลองแล้วตรวจสอบผลลัพธ์แบบ
programmatic โดยไม่ต้องเปิด browser จริงทุกครั้งที่ CI รัน — เนื้อหาการเขียน test เต็มรูปแบบสำหรับเว็บแอปจะถูก
รวบรวมให้ครบใน **Part 95 (Testing Web Applications แบบครบวงจร)** ซึ่งจะกลับมาที่ Yew (และ Leptos/Dioxus)
อีกครั้งพร้อมกับเทคนิค unit/integration/e2e เต็มชุด บทนี้ขอให้แค่รู้จักชื่อเครื่องมือและตำแหน่งของมันไว้ก่อน

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม derive `PartialEq` บน props struct**

```rust
#[derive(Properties)]  // ลืม PartialEq!
struct BookCardProps {
    book: Book,
}
```

error จริงจาก compiler (จับได้ตั้งแต่จุดที่ derive macro `Properties` ทำงาน ไม่ต้องรอไปถึง
`#[function_component]`):

```
error[E0277]: can't compare `BookCardProps` with `BookCardProps`
  --> src/main.rs:9:10
   |
 9 | #[derive(Properties)]
   |          ^^^^^^^^^^ no implementation for `BookCardProps == BookCardProps`
   |
   = help: the trait `PartialEq` is not implemented for `BookCardProps`
note: required by a bound in `yew::Properties`
  --> .../yew-0.23.0/src/html/component/properties.rs:6:23
   |
 6 | pub trait Properties: PartialEq {
   |                       ^^^^^^^^^ required by this bound in `Properties`
help: consider annotating `BookCardProps` with `#[derive(PartialEq)]`
   |
10 + #[derive(PartialEq)]
11 | struct BookCardProps {
   |
```

สังเกตว่า trait `Properties` ของ Yew เอง**ประกาศ** `PartialEq` เป็น supertrait ตรง ๆ (`pub trait Properties:
PartialEq`) — นี่คือเหตุผลที่ compiler ฟ้องตั้งแต่บรรทัดที่ derive `Properties` เลย ไม่ต้องรอให้ไปถึงจุดที่ใช้
component จริง Yew ต้องเทียบ props เก่ากับ props ใหม่ด้วย `PartialEq` เพื่อรู้ว่าควร re-render component ลูก
หรือไม่ (memoization อัตโนมัติ) — วิธีแก้คือเติม `PartialEq` เข้าไปใน derive list เสมอ: `#[derive(Properties,
PartialEq)]` (compiler ยังใจดีพอที่จะแนะนำ fix ให้ตรง ๆ ผ่าน `help:` ด้วย) และทุก field ของ props (รวม field
ของ `Book` ที่อยู่ข้างในด้วย) ก็ต้อง `PartialEq` เช่นกัน — ถ้า field ไหนเป็น type ที่ derive `PartialEq` ไม่ได้
(เช่น `f64` ที่มี `NaN`) ต้อง implement เองหรือเปลี่ยนชนิดข้อมูล

**2. ใช้ `.clone()` ผิดจุดใน closure ของ callback ทำให้ borrow checker ฟ้อง หรือ state ไม่อัปเดต**

```rust
let count = use_state(|| 0);
let onclick = Callback::from(move |_| {
    count.set(*count + 1); // ผิด! count ถูก move เข้า closure ไปแล้ว แต่ยังใช้ count ข้างนอกซ้ำ
});
html! { <div>
    <button {onclick}>{"+"}</button>
    <p>{*count}</p> // error: count ถูก move ไปแล้วที่ onclick
</div> }
```

error จริง (คัดมาเฉพาะส่วนสำคัญ — compiler พิมพ์ยาวกว่านี้):

```
error[E0382]: borrow of moved value: `count`
   --> src/main.rs:11:14
    |
  5 |     let count = use_state(|| 0);
    |         ----- move occurs because `count` has type `UseStateHandle<i32>`, which does not implement the `Copy` trait
  6 |     let onclick = Callback::from(move |_| {
    |                                  -------- value moved into closure here
  7 |         count.set(*count + 1);
    |         ----- variable moved due to use in closure
...
 11 |         <p>{*count}</p>
    |              ^^^^^ value borrowed here after move
    |
    = note: borrow occurs due to deref coercion to `i32`
help: consider cloning the value before moving it into the closure
    |
  6 ~     let value = count.clone();
  7 ~     let onclick = Callback::from(move |_| {
  8 ~         value.set(*count + 1);
    |
```

compiler ระบุชัดว่า `UseStateHandle<i32>` **ไม่ implement `Copy`** จึงถูก move เข้า closure ทั้งตัว และยังใจดี
เสนอ fix ที่ตรงกับ pattern มาตรฐานของ Yew มาให้เลย (`help:` ด้านบน) — วิธีแก้ที่ Yew ใช้เป็นธรรมเนียมทั่วทั้ง
ecosystem: **clone ก่อนแล้วค่อย move ตัว clone เข้า closure** — เขียน `let count = count.clone();` ในบรรทัดใหม่
ก่อนสร้าง `Callback::from(move |_| ...)` (มัก shadow ชื่อตัวแปรเดิมในบล็อกแยกด้วย `{ ... }` เพื่อไม่ให้ตัวแปรที่
clone มาไปรบกวนโค้ดส่วนอื่น) — เพราะ `UseStateHandle<T>` clone ถูกมาก (แค่ increment `Rc` reference count) การ
clone ที่ดูเหมือน "เยอะ" ในโค้ด Yew จึงไม่ใช่ปัญหา performance จริง

**3. เรียก `.await` ตรง ๆ ในตัว closure ของ `use_effect_with` โดยไม่ผ่าน `spawn_local`**

closure ที่ส่งเข้า `use_effect_with`/`use_effect` เป็น **closure ธรรมดา ไม่ใช่ `async` closure** (ทวนจาก Part
46-47 เรื่อง async/await: `.await` ใช้ได้เฉพาะใน `async fn`/`async` block เท่านั้น) มือใหม่ที่เพิ่งเรียน fetch
ข้อมูล (หัวข้อ 88.8) มักลืมจุดนี้แล้วเขียนแบบนี้:

```rust
use_effect_with((), move |_| {
    let resp = Request::get("http://127.0.0.1:3000/api/books").send().await; // ผิด!
    || ()
});
```

error จริง:

```
error[E0728]: `await` is only allowed inside `async` functions and blocks
 --> src/main.rs:7:75
  |
6 |     use_effect_with((), move |_| {
  |                         -------- this is not `async`
7 |         let resp = Request::get("http://127.0.0.1:3000/api/books").send().await;
  |                                                                           ^^^^^ only allowed inside `async` functions and blocks
```

วิธีแก้คือห่อ `.await` ทั้งก้อนด้วย `wasm_bindgen_futures::spawn_local(async move { ... })` แบบที่หัวข้อ 88.8
สอน — closure ของ `use_effect_with` เองยังเป็น closure ธรรมดา (sync) เสมอ มันแค่ "เปิด" async task แยกออกไป
รันบน JS event loop ของเบราว์เซอร์เท่านั้น ไม่ได้ `.await` อะไรตรง ๆ ในตัวมันเอง

**4. ใส่ URL ของ API เป็น absolute path ผิด scheme ตอน deploy จริง แล้วโดน CORS/Mixed Content บล็อก**

โค้ดตัวอย่างในหัวข้อ 88.8 hardcode `"http://127.0.0.1:3000/api/books"` ไว้ตรง ๆ เพื่อความง่ายตอนพัฒนา แต่ถ้า
deploy frontend จริงไปที่ domain ที่ใช้ `https://` แล้ว fetch ไป `http://` (ไม่ใช่ `https://`) เบราว์เซอร์จะ
บล็อกด้วย **Mixed Content policy** ทันที — error ที่เห็นจริงใน console ของเบราว์เซอร์ (ไม่ใช่ error จาก Rust
compiler แต่เป็น runtime error ฝั่งเบราว์เซอร์ ที่ต้องรู้จักเพราะเกี่ยวกับ deployment โดยตรง):

```
Mixed Content: The page at 'https://myapp.example/' was loaded over HTTPS, but requested an insecure
resource 'http://api.example/api/books'. This request has been blocked; the content must be served
over HTTPS.
```

วิธีแก้ทั่วไปคือใช้ relative path (`/api/books`) แล้วให้ reverse proxy ฝั่งเดียวกัน (nginx หรือ Axum เอง)
ทำหน้าที่ route ไปยัง backend จริง หรืออ่าน base URL ของ API จาก environment variable ที่ตั้งตอน build ผ่าน
`env!()` macro (ทวนจาก Part 35 เรื่อง Cargo advanced/build script) แทนการ hardcode

**5. สับสนว่า `html!` เป็น "HTML string" แล้วพยายามใส่ตัวแปร Rust แบบ string interpolation ตรง ๆ**

```rust
let name = "Rust";
html! {
    <p>สวัสดี {name}</p>  // ผิด! ลืม wrap ด้วย { } รอบข้อความ literal
}
```

error จริง (สั้นและตรงประเด็นกว่าที่คาด — `html!` เป็น proc macro ที่ parse token tree ของตัวเองก่อนจะรู้ด้วย
ซ้ำว่าเราตั้งใจเขียน text):

```
error: expected a valid html element
 --> src/main.rs:7:12
  |
7 |         <p>สวัสดี {name}</p>
  |            ^^^^
```

`html!` แยกความต่างระหว่าง **text literal** กับ **expression** อย่างเข้มงวด — ข้อความ literal ทุกก้อนต้อง
wrap ด้วย `{ }` เสมอ (เช่น `{ "สวัสดี " }`) แล้วค่อยตาม `{name}` เป็น expression อีกก้อน: `<p>{ "สวัสดี " }
{name}</p>` เขียนแบบ mix ข้อความดิบกับ `{}` ปนกันในบรรทัดเดียวแบบภาษาอื่น (เช่น JSX ของ React ที่อนุญาตให้
เขียน `<p>สวัสดี {name}</p>` ได้ตรง ๆ) จะ parse ไม่ผ่านใน `html!` — นี่คือความต่างที่ชัดเจนจาก JSX แม้หน้าตา
จะคล้ายกันมาก: `html!` เป็น macro ที่ต้อง parse เป็น Rust token tree ที่ถูกต้องตามหลักไวยากรณ์ Rust จริง ๆ
ไม่ใช่ template language ที่ยืดหยุ่นแบบ string-based

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** สร้าง component `Greeting` ที่รับ prop `name: String` แล้วแสดงข้อความ `"สวัสดี, {name}!"` พร้อม
   ปุ่มที่กดแล้วเปลี่ยนสีตัวอักษรสลับกันระหว่างสีปกติกับสีแดง (ใช้ `use_state<bool>` เก็บสถานะว่ากำลัง
   highlight อยู่หรือไม่ แล้วใช้ `classes!` หรือ inline style ตาม state นั้น)
   *Hint*: ใช้ `use_state(|| false)` แล้ว toggle ด้วย `state.set(!*state)` ใน `onclick`

2. **(กลาง)** ขยายตัวอย่าง `BookSearch` จากหัวข้อ 88.5 ให้เพิ่ม dropdown สำหรับเลือกเรียงลำดับผลลัพธ์ (เรียง
   ตามชื่อ A-Z หรือ Z-A) — ต้องมี state เพิ่มอีกตัวสำหรับเก็บโหมดการเรียง แล้วใช้ `.sort_by(...)` (ทวนจาก
   Part 25-26) ก่อน `.collect()` ผลลัพธ์ที่กรองแล้ว
   *Hint*: ผูก `onchange` ของ `<select>` เข้ากับ `Callback<Event>` ที่ทำ `event.target_dyn_into::<
   HtmlSelectElement>()` แบบเดียวกับที่ `<input>` ทำ `target_dyn_into::<HtmlInputElement>()`

3. **(ยาก)** สร้างระบบ "ตะกร้าหนังสือที่จะยืม" (borrow cart) แบบเต็มรูปแบบ: `BookList` (parent) ส่ง
   `on_add_to_cart: Callback<Book>` ลงไปให้ `BookCard` (child) ทุกใบ, มี component `CartSummary` แยกที่แสดง
   จำนวนหนังสือในตะกร้าและปุ่ม "ยืมทั้งหมด" — ให้ทั้งสาม component (`BookList`, `BookCard`, `CartSummary`)
   แชร์ข้อมูลตะกร้าผ่าน `use_context` แทนการส่ง prop ตรงทุกชั้น (เพราะ `CartSummary` อยู่นอกต้นไม้ของ
   `BookList` — เป็น sibling กัน ไม่ใช่ parent-child) แล้วทดสอบว่ากด "ยืมทั้งหมด" แล้วตะกร้าว่างและปุ่ม
   "ยืมเล่มนี้" ของหนังสือที่ถูกยืมกลายเป็นปุ่ม disabled จริง
   *Hint*: ใส่ `Callback<Book>` ไว้ใน struct context เดียวกับ `Vec<Book>` (state ของตะกร้า) แล้วให้
   `ContextProvider` ครอบทั้ง `BookList` และ `CartSummary` ไว้ในระดับเดียวกัน (เช่นครอบที่ `App`)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ต่อยอดตัวอย่างในหัวข้อ 88.8-88.9 ให้เป็นแอปเต็มรูปแบบ 2 หน้า: หน้าแรก
   (`/`) fetch รายการหนังสือจาก Axum API จริงแล้วแสดงเป็นลิงก์ (ผสมหัวข้อ 88.8 เข้ากับ 88.9), หน้าที่สอง
   (`/books/:id`) fetch รายละเอียดหนังสือเล่มนั้นเพิ่มจาก endpoint `/api/books/:id` ของ Axum server เดียวกัน
   (ต้องเขียน handler ใหม่ฝั่ง Axum ด้วย ทวนจาก Part 63 เรื่อง path parameter extractor `Path<u32>`) — ให้
   จัดการทั้ง 3 สถานะ (loading/error/success) ในทั้งสองหน้าให้ครบ
   *Hint*: หน้า detail ต้องอ่าน `id` จาก props ที่ `Switch<Route>` ส่งมา แล้วใช้ `id` นั้นเป็นส่วนหนึ่งของ
   deps ใน `use_effect_with` (ไม่ใช่ `()`) เพื่อให้ fetch ใหม่ทุกครั้งที่ผู้ใช้กด link ไปหนังสือเล่มอื่นโดยไม่
   reload หน้า

## สรุป

บทนี้พาคุณจากการจัดการ DOM แบบ imperative ด้วยมือ (Part 87) ไปสู่การเขียน UI แบบ **declarative,
component-based** ด้วย **Yew** — framework ที่ยืม mental model ของ React มาเต็มรูปแบบ (component, props,
state ผ่าน hooks, virtual DOM diffing) แต่ implement ทั้งหมดด้วย Rust ล้วน สร้างอยู่บน `wasm-bindgen`/
`web-sys` ที่ Part 87 สอนไว้เป๊ะ ๆ ไม่มีการข้ามชั้นไปทางลัด คุณได้ลงมือสร้าง component จริง (`BookCard`,
`BookList`, `BookSearch`) ที่รับ props, จัดการ state ด้วย `use_state`/`use_effect_with`, ตอบสนอง event ด้วย
`Callback<T>`, ส่ง callback ลงเป็น prop เพื่อให้ child สื่อสารกลับขึ้นไปยัง parent ได้ (lifting state up),
รู้จัก `use_context` สำหรับกรณีที่ต้นไม้ component ลึกเกินจะส่ง prop ทีละชั้น, เชื่อมต่อกับ REST API ของ
Axum ที่คุณสร้างไว้แล้วในโมดูล 4 ผ่าน `gloo-net`, และสร้างแอปหลายหน้าด้วย `yew-router` — ทุกตัวอย่างถูก build
และรันจริงในเบราว์เซอร์ ไม่ใช่โค้ดที่เขียนลอย ๆ โดยไม่ได้ตรวจสอบ

สิ่งที่ควรจำไว้ก่อนไปต่อ: Yew คือ framework ที่**เก่าแก่และมี ecosystem ใหญ่ที่สุด**ในสาม Rust frontend
framework ที่คอร์สนี้จะสอน แต่ใช้กลไก **virtual DOM diffing** ที่ต่างจาก **fine-grained reactivity** ของ
Leptos และแนวทางของ Dioxus ที่จะเห็นต่อไป — Part 89 จะพาไปดู Leptos ซึ่งแก้ปัญหาเดียวกัน (สร้าง UI แบบ
declarative ด้วย Rust/WASM) ด้วยสถาปัตยกรรมภายในที่ต่างออกไปอย่างมีนัยสำคัญ และยังเป็น full-stack framework
ที่ทำ SSR ได้ในตัวแบบที่ Yew (โหมด CSR ที่บทนี้สอน) ไม่ได้ทำให้ตรง ๆ ตั้งแต่ต้น — เตรียมใจเปรียบเทียบกลไก
ภายในของทั้งสองให้แม่นก่อนจะไปถึง Dioxus ใน Part 90 และการตัดสินสามทางแบบเต็มรูปแบบ

---

**Part ก่อนหน้า:** [wasm-bindgen และ JavaScript Interop](part-087-wasm-bindgen.md) | **Part ถัดไป:** [Leptos Framework: Full-stack Rust](part-089-leptos-framework.md)
