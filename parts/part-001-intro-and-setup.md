# Part 1: แนะนำ Rust และการติดตั้งเครื่องมือ

> โมดูล: เริ่มต้นใช้งาน Rust (Getting Started) | ระดับ: พื้นฐาน | เวลาโดยประมาณ: 60 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า Rust คืออะไร แก้ปัญหาอะไร และเหมาะกับงานประเภทไหน
- เข้าใจแนวคิดหลักที่ทำให้ Rust แตกต่างจากภาษาอื่น (ownership, zero-cost abstraction, ไม่มี garbage collector)
- ติดตั้ง Rust toolchain (`rustup`, `cargo`, `rustc`) บนเครื่องของตัวเองได้ ไม่ว่าจะใช้ Windows, macOS หรือ Linux
- ตั้งค่า editor (VS Code + rust-analyzer) ให้พร้อมสำหรับการเขียนโค้ด Rust
- เขียนและรันโปรแกรม "Hello, World!" โปรแกรมแรกได้ด้วยตัวเอง
- เข้าใจภาพรวมของ toolchain: `rustc`, `cargo`, `rustup`, `rustfmt`, `clippy` ว่าแต่ละตัวทำหน้าที่อะไร

## ความรู้ที่ต้องมีมาก่อน

ไม่มี — นี่คือบทแรกของหลักสูตร ขอเพียงคุณมีคอมพิวเตอร์ที่เชื่อมต่ออินเทอร์เน็ตได้ และเคยเปิด terminal/command line มาบ้าง
(ไม่จำเป็นต้องเขียนโปรแกรมเก่งมาก่อน แต่ถ้าเคยเขียนภาษาอื่นมาบ้าง เช่น Python, JavaScript, C++, Java จะช่วยให้เข้าใจการเปรียบเทียบได้ง่ายขึ้น)

## เนื้อหา

### 1.1 Rust คืออะไร และทำไมถึงสำคัญ

Rust เป็นภาษาโปรแกรมมิ่งระดับระบบ (systems programming language) ที่พัฒนาโดย Mozilla Research เริ่มต้นในปี 2006
โดย Graydon Hoare และเปิดตัวเวอร์ชันเสถียรตัวแรก (1.0) ในปี 2015 ปัจจุบัน Rust ถูกดูแลโดย **Rust Foundation**
ซึ่งมีบริษัทใหญ่ ๆ อย่าง AWS, Google, Microsoft, Huawei, Meta เป็นสมาชิกผู้ก่อตั้ง

จุดขายหลักของ Rust คือประโยคที่ว่า:

> **"Fast, reliable, and productive — pick all three"**  
> (เร็ว น่าเชื่อถือ และเขียนได้อย่างมีประสิทธิภาพ — ได้ทั้งสามอย่างพร้อมกัน)

ในโลกของการเขียนโปรแกรมระดับระบบ (systems programming) แบบดั้งเดิม เรามักต้องเลือกอย่างใดอย่างหนึ่งระหว่าง:

1. **ภาษาที่เร็วแต่เสี่ยงบั๊ก** เช่น C, C++ — คุณควบคุม memory ได้เต็มที่ ทำให้โปรแกรมเร็วมาก แต่ก็เสี่ยงต่อบั๊กประเภท
   use-after-free, buffer overflow, data race, null pointer dereference ซึ่งเป็นสาเหตุของช่องโหว่ความปลอดภัย
   (security vulnerability) กว่า 70% ในโปรเจกต์ C/C++ ขนาดใหญ่ตามรายงานของ Microsoft และ Google
2. **ภาษาที่ปลอดภัยแต่ช้ากว่าและกินทรัพยากรมากกว่า** เช่น Java, Python, C# — มี garbage collector (GC) ช่วยจัดการ memory
   ให้อัตโนมัติ ทำให้ปลอดภัยจากบั๊กประเภท memory corruption แต่ GC ก็มาพร้อม overhead ด้านเวลาและหน่วยความจำ
   ทำให้ไม่เหมาะกับงานที่ต้องการ performance สูงสุดหรือ latency ต่ำมาก ๆ

