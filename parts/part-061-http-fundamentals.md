# Part 61: HTTP Fundamentals และ REST API Concepts

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า HTTP request/response **จริง ๆ แล้วคืออะไรในระดับ wire** — ไม่ใช่ "JSON ที่ส่งไปมา"
  อย่างที่คนจำนวนมากเข้าใจผิด แต่คือ**ข้อความ (text) ที่มีโครงสร้างตายตัว** ส่งผ่าน TCP connection — และพิสูจน์
  ความเข้าใจนี้ด้วยการเขียน **raw HTTP server ด้วยมือ** โดยใช้ `std::net::TcpListener` ที่ parse request line
  และ header ทีละไบต์ ไม่พึ่ง framework หรือ library HTTP ใด ๆ เลย แล้วรันจริงคุยกับ `curl` เพื่อดู byte
  ที่วิ่งผ่านสายจริง ๆ
- จำแนก **HTTP method** ทั้ง 7 ตัว (GET/HEAD/POST/PUT/PATCH/DELETE/OPTIONS) ตามคุณสมบัติ **safe** และ
  **idempotent** ได้อย่างถูกต้อง 100% พร้อมอธิบายเหตุผลเชิง semantic ว่าทำไมสองคุณสมบัตินี้สำคัญต่อการออกแบบ
  API จริง (ไม่ใช่แค่ท่องจำตาราง) โดยใช้ตัวอย่างระบบจองตั๋วงานอีเวนต์เป็นโดเมนหลักตลอดทั้งบท
- เลือกใช้ **HTTP status code** ได้ถูกต้องเหมาะสมกับสถานการณ์จริง ครอบคลุมทั้ง 5 class (1xx-5xx) และรู้จัก
  status code ที่ backend developer ทุกคน**ต้องรู้ขึ้นใจ**โดยไม่ต้องเปิดตาราง (200, 201, 204, 301/302, 304,
  400, 401, 403, 404, 405, 409, 422, 429, 500, 502, 503) พร้อมสถานการณ์ประกอบที่สมเหตุสมผลของแต่ละตัว
- อธิบาย HTTP header สำคัญ (`Content-Type`, `Content-Length`, `Accept`, `Cache-Control`, `ETag`,
  `Authorization`, CORS headers) ได้ว่าแต่ละตัวแก้ปัญหาอะไร และทำไม header เหล่านี้จึงเป็นพื้นฐานที่บท
  Authentication (Part 76 เป็นต้นไป) และ Middleware (Part 65) จะต่อยอด
- อธิบายได้ว่าทำไม **statelessness** ของ HTTP คือคุณสมบัติที่ทำให้ web server สเกลได้ในแนวนอน (horizontal
  scaling) โดยเชื่อมกับความเข้าใจเรื่อง `Send`/`Sync`/shared state จาก Part 39-40 และเข้าใจว่า **REST** เป็น
  แค่ **สถาปัตยกรรม (architectural style)** ไม่ใช่ protocol หรือ standard — รวมถึงเข้าใจ Richardson Maturity
  Model และความแตกต่างจาก RPC-style API แบบตรงไปตรงมา
- ประเมินได้ว่า**ทำไมเราจะไม่เขียน raw TCP HTTP parser เองในโปรเจกต์จริง** — สรุปความเจ็บปวดทั้งหมดที่เจอในบทนี้
  ให้เห็นภาพชัดว่า web framework อย่าง **Axum** (ที่ Part 62 จะแนะนำ) แก้ปัญหาอะไรให้ "ฟรี" (routing, extraction,
  middleware, error handling) ก่อนที่จะไปแตะโค้ด Axum จริงในบทต่อไป

## ความรู้ที่ต้องมีมาก่อน

- **Part 49 (Tokio: I/O และ Networking)**: บทนี้จะใช้ `TcpListener`/`TcpStream` ในระดับ `std::net` (synchronous)
  เพื่อสร้าง raw HTTP server ตัวอย่าง — กลไกการ `.bind()`/`.accept()` เหมือนกับที่ Part 49 สอนไว้ทุกประการ
  (แค่ไม่มี `.await` เพราะเป็น blocking I/O) เราจะอ้างอิงกลับไปที่ Part 49 ตลอดหัวข้อ 61.1 เพื่อโยงว่า framework
  อย่าง Axum (ซึ่งสร้างบน Tokio) จริง ๆ แล้วทำสิ่งเดียวกันนี้ในระดับที่ซับซ้อนกว่ามากด้วย async
- **Part 39-40 (Send/Sync, Threads, Shared State)**: หัวข้อ statelessness (61.6) จะอ้างอิงโดยตรงถึงความเข้าใจ
  เรื่อง shared mutable state ข้าม thread ที่ Part 39-40 วางไว้ — เพื่ออธิบายว่าทำไม "ไม่มี state ที่ผูกกับ
  connection เดิม" ถึงทำให้ scale ง่ายกว่า "มี state ที่ผูกกับ connection" อย่างมีนัยสำคัญ
- **Part 57-58 (Serde เบื้องต้น/ขั้นสูง)**: หัวข้อ content negotiation (61.5) จะใช้ `serde`/`serde_json` ที่สอน
  ไว้แล้วเป็นฐาน — ถ้ายังไม่แน่นเรื่อง `#[derive(Serialize, Deserialize)]` ควรกลับไปทวนก่อน เพราะบทนี้จะไม่สอน
  serde ซ้ำจากศูนย์
- **Part 12 (Result และ Error Handling เบื้องต้น)**, **Part 30-31 (thiserror/anyhow)**: ใช้เป็นฐานความเข้าใจ
  เรื่อง "error ที่คาดไว้แล้วจัดการได้" ตอนอธิบาย status code 4xx เทียบกับ 5xx — 4xx คือ error ที่ client
  ควรจะจัดการได้ (คล้าย `Result::Err` ที่ recoverable) ส่วน 5xx คือ "bug/ปัญหาไม่คาดคิดของ server" (คล้าย
  `panic!`)
- **Part 60 (Logging และ Tracing เบื้องต้น)**: ไม่บังคับต้องอ่านมาก่อนเพื่อเข้าใจบทนี้ แต่จะช่วยตอนอ่านหัวข้อ
  501.11 ที่พูดถึง middleware สำหรับ logging request/response ซึ่ง Part 65 จะสอนละเอียดอีกที
- บทนี้เป็น**บทแรกของโมดูล 4 (Web Development, Part 61-85)** และเป็น**บทสุดท้ายที่ไม่แตะ framework ใด ๆ เลย**
  — ตั้งใจให้เป็นพื้นฐานความเข้าใจ HTTP/REST ที่ล่องลอยอยู่เหนือ framework ไหนก็ได้ (Axum, Actix-web, หรือแม้แต่
  framework ในภาษาอื่น) ก่อนที่ Part 62 จะเริ่มแนะนำ Axum และใช้ทุกแนวคิดจากบทนี้เป็นฐานอ้างอิงตลอดทั้งโมดูล

## เนื้อหา

### 61.1 HTTP คืออะไรกันแน่ในระดับ Wire: เขียน Raw HTTP Server ด้วยมือ

#### ความเข้าใจผิดที่พบบ่อยที่สุด: "HTTP คือ JSON"

นักพัฒนาจำนวนมากที่เริ่มงาน backend มาจากการเขียน frontend หรือใช้ framework ระดับสูงมาตลอด (Express,
Django, Rails, หรือแม้แต่ Axum ที่เรากำลังจะเรียน) มักมีภาพในหัวว่า "HTTP request คือการส่ง JSON object ไปให้
server แล้ว server ส่ง JSON object กลับมา" — ภาพนี้**ไม่ผิดทั้งหมด** (JSON เป็น body format ที่นิยมที่สุดใน
ปัจจุบันจริง ตามที่หัวข้อ 61.5 จะอธิบาย) แต่มันเป็นแค่**ชั้นบนสุด** ของสิ่งที่เกิดขึ้นจริง ก่อนที่ JSON แม้แต่ตัว
เดียวจะถูกส่ง สิ่งที่วิ่งผ่านสาย (หรือ WiFi) จริง ๆ คือ**ข้อความ (plain text) ที่มีโครงสร้างตายตัวตาม RFC 7230/
7231** ห่อ JSON นั้นไว้อีกชั้นหนึ่ง

ทำไมเรื่องนี้สำคัญกับคนที่กำลังจะเรียน Axum ใน Part 62? เพราะ**ทุกอย่างที่ Axum ทำให้คุณ "ฟรี"** — การอ่าน
method, การอ่าน path, การ parse header, การแปลง body เป็น struct — คือการ**หุ้ม (abstract)** สิ่งที่บทนี้
กำลังจะให้คุณทำด้วยมือทั้งหมด ถ้าคุณไม่เคยเห็นชั้นล่างสุดนี้มาก่อน คุณจะไม่มีวันเข้าใจว่า `Json<T>` extractor
ของ Axum "มายากล" อะไรอยู่เบื้องหลัง — และเมื่อ Axum error ในรูปแบบที่ไม่คุ้นเคย (เช่น
`400 Bad Request: Failed to deserialize the JSON body`) คุณจะไม่รู้ว่าจะไปดีบั๊กจากมุมไหน

#### HTTP วิ่งอยู่บน TCP: ทวนจาก Part 49

จาก **Part 49** คุณรู้จัก `TcpListener`/`TcpStream` มาแล้วในฐานะ transport-layer primitive — สิ่งที่ทำได้แค่
"ส่ง/รับ stream ของไบต์ที่รับประกันว่ามาถึงตามลำดับและครบถ้วน" (reliable, ordered byte stream) โดย TCP
**ไม่รู้จักคำว่า "HTTP request" เลยแม้แต่นิดเดียว** — TCP เห็นแค่ไบต์ไหลเข้ามา ไม่รู้ว่าไบต์เหล่านั้นคือ HTTP,
FTP, หรือ protocol ที่คุณเขียนขึ้นมาเองก็ได้

**HTTP คือ protocol ที่นิยาม "รูปแบบของไบต์" ที่ทั้งสองฝั่ง (client/server) ตกลงกันไว้ล่วงหน้า** ว่าจะเขียน/อ่าน
ไบต์เหล่านั้นยังไง — พูดง่าย ๆ คือ HTTP เป็น**ชั้น application-layer protocol** ที่วางอยู่บน TCP (transport
layer) อีกที เหมือนกับที่ Part 49 อธิบายว่า `BufReader`/`BufWriter` ห่อ `TcpStream` ไว้อีกชั้นเพื่อให้อ่าน
ทีละบรรทัดได้สะดวกขึ้น — HTTP ก็คือ "ข้อตกลงเรื่อง encoding ของบรรทัดพวกนั้น"

#### โครงสร้างของ HTTP/1.1 Request แบบตายตัว

HTTP/1.1 request ที่ถูกต้องตาม RFC 7230 มีโครงสร้าง 4 ส่วนเสมอ เรียงกันเป๊ะ ๆ แบบนี้:

```text
METHOD SP REQUEST-TARGET SP HTTP-VERSION CRLF     <- (1) request line
Header-Name: header-value CRLF                    <- (2) header field (มีได้หลายบรรทัด)
Header-Name-2: header-value-2 CRLF
CRLF                                               <- (3) บรรทัดว่าง = จุดแบ่ง header/body
optional-body-bytes                                <- (4) body (มีหรือไม่มีก็ได้ ขึ้นกับ method/header)
```

**`CRLF`** คือ 2 ไบต์ `\r\n` (carriage return + line feed, ไบต์ `0x0D 0x0A`) — HTTP **บังคับ**ใช้ `\r\n` เป็น
ตัวคั่นบรรทัดเสมอ (ไม่ใช่แค่ `\n` แบบที่ Unix/Linux ใช้กันตามปกติ) ซึ่งเป็นมรดกตกทอดมาจาก protocol รุ่นก่อน ๆ
อย่าง SMTP/FTP — นี่คือรายละเอียดเล็ก ๆ ที่จะกลับมากัดคุณตอนเขียน parser เอง (ดูกับดักข้อ 2)

ตัวอย่าง request line จริง ๆ ที่ curl ส่งเวลาเรียก `GET /hello`:

```text
GET /hello HTTP/1.1
```

แยกได้เป็นสามส่วน คั่นด้วย space (SP, ไบต์ `0x20`) เดี่ยว ๆ:

- **`GET`** — HTTP method (หัวข้อ 61.2 จะเจาะลึก)
- **`/hello`** — request target หรือที่เรียกกันทั่วไปว่า "path" (บางครั้งมี query string ต่อท้าย เช่น
  `/hello?lang=th`)
- **`HTTP/1.1`** — เวอร์ชันของ protocol ที่ client ใช้ (หัวข้อ 61.10 จะเทียบกับ HTTP/2, HTTP/3)

ตามด้วย **header field** ศูนย์บรรทัดขึ้นไป แต่ละบรรทัดคือ `Name: value` (มี `:` แล้ว space แล้วค่อยเป็นค่า —
ตามธรรมเนียม แต่ parser ที่ดีต้องทนกับการไม่มี space หลัง `:` ได้ด้วยตาม RFC) จบด้วย**บรรทัดว่างหนึ่งบรรทัด**
(คือแค่ `\r\n` เปล่า ๆ ไม่มีอะไรก่อนหน้า) ซึ่งเป็น**สัญญาณเดียว**ที่บอกว่า header จบแล้วและ body (ถ้ามี) เริ่มต่อ
จากนี้ทันที — ไม่มีบรรทัดว่างนี้ = parser ไม่มีวันรู้ว่า header จบตรงไหน (นี่คือรากของกับดักข้อ 2)

#### โครงสร้างของ HTTP/1.1 Response

ฝั่ง response มีโครงสร้างคล้ายกันเป๊ะ แค่บรรทัดแรกเปลี่ยนจาก request line เป็น **status line**:

```text
HTTP-VERSION SP STATUS-CODE SP REASON-PHRASE CRLF  <- (1) status line
Header-Name: header-value CRLF                     <- (2) header field
CRLF                                                <- (3) บรรทัดว่าง
optional-body-bytes                                 <- (4) body
```

เช่น:

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 37
Connection: close

