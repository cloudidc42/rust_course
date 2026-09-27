# Part 80: gRPC ด้วย Tonic

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า **gRPC** คืออะไร ต่างจาก REST/JSON (Part 61-66) และ GraphQL (Part 79) อย่างไรในระดับกลไก ไม่ใช่แค่ "อีกวิธีเรียก API" — และบอกได้อย่างเจาะจงว่า**เมื่อไหร่**ควรเลือก gRPC จริง ๆ (คำตอบคือ: service-to-service ภายในระบบที่คุณควบคุมทั้งสองฝั่ง ไม่ใช่ API สาธารณะที่ client ควบคุมไม่ได้)
- เขียนไฟล์ **`.proto`** (Protocol Buffers) นิยาม message และ service ได้ถูกต้องตาม syntax proto3 และรู้จัก scalar type ของ Protobuf ทุกตัวพร้อม mapping เป็น Rust type ที่ `prost` generate ให้
- ตั้งค่า `build.rs` ให้ compile `.proto` เป็นโค้ด Rust ตอน build time ได้จริงด้วย `tonic-prost-build` (ไม่ใช่ `tonic-build` ตัวเก่าที่บทความ/tutorial จำนวนมากบนอินเทอร์เน็ตยังอ้างถึง) และอธิบายได้อย่างแม่นยำว่าการ generate โค้ดแบบนี้ **ไม่ใช่ proc macro** ตามความหมายที่ Part 44-45 สอนไว้ แม้จะให้ผลลัพธ์ที่คล้ายกัน
- implement gRPC server จริงด้วย `#[tonic::async_trait]` ที่ครอบคลุมการเรียกทั้ง **4 รูปแบบ**: unary, server streaming, client streaming, และ bidirectional streaming — รันได้จริง มี state จริง (in-memory store)
- เขียน gRPC client ด้วย Tonic ที่คุยกับ server ข้างต้นได้ทั้ง 4 รูปแบบ พร้อมอ่านและตีความ output จริงที่ได้จากการรัน
- จัดการ error ด้วย `tonic::Status` และ status code มาตรฐานของ gRPC ได้ถูกต้อง เทียบเคียงกับ HTTP status code จาก Part 61 ได้ พร้อมรู้จักการแนบ error details/metadata เพิ่มเติม
- เขียน **interceptor** สำหรับตรวจสอบสิทธิ์ (auth) ผ่าน gRPC metadata ได้ และอธิบายได้ว่า interceptor ของ Tonic สัมพันธ์กับ `tower` middleware อย่างไร (ข้อเท็จจริงเชิงสถาปัตยกรรมที่เชื่อมกับ Part 68)
- ประเมินได้อย่างตรงไปตรงมาว่า gRPC มีข้อจำกัดอะไรบ้าง (โดยเฉพาะเรื่อง browser) และวางตำแหน่งของ gRPC ในสถาปัตยกรรมระบบจริงเทียบกับ REST/GraphQL ได้ถูกต้อง เป็นการเตรียมพื้นฐานสำหรับ Part 81 (Microservices)

## ความรู้ที่ต้องมีมาก่อน

- **Part 57-58 (Serde เบื้องต้น/ขั้นสูง)**: บทนี้จะเปรียบเทียบ Protocol Buffers (binary serialization) กับ JSON (text serialization) ที่ `serde`/`serde_json` ใช้อยู่ตลอดเวลา คุณต้องเข้าใจแนวคิดพื้นฐานของ serialization/deserialization มาก่อนจึงจะเห็นความแตกต่างได้ชัด
- **Part 61 (HTTP Fundamentals)**: บทนี้อธิบายไว้ว่า HTTP/1.1 เป็น text protocol ที่ parse ทีละบรรทัด และแนะนำสั้น ๆ ว่า HTTP/2 มีอยู่ — gRPC**วิ่งอยู่บน HTTP/2 เท่านั้น** (ไม่รองรับ HTTP/1.1) เราจะอ้างอิงกลับไปที่ Part 61 ตลอดหัวข้อ 80.1 เพื่ออธิบายว่า multiplexing และ header compression ของ HTTP/2 คือเหตุผลเชิงกลไกที่ gRPC เร็วกว่า REST/1.1 แค่ไหนและทำไม อีกทั้งตารางเทียบ gRPC status code กับ HTTP status code ในหัวข้อ 80.7 จะอ้างตารางจาก Part 61 ตรง ๆ
- **Part 44-45 (Procedural Macros เบื้องต้น/ขั้นสูง)**: บทนี้จะเปรียบเทียบ**การ generate โค้ดจาก `.proto` ตอน build time**กับ**การ generate โค้ดด้วย proc macro ตอน compile time**ที่ Part 44-45 สอนไว้ — ทั้งสองแบบให้ความรู้สึกคล้ายกัน (เขียนสั้น ได้โค้ดยาวมาให้ใช้) แต่กลไกเบื้องหลัง**ต่างกันโดยสิ้นเชิง**ในระดับ compilation pipeline ซึ่งบทนี้จะอธิบายให้ชัดในหัวข้อ 80.3
- **Part 68 (Actix-web Middleware)**: บทนั้นสรุปไว้ว่า Axum และ Actix-web มี middleware คนละระบบเพราะ Axum สร้างบน `tower` ส่วน Actix-web มีระบบของตัวเอง — บทนี้จะเพิ่มข้อเท็จจริงที่น่าสนใจอีกชั้นคือ **Tonic ก็สร้างบน `tower` เหมือน Axum** (ทั้งคู่มาจากทีม Tokio) ทำให้ Axum กับ Tonic ใกล้กันทางสถาปัตยกรรมมากกว่า Axum กับ Actix-web
- **Part 70 (SQLx และ PostgreSQL)**: บทนี้จะเลือกใช้ in-memory store สำหรับตัวอย่างหลัก (มีเหตุผลอธิบายไว้ในหัวข้อ 80.4) แต่จะชี้ให้เห็นชัดว่าการสลับไปใช้ SQLx ทำได้โดยไม่กระทบ signature ของ trait ที่ generate มาจาก `.proto` เลยแม้แต่บรรทัดเดียว
- **Part 74 (JWT Authentication)**: หัวข้อ interceptor (80.8) จะใช้แนวคิด bearer token เดียวกับที่ Part 74 สอน เพียงแต่ส่งผ่าน gRPC metadata แทน HTTP `Authorization` header — เราจะอ้างอิงกลับไปที่ Part 74 สำหรับรายละเอียดการ decode/verify JWT แบบเต็มรูปแบบ
- **Part 77 (WebSocket)**: หัวข้อ server streaming (80.6.2) จะเทียบกับแนวคิด "server ส่งข้อมูลมาเรื่อย ๆ โดยไม่ต้องให้ client ขอใหม่ทุกครั้ง" ที่ Part 77 สอนผ่าน WebSocket — gRPC streaming ทำสิ่งเดียวกันได้แต่ด้วยกลไกที่ผูกกับ schema ที่ strict กว่ามาก
- **Part 79 (GraphQL ด้วย async-graphql)**: บทก่อนหน้านี้แนะนำ GraphQL ในฐานะ API ที่เน้น flexibility ให้ client เลือกข้อมูลเอง — บทนี้จะเป็นขั้วตรงข้ามที่เน้น schema strict และ performance สำหรับการสื่อสารภายในระบบ ตารางเทียบสามทางในหัวข้อ 80.10 จะสรุปจุดยืนของทั้งสามแนวทางนี้ให้ชัดเจน
- **Part 39-40 (Send/Sync, Threads, Shared State)**: ตัวอย่าง `BookStore` ในหัวข้อ 80.4 ใช้ `Mutex<HashMap<...>>` ที่แชร์ข้าม request/thread — ต้องเข้าใจพื้นฐานเรื่องนี้มาก่อน

## เนื้อหา

### 80.1 ทำไมต้องมี gRPC: Protocol Buffers บน HTTP/2 สำหรับ Service-to-Service

#### จุดยืนที่ต้องชัดเจนตั้งแต่บรรทัดแรก: gRPC ไม่ได้มาแทน REST

ก่อนจะลงรายละเอียดทางเทคนิคใด ๆ ต้องวางกรอบความคิดให้ถูกก่อน เพราะเป็นความเข้าใจผิดที่พบบ่อยที่สุด: **gRPC ไม่ได้ถูกออกแบบมาแข่งกับ REST API ที่ Part 61-66 สอน** ในความหมายที่ว่า "อันไหนดีกว่าให้ใช้อันนั้นทุกที่" — REST (และ GraphQL จาก Part 79) ถูกออกแบบมาให้เหมาะกับสถานการณ์ที่**คุณไม่ได้ควบคุม client**: มือถือ, เว็บเบราว์เซอร์, นักพัฒนาภายนอกที่เรียก public API ของคุณ, หรือระบบที่คุณไม่รู้ล่วงหน้าว่าใครจะมาเรียกบ้าง — สถานการณ์แบบนี้ต้องการ protocol ที่ **debug ง่าย** (เปิด browser dev tools อ่าน JSON ได้ทันที), **เข้าถึงง่าย** (ยิง `curl` เปล่า ๆ ก็ใช้ได้), และ**ทนต่อ client เวอร์ชันเก่า**ได้นาน

แต่มีอีกสถานการณ์หนึ่งที่ REST ไม่ใช่ตัวเลือกที่ดีที่สุด: **การสื่อสารระหว่าง service ภายในระบบเดียวกันที่คุณเขียนทั้งสองฝั่งเอง** — เช่น `order-service` เรียก `inventory-service` เพื่อเช็คสต๊อกก่อนยืนยันคำสั่งซื้อ, หรือ `notification-service` เรียก `user-service` เพื่อดึงอีเมลผู้ใช้ ในสถานการณ์นี้:

- คุณควบคุมทั้งสองฝั่งของการสื่อสาร → ไม่ต้องกังวลเรื่อง "client เวอร์ชันเก่าที่ยังไม่ได้อัปเดต" มากเท่า public API
- ความเร็ว (latency, throughput) สำคัญกว่า debuggability ด้วยตาเปล่า — เพราะ service เหล่านี้อาจถูกเรียกหลักพัน/หมื่นครั้งต่อวินาทีภายในระบบเดียวกัน ไม่ใช่ถูกเรียกครั้งเดียวโดยผู้ใช้ปลายทาง
- schema ที่ strict คือข้อดี ไม่ใช่ข้อเสีย — เพราะทั้งสองฝั่งพัฒนาไปด้วยกัน การมี contract ที่ compiler ตรวจสอบให้อัตโนมัติ (จาก `.proto`) ป้องกัน bug ประเภท "ลืมเปลี่ยน field name ให้ตรงกันทั้งสองฝั่ง" ได้ดีกว่า JSON ที่ไม่มี schema บังคับ

นี่คือเหตุผลที่บทนี้จะเน้นย้ำ**การสื่อสารระหว่าง service** เป็นหลัก และเป็นเหตุผลที่ Part 81 (Microservices Architecture) จะใช้ gRPC ที่เรียนในบทนี้เป็นเครื่องมือสื่อสารหลักระหว่าง service ต่าง ๆ ในระบบ — ส่วน REST ด้วย Axum ยังคงเป็นแนวทางหลักของหลักสูตรนี้สำหรับ API ที่ client ภายนอก (เว็บ/มือถือ) เรียกใช้

#### Protocol Buffers: Binary Serialization ที่ตรงข้ามกับ JSON

จาก **Part 57-58** คุณรู้จัก `serde`/`serde_json` ในฐานะเครื่องมือแปลง struct Rust เป็น **JSON** (text format ที่มนุษย์อ่านได้) และกลับกัน — **Protocol Buffers** (หรือ "Protobuf") ทำสิ่งเดียวกันในเชิงหน้าที่ (แปลง structured data เป็น bytes เพื่อส่งผ่านเครือข่าย) แต่เลือกวิธีที่**ตรงข้ามกันโดยสิ้นเชิง**: มันเข้ารหัสข้อมูลเป็น**binary format ที่แน่นที่สุดเท่าที่จะทำได้** ไม่ใช่ text

ตัวอย่างให้เห็นภาพแบบวัดจริง ไม่ใช่ประมาณเอาเอง: เขียนโปรแกรมทดลองเล็ก ๆ เอา `Book { id: 1, title: "Rust", author: "The Rust Team", year: 2024, price: 39.99 }` ตัวเดียวกันไป encode ด้วยทั้ง `serde_json::to_vec` และ `prost::Message::encode` แล้ววัดขนาดจริง:

```rust
use grpc_demo::book::Book;
use prost::Message;

let book = Book {
    id: 1,
    title: "Rust".to_string(),
    author: "The Rust Team".to_string(),
    year: 2024,
    price: 39.99,
};

let json_bytes = serde_json::to_vec(&book_as_json_struct).unwrap();

let mut proto_bytes = Vec::new();
book.encode(&mut proto_bytes).unwrap();

println!("JSON  size: {} bytes", json_bytes.len());
println!("Proto size: {} bytes", proto_bytes.len());
```

**ผลลัพธ์จริงจากการรัน:**

```text
JSON:     {"id":1,"title":"Rust","author":"The Rust Team","year":2024,"price":39.99}
JSON  size: 74 bytes
Proto size: 35 bytes
Proto bytes (hex): 08 01 12 04 52 75 73 74 1a 0d 54 68 65 20 52 75 73 74 20 54 65 61 6d 20 e8 0f 29 1f 85 eb 51 b8 fe 43 40
ประหยัดลง: 52.7%
```

ข้อมูลชุดเดียวกันเป๊ะ ๆ ได้ขนาด**เล็กลงกว่าครึ่ง**เมื่อเปลี่ยนจาก JSON เป็น Protobuf — key อย่าง `"id"`, `"title"`, `"author"` ถูกส่งไปเป็น**ข้อความจริง**ทุกครั้งในฝั่ง JSON แม้ว่าทั้งสองฝั่งจะรู้ schema นี้ล่วงหน้าอยู่แล้วก็ตาม (เพราะ JSON ไม่มีแนวคิดของ "schema ที่ทั้งสองฝั่งตกลงกันไว้ล่วงหน้า" ในตัวโปรโตคอลเอง — มันพึ่งพา `struct` + `#[derive(Deserialize)]` ที่ฝั่ง Rust เขียนเองหลังบ้าน) ในขณะที่ Protobuf ไม่ส่งชื่อ field ไปด้วยเลย

#### ถอดรหัส Bytes จริงทีละไบต์: Wire Format ของ Protobuf ทำงานอย่างไร

เพื่อไม่ให้ "35 bytes" เป็นแค่ตัวเลขลอย ๆ มาถอดรหัส hex string ข้างบนทีละกลุ่มให้เห็นว่า Protobuf เข้ารหัสอย่างไรจริง ๆ — หลักการคือทุก field เริ่มด้วย **tag byte** ที่คำนวณจาก `(field_number << 3) | wire_type` แล้วตามด้วยค่าตาม wire type นั้น:

| Wire type | เลข | ใช้กับ Protobuf type ไหน |
|---|---|---|
| Varint | 0 | `int32/64`, `uint32/64`, `sint32/64`, `bool`, `enum` |
| 64-bit | 1 | `fixed64`, `sfixed64`, `double` |
| Length-delimited | 2 | `string`, `bytes`, message ที่ฝังอยู่ข้างใน, `repeated` แบบ packed |
| 32-bit | 5 | `fixed32`, `sfixed32`, `float` |

ถอดรหัส hex ของ `Book` ข้างบนตามตารางนี้:

| Bytes | Tag คำนวณจาก | หมายถึง |
|---|---|---|
| `08 01` | `08` = `(1<<3)\|0` → field 1, varint | `id = 1` |
| `12 04 52 75 73 74` | `12` = `(2<<3)\|2` → field 2, length-delimited ยาว 4 ไบต์ | `title = "Rust"` (ASCII `52 75 73 74`) |
| `1a 0d 54 68 65 ...` | `1a` = `(3<<3)\|2` → field 3, length-delimited ยาว 13 ไบต์ | `author = "The Rust Team"` |
| `20 e8 0f` | `20` = `(4<<3)\|0` → field 4, varint (สองไบต์เพราะ 2024 เกิน 127) | `year = 2024` |
| `29 1f 85 eb 51 b8 fe 43 40` | `29` = `(5<<3)\|1` → field 5, 64-bit fixed (double) | `price = 39.99` (IEEE754 double, little-endian) |

สังเกตว่า**ไม่มีไบต์ไหนเลยที่เป็นชื่อ field** — ผู้ถอดรหัสต้องมี `.proto` schema (ที่บอกว่า field 1 คือ `id`, field 2 คือ `title` ฯลฯ) อยู่ก่อนแล้วจึงจะรู้ความหมาย ตรงตามที่อธิบายไว้ในหัวข้อก่อนหน้าทุกประการ — นี่คือเหตุผลเชิงกลไกตัวจริงที่ทำให้ Protobuf เล็กกว่า JSON และเป็นเหตุผลเดียวกันที่ทำให้อ่านด้วยตาเปล่าไม่ได้โดยไม่มี schema

**Tradeoff ที่ต้องเข้าใจให้ครบทั้งสองด้าน (นี่คือประเด็นสำคัญที่สุดของหัวข้อนี้):**

| ด้าน | JSON (Part 57-58) | Protocol Buffers |
|---|---|---|
| ขนาดข้อมูลบน wire | ใหญ่กว่า (มี field name + syntax character ซ้ำ ๆ) | เล็กกว่ามาก (binary, ไม่มี field name) |
| ความเร็วในการ (de)serialize | ช้ากว่า (ต้อง parse text, แปลง string เป็นตัวเลข ฯลฯ) | เร็วกว่ามาก (อ่าน byte ตรง ๆ ตาม wire format ที่รู้ล่วงหน้า) |
| อ่านด้วยตาเปล่า (human-readable) | **ได้** — เปิดด้วย text editor หรือ `curl` เปล่า ๆ ก็อ่านออก | **ไม่ได้** — ต้องมี `.proto` schema และเครื่องมือถอดรหัส (เช่น `grpcurl`, `protoc --decode`) จึงจะอ่านออก |
| ความเข้ากันได้ข้าม client ที่ไม่รู้จักกันมาก่อน | ดีมาก — client ไหนก็ parse JSON ได้ ไม่ต้องมี schema ล่วงหน้า | ต้องมี `.proto` ไฟล์เดียวกัน (หรือใช้ reflection — หัวข้อ 80.9) ทั้งสองฝั่งจึงจะคุยกันได้ |
| Schema evolution (เพิ่ม/ลบ field ทีหลัง) | ทำได้แต่ไม่มีการันตีจาก protocol เอง (ต้องระวังเอง) | Protobuf ออกแบบมาให้ backward/forward compatible โดยธรรมชาติ (field ที่ไม่รู้จักจะถูกข้าม ไม่ error) ถ้าทำตามกฎ (ไม่เปลี่ยน/ใช้ tag number ซ้ำ) |

สรุปเป็นหลักการเดียว: **binary format ชนะเรื่อง performance แต่แลกมาด้วย debuggability** — นี่คือเหตุผลเชิงลึกว่าทำไม Protobuf เหมาะกับ service-to-service (ที่ performance สำคัญ และทั้งสองฝั่งมี `.proto` เดียวกันอยู่แล้วในโค้ด ไม่ต้องอ่าน wire format ด้วยตาเปล่าบ่อย ๆ) แต่ไม่เหมาะกับ public API ที่นักพัฒนาภายนอกต้อง debug ด้วยตัวเองบ่อย ๆ