**Rust แก้ปัญหานี้ด้วยแนวคิด "Ownership" และ "Borrow Checker"** ซึ่งเป็นระบบตรวจสอบความถูกต้องของการใช้ memory
**ที่ทำงานตอน compile time (ไม่ใช่ runtime)** พูดง่าย ๆ คือ compiler ของ Rust (`rustc`) จะปฏิเสธไม่ให้โค้ดที่มีความเสี่ยงต่อ
memory bug คอมไพล์ผ่านได้เลย ผลลัพธ์คือ:

- โปรแกรมที่คอมไพล์ผ่าน **จะไม่มี** null pointer dereference, use-after-free, double-free, data race (ในโค้ดที่เป็น "safe Rust")
- ไม่มี garbage collector จึงไม่มี overhead จาก GC — ความเร็วเทียบเท่า C/C++
- ยังคงเขียนโค้ดได้สะดวกในระดับที่ใกล้เคียงกับภาษาระดับสูง (high-level language) เช่นมี generics, closures, pattern matching,
  trait system ที่ทรงพลัง

Rust ติดโหวต **"ภาษาโปรแกรมมิ่งที่นักพัฒนารักมากที่สุด" (Most Loved Programming Language)** ใน Stack Overflow Developer Survey
ต่อเนื่องกันหลายปี (2016–2023+) และถูกนำไปใช้ในโปรเจกต์สำคัญระดับโลก เช่น:

- **Linux kernel** — เริ่มรองรับการเขียน driver ด้วย Rust ตั้งแต่ปี 2022
- **Windows** — Microsoft ใช้ Rust เขียนส่วนประกอบของระบบปฏิบัติการ (เช่น บางส่วนของ DirectWrite, Windows kernel components)
- **Firefox** — เอนจิน rendering ตัวใหม่ (Servo, Stylo) เขียนด้วย Rust
- **Discord** — ย้าย service สำคัญจาก Go มาเป็น Rust เพื่อลด latency spike จาก garbage collector
- **Cloudflare, AWS (Firecracker), Dropbox, npm registry, Deno, Figma** และอีกมากมาย ใช้ Rust ในระบบ production

### 1.2 Rust เหมาะกับงานประเภทไหน

Rust ถูกออกแบบมาให้เป็น "ภาษาสารพัดประโยชน์" (general-purpose) แต่จุดแข็งชัดเจนที่สุดในงานประเภทนี้:

| ประเภทงาน | ตัวอย่าง |
|---|---|
| Systems programming | Operating system, driver, embedded firmware |
| Network services / Backend | Web server, API service, microservices, proxy (เช่น เนื้อหาที่เราจะเรียนในโมดูล 4) |
| CLI tools | เครื่องมือ command-line เช่น `ripgrep`, `fd`, `bat`, `starship` |
| WebAssembly (WASM) | โค้ดที่รันในเบราว์เซอร์ด้วยความเร็วใกล้เคียง native (โมดูล 5) |
| Game engines | Bevy, Fyrox |
| Blockchain | Solana, Polkadot/Substrate เขียนด้วย Rust ทั้งหมด |
| Data engineering / performance-critical code | Apache Arrow (ส่วน Rust), Polars (DataFrame library ที่เร็วกว่า pandas มาก) |

สิ่งที่ Rust **ไม่ใช่** เครื่องมือที่ดีที่สุดเสมอไป: งาน prototype ด่วน ๆ ที่ไม่สนใจ performance เลย หรือ script สั้น ๆ
ใช้ครั้งเดียวทิ้ง — กรณีเหล่านี้ Python อาจเขียนเร็วกว่า แต่เมื่อโปรเจกต์เติบโตขึ้นและต้องการความน่าเชื่อถือระยะยาว
Rust จะเริ่มแสดงจุดแข็งของมันชัดเจนขึ้นเรื่อย ๆ

### 1.3 แนวคิดหลัก 3 อย่างที่ทำให้ Rust แตกต่าง

ก่อนจะติดตั้งเครื่องมือ เรามาทำความเข้าใจภาพกว้าง ๆ ของ 3 แนวคิดที่จะปรากฏซ้ำ ๆ ตลอดหลักสูตรนี้
(ยังไม่ต้องเข้าใจลึกตอนนี้ — เราจะอธิบายรายละเอียดใน Part 6-7)

1. **Ownership (การเป็นเจ้าของ)** — ทุกค่า (value) ในโปรแกรม Rust มี "เจ้าของ" เพียงหนึ่งเดียวในเวลาใดเวลาหนึ่ง
   เมื่อเจ้าของหมดอายุ (ออกจาก scope) ค่านั้นจะถูกทำลาย (drop) โดยอัตโนมัติทันที ไม่ต้องรอ garbage collector
