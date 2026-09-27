# Part 98: Observability: Metrics ด้วย Prometheus

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: สูง | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า **logs** (Part 60), **metrics** (บทนี้), และ **traces** (Part 99) คือสามเสาหลักของ
  observability ที่ทำงาน**เสริมกัน**ไม่ใช่แทนกัน — รู้ว่าเมื่อไหร่ควรมองที่ตัวไหนก่อนเวลาแก้ปัญหาจริง และทำไม
  ระบบ production ที่จริงจังต้องมีครบทั้งสามอย่าง ไม่ใช่แค่อย่างใดอย่างหนึ่ง
- แยกแยะ metric ทั้งสี่ประเภทของ Prometheus — **Counter**, **Gauge**, **Histogram**, **Summary** — ได้ถูกต้อง
  100% ว่าแต่ละแบบเหมาะกับข้อมูลลักษณะไหน พร้อมยกตัวอย่างจากระบบห้องสมุด/API จริงได้ทันทีโดยไม่ต้องท่อง
- อธิบายโมเดล **pull-based** ของ Prometheus (server ไป "ดูด" ข้อมูลจาก endpoint `/metrics` ของแอปเป็นระยะ ๆ)
  ได้ว่าทำไมถึงออกแบบมาแบบนี้ ต่างจากโมเดล **push-based** (เช่น StatsD) อย่างไร และข้อดี/ข้อเสียของแต่ละแบบ
- ติดตั้ง **`metrics`** crate (facade แบบเดียวกับ `log` ใน Part 60) คู่กับ **`metrics-exporter-prometheus`**
  (backend ที่แปลง metric เป็น Prometheus text format จริง) เข้า Axum app (ต่อยอด Part 62) แล้วเปิด endpoint
  `/metrics` ที่ `curl` จริงยิงแล้วเห็น output ตามสเปก Prometheus ได้ทันที
- เขียน **middleware** (ต่อยอด Part 65) ที่วัด HTTP request counter และ latency histogram ให้ **ทุก endpoint
  โดยอัตโนมัติ** โดย handler ไม่ต้องรู้เรื่อง metrics เลยแม้แต่บรรทัดเดียว — พิสูจน์ด้วยการยิง request จริงหลาย
  แบบแล้วเห็นค่าใน `/metrics` เปลี่ยนไปตามจริง
- เพิ่ม **custom business metric** เฉพาะโดเมน (เช่น จำนวนหนังสือที่ถูกยืมทั้งหมด, จำนวนสำเนาที่ว่างอยู่ ณ ขณะนี้)
  เข้าไปใน handler `borrow_book`/`return_book` ของระบบห้องสมุด (ต่อยอด Module 4-5 และ Part 92-94)
- ระบุและแก้ **กับดัก high-cardinality label** ได้ — เข้าใจว่าทำไมการ label ด้วยค่าที่ไม่จำกัด (เช่น raw path
  ที่มี id ติดมา, user id ดิบ ๆ) ทำให้จำนวน time series ระเบิดจนระบบ monitoring รับไม่ไหว พร้อมพิสูจน์ตัวเลขจริง
  ว่าต่างกันแค่ไหนระหว่างวิธีที่ถูกกับผิด
- รัน **Prometheus server จริง** ในเครื่อง (ผ่าน Docker ต่อยอด Part 96) ตั้ง `prometheus.yml` ให้ scrape แอปของ
  ตัวเอง แล้วยืนยันว่า target ขึ้นสถานะ `UP` จริงผ่านหน้า `/targets` และ API ของ Prometheus เอง
- เขียนและรัน **PromQL** พื้นฐาน — `rate()` คำนวณอัตราคำขอต่อวินาทีจาก counter, `histogram_quantile()` คำนวณ
  p50/p95/p99 latency จาก histogram — ด้วยการยิง query จริงไปที่ Prometheus API แล้วอ่านผลลัพธ์จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 60 (Logging และ Tracing เบื้องต้น)**: บทนี้วางกรอบ "observability triad" โดยอ้างอิง logs กับ `log`/
  `tracing` ที่ Part 60 สอนไว้ตรง ๆ ในหัวข้อแรก และดึง**แนวคิด facade pattern** (`log` เป็น interface,
  `env_logger` เป็น backend) มาอธิบาย `metrics`/`metrics-exporter-prometheus` แบบเดียวกันเป๊ะ — ถ้าจำ analogy
  นี้จาก Part 60 ไม่ได้ ให้กลับไปทวนหัวข้อ 60.2 ก่อน เพราะบทนี้จะอ้างอิงกลับไปตลอดโดยไม่อธิบาย facade pattern
  ซ้ำจากศูนย์
- **Part 62-64 (Axum: Routing, Handlers, State Management, Extractors)**: ตัวอย่างทั้งหมดในบทนี้เป็น Axum
  application ที่มี `Router`, handler แบบ async, `State<T>` extractor, และ `Path<T>` extractor ตามที่สาม Part
  นี้สอนไว้ — บทนี้สมมติว่าคุณสร้าง endpoint พื้นฐานได้แล้วโดยไม่อธิบายกลไก extractor ซ้ำ
- **Part 65 (Axum: Middleware)**: middleware ที่วัด HTTP metrics อัตโนมัติในหัวข้อ 98.5 ใช้
  `axum::middleware::from_fn` แบบเดียวกับที่ Part 65 สอนไว้ตรง ๆ และที่สำคัญกว่านั้นคือบทนี้จะพิสูจน์ **ความ
  แตกต่างจริงระหว่าง `.layer()` กับ `.route_layer()`** ที่ Part 65 หัวข้อ 65.5-65.6 พูดถึงไว้แล้วว่ามีผลต่อ
  พฤติกรรม (404 ถูก middleware เห็นหรือไม่) — บทนี้จะนำความเข้าใจนั้นมาใช้จริงกับปัญหา metrics cardinality
  โดยตรง ถ้า Part 65 ยังไม่แน่น ให้กลับไปทวนก่อน
- **Part 66 (Error Handling ใน Axum)**: handler ในบทนี้คืน `Result<T, E>` ที่แปลงเป็น HTTP status code ตามที่
  Part 66 สอนไว้ (เช่น `404`, `409`) — บทนี้ไม่สอน error handling ซ้ำ แต่ metrics ที่วัด status code จะอ้างอิง
  status ที่ Part 66 ทำให้ handler คืนได้ถูกต้องอยู่แล้ว
- **Part 70-71 (SQLx: Database และ Migrations)** และ **Part 92-94 (Full-Stack Project)**: หัวข้อ 98.6 และ 98.10
  ต่อยอดตรงจาก handler `borrow_book`/`return_book` ของระบบห้องสมุดที่ Part 92 เขียนไว้เต็มรูปแบบด้วย SQLx —
  บทนี้จะแสดงว่าจะเพิ่มโค้ด metrics เข้าไปใน handler เหล่านั้น**ตรงจุดไหน**โดยไม่ต้องอธิบาย transaction/SQLx
  ซ้ำ (สมมติว่าคุณมี `AppState { db: PgPool, ... }` และ handler ที่ทำงานถูกต้องอยู่แล้วตามที่ Part 92 สอนไว้)
- **Part 96 (Docker และ Containerization สำหรับ Rust)**: หัวข้อ 98.8 รัน Prometheus server จริงผ่าน `docker run`
  — บทนี้ใช้ Docker image สำเร็จรูป (`prom/prometheus`) ไม่ใช่ build image เอง จึงไม่จำเป็นต้องเขียน
  `Dockerfile` ใหม่ แต่สมมติว่าคุณมี Docker ทำงานอยู่แล้วตามที่ Part 96 ติดตั้งไว้ (หัวข้อ 98.10 ยังโยงกลับไปที่
  `docker-compose` ซึ่ง Part 96 แนะนำไว้เป็นทางเลือกเมื่อ containerize ทั้งแอปและ Prometheus พร้อมกัน)
- **Part 95 (Testing Web Applications แบบครบวงจร)**: หัวข้อ 98.11 เขียน integration test ให้ metrics
  middleware ด้วย `tower::ServiceExt::oneshot()` ตามเทคนิคเดียวกับที่ Part 95 สอนไว้สำหรับทดสอบ Axum handler
  ทั่วไป — บทนี้ไม่อธิบายกลไก `oneshot()`/`tokio::test` ซ้ำจากศูนย์
- **Part 99 (Observability: Distributed Tracing ด้วย OpenTelemetry)**: บทถัดไปจะพาไปดูเสาที่สามของ
  observability triad ที่บทนี้แค่แนะนำแนวคิดไว้ก่อน (98.1) — เมื่อจบ Part 99 คุณจะมีภาพครบทั้งสามเสา logs +
  metrics + traces พร้อมกัน

## เนื้อหา

### 98.1 Observability Triad: Logs, Metrics, Traces — สามเสาหลักที่ทำงานเสริมกัน

ลองนึกภาพสถานการณ์จริงที่เกิดขึ้นกับทุกทีมที่ดูแลระบบ production: เวลาตี 3 ทีม on-call ได้รับ alert ว่า API
ระบบห้องสมุด (ที่ Part 92-94 สร้างไว้) **ช้าผิดปกติ** — คำถามแรกที่ต้องตอบคือ "ช้าตรงไหน กระทบใคร แค่ไหน
และทำไม" คำตอบของคำถามนี้ไม่มีเครื่องมือตัวเดียวที่ตอบได้ครบ — ต้องใช้**สามเครื่องมือที่ต่างกัน** ทำงานร่วมกัน
ซึ่งวงการ observability เรียกรวมกันว่า **"สามเสาหลักของ observability" (the three pillars of
observability)**:

| เสาหลัก | ตอบคำถาม | ลักษณะข้อมูล | ต้นทุนที่ scale ใหญ่ | บทที่สอน |
|---|---|---|---|---|
| **Logs** | "เกิดอะไรขึ้น**เป๊ะ ๆ**ที่จุดนี้ พร้อม context ละเอียด?" | เหตุการณ์แยกเป็นชิ้น ๆ (discrete event) มีข้อความ + field ละเอียดสูง | แพงมากถ้าเก็บทุกบรรทัดที่ verbosity สูง ๆ ตลอดเวลา (ข้อมูลไม่ถูก aggregate เลย) | Part 60 |
| **Metrics** | "ระบบ**โดยรวม**เป็นอย่างไรตามเวลา มีแนวโน้มอะไรเปลี่ยนไปบ้าง?" | ตัวเลขที่ถูก**รวม (aggregate)** ไว้แล้วตามช่วงเวลา (ไม่ใช่ event เดี่ยว ๆ) | **ถูกมาก** เพราะเก็บแค่ตัวเลขสรุปต่อช่วงเวลา ไม่ใช่ทุก event ดิบ — เก็บ/query ได้เร็วแม้มี traffic สูงมาก | บทนี้ (98) |
| **Traces** | "request **เส้นทางนี้เส้นทางเดียว** เดินทางผ่านระบบยังไง จุดไหนกินเวลานานสุด?" | เส้นทางของ**หนึ่ง request** ข้าม service/function หลายตัว พร้อม timing แต่ละช่วง | ปานกลาง-แพง ถ้าเก็บทุก request (มักใช้ sampling ลดปริมาณ) | Part 99 |

**ทำไมต้นทุนที่ scale ใหญ่ต่างกันขนาดนี้ — ตัวอย่างตัวเลขที่ทำให้เห็นภาพชัด**: สมมติ API ระบบห้องสมุดมี
traffic 1,000 request ต่อวินาที รันต่อเนื่อง 24 ชั่วโมง ถ้าเก็บ **log หนึ่งบรรทัดต่อ request** (แนวทางปกติของ
logging) จะได้ประมาณ **86.4 ล้านบรรทัด log ต่อวัน** ที่ต้องเก็บ, index, และค้นหาได้ — พื้นที่จัดเก็บโตเป็นสัด
ส่วนตรงกับจำนวน request เสมอ (linear growth) ในทางกลับกัน ถ้าเก็บเป็น **metric** (เช่น counter
`http_requests_total` กับ histogram latency) ปริมาณข้อมูลที่ Prometheus ต้องเก็บ**ไม่ขึ้นกับจำนวน request
เลย** — ไม่ว่าจะมี 1,000 หรือ 1,000,000 request ต่อวินาที จำนวน **time series** (ซึ่งขึ้นกับจำนวน label
combination ที่มีขอบเขตจำกัด ตามที่หัวข้อ 98.7 จะพิสูจน์ให้เห็น) ยังเท่าเดิม มีแค่**ตัวเลขในแต่ละ time series
ที่เปลี่ยนไป**และ Prometheus เก็บแค่ค่าใหม่ทุกรอบ scrape (ทุก 5-15 วินาทีตามที่ config ไว้) ไม่ใช่ทุก request
ดิบ — นี่คือเหตุผลเชิงลึกที่ metrics ถูกเรียกว่า**ถูกกว่า logs มากที่ scale ใหญ่**: ต้นทุนของ metrics ขึ้นกับ
"จำนวน time series ที่ design ไว้" ไม่ใช่ "จำนวน request ที่เกิดขึ้นจริง" ในขณะที่ต้นทุนของ logs ขึ้นกับจำนวน
request ตรง ๆ เสมอ

**เครื่องมือจริงที่ใช้กับแต่ละเสาในโลกจริง (ระดับ awareness เท่านั้น ไม่ใช่เนื้อหาหลักสูตร)**: ระบบ
observability ระดับ production มักมีเครื่องมือแยกกันสามชุดสำหรับสามเสานี้ (แม้บางผลิตภัณฑ์การค้าจะรวมทั้งสาม
อย่างไว้ในแพลตฟอร์มเดียว) — สำหรับ logs มักใช้ Elasticsearch/Loki, สำหรับ metrics มักใช้ Prometheus (ตัวที่
บทนี้สอน) หรือ Mimir/Thanos สำหรับ metrics ที่ scale ข้ามหลาย cluster/data center, และสำหรับ traces มักใช้
Jaeger หรือ Tempo (Part 99 จะพาไปดู OpenTelemetry ซึ่งเป็นมาตรฐานกลางสำหรับส่ง trace ไปยังเครื่องมือเหล่านี้)
— บทนี้ไม่ลงรายละเอียดของเครื่องมืออื่นนอกจาก Prometheus เพราะมันคือมาตรฐาน de facto ของฝั่ง metrics ใน
ecosystem แบบ open-source และเป็นสิ่งที่ `metrics-exporter-prometheus` ผูกไว้ตรงตัวตามชื่อ

**กลับไปที่สถานการณ์ตี 3**: ลำดับการใช้งานจริงที่ทีม on-call มักทำคือ

1. **เริ่มจาก metrics** (เพราะถูกและเร็วที่สุดในการดูภาพรวม) — เปิด dashboard ดู
   `http_request_duration_seconds` (histogram ที่บทนี้จะสอนให้สร้าง) แล้วเห็นว่า **p99 latency ของ endpoint
   `/books/{id}/borrow` พุ่งขึ้นจาก 50ms เป็น 3 วินาทีตั้งแต่เวลา 02:47** — นี่คือคำตอบของ "ช้าตรงไหน (endpoint
   ไหน) และตั้งแต่เมื่อไหร่" ที่ metrics ตอบได้ดีที่สุดเพราะเป็นข้อมูลสรุปที่ query เร็วแม้ traffic สูง
2. **ขยับไป traces** (Part 99) เพื่อดูว่า request หนึ่งตัวของ endpoint นั้นไปกินเวลาอยู่ที่ไหน — เปิด trace
   ของ request ที่ช้าตัวหนึ่งดู แล้วเห็นว่า **90% ของเวลาทั้งหมดอยู่ที่ span การรอ database lock** (ไม่ใช่ที่
   business logic) — นี่คือคำตอบของ "ทำไมช้า ที่จุดไหนในเส้นทางของ request เดียว"
3. **ปิดท้ายด้วย logs** (Part 60) เพื่อดู**รายละเอียดที่มีบริบทเจาะจง**ของ request ที่มีปัญหา — grep log ของ
   ช่วงเวลานั้นด้วย request id ที่ได้จาก trace แล้วเห็นข้อความ error/warning ที่อธิบายว่า **transaction ไหน
   ค้างอยู่นานเพราะอะไร** (เช่น "constraint violation retry ครั้งที่ 12") — นี่คือรายละเอียดระดับเม็ดเล็กที่
   metrics/traces ไม่มีพื้นที่พอจะเก็บ (metrics เป็นแค่ตัวเลข ไม่มีข้อความอธิบาย, trace บอกแค่ span ไหนกินเวลา
   ไม่บอกว่า "ทำไม" ในระดับ business logic)

**ข้อสังเกตสำคัญที่เป็นหัวใจของหัวข้อนี้**: ทั้งสามเสาไม่ได้แข่งกัน แต่ละตัวถูกออกแบบมาให้เหมาะกับ**ขนาดข้อมูล
และคำถามที่ต่างกัน** — ถ้าพยายามใช้ log เพื่อตอบคำถามระดับ "แนวโน้มโดยรวม" (เช่น "p99 latency เดือนที่แล้วเทียบ
เดือนนี้เป็นยังไง") คุณจะต้อง grep/aggregate log หลายสิบล้านบรรทัดเอง ซึ่งช้ามากและมีค่าใช้จ่ายสูงมากเทียบกับ
metrics ที่ถูกออกแบบมาให้ตอบคำถามแบบนี้โดยตรงตั้งแต่ต้น ในทางกลับกัน ถ้าพยายามใช้ metrics ตอบคำถาม "request
เจาะจงตัวนี้ทำไมถึง error" คุณจะไม่มีทางรู้ได้เลยเพราะ metrics ไม่เก็บรายละเอียดระดับ request เดี่ยว ๆ (ถูก
aggregate ทิ้งไปแล้วตั้งแต่ก่อนจะถูกเก็บ)

บทนี้เจาะจงไปที่เสาหลักตัวกลาง — **metrics** — และตัวระบบที่เป็นมาตรฐานโลกจริงของฝั่ง open-source สำหรับเก็บ
และ query metrics: **Prometheus**

### 98.2 Metric Types: Counter, Gauge, Histogram, Summary

Prometheus (และ ecosystem รอบตัวมันแทบทั้งหมด รวมถึง `metrics` crate ที่จะใช้ในบทนี้) นิยาม metric ไว้สี่
ประเภทหลัก — เลือกประเภทผิดคือกับดักที่พบบ่อยที่สุดของคนที่เพิ่งเริ่มทำ metrics เพราะมันไม่ compile error ให้
เห็น (ทุกประเภทเก็บเป็นตัวเลข `f64` เหมือนกันหมดทางเทคนิค) แต่ทำให้ query ทีหลังผิดความหมายไปเลยทั้งระบบ

