# Part 74: Authentication: JWT

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 280 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างถูกต้องและพิสูจน์ได้ด้วยโค้ดจริงว่า **JWT (JSON Web Token) ไม่ใช่การเข้ารหัส (encryption)** —
  มันคือข้อมูล JSON ที่เข้ารหัสฐาน (encode) ด้วย Base64URL แล้ว**เซ็นชื่อ (sign)** ไว้เพื่อพิสูจน์ว่าไม่ถูกแก้ไข
  ไม่ใช่เพื่อปิดไม่ให้อ่าน — ใครก็ตามที่มี token ในมือ (แม้ไม่มี secret key เลย) **อ่าน payload ข้างในได้ทันที**
  ด้วยการ decode base64url ธรรมดา ๆ ไม่ต้องแฮกอะไรเลย
- แยกแยะและ implement การเซ็น JWT ทั้งสองแบบด้วย crate `jsonwebtoken` จริง: **HS256** (symmetric — secret
  เดียวใช้ทั้งเซ็นและตรวจ เหมาะกับระบบ monolith ที่มี backend เดียว) และ **RS256** (asymmetric — private key
  เซ็น, public key ตรวจ เหมาะกับระบบที่ต้องแจก "ความสามารถตรวจสอบ token" ให้หลาย service โดยไม่ให้ service
  เหล่านั้นปลอมสร้าง token ใหม่ได้ — ปูทางไปสู่ Part 81 เรื่อง microservices)
- ออกแบบ `Claims` struct ที่ผสาน standard claims ตามมาตรฐาน RFC 7519 (`sub`, `exp`, `iat`, `nbf`, `aud`,
  `iss`) เข้ากับ custom claims ของแอปเอง (`role`, `username`) ด้วย `#[derive(Serialize, Deserialize)]`
  ตามที่ Part 57 สอนไว้ ให้ตรงกับ use case จริงของระบบ
- เขียนระบบ signup/login ที่ **hash password ด้วย `argon2` ก่อนเก็บลง storage เสมอ ไม่มีข้อยกเว้น** — ไม่เคย
  เก็บหรือเทียบ password เป็น plaintext แม้แต่จุดเดียว — แล้วออก JWT คู่ (access + refresh) ให้ผู้ใช้ที่ login
  สำเร็จ ด้วยโค้ดที่ compile และรันได้จริง ทดสอบด้วย `curl` จริงทุกเส้นทาง
- เขียน **custom Axum extractor** (`CurrentUser`) ที่ implement `FromRequestParts` ตามแพทเทิร์นที่ Part 64
  สอนไว้ เพื่ออ่าน header `Authorization: Bearer <token>`, ตรวจสอบลายเซ็นและวันหมดอายุผ่าน
  `jsonwebtoken::decode`, แล้วส่ง `Claims`/`CurrentUser` เข้า handler โดยอัตโนมัติ — พร้อมพิสูจน์ด้วย `curl`
  จริงครบ 5 สถานการณ์ (token ถูก, ไม่มี header, token ผิดรูปแบบ, token หมดอายุ, token ถูกแก้ไข signature)
  รวม **error variant จริงจาก `jsonwebtoken`** ที่ capture มาทุกกรณี
- ระบุ**กับดักด้านความปลอดภัยที่ร้ายแรงที่สุดของ JWT** ได้อย่างเจาะจง (algorithm confusion / `alg: none`,
  การปิด validation ของ `exp` เอง, การเก็บ token ฝั่ง client แบบไม่ปลอดภัย, การ commit secret เข้า source
  control) พร้อมพิสูจน์แต่ละข้อด้วยโค้ดและ error message จริง ไม่ใช่แค่ท่องจำว่า "ห้ามทำ"

## ความรู้ที่ต้องมีมาก่อน

- **Part 64 (Axum: State Management และ Extractors)**: บทนี้เขียน `CurrentUser` เป็น custom extractor
  โดย implement `FromRequestParts<S>` ตรงตามแพทเทิร์นที่ Part 64 หัวข้อ 64.6 สอนไว้ทุกประการ (แค่เปลี่ยนจาก
  ตรวจ header ธรรมดาเป็นตรวจ JWT) — ถ้ายังไม่แน่นเรื่อง `FromRequestParts` vs `FromRequest`, `Rejection`
  ที่ implement `IntoResponse` เอง, หรือทำไม extractor ไม่ต้องมี `#[async_trait]` ใน Axum เวอร์ชันปัจจุบัน
  ควรกลับไปทวนก่อน เพราะบทนี้จะไม่อธิบายกลไกพื้นฐานของ extractor ซ้ำจากศูนย์
- **Part 66 (Axum: Error Handling แบบมืออาชีพ)**: JSON shape ของ error ที่บทนี้ตอบกลับ
  (`{"error": {"code": "...", "message": "..."}}`) ใช้รูปแบบเดียวกับ `AppError` ที่ Part 66 ออกแบบไว้ทุก
  ประการ — เพียงแต่บทนี้มี error type ของตัวเอง (`JwtRejection`) ที่ทำหน้าที่เดียวกันเฉพาะสำหรับ auth
- **Part 70 (Database: เชื่อมต่อ PostgreSQL ด้วย SQLx)**: บทนี้ใช้ `HashMap<String, UserRecord>` ใน
  หน่วยความจำแทนตาราง `users` จริง (อธิบายเหตุผลของการตัดสินใจนี้อย่างตรงไปตรงมาในหัวข้อ 74.6) — ถ้าคุณเข้าใจ
  pattern `PgPool` + `AppState` + query parameterized จาก Part 70 มาแล้ว จะเห็นได้ทันทีว่าการสลับจาก
  `HashMap` เป็น query จริงทำตรงไหนบ้าง (แบบฝึกหัดข้อ 2 ท้ายบทให้ลองทำจริง)
- **Part 61 (HTTP Fundamentals)**: บทนี้อ้างอิง header `Authorization`, status code `200`/`401`/`403`/`409`
  ตามความหมายมาตรฐานที่ Part 61 สอนไว้ตรง ๆ
- **Part 57 (Serialization: Serde เบื้องต้น)**: `Claims` และ struct request/response ทั้งหมดในบทนี้ใช้
  `#[derive(Serialize, Deserialize)]` ตามที่ Part 57 สอนไว้
- **Part 12 และ Part 30-31 (Error Handling)**: `JwtRejection` เป็น enum ที่ implement `IntoResponse` ตรงตาม
  หลักการ custom error type ที่ Part 30 สอนไว้ (แต่ target คือ `IntoResponse` ของ Axum ไม่ใช่
  `std::error::Error` — เหมือนกับ `ApiKeyRejection` ที่ Part 64 หัวข้อ 64.6 ทำไว้ก่อนแล้ว)
- **บทนี้เป็นจุดเริ่มต้นของหัวข้อ Authentication ทั้งหมดในหลักสูตร** — Part 75 จะสอน session-based auth และ
  OAuth2 เทียบกับ JWT ที่บทนี้สอน, Part 76 จะสอน RBAC (Role-Based Access Control) ต่อยอดจาก claim `role`
  ที่บทนี้ออกแบบไว้, Part 81 (Microservices) จะใช้แนวคิด asymmetric signing ที่บทนี้แนะนำไว้เต็มรูปแบบ, และ
  Part 83 (Redis) จะเป็นที่ที่เราเก็บ state ของ refresh token แบบ revocable จริงจัง (บทนี้แค่ sketch แนวคิดไว้)

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ทุกตัวอย่างในบทนี้ผู้เขียน **compile และรันจริง** ด้วย scratch project แยกไว้นอก repo ของหลักสูตร (ไม่กระทบ
ไฟล์ใด ๆ ในโค้สนี้เลย) ใช้เวอร์ชันจริงจาก crates.io ณ วันที่เขียนบทนี้:

```toml
[dependencies]
axum = "0.8.9"
tokio = { version = "1.53.1", features = ["full"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
argon2 = "0.6.0"
chrono = { version = "0.4.45", features = ["serde"] }
base64 = "0.22.1"
```

**การตัดสินใจที่สำคัญที่สุดของบทนี้ที่ต้องบอกตรง ๆ ก่อนเริ่ม**: บทนี้ใช้ `HashMap<String, UserRecord>` ห่อ
ด้วย `Arc<Mutex<...>>` เป็น "ตาราง users" แทนการต่อ PostgreSQL จริงแบบที่ Part 70 สอนไว้ — **นี่คือการตัดสินใจ
ที่ตั้งใจทำ ไม่ใช่ทางลัดที่ไม่มีเหตุผล**: หัวใจของบทนี้คือกลไกของ JWT (การเซ็น การตรวจ การออกแบบ claims การ
จัดการ token หมดอายุ/refresh) ซึ่งเป็นเรื่องที่**แยกออกจากชนิดของ storage ที่เก็บ user โดยสิ้นเชิง** — ไม่ว่า
`find_user_by_username` จะไปอ่านจาก `HashMap` หรือจาก `PgPool::query_as!` ตรรกะเรื่อง JWT ทั้งหมดในบทนี้ทำงาน
เหมือนกันทุกประการ การใช้ `HashMap` ทำให้เนื้อหาโฟกัสที่ auth ล้วน ๆ โดยไม่ต้องแบกภาระเรื่อง migration/connection
pool มาปนกัน (แบบฝึกหัดข้อ 2 ท้ายบทให้คุณลองสลับเป็น `PgPool` จริงด้วยตัวเอง โดยใช้ pattern จาก Part 70 ตรง ๆ)

ทุกอย่างที่อ้างผลลัพธ์ (`cargo build`, `cargo run`, error message, JWT ที่ออกจริง, hash ที่ argon2 คำนวณจริง,
`curl` transcript) ในบทนี้คือผลลัพธ์ที่**รันจริงแล้วคัดลอกมา** ไม่มีการแต่งขึ้นเอง รวมถึง error variant ของ
`jsonwebtoken` สำหรับ token หมดอายุ/token ถูกแก้ไข signature/token ผิดรูปแบบ ที่หลายบทความออนไลน์มักจะ "เดา"
ข้อความ error แทนการรันจริง — บทนี้ capture ของจริงมาให้หมด

## เนื้อหา

### 74.1 JWT คืออะไรจริง ๆ: มันไม่ใช่การเข้ารหัส

ก่อนแตะโค้ดสักบรรทัด ต้องเคลียร์ความเข้าใจผิดที่พบบ่อยที่สุดเกี่ยวกับ JWT ก่อน เพราะความเข้าใจผิดนี้เป็นต้นตอ
ของช่องโหว่ด้านความปลอดภัยจำนวนมากในระบบจริง:

> **JWT (JSON Web Token) ไม่ได้ "เข้ารหัส" (encrypt) ข้อมูลข้างในเลย — มันแค่ "เซ็นชื่อ" (sign) ข้อมูลนั้น**

ความต่างระหว่างสองคำนี้สำคัญมาก:

- **Encryption (การเข้ารหัส)** — แปลงข้อมูลให้ **อ่านไม่ออก** ถ้าไม่มี key ที่ถูกต้อง เป้าหมายคือ
  **ความลับ (confidentiality)**
- **Signing (การเซ็นชื่อ)** — แปลงข้อมูลให้ **พิสูจน์ได้ว่าไม่ถูกแก้ไข** และ **มาจากคนที่ครอบครอง key จริง**
  แต่**ไม่ได้ปิดไม่ให้อ่าน**เลย เป้าหมายคือ **ความสมบูรณ์ (integrity) และการยืนยันแหล่งที่มา (authenticity)**
  ไม่ใช่ความลับ

JWT มาตรฐาน (ที่ใช้ `alg` แบบ `HS256`/`RS256`/`ES256` ซึ่งเป็นส่วนใหญ่ของ JWT ที่เจอในโลกจริง) ทำแค่การ**เซ็น**
เท่านั้น — มีอีกมาตรฐานที่เรียกว่า **JWE (JSON Web Encryption)** ที่เข้ารหัสจริง แต่ใช้กันน้อยกว่ามากและมี
โครงสร้างต่างออกไป บทนี้ (และโลกจริงส่วนใหญ่ที่พูดถึง "JWT") หมายถึง **JWS (JSON Web Signature)** ที่แค่เซ็น
ไม่เข้ารหัส

#### โครงสร้างจริงบนสาย (wire format)

JWT คือ **string เดียว** ที่ประกอบจากสามส่วนคั่นด้วยจุด (`.`):

```
header.payload.signature
```

แต่ละส่วน:

1. **header** — JSON บอกว่า token นี้ใช้ algorithm อะไรเซ็น (`alg`) และเป็น token ชนิดไหน (`typ`) เช่น
   `{"typ":"JWT","alg":"HS256"}`
2. **payload** — JSON ที่มี **claims** (ข้อมูลที่ต้องการส่ง เช่น `sub`, `exp`, `role`) — **นี่คือส่วนที่คนเข้าใจ
   ผิดบ่อยที่สุดว่าถูก "ป้องกัน" ไว้ — มันไม่ถูกป้องกันเลย**
3. **signature** — ผลลัพธ์ของฟังก์ชัน cryptographic (เช่น HMAC-SHA256 สำหรับ `HS256`) ที่คำนวณจาก
   `base64url(header) + "." + base64url(payload)` กับ secret/private key — ใช้**ตรวจสอบ**เท่านั้นว่าสอง
   ส่วนแรกไม่ถูกแก้ไขไปจากตอนที่เซ็น

แต่ละส่วนถูก **encode ด้วย Base64URL** (ไม่ใช่ Base64 ธรรมดา) ก่อนต่อกัน — ต่างจาก Base64 ธรรมดาสองจุด: (1)
ใช้ `-`/`_` แทน `+`/`/` เพื่อไม่ชนกับตัวอักษรที่มีความหมายพิเศษใน URL/header และ (2) **ไม่มี padding
(`=`)** — เหตุผลที่เลือกแบบนี้เพราะ JWT มักถูกส่งผ่าน URL query string หรือ HTTP header ที่ตัวอักษรบางตัวมี
ความหมายพิเศษ (encode/decode เพิ่มไม่ต้อง escape) — แต่ **Base64URL ก็ยังเป็นแค่ encoding ไม่ใช่ encryption**
เหมือน Base64 ธรรมดา 100% — ใครก็ decode กลับได้โดยไม่ต้องมี key อะไรเลย

### 74.2 พิสูจน์ด้วยโค้ดจริง: ถอด JWT ด้วยมือ

มาพิสูจน์ข้อความข้างบนด้วยโค้ดจริง ไม่ใช่แค่คำกล่าวลอย ๆ — สร้าง JWT ด้วย `jsonwebtoken::encode` ปกติ แล้ว
**ไม่ใช้ `jsonwebtoken::decode` เลย** แต่ split string ด้วยมือ แล้ว decode แต่ละส่วนด้วย `base64` crate ตรง ๆ
เพื่อดูว่าจะเห็นอะไรบ้างโดยไม่มี secret key เลยแม้แต่นิดเดียว:

```rust
use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine as _};
use jsonwebtoken::{encode, Algorithm, EncodingKey, Header};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    role: String,
    exp: usize,
    iat: usize,
}

fn main() {
    let claims = Claims {
        sub: "user-42".to_string(),
        role: "member".to_string(),
        iat: 1_790_000_000,
        exp: 1_790_003_600,
    };

    let header = Header::new(Algorithm::HS256);
    let key = EncodingKey::from_secret(b"super-secret-signing-key-for-demo-only");
    let token = encode(&header, &claims, &key).unwrap();

    println!("token เต็ม:\n{token}\n");

    // แยก token ด้วยมือ -- ไม่เรียก jsonwebtoken::decode เลยแม้แต่ครั้งเดียว
    let parts: Vec<&str> = token.split('.').collect();
    println!("จำนวนส่วนที่คั่นด้วยจุด: {}", parts.len());
    println!("header (base64url):  {}", parts[0]);
    println!("payload (base64url): {}", parts[1]);
    println!("signature (base64url): {}\n", parts[2]);

    // decode สองส่วนแรกด้วย base64url ธรรมดา -- ไม่ต้องมี key ใด ๆ เลย
    let header_bytes = URL_SAFE_NO_PAD.decode(parts[0]).unwrap();
    let payload_bytes = URL_SAFE_NO_PAD.decode(parts[1]).unwrap();

    println!("header decode แล้ว:\n{}\n", String::from_utf8(header_bytes).unwrap());
    println!("payload decode แล้ว (readable ตรง ๆ ไม่ต้องมี key เช่นกัน!):\n{}\n",
        String::from_utf8(payload_bytes).unwrap());
}
```

รันจริงด้วย `cargo run` ได้ผลลัพธ์ตรงตามที่อธิบายไว้ทุกประการ:

```
token เต็ม:
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyLTQyIiwicm9sZSI6Im1lbWJlciIsImV4cCI6MTc5MDAwMzYwMCwiaWF0IjoxNzkwMDAwMDAwfQ.8Bgcy9PN4mhvq_47D6ic5tSFK20ymk6hhflqMLcDQnM

จำนวนส่วนที่คั่นด้วยจุด: 3
header (base64url):  eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
payload (base64url): eyJzdWIiOiJ1c2VyLTQyIiwicm9sZSI6Im1lbWJlciIsImV4cCI6MTc5MDAwMzYwMCwiaWF0IjoxNzkwMDAwMDAwfQ
signature (base64url): 8Bgcy9PN4mhvq_47D6ic5tSFK20ymk6hhflqMLcDQnM

header decode แล้ว:
{"typ":"JWT","alg":"HS256"}

payload decode แล้ว (readable ตรง ๆ ไม่ต้องมี key เช่นกัน!):
{"sub":"user-42","role":"member","exp":1790003600,"iat":1790000000}
```

**สังเกตให้ชัด**: เราไม่ได้แตะ `super-secret-signing-key-for-demo-only` เลยแม้แต่บรรทัดเดียวในโค้ดถอดรหัส —
แค่ `split('.')` แล้ว `base64url decode` ธรรมดา ก็เห็น `role: "member"`, `sub: "user-42"` ชัดเจนแบบ plain text
JSON ทันที **ใครก็ตามที่มี token นี้ในมือ (ไม่ว่าจะขโมยมาจากไหน) อ่านข้อมูลนี้ได้เหมือนกันเป๊ะ** — ลองก็อปปี้
`token เต็ม` ข้างบนไปวางในเว็บไซต์ [jwt.io](https://jwt.io) (ไม่ต้องใส่ secret key อะไรเลย) จะเห็น payload
เดียวกันนี้ปรากฏขึ้นทันที เว็บไซต์นั้นทำสิ่งเดียวกับโค้ดข้างบนนี้เป๊ะ

ส่วนที่**decode ไม่ออกเป็น JSON** คือ **signature** (ส่วนที่ 3) — เพราะมันไม่ใช่ JSON ที่ถูก encode ไว้ มันคือ
**binary output ดิบ ๆ ของฟังก์ชัน HMAC-SHA256** ที่คำนวณจาก `header.payload` กับ secret key แล้วเอา binary
นั้นมา base64url encode อีกที — ต่อให้ decode base64url ของมันสำเร็จ (มันจะสำเร็จ เพราะ base64url decode ไม่
สน "ความหมาย" ของ byte ที่ได้) ก็จะได้ byte สุ่ม ๆ ที่ไม่ใช่ JSON ไม่มีทางอ่านเป็นข้อความได้ — **นี่คือส่วน
เดียวที่ "ปลอดภัย" ในความหมายที่ว่าปลอมมันไม่ได้ (ถ้าไม่รู้ secret key)** แต่ก็ยังไม่ใช่ "ความลับ" เพราะ
signature ไม่ได้มีข้อมูลอะไรที่ต้องปิดบังอยู่แล้ว

**ผลตามมาที่ต้องจำไว้เสมอ: ห้ามใส่ข้อมูลลับ (password, PII ที่ละเอียดอ่อน, credit card, internal system
detail) ลงใน JWT payload เด็ดขาด** — ไม่ว่าจะเซ็นด้วย algorithm ที่แข็งแรงแค่ไหนก็ตาม เพราะ payload อ่านออก
เสมอโดยไม่ต้องมี key อะไรเลย ถ้าต้องการความลับจริง ๆ ต้องใช้ JWE (นอกสโคปบทนี้) หรือง่ายกว่าคือ**อย่าเก็บข้อมูล
ลับใน token — เก็บแค่ตัวชี้ (เช่น `user_id`) แล้วให้ server ไป query รายละเอียดจากฐานข้อมูลเอาเองทุกครั้งที่
ต้องใช้**

### 74.3 HS256: Symmetric Signing — Secret เดียวใช้ทั้งเซ็นและตรวจ

ทีนี้มาดูว่า "เซ็น" จริง ๆ ทำงานอย่างไร เริ่มจากแบบที่ง่ายที่สุด: **HS256** (HMAC ผสม SHA-256)

**แนวคิด**: มี secret key **หนึ่งตัว** (เป็น byte string ธรรมดา ไม่ใช่ key pair) ที่ทั้ง**เซ็น**และ**ตรวจ**
token ใช้ตัวเดียวกัน — ฟังก์ชัน HMAC-SHA256 รับ (ข้อมูล, secret) แล้วคำนวณ signature ที่**เฉพาะคนที่มี secret
ตัวเดียวกันเท่านั้น**ที่คำนวณค้ำได้ตรงกัน — ตรวจสอบ token จึงทำโดยคำนวณ HMAC ใหม่จาก header+payload ของ token
ที่ได้รับมา ด้วย secret ตัวเดียวกัน แล้วเทียบว่าตรงกับ signature ที่แนบมาไหม

**เหมาะกับสถานการณ์ไหน**: ระบบที่มี **backend เดียว** (monolith หรือกลุ่ม service เล็ก ๆ ที่ deploy ด้วยกัน
และแชร์ secret ตัวเดียวกันได้อย่างปลอดภัย) — ทุกจุดที่ต้องเซ็นหรือตรวจ token ใช้ secret ตัวเดียวกันหมด ตั้งค่า
ง่าย ไม่ต้องจัดการ key pair — **ข้อเสีย**: ถ้าต้องแจกความสามารถ "ตรวจสอบ token" ให้ service อื่นที่ไม่ควรมี
สิทธิ์ "สร้าง token ใหม่" (เช่น service ฝั่ง frontend-facing ที่แค่ต้องตรวจว่า user login แล้วหรือยัง) คุณทำ
ไม่ได้เลย เพราะการแจก secret ตัวเดียวกันไปให้ service อื่น = แจกความสามารถ "เซ็น token ปลอม" ไปด้วยเสมอ

#### Implement จริง

```rust
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    role: String,
    exp: usize,
}

fn hs256_roundtrip() {
    let secret = b"super-secret-signing-key-for-demo-only";

    // --- เซ็น (encode) ---
    let header = Header::new(Algorithm::HS256);
    let key = EncodingKey::from_secret(secret);
    let claims = Claims { sub: "user-42".into(), role: "member".into(), exp: 9_999_999_999 };
    let token = encode(&header, &claims, &key).unwrap();

    // --- ตรวจ (decode + validate) ---
    let decoding_key = DecodingKey::from_secret(secret); // secret ตัวเดียวกัน
    let validation = Validation::new(Algorithm::HS256);
    let decoded = decode::<Claims>(&token, &decoding_key, &validation).unwrap();

    println!("{:?}", decoded.claims);
}
```

**สังเกต**: `EncodingKey::from_secret(secret)` และ `DecodingKey::from_secret(secret)` รับ **byte slice
เดียวกัน** — นี่คือความหมายของคำว่า "symmetric" ตรง ๆ ไม่มีความซับซ้อนเรื่อง key pair เลย

#### กับดักเฉพาะเวอร์ชันที่ต้องรู้: `jsonwebtoken` 11.x ต้องเลือก crypto backend เอง

ถ้าคุณ `cargo add jsonwebtoken` แล้วเรียก `encode`/`decode` ตรง ๆ โดยไม่ได้อ่านอะไรเพิ่ม จะเจอ **panic ทันที
ตอน runtime** (ไม่ใช่ compile error) ที่หลายคนงงมาก เพราะ `jsonwebtoken` เวอร์ชัน 11 ขึ้นไปแยก
implementation ของ cryptographic primitive ออกเป็น "backend" ที่ต้องเลือกเองผ่าน Cargo feature (`rust_crypto`
สำหรับ implementation ล้วน Rust หรือ `aws_lc_rs` สำหรับ binding ไปยัง AWS-LC) — ผู้เขียนรันจริงแล้วเจอ panic
นี้ก่อนจะรู้ต้องเปิด feature:

```
thread 'main' panicked at .../jsonwebtoken-11.1.0/src/crypto/mod.rs:124:40:

Could not automatically determine the process-level CryptoProvider from jsonwebtoken crate features.
Call CryptoProvider::install_default() before this point to select a provider manually, or make sure
exactly one of the 'rust_crypto' and 'aws_lc_rs' features is enabled.
See the documentation of the CryptoProvider type for more information.
```

วิธีแก้คือเปิด feature ที่ต้องการตอน `cargo add`:

```bash
cargo add jsonwebtoken --features rust_crypto
```

ผลลัพธ์จริง (ตัดบางส่วนเพื่อความกระชับ — เพิ่ม dependency สาย crypto มาหลายตัวเพราะ `rust_crypto` implement
ทั้ง HMAC/RSA/ECDSA/EdDSA ล้วน Rust เอง ไม่พึ่งไลบรารี C ภายนอก):

```
      Adding jsonwebtoken v11.1.0 to dependencies
             Features:
             + rust_crypto
      Adding hmac v0.12.1
      Adding rsa v0.9.10
      Adding p256 v0.13.2
      Adding p384 v0.13.1
      Adding ed25519-dalek v2.2.0
      Adding sha2 v0.10.9
      ... (ตัดบางส่วน)
```

หลังเปิด feature นี้แล้ว `encode`/`decode` ทำงานได้ตามปกติทันที ไม่มี panic อีก — **นี่คือตัวอย่างที่ดีของ
"เปลี่ยนแปลงเชิง breaking" ที่ไม่แสดงผลตอน `cargo build` แต่ปะทุตอน runtime แทน** (เพราะ `CryptoProvider` ถูก
resolve แบบ dynamic ตอนเรียก `encode`/`decode` ครั้งแรกจริง ๆ ไม่ใช่ตอน compile) — ถ้าเจอ panic แบบนี้กับ
crate เวอร์ชันใหม่กว่าที่ระบุในบทนี้ ให้ตรวจสอบ changelog ของ `jsonwebtoken` เสมอ เพราะพฤติกรรมนี้อาจเปลี่ยน
ต่อไปได้อีกในเวอร์ชันอนาคต — ecosystem ของ Rust อัปเดตเร็ว การอ้าง error message ในบทนี้คือ snapshot ของเวอร์ชัน
11.1.0 ณ วันที่เขียน ไม่ใช่ตัวเลขตายตัวตลอดไป

### 74.4 RS256/ES256: Asymmetric Signing — Private Key เซ็น, Public Key ตรวจ

ทีนี้มาดูอีกฝั่ง: **RS256** (RSA + SHA-256) และ **ES256** (ECDSA บน elliptic curve P-256 + SHA-256) —
ทั้งสองอยู่ในกลุ่ม **asymmetric signing** ที่ใช้ **key pair** (private key + public key) แทน secret เดียว

**แนวคิด**: **private key เท่านั้นที่เซ็น token ได้** ส่วน **public key ตรวจสอบลายเซ็นได้อย่างเดียว
ไม่สามารถย้อนกลับไปเซ็น token ใหม่ได้เลย** (นี่คือคุณสมบัติทาง cryptography ของ RSA/ECDSA — การมี public key
ไม่ได้ให้ข้อมูลพอที่จะคำนวณ private key กลับ หรือปลอมลายเซ็นได้)

**เหมาะกับสถานการณ์ไหน**: ระบบที่มีหลาย service ที่ต้อง**ตรวจสอบ**ว่า token ถูกต้องไหม แต่**ไม่ควรมีสิทธิ์
สร้าง token ใหม่** — เช่นสถาปัตยกรรม microservices ที่ Part 81 จะสอนเต็มรูปแบบ: มี **auth service ตัวเดียว**
ที่ครอบครอง private key และมีหน้าที่ออก token เท่านั้น ส่วน service อื่น ๆ ทั้งหมด (order service, payment
service, inventory service) ได้รับแค่ **public key** (แจกจ่ายแบบเปิดได้เลย ไม่ต้องเก็บเป็นความลับเหมือน
private key) ไปตรวจสอบ token ที่ client แนบมา — ถ้า order service ถูกแฮก แฮกเกอร์ที่ได้ public key ไป**ไม่มี
ทางปลอมสร้าง token ใหม่ที่ผ่านการตรวจได้เลย** เทียบกับ HS256 ที่ถ้า secret เดียวกันหลุดจาก service ไหนก็ตาม
ทุก service ที่แชร์ secret นั้นตกอยู่ในความเสี่ยงทันที

#### Implement จริงด้วย RSA key pair จริง

สร้าง key pair จริงด้วย `openssl` (เครื่องมือมาตรฐานที่ทุกเครื่อง dev มักมีติดตั้งอยู่แล้ว):

```bash
openssl genrsa -out rs256_private.pem 2048
openssl rsa -in rs256_private.pem -pubout -out rs256_public.pem
```

ได้ private key (เก็บเป็นความลับสูงสุด — มีแค่ auth service เท่านั้นที่ควรเข้าถึงไฟล์นี้) และ public key
(แจกจ่ายได้อย่างเปิดเผย):

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxB8KuWaIHFO8vmWLAt9F
Si33pruqYDfVhwSMstK/yvTay36EtyiWYoDWjnLZ1SRRdWZMU4t/fR6wdT4Phaxh
B7+YbI2edwAC8zMveYUQiG49lBQsRJctosg4UihhKMyHx8x1j5lJ4ehu3sWIRdis
lTKZwiy4atjyzfasg++h+E2KHH9BwX5fSUEu0dWo0TkT33tdesIHkFb4L2VJIxbO
g6Wrq5Wmf0xs3vvjDa5c0YZnUfREN9WMzngSygiShgz+qvs65RXcrQkcgxUFX4R+
LdVknZ6441KJfy4PC6rO0jV9+G5uEpHx38OlFZMeDm/iTQh0QeKcwXrFzxy7QVpa
YwIDAQAB
-----END PUBLIC KEY-----
```

โค้ด Rust ที่โหลดทั้งสอง key มาเซ็น/ตรวจสอบจริง:

```rust
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    role: String,
    exp: usize,
}

fn rs256_roundtrip() {
    let private_pem = std::fs::read("keys/rs256_private.pem").unwrap();
    let public_pem = std::fs::read("keys/rs256_public.pem").unwrap();

    let encoding_key = EncodingKey::from_rsa_pem(&private_pem).unwrap();
    let decoding_key = DecodingKey::from_rsa_pem(&public_pem).unwrap();

    let claims = Claims {
        sub: "user-42".to_string(),
        role: "admin".to_string(),
        exp: (chrono::Utc::now().timestamp() + 3600) as usize,
    };

    // เซ็นด้วย private key เท่านั้น -- เฉพาะ auth service ที่ครอบครอง private key ตัวนี้ทำได้
    let header = Header::new(Algorithm::RS256);
    let token = encode(&header, &claims, &encoding_key).unwrap();

    // ตรวจสอบด้วย public key เท่านั้น -- service อื่น ๆ ที่ได้รับแค่ public key
    // ตรวจลายเซ็นได้ แต่ "สร้าง" token ปลอมที่ผ่านการตรวจไม่ได้เลย เพราะไม่มี private key
    let mut validation = Validation::new(Algorithm::RS256);
    validation.set_required_spec_claims(&["exp", "sub"]);
    let decoded = decode::<Claims>(&token, &decoding_key, &validation).unwrap();
    println!("ตรวจสอบด้วย public key สำเร็จ: {:?}", decoded.claims);
}
```

รันจริง ได้ token จริงและตรวจสอบผ่านจริง:

```
RS256 token:
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyLTQyIiwicm9sZSI6ImFkbWluIiwiZXhwIjoxNzkwNDcyNDk0fQ.So91v8825ORUVX9cObTzr2OLSWjZNcVgfqauekg1hYU9xFlrmuEWbYr9KNbXrKeaxcBBVMGm7zntx0fmKKC30IxG8b0CaZZpAt-IkWP3lNp_pX9n3TO4_RdwEyNzWyqfr-rl3mlfxANSEkBFTNpfwO7GU3sIZ89-UMnU4panfEJC2oyQQ85V1ajL8eVGl_HOZCUpnT99OL6ZlK3SGzcUh-lDjpYhlhHPrj5xLPaP1A_jgkDK45raouSOLW7MClwvYsQC2MvnSGW8uvueIrQvAGaNmXcJytfJU0yKq7GJMnW7zf24adisgdb3MRUiZpxVvYjqrXk0cM46K5UoBxfuOg