#### HTTP/2: ทวนจาก Part 61 และเหตุผลที่ gRPC เลือกมันเป็น transport

**Part 61** แนะนำ HTTP/1.1 ไว้อย่างละเอียดในฐานะ text protocol ที่ parse ทีละบรรทัด (request line → headers → บรรทัดว่าง → body) และเอ่ยถึง HTTP/2 สั้น ๆ ว่ามีอยู่ — gRPC เลือกใช้ **HTTP/2 เป็น transport เพียงแบบเดียว** (ไม่มีโหมด fallback ไป HTTP/1.1) เพราะ HTTP/2 มีคุณสมบัติสองอย่างที่ REST บน HTTP/1.1 ทั่วไปไม่ได้ใช้ประโยชน์เต็มที่:

1. **Multiplexing**: HTTP/1.1 (แม้เปิด keep-alive) ยังมีข้อจำกัดคือ**หนึ่ง TCP connection รับได้ทีละ request-response คู่หนึ่งในเวลาเดียวกัน** (head-of-line blocking ที่ระดับ HTTP แม้ TCP connection เดียวกันจะถูก reuse ก็ตาม) เบราว์เซอร์แก้ปัญหานี้ด้วยการเปิดหลาย connection พร้อมกัน (มักจำกัดที่ 6 connection ต่อ host) — HTTP/2 แก้ปัญหานี้ที่ราก: **หลาย request/response วิ่งสลับกันบน TCP connection เดียวกันได้พร้อมกันจริง ๆ** ผ่านแนวคิด "stream" ที่มี ID ของตัวเอง ทำให้ service ที่ยิง RPC หลายตัวพร้อมกันไปยัง service ปลายทางเดียวกันไม่ต้องเปิด connection ใหม่ทุกครั้ง หรือรอ RPC ก่อนหน้าให้จบก่อน
2. **Header compression (HPACK)**: HTTP/1.1 ส่ง header เป็น text ซ้ำ ๆ ทุก request (เช่น `User-Agent`, `Authorization` ก็ถูกส่งซ้ำทุกครั้ง) — HTTP/2 บีบอัด header ด้วย HPACK ซึ่งจดจำ header ที่เคยส่งไปแล้วและส่งแค่ "reference" กลับมาถ้าเหมือนเดิม ประหยัด bandwidth ได้มากเมื่อมีการเรียกซ้ำ ๆ ถี่ ๆ (ตรงกับลักษณะการเรียกระหว่าง service ภายในระบบพอดี)

ผลลัพธ์เชิงปฏิบัติที่สำคัญมากซึ่งเป็นกับดักที่พบบ่อย (ดูหัวข้อกับดักข้อ 4 ท้ายบท): **เพราะ gRPC ผูกกับ HTTP/2 อย่างเข้มงวด คุณจะยิง `curl` ธรรมดา (HTTP/1.1) เข้าไปที่ gRPC server ไม่ได้เลย** — เราจะพิสูจน์ข้อเท็จจริงนี้ด้วยการรันจริงในหัวข้อ 80.7 และ 80.9

#### พิสูจน์ Multiplexing จริงด้วยการวัดเวลา ไม่ใช่แค่คำอธิบายทฤษฎี

คำอธิบายเรื่อง multiplexing ข้างบนพิสูจน์ได้จริงด้วยการวัดเวลา: ใช้ RPC `ListBooks` (server streaming ที่ server จำลอง latency 150ms ต่อเล่มตามที่หัวข้อ 80.6.2 จะอธิบาย) ยิงแบบ **sequential** (ทีละครั้ง รอให้จบก่อนยิงครั้งถัดไป) เทียบกับยิงแบบ **concurrent** (พร้อมกัน 3 ครั้งบน `Channel` เดียวกัน ด้วย `tokio::join!`) — ถ้า multiplexing ทำงานจริงตามที่อธิบายไว้ เวลารวมของฝั่ง concurrent ควรใกล้เคียงกับการยิงครั้งเดียว **ไม่ใช่ 3 เท่า**:

```rust
use std::time::Instant;

async fn drain_list_books(client: &mut BookServiceClient<Channel>, label: &str) -> usize {
    let mut stream = client
        .list_books(with_auth(ListBooksRequest { page_size: 1 }))
        .await.unwrap().into_inner();
    let mut count = 0;
    while stream.message().await.unwrap().is_some() {
        count += 1;
    }
    count
}

// Sequential: ยิงทีละครั้ง
let start = Instant::now();
drain_list_books(&mut client, "seq-1").await;
drain_list_books(&mut client, "seq-2").await;
drain_list_books(&mut client, "seq-3").await;
println!("Sequential รวมเวลา: {:?}", start.elapsed());

// Concurrent: ยิงพร้อมกันบน channel เดียวกัน (คนละ client handle แต่ใช้ Channel เดียวกัน)
let start = Instant::now();
tokio::join!(
    drain_list_books(&mut c1, "conc-1"),
    drain_list_books(&mut c2, "conc-2"),
    drain_list_books(&mut c3, "conc-3"),
);
println!("Concurrent รวมเวลา: {:?}", start.elapsed());
```

**ผลลัพธ์จริงจากการรัน** (3 เล่มในระบบ, 150ms ต่อเล่ม → คาดว่า sequential ≈ 3×3×150ms = 1,350ms, concurrent ≈ 3×150ms = 450ms ถ้า multiplex ได้จริง):

```text
  [seq-1] ดึงครบ 3 เล่ม
  [seq-2] ดึงครบ 3 เล่ม
  [seq-3] ดึงครบ 3 เล่ม
Sequential (3 ครั้งทีละอัน) รวมเวลา: 1.361816661s
  [conc-3] ดึงครบ 3 เล่ม
  [conc-1] ดึงครบ 3 เล่ม
  [conc-2] ดึงครบ 3 เล่ม
Concurrent (3 ครั้งพร้อมกัน บน connection เดียวกัน) รวมเวลา: 455.253942ms
```

ตัวเลขจริงตรงกับที่คาดการณ์ไว้เป๊ะ: **1.36 วินาทีเทียบกับ 455 มิลลิวินาที** — เร็วขึ้นประมาณ 3 เท่าพอดีกับจำนวน RPC ที่ยิงพร้อมกัน เพราะทั้ง 3 การเรียกวิ่งอยู่บน**TCP connection เดียวกัน**ผ่านคนละ HTTP/2 stream ID พร้อมกันจริง ๆ (สังเกตด้วยว่าลำดับที่ log ออกมาไม่เรียงกัน `conc-3`, `conc-1`, `conc-2` — เป็นอีกหลักฐานว่าทั้งสามงานสลับกันทำงานคู่กันจริง ไม่ได้ทำเสร็จเรียงลำดับที่ยิงออกไป) นี่คือประโยชน์เชิงปฏิบัติของ HTTP/2 multiplexing ที่ REST บน HTTP/1.1 ทั่วไปไม่ได้มาให้แบบนี้โดยอัตโนมัติ

### 80.2 Protocol Buffers และไฟล์ `.proto`: นิยาม Message และ Service

#### Scalar Types ของ Protobuf และการ Mapping เป็น Rust

ก่อนเขียนไฟล์ `.proto` จริง ต้องรู้จัก scalar type ทั้งหมดของ proto3 ก่อน เพราะ `prost` (library ที่ tonic ใช้แปลง Protobuf เป็น Rust) จะ map แต่ละ type เข้ากับ Rust type ที่เจาะจงตามตารางนี้:

| Protobuf type | ขนาด/ลักษณะ | Rust type ที่ `prost` generate ให้ |
|---|---|---|
| `double` | floating point 64-bit | `f64` |
| `float` | floating point 32-bit | `f32` |
| `int32` | signed integer, wire encoding แบบ variable-length (ไม่เหมาะกับเลขลบมาก ๆ) | `i32` |
| `int64` | signed integer 64-bit, variable-length | `i64` |
| `uint32` / `uint64` | unsigned integer, variable-length | `u32` / `u64` |
| `sint32` / `sint64` | signed integer เข้ารหัสแบบ zigzag (เหมาะกับเลขลบมากกว่า `int32`/`int64`) | `i32` / `i64` |
| `fixed32` / `fixed64` | ขนาดคงที่เสมอ (ไม่ variable-length) เร็วกว่าถ้าค่ามักจะใหญ่ | `u32` / `u64` |
| `sfixed32` / `sfixed64` | เหมือน fixed แต่เป็น signed | `i32` / `i64` |
| `bool` | true/false | `bool` |
| `string` | UTF-8 text | `String` |
| `bytes` | raw binary data | `prost::bytes::Bytes` (หรือ `Vec<u8>` ได้ถ้าตั้งค่า) |
| `repeated T` | array/list ของ type `T` | `Vec<T>` |
| `map<K, V>` | key-value map | `HashMap<K, V>` (หรือ `BTreeMap` ถ้าตั้งค่า) |
| `message` (นิยามเอง) | struct ที่ประกอบจาก field อื่น ๆ | `struct` ที่ generate ให้ตรงตามชื่อ message |
| `enum` (นิยามเอง) | ค่าคงที่จำนวนเต็มมีชื่อ | Rust `enum` (เป็น `i32` ภายใน ตาม proto3 spec) |

**ข้อสังเกตสำคัญ**: proto3 (เวอร์ชันที่ใช้กันเป็นมาตรฐานในปัจจุบัน และเป็นเวอร์ชันที่บทนี้ใช้ทั้งหมด) **ไม่มีแนวคิด `required`/`optional` แบบ proto2 เดิม** — ทุก field ที่ไม่ใส่ค่าจะได้ค่า default ของ type นั้นเสมอ (`0` สำหรับตัวเลข, `""` สำหรับ string, `false` สำหรับ bool) แทนที่จะเป็น `null`/absent สังเกตว่านี่ต่างจาก JSON+serde ที่ Part 57 สอนไว้ว่า field ที่ไม่มีค่ามักแทนด้วย `Option<T>` — ถ้าต้องการความหมาย "ไม่มีค่าจริง ๆ" (ไม่ใช่ "ค่าเป็น 0") ใน proto3 ต้องใช้ `optional` keyword ที่เพิ่มกลับมาใน proto3 เวอร์ชันใหม่ ๆ (ซึ่งจะ generate เป็น `Option<T>` ใน Rust) — บทนี้จะไม่ลงรายละเอียดจุดนี้เพิ่มเพราะโดเมนตัวอย่างของเราไม่จำเป็นต้องใช้

#### เขียนไฟล์ `book.proto` จริง: BookService สำหรับระบบห้องสมุด

เราจะใช้โดเมน**ระบบห้องสมุด (library)** เป็นตัวอย่างหลักตลอดบทนี้ — มี `BookService` ที่รองรับ RPC 5 ตัว ครอบคลุมการเรียกทั้ง 4 รูปแบบที่ gRPC มี (หัวข้อ 80.6 จะอธิบายแต่ละรูปแบบละเอียด):

```protobuf
syntax = "proto3";

package library.v1;

service BookService {
  // unary: หนึ่ง request หนึ่ง response
  rpc GetBook(GetBookRequest) returns (Book);
  rpc CreateBook(CreateBookRequest) returns (Book);

  // server streaming: หนึ่ง request, server ส่ง response กลับมาหลายชิ้น
  rpc ListBooks(ListBooksRequest) returns (stream Book);

  // client streaming: client ส่ง request หลายชิ้น, server ตอบกลับหนึ่งเดียว
  rpc UploadBooks(stream CreateBookRequest) returns (UploadSummary);

  // bidirectional streaming: ทั้งสองฝั่งส่ง stream พร้อมกัน อิสระจากกัน
  rpc WatchAvailability(stream AvailabilityUpdate) returns (stream AvailabilityUpdate);
}

message Book {
  int64 id = 1;
  string title = 2;
  string author = 3;
  int32 year = 4;
  double price = 5;
}

message GetBookRequest {
  int64 id = 1;
}

message CreateBookRequest {
  string title = 1;
  string author = 2;
  int32 year = 3;
  double price = 4;
}

message ListBooksRequest {
  int32 page_size = 1;
}

message UploadSummary {
  int32 total_received = 1;
  int32 total_created = 2;
  repeated string errors = 3;
}

message AvailabilityUpdate {
  int64 book_id = 1;
  int32 copies_available = 2;
}
```

ไฟล์นี้ถูกบันทึกไว้ที่ `proto/book.proto` ในโปรเจกต์ (ตามธรรมเนียมที่ใช้กันแพร่หลาย — เก็บไฟล์ `.proto` ไว้ใน directory `proto/` แยกจาก `src/`) มาดูทีละส่วนว่าแต่ละบรรทัดหมายถึงอะไร:

- **`syntax = "proto3";`**: บรรทัดแรกของทุกไฟล์ `.proto` บอก compiler ว่าใช้ syntax เวอร์ชันไหน (proto3 คือมาตรฐานปัจจุบัน — proto2 เป็นเวอร์ชันเก่าที่ยังพบใน codebase เก่า ๆ แต่ไม่แนะนำให้เริ่มโปรเจกต์ใหม่ด้วย proto2 แล้ว)
- **`package library.v1;`**: namespace ของ message/service ทั้งหมดในไฟล์นี้ — คล้ายกับ Rust module path แนวคิดการใส่เลขเวอร์ชันไว้ใน package name (`v1`) เป็นธรรมเนียมที่ช่วยจัดการ schema evolution ระยะยาว (ถ้าต้อง breaking change แบบใหญ่ ค่อยขึ้น `v2` แยกไปเลย)
- **`service BookService { ... }`**: นิยาม RPC ทั้งหมดที่ service นี้มี — สังเกต keyword `stream` หน้า type parameter ว่าคือจุดที่กำหนดว่า RPC ตัวนั้นเป็นแบบ streaming ฝั่งไหน (ไม่มี `stream` เลย = unary, มีแค่ request = client streaming, มีแค่ response = server streaming, มีทั้งคู่ = bidirectional)
- **field number (`= 1`, `= 2`, ...)**: **นี่คือส่วนที่สำคัญที่สุดและพลาดบ่อยที่สุดสำหรับคนมาจาก JSON** — เลขเหล่านี้ไม่ใช่แค่ลำดับสวย ๆ แต่คือ **wire tag** ที่ใช้จริงตอนเข้ารหัสข้อมูล (ตามที่อธิบายไว้ในหัวข้อ 80.1) เปลี่ยนเลขนี้ทีหลัง = breaking change ทันที (client เก่าจะอ่านค่าผิด field) ในขณะที่เปลี่ยนชื่อ field (`title` → `book_title`) ไม่กระทบ wire format เลยเพราะชื่อไม่ได้ถูกส่งไปด้วย — นี่คือความต่างที่สำคัญมากจาก JSON ที่ **ชื่อ key คือสิ่งที่มีผลจริง ไม่ใช่ลำดับ**

#### พิสูจน์ Schema Evolution จริง: เพิ่ม Field ใหม่โดยไม่พัง Client เก่า

ตารางเปรียบเทียบในหัวข้อ 80.1 อ้างว่า Protobuf ออกแบบมาให้ backward/forward compatible โดยธรรมชาติ — มาพิสูจน์ข้ออ้างนี้ด้วยโค้ดจริง จำลองสถานการณ์ **rolling deployment** ที่พบได้ทั่วไปในระบบ microservices (ซึ่ง Part 81 จะพูดถึงเพิ่ม): ระหว่าง deploy service เวอร์ชันใหม่ มักมี "เวอร์ชันเก่า" และ "เวอร์ชันใหม่" รันคู่กันชั่วคราว (rolling update ทีละ instance) — สร้าง message สองรุ่นเทียบกัน คือ `BookV1` (มีแค่ `id`, `title`) และ `BookV2` (มี `id`, `title`, และ `isbn` field ใหม่ที่ tag `= 3`):

```rust
#[derive(Clone, PartialEq, ::prost::Message)]
struct BookV1 {
    #[prost(int64, tag = "1")]
    id: i64,
    #[prost(string, tag = "2")]
    title: String,
}

#[derive(Clone, PartialEq, ::prost::Message)]
struct BookV2 {
    #[prost(int64, tag = "1")]
    id: i64,
    #[prost(string, tag = "2")]
    title: String,
    #[prost(string, tag = "3")]
    isbn: String, // field ใหม่ที่ v1 ไม่รู้จัก
}
```

ทดสอบสองทิศทาง: (1) client เก่า (v1) ส่งไปยัง server ใหม่ (v2) — server ใหม่ควรอ่านได้ปกติ ได้ `isbn` เป็นค่า default และ (2) client ใหม่ (v2) ส่งไปยัง server เก่า (v1) ที่ยังไม่รู้จัก field `isbn` — server เก่าควรอ่านได้ปกติ แค่ "มองไม่เห็น" field ที่ไม่รู้จัก:

```rust
// v1 -> v2 (backward compatibility)
let old_client_msg = BookV1 { id: 1, title: "Rust".to_string() };
let mut buf = Vec::new();
old_client_msg.encode(&mut buf).unwrap();
let decoded_by_new_server = BookV2::decode(&buf[..]).unwrap();

// v2 -> v1 (forward compatibility)
let new_client_msg = BookV2 { id: 2, title: "Programming Rust".to_string(), isbn: "978-1492052593".to_string() };
let mut buf2 = Vec::new();
new_client_msg.encode(&mut buf2).unwrap();
let decoded_by_old_server = BookV1::decode(&buf2[..]).unwrap();
```

**ผลลัพธ์จริงจากการรัน (ไม่มี error เกิดขึ้นเลยแม้แต่ทิศทางเดียว):**

```text
[backward compat] v1 bytes decoded เป็น v2: BookV2 { id: 1, title: "Rust", isbn: "" }
[forward compat]  v2 bytes decoded เป็น v1: BookV1 { id: 2, title: "Programming Rust" }
  (isbn ที่ v2 ส่งมาถูกข้ามไปเงียบ ๆ เพราะ v1 struct ไม่มี field นี้เลย)

สรุป: ทั้งสองทิศทางไม่ error แม้ schema ไม่ตรงกันเป๊ะ — ตราบใดที่ tag number เดิมไม่ถูกเปลี่ยน/ใช้ซ้ำ
```

นี่คือคุณสมบัติที่ทำให้ Protobuf เหมาะกับระบบ microservices ที่ deploy service หลายตัวแยกจากกัน (Part 81): **เพิ่ม field ใหม่ได้อย่างปลอดภัยโดยไม่ต้อง deploy ทุก service ที่เกี่ยวข้องพร้อมกันในเวลาเดียวกันเป๊ะ** — ตราบใดที่ทำตามกฎง่าย ๆ สองข้อ: (1) ห้ามเปลี่ยนความหมายหรือ type ของ tag number ที่มีอยู่แล้ว และ (2) ห้ามนำ tag number ที่เคยถูกลบไปแล้วกลับมาใช้ใหม่กับความหมายที่ต่างออกไป (ถ้าต้อง "เลิกใช้" field ควรใช้ keyword `reserved` ใน `.proto` เพื่อบอก `protoc` ว่าห้ามใครเผลอใช้ tag number นั้นซ้ำในอนาคต)

### 80.3 ติดตั้งและ Setup: `tonic`, `prost`, และ Build-Time Code Generation

#### เวอร์ชันที่ใช้จริงในบทนี้ (ตรวจสอบแล้วจาก crates.io ณ ปัจจุบัน)

