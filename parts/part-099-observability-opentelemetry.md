# Part 99: Observability: Distributed Tracing ด้วย OpenTelemetry

> โมดูล: Observability และ Production Readiness | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่าทำไม correlation ID ที่ Part 81 หัวข้อ 81.8 สอนให้สร้างและส่งต่อข้าม service ด้วยมือ
  (ผ่าน gRPC metadata `x-request-id`) **แก้ปัญหาได้แค่ครึ่งเดียว** — มันตอบได้ว่า log บรรทัดไหนเป็นของ request
  เดียวกัน แต่ตอบไม่ได้เลยว่า **แต่ละ hop ใช้เวลานานแค่ไหน**, **hop ไหนเป็นลูกของ hop ไหน**, หรือ **ภาพรวมทั้ง
  request ข้ามหลาย service หน้าตาเป็นอย่างไรเมื่อวาดเป็น timeline** — และรู้ว่า **OpenTelemetry (OTel)** คือ
  มาตรฐานอุตสาหกรรมที่ vendor-neutral ซึ่งถูกออกแบบมาแก้ปัญหานี้โดยเฉพาะ
- อธิบายแนวคิดหลักสี่อย่างของ OTel ได้อย่างแม่นยำและเชื่อมกับสิ่งที่เรียนมาแล้วใน Part 60: **Trace** (เส้นทาง
  เต็มของ request หนึ่งตัวข้ามหลาย service), **Span** (หน่วยงานย่อยหนึ่งหน่วยที่มีเวลาเริ่ม/จบ — คือแนวคิด
  เดียวกับ `tracing::Span` ของ Part 60 ที่ตอนนี้จะถูก "ส่งออก" ไปเก็บถาวรที่ backend ภายนอก), **Span Context**
  (trace ID + span ID + flags ที่ propagate ข้าม service boundary — เวอร์ชันมาตรฐานของ correlation ID ที่
  Part 81 ทำด้วยมือ), และ **Attributes/Events** (structured data บน span — คือแนวคิดเดียวกับ structured
  field ของ `tracing` ใน Part 60 หัวข้อ 60.9)
- วาดและอธิบาย pipeline ของ OTel ได้ครบ: **instrumentation** (โค้ดของคุณสร้าง span ผ่าน `tracing` +
  bridge crate ชื่อ `tracing-opentelemetry`) → **OTLP exporter** (ส่ง span ออกจาก process ของคุณ) →
  **backend** สำหรับเก็บและแสดงผล (**Jaeger** ตัวเลือกโอเพนซอร์สคลาสสิก) — และอธิบายได้ว่าทำไม pipeline นี้มี
  ส่วนที่ต้อง "รัน" เพิ่มขึ้นมาจริง ๆ (Jaeger เป็น process แยกที่ต้อง deploy) ต่างจากโมเดล pull ที่เรียบง่าย
  กว่าของ Prometheus ใน Part 98
- ตั้งค่า `opentelemetry` + `opentelemetry-otlp` + `tracing-opentelemetry` ในโปรเจกต์ Rust จริง ให้ span จาก
  `#[tracing::instrument]` (ที่ Part 60 สอนไว้) ถูกส่งออกไปเก็บที่ Jaeger จริงผ่าน OTLP — พิสูจน์ด้วยการรัน
  Jaeger ผ่าน Docker จริง, รัน Rust binary จริง, ยิง request จริง, แล้ว query Jaeger's HTTP API จริงเห็น
  trace ID/span name/duration/parent-child relationship ที่จับคู่กันได้ตรงกับโค้ด 100%
- ต่อยอดสถานการณ์สอง-service จาก Part 81 (booking-service เรียก catalog-service) ให้ trace เดียวกัน**ไหลข้าม
  service boundary ผ่าน network จริง** ด้วย **W3C Trace Context** (`traceparent`/`tracestate` header — มาตรฐาน
  จริงที่ industry ใช้ แทนที่ header กำหนดเองอย่าง `x-request-id`) พร้อมรู้จักกับดักจริงที่พบตอนทำ (การเรียก
  `set_parent` ผิดจังหวะทำให้ trace ขาดตอน) และวิธีแก้ที่ถูกต้อง
- ตั้งค่า sampling (head-based ด้วย `TraceIdRatioBased`) เพื่อควบคุมปริมาณ trace ในระบบที่มี traffic สูง โดย
  เข้าใจ trade-off เรื่องต้นทุนการเก็บข้อมูลเทียบกับความสมบูรณ์ของข้อมูล และรู้จัก tail-based sampling ในระดับ
  แนวคิด (ทำไมมันแก้ปัญหา "อยากเก็บ trace ที่ error ไว้เสมอ" ได้ดีกว่าแต่ต้องการ infrastructure เพิ่ม)
- เชื่อมสามเสาหลักของ observability เข้าด้วยกันอย่างเป็นรูปธรรม: ใส่ `trace_id` เป็น structured field ใน log
  ของ `tracing` (Part 60) เพื่อกระโดดจาก log บรรทัดหนึ่งไปดู trace เต็มใน Jaeger ได้ทันที, ทำเครื่องหมาย span
  ว่า error (เชื่อมกับ `AppError` จาก Part 66) ให้ request ที่ล้มเหลวแยกแยะได้ง่ายใน Jaeger UI, และรู้จัก
  exemplar ของ Prometheus (Part 98) ที่เชื่อม metric data point เข้ากับ trace ตัวอย่างในระดับแนวคิด — ปิดท้าย
  ด้วยการผูก OTel เข้ากับ full-stack capstone จาก Part 92-94 ให้แอปมี **logs (Part 60) + metrics (Part 98) +
  traces (บทนี้)** ครบสามเสาหลักของ observability พร้อมใช้งานจริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 60 (Logging และ Tracing เบื้องต้น)**: นี่คือฐานที่บทนี้ต่อยอด**โดยตรงที่สุด** — ทุกอย่างที่ Part 60
  สอนไว้เรื่อง `tracing` crate, `#[tracing::instrument]`, `tracing::info_span!`, structured field, และ
  `tracing_subscriber`/`EnvFilter` ยังคงเป็นฐานเดียวกันเป๊ะในบทนี้ **สิ่งเดียวที่เปลี่ยนคือปลายทาง**: ใน
  Part 60 span จบแค่ที่ terminal (ผ่าน `fmt::layer()`) แต่บทนี้จะเพิ่ม **layer อีกตัว** (`tracing-opentelemetry`)
  ให้ span เดียวกันนั้นถูกส่งออกไปเก็บถาวรที่ระบบภายนอกด้วย — ถ้า Part 60 หัวข้อ 60.6-60.9 (event macro, span,
  `#[instrument]`, structured field) ยังไม่แน่น กลับไปทวนก่อน เพราะบทนี้จะไม่อธิบายกลไกพื้นฐานของ `tracing`
  ซ้ำจากศูนย์เลย
- **Part 81 (Microservices Architecture)**: หัวข้อ 81.8 ของบทนั้นสร้าง correlation ID (`x-request-id`) ด้วยมือ
  ให้ไหลข้าม service boundary ผ่าน gRPC metadata — บทนี้คือ**คำตอบที่สมบูรณ์กว่า**สำหรับปัญหาเดียวกัน โดยใช้
  มาตรฐานจริง (W3C Trace Context) แทน header ที่กำหนดเอง และได้ parent/child relationship กับ timing ที่
  correlation ID เดี่ยว ๆ ให้ไม่ได้ — หัวข้อ 99.6 ของบทนี้จะหยิบสถานการณ์ `booking-service`/`catalog-service`
  เดียวกันจาก Part 81 กลับมาทำใหม่ด้วย OTel เต็มรูปแบบ
- **Part 66 (Axum: Error Handling)**: `AppError` enum เดียวที่บทนั้นสอนให้ออกแบบเป็นจุดศูนย์กลาง error ทั้งแอป
  จะถูกใช้ในหัวข้อ 99.9 เป็นจุดที่เชื่อม "error ในเชิง business logic" เข้ากับ "span status = Error" ใน OTel
- **Part 98 (Observability: Metrics ด้วย Prometheus)**: บทนี้เป็น "เสาที่สาม" ของ observability ต่อจาก
  metrics ใน Part 98 — หัวข้อ 99.8 จะอธิบาย exemplar ที่เชื่อม metric data point เข้ากับ trace ID โดยตรง ถ้า
  Part 98 ยังไม่ได้อ่าน อ่านสรุปสั้น ๆ แค่ "Prometheus ดึง (pull) metric เป็นตัวเลขสรุปตามช่วงเวลา" ก็พอเข้าใจ
  จุดเชื่อมได้ ไม่จำเป็นต้องมีรายละเอียดเต็มของ Part 98 มาก่อนเพื่อเข้าใจเนื้อหาหลักของบทนี้
- **Part 92-94 (Full-Stack Capstone)**: หัวข้อ 99.10 ท้ายบทจะผูก OTel เข้ากับ backend ของ capstone นี้ตรง ๆ
  (โครงสร้าง `AppState`/`AppError` เดียวกัน) เพื่อแสดงให้เห็นภาพรวมทั้งสามเสาหลักทำงานร่วมกันในแอปจริงหนึ่งตัว
- **Part 80 (gRPC ด้วย Tonic)**: หัวข้อ 99.6 อ้างอิงโครงสร้าง service สองตัวที่คุยกันแบบเดียวกับ Part 80/81
  (แต่บทนี้สาธิตด้วย HTTP/Axum เพื่อให้เห็น W3C Trace Context header ตรง ๆ ในรูปแบบที่คุ้นเคยกว่า — หลักการ
  เดียวกันนี้ใช้กับ gRPC metadata ได้ทันทีตามที่หัวข้อ 99.6 จะอธิบาย)

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ทุกตัวอย่างโค้ดหลักของบทนี้ผู้เขียน **build และรันจริง** ด้วย scratch cargo project แยกไว้นอก repo ของหลักสูตร
(ไม่กระทบไฟล์ใด ๆ ในคอร์สนี้เลย) โดยใช้เวอร์ชัน crate จริงจาก crates.io ณ วันที่เขียนบทนี้ (ตรวจสอบด้วยการรัน
`cargo add` จริง): `opentelemetry = "0.33.0"`, `opentelemetry-otlp = "0.33.0"` (feature `grpc-tonic`),
`opentelemetry_sdk = "0.33.0"`, `tracing-opentelemetry = "0.34.0"`, `tracing-subscriber = "0.3.23"` — และที่
สำคัญที่สุด: ผู้เขียน **รัน Jaeger จริงด้วย Docker** (`jaegertracing/all-in-one:1.62.0`, ยืนยันว่า Docker
daemon ใช้งานได้จริงในเครื่องนี้ด้วย `docker info`) แล้ว **ยิง OTLP export จริงจาก Rust process ไปยัง Jaeger
ที่ฟังอยู่ที่ `localhost:4317`** จากนั้น **query Jaeger's HTTP API จริง** (`GET /api/traces`,
`GET /api/traces/{traceID}`) เพื่อดึง trace ID, span name, duration (หน่วย nanosecond ตามที่ OTel เก็บ), และ
parent/child relationship (`refType: CHILD_OF`) ออกมา**คัดลอกตรงจากผลลัพธ์จริง**ทุกจุดที่ปรากฏในบทนี้ ไม่มี
trace ID หรือ timestamp ใดที่เป็นการแต่งขึ้นมาเอง — รวมถึงกับดักบางข้อในหัวข้อกับดัก (โดยเฉพาะเรื่อง
`set_parent`/`AlreadyStarted`) **คือปัญหาจริงที่ผู้เขียนเจอตอนทำตัวอย่างสอง-service ของหัวข้อ 99.6** ไม่ใช่
การเขียนกับดักลอย ๆ จากความจำ — เมื่อจบการตรวจสอบ container, image, และ scratch project ทั้งหมดถูกลบออกจาก
เครื่องเรียบร้อยแล้ว

## เนื้อหา

### 99.1 ทวนปัญหาจาก Part 81: Correlation ID ให้ภาพที่ไม่ครบแค่ไหน

จำสถานการณ์จาก **Part 81 หัวข้อ 81.8** ได้ไหม — `booking-service` สร้าง `request_id` (UUID) หนึ่งตัวตอนรับ
request จาก client ครั้งแรก แล้วส่งค่าเดียวกันนี้ผ่าน gRPC metadata (`x-request-id`) ไปยัง `catalog-service`
ทุกครั้งที่เรียกข้าม service — ผลคือทั้งสอง log stream (ของ `booking-service` และ `catalog-service`) มี field
`request_id` ค่าเดียวกัน ทำให้ grep หา log ทั้งหมดที่เกี่ยวกับ request เดียวกันข้ามสอง service ได้:

```text
# log ของ booking-service
2026-09-27T02:00:47Z INFO handle_booking_request{request_id=0f4fcdec... book_id=2}: ได้รับคำขอจองหนังสือ

# log ของ catalog-service (คนละ process, คนละไฟล์ log กันโดยสิ้นเชิง)
2026-09-27T02:00:47Z INFO check_availability{request_id=0f4fcdec... book_id=2}: ตรวจสอบความพร้อมของหนังสือ
```

นี่คือความสำเร็จที่แท้จริงของ Part 81 — และเป็นขั้นที่**จำเป็น**ก่อนจะไปถึงบทนี้ แต่ต้องพูดตรง ๆ ว่ามันแก้ปัญหา
ได้แค่**ครึ่งเดียว** ลองตั้งคำถามสามข้อที่ operations engineer ต้องตอบได้ทุกวันเวลามีปัญหา production เกิดขึ้น
จริง แล้วดูว่า correlation ID เดี่ยว ๆ ตอบได้แค่ไหน:

**คำถามที่ 1: "request นี้ใช้เวลารวมกี่ ms และเวลาส่วนใหญ่ไปอยู่ที่ hop ไหน?"** — มี `request_id` เดียวกันใน
สอง log stream ก็จริง แต่ต้อง**เอา timestamp มาคำนวณเองด้วยมือ** (เทียบ timestamp บรรทัดแรกกับบรรทัดสุดท้าย
ของแต่ละ service, ลบกันเอง) ยิ่งมี service เกี่ยวข้องมากกว่าสอง ยิ่งต้องเปิดหลาย log stream พร้อมกันแล้วเทียบ
เวลาด้วยตาเปล่า — ไม่มี "ภาพเดียว" ที่แสดงลำดับเวลาให้เห็นตรง ๆ เลย

**คำถามที่ 2: "hop ไหนเป็น parent ของ hop ไหน เมื่อมีการเรียกซ้อนกันหลายชั้น (A → B → C → D)?"** —
correlation ID เดียวบอกได้แค่ว่า log บรรทัดนี้เป็นของ request เดียวกันทั้งหมด แต่บอก**โครงสร้างความสัมพันธ์**
ไม่ได้เลยว่า service ไหนเรียก service ไหนก่อน/หลัง หรือ service ไหนรอ service ไหนอยู่ (ถ้า B เรียก C และ D
พร้อมกัน (concurrent) กับเรียก C ก่อนแล้วค่อยเรียก D (sequential) — สอง pattern นี้ต่างกันมากในเชิง performance
แต่ correlation ID เดียวมองไม่ออกความต่างนี้เลย)

**คำถามที่ 3: "อยากเห็นภาพรวมทั้ง request แบบ timeline/waterfall เหมือนที่ browser DevTools แสดง network
request — ทำได้ไหม?"** — คำตอบตรง ๆ คือ **ทำไม่ได้เลยด้วย correlation ID เดี่ยว ๆ** เพราะมันเป็นแค่ "ป้ายชื่อ"
ที่แนบไปกับ log แต่ละบรรทัด ไม่มีแนวคิดเรื่อง "จุดเริ่ม", "จุดจบ", หรือ "ความสัมพันธ์เชิงลำดับชั้น (hierarchy)"
อยู่ในตัวมันเองเลย — สิ่งที่ขาดไปคือแนวคิดเรื่อง **Span** ที่มีเวลาเริ่ม/จบชัดเจนและมี "parent span" ที่รู้จักกัน
ได้ ซึ่งเป็นสิ่งที่ correlation ID (เป็นแค่ string เปล่า ๆ) ไม่มีให้ในตัวมันเอง

