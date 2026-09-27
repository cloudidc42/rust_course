# Part 77: WebSockets ด้วย Axum

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 270 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างละเอียดว่าทำไม HTTP request/response แบบที่เรียนมาตั้งแต่ Part 61 ถึง Part 76 (ทุก request
  ต้องเริ่มจาก client เสมอ, connection ปิดทันทีที่ response ส่งเสร็จ) **ไม่เพียงพอ**สำหรับข้อมูลแบบ real-time
  ที่ server ต้อง "ดัน" (push) ข้อมูลไปให้ client เองโดยที่ client ไม่ได้ถามก่อน — และอธิบาย **WebSocket
  handshake** ได้ตรงตาม RFC 6455 จริง ว่ามันคือ HTTP request ธรรมดาที่ "ขอเปลี่ยนโพรโทคอล" (`Upgrade`
  header) ไม่ใช่โพรโทคอลคนละโลกกับ HTTP เลย พร้อมพิสูจน์ด้วย handshake จริงที่ capture มาจากการรันโปรแกรมจริง
- เขียน WebSocket echo server ตัวแรกด้วย extractor `WebSocketUpgrade` ของ Axum ได้ครบ compile และรันได้จริง
  ทดสอบด้วย WebSocket client ที่เขียนเป็นภาษา Rust เอง (ผ่าน `tokio-tungstenite`) พร้อม capture การสนทนาจริง
  ทุกข้อความที่ส่ง-รับ
- แยกแยะและจัดการ `Message` ทั้ง 5 variant ของ Axum (`Text`, `Binary`, `Ping`, `Pong`, `Close`) ได้ถูกต้อง
  พร้อมพิสูจน์ด้วยการรันจริงว่า **Ping/Pong ถูกตอบกลับโดยอัตโนมัติที่ระดับ protocol** โดยไม่ต้องเขียนโค้ดจัดการ
  เองแม้แต่บรรทัดเดียว
- แยก `WebSocket` ออกเป็น `SplitSink`/`SplitStream` ด้วย `.split()` แล้ว `tokio::spawn` งานเขียนแยกจากงานอ่าน
  (ต่อยอดจาก Part 39/48 เรื่อง task และ concurrency) เพื่อให้ server "ดัน" ข้อมูลไปยัง client ได้อย่างเป็นอิสระ
  จากข้อความที่ client กำลังส่งมา — พิสูจน์ด้วยการรันจริงว่าทั้งสอง task ทำงานคู่ขนานกันได้จริง
- สร้างระบบ broadcast/chat-room เต็มรูปแบบด้วย `tokio::sync::broadcast` (ต่อยอดตรงจาก Part 50) ที่แชร์ผ่าน
  `AppState` (ต่อยอดจาก Part 64) — แต่ละ WebSocket connection subscribe เข้าไปรับข้อความและส่งข้อความของตัวเอง
  เข้าไปใน broadcast channel เดียวกันได้ พร้อมพิสูจน์ด้วยการรัน client 2 ตัวจริงพร้อมกัน
- เก็บ **per-connection state** (เช่น `HashMap<ClientId, Sender>`) เพื่อส่งข้อความเจาะจงถึง client รายบุคคล
  (ไม่ใช่ broadcast ให้ทุกคน) พร้อมตัวอย่างจริงที่ REST endpoint ยิงข้อความส่วนตัวไปหา client ที่ระบุ id เท่านั้น
- ยืนยัน (authenticate) การเชื่อมต่อ WebSocket ตอน handshake ด้วยแนวคิดเดียวกับ JWT (Part 74) หรือ session
  (Part 75) — พร้อมอธิบายข้อจำกัดจริงของ browser WebSocket API ที่ไม่สามารถกำหนด custom header ได้ และวิธีแก้
  ที่ใช้กันจริงในโลกจริง (query parameter / cookie)
- ตรวจจับการหลุดการเชื่อมต่อของ client และเก็บกวาด state ที่เกี่ยวข้องอย่างถูกต้องด้วยแนวคิด RAII/`Drop`
  (ต่อยอดจาก Part 6) พร้อมพิสูจน์ว่า panic ภายใน task ของ connection หนึ่งไม่กระทบ connection อื่นเลย (ต่อยอด
  จาก Part 48)
- ประกอบทุกอย่างเข้าด้วยกันเป็นระบบ **"แสดงจำนวนที่นั่งคงเหลือแบบ real-time"** ที่ REST endpoint (สไตล์ Part
  63) กระตุ้นให้เกิดการ broadcast ไปยัง WebSocket client ที่ authenticated และถูก track อยู่ทุกคนพร้อมกัน

## ความรู้ที่ต้องมีมาก่อน

- **Part 61 (HTTP Fundamentals และ REST API Concepts)**: บทนี้อ้างอิงกลไก HTTP request/response,
  status code, และคำว่า "connection" ตรง ๆ ตลอดทั้งบท โดยเฉพาะหัวข้อ 77.1 ที่จะเทียบให้เห็นว่า WebSocket
  ต่างจาก HTTP ปกติที่ Part 61 สอนไว้อย่างไร
- **Part 62-64 (Axum Intro, Routing/Handlers, State/Extractors)**: บทนี้ใช้ `Router`, `.route()`,
  extractor แบบ `State<T>`/`Query<T>`/`Path<T>`, และแพทเทิร์น `AppState` + `.with_state()` ที่ Part 64 สอน
  ไว้เต็มรูปแบบ ต่อยอดตรง ๆ ไม่อธิบายกลไกพื้นฐานซ้ำ
- **Part 66 (Axum: Error Handling แบบมืออาชีพ)**: ใช้ `StatusCode`, `IntoResponse` ในการตอบ rejection ของ
  WebSocket upgrade ที่ auth ไม่ผ่าน
- **Part 39 (Mutex, Arc และ Shared-State Concurrency)** และ **Part 50 (Async Channels และ Synchronization
  — tokio::sync)**: หัวใจของบทนี้คือ `tokio::sync::broadcast` ที่ Part 50 สอนไว้เต็มรูปแบบแล้ว (รวมถึงกฎ
  `RecvError::Lagged` vs `RecvError::Closed`, และกฎการตัดสินใจ `std::sync::Mutex` vs `tokio::sync::Mutex`
  จากหัวข้อ 50.6) — บทนี้เอาเครื่องมือเดียวกันมาต่อสาย (wire) เข้ากับ WebSocket connection ของ Axum โดยตรง
  ถ้ายังไม่แม่นเรื่อง `broadcast::channel`, `.subscribe()`, หรือกฎ "ดูที่ critical section ไม่ใช่ดูที่บริบท
  รอบข้าง" กลับไปทวนก่อน เพราะจะไม่อธิบายซ้ำจากศูนย์
- **Part 48 (Tokio: Runtime และ Tasks)**: บทนี้ใช้ `tokio::spawn` ต่อยอดตรง ๆ และอ้างอิงพฤติกรรมสำคัญที่
  Part 48 พิสูจน์ไว้แล้ว — panic ของ task ที่ spawn ไว้**ไม่ทำให้ทั้งโปรแกรม crash** (ถูก catch แล้วแปลงเป็น
  `JoinError`) — บทนี้จะพิสูจน์ว่ากฎเดียวกันนี้ใช้ได้กับ task ของ WebSocket connection แต่ละอันที่ Axum
  สร้างให้เองด้วย
- **Part 6 (Ownership เบื้องต้น และ Drop)**: หัวข้อ 77.9 ใช้ `Drop` trait เป็น RAII guard เพื่อการันตีว่า
  cleanup (ลบ client ออกจาก map) จะเกิดขึ้นเสมอไม่ว่า task จะจบด้วยเหตุผลอะไร — ต่อยอดตรงจากแนวคิด "หลุด
  scope แล้วปลดปล่อยอัตโนมัติ" ที่ Part 6 สอนไว้
- **Part 74 (Authentication: JWT)** และ **Part 75 (Session และ OAuth2)**: หัวข้อ 77.8 ใช้แนวคิดการตรวจสอบ
  token/session ที่สองบทนี้สอนไว้ตรง ๆ (ในตัวอย่างของบทนี้จะย่อขั้นตอนตรวจสอบให้ง่ายลงเพื่อโฟกัสที่ประเด็นของ
  WebSocket แต่หลักการเดียวกันทุกประการ)
- **Part 63 (Axum: Routing และ Handlers)**: capstone ท้ายบทใช้ REST endpoint สไตล์ CRUD ที่ Part 63 สอนไว้
  ควบคู่กับ WebSocket endpoint ในแอปเดียวกัน

## เนื้อหา

### 77.1 ทำไม HTTP Request/Response ไม่พอสำหรับข้อมูลแบบ Real-Time

ตั้งแต่ Part 61 เป็นต้นมา ทุกอย่างที่หลักสูตรนี้สอนเกี่ยวกับเว็บล้วนอยู่ภายใต้กฎเดียวกันของ HTTP: **client เป็น
ฝ่ายเริ่มเสมอ** — client ส่ง request มา, server ประมวลผลแล้วส่ง response กลับ, จบ connection (หรือถ้าใช้
`Connection: keep-alive` ก็แค่เปิด TCP connection ไว้เผื่อ request ถัดไป แต่ตัว "การสนทนา" ระดับ HTTP ก็ยังเป็น
คู่ request-response ทีละคู่เหมือนเดิม) — **server ไม่มีทางส่งอะไรไปให้ client โดยที่ client ไม่ได้ขอมาก่อน
เลย** นี่คือข้อจำกัดพื้นฐานที่สุดของโมเดล HTTP request/response ที่ Part 61-76 ใช้มาตลอด

ลองนึกภาพระบบจองตั๋วที่สร้างกันมาตลอดหลักสูตร (Part 63-64) ที่ต้องการฟีเจอร์ใหม่: **แสดงจำนวนที่นั่งคงเหลือ
แบบสด ๆ** — ถ้ามีคนอื่นซื้อตั๋วไปแล้วที่นั่งลดลง หน้าเว็บของทุกคนที่เปิดดูหน้า event นั้นอยู่ต้องอัปเดตทันที
โดยที่ผู้ใช้ไม่ต้องกด refresh เอง ด้วยเครื่องมือที่มีแค่ HTTP request/response ธรรมดา มีทางเลือกจำกัดอยู่สองทาง
ที่ทั้งคู่มีปัญหาจริง:

**ทางเลือกที่ 1: Polling** — ให้ browser ยิง HTTP request ไปถาม server ซ้ำ ๆ ทุก ๆ N วินาที ("มีอะไรเปลี่ยน
ไหม?") ปัญหาคือ **trade-off ระหว่างความสดของข้อมูลกับภาระที่เพิ่มขึ้นแบบเป็นเส้นตรงตามจำนวน client**: ถ้า
poll ทุก 1 วินาทีเพื่อให้ข้อมูลสดพอ แอปที่มี 10,000 คนเปิดหน้าเดียวกันพร้อมกันจะสร้าง HTTP request ใหม่ **10,000
ครั้งต่อวินาที** ไปที่ server ทั้งที่ส่วนใหญ่คำตอบจะเป็น "ไม่มีอะไรเปลี่ยน" เกือบทุกครั้ง — เปลืองทั้ง CPU
ฝั่ง server ที่ต้องสร้าง response ใหม่ทุกครั้ง (แม้จะตอบว่าไม่มีอะไรเปลี่ยน ก็ยังต้องผ่านทุกขั้นตอนของ HTTP
request/response: parse header, route matching, สร้าง response object ใหม่) และเปลือง bandwidth ที่ต้องส่ง
HTTP header เต็มรูปแบบไปมาซ้ำ ๆ (แต่ละ HTTP request มี header หลายร้อยไบต์เป็น overhead แม้ body จะว่างเปล่า)

**ทางเลือกที่ 2: Long Polling** — client ส่ง request ไปแล้ว server "ถ่วงเวลา" ไม่ตอบทันที รอจนกว่าจะมีข้อมูล
ใหม่จริง ๆ ค่อยตอบ (หรือ timeout แล้วตอบว่าง ๆ ให้ client ยิงมาใหม่) นี่ลดจำนวน request ที่ "ไม่มีอะไรเปลี่ยน"
ลงได้มาก แต่ยังมีข้อจำกัดที่แก้ไม่ได้: **แต่ละ long-polling request ยังคือ HTTP request/response แบบทาง
เดียวอยู่ดี** — ถ้า server ต้องการส่งข้อความสองอันติดกันในเวลาไล่เลี่ยกัน มันต้องรอให้ response แรกถูกส่งจบและ
client ยิง request ใหม่มาก่อน ถึงจะส่งอันที่สองได้ — และการเปิด/ปิด HTTP connection ซ้ำ ๆ ต่อเนื่องยังมี
overhead ของการสร้าง response object และ header ใหม่ทุกรอบเหมือนเดิม

**WebSocket แก้ปัญหานี้ที่ต้นตอ**: มันคือโพรโทคอลที่เปลี่ยน TCP connection เดิมที่เปิดผ่าน HTTP ให้กลายเป็น
**full-duplex channel ที่เปิดค้างไว้ยาว ๆ** — ทั้ง client และ server ส่งข้อความ (frame) ไปมาได้ **ทั้งสอง
ทิศทางพร้อมกัน โดยไม่ต้องรอให้อีกฝ่าย "ขอ" ก่อนเลย** ไม่มีแนวคิดเรื่อง request/response คู่กันอีกต่อไป — มีแค่
"ข้อความ" ที่ฝั่งไหนอยากส่งเมื่อไหร่ก็ส่งได้ทันที และ **connection เดียวกันนั้นอยู่ค้างไว้ตลอดอายุของการสนทนา**
(อาจเป็นนาทีหรือชั่วโมง ไม่ใช่แค่เสี้ยววินาทีแบบ HTTP request/response ปกติ) — นี่คือความต่างเชิงโครงสร้างที่
สำคัญที่สุดที่ทำให้บทนี้เป็นบทแรกในหลักสูตรที่ต้องคิดเรื่อง **connection ที่มีอายุยืน (long-lived connection)**
จริง ๆ ต่างจากทุกบทก่อนหน้าตั้งแต่ Part 61 ที่ connection แต่ละอันจบไปพร้อมกับ response เดียว

หัวข้อถัดไปจะแสดงให้เห็นว่า WebSocket "เกิดขึ้น" จาก HTTP request ธรรมดาได้อย่างไร — มันไม่ใช่โพรโทคอลคนละโลก
ที่ไม่เกี่ยวกับ HTTP เลย แต่เป็นการ **"อัปเกรด" HTTP connection ที่มีอยู่แล้ว** ให้เปลี่ยนพฤติกรรมไปเป็น
full-duplex

### 77.2 WebSocket Handshake: การ "อัปเกรด" HTTP Connection ที่มีอยู่แล้ว

จุดที่สำคัญที่สุดที่ต้องเข้าใจให้ถูก (และเป็นสิ่งที่คนจำนวนมากเข้าใจผิด): **WebSocket ไม่ได้เริ่มต้นด้วย
โพรโทคอลใหม่ที่ไม่เกี่ยวกับ HTTP เลย** — มันเริ่มต้นด้วย **HTTP request ธรรมดาเป๊ะ ๆ** (method `GET`, มี
header, ตาม RFC 7230 ที่ Part 61 อธิบายไว้ทุกประการ) ที่มี header พิเศษบางตัวขอ "เปลี่ยนโพรโทคอล" — ถ้า
server ยินยอม มันจะตอบ HTTP status `101 Switching Protocols` แล้ว **TCP connection เดิมที่เปิดไว้ตั้งแต่
request แรก** จะเปลี่ยนความหมายไปเป็น WebSocket frame stream ทันที ไม่มีการเปิด connection ใหม่เลย

Header สำคัญที่ทำให้เกิดการอัปเกรดนี้ (นิยามใน RFC 6455) มีดังนี้:

- **`Upgrade: websocket`** — บอก server ว่า client อยากเปลี่ยนโพรโทคอลไปเป็น WebSocket
- **`Connection: Upgrade`** — บอกว่า header `Upgrade` มีผลจริง (ไม่ใช่แค่ header เฉย ๆ)
- **`Sec-WebSocket-Version: 13`** — เวอร์ชันของโพรโทคอล WebSocket ที่ใช้ (13 คือเวอร์ชันมาตรฐานที่ browser
  ทุกตัวใช้ในปัจจุบัน)
- **`Sec-WebSocket-Key`** — ค่า random 16 ไบต์ที่เข้ารหัสด้วย Base64 ที่ client สร้างขึ้นมาใหม่ทุกครั้ง —
  มีไว้เพื่อพิสูจน์ว่า server ที่ตอบกลับมาเข้าใจโพรโทคอล WebSocket จริง (ไม่ใช่ HTTP proxy เก่า ๆ ที่ไม่รู้จัก
  WebSocket แล้วสับสนกับ header เหล่านี้)
- **`Sec-WebSocket-Accept`** (ฝั่ง response) — server ต้องคำนวณค่านี้จาก `Sec-WebSocket-Key` ที่ client ส่งมา
  ด้วยสูตรตายตัวตาม RFC 6455: **ต่อ string ของ key เข้ากับ GUID คงที่ `258EAFA5-E914-47DA-95CA-C5AB0DC85B11`
  แล้ว hash ด้วย SHA-1 จากนั้น encode ด้วย Base64** — ถ้า server คำนวณค่านี้ผิด (หรือไม่ใส่มาเลย) client จะ
  ปฏิเสธ handshake ทันที เพราะนี่คือหลักฐานเดียวที่พิสูจน์ว่า server "เข้าใจ" WebSocket จริง ไม่ใช่ HTTP
  server ธรรมดาที่ตอบ `101` มาแบบสุ่ม ๆ

ค่า GUID คงที่ `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` นี้ไม่ใช่ secret อะไรเลย — มันเป็นค่าที่**เขียนตายตัวไว้
ใน RFC 6455 เอง** และทุก implementation (browser, Axum, Node.js, ทุกภาษา) ต้องใช้ค่าเดียวกันนี้เป๊ะ ๆ เพื่อให้
คำนวณผลลัพธ์ตรงกัน

#### พิสูจน์ด้วย handshake จริงที่ capture มาจากการรัน

มาดู handshake จริง ๆ ที่เกิดขึ้นตอนรันโปรแกรม — เขียน raw TCP client (ใช้ `std::net::TcpStream` ธรรมดา ไม่ผ่าน
library WebSocket ใด ๆ เลย เพื่อให้เห็น byte ที่ไหลผ่านสายจริง ๆ) ส่ง GET request ที่มี header ครบตาม RFC 6455
โดยใช้ **ค่า `Sec-WebSocket-Key` ที่ตรงกับตัวอย่างในตัว RFC 6455 เอง** (`dGhlIHNhbXBsZSBub25jZQ==`) เพื่อให้
เทียบผลลัพธ์กับสิ่งที่ spec บอกไว้ได้ตรง ๆ:

```rust
use std::io::{Read, Write};
use std::net::TcpStream;

fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:4001").expect("connect");

    let request = "GET /ws HTTP/1.1\r\n\
Host: 127.0.0.1:4001\r\n\
Connection: Upgrade\r\n\
Upgrade: websocket\r\n\
Sec-WebSocket-Version: 13\r\n\
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==\r\n\
\r\n";

    stream.write_all(request.as_bytes()).expect("write request");

    let mut buf = Vec::new();
    let mut chunk = [0u8; 512];
    loop {
        let n = stream.read(&mut chunk).expect("read response");
        if n == 0 {
            break;
        }
        buf.extend_from_slice(&chunk[..n]);
        if buf.windows(4).any(|w| w == b"\r\n\r\n") {
            break;
        }
    }

    println!("{}", String::from_utf8_lossy(&buf));
}
```

ฝั่ง server เป็น Axum WebSocket handler ธรรมดาที่สุดที่จะอธิบายเต็มรูปแบบในหัวข้อ 77.3 ถัดไป — รันจริงแล้วได้
**response ที่ capture มาตรง ๆ** ดังนี้:

```
GET /ws HTTP/1.1
Host: 127.0.0.1:4001
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

----- RESPONSE ที่ได้กลับมาจริง -----
HTTP/1.1 101 Switching Protocols
connection: upgrade
upgrade: websocket
sec-websocket-accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
date: Sun, 27 Sep 2026 01:22:36 GMT
```

สังเกตให้ชัด: **`sec-websocket-accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=` ตรงกับค่าตัวอย่างใน RFC 6455 เป๊ะทุก
ตัวอักษร** — นี่คือการพิสูจน์ตรง ๆ ว่า Axum implement สูตรคำนวณตาม spec ถูกต้อง 100% (SHA-1 ของ
`"dGhlIHNhbXBsZSBub25jZQ==" + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"` แล้ว encode Base64) และเห็นด้วยตาว่า
`GET /ws HTTP/1.1` ตัวเดียวกันนี้**คือ HTTP request ปกติทุกประการ** — Axum จัดการมันผ่าน `Router` เดียวกันกับ
route อื่น ๆ ในแอป (คนละ handler กับ REST endpoint ธรรมดาก็ได้ แต่ **อยู่ใน `Router` เดียวกัน** ตามที่จะเห็นใน
capstone ท้ายบทที่มีทั้ง REST endpoint และ WebSocket endpoint ใน `Router` ตัวเดียว) — status `101 Switching
Protocols` คือสิ่งที่บอก client (และ proxy ทุกตัวระหว่างทาง) ว่า "จากบรรทัดนี้ไป TCP connection นี้จะไม่ใช่
HTTP ธรรมดาแล้ว แต่เป็น WebSocket frame stream"

ในทางปฏิบัติ คุณไม่จำเป็นต้องคำนวณ `Sec-WebSocket-Accept` เองแม้แต่ครั้งเดียว — Axum (ผ่าน extractor
`WebSocketUpgrade` ที่หัวข้อถัดไปจะสอน) จัดการ header ทั้งหมดนี้ให้อัตโนมัติสมบูรณ์ แต่การเข้าใจว่ามันเกิดขึ้น
ยังไงจริง ๆ ช่วยให้ debug ปัญหา handshake (เช่น proxy หรือ load balancer บางตัวที่ตัด header `Upgrade` ทิ้ง
โดยไม่รู้ตัว ทำให้ WebSocket connection ผ่าน proxy นั้นไม่ได้) ได้แม่นยำขึ้นมาก

### 77.3 Echo Server ตัวแรกด้วย `WebSocketUpgrade`

ทีนี้มาเขียน WebSocket server ตัวแรกด้วย Axum จริง — extractor ชื่อ `WebSocketUpgrade` (อยู่ใน
`axum::extract::ws`) ทำหน้าที่ "จับ" HTTP request ที่มี header ครบตาม RFC 6455 (ที่หัวข้อ 77.2 อธิบายไป) แล้ว
ให้ handler เลือกว่าจะ "อนุมัติ" การอัปเกรดหรือไม่ ผ่าน method `.on_upgrade(callback)` — `callback` คือ
closure/ฟังก์ชันแบบ `async` ที่รับ `WebSocket` (ตัวแทนของ connection ที่อัปเกรดสำเร็จแล้ว) เข้ามา แล้วทำหน้าที่
อ่าน-เขียนข้อความไปตลอดอายุของ connection นั้น

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::Response,
    routing::any,
    Router,
};

// ฟังก์ชันนี้รันหลัง handshake สำเร็จแล้ว -- socket ตัวนี้คือ full-duplex channel เต็มรูปแบบ
async fn handle_socket(mut socket: WebSocket) {
    // recv().await คืน Option<Result<Message, axum::Error>>
    // None แปลว่า client หลุดการเชื่อมต่อไปแล้ว (จะอธิบายเต็มรูปแบบในหัวข้อ 77.9)
    while let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(text) = msg {
            // ส่งข้อความเดียวกันกลับไปทันที -- นี่คือความหมายของ "echo server"
            if socket.send(Message::Text(text)).await.is_err() {
                // ส่งไม่สำเร็จ = client หลุดการเชื่อมต่อไปแล้ว ไม่มีประโยชน์ที่จะพยายามอ่านต่อ
                break;
            }
        }
    }
}

// handler ธรรมดาที่รับ WebSocketUpgrade เป็น extractor -- เหมือน extractor อื่น ๆ ที่ Part 64 สอนไว้ทุกประการ
async fn ws_handler(ws: WebSocketUpgrade) -> Response {
    // .on_upgrade() คืนค่าเป็น Response (สถานะ 101 Switching Protocols พร้อม header ที่ถูกต้องครบ)
    // แล้ว spawn task แยกไปรัน handle_socket เมื่อ TCP connection อัปเกรดสำเร็จจริง ๆ
    ws.on_upgrade(handle_socket)
}