#### Counter: ตัวเลขที่**เพิ่มขึ้นอย่างเดียว** ไม่เคยลด

**Counter** คือตัวเลขที่มีทิศทางเดียว — **เพิ่มขึ้นเรื่อย ๆ ตลอดชีวิตของโปรเซส ไม่เคยลดลง** (ยกเว้นตอนโปรเซส
restart ที่มันรีเซ็ตกลับไป 0 ซึ่งเป็นพฤติกรรมที่ยอมรับได้และ Prometheus มีกลไกจัดการเรื่องนี้เองในหัวข้อ 98.9)
ตัวอย่างที่เหมาะกับ Counter ในระบบห้องสมุด:

- `http_requests_total` — จำนวน HTTP request ที่ได้รับทั้งหมดตั้งแต่โปรเซสเริ่มทำงาน (ไม่มีทางที่จำนวน request
  ที่รับไปแล้วจะ "ลดลง" ได้ — ต่อให้ server เงียบไปเป็นวัน ตัวเลขนี้ก็ค้างอยู่ที่เดิม ไม่ลด)
- `library_books_borrowed_total` — จำนวนครั้งที่มีการยืมหนังสือสำเร็จทั้งหมด
- `library_books_returned_total` — จำนวนครั้งที่มีการคืนหนังสือสำเร็จทั้งหมด

สิ่งที่ต้องเข้าใจให้แม่น: **Counter ตัวเดียวโดยตัวมันเองมีประโยชน์จำกัด** — ตัวเลข "มีคนยืมหนังสือไปแล้ว 48,213
ครั้งนับตั้งแต่ deploy ล่าสุด" ไม่ได้บอกอะไรที่น่าสนใจมากนักโดยตัวมันเอง สิ่งที่มีประโยชน์จริง ๆ คือ**อัตราการ
เปลี่ยนแปลง**ของมันเทียบเวลา (เช่น "ตอนนี้มีคนยืมหนังสือ 4.2 ครั้งต่อวินาที") ซึ่งคือหน้าที่ของฟังก์ชัน
**`rate()`** ใน PromQL ที่จะเรียนในหัวข้อ 98.9 — Counter คือ**ข้อมูลดิบ**ที่ต้องผ่าน `rate()` ก่อนถึงจะกลาย
เป็นตัวเลขที่มีความหมายในการเฝ้าดู

#### Gauge: ตัวเลขที่ขึ้น**และ**ลงได้ตามสถานะปัจจุบัน

**Gauge** คือตัวเลข ณ ขณะหนึ่ง ที่**ขึ้นก็ได้ ลงก็ได้** ตามสถานะจริงของระบบ ณ เวลานั้น — ต่างจาก Counter ที่มี
ทิศทางเดียวโดยธรรมชาติ ตัวอย่างที่เหมาะกับ Gauge:

- `library_books_available` — จำนวนสำเนาหนังสือที่ว่างอยู่ตอนนี้ (ยืมไปแล้วก็ลด, คืนแล้วก็เพิ่ม — ขึ้นลงได้
  ตามการยืม/คืนจริง)
- จำนวน **active database connection** ในตอนนี้ (connection ใหม่เปิด → เพิ่ม, connection ปิด → ลด)
- จำนวน **item ที่อยู่ใน queue** รอประมวลผลตอนนี้ (Part 84 เรื่อง background job — งานเข้าคิว → เพิ่ม, งาน
  เสร็จ/ถูกดึงออกจากคิว → ลด)
- **memory usage ปัจจุบันของโปรเซส** (ขึ้นลงตามการใช้งานจริง ไม่มีทิศทางตายตัว)

จุดที่มักสับสน: **"จำนวนหนังสือที่ถูกยืมทั้งหมด" เป็น Counter (เพิ่มอย่างเดียว) แต่ "จำนวนหนังสือที่ว่างอยู่ตอน
นี้" เป็น Gauge (ขึ้นลงได้)** — แม้ทั้งคู่มาจากเหตุการณ์เดียวกัน (การยืม/คืนหนังสือ) แต่คำถามที่แต่ละตัวตอบต่าง
กันคนละเรื่อง: Counter ตอบ "เกิดเหตุการณ์นี้ไปกี่ครั้งแล้วสะสม" ส่วน Gauge ตอบ "สถานะตอนนี้เป็นยังไง"

#### Histogram: การกระจายตัว (distribution) ของค่าที่วัดได้ พร้อม bucket

**Histogram** ใช้เมื่อสิ่งที่อยากรู้ไม่ใช่แค่ "ค่าเฉลี่ย" หรือ "ค่าล่าสุด" แต่คือ**การกระจายตัว**ของค่าที่วัดได้
หลาย ๆ ครั้ง — ตัวอย่างคลาสสิกที่สุดคือ **latency ของ HTTP request**: ค่าเฉลี่ย (average) ของ latency
สามารถ**หลอกตาได้ง่ายมาก** — สมมติมี 100 request ที่ 99 ตัวใช้เวลา 10ms และ 1 ตัวใช้เวลา 10 วินาที ค่าเฉลี่ยจะ
อยู่ที่ประมาณ 109ms ซึ่งดูเหมือน "ปกติดี" ทั้งที่จริง ๆ มี request หนึ่งตัวที่ผู้ใช้คนนั้นเจอปัญหาหนักมาก —
Histogram แก้ปัญหานี้ด้วยการเก็บ**จำนวน request ที่ตกอยู่ในแต่ละช่วง (bucket)** ของค่า เช่น "มีกี่ request ที่
เร็วกว่า 10ms, กี่ request ที่เร็วกว่า 50ms, กี่ request ที่เร็วกว่า 100ms, ..." ทำให้คำนวณ**quantile** (เช่น
p50, p95, p99 — "90% ของ request เร็วกว่ากี่ ms") ได้ทีหลังจากข้อมูล bucket เหล่านี้ ผ่านฟังก์ชัน
`histogram_quantile()` (หัวข้อ 98.9)

ในทางเทคนิค เมื่อคุณสร้าง histogram หนึ่งตัวชื่อ `http_request_duration_seconds` ระบบจะสร้าง **time series
หลายตัวพร้อมกัน**:

- `http_request_duration_seconds_bucket{le="0.01"}` — จำนวน request ที่ใช้เวลา **น้อยกว่าหรือเท่ากับ** 0.01
  วินาที (สะสม, `le` ย่อจาก "less than or equal")
- `http_request_duration_seconds_bucket{le="0.05"}`, `{le="0.1"}`, ... — bucket boundary อื่น ๆ ที่คุณกำหนด
- `http_request_duration_seconds_bucket{le="+Inf"}` — จำนวน request ทั้งหมด (ไม่มี upper bound)
- `http_request_duration_seconds_sum` — **ผลรวม**ของค่าที่วัดได้ทั้งหมด (ใช้คำนวณค่าเฉลี่ยได้ถ้าอยากรู้)
- `http_request_duration_seconds_count` — **จำนวน** observation ทั้งหมด (เท่ากับ `_bucket{le="+Inf"}`)

หัวข้อ 98.4-98.9 จะพิสูจน์ทุกอย่างนี้ด้วยการรันจริง ไม่ใช่แค่พูดลอย ๆ

#### Summary: คล้าย Histogram แต่คำนวณ Quantile ฝั่ง Client แทน

**Summary** พยายามตอบคำถามเดียวกับ Histogram (การกระจายตัว/quantile ของ latency) แต่ด้วยวิธีที่**ต่างกันโดย
พื้นฐาน** — Summary คำนวณ quantile **ที่ฝั่งแอปพลิเคชันเอง** (client-side) ก่อนส่งออกไปเป็นตัวเลขสำเร็จรูป
(เช่น "p50 = 0.05, p95 = 0.3, p99 = 1.2") ในขณะที่ Histogram ส่งออกไปแค่**ข้อมูล bucket ดิบ** แล้วให้
Prometheus server เป็นคนคำนวณ quantile เอาเองทีหลังผ่าน `histogram_quantile()`

| ประเด็น | Histogram | Summary |
|---|---|---|
| ใครคำนวณ quantile | **Prometheus server** ตอน query (`histogram_quantile()`) | **แอปพลิเคชันเอง** ตอน export |
| รวมข้อมูลจากหลาย instance ได้ไหม (เช่น p95 รวมทุก pod) | **ได้** — เพราะข้อมูล bucket รวมกันได้ (บวก `_bucket`/`_count` ของแต่ละ instance เข้าด้วยกันก่อนคำนวณ quantile) | **ไม่ได้ถูกต้อง** — quantile ที่คำนวณมาแล้วของแต่ละ instance เอามาเฉลี่ย/รวมกันตรง ๆ ไม่ได้ทางคณิตศาสตร์ (p95 ของ instance A รวมกับ p95 ของ instance B ไม่เท่ากับ p95 ของข้อมูลทั้งสองรวมกัน) |
| ต้องกำหนด bucket boundary ล่วงหน้าไหม | ต้อง (และเลือกให้เหมาะกับข้อมูลจริง ไม่งั้น quantile จะไม่แม่น — พิสูจน์จริงในหัวข้อ 98.9) | ไม่ต้อง (เลือก quantile ที่อยากได้ เช่น 0.5, 0.95, 0.99 แทน) |
| ความแม่นยำของ quantile ที่ export ออกมา | ประมาณ (interpolate ภายใน bucket) | แม่นกว่าในทางทฤษฎี (คำนวณจากข้อมูลจริงทั้งหมด ณ ขณะนั้น) แต่ใช้รวมข้าม instance ไม่ได้ |

**สรุปเชิงปฏิบัติที่ใช้ตัดสินใจได้จริง**: ระบบ web service ที่รันหลาย instance พร้อมกัน (ซึ่งคือ**เกือบทุกระบบ
production จริง** — มี load balancer กระจาย traffic ไปหลาย pod/instance) **ควรใช้ Histogram เป็นค่าเริ่มต้น
เสมอ** เพราะคุณจะต้องการ p95 latency ที่**รวมทุก instance เข้าด้วยกัน** แทบทุกครั้งที่ query จริง — Summary
เหมาะกับกรณีที่รันแค่ instance เดียวเท่านั้น หรือกรณีที่คุณต้องการความแม่นยำของ quantile สูงกว่าที่ bucket
boundary จะให้ได้ และยอมรับว่าจะดู quantile ได้แค่ต่อ instance เดียวเท่านั้น จริง ๆ แล้วเวอร์ชันใหม่ ๆ ของ
Prometheus ecosystem แนะนำให้ใช้ Histogram เป็นค่า default แทบทุกกรณีเช่นกัน — บทนี้จะใช้ Histogram สำหรับ
latency ตลอดทั้งบท และหัวข้อ 98.9 จะพิสูจน์ให้เห็นจริงว่าทำไมการเผลอปล่อยให้กลายเป็น Summary (ถ้าลืมตั้ง
bucket) ถึงเป็นกับดักที่ต้องระวัง

#### กรอบการตัดสินใจ: เลือก Metric Type ให้ถูกโดยไม่ต้องเดา

ก่อนเขียน metric ใหม่ทุกครั้ง ให้ไล่คำถามสามข้อนี้ตามลำดับ — คำตอบจะนำไปสู่ประเภทที่ถูกต้องเสมอโดยไม่ต้องท่อง
นิยาม:

1. **"ค่านี้มีทางลดลงได้ไหม ในความเป็นจริงของโดเมนนี้?"**
   - **ไม่มีทาง** (มีแต่เพิ่มขึ้นตลอดชีวิตของโปรเซส) → **Counter** เช่น "จำนวนครั้งที่ยืมหนังสือสำเร็จสะสม" —
     ต่อให้คืนหนังสือไปแล้วเมื่อไหร่ ตัวเลข "เคยยืมสำเร็จไปกี่ครั้ง" ก็ไม่มีทางลดกลับ
   - **มีทาง** (ขึ้นได้ลงได้ตามสถานะจริง) → ไปคำถามที่ 2
