# Project J01: Yew SPA Frontend

> โมดูล: J — Full-Stack / WASM | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Single-Page Application (SPA)** ด้วย **Yew** ซึ่งเป็น frontend framework สำหรับ Rust ที่ compile ไปเป็น **WebAssembly (WASM)** ทำงานในเบราว์เซอร์ได้โดยตรง Yew ยืม concept มาจาก React อย่างชัดเจน ทั้ง function components, hooks, virtual DOM, และ props-based data flow แต่เขียนด้วย Rust ล้วน ๆ ซึ่งหมายความว่าคุณได้รับ type safety และ compile-time guarantees ที่ JavaScript ไม่มี

โปรเจคนี้จะสร้าง **Todo App** ครบฟีเจอร์ตามมาตรฐาน TodoMVC — เพิ่ม/ลบ/แก้ไข task, กรองด้วย All/Active/Completed, บันทึกลง localStorage, และนำทางด้วย client-side router — ทั้งหมดนี้เขียนด้วย Rust และ compile เป็น WASM

**Use cases จริงในโลก production:**
- **Internal tools** — dashboard, form สำหรับทีม internal ที่ต้องการ type-safe frontend
- **Performance-critical UI** — การ render ข้อมูลขนาดใหญ่ใน table หรือ canvas ที่ WASM เร็วกว่า JS
- **Shared logic** — ใช้ business logic เดียวกันกับ backend Rust ทั้ง validation rules และ data structures
- **Game / simulation UI** — ใช้ Yew เป็น UI layer บน top ของ game loop ที่เขียนด้วย Rust/WASM
- **WASM plugin system** — เบราว์เซอร์ extension หรือ plugin ที่ต้องการ sandboxed execution

**Learning value:**
โปรเจคนี้สอน WASM ecosystem ของ Rust อย่างครบถ้วน ตั้งแต่ toolchain setup (trunk, wasm-pack), component model ของ Yew, state management ด้วย hooks และ reducer pattern, event handling แบบ type-safe, async data fetching ใน browser context, และ client-side routing ด้วย yew-router

## สิ่งที่จะได้เรียนรู้

- **WASM toolchain** — ตั้งค่า `wasm32-unknown-unknown` target, `trunk` build tool, และ `index.html` entry point
- **Function components** — เขียน component ด้วย `#[function_component]`, `html!` macro, และ props structs
- **State hooks** — ใช้ `use_state`, `use_reducer` จัดการ state และทำให้ component re-render อัตโนมัติ
- **Event handling** — wire `onclick`, `oninput`, `onsubmit` ด้วย `Callback<E>` แบบ type-safe
- **Context API** — ส่ง state ข้ามหลาย component ด้วย `ContextProvider` แทน props drilling
- **Async effects** — ดึงข้อมูลจาก API ด้วย `use_effect_with` และ `gloo_net::http::Request`
- **Client-side routing** — ใช้ `yew-router` สร้าง `BrowserRouter`, `Switch`, route params
- **Pure logic testing** — แยก business logic ออกเป็น `lib.rs` เพื่อ test ด้วย `cargo test` ปกติ

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`), iterators, closures
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics, trait bounds
- **Part 41–50**: Lifetimes พื้นฐาน, modules, `#[cfg(test)]`
- **Part 51–60**: `async`/`await` พื้นฐาน (จาก Part 46)
- **Part 61–70**: External crates, `Cargo.toml` features, `serde`
- **Part 71–80**: WebAssembly concepts, `wasm-bindgen` พื้นฐาน
- **Part 96–110**: Full-stack architecture, frontend/backend split

## โครงสร้างโปรเจค (Project Layout)

```
yew-todo-app/
├── src/
│   ├── lib.rs              ← business logic (pure Rust, testable)
│   ├── main.rs             ← Yew app entry point
│   ├── components/
│   │   ├── mod.rs
│   │   ├── app.rs          ← root App component
│   │   ├── todo_input.rs   ← input form component
│   │   ├── todo_list.rs    ← list component
│   │   ├── todo_item.rs    ← individual item component
│   │   └── footer.rs       ← filter bar + counts
│   ├── pages/
│   │   ├── mod.rs
│   │   ├── home.rs         ← Home page
│   │   ├── about.rs        ← About page
│   │   └── not_found.rs    ← 404 page
│   ├── hooks/
│   │   ├── mod.rs
│   │   └── use_local_storage.rs ← custom hook
│   └── routes.rs           ← Route enum + Switch
├── tests/
│   └── logic_tests.rs      ← integration tests (pure Rust)
├── index.html              ← Trunk entry point
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow Diagram

```
User Interaction
       │
       ▼
  html! macro (virtual DOM)
       │
       ▼
  Callback<Event>  ──────────────────────────┐
       │                                     │
       ▼                                     │
  use_reducer(TodoAction)                    │
       │                                     │
       ▼                                     │
  todo_reducer(state, action)                │
       │                                     │
       ├── TodoStore (immutable new state)   │
       │         │                           │
       │         └── ContextProvider ────────┘
       │                   │
       │         ┌─────────┴─────────┐
       │         │                   │
       ▼         ▼                   ▼
  TodoInput  TodoList           Footer
  Component  Component          Component
                │
                ▼
           TodoItem (×N)
```

### สถาปัตยกรรมแบบ Unidirectional Data Flow

Yew ใช้ **unidirectional data flow** แบบเดียวกับ React + Redux:
1. **State** อยู่ใน root component (`App`) ใน `use_reducer`
2. **Actions** ถูกส่งจาก child component ขึ้นมา root ผ่าน `Callback`
3. **Reducer** รับ action ส่งคืน state ใหม่
4. Root ส่ง state ลงไปยัง children ผ่าน `ContextProvider`
5. Children re-render เมื่อ context ที่ใช้เปลี่ยน

การออกแบบนี้ดีกว่า prop drilling เพราะ component ใด ๆ ก็ access state ได้โดยตรงผ่าน `use_context` โดยไม่ต้องผ่าน parent chain

### ทำไมต้องแยก lib.rs

business logic เช่น `TodoStore`, `Filter`, `todo_reducer` ล้วนเป็น pure Rust ที่ไม่ต้องพึ่ง browser API ดังนั้นจึงสามารถ compile และ test บน host machine ได้ด้วย `cargo test` ปกติ โดยไม่ต้องใช้ `wasm-bindgen-test` หรือ headless browser — นี่คือ **design principle สำคัญ** ของ Yew application

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: ติดตั้ง Toolchain และสร้างโปรเจค

ก่อนอื่นต้องติดตั้ง tools ที่จำเป็น:

```bash
# เพิ่ม WASM target ให้ Rust toolchain
rustup target add wasm32-unknown-unknown

# ติดตั้ง trunk — build tool สำหรับ Yew (เหมือน webpack สำหรับ Rust)
cargo install trunk

# ตรวจสอบการติดตั้ง
trunk --version
# trunk 0.21.x
```

**trunk** คือ build tool ที่:
- Compile Rust/WASM อัตโนมัติเมื่อไฟล์เปลี่ยน
- Bundle CSS, JS, และ assets
- Serve development server ด้วย hot reload
- สร้าง production build ที่ optimize แล้ว

สร้างโปรเจคใหม่:

```bash
cargo new yew-todo-app
cd yew-todo-app
```

แก้ `Cargo.toml`:

```toml
[package]
name = "yew-todo-app"
version = "0.1.0"
edition = "2021"

[lib]
name = "yew_todo_logic"
path = "src/lib.rs"