#[tokio::main]
async fn main() {
    // routing::any() รับได้ทุก HTTP method -- ในทางปฏิบัติ WebSocket handshake มาในรูป GET เสมอ
    // แต่ Axum แนะนำให้ใช้ any() เพราะ browser บางตัว/HTTP version บางเวอร์ชันอาจใช้ method อื่นได้ในอนาคต
    let app = Router::new().route("/ws", any(ws_handler));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:4001").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

สังเกตว่า `ws_handler` เป็น **`async fn` ธรรมดา** ที่ประกาศต้องการ `WebSocketUpgrade` ผ่าน parameter ตรง ๆ —
ไม่ต่างอะไรกับ handler ที่ขอ `State<T>` หรือ `Path<T>` ที่ Part 64 สอนไว้เลย นี่คือจุดสำคัญ: **WebSocket ใน
Axum ไม่ใช่ระบบแยกต่างหากจาก HTTP routing ปกติ** — มันคือ extractor ตัวหนึ่งที่ใช้ระบบ `FromRequestParts`
เดียวกันกับที่ Part 64 สอนไว้ (สามารถผสมกับ `State<T>`/`Query<T>`/custom extractor อื่น ๆ ในตัว handler
เดียวกันได้ตามลำดับกฎเดิมทุกประการ — จะเห็นตัวอย่างจริงในหัวข้อ 77.8 ที่ผสม `WebSocketUpgrade` กับ `Query<T>`
สำหรับ authentication)

#### ทดสอบด้วย WebSocket Client ที่เขียนเป็น Rust เอง

`curl` ไม่สามารถขับ WebSocket handshake เต็มรูปแบบและอ่าน-ส่งข้อความหลังจากนั้นได้ (มันทำได้แค่ handshake เปล่า
ๆ ด้วยแฟล็กพิเศษ ไม่ต่อเนื่องเป็นการสนทนา) จึงต้องเขียน client จริงด้วย crate `tokio-tungstenite` (WebSocket
client/server library ระดับ production ที่ Axum เองก็ใช้ `tungstenite` เป็นแกนกลางภายใน):

```rust
use futures_util::{SinkExt, StreamExt};
use tokio_tungstenite::connect_async;
use tokio_tungstenite::tungstenite::Message as TMessage;

#[tokio::main]
async fn main() {
    let (mut ws_stream, response) = connect_async("ws://127.0.0.1:4001/ws")
        .await
        .expect("Failed to connect");
    println!("handshake response status: {}", response.status());

    for text in ["สวัสดีครับ", "Hello WebSocket", "ping test 3"] {
        ws_stream.send(TMessage::Text(text.into())).await.unwrap();
        let reply = ws_stream.next().await.unwrap().unwrap();
        let reply_text = reply.to_text().unwrap();
        println!("ส่งไป: {text:?}  ได้รับกลับ: {reply_text:?}");
    }

    ws_stream.close(None).await.unwrap();
    println!("ปิดการเชื่อมต่อแล้ว");
}
```

รันจริง (เปิด server ไว้ก่อน แล้วรัน client) ได้ผลลัพธ์จริงดังนี้:

```
handshake response status: 101 Switching Protocols
ส่งไป: "สวัสดีครับ"  ได้รับกลับ: "สวัสดีครับ"
ส่งไป: "Hello WebSocket"  ได้รับกลับ: "Hello WebSocket"
ส่งไป: "ping test 3"  ได้รับกลับ: "ping test 3"
ปิดการเชื่อมต่อแล้ว
```

ทุกข้อความ (รวมถึงข้อความภาษาไทยที่มีทั้ง multi-byte UTF-8 และ combining character) ถูก echo กลับมาถูกต้อง
ครบถ้วน 100% — พิสูจน์ว่า handshake, การส่ง, และการรับทำงานถูกต้องตลอดสาย และ WebSocket frame รองรับ UTF-8
เต็มรูปแบบโดยไม่ต้อง encode/decode พิเศษอะไรเพิ่มเลย (Axum ใช้ type `Utf8Bytes` ภายใน `Message::Text` ที่
การันตีความถูกต้องของ UTF-8 ไว้แล้วตั้งแต่ตอนสร้าง — ผิดจาก UTF-8 จะไม่มีทาง compile เป็น `Message::Text` ได้
เลยด้วยซ้ำในหลายเส้นทางการสร้าง)

ทั้ง `WebSocketUpgrade` (ฝั่ง server) และ client ทั้ง `tokio-tungstenite` มาจากตระกูล crate เดียวกัน
(`tungstenite`) — Axum 0.8.9 พึ่งพา `tungstenite` ผ่าน dependency ภายในของมันเอง (คนละเวอร์ชันกับที่แอป client
เลือกใช้ก็ได้ Cargo จะจัดการให้ทั้งสองเวอร์ชันอยู่ร่วมกันได้ในเวลา compile เพราะมันแค่เป็น dependency ภายในที่
ไม่ leak type ออกมาให้ผู้ใช้ต้อง match เวอร์ชันตรงกัน) — `axum::extract::ws::Message` เป็น enum **ของ Axum เอง**
(ไม่ใช่ type เดียวกับ `tokio_tungstenite::tungstenite::Message` ที่ client ใช้ตรง ๆ) ที่ทำหน้าที่แปลงไปมากับ
`tungstenite::Message` ภายใน — เจตนาของการออกแบบแบบนี้คือ **กันไม่ให้แอปของคุณต้องผูก (couple) กับเวอร์ชันของ
`tungstenite` ที่ Axum เลือกใช้ภายในโดยตรง** ถ้า Axum เปลี่ยนเวอร์ชัน `tungstenite` ภายในในอนาคต โค้ดฝั่ง server
ของคุณที่ใช้ `axum::extract::ws::Message` จะไม่ได้รับผลกระทบเลย

### 77.4 Message Enum: `Text`, `Binary`, `Ping`, `Pong`, `Close`

`axum::extract::ws::Message` เป็น enum ที่มี 5 variant ครอบคลุมทุกประเภทของ WebSocket frame ตาม RFC 6455:

```rust
pub enum Message {
    Text(Utf8Bytes),           // ข้อความ text (การันตี valid UTF-8 เสมอ)
    Binary(Bytes),              // ข้อมูล binary ดิบ ๆ (รูปภาพ, protobuf, ฯลฯ)
    Ping(Bytes),                 // frame สำหรับตรวจสอบว่า connection ยังมีชีวิตอยู่
    Pong(Bytes),                 // การตอบกลับ Ping
    Close(Option<CloseFrame>),  // frame ที่บอกว่า "จะปิด connection แล้ว" พร้อม code/reason (ถ้ามี)
}
```

จุดที่น่าประหลาดใจที่สุดสำหรับคนที่เพิ่งเจอ WebSocket ครั้งแรก: **`Ping`/`Pong` ถูกจัดการโดยอัตโนมัติที่ระดับ
protocol ทั้งฝั่ง Axum (server) และ `tungstenite` (ที่ทั้ง server และ client ใช้เป็นแกนกลาง)** — คุณไม่ต้อง
เขียนโค้ด "เมื่อได้ Ping ให้ตอบ Pong" เองแม้แต่บรรทัดเดียว มันเกิดขึ้นที่ชั้นล่างกว่า loop ที่คุณเขียนเองเสียอีก

#### พิสูจน์ด้วยการรันจริง: server ไม่ตอบ Pong เอง แต่ Pong ก็มาอยู่ดี

เขียน server ที่ **ไม่มี logic ตอบ Pong เองเลยแม้แต่นิดเดียว** (แค่ log ว่าเห็น frame อะไรบ้าง):

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::Response,
    routing::any,
    Router,
};

async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        match msg {
            Message::Text(text) => {
                println!("[server] ได้ Text: {}", text.as_str());
                if socket.send(Message::Text(format!("echo: {text}").into())).await.is_err() {
                    break;
                }
            }
            Message::Binary(data) => {
                println!("[server] ได้ Binary: {} bytes", data.len());
                if socket.send(Message::Binary(data)).await.is_err() {
                    break;
                }
            }
            Message::Ping(payload) => {
                // ไม่มีโค้ดตอบ Pong เองที่นี่เลย -- log ไว้เฉย ๆ เพื่อพิสูจน์ว่า
                // frame ยัง "โผล่มาถึง" loop นี้ได้ (สำหรับ use case ที่ต้องการรู้ว่ามี ping มา
                // เช่นทำ custom keepalive tracking) แต่ Pong ที่ตอบกลับไปจริงเกิดที่ชั้นล่างกว่านี้
                println!("[server] เห็น Ping frame: {payload:?} (axum ตอบ Pong ให้เองแล้ว)");
            }
            Message::Pong(payload) => {
                println!("[server] เห็น Pong frame: {payload:?}");
            }
            Message::Close(frame) => {
                println!("[server] ได้ Close frame: {frame:?}");
                break;
            }
        }
    }
}

async fn ws_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}
```

แล้วเขียน client ที่ **ส่ง `Ping` เองตรง ๆ** (ปกติ browser/library ระดับสูงจะส่ง Ping ให้เองเป็นระยะเพื่อ
keepalive แต่ในตัวอย่างนี้เราส่งเองด้วยมือเพื่อดูปฏิกิริยาให้ชัด):

```rust
use bytes::Bytes;
use futures_util::{SinkExt, StreamExt};
use tokio_tungstenite::connect_async;
use tokio_tungstenite::tungstenite::Message as TMessage;

#[tokio::main]
async fn main() {
    let (mut ws_stream, _resp) = connect_async("ws://127.0.0.1:4002/ws").await.unwrap();

    ws_stream
        .send(TMessage::Ping(Bytes::from_static(b"keepalive-1")))
        .await
        .unwrap();
    let first = ws_stream.next().await.unwrap().unwrap();
    println!("[client] ส่ง Ping(\"keepalive-1\") ไป -- ได้กลับมา: {first:?}");
}
```

ผลลัพธ์จริงที่ capture มา (ฝั่ง client):

```
[client] ส่ง Ping("keepalive-1") ไป -- ได้กลับมา: Pong(b"keepalive-1")
```

และฝั่ง server (log ที่ capture มา):

```
[server] เห็น Ping frame: b"keepalive-1" (axum ตอบ Pong ให้เองแล้ว)
```

สังเกตให้ทะลุ: **client ได้ `Pong` กลับมาเป็น**หลังแรก**ก่อนอะไรอื่นทั้งหมด แม้ว่า `match` arm ของ
`Message::Ping` ฝั่ง server จะไม่มีโค้ดสั่ง `socket.send(Message::Pong(...))` เลยแม้แต่บรรทัดเดียว** — นี่คือ
พฤติกรรมที่เกิดที่ชั้น `tungstenite` protocol layer ซึ่งอยู่**ต่ำกว่า**loop ที่คุณเห็นและเขียนเองอีกที (เมื่อ
`tungstenite` อ่าน frame `Ping` ออกมาจาก TCP stream มันจะคิว frame `Pong` ตอบกลับโดยอัตโนมัติทันที ก่อนที่
frame `Ping` นั้นจะถูกส่งขึ้นมาให้ loop ของคุณเห็นด้วยซ้ำ) — เหตุผลที่ยังมี `Message::Ping`/`Message::Pong`
โผล่มาให้ match ได้อยู่ดีคือเพื่อเปิดทางให้แอปที่ต้องการ **custom keepalive tracking** (เช่นนับว่านานเท่าไหร่
แล้วที่ไม่ได้รับ Pong จาก client เพื่อตัดสินใจปิด connection ที่ค้างนิ่งเอง) ทำได้โดยไม่ต้องยุ่งกับการตอบ
protocol-level Pong ที่ Axum/tungstenite จัดการให้แล้วอยู่ดี — นี่คือความแตกต่างสำคัญกับหลายภาษา/library อื่น
ที่บาง implementation ต้องเขียนโค้ดตอบ Pong เองเสมอ

การรับ `Text`/`Binary`/`Close` ในตัวอย่างเดียวกันก็ยืนยันครบ: ส่ง `Text("hello")` ได้ `Text("echo: hello")`
กลับมา, ส่ง `Binary([1,2,3,4])` ได้ `Binary([1,2,3,4])` กลับมา (เพราะโค้ด server ส่งกลับตรง ๆ), และ Close
frame ก็ log ออกมาถูกต้องว่า `Close: None` (ไม่มี close code/reason ระบุมา) แล้ว loop จบทันทีตามที่ `break`
สั่งไว้ในโค้ด

### 77.5 แยก Socket เพื่ออ่าน-เขียนพร้อมกัน: `.split()`

ตัวอย่างทั้งหมดที่ผ่านมาใน 77.3-77.4 มีข้อจำกัดสำคัญ: **มันตอบสนองข้อความของ client เท่านั้น** — server ส่ง
อะไรกลับไปได้ก็ต่อเมื่อ client ส่งข้อความมาก่อนเสมอ (`while let Some(Ok(msg)) = socket.recv().await { ...
socket.send(...) }`) แต่ในสถานการณ์จริงจำนวนมาก (รวมถึง capstone ท้ายบทนี้) **server ต้องส่งข้อความไปให้
client ได้เองโดยไม่ต้องรอให้ client ถามก่อน** เช่น แจ้งอัปเดตจำนวนที่นั่งที่เปลี่ยนไปเพราะคนอื่นซื้อตั๋ว —
event นั้นไม่ได้เกิดจาก message ที่ client คนนี้ส่งมาเลย

`WebSocket` (ที่ `handle_socket` ได้รับมา) implement ทั้ง `Stream` (อ่านได้) และ `Sink` (เขียนได้) จาก crate
`futures_util` (ที่ Part 46-50 อาจแนะนำผ่าน ๆ มาแล้วตอนพูดถึง `Future`/`Stream`) — method `.split()` แยก
`WebSocket` ตัวเดียวออกเป็น **`SplitSink`** (เขียนได้อย่างเดียว) กับ **`SplitStream`** (อ่านได้อย่างเดียว) ที่
เป็นอิสระจากกัน — พอแยกแล้ว สามารถ `tokio::spawn` งานเขียนให้รันเป็น task แยกออกไปจากงานอ่านได้ (ต่อยอดจาก
Part 39/48 เรื่อง task concurrency ตรง ๆ) ทำให้ทั้งสองฝั่งทำงานพร้อมกันได้อย่างแท้จริง โดยไม่ต้องรอกันเลย:

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    response::Response,
    routing::any,
    Router,
};
use futures_util::{SinkExt, StreamExt};
use std::time::Duration;

async fn handle_socket(socket: WebSocket) {
    // แยก WebSocket ตัวเดียวออกเป็นสองครึ่งที่เป็นอิสระจากกัน
    let (mut sender, mut receiver) = socket.split();

    // Task ที่ 1: "ดัน" ข้อความจากฝั่ง server ไปให้ client เป็นระยะ ๆ
    // โดยไม่สนใจเลยว่า client กำลังส่งอะไรมาหรือไม่ในเวลาเดียวกัน
    let mut push_task = tokio::spawn(async move {
        let mut tick: u32 = 0;
        loop {
            tokio::time::sleep(Duration::from_millis(150)).await;
            tick += 1;
            let text = format!("[server push] tick #{tick}");
            if sender.send(Message::Text(text.into())).await.is_err() {
                break;
            }
        }
    });

    // Task หลัก (ยังอยู่บน task ของ connection นี้เอง): อ่านข้อความจาก client แล้ว log ทันที
    // สังเกตว่า loop นี้ "อ่านอย่างเดียว" ไม่ต้อง .send() อะไรกลับเองในนี้เลย
    // -- ถ้าจะตอบกลับ client ก็ยังต้องผ่าน sender ที่ push_task ถือไปแล้ว (ดูข้อสังเกตด้านล่าง)
    let mut recv_count = 0;
    while let Some(Ok(msg)) = receiver.next().await {
        if let Message::Text(text) = msg {
            recv_count += 1;
            println!("[server recv] ข้อความที่ {recv_count} จาก client: {text}");
        }
        if recv_count >= 2 {
            break;
        }
    }

    push_task.abort();
    let _ = push_task.await;
    println!("[server] จบ connection นี้แล้ว");
}

async fn ws_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}
```

