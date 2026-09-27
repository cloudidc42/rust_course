# Part 97: CI/CD Pipeline ด้วย GitHub Actions

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้ว่า CI/CD แก้ปัญหาอะไรให้กับทีมพัฒนา Rust โดยเฉพาะ และทำไม "compile time ที่ช้า" ถึงเป็นประเด็นสำคัญ
  ที่สุดที่ต้องออกแบบ pipeline ให้รับมือกับมันอย่างจริงจัง ไม่ใช่แค่ copy workflow ของภาษาอื่นมาใช้ตรง ๆ
- เขียน GitHub Actions workflow (ไฟล์ YAML ใน `.github/workflows/`) ตั้งแต่ตัวที่ง่ายที่สุดไปจนถึง pipeline
  ระดับ production จริง เข้าใจ syntax ของ `on`, `jobs`, `steps`, `runs-on`, `strategy.matrix`, `services`,
  `needs`, `if`, `permissions` และ `env` ครบทุกจุดที่ใช้งานจริง
- ตั้งค่า caching สำหรับ dependency ของ Cargo อย่างถูกต้อง ด้วย `Swatinem/rust-cache` และเข้าใจว่าทำไมการ cache
  โปรเจกต์ Rust ถึงซับซ้อนกว่าภาษาอื่น (เชื่อมโยงกับแนวคิด layer caching ของ Docker จาก Part 96)
- สร้าง CI pipeline ที่รัน `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo build`, และ `cargo test`
  ครบชุดทุกครั้งที่มี pull request พร้อม matrix build ข้าม Rust channel (stable/beta/nightly) และข้าม OS
- รัน integration test ที่ต้องพึ่ง PostgreSQL จริงบน GitHub Actions ด้วยฟีเจอร์ `services:` และเข้าใจปัญหาเฉพาะตัว
  ของ `sqlx::query!` macro ที่ต้องต่อฐานข้อมูลจริงตอน compile time
- วัด code coverage ด้วย `cargo-llvm-cov` และเข้าใจภาพรวมของการส่งผลลัพธ์ไปยังบริการภายนอกอย่าง Codecov
- เขียน workflow ที่ build Docker image (ต่อยอด Dockerfile จาก Part 96) แล้ว push ขึ้น GitHub Container Registry
  (`ghcr.io`) และเขียน release automation ที่สร้าง binary หลายแพลตฟอร์มพร้อมแนบเข้า GitHub Release อัตโนมัติ
- ใช้ `cargo-audit` ตรวจสอบ dependency ของโปรเจกต์เทียบกับฐานข้อมูลช่องโหว่ความปลอดภัย (RustSec Advisory Database)
  และผนวกเข้าเป็นส่วนหนึ่งของ CI pipeline
- ตั้งค่า Branch Protection Rules ให้ผลลัพธ์ของ CI "บังคับผ่านจริง" ก่อน merge ได้ และเข้าใจข้อจำกัดด้านความ
  ปลอดภัยของ `secrets`/`GITHUB_TOKEN` เมื่อ workflow ถูก trigger จาก pull request ของ fork ภายนอก

## ความรู้ที่ต้องมีมาก่อน

- **Part 5 (rustfmt และ Clippy)** — บทนี้ใช้ `cargo fmt --check` และ `cargo clippy -- -D warnings` เป็นหัวใจของ
  CI job ตัวแรกโดยตรง ต้องเข้าใจความต่างระหว่างสองคำสั่งนี้ และทำไม CI ต้องใช้ `--check` ไม่ใช่รัน `cargo fmt`
  แก้ไฟล์ตรง ๆ มาก่อนแล้ว
- **Part 32-33 (Unit Test และ Integration Test)** — pipeline ทั้งบทนี้มีหน้าที่หลักคือรัน test suite ที่สองบทนี้
  สอนให้เขียน ถ้ายังเขียน `#[test]`/`#[tokio::test]` ไม่คล่อง จะเข้าใจว่า "ทำไม CI ต้องรันแบบนี้" ได้ยาก
- **Part 70-71 (SQLx และ PostgreSQL)** — จำเป็นมากสำหรับหัวข้อ services container เพราะต้องเข้าใจว่า
  `sqlx::query!`/`query_as!` ตรวจสอบ SQL กับฐานข้อมูลจริงตอน compile time (compile-time verification) ซึ่งเป็น
  จุดที่ทำให้ CI สำหรับโปรเจกต์ที่ใช้ SQLx ต่างจากโปรเจกต์ Rust ทั่วไปอย่างชัดเจน
- **Part 92-94 (Full-Stack Capstone: library_api)** — บทนี้ใช้โปรเจกต์ `library_api` จากสามบทนี้เป็นตัวอย่างหลัก
  ตลอดทั้งบท โดยเฉพาะหัวข้อ capstone ท้ายบทที่เขียน pipeline สมบูรณ์สำหรับระบบนี้ตรง ๆ
- **Part 95 (Testing Web Applications)** — เข้าใจ testing pyramid ของ full-stack app (`oneshot` test, `#[sqlx::test]`,
  `wasm-pack test`) เพื่อรู้ว่า CI pipeline ต้องรัน test แต่ละชั้นด้วยวิธีที่ต่างกันอย่างไร
- **Part 96 (Docker และ Containerization)** — บทนี้อ้างอิง Dockerfile และแนวคิด layer caching, musl static
  linking จาก Part 96 โดยตรงในหัวข้อ build/push Docker image และ release binary ถ้า Part 96 ยังไม่เผยแพร่ในตอนที่
  คุณอ่านบทนี้ ให้เข้าใจว่าเนื้อหาที่อ้างถึงคือ "Dockerfile แบบ multi-stage build ที่ produce binary ขนาดเล็ก" และ
  "target `x86_64-unknown-linux-musl` สำหรับ static binary ที่รันบน distroless/scratch image ได้" ซึ่งเป็นแนวคิด
  มาตรฐานที่จะอธิบายซ้ำสั้น ๆ ในบทนี้เท่าที่จำเป็น
- **Part 17 (Packages, Crates, Workspaces)** — จำเป็นสำหรับความเข้าใจเรื่อง `Cargo.lock` ว่าทำไม cache key ของ
  หัวข้อ 97.4 ถึงต้องผูกกับไฟล์นี้ และเรื่อง Semantic Versioning ที่ใช้ตั้งชื่อ tag release ในหัวข้อ 97.10

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้มีข้อจำกัดในการตรวจสอบที่ต้องพูดตรง ๆ ก่อนเริ่ม: **GitHub Actions เป็นระบบที่รันบน infrastructure ของ GitHub
เท่านั้น** ไม่มีทางรัน workflow จริงแบบ end-to-end ได้จากเครื่อง/สภาพแวดล้อมที่ใช้เขียนบทนี้ (ไม่มี GitHub
repository จริงที่ push แล้วดู Actions tab ทำงาน) มีเครื่องมือชื่อ **`act`** ที่จำลองการรัน GitHub Actions workflow
บนเครื่อง local ผ่าน Docker ได้ — เราตรวจสอบแล้วด้วยคำสั่ง `which act` ว่า**เครื่องมือนี้ไม่ได้ติดตั้งอยู่ในสภาพ
แวดล้อมที่ใช้เขียนบทนี้** และการติดตั้งเพิ่มก็ยังต้องพึ่ง Docker daemon ที่จำลอง runner ของ GitHub ซึ่งไม่ตรง 100%
กับ runner จริงอยู่ดี (แม้ `act` มีอยู่จริงและเป็นเครื่องมือที่ทีมจริงใช้ debug workflow ก่อน push ก็ตาม — เราจะ
กล่าวถึงมันในหัวข้อ 97.2 สำหรับผู้อ่านที่ต้องการใช้)

ดังนั้น**วิธีที่เราใช้ตรวจสอบความถูกต้องของเนื้อหาบทนี้จริง** มีสองส่วน:

1. **รันทุกคำสั่งที่ workflow เรียกจริงบนเครื่อง** — `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`,
   `cargo build`, `cargo test`, `cargo llvm-cov`, `cargo audit` ทุกคำสั่งเหล่านี้ถูกรันจริงกับโปรเจกต์ Rust scratch
   ที่สร้างขึ้นเฉพาะสำหรับตรวจสอบบทนี้ (ลบทิ้งหลังตรวจสอบเสร็จ ไม่ปนกับไฟล์ในหลักสูตร) ผลลัพธ์ทั้งหมดที่แสดงเป็น
   ข้อความ error/warning ในบทนี้ — เช่น ข้อความเตือนของ `clippy::needless_return`, ผลลัพธ์ของ `cargo audit` ตอนพบ
   RUSTSEC advisory จริง, ตาราง coverage จาก `cargo llvm-cov` — **คือผลลัพธ์จริงที่ capture มาจากการรันจริง** ไม่ใช่
   ข้อความที่แต่งขึ้น
2. **ตรวจ syntax ของทุกไฟล์ YAML ด้วย YAML parser** (`python3` + `PyYAML`) เพื่อยืนยันว่าทุก workflow ในบทนี้เป็น
   YAML ที่ถูกต้องตามหลักไวยากรณ์ 100% (indentation, key ซ้อน, list ฯลฯ) ร่วมกับการตรวจสอบด้วยมืออย่างละเอียดเทียบกับ
   เอกสารทางการของ GitHub Actions และ action ทางการ/action ที่ได้รับการยอมรับกว้างขวางในชุมชน (`actions/checkout`,
   `actions/cache`, `Swatinem/rust-cache`, `dtolnay/rust-toolchain`, `docker/build-push-action`,
   `softprops/action-gh-release`, `codecov/codecov-action`, `taiki-e/install-action` ฯลฯ)

**สิ่งที่บทนี้ไม่สามารถยืนยันได้จริง** และจะบอกไว้ชัดเจนทุกครั้งที่เกี่ยวข้อง: การรัน workflow จริงบน GitHub
Actions runner, การ push image ขึ้น `ghcr.io` จริง, การอัปโหลดผลลัพธ์ coverage ไปยัง Codecov จริง (ต้องมี account
และ token จริงของ repository), และการสร้าง GitHub Release จริง — ทั้งหมดนี้ต้องมี GitHub repository จริงที่เชื่อม
ต่อบริการเหล่านี้ ซึ่งอยู่นอกเหนือสิ่งที่ sandbox การเขียนบทนี้ทำได้ เราจะอธิบาย "พฤติกรรมที่คาดหวัง" ของแต่ละ
ขั้นตอนเหล่านี้อย่างชัดเจนโดยอ้างอิงจากเอกสารทางการของ action นั้น ๆ พร้อมระบุไว้ว่าเป็นพฤติกรรมที่บันทึกไว้
(documented behavior) ไม่ใช่ผลจากการรันจริงในบทนี้

## เนื้อหา

### 97.1 CI/CD แก้ปัญหาอะไร และทำไม Rust ถึงมีมุมที่ต้องระวังเป็นพิเศษ

**CI (Continuous Integration)** คือแนวคิดที่ให้ทุกการเปลี่ยนแปลงโค้ด (ทุก commit, ทุก pull request) ถูกตรวจสอบ
อัตโนมัติทันทีด้วยชุดคำสั่งเดียวกัน บนสภาพแวดล้อมเดียวกัน โดยไม่ต้องพึ่งว่า "ผู้เขียนโค้ดคนนั้นจำได้ไหมว่าต้องรัน
test ก่อน push" — **CD (Continuous Delivery/Deployment)** ต่อยอดจากนั้นด้วยการทำให้การส่งมอบซอฟต์แวร์ (build
image, สร้าง release, deploy ขึ้น production) เป็นอัตโนมัติเช่นกันเมื่อโค้ดผ่านเงื่อนไขที่กำหนด

ฟังดูเป็นแนวคิดทั่วไปที่ใช้กับภาษาโปรแกรมมิ่งไหนก็ได้ แต่ Rust มีลักษณะเฉพาะบางอย่างที่ทำให้การออกแบบ CI/CD
pipeline สำหรับโปรเจกต์ Rust ต้องคิดมากกว่าปกติ:

**1. Rust เป็นภาษา compiled แบบเข้มงวด — ข้อผิดพลาดจำนวนมากถูกจับได้ "ก่อน" runtime อยู่แล้ว แต่ก็ต้องมีคนรันมัน**

จาก Part 12 เราเรียนแล้วว่า Rust ผ่าน borrow checker และ type system ที่เข้มงวดมาก ทำให้บั๊กหลายประเภทที่ภาษาอื่น
ต้องไปจับตอน runtime (null pointer, race condition ของ shared mutable state, use-after-free) ถูก compiler
ปฏิเสธตั้งแต่ `cargo build` ไม่ผ่าน — นี่คือข้อดีมหาศาลของ Rust แต่มันมีเงื่อนไขซ่อนอยู่: **ข้อดีนี้เกิดขึ้นได้ก็ต่อ
เมื่อมีคน (หรือเครื่อง) รัน `cargo build` จริง ๆ** ถ้าผู้เขียนโค้ด push การเปลี่ยนแปลงที่ตัวเองไม่ได้ build ล่าสุด
(เช่น แก้ merge conflict ด้วยมือแล้วไม่ได้ build ซ้ำ) ข้อดีทั้งหมดของ type system ก็ไม่มีประโยชน์อะไรถ้าไม่มีระบบ
บังคับให้ทุก commit ถูก build จริงก่อน merge — นี่คือหน้าที่พื้นฐานที่สุดของ CI: **เป็นประตูที่บังคับให้ทุกอย่าง
ที่ Rust compiler ตรวจสอบได้ ถูกตรวจสอบจริงทุกครั้ง ไม่มีข้อยกเว้น ไม่ขึ้นกับความจำของมนุษย์**

**2. Lint (clippy) และ format (rustfmt) ต้องถูกบังคับที่ระดับ pipeline ไม่ใช่แค่ "ขอความร่วมมือ"**

จาก Part 5 เราเรียนแล้วว่า `cargo fmt --check` และ `cargo clippy -- -D warnings` คืนค่า exit code ที่ไม่ใช่ 0
เมื่อพบปัญหา — คุณสมบัตินี้มีประโยชน์ก็ต่อเมื่อมีระบบที่ **อ่าน exit code นั้นแล้วบล็อกการ merge** ถ้าปล่อยให้
เป็นแค่ "แนวทางที่ขอให้ทุกคนรันเองก่อน push" ในทางปฏิบัติจะมีคนลืมเสมอ ไม่ช้าก็เร็ว — CI pipeline คือกลไกที่ทำให้
มาตรฐานเหล่านี้ **บังคับได้จริงในระดับ repository** ไม่ใช่แค่ระดับความรับผิดชอบส่วนบุคคล

**3. สภาพแวดล้อม build ที่ไม่สอดคล้องกัน (works on my machine)**

ปัญหานี้ไม่ใช่ปัญหาเฉพาะของ Rust แต่ Rust มีมุมที่ซับซ้อนกว่าเพิ่มเข้ามา: เวอร์ชันของ `rustc`/`cargo`, เวอร์ชันของ
system library ที่ crate บางตัวต้องพึ่ง (เช่น `openssl-sys` ต้องมี OpenSSL headers ในระบบ, `sqlx` ที่ใช้ฐานข้อมูล
จริงตอน compile), และแม้แต่ target platform (x86_64 vs aarch64) ล้วนส่งผลต่อว่าโค้ดจะ compile ผ่านหรือไม่ Part 96
สอนวิธีแก้ปัญหานี้ด้วย **Docker/containerization** — สร้าง image ที่มีสภาพแวดล้อม build ตายตัวแน่นอน ไม่ว่าจะรัน
ที่ไหน CI/CD ต่อยอดแนวคิดนี้อีกขั้น: **runner ของ GitHub Actions ก็คือเครื่องเปล่าที่ถูกสร้างใหม่ทุกครั้ง (ephemeral)
ด้วย image มาตรฐานเดียวกันเสมอ** ทำให้ "มัน build ผ่านบนเครื่องฉัน" ไม่ใช่คำตอบที่น่าเชื่อถืออีกต่อไป — คำตอบที่
น่าเชื่อถือคือ "มัน build ผ่านบน runner ที่ทุกคน (และทุก pull request) ใช้ร่วมกัน"

**4. ปัญหาเฉพาะของ Rust ที่ต้องออกแบบรับมือเป็นพิเศษ: compile time ที่ช้า**

นี่คือประเด็นที่จะเป็น**ธีมหลัก**ของบทนี้ Rust แลกความปลอดภัยและ performance ระดับสูงมากับเวลา compile ที่ช้ากว่า
ภาษาที่ compile แบบหลวม ๆ (เช่น Go) หรือภาษา interpreted/JIT (Python, JavaScript) อย่างมีนัยสำคัญ เหตุผลเชิงลึก
คือ: (ก) monomorphization ของ generic (จาก Part 18) สร้างโค้ดแยกสำหรับทุก concrete type ที่ใช้ ทำให้ปริมาณโค้ด
ที่ต้อง compile จริงมากกว่าที่เห็นในต้นฉบับ (ข) LLVM backend ที่ Rust ใช้ทำ optimization ละเอียดมากในโหมด release
(ค) แต่ละ crate dependency ต้อง compile จากซอร์สทุกตัว (ไม่มี pre-compiled binary package แบบ apt/npm โดย default)
โปรเจกต์ web application ระดับ production อย่าง `library_api` จาก Part 92 ที่มี dependency กว่า 15 ตัว (axum,
sqlx, tokio, utoipa ฯลฯ) แต่ละตัวก็ดึง transitive dependency ตามมาอีกหลายสิบ — การ compile จากศูนย์ (clean build)
อาจใช้เวลาหลายนาทีถึงสิบกว่านาทีบน CI runner ที่มี CPU/RAM จำกัดกว่าเครื่อง developer