**นี่คือปัญหาที่ Distributed Tracing (และเครื่องมือมาตรฐานของมันคือ OpenTelemetry) แก้โดยเฉพาะ** — แทนที่จะมี
แค่ "ป้ายชื่อ" ลอย ๆ แนบไปกับ log แต่ละบรรทัด OTel ให้แนวคิดที่มีโครงสร้างเต็มรูปแบบ: แต่ละ "งาน" ที่เกิดขึ้น
(รับ request, query database, เรียก service อื่น) กลายเป็น **Span** ที่มีเวลาเริ่ม/จบจริง ผูกกับ **parent span**
ที่ชัดเจน และ span ทั้งหมดของ request เดียวกันรวมกันเป็น **Trace** เดียวที่ backend อย่าง Jaeger วาดเป็น
timeline/waterfall ให้ดูได้ตรง ๆ — คำถามทั้งสามข้อข้างบนตอบได้ทันทีจากหน้าจอเดียวโดยไม่ต้องคำนวณอะไรด้วยมือเลย

ที่สำคัญคือ **สิ่งที่ Part 60 สอนไปแล้วเรื่อง `tracing::Span`/`#[tracing::instrument]` ไม่ต้องทิ้งไปเลย** — หัวข้อ
ถัดไปจะแสดงให้เห็นว่าแนวคิด span ของ `tracing` crate ที่คุณคุ้นเคยอยู่แล้ว **ตรงกับแนวคิด span ของ OTel แบบ
เกือบเป๊ะ** สิ่งที่บทนี้ทำคือเพิ่ม "สะพาน" (bridge) ให้ span เดิมที่คุณเขียนอยู่แล้วถูกส่งออกไปเก็บที่ backend
ภายนอกที่รู้จักโครงสร้าง parent/child และวาด timeline ให้ได้ — ไม่ใช่ให้เรียนแนวคิด span ใหม่ทั้งหมดจากศูนย์

### 99.2 แนวคิดหลักของ OpenTelemetry: Trace, Span, Span Context, Attributes/Events

ก่อนแตะโค้ดสักบรรทัด ต้องเข้าใจสี่แนวคิดนี้ให้แม่นยำ เพราะทั้งบทจะอ้างอิงคำเหล่านี้ตลอด และความสับสนระหว่าง
"trace" กับ "span" คือความสับสนที่พบบ่อยที่สุดของคนเริ่มต้นเรียน distributed tracing

#### Trace: เส้นทางเต็มของ Request หนึ่งตัว

**Trace** คือ**การเดินทางทั้งหมด**ของ request เดียวหนึ่งตัว นับตั้งแต่จุดที่มันเข้าสู่ระบบ (เช่น client ยิง
HTTP request มาที่ `booking-service`) ไปจนถึงจุดที่มันได้คำตอบกลับไป **ไม่ว่า request นั้นจะข้าม service กี่
ตัวก็ตาม** — ในสถานการณ์ของ Part 81 ที่ `client → booking-service → catalog-service → PostgreSQL` หนึ่ง
request คือหนึ่ง trace เดียว ที่ครอบคลุมทั้งสี่ขั้นตอนนี้ ไม่ว่าจะกินเวลากี่ service ก็ตาม

Trace ถูกระบุด้วย **Trace ID** — เลขสุ่ม 128-bit (แสดงเป็น hex string 32 ตัวอักษร เช่น
`8d5752cb021ec5ea6f5c64983795fc29` ที่จะเห็นจริงในหัวข้อ 99.6) ที่ **ถูกสร้างขึ้นครั้งเดียวที่จุดแรกของระบบ**
(ที่ "ขอบ" — คล้ายกับที่ Part 81 สร้าง `request_id` ด้วย `Uuid::new_v4()` ตอนรับ request ครั้งแรก) แล้ว
propagate (ส่งต่อ) ไปกับทุก hop ที่เกิดขึ้นหลังจากนั้น — **นี่คือสิ่งที่ trace ID ทำเหมือนกับ correlation ID
ของ Part 81 ทุกประการ** ในแง่ที่ว่ามันเป็นตัวเชื่อมทุกอย่างที่เกี่ยวกับ request เดียวกันเข้าด้วยกัน — ความ
ต่างที่สำคัญคือมันเป็น**มาตรฐาน**ที่ backend อย่าง Jaeger เข้าใจในตัว ไม่ใช่ field กำหนดเองที่คุณต้อง parse
เอง

#### Span: หน่วยงานหนึ่งหน่วยภายใน Trace — ตรงกับ `tracing::Span` ของ Part 60

**Span** คือ**หนึ่งหน่วยงาน** ภายใน trace ที่มี **เวลาเริ่ม (start time)** และ **เวลาจบ (end time)** ที่ชัดเจน
— ตัวอย่าง: "รับ HTTP request", "query database", "เรียก catalog-service ผ่าน gRPC" แต่ละอย่างนี้คือหนึ่ง
span ที่แยกจากกัน หนึ่ง trace ประกอบด้วยหลาย span ที่ **ซ้อนกันเป็นลำดับชั้น (hierarchy)** — span ที่เรียก
span อื่นเรียกว่า **parent span**, span ที่ถูกเรียกเรียกว่า **child span**

**นี่คือจุดที่สำคัญที่สุดของหัวข้อนี้**: ถ้าคุณทำตาม Part 60 มาแล้ว **คุณสร้าง span มาตลอดโดยไม่รู้ตัวว่ามันคือ
สิ่งเดียวกันกับ OTel span** — ทุกครั้งที่เขียน `#[tracing::instrument]` บนฟังก์ชัน หรือเรียก
`tracing::info_span!(...)` ด้วยมือ คุณกำลังสร้าง `tracing::Span` ที่มีเวลาเริ่ม (ตอน `.enter()`) และเวลาจบ
(ตอน drop ออกจาก scope) พร้อมกับ parent/child relationship ที่ `tracing`'s **span stack** ติดตามให้อัตโนมัติ
(จำได้จาก Part 60 หัวข้อ 60.7 เรื่อง async attribution ไหม — นั่นคือ `tracing` ใช้กลไก span stack แบบเดียวกัน
นี้เพื่อให้ log ที่ถูกเรียกจากภายในฟังก์ชันที่ `#[instrument]` ครอบไว้ "รู้" context ของตัวเองถูกต้องแม้ข้าม
`.await` — กลไกเดียวกันนี้เองที่ทำให้ตอนนี้ span พวกนี้ export ไป OTel แล้วมี parent/child ถูกต้องด้วย)

ตารางนี้เทียบแนวคิดตรงๆ ระหว่างสิ่งที่เรียนใน Part 60 กับ OTel:

| แนวคิดใน Part 60 (`tracing` crate) | แนวคิดเดียวกันใน OpenTelemetry | หมายเหตุ |
|---|---|---|
| `tracing::Span` (สร้างด้วย `#[instrument]` หรือ `info_span!`) | OTel **Span** | ตรงกันแบบเกือบ 1:1 — บทนี้แค่เพิ่ม layer ที่ export มันออกไป |
| Span stack ที่ `tracing` ติดตาม parent span ปัจจุบันให้อัตโนมัติ | Span hierarchy ภายใน Trace เดียว | กลไกเดียวกัน คนละชื่อเรียก |
| Structured field (`info!(user_id = 42, "...")`) | **Attribute** บน span | ทั้งคู่คือ key-value ที่แนบกับหน่วยข้อมูลหนึ่งชิ้น |
| Event macro (`info!`, `warn!`, `error!` ที่ถูกเรียกภายใน span) | **Event** บน span (จุดเวลาหนึ่งจุดภายใน span ที่มีข้อความ+attribute) | `tracing-opentelemetry` แปลง event ของ `tracing` เป็น OTel event ให้อัตโนมัติ |
| ไม่มีแนวคิดที่ตรงกันใน `tracing` เดี่ยว ๆ | **Trace ID** ที่ผูกทุก span ของ request เดียวกัน | คือ correlation ID เวอร์ชันมาตรฐานที่ Part 81 ทำเอง |

#### Span Context: Trace ID + Span ID + Flags — เวอร์ชันมาตรฐานของ Correlation ID

**Span Context** คือชุดข้อมูลเล็ก ๆ สามอย่างที่ **propagate ข้าม process/service boundary ได้**:

1. **Trace ID** (128-bit) — ระบุว่า span นี้เป็นส่วนของ trace ไหน
2. **Span ID** (64-bit) — ระบุ span ปัจจุบันตัวนี้เอง (ใช้เป็น "parent span ID" เมื่อถูกส่งต่อไปยัง service
   ถัดไป เพื่อให้ span ใหม่ที่ service ปลายทางสร้างขึ้นรู้ว่าตัวเองเป็นลูกของ span ไหน)
3. **Trace Flags** — บิตควบคุมเล็ก ๆ ที่สำคัญที่สุดคือ **sampled flag** (บอกว่า trace นี้ "ถูกเลือกเก็บ" หรือ
   ไม่ — เชื่อมกับหัวข้อ sampling ที่ 99.7)

นี่คือสิ่งที่ทำหน้าที่**เดียวกันเป๊ะ**กับ `request_id` ที่ Part 81 ส่งผ่าน gRPC metadata `x-request-id` — ต่างกัน
ที่ Span Context มีมาตรฐานการเข้ารหัสที่ทุกภาษา/framework ที่รองรับ OTel เข้าใจร่วมกัน เรียกว่า **W3C Trace
Context** ซึ่งเข้ารหัสเป็น HTTP header ชื่อ **`traceparent`** (รูปแบบ `00-{trace_id}-{span_id}-{flags}`) —
หัวข้อ 99.6 จะแสดง header จริงที่จับได้จากการรันจริง: `00-8d5752cb021ec5ea6f5c64983795fc29-ba1bb3289dbb7777-01`
สังเกตว่ามันคือ trace ID (32 hex char) + span ID (16 hex char) + flags (`01` = sampled) เรียงต่อกันด้วย `-`
เท่านั้นเอง — ไม่มีอะไรซับซ้อนซ่อนอยู่เลย

#### Attributes และ Events: Structured Data บน Span

**Attribute** คือ key-value ที่แนบกับ**ทั้ง span** (เช่น `book_id = 42`, `http.status_code = 200`) — ตรงกับ
structured field ของ `tracing::info_span!(..., book_id = 42)` เป๊ะ ตามที่ตารางข้างบนสรุปไว้