ตรวจสอบด้วย public key สำเร็จ: Claims { sub: "user-42", role: "admin", exp: 1790472494 }
```

**พิสูจน์ว่า public key เซ็น token ไม่ได้จริง**: ลองเอา public key มาพยายามใช้เป็น encoding key (ผิดชนิด key
โดยตั้งใจ) แล้วพยายามเซ็น token:

```rust
match EncodingKey::from_rsa_pem(&public_pem) {
    Ok(bad_key) => match encode(&header, &claims, &bad_key) {
        Ok(_) => println!("(ไม่คาดคิด) เซ็น token ด้วย public key สำเร็จ"),
        Err(e) => println!("พยายามเซ็น token ด้วย public key (ผิดชนิด key) -> error จริง: {e}"),
    },
    Err(e) => println!("parse public key เป็น encoding key ล้มเหลวตั้งแต่ต้น -> error จริง: {e}"),
}
```

ผลลัพธ์จริง — parse PEM structurally ผ่าน (เพราะ public key ก็เป็น PEM ที่ parse ได้เหมือนกัน) แต่พอพยายาม
**เซ็น** จริง ๆ ระบบ cryptography ข้างใน RSA เจอทันทีว่า data structure ที่ได้ไม่มีข้อมูลที่จำเป็นสำหรับการ
เซ็น (ไม่มี private exponent ที่ private key เท่านั้นมี) — error จริงที่ได้:

```
พยายามเซ็น token ด้วย public key (ผิดชนิด key) -> error จริง: Signing failed: signature error: PKCS#8 ASN.1 error: ASN.1 INTEGER not canonically encoded as DER
```

ข้อความ error นี้อ่านตรงเป๊ะไม่ง่ายนัก (เป็น error ระดับ ASN.1 parsing ภายใน) แต่สิ่งที่ต้องเข้าใจคือ**หลักการ**
ที่พิสูจน์ได้: **ไม่มีทางใดที่จะเอา public key ไปเซ็น token ที่ตรวจสอบผ่านได้เลย** — คุณสมบัติทาง cryptography
นี้คือฐานทั้งหมดของความปลอดภัยของ asymmetric signing

#### ES256: ตัวเลือกอื่นในกลุ่ม asymmetric

`ES256` (ECDSA บน curve P-256) ทำงานตามแนวคิดเดียวกับ `RS256` ทุกประการ (private key เซ็น, public key ตรวจ)
เพียงแต่ใช้ elliptic curve cryptography แทน RSA — ข้อดีเชิงปฏิบัติของ ES256 คือ **key และ signature มีขนาด
เล็กกว่า RSA มาก** ที่ระดับความปลอดภัยเทียบเท่ากัน (P-256 key ~256 bit เทียบกับ RSA ที่ต้องใช้ 2048 บิตขึ้นไป
ถึงจะปลอดภัยพอ) ทำให้ token สั้นกว่า ประหยัด bandwidth ต่อ request มากขึ้นเมื่อระบบมี traffic สูง — API ของ
`jsonwebtoken` เหมือนกับ RS256 ทุกประการ เปลี่ยนแค่ `Algorithm::ES256` และใช้
`EncodingKey::from_ec_pem`/`DecodingKey::from_ec_pem` แทน `_rsa_pem` — เลือกใช้ตัวไหนขึ้นกับว่าระบบ/library
ฝั่งอื่นที่ต้องคุยด้วยรองรับอะไร (RSA เก่ากว่า รองรับกว้างกว่า, ECDSA ใหม่กว่า มีประสิทธิภาพดีกว่า)

#### ตารางสรุป: เลือก HS256 หรือ RS256/ES256

| | HS256 (symmetric) | RS256/ES256 (asymmetric) |
|---|---|---|
| Key | secret เดียว ใช้ทั้งเซ็น/ตรวจ | key pair — private เซ็น, public ตรวจ |
| ความซับซ้อนของการตั้งค่า | ต่ำมาก (byte string เดียว) | สูงกว่า (ต้องสร้าง/กระจาย key pair, จัดการ rotation) |
| แจก "ความสามารถตรวจสอบ" ให้ service อื่นได้ไหม | ไม่ได้ (แจก secret = แจกสิทธิ์เซ็นด้วย) | ได้ — แจกแค่ public key |
| เหมาะกับ | Monolith หรือกลุ่ม service เดียวที่แชร์ secret ปลอดภัยได้ | ระบบ microservices ที่มี auth service กลาง (Part 81) |
| ความเสี่ยงถ้า service ที่ถือ key หลุด | ทุกจุดที่แชร์ secret เดียวกันตกอยู่ในความเสี่ยง | เสี่ยงแค่ auth service ที่ถือ private key เท่านั้น |

### 74.5 ออกแบบ Claims: Standard Claims และ Custom Claims

**Claims** คือข้อมูลที่อยู่ใน payload ของ JWT — RFC 7519 กำหนด **standard claims** ที่มีความหมายตายตัวไว้ให้
(ไม่บังคับใช้ทุกตัว แต่ถ้าใช้ต้องมีความหมายตรงตามที่กำหนด เพื่อให้ library/ระบบอื่นที่อ่าน JWT เข้าใจตรงกัน):

| Claim | ความหมาย | ตัวอย่าง |
|---|---|---|
| `sub` (subject) | ใครคือเจ้าของ token นี้ — มักเป็น user id | `"user-42"` |
| `exp` (expiration) | เวลาที่ token หมดอายุ (Unix timestamp, วินาที) — **`jsonwebtoken` ตรวจสอบให้อัตโนมัติเสมอ** ถ้าไม่ปิดเอง | `1790472494` |
| `iat` (issued at) | เวลาที่ token ถูกออก — มีประโยชน์เชิง audit/debug | `1790468894` |
| `nbf` (not before) | token นี้ "ยังใช้ไม่ได้" ก่อนเวลานี้ — เหมาะกับ token ที่ออกล่วงหน้าแต่ให้เริ่มใช้ทีหลัง | `1790468894` |
| `aud` (audience) | ใครควรเป็นผู้รับ/ตรวจสอบ token นี้ — ป้องกัน token ที่ออกให้ service A ถูกเอาไปใช้กับ service B | `"orders-api"` |
| `iss` (issuer) | ใครออก token นี้ — มีประโยชน์เมื่อมีหลาย auth service หรือรับ token จาก third-party (เช่น OAuth2 provider ที่ Part 75 จะสอน) | `"https://auth.example.com"` |

นอกจาก standard claims แล้ว คุณใส่ **custom claims** อะไรก็ได้ที่แอปต้องการ (ตราบใดที่ไม่ชนชื่อกับ standard
claims) — บทนี้ใช้:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,      // subject -- user id (เป็น String ตามธรรมเนียม JWT แม้ id จริงเป็นเลข)
    username: String, // custom claim -- สะดวกให้ handler ใช้ตรง ๆ ไม่ต้อง query ซ้ำ
    role: String,     // custom claim -- ใช้ต่อใน Part 76 (RBAC)
    iat: usize,        // issued at
    exp: usize,        // expiration
    typ: String,        // custom claim ของเราเอง -- "access" หรือ "refresh" (หัวข้อ 74.10)
}
```

**อธิบายการออกแบบทีละจุด**:

- **`#[derive(Serialize, Deserialize)]`** — ตรงตามที่ Part 57 สอนไว้ทุกประการ: `jsonwebtoken::encode`
  ต้องการให้ type ที่ส่งเข้าไป implement `Serialize` (แปลงเป็น JSON ก่อน encode base64url) และ
  `jsonwebtoken::decode` ต้องการ `Deserialize` (parse JSON ที่ decode ออกมากลับเป็น struct)
- **`sub: String`** ไม่ใช่ `sub: u32`** — แม้ `user_id` จริงในระบบเป็นเลข แต่ธรรมเนียม JWT (และ library ส่วน
  ใหญ่ในหลายภาษา) กำหนดให้ `sub` เป็น string เสมอ เพื่อความเข้ากันได้ข้ามระบบที่อาจใช้ id เป็นชนิดต่างกัน
  (UUID, ObjectId, integer) — แปลงเป็น string ตอนสร้าง claims แล้วแปลงกลับตอนใช้งาน
- **`username`, `role`, `typ`** — custom claims ที่ไม่มีใน RFC 7519 แต่แอปนี้ต้องการ: `username` สะดวกให้
  handler แสดงผลได้ตรง ๆ โดยไม่ต้อง query ซ้ำ, `role` ใช้ต่อใน Part 76 (ตรวจว่า user มีสิทธิ์เรียก endpoint
  นี้ไหม), `typ` เป็นวิธีของเราเองในการแยกว่า token นี้เป็น access token หรือ refresh token (หัวข้อ 74.10)
- **ชื่อ claim สั้น (`sub` ไม่ใช่ `subject`, `exp` ไม่ใช่ `expiration`)** — เพราะ JWT ถูกส่งไปกับ**ทุก** HTTP
  request ที่ต้อง auth (ปกติผ่าน header `Authorization`) การใช้ชื่อสั้นช่วยลดขนาด token ลงเล็กน้อย (สะสม
  เยอะขึ้นถ้า request จำนวนมาก) — นี่คือเหตุผลที่ RFC 7519 เลือกชื่อ claim มาตรฐานสั้น ๆ ตั้งแต่ต้น custom
  claims ของคุณเองก็ควรตั้งชื่อสั้นกระชับตามแนวทางนี้ถ้าเป็นไปได้ (แม้ไม่บังคับ)

### 74.6 เก็บผู้ใช้และ Hash Password ด้วย Argon2 (Signup)

ก่อนจะออก JWT ได้ ต้องมีระบบผู้ใช้ก่อน — และก่อนเก็บ user ลง storage ต้องเข้าใจหลักการที่**สำคัญที่สุดข้อหนึ่ง
ของความปลอดภัยเว็บทั้งหมด**:

> **ห้ามเก็บ password เป็น plaintext ลง storage เด็ดขาด ไม่มีข้อยกเว้น แม้แต่ "แค่ตัวอย่างสอน" ก็ไม่ควรเขียน
> โค้ดที่เก็บ plaintext เพราะโค้ดตัวอย่างมักถูก copy ไปใช้จริงโดยไม่มีใครแก้**

ถ้าฐานข้อมูลรั่ว (เกิดขึ้นบ่อยกว่าที่คิด — data breach เป็นข่าวเกือบทุกเดือน) การเก็บ plaintext แปลว่า
password ของผู้ใช้**ทุกคน**หลุดออกไปทันที และเพราะคนส่วนใหญ่ใช้ password ซ้ำกันหลายเว็บ ผลกระทบจะลามไปยัง
บัญชีอื่นของผู้ใช้คนเดียวกันในเว็บอื่นด้วย (credential stuffing attack)

#### ทำไมเลือก `argon2` ไม่ใช่ `bcrypt`

ทั้ง `argon2` และ `bcrypt` เป็น **password hashing function** ที่ออกแบบมาให้ "ช้าโดยตั้งใจ" (ต่างจาก
`sha256`/`md5` ที่เร็วมาก ซึ่งเป็นข้อเสียเมื่อใช้กับ password เพราะ attacker brute-force ได้เร็วตามไปด้วย) —
แต่ทั้งสองต่างกันที่**คุณสมบัติการต้านทาน**:

- **`bcrypt`** (ปี 1999) — ต้านทาน brute-force ด้วยการทำให้**ช้า** (ปรับ cost factor ได้) แต่ใช้**หน่วยความจำ
  คงที่และน้อย** — ทำให้ GPU/ASIC ที่มี core จำนวนมากแต่ memory ต่อ core น้อย สามารถรัน bcrypt แบบ**parallel
  จำนวนมากพร้อมกัน**ได้อย่างมีประสิทธิภาพ (brute-force เร็วกว่าที่ควรในทางทฤษฎี) นอกจากนี้ยังมีข้อจำกัดที่รู้
  จักกันดีคือ**ตัด password ที่ยาวกว่า 72 byte ทิ้งแบบเงียบ ๆ** (ส่วนที่เกินไม่มีผลต่อ hash เลย)
- **`argon2`** (ผู้ชนะ Password Hashing Competition ปี 2015 ซึ่งเป็นการแข่งขันสาธารณะที่คัดเลือกฟังก์ชัน
  hashing password ที่ดีที่สุดยุคใหม่) — เป็น **memory-hard function**: ต้านทานด้วยการบีบให้ต้องใช้
  **หน่วยความจำจำนวนมาก** ต่อการคำนวณหนึ่งครั้ง (ปรับพารามิเตอร์ได้ทั้ง memory cost, time cost, parallelism)
  ทำให้ GPU/ASIC (ที่ core เยอะแต่ memory ต่อ core จำกัด) **parallel ได้ยากกว่า bcrypt มาก** — และมีตัวแปร
  **Argon2id** (ค่า default ของ crate `argon2`) ที่ผสมจุดเด่นของ Argon2i (ต้านทาน side-channel attack) กับ
  Argon2d (ต้านทาน GPU cracking) เข้าด้วยกัน — **OWASP Password Storage Cheat Sheet แนะนำ Argon2id เป็นตัวเลือก
  แรกสุด** สำหรับระบบใหม่ในปัจจุบัน ด้วยเหตุผลเหล่านี้บทนี้เลือกสอน `argon2`

#### Implement จริง

```rust
use argon2::{
    password_hash::{phc::PasswordHash, PasswordHasher, PasswordVerifier},
    Argon2,
};

fn main() {
    let password = b"correct-horse-battery-staple";

    // Argon2 พร้อม default params (Argon2id, v19) -- generate salt สุ่มให้เองข้างในทุกครั้งที่
    // hash_password ถูกเรียก (ต้องเปิด feature "getrandom" ซึ่งเป็น default feature ของ argon2 อยู่แล้ว)
    let argon2 = Argon2::default();

    let hash = argon2.hash_password(password).unwrap().to_string();
    println!("argon2 hash ที่ได้ (เก็บอันนี้ลง DB เท่านั้น):\n{hash}");

    // verify: ตอน login เอา hash ที่เก็บไว้มา parse แล้วเทียบกับ password ที่ผู้ใช้กรอกมา
    let parsed_hash = PasswordHash::new(&hash).unwrap();
    let ok = argon2.verify_password(password, &parsed_hash).is_ok();
    println!("verify ด้วย password ที่ถูก: {ok}");
}
```

รันจริงได้ hash จริง:

```
argon2 hash ที่ได้ (เก็บอันนี้ลง DB เท่านั้น):
$argon2id$v=19$m=19456,t=2,p=1$Ad/OK19sUcicTfwxRluAuw$iDzl/Eh9+Chku7ijymj2m/yDlOyBYt9pyOS5y5tg/Vg

verify ด้วย password ที่ถูก: true
```

**อธิบายรูปแบบผลลัพธ์ (PHC string format — มาตรฐานเปิดสำหรับเก็บ password hash)**:

```
$argon2id$v=19$m=19456,t=2,p=1$Ad/OK19sUcicTfwxRluAuw$iDzl/Eh9+Chku7ijymj2m/yDlOyBYt9pyOS5y5tg/Vg
  ↑algorithm  ↑version  ↑params (memory=19456 KiB, time=2 iterations, parallelism=1 lane)   ↑salt (base64)   ↑hash output (base64)
```

**จุดสำคัญที่ต้องสังเกต: salt ถูกฝังอยู่ใน string ผลลัพธ์นี้เอง** — ไม่ต้องมี column แยกสำหรับเก็บ salt ใน
ตาราง `users` เลย เก็บ string เดียวนี้ลง column `password_hash` ก็ครบถ้วนพอสำหรับ verify ในอนาคต เพราะตอน
verify, `PasswordHash::new(&hash)` จะ parse ทั้ง algorithm, params, และ salt ออกมาจาก string เดียวกันนี้ แล้ว
ใช้ค่าเหล่านั้น (ไม่ใช่ค่าที่ตั้งไว้ใน `Argon2::default()` ตอน verify) คำนวณ hash ใหม่จาก password ที่ผู้ใช้
กรอกมา แล้วเทียบว่าตรงกันไหม

**พิสูจน์ว่า salt สุ่มใหม่ทุกครั้ง** — hash password เดียวกันสองครั้ง:

```
hash ครั้งที่ 2 (password เดิม, salt สุ่มใหม่):
$argon2id$v=19$m=19456,t=2,p=1$QzLIpDK0wCBAlO/STC7A2w$siANLOYhwo8+oncE5iWKNe6afnMB9qKwC9j08LRLNoI

สองอัน hash เดียวกันไหม: false
```

**นี่คือคุณสมบัติที่ตั้งใจ ไม่ใช่บั๊ก**: salt สุ่มใหม่ทุกครั้งที่ hash ทำให้ผู้ใช้สองคนที่ใช้ password
เดียวกันเป๊ะ (`"123456"` เป็นตัวอย่างคลาสสิก) ได้ `password_hash` ที่**ต่างกันสิ้นเชิง**ใน storage — ป้องกัน
attacker ที่รั่วฐานข้อมูลจากการสร้าง **rainbow table** (ตาราง lookup ล่วงหน้าของ hash ที่พบบ่อย) มาเทียบหา
password ทีเดียวสำหรับทั้งฐานข้อมูล — ต้อง brute-force ทีละ hash แยกกันเสมอ แม้สอง hash นั้นจะมาจาก password
เดียวกันก็ตาม

พิสูจน์ error จริงตอน verify ด้วย password ผิด:

```
verify_password คืน error จริงตอน password ผิด: PasswordInvalid
```

#### ใช้จริงใน signup handler