2. **"สิ่งที่อยากรู้คือค่า ณ ขณะนี้ (สถานะปัจจุบัน) หรือคือการกระจายตัวของหลาย ๆ ค่าที่วัดมา?"**
   - **ค่า ณ ขณะนี้ตัวเดียว** (เช่น "ตอนนี้เหลือสำเนาว่างกี่เล่ม", "ตอนนี้มี connection เปิดอยู่กี่ตัว") →
     **Gauge**
   - **การกระจายตัวของหลายค่าที่วัดซ้ำ ๆ** (เช่น "latency ของ request แต่ละตัวที่ผ่านมา", "ขนาด payload ของ
     แต่ละ request") → ไปคำถามที่ 3
3. **"ระบบนี้รันหลาย instance พร้อมกันไหม และต้องการรวม quantile ข้าม instance หรือไม่?"**
   - **ใช่ (รันหลาย instance และต้องการ p95 รวมทุกตัว)** → **Histogram** (ค่าเริ่มต้นที่ควรใช้เกือบทุกกรณีตาม
     ที่อธิบายไว้ข้างบน)
   - **ไม่ (รันแค่ instance เดียว หรือไม่สนใจรวมข้าม instance เลย และต้องการความแม่นยำสูงสุดของ quantile)** →
     **Summary** เป็นตัวเลือกที่ยอมรับได้ แต่ถ้าไม่แน่ใจให้เลือก Histogram ไว้ก่อนเสมอ (ย้อนกลับไปดูใน dashboard
     ทีหลังง่ายกว่าการเปลี่ยนจาก Summary เป็น Histogram ทีหลัง ซึ่งทำให้ query ที่เขียนไว้ก่อนหน้าใช้ไม่ได้ทันที)

**ตัวอย่างที่มักเลือกผิดในทางปฏิบัติ**: มือใหม่มักเลือก **Gauge** สำหรับ "จำนวน request ทั้งหมดที่ได้รับ" เพราะ
คิดว่า "เดี๋ยวก็ set ค่าใหม่ทุกครั้งที่มี request เข้ามา" — ปัญหาคือ Gauge ไม่ได้ถูกออกแบบมาให้ query ด้วย
`rate()` อย่างมีความหมาย (Prometheus ยอมให้เขียน `rate(some_gauge[1m])` ได้ทางเทคนิคก็จริง แต่ผลลัพธ์ไม่มี
ความหมายที่ถูกต้องเพราะ Gauge ไม่มีสมบัติ "สะสมไม่ลด" ที่ `rate()` ต้องพึ่งพา) — "จำนวน request ทั้งหมด" ควรเป็น
**Counter** เสมอ (เพิ่มทีละ 1 ทุกครั้งที่มี request) แล้วค่อยผ่าน `rate()` ตอน query ตามที่หัวข้อ 98.9 สอน

### 98.3 โมเดล Pull-Based ของ Prometheus: ทำไม Server ต้องมา "ดูด" ข้อมูลเอง

ความแตกต่างเชิงสถาปัตยกรรมที่สำคัญที่สุดของ Prometheus เทียบกับระบบ metrics แบบเดิม (เช่น StatsD ที่นิยมมา
ก่อนหน้า) คือ**ทิศทางการไหลของข้อมูล**:

- **Push-based (เช่น StatsD)**: แอปพลิเคชันของคุณเป็นฝ่าย**ส่ง** metric ออกไปยัง collector server เอง
  (เช่น ยิง UDP packet ไปที่ StatsD daemon ทุกครั้งที่มี event เกิดขึ้น) — collector server แค่**นั่งรอรับ**
  ข้อมูลที่แอปส่งมา
- **Pull-based (Prometheus)**: แอปพลิเคชันของคุณแค่เปิด HTTP endpoint (มาตรฐานคือ `/metrics`) ที่**บอกสถานะ
  ปัจจุบันของ metric ทั้งหมด**เมื่อมีคนมาเรียกดู — Prometheus server เป็นฝ่าย**ไปดึง (scrape)** ข้อมูลจาก
  endpoint นี้เองเป็นระยะ ๆ ตามช่วงเวลาที่ตั้งไว้ (ค่ามาตรฐานคือทุก 15 วินาที)

**เหตุผลเชิงลึกที่ Prometheus เลือกโมเดล pull**:

1. **แอปพลิเคชันฝั่งเรารู้จักแค่ "สถานะตัวเอง ณ ขณะนี้" ไม่ต้องรู้จัก collector server เลย** — โค้ดแอปแค่
   maintain ตัวแปร in-memory (counter/gauge/histogram) แล้วมี handler เดียวที่ render มันออกมาเป็น text ตอนมี
   คนขอดู ไม่ต้องมี logic เรื่อง "ส่งไปที่ไหน, retry ยังไงถ้าส่งไม่สำเร็จ, ต้อง buffer ข้อมูลไว้รอส่งไหม" เลย
   ซึ่งเป็นความซับซ้อนที่ระบบ push-based ต้องจัดการเองทั้งหมด (ถ้า collector server ล่มชั่วคราว metric ที่ push
   ไปตอนนั้นจะหายไปเลย เว้นแต่เขียน retry/buffer เพิ่ม)
2. **เชื่อมกับ service discovery ได้เป็นธรรมชาติมาก** — Prometheus server สามารถถาม infrastructure
   (Kubernetes API, Consul, ฯลฯ) ว่า "ตอนนี้มี instance ของ service ไหนรันอยู่ที่ไหนบ้าง" แล้วไป scrape ทุก
   instance เองอัตโนมัติ โดยที่**แอปพลิเคชันแต่ละตัวไม่ต้องรู้เลยว่า Prometheus server อยู่ที่ไหน** — ตรงข้ามกับ
   push-based ที่แอปทุกตัวต้อง config ไว้ล่วงหน้าว่า collector server อยู่ที่ address ไหน (ถ้า collector
   ย้ายที่ ต้อง config แอปทุกตัวใหม่)
3. **ตรวจ "แอปยังทำงานอยู่ไหม" ได้ในตัว** — ถ้า Prometheus scrape endpoint แล้วเชื่อมต่อไม่ได้ (connection
   refused, timeout) มันจะรู้ทันทีว่า instance นั้นตายหรือ network มีปัญหา (metric `up` ที่จะเห็นจริงในหัวข้อ
   98.8 คือค่านี้เป๊ะ ๆ) — ระบบ push-based ตรวจแบบนี้ไม่ได้ตรง ๆ เพราะแอปที่ตายไปแล้วก็แค่**เงียบไปเฉย ๆ** ไม่
   push อะไรมา ซึ่งแยกไม่ออกจาก "แอปทำงานปกติแต่บังเอิญไม่มี event เกิดขึ้นเลยในช่วงนั้น"

**ข้อจำกัดของ pull-based ที่ควรรู้ในระดับ awareness** (StatsD และ push-based ยังมีที่ใช้จริงอยู่ในบางกรณี):
งาน batch/short-lived (เช่น cron job ที่รันเสร็จแล้วปิดตัวไปเลยภายในไม่กี่วินาที) **ไม่มีช่วงเวลาให้ Prometheus
มาดึงข้อมูลทัน** เพราะโปรเซสตายไปก่อนที่ scrape cycle รอบต่อไปจะมาถึง — Prometheus แก้ปัญหานี้ด้วย component
เสริมชื่อ **Pushgateway** (job สั้น ๆ push ผลลัพธ์สุดท้ายไปที่ Pushgateway ก่อนตาย แล้ว Prometheus ไป scrape
Pushgateway ตามปกติ) ซึ่งเป็นกรณีพิเศษที่ยืม pattern ของ push-based มาแก้ข้อจำกัดของ pull-based เฉพาะจุด — บท
นี้จะไม่ลงรายละเอียด Pushgateway เพราะ web service ที่รันตลอดเวลา (long-running, ซึ่งคือ 99% ของสิ่งที่หลักสูตร
นี้สอน) ใช้โมเดล pull ตรง ๆ ได้เต็มรูปแบบอยู่แล้วโดยไม่ต้องพึ่ง Pushgateway เลย

**Exporter: metric ของสิ่งที่ไม่ใช่โค้ดของคุณเอง (ระดับ awareness)** — บทนี้ทั้งบทสอนการเปิด `/metrics`
endpoint ใน**แอปพลิเคชันของคุณเอง** (ผ่าน `metrics-exporter-prometheus` ที่ห่อ business logic ของคุณตรง ๆ)
แต่ในระบบจริงยังมีความต้องการวัด metric ของ**สิ่งที่ไม่ได้เขียนโค้ดเอง** เช่น "CPU/memory ของเครื่อง server
ทั้งเครื่องใช้ไปเท่าไหร่" (ไม่ใช่แค่ของโปรเซสแอปตัวเดียว) หรือ "container ตัวไหนใน Docker กิน memory มากผิด
ปกติ" — Prometheus ecosystem มี **exporter สำเร็จรูป**ที่เขียนไว้แล้วสำหรับงานลักษณะนี้โดยเฉพาะ ไม่ต้องเขียนโค้ด
เอง: **`node_exporter`** เปิด `/metrics` ที่รายงานสถานะของทั้งเครื่อง (CPU, memory, disk, network) และ
**`cAdvisor`** ทำแบบเดียวกันแต่เจาะจงระดับ container — ทั้งสองตัวทำงานตามโมเดล pull-based เดียวกันเป๊ะกับที่
บทนี้สอน (Prometheus scrape `/metrics` ของมันเหมือนที่ scrape แอป Axum ของเรา) เพียงแต่**คนอื่นเขียน exporter
ให้แล้ว** คุณแค่รันมันคู่กับแอปแล้วเพิ่ม target ใหม่ใน `prometheus.yml` — บทนี้ไม่ลงรายละเอียดเพราะโฟกัสที่
metric ระดับแอปพลิเคชัน (application-level metrics) ที่คุณเขียนโค้ดเองผ่าน `metrics` crate ตรง ๆ ซึ่งเป็น
เนื้อหาหลักที่สำคัญกว่าสำหรับนักพัฒนา ส่วน metric ระดับ infrastructure (infrastructure-level metrics) เป็น
ความรับผิดชอบของทีม platform/DevOps มากกว่า แต่ทั้งสองแบบ query ผ่าน PromQL เดียวกันและแสดงบน Grafana
dashboard เดียวกันได้ตามที่หัวข้อ 98.10 อธิบายไว้

### 98.4 ติดตั้ง `metrics` + `metrics-exporter-prometheus`: เปิด `/metrics` Endpoint แรก

มาถึงส่วนของโค้ดจริง — สร้างโปรเจกต์ Axum ใหม่ (ต่อยอด Part 62):

```bash
cargo new metrics_demo
cd metrics_demo
cargo add axum tokio --features tokio/full
cargo add metrics metrics-exporter-prometheus
```

ได้ `Cargo.toml` (เวอร์ชันจริงที่ทดสอบผ่านในบทนี้):

```toml
[package]
name = "metrics_demo"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.8.9"
metrics = "0.24.6"
metrics-exporter-prometheus = "0.18.3"
tokio = { version = "1.53.1", features = ["full"] }
```

#### Facade Pattern อีกครั้ง: `metrics` คือ `log` เวอร์ชัน Metrics

จำ analogy จาก **Part 60** ได้ไหม — `log` crate นิยามแค่ macro (`info!`, `warn!`, ...) โดยไม่รู้เลยว่า log จะ
ไปโผล่ที่ไหน แล้วปล่อยให้ backend อย่าง `env_logger` เป็นคนตัดสินใจจริง **`metrics` crate ใช้แนวคิดเดียวกันเป๊ะ
กับตัวเลข**:

- **`metrics` crate** นิยามแค่ macro (`counter!`, `gauge!`, `histogram!`) และ trait `Recorder` ที่อธิบายว่า
  "backend ตัวหนึ่งต้อง implement อะไรถึงจะรับค่า metric ได้" — มันไม่รู้เลยว่าตัวเลขที่บันทึกไปจะไปโผล่ที่ไหน
  (Prometheus? StatsD? console? ไม่ใช่เรื่องของ `metrics` เลย)
- **`metrics-exporter-prometheus`** คือ backend ที่เป็นรูปธรรม — implement `Recorder` จริง แล้วแปลงตัวเลขที่
  สะสมไว้ให้กลายเป็น**ข้อความตามฟอร์แมตที่ Prometheus อ่านเข้าใจ** พร้อมเปิด HTTP handler ให้ scrape ได้

เหมือนกับที่ `log` มีปัญหา "ไม่ init backend = ไม่มีอะไรเกิดขึ้นเลยแบบเงียบ ๆ" `metrics` ก็มีปัญหาเดียวกันเป๊ะ —
พิสูจน์ด้วยโปรแกรมสั้น ๆ ที่**ไม่**ติดตั้ง recorder เลย:

```rust
// ตัวอย่างนี้ "ผิด" โดยตั้งใจ — ไม่ได้เรียก install_recorder() หรือ set_global_recorder() เลย
fn main() {
    metrics::counter!("orphan_counter").increment(5);
    metrics::gauge!("orphan_gauge").set(42.0);
    println!("จบโปรแกรมแล้ว — ไม่มี error/panic อะไรเกิดขึ้นเลย แม้ไม่ได้ตั้ง recorder");
}
```

รันจริงได้ output แค่บรรทัดเดียว (ยืนยันด้วยการรันจริงในสภาพแวดล้อมทดสอบ):

```
จบโปรแกรมแล้ว — ไม่มี error/panic อะไรเกิดขึ้นเลย แม้ไม่ได้ตั้ง recorder
```

เหมือนกับ `log` เป๊ะ — `metrics::counter!(...).increment(5)` ทำงานได้ **ไม่ panic ไม่ error** แต่ค่าที่บันทึกไป
หายไปในอากาศทันที เพราะไม่มี recorder ตัวไหนติดตั้งไว้ให้มันไปเก็บ (ภายใน `metrics` มี no-op recorder เป็น
ค่าเริ่มต้น เหมือนที่ `log` มี no-op logger) — จำไว้ให้แม่น เพราะนี่คือกับดักแรกของหัวข้อกับดักท้ายบท

#### เขียนแอปที่มี `/metrics` Endpoint จริง

```rust
use axum::{extract::State, routing::get, Router};
use metrics_exporter_prometheus::{PrometheusBuilder, PrometheusHandle};

// PrometheusHandle เป็น handle ที่ Clone ได้ถูก ๆ (ข้างในเป็น Arc) — ใช้เป็น AppState ได้ตรง ๆ
async fn metrics_handler(State(handle): State<PrometheusHandle>) -> String {
    // .render() แปลง metric ที่สะสมไว้ทั้งหมดให้เป็น Prometheus text format ตามสเปกจริง
    handle.render()
}

async fn health() -> &'static str {
    "ok"
}

#[tokio::main]
async fn main() {
    // PrometheusBuilder::install_recorder() ทำสองอย่างพร้อมกัน: (1) ติดตั้ง global recorder ให้
    // metrics::counter!/gauge!/histogram! ทั้งโปรแกรมส่งค่ามาที่นี่ (เทียบเท่า env_logger::init() ของ
    // Part 60) และ (2) คืน PrometheusHandle ที่ render metric ปัจจุบันเป็น text ได้ตอนที่เราต้องการ
    let prometheus_handle = PrometheusBuilder::new()
        .install_recorder()
        .expect("ติดตั้ง Prometheus recorder ไม่สำเร็จ");

    let app = Router::new()
        .route("/health", get(health))
        .route("/metrics", get(metrics_handler))
        .with_state(prometheus_handle);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3300").await.unwrap();
    println!("ฟังอยู่ที่ http://127.0.0.1:3300");
    axum::serve(listener, app).await.unwrap();
}
```

**ทางเลือกอื่นที่ควรรู้จัก: `.install()` เปิด HTTP server แยกของตัวเอง โดยไม่ต้องพึ่ง Axum route**
— ตัวอย่างข้างบนใช้ `.install_recorder()` แล้วสร้าง route `/metrics` เข้ากับ `Router` ของ Axum เอง (วิธีที่
บทนี้แนะนำเพราะควบคุมได้เต็มที่ ผสานเข้ากับ middleware/state ของแอปได้ตรงไปตรงมา) แต่
`metrics-exporter-prometheus` ยังมีเมธอด **`.install()`** ที่**สปอว์น HTTP server ของตัวเองแยกไปเลย** บน
address ที่กำหนด — เหมาะกับสถานการณ์ที่แอปของคุณ**ไม่ใช่** web server อยู่แล้ว (เช่น CLI tool ที่ Part 59 สอน,
หรือ background worker ที่ Part 84 สอน ที่ไม่มี `Router`/`axum::serve` ให้แนบ route เข้าไปอยู่แล้วตั้งแต่ต้น):

```rust
use metrics_exporter_prometheus::PrometheusBuilder;

#[tokio::main]
async fn main() {
    // .install() สปอว์น task ของตัวเองที่ฟัง HTTP request บน port 9500 แยกจากส่วนอื่นของโปรแกรมทั้งหมด
    PrometheusBuilder::new()
        .with_http_listener(([127, 0, 0, 1], 9500))
        .install()
        .expect("ติดตั้ง standalone exporter ไม่สำเร็จ");

    metrics::counter!("standalone_demo_total").increment(7);
    // ... ตรรกะหลักของโปรแกรม (ไม่ใช่ web server) ทำงานต่อไปตามปกติ ...
}
```

รันจริงแล้วยิง `curl` ไปที่พอร์ตแยกที่ตั้งไว้ (9500 ไม่ใช่ 3300) ได้ output จริง:

```bash
curl -sS http://127.0.0.1:9500/metrics
```

```
# TYPE standalone_demo_total counter
standalone_demo_total 7
```

**หลักการเลือกระหว่างสองวิธี**: ถ้าแอปของคุณเป็น Axum web server อยู่แล้ว (ซึ่งคือกรณีหลักของบทนี้ทั้งบท)
ใช้ `.install_recorder()` แล้วแนบ `/metrics` เป็นอีก route หนึ่งของ `Router` เดิม (ตามตัวอย่างหลักของบทนี้)
เพราะไม่ต้องเปิด port ที่สองแยก, ใช้ middleware/logging stack เดียวกันได้, และ deploy ง่ายกว่า (มี port
เดียวให้ดูแล) — ใช้ `.install()` เฉพาะเมื่อโปรแกรมของคุณไม่มี HTTP server อยู่แล้วตั้งแต่ต้น

`cargo build` ผ่านสะอาด (เวอร์ชันจริงที่ทดสอบ: axum 0.8.9, metrics 0.24.6, metrics-exporter-prometheus
0.18.3) รันแล้วยิง `curl` จริง:

```bash
curl -sS http://127.0.0.1:3300/metrics
```

ผลลัพธ์จริง (โปรแกรมนี้ยังไม่ได้บันทึก metric อะไรเข้าไปเลยนอกจากที่ exporter สร้างให้อัตโนมัติ):

```
```

**ว่างเปล่าสนิท** — และนี่คือพฤติกรรมที่ถูกต้อง 100% ไม่ใช่ bug: recorder ติดตั้งแล้วก็จริง แต่**ยังไม่มีใคร
เรียก `counter!`/`gauge!`/`histogram!` แม้แต่ครั้งเดียว** ตัวเลขจึงยังไม่มีอะไรให้ render ออกมา ต่างจากตัวอย่าง
`orphan_counter` ก่อนหน้าที่เรียก macro แต่ไม่มี recorder — คราวนี้กลับกัน: **มี recorder แต่ไม่มีใครเรียก
macro** ผลลัพธ์เลยว่างเหมือนกันแต่ด้วยเหตุผลตรงข้ามกัน หัวข้อถัดไปจะเติมส่วนที่ขาดไป

#### ธรรมเนียมการตั้งชื่อ Metric ของ Prometheus และ HELP Text

ก่อนเขียน metric จริงตัวแรก ควรรู้จัก**ธรรมเนียมการตั้งชื่อ (naming convention)** ที่ Prometheus ecosystem
ทั้งหมดยึดถือร่วมกัน (ไม่ใช่กฎที่ compiler บังคับ แต่เป็นธรรมเนียมที่เครื่องมือรอบ ๆ อย่าง Grafana คาดหวังไว้ —
ผิดธรรมเนียมนี้ไม่ทำให้ metric พังทางเทคนิค แต่ทำให้คนอื่นอ่าน dashboard ของคุณแล้วงงหรือเข้าใจผิดหน่วยได้ง่าย):

1. **ชื่อเป็น `snake_case` เสมอ** — `library_books_borrowed_total` ไม่ใช่ `libraryBooksBorrowedTotal`
2. **Counter ควรลงท้ายด้วย `_total`** — บอกผู้อ่านทันทีว่านี่คือค่าสะสมที่มีแต่เพิ่มขึ้น ต้องผ่าน `rate()`/
   `increase()` ก่อนถึงจะมีความหมายเป็น "อัตรา" (ตามที่อธิบายไว้ในหัวข้อ 98.2 และ 98.9) — `metrics-exporter-
   prometheus` ยังเติม `_total` ให้อัตโนมัติด้วยถ้าคุณลืมต่อท้ายเองตอนประกาศผ่าน `counter!()`
3. **หน่วยวัดต้องอยู่ในชื่อ และต้องเป็น base unit เสมอ** — เวลาใช้ `_seconds` (ไม่ใช่ `_ms`/`_milliseconds`),
   ขนาดข้อมูลใช้ `_bytes` (ไม่ใช่ `_kb`/`_mb`), เพราะ PromQL ไม่รู้จัก "หน่วย" ในทางโปรแกรม มันเห็นแค่ตัวเลข
   ดิบ ๆ — ถ้าทีมหนึ่งใช้ `_ms` อีกทีมใช้ `_seconds` แล้วนำมา query รวมกัน (เช่น `sum()` ข้าม service) ตัวเลข
   จะผิดพลาดแบบเงียบ ๆ โดยไม่มี type system ใดมาเตือน (ดูกับดักข้อ 5 ท้ายบทที่พิสูจน์ปัญหานี้ไว้แบบรันจริง)
4. **ชื่อ metric ไม่ควรมีข้อมูลที่ควรเป็น label ผสมอยู่** — เขียน `http_requests_total{method="GET"}` ไม่ใช่
   `http_get_requests_total` — เหตุผลคือ label ทำให้ query ข้าม method ต่าง ๆ รวมกันได้ (`sum without
   (method) (...)`) ในขณะที่ถ้าฝังไว้ในชื่อ metric ต้องเขียน query แยกกันสำหรับแต่ละ method ทุกครั้ง

Prometheus text format ยังรองรับ**บรรทัดอธิบาย (`# HELP`)** ที่ทำให้คนอื่น (หรือตัวคุณเองอีกหกเดือนข้างหน้า)
เข้าใจว่า metric แต่ละตัวหมายถึงอะไรโดยไม่ต้องไปเปิดโค้ดดู — `metrics` crate มี macro
`describe_counter!`/`describe_gauge!`/`describe_histogram!` สำหรับใส่คำอธิบายนี้ (เรียกครั้งเดียวตอน `main()`
เริ่มทำงาน ก่อนหรือหลัง `counter!()`/`gauge!()` ก็ได้ — ลำดับไม่สำคัญเพราะเป็นแค่ metadata แยกจากค่าตัวเลข):

```rust
metrics::describe_counter!(
    "library_books_borrowed_total",
    "จำนวนครั้งทั้งหมดที่มีการยืมหนังสือสำเร็จ นับตั้งแต่โปรเซสเริ่มทำงาน"
);
metrics::describe_gauge!(
    "library_books_available",
    "จำนวนสำเนาหนังสือที่ว่างอยู่ในระบบ ณ ขณะนี้"
);
```

รันจริงแล้ว `/metrics` จะมีบรรทัด `# HELP` นำหน้าทุก metric ที่ถูก describe ไว้ (ยืนยันด้วยการรันจริง):

```
# HELP library_books_borrowed_total จำนวนครั้งทั้งหมดที่มีการยืมหนังสือสำเร็จ นับตั้งแต่โปรเซสเริ่มทำงาน
# TYPE library_books_borrowed_total counter
library_books_borrowed_total 3

# HELP library_books_available จำนวนสำเนาหนังสือที่ว่างอยู่ในระบบ ณ ขณะนี้
# TYPE library_books_available gauge
library_books_available 5
```

ในเครื่องมือที่มี autocomplete สำหรับ PromQL (เช่นหน้า "Graph" ของ Prometheus เอง หรือ Grafana) ข้อความ
`# HELP` นี้จะโผล่ขึ้นมาเป็น tooltip ตอนพิมพ์ชื่อ metric — เป็นการลงทุนเล็ก ๆ (เพิ่มไม่กี่บรรทัดตอน `main()`)
ที่ช่วยคนอื่นในทีมได้มากตอนพยายามทำความเข้าใจ dashboard ที่ไม่ได้เขียนเอง

### 98.5 Middleware วัด HTTP Metrics อัตโนมัติทุก Request (ต่อยอด Part 65)

เป้าหมายของหัวข้อนี้: **handler ทุกตัวในระบบ**ควรมี HTTP request counter กับ latency histogram ให้อัตโนมัติ
โดย**ไม่ต้องเขียนโค้ด metrics ซ้ำในทุก handler** — นี่คือปัญหาแบบเดียวกับที่ Part 65 หัวข้อ 65.1 อธิบายไว้เรื่อง
"cross-cutting concern" (logging, timing) พอดี เพียงแต่คราวนี้สิ่งที่อยากทำซ้ำทุก request คือการบันทึก metric
แทน log

#### เลือก `.route_layer()` ไม่ใช่ `.layer()`: เหตุผลที่พิสูจน์ได้จริง

ก่อนเขียน middleware ต้องตัดสินใจก่อนว่าจะแนบมันเข้า `Router` ด้วยเมธอดไหน — **Part 65 หัวข้อ 65.5-65.6**
บอกไว้แล้วว่า `.layer()` กับ `.route_layer()` มีพฤติกรรมต่างกันเรื่อง "เห็น request ที่ไม่ match route ไหนเลย
(404) หรือไม่" — บทนี้พิสูจน์ผลที่ตามมาของความแตกต่างนี้กับ metrics โดยตรงด้วยการรันจริง เขียน middleware
probe สองตัวที่แค่ print ว่าเห็น `MatchedPath` extension หรือไม่ (extension ที่ Axum เติมให้ request ที่ match
route สำเร็จ บอก route template เช่น `/books/{id}`):

```rust
use axum::{
    extract::{MatchedPath, Path},
    http::Request,
    middleware::{self, Next},
    response::Response,
    routing::get,
    Router,
};

async fn get_book(Path(_id): Path<u32>) -> &'static str {
    "book"
}

async fn probe_with_layer(req: Request<axum::body::Body>, next: Next) -> Response {
    let mp = req.extensions().get::<MatchedPath>().map(|m| m.as_str().to_owned());
    println!("[.layer()]       MatchedPath = {mp:?} (uri.path()={})", req.uri().path());
    next.run(req).await
}

async fn probe_with_route_layer(req: Request<axum::body::Body>, next: Next) -> Response {
    let mp = req.extensions().get::<MatchedPath>().map(|m| m.as_str().to_owned());
    println!("[.route_layer()] MatchedPath = {mp:?} (uri.path()={})", req.uri().path());
    next.run(req).await
}

#[tokio::main]
async fn main() {
    let app_layer = Router::new()
        .route("/books/{id}", get(get_book))
        .layer(middleware::from_fn(probe_with_layer));

    let app_route_layer = Router::new()
        .route("/books/{id}", get(get_book))
        .route_layer(middleware::from_fn(probe_with_route_layer));

    let l1 = tokio::net::TcpListener::bind("127.0.0.1:4401").await.unwrap();
    let l2 = tokio::net::TcpListener::bind("127.0.0.1:4402").await.unwrap();
    tokio::spawn(async move { axum::serve(l1, app_layer).await.unwrap() });
    tokio::spawn(async move { axum::serve(l2, app_route_layer).await.unwrap() });

    println!("READY");
    tokio::time::sleep(std::time::Duration::from_secs(10)).await;
}
```

ยิง request สองแบบไปที่ทั้งสอง server — ครั้งแรก path ที่**match** route (`/books/42`) ครั้งที่สอง path ที่
**ไม่ match** เลย (`/does-not-exist`):

```bash
curl -sS http://127.0.0.1:4401/books/42 > /dev/null   # .layer()
curl -sS http://127.0.0.1:4402/books/42 > /dev/null   # .route_layer()
curl -sS http://127.0.0.1:4401/does-not-exist > /dev/null   # .layer()
curl -sS http://127.0.0.1:4402/does-not-exist > /dev/null   # .route_layer()
```

ผลลัพธ์จริงที่ terminal ของ server (คัดลอกตรงจากการรันจริง — สังเกตให้ดีว่ามีทั้งหมด**สามบรรทัด** ไม่ใช่สี่):

```
READY
[.layer()]       MatchedPath = Some("/books/{id}") (uri.path()=/books/42)
[.route_layer()] MatchedPath = Some("/books/{id}") (uri.path()=/books/42)
[.layer()]       MatchedPath = None (uri.path()=/does-not-exist)
```

สามข้อเท็จจริงที่พิสูจน์ได้จากผลลัพธ์นี้:

1. **สำหรับ request ที่ match route สำเร็จ ทั้ง `.layer()` และ `.route_layer()` เห็น `MatchedPath` เหมือนกัน**
   ("`Some("/books/{id}")`" ทั้งคู่) — ตรงข้ามกับที่มักเข้าใจผิดว่า `.route_layer()` เท่านั้นที่เห็น
   `MatchedPath` ได้ ความจริงคือทั้งคู่เห็นได้เหมือนกันสำหรับ request ที่ match
2. **`.layer()` ยังคงถูกเรียกสำหรับ request ที่ไม่ match route ไหนเลย (404)** — และตอนนั้น `MatchedPath` คือ
   `None` (ไม่มี route template ให้ เพราะไม่มี route ไหน match เลย)
3. **`.route_layer()` ไม่ถูกเรียกเลยสำหรับ request ที่ไม่ match** — สังเกตว่ามีแค่**สาม**บรรทัด print ออกมา
   ไม่ใช่สี่ — บรรทัดที่ควรจะเป็น `[.route_layer()] ... /does-not-exist` **ไม่ปรากฏเลย** เพราะ
   `.route_layer()` ถูก apply เข้ากับแต่ละ route โดยตรง (เป็นส่วนหนึ่งของ per-route service chain) ไม่ใช่ห่อ
   ทั้ง `Router` เหมือน `.layer()` — request ที่ไม่ match route ไหนไม่มีทาง "ผ่าน" middleware นี้ไปได้เลย
   เพราะมันไม่ได้ผูกอยู่กับ route ไหนที่ match

**ผลกระทบต่อ metrics โดยตรง**: ถ้าใช้ `.layer()` แล้วเขียน fallback ตอนที่ `MatchedPath` เป็น `None` ให้ใช้
`req.uri().path()` ดิบ ๆ แทน (ดูเผิน ๆ เหมือนความคิดที่สมเหตุสมผล — "อย่างน้อยก็ยังอยากรู้ว่ามี 404 เกิดขึ้นไหม")
**ทุก path แปลก ๆ ที่ยิงมาแบบสุ่ม** (bot scanner ที่ลองยิง `/wp-admin`, `/.env`, `/books/1`, `/books/2`, ...,
`/books/999999` เพื่อหาช่องโหว่ — ซึ่งเกิดขึ้นจริงกับทุก public endpoint บนอินเทอร์เน็ต) **จะกลายเป็น time
series ใหม่ทุกครั้งไม่จำกัด** เพราะ path ที่ไม่ match ไม่มี "template" ให้ยึด — นี่คือเหตุผลที่บทนี้เลือกใช้
`.route_layer()` เป็นค่าเริ่มต้นสำหรับ metrics middleware: มันตัดปัญหานี้ทิ้งไปเลยตั้งแต่ต้นโดยไม่ต้องเขียน
guard เพิ่ม (ข้อแลกเปลี่ยนคือคุณจะไม่เห็น metric ของ 404 เลยจาก middleware ตัวนี้ — ถ้าต้องการนับ 404 จริง ๆ
ควรทำผ่าน `Router::fallback()` แยกเป็นอีก metric ต่างหากที่ label คงที่ เช่น `path="unmatched"` ไม่ใช่ raw
path — รายละเอียดเต็มอยู่ในหัวข้อ 98.7)

#### เขียน Middleware วัด Metrics จริง

```rust
use axum::{
    extract::MatchedPath,
    http::Request,
    middleware::Next,
    response::Response,
};
use std::time::Instant;

/// Middleware ที่บันทึก HTTP request counter + duration histogram ให้ทุก route ที่ผ่านการ match แล้ว
/// ต้องแนบด้วย `.route_layer()` (ไม่ใช่ `.layer()`) ตามเหตุผลที่พิสูจน์ไว้ข้างบน
pub async fn track_http_metrics(req: Request<axum::body::Body>, next: Next) -> Response {
    let start = Instant::now();

    // ใช้ MatchedPath (route template เช่น "/books/{id}/borrow") ไม่ใช่ req.uri().path() ดิบ ๆ
    // (เช่น "/books/42/borrow") — หัวใจของการเลี่ยงกับดัก high-cardinality label ในหัวข้อ 98.7
    let path = req
        .extensions()
        .get::<MatchedPath>()
        .map(|mp| mp.as_str().to_owned())
        .unwrap_or_else(|| req.uri().path().to_owned());
    let method = req.method().to_string();

    let response = next.run(req).await;

    let status = response.status().as_u16().to_string();
    let elapsed = start.elapsed().as_secs_f64(); // Prometheus convention: หน่วยเป็นวินาทีเสมอ ไม่ใช่ ms

    let labels = [("method", method), ("path", path), ("status", status)];

    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_request_duration_seconds", &labels).record(elapsed);

    response
}
```

ประกอบเข้ากับ router (โดเมนห้องสมุดง่าย ๆ ที่จะขยายเป็น business metric ในหัวข้อถัดไป):

```rust
use axum::{routing::{get, post}, Router};

let instrumented_routes = Router::new()
    .route("/health", get(health))
    .route("/books", get(list_books))
    .route("/books/{id}/borrow", post(borrow_book))
    .route("/books/{id}/return", post(return_book))
    .route_layer(axum::middleware::from_fn(track_http_metrics));

let app = Router::new()
    .merge(instrumented_routes)
    .route("/metrics", get(metrics_handler)) // ไม่ต้องผ่าน metrics middleware ของตัวมันเอง
    .with_state(state);
```

สังเกตว่า **`/metrics` ไม่ได้อยู่ใน `instrumented_routes`** — ตั้งใจแยกไว้ไม่ให้ middleware วัด metrics ของ
ตัวเอง (การเรียก `/metrics` เองไม่ควรถูกนับเป็น "business traffic" ปนกับ endpoint จริงของแอป แม้จะทำได้ก็ตาม
ไม่มีประโยชน์อะไรเป็นพิเศษ)

**พิสูจน์ด้วยการยิง request จริงหลายแบบ** แล้วดู `/metrics`:

```bash
curl -sS http://127.0.0.1:3300/health
curl -sS http://127.0.0.1:3300/books
curl -sS -X POST http://127.0.0.1:3300/books/1/borrow
curl -sS -X POST http://127.0.0.1:3300/books/1/borrow
curl -sS -X POST http://127.0.0.1:3300/books/1/borrow   # ครั้งที่สาม: หมดสต็อกแล้ว -> 409
curl -sS -X POST http://127.0.0.1:3300/books/999/borrow  # ไม่มีหนังสือ id นี้ -> 404
curl -sS -X POST http://127.0.0.1:3300/books/1/return
```

ผลลัพธ์จริงจาก `curl -sS http://127.0.0.1:3300/metrics` (ตัดเฉพาะส่วน `http_requests_total`):

```
# TYPE http_requests_total counter
http_requests_total{method="POST",path="/books/{id}/return",status="200"} 1
http_requests_total{method="POST",path="/books/{id}/borrow",status="409"} 1
http_requests_total{method="POST",path="/books/{id}/borrow",status="200"} 2
http_requests_total{method="POST",path="/books/{id}/borrow",status="404"} 1
http_requests_total{method="GET",path="/books",status="200"} 1
http_requests_total{method="GET",path="/health",status="200"} 1
```

**ทุกตัวเลขตรงกับที่ยิง request ไปเป๊ะ**: `borrow` สำเร็จ (`status="200"`) 2 ครั้ง (ครั้งที่ 1 และ 2), หมด
สต็อก (`status="409"`) 1 ครั้ง (ครั้งที่ 3), หนังสือไม่มีจริง (`status="404"`) 1 ครั้ง, `return` สำเร็จ 1 ครั้ง
— และที่สำคัญมาก: **path ทั้งสี่ borrow request รวมกันเป็นแค่สามบรรทัด** (`status="200"` มีค่า 2 ไม่ใช่สี่
บรรทัดแยกกันตาม path จริงที่ยิงไป เช่น `/books/1/borrow`) เพราะ label `path` ใช้ **route template**
`/books/{id}/borrow` เดียวกันสำหรับทุก book id — นี่คือผลลัพธ์ที่ถูกต้องของการใช้ `MatchedPath` ที่จะอธิบาย
ความสำคัญเต็มรูปแบบในหัวข้อ 98.7

ลองยิง path ที่ไม่มีจริงดูด้วย เพื่อพิสูจน์ว่า `.route_layer()` ทำงานตามที่คาดจริง:

```bash
curl -sS http://127.0.0.1:3300/does-not-exist
curl -sS http://127.0.0.1:3300/metrics | grep does-not-exist
```

ผลลัพธ์จริง: **`grep` ไม่เจออะไรเลย** — request ไปที่ path ที่ไม่มีจริงไม่ได้เพิ่ม time series ใหม่เข้ามาใน
`/metrics` แม้แต่ตัวเดียว ตรงตามที่พิสูจน์ไว้ในหัวข้อก่อนหน้าว่า `.route_layer()` ไม่ถูกเรียกสำหรับ request ที่
ไม่ match route

### 98.6 Custom Business Metrics: ยืม/คืนหนังสือ (ต่อยอด Module 4-5 และ Part 92)

Metrics ระดับ HTTP (request count, latency) ตอบคำถาม**เชิงเทคนิค**ได้ดี ("API ช้าไหม, error rate สูงไหม")
แต่ไม่ตอบคำถาม**เชิงธุรกิจ**เลย — "วันนี้มีคนยืมหนังสือกี่เล่ม", "ตอนนี้เหลือสำเนาว่างกี่เล่มในระบบ" คือคำถามที่
ทีมธุรกิจ/ห้องสมุดสนใจโดยตรง ไม่เกี่ยวกับ HTTP status code เลย — **custom business metric** คือคำตอบของ
คำถามกลุ่มนี้ และมันคือจุดที่ metrics มีค่ามากที่สุดในทางปฏิบัติ เพราะ dashboard ที่มีแค่ "request/sec" ไม่ได้
บอกอะไรเกี่ยวกับสุขภาพของธุรกิจจริง ๆ เลย

เพิ่มสองบรรทัดเข้าไปใน handler `borrow_book`/`return_book` ที่ Part 92 เขียนไว้แล้วด้วย SQLx — จุดที่เพิ่ม
ต้องอยู่**หลัง `tx.commit().await?` สำเร็จเท่านั้น** (ไม่ใช่ก่อน) เพราะถ้า transaction rollback (early return ที่
Part 92 อธิบายไว้ว่าปลอดภัยด้วย `Drop` ของ `Transaction`) การยืม/คืนนั้นไม่ได้เกิดขึ้นจริงในฐานข้อมูล — metric
ก็ไม่ควรนับว่าเกิดขึ้นตามไปด้วย:

```rust
// src/routes/borrows.rs — ส่วนที่ต่อจาก Part 92 (โค้ดเดิมของ Part 92 คงไว้ทั้งหมด เพิ่มแค่สองบรรทัด metrics)
pub async fn borrow_book(
    State(state): State<AppState>,
    current_user: CurrentUser,
    Path(id): Path<i64>,
) -> Result<(StatusCode, Json<BorrowResponse>), AppError> {
    let mut tx = state.db.begin().await?;

    // ... (เช็คหนังสือมีจริงไหม, เช็คยืมค้างอยู่ไหม, atomic decrement available_copies — เหมือน Part 92 เดิมทุกจุด) ...

    let borrow = sqlx::query_as!(/* ... เหมือน Part 92 เดิม ... */)
        .fetch_one(&mut *tx)
        .await?;

    tx.commit().await?; // <- ต้องอยู่ก่อนเส้นด้านล่างนี้เสมอ: metric ต้องนับ "เหตุการณ์ที่เกิดขึ้นจริง" เท่านั้น

    // *** สองบรรทัดใหม่ที่ Part 98 เพิ่มเข้ามา ***
    metrics::counter!("library_books_borrowed_total").increment(1);
    metrics::gauge!("library_books_available").decrement(1.0);

    tracing::info!(book_id = id, user_id = current_user.user_id, "ยืมหนังสือสำเร็จ");
    Ok((StatusCode::CREATED, Json(borrow)))
}
```

และ `return_book` แบบเดียวกัน (เพิ่มหลัง `tx.commit().await?`):

```rust
pub async fn return_book(/* พารามิเตอร์เดิมจาก Part 92 */) -> Result<Json<BorrowResponse>, AppError> {
    // ... (เช็คหนังสือมีจริงไหม, เช็คมีคนยืมค้างอยู่จริงไหม, เช็คว่าเป็นคนยืมเองไหม — เหมือน Part 92 เดิม) ...

    tx.commit().await?;

    metrics::counter!("library_books_returned_total").increment(1);
    metrics::gauge!("library_books_available").increment(1.0);

    tracing::info!(book_id = id, user_id = current_user.user_id, "คืนหนังสือสำเร็จ");
    Ok(Json(borrow))
}
```

**สังเกตการเลือกใช้ Gauge สำหรับ `library_books_available`**: เพิ่มด้วย `.increment(1.0)` ตอนคืน ลดด้วย
`.decrement(1.0)` ตอนยืม — Gauge เก็บ**ค่าที่เปลี่ยนแปลงสัมพัทธ์กับค่าปัจจุบัน**ได้โดยตรง ไม่ต้อง query
database มาคำนวณใหม่ทุกครั้งที่มีคน scrape `/metrics` (ซึ่งจะทำให้ endpoint `/metrics` เองมี side-effect ไป
กระทบ database — ผิดหลักการที่ metric ควรเป็น **passive** ไม่กระทบระบบที่มันสังเกตอยู่)

**พิสูจน์ด้วยการรันจริง** — เพื่อความกระชับ ตัวอย่างที่ทดสอบในบทนี้ใช้ระบบห้องสมุดจำลองแบบ in-memory (ไม่ต้อง
พึ่ง Postgres จริงเพื่อลดความซับซ้อนในการสาธิต) แต่**โค้ด instrumentation ที่เพิ่มเข้าไปสองบรรทัดข้างบนเหมือน
กันเป๊ะ**กับที่จะใส่ใน handler จริงของ Part 92 ที่ใช้ SQLx — catalog เริ่มต้นมี 3 เล่ม (เล่ม A มี 2 สำเนา, เล่ม
B มี 3 สำเนา, เล่ม C มี 1 สำเนา = รวม 6 สำเนา):

```bash
curl -sS -X POST http://127.0.0.1:3300/books/1/borrow > /dev/null
curl -sS -X POST http://127.0.0.1:3300/books/1/borrow > /dev/null
curl -sS -X POST http://127.0.0.1:3300/books/2/borrow > /dev/null
curl -sS -X POST http://127.0.0.1:3300/books/1/return > /dev/null
curl -sS http://127.0.0.1:3300/metrics | grep -E "^library_books"
```

ผลลัพธ์จริง:

```
# TYPE library_books_returned_total counter
library_books_returned_total 1

# TYPE library_books_borrowed_total counter
library_books_borrowed_total 3

# TYPE library_books_available gauge
library_books_available 4
```

ตรวจสอบตัวเลข: เริ่มจาก 6 สำเนาว่าง → ยืมสำเร็จ 3 ครั้ง (−3) → คืน 1 ครั้ง (+1) = **4 สำเนาว่าง** ตรงกับ
`library_books_available = 4` เป๊ะ และ `library_books_borrowed_total = 3` ตรงกับจำนวนครั้งที่ยืมสำเร็จจริง —
สังเกตว่า **Counter ไม่ลดแม้จะมีการคืนหนังสือเกิดขึ้น** (`library_books_borrowed_total` ไม่ลดกลับเป็น 2 ตอน
คืน) เพราะ "จำนวนครั้งที่เคยยืมสำเร็จสะสม" เป็นข้อเท็จจริงในอดีตที่ไม่เปลี่ยนแปลง ตรงตามคำนิยามของ Counter ใน
หัวข้อ 98.2

### 98.7 Labels/Dimensions และกับดัก High-Cardinality

Label คือสิ่งที่ทำให้ metric หนึ่งชื่อ (เช่น `http_requests_total`) แตกออกเป็น**หลาย time series** ตามค่าของ
label นั้น — `http_requests_total{method="GET", path="/books", status="200"}` กับ
`http_requests_total{method="POST", path="/books/{id}/borrow", status="200"}` คือ **time series คนละตัว
กัน** แม้จะมีชื่อ metric เดียวกัน (`http_requests_total`) เพราะค่า label ต่างกัน — จำนวน time series ทั้งหมด
ของ metric หนึ่งชื่อ = **จำนวนค่าที่เป็นไปได้ของ label แต่ละตัว คูณกัน**: ถ้ามี 5 method ที่เป็นไปได้ × 10 path
ที่เป็นไปได้ × 6 status code ที่เป็นไปได้ = **สูงสุด 300 time series** สำหรับ metric ตัวนี้ — ตัวเลขนี้**คำนวณ
ล่วงหน้าได้และมีขอบเขตจำกัด (bounded)** เพราะ method/path/status ทั้งหมดมาจาก**ชุดค่าที่รู้ล่วงหน้า** (route
ทั้งหมดของแอปถูก define ไว้ตายตัวใน `Router`)

**กับดักที่ร้ายแรงที่สุดของ Prometheus ทั้งระบบ**: label ที่มีค่า**ไม่จำกัด**หรือ**ไม่รู้ล่วงหน้า** — ทำให้
จำนวน time series **ไม่มีขอบเขตบน** และเติบโตไปเรื่อย ๆ ตาม traffic จริง ไม่ใช่ตาม design ของแอป ตัวอย่างที่
เกิดขึ้นบ่อยที่สุดในโลกจริง:

- **Label ด้วย raw path ที่มี id ติดมา** (เช่น `/books/1`, `/books/2`, ..., `/books/847293`) แทน route
  template (`/books/{id}`) — จำนวน book id ในระบบจริงเพิ่มขึ้นเรื่อย ๆ ไม่มีเพดาน (ยิ่งห้องสมุดมีหนังสือเยอะขึ้น
  ยิ่งมี series เยอะขึ้น)
- **Label ด้วย user id ดิบ ๆ** (เช่น `borrow_requests_total{user_id="8823"}`) — จำนวนผู้ใช้เพิ่มขึ้นเรื่อย ๆ
  ไม่มีเพดานเหมือนกัน แถมยังเสี่ยงเรื่อง privacy ถ้า user id เชื่อมโยงกับข้อมูลส่วนตัวได้
- **Label ด้วย session id, request id, IP address ดิบ** — แต่ละอย่างนี้**ไม่ซ้ำกันเลยแทบทุก request** ทำให้
  ทุก request สร้าง time series ใหม่หนึ่งตัวเสมอ (กรณีที่ร้ายแรงที่สุด — เท่ากับ "1 series ต่อ 1 request" ซึ่ง
  ขัดกับจุดประสงค์ของการ aggregate โดยสิ้นเชิง)

**พิสูจน์ตัวเลขจริง**: เขียนตัวอย่าง "ผิด" ที่ label ด้วย raw path (`req.uri().path()` ตรง ๆ ไม่ผ่าน
`MatchedPath`) แล้วยิง request ไปที่ book id 40 ตัวที่ต่างกัน:

```rust
// *** ตัวอย่างนี้ "ผิด" โดยตั้งใจ — ห้ามเอาไปใช้จริง ใช้สาธิตปัญหาเท่านั้น ***
async fn track_metrics_bad(req: Request<axum::body::Body>, next: Next) -> Response {
    let path = req.uri().path().to_owned(); // <- กับดัก: raw path มี id ดิบติดมาด้วย เช่น "/books/42"
    let method = req.method().to_string();
    let response = next.run(req).await;
    let status = response.status().as_u16().to_string();
    let labels = [("method", method), ("path", path), ("status", status)];
    metrics::counter!("http_requests_total_bad", &labels).increment(1);
    response
}
```

```bash
for id in $(seq 1 40); do curl -sS -o /dev/null http://127.0.0.1:3301/books/$id; done
curl -sS http://127.0.0.1:3301/metrics | grep -c '^http_requests_total_bad{'
```

ผลลัพธ์จริง:

```
40
```

**40 request ไปที่ 40 book id ที่ต่างกัน → 40 time series ที่ต่างกัน** — ตัวอย่างจริงบางบรรทัดจาก `/metrics`:

```
http_requests_total_bad{method="GET",path="/books/1",status="200"} 1
http_requests_total_bad{method="GET",path="/books/33",status="200"} 1
http_requests_total_bad{method="GET",path="/books/38",status="200"} 1
http_requests_total_bad{method="GET",path="/books/14",status="200"} 1
```

เทียบกับเวอร์ชัน**ที่ถูก**ในหัวข้อ 98.5 ที่ใช้ `MatchedPath` — ยิง request ไปที่ book id ต่างกัน**เท่าไหร่ก็ตาม**
ผ่าน endpoint `/books/{id}/borrow` จะรวมกันเป็น**แค่ time series เดียว** (`path="/books/{id}/borrow"`) ตามที่
พิสูจน์แล้วในหัวข้อ 98.5 (ยิง `/books/1/borrow` และ `/books/2/borrow` แล้วรวมกันเป็นบรรทัดเดียวที่มีค่า
`status="200"` เท่ากับ 2 ไม่ใช่สองบรรทัดแยกกัน)

**ทำไมเรื่องนี้ถึงร้ายแรงในระบบจริง**: ห้องสมุดที่มีหนังสือ 100,000 เล่ม กับ user 1,000,000 คน ถ้า label ผิดทั้ง
สองมิติพร้อมกัน (path ดิบ + user id ดิบ) จำนวน time series ที่เป็นไปได้ทางทฤษฎีคือ **100,000 × 1,000,000 =
100,000,000,000 (หนึ่งแสนล้าน) time series** ที่อาจถูกสร้างขึ้นจริงตาม traffic — Prometheus เก็บทุก time
series ไว้ใน memory เพื่อให้ query เร็ว (in-memory TSDB) ตัวเลขระดับนี้ทำให้ memory การใช้งานของ Prometheus
server พุ่งจนล่มได้ง่าย ๆ (เอกสารทางการของ Prometheus เตือนเรื่องนี้ไว้ตรง ๆ ในหัวข้อ "cardinality" ของ best
practice guide) — **นี่คือเหตุผลที่ทำไมการเลือก label ต้องคิดล่วงหน้าเสมอว่า "ค่าที่เป็นไปได้ของ label นี้มี
ขอบเขตจำกัดและรู้ล่วงหน้าไหม"**

**หลักการที่ใช้ตัดสินใจได้จริงในทางปฏิบัติ**: label ที่ปลอดภัยคือ label ที่มาจาก**ชุดค่าคงที่ที่ออกแบบไว้ล่วง
หน้า** (route template, HTTP method, ช่วง status code, ชื่อ environment, region, ประเภทหนังสือ/genre ถ้ามี
จำนวนประเภทจำกัด) — ถ้าอยากรู้ข้อมูลระดับเจาะจงตัวบุคคล/entity เดี่ยว ๆ (เช่น "user คนไหนยืมหนังสือเล่มไหนบ้าง
บ่อยแค่ไหน") **นั่นคือหน้าที่ของ logs หรือ traces** (กลับไปที่หัวข้อ 98.1) **ไม่ใช่หน้าที่ของ metrics label** —
metrics ถูกออกแบบมาให้ตอบคำถามระดับ**สรุปรวม** ไม่ใช่ระดับรายบุคคล การพยายามยัดข้อมูลระดับรายบุคคลเข้าไปใน
label คือการใช้เครื่องมือผิดประเภทตั้งแต่ต้น

### 98.8 รัน Prometheus จริงในเครื่อง (Docker, ต่อยอด Part 96)

ตอนนี้แอปมี `/metrics` endpoint ที่ทำงานถูกต้องแล้ว — มาให้ Prometheus server จริงมา scrape มันตามโมเดล
pull-based ที่อธิบายไว้ในหัวข้อ 98.3

#### เขียน `prometheus.yml`

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: "metrics_demo"
    static_configs:
      - targets: ["127.0.0.1:3300"]
```

`job_name` คือชื่อที่ Prometheus ใช้กำกับกลุ่มของ target ทั้งหมดที่ config นี้ scrape (จะกลายเป็นค่า label
`job` ที่ติดมากับทุก metric ที่ scrape ได้จาก target กลุ่มนี้อัตโนมัติ) `static_configs.targets` คือรายการ
`host:port` ที่จะไปดึงข้อมูลจาก path `/metrics` เป็นค่าเริ่มต้น (เปลี่ยนได้ด้วย `metrics_path` ถ้า endpoint
ของคุณอยู่ที่ path อื่น) — ระบบ production จริงที่รันบน Kubernetes มักใช้ `kubernetes_sd_configs` แทน
`static_configs` เพื่อให้ Prometheus ค้นหา target เองอัตโนมัติผ่าน service discovery (ตามที่อธิบายไว้ใน
หัวข้อ 98.3) แต่สำหรับการทดสอบในเครื่องคนเดียว `static_configs` แบบตายตัวนี้เพียงพอและตรงไปตรงมาที่สุด

#### รัน Prometheus ด้วย Docker (ต่อยอด Part 96)

```bash
docker run -d --name prom_demo --network host \
  -v "$(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml:ro" \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml --web.listen-address=127.0.0.1:9090
```

สังเกต flag `--network host`: ใช้เพราะแอป Axum ของเรารันอยู่บนเครื่อง host โดยตรง (ไม่ได้ containerize) ที่
`127.0.0.1:3300` — ถ้าให้ Prometheus container ใช้ network แบบ default (bridge) มันจะมองไม่เห็น
`127.0.0.1:3300` ของ host เลย (เพราะ `127.0.0.1` ข้างในตัว container หมายถึง container ตัวเอง ไม่ใช่ host)
`--network host` ทำให้ container แชร์ network namespace เดียวกับ host ตรง ๆ จึงเห็น `127.0.0.1:3300` ได้ปกติ
— **ถ้าแอปของคุณ containerize ไว้เหมือนกันแล้วรันคู่กันด้วย `docker-compose`** (สถานการณ์ที่พบบ่อยกว่าใน
production จริงตามที่ Part 96 สอน) ให้ใช้ชื่อ service ที่ Docker network ตั้งให้แทน (เช่น
`targets: ["metrics_demo:3300"]`) โดยไม่ต้องพึ่ง `--network host` เลย

ตรวจว่า container รันขึ้นจริง:

```bash
docker ps --filter name=prom_demo
```

ผลลัพธ์จริง:

```
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS     NAMES
77077994399a   prom/prometheus:latest   "/bin/prometheus --c…"   3 seconds ago   Up 3 seconds             prom_demo
```

#### ยืนยันว่า Scrape สำเร็จจริง: หน้า `/targets` และ Metric `up`

Prometheus เปิด HTTP API ของตัวเองไว้ให้ query สถานะได้ — ยิง `/api/v1/targets` เพื่อดูว่า target ที่ตั้งไว้
ใน `prometheus.yml` ถูก scrape สำเร็จหรือไม่:

```bash
curl -sS http://127.0.0.1:9090/api/v1/targets
```

ผลลัพธ์จริง (ตัดเฉพาะส่วนสำคัญ):

```json
{
    "status": "success",
    "data": {
        "activeTargets": [
            {
                "discoveredLabels": {
                    "__address__": "127.0.0.1:3300",
                    "__metrics_path__": "/metrics",
                    "job": "metrics_demo"
                },
                "labels": {
                    "instance": "127.0.0.1:3300",
                    "job": "metrics_demo"
                },
                "scrapeUrl": "http://127.0.0.1:3300/metrics",
                "lastError": "",
                "lastScrapeDuration": 0.005403059,
                "health": "up"
            }
        ]
    }
}
```

**`"health": "up"`** และ **`"lastError": ""`** (ไม่มี error เลย) คือหลักฐานตรง ๆ ว่า Prometheus scrape
`http://127.0.0.1:3300/metrics` สำเร็จจริง — ถ้าแอปยังไม่รันหรือ network ไปไม่ถึง ค่านี้จะเป็น `"down"` และ
`lastError` จะมีข้อความ error จริง เช่น `"connection refused"` ให้เห็นตรงนี้

Prometheus เก็บผลลัพธ์ของการ scrape แต่ละครั้งไว้เป็น metric พิเศษชื่อ **`up`** เสมอ (`1` = scrape สำเร็จ, `0`
= ล้มเหลว) — query ผ่าน HTTP API endpoint `/api/v1/query`:

```bash
curl -sS 'http://127.0.0.1:9090/api/v1/query?query=up'
```

ผลลัพธ์จริง:

```json
{
    "status": "success",
    "data": {
        "resultType": "vector",
        "result": [
            {
                "metric": {
                    "__name__": "up",
                    "instance": "127.0.0.1:3300",
                    "job": "metrics_demo"
                },
                "value": [
                    1790486348.193,
                    "1"
                ]
            }
        ]
    }
}
```

`"value": [1790486348.193, "1"]` — timestamp (Unix epoch) คู่กับค่า `"1"` (up) — ยืนยันชัดเจนว่า Prometheus
server จริงกำลัง scrape แอป Axum ของเราสำเร็จอยู่ ณ ขณะนั้น นี่คือ metric ตัวแรกที่ควรตรวจสอบเสมอเมื่อ debug
ปัญหา "ทำไม dashboard ไม่มีข้อมูล" — ถ้า `up` เป็น `0` แปลว่าปัญหาอยู่ที่การเชื่อมต่อ/scrape ไม่ใช่ที่ตัว query
หรือ dashboard

### 98.9 PromQL พื้นฐาน: `rate()` และ `histogram_quantile()`

มีข้อมูลจริงใน Prometheus แล้ว มาเรียน**PromQL** (ภาษา query ของ Prometheus) สองฟังก์ชันที่ใช้บ่อยที่สุดใน
งานจริง

#### `rate()`: แปลง Counter สะสมให้เป็นอัตราต่อวินาที

จำจากหัวข้อ 98.2 ว่า Counter ดิบ ๆ (เช่น `http_requests_total = 48213`) ไม่มีประโยชน์มากนักโดยตัวมันเอง —
สิ่งที่มีประโยชน์คือ**อัตราการเปลี่ยนแปลง**เทียบเวลา `rate(metric[ช่วงเวลา])` คำนวณ**อัตราเฉลี่ยต่อวินาที**
ของ counter ในช่วงเวลาที่กำหนด (และจัดการกรณี counter reset ตอน process restart ให้อัตโนมัติ — ถ้าค่าลดลง
กะทันหันระหว่างสองจุดข้อมูล `rate()` จะตีความว่าเป็น restart ไม่ใช่ "ค่าลดลงจริง" ซึ่งสมเหตุสมผลเพราะ Counter
ไม่มีทางลดได้ตามคำนิยาม)

สร้าง traffic ต่อเนื่องแล้ว query จริง:

```bash
for i in $(seq 1 20); do
  curl -sS -o /dev/null http://127.0.0.1:3300/books
  curl -sS -o /dev/null -X POST http://127.0.0.1:3300/books/2/borrow
  curl -sS -o /dev/null -X POST http://127.0.0.1:3300/books/2/return
  sleep 0.3
done
```

```bash
curl -sS --data-urlencode 'query=rate(http_requests_total[1m])' http://127.0.0.1:9090/api/v1/query
```

ผลลัพธ์จริง (ตัดเฉพาะ series สำคัญ):

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {"method": "GET", "path": "/books", "status": "200"},
        "value": [1790486381.21, "0.3740095238095238"]
      },
      {
        "metric": {"method": "POST", "path": "/books/{id}/borrow", "status": "200"},
        "value": [1790486381.21, "0.3740095238095238"]
      },
      {
        "metric": {"method": "POST", "path": "/books/{id}/return", "status": "200"},
        "value": [1790486381.21, "0.3668666666666667"]
      }
    ]
  }
}
```

**ประมาณ 0.37 request ต่อวินาที** สำหรับ `/books` และ `/books/{id}/borrow` — สมเหตุสมผลกับ traffic ที่สร้างไว้
(1 request ทุก ~0.9 วินาทีต่อ endpoint จาก loop ที่ sleep 0.3 วินาทีระหว่างสาม request ต่อรอบ ≈ 1/0.9 ≈ 0.37
ครั้งต่อวินาที เฉลี่ยผ่าน window 1 นาที) — สังเกตว่า `rate()` คืนค่าต่อ**label combination** แยกกัน (แต่ละ
`path`/`method`/`status` ได้ rate ของตัวเอง) ถ้าอยากได้ **rate รวมทุก path** ต้องห่อด้วย `sum by (...)`:

```bash
curl -sS --data-urlencode 'query=sum by (path) (rate(http_requests_total[1m]))' http://127.0.0.1:9090/api/v1/query
```

ผลลัพธ์จริง:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {"metric": {"path": "/books/{id}/return"}, "value": [1790486374.049, "0.41146666666666665"]},
      {"metric": {"path": "/books"}, "value": [1790486374.049, "0.42813333333333337"]},
      {"metric": {"path": "/books/{id}/borrow"}, "value": [1790486374.049, "0.42813333333333337"]}
    ]
  }
}
```