Tonic เป็น ecosystem ที่มีการเปลี่ยนโครงสร้าง crate ค่อนข้างมากในช่วงหลัง ถ้าคุณค้นหา tutorial เก่า ๆ บนอินเทอร์เน็ตจะเจอโค้ดที่ใช้ API คนละแบบกับที่ใช้งานได้จริงในเวอร์ชันปัจจุบัน — บทนี้ตรวจสอบเวอร์ชันจริงด้วยการรัน `cargo add` ในโปรเจกต์ทดลองจริง ได้ผลดังนี้:

```toml
[dependencies]
prost = "0.14.4"
tokio = { version = "1.53.1", features = ["full"] }
tonic = "0.14.6"
tonic-prost = "0.14.6"
tonic-reflection = "0.14.6"
futures = "0.3"
tokio-stream = "0.1"
async-stream = "0.3.6"
bytes = "1.12.1"

[build-dependencies]
tonic-prost-build = "0.14.6"
```

**การเปลี่ยนแปลงที่สำคัญที่สุดที่ต้องรู้ (และเป็นสาเหตุที่ tutorial เก่าใช้ไม่ได้)**: ก่อนหน้านี้ (Tonic เวอร์ชัน 0.12 และเก่ากว่า) โค้ดในบทความส่วนใหญ่จะเขียนแค่:

```toml
# แบบเก่า (tonic <= 0.12) — ใช้ไม่ได้แล้วกับเวอร์ชันปัจจุบัน
[build-dependencies]
tonic-build = "0.12"
```

แล้วเรียก `tonic_build::compile_protos(...)` ตรง ๆ ใน `build.rs` — แต่ตั้งแต่ Tonic 0.13 เป็นต้นมา **ทีม Tonic แยกความรับผิดชอบของ `tonic-build` ออกเป็นสองส่วน**: `tonic-build` ตอนนี้เป็นแค่ **codegen abstraction ระดับต่ำ** ที่ไม่ผูกกับ Protobuf โดยเฉพาะอีกต่อไป (เผื่อในอนาคตมีคนอยากใช้ serialization format อื่นแทน Protobuf) ส่วนโค้ดที่คุย กับ `protoc`/`prost-build` จริง ๆ ถูกย้ายไปอยู่ที่ crate ใหม่ชื่อ **`tonic-prost-build`** — นี่คือคำอธิบายตรงจาก README ของ `tonic-build` เวอร์ชัน 0.14.6 ที่ตรวจสอบจริงจาก source:

> "Provides code generation for service stubs to use with tonic. For protobuf compilation via prost, use the `tonic-prost-build` crate instead."

ในทางเดียวกัน ที่ runtime (ไม่ใช่ build time) ก็มีการแยกแบบเดียวกัน: `tonic` เวอร์ชันใหม่ไม่ผูก codec ของ Protobuf ไว้ในตัวเองอีกต่อไป แต่ย้าย `ProstCodec` ไปอยู่ที่ crate **`tonic-prost`** ซึ่งต้องเพิ่มเป็น dependency ตรง ๆ ด้วย (โค้ดที่ generate มาจะ `use tonic_prost::ProstCodec` เอง ไม่ต้อง import อะไรเพิ่มในโค้ดที่คุณเขียน แต่ crate นี้ต้องอยู่ใน `Cargo.toml`)

สรุปเป็นกฎที่จำง่าย: **build-time ใช้ `tonic-prost-build`, runtime ใช้ `tonic` + `tonic-prost` + `prost`** — ถ้าเจอ tutorial ที่บอกให้ใช้แค่ `tonic-build` เป็น build-dependency เดี่ยว ๆ แล้วเรียก `tonic_build::compile_protos` ตรง ๆ นั่นคือโค้ดสำหรับเวอร์ชันเก่า จะ compile ไม่ผ่านกับ dependency ปัจจุบัน (ดูข้อพิสูจน์จริงในกับดักข้อ 3 ท้ายบท)

#### ต้องมี `protoc` ติดตั้งในเครื่อง (หรือ CI) ด้วย

`tonic-prost-build` ไม่ได้ parse ไฟล์ `.proto` ด้วยตัวเอง แต่เรียกใช้ **`protoc`** (Protocol Buffers Compiler ตัวจริงจาก Google ที่เขียนด้วย C++) เป็น external process เพื่อแปลง `.proto` เป็น `FileDescriptorSet` แล้ว `prost-build` ค่อยแปลง descriptor นั้นเป็นโค้ด Rust อีกที — นี่หมายความว่า **เครื่องที่ build โปรเจกต์นี้ (รวม CI) ต้องมี `protoc` อยู่ใน `PATH` หรือกำหนด environment variable `PROTOC` ชี้ไปที่ตำแหน่งของมัน** ไม่เช่นนั้น build script จะ fail ทันที (ดูข้อพิสูจน์จริงพร้อม error message เป๊ะ ๆ ในกับดักข้อ 1)

ในสภาพแวดล้อมที่ใช้เขียนบทนี้ ตรวจสอบแล้วว่า**ไม่มี `protoc` ติดตั้งมาให้ตั้งแต่ต้น** ต้องติดตั้งเพิ่มด้วย (บน Debian/Ubuntu):

```bash
sudo apt-get install -y protobuf-compiler
```

หลังติดตั้งแล้วตรวจสอบเวอร์ชันได้ด้วย `protoc --version` (ในเครื่องที่ใช้ทดสอบบทนี้ได้ `libprotoc 3.21.12`) โปรเจกต์จริงบางทีมเลือกใช้ crate อย่าง `protobuf-src` ที่ vendor ตัว `protoc` มาให้เลยเพื่อไม่ต้องพึ่งการติดตั้งแยกในทุกเครื่อง/CI แต่แนวทางนั้นเพิ่มเวลา compile ครั้งแรกขึ้นมากพอสมควร (ต้อง compile `protoc` จาก C++ source) บทนี้เลือกสาธิตด้วยการติดตั้ง `protoc` ตรง ๆ เพราะเรียบง่ายกว่าและตรงกับ setup ส่วนใหญ่ที่พบในทีมจริง

#### `build.rs`: จุดที่ `.proto` ถูกแปลงเป็น Rust ตอน Build Time

```rust
// build.rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let out_dir = std::path::PathBuf::from(std::env::var("OUT_DIR")?);

    tonic_prost_build::configure()
        .build_server(true)
        .build_client(true)
        .file_descriptor_set_path(out_dir.join("book_descriptor.bin"))
        .compile_protos(&["proto/book.proto"], &["proto"])?;
    Ok(())
}
```

อธิบายทีละส่วน:

- **`OUT_DIR`**: environment variable ที่ Cargo กำหนดให้ทุก build script โดยอัตโนมัติ ชี้ไปยัง directory ชั่วคราวใต้ `target/` ที่สงวนไว้สำหรับไฟล์ที่ build script สร้างขึ้น (คนละที่กับ `src/` ของคุณเสมอ) — โค้ดที่ generate จาก `.proto` จะถูกวางไว้ที่นี่ ไม่ใช่ใน `src/` ตรง ๆ (ต่างจาก proc macro ใน Part 44-45 ที่ generate โค้ดแบบ "แทรกเข้าไปในตำแหน่งเดิม" ตอน compile — ดูหัวข้อถัดไปสำหรับความต่างที่ลึกกว่านี้)
- **`.build_server(true)` / `.build_client(true)`**: บอกให้ generate ทั้ง server-side trait (`BookService` trait + `BookServiceServer` wrapper) และ client-side struct (`BookServiceClient`) — ปิดตัวใดตัวหนึ่งได้ถ้าโปรเจกต์นั้นทำหน้าที่แค่ client หรือแค่ server (ลดขนาดโค้ดที่ generate ลงได้)
- **`.file_descriptor_set_path(...)`**: บอกให้เขียน binary `FileDescriptorSet` ของ schema ทั้งหมดลงไฟล์ไว้ด้วย — ใช้สำหรับ **server reflection** ในหัวข้อ 80.9 (ถ้าไม่ต้องการ reflection ก็ตัดบรรทัดนี้ออกได้)
- **`.compile_protos(&["proto/book.proto"], &["proto"])`**: อาร์กิวเมนต์แรกคือรายชื่อไฟล์ `.proto` ที่จะ compile (รับได้หลายไฟล์) อาร์กิวเมนต์ที่สองคือ include path — directory ที่ `protoc` จะค้นหาไฟล์ที่ถูก `import` จากไฟล์อื่น (โปรเจกต์เรามีไฟล์เดียวไม่มี import แต่ต้องระบุไว้เสมอเพื่อให้ `protoc` หาไฟล์ `proto/book.proto` เจอ)

#### รับ Rust Module จากไฟล์ที่ Generate: `include_proto!`

หลังจาก `build.rs` compile สำเร็จ โค้ด Rust ที่ generate ได้จะถูกวางไว้ที่ `$OUT_DIR/library.v1.rs` (ชื่อไฟล์มาจากชื่อ `package` ในไฟล์ `.proto`) — เราไม่ต้อง `include!` ไฟล์นี้ด้วยมือ เพราะ Tonic มี macro ช่วยเหลือชื่อ `tonic::include_proto!` ที่ทำหน้าที่นี้ให้:

```rust
pub mod book {
    tonic::include_proto!("library.v1");

    pub const FILE_DESCRIPTOR_SET: &[u8] = tonic::include_file_descriptor_set!("book_descriptor");
}
```

`tonic::include_proto!("library.v1")` ขยายออกมาเป็นการเรียก `include!(concat!(env!("OUT_DIR"), "/library.v1.rs"));` แบบตรง ๆ — ส่วน `tonic::include_file_descriptor_set!("book_descriptor")` ก็ทำงานคล้ายกันแต่ดึงไฟล์ binary ที่เราสั่งเขียนไว้ด้วย `.file_descriptor_set_path()` ข้างบน มาเป็น `&'static [u8]` ผ่าน `include_bytes!` ภายใน

**นี่คือจุดสำคัญที่ต้องเข้าใจให้ถูกต้องแม่นยำ (เชื่อมกับ Part 44-45 ตามที่สัญญาไว้ในเป้าหมายบทเรียน)**: `include_proto!`/`include_file_descriptor_set!` **เป็น declarative macro ธรรมดา** (แบบเดียวกับที่ Part 36 สอน ไม่ใช่ procedural macro) ที่ทำหน้าที่แค่ **ประกอบ path string แล้ว `include!` ไฟล์ที่มีอยู่แล้วในดิสก์** — ตัวโค้ด Rust จริง ๆ ที่กลายเป็น `struct Book { ... }`, `trait BookService { ... }` ฯลฯ ถูกสร้างขึ้น**ก่อนหน้านี้แล้ว**โดย `build.rs` ที่รันเป็นโปรแกรมแยกต่างหากในขั้นตอน **build script execution** (ขั้นตอนที่เกิดขึ้นก่อนการ compile โค้ดหลักของ crate เสียอีก) ไม่ใช่ในขั้นตอน**macro expansion**ที่ proc macro ทำงานอยู่

เทียบให้เห็นความต่างชัด ๆ กับ derive macro ที่ Part 44 สอน:

| | Derive macro (`#[derive(MyTrait)]` จาก Part 44) | `tonic-prost-build` + `include_proto!` |
|---|---|---|
| รันตอนไหน | ระหว่าง**compile ตัว crate นั้นเอง** — compiler เจอ attribute แล้วเรียก proc macro function ที่รับ `TokenStream` เข้า คืน `TokenStream` ออกมาแทรกกลับเข้าไปตรงตำแหน่งเดิม | รันเป็น**โปรแกรมแยกก่อน**การ compile crate เริ่มด้วยซ้ำ (`build.rs` คือ binary ที่ Cargo compile และรันเองก่อน) เขียนไฟล์ `.rs` ออกมาที่ `OUT_DIR` แล้ว**หลังจากนั้น**โค้ดหลักของ crate จึงถูก compile โดยมี `include!` ดึงไฟล์นั้นเข้ามาเป็นส่วนหนึ่งของ source |
| Input | `TokenStream` ของ Rust syntax ที่ compiler parse มาให้แล้ว | ไฟล์ `.proto` ที่เป็นภาษาของตัวเอง (ไม่ใช่ Rust เลย) ผ่าน `protoc` ซึ่งเป็นโปรแกรม C++ แยกต่างหาก |
| เครื่องมือ parse | `syn` (แปลง `TokenStream` เป็น AST ของ Rust) | `protoc` (แปลง `.proto` เป็น `FileDescriptorSet`) แล้ว `prost-build` แปลง descriptor นั้นเป็น Rust source string เอง (ไม่ใช้ `syn`/`quote` เลย) |
| ผลลัพธ์สุดท้ายที่ compiler เห็น | โค้ด Rust ที่ "แทรก" เข้าไปตรงตำแหน่ง `#[derive(...)]` โดยไม่มีไฟล์แยกให้เห็นเป็นชิ้น ๆ | ไฟล์ `.rs` จริงที่เขียนไว้ในดิสก์ (`$OUT_DIR/library.v1.rs`) ที่คุณสามารถเปิดอ่านด้วยตาได้ตรง ๆ (ผ่าน `cargo build` แล้วไปเปิดใน `target/.../out/`) |

สรุปสั้น ๆ ว่า: **proc macro คือ "ฟังก์ชัน Rust ที่ compiler เรียกระหว่างขั้นตอน compile"** ส่วน **`tonic-prost-build` คือ "โปรแกรมแยกที่รันก่อน แล้วเขียนไฟล์ Rust ธรรมดาไว้ให้ compile ทีหลัง"** — ทั้งสองแบบให้ "ความรู้สึก" เดียวกันจากมุมมองคนเขียนโค้ด (เขียนสั้น ได้ struct/trait เพียบมาให้ใช้แบบไม่ต้องพิมพ์มือ) แต่เป็นกลไกคนละชั้นของ Rust build pipeline โดยสิ้นเชิง — proc macro เป็นส่วนหนึ่งของ **compiler frontend**, ส่วน build script เป็นแค่ **โปรแกรมที่ Cargo รันแล้วดูผลลัพธ์ไฟล์** เท่านั้น Cargo ไม่รู้ (และไม่สนใจ) ด้วยซ้ำว่าไฟล์ที่ `build.rs` เขียนออกมาคือ "โค้ด Protobuf ที่ generate" หรือ "ไฟล์อะไรก็ได้ที่โปรแกรมนั้นอยากเขียน"

#### โค้ดที่ Generate จริง: เปิดดูเนื้อในให้เห็นภาพ

เพื่อไม่ให้ทุกอย่างเป็นเพียง "มายากล" มาดูโค้ดจริงที่ `tonic-prost-build` generate ออกมาจาก `book.proto` ข้างบน (คัดมาจาก `$OUT_DIR/library.v1.rs` หลัง `cargo build` จริง) — เริ่มจาก message `Book`:

```rust
// This file is @generated by prost-build.
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct Book {
    #[prost(int64, tag = "1")]
    pub id: i64,
    #[prost(string, tag = "2")]
    pub title: ::prost::alloc::string::String,
    #[prost(string, tag = "3")]
    pub author: ::prost::alloc::string::String,
    #[prost(int32, tag = "4")]
    pub year: i32,
    #[prost(double, tag = "5")]
    pub price: f64,
}
```

สังเกตว่าโค้ดนี้ยังใช้ `#[derive(::prost::Message)]` อยู่ — นี่**คือ**derive macro ตัวจริงตามความหมายของ Part 44 (มาจาก crate `prost-derive`) เพียงแต่**ตัวไฟล์ที่มี `#[derive(...)]` นี้เขียนอยู่**ถูกสร้างขึ้นโดย build script ก่อนหน้านี้แล้ว — พูดให้ชัดคือ: build script (`tonic-prost-build`) ทำหน้าที่ "เขียน struct + แนบ `#[derive(...)]` ให้อัตโนมัติ" แล้ว derive macro (`prost-derive`) ค่อยทำงานตอน compile struct นั้นตามปกติอีกต่อหนึ่ง — สองกลไกซ้อนกันอยู่ในระบบเดียวกัน คนละหน้าที่กัน

ต่อด้วยฝั่ง client — คัดมาเฉพาะ 3 เมธอดที่แสดงรูปแบบการเรียกทั้ง 4 แบบ (unary, server streaming, client streaming, bidirectional) ให้เห็นครบในที่เดียว (จาก `book_service_client` module):

```rust
pub async fn list_books(
    &mut self,
    request: impl tonic::IntoRequest<super::ListBooksRequest>,
) -> std::result::Result<tonic::Response<tonic::codec::Streaming<super::Book>>, tonic::Status> {
    // ... (self.inner.server_streaming(...) ด้านใน)
}

pub async fn upload_books(
    &mut self,
    request: impl tonic::IntoStreamingRequest<Message = super::CreateBookRequest>,
) -> std::result::Result<tonic::Response<super::UploadSummary>, tonic::Status> {
    // ... (self.inner.client_streaming(...) ด้านใน)
}

pub async fn watch_availability(
    &mut self,
    request: impl tonic::IntoStreamingRequest<Message = super::AvailabilityUpdate>,
) -> std::result::Result<
    tonic::Response<tonic::codec::Streaming<super::AvailabilityUpdate>>,
    tonic::Status,
> {
    // ... (self.inner.streaming(...) ด้านใน — ใช้เมธอดคนละชื่อกับ server_streaming/client_streaming
    //     เพราะเป็น bidirectional โดยเฉพาะ)
}
```

สังเกตว่า**ชนิดของ parameter/return แต่ละเมธอดสะท้อนรูปแบบการเรียกตรง ๆ** ตามที่หัวข้อ 80.6 จะอธิบายละเอียด: `get_book`/`create_book` (unary จากโค้ดก่อนหน้า) รับ `impl IntoRequest<T>` คืน `Response<T>` เดี่ยว ๆ, `list_books` (server streaming) รับ request เดี่ยวแต่คืน `Response<Streaming<T>>`, `upload_books` (client streaming) รับ `impl IntoStreamingRequest<...>` แต่คืน response เดี่ยว, และ `watch_availability` (bidirectional) รับ stream คืน stream ทั้งสองด้าน — Tonic เลือกเรียก internal method คนละชื่อ (`unary`, `server_streaming`, `client_streaming`, `streaming`) ตามรูปแบบที่ตรงกัน ทั้งหมดนี้ compiler ตรวจสอบให้ตอน compile time เต็มรูปแบบ ผิดรูปแบบการเรียก (เช่น พยายามส่ง stream ให้ RPC ที่เป็น unary) จะเจอ compile error ทันที ไม่ใช่ runtime error

ต่อด้วย trait ฝั่ง server ที่คุณต้อง implement (จะใช้จริงในหัวข้อ 80.4):

```rust
pub mod book_service_server {
    use tonic::codegen::*;

    #[async_trait]
    pub trait BookService: std::marker::Send + std::marker::Sync + 'static {
        async fn get_book(
            &self,
            request: tonic::Request<super::GetBookRequest>,
        ) -> std::result::Result<tonic::Response<super::Book>, tonic::Status>;

        async fn create_book(
            &self,
            request: tonic::Request<super::CreateBookRequest>,
        ) -> std::result::Result<tonic::Response<super::Book>, tonic::Status>;

        /// Server streaming response type for the ListBooks method.
        type ListBooksStream: tonic::codegen::tokio_stream::Stream<
                Item = std::result::Result<super::Book, tonic::Status>,
            > + std::marker::Send + 'static;
        async fn list_books(
            &self,
            request: tonic::Request<super::ListBooksRequest>,
        ) -> std::result::Result<tonic::Response<Self::ListBooksStream>, tonic::Status>;

        // ... upload_books, watch_availability ในรูปแบบเดียวกัน
    }

    #[derive(Debug)]
    pub struct BookServiceServer<T> {
        inner: Arc<T>,
        // ฟิลด์ควบคุม compression/ขนาด message สูงสุด (ไม่ใช้ในบทนี้)
        accept_compression_encodings: EnabledCompressionEncodings,
        send_compression_encodings: EnabledCompressionEncodings,
        max_decoding_message_size: Option<usize>,
        max_encoding_message_size: Option<usize>,
    }

    impl<T> BookServiceServer<T> {
        pub fn new(inner: T) -> Self {
            Self::from_arc(Arc::new(inner))
        }
        pub fn with_interceptor<F>(inner: T, interceptor: F) -> InterceptedService<Self, F>
        where
            F: tonic::service::Interceptor,
        {
            // ...
        }
    }
}
```