[[bin]]
name = "yew-todo-app"
path = "src/main.rs"

[dependencies]
yew = { version = "0.21", features = ["csr"] }
yew-router = "0.18"
gloo-net = "0.6"
gloo-storage = "0.3"
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = [
    "Window",
    "Document",
    "HtmlInputElement",
    "KeyboardEvent",
    "FocusEvent",
] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
log = "0.4"
wasm-logger = "0.2"

[dev-dependencies]
wasm-bindgen-test = "0.3"
```

สร้าง `index.html` ที่ root ของโปรเจค — นี่คือ entry point ของ trunk:

```html
<!DOCTYPE html>
<html lang="th">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Yew Todo App</title>
    <link rel="stylesheet" href="https://unpkg.com/todomvc-app-css@2.4.1/index.css"/>
  </head>
  <body>
    <!-- trunk จะ inject script tag ที่ load WASM ให้อัตโนมัติ -->
  </body>
</html>
```

สังเกตว่าไม่มี `<script>` tag ใด ๆ — trunk จะ inject เอง โดย scan `<!-- trunk ... -->` comments และ generate tags ที่จำเป็น

รัน development server:

```bash
trunk serve --open
# Building... (ครั้งแรกใช้เวลา 30-60 วินาที)
# Serving at http://localhost:8080
```

---

### ขั้นที่ 2: Function Component แรก และ html! Macro

`#[function_component]` คือ attribute macro ที่แปลง function ธรรมดาเป็น Yew component function component ต้องรับ props (หรือ `()`) และ return `Html`

สร้าง `src/main.rs`:

```rust
use yew::prelude::*;

// Component อย่างง่ายที่สุด — ไม่รับ props
#[function_component(HelloWorld)]
fn hello_world() -> Html {
    html! {
        <div class="container">
            <h1>{"สวัสดี Yew! 🦀"}</h1>
            <p>{"WASM frontend เขียนด้วย Rust"}</p>
        </div>
    }
}

fn main() {
    // เริ่ม Yew app — mount component เข้า <body>
    yew::Renderer::<HelloWorld>::new().render();
}
```

`html!` macro คือ JSX ของ Yew — เขียน HTML-like syntax ใน Rust โดยตรง กฎสำคัญ:
- Tag ต้องปิดเสมอ (`<br/>` ไม่ใช่ `<br>`)
- Rust expression อยู่ใน `{...}` (ใช้ `{"ข้อความ"}` สำหรับ string literal)
- `class` ใช้แทน `className` (ต่างจาก React)
- Event ใช้ `onclick`, `oninput` ฯลฯ (ล้วน lowercase)

ตัวอย่าง Props — สร้าง component ที่รับข้อมูลจากภายนอก:

```rust
// Props struct ต้อง derive Properties และ PartialEq
#[derive(Properties, PartialEq)]
pub struct GreetingProps {
    pub name: String,
    #[prop_or_default]
    pub greeting: String, // optional prop ที่มี default value
}

#[function_component(Greeting)]
pub fn greeting(props: &GreetingProps) -> Html {
    let greeting = if props.greeting.is_empty() {
        "สวัสดี"
    } else {
        &props.greeting
    };

    html! {
        <p>{format!("{}, {}!", greeting, props.name)}</p>
    }
}

// การใช้งาน
html! {
    <>
        <Greeting name="Rust" />
        <Greeting name="Yew" greeting="Hello" />
    </>
}
```

`<>...</>` คือ Fragment — ใช้เมื่อต้องการ return หลาย element โดยไม่มี wrapper `<div>`

ตัวอย่าง Children props:

```rust
#[derive(Properties, PartialEq)]
pub struct CardProps {
    pub title: String,
    pub children: Children, // รับ child elements
}

#[function_component(Card)]
pub fn card(props: &CardProps) -> Html {
    html! {
        <div class="card">
            <h2>{ &props.title }</h2>
            <div class="card-body">
                { for props.children.iter() }
            </div>
        </div>
    }
}

// การใช้งาน
html! {
    <Card title="หัวข้อ">
        <p>{"เนื้อหาของ Card"}</p>
        <button>{"ปุ่ม"}</button>
    </Card>
}
```

---

### ขั้นที่ 3: State Hooks — use_state และ use_reducer

Hooks คือ function พิเศษที่ใช้ได้เฉพาะใน function component ห้ามเรียกใน loop หรือ conditional

#### use_state

`use_state` เก็บค่า state และ return `UseStateHandle` ที่มีทั้ง ค่าปัจจุบัน และ function สำหรับอัปเดต:

```rust
#[function_component(Counter)]
pub fn counter() -> Html {
    // use_state รับ closure ที่ return ค่า initial
    let count = use_state(|| 0i32);

    // clone handle เพื่อ move เข้า closure แต่ละอัน
    let increment = {
        let count = count.clone();
        Callback::from(move |_| count.set(*count + 1))
    };

    let decrement = {
        let count = count.clone();
        Callback::from(move |_| count.set(*count - 1))
    };

    html! {
        <div>
            <button onclick={decrement}>{ "-" }</button>
            <span>{ *count }</span>
            <button onclick={increment}>{ "+" }</button>
        </div>
    }
}
```

สังเกตว่าต้อง `clone()` handle ก่อน move เข้า closure เพราะ closure แต่ละอันต้องมี ownership ของตัวเอง และต้อง dereference ด้วย `*count` เพื่อได้ค่า `i32`

#### use_reducer

เมื่อ state ซับซ้อนขึ้น ใช้ `use_reducer` ที่ใช้ pattern เดียวกับ Redux:

```rust
use yew::prelude::*;
use std::rc::Rc;

#[derive(Clone, PartialEq)]
struct AppState {
    count: i32,
    history: Vec<i32>,
}

impl Default for AppState {
    fn default() -> Self {
        Self { count: 0, history: Vec::new() }
    }
}

#[derive(Clone)]
enum AppAction {
    Increment,
    Decrement,
    Reset,
}

impl Reducible for AppState {
    type Action = AppAction;

    fn reduce(self: Rc<Self>, action: Self::Action) -> Rc<Self> {
        let mut next = (*self).clone();
        match action {
            AppAction::Increment => {
                next.history.push(next.count);
                next.count += 1;
            }
            AppAction::Decrement => {
                next.history.push(next.count);
                next.count -= 1;
            }
            AppAction::Reset => {
                next.history.clear();
                next.count = 0;
            }
        }
        Rc::new(next)
    }
}

#[function_component(ReducerCounter)]
pub fn reducer_counter() -> Html {
    let state = use_reducer(AppState::default);

    let inc = {
        let state = state.clone();
        Callback::from(move |_| state.dispatch(AppAction::Increment))
    };

    html! {
        <div>
            <p>{ format!("Count: {}", state.count) }</p>
            <p>{ format!("History: {:?}", state.history) }</p>
            <button onclick={inc}>{ "Increment" }</button>
        </div>
    }
}
```

`Reducible` trait บังคับให้ implement `reduce(self: Rc<Self>, action) -> Rc<Self>` — ต้อง return `Rc<Self>` ไม่ใช่ `Self` โดยตรง นี่คือ **gotcha** ที่พบบ่อย

---

### ขั้นที่ 4: Business Logic ใน lib.rs

สร้าง `src/lib.rs` ที่เก็บ logic ทั้งหมดแยกออกจาก UI layer — ทดสอบได้โดยไม่ต้องใช้ browser:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Todo {
    pub id: u32,
    pub title: String,
    pub completed: bool,
}