`sum by (path) (...)` รวม series ที่มี `method`/`status` ต่างกันแต่ `path` เดียวกันเข้าด้วยกัน — pattern นี้คือ
สิ่งที่ dashboard จริง (เช่น Grafana) ใช้สร้างกราฟ "request rate ต่อ endpoint" ที่เห็นกันทั่วไป

#### `histogram_quantile()`: คำนวณ p50/p95/p99 Latency จาก Histogram

ก่อนอื่นต้องแน่ใจว่า metric เป็น **Histogram จริง** (มี `_bucket`) ไม่ใช่ Summary — ย้อนกลับไปหัวข้อ 98.2:
ถ้าไม่กำหนด bucket boundary ให้ `metrics-exporter-prometheus` ตอน build recorder มันจะ export
`histogram!()` เป็น **Summary** โดยอัตโนมัติ (ควบคุมด้วยตัวมันเองภายใน ไม่ใช่ Histogram) พิสูจน์ได้จริง — ถ้า
`main()` เขียนแค่:

```rust
let prometheus_handle = PrometheusBuilder::new()
    .install_recorder() // ไม่มี .set_buckets_for_metric(...) เลย
    .unwrap();
```

`/metrics` จะโชว์:

```
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{method="GET",path="/books",status="200",quantile="0.95"} 0.000056194029822588095
http_request_duration_seconds_sum{method="GET",path="/books",status="200"} 0.00045940000000000005
http_request_duration_seconds_count{method="GET",path="/books",status="200"} 9
```