สังเกตสามจุดที่สำคัญมากในโค้ดนี้:

1. **`#[async_trait]`** ที่อยู่เหนือ `pub trait BookService` — นี่คือ attribute macro จาก crate `async-trait` ที่ `tonic-prost-build` แนบมาให้อัตโนมัติเพราะ Rust trait (จนถึงและรวมเวอร์ชันที่ใช้ในบทนี้) ยังมีข้อจำกัดบางอย่างกับ `async fn` ใน trait ที่ต้องเป็น trait object ได้ (`dyn BookService`) — `async-trait` แปลง `async fn foo(&self) -> T` เป็น `fn foo(&self) -> Pin<Box<dyn Future<Output = T> + Send>>` เบื้องหลัง ซึ่งเป็น trick ที่ทำให้ trait ที่มี async method ยังใช้เป็น trait object ได้ **นี่คือ proc macro ตัวจริงตามความหมายของ Part 44** ที่ทำงานตอน compile — ต่างจากตัว `.rs` ไฟล์เองที่ถูก build script generate มาก่อนแล้ว (สองกลไกซ้อนกัน เหมือนที่อธิบายไปแล้วข้างบนกับ `prost::Message`)
2. **`type ListBooksStream: Stream<...> + Send + 'static`** — associated type ที่คุณต้องกำหนดเองตอน implement (หัวข้อ 80.4/80.6 จะใช้ `Pin<Box<dyn Stream<...> + Send>>` เป็นคำตอบที่ตรงไปตรงมาที่สุด)
3. **`BookServiceServer::with_interceptor(inner, interceptor)`** — เมธอดนี้คือจุดเชื่อมกับหัวข้อ 80.8 (Interceptor) ที่จะสาธิตจริงต่อไป

### 80.4 Implementing Server: Unary RPCs กับ `BookStore`

#### เลือก In-Memory Store แทน SQLx: เหตุผลของการเลือกในบทนี้

Part 70 สอน SQLx สำหรับต่อ PostgreSQL ไว้แล้ว และตัวอย่างในบทนี้**เลือกใช้ `Mutex<HashMap<i64, Book>>` แบบ in-memory แทน** — นี่เป็นการตัดสินใจที่ตั้งใจ ไม่ใช่การมองข้าม ด้วยเหตุผลตรงไปตรงมาสองข้อ:

1. **โฟกัสของบทนี้คือกลไกของ gRPC** (การเขียน `.proto`, การ implement trait ที่ generate มา, การจัดการ streaming ทั้ง 4 แบบ, interceptor, error handling) — การผสม async database pool เข้ามาด้วยจะเพิ่ม failure mode ที่ไม่เกี่ยวกับ gRPC เลย (connection pool exhaustion, migration, transaction) ซึ่งเบี่ยงความสนใจออกจากประเด็นหลัก
2. **Trait ที่ `tonic-prost-build` generate ให้ไม่ผูกกับวิธีเก็บข้อมูลเลยแม้แต่นิดเดียว** — `BookService::get_book(&self, request: Request<GetBookRequest>) -> Result<Response<Book>, Status>` เป็น signature ที่เป็นกลางสมบูรณ์ ข้างในจะดึงจาก `HashMap` ในหน่วยความจำ หรือจะ `sqlx::query_as!(...)` เรียก PostgreSQL ก็ implement ได้เหมือนกันทุกประการในเชิง signature — **การสลับจาก in-memory ไปเป็น SQLx ในโปรเจกต์จริงทำได้โดยไม่ต้องแก้ trait bound หรือโค้ดที่ generate มาแม้แต่บรรทัดเดียว** เปลี่ยนแค่เนื้อในของแต่ละ method เท่านั้น (เปลี่ยน `self.store.get(id)` เป็น `sqlx::query_as!("SELECT * FROM books WHERE id = $1", id).fetch_optional(&self.pool).await` แล้วแปลง error เป็น `Status::internal` ตามความเหมาะสม)

```rust
pub mod book {
    tonic::include_proto!("library.v1");

    pub const FILE_DESCRIPTOR_SET: &[u8] = tonic::include_file_descriptor_set!("book_descriptor");
}

use book::Book;
use std::collections::HashMap;
use std::sync::Mutex;

/// เก็บข้อมูลหนังสือแบบ in-memory (ใช้ Mutex<HashMap> ธรรมดา เพียงพอสำหรับสาธิต
/// unary/streaming ทุกรูปแบบในบทนี้ — โปรเจกต์จริงสลับไปใช้ SQLx (Part 70) ได้โดย
/// ไม่ต้องแก้ signature ของ trait ที่ tonic-prost-build generate ให้เลยแม้แต่บรรทัดเดียว)
pub struct BookStore {
    books: Mutex<HashMap<i64, Book>>,
    next_id: Mutex<i64>,
}

impl BookStore {
    pub fn new() -> Self {
        let mut seed = HashMap::new();
        seed.insert(1, Book {
            id: 1,
            title: "The Rust Programming Language".to_string(),
            author: "Steve Klabnik & Carol Nichols".to_string(),
            year: 2019,
            price: 39.99,
        });
        seed.insert(2, Book {
            id: 2,
            title: "Programming Rust".to_string(),
            author: "Jim Blandy & Jason Orendorff".to_string(),
            year: 2021,
            price: 44.99,
        });
        seed.insert(3, Book {
            id: 3,
            title: "Zero To Production In Rust".to_string(),
            author: "Luca Palmieri".to_string(),
            year: 2022,
            price: 29.99,
        });
        BookStore { books: Mutex::new(seed), next_id: Mutex::new(4) }
    }

    pub fn get(&self, id: i64) -> Option<Book> {
        self.books.lock().unwrap().get(&id).cloned()
    }

    pub fn list(&self) -> Vec<Book> {
        let mut items: Vec<Book> = self.books.lock().unwrap().values().cloned().collect();
        items.sort_by_key(|b| b.id);
        items
    }

    pub fn create(&self, title: String, author: String, year: i32, price: f64) -> Book {
        let mut next_id = self.next_id.lock().unwrap();
        let id = *next_id;
        *next_id += 1;
        let book = Book { id, title, author, year, price };
        self.books.lock().unwrap().insert(id, book.clone());
        book
    }
}
```

ใช้ `std::sync::Mutex` (ไม่ใช่ `tokio::sync::Mutex`) เพราะทุก operation ในนี้เป็นแค่การอ่าน/เขียน `HashMap` ที่จบเร็วมาก ไม่มีจุดใด `.await` ขณะถือ lock — ตรงตามหลักที่ Part 39-40 สอนไว้ว่าให้ใช้ `std::sync::Mutex` เมื่อ critical section ไม่มี `.await` (เร็วกว่า async mutex เพราะไม่ต้องผ่าน executor)

#### Implement `BookService` trait: `get_book` และ `create_book`

```rust
use grpc_demo::book::book_service_server::{BookService, BookServiceServer};
use grpc_demo::book::{Book, CreateBookRequest, GetBookRequest};
use grpc_demo::BookStore;
use std::sync::Arc;
use tonic::{Request, Response, Status};

pub struct MyBookService {
    store: Arc<BookStore>,
}

#[tonic::async_trait]
impl BookService for MyBookService {
    async fn get_book(&self, request: Request<GetBookRequest>) -> Result<Response<Book>, Status> {
        let id = request.into_inner().id;
        println!("[server] GetBook id={id}");
        match self.store.get(id) {
            Some(book) => Ok(Response::new(book)),
            None => Err(Status::not_found(format!("ไม่พบหนังสือ id={id}"))),
        }
    }

    async fn create_book(
        &self,
        request: Request<CreateBookRequest>,
    ) -> Result<Response<Book>, Status> {
        let req = request.into_inner();
        println!("[server] CreateBook title={}", req.title);

        if req.title.trim().is_empty() {
            return Err(Status::invalid_argument("title ต้องไม่เป็นค่าว่าง"));
        }
        if req.price < 0.0 {
            return Err(Status::invalid_argument(format!(
                "price ต้องไม่ติดลบ (ได้รับ {})",
                req.price
            )));
        }

        let book = self.store.create(req.title, req.author, req.year, req.price);
        Ok(Response::new(book))
    }

    // ... (list_books, upload_books, watch_availability — ดูหัวข้อ 80.6)
}
```

จุดที่ต้องสังเกตให้ชัด:

- **`#[tonic::async_trait]`**: นี่คือ **re-export ของ `async_trait::async_trait` ตัวเดียวกันที่ trait ที่ generate มาใช้** (ดูหัวข้อ 80.3) — Tonic export มันให้ผ่าน path ของตัวเองเพื่อความสะดวก (ไม่ต้องเพิ่ม `async-trait` เป็น dependency ตรง ๆ ในโปรเจกต์ที่ใช้แค่ Tonic) แต่เป็น macro ตัวเดียวกันเป๊ะ **ต้องแนบ attribute นี้ไว้เหนือ `impl` block เสมอ** เพราะ trait ที่นิยามไว้ (จากหัวข้อ 80.3) ถูกแปลงร่างโดย `#[async_trait]` ไปแล้ว — ถ้า `impl` ไม่มี attribute เดียวกัน สอง "รูปร่าง" ของ async function จะไม่ตรงกัน (native `async fn` เทียบกับ `fn -> Pin<Box<dyn Future>>`) ทำให้ compile ไม่ผ่าน (ดูข้อพิสูจน์จริงพร้อม error message เต็มในกับดักข้อ 2)
- **`request.into_inner()`**: `tonic::Request<T>` ห่อ payload จริงไว้ข้างในพร้อม metadata (คล้าย `axum::extract::Request` ที่ห่อ body พร้อม header) — `.into_inner()` ดึงเอาแค่ค่า `T` ออกมาตรง ๆ ทิ้ง metadata (ใช้เมื่อไม่สนใจ metadata อีกแล้ว) ส่วน `.metadata()` (ใช้ในหัวข้อ interceptor) เข้าถึง metadata โดยไม่ทำลาย request
- **`Response::new(book)`**: ห่อ payload กลับเป็น `tonic::Response<T>` — สมมาตรกับ `Request<T>` (มี metadata ที่ปรับได้เหมือนกันถ้าต้องการส่ง header กลับ)

#### รัน Server จริงและตรวจสอบผลลัพธ์

หลัง implement ครบทั้ง 5 RPC (ดูโค้ดเต็มในหัวข้อ 80.6/80.8) ต่อด้วยฟังก์ชัน `main` ที่ผูก `MyBookService` เข้ากับ `Server` ของ Tonic:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "127.0.0.1:50051".parse()?;
    let store = Arc::new(BookStore::new());
    let book_service = MyBookService { store };

    println!("BookService gRPC server listening on {addr}");

    tonic::transport::Server::builder()
        .add_service(BookServiceServer::new(book_service))
        .serve(addr)
        .await?;

    Ok(())
}
```

รันจริงด้วย `cargo run --release --bin server` (โค้ดฉบับสมบูรณ์ที่รันจริงในบทนี้ผูก interceptor และ reflection เข้ามาด้วย — ดูหัวข้อ 80.8-80.9) ได้ output:

```text
BookService gRPC server listening on 127.0.0.1:50051
```

server จะค้างรออยู่ตรงนี้ (เพราะ `.serve(addr).await` เป็น future ที่ไม่จบจนกว่าจะถูก interrupt) พร้อมรับ connection — หัวข้อถัดไปจะเขียน client มาคุยกับมันจริง

#### เชื่อมกับ Part 39-40: ทำไม Trait ต้องมี Bound `Send + Sync + 'static`

สังเกตกลับไปที่ trait `BookService` ที่ generate มา (หัวข้อ 80.3): `pub trait BookService: std::marker::Send + std::marker::Sync + 'static` — bound นี้ไม่ได้ใส่มาเล่น ๆ แต่จำเป็นโดยตรงเพราะ Tonic server รับ**หลาย connection พร้อมกัน**และแต่ละ connection อาจยิงหลาย RPC พร้อมกันด้วย (ตามที่หัวข้อ 80.1 พิสูจน์เรื่อง multiplexing ไปแล้ว) — ทุก request handler ที่ทำงานพร้อมกันเหล่านี้ต้องแชร์ `&self` (คือ `MyBookService` ตัวเดียวกัน) ข้าม thread ของ Tokio runtime ได้อย่างปลอดภัย ตรงตามที่ **Part 39-40** สอนไว้ว่า type ที่แชร์ข้าม thread ต้องเป็น `Sync` (เข้าถึงพร้อมกันจากหลาย thread ได้อย่างปลอดภัย) และค่าที่ถูกส่งเข้า thread อื่นต้องเป็น `Send`

`MyBookService { store: Arc<BookStore> }` ผ่านเงื่อนไขนี้ได้เพราะ `Arc<T>` เป็น `Send + Sync` เมื่อ `T: Send + Sync` (`BookStore` มีแค่ `Mutex<HashMap<...>>` ซึ่งเป็น `Send + Sync` ทั้งคู่ตามกฎที่ Part 39-40 อธิบายไว้ว่า `Mutex<T>` ทำให้ `T` ที่ไม่ใช่ `Sync` กลายเป็น `Sync` ได้ผ่านการล็อก) — พิสูจน์ให้เห็นภาพจริงด้วยการยิง `GetBook` **พร้อมกัน 20 ครั้ง** จาก 20 `tokio::spawn` task บน connection เดียวกัน:

```rust
let mut handles = Vec::new();
for i in 0..20 {
    let channel = channel.clone();
    handles.push(tokio::spawn(async move {
        let mut client = BookServiceClient::new(channel);
        let req = with_auth(GetBookRequest { id: (i % 3) + 1 });
        let resp = client.get_book(req).await.unwrap().into_inner();
        (i, resp.id, resp.title)
    }));
}
for h in handles {
    results.push(h.await.unwrap());
}
```

**ผลลัพธ์จริงจากการรัน** (ตัดมาบางส่วน — ครบทั้ง 20 อันไม่มี error/panic เลยแม้แต่ตัวเดียว):

```text
ยิง GetBook พร้อมกัน 20 request สำเร็จทั้งหมด:
  task#00 -> book id=1 title=The Rust Programming Language
  task#01 -> book id=2 title=Programming Rust
  task#02 -> book id=3 title=Zero To Production In Rust
  ...
  task#19 -> book id=2 title=Programming Rust
รวม 20 response ครบทุกอัน ไม่มี error/panic แม้แต่ตัวเดียว
```

ทั้ง 20 task เข้าถึง `self.store.get(id)` (ซึ่งข้างในคือ `self.books.lock().unwrap()`) พร้อมกันได้อย่างปลอดภัยเพราะ `std::sync::Mutex` การันตี **mutual exclusion** ตามที่ Part 39-40 สอนไว้ — ต่อให้สอง request มาถึงจังหวะเดียวกันเป๊ะ ตัวหนึ่งจะรอ (block เฉพาะ thread นั้น ไม่ใช่ทั้ง runtime เพราะ critical section เร็วมากและไม่มี `.await` อยู่ข้างใน) จนกว่าตัวแรกปลดล็อก ไม่มีทางที่ `HashMap` จะถูกอ่าน/เขียนพร้อมกันแบบ data race ได้เลยแม้แต่ทางทฤษฎี — คุณสมบัตินี้ borrow checker และ trait bound (`Send`/`Sync`) การันตีให้**ตั้งแต่ตอน compile** ไม่ต้องรอไป crash ตอน production เหมือนภาษาที่ไม่มีการตรวจสอบนี้

### 80.5 Implementing Client: Round Trip จริงพร้อม Output จริง

#### `BookServiceClient`: เชื่อมต่อและเรียก RPC

```rust
use grpc_demo::book::book_service_client::BookServiceClient;
use grpc_demo::book::GetBookRequest;
use tonic::Request;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = BookServiceClient::connect("http://127.0.0.1:50051").await?;

    let resp = client.get_book(Request::new(GetBookRequest { id: 1 })).await?;
    println!("ได้รับ: {:?}", resp.into_inner());

    Ok(())
}
```

`BookServiceClient::connect(...)` สร้าง HTTP/2 connection ไปยัง server แล้วคืน client ที่มีเมธอดตรงกับทุก RPC ใน `.proto` (`get_book`, `create_book`, `list_books`, `upload_books`, `watch_availability`) — สังเกตว่า**เมธอดพวกนี้ type-safe เต็มรูปแบบ**: `client.get_book(...)` รับได้แค่สิ่งที่แปลงเป็น `Request<GetBookRequest>` ได้เท่านั้น ผิด type จะเจอ compile error ทันที ไม่ใช่ runtime error แบบที่มักเกิดกับ REST client ที่ยิง JSON แล้วพลาด field ไปเงียบ ๆ

รันจริง (มี server รันอยู่ที่ port 50051 อยู่ก่อนแล้ว) ด้วย `cargo run --release --bin client` — ตัวอย่าง client เต็มรูปแบบที่ใช้จริงตลอดบทนี้สาธิตทุกสถานการณ์ (unary ทั้งสำเร็จ/error, streaming ทั้ง 3 แบบ, error details) ในโปรแกรมเดียว มาดูผลลัพธ์จริงทีละส่วนในหัวข้อถัดไปควบคู่กับการอธิบายแต่ละรูปแบบการเรียก

### 80.6 รูปแบบการเรียก gRPC ทั้ง 4 แบบ

gRPC นิยามรูปแบบการเรียกไว้ 4 แบบตามจำนวนข้อความที่วิ่งไปมาในทั้งสองทิศทาง — ความแตกต่างนี้ถูกกำหนดไว้ **ที่ระดับ `.proto`** (ผ่าน keyword `stream`) ไม่ใช่สิ่งที่ตัดสินใจได้ตอน runtime เหมือนการเลือก HTTP method ใน REST

#### 80.6.1 Unary: หนึ่ง Request หนึ่ง Response

นี่คือรูปแบบพื้นฐานที่สุด (ที่ `get_book`/`create_book` ใช้ในหัวข้อ 80.4/80.5) — client ส่ง request หนึ่งชิ้น รอ server ตอบกลับหนึ่งชิ้น จบ เทียบเท่ากับการเรียก REST API แบบปกติทุกประการในเชิงรูปแบบการสื่อสาร (ต่างกันแค่ wire format และ transport อย่างที่อธิบายในหัวข้อ 80.1)

ผลลัพธ์จริงจากการรัน client (ส่วน unary ทั้งหมด รวม error path):

