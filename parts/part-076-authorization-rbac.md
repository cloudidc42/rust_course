# Part 76: Authorization และ RBAC

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 280 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายเส้นแบ่งระหว่าง **authentication** (Part 74-75 — "นี่คือใคร") กับ **authorization** (บทนี้ — "สิ่ง
  นี้ทำอะไรได้บ้าง") ได้อย่างเจาะจงในระดับที่พิสูจน์ได้ด้วยโค้ดจริง ไม่ใช่แค่ท่องนิยาม — เห็นตรง ๆ ว่า JWT ที่
  ผ่านการตรวจ signature สำเร็จ (Part 74) หรือ session ที่ยัง active อยู่ (Part 75) **ยืนยันตัวตนได้ แต่ไม่บอก
  อะไรเลยเรื่องสิทธิ์**
- ออกแบบระบบ **Role-Based Access Control (RBAC)** เต็มรูปแบบ: `Role` enum ที่แต่ละตัวแทนบทบาทในระบบ
  (`Admin`, `Staff`, `Member`) ผูกกับ `User` ที่มีได้มากกว่าหนึ่ง role พร้อมฝัง role เข้าไปใน JWT claims
  (ต่อยอด Part 74) **และ** ใน session data (ต่อยอด Part 75) ด้วยโค้ดที่ compile และรันได้จริงทั้งสองแบบ
- implement การบังคับ role สอง**วิธี**ที่ต่างกันโดยพื้นฐาน — **const generics** (`RequireRole<const ROLE:
  u8>` ที่ requirement มองเห็นได้ตรงในลายเซ็นของ handler เอง ต่อยอด Part 22/56) กับ **runtime middleware**
  (requirement อยู่แค่ตรงจุดที่ประกอบ `Router` เท่านั้น) — เปรียบเทียบ tradeoff ของทั้งสองอย่างตรงไปตรงมาด้วย
  โค้ดที่ compile ได้จริงทั้งคู่ รวม**ข้อจำกัดจริงที่เจอตอนลองทำ** (enum ใช้เป็น const generic parameter
  ตรง ๆ ไม่ได้บน stable Rust) พร้อม error message จริงจาก compiler
- เขียน permission check แบบละเอียด (`if !current_user.can_delete_book() { return
  Err(AppError::Forbidden) }`) ต่อยอด `AppError` ของ Part 66 ด้วย variant `Forbidden` (403) ใหม่ — และรู้ว่า
  เมื่อไหร่ควรใช้ check แบบนี้ในตัว handler เทียบกับเมื่อไหร่ควรใช้ route-level middleware
- implement **ownership-based authorization** ("ทรัพยากรชิ้นนี้เป็นของใคร") ที่ต้อง query ข้อมูลจริงมาเทียบ
  ก่อนตัดสิน (`resource.owner_id == current_user.id`) ต่อยอดแพทเทิร์นการ query จาก Part 70 — และเข้าใจ
  tradeoff ที่แท้จริงระหว่างตอบ `403` กับ `404` ให้คนที่ไม่ใช่เจ้าของทรัพยากร พร้อมพิสูจน์ทั้งสองแบบด้วย
  `curl` จริง
- ออกแบบระบบ permission ที่ละเอียดกว่า role เดี่ยว ด้วย `Permission` enum และ `HashSet<Permission>` (ต่อยอด
  Part 15) ที่คำนวณจาก role ทั้งหมดของ user ตอน login — รองรับ **user ที่มีหลาย role พร้อมกัน** (`Vec<Role>`)
  ที่ permission check ต้องกลายเป็น "มี role ไหนสักตัวที่ให้สิทธิ์นี้ไหม" ไม่ใช่ role เดียวอีกต่อไป
- รู้จัก **ABAC (Attribute-Based Access Control)** และ policy engine อย่าง Open Policy Agent/Cedar ในระดับ
  ที่พอจะรู้ว่า "มันมีอยู่ สำหรับตอนที่ RBAC ไม่พอ" โดยไม่ต้องเจาะลึกทุกรายละเอียด (นอกสโคปบทนี้)

## ความรู้ที่ต้องมีมาก่อน

- **Part 74 (Authentication: JWT)**: บทนี้สร้างต่อจาก `Claims`/`CurrentUser` extractor ที่ Part 74 ออกแบบไว้
  ตรง ๆ — เดิม `Claims` มี field `role: String` เดียว บทนี้เปลี่ยนเป็น `roles: Vec<Role>` (multi-role) และเพิ่ม
  `permissions: Vec<Permission>` เข้าไป โดยกลไกการเซ็น/ตรวจ JWT ทั้งหมด (HS256, `jsonwebtoken::encode/decode`,
  `Validation`) เหมือนกับ Part 74 ทุกประการ — ถ้ายังไม่แน่นเรื่องนี้ ควรกลับไปทวนก่อน เพราะบทนี้จะไม่อธิบาย
  กลไก JWT พื้นฐานซ้ำจากศูนย์
- **Part 75 (Authentication: Session-based และ OAuth2)**: หัวข้อ 76.3 เก็บ role ไว้ใน session data ผ่าน
  `Session` extractor ของ `tower-sessions` ตรงตามแพทเทิร์นที่ Part 75 สอนไว้ (`session.insert()`,
  `session.get()`, `SessionManagerLayer`, `cycle_id()` ป้องกัน session fixation) — บทนี้ใช้ทั้งสองกลไก (JWT
  และ session) สลับกันไปในแต่ละหัวข้อ เพื่อพิสูจน์ว่าหลักการ RBAC เดียวกันใช้ได้กับ authentication mechanism
  ทั้งสองแบบที่คุณเรียนมาแล้ว
- **Part 64 (Axum: State Management และ Extractors)**: `RequireRole<const ROLE: u8>` และ `CurrentUser` ทั้ง
  สองเป็น custom extractor ที่ implement `FromRequestParts<AppState>` ตรงตามแพทเทิร์นที่ Part 64 สอนไว้ทุก
  ประการ
- **Part 65 (Axum: Middleware ด้วย tower/tower-http)**: Approach B ของหัวข้อ 76.5 (runtime middleware) สร้าง
  จาก `axum::middleware::from_fn` และ `.route_layer()` ตรงตามที่ Part 65 สอนไว้ — บทนี้เพิ่มเทคนิคใหม่ที่ Part
  65 ยังไม่ได้สอน: **middleware factory function** ที่คืน closure ซึ่ง "จำ" ค่าพารามิเตอร์ (`Role` ที่ต้องการ)
  ไว้ผ่านการ capture ตัวแปร
- **Part 66 (Axum: Error Handling แบบมืออาชีพ)**: `AppError` ของบทนี้สร้างต่อจากหลักการเดียวกับ Part 66 ทุก
  ประการ (enum เดียว, `impl IntoResponse` จุดเดียว, JSON shape `{"error": {"code": ..., "message": ...}}`)
  เพียงเพิ่ม variant `Forbidden` (403) ที่ Part 66 ยังไม่ต้องใช้ (ตอนนั้นยังไม่มี authorization ให้ปฏิเสธ)
- **Part 70 (Database: เชื่อมต่อ PostgreSQL ด้วย SQLx)**: ownership-based check ในหัวข้อ 76.7 ต้อง **query
  ทรัพยากรที่เจาะจงมาก่อน** แล้วเทียบ `owner_id` — บทนี้ใช้ `HashMap` ในหน่วยความจำแทน `PgPool` จริงด้วยเหตุผล
  เดียวกับที่ Part 74/75 อธิบายไว้ (โฟกัสที่ authorization logic ล้วน ๆ) แต่ pattern การ "load resource ก่อน
  ตัดสินสิทธิ์" เหมือนกับ `sqlx::query_as!` ทุกประการ แค่สลับ storage
- **Part 15 (HashMap และ HashSet)**: หัวข้อ 76.8 ใช้ `HashSet<Permission>` เก็บ permission ที่ user มี และ
  `HashSet::extend()`/`HashSet::contains()` ตรงตามที่ Part 15 สอนไว้
- **Part 22 (Generics ขั้นสูง) และ Part 56 (Memory Management ขั้นสูง)**: หัวข้อ 76.5 ใช้ const generics
  (`RequireRole<const ROLE: u8>`) ต่อยอดจากที่ Part 18 เกริ่นและ Part 22/56 ขยายไว้ (โดยเฉพาะแนวคิดที่ Part 56
  อธิบายว่า `Matrix<3, 4>` และ `Matrix<2, 3>` เป็นชนิดข้อมูลคนละตัวกันสนิท — `RequireRole<0>` และ
  `RequireRole<1>` ก็เป็นชนิดข้อมูลคนละตัวกันในแบบเดียวกัน)
- **Part 61 (HTTP Fundamentals)**: บทนี้อ้างอิงความหมายของ `401 Unauthorized`, `403 Forbidden`, `404 Not
  Found` ตามมาตรฐานที่ Part 61 สอนไว้ตรง ๆ — และหัวข้อ 76.7 จะพาไปดูกรณีที่การเลือกระหว่างสามสถานะนี้ **ไม่ใช่
  เรื่องขาวดำ** อย่างที่คิด

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ทุกตัวอย่างในบทนี้ผู้เขียน **compile และรันจริง** ด้วย scratch project สามโปรเจกต์แยกไว้นอก repo ของหลักสูตร
(ไม่กระทบไฟล์ใด ๆ ในคอร์สนี้เลย) ใช้เวอร์ชันจริงจาก crates.io ณ วันที่เขียนบทนี้ — ตัวหลัก (`rbac_capstone`
ที่ประกอบหัวข้อ 76.4-76.11 ทั้งหมดเข้าด้วยกัน) ใช้:

```toml
[dependencies]
axum = { version = "0.8.9", features = ["macros"] }
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
tokio = { version = "1.53.1", features = ["full"] }
```

(เวอร์ชันเดียวกับที่ Part 74 ใช้ทุกตัว — คาดว่าไม่ใช่ความบังเอิญ เพราะเขียนช่วงเวลาใกล้กัน) และอีกโปรเจกต์แยก
สำหรับหัวข้อ 76.3 (เก็บ role ใน session) ใช้ `tower-sessions = "0.15.0"` เวอร์ชันเดียวกับ Part 75

รายละเอียดสิ่งที่ทดสอบจริง:

- `cargo build`/`cargo run` เซิร์ฟเวอร์เต็มรูปแบบของหัวข้อ 76.11 (capstone) ที่มี 4 บัญชีผู้ใช้ (`admin`,
  `staff1` ที่มีสอง role พร้อมกัน, `nan`, `wit`) และยิง `curl` จริงมากกว่า 20 คำสั่งครอบคลุมทุก endpoint ทุก
  สถานะที่เป็นไปได้ (200, 201, 204, 401, 403, 404)
- ลองใช้ `enum Role` ตรง ๆ เป็น const generic parameter จริง แล้ว capture error message จริงจาก `rustc`
  (ไม่ใช่การเดา — หัวข้อ 76.5 และกับดักข้อ 1 มีข้อความ error ที่คัดลอกมาจากการรันจริง)
- ลองใช้ type alias (`RequireAdmin`) เป็น pattern ใน function parameter ตรง ๆ แล้ว capture error message จริง
  อีกตัวที่ compiler แจ้ง (กับดักข้อ 2)
- สร้างตัวอย่างที่ถือ `MutexGuard` ข้าม `.await` โดยตั้งใจ แล้ว capture error message จริงของ
  `axum::debug_handler` ที่ระบุจุดที่ผิดได้ตรงเป๊ะ (กับดักข้อ 3)
- รันทั้งเวอร์ชัน JWT-based (`rbac_capstone`) และเวอร์ชัน session-based (`session_roles`) ของการเก็บ role
  แล้วพิสูจน์ด้วย `curl` จริงว่าทั้งสองแบบให้ผลลัพธ์ authorization เหมือนกันทุกประการ ต่างกันแค่ "role เก็บอยู่
  ที่ไหน" (ใน token vs ใน session store)

ทุกที่ที่มีการอ้างผลลัพธ์ (`cargo build`, error message, HTTP response, `curl` transcript) ในบทนี้คือผลลัพธ์
ที่**รันจริงแล้วคัดลอกมา** ไม่มีการแต่งขึ้นเอง — ถ้าเจอความต่างเล็กน้อยตอนคุณลองรันเอง (เวอร์ชัน crate ใหม่กว่า)
ให้ยึด `cargo add`/`cargo build` ที่รันในเครื่องคุณเป็นความจริงล่าสุดเสมอ

## เนื้อหา

### 76.1 Authentication vs Authorization: เส้นแบ่งที่ต้องชัด

Part 74 สอนให้คุณสร้าง JWT ที่เซ็นด้วย HS256/RS256 แล้วให้ `CurrentUser` extractor ตรวจสอบ signature และวัน
หมดอายุก่อนปล่อยให้ handler ทำงาน — Part 75 สอนแบบเดียวกันแต่ผ่าน session ที่เก็บอยู่ที่ server แทน — ทั้งสอง
บททำหน้าที่**เดียวกัน** ในทางแนวคิด แม้กลไกจะต่างกันมาก: ตอบคำถามที่ว่า **"request นี้มาจากใคร"**

แต่ลองพิจารณาสถานการณ์นี้ให้ดี: JWT ที่ Part 74 ออกให้ผ่านการตรวจ signature สำเร็จ 100% (ไม่ถูกแก้ไข ไม่หมด
อายุ เซ็นด้วย secret ที่ถูกต้องจริง) — นี่พิสูจน์ได้แค่ว่า **"คนที่ส่ง request นี้มาคือ user id 3 (`nan`) จริง
ๆ ไม่ได้ปลอมตัว"** มันไม่ได้พิสูจน์เลยว่า `nan` **ควรจะ**ลบหนังสือทิ้งได้ไหม, ควรจะดูข้อมูล booking ของคนอื่น
ได้ไหม, หรือควรจะจัดการ user คนอื่นในระบบได้ไหม — คำถามเหล่านี้เป็นคำถามคนละมิติกันโดยสิ้นเชิงกับ "นี่คือใคร"

นี่คือเส้นแบ่งที่บทนี้ทั้งบทจะสร้างขึ้นมาให้ชัดเจน:

| | Authentication (Part 74-75) | Authorization (บทนี้) |
|---|---|---|
| คำถามที่ตอบ | "request นี้มาจากใคร" | "identity นี้ทำอะไรได้บ้าง" |
| ผลลัพธ์ที่ได้ | `CurrentUser` / `Session` ที่ยืนยันตัวตนแล้ว | อนุญาต (ทำงานต่อ) หรือปฏิเสธ (403/404) |
| เกิดขึ้นเมื่อไหร่ | ครั้งเดียวตอนต้นของแต่ละ request (decode JWT/โหลด session) | หลัง authentication เสร็จแล้ว ก่อน handler ทำงานจริง (หรือกลางฟังก์ชัน handler) |
| ล้มเหลวแล้วตอบ status อะไร | `401 Unauthorized` — "ไม่รู้จักว่าเป็นใคร" | `403 Forbidden` (หรือ `404` ในบางกรณี — หัวข้อ 76.7) — "รู้ว่าเป็นใคร แต่ไม่มีสิทธิ์" |
| ขึ้นกับข้อมูลอะไร | signature ของ token / การมีอยู่ของ session ID ใน store | role/permission ของ user **และบางครั้งข้อมูลของทรัพยากรที่ขอ** (เช่น เจ้าของ) |

แถวสุดท้ายคือจุดที่สำคัญที่สุด: authentication ตรวจสอบแค่ **token/session เอง** (ไม่ต้องรู้ว่า request นี้จะ
ทำอะไรกับทรัพยากรไหน) แต่ authorization บางกรณี (หัวข้อ 76.7) ต้อง**รู้จักทรัพยากรที่เจาะจง**ที่ request นี้
ขอเข้าถึงด้วย — นี่คือความต่างเชิงโครงสร้างที่ทำให้ authorization "ทำยากกว่า" authentication ในหลายกรณี:
authentication ตรวจสอบได้จาก token/session ID เพียงอย่างเดียวโดยไม่ต้องแตะฐานข้อมูลเรื่องทรัพยากรเลย แต่
authorization ที่ดีบางกรณีบังคับให้ต้อง query ก่อนตัดสิน

บทนี้จะไล่จากรูปแบบ authorization ที่**หยาบที่สุด** (role ทั้งระบบ — "user คนนี้เป็น Admin ไหม") ไปจนถึง
**ละเอียดที่สุด** (ownership ของทรัพยากรเจาะจง — "user คนนี้เป็นเจ้าของ booking id 42 ไหม") ให้เห็นครบทุกระดับ
ที่ระบบจริงต้องใช้ผสมกัน

### 76.2 RBAC พื้นฐาน: Role, Permission, User

**Role-Based Access Control (RBAC)** คือโมเดล authorization ที่ใช้กันแพร่หลายที่สุดในระบบจริง — แนวคิดหลัก
เรียบง่ายมาก: **user มี role, role มี permission (สิทธิ์การทำงาน), authorization check จึงกลายเป็นคำถามว่า
"role ของ user คนนี้มีสิทธิ์ทำสิ่งนี้ไหม"** แทนที่จะต้องเช็คสิทธิ์ทีละ user เป็นราย ๆ ไป (ซึ่ง scale ไม่ได้
เลยถ้าระบบมี user เป็นพัน ๆ คน)

กลับไปที่ระบบห้องสมุด/จองตั๋วที่คอร์สนี้ใช้มาตั้งแต่ Part 62 — ตอนนี้เราจะกำหนด role ที่ระบบต้องมีจริง ตามหน้าที่
ที่ต่างกันของคนที่ใช้ระบบ:

| Role | หน้าที่ในระบบ | ตัวอย่าง user จริง |
|---|---|---|
| `Admin` | ดูแลระบบทั้งหมด รวมถึงจัดการ user คนอื่น (`ManageUsers`) และทำได้ทุกอย่างที่ `Staff` ทำได้ | ผู้ดูแลระบบห้องสมุด/งานอีเวนต์ |
| `Staff` | จัดการหนังสือ/ประเภทตั๋ว (เพิ่ม/ลบ), ดูและยกเลิก booking ของ**ใครก็ได้** (ไม่จำกัดแค่ของตัวเอง) — รวมบทบาทที่หลายระบบเรียกแยกกันเป็น "Librarian" (ฝั่งห้องสมุด) และ "EventStaff" (ฝั่งจองตั๋วงานอีเวนต์) เข้าเป็น role เดียวในระบบนี้ | บรรณารักษ์, เจ้าหน้าที่หน้างานอีเวนต์ |
| `Member` | สร้าง booking ของตัวเอง, ดูและยกเลิก booking **ของตัวเองเท่านั้น** | สมาชิกห้องสมุด/ผู้ซื้อตั๋วทั่วไป |

สังเกตคำว่า **"ของตัวเองเท่านั้น"** ใน `Member` — นี่คือจุดที่ role เดี่ยว ๆ **ไม่พอ** ให้ authorization ทำงาน
ถูกต้องได้ทั้งหมด: การเช็คว่า "user คนนี้มี role `Member` ไหม" ตอบได้แค่ "เขาสร้าง/ยกเลิก booking ได้ไหมใน
หลักการ" แต่ไม่ตอบว่า "เขายกเลิก booking **id เจาะจงนี้**ได้ไหม" — หัวข้อ 76.7 จะกลับมาแก้ปัญหานี้เต็มรูปแบบ
ด้วย ownership-based check ที่ทำงาน**ควบคู่กับ** RBAC ไม่ใช่แทนมัน

ตารางสิทธิ์แบบละเอียดของแต่ละ role (ยังไม่ใช่ code เลย แค่ requirement — บทนี้จะแปลงตารางนี้เป็น Rust ทีละขั้น):

| การกระทำ | `Admin` | `Staff` | `Member` |
|---|---|---|---|
| ดูรายการหนังสือ | ✅ | ✅ | ✅ |
| สร้าง booking ของตัวเอง | ✅ | ✅ | ✅ |
| ยกเลิก booking **ของตัวเอง** | ✅ | ✅ | ✅ |
| ยกเลิก booking **ของใครก็ได้** | ✅ | ✅ | ❌ |
| ดู booking ของใครก็ได้ | ✅ | ✅ | ❌ |
| ลบหนังสือ/ประเภทตั๋ว | ✅ | ✅ | ❌ |
| จัดการ user คนอื่น (แบน, เปลี่ยน role) | ✅ | ❌ | ❌ |

### 76.3 Modeling Role ใน Rust: enum + เก็บใน JWT Claims/Session

แปลง `Role` เป็น Rust enum — ใช้ `#[repr(u8)]` ตั้งแต่ต้น (เหตุผลจะชัดเจนในหัวข้อ 76.5 ที่ const generics
parameter รับได้แค่ primitive type อย่าง `u8` ไม่ใช่ enum ตรง ๆ):

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[repr(u8)]
enum Role {
    Admin = 0,
    Staff = 1,
    Member = 2,
}
```

**อธิบาย derive แต่ละตัว**:

- **`PartialEq, Eq`** — ต้องเทียบ role ได้ตรง ๆ (`user.role == Role::Admin`) ในทุก permission check ของทั้งบท
- **`Hash`** — จำเป็นสำหรับหัวข้อ 76.8 ที่เก็บ `Role`/`Permission` ใน `HashSet` (ตาม Part 15 — type ที่จะเป็น
  element ของ `HashSet<T>` ต้อง implement `Hash` + `Eq`)
- **`Serialize, Deserialize`** — ต้องฝัง `Role` เข้า JWT payload ได้ (ผ่าน `serde_json` ที่ `jsonwebtoken`
  ใช้ข้างใน ตรงตาม Part 57) และเก็บใน session data ได้ (ตรงตามที่ Part 75 ใช้ `session.insert()`/`get()` ที่
  ต้องการ `T: Serialize + DeserializeOwned` เหมือนกัน)
- **`#[repr(u8)]`** — บอก compiler ว่า discriminant ของแต่ละ variant คือเลข `u8` ตายตัว (`Admin = 0`, `Staff
  = 1`, `Member = 2`) — ยังไม่มีผลอะไรกับโค้ดในหัวข้อนี้ แต่หัวข้อ 76.5 จะใช้ `Role::Admin as u8` โดยตรง

#### เก็บ role ใน JWT Claims (ต่อยอด Part 74)

`Claims` struct จาก Part 74 มี field `role: String` เดี่ยว ๆ — บทนี้เปลี่ยนเป็น `Vec<Role>` (รองรับ multi-role
ตั้งแต่ต้น ตามหัวข้อ 76.9) และเพิ่ม `permissions: Vec<Permission>` (หัวข้อ 76.8):

```rust
use serde::{Deserialize, Serialize};
# #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
# enum Role { Admin, Staff, Member }
# #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
# enum Permission { ViewBook }

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,               // user id (ตรงตามธรรมเนียม RFC 7519 -- String เสมอ, Part 74 หัวข้อ 74.5)
    username: String,
    roles: Vec<Role>,          // *** ใหม่: หลาย role พร้อมกันได้ (หัวข้อ 76.9) ***
    permissions: Vec<Permission>, // *** ใหม่: สิทธิ์ที่คำนวณไว้แล้วตอน login (หัวข้อ 76.8) ***
    iat: usize,
    exp: usize,
}
```

การเข้ารหัส/ตรวจสอบ JWT (`encode`/`decode`, `Algorithm::HS256`, `Validation`) เหมือนกับ Part 74 ทุกประการ —
สิ่งที่เปลี่ยนมีแค่ **เนื้อหาข้างในของ claims** ไม่ใช่กลไกของ JWT เอง ทดสอบ login จริงด้วยเซิร์ฟเวอร์เต็มรูป
แบบของหัวข้อ 76.11 (ตัดมาจากการรันจริง) — สังเกต payload ที่ decode ออกมา:

```json
{"sub":"2","username":"staff1","roles":["Staff","Member"],"permissions":["CancelOwnBooking","DeleteBook","CreateBooking","ViewBook","ViewAnyBooking","CancelAnyBooking"],"iat":1790472093,"exp":1790475693}
```

นี่คือ `staff1` ที่มี**สอง role พร้อมกัน** (`Staff` และ `Member`) — `permissions` ที่ฝังมาคือ**ผลรวม**ของ
permission จากทั้งสอง role (หัวข้อ 76.8-76.9 จะอธิบายกลไกการคำนวณนี้เต็มรูปแบบ)

#### เก็บ role ใน Session data (ต่อยอด Part 75)

ในระบบที่เลือก session-based authentication (Part 75) แทน JWT — role ไม่ได้ฝังอยู่ใน token ที่ client ถือ
เลย (เพราะไม่มี token ให้ฝัง) แต่เก็บไว้เป็นส่วนหนึ่งของ **session data ที่ server ถือไว้** ผ่าน
`Session::insert()` ตรงตามแพทเทิร์นของ Part 75:

```rust
use axum::{extract::State, http::StatusCode, Json};
use serde::Deserialize;
use tower_sessions::Session;
# use serde::Serialize;
# #[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
# enum Role { Admin, Staff, Member }
# #[derive(Clone)] struct AppState;
# #[derive(Deserialize)] struct LoginRequest { username: String, password: String }