ถ้า CI pipeline ต้อง clean build ทุกครั้งที่มี push หรือ pull request ใหม่ (และ pull request มักถูก push หลาย
รอบต่อวันระหว่าง code review) เวลารอผลจะยาวจนกระทบ productivity ของทีมอย่างจริงจัง — developer ที่ต้องรอ CI 15
นาทีทุกครั้งที่แก้ typo เล็ก ๆ จะเสีย flow การทำงานไปมาก นี่คือเหตุผลที่ **caching dependency อย่างถูกต้อง** ไม่ใช่
แค่ "ทางเลือกที่ดี" แต่เป็น**ส่วนที่ต้องออกแบบอย่างจริงจังตั้งแต่ workflow แรก** สำหรับโปรเจกต์ Rust — เราจะเจาะลึก
เรื่องนี้ในหัวข้อ 97.5 ซึ่งเป็นหัวใจสำคัญที่สุดหัวข้อหนึ่งของบทนี้

เปรียบเทียบให้เห็นภาพ: ทีมที่เขียน Node.js อาจแค่ `npm ci` (ติดตั้ง dependency ที่ pre-built มาแล้วจาก npm registry
เป็นไฟล์ JavaScript ธรรมดา ไม่ต้อง compile) แล้วรัน test ได้ในไม่กี่สิบวินาที ในขณะที่ทีม Rust ที่ไม่ตั้ง cache
ให้ดี อาจเสียเวลา compile dependency ซ้ำทุกครั้งนานกว่าตัวโค้ดของทีมเองเสียอีก — นี่ไม่ใช่ข้อเสียของภาษาที่แก้ไม่
ได้ แต่เป็น**ปัญหาทาง engineering ที่ต้องแก้ด้วยการออกแบบ pipeline ที่ดี** ซึ่งเป็นสิ่งที่บทนี้จะสอน

### 97.2 GitHub Actions พื้นฐาน: Workflow, Trigger, Job, Step, Runner

**GitHub Actions** คือระบบ CI/CD ที่ผนวกเข้ากับ GitHub โดยตรง ไม่ต้องตั้ง server แยก ไม่ต้องเชื่อม third-party
service ใด ๆ เพิ่มเติม (แม้จะเชื่อมได้ก็ตาม) — เพียงแค่สร้างไฟล์ YAML ไว้ในโฟลเดอร์ `.github/workflows/` ที่ root
ของ repository, push ขึ้น GitHub, ระบบจะรัน workflow นั้นอัตโนมัติตามเงื่อนไข (`trigger`) ที่กำหนดไว้ในไฟล์

**คำศัพท์หลักที่ต้องเข้าใจก่อน:**

- **Workflow** — ไฟล์ YAML หนึ่งไฟล์ในโฟลเดอร์ `.github/workflows/` (นามสกุล `.yml` หรือ `.yaml`) แต่ละไฟล์คือ
  workflow หนึ่งชุด ที่มี trigger, job, step ของตัวเอง โปรเจกต์หนึ่งมีได้หลาย workflow (เช่น `ci.yml` สำหรับ
  test, `release.yml` สำหรับ release)
- **Trigger (`on:`)** — เงื่อนไขที่ทำให้ workflow เริ่มรัน เช่น `push` (มีการ push commit), `pull_request`
  (มี PR ถูกเปิด/อัปเดต), `schedule` (ตามตารางเวลาแบบ cron), `workflow_dispatch` (กดรันด้วยมือผ่านหน้าเว็บ
  GitHub), หรือ tag push (`push: tags:`) สำหรับ release
- **Job** — หน่วยงานหนึ่งชุดที่รันบน **runner หนึ่งตัว** (เครื่องเสมือนหนึ่งเครื่อง) หนึ่ง workflow มีได้หลาย job
  และ job ต่างกันจะรัน**พร้อมกัน (parallel)** โดย default เว้นแต่ระบุ `needs:` ให้รอ job อื่นก่อน
- **Step** — คำสั่งย่อยหนึ่งคำสั่งภายใน job รันตามลำดับบนบรรทัด (sequential) ภายในเครื่องเดียวกัน มีสอง
  รูปแบบหลัก: `run:` (รันคำสั่ง shell ตรง ๆ) และ `uses:` (เรียกใช้ "Action" ที่คนอื่นหรือ GitHub เขียนไว้แล้ว
  เช่น `actions/checkout@v4` สำหรับ clone repository)
- **Runner** — เครื่องเสมือนจริงที่ GitHub เตรียมไว้ให้ (`ubuntu-latest`, `macos-latest`, `windows-latest`)
  หรือเครื่อง self-hosted ของทีมเอง แต่ละ job เริ่มต้นบน runner ที่ **สร้างใหม่ทั้งหมด (ephemeral)** ทุกครั้ง —
  ไม่มีไฟล์อะไรหลงเหลือจาก run ก่อนหน้า (นี่คือเหตุผลที่ต้อง cache อย่างชัดเจน จะพูดถึงในหัวข้อ 97.5)

มาดู workflow ที่**เรียบง่ายที่สุดที่ยังใช้งานได้จริง**: แค่ `cargo build` ทุกครั้งที่มี push ขึ้น branch `main`

```yaml
# .github/workflows/build.yml
name: Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build
        run: cargo build --verbose
```

**อธิบายทีละส่วน:**

- `name: Build` — ชื่อ workflow ที่จะแสดงในแท็บ "Actions" ของ GitHub repository (ไม่บังคับ แต่ควรใส่เสมอเพื่อ
  แยกแยะ workflow หลายตัว)
- `on: push: branches: [main]` — trigger: รันเมื่อมี push เข้า branch `main` เท่านั้น (ไม่รันตอน push เข้า branch
  อื่น หรือ PR — เราจะขยายให้ครอบคลุม PR ด้วยในหัวข้อถัดไป)
- `jobs: build:` — กำหนด job ชื่อ `build` (ชื่อนี้ตั้งเองได้ ใช้อ้างอิงใน `needs:` ของ job อื่นได้ในอนาคต)
- `runs-on: ubuntu-latest` — สั่งให้รันบน runner Ubuntu เวอร์ชันล่าสุดที่ GitHub ดูแลให้ (มี Rust toolchain
  ติดตั้งไว้แล้วบางเวอร์ชัน แต่ในทางปฏิบัติเราควรระบุ toolchain เองเสมอ ดังที่จะเห็นในหัวข้อถัดไป เพื่อไม่ให้ผลลัพธ์
  ขึ้นกับว่า GitHub อัปเดตเวอร์ชัน default บน runner เมื่อไหร่โดยที่เราไม่รู้ตัว)
- `steps:` — รายการคำสั่งที่รันตามลำดับ ก้อนแรก `actions/checkout@v4` เป็น **Action ทางการของ GitHub** ที่ clone
  repository ปัจจุบันเข้ามาในเครื่อง runner (ไม่มีขั้นตอนนี้ runner จะเป็นเครื่องเปล่าที่ไม่มีโค้ดของเราอยู่เลย —
  นี่คือ step ที่**ต้องมีเกือบทุก workflow เสมอ**) ก้อนที่สองรันคำสั่ง `cargo build --verbose` ตรง ๆ ผ่าน shell
  ของ runner (ซึ่งมี `cargo`/`rustc` ติดตั้งไว้แล้วบน `ubuntu-latest` image มาตรฐาน)

**ตัวเลข version ต่อท้าย action (`@v4`) สำคัญมาก:** action ทุกตัวบน GitHub Actions คือ repository หนึ่งอันที่มี
tag เวอร์ชัน — `@v4` หมายถึง "ใช้ tag major version 4 ล่าสุด" ซึ่งเป็นวิธี pin เวอร์ชันที่สมดุลระหว่างความ
ปลอดภัย (ได้ patch version ใหม่ ๆ อัตโนมัติ) กับความเสถียร (ไม่เปลี่ยน major version ที่อาจ breaking change แบบไม่
รู้ตัว) ทีมที่ต้องการความปลอดภัยสูงสุด (ป้องกัน supply-chain attack ที่ tag ถูกแก้ย้อนหลัง) อาจ pin เป็น commit SHA
เต็ม ๆ แทน (เช่น `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`) แต่บทนี้จะใช้ tag แบบ `@v4` ตาม
ธรรมเนียมทั่วไปที่โปรเจกต์ Rust ส่วนใหญ่ใช้ เพื่อให้อ่านง่ายและเห็นภาพชัด

**เกี่ยวกับ `act` — เครื่องมือรัน GitHub Actions บนเครื่อง local:** สำหรับผู้อ่านที่ต้องการทดสอบ workflow ก่อน
push จริง มีเครื่องมือ open-source ชื่อ [`act`](https://github.com/nektos/act) ที่จำลอง event ของ GitHub Actions
แล้วรัน workflow YAML ผ่าน Docker container บนเครื่องตัวเอง ติดตั้งได้ผ่าน package manager ทั่วไป (เช่น
`brew install act` บน macOS หรือดาวน์โหลด binary จาก release page) แล้วรันด้วยคำสั่งสั้น ๆ ที่ root ของ repository:

```bash
act push                    # จำลอง event push แล้วรัน job ที่ trigger ด้วย push
act pull_request            # จำลอง event pull_request
act -j build                # รันเฉพาะ job ชื่อ build
```

ข้อจำกัดที่ควรรู้ก่อนใช้: `act` รัน image Docker ที่จำลอง runner ของ GitHub แต่**ไม่ใช่ runner จริง 100%** —
บาง action ที่พึ่งพา feature เฉพาะของ GitHub (เช่น GitHub Container Registry authentication แบบอัตโนมัติผ่าน
`secrets.GITHUB_TOKEN`, artifact storage ของ GitHub) อาจทำงานต่างจากบน GitHub จริงเล็กน้อยหรือใช้งานไม่ได้เต็ม
รูปแบบ ทีมส่วนใหญ่ใช้ `act` เพื่อ**ตรวจ syntax และ logic คร่าว ๆ ก่อน push** (ประหยัดรอบการ push-แล้วรอดู) แต่ยัง
ต้องดูผลจริงบน GitHub Actions อีกครั้งเสมอสำหรับ workflow ที่ผลลัพธ์สำคัญ (เช่น release, docker push)

### 97.3 Trigger ที่ใช้บ่อยที่สุด: `push` และ `pull_request`

ในทางปฏิบัติ workflow สำหรับ CI (ตรวจ format/lint/test) มักต้องรันทั้งตอน push เข้า branch หลัก **และ** ตอนมี
pull request เปิดเข้ามา (เพื่อให้เห็นผลก่อน merge จริง) วิธีเขียน trigger ที่ครอบคลุมทั้งสองกรณี:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
```

**ทำไมต้องมีทั้งสองอย่าง ไม่ใช่แค่อย่างเดียว?**

- `pull_request` เท่านั้น — จะไม่รันตอนมี push ตรงเข้า `main` โดยไม่ผ่าน PR (เช่น admin push ตรง หรือ merge จาก
  branch ที่ไม่ผ่าน PR workflow) ทำให้ `main` อาจมีสถานะที่ไม่ผ่าน CI ได้โดยไม่มีใครรู้
- `push` เท่านั้น — จะไม่มีการรัน check ให้เห็น**ก่อน** merge PR เข้า `main` (รันหลัง merge ไปแล้วถึงจะรู้ว่าพัง)
  ซึ่งเสียจุดประสงค์หลักของ CI ที่ต้องการบล็อกโค้ดเสียก่อนที่มันจะเข้า branch หลัก

การมีทั้งสอง trigger ทำให้: PR ทุกใบเห็นผล CI ก่อน merge (ผ่าน `pull_request`) และถ้ามีทางอื่นที่โค้ดเข้า `main`
ได้โดยไม่ผ่าน PR ก็ยังมี CI คอยตรวจซ้ำอีกชั้น (ผ่าน `push`) — สอง trigger นี้ทับซ้อนกันโดยตั้งใจ เพื่อไม่ให้มีช่อง
โหว่ที่โค้ดหลุดผ่านไปได้โดยไม่ถูกตรวจ

**field อื่น ๆ ที่ควรรู้จักของ `pull_request`:** โดย default `pull_request` trigger จะรันซ้ำทุกครั้งที่มีการ push
commit ใหม่เข้า branch ของ PR นั้น (activity type `opened`, `synchronize`, `reopened` เป็นค่า default) ถ้าต้องการ
จำกัด event ให้ระบุ `types:` เพิ่มได้ เช่น `types: [opened, synchronize]`

### 97.4 Caching Dependency: ปัญหาที่ซับซ้อนกว่าที่คิด และวิธีแก้ที่ถูกต้อง

นี่คือหัวข้อที่สำคัญที่สุดหัวข้อหนึ่งของบทนี้ ตามที่กล่าวไว้ในหัวข้อ 97.1 — compile time ของ Rust ที่ช้าทำให้
caching ไม่ใช่ทางเลือก แต่เป็นสิ่งจำเป็น มาดูกันว่าทำไมมันซับซ้อนกว่าที่คนใหม่ ๆ คิด

**สิ่งที่ต้อง cache มีสองส่วนหลัก:**

1. **`~/.cargo/registry/`** — ไฟล์ source code ของทุก crate ที่ดาวน์โหลดมาจาก crates.io (ไม่ต้องดาวน์โหลดซ้ำ
   ถ้า `Cargo.lock` ไม่เปลี่ยน)
2. **`target/`** — ผลลัพธ์การ compile แต่ละ crate ที่เก็บไว้เป็น intermediate object file (`.rlib`, `.d` ฯลฯ)
   ถ้า cache ส่วนนี้ได้สำเร็จ การ build ครั้งต่อไปจะ**ไม่ต้อง compile dependency ใหม่เลย** ถ้าไม่มีอะไรเปลี่ยน —
   นี่คือส่วนที่ประหยัดเวลาได้มากที่สุด เพราะการ compile ใหม่ (ไม่ใช่แค่ดาวน์โหลด) คือส่วนที่ใช้เวลานานที่สุด

**ทำไม cache `target/` ตรง ๆ ด้วย `actions/cache` แบบทั่วไปถึงมีปัญหา?**

Action ทางการ `actions/cache@v4` เป็น action อเนกประสงค์ที่ cache/restore โฟลเดอร์ไหนก็ได้ตาม key ที่กำหนด
ลองดูตัวอย่างการใช้แบบตรงไปตรงมาก่อน เพื่อเข้าใจปัญหา:

```yaml
      - name: Cache cargo registry and target
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
```

การใช้แบบนี้**ทำงานได้จริง** และดีกว่าไม่มี cache เลยแน่นอน แต่มีข้อจำกัดสามข้อที่ทำให้มันไม่สมบูรณ์แบบสำหรับ
โปรเจกต์ Rust โดยเฉพาะ:

1. **ขนาด cache บวมขึ้นเรื่อย ๆ แบบไม่มีที่สิ้นสุด** — โฟลเดอร์ `target/` เก็บทุก build artifact ของทุกเวอร์ชัน
   ของทุก dependency ที่เคยถูก compile ไว้ ถ้า `Cargo.lock` เปลี่ยนบ่อย (เพิ่ม dependency ใหม่, อัปเดตเวอร์ชัน)
   cache key จะเปลี่ยนตาม ทำให้ต้องสร้าง cache ใหม่ทั้งหมด แต่ cache เก่าก็ไม่ถูกลบทันที (GitHub เก็บ cache สูงสุด
   10 GB ต่อ repository และลบ cache ที่เก่าที่สุดเมื่อเกินขนาด) เกิดเป็นปัญหา "cache thrashing" ที่ cache ที่ควร
   จะช่วยประหยัดเวลา กลับกลายเป็นตัวใช้เวลาอัปโหลด/ดาวน์โหลดจำนวนมากในตัวเอง
2. **incremental compilation artifact ของ Cargo ไม่เหมาะกับการ cache ข้าม CI run** — Cargo เก็บข้อมูล
   incremental compilation (สำหรับ build ซ้ำ ๆ บนเครื่อง developer คนเดียว) ไว้ใน `target/` ด้วย ข้อมูลนี้ผูกกับ
   timestamp ของไฟล์และ path ที่ตายตัว การย้าย cache ข้าม runner (ที่ path อาจไม่ตรงกันเป๊ะ หรือ timestamp ของไฟล์
   หลัง extract cache ต่างจากตอน build จริง) อาจทำให้ Cargo ตัดสินใจ compile ใหม่บางส่วนอยู่ดี หรือในกรณีเลวร้าย
   กว่านั้นคือทำให้ build ผลลัพธ์ไม่ถูกต้อง (แม้จะเกิดยากก็ตาม)
3. **ไฟล์บางส่วนใน `target/` ไม่ควร cache เลย** เช่นไฟล์ lock ของ build process (`target/.cargo-lock`),
   ไฟล์ของ crate ที่เป็นตัวโปรเจกต์เราเอง (ไม่ใช่ dependency) ที่เปลี่ยนบ่อยเกินกว่าจะได้ประโยชน์จากการ cache

**นี่คือเหตุผลที่ทำไม ecosystem Rust จึงมี action เฉพาะทางสำหรับปัญหานี้: [`Swatinem/rust-cache`](https://github.com/Swatinem/rust-cache)**

`Swatinem/rust-cache` เป็น action (ไม่ใช่ทางการของ GitHub แต่เป็น action ที่ชุมชน Rust ยอมรับกว้างขวางและใช้กัน
เป็นมาตรฐานโดยพฤตินัยสำหรับ caching บน GitHub Actions) ที่แก้ปัญหาทั้งสามข้อข้างต้นให้อัตโนมัติ:

- เลือก cache **เฉพาะส่วนที่ปลอดภัยและมีประโยชน์จริง** ของ `~/.cargo` และ `target/` (ไม่ cache ไฟล์ของ crate
  ตัวเราเอง เฉพาะ dependency ที่ compile แล้วเท่านั้น)
- สร้าง cache key ที่ฉลาดกว่าการ hash แค่ `Cargo.lock` ตรง ๆ — คำนวณจากปัจจัยที่เกี่ยวข้องจริง เช่นเวอร์ชันของ
  `rustc`, target platform, และเนื้อหา `Cargo.lock`
- มีกลไก "cache cleanup" อัตโนมัติที่ลบ artifact เก่าที่ไม่เกี่ยวข้องอีกแล้วออกจาก cache ก่อนบันทึก ทำให้ขนาด
  cache ไม่บวมแบบไม่มีที่สิ้นสุดเหมือนวิธีแบบพื้นฐาน

**การใช้งานเรียบง่ายมาก** — แค่เพิ่ม step เดียวหลัง checkout และหลังติดตั้ง Rust toolchain:

```yaml
      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2
```

แค่นี้ก็ครอบคลุม 90% ของกรณีใช้งานทั่วไป ไม่ต้องกำหนด `path`/`key` เองเลย action จะตรวจ `Cargo.lock` ในโปรเจกต์
อัตโนมัติ สำหรับ workspace ที่มีหลาย crate หรือต้องการแยก cache ตาม matrix (เช่นแยก cache ตาม Rust channel หรือ
OS ที่ต่างกัน เพราะ binary ที่ compile บน stable กับ nightly ไม่สามารถใช้ cache ร่วมกันได้) สามารถกำหนด `key`
เพิ่มเป็นตัวแบ่งได้:

```yaml
      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.rust }}-${{ matrix.os }}