รัน client ที่อ่าน push message สองอันแรกก่อนโดยไม่ส่งอะไรไปเลย แล้วค่อยส่งข้อความของตัวเองสองครั้ง:

```rust
use futures_util::{SinkExt, StreamExt};
use tokio_tungstenite::connect_async;
use tokio_tungstenite::tungstenite::Message as TMessage;

#[tokio::main]
async fn main() {
    let (mut ws_stream, _resp) = connect_async("ws://127.0.0.1:4003/ws").await.unwrap();

    for _ in 0..2 {
        let msg = ws_stream.next().await.unwrap().unwrap();
        println!("[client] ได้รับ push ระหว่างที่ยังไม่ได้ส่งอะไรไปเลย: {msg:?}");
    }

    ws_stream.send(TMessage::Text("จาก client ครั้งที่ 1".into())).await.unwrap();
    ws_stream.send(TMessage::Text("จาก client ครั้งที่ 2".into())).await.unwrap();
}
```

ผลลัพธ์จริงฝั่ง client:

```
[client] ได้รับ push ระหว่างที่ยังไม่ได้ส่งอะไรไปเลย: Text(Utf8Bytes(b"[server push] tick #1"))
[client] ได้รับ push ระหว่างที่ยังไม่ได้ส่งอะไรไปเลย: Text(Utf8Bytes(b"[server push] tick #2"))
```

ผลลัพธ์จริงฝั่ง server:

```
[server recv] ข้อความที่ 1 จาก client: จาก client ครั้งที่ 1
[server recv] ข้อความที่ 2 จาก client: จาก client ครั้งที่ 2
[server] จบ connection นี้แล้ว
```

พิสูจน์ชัดเจน: **`push_task` ยังเดินต่อ (ส่ง tick ทุก 150ms) โดยไม่สนใจเลยว่า main loop กำลังรออ่านข้อความจาก
client อยู่หรือไม่** — และ main loop ก็อ่านข้อความของ client ได้ครบถูกต้องโดยไม่ต้องรอ `push_task` เลยเช่นกัน
ทั้งสอง task ทำงานคู่ขนานอย่างแท้จริงบน connection เดียวกัน (คนละ task ที่ Tokio scheduler สลับกันรันบน worker
thread pool เดียวกัน ตามหลักการที่ Part 48 สอนไว้)

**ข้อสังเกตเชิงออกแบบที่สำคัญ**: หลังจาก `.split()` แล้ว **`sender` ถูกย้ายเข้าไปอยู่ใน `push_task` เพียง
ผู้เดียว** — ถ้า main loop (ที่ถือ `receiver`) ต้องการตอบกลับ client ด้วย ต้องมีช่องทางส่งคำขอไปให้
`push_task` ส่งแทน (ปกติผ่าน `mpsc` channel — Part 38/50) ไม่สามารถ `sender.send(...)` ตรง ๆ จาก main loop
ได้อีกเพราะ ownership ของ `sender` ถูกย้ายไปแล้ว — นี่คือเหตุผลที่ pattern การใช้งานจริง (หัวข้อ 77.6-77.7)
มักจะมี **channel กลาง** (broadcast หรือ mpsc) ที่ทั้งสอง task คุยกันผ่านช่องทางนั้นแทนที่จะพยายามแบ่ง
`sender` กันใช้ตรง ๆ

### 77.6 Broadcast Pattern: ห้องแชทด้วย `tokio::sync::broadcast`

ตอนนี้มาประกอบทุกอย่างที่เรียนมาเข้าด้วยกัน: `WebSocketUpgrade` (77.3), `.split()` (77.5), `AppState` (Part
64), และ `tokio::sync::broadcast` (Part 50 หัวข้อ 50.10) เพื่อสร้าง**ห้องแชทที่ทุกคนที่เชื่อมต่ออยู่เห็น
ข้อความของทุกคนคนอื่นแบบ real-time** — เลือกใช้ตัวอย่างห้องแชทธรรมดาก่อนในหัวข้อนี้ (ไม่ใช่ระบบตั๋วที่เป็น
theme หลักของหลักสูตร) เพราะมันเป็น **use case ที่บริสุทธิ์ที่สุดของ broadcast**: ทุกคนในห้องต้องเห็นข้อความ
เดียวกันทุกข้อความไม่มีข้อยกเว้น ไม่ต้องมีการกรอง/แบ่งกลุ่ม — เก็บความซับซ้อนของ "กรองข้อความตาม event ที่
subscribe" ไว้ให้กับ capstone ท้ายบท (77.11) ที่จะนำแนวคิดเดียวกันนี้ไปต่อยอดกับ theme ระบบตั๋วแทน

จาก Part 50 เราเรียนมาแล้วว่า `broadcast::channel::<T>(capacity)` คืน `(Sender<T>, Receiver<T>)` ที่
`Sender` clone ได้หลายตัว, `Receiver` เกิดใหม่ได้ผ่าน `.subscribe()` และ **ทุก receiver ที่ subscribe ไว้จะ
ได้รับข้อความทุกข้อความที่ถูกส่งหลังจากตัวเอง subscribe** (ไม่ใช่แค่ผู้รับคนเดียวแบบ `mpsc`) — เก็บ
`broadcast::Sender<String>` ไว้ใน `AppState` แบบเดียวกับที่ Part 64 สอนไว้เรื่อง state ที่แชร์ข้าม handler:

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    extract::State,
    response::Response,
    routing::any,
    Router,
};
use futures_util::{SinkExt, StreamExt};
use tokio::sync::broadcast;

#[derive(Clone)]
struct AppState {
    tx: broadcast::Sender<String>,
}

async fn handle_socket(socket: WebSocket, state: AppState) {
    let (mut sender, mut receiver) = socket.split();
    // แต่ละ connection subscribe เข้าไปรับข้อความของตัวเอง -- ทุก connection ได้ Receiver แยกกัน
    // แต่ทุกตัวรับข้อความเดียวกันจาก Sender ตัวเดียวกันใน AppState
    let mut rx = state.tx.subscribe();

    // Task เขียน: ฟังข้อความจาก broadcast channel แล้วส่งให้ client ตัวเองผ่าน sender
    let mut send_task = tokio::spawn(async move {
        while let Ok(text) = rx.recv().await {
            if sender.send(Message::Text(text.into())).await.is_err() {
                break;
            }
        }
    });

    // Task อ่าน: รับข้อความจาก client ตัวเองแล้วส่งเข้า broadcast channel ให้ทุกคน (รวมตัวเอง) เห็น
    let tx = state.tx.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(Message::Text(text))) = receiver.next().await {
            let _ = tx.send(text.to_string());
        }
    });

    // ถ้า task ใด task หนึ่งจบ (client หลุด หรือ error) ให้ยกเลิกอีก task ที่เหลือทันที
    // ไม่ปล่อยให้ task ที่เหลือค้างทำงานต่อไปโดยไม่มีประโยชน์ (ทั้งคู่ผูกกับ connection เดียวกัน)
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }
}

async fn ws_handler(ws: WebSocketUpgrade, State(state): State<AppState>) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

#[tokio::main]
async fn main() {
    let (tx, _rx) = broadcast::channel::<String>(16);
    let state = AppState { tx };

    let app = Router::new()
        .route("/chat", any(ws_handler))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4004").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

ทดสอบด้วย client 2 ตัวที่รันพร้อมกันจริง (alice ส่งข้อความ, bob แค่ฟัง):

```rust
use futures_util::{SinkExt, StreamExt};
use tokio_tungstenite::connect_async;
use tokio_tungstenite::tungstenite::Message as TMessage;

#[tokio::main]
async fn main() {
    let name = std::env::args().nth(1).unwrap_or_else(|| "guest".to_string());
    let (mut ws_stream, _resp) = connect_async("ws://127.0.0.1:4004/chat").await.unwrap();

    if name == "alice" {
        tokio::time::sleep(std::time::Duration::from_millis(200)).await;
        ws_stream
            .send(TMessage::Text(format!("[{name}] สวัสดีทุกคนในห้องแชท").into()))
            .await
            .unwrap();
    }

    if let Some(Ok(TMessage::Text(text))) = ws_stream.next().await {
        println!("[{name}] ได้รับข้อความในห้องแชท: {text}");
    }
}
```

รัน `bob` ก่อน (ปล่อยให้รอฟังอย่างเดียว) แล้วรัน `alice` ตามหลังไม่นาน — ผลลัพธ์จริงที่ capture มา:

```
[alice] ได้รับข้อความในห้องแชท: [alice] สวัสดีทุกคนในห้องแชท
[bob] ได้รับข้อความในห้องแชท: [alice] สวัสดีทุกคนในห้องแชท
```

**ทั้ง `alice` และ `bob` ได้รับข้อความเดียวกัน** — พิสูจน์ว่า `broadcast` กระจายข้อความให้ทุก subscriber
จริง สังเกตจุดที่น่าสนใจ: **`alice` ได้รับข้อความของตัวเองกลับมาด้วย** (ไม่ใช่แค่ bob) เพราะโค้ดในตัวอย่างนี้
ให้ `recv_task` ส่งข้อความที่ได้จาก client เข้า broadcast channel แบบไม่แยกแยะว่าใครเป็นคนส่ง แล้ว
`send_task` ของทุก connection (รวมถึง connection ของ alice เอง) ก็ subscribe รับข้อความเดียวกันหมด — นี่คือ
พฤติกรรมที่ถูกต้องสำหรับห้องแชทธรรมดา (client ฝั่ง UI ส่วนใหญ่จะ render ข้อความของตัวเองจาก input ที่พิมพ์ไป
ตรง ๆ อยู่แล้ว ไม่ต้องรอ echo กลับมา) แต่ถ้าต้องการ **ไม่ส่ง echo กลับไปให้ผู้ส่งเอง** ก็ทำได้ง่าย ๆ ด้วยการ
แนบ "ผู้ส่ง" (เช่น `ClientId` จากหัวข้อถัดไป) ไปกับข้อความที่ broadcast แล้วให้แต่ละ connection เช็คว่า
ข้อความนั้นมาจากตัวเองหรือไม่ก่อนส่งให้ client (จะเห็นเทคนิคการกรองแบบนี้เต็มรูปแบบใน capstone 77.11 ที่กรอง
ตาม `event_id` แทน)

### 77.7 Per-Connection State: ส่งข้อความเจาะจงถึง Client รายบุคคล

`broadcast` เหมาะกับ "ส่งให้ทุกคน" แต่สถานการณ์จริงจำนวนมากต้องการ **ส่งข้อความไปยัง client ที่ระบุตัวได้
เจาะจงเพียงคนเดียว** — เช่น "แจ้งเตือนส่วนตัวถึงผู้ใช้ที่ id เท่านี้เท่านั้น" ไม่ใช่ broadcast ให้ทุกคน วิธี
แก้คือเก็บ **map จาก client id ไปยังช่องทางส่งข้อความของ connection นั้น** ไว้ใน `AppState` — ให้แต่ละ
connection ที่เชื่อมต่อสำเร็จลงทะเบียนตัวเองเข้า map นี้ (พร้อม `mpsc::UnboundedSender` ของตัวเอง)

จุดที่น่าสนใจเชิงการตัดสินใจ (เชื่อมกับกฎของ Part 50 หัวข้อ 50.6 ตรง ๆ): **critical section ที่ล็อก map ตัวนี้
มีแค่ `insert`/`remove`/`get` — ไม่มี `.await` ข้างในเลยแม้แต่จุดเดียว** ดังนั้นตามกฎการตัดสินใจของ Part 50
**`std::sync::Mutex` ยังเหมาะสมกว่า** ไม่จำเป็นต้องใช้ `tokio::sync::Mutex` เลย:

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    extract::{Path, State},
    response::Response,
    routing::{any, post},
    Router,
};
use futures_util::{SinkExt, StreamExt};
use std::collections::HashMap;
use std::sync::{
    atomic::{AtomicU64, Ordering},
    Arc, Mutex,
};
use tokio::sync::mpsc;

type ClientId = u64;

#[derive(Clone)]
struct AppState {
    next_id: Arc<AtomicU64>,
    // critical section ของ Mutex ตัวนี้ไม่มี .await ข้างในเลย -> std::sync::Mutex เหมาะสมกว่า
    // ตามกฎของ Part 50 หัวข้อ 50.6 ("ดูที่ critical section ไม่ใช่ดูที่บริบทรอบข้าง")
    clients: Arc<Mutex<HashMap<ClientId, mpsc::UnboundedSender<String>>>>,
}

async fn handle_socket(socket: WebSocket, state: AppState) {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);
    let (mut sender, mut receiver) = socket.split();
    let (tx, mut rx) = mpsc::unbounded_channel::<String>();

    // ลงทะเบียน sender ของ connection นี้เข้า map กลาง ก่อนอื่นใด
    state.clients.lock().unwrap().insert(id, tx);
    let _ = sender.send(Message::Text(format!("YOUR_ID:{id}").into())).await;

    let mut writer = tokio::spawn(async move {
        while let Some(text) = rx.recv().await {
            if sender.send(Message::Text(text.into())).await.is_err() {
                break;
            }
        }
    });

    while let Some(Ok(msg)) = receiver.next().await {
        if let Message::Close(_) = msg {
            break;
        }
    }

    writer.abort();
    state.clients.lock().unwrap().remove(&id);
}

async fn ws_handler(ws: WebSocketUpgrade, State(state): State<AppState>) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