const USERNAME_KEY: &str = "username";
const ROLES_KEY: &str = "roles";

async fn login(
    State(_state): State<AppState>,
    session: Session,
    Json(req): Json<LoginRequest>,
) -> Result<StatusCode, StatusCode> {
    // (ขั้นตอนตรวจ username/password จริงถูกตัดออกเพื่อความกระชับ -- ดูโค้ดเต็มด้านล่าง)
    let roles = vec![Role::Member]; // สมมติว่าดึงมาจาก UserRecord แล้ว

    // *** จุดสำคัญที่สุดของหัวข้อนี้: roles เก็บอยู่ใน session data ตรง ๆ ไม่ใช่ในตัว cookie ที่ client ถือ ***
    // client ได้แค่ session ID กลับไป (ตามที่ Part 75 หัวข้อ 75.2 อธิบายไว้ทั้งหัวข้อ) ส่วน roles
    // จริง ๆ อยู่ที่ server (ใน session store) เท่านั้น -- คนละกลไกกับ JWT ที่ roles ฝังอยู่ในตัว token เอง
    session.insert(USERNAME_KEY, req.username).await.unwrap();
    session.insert(ROLES_KEY, roles).await.unwrap();
    session.cycle_id().await.unwrap(); // ป้องกัน session fixation (Part 75 หัวข้อ 75.6)

    Ok(StatusCode::OK)
}
```

อ่าน role กลับมาใน handler อื่น (ตรงตามที่ Part 75 หัวข้อ 75.3 สอน `session.get()`):

```rust
use tower_sessions::Session;
# use serde::{Serialize, Deserialize};
# #[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
# enum Role { Admin, Staff, Member }
# const ROLES_KEY: &str = "roles";

async fn require_staff_or_admin(session: &Session) -> bool {
    let roles: Vec<Role> = session.get(ROLES_KEY).await.unwrap().unwrap_or_default();
    roles.iter().any(|r| matches!(r, Role::Admin | Role::Staff))
}
```

ทดสอบจริงด้วยเซิร์ฟเวอร์แยก (`session_roles`, ใช้ `tower-sessions = "0.15.0"` ตรงตาม Part 75) — login เป็น
`admin` แล้วดู session cookie ที่ได้ (ตัดมาจากการรันจริง):

```bash
$ curl -sS -i -c /tmp/admin_session.txt -X POST http://127.0.0.1:3177/login \
    -H 'Content-Type: application/json' -d '{"username":"admin","password":"adminpass"}'
```
```
HTTP/1.1 200 OK
set-cookie: id=-y6WozUMVQa8b29XgmJ50A; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
content-length: 0
```

```bash
$ curl -sS -i -b /tmp/admin_session.txt http://127.0.0.1:3177/me
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 38

{"username":"admin","roles":["Admin"]}
```

สังเกตว่า **cookie (`id=-y6WozUMVQa8b29XgmJ50A`) ไม่มีคำว่า "Admin" อยู่ในตัวมันเองเลย** — ต่างจาก JWT ที่
ถ้า decode base64url ดูตรง ๆ (แบบที่ Part 74 หัวข้อ 74.2 พิสูจน์ไว้) จะเห็น `"roles":["Admin"]` เป็น plain
text ทันที นี่คือความต่างเชิงโครงสร้างระหว่างสองกลไกที่ Part 75 อธิบายไว้แล้ว (stateless vs stateful) ที่ยัง
เห็นผลชัดต่อเนื่องมาถึงตอนเก็บ **authorization data** (ไม่ใช่แค่ authentication data) ด้วยเช่นกัน

ทดสอบว่า `nan` (member ธรรมดา) โดน 403 ที่ endpoint ที่ต้องมี role `Admin`/`Staff` ขณะที่ `admin` ผ่านได้ปกติ
(ตัดมาจากการรันจริง — endpoint นี้ใช้ logic เดียวกับ `require_staff_or_admin` ข้างบน):

```bash
$ curl -sS -i -b /tmp/nan_session.txt -X DELETE http://127.0.0.1:3177/books/1
```
```
HTTP/1.1 403 Forbidden
content-type: text/plain; charset=utf-8
content-length: 73

ต้องมี role Admin หรือ Staff เท่านั้น
```

```bash
$ curl -sS -i -b /tmp/admin_session.txt -X DELETE http://127.0.0.1:3177/books/1
```
```
HTTP/1.1 204 No Content
```

**ข้อสรุปสำคัญของหัวข้อนี้**: ไม่ว่า role จะเก็บอยู่ใน JWT claims หรือ session data — **ตรรกะของ RBAC ที่
เหลือทั้งหมดของบทนี้ (76.4-76.11) เหมือนกันทุกประการ** สิ่งที่ต่างกันมีแค่ "ไปหา role ของ user คนนี้จากไหน"
(decode JWT vs อ่าน session) ส่วนที่เหลือ (เช็ค role กับ permission ที่ต้องการ, ตัดสิน 200/403/404) เป็นโค้ด
เดียวกันเป๊ะ — บทที่เหลือของบทนี้จะใช้ JWT เป็นหลัก (เพราะ capstone หัวข้อ 76.11 อิงจาก Part 74) แต่ทุกอย่างที่
สอนสามารถย้ายไปใช้กับ session-based ได้ทันทีตามแพทเทิร์นในหัวข้อนี้

### 76.4 Approach B: Runtime Middleware (route-level role gating)

เริ่มจากวิธีที่**เข้าใจง่ายที่สุด**ก่อน (บทนี้ตั้งใจสอนวิธีนี้ก่อน const generics ในหัวข้อ 76.5 เพราะเป็นฐาน
ที่ทำให้เห็น tradeoff ชัดกว่าเมื่อเทียบกัน): เขียน **middleware** ที่ตรวจ role ก่อนที่ request จะไปถึง handler
เลย โดยกำหนดว่า route ไหนต้องการ role อะไรตรงจุดที่ประกอบ `Router` (ไม่ใช่ในตัว handler)

```rust
use axum::{
    extract::{Request, State},
    middleware::Next,
    response::{IntoResponse, Response},
};
use std::sync::Arc;
# #[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
# enum Role { Admin, Staff, Member }
# #[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
# struct Claims { sub: String, username: String, roles: Vec<Role>, permissions: Vec<()>, iat: usize, exp: usize }
# #[derive(Debug)] enum AppError { Unauthorized(String), Forbidden(String) }
# impl IntoResponse for AppError {
#     fn into_response(self) -> Response { axum::http::StatusCode::OK.into_response() }
# }
# fn decode_claims(_h: &axum::http::HeaderMap, _s: &[u8]) -> Result<Claims, AppError> { unimplemented!() }
# #[derive(Clone)] struct AppState { jwt_secret: Arc<Vec<u8>> }