2. **Borrowing (การยืม)** — คุณสามารถ "ยืม" ค่ามาใช้ชั่วคราวโดยไม่ต้องเป็นเจ้าของ ผ่าน reference (`&T`, `&mut T`)
   compiler จะตรวจสอบว่าการยืมนั้นปลอดภัยเสมอ (ไม่มีใครเขียนพร้อมกับที่อีกคนอ่าน เป็นต้น)
3. **Zero-cost abstraction** — feature ระดับสูงของ Rust (เช่น iterator, generic, trait) ถูกออกแบบให้ compiler แปลงเป็น
   machine code ที่มีประสิทธิภาพเทียบเท่ากับการเขียนโค้ดระดับต่ำด้วยมือ คุณไม่ต้องเสียสละความเร็วเพื่อความสะดวก

### 1.4 ติดตั้ง Rust ด้วย `rustup`

`rustup` คือตัวจัดการเวอร์ชันของ Rust toolchain (คล้าย `nvm` สำหรับ Node.js หรือ `pyenv` สำหรับ Python)
เป็นวิธีติดตั้งที่ทีม Rust official แนะนำ เพราะช่วยให้อัปเดตเวอร์ชัน, สลับ toolchain (stable/beta/nightly),
และเพิ่ม target สำหรับ cross-compilation (เช่น WASM ในโมดูล 5) ได้ง่าย

#### macOS และ Linux

เปิด terminal แล้วรันคำสั่งนี้ (คำสั่ง official จาก rust-lang.org):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

ระบบจะถามตัวเลือกการติดตั้ง ให้เลือก **`1) Proceed with installation (default)`** เพื่อติดตั้งแบบมาตรฐาน
หลังติดตั้งเสร็จ ให้รีสตาร์ท terminal หรือรันคำสั่งนี้เพื่อโหลด environment variable ทันที:

```bash
source "$HOME/.cargo/env"
```

#### Windows

มี 2 วิธีหลัก:

**วิธีที่ 1 (แนะนำ): ใช้ `rustup-init.exe`**

1. ดาวน์โหลดและรัน `rustup-init.exe` จากหน้าเว็บ official: https://www.rust-lang.org/tools/install
2. โปรแกรมจะตรวจสอบว่ามี **Microsoft C++ Build Tools** หรือยัง (Rust บน Windows ต้องใช้ linker จาก Visual Studio)
   ถ้ายังไม่มี ระบบจะแนะนำให้ติดตั้ง "Desktop development with C++" workload จาก Visual Studio Installer
3. เลือก `1) Proceed with installation (default)`

**วิธีที่ 2: ใช้ WSL (Windows Subsystem for Linux)**

หากคุณต้องการประสบการณ์แบบ Linux เต็มรูปแบบ (แนะนำสำหรับงาน backend/web development ในโมดูล 4-6)
ให้ติดตั้ง WSL2 ก่อน แล้วรันคำสั่งเดียวกับ Linux ข้างต้นภายใน WSL terminal

```powershell
# รันใน PowerShell (Administrator) เพื่อติดตั้ง WSL2
wsl --install
```

#### ตรวจสอบการติดตั้ง

หลังติดตั้งเสร็จ (ทุก OS) ให้ตรวจสอบว่าติดตั้งสำเร็จด้วยคำสั่ง:

```bash
rustc --version
cargo --version
rustup --version
```

ผลลัพธ์ที่ควรได้ (เวอร์ชันอาจใหม่กว่านี้เมื่อคุณอ่านบทนี้ — ไม่เป็นไร ใช้เวอร์ชัน stable ล่าสุดได้เลย):

```
rustc 1.83.0 (90b35a623 2024-11-26)
cargo 1.83.0 (5ffbef321 2024-10-29)
rustup 1.27.1 (54dd3d00f 2024-04-24)
```

- `rustc` = compiler ตัวจริงที่แปลง Rust source code (`.rs`) เป็น machine code
- `cargo` = build tool + package manager (เราจะใช้ตัวนี้เป็นหลักเกือบตลอดหลักสูตร แทบไม่เรียก `rustc` ตรง ๆ เลย)
- `rustup` = ตัวจัดการเวอร์ชันของ toolchain ทั้งหมด