สังเกต **`# TYPE ... summary`** และไม่มี `_bucket` เลยแม้แต่ตัวเดียว — `histogram_quantile()` ใน PromQL ทำงาน
กับ vector ที่มาจาก series ที่ลงท้าย `_bucket` เท่านั้นตามสเปกของมัน ถ้า metric ของคุณกลายเป็น Summary โดยไม่
ตั้งใจ (ลืมกำหนด bucket) การเขียน query `histogram_quantile(0.95, ...)` จะได้ **vector ว่างเปล่า** กลับมาเสมอ
เพราะไม่มี series `_bucket` ให้มันหาเจอ — นี่ไม่ใช่ error ที่ Prometheus แจ้งเตือนตรง ๆ แต่เป็นผลลัพธ์เงียบ ๆ
ที่ทำให้ dashboard latency ว่างเปล่าโดยไม่มีคำอธิบาย จึงต้องระวังให้แม่นตั้งแต่ตอน config recorder

**แก้ไขให้ export เป็น Histogram จริง** ด้วย `set_buckets_for_metric`:

```rust
let prometheus_handle = PrometheusBuilder::new()
    .set_buckets_for_metric(
        metrics_exporter_prometheus::Matcher::Suffix("duration_seconds".to_string()),
        &[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
    )
    .expect("ตั้งค่า bucket ไม่สำเร็จ")
    .install_recorder()
    .expect("ติดตั้ง Prometheus recorder ไม่สำเร็จ");
```