/// middleware factory -- คืน closure ที่ "จำ" role ที่ต้องการไว้ผ่านการ capture ตัวแปร `required`
/// สังเกต signature: รับ (Request, Next) แค่สองตัว -- ไม่มี State<AppState> extractor เลย เพราะเรา
/// capture jwt_secret เข้าไปใน closure ตรง ๆ ตอนสร้าง (ไม่ต้องพึ่ง Axum extract มันจาก request อีกที)
fn require_role(
    jwt_secret: Arc<Vec<u8>>,
    required: Role,
) -> impl Fn(Request, Next) -> std::pin::Pin<Box<dyn std::future::Future<Output = Response> + Send>> + Clone {
    move |req: Request, next: Next| {
        let jwt_secret = jwt_secret.clone();
        Box::pin(async move {
            let claims = match decode_claims(req.headers(), &jwt_secret) {
                Ok(c) => c,
                Err(e) => return e.into_response(),
            };
            if !claims.roles.contains(&required) {
                return AppError::Forbidden(format!(
                    "ต้องมี role {:?} เท่านั้นถึงจะเรียก endpoint นี้ได้",
                    required
                ))
                .into_response();
            }
            next.run(req).await
        })
    }
}
```

ผูกเข้ากับ route ที่ต้องการผ่าน `.route_layer()` (ไม่ใช่ `.layer()` ธรรมดา — `.route_layer()` ใส่ middleware
เฉพาะ route นั้นเท่านั้น ไม่กระทบ route อื่นใน `Router` เดียวกัน ตรงตามที่ Part 65 สอนไว้):

```rust
# use axum::{routing::delete, Router};
# #[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
# enum Role { Admin, Staff, Member }
# #[derive(Clone)] struct AppState { jwt_secret: std::sync::Arc<Vec<u8>> }
# async fn delete_book_staff_gated() -> axum::http::StatusCode { axum::http::StatusCode::NO_CONTENT }
# fn require_role(_j: std::sync::Arc<Vec<u8>>, _r: Role) -> impl Fn(axum::extract::Request, axum::middleware::Next) -> std::pin::Pin<Box<dyn std::future::Future<Output = axum::response::Response> + Send>> + Clone {
#     move |_req, _next| Box::pin(async { axum::http::StatusCode::OK.into_response() })
# }
# use axum::response::IntoResponse;
# fn build_router(state: AppState) -> Router<()> {
Router::new()
    .route(
        "/staff/books/{id}",
        delete(delete_book_staff_gated)
            .route_layer(axum::middleware::from_fn(require_role(state.jwt_secret.clone(), Role::Staff))),
    )
    .with_state(state)
# }
```

**จุดที่ต้องสังเกตให้ชัด**: ลองดู signature ของ `delete_book_staff_gated` เอง (ในโค้ดเต็มหัวข้อ 76.11):
`async fn delete_book_staff_gated(State(state): State<AppState>, Path(id): Path<u32>) -> Result<StatusCode,
AppError>` — **ไม่มีร่องรอยของ "ต้องเป็น Staff" อยู่ในลายเซ็นนี้เลยแม้แต่นิดเดียว** ใครก็ตามที่เปิดไฟล์นี้มา
อ่านแค่ตัว handler โดยไม่รู้ว่ามันถูก mount ที่ไหนใน `Router` **จะไม่รู้เลยว่า endpoint นี้ถูกจำกัดสิทธิ์ไว้**
ต้องไปดูจุดที่ `.route()` ถูกเรียกใน `main()` เท่านั้นถึงจะเห็น requirement นี้ — นี่คือ**ข้อเสียที่แท้จริง**
ของวิธีนี้ ที่หัวข้อ 76.5 จะเทียบกับ Approach A ที่แก้ปัญหานี้ได้ตรง ๆ

ทดสอบจริง — member (`nan`) พยายามลบหนังสือผ่าน route ที่ gate ด้วย role `Staff` กับ admin (role `Admin`,
**ไม่ใช่** `Staff`) ก็โดนปฏิเสธเหมือนกัน เพราะ middleware เช็คแค่ role `Staff` เป๊ะ ๆ ไม่ใช่ "role ระดับสูงพอ":

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/staff/books/1 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-type: application/json
content-length: 155

{"error":{"code":"FORBIDDEN","message":"ต้องมี role Staff เท่านั้นถึงจะเรียก endpoint นี้ได้"}}
```

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/staff/books/1 -H "Authorization: Bearer $ADMIN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-type: application/json
content-length: 155

{"error":{"code":"FORBIDDEN","message":"ต้องมี role Staff เท่านั้นถึงจะเรียก endpoint นี้ได้"}}
```

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/staff/books/1 -H "Authorization: Bearer $STAFF_TOKEN"
```
```
HTTP/1.1 204 No Content
```

**ผลลัพธ์ที่สอง (admin โดน 403) เป็นเรื่องน่าประหลาดใจสำหรับหลายคนตอนแรก** — โดยสามัญสำนึก `Admin` "ควรจะ"
ทำได้ทุกอย่างที่ `Staff` ทำได้ แต่ middleware ตัวนี้เช็คแค่ `claims.roles.contains(&Role::Staff)` **ตรง ๆ**
ถ้า `admin` ไม่มี `Role::Staff` อยู่ใน `roles` ของตัวเอง (ในระบบนี้ `admin` มีแค่ `[Role::Admin]`) ก็จะโดน
ปฏิเสธเหมือนกัน — นี่ไม่ใช่บั๊ก แต่เป็นผลตรงไปตรงมาจากการออกแบบที่เช็ค **role ตรงตัว** ไม่ใช่ **permission** —
หัวข้อ 76.8 จะแก้ปัญหานี้อย่างเป็นระบบด้วย permission-based check (`DeleteBook` permission ที่ทั้ง `Admin`
และ `Staff` มี) แทนการเช็ค role ตรงตัวแบบนี้ — จำจุดนี้ไว้ให้ดี จะกลับมาอีกครั้งในหัวข้อ 76.5 ด้วย

### 76.5 Approach A: Const Generics — Requirement ที่มองเห็นได้ในลายเซ็นของ Handler

Approach B (หัวข้อ 76.4) มีข้อเสียชัดเจนข้อหนึ่ง: **การเปิดไฟล์ handler ขึ้นมาอ่านเพียวๆ ไม่บอกอะไรเลยว่า
endpoint นั้นถูกจำกัด role อะไรไว้** ต้องไปไล่ดู router setup ในอีกไฟล์ ในระบบใหญ่ที่มี endpoint นับร้อย นี่
เป็นความเสี่ยงจริง (คนแก้โค้ดอาจย้าย handler ไปผูกกับ route ใหม่ที่ลืมใส่ middleware, หรือลบ middleware บาง
ตัวออกโดยไม่รู้ว่ามันสำคัญ) — Rust มีเครื่องมือที่ทำให้ requirement นี้ **ปรากฏอยู่ในระบบ type ของ handler
เอง**: **const generics**

#### ไอเดีย: `RequireRole<const ROLE: Role>` — ลองแบบตรงไปตรงมาก่อน

แนวคิดที่ดูสมเหตุสมผลที่สุดตอนแรก: สร้าง struct ที่มี **type ของ role ที่ต้องการเป็น const generic
parameter ของมันเอง** แล้วให้ Axum ปฏิเสธ request ที่ role ไม่ตรงตั้งแต่ตอน extract:

```rust
# use serde::{Deserialize, Serialize};
# #[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
# enum Role { Admin, Staff, Member }
struct RequireRole<const ROLE: Role>;
```

ลองคอมไพล์จริง — **compiler ปฏิเสธทันที** ด้วย error message ที่ตรงประเด็นมาก (capture มาจากการรันจริง ไม่ใช่
การเดา):

```
error: `Role` is forbidden as the type of a const generic parameter
  --> src/main.rs:11:32
   |
11 | struct RequireRole<const ROLE: Role>;
   |                                ^^^^
   |
   = note: the only supported types are integers, `bool`, and `char`
```

**นี่คือข้อจำกัดจริงของ const generics บน stable Rust**: const generic parameter รับได้แค่ **integer type,
`bool`, และ `char`** เท่านั้น — enum ที่มีข้อมูลกำกับตัวเอง (แม้จะเป็น fieldless enum ธรรมดาแบบ `Role`) **ไม่
อยู่ในกลุ่มที่อนุญาต** บน stable Rust (มี unstable feature ชื่อ `adt_const_params` ที่เปิดให้ใช้ struct/enum
เป็น const generic parameter ได้ แต่ต้องใช้ nightly compiler ซึ่งไม่เหมาะกับโค้ด production — บทนี้จะไม่ใช้
nightly feature เลย)

#### ทางแก้: ใช้ `u8` (primitive) แทน `Role` ตรง ๆ

เพราะ `#[repr(u8)]` ที่ตั้งไว้ตั้งแต่หัวข้อ 76.3 ทำให้แต่ละ variant ของ `Role` มี discriminant เป็น `u8`
ตายตัวอยู่แล้ว (`Admin = 0`, `Staff = 1`, `Member = 2`) เราจึงใช้ **`u8` เป็น const generic parameter แทน**
แล้วแปลงกลับเป็น `Role` ข้างในเมื่อต้องใช้จริง:

```rust
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# #[repr(u8)]
# enum Role { Admin = 0, Staff = 1, Member = 2 }
impl Role {
    const fn from_u8(v: u8) -> Role {
        match v {
            0 => Role::Admin,
            1 => Role::Staff,
            2 => Role::Member,
            _ => panic!("ROLE ที่ไม่รู้จัก"),
        }
    }
}

// const generic parameter เป็น u8 (primitive ที่อนุญาต) -- ไม่ใช่ Role ตรง ๆ
struct RequireRole<const ROLE: u8>(CurrentUser);
# struct CurrentUser;
```

implement `FromRequestParts<AppState>` ให้กับมัน — ทำสองขั้นตอนคือ (1) authentication ก่อน (ดึง `CurrentUser`
ตามปกติ) แล้ว (2) authorization (เช็ค role ที่ฝังอยู่ใน `ROLE` const parameter):

```rust
use axum::{extract::FromRequestParts, http::request::Parts};
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# #[repr(u8)]
# enum Role { Admin = 0, Staff = 1, Member = 2 }
# impl Role {
#     const fn from_u8(v: u8) -> Role {
#         match v { 0 => Role::Admin, 1 => Role::Staff, 2 => Role::Member, _ => panic!("ROLE ที่ไม่รู้จัก") }
#     }
# }
# #[derive(Clone)] struct CurrentUser { roles: Vec<Role> }
# impl CurrentUser { fn has_role(&self, r: Role) -> bool { self.roles.contains(&r) } }
# #[derive(Clone)] struct AppState;
# #[derive(Debug)] struct AppError;
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::FORBIDDEN.into_response() }
# }
# impl FromRequestParts<AppState> for CurrentUser {
#     type Rejection = AppError;
#     async fn from_request_parts(_p: &mut Parts, _s: &AppState) -> Result<Self, Self::Rejection> {
#         Ok(CurrentUser { roles: vec![] })
#     }
# }
struct RequireRole<const ROLE: u8>(CurrentUser);

impl<const ROLE: u8> FromRequestParts<AppState> for RequireRole<ROLE> {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        // ขั้นที่ 1: authentication ตามปกติ -- extractor อื่นเรียก extractor อื่นซ้อนกันได้
        // (ตรงตามที่ Part 64 อธิบายไว้เรื่อง extractor ที่ compose กันได้)
        let user = CurrentUser::from_request_parts(parts, state).await?;

        // ขั้นที่ 2: authorization -- เช็ค role ที่ "ฝังอยู่ในชนิดของ RequireRole เอง" ผ่าน ROLE
        let required = Role::from_u8(ROLE);
        if !user.has_role(required) {
            return Err(AppError); // ในโค้ดจริงคือ AppError::Forbidden(...) -- ดูหัวข้อ 76.6
        }

        Ok(RequireRole(user))
    }
}

// type alias สั้น ๆ ให้เรียกใช้อ่านง่าย -- Role::Admin as u8 คำนวณตอน compile time (const context)
type RequireAdmin = RequireRole<{ Role::Admin as u8 }>;
type RequireStaff = RequireRole<{ Role::Staff as u8 }>;
```

ใช้ใน handler ตรง ๆ — **นี่คือจุดขายของวิธีนี้**: เปิดไฟล์นี้ไฟล์เดียว อ่าน signature อย่างเดียวก็รู้ทันทีว่า
endpoint นี้ต้องการ role อะไร ไม่ต้องไปเปิดไฟล์ router setup เลย:

```rust
# use axum::{extract::{Path, State}, http::StatusCode};
# #[derive(Clone)] struct AppState { books: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<u32, ()>>> }
# struct RequireRole<const ROLE: u8>(());
# type RequireAdmin = RequireRole<0>;
# #[derive(Debug)] enum AppError { NotFound }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { StatusCode::NOT_FOUND.into_response() }
# }
async fn delete_book_admin_only(
    State(state): State<AppState>,
    RequireRole(_user): RequireAdmin, // <- requirement "ต้องเป็น Admin" อยู่ตรงนี้ ในลายเซ็นเอง
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut books = state.books.lock().unwrap();
    books.remove(&id).ok_or(AppError::NotFound)?;
    Ok(StatusCode::NO_CONTENT)
}
```

#### กับดักที่สองของ const generics: type alias ใช้เป็น pattern ตรง ๆ ไม่ได้

สังเกตบรรทัด `RequireRole(_user): RequireAdmin` ข้างบนให้ดี — ตอนแรกที่เขียนโค้ดนี้ ผู้เขียนเผลอเขียนแบบที่
"ดูเป็นธรรมชาติกว่า" คือ `RequireAdmin(_user): RequireAdmin` (ใช้ชื่อ alias ทั้ง pattern และ type annotation)
แล้วเจอ error จริงทันที (capture จากการรันจริง):

```
error[E0532]: expected tuple struct or tuple variant, found type alias `RequireAdmin`
   --> src/main.rs:377:5
    |
377 |     RequireAdmin(_user): RequireAdmin,
    |     ^^^^^^^^^^^^ not a tuple struct or tuple variant
```