Hello from a hand-rolled HTTP server!
```

`200` คือ status code (หัวข้อ 61.3), `OK` คือ reason phrase (แค่ข้อความอธิบายให้มนุษย์อ่าน — client ที่ดีจะ
ตัดสินใจจาก**เลข** 200 ไม่ใช่จากคำว่า "OK" เพราะ reason phrase เปลี่ยนได้ตามใจ server โดยไม่ผิดสเปก)

#### เขียน Raw HTTP Server ด้วย `std::net::TcpListener`

มาเขียน HTTP server ที่**ไม่ใช้ library HTTP ใด ๆ เลย** — parse request line และ header ด้วยมือทีละบรรทัด
เพื่อพิสูจน์ว่าทุกอย่างที่อธิบายไปข้างบนคือความจริงแท้ ไม่ใช่ทฤษฎีลอย ๆ (โค้ดนี้ compile และรันจริงแล้วด้วย
`cargo build`/`cargo run` บนเครื่องจริง ไม่ใช่ pseudo-code):

```rust
use std::io::{BufRead, BufReader, Write};
use std::net::TcpListener;

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7979")?;
    println!("raw_server listening on 127.0.0.1:7979");

    // รับแค่ 1 connection แล้วจบโปรแกรม (เดโมเท่านั้น ไม่ใช่ production server)
    let (stream, addr) = listener.accept()?;
    println!("accepted connection from {addr}");

    let mut reader = BufReader::new(stream.try_clone()?);
    let mut stream = stream;

    // อ่าน request line แรก เช่น "GET /hello HTTP/1.1"
    let mut request_line = String::new();
    reader.read_line(&mut request_line)?;
    print!("--- raw request line ---\n{request_line}");

    // อ่าน header ต่อไปเรื่อย ๆ จนกว่าจะเจอบรรทัดว่าง (\r\n เปล่า) ซึ่งตาม RFC 7230 คือจุดแบ่ง header/body
    let mut headers = Vec::new();
    loop {
        let mut line = String::new();
        let bytes_read = reader.read_line(&mut line)?;
        if bytes_read == 0 || line == "\r\n" || line == "\n" {
            break;
        }
        headers.push(line.trim_end().to_string());
    }
    println!("--- raw headers ---");
    for h in &headers {
        println!("{h}");
    }

    // parse request line แบบหยาบ ๆ: METHOD SP PATH SP VERSION
    let mut parts = request_line.trim_end().split(' ');
    let method = parts.next().unwrap_or("");
    let path = parts.next().unwrap_or("/");
    println!("--- parsed ---\nmethod={method} path={path}");

    let body = "Hello from a hand-rolled HTTP server!";
    let response = format!(
        "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{}",
        body.len(),
        body
    );

    stream.write_all(response.as_bytes())?;
    stream.flush()?;
    println!("--- response sent ---\n{response}");

    Ok(())
}
```

สังเกตทุกจุดที่โค้ดนี้ทำสิ่งที่หัวข้อก่อนหน้าอธิบายไว้ตรง ๆ:

- **`reader.read_line(&mut request_line)`**: อ่านจนเจอ `\n` ตัวแรก (มาตรฐาน `BufRead::read_line` ของ Rust ถือ
  `\n` เป็นตัวจบบรรทัด และ**เก็บมันไว้ในผลลัพธ์ด้วย** — นี่คือเหตุผลที่โค้ดต้อง `trim_end()` ทีหลังเพื่อตัด
  `\r\n` ที่ติดมาออก)
- **loop อ่าน header จนเจอ `line == "\r\n"`**: คือการ parse "บรรทัดว่างที่แบ่ง header/body" ด้วยมือตรง ๆ ตามที่
  RFC 7230 นิยามไว้ — ถ้าไม่มีเงื่อนไขนี้ loop จะพยายามอ่านต่อไปเรื่อย ๆ (จนกว่า connection ปิดหรือ timeout)
  โดยไม่รู้ว่า header "จบ" ตรงไหน
- **`Content-Length: {}`คำนวณจาก `body.len()` เอง**: นี่คือจุดที่ต้อง**คำนวณให้ตรงกับความยาวจริงของ body เป็น
  ไบต์เสมอ** — ผิดพลาดจุดนี้แม้แค่ 1 ไบต์ก็ทำให้ client รอ/error (พิสูจน์จริงในกับดักข้อ 1)
- **`Connection: close`**: บอก client ว่า server จะปิด connection หลังตอบ response นี้ — ไม่รองรับ
  **keep-alive** (การใช้ TCP connection เดียวส่งหลาย request/response ต่อกัน ซึ่งเป็นค่าเริ่มต้นของ HTTP/1.1
  จริง ๆ) เพื่อให้โค้ดตัวอย่างนี้เรียบง่ายพอจะอ่านเข้าใจได้ในบทเดียว — production server (และ Axum ที่จะเรียน)
  จัดการ keep-alive ให้อัตโนมัติทั้งหมด

#### รันจริง: ผลลัพธ์ที่ได้จาก Server และจาก `curl -v`

รันเซิร์ฟเวอร์ด้วย `cargo run --bin raw_server` แล้วเปิด terminal อีกอันยิง `curl -v` เข้าไป — นี่คือ output
จริงที่ได้จากการรันสองโปรแกรมนี้คู่กัน (ไม่ใช่ตัวอย่างสมมติ):

**ฝั่ง server (stdout ของ `raw_server`):**

```text
raw_server listening on 127.0.0.1:7979
accepted connection from 127.0.0.1:39612
--- raw request line ---
GET /hello HTTP/1.1
--- raw headers ---
Host: 127.0.0.1:7979
User-Agent: curl/8.5.0
Accept: text/plain
--- parsed ---
method=GET path=/hello
--- response sent ---
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 37
Connection: close

Hello from a hand-rolled HTTP server!
```

**ฝั่ง client (`curl -v http://127.0.0.1:7979/hello -H "Accept: text/plain"`):**

```text
*   Trying 127.0.0.1:7979...
* Connected to 127.0.0.1 (127.0.0.1) port 7979
> GET /hello HTTP/1.1
> Host: 127.0.0.1:7979
> User-Agent: curl/8.5.0
> Accept: text/plain
>
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Content-Length: 37
< Connection: close
<
* Closing connection
Hello from a hand-rolled HTTP server!
```

เครื่องหมาย `>` ใน output ของ curl คือ**บรรทัดที่ client ส่งออกไป** และ `<` คือ**บรรทัดที่ client ได้รับกลับมา**
— เทียบกับ output ฝั่ง server จะเห็นว่า**มันคือข้อมูลชุดเดียวกันเป๊ะ** เพียงแค่มองจากสองฝั่งของสาย TCP เส้น
เดียวกัน สิ่งที่น่าสังเกตคือ **curl เพิ่ม header 3 ตัวให้เองโดยที่เราไม่ได้สั่ง**: `Host` (บังคับใน HTTP/1.1 —
บอกว่า client ตั้งใจคุยกับ virtual host ไหน จำเป็นมากเมื่อ server เดียวรับผิดชอบหลายโดเมน), `User-Agent`
(identify ตัว client เอง), และ `Accept: text/plain` (จากที่เราสั่งด้วย `-H` — หัวข้อ 61.5 จะอธิบายว่า header
นี้ใช้ทำ content negotiation) — นี่คือตัวอย่างจริงว่า HTTP client ทุกตัว (ไม่ใช่แค่ curl) แนบ metadata มาด้วย
เสมอ แม้ผู้เรียกจะไม่รู้ตัว

#### ทำไมการ Parse HTTP เองถึงเปราะบาง: Edge Case ที่โค้ดข้างบนยังไม่ได้จัดการ

โค้ด `raw_server.rs` ข้างบนทำงานได้ถูกต้องกับ request ธรรมดา ๆ แบบที่ curl ส่งมา แต่มันเป็น**เวอร์ชันที่ตัด
มุมทุกมุมที่ตัดได้** ยังไม่จัดการอย่างน้อย 6 เรื่องที่ HTTP server ระดับ production **ต้อง**จัดการให้ถูกต้อง:

1. **Chunked Transfer-Encoding** — request/response ขนาดใหญ่ที่ไม่รู้ความยาวล่วงหน้า (เช่น streaming) ไม่ใช้
   `Content-Length` แต่ใช้ `Transfer-Encoding: chunked` ซึ่งมี format การแบ่ง chunk เป็นของตัวเองอีกชั้น
   (แต่ละ chunk มี hex length ของตัวเองนำหน้า) — โค้ดข้างบน**ไม่รองรับเลย**
2. **Keep-Alive / Connection Reuse** — HTTP/1.1 ค่า default คือเปิด connection ทิ้งไว้ใช้ซ้ำหลาย request
   (ไม่ใช่ปิดทุกครั้งแบบโค้ดข้างบน) การจัดการ connection pool, timeout, และรู้ว่า request ไหนจบตรงไหนโดยไม่ปิด
   connection ซับซ้อนกว่าที่เห็นมาก
3. **Header Value ที่มี Multiple Lines (Obsolete Line Folding)** — RFC เดิมเคยอนุญาตให้ header value ยาวๆ
   ขึ้นบรรทัดใหม่โดยขึ้นต้นด้วย space/tab (ปัจจุบัน deprecated แล้วเพราะเป็นช่องโหว่ security แต่ parser ที่ดี
   ยังต้องรู้จักปฏิเสธมันอย่างถูกต้อง ไม่ใช่ crash หรือตีความผิด)
4. **Case-Insensitive Header Name** — header name เปรียบเทียบแบบไม่สนตัวพิมพ์เล็ก/ใหญ่ตามสเปก (ดูกับดักข้อ 4
   ที่จะพิสูจน์ด้วยโค้ดจริงว่าอันตรายแค่ไหนถ้าลืมเรื่องนี้)
5. **Malformed/Malicious Input** — request line ที่ไม่มี space, path ที่มี URL-encoded byte แปลก ๆ,
   ความยาวบรรทัด header ที่ยาวจนโจมตี memory (DoS) — โค้ดข้างบนใช้ `unwrap_or("")` กัน panic แบบหยาบ ๆ เท่านั้น
   ยังไม่ปฏิเสธ request ที่ผิดรูปแบบด้วย `400 Bad Request` อย่างถูกต้อง
6. **HTTP Request Smuggling** — ถ้ามี reverse proxy อยู่หน้า server และสอง parser (proxy กับ server) ตีความ
   ความกำกวมของ `Content-Length` vs `Transfer-Encoding` ต่างกัน อาจนำไปสู่ช่องโหว่ security ระดับร้ายแรง
   (แฮกเกอร์ "ซ่อน" request ที่สองไว้ในตัวเดียว) — นี่คือเหตุผลที่ HTTP parser ระดับ production ต้อง strict
   และผ่านการ audit security อย่างเข้มงวด ไม่ใช่สิ่งที่เขียนเองแบบขำ ๆ ได้ปลอดภัย

รายการนี้ยังไม่ครบทั้งหมดด้วยซ้ำ — และนี่คือเหตุผลตรง ๆ ว่าทำไมหัวข้อ 61.11 ท้ายบทจะสรุปว่า**ทุกโปรเจกต์จริง
ควรใช้ library ที่ผ่านการทดสอบมาแล้ว** (`hyper` ในกรณีของ Rust ที่ Axum สร้างอยู่บนมัน) แทนที่จะเขียน parser
เองแบบนี้ — โค้ดในหัวข้อนี้มีไว้เพื่อ**เข้าใจว่า Axum ทำอะไรให้เราอยู่เบื้องหลัง** ไม่ใช่แนวทางที่แนะนำให้ใช้จริง

### 61.2 HTTP Methods: Semantics, Safe, และ Idempotent

#### Method ไม่ใช่แค่ "ชื่อ" — มันคือสัญญาความหมาย (Semantic Contract)

HTTP method ทั้ง 7 ตัวที่ใช้กันทั่วไปไม่ใช่แค่คำ enum ให้เลือกตามใจ — แต่ละตัวมี**สัญญาความหมาย (semantic
contract)** ที่ตกลงกันไว้ในสเปกว่า client/server/proxy/cache ทุกตัวที่เกี่ยวข้องในทางเดินของ request จะ
**คาดหวัง**พฤติกรรมบางอย่างจาก method นั้น — ถ้า API ของคุณละเมิดสัญญานี้ (เช่น ใช้ `GET` เพื่อลบข้อมูล) มันจะ
ทำงาน "ได้" ในทดสอบตอนแรก แต่จะพังในรูปแบบที่ประหลาดและตามหาสาเหตุยากมากเมื่อ proxy/cache/crawler เข้ามา
เกี่ยวข้อง (ดูกับดักข้อ 3 ที่เล่าเหตุการณ์จริงของเรื่องนี้)

สองคุณสมบัติที่สำคัญที่สุดของสัญญานี้คือ **safe** และ **idempotent**:

- **Safe** (ปลอดภัย): method ที่ **ไม่ควรเปลี่ยนสถานะ (state) ใด ๆ บนฝั่ง server เลย** ในทางความหมาย — เป็นการ
  "อ่าน" อย่างเดียว ผลที่ตามมาคือ: browser/crawler/proxy **สามารถ**เรียก method safe ซ้ำได้ตามใจ (เช่น
  prefetch link, retry อัตโนมัติ) โดยไม่ต้องกลัวว่าจะเกิดผลข้างเคียงซ้ำซ้อน
- **Idempotent** (เรียกซ้ำได้ผลเหมือนเดิม): เรียก method นี้**กี่ครั้งก็ตาม**ด้วย input เดียวกัน ผลลัพธ์สุดท้าย
  (สถานะของ resource บน server) **เหมือนกับเรียกแค่ครั้งเดียว** — สังเกตว่า idempotent **ไม่ได้แปลว่า "ไม่มี
  side effect"** (ต่างจาก safe) แค่แปลว่า side effect นั้น**ไม่ทวีคูณขึ้นเมื่อเรียกซ้ำ**

ตารางสรุปทั้ง 7 method พร้อมตัวอย่างจากระบบ**จองตั๋วงานอีเวนต์** (domain ที่จะใช้ตลอดบทนี้):

| Method | Safe? | Idempotent? | ตัวอย่างจริงในระบบจองตั๋ว | คำอธิบาย |
|---|---|---|---|---|
| **GET** | ✅ ใช่ | ✅ ใช่ | `GET /api/v1/tickets/4821` | อ่านข้อมูลตั๋วใบเดียว เรียกกี่ครั้งก็ได้ผลเดิม ไม่เปลี่ยนอะไรเลย |
| **HEAD** | ✅ ใช่ | ✅ ใช่ | `HEAD /api/v1/tickets/4821` | เหมือน GET แต่ server ส่งกลับแค่ header (ไม่มี body) — ใช้เช็คว่า resource มีอยู่จริงหรือขนาดไฟล์เท่าไหร่โดยไม่ต้องโอนข้อมูลจริงทั้งหมด |
| **OPTIONS** | ✅ ใช่ | ✅ ใช่ | `OPTIONS /api/v1/tickets/4821` | ถาม server ว่า resource นี้รองรับ method ไหนบ้าง — ใช้เป็น CORS preflight request (หัวข้อ 61.4) |
| **PUT** | ❌ ไม่ | ✅ ใช่ | `PUT /api/v1/tickets/4821` (body: สถานะใหม่ทั้งใบ) | แทนที่ตั๋วทั้งใบด้วยข้อมูลใหม่ — เรียกซ้ำด้วย body เดิม ผลลัพธ์เหมือนกันเป๊ะ (ตั๋วมีค่าตามที่ส่งไปเสมอ) |
| **DELETE** | ❌ ไม่ | ✅ ใช่ | `DELETE /api/v1/tickets/4821` | ลบตั๋วใบนี้ — เรียกครั้งแรกลบสำเร็จ (`204`), เรียกซ้ำอีกครั้งตั๋วก็ยังหายไปเหมือนเดิม (มักได้ `404` แต่สถานะสุดท้ายเหมือนกัน) |
| **POST** | ❌ ไม่ | ❌ ไม่ | `POST /api/v1/tickets` (body: ข้อมูลตั๋วใหม่) | สร้างตั๋วใบใหม่ — เรียกซ้ำ 3 ครั้งด้วย body เดิม **ได้ตั๋ว 3 ใบที่แตกต่างกัน** (คนละ id) — นี่คือรากของหัวข้อ 61.9 |
| **PATCH** | ❌ ไม่ | ❌ ไม่ (โดยทั่วไป) | `PATCH /api/v1/tickets/4821` (body: `{"status": "cancelled"}`) | แก้เฉพาะบางฟิลด์ — ถ้า body เป็น "เพิ่มค่า" (เช่น `{"increment_view_count": 1}`) เรียกซ้ำจะเพิ่มค่าซ้ำ ไม่ idempotent โดยธรรมชาติ |