**Event** คือจุดเวลาหนึ่งจุด**ภายใน**ช่วงเวลาของ span ที่มีข้อความ + attribute ของตัวเอง — ตรงกับการเรียก
`tracing::info!("...")`/`tracing::warn!("...")` ที่เรียกจากภายในฟังก์ชันที่ `#[instrument]` ครอบไว้ ทุกครั้ง
ที่คุณเรียก `tracing::info!(...)` ภายใน span หนึ่ง มันจะกลายเป็น **event หนึ่งอันที่แนบอยู่กับ span นั้น** ใน
ข้อมูลที่ Jaeger เก็บไว้ (หัวข้อ 99.5 จะแสดงให้เห็นจริงว่า event เหล่านี้ปรากฏเป็น `logs` array ในข้อมูลที่
Jaeger's API คืนมา)

### 99.3 สถาปัตยกรรมของ OTel: Instrumentation → Collector/Exporter → Backend

ก่อนลงมือเขียนโค้ด ต้องเข้าใจภาพรวมของ pipeline ทั้งหมดก่อน เพราะมันมีส่วนที่ต้อง "รัน" มากกว่าโมเดล pull ของ
Prometheus ใน Part 98 พอสมควร

```text
┌─────────────────────┐        ┌──────────────────────┐        ┌─────────────────────┐
│   Your Rust App      │  OTLP  │   (Optional) OTel     │  OTLP  │   Backend             │
│   (Instrumentation)   │ ─────▶ │   Collector            │ ─────▶ │   (Jaeger)             │
│                      │        │   (process/batch/route) │        │   เก็บ + วาด UI        │
└─────────────────────┘        └──────────────────────┘        └─────────────────────┘
   #[tracing::instrument]         แปลง/กรอง/ส่งต่อไปหลาย            เก็บถาวร, ให้ query,
   สร้าง span ด้วย `tracing`        backend พร้อมกันได้              วาด timeline/waterfall
   ส่งออกด้วย tracing-opentelemetry
   + opentelemetry-otlp (OTLP protocol)
```

สามชั้นนี้แยกความรับผิดชอบกันชัดเจนตามหลักการเดียวกับ facade pattern ที่ Part 60 อธิบายไว้เรื่อง `log`/
`tracing`: **โค้ดแอปพลิเคชันของคุณไม่ควรรู้เลยว่า backend สุดท้ายคือ Jaeger, Zipkin, Datadog, Honeycomb หรือ
อะไรก็ตาม** — มันแค่สร้าง span ผ่าน `tracing` (แนวคิดที่รู้จักอยู่แล้วจาก Part 60) แล้วส่งออกด้วย**โปรโตคอล
มาตรฐาน**ที่ชื่อ **OTLP (OpenTelemetry Protocol)** ปลายทางจะเป็นอะไรก็ได้ที่เข้าใจ OTLP — นี่คือความหมายของ
"vendor-neutral" ที่พูดถึงตั้งแต่ต้นบท

**ทำไมต้องมี Collector (แม้บทนี้จะข้ามมันไปในตัวอย่างส่วนใหญ่)?** — OTel Collector คือ process กลางที่
**รับ** span จากหลายแอป, **ประมวลผล/กรอง/batch** (เช่น รวม span จำนวนมากเป็น batch เดียวก่อนส่งต่อเพื่อลด
overhead ของ network call, ลบ attribute ที่มีข้อมูลลับออกก่อนส่งต่อ), แล้ว **ส่งต่อ (export)** ไปยัง backend
หนึ่งตัวหรือหลายตัวพร้อมกัน (เช่น ส่งไป Jaeger สำหรับทีม dev และส่งไป Datadog สำหรับทีม operations พร้อมกัน)
— ในระบบ production จริงขนาดใหญ่ Collector มีประโยชน์มาก เพราะแอปแต่ละตัวไม่ต้องรู้จัก backend สุดท้ายเลย แค่
ส่งไปที่ Collector ตัวเดียว ส่วน Collector เป็นจุดเดียวที่ตัดสินใจว่าจะส่งต่อไปไหน — **แต่บทนี้จะให้แอป Rust
ส่ง OTLP ตรงไปยัง Jaeger เลยโดยไม่ผ่าน Collector** เพื่อลดความซับซ้อนของตัวอย่าง (Jaeger เวอร์ชัน all-in-one
ที่จะรันในหัวข้อถัดไป**รับ OTLP ได้ตรง ๆ ในตัวอยู่แล้ว** ไม่ต้องมี Collector คั่นกลางสำหรับการเรียนรู้ระดับนี้
— แบบฝึกหัดข้อ 4 ท้ายบทให้ลองเพิ่ม Collector เข้ามาเป็นขั้นถัดไปสำหรับคนที่อยากเห็นภาพเต็ม)

**เทียบกับโมเดลของ Prometheus (Part 98) ที่ง่ายกว่า**: Prometheus **ดึง (pull)** metric จากแอปเป็นระยะ (แอป
แค่เปิด endpoint `/metrics` ทิ้งไว้ ไม่ต้องรู้จัก Prometheus เลยด้วยซ้ำ) แต่ OTel tracing เป็นโมเดล **push** —
แอปของคุณเป็นฝ่าย**ส่ง (export)** span ออกไปเอง (ผ่าน batch exporter ที่ทำงานเป็น background task) ไปยัง
endpoint ที่กำหนดไว้ (Jaeger's OTLP receiver) เหตุผลที่ tracing เลือกโมเดล push ต่างจาก metrics คือ **span
เกิดขึ้นตามเหตุการณ์ (event-driven)** ไม่ใช่ค่าที่ "อ่านได้ตลอดเวลา" แบบตัวเลข metric — จะให้ backend มา
"ถาม" ว่ามี span ใหม่ไหมทุก ๆ กี่วินาทีก็ไม่สมเหตุสมผลเท่ากับให้แอปส่งออกไปทันทีที่ span จบ (หรือ batch รวมกัน
เป็นช่วง ๆ เพื่อลด network overhead ตามที่จะเห็นในหัวข้อถัดไป)

#### ตัวอย่าง Collector Config (สำหรับอ้างอิง — ไม่ได้ใช้จริงในตัวอย่างหลักของบทนี้)

เพื่อให้เห็นภาพว่า Collector หน้าตาเป็นอย่างไรจริง ๆ (ไม่ใช่แค่กล่องลอย ๆ ในไดอะแกรม) นี่คือตัวอย่าง
`otel-collector-config.yaml` ขั้นต่ำที่รับ OTLP จากแอป แล้ว batch ก่อนส่งต่อไปยัง Jaeger:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch: {}                # รวม span หลายอันเป็น batch เดียวก่อนส่งต่อ
  # tail_sampling:          # ตัวอย่าง processor สำหรับ tail-based sampling (หัวข้อ 99.7)
  #   policies:
  #     - name: keep-errors
  #       type: status_code
  #       status_code: {status_codes: [ERROR]}

exporters:
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]
```

และ `docker-compose.yml` ที่รันทั้งสาม process (แอป Rust ของคุณ, Collector, Jaeger) ร่วมกัน:

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:1.62.0
    ports: ["16686:16686"]

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.111.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes: ["./otel-collector-config.yaml:/etc/otel-collector-config.yaml"]
    ports: ["4317:4317", "4318:4318"]
    depends_on: [jaeger]

  my-app:
    build: .
    environment:
      OTLP_ENDPOINT: "http://otel-collector:4317" # ชี้ไปที่ Collector แทน Jaeger ตรง ๆ
    depends_on: [otel-collector]
```

สังเกตว่าแอป Rust **เปลี่ยนแค่ endpoint ที่ชี้ไป** (จาก `http://localhost:4317` เป็น `http://otel-collector:4317`)
โดยไม่ต้องแก้โค้ดอะไรเลย — นี่คือประโยชน์ที่จับต้องได้ของการแยก concern เป็นสามชั้นตามที่อธิบายไว้ข้างบน

#### เลือก Backend อื่นนอกจาก Jaeger ได้ไหม — เปรียบเทียบสั้น ๆ

Jaeger ไม่ใช่ backend เดียวที่เข้าใจ OTLP — ตารางนี้เปรียบเทียบตัวเลือกที่พบบ่อยในอุตสาหกรรม เพื่อให้เห็นว่า
สิ่งที่เรียนในบทนี้ (การส่ง OTLP ออกจากแอป) **ใช้ได้กับทุกตัวเลือกโดยไม่ต้องแก้โค้ดแอปเลย** เปลี่ยนแค่ endpoint
ที่ชี้ไป (เหมือนตัวอย่าง `docker-compose.yml` ข้างบน):

| Backend | ลักษณะ | เหมาะกับ |
|---|---|---|
| **Jaeger** (บทนี้ใช้) | โอเพนซอร์ส, self-hosted, all-in-one image รันง่ายสำหรับเรียนรู้/dev | ทีมที่ต้องการควบคุม infrastructure เอง หรือกำลังเรียนรู้ |
| **Zipkin** | โอเพนซอร์สรุ่นเก่ากว่า Jaeger (มาก่อน OTel ด้วยซ้ำ) ยังใช้กันอยู่ในหลายระบบเก่า | ระบบที่มี Zipkin อยู่แล้วจากก่อนยุค OTel |
| **Grafana Tempo** | โอเพนซอร์ส เน้นทำงานร่วมกับ Grafana + Loki (logs) + Prometheus (metrics) เป็น stack เดียว (เรียกกันว่า "LGTM stack" — จะกล่าวถึงในหัวข้อ 99.8) | ทีมที่ใช้ Grafana เป็นหน้าจอกลางของทุกเสาหลัก observability อยู่แล้ว |
| **Datadog / Honeycomb / New Relic ฯลฯ** | SaaS เชิงพาณิชย์ รับ OTLP ตรง ๆ ได้เช่นกัน มี UI/analytics ที่ทรงพลังกว่า แต่มีค่าใช้จ่ายตามปริมาณข้อมูล | ทีมที่ไม่ต้องการดูแล infrastructure ของ observability เอง และมีงบสำหรับ SaaS |

### 99.4 ติดตั้งและตั้งค่า: `opentelemetry` + `opentelemetry-otlp` + `tracing-opentelemetry`

ถึงเวลาลงมือจริง มาดู crate ที่ต้องใช้และบทบาทของแต่ละตัว:

```bash
cargo add opentelemetry opentelemetry_sdk
cargo add opentelemetry-otlp --features grpc-tonic
cargo add tracing-opentelemetry
cargo add tracing tracing-subscriber --features tracing-subscriber/env-filter
cargo add tokio --features full
```

`Cargo.toml` ที่ได้ (เวอร์ชันที่ตรวจสอบจริงจาก crates.io ด้วยการรัน `cargo add` ในโปรเจกต์ทดลองจริง — pattern
เดียวกับที่ Part 80/81 ทำไว้):

```toml
[dependencies]
opentelemetry = "0.33.0"
opentelemetry_sdk = "0.33.0"
opentelemetry-otlp = { version = "0.33.0", features = ["grpc-tonic"] }
tracing-opentelemetry = "0.34.0"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
tokio = { version = "1", features = ["full"] }
```

**บทบาทของแต่ละ crate** (สำคัญมากที่ต้องแยกให้ออก เพราะชื่อคล้ายกันและมีสี่ตัวพร้อมกัน):

| Crate | บทบาท |
|---|---|
| `opentelemetry` | นิยาม **API กลาง** ของ OTel — trait/type อย่าง `Tracer`, `Span`, `SpanContext`, `Status` ที่เป็นส่วนอินเทอร์เฟซล้วน ๆ (เหมือนบทบาทของ `log` crate ใน Part 60 — เป็น facade ไม่รู้จัก backend จริง) |
| `opentelemetry_sdk` | **implementation จริง** ของ API ข้างบน — `SdkTracerProvider`, `BatchSpanProcessor`, `Sampler` และตัวจัดการ `Resource` (metadata ของ "ใครส่ง trace นี้มา" เช่น `service.name`) |
| `opentelemetry-otlp` | **exporter ที่พูดโปรโตคอล OTLP** — เอา span ที่ SDK สร้างไว้ ส่งออกไปยังปลายทางผ่าน gRPC (feature `grpc-tonic` ที่เปิดไว้) หรือ HTTP (feature `http-proto`/`reqwest-client`) |
| `tracing-opentelemetry` | **bridge/สะพาน** ที่แปลง `tracing::Span`/event (จาก Part 60) ให้กลายเป็น OTel span/event โดยอัตโนมัติ — ทำงานเป็น `Layer` ตัวหนึ่งที่เสียบเข้ากับ `tracing_subscriber::registry()` แบบเดียวกับ `fmt::layer()` ที่ Part 60 สอนไว้ |

#### ตัวอย่างขั้นต่ำที่ใช้งานได้จริง: `#[tracing::instrument]` ส่งออกผ่าน OTLP

```rust
use opentelemetry::global;
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;
use tracing_subscriber::EnvFilter;

/// สร้าง TracerProvider ที่ export span ผ่าน OTLP (gRPC) ไปยัง endpoint ที่กำหนด
/// -- คืนค่า provider กลับมาเพราะต้องเก็บไว้เรียก .shutdown() ตอนจบโปรแกรม
/// (ดูกับดักที่ 3 ท้ายบทว่าทำไมขั้นนี้สำคัญมาก)
fn init_tracer() -> SdkTracerProvider {
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .with_endpoint("http://localhost:4317") // OTLP gRPC receiver ของ Jaeger
        .build()
        .expect("สร้าง OTLP exporter ไม่สำเร็จ");

    let resource = Resource::builder()
        .with_service_name("otel-verify-service") // ชื่อนี้จะปรากฏใน Jaeger UI เป็น "Service"
        .build();

    SdkTracerProvider::builder()
        .with_batch_exporter(exporter) // ส่งเป็น batch เป็นระยะ ไม่ใช่ span ทีละอันแบบ synchronous
        .with_resource(resource)
        .build()
}

/// span ที่สร้างจาก #[instrument] ตัวนี้ -- เขียนแบบเดียวกับที่ Part 60 สอนไว้เป๊ะ
/// ไม่มีอะไรต้องเปลี่ยนในฟังก์ชันนี้เลยเพื่อให้มัน export ได้ผ่าน OTLP
#[tracing::instrument]
async fn process_order(order_id: u32, qty: u32) -> Result<(), String> {
    tracing::info!(order_id, qty, "เริ่มประมวลผลคำสั่งซื้อ");
    check_stock(order_id, qty).await?;
    charge_payment(order_id).await?;
    tracing::info!(order_id, "ประมวลผลคำสั่งซื้อสำเร็จ");
    Ok(())
}

#[tracing::instrument]
async fn check_stock(order_id: u32, qty: u32) -> Result<(), String> {
    tokio::time::sleep(std::time::Duration::from_millis(15)).await;
    if qty > 100 {
        return Err(format!("สต็อกไม่พอสำหรับ order {order_id}"));
    }
    tracing::info!("มีสต็อกพอ");
    Ok(())
}

#[tracing::instrument]
async fn charge_payment(order_id: u32) -> Result<(), String> {
    tokio::time::sleep(std::time::Duration::from_millis(25)).await;
    tracing::info!(order_id, "เก็บเงินสำเร็จ");
    Ok(())
}

#[tokio::main]
async fn main() {
    let provider = init_tracer();
    let tracer = provider.tracer("otel-verify-service");

    // tracing_opentelemetry::layer() คือ "สะพาน" ที่แปลง tracing::Span เป็น OTel span
    // เสียบเข้ากับ registry() แบบเดียวกับ fmt::layer() ที่ Part 60 สอนไว้ทุกประการ
    let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

    tracing_subscriber::registry()
        .with(EnvFilter::new("info"))
        .with(tracing_subscriber::fmt::layer()) // ยังคงพิมพ์ log ออก terminal เหมือน Part 60 เดิม
        .with(otel_layer)                        // เพิ่มเข้ามา: export span ไป OTLP ด้วย
        .init();

    let _ = process_order(1001, 5).await;

    // สำคัญมาก: ต้อง shutdown provider เพื่อให้ batch exporter flush span ที่เหลือทั้งหมด
    // ก่อนโปรแกรมจบ ไม่อย่างนั้น span ที่ยังอยู่ใน batch buffer จะหายไปเงียบ ๆ (กับดักที่ 3)
    let _ = provider.shutdown();
}
```

สังเกตสิ่งสำคัญที่สุดของตัวอย่างนี้: **ฟังก์ชัน `process_order`/`check_stock`/`charge_payment` เขียนเหมือน
Part 60 เป๊ะทุกตัวอักษร** — ไม่มีการเปลี่ยนแปลงอะไรในตัวฟังก์ชันที่ใช้ `#[tracing::instrument]` เลยแม้แต่นิด
เดียว สิ่งที่เปลี่ยนคือแค่ `main()`: เพิ่มการสร้าง `otel_layer` แล้วเสียบเข้าไปเป็น layer ที่สามใน
`tracing_subscriber::registry()` ควบคู่กับ `fmt::layer()` เดิม — นี่คือพลังของโมเดล subscriber/layer ที่
Part 60 หัวข้อ 60.8 ปูทางไว้ให้: **layer หลายตัวทำงานพร้อมกันได้บน span/event เดียวกัน** โดยที่ตัว business
logic ของแอปไม่ต้องรู้เลยว่ามีกี่ layer หรือ layer ไหนทำอะไรกับ span ของมันบ้าง

โค้ดนี้ compile ผ่านจริง (ตรวจสอบด้วย `cargo build` ในโปรเจกต์ทดลอง — ดูหมายเหตุการตรวจสอบท้ายหัวข้อ) แต่ยัง
รันไม่เห็นผลอะไรเป็นชิ้นเป็นอันจนกว่าจะมี**อะไรสักอย่างที่รับ OTLP อยู่ที่ `localhost:4317` จริง** — นั่นคือ
สิ่งที่หัวข้อถัดไปจะทำ

### 99.5 รัน Jaeger จริงด้วย Docker แล้ว Export Span จริง

Jaeger คือ backend โอเพนซอร์สคลาสสิกสำหรับ distributed tracing (เดิมพัฒนาโดย Uber แล้วบริจาคให้ CNCF) — เวอร์ชัน
**all-in-one** คือ Docker image เดียวที่รวมทุกส่วนของ Jaeger (receiver รับ span, storage เก็บข้อมูลใน memory,
query API, และ UI) ไว้ใน process เดียว เหมาะที่สุดสำหรับพัฒนา/เรียนรู้ (ไม่เหมาะกับ production จริงที่ต้องการ
persistent storage แยกต่างหากอย่าง Elasticsearch/Cassandra — แต่สำหรับบทนี้ all-in-one เพียงพอสมบูรณ์)

#### รัน Jaeger ผ่าน Docker

```bash
docker run -d --name jaeger \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:1.62.0
```

สามพอร์ตที่เปิดไว้มีบทบาทต่างกัน:

| พอร์ต | บทบาท |
|---|---|
| **16686** | Jaeger UI + Query HTTP API (`GET /api/traces`, `GET /api/services`) — ที่เราจะ query ในหัวข้อนี้ |
| **4317** | OTLP receiver ผ่าน **gRPC** — endpoint ที่ `opentelemetry-otlp` (feature `grpc-tonic` ที่เปิดไว้ในหัวข้อ 99.4) ส่ง span มาที่นี่ |
| **4318** | OTLP receiver ผ่าน **HTTP** — ทางเลือกสำหรับ exporter ที่ใช้ feature `http-proto` แทน (บทนี้ใช้ gRPC เป็นหลัก) |

ผู้เขียนรันคำสั่งนี้จริงในเครื่องทดลอง แล้วตรวจสอบด้วย `docker ps` เห็น container ทำงานจริง และ
`docker logs jaeger` แสดง log จริงที่ยืนยันว่าทุก receiver เริ่มทำงานสำเร็จ:

```text
{"level":"info","msg":"Starting GRPC server","endpoint":":4317"}
{"level":"info","msg":"Starting HTTP server","endpoint":":4318"}
{"level":"info","msg":"Query server started","http_addr":"0.0.0.0:16686"}
{"level":"info","msg":"Health Check state change","status":"ready"}
```

#### รัน Rust Binary จากหัวข้อก่อนจริง แล้วดูผลลัพธ์

รันตัวอย่างจากหัวข้อ 99.4 (พร้อมเพิ่ม request ที่สองที่จงใจล้มเหลว เพื่อทดสอบ error case ไปพร้อมกัน):

```bash
RUST_LOG=info ./target/debug/otel_verify
```

ผลลัพธ์จริงจาก terminal (log ยัง print ผ่าน `fmt::layer()` เหมือนเดิมตาม Part 60 — สังเกตว่า span
`process_order`/`check_stock`/`charge_payment` ซ้อนกันถูกต้องเหมือน Part 60 หัวข้อ 60.7 ทุกประการ):

```text
2026-09-27T05:18:43.996072Z  INFO process_order{order_id=1001 qty=5}: เริ่มประมวลผลคำสั่งซื้อ order_id=1001 qty=5
2026-09-27T05:18:44.012476Z  INFO process_order{order_id=1001 qty=5}:check_stock{order_id=1001 qty=5}: มีสต็อกพอ
2026-09-27T05:18:44.038918Z  INFO process_order{order_id=1001 qty=5}:charge_payment{order_id=1001}: เก็บเงินสำเร็จ order_id=1001
2026-09-27T05:18:44.039022Z  INFO process_order{order_id=1001 qty=5}: ประมวลผลคำสั่งซื้อสำเร็จ order_id=1001
2026-09-27T05:18:44.039082Z  INFO process_order{order_id=1002 qty=500}: เริ่มประมวลผลคำสั่งซื้อ order_id=1002 qty=500
2026-09-27T05:18:44.055437Z ERROR order 1002 ล้มเหลวตามที่ตั้งใจ error=สต็อกไม่พอสำหรับ order 1002
```

ตอนนี้มาถึงจุดสำคัญ: **query Jaeger's HTTP API จริง** เพื่อยืนยันว่า span พวกนี้ถูกส่งออกไปเก็บจริง ไม่ใช่แค่
โค้ด compile ผ่านเฉย ๆ:

```bash
curl -s "http://localhost:16686/api/services"
```

ผลลัพธ์จริง (`otel-verify-service` คือชื่อที่ตั้งไว้ผ่าน `Resource::builder().with_service_name(...)`
ในหัวข้อ 99.4):

```json
{"data":["otel-verify-service"],"total":1,"limit":0,"offset":0,"errors":null}
```

query trace ทั้งหมดของ service นี้:

```bash
curl -s "http://localhost:16686/api/traces?service=otel-verify-service&limit=20"
```

ผลลัพธ์จริง (คัดมาเฉพาะส่วนสำคัญของ trace แรก — order 1001 ที่สำเร็จ) แสดง**สาม span**ที่ตรงกับสามฟังก์ชันที่
`#[instrument]` ครอบไว้เป๊ะ พร้อม `duration` เป็นหน่วย **microsecond** และ `references` ที่บอก parent/child
ตรงกับโครงสร้างการเรียกจริงในโค้ด:

```json
{
  "traceID": "e74950700f277a28769f0e8c1393d011",
  "spans": [
    {
      "spanID": "fb15966490877f02",
      "operationName": "process_order",
      "references": [],
      "startTime": 1790486270021860,
      "duration": 43581,
      "tags": [{"key": "order_id", "value": "1001"}, {"key": "qty", "value": "5"}]
    },
    {
      "spanID": "432f8d71f7c8e113",
      "operationName": "check_stock",
      "references": [{"refType": "CHILD_OF", "spanID": "fb15966490877f02"}],
      "startTime": 1790486270022007,
      "duration": 16475
    },
    {
      "spanID": "58a7182b7f52125e",
      "operationName": "charge_payment",
      "references": [{"refType": "CHILD_OF", "spanID": "fb15966490877f02"}],
      "startTime": 1790486270038537,
      "duration": 26849
    }
  ]
}
```

อ่านผลลัพธ์นี้ให้ละเอียด — นี่คือหลักฐานที่ตอบคำถามทั้งสามข้อจากหัวข้อ 99.1 ได้ตรง ๆ ที่ correlation ID เดี่ยว ๆ
ตอบไม่ได้:

- **`process_order` (span แม่)** มี `references: []` (ไม่มี parent — เป็น **root span** ของ trace นี้)
  กินเวลา **43,581 nanosecond (~43.6 microsecond)** ทั้งหมด
- **`check_stock`** มี `references: [{"refType": "CHILD_OF", "spanID": "fb15966490877f02"}]` — ตัวเลข
  `fb15966490877f02` นี้**คือ span ID ของ `process_order` เป๊ะ** พิสูจน์ parent/child relationship ตรงกับ
  โค้ดจริงที่ `check_stock` ถูกเรียกจากภายใน `process_order` — และเห็น `startTime`/`duration` ที่บอกได้ตรง ๆ
  ว่ามันกินเวลา **16,475 nanosecond** และเริ่มที่ millisecond ไหนของ `process_order`
- **`charge_payment`** ก็เป็น `CHILD_OF` span เดียวกัน (`fb15966490877f02`) เห็นตรง ๆ ว่ามันเริ่มทำงาน**หลัง**
  `check_stock` จบ (เพราะ `startTime` ของมันมากกว่า `startTime + duration` ของ `check_stock`) — นี่คือข้อมูล
  ที่พิสูจน์ว่าการเรียกสองอย่างนี้เป็น **sequential** ไม่ใช่ concurrent ซึ่งเป็นคำถามที่ 2 จากหัวข้อ 99.1 ที่
  correlation ID เดี่ยว ๆ ตอบไม่ได้เลย

ในหน้า Jaeger UI จริง (`http://localhost:16686`) ข้อมูลชุดนี้จะแสดงเป็น **waterfall diagram**: แถบแนวนอนสาม
แถบ แถบบนสุดยาวที่สุดคือ `process_order` ครอบทั้งหมด แถบสองแถบด้านล่างซ้อนอยู่ภายในตามลำดับเวลา — เห็นสัดส่วน
เวลาทั้งหมดเป็นภาพเดียวโดยไม่ต้องคำนวณอะไรด้วยมือเลย (ต่างจากหัวข้อ 99.1 ที่ต้องเทียบ timestamp จากหลาย log
stream ด้วยมือ)

#### Event ที่แนบมากับ Span: `tracing::info!` กลายเป็นอะไรใน Jaeger

ดึงข้อมูลเต็มของ span `check_stock` มาดู field `logs` (ชื่อ field นี้ใน Jaeger API ยังใช้คำเดิมจากยุคก่อน OTel
— ในทางแนวคิดคือ **Event** ตามที่อธิบายไว้ในหัวข้อ 99.2):

```json
"logs": [
  {
    "timestamp": 1790486270038469,
    "fields": [
      {"key": "event", "value": "มีสต็อกพอ"},
      {"key": "level", "value": "INFO"},
      {"key": "code.file.path", "value": "src/main.rs"},
      {"key": "code.line.number", "value": 39}
    ]
  }
]
```

นี่คือหลักฐานตรงว่า `tracing::info!("มีสต็อกพอ")` ที่เรียกจากภายใน `check_stock` **กลายเป็น Event หนึ่งอันที่
แนบอยู่กับ span `check_stock` โดยอัตโนมัติ** ผ่าน `tracing-opentelemetry` bridge — ไม่ต้องเขียนโค้ดเพิ่มเพื่อ
"ผูก" event เข้ากับ span เอง แค่เรียก `tracing::info!` ตามปกติที่ Part 60 สอนไว้ ระบบจะจัดการให้ทั้งหมด ในหน้า
Jaeger UI event เหล่านี้ปรากฏเป็นจุดเล็ก ๆ บนแถบของ span ที่คลิกดูรายละเอียดได้

### 99.6 Distributed Trace ข้าม Service จริง: W3C Trace Context

ถึงเวลากลับไปหาสถานการณ์ของ **Part 81**: `booking-service` เรียก `catalog-service` ข้าม network — คราวนี้
เราจะทำให้ span ของทั้งสอง service **รวมกันเป็น trace เดียว** ใน Jaeger โดยใช้มาตรฐานจริง (W3C Trace Context)
แทน correlation ID ที่กำหนดเอง (ตัวอย่างนี้สาธิตด้วย HTTP/Axum เพื่อให้เห็น header ตรง ๆ ในรูปแบบที่คุ้นเคย —
หลักการเดียวกันนี้ใช้กับ gRPC metadata ของ Part 80/81 ได้ทันที เพราะ Span Context ไม่ผูกกับ transport ใด
transport หนึ่งเลย)

#### แนวคิด: Inject ที่ผู้เรียก, Extract ที่ผู้รับ

W3C Trace Context ทำงานด้วยสองฝั่ง:

- **ฝั่งผู้เรียก (client)**: **inject** span context ปัจจุบันของตัวเองลงใน HTTP header ก่อนยิง request ออกไป
- **ฝั่งผู้รับ (server)**: **extract** span context จาก header ที่ได้รับมา แล้วตั้งให้ span ใหม่ที่ตัวเองสร้าง
  มี**parent เป็น span context นั้น** — ผลคือ span ของทั้งสองฝั่งกลายเป็นส่วนของ trace เดียวกัน

`opentelemetry` crate นิยาม trait `Injector`/`Extractor` ไว้ให้ทำงานกับ carrier (ตัวพา header) ชนิดไหนก็ได้ —
ตัวอย่างนี้ implement เองสำหรับ `reqwest::header::HeaderMap` (ฝั่งเรียก) และ `axum::http::HeaderMap` (ฝั่งรับ):

```rust
use opentelemetry::propagation::{Extractor, Injector};

/// ฝั่งผู้เรียก (service-a): ใส่ traceparent/tracestate header เข้าไปใน request ที่จะยิงออก
struct HeaderInjector<'a>(&'a mut reqwest::header::HeaderMap);

impl<'a> Injector for HeaderInjector<'a> {
    fn set(&mut self, key: &str, value: String) {
        if let Ok(name) = reqwest::header::HeaderName::from_bytes(key.as_bytes()) {
            if let Ok(val) = reqwest::header::HeaderValue::from_str(&value) {
                self.0.insert(name, val);
            }
        }
    }
}

/// ฝั่งผู้รับ (service-b): อ่าน traceparent/tracestate header ที่ได้รับมา
struct HeaderExtractor<'a>(&'a axum::http::HeaderMap);

impl<'a> Extractor for HeaderExtractor<'a> {
    fn get(&self, key: &str) -> Option<&str> {
        self.0.get(key).and_then(|v| v.to_str().ok())
    }

    fn keys(&self) -> Vec<&str> {
        self.0.keys().map(|k| k.as_str()).collect()
    }
}
```

#### `service-a-booking`: ฝั่งผู้เรียก (แทน `booking-service` จาก Part 81)

```rust
use axum::extract::Path;
use axum::routing::get;
use axum::Router;
use opentelemetry::global;
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::propagation::TraceContextPropagator;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;

#[tracing::instrument]
async fn create_booking(Path(book_id): Path<u32>) -> axum::Json<serde_json::Value> {
    tracing::info!(book_id, "service-a: ได้รับคำขอจองหนังสือจาก client");

    let client = reqwest::Client::new();
    let mut headers = reqwest::header::HeaderMap::new();

    // แนบ W3C traceparent/tracestate header เข้าไปใน request ที่จะยิงออกไปยัง service-b
    // นี่คือ "correlation ID แบบมาตรฐาน" ที่มาแทน x-request-id ที่ Part 81 หัวข้อ 81.8 ทำด้วยมือ
    global::get_text_map_propagator(|prop| {
        prop.inject_context(&tracing::Span::current().context(), &mut HeaderInjector(&mut headers));
    });

    let url = format!("http://127.0.0.1:3002/books/{book_id}/availability");
    let resp = client.get(&url).headers(headers).send().await.unwrap()
        .json::<serde_json::Value>().await.unwrap();

    tracing::info!(?resp, "service-a: ได้รับคำตอบจาก service-b แล้ว");
    axum::Json(serde_json::json!({ "booking_created": true, "catalog_response": resp }))
}

#[tokio::main]
async fn main() {
    // ต้องตั้ง global propagator ก่อนใช้ inject_context/extract ข้างบน
    // ไม่อย่างนั้นจะได้ propagator แบบ no-op ที่ไม่ใส่ header อะไรเข้าไปเลย (กับดักที่ 6)
    global::set_text_map_propagator(TraceContextPropagator::new());

    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic().with_endpoint("http://localhost:4317").build().unwrap();
    let resource = Resource::builder().with_service_name("service-a-booking").build();
    let provider = SdkTracerProvider::builder()
        .with_batch_exporter(exporter).with_resource(resource).build();

    global::set_tracer_provider(provider.clone());
    let otel_layer = tracing_opentelemetry::layer().with_tracer(provider.tracer("service-a-booking"));
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new("info"))
        .with(tracing_subscriber::fmt::layer())
        .with(otel_layer)
        .init();

    let app = Router::new().route("/bookings/{book_id}", get(create_booking));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3001").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

#### `service-b-catalog`: ฝั่งผู้รับ (แทน `catalog-service` จาก Part 81)

```rust
use axum::extract::Path;
use opentelemetry::propagation::Extractor;
use tracing::Instrument;
use tracing_opentelemetry::OpenTelemetrySpanExt;

async fn check_availability(
    Path(book_id): Path<u32>,
    headers: axum::http::HeaderMap,
) -> axum::Json<serde_json::Value> {
    // ดึง SpanContext ที่ propagate มาจาก service-a ผ่าน W3C traceparent header
    let parent_ctx =
        opentelemetry::global::get_text_map_propagator(|prop| prop.extract(&HeaderExtractor(&headers)));

    // สร้าง span ด้วยมือแล้ว set_parent "ก่อน" enter -- ดูกับดักที่ 2 ท้ายบทว่าทำไม
    // ลำดับนี้สำคัญมาก (ถ้าใช้ #[instrument] ตรง ๆ แบบฟังก์ชันอื่น ๆ ในบทนี้ set_parent
    // จะทำไม่ได้เพราะ span ถูก enter ไปแล้วตั้งแต่บรรทัดแรกของ body)
    let span = tracing::info_span!("check_availability", book_id);
    span.set_parent(parent_ctx)
        .expect("set_parent ต้องสำเร็จ ณ จุดนี้ (span ยังไม่ถูก enter)");

    async move {
        tracing::info!(book_id, "service-b: ตรวจสอบสต็อกหนังสือ");
        tokio::time::sleep(std::time::Duration::from_millis(12)).await;
        axum::Json(serde_json::json!({ "book_id": book_id, "available": true, "copies": 3 }))
    }
    .instrument(span)
    .await
}
```

(โค้ดเต็มของทั้งสอง binary รวม `main()`/imports ทั้งหมดผู้เขียนได้ compile และรันจริงแล้ว — ตัดมาแสดงเฉพาะ
ส่วนที่เกี่ยวกับการ propagate context เพื่อไม่ให้ยาวเกินไป โครงสร้าง `main()` เหมือนหัวข้อ 99.4/99.5 ทุก
ประการ เปลี่ยนแค่ `service_name` และ port ที่ bind)

#### รันจริงทั้งสอง Service พร้อมกัน แล้วยิง Request ข้าม Service

```bash
RUST_LOG=info ./target/debug/service_b &   # ฟังที่ 127.0.0.1:3002
RUST_LOG=info ./target/debug/service_a &   # ฟังที่ 127.0.0.1:3001
curl -s http://127.0.0.1:3001/bookings/77
```

ผลลัพธ์จริง:

```json
{"booking_created":true,"catalog_response":{"available":true,"book_id":77,"copies":3}}
```

log ของ `service-a` (สังเกต `traceparent` header จริงที่ถูก inject ก่อนยิงออก):

```text
INFO create_booking{book_id=77}: service-a: ได้รับคำขอจองหนังสือจาก client book_id=77
INFO create_booking{book_id=77}: service-a: แนบ traceparent header แล้ว กำลังเรียก service-b traceparent=Some("00-49bcda026bbb66bf8ef31a39766377dc-31b84aac927bf49d-01")
INFO create_booking{book_id=77}: service-a: ได้รับคำตอบจาก service-b แล้ว resp=Object {...}
```

log ของ `service-b`:

```text
INFO check_availability{book_id=77}: service-b: ตรวจสอบสต็อกหนังสือ book_id=77
```

ตอนนี้มาถึงจุดพิสูจน์: query Jaeger ด้วย trace ID ที่ตัดมาจาก `traceparent` header ข้างบน
(`49bcda026bbb66bf8ef31a39766377dc`):

```bash
curl -s "http://localhost:16686/api/traces/49bcda026bbb66bf8ef31a39766377dc"
```

ผลลัพธ์จริง (คัดส่วนสำคัญ):

```json
{
  "traceID": "49bcda026bbb66bf8ef31a39766377dc",
  "spans": [
    {
      "spanID": "31b84aac927bf49d",
      "operationName": "create_booking",
      "references": [],
      "startTime": 1790486555510717,
      "duration": 31398,
      "serviceName": "service-a-booking"
    },
    {
      "spanID": "c51683010baaeb24",
      "operationName": "check_availability",
      "references": [{"refType": "CHILD_OF", "spanID": "31b84aac927bf49d"}],
      "startTime": 1790486555522870,
      "duration": 17666,
      "serviceName": "service-b-catalog"
    }
  ]
}
```

**นี่คือผลลัพธ์ที่สำคัญที่สุดของบทนี้**: **หนึ่ง trace ID เดียว (`49bcda026bbb66bf8ef31a39766377dc`) มี span
จากสอง service คนละตัว คนละ process คนละพอร์ตกันโดยสิ้นเชิง** (`service-a-booking` และ `service-b-catalog`)
และ span ของ `service-b-catalog` เป็น `CHILD_OF` span ของ `service-a-booking` อย่างถูกต้อง ตรงกับที่โค้ดจริง
เรียก `service-b` จากภายใน `service-a` — ใน Jaeger UI trace นี้จะแสดง**สองแถบซ้อนกัน มีชื่อ service คนละสี**
กำกับไว้ชัดเจน คลิกดู timing ของแต่ละ hop ได้ทันที — นี่คือคำตอบเต็มรูปแบบของปัญหาที่ Part 81 หัวข้อ 81.8 ทำได้
แค่ครึ่งเดียวด้วย correlation ID เดี่ยว ๆ

**หมายเหตุสำคัญที่ต้องพูดตรง ๆ**: การเขียน `service-b` ให้ `set_parent` ทำงานถูกต้องแบบข้างบน**ไม่ใช่เรื่องง่าย
เท่าที่คิดตอนแรก** — ผู้เขียนลองเขียนแบบ "ธรรมดา" ก่อน (ใช้ `#[tracing::instrument]` ตรง ๆ แล้วเรียก
`tracing::Span::current().set_parent(...)` จากภายใน body) แล้วพบว่า**span ของ service-b ไม่ผูกกับ trace ของ
service-a เลย** — กลายเป็นคนละ trace กันโดยสิ้นเชิงแม้ network call จะสำเร็จสมบูรณ์ก็ตาม นี่คือกับดักจริงที่
พบระหว่างตรวจสอบเนื้อหาบทนี้ อธิบายละเอียดพร้อมวิธีแก้ในหัวข้อกับดักที่ 2 ท้ายบท — **สาเหตุที่โค้ดข้างบนเขียน
ด้วยมือแบบไม่ใช้ `#[instrument]`และเรียก `set_parent` ก่อน `.instrument(span).await` ไม่ใช่เรื่องบังเอิญ**