// REST endpoint สไตล์ Part 63 -- ส่งข้อความส่วนตัวไปยัง client ที่ id ที่ระบุเท่านั้น
async fn notify(
    State(state): State<AppState>,
    Path(target_id): Path<ClientId>,
    body: String,
) -> String {
    let clients = state.clients.lock().unwrap();
    match clients.get(&target_id) {
        Some(tx) => {
            let _ = tx.send(format!("PRIVATE: {body}"));
            format!("ส่งข้อความถึง client #{target_id} แล้ว")
        }
        None => format!("ไม่พบ client #{target_id} (อาจหลุดการเชื่อมต่อไปแล้ว)"),
    }
}

#[tokio::main]
async fn main() {
    let state = AppState {
        next_id: Arc::new(AtomicU64::new(1)),
        clients: Arc::new(Mutex::new(HashMap::new())),
    };
    let app = Router::new()
        .route("/ws", any(ws_handler))
        .route("/notify/{id}", post(notify))
        .with_state(state);
    let listener = tokio::net::TcpListener::bind("127.0.0.1:4005").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

ทดสอบจริง: เชื่อมต่อ client 2 ตัว (ได้ id 1 และ 2 ตามลำดับ) แล้วยิง REST `POST /notify/2` (ไม่ใช่ WebSocket
เลย — เป็น HTTP request ปกติสไตล์ Part 63):

```
[client 1] Text(Utf8Bytes(b"YOUR_ID:1"))
[client 2] Text(Utf8Bytes(b"YOUR_ID:2"))
[REST] response:
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 62

ส่งข้อความถึง client #2 แล้ว
[client 2] ข้อความที่ได้รับเพิ่ม: Ok(Some(Ok(Text(Utf8Bytes(b"PRIVATE: ข้อความส่วนตัวถึง client 2 เท่านั้น")))))
[client 1] ข้อความที่ได้รับเพิ่ม (คาดว่าจะ timeout เพราะไม่ใช่เป้าหมาย): Err(Elapsed(()))
```

พิสูจน์ชัดเจน: **client #2 ได้รับข้อความส่วนตัว ในขณะที่ client #1 ไม่ได้รับอะไรเลย** (รอจน timeout ที่ตั้งไว้
500ms ก็ยัง `Err(Elapsed(()))` เพราะไม่มีข้อความมาให้อ่านจริง ๆ) — นี่คือความต่างสำคัญกับหัวข้อ 77.6:
`broadcast` ส่งให้ **ทุกคน** เสมอ ในขณะที่ pattern ของ `HashMap<ClientId, Sender>` ให้ **เลือกเจาะจงคนใดคนหนึ่ง
ได้** — ทั้งสองแบบมีที่ใช้ต่างกัน และแอปจริงจำนวนมาก (รวมถึง capstone ท้ายบท) ใช้**ทั้งสองแบบร่วมกัน**: ใช้
`broadcast` สำหรับข้อมูลที่ทุกคนต้องรู้เหมือนกัน (เช่นจำนวนที่นั่งคงเหลือ) และใช้ per-connection map สำหรับ
ข้อความที่เจาะจงคนเดียว (เช่นแจ้งเตือนส่วนตัว, หรือแจ้ง `YOUR_ID` ตอน connect อย่างที่เห็นในตัวอย่างนี้)

log ฝั่ง server ยืนยันวงจรชีวิตของ connection ครบถ้วน:

```
[server] client #1 เชื่อมต่อแล้ว (รวมทั้งหมดตอนนี้ 1 คน)
[server] client #2 เชื่อมต่อแล้ว (รวมทั้งหมดตอนนี้ 2 คน)
[server] client #2 หลุดการเชื่อมต่อ ลบออกจาก map แล้ว
[server] client #1 หลุดการเชื่อมต่อ ลบออกจาก map แล้ว
```

### 77.8 Authentication ตอน Upgrade: JWT/Session + ข้อจำกัดของ Browser

WebSocket handshake ยังคือ HTTP request ธรรมดา (77.2) — แปลว่าระบบ authentication ที่สร้างไว้แล้วใน Part 74
(JWT) หรือ Part 75 (session/cookie) สามารถทำงานได้ตอน handshake นี้เช่นกัน **ก่อนที่จะยอมรับ (accept) การ
อัปเกรด** — ปฏิเสธ handshake ตั้งแต่ต้นถ้า credential ไม่ถูกต้อง ดีกว่าปล่อยให้อัปเกรดสำเร็จแล้วค่อยเช็คทีหลัง
(เพราะ handshake ที่สำเร็จแล้วปิดยากกว่าการปฏิเสธด้วย HTTP status code ธรรมดา)

แต่มีข้อจำกัดจริงที่ต้องรู้ก่อนออกแบบ: **JavaScript `WebSocket` API มาตรฐานของ browser (`new
WebSocket(url, protocols)`) ไม่มีพารามิเตอร์ให้กำหนด custom HTTP header ได้เลย** — รับแค่ URL กับรายชื่อ
subprotocol เท่านั้น (ต่างจาก `fetch()`/`XMLHttpRequest` ที่ตั้ง header `Authorization: Bearer <token>` ได้
ตรง ๆ ตามที่ Part 74 สอนไว้) นี่ไม่ใช่ข้อจำกัดของ Axum หรือ Rust เลย — เป็นข้อจำกัดของ **spec ของ browser เอง**
ที่ยังไม่มีทางออกอย่างเป็นทางการจนถึงปัจจุบัน

วิธีแก้ที่ใช้กันจริงในโลกจริงมีสองทางหลัก:

1. **ส่ง token ผ่าน query parameter** (`wss://api.example.com/ws?token=<jwt>`) — ง่ายที่สุด ทำงานได้กับ
   `new WebSocket(url)` ตรง ๆ แต่มีข้อเสียด้านความปลอดภัยที่ต้องรู้: URL (รวม query string) มักถูก log ไว้ใน
   access log ของ server, proxy, และ browser history — ถ้า token หลุดไปอยู่ใน log พวกนี้ อาจถูกขโมยใช้ซ้ำได้
   ทางแก้บางส่วนคือให้ token มีอายุสั้นมาก (เช่นออก token ชั่วคราวเฉพาะสำหรับ WebSocket handshake ที่ต่างจาก
   access token หลักและหมดอายุเร็วมาก)
2. **ใช้ session cookie** (Part 75) — เพราะ browser **ส่ง cookie ไปกับทุก HTTP request ไปยัง origin เดียวกัน
   โดยอัตโนมัติเสมอ รวมถึง WebSocket handshake request ด้วย** (cookie ไม่ได้ผูกกับ "API ที่ใช้เขียนโค้ด" แต่
   ผูกกับ "HTTP request ที่เบราว์เซอร์ส่งออกไปจริง" — และ handshake ก็คือ HTTP request ธรรมดาตามที่ 77.2
   พิสูจน์ไว้) วิธีนี้ปลอดภัยกว่าเพราะไม่มี token ปรากฏใน URL เลย แต่ต้องระวังเรื่อง CSRF ที่ Part 75 อธิบายไว้
   (WebSocket handshake ก็เสี่ยงต่อ Cross-Site WebSocket Hijacking ได้เหมือนกันถ้าไม่ตรวจสอบ `Origin` header)

ตัวอย่างนี้ใช้แนวทางที่ 1 (query parameter) เพราะเขียนสาธิตง่ายที่สุดโดยไม่ต้องพึ่ง cookie jar ของ browser
จริง — ในระบบจริงคุณจะเรียก `jsonwebtoken::decode(...)` แบบที่ Part 74 สอนไว้เต็มรูปแบบ ตัวอย่างนี้ย่อฟังก์ชัน
ตรวจสอบให้ง่ายลงเพื่อโฟกัสที่ประเด็นของบทนี้ (การปฏิเสธ/อนุมัติตอน upgrade):

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    extract::Query,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::any,
    Router,
};
use serde::Deserialize;

#[derive(Deserialize)]
struct AuthQuery {
    token: Option<String>,
}

// ในระบบจริงเรียก jsonwebtoken::decode(...) ตาม Part 74 เต็มรูปแบบ -- ย่อไว้ให้ง่ายในตัวอย่างนี้
fn validate_token(token: &str) -> Option<String> {
    if token == "valid-token-abc123" {
        Some("phutjirakul".to_string())
    } else {
        None
    }
}

async fn handle_socket(mut socket: WebSocket, username: String) {
    let _ = socket
        .send(Message::Text(format!("ยินดีต้อนรับ {username} เข้าสู่ระบบแบบ authenticated แล้ว").into()))
        .await;
    while socket.recv().await.is_some() {}
}

// สังเกตลำดับ parameter: WebSocketUpgrade และ Query<T> ทั้งคู่เป็น FromRequestParts (Part 64 หัวข้อ 64.5)
// จึงผสมกันได้อย่างอิสระไม่ต้องเรียงลำดับพิเศษ (ต่างจาก Json<T> ที่ต้องมาตัวสุดท้ายเสมอ)
async fn ws_handler(ws: WebSocketUpgrade, Query(query): Query<AuthQuery>) -> Response {
    let token = match query.token {
        Some(t) => t,
        None => return (StatusCode::UNAUTHORIZED, "ต้องใส่ ?token=... มาด้วยเสมอ").into_response(),
    };

    match validate_token(&token) {
        // อนุมัติ upgrade เฉพาะเมื่อ token ถูกต้องแล้วเท่านั้น -- ปฏิเสธก่อนเข้าสู่ WebSocket เลย
        Some(username) => ws.on_upgrade(move |socket| handle_socket(socket, username)),
        None => (StatusCode::UNAUTHORIZED, "token ไม่ถูกต้องหรือหมดอายุ").into_response(),
    }
}
```

ทดสอบจริง 3 กรณี:

```
===== กรณีที่ 1: ไม่ส่ง token มาเลย =====
HTTP/1.1 401 Unauthorized
content-type: text/plain; charset=utf-8
content-length: 63

ต้องใส่ ?token=... มาด้วยเสมอ

===== กรณีที่ 2: ส่ง token ผิด =====
HTTP/1.1 401 Unauthorized
content-type: text/plain; charset=utf-8
content-length: 69

token ไม่ถูกต้องหรือหมดอายุ

===== กรณีที่ 3: ส่ง token ถูกต้อง =====
handshake status: 101 Switching Protocols
ได้รับ: Text(Utf8Bytes(b"ยินดีต้อนรับ phutjirakul เข้าสู่ระบบแบบ authenticated แล้ว"))
```

พิสูจน์ชัดเจน: **การเชื่อมต่อที่ไม่มี token หรือ token ผิด ไม่มีทางไปถึงขั้น `101 Switching Protocols` เลย** —
มันถูกปฏิเสธที่ระดับ HTTP status code ธรรมดา (`401 Unauthorized`) ตั้งแต่ก่อนที่ TCP connection จะถูกอัปเกรด
เป็น WebSocket ด้วยซ้ำ — ฝั่ง client ที่พยายาม `connect_async` ด้วย token ผิด (หรือไม่มี) จะได้ error กลับมา
ทันทีจาก `connect_async` เอง (เพราะมันตรวจสอบว่า response เป็น `101` หรือไม่ ถ้าไม่ใช่จะคืน `Err` แทนที่จะ
คืน `WebSocketStream` มา) — นี่คือรูปแบบการปฏิเสธที่ถูกต้องและปลอดภัยที่สุด: **ปฏิเสธที่จุดเดียวกันกับที่ REST
API ปฏิเสธ request ที่ไม่มี auth** (status code เดียวกัน, กลไกเดียวกัน) ไม่ต้องสร้างกลไก error handling
แยกต่างหากสำหรับ WebSocket โดยเฉพาะเลย

### 77.9 Disconnect อย่างสมบูรณ์: ตรวจจับและเก็บกวาดด้วย RAII

ทุกตัวอย่างที่ผ่านมามี logic ลบ client ออกจาก state ตอนจบ connection กระจายอยู่ (`state.clients.lock()
.unwrap().remove(&id)` ในหัวข้อ 77.7 เป็นต้น) — แต่โค้ดแบบนั้นมีความเสี่ยง: **ถ้ามีจุด `return` หรือ `break`
ใหม่ ๆ ถูกเพิ่มเข้ามาทีหลัง (หรือแย่กว่านั้นคือ panic เกิดขึ้นกลางทาง) แล้วลืมเรียก `.remove()` ตรงนั้น** client
นั้นจะ**ค้างอยู่ใน map ตลอดไป** ทั้งที่ connection จริงหลุดไปแล้ว (memory leak เชิงตรรกะที่ตรวจจับยากมาก เพราะ
โปรแกรมไม่ crash แค่ state ผิดเพี้ยนไปเรื่อย ๆ)

Part 6 สอนหลักการที่แก้ปัญหานี้ได้สมบูรณ์: **RAII (Resource Acquisition Is Initialization)** ผ่าน `Drop`
trait — สร้าง struct guard ตัวหนึ่งที่ตอนสร้าง (`new`) บอกว่า "resource นี้ถูกครอบครองแล้ว" และ implement
`Drop::drop` ให้ปลดปล่อย resource นั้นอัตโนมัติ **ไม่ว่า scope จะจบลงด้วยเหตุผลอะไรก็ตาม** (`return` ปกติ,
`break`, `?` ที่ propagate error ออกไป, หรือแม้แต่ **panic ที่ unwind ผ่าน scope นั้น**) — Rust การันตีว่า
`Drop::drop` ของทุกตัวแปรที่ยังอยู่ใน scope จะถูกเรียกเสมอตอน stack unwinding ระหว่าง panic (ยกเว้นกรณีที่
ตั้งค่า panic strategy เป็น `abort` ซึ่งไม่ unwind เลย — แต่ค่าเริ่มต้นของ Rust คือ `unwind`)

```rust
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