```rust
use argon2::{password_hash::{PasswordHasher}, Argon2};
use axum::{extract::State, http::StatusCode, Json};
use serde::Deserialize;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Debug, Clone)]
struct UserRecord {
    id: u32,
    username: String,
    password_hash: String, // *** ไม่ใช่ password ตรง ๆ เด็ดขาด -- เก็บแค่ hash นี้เท่านั้น ***
    role: String,
}

// ใช้ HashMap ในหน่วยความจำแทนตาราง `users` จริงของ Part 70 -- ดูคำอธิบายเหตุผลในหมายเหตุต้นบท
#[derive(Default)]
struct UserStore {
    by_username: Mutex<HashMap<String, UserRecord>>,
    next_id: Mutex<u32>,
}

#[derive(Clone)]
struct AppState {
    users: Arc<UserStore>,
    jwt_secret: Arc<String>,
}

#[derive(Debug, Deserialize)]
struct SignupRequest {
    username: String,
    password: String,
}

async fn signup(
    State(state): State<AppState>,
    Json(req): Json<SignupRequest>,
) -> Result<StatusCode, (StatusCode, Json<serde_json::Value>)> {
    let mut users = state.users.by_username.lock().unwrap();
    if users.contains_key(&req.username) {
        return Err((
            StatusCode::CONFLICT,
            Json(serde_json::json!({"error": {"code": "USERNAME_TAKEN", "message": "username นี้ถูกใช้ไปแล้ว"}})),
        ));
    }

    // *** ห้ามเก็บ/เทียบ password เป็น plaintext เด็ดขาด -- hash ด้วย argon2 ก่อนเก็บเสมอ ***
    let argon2 = Argon2::default();
    let password_hash = argon2
        .hash_password(req.password.as_bytes())
        .map_err(|_| {
            (
                StatusCode::INTERNAL_SERVER_ERROR,
                Json(serde_json::json!({"error": {"code": "HASH_ERROR", "message": "hash password ไม่สำเร็จ"}})),
            )
        })?
        .to_string();

    let mut next_id = state.users.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;

    users.insert(
        req.username.clone(),
        UserRecord { id, username: req.username, password_hash, role: "member".to_string() },
    );

    Ok(StatusCode::CREATED)
}
```

**สังเกตว่า field `password: String` จาก `SignupRequest` (ที่รับมาจาก client) ไม่ถูกเก็บลง `UserRecord`
เลยแม้แต่บรรทัดเดียว** — มันถูกใช้แค่ครั้งเดียวตอนเรียก `argon2.hash_password(...)` แล้วก็หายไปจาก scope
(ตัวแปร `req` ถูก drop ตอนจบฟังก์ชัน ไม่มีการ clone เก็บไว้ที่ไหนเลย) สิ่งเดียวที่รอดไปถึง `UserRecord` คือ
`password_hash` ที่ผ่าน argon2 มาแล้ว

ทดสอบด้วย `curl` จริง:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3074/signup -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"correct-horse-battery-staple"}'
HTTP/1.1 201 Created
content-length: 0
date: Sun, 27 Sep 2026 00:36:53 GMT

$ curl -sS -i -X POST http://127.0.0.1:3074/signup -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"anything"}'
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 102

{"error":{"code":"USERNAME_TAKEN","message":"username นี้ถูกใช้ไปแล้ว"}}
```

### 74.7 Login Endpoint: ตรวจสอบรหัสผ่านและออก Token

Login ทำสองงาน: (1) ตรวจสอบว่า username/password ที่ส่งมาถูกต้องไหม (2) ถ้าถูกต้อง สร้างและออก JWT กลับไป

```rust
use argon2::{
    password_hash::{phc::PasswordHash, PasswordHasher, PasswordVerifier},
    Argon2,
};
use axum::{extract::State, http::StatusCode, Json};
use chrono::Utc;
use jsonwebtoken::{encode, Algorithm, EncodingKey, Header};
use serde::{Deserialize, Serialize};
# use std::sync::Arc;
# #[derive(Clone)] struct AppState { users: Arc<UserStore>, jwt_secret: Arc<String> }
# #[derive(Default)] struct UserStore { by_username: std::sync::Mutex<std::collections::HashMap<String, UserRecord>>, next_id: std::sync::Mutex<u32> }
# #[derive(Debug, Clone)] struct UserRecord { id: u32, username: String, password_hash: String, role: String }

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,
    username: String,
    role: String,
    iat: usize,
    exp: usize,
    typ: String,
}

#[derive(Debug, Deserialize)]
struct LoginRequest {
    username: String,
    password: String,
}

#[derive(Debug, Serialize)]
struct TokenResponse {
    access_token: String,
    refresh_token: String,
    token_type: &'static str,
    expires_in: i64,
}

fn issue_token_pair(state: &AppState, user: &UserRecord) -> TokenResponse {
    let now = Utc::now().timestamp() as usize;
    let access_ttl = 900; // 15 นาที -- access token อายุสั้นตั้งใจ (ดูหัวข้อ 74.10)
    let refresh_ttl = 60 * 60 * 24 * 7; // 7 วัน

    let header = Header::new(Algorithm::HS256);
    let key = EncodingKey::from_secret(state.jwt_secret.as_bytes());

    let access_claims = Claims {
        sub: user.id.to_string(), username: user.username.clone(), role: user.role.clone(),
        iat: now, exp: now + access_ttl, typ: "access".to_string(),
    };
    let refresh_claims = Claims {
        sub: user.id.to_string(), username: user.username.clone(), role: user.role.clone(),
        iat: now, exp: now + refresh_ttl, typ: "refresh".to_string(),
    };

    TokenResponse {
        access_token: encode(&header, &access_claims, &key).unwrap(),
        refresh_token: encode(&header, &refresh_claims, &key).unwrap(),
        token_type: "Bearer",
        expires_in: access_ttl as i64,
    }
}

async fn login(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>,
) -> Result<Json<TokenResponse>, (StatusCode, Json<serde_json::Value>)> {
    let users = state.users.by_username.lock().unwrap();
    let user = users.get(&req.username).ok_or_else(|| {
        (
            StatusCode::UNAUTHORIZED,
            Json(serde_json::json!({"error": {"code": "INVALID_CREDENTIALS", "message": "username หรือ password ไม่ถูกต้อง"}})),
        )
    })?;

    let parsed_hash = PasswordHash::new(&user.password_hash).map_err(|_| {
        (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(serde_json::json!({"error": {"code": "HASH_ERROR", "message": "hash เสียหาย"}})),
        )
    })?;

    // *** เทียบ password ด้วย argon2 verify เท่านั้น -- ไม่มีการเทียบ String กับ String ตรง ๆ ที่ไหนเลย ***
    if Argon2::default().verify_password(req.password.as_bytes(), &parsed_hash).is_err() {
        // ตั้งใจตอบข้อความเดียวกันกับ "ไม่พบ username" ด้านบนทุกตัวอักษร -- ไม่บอกว่า
        // "username ถูกแต่ password ผิด" เพราะข้อมูลนั้นช่วย attacker เดา username ที่มีอยู่จริงได้
        return Err((
            StatusCode::UNAUTHORIZED,
            Json(serde_json::json!({"error": {"code": "INVALID_CREDENTIALS", "message": "username หรือ password ไม่ถูกต้อง"}})),
        ));
    }

    Ok(Json(issue_token_pair(&state, user)))
}
```

**อธิบายจุดที่ตั้งใจออกแบบเรื่องความปลอดภัย**: ทั้งกรณี "ไม่พบ username เลย" และกรณี "พบ username แต่
password ผิด" **ตอบข้อความเดียวกันทุกตัวอักษร** (`INVALID_CREDENTIALS`) — นี่ไม่ใช่ความบังเอิญ ถ้าระบบตอบ
ต่างกัน (เช่น "ไม่พบ user" กับ "password ผิด") attacker ที่ลองยิง username จำนวนมากจะสามารถ**แยกแยะได้ว่า
username ไหนมีอยู่จริงในระบบ**จากข้อความ error ที่ต่างกัน (เรียกว่า **username enumeration**) แม้จะยัง login
ไม่สำเร็จ ข้อมูลนี้ก็มีค่าสำหรับ attacker ในการวางแผนโจมตีขั้นต่อไป (เช่น brute-force password เฉพาะ username
ที่ยืนยันแล้วว่ามีจริง) — การตอบข้อความเดียวกันเสมอปิดช่องทางนี้ตั้งแต่ต้น

ทดสอบจริง:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3074/login -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"wrong-password"}'
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 114

{"error":{"code":"INVALID_CREDENTIALS","message":"username หรือ password ไม่ถูกต้อง"}}

$ curl -sS -X POST http://127.0.0.1:3074/login -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"correct-horse-battery-staple"}'
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIwIiwidXNlcm5hbWUiOiJuYW4iLCJyb2xlIjoibWVtYmVyIiwiaWF0IjoxNzkwNDY5NDE1LCJleHAiOjE3OTA0NzAzMTUsInR5cCI6ImFjY2VzcyJ9.-cn3jy_CRnSS7bMATfVW3um5YdRfqHk_HOEXnY19Yyo","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIwIiwidXNlcm5hbWUiOiJuYW4iLCJyb2xlIjoibWVtYmVyIiwiaWF0IjoxNzkwNDY5NDE1LCJleHAiOjE3OTEwNzQyMTUsInR5cCI6InJlZnJlc2gifQ.B-xmK30klQjijXIAjbcGSUhjUUjXC3M9K7CfbrE50f0","token_type":"Bearer","expires_in":900}
```

ลอง decode `access_token` นี้ด้วยเทคนิคจากหัวข้อ 74.2 จะเห็น payload จริง:
`{"sub":"0","username":"nan","role":"member","iat":1790469415,"exp":1790470315,"typ":"access"}` — ตรงตามที่
ออกแบบไว้ทุกประการ

### 74.8 Custom Extractor: `CurrentUser` ตรวจสอบ Token อัตโนมัติทุก Handler

ตอนนี้เรามี token แล้ว งานต่อไปคือทำให้ handler ที่ต้องการ "ผู้ใช้ที่ login แล้ว" เท่านั้นถึงจะทำงานได้ — Part
64 หัวข้อ 64.6 สอนแพทเทิร์น custom extractor ผ่าน `FromRequestParts` ไว้แล้วด้วยตัวอย่าง API key แบบง่าย ๆ
บทนี้ใช้แพทเทิร์นเดียวกันเป๊ะ เพียงแค่เปลี่ยน logic การตรวจสอบจาก "เทียบ string ตรง ๆ" เป็น "ตรวจสอบ JWT เต็ม
รูปแบบ"

```rust
use axum::{
    extract::FromRequestParts,
    http::request::Parts,
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};
# #[derive(Clone)] struct AppState { jwt_secret: std::sync::Arc<String> }
# #[derive(Debug, serde::Serialize, serde::Deserialize, Clone)]
# struct Claims { sub: String, username: String, role: String, iat: usize, exp: usize, typ: String }

struct CurrentUser {
    claims: Claims,
}

// Rejection ของเราเอง -- enum ที่แยกแยะสาเหตุความล้มเหลวได้หลายแบบ ตามแพทเทิร์นเดียวกับ
// ApiKeyRejection ของ Part 64 หัวข้อ 64.6 และ AppError ของ Part 66
#[derive(Debug)]
enum JwtRejection {
    MissingHeader,
    MalformedHeader,
    InvalidToken(jsonwebtoken::errors::ErrorKind),
    WrongTokenType,
}

impl IntoResponse for JwtRejection {
    fn into_response(self) -> Response {
        let (status, code, message) = match &self {
            JwtRejection::MissingHeader => (
                StatusCode::UNAUTHORIZED, "MISSING_TOKEN",
                "ไม่พบ header Authorization: Bearer <token>".to_string(),
            ),
            JwtRejection::MalformedHeader => (
                StatusCode::UNAUTHORIZED, "MALFORMED_HEADER",
                "รูปแบบ Authorization header ไม่ถูกต้อง (ต้องเป็น 'Bearer <token>')".to_string(),
            ),
            JwtRejection::WrongTokenType => (
                StatusCode::UNAUTHORIZED, "WRONG_TOKEN_TYPE",
                "ใช้ refresh token เรียก endpoint ที่ต้องการ access token (หรือกลับกัน)".to_string(),
            ),
            JwtRejection::InvalidToken(kind) => {
                let msg = match kind {
                    jsonwebtoken::errors::ErrorKind::ExpiredSignature =>
                        "token หมดอายุแล้ว กรุณาเข้าสู่ระบบใหม่หรือใช้ refresh token".to_string(),
                    jsonwebtoken::errors::ErrorKind::InvalidSignature =>
                        "ลายเซ็นของ token ไม่ถูกต้อง (token อาจถูกแก้ไขหรือเซ็นด้วย key ผิด)".to_string(),
                    other => format!("token ไม่ผ่านการตรวจสอบ: {other:?}"),
                };
                (StatusCode::UNAUTHORIZED, "INVALID_TOKEN", msg)
            }
        };
        (status, Json(serde_json::json!({ "error": { "code": code, "message": message } }))).into_response()
    }
}

impl FromRequestParts<AppState> for CurrentUser {
    type Rejection = JwtRejection;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &AppState,
    ) -> Result<Self, Self::Rejection> {
        let header_value = parts
            .headers
            .get("Authorization")
            .ok_or(JwtRejection::MissingHeader)?
            .to_str()
            .map_err(|_| JwtRejection::MalformedHeader)?;

        let token = header_value
            .strip_prefix("Bearer ")
            .ok_or(JwtRejection::MalformedHeader)?;

        // *** จุดสำคัญที่สุดของบทนี้ในเชิงความปลอดภัย (ดูรายละเอียดเต็มในหัวข้อกับดัก) ***
        // Validation::new(Algorithm::HS256) ล็อกอัลกอริทึมที่ "ยอมรับ" ไว้ตายตัวฝั่งเซิร์ฟเวอร์
        // -- ไม่เคยอ่าน "alg" จาก header ของ token แล้วเชื่อตามนั้น
        let mut validation = Validation::new(Algorithm::HS256);
        validation.set_required_spec_claims(&["exp", "sub"]);

        let decoding_key = DecodingKey::from_secret(state.jwt_secret.as_bytes());
        let token_data = decode::<Claims>(token, &decoding_key, &validation)
            .map_err(|e| JwtRejection::InvalidToken(e.into_kind()))?;

        if token_data.claims.typ != "access" {
            return Err(JwtRejection::WrongTokenType);
        }

        Ok(CurrentUser { claims: token_data.claims })
    }
}
```

**อธิบายทีละขั้น**:

1. **`parts.headers.get("Authorization")`** — เหมือนกับที่ `ApiKey` extractor ใน Part 64 อ่าน
   `X-Api-Key` เป๊ะ — ต่างกันแค่ชื่อ header และรูปแบบค่า (`Bearer <token>` เป็นรูปแบบมาตรฐานของ RFC 6750
   ที่ HTTP client/library ส่วนใหญ่รู้จัก ต่างจาก custom header ธรรมดา)
2. **`strip_prefix("Bearer ")`** — ตัดคำว่า `"Bearer "` (มีวรรคท้าย) ออกจากค่า header เหลือแค่ token ล้วน ๆ
   ถ้า header ไม่ได้ขึ้นต้นด้วยคำนี้ (เช่นส่ง token มาตรง ๆ ไม่มี `"Bearer "` นำหน้า) `strip_prefix` คืน
   `None` แปลงเป็น `JwtRejection::MalformedHeader` ทันที
3. **`Validation::new(Algorithm::HS256)`** — สร้าง validation policy ที่ **ล็อกว่ายอมรับแค่ HS256 เท่านั้น**
   (รายละเอียดว่าทำไมขั้นนี้สำคัญมากอยู่ในหัวข้อกับดักท้ายบท)
4. **`decode::<Claims>(token, &decoding_key, &validation)`** — ตรวจสอบทั้ง**ลายเซ็น** (ว่า token นี้เซ็นด้วย
   secret ตัวเดียวกันจริงไหม) และ **`exp`** (ว่ายังไม่หมดอายุ — `jsonwebtoken` ตรวจให้อัตโนมัติเสมอถ้าไม่ปิด
   เอง) ในการเรียกครั้งเดียว — ถ้าล้มเหลวไม่ว่าจะด้วยสาเหตุไหน คืน `jsonwebtoken::errors::Error` ที่มี
   `.kind()` บอกสาเหตุที่เจาะจง (`ExpiredSignature`, `InvalidSignature`, `Base64(...)`, ฯลฯ)
5. **`if token_data.claims.typ != "access"`** — ตรวจเพิ่มว่า token นี้เป็น **access token** จริง ไม่ใช่
   refresh token ที่เอามาใช้ผิดที่ (refresh token มี `exp` ยาวกว่าและ `sub`/ลายเซ็นถูกต้องเหมือนกัน ถ้าไม่
   เช็ค `typ` เพิ่ม ผู้ใช้ที่มีแค่ refresh token จะสามารถเรียก endpoint ที่ต้องการ access token ได้ ซึ่งไม่ควร
   เกิดขึ้น — refresh token ควรใช้ได้แค่กับ `/refresh` เท่านั้น)

**handler ที่ใช้ extractor นี้ไม่ต้องรู้เรื่อง JWT เลยแม้แต่นิดเดียว**:

```rust
use axum::Json;
use serde::Serialize;
# struct CurrentUser { claims: Claims }
# #[derive(serde::Serialize, serde::Deserialize, Clone)] struct Claims { sub: String, username: String, role: String, iat: usize, exp: usize, typ: String }

#[derive(Debug, Serialize)]
struct MeResponse {
    user_id: String,
    username: String,
    role: String,
}

async fn me(current_user: CurrentUser) -> Json<MeResponse> {
    Json(MeResponse {
        user_id: current_user.claims.sub,
        username: current_user.claims.username,
        role: current_user.claims.role,
    })
}
```