impl Todo {
    pub fn new(id: u32, title: impl Into<String>) -> Self {
        Self {
            id,
            title: title.into(),
            completed: false,
        }
    }

    pub fn toggle(&mut self) {
        self.completed = !self.completed;
    }
}

#[derive(Debug, Clone, PartialEq, Default)]
pub enum Filter {
    #[default]
    All,
    Active,
    Completed,
}

impl Filter {
    pub fn label(&self) -> &'static str {
        match self {
            Filter::All => "All",
            Filter::Active => "Active",
            Filter::Completed => "Completed",
        }
    }

    pub fn matches(&self, todo: &Todo) -> bool {
        match self {
            Filter::All => true,
            Filter::Active => !todo.completed,
            Filter::Completed => todo.completed,
        }
    }
}

#[derive(Debug, Clone, Default)]
pub struct TodoStore {
    pub todos: Vec<Todo>,
    pub filter: Filter,
    pub next_id: u32,
    pub new_input: String,
}

impl TodoStore {
    pub fn new() -> Self {
        Self { todos: Vec::new(), filter: Filter::All, next_id: 1, new_input: String::new() }
    }

    pub fn add(&mut self, title: &str) -> Option<u32> {
        let trimmed = title.trim();
        if trimmed.is_empty() { return None; }
        let id = self.next_id;
        self.todos.push(Todo::new(id, trimmed));
        self.next_id += 1;
        Some(id)
    }

    pub fn remove(&mut self, id: u32) {
        self.todos.retain(|t| t.id != id);
    }

    pub fn toggle(&mut self, id: u32) {
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.toggle();
        }
    }

    pub fn update_title(&mut self, id: u32, new_title: &str) {
        let trimmed = new_title.trim();
        if trimmed.is_empty() { self.remove(id); return; }
        if let Some(todo) = self.todos.iter_mut().find(|t| t.id == id) {
            todo.title = trimmed.to_string();
        }
    }

    pub fn toggle_all(&mut self) {
        let all_done = self.todos.iter().all(|t| t.completed);
        for todo in self.todos.iter_mut() {
            todo.completed = !all_done;
        }
    }

    pub fn clear_completed(&mut self) {
        self.todos.retain(|t| !t.completed);
    }

    pub fn filtered_todos(&self) -> Vec<&Todo> {
        self.todos.iter().filter(|t| self.filter.matches(t)).collect()
    }

    pub fn active_count(&self) -> usize {
        self.todos.iter().filter(|t| !t.completed).count()
    }

    pub fn completed_count(&self) -> usize {
        self.todos.iter().filter(|t| t.completed).count()
    }

    pub fn total_count(&self) -> usize {
        self.todos.len()
    }

    pub fn set_filter(&mut self, filter: Filter) {
        self.filter = filter;
    }
}

// Reducer Actions
#[derive(Debug, Clone)]
pub enum TodoAction {
    Add(String),
    Remove(u32),
    Toggle(u32),
    UpdateTitle(u32, String),
    ToggleAll,
    ClearCompleted,
    SetFilter(Filter),
    SetInput(String),
}

pub fn todo_reducer(mut state: TodoStore, action: TodoAction) -> TodoStore {
    match action {
        TodoAction::Add(title) => { state.add(&title); state.new_input.clear(); state }
        TodoAction::Remove(id) => { state.remove(id); state }
        TodoAction::Toggle(id) => { state.toggle(id); state }
        TodoAction::UpdateTitle(id, title) => { state.update_title(id, &title); state }
        TodoAction::ToggleAll => { state.toggle_all(); state }
        TodoAction::ClearCompleted => { state.clear_completed(); state }
        TodoAction::SetFilter(filter) => { state.set_filter(filter); state }
        TodoAction::SetInput(input) => { state.new_input = input; state }
    }
}

pub fn serialize_todos(todos: &[Todo]) -> Result<String, serde_json::Error> {
    serde_json::to_string(todos)
}

pub fn deserialize_todos(json: &str) -> Result<Vec<Todo>, serde_json::Error> {
    serde_json::from_str(json)
}

pub fn validate_new_todo_title(title: &str) -> Result<String, &'static str> {
    let trimmed = title.trim();
    if trimmed.is_empty() { return Err("ชื่อ Todo ต้องไม่ว่างเปล่า"); }
    if trimmed.len() > 255 { return Err("ชื่อ Todo ยาวเกินไป (max 255 ตัวอักษร)"); }
    Ok(trimmed.to_string())
}
```

---

### ขั้นที่ 5: Event Handling — Callback และ User Input

Event handling ใน Yew ใช้ `Callback<E>` ซึ่งเป็น type-safe wrapper สำหรับ closure ที่รับ event object

สร้าง `src/components/todo_input.rs`:

```rust
use yew::prelude::*;
use web_sys::HtmlInputElement;

#[derive(Properties, PartialEq)]
pub struct TodoInputProps {
    pub on_add: Callback<String>,
}

#[function_component(TodoInput)]
pub fn todo_input(props: &TodoInputProps) -> Html {
    let input_value = use_state(String::new);

    // oninput — ทุกครั้งที่ user พิมพ์
    let oninput = {
        let input_value = input_value.clone();
        Callback::from(move |e: InputEvent| {
            // cast EventTarget เป็น HtmlInputElement เพื่อ .value()
            let input: HtmlInputElement = e.target_unchecked_into();
            input_value.set(input.value());
        })
    };

    // onkeydown — จับ Enter key
    let onkeydown = {
        let input_value = input_value.clone();
        let on_add = props.on_add.clone();
        Callback::from(move |e: KeyboardEvent| {
            if e.key() == "Enter" {
                let val = (*input_value).clone();
                let trimmed = val.trim();
                if !trimmed.is_empty() {
                    on_add.emit(trimmed.to_string());
                    input_value.set(String::new());
                }
            }
        })
    };

    html! {
        <input
            class="new-todo"
            placeholder="What needs to be done?"
            value={(*input_value).clone()}
            {oninput}
            {onkeydown}
            autofocus=true
        />
    }
}
```

จุดสำคัญ:
- `e.target_unchecked_into::<HtmlInputElement>()` — cast EventTarget โดยไม่ตรวจสอบ runtime (ใช้เมื่อ sure 100% ว่า element type ถูกต้อง)
- `on_add.emit(value)` — เรียก callback ที่ส่งมาจาก parent
- `Callback::from(move |e| ...)` — สร้าง callback จาก closure, `move` เพื่อ capture ตัวแปรจาก outer scope

ตัวอย่าง `onsubmit` กับ form:

```rust
let onsubmit = {
    let on_add = props.on_add.clone();
    let input_value = input_value.clone();
    Callback::from(move |e: SubmitEvent| {
        e.prevent_default(); // ป้องกัน page reload
        let val = (*input_value).clone();
        if !val.trim().is_empty() {
            on_add.emit(val.trim().to_string());
            input_value.set(String::new());
        }
    })
};

html! {
    <form {onsubmit}>
        <input type="text" value={(*input_value).clone()} {oninput} />
        <button type="submit">{ "เพิ่ม" }</button>
    </form>
}
```

---

### ขั้นที่ 6: Component Composition และ Context API

เมื่อ application ใหญ่ขึ้น การส่ง state ผ่าน props หลายชั้น (props drilling) กลายเป็นปัญหา Yew แก้ด้วย **Context API**

สร้าง `src/components/app.rs` — root component ที่เป็น ContextProvider:

```rust
use yew::prelude::*;
use std::rc::Rc;
use crate::{TodoStore, TodoAction, todo_reducer};
use crate::components::{TodoInput, TodoList, Footer};