### 1.5 ทำความเข้าใจ Rust Toolchain แต่ละตัว

| เครื่องมือ | หน้าที่ | ใช้บ่อยแค่ไหน |
|---|---|---|
| `rustc` | Compiler หลัก แปลงโค้ด `.rs` → binary | ไม่ค่อยเรียกตรง ๆ (cargo เรียกให้อัตโนมัติ) |
| `cargo` | Build system, package manager, test runner, ตัวจัดการ dependency | ใช้ทุกวัน |
| `rustup` | จัดการเวอร์ชัน toolchain (stable/beta/nightly), เพิ่ม target สำหรับ cross-compile | ใช้ตอน setup/อัปเดต |
| `rustfmt` | จัดรูปแบบโค้ดอัตโนมัติตามมาตรฐาน (จะเรียนใน Part 5) | ใช้ทุกวัน (ผ่าน editor หรือ `cargo fmt`) |
| `clippy` | Linter ที่ช่วยตรวจจับโค้ดที่ไม่ใช่ idiomatic Rust หรือมีข้อผิดพลาดเชิง logic (Part 5) | ใช้บ่อย |

Rust มี 3 "release channel":

1. **stable** — เวอร์ชันที่ผ่านการทดสอบแล้ว ออกใหม่ทุก 6 สัปดาห์ **ใช้ตัวนี้ตลอดหลักสูตร**
2. **beta** — เวอร์ชันก่อนหน้า stable รอบถัดไป ใช้สำหรับทดสอบก่อนปล่อยจริง
3. **nightly** — เวอร์ชันล่าสุดที่มี feature ทดลอง (unstable feature) ใช้เมื่อต้องการ feature ใหม่ที่ยังไม่ stabilize
   (จะพูดถึงอีกครั้งในโมดูล WASM/embedded ที่บาง feature ยังอยู่ใน nightly)

ตรวจสอบ/สลับ channel ด้วยคำสั่ง:

```bash
rustup show                    # ดู toolchain ที่ติดตั้งอยู่ และตัวที่ใช้งานอยู่ปัจจุบัน
rustup update                  # อัปเดต stable toolchain เป็นเวอร์ชันล่าสุด
rustup default stable          # กำหนดให้ stable เป็น default
rustup toolchain install nightly   # ติดตั้ง nightly เพิ่ม (ไม่ลบ stable)
```

### 1.6 ตั้งค่า Editor: VS Code + rust-analyzer

แม้ Rust จะใช้ editor ไหนก็ได้ (Vim, Neovim, IntelliJ RustRover, Emacs, Sublime Text) แต่หลักสูตรนี้จะแนะนำ
**Visual Studio Code (VS Code)** เพราะติดตั้งง่ายและมี extension official ที่ดีมาก

**ขั้นตอนติดตั้ง:**

1. ดาวน์โหลด VS Code จาก https://code.visualstudio.com/
2. เปิด VS Code → ไปที่แถบ Extensions (icon สี่เหลี่ยมด้านซ้าย หรือกด `Ctrl+Shift+X` / `Cmd+Shift+X`)
3. ค้นหา **"rust-analyzer"** (ผู้พัฒนา: The Rust Programming Language) แล้วกด Install
4. (แนะนำเพิ่ม) ติดตั้ง extension **"CodeLLDB"** สำหรับ debug โปรแกรม Rust แบบ step-through ได้

**rust-analyzer** คือ Language Server Protocol (LSP) implementation สำหรับ Rust มันจะให้:

- Auto-completion (พิมพ์ `.` แล้วเห็น method ที่ใช้ได้ทันที)
- แสดง type ของตัวแปรแบบ inline (inlay hints) — สำคัญมากสำหรับการเรียนรู้ เพราะ Rust ใช้ type inference บ่อย
- ตรวจจับ error/warning แบบ real-time (ไม่ต้องรอ compile เสร็จ)
- Go to definition, find references, rename symbol
- แสดง documentation เมื่อ hover เมาส์บนฟังก์ชัน/type

**ตั้งค่าเพิ่มเติมที่แนะนำ** (เปิด `settings.json` ด้วย `Ctrl+Shift+P` → พิมพ์ "Preferences: Open User Settings (JSON)"):