**ข้อสังเกตเชิงลึกที่สำคัญมาก**: `POST` และ `PATCH` เป็นสอง method เดียวที่**ไม่การันตี idempotent ไว้ในสเปก
เลย** — นี่ไม่ใช่ข้อบกพร่องของ HTTP แต่เป็นการออกแบบที่ตั้งใจ เพราะการ "สร้างสิ่งใหม่" (POST) โดยธรรมชาติ
**ควรจะ**สร้างของใหม่ทุกครั้งที่เรียก (นั่นแหละคือความหมายของมัน) — ปัญหาจะเกิดตอนที่ **network ไม่แน่นอน**
(request timeout, connection หลุดก่อนได้ response) แล้ว client ตัดสินใจ retry — หัวข้อ 61.9 จะแก้ปัญหานี้ด้วย
**Idempotency-Key** header

#### เขียนโค้ดจำแนก Method ด้วย Rust: พิสูจน์ตารางด้วยโปรแกรมจริง

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum HttpMethod {
    Get,
    Head,
    Post,
    Put,
    Patch,
    Delete,
    Options,
}

impl HttpMethod {
    // safe = ไม่เปลี่ยนสถานะฝั่ง server เลย (read-only ในทางความหมาย)
    fn is_safe(self) -> bool {
        matches!(self, HttpMethod::Get | HttpMethod::Head | HttpMethod::Options)
    }

    // idempotent = เรียกซ้ำกี่ครั้งก็ให้ผลลัพธ์สุดท้ายเหมือนกับเรียกครั้งเดียว
    fn is_idempotent(self) -> bool {
        matches!(
            self,
            HttpMethod::Get
                | HttpMethod::Head
                | HttpMethod::Options
                | HttpMethod::Put
                | HttpMethod::Delete
        )
    }
}

fn main() {
    let methods = [
        HttpMethod::Get,
        HttpMethod::Post,
        HttpMethod::Put,
        HttpMethod::Patch,
        HttpMethod::Delete,
        HttpMethod::Head,
        HttpMethod::Options,
    ];

    for m in methods {
        println!(
            "{:<8} safe={:<5} idempotent={:<5}",
            format!("{m:?}"),
            m.is_safe(),
            m.is_idempotent()
        );
    }
}
```

รันจริงได้ output ตรงกับตารางข้างบนทุกแถว:

```text
Get      safe=true  idempotent=true
Post     safe=false idempotent=false
Put      safe=false idempotent=true
Patch    safe=false idempotent=false
Delete   safe=false idempotent=true
Head     safe=true  idempotent=true
Options  safe=true  idempotent=true
```

สังเกตว่าในหัวข้อ 61.11 (และ Part 63) เมื่อเจอ Axum จริง จะเห็นว่า `axum::http::Method` ที่ Axum ใช้ (มาจาก
crate `http`) มี enum เดียวกันเป๊ะ — เพียงแต่ Axum **ไม่ได้บอก** ว่า method ไหน safe/idempotent ให้อัตโนมัติ
(สิ่งนี้ยังคงเป็น**ความรับผิดชอบของคนออกแบบ API**ที่ต้องเลือก method ให้ถูกความหมายเอง framework ไม่ได้บังคับ
หรือตรวจสอบให้)

#### เหตุผลเชิงลึก: ทำไม Idempotent ต้อง "ผลลัพธ์เหมือนกัน" ไม่ใช่ "ทำแบบเดียวกัน"

จุดที่มักเข้าใจผิดคือคิดว่า idempotent แปลว่า "เรียกซ้ำแล้ว server จะไม่ทำอะไรเลย (no-op)" — ที่ถูกคือ
idempotent วัดที่**ผลลัพธ์สุดท้าย (end state)** ไม่ใช่ที่ "การกระทำ" ตัวอย่าง `PUT /api/v1/tickets/4821` ที่ส่ง
body `{"status": "cancelled"}`:

- เรียกครั้งที่ 1: server เขียนแถวในฐานข้อมูล เปลี่ยน `status` จาก `open` เป็น `cancelled`, อัปเดต
  `updated_at` timestamp
- เรียกครั้งที่ 2 (ด้วย body เดิมเป๊ะ): server เขียนแถวในฐานข้อมูล**อีกครั้ง** เปลี่ยน `status` เป็น
  `cancelled` (เหมือนเดิม) แต่ `updated_at` **เปลี่ยนเป็นเวลาใหม่**

สังเกตว่า `updated_at` **เปลี่ยนไปทุกครั้งที่เรียก** แต่เรายังเรียกว่า `PUT` นี้ idempotent อยู่ดี เพราะ
**resource ที่ผู้ใช้สนใจ** (status ของตั๋ว) จบลงที่ค่าเดียวกันเสมอไม่ว่าจะเรียกกี่ครั้ง — timestamp การอัปเดต
เป็นรายละเอียดภายในที่ไม่ถูกนับเป็นส่วนหนึ่งของ "สถานะที่มีนัยสำคัญทางธุรกิจ" การเข้าใจความแตกต่างนี้สำคัญมาก
ตอนออกแบบระบบ retry ในหัวข้อ 61.9

### 61.3 HTTP Status Codes: 5 Class และตัวที่ต้องรู้ขึ้นใจ

Status code เป็นตัวเลข 3 หลักที่แบ่งเป็น 5 กลุ่มตามหลักแรก บอกความหมายกว้าง ๆ ก่อนอ่านตัวเลขที่แม่นยำกว่า:

| Class | ความหมายกว้าง ๆ | ใช้บ่อยแค่ไหนใน REST API ทั่วไป |
|---|---|---|
| **1xx** (Informational) | คำตอบชั่วคราว บอกว่ากำลังดำเนินการต่อ | น้อยมากในระดับ application (เช่น `100 Continue` มักถูกจัดการโดย HTTP library ให้อัตโนมัติ ไม่ใช่สิ่งที่ handler เขียนเอง) |
| **2xx** (Success) | request สำเร็จ | ใช้บ่อยที่สุด — ทุก endpoint ที่ทำงานถูกต้องจบด้วย class นี้ |
| **3xx** (Redirection) | ต้องทำอะไรเพิ่มเพื่อให้สำเร็จ (เช่น ไปที่ URL อื่น) | ใช้ปานกลาง มักเจอตอนจัดการ API versioning หรือ caching |
| **4xx** (Client Error) | **client** ทำอะไรผิด (request ผิดรูปแบบ, ไม่มีสิทธิ์, resource ไม่มีอยู่) | ใช้บ่อยมาก — เป็นเสียงตอบกลับหลักตอน validation ล้มเหลว |
| **5xx** (Server Error) | **server** มีปัญหา แม้ request จาก client ถูกต้องสมบูรณ์แล้ว | ควรใช้**น้อยที่สุด** — ถ้าเกิดบ่อยแปลว่ามี bug หรือ infrastructure มีปัญหา |

**หลักคิดสำคัญที่สุดในการเลือก class**: ถามว่า "ถ้า request แบบนี้ถูกส่งซ้ำอีกครั้งโดยไม่แก้ไขอะไร มันจะสำเร็จ
ไหม" — ถ้าคำตอบคือ "ไม่ทาง เพราะ client ส่งอะไรผิดมา" → 4xx (client ต้องแก้ก่อนถึงจะสำเร็จ) ถ้าคำตอบคือ
"ควรจะสำเร็จนะ เพราะ request มันถูกต้องแล้ว แค่ฝั่งเราเอง (server) มีปัญหาชั่วคราว" → 5xx (เหตุผลนี้ผูกกับ
Part 12 ที่สอนไว้ว่า error ที่ "คาดไว้แล้วจัดการได้" ต่างจาก error ที่ "ไม่ควรเกิดขึ้น" — 4xx คือฝั่งแรก, 5xx
คือฝั่งหลัง)

#### ตัวที่ต้องรู้ขึ้นใจ: พร้อมสถานการณ์จริงในระบบจองตั๋ว

| Code | ชื่อ | สถานการณ์จริงในระบบจองตั๋ว |
|---|---|---|
| **200** | OK | `GET /api/v1/tickets/4821` สำเร็จ ได้ข้อมูลตั๋วกลับมาใน body |
| **201** | Created | `POST /api/v1/tickets` สร้างตั๋วใหม่สำเร็จ — ตาม convention ที่ดี response ควรมี header `Location: /api/v1/tickets/4822` บอก URL ของ resource ที่สร้างขึ้นด้วย |
| **204** | No Content | `DELETE /api/v1/tickets/4821` ลบสำเร็จ — ไม่มีอะไรจะส่งกลับ (ไม่ใช่ error แต่ไม่มี body เลย ต่าง `200` ที่ปกติมี body) |
| **301/302** | Moved Permanently / Found | `GET /api/tickets/4821` (endpoint เวอร์ชันเก่าที่เลิกใช้แล้ว) redirect ไปยัง `GET /api/v1/tickets/4821` — `301` บอกว่าย้ายแบบถาวร (client/cache ควรจำ URL ใหม่ไปใช้ตลอด) ส่วน `302`/`303` บอกว่าย้ายชั่วคราว (ครั้งหน้ายังต้องมาถาม URL เดิมอีก) |
| **304** | Not Modified | `GET /api/v1/tickets/4821` พร้อม header `If-None-Match` ที่ตรงกับ `ETag` ปัจจุบันของ resource — server ตอบว่า "ข้อมูลไม่เปลี่ยนตั้งแต่ครั้งก่อน ใช้ค่าที่ cache ไว้ต่อได้เลย" (ไม่มี body เพื่อประหยัด bandwidth — หัวข้อ 61.4 อธิบายละเอียด) |
| **400** | Bad Request | `POST /api/v1/tickets` ด้วย body ที่**ไม่ใช่ JSON ที่ถูกต้องตาม syntax เลย** (เช่น comma เกินมา, bracket ไม่ปิด) — server อ่าน structure ไม่ออกตั้งแต่ต้น |
| **401** | Unauthorized | เรียก `POST /api/v1/tickets/4821/cancel` โดย**ไม่แนบ** header `Authorization` เลย หรือแนบ token ที่หมดอายุ/ปลอมแปลง — ระบบไม่รู้ว่า "คุณคือใคร" |
| **403** | Forbidden | ระบบรู้แล้วว่า "คุณคือใคร" (authenticated สำเร็จ) แต่คุณพยายามยกเลิกตั๋วของ**คนอื่น** ที่ไม่ใช่ของคุณ — รู้ตัวตนแล้ว แต่ไม่มีสิทธิ์ทำสิ่งนี้ |
| **404** | Not Found | `GET /api/v1/tickets/99999` — ไม่มีตั๋วเลขที่นี้อยู่ในระบบ |
| **405** | Method Not Allowed | `DELETE /api/v1/tickets` (ยิงไปที่ collection ทั้งชุด ไม่ใช่ตั๋วใบเดียว) — endpoint นี้รองรับแค่ `GET`/`POST` เท่านั้น ไม่รองรับ `DELETE` ที่ระดับ collection |
| **409** | Conflict | `PUT /api/v1/tickets/4821` พยายามเปลี่ยนสถานะเป็น `reserved` แต่ตั๋วใบนี้ถูกคนอื่นจองไปแล้วเมื่อ 1 วินาทีก่อน (race condition ระดับธุรกิจ) — request ถูกต้องตาม syntax แต่ขัดกับสถานะปัจจุบันของ resource |
| **422** | Unprocessable Entity | `POST /api/v1/tickets` ด้วย JSON ที่**ถูก syntax สมบูรณ์** แต่ค่า **ผิดทางความหมาย** เช่น `price_cents: -500` (ราคาติดลบ) — parse ผ่านแต่ business validation ไม่ผ่าน (ต่างจาก `400` ตรงนี้) |
| **429** | Too Many Requests | client ยิง `GET /api/v1/tickets` ถี่เกินกำหนด (เช่น 1000 ครั้ง/นาที) — server จำกัด rate เพื่อป้องกัน abuse มักมาพร้อม header `Retry-After: 30` บอกว่าให้รอกี่วินาทีก่อนลองใหม่ |
| **500** | Internal Server Error | handler ของ `POST /api/v1/tickets` เจอ bug ที่ไม่คาดคิด (เช่น unwrap บน `None` แล้ว panic — คล้ายกับที่ Part 12 อธิบายว่าเป็น error ที่ "ไม่ควรเกิดขึ้น") |
| **502** | Bad Gateway | มี reverse proxy (เช่น nginx) อยู่หน้า Axum server แล้ว Axum process **crash หรือยังไม่ start** — proxy ยิง request ไปหา backend ไม่เจอใครตอบเลย |
| **503** | Service Unavailable | server ทำงานอยู่ แต่โอเวอร์โหลดเกินจะรับ request เพิ่ม หรืออยู่ระหว่าง maintenance ที่ตั้งใจ — ต่างจาก `502` ตรงที่ server **มีตัวตน** และ**เลือก**ที่จะปฏิเสธ request ใหม่ ไม่ใช่ "หายไปเงียบ ๆ" |

**ข้อสังเกตที่สำคัญมาก (และเป็นกับดักข้อ 5 ท้ายบท)**: `400` กับ `422` มักถูกใช้ปนกันผิด ๆ ในโค้ดจริงจำนวนมาก
— ให้จำหลักง่าย ๆ ว่า **`400` = "ฉันอ่าน (parse) สิ่งที่คุณส่งมาไม่ออกเลย"** ส่วน **`422` = "ฉันอ่านออกครบ
ถูก syntax ทุกอย่าง แต่ค่าที่คุณส่งมาขัดกับกฎทางธุรกิจ"** — ความแตกต่างนี้สำคัญกับ client ที่ต้องเขียนโค้ด
handle error ต่างกัน (`400` อาจแปลว่า client มี bug ในการสร้าง request เอง ส่วน `422` อาจแปลว่าต้องโชว์ข้อความ
แจ้งผู้ใช้ปลายทางว่า "กรอกข้อมูลผิด")

### 61.4 Headers เจาะลึก: Metadata ที่ควบคุมทุกอย่างที่ Body ทำเองไม่ได้

Header คือ **metadata คู่ `key: value`** ที่แนบไปกับ request/response — สิ่งที่ header ทำได้แต่ body ทำไม่ได้
คือ**บอกวิธี "อ่าน"/"จัดการ" ตัว body** ก่อนที่จะอ่าน body จริง ๆ ด้วยซ้ำ (header มาก่อน body เสมอในโครงสร้างที่
เห็นในหัวข้อ 61.1)

#### Content-Type และ Content-Length: บอกว่า Body คืออะไรและยาวแค่ไหน

- **`Content-Type`**: บอก **MIME type** ของ body — `application/json` (หัวข้อ 61.5), `text/html`,
  `application/x-www-form-urlencoded`, `multipart/form-data; boundary=...` (ตอนอัปโหลดไฟล์) — server/client
  ที่ดีจะ**เชื่อ header นี้**ในการเลือกวิธี parse body ไม่ใช่เดาจากเนื้อหา (ความปลอดภัย: การเดา MIME type จาก
  เนื้อหา body เคยเป็นช่องโหว่ security จริงในหลาย browser รุ่นเก่า)
- **`Content-Length`**: บอกความยาว body เป็น**ไบต์** (ไม่ใช่ตัวอักษร! สำคัญมากกับข้อความ UTF-8 ภาษาไทยที่ 1
  ตัวอักษรอาจกิน 3 ไบต์) — หัวข้อ 61.1 พิสูจน์แล้วว่าตัวเลขนี้ผิดแม้แค่นิดเดียวทำให้ client รอค้างหรือ error
  ตรง ๆ

#### Accept และ Accept-Encoding: Content Negotiation

- **`Accept`**: client บอก server ว่า "รูปแบบไหนที่ฉันเข้าใจ/ต้องการ" เช่น `Accept: application/json` — API
  บางตัวรองรับหลายรูปแบบผลลัพธ์ (JSON, XML, CSV) จาก endpoint เดียวกัน แล้วใช้ header นี้เลือกว่าจะตอบแบบไหน
  (กลไกนี้เรียกว่า **content negotiation** — หัวข้อ 61.5 จะขยายต่อ)
- **`Accept-Encoding`**: client บอกว่ารองรับการ**บีบอัด**แบบไหน (`gzip`, `br` สำหรับ Brotli, `deflate`) —
  server ที่ฉลาดจะบีบอัด body ก่อนส่ง (ประหยัด bandwidth มาก โดยเฉพาะ JSON ที่มี key ซ้ำ ๆ บีบอัดได้ดีมาก) แล้ว
  ใส่ header `Content-Encoding: gzip` กลับมาบอกว่า "body ที่ส่งไปถูกบีบอัดแบบนี้นะ ต้องแตกก่อนอ่าน"

#### Authorization: กุญแจสู่บท Auth ในอนาคต

**`Authorization`** คือ header ที่ใส่ credential เพื่อพิสูจน์ตัวตน รูปแบบที่พบบ่อยที่สุดคือ:

```text
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