`Matcher::Suffix("duration_seconds")` บอกว่า **metric ชื่อไหนก็ตามที่ลงท้ายด้วย `duration_seconds`** ให้ใช้
bucket boundary ชุดนี้ (มีประโยชน์เพราะแอปจริงมักมี histogram หลายตัว ไม่ใช่แค่ตัวเดียว — ตั้งกฎครั้งเดียวครอบ
คลุมทุกตัวที่ตรง pattern) — build ใหม่แล้ว `/metrics` จะเปลี่ยนเป็น:

```
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="GET",path="/books",status="200",le="0.001"} 8
http_request_duration_seconds_bucket{method="GET",path="/books",status="200",le="0.005"} 8
http_request_duration_seconds_bucket{method="GET",path="/books",status="200",le="0.01"} 8
...
http_request_duration_seconds_bucket{method="GET",path="/books",status="200",le="+Inf"} 8
http_request_duration_seconds_sum{method="GET",path="/books",status="200"} 0.00039306
http_request_duration_seconds_count{method="GET",path="/books",status="200"} 8
```

`# TYPE ... histogram` และมี `_bucket` ครบทุก boundary — ตอนนี้ query จริงได้แล้ว:

```bash
curl -sS --data-urlencode \
  'query=histogram_quantile(0.95, sum by (le, path) (rate(http_request_duration_seconds_bucket[5m])))' \
  http://127.0.0.1:9090/api/v1/query
```

ผลลัพธ์จริง:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {"metric": {"path": "/books/{id}/borrow"}, "value": [1790486381.318, "0.0009500000000000001"]},
      {"metric": {"path": "/books"}, "value": [1790486381.318, "0.0009500000000000001"]},
      {"metric": {"path": "/books/{id}/return"}, "value": [1790486381.318, "0.00095"]}
    ]
  }
}
```

**p95 latency ≈ 0.00095 วินาที (0.95ms)** สำหรับทุก endpoint — สมเหตุสมผลเพราะแอปทดสอบนี้เป็น in-memory ล้วน
ๆ ไม่มี I/O จริงเลย จึงเร็วระดับ microsecond

**ข้อสังเกตสำคัญที่ต้องรู้จริงจากตัวเลขนี้**: latency จริงที่วัดได้ (จากหัวข้อ 98.5) อยู่ที่ระดับ **20-90
ไมโครวินาที (0.00002-0.00009 วินาที)** ซึ่ง**เร็วกว่า bucket boundary ที่เล็กที่สุดที่ตั้งไว้ (`0.001` = 1
มิลลิวินาที) มาก** — ทุก observation จริงจึงตกอยู่ใน bucket แรกสุดหมด (`le="0.001"` มีค่าเท่ากับ `_count`
ทั้งหมดพอดี) ทำให้ `histogram_quantile()` ไม่มีข้อมูลเพียงพอที่จะบอกตำแหน่งที่แม่นยำภายใน bucket นั้น ต้อง
**สมมติว่าข้อมูลกระจายสม่ำเสมอ (linear interpolation)** ภายในช่วง `[0, 0.001]` แทน — ได้ผลลัพธ์ที่เป็นแค่
**ค่าประมาณอย่างหยาบ** (0.00095 คือประมาณกึ่งกลางบนของ bucket แรก ไม่ใช่ตัวเลขที่วัดได้จริงตรง ๆ) นี่คือบท
เรียนสำคัญที่ต้องจำ: **bucket boundary ต้องเลือกให้เหมาะกับ latency จริงของระบบที่กำลังวัด** — ระบบ web
service จริงที่มี I/O (database, network call ไปยัง service อื่น) มัก latency อยู่ในช่วง 1ms ถึงหลายวินาที
boundary แบบที่ตั้งในตัวอย่างนี้ (1ms ถึง 5 วินาที) เหมาะกับสถานการณ์นั้นมากกว่าแอป in-memory ล้วน ๆ ที่ใช้
สาธิตในบทนี้ — ถ้า latency จริงส่วนใหญ่กระจุกอยู่ใน bucket เดียว `histogram_quantile()` จะให้ผลลัพธ์ที่ไม่มี
ความละเอียดพอจะแยกความแตกต่างได้ ควรเพิ่ม bucket ที่ละเอียดขึ้นในช่วงที่ traffic จริงตกอยู่

#### Query Business Metric ตรง ๆ: ไม่ต้องผ่าน `rate()`/`histogram_quantile()` เสมอไป

Counter/Gauge บางตัวไม่จำเป็นต้องผ่านฟังก์ชันอะไรเลย — บางครั้งค่า**ปัจจุบัน**ตรง ๆ (instant value) คือสิ่งที่
ต้องการอยู่แล้ว ต่างจาก `http_requests_total` ที่ต้องผ่าน `rate()` ก่อนถึงจะมีความหมาย (เพราะ "จำนวนสะสมตั้งแต่
restart" ไม่ค่อยมีประโยชน์) — business metric อย่าง `library_books_available` (Gauge) กับ
`library_books_borrowed_total` (Counter) มีประโยชน์แม้ query แบบ instant value ตรง ๆ:

```bash
curl -sS 'http://127.0.0.1:9090/api/v1/query?query=library_books_borrowed_total'
curl -sS 'http://127.0.0.1:9090/api/v1/query?query=library_books_available'
```

ผลลัพธ์จริง:

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [{
      "metric": {"__name__": "library_books_borrowed_total", "instance": "127.0.0.1:3300", "job": "metrics_demo"},
      "value": [1790486374.102, "24"]
    }]
  }
}
```

```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [{
      "metric": {"__name__": "library_books_available", "instance": "127.0.0.1:3300", "job": "metrics_demo"},
      "value": [1790486374.151, "3"]
    }]
  }
}
```