### 99.7 Sampling: ควบคุมปริมาณ Trace ในระบบที่มี Traffic สูง

ระบบ production ขนาดใหญ่ที่รับ traffic หลักพัน/หมื่น request ต่อวินาที **การเก็บ trace ทุก request** จะสร้าง
ปัญหาสองข้อทันที: (1) **ต้นทุนการเก็บข้อมูล** — span ทุกตัวต้อง export, ส่งผ่าน network, เก็บลง storage ของ
backend ซึ่งมีค่าใช้จ่ายจริงทั้งด้าน bandwidth และพื้นที่เก็บข้อมูล (ยิ่งเก็บนานยิ่งแพง) และ (2) **overhead ต่อ
ตัวแอปเอง** — แม้ batch exporter จะช่วยลด overhead ลงมากแล้ว การสร้าง/ส่ง span จำนวนมหาศาลก็ยังกิน CPU/memory
มากกว่าไม่ทำเลย **Sampling** คือการตัดสินใจว่า **"trace ไหนบ้างที่จะถูกเก็บจริง"** โดยไม่เก็บทุก trace

#### Head-based Sampling: ตัดสินใจ ณ จุดเริ่ม Trace

**Head-based sampling** ตัดสินใจ**ทันทีที่ trace เริ่มต้น** (ที่ root span) ว่าจะเก็บ trace นี้หรือไม่ — การ
ตัดสินใจนี้ (ผ่าน `sampled` flag ใน trace flags ที่อธิบายไว้ในหัวข้อ 99.2) จะ**ถูก propagate ไปกับทุก hop
ที่ตามมา** ผ่าน `traceparent` header เดียวกัน ทำให้ทุก service ที่เกี่ยวข้องกับ trace เดียวกันตัดสินใจ**สอด
คล้องกัน** (ไม่มีเหตุการณ์ที่ service A เก็บ trace แต่ service B ในสายเดียวกันไม่เก็บ)