`Bearer` scheme (ตามด้วย token — มักเป็น **JWT**, JSON Web Token) คือมาตรฐานที่ API สมัยใหม่ส่วนใหญ่ใช้กัน
บทนี้จะยังไม่เจาะลึกเรื่อง JWT/OAuth (Part 76 เป็นต้นไปในโมดูลนี้จะสอนเต็มรูปแบบ) แต่ต้องรู้ไว้ก่อนว่า
`Authorization` คือ header ที่ middleware (Part 65) จะดักอ่านก่อน handler ทุกครั้งเพื่อตัดสินว่า request นี้
"คือใคร" — และถ้าไม่มี/ไม่ถูกต้อง server ควรตอบ `401` (ตามตารางหัวข้อ 61.3)

#### Cache-Control, ETag, If-None-Match: การ Cache HTTP Response

- **`Cache-Control`**: บอกว่า response นี้**เก็บไว้ใช้ซ้ำได้นานแค่ไหน/แบบไหน** เช่น
  `Cache-Control: max-age=3600` (เก็บไว้ใช้ได้ 1 ชั่วโมงโดยไม่ต้องถาม server ใหม่เลย), `no-store` (ห้าม cache
  เด็ดขาด — เหมาะกับข้อมูล sensitive เช่น ยอดเงินในบัญชี), `private` (cache ได้แค่ browser ของ user คนนั้น
  ห้าม shared cache/CDN เก็บ)
- **`ETag`**: server แนบ "ลายนิ้วมือ" ของ resource ณ เวลานั้น (มักเป็น hash ของเนื้อหา) เช่น
  `ETag: "a1b2c3d4"` — ครั้งต่อไปที่ client จะขอ resource เดียวกัน ส่ง header
  **`If-None-Match: "a1b2c3d4"`** กลับมา — ถ้า resource **ยังไม่เปลี่ยน** (ETag ตรงกัน) server ตอบ `304 Not
  Modified` แบบไม่มี body เลย (ประหยัด bandwidth มหาศาลสำหรับ resource ที่เปลี่ยนไม่บ่อยแต่ถูกขอบ่อย)

กลไก ETag/If-None-Match คือตัวอย่างที่ดีของการที่ **header ทำงานร่วมกันเป็นคู่ (request header คู่กับ response
header)** — `ETag` (response) จับคู่กับ `If-None-Match` (request ครั้งถัดไป) เหมือนกับที่ `Content-Type` จับคู่
กับ `Accept`

#### CORS Headers: ทำไมเบราว์เซอร์ถึงบล็อก Request ข้าม Origin

**CORS (Cross-Origin Resource Sharing)** คือกลไกที่**เบราว์เซอร์**บังคับใช้ (ไม่ใช่ server หรือ HTTP protocol
เอง) เพื่อป้องกันไม่ให้ JavaScript จากเว็บไซต์ A แอบยิง request ไปขโมยข้อมูลจากเว็บไซต์ B โดยที่ผู้ใช้ไม่รู้ตัว
(ใช้ session cookie ที่ login ไว้กับ B) — ถ้า frontend ของคุณอยู่ที่ `https://booking-app.example.com` และ
เรียก API ที่ `https://api.example.com` (คนละ origin เพราะ subdomain ต่างกัน) เบราว์เซอร์จะ**บล็อก response**
โดยอัตโนมัติ**ยกเว้น** server ตอบ header ที่อนุญาตชัดเจน:

- **`Access-Control-Allow-Origin: https://booking-app.example.com`** (หรือ `*` เพื่ออนุญาตทุก origin — ใช้กับ
  public API เท่านั้น ไม่ใช้กับ API ที่ต้อง authenticate)
- **`Access-Control-Allow-Methods: GET, POST, PUT, DELETE`** — บอกว่า origin นี้เรียก method อะไรได้บ้าง
- **`Access-Control-Allow-Headers: Authorization, Content-Type`** — บอกว่า header ไหนที่ client ส่งมาได้

สำหรับ request ที่ "ไม่ปลอดภัย" (เช่น มี `Content-Type: application/json` หรือมี `Authorization`) เบราว์เซอร์
จะยิง **preflight request** ก่อนด้วย method **`OPTIONS`** (ตรงกับที่หัวข้อ 61.2 บอกว่า `OPTIONS` เป็น safe
method) ถามว่า "ถ้าฉันจะยิง request จริงแบบนี้ อนุญาตไหม" — server ต้องตอบ CORS header ให้ครบใน response ของ
`OPTIONS` นั้นก่อน browser จะยอมส่ง request จริงตามมา บทนี้จะแค่ให้รู้จักกลไกนี้ไว้ก่อน — Part 65 (Middleware)
จะสอนวิธีตั้งค่า CORS middleware (`tower-http::cors`) ให้ Axum จัดการเรื่องนี้ให้อัตโนมัติ

#### Custom Headers: X-* และ Idempotency-Key

Header ที่ไม่ได้อยู่ในสเปกมาตรฐานแต่นิยมใช้กันในทางปฏิบัติ มักขึ้นต้นด้วย `X-` (แม้ RFC 6648 จะแนะนำให้เลิก
ใช้ prefix นี้แล้วก็ตาม แต่ในทางปฏิบัติยังเห็นทั่วไป):

- **`X-Request-Id`**: ID เฉพาะของ request นี้ — server แนบกลับไปใน response header เดียวกัน เพื่อให้ client
  อ้างอิงตอน report bug ("request ที่มีปัญหาคือ id นี้") และเชื่อมกับ **Part 60** ที่สอนเรื่อง `tracing` span
  — ระบบจริงมักผูก `X-Request-Id` เข้ากับ span ID เพื่อ trace request หนึ่งตัวข้ามหลาย log line/หลาย service
- **`Idempotency-Key`**: header ที่ client สร้างขึ้น (มักเป็น UUID) แนบไปกับ `POST` ที่ต้องการ "กันการทำซ้ำ"
  ตอน retry — หัวข้อ 61.9 จะเจาะลึกกลไกนี้เต็มรูปแบบพร้อมโค้ดจริง

### 61.5 Request/Response Bodies และ Content Negotiation

#### ทำไม JSON ครองตลาด

Body คือส่วนข้อมูลจริงที่ถูกส่ง — ในยุคนี้ **JSON (`application/json`)** ครองสัดส่วนการใช้งานส่วนใหญ่ของ REST
API เพราะเหตุผลเชิงปฏิบัติหลายข้อ: อ่านง่ายด้วยตาเปล่า (human-readable ต่างจาก binary format), ทุกภาษา
โปรแกรมมี parser พร้อมใช้ (ฝั่ง Rust คือ `serde_json` ที่ **Part 57** สอนไปแล้ว), map ตรงกับโครงสร้างข้อมูลของ
ภาษาสมัยใหม่ส่วนใหญ่ได้ตรงไปตรงมา (object/array/string/number/bool/null)

#### Content Negotiation ด้วย Rust + Serde: จำลอง Resource ตั๋วจริง

```rust
use serde::{Deserialize, Serialize};

// resource ตัวอย่าง: ตั๋วในระบบจองตั๋วคอนเสิร์ต (ตาม style guide ที่ให้ใช้ตัวอย่างโลกจริง)
#[derive(Debug, Serialize, Deserialize)]
struct Ticket {
    id: u64,
    event_name: String,
    status: TicketStatus,
    price_cents: u64,
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
enum TicketStatus {
    Open,
    Reserved,
    Sold,
    Cancelled,
}

// request body สำหรับ POST /api/v1/tickets (ยังไม่มี id เพราะ server จะสร้างให้)
#[derive(Debug, Deserialize)]
struct CreateTicketRequest {
    event_name: String,
    price_cents: u64,
}

fn main() {
    let ticket = Ticket {
        id: 4821,
        event_name: "Rust Conf Bangkok 2026".to_string(),
        status: TicketStatus::Reserved,
        price_cents: 150000,
    };

    let response_body = serde_json::to_string_pretty(&ticket).unwrap();
    println!("--- response body (Content-Type: application/json) ---");
    println!("{response_body}");

    let incoming = r#"{"event_name": "Rust Conf Bangkok 2026", "price_cents": 150000}"#;
    let parsed: CreateTicketRequest = serde_json::from_str(incoming).unwrap();
    println!("--- parsed request body ---");
    println!("{parsed:?}");
}
```

รันจริงได้ output:

```text
--- response body (Content-Type: application/json) ---
{
  "id": 4821,
  "event_name": "Rust Conf Bangkok 2026",
  "status": "reserved",
  "price_cents": 150000
}
--- parsed request body ---
CreateTicketRequest { event_name: "Rust Conf Bangkok 2026", price_cents: 150000 }
```

สังเกตว่า `#[serde(rename_all = "snake_case")]` (จาก Part 57-58) แปลง `TicketStatus::Reserved` (Rust
convention: `PascalCase` สำหรับ enum variant) เป็น `"reserved"` ใน JSON (convention ของ JSON ส่วนใหญ่:
`snake_case` หรือ `camelCase`) — นี่คือตัวอย่างจริงว่าทำไม Part 57-58 สำคัญมากสำหรับบทนี้: **`Ticket` struct
ตัวนี้แหละที่ Part 63 จะเอาไปใช้เป็น return type ของ Axum handler ตรง ๆ** ผ่าน extractor `Json<Ticket>` —
สิ่งที่ต่างจากตัวอย่างนี้แค่ Axum จะเซ็ต `Content-Type: application/json` และคำนวณ `Content-Length` ให้เอง
โดยอัตโนมัติ (สิ่งที่หัวข้อ 61.1 ต้องทำมือ)

#### รูปแบบอื่นที่ไม่ใช่ JSON: เมื่อไหร่ที่ควรใช้

JSON ไม่ใช่ทางเลือกเดียวเสมอไป:

- **`application/x-www-form-urlencoded`**: format แบบเดียวกับ query string (`key1=value1&key2=value2`) —
  ยังเจอบ่อยใน form HTML แบบเก่า หรือ endpoint ที่ต้อง compatible กับระบบเก่า (เช่น OAuth token endpoint
  มาตรฐาน RFC 6749 กำหนดให้ใช้ format นี้)
- **`multipart/form-data`**: ใช้เมื่อต้อง**อัปโหลดไฟล์**ปนกับข้อมูล text ในคำขอเดียว (เช่น ฟอร์มจองตั๋วที่แนบ
  รูปบัตรประชาชน) — body ถูกแบ่งเป็น "ส่วน (part)" หลายส่วนคั่นด้วย boundary string ที่กำหนดไว้ใน
  `Content-Type` header เอง (`Content-Type: multipart/form-data; boundary=----abc123`) — Part 64 (Axum State
  และ Extractors) จะกลับมาสอนวิธีจัดการ multipart จริงด้วย extractor ของ Axum
- **Protocol Buffers / MessagePack / อื่น ๆ**: format แบบ binary ที่กระชับกว่า JSON มาก เหมาะกับระบบที่
  performance สำคัญมาก (เช่น การสื่อสารระหว่าง microservice ปริมาณสูง) — เชื่อมกับ **Part 80 (gRPC)** ที่จะ
  สอนในโมดูลนี้ต่อไป ซึ่งใช้ Protocol Buffers เป็น default

### 61.6 Statelessness: ทำไม HTTP "จำอะไรไม่ได้" ถึงเป็นข้อดี

#### นิยาม Stateless ให้ชัด

**HTTP เป็น stateless protocol** — แปลว่า **แต่ละ request เป็นอิสระจากกันโดยสมบูรณ์** server ไม่ผูก "ความจำ"
ใด ๆ ไว้กับ **connection** ที่ request มาจาก request หนึ่งไม่มีสิทธิ์รู้ว่า request ก่อนหน้าจาก connection
เดียวกันคืออะไร — ทุกข้อมูลที่ server ต้องรู้เพื่อประมวลผล request หนึ่ง**ต้องมาพร้อมกับ request นั้นเอง**
(header, body, หรือข้อมูลที่ query จาก database ภายนอกด้วย key ที่ request บอกมา เช่น `Authorization` token)

ตัวอย่างที่ทำให้เห็นภาพชัด: สมมติ user login สำเร็จผ่าน `POST /api/v1/auth/login` แล้วอยากจองตั๋วผ่าน
`POST /api/v1/tickets/4821/reserve` — HTTP **ไม่มีกลไกในตัวเองที่บอกว่า "นี่คือคนเดียวกัน"** ระหว่างสอง
request นี้เลย ถ้าไม่มีอะไรมาเชื่อมสองอันนี้ (ปกติคือ token ที่ได้จาก login แนบกลับมาใน
`Authorization` header ของ request ที่สอง) server จะมองสอง request นี้เป็นคนละคนกันโดยสิ้นเชิง

#### เชื่อมกับ Part 39-40: ทำไม Stateless ถึง Scale ง่าย

จาก **Part 39-40** คุณรู้แล้วว่า shared mutable state ข้าม thread ต้องใช้ `Arc<Mutex<T>>` (หรือกลไก
synchronization อื่น) และการ synchronize มี**ต้นทุน**เสมอ (lock contention, cache coherency ข้าม CPU core) —
ทีนี้ลองขยายภาพจาก "หลาย thread บนเครื่องเดียว" ไปเป็น "หลาย**เครื่อง**" (หลาย server instance อยู่หลัง load
balancer): ถ้า server ต้องจำ state ที่ผูกกับ user คนหนึ่งไว้ (เช่น "user คนนี้ login ที่ server เครื่อง B
เท่านั้น") load balancer **ต้องส่ง request ทุกตัวของ user คนนั้นไปที่เครื่อง B เท่านั้นตลอด** (เรียกว่า
**sticky session**) — ถ้าเครื่อง B ล้ม, user คนนั้นก็เสีย session ทันที และการเพิ่ม/ลด server instance ตาม
load (auto-scaling) ก็ทำได้ยากขึ้นมาก เพราะต้องคำนึงว่า "ใครอยู่ที่เครื่องไหน" อยู่ตลอด