**มีคนยืมหนังสือสำเร็จสะสมทั้งหมด 24 ครั้ง และตอนนี้เหลือสำเนาว่างอยู่ 3 เล่ม** — ตัวเลขทั้งสองนี้ตอบคำถามทาง
ธุรกิจได้ตรง ๆ ทันทีโดยไม่ต้องแปลงอะไรเลย ต่างจาก `http_requests_total` ที่ค่าสะสมดิบ (เช่น "48,213 request")
ไม่มีความหมายที่น่าสนใจด้วยตัวเอง — **หลักการเลือกว่า query ตรง ๆ หรือต้องผ่าน `rate()`/`histogram_quantile()`
ก่อน**: ถามตัวเองว่า "ค่า ณ ขณะนี้ (หรือค่าสะสมทั้งหมด) มีความหมายทางธุรกิจในตัวเองไหม" ถ้าใช่ (เช่น "สำเนาว่าง
เหลือกี่เล่ม", "ยืมไปแล้วกี่ครั้งทั้งหมด") query ตรง ๆ ได้เลย — ถ้าสิ่งที่อยากรู้จริง ๆ คือ "อัตราการเปลี่ยนแปลง
เทียบเวลา" (เช่น "เร็วแค่ไหน", "ช่วงนี้มีคนยืมถี่ขึ้นไหมเทียบเมื่อวาน") ต้องผ่าน `rate()`/`increase()` ก่อนเสมอ

#### Aggregation Operators: `sum()`, `avg()`, `max()`, `count()`

หัวข้อก่อนหน้าใช้ `sum by (path) (...)` ไปแล้วครั้งหนึ่งโดยไม่ได้อธิบายรายละเอียด — PromQL มี aggregation
operator มาตรฐานที่ทำงานคล้าย `GROUP BY` ของ SQL (เชื่อมกับความเข้าใจเรื่อง SQL aggregate function จาก Part
70-71 ได้ตรง ๆ):

| Operator | ความหมาย | ตัวอย่างการใช้กับระบบห้องสมุด |
|---|---|---|
| `sum by (label) (...)` | รวมค่าทุก series ที่มีค่า `label` เดียวกันเข้าด้วยกัน | `sum by (path) (rate(http_requests_total[1m]))` — request rate รวมทุก status/method ต่อ endpoint |
| `sum without (label) (...)` | รวมทุก series เข้าด้วยกัน **ยกเว้น** ที่ระบุ (ตรงข้ามกับ `by`) | `sum without (instance) (up)` — จำนวน target ที่ up รวมทุก instance ของ job เดียวกัน |
| `avg by (label) (...)` | ค่าเฉลี่ยของ series ที่ label ตรงกัน | `avg by (job) (http_request_duration_seconds_sum / http_request_duration_seconds_count)` — latency เฉลี่ยต่อ job |
| `max by (label) (...)` | ค่าสูงสุด — มีประโยชน์มากตอนหา instance ที่ผิดปกติ | `max by (instance) (rate(http_requests_total{status="500"}[5m]))` — instance ไหนมี error rate สูงสุด |
| `count(...)` | จำนวน series ที่ match (ไม่ใช่ผลรวมค่า แต่นับจำนวน series) | `count(up == 1)` — จำนวน instance ที่ยังมีชีวิตอยู่ตอนนี้ |

ไม่มี label ระบุ (`sum(...)` เฉย ๆ ไม่มี `by`/`without`) จะรวม**ทุก series ที่ match เข้าเป็นตัวเดียว** — เช่น
`sum(rate(http_requests_total[1m]))` ให้ request rate รวม**ทั้งระบบ** ไม่แยกตาม path/method/status เลย มี
ประโยชน์ตอนอยากรู้ตัวเลขภาพรวมสุด ๆ แบบเดียว (เช่น panel ตัวเลขเดี่ยวบน dashboard "total request/sec ทั้งระบบ")

#### `increase()`: เพื่อนคู่ของ `rate()` ที่ให้ผลลัพธ์เป็นจำนวนครั้ง ไม่ใช่อัตรา

`rate(counter[5m])` คืน**อัตราต่อวินาที** (หน่วยเป็น "ต่อวินาที" เสมอ ไม่ว่า range vector จะยาวแค่ไหน) —
บางครั้งคำถามที่อยากตอบไม่ใช่ "อัตราต่อวินาที" แต่คือ **"เกิดขึ้นไปกี่ครั้งในช่วงเวลานี้"** ตรง ๆ — ฟังก์ชัน
`increase(counter[ช่วงเวลา])` ตอบคำถามนี้โดยตรง (ทางคณิตศาสตร์ `increase(m[5m])` ≈ `rate(m[5m]) * 300` เพราะ
5 นาที = 300 วินาที — `increase()` แค่คูณ range กลับเข้าไปให้ ไม่ต้องคำนวณเองอีกที) ตัวอย่างที่เหมาะกับ
`increase()` มากกว่า `rate()`: "มีคนยืมหนังสือไปกี่ครั้งใน 1 ชั่วโมงที่แล้ว" — เขียนเป็น
`increase(library_books_borrowed_total[1h])` อ่านง่ายกว่าและตรงคำถามกว่าเทียบกับต้องคูณ 3600 เข้ากับ
`rate(library_books_borrowed_total[1h])` เอง

**หลักการเลือก**: ใช้ `rate()` เมื่อจะสร้างกราฟ "แนวโน้มอัตราเทียบเวลา" (แกน Y เป็น "ต่อวินาที") ใช้
`increase()` เมื่อต้องการ "จำนวนครั้งสะสมในช่วงเวลาที่กำหนด" (แกน Y เป็น "จำนวนครั้ง") — ทั้งสองใช้ได้กับ
Counter เท่านั้น (ใช้กับ Gauge ไม่มีความหมาย เพราะ Gauge ขึ้นลงได้ ไม่ใช่ค่าสะสม)

#### Recording Rules: ลดภาระของ Query ที่ซับซ้อนและถูกเรียกบ่อย (ระดับ Awareness)

Query อย่าง `histogram_quantile(0.95, sum by (le, path) (rate(http_request_duration_seconds_bucket[5m])))`
ที่เขียนไว้ข้างบนต้องคำนวณใหม่ทุกครั้งที่ dashboard refresh (ทุก 5-30 วินาทีตามที่ Grafana ตั้งไว้) — ถ้ามี
หลาย panel ที่ใช้ query แบบเดียวกันซ้ำ ๆ (เช่น ทีมมี 10 dashboard ที่ต่างก็มี panel "p95 latency" เหมือนกัน)
Prometheus จะคำนวณ query เดียวกันซ้ำ ๆ หลายรอบโดยไม่จำเป็น — Prometheus มีกลไก **recording rule** ที่ให้ตั้งค่า
query ที่ซับซ้อนไว้ล่วงหน้าใน config แยก แล้วให้ Prometheus **คำนวณผลลัพธ์เก็บไว้เป็น metric ใหม่**ตามรอบเวลาที่
กำหนด (เช่นทุก 1 นาที) — dashboard แค่ query metric ที่คำนวณเสร็จแล้วนี้ตรง ๆ (เร็วกว่าเพราะไม่ต้องคำนวณสด) บท
นี้ไม่ลงรายละเอียดการตั้งค่า recording rule เพราะสำหรับ traffic ระดับที่หลักสูตรนี้สอน (single instance, ไม่
ใช่ scale ระดับหลักพัน query ต่อวินาที) การ query สดตรง ๆ ตามที่สอนไว้ทั้งบทเพียงพอแล้ว — recording rule คือ
optimization ที่ควรพิจารณาเมื่อพบว่า query ตัวใดตัวหนึ่งกินเวลานานผิดปกติหรือถูกเรียกซ้ำบ่อยมากในหลาย dashboard

### 98.10 Capstone: Instrument ระบบห้องสมุดเต็มรูปแบบ (ต่อยอด Part 92-94) + Grafana ในระดับ Awareness

ประกอบทุกอย่างเข้าด้วยกัน — นี่คือ `main.rs` ฉบับสมบูรณ์ของตัวอย่างที่ใช้พิสูจน์ผลลัพธ์ทุกอย่างในบทนี้ (โดเมน
ห้องสมุดแบบ in-memory ที่ยืนแทนระบบ Postgres+SQLx เต็มรูปแบบของ Part 92 — โครงสร้าง handler/middleware ที่
แสดงในหัวข้อ 98.5-98.6 คือสิ่งที่จะเพิ่มเข้าไปใน `main.rs`/`routes/borrows.rs` จริงของ Part 92 ตรง ๆ ไม่ต่างกัน):

```rust
use axum::{
    extract::{MatchedPath, Path, State},
    http::{Request, StatusCode},
    middleware::{self, Next},
    response::Response,
    routing::{get, post},
    Json, Router,
};
use metrics_exporter_prometheus::{Matcher, PrometheusBuilder, PrometheusHandle};
use serde::Serialize;
use std::{
    collections::HashMap,
    sync::{Arc, Mutex},
    time::Instant,
};

#[derive(Clone)]
struct AppState {
    catalog: Arc<Mutex<HashMap<u32, Book>>>,
    prometheus: PrometheusHandle,
}

#[derive(Clone, Serialize)]
struct Book {
    id: u32,
    title: String,
    total_copies: u32,
    available_copies: u32,
}

fn seed_catalog() -> HashMap<u32, Book> {
    let mut m = HashMap::new();
    m.insert(1, Book { id: 1, title: "Zero To Production In Rust".into(), total_copies: 2, available_copies: 2 });
    m.insert(2, Book { id: 2, title: "The Rust Programming Language".into(), total_copies: 3, available_copies: 3 });
    m.insert(3, Book { id: 3, title: "Programming Rust".into(), total_copies: 1, available_copies: 1 });
    m
}

fn total_available(catalog: &HashMap<u32, Book>) -> f64 {
    catalog.values().map(|b| b.available_copies as f64).sum()
}

async fn health() -> &'static str { "ok" }

async fn list_books(State(state): State<AppState>) -> Json<Vec<Book>> {
    let catalog = state.catalog.lock().unwrap();
    Json(catalog.values().cloned().collect())
}

async fn borrow_book(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<Json<Book>, (StatusCode, String)> {
    let mut catalog = state.catalog.lock().unwrap();
    let book = catalog.get_mut(&id).ok_or((StatusCode::NOT_FOUND, format!("ไม่พบหนังสือ id {id}")))?;
    if book.available_copies == 0 {
        return Err((StatusCode::CONFLICT, "ไม่มีสำเนาว่างให้ยืม".to_string()));
    }
    book.available_copies -= 1;
    let snapshot = book.clone();

    metrics::counter!("library_books_borrowed_total").increment(1);
    metrics::gauge!("library_books_available").set(total_available(&catalog));

    Ok(Json(snapshot))
}

async fn return_book(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<Json<Book>, (StatusCode, String)> {
    let mut catalog = state.catalog.lock().unwrap();
    let book = catalog.get_mut(&id).ok_or((StatusCode::NOT_FOUND, format!("ไม่พบหนังสือ id {id}")))?;
    if book.available_copies >= book.total_copies {
        return Err((StatusCode::CONFLICT, "หนังสือเล่มนี้ไม่มีการยืมค้างอยู่".to_string()));
    }
    book.available_copies += 1;
    let snapshot = book.clone();

    metrics::counter!("library_books_returned_total").increment(1);
    metrics::gauge!("library_books_available").set(total_available(&catalog));

    Ok(Json(snapshot))
}

async fn metrics_handler(State(state): State<AppState>) -> String {
    state.prometheus.render()
}

async fn track_http_metrics(req: Request<axum::body::Body>, next: Next) -> Response {
    let start = Instant::now();
    let path = req
        .extensions()
        .get::<MatchedPath>()
        .map(|mp| mp.as_str().to_owned())
        .unwrap_or_else(|| req.uri().path().to_owned());
    let method = req.method().to_string();

    let response = next.run(req).await;

    let status = response.status().as_u16().to_string();
    let elapsed = start.elapsed().as_secs_f64();
    let labels = [("method", method), ("path", path), ("status", status)];

    metrics::counter!("http_requests_total", &labels).increment(1);
    metrics::histogram!("http_request_duration_seconds", &labels).record(elapsed);

    response
}

#[tokio::main]
async fn main() {
    let prometheus_handle = PrometheusBuilder::new()
        .set_buckets_for_metric(
            Matcher::Suffix("duration_seconds".to_string()),
            &[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
        )
        .expect("ตั้งค่า bucket ไม่สำเร็จ")
        .install_recorder()
        .expect("ติดตั้ง Prometheus recorder ไม่สำเร็จ");

    let state = AppState {
        catalog: Arc::new(Mutex::new(seed_catalog())),
        prometheus: prometheus_handle,
    };

    let instrumented_routes = Router::new()
        .route("/health", get(health))
        .route("/books", get(list_books))
        .route("/books/{id}/borrow", post(borrow_book))
        .route("/books/{id}/return", post(return_book))
        .route_layer(middleware::from_fn(track_http_metrics));

    let app = Router::new()
        .merge(instrumented_routes)
        .route("/metrics", get(metrics_handler))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3300").await.unwrap();
    println!("ฟังอยู่ที่ http://127.0.0.1:3300");
    axum::serve(listener, app).await.unwrap();
}
```

**การไหลของงานทั้งหมด** (สรุปสิ่งที่บทนี้พิสูจน์ไว้ทั้งหมดเป็นภาพเดียว):

1. รัน `cargo run` → แอป Axum เปิดพอร์ต 3300 พร้อม `/metrics`
2. รัน Prometheus จริงผ่าน Docker ด้วย `prometheus.yml` ที่ชี้ไปที่ `127.0.0.1:3300` (หัวข้อ 98.8) → หน้า
   `/targets` ของ Prometheus ยืนยัน `"health": "up"` → metric `up` มีค่า `1`
3. ยิง traffic จริงเข้าแอป — บาง request เป็น `borrow`/`return` (business metric) บางส่วนเป็น `GET /books`
   ธรรมดา (HTTP metric ล้วน ๆ)
4. Middleware ที่แนบด้วย `.route_layer()` (หัวข้อ 98.5) บันทึก `http_requests_total` +
   `http_request_duration_seconds` ให้**ทุก endpoint โดยอัตโนมัติ** โดย handler ไม่ต้องรู้เรื่อง metrics เลย
5. Handler `borrow_book`/`return_book` เพิ่ม `library_books_borrowed_total`,
   `library_books_returned_total`, `library_books_available` (หัวข้อ 98.6) ให้ข้อมูลระดับธุรกิจที่ HTTP
   metric อย่างเดียวตอบไม่ได้
6. Query จริงผ่าน PromQL (หัวข้อ 98.9): `sum by (path) (rate(http_requests_total[1m]))` ดู request rate ต่อ
   endpoint, `histogram_quantile(0.95, ...)` ดู p95 latency, `library_books_borrowed_total` ดูยอดยืมสะสม
   ตรง ๆ, `library_books_available` ดูจำนวนสำเนาว่างปัจจุบัน

#### ขยายไปยัง Endpoint อื่นของ Part 92: `books.rs` และ `auth.rs`

ตัวอย่างข้างบนโฟกัสที่ `borrow_book`/`return_book` เพราะเป็น business event ที่ชัดเจนที่สุด แต่หลักการเดียวกัน
ใช้กับ handler อื่นของ Part 92 ได้ทั้งหมด — สองตัวอย่างที่คุ้มค่าเพิ่มเข้าไปจริงในระบบ production:

**`routes/books.rs` — `create_book` (เฉพาะ admin ตาม Part 76)**: เพิ่ม gauge ที่รายงานขนาด catalog รวม
(จำนวนชื่อเรื่องทั้งหมด ไม่ใช่จำนวนสำเนา) ทุกครั้งที่ admin เพิ่มหนังสือใหม่สำเร็จ — ค่านี้มีขอบเขตจำกัดที่รู้
ล่วงหน้าได้ในทางปฏิบัติ (จำนวนชื่อเรื่องในห้องสมุดหนึ่งไม่ใช่ตัวเลขที่ระเบิดแบบไม่มีเพดานเหมือน raw request
path) จึงปลอดภัยที่จะเป็น Gauge เดี่ยว ๆ โดยไม่มี label:

```rust
// src/routes/books.rs — เพิ่มเข้าไปใน create_book ของ Part 92 หลัง tx.commit().await? สำเร็จ
metrics::counter!("library_books_created_total").increment(1);
metrics::gauge!("library_catalog_size").increment(1.0);
```

**`routes/auth.rs` — `login` (ต่อยอด Part 74 เรื่อง JWT)**: เพิ่ม counter ที่แยก label ตามผลลัพธ์ (`outcome`)
ของการ login — ค่าที่เป็นไปได้ของ `outcome` มีแค่สองค่า (`"success"`/`"failure"`) จึงไม่เสี่ยง high-cardinality
ตามหลักการหัวข้อ 98.7 (ตรงข้ามกับการ label ด้วย username/email ดิบ ๆ ที่จะเป็นกับดักทันที):

```rust
// src/routes/auth.rs — เพิ่มเข้าไปใน login ของ Part 92/74 ทั้งสองจุดที่ return
pub async fn login(/* ... */) -> Result<Json<LoginResponse>, AppError> {
    let user = match find_user_by_email(&state.db, &payload.email).await? {
        Some(u) if verify_password(&payload.password, &u.password_hash) => u,
        _ => {
            metrics::counter!("library_login_attempts_total", "outcome" => "failure").increment(1);
            return Err(AppError::Unauthorized("อีเมลหรือรหัสผ่านไม่ถูกต้อง".to_string()));
        }
    };

    metrics::counter!("library_login_attempts_total", "outcome" => "success").increment(1);
    // ... ออก JWT ตามที่ Part 74 สอนไว้ ...
    Ok(Json(login_response))
}
```

Metric `library_login_attempts_total{outcome="failure"}` ที่โตเร็วผิดปกติ (ตรวจด้วย `rate()` เหมือนหัวข้อ
98.9) คือสัญญาณเตือนที่มีประโยชน์มากในทางปฏิบัติ — อาจบอกว่ามีคนพยายาม brute-force รหัสผ่านอยู่ หรือ frontend
มี bug ส่ง credential ผิดซ้ำ ๆ โดยไม่ตั้งใจ ทั้งสองกรณีเป็นเรื่องที่ทีม on-call อยากรู้ทันทีโดยไม่ต้องรอไปเจอ
จาก log ทีละบรรทัด

#### ตารางสรุป Metric ทั้งหมดที่ระบบห้องสมุดที่ Instrument เต็มรูปแบบมี

| Metric | ประเภท | Label | ตอบคำถามอะไร |
|---|---|---|---|
| `http_requests_total` | Counter | `method`, `path`, `status` | มี traffic เข้าเท่าไหร่ ต่อ endpoint ต่อ status code |
| `http_request_duration_seconds` | Histogram | `method`, `path`, `status` | latency กระจายตัวยังไง (p50/p95/p99 ผ่าน `histogram_quantile()`) |
| `library_books_borrowed_total` | Counter | ไม่มี | มีการยืมสำเร็จสะสมทั้งหมดกี่ครั้ง |
| `library_books_returned_total` | Counter | ไม่มี | มีการคืนสำเร็จสะสมทั้งหมดกี่ครั้ง |
| `library_books_available` | Gauge | ไม่มี | ตอนนี้เหลือสำเนาว่างกี่เล่มในระบบ |
| `library_books_created_total` | Counter | ไม่มี | admin เพิ่มหนังสือใหม่ไปแล้วกี่ครั้ง |
| `library_catalog_size` | Gauge | ไม่มี | ตอนนี้มีชื่อเรื่องกี่เรื่องในระบบ |
| `library_login_attempts_total` | Counter | `outcome` (`success`/`failure`) | อัตราการ login สำเร็จ/ล้มเหลวเป็นยังไง |

สังเกตว่า**เกือบทุก business metric ไม่มี label เลย หรือมี label แค่ตัวเดียวที่มีค่าจำกัดสองสามค่า** — ตรงกับ
หลักการหัวข้อ 98.7 ที่ว่า business metric ระดับสรุป (aggregate) ไม่ต้องการความละเอียดสูงเท่า HTTP metric ที่
ต้องแยกตาม endpoint จริง ๆ (ซึ่งก็ยังมีขอบเขตจำกัดเพราะ endpoint ถูก define ไว้ตายตัวใน `Router` เหมือนกัน)

#### Grafana: ขั้นต่อไปสำหรับ Dashboard (ระดับ Awareness)

ตลอดบทนี้เราดู metric ผ่าน Prometheus API ตรง ๆ (`/api/v1/query`) และหน้า `/targets` ของ Prometheus เอง —
เพียงพอสำหรับการเรียนรู้และ debug เฉพาะจุด แต่ระบบ production จริงต้องการ **dashboard ที่แสดงกราฟแบบ visual
ต่อเนื่อง** ให้ทีม operations ดูภาพรวมได้ทันทีโดยไม่ต้องเขียน PromQL เองทุกครั้ง — เครื่องมือมาตรฐานของโลกจริง
สำหรับงานนี้คือ **Grafana**: เป็น dashboard tool แยกต่างหากที่**เชื่อมต่อ Prometheus เป็น data source**
(ชี้ไปที่ URL ของ Prometheus server เดียวกับที่บทนี้ตั้งค่าไว้) แล้วให้คุณสร้าง **panel** แต่ละอันจาก PromQL
query เดียวกับที่เรียนในหัวข้อ 98.9 ตรง ๆ (เช่น panel กราฟ request rate จาก
`sum by (path) (rate(http_requests_total[1m]))`, panel latency percentile จาก `histogram_quantile(...)`,
panel ตัวเลขเดี่ยว "จำนวนหนังสือว่างตอนนี้" จาก `library_books_available`) — Grafana ไม่ได้แทนที่ Prometheus
เลย มันเป็นแค่ **ชั้นการแสดงผล (visualization layer)** ที่วางอยู่บนข้อมูลที่ Prometheus เก็บไว้อยู่แล้ว ทุก
query ที่เขียนได้ในหัวข้อ 98.9 นำไปใส่ใน Grafana panel ได้โดยไม่ต้องเรียนรู้ภาษาใหม่เลย หลักสูตรนี้ยังไม่มีบท
ที่ลงรายละเอียดการตั้งค่า Grafana เต็มรูปแบบ — ถ้าคุณต้องทำ dashboard จริงในงาน ขั้นตอนถัดไปที่ควรลองด้วยตัวเอง
คือติดตั้ง Grafana (มี Docker image สำเร็จรูปเช่นเดียวกับ Prometheus ที่ทดสอบในบทนี้), เพิ่ม Prometheus เป็น
data source ผ่านหน้า UI, แล้วเอา PromQL query ที่เขียนไว้แล้วในหัวข้อ 98.9 มาวางเป็น panel ได้ทันที

#### Alertmanager: แจ้งเตือนอัตโนมัติจาก Metric (ระดับ Awareness)

Dashboard (Grafana) แก้ปัญหา "อยากดูตัวเลขภาพรวม" แต่ไม่แก้ปัญหา "อยากรู้**ทันที**ตอนมีอะไรผิดปกติโดยไม่ต้อง
เปิด dashboard เฝ้าดูตลอดเวลา" — Prometheus มี component เสริมชื่อ **Alertmanager** ที่ทำหน้าที่นี้โดยเฉพาะ:
คุณกำหนด**alerting rule** เป็น PromQL expression ที่ถ้าเป็นจริงต่อเนื่องนานเกินเวลาที่กำหนด (เช่น "p95 latency
สูงกว่า 1 วินาทีต่อเนื่อง 5 นาที") Prometheus จะส่ง alert ไปให้ Alertmanager แล้ว Alertmanager เป็นคนจัดการ
เรื่อง**ช่องทางแจ้งเตือน** (ส่ง Slack, email, PagerDuty, ...) รวมถึงกฎการจัดกลุ่ม/ระงับ alert ที่ซ้ำกัน (เช่น
ไม่ต้องส่ง alert เดิม 100 ครั้งถ้ามันยังไม่หาย) — ตัวอย่าง alerting rule ที่ใช้ query จากหัวข้อ 98.9 ตรง ๆ:

```yaml
# alert.rules.yml (ตัวอย่างแนวคิด — ไม่ได้รันจริงในบทนี้)
groups:
  - name: library_api_alerts
    rules:
      - alert: HighP95Latency
        expr: histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m]))) > 1
        for: 5m
        annotations:
          summary: "p95 latency สูงกว่า 1 วินาทีต่อเนื่องเกิน 5 นาที"
```

สังเกตว่า `expr` คือ PromQL query แบบเดียวกับที่เรียนในหัวข้อ 98.9 เป๊ะ (`histogram_quantile()` รวมกับ
`rate()`) — ความรู้ PromQL ที่ได้จากบทนี้ใช้ต่อยอดไปเขียน alerting rule ได้ตรง ๆ โดยไม่ต้องเรียนภาษาใหม่ เช่น
เดียวกับที่ใช้ต่อยอดไปเขียน Grafana panel ได้ — บทนี้ไม่ลงรายละเอียดการตั้งค่า Alertmanager เต็มรูปแบบ เพราะ
เป็นเนื้อหาที่ควรมาหลังจากมี metric ที่ออกแบบถูกต้องแล้ว (ตามที่บทนี้ทั้งบทปูพื้นไว้) — เขียน alerting rule
ที่ query metric ที่ label ผิด/cardinality สูงจะได้ alert ที่ไม่น่าเชื่อถือหรือช้าเกินจะมีประโยชน์

#### รันแอปคู่กับ Prometheus ด้วย `docker-compose` (ทางเลือกที่ใช้บ่อยกว่าในงานจริง)

บทนี้ใช้ `docker run --network host` ในหัวข้อ 98.8 เพราะแอป Axum รันอยู่บนเครื่อง host โดยตรง — แต่ระบบ
production จริงส่วนใหญ่ containerize **ทั้งแอปและ Prometheus พร้อมกัน** (ต่อยอด Part 96) ซึ่งเปิดโอกาสให้ใช้
`docker-compose` จัดการทั้งคู่ในไฟล์เดียว โดยไม่ต้องพึ่ง `--network host` เลย (Docker network แบบ default ของ
compose ทำให้ container คุยกันด้วย**ชื่อ service** ได้ตรง ๆ):

```yaml
# docker-compose.yml — ตัวอย่างแนวทาง (ไม่ได้รันจริงในบทนี้ แต่เป็น pattern มาตรฐานเมื่อ containerize ทั้งคู่)
services:
  metrics_demo:
    build: .
    ports:
      - "3300:3300"

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    depends_on:
      - metrics_demo
```

จุดที่ต้องเปลี่ยนคือ `prometheus.yml` — แทนที่จะชี้ไปที่ `127.0.0.1:3300` (ซึ่งข้างใน container ของ
`prometheus` หมายถึง container ตัวเองเสมอ ไม่ใช่ container อื่นในเครือข่ายเดียวกัน) ให้ใช้**ชื่อ service**
ที่ compose ตั้งให้ตรง ๆ:

```yaml
scrape_configs:
  - job_name: "metrics_demo"
    static_configs:
      - targets: ["metrics_demo:3300"] # ชื่อ service ใน docker-compose.yml ไม่ใช่ IP/127.0.0.1
```

Docker network ภายใน compose มี DNS ในตัวที่ resolve ชื่อ service เป็น IP ของ container นั้นให้อัตโนมัติ — นี่
คือเหตุผลที่ `--network host` ไม่จำเป็นอีกต่อไปเมื่อทั้งสองฝั่งอยู่ใน compose stack เดียวกัน (ต่างจากหัวข้อ
98.8 ที่แอปรันบน host โดยตรงและ Prometheus รันใน container แยก จึงต้องพึ่ง `--network host` เพื่อให้มองเห็น
`127.0.0.1` ร่วมกัน)

### 98.11 ทดสอบ Metrics Middleware อัตโนมัติด้วย `tower::ServiceExt` (ต่อยอด Part 95)

ทุกอย่างที่บทนี้พิสูจน์มาจนถึงตอนนี้ทำผ่าน `curl` แบบ manual — เหมาะกับการเรียนรู้และ debug ครั้งเดียว แต่ระบบ
production จริงต้องมั่นใจว่า middleware ยังบันทึก metric ถูกต้องอยู่**ทุกครั้งที่มีคนแก้โค้ด** ไม่ใช่แค่ตอน
เขียนครั้งแรก — **Part 95 (Testing Web Applications แบบครบวงจร)** สอนวิธีเขียน integration test ให้ Axum
router ด้วย `tower::ServiceExt::oneshot()` (ยิง request หนึ่งตัวเข้า `Router` ตรง ๆ ในหน่วยความจำ โดยไม่ต้อง
เปิด TCP listener จริงเลย) — เทคนิคเดียวกันนี้ใช้ยืนยันพฤติกรรมของ metrics middleware ได้โดยตรง:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn health_endpoint_increments_request_counter() {
        // ติดตั้ง recorder ใหม่แยกต่างหากสำหรับ test นี้ — สำคัญมาก: ห้ามใช้ recorder ตัวเดียวกับที่
        // main() ติดตั้งไว้ (ถ้า test รันพร้อมกันหลายตัว จะแย่ง global recorder กันจนค่านับผิด)
        let handle = metrics_exporter_prometheus::PrometheusBuilder::new()
            .install_recorder()
            .unwrap();

        let app = build_app(); // ฟังก์ชันที่คืน Router พร้อม middleware ตามหัวข้อ 98.5

        let response = app
            .oneshot(Request::builder().uri("/health").body(Body::empty()).unwrap())
            .await
            .unwrap();

        assert_eq!(response.status(), 200);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"ok");

        // ยืนยันว่า middleware บันทึก metric จริง ไม่ใช่แค่ handler ตอบ response ถูกต้องอย่างเดียว
        let rendered = handle.render();
        assert!(rendered.contains(
            r#"http_requests_total{method="GET",path="/health",status="200"} 1"#
        ));
    }
}
```

รันจริงด้วย `cargo test` ได้ผลลัพธ์จริง:

```
running 1 test
test tests::health_endpoint_increments_request_counter ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**สิ่งที่ test นี้พิสูจน์**: ไม่ใช่แค่ "handler ตอบ 200 ถูกต้อง" (ซึ่ง test ปกติของ Part 95 ก็เช็คได้อยู่แล้ว)
แต่พิสูจน์ว่า **middleware ที่แนบไว้ด้วย `.route_layer()` บันทึก metric ที่ label ถูกต้องเป๊ะจริง** — ถ้ามีคน
มาแก้ path ของ route ในอนาคต (เช่นเปลี่ยน `/health` เป็น `/healthz`) โดยลืมอัปเดต assertion ในเทสต์ (หรือกลับกัน
ลืมว่า `MatchedPath` จะเปลี่ยนตาม route ใหม่โดยอัตโนมัติ) — test นี้จะ fail ทันทีเป็นสัญญาณเตือนแทนที่จะไปรู้
ตัวอีกทีตอน dashboard ใน production แสดงข้อมูลผิดเงียบ ๆ — นี่คือคุณค่าของการทดสอบ observability infrastructure
เองด้วย เช่นเดียวกับที่ Part 95 สอนให้ทดสอบ business logic