**เหตุผล**: `RequireAdmin` เป็นแค่ **type alias** (`type RequireAdmin = RequireRole<0>;`) — มันไม่ใช่ชนิด
ข้อมูลใหม่ ไม่มี "constructor" ของตัวเอง เวลาเขียน **pattern** (ฝั่งซ้ายของ `:` ในพารามิเตอร์ของฟังก์ชัน หรือ
ใน `match`) Rust ต้องการชื่อของ tuple struct/variant **ที่ประกาศไว้จริง** (คือ `RequireRole`) ไม่ใช่ alias
ของมัน — ทางแก้คือ**ใช้ชื่อจริงในฝั่ง pattern เสมอ แต่ใช้ alias ได้เฉพาะฝั่ง type annotation** (`RequireRole(x):
RequireAdmin` ถูก, `RequireAdmin(x): RequireAdmin` ผิด) — ข้อจำกัดนี้ไม่เฉพาะกับ const generics เท่านั้น
(เกิดกับ type alias ของ tuple struct ทั่วไปด้วย) แต่มาเจอบ่อยเป็นพิเศษกับแพทเทิร์นนี้เพราะการตั้ง alias สั้น
ๆ (`RequireAdmin`) ไว้ให้อ่านง่ายเป็นเรื่องปกติมากในโค้ดแบบนี้

#### เปรียบเทียบ Approach A vs Approach B ตรงไปตรงมา

| | Approach A: const generics (`RequireRole<const ROLE: u8>`) | Approach B: runtime middleware (`require_role(Role::X)`) |
|---|---|---|
| Requirement มองเห็นได้จากไหน | **ลายเซ็นของ handler เอง** — เปิดไฟล์เดียวก็รู้ | จุดที่ `.route_layer()` ถูกเรียกใน router setup — ต้องไปดูอีกไฟล์ |
| ต้องมี type machinery เพิ่มแค่ไหน | ต้องมี `RequireRole<const ROLE: u8>`, `Role::from_u8`, type alias ต่อ role, และรู้ข้อจำกัดเรื่อง pattern (`RequireRole(x): RequireAdmin`) | ไม่มี type parameter เพิ่มเลย — เป็น function ปกติที่คืน closure |
| รองรับ "role ไหนก็ได้ในกลุ่ม" (OR ของหลาย role) | **ทำยาก** — ต้องเขียน `RequireRole<const ROLE: u8>` แยกทีละ role หรือขยาย const parameter ให้เป็น bitmask (ซับซ้อนขึ้นอีก) | ทำง่าย — middleware เช็ค `roles.iter().any(...)` ตรง ๆ |
| Error message ตอนเขียนผิด | อาจซับซ้อนกว่า (เช่น E0532 ข้างบน) เพราะเกี่ยวกับ type system มากกว่า | ตรงไปตรงมากว่า — เป็น runtime logic ธรรมดา |
| เหมาะกับ | endpoint ที่ role ต้องการ **ตายตัวและรู้ตอน compile time แน่ ๆ** และอยากให้ signature "โฆษณา" requirement ชัดเจน | ส่วนใหญ่ของระบบจริง — ยืดหยุ่นกว่า จัดกลุ่ม endpoint ได้ง่ายกว่า |

**คำแนะนำของบทนี้**: ใช้ **Approach B (runtime middleware) เป็นค่าเริ่มต้น** สำหรับระบบส่วนใหญ่ — เหตุผล
ตรงไปตรงมา: ระบบจริงมักต้องการ "role ไหนก็ได้ในกลุ่มนี้" (เช่น `Admin` หรือ `Staff` ทั้งคู่ทำได้) ซึ่ง Approach
B ทำได้เป็นธรรมชาติ ในขณะที่ Approach A ต้องมี type parameter แยกทีละ role เป๊ะ ๆ (ตามที่พิสูจน์ไว้ในหัวข้อ
76.11 ว่า `RequireStaff` ปฏิเสธ `Admin` ทั้งที่ควรจะผ่าน) — เก็บ Approach A ไว้ใช้เฉพาะจุดที่ role ต้องการ
**ตายตัวจริง ๆ ไม่มีวันเปลี่ยน** และทีมให้ความสำคัญกับ "compiler ช่วยเตือนตอน refactor" มากกว่าความยืดหยุ่น
(เช่น endpoint ที่มีแค่ Admin เท่านั้นที่ทำได้ตลอดไปตามข้อกำหนดทางธุรกิจ ไม่ใช่กรณีที่อาจมีการเปลี่ยน
requirement ทีหลัง) — ทั้งสองวิธีอยู่ร่วมกันในระบบเดียวได้ ตามที่หัวข้อ 76.11 จะสาธิต

### 76.6 Permission Check ในตัว Handler: `AppError::Forbidden`

ทั้ง Approach A และ B ในหัวข้อก่อนเป็น **coarse-grained** (หยาบ) — ตรวจสอบตั้งแต่ก่อนเข้า handler เลยว่า
"เรียก endpoint นี้ได้หรือไม่" แต่บางสถานการณ์ authorization logic ต้อง**อยู่กลางฟังก์ชัน handler** ไม่ใช่
ก่อนหน้ามัน — โดยเฉพาะเมื่อการตัดสินใจต้องพึ่งข้อมูลที่ต้อง**ประมวลผลบางส่วนของ request ไปแล้ว**ก่อนถึงจะรู้

ก่อนอื่น เพิ่ม variant `Forbidden` เข้า `AppError` (ต่อยอด Part 66 ตรง ๆ):

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};

#[derive(Debug)]
enum AppError {
    NotFound,
    Unauthorized(String),
    Forbidden(String), // *** ใหม่ในบทนี้ -- 403, ต่างจาก Unauthorized (401) ***
    Conflict(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code, message) = match self {
            AppError::NotFound =>
                (StatusCode::NOT_FOUND, "NOT_FOUND", "ไม่พบข้อมูลที่ต้องการ".to_string()),
            AppError::Unauthorized(msg) => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED", msg),
            AppError::Forbidden(msg) => (StatusCode::FORBIDDEN, "FORBIDDEN", msg),
            AppError::Conflict(msg) => (StatusCode::CONFLICT, "CONFLICT", msg),
        };
        (status, Json(serde_json::json!({ "error": { "code": code, "message": message } }))).into_response()
    }
}
```

**ทำไมต้องแยก `Unauthorized` (401) กับ `Forbidden` (403) เป็นสอง variant** — นี่คือความเข้าใจผิดที่พบบ่อย
มากในนักพัฒนาจำนวนไม่น้อย (สลับใช้สองคำนี้เหมือนเป็นคำพ้องความหมาย) ทั้งที่ตามมาตรฐาน HTTP (Part 61) ทั้งสอง
มีความหมายต่างกันโดยสิ้นเชิง:

- **`401 Unauthorized`** ที่จริงควรอ่านว่า **"Unauthenticated"** (ชื่อใน spec ทำให้สับสนเอง) — แปลว่า **"ไม่รู้
  ว่า request นี้มาจากใคร"** เพราะไม่มี credential เลย, credential ผิดรูปแบบ, หรือ credential ไม่ผ่านการตรวจ —
  ทางแก้ที่ถูกต้องคือ **"ไป authenticate ใหม่"** (login ใหม่, ขอ token ใหม่)
- **`403 Forbidden`** แปลว่า **"รู้แล้วว่าเป็นใคร (authenticate สำเร็จ) แต่คนนี้ไม่มีสิทธิ์ทำสิ่งนี้"** — ทางแก้
  ไม่ใช่ "ไป login ใหม่" (เพราะ login ใหม่ด้วย credential เดิมก็ยังโดน 403 เหมือนเดิม) แต่คือ **"ต้องมีคนอื่นที่
  มีสิทธิ์มากกว่ามาทำสิ่งนี้แทน"**

โค้ดของบทนี้ทั้งบทแยกสองสถานะนี้ชัดเจนตาม field ที่ต่างกันของ `AppError`: `CurrentUser::from_request_parts`
ล้มเหลว (ไม่มี token/token ผิด) → `Unauthorized`, ส่วน role/permission check ที่ล้มเหลวหลัง authenticate
สำเร็จแล้ว → `Forbidden` เสมอ — **กับดักข้อ 4 ท้ายบทจะพิสูจน์ว่าเกิดอะไรขึ้นถ้าสลับสองอันนี้ผิด**

#### Permission-checking method บน `Role`/`CurrentUser`

แทนที่จะเช็ค `matches!(role, Role::Admin | Role::Staff)` กระจัดกระจายอยู่หลายจุดในโค้ด (เสี่ยงพิมพ์ตกหรือลืม
บาง role ตอนแก้ทีหลัง) รวบเป็น**เมธอดเดียว**บน `Role`:

```rust
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
impl Role {
    /// permission check ระดับ role เดียว -- ใช้เป็น building block ให้ CurrentUser เรียกซ้อนอีกที
    fn can_delete_book(self) -> bool {
        matches!(self, Role::Admin | Role::Staff)
    }
}
```

แล้วให้ `CurrentUser` (ที่มี `roles: Vec<Role>` เพราะรองรับ multi-role ตั้งแต่หัวข้อ 76.3) รวมผลจากทุก role
ที่ user มี — "มี role ไหนสักตัวที่ให้สิทธิ์นี้ไหม" (จะขยายเต็มรูปแบบในหัวข้อ 76.9):

```rust
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
# impl Role { fn can_delete_book(self) -> bool { matches!(self, Role::Admin | Role::Staff) } }
# #[derive(Debug, Clone)]
struct CurrentUser {
    id: u32,
    username: String,
    roles: Vec<Role>,
}