นี่คือพลังของ extractor pattern ที่ Part 64 ปูพื้นไว้: `me` ไม่มี `if`/`match` เกี่ยวกับ token เลยแม้แต่บรรทัด
เดียว — ถ้าโค้ดใน body ของ `me` ได้รันถึง แปลว่า `CurrentUser` ถูกสร้างสำเร็จแล้วเรียบร้อย (ผ่านการตรวจสอบ
ลายเซ็น, `exp`, และ `typ` มาแล้วทั้งหมด) — คล้ายกับ typestate pattern ที่ Part 53 สอนไว้: ใช้ type system เป็น
ตัวการันตี invariant แทนการเช็คด้วยมือซ้ำในทุก handler

### 74.9 ทดสอบด้วย `curl` จริง: ครบ 5 สถานการณ์

รันเซิร์ฟเวอร์จริงแล้วยิง `curl` ทดสอบทุกเส้นทางที่ extractor ต้องรับมือ — ทุก response ด้านล่างนี้คือ
**ผลลัพธ์จริงที่รันแล้วคัดลอกมา**

**1) Token ถูกต้อง → `200 OK`**:

```bash
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $ACCESS"
HTTP/1.1 200 OK
content-type: application/json
content-length: 48

{"user_id":"0","username":"nan","role":"member"}
```

**2) ไม่ส่ง header `Authorization` มาเลย → `401`**:

```bash
$ curl -sS -i http://127.0.0.1:3074/me
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 99

{"error":{"code":"MISSING_TOKEN","message":"ไม่พบ header Authorization: Bearer <token>"}}
```

**3) Token ผิดรูปแบบสิ้นเชิง (ไม่ใช่ JWT จริง — พิมพ์ข้อความมือ) → `401` พร้อม error variant จริงจาก
`jsonwebtoken`**:

```bash
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer not.a.realtoken"
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 139

{"error":{"code":"INVALID_TOKEN","message":"token ไม่ผ่านการตรวจสอบ: Base64(InvalidLastSymbol(2, 116))"}}
```

สังเกตว่า error variant ที่ได้คือ `jsonwebtoken::errors::ErrorKind::Base64(...)` — เพราะ `"not.a.realtoken"`
แยกด้วย `.` ได้สามส่วน (`not`, `a`, `realtoken`) แต่ **สองส่วนแรกไม่ใช่ base64url ที่ decode ได้เป็น JSON
ที่ valid เลย** — `jsonwebtoken` เจอปัญหาตั้งแต่ขั้น decode base64 ก่อนจะไปถึงขั้นตรวจลายเซ็นด้วยซ้ำ

**4) Token ที่หมดอายุแล้วจริง → `401` พร้อม `ErrorKind::ExpiredSignature` จริง**:

สร้าง token ที่ตั้งใจให้ `exp` เป็นเมื่อ 1 ชั่วโมงที่แล้ว (สร้างด้วย secret เดียวกันกับเซิร์ฟเวอร์ เพื่อให้
ผ่านการตรวจลายเซ็น แล้วไปติดที่การตรวจ `exp` แทน — พิสูจน์ว่า `jsonwebtoken` ตรวจ`exp`แยกจากลายเซ็นจริง):

```bash
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $EXPIRED_TOKEN"
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 182

{"error":{"code":"INVALID_TOKEN","message":"token หมดอายุแล้ว กรุณาเข้าสู่ระบบใหม่หรือใช้ refresh token"}}
```

(ข้อความนี้ถูกแปลงมาจาก `jsonwebtoken::errors::ErrorKind::ExpiredSignature` ที่ `match` ไว้ใน
`JwtRejection::InvalidToken` — พิสูจน์ error variant ดิบด้วยการเรียก `decode()` ตรง ๆ นอก Axum context ในหัวข้อ
กับดักท้ายบท จะเห็น `Error(ExpiredSignature)` ตรงตัวจาก `{:?}`)

**ข้อสังเกตเชิงลึกที่ผู้เขียนพบระหว่างทดสอบจริง**: ครั้งแรกที่ลองสร้าง expired token ด้วย `exp` ที่หมดอายุไป
แค่ **60 วินาที** เซิร์ฟเวอร์กลับตอบ `200 OK` เฉย ๆ ทั้งที่ `exp` ผ่านไปแล้ว! สาเหตุคือ `jsonwebtoken::Validation`
มีค่า **`leeway` (ค่าเผื่อ clock skew ระหว่างเครื่อง client/server) เป็นค่า default 60 วินาที** — หมายความว่า
token ที่หมดอายุไป**ไม่เกิน 60 วินาที**จะยังผ่านการตรวจสอบได้ (เพราะ `jsonwebtoken` สมมติว่าอาจเป็นความต่างของ
นาฬิการะบบระหว่างเครื่องที่ออก token กับเครื่องที่ตรวจสอบ ไม่ใช่ token ที่หมดอายุจริง) — ต้องทดสอบด้วย `exp`
ที่หมดอายุไปนานพอ (ในตัวอย่างนี้ใช้ 1 ชั่วโมง) จึงจะเห็น `ExpiredSignature` ชัดเจนแบบไม่มีข้อสงสัย **นี่คือ
รายละเอียดที่มักไม่มีใครพูดถึง แต่สำคัญมากถ้าคุณกำลัง debug ว่า "ทำไม token ที่ควรหมดอายุแล้วยังผ่านอยู่"**

**5) Token ที่ลายเซ็นถูกแก้ไข (tampered) → `401` พร้อม `ErrorKind::InvalidSignature` จริง**:

แก้ตัวอักษรสุดท้ายของ signature (ส่วนที่ 3 ของ token) ให้เปลี่ยนไปหนึ่งตัว (ยังเป็น base64url ที่ valid แต่
ค่าคำนวณต่างไปจากเดิม):

```bash
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $TAMPERED_TOKEN"
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 206

{"error":{"code":"INVALID_TOKEN","message":"ลายเซ็นของ token ไม่ถูกต้อง (token อาจถูกแก้ไขหรือเซ็นด้วย key ผิด)"}}
```

**สังเกตความต่างระหว่างกรณี 3 กับกรณี 5**: กรณี 3 (token ผิดรูปแบบสิ้นเชิง) ล้มเหลวตั้งแต่ขั้น **decode
base64** (`ErrorKind::Base64`) — ยังไม่ได้ไปถึงขั้นตรวจลายเซ็นเลยด้วยซ้ำ ส่วนกรณี 5 (แก้ signature) **decode
base64 สำเร็จ** (เพราะ base64url ยังถูกต้องตามรูปแบบ) แต่**ลายเซ็นที่คำนวณใหม่ไม่ตรงกับที่แนบมา**
(`ErrorKind::InvalidSignature`) — ทั้งสองกรณีจบลงที่ HTTP `401` เหมือนกัน แต่สาเหตุภายในต่างกันโดยสิ้นเชิง ซึ่ง
`match` บน `ErrorKind` ในโค้ดของเราแยกแยะได้ชัดเจนสำหรับการ log/debug (แม้ response ที่ client เห็นจะคล้ายกัน
โดยตั้งใจ — ไม่บอก attacker ว่า "เกือบถูกแล้ว แค่ signature ผิด" มากเกินไป)

### 74.10 ปกป้อง Route: Extractor ต่อ Handler เทียบกับ Middleware ทั้งกลุ่ม

มีสองวิธีหลักในการ "บังคับ" ว่า route หนึ่งต้อง login ก่อนถึงจะเข้าได้ — บทนี้ใช้วิธีแรก (extractor) เป็นหลัก
มาตลอด แต่ Part 65 สอน middleware ไว้แล้ว ควรรู้ทั้งสองแบบและเลือกให้ถูกจุด

#### วิธีที่ 1: Custom Extractor ต่อ Handler (แบบที่บทนี้ใช้)

**ข้อดี**: (1) **มองเห็นได้ตรงจาก signature ของ handler** — ใครอ่านโค้ด `async fn me(current_user:
CurrentUser)` ก็รู้ทันทีว่า endpoint นี้ต้อง auth โดยไม่ต้องไปเปิดดู `.layer()` ที่อาจอยู่ไกลจากตัว handler
(2) **ได้ค่า `Claims`/`CurrentUser` มาใช้ตรง ๆ ใน handler** — ไม่ต้องดึงออกจาก `Extension`/request extensions
อีกชั้น (3) **เลือกได้ต่อ handler** — บาง route ในกลุ่มเดียวกัน (เช่นอยู่ใต้ path prefix เดียวกัน) อยากให้
auth บางตัวไม่ต้อง auth บางตัว ก็ทำได้ตรง ๆ แค่ใส่/ไม่ใส่ extractor **ข้อเสีย**: ถ้ามี handler จำนวนมากที่
ต้อง auth เหมือนกันหมด ต้องใส่ extractor เดิมซ้ำในทุก handler (แม้ implement แค่ครั้งเดียว แต่ "ประกาศใช้"
ซ้ำทุกที่)

#### วิธีที่ 2: Middleware ครอบทั้งกลุ่ม Route ที่ `.nest()` ไว้ (ตามแนวทาง Part 65)

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};
use serde::{Deserialize, Serialize};

#[derive(Clone)]
struct AppState {
    jwt_secret: std::sync::Arc<String>,
}

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,
    exp: usize,
}

// middleware เดียว ครอบทั้งกลุ่ม route ที่ nest ไว้ -- ไม่ต้องใส่ extractor ซ้ำในแต่ละ handler เลย
async fn require_auth(
    State(state): State<AppState>,
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let header = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let validation = Validation::new(Algorithm::HS256);
    let key = DecodingKey::from_secret(state.jwt_secret.as_bytes());
    decode::<Claims>(header, &key, &validation).map_err(|_| StatusCode::UNAUTHORIZED)?;

    Ok(next.run(req).await)
}

async fn admin_dashboard() -> &'static str { "admin dashboard" }
async fn admin_reports() -> &'static str { "admin reports" }
async fn public_health() -> &'static str { "ok" }

fn build_router(state: AppState) -> Router {
    // กลุ่ม route ที่ต้อง auth ทั้งกลุ่ม -- .route_layer() ครอบทุก route ที่ nest เข้ามาใต้ prefix นี้
    let admin_routes = Router::new()
        .route("/dashboard", get(admin_dashboard))
        .route("/reports", get(admin_reports))
        .route_layer(middleware::from_fn_with_state(state.clone(), require_auth));

    Router::new()
        .route("/health", get(public_health)) // ไม่ต้อง auth
        .nest("/admin", admin_routes)          // ทุก route ใต้ /admin/* ต้อง auth ทั้งหมด
        .with_state(state)
}
```

ผู้เขียนคอมไพล์โค้ดนี้จริง (ไม่ต้องรันเพราะโครงสร้างพิสูจน์ได้จาก `cargo build` ผ่านแล้ว) — `cargo build`
ผ่านสำเร็จ ยืนยันว่า syntax ของการผสม `.route_layer()` (middleware ที่ครอบเฉพาะ route ในกลุ่มนี้ ไม่ครอบ route
พี่น้องที่ประกาศก่อนหน้า — ต่างจาก `.layer()` ที่ครอบทุก route ที่ประกาศไว้**ก่อนหน้า**มันตามลำดับที่ Part 65
อธิบายไว้) กับ `.nest()` ถูกต้อง

**ข้อดี**: (1) เขียน auth logic ครั้งเดียว ครอบทุก route ใต้ prefix เดียวกันอัตโนมัติ — เพิ่ม route ใหม่ใต้
`/admin` ในอนาคตไม่ต้องจำใส่ extractor เองเลย (2) เหมาะกับกลุ่ม route จำนวนมากที่ **ทุกตัวต้อง auth เหมือนกัน
หมดแบบไม่มีข้อยกเว้น** (เช่น admin panel ทั้งชุด) **ข้อเสีย**: (1) handler ไม่เห็นจาก signature ว่าต้อง auth
— ต้องไปดู router setup แยกจากตัว handler เพื่อรู้ (2) middleware แบบนี้ไม่ได้ "ส่ง" `Claims` เข้า handler
ให้ตรง ๆ (ต่างจาก extractor) — ถ้า handler อยากรู้ว่า "ใคร" login มา ต้องแทรก `Claims` ผ่าน
`req.extensions_mut().insert(claims)` ในตัว middleware เอง แล้ว handler ไปดึงออกด้วย `Extension<Claims>`
(ตามแพทเทิร์นที่ Part 64 หัวข้อ 64.7 สอนไว้ — และต้องรับความเสี่ยงแบบ `Extension<T>` ที่ compiler ไม่การันตี
ว่ามันจะถูก insert จริงตามที่ Part 64 อธิบายไว้)

#### คำแนะนำเลือกใช้

**ใช้ custom extractor เป็นค่าเริ่มต้นเสมอ** เมื่อ handler ต้องใช้ตัวตนของผู้ใช้จริง ๆ (ส่วนใหญ่ของ endpoint
ที่ต้อง auth เป็นแบบนี้ — ต้องรู้ `user_id` เพื่อ query ข้อมูลของ user คนนั้นโดยเฉพาะ) เพราะได้ทั้งความชัดเจน
ใน signature และค่า `Claims` มาใช้ตรง ๆ พร้อมกัน — **สลับไปใช้ middleware ทั้งกลุ่ม** เมื่อมี route จำนวนมาก
ภายใต้ prefix เดียวกันที่ทุกตัวต้อง auth แบบเดียวกันเป๊ะ โดยไม่มีข้อยกเว้น และ handler ส่วนใหญ่ไม่ได้สนใจ
"ใคร" login มา แค่ต้องการ **gate การเข้าถึงเท่านั้น** (เช่น health check ภายในของทีม ops ที่แค่ต้องพิสูจน์ว่า
มาจากคนใน ไม่สนว่าเป็นใคร) — ในระบบจริงหลายระบบใช้**ทั้งสองแบบผสมกัน**: middleware เป็นชั้น gate หยาบ ๆ
(defense in depth) แล้วยังใส่ extractor ในบาง handler ที่ต้องใช้ค่า `Claims` จริง ๆ ก็ทำได้ ไม่ขัดกัน

### 74.11 Refresh Token: Access Token อายุสั้น, Refresh Token อายุยาว

สังเกตว่าโค้ดในหัวข้อ 74.7 ออก **สอง token พร้อมกัน**ทุกครั้งที่ login สำเร็จ: `access_token` (อายุ 15 นาที)
กับ `refresh_token` (อายุ 7 วัน) — ทำไมต้องแยกสองแบบ ไม่ออก token เดียวอายุยาวไปเลย?

#### ทำไม Access Token ต้องอายุสั้น

JWT เป็น **stateless** โดยธรรมชาติ — เมื่อออกไปแล้ว server **ไม่มีทางเรียก token กลับหรือ "revoke" มันก่อน
`exp`** ได้เลย (ต่างจาก session-based auth ที่ Part 75 จะสอน ที่ server เก็บ session id ไว้เองจึงลบทิ้งได้
ทันที) เพราะ `jsonwebtoken::decode` ตรวจแค่ลายเซ็นและ `exp` เท่านั้น ไม่มีการเช็คกับฐานข้อมูลกลางใด ๆ —
**ถ้า access token หลุดไปอยู่ในมือ attacker (เช่นถูกขโมยผ่าน XSS, log ที่รั่ว, หรือ network ที่ไม่ปลอดภัย)
attacker ใช้มันได้จนกว่าจะถึง `exp` โดยที่คุณทำอะไรไม่ได้เลย** — การตั้ง `exp` ให้สั้น (15 นาทีในตัวอย่างนี้
เป็นค่าที่ใช้กันทั่วไป บางระบบใช้สั้นกว่านี้อีก) **จำกัดหน้าต่างความเสียหาย (blast radius)** ให้เล็กที่สุด
ถ้า token หลุดไปจริง ๆ attacker มีเวลาใช้มันได้ไม่เกิน 15 นาทีเท่านั้น

#### ทำไมต้องมี Refresh Token เลย (ทำไมไม่ให้ผู้ใช้ login ใหม่ทุก 15 นาที)

ถ้า access token อายุสั้นอย่างเดียวโดยไม่มีทางต่ออายุ ผู้ใช้จะต้อง**กรอก username/password ใหม่ทุก 15 นาที**
ซึ่งเป็น UX ที่แย่มาก — **refresh token** แก้ปัญหานี้: มันอายุยาวกว่ามาก (7 วันในตัวอย่างนี้) และมีหน้าที่
**เดียว** คือแลกเป็น access token ใบใหม่ (ผ่าน endpoint `/refresh`) โดยไม่ต้องกรอก password ซ้ำ — ผู้ใช้ login
ครั้งเดียว แล้ว client (เช่น mobile app หรือ SPA) เก็บ refresh token ไว้ เรียก `/refresh` อัตโนมัติเบื้องหลัง
ทุกครั้งที่ access token เก่าใกล้หมดอายุ โดยผู้ใช้ไม่รู้สึกอะไรเลย

#### ทำไม Refresh Token ต้อง "Revocable ได้ที่ Server" (สิ่งที่บทนี้ยังทำไม่ครบ — sketch ไว้ก่อน)

นี่คือจุดที่ต้องพูดตรง ๆ อย่างซื่อสัตย์: **refresh token ในตัวอย่างของบทนี้ (ที่เป็น JWT ล้วน ๆ เหมือนกับ
access token) ยังมีข้อจำกัดเดียวกับ access token — server revoke มันก่อน `exp` ไม่ได้เลย** ถ้าผู้ใช้กด
"ออกจากระบบ" หรือสงสัยว่า refresh token หลุด (เช่นเปลี่ยน password เพราะสงสัยว่าโดนแฮก) **ระบบตามที่เขียนไว้
ในบทนี้ไม่มีทางบล็อก refresh token ตัวเก่าได้เลย** มันยังใช้แลก access token ใหม่ได้ต่อไปจนกว่าจะถึง 7 วัน

**ทางแก้ที่ถูกต้องในระบบจริง (ซึ่งบทนี้ทำได้แค่ sketch ระดับออกแบบ รอ Part 83 เรื่อง Redis ที่จะสอนเต็ม
รูปแบบ)**: เก็บ **state ของ refresh token ไว้ฝั่ง server** — วิธีที่ใช้กันทั่วไปคือ:

1. เพิ่ม custom claim `jti` (JWT ID — เป็น standard claim อีกตัวจาก RFC 7519 ที่ยังไม่ได้แนะนำในตารางหัวข้อ
   74.5 เพราะยังไม่มีประโยชน์จนถึงจุดนี้) ให้ refresh token ทุกใบ เป็นค่าสุ่มไม่ซ้ำ (เช่น UUID)
2. เก็บคู่ `(jti, user_id, revoked: bool)` ไว้ใน **Redis** (Part 83) หรือตาราง `refresh_tokens` ใน
   PostgreSQL (Part 70) — ทุกครั้งที่ `/refresh` ถูกเรียก **หลังจาก** ตรวจสอบลายเซ็น/`exp`/`typ` ผ่านแล้ว
   ให้เช็คเพิ่มว่า `jti` นี้ยังไม่ถูก mark ว่า `revoked` ในฐานข้อมูล/Redis ไหม — ถ้าถูก revoke ไปแล้ว ปฏิเสธ
   ทันทีแม้ลายเซ็นและ `exp` จะยังถูกต้องทุกอย่างก็ตาม
3. Endpoint `/logout` แค่ mark `jti` นั้นเป็น `revoked = true` ในฐานข้อมูล/Redis — **ไม่ต้องแก้ไข token
   ที่ออกไปแล้วเลย** (ทำไม่ได้อยู่แล้วเพราะ JWT ที่เซ็นแล้วแก้ไม่ได้) แค่บอกระบบว่า "ถ้าเจอ `jti` นี้มาขอ
   refresh อีก ให้ปฏิเสธ"

**ทำไมทำแบบนี้ที่ refresh token ไม่ใช่ที่ access token**: เพราะ access token อายุสั้นมาก (15 นาที) ความเสี่ยง
จากการ revoke ไม่ได้ทันทีมีจำกัดอยู่แล้วโดยธรรมชาติของ `exp` สั้น — แต่ refresh token อายุยาว (7 วัน) ความ
เสี่ยงถ้า revoke ไม่ได้จะสูงกว่ามาก จึงคุ้มค่าที่จะแบก "ต้นทุนของการเช็คฐานข้อมูล/Redis ทุกครั้งที่ใช้" (ซึ่ง
ทำให้ refresh token ไม่ pure-stateless แบบ JWT ดั้งเดิมอีกต่อไป — เป็นการยอมแลก "ความสามารถ revoke" กับ
"ความเป็น stateless เต็มรูปแบบ" อย่างตั้งใจ) เทียบกับ access token ที่ถูกใช้ถี่กว่ามาก (ทุก request ที่ต้อง
auth) การเช็คฐานข้อมูลทุกครั้งจะกลายเป็นคอขวดด้านประสิทธิภาพทันที — จุดนี้ก็ผูกกับ Part 76 (RBAC) ด้วย: ถ้า
role ของผู้ใช้เปลี่ยน (เช่นถูกลด permission) การบังคับให้ต้อง `/refresh` ใหม่ (แทนที่จะรอ access token เดิม
หมดอายุเอง) คือจุดธรรมชาติที่จะ "รีเฟรช" สิทธิ์ในโปรเจกต์จริง

#### Implement Endpoint `/refresh` จริง (เวอร์ชันที่บทนี้ทำได้ตอนนี้)

```rust
use axum::{extract::State, Json};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};
use serde::Deserialize;
# #[derive(Clone)] struct AppState { users: std::sync::Arc<UserStore>, jwt_secret: std::sync::Arc<String> }
# #[derive(Default)] struct UserStore { by_username: std::sync::Mutex<std::collections::HashMap<String, UserRecord>>, next_id: std::sync::Mutex<u32> }
# #[derive(Debug, Clone)] struct UserRecord { id: u32, username: String, password_hash: String, role: String }
# #[derive(Debug, serde::Serialize, serde::Deserialize, Clone)] struct Claims { sub: String, username: String, role: String, iat: usize, exp: usize, typ: String }
# #[derive(Debug, serde::Serialize)] struct TokenResponse { access_token: String, refresh_token: String, token_type: &'static str, expires_in: i64 }
# #[derive(Debug)] enum JwtRejection { MissingHeader, MalformedHeader, InvalidToken(jsonwebtoken::errors::ErrorKind), WrongTokenType }
# impl axum::response::IntoResponse for JwtRejection { fn into_response(self) -> axum::response::Response { axum::http::StatusCode::UNAUTHORIZED.into_response() } }
# fn issue_token_pair(_s: &AppState, _u: &UserRecord) -> TokenResponse { unimplemented!() }