```text
=== 1) Unary: GetBook(id=1) พร้อม token ที่ถูกต้อง ===
ได้รับ: Book { id: 1, title: "The Rust Programming Language", author: "Steve Klabnik & Carol Nichols", year: 2019, price: 39.99 }

=== 2) Unary: GetBook(id=999) — ควรได้ NOT_FOUND ===
gRPC error: code=NotFound message="ไม\u{e48}พบหน\u{e31}งส\u{e37}อ id=999"

=== 4) Unary: CreateBook ด้วยข้อมูลถูกต้อง ===
สร้างสำเร็จ: Book { id: 4, title: "Rust for Rustaceans", author: "Jon Gjengset", year: 2021, price: 34.99 }

=== 5) Unary: CreateBook ด้วย price ติดลบ — ควรได้ INVALID_ARGUMENT ===
gRPC error: code=InvalidArgument message="price ต\u{e49}องไม\u{e48}ต\u{e34}ดลบ (ได\u{e49}ร\u{e31}บ -10)"
```

**หมายเหตุเกี่ยวกับ `\u{e48}` ที่ปรากฏใน output จริง**: นี่ไม่เกี่ยวข้องกับ gRPC เลยแม้แต่น้อย — เป็นพฤติกรรมของ `{:?}` (trait `Debug`) ของ Rust ที่ escape **Unicode combining mark** (เช่น ไม้เอก U+0E48 ในคำว่า "ไม่") ให้เป็น `\u{...}` เสมอเมื่อ debug-print เพราะตัวอักษรกลุ่มนี้ "เกาะ" กับตัวอักษรก่อนหน้าและอาจทำให้ debug output อ่านกำกวมถ้าพิมพ์ตรง ๆ — ถ้า print ด้วย `{}` (Display) ธรรมดาแทน `{:?}` (Debug) ข้อความจะแสดงภาษาไทยปกติทุกตัวอักษร (สังเกตว่าข้อความอื่น ๆ ในบทนี้ที่ print ด้วย `println!("{msg}")` ตรง ๆ ไม่ผ่าน `{:?}` แสดงภาษาไทยได้ปกติไม่มีปัญหานี้เลย)

#### 80.6.2 Server Streaming: หนึ่ง Request, Server ส่ง Response กลับมาหลายชิ้น

`ListBooks` เป็นตัวอย่างที่ realistic มาก: การดึง**แคตตาล็อกหนังสือทั้งหมด**อาจมีข้อมูลจำนวนมากเกินกว่าจะส่งเป็น response เดียวได้อย่างมีประสิทธิภาพ (ทั้งเรื่อง memory ฝั่ง server ที่ต้องโหลดทุกอย่างมาก่อนส่ง และ latency ฝั่ง client ที่ต้องรอทุกอย่างมาครบก่อนเริ่มประมวลผล) — server streaming ให้ server **ส่งทีละชิ้นทันทีที่พร้อม** โดย client เริ่มประมวลผลได้จากชิ้นแรกโดยไม่ต้องรอชิ้นสุดท้าย

ความรู้สึกนี้คล้ายกับที่ **Part 77 (WebSocket)** สอนไว้เรื่อง "server ส่งข้อมูลมาเรื่อย ๆ โดยไม่ต้องให้ client ขอใหม่ทุกครั้ง" — แต่กลไกต่างกันอย่างสำคัญ: WebSocket เป็น protocol ที่เปิด full-duplex channel แบบดิบ ๆ (คุณต้องกำหนด message format เองทั้งหมด) ส่วน gRPC server streaming ยังอยู่ภายใต้ schema ที่ `.proto` กำหนดไว้ทุกชิ้นที่ stream ผ่านมา — type-safe และ strict กว่า WebSocket มาก แต่ไม่ได้ให้ความอิสระแบบ full-duplex ในทันที (การเรียกเป็น RPC ครั้งเดียวที่ "ตอบกลับเป็น stream" ไม่ใช่ช่องทางสื่อสารสองทางแบบเปิดตลอดเหมือน WebSocket — ถ้าต้องการสองทางพร้อมกันจริง ๆ ต้องใช้ bidirectional streaming ในหัวข้อ 80.6.4)

**ฝั่ง Server:**

```rust
use futures::Stream;
use std::pin::Pin;
use std::time::Duration;

type ListBooksStream = Pin<Box<dyn Stream<Item = Result<Book, Status>> + Send + 'static>>;

async fn list_books(
    &self,
    request: Request<ListBooksRequest>,
) -> Result<Response<Self::ListBooksStream>, Status> {
    let page_size = request.into_inner().page_size.max(1) as usize;
    let books = self.store.list();
    println!("[server] ListBooks page_size={page_size} (มีทั้งหมด {} เล่ม)", books.len());

    let (tx, rx) = tokio::sync::mpsc::channel(4);
    tokio::spawn(async move {
        for chunk in books.chunks(page_size) {
            for book in chunk {
                // จำลอง latency ของแต่ละ "หน้า" แคตตาล็อกที่ต้องดึงจากดิสก์/DB จริง
                tokio::time::sleep(Duration::from_millis(150)).await;
                if tx.send(Ok(book.clone())).await.is_err() {
                    return; // client ปิด stream ไปแล้ว ไม่ต้อง stream ต่อ
                }
            }
        }
    });

    let output_stream = tokio_stream::wrappers::ReceiverStream::new(rx);
    Ok(Response::new(Box::pin(output_stream) as Self::ListBooksStream))
}
```

รูปแบบนี้เป็น**pattern มาตรฐานที่ใช้กันแพร่หลายที่สุด**สำหรับ server streaming ใน Tonic: สร้าง `tokio::sync::mpsc::channel`, `tokio::spawn` งานที่ค่อย ๆ `.send()` เข้า channel ทีละชิ้น (ในนี้จำลอง latency ด้วย `sleep` เพื่อให้เห็นว่าแต่ละชิ้นมาไม่พร้อมกันจริง ๆ) แล้วห่อฝั่งรับ (`rx`) ด้วย `ReceiverStream` จาก `tokio-stream` ให้กลายเป็น `impl Stream` ที่ trait ต้องการ — สังเกตว่า `if tx.send(...).is_err() { return; }` คือการเช็คว่า**client ปิด connection ไปแล้วหรือยัง** (ถ้าปิดแล้ว `rx` จะถูก drop ทำให้ `tx.send()` คืน `Err`) — ถ้าไม่เช็คจุดนี้ task ที่ spawn ไว้จะยังพยายามทำงาน (และ sleep) ต่อไปทั้งที่ไม่มีใครรอผลแล้ว เปลืองทรัพยากรฝั่ง server โดยไม่จำเป็น

**ฝั่ง Client:**

```rust
let mut stream = client
    .list_books(Request::new(ListBooksRequest { page_size: 2 }))
    .await?
    .into_inner();

while let Some(book) = stream.message().await? {
    println!("  stream chunk -> {} โดย {}", book.title, book.author);
}
println!("  (server ปิด stream แล้ว)");
```

`stream.message().await?` คืน `Some(Book)` ทีละชิ้นจนกว่า server จะปิด stream (แล้วคืน `None`) — สังเกตว่า loop นี้หน้าตาคล้ายการวน `while let Some(msg) = websocket.next().await` ที่ Part 77 สอนไว้มาก แม้เบื้องหลังเป็นกลไกคนละแบบตามที่อธิบายไปแล้ว

**Output จริงจากการรัน** (มี 4 เล่มในระบบตอนนี้เพราะ CreateBook ในหัวข้อก่อนเพิ่มไปแล้ว 1 เล่ม, `page_size=2`):

```text
=== 6) Server streaming: ListBooks(page_size=2) ===
  stream chunk -> The Rust Programming Language โดย Steve Klabnik & Carol Nichols
  stream chunk -> Programming Rust โดย Jim Blandy & Jason Orendorff
  stream chunk -> Zero To Production In Rust โดย Luca Palmieri
  stream chunk -> Rust for Rustaceans โดย Jon Gjengset
  (server ปิด stream แล้ว)
```

สังเกตว่าแม้ตั้ง `page_size: 2` (จำลองการแบ่งหน้าละ 2 เล่ม) client ก็ยังเห็นเป็น**การไหลของ `Book` ทีละชิ้นต่อเนื่องกัน** ไม่ใช่ chunk ละ 2 เล่ม — เพราะ gRPC stream ส่ง**message เดี่ยว ๆ**ทีละชิ้นเสมอ (ในตัวอย่างนี้ `page_size` มีผลแค่ต่อ**จังหวะ**การ sleep ระหว่าง "หน้า" ฝั่ง server เท่านั้น ไม่ได้เปลี่ยนรูปแบบของสิ่งที่ client เห็น) — ถ้าต้องการให้ client เห็นเป็นชุด ๆ จริง ๆ ต้องออกแบบ message type ให้เป็น `repeated Book books = 1;` ภายในหนึ่ง response message แทน

#### 80.6.3 Client Streaming: Client ส่ง Request หลายชิ้น, Server ตอบกลับหนึ่งเดียว

`UploadBooks` จำลองสถานการณ์ **batch upload**: client มีหนังสือหลายเล่มที่ต้องการเพิ่มเข้าระบบทีเดียว (เช่น import จากไฟล์ CSV/Excel) แทนที่จะเรียก `CreateBook` (unary) ทีละเล่มซึ่งต้องรอ round trip ครบทุกครั้ง client streaming ให้ส่งทุกเล่ม**ไปเรื่อย ๆ บน connection เดียว**แล้วรอสรุปผลครั้งเดียวตอนจบ

**ฝั่ง Server:**

```rust
async fn upload_books(
    &self,
    request: Request<Streaming<CreateBookRequest>>,
) -> Result<Response<UploadSummary>, Status> {
    let mut stream = request.into_inner();
    let mut total_received = 0;
    let mut total_created = 0;
    let mut errors = Vec::new();

    while let Some(item) = stream.message().await? {
        total_received += 1;
        println!("[server] UploadBooks รับ #{total_received}: {}", item.title);
        if item.title.trim().is_empty() {
            errors.push(format!("รายการที่ {total_received}: title ว่าง — ข้าม"));
            continue;
        }
        self.store.create(item.title, item.author, item.year, item.price);
        total_created += 1;
    }

    Ok(Response::new(UploadSummary { total_received, total_created, errors }))
}
```

สังเกตว่า signature ตรงข้ามกับ server streaming เป๊ะ ๆ: **input เป็น `Streaming<T>` (ต้องอ่านหลายครั้ง) แต่ output เป็น `Response<T>` เดี่ยว ๆ (คืนครั้งเดียวตอนจบ)** — logic ข้างในคือ loop อ่านทีละชิ้นด้วย `stream.message().await?` (เหมือนฝั่ง client ตอนอ่าน server streaming ในหัวข้อก่อน) สะสมผลไว้ในตัวแปรท้องถิ่น แล้วค่อยสร้าง `Response` เดียวตอน loop จบ (คือตอนที่ client ปิด stream ฝั่งส่งของตัวเองแล้ว)

**ฝั่ง Client:**

```rust
let uploads = vec![
    CreateBookRequest { title: "Async Rust".to_string(), author: "Multiple Authors".to_string(), year: 2023, price: 19.99 },
    CreateBookRequest { title: "".to_string(), author: "Ghost".to_string(), year: 2023, price: 9.99 },
    CreateBookRequest { title: "Effective Rust".to_string(), author: "David Drysdale".to_string(), year: 2024, price: 24.99 },
    CreateBookRequest { title: "Rust Atomics and Locks".to_string(), author: "Mara Bos".to_string(), year: 2023, price: 22.5 },
];
let outbound = tokio_stream::iter(uploads);
let summary = client.upload_books(outbound).await?.into_inner();
println!("  summary: received={} created={} errors={:?}", summary.total_received, summary.total_created, summary.errors);
```

`tokio_stream::iter(uploads)` แปลง `Vec<CreateBookRequest>` ธรรมดาให้กลายเป็น `impl Stream<Item = CreateBookRequest>` — Tonic client รับ**อะไรก็ตามที่เป็น `Stream`** เป็น request ของ client-streaming RPC ได้ทันที (ไม่จำเป็นต้องมาจาก channel เสมอไป) สังเกตว่ารายการที่สองตั้งใจให้ `title` เป็นค่าว่างเพื่อดู error-handling path ฝั่ง server (สะสมเป็น error message ใน `errors` แทนที่จะทำให้ทั้ง RPC ล้มเหลว — เป็นการออกแบบที่ต่างจาก unary ตรงที่ **error ของ "รายการย่อยหนึ่งชิ้นใน stream" ไม่จำเป็นต้องทำให้ RPC ทั้งตัวล้มเหลว** ถ้าไม่ต้องการให้เป็นแบบนั้น)

**Output จริงจากการรัน:**

```text
=== 7) Client streaming: UploadBooks (ส่ง 4 เล่มรวด) ===
  summary: received=4 created=3 errors=["รายการท\u{e35}\u{e48} 2: title ว\u{e48}าง — ข\u{e49}าม"]
```

ฝั่ง server log สอดคล้องกันเป๊ะ (พิสูจน์ว่า message มาถึงทีละชิ้นจริง ไม่ใช่รอครบก่อนแล้วประมวลผลทีเดียว):

```text
[server] UploadBooks รับ #1: Async Rust
[server] UploadBooks รับ #2:
[server] UploadBooks รับ #3: Effective Rust
[server] UploadBooks รับ #4: Rust Atomics and Locks
[server] UploadBooks จบ: received=4 created=3 errors=1
```

#### 80.6.4 Bidirectional Streaming: ทั้งสองฝั่ง Stream พร้อมกัน อิสระจากกัน

`WatchAvailability` คือรูปแบบที่ทรงพลังที่สุดและซับซ้อนที่สุด: client ส่ง `AvailabilityUpdate` เข้ามาเรื่อย ๆ (เช่น อัปเดตจำนวนสำเนาที่พร้อมยืมของหนังสือแต่ละเล่มจากอุปกรณ์สแกนบาร์โค้ดหน้าเคาน์เตอร์) **ในเวลาเดียวกัน** server ก็ส่ง response กลับมาเรื่อย ๆ ทาง stream ของตัวเอง (ในตัวอย่างนี้คือ "ยืนยัน" ทุกอัปเดตที่รับมา — ในระบบจริงคือจุดที่ server จะ broadcast ไปยัง client รายอื่นที่กำลัง watch เล่มเดียวกันอยู่พอดี) **ทั้งสอง stream เป็นอิสระจากกันโดยสิ้นเชิง** ไม่ต้องรอกัน — นี่คือสิ่งที่ server streaming เพียงอย่างเดียวทำไม่ได้ (server streaming มีแค่ทิศทางเดียว)

**ฝั่ง Server** (สาธิตด้วยข้อมูลจริง ไม่ใช่แค่ sketch — รันและ verify แล้ว):

```rust
type WatchAvailabilityStream = Pin<Box<dyn Stream<Item = Result<AvailabilityUpdate, Status>> + Send + 'static>>;

async fn watch_availability(
    &self,
    request: Request<Streaming<AvailabilityUpdate>>,
) -> Result<Response<Self::WatchAvailabilityStream>, Status> {
    let mut in_stream = request.into_inner();
    let (tx, rx) = tokio::sync::mpsc::channel(4);

    tokio::spawn(async move {
        while let Some(result) = in_stream.message().await.transpose() {
            match result {
                Ok(update) => {
                    println!(
                        "[server] WatchAvailability รับอัปเดตจาก client: book_id={} copies={}",
                        update.book_id, update.copies_available
                    );
                    let ack = AvailabilityUpdate {
                        book_id: update.book_id,
                        copies_available: update.copies_available,
                    };
                    if tx.send(Ok(ack)).await.is_err() {
                        return;
                    }
                }
                Err(status) => {
                    let _ = tx.send(Err(status)).await;
                    return;
                }
            }
        }
        println!("[server] WatchAvailability: client ปิด stream ฝั่งส่งแล้ว");
    });

    let output_stream = tokio_stream::wrappers::ReceiverStream::new(rx);
    Ok(Response::new(Box::pin(output_stream) as Self::WatchAvailabilityStream))
}
```

สังเกตว่า signature นี้คือ**การรวมกันของทั้งสองแบบก่อนหน้า**: input เป็น `Streaming<T>` (เหมือน client streaming) และ output เป็น `Result<Response<Self::WatchAvailabilityStream>, Status>` (เหมือน server streaming) — โครงสร้างข้างในคือ `tokio::spawn` งานที่ loop อ่าน `in_stream` ไปเรื่อย ๆ แล้ว `.send()` ผลลัพธ์เข้า `tx` (ฝั่งขาออก) ทุกครั้งที่ได้รับข้อความใหม่ ทั้งหมดนี้เกิดขึ้นแบบ concurrent กับที่ Tonic framework กำลัง poll `rx` ส่งกลับไปยัง client อยู่พร้อมกัน

**ฝั่ง Client** (ใช้ `async_stream::stream!` macro เพื่อสร้าง outbound stream ที่ทยอยส่งค่าออกไปพร้อม `sleep` ระหว่างกลาง จำลองว่าอัปเดตมาไม่พร้อมกันจริง ๆ):

```rust
let outbound_updates = async_stream::stream! {
    let updates = vec![
        AvailabilityUpdate { book_id: 1, copies_available: 5 },
        AvailabilityUpdate { book_id: 2, copies_available: 0 },
        AvailabilityUpdate { book_id: 1, copies_available: 4 },
    ];
    for u in updates {
        tokio::time::sleep(std::time::Duration::from_millis(100)).await;
        yield u;
    }
};
let response = client.watch_availability(outbound_updates).await?;
let mut inbound = response.into_inner();
while let Some(update) = inbound.message().await? {
    println!("  ack from server -> book_id={} copies_available={}", update.book_id, update.copies_available);
}
```

**Output จริงจากการรัน (ทั้งสองฝั่ง verify ครบ ไม่ใช่ sketch)**:

```text
=== 8) Bidirectional streaming: WatchAvailability ===
  ack from server -> book_id=1 copies_available=5
  ack from server -> book_id=2 copies_available=0
  ack from server -> book_id=1 copies_available=4
  (bidirectional stream จบทั้งสองฝั่ง)
```

ฝั่ง server log:

```text
[server] WatchAvailability รับอัปเดตจาก client: book_id=1 copies=5
[server] WatchAvailability รับอัปเดตจาก client: book_id=2 copies=0
[server] WatchAvailability รับอัปเดตจาก client: book_id=1 copies=4
[server] WatchAvailability: client ปิด stream ฝั่งส่งแล้ว
```

**สรุปสถานะการ verify ของบทนี้อย่างตรงไปตรงมา**: ทั้ง 4 รูปแบบการเรียก (unary, server streaming, client streaming, **และ bidirectional streaming**) ถูก implement, compile, รันจริง และ capture output จริงครบทั้งหมดในบทนี้ — ไม่มีรูปแบบใดที่เป็นแค่ sketch/pseudo-code ทฤษฎีเปล่า ๆ

### 80.7 Error Handling: `tonic::Status`

#### `tonic::Status` และ Status Code มาตรฐาน

gRPC ไม่ใช้ HTTP status code (`200`, `404`, `500` แบบที่ Part 61 สอน) เป็นตัวบอกผลลัพธ์ของ RPC โดยตรง — มันมี**status code ของตัวเอง** (ตัวเลข 0-16) ที่ส่งผ่าน gRPC-specific header ชื่อ `grpc-status` (และข้อความอธิบายผ่าน `grpc-message`) ซ้อนอยู่**ภายใน**ชั้น HTTP/2 อีกที — นี่คือข้อเท็จจริงที่สำคัญมากและมักสร้างความสับสน: **การเรียก gRPC ที่ "fail" ในความหมายของ business logic (เช่น หา resource ไม่เจอ) มักได้ HTTP/2 status `200 OK` เสมอ** ที่ระดับ transport เพราะ "การส่ง response กลับมาสำเร็จ" (ชั้น HTTP/2) กับ "RPC ทำสิ่งที่ขอสำเร็จหรือไม่" (ชั้น gRPC ที่ซ้อนอยู่ข้างใน) เป็นคนละเรื่องกันโดยสิ้นเชิงในสถาปัตยกรรมของ gRPC