```json
{
  "editor.formatOnSave": true,
  "rust-analyzer.check.command": "clippy",
  "rust-analyzer.inlayHints.parameterHints.enable": true,
  "rust-analyzer.inlayHints.typeHints.enable": true
}
```

- `editor.formatOnSave: true` — จัดรูปแบบโค้ดด้วย `rustfmt` อัตโนมัติทุกครั้งที่ save (จะเข้าใจลึกใน Part 5)
- `rust-analyzer.check.command: "clippy"` — ใช้ clippy แทน `cargo check` ธรรมดา ทำให้เห็น warning ที่ละเอียดขึ้น

### 1.7 โปรแกรมแรก: "Hello, World!"

มาเขียนโปรแกรมแรกกัน เราจะทำ 2 วิธี: **วิธีตรง (rustc)** เพื่อเข้าใจพื้นฐาน และ **วิธีที่ใช้จริง (cargo)**
ซึ่งเป็นวิธีที่เราจะใช้ตลอดหลักสูตรตั้งแต่ Part 2 เป็นต้นไป

#### วิธีที่ 1: ใช้ `rustc` ตรง ๆ (เพื่อความเข้าใจ)

สร้างไฟล์ชื่อ `hello.rs`:

```rust
fn main() {
    println!("Hello, World!");
    println!("สวัสดี Rust!");
}
```

คอมไพล์และรัน:

```bash
rustc hello.rs
./hello        # บน Windows ใช้ .\hello.exe
```

ผลลัพธ์:

```
Hello, World!
สวัสดี Rust!
```

**อธิบายโค้ดทีละส่วน:**

- `fn main() { ... }` — `fn` คือ keyword ประกาศฟังก์ชัน `main` คือฟังก์ชันพิเศษที่ **ทุกโปรแกรม Rust ต้องมี**
  เป็นจุดเริ่มต้นการทำงานของโปรแกรม (เหมือน `main()` ใน C/C++/Java หรือโค้ด top-level ใน Python)
- `println!("...")` — สังเกตเครื่องหมาย `!` ต่อท้าย `println` นี่ไม่ใช่ฟังก์ชันธรรมดา แต่เป็น **macro**
  (เราจะเรียนเรื่อง macro ละเอียดใน Part 36) ตอนนี้จำไว้ก่อนว่า macro ที่ใช้พิมพ์ข้อความออกหน้าจอจะมี `!` เสมอ
- Rust รองรับ UTF-8 เต็มรูปแบบ จึงพิมพ์ภาษาไทยได้โดยไม่ต้องตั้งค่าอะไรเพิ่ม (แต่การ**จัดการ**ตัวอักษรไทยในระดับ index
  จะมีรายละเอียดที่ต้องระวัง ซึ่งจะพูดถึงใน Part 14 เรื่อง String และ UTF-8)
- ไม่มี semicolon หลังปีกกาเปิด/ปิดของฟังก์ชัน แต่**มี** semicolon (`;`) ท้ายแต่ละ statement ภายในฟังก์ชัน

#### วิธีที่ 2: ใช้ `cargo` (วิธีที่ใช้จริงตลอดหลักสูตร)

```bash
cargo new hello_cargo
cd hello_cargo
cargo run
```

คำสั่ง `cargo new hello_cargo` จะสร้างโครงสร้างโฟลเดอร์ให้อัตโนมัติ:

```
hello_cargo/
├── Cargo.toml
└── src/
    └── main.rs
```

และไฟล์ `src/main.rs` จะมีโค้ด "Hello, world!" ให้พร้อมใช้งานทันที เราจะเจาะลึกเรื่อง `Cargo.toml`,
โครงสร้างโปรเจกต์, และคำสั่ง cargo ต่าง ๆ (`build`, `run`, `check`, `test`) แบบละเอียดใน **Part 2** ถัดไป

ตอนนี้ให้สังเกตความแตกต่าง: `cargo run` ทำ 2 อย่างในคำสั่งเดียว คือ **compile** (`cargo build`) แล้ว **รัน binary
ที่ได้** ต่อทันที — สะดวกกว่าการเรียก `rustc` ตรง ๆ มาก และนี่เป็นเพียงส่วนเล็ก ๆ ของความสามารถของ cargo

### 1.8 เปรียบเทียบ Rust กับภาษาที่คุณอาจรู้จักมาก่อน