```

**เชื่อมโยงกับแนวคิด Docker layer caching จาก Part 96:** ถ้าคุณจำ Dockerfile แบบ multi-stage build จาก Part 96
ได้ — เทคนิคที่แนะนำไว้คือ `COPY Cargo.toml Cargo.lock ./` แล้ว `cargo build --release` เป็น layer แยกออกมา
**ก่อน** ที่จะ `COPY src/ ./src/` แล้ว build จริงอีกที เหตุผลคือ Docker cache แต่ละ layer ตาม "input ที่ไม่เปลี่ยน"
— ถ้า `Cargo.toml`/`Cargo.lock` ไม่เปลี่ยน Docker จะ**ใช้ layer เดิมที่ compile dependency ไว้แล้วซ้ำ** โดยไม่ต้อง
compile ใหม่ แม้ว่า source code ของแอปเราเองจะเปลี่ยนไปกี่ครั้งก็ตาม

นี่คือ**หลักการเดียวกันเป๊ะ ๆ** กับที่ `Swatinem/rust-cache` ทำบน GitHub Actions — แยก "สิ่งที่เปลี่ยนไม่บ่อย"
(dependency ที่ compile แล้ว) ออกจาก "สิ่งที่เปลี่ยนบ่อย" (source code ของเราเอง) แล้ว cache เฉพาะส่วนแรกอย่าง
ฉลาด ทั้งสองกรณีคือการประยุกต์ใช้แนวคิดเดียวกัน: **การ compile Rust dependency มีต้นทุนสูง ให้ทำเพียงครั้งเดียว
แล้ว reuse ผลลัพธ์นั้นให้ได้มากที่สุด ไม่ว่าจะอยู่ใน layer ของ Docker image หรือใน cache ของ CI runner**

### 97.5 CI Workflow ฉบับสมบูรณ์: fmt, clippy, build, test

ตอนนี้เรามีองค์ประกอบครบแล้ว: trigger, toolchain, caching มาประกอบเป็น CI workflow ที่ใช้งานได้จริงในโปรเจกต์
production กัน โดยใช้ [`dtolnay/rust-toolchain`](https://github.com/dtolnay/rust-toolchain) ในการติดตั้ง Rust
toolchain (action ที่ยอมรับกว้างขวางในชุมชน สำหรับติดตั้ง toolchain เร็วกว่าการดาวน์โหลด rustup ใหม่ทุกครั้ง
เพราะ runner ของ GitHub มักมี Rust ติดตั้งไว้บางเวอร์ชันอยู่แล้ว action นี้แค่สลับ/ติดตั้ง component ที่ขาดเพิ่ม)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always

# ยกเลิก run เก่าของ workflow เดียวกันบน ref เดียวกันโดยอัตโนมัติ เมื่อมี push ใหม่เข้ามาทับ
# (ประหยัดเวลา runner เมื่อมีคน push แก้ typo ซ้อน ๆ กันหลายครั้งในเวลาสั้น ๆ)
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Format and Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Run clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

  test:
    name: Build and Test
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2

      - name: Build
        run: cargo build --all-targets --verbose

      - name: Run tests
        run: cargo test --all-features --verbose
```

**อธิบายจุดที่เพิ่มเข้ามาใหม่ทีละส่วน:**

- **`env: CARGO_TERM_COLOR: always`** — บอกให้ Cargo แสดง output แบบมีสี แม้ terminal ของ CI จะไม่ใช่ TTY
  จริง (ปกติ Cargo จะปิดสีอัตโนมัติเมื่อ output ไม่ได้ไปที่ terminal คนใช้จริง) ทำให้อ่าน log บนหน้าเว็บ GitHub
  Actions ได้ง่ายขึ้นมาก (แยก error สีแดงจาก warning สีเหลืองได้ทันทีด้วยตา)
- **`concurrency:`** — กลุ่มบล็อกนี้บอกว่า run ใหม่ของ workflow เดียวกันบน `ref` เดียวกัน (เช่น branch ของ PR
  เดียวกัน) ให้**ยกเลิก run ก่อนหน้าที่ยังไม่จบ** (`cancel-in-progress: true`) ทันทีที่มี push ใหม่เข้ามา — มี
  ประโยชน์มากในทีมที่ push บ่อย เพราะไม่มีประโยชน์อะไรที่จะรอผลของ commit เก่าที่ถูก commit ใหม่ทับไปแล้ว
  ประหยัดทั้งเวลาและ CI minute ที่ใช้ (GitHub Actions มี free tier จำกัดจำนวนนาทีต่อเดือนสำหรับ private
  repository)
- **แยกเป็นสอง job (`lint` และ `test`)** — แม้ทั้งสอง job ต้อง checkout และติดตั้ง toolchain ซ้ำ (ดูเหมือน
  ทำงานซ้ำ) แต่**รันพร้อมกัน (parallel)** โดย default เพราะไม่มี `needs:` ระบุไว้ ทำให้เวลารวมของทั้ง pipeline
  เท่ากับเวลาของ job ที่ช้าที่สุด ไม่ใช่ผลรวมของทั้งสอง job — ถ้ารวมทุกอย่างไว้ใน job เดียว ทุก step ต้องรันตาม
  ลำดับ (sequential) เสียเวลารวมมากกว่า การแยก job ตามความรับผิดชอบ (lint ไม่เกี่ยวกับ test โดยตรง) แล้วให้รัน
  พร้อมกันจึงเป็นแนวทางที่ดีกว่าเสมอเมื่อ job นั้นไม่ได้ต้องพึ่งผลลัพธ์ของกันและกัน
- **`--all-targets`** ใน `cargo build`/`cargo clippy` — บอกให้ build/ตรวจสอบทุก target ไม่ใช่แค่ `src/main.rs`/
  `src/lib.rs` แต่รวม `tests/`, `examples/`, `benches/` ด้วย (ป้องกันกรณีที่โค้ดใน `tests/` compile ไม่ผ่านแต่
  โค้ดหลักผ่าน ซึ่งเป็นปัญหาที่พบได้จริงถ้าลืม `--all-targets`)
- **`--all-features`** ใน `cargo clippy`/`cargo test` — ถ้าโปรเจกต์มี Cargo feature flag (จาก Part 17) ควร
  ตรวจสอบว่าทุก feature combination compile ผ่านด้วย ไม่ใช่แค่ default feature (ในโปรเจกต์ที่ feature flag
  ซับซ้อนมาก อาจต้องมี matrix แยกทดสอบแต่ละ combination — แต่สำหรับโปรเจกต์ทั่วไป `--all-features` ครอบคลุมพอ)

**ทดสอบยืนยันจริงว่าคำสั่งเหล่านี้ทำงานถูกต้อง:** เราสร้างโปรเจกต์ Rust scratch ขนาดเล็กที่มีโค้ดจัดรูปแบบถูกต้อง
และผ่าน clippy สะอาด แล้วรันคำสั่งชุดเดียวกันกับที่ workflow ข้างต้นเรียก ได้ผลลัพธ์จริงดังนี้:

```
$ cargo fmt --check
$ echo "exit: $?"
exit: 0

$ cargo clippy --all-targets -- -D warnings
    Checking ci_demo v0.1.0 (/tmp/.../ci_demo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.18s
$ echo "exit: $?"
exit: 0

$ cargo test
   Compiling ci_demo v0.1.0 (/tmp/.../ci_demo)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.20s
     Running unittests src/lib.rs (target/debug/deps/ci_demo-eed3d499ec41ce2b)

running 4 tests
test tests::cart_total_sums_all_prices ... ok
test tests::low_stock_detection ... ok
test tests::shipping_fee_over_20kg_uses_bulk_rate ... ok
test tests::shipping_fee_under_20kg_uses_standard_rate ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

ทุกคำสั่งคืน exit code 0 เมื่อโค้ดสะอาด — ตรงตามพฤติกรรมที่ CI คาดหวัง (job ผ่าน = merge ได้) ในหัวข้อ กับดักที่
พบบ่อย ท้ายบทเราจะแสดงสิ่งที่เกิดขึ้นเมื่อโค้ด**ไม่**สะอาด เพื่อให้เห็นทั้งสองด้าน

### 97.6 Matrix Builds: ทดสอบข้าม Rust Channel และข้าม OS

**`strategy.matrix`** เป็นฟีเจอร์ของ GitHub Actions ที่สร้าง job หลายตัวจาก template เดียว โดยรันทุก
combination ของค่าที่กำหนดไว้ — ใช้บ่อยที่สุดสำหรับสองมิติ: **Rust release channel** (stable/beta/nightly จาก
แนวคิด release train ที่ Rust ใช้) และ **operating system** (Linux/macOS/Windows)

**ตัวอย่าง matrix ข้าม Rust channel และ OS:**

```yaml
# .github/workflows/matrix-ci.yml
name: Matrix CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test (${{ matrix.rust }} on ${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        rust: [stable, beta, nightly]
    continue-on-error: ${{ matrix.rust == 'nightly' }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain (${{ matrix.rust }})
        uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.rust }}

      - name: Build
        run: cargo build --verbose

      - name: Run tests
        run: cargo test --verbose
```

**อธิบายรายละเอียด:**

- **`matrix: os: [...] rust: [...]`** — GitHub Actions จะสร้าง job แยกสำหรับทุก combination โดยอัตโนมัติ ใน
  ตัวอย่างนี้คือ 3 OS × 3 channel = **9 job** ที่รันพร้อมกัน (ภายในขีดจำกัดจำนวน job พร้อมกันที่ plan ของ
  repository อนุญาต)
- **`fail-fast: false`** — โดย default ถ้า job ใดใน matrix ล้มเหลว GitHub Actions จะ**ยกเลิก job ที่เหลือทั้งหมด
  ทันที** (`fail-fast: true` เป็นค่า default) ซึ่งมีประโยชน์เมื่อต้องการรู้ผลเร็วและประหยัดเวลา runner แต่มีข้อเสีย
  สำหรับ matrix testing: ถ้าต้องการรู้**ผลของทุก combination** (เช่น อยากรู้ว่าพังบน Windows หรือ macOS ด้วยไหม
  ไม่ใช่แค่ตัวแรกที่เจอ) ต้องตั้ง `fail-fast: false` เพื่อให้ job อื่นรันจนจบแม้บาง job จะล้มเหลวไปแล้ว
- **`continue-on-error: ${{ matrix.rust == 'nightly' }}`** — เทคนิคสำคัญ: nightly toolchain ของ Rust ไม่มี
  การันตีความเสถียร (อาจมี regression หรือ feature ที่ถูกถอนออกกะทันหัน) การให้ job ที่รันบน nightly ล้มเหลวได้
  โดยไม่ทำให้ **check โดยรวมของ PR แสดงเป็นสีแดง** เป็นแนวทางที่สมดุล — เราต้องการ*รู้*ว่า nightly มีปัญหาไหม
  (เพื่อเตรียมตัวล่วงหน้าก่อนที่ปัญหานั้นจะเข้ามาสู่ stable channel) แต่ไม่ต้องการให้มันบล็อกการ merge PR ที่โค้ด
  ถูกต้องบน stable/beta อยู่แล้ว — `continue-on-error: true` ทำให้ job แสดงผลเป็น "⚠️" (เตือน) แทน "❌" (ล้มเหลว
  แบบบล็อก) เมื่อพัง
- **`dtolnay/rust-toolchain@master`** (ไม่ใช่ `@stable`) — สังเกตว่าเราเปลี่ยนจาก `@stable` (หัวข้อก่อนหน้า) เป็น
  `@master` ที่นี่ เพราะ `dtolnay/rust-toolchain` ใช้ branch ของ repository เป็นตัวเลือก toolchain: branch
  `stable`/`beta`/`nightly` คือ shortcut ที่ตายตัว ในขณะที่ branch `master` รับ input `toolchain:` เพื่อระบุค่า
  แบบ dynamic ได้ (จำเป็นสำหรับ matrix ที่ค่า `matrix.rust` เปลี่ยนไปตาม combination)

**เมื่อไหร่ที่ matrix build แบบนี้คุ้มค่า และเมื่อไหร่ที่ไม่จำเป็น?** นี่คือคำถามการออกแบบที่สำคัญ ไม่ใช่ว่าทุก
โปรเจกต์ต้องมี matrix ครบ 9 combination เสมอไป:

- **Library ที่ publish บน crates.io** (เช่น crate ยูทิลิตี้ที่คนอื่นจะเอาไป `cargo add` ในโปรเจกต์ของตัวเอง) —
  **ควรมี matrix ครบทั้ง OS และ channel** เพราะไม่รู้ว่าผู้ใช้ crate จะรันบนสภาพแวดล้อมไหน การพังบน Windows หรือ
  บน nightly (ที่บางคนยังใช้เพื่อทดลอง feature ใหม่) คือความเสี่ยงที่ทีม maintainer ต้องรู้ก่อนที่ผู้ใช้จะมาแจ้ง
  bug — ความน่าเชื่อถือข้าม platform คือคุณค่าหลักของการเป็น library ที่ดี
- **Application ภายในที่ deploy ไปยัง environment ที่รู้จักแน่นอนเพียงหนึ่งเดียว** (เช่น `library_api` จาก
  Part 92 ที่ deploy เป็น Docker container บน Linux server เท่านั้น ตาม Part 96) — **matrix แบบครบทุก OS ไม่คุ้ม
  ค่า** เพราะไม่มีใครรัน production บน Windows หรือ macOS ทีมประหยัด CI minute และความซับซ้อนได้มากโดยทดสอบแค่
  `ubuntu-latest` (ตรงกับ base image ของ container ที่ใช้จริง) และ `stable` channel เดียว (เพราะ production
  binary ก็ build ด้วย stable เสมอ ไม่มีเหตุผลที่จะ deploy binary ที่ build ด้วย nightly)

หลักการตัดสินใจง่าย ๆ คือ: **matrix ควรสะท้อนพื้นผิว (surface) ของสภาพแวดล้อมที่โค้ดจะถูกใช้งานจริง** ถ้าพื้นผิว
นั้นแคบ (deploy ที่เดียว) การทดสอบข้าม matrix กว้างเป็นการเสียเวลา CI โดยไม่ได้ข้อมูลที่มีประโยชน์เพิ่ม ถ้าพื้นผิว
นั้นกว้าง (library สาธารณะ) การไม่ทดสอบข้าม matrix คือการเสี่ยงที่ไม่จำเป็น

### 97.7 ทดสอบกับ PostgreSQL จริง: GitHub Actions `services:`

Part 92 สร้าง `library_api` ที่ใช้ SQLx คุยกับ PostgreSQL โดยตรง และ Part 95 สอนการเขียน integration test ด้วย
`#[sqlx::test]` ที่ต้องมีฐานข้อมูล PostgreSQL รันอยู่จริงตอน test (ไม่ใช่ mock) — คำถามคือ: **CI runner ที่เป็น
เครื่องเปล่า ไม่มี PostgreSQL ติดตั้งไว้ จะรัน test เหล่านี้ได้อย่างไร?**

คำตอบคือฟีเจอร์ **`services:`** ของ GitHub Actions — บอกให้ runner สตาร์ท Docker container เพิ่มเติม (ทำงาน
"ข้าง ๆ" job หลัก) ก่อนที่ step ใด ๆ ของ job จะเริ่มรัน แล้ว container นั้นจะพร้อมให้เชื่อมต่อผ่าน `localhost`
ตลอดเวลาที่ job ทำงาน (และถูกทำลายทิ้งอัตโนมัติเมื่อ job จบ — ไม่ต้องเขียน cleanup เอง)

```yaml
# .github/workflows/backend-ci.yml
name: Backend CI (with PostgreSQL)

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  DATABASE_URL: postgres://postgres:postgres@localhost:5432/library_api_test

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: library_api_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U postgres"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2

      - name: Install sqlx-cli
        run: cargo install sqlx-cli --no-default-features --features rustls,postgres --locked

      - name: Run database migrations
        run: sqlx migrate run

      - name: Build
        run: cargo build --all-targets --verbose

      - name: Run tests
        run: cargo test --all-features --verbose
```

**อธิบายรายละเอียดของ `services:` ทีละส่วน:**

- **`postgres: image: postgres:16`** — ระบุ Docker image ที่จะใช้เป็น service ในตัวอย่างนี้คือ official
  PostgreSQL image เวอร์ชัน 16 (ควรตรงกับเวอร์ชันที่ใช้จริงตอน development และ production ตาม Part 70 — ความไม่
  ตรงกันของเวอร์ชัน PostgreSQL ระหว่าง dev/CI/production อาจทำให้พฤติกรรมบางอย่างต่างกันเล็กน้อย เช่น SQL feature
  ใหม่บางตัว)
- **`env:` ภายใต้ `postgres:`** — environment variable ที่ส่งให้ container ของ PostgreSQL ตอนสตาร์ท (ตาม official
  image ของ PostgreSQL, `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` คือชื่อ env var มาตรฐานที่ image นี้
  อ่านเพื่อสร้าง user/database เริ่มต้นให้อัตโนมัติ)
- **`ports: - 5432:5432`** — map port 5432 ของ container ออกมาที่ port 5432 ของ runner (`localhost`) ทำให้โค้ด
  Rust ที่รันในขั้นตอนถัดไปเชื่อมต่อผ่าน `localhost:5432` ได้เหมือนกับกำลังรัน PostgreSQL บนเครื่องตัวเอง
- **`options: --health-cmd=...`** — นี่คือจุดสำคัญมากที่พลาดกันบ่อย: container ของ PostgreSQL ใช้เวลาสองสาม
  วินาทีในการเริ่มต้นระบบข้างในให้พร้อมรับ connection จริง (ไม่ใช่แค่ process เริ่มรันแล้วพร้อมทันที) ถ้า step
  ถัดไป (`sqlx migrate run`) รันเร็วเกินไปก่อนที่ PostgreSQL จะพร้อม จะเจอ connection refused — GitHub Actions
  แก้ปัญหานี้ให้อัตโนมัติด้วย **health check**: มันจะรันคำสั่ง `pg_isready -U postgres` ซ้ำ ๆ ทุก 10 วินาที
  (`--health-interval=10s`) สูงสุด 5 ครั้ง (`--health-retries=5`) และ**รอจนกว่า health check จะผ่าน** ก่อนที่จะ
  ให้ step แรกของ job เริ่มรัน — ทำให้เราไม่ต้องเขียน retry loop เองในโค้ด CI
- **`DATABASE_URL` ระดับ `env:` ของ workflow** — ตั้งค่านี้ให้ตรงกับ credential ที่ตั้งไว้ใน service container
  พอดี (`postgres:postgres@localhost:5432/library_api_test`) ตัวแปรนี้จะถูกอ่านทั้งโดย `sqlx-cli` (คำสั่ง
  `sqlx migrate run`) และโดย `sqlx::query!`/`query_as!` macro ตอน compile time (ตามที่ Part 70-71 สอนไว้)

**ปัญหาเฉพาะของ SQLx ที่ต้องเข้าใจให้ลึก: `sqlx::query!` ต้องต่อฐานข้อมูลจริงตอน compile time**

นี่คือจุดที่ทำให้ CI ของโปรเจกต์ที่ใช้ SQLx **ต่างจาก** โปรเจกต์ Rust ทั่วไปอย่างมีนัยสำคัญ จาก Part 70 เราเรียน
แล้วว่า `sqlx::query!`/`query_as!` เป็น macro ที่ตรวจสอบ SQL กับ schema จริงของฐานข้อมูล**ตอน compile** (ไม่ใช่
ตอน runtime) — นี่คือฟีเจอร์ที่ดีมาก (จับ SQL ผิดพลาดได้ก่อน deploy) แต่มันหมายความว่า **`cargo build` ของ
โปรเจกต์นี้ต้องมี฀ฐานข้อมูลที่เชื่อมต่อได้จริงอยู่ ณ เวลาที่ compile** ซึ่งใน workflow ข้างต้นเราแก้ด้วยการรัน
`sqlx migrate run` (สร้างตารางให้ตรง schema จริง) **ก่อน** `cargo build` เสมอ — ถ้าสลับลำดับผิด (build ก่อน
migrate) จะเจอ compile error ทันที

**ทางเลือกที่สอง — SQLx offline mode ด้วย query cache ที่ commit เข้า repository:**

SQLx มีโหมดที่ไม่ต้องพึ่งฐานข้อมูลจริงตอน compile เลย เรียกว่า **offline mode** โดยใช้คำสั่ง `cargo sqlx prepare`
(รันครั้งเดียวตอน development โดยมีฐานข้อมูลจริงอยู่) เพื่อสร้างโฟลเดอร์ `.sqlx/` ที่เก็บผลลัพธ์การตรวจสอบ query
ทั้งหมดไว้เป็นไฟล์ JSON แล้ว commit โฟลเดอร์นี้เข้า git ปกติ:

```bash
# รันบนเครื่อง development ที่มี DATABASE_URL ชี้ไปยังฐานข้อมูลจริง (ครั้งเดียวหลังแก้ query)
cargo sqlx prepare
git add .sqlx/
git commit -m "Update SQLx offline query cache"
```

เมื่อมี `.sqlx/` อยู่ใน repository แล้ว ตั้ง environment variable `SQLX_OFFLINE=true` ตอน build บน CI:

```yaml
      - name: Build (offline — ไม่ต้องมี DATABASE_URL ระหว่าง build)
        env:
          SQLX_OFFLINE: "true"
        run: cargo build --all-targets --verbose
```

macro จะอ่านผลลัพธ์ที่ cache ไว้ใน `.sqlx/` แทนการต่อฐานข้อมูลจริง ทำให้ **`cargo build` ไม่ต้องมี service
`postgres:` เลย** — แต่ระวังจุดสำคัญ: **`cargo test` ที่รัน integration test แบบ `#[sqlx::test]` ยังต้องมี
PostgreSQL จริงอยู่เสมอ** เพราะ offline mode ช่วยแค่ตอน macro ตรวจสอบ SQL syntax/schema ตอน compile เท่านั้น
มันไม่ได้ช่วยตอน**รัน**ควรี่จริงในระหว่าง test — ดังนั้น service container `postgres:` ยังต้องอยู่ใน workflow
เสมอถ้ามี integration test ที่ยิง query จริง สิ่งที่ offline mode ช่วยประหยัดคือ**ขั้นตอน build เท่านั้น** ทำให้
job ที่ต้องการแค่ `cargo build --check` เร็ว ๆ (เช่น job `lint` ในหัวข้อก่อนหน้า) ไม่ต้องแบก service container
ที่หนักเพิ่มโดยไม่จำเป็น — เป็นการ optimize ที่คุ้มค่าเมื่อ pipeline มีหลาย job ที่ไม่ได้ต้องการรัน test จริงทุกตัว

ตารางสรุปสองแนวทาง:

| แนวทาง | Build ต้องมี DB จริงไหม | Test ต้องมี DB จริงไหม | เหมาะกับ |
|---|---|---|---|
| ต่อ DB จริงตอน build (`sqlx migrate run` ก่อน `cargo build`) | ต้องมี | ต้องมี | ทีมเล็ก, pipeline เดียว, ไม่ต้องดูแล `.sqlx/` เพิ่ม |
| Offline mode (`.sqlx/` + `SQLX_OFFLINE=true`) | ไม่ต้อง | ต้องมี | ทีมที่มีหลาย job/pipeline ที่ต้องการ build เร็วโดยไม่แบก service container ทุก job |

### 97.8 Code Coverage: `cargo-llvm-cov`

**Code coverage** คือเปอร์เซ็นต์ของโค้ดที่ถูกรันจริงระหว่าง test suite — ใช้เป็นสัญญาณ (ไม่ใช่การันตี) ว่าส่วน
ไหนของโค้ดยังไม่มี test ครอบคลุมเลย เครื่องมือที่นิยมใช้กับ Rust มีสองตัวหลักที่ต้องรู้จัก:

- **`cargo-tarpaulin`** — เครื่องมือ coverage ตัวแรก ๆ ที่ได้รับความนิยมในระบบ Rust รองรับเฉพาะ Linux เป็นหลัก
  (แม้เริ่มรองรับแพลตฟอร์มอื่นในเวอร์ชันหลัง ๆ แต่ก็ยังมีข้อจำกัดมากกว่า) ใช้เทคนิค ptrace หรือ instrumentation
  แบบ source-based
- **`cargo-llvm-cov`** — เครื่องมือที่ใหม่กว่า ใช้ฟีเจอร์ **source-based code coverage** ที่เป็น native feature
  ของ LLVM (`-C instrument-coverage`) โดยตรง — เนื่องจาก Rust compiler (`rustc`) ใช้ LLVM เป็น backend อยู่แล้ว
  การใช้ฟีเจอร์ coverage ที่ built-in อยู่ใน LLVM ทำให้ `cargo-llvm-cov` มีความแม่นยำสูงและรองรับทุกแพลตฟอร์ม
  (Linux/macOS/Windows) ได้ดีกว่า **บทนี้เลือกใช้ `cargo-llvm-cov` เป็นหลัก** เพราะเป็นแนวทางที่ทันสมัยกว่าและ
  ทีม Rust official (`rustc`) ดูแล mechanism เบื้องหลังโดยตรง ทำให้มีความเสถียรและ compatibility กับ toolchain
  รุ่นใหม่ ๆ ดีกว่าในระยะยาว

**ติดตั้งและรันบนเครื่อง:**

```bash
rustup component add llvm-tools-preview
cargo install cargo-llvm-cov --locked
```

เราติดตั้งและรันจริงกับโปรเจกต์ scratch ที่มี unit test 4 ตัว (จากหัวข้อ 97.5) ได้ผลลัพธ์จริงดังนี้:

```
$ cargo llvm-cov --summary-only
   Compiling ci_demo v0.1.0 (/tmp/.../ci_demo)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.31s
     Running unittests src/lib.rs (target/llvm-cov-target/debug/deps/ci_demo-eed3d499ec41ce2b)

running 4 tests
test tests::shipping_fee_over_20kg_uses_bulk_rate ... ok
test tests::low_stock_detection ... ok
test tests::cart_total_sums_all_prices ... ok
test tests::shipping_fee_under_20kg_uses_standard_rate ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

Filename             Regions    Missed Regions     Cover   Functions  Missed Functions  Executed       Lines      Missed Lines     Cover
------------------------------------------------------------------------------------------------------------------------------------------
src/lib.rs                32                 0   100.00%           7                 0   100.00%          24                 0   100.00%
------------------------------------------------------------------------------------------------------------------------------------------
TOTAL                     32                 0   100.00%           7                 0   100.00%          24                 0   100.00%
```

โปรเจกต์ scratch นี้มี test ครอบคลุมทุกฟังก์ชัน (100%) เพราะตั้งใจเขียนให้ครบเพื่อสาธิต — ในโปรเจกต์จริงตัวเลขนี้
มักไม่ถึง 100% และไม่จำเป็นต้องถึง 100% เสมอไป (100% coverage ไม่ได้แปลว่าไม่มีบั๊ก มันแค่แปลว่าทุกบรรทัดถูกรัน
ผ่านสักครั้ง ไม่ได้การันตีว่า assertion ที่ตรวจครอบคลุมพอ) ทีมส่วนใหญ่ตั้งเกณฑ์ที่สมเหตุสมผล เช่น "ห้ามลดลงจาก
baseline" หรือ "ต้องเกิน 70-80% สำหรับโค้ด business logic หลัก" มากกว่าจะไล่ให้ถึง 100%

**Format ที่ต้องใช้ตอนส่งเข้า CI: LCOV** — สำหรับส่งผลลัพธ์ไปยังบริการภายนอกอย่าง Codecov ต้อง export เป็น
format `lcov` ซึ่งเราทดสอบจริงแล้วว่าทำงานถูกต้อง:

```
$ cargo llvm-cov --lcov --output-path lcov.info
    Finished report saved to lcov.info

$ head -5 lcov.info
SF:/tmp/.../ci_demo/src/lib.rs
FN:38,_RNvNtCs1U0UAmcI1vd_7ci_demo5testss_19low_stock_detection
FN:44,_RNvNtCs1U0UAmcI1vd_7ci_demo5testss_26cart_total_sums_all_prices
FN:33,_RNvNtCs1U0UAmcI1vd_7ci_demo5testss_37shipping_fee_over_20kg_uses_bulk_rate
FN:28,_RNvNtCs1U0UAmcI1vd_7ci_demo5testss_42shipping_fee_under_20kg_uses_standard_rate
```

**นำมาผนวกเป็น job ใน workflow** สำหรับโปรเจกต์ `library_api` ที่ต้อง PostgreSQL ด้วย (รวม `services:` จากหัวข้อ
ก่อนหน้าเข้าไปด้วย):

```yaml
  coverage:
    name: Code Coverage
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: library_api_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U postgres"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/library_api_test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: llvm-tools-preview

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2

      # ติดตั้ง cargo-llvm-cov จาก prebuilt binary แทนการ compile จาก source
      # (compile เองใช้เวลาเป็นนาที ในขณะที่ดาวน์โหลด binary สำเร็จรูปใช้เวลาไม่กี่วินาที
      # นี่คือเทคนิค caching อีกรูปแบบหนึ่ง: หลีกเลี่ยงการ compile ที่ไม่จำเป็นตั้งแต่แรก)
      - name: Install cargo-llvm-cov
        uses: taiki-e/install-action@cargo-llvm-cov

      - name: Install sqlx-cli
        run: cargo install sqlx-cli --no-default-features --features rustls,postgres --locked

      - name: Run database migrations
        run: sqlx migrate run

      - name: Generate coverage report
        run: cargo llvm-cov --all-features --lcov --output-path lcov.info

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v5
        with:
          files: lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true
```

**สิ่งที่ตรวจสอบได้จริง vs. สิ่งที่ต้องอธิบายเป็น documented behavior:** ทุก step ก่อน `Upload coverage to
Codecov` เราตรวจสอบได้จริงด้วยการรันคำสั่งบนเครื่อง (ผลลัพธ์ด้านบนคือของจริง) แต่ **step สุดท้ายที่ upload ไปยัง
Codecov ต้องมี repository ที่เชื่อมกับ Codecov จริงและมี `secrets.CODECOV_TOKEN` ที่ตั้งไว้ใน GitHub repository
settings** — สิ่งนี้อยู่นอกเหนือสิ่งที่ sandbox การเขียนบทนี้ตรวจสอบได้ พฤติกรรมที่คาดหวัง (ตามเอกสารทางการของ
`codecov/codecov-action`) คือ: action จะอ่านไฟล์ `lcov.info`, ส่งขึ้น Codecov ผ่าน API, Codecov จะประมวลผลและ
แสดง coverage percentage พร้อม diff ของ PR นั้นเทียบกับ branch หลัก, และโพสต์ comment สรุปกลับเข้า pull request
โดยอัตโนมัติ (ฟีเจอร์นี้ต้องเปิดใช้งานใน Codecov dashboard ของ repository ก่อน) — สำหรับโปรเจกต์ private
repository ต้องสร้าง token จาก Codecov แล้วเก็บไว้ใน **Settings → Secrets and variables → Actions** ของ GitHub
repository ในชื่อ `CODECOV_TOKEN` สำหรับ public repository จำนวนมาก Codecov อนุญาตให้ไม่ต้องใช้ token เลยก็ได้
(tokenless upload) แต่มีข้อจำกัดเรื่อง rate limit ที่เข้มงวดกว่า

### 97.9 Build และ Push Docker Image ขึ้น GitHub Container Registry

Part 96 สอนการเขียน Dockerfile แบบ multi-stage build สำหรับ `library_api` — ตอนนี้เราจะนำ image นั้นมา build
และ push ขึ้น registry อัตโนมัติผ่าน CI ทุกครั้งที่โค้ดถูก merge เข้า `main`

**GitHub Container Registry (`ghcr.io`)** คือ registry สำหรับเก็บ Docker image ที่ผนวกเข้ากับ GitHub โดยตรง —
เหตุผลที่บทนี้เลือกใช้ `ghcr.io` เป็นตัวอย่างหลัก (แทน Docker Hub หรือ registry อื่น) คือมันไม่ต้องสร้าง account
หรือตั้งค่าอะไรเพิ่มนอกเหนือจาก GitHub repository ที่มีอยู่แล้ว — authentication ทำผ่าน `secrets.GITHUB_TOKEN` ที่
GitHub Actions **สร้างให้อัตโนมัติทุก run** โดยไม่ต้องไปสร้าง credential เพิ่มเติมที่ไหนเลย (ต่างจาก Docker Hub
ที่ต้องสร้าง account, สร้าง access token, แล้วเอามาเก็บเป็น secret เอง)

```yaml
# .github/workflows/docker.yml (ต่อจาก job อื่นในไฟล์เดียวกัน หรือแยกไฟล์ก็ได้)
  docker:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: [lint, test]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract image metadata (tags/labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,format=long
            type=raw,value=latest

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**อธิบายทีละส่วนที่สำคัญ:**

- **`needs: [lint, test]`** — job นี้จะรอให้ job `lint` และ `test` (จากหัวข้อ 97.5) ผ่านก่อนเท่านั้นถึงจะเริ่ม
  รัน นี่คือหลักการสำคัญของ CD ที่ดี: **อย่า build/push image จากโค้ดที่ยังไม่ผ่านการตรวจสอบ** ถ้า `lint` หรือ
  `test` ล้มเหลว job `docker` จะไม่รันเลย (ถูก skip โดยอัตโนมัติ)
- **`if: github.ref == 'refs/heads/main' && github.event_name == 'push'`** — เงื่อนไขสำคัญที่ป้องกันไม่ให้
  image ถูก build/push จาก pull request (ที่อาจมาจาก fork หรือโค้ดที่ยังไม่ผ่านการ review) — เฉพาะการ push
  (หรือ merge) เข้า `main` โดยตรงเท่านั้นที่จะ trigger การ push image จริง `github.ref` และ `github.event_name`
  คือ **context ที่มีให้ใช้ในทุก workflow** (`github.` เป็น namespace ของข้อมูลเกี่ยวกับ event/repository
  ปัจจุบันที่ GitHub เติมให้อัตโนมัติ)
- **`permissions: packages: write`** — โดย default `secrets.GITHUB_TOKEN` มีสิทธิ์อ่านอย่างเดียว (read-only)
  สำหรับความปลอดภัย ต้อง**ประกาศสิทธิ์ที่ต้องการเพิ่มอย่างชัดเจน**ใน job ที่ต้องการมัน — `packages: write`
  จำเป็นเพื่อให้ token นี้มีสิทธิ์ push image ขึ้น `ghcr.io` ได้ (หลักการ least privilege: job อื่นที่ไม่เกี่ยว
  กับ Docker ไม่ควรมีสิทธิ์นี้เลย)
- **`docker/setup-buildx-action`** — ติดตั้ง BuildKit (build engine รุ่นใหม่ของ Docker ที่รองรับ cache แบบ
  `type=gha` และการ build แบบ multi-platform)
- **`docker/metadata-action`** — สร้าง tag และ label ของ image อัตโนมัติตาม pattern ที่กำหนด ในตัวอย่างนี้สร้าง
  สอง tag: `sha-<commit-sha-เต็ม>` (สำหรับ traceability — รู้แน่ชัดว่า image นี้ build จาก commit ไหน) และ
  `latest` (สำหรับใช้งานทั่วไปที่ต้องการ image ล่าสุดเสมอ)
- **`cache-from: type=gha` / `cache-to: type=gha,mode=max`** — นี่คือการนำแนวคิด **GitHub Actions cache**
  (เดียวกับที่ `Swatinem/rust-cache` ใช้เก็บ Cargo dependency) มาใช้กับ **Docker layer cache** โดยตรง ทำให้
  layer ของ Dockerfile ที่ไม่เปลี่ยน (เช่น layer ที่ compile dependency ตามที่ Part 96 อธิบายไว้) ถูก cache ไว้
  ข้าม CI run ได้ ไม่ต้อง build ใหม่ทุกครั้งที่ push — เป็นการผนวกสองแนวคิดที่เราพูดถึงตลอดบทนี้ (Rust dependency
  caching และ Docker layer caching) เข้าด้วยกันในจุดเดียว
- **`docker/build-push-action`** — step หลักที่สั่ง build แล้ว push ในคำสั่งเดียว `context: .` หมายถึง build
  context คือ root ของ repository (ที่มี `Dockerfile` ตาม Part 96) `push: true` สั่งให้ push ขึ้น registry
  จริงหลัง build สำเร็จ (ถ้าตั้งเป็น `false` จะ build ไว้ในเครื่อง runner เฉย ๆ ไม่ push — มีประโยชน์สำหรับ PR
  ที่อยากรู้แค่ว่า Dockerfile build ผ่านไหม โดยไม่ต้อง push อะไรเลย)

**ความซื่อตรงเรื่องการตรวจสอบ:** เราตรวจสอบ syntax ของ YAML ข้างต้นด้วย YAML parser และตรวจสอบ input/output ของ
แต่ละ action เทียบกับเอกสารทางการของ `docker/build-push-action`, `docker/login-action`, `docker/metadata-action`
อย่างละเอียด แต่**ไม่มีทางรันจริงและยืนยันได้ว่า image ถูก push ขึ้น `ghcr.io` สำเร็จ** ในสภาพแวดล้อมที่เขียนบทนี้
เพราะต้องมี GitHub repository จริงที่เปิดใช้ GitHub Container Registry, มี `GITHUB_TOKEN` ที่ใช้งานได้จริงตอนรัน
workflow บน GitHub Actions infrastructure จริง — พฤติกรรมที่อธิบายไว้ข้างต้นคือพฤติกรรมที่บันทึกไว้ในเอกสาร
ทางการของแต่ละ action ไม่ใช่ผลจากการรันจริงในสภาพแวดล้อมนี้

### 97.10 Release Automation: Multi-Platform Binary และ GitHub Release

ขั้นถัดไปของ CD คือการสร้าง **release** อัตโนมัติเมื่อมีการ tag เวอร์ชันใหม่ (เช่น `v1.2.0`) — workflow นี้จะ
build binary สำหรับหลายแพลตฟอร์ม แล้วแนบเข้า GitHub Release โดยอัตโนมัติ ให้ผู้ใช้ดาวน์โหลด binary สำเร็จรูปได้
โดยไม่ต้อง compile เอง

**เชื่อมโยงกับ musl static linking จาก Part 96:** สำหรับ binary บน Linux เราจะ build เป็น target
`x86_64-unknown-linux-musl` แทน `x86_64-unknown-linux-gnu` ทั่วไป — ตามที่ Part 96 อธิบายไว้ musl libc ทำให้ได้
**static binary** ที่ไม่ต้องพึ่ง shared library ของระบบปฏิบัติการเลย (ไม่ dynamic link กับ glibc) ทำให้ binary
เดียวนี้รันได้บน Linux distribution ไหนก็ได้โดยไม่ต้องกังวลเรื่องเวอร์ชัน glibc ที่ต่างกัน — เหมาะมากสำหรับ
binary ที่แจกจ่ายให้ผู้ใช้ทั่วไปดาวน์โหลดไปรันตรง ๆ

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - "v*.*.*"

permissions:
  contents: write

jobs:
  build:
    name: Build (${{ matrix.target }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
            binary_name: library_api
            asset_name: library_api-linux-x86_64-musl
          - target: x86_64-apple-darwin
            os: macos-latest
            binary_name: library_api
            asset_name: library_api-macos-x86_64
          - target: x86_64-pc-windows-msvc
            os: windows-latest
            binary_name: library_api.exe
            asset_name: library_api-windows-x86_64.exe
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install musl build tools
        if: matrix.target == 'x86_64-unknown-linux-musl'
        run: sudo apt-get update && sudo apt-get install -y musl-tools

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.target }}

      - name: Build release binary
        run: cargo build --release --target ${{ matrix.target }}

      - name: Package binary
        shell: bash
        run: |
          mkdir -p dist
          cp "target/${{ matrix.target }}/release/${{ matrix.binary_name }}" "dist/${{ matrix.asset_name }}"

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.asset_name }}
          path: dist/${{ matrix.asset_name }}

  release:
    name: Create GitHub Release
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - name: Download all build artifacts
        uses: actions/download-artifact@v4
        with:
          path: dist

      - name: Create Release and attach binaries
        uses: softprops/action-gh-release@v2
        with:
          files: dist/**/*
          generate_release_notes: true
```

**อธิบายทีละส่วน:**

- **`on: push: tags: ["v*.*.*"]`** — trigger เฉพาะเมื่อมีการ push **tag** ที่ตรงตาม pattern (เช่น `v1.0.0`,
  `v2.3.1` — ตาม convention ของ [Semantic Versioning](https://semver.org/) ที่ Part 17 แนะนำไว้แล้วตอนพูดถึง
  `Cargo.toml` version) การสร้าง release ในทางปฏิบัติทำโดย `git tag v1.0.0 && git push origin v1.0.0` เท่านั้น
  ไม่เกี่ยวกับ push ไปยัง branch แบบปกติเลย
- **`permissions: contents: write`** — จำเป็นเพื่อให้ `GITHUB_TOKEN` มีสิทธิ์สร้าง Release ใหม่ในหน้า Releases
  ของ repository ได้ (การสร้าง release ถือเป็นการเปลี่ยนแปลง "content" ของ repository ในมุมมองของ permission
  model ของ GitHub)
- **`strategy.matrix.include:`** — รูปแบบ matrix ที่ต่างจากหัวข้อ 97.6 เล็กน้อย: แทนที่จะ cross-product ค่าจาก
  หลาย list (`os` × `rust`) เราใช้ `include:` เพื่อกำหนด **combination ที่ต้องการแบบเจาะจง** ทีละชุด (target,
  os, ชื่อ binary, ชื่อไฟล์ปลายทาง) — เหมาะกับกรณีที่ค่าต่าง ๆ ผูกกันเป็นชุด (target นี้ต้องคู่กับ os นี้เท่านั้น
  ไม่ใช่ทุก target รันได้ทุก os) ต่างจาก matrix แบบ cross-product ในหัวข้อ 97.6 ที่ทุก combination สมเหตุสมผลจริง
- **`musl-tools` บน Ubuntu** — target `x86_64-unknown-linux-musl` ต้องใช้ linker และ library ของ musl libc ที่
  ไม่ได้ติดตั้งมาโดย default บน `ubuntu-latest` runner ต้องติดตั้งเพิ่มผ่าน `apt-get install musl-tools` ก่อน
  build เสมอ (ถ้าลืมขั้นตอนนี้ `cargo build --target x86_64-unknown-linux-musl` จะ fail ด้วย linker error ทันที)
- **`actions/upload-artifact` / `actions/download-artifact`** — คู่ action ทางการที่ใช้ส่งไฟล์ **ข้าม job**
  ภายใน workflow เดียวกัน (จำได้จากหัวข้อ 97.2 ว่าแต่ละ job รันบน runner คนละเครื่อง ไม่มีการแชร์ filesystem กัน
  โดยตรง) — job `build` (ที่รันเป็น matrix สามตัวพร้อมกัน) แต่ละตัว upload binary ของตัวเองเป็น artifact แยกกัน
  แล้ว job `release` (ตัวเดียว รอด้วย `needs: [build]`) download **artifact ทั้งหมด** มารวมกันในโฟลเดอร์ `dist/`
  ก่อนสร้าง release
- **`softprops/action-gh-release`** — action ที่ยอมรับกว้างขวางในชุมชนสำหรับสร้าง GitHub Release พร้อมแนบไฟล์
  (`files: dist/**/*` ใช้ glob pattern ระบุไฟล์ทั้งหมดที่ดาวน์โหลดมา) `generate_release_notes: true` สั่งให้
  GitHub สร้าง release note อัตโนมัติจาก pull request/commit ที่รวมอยู่ระหว่าง tag นี้กับ tag ก่อนหน้า (ฟีเจอร์
  ทางการของ GitHub Release ไม่ต้องเขียน changelog มือ)

**สิ่งที่ตรวจสอบได้จริงในบทนี้:** เราตรวจสอบ syntax ของ YAML ทั้งไฟล์ด้วย parser และยืนยันด้วยการรันจริงบน
เครื่องว่า `cargo build --release --target x86_64-unknown-linux-musl` (หลังติดตั้ง `musl-tools` และ
`rustup target add x86_64-unknown-linux-musl`) เป็นคำสั่งที่ทำงานถูกต้องตามที่ Part 96 อธิบายไว้ — แต่การสร้าง
GitHub Release จริง, การ push tag จริง, และการที่ artifact ถูกดาวน์โหลด/แนบเข้า release จริง ล้วนต้องอาศัย
GitHub repository จริงที่รัน Actions ได้จริง ซึ่งนอกเหนือขอบเขตที่ sandbox นี้ยืนยันได้ตรง ๆ

### 97.11 Security Scanning: `cargo-audit`

เครื่องมือตรวจสอบความปลอดภัยของ dependency ที่เป็นมาตรฐานหลักในระบบ Rust คือ **`cargo-audit`** — เครื่องมือ
ทางการของทีม [RustSec](https://rustsec.org/) ที่ดูแล **RustSec Advisory Database** (ฐานข้อมูลกลางที่รวบรวม
ช่องโหว่ความปลอดภัยที่พบใน crate บน crates.io ตั้งชื่อรหัสแบบ `RUSTSEC-YYYY-NNNN`) มีเครื่องมืออีกตัวชื่อ
**`cargo-deny`** ที่ทำหน้าที่กว้างกว่า (ตรวจทั้งช่องโหว่ความปลอดภัย, license compliance, และ dependency ที่ถูก
แบนหรือซ้ำซ้อนหลายเวอร์ชัน) — สำหรับบทนี้เราโฟกัสที่ `cargo-audit` เพราะเป็นเครื่องมือที่ตรงประเด็นเรื่อง
"ช่องโหว่ความปลอดภัยของ dependency" ที่สุด ใช้งานง่ายกว่าสำหรับการเริ่มต้น (ทีมที่ต้องการตรวจ license เพิ่มด้วย
ควรพิจารณา `cargo-deny` ต่อยอด)

**ติดตั้งและทดสอบจริง** — เราสร้างโปรเจกต์ scratch ที่เพิ่ม dependency เวอร์ชันเก่าที่มีช่องโหว่ที่รู้จักแล้ว
(`time = "0.1.45"`) เพื่อพิสูจน์ว่า `cargo-audit` จับได้จริง:

```bash
cargo install cargo-audit --locked
```

```
$ cargo audit
    Fetching advisory database from `https://github.com/RustSec/advisory-db.git`
      Loaded 1271 security advisories (from /root/.cargo/advisory-db)
    Updating crates.io index
    Scanning Cargo.lock for vulnerabilities (7 crate dependencies)
Crate:     time
Version:   0.1.45
Title:     Potential segfault in the time crate
Date:      2020-11-18
ID:        RUSTSEC-2020-0071
URL:       https://rustsec.org/advisories/RUSTSEC-2020-0071
Severity:  6.2 (medium)
Solution:  Upgrade to >=0.2.23

error: 1 vulnerability found!
```

(`$?` หลังคำสั่งนี้คือ `1` — exit code ที่ไม่ใช่ 0 พิสูจน์ว่า `cargo-audit` เหมาะกับการใช้เป็น CI gate ได้จริง
เช่นเดียวกับ `cargo fmt --check`/`cargo clippy -- -D warnings`) หลังจากลบ dependency ที่มีปัญหาออก (`time`
เวอร์ชันเก่า) แล้วรัน `cargo audit` ซ้ำ คำสั่งจบด้วย exit code `0` โดยไม่มีข้อความ error แสดงเลย — ยืนยันว่า
เครื่องมือทำงานตามที่คาดหวังทั้งสองกรณี (พบช่องโหว่ → fail, ไม่พบ → pass)

**ผนวกเข้า CI workflow** ใช้ [`taiki-e/install-action`](https://github.com/taiki-e/install-action) (action ที่
รู้จักกันดีในชุมชน Rust สำหรับดาวน์โหลด binary สำเร็จรูปของเครื่องมือ cargo subcommand ที่นิยม แทนการ
`cargo install` ที่ต้อง compile จาก source ทุกครั้ง — ประหยัดเวลาได้มาก เพราะ `cargo-audit` เองก็เป็นโปรแกรม
Rust ขนาดใหญ่ที่ใช้เวลา compile นานหลายนาทีถ้า compile จาก source ทุก CI run):

```yaml
# .github/workflows/security-audit.yml
name: Security Audit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    # รันทุกวันจันทร์ 06:00 UTC เพื่อจับช่องโหว่ใหม่ที่ถูกประกาศเข้าฐานข้อมูล
    # แม้ว่า Cargo.lock ของโปรเจกต์เราจะไม่ได้เปลี่ยนแปลงเลยก็ตาม
    - cron: "0 6 * * 1"

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install cargo-audit
        uses: taiki-e/install-action@cargo-audit

      - name: Run cargo audit
        run: cargo audit
```

**ทำไมต้องมี `schedule:` trigger ด้วย ไม่ใช่แค่ `push`/`pull_request`?** จุดนี้สำคัญมากและเป็นเหตุผลเชิงลึกที่
มือใหม่มักมองข้าม: ช่องโหว่ความปลอดภัยของ dependency **ถูกค้นพบและประกาศ "หลังจาก" ที่โค้ดของเราถูก merge ไปแล้ว
ก็ได้** — สมมติว่า `Cargo.lock` ของเราไม่เปลี่ยนแปลงเลยมาสามเดือน (ไม่มี push/PR ใหม่) แต่เดือนนี้มีคนค้นพบ
ช่องโหว่ใน dependency ตัวหนึ่งที่เราใช้อยู่ (RUSTSEC ประกาศ advisory ใหม่) — ถ้า workflow รันแค่ตอน
`push`/`pull_request` เราจะไม่รู้เรื่องช่องโหว่นี้เลยจนกว่าจะมีการเปลี่ยนแปลงโค้ดครั้งถัดไป การมี
`schedule: cron: "0 6 * * 1"` (ทุกวันจันทร์) ทำให้ pipeline **ไปตรวจสอบเชิงรุกเป็นระยะ แม้ไม่มีการเปลี่ยน
แปลงโค้ดใด ๆ** เพื่อจับความเสี่ยงที่เกิดจาก "โลกภายนอกเปลี่ยน" ไม่ใช่แค่ "โค้ดของเราเปลี่ยน"

### 97.12 Capstone: CI/CD Pipeline สมบูรณ์สำหรับ `library_api` (Part 92-94)

มาถึงจุดสุดยอดของบทนี้: ประกอบทุกอย่างที่เรียนมาทั้งหมดเข้าเป็น **pipeline เดียวที่ใช้งานได้จริงในระดับ
production** สำหรับ `library_api` (โปรเจกต์ capstone จาก Part 92-94) ครอบคลุมทุกความต้องการ: lint/test/coverage
บนทุก pull request, build+push Docker image เมื่อ merge เข้า `main`, และ build+release binary หลายแพลตฟอร์ม
เมื่อมีการ tag เวอร์ชันใหม่ — ทั้งหมดในไฟล์เดียว โดยใช้ `if:` แยกเงื่อนไขของแต่ละ job ตามประเภท event

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  DATABASE_URL: postgres://postgres:postgres@localhost:5432/library_api_test

concurrency:
  group: ci-cd-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ────────────────────────────────────────────────────────────
  # Job 1: Format + Clippy — รันทุกครั้งที่มี push/PR (เร็วที่สุด, ไม่ต้องมี Postgres)
  # ────────────────────────────────────────────────────────────
  lint:
    name: Format and Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Run clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

  # ────────────────────────────────────────────────────────────
  # Job 2: Build + Test พร้อม PostgreSQL จริง — รันทุกครั้งที่มี push/PR
  # ────────────────────────────────────────────────────────────
  test:
    name: Build and Test (${{ matrix.rust }})
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        rust: [stable, beta]
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: library_api_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U postgres"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain (${{ matrix.rust }})
        uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.rust }}

      - name: Install sqlx-cli
        run: cargo install sqlx-cli --no-default-features --features rustls,postgres --locked

      - name: Run database migrations
        run: sqlx migrate run

      - name: Build
        run: cargo build --all-targets --verbose

      - name: Run tests
        run: cargo test --all-features --verbose

  # ────────────────────────────────────────────────────────────
  # Job 3: Code Coverage — รันบน push/PR เฉพาะ stable channel (ไม่ต้องซ้ำทุก matrix)
  # ────────────────────────────────────────────────────────────
  coverage:
    name: Code Coverage
    needs: [test]
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: library_api_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd="pg_isready -U postgres"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          components: llvm-tools-preview

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2

      - name: Install cargo-llvm-cov
        uses: taiki-e/install-action@cargo-llvm-cov

      - name: Install sqlx-cli
        run: cargo install sqlx-cli --no-default-features --features rustls,postgres --locked

      - name: Run database migrations
        run: sqlx migrate run

      - name: Generate coverage report
        run: cargo llvm-cov --all-features --lcov --output-path lcov.info

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v5
        with:
          files: lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false

  # ────────────────────────────────────────────────────────────
  # Job 4: Security Audit — รันทุกครั้งที่มี push/PR
  # ────────────────────────────────────────────────────────────
  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install cargo-audit
        uses: taiki-e/install-action@cargo-audit

      - name: Run cargo audit
        run: cargo audit

  # ────────────────────────────────────────────────────────────
  # Job 5: Build + Push Docker Image — เฉพาะตอน merge เข้า main เท่านั้น
  # ────────────────────────────────────────────────────────────
  docker:
    name: Build and Push Docker Image
    needs: [lint, test, audit]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,format=long
            type=raw,value=latest

      - name: Build and push image
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ────────────────────────────────────────────────────────────
  # Job 6+7: Release — เฉพาะตอน push tag รูปแบบ v*.*.* เท่านั้น
  # ────────────────────────────────────────────────────────────
  release-build:
    name: Build Release Binary (${{ matrix.target }})
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
            binary_name: library_api
            asset_name: library_api-linux-x86_64-musl
          - target: x86_64-apple-darwin
            os: macos-latest
            binary_name: library_api
            asset_name: library_api-macos-x86_64
          - target: x86_64-pc-windows-msvc
            os: windows-latest
            binary_name: library_api.exe
            asset_name: library_api-windows-x86_64.exe
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install musl build tools
        if: matrix.target == 'x86_64-unknown-linux-musl'
        run: sudo apt-get update && sudo apt-get install -y musl-tools

      - name: Cache cargo dependencies
        uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.target }}

      - name: Build release binary
        run: cargo build --release --target ${{ matrix.target }}

      - name: Package binary
        shell: bash
        run: |
          mkdir -p dist
          cp "target/${{ matrix.target }}/release/${{ matrix.binary_name }}" "dist/${{ matrix.asset_name }}"

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.asset_name }}
          path: dist/${{ matrix.asset_name }}

  release-publish:
    name: Create GitHub Release
    if: startsWith(github.ref, 'refs/tags/v')
    needs: [release-build]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Download all build artifacts
        uses: actions/download-artifact@v4
        with:
          path: dist

      - name: Create Release and attach binaries
        uses: softprops/action-gh-release@v2
        with:
          files: dist/**/*
          generate_release_notes: true
```

**ภาพรวมการไหลของ pipeline นี้ (อธิบายเป็นลำดับเหตุการณ์):**

1. **ทุก pull request** → job `lint`, `test` (matrix stable+beta), `audit` รันพร้อมกันทันที ถ้าทั้งหมดผ่าน job
   `coverage` จะรันต่อ (รอ `test` เสร็จก่อนตาม `needs: [test]`) และอัปโหลดผลไปยัง Codecov — job `docker` และ
   `release-*` จะ**ไม่รันเลย** เพราะเงื่อนไข `if:` ของ job เหล่านั้นตรวจ `github.ref`/`github.event_name` ที่
   ไม่ตรงกับ pull request
2. **เมื่อ PR ถูก merge เข้า `main`** (เกิด push event บน `refs/heads/main`) → `lint`, `test`, `audit`,
   `coverage` รันอีกครั้ง (ยืนยันซ้ำว่า `main` ยังผ่านหลัง merge) และครั้งนี้เงื่อนไขของ job `docker` เป็นจริง
   ทำให้มันรันต่อ (หลังรอ `needs: [lint, test, audit]` ผ่านก่อน) — build image ใหม่แล้ว push ขึ้น `ghcr.io`
   พร้อม tag `latest` และ `sha-<commit>`
3. **เมื่อมีคน tag เวอร์ชันใหม่** (`git tag v1.2.0 && git push origin v1.2.0`) → เกิด push event บน
   `refs/tags/v1.2.0` ทำให้เงื่อนไขของ `release-build`/`release-publish` เป็นจริง (ส่วน `lint`/`test`/`audit`/
   `coverage`/`docker` ไม่รันในกรณีนี้ เพราะ trigger `push` ใน workflow นี้ระบุ `branches: [main]` **และ**
   `tags: ["v*.*.*"]` แยกกัน — เมื่อ ref ที่ trigger เป็น tag ไม่ใช่ branch `main` เงื่อนไข `if:` ของ job ที่เช็ค
   `github.ref == 'refs/heads/main'` จะเป็นเท็จ ทำให้ job เหล่านั้น skip แต่ job ที่เช็ค `startsWith(github.ref,
   'refs/tags/v')` จะเป็นจริงพอดี) → build binary 3 แพลตฟอร์ม → สร้าง GitHub Release พร้อมแนบไฟล์ทั้งหมด

การออกแบบนี้แสดงหลักการสำคัญของ CD ที่ดี: **แต่ละ job ทำหน้าที่เดียวที่ชัดเจน, พึ่งพากันผ่าน `needs:` เท่าที่
จำเป็นจริง (ไม่ผูกทุกอย่างเป็น sequential โดยไม่มีเหตุผล), และเงื่อนไข `if:` ควบคุมว่า "การกระทำที่มีผลกระทบจริง
ภายนอก" (push image, สร้าง release) เกิดขึ้นเฉพาะตอนที่ถูกต้องเท่านั้น** — pull request ทดสอบได้เต็มรูปแบบโดยไม่
เสี่ยง side effect ใด ๆ ต่อ production, ในขณะที่ `main` และ tag แต่ละแบบมีผลลัพธ์ที่เหมาะสมกับบทบาทของมัน

### 97.13 Secrets, Permissions และข้อจำกัดด้านความปลอดภัยของ Pull Request จาก Fork

ตลอดบทนี้เราใช้ `secrets.GITHUB_TOKEN` และ `secrets.CODECOV_TOKEN` อยู่หลายจุด ถึงเวลาเจาะลึกกลไกเบื้องหลังของ
**secret** ใน GitHub Actions ให้ครบ เพราะมีข้อจำกัดด้านความปลอดภัยที่ถ้าไม่เข้าใจจะทำให้ debug pipeline ยากมาก
โดยเฉพาะ workflow ที่เกี่ยวกับ fork

**Secret สองระดับที่ต่างกัน:**

- **Repository secret** — ตั้งค่าที่ **Settings → Secrets and variables → Actions** ของ repository ใช้ได้กับ
  ทุก workflow ในทุก branch (เช่น `CODECOV_TOKEN` ที่เราตั้งค่าไว้ในหัวข้อ 97.8)
- **Environment secret** — ผูกกับ "environment" ที่ตั้งไว้ล่วงหน้า (เช่น `production`, `staging`) มีประโยชน์
  มากสำหรับ workflow ที่ deploy จริง เพราะสามารถตั้ง **protection rule** ได้ เช่น "ต้องมีคน approve ก่อน job
  ที่ใช้ environment นี้จะรันต่อได้" — ใช้บ่อยในบริบท CD ที่ deploy ขึ้น production จริง (นอกเหนือขอบเขตของบทนี้
  ที่โฟกัสแค่ push image ขึ้น registry แต่ควรรู้จักไว้สำหรับขั้นต่อไปที่ deploy จริงบน cloud ตาม Part 101)

**`GITHUB_TOKEN` คือ secret พิเศษที่ GitHub สร้างให้อัตโนมัติทุก workflow run** ไม่ต้องสร้างเองหรือตั้งค่าอะไร
เพิ่ม — มีอายุจำกัดแค่ตลอด run เดียว (หมดอายุอัตโนมัติเมื่อ job จบ) และสิทธิ์ของมันถูกกำหนดผ่าน block
`permissions:` ที่เราใช้ไปแล้วในหัวข้อ 97.9/97.10 (`packages: write` สำหรับ push image, `contents: write`
สำหรับสร้าง release) — ค่า default ของสิทธิ์นี้คือ **read-only ทุกอย่าง** เพื่อความปลอดภัย (นโยบายนี้เปลี่ยนมา
จากเดิมที่ default เป็น read-write ทุกอย่าง — ถ้า workflow ของคุณเก่ากว่านี้และไม่เคยระบุ `permissions:` มา
ก่อน ควรตรวจสอบว่า repository ตั้ง default ไว้อย่างไรที่ **Settings → Actions → General → Workflow permissions**)

**ข้อจำกัดสำคัญที่สุดที่ต้องรู้: secret ทุกตัว (รวม `GITHUB_TOKEN` ที่มีสิทธิ์เขียน) จะ "ว่างเปล่า" โดยอัตโนมัติ
เมื่อ workflow ถูก trigger จาก pull request ที่มาจาก fork ของคนนอก** นี่คือฟีเจอร์ความปลอดภัยที่ GitHub ออกแบบมา
ตั้งใจ — ป้องกันไม่ให้คนแปลกหน้าส่ง pull request ที่มีโค้ดแอบขโมย secret ของ repository เรา (เช่น เขียน step
`run: echo ${{ secrets.CODECOV_TOKEN }} | curl ...` ส่ง token ออกไปที่เซิร์ฟเวอร์ของตัวเอง) ผลกระทบเชิงปฏิบัติ:

- Job `docker` และ `release-*` ในบทนี้จะไม่มีวันรันจาก pull request อยู่แล้ว (ตามเงื่อนไข `if:` ที่ตรวจ
  `github.event_name == 'push'`) จึงไม่กระทบ — เขียนไว้แบบนี้ตั้งแต่แรกก็เพราะเหตุผลด้านความปลอดภัยนี้ส่วนหนึ่ง
  ด้วย ไม่ใช่แค่เหตุผลเรื่อง "อย่า deploy จากโค้ดที่ยังไม่ merge" เพียงอย่างเดียว
- Job `coverage` ที่ upload ไปยัง Codecov ด้วย `secrets.CODECOV_TOKEN` **จะรันจาก pull request ได้ปกติ** (ตาม
  design ของบทนี้ที่อยากเห็น coverage ก่อน merge) แต่ถ้า PR นั้นมาจาก fork ของคนนอก ค่า `secrets.CODECOV_TOKEN`
  จะเป็นค่าว่าง ทำให้ step upload อาจ fail หรือถูก skip แบบเงียบ ๆ (ขึ้นกับพฤติกรรมของ action นั้น ๆ) — สำหรับ
  repository ที่เป็น open source และรับ PR จากคนนอกบ่อย ควรตั้ง `fail_ci_if_error: false` (ตามที่บทนี้ตั้งไว้ใน
  job `coverage` ของหัวข้อ 97.12) เพื่อไม่ให้ PR ที่มาจาก fork ถูกบล็อกจากปัญหานี้ที่ผู้ส่ง PR ควบคุมไม่ได้เลย

หลักการที่ควรจำ: **อย่าพึ่งพา secret ใน job ที่ต้องรันจาก pull request ของคนนอกเป็นเงื่อนไขบังคับผ่าน (hard gate)**
— ใช้ secret ได้ในบริบทที่เป็น "extra info" (เช่น coverage report ที่ไม่มีก็ยังใช้งานได้) แต่ไม่ควรใช้ในบริบทที่
ทำให้ PR ทั้งใบ merge ไม่ได้เพราะ secret ที่ contributor ไม่มีสิทธิ์เข้าถึงโดยธรรมชาติของระบบ

### 97.14 Trigger เพิ่มเติมที่ควรรู้จัก และการควบคุมต้นทุนของ CI Minutes

นอกจาก `push`/`pull_request`/`schedule` ที่ใช้ไปแล้วตลอดบทนี้ ยังมี trigger อีกสองสามตัวที่มีประโยชน์มากในทาง
ปฏิบัติ:

**`workflow_dispatch` — รันด้วยมือผ่านหน้าเว็บ GitHub**

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment ที่จะ deploy"
        required: true
        default: "staging"
        type: choice
        options:
          - staging
          - production
```

เมื่อเพิ่ม trigger นี้ หน้า **Actions** ของ repository จะมีปุ่ม "Run workflow" ให้กดรันด้วยมือได้ทันที พร้อม
ฟอร์มให้กรอกค่า `inputs` ที่กำหนดไว้ (เข้าถึงค่าที่กรอกได้ผ่าน `${{ github.event.inputs.environment }}` ในทุก
step ของ workflow) มีประโยชน์มากสำหรับ workflow ที่ไม่ต้องการให้รันอัตโนมัติทุกครั้ง เช่น การ deploy ขึ้น
production ที่ต้องการให้คนกดยืนยันเองอย่างมีสติ ไม่ใช่รันอัตโนมัติทุกครั้งที่มี push (ต่างจาก workflow ในบทนี้ที่
ตั้งใจให้ทุกอย่างอัตโนมัติเต็มรูปแบบตามเงื่อนไข event เพราะ scope ของ capstone นี้ยังไม่ต้องมี manual gate)

**การควบคุมต้นทุนของ CI minutes**

GitHub Actions ให้ **นาทีการรันฟรีไม่จำกัดสำหรับ public repository** บน GitHub-hosted runner มาตรฐาน แต่สำหรับ
**private repository บน Free plan จะได้โควตาจำกัดต่อเดือน** (จำนวนที่แน่นอนเปลี่ยนแปลงได้ตามนโยบายราคาของ GitHub
ในแต่ละช่วงเวลา ควรตรวจสอบหน้า pricing ปัจจุบันของ GitHub เสมอ) ที่สำคัญกว่าตัวเลขคือ**ตัวคูณต้นทุนที่ต่างกันตาม
runner OS**: runner `ubuntu-latest` คิดอัตรา 1x ต่อนาทีที่ใช้จริง ในขณะที่ `windows-latest` คิดอัตราสูงกว่า และ
`macos-latest` คิดอัตราสูงกว่านั้นอีกมาก (สาเหตุคือต้นทุนฮาร์ดแวร์ของ Apple ที่ GitHub ต้องเช่า/ดูแลแพงกว่ามาก)
ตัวเลขที่แน่นอนของตัวคูณเปลี่ยนแปลงได้ตามประกาศราคาของ GitHub ควรตรวจสอบหน้าเอกสารทางการก่อนวางแผนงบประมาณจริง
แต่หลักการที่ไม่เปลี่ยนคือ: **matrix ที่รวม `macos-latest`/`windows-latest` มีค่าใช้จ่ายต่อนาทีสูงกว่า `ubuntu-latest`
เสมอ** ตอกย้ำประเด็นจากหัวข้อ 97.6 อีกครั้งว่าการเลือก matrix ให้ตรงกับพื้นผิวการใช้งานจริงของโปรเจกต์ (ไม่ใช่
เพิ่มทุก OS/channel "เผื่อไว้" โดยไม่มีเหตุผล) ไม่ได้ช่วยแค่เรื่องเวลารอผลลัพธ์ แต่ช่วยเรื่องงบประมาณจริงของทีม
ด้วย โดยเฉพาะทีมที่มี private repository และ workflow ที่รันบ่อยหลายสิบครั้งต่อวัน

**`timeout-minutes` — ป้องกัน job ที่ค้างไม่จบ**

อีกเทคนิคหนึ่งที่ควบคุมต้นทุนได้ดีคือกำหนด `timeout-minutes:` ให้ทุก job (ค่า default ของ GitHub คือ 360 นาที
ถ้าไม่กำหนด ซึ่งนานเกินไปมากสำหรับ job ที่ควรจบใน 5-10 นาที) เช่น:

```yaml
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      # ...
```

ถ้า job นี้ค้างเกิน 15 นาที (เช่น test แขวนรอ resource ที่ไม่มีวันพร้อม หรือ deadlock ที่ Part 40 อธิบายไว้ว่า
เกิดขึ้นได้จริงถ้า concurrency ออกแบบผิด) GitHub Actions จะ**ยกเลิก job นั้นทันทีแทนที่จะรอจนครบ 360 นาที** ซึ่ง
ช่วยประหยัด CI minute จำนวนมากจากบั๊กที่ทำให้ job ค้างโดยไม่ตั้งใจ และยังทำให้ทีมรู้ผลเร็วขึ้นว่ามีปัญหาเกิดขึ้น
ด้วย

### 97.15 ทำให้ CI "บังคับผ่านจริง": Branch Protection และ Required Status Checks

หัวข้อ 97.1 อ้างว่า CI คือ "ประตูที่บังคับ" ให้ทุก commit ผ่านการตรวจสอบก่อน merge — แต่ในความเป็นจริง **แค่มี
workflow ที่รันแล้วแสดงผล ❌/✅ บนหน้า pull request ยังไม่ได้ "บังคับ" อะไรเลยโดยอัตโนมัติ** ถ้าไม่มีการตั้งค่า
เพิ่มเติม ปุ่ม "Merge pull request" บนหน้า GitHub จะยังกดได้ตามปกติแม้ job `test` จะแสดงเป็นสีแดงอยู่ก็ตาม — สิ่ง
ที่ทำให้ CI "มีเขี้ยว" จริงคือฟีเจอร์ **Branch Protection Rules** ที่ตั้งแยกจากไฟล์ workflow (ตั้งที่ **Settings →
Branches** ของ repository ไม่ใช่ในไฟล์ YAML) ประกอบด้วยเงื่อนไขสำคัญที่เกี่ยวข้องกับบทนี้โดยตรง:

- **"Require status checks to pass before merging"** — เลือก job ที่ต้องการบังคับจากรายการ (GitHub จะแสดง
  รายชื่อ job ที่เคยรันบน repository ให้เลือก เช่น `lint`, `test (stable)`, `test (beta)`, `audit` จาก workflow
  ในหัวข้อ 97.12) เมื่อตั้งค่านี้แล้ว **ปุ่ม merge จะถูก disable โดยอัตโนมัติ** จนกว่า job ที่เลือกไว้ทั้งหมด
  จะรันจบและผ่าน (สีเขียว) เท่านั้น — นี่คือจุดที่เปลี่ยน CI จาก "ข้อมูลให้ดูเฉย ๆ" ให้เป็น "เงื่อนไขบังคับ" จริง
- **"Require branches to be up to date before merging"** — บังคับให้ branch ของ PR ต้อง merge/rebase เอา
  commit ล่าสุดของ `main` เข้ามาก่อน แล้วรัน CI ซ้ำกับโค้ดที่รวมกันแล้วจริง ๆ (ไม่ใช่แค่รัน CI กับโค้ดของ PR
  เดี่ยว ๆ ที่อาจไม่มีปัญหาเลย แต่พอ merge เข้ากับ `main` ที่มีการเปลี่ยนแปลงอื่นมาแล้วกลับชนกัน) ป้องกันปัญหาที่
  เรียกว่า **"semantic merge conflict"** — โค้ดสอง PR ที่แต่ละอันแยกกันดูปกติ compile ผ่าน แต่พอรวมกันแล้ว
  compile ไม่ผ่านหรือ logic ขัดกัน (เช่น PR หนึ่งลบฟังก์ชันที่อีก PR เพิ่งเพิ่มโค้ดมาเรียกใช้)
- **"Require a pull request before merging" + "Require approvals"** — ไม่เกี่ยวกับ CI โดยตรง แต่มักตั้งควบคู่
  กันเสมอในทีมจริง (บังคับให้ต้องมีคน review อนุมัติ ไม่ใช่แค่ CI ผ่านอย่างเดียว)

**ทำไมต้องแยกการตั้งค่านี้ออกจากไฟล์ YAML?** เพราะ Branch Protection Rule เป็นกฎของ **repository/branch**
(ตัดสินใจโดยทีม/เจ้าของ repository ว่า "เกณฑ์อะไรที่ยอมรับได้สำหรับ branch หลัก") ในขณะที่ไฟล์ workflow เป็น
**นิยามว่า check นั้นทำงานอย่างไร** (ตัดสินใจโดยคนเขียนโค้ด/DevOps) การแยกสองสิ่งนี้ออกจากกันทำให้เปลี่ยนได้
อิสระจากกัน — เพิ่ม/ลด job ที่ "บังคับ" ได้โดยไม่ต้องแก้ไฟล์ YAML เลย (แค่ไปติ๊ก/ถอดติ๊กใน Settings) และในทาง
กลับกัน แก้รายละเอียดภายในของ job (เช่น เปลี่ยนคำสั่งที่รันภายใน `test`) ได้โดยไม่ต้องยุ่งกับ policy ของ branch

**ข้อควรระวังที่พบบ่อย:** ถ้า workflow ใช้ `strategy.matrix` (เช่น `test (stable)` และ `test (beta)` จากหัวข้อ
97.12) ชื่อ job ที่ปรากฏใน Branch Protection settings จะเป็นชื่อ**เฉพาะของแต่ละ combination** (เช่น
`Build and Test (stable)` และ `Build and Test (beta)` แยกกันเป็นสอง entry) — ถ้าเพิ่ม/ลด matrix ในไฟล์ YAML
ภายหลัง (เช่น เพิ่ม `nightly` เข้าไปแล้วเอา `beta` ออก) ชื่อ job ใน Branch Protection settings ที่ตั้งค้างไว้เดิม
จะ**อ้างถึง job ที่ไม่มีอยู่แล้ว** ทำให้ GitHub แสดงสถานะ "Expected — Waiting for status to be reported" ค้างอยู่
ตลอดไป (เพราะรอ check ที่ชื่อนั้นซึ่งไม่มีวันเกิดขึ้นอีก) และ**บล็อกการ merge อย่างไม่มีที่สิ้นสุด** — ต้องกลับไป
อัปเดตรายชื่อ required status check ใน Branch Protection settings ทุกครั้งที่เปลี่ยนโครงสร้าง matrix หรือ
ชื่อ job ในไฟล์ workflow เป็นความรับผิดชอบที่มักถูกมองข้ามเมื่อ refactor workflow แบบไม่ได้ตรวจสอบผลกระทบให้ครบ

### 97.16 ตารางสรุป Action ทั้งหมดที่ใช้ในบทนี้

เพื่อให้ง่ายต่อการอ้างอิงกลับมาใช้จริง (และช่วยตรวจสอบว่า pin เวอร์ชันตรงกันทุกที่ในโปรเจกต์เดียวกัน) นี่คือ
ตารางสรุป action ทุกตัวที่ปรากฏในบทนี้ พร้อมหน้าที่และหัวข้อที่อธิบายไว้:

| Action | เวอร์ชันที่ใช้ | หน้าที่ | อธิบายไว้ที่หัวข้อ |
|---|---|---|---|
| `actions/checkout` | `@v4` | Clone repository เข้า runner — ต้องมีเกือบทุก workflow | 97.2 |
| `dtolnay/rust-toolchain` | `@stable` / `@master` | ติดตั้ง/สลับ Rust toolchain พร้อม component (`rustfmt`, `clippy`, `llvm-tools-preview`) และ target เพิ่มเติม | 97.5, 97.6 |
| `Swatinem/rust-cache` | `@v2` | Cache `~/.cargo` และ `target/` อย่างฉลาด แก้ปัญหา cache thrashing ของ Cargo | 97.4 |
| `actions/cache` | `@v4` | Cache โฟลเดอร์ทั่วไป (ทางเลือกพื้นฐานกว่า `Swatinem/rust-cache` สำหรับกรณีอื่นที่ไม่ใช่ Cargo) | 97.4 |
| `taiki-e/install-action` | `@cargo-llvm-cov`, `@cargo-audit` | ติดตั้ง prebuilt binary ของ cargo subcommand ที่นิยม แทนการ compile จาก source | 97.8, 97.11 |
| `docker/setup-buildx-action` | `@v3` | ติดตั้ง BuildKit สำหรับ build image แบบรองรับ cache ประเภท `type=gha` | 97.9 |
| `docker/login-action` | `@v3` | Login เข้า container registry (`ghcr.io`) | 97.9 |
| `docker/metadata-action` | `@v5` | สร้าง tag/label ของ image อัตโนมัติตาม pattern ที่กำหนด | 97.9 |
| `docker/build-push-action` | `@v6` | Build และ push Docker image ในคำสั่งเดียว | 97.9 |
| `actions/upload-artifact` | `@v4` | ส่งไฟล์ออกจาก job หนึ่งเพื่อให้ job อื่นใช้ต่อ (ข้าม runner) | 97.10 |
| `actions/download-artifact` | `@v4` | ดาวน์โหลดไฟล์ที่ upload ไว้ก่อนหน้า เข้ามารวมกันใน job ปัจจุบัน | 97.10 |
| `softprops/action-gh-release` | `@v2` | สร้าง GitHub Release พร้อมแนบไฟล์และ auto-generate release notes | 97.10 |
| `codecov/codecov-action` | `@v5` | อัปโหลดผลลัพธ์ coverage (LCOV) ไปยัง Codecov | 97.8 |

**ข้อสังเกตเรื่องความรับผิดชอบของแต่ละ action:** สังเกตว่า action ในตารางนี้แบ่งได้เป็นสามกลุ่มตามผู้ดูแล — (1)
action ทางการของ GitHub เอง (`actions/*`) ที่ได้รับการดูแลและรับประกันความเข้ากันได้ระดับสูงสุด (2) action ของ
องค์กร/ทีมที่เป็นเจ้าของเทคโนโลยีนั้นโดยตรง (`docker/*` ดูแลโดยทีม Docker, `codecov/*` ดูแลโดยทีม Codecov) และ
(3) action ของนักพัฒนาอิสระในชุมชนที่ได้รับการยอมรับกว้างขวางจนกลายเป็นมาตรฐานโดยพฤตินัยสำหรับงานนั้น ๆ
(`Swatinem/rust-cache`, `dtolnay/rust-toolchain`, `taiki-e/install-action`, `softprops/action-gh-release`) —
กลุ่มที่สามนี้คุ้มค่าที่จะใช้เพราะแก้ปัญหาเฉพาะทางที่ action ทางการไม่ได้ครอบคลุม (เช่นปัญหา Cargo caching ที่
`actions/cache` ธรรมดาแก้ได้ไม่ดีพอตามที่อธิบายในหัวข้อ 97.4) แต่ก็ควรตรวจสอบว่ายังมีการดูแล/อัปเดตต่อเนื่องอยู่
(ดูจาก commit ล่าสุด, จำนวน GitHub star, และว่าโปรเจกต์ใหญ่ ๆ ในระบบ Rust ใช้ action เดียวกันอยู่หรือไม่ — ทั้ง
สาม action ในกลุ่มนี้ที่กล่าวถึงในบทนี้ถูกใช้อย่างแพร่หลายในโปรเจกต์ Rust ระดับ production จำนวนมาก ณ เวลาที่
เขียนบทนี้)

### 97.17 GitHub-Hosted Runner vs Self-Hosted Runner: ผลกระทบต่อ Caching

ทุก workflow ในบทนี้ใช้ `runs-on: ubuntu-latest` (หรือ `macos-latest`/`windows-latest`) ซึ่งคือ **GitHub-hosted
runner** — เครื่องเสมือนที่ GitHub สร้างขึ้นใหม่ทั้งหมด (provision จาก image มาตรฐาน) ก่อนแต่ละ job เริ่มรัน แล้ว
**ทำลายทิ้งทันทีที่ job จบ** คุณสมบัติ "เครื่องใหม่เอี่ยมทุกครั้ง" (ephemeral) นี้คือสาเหตุที่การ caching ทุกเรื่อง
ในบทนี้ต้องทำผ่านฟีเจอร์ cache ของ GitHub Actions เอง (`actions/cache`/`Swatinem/rust-cache`) อย่างชัดเจน — ไม่มี
ไฟล์อะไรจากการรันครั้งก่อนหลงเหลืออยู่บนเครื่องให้ใช้ต่อโดยธรรมชาติ

ทีมที่มี workload มากพอ (รัน CI บ่อยมากจนต้นทุนของ GitHub-hosted runner สูงเกินไป ตามที่อธิบายไว้ในหัวข้อ 97.14
หรือต้องการ hardware เฉพาะทาง เช่น GPU สำหรับ machine learning) อาจเลือกใช้ **self-hosted runner** แทน — คือ
เครื่องจริง (หรือ container) ที่ทีมดูแลเอง แล้วติดตั้ง GitHub Actions runner agent ให้มาคอยรับงานจาก repository
ระบุด้วย `runs-on: self-hosted` (หรือ label ที่ทีมกำหนดเอง เช่น `runs-on: [self-hosted, linux, gpu]`)

**ผลกระทบสำคัญต่อการออกแบบ caching:** self-hosted runner **ไม่ใช่ ephemeral โดย default** — ถ้าทีมตั้งให้เป็น
เครื่องที่รันต่อเนื่อง (ไม่ใช่ container ที่สร้างใหม่ทุกครั้ง) โฟลเดอร์ `~/.cargo` และ `target/` จาก job ก่อนหน้า
จะยังอยู่บนดิสก์จริงเมื่อ job ถัดไปมารัน — ทำให้ในทางทฤษฎีไม่จำเป็นต้องพึ่ง `actions/cache`/`Swatinem/rust-cache`
เลยก็ได้ (เพราะ `target/` ที่ compile ไว้แล้วยังอยู่ตรงนั้น) แต่ก็มีข้อเสียแลกมา: **build ที่ต่างกันของหลาย
branch/PR อาจ "เปื้อน" กันเอง** ถ้าไม่ระมัดระวัง (เช่น PR สอง branch compile ทับ `target/` เดียวกันพร้อมกันถ้า
รันสอง job บนเครื่องเดียวกันในเวลาเดียวกัน) ทำให้ทีมที่ใช้ self-hosted runner ยังต้องออกแบบเรื่อง isolation ระหว่าง
job อย่างรอบคอบ (เช่น ให้แต่ละ job รันใน container แยกที่ mount volume คนละตัว) — ซึ่งอยู่นอกเหนือขอบเขตเบื้องต้น
ของบทนี้ แต่ควรรู้จักไว้เป็นแนวคิดขยายผลสำหรับทีมที่เติบโตจนต้นทุน GitHub-hosted runner สูงเกินจุดคุ้มทุน สำหรับ
โปรเจกต์ระดับที่บทนี้ครอบคลุม (`library_api` จาก Part 92-94) GitHub-hosted runner ร่วมกับ `Swatinem/rust-cache`
ตามที่อธิบายไว้ทั้งบทนี้เพียงพอและเหมาะสมที่สุดแล้ว

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืมว่า `-D warnings` เปลี่ยน clippy warning ให้เป็น hard error ที่ทำให้ job ทั้งหมดล้มเหลว

เมื่อรัน `cargo clippy --all-targets -- -D warnings` กับโค้ดที่มี lint ผิดแม้แต่ตัวเดียว jobจะ**หยุดทำงานทันที**
ไม่ใช่แค่แสดง warning เฉย ๆ แล้วผ่านต่อไป เราทดสอบจริงโดยเพิ่มโค้ดที่มี `clippy::needless_return`:

```
error: unneeded `return` statement
  --> src/lib.rs:50:5
   |
50 |     return x + 1;
   |     ^^^^^^^^^^^^
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.94.0/index.html#needless_return
   = note: `-D clippy::needless-return` implied by `-D warnings`
   = help: to override `-D warnings` add `#[allow(clippy::needless_return)]`

error: could not compile `ci_demo` (lib) due to 1 previous error
```

exit code ของคำสั่งนี้คือ `101` (ไม่ใช่ `0`) — ทำให้ CI job แสดงเป็น "❌ Failed" ทันที นี่คือพฤติกรรมที่**ตั้งใจ
ให้เกิดขึ้น** (เป็นเหตุผลที่เราเลือกใช้ `-D warnings` ตั้งแต่แรกใน Part 5) แต่มือใหม่มักตกใจตอนเจอครั้งแรกว่า
"แค่ warning ทำไม job ถึงพังไปเลย" — คำตอบคือ: **ใน CI เราต้องการให้ lint เป็นเงื่อนไขบังคับผ่าน (gate) ไม่ใช่
แค่คำแนะนำที่มองข้ามได้** วิธีแก้ที่ถูกต้องคือแก้โค้ดตามคำแนะนำของ clippy (ในกรณีนี้คือลบ `return` และ `;` ท้าย)
ไม่ใช่การใส่ `#[allow(clippy::needless_return)]` ปิด lint แบบไม่มีเหตุผล (ตามที่ Part 5 เตือนไว้เรื่อง "blanket
allow")

### 2. `sqlx::query!` compile ไม่ผ่านบน CI ทั้งที่ผ่านบนเครื่อง developer

ข้อความ error ที่พบบ่อยเมื่อ CI ไม่มี PostgreSQL รันอยู่ (ไม่มี `services: postgres:` หรือ `DATABASE_URL` ไม่ถูก
ตั้งค่า) หรือรันแต่ยังไม่ผ่าน migration:

```
error: error communicating with database: error connecting to server: Connection refused (os error 111)
  --> src/routes/books.rs:23:5
   |
23 |     sqlx::query_as!(BookRow, "SELECT id, title, author FROM books WHERE id = $1", id)
   |     ^^^^^^^^^^^^^^^^^^^^^^^^
```

หรือถ้าเชื่อมต่อฐานข้อมูลได้แต่ยังไม่รัน migration (ตารางยังไม่ถูกสร้าง):

```
error: error returned from database: relation "books" does not exist
```

สาเหตุคือลำดับ step ใน workflow ผิด — `sqlx migrate run` **ต้องรันก่อน** `cargo build`/`cargo test` เสมอ (ตามที่
อธิบายไว้ในหัวข้อ 97.7) และ `DATABASE_URL` ต้องตรงกับ credential ของ service container พอดี ตัวช่วยตรวจสอบง่าย
ๆ: เพิ่ม step `run: psql "$DATABASE_URL" -c '\dt'` (list ตารางทั้งหมด) ก่อน `cargo build` ชั่วคราวตอน debug
workflow ใหม่ ๆ เพื่อยืนยันว่าเชื่อมต่อได้และตารางถูกสร้างจริงก่อนที่จะไปหาสาเหตุอื่น

### 3. Cache ไม่ถูก restore เพราะ cache key ไม่ตรงกันข้าม matrix job

ถ้าใช้ `strategy.matrix` (เช่น หลาย Rust channel หรือหลาย OS) แต่ไม่ได้แยก `key:` ของ `Swatinem/rust-cache` ตาม
matrix เหมือนในหัวข้อ 97.6 job แต่ละตัวใน matrix จะแย่ง cache key เดียวกัน (เพราะค่า default คำนวณจาก
`Cargo.lock` ที่เหมือนกันทุก matrix combination) ทำให้เกิดพฤติกรรมที่ดูงงคือ: cache "ถูก save" จาก job ที่ทำงาน
เสร็จก่อน (เช่น `ubuntu` + `stable`) แล้ว job อื่น (เช่น `windows` + `nightly`) ไป "restore" cache ที่ผิด
platform เข้ามา ทำให้ build fail แปลก ๆ หรือช้าลงเพราะต้อง compile ใหม่ทั้งหมดอยู่ดี (binary ของ platform หนึ่ง
ใช้กับอีก platform ไม่ได้) วิธีแก้คือใส่ `key:` ที่รวมค่าที่แยกความแตกต่างของ matrix เข้าไปเสมอ เช่น
`key: ${{ matrix.os }}-${{ matrix.rust }}`

### 4. `cargo audit`/`cargo llvm-cov`/`cargo-*` subcommand ไม่มีอยู่บน runner โดย default

รันคำสั่งตรง ๆ โดยไม่ได้ติดตั้งก่อน จะเจอ error แบบนี้ (พฤติกรรมมาตรฐานของ Cargo เมื่อไม่รู้จัก subcommand ใด ๆ
ที่ไม่ได้อยู่ใน core ของ Cargo เอง):

```
error: no such command: `audit`

        View all installed commands with `cargo --list`
        Find a package to install `audit` with `cargo search cargo-audit`
```

subcommand เพิ่มเติมอย่าง `cargo-audit`, `cargo-llvm-cov`, `cargo-tarpaulin`, `sqlx-cli` **ไม่ได้ติดตั้งมาโดย
default** บน `ubuntu-latest`/`macos-latest`/`windows-latest` image ต้องติดตั้งก่อนใช้เสมอ ผ่านการ
`cargo install <tool> --locked` (ช้ากว่า เพราะ compile จาก source) หรือผ่าน `taiki-e/install-action` (เร็วกว่า
มาก เพราะดาวน์โหลด prebuilt binary — ควรเลือกใช้เมื่อเครื่องมือนั้นมีอยู่ในรายการที่ action นี้รองรับ) การ
เลือกวิธีติดตั้งที่เร็วกว่ามีผลต่อเวลารวมของ pipeline อย่างมีนัยสำคัญ เพราะเครื่องมืออย่าง `cargo-audit` ที่ต้อง
compile dependency ของตัวมันเอง (ไม่ใช่ dependency ของโปรเจกต์เรา) ก็ใช้เวลาหลายนาทีเช่นกันถ้า compile จาก
source ทุกครั้งที่ CI รัน

### 5. YAML indentation ผิดแม้แค่ช่องเดียว ทำให้ workflow ไม่รันเลยโดยไม่มี error ที่ชัดเจนในโค้ด

YAML ใช้ indentation (จำนวน space) เป็นตัวกำหนดโครงสร้างข้อมูลทั้งหมด ไม่มีวงเล็บปิดเปิดแบบ JSON ที่ช่วยให้เห็น
ขอบเขตชัดเจน — การ indent ผิดแม้แค่ 1 ช่อง (เช่น ลืมเยื้อง `run:` ให้อยู่ใต้ `steps:` ในระดับที่ถูกต้อง หรือใช้ tab
ปนกับ space ซึ่ง YAML ไม่อนุญาต) ทำให้ **GitHub ปฏิเสธ workflow ทั้งไฟล์ทันทีตั้งแต่ก่อนจะรันด้วยซ้ำ** ไม่ใช่แค่
step เดียวที่ error — หน้า "Actions" ของ repository จะแสดงข้อความในกลุ่ม "This run likely failed because of a
workflow file issue" พร้อมชี้ตำแหน่งบรรทัด/คอลัมน์ที่ parser เจอปัญหา (ข้อความที่แน่นอนอาจต่างกันไปตามรูปแบบของ
error เช่น "mapping values are not allowed in this context" สำหรับกรณีลืมเยื้อง หรือ "found character that
cannot start any token" สำหรับกรณีใช้ tab ปนกับ space) — ทั้งหมดนี้คือพฤติกรรมมาตรฐานของ YAML parser ที่ GitHub
Actions ใช้ตรวจสอบไฟล์ก่อนรันจริง (documented behavior จากเอกสารทางการของ GitHub Actions ไม่ใช่ผลจากการรันจริง
ในสภาพแวดล้อมของบทนี้ เนื่องจากต้องมี repository จริงบน GitHub เท่านั้นถึงจะเห็นข้อความนี้)

วิธีป้องกันที่ดีที่สุดคือตรวจ syntax ของไฟล์ YAML **ก่อน** push เสมอ ด้วย YAML parser ตัวไหนก็ได้ เช่นถ้ามี
Python บนเครื่อง:

```bash
python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))" .github/workflows/ci.yml
```

ถ้าไฟล์ syntax ถูกต้อง คำสั่งนี้จะจบแบบเงียบ ๆ ไม่มี output ใด ๆ (exit code 0) ถ้าผิด จะ throw exception พร้อม
บอกบรรทัด/คอลัมน์ที่ parser สับสน — เราใช้เทคนิคเดียวกันนี้ตรวจสอบทุก workflow YAML ในบทนี้จริง ก่อนยืนยันว่า
เนื้อหาถูกต้องตามหลักไวยากรณ์ (ตามที่อธิบายไว้ในหมายเหตุการตรวจสอบเนื้อหาต้นบท)

### 6. Path separator ต่างกันระหว่าง Linux/macOS กับ Windows ใน matrix build

เมื่อทำ matrix build ที่รวม `windows-latest` (เช่นหัวข้อ 97.6/97.10) shell เริ่มต้นของ runner **ต่างกัน**:
`ubuntu-latest`/`macos-latest` ใช้ `bash` เป็น default แต่ `windows-latest` ใช้ **PowerShell** เป็น default —
คำสั่งที่เขียนด้วย syntax ของ `bash` (เช่น `mkdir -p dist && cp foo dist/`) จะ**ล้มเหลว**บน Windows runner ถ้าไม่
ระบุ shell ให้ชัดเจน เพราะ PowerShell ไม่รู้จัก flag `-p` ของ `mkdir` แบบ POSIX และตัวดำเนินการ `&&` มีความหมาย
ต่างกัน ตัวอย่าง error message จริงที่พบบ่อยเมื่อ PowerShell พยายาม parse คำสั่ง bash-style:

```
mkdir : A positional parameter cannot be found that accepts argument '-p'.
At line:1 char:1
+ mkdir -p dist
+ ~~~~~~~~~~~~~
```

ทางแก้ที่ใช้ในบทนี้ (ดูหัวข้อ 97.10 ที่ step `Package binary` มี `shell: bash` ระบุไว้ชัดเจน) คือ**บังคับ shell
ให้เป็น `bash` เสมอในทุก step ที่มีคำสั่งแบบ POSIX** แม้ runner จะเป็น Windows ก็ตาม (`windows-latest` มี Git
Bash ติดตั้งไว้ให้อยู่แล้วโดย default ทำให้ `shell: bash` ใช้งานได้จริงแม้บน Windows) — หลักการคือ: **ถ้า step
ไหนมีคำสั่ง shell ที่ต้องใช้ syntax เดียวกันข้าม OS ทั้งหมดใน matrix ให้ระบุ `shell: bash` ไว้เสมอ อย่าพึ่งพา
shell default ของ runner ที่เปลี่ยนไปตาม OS**

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียน workflow ชื่อ `.github/workflows/hello-ci.yml` ที่ trigger ด้วย `push` เข้า branch `main`
   เท่านั้น มี job เดียวที่ทำ 3 อย่างตามลำดับ: checkout code, ติดตั้ง Rust toolchain ผ่าน `dtolnay/rust-toolchain`,
   รัน `cargo test --verbose` — *Hint*: โครงสร้างเหมือนหัวข้อ 97.2 แต่เพิ่ม step ติดตั้ง toolchain และเปลี่ยน
   คำสั่งสุดท้ายจาก `cargo build` เป็น `cargo test`

2. **(กลาง)** ต่อยอด workflow จากข้อ 1 ให้เพิ่ม job ที่สองชื่อ `lint` ที่รัน `cargo fmt --all -- --check` และ
   `cargo clippy --all-targets -- -D warnings` แล้วปรับให้ทั้งสอง job ใช้ `Swatinem/rust-cache@v2` เพื่อ cache
   dependency — *Hint*: job สองตัวควรแยกกัน (ไม่รวมเป็น job เดียว) เพื่อให้รันพร้อมกันได้ตามที่อธิบายไว้ในหัวข้อ
   97.5 อย่าลืมเพิ่ม `components: rustfmt, clippy` ใน step ติดตั้ง toolchain ของ job `lint`

3. **(ยาก)** เขียน workflow ที่รัน integration test ของโปรเจกต์ที่ใช้ SQLx กับ PostgreSQL จริงผ่าน
   `services: postgres:` ครบทุกส่วน (image, env, ports, health check) แล้วเพิ่ม job ที่สองที่รัน `cargo audit`
   โดยให้ job หลังนี้ใช้ `taiki-e/install-action@cargo-audit` ในการติดตั้งเครื่องมือ (ไม่ใช้ `cargo install`
   ตรง ๆ) — *Hint*: ทวนโครงสร้าง `services:` จากหัวข้อ 97.7 ให้ครบ 4 field (`image`, `env`, `ports`, `options`)
   และดูตัวอย่าง `taiki-e/install-action` จากหัวข้อ 97.8/97.11 เป็นแบบ

4. **(ยาก/ประยุกต์)** ออกแบบและเขียน workflow เดียวที่ครอบคลุมทั้งสามสถานการณ์ต่อไปนี้ด้วยเงื่อนไข `if:` ที่
   ต่างกัน: (ก) ทุก pull request รัน lint+test+audit, (ข) merge เข้า `main` เพิ่มเติมด้วยการ build (ไม่ต้อง push
   จริง ตั้ง `push: false` ใน `docker/build-push-action`) เพื่อยืนยันว่า Dockerfile build ผ่าน, (ค) push tag
   `v*.*.*` รัน build binary สำหรับ `x86_64-unknown-linux-musl` เท่านั้น (ไม่ต้องทำ multi-platform เต็มแบบหัวข้อ
   97.10) แล้วสร้าง GitHub Release — *Hint*: นำโครงสร้างจากหัวข้อ 97.12 มาตัดให้เหลือ scope เล็กลงตามที่โจทย์นี้
   กำหนด โฟกัสที่ความถูกต้องของเงื่อนไข `if:` ของแต่ละ job ให้ตรงกับ event ที่ต้องการเป๊ะ ๆ (ลองไล่ trace ว่าถ้า
   เกิด pull request job ไหนควรรัน/ไม่ควรรัน และถ้าเกิด push tag job ไหนควรรัน/ไม่ควรรัน)

**แนวทางตรวจคำตอบด้วยตัวเอง (สำหรับทุกข้อ):** ก่อนเชื่อว่า workflow ที่เขียนถูกต้อง ให้ตรวจ syntax ด้วย YAML
parser เสมอตามที่อธิบายไว้ในกับดักที่ 5 (เช่น `python3 -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))"
.github/workflows/<ไฟล์ของคุณ>.yml`) แล้วไล่อ่านทวนทุกคำสั่ง `run:` ด้วยการรันจริงบนเครื่องตัวเองก่อน (เช่น
`cargo fmt --all -- --check`, `cargo clippy --all-targets -- -D warnings`) เพื่อยืนยันว่าคำสั่งเหล่านั้นทำงาน
ถูกต้องในสภาพแวดล้อมจริงก่อนที่จะเชื่อว่า workflow ทั้งไฟล์จะทำงานถูกต้องบน GitHub Actions — หลักการเดียวกับที่
บทนี้ใช้ตรวจสอบเนื้อหาของตัวเองตามที่อธิบายไว้ในหมายเหตุต้นบท

## สรุป

บทนี้พาเราสร้าง CI/CD pipeline สำหรับโปรเจกต์ Rust ตั้งแต่ workflow ที่เรียบง่ายที่สุด (แค่ `cargo build` ตอน
push) ไปจนถึง pipeline ระดับ production ที่ครอบคลุมทุกด้าน: **lint** (`cargo fmt --check` + `cargo clippy -- -D
warnings` จาก Part 5), **test** ทั้ง unit และ integration ที่ต้องพึ่ง PostgreSQL จริงผ่าน `services:` (ต่อยอด
Part 32/33/70/71/92/95), **coverage** ด้วย `cargo-llvm-cov`, **security scanning** ด้วย `cargo-audit`,
**containerization** ที่ build/push Docker image ขึ้น `ghcr.io` (ต่อยอด Part 96), และ **release automation**
ที่สร้าง multi-platform binary พร้อม GitHub Release อัตโนมัติ

ธีมที่ย้ำซ้ำตลอดบทคือ**การจัดการ compile time ที่ช้าของ Rust อย่างมีสติ** ผ่าน `Swatinem/rust-cache` สำหรับ
Cargo dependency, `cache-from`/`cache-to` แบบ `type=gha` สำหรับ Docker layer, และ `taiki-e/install-action`
สำหรับติดตั้ง CLI tool โดยไม่ต้อง compile จาก source — ทั้งหมดคือการประยุกต์หลักการเดียวกัน: **แยกสิ่งที่เปลี่ยน
บ่อยออกจากสิ่งที่เปลี่ยนไม่บ่อย แล้ว reuse ผลลัพธ์ของสิ่งที่ไม่เปลี่ยนให้ได้มากที่สุด** ซึ่งเป็นแนวคิดเดียวกันกับ
Docker layer caching ที่ Part 96 สอนไว้ เพียงแค่ประยุกต์ใช้ในบริบทของ CI runner แทน container image

จุดที่ควรจำให้แม่นคือความแตกต่างของ**ขอบเขตความรับผิดชอบของแต่ละ job**: `lint`/`test`/`audit`/`coverage` ควรรัน
ทุกครั้งที่มีการเปลี่ยนแปลงโค้ด (push/PR) เพื่อเป็นประตูป้องกันไม่ให้โค้ดเสียเข้า `main`, `docker` ควรรันเฉพาะตอน
merge เข้า `main` สำเร็จแล้ว (ไม่ใช่จาก PR ที่ยังไม่ merge), และ `release-*` ควรรันเฉพาะตอน push tag เวอร์ชันใหม่
เท่านั้น — การควบคุมด้วย `if:`, `needs:`, และการแยก trigger (`branches:` vs `tags:`) ให้ตรงกับบทบาทของแต่ละ job
คือหัวใจของการออกแบบ CD ที่ปลอดภัยและคาดเดาผลลัพธ์ได้

สุดท้าย อย่าลืมว่า workflow ที่เขียนไว้สวยงามแค่ไหนก็ไม่มีความหมายถ้าไม่ถูกตั้งเป็น **required status check** ใน
Branch Protection Rules ของ repository (หัวข้อ 97.15) — ไฟล์ YAML กำหนด "check ทำงานอย่างไร" แต่การตัดสินใจว่า
"check ไหนต้องผ่านก่อน merge ได้" เป็นการตั้งค่าคนละชั้นที่ต้องทำคู่กันเสมอ ทีมที่เขียน CI pipeline ครบถ้วนตาม
บทนี้แล้วแต่ลืมขั้นตอนนี้ จะยังคงเห็นโค้ดที่ไม่ผ่าน CI ถูก merge เข้า `main` ได้อยู่ดี ซึ่งขัดกับเจตนารมณ์ทั้งหมด
ที่อธิบายไว้ตั้งแต่หัวข้อ 97.1

Part ถัดไป (**Part 98: Observability: Metrics ด้วย Prometheus**) จะพาเราไปดูว่าเมื่อ `library_api` ถูก deploy
ขึ้น production จริงแล้ว (ผ่าน pipeline ที่เราสร้างในบทนี้) เราจะรู้ได้อย่างไรว่าระบบทำงานถูกต้องและมี performance
ที่ดีอยู่หรือไม่ — เป็นขั้นถัดไปที่ต่อเนื่องโดยตรงจากที่ CI/CD ส่งมอบซอฟต์แวร์ขึ้นไปแล้ว

---

**Part ก่อนหน้า:** [Docker และ Containerization สำหรับ Rust](part-096-docker-containerization.md) | **Part ถัดไป:** [Observability: Metrics ด้วย Prometheus](part-098-observability-prometheus.md)