เราพิสูจน์ข้อเท็จจริงนี้ได้จริงด้วย `curl` (ที่รองรับ HTTP/2 ผ่าน flag `--http2-prior-knowledge`) ยิงตรงไปที่ gRPC server ที่รันอยู่ในบทนี้ (เรียก `GetBook` โดยไม่แนบ metadata ที่ interceptor ต้องการ — ดูหัวข้อ 80.8):

```text
$ curl -v --http2-prior-knowledge http://127.0.0.1:50051/library.v1.BookService/GetBook -d ''
> POST /library.v1.BookService/GetBook HTTP/2
> content-type: application/x-www-form-urlencoded
>
< HTTP/2 200
< content-type: application/grpc
< grpc-status: 16
< grpc-message: %E0%B9%84%E0%B8%A1%E0%B9%88%E0%B8%9E%E0%B8%9A%20authorization%20metadata
< date: Sun, 27 Sep 2026 01:24:45 GMT
```

สังเกตทุกจุดในผลลัพธ์จริงนี้:

- **`HTTP/2 200`**: แม้ RPC นี้ถูกปฏิเสธจริง (เพราะ interceptor บล็อกไว้) แต่ status ที่ระดับ HTTP/2 ยังเป็น `200 OK` เพราะ Tonic "ส่ง response header กลับมาได้สำเร็จ" — request นี้ไม่ใช่ error ที่ระดับ transport แต่อย่างใด
- **`grpc-status: 16`**: นี่คือ**ตัวเลขจริง**ที่บอกผลลัพธ์ของ RPC — เลข `16` ตรงกับ `Code::Unauthenticated` (ตรวจสอบตรงจาก source ของ `tonic::Code` — ดูตารางแบบเต็มด้านล่าง)
- **`grpc-message: %E0%B9%84%E0%B8%A1...`**: ข้อความ error ภาษาไทย ("ไม่พบ authorization metadata") ที่ถูก **percent-encode** (เหมือน URL encoding) ตามข้อกำหนดของ gRPC-over-HTTP/2 spec เพราะ HTTP/2 header value ต้องเป็น ASCII เท่านั้น ข้อความที่มี byte นอก ASCII (เช่น UTF-8 ภาษาไทย) จึงต้องถูกเข้ารหัสก่อนแนบไปเป็น header เสมอ — Tonic ทำให้อัตโนมัติทั้ง encode (ฝั่งส่ง) และ decode (ฝั่งรับ ที่เราเห็นเป็นข้อความไทยปกติตอน print `status.message()` ในหัวข้อก่อน)

ตารางเทียบ `tonic::Code` ทั้งหมด (ตรวจสอบค่าตัวเลขจริงจาก source ของ `tonic` 0.14.6) กับ HTTP status code ที่ใกล้เคียงที่สุดจาก Part 61 (การเทียบนี้เป็น**แนวทางที่นิยมใช้กัน**เมื่อต้องแปลงระหว่างสองโลก เช่นเวลาทำ REST-to-gRPC gateway ไม่ใช่กฎที่ gRPC บังคับเอง — gRPC เองไม่ใช้ HTTP status code เลยตามที่อธิบายไปแล้ว):

| gRPC `Code` | เลข | ความหมาย | HTTP status ที่ใกล้เคียง (Part 61) | สถานการณ์ในระบบห้องสมุด |
|---|---|---|---|---|
| `Ok` | 0 | สำเร็จ | 200 | RPC ทำงานถูกต้องตามที่ขอ |
| `Cancelled` | 1 | client ยกเลิก RPC เอง | 499 (ไม่เป็นทางการ) | client ปิด connection ก่อน server ตอบ |
| `Unknown` | 2 | error ที่ไม่รู้สาเหตุ | 500 | handler panic โดยไม่คาดคิด |
| `InvalidArgument` | 3 | request ผิด syntax/ค่าไม่ถูกต้อง | 400 | `CreateBook` ที่ `price` ติดลบ |
| `DeadlineExceeded` | 4 | เกิน timeout ที่กำหนด | 504 | RPC ใช้เวลานานเกินที่ client ตั้ง deadline ไว้ |
| `NotFound` | 5 | ไม่พบ resource | 404 | `GetBook(id=999)` ที่ไม่มีอยู่จริง |
| `AlreadyExists` | 6 | resource มีอยู่แล้ว | 409 | สร้างหนังสือด้วย ISBN ที่ซ้ำ (ในระบบจริงที่มี unique constraint) |
| `PermissionDenied` | 7 | รู้ตัวตนแล้วแต่ไม่มีสิทธิ์ | 403 | `CreateBook` ด้วยชื่อที่ถูก policy บล็อก |
| `ResourceExhausted` | 8 | เกิน quota/rate limit | 429 | ยิง RPC ถี่เกินกำหนดที่ server รับได้ |
| `FailedPrecondition` | 9 | ระบบไม่พร้อมสำหรับ operation นี้ | 400/409 | ยืมหนังสือที่ถูก reserve ไว้แล้ว |
| `Aborted` | 10 | operation ถูกยกเลิกกลางทาง (มักเจอกับ concurrency conflict) | 409 | transaction ชนกันแบบ optimistic lock |
| `OutOfRange` | 11 | ค่า/ตำแหน่งเกินขอบเขตที่ยอมรับได้ | 400 | ขอ `page_size` เป็นค่าลบ |
| `Unimplemented` | 12 | RPC นี้ไม่ได้ implement ไว้ | 501 | เรียก method ที่ยังไม่มีในเวอร์ชันนี้ |
| `Internal` | 13 | bug ภายใน server | 500 | unwrap บน `None` โดยไม่ตั้งใจ (คล้าย Part 12) |
| `Unavailable` | 14 | server ไม่พร้อมให้บริการชั่วคราว | 503 | service ปิดปรับปรุง/overload |
| `DataLoss` | 15 | ข้อมูลเสียหาย/สูญหายที่กู้คืนไม่ได้ | 500 | ข้อมูลใน stream ถูก corrupt กลางทาง |
| `Unauthenticated` | 16 | ไม่รู้ตัวตนของผู้เรียก | 401 | ไม่แนบ token หรือ token ผิด (ตัวอย่างจริงข้างบน) |

#### แนบ Error Details และ Metadata เพิ่มเติม

บางสถานการณ์ต้องการมากกว่าแค่ `code` + `message` — เช่นบอกว่า "นโยบายไหน" ที่ทำให้ปฏิเสธ request เพื่อให้ client (หรือทีม support) ตรวจสอบย้อนหลังได้ `tonic::Status` มีสองช่องทางสำหรับข้อมูลเสริมนี้:

```rust
if req.title == "Banned Book" {
    let mut metadata = tonic::metadata::MetadataMap::new();
    metadata.insert("x-error-reason", "title-blocklisted".parse().unwrap());
    return Err(Status::with_details_and_metadata(
        tonic::Code::PermissionDenied,
        "หนังสือชื่อนี้ถูกระงับการเพิ่มโดยนโยบายเนื้อหา",
        bytes::Bytes::from_static(b"policy_id=CONTENT_BLOCKLIST_7"),
        metadata,
    ));
}
```