เพื่อช่วยปรับกรอบความคิด (mental model) มาดูตัวอย่างโค้ดเดียวกัน — คำนวณผลรวมของตัวเลข 1 ถึง 5 — ในภาษาต่าง ๆ:

**Python:**
```python
total = sum(range(1, 6))
print(total)  # 15
```

**JavaScript:**
```javascript
const total = Array.from({length: 5}, (_, i) => i + 1).reduce((a, b) => a + b, 0);
console.log(total); // 15
```

**Rust:**
```rust
fn main() {
    let total: i32 = (1..=5).sum();
    println!("{total}");  // 15
}
```

สังเกตว่า Rust เขียนได้กระชับไม่แพ้ภาษา scripting เลย (`(1..=5)` คือ range แบบ inclusive, `.sum()` คือ iterator
method — เราจะเรียนละเอียดใน Part 25-26) ความแตกต่างที่สำคัญคือ:

- Rust ต้องระบุ type ให้ตัวแปร `total` ชัดเจนในบางกรณี (ที่นี่คือ `: i32`) แต่ส่วนใหญ่ compiler จะ **inference**
  (เดา type) ให้อัตโนมัติจาก context — ไม่ต้องเขียน type ทุกที่แบบ Java/C#
- ไม่มี "null" หรือ "undefined" ใน Rust แบบที่ Python/JavaScript มี (จะเข้าใจลึกใน Part 11 เรื่อง `Option<T>`)
- โค้ด Rust ที่ compile ผ่านจะเร็วกว่า Python หลายสิบถึงหลายร้อยเท่าในงานประมวลผลหนัก เพราะไม่มี interpreter overhead

### 1.9 ภาพรวม Workflow การพัฒนาโปรแกรม Rust ที่จะใช้ตลอดหลักสูตร

เพื่อให้เห็นภาพว่าตลอดหลักสูตรนี้เราจะทำงานแบบไหน สรุป loop การทำงานทั่วไป:

1. เขียนโค้ดใน `src/main.rs` (หรือไฟล์อื่นที่จัดโครงสร้างไว้ — Part 16)
2. รัน `cargo check` บ่อย ๆ ระหว่างเขียน (เร็วกว่า `cargo build` มาก เพราะไม่สร้าง binary จริง แค่ตรวจ syntax/type)
3. รัน `cargo run` เมื่อต้องการรันจริง หรือ `cargo test` เมื่อต้องการรัน test (Part 32-33)
4. รัน `cargo fmt` เพื่อจัดรูปแบบโค้ด (หรือให้ editor ทำอัตโนมัติตอน save)
5. รัน `cargo clippy` เพื่อดู suggestion เชิง best-practice
6. Commit โค้ดด้วย git

Workflow นี้จะกลายเป็นความเคยชินอย่างรวดเร็วเมื่อคุณผ่านไปสัก 3-4 บท

## กับดักที่พบบ่อย (Common Pitfalls)

**1. ลืม source environment หลังติดตั้งบน Linux/macOS**

หลังรัน `rustup` แล้วพิมพ์ `cargo --version` ทันทีในหน้าต่าง terminal เดิม อาจได้ error:

```
command not found: cargo
```

วิธีแก้: ปิดแล้วเปิด terminal ใหม่ หรือรัน `source "$HOME/.cargo/env"` ก่อน (ตัว installer มักจะแก้ `~/.bashrc`
หรือ `~/.zshrc` ให้อัตโนมัติ แต่ terminal ที่เปิดอยู่ก่อนติดตั้งจะไม่รับรู้การเปลี่ยนแปลงจนกว่าจะเปิดใหม่)

**2. บน Windows ลืมติดตั้ง C++ Build Tools**

จะเจอ error แบบนี้ตอน `cargo build`:

```
error: linker `link.exe` not found
```

วิธีแก้: เปิด Visual Studio Installer → Modify → เลือก workload **"Desktop development with C++"** → Install
แล้วรัน `cargo build` ใหม่

**3. สับสนระหว่าง `rustc` กับ `cargo`**

มือใหม่หลายคนพยายามรัน `rustc src/main.rs` ในโปรเจกต์ที่สร้างด้วย `cargo new` แล้วงงว่าทำไม error
(เพราะโปรเจกต์ cargo มี dependency และโครงสร้างที่ `rustc` ตรง ๆ ไม่รู้จัก) **กฎง่าย ๆ**: ถ้าใช้ `cargo new`
สร้างโปรเจกต์ ให้ใช้คำสั่ง `cargo` ตลอด (`cargo run`, `cargo build`) ไม่ต้องเรียก `rustc` เองอีก