ในทางกลับกัน ถ้า server เป็น **stateless จริง ๆ** (ทุกข้อมูลที่ต้องรู้มาจาก request เอง เช่น JWT token ที่มี
ข้อมูล user encode ไว้ในตัวมันเอง ไม่ต้องเก็บ session ไว้บน server) — load balancer สามารถส่ง request ไปยัง
server instance **ตัวไหนก็ได้**ที่ว่างในขณะนั้น (round-robin ธรรมดา) เครื่องหนึ่งล้มไปก็ไม่กระทบ user เลย
(request ครั้งถัดไปไปเครื่องอื่นได้ทันที) และการเพิ่ม server instance ใหม่ตอน traffic สูงก็ทำได้อย่างอิสระโดย
ไม่ต้อง "sync" state อะไรระหว่างเครื่องก่อน — นี่คือรากฐานของ **horizontal scaling** ที่ระบบ backend ยุคใหม่
พึ่งพา และเป็นเหตุผลที่ **Part 81 (Microservices)** ในโมดูลนี้จะยึดหลัก "stateless service" เป็นข้อสมมติฐาน
พื้นฐานข้อแรกในการออกแบบระบบแบบกระจาย (distributed system)

**ข้อสังเกตสำคัญ**: statelessness ไม่ได้แปลว่า "แอปพลิเคชันไม่มี state เลย" (แน่นอนว่าฐานข้อมูลมี state —
ตั๋วมีสถานะจองแล้ว/ยังไม่จอง) มันแปลว่า **"server process ที่รับ request ไม่ผูก state ไว้กับตัวเอง"** — state
จริงทั้งหมดถูกผลักไปอยู่ที่**ที่เก็บข้อมูลกลาง** (database, cache แบบ Redis) ที่ server instance ไหนก็เข้าถึง
ได้เหมือนกัน ไม่ใช่ผูกกับ memory ของ process หนึ่งตัวโดยเฉพาะ

### 61.7 REST คือสถาปัตยกรรม ไม่ใช่ Protocol: Resources, HATEOAS, Richardson Maturity Model

#### REST ไม่ใช่มาตรฐาน — มันคือชุดหลักการ

**REST (Representational State Transfer)** เป็นคำที่ Roy Fielding บัญญัติไว้ในวิทยานิพนธ์ปริญญาเอกปี 2000
— มันคือ**สถาปัตยกรรม (architectural style)** ชุดหนึ่ง ไม่ใช่ protocol, ไม่ใช่ standard, ไม่มีองค์กรไหน
"รับรอง" ว่า API ตัวไหน "เป็น REST อย่างเป็นทางการ" — ต่างจาก HTTP ที่มี RFC ชัดเจนเป็นลายลักษณ์อักษร REST
เป็นแค่**แนวคิด**ที่บอกว่า "ถ้าออกแบบตามหลักการเหล่านี้ ระบบจะได้คุณสมบัติที่ดี" หลักการหลัก ๆ ได้แก่:

- **Client-Server**: แยกความรับผิดชอบชัดเจน — client จัดการ UI/state ของผู้ใช้ ส่วน server จัดการข้อมูล/
  business logic
- **Stateless**: ตามหัวข้อ 61.6 — แต่ละ request เป็นอิสระ
- **Cacheable**: response ควรระบุได้ชัดว่า cache ได้หรือไม่ (เชื่อม `Cache-Control` จากหัวข้อ 61.4)
- **Uniform Interface**: ใช้ interface เดียวกัน (HTTP methods, status codes มาตรฐาน) กับทุก resource — นี่คือ
  หลักการที่ทำให้ REST "คาดเดาได้" ข้าม API ต่างระบบ
- **Resource-Based**: ทุกอย่างในระบบถูกมองเป็น **"resource"** ที่มี **URI (identifier)** เฉพาะของตัวเอง —
  `/api/v1/tickets/4821` **คือ** ตั๋วใบที่ 4821 ไม่ใช่ "คำสั่งเรียกฟังก์ชัน get_ticket(4821)"

#### Resource-Oriented Thinking vs RPC-Style Thinking

ความแตกต่างสำคัญที่สุดระหว่าง REST กับ **RPC (Remote Procedure Call)** style คือ**หน่วยคิด**:

- **RPC-style**: คิดเป็น **"การเรียกฟังก์ชัน"** — `POST /getTicket`, `POST /cancelTicket`, `POST
  /createTicket` — ทุก endpoint คือชื่อ action, มักใช้ `POST` กับทุกอย่างไม่ว่าจะอ่านหรือเขียน (เพราะมันคือ
  "การเรียกฟังก์ชัน" ไม่ใช่ "การเข้าถึง resource")
- **REST-style**: คิดเป็น **"resource + method"** — resource คือ `/api/v1/tickets/4821` (noun, คำนาม) ส่วน
  "การกระทำ" มาจาก **HTTP method** ไม่ใช่จากชื่อ endpoint: `GET` (อ่าน), `PUT`/`PATCH` (แก้ไข), `DELETE` (ลบ)
  — endpoint เดียวกันทำได้หลายอย่างต่างกันแค่เปลี่ยน method

ตัวอย่างเทียบตรงในโดเมนตั๋ว:

| การกระทำ | RPC-style | REST-style |
|---|---|---|
| อ่านตั๋วใบหนึ่ง | `POST /getTicket` (body: `{"id": 4821}`) | `GET /api/v1/tickets/4821` |
| สร้างตั๋วใหม่ | `POST /createTicket` | `POST /api/v1/tickets` |
| ยกเลิกตั๋ว | `POST /cancelTicket` (body: `{"id": 4821}`) | `PATCH /api/v1/tickets/4821` (body: `{"status": "cancelled"}`) หรือ `DELETE /api/v1/tickets/4821` |
| ดูตั๋วทั้งหมดของ event หนึ่ง | `POST /listTicketsForEvent` (body: `{"event_id": 99}`) | `GET /api/v1/events/99/tickets` |

ทั้งสองแบบ**ทำงานได้จริง**ทั้งคู่ — REST ไม่ใช่ "ทางเดียวที่ถูก" แต่ให้ประโยชน์เรื่อง**ความคาดเดาได้**
(predictability): เมื่อเห็น `GET` ก็รู้ทันทีว่าปลอดภัย เรียกซ้ำได้ (จากหัวข้อ 61.2) เห็น status code
มาตรฐานก็ตีความได้ทันทีข้ามระบบ (จากหัวข้อ 61.3) — HTTP cache/proxy/browser ทั่วโลกก็เข้าใจ semantic นี้
โดยอัตโนมัติเพราะมันสร้างขึ้นบนสัญญาความหมายของ HTTP เอง ไม่ใช่ convention ที่แต่ละทีมคิดขึ้นเอง — **Part 79
(GraphQL)** และ **Part 80 (gRPC)** ในโมดูลนี้จะแนะนำอีกสองแนวทางที่แก้ปัญหาคนละมุม (GraphQL: client เลือกได้ว่า
จะเอา field ไหนจาก resource เดียว ลดปัญหา over-fetching; gRPC: RPC-style ที่เข้มงวดเรื่อง schema/performance
ด้วย Protocol Buffers) — บทนี้แค่ปูพื้นให้เห็นว่า REST คือหนึ่งในตัวเลือก ไม่ใช่ตัวเลือกเดียวที่มีอยู่

#### HATEOAS: อุดมคติที่ API จริงส่วนใหญ่ไม่ได้ทำเต็มรูปแบบ

**HATEOAS (Hypermedia As The Engine Of Application State)** คือหลักการระดับสูงสุดของ REST ตามที่ Fielding
นิยามไว้ดั้งเดิม — แนวคิดคือ **response ควรมี link ที่บอกว่า "จากสถานะนี้ ทำอะไรต่อได้บ้าง"** ฝัง**อยู่ในตัว
response เอง** ไม่ใช่ให้ client ต้อง hard-code URL ไว้ล่วงหน้า ตัวอย่าง response ของตั๋วที่ทำ HATEOAS เต็มรูป:

```json
{
  "id": 4821,
  "event_name": "Rust Conf Bangkok 2026",
  "status": "reserved",
  "price_cents": 150000,
  "_links": {
    "self": { "href": "/api/v1/tickets/4821" },
    "cancel": { "href": "/api/v1/tickets/4821/cancel", "method": "POST" },
    "event": { "href": "/api/v1/events/99" }
  }
}
```

ในทางทฤษฎี client ที่ดี**ไม่ควร hard-code** URL อย่าง `/api/v1/tickets/4821/cancel` ไว้เลย — แต่ควร**ตาม
link** ที่ server บอกมาใน `_links.cancel` เท่านั้น เพื่อให้ server เปลี่ยนโครงสร้าง URL ได้อย่างอิสระโดยไม่พัง
client เก่า

**ความจริงที่ต้องพูดตรง ๆ**: API สาธารณะส่วนใหญ่ในอุตสาหกรรม (Stripe, GitHub, Twitter/X, และแทบทุก API ที่คุณ
เคยใช้) **ไม่ได้ทำ HATEOAS เต็มรูปแบบ** — เหตุผลเชิงปฏิบัติคือ: (1) มันเพิ่มความซับซ้อนของทั้ง server และ client
มาก โดยประโยชน์ (การเปลี่ยน URL อย่างอิสระ) ไม่ได้เกิดขึ้นบ่อยพอจะคุ้ม ในทางปฏิบัติทีม frontend/backend ของ
บริษัทเดียวกันมักคุยกันแล้ว hard-code URL ไว้ตรง ๆ ก็เพียงพอ (2) client library ส่วนใหญ่ (เช่น
`reqwest` ที่จะเรียนใน Part เกี่ยวกับ HTTP client) ไม่ได้ออกแบบมาให้ "ตาม link" อัตโนมัติ ต้องเขียน logic
เพิ่มเอง (3) API ที่มี OpenAPI/Swagger spec (ซึ่งเป็นที่นิยมกว่ามากในทางปฏิบัติ) ให้ประโยชน์เรื่อง discovery
คล้ายกันแต่ทำนอก runtime (เอกสารแยก ไม่ใช่ฝังใน response) — สรุปคือ: **รู้จัก HATEOAS ไว้เพื่อเข้าใจว่า REST
"เต็มรูปแบบ" คืออะไร แต่ไม่ต้องคาดหวังว่า API ที่คุณจะเจอหรือสร้างเองจะทำแบบนี้เสมอไป** — สิ่งที่สำคัญกว่าใน
ทางปฏิบัติคือหลักการอื่น ๆ ของ REST (resource-based URI, ใช้ HTTP method/status code ให้ถูกความหมาย)

#### Richardson Maturity Model: วัดว่า API หนึ่ง "REST แค่ไหน"

Leonard Richardson เสนอโมเดล 4 ระดับ (0-3) เพื่อวัดว่า API หนึ่งใช้หลักการ REST มากแค่ไหน:

| Level | ชื่อ | ลักษณะ | ตัวอย่าง |
|---|---|---|---|
| **0** | The Swamp of POX (Plain Old XML) | endpoint เดียว, method เดียว (มัก `POST`) ทำทุกอย่าง — เหมือน RPC เต็มรูปแบบ | `POST /api` ทุก action ส่ง `{"action": "getTicket", ...}` ไปในตัวเดียวกัน |
| **1** | Resources | แยก URI ตาม resource แล้ว แต่ยังใช้ method เดียว (`POST`) กับทุก action | `POST /getTicket`, `POST /cancelTicket` แยก endpoint แต่ยังไม่ใช้ HTTP method ให้ถูกความหมาย |
| **2** | HTTP Verbs | ใช้ HTTP method (GET/POST/PUT/DELETE) และ status code ตามความหมายจริง — **นี่คือระดับที่ API ส่วนใหญ่ในอุตสาหกรรมอยู่** รวมถึงตัวอย่างในบทนี้ทั้งหมด | `GET /api/v1/tickets/4821`, `PATCH /api/v1/tickets/4821`, ตอบ `404`/`409` ตามสถานการณ์จริง |
| **3** | Hypermedia Controls | เพิ่ม HATEOAS เข้ามาเต็มรูปแบบ (ดูหัวข้อก่อนหน้า) | response มี `_links` บอกว่าทำอะไรต่อได้บ้าง |

**API ที่ Part 62-66 จะสอนสร้างด้วย Axum ในบทต่อ ๆ ไปของหลักสูตรนี้จะอยู่ที่ Level 2** — ซึ่งเป็นจุดที่ให้
ประโยชน์เชิงปฏิบัติสูงสุดเทียบกับความซับซ้อนที่เพิ่มขึ้น (Level 3 ให้ประโยชน์เพิ่มไม่มากนักเทียบกับความซับซ้อน
ที่เพิ่มขึ้นแบบไม่เป็นเส้นตรง ตามที่อธิบายไปในหัวข้อ HATEOAS)

### 61.8 URL Structure: Path Params, Query Params, และ Body — เลือกใช้ตัวไหนตอนไหน

การออกแบบ URL ที่ดีต้องแยกให้ถูกว่าข้อมูลชิ้นหนึ่งควรอยู่ใน**ส่วนไหน**ของ request สามที่เลือกได้คือ **path
parameter**, **query parameter**, และ **body** — แต่ละที่มีบทบาทต่างกันชัดเจน

ตัวอย่างที่รวมทั้งสามอย่างในคำขอเดียว:

```text
GET /api/v1/tickets/4821?include=event,venue&lang=th
```

แยกส่วน:

- **`/api/v1/tickets/4821`** — path: `4821` คือ **path parameter** (บางครั้งเขียนแทนด้วย `{id}` ใน route
  definition เช่น `/api/v1/tickets/{id}`)
- **`?include=event,venue&lang=th`** — **query parameter** สองตัว: `include=event,venue` และ `lang=th`

#### หลักการเลือก: Path Param vs Query Param vs Body

**Path parameter** ใช้กับข้อมูลที่**ระบุตัวตนของ resource** (identifier) — เป็นส่วนหนึ่งของ "resource คือ
อะไร" ไม่ใช่แค่ "ตัวกรอง/ตัวเลือกเสริม": `4821` ใน `/tickets/4821` ไม่ใช่ตัวกรอง มันคือ**ตัวตนของตั๋วใบนั้น
เอง** — ถ้าเปลี่ยนเลขนี้ resource ที่อ้างถึงก็เปลี่ยนไปคนละใบเลย

**Query parameter** ใช้กับ**ตัวกรอง (filter), การเรียงลำดับ (sort), การแบ่งหน้า (pagination), หรือตัวเลือก
เสริมที่ไม่จำเป็นต้องมี** เช่น `/api/v1/tickets?status=open&sort=price_asc&page=2&per_page=20` — สังเกตว่า
resource ที่อ้างถึงยังเป็น "collection ของตั๋วทั้งหมด" เหมือนเดิม query param แค่**กรอง/ปรับมุมมอง**ของ
collection นั้น ไม่ได้เปลี่ยนว่ากำลังพูดถึง resource อะไรอยู่ (ต่างจาก path param)