## กับดักที่พบบ่อย (Common Pitfalls)

1. **ลืมติดตั้ง recorder — metric เงียบหายไปทั้งหมดโดยไม่มี error**: เหมือนกับดักคลาสสิกของ `log` crate ใน
   Part 60 เป๊ะ — ถ้าไม่เรียก `PrometheusBuilder::new().install_recorder()` (หรือเรียกแต่ไม่ได้ `.expect()`/
   `?` จน error หายไปเงียบ ๆ) ทุกครั้งที่เรียก `metrics::counter!(...)`, `metrics::gauge!(...)`,
   `metrics::histogram!(...)` จะทำงาน**ผ่านไปเรื่อย ๆ ไม่ panic ไม่ error แต่ไม่มีอะไรถูกบันทึกจริง** พิสูจน์
   ได้จริงด้วยการรันโปรแกรมที่เรียก macro เหล่านี้โดยไม่ติดตั้ง recorder เลย — จะได้ output ปกติของโปรแกรม
   โดยไม่มีสัญญาณเตือนอะไรว่า metrics หายไป วิธีตรวจสอบที่ใช้ได้จริง: ยิง `curl` ไปที่ `/metrics` ทันทีหลัง
   deploy ตัว build ใหม่ ถ้าเจอ endpoint ว่างเปล่าสนิท (ไม่ใช่แค่ metric ที่คาดหวังหายไปตัวเดียว แต่**ทุกอย่าง**
   หายไปรวมถึง metric มาตรฐานที่ `metrics-exporter-prometheus` ควรสร้างให้อัตโนมัติด้วย) ให้ตรวจว่า
   `install_recorder()` ถูกเรียกจริงตอน `main()` เริ่มทำงานก่อนจุดอื่นใด

2. **ใช้ `.layer()` แล้ว fallback ไปใช้ raw path ตอน `MatchedPath` เป็น `None` — เปิดช่องให้ bot/scanner สร้าง
   time series ไม่จำกัด**: พิสูจน์ไว้ในหัวข้อ 98.5 ว่า `.layer()` (ต่างจาก `.route_layer()`) ยังถูกเรียกสำหรับ
   request ที่**ไม่ match route ไหนเลย** และตอนนั้น `req.extensions().get::<MatchedPath>()` คืน `None` — ถ้า
   middleware เขียน `.unwrap_or_else(|| req.uri().path().to_owned())` เป็น fallback (ดูเหมือนปลอดภัยตอนเขียน
   ครั้งแรก) **ทุก path ที่ scanner ยิงมาแบบสุ่ม** (`/wp-admin`, `/.env`, `/books/99999999`, ...) จะกลายเป็น
   time series ใหม่ทุกครั้งไม่จำกัด แก้ได้สองทาง: (ก) ใช้ `.route_layer()` แทน (ตัดปัญหานี้ทิ้งไปเลยเพราะไม่
   ถูกเรียกสำหรับ request ที่ไม่ match) หรือ (ข) ถ้าจำเป็นต้องใช้ `.layer()` เพราะต้องการนับ 404 ด้วย ให้
   fallback เป็นค่าคงที่ เช่น `"unmatched"` แทน raw path เสมอ ไม่ใช่ path จริง

3. **Label ด้วยค่าที่ไม่มีขอบเขต (raw path/user id/session id) — time series ระเบิด**: พิสูจน์ไว้ในหัวข้อ 98.7
   ว่า label ด้วย raw path (ไม่ผ่าน `MatchedPath`) ทำให้ 40 request ไปที่ 40 book id ที่ต่างกันสร้าง **40 time
   series** ขึ้นมาจริง เทียบกับแค่ **1 time series** ถ้าใช้ route template — ในระบบจริงที่มี traffic สูงและ
   entity (book id, user id) เพิ่มขึ้นไม่มีเพดาน ปัญหานี้ทำให้ memory ของ Prometheus server โตแบบไม่มีขอบเขต
   จนล่มได้ วิธีตรวจจับ: ก่อน deploy metric ใหม่ ถามตัวเองเสมอว่า "ค่าที่เป็นไปได้ของ label ตัวนี้มีขอบเขต
   จำกัดที่รู้ล่วงหน้าไหม" ถ้าคำตอบคือ "ไม่รู้ ขึ้นกับ data ที่เพิ่มเข้ามาเรื่อย ๆ" ให้ตัด label นั้นทิ้งแล้วใช้
   logs/traces แทนสำหรับข้อมูลระดับ entity เดี่ยว ๆ

4. **สับสน Histogram กับ Summary — `histogram_quantile()` ได้ vector ว่างเปล่าเงียบ ๆ**: พิสูจน์ไว้ในหัวข้อ
   98.9 ว่า `PrometheusBuilder::new().install_recorder()` โดยไม่เรียก `.set_buckets_for_metric(...)` (หรือ
   `.set_buckets(...)`) จะทำให้ `metrics::histogram!(...)` export เป็น Prometheus **Summary** (มี `# TYPE ...
   summary` ไม่มี `_bucket`) แทน **Histogram** จริง — `histogram_quantile()` ใน PromQL ทำงานกับ series ที่ลง
   ท้าย `_bucket` เท่านั้น เมื่อไม่มี `_bucket` เลย query จะคืน **vector ว่างเปล่า** โดยไม่มี error ใด ๆ แจ้ง
   เตือน ทำให้ panel latency บน dashboard ว่างเปล่าอย่างงงงวยว่าเกิดจากอะไร วิธีตรวจสอบ: `curl` ไปที่
   `/metrics` แล้วดูบรรทัด `# TYPE metric_name ...` ของ histogram metric ทุกตัวว่าเป็น `histogram` (มี
   `_bucket`) ไม่ใช่ `summary`

5. **ลืมแปลงหน่วยเวลาเป็นวินาที — ผิด Prometheus convention**: Prometheus (และเครื่องมือรอบ ๆ มันเกือบทั้งหมด
   รวมถึง Grafana) มี**ธรรมเนียมปฏิบัติที่เข้มงวด**ว่า metric ที่วัดเวลา**ต้องใช้หน่วยวินาที (`_seconds`)
   เสมอ** ไม่ใช่ milliseconds — ถ้าเขียน `start.elapsed().as_millis() as f64` แทน
   `start.elapsed().as_secs_f64()` แล้วตั้งชื่อ metric ว่า `http_request_duration_seconds` (ชื่อบอกว่าเป็น
   วินาทีแต่ค่าจริงเป็น milliseconds) ตัวเลขจะผิดไปถึง **1,000 เท่า** อย่างเงียบ ๆ ทุก query ที่อ้างอิง metric
   นี้ (รวมถึง `histogram_quantile()`, alert threshold ที่ตั้งไว้) จะผิดพลาดตามไปหมดโดยไม่มี error ใด ๆ เตือน
   เพราะทางเทคนิคมันคือแค่ตัวเลข `f64` ธรรมดา — ตรวจสอบด้วยการดูชื่อตัวเองเทียบกับหน่วยที่ใช้จริงเสมอ: ชื่อ
   ลงท้าย `_seconds` ต้องมาจาก `.as_secs_f64()` เท่านั้น

6. **สร้าง label array จากค่าที่มี type ต่างกัน — compile error ที่ error message ไม่ชี้ตรงไปที่ต้นเหตุ**:
   array literal ของ Rust ต้องมีสมาชิกทุกตัว**type เดียวกันเป๊ะ** (ตามที่ Part 8 สอนไว้เรื่อง array) —
   `labels` ที่ประกอบจาก tuple หลายตัวเพื่อส่งให้ `metrics::counter!(name, &labels)` ก็ต้องเป็นแบบนี้เหมือนกัน:
   ถ้า element ที่สองของแต่ละ tuple เป็นคนละ type กัน (เช่น tuple แรกเป็น `(&str, axum::http::Method)` เพราะ
   ลืมเรียก `.to_string()` ตอนดึง method ออกมา แต่ tuple ที่สองเป็น `(&str, String)`) จะได้ error จริงแบบนี้:

   ```
   error[E0308]: mismatched types
    --> src/main.rs:4:48
     |
   4 |     let labels = [("method", method), ("path", "/books".to_string())];
     |                                                ^^^^^^^^^^^^^^^^^^^^ expected `Method`, found `String`
   ```

   จุดที่ทำให้กับดักนี้น่าหลงทาง: error ชี้ไปที่บรรทัดสร้าง `labels` (ตำแหน่งที่ถูกต้อง) แต่คำอธิบาย
   "expected `Method`, found `String`" ไม่ได้บอกตรง ๆ ว่า "ปัญหาคือคุณลืม `.to_string()` ตัวแรก" — ต้องไล่ดู
   เองว่า tuple ไหนใน array ที่มี type ไม่ตรงกับตัวอื่น วิธีป้องกันที่ใช้ได้จริงเสมอ: **แปลงทุกค่าใน label
   array ให้เป็น `String` (หรือ `&'static str` ถ้าเป็นค่าคงที่) อย่างสม่ำเสมอตั้งแต่ต้น** ก่อนประกอบเข้า array
   เดียวกัน แบบที่โค้ดตัวอย่างในหัวข้อ 98.5 ทำ (`method.to_string()`, `status.as_u16().to_string()`)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม Counter ชื่อ `library_health_checks_total` ที่นับจำนวนครั้งที่ endpoint `/health` ถูกเรียก
   (แยกจาก `http_requests_total` ที่ middleware บันทึกให้อัตโนมัติอยู่แล้ว) — เพิ่มโค้ดเข้าไปในฟังก์ชัน
   `health()` ตรง ๆ แล้วพิสูจน์ด้วยการยิง `curl /health` สามครั้งแล้วดูค่าใน `/metrics` ว่าตรงกับ 3 จริง
   *(hint: `metrics::counter!("library_health_checks_total").increment(1);` ใส่ในบรรทัดแรกของฟังก์ชัน
   `health()`)*

2. **(กลาง)** เพิ่ม Gauge ชื่อ `library_catalog_size` ที่รายงานจำนวนหนังสือทั้งหมด (ไม่ใช่จำนวนสำเนา แต่จำนวน
   "ชื่อเรื่อง" ที่แตกต่างกันในระบบ) แล้วเพิ่ม endpoint ใหม่ `POST /books` ที่รับ JSON `{title, total_copies}`
   เพื่อเพิ่มหนังสือใหม่เข้า catalog — อัปเดต gauge ทุกครั้งที่มีการเพิ่มหนังสือสำเร็จ พิสูจน์ด้วยการเพิ่ม
   หนังสือ 2 เล่มแล้วดูว่า `library_catalog_size` เปลี่ยนจาก 3 เป็น 5 *(hint: catalog เดิมเริ่มที่ 3 เล่ม ตาม
   `seed_catalog()` — ตั้งค่าเริ่มต้นของ gauge นี้ตอน `main()` เริ่มทำงานด้วย
   `metrics::gauge!("library_catalog_size").set(3.0)` ก่อน แล้วค่อย increment ทุกครั้งที่เพิ่มหนังสือใหม่
   สำเร็จ)*

3. **(ยาก)** ปรับ middleware `track_http_metrics` ให้เพิ่ม label ที่สี่ชื่อ `is_error` (ค่าเป็น `"true"` ถ้า
   status code ≥ 400, `"false"` ถ้าไม่ใช่) โดยที่**ไม่ทำให้ label combination ที่เป็นไปได้เพิ่มขึ้นแบบไม่จำกัด**
   (เพราะ `is_error` มีค่าที่เป็นไปได้แค่สองค่า ตามหลักการหัวข้อ 98.7) แล้วเขียน PromQL query
   `sum by (path) (rate(http_requests_total{is_error="true"}[1m]))` เพื่อดู **error rate ต่อ endpoint** โดย
   เฉพาะ ยิง request ที่ทำให้เกิด 404/409 หลายครั้งแล้วยืนยันด้วย query จริงว่าตัวเลขที่ได้ถูกต้อง *(hint:
   คำนวณ `let is_error = if response.status().as_u16() >= 400 { "true" } else { "false" };` ก่อนสร้าง
   `labels` array แล้วเพิ่มเข้าไปเป็น element ที่สี่)*

4. **(ยาก/ประยุกต์ใช้งานจริง)** สร้าง endpoint ใหม่ `GET /books/popular` ที่คืนหนังสือที่มีการยืมมากที่สุด 3
   อันดับ — ระหว่างทำ ให้ตัดสินใจว่าจะเก็บ "จำนวนครั้งที่ถูกยืมแยกตามเล่ม" เป็น metric label
   (`library_books_borrowed_total{book_id="1"}`) หรือเป็นข้อมูลใน database/in-memory store แยกต่างหาก
   (ไม่ผ่าน metrics เลย) พร้อมอธิบายเหตุผลการเลือกเป็นลายลักษณ์อักษรโดยอ้างอิงหลักการ cardinality จากหัวข้อ
   98.7 — ระบบห้องสมุดจริงมีหนังสือเป็นหมื่นเป็นแสนเล่ม *(hint: คำตอบที่ถูกต้องคือ**ไม่ควร**ใช้ label
   `book_id` เพราะจำนวนหนังสือไม่มีขอบเขตจำกัดที่รู้ล่วงหน้า — เก็บ "จำนวนครั้งที่ถูกยืมต่อเล่ม" ไว้ใน field
   ของ `Book` struct เอง หรือ column ในฐานข้อมูลแทน แล้วให้ metrics เก็บแค่ระดับสรุป เช่น
   `library_books_borrowed_total` รวมทั้งระบบเหมือนเดิม — นี่คือตัวอย่างจริงของการแยก "สิ่งที่ metrics ควรทำ"
   ออกจาก "สิ่งที่ database ควรทำ" ตามหลักการหัวข้อ 98.1 และ 98.7)*

## สรุป

บทนี้พาไปรู้จัก**เสาที่สองของ observability triad** — metrics — โดยวางกรอบความสัมพันธ์กับ logs (Part 60)
และ traces (Part 99) ให้ชัดเจนก่อนเจาะลึกในรายละเอียด: metrics เหมาะกับคำถามเชิง**แนวโน้มรวม**ที่ query ได้เร็ว
และถูก ต่างจาก logs ที่เหมาะกับรายละเอียดเจาะจง และ traces ที่เหมาะกับเส้นทางของ request เดี่ยว ๆ จากนั้นได้
เรียนรู้ metric ทั้งสี่ประเภทของ Prometheus (Counter, Gauge, Histogram, Summary) พร้อมเหตุผลเชิงลึกว่าทำไม
Histogram ถึงเป็นตัวเลือกที่ดีกว่า Summary สำหรับระบบที่รันหลาย instance — เข้าใจโมเดล pull-based ของ
Prometheus ที่ทำให้แอปพลิเคชันไม่ต้องรู้จัก collector server เลย ติดตั้ง `metrics` crate (facade แบบเดียวกับ
`log` ใน Part 60) คู่กับ `metrics-exporter-prometheus` เขียน middleware ที่วัด HTTP metrics อัตโนมัติทุก
request ด้วย `.route_layer()` (พิสูจน์ความแตกต่างจริงกับ `.layer()` จาก Part 65) เพิ่ม custom business metric
เข้า handler ยืม/คืนหนังสือของ Part 92 ตรง ๆ และที่สำคัญที่สุด — เข้าใจกับดัก **high-cardinality label** อย่าง
ลึกซึ้งพร้อมพิสูจน์ตัวเลขจริง (40 series จากการ label ผิด เทียบกับ 1 series จากการใช้ route template) ปิดท้าย
ด้วยการรัน Prometheus server จริงผ่าน Docker (ต่อยอด Part 96) scrape แอปจริง และเขียน PromQL (`rate()`,
`histogram_quantile()`) ยิง query จริงได้ผลลัพธ์จริงกลับมา

Part ถัดไป (Part 99) จะพาไปดูเสาที่สามที่เหลือ — **distributed tracing ด้วย OpenTelemetry** — ที่ตอบคำถาม
"request หนึ่งตัวเดินทางผ่านระบบยังไง" ซึ่งเป็นคำถามที่ metrics ในบทนี้ตอบไม่ได้เลย (metrics บอกได้แค่ "p95
latency สูง" แต่บอกไม่ได้ว่า request ตัวไหนกินเวลาที่จุดไหนในเส้นทางของมันเอง) — เมื่อจบ Part 99 คุณจะมีภาพ
ครบทั้งสามเสาของ observability พร้อมนำไปประกอบเป็นระบบ monitoring ที่สมบูรณ์ให้กับ API จริงที่สร้างมาตลอด
หลักสูตรนี้

---

**Part ก่อนหน้า:** [CI/CD Pipeline ด้วย GitHub Actions](part-097-cicd-github-actions.md) | **Part ถัดไป:**
[Observability: Distributed Tracing ด้วย OpenTelemetry](part-099-observability-opentelemetry.md)