- **`details`** (พารามิเตอร์ที่ 3): ช่องเก็บ **bytes ดิบ** — ระบบจริงที่ทำตาม [richer error model](https://grpc.io/docs/guides/error/) ของ gRPC มักเข้ารหัสข้อมูลตรงนี้เป็น Protobuf ของ `google.rpc.ErrorInfo`/`google.rpc.BadRequest` (มี schema มาตรฐานให้ client ทุกภาษาถอดรหัสได้เหมือนกัน) — ตัวอย่างในบทนี้ใส่แค่ string ธรรมดาเพื่อความง่าย แต่หลักการเดียวกัน
- **`metadata`** (พารามิเตอร์ที่ 4): เหมือน HTTP header เพิ่มเติมที่แนบไปกับ error response — ผู้เรียกอ่านได้ผ่าน `status.metadata()`

ฝั่ง client อ่านค่าทั้งสองกลับออกมาได้:

```rust
Err(status) => {
    println!("gRPC error: code={:?} message={:?}", status.code(), status.message());
    println!("  details (raw bytes as utf8): {:?}", String::from_utf8_lossy(status.details()));
    if let Some(reason) = status.metadata().get("x-error-reason") {
        println!("  metadata x-error-reason = {:?}", reason.to_str().unwrap());
    }
}
```

**Output จริงจากการรัน:**

```text
=== 5b) Unary: CreateBook ด้วยชื่อที่ถูกแบน — ดู error details/metadata เพิ่มเติม ===
gRPC error: code=PermissionDenied message="หน\u{e31}งส\u{e37}อช\u{e37}\u{e48}อน\u{e35}\u{e49}ถ\u{e39}กระง\u{e31}บการเพ\u{e34}\u{e48}มโดยนโยบายเน\u{e37}\u{e49}อหา"
  details (raw bytes as utf8): "policy_id=CONTENT_BLOCKLIST_7"
  metadata x-error-reason = "title-blocklisted"
```

### 80.8 Interceptor: Middleware ของ Tonic

#### ข้อเท็จจริงเชิงสถาปัตยกรรมที่สำคัญ: Tonic สร้างอยู่บน `tower` เหมือน Axum

**Part 68** สรุปไว้ว่า Axum กับ Actix-web มีระบบ middleware คนละแบบเพราะ Axum สร้างอยู่บน `tower` (ecosystem กลางของ Tokio สำหรับ request/response middleware ที่เป็นกลางไม่ผูกกับ protocol เฉพาะ) ส่วน Actix-web มีระบบ `Service`/`Transform` ของตัวเอง — ข้อเท็จจริงที่น่าสนใจมากซึ่งตรวจสอบได้จริงจาก `Cargo.toml` ของ `tonic` เองคือ **`tonic` ก็ depend on `tower`, `tower-layer`, และ `tower-service` โดยตรง** เพราะ Tonic เป็นโปรเจกต์ของทีม Tokio เหมือนกับ Axum — นี่แปลว่า**สถาปัตยกรรมของ Axum และ Tonic ใกล้กันกว่า Axum และ Actix-web มาก** แม้ Axum จะเน้น HTTP/REST ส่วน Tonic เน้น gRPC ก็ตาม (คนละ layer ของ "สิ่งที่โฟกัส" แต่ใช้ foundation เดียวกัน)

อย่างไรก็ตาม Tonic มีระบบ middleware แบบง่ายของตัวเองด้วยชื่อ **`tonic::service::Interceptor`** — สาเหตุที่มีอยู่คู่กับ `tower::Layer` คือ interceptor ถูกออกแบบมาให้ใช้งาน**ง่ายกว่า**สำหรับ use case ที่พบบ่อยที่สุด (ตรวจสอบ/แก้ไข request ก่อนส่งเข้า handler) โดยไม่ต้องเขียน `tower::Layer` เต็มรูปแบบ — เอกสารต้นฉบับของ Tonic (`src/service/interceptor.rs` ที่ตรวจสอบจริง) บอกไว้ตรง ๆ ว่า:

> "If you need more powerful middleware, tower is the recommended approach ... interceptors is not the recommended way to add logging to your service. For that a tower middleware is more appropriate since it can also act on the response."

พูดให้ชัด: **`Interceptor` ทำได้แค่ตรวจสอบ/ปฏิเสธ request ขาเข้าเท่านั้น (ไม่เห็น response ขาออกเลย)** ถ้าต้องการ middleware ที่ทำอะไรกับ response ด้วย (เช่น logging เวลาที่ใช้ทั้ง request/response, การแก้ response header) ต้องเขียนเป็น `tower::Layer` เต็มรูปแบบ (แนวคิดเดียวกับ `tower::Layer` ที่ Axum middleware ใน Part 65 ใช้อยู่แล้ว) — บทนี้จะสาธิตแค่ `Interceptor` เพราะโจทย์ (auth check ผ่าน metadata) ตรงกับสิ่งที่มันถูกออกแบบมาให้ทำพอดี

#### ข้อเท็จจริงที่ลึกกว่านั้นอีกขั้น: `tonic::transport::Server` ใช้ `axum::Router` เป็นตัว Route ภายในจริง ๆ

ตรวจสอบ source code ของ `tonic` 0.14.6 ลึกลงไปอีกชั้นพบข้อเท็จจริงที่คอนกรีตกว่าคำว่า "ทั้งคู่ใช้ tower" มาก: ฟีเจอร์ `router` ของ `tonic` (เปิดอยู่โดย default ผ่าน `default = ["router", "transport", "codegen"]` ใน `Cargo.toml` ของมันเอง) ประกาศ dependency ตรง ๆ ไปที่ **`axum`** และไฟล์ `src/service/router.rs` ของ Tonic ก็มี struct ที่เก็บ `router: axum::Router` อยู่ข้างในจริง ๆ — พูดให้ชัดที่สุด: **เวลาคุณเรียก `Server::builder().add_service(...).add_service(...)` เพื่อผูกหลาย gRPC service เข้ากับ server ตัวเดียว เบื้องหลัง Tonic กำลังประกอบ `axum::Router` ตัวหนึ่งขึ้นมาเพื่อ route request ไปยัง service ที่ถูกต้องตาม path (`/library.v1.BookService/GetBook` ก็คือ "path" หนึ่งในความหมายของ Axum routing ตามที่ Part 62-63 สอนไว้ทุกประการ)** — Tonic เปิดเผยความจริงนี้ตรง ๆ ด้วยซ้ำผ่านเมธอด `Routes::into_axum_router()` ที่แปลง route ของ gRPC service ให้กลายเป็น `axum::Router` ธรรมดาที่หยิบไป `.merge()` เข้ากับ Axum app ที่มี REST endpoint อยู่แล้วได้เลย (เทคนิคที่ใช้กันจริงเวลาต้องการรัน gRPC และ REST คู่กันบน port เดียวกัน — หัวข้อที่ Part 81 จะพูดถึงเพิ่มเติมในบริบท microservices) ข้อเท็จจริงนี้ยืนยันชัดกว่าที่ Part 68 เคยตั้งข้อสังเกตไว้อีกขั้น: Axum และ Tonic ไม่ได้แค่ "อยู่บน foundation เดียวกัน" (tower) แต่ Tonic เวอร์ชันปัจจุบัน**ใช้ Axum เป็นส่วนประกอบภายในตรง ๆ**เลย

#### เขียน Auth Interceptor จริงด้วย Bearer Token ผ่าน gRPC Metadata

Trait `Interceptor` มีเมธอดเดียว: `fn call(&mut self, request: Request<()>) -> Result<Request<()>, Status>` — และ Tonic implement trait นี้ให้กับ**ทุกฟังก์ชัน**ที่มี signature ตรงกันโดยอัตโนมัติ (ผ่าน blanket impl) แปลว่าเขียนเป็นฟังก์ชันธรรมดาได้เลยโดยไม่ต้องประกาศ `struct`/`impl Interceptor for ...` ถ้า logic ไม่ต้องเก็บ state:

```rust
const VALID_TOKEN: &str = "Bearer secret-token-123";

fn check_auth(req: Request<()>) -> Result<Request<()>, Status> {
    match req.metadata().get("authorization") {
        Some(value) if value.to_str().unwrap_or("") == VALID_TOKEN => Ok(req),
        Some(_) => Err(Status::unauthenticated("token ไม่ถูกต้อง")),
        None => Err(Status::unauthenticated("ไม่พบ authorization metadata")),
    }
}
```

ผูก interceptor เข้ากับ service ตอนสร้าง server ด้วย `.with_interceptor(...)` (เมธอดที่เห็นในโค้ด generate จากหัวข้อ 80.3):

```rust
Server::builder()
    .add_service(BookServiceServer::with_interceptor(book_service, check_auth))
    .serve(addr)
    .await?;
```

ฝั่ง client แนบ metadata ก่อนส่งทุก request ที่ต้องผ่าน auth:

```rust
fn with_auth<T>(msg: T) -> Request<T> {
    let mut req = Request::new(msg);
    req.metadata_mut()
        .insert("authorization", "Bearer secret-token-123".parse().unwrap());
    req
}

let resp = client.get_book(with_auth(GetBookRequest { id: 1 })).await?;
```

**gRPC metadata** ทำหน้าที่เดียวกับ HTTP header ทุกประการในเชิงความหมาย (key-value ที่แนบไปกับ request/response แต่ไม่ใช่ตัว payload) — Tonic ใช้ `MetadataMap` เป็น type ของมันแทน `http::HeaderMap` ตรง ๆ (เพราะ metadata ของ gRPC มีข้อจำกัดเรื่อง encoding ที่ต่างจาก HTTP header ทั่วไปนิดหน่อย เช่น key ที่ลงท้ายด้วย `-bin` ต้องเป็น binary-safe value) แต่ใช้งานคล้ายกันมากในทางปฏิบัติ

**เชื่อมกับ Part 74 (JWT Authentication)**: ตัวอย่างในบทนี้เทียบ token กับค่า static string ตรง ๆ เพื่อโฟกัสที่กลไกของ interceptor เอง — ระบบจริงจะแทนที่การเทียบ `==` ตรงนี้ด้วยการเรียก `jsonwebtoken::decode::<Claims>(&token, &decoding_key, &validation)` แบบเดียวกับที่ Part 74 สอนไว้เต็มรูปแบบ (verify signature, ตรวจ `exp`, ดึง claims ออกมาเพื่อรู้ว่า "ใคร" เรียกอยู่) แล้วแนบ claims ที่ decode ได้เข้าไปกับ `Request` ต่อ (ผ่าน `request.extensions_mut().insert(claims)`) ให้ handler เรียกดูได้ทีหลัง — ส่วนที่**ต่างจาก HTTP `Authorization` header**มีแค่จุดเดียวคือ**วิธีอ่านค่าเข้ามา** (`req.metadata().get("authorization")` แทน `headers.get("authorization")`) ตรรกะการ verify JWT ที่เหลือทั้งหมดเหมือนกันทุกประการกับที่ Part 74 สอนไว้

**Output จริงพิสูจน์ว่า interceptor ทำงาน (ทั้งกรณีผ่านและถูกบล็อก):**

```text
=== 1) Unary: GetBook(id=1) พร้อม token ที่ถูกต้อง ===
ได้รับ: Book { id: 1, title: "The Rust Programming Language", ... }

=== 3) Unary: GetBook(id=1) โดยไม่แนบ token — ควรถูก interceptor บล็อก ===
gRPC error: code=Unauthenticated message="ไม\u{e48}พบ authorization metadata"
```

สังเกตว่า request ที่ 3 **ไม่ได้ไปถึง `MyBookService::get_book` เลยแม้แต่บรรทัดเดียว** — `check_auth` ปฏิเสธไว้ก่อนที่ trait method ของ business logic จะถูกเรียกด้วยซ้ำ (พิสูจน์ได้จาก server log ที่ไม่มี `[server] GetBook id=1` ตัวที่สองปรากฏขึ้นเลยตอนรัน request ที่ไม่มี token) — นี่คือคุณค่าหลักของ middleware/interceptor pattern: **แยก concern เรื่อง auth ออกจาก business logic โดยสมบูรณ์** เหมือนกับที่ Part 65 (Axum Middleware) สอนไว้ในบริบทของ REST

### 80.9 Reflection และเครื่องมือ: `grpcurl` และ `tonic-reflection`

#### `grpcurl`: `curl` เวอร์ชันสำหรับ gRPC

**`grpcurl`** เป็นเครื่องมือ command-line ที่ทำหน้าที่เดียวกับ `curl` แต่คุยด้วย gRPC โดยเฉพาะ — รับ argument เป็น JSON แทน raw bytes (แปลง JSON เป็น Protobuf binary ให้อัตโนมัติก่อนส่ง) วิธีใช้งานทั่วไป (syntax มาตรฐานของเครื่องมือนี้):

```bash
# เรียก unary RPC ธรรมดา (-plaintext เพราะไม่ได้เปิด TLS ในเดโมนี้)
grpcurl -plaintext -d '{"id": 1}' 127.0.0.1:50051 library.v1.BookService/GetBook

# แนบ metadata (เทียบเท่า -H ของ curl)
grpcurl -plaintext -H 'authorization: Bearer secret-token-123' \
  -d '{"id": 1}' 127.0.0.1:50051 library.v1.BookService/GetBook

# ให้ grpcurl ค้นหา service ทั้งหมดผ่าน reflection โดยไม่ต้องมีไฟล์ .proto อยู่ในเครื่อง
grpcurl -plaintext 127.0.0.1:50051 list
```

**หมายเหตุความซื่อสัตย์ที่สำคัญ**: สภาพแวดล้อมที่ใช้เขียนและ verify บทนี้**ไม่มี `grpcurl` ติดตั้งมาให้** (ตรวจสอบด้วย `which grpcurl` แล้วไม่พบ) จึงไม่สามารถรันคำสั่งข้างบนเพื่อ capture output จริงได้ — คำสั่งข้างบนเขียนตาม syntax มาตรฐานที่ documented ไว้อย่างกว้างขวางในเอกสารของ `grpcurl` เอง (ถูกต้องตามหลักการแน่นอน) แต่**ไม่ได้ verify ด้วยการรันจริงในบทนี้** — สิ่งที่ verify ได้จริงแทนคือการเขียน**gRPC client เองด้วย Tonic** (หัวข้อ 80.5) และ**client ทดสอบ reflection service ตรง ๆ** (ด้านล่าง) ซึ่งพิสูจน์ผลลัพธ์เดียวกันในเชิงหน้าที่: ทั้งสองคือ "โปรแกรมภายนอกที่คุยกับ gRPC server ผ่าน network จริง"

#### `tonic-reflection`: ให้เครื่องมืออย่าง `grpcurl` รู้จัก Schema โดยไม่ต้องมีไฟล์ `.proto`

ปกติแล้ว `grpcurl`/Postman ต้องมีไฟล์ `.proto` (หรือ compiled descriptor) อยู่ในเครื่องก่อนจึงจะรู้ว่า service มี RPC อะไรบ้าง รับ/คืนอะไร — **Server Reflection** คือ feature ที่ให้ gRPC server "รายงานตัวเอง" ว่ามี service/message อะไรบ้างผ่าน RPC พิเศษ (`grpc.reflection.v1.ServerReflection`) ทำให้เครื่องมือฝั่ง client**ค้นพบ schema เองได้แบบ dynamic** โดยไม่ต้องแจก `.proto` แยกให้ทุกคนที่ต้องการทดสอบ (คล้ายกับที่ GraphQL introspection ใน Part 79 ให้ client ถาม schema ของตัวเองได้ — แนวคิดเดียวกัน ต่างแค่ protocol)

เปิดใช้งานด้วย crate `tonic-reflection` โดยใช้ไฟล์ `FileDescriptorSet` ที่เรา build ไว้แล้วในหัวข้อ 80.3:

```rust
let reflection_service = tonic_reflection::server::Builder::configure()
    .register_encoded_file_descriptor_set(grpc_demo::book::FILE_DESCRIPTOR_SET)
    .build_v1()?;

Server::builder()
    .add_service(reflection_service)
    .add_service(BookServiceServer::with_interceptor(book_service, check_auth))
    .serve(addr)
    .await?;
```

`.build_v1()` สร้าง reflection service ตาม protocol เวอร์ชัน `v1` (เวอร์ชันปัจจุบันที่เพิ่งกลายเป็น stable — ก่อนหน้านี้มีแค่ `v1alpha` ซึ่ง `tonic-reflection` ยังรองรับคู่กันไว้ผ่าน `.build_v1alpha()` สำหรับ client เก่าที่ยังไม่รองรับ `v1`)

#### Verify Reflection จริงด้วย Client ที่เขียนเอง (แทน `grpcurl`)

เพราะไม่มี `grpcurl` เราเขียน client เล็ก ๆ ที่คุยกับ `grpc.reflection.v1.ServerReflection` service ตรง ๆ (types ที่ต้องใช้ถูก export ไว้ที่ `tonic_reflection::pb::v1`):

```rust
use tonic::Request;
use tonic_reflection::pb::v1::server_reflection_client::ServerReflectionClient;
use tonic_reflection::pb::v1::server_reflection_request::MessageRequest;
use tonic_reflection::pb::v1::server_reflection_response::MessageResponse;
use tonic_reflection::pb::v1::ServerReflectionRequest;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let channel = tonic::transport::Channel::from_static("http://127.0.0.1:50051")
        .connect()
        .await?;
    let mut client = ServerReflectionClient::new(channel);

    let request = ServerReflectionRequest {
        host: String::new(),
        message_request: Some(MessageRequest::ListServices(String::new())),
    };
    let outbound = tokio_stream::once(request);
    let mut inbound = client
        .server_reflection_info(Request::new(outbound))
        .await?
        .into_inner();

    if let Some(response) = inbound.message().await? {
        if let Some(MessageResponse::ListServicesResponse(list)) = response.message_response {
            println!("services ที่ reflection service รายงาน:");
            for s in list.service {
                println!("  - {}", s.name);
            }
        }
    }
    Ok(())
}
```

สังเกตว่า `ServerReflectionInfo` เป็น **bidirectional streaming RPC** เอง (ตรงตามหัวข้อ 80.6.4) — เราส่ง request แค่ชิ้นเดียวผ่าน `tokio_stream::once(...)` (stream ที่มีค่าเดียวแล้วปิด) เพราะกรณีนี้ต้องการแค่ถามคำถามเดียว (`ListServices`) แต่ protocol ออกแบบให้เป็น stream เพื่อรองรับการถามหลายคำถามต่อเนื่องบน connection เดียวได้ (เช่น ถาม `ListServices` แล้วต่อด้วย `file_containing_symbol` เพื่อดึง schema แบบละเอียดของแต่ละ service)

**Output จริงจากการรัน** (พิสูจน์ว่า reflection service ทำงานได้จริง โดยไม่ต้องมีไฟล์ `.proto` อยู่ในโค้ด client เลยแม้แต่บรรทัดเดียว — client รู้จัก `library.v1.BookService` ผ่าน reflection ล้วน ๆ):

```text
services ที่ reflection service รายงาน:
  - library.v1.BookService
  - grpc.reflection.v1.ServerReflection
```

สังเกตว่า reflection service เห็น**ตัวเองด้วย** (`grpc.reflection.v1.ServerReflection`) เพราะมันก็ถูก register เป็น service ปกติตัวหนึ่งบน `Server` เดียวกัน

### 80.10 gRPC vs REST vs GraphQL: เลือกใช้เมื่อไหร่

ถึงจุดนี้คุณได้เรียนครบทั้งสามแนวทางหลักในการสร้าง API ด้วย Rust แล้ว (REST+Axum จาก Part 62-66, GraphQL จาก Part 79, และ gRPC จากบทนี้) — มาสรุปเป็นตารางเทียบสามทางเพื่อใช้ตัดสินใจในโปรเจกต์จริง:

| มิติ | REST (Axum, Part 62-66) | GraphQL (Part 79) | gRPC (Tonic, บทนี้) |
|---|---|---|---|
| Wire format | JSON (text) | JSON (text) | Protocol Buffers (binary) |
| Transport | HTTP/1.1 หรือ HTTP/2 | HTTP/1.1 หรือ HTTP/2 | **HTTP/2 เท่านั้น** |
| Schema strictness | อ่อน (schema อยู่แค่ในเอกสาร/OpenAPI ที่แยกจาก runtime) | เข้ม (schema บังคับผ่าน GraphQL type system, ตรวจสอบตอน query) | **เข้มที่สุด** (schema คือ `.proto` ที่ compiler ตรวจสอบให้ทั้งสองฝั่งตอน build time) |
| ประสิทธิภาพ (ขนาดข้อมูล/ความเร็ว) | กลาง | กลาง (มักดีกว่า REST เรื่อง over-fetching แต่ยังเป็น JSON) | **ดีที่สุด** (binary + HTTP/2 multiplexing) |
| Debuggability ด้วยตาเปล่า | **ดีที่สุด** (`curl` + อ่าน JSON ได้ทันที) | ดี (มี GraphiQL/Playground ให้ทดลอง query) | แย่ที่สุด (ต้องใช้ `grpcurl`/reflection หรือ decode ด้วยมือ) |
| ใช้จาก Browser ได้ตรง ๆ ไหม | **ได้** (fetch/XHR ธรรมดา) | **ได้** (fetch/XHR ธรรมดา) | **ไม่ได้โดยตรง** (ดูรายละเอียดด้านล่าง) |
| Client กำหนด "รูปร่าง" ของข้อมูลที่ต้องการได้ไหม | ไม่ได้ (server กำหนด response shape ตายตัวต่อ endpoint) | **ได้** (จุดเด่นหลักของ GraphQL) | ไม่ได้ (server กำหนด message shape ตายตัวเหมือน REST) |
| เหมาะกับ | Public API, client ภายนอกที่ไม่รู้จักล่วงหน้า | Client ที่ต้องการความยืดหยุ่นสูงในการเลือกข้อมูล (เช่น mobile app หลายเวอร์ชันที่ต้องการข้อมูลต่างกัน) | **Service-to-service ภายในระบบที่ควบคุมทั้งสองฝั่ง** |

#### ข้อจำกัดที่ต้องพูดตรง ๆ: Browser คุย gRPC ตรง ๆ ไม่ได้

นี่คือข้อเท็จจริงเชิงเทคนิคที่สำคัญมากและมักสร้างความสับสนให้คนที่เพิ่งเรียน gRPC — **เว็บเบราว์เซอร์ไม่มี API ระดับ JavaScript ที่เข้าถึง HTTP/2 frame ระดับต่ำได้** (`fetch`/`XMLHttpRequest` ทำงานอยู่เหนือ HTTP abstraction ที่เบราว์เซอร์จัดการให้ ไม่ให้ JavaScript ควบคุม trailer, stream ID, หรือฟีเจอร์ HTTP/2 ระดับ frame ที่ gRPC protocol ต้องใช้) เราพิสูจน์ผลที่ตามมาได้จริงด้วยการยิง `curl` แบบ HTTP/1.1 ธรรมดา (ซึ่งพฤติกรรมเหมือน browser ที่คุยแบบ HTTP/1.1 ทุกประการ) ไปยัง gRPC server เดียวกัน:

```text
$ curl -v http://127.0.0.1:50051/library.v1.BookService/GetBook
> GET /library.v1.BookService/GetBook HTTP/1.1
* Received HTTP/0.9 when not allowed
curl: (1) Received HTTP/0.9 when not allowed
```

server (ที่สร้างด้วย Tonic ซึ่งพูด HTTP/2 เท่านั้น) ไม่เข้าใจ HTTP/1.1 request line ที่ curl ส่งมาเลย — พยายาม parse มันตามกฎ HTTP/2 binary framing แล้วล้มเหลวโดยสิ้นเชิง (`curl` ตีความผลลัพธ์ที่งงงวยนี้ว่าเป็น "HTTP/0.9" ซึ่งเป็นการเดาของ `curl` เอง ไม่ใช่ protocol ที่ server ตั้งใจส่งจริง ๆ) — นี่คือหลักฐานที่จับต้องได้ว่า**gRPC (แบบดั้งเดิม) ใช้ตรง ๆ จาก browser ไม่ได้เลย**

ทางออกมาตรฐานสำหรับปัญหานี้คือ **gRPC-Web** — protocol variant ที่ทีม gRPC ออกแบบมาให้ทำงานผ่าน HTTP/1.1 ได้ (encode stream/trailer ผ่านกลไกที่ HTTP/1.1 รองรับ) แต่ต้องผ่าน**proxy พิเศษ** (เช่น Envoy ที่มี gRPC-Web filter) คั่นระหว่าง browser กับ gRPC server ตัวจริง เพราะ gRPC server (รวมถึง Tonic) ยังคุยแค่ gRPC โปรโตคอลมาตรฐานเท่านั้น ไม่ได้แปลง gRPC-Web ให้อัตโนมัติ — นี่คือ**ต้นทุนเพิ่มเติมทางสถาปัตยกรรม**ที่ REST/GraphQL ไม่ต้องมี (เพราะทั้งสองใช้ HTTP ธรรมดาที่ browser คุยตรงได้อยู่แล้ว) และเป็นเหตุผลสำคัญอีกข้อที่ยืนยันจุดยืนของบทนี้: **gRPC เหมาะกับ service-to-service ที่ทั้งสองฝั่งเป็นโปรแกรม backend ที่คุณควบคุมเอง ไม่ใช่ API ที่ browser จะเรียกตรง ๆ**

#### จุดยืนของหลักสูตรนี้ (และการเชื่อมไปสู่ Part 81)

สรุปให้ชัดเจนที่สุด: **REST ด้วย Axum ยังคงเป็นแนวทางหลักของหลักสูตรนี้สำหรับ API ที่ client ภายนอก (เว็บ/มือถือ) เรียกใช้** — GraphQL (Part 79) เป็นตัวเลือกเสริมเมื่อความยืดหยุ่นของ client สำคัญกว่าความเรียบง่าย ส่วน **gRPC (บทนี้) คือเครื่องมือสำหรับการสื่อสารระหว่าง service ภายในระบบเดียวกัน** — สถานการณ์ที่จะกลายเป็นหัวใจของ **Part 81 (Microservices Architecture)**: เมื่อระบบถูกแตกเป็นหลาย service (เช่น `order-service`, `inventory-service`, `notification-service`) การสื่อสารระหว่างพวกมันเองคือจุดที่ gRPC (schema strict, ประสิทธิภาพสูง, ทั้งสองฝั่งควบคุมได้) เข้ามาแทนที่ REST ได้อย่างเป็นธรรมชาติ ในขณะที่ REST ยังคงทำหน้าที่เป็น "หน้าบ้าน" (API gateway) ที่ client ภายนอกเรียกเข้ามาเหมือนเดิม

### 80.11 ข้อพิจารณาสำหรับ Production (เกริ่นไว้ก่อน Part 81)

หัวข้อนี้เกริ่นสั้น ๆ ถึงสิ่งที่โปรเจกต์จริงต้องตั้งค่าเพิ่มเติมนอกเหนือจากที่บทนี้สาธิต — บางส่วนตรวจสอบได้จาก source code ของ `tonic` จริง (API มีอยู่จริงตามที่อ้าง) แต่**ไม่ได้ setup TLS/certificate จริงเพื่อรันทดสอบในบทนี้** (ต้องมีการสร้าง certificate ซึ่งอยู่นอกขอบเขตของบทที่โฟกัสกลไกของ gRPC เอง) — ระบุไว้ตรง ๆ เพื่อไม่ให้สับสนกับส่วนอื่นของบทที่รันจริงและ capture output จริงทั้งหมด

**TLS และ mutual TLS (mTLS)**: `tonic::transport::Server` มีเมธอด `.tls_config(ServerTlsConfig)` และ `Channel` (ฝั่ง client) มี `.tls_config(ClientTlsConfig)` — ตั้งค่า certificate ของ server ผ่าน `ServerTlsConfig::new().identity(Identity::from_pem(cert, key))` เพื่อเปิด TLS ธรรมดา (client ตรวจสอบ identity ของ server แต่ server ไม่ตรวจ client) หรือเพิ่ม `.client_ca_root(Certificate::from_pem(ca_cert))` เพื่อเปิด **mutual TLS** ที่ server ตรวจสอบ certificate ของ client ด้วย — mTLS เป็นแนวทางที่นิยมมากในระบบ microservices ภายใน (Part 81) สำหรับพิสูจน์ตัวตนระหว่าง service โดยไม่ต้องพึ่ง JWT ผ่าน metadata แบบที่หัวข้อ 80.8 สาธิตไว้เลย (เป็นอีกวิธีหนึ่งที่ทำหน้าที่คล้ายกัน แต่ยืนยันตัวตนที่ระดับ TLS handshake ก่อนจะมีข้อมูล gRPC ไหนวิ่งเลยด้วยซ้ำ) หลายระบบเลือกใช้ทั้งสองแบบพร้อมกัน (mTLS ยืนยันว่า "service ไหน" กำลังเรียก ส่วน JWT/metadata ยืนยันว่า "ผู้ใช้คนไหน" อยู่เบื้องหลัง request นั้น)

**Keepalive**: ทั้งฝั่ง server (`.tcp_keepalive(Some(duration))`, `.http2_keepalive_interval(...)`, `.http2_keepalive_timeout(...)`) และฝั่ง client (`Channel::tcp_keepalive(...)`, `.keep_alive_timeout(...)`, `.keep_alive_while_idle(true)`) มีการตั้งค่าเพื่อส่ง HTTP/2 ping frame เป็นระยะ ป้องกัน connection ที่ดูเหมือนยังเปิดอยู่แต่จริง ๆ ตายไปแล้ว (เช่น load balancer/firewall ตัดการเชื่อมต่อเงียบ ๆ โดยไม่ส่ง TCP FIN ที่ถูกต้อง) — สำคัญมากสำหรับ connection ระยะยาวแบบที่ streaming RPC (หัวข้อ 80.6) มักใช้งาน

**ขนาด Message สูงสุด**: สังเกตจากโค้ดที่ generate จริงในหัวข้อ 80.3 ว่าทั้ง client และ server struct มีเมธอด `.max_decoding_message_size(limit)` / `.max_encoding_message_size(limit)` ให้ปรับ (ค่า default ของการ decode คือ 4MB ตามที่ doc comment ในโค้อดที่ generate บอกไว้ตรง ๆ) — ควรตั้งค่านี้ให้เหมาะกับโดเมนจริงเสมอ เพราะ message ที่ไม่จำกัดขนาดเป็นช่องโหว่ DoS ที่ชัดเจน (client ส่ง message ขนาดยักษ์มาเพื่อให้ server ใช้ memory จนล้ม)

**Compression**: `tonic` มี feature flag `gzip` และ `deflate` (ปิดโดย default ต้องเปิดเองใน `Cargo.toml`) ที่เปิดใช้ผ่าน `.send_compressed(CompressionEncoding::Gzip)` / `.accept_compressed(CompressionEncoding::Gzip)` ทั้งฝั่ง client และ server — มีประโยชน์เมื่อ message มีข้อมูลซ้ำ ๆ มาก (เช่น text ยาว ๆ) แม้ Protobuf จะแน่นกว่า JSON อยู่แล้วตามหัวข้อ 80.1 ก็ตาม แลกกับ CPU time ที่ต้อง compress/decompress เพิ่ม (tradeoff แบบเดียวกับที่ Part 61 กล่าวถึง `Accept-Encoding`/`Content-Encoding` สำหรับ REST)

**Load Balancing**: ในระบบ microservices จริง (Part 81) มักมี service instance เดียวกันหลายตัวรันพร้อมกัน (สำหรับ scale และ fault tolerance) — Tonic รองรับ client-side load balancing แบบพื้นฐานผ่าน feature `channel` ที่ดึง `tower::balance` เข้ามา (สังเกตได้จาก `Cargo.toml` ของ `tonic` ที่ประกาศ `"tower?/balance"` ไว้ในนิยามของ feature `channel`) แต่ในระบบจริงขนาดใหญ่ทีมส่วนมากเลือกใช้ **service mesh** (เช่น Istio/Linkerd ที่ทำงานร่วมกับ Envoy proxy) จัดการเรื่อง load balancing, retry, circuit breaking ที่ระดับ infrastructure แทนที่จะทำในโค้ด Rust เอง — Part 81 จะพูดถึงแนวคิดนี้ต่อในบริบทของการออกแบบระบบ microservices แบบเต็มรูปแบบ

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืมติดตั้ง `protoc` — build script fail ทันที

`tonic-prost-build` ไม่ได้ parse `.proto` ด้วยตัวเอง แต่เรียก `protoc` เป็น external process (หัวข้อ 80.3) — ถ้าเครื่อง (หรือ CI) ไม่มี `protoc` อยู่ใน `PATH` จะได้ error ที่ตรงประเด็นมาก (พิสูจน์จริงด้วยการถอด `protoc` ออกจาก `PATH` ชั่วคราวแล้ว `cargo build`):

```text
error: failed to run custom build command for `grpc_demo v0.1.0 (...)`

Caused by:
  process didn't exit successfully: `.../build-script-build` (exit status: 1)
  --- stderr
  Error: Custom { kind: NotFound, error: "Could not find `protoc`. If `protoc` is installed,
  try setting the `PROTOC` environment variable to the path of the `protoc` binary.
  To install it on Debian, run `apt-get install protobuf-compiler`.
  It is also available at https://github.com/protocolbuffers/protobuf/releases
  For more information: https://docs.rs/prost-build/#sourcing-protoc" }
```

**วิธีแก้**: ติดตั้งด้วย `apt-get install -y protobuf-compiler` (Debian/Ubuntu) หรือดาวน์โหลด binary จาก GitHub release ของ `protocolbuffers/protobuf` แล้วตั้ง environment variable `PROTOC` ให้ชี้ไปที่ path ของมันถ้าไม่อยากพึ่ง `PATH` ระบบ (มีประโยชน์มากใน CI ที่ควบคุม environment เองทั้งหมด) โปรเจกต์ที่อยากเลี่ยงปัญหานี้ในทุกเครื่องถาวรอาจพิจารณาใช้ crate `protobuf-src` ที่ vendor `protoc` มาให้ในตัว (แลกกับเวลา compile ครั้งแรกที่นานขึ้นมาก)

### 2. ลืม `#[tonic::async_trait]` เหนือ `impl` block

trait ที่ `tonic-prost-build` generate ให้ (หัวข้อ 80.3) ถูกแปลงร่างด้วย `#[async_trait]` ไปแล้วตอน generate — ถ้า `impl` block ของคุณไม่แนบ attribute เดียวกัน compiler จะมองว่า signature ของ `async fn` ทั้งสองฝั่งไม่ตรงกัน (native async fn เทียบกับรูปแบบที่ `async_trait` แปลงให้แล้ว) พิสูจน์จริงด้วยการลบ `#[tonic::async_trait]` ออกจาก `impl BookService for MyBookService`:

```text
error[E0195]: lifetime parameters or bounds on method `get_book` do not match the trait declaration
  --> src/bin/server.rs:21:22
   |
21 |     async fn get_book(&self, request: Request<GetBookRequest>) -> Result<Response<Book>, Status> {
   |                      ^ lifetimes do not match method in trait
   |
  ::: .../out/library.v1.rs:271:9
   |
271 | /         async fn get_book(
272 | |             &self,
273 | |             request: tonic::Request<super::GetBookRequest>,
274 | |         ) -> std::result::Result<tonic::Response<super::Book>, tonic::Status>;
   | |______________________________________________________________________________- lifetimes in impl do not match this method in trait
```

**วิธีแก้**: แนบ `#[tonic::async_trait]` ไว้เหนือ `impl` block เสมอ ทุกครั้งที่ implement trait ที่ `tonic-prost-build` generate ให้ — เป็นกฎที่ต้องจำ ไม่มี exception

### 3. เขียน `build.rs` ตามแบบ Tutorial เก่าที่ใช้ `tonic-build` ตรง ๆ (ไม่มี `tonic-prost-build`)

Tutorial จำนวนมากบนอินเทอร์เน็ต (ที่เขียนก่อน Tonic 0.13) สอนให้เรียก `tonic_build::compile_protos(...)` ตรง ๆ — แต่ตั้งแต่ Tonic 0.13 เป็นต้นมา ฟังก์ชันนี้ถูกย้ายไปที่ `tonic-prost-build` แล้ว (หัวข้อ 80.3) พิสูจน์จริงด้วยการเขียน `build.rs` ตามแบบเก่าโดยมี `tonic-build` เป็น build-dependency:

```rust
// build.rs แบบเก่าที่ใช้ไม่ได้แล้ว
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::compile_protos("proto/book.proto")?;
    Ok(())
}
```

```text
error[E0425]: cannot find function `compile_protos` in crate `tonic_build`
 --> build.rs:2:18
  |
2 |     tonic_build::compile_protos("proto/book.proto")?;
  |                  ^^^^^^^^^^^^^^ not found in `tonic_build`
  |
help: consider importing this function
  |
1 + use tonic_prost_build::compile_protos;
  |
```

น่าสังเกตว่า compiler แนะนำวิธีแก้ที่ถูกต้องมาให้เองพอดี (`use tonic_prost_build::compile_protos`) — **วิธีแก้**: เปลี่ยนเป็น `tonic_prost_build::configure().compile_protos(...)` ตามที่หัวข้อ 80.3 สอนไว้ และเพิ่ม `tonic-prost-build` เป็น build-dependency แทน (หรือคู่กับ `tonic-build` ก็ได้ถ้าต้องใช้ฟีเจอร์ manual codegen ระดับต่ำของมันด้วย แต่ส่วนที่ compile `.proto` ต้องมาจาก `tonic-prost-build` เท่านั้น)

### 4. คิดว่ายิง `curl` ธรรมดาเข้า gRPC server ได้เหมือน REST API

เพราะ gRPC วิ่งบน HTTP/2 เท่านั้น (หัวข้อ 80.1) การยิง `curl` แบบปกติ (ที่เริ่มต้นด้วย HTTP/1.1 แล้วค่อย upgrade ถ้า server รองรับ) เข้า gRPC server ที่พูด HTTP/2 ล้วน ๆ จะล้มเหลวแบบงงงวย พิสูจน์จริง:

```text
$ curl -v http://127.0.0.1:50051/library.v1.BookService/GetBook
> GET /library.v1.BookService/GetBook HTTP/1.1
> Host: 127.0.0.1:50051
>
* Received HTTP/0.9 when not allowed
curl: (1) Received HTTP/0.9 when not allowed
```

**วิธีแก้**: ใช้ `grpcurl` (หัวข้อ 80.9), เขียน gRPC client จริง (หัวข้อ 80.5), หรือถ้าต้องการใช้ `curl` ทดสอบจริง ๆ ต้องบังคับ `curl --http2-prior-knowledge` (บอก curl ว่าให้เริ่มคุยด้วย HTTP/2 ทันทีโดยไม่ negotiate) พร้อมส่ง body เป็น Protobuf binary ที่ encode มาถูกต้อง (ซับซ้อนเกินจะทำด้วยมือในทางปฏิบัติ — นี่คือเหตุผลที่มี `grpcurl` อยู่)

### 5. ลืมแนบ metadata แล้วสงสัยว่าทำไม request ถูกปฏิเสธเงียบ ๆ

ถ้ามี interceptor ตรวจ auth ไว้ (หัวข้อ 80.8) แล้วลืมแนบ metadata ฝั่ง client จะได้ error `Unauthenticated` ที่บางครั้งดูเหมือน "ทำไม request ปกติ ๆ ถึง fail" ถ้าไม่ได้อ่าน error message ให้ครบ:

```text
gRPC error: code=Unauthenticated message="ไม่พบ authorization metadata"
```

**วิธีแก้**: ตรวจสอบเสมอว่า `Request::metadata_mut().insert(...)` ถูกเรียกก่อนส่ง request ทุกครั้งที่ RPC นั้นต้องผ่าน interceptor ที่ตรวจ auth — และจำไว้ว่า metadata key ต้องเป็นตัวพิมพ์เล็กเสมอ (`"authorization"` ไม่ใช่ `"Authorization"`) เพราะ metadata key ของ gRPC (ต่างจาก HTTP header name ที่ case-insensitive) ตาม spec ต้องเป็นตัวพิมพ์เล็กล้วนเท่านั้น พิสูจน์จริงด้วยการลองใส่ตัวพิมพ์ใหญ่ปนดู:

```rust
req.metadata_mut().insert("Authorization", "Bearer x".parse().unwrap());
```

```text
thread 'main' panicked at .../http-1.5.0/src/header/name.rs:1254:13:
HeaderName::from_static with invalid bytes
```

`MetadataMap::insert` เมื่อรับ key เป็น `&'static str` ตรง ๆ จะเรียก `MetadataKey::from_static` ซึ่ง**panic ทันทีตอน runtime** ถ้า key มีตัวอักษรที่ไม่ใช่ตัวพิมพ์เล็ก/ตัวเลข/`-`/`_` (ไม่ใช่ compile-time error เพราะ Rust ตรวจสอบเนื้อหาของ string literal ไม่ได้ตอน compile) — นี่คือกับดักที่อันตรายเป็นพิเศษเพราะโค้ดจะ compile ผ่านสนิท แล้วไป panic ตอนรันจริงเท่านั้น (มักเจอตอน copy-paste header name จาก REST API เดิมที่ใช้ตัวพิมพ์ใหญ่แบบ `Authorization` ตามธรรมเนียม HTTP)

### 6. ตั้ง Deadline ฝั่ง Client แล้วคาดหวัง `DeadlineExceeded` แต่ได้ `Cancelled` แทน

`tonic::Request` มีเมธอด `set_timeout(Duration)` ให้กำหนดเวลาสูงสุดที่ client จะรอ RPC นั้น — พิสูจน์จริงด้วยการเรียก RPC ที่ server จำลอง delay 500ms แต่ client ตั้ง `set_timeout(Duration::from_millis(100))`:

```text
gRPC error หลังจาก 101.519893ms: code=Cancelled message="Timeout expired"
```

สังเกตว่า `code` ที่ได้คือ **`Cancelled`** (เลข 1) **ไม่ใช่ `DeadlineExceeded`** (เลข 4) ตามที่อาจคาดไว้ตามสัญชาตญาณ — เหตุผลคือ `set_timeout` ของ Tonic ฝั่ง client เป็นกลไก**local timeout ที่ client บังคับยกเลิกเองฝั่งตัวเอง**เมื่อรอเกินเวลาที่กำหนด (ยกเลิก request โดยไม่รอ server เลย) ต่างจาก `DeadlineExceeded` ที่มักหมายถึง**server เองตรวจพบว่าเกิน deadline ที่ client ส่งมาให้ผ่าน `grpc-timeout` header แล้ว server ตัดสินใจปฏิเสธเอง** — สองสถานการณ์นี้ต่างกันในรายละเอียด (ใครเป็นคนตัดสินใจเลิกรอ) แต่ให้ผลลัพธ์ที่ผู้ใช้เห็นคล้ายกันมาก **วิธีแก้**: ในโค้ดที่ต้อง handle timeout ให้ครอบคลุมทั้งสองกรณี ควร match ทั้ง `Code::Cancelled` และ `Code::DeadlineExceeded` แทนที่จะเช็คแค่ตัวเดียว ถ้าความหมายทางธุรกิจของทั้งสองกรณีคือ "ต้อง retry หรือแจ้งผู้ใช้ว่าช้าเกินไป" เหมือนกัน

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม RPC ใหม่ชื่อ `DeleteBook(DeleteBookRequest) returns (DeleteBookResponse)` ใน `book.proto` (unary ธรรมดา) โดย `DeleteBookRequest` มี field `id` (int64) และ `DeleteBookResponse` มี field `deleted` (bool) — implement ฝั่ง server ให้คืน `NotFound` ถ้า id ไม่มีอยู่จริง แล้วเขียน client เรียกทดสอบทั้งกรณีลบสำเร็จและกรณี id ไม่มีอยู่ *(hint: เพิ่ม method ใน `BookStore` ชื่อ `delete(&self, id: i64) -> bool` ที่ใช้ `HashMap::remove` แล้วเช็คว่า `Option` ที่ได้กลับมาเป็น `Some`/`None`)*

2. **(กลาง)** เพิ่ม field `optional string isbn = 6;` เข้าไปใน message `Book` (proto3 `optional` — generate เป็น `Option<String>` ใน Rust) แล้วแก้ `BookStore::create` ให้รับ `isbn` เพิ่ม และแก้ `GetBook` ให้คืน `INVALID_ARGUMENT` ถ้ามีคนพยายาม `CreateBook` ด้วย `isbn` ที่มีอยู่แล้วในระบบ (ทำ uniqueness check เอง เพราะ `HashMap<i64, Book>` ไม่มี index บน `isbn`) *(hint: เพิ่ม `HashSet<String>` แยกไว้เก็บ isbn ที่ใช้ไปแล้ว หรือ loop หา `values()` ทุกครั้งก็ได้ถ้าข้อมูลไม่มาก)*

3. **(ยาก)** ออกแบบ RPC ใหม่ชื่อ `SearchBooks` ที่เป็น **server streaming** รับ `SearchBooksRequest { query: string }` แล้ว stream คืนเฉพาะ `Book` ที่ `title` หรือ `author` มีคำว่า `query` เป็น substring (case-insensitive) — เพิ่ม interceptor ตัวที่สอง (แยกจาก auth) ที่ตรวจว่า `query` ต้องมีความยาวอย่างน้อย 2 ตัวอักษร ไม่เช่นนั้นคืน `InvalidArgument` ทันทีโดยไม่ต้องเข้า handler เลย — คำถามให้คิดต่อ: `Interceptor` ตัวเดียวตรวจได้ทุก RPC ของ service พร้อมกันไหม หรือต้องแยก service ถ้าอยาก apply เฉพาะบาง RPC? *(hint: อ่านหัวข้อ 80.8 อีกครั้ง — `Interceptor::call` รับ `Request<()>` ที่ไม่รู้ว่าเป็น RPC ไหน คุณจะรู้ได้จาก `request.metadata()` เท่านั้น ไม่มี body ให้ตรวจ ต้องหาทางอื่นถ้าต้องการ validate ค่าที่อยู่ใน body — คำตอบที่ตรงกว่าคือ validate ข้างใน handler เอง ไม่ใช่ interceptor ซึ่งเป็นข้อจำกัดที่ควรสรุปเป็นคำตอบของแบบฝึกหัดนี้ด้วย)*

4. **(ยาก/ประยุกต์)** สลับ `BookStore` จาก in-memory (`Mutex<HashMap>`) ไปใช้ SQLx + PostgreSQL ตามที่ Part 70-71 สอนไว้ โดยที่ **ห้ามแก้ signature ของ trait `BookService` แม้แต่บรรทัดเดียว** — สร้างตาราง `books` ที่มี column ตรงกับ field ของ message `Book`, เปลี่ยน `MyBookService` ให้เก็บ `sqlx::PgPool` แทน `Arc<BookStore>`, แล้วแก้ทุก method ให้เรียก `sqlx::query_as!`/`sqlx::query!` แทนการเรียก `self.store.*` — แปลง `sqlx::Error` เป็น `tonic::Status::internal(...)` ที่จุดไหนให้เหมาะสม และคิดว่า `list_books` (server streaming) ควรใช้ `sqlx::query_as!(...).fetch()` (ที่คืน stream แบบ lazy จาก database driver ตรง ๆ) หรือ `.fetch_all()` แล้วค่อยแปลงเป็น stream ทีหลัง — เหตุผลอะไรที่ทำให้เลือกทางหนึ่งดีกว่าอีกทางในสถานการณ์ตารางข้อมูลขนาดใหญ่จริง ๆ

## สรุป

บทนี้แนะนำ **gRPC** ในฐานะเครื่องมือสื่อสารที่**ต่างขั้ว**จาก REST/GraphQL ที่เรียนมาก่อนหน้าโดยตั้งใจ: ใช้ **Protocol Buffers** (binary format ที่เล็กและเร็วกว่า JSON แต่แลกมาด้วย debuggability) ส่งผ่าน **HTTP/2** (multiplexing + header compression ที่ REST บน HTTP/1.1 ทั่วไปไม่ได้ใช้ประโยชน์เต็มที่) เราเขียน `.proto` จริง, ตั้งค่า `build.rs` ด้วย `tonic-prost-build` (ชี้ให้เห็นการเปลี่ยนแปลง API ที่สำคัญจาก `tonic-build` เวอร์ชันเก่า พร้อมพิสูจน์ error จริงที่จะเจอถ้าใช้แบบเก่า) และอธิบายอย่างแม่นยำว่าการ generate โค้ดแบบนี้เป็น**กลไกคนละชั้น**กับ proc macro ที่ Part 44-45 สอน — build script ที่รันก่อน compile เขียนไฟล์ `.rs` ธรรมดาไว้ให้ ไม่ใช่การแทรก `TokenStream` เข้าไประหว่าง compile

เราได้ implement server และ client ที่รันได้จริงครบทั้ง **4 รูปแบบการเรียก** (unary, server streaming, client streaming, และ bidirectional streaming — ทุกแบบ compile, รัน, และ capture output จริงหมด ไม่มีส่วนใดเป็นแค่ทฤษฎี) จัดการ error ด้วย `tonic::Status` พร้อมเทียบ status code กับ HTTP status code จาก Part 61 และพิสูจน์ข้อเท็จจริงสำคัญว่า gRPC ซ้อน status ของตัวเองอยู่ภายใน HTTP/2 `200 OK` เสมอ เขียน **interceptor** สำหรับตรวจสอบ auth ผ่าน metadata (พบข้อเท็จจริงเชิงสถาปัตยกรรมที่น่าสนใจว่า Tonic สร้างอยู่บน `tower` เหมือน Axum) และเปิดใช้ **server reflection** ให้เครื่องมือภายนอกสำรวจ schema ได้โดยไม่ต้องมี `.proto` แยก

ท้ายบทวางจุดยืนของ gRPC ให้ชัดเจนที่สุด: **เหมาะกับการสื่อสารระหว่าง service ภายในระบบที่คุณควบคุมทั้งสองฝั่ง** ไม่ใช่ตัวเลือกสำหรับ public API ที่ client ภายนอก (โดยเฉพาะ browser ซึ่งพิสูจน์แล้วว่าคุย gRPC ตรง ๆ ไม่ได้) จะเรียกใช้ — REST ด้วย Axum ยังคงเป็นแนวทางหลักของหลักสูตรนี้สำหรับ client ภายนอก ส่วน gRPC ที่เรียนในบทนี้จะกลายเป็นเครื่องมือสื่อสารหลักระหว่าง service ต่าง ๆ ใน **Part 81 (Microservices Architecture ด้วย Rust)** ที่จะเรียนต่อไป

---

**Part ก่อนหน้า:** [GraphQL ด้วย async-graphql](part-079-graphql.md) | **Part ถัดไป:** [Microservices Architecture ด้วย Rust](part-081-microservices.md)