**Body** ใช้กับข้อมูลที่**ยาว/ซับซ้อน/เป็นโครงสร้าง** โดยเฉพาะตอนสร้าง/แก้ไข resource (`POST`/`PUT`/`PATCH`)
— เหตุผลเชิงเทคนิคที่สำคัญคือ **URL มีขีดจำกัดความยาวในทางปฏิบัติ** (ส่วนใหญ่ browser/server จำกัดไว้ราว 2,000-
8,000 ตัวอักษรขึ้นกับ implementation แม้ HTTP spec เองไม่ได้กำหนดขีดจำกัดตายตัว) และ URL **ถูกบันทึกลง
server log/browser history/proxy log เต็ม ๆ โดยธรรมชาติ** — ข้อมูล sensitive (เช่น รหัสผ่าน, เลขบัตรเครดิต)
**ไม่ควรอยู่ใน URL เด็ดขาด** (ทั้ง query param และ path param) เพราะจะไปโผล่ใน log ที่ไม่ได้ตั้งใจให้เก็บ
ข้อมูล sensitive เหล่านี้ — นี่คือเหตุผลที่ login ใช้ `POST` พร้อม body เสมอ ไม่มี API ไหนออกแบบเป็น
`GET /login?username=...&password=...`

ตัวอย่างสรุปในโดเมนตั๋วที่ใช้ทั้งสามอย่างถูกที่ถูกทาง:

```text
PATCH /api/v1/tickets/4821?notify_customer=true
Content-Type: application/json

{"status": "cancelled", "reason": "ลูกค้าขอยกเลิกเอง"}
```

- `4821` (path): "นี่คือตั๋วใบไหน"
- `notify_customer=true` (query): "ตัวเลือกเสริม — ให้ส่งอีเมลแจ้งลูกค้าด้วยไหม" (ไม่ใช่ส่วนหนึ่งของตัวตน
  resource แค่ปรับพฤติกรรมของการประมวลผลครั้งนี้)
- `{"status": "cancelled", "reason": "..."}` (body): "ข้อมูลโครงสร้างของการเปลี่ยนแปลงที่ต้องการทำ"

### 61.9 Idempotency ในทางปฏิบัติ: ทำไม PUT ต้อง Idempotent และทำไม POST ต้องมี Idempotency Key

#### ทวนจากหัวข้อ 61.2: ปัญหาจริงของ POST ที่ไม่ Idempotent

หัวข้อ 61.2 บอกไว้ว่า `POST` ไม่ idempotent โดยธรรมชาติ — ปัญหานี้ **ไม่ใช่ปัญหาทางทฤษฎี** มันเป็นปัญหาจริงที่
เกิดขึ้นทุกวันในระบบ production: สมมติ client ยิง `POST /api/v1/tickets/4821/reserve` เพื่อจองตั๋ว — server
ประมวลผลสำเร็จ (หักเงินจากบัตรเครดิต, บันทึกการจองในฐานข้อมูล) แต่**ระหว่างที่ response กำลังเดินทางกลับมา
network หลุด** — client เห็นแค่ "timeout ไม่ได้ response" **ไม่รู้เลยว่า server ทำสำเร็จไปแล้วหรือยัง** — ทาง
เลือกของ client มีแค่สองทาง: (1) ไม่ retry เลย → เสี่ยงที่ผู้ใช้จะคิดว่าการจองล้มเหลวทั้งที่จริงสำเร็จแล้ว
(2) retry เดิม → ถ้า server ทำสำเร็จไปแล้วจริง ๆ การ retry จะ**สร้างการจองซ้ำอีกใบ หักเงินซ้ำอีกรอบ**

นี่คือปัญหาพื้นฐานของระบบแบบกระจาย (distributed system) ที่ **Part 81-84** ในโมดูลนี้จะกลับมาขยายความอย่าง
เป็นระบบ (retry strategy, exactly-once vs at-least-once delivery) — บทนี้จะแก้เฉพาะกรณีของ HTTP ด้วยเครื่องมือ
ที่ใช้กันจริงในอุตสาหกรรม: **Idempotency-Key header**

#### กลไก Idempotency Key

Client สร้าง **key เฉพาะของ "ความตั้งใจ (intent)" นี้ครั้งเดียว** (มักเป็น UUID v4) แนบไปกับ `POST` ทุกครั้ง
ที่พยายามทำ intent เดียวกันนี้ (รวมถึงตอน retry ด้วย key เดิมเป๊ะ) — server เก็บ mapping ระหว่าง key กับ
ผลลัพธ์ที่เคยประมวลผลไปแล้ว ถ้าเจอ key ที่เคยเห็นมาก่อน **ส่งผลลัพธ์เดิมกลับไปโดยไม่ประมวลผลซ้ำ**:

```rust
use std::collections::HashMap;

// จำลอง server เก็บผลลัพธ์ของ POST ที่เคยประมวลผลไปแล้วต่อ idempotency key หนึ่งตัว
struct BookingServer {
    processed: HashMap<String, u64>, // idempotency key -> booking_id ที่สร้างไปแล้ว
    next_booking_id: u64,
}

impl BookingServer {
    fn new() -> Self {
        Self { processed: HashMap::new(), next_booking_id: 1 }
    }

    // จำลอง POST /api/v1/bookings พร้อม header Idempotency-Key
    fn create_booking(&mut self, idempotency_key: &str) -> (u64, bool) {
        if let Some(&existing_id) = self.processed.get(idempotency_key) {
            // เคยประมวลผล key นี้ไปแล้ว -> คืน booking เดิม ไม่สร้างใหม่ซ้ำ
            return (existing_id, false);
        }
        let id = self.next_booking_id;
        self.next_booking_id += 1;
        self.processed.insert(idempotency_key.to_string(), id);
        (id, true)
    }
}

fn main() {
    let mut server = BookingServer::new();

    // สถานการณ์จริง: client ส่ง POST แต่ network timeout ก่อนได้ response -> client retry ด้วย key เดิม
    let key = "client-retry-abc-123";
    let (id1, created1) = server.create_booking(key);
    println!("ครั้งที่ 1: booking_id={id1} created_new={created1}");

    let (id2, created2) = server.create_booking(key); // retry ด้วย idempotency key เดิม
    println!("ครั้งที่ 2 (retry): booking_id={id2} created_new={created2}");

    let (id3, created3) = server.create_booking("different-key-999"); // request คนละใบจริง ๆ
    println!("request อื่น: booking_id={id3} created_new={created3}");
}
```

รันจริงได้ output:

```text
ครั้งที่ 1: booking_id=1 created_new=true
ครั้งที่ 2 (retry): booking_id=1 created_new=false
request อื่น: booking_id=2 created_new=true
```

สังเกตผลลัพธ์: **ครั้งที่ 2 ที่เป็น retry ด้วย key เดิมได้ `booking_id=1` เหมือนครั้งแรกเป๊ะ** (ไม่สร้างใบใหม่
`created_new=false`) ในขณะที่ request อื่นที่ใช้ key ต่างออกไป (`different-key-999`, สื่อถึงความตั้งใจจองใบ
ใหม่จริง ๆ ไม่ใช่ retry) ได้ `booking_id` ใหม่ตามที่ควรจะเป็น — นี่คือวิธีที่ API จริงอย่าง Stripe ใช้แก้ปัญหา
"หักเงินซ้ำจาก retry" ได้จริงในระบบ production (Stripe เรียก header นี้ตรงว่า `Idempotency-Key` เหมือนกัน)

**ข้อควรระวังเชิงปฏิบัติ**: การ implement จริงต้องคิดเพิ่มอีกสองเรื่อง (1) **TTL ของ key** — เก็บ mapping นี้
ไว้ตลอดไปไม่ได้ (memory จะเต็ม) ต้องมี expiry (เช่น 24 ชั่วโมง — นานพอสำหรับ retry ตามธรรมชาติ แต่ไม่นานจนกิน
memory ไม่จำกัด) (2) **ต้องตรวจว่า request body เหมือนกันด้วย** ไม่ใช่แค่ key ตรงกัน (ถ้า client ส่ง key เดิม
แต่ body ต่างกัน คือ bug ของ client ที่ควร reject ด้วย `422` ไม่ใช่เงียบ ๆ คืนผลลัพธ์เก่าไปให้)

#### ทำไม PUT ไม่ต้องใช้ Idempotency Key แต่ POST ต้อง

คำถามที่ตามมาคือ: ในเมื่อ `PUT` idempotent อยู่แล้วตามสัญญาความหมาย (หัวข้อ 61.2) ทำไมไม่ต้องมี
`Idempotency-Key` เหมือนกัน? — เพราะ `PUT` **idempotent ได้ตามธรรมชาติของ semantic ของมันเอง** (แทนที่ทั้ง
resource ด้วยค่าที่ระบุ ไม่ว่าเรียกกี่ครั้งค่าสุดท้ายก็เหมือนกัน) โดย**ไม่ต้องมี key อะไรมาช่วยเลย** — แต่ `POST`
("สร้างสิ่งใหม่") มีความกำกวมโดยธรรมชาติว่า "เรียกซ้ำ" กับ "ตั้งใจสร้างอีกใบจริง ๆ " ต่างกันยังไง ซึ่ง server
เดารู้เองไม่ได้ — ต้องมี key จาก client มาบอกความตั้งใจตรง ๆ เท่านั้น

### 61.10 HTTP/1.1 vs HTTP/2 vs HTTP/3: สิ่งที่ Backend Developer ควรรู้ (Awareness Level)

บทนี้เน้น semantic ของ HTTP (method, status, header, body) ซึ่ง**เหมือนกันทุกเวอร์ชัน** — สิ่งที่เปลี่ยนไป
ระหว่างเวอร์ชันคือ**การขนส่ง (transport)** ล้วน ๆ ในระดับที่ backend developer ควบรู้ไว้แม้จะไม่ต้องเจาะลึกก็
ตาม (เพราะ Axum/reverse proxy จัดการให้อัตโนมัติแทบทั้งหมด):

**HTTP/1.1** (ที่ใช้สาธิตทั้งบทนี้ผ่าน raw TCP server) — **1 connection ต่อ 1 request-in-flight** (ต้องรอ
response ของ request ก่อนถึงจะส่ง request ถัดไปบน connection เดียวกันได้ ยกเว้นทำ **pipelining** ซึ่งมีปัญหา
เรื่อง **head-of-line blocking**: ถ้า response แรกช้า response ถัดไปก็ติดคอรอด้วย แม้จะพร้อมแล้วก็ตาม) —
เพื่อแก้ปัญหานี้ browser สมัยก่อนจึงเปิดหลาย connection พร้อมกันไปยัง host เดียวกัน (ปกติจำกัดไว้ราว 6
connection ต่อ host)

**HTTP/2** — แก้ปัญหา head-of-line blocking ด้วย **multiplexing**: หลาย request/response สามารถ**สลับกัน
วิ่งบน TCP connection เดียวกัน**ได้พร้อมกันจริง (แบ่งเป็น "stream" หลายตัวคนละ ID บน connection เดียว) ไม่ต้อง
เปิดหลาย connection แบบ HTTP/1.1 อีกต่อไป และมี **header compression** (HPACK) ที่ช่วยลด overhead ของ header
ที่ซ้ำ ๆ กันในหลาย request (เช่น `Cookie`, `User-Agent` ที่เหมือนกันทุก request จาก client เดียวกัน) — HTTP/2
วิ่งอยู่บน TCP เหมือนเดิม แค่เปลี่ยนวิธี encode ข้อความจาก text-based (ตามหัวข้อ 61.1) เป็น **binary framing**
ล้วน ๆ (จึงไม่สามารถใช้ raw TCP server แบบหัวข้อ 61.1 คุยกับ HTTP/2 client ตรง ๆ ได้เลย ต้อง implement binary
protocol ใหม่ทั้งหมด — เหตุผลอีกข้อที่ต้องพึ่ง library อย่าง `hyper`)

**HTTP/3** — เปลี่ยนฐานจาก TCP ไปเป็น **QUIC** (สร้างอยู่บน **UDP**) แก้ปัญหา head-of-line blocking ที่ยังหลง
เหลืออยู่ใน HTTP/2 ในระดับที่ลึกกว่า (TCP เองมี head-of-line blocking ระดับ packet: packet หนึ่งหายไป TCP
บล็อกทุก stream รอ retransmit แม้ stream อื่นไม่เกี่ยวก็ตาม — QUIC แก้ปัญหานี้เพราะ stream แต่ละตัวเป็นอิสระ
จากกันจริง ๆ ในระดับ transport) และมี **connection migration** (เปลี่ยนเครือข่าย เช่น จาก WiFi ไป 4G โดยไม่
ต้องเริ่ม connection ใหม่ — มีประโยชน์มากกับ mobile client)

**สิ่งที่ backend developer ต้องรู้ในทางปฏิบัติ**: (1) **Axum/hyper รองรับ HTTP/1.1 และ HTTP/2 ให้อัตโนมัติ**
โดย application code (handler ที่คุณเขียน) **ไม่ต้องรู้เลยว่ากำลังคุยด้วยเวอร์ชันไหน** — semantic ทั้งหมดที่
เรียนในบทนี้เหมือนกันทุกเวอร์ชัน (2) ในทางปฏิบัติจริง **HTTP/2/3 มักถูกจัดการที่ระดับ reverse proxy/load
balancer** (nginx, Cloudflare, AWS ALB) ที่รับ HTTP/2/3 จาก client แล้วแปลงเป็น HTTP/1.1 คุยกับ backend
service ภายในเครือข่ายเดียวกัน (เพราะ backend ภายในไม่ต้องการ overhead ของ multiplexing ที่ HTTP/2 แก้ปัญหา
ให้ — ปัญหานั้นสำคัญตอน "ข้าม internet ที่ latency สูง" มากกว่า "คุยกันในเครือข่ายภายในเดียวกัน") (3) เรื่องนี้
เป็นความรู้ระดับ awareness เท่านั้นสำหรับหลักสูตรนี้ — ไม่ต้อง implement HTTP/2/3 เองแม้แต่กรณีเดียว

### 61.11 ทำไมต้องมี Web Framework: สรุปความเจ็บปวดจาก Raw TCP และสิ่งที่ Axum ให้ "ฟรี"

#### ทวนความเจ็บปวดทั้งหมดจากหัวข้อ 61.1

โค้ด `raw_server.rs` ในหัวข้อ 61.1 ทำงานได้กับ request ธรรมดา ๆ หนึ่งตัว แต่ต้องเขียนมือทุกจุด: parse
request line เอง, parse header เอง (แบบหยาบ ๆ ไม่ handle case-insensitive อย่างถูกต้อง — ดูกับดักข้อ 4),
คำนวณ `Content-Length` เอง (ผิดแล้ว client hang — กับดักข้อ 1), เขียนบรรทัดว่างคั่น header/body เองด้วยมือ
(ลืมแล้ว parser พัง — กับดักข้อ 2), ไม่รองรับ keep-alive, ไม่รองรับ chunked encoding, ไม่รองรับ HTTP/2/3 เลย
แม้แต่นิดเดียว (หัวข้อ 61.10), ไม่มีการ route ไปยัง handler ต่างกันตาม path (โค้ดข้างบนตอบ "Hello" เดียวกัน
ทุก request ไม่ว่า path จะเป็นอะไร), ไม่มี validation ของ body อัตโนมัติ, ไม่มีระบบ middleware สำหรับ concern
ที่ใช้ร่วมกันทุก endpoint (logging จาก Part 60, authentication, CORS จากหัวข้อ 61.4)

#### สิ่งที่ Web Framework (Axum) ให้ "ฟรี"