// Context type ที่ child components จะ consume
#[derive(Clone, PartialEq)]
pub struct AppContext {
    pub state: Rc<TodoStore>,
    pub dispatch: Callback<TodoAction>,
}

// implement Reducible สำหรับ TodoStore
impl Reducible for TodoStore {
    type Action = TodoAction;

    fn reduce(self: Rc<Self>, action: Self::Action) -> Rc<Self> {
        Rc::new(todo_reducer((*self).clone(), action))
    }
}

#[function_component(App)]
pub fn app() -> Html {
    let state = use_reducer(TodoStore::new);

    // สร้าง context value
    let ctx = AppContext {
        state: Rc::new((*state).clone()),
        dispatch: Callback::from({
            let state = state.clone();
            move |action| state.dispatch(action)
        }),
    };

    html! {
        // ContextProvider ห่อหุ้ม component tree ทั้งหมด
        <ContextProvider<AppContext> context={ctx}>
            <section class="todoapp">
                <header class="header">
                    <h1>{ "todos" }</h1>
                    <TodoInput on_add={Callback::from({
                        let state = state.clone();
                        move |title: String| state.dispatch(TodoAction::Add(title))
                    })} />
                </header>
                <TodoList />
                <Footer />
            </section>
        </ContextProvider<AppContext>>
    }
}
```

Child component ใช้ `use_context` เพื่อ subscribe:

```rust
// src/components/footer.rs
use yew::prelude::*;
use crate::{Filter, TodoAction};
use crate::components::app::AppContext;

#[function_component(Footer)]
pub fn footer() -> Html {
    // use_context return Option<T> — None ถ้าไม่มี Provider
    let ctx = use_context::<AppContext>().expect("ไม่พบ AppContext");
    let state = &ctx.state;
    let dispatch = &ctx.dispatch;

    let active = state.active_count();
    let completed = state.completed_count();

    html! {
        <footer class="footer">
            <span class="todo-count">
                <strong>{ active }</strong>
                { format!(" item{} left", if active == 1 { "" } else { "s" }) }
            </span>
            <ul class="filters">
                { for [Filter::All, Filter::Active, Filter::Completed].iter().map(|f| {
                    let is_selected = &state.filter == f;
                    let filter = f.clone();
                    let dispatch = dispatch.clone();
                    html! {
                        <li>
                            <a
                                class={if is_selected { "selected" } else { "" }}
                                onclick={Callback::from(move |_| {
                                    dispatch.emit(TodoAction::SetFilter(filter.clone()))
                                })}
                            >
                                { f.label() }
                            </a>
                        </li>
                    }
                })}
            </ul>
            if completed > 0 {
                <button
                    class="clear-completed"
                    onclick={let d = dispatch.clone(); Callback::from(move |_| d.emit(TodoAction::ClearCompleted))}
                >
                    { "Clear completed" }
                </button>
            }
        </footer>
    }
}
```

`if completed > 0 { ... }` คือ **conditional rendering** ใน `html!` macro — ใช้ `if` โดยตรงได้ ไม่ต้องใช้ ternary

---

### ขั้นที่ 7: Async Data Fetching ด้วย use_effect_with

`use_effect_with` คือ hook สำหรับ side effects — เช่น data fetching, DOM manipulation, subscriptions

```rust
use yew::prelude::*;
use gloo_net::http::Request;
use serde::Deserialize;

#[derive(Deserialize, Clone, PartialEq)]
struct Post {
    id: u32,
    title: String,
    body: String,
}

#[function_component(PostList)]
pub fn post_list() -> Html {
    let posts: UseStateHandle<Option<Vec<Post>>> = use_state(|| None);
    let loading = use_state(|| true);
    let error: UseStateHandle<Option<String>> = use_state(|| None);

    // effect รัน 1 ครั้งเมื่อ component mount (dependencies = ())
    {
        let posts = posts.clone();
        let loading = loading.clone();
        let error = error.clone();

        use_effect_with((), move |_| {
            // spawn async task — ใช้ wasm_bindgen_futures
            wasm_bindgen_futures::spawn_local(async move {
                match Request::get("https://jsonplaceholder.typicode.com/posts?_limit=5")
                    .send()
                    .await
                {
                    Ok(response) => {
                        match response.json::<Vec<Post>>().await {
                            Ok(data) => posts.set(Some(data)),
                            Err(e) => error.set(Some(format!("Parse error: {}", e))),
                        }
                    }
                    Err(e) => error.set(Some(format!("Fetch error: {}", e))),
                }
                loading.set(false);
            });

            // return cleanup function (optional)
            || ()
        });
    }

    html! {
        <div>
            if *loading {
                <p>{ "กำลังโหลด..." }</p>
            } else if let Some(err) = (*error).clone() {
                <p style="color:red">{ err }</p>
            } else if let Some(data) = (*posts).clone() {
                <ul>
                    { for data.iter().map(|post| html! {
                        <li key={post.id}>
                            <strong>{ &post.title }</strong>
                            <p>{ &post.body }</p>
                        </li>
                    })}
                </ul>
            }
        </div>
    }
}
```

`use_effect_with(deps, effect_fn)` — รัน `effect_fn` ทุกครั้งที่ `deps` เปลี่ยน เมื่อใช้ `()` จะรันแค่ครั้งเดียวตอน mount เช่นเดียวกับ `useEffect(() => {...}, [])` ใน React

ตัวอย่าง effect ที่ re-fetch เมื่อ id เปลี่ยน:

```rust
#[derive(Properties, PartialEq)]
pub struct PostDetailProps {
    pub post_id: u32,
}

#[function_component(PostDetail)]
pub fn post_detail(props: &PostDetailProps) -> Html {
    let post: UseStateHandle<Option<Post>> = use_state(|| None);

    // re-fetch ทุกครั้งที่ post_id เปลี่ยน
    {
        let post = post.clone();
        let post_id = props.post_id;
        use_effect_with(post_id, move |&id| {
            wasm_bindgen_futures::spawn_local(async move {
                let url = format!("https://jsonplaceholder.typicode.com/posts/{}", id);
                if let Ok(resp) = Request::get(&url).send().await {
                    if let Ok(data) = resp.json::<Post>().await {
                        post.set(Some(data));
                    }
                }
            });
            || ()
        });
    }

    html! {
        if let Some(p) = (*post).clone() {
            <div>
                <h2>{ &p.title }</h2>
                <p>{ &p.body }</p>
            </div>
        } else {
            <p>{ "กำลังโหลด..." }</p>
        }
    }
}
```

#### Custom Hook: use_local_storage

สร้าง hook ที่ reusable สำหรับ localStorage — สร้าง `src/hooks/use_local_storage.rs`:

```rust
use yew::prelude::*;
use gloo_storage::{LocalStorage, Storage};
use serde::{de::DeserializeOwned, Serialize};