#[derive(Debug, Deserialize)]
struct RefreshRequest {
    refresh_token: String,
}

async fn refresh(
    State(state): State<AppState>,
    Json(req): Json<RefreshRequest>,
) -> Result<Json<TokenResponse>, JwtRejection> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.set_required_spec_claims(&["exp", "sub"]);
    let decoding_key = DecodingKey::from_secret(state.jwt_secret.as_bytes());

    let token_data = decode::<Claims>(&req.refresh_token, &decoding_key, &validation)
        .map_err(|e| JwtRejection::InvalidToken(e.into_kind()))?;

    if token_data.claims.typ != "refresh" {
        return Err(JwtRejection::WrongTokenType);
    }

    // *** จุดที่ Part 83 (Redis) จะเข้ามาเสริม: เช็ค jti ของ refresh token นี้กับ revocation
    // store ก่อนไปต่อ -- บทนี้ยังไม่มีขั้นนี้ เป็นข้อจำกัดที่ยอมรับตรง ๆ ตามที่อธิบายไว้ข้างบน ***

    let users = state.users.by_username.lock().unwrap();
    let user = users
        .values()
        .find(|u| u.id.to_string() == token_data.claims.sub)
        .ok_or(JwtRejection::WrongTokenType)?;

    Ok(Json(issue_token_pair(&state, user)))
}
```

ทดสอบจริง — ใช้ `refresh_token` จากหัวข้อ 74.7 แลก access token ใบใหม่:

```bash
$ curl -sS -i -X POST http://127.0.0.1:3074/refresh -H 'Content-Type: application/json' \
    -d "{\"refresh_token\":\"$REFRESH\"}"
HTTP/1.1 200 OK
content-type: application/json
content-length: 489