**Part 62** จะแนะนำ **Axum** อย่างเป็นทางการ — แต่ก่อนไปถึงตรงนั้น สรุปสิ่งที่ framework ระดับ production ให้
เราโดยไม่ต้องเขียนเองแม้แต่บรรทัดเดียว (ตรงข้ามกับทุกอย่างที่ต้องทำมือในหัวข้อ 61.1):

- **Routing**: จับคู่ `(method, path)` ไปยังฟังก์ชัน handler ที่ถูกต้องอัตโนมัติ รวมถึง path parameter
  (`/tickets/{id}` แยก `id` ให้เป็นตัวแปรใช้งานได้ทันที ไม่ต้อง parse string เอง)
- **Extraction**: แปลง body/header/query param เป็น Rust type ที่มี type safety เต็มรูปแบบ (เช่น
  `Json<CreateTicketRequest>` ที่ใช้ `serde` แปลง JSON เป็น struct ให้อัตโนมัติ พร้อม error ที่เหมาะสม
  (`400`) ถ้า parse ไม่ผ่าน — สิ่งที่หัวข้อ 61.5 ทำมือด้วย `serde_json::from_str` ตรง ๆ)
- **Middleware**: จุดที่ logic ที่ใช้ร่วมกันทุก endpoint (logging แบบ Part 60, authentication ตรวจ
  `Authorization` header, CORS header จากหัวข้อ 61.4, rate limiting สำหรับ `429`) เขียนครั้งเดียวแล้ว "ครอบ"
  ทุก route ได้ ไม่ต้องเขียนซ้ำในทุก handler (Part 65 จะสอนเต็มรูปแบบ)
- **Error Handling**: กลไกแปลง `Result<T, E>` (ที่คุ้นเคยจาก Part 12/30-31) ให้กลายเป็น HTTP response ที่
  ถูกต้องอัตโนมัติ (error ประเภทหนึ่งแปลงเป็น `404`, อีกประเภทแปลงเป็น `500` เป็นต้น — Part 66 จะสอนเต็ม
  รูปแบบ)
- **Protocol Handling**: จัดการ keep-alive, chunked encoding, HTTP/2 multiplexing (หัวข้อ 61.10),
  connection pooling ให้ทั้งหมดโดย application code ไม่ต้องรู้รายละเอียดเหล่านี้เลย (สร้างอยู่บน `hyper` ซึ่ง
  เป็น HTTP library ระดับ production ที่ผ่านการทดสอบและ audit security มาอย่างเข้มงวด ต่างจากโค้ดขำ ๆ ใน
  หัวข้อ 61.1 โดยสิ้นเชิง)

จุดสำคัญที่สุดที่อยากให้จำจากบทนี้ก่อนไป Part 62: **ทุกสิ่งที่ Axum "ทำให้ฟรี" ไม่ใช่เวทมนตร์** — มันคือโค้ดที่
ทำสิ่งเดียวกันกับที่หัวข้อ 61.1-61.9 อธิบายไว้ทั้งหมด เพียงแต่เขียนมาแล้วอย่างถูกต้อง ทดสอบมาแล้วอย่างละเอียด
และครอบคลุม edge case ที่หัวข้อ 61.1 บอกไว้ว่ายังไม่ได้ทำ — เมื่อคุณเห็น Axum handler ที่ดูเรียบง่ายเพียงไม่กี่
บรรทัดใน Part 62 คุณจะรู้ว่าเบื้องหลังมันคือทุกกลไกที่เพิ่งเรียนไปในบทนี้ทั้งหมด

### 61.12 เปรียบเทียบกับ Ecosystem อื่น: ทุกภาษาแก้ปัญหาเดียวกันนี้

ความรู้ทั้งหมดในบทนี้ไม่ใช่ของ Rust หรือ Axum โดยเฉพาะ — มันคือคุณสมบัติของ HTTP เอง เพราะฉะนั้น framework ใน
ภาษาอื่นที่คุณอาจเคยเจอมาก่อนก็แก้ปัญหาเดียวกันกับที่หัวข้อ 61.11 สรุปไว้ (routing, extraction, middleware,
error handling) เพียงแค่ใช้คำศัพท์และ syntax ต่างกัน — เห็นภาพเทียบกันจะช่วยยืนยันว่าแนวคิดที่เรียนไปเป็น
แนวคิดระดับสากล ไม่ใช่เรื่องเฉพาะของ Rust:

| แนวคิด | Rust (Axum, Part 62+) | Node.js (Express) | Python (Flask) | Go (`net/http`) |
|---|---|---|---|---|
| นิยาม route | `.route("/tickets/{id}", get(handler))` | `app.get('/tickets/:id', handler)` | `@app.route('/tickets/<id>')` | `mux.HandleFunc("/tickets/", handler)` |
| อ่าน path param | extractor `Path<u64>` (type-safe, แปลงเป็น `u64` ให้อัตโนมัติพร้อม error ถ้าแปลงไม่ผ่าน) | `req.params.id` (**string เสมอ** ต้อง `parseInt` เอง ไม่มี type safety) | `<int:id>` ใน route (แปลงเป็น `int` ให้ แต่ error ตอน parse ไม่ผ่านต้องจัดการเอง) | `r.PathValue("id")` (string เสมอ, ต้อง `strconv.Atoi` เอง) |
| แปลง body เป็น struct | `Json<CreateTicketRequest>` (compile-time ตรวจ field ครบ/ชนิดตรง) | `req.body` (ต้อง validate เองทั้งหมด runtime, ไม่มี type safety ใน JS เอง) | `request.get_json()` แล้ว validate เอง (หรือใช้ Pydantic เสริม) | `json.NewDecoder(r.Body).Decode(&v)` (runtime error ถ้า field ไม่ตรง) |
| Middleware | `tower::Layer` (composable, type-checked ที่ compile time) | `app.use(middleware)` (runtime, ไม่มีการตรวจชนิดข้อมูลระหว่าง middleware) | `@app.before_request` decorator | `http.Handler` wrapping pattern |
| Error → HTTP response | `impl IntoResponse for MyError` (compile-time บังคับว่าทุก error path ต้องแปลงเป็น response ได้) | try/catch + error-handling middleware (runtime, พลาดจุดไหนจุดหนึ่งแอปพัง) | `@app.errorhandler(Exception)` | คืน error แล้วเรียก `http.Error()` เอง ทุกจุด |

**ข้อสังเกตที่สำคัญที่สุดจากตารางนี้**: จุดที่ Rust/Axum ต่างจากภาษาอื่นอย่างชัดเจนคือ**ระดับของการตรวจสอบที่
เกิดขึ้นตอน compile time เทียบกับ runtime** — ใน Express/Flask ถ้า handler สมมติว่า body มี field
`price_cents` แต่ client ไม่ส่งมา คุณจะได้ `undefined`/`None` แล้ว error runtime ตอนพยายามใช้งานมัน (หรือแย่
กว่านั้น ไม่ error แต่ทำงานผิดเงียบ ๆ) — ใน Axum ถ้า `CreateTicketRequest` (จากหัวข้อ 61.5) กำหนด
`price_cents: u64` ไว้ และ client ไม่ส่งมา extractor `Json<CreateTicketRequest>` จะ**ปฏิเสธ request ด้วย
`400` ก่อนที่ handler code ของคุณจะได้รันแม้แต่บรรทัดเดียว** — นี่คือผลพวงตรงของ type system ที่เรียนมาตั้งแต่
Part 1-12 ของหลักสูตรนี้ ที่ตอนนี้กำลังจะออกดอกออกผลเต็มที่ในโลกของ web development

### 61.13 HTTP Client ฝั่งเรียก: มุมมองสั้น ๆ ก่อนเจอ `reqwest`

บทนี้เน้นมุมมองฝั่ง **server** (รับ request, ตอบ response) เป็นหลัก เพราะเป็นมุมที่ Axum (Part 62+) จะยืนอยู่
แต่ระบบจริงมักต้องเป็น**ฝั่ง client** ด้วยเช่นกัน — เช่น service ตั๋วของเราอาจต้องเรียก payment gateway
ภายนอกเพื่อหักเงินตอนจองสำเร็จ (`POST` ไปยัง API ของผู้ให้บริการชำระเงิน) ทุกแนวคิดที่เรียนในบทนี้ใช้ได้กับ
ฝั่ง client เหมือนกันทุกประการ เพียงแค่สลับมุมมอง:

- Method/idempotency (61.2, 61.9): ฝั่ง client คือผู้**เลือก**ว่าจะยิง method ไหน และเป็นผู้ที่ต้อง**สร้าง**
  `Idempotency-Key` เองก่อนยิง `POST` ที่สำคัญ (เช่น การหักเงิน)
- Status code (61.3): ฝั่ง client คือผู้**อ่าน**และตัดสินใจว่าจะ retry (`503`, `429`) หรือหยุดทันที (`400`,
  `422`) ตามที่แบบฝึกหัดข้อ 2 ให้ลองออกแบบ
- Headers (61.4): ฝั่ง client คือผู้**ส่ง** `Authorization`, `Accept`, `Idempotency-Key` และเป็นผู้**อ่าน**
  `Content-Type`, `ETag` ที่ server ตอบมา

`curl` ที่ใช้ตลอดบทนี้ก็คือ HTTP client ตัวหนึ่ง (เขียนด้วย C) — ในโลก Rust เมื่อโค้ด Axum service ของเรา
(ที่กำลังจะเรียนใน Part 62) ต้องเป็น client เรียก service อื่นด้วย จะใช้ crate **`reqwest`** ซึ่งเป็น HTTP
client ระดับ production ที่สร้างอยู่บน `hyper` ตัวเดียวกับที่ Axum ใช้เป็นฐาน (สอดคล้องกับที่หัวข้อ 61.11
อธิบายไว้ว่า `hyper` คือ HTTP library หลักของ Rust ecosystem) — บทที่เกี่ยวกับการเรียก external API ในโมดูล
นี้จะแนะนำ `reqwest` โดยละเอียดอีกที ตอนนี้แค่รู้ไว้ก่อนว่าทุกแนวคิดของบทนี้ใช้ได้กับทั้งสองฝั่งของการสื่อสาร

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `Content-Length` ผิดจากความยาว Body จริง → Client ค้าง/Error

ถ้าคำนวณ `Content-Length` ผิด (เช่น hardcode ตัวเลขแทนการคำนวณจาก `body.len()` จริง) client จะรอ byte ที่ไม่
มีวันมาถึง เพราะ client เชื่อ header ว่า body ยาวเท่านี้ ไม่ใช่เชื่อความยาวจริงที่ได้รับ — พิสูจน์จริงด้วยการ
แก้ `raw_server.rs` ให้ตั้ง `Content-Length: 100` ทั้งที่ body จริงมีแค่ 37 ไบต์ แล้วยิง `curl -v` เข้าไป
(พร้อม `--max-time` กันค้างตลอดไป):

```text
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Content-Length: 100
< Connection: close
<
* transfer closed with 63 bytes remaining to read
curl: (18) transfer closed with 63 bytes remaining to read
Hello from a hand-rolled HTTP server!
```