type ClientId = u64;

// RAII guard: ตราบใดที่ struct นี้ยังไม่ถูก drop แปลว่า client ตัวนี้ยัง "นับว่าเชื่อมต่ออยู่"
// Drop::drop ของมันถูกเรียกเสมอไม่ว่า task ของ connection นี้จะจบด้วย return ปกติ, break, หรือ panic
struct ConnectionGuard {
    id: ClientId,
    connected: Arc<Mutex<HashSet<ClientId>>>,
}

impl Drop for ConnectionGuard {
    fn drop(&mut self) {
        self.connected.lock().unwrap().remove(&self.id);
        println!("[server] Drop::drop ทำงาน -- ลบ client #{} ออกจาก set แล้ว", self.id);
    }
}
```

ใช้งานจริงในตัว handler — สร้าง guard ทันทีหลัง insert แล้วไม่ต้องเรียก `.remove()` ด้วยมืออีกเลยตลอด
ฟังก์ชัน:

```rust
async fn handle_socket(mut socket: WebSocket, state: AppState) {
    let id = state.next_id.fetch_add(1, Ordering::SeqCst);
    state.connected.lock().unwrap().insert(id);

    // guard ถูกสร้างตรงนี้ -- ไม่ว่าฟังก์ชันนี้จะ return ทางไหนก็ตาม (แม้ panic!)
    // Drop::drop จะทำงานตอนหลุด scope เสมอ ลบ id ออกจาก set ให้อัตโนมัติ
    let _guard = ConnectionGuard { id, connected: state.connected.clone() };

    loop {
        match socket.recv().await {
            Some(Ok(Message::Close(_))) => break, // client ปิดแบบสุภาพด้วย Close frame
            Some(Ok(_)) => continue,
            Some(Err(_)) => break, // เจอ protocol error (เช่น connection reset กลางทาง)
            None => break,          // stream จบแล้ว (recv คืน None)
        }
    }
    // _guard หลุด scope ตรงนี้ -> Drop::drop ทำงานทันที ไม่ว่าจะมาถึงจุดนี้ผ่านทาง break ไหน
}
```

ทดสอบจริงด้วยการปิด connection **สองแบบต่างกัน** เพื่อพิสูจน์ว่า guard ทำงานถูกต้องทั้งคู่: (1) drop
`WebSocketStream` ทิ้งตรง ๆ โดยไม่ส่ง Close frame (จำลอง client ที่ปิดโปรแกรม/เน็ตหลุดกลางทาง) และ (2) เรียก
`.close()` แบบสุภาพ:

```
ก่อนเชื่อมต่อ: /count = 0
หลังจากเชื่อมต่อ 2 client: /count = 2
หลังจาก client 1 หลุดกลางทาง: /count = 1
หลังจาก client 2 ปิดแบบสุภาพ: /count = 0
```

และ log ฝั่ง server ที่ยืนยันว่าทั้งสองกรณีถูกจัดการถูกต้องผ่าน error/branch ที่ต่างกัน แต่ guard ทำงานเหมือน
กันทั้งคู่:

```
[server] client #1 เชื่อมต่อ (ตอนนี้ 1 คน)
[server] client #2 เชื่อมต่อ (ตอนนี้ 2 คน)
[server] client #1 เจอ error ตอนอ่าน: WebSocket protocol error: Connection reset without closing handshake
[server] Drop::drop ทำงาน -- ลบ client #1 ออกจาก set แล้ว (เหลือ 1 คน)
[server] client #2 ส่ง Close frame มาแบบสุภาพ
[server] Drop::drop ทำงาน -- ลบ client #2 ออกจาก set แล้ว (เหลือ 0 คน)
```

สังเกตความต่างที่สำคัญของทั้งสองกรณี: **client #1 (หลุดกลางทางแบบไม่บอกลา)** ทำให้ `socket.recv()` คืน
`Some(Err(...))` พร้อม error message จริง **`"WebSocket protocol error: Connection reset without closing
handshake"`** (TCP connection ถูกปิดโดยไม่มี WebSocket Close frame ส่งมาก่อน — tungstenite มองว่านี่คือ
protocol violation ที่ควรรายงานเป็น error ไม่ใช่แค่ "จบปกติ") ในขณะที่ **client #2 (ปิดแบบสุภาพด้วย
`.close()`)** ทำให้ `socket.recv()` คืน `Some(Ok(Message::Close(_)))` ตามที่กำหนดไว้ใน RFC 6455 — ทั้งสอง
กรณีจบลงด้วย `break` ที่ต่างสาเหตุกัน แต่ **`ConnectionGuard` ไม่สนใจว่า `break` มาจากสาเหตุไหน** มันแค่รอ
ให้ scope หลุดแล้วทำงานเหมือนกันเป๊ะทุกครั้ง — นี่คือพลังของ RAII: แยก **"logic ตรวจจับสาเหตุที่จบ"**
ออกจาก **"logic การเก็บกวาด"** อย่างสมบูรณ์ ทำให้เพิ่มจุดจบใหม่ ๆ ในอนาคต (เช่น timeout, custom error ใหม่)
โดยไม่ต้องแก้ไข cleanup logic เลยแม้แต่นิดเดียว

### 77.10 Error Handling และ Panic ภายใน WebSocket Task

หัวข้อ 77.9 พิสูจน์ไปแล้วว่า RAII guard เก็บกวาด state ได้ถูกต้องแม้ connection จะจบแบบผิดปกติ — แต่ยังมี
คำถามที่สำคัญกว่านั้นอีกขั้น: **ถ้าโค้ดข้างในตัว handler ของ WebSocket connection หนึ่ง panic ขึ้นมาจริง ๆ
(บั๊กจริง เช่น `.unwrap()` บน `Option` ที่ดันเป็น `None`, index เกินขอบ array) จะเกิดอะไรขึ้นกับ connection
อื่น ๆ ที่กำลังใช้งาน server ตัวเดียวกันอยู่?**

คำตอบอยู่ในซอร์สโค้ดของ Axum เอง (`axum::extract::ws::WebSocketUpgrade::on_upgrade`) — ทุกครั้งที่เรียก
`.on_upgrade(callback)` Axum จะ **`tokio::spawn` task ใหม่แยกออกไปหนึ่งตัวสำหรับ connection นั้นโดยเฉพาะ**
ก่อนที่จะส่ง response `101 Switching Protocols` กลับไปด้วยซ้ำ — พูดอีกแบบคือ **ทุก WebSocket connection มี
task ของตัวเองแยกจาก connection อื่นสมบูรณ์ ตั้งแต่วินาทีแรกที่อัปเกรดสำเร็จ** ซึ่งตรงกับสิ่งที่ Part 48 สอนไว้
เรื่อง `tokio::spawn` เป๊ะ: **panic ที่เกิดขึ้นในหนึ่ง task ถูก catch ไว้ที่ขอบของ task นั้น (ผ่าน
`std::panic::catch_unwind` ภายในของ Tokio) แล้วแปลงเป็น `JoinError` — ไม่มีทาง propagate ไปกระทบ task อื่น
หรือทำให้ทั้งโปรแกรม (process) crash ได้เลย**

#### พิสูจน์ด้วยการรันจริง: panic ในหนึ่ง connection ไม่กระทบ connection อื่น

เขียน server ที่ **ตั้งใจ panic** เมื่อได้รับข้อความ `"boom"` (จำลองบั๊กจริงในระบบ production):

```rust
async fn handle_socket(mut socket: WebSocket) {
    while let Some(Ok(msg)) = socket.recv().await {
        if let Message::Text(text) = msg {
            if text.as_str() == "boom" {
                panic!("จำลอง bug ร้ายแรงในการประมวลผลข้อความจาก client!");
            }
            let reply = format!("ได้รับ: {text}");
            if socket.send(Message::Text(reply.into())).await.is_err() {
                break;
            }
        }
    }
}
```

ทดสอบด้วย 3 client: **A** ส่งคำที่ทำให้ task ของตัวเอง panic, **B** เป็น connection ปกติที่ทดสอบทั้งก่อนและ
หลัง A panic, **C** เป็น connection ใหม่ที่เชื่อมต่อ **หลังจาก** A panic ไปแล้ว (เพื่อพิสูจน์ว่า server ตัว
เดิมยังรับ connection ใหม่ได้ปกติ):

```
[client B] ก่อน A panic -- ได้รับ: Text(Utf8Bytes(b"\xe0..."))   // "ได้รับ: ping จาก B ครั้งที่ 1"
[client A] หลัง 'boom' -- ผลลัพธ์จากการอ่านต่อ: Some(Err(Protocol(ResetWithoutClosingHandshake)))
[client B] หลัง A panic ไปแล้ว -- ได้รับ: Text(Utf8Bytes(b"\xe0..."))  // "ได้รับ: ping จาก B ครั้งที่ 2"
[client C] connection ใหม่หลัง panic -- ได้รับ: Text(Utf8Bytes(b"\xe0..."))  // "ได้รับ: ping จาก C (connection ใหม่)"
```

และ stderr ของ server process (capture มาจากการรันจริง ตัดบางส่วนของ backtrace ที่ซ้ำซ้อนออก):

```
thread 'tokio-rt-worker' (23295) panicked at src/bin/panic_server.rs:14:17:
จำลอง bug ร้ายแรงในการประมวลผลข้อความจาก client!
stack backtrace:
   0: __rustc::rust_begin_unwind
   1: core::panicking::panic_fmt
   2: panic_server::handle_socket::{{closure}}
   3: axum::extract::ws::WebSocketUpgrade<F>::on_upgrade::{{closure}}
   4: tokio::runtime::task::core::Core<T,S>::poll::{{closure}}
   ...
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

และตรวจสอบด้วย `ps` ทันทีหลังจากนั้น — **process ของ server (PID เดิม) ยังทำงานอยู่ปกติ ไม่ตายไปกับ panic
เลย**:

```
  PID CMD