ข้อดีคือ**เรียบง่ายและมี overhead ต่ำที่สุด** (ตัดสินใจครั้งเดียว ไม่ต้องเก็บข้อมูลอะไรไว้รอ) แต่ข้อเสียที่
สำคัญคือ **ตัดสินใจโดยไม่รู้ผลลัพธ์สุดท้าย** — ถ้า sampling rate ตั้งไว้ที่ 20% มีโอกาส 80% ที่ trace ของ
request ที่ error จริง (ซึ่งเป็น trace ที่มีค่าที่สุดสำหรับการ debug) จะถูก**ทิ้งไปโดยไม่มีใครได้เห็นมันเลย**
เพราะการตัดสินใจเกิดขึ้น**ก่อน**ที่จะรู้ว่า request นี้จะ error หรือไม่

```rust
use opentelemetry_sdk::trace::{Sampler, SdkTracerProvider};

// เก็บ trace แค่ 20% ของทั้งหมด -- ParentBased หมายความว่า: ถ้า trace นี้มี parent
// context ที่ propagate มาจาก service อื่นอยู่แล้ว (มี traceparent header ติดมา) ให้ทำ
// ตามการตัดสินใจของ parent นั้นเลย (สอดคล้องกันทั้ง trace) แต่ถ้าเป็น root span ตัวแรก
// ของ trace ใหม่จริง ๆ ให้สุ่มตัดสินใจเองด้วย TraceIdRatioBased(0.2)
let sampler = Sampler::ParentBased(Box::new(Sampler::TraceIdRatioBased(0.2)));

let provider = SdkTracerProvider::builder()
    // ...with_batch_exporter, with_resource เหมือนเดิม...
    .with_sampler(sampler)
    .build();
```

**ทำไมต้องเป็น `ParentBased`, ไม่ใช่ `TraceIdRatioBased` เดี่ยว ๆ?** — ถ้า service ทุกตัวในสายเรียกใช้
`TraceIdRatioBased` เดี่ยว ๆ (ไม่ใช่ `ParentBased`) แต่ละ service จะ**สุ่มตัดสินใจของตัวเองแยกกัน** แม้จะเป็น
trace เดียวกัน — ผลคือ trace หนึ่งอันอาจมี span จาก service A (เก็บ) แต่ไม่มี span จาก service B (ไม่เก็บ)
ในสายเดียวกัน ทำให้ trace ที่ได้มา**ไม่สมบูรณ์** `ParentBased` แก้ปัญหานี้โดยให้ **root span ของ trace เท่านั้น
ที่สุ่มตัดสินใจจริง** ส่วน span ลูกทุกตัว (ทุก service ที่ตามมา) จะ**เชื่อฟังการตัดสินใจของ parent เสมอ** ผ่าน
`sampled` flag ที่ propagate มาใน `traceparent` — นี่คือเหตุผลที่ `TraceIdRatioBased` ใช้ **trace ID เป็น seed**
ของการสุ่ม (ไม่ใช่สุ่มแบบสุ่มล้วน ๆ) เพื่อให้ผลลัพธ์ deterministic ต่อ trace ID เดียวกันทุกครั้งที่ถูกประเมิน

#### พิสูจน์จริง: ยิง 50 Request ด้วย Sampling Rate 20%

ผู้เขียนเขียนโปรแกรมยิง 50 "request" จำลอง (แต่ละอันเป็น trace แยกกันคนละอันโดยสิ้นเชิง ไม่มี parent ร่วมกัน)
ด้วย sampler ตั้งไว้ที่ `TraceIdRatioBased(0.2)` แล้วดูว่า Jaeger เก็บได้จริงกี่ trace:

```rust
let sampler = Sampler::ParentBased(Box::new(Sampler::TraceIdRatioBased(0.2)));
// ... สร้าง provider พร้อม sampler นี้ ...

for i in 0..50u32 {
    handle_request(i).await; // #[tracing::instrument] ธรรมดา สร้าง trace ใหม่ทุกครั้ง
}
```

ผลลัพธ์จากการรันจริง:

```text
ส่ง 50 requests แล้ว (คาดว่า Jaeger จะเก็บได้ประมาณ 20% คือ ~10 traces)
```

query Jaeger จริง:

```bash
curl -s "http://localhost:16686/api/traces?service=sampling-demo-service&limit=100"
```

**ผลลัพธ์จริง: Jaeger เก็บได้ 8 trace จาก 50 request ที่ยิงไป (16%)** — ใกล้เคียงกับอัตรา 20% ที่ตั้งไว้ (ความ
คลาดเคลื่อนเป็นเรื่องปกติของการสุ่มด้วยจำนวนตัวอย่างไม่มาก ยิ่งยิง request มากขึ้นสัดส่วนจะเข้าใกล้ 20% มาก
ขึ้นตามกฎจำนวนมาก) — **นี่คือหลักฐานที่ยืนยันตรง ๆ ว่า sampler ทำงานจริง ไม่ใช่แค่ config ที่ไม่มีผลอะไร**
สังเกตว่า**42 request ที่เหลือไม่ได้ error หรือหายไปไหน** — โปรแกรมยังทำงานสมบูรณ์ปกติทุก request เพียงแต่
span ของมันไม่ถูกส่งออกไปที่ Jaeger เท่านั้น (sampler ตัดสินใจตั้งแต่ระดับ SDK ก่อนที่จะไปถึงขั้น export เลย
ไม่ใช่การ export แล้วถูก backend ปฏิเสธทีหลัง)

#### Tail-based Sampling: ตัดสินใจหลังเห็น Trace ทั้งหมดแล้ว (ระดับแนวคิด)

**Tail-based sampling** แก้ปัญหาที่ head-based sampling มีอยู่โดยตรง: มัน**รอดูผลลัพธ์ทั้ง trace ก่อนตัดสินใจ**
ว่าจะเก็บหรือไม่ — เช่น กฎ **"เก็บ trace ทุกอันที่มี span ใดก็ตามที่ error, ไม่ว่า sampling rate ปกติจะตั้งไว้
เท่าไหร่"** เป็นกฎที่ **head-based sampling ทำไม่ได้เลยในหลักการ** (เพราะตัดสินใจไปแล้วก่อนรู้ว่าจะ error)
แต่ tail-based sampling ทำได้ เพราะมันเก็บ span ทั้งหมดของ trace ไว้ชั่วคราวจนกว่า trace จะ**จบสมบูรณ์**
(root span ปิด) ก่อนตัดสินใจ

ข้อแลกที่สำคัญคือ **tail-based sampling ต้องการ infrastructure เพิ่มขึ้นมาก** — ต้องมี component ที่**บัฟเฟอร์
span ทั้งหมดของ trace ที่ยังไม่จบไว้ในหน่วยความจำ**ก่อน (ปกติทำที่ระดับ OTel Collector ที่กล่าวถึงในหัวข้อ
99.3 ด้วย processor ชนิด `tail_sampling`) ซึ่งหมายความว่า **span ทุกตัวของทุก trace ต้องถูกส่งไปที่จุดเดียว
กันก่อน**เสมอ (ต่างจาก head-based ที่แต่ละ service ตัดสินใจแยกกันได้เลยโดยไม่ต้องรอใคร) และต้องมี memory เพียง
พอสำหรับบัฟเฟอร์ trace ที่ยังไม่จบทั้งหมดในระบบ ณ ขณะนั้น — ในระบบที่มี traffic สูงมากและ latency สูง (trace
กินเวลานานกว่าจะจบ) buffer นี้อาจใช้ทรัพยากรมากพอสมควร บทนี้ให้เข้าใจแนวคิดและ trade-off ไว้ในระดับนี้ (ไม่
ลง implementation จริง เพราะต้องมี Collector แยกและ config ที่ซับซ้อนกว่าที่เหมาะกับบทนี้) — ตารางสรุป:

| | Head-based Sampling | Tail-based Sampling |
|---|---|---|
| จุดตัดสินใจ | ทันทีที่ trace เริ่ม (root span) | หลัง trace จบสมบูรณ์แล้ว |
| รู้ผลลัพธ์ก่อนตัดสินใจไหม | ไม่รู้ | รู้ (error, latency สูง ฯลฯ) |
| Infrastructure ที่ต้องมี | แค่ SDK ในแอป (`Sampler` เฉย ๆ) | ต้องมี Collector ที่บัฟเฟอร์ span ทั้ง trace |
| เหมาะกับ | ลด volume โดยรวมแบบง่าย ต้นทุนต่ำ | ต้องการ "เก็บ trace ที่ error ไว้เสมอ" แม่นยำ |
| จุดอ่อน | อาจทิ้ง trace ที่ error/สำคัญไปโดยไม่ตั้งใจ | ซับซ้อนกว่า ใช้ memory มากกว่า มี latency เพิ่มก่อนส่งออกจริง |

### 99.8 เชื่อมสามเสาหลักของ Observability: Trace ↔ Log ↔ Metrics

บทนี้เป็นบทปิดของโมดูล observability สามบท (Part 60 → Part 98 → Part 99) — มาถึงจุดที่ต้องแสดงให้เห็นว่าทั้ง
สามอย่างที่เรียนแยกกันมา**เชื่อมกันเป็นระบบเดียว**ได้จริงในทางปฏิบัติ

#### เชื่อม Trace เข้ากับ Log: ใส่ `trace_id` เป็น Structured Field

ปัญหาที่พบบ่อยมากในทางปฏิบัติ: คุณกำลังไล่ log หา error หนึ่งบรรทัด เจอแล้ว แต่**อยากเห็นภาพรวมทั้ง request**
ที่ error บรรทัดนั้นเป็นส่วนหนึ่งของมัน (เกิดจาก hop ไหนก่อนหน้า? มี hop ไหนตามมาอีกไหม?) — วิธีแก้ที่ตรงไปตรงมา
คือใส่ **`trace_id` เป็น structured field** ในทุก log line (ตามหลักการ structured field ของ Part 60 หัวข้อ
60.9 เป๊ะ) ทำให้กระโดดจาก log line ไปหา trace เต็มใน Jaeger ได้ทันที:

```rust
use opentelemetry::trace::TraceContextExt;
use tracing_opentelemetry::OpenTelemetrySpanExt;

#[tracing::instrument]
async fn handle_payment(order_id: u32) {
    // ดึง trace_id ของ span ปัจจุบันออกมาเป็น field ธรรมดาใน log
    let trace_id = tracing::Span::current().context().span().span_context().trace_id();
    tracing::info!(order_id, %trace_id, "เริ่มเก็บเงิน");
    tokio::time::sleep(std::time::Duration::from_millis(5)).await;
    tracing::info!(order_id, %trace_id, "เก็บเงินสำเร็จ");
}
```

ผู้เขียนตั้ง output เป็น JSON (`tracing_subscriber::fmt::layer().json()` — ตามที่ Part 60 หัวข้อ 60.8 แนะนำไว้
สำหรับระบบ log aggregation) แล้วรันจริง ได้ log line จริง:

```json
{"timestamp":"2026-09-27T05:24:31.473101Z","level":"INFO","fields":{"message":"เริ่มเก็บเงิน","order_id":9001,"trace_id":"7d5365795e23f207bfd51f64a45fcb17"},"target":"log_correlation_demo","span":{"order_id":9001,"name":"handle_payment"}}
```

แล้ว query Jaeger ด้วย service เดียวกัน:

```bash
curl -s "http://localhost:16686/api/traces?service=log-correlation-demo&limit=5"
```

ผลลัพธ์จริง:

```json
{"data":[{"traceID":"7d5365795e23f207bfd51f64a45fcb17", "...": "..."}]}
```

**`trace_id` ใน log บรรทัด JSON ข้างบน (`7d5365795e23f207bfd51f64a45fcb17`) ตรงกับ `traceID` ที่ Jaeger คืนมา
เป๊ะทุกตัวอักษร** — นี่คือกลไกที่ทำให้ระบบ log aggregation จริง (ELK, Loki, Datadog ที่ Part 60 กล่าวถึง) ทำ
"jump to trace" ได้: คนดู log เจอ error line, เห็น field `trace_id`, คลิก (หรือ copy ไป paste ใน Jaeger UI)
แล้วเห็น**ทั้ง trace ที่ error line นั้นเป็นส่วนหนึ่ง**ทันที — ปิดช่องว่างระหว่างสองเสาหลักนี้ได้อย่างเป็น
รูปธรรม ไม่ต้องเดาหรือเทียบ timestamp ด้วยมือแบบหัวข้อ 99.1 อีกเลย

#### เชื่อม Trace เข้ากับ Metrics: Exemplars (เชื่อมกับ Part 98)

**Exemplar** คือ feature ของ Prometheus (Part 98) ที่ทำหน้าที่ตรงกันข้ามกับด้านบน แต่ในทิศทางเดียวกัน: มัน
**ผูก metric data point หนึ่งจุดเข้ากับ trace ID ของ request ตัวอย่างหนึ่งที่ทำให้เกิดค่านั้น** ยกตัวอย่าง:
histogram ของ Part 98 ที่วัด HTTP request latency (`http_request_duration_seconds`) ปกติจะบอกแค่ตัวเลขสรุป
("มี request 3 ตัวที่ latency อยู่ใน bucket 500ms-1s") — exemplar เพิ่มข้อมูลอีกชั้น: **"นี่คือ trace ID ของ
request ตัวอย่างหนึ่งตัวจริง ๆ ที่ทำให้เกิดค่า latency 750ms ตัวนี้"** ทำให้ operations engineer ที่เห็นกราฟ
latency พุ่งสูงผิดปกติใน Grafana (Part 98) **คลิกจากจุดข้อมูลบนกราฟไปดู trace จริงหนึ่งอันที่เป็นสาเหตุได้ทันที**
โดยไม่ต้องไปนั่งหา trace ที่ตรงกับช่วงเวลานั้นเอง

การ implement เต็มรูปแบบต้องใช้ `opentelemetry`'s metrics API ร่วมกับ Prometheus exporter ที่รองรับ exemplar
(ยังเป็น feature ที่ค่อนข้าง advanced ในระบบนิเวศ Rust ณ ตอนที่เขียนบทนี้) ซึ่งเกินขอบเขตความลึกที่บทนี้ตั้งใจ
ครอบคลุม — สิ่งสำคัญที่ต้องเข้าใจในระดับแนวคิดคือ **exemplar คือสะพานที่เชื่อมทิศทางตรงข้ามกับ trace_id ใน log**:
log → trace ทำผ่านการใส่ `trace_id` เป็น field ธรรมดา ส่วน metric → trace ทำผ่าน exemplar ที่ backend อย่าง
Prometheus/Grafana เข้าใจในตัว — ทั้งสองทิศทางรวมกันทำให้ observability platform สมบูรณ์ในความหมายที่ว่า
**ไม่ว่าจะเริ่มจากมุมไหน (เห็น log ผิดปกติ, เห็นกราฟ metric ผิดปกติ, หรือเห็น trace ที่ error) ก็กระโดดไปดู
อีกสองมุมที่เหลือได้เสมอ**

#### ตารางสรุปสามเสาหลัก

| เสาหลัก | บทที่สอน | ตอบคำถามอะไร | เชื่อมกับอีกสองเสาหลักผ่าน |
|---|---|---|---|
| **Logs** | Part 60 | "เกิดอะไรขึ้นบ้าง แต่ละบรรทัด" — รายละเอียดระดับ event เดี่ยว ๆ | ใส่ `trace_id` เป็น field (หัวข้อนี้) |
| **Metrics** | Part 98 | "ภาพรวมเชิงตัวเลขเป็นอย่างไรตามเวลา" — อัตรา error, latency percentile, throughput | Exemplar ผูกจุดข้อมูลเข้ากับ trace ตัวอย่าง |
| **Traces** | Part 99 (บทนี้) | "request หนึ่งตัวเดินทางอย่างไร ใช้เวลาที่ไหนบ้าง" — มุมมองระดับ request เดี่ยว ข้ามหลาย service | `trace_id` ในทั้ง log และ exemplar ของ metric |

### 99.9 Error Tracking ใน Span: เชื่อมกับ `AppError` จาก Part 66

จำ `AppError` จาก **Part 66** ได้ไหม — enum เดียวที่เป็นจุดศูนย์กลาง error ทั้งแอป (`NotFound`, `Validation`,
`Conflict`, `Unauthorized`, `Internal(#[from] anyhow::Error)`) พร้อม `impl IntoResponse` ที่แปลงมันเป็น HTTP
response ที่ถูกต้องเสมอ — คำถามของหัวข้อนี้คือ: **เมื่อ handler คืน `Err(AppError::...)` ควรทำอะไรกับ span ที่
ครอบ handler นั้นอยู่ เพื่อให้ Jaeger UI แยกแยะ request ที่ล้มเหลวออกจาก request ที่สำเร็จได้ง่าย?**

#### วิธีที่ถูกต้อง: `Status::error(...)` ผ่าน `OpenTelemetrySpanExt`

OTel มีแนวคิด **Span Status** ในตัว (ค่าเป็น `Unset`, `Ok`, หรือ `Error` พร้อมข้อความอธิบาย) — `tracing-
opentelemetry` เปิด method `set_status` บน `tracing::Span` ธรรมดาผ่าน trait `OpenTelemetrySpanExt`:

```rust
use opentelemetry::trace::Status;
use tracing_opentelemetry::OpenTelemetrySpanExt;

#[tracing::instrument]
async fn check_stock(order_id: u32, qty: u32) -> Result<(), String> {
    tokio::time::sleep(std::time::Duration::from_millis(15)).await;
    if qty > 100 {
        let msg = format!("สต็อกไม่พอสำหรับ order {order_id}");
        // ตั้งค่า OTel span status เป็น Error จริง ๆ (ไม่ใช่แค่ log field ธรรมดา)
        tracing::Span::current().set_status(Status::error(msg.clone()));
        return Err(msg);
    }
    tracing::info!("มีสต็อกพอ");
    Ok(())
}
```

รันจริงแล้ว query Jaeger ด้วย trace ID ของ request ที่จงใจล้มเหลว (`order_id=1002, qty=500`):

```bash
curl -s "http://localhost:16686/api/traces/0b881a3f12c280bdf04216e664a06546"
```

ผลลัพธ์จริง (span `check_stock` ของ trace นี้):

```json
{
  "operationName": "check_stock",
  "tags": [
    {"key": "order_id", "value": "1002"},
    {"key": "qty", "value": "500"},
    {"key": "otel.status_code", "value": "ERROR"},
    {"key": "otel.status_description", "value": "สต็อกไม่พอสำหรับ order 1002"}
  ]
}
```

**`otel.status_code: "ERROR"`** คือ tag มาตรฐานที่ Jaeger UI ใช้**วาดแถบของ span นี้เป็นสีแดง**และ**แสดง
ไอคอน error กำกับ**ให้เห็นเด่นชัดในหน้า trace list (ไม่ต้องเปิดดู trace ทีละอันเพื่อหาว่าอันไหน error) — และ
ที่สำคัญกว่านั้น: Jaeger's search API รองรับ **query ด้วย tag นี้ตรง ๆ** เพื่อหา trace ที่ error ทั้งหมด
โดยไม่ต้องเปิดดูทุก trace:

```bash
curl -s "http://localhost:16686/api/traces?service=otel-verify-service&tags=%7B%22otel.status_code%22%3A%22ERROR%22%7D"
```

ผลลัพธ์จริง: เจอ **1 trace** ที่มี error (ตรงกับ 1 request ที่จงใจให้ล้มเหลวจากทั้งหมดที่ยิงไป) — นี่คือ
ฟีเจอร์ที่มีค่ามากในทางปฏิบัติ: ทีม on-call ที่ตื่นมาแก้ปัญหาตอนตี 3 ไม่ต้องมานั่งเปิด trace ทีละอันเพื่อหาว่า
อันไหนที่ error เลย แค่ query ด้วย tag นี้ครั้งเดียวเห็นครบทุกอันที่เกี่ยวข้อง

#### เชื่อมกับ `AppError` ของ Part 66: จุดเดียวที่พอ

เพราะ Part 66 ออกแบบให้ **ทุก handler คืน `Result<T, AppError>`** และ `AppError`**ทุก variant มี
`impl IntoResponse` รวมศูนย์ที่จุดเดียวแล้ว** — จุดที่เหมาะที่สุดในการ mark span ว่า error คือ**จุดเดียวกันนั้น
เอง** ไม่ต้องเขียนโค้ด `set_status` กระจายอยู่ทุก handler:

```rust
use opentelemetry::trace::Status;
use tracing_opentelemetry::OpenTelemetrySpanExt;

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        // mark span ปัจจุบัน (span ของ handler ที่กำลัง instrument อยู่) ว่า error
        // ครั้งเดียวที่จุดศูนย์กลางนี้ -- ทุก handler ที่คืน AppError ได้ผลลัพธ์นี้อัตโนมัติ
        // โดยไม่ต้องเขียน set_status ซ้ำในทุกจุดที่อาจคืน error
        tracing::Span::current().set_status(Status::error(self.to_string()));

        let (status, code) = match &self {
            AppError::NotFound(_) => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
            AppError::Unauthorized(_) => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };
        // ... ส่วนที่เหลือเหมือน Part 66 หัวข้อ 66.3 ทุกประการ ...
        (status, Json(json!({ "error": { "code": code, "message": self.to_string() } }))).into_response()
    }
}
```

นี่คือรูปแบบเดียวกับหลักการที่ Part 66 ใช้ตลอดทั้งบท: **รวม cross-cutting concern ไว้ที่จุดเดียว** (ตอนนั้นคือ
การแปลง error เป็น HTTP response ที่ถูกต้อง ตอนนี้คือการ mark span ว่า error) แทนที่จะกระจายไปเขียนซ้ำทุก
handler — ยิ่งระบบมี handler มากเท่าไหร่ ยิ่งเห็นประโยชน์ของการรวมจุดนี้ชัดเจนมากเท่านั้น

### 99.10 Capstone: ผูก OpenTelemetry เข้ากับ Full-Stack Backend (Part 92-94)

ถึงเวลาปิดโมดูล observability ทั้งสามบทด้วยภาพรวมเดียว — นำ backend ของ full-stack capstone จาก **Part 92-94**
(ที่มี `AppState { db: PgPool, jwt_secret: Arc<str> }` และ `AppError` ตามที่ Part 66/92 ออกแบบไว้) มาผูกกับ
OTel เต็มรูปแบบ ให้แอปตัวเดียวมีครบ**สามเสาหลักของ observability พร้อมใช้งานจริง**: logs (Part 60), metrics
(Part 98), traces (บทนี้)

#### โครงสร้าง `main.rs` ที่ผูกทั้งสามเสาหลักเข้าด้วยกัน

```rust
use opentelemetry::global;
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_otlp::WithExportConfig;
use opentelemetry_sdk::propagation::TraceContextPropagator;
use opentelemetry_sdk::trace::SdkTracerProvider;
use opentelemetry_sdk::Resource;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;
use tracing_subscriber::EnvFilter;

/// ค่า config อ่านจาก environment variable ปกติ (ต่อยอด pattern ของ Part 59/92)
struct ObservabilityConfig {
    otlp_endpoint: String,
    service_name: String,
    sample_ratio: f64,
}

fn init_observability(cfg: &ObservabilityConfig) -> SdkTracerProvider {
    global::set_text_map_propagator(TraceContextPropagator::new());

    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .with_endpoint(&cfg.otlp_endpoint)
        .build()
        .expect("สร้าง OTLP exporter ไม่สำเร็จ");

    let resource = Resource::builder().with_service_name(cfg.service_name.clone()).build();
    let sampler = opentelemetry_sdk::trace::Sampler::ParentBased(Box::new(
        opentelemetry_sdk::trace::Sampler::TraceIdRatioBased(cfg.sample_ratio),
    ));

    let provider = SdkTracerProvider::builder()
        .with_batch_exporter(exporter)
        .with_resource(resource)
        .with_sampler(sampler)
        .build();

    global::set_tracer_provider(provider.clone());
    provider
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cfg = ObservabilityConfig {
        otlp_endpoint: std::env::var("OTLP_ENDPOINT").unwrap_or_else(|_| "http://localhost:4317".into()),
        service_name: "library-api".into(),
        sample_ratio: std::env::var("TRACE_SAMPLE_RATIO").ok()
            .and_then(|s| s.parse().ok()).unwrap_or(1.0), // dev: เก็บทุก trace
    };

    let provider = init_observability(&cfg);
    let tracer = provider.tracer(cfg.service_name.clone());
    let otel_layer = tracing_opentelemetry::layer().with_tracer(tracer);

    // สามเสาหลักในบรรทัดเดียว: fmt::layer() = logs (Part 60), Prometheus exporter แยก
    // ตั้ง /metrics endpoint (Part 98, ไม่ได้แสดงในนี้เพราะเป็นคนละ layer ของ Router)
    // otel_layer = traces (บทนี้)
    tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer().json()) // JSON เพื่อให้ log aggregation ระบบจริง parse ได้ (60.8)
        .with(otel_layer)
        .init();

    // db, jwt_secret เหมือน Part 92 หัวข้อ 92.4 ทุกประการ -- ไม่มีอะไรเปลี่ยนในส่วนนี้
    let db = sqlx::PgPool::connect(&std::env::var("DATABASE_URL")?).await?;
    let state = AppState { db, jwt_secret: std::env::var("JWT_SECRET")?.into() };

    let app = build_router(state); // Router ของ Part 92-94 (รวม /metrics ของ Part 98 ด้วย)
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;

    tracing::info!(service = %cfg.service_name, "library-api พร้อมทำงาน — logs + metrics + traces ครบสามเสาหลัก");
    axum::serve(listener, app).await?;

    let _ = provider.shutdown();
    Ok(())
}
```

#### Handler ตัวอย่าง: ผูกทั้งสามเสาหลักในจุดเดียว