curl ได้ body มาครบ 37 ไบต์จริง ๆ (`Hello from a hand-rolled HTTP server!` ปรากฏขึ้นมาจริง) แต่**ยัง error
ออกมา**เพราะมันรอ byte ที่เหลืออีก 63 ไบต์ตามที่ header สัญญาไว้ ("transfer closed with **63** bytes
remaining to read" = 100 - 37) จนกว่า connection จะถูกปิด (จาก `Connection: close`) ถึงจะยอมแพ้แล้วรายงาน
error — วิธีแก้: **คำนวณ `Content-Length` จาก `.len()` ของ byte จริงของ body เสมอ ไม่ hardcode เด็ดขาด**
(และถ้า body เป็น UTF-8 ที่มีตัวอักษรไทย ต้องใช้ `.len()` ของ `&[u8]`/`String` ที่นับ**ไบต์** ไม่ใช่
`.chars().count()` ที่นับตัวอักษร เพราะภาษาไทย 1 ตัวอักษรกินหลายไบต์ใน UTF-8)

### 2. ลืมบรรทัดว่าง (`\r\n\r\n`) คั่น Header กับ Body

ตาม RFC 7230 บรรทัดว่างคือ**สัญญาณเดียว**ที่บอกว่า header จบแล้ว — ถ้าลืม client จะตีความ body ว่าเป็น
header บรรทัดต่อไป (หรือแย่กว่านั้น สับสนจนไม่รู้ว่า header จบตรงไหน) พิสูจน์จริงด้วยการส่ง response ที่ตั้งใจ
ลืมบรรทัดว่าง:

```text
HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nContent-Length: 5\r\nHello
```

ยิง `curl -v` เข้าไปได้ผลลัพธ์:

```text
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Content-Length: 5
* transfer closed with 5 bytes remaining to read
curl: (18) transfer closed with 5 bytes remaining to read
```

สังเกตว่า**ไม่มีบรรทัด `<` เปล่า** ก่อนหน้า error เลย (เทียบกับกับดักข้อ 1 ที่มี) — แปลว่า curl **ไม่เคยรู้ว่า
header จบแล้ว** เพราะไม่เจอบรรทัดว่างที่คาดหวัง ผลคือ curl มองว่า header section ยังไม่ปิด "Hello" ที่ตั้งใจ
ส่งเป็น body **หายไปเงียบ ๆ ในสายตาของ client โดยสิ้นเชิง** (ไม่ได้ถูกตีความว่าเป็น body หรือ header เลย เพราะ
connection ปิดไปก่อนที่ parser จะไปถึงจุดนั้น) — ยืนยันว่าบรรทัดว่างนี้**ไม่ใช่รายละเอียดเล็ก ๆ ที่มองข้ามได้**
แต่เป็นส่วนหนึ่งของโครงสร้าง HTTP ที่ parser (ทั้งของเราและของ client) พึ่งพาอย่างสมบูรณ์

### 3. ใช้ `GET` สำหรับ Action ที่มี Side Effect (ละเมิดคุณสมบัติ Safe)

นี่คือกับดักเชิงออกแบบที่ไม่มี compiler error เตือน แต่ส่งผลร้ายแรงในโลกจริง — เหตุการณ์คลาสสิกที่มักถูกยกมา
เล่าในวงการคือ **Google Web Accelerator** (เครื่องมือของ Google ในยุค 2000s ที่ prefetch ทุก link ในหน้าเว็บ
ล่วงหน้าเพื่อให้โหลดเร็วขึ้นตอนคลิกจริง) — มันสมมติ (ถูกต้องตามสัญญาความหมายของ HTTP) ว่า **`GET` เป็น safe
method เสมอ** เพราะฉะนั้น prefetch ลิงก์ที่เป็น `GET` ทั้งหมดโดยไม่มีความเสี่ยง แต่เว็บแอปพลิเคชันจำนวนไม่น้อย
ในยุคนั้นสร้างลิงก์อย่าง `<a href="/deleteItem?id=42">ลบ</a>` (ใช้ `GET` เพราะเขียนง่ายกว่า form `POST`) —
ผลคือ Google Web Accelerator prefetch ลิงก์ "ลบ" ทุกอันบนหน้าเว็บโดยผู้ใช้ไม่ได้คลิกอะไรเลย ทำให้ข้อมูลถูกลบ
หายไปโดยไม่ได้ตั้งใจในหลายเว็บแอปพลิเคชัน

ในโดเมนตั๋วของเรา ถ้าออกแบบ `GET /api/v1/tickets/4821/cancel` (ใช้ `GET` ยกเลิกตั๋ว) — ไม่ใช่แค่ browser
extension ที่จะเป็นปัญหา แต่**search engine crawler** ก็อาจตาม link นี้ไป crawl ด้วย เพราะมันมีสิทธิ์เต็มที่
จะสมมติว่า `GET` ปลอดภัยเสมอตามสเปก — วิธีแก้คือ**ยึดสัญญาความหมายของ method อย่างเคร่งครัด**เสมอ (ตามตาราง
หัวข้อ 61.2): action ที่เปลี่ยนสถานะต้องเป็น `POST`/`PUT`/`PATCH`/`DELETE` เท่านั้น ไม่มีข้อยกเว้นแม้จะดู
"สะดวก" กว่าตอนเขียน frontend ก็ตาม

### 4. Header Name เป็น Case-Insensitive แต่โค้ด Parser เขียนแบบ Exact-Match

RFC 7230 ระบุชัดว่าชื่อ header เป็น **case-insensitive** (`Content-Type`, `content-type`, `CONTENT-TYPE`
ถือว่าเป็น header ตัวเดียวกันทั้งหมด) — HTTP/2 (RFC 7540) ยิ่งไปไกลกว่านั้นด้วยการ**บังคับ**ให้ทุก header
name ต้องเป็นตัวพิมพ์เล็กเสมอตอนส่งจริง แต่โค้ด parser ที่เขียนขึ้นแบบไม่ทันคิดเรื่องนี้ (เช่น ใช้
`HashMap<String, String>` แล้วค้นด้วย exact string match ตามที่ "เคยเห็นจาก curl") จะพังทันทีที่เจอ client ที่
ส่ง header ด้วย case ต่างออกไป:

```rust
let mut headers: HashMap<String, String> = HashMap::new();
headers.insert("content-type".to_string(), "application/json".to_string());

let lookup_result = headers.get("Content-Type"); // ค้นด้วยตัวพิมพ์ใหญ่ตามที่คาดว่า client จะส่งมา
println!("headers.get(\"Content-Type\") = {lookup_result:?}");
```

รันจริงได้ผลลัพธ์ที่พิสูจน์บั๊กนี้ตรง ๆ:

```text
headers.get("Content-Type") = None
headers.get("content-type") = Some("application/json")
get_header_case_insensitive("Content-Type") = Some("application/json")
```

`headers.get("Content-Type")` คืน **`None`** ทั้งที่ header นี้ **"มีอยู่จริง"** ในความหมายของ HTTP (แค่เก็บ
มาด้วยตัวพิมพ์เล็ก) — วิธีแก้ที่ถูกต้องคือ**normalize เป็น lowercase ทุกครั้งทั้งตอนเก็บและตอนค้นหา** (ฟังก์ชัน
`get_header_case_insensitive` ในตัวอย่างด้านบน) หรือใช้ type ที่ออกแบบมาให้ case-insensitive โดยตรง — นี่คือ
เหตุผลอีกข้อที่ **Axum ใช้ `http::HeaderMap`** (จาก crate `http` ที่เป็นรากฐานร่วมของ ecosystem HTTP ใน Rust
ทั้งหมด รวมถึง `hyper`/`reqwest`) แทน `HashMap<String, String>` ธรรมดา — `HeaderMap` จัดการเรื่อง
case-insensitivity ให้ถูกต้องตามสเปกโดยอัตโนมัติ ไม่ต้องกังวลเรื่องนี้เองอีกเลยเมื่อไปถึง Part 62

### 5. สับสนระหว่าง `400 Bad Request` กับ `422 Unprocessable Entity`

กับดักเชิง design ที่พบบ่อยมากคือใช้ `400` กับทุกกรณีที่ request "ไม่ถูกต้อง" โดยไม่แยกว่า**ไม่ถูกต้องแบบ
ไหน** — ตามตารางหัวข้อ 61.3: `400` ควรสงวนไว้กับกรณีที่ **parse ไม่ผ่านตั้งแต่ syntax** (เช่น JSON เขียนผิด
รูปแบบ อ่านไม่ออกเลย) ส่วน `422` คือกรณีที่ **parse ผ่านสมบูรณ์แต่ค่าขัดกับกฎทางธุรกิจ** (เช่น
`price_cents: -500` เป็น JSON number ที่ถูกต้องตาม syntax เป๊ะ แต่ค่าติดลบไม่สมเหตุสมผลสำหรับราคาตั๋ว) — การ
ใช้ `400` ปนกันทั้งสองแบบทำให้ client ที่พยายามเขียนโค้ด handle error แยกตามประเภท (เช่น "ถ้า 400 คือ bug
ในโค้ด client เอง ควร log ไว้ดีบั๊ก ถ้า 422 คือ input ของ user ผิด ควรโชว์ข้อความแจ้ง user") ทำแบบนั้นไม่ได้
เลย เพราะสอง error type ที่มีความหมายต่างกันโดยพื้นฐานถูกยัดเป็นโค้ดเดียวกัน

### 6. คิดว่า Idempotent แปลว่า "ไม่มี Side Effect" (สับสนกับ Safe)

หัวข้อ 61.2 อธิบายไว้แล้วว่า idempotent วัดที่**ผลลัพธ์สุดท้าย** ไม่ใช่ "ไม่มีอะไรเกิดขึ้นเมื่อเรียกซ้ำ" —
กับดักที่พบบ่อยคือนักพัฒนามือใหม่คิดว่าถ้า method หนึ่ง idempotent แล้วก็ "ปลอดภัยเรียกซ้ำได้เหมือน `GET`"
โดยไม่แยกว่า `PUT`/`DELETE` **ยังมี side effect จริง** (เขียนฐานข้อมูล, ส่ง event ไปยัง message queue ให้
service อื่นประมวลผลต่อ) เพียงแต่ side effect นั้นไม่ทวีคูณขึ้นเมื่อเรียกซ้ำเท่านั้น — ถ้าระบบมี logic ที่ผูก
กับ "ทุกครั้งที่มีการเขียนฐานข้อมูลสำเร็จ ให้ส่ง notification" การเรียก `PUT` ซ้ำสองครั้งด้วย body เดิม (แม้
end state ของ resource เหมือนกัน) **ก็ยังส่ง notification ซ้ำสองครั้งอยู่ดี** ถ้าไม่ได้ออกแบบ dedup แยกส่วนนั้น
ไว้เอง — บทเรียนคือ: idempotency เป็นคุณสมบัติของ**สถานะสุดท้ายของ resource หลัก** เท่านั้น ไม่ได้แปลว่า
**ทุก side effect ทางอ้อม**ในระบบจะปลอดภัยจากการเรียกซ้ำไปด้วยโดยอัตโนมัติ — ระบบที่ซับซ้อน (มี notification,
webhook, event ต่อเนื่อง) ยังต้องคิดเรื่อง idempotency ของ side effect เหล่านั้นแยกต่างหาก (เชื่อมกับหัวข้อ
61.9 และ Part 81-84 เรื่อง distributed system)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** แก้ไข `raw_server.rs` จากหัวข้อ 61.1 ให้ตอบสนองต่อ path ที่ต่างกันด้วยเนื้อหาต่างกัน: ถ้า
   `path == "/hello"` ตอบ body `"Hello!"`, ถ้า `path == "/goodbye"` ตอบ body `"Goodbye!"`, กรณีอื่นตอบ status
   line เป็น `HTTP/1.1 404 Not Found` พร้อม body `"Not Found"` — ทดสอบด้วย `curl` ยิงทั้งสาม path แล้วเทียบ
   status code/body ที่ได้กับที่ควรจะเป็น
   *Hint*: ใช้ `match path { "/hello" => ..., "/goodbye" => ..., _ => ... }` แล้วให้แต่ละ branch คืน tuple
   `(status_line, body)` ก่อนประกอบเป็น response string เดียวกัน — อย่าลืมว่า `Content-Length` ต้องคำนวณจาก
   `body.len()` ของ branch ที่เลือกจริง ไม่ใช่ hardcode ค่าเดียว

2. **(กลาง)** เขียนฟังก์ชัน `classify_status(code: u16) -> &'static str` ที่รับ status code แล้วคืนชื่อ class
   ที่ถูกต้อง (`"Informational"`, `"Success"`, `"Redirection"`, `"Client Error"`, `"Server Error"`,
   `"Unknown"` สำหรับค่านอกช่วง 100-599) จากนั้นเขียนอีกฟังก์ชัน `is_retryable(code: u16) -> bool` ที่บอกว่า
   ถ้า client ได้ status code นี้กลับมา **ควร retry request เดิมโดยอัตโนมัติหรือไม่** (คำใบ้: `429` และ `503`
   ควร retry ได้ถ้ามี backoff ที่เหมาะสม, `500` ควร retry ได้ถ้า request เดิมเป็น idempotent method เท่านั้น,
   `400`/`401`/`403`/`404`/`409`/`422` **ไม่ควร** retry อัตโนมัติเด็ดขาดเพราะปัญหาอยู่ที่ตัว request เอง
   retry ซ้ำก็ยังผิดเหมือนเดิม)
   *Hint*: เขียน unit test (`#[test]`) ยืนยันว่า `is_retryable(404)` ต้องเป็น `false` และ `is_retryable(503)`
   ต้องเป็น `true` เป็นอย่างน้อย เพื่อป้องกันไม่ให้ logic กลับตาลปัตรโดยไม่ตั้งใจ

3. **(ยาก)** ขยาย `idempotency_demo.rs` จากหัวข้อ 61.9 ให้เก็บ**ทั้ง idempotency key และ hash ของ request
   body** คู่กัน — ถ้า client ส่ง key ที่เคยเห็นมาก่อนแต่ body **ต่างจากครั้งแรก** ให้ฟังก์ชันคืนค่าเป็น
   `Result<(u64, bool), String>` ที่เป็น `Err("idempotency key conflict: request body ต่างจากครั้งก่อน")`
   แทนที่จะคืนผลลัพธ์เก่าไปเงียบ ๆ (ตามที่กับดักย่อยในหัวข้อ 61.9 เตือนไว้)
   *Hint*: เปลี่ยน `HashMap<String, u64>` เป็น `HashMap<String, (u64, u64)>` โดยค่าที่สองเป็น hash ของ body
   (ใช้ `std::collections::hash_map::DefaultHasher` กับ `std::hash::{Hash, Hasher}` ก็พอสำหรับแบบฝึกหัดนี้
   ไม่ต้องใช้ cryptographic hash) แล้วเทียบ hash ก่อนตัดสินใจว่าจะคืนผลลัพธ์เก่าหรือ error

4. **(ยาก/ประยุกต์)** ออกแบบ (เขียนเป็นตาราง endpoint พร้อม method/status code ที่คาดหวัง ไม่ต้องเขียนโค้ด
   จริง) REST API เต็มรูปแบบสำหรับระบบ**จองโต๊ะร้านอาหาร** ให้ครอบคลุม: (ก) resource หลักคืออะไร (โต๊ะ? การจอง?
   ร้าน?) กำหนด URI ให้เหมาะสมตามหลักหัวข้อ 61.7-61.8 (ข) endpoint สำหรับดูโต๊ะที่ว่างในช่วงเวลาหนึ่ง (ต้อง
   ตัดสินใจว่าเงื่อนไขเวลาควรอยู่ใน query param หรือที่อื่น พร้อมให้เหตุผล) (ค) endpoint สำหรับสร้างการจอง
   พร้อมพิจารณาว่าจำเป็นต้องมี `Idempotency-Key` หรือไม่ (ง) สถานการณ์ที่ควรตอบ `409 Conflict` (จ) สถานการณ์
   ที่ควรตอบ `403` เทียบกับ `404` (ลูกค้าพยายามดูรายละเอียดการจองของคนอื่น ควรตอบแบบไหนระหว่างสองอย่างนี้ และ
   เพราะอะไร — คำใบ้: มีข้อถกเถียงจริงในวงการเรื่องนี้เกี่ยวกับการรั่วไหลของข้อมูลว่า resource มีอยู่จริงหรือไม่)
   *Hint*: เริ่มจากเขียน resource เป็นคำนามก่อน (เช่น `/api/v1/restaurants/{restaurant_id}/tables`,
   `/api/v1/reservations/{id}`) แล้วค่อยเติม method/query param ทีละ endpoint — ระวังอย่าให้ endpoint ไหน
   กลายเป็น RPC-style (เช่น `/api/v1/reserveTable`) โดยไม่ทันสังเกต

## สรุป

บทนี้พาไปเข้าใจ HTTP ในระดับ wire โดยไม่พึ่ง framework ใด ๆ เลย — เขียน raw HTTP server ด้วย
`std::net::TcpListener` ที่ parse request line/header ด้วยมือ (หัวข้อ 61.1) เพื่อพิสูจน์ว่า HTTP คือข้อความ
ที่มีโครงสร้างตายตัวส่งผ่าน TCP ไม่ใช่ "JSON วิ่งไปมา" อย่างที่มักเข้าใจผิด จากนั้นเจาะลึก **HTTP methods**
พร้อมคุณสมบัติ safe/idempotent (61.2), **status codes** ทั้ง 5 class พร้อมตัวที่ต้องรู้ขึ้นใจ (61.3),
**headers** สำคัญที่ควบคุม content negotiation, caching, CORS (61.4), **body** และ content negotiation ด้วย
JSON/serde (61.5), **statelessness** ที่เชื่อมกับความเข้าใจเรื่อง shared state จาก Part 39-40 และเป็นฐานของ
horizontal scaling (61.6), **REST เป็นสถาปัตยกรรมไม่ใช่ protocol** พร้อม Richardson Maturity Model และ
HATEOAS (61.7), **URL structure** แยก path/query/body ให้ถูกที่ (61.8), **idempotency ในทางปฏิบัติ**ด้วย
Idempotency-Key เพื่อแก้ปัญหา retry ในระบบแบบกระจาย (61.9), ภาพรวมของ **HTTP/1.1 vs 2 vs 3** ในระดับที่
backend developer ควรรู้ (61.10) และปิดท้ายด้วยการสรุปว่า**ทำไมต้องมี web framework** — ทุกความเจ็บปวดที่
เจอในหัวข้อ 61.1-61.9 คือสิ่งที่ Axum จะแก้ให้แบบ "ฟรี" ผ่าน routing, extraction, middleware, และ error
handling (61.11)

ความรู้ทั้งหมดในบทนี้**ไม่ผูกกับ framework ไหนเลย** — ไม่ว่าคุณจะเขียน Axum, Actix-web (Part 67-70), หรือแม้แต่
เปลี่ยนไปเขียนภาษาอื่นในอนาคต หลักการเรื่อง method semantics, status code, header, REST, idempotency เหล่านี้
ยังคงเป็นจริงเสมอ เพราะมันคือคุณสมบัติของ **HTTP protocol เอง** ไม่ใช่ของ library ตัวใดตัวหนึ่ง — Part 62
จะเริ่มลงมือเขียน Axum จริงเป็นครั้งแรก และจะอ้างอิงกลับมาที่บทนี้ตลอดเวลาทุกครั้งที่อธิบายว่า Axum feature
หนึ่ง ๆ "แทนที่" การทำมืออะไรที่เพิ่งเรียนไปในบทนี้

---

**Part ก่อนหน้า:** [Logging และ Tracing เบื้องต้น (log, tracing crate)](part-060-logging-tracing-basics.md) | **Part ถัดไป:** [แนะนำ Axum Framework](part-062-axum-intro.md)