23290 ./target/debug/panic_server
```

ผลลัพธ์นี้ยืนยันทุกอย่างที่คาดไว้ตามหลักการของ Part 48: **`panic!` ของ client A ทำให้ task ของ A เท่านั้นที่
จบลง** (สังเกต frame ที่ 3 ในสแต็ก: `axum::extract::ws::WebSocketUpgrade::on_upgrade::{{closure}}` — นี่คือ
closure ที่ Axum `tokio::spawn` ไว้เฉพาะสำหรับ connection ของ A) — connection ของ A ทางฝั่ง client เห็นผลเป็น
`Protocol(ResetWithoutClosingHandshake)` (เพราะ TCP socket ของ connection นั้นถูกปิดกะทันหันโดยไม่มี Close
frame — task ที่ panic ไม่มีโอกาสได้ส่ง Close frame บอกลาก่อนตายเลย) — ในขณะที่ **client B ยังคุยกับ server
ได้ปกติทั้งก่อนและหลัง A panic** และ **client C ที่เชื่อมต่อใหม่ทีหลังก็ยังใช้งานได้ปกติ 100%** เพราะ
`axum::serve(listener, app)` (event loop หลักที่รับ connection ใหม่) รันอยู่บน task คนละตัวโดยสิ้นเชิงจาก
task ของ connection แต่ละอัน — panic ของ connection หนึ่งไม่มีทางเดินทางย้อนขึ้นไปกระทบ event loop หลักได้เลย

#### แยกแยะ: client หลุดกะทันหันกับ genuine protocol error

จาก log ข้างบนจะเห็น error message สองแบบที่คล้ายกันแต่ความหมายต่างกัน ต้องแยกแยะให้ถูกในโค้ด production
จริง:

- **`Protocol(ResetWithoutClosingHandshake)`** — TCP connection ถูกปิดโดยไม่มี WebSocket Close frame มาก่อน
  สาเหตุที่พบบ่อยที่สุดคือ **client หลุดกะทันหัน** (ปิด browser tab, เน็ตหลุด, มือถือสลับแอปแบบที่ OS ฆ่า
  connection ทิ้ง) — ไม่ใช่บั๊กของ server แต่ **เป็นเรื่องปกติที่ต้องคาดหวังไว้เสมอ** ในระบบที่มี client
  จำนวนมาก ไม่ควร log เป็น `error!` ระดับสูง (ไม่ต้องแจ้งเตือนทีม on-call) แค่ log เป็น `debug!`/`info!`
  ธรรมดาแล้วปล่อยให้ RAII guard (77.9) เก็บกวาดไปตามปกติ
- **`JoinError` จาก `.is_panic()`** (ถ้าคุณ `.await` `JoinHandle` ของ task นั้นเก็บไว้ ซึ่งปกติ Axum ไม่ทำเพราะ
  discard `JoinHandle` ไปเลยตามที่เห็นในซอร์สโค้ด) — นี่คือ **บั๊กจริงในโค้ดของคุณเอง** ที่ควร log เป็น
  `error!` ระดับสูงสุดและแจ้งเตือนทีมทันที เพราะมันหมายถึงมี edge case ที่โค้ดไม่ได้ตรวจสอบไว้ล่วงหน้า

ในทางปฏิบัติ ถ้าต้องการ **สังเกตเห็น panic แล้วบันทึก log ที่มีข้อมูลมากกว่าที่ default panic hook ให้มา**
(เช่น แนบ `client_id` เข้าไปในข้อความ) แนวทางที่ทำได้คือห่อ callback ของ `.on_upgrade()` ด้วย
`tokio::spawn` ของตัวเอง (ไม่ใช่ปล่อยให้ Axum spawn ให้เฉย ๆ) แล้ว `.await` `JoinHandle` เพื่อตรวจ
`.is_panic()` เอง หรือติดตั้ง custom panic hook ด้วย `std::panic::set_hook` ที่ log แบบมีบริบทมากขึ้นก่อน
ที่ panic จะ unwind — ทั้งสองเทคนิคนี้ **ไม่เปลี่ยนพฤติกรรมพื้นฐานที่พิสูจน์ไปแล้ว** (task อื่นไม่กระทบ,
process ไม่ตาย) แค่เพิ่มการสังเกตการณ์ (observability) ให้ดีขึ้นสำหรับระบบ production จริง

### 77.11 Capstone: ระบบแสดงจำนวนที่นั่งคงเหลือแบบ Real-Time

ตอนนี้มาประกอบทุกหัวข้อของบทนี้เข้าด้วยกันเป็นระบบเดียว ต่อยอดจาก theme ระบบจองตั๋วที่ใช้มาตลอดหลักสูตร
(Part 63-64): **หน้าเว็บที่กำลังดูรายละเอียด event หนึ่งอยู่ ต้องเห็นจำนวนที่นั่งคงเหลืออัปเดตแบบสด ๆ ทันทีที่
มีคนอื่นซื้อตั๋วไป** — โดยที่การซื้อตั๋วยังทำผ่าน REST endpoint ธรรมดาสไตล์ Part 63 ทุกประการ (ไม่มีอะไรพิเศษ
ฝั่งนั้น) แต่ REST endpoint นั้นจะ **กระตุ้น (trigger) ให้เกิดการ broadcast ไปยัง WebSocket client ทุกคนที่
กำลังดู event เดียวกันอยู่** ผสานทั้ง authentication (77.8) และ per-client tracking (77.7/77.9) เข้าด้วยกัน:

```rust
use axum::{
    extract::ws::{Message, WebSocket, WebSocketUpgrade},
    extract::{Path, Query, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::{any, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{
    atomic::{AtomicU64, Ordering},
    Arc, Mutex,
};
use tokio::sync::broadcast;

type ClientId = u64;
type EventId = u32;

#[derive(Clone, Serialize)]
struct EventInfo {
    name: String,
    seats_available: u32,
}

#[derive(Clone, Serialize)]
struct AvailabilityUpdate {
    event_id: EventId,
    seats_available: u32,
}

#[derive(Clone)]
struct AppState {
    events: Arc<Mutex<HashMap<EventId, EventInfo>>>,
    tx: broadcast::Sender<AvailabilityUpdate>,
    next_client_id: Arc<AtomicU64>,
    connected_clients: Arc<Mutex<HashMap<ClientId, EventId>>>,
}

#[derive(Deserialize)]
struct PurchaseRequest {
    seats: u32,
}

#[derive(Deserialize)]
struct AuthQuery {
    token: Option<String>,
}

fn validate_token(token: &str) -> Option<String> {
    // ในระบบจริงเรียก jsonwebtoken::decode(...) ตาม Part 74 เต็มรูปแบบ
    if token == "valid-token-abc123" {
        Some("phutjirakul".to_string())
    } else {
        None
    }
}

// ----- REST endpoint (สไตล์ Part 63): ซื้อตั๋ว แล้วยิง broadcast update -----
async fn purchase(
    State(state): State<AppState>,
    Path(event_id): Path<EventId>,
    Json(body): Json<PurchaseRequest>,
) -> Result<Json<EventInfo>, StatusCode> {
    let mut events = state.events.lock().unwrap();
    let event = events.get_mut(&event_id).ok_or(StatusCode::NOT_FOUND)?;

    if event.seats_available < body.seats {
        return Err(StatusCode::CONFLICT);
    }
    event.seats_available -= body.seats;
    let snapshot = event.clone();
    drop(events); // ปล่อย lock ก่อน broadcast (ไม่จำเป็นเพราะ broadcast::send ไม่ await แต่เป็น good practice)

    // ไม่สนใจผลลัพธ์ของ .send() -- ถ้ายังไม่มี WS client subscribe อยู่เลยก็ไม่ใช่ error
    let _ = state.tx.send(AvailabilityUpdate {
        event_id,
        seats_available: snapshot.seats_available,
    });

    Ok(Json(snapshot))
}

// ----- WebSocket endpoint: อัปเดตแบบ real-time เฉพาะ event ที่ subscribe -----
struct ClientGuard {
    id: ClientId,
    connected: Arc<Mutex<HashMap<ClientId, EventId>>>,
}

impl Drop for ClientGuard {
    fn drop(&mut self) {
        self.connected.lock().unwrap().remove(&self.id);
    }
}

async fn handle_live_socket(
    mut socket: WebSocket,
    state: AppState,
    event_id: EventId,
    username: String,
) {
    let client_id = state.next_client_id.fetch_add(1, Ordering::SeqCst);
    state.connected_clients.lock().unwrap().insert(client_id, event_id);
    println!("[server] client #{client_id} ({username}) เชื่อมต่อดู event #{event_id} แบบ real-time");
    let _guard = ClientGuard { id: client_id, connected: state.connected_clients.clone() };

    let mut rx = state.tx.subscribe();
    loop {
        tokio::select! {
            update = rx.recv() => {
                match update {
                    // กรองเอาแค่ update ของ event ที่ client รายนี้กำลังดูอยู่เท่านั้น
                    Ok(update) if update.event_id == event_id => {
                        let json = serde_json::to_string(&update).unwrap();
                        if socket.send(Message::Text(json.into())).await.is_err() {
                            break;
                        }
                    }
                    Ok(_other_event) => continue,
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        println!("[server] client #{client_id} ตกอัปเดตไป {n} ข้อความ");
                        continue;
                    }
                    Err(broadcast::error::RecvError::Closed) => break,
                }
            }
            incoming = socket.recv() => {
                match incoming {
                    Some(Ok(Message::Close(_))) | None => break,
                    Some(Ok(_)) => continue,
                    Some(Err(_)) => break,
                }
            }
        }
    }
}

async fn live_handler(
    ws: WebSocketUpgrade,
    Path(event_id): Path<EventId>,
    Query(query): Query<AuthQuery>,
    State(state): State<AppState>,
) -> Response {
    let token = match query.token {
        Some(t) => t,
        None => return (StatusCode::UNAUTHORIZED, "ต้องมี ?token=...").into_response(),
    };
    match validate_token(&token) {
        Some(username) => {
            ws.on_upgrade(move |socket| handle_live_socket(socket, state, event_id, username))
        }
        None => (StatusCode::UNAUTHORIZED, "token ไม่ถูกต้อง").into_response(),
    }
}

#[tokio::main]
async fn main() {
    let mut initial = HashMap::new();
    initial.insert(1, EventInfo { name: "Rust Conf 2026".to_string(), seats_available: 5 });

    let (tx, _rx) = broadcast::channel::<AvailabilityUpdate>(32);
    let state = AppState {
        events: Arc::new(Mutex::new(initial)),
        tx,
        next_client_id: Arc::new(AtomicU64::new(1)),
        connected_clients: Arc::new(Mutex::new(HashMap::new())),
    };

    let app = Router::new()
        .route("/events/{id}/purchase", post(purchase))
        .route("/events/{id}/live", any(live_handler))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:4009").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

สังเกตจุดที่ประกอบทุกหัวข้อของบทนี้เข้าด้วยกัน: **`tokio::select!`** ในหัวข้อ 77.11 ทำหน้าที่คล้าย
`.split()` + สอง task ใน 77.5 แต่รวมอยู่ใน task เดียว (ทั้งฟังก์ชั่น `rx.recv()` จาก broadcast **และ**
`socket.recv()` จาก client แข่งกันในแต่ละรอบของ loop — ตัวไหนพร้อมก่อนก็ทำงานก่อน) นี่เป็นอีกวิธีหนึ่งที่ทำ
สิ่งเดียวกับการ spawn สอง task แยกกัน แต่เหมาะกับกรณีที่ทั้งสองฝั่ง (อ่านจาก broadcast, อ่านจาก client) ทำงาน
บน `&mut socket` ตัวเดียวกันโดยไม่จำเป็นต้อง `.split()` เพราะไม่มีฝั่งไหนต้อง "เขียน" พร้อมกับอีกฝั่งพร้อมกัน
จริง ๆ (ต่างจาก 77.6 ที่ทั้งสอง task ต้องเขียน/อ่านพร้อมกันจริง จำเป็นต้อง `.split()`)

#### ทดสอบเต็มรูปแบบ: multi-client + REST trigger broadcast

ทดสอบด้วย client 2 ตัวที่ authenticated แล้วเชื่อมต่อดู event #1 พร้อมกัน จากนั้นยิง REST `POST
/events/1/purchase` (ซื้อ 2 ที่นั่ง) แล้วดูว่าทั้งสอง viewer ได้รับ broadcast หรือไม่:

```
===== ทดสอบ: WS โดยไม่มี token ต้องถูกปฏิเสธ =====
ผลลัพธ์ (คาดว่า error เพราะ handshake ไม่ผ่าน): true

===== เชื่อมต่อ WS สองคนเข้าดู event #1 แบบ real-time =====
===== ยิง REST POST /events/1/purchase (ซื้อ 2 ที่นั่ง) =====
REST response:
HTTP/1.1 200 OK
content-type: application/json
content-length: 45

{"name":"Rust Conf 2026","seats_available":3}

[viewer A] ได้รับ broadcast: Some(Ok(Text(Utf8Bytes(b"{\"event_id\":1,\"seats_available\":3}"))))
[viewer B] ได้รับ broadcast: Some(Ok(Text(Utf8Bytes(b"{\"event_id\":1,\"seats_available\":3}"))))

===== ซื้อเพิ่มอีกครั้ง (1 ที่นั่ง) ดูว่า broadcast ทำงานซ้ำได้ =====
REST response:
HTTP/1.1 200 OK
content-type: application/json
content-length: 45

{"name":"Rust Conf 2026","seats_available":2}

[viewer A] ได้รับ broadcast รอบสอง: Ok(Text(Utf8Bytes(b"{\"event_id\":1,\"seats_available\":2}")))
```

และ log ฝั่ง server ยืนยันการ track connect/disconnect ครบถ้วน:

```
[server] client #1 (phutjirakul) เชื่อมต่อดู event #1 แบบ real-time
[server] client #2 (phutjirakul) เชื่อมต่อดู event #1 แบบ real-time
[server] client #2 หลุดการเชื่อมต่อ ลบออกจาก tracking แล้ว
[server] client #1 หลุดการเชื่อมต่อ ลบออกจาก tracking แล้ว
```

นี่คือการพิสูจน์ครบทุกข้อของ capstone: **(1)** การเชื่อมต่อที่ไม่มี token ถูกปฏิเสธก่อนถึง WebSocket ด้วยซ้ำ
(77.8) **(2)** REST endpoint ธรรมดาสไตล์ Part 63 (`POST /events/1/purchase`) เป็นตัวกระตุ้นให้เกิดการ
broadcast ไปยัง WebSocket client โดยที่ REST endpoint นั้นไม่ต้องรู้จัก WebSocket connection ใด ๆ เป็นการ
ส่วนตัวเลย (คุยกันผ่าน `broadcast::Sender` ตัวเดียวใน `AppState` — สถาปัตยกรรมแบบ **publish-subscribe** ที่
แยก "ผู้ผลิต event" ออกจาก "ผู้บริโภค event" อย่างสมบูรณ์) **(3)** ทั้งสอง viewer ที่ authenticated แล้วได้รับ
update ตรงกันทั้งคู่ พร้อมกัน แบบ real-time จริง (77.6) **(4)** ระบบยังรองรับการ broadcast ซ้ำได้หลายรอบ
ต่อเนื่อง (การซื้อครั้งที่สอง) **(5)** connect/disconnect ของแต่ละ client ถูก track อย่างถูกต้องด้วย RAII
guard (77.7/77.9) ตลอดวงจรชีวิตของการทดสอบทั้งหมด

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ถือ `std::sync::MutexGuard` ข้าม `.await` ภายใน WebSocket handler

เช่นเดียวกับที่ Part 50 เตือนไว้เรื่อง async ทั่วไป แต่ WebSocket handler มีจุดที่ทำให้พลาดง่ายกว่าเดิม เพราะ
`ws.on_upgrade(callback)` **บังคับให้ future ของ `callback` ต้อง `Send`** (มัน `tokio::spawn` ให้เองเสมอ) —
ถ้า critical section ที่ล็อก `std::sync::Mutex` มี `.await` ข้างในแม้แต่จุดเดียว จะ compile ไม่ผ่านทันที
error จริงที่ capture มาจากการ compile:

```
error: future cannot be sent between threads safely
   --> src/bin/bad_mutex_await.rs:27:8
    |
 27 |     ws.on_upgrade(move |socket| handle_socket(socket, state))
    |        ^^^^^^^^^^ future returned by `handle_socket` is not `Send`
    |
    = help: within `impl Future<Output = ()>`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, HashMap<String, String>>`
note: future is not `Send` as this value is used across an await
   --> src/bin/bad_mutex_await.rs:21:65
    |
 18 |         let mut cache = state.cache.lock().unwrap();
    |             --------- has type `std::sync::MutexGuard<'_, HashMap<String, String>>` which is not `Send`
...
 21 |         tokio::time::sleep(std::time::Duration::from_millis(1)).await;
    |                                                                 ^^^^^ await occurs here, with `mut cache` maybe used later
note: required by a bound in `WebSocketUpgrade::<F>::on_upgrade`
   --> axum-0.8.9/src/extract/ws.rs:350:36
    |
347 |     pub fn on_upgrade<C, Fut>(self, callback: C) -> Response
    |            ---------- required by a bound in this associated function
...
350 |         Fut: Future<Output = ()> + Send + 'static,
    |                                    ^^^^ required by this bound in `WebSocketUpgrade::<F>::on_upgrade`
```

**วิธีแก้**: ปล่อย lock ก่อน `.await` เสมอ (drop guard ทันทีที่ใช้เสร็จ) หรือถ้าจำเป็นต้องถือ lock ข้าม
`.await` จริง ๆ ให้เปลี่ยนไปใช้ `tokio::sync::Mutex` ตามกฎการตัดสินใจของ Part 50 หัวข้อ 50.6

### 2. Broadcast channel ล้น (`RecvError::Lagged`) เมื่อมี client ที่อ่านช้ากว่าอัตราการส่ง

`tokio::sync::broadcast` ไม่มี backpressure (Part 50 หัวข้อ 50.10) — ถ้า producer ส่งเร็วกว่าที่ subscriber
ตัวใดตัวหนึ่งอ่านทัน (เช่น network ของ client ตัวนั้นช้า หรือ client กำลังประมวลผลอะไรหนักอยู่) ring buffer
ภายในของ channel จะเต็มและ **ข้อความเก่าที่ subscriber ตัวนั้นยังไม่ได้อ่านจะถูกทิ้งไปเงียบ ๆ** — ครั้งถัดไป
ที่เรียก `.recv()` จะได้ `Err(RecvError::Lagged(n))` แทนค่าจริง โดย `n` คือจำนวนข้อความที่ตกไป ทดสอบจริงด้วย
channel capacity 2 กับ client ที่อ่านช้ากว่าอัตราส่งของ producer มาก ได้ผลลัพธ์จริง:

```
Text(Utf8Bytes(b"got 0"))
Text(Utf8Bytes(b"LAGGED missed=6"))
Text(Utf8Bytes(b"LAGGED missed=8"))
Text(Utf8Bytes(b"LAGGED missed=8"))
Text(Utf8Bytes(b"LAGGED missed=8"))
Text(Utf8Bytes(b"LAGGED missed=8"))
Text(Utf8Bytes(b"LAGGED missed=7"))
```

**วิธีแก้**: อย่าเขียน `while let Ok(msg) = rx.recv().await { ... }` เฉย ๆ (จะออกจาก loop ทันทีที่เจอ
`Lagged` ครั้งแรก ทั้งที่ channel ยังไม่ปิดจริง) ต้อง `match` แยก `Lagged` ออกมาต่างหากแล้ว **ทำงานต่อ**
(อย่างที่ทำในทุกตัวอย่างของบทนี้ตั้งแต่ 77.11) และตั้ง `capacity` ของ `broadcast::channel` ให้ใหญ่พอสมควรกับ
อัตราการส่งข้อความจริงของระบบ เพื่อลดโอกาสเกิด `Lagged` ในสถานการณ์ปกติ

### 3. ลืม `.split()` เมื่อต้อง push ข้อมูลจาก server โดยไม่รอ client ถามก่อน

ถ้าเขียน loop เดียวที่ `socket.recv()` แล้วค่อย `socket.send()` ตอบกลับ (แบบ echo server ใน 77.3) — server
จะ**ไม่มีทางส่งข้อมูลอะไรได้เองเลยถ้า client ไม่ส่งอะไรมาก่อน** เพราะ loop ทั้งหมดค้างอยู่ที่
`socket.recv().await` รอ client อยู่ตลอดเวลา — อาการที่พบบ่อยคือ "ทำไม server push ไม่ทำงาน ทั้งที่โค้ดดู
เหมือนถูกทุกอย่าง" **วิธีแก้**: ใช้ `.split()` ตามหัวข้อ 77.5 เพื่อแยกงานอ่านกับงานเขียนออกเป็นสอง task
อิสระ (หรือใช้ `tokio::select!` แบบ capstone 77.11 ถ้าไม่จำเป็นต้องเขียนพร้อมกันจากสองที่)

### 4. Browser native WebSocket API ไม่รองรับ custom header — ทดสอบผ่าน server-side client แล้ว "ลืม" ว่า production ต้องใช้ query param/cookie

โค้ดตัวอย่างในหัวข้อ 77.8 ทดสอบผ่าน `tokio-tungstenite` ที่ตั้ง custom header ได้อย่างอิสระ (เพราะเป็น library
ระดับ low-level ที่ยืดหยุ่นกว่า) — นักพัฒนาที่ทดสอบผ่าน Rust client แบบนี้เท่านั้นอาจ**หลงลืมไปว่า** พอเอาไปใช้
จริงกับหน้าเว็บ (browser) แล้ว `new WebSocket(url)` **ไม่มีทางตั้ง `Authorization: Bearer <token>` header
ได้เลย** ต้องออกแบบให้ token มาทาง query parameter หรือ cookie ตั้งแต่แรกตามที่ 77.8 อธิบายไว้ — ไม่ใช่ error
ที่ compiler จับได้ (เพราะเป็นข้อจำกัดของสภาพแวดล้อมที่รันจริง ไม่ใช่ของภาษา Rust) จึงต้องตรวจสอบด้วยการทดสอบ
กับ browser จริงเสมอ ไม่พึ่งพา Rust test client เพียงอย่างเดียวเมื่อทำ integration test สุดท้ายก่อน deploy

### 5. ไม่ match `Message::Close` แล้วพยายาม `.send()` ต่อหลังจากได้รับมันแล้ว

ตาม doc comment ของ Axum เอง: หลังจาก `socket.recv()` คืน `Message::Close(_)` มาแล้ว **จะไม่มีข้อความอื่นเข้า
มาให้อ่านอีกเลย** และถ้าพยายาม `socket.send(...)` เพิ่มหลังจากนั้น มีโอกาสได้ error กลับมา (connection อยู่
ในสถานะกำลังปิด) — ถ้า loop ของคุณไม่มี arm จัดการ `Message::Close` แล้วปล่อยให้ code ตกไปทำ logic อื่นต่อ
(เช่นเผลอไปเข้า arm `_ => {}` ที่ทำอะไรบางอย่างต่อ) อาจเจอ error ที่งงว่า "ทำไม send ไม่ผ่านทั้งที่ยังไม่ได้
เรียก `.close()` เอง" **วิธีแก้**: จัดการ `Message::Close` แยกให้ `break` ออกจาก loop ทันทีเสมอ ตามที่ทุก
ตัวอย่างในบทนี้ทำ (77.4, 77.7, 77.9, 77.11)

### 6. เข้าใจผิดว่า panic ใน task ของ WebSocket connection หนึ่งจะทำให้ connection อื่นหรือ server ทั้งตัวล้มไปด้วย

จากความกลัวว่า WebSocket เป็น "connection ที่อยู่ยาว จัดการยาก" นักพัฒนาบางคนเขียนโค้ดป้องกัน panic แบบ
overengineer (เช่นห่อทุก handler ด้วย `std::panic::catch_unwind` เอง) ทั้งที่ Axum จัดการเรื่องนี้ให้แล้ว
โดยอัตโนมัติผ่าน `tokio::spawn` ที่พิสูจน์ไว้แล้วในหัวข้อ 77.10 — ป้องกันซ้ำสองชั้นแบบนี้ไม่ได้ทำอันตรายอะไร
แต่เป็นความซับซ้อนที่ไม่จำเป็น (และ `catch_unwind` เองก็มีข้อจำกัดหลายอย่างที่ Part 48 บางส่วนอาจแนะนำผ่าน ๆ
มา) สิ่งที่ควรทำจริง ๆ คือแค่ **มั่นใจว่า RAII guard (77.9) ครอบคลุม state ที่ต้อง cleanup ทุกก้อน** เพราะนั่น
คือสิ่งเดียวที่การันตี cleanup ได้แม้ตอน panic จริง

## แบบฝึกหัด (Exercises)

1. **ระดับง่าย**: ต่อยอดจาก echo server ในหัวข้อ 77.3 — เพิ่มการนับจำนวนข้อความที่ได้รับทั้งหมดต่อ connection
   แล้วส่งกลับไปพร้อมกับ echo (เช่น ส่ง `"hi"` ครั้งที่ 3 ได้ `"echo #3: hi"` กลับมา) *Hint*: เพิ่มตัวแปร
   `let mut count = 0;` ไว้นอก loop แล้วเพิ่มค่าทุกครั้งที่ได้ `Message::Text` ก่อนส่งกลับ

2. **ระดับกลาง**: เขียนระบบแจ้งเตือนแบบง่ายที่ผสมทั้ง broadcast (77.6) และ per-connection targeting (77.7)
   ใน endpoint เดียวกัน — client ที่เชื่อมต่อเข้ามาจะได้รับทั้ง (ก) ข้อความ "ยินดีต้อนรับ" ที่ส่งถึงตัวเอง
   เท่านั้น และ (ข) ข้อความ broadcast "มีคนเข้าห้อง N คนแล้ว" ที่ทุกคนเห็นพร้อมกันทุกครั้งที่มีคนเข้า/ออกห้อง
   *Hint*: ใช้ `mpsc::UnboundedSender` สำหรับข้อความส่วนตัวแบบ 77.7 ควบคู่กับ `broadcast::Sender<String>`
   สำหรับข้อความรวม แล้ว `tokio::select!` ระหว่างสองแหล่งนั้นในงานเขียน

3. **ระดับยาก**: เพิ่ม rate limiting ให้กับ WebSocket connection ในหัวข้อ 77.7 — ถ้า client ส่งข้อความเกิน 5
   ข้อความภายใน 1 วินาที ให้ server ส่ง `Message::Close` พร้อม close code/reason ที่อธิบายเหตุผล
   (`CloseFrame { code: 1008, reason: "rate limit exceeded".into() }` — code `1008` ใน RFC 6455 หมายถึง
   "Policy Violation") แล้วปิด connection นั้นทันที *Hint*: เก็บ `VecDeque<Instant>` ของเวลาที่ข้อความล่าสุด
   มาถึง ต่อ connection แล้วเช็คว่าในช่วง 1 วินาทีที่ผ่านมามีกี่ข้อความก่อนประมวลผลข้อความใหม่ทุกครั้ง

4. **ระดับยาก/ประยุกต์ใช้งานจริง**: ขยาย capstone ในหัวข้อ 77.11 ให้รองรับ **หลาย event พร้อมกัน** โดยที่
   client หนึ่งสามารถ subscribe ดูได้มากกว่า 1 event ในเวลาเดียวกัน (ผ่าน query parameter เช่น
   `?events=1,2,3`) และเพิ่ม REST endpoint `GET /events/{id}/viewers` ที่คืนจำนวนคนที่กำลังดู event นั้นอยู่
   real-time (นับจาก `connected_clients` map) *Hint*: เปลี่ยน `HashMap<ClientId, EventId>` เป็น
   `HashMap<ClientId, HashSet<EventId>>` แล้วกรอง update ใน `handle_live_socket` ด้วย
   `.contains(&update.event_id)` แทนการเทียบ `==` ตรง ๆ

## สรุป

บทนี้เป็นบทแรกในหลักสูตรที่เปิดโลกของ connection แบบ **full-duplex ที่มีอายุยืน** — ต่างจากทุกบทตั้งแต่ Part 61
ที่ connection แต่ละอันจบไปพร้อมกับ response เดียว เราเรียนไปว่า WebSocket **ไม่ใช่โพรโทคอลคนละโลกจาก HTTP**
แต่เป็นการ "อัปเกรด" HTTP connection ที่มีอยู่แล้วผ่าน header `Upgrade`/`Sec-WebSocket-*` ตาม RFC 6455 — และ
พิสูจน์ทุกกลไกสำคัญด้วยการรันจริง ไม่ใช่แค่ท่องจำ: handshake จริงที่ตรงกับ RFC test vector, การตอบ Ping/Pong
อัตโนมัติที่ระดับ protocol, การแยก socket ด้วย `.split()` เพื่อให้ server push ข้อมูลได้เอง, การผสาน
`tokio::sync::broadcast` (Part 50) เข้ากับ `AppState` (Part 64) เพื่อสร้าง pub-sub pattern เต็มรูปแบบ, การ
เก็บ per-connection state สำหรับส่งข้อความเจาะจง, การ authenticate ตอน handshake พร้อมข้อจำกัดจริงของ browser
API, การเก็บกวาด connection ที่หลุดด้วย RAII (Part 6), และการพิสูจน์ว่า panic ในหนึ่ง connection ไม่กระทบ
connection อื่นเลยเพราะกลไก `tokio::spawn` ต่อ connection ของ Axum (Part 48) — ปิดท้ายด้วย capstone ที่ผสาน
REST endpoint (Part 63) เข้ากับ WebSocket push แบบ real-time เต็มรูปแบบ พิสูจน์ด้วยการทดสอบ multi-client จริง

ทักษะเรื่อง long-lived connection, pub-sub ผ่าน channel ที่แชร์ใน state, และการจัดการ concurrent
read/write บน resource เดียวกันที่เรียนในบทนี้ จะเป็นพื้นฐานสำคัญสำหรับการออกแบบ API ที่ดีในบทถัดไป — Part 78
จะกลับมาที่ REST API เต็มรูปแบบอีกครั้ง แต่คราวนี้มองในมุม **best practices ระดับการออกแบบ** (versioning,
pagination, HATEOAS, idempotency) ที่ใช้ได้ทั้งกับ endpoint REST ธรรมดาและ endpoint ที่ทำงานคู่กับ WebSocket
แบบที่บทนี้สร้างไว้

---

**Part ก่อนหน้า:** [Authorization และ RBAC](part-076-authorization-rbac.md) | **Part ถัดไป:** [RESTful API Design Best Practices](part-078-rest-api-design.md)