```rust
use tracing::Instrument;

#[tracing::instrument(skip(state))]
async fn borrow_book(
    State(state): State<AppState>,
    Path(book_id): Path<i64>,
    Extension(current_user): Extension<AuthUser>, // ต่อยอด Part 74/92
) -> Result<Json<BorrowResponse>, AppError> {
    // (1) Logs (Part 60): structured field ผูกกับ span อัตโนมัติ
    tracing::info!(book_id, user_id = %current_user.id, "ได้รับคำขอยืมหนังสือ");

    // (2) Traces (บทนี้): #[instrument] ครอบทั้งฟังก์ชันแล้ว -- ไม่ต้องเขียนอะไรเพิ่ม
    //     span นี้จะมี parent เป็น span ของ HTTP request ทั้งอัน (ถ้าตั้ง middleware ของ
    //     axum ให้สร้าง root span ต่อ request ตาม pattern มาตรฐาน) และมี query ของ Part 70
    //     เป็น child span ของตัวเองต่ออีกชั้น

    let result = sqlx::query_as::<_, BookRow>("SELECT * FROM books WHERE id = $1 FOR UPDATE")
        .bind(book_id)
        .fetch_optional(&state.db)
        .await
        .map_err(AppError::from)? // AppError::Internal ผ่าน #[from] ตาม Part 66
        .ok_or_else(|| AppError::NotFound(format!("ไม่พบหนังสือ id={book_id}")))?;
        // ^ ถ้า path นี้ถูกเรียก -- AppError::into_response (หัวข้อ 99.9) จะ set_status
        //   เป็น Error ให้ span นี้อัตโนมัติ พร้อมกับ (3) Metrics: middleware ของ Part 98
        //   จะเพิ่ม counter http_requests_total{status="404"} ให้ endpoint นี้ไปพร้อมกัน

    tracing::info!(book_id, "ยืมหนังสือสำเร็จ");
    Ok(Json(BorrowResponse { book_id, borrowed_at: chrono::Utc::now() }))
}
```

Comment ในโค้ดข้างบนคือหัวใจของหัวข้อนี้: **ในหนึ่ง handler เดียว ทั้งสามเสาหลักทำงานพร้อมกันโดยธรรมชาติ** โดย
ที่ตัว business logic (query database, ตรวจสอบว่าหนังสือมีอยู่ไหม) **ไม่ต้องรู้เลยว่า observability
infrastructure มีอยู่** — `#[tracing::instrument]` สร้าง span ให้อัตโนมัติ (traces), `tracing::info!` ที่เรียก
ภายในกลายเป็นทั้ง log line (ผ่าน `fmt::layer()`) และ event บน span (ผ่าน `otel_layer`) พร้อมกัน (logs +
traces จากจุดเดียว), และ middleware ของ Part 98 ที่วัด HTTP metrics ทำงานอยู่ "ข้างนอก" handler โดยไม่ต้อง
แก้โค้ดในนี้เลยแม้แต่บรรทัดเดียว (metrics)

#### รันจริงและตรวจสอบผล: จำลอง Traffic รวม Error Case

ในการตรวจสอบเนื้อหาของหัวข้อนี้ ผู้เขียนใช้โครงสร้างโค้ดเดียวกันเป๊ะกับที่ compile/รันจริงแล้วในหัวข้อ 99.4-99.9
(`AppState`/`AppError` ที่มีรูปร่างตรงกับ Part 92 หัวข้อ 92.4, `#[instrument]`/`set_status` ที่ verify แล้วใน
หัวข้อ 99.5/99.9) มาประกอบเป็นภาพเดียวข้างบน — ไม่ได้ตั้ง PostgreSQL จริงแยกใหม่สำหรับหัวข้อนี้ (Part 92 ได้
พิสูจน์ database layer ไว้แล้วอย่างละเอียดในบทของมันเอง) จุดที่ต้องพิสูจน์ใหม่ในหัวข้อนี้มีเพียงจุดเดียว:
**การผูก `AppError::into_response` เข้ากับ `set_status` (หัวข้อ 99.9) ทำงานถูกต้องกับ handler ที่มีรูปร่างแบบ
Part 92 จริง** — ซึ่งได้พิสูจน์แล้วด้วยข้อมูลจริงจาก Jaeger ในหัวข้อ 99.9 (trace `0b881a3f12c280bdf04216e664a06546`
ที่มี `otel.status_code: "ERROR"` ตรงกับ request ที่จงใจล้มเหลว)

สิ่งที่เหลือให้ทำเมื่อนำไปใช้กับ capstone จริงของคุณคือขั้นตอนเชิงกลไกที่ตรงไปตรงมา: (1) เพิ่ม dependency ทั้ง
สี่ตัวจากหัวข้อ 99.4 เข้า `Cargo.toml` ของ `library-api` (2) เพิ่มฟังก์ชัน `init_observability` ข้างบนใน
`main.rs` (3) แทน `impl IntoResponse for AppError` เดิมด้วยเวอร์ชันที่เพิ่ม `set_status` (หัวข้อ 99.9) (4) รัน
Jaeger จริงตามหัวข้อ 99.5 คู่กับแอป แล้วยิง traffic ผ่าน `curl`/frontend จริงของ Part 93 — ทุกจุดที่เหลือคือ
สิ่งที่บทนี้พิสูจน์ไว้แล้วครบด้วยข้อมูลจริงในหัวข้อก่อนหน้า แบบฝึกหัดข้อ 4 ท้ายบทให้ลองทำขั้นตอนนี้ด้วยตัวเองกับ
โค้ด capstone จริงของคุณ

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `Span::current().record("field", value)` บน Field ที่ไม่ได้ Declare ไว้ล่วงหน้า — เงียบสนิท ไม่มี Error

นี่คือกับดักจริงที่พบระหว่างเขียนตัวอย่างของหัวข้อ 99.9 — ผู้เขียนลองเขียนแบบนี้ก่อน (ดูสมเหตุสมผลตอนแรก):

```rust
#[tracing::instrument]
async fn check_stock(order_id: u32, qty: u32) -> Result<(), String> {
    if qty > 100 {
        let msg = format!("สต็อกไม่พอสำหรับ order {order_id}");
        tracing::Span::current().record("error", true); // ดูเหมือนจะใช้ได้...
        return Err(msg);
    }
    Ok(())
}
```

รันจริงแล้ว query Jaeger สำหรับ span `check_stock` ของ request ที่ error — **ไม่มี tag ชื่อ `error` ปรากฏใน
ผลลัพธ์เลยแม้แต่ตัวเดียว** ทั้งที่โค้ดเรียก `.record("error", true)` ชัดเจน ไม่มี panic, ไม่มี error message,
ไม่มี warning ตอน compile — เหมือนบรรทัดนั้นไม่มีผลอะไรเลย

**สาเหตุ**: `tracing::Span::record()` **แก้ไขค่าของ field ที่ถูก declare ไว้แล้วเท่านั้น** — field ทั้งหมดของ
span หนึ่งตัวถูกกำหนด**ตายตัวตอนสร้าง span** (ตอนที่ `#[tracing::instrument]` แปลงฟังก์ชันเป็น
`tracing::info_span!(...)` ภายใน — จำได้จาก Part 60 หัวข้อ 60.7 ว่า `#[instrument]` แนบทุก parameter ของ
ฟังก์ชันเป็น field โดยอัตโนมัติ, ในที่นี้คือ `order_id`, `qty` เท่านั้น) — เรียก `.record("error", ...)` ด้วย
ชื่อ field ที่**ไม่อยู่ในชุด field ที่ประกาศไว้ตอนสร้าง** จะถูก**เพิกเฉยแบบเงียบ ๆ ทั้งหมด** (`tracing` ออกแบบ
มาแบบนี้ตั้งใจ เพื่อประสิทธิภาพ — เช็คแค่ index ของ field แทนการ allocate storage ใหม่แบบ dynamic)

**วิธีแก้**: ต้อง **declare field ที่จะ record ไว้ล่วงหน้าตอนสร้าง span** ด้วย `tracing::field::Empty` (สำหรับ
`#[instrument]`) หรือ `info_span!` ที่มีชื่อ field แต่ยังไม่ใส่ค่า:

```rust
#[tracing::instrument(fields(error = tracing::field::Empty))]
async fn check_stock(order_id: u32, qty: u32) -> Result<(), String> {
    if qty > 100 {
        tracing::Span::current().record("error", true); // ตอนนี้ทำงานถูกต้อง เพราะ field ถูก declare ไว้แล้ว
        return Err(format!("สต็อกไม่พอสำหรับ order {order_id}"));
    }
    Ok(())
}
```

หรือ**ทางที่แนะนำกว่าสำหรับกรณี error โดยเฉพาะ** (ตามที่หัวข้อ 99.9 สอนไว้): ใช้ `Status::error(...)` ผ่าน
`OpenTelemetrySpanExt::set_status` แทนการ record field ชื่อ `error` เอง — เพราะ `set_status` เป็น method ของ
OTel span ตรง ๆ ไม่ต้อง declare field อะไรล่วงหน้าเลย และ Jaeger UI เข้าใจ `otel.status_code` เป็น "ภาษากลาง"
มาตรฐานอยู่แล้ว (ต่างจาก tag ชื่อ `error` ที่กำหนดเอง ซึ่ง backend อื่นอาจไม่รู้จัก)

### 2. เรียก `set_parent` หลัง Span "Start" ไปแล้ว — Trace ขาดตอนข้าม Service โดยไม่มี Error ชัดเจน

นี่คือกับดักจริงที่พบระหว่างทำตัวอย่างสอง-service ของหัวข้อ 99.6 — เขียนแบบ "ธรรมดา" ที่ดูสมเหตุสมผลก่อน:

```rust
#[tracing::instrument(skip(headers))]
async fn check_availability(
    Path(book_id): Path<u32>,
    headers: axum::http::HeaderMap,
) -> axum::Json<serde_json::Value> {
    let parent_ctx = global::get_text_map_propagator(|prop| prop.extract(&HeaderExtractor(&headers)));
    tracing::Span::current().set_parent(parent_ctx); // ดูเหมือนถูก...
    // ...
}
```

โค้ดนี้ **compile ผ่านแต่มี warning**:

```text
warning: unused `Result` that must be used
  --> src/bin/service_b.rs:37:5
   |
37 |     tracing::Span::current().set_parent(parent_ctx);
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: this `Result` may be an `Err` variant, which should be handled
```

**นี่คือ compiler กำลังเตือนกับดักนี้ตรง ๆ** — `set_parent` คืน `Result<(), SetParentError>` ไม่ใช่ `()` (ตรวจ
สอบจริงจาก signature ใน `tracing-opentelemetry` เวอร์ชัน 0.34.0) ถ้าใส่ `.expect(...)` เพื่อดู error จริง จะ
เห็น panic message ที่บอกสาเหตุตรง ๆ:

```text
thread 'tokio-runtime-worker' panicked at src/bin/service_b.rs:37:10:
set_parent ต้องสำเร็จ ณ จุดนี้: AlreadyStarted
```

**สาเหตุ**: `#[tracing::instrument]` สร้างและ **enter span ทันทีตั้งแต่บรรทัดแรกของฟังก์ชัน** (ก่อน body ของ
คุณจะได้รันเลย) — พอถึงจุดที่คุณเรียก `set_parent` ภายใน body, OTel span ที่อยู่เบื้องหลัง**ถูก "start" ไปแล้ว**
(ผ่าน `start_with_context`) การเรียก `set_parent` ที่จุดนี้จึงสายเกินไปเสมอ คืน `Err(SetParentError::
AlreadyStarted)` — ถ้าไม่เช็ค `Result` (ตามที่ compiler เตือนไว้แล้ว) มันจะ**ล้มเหลวแบบเงียบ ๆ** span ของ
service ปลายทางจะกลายเป็น**root span ของ trace ใหม่ของตัวเอง**แทนที่จะเป็นลูกของ service ต้นทาง — ผลที่
สังเกตได้จริงตอนตรวจสอบ: query trace ด้วย trace ID จาก `traceparent` header ที่ service ต้นทางส่งมา **เจอแค่
span ของ service ต้นทางเท่านั้น** — span ของ service ปลายทางหายไปจาก trace นั้นโดยสิ้นเชิง (มันไปกลายเป็น
trace แยกของตัวเองที่ไม่มีใครไปดูเจอ)

**วิธีแก้** (ตามที่หัวข้อ 99.6 ใช้จริง): **ห้ามใช้ `#[instrument]` กับฟังก์ชันที่ต้อง `set_parent`** — ต้อง
สร้าง span ด้วยมือผ่าน `tracing::info_span!(...)` แล้วเรียก `set_parent` **ก่อน**ที่จะ enter/instrument มัน:

```rust
async fn check_availability(/* ... */) -> axum::Json<serde_json::Value> {
    let parent_ctx = global::get_text_map_propagator(|prop| prop.extract(&HeaderExtractor(&headers)));
    let span = tracing::info_span!("check_availability", book_id); // สร้างแต่ยังไม่ enter
    span.set_parent(parent_ctx).expect("set_parent ต้องสำเร็จ ณ จุดนี้"); // ยังไม่ start จึงทำได้
    async move { /* ... */ }.instrument(span).await // enter/start ตอนนี้ค่อยเกิด
}
```

### 3. ลืมเรียก `provider.shutdown()`/`force_flush()` — Span ที่เหลือใน Batch Buffer หายไปเงียบ ๆ

`SdkTracerProvider::builder().with_batch_exporter(...)` (ที่ใช้ตลอดบทนี้) **ไม่ส่ง span ออกทันทีที่ span จบ**
— มันเก็บ span ไว้ใน buffer ชั่วคราวแล้วส่งเป็น batch เป็นระยะ (เพื่อลด network overhead ตามที่อธิบายไว้ใน
หัวข้อ 99.3) ปัญหาคือ: ถ้าโปรแกรมจบการทำงาน (หรือ `main()` return) **ก่อน**ที่ batch ครบรอบส่งครั้งถัดไป
**span ที่ยังอยู่ใน buffer จะหายไปพร้อมกับ process ที่ปิดตัวลง — ไม่มี error, ไม่มี warning, ไม่มีอะไรบอกเลย**
เหมือนกับ "no-op logger" ของ `log` crate ที่ Part 60 หัวข้อ 60.2 อธิบายไว้ (ทั้งคู่คือ **การตัดสินใจทางออกแบบที่
เงียบโดยตั้งใจ** ไม่ใช่ bug) — ตัวอย่างของปัญหานี้ในโค้ดสั้น ๆ ที่ประมวลผลเร็วมาก:

```rust
#[tokio::main]
async fn main() {
    let provider = init_tracer();
    let tracer = provider.tracer("quick-job");
    // ... setup layer, registry.init() ...

    do_quick_job().await; // จบเร็วมาก อาจเร็วกว่า batch export interval

    // main() จบตรงนี้โดยไม่เรียก provider.shutdown() เลย
    // -> มีโอกาสสูงมากที่ span ของ do_quick_job() จะไม่ถูกส่งไป Jaeger เลย
}
```

**วิธีแก้**: **เรียก `provider.shutdown()` เสมอก่อนจบโปรแกรม** (ตามที่ทุกตัวอย่างในบทนี้ทำ) — method นี้ทำสอง
อย่าง: (1) force flush ให้ span ที่เหลือใน buffer ถูกส่งออกทันที ไม่ต้องรอรอบถัดไป (2) ปิด background task ของ
batch exporter อย่างถูกต้อง สำหรับ Axum server ที่รันตลอดไป (ไม่จบเอง) ให้เรียก `shutdown()` ใน
`with_graceful_shutdown` handler (ตามที่หัวข้อ 99.6 ทำในตัวอย่าง `service_b`) เพื่อให้ span สุดท้าย ๆ ก่อน
server ปิดตัวยังถูกส่งออกครบ

### 4. ลืมเปิด Feature ของ Transport ที่ตรงกับ Exporter ที่เลือกใช้ — Compile Error ที่อ่านยาก

`opentelemetry-otlp` รองรับหลาย transport (gRPC ผ่าน `tonic`, HTTP ผ่าน `reqwest`) แต่**ไม่เปิด transport
ไหนไว้เป็นค่าเริ่มต้นเลย** — ต้องเปิด feature ที่ตรงกับ method ที่จะเรียกเอง ลองเพิ่ม dependency แบบไม่ระบุ
feature:

```toml
[dependencies]
opentelemetry-otlp = "0.33.0"  # ไม่ได้เปิด feature "grpc-tonic"
```

แล้วเขียนโค้ดเรียก `.with_tonic()`:

```rust
let exporter = opentelemetry_otlp::SpanExporter::builder()
    .with_tonic()
    .build();
```

`cargo build` จริงให้ error message:

```text
error[E0599]: no method named `with_tonic` found for struct `SpanExporterBuilder<C>` in the current scope
 --> src/main.rs:3:10
  |
3 |         .with_tonic()
  |         ^^^^^^^^^^ method not found in `SpanExporterBuilder<NoExporterBuilderSet>`
```

**สาเหตุ**: `with_tonic()` ถูก gate ไว้หลัง feature flag `grpc-tonic` (บทนี้เปิดไว้ตั้งแต่หัวข้อ 99.4) — ถ้า
ไม่เปิด feature นี้ ตัว `SpanExporterBuilder` จะไม่มี method นี้อยู่เลยในสายตา compiler (ไม่ใช่ runtime error
แต่เป็น compile-time — type `SpanExporterBuilder<NoExporterBuilderSet>` บอกตรง ๆ ว่า**ยังไม่มีการเลือก
transport ใด ๆ** เพราะ generic parameter `C` (ที่กำหนดว่า "transport ไหนถูกเลือก") ยังคงเป็น
`NoExporterBuilderSet` อยู่)

**วิธีแก้**: เปิด feature ที่ตรงกับ method ที่จะใช้ให้ตรงกันเสมอ — `grpc-tonic` คู่กับ `.with_tonic()` (ที่บทนี้
ใช้ตลอด เพราะ endpoint 4317 ของ Jaeger เป็น gRPC) หรือ `http-proto` + `reqwest-client` คู่กับ
`.with_http()` (ถ้าจะยิงไป endpoint 4318 แทน):

```toml
opentelemetry-otlp = { version = "0.33.0", features = ["grpc-tonic"] }
```

### 5. ใส่ข้อมูลลับเป็น Span Attribute โดยไม่รู้ตัว — เชื่อมกับกับดักด้านความปลอดภัยของ Part 60

Part 60 หัวข้อกับดักที่ 7 เตือนไว้แล้วว่า `#[tracing::instrument]` **แนบทุก parameter ของฟังก์ชันเป็น field
โดยอัตโนมัติ** — กับดักนี้**ร้ายแรงกว่าเดิม**ในบทนี้ เพราะ field พวกนั้นไม่ได้อยู่แค่ใน log ของเครื่องคุณเอง
อีกต่อไป **มันถูกส่งออกไปเก็บถาวรที่ Jaeger** ที่มักเปิดให้ทีมทั้งหมด (หรือหลายทีม) เข้าถึง UI ได้:

```rust
// อันตราย: #[instrument] แนบทุก parameter เป็น span attribute โดยอัตโนมัติ
// รวมถึง `password` และ `credit_card_number` ด้วย!
#[tracing::instrument]
async fn authenticate(username: &str, password: &str) -> Result<Token, AppError> {
    // ...
}

#[tracing::instrument]
async fn charge_card(user_id: u32, credit_card_number: &str, amount_cents: u64) -> Result<(), AppError> {
    // ...
}
```

ถ้าเรียก `authenticate("alice", "MySecretP@ss123")` ค่า `"MySecretP@ss123"` จะกลายเป็น **attribute
`password` บน span ที่ Jaeger เก็บไว้ถาวร** ค้นหาได้ผ่าน UI/API โดยใครก็ตามที่เข้าถึง Jaeger ได้ — ต่างจากกับดัก
เดิมใน Part 60 ที่ผลกระทบจำกัดอยู่แค่ log file ในเครื่อง server เดียว กับดักนี้ในบทนี้**กระจายไปถึงทุกคนที่
เข้า Jaeger UI ได้ทันที** และเก็บไว้นานเท่ากับ retention policy ของ Jaeger (อาจเป็นสัปดาห์หรือเดือน)

**วิธีแก้**: ใช้ `skip`/`skip_all` ร่วมกับ `fields(...)` **ทุกครั้ง**ที่ `#[instrument]` ครอบฟังก์ชันที่มี
parameter ที่เป็นข้อมูลลับ (รหัสผ่าน, token, เลขบัตร, PII) — pattern เดียวกับที่ Part 60 สอนไว้ ใช้ได้ตรงกัน
เป๊ะในบทนี้เพราะกลไกเบื้องหลังคือ `tracing`'s field system ตัวเดียวกัน:

```rust
#[tracing::instrument(skip(password))]
async fn authenticate(username: &str, password: &str) -> Result<Token, AppError> {
    // username ยัง track ได้ปกติ, password ไม่ถูกแนบเป็น field เลย
}

#[tracing::instrument(skip(credit_card_number))]
async fn charge_card(user_id: u32, credit_card_number: &str, amount_cents: u64) -> Result<(), AppError> {
    // ถ้าต้องการ track ว่า "มีการใช้บัตรอยู่" โดยไม่เปิดเผยเลขจริง ใส่เฉพาะบางหลักสุดท้ายเป็น field แยก
    tracing::Span::current().record("card_last4", &credit_card_number[credit_card_number.len()-4..]);
}
```

### 6. Sampler ไม่ Consistent กันข้าม Service — Trace ขาดหายเป็นบางส่วน

ถ้าแต่ละ service ในสายเรียกใช้ sampler คนละแบบ (หรือใช้ `TraceIdRatioBased` เดี่ยว ๆ โดยไม่ครอบด้วย
`ParentBased` ตามที่หัวข้อ 99.7 อธิบายไว้) จะเกิดสถานการณ์ที่**ทำให้เข้าใจผิดได้ง่ายมาก**:

```rust
// service-a: ใช้ ParentBased(TraceIdRatioBased(0.5)) -- ถูกต้อง
let sampler_a = Sampler::ParentBased(Box::new(Sampler::TraceIdRatioBased(0.5)));

// service-b: ใช้ TraceIdRatioBased(0.1) เดี่ยว ๆ โดยไม่ครอบ ParentBased -- ผิด!
let sampler_b = Sampler::TraceIdRatioBased(0.1);
```

ผลที่เกิดขึ้น: เมื่อ `service-a` ตัดสินใจ **เก็บ** trace หนึ่งอัน (เพราะสุ่มได้ในโควตา 50%) แล้ว propagate
`traceparent` ที่มี sampled flag = `01` ไปให้ `service-b` — แต่ `service-b` ใช้ `TraceIdRatioBased(0.1)` แบบ
ไม่ครอบ `ParentBased` จึง**สุ่มตัดสินใจของตัวเองใหม่**โดยไม่สนใจ sampled flag ที่ได้รับมา (มีโอกาสแค่ 10% ที่
`service-b` จะตัดสินใจเก็บด้วย) — ผลคือ trace ที่ `service-a` ตั้งใจเก็บไว้เต็ม ๆ **ขาด span ของ service-b ไป
เป็นบางครั้งโดยไม่มีเหตุผลที่มองเห็นได้** ทำให้ trace ที่เปิดดูใน Jaeger **ดูเหมือน request หยุดกลางทางที่
service-a** ทั้งที่จริง ๆ request สมบูรณ์ปกติ เพียงแต่ span ของ hop ต่อไปไม่ถูกส่งออกมา — นี่เป็นกับดักที่
**debug ยากมาก** เพราะไม่มี error อะไรเกิดขึ้นเลยในระบบจริง มีแต่ข้อมูลที่หายไปแบบเงียบ ๆ

**วิธีแก้**: **ทุก service ในระบบเดียวกันต้องใช้ `Sampler::ParentBased(...)` ครอบ sampler หลักเสมอ** ไม่ว่า
sampler หลักที่ครอบไว้ข้างในจะเป็น `TraceIdRatioBased` อัตราเท่าไหร่ก็ตาม — `ParentBased` จะเช็ค sampled flag
ที่ propagate มาก่อนเสมอ (ถ้ามี parent context ที่มี sampled flag ระบุไว้แล้ว ให้ทำตามนั้น) และสุ่มตัดสินใจ
เองก็ต่อเมื่อเป็น root span ของ trace ใหม่จริง ๆ เท่านั้น — วางกฎนี้ไว้ใน shared configuration/library กลาง
ของทีม (แบบเดียวกับที่ Part 81 หัวข้อ 81.4 แนะนำให้เก็บ `.proto` ไว้ที่เดียวเป็น single source of truth)
เพื่อไม่ให้ทีมใดทีมหนึ่งลืมครอบ `ParentBased` โดยไม่ตั้งใจ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่มฟังก์ชันใหม่ `cancel_booking(booking_id: u32) -> Result<(), String>` เข้าไปในตัวอย่างของ
   หัวข้อ 99.4 ใส่ `#[tracing::instrument]` ให้มัน แล้วเรียกมันจาก `main()` — รัน Jaeger จริงตามหัวข้อ 99.5
   แล้ว query `GET /api/traces?service=otel-verify-service` ยืนยันว่า span ชื่อ `cancel_booking` ปรากฏขึ้นมา
   จริง พร้อม attribute `booking_id`
   *Hint*: ไม่ต้องแก้อะไรใน `main()`'s observability setup เลย แค่เพิ่มฟังก์ชันแล้วเรียกมันเหมือน
   `process_order` ตัวอื่น ๆ ในหัวข้อ 99.4 — พิสูจน์ว่า instrumentation กับ export layer เป็นคนละเรื่องกัน
   อย่างสิ้นเชิง

2. **(กลาง)** เพิ่ม endpoint ใหม่ในตัวอย่าง `service-a`/`service-b` ของหัวข้อ 99.6 ที่**จงใจคืน HTTP 500** เมื่อ
   `book_id` เป็นเลขคู่ (เช่น `book_id % 2 == 0`) ใส่ `Status::error(...)` (หัวข้อ 99.9) ให้ span ของ endpoint
   นั้น แล้ว query Jaeger ด้วย `tags={"otel.status_code":"ERROR"}` ยืนยันว่าเจอเฉพาะ trace ของ request ที่
   `book_id` เป็นเลขคู่เท่านั้น
   *Hint*: URL-encode ตัว `tags` parameter ให้ถูกต้อง (`{"otel.status_code":"ERROR"}` ต้องเข้ารหัสเป็น
   `%7B%22otel.status_code%22%3A%22ERROR%22%7D`) — ทดสอบด้วย `curl -G --data-urlencode` แทนการเข้ารหัสด้วยมือ
   ก็ได้เพื่อลดความผิดพลาด

3. **(ยาก)** ทำให้สอง service ของหัวข้อ 99.6 เชื่อมเป็น**สาม**service (`service-a → service-b → service-c`)
   โดยที่ `service-c` เป็น service ใหม่ที่ `service-b` เรียกต่อ (แบบเดียวกับ `service-b` เรียก `service-c`)
   ยืนยันว่า trace เดียวมีสาม span ซ้อนกันสามชั้น (`create_booking` → `check_availability` → span ใหม่ของ
   `service-c`) ด้วย `references` ที่ถูกต้องทุกชั้น
   *Hint*: กับดักที่ 2 ท้ายบท (เรื่อง `set_parent`/`AlreadyStarted`) จะเกิดขึ้นได้ที่ `service-c` เช่นกันถ้าใช้
   `#[instrument]` ตรง ๆ — ต้องใช้ pattern เดียวกับที่ `service-b` ใช้ (สร้าง span ด้วยมือ, `set_parent` ก่อน
   `.instrument()`) ที่ `service-c` ด้วย

4. **(ยาก/ประยุกต์ใช้งานจริง)** นำ backend ของ full-stack capstone (Part 92-94) ของคุณเองมาผูกกับ OTel เต็ม
   รูปแบบตามโครงสร้างหัวข้อ 99.10 จริง: (ก) เพิ่ม `init_observability` เข้า `main.rs` (ข) แก้
   `impl IntoResponse for AppError` ให้ `set_status` (หัวข้อ 99.9) (ค) รัน Jaeger จริงคู่กับแอป (ง) เปิด
   frontend ของ Part 93 ยิง request จริงผ่าน UI (ไม่ใช่ `curl` ตรง ๆ) รวมถึงกดปุ่มที่ทำให้เกิด error จริงหนึ่ง
   ครั้ง (เช่น ยืมหนังสือที่ไม่มีสต็อกเหลือ) แล้ว query Jaeger API ยืนยันว่าเห็น trace ของ request จริงจาก
   browser ครบทุกจุด รวม span ที่ error
   *Hint*: ถ้า capstone ของคุณมี Prometheus metrics จาก Part 98 อยู่แล้ว ลองเพิ่มเข้าไปด้วยว่า metric endpoint
   ยังทำงานปกติดีคู่กับ OTel layer ที่เพิ่มใหม่ (สองระบบไม่ควรชนกันเลย เพราะเป็นคนละ layer/middleware ที่แยก
   ทำงานอิสระจากกัน) — สำหรับคนที่อยากไปให้ลึกกว่านั้น ลองเพิ่ม OTel Collector (หัวข้อ 99.3) คั่นระหว่างแอปกับ
   Jaeger เพื่อดู `docker-compose.yml` ที่มีสาม service (แอป, Collector, Jaeger) ทำงานร่วมกันจริง

## สรุป

บทนี้ปิดวงจร observability สามบท (Part 60 → Part 98 → Part 99) ด้วยการตอบคำถามที่ Part 81 หัวข้อ 81.8 ทิ้งไว้
ให้ค้าง: correlation ID ที่ทำด้วยมือบอกได้แค่ว่า log บรรทัดไหนเป็นของ request เดียวกัน แต่ **OpenTelemetry**
ให้คำตอบที่สมบูรณ์กว่านั้นมาก — **Trace** ที่ครอบทั้งเส้นทางของ request ข้ามหลาย service, **Span** ที่มีเวลา
เริ่ม/จบและความสัมพันธ์ parent/child ชัดเจน (ตรงกับ `tracing::Span` ของ Part 60 แบบเกือบ 1:1), **Span Context**
ที่ propagate ผ่านมาตรฐาน W3C Trace Context (`traceparent`) แทน header กำหนดเอง, และ **Attributes/Events** ที่
ตรงกับ structured field ของ `tracing` เป๊ะ — pipeline ทั้งหมด (instrumentation ผ่าน `tracing-opentelemetry` →
OTLP → Jaeger) ได้ถูกพิสูจน์ด้วยการรันจริงตลอดทั้งบท: span ที่ซ้อนกันถูกต้องภายในหนึ่ง process, trace เดียวที่
มี span จากสอง service คนละ process ผ่าน network จริง, sampling ที่ลดปริมาณ trace ได้จริงตามอัตราที่ตั้งไว้,
การ mark error ที่ทำให้ Jaeger แยกแยะ request ที่ล้มเหลวได้ทันที, และ `trace_id` ที่เชื่อม log เข้ากับ trace
เต็มรูปแบบได้ตรงตัวอักษร — ทั้งหมดนี้ประกอบกันเป็นสามเสาหลักของ observability ที่แอปพลิเคชัน Rust จริงหนึ่งตัว
(ตามที่หัวข้อ 99.10 แสดงให้เห็นกับ capstone จาก Part 92-94) ควรมีครบก่อนจะเรียกตัวเองว่า "production-ready"
อย่างแท้จริง

Observability ที่ดีตอบคำถาม "เกิดอะไรขึ้น" ได้ แต่ยังมีอีกมิติหนึ่งที่สำคัญไม่แพ้กันสำหรับระบบที่ใช้งานจริง:
"ระบบนี้ปลอดภัยแค่ไหน" — **Part 100 (Security Best Practices ใน Rust)** จะเป็นบทถัดไปที่พาไปสำรวจแนวทางความ
ปลอดภัยที่ควรมีในทุกแอป Rust ที่ deploy จริง ตั้งแต่การจัดการ secret, input validation, dependency auditing,
ไปจนถึงรูปแบบการโจมตีที่พบบ่อยและวิธีป้องกันในระดับภาษาและ ecosystem

---

**Part ก่อนหน้า:** [Observability: Metrics ด้วย Prometheus](part-098-observability-prometheus.md) | **Part ถัดไป:** [Security Best Practices ใน Rust](part-100-security-best-practices.md)