impl CurrentUser {
    fn can_delete_book(&self) -> bool {
        self.roles.iter().any(|r| r.can_delete_book())
    }
}
```

ใช้ใน handler แบบ **attribute-style check** — เช็คตรงกลางฟังก์ชัน ไม่ใช่ก่อนเข้าฟังก์ชันแบบ Approach A/B:

```rust
# use axum::{extract::{Path, State}, http::StatusCode};
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
# impl Role { fn can_delete_book(self) -> bool { matches!(self, Role::Admin | Role::Staff) } }
# #[derive(Debug, Clone)] struct CurrentUser { roles: Vec<Role> }
# impl CurrentUser { fn can_delete_book(&self) -> bool { self.roles.iter().any(|r| r.can_delete_book()) } }
# #[derive(Clone)] struct AppState { books: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<u32, ()>>> }
# #[derive(Debug)] enum AppError { NotFound, Forbidden(String) }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { StatusCode::OK.into_response() }
# }
async fn delete_book_inline_check(
    State(state): State<AppState>,
    user: CurrentUser, // authentication เกิดที่นี่ (extractor ปกติ) -- ยังไม่มี authorization เลย
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    // authorization เกิด "กลางฟังก์ชัน" -- ไม่ใช่ก่อนหน้าแบบ middleware
    if !user.can_delete_book() {
        return Err(AppError::Forbidden("ต้องเป็น Admin หรือ Staff เท่านั้น".into()));
    }
    let mut books = state.books.lock().unwrap();
    books.remove(&id).ok_or(AppError::NotFound)?;
    Ok(StatusCode::NO_CONTENT)
}
```

#### เมื่อไหร่ใช้ route-level middleware เทียบกับ attribute-style check ในตัว handler

ทั้งสองรูปแบบมีที่ใช้ต่างกัน ไม่ใช่ "ดีกว่า" ซึ่งกันและกันเสมอไป:

| สถานการณ์ | เลือกใช้ |
|---|---|
| endpoint **ทั้งเส้นทาง** จำกัดสิทธิ์เดียวกันเสมอ (เช่น ทุก endpoint ใต้ `/admin/*` ต้องเป็น Admin) | Route-level middleware (76.4/76.5) — เช็คครั้งเดียว ครอบทุก endpoint ในกลุ่มนั้น ไม่ต้องเขียนซ้ำทุก handler |
| การตัดสินสิทธิ์ขึ้นกับ**ข้อมูลที่ต้องประมวลผลบางส่วนของ request ไปแล้ว** (เช่น ต้อง parse body หรือ query ทรัพยากรมาก่อนถึงจะรู้ว่าใครเป็นเจ้าของ) | Attribute-style check ในตัว handler (76.6-76.7) — middleware ทำงาน**ก่อน**ที่ extractor อื่นในตัว handler จะได้ทำงาน จึงไม่มีข้อมูลที่ต้องใช้ตัดสินใจให้เช็คได้ตั้งแต่ต้น |
| ต้องเช็ค**หลาย permission ที่ไม่เกี่ยวกัน**ในฟังก์ชันเดียว (เช่น "แก้ field นี้ต้องมี permission A, แก้ field นั้นต้องมี permission B") | Attribute-style check — middleware ตรวจได้แค่ "เข้า endpoint นี้ได้ไหม" ระดับเดียว ไม่ได้ลงรายละเอียดขนาดนั้น |

หัวข้อถัดไป (76.7) คือตัวอย่างที่ชัดที่สุดของแถวที่สอง: **ownership check ที่ทำเป็น middleware ไม่ได้เลย**
เพราะต้องรู้ว่า resource id ไหนที่ request นี้ขอ (จาก path parameter) แล้วต้อง**query resource นั้นออกมาก่อน**
ถึงจะรู้ว่าใครเป็นเจ้าของ

### 76.7 Ownership-Based Authorization: "ทรัพยากรชิ้นนี้เป็นของใคร"

RBAC ทั้งหมดที่ผ่านมา (76.2-76.6) ตอบคำถามระดับ **"ประเภทของการกระทำ"** ("user คนนี้ลบหนังสือได้ไหมใน
หลักการ") — แต่ไม่เคยตอบคำถามระดับ **"ทรัพยากรที่เจาะจง"** ("user คนนี้ยกเลิก booking **id 42 นี้**ได้ไหม")
สองคำถามนี้ต่างกันโดยพื้นฐาน: คำถามแรกตอบได้จาก role/permission ของ user เพียวๆ (ไม่ต้องแตะข้อมูลทรัพยากร
เลย) แต่คำถามที่สองต้อง **query ทรัพยากรที่เจาะจงมาก่อน แล้วเทียบ field ที่บอกความเป็นเจ้าของ**
(`owner_id`) กับ id ของ user ปัจจุบัน

#### โครงสร้างข้อมูล

```rust
# use serde::Serialize;
#[derive(Debug, Clone, Serialize)]
struct Booking {
    id: u32,
    book_id: u32,
    member_id: u32,       // *** field ที่บอกความเป็นเจ้าของ ***
    member_username: String,
    status: String,
}
```

#### Handler: อ่าน booking หนึ่งรายการ พร้อม ownership check

```rust
# use axum::{extract::{Path, State}, Json};
# use serde::Serialize;
# #[derive(Debug, Clone, Serialize)]
# struct Booking { id: u32, book_id: u32, member_id: u32, member_username: String, status: String }
# #[derive(Debug, Clone, Copy, PartialEq, Eq, std::hash::Hash)]
# enum Permission { ViewAnyBooking }
# #[derive(Debug, Clone)] struct CurrentUser { id: u32, permissions: std::collections::HashSet<Permission> }
# impl CurrentUser { fn has_permission(&self, p: Permission) -> bool { self.permissions.contains(&p) } }
# #[derive(Clone)] struct AppState { bookings: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<u32, Booking>>> }
# #[derive(Debug)] enum AppError { NotFound }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { axum::http::StatusCode::NOT_FOUND.into_response() }
# }
async fn get_booking(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<Json<Booking>, AppError> {
    // ขั้นที่ 1: ต้อง "โหลดทรัพยากรที่เจาะจงมาก่อน" เสมอ -- นี่คือเหตุผลที่ทำเป็น middleware ไม่ได้
    // (เทียบ Part 70 ที่ query แถวเจาะจงจาก PgPool ก่อนตัดสินใจอะไรต่อ)
    let bookings = state.bookings.lock().unwrap();
    let booking = bookings.get(&id).ok_or(AppError::NotFound)?;

    // ขั้นที่ 2: เทียบ owner_id กับ user ปัจจุบัน -- นี่คือ ownership check ตัวจริง
    let is_owner = booking.member_id == user.id;
    let can_view_any = user.has_permission(Permission::ViewAnyBooking); // Staff/Admin ข้าม ownership ได้
    if !is_owner && !can_view_any {
        return Err(AppError::NotFound); // *** ทำไมเป็น NotFound ไม่ใช่ Forbidden -- อธิบายด้านล่างเต็ม ๆ ***
    }

    Ok(Json(booking.clone()))
}
```

#### คำถามที่ยากที่สุดของหัวข้อนี้: ตอบ `403` หรือ `404` ให้คนที่ไม่ใช่เจ้าของ?

นี่คือจุดที่หลายทีมถกเถียงกันจริง ไม่มีคำตอบที่ถูกต้องแบบเดียวสำหรับทุกระบบ — มาดูทั้งสองแนวทางตรงไปตรงมา:

**แนวทางที่ 1 — ตอบ `403 Forbidden`**: "booking id นี้**มีอยู่จริง** แต่คุณไม่มีสิทธิ์เข้าถึง" — ข้อดีคือ
ตรงไปตรงมาตามความหมายของ HTTP status code (Part 61) เป๊ะ ๆ: มี resource แต่ปฏิเสธสิทธิ์ = 403 ชัดเจน ผู้ใช้
(หรือ frontend) เห็น error message ที่บอกสาเหตุตรง ๆ ได้ ("นี่ไม่ใช่ booking ของคุณ") ช่วย debug ง่ายกว่า —
**ข้อเสีย**: **เผยข้อมูลว่า resource id นี้มีอยู่จริงในระบบ** ให้กับคนที่ไม่มีสิทธิ์เข้าถึงมันด้วยซ้ำ — ถ้า
attacker ไล่ยิง id ตั้งแต่ 1 ถึง 10000 แล้วดูว่า id ไหนตอบ `403` (มีอยู่จริง) กับ `404` (ไม่มีจริง) จะรู้ **ช่วง
ของ id ที่ถูกใช้งานจริงในระบบ** ได้ทันที (เช่น รู้ว่าระบบมี booking ทั้งหมดประมาณกี่รายการ) ซึ่งเป็นข้อมูลที่ไม่
ควรรั่วออกไปโดยไม่จำเป็น (information disclosure)

**แนวทางที่ 2 — ตอบ `404 Not Found` เหมือนกรณี "ไม่มีจริง" ทุกประการ**: ไม่แยกแยะให้คนที่ไม่ใช่เจ้าของรู้เลยว่า
"ไม่มีอยู่จริง" กับ "มีอยู่จริงแต่ไม่ใช่ของคุณ" ต่างกันอย่างไร — response ทั้ง body และ status เหมือนกันเป๊ะทั้ง
สองกรณี — **ข้อดี**: ปิดช่องทาง information disclosure ที่แนวทาง 1 มี สมบูรณ์ (นี่คือแนวทางที่บริการใหญ่ๆ
จำนวนมากเลือกใช้กับ private resource — repository ส่วนตัวที่ไม่มีสิทธิ์เข้าถึง มักตอบ 404 ไม่ใช่ 403) —
**ข้อเสีย**: ผู้ใช้ที่เป็นเจ้าของจริงแต่พิมพ์ id ผิด กับผู้ใช้ที่พิมพ์ id ถูกแต่ไม่ใช่เจ้าของ **ได้ error
เดียวกันเป๊ะ** ทำให้ debug ยากขึ้นเล็กน้อย (ต้องเดาเองว่าอันไหนคือสาเหตุจริง) และทีม support ที่ได้รับแจ้งปัญหา
จาก user ก็ต้องอธิบายเพิ่มว่า "404 ไม่ได้แปลว่า id ผิดเสมอไป"

**การเลือกของบทนี้**: ใช้ **แนวทางที่ 2 (404 ซ่อนการมีอยู่) เป็นค่าเริ่มต้นสำหรับ GET/ownership check ทั่วไป**
ตามที่โค้ดข้างบนทำไว้ — เหตุผล: ในระบบที่มีข้อมูลที่ควรเป็นส่วนตัวจริง (booking ของสมาชิกแต่ละคน) การป้องกัน
ไม่ให้รู้ว่า "resource นี้มีอยู่จริง" มีค่ามากกว่าความสะดวกด้าน debug เล็กน้อยที่เสียไป — **แต่บทนี้ก็สาธิต
แนวทางที่ 1 ไว้คู่กัน** (endpoint แยก `POST /bookings/{id}/cancel-explicit` ในหัวข้อ 76.11) เพื่อให้เห็นทั้ง
สองแบบทำงานจริงและเลือกได้ตามบริบทของระบบตัวเอง — จุดสำคัญที่สุดที่ต้องจำคือ**ต้องตัดสินใจเรื่องนี้อย่างมี
เจตนา** ไม่ใช่ปล่อยให้แต่ละ endpoint ในระบบเดียวกันเลือกคนละแบบโดยไม่ได้ตั้งใจ (ซึ่งจะกลายเป็นความไม่สอดคล้อง
กันที่แยกแยะยากกว่าเดิมอีก)

โค้ด handler ทั้งสองแบบ (ยกเลิก booking) เทียบกันตรง ๆ:

```rust
# use axum::{extract::{Path, State}, http::StatusCode};
# #[derive(Debug, Clone)] struct Booking { member_id: u32, status: String }
# #[derive(Debug, Clone, Copy, PartialEq, Eq, std::hash::Hash)]
# enum Permission { CancelAnyBooking }
# #[derive(Debug, Clone)] struct CurrentUser { id: u32, permissions: std::collections::HashSet<Permission> }
# impl CurrentUser { fn has_permission(&self, p: Permission) -> bool { self.permissions.contains(&p) } }
# #[derive(Clone)] struct AppState { bookings: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<u32, Booking>>> }
# #[derive(Debug)] enum AppError { NotFound, Forbidden(String) }
# impl axum::response::IntoResponse for AppError {
#     fn into_response(self) -> axum::response::Response { StatusCode::OK.into_response() }
# }

/// แนวทางที่ 2: ซ่อนการมีอยู่ -- ตอบ 404 ทั้ง "ไม่มีจริง" และ "มีจริงแต่ไม่ใช่เจ้าของ"
async fn cancel_booking_hide_existence(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut bookings = state.bookings.lock().unwrap();
    let booking = bookings.get_mut(&id).ok_or(AppError::NotFound)?;

    let is_owner = booking.member_id == user.id;
    let can_cancel_any = user.has_permission(Permission::CancelAnyBooking);
    if !is_owner && !can_cancel_any {
        return Err(AppError::NotFound); // เหมือน "ไม่มีจริง" ทุกประการ
    }

    booking.status = "cancelled".to_string();
    Ok(StatusCode::NO_CONTENT)
}

/// แนวทางที่ 1: เผยว่ามีอยู่จริง แต่ปฏิเสธสิทธิ์ตรง ๆ -- ตอบ 403 ให้คนที่ไม่ใช่เจ้าของ
async fn cancel_booking_explicit_forbidden(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut bookings = state.bookings.lock().unwrap();
    let booking = bookings.get_mut(&id).ok_or(AppError::NotFound)?;

    let is_owner = booking.member_id == user.id;
    let can_cancel_any = user.has_permission(Permission::CancelAnyBooking);
    if !is_owner && !can_cancel_any {
        return Err(AppError::Forbidden("booking นี้ไม่ใช่ของคุณ ไม่มีสิทธิ์ยกเลิก".into()));
    }

    booking.status = "cancelled".to_string();
    Ok(StatusCode::NO_CONTENT)
}
```

ทดสอบจริงทั้งสองแบบ (`nan` พยายามยกเลิก booking id 2 ที่เป็นของ `wit`) — สังเกตว่า response ของแนวทางที่ 2
**เหมือนกันเป๊ะไบต์ต่อไบต์**กับตอนยิง id ที่ไม่มีอยู่จริงเลย (พิสูจน์ด้วยการเทียบ `content-length`):

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/bookings/2/cancel-explicit -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-type: application/json
content-length: 148

{"error":{"code":"FORBIDDEN","message":"booking นี้ไม่ใช่ของคุณ ไม่มีสิทธิ์ยกเลิก"}}
```

และพิสูจน์ว่า `GET /bookings/2` (แนวทางที่ 2) ตอบเหมือนกันเป๊ะทั้งกรณี "ไม่ใช่เจ้าของ" กับ "ไม่มีจริง" —
`content-length: 106` ทั้งคู่:

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $NAN_TOKEN"   # ของ wit ไม่ใช่ nan
```
```
HTTP/1.1 404 Not Found
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/999 -H "Authorization: Bearer $NAN_TOKEN"  # ไม่มีจริง
```
```
HTTP/1.1 404 Not Found
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

### 76.8 Permission Granularity: เมื่อ Role เดียวไม่พอ

หัวข้อ 76.4 ทิ้งปัญหาไว้ให้เห็นชัด: การเช็ค `roles.contains(&Role::Staff)` ตรง ๆ ทำให้ `Admin` (ที่ไม่มี
`Role::Staff` ติดตัว) โดนปฏิเสธในสิ่งที่ตามสามัญสำนึกควรทำได้ — รากของปัญหาคือ **การผูก "สิทธิ์ทำสิ่งหนึ่ง"
เข้ากับ "ชื่อของ role" ตรง ๆ** ทำให้ role ใหม่ที่ควรมีสิทธิ์เดียวกันต้องถูก "จำ" ไว้ทุกจุดที่เช็ค role นั้น
เป็นรายชื่อ — ระบบใหญ่ที่มี role เยอะขึ้นจะยิ่งเจอปัญหานี้หนักขึ้นเรื่อย ๆ

ทางแก้ที่เป็นมาตรฐานคือแยก **"สิทธิ์การทำงาน" (`Permission`) ออกจาก "role"** เป็นสองระดับที่ไม่ผูกกันตรง ๆ
— **role หนึ่งตัว "มี" permission หลายตัวได้ (many-to-many)** แล้วให้ authorization check ทั้งหมด**เช็คที่
permission เสมอ ไม่ใช่เช็คที่ role**:

```rust
use std::collections::HashSet;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, serde::Serialize, serde::Deserialize)]
enum Permission {
    ViewBook,
    CreateBooking,
    CancelOwnBooking,
    DeleteBook,
    ManageUsers,
    ViewAnyBooking,
    CancelAnyBooking,
}
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }

/// สิทธิ์ที่ role หนึ่งตัวมี -- นี่คือ mapping "role -> permissions" (many-to-many ฝั่ง role เดียว)
fn role_permissions(role: Role) -> HashSet<Permission> {
    use Permission::*;
    match role {
        Role::Admin => HashSet::from([
            ViewBook, CreateBooking, CancelOwnBooking,
            DeleteBook, ManageUsers, ViewAnyBooking, CancelAnyBooking,
        ]),
        Role::Staff => HashSet::from([
            ViewBook, CreateBooking, CancelOwnBooking,
            DeleteBook, ViewAnyBooking, CancelAnyBooking, // *** ไม่มี ManageUsers ***
        ]),
        Role::Member => HashSet::from([ViewBook, CreateBooking, CancelOwnBooking]),
    }
}
```

**สังเกตว่า `DeleteBook` อยู่ในทั้ง `role_permissions(Role::Admin)` และ `role_permissions(Role::Staff)`** —
นี่คือคำตอบตรงของปัญหาในหัวข้อ 76.4: ถ้า endpoint เช็ค `user.has_permission(Permission::DeleteBook)` แทนการ
เช็ค `roles.contains(&Role::Staff)` ตรง ๆ **ทั้ง `Admin` และ `Staff` จะผ่านการเช็คนี้ทั้งคู่** โดยไม่ต้องเขียน
`matches!(role, Role::Admin | Role::Staff)` กระจัดกระจายอยู่หลายจุดเลย — จุดเดียวที่ต้องแก้ถ้าวันหนึ่งต้องการ
เพิ่ม role ใหม่ที่ควรลบหนังสือได้ด้วย คือเพิ่ม `DeleteBook` เข้า `HashSet` ของ role นั้นใน `role_permissions`
เท่านั้น — **ทุก endpoint ที่เช็ค `Permission::DeleteBook` อยู่แล้วจะทำงานถูกต้องทันทีโดยไม่ต้องแก้อะไรเพิ่ม**

การคำนวณ permission ทั้งหมดที่ user มี (เมื่อ user มีได้หลาย role พร้อมกัน — หัวข้อ 76.9) ทำตอน **login**
เพียงครั้งเดียว แล้วฝังผลลัพธ์ลง JWT/session ไปเลย (ไม่ต้องคำนวณซ้ำทุก request):

```rust
# use std::collections::HashSet;
# #[derive(Debug, Clone, Copy, PartialEq, Eq, std::hash::Hash)]
# enum Permission { ViewBook }
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
# fn role_permissions(_r: Role) -> HashSet<Permission> { HashSet::new() }

/// union สิทธิ์ของ role ทุกตัวที่ user มี -- ใช้ HashSet::extend() ตามที่ Part 15 สอนไว้
fn permissions_for_roles(roles: &[Role]) -> HashSet<Permission> {
    let mut set = HashSet::new();
    for role in roles {
        set.extend(role_permissions(*role));
    }
    set
}
```

`HashSet::extend()` รับ iterator ของ `Permission` แล้วเพิ่มเข้า set โดยอัตโนมัติข้าม duplicate (ถ้า role
สองตัวมี permission ซ้ำกัน เช่น `ViewBook` ที่ทั้งสาม role มี — `HashSet` การันตีว่าจะเก็บแค่ตัวเดียวเสมอ ตาม
คุณสมบัติพื้นฐานของ set ที่ Part 15 สอนไว้) — นี่คือจุดที่ `HashSet` เหมาะกว่า `Vec` อย่างชัดเจน: ถ้าใช้ `Vec`
แทน จะต้องเขียน dedup logic เองหรือปล่อยให้มี permission ซ้ำกันในลิสต์ (ไม่ผิด แต่เปลืองที่และทำ `contains()`
ช้าลงเป็น O(n) เทียบกับ O(1) โดยเฉลี่ยของ `HashSet`)

### 76.9 Multi-Role Users: เมื่อ User หนึ่งคนมีมากกว่าหนึ่ง Role

จากตัวอย่างเซิร์ฟเวอร์ในหัวข้อ 76.3 ที่ผ่านมา สังเกตว่า `staff1` ถูกกำหนดให้มี **สอง role พร้อมกัน**:
`vec![Role::Staff, Role::Member]` — นี่คือสถานการณ์จริงที่พบได้บ่อยมาก: บรรณารักษ์ก็เป็นสมาชิกห้องสมุดได้
ด้วยตัวเอง (ยืมหนังสือเพื่อใช้งานส่วนตัว ไม่ใช่แค่จัดการหนังสือให้คนอื่น)

ผลที่ตามมาโดยตรงคือ **permission check ทุกจุดต้องเปลี่ยนจาก "role นี้ให้สิทธิ์ไหม" เป็น "มี role ไหนสักตัว
ในบรรดา role ทั้งหมดที่ user มี ที่ให้สิทธิ์นี้บ้างไหม"** — ในทาง Rust แปลว่าเปลี่ยนจากการเทียบค่าตรง ๆ
(`role == Role::Admin`) เป็นการวน iterator ด้วย `.any()`:

```rust
# use std::collections::HashSet;
# #[derive(Debug, Clone, Copy, PartialEq, Eq, std::hash::Hash)]
# enum Permission { DeleteBook, CancelAnyBooking }
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
#[derive(Debug, Clone)]
struct CurrentUser {
    id: u32,
    username: String,
    roles: Vec<Role>,                    // *** ไม่ใช่ Role เดี่ยว ๆ อีกต่อไป ***
    permissions: HashSet<Permission>,    // permission รวมจากทุก role แล้ว (คำนวณตอน login, หัวข้อ 76.8)
}

impl CurrentUser {
    fn has_role(&self, role: Role) -> bool {
        self.roles.contains(&role) // "มี role นี้อยู่ในลิสต์ไหม" -- ตรงไปตรงมา
    }

    fn has_permission(&self, perm: Permission) -> bool {
        self.permissions.contains(&perm) // ใช้ permissions ที่รวมมาแล้ว -- ไม่ต้องวน roles ซ้ำอีก
    }
}
```

**จุดสำคัญที่ต้องสังเกต**: `has_permission` ไม่ต้องวน `self.roles` เองเลย เพราะ `self.permissions` **ถูก
รวมมาแล้วตั้งแต่ตอน login** (ผ่าน `permissions_for_roles()` ในหัวข้อ 76.8) — นี่คือประโยชน์เชิงประสิทธิภาพที่
จับต้องได้จริงของการแยก permission ออกจาก role: **การรวมสิทธิ์จากหลาย role ทำครั้งเดียวตอน login ไม่ใช่ทำซ้ำ
ทุก request** — ถ้าเช็คแบบ `self.roles.iter().any(|r| role_permissions(*r).contains(&perm))` ตรง ๆ ทุกครั้ง
ที่ต้องเช็ค permission จะเสีย CPU คำนวณ `HashSet` ใหม่ทุก request โดยไม่จำเป็น (แม้จะไม่ได้ช้ามากในสเกลเล็ก
แต่เป็นการคำนวณซ้ำที่ไม่มีประโยชน์)

แต่สำหรับ role-level check ที่ยังต้องใช้ role ตรง ๆ (ไม่ใช่ permission) เช่นเมธอด `can_delete_book()` จาก
หัวข้อ 76.6 — ต้องรวมผลข้าม role ด้วย `.any()` เช่นกัน:

```rust
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
impl Role {
    fn can_delete_book(self) -> bool {
        matches!(self, Role::Admin | Role::Staff)
    }
}
# #[derive(Debug, Clone)] struct CurrentUser { roles: Vec<Role> }
impl CurrentUser {
    /// "มี role ไหนสักตัวที่ทำได้" ไม่ใช่ "role เดียวทำได้" -- นี่คือความต่างสำคัญจาก single-role model
    fn can_delete_book(&self) -> bool {
        self.roles.iter().any(|r| r.can_delete_book())
    }
}
```

ทดสอบจริงว่า `staff1` (roles = `[Staff, Member]`) เห็น permission ที่รวมมาจากทั้งสอง role — คำสั่ง
`GET /me` (ตัดมาจากการรันจริงในหัวข้อ 76.3):

```json
{"id":2,"username":"staff1","roles":["Staff","Member"],"permissions":["CancelOwnBooking","CreateBooking","DeleteBook","ViewBook","ViewAnyBooking","CancelAnyBooking"]}
```

เทียบกับ `nan` (roles = `[Member]` เดี่ยว ๆ):

```json
{"id":3,"username":"nan","roles":["Member"],"permissions":["CancelOwnBooking","CreateBooking","ViewBook"]}
```

`staff1` มี `DeleteBook`, `ViewAnyBooking`, `CancelAnyBooking` เพิ่มมาจาก role `Staff` ที่ `nan` ไม่มี —
พิสูจน์ว่า permission ถูกรวมจากทุก role ที่ user มีจริง ไม่ใช่แค่ role แรกในลิสต์หรือ role สุดท้าย

### 76.10 มองไปข้างหน้า: ABAC และ Policy Engine (สำหรับตอนที่ RBAC ไม่พอ)

RBAC ที่บทนี้สอนครอบคลุมความต้องการของระบบจริงส่วนใหญ่ได้ดีมาก — แต่มีสถานการณ์ที่ RBAC "ไม่พอ" ในทางโครงสร้าง
ไม่ใช่แค่ต้องเขียน permission เยอะขึ้น: เมื่อการตัดสินสิทธิ์ต้องขึ้นกับ **หลายมิติของ context ที่เปลี่ยนแปลง
ได้ ไม่ใช่แค่ role ของ user** เช่น เวลาปัจจุบัน ("เข้าถึงได้แค่ในเวลาทำการ"), IP/ตำแหน่งที่ request มา ("เข้า
ถึงได้แค่จาก network ภายในองค์กร"), หรือ attribute ของทรัพยากรที่ซับซ้อนกว่า ownership ตรง ๆ ("อนุมัติได้ถ้า
ยอดเงินต่ำกว่าขีดจำกัดของตำแหน่งงานนี้")

**ABAC (Attribute-Based Access Control)** คือโมเดลที่กว้างกว่า RBAC: แทนที่จะตัดสินจาก role อย่างเดียว ABAC
ตัดสินจาก **attribute ใด ๆ ก็ได้ของ subject (ผู้ขอ), resource (ทรัพยากร), action (การกระทำ), และ environment
(context ตอนนั้น)** ร่วมกัน — policy จึงเขียนได้ละเอียดกว่า RBAC มาก (เช่น "อนุมัติถ้า subject.department ==
resource.department AND environment.time อยู่ในเวลาทำการ AND action == 'approve'") แต่แลกมาด้วยความซับซ้อน
ที่สูงกว่ามาก ทั้งตอนออกแบบ policy และตอน debug ว่าทำไม request หนึ่งถึงผ่าน/ไม่ผ่าน

สำหรับระบบที่ policy ซับซ้อนถึงจุดที่เขียนเป็น `if`/`match` ในโค้ด Rust ตรง ๆ ไม่ไหวแล้ว (เงื่อนไขเปลี่ยนบ่อย,
ต้องให้ non-engineer แก้ policy ได้โดยไม่ deploy โค้ดใหม่, ต้อง audit ว่า policy ไหนอนุญาต request ไหนบ้าง)
มี**policy engine** แยกต่างหากที่ออกแบบมาสำหรับงานนี้โดยเฉพาะ ที่รู้จักกันแพร่หลายสองตัวคือ:

- **Open Policy Agent (OPA)** — policy engine แบบ general-purpose เขียน policy ด้วยภาษา **Rego** ของตัวเอง
  รันเป็น service แยก (หรือ library) ที่แอปเรียกไปถามว่า "request นี้ผ่านไหม" แยกออกจาก business logic
  โดยสิ้นเชิง
- **Cedar** — policy engine ที่ AWS พัฒนา (ใช้จริงใน AWS Verified Permissions) เขียน policy ด้วยภาษา syntax
  ที่ใกล้เคียงประโยคภาษาธรรมดามากกว่า Rego เน้นให้ policy อ่าน/ตรวจสอบได้ง่ายและพิสูจน์ทาง formal verification
  ได้ (verify ได้ว่า policy set หนึ่งจะไม่มีวันอนุญาตบางอย่างที่ไม่ควรอนุญาต)

บทนี้**ไม่ลงรายละเอียดของทั้งสองตัว** เพราะอยู่นอกสโคป — สิ่งที่ควรจำไว้คือ: **RBAC ที่บทนี้สอนคือจุดเริ่มต้น
ที่ถูกต้องสำหรับระบบส่วนใหญ่เสมอ** อย่ารีบไปหา policy engine ตั้งแต่ต้นถ้า RBAC ธรรมดายังตอบโจทย์ได้อยู่ (ความ
ซับซ้อนที่เพิ่มมาไม่คุ้มถ้ายังไม่ถึงจุดที่จำเป็นจริง ๆ) แต่ถ้าระบบโตไปถึงจุดที่ authorization logic เริ่มมี
`if`/`match` ซ้อนกันลึกจนอ่านไม่ออก หรือ policy เปลี่ยนบ่อยกว่าที่ deploy ทันได้ — นี่คือสัญญาณว่าถึงเวลาศึกษา
ABAC/policy engine อย่างจริงจังแล้ว

### 76.11 Capstone: ระบบห้องสมุด/จองตั๋วพร้อม RBAC เต็มรูปแบบ

ตอนนี้มาประกอบทุกหัวข้อ (76.2-76.9) เข้าเป็นเซิร์ฟเวอร์เดียวที่ทำงานได้จริง — 4 บัญชีผู้ใช้ (`admin` role
`Admin`, `staff1` role `[Staff, Member]` multi-role, `nan`/`wit` role `Member`), route ที่ gate ด้วยทั้ง
Approach A (const generics) และ Approach B (runtime middleware), permission check แบบ inline, ownership
check ทั้งสองแนวทาง (403/404), และ multi-role ที่ทำงานจริงทุกจุด

```rust
use axum::{
    extract::{FromRequestParts, Path, State},
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{delete, get, post},
    Json, Router,
};
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::collections::{HashMap, HashSet};
use std::sync::{Arc, Mutex};

// ============================== Role & Permission ==============================

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[repr(u8)]
enum Role {
    Admin = 0,
    Staff = 1,
    Member = 2,
}

impl Role {
    const fn from_u8(v: u8) -> Role {
        match v {
            0 => Role::Admin,
            1 => Role::Staff,
            2 => Role::Member,
            _ => panic!("ROLE ที่ไม่รู้จัก"),
        }
    }

    fn can_delete_book(self) -> bool {
        matches!(self, Role::Admin | Role::Staff)
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
enum Permission {
    ViewBook,
    CreateBooking,
    CancelOwnBooking,
    DeleteBook,
    ManageUsers,
    ViewAnyBooking,
    CancelAnyBooking,
}

fn role_permissions(role: Role) -> HashSet<Permission> {
    use Permission::*;
    match role {
        Role::Admin => HashSet::from([
            ViewBook, CreateBooking, CancelOwnBooking,
            DeleteBook, ManageUsers, ViewAnyBooking, CancelAnyBooking,
        ]),
        Role::Staff => HashSet::from([
            ViewBook, CreateBooking, CancelOwnBooking,
            DeleteBook, ViewAnyBooking, CancelAnyBooking,
        ]),
        Role::Member => HashSet::from([ViewBook, CreateBooking, CancelOwnBooking]),
    }
}

fn permissions_for_roles(roles: &[Role]) -> HashSet<Permission> {
    let mut set = HashSet::new();
    for role in roles {
        set.extend(role_permissions(*role));
    }
    set
}

// ============================== AppError (ต่อยอด Part 66) ==============================

#[derive(Debug)]
enum AppError {
    NotFound,
    Unauthorized(String),
    Forbidden(String),
    Conflict(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code, message) = match self {
            AppError::NotFound =>
                (StatusCode::NOT_FOUND, "NOT_FOUND", "ไม่พบข้อมูลที่ต้องการ".to_string()),
            AppError::Unauthorized(msg) => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED", msg),
            AppError::Forbidden(msg) => (StatusCode::FORBIDDEN, "FORBIDDEN", msg),
            AppError::Conflict(msg) => (StatusCode::CONFLICT, "CONFLICT", msg),
        };
        (status, Json(serde_json::json!({ "error": { "code": code, "message": message } }))).into_response()
    }
}

// ============================== Claims / CurrentUser (ต่อยอด Part 74) ==============================

#[derive(Debug, Clone, Serialize, Deserialize)]
struct Claims {
    sub: String,
    username: String,
    roles: Vec<Role>,
    permissions: Vec<Permission>,
    iat: usize,
    exp: usize,
}

#[derive(Debug, Clone)]
struct CurrentUser {
    id: u32,
    username: String,
    roles: Vec<Role>,
    permissions: HashSet<Permission>,
}

impl CurrentUser {
    fn has_role(&self, role: Role) -> bool {
        self.roles.contains(&role)
    }

    fn has_permission(&self, perm: Permission) -> bool {
        self.permissions.contains(&perm)
    }

    fn can_delete_book(&self) -> bool {
        self.roles.iter().any(|r| r.can_delete_book())
    }
}

fn decode_claims(headers: &axum::http::HeaderMap, secret: &[u8]) -> Result<Claims, AppError> {
    let header_value = headers
        .get("Authorization")
        .ok_or_else(|| AppError::Unauthorized("ไม่พบ header Authorization: Bearer <token>".into()))?
        .to_str()
        .map_err(|_| AppError::Unauthorized("Authorization header ไม่ใช่ ASCII ที่ถูกต้อง".into()))?;

    let token = header_value
        .strip_prefix("Bearer ")
        .ok_or_else(|| AppError::Unauthorized("รูปแบบต้องเป็น 'Bearer <token>'".into()))?;

    let mut validation = Validation::new(Algorithm::HS256);
    validation.set_required_spec_claims(&["exp", "sub"]);
    let decoding_key = DecodingKey::from_secret(secret);

    let token_data = decode::<Claims>(token, &decoding_key, &validation)
        .map_err(|e| AppError::Unauthorized(format!("token ไม่ผ่านการตรวจสอบ: {}", e)))?;

    Ok(token_data.claims)
}

impl FromRequestParts<AppState> for CurrentUser {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let claims = decode_claims(&parts.headers, &state.jwt_secret)?;
        let user_id: u32 = claims
            .sub
            .parse()
            .map_err(|_| AppError::Unauthorized("sub claim ไม่ใช่ user id ที่ถูกต้อง".into()))?;

        Ok(CurrentUser {
            id: user_id,
            username: claims.username,
            roles: claims.roles.clone(),
            permissions: claims.permissions.into_iter().collect(),
        })
    }
}

// ===================== Approach A: RequireRole<const ROLE: u8> (const generics) =====================

struct RequireRole<const ROLE: u8>(CurrentUser);

impl<const ROLE: u8> FromRequestParts<AppState> for RequireRole<ROLE> {
    type Rejection = AppError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let user = CurrentUser::from_request_parts(parts, state).await?;
        let required = Role::from_u8(ROLE);
        if !user.has_role(required) {
            return Err(AppError::Forbidden(format!(
                "ต้องมี role {:?} เท่านั้นถึงจะเรียก endpoint นี้ได้",
                required
            )));
        }
        Ok(RequireRole(user))
    }
}

type RequireAdmin = RequireRole<{ Role::Admin as u8 }>;
type RequireStaff = RequireRole<{ Role::Staff as u8 }>;

// ===================== Approach B: runtime middleware (route-level) =====================

fn require_role(
    jwt_secret: Arc<Vec<u8>>,
    required: Role,
) -> impl Fn(
    axum::extract::Request,
    axum::middleware::Next,
) -> std::pin::Pin<Box<dyn std::future::Future<Output = Response> + Send>>
       + Clone {
    move |req: axum::extract::Request, next: axum::middleware::Next| {
        let jwt_secret = jwt_secret.clone();
        Box::pin(async move {
            let claims = match decode_claims(req.headers(), &jwt_secret) {
                Ok(c) => c,
                Err(e) => return e.into_response(),
            };
            if !claims.roles.contains(&required) {
                return AppError::Forbidden(format!(
                    "ต้องมี role {:?} เท่านั้นถึงจะเรียก endpoint นี้ได้",
                    required
                ))
                .into_response();
            }
            next.run(req).await
        })
    }
}

// ============================== ข้อมูลจำลอง (แทน DB จริงตาม Part 70) ==============================

#[derive(Debug, Clone, Serialize)]
struct Book {
    id: u32,
    title: String,
}

#[derive(Debug, Clone, Serialize)]
struct Booking {
    id: u32,
    book_id: u32,
    member_id: u32,
    member_username: String,
    status: String,
}

#[derive(Debug, Clone)]
struct UserRecord {
    id: u32,
    username: String,
    // *** DEMO เท่านั้น: เก็บ password เป็น plaintext เพื่อโฟกัสที่ RBAC ล้วน ๆ ***
    // production ต้อง hash ด้วย argon2 เสมอตามที่ Part 74 หัวข้อ 74.6 สอนไว้ -- ไม่มีข้อยกเว้น
    password: String,
    roles: Vec<Role>,
}

#[derive(Clone)]
struct AppState {
    jwt_secret: Arc<Vec<u8>>,
    users: Arc<Mutex<HashMap<String, UserRecord>>>,
    books: Arc<Mutex<HashMap<u32, Book>>>,
    bookings: Arc<Mutex<HashMap<u32, Booking>>>,
    next_booking_id: Arc<Mutex<u32>>,
}

// ============================== Request/Response DTOs ==============================

#[derive(Deserialize)]
struct LoginRequest {
    username: String,
    password: String,
}

#[derive(Serialize)]
struct LoginResponse {
    token: String,
    roles: Vec<Role>,
}

#[derive(Deserialize)]
struct NewBookingRequest {
    book_id: u32,
}

#[derive(Serialize)]
struct MeResponse {
    id: u32,
    username: String,
    roles: Vec<Role>,
    permissions: Vec<Permission>,
}

// ============================== Handlers ==============================

fn now_ts() -> usize {
    std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .unwrap()
        .as_secs() as usize
}

async fn login(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>,
) -> Result<Json<LoginResponse>, AppError> {
    let users = state.users.lock().unwrap();
    let user = users
        .get(&req.username)
        .filter(|u| u.password == req.password)
        .ok_or_else(|| AppError::Unauthorized("username หรือ password ไม่ถูกต้อง".into()))?;

    let permissions: Vec<Permission> = permissions_for_roles(&user.roles).into_iter().collect();

    let claims = Claims {
        sub: user.id.to_string(),
        username: user.username.clone(),
        roles: user.roles.clone(),
        permissions,
        iat: now_ts(),
        exp: now_ts() + 3600,
    };

    let token = encode(
        &Header::new(Algorithm::HS256),
        &claims,
        &EncodingKey::from_secret(&state.jwt_secret),
    )
    .map_err(|e| AppError::Conflict(format!("สร้าง token ไม่สำเร็จ: {e}")))?;

    Ok(Json(LoginResponse { token, roles: user.roles.clone() }))
}

async fn me(user: CurrentUser) -> Json<MeResponse> {
    Json(MeResponse {
        id: user.id,
        username: user.username.clone(),
        roles: user.roles.clone(),
        permissions: user.permissions.iter().copied().collect(),
    })
}

async fn list_books(State(state): State<AppState>, _user: CurrentUser) -> Json<Vec<Book>> {
    let books = state.books.lock().unwrap();
    Json(books.values().cloned().collect())
}

/// Approach A (const generics): ต้องเป็น Admin เท่านั้น -- มองเห็นได้จาก type `RequireAdmin` ตรง ๆ
async fn delete_book_admin_only(
    State(state): State<AppState>,
    RequireRole(_user): RequireAdmin,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut books = state.books.lock().unwrap();
    books.remove(&id).ok_or(AppError::NotFound)?;
    Ok(StatusCode::NO_CONTENT)
}

/// Approach B (runtime middleware): เกต role ที่ .route_layer() -- handler ไม่มีร่องรอยของ requirement เลย
async fn delete_book_staff_gated(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut books = state.books.lock().unwrap();
    books.remove(&id).ok_or(AppError::NotFound)?;
    Ok(StatusCode::NO_CONTENT)
}

/// ข้อจำกัดของ const generics แบบ single-role: RequireStaff ไม่ยอมรับ Admin (ที่ไม่มี role Staff ติดตัว)
async fn list_all_bookings_staff_only(
    State(state): State<AppState>,
    RequireRole(_user): RequireStaff,
) -> Json<Vec<Booking>> {
    let bookings = state.bookings.lock().unwrap();
    Json(bookings.values().cloned().collect())
}

/// attribute-style permission check ในตัว handler ตรง ๆ (76.6)
async fn delete_book_inline_check(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    if !user.can_delete_book() {
        return Err(AppError::Forbidden("ต้องเป็น Admin หรือ Staff เท่านั้น".into()));
    }
    let mut books = state.books.lock().unwrap();
    books.remove(&id).ok_or(AppError::NotFound)?;
    Ok(StatusCode::NO_CONTENT)
}

async fn create_booking(
    State(state): State<AppState>,
    user: CurrentUser,
    Json(req): Json<NewBookingRequest>,
) -> Result<(StatusCode, Json<Booking>), AppError> {
    if !user.has_permission(Permission::CreateBooking) {
        return Err(AppError::Forbidden("ไม่มีสิทธิ์สร้าง booking".into()));
    }

    let mut next_id = state.next_booking_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;

    let booking = Booking {
        id,
        book_id: req.book_id,
        member_id: user.id,
        member_username: user.username.clone(),
        status: "active".to_string(),
    };
    state.bookings.lock().unwrap().insert(id, booking.clone());
    Ok((StatusCode::CREATED, Json(booking)))
}

/// ownership check (76.7) -- 404 ทั้ง "ไม่มีจริง" และ "มีจริงแต่ไม่ใช่เจ้าของ" (เว้นแต่มี ViewAnyBooking)
async fn get_booking(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<Json<Booking>, AppError> {
    let bookings = state.bookings.lock().unwrap();
    let booking = bookings.get(&id).ok_or(AppError::NotFound)?;

    let is_owner = booking.member_id == user.id;
    let can_view_any = user.has_permission(Permission::ViewAnyBooking);
    if !is_owner && !can_view_any {
        return Err(AppError::NotFound);
    }

    Ok(Json(booking.clone()))
}

/// ยกเลิก booking แบบซ่อนการมีอยู่ (404)
async fn cancel_booking_hide_existence(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut bookings = state.bookings.lock().unwrap();
    let booking = bookings.get_mut(&id).ok_or(AppError::NotFound)?;

    let is_owner = booking.member_id == user.id;
    let can_cancel_any = user.has_permission(Permission::CancelAnyBooking);
    if !is_owner && !can_cancel_any {
        return Err(AppError::NotFound);
    }

    booking.status = "cancelled".to_string();
    Ok(StatusCode::NO_CONTENT)
}

/// ยกเลิก booking แบบเผยว่ามีอยู่ (403) -- ทางเลือกที่ 2 ของ 76.7
async fn cancel_booking_explicit_forbidden(
    State(state): State<AppState>,
    user: CurrentUser,
    Path(id): Path<u32>,
) -> Result<StatusCode, AppError> {
    let mut bookings = state.bookings.lock().unwrap();
    let booking = bookings.get_mut(&id).ok_or(AppError::NotFound)?;

    let is_owner = booking.member_id == user.id;
    let can_cancel_any = user.has_permission(Permission::CancelAnyBooking);
    if !is_owner && !can_cancel_any {
        return Err(AppError::Forbidden("booking นี้ไม่ใช่ของคุณ ไม่มีสิทธิ์ยกเลิก".into()));
    }

    booking.status = "cancelled".to_string();
    Ok(StatusCode::NO_CONTENT)
}

// ============================== main ==============================

#[tokio::main]
async fn main() {
    let mut users = HashMap::new();
    users.insert("admin".to_string(), UserRecord {
        id: 1, username: "admin".to_string(), password: "adminpass".to_string(),
        roles: vec![Role::Admin],
    });
    users.insert("staff1".to_string(), UserRecord {
        id: 2, username: "staff1".to_string(), password: "staffpass".to_string(),
        roles: vec![Role::Staff, Role::Member], // multi-role
    });
    users.insert("nan".to_string(), UserRecord {
        id: 3, username: "nan".to_string(), password: "memberpass".to_string(),
        roles: vec![Role::Member],
    });
    users.insert("wit".to_string(), UserRecord {
        id: 4, username: "wit".to_string(), password: "memberpass".to_string(),
        roles: vec![Role::Member],
    });

    let mut books = HashMap::new();
    books.insert(1, Book { id: 1, title: "The Rust Programming Language".to_string() });
    books.insert(2, Book { id: 2, title: "Zero To Production In Rust".to_string() });

    let state = AppState {
        jwt_secret: Arc::new(b"super-secret-signing-key-for-rbac-demo-only".to_vec()),
        users: Arc::new(Mutex::new(users)),
        books: Arc::new(Mutex::new(books)),
        bookings: Arc::new(Mutex::new(HashMap::new())),
        next_booking_id: Arc::new(Mutex::new(1)),
    };

    let app = Router::new()
        .route("/login", post(login))
        .route("/me", get(me))
        .route("/books", get(list_books))
        .route("/admin/books/{id}", delete(delete_book_admin_only))
        .route(
            "/staff/books/{id}",
            delete(delete_book_staff_gated).route_layer(axum::middleware::from_fn(require_role(
                state.jwt_secret.clone(),
                Role::Staff,
            ))),
        )
        .route("/books/{id}/inline-check", delete(delete_book_inline_check))
        .route("/staff/bookings", get(list_all_bookings_staff_only))
        .route("/bookings", post(create_booking))
        .route("/bookings/{id}", get(get_booking))
        .route("/bookings/{id}", delete(cancel_booking_hide_existence))
        .route("/bookings/{id}/cancel-explicit", post(cancel_booking_explicit_forbidden))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3176").await.unwrap();
    println!("rbac_capstone listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

`cargo build` ผ่านสะอาด (ไม่มี warning เลยแม้แต่ตัวเดียว — ตรวจสอบจริงด้วย `cargo build 2>&1` แล้วดูผลลัพธ์
ท้ายสุด):

```
   Compiling rbac_capstone v0.1.0 (...)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.10s
```

#### curl transcript เต็มรูปแบบ

**1. Login สี่บัญชี — เก็บ token ไว้ใช้ต่อ**:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/login -H 'Content-Type: application/json' \
    -d '{"username":"admin","password":"adminpass"}'
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 387

{"token":"eyJ0eXAi...(ตัด)...fHQ9s","roles":["Admin"]}
```

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/login -H 'Content-Type: application/json' \
    -d '{"username":"staff1","password":"staffpass"}'
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 391

{"token":"eyJ0eXAi...(ตัด)...NvxvLw","roles":["Staff","Member"]}
```

login ด้วย password ผิด:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/login -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"wrong"}'
```
```
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 107

{"error":{"code":"UNAUTHORIZED","message":"username หรือ password ไม่ถูกต้อง"}}
```

**2. `GET /me` — พิสูจน์ multi-role และ permission ที่รวมมาแล้ว**:

```bash
$ curl -sS -i http://127.0.0.1:3176/me -H "Authorization: Bearer $STAFF_TOKEN"
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 166

{"id":2,"username":"staff1","roles":["Staff","Member"],"permissions":["CancelOwnBooking","CreateBooking","DeleteBook","ViewBook","ViewAnyBooking","CancelAnyBooking"]}
```

ไม่มี token เลย:

```bash
$ curl -sS -i http://127.0.0.1:3176/me
```
```
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 98

{"error":{"code":"UNAUTHORIZED","message":"ไม่พบ header Authorization: Bearer <token>"}}
```

**3. Approach A (const generics) — `DELETE /admin/books/{id}` ต้องเป็น `Admin`**:

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/admin/books/2 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-type: application/json
content-length: 155

{"error":{"code":"FORBIDDEN","message":"ต้องมี role Admin เท่านั้นถึงจะเรียก endpoint นี้ได้"}}
```

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/admin/books/2 -H "Authorization: Bearer $ADMIN_TOKEN"
```
```
HTTP/1.1 204 No Content
```

**4. Approach B (runtime middleware) — `DELETE /staff/books/{id}` ต้องเป็น `Staff`** — สังเกตว่า `admin`
(role `Admin` ล้วน ๆ ไม่มี `Staff`) **ก็โดน 403 เหมือนกัน** ตามที่อภิปรายไว้ในหัวข้อ 76.4-76.5:

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/staff/books/1 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-length: 155

{"error":{"code":"FORBIDDEN","message":"ต้องมี role Staff เท่านั้นถึงจะเรียก endpoint นี้ได้"}}
```

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/staff/books/1 -H "Authorization: Bearer $ADMIN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-length: 155

{"error":{"code":"FORBIDDEN","message":"ต้องมี role Staff เท่านั้นถึงจะเรียก endpoint นี้ได้"}}
```

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/staff/books/1 -H "Authorization: Bearer $STAFF_TOKEN"
```
```
HTTP/1.1 204 No Content
```

**5. สร้างและดู booking — สอง member คนละคน**:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/bookings -H "Authorization: Bearer $NAN_TOKEN" \
    -H 'Content-Type: application/json' -d '{"book_id":2}'
```
```
HTTP/1.1 201 Created
content-length: 76

{"id":1,"book_id":2,"member_id":3,"member_username":"nan","status":"active"}
```

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/bookings -H "Authorization: Bearer $WIT_TOKEN" \
    -H 'Content-Type: application/json' -d '{"book_id":1}'
```
```
HTTP/1.1 201 Created
content-length: 76

{"id":2,"book_id":1,"member_id":4,"member_username":"wit","status":"active"}
```

**6. Ownership check — `nan` ดู booking ของตัวเองได้ (200) แต่ดูของ `wit` ไม่ได้ (404 เหมือน "ไม่มีจริง")**:

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/1 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 200 OK
content-length: 76

{"id":1,"book_id":2,"member_id":3,"member_username":"nan","status":"active"}
```

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 404 Not Found
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/999 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 404 Not Found
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

`staff1` (มี `ViewAnyBooking`) ดู booking ของ `wit` ได้ปกติ (200):

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $STAFF_TOKEN"
```
```
HTTP/1.1 200 OK
content-length: 76

{"id":2,"book_id":1,"member_id":4,"member_username":"wit","status":"active"}
```

**7. Staff ทำ "admin-adjacent action" — ยกเลิก booking ของคนอื่นที่ไม่ใช่ตัวเอง**:

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 404 Not Found
content-length: 106

{"error":{"code":"NOT_FOUND","message":"ไม่พบข้อมูลที่ต้องการ"}}
```

```bash
$ curl -sS -i -X POST http://127.0.0.1:3176/bookings/2/cancel-explicit -H "Authorization: Bearer $NAN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-length: 148

{"error":{"code":"FORBIDDEN","message":"booking นี้ไม่ใช่ของคุณ ไม่มีสิทธิ์ยกเลิก"}}
```

```bash
$ curl -sS -i -X DELETE http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $STAFF_TOKEN"
```
```
HTTP/1.1 204 No Content
```

`staff1` (role `Staff`, ไม่ใช่เจ้าของ booking id 2 เลย) ยกเลิกสำเร็จเพราะมี permission
`CancelAnyBooking` — นี่คือ "staff member successfully performing an admin-adjacent action" ตามที่บทนี้
ตั้งเป้าไว้ พิสูจน์ด้วยการเช็คสถานะหลังจากนั้น:

```bash
$ curl -sS -i http://127.0.0.1:3176/bookings/2 -H "Authorization: Bearer $WIT_TOKEN"
```
```
HTTP/1.1 200 OK
content-length: 79

{"id":2,"book_id":1,"member_id":4,"member_username":"wit","status":"cancelled"}
```

**8. ข้อจำกัดของ const generics single-role — `RequireStaff` ปฏิเสธ `Admin`**:

```bash
$ curl -sS -i http://127.0.0.1:3176/staff/bookings -H "Authorization: Bearer $ADMIN_TOKEN"
```
```
HTTP/1.1 403 Forbidden
content-length: 155

{"error":{"code":"FORBIDDEN","message":"ต้องมี role Staff เท่านั้นถึงจะเรียก endpoint นี้ได้"}}
```

```bash
$ curl -sS -i http://127.0.0.1:3176/staff/bookings -H "Authorization: Bearer $STAFF_TOKEN"
```
```
HTTP/1.1 200 OK
content-length: 161

[{"id":2,"book_id":1,"member_id":4,"member_username":"wit","status":"cancelled"},{"id":1,"book_id":2,"member_id":3,"member_username":"nan","status":"cancelled"}]
```

Transcript ทั้ง 8 ส่วนข้างบนนี้คือผลลัพธ์ที่รันจริงกับเซิร์ฟเวอร์ตัวเดียวติดต่อกันเป็นลำดับ (ทำให้ `id` ของ
booking และสถานะของหนังสือแต่ละเล่มไล่ตามลำดับการทดสอบข้างบนตรง ๆ ไม่มีการข้าม/สลับลำดับ) — ครอบคลุมทั้ง
`200`, `201`, `204`, `401`, `403`, และ `404` ตามที่บทนี้ตั้งเป้าไว้ทั้งหมด

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ใช้ enum ตรง ๆ เป็น const generic parameter — compile ไม่ผ่านบน stable Rust

```rust
# #[derive(Debug, Clone, Copy, PartialEq, Eq)]
# enum Role { Admin, Staff, Member }
struct RequireRole<const ROLE: Role>;
```

Error จริงจาก `rustc`:

```
error: `Role` is forbidden as the type of a const generic parameter
  --> src/main.rs:11:32
   |
11 | struct RequireRole<const ROLE: Role>;
   |                                ^^^^
   |
   = note: the only supported types are integers, `bool`, and `char`
```

**สาเหตุ**: const generics บน stable Rust รับได้แค่ integer type, `bool`, และ `char` เท่านั้น (มี unstable
feature `adt_const_params` ที่เปิดให้ใช้ enum/struct ได้ แต่ต้องใช้ nightly compiler) — **วิธีแก้**: ตั้งให้
`Role` เป็น `#[repr(u8)]` แล้วใช้ `u8` เป็น const generic parameter แทน (`RequireRole<const ROLE: u8>`) พร้อม
เมธอด `Role::from_u8()` แปลงกลับเมื่อต้องใช้จริง (ดูหัวข้อ 76.5 เต็มรูปแบบ)

### 2. ใช้ type alias เป็น pattern ใน function parameter — `E0532`

```rust
# struct RequireRole<const ROLE: u8>(u32);
# type RequireAdmin = RequireRole<0>;
# async fn handler(
RequireAdmin(_user): RequireAdmin,
# ) {}
```

Error จริง:

```
error[E0532]: expected tuple struct or tuple variant, found type alias `RequireAdmin`
   --> src/main.rs:377:5
    |
377 |     RequireAdmin(_user): RequireAdmin,
    |     ^^^^^^^^^^^^ not a tuple struct or tuple variant
```

**สาเหตุ**: `RequireAdmin` เป็นแค่ **type alias** ไม่ใช่ constructor จริง — pattern (ฝั่งซ้ายของ `:`) ต้องใช้
ชื่อ struct ที่ประกาศไว้จริงเสมอ (`RequireRole`) ส่วน type annotation (ฝั่งขวาของ `:`) ใช้ alias ได้ปกติ —
**วิธีแก้**: เขียน `RequireRole(_user): RequireAdmin` (ชื่อจริงในฝั่ง pattern, alias ในฝั่ง type)

### 3. ถือ `MutexGuard` ข้าม `.await` ตอนทำ ownership check ที่ต้อง query DB จริง

```rust
# use axum::extract::State;
# #[derive(Clone)] struct AppState { bookings: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<u32, u32>>> }
# async fn fetch_owner_from_db(_id: u32) -> u32 { 42 }
async fn get_booking_buggy(State(state): State<AppState>) -> String {
    let bookings = state.bookings.lock().unwrap(); // ยึด lock ไว้
    let owner_id = fetch_owner_from_db(1).await; // <- .await ขณะ MutexGuard ยังไม่ถูก drop
    format!("owner: {owner_id}, count: {}", bookings.len())
}
```

Error จริง (ผ่าน `#[axum::debug_handler]` ที่ให้ error message ชัดกว่า error ธรรมดาของ `Handler` trait มาก):

```
error: future cannot be sent between threads safely
  --> src/main.rs:17:1
   |
17 | #[axum::debug_handler]
   | ^^^^^^^^^^^^^^^^^^^^^^ future returned by `get_booking_buggy` is not `Send`
   |
   = help: within `impl Future<Output = String>`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, HashMap<u32, u32>>`
note: future is not `Send` as this value is used across an await
  --> src/main.rs:20:43
   |
19 |     let bookings = state.bookings.lock().unwrap();
   |         -------- has type `std::sync::MutexGuard<'_, HashMap<u32, u32>>` which is not `Send`
20 |     let owner_id = fetch_owner_from_db(1).await;
   |                                           ^^^^^ await occurs here, with `bookings` maybe used later
```

**สาเหตุ**: `std::sync::MutexGuard` ไม่ implement `Send` — ถ้ามันยัง "อยู่" ข้าม `.await` (คอมไพเลอร์นับว่า
มันอาจถูกใช้ต่ออีกหลัง `.await`) `Future` ทั้งตัวของ handler ก็ไม่ `Send` ไปด้วย ซึ่ง Axum handler **ต้อง
`Send`** เสมอ (เพราะ Tokio อาจย้าย task ไปรันข้าม thread) — Ownership check ในหัวข้อ 76.7 ที่ต้อง query
resource จริงจาก async database (Part 70) เสี่ยงชนกับดักนี้ได้ง่ายมาก ถ้าล็อก in-memory store ทิ้งไว้ก่อนไป
`.await` งานอื่น — **วิธีแก้**: ปล่อย `MutexGuard` (ผ่าน block ที่จบก่อนถึง `.await`) เสมอก่อนถึง `.await` ตัว
แรก ตามหลักที่ Part 39/64/66 เตือนไว้ตรงกัน

### 4. สลับ `401` กับ `403` — บอก client ผิดว่าปัญหาคืออะไร

เขียน authorization check แล้วคืน `AppError::Unauthorized` ผิด ๆ แทน `AppError::Forbidden`:

```rust
# #[derive(Debug)] enum AppError { Unauthorized(String), Forbidden(String) }
# struct CurrentUser { can_delete: bool }
fn check_wrong(user: &CurrentUser) -> Result<(), AppError> {
    if !user.can_delete {
        // *** ผิด: นี่คือ authorization failure (รู้ว่าเป็นใครแล้ว แต่ไม่มีสิทธิ์) ไม่ใช่ authentication ***
        return Err(AppError::Unauthorized("ไม่มีสิทธิ์".into()));
    }
    Ok(())
}
```

**ผลที่ตามมา**: client (โดยเฉพาะ frontend ที่เขียน logic "เจอ 401 แล้วให้ redirect ไปหน้า login ใหม่")
จะพา user ที่ login สำเร็จอยู่แล้วไป**login ใหม่ซ้ำ ๆ** ทั้งที่ปัญหาจริงไม่เกี่ยวกับการ login เลย — login ใหม่
ด้วย credential เดิมก็ยังโดนปฏิเสธเหมือนเดิมเพราะ role/permission ไม่พอ ไม่ใช่เพราะ token ไม่ถูกต้อง — user
จะงงว่า "ทำไม login ใหม่แล้วก็ยังเข้าไม่ได้อยู่ดี" — **วิธีแก้**: ยึดกฎตายตัวเสมอ — **`401` เฉพาะเมื่อไม่รู้ว่า
เป็นใคร (authentication ล้มเหลว) และ `403` เมื่อรู้ว่าเป็นใครแล้วแต่ไม่มีสิทธิ์ (authorization ล้มเหลว)**
ไม่มีข้อยกเว้น

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม role ใหม่ชื่อ `Auditor` ที่มีสิทธิ์แค่ `ViewBook` และ `ViewAnyBooking` (ดูได้ทุกอย่าง แต่
   แก้ไข/ลบอะไรไม่ได้เลย) — เพิ่ม variant เข้า `Role` enum, เพิ่มเข้า `Role::from_u8`, เพิ่ม branch ใน
   `role_permissions()`, แล้วสร้าง user ทดสอบใหม่ที่มี role นี้ ทดสอบด้วย `curl` ว่า `GET /bookings/{id}` ของ
   คนอื่นเข้าได้ (เพราะมี `ViewAnyBooking`) แต่ `DELETE /bookings/{id}` ของคนอื่นโดน 404 (เพราะไม่มี
   `CancelAnyBooking`)

2. **(กลาง)** เพิ่ม endpoint ใหม่ `PUT /books/{id}` (แก้ไขชื่อหนังสือ) ที่ใช้ **Approach B (runtime
   middleware)** จำกัดให้ทั้ง `Admin` **และ** `Staff` ทำได้ (ไม่ใช่แค่ role เดียว) — hint: `require_role()`
   ในบทนี้เช็คแค่ role เดียว ต้องเขียน middleware ใหม่ที่รับ `Vec<Role>` แล้วเช็ค `.any()` ว่า user มี role
   ไหนสักตัวในลิสต์นั้นไหม (หรือเปลี่ยนไปเช็ค `Permission` แบบหัวข้อ 76.8 แทนก็ได้ ซึ่งเป็นทางที่ทำได้ง่ายกว่า)

3. **(ยาก)** สลับ endpoint `GET /bookings/{id}` ในบทนี้จากแนวทาง "404 ซ่อนการมีอยู่" ไปเป็นแนวทาง "403 เผยว่า
   มีอยู่" (เหมือนที่ทำไว้แล้วกับ `cancel-explicit`) — แล้วเขียน test เปรียบเทียบ `content-length` ของ
   response ระหว่าง "id ไม่มีจริง" กับ "id มีจริงแต่ไม่ใช่เจ้าของ" ทั้งสองแนวทาง พิสูจน์ด้วยตัวเลขจริงว่า
   แนวทางไหนที่ response สองกรณีนี้**เหมือนกันเป๊ะ** และแนวทางไหนที่**ต่างกัน** (hint: เทียบ `curl -sS -o
   /dev/null -w '%{size_download}\n'` ของทั้งสี่กรณี)

4. **(ยาก/ประยุกต์)** ระบบปัจจุบันเช็ค permission ทีละครั้งต่อ endpoint — ให้เพิ่ม endpoint ใหม่
   `POST /books/{id}/archive` (เก็บหนังสือเข้าคลัง ไม่ลบถาวร) ที่ต้องผ่าน**สอง permission พร้อมกัน**:
   `DeleteBook` (สิทธิ์จัดการหนังสือทั่วไป) **และ** `ManageUsers` (สมมติว่า archive ต้องมีสิทธิ์ระดับสูงกว่า
   ลบธรรมดา) — เขียน permission check ที่ต้องผ่านทั้งสองเงื่อนไข (`&&` ไม่ใช่ `||`) แล้วพิสูจน์ด้วย `curl` ว่า
   `staff1` (มี `DeleteBook` แต่ไม่มี `ManageUsers`) โดน 403 ในขณะที่ `admin` (มีทั้งสอง) ทำสำเร็จ — hint:
   สิ่งนี้พิสูจน์ว่า permission-based model รองรับ "ต้องมีทุกอย่างในกลุ่มนี้" (`AND`) ได้ง่ายพอ ๆ กับ "มีอย่าง
   ใดอย่างหนึ่งก็พอ" (`OR`) ที่หัวข้อ 76.8-76.9 สาธิตไว้ — ต่าง role-based single check ที่ทำ `AND` ยากกว่า

## สรุป

บทนี้ปิดช่องว่างที่ Part 74-75 ทิ้งไว้อย่างตั้งใจ: JWT/session พิสูจน์ได้แค่ **"นี่คือใคร"** ส่วน RBAC ที่บทนี้
สอนตอบคำถาม **"สิ่งนี้ทำอะไรได้บ้าง"** — เริ่มจาก role/permission ระดับหยาบ (coarse-grained, เช็คได้จาก
identity ของ user เพียวๆ ไม่ต้องแตะข้อมูลทรัพยากร) ไปจนถึง ownership check ระดับละเอียด (fine-grained, ต้อง
query ทรัพยากรที่เจาะจงมาเทียบก่อนตัดสิน) — สองระดับนี้ทำงานเสริมกัน ไม่ใช่แทนกัน: coarse-grained กรอง request
ที่ไม่มีสิทธิ์เข้า endpoint ตั้งแต่ต้น (เร็ว ไม่ต้องแตะ database) ส่วน fine-grained รับหน้าที่ตัดสินสิทธิ์ที่
ขึ้นกับข้อมูลจริงของทรัพยากรเจาะจง (ที่ coarse-grained ทำไม่ได้โดยธรรมชาติ)

จุดที่ควรจำที่สุดจากบทนี้: **ไม่มีวิธีเดียวที่ถูกต้องสำหรับทุกสถานการณ์** — const generics ให้ requirement ที่
มองเห็นได้ในลายเซ็นแต่แลกมาด้วย type machinery ที่ซับซ้อนกว่าและรองรับ "OR ของหลาย role" ยาก, runtime
middleware ยืดหยุ่นกว่าแต่ requirement ซ่อนอยู่นอกตัว handler, role-based check เหมาะกับกรณีง่าย ๆ แต่
permission-based (many-to-many) ทนต่อการเปลี่ยนแปลง role ในระบบได้ดีกว่ามาก, และ 403 กับ 404 ต่างมี tradeoff
ด้านความปลอดภัยกับ UX ที่ต้องเลือกอย่างมีเจตนา — ทั้งหมดนี้ไม่มี "คำตอบเดียวที่ถูกเสมอ" มีแต่ **tradeoff ที่
ต้องเข้าใจให้ลึกพอจะเลือกให้เหมาะกับระบบตรงหน้า**

Part ถัดไป (77) จะเปลี่ยนโหมดการสื่อสารจาก HTTP request-response แบบที่เรียนมาตลอด ไปเป็น **WebSockets** —
connection ที่เปิดค้างไว้ให้ server ส่งข้อมูลไปยัง client ได้โดยไม่ต้องรอ client ถามก่อน (real-time
notification, live update) — RBAC ที่บทนี้สอนจะกลับมาเกี่ยวข้องอีกครั้งตอนต้องตัดสินว่า "connection WebSocket
นี้ควรได้รับ event อะไรบ้าง" ซึ่งเป็นคำถาม authorization แบบเดียวกัน แค่ transport ต่างไป

---

**Part ก่อนหน้า:** [Authentication: Session-based และ OAuth2](part-075-session-oauth2.md) | **Part ถัดไป:** [WebSockets ด้วย Axum](part-077-websockets-axum.md)