**4. VS Code ไม่แสดง autocomplete / inline error**

ถ้า rust-analyzer ติดตั้งแล้วแต่ไม่ทำงาน ให้ตรวจสอบว่าเปิดโฟลเดอร์ที่มีไฟล์ `Cargo.toml` อยู่ (เปิด **โฟลเดอร์โปรเจกต์**
ไม่ใช่เปิดแค่ไฟล์ `.rs` เดี่ยว ๆ) เพราะ rust-analyzer ต้องอ่าน `Cargo.toml` เพื่อรู้จัก dependency ทั้งหมดก่อน

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** ติดตั้ง Rust toolchain ให้สำเร็จ แล้วรัน `rustc --version`, `cargo --version` มา screenshot หรือ copy
   ผลลัพธ์เก็บไว้ (ใช้ตรวจสอบตัวเองว่าติดตั้งถูกต้อง)

2. **[ง่าย]** แก้ไขโปรแกรม `hello.rs` (วิธี rustc ตรง ๆ) ให้พิมพ์ชื่อของคุณเองแทนคำว่า "World" เช่น
   `println!("Hello, {}!", "สมชาย");` แล้วลองเปลี่ยนเป็นใช้ตัวแปร:
   ```rust
   fn main() {
       let name = "สมชาย";
       println!("Hello, {name}!");
   }
   ```
   สังเกตความแตกต่างของ syntax ทั้งสองแบบ (แบบ positional `{}`  กับแบบ named `{name}`)

3. **[กลาง]** สร้างโปรเจกต์ใหม่ด้วย `cargo new my_first_project` แล้วแก้ `src/main.rs` ให้พิมพ์ตารางสูตรคูณแม่ 5
   (5x1=5 ถึง 5x12=60) โดยใช้ `for` loop — แม้ยังไม่ได้เรียน `for` loop อย่างเป็นทางการ (จะเรียนใน Part 4)
   ให้ลองค้นหาและทดลองดูก่อน นี่คือวิธีเรียนที่ดีมากในการเขียนโปรแกรม: ลองผิดลองถูกจาก documentation
   (hint: `for i in 1..=12 { ... }` และใช้ `println!("5 x {i} = {}", 5 * i);`)

4. **[ยาก/ประยุกต์]** ค้นคว้าเพิ่มเติม (จาก rust-lang.org หรือ The Rust Book บทที่ 1) แล้วเขียนสรุปสั้น ๆ (3-5 บรรทัด)
   ว่า **"edition"** ของ Rust (2015, 2018, 2021, 2024) คืออะไร แตกต่างจากการอัปเดตเวอร์ชัน compiler อย่างไร
   และทำไม Rust ถึงออกแบบระบบ edition นี้ขึ้นมา (เกี่ยวข้องกับการรักษา backward compatibility)

## สรุป

ในบทนี้เราได้ทำความรู้จักกับ Rust ในภาพกว้าง: เข้าใจว่า Rust แก้ปัญหาความขัดแย้งระหว่าง "เร็ว" กับ "ปลอดภัย"
ที่ภาษาระบบดั้งเดิมเจอมาตลอดได้อย่างไรผ่านแนวคิด ownership และ borrow checker เราติดตั้ง Rust toolchain
ผ่าน `rustup` สำเร็จ ตั้งค่า editor ด้วย VS Code + rust-analyzer และเขียนโปรแกรมแรกทั้งแบบ `rustc` ตรง ๆ
และแบบ `cargo` ซึ่งเป็นวิธีที่เราจะใช้ตลอดหลักสูตรนี้

ใน **Part 2** เราจะเจาะลึกเรื่อง **Cargo** — build tool และ package manager ที่เป็นหัวใจของ ecosystem Rust
ทั้งหมด ตั้งแต่โครงสร้างไฟล์ `Cargo.toml`, การจัดการ dependency, ไปจนถึงคำสั่งต่าง ๆ ที่คุณจะใช้ทุกวันในการพัฒนา

---

**Part ถัดไป:** [Part 2: Cargo และโครงสร้างโปรเจกต์](part-002-cargo-and-project-structure.md)