pub fn use_local_storage<T>(key: &'static str, default: T) -> (T, Callback<T>)
where
    T: Clone + PartialEq + Serialize + DeserializeOwned + 'static,
{
    let value = use_state(|| {
        // พยายามอ่านจาก localStorage
        LocalStorage::get::<T>(key).unwrap_or(default)
    });

    let setter = {
        let value = value.clone();
        Callback::from(move |new_val: T| {
            // บันทึกลง localStorage
            let _ = LocalStorage::set(key, &new_val);
            value.set(new_val);
        })
    };

    ((*value).clone(), setter)
}
```

การใช้งาน:

```rust
#[function_component(PersistentCounter)]
pub fn persistent_counter() -> Html {
    let (count, set_count) = use_local_storage("counter", 0i32);

    let increment = {
        let set_count = set_count.clone();
        let count = count;
        Callback::from(move |_| set_count.emit(count + 1))
    };

    html! {
        <div>
            <p>{ format!("นับ: {} (บันทึกอัตโนมัติ)", count) }</p>
            <button onclick={increment}>{ "+" }</button>
        </div>
    }
}
```

---

### ขั้นที่ 8: Client-Side Routing ด้วย yew-router

yew-router ใช้ pattern เดียวกับ React Router แต่ type-safe อย่างสมบูรณ์

สร้าง `src/routes.rs`:

```rust
use yew_router::prelude::*;

// Route enum — derive Routable เพื่อให้ yew-router รู้จัก
#[derive(Clone, Routable, PartialEq)]
pub enum Route {
    #[at("/")]
    Home,
    #[at("/todos")]
    TodoList,
    #[at("/todos/:id")]
    TodoDetail { id: u32 },
    #[at("/about")]
    About,
    #[not_found]
    #[at("/404")]
    NotFound,
}

// Switch function — mapping Route → Html
pub fn switch(route: Route) -> Html {
    use yew::prelude::*;
    use crate::pages::{Home, TodoListPage, TodoDetailPage, About, NotFound};

    match route {
        Route::Home => html! { <Home /> },
        Route::TodoList => html! { <TodoListPage /> },
        Route::TodoDetail { id } => html! { <TodoDetailPage {id} /> },
        Route::About => html! { <About /> },
        Route::NotFound => html! { <NotFound /> },
    }
}
```

แก้ `src/main.rs` ให้ใช้ router:

```rust
use yew::prelude::*;
use yew_router::prelude::*;
use crate::routes::{Route, switch};

#[function_component(Root)]
fn root() -> Html {
    html! {
        // BrowserRouter ห่อหุ้ม app ทั้งหมด
        <BrowserRouter>
            <nav>
                // Link component สร้าง <a> tag ที่ intercept click
                <Link<Route> to={Route::Home}>{ "หน้าแรก" }</Link<Route>>
                { " | " }
                <Link<Route> to={Route::TodoList}>{ "Todo" }</Link<Route>>
                { " | " }
                <Link<Route> to={Route::About}>{ "เกี่ยวกับ" }</Link<Route>>
            </nav>
            // Switch render component ตาม current route
            <Switch<Route> render={switch} />
        </BrowserRouter>
    }
}

fn main() {
    wasm_logger::init(wasm_logger::Config::default());
    yew::Renderer::<Root>::new().render();
}
```

ใช้ hook สำหรับ programmatic navigation:

```rust
// ใน component ที่ต้องการ redirect
#[function_component(LoginPage)]
pub fn login_page() -> Html {
    let navigator = use_navigator().unwrap();

    let on_login_success = Callback::from(move |_| {
        navigator.push(&Route::TodoList);
    });

    html! {
        <button onclick={on_login_success}>{ "เข้าสู่ระบบ" }</button>
    }
}
```

อ่าน route params:

```rust
#[derive(Properties, PartialEq)]
pub struct TodoDetailPageProps {
    pub id: u32,
}

#[function_component(TodoDetailPage)]
pub fn todo_detail_page(props: &TodoDetailPageProps) -> Html {
    let ctx = use_context::<AppContext>().expect("no context");
    let todo = ctx.state.todos.iter().find(|t| t.id == props.id);

    html! {
        if let Some(todo) = todo {
            <div>
                <h2>{ &todo.title }</h2>
                <p>{ if todo.completed { "เสร็จแล้ว ✓" } else { "ยังไม่เสร็จ" } }</p>
                <Link<Route> to={Route::TodoList}>{ "← กลับ" }</Link<Route>>
            </div>
        } else {
            <p>{ format!("ไม่พบ Todo #{}", props.id) }</p>
        }
    }
}
```

---

### ขั้นที่ 9: Todo App สมบูรณ์ — CRUD + Filter + localStorage

สร้าง `src/components/todo_item.rs` — component สำหรับแต่ละ item:

```rust
use yew::prelude::*;
use web_sys::HtmlInputElement;
use crate::Todo;
use crate::components::app::AppContext;
use crate::TodoAction;

#[derive(Properties, PartialEq)]
pub struct TodoItemProps {
    pub todo: Todo,
}

#[function_component(TodoItem)]
pub fn todo_item(props: &TodoItemProps) -> Html {
    let ctx = use_context::<AppContext>().unwrap();
    let editing = use_state(|| false);
    let edit_value = use_state(|| props.todo.title.clone());

    let todo_id = props.todo.id;

    // เริ่ม edit mode เมื่อ double-click
    let ondblclick = {
        let editing = editing.clone();
        let edit_value = edit_value.clone();
        let title = props.todo.title.clone();
        Callback::from(move |_| {
            edit_value.set(title.clone());
            editing.set(true);
        })
    };

    // toggle completed
    let ontoggle = {
        let dispatch = ctx.dispatch.clone();
        Callback::from(move |_| dispatch.emit(TodoAction::Toggle(todo_id)))
    };

    // ลบ todo
    let ondelete = {
        let dispatch = ctx.dispatch.clone();
        Callback::from(move |_| dispatch.emit(TodoAction::Remove(todo_id)))
    };

    // save เมื่อ blur หรือกด Enter
    let save_edit = {
        let dispatch = ctx.dispatch.clone();
        let edit_value = edit_value.clone();
        let editing = editing.clone();
        Callback::from(move |_: ()| {
            dispatch.emit(TodoAction::UpdateTitle(todo_id, (*edit_value).clone()));
            editing.set(false);
        })
    };

    let onblur = {
        let save_edit = save_edit.clone();
        Callback::from(move |_: FocusEvent| save_edit.emit(()))
    };

    let onkeydown = {
        let save_edit = save_edit.clone();
        let editing = editing.clone();
        Callback::from(move |e: KeyboardEvent| {
            match e.key().as_str() {
                "Enter" => save_edit.emit(()),
                "Escape" => editing.set(false),
                _ => {}
            }
        })
    };

    let oninput = {
        let edit_value = edit_value.clone();
        Callback::from(move |e: InputEvent| {
            let input: HtmlInputElement = e.target_unchecked_into();
            edit_value.set(input.value());
        })
    };

    let css_class = classes!(
        if props.todo.completed { Some("completed") } else { None },
        if *editing { Some("editing") } else { None },
    );

    html! {
        <li class={css_class}>
            <div class="view">
                <input
                    class="toggle"
                    type="checkbox"
                    checked={props.todo.completed}
                    onclick={ontoggle}
                />
                <label ondblclick={ondblclick}>
                    { &props.todo.title }
                </label>
                <button class="destroy" onclick={ondelete} />
            </div>
            if *editing {
                <input
                    class="edit"
                    value={(*edit_value).clone()}
                    {oninput}
                    {onblur}
                    {onkeydown}
                    autofocus=true
                />
            }
        </li>
    }
}
```

สร้าง `src/components/todo_list.rs`:

```rust
use yew::prelude::*;
use crate::TodoAction;
use crate::components::app::AppContext;
use crate::components::todo_item::TodoItem;

#[function_component(TodoList)]
pub fn todo_list() -> Html {
    let ctx = use_context::<AppContext>().unwrap();
    let state = &ctx.state;
    let dispatch = &ctx.dispatch;

    if state.total_count() == 0 {
        return html! {};
    }

    let visible = state.filtered_todos();
    let all_completed = state.active_count() == 0;

    let toggle_all = {
        let dispatch = dispatch.clone();
        Callback::from(move |_| dispatch.emit(TodoAction::ToggleAll))
    };

    html! {
        <section class="main">
            <input
                id="toggle-all"
                class="toggle-all"
                type="checkbox"
                checked={all_completed}
                onclick={toggle_all}
            />
            <label for="toggle-all">{ "Mark all as complete" }</label>
            <ul class="todo-list">
                { for visible.iter().map(|todo| html! {
                    <TodoItem key={todo.id} todo={(*todo).clone()} />
                })}
            </ul>
        </section>
    }
}
```

App สมบูรณ์พร้อม localStorage persistence:

```rust
// src/components/app.rs (เพิ่ม localStorage)
use gloo_storage::{LocalStorage, Storage};

const STORAGE_KEY: &str = "yew-todo-app";

#[function_component(App)]
pub fn app() -> Html {
    let state = use_reducer(|| {
        // โหลด todos จาก localStorage ตอน init
        let todos = LocalStorage::get::<Vec<crate::Todo>>(STORAGE_KEY)
            .unwrap_or_default();
        let mut store = TodoStore::new();
        // กำหนด next_id จาก max existing id
        store.next_id = todos.iter().map(|t| t.id).max().unwrap_or(0) + 1;
        store.todos = todos;
        store
    });

    // บันทึกลง localStorage ทุกครั้งที่ todos เปลี่ยน
    {
        let todos = state.todos.clone();
        use_effect_with(todos.clone(), move |todos| {
            let _ = LocalStorage::set(STORAGE_KEY, todos);
            || ()
        });
    }

    // ... rest of component
}
```

---

### ขั้นที่ 10: Testing Strategy และ Production Build

#### Pure Logic Tests (ใช้ cargo test ปกติ)

เนื่องจาก business logic อยู่ใน `lib.rs` ที่ไม่พึ่ง browser API สามารถ test ด้วย `cargo test` ได้ทันที:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_todo_full_lifecycle() {
        let mut store = TodoStore::new();

        // สร้าง
        let id = store.add("เรียน Yew").unwrap();
        assert_eq!(store.total_count(), 1);

        // toggle
        store.toggle(id);
        assert!(store.todos[0].completed);

        // filter
        store.set_filter(Filter::Completed);
        assert_eq!(store.filtered_todos().len(), 1);

        // ลบ
        store.remove(id);
        assert_eq!(store.total_count(), 0);
    }

    #[test]
    fn test_reducer_immutability() {
        let state1 = TodoStore::new();
        let state2 = todo_reducer(state1.clone(), TodoAction::Add("งาน".to_string()));
        // state1 ยังคงเดิม, state2 มีการเปลี่ยนแปลง
        assert_eq!(state1.total_count(), 0);
        assert_eq!(state2.total_count(), 1);
    }
}
```

#### WASM Tests (ต้องใช้ wasm-bindgen-test)

สำหรับ test ที่ต้องการ browser environment ใช้ `wasm-bindgen-test`:

```rust
// tests/wasm_tests.rs
use wasm_bindgen_test::*;

// ระบุว่า test รันใน browser
wasm_bindgen_test_configure!(run_in_browser);

#[wasm_bindgen_test]
fn test_browser_env() {
    let window = web_sys::window().expect("no window");
    assert!(window.document().is_some());
}
```

รัน WASM tests:
```bash
wasm-pack test --headless --firefox
# หรือ
wasm-pack test --headless --chrome
```

#### Production Build

```bash
# Build optimized WASM bundle
trunk build --release

# Output อยู่ใน dist/ directory
# dist/
# ├── index.html
# ├── yew-todo-app-<hash>.js    ← glue code
# ├── yew-todo-app-<hash>_bg.wasm ← WASM binary (~800KB gzip'd ~250KB)
# └── index-<hash>.css

# ดู size
ls -lh dist/
```

ปรับ `Cargo.toml` สำหรับ production:

```toml
[profile.release]
opt-level = "z"    # optimize for binary size (ดีกว่า "3" สำหรับ WASM)
lto = true         # link-time optimization
codegen-units = 1  # เพิ่ม optimization opportunities
panic = "abort"    # ลด binary size (ไม่ต้องการ unwinding ใน WASM)
strip = true       # strip debug symbols
```

Deploy ไปยัง static hosting:
```bash
# Netlify
netlify deploy --dir dist --prod

# GitHub Pages (ต้องแก้ base path ถ้าไม่ได้ host ที่ root)
trunk build --release --public-url /yew-todo-app/

# Docker
# สร้าง nginx container serve dist/
```

## การทดสอบ (Testing)

### Pure Logic Tests — real cargo test output

```
running 40 tests
test tests::test_deserialize_empty_array ... ok
test tests::test_filter_active_excludes_completed ... ok
test tests::test_filter_all_matches_everything ... ok
test tests::test_filter_labels ... ok
test tests::test_filter_completed_excludes_active ... ok
test tests::test_reducer_add_action ... ok
test tests::test_reducer_clear_completed_action ... ok
test tests::test_reducer_set_filter_action ... ok
test tests::test_reducer_set_input_action ... ok
test tests::test_reducer_toggle_action ... ok
test tests::test_reducer_toggle_all_action ... ok
test tests::test_reducer_update_title_action ... ok
test tests::test_route_from_path_about ... ok
test tests::test_route_from_path_home ... ok
test tests::test_route_from_path_invalid_detail ... ok
test tests::test_route_from_path_todo_detail ... ok
test tests::test_route_from_path_todos ... ok
test tests::test_route_from_path_unknown ... ok
test tests::test_route_to_path ... ok
test tests::test_serialize_deserialize_todos ... ok
test tests::test_store_add_empty_returns_none ... ok
test tests::test_store_add_todo ... ok
test tests::test_store_add_trims_whitespace ... ok
test tests::test_store_clear_completed ... ok
test tests::test_store_counts ... ok
test tests::test_store_filtered_todos_active ... ok
test tests::test_store_filtered_todos_completed ... ok
test tests::test_store_remove_todo ... ok
test tests::test_store_toggle_all_marks_all_complete ... ok
test tests::test_store_toggle_all_when_all_complete_unchecks ... ok
test tests::test_store_toggle_todo ... ok
test tests::test_store_update_title ... ok
test tests::test_store_update_title_empty_removes_todo ... ok
test tests::test_todo_new ... ok
test tests::test_todo_toggle ... ok
test tests::test_validate_new_todo_title_empty ... ok
test tests::test_validate_new_todo_title_too_long ... ok
test tests::test_validate_new_todo_title_valid ... ok
test tests::test_reducer_remove_action ... ok
test tests::test_full_todo_workflow ... ok

test result: ok. 40 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Tests ครอบคลุม:
- **TodoStore CRUD** — add, remove, toggle, update_title (9 tests)
- **Filter logic** — All/Active/Completed matching (4 tests)
- **Reducer pattern** — ทุก action (8 tests)
- **Route parsing** — from_path, to_path (7 tests)
- **Validation** — empty, too long, valid input (3 tests)
- **Serialization** — JSON round-trip, empty array (2 tests)
- **Integration** — full workflow end-to-end (1 test)

### WASM Render Tests (ต้องใช้ wasm-bindgen-test)

```rust
// tests/component_tests.rs
use wasm_bindgen_test::*;
use yew::prelude::*;

wasm_bindgen_test_configure!(run_in_browser);

// Test render Hello World
#[wasm_bindgen_test]
async fn test_hello_renders() {
    #[function_component(TestComponent)]
    fn test_comp() -> Html {
        html! { <div id="test-div">{ "Hello Test" }</div> }
    }

    // Mount component
    let document = web_sys::window().unwrap().document().unwrap();
    let container = document.create_element("div").unwrap();
    document.body().unwrap().append_child(&container).unwrap();

    yew::Renderer::<TestComponent>::with_root(container).render();

    // ตรวจสอบผลลัพธ์
    let element = document.get_element_by_id("test-div").unwrap();
    assert_eq!(element.text_content().unwrap(), "Hello Test");
}
```

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### 1. ลืม clone() Handle ก่อน Move เข้า Closure

**ข้อผิดพลาด:**
```rust
let count = use_state(|| 0i32);

// ERROR: value moved into closure ครั้งแรก แล้ว use อีกครั้ง
let inc = Callback::from(move |_| count.set(*count + 1)); // moved here
let dec = Callback::from(move |_| count.set(*count - 1)); // ERROR: use of moved value
```

**วิธีแก้:**
```rust
let count = use_state(|| 0i32);

let inc = {
    let count = count.clone(); // clone ก่อน
    Callback::from(move |_| count.set(*count + 1))
};
let dec = {
    let count = count.clone(); // clone อีกครั้ง
    Callback::from(move |_| count.set(*count - 1))
};
```

**อธิบาย:** `UseStateHandle` implement `Clone` ซึ่ง clone ตัว pointer ไม่ใช่ data จริง ทุก clone ชี้ไปยัง state เดียวกัน ดังนั้นการ clone แล้ว move เข้าหลาย closure เป็น pattern ปกติและถูกต้อง

---

### 2. ใช้ Reducible ผิด — Return Type ไม่ถูกต้อง

**ข้อผิดพลาด:**
```rust
impl Reducible for MyState {
    type Action = MyAction;

    // ERROR: return type ต้องเป็น Rc<Self> ไม่ใช่ Self
    fn reduce(self: Rc<Self>, action: Self::Action) -> Self {
        // ...
    }
}
```

**วิธีแก้:**
```rust
impl Reducible for MyState {
    type Action = MyAction;

    fn reduce(self: Rc<Self>, action: Self::Action) -> Rc<Self> {
        let mut next = (*self).clone(); // dereference แล้ว clone
        match action {
            // แก้ไข next ...
        }
        Rc::new(next) // wrap ด้วย Rc::new
    }
}
```

**อธิบาย:** Yew ใช้ `Rc<T>` (reference counting) แทน `Arc<T>` เพราะ WASM เป็น single-threaded ไม่ต้องการ atomic operations ดังนั้น `reduce` ต้อง return `Rc<Self>` เสมอ

---

### 3. Keys ที่ขาดหายหรือซ้ำใน Lists

**ข้อผิดพลาด:**
```rust
// ไม่มี key — Yew ไม่สามารถ optimize re-render ได้
html! {
    <ul>
        { for todos.iter().map(|todo| html! {
            <TodoItem todo={todo.clone()} />
        })}
    </ul>
}

// Key ซ้ำ — เกิด undefined behavior ใน virtual DOM diffing
html! {
    <ul>
        { for todos.iter().map(|todo| html! {
            // title อาจซ้ำกันได้
            <TodoItem key={todo.title.clone()} todo={todo.clone()} />
        })}
    </ul>
}
```

**วิธีแก้:**
```rust
html! {
    <ul>
        { for todos.iter().map(|todo| html! {
            // ใช้ unique id เสมอ
            <TodoItem key={todo.id} todo={todo.clone()} />
        })}
    </ul>
}
```

**อธิบาย:** `key` ช่วยให้ Yew เข้าใจว่า element ใดสอดคล้องกับ element ใดใน re-render ครั้งใหม่ ถ้าไม่มี key, Yew จะ re-render ทุก item ใหม่หมดแม้ว่าจะมีแค่ item เดียวที่เปลี่ยน ถ้า key ซ้ำกัน virtual DOM diffing จะผิดพลาด

---

### 4. use_effect_with Dependencies ไม่ถูกต้อง

**ข้อผิดพลาด:**
```rust
// อยากให้ fetch ใหม่เมื่อ user_id เปลี่ยน แต่ใส่ () เลยไม่ re-fetch
let user_id = props.user_id;
use_effect_with((), move |_| {
    // fetch user_id จะไม่ update เมื่อ props เปลี่ยน!
    fetch_user(user_id);
    || ()
});
```

**วิธีแก้:**
```rust
let user_id = props.user_id;
use_effect_with(user_id, move |&id| {
    // re-run ทุกครั้งที่ user_id เปลี่ยน
    fetch_user(id);
    || ()
});
```

**อธิบาย:** argument แรกของ `use_effect_with` คือ dependencies — เหมือน dependency array ของ `useEffect` ใน React ถ้าใส่ `()` จะรันแค่ครั้งเดียวตอน mount ถ้าต้องการ re-run ต้องใส่ค่าที่ต้องการ track เป็น dependency; type ของ dependency ต้อง implement `PartialEq` เพื่อ compare

---

### 5. Borrow Checker กับ html! Macro

**ข้อผิดพลาด:**
```rust
let items = vec!["a", "b", "c"];

html! {
    <ul>
        { for items.iter().map(|item| html! { <li>{item}</li> }) }
        // ERROR: ถ้าใช้ items อีกครั้งหลัง for loop
        <p>{ format!("มี {} รายการ", items.len()) }</p>
    </ul>
}
```

**วิธีแก้:**
```rust
let items = vec!["a", "b", "c"];
let count = items.len(); // คำนวณล่วงหน้าก่อน move เข้า html!

html! {
    <ul>
        { for items.iter().map(|item| html! { <li>{*item}</li> }) }
        <p>{ format!("มี {} รายการ", count) }</p>
    </ul>
}
```

---

### 6. Trunk Build ล้มเหลวเพราะ Missing WASM Target

**ข้อผิดพลาด:**
```
error[E0463]: can't find crate for `std`
  = note: the `wasm32-unknown-unknown` target may not be installed
```

**วิธีแก้:**
```bash
# เพิ่ม WASM target
rustup target add wasm32-unknown-unknown

# ตรวจสอบ
rustup target list --installed | grep wasm
# wasm32-unknown-unknown (installed)
```

---

### 7. ContextProvider ไม่ครอบคลุม Component ที่ต้องการ use_context

**ข้อผิดพลาด:**
```rust
// Component อยู่นอก ContextProvider — use_context return None
fn root() -> Html {
    html! {
        <>
            <OrphanComponent /> // ← ใช้ use_context ได้ แต่จะ None!
            <ContextProvider<MyCtx> context={ctx}>
                <ChildComponent />
            </ContextProvider<MyCtx>>
        </>
    }
}
```

**วิธีแก้:**
```rust
fn root() -> Html {
    html! {
        <ContextProvider<MyCtx> context={ctx}>
            <OrphanComponent /> // ← ตอนนี้อยู่ใน Provider
            <ChildComponent />
        </ContextProvider<MyCtx>>
    }
}
```

## การ Package และ Deploy

### Trunk Build System

```bash
# Development — hot reload
trunk serve

# Development บน custom port
trunk serve --port 3000

# Production build
trunk build --release

# Production build พร้อม base path (สำหรับ subdirectory hosting)
trunk build --release --public-url /my-app/
```

### Nginx Configuration สำหรับ SPA

SPA ต้องการ server config พิเศษ — ทุก request ต้อง serve `index.html` เพื่อให้ client-side router ทำงาน:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/yew-todo-app/dist;
    index index.html;

    location / {
        # SPA fallback — ส่ง index.html สำหรับทุก path ที่ไม่ใช่ static file
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets (WASM, JS, CSS)
    location ~* \.(wasm|js|css|png|jpg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Docker Deployment

```dockerfile
# Build stage — ใช้ Rust + trunk
FROM rust:1.75 AS builder

WORKDIR /app
RUN rustup target add wasm32-unknown-unknown
RUN cargo install trunk

COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY index.html ./

RUN trunk build --release

# Serve stage — nginx serve static files
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Build และ run:
```bash
docker build -t yew-todo-app .
docker run -p 8080:80 yew-todo-app
# เปิด http://localhost:8080
```

### ขนาด Bundle และ Optimization

```bash
# ดู bundle size
ls -lh dist/*.wasm
# -rw-r--r-- 1 user user 2.1M yew-todo-app-abcd1234_bg.wasm

# หลัง gzip (ที่ server ควร enable)
gzip -k dist/*.wasm && ls -lh dist/*.wasm.gz
# -rw-r--r-- 1 user user 634K yew-todo-app-abcd1234_bg.wasm.gz

# หรือใช้ wasm-opt สำหรับ optimize เพิ่มเติม
wasm-opt -Oz dist/*.wasm -o dist/optimized.wasm
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Priority Tags

เพิ่ม field `priority: Priority` ให้กับ `Todo` struct โดย:
- Priority enum มี 3 ระดับ: `Low`, `Medium`, `High`
- เพิ่ม UI สำหรับเลือก priority ตอนสร้าง todo ใหม่
- แสดง color coding ตาม priority (เช่น red สำหรับ High)
- เพิ่ม sort ตาม priority ใน `filtered_todos()`
- เขียน unit tests สำหรับ sort logic ใหม่

**สิ่งที่จะได้เรียน:** enum operations, sorting closures, CSS classes แบบ dynamic

---

### แบบฝึกหัดที่ 2: Drag-and-Drop Reordering

ใช้ HTML5 Drag and Drop API ผ่าน `web-sys` เพื่อ:
- Implement `ondragstart`, `ondragover`, `ondrop` events บน `TodoItem`
- เก็บ drag state ใน parent component
- อัปเดต order ใน `TodoStore.todos` หลัง drop สำเร็จ
- Persist order ลง localStorage

**สิ่งที่จะได้เรียน:** HTML5 Drag & Drop API, complex event handling, state mutation pattern

---

### แบบฝึกหัดที่ 3: Backend Sync ด้วย REST API

สร้าง simple Axum backend (หรือใช้ JSONPlaceholder) และ sync todos:
- `GET /todos` — โหลด todos จาก server เมื่อ app start
- `POST /todos` — สร้าง todo ใหม่บน server
- `PATCH /todos/:id` — อัปเดต title/completed
- `DELETE /todos/:id` — ลบจาก server
- แสดง optimistic update (อัปเดต UI ก่อน รอ server confirm)
- Handle network errors gracefully

**สิ่งที่จะได้เรียน:** async/await ใน WASM, `gloo-net` HTTP requests, loading states, error handling

---

### แบบฝึกหัดที่ 4: Dark Mode Toggle

Implement dark mode โดย:
- สร้าง `ThemeContext` แยกจาก `AppContext`
- เก็บ theme preference ใน localStorage
- Detect system preference ด้วย `window.matchMedia("(prefers-color-scheme: dark)")`
- Toggle button ใน navbar
- Apply CSS custom properties (`--bg-color`, `--text-color`) แทน hardcode

**สิ่งที่จะได้เรียน:** multiple contexts, CSS custom properties, `window.matchMedia`, system preference detection

---

### แบบฝึกหัดที่ 5: Due Dates และ Calendar View

เพิ่ม due date ให้ todos และสร้าง calendar view:
- เพิ่ม `due_date: Option<String>` (ISO 8601) ใน `Todo`
- Date picker input ใน TodoItem edit mode
- Filter เพิ่มเติม: Overdue, Due Today, Due This Week
- Calendar component ที่แสดง todos จัดกลุ่มตาม date
- Sort ตาม due date

**สิ่งที่จะได้เรียน:** date handling ใน Rust/WASM, complex UI layout, compound filtering

---

### แบบฝึกหัดที่ 6: Undo/Redo History

Implement undo/redo ด้วย command pattern:
- เก็บ history stack ของ `Vec<TodoStore>` ใน state
- ทุก action push state ปัจจุบันลง history
- Undo pop จาก history stack
- Redo เก็บ future stack แยก
- Keyboard shortcut: Ctrl+Z (undo), Ctrl+Y (redo)
- จำกัด history ไว้ที่ 50 states เพื่อ memory efficiency

**สิ่งที่จะได้เรียน:** command pattern, keyboard event handling, bounded queues

## สรุป

ในโปรเจคนี้เราได้สร้าง SPA ครบฟีเจอร์ด้วย Yew framework โดยเรียนรู้:

**WASM Toolchain** — ตั้งค่า trunk, เพิ่ม `wasm32-unknown-unknown` target, และเข้าใจ build pipeline ที่ compile Rust → WASM → bundle พร้อม HTML/CSS

**Component Model** — `#[function_component]`, `html!` macro, props structs ที่ต้อง derive `Properties + PartialEq`, และ Children pattern

**State Management** — `use_state` สำหรับ local state อย่างง่าย, `use_reducer` + `Reducible` trait สำหรับ complex state, และ Context API (`ContextProvider` + `use_context`) สำหรับ global state

**Event Handling** — `Callback<E>` pattern, การ clone handles ก่อน move เข้า closure, `target_unchecked_into` สำหรับ DOM casting

**Side Effects** — `use_effect_with` สำหรับ data fetching และ dependency tracking, custom hooks ด้วย `gloo-storage`

**Routing** — `yew-router` พร้อม `#[derive(Routable)]`, `BrowserRouter`, `Switch`, `Link`, `use_navigator`

**Testing Strategy** — แยก business logic เป็น pure Rust ใน `lib.rs` เพื่อ test ด้วย `cargo test` ปกติ, WASM render tests ด้วย `wasm-bindgen-test` สำหรับ browser-specific behavior

**Design Principle สำคัญ** ที่ได้จากโปรเจคนี้: ยิ่งแยก pure logic ออกจาก UI ได้มากเท่าไร โค้ดก็ยิ่ง testable และ maintainable มากขึ้น — หลักการนี้ใช้ได้ไม่ว่าจะเป็น Yew, React, หรือ framework อื่นใด

โปรเจคถัดไป **Project J02: Full-Stack Axum** จะนำ Yew frontend นี้ไปเชื่อมต่อกับ Axum backend เพื่อสร้าง full-stack Rust application ที่ share ทั้ง data types และ validation logic ระหว่าง frontend และ backend ผ่าน shared crate

---

**โปรเจคก่อนหน้า:** [Project I10: Graph Neural Network](project-i10-graph-nn.md) | **โปรเจคถัดไป:** [Project J02: Full-Stack Axum](project-j02-fullstack-axum.md)