{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...","token_type":"Bearer","expires_in":900}
```

และยืนยันว่าเช็ค `typ` ทำงานถูกต้อง — เอา `refresh_token` ไปเรียก `/me` (endpoint ที่ต้องการ access token
เท่านั้น) ต้องถูกปฏิเสธ:

```bash
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $REFRESH"
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 178

{"error":{"code":"WRONG_TOKEN_TYPE","message":"ใช้ refresh token เรียก endpoint ที่ต้องการ access token (หรือกลับกัน)"}}
```

### 74.12 ระบบ Capstone เต็มรูปแบบ: Signup + Login + Protected Route + Refresh

รวมทุกอย่างจากหัวข้อ 74.6-74.11 เข้าเป็นเซิร์ฟเวอร์ Axum เดียวที่ compile และรันได้จริง:

```rust
use argon2::{
    password_hash::{phc::PasswordHash, PasswordHasher, PasswordVerifier},
    Argon2,
};
use axum::{
    extract::{FromRequestParts, State},
    http::{request::Parts, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use chrono::Utc;
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

// -------- โดเมน: ระบบผู้ใช้ + JWT ต่อยอดจากระบบจองตั๋วของ Part 62-70 --------

#[derive(Debug, Clone)]
struct UserRecord {
    id: u32,
    username: String,
    password_hash: String,
    role: String,
}

#[derive(Default)]
struct UserStore {
    by_username: Mutex<HashMap<String, UserRecord>>,
    next_id: Mutex<u32>,
}

#[derive(Clone)]
struct AppState {
    users: Arc<UserStore>,
    jwt_secret: Arc<String>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Claims {
    sub: String,
    username: String,
    role: String,
    iat: usize,
    exp: usize,
    typ: String,
}

#[derive(Debug, Deserialize)]
struct SignupRequest { username: String, password: String }

#[derive(Debug, Deserialize)]
struct LoginRequest { username: String, password: String }

#[derive(Debug, Serialize)]
struct TokenResponse {
    access_token: String,
    refresh_token: String,
    token_type: &'static str,
    expires_in: i64,
}

#[derive(Debug, Serialize)]
struct MeResponse { user_id: String, username: String, role: String }

#[derive(Debug)]
enum JwtRejection {
    MissingHeader,
    MalformedHeader,
    InvalidToken(jsonwebtoken::errors::ErrorKind),
    WrongTokenType,
}

impl IntoResponse for JwtRejection {
    fn into_response(self) -> Response {
        let (status, code, message) = match &self {
            JwtRejection::MissingHeader => (StatusCode::UNAUTHORIZED, "MISSING_TOKEN",
                "ไม่พบ header Authorization: Bearer <token>".to_string()),
            JwtRejection::MalformedHeader => (StatusCode::UNAUTHORIZED, "MALFORMED_HEADER",
                "รูปแบบ Authorization header ไม่ถูกต้อง (ต้องเป็น 'Bearer <token>')".to_string()),
            JwtRejection::WrongTokenType => (StatusCode::UNAUTHORIZED, "WRONG_TOKEN_TYPE",
                "ใช้ refresh token เรียก endpoint ที่ต้องการ access token (หรือกลับกัน)".to_string()),
            JwtRejection::InvalidToken(kind) => {
                let msg = match kind {
                    jsonwebtoken::errors::ErrorKind::ExpiredSignature =>
                        "token หมดอายุแล้ว กรุณาเข้าสู่ระบบใหม่หรือใช้ refresh token".to_string(),
                    jsonwebtoken::errors::ErrorKind::InvalidSignature =>
                        "ลายเซ็นของ token ไม่ถูกต้อง (token อาจถูกแก้ไขหรือเซ็นด้วย key ผิด)".to_string(),
                    other => format!("token ไม่ผ่านการตรวจสอบ: {other:?}"),
                };
                (StatusCode::UNAUTHORIZED, "INVALID_TOKEN", msg)
            }
        };
        (status, Json(serde_json::json!({ "error": { "code": code, "message": message } }))).into_response()
    }
}

struct CurrentUser { claims: Claims }

impl FromRequestParts<AppState> for CurrentUser {
    type Rejection = JwtRejection;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, Self::Rejection> {
        let header_value = parts.headers.get("Authorization")
            .ok_or(JwtRejection::MissingHeader)?
            .to_str().map_err(|_| JwtRejection::MalformedHeader)?;
        let token = header_value.strip_prefix("Bearer ").ok_or(JwtRejection::MalformedHeader)?;

        let mut validation = Validation::new(Algorithm::HS256);
        validation.set_required_spec_claims(&["exp", "sub"]);
        let decoding_key = DecodingKey::from_secret(state.jwt_secret.as_bytes());
        let token_data = decode::<Claims>(token, &decoding_key, &validation)
            .map_err(|e| JwtRejection::InvalidToken(e.into_kind()))?;

        if token_data.claims.typ != "access" {
            return Err(JwtRejection::WrongTokenType);
        }
        Ok(CurrentUser { claims: token_data.claims })
    }
}

async fn signup(
    State(state): State<AppState>,
    Json(req): Json<SignupRequest>,
) -> Result<StatusCode, (StatusCode, Json<serde_json::Value>)> {
    let mut users = state.users.by_username.lock().unwrap();
    if users.contains_key(&req.username) {
        return Err((StatusCode::CONFLICT,
            Json(serde_json::json!({"error": {"code": "USERNAME_TAKEN", "message": "username นี้ถูกใช้ไปแล้ว"}}))));
    }

    let argon2 = Argon2::default();
    let password_hash = argon2.hash_password(req.password.as_bytes())
        .map_err(|_| (StatusCode::INTERNAL_SERVER_ERROR,
            Json(serde_json::json!({"error": {"code": "HASH_ERROR", "message": "hash password ไม่สำเร็จ"}}))))?
        .to_string();

    let mut next_id = state.users.next_id.lock().unwrap();
    let id = *next_id;
    *next_id += 1;
    users.insert(req.username.clone(),
        UserRecord { id, username: req.username, password_hash, role: "member".to_string() });

    Ok(StatusCode::CREATED)
}

fn issue_token_pair(state: &AppState, user: &UserRecord) -> TokenResponse {
    let now = Utc::now().timestamp() as usize;
    let access_ttl = 900;
    let refresh_ttl = 60 * 60 * 24 * 7;
    let header = Header::new(Algorithm::HS256);
    let key = EncodingKey::from_secret(state.jwt_secret.as_bytes());

    let access_claims = Claims { sub: user.id.to_string(), username: user.username.clone(),
        role: user.role.clone(), iat: now, exp: now + access_ttl, typ: "access".to_string() };
    let refresh_claims = Claims { sub: user.id.to_string(), username: user.username.clone(),
        role: user.role.clone(), iat: now, exp: now + refresh_ttl, typ: "refresh".to_string() };

    TokenResponse {
        access_token: encode(&header, &access_claims, &key).unwrap(),
        refresh_token: encode(&header, &refresh_claims, &key).unwrap(),
        token_type: "Bearer",
        expires_in: access_ttl as i64,
    }
}

async fn login(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>,
) -> Result<Json<TokenResponse>, (StatusCode, Json<serde_json::Value>)> {
    let users = state.users.by_username.lock().unwrap();
    let user = users.get(&req.username).ok_or_else(|| (StatusCode::UNAUTHORIZED,
        Json(serde_json::json!({"error": {"code": "INVALID_CREDENTIALS", "message": "username หรือ password ไม่ถูกต้อง"}}))))?;

    let parsed_hash = PasswordHash::new(&user.password_hash).map_err(|_| (StatusCode::INTERNAL_SERVER_ERROR,
        Json(serde_json::json!({"error": {"code": "HASH_ERROR", "message": "hash เสียหาย"}}))))?;

    if Argon2::default().verify_password(req.password.as_bytes(), &parsed_hash).is_err() {
        return Err((StatusCode::UNAUTHORIZED,
            Json(serde_json::json!({"error": {"code": "INVALID_CREDENTIALS", "message": "username หรือ password ไม่ถูกต้อง"}}))));
    }

    Ok(Json(issue_token_pair(&state, user)))
}

async fn me(current_user: CurrentUser) -> Json<MeResponse> {
    Json(MeResponse { user_id: current_user.claims.sub, username: current_user.claims.username,
        role: current_user.claims.role })
}

#[derive(Debug, Deserialize)]
struct RefreshRequest { refresh_token: String }

async fn refresh(
    State(state): State<AppState>,
    Json(req): Json<RefreshRequest>,
) -> Result<Json<TokenResponse>, JwtRejection> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.set_required_spec_claims(&["exp", "sub"]);
    let decoding_key = DecodingKey::from_secret(state.jwt_secret.as_bytes());

    let token_data = decode::<Claims>(&req.refresh_token, &decoding_key, &validation)
        .map_err(|e| JwtRejection::InvalidToken(e.into_kind()))?;
    if token_data.claims.typ != "refresh" {
        return Err(JwtRejection::WrongTokenType);
    }

    let users = state.users.by_username.lock().unwrap();
    let user = users.values().find(|u| u.id.to_string() == token_data.claims.sub)
        .ok_or(JwtRejection::WrongTokenType)?;

    Ok(Json(issue_token_pair(&state, user)))
}

#[tokio::main]
async fn main() {
    let state = AppState {
        users: Arc::new(UserStore::default()),
        // *** อ่าน secret จาก environment variable เสมอ -- ไม่ hardcode ในโค้ด (ดูหัวข้อกับดัก) ***
        jwt_secret: Arc::new(
            std::env::var("JWT_SECRET").unwrap_or_else(|_| "dev-only-secret-do-not-use-in-prod".to_string()),
        ),
    };

    let app: Router = Router::new()
        .route("/signup", post(signup))
        .route("/login", post(login))
        .route("/me", get(me))
        .route("/refresh", post(refresh))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3074").await.unwrap();
    println!("jwt_demo ฟังอยู่ที่ http://127.0.0.1:3074");
    axum::serve(listener, app).await.unwrap();
}
```

`cargo build` ผ่านสำเร็จโดยไม่มี error หรือ warning ที่เกี่ยวข้อง — รันจริงและทดสอบ end-to-end เต็มเส้นทางด้วย
`curl` (ทุกบรรทัดคือผลลัพธ์จริง เรียงตามลำดับที่รันจริง):

```bash
### 1) signup ###
$ curl -sS -i -X POST http://127.0.0.1:3074/signup -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"correct-horse-battery-staple"}'
HTTP/1.1 201 Created
content-length: 0

### 2) signup ซ้ำ username เดิม -> 409 ###
$ curl -sS -i -X POST http://127.0.0.1:3074/signup -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"anything"}'
HTTP/1.1 409 Conflict
content-type: application/json

{"error":{"code":"USERNAME_TAKEN","message":"username นี้ถูกใช้ไปแล้ว"}}

### 3) login ด้วย password ผิด -> 401 ###
$ curl -sS -i -X POST http://127.0.0.1:3074/login -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"wrong-password"}'
HTTP/1.1 401 Unauthorized
content-type: application/json

{"error":{"code":"INVALID_CREDENTIALS","message":"username หรือ password ไม่ถูกต้อง"}}

### 4) login สำเร็จ -> ได้ access_token + refresh_token ###
$ curl -sS -X POST http://127.0.0.1:3074/login -H 'Content-Type: application/json' \
    -d '{"username":"nan","password":"correct-horse-battery-staple"}'
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIwIiwidXNlcm5hbWUiOiJuYW4iLCJyb2xlIjoibWVtYmVyIiwiaWF0IjoxNzkwNDY5NDE1LCJleHAiOjE3OTA0NzAzMTUsInR5cCI6ImFjY2VzcyJ9.-cn3jy_CRnSS7bMATfVW3um5YdRfqHk_HOEXnY19Yyo","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIwIiwidXNlcm5hbWUiOiJuYW4iLCJyb2xlIjoibWVtYmVyIiwiaWF0IjoxNzkwNDY5NDE1LCJleHAiOjE3OTEwNzQyMTUsInR5cCI6InJlZnJlc2gifQ.B-xmK30klQjijXIAjbcGSUhjUUjXC3M9K7CfbrE50f0","token_type":"Bearer","expires_in":900}

### 5) เรียก /me ด้วย access_token ที่ถูก -> 200 ###
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $ACCESS"
HTTP/1.1 200 OK
content-type: application/json

{"user_id":"0","username":"nan","role":"member"}

### 6) เรียก /me โดยไม่ส่ง header เลย -> 401 ###
$ curl -sS -i http://127.0.0.1:3074/me
HTTP/1.1 401 Unauthorized
content-type: application/json

{"error":{"code":"MISSING_TOKEN","message":"ไม่พบ header Authorization: Bearer <token>"}}

### 7) เรียก /me ด้วย token ปลอม (พิมพ์เอง ไม่ใช่ JWT จริง) -> 401 ###
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer not.a.realtoken"
HTTP/1.1 401 Unauthorized
content-type: application/json

{"error":{"code":"INVALID_TOKEN","message":"token ไม่ผ่านการตรวจสอบ: Base64(InvalidLastSymbol(2, 116))"}}

### 8) เรียก /me ด้วย access_token ที่หมดอายุแล้ว (exp เมื่อ 1 ชั่วโมงก่อน) -> 401 ###
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $EXPIRED"
HTTP/1.1 401 Unauthorized
content-type: application/json

{"error":{"code":"INVALID_TOKEN","message":"token หมดอายุแล้ว กรุณาเข้าสู่ระบบใหม่หรือใช้ refresh token"}}

### 9) refresh ด้วย refresh_token -> ได้ access_token ใบใหม่ ###
$ curl -sS -i -X POST http://127.0.0.1:3074/refresh -H 'Content-Type: application/json' \
    -d "{\"refresh_token\":\"$REFRESH\"}"
HTTP/1.1 200 OK
content-type: application/json

{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...","refresh_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...","token_type":"Bearer","expires_in":900}

### 10) เรียก /me ด้วย refresh_token (ผิดประเภท token) -> 401 ###
$ curl -sS -i http://127.0.0.1:3074/me -H "Authorization: Bearer $REFRESH"
HTTP/1.1 401 Unauthorized
content-type: application/json

{"error":{"code":"WRONG_TOKEN_TYPE","message":"ใช้ refresh token เรียก endpoint ที่ต้องการ access token (หรือกลับกัน)"}}
```

ครบทุกเส้นทางตั้งแต่สมัครสมาชิกจนถึงใช้งาน/refresh/ปฏิเสธ token ผิดประเภท — ระบบเดียวที่โตขึ้นมาตั้งแต่หัวข้อ
74.6 ทำงานสอดคล้องกันทั้งหมด

## กับดักที่พบบ่อย (Common Pitfalls)

บทนี้เป็นเรื่องความปลอดภัยโดยตรง — กับดักในหัวข้อนี้ **ร้ายแรงกว่าปกติมาก** เพราะพลาดแค่จุดเดียวอาจแปลว่า
ระบบทั้งระบบถูก bypass auth ได้ทั้งหมด ไม่ใช่แค่ "โค้ดพัง" ธรรมดา อ่านให้ครบทุกข้อก่อนเอาระบบคล้ายบทนี้ขึ้น
production จริง

### 1. Algorithm Confusion: ไม่ปักหมุด algorithm ที่ยอมรับไว้ตายตัว (`alg: none` และการเชื่อ `alg` จาก header)

**ปัญหา**: header ของ JWT มี field `alg` ที่บอกว่า token นี้เซ็นด้วย algorithm อะไร — attacker ที่ควบคุม
เนื้อหาของ token ได้ (เพราะอย่างที่หัวข้อ 74.1 พิสูจน์ไว้ ทุกส่วนของ JWT แก้ไขได้ ยกเว้นต้องผ่านการตรวจ
signature) สามารถ**แก้ header ให้ `alg` เป็น `"none"`** (แปลว่า "ไม่มีการเซ็นเลย") **แล้วตัด signature ทิ้ง
ไปเลย** — ถ้า library ฝั่งตรวจสอบ **เชื่อ `alg` จาก header ของ token ที่รับมาตรง ๆ** (แทนที่จะกำหนดไว้ตายตัว
ฝั่งเซิร์ฟเวอร์เอง) มันอาจ "ยอมรับ" ว่า token นี้ไม่ต้องมี signature เลย ปลอม token อะไรก็ได้ผ่านการตรวจสอบ
ทันที — นี่คือ **JWT algorithm confusion attack** ที่เป็นช่องโหว่จริงที่เคยพบใน library JWT หลายภาษาในอดีต

**การป้องกันใน `jsonwebtoken`**: `Validation::new(Algorithm::HS256)` (หรือ `RS256`/`ES256` แล้วแต่ระบบ)
**ปักหมุด algorithm ที่ยอมรับไว้ตายตัวฝั่งเซิร์ฟเวอร์** — `decode()` จะปฏิเสธ token ที่ header ประกาศ
algorithm อื่นทันที **โดยไม่แม้แต่จะไปเชื่อค่าที่ token บอกมา** — พิสูจน์ด้วยโค้ดจริง: ลองสร้าง token ปลอม
ด้วยมือที่ header ประกาศ `"alg":"none"` แล้วไม่มี signature เลย (ไม่ได้ใช้ `jsonwebtoken::encode` สร้าง เพราะ
มันไม่มี `Algorithm::None` ให้เลือกด้วยซ้ำ — ต้องประกอบ base64url เองเพื่อจำลอง attacker):

```rust
use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine as _};
use jsonwebtoken::{decode, Algorithm, DecodingKey, Validation};
# #[derive(serde::Serialize, serde::Deserialize, Debug)] struct Claims { sub: String, exp: usize }

let fake_header = URL_SAFE_NO_PAD.encode(br#"{"typ":"JWT","alg":"none"}"#);
let fake_payload = URL_SAFE_NO_PAD.encode(br#"{"sub":"1","exp":9999999999}"#);
let none_alg_token = format!("{fake_header}.{fake_payload}."); // ไม่มี signature เลย

let decoding_key = DecodingKey::from_secret(b"demo-secret-for-pitfalls");
let validation = Validation::new(Algorithm::HS256);
let result = decode::<Claims>(&none_alg_token, &decoding_key, &validation);
println!("{:?}", result);
```

ผลลัพธ์จริง — decode ล้มเหลวตั้งแต่ขั้น**parse header** ด้วยซ้ำ ก่อนจะไปถึงขั้นเทียบ algorithm กับ
`Validation` เสียอีก:

```
Err(Error(Json(Error("unknown variant `none`, expected one of `HS256`, `HS384`, `HS512`, `ES256`, `ES384`,
`RS256`, `RS384`, `RS512`, `PS256`, `PS384`, `PS512`, `EdDSA`", line: 1, column: 25))))
```

**สิ่งที่น่าสนใจที่พบจริง**: การป้องกันในกรณีนี้แข็งแรงกว่าที่คาดไว้อีกชั้น — **enum `Algorithm` ของ
`jsonwebtoken` ไม่มี variant สำหรับ `"none"` อยู่เลยด้วยซ้ำ** ทำให้การ deserialize header ที่มี `"alg":"none"`
**ล้มเหลวทันทีที่ระดับ type** ก่อนจะไปถึงขั้นตรวจสอบ `Validation.algorithms` เสียอีก — นี่คือตัวอย่างที่ดีของ
การใช้ type system ป้องกันบั๊กเชิงความปลอดภัยตั้งแต่การออกแบบ (แนวคิดเดียวกับ typestate pattern ที่ Part 53
สอนไว้) — **แต่บทเรียนสำคัญที่ต้องจำคือ: อย่าพึ่งพาเฉพาะ "library เวอร์ชันนี้บังเอิญป้องกันได้" — เขียน
`Validation::new(Algorithm::X)` ที่ปักหมุดชัดเจนเสมอ ไม่ว่า library จะป้องกันชั้นอื่นให้ด้วยหรือไม่** เพราะ
library อื่นในภาษาอื่น (หรือ `jsonwebtoken` เวอร์ชันอนาคตที่ออกแบบต่างไป) อาจไม่มีการป้องกันชั้นนี้ให้ฟรี ๆ

**กับดักที่ซ่อนอยู่อีกชั้น — Key Confusion Attack**: ถ้าระบบตั้ง `Validation` ให้ยอมรับ**หลาย algorithm พร้อม
กัน** (เช่นยอมรับทั้ง `RS256` และ `HS256`) และใช้ค่าตัวเดียวกัน (เช่น RSA public key ที่เป็น PEM string) เป็น
ทั้ง "public key สำหรับตรวจ RS256" และ (โดยไม่ตั้งใจ) "secret สำหรับตรวจ HS256" — attacker ที่รู้ public key
(ซึ่งเปิดเผยได้อยู่แล้วตามหัวข้อ 74.4) สามารถ**เอา public key string นั้นมาเป็น HMAC secret** เซ็น token
`HS256` ปลอมขึ้นมา แล้ว server (ที่ยอมรับทั้งสอง algorithm) จะตรวจสอบผ่านเพราะดันไปใช้ค่าเดียวกันเป็น HMAC
secret จริง ๆ! **การป้องกัน**: ปักหมุด `Validation` ให้ยอมรับ **algorithm เดียวเท่านั้นเสมอ** ต่อ endpoint/
ต่อ token type (`Validation::new(Algorithm::HS256)` แบบในตัวอย่างบทนี้ ไม่ใช่ตั้ง `.algorithms` เป็น list ที่
มีหลายตัว) และไม่เคยใช้ค่าตัวเดียวกันเป็นทั้ง public key และ HMAC secret ในระบบเดียวกัน

### 2. ปิดการตรวจสอบ `exp` เอง (`validation.validate_exp = false`)

**ปัญหา**: `jsonwebtoken::Validation` มี field `validate_exp` ที่ **default เป็น `true` เสมอ** (ตรวจสอบ
`exp` อัตโนมัติทุกครั้งที่ `decode()`) — แต่มันเปิดให้ปิดได้ (`validation.validate_exp = false`) เผื่อ use
case พิเศษบางอย่าง (เช่นต้องการ parse claims ของ token เก่าเพื่อ debug โดยไม่สนว่าหมดอายุไปแล้วหรือยัง) —
**ถ้าโค้ด production เผลอปิดค่านี้ (หรือ copy ตัวอย่าง debug มาใช้จริงโดยไม่ได้ตั้งใจ) ระบบจะยอมรับ token ที่
หมดอายุไปแล้วนานเท่าไหร่ก็ได้** พิสูจน์ด้วยโค้ดจริง:

```rust
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
# #[derive(Debug, serde::Serialize, serde::Deserialize)] struct Claims { sub: String, exp: usize }

let secret = b"demo-secret-for-pitfalls";
let now = chrono::Utc::now().timestamp() as usize;
let expired_claims = Claims { sub: "1".into(), exp: now - 3600 }; // หมดอายุไป 1 ชั่วโมงแล้ว
let expired_token = encode(&Header::new(Algorithm::HS256), &expired_claims, &EncodingKey::from_secret(secret)).unwrap();

let mut no_exp_check = Validation::new(Algorithm::HS256);
no_exp_check.validate_exp = false; // *** อย่าทำแบบนี้ในโค้ดจริงเด็ดขาด ***
let result = decode::<Claims>(&expired_token, &DecodingKey::from_secret(secret), &no_exp_check);
println!("{:?}", result);
```

ผลลัพธ์จริง — decode "สำเร็จ" ทั้งที่หมดอายุไปแล้วเต็ม ๆ 1 ชั่วโมง:

```
decode 'สำเร็จ' ทั้งที่ exp หมดอายุไปแล้ว 1 ชั่วโมง!! claims=Claims { sub: "1", exp: 1790465595 }
```

เทียบกับพฤติกรรม default (ไม่ปิดอะไรเลย) ที่ decode คืน error ทันที:

```
decode คืน Err จริง: Error(ExpiredSignature)
ErrorKind: ExpiredSignature
```

**บทเรียน**: อย่าปิด `validate_exp` ในโค้ด production เด็ดขาด — ถ้าเจอความจำเป็นต้องอ่าน claims ของ token
ที่หมดอายุไปแล้ว (เช่นเพื่อ log ว่า "ใครพยายามใช้ token หมดอายุ") ให้ `decode()` แบบปกติก่อน (จะได้ `Err` ตาม
ที่คาด) แล้วถ้าต้องการ claims จริง ๆ ค่อยแยกเขียน path พิเศษที่ทำเครื่องหมายชัดเจนว่า "เส้นทางนี้ไม่ใช่เส้นทาง
auth ปกติ" ไม่ใช่เผลอปิด flag นี้ในเส้นทาง auth หลัก

### 3. เก็บ JWT ไว้ฝั่ง Client แบบไม่ปลอดภัย: `localStorage` vs httpOnly Cookie

**ปัญหา**: บทนี้โฟกัสที่ฝั่ง server ทั้งหมด แต่ token ที่ออกไปต้องถูกเก็บไว้ที่**ไหนสักที่ฝั่ง client** —
ตัวเลือกที่นิยมสองแบบมี trade-off ที่ต่างกันมาก และเลือกผิดคือช่องโหว่ทันที:

- **`localStorage`** — JavaScript อ่าน/เขียนได้ตรง ๆ ง่ายต่อการ implement (SPA framework จำนวนมากทำแบบนี้
  เป็นค่า default) แต่ **ถ้าเว็บมีช่องโหว่ XSS (Cross-Site Scripting) แม้แต่จุดเดียว attacker ที่ inject
  JavaScript สำเร็จจะอ่าน token จาก `localStorage` ได้ตรง ๆ ทันที** (`localStorage.getItem(...)` ธรรมดา ไม่มี
  การป้องกันอะไรเลยในระดับ browser) แล้วส่ง token นั้นไปให้ตัวเองใช้งานแทนผู้ใช้จริงได้เลย
- **httpOnly Cookie** — browser ไม่ยอมให้ JavaScript อ่านค่าคุกกี้ที่ตั้ง flag `HttpOnly` ได้เลย (ป้องกัน XSS
  ในมุมนี้ได้ดีกว่ามาก) แต่คุกกี้ถูกส่งไปกับทุก request ไปยัง domain นั้นโดยอัตโนมัติ (รวม request ที่มาจาก
  เว็บอื่นด้วย ถ้าไม่ตั้ง `SameSite` ให้ถูกต้อง) ทำให้เปิดช่องโหว่ **CSRF (Cross-Site Request Forgery)** แทน
  — ต้องป้องกันเพิ่มด้วย `SameSite=Strict`/`Lax` และ/หรือ CSRF token คู่กัน

**ไม่มีคำตอบที่ "ถูกเสมอ" — ต้องเข้าใจ trade-off**: httpOnly cookie ปิดความเสี่ยง XSS-อ่าน-token ได้ดีกว่า
แต่เปิดความเสี่ยง CSRF ที่ต้องจัดการเพิ่ม ส่วน `localStorage` ไม่มีความเสี่ยง CSRF เลย (เพราะไม่ได้ส่งอัตโนมัติ
กับ request) แต่เสี่ยง XSS เต็ม ๆ ถ้าเว็บมีช่องโหว่นั้น — Part 75 (Session-based และ OAuth2) จะลงรายละเอียด
เรื่อง cookie-based session auth เต็มรูปแบบ ซึ่งเป็นกรณีที่ httpOnly cookie เหมาะสมกว่า JWT ที่ต้องส่งด้วย
`Authorization` header เอง (เพราะ header ต้องให้ JavaScript แนบเอง จึงมักจับคู่กับ token ที่เก็บใน memory
ของ SPA แทน `localStorage` เพื่อลดหน้าต่างความเสี่ยง XSS — รายละเอียดเชิงลึกเกินสโคปของบทนี้ที่โฟกัสฝั่ง
server แต่ต้อง**รู้ตัวไว้เสมอว่าการเลือกที่เก็บ token ฝั่ง client มีผลต่อความปลอดภัยจริง ไม่ใช่รายละเอียดเล็ก
น้อย**

### 4. Commit Secret เข้า Source Control / Hardcode Secret ในโค้ด

**ปัญหา**: `EncodingKey::from_secret(...)` และ `DecodingKey::from_secret(...)` ต้องการ byte string เป็น
secret — ถ้า hardcode secret นี้ตรง ๆ ในโค้ดแล้ว commit เข้า git (เช่น `let key =
EncodingKey::from_secret(b"my-actual-production-secret");`) **ใครก็ตามที่เข้าถึง repository ได้ (รวม
history เก่าที่อาจ leak ผ่าน fork/clone ไปแล้ว แม้จะลบออกจาก commit ล่าสุด) จะปลอมสร้าง JWT อะไรก็ได้ที่ผ่าน
การตรวจสอบของระบบทันที** — ต่างจาก password ที่รั่วแล้วยัง "แค่" เข้าบัญชีเดียวได้ การที่ secret สำหรับเซ็น
JWT รั่ว **แปลว่าปลอม token ของ user คนไหนก็ได้ ด้วย role อะไรก็ได้ (ถ้ารู้โครงสร้าง Claims)** — ถือเป็นข้อมูล
ลับระดับสูงสุดของระบบทั้งระบบ

**การป้องกันตามหลักการที่หลักสูตรนี้ใช้ตลอด (เหมือน `DATABASE_URL` ใน Part 70)**: อ่าน secret จาก
**environment variable** เสมอ ไม่ hardcode ในโค้ด:

```rust
let jwt_secret = std::env::var("JWT_SECRET")
    .expect("ต้องตั้ง JWT_SECRET ก่อนรันโปรแกรม (ห้ามมีค่า default ที่ใช้งานได้จริงใน production)");
```

**ข้อสังเกตที่ต้องซื่อสัตย์เกี่ยวกับโค้ดในบทนี้เอง**: โค้ดตัวอย่างทั้งบทใช้
`std::env::var("JWT_SECRET").unwrap_or_else(|_| "dev-only-secret-do-not-use-in-prod".to_string())` — มี
**ค่า fallback ที่ใช้งานได้จริง** ถ้าไม่ได้ตั้ง environment variable ไว้ ซึ่งสะดวกมากสำหรับการรันตัวอย่างใน
บทเรียนนี้ (ไม่ต้องตั้งค่าอะไรก่อนรัน) **แต่เป็นแพทเทิร์นที่อันตรายถ้า copy ไปใช้ production ตรง ๆ** — ถ้า
ทีมลืมตั้ง `JWT_SECRET` จริงตอน deploy ระบบจะยัง "ทำงานได้" (ไม่ error ให้เห็นทันที) แต่ใช้ secret ที่**อยู่ใน
source code สาธารณะของบทเรียนนี้** ซึ่งใครก็ pastebin ไปหาได้ — ในโค้ด production จริง ควรเปลี่ยนจาก
`.unwrap_or_else(...)` เป็น `.expect(...)` ตรง ๆ (panic ทันทีตอน startup ถ้าไม่ได้ตั้งค่า ดีกว่ารันต่อไปแบบไม่
ปลอดภัยแบบเงียบ ๆ) หรือดีกว่านั้นคือเช็คด้วย `if cfg!(debug_assertions)` แยก path dev/production ออกจากกัน
ชัดเจน

### 5. ไม่แยกประเภท Token (Access ↔ Refresh สับสนกัน)

**ปัญหา**: ถ้า Claims ของ access token กับ refresh token มีโครงสร้างเหมือนกันเป๊ะ (ต่างกันแค่ `exp`) และไม่มี
field ไหนบอกว่า "นี่คือ token ประเภทไหน" — ผู้ใช้ที่มีแค่ refresh token (ซึ่งควรใช้ได้แค่กับ `/refresh`
เท่านั้น) จะสามารถเอามันไปเรียก endpoint อื่นที่ต้องการ access token ได้เลย เพราะ signature และ `exp` (ที่ยัง
ไม่หมดอายุ เพราะ refresh token อายุยาวกว่า) ผ่านการตรวจสอบทุกอย่าง — บทนี้แก้ด้วย custom claim `typ`
(`"access"` หรือ `"refresh"`) แล้วเช็คเพิ่มทุกครั้งหลัง `decode()` สำเร็จ (ดูหัวข้อ 74.8/74.11) — พิสูจน์ด้วย
`curl` จริงไปแล้วในหัวข้อ 74.9/74.11 ว่าการเช็คนี้ทำงานถูกต้อง (`WRONG_TOKEN_TYPE` 401)

**ถ้าลืมเช็คจุดนี้**: ผลกระทบคือ refresh token (อายุยาว 7 วัน) กลายเป็น "access token ที่อายุยาวกว่าที่ตั้งใจ
มาก" โดยไม่รู้ตัว — ทำให้หน้าต่างความเสียหายถ้า token หลุด (ตามที่อธิบายในหัวข้อ 74.11) ยาวขึ้นจาก 15 นาทีเป็น
7 วันทันที ซึ่งขัดกับเหตุผลทั้งหมดที่แยก access/refresh token ออกจากกันตั้งแต่ต้น

### 6. Secret สำหรับ HS256 สั้น/คาดเดาง่ายเกินไป

**ปัญหา**: `EncodingKey::from_secret(...)` รับ byte slice **ความยาวเท่าไหร่ก็ได้** — ไม่มีการบังคับความยาว
ขั้นต่ำจาก library — ถ้าตั้ง secret สั้นเกินไป (เช่น `b"abc123"` หรือคำที่เดาง่าย) attacker ที่มี token จริง
สักใบ (ซึ่งได้ signature จริงมาด้วย) สามารถ **brute-force หา secret** ได้ด้วยการลอง secret ที่เป็นไปได้ทุก
แบบจนกว่า HMAC ที่คำนวณได้จะตรงกับ signature ที่เห็น — ยิ่ง secret สั้น/เดาง่าย ยิ่งทำได้เร็ว (คล้ายกับปัญหา
password สั้นที่ argon2 ในหัวข้อ 74.6 พยายามป้องกัน แต่ตรงนี้ไม่มี argon2 มาช่วย เพราะ secret ของ HS256 ไม่ได้
ถูก hash — มันถูกใช้ตรง ๆ เป็น HMAC key) — **แนวทางที่ถูกต้อง**: ใช้ secret ที่เป็น **random byte string
ความยาวอย่างน้อย 32 byte (256 bit)** ที่สุ่มด้วย cryptographically secure random generator (เช่น
`openssl rand -base64 32`) ไม่ใช่คำหรือประโยคที่มนุษย์คิดขึ้นเอง

### 7. ลืมเปิด Crypto Backend Feature ของ `jsonwebtoken` 11.x

ตามที่พิสูจน์ไว้แล้วในหัวข้อ 74.3 — ถ้า `cargo add jsonwebtoken` โดยไม่เปิด feature `rust_crypto` หรือ
`aws_lc_rs` โปรแกรมจะ **compile ผ่านสำเร็จ** (ไม่มี compile error เลย) แต่ **panic ทันทีตอน runtime** ที่
เรียก `encode`/`decode` ครั้งแรก:

```
thread 'main' panicked at .../jsonwebtoken-11.1.0/src/crypto/mod.rs:124:40:

Could not automatically determine the process-level CryptoProvider from jsonwebtoken crate features.
Call CryptoProvider::install_default() before this point to select a provider manually, or make sure
exactly one of the 'rust_crypto' and 'aws_lc_rs' features is enabled.
```

นี่คือกับดักเฉพาะเวอร์ชันที่ **ไม่มีทางรู้จาก `cargo build`** เพราะ backend ถูก resolve แบบ dynamic ตอน
runtime ไม่ใช่ compile time — วิธีป้องกันคือเขียน integration test ง่าย ๆ ที่เรียก `encode`/`decode` จริง
สักครั้งเป็นส่วนหนึ่งของ CI (ไม่ใช่แค่ `cargo build` เฉย ๆ) เพื่อให้ panic แบบนี้โผล่ให้เห็นก่อนขึ้น
production เสมอ — เตือนไว้อีกครั้งว่าพฤติกรรมนี้ผูกกับเวอร์ชัน `jsonwebtoken` ที่ระบุในบทนี้ (11.1.0) เวอร์ชัน
ใหม่กว่าอาจเปลี่ยนแปลงพฤติกรรมนี้ได้ ให้ตรวจสอบ error จริงจากเวอร์ชันที่คุณใช้เสมอ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม standard claims `iss` (issuer) และ `aud` (audience) เข้า `Claims` struct — ตั้งค่า
   `iss` เป็น `"rust-course-auth"` ทุกครั้งที่ออก token และ `aud` เป็น `"rust-course-api"` แล้วใน
   `CurrentUser::from_request_parts` เรียก `validation.set_issuer(&["rust-course-auth"])` และ
   `validation.set_audience(&["rust-course-api"])` เพิ่ม ทดสอบด้วยการสร้าง token ที่ `aud` เป็นค่าอื่น (เช่น
   `"some-other-api"`) แล้วยิงไปที่ `/me` — ต้องได้ `401` แม้ signature/`exp`/`typ` จะถูกต้องทุกอย่าง (hint:
   ดู error variant `ErrorKind::InvalidAudience` ที่จะได้จาก `jsonwebtoken`)

2. **(กลาง)** สลับจาก `HashMap<String, UserRecord>` เป็น `PgPool` จริงตามแนวทางที่ Part 70 สอนไว้ — สร้าง
   ตาราง `users(id BIGSERIAL PRIMARY KEY, username TEXT UNIQUE NOT NULL, password_hash TEXT NOT NULL, role
   TEXT NOT NULL)` ด้วย `sqlx migrate add`, เขียน `find_user_by_username`/`insert_user` ด้วย
   `sqlx::query_as!`/`sqlx::query!` ที่ compile-time checked ตามที่ Part 70 หัวข้อ 70.4 สอนไว้ (hint: โค้ด
   ส่วน argon2/JWT ทั้งหมดในบทนี้**ไม่ต้องแก้เลย** — แก้แค่ที่ `signup`/`login` เรียก query แทนการ
   `.lock()`/`HashMap` ตรง ๆ)

3. **(ยาก)** implement การ revoke refresh token แบบพื้นฐาน (จำลอง Redis ก่อนถึง Part 83 ด้วย
   `Arc<Mutex<HashSet<String>>>` ใน `AppState`) — เพิ่ม claim `jti` (สุ่มด้วย UUID ทุกครั้งที่ออก refresh
   token ใหม่) เก็บ set ของ `jti` ที่ถูก revoke ไว้ เพิ่ม endpoint `POST /logout` ที่รับ refresh token ปัจจุบัน
   แล้ว insert `jti` ของมันเข้า set นั้น จากนั้นแก้ `/refresh` ให้เช็คว่า `jti` ของ token ที่ส่งมาอยู่ใน
   revoked set ไหม **ก่อน**จะออก token ใหม่ให้ (hint: ต้องเช็คแม้ signature/`exp`/`typ` จะถูกต้องครบทุกอย่าง
   — revocation check เป็นชั้นที่เพิ่มเข้ามา**เหนือ**การตรวจสอบ JWT ปกติ ไม่ใช่แทนมัน)

4. **(ประยุกต์ใช้จริง)** จำลองสถาปัตยกรรม microservices แบบง่าย ๆ ที่ Part 81 จะสอนเต็มรูปแบบ — แยกโค้ดเป็น
   สอง binary: `auth_service` (ถือ RSA private key, มี endpoint `/login` ที่ออก `RS256` token เท่านั้น) และ
   `resource_service` (ถือแค่ RSA public key ที่อ่านจาก path แยกผ่าน environment variable, มี endpoint
   `/protected` ที่ใช้ `CurrentUser` extractor ตรวจสอบด้วย public key เท่านั้น — **ไม่มี private key อยู่ใน
   binary นี้เลย**) พิสูจน์ว่า `resource_service` **compile และรันได้แม้ไม่มีไฟล์ private key อยู่ในเครื่อง
   ที่รันมันเลย** (เพราะมันไม่เคยต้องใช้) และพิสูจน์ว่า token ที่ `auth_service` ออกให้ ใช้กับ
   `resource_service` ได้จริงข้าม process (hint: ระวังเรื่อง path ของไฟล์ key — ใช้ environment variable
   แยกกันสองตัวชัดเจน เช่น `AUTH_PRIVATE_KEY_PATH` กับ `RESOURCE_PUBLIC_KEY_PATH`)

## สรุป

- **JWT ไม่ใช่การเข้ารหัส** — มันคือ `header.payload.signature` ที่ encode ด้วย Base64URL แล้วเซ็นด้วย
  cryptographic signature; payload **อ่านออกได้เสมอโดยไม่ต้องมี key** ห้ามใส่ข้อมูลลับลงไปเด็ดขาด — บทนี้
  พิสูจน์ด้วยการ decode JWT ด้วยมือจริง ไม่ต้องพึ่ง `jsonwebtoken::decode` เลยแม้แต่ครั้งเดียว
- **HS256** (symmetric, secret เดียว) เหมาะกับ monolith ที่มี backend เดียว — **RS256/ES256** (asymmetric,
  private key เซ็น/public key ตรวจ) เหมาะกับระบบที่ต้องแจกความสามารถ "ตรวจสอบ" ให้หลาย service โดยไม่ให้
  ปลอมสร้าง token ใหม่ได้ — ทั้งสองแบบ implement ด้วย crate `jsonwebtoken` จริง คอมไพล์และรันจริงแล้ว
- **ห้ามเก็บ/เทียบ password เป็น plaintext เด็ดขาด** — hash ด้วย `argon2` (Argon2id) ก่อนเก็บเสมอ ตรวจสอบด้วย
  `verify_password` ที่ parse params/salt กลับมาจาก PHC string เดียวกันกับที่เก็บไว้ — พิสูจน์ด้วย hash จริง
  จาก crate จริง
- **Custom extractor** (`CurrentUser` ผ่าน `FromRequestParts`) ต่อยอดจาก Part 64 ทำให้ handler ไม่ต้องรู้
  เรื่อง JWT เลยแม้แต่นิดเดียว — พิสูจน์ครบ 5 สถานการณ์ (ถูก/ไม่มี header/ผิดรูปแบบ/หมดอายุ/ถูกแก้ไข) ด้วย
  `curl` จริงและ error variant จริงจาก `jsonwebtoken`
- **Access token อายุสั้น + refresh token อายุยาว** จำกัดความเสียหายถ้า token หลุด แลกกับ UX ที่ไม่ต้อง
  login ใหม่บ่อย — การ revoke refresh token ก่อนหมดอายุต้องมี state ฝั่ง server (sketch ไว้รอ Part 83)
- **กับดักด้านความปลอดภัยของ JWT ร้ายแรงกว่าปกติ**: algorithm confusion (`alg: none`, key confusion),
  การปิด `validate_exp` เอง, การเก็บ token ฝั่ง client แบบไม่ปลอดภัย, secret ที่หลุดเข้า source control —
  ทุกข้อพิสูจน์ด้วยโค้ดและ error message จริง ไม่ใช่คำเตือนลอย ๆ

บทถัดไป **Part 75 (Authentication: Session-based และ OAuth2)** จะเทียบ JWT ที่บทนี้สอนกับแนวทาง
session-based auth (ที่ server เก็บ state และ revoke ได้ทันที ต่างจาก JWT ที่ stateless) และสอน OAuth2
flow สำหรับ "login ด้วย Google/GitHub" ที่ใช้กันทั่วไปในระบบจริง — จากนั้น **Part 76 (RBAC)** จะต่อยอด claim
`role` ที่บทนี้ออกแบบไว้ให้เป็นระบบสิทธิ์เต็มรูปแบบ, **Part 81 (Microservices)** จะขยายแนวคิด RS256/ES256 ที่
บทนี้แนะนำไว้ให้เป็นสถาปัตยกรรมจริง, และ **Part 83 (Redis)** จะเป็นที่ที่เราสร้างระบบ revocable refresh token
ที่บทนี้ sketch ไว้ให้สมบูรณ์

---

**Part ก่อนหน้า:** [SeaORM เบื้องต้น](part-073-seaorm.md) | **Part ถัดไป:** [Authentication: Session-based และ OAuth2](part-075-session-oauth2.md)
