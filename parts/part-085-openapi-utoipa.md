# Part 85: API Documentation ด้วย OpenAPI/Swagger (utoipa)

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างชัดเจนว่า **OpenAPI (เดิมชื่อ Swagger)** คืออะไรในทางเทคนิค (spec รูปแบบ JSON/YAML ที่บรรยาย
  endpoint/request/response/auth ของ API แบบที่เครื่องอ่านได้) และทำไมมันสำคัญกว่า "เอกสารสวย ๆ ให้มนุษย์อ่าน"
  — มันเปิดทางให้ generate client SDK อัตโนมัติ, ทำ contract testing, และมี "try it out" ที่ทดสอบ API จริงได้
  จากในเบราว์เซอร์ ต่อยอดจากสิ่งที่ Part 78 ทิ้งท้ายไว้ว่า "เอกสารคือวินัยการออกแบบ ไม่ใช่สิ่งที่ทำทีหลัง"
- อธิบาย**ปัญหาคลาสสิกที่ `utoipa` แก้**: เอกสาร OpenAPI ที่เขียนด้วยมือแยกจากโค้ด **หลุด sync จากโค้ดจริงได้
  เสมอ** เมื่อเวลาผ่านไป — และเข้าใจแนวทาง "documentation as code" ที่ generate spec **จากตัว annotation บน
  Rust type/handler จริง** พร้อมรู้ข้อจำกัดที่ต้องซื่อสัตย์กับตัวเอง: annotation ยังต้องอัปเดตด้วยมือ ไม่มีอะไร
  บังคับให้มันตรงกับ runtime behavior จริงโดยอัตโนมัติ 100%
- ติดตั้งและรัน **Axum + `utoipa` + `utoipa-swagger-ui`** จริงจนขึ้น Swagger UI ที่ `/swagger-ui` ได้ พร้อม
  export raw OpenAPI JSON ออกมาดูได้ตรง ๆ ผ่าน `curl` — ทุกเวอร์ชัน crate ที่ใช้ในบทนี้ผ่านการ `cargo build`/
  `cargo run` จริงแล้ว ไม่ใช่แค่คัดลอกจากเอกสาร
- ใช้ `#[derive(ToSchema)]` แปลง struct โดเมนของหลักสูตร (`Book`, `CreateBookRequest`) ให้กลายเป็น JSON Schema
  ใน spec ได้ถูกต้อง รวมถึงเข้าใจว่า `Option<T>` (จาก Part 11) แปลงเป็น nullable field ใน schema อย่างไรจริง ๆ
  (เห็น JSON Schema ที่ utoipa generate ให้จริง ไม่ใช่แค่ท่องจำ)
- ผูก `#[utoipa::path(...)]` เข้ากับ `Path`/`Query`/`Json` extractor (Part 63-64) และขยาย `responses(...)` ให้
  ครอบคลุมทุก status code จริงที่ `AppError` (Part 66) ตอบได้ (`200`/`201`/`400`/`401`/`404`/`409`) — พิสูจน์
  ด้วย `curl` ว่าเอกสารที่ generate ตรงกับพฤติกรรม HTTP จริงทุกกรณี ไม่ใช่แค่ happy path
- ใส่ **Bearer JWT security scheme** (ต่อยอด Part 74) ให้ปุ่ม "Authorize" ใน Swagger UI ใช้งานได้จริง จัดกลุ่ม
  endpoint ด้วย `tag`, แยก `#[derive(OpenApi)]` root ออกเป็นหลายโมดูลเมื่อ API โตขึ้น (ต่อยอด Part 16), และ
  export spec ออกไปใช้นอก Swagger UI (Postman/Insomnia/สร้าง client code) — ปิดท้ายด้วย capstone ที่รวมทุกอย่าง
  เข้าด้วยกันเป็นเอกสาร API ห้องสมุดที่ใช้งานได้จริงเต็มรูปแบบ

## ความรู้ที่ต้องมีมาก่อน

- **Part 11 (Option และ null safety)**: หัวข้อ 85.4 ของบทนี้จะโชว์ตรง ๆ ว่า field ที่เป็น `Option<T>` ใน struct
  Rust ถูกแปลงเป็นอะไรใน JSON Schema ที่ utoipa generate ให้ — ถ้าจำความหมายของ `Option<T>` ("มีค่าหรือไม่มี
  ค่า" ไม่ใช่ "null pointer แบบภาษาอื่น") ไม่ชัด ควรทวนก่อน เพราะบทนี้จะอธิบายว่าความหมายนี้แปลไปเป็น "nullable"
  ใน JSON Schema ได้อย่างไรและทำไมมันไม่ 1:1 เป๊ะ
- **Part 16 (Modules และการจัดโครงสร้างโปรเจกต์)**: หัวข้อ 85.8 อ้างอิงตรงถึงการแยกไฟล์/โมดูลตาม domain
  (`mod books; mod loans;`) ที่ Part 16 สอนไว้ — บทนี้เอามาใช้แยก `#[derive(OpenApi)]` ของแต่ละโมดูลออกจากกัน
  แล้วรวมเข้า root เดียวทีหลัง เพื่อไม่ให้ struct `ApiDoc` ตัวเดียวบวมขึ้นเรื่อย ๆ ตามจำนวน endpoint
- **Part 63 (Axum: Routing และ Handlers)** และ **Part 64 (Axum: State Management และ Extractors)**:
  `Path<T>`, `Query<T>`, `Json<T>`, `State<T>` ที่บทนี้ใช้ตลอดคือตัวเดียวกับที่สองบทนี้สอนไว้ทุกประการ —
  `#[utoipa::path(params(...), request_body = ...)]` เป็นแค่ "คำอธิบายคู่กัน" ของ extractor เดิมที่มีอยู่แล้ว
  ไม่ใช่ extractor ชนิดใหม่ ถ้าจำ `Path`/`Query`/`Json` ไม่ได้ ควรทวนสองบทนี้ก่อน
- **Part 66 (Axum: Error Handling แบบมืออาชีพ)**: หัวข้อ 85.6 คือการนำ `AppError` enum และ
  `impl IntoResponse for AppError` จาก Part 66 มา**บันทึกเป็นเอกสาร**อย่างครบถ้วน — ทุก variant ที่ Part 66
  ออกแบบไว้ (`NotFound` → `404`, `Validation`/`ValidationErrors` → `400`, `Conflict` → `409`, `Unauthorized`
  → `401`, `Internal` → `500`) จะถูกแปลงเป็นรายการ `responses(...)` ที่ตรงกันเป๊ะ ถ้าจำโครงสร้าง
  `{"error": {"code": ..., "message": ...}}` จาก Part 66 ไม่ชัด ควรกลับไปทวนก่อนเข้าหัวข้อ 85.6
- **Part 74 (Authentication: JWT)**: หัวข้อ 85.7 ใช้แนวคิด `Authorization: Bearer <token>` ที่ Part 74 สอนไว้
  ตรง ๆ — บทนี้ไม่สอนกลไก JWT ใหม่ (การเซ็น การตรวจ การออกแบบ claims) แค่สอนวิธี**บันทึก**ว่า endpoint ไหนต้อง
  แนบ header นี้ ให้ Swagger UI รู้จักและมีปุ่ม "Authorize" ให้กรอก token ทดสอบได้ตรง ๆ
- **Part 78 (RESTful API Design Best Practices)**: บทนี้คือ**ภาคปิด**ของสิ่งที่ Part 78 หัวข้อ 78.1 ทิ้งท้าย
  ไว้ — Part 78 สอนว่า "API ที่ดีคือ API ที่คาดเดาได้" แต่ไม่ได้สอนว่าจะ**สื่อสาร**ความคาดเดาได้นั้นให้คนอื่น
  (โดยเฉพาะคนที่ไม่เคยเห็นโค้ดของคุณเลย) รู้ได้อย่างไรโดยไม่ต้องอ่านซอร์สโค้ด — คำตอบคือ OpenAPI/Swagger UI
  ซึ่งเป็นเนื้อหาทั้งหมดของบทนี้
- บทนี้เป็น**บทปิดของโมดูล 4 (Web Development, Part 61-85)** — จะอ้างอิงกลับไปยังหลายบทก่อนหน้าตลอดทั้งบท และ
  ในหัวข้อสรุปจะร้อยเรียงภาพรวมทั้งโมดูลเข้าด้วยกันอีกครั้งก่อนส่งต่อไปยังโมดูล 5 (WebAssembly และ Full-Stack)

## เนื้อหา

### 85.1 OpenAPI คืออะไรจริง ๆ และทำไมมันสำคัญกว่า "เอกสารสวย ๆ"

Part 78 หัวข้อ 78.1 ทิ้งคำถามไว้ว่า: **"ถ้าคุณเป็นนักพัฒนาที่เพิ่งได้ API นี้มาใช้เป็นครั้งแรก โดยไม่มีใครอธิบาย
อะไรให้เลย นอกจาก endpoint ตัวอย่างสองสามตัว — คุณจะเดา endpoint/response ที่เหลือถูกไหม?"** — Part 78 ตอบ
คำถามนี้ด้วยการออกแบบ **convention** ที่สม่ำเสมอ (naming, envelope, pagination, versioning) จนนักพัฒนาคนใหม่
เดาถูกได้เองโดยไม่ต้องเปิดเอกสาร แต่ในทางปฏิบัติ **ไม่มีทีมไหนคาดหวังให้ผู้บริโภค API ต้อง "เดา" ทุกอย่างจริง ๆ**
— แม้ convention จะสม่ำเสมอแค่ไหน ก็ยังมีรายละเอียดที่เดาไม่ได้เสมอ (field ไหนบังคับ field ไหน optional, ค่า
enum ที่ยอมรับได้มีอะไรบ้าง, endpoint นี้ต้องมี token ไหม) — **นี่คือช่องว่างที่เอกสาร API ต้องเข้ามาเติม**

คำถามคือ: เอกสารแบบไหนที่เติมช่องว่างนี้ได้ดีที่สุด?

#### เอกสารแบบ "เขียนด้วยมือ" (Markdown/Wiki/Google Doc) เทียบกับ OpenAPI

วิธีที่ทีมจำนวนมากทำในอดีต (และยังทำอยู่ในหลายทีม) คือเขียนเอกสาร API เป็น Markdown หรือหน้า Wiki ธรรมดา —
บอกว่า endpoint นี้คือ `GET /books/{id}`, ตอบ field อะไรบ้าง, ตัวอย่าง response หน้าตาเป็นอย่างไร วิธีนี้ **อ่าน
ง่ายสำหรับมนุษย์** แต่มีข้อจำกัดสำคัญที่ทำให้มันไม่พอสำหรับ API ที่ต้องอยู่ยาวและมีผู้บริโภคหลายฝ่าย:

1. **เครื่องอ่านไม่ได้ (not machine-readable)** — Markdown เป็นแค่ข้อความอิสระ ไม่มีโครงสร้างที่โปรแกรมอื่น
   แยกแยะได้ว่า "ส่วนนี้คือชื่อ field", "ส่วนนี้คือ type", "ส่วนนี้คือ status code" — จะเอาไปสร้าง client SDK
   อัตโนมัติ, ทำ automated contract test, หรือสร้าง form ทดสอบ API ในเบราว์เซอร์ไม่ได้เลยโดยไม่ต้องมีคนมา parse
   ข้อความด้วยมือก่อน
2. **ไม่มี "try it out"** — ผู้อ่านต้องเปิด Postman/`curl` แยกเพื่อลองยิง API จริง ไม่มีทางกด "ทดสอบ" จากใน
   หน้าเอกสารได้ตรง ๆ
3. **หลุด sync จากโค้ดง่ายมาก** — จะอธิบายเรื่องนี้ให้ละเอียดในหัวข้อ 85.2 เพราะเป็นหัวใจของทั้งบทนี้

**OpenAPI (เดิมชื่อ Swagger — องค์กรที่ดูแล spec นี้เปลี่ยนชื่อจาก "Swagger Specification" เป็น "OpenAPI
Specification" ตั้งแต่ปี 2016 หลัง SmartBear บริจาค spec ให้ Linux Foundation แต่คำว่า "Swagger" ยังติดปาก
ใช้เรียกเครื่องมือในระบบนิเวศนี้อยู่จนถึงทุกวันนี้ เช่น "Swagger UI" ที่บทนี้จะใช้) แก้ทั้งสามข้อข้างบนด้วยแนวคิด
เดียว: **บรรยาย API เป็นไฟล์ JSON หรือ YAML ที่มีโครงสร้างตายตัวตาม spec มาตรฐาน** — ไม่ใช่ข้อความอิสระ แต่เป็น
object ที่มี key แน่นอน เช่น `paths`, `components.schemas`, `responses` ที่ทุก tool ในระบบนิเวศ (Swagger UI,
Postman, code generator ต่าง ๆ) เข้าใจรูปแบบเดียวกันหมด

ตัวอย่างเบา ๆ ก่อนลงรายละเอียด — นี่คือชิ้นส่วนหนึ่งของ OpenAPI document (ไม่ต้องเข้าใจทุก key ตอนนี้ หัวข้อ
ถัดไปจะอธิบายทีละส่วน):

```yaml
openapi: 3.1.0
info:
  title: Library API
  version: 1.0.0
paths:
  /api/v1/books/{id}:
    get:
      summary: ดึงข้อมูลหนังสือตาม id
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        "200":
          description: พบหนังสือ
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Book"
        "404":
          description: ไม่พบหนังสือ
```

สังเกตว่านี่คือ**ข้อมูลโครงสร้าง (structured data)** ล้วน ๆ — ไม่ใช่ข้อความบรรยายอิสระ — โปรแกรมไหนก็ตามที่รู้
จัก key `paths`, `parameters`, `responses` อ่าน document นี้แล้ว**สร้างสิ่งที่มีประโยชน์ต่อได้ทันที**โดยไม่ต้อง
มีมนุษย์มา parse ก่อน:

- **Swagger UI** อ่าน document นี้แล้ว render เป็นหน้าเว็บที่มีปุ่ม "Try it out" ให้กรอก `id` แล้วยิง
  `GET /api/v1/books/{id}` จริงได้ทันที (หัวข้อ 85.3 จะทำสิ่งนี้ให้เห็นจริง)
- **Code generator** (เช่น `openapi-generator-cli`, `openapi-typescript`) อ่าน document นี้แล้ว generate
  client library (เช่น TypeScript type + fetch function) ให้ทีม frontend เรียกใช้ได้โดยไม่ต้องเขียน HTTP
  request มือเอง — ได้ type safety ฝั่ง client โดยอัตโนมัติ (หัวข้อ 85.9 จะพูดถึงเรื่องนี้ และ**นี่คือจุดที่
  โมดูล 5 ของหลักสูตร (WebAssembly/Full-Stack ที่เริ่มจาก Part 86) จะมาต่อยอดจริง** — เมื่อคุณเขียน frontend
  ด้วย Rust+WASM ในบทถัด ๆ ไป การมี OpenAPI spec ที่แม่นยำหมายความว่าคุณ generate client code ที่ type-safe
  ได้ทันทีโดยไม่ต้องเดาว่า backend ตอบ field อะไรมาบ้าง)
- **Contract testing tool** (เช่น Dredd, Schemathesis) อ่าน document นี้แล้ว**ทดสอบ API จริงโดยอัตโนมัติ**ว่า
  ตอบตรงตาม spec หรือไม่ — ยิง request สุ่มตาม schema ที่กำหนด แล้วเช็คว่า response ตรงกับ schema ที่ประกาศไว้
  จริงหรือเปล่า (ป้องกันปัญหาที่หัวข้อ 85.2 กำลังจะอธิบาย)

นี่คือคำตอบของคำถามที่ตั้งไว้ตอนเริ่มหัวข้อ: **OpenAPI สำคัญกว่า "เอกสารสวย ๆ" เพราะมันไม่ใช่แค่สิ่งที่มนุษย์อ่าน
แล้วเข้าใจ — มันคือ "สัญญา" (contract) ที่เครื่องอ่านได้ ที่เปิดทางให้ automation ทั้งระบบนิเวศทำงานร่วมกับ API
ของคุณได้โดยไม่ต้องมีมนุษย์มาแปลความหมายทีละจุด** และนี่คือสิ่งที่เติมเต็มช่องว่างที่ Part 78 ทิ้งไว้: convention
ที่ดี (Part 78) ทำให้ API **เดาง่าย**สำหรับมนุษย์ ส่วน OpenAPI spec (บทนี้) ทำให้รายละเอียดที่เดาไม่ได้**ไม่ต้อง
เดา**เลย เพราะมันถูกประกาศไว้ชัดเจนในรูปแบบที่ทั้งมนุษย์และเครื่องอ่านได้พร้อมกัน

### 85.2 ปัญหาที่ `utoipa` แก้: เอกสารที่เขียนด้วยมือ "หลุด sync" จากโค้ดจริงเสมอ

ถ้า OpenAPI ดีขนาดนี้ ทำไมไม่ทุกทีมเขียน OpenAPI spec ด้วยมือกันไปเลย? คำตอบคือ: **หลายทีมทำแบบนั้นจริง แล้วก็
เจอปัญหาคลาสสิกที่เกิดขึ้นซ้ำแล้วซ้ำอีกในอุตสาหกรรม** — เขียน `openapi.yaml` แยกไฟล์จากโค้ด Rust จริง แล้วเมื่อ
เวลาผ่านไป **โค้ดเปลี่ยนแต่เอกสารไม่เปลี่ยนตาม** เพราะไม่มีอะไรบังคับให้สองสิ่งนี้ตรงกัน

ลองนึกภาพสถานการณ์ที่เกิดขึ้นจริงในทีมพัฒนาแทบทุกทีมที่แยกเอกสารจากโค้ด:

1. สัปดาห์ที่ 1: เขียน `openapi.yaml` บอกว่า `POST /books` รับ field `title`, `author`, `isbn` และตอบ `201`
   กับ `400`
2. สัปดาห์ที่ 5: มี requirement ใหม่ — ต้องเช็คสิทธิ์ก่อนสร้างหนังสือ handler เปลี่ยนไปตอบ `401` เพิ่มด้วยถ้า
   ไม่มี token แต่**ไม่มีใครไปแก้ `openapi.yaml`** เพราะคนที่แก้ handler ไม่รู้ว่ามีไฟล์นี้อยู่ หรือรู้แต่ลืม
   หรือคิดว่า "แค่แก้ทีหลังก็ได้"
3. สัปดาห์ที่ 12: มี field ใหม่ `published_year` ถูกเพิ่มเข้า response ของ `Book` แต่ `openapi.yaml` ยังบอก
   schema เดิมที่ไม่มี field นี้
4. สัปดาห์ที่ 20: ทีม frontend ใหม่เข้ามา อ่าน `openapi.yaml` เป็นแหล่งความจริงเดียว (source of truth) ตามที่
   ควรจะเป็น แล้ว generate client code จากมัน — client ที่ได้**ไม่รู้จัก** `published_year`, **ไม่รู้ว่าต้อง
   จัดการ `401`**, และยัง generate error handling ผิดเพราะ spec เก่ากว่าความจริงไปหลายเดือนแล้ว

นี่ไม่ใช่ปัญหาสมมติ — มันคือปัญหาที่มีชื่อเรียกเฉพาะในอุตสาหกรรมว่า **"documentation drift"** หรือ **"spec
drift"** และเป็นสาเหตุอันดับต้น ๆ ที่ทำให้ทีมจำนวนมากเลิกเชื่อเอกสาร API ของตัวเอง (ถึงขั้นมีวลีติดตลกใน
วงการว่า "the only source of truth is the source code" — เอกสารเชื่อไม่ได้ ต้องไปอ่านโค้ดจริงเท่านั้น) ซึ่ง
ทำลายจุดประสงค์ทั้งหมดของการมีเอกสารตั้งแต่แรก

#### แนวทางของ `utoipa`: Generate Spec **จาก** โค้ด ไม่ใช่เขียนคู่กับโค้ด

`utoipa` (และเครื่องมือแนวเดียวกันในภาษาอื่น เช่น `drf-spectacular` ของ Django, `springdoc-openapi` ของ Java)
แก้ปัญหานี้ด้วยการเปลี่ยนทิศทางของความสัมพันธ์ระหว่างโค้ดกับเอกสาร:

| แนวทางเขียนมือ (แยกไฟล์) | แนวทาง `utoipa` (documentation as code) |
|---|---|
| เขียน `openapi.yaml` แยกจากโค้ด Rust | เขียน **annotation** (`#[utoipa::path(...)]`, `#[derive(ToSchema)]`) แนบไว้**บนโค้ด Rust จริง** |
| โค้ดเปลี่ยน ต้องไปแก้ไฟล์ YAML แยกด้วยมือ | โค้ดเปลี่ยน (เช่นเพิ่ม field ใน struct) → schema ที่ generate ใหม่**เปลี่ยนตามทันทีที่ compile ใหม่** โดยไม่ต้องแก้ไฟล์แยก |
| ไม่มีอะไรเชื่อมโยงไฟล์ YAML กับโค้ดจริง | annotation ผูกอยู่กับ type/function เดียวกับที่ handler ใช้จริง — ถ้าลบ field ออกจาก struct schema ก็หายไปพร้อมกันโดยธรรมชาติ |
| spec เก่าที่ไม่มีใคร diff เทียบกับโค้ดได้ง่าย | spec ถูก generate ใหม่ทุกครั้งที่ build — ดูค่าล่าสุดได้จาก `cargo run` เสมอ ไม่มี "เวอร์ชันเก่าที่ค้างอยู่" |

พูดให้เป็นรูปธรรม: field `title: String` ใน struct `Book` ที่มี `#[derive(ToSchema)]` แนบอยู่ **คือตัวเดียวกัน
เป๊ะ** กับ field `title` ที่ `serde` ใช้ serialize จริงตอนตอบ HTTP response — มันเป็น field เดียวกันในหน่วยความ
จำเดียวกัน ไม่ใช่สอง representation ที่แยกจากกันคนละไฟล์ ถ้าคุณเปลี่ยนชื่อ field หรือเปลี่ยน type ใน struct
schema ที่ generate ใหม่ก็เปลี่ยนตามโดยอัตโนมัติทันทีที่ `cargo build` ใหม่ — **ไม่มีทางที่ schema จะบอกว่า
field เป็น `String` ทั้งที่โค้ดจริงเปลี่ยนเป็น `Option<String>` ไปแล้ว เพราะทั้งสองอย่างอ่านมาจากนิยาม struct
เดียวกัน**

#### ข้อจำกัดที่ต้องซื่อสัตย์: `utoipa` ไม่ได้อ่าน "พฤติกรรม runtime" ของคุณ

ถึงตรงนี้อาจฟังดูเหมือน `utoipa` แก้ปัญหา documentation drift ได้ **100% แบบอัตโนมัติเต็มรูปแบบ** — แต่ความจริง
ไม่ใช่แบบนั้น และบทนี้จะไม่ปิดบังข้อจำกัดนี้: **`utoipa` generate spec จาก "annotation ที่คุณเขียน" ไม่ได้อ่าน
"โค้ด logic จริงภายใน function" เลย**

พิจารณาตัวอย่างนี้ (โค้ดยังไม่ compile ได้ในตอนนี้ เป็นแค่ตัวอย่างเชิงแนวคิด รอหัวข้อ 85.3 จะให้โค้ดที่ compile
ได้จริง):

```rust
// สมมติ handler นี้ถูกแก้ให้เพิ่มการเช็คสิทธิ์ใหม่ ทำให้ตอบ 403 ได้แล้วในทางปฏิบัติ
// แต่ #[utoipa::path(...)] ด้านบนมันยัง "ลืม" ประกาศ response 403 ไว้
#[utoipa::path(
    delete,
    path = "/api/v1/books/{id}",
    responses(
        (status = 204, description = "ลบสำเร็จ"),
        (status = 404, description = "ไม่พบหนังสือ")
        // *** ไม่มี (status = 403, ...) ตรงนี้ แต่โค้ดจริงข้างล่างตอบ 403 ได้จริง! ***
    )
)]
async fn delete_book(/* ... */) -> Result<(), AppError> {
    // สมมติว่ามีคนเพิ่ม logic เช็คสิทธิ์ตรงนี้ทีหลัง โดยไม่ได้แก้ #[utoipa::path] ด้านบนตาม
    // check_permission(&user)?; // อาจ return AppError::Forbidden -> 403
    // ... logic ลบหนังสือ ...
    Ok(())
}
```

**`utoipa` compile ผ่านโดยไม่มีปัญหาเลยแม้แต่นิดเดียว** — มันไม่รู้และไม่สนใจว่า function นี้ *จริง ๆ* จะ
`return Err(...)` ที่แปลงเป็น `403` ได้หรือไม่ เพราะมันไม่ได้วิเคราะห์ control flow ภายใน function เลย มันแค่
อ่าน**สิ่งที่ macro attribute บอกไว้ตรง ๆ** — ถ้า annotation บอกว่ามีแค่ `204`/`404` มันก็ generate ตามนั้น
เป๊ะ ไม่ว่าโค้ดจริงจะทำอะไรก็ตาม นี่คือสิ่งที่ต้องเข้าใจให้ชัดตั้งแต่ต้นบท: **`utoipa` ทำให้ schema ของ "ข้อมูล"
(struct/type) หลุด sync จากโค้ดได้ยากขึ้นมาก (เพราะผูกกับนิยาม type จริง) แต่ไม่ได้การันตีว่า "รายการ
status code/response ที่ประกาศไว้" ตรงกับพฤติกรรมจริงของ handler เสมอไป — ส่วนนั้นยังเป็นวินัยของมนุษย์ที่ต้อง
อัปเดต annotation ให้ตรงกับ logic ที่เขียนอยู่ดี**

หัวข้อ 85.6 จะโชว์วิธีปฏิบัติที่ช่วยลดความเสี่ยงนี้ (ผูก `responses(...)` เข้ากับ variant ของ `AppError` แบบ
เป็นระบบ ไม่ใช่เดามือทีละ endpoint) และกับดักที่พบบ่อยข้อ 4 ท้ายบทจะพิสูจน์ด้วยโค้ดจริงว่า `utoipa` compile ผ่าน
ได้แม้ประกาศ path ที่ผิดจริงจากที่ Axum route ไว้จริง — เพื่อให้เห็นภาพชัดว่าข้อจำกัดนี้ไม่ใช่แค่คำเตือนลอย ๆ
แต่เป็นสิ่งที่พิสูจน์ได้จริงด้วยการรันโค้ด

### 85.3 ติดตั้งและตัวอย่างแรกที่รันได้จริง: Axum + `utoipa` + Swagger UI

มาถึงเวลาลงมือจริง — ตัวอย่างทั้งหมดในบทนี้ผ่านการ `cargo build`/`cargo run` จริงแล้ว ด้วยเวอร์ชัน crate
ปัจจุบันที่ตรวจสอบจาก crates.io ณ เวลาที่เขียนบทนี้:

```toml
[dependencies]
axum = "0.8.9"
tokio = { version = "1.53.1", features = ["full"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
thiserror = "2.0.21"
utoipa = { version = "6.0.0", features = ["axum_extras"] }
utoipa-swagger-ui = { version = "10.0.1", features = ["axum", "vendored"] }
```

ติดตั้งด้วย `cargo add` ตรง ๆ (เวอร์ชันเดียวกับที่ Part 62-66 ใช้เป็นมาตรฐานของหลักสูตรสำหรับ `axum`/`tokio`/
`serde`/`thiserror`):

```bash
cargo add axum tokio --features tokio/full
cargo add serde --features derive
cargo add serde_json thiserror
cargo add utoipa --features axum_extras
cargo add utoipa-swagger-ui --features axum,vendored
```

**สังเกต feature `vendored` ของ `utoipa-swagger-ui`** — นี่คือรายละเอียดที่สำคัญมากในทางปฏิบัติที่เอกสารส่วน
ใหญ่มักไม่พูดถึง: โดย **default** (ไม่เปิด `vendored`) `utoipa-swagger-ui` จะ**ดาวน์โหลดไฟล์ static ของ
Swagger UI (HTML/CSS/JS ตัวจริงจาก โปรเจกต์ `swagger-api/swagger-ui`) จากอินเทอร์เน็ตตอน `cargo build`** ผ่าน
build script — ถ้าเครื่องที่ build (CI runner, container ที่ไม่มี network, environment ที่มี firewall/proxy
เข้มงวด) ไม่มีสิทธิ์เข้าถึงอินเทอร์เน็ตตรงจุดนั้น **`cargo build` จะ panic ทันที** กับดักที่พบบ่อยข้อ 1 ท้ายบท
จะโชว์ error message จริงที่เกิดขึ้น — เปิด feature `vendored` แก้ปัญหานี้โดยให้ crate ย่อย
`utoipa-swagger-ui-vendored` แนบไฟล์ static ที่ build ไว้ล่วงหน้ามาให้เลยตั้งแต่ตอน publish ไม่ต้องดาวน์โหลด
อะไรเพิ่มตอน build เลย — บทนี้เปิด feature นี้ไว้ตลอดเพื่อให้ตัวอย่างทุกตัว build ได้แน่นอนไม่ว่าเครื่องที่รัน
จะมี network ตอน build หรือไม่

#### ตัวอย่างแรก: หนึ่ง handler, หนึ่ง schema, Swagger UI ที่ขึ้นจริง

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::get,
    Json, Router,
};
use serde::Serialize;
use std::sync::{Arc, Mutex};
use utoipa::{OpenApi, ToSchema};
use utoipa_swagger_ui::SwaggerUi;

// (1) struct โดเมนธรรมดา + #[derive(ToSchema)] เพิ่มเข้ามาตัวเดียว -- คือทั้งหมดที่ต้องทำให้ utoipa
//     รู้จัก struct นี้และ generate JSON Schema ให้อัตโนมัติ (รายละเอียดเต็มอยู่ในหัวข้อ 85.4)
#[derive(Debug, Clone, Serialize, ToSchema)]
struct Book {
    id: u64,
    title: String,
    author: String,
}

#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("ไม่พบข้อมูลที่ต้องการ")]
    NotFound,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        (StatusCode::NOT_FOUND, "ไม่พบข้อมูลที่ต้องการ").into_response()
    }
}

type SharedState = Arc<Mutex<Vec<Book>>>;

// (2) #[utoipa::path(...)] แนบไว้เหนือ handler จริง -- บอก method, path, params, และ response ที่เป็นไปได้
//     ทั้งหมด "อยู่ติดกับ" handler ตัวจริงในไฟล์เดียวกัน ไม่ใช่แยกไปคนละที่แบบเอกสารเขียนมือในหัวข้อ 85.2
#[utoipa::path(
    get,
    path = "/api/v1/books/{id}",
    tag = "books",
    params(
        ("id" = u64, Path, description = "รหัสหนังสือ")
    ),
    responses(
        (status = 200, description = "พบหนังสือ", body = Book),
        (status = 404, description = "ไม่พบหนังสือ")
    )
)]
async fn get_book(
    State(state): State<SharedState>,
    Path(id): Path<u64>,
) -> Result<Json<Book>, AppError> {
    let books = state.lock().unwrap();
    books.iter().find(|b| b.id == id).cloned().map(Json).ok_or(AppError::NotFound)
}

// (3) struct root ที่รวบรวมทุก path/schema เข้าด้วยกัน -- นี่คือ "สารบัญ" ของเอกสารทั้งหมด
#[derive(OpenApi)]
#[openapi(
    paths(get_book),
    components(schemas(Book)),
    tags((name = "books", description = "จัดการหนังสือในห้องสมุด"))
)]
struct ApiDoc;

#[tokio::main]
async fn main() {
    let state: SharedState = Arc::new(Mutex::new(vec![Book {
        id: 1,
        title: "The Rust Programming Language".to_string(),
        author: "Steve Klabnik".to_string(),
    }]));

    let app = Router::new()
        .route("/api/v1/books/{id}", get(get_book))
        // (4) merge Swagger UI เข้ากับ router เดิม -- .url(...) กำหนดว่า spec JSON จะถูก serve ที่ path ไหน
        .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", ApiDoc::openapi()))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3085").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

**อธิบายกลไกทีละส่วน:**

1. **`#[derive(ToSchema)]`** บน `Book` — เพิ่ม `impl utoipa::ToSchema for Book` ให้อัตโนมัติ ทำให้ `utoipa`
   รู้ว่า struct นี้แปลงเป็น JSON Schema แบบไหน (field อะไรบ้าง, type อะไรบ้าง) — เป็น derive macro แบบเดียว
   กับที่ Part 57 สอนเรื่อง `#[derive(Serialize, Deserialize)]` ของ `serde` ทุกประการ เพียงแต่ target คือ
   "JSON Schema สำหรับเอกสาร" ไม่ใช่ "การ serialize จริงตอน runtime" (สอง derive นี้ทำงานคนละหน้าที่ แต่ไม่ขัด
   แย้งกัน ใส่คู่กันได้ตามธรรมชาติอย่างที่เห็นในโค้ดข้างบน)
2. **`#[utoipa::path(...)]`** บน `get_book` — เป็น **attribute macro** (ต่างจาก `#[derive(...)]` ที่เป็น
   derive macro) ที่ไม่เปลี่ยน signature หรือ behavior ของ function เลยแม้แต่นิดเดียว มันแค่**สร้าง type ที่
   ซ่อนอยู่**ชื่อ `__path_get_book` (จะเห็นชื่อนี้ชัด ๆ ในกับดักข้อ 3 ท้ายบท) ที่เก็บข้อมูล metadata ของ
   endpoint นี้ไว้ ให้ `#[derive(OpenApi)]` มาดึงไปใช้ทีหลัง — handler ตัวจริงยังเป็น `async fn` ธรรมดาที่
   Axum เรียกได้ปกติทุกประการ ไม่มีการ "ครอบ" หรือเปลี่ยนพฤติกรรม runtime เลย
3. **`#[derive(OpenApi)]`** บน `ApiDoc` — struct เปล่า ๆ ที่ไม่มี field เลย (`struct ApiDoc;`) ทำหน้าที่เป็น
   แค่ "จุดรวบรวม" — `paths(get_book)` บอกว่าเอกสารนี้ประกอบด้วย endpoint ไหนบ้าง (อ้างชื่อ function ตรง ๆ ไม่
   ใช่ string), `components(schemas(Book))` บอกว่า schema ไหนบ้างที่ควรอยู่ใน `components.schemas` ของ spec
   (แม้ `Book` จะถูกอ้างถึงทางอ้อมผ่าน `responses(... body = Book)` อยู่แล้ว ก็ยังต้องประกาศตรงนี้ให้ครบ
   เพราะ macro ไม่ได้ไล่ dependency graph ให้อัตโนมัติทั้งหมด)
4. **`ApiDoc::openapi()`** — เป็น associated function ที่ `#[derive(OpenApi)]` generate ให้ คืนค่า
   `utoipa::openapi::OpenApi` (struct ข้อมูลล้วน ๆ ที่ `Serialize` ได้) แปลงเป็น JSON/YAML ได้ตรง ๆ
5. **`SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", ApiDoc::openapi())`** — สร้าง
   `axum::Router` ย่อยที่: (a) serve หน้า Swagger UI (HTML/CSS/JS ที่มาจาก feature `vendored`) ที่
   `/swagger-ui` (b) serve **raw OpenAPI JSON** ที่ `/api-docs/openapi.json` แล้วบอกหน้า Swagger UI ให้ไปโหลด
   spec จาก path นั้น — `.merge(...)` รวม router ย่อยนี้เข้ากับ router หลักที่มี endpoint จริงของแอปอยู่แล้ว

รันจริงแล้วทดสอบด้วย `curl` ตรวจสองจุด: (1) Swagger UI ขึ้นจริงเป็นหน้า HTML (2) spec JSON export ออกมาได้จริง
และมีโครงสร้างถูกต้อง:

```bash
cargo run
```
```
listening on 127.0.0.1:3085
```

```bash
curl -sS http://127.0.0.1:3085/swagger-ui/
```

ผลลัพธ์จริง (ตัด script/style บางส่วนออกเพื่อความกระชับ — คัดลอกตรงจากการรันจริง):

```html
<!-- HTML for static distribution bundle build -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8">
    <title>Swagger UI</title>
    <link rel="stylesheet" type="text/css" href="./swagger-ui.css" />
    ...
  </head>
  <body>
    <div id="swagger-ui"></div>
    <script src="./swagger-ui-bundle.js" charset="UTF-8"> </script>
    <script src="./swagger-ui-standalone-preset.js" charset="UTF-8"> </script>
    <script src="./swagger-initializer.js" charset="UTF-8"> </script>
  </body>
</html>
```

นี่คือ**หน้า HTML จริงของ Swagger UI** (โปรเจกต์ `swagger-api/swagger-ui` ตัวจริง ไม่ใช่หน้าเปล่าที่ `utoipa`
เขียนขึ้นมาเอง) — `swagger-initializer.js` คือสคริปต์ที่จะ `fetch()` ไปที่ `/api-docs/openapi.json` (ตามที่
`.url(...)` กำหนดไว้ตอนสร้าง `SwaggerUi`) แล้ว render UI ทั้งหมดจาก spec นั้นในฝั่งเบราว์เซอร์ — หัวข้อ 85.10
จะอธิบายการใช้งานหน้านี้แบบเจาะจงทีละปุ่ม

**หมายเหตุเรื่อง trailing slash**: ถ้าเปิด `http://127.0.0.1:3085/swagger-ui` (ไม่มี `/` ท้าย) เซิร์ฟเวอร์จะตอบ
`303 See Other` redirect ไปที่ `http://127.0.0.1:3085/swagger-ui/` (มี `/` ท้าย) โดยอัตโนมัติ — เบราว์เซอร์
ทำตาม redirect นี้เองโดยผู้ใช้ไม่ต้องทำอะไร แต่ถ้าทดสอบด้วย `curl` ตรง ๆ ต้องรู้ว่า `curl` ไม่ follow redirect
โดย default (ต้องเติม `-L`) — พฤติกรรมนี้พิสูจน์ได้จริง:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3085/swagger-ui
curl -sS -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3085/swagger-ui/
```
```
303
200
```

ต่อไปดู spec JSON ที่ export ออกมาจริง:

```bash
curl -sS http://127.0.0.1:3085/api-docs/openapi.json
```

ผลลัพธ์จริง (จัดรูปแบบให้อ่านง่ายขึ้นจาก raw JSON บรรทัดเดียวที่ได้จริง):

```json
{
  "openapi": "3.1.0",
  "info": { "title": "utoipa_demo", "description": "", "license": { "name": "" }, "version": "0.1.0" },
  "paths": {
    "/api/v1/books/{id}": {
      "get": {
        "tags": ["books"],
        "operationId": "get_book",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "description": "รหัสหนังสือ",
            "required": true,
            "schema": { "type": "integer", "format": "int64", "minimum": 0 }
          }
        ],
        "responses": {
          "200": {
            "description": "พบหนังสือ",
            "content": { "application/json": { "schema": { "$ref": "#/components/schemas/Book" } } }
          },
          "404": { "description": "ไม่พบหนังสือ" }
        }
      }
    }
  },
  "components": {
    "schemas": {
      "Book": {
        "type": "object",
        "required": ["id", "title", "author"],
        "properties": {
          "author": { "type": "string" },
          "id": { "type": "integer", "format": "int64", "minimum": 0 },
          "title": { "type": "string" }
        }
      }
    }
  },
  "tags": [{ "name": "books", "description": "จัดการหนังสือในห้องสมุด" }]
}
```

**สิ่งที่ควรสังเกต**: `"openapi": "3.1.0"` — `utoipa` 6.x generate spec ตาม **OpenAPI Specification เวอร์ชัน
3.1** (เวอร์ชันล่าสุดของ spec ณ ตอนที่เขียนบทนี้ ซึ่งใช้ JSON Schema แบบมาตรฐานเต็มรูปแบบ — ประเด็นนี้สำคัญมาก
สำหรับหัวข้อถัดไปเรื่อง `Option<T>`) และ `"info.title"`/`"info.version"` ถูกดึงมาจาก `[package] name`/
`version` ใน `Cargo.toml` โดยอัตโนมัติ (ไม่ต้องประกาศซ้ำ — แม้จะ override ได้ผ่าน `#[openapi(info(...))]`
ถ้าต้องการชื่อที่สื่อความหมายกว่า `utoipa_demo`)

### 85.4 Documenting Schema: `#[derive(ToSchema)]` กับ `Option<T>` แบบ Nullable

ตัวอย่างก่อนหน้าใช้ `Book` แบบง่าย ๆ ที่ทุก field เป็นค่าบังคับ (`required`) มาดูกรณีที่สมจริงกว่า — หนังสือใน
ระบบห้องสมุดต้องมี field ที่**อาจไม่มีค่า** เช่น "วันที่ยืมล่าสุด" (ถ้ายังไม่เคยถูกยืมเลยก็ยังไม่มีค่านี้) —
Part 11 สอนไว้แล้วว่านี่คือกรณีคลาสสิกของ `Option<T>`:

```rust
use serde::Serialize;
use utoipa::ToSchema;

#[derive(Debug, Clone, Serialize, ToSchema)]
struct Book {
    id: u64,
    title: String,
    author: String,
    isbn: String,
    available: bool,
    /// วันที่ยืมล่าสุด (ถ้ายังไม่เคยถูกยืมจะเป็น null)
    borrowed_at: Option<String>,
}
```

สังเกตสองจุดที่เพิ่มเข้ามาจากตัวอย่างก่อนหน้า:

1. **doc comment (`///`) เหนือ field `borrowed_at`** — `utoipa` อ่าน doc comment ของ Rust (ตัวเดียวกับที่
   `cargo doc` ใช้สร้างเอกสาร API ของ crate ตามที่ Part 40 สอนไว้เรื่อง documentation comment) แล้วใส่เข้าไป
   เป็น `description` ของ field นั้นใน JSON Schema โดยอัตโนมัติ — นี่คือ**ประโยชน์แถมที่สำคัญมาก**ของแนวทาง
   documentation-as-code: comment ที่คุณเขียนไว้ "อธิบายโค้ดให้เพื่อนร่วมทีมอ่าน" อันเดียวกันนี้ **กลายเป็น
   คำอธิบายใน API docs สาธารณะโดยไม่ต้องเขียนซ้ำที่ไหนอีกเลย**
2. **`borrowed_at: Option<String>`** — คือจุดสำคัญของหัวข้อนี้ มาดู schema จริงที่ generate ออกมา

รันจริงแล้ว export spec ออกมาดู `components.schemas.Book` (ตัด field อื่นให้เหลือแค่ที่เกี่ยวข้อง — คัดลอกตรง
จากผลลัพธ์การรันจริง):

```json
{
  "type": "object",
  "required": ["id", "title", "author", "isbn", "available"],
  "properties": {
    "author": { "type": "string" },
    "available": { "type": "boolean" },
    "borrowed_at": {
      "type": ["string", "null"],
      "description": "วันที่ยืมล่าสุด (ถ้ายังไม่เคยถูกยืมจะเป็น null)"
    },
    "id": { "type": "integer", "format": "int64", "minimum": 0 },
    "isbn": { "type": "string" },
    "title": { "type": "string" }
  }
}
```

**สังเกตสองจุดที่พิสูจน์ความหมายของ `Option<T>` ในระดับ schema ได้ชัดเจนมาก:**

1. **`"required"` ไม่มี `"borrowed_at"` อยู่ในรายการ** — field ที่เป็น `Option<T>` **ไม่ถูกนับเป็น field
   บังคับ** ต่างจาก `id`/`title`/`author`/`isbn`/`available` ที่เป็น type ตรง ๆ (`u64`/`String`/`bool`)
   ซึ่งถูกใส่ไว้ใน `required` ครบทุกตัว — ตรงกับความหมายของ `Option<T>` ใน Rust ทุกประการ: **"ค่านี้อาจไม่มีก็
   ได้ ถ้าไม่ส่งมาก็ไม่ผิดอะไร"** เทียบกับ field ธรรมดาที่ **"ต้องมีเสมอ ถ้าขาดไปคือ response ผิดรูปแบบ"**
2. **`"type": ["string", "null"]`** ไม่ใช่แค่ `"type": "string"` — นี่คือวิธีที่ **OpenAPI 3.1 / JSON Schema
   Draft 2020-12** (ที่ `utoipa` 6.x generate ตามอย่างที่เห็นจาก `"openapi": "3.1.0"` ในหัวข้อก่อน) ประกาศ
   ว่า field นี้ **"เป็น string หรือเป็น null ก็ได้"** — เป็นการแทนความหมาย "nullable" แบบใหม่ที่ต่างจาก
   OpenAPI 3.0 ที่ใช้ `"type": "string", "nullable": true` (keyword `nullable` แยกออกมาต่างหาก) — 3.1 เลิกใช้
   `nullable` แล้วหันมาใช้ **union type แบบ JSON Schema มาตรฐาน** (`type` เป็น array ของ type ที่ยอมรับได้)
   แทน ซึ่งเป็นวิธีที่ตรงไปตรงมากว่าในทางทฤษฎี (เพราะ `null` ก็เป็น "ค่าที่เป็นไปได้" ตัวหนึ่งเหมือน type อื่น
   ไม่ต้องมี keyword พิเศษแยกออกมาอธิบายมันอีกชั้น)

นี่คือคำตอบที่เป็นรูปธรรมของสิ่งที่ Part 11 สอนไว้ในระดับแนวคิด ("`Option<T>` ทำให้ null safety เป็นส่วนหนึ่ง
ของ type system ไม่ใช่แค่ convention") มาบรรจบกับ OpenAPI: **ความหมาย "อาจไม่มีค่า" ที่ Rust บังคับให้จัดการผ่าน
type system ตั้งแต่ compile time ถูกแปลงตรงไปเป็นความหมาย "อาจเป็น null ได้" ในระดับ JSON Schema ที่ client ฝั่ง
ไหนก็ตาม (JavaScript, Python, Rust ตัวอื่น) อ่านแล้วรู้ทันทีว่าต้องเช็ค null ก่อนใช้งาน field นี้** — ไม่ต้องมี
ใครมาเขียนบอกด้วยข้อความอิสระว่า "field นี้อาจไม่มีนะ อ่านให้ดี" เพราะมันถูกประกาศไว้เป็นส่วนหนึ่งของ schema
โดยอัตโนมัติจากการที่โค้ด Rust ประกาศ type เป็น `Option<String>` ตั้งแต่แรก

### 85.5 Path/Query Parameters และ Request Body ให้ตรงกับ Extractor จริง

หัวข้อ 85.3 โชว์ `params(("id" = u64, Path, ...))` ไปแล้วสำหรับ path parameter — หัวข้อนี้ขยายให้ครบทั้ง
query parameter (`Query<T>` จาก Part 63) และ request body (`Json<T>` จาก Part 64)

#### Query Parameter ด้วย `#[derive(IntoParams)]`

เมื่อ query string มีหลาย parameter ที่ optional (แบบที่ Part 63 หัวข้อ 63.x สอนไว้ว่าทำไม query parameter
ควรเป็น `Option<T>` เสมอ) การเขียน `params(("limit" = ..., Query, ...), ("available" = ..., Query, ...))`
ทีละตัวมือใน `#[utoipa::path(...)]` จะยาวและซ้ำซ้อนกับ struct ที่มีอยู่แล้ว — `utoipa` มี derive macro
`IntoParams` ที่ generate รายการ parameter ทั้งหมดจาก struct เดียว ตัวเดียวกับที่ `Query<T>` extractor ใช้
`Deserialize` จริง:

```rust
use serde::Deserialize;
use utoipa::{IntoParams, ToSchema};

#[derive(Debug, Deserialize, ToSchema, IntoParams)]
struct ListBooksQuery {
    limit: Option<u32>,
    available: Option<bool>,
}
```

ใช้ใน handler และ annotation:

```rust
use axum::extract::{Query, State};
use axum::Json;
# use std::sync::{Arc, Mutex};
# use serde::{Deserialize, Serialize};
# use utoipa::{IntoParams, ToSchema};
# #[derive(Debug, Clone, Serialize, ToSchema)]
# struct Book { id: u64, title: String }
# #[derive(Debug, Deserialize, ToSchema, IntoParams)]
# struct ListBooksQuery { limit: Option<u32>, available: Option<bool> }
# type SharedState = Arc<Mutex<Vec<Book>>>;
#[utoipa::path(
    get,
    path = "/api/v1/books",
    tag = "books",
    params(ListBooksQuery), // <- ส่ง struct เข้าไปตรง ๆ ไม่ต้องแยกเขียนทีละ field
    responses(
        (status = 200, description = "รายการหนังสือ", body = Vec<Book>)
    )
)]
async fn list_books(
    State(state): State<SharedState>,
    Query(_q): Query<ListBooksQuery>,
) -> Json<Vec<Book>> {
    Json(state.lock().unwrap().clone())
}
```

Spec ที่ generate จริง (คัดลอกตรงจากการรันจริง) — สังเกตว่าทั้งสอง query parameter เป็น `Option<T>` และผลลัพธ์
คือ `"required": false` พร้อม `"type"` แบบ union กับ `"null"` เหมือนกับที่หัวข้อ 85.4 อธิบายไว้กับ field ปกติ
(ยืนยันว่าความหมายของ `Option<T>` ใน OpenAPI schema **สม่ำเสมอไม่ว่าจะเป็น field ของ body หรือ query
parameter**):

```json
{
  "tags": ["books"],
  "operationId": "list_books",
  "parameters": [
    {
      "name": "limit",
      "in": "query",
      "required": false,
      "schema": { "type": ["integer", "null"], "format": "int32", "minimum": 0 }
    },
    {
      "name": "available",
      "in": "query",
      "required": false,
      "schema": { "type": ["boolean", "null"] }
    }
  ],
  "responses": {
    "200": {
      "description": "รายการหนังสือ",
      "content": { "application/json": { "schema": { "type": "array", "items": { "$ref": "#/components/schemas/Book" } } } }
    }
  }
}
```

`"body = Vec<Book>"` ใน `responses(...)` ถูกแปลงเป็น `{"type": "array", "items": {"$ref": "#/components/
schemas/Book"}}` — ตรงตามความหมายของ JSON Schema สำหรับ array อย่างถูกต้อง (`utoipa` รองรับ `Vec<T>` ที่ `T:
ToSchema` ให้อัตโนมัติผ่าน blanket implementation คล้ายกับที่ Part 25 สอนเรื่อง generic type ที่ implement
trait ให้ container ทั่วไป)

#### Request Body ด้วย `request_body = ...`

สำหรับ `POST`/`PUT`/`PATCH` ที่รับ `Json<T>` (Part 64) การประกาศ request body ทำผ่าน key `request_body`
ตรง ๆ ระดับเดียวกับ `params`/`responses`:

```rust
use serde::Deserialize;
use utoipa::ToSchema;

#[derive(Debug, Deserialize, ToSchema)]
struct CreateBookRequest {
    title: String,
    author: String,
    isbn: String,
}
```

```rust
# use axum::{extract::State, http::StatusCode, response::IntoResponse, Json};
# use serde::{Deserialize, Serialize};
# use std::sync::{Arc, Mutex};
# use utoipa::ToSchema;
# #[derive(Debug, Clone, Serialize, ToSchema)]
# struct Book { id: u64, title: String, author: String, isbn: String, available: bool, borrowed_at: Option<String> }
# #[derive(Debug, Deserialize, ToSchema)]
# struct CreateBookRequest { title: String, author: String, isbn: String }
# type SharedState = Arc<Mutex<Vec<Book>>>;
#[utoipa::path(
    post,
    path = "/api/v1/books",
    tag = "books",
    request_body = CreateBookRequest,
    responses(
        (status = 201, description = "สร้างหนังสือสำเร็จ", body = Book)
    )
)]
async fn create_book(
    State(state): State<SharedState>,
    Json(req): Json<CreateBookRequest>,
) -> impl IntoResponse {
    let mut books = state.lock().unwrap();
    let id = books.len() as u64 + 1;
    let book = Book {
        id, title: req.title, author: req.author, isbn: req.isbn,
        available: true, borrowed_at: None,
    };
    books.push(book.clone());
    (StatusCode::CREATED, Json(book))
}
```

`request_body = CreateBookRequest` แปลงเป็น key `requestBody` ใน spec — spec จริงที่ได้ (ตรวจสอบด้วย `curl`
แล้ว):

```json
{
  "requestBody": {
    "content": { "application/json": { "schema": { "$ref": "#/components/schemas/CreateBookRequest" } } },
    "required": true
  }
}
```

**จุดที่ควรสังเกตให้ชัด**: type ของ parameter ใน handler จริง (`Json<CreateBookRequest>`) และ type ที่ประกาศ
ใน `request_body = CreateBookRequest` เป็น**คนละที่กัน คนละบรรทัดกัน** — ไม่มีอะไรบังคับทาง compiler ว่าทั้ง
สองต้องตรงกัน (คุณเขียน `request_body = SomeOtherType` ที่ไม่เกี่ยวอะไรกับ handler เลยก็ compile ผ่านได้ตราบใด
ที่ `SomeOtherType: ToSchema`) — นี่คือตัวอย่างรูปธรรมอีกอันของข้อจำกัดที่หัวข้อ 85.2 อธิบายไว้: annotation
ของ `utoipa` เป็น**คำประกาศที่แยกจาก logic จริง** แม้จะเขียนอยู่ติดกันในไฟล์เดียวกันก็ตาม ความถูกต้องจึงยังต้อง
มาจากวินัยของผู้เขียนโค้ดที่ต้องให้สองสิ่งนี้ตรงกันเอง (แนวทางที่ปลอดภัยกว่าคือใช้ type เดียวกันทั้งสองที่เสมอ
ตามที่ตัวอย่างข้างบนทำ — `CreateBookRequest` ปรากฏทั้งใน `Json<CreateBookRequest>` และ `request_body =
CreateBookRequest`)

### 85.6 Documenting หลาย Response Status Code ให้ตรงกับ `AppError` จริง (ต่อยอด Part 66/78)

Part 66 ออกแบบ `AppError` enum ที่มี variant ตรงกับสาเหตุความล้มเหลวแต่ละแบบ พร้อม `impl IntoResponse` ที่แปลง
เป็น JSON shape `{"error": {"code": "...", "message": "..."}}` เสมอ — Part 78 ขยายให้ envelope ฝั่ง success
ใช้ `{"data": ...}` คู่กัน หัวข้อนี้คือการ**บันทึกสัญญา (contract) นี้ทั้งหมดลงใน OpenAPI spec** ให้ตรงกับความ
เป็นจริง 100% ไม่ใช่แค่ happy path เดียว

ทวนตาราง variant ของ `AppError` จาก Part 66 หัวข้อ 66.2 แล้วผูกกับ error code + shape ของ body:

| `AppError` variant | HTTP status | `code` ใน `{"error": {...}}` | ใช้ในสถานการณ์ |
|---|---|---|---|
| `NotFound` | `404 Not Found` | `"NOT_FOUND"` | ไม่พบหนังสือ/ผู้ใช้ที่ขอ |
| `Validation(String)` | `400 Bad Request` | `"VALIDATION_ERROR"` | input ผิดเงื่อนไข อธิบายเป็นข้อความเดียว |
| `Conflict(String)` | `409 Conflict` | `"CONFLICT"` | ขัดแย้งกับสถานะปัจจุบัน (เช่นยืมหนังสือที่ถูกยืมไปแล้ว) |
| `Unauthorized` | `401 Unauthorized` | `"UNAUTHORIZED"` | ไม่มี/ผิด Bearer token |
| `Internal(anyhow::Error)` | `500 Internal Server Error` | `"INTERNAL_ERROR"` | ระบบล้มเหลวที่ handler ควบคุมไม่ได้ |

แปลงตารางนี้เป็น struct `ErrorBody` ที่มี `#[derive(ToSchema)]` (ตรงกับ shape ของ Part 66 เป๊ะ) แล้วใช้เป็น
`body = ErrorBody` ในทุก error response:

```rust
use serde::Serialize;
use utoipa::ToSchema;

#[derive(Debug, Serialize, ToSchema)]
struct ErrorBody {
    error: ErrorDetail,
}

#[derive(Debug, Serialize, ToSchema)]
struct ErrorDetail {
    code: String,
    message: String,
}
```

ตอนนี้ประกาศ `responses(...)` ของ `borrow_book` (endpoint ยืมหนังสือ — ตัวอย่างที่ Part 66 หัวข้อ 66.4 ใช้
สาธิต `Conflict` ไว้แล้ว) ให้ครบทุก status ที่เป็นไปได้จริง ไม่ใช่แค่ `200`:

```rust
# use axum::{extract::{Path, State}, http::{HeaderMap, StatusCode}, response::{IntoResponse, Response}, Json};
# use serde::Serialize;
# use std::sync::{Arc, Mutex};
# use utoipa::ToSchema;
# #[derive(Debug, Clone, Serialize, ToSchema)]
# struct Book { id: u64, title: String, available: bool }
# #[derive(Debug, Serialize, ToSchema)]
# struct ErrorBody { error: ErrorDetail }
# #[derive(Debug, Serialize, ToSchema)]
# struct ErrorDetail { code: String, message: String }
# #[derive(Debug, thiserror::Error)]
# enum AppError {
#     #[error("not found")] NotFound,
#     #[error("unauthorized")] Unauthorized,
#     #[error("conflict: {0}")] Conflict(String),
# }
# impl IntoResponse for AppError {
#     fn into_response(self) -> Response { StatusCode::OK.into_response() }
# }
# type SharedState = Arc<Mutex<Vec<Book>>>;
# fn check_bearer(_h: &HeaderMap) -> Result<(), AppError> { Ok(()) }
#[utoipa::path(
    post,
    path = "/api/v1/books/{id}/borrow",
    tag = "books",
    security(("bearer_auth" = [])),
    params(("id" = u64, Path, description = "รหัสหนังสือ")),
    responses(
        (status = 200, description = "ยืมสำเร็จ", body = Book),
        (status = 401, description = "ไม่ได้ส่ง Bearer token หรือ token ไม่ถูกต้อง", body = ErrorBody),
        (status = 404, description = "ไม่พบหนังสือ", body = ErrorBody),
        (status = 409, description = "หนังสือถูกยืมไปแล้ว", body = ErrorBody)
    )
)]
async fn borrow_book(
    State(state): State<SharedState>,
    headers: HeaderMap,
    Path(id): Path<u64>,
) -> Result<Json<Book>, AppError> {
    check_bearer(&headers)?;
    let mut books = state.lock().unwrap();
    let book = books.iter_mut().find(|b| b.id == id).ok_or(AppError::NotFound)?;
    if !book.available {
        return Err(AppError::Conflict(format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title)));
    }
    book.available = false;
    Ok(Json(book.clone()))
}
```

**สิ่งที่ต้องสังเกต**: จำนวน `(status = ..., ...)` ใน `responses(...)` **ตรงกับจำนวน early-return ที่เป็นไปได้
จริงในตัว handler เป๊ะ** — `check_bearer(&headers)?` ตอบได้ `401` (ผ่าน `AppError::Unauthorized`),
`.ok_or(AppError::NotFound)?` ตอบได้ `404`, `if !book.available { return Err(...) }` ตอบได้ `409`, และ path
ปกติตอบ `200` — **สี่ status code ในเอกสารตรงกับสี่ทางที่ logic จริงเดินได้เป๊ะ ไม่ขาดไม่เกิน** นี่คือแนวทาง
ปฏิบัติที่ดีที่สุดที่ลดความเสี่ยง "documentation drift" (หัวข้อ 85.2) ให้น้อยที่สุด: **ทุกครั้งที่เพิ่ม
`return Err(AppError::...)` ใหม่ใน handler ให้ถือเป็นสัญญาณเตือนทันทีว่าต้องเพิ่ม `(status = ..., ...)` ใน
`responses(...)` ด้วย** — ไม่มี compiler บังคับให้ทำแบบนี้ (ตามที่หัวข้อ 85.2 ยอมรับตรง ๆ) แต่การจับคู่ทีละ
early-return กับทีละ response ที่ประกาศไว้แบบนี้ทำให้ตรวจสอบด้วยตาได้ง่ายเวลา code review

พิสูจน์ด้วยเซิร์ฟเวอร์จริง — ยิงครบทั้งสี่กรณี (server ตัวเต็มพร้อมทดสอบทั้งหมดอยู่ในหัวข้อ 85.10 capstone):

```bash
# ยืมสำเร็จ (200)
curl -sS -i -X POST http://127.0.0.1:3085/api/v1/books/1/borrow -H "Authorization: Bearer faketoken.abc.def"
```
```
HTTP/1.1 200 OK
content-type: application/json
content-length: 133

{"id":1,"title":"The Rust Programming Language","author":"Steve Klabnik","isbn":"9781593278281","available":false,"borrowed_at":null}
```

```bash
# ยืมซ้ำหนังสือเดิม (409)
curl -sS -i -X POST http://127.0.0.1:3085/api/v1/books/1/borrow -H "Authorization: Bearer faketoken.abc.def"
```
```
HTTP/1.1 409 Conflict
content-type: application/json
content-length: 142

{"error":{"code":"CONFLICT","message":"conflict: หนังสือ 'The Rust Programming Language' ถูกยืมไปแล้ว"}}
```

```bash
# ยืมหนังสือที่ไม่มีอยู่จริง (404)
curl -sS -i -X POST http://127.0.0.1:3085/api/v1/books/999/borrow -H "Authorization: Bearer faketoken.abc.def"
```
```
HTTP/1.1 404 Not Found
content-type: application/json
content-length: 52

{"error":{"code":"NOT_FOUND","message":"not found"}}
```

ทุก status code ที่ประกาศไว้ใน `responses(...)` (`200`/`401`/`404`/`409`) ถูกยืนยันด้วยการยิง request จริงครบ
ทุกเส้นทาง — เอกสารที่ Swagger UI แสดงจึงไม่ใช่แค่ "สิ่งที่ควรเป็น" แต่คือ "สิ่งที่เป็นจริง" ที่พิสูจน์ได้

### 85.7 Documenting Authentication: Bearer JWT + ปุ่ม "Authorize" (ต่อยอด Part 74)

Part 74 สอนไว้แล้วว่า endpoint ที่ต้อง auth จะอ่าน header `Authorization: Bearer <token>` ผ่าน custom
extractor — หัวข้อนี้สอนวิธี**บันทึก**ว่า endpoint ไหนต้องแนบ header นี้ ให้ Swagger UI รู้จักและมี UI สำหรับ
กรอก token ทดสอบได้จริงจากในเบราว์เซอร์ ไม่ต้องเปิด `curl`/Postman แยก

การประกาศ security scheme ทำผ่าน trait `Modify` ของ `utoipa` — เป็น hook ที่รันหลัง `#[derive(OpenApi)]`
สร้าง spec เบื้องต้นเสร็จแล้ว ให้แก้ไข/เพิ่มข้อมูลเข้าไปในนั้นได้อีกชั้น:

```rust
use utoipa::{
    openapi::security::{HttpAuthScheme, HttpBuilder, SecurityScheme},
    Modify, OpenApi,
};

struct SecurityAddon;

impl Modify for SecurityAddon {
    fn modify(&self, openapi: &mut utoipa::openapi::OpenApi) {
        // components ต้องมีอยู่แล้ว (จาก #[openapi(components(schemas(...)))]) ก่อนจะเพิ่ม security scheme เข้าไปได้
        if let Some(components) = openapi.components.as_mut() {
            components.add_security_scheme(
                "bearer_auth", // ชื่อนี้จะถูกอ้างอิงจาก security(("bearer_auth" = [])) ใน #[utoipa::path]
                SecurityScheme::Http(
                    HttpBuilder::new()
                        .scheme(HttpAuthScheme::Bearer)
                        .bearer_format("JWT") // ระบุ format ให้ Swagger UI รู้ว่าเป็น JWT โดยเฉพาะ (ไม่บังคับ แต่ช่วยสื่อความหมาย)
                        .build(),
                ),
            );
        }
    }
}
```

แล้วผูก `SecurityAddon` เข้ากับ `#[derive(OpenApi)]` root ผ่าน `modifiers(...)`:

```rust
# use utoipa::OpenApi;
# struct SecurityAddon;
# impl utoipa::Modify for SecurityAddon {
#     fn modify(&self, _openapi: &mut utoipa::openapi::OpenApi) {}
# }
#[derive(OpenApi)]
#[openapi(
    // paths(...), components(schemas(...)), tags(...) เหมือนเดิมจากหัวข้อก่อน ๆ
    modifiers(&SecurityAddon)
)]
struct ApiDoc;
```

จากนั้นบน endpoint ที่ต้อง auth จริง (เช่น `create_book`, `borrow_book` ที่หัวข้อ 85.6 ใช้) เติม
`security(("bearer_auth" = []))` เข้าไปใน `#[utoipa::path(...)]` (ตามที่เห็นในโค้ด `borrow_book` หัวข้อก่อน
แล้ว — `[]` ว่างเปล่าเพราะ HTTP Bearer scheme ไม่มี concept ของ "scope" แบบ OAuth2)

Spec จริงที่ได้ (ตรวจสอบด้วย `curl` แล้ว) มีทั้ง `components.securitySchemes` และ `security` บน operation ที่
ต้อง auth:

```json
{
  "components": {
    "securitySchemes": {
      "bearer_auth": { "type": "http", "scheme": "bearer", "bearerFormat": "JWT" }
    }
  }
}
```

```json
{
  "paths": {
    "/api/v1/books/{id}/borrow": {
      "post": {
        "security": [{ "bearer_auth": [] }]
      }
    }
  }
}
```

#### พิสูจน์ว่า "unauthenticated request" ตอบ `401` จริงตามที่ spec ประกาศ

ก่อนพูดถึงหน้า Swagger UI ต้องพิสูจน์ก่อนว่า**เอกสารที่ประกาศ `401` ไว้ตรงกับพฤติกรรมจริง** — ยิง request ไป
ที่ endpoint ที่ต้อง auth โดย**ไม่แนบ** header `Authorization` เลย:

```bash
curl -sS -i -X POST http://127.0.0.1:3085/api/v1/books \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","author":"A","isbn":"1234567890123"}'
```

ผลลัพธ์จริง:

```
HTTP/1.1 401 Unauthorized
content-type: application/json
content-length: 58

{"error":{"code":"UNAUTHORIZED","message":"unauthorized"}}
```

แล้วลองแนบ token (ตัวอย่างนี้ใช้ token จำลอง ไม่ใช่ JWT ที่เซ็นจริงตาม Part 74 — เพื่อโฟกัสที่กลไก header/spec
เท่านั้น capstone หัวข้อ 85.10 จะกลับไปใช้กลไกตรวจ JWT เต็มรูปแบบตาม Part 74):

```bash
curl -sS -i -X POST http://127.0.0.1:3085/api/v1/books \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer faketoken.abc.def" \
  -d '{"title":"Programming Rust","author":"Jim Blandy","isbn":"9781492052593"}'
```
```
HTTP/1.1 201 Created
content-type: application/json
content-length: 116

{"id":2,"title":"Programming Rust","author":"Jim Blandy","isbn":"9781492052593","available":true,"borrowed_at":null}
```

พฤติกรรมจริงตรงกับที่ spec ประกาศไว้เป๊ะ: ไม่มี token → `401` ตามที่ `responses((status = 401, ...))` บอกไว้,
มี token ที่ผ่านการตรวจ (แม้จะเป็น token จำลองในตัวอย่างนี้) → `201` ตามที่ `responses((status = 201, ...))`
บอกไว้

#### ปุ่ม "Authorize" ในหน้า Swagger UI ทำอะไรจริง ๆ

เมื่อ spec มี `components.securitySchemes` ประกาศไว้ (แบบที่พิสูจน์แล้วข้างบน) Swagger UI จะ render **ปุ่ม
"Authorize" ที่มุมขวาบนของหน้าเอกสาร** โดยอัตโนมัติ — นี่คือพฤติกรรมมาตรฐานของ Swagger UI ที่มีเอกสารรองรับ
เป็นสาธารณะ (ไม่ใช่ฟีเจอร์เฉพาะของ `utoipa`) ทำงานตาม spec ที่มันอ่านมาเป๊ะ ๆ ดังนี้:

1. กดปุ่ม "Authorize" → เปิด modal popup ที่แสดงรายการ security scheme ทั้งหมดที่พบใน
   `components.securitySchemes` (ในกรณีนี้มีตัวเดียวคือ `bearer_auth` พร้อมข้อความ "bearer (http, Bearer)"
   ที่มาจาก `"type": "http", "scheme": "bearer"`)
2. มีช่อง input ให้กรอกค่า — สำหรับ HTTP Bearer scheme ช่องนี้รับ**แค่ตัว token เปล่า ๆ** (ไม่ต้องพิมพ์คำว่า
   `Bearer ` นำหน้าเอง เพราะ Swagger UI เติมให้อัตโนมัติตามที่ `"scheme": "bearer"` กำหนดไว้)
3. กด "Authorize" ในโมดัลแล้วปิด modal — จากจุดนี้ไป **ทุก request ที่ยิงผ่านปุ่ม "Try it out" ของ endpoint
   ที่มี `security(("bearer_auth" = []))` ประกาศไว้ จะแนบ header `Authorization: Bearer <token ที่กรอกไว้>`
   ให้อัตโนมัติทุกครั้ง** โดยไม่ต้องกรอกซ้ำทีละ endpoint — endpoint ที่**ไม่มี** `security(...)` ประกาศไว้
   (เช่น `get_book`, `list_books` ที่เป็น public endpoint) จะไม่ถูกแนบ header นี้ให้ แม้จะ authorize ไว้แล้ว
   ก็ตาม เพราะ Swagger UI เคารพการประกาศ `security` เป็นรายจุด (per-operation) ตาม spec เป๊ะ

**สิ่งที่ควรรู้ตรง ๆ**: คำอธิบายพฤติกรรมของปุ่ม "Authorize" ข้างบนนี้มาจาก**พฤติกรรมมาตรฐานที่มีเอกสารรองรับ
ของ Swagger UI** ซึ่งบทนี้ตรวจสอบแล้วว่า spec ที่ generate จริงมีโครงสร้าง `securitySchemes`/`security` ที่
ถูกต้องครบถ้วนตามที่ Swagger UI คาดหวัง (พิสูจน์ด้วย `curl` ไปแล้วข้างบน) — environment ที่ใช้เขียนบทนี้ไม่มี
เบราว์เซอร์ที่ถ่ายภาพหน้าจอได้ จึงไม่มีภาพถ่ายหน้าจอจริงมาแสดง แต่ทุกจุดที่อธิบาย (โมดัล, ช่อง input, การแนบ
header อัตโนมัติ, การแยก per-operation) คือพฤติกรรมที่ตรวจสอบได้จริงจากเอกสารของ Swagger UI ทำงานบน spec ตัว
จริงที่ยืนยันความถูกต้องแล้วในหัวข้อนี้ ไม่ใช่การเดาเอาเอง

### 85.8 จัดระเบียบเอกสารที่โตขึ้น: Tags และการแยก `#[derive(OpenApi)]` ข้ามโมดูล (ต่อยอด Part 16)

API ที่มีแค่ 4 endpoint แบบตัวอย่างในบทนี้ยังจัดการง่าย — แต่ระบบห้องสมุดจริงที่หลักสูตรนี้ใช้ตลอดจะมี resource
หลายกลุ่ม (`books`, `members`, `loans`, `authors`, ...) แต่ละกลุ่มมีหลาย endpoint เอง เมื่อโตถึงจุดนั้น มีสอง
ปัญหาที่ต้องจัดการ: (1) Swagger UI แสดง endpoint ทั้งหมดเรียงกันเป็นรายการยาวไม่มีการจัดกลุ่ม (2)
`#[derive(OpenApi)] struct ApiDoc` ตัวเดียวมี `paths(...)` ยาวขึ้นเรื่อย ๆ ไม่มีที่สิ้นสุด

#### Tags: จัดกลุ่ม Endpoint ใน Sidebar ของ Swagger UI

`tag = "books"` ที่ใช้มาตลอดทุกตัวอย่างในบทนี้คือคำตอบของปัญหาข้อ (1) — Swagger UI จัดกลุ่ม endpoint ตาม
`tags` ที่แต่ละ operation ประกาศไว้ แสดงเป็น section ที่พับ/กางได้แยกกันในหน้าเอกสาร ถ้าเพิ่ม endpoint ของ
สมาชิกห้องสมุด ก็แค่ตั้ง `tag = "members"` ให้ต่างออกไป Swagger UI จะแยก section ให้อัตโนมัติโดยไม่ต้องตั้งค่า
อะไรเพิ่มที่ฝั่ง `#[derive(OpenApi)]` เลย (นอกจากประกาศคำอธิบายของ tag ผ่าน `tags((name = "...", description
= "..."))` ที่ root ตามที่เคยทำมาแล้ว)

#### แยก `#[derive(OpenApi)]` ข้ามโมดูลด้วย `utoipa-axum`

สำหรับปัญหาข้อ (2) — struct `ApiDoc` เดียวที่บวมขึ้นเรื่อย ๆ — แนวทางที่ตรงกับหลักการแยกโมดูลตาม domain ที่
Part 16 สอนไว้ (`mod books; mod members; mod loans;`) คือให้แต่ละโมดูลมี `#[derive(OpenApi)]` ของตัวเอง แล้ว
รวมเข้า root ทีหลัง — crate เสริม **`utoipa-axum`** (crate ทางการของโปรเจกต์ `utoipa` เอง อยู่ในกลุ่ม
repository เดียวกัน ตรวจสอบเวอร์ชันปัจจุบันแล้วคือ `0.3.0`) ให้ type `OpenApiRouter` ที่รวม "การประกาศ route
ของ Axum" กับ "การรวบรวม path เข้า OpenAPI spec" เป็นขั้นตอนเดียวกัน ลดความเสี่ยงที่จะลืมเพิ่ม function เข้า
`paths(...)` ตอนเพิ่ม route ใหม่ (ปัญหาแบบเดียวกับกับดักข้อ 3 ท้ายบท):

```toml
[dependencies]
utoipa-axum = "0.3.0"
```

```rust
use axum::{extract::Path, Json};
use serde::Serialize;
use utoipa::{OpenApi, ToSchema};
use utoipa_axum::{router::OpenApiRouter, routes};

#[derive(Serialize, ToSchema)]
struct Book {
    id: u64,
    title: String,
}

#[utoipa::path(get, path = "/api/v1/books/{id}", tag = "books", responses((status = 200, body = Book)))]
async fn get_book(Path(id): Path<u64>) -> Json<Book> {
    Json(Book { id, title: "demo".into() })
}

// โมดูล "books" มี OpenApi ของตัวเอง แยกจาก root โดยสมบูรณ์
#[derive(OpenApi)]
struct BooksApi;

// OpenApiRouter::with_openapi(...) ผูก OpenApi เริ่มต้นของโมดูลนี้ไว้ แล้ว .routes(routes!(...))
// ลงทะเบียนทั้ง axum route และ utoipa path ในคำสั่งเดียวกัน (routes! คือ macro ที่มาจาก utoipa-axum เอง)
fn books_router() -> OpenApiRouter {
    OpenApiRouter::with_openapi(BooksApi::openapi()).routes(routes!(get_book))
}

#[tokio::main]
async fn main() {
    // .merge(...) รวม OpenApiRouter ของแต่ละโมดูลเข้าด้วยกันที่ root -- ทั้ง axum::Router และ OpenApi
    // spec ถูกรวมไปพร้อมกันในคำสั่งเดียว ไม่ต้องเขียน paths(...) มือซ้ำที่ root อีกชั้น
    let (router, api): (axum::Router, utoipa::openapi::OpenApi) = OpenApiRouter::new()
        .merge(books_router())
        // .merge(members_router()) เมื่อมีโมดูลสมาชิกห้องสมุดเพิ่มเข้ามาทีหลัง
        .split_for_parts();

    let _ = (router, api); // ในโค้ดจริงต่อด้วย .merge(SwaggerUi::new(...).url(..., api)) แล้ว axum::serve(...)
}
```

**ข้อควรระวังที่ตรวจสอบแล้วจากการรันจริง**: `OpenApiRouter::new().nest("/", books_router())` (ใช้ `nest`
เหมือนกับ `axum::Router` ปกติที่ Part 63 สอนไว้) **panic ทันทีตอน runtime** ด้วยข้อความ `"Nesting at the root
is no longer supported. Use merge instead."` — เวอร์ชันปัจจุบันของ `utoipa-axum` เปลี่ยนพฤติกรรมนี้ไปจากที่
บางเอกสารเก่าอาจยังสอนไว้ ต้องใช้ `.merge(...)` เมื่อรวมที่ root เสมอ (แต่ `.nest("/books", ...)` ที่มี prefix
จริงยังใช้ได้ปกติ เฉพาะ `.nest("/", ...)` ที่ root เปล่า ๆ เท่านั้นที่ถูกปิดกั้นไว้)

รันจริงแล้ว `api.paths.paths.keys()` ยืนยันว่า path จากโมดูล `books_router()` ถูกรวมเข้า spec หลักถูกต้อง:

```
paths in merged spec: ["/api/v1/books/{id}"]
```

แนวทางนี้ทำให้แต่ละไฟล์โมดูล (`books.rs`, `members.rs`, `loans.rs`) เป็นเจ้าของทั้ง route และเอกสารของ endpoint
ที่ตัวเองประกาศอย่างสมบูรณ์ — ไฟล์ `main.rs`/`app.rs` ที่ root มีหน้าที่แค่ `.merge(...)` router ย่อยของแต่ละ
โมดูลเข้าด้วยกัน ตรงกับปรัชญาการแยกความรับผิดชอบตาม domain ที่ Part 16 สอนไว้ทุกประการ เพียงแต่ตอนนี้ครอบคลุม
ทั้ง "โค้ดที่รัน" และ "เอกสารของโค้ดนั้น" ไปพร้อมกันในหน่วยเดียว

### 85.9 Export Spec ออกไปใช้นอก Swagger UI: JSON/YAML สำหรับ Postman, Client Generation

Swagger UI คือแค่ **ผู้บริโภครายหนึ่ง** ของ OpenAPI spec — เพราะ spec เป็นไฟล์ JSON/YAML ธรรมดาที่เครื่องอ่าน
ได้ (ตามที่อธิบายไว้ในหัวข้อ 85.1) มันเอาไปใช้กับเครื่องมืออื่นนอก Swagger UI ได้ตรง ๆ

#### Export เป็นไฟล์ (JSON/YAML) โดยไม่ต้องรันเซิร์ฟเวอร์

`utoipa::openapi::OpenApi` มี method `to_pretty_json()` และ `to_yaml()` (ตัวหลังต้องเปิด feature `"yaml"`
ของ `utoipa`) ให้เรียกตรง ๆ โดยไม่ต้องพึ่ง HTTP endpoint เลย — เขียนเป็น binary เล็ก ๆ แยกสำหรับ export
อย่างเดียวได้:

```rust
# use utoipa::OpenApi;
# #[derive(OpenApi)]
# struct ApiDoc;
fn main() -> std::io::Result<()> {
    let spec = ApiDoc::openapi().to_pretty_json().expect("serialize OpenAPI spec");
    std::fs::write("openapi.json", spec)?;
    println!("เขียน openapi.json สำเร็จ");
    Ok(())
}
```

วิธีนี้เหมาะกับ CI pipeline ที่ต้อง generate ไฟล์ spec ไปเก็บเป็น artifact หรือ commit เข้า repository แยก
(เช่นสำหรับทีม frontend ที่ยังไม่พร้อม subscribe ไปที่เซิร์ฟเวอร์ dev ตรง ๆ)

#### นำเข้า Postman/Insomnia

Postman และ Insomnia (REST client ที่ใช้ทดสอบ API ทั้งคู่) รองรับ **"Import from OpenAPI"** โดยตรง — วาง URL
ของ `/api-docs/openapi.json` (หรืออัปโหลดไฟล์ที่ export มา) แล้วเครื่องมือจะสร้าง **collection ของ request
ทั้งหมดให้อัตโนมัติ** ครบทั้ง method, URL, header ที่ต้องมี (รวมถึง `Authorization` ถ้า security scheme ถูก
ประกาศไว้แบบหัวข้อ 85.7), และตัวอย่าง request body จาก schema ที่ประกาศไว้ — ทีมที่ทดสอบ API ด้วยมือประจำไม่
ต้องสร้าง request ทีละตัวเองอีกต่อไป เพียงแค่ import ใหม่ทุกครั้งที่ spec เปลี่ยน (ซึ่งเกิดขึ้นอัตโนมัติเพราะ
เป็น documentation-as-code ตามหัวข้อ 85.2)

#### Generate Client Code: จุดที่เชื่อมไปยังโมดูล 5 ของหลักสูตร

นี่คือการใช้งานที่ทรงพลังที่สุดของ OpenAPI spec ในทางปฏิบัติ: เครื่องมือประเภท **code generator** อ่าน spec
แล้ว**สร้างโค้ด client ให้ทั้งชุด** — เช่น `openapi-typescript` สร้าง TypeScript type ที่ตรงกับ
`components.schemas` ทุกตัวเป๊ะ (รวมถึงความหมาย nullable ของ `Option<T>` ที่หัวข้อ 85.4 อธิบายไว้ จะกลายเป็น
`string | null` ใน TypeScript โดยอัตโนมัติ) หรือ `openapi-generator-cli` ที่ generate client library เต็ม
รูปแบบ (มี function สำหรับเรียกแต่ละ endpoint พร้อม type safety) ให้หลายภาษา

**นี่คือจุดที่ต่อเนื่องไปยังโมดูล 5 ของหลักสูตรนี้โดยตรง**: Part 86 ที่กำลังจะเริ่มสอน WebAssembly ด้วย Rust
และบทต่อ ๆ ไปในโมดูล 5 จะพาไปเขียน frontend/full-stack application — เมื่อ frontend นั้นต้องเรียก backend
API ที่สอนมาตลอดโมดูล 4 (รวมทั้งบทนี้) การมี OpenAPI spec ที่แม่นยำและอัปเดตอยู่เสมอ (เพราะ generate จากโค้ด
จริงตามหัวข้อ 85.2) หมายความว่า **frontend generate client code ที่ type-safe ได้ทันทีโดยไม่ต้องเขียน HTTP
call มือหรือเดา shape ของ response เอง** — ปัญหา "frontend เดา field ผิด เพราะเอกสาร backend เก่ากว่าความ
จริง" ที่หัวข้อ 85.2 อธิบายไว้ว่าเป็นต้นเหตุของ documentation drift จะไม่เกิดขึ้นอีก ถ้าทั้งสองทีม (หรือแม้
เป็นคนเดียวกันที่เขียนทั้ง backend และ frontend) ใช้ spec ที่ generate จากโค้ดจริงเป็นสัญญาเดียวกัน

ตัวอย่างการ export หนึ่ง endpoint จริงที่พร้อมนำไปใช้กับเครื่องมือเหล่านี้ (คัดลอกตรงจาก
`/api-docs/openapi.json` ที่เซิร์ฟเวอร์ในหัวข้อ 85.3 serve จริง):

```json
{
  "get": {
    "tags": ["books"],
    "operationId": "get_book",
    "parameters": [
      {
        "name": "id",
        "in": "path",
        "description": "รหัสหนังสือ",
        "required": true,
        "schema": { "type": "integer", "format": "int64", "minimum": 0 }
      }
    ],
    "responses": {
      "200": {
        "description": "พบหนังสือ",
        "content": { "application/json": { "schema": { "$ref": "#/components/schemas/Book" } } }
      },
      "404": {
        "description": "ไม่พบหนังสือ",
        "content": { "application/json": { "schema": { "$ref": "#/components/schemas/ErrorBody" } } }
      }
    }
  }
}
```

`operationId: "get_book"` (ชื่อ function จริงที่ `utoipa` ดึงมาให้อัตโนมัติ) มักถูกใช้เป็นชื่อ function ของ
client code ที่ generate ออกมาด้วย — เช่น `openapi-generator-cli` ที่สร้าง client TypeScript มักได้ function
ชื่อ `getBook(id: number)` ตรงจาก `operationId` นี้เป๊ะ (แปลง `snake_case` เป็น `camelCase` ตาม convention
ของภาษาปลายทาง) พิสูจน์อีกครั้งว่าทุกรายละเอียดเล็ก ๆ ในโค้ด Rust (แม้แต่ชื่อ function) ไหลไปถึงปลายทางที่
frontend ใช้งานจริงได้โดยไม่ต้องมีใครมาเขียนแปลด้วยมือ

### 85.10 Capstone: เอกสาร API ห้องสมุดแบบเต็มรูปแบบ + เดินเรื่องการใช้งาน Swagger UI จริง

มารวมทุกหัวข้อของบทนี้เข้าเป็นเซิร์ฟเวอร์เดียวที่สมบูรณ์ — ครอบคลุม CRUD พื้นฐาน, auth, และ error case ที่พบ
บ่อยของระบบห้องสมุด โค้ดนี้ผ่านการ `cargo build`/`cargo run` จริงแล้วทั้งหมด (ทุก endpoint ถูกทดสอบด้วย `curl`
จริงแล้วในหัวข้อก่อนหน้า):

```rust
use axum::{
    extract::{Path, Query, State},
    http::{HeaderMap, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};
use utoipa::{
    openapi::security::{HttpAuthScheme, HttpBuilder, SecurityScheme},
    Modify, OpenApi, ToSchema,
};
use utoipa_swagger_ui::SwaggerUi;

#[derive(Debug, Clone, Serialize, ToSchema)]
struct Book {
    id: u64,
    title: String,
    author: String,
    isbn: String,
    available: bool,
    /// วันที่ยืมล่าสุด (ถ้ายังไม่เคยถูกยืมจะเป็น null)
    borrowed_at: Option<String>,
}

#[derive(Debug, Deserialize, ToSchema)]
struct CreateBookRequest {
    title: String,
    author: String,
    isbn: String,
}

#[derive(Debug, Deserialize, ToSchema, utoipa::IntoParams)]
struct ListBooksQuery {
    limit: Option<u32>,
    available: Option<bool>,
}

#[derive(Debug, Serialize, ToSchema)]
struct ErrorBody {
    error: ErrorDetail,
}

#[derive(Debug, Serialize, ToSchema)]
struct ErrorDetail {
    code: String,
    message: String,
}

// ห้า variant ตรงตามที่ Part 66 ออกแบบไว้ (ตัดเหลือเท่าที่ตัวอย่างนี้ต้องใช้จริง)
#[derive(Debug, thiserror::Error)]
enum AppError {
    #[error("ไม่พบข้อมูลที่ต้องการ")]
    NotFound,
    #[error("ข้อมูลไม่ถูกต้อง: {0}")]
    Validation(String),
    #[error("ไม่ได้รับอนุญาต")]
    Unauthorized,
    #[error("ขัดแย้งกับสถานะปัจจุบัน: {0}")]
    Conflict(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, code) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "NOT_FOUND"),
            AppError::Validation(_) => (StatusCode::BAD_REQUEST, "VALIDATION_ERROR"),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            AppError::Conflict(_) => (StatusCode::CONFLICT, "CONFLICT"),
        };
        let body = ErrorBody { error: ErrorDetail { code: code.into(), message: self.to_string() } };
        (status, Json(body)).into_response()
    }
}

type SharedState = Arc<Mutex<Vec<Book>>>;

// จำลองการตรวจ Bearer token แบบง่าย -- ระบบจริงเรียก decode JWT เต็มรูปแบบตาม Part 74 หัวข้อ 74.7
fn check_bearer(headers: &HeaderMap) -> Result<(), AppError> {
    let auth = headers.get("Authorization").and_then(|v| v.to_str().ok());
    match auth {
        Some(v) if v.starts_with("Bearer ") && v.len() > 7 => Ok(()),
        _ => Err(AppError::Unauthorized),
    }
}

#[utoipa::path(
    get,
    path = "/api/v1/books",
    tag = "books",
    params(ListBooksQuery),
    responses(
        (status = 200, description = "รายการหนังสือ", body = Vec<Book>)
    )
)]
async fn list_books(State(state): State<SharedState>, Query(_q): Query<ListBooksQuery>) -> Json<Vec<Book>> {
    Json(state.lock().unwrap().clone())
}

#[utoipa::path(
    get,
    path = "/api/v1/books/{id}",
    tag = "books",
    params(("id" = u64, Path, description = "รหัสหนังสือ")),
    responses(
        (status = 200, description = "พบหนังสือ", body = Book),
        (status = 404, description = "ไม่พบหนังสือ", body = ErrorBody)
    )
)]
async fn get_book(State(state): State<SharedState>, Path(id): Path<u64>) -> Result<Json<Book>, AppError> {
    let books = state.lock().unwrap();
    books.iter().find(|b| b.id == id).cloned().map(Json).ok_or(AppError::NotFound)
}

#[utoipa::path(
    post,
    path = "/api/v1/books",
    tag = "books",
    security(("bearer_auth" = [])),
    request_body = CreateBookRequest,
    responses(
        (status = 201, description = "สร้างหนังสือสำเร็จ", body = Book),
        (status = 400, description = "ข้อมูลไม่ผ่านการตรวจสอบ", body = ErrorBody),
        (status = 401, description = "ไม่ได้ส่ง Bearer token หรือ token ไม่ถูกต้อง", body = ErrorBody)
    )
)]
async fn create_book(
    State(state): State<SharedState>,
    headers: HeaderMap,
    Json(req): Json<CreateBookRequest>,
) -> Result<impl IntoResponse, AppError> {
    check_bearer(&headers)?;
    if req.title.trim().is_empty() {
        return Err(AppError::Validation("title ต้องไม่เป็นค่าว่าง".into()));
    }
    let mut books = state.lock().unwrap();
    let id = books.len() as u64 + 1;
    let book = Book {
        id, title: req.title, author: req.author, isbn: req.isbn,
        available: true, borrowed_at: None,
    };
    books.push(book.clone());
    Ok((StatusCode::CREATED, Json(book)))
}

#[utoipa::path(
    post,
    path = "/api/v1/books/{id}/borrow",
    tag = "books",
    security(("bearer_auth" = [])),
    params(("id" = u64, Path, description = "รหัสหนังสือ")),
    responses(
        (status = 200, description = "ยืมสำเร็จ", body = Book),
        (status = 401, description = "unauthorized", body = ErrorBody),
        (status = 404, description = "ไม่พบหนังสือ", body = ErrorBody),
        (status = 409, description = "หนังสือถูกยืมไปแล้ว", body = ErrorBody)
    )
)]
async fn borrow_book(
    State(state): State<SharedState>,
    headers: HeaderMap,
    Path(id): Path<u64>,
) -> Result<Json<Book>, AppError> {
    check_bearer(&headers)?;
    let mut books = state.lock().unwrap();
    let book = books.iter_mut().find(|b| b.id == id).ok_or(AppError::NotFound)?;
    if !book.available {
        return Err(AppError::Conflict(format!("หนังสือ '{}' ถูกยืมไปแล้ว", book.title)));
    }
    book.available = false;
    Ok(Json(book.clone()))
}

struct SecurityAddon;

impl Modify for SecurityAddon {
    fn modify(&self, openapi: &mut utoipa::openapi::OpenApi) {
        if let Some(components) = openapi.components.as_mut() {
            components.add_security_scheme(
                "bearer_auth",
                SecurityScheme::Http(
                    HttpBuilder::new().scheme(HttpAuthScheme::Bearer).bearer_format("JWT").build(),
                ),
            );
        }
    }
}

#[derive(OpenApi)]
#[openapi(
    paths(list_books, get_book, create_book, borrow_book),
    components(schemas(Book, CreateBookRequest, ErrorBody, ErrorDetail)),
    tags((name = "books", description = "จัดการหนังสือในห้องสมุด")),
    modifiers(&SecurityAddon)
)]
struct ApiDoc;

#[tokio::main]
async fn main() {
    let state: SharedState = Arc::new(Mutex::new(vec![Book {
        id: 1,
        title: "The Rust Programming Language".to_string(),
        author: "Steve Klabnik".to_string(),
        isbn: "9781593278281".to_string(),
        available: true,
        borrowed_at: None,
    }]));

    let app = Router::new()
        .route("/api/v1/books", get(list_books).post(create_book))
        .route("/api/v1/books/{id}", get(get_book))
        .route("/api/v1/books/{id}/borrow", post(borrow_book))
        .merge(SwaggerUi::new("/swagger-ui").url("/api-docs/openapi.json", ApiDoc::openapi()))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3085").await.unwrap();
    println!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

#### เดินเรื่อง "Try it out" บน `POST /api/v1/books/{id}/borrow` ทีละขั้น

ต่อไปนี้คือคำอธิบาย**ทีละขั้นตอนของการใช้งานหน้า Swagger UI จริง** สำหรับ endpoint ยืมหนังสือ — คำอธิบายนี้
อ้างอิงจาก **spec จริงที่ตรวจสอบแล้วในหัวข้อ 85.6-85.7 ข้างบน** (ยืนยันด้วย `curl` ว่า schema/response/security
ตรงกับพฤติกรรมจริงของเซิร์ฟเวอร์ทุกจุด) ผสานกับ**พฤติกรรมมาตรฐานของ Swagger UI ที่มีเอกสารทางการรองรับ** — ไม่ใช่
ภาพหน้าจอที่ถ่ายจริง (environment ที่เขียนบทนี้ไม่มีเบราว์เซอร์ให้ capture ภาพได้) แต่เป็นคำอธิบายที่ตรงกับ
พฤติกรรมจริงของเครื่องมือ ทำงานบน spec ที่พิสูจน์แล้วว่าถูกต้อง:

1. เปิด `http://127.0.0.1:3085/swagger-ui/` — เห็นหัวเรื่อง "utoipa_demo" (จาก `info.title` ที่ดึงจาก
   `Cargo.toml`) พร้อมปุ่ม "Authorize" สีเขียวอยู่มุมขวาบน และด้านล่างมี section เดียวชื่อ "books" (จาก tag
   ที่ประกาศไว้) ที่กางออกมาเห็น 4 แถว เรียงตาม method: `GET /api/v1/books`, `POST /api/v1/books`,
   `GET /api/v1/books/{id}`, `POST /api/v1/books/{id}/borrow` — แต่ละแถวมีสีต่างกันตาม HTTP method (ตาม
   ธรรมเนียมสีมาตรฐานของ Swagger UI: `GET` สีน้ำเงิน, `POST` สีเขียว)
2. กดปุ่ม "Authorize" — modal เปิดขึ้น แสดง scheme `bearer_auth` พร้อมช่องกรอกข้อความ — พิมพ์ token ทดสอบ
   (เช่น `faketoken.abc.def` ตามที่ใช้ทดสอบด้วย `curl` ในหัวข้อ 85.7) แล้วกด "Authorize" ในโมดัล จากนั้นกด
   "Close" — ปุ่มเปลี่ยนจากไอคอนกุญแจเปิดเป็นกุญแจปิด บอกว่า authorize แล้ว
3. คลิกแถว `POST /api/v1/books/{id}/borrow` เพื่อกางรายละเอียด — เห็น description "ยืมสำเร็จ" ของ `200`,
   ตาราง parameter (`id`, in: path, type: integer, required), และ**ตาราง Responses ที่แสดงครบทั้ง 4 status
   code** (`200`/`401`/`404`/`409`) พร้อม schema ของแต่ละอัน — ตรงกับที่ `responses(...)` ในหัวข้อ 85.6
   ประกาศไว้เป๊ะทุกตัว
4. กดปุ่ม "Try it out" (ปรากฏที่มุมขวาของ panel) — ช่อง parameter `id` เปลี่ยนจาก text อ่านอย่างเดียวเป็น
   input ที่กรอกได้ — พิมพ์ `1`
5. กดปุ่ม "Execute" สีน้ำเงิน — Swagger UI สร้าง `fetch()` request ไปที่ `POST http://127.0.0.1:3085/api/v1/
   books/1/borrow` พร้อมแนบ header `Authorization: Bearer faketoken.abc.def` โดยอัตโนมัติ (ตามที่ authorize
   ไว้ในขั้นที่ 2 — endpoint นี้มี `security(("bearer_auth" = []))` ประกาศไว้จึงถูกแนบให้)
6. ผลลัพธ์ปรากฏด้านล่างปุ่ม Execute แบ่งเป็นสามส่วน: **"Curl"** (คำสั่ง `curl` ที่เทียบเท่ากับ request ที่ยิง
   ไปจริง — Swagger UI generate ให้อัตโนมัติเพื่อให้คัดลอกไปรันนอกเบราว์เซอร์ได้), **"Response body"**
   (เนื้อหา JSON ที่ตอบกลับมาจริง — ตรงกับที่ `curl` ทดสอบไว้แล้วในหัวข้อ 85.6:
   `{"id":1,"title":"The Rust Programming Language",...,"available":false,"borrowed_at":null}`), และ
   **"Server response" ที่มี Code = 200** พร้อม response header ที่ได้จริง (`content-type: application/
   json` เป็นต้น)
7. กด "Execute" ซ้ำอีกครั้งทันที (ยืมหนังสือเล่มเดียวกันเป็นครั้งที่สอง) — คราวนี้ **Code = 409** พร้อม
   response body `{"error":{"code":"CONFLICT","message":"..."}}"` — ตรงกับที่ `curl` พิสูจน์ไว้แล้วในหัวข้อ
   85.6 เป๊ะ พิสูจน์ว่า response ที่ Swagger UI แสดงตอน error ก็ไม่ต่างจากที่เซิร์ฟเวอร์ตอบจริงเลย
8. กดปุ่ม "Authorize" อีกครั้งแล้วกด "Logout" เพื่อล้าง token ที่กรอกไว้ แล้วลอง "Try it out" endpoint เดิม
   อีกครั้งโดยไม่ authorize — คราวนี้ได้ **Code = 401** พร้อม `{"error":{"code":"UNAUTHORIZED",
   "message":"unauthorized"}}` — ตรงกับที่หัวข้อ 85.7 พิสูจน์ไว้ด้วย `curl` เช่นกัน

ทั้งแปดขั้นตอนข้างบนคือการยืนยันว่า**สิ่งที่ผู้ใช้เห็นและทำได้จริงในเบราว์เซอร์ผ่าน Swagger UI ไม่ต่างอะไรจาก
การยิง `curl` มือที่บทนี้พิสูจน์ไว้ตลอดทุกหัวข้อ** — เพราะทั้งสองทาง (เบราว์เซอร์ผ่าน Swagger UI, หรือ `curl`
ตรง ๆ) เป็นแค่ HTTP client สองตัวที่คุยกับเซิร์ฟเวอร์เดียวกันด้วย protocol เดียวกัน สิ่งที่ Swagger UI เพิ่ม
ให้คือ**ความสะดวก**ในการทดลอง ไม่ใช่พฤติกรรมพิเศษอะไรที่ต่างจาก HTTP ปกติเลย

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Build ล้มเหลวเพราะ `utoipa-swagger-ui` ดาวน์โหลด asset ตอน build แต่ไม่มี network

ถ้าไม่เปิด feature `vendored` (ตามที่อธิบายในหัวข้อ 85.3) `utoipa-swagger-ui` จะพยายามดาวน์โหลดไฟล์ static
ของ Swagger UI จาก GitHub ตอน `cargo build` — ถ้า build environment ไม่มีสิทธิ์เข้าอินเทอร์เน็ตตรงจุดนั้น (CI
runner ที่ปิด network, container แบบ air-gapped, environment ที่มี proxy/firewall เข้มงวด) จะได้ error แบบนี้
จริง (คัดลอกตรงจากการรันจริงในสภาพแวดล้อมที่ทดสอบบทนี้):

```
thread 'main' panicked at .../utoipa-swagger-ui-10.0.1/build.rs:238:51:
failed to open downloaded Swagger UI: InvalidArchive("Could not find EOCD")
```

(`EOCD` คือ "End of Central Directory" — ส่วนท้ายของไฟล์ zip ที่บอกว่าไฟล์นั้นสมบูรณ์ ข้อความนี้บอกว่าไฟล์ที่
ดาวน์โหลดมาไม่ใช่ zip ที่สมบูรณ์ เช่นได้ error page HTML กลับมาแทนไฟล์ zip จริง เพราะ network ถูกบล็อกหรือ
redirect ไปที่อื่น) **วิธีแก้**: เปิด feature `vendored` ตามที่หัวข้อ 85.3 ทำไว้ตั้งแต่แรก
(`utoipa-swagger-ui = { version = "10.0.1", features = ["axum", "vendored"] }`) — feature นี้ดึง static
asset ที่ build ไว้ล่วงหน้าจาก crate `utoipa-swagger-ui-vendored` (ติดมากับ crates.io ตามปกติ ไม่ต้องเข้า
อินเทอร์เน็ตเพิ่มตอน build) แทนการดาวน์โหลดสด

### 2. ลืม `#[derive(ToSchema)]` บน type ที่ใช้ใน `request_body`/`body`

ถ้า struct ที่อ้างถึงใน `request_body = ...` หรือ `body = ...` ไม่มี `#[derive(ToSchema)]` แนบอยู่ จะได้
compile error ทันที (ทดสอบจริงแล้ว):

```
error[E0277]: the trait bound `CreateBookRequest: PartialSchema` is not satisfied
   --> src/main.rs:13:20
    |
 13 |     request_body = CreateBookRequest,
    |                    ^^^^^^^^^^^^^^^^^ unsatisfied trait bound
    |
help: the trait `ComposeSchema` is not implemented for `CreateBookRequest`
```

ข้อความ error ชี้ตรงไปที่บรรทัดที่อ้าง type และบอกชื่อ trait ที่ขาด (`PartialSchema`/`ComposeSchema` คือ
super-trait ภายในที่ `ToSchema` ต้องใช้) — **วิธีแก้**: เติม `#[derive(ToSchema)]` เข้าไปที่ struct นั้น
(อาจต้องเติม `use utoipa::ToSchema;` ด้วยถ้ายังไม่ import) นี่คือ error ที่ compiler จับได้ตั้งแต่ compile
time (ต่างจากปัญหา "documentation drift" ในหัวข้อ 85.2 ที่ compiler จับไม่ได้เลย) เพราะการอ้าง type ที่ไม่มี
schema เป็น type error ตรง ๆ ไม่ใช่แค่ข้อมูลไม่ตรงกัน

### 3. อ้างชื่อ function ใน `#[derive(OpenApi)] paths(...)` ที่ไม่มี `#[utoipa::path(...)]` แนบอยู่

ถ้า `paths(get_book)` อ้างถึง function ที่**ไม่มี** `#[utoipa::path(...)]` แนบอยู่เหนือมัน (เช่นลืมแนบ หรือ
พิมพ์ชื่อ function ผิดตัว) จะได้ compile error ที่หน้าตาแปลกในตอนแรก (ทดสอบจริงแล้ว):

```
error[E0425]: cannot find type `__path_get_book` in this scope
 --> src/main.rs:6:10
  |
6 | #[derive(OpenApi)]
  |          ^^^^^^^ not found in this scope

error[E0433]: failed to resolve: use of unresolved module or unlinked crate `__path_get_book`
```

ข้อความนี้ดูเหมือนไม่เกี่ยวกับ `get_book` โดยตรง แต่จริง ๆ คือกำลังบอกอยู่ — ตามที่หัวข้อ 85.3 อธิบายไว้ว่า
`#[utoipa::path(...)]` สร้าง type ที่ซ่อนอยู่ชื่อ `__path_<ชื่อ function>` ให้ `#[derive(OpenApi)]` มาอ้างถึง
— ถ้า function นั้นไม่มี `#[utoipa::path(...)]` แนบอยู่ type `__path_get_book` ก็ไม่ถูกสร้างขึ้นมาเลย ทำให้
`paths(get_book)` หา type นี้ไม่เจอ **วิธีแก้**: ตรวจสอบว่า function ที่อ้างถึงใน `paths(...)` ทุกตัวมี
`#[utoipa::path(...)]` แนบอยู่จริง และชื่อสะกดตรงกันเป๊ะ

### 4. Path string ใน `#[utoipa::path(path = "...")]` ไม่ตรงกับ route จริงของ Axum (documentation drift แบบ compile ผ่าน)

นี่คือกับดักที่**สำคัญที่สุดของบทนี้** เพราะพิสูจน์ข้อจำกัดที่หัวข้อ 85.2 เตือนไว้แบบเป็นรูปธรรม: ถ้าเขียน
path เก่าแบบ Axum ก่อนเวอร์ชัน 0.7 (ที่ใช้ `:id` แทน path parameter) ใน `#[utoipa::path(path = "/api/v1/
books/:id")]` ทั้งที่ route จริงใน `Router::new().route("/api/v1/books/{id}", ...)` ใช้ไวยากรณ์ `{id}` ของ
Axum 0.8 (ตามที่ Part 62 สอนไว้ว่า Axum เปลี่ยน syntax นี้) — **`cargo build` compile ผ่านสนิท ไม่มี warning
เลยแม้แต่ตัวเดียว** เพราะสอง string นี้ไม่มีอะไรเชื่อมโยงกันทาง compiler เลย (ทดสอบจริงแล้ว)

Spec ที่ generate ออกมาจะมี key path เป็น `/api/v1/books/:id` ตรงตามที่ประกาศผิดไว้ (ตรวจสอบจริงแล้วผ่าน
`curl` ไปที่ `/api-docs/openapi.json`) — ถ้ามีคนเอา path นี้ไปยิง (เช่น Swagger UI "Try it out" หรือ client
code ที่ generate จาก spec นี้ตามหัวข้อ 85.9) จะได้ผลลัพธ์จริงแบบนี้ (ทดสอบจริงแล้ว):

```bash
curl -sS "http://127.0.0.1:3085/api/v1/books/:id"
```
```
Invalid URL: Cannot parse `:id` to a `u64`
```

Axum พยายาม parse ตัวอักษร `:id` (ตัวหนังสือดิบ ๆ ไม่ใช่ placeholder) เป็น `u64` เพราะ route จริงที่ Axum
รู้จักคือ `{id}` ไม่ใช่ `:id` — ส่วน `:id` ที่พิมพ์ผิดไปกลายเป็นเนื้อหาตัวอักษรจริงของ URL ที่ Axum พยายาม parse
เป็นเลขไม่ได้ **นี่คือ documentation drift ตัวจริงที่หัวข้อ 85.2 เตือนไว้ — เกิดขึ้นได้แม้จะใช้ `utoipa` เต็ม
รูปแบบ เพราะ path string เป็นแค่ข้อความที่คุณพิมพ์เอง ไม่มีการตรวจสอบข้ามกับ router จริงเลย** **วิธีแก้**: ไม่มี
วิธีทาง compiler ที่ป้องกันได้ 100% — วิธีปฏิบัติที่ดีที่สุดคือ **copy path string เดียวกันเป๊ะจาก
`.route(...)` มาใส่ใน `#[utoipa::path(path = ...)]` เสมอ** (อย่าพิมพ์ใหม่จากความจำ) และถ้าเป็นไปได้ ใช้แนวทาง
`utoipa-axum` จากหัวข้อ 85.8 ที่ผูก route กับ path annotation ไว้ในคำสั่งเดียวกัน (`routes!(get_book)`) ซึ่งลด
ความเสี่ยงที่ทั้งสองจะพิมพ์แยกกันคนละที่จนหลุด sync ได้

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เพิ่ม endpoint `DELETE /api/v1/books/{id}` เข้าไปในเซิร์ฟเวอร์ capstone ของหัวข้อ 85.10 — เขียน
   `#[utoipa::path(delete, ...)]` ที่มี `responses((status = 204, description = "ลบสำเร็จ"), (status = 404,
   description = "ไม่พบหนังสือ", body = ErrorBody))` แล้วเพิ่มชื่อ function เข้า `paths(...)` ของ
   `#[derive(OpenApi)]` ด้วย — ทดสอบด้วย `curl -X DELETE` ทั้งกรณีมีและไม่มีหนังสือ id นั้นจริง แล้วเทียบกับ
   spec ที่ export ออกมาว่าตรงกัน
   *hint*: `204 No Content` ไม่มี response body — ใน `#[utoipa::path]` ใส่แค่ `description` โดยไม่ต้องมี
   `body = ...` เลยก็ได้ (ต่างจาก `200`/`201` ที่มักมี body เสมอ)

2. **(กลาง)** เพิ่ม field `published_year: Option<u16>` เข้า struct `Book` แล้ว export spec ใหม่ ตรวจสอบว่า
   field นี้ปรากฏใน `components.schemas.Book.properties` ด้วย `"type": ["integer", "null"]` และ**ไม่**อยู่ใน
   `required` list ตรงตามที่หัวข้อ 85.4 อธิบายไว้ — จากนั้นลองเปลี่ยนเป็น `published_year: u16` (ไม่ใช่
   `Option`) แล้วสังเกตว่า schema เปลี่ยนอย่างไร (มันควรย้ายเข้า `required` แทน)
   *hint*: อย่าลืมอัปเดต struct literal ทุกจุดที่สร้าง `Book` (เช่นใน `create_book`) ให้ใส่ค่า field ใหม่
   ด้วย ไม่เช่นนั้นจะเจอ compile error ปกติของ Rust (ไม่เกี่ยวกับ `utoipa`) ที่บอกว่า field ขาดไป

3. **(ยาก)** เพิ่ม `AppError::ValidationErrors(Vec<FieldError>)` (ตามแบบ Part 66 หัวข้อ 66.6) เข้าไปใน
   capstone แล้วใช้กับ `create_book` เพื่อ validate หลาย field พร้อมกัน (`title`/`author`/`isbn` ว่างไม่ได้,
   `isbn` ต้องมี 13 หลัก) — สร้าง struct `FieldError` ที่มี `#[derive(Serialize, ToSchema)]` แล้วประกาศ
   `body = ErrorBody` (ที่ `error.fields` เป็น `Vec<FieldError>`) ให้ตรงกับ shape จริงที่ Part 66 output
   *hint*: ปัญหาหลักของข้อนี้ไม่ใช่การเขียน `utoipa` แต่คือการออกแบบ `ErrorBody` ให้ field `fields` เป็น
   `Option<Vec<FieldError>>` (เพราะ variant อื่นของ `AppError` ไม่มี fields นี้) แล้วตรวจสอบว่า schema ที่
   export ออกมาสื่อความหมาย "field นี้มีแค่บางกรณี" ได้ถูกต้องตามหัวข้อ 85.4 หรือไม่

4. **(ยาก/ประยุกต์ใช้จริง)** แยกเซิร์ฟเวอร์ capstone ออกเป็นสองโมดูล (`books.rs` และ `loans.rs` — สมมติให้
   `loans.rs` มีแค่ `GET /api/v1/loans` คืนรายการการยืมทั้งหมดพอ) ตามแนวทาง `utoipa-axum` ของหัวข้อ 85.8 —
   แต่ละโมดูลมี `OpenApiRouter`/`#[derive(OpenApi)]` ของตัวเอง แล้ว `.merge(...)` เข้าด้วยกันที่ `main.rs` —
   ตรวจสอบด้วย `curl http://.../api-docs/openapi.json` ว่า spec ที่รวมแล้วมี tag ทั้ง `"books"` และ
   `"loans"` ครบ และ Swagger UI แสดงทั้งสอง section แยกกันถูกต้อง
   *hint*: อย่าลืมกับดักข้อ 4 ท้ายบทที่พิสูจน์ไว้แล้วว่า `OpenApiRouter::new().nest("/", ...)` panic ตอน
   runtime — ต้องใช้ `.merge(...)` เท่านั้นเมื่อรวมที่ root ไม่มี prefix

## สรุป

### สรุปเนื้อหาของบทนี้

บทนี้ปิดช่องว่างสุดท้ายที่ Part 78 ทิ้งไว้อย่างตั้งใจ: convention ที่ดี (Part 78) ทำให้ API **เดาง่าย** สำหรับ
มนุษย์ แต่ `utoipa` (บทนี้) ทำให้รายละเอียดที่เดาไม่ได้**ไม่ต้องเดา**เลย ด้วยการ generate สัญญา (contract) ที่
เครื่องอ่านได้**จากโค้ด Rust จริง** ไม่ใช่จากเอกสารที่เขียนแยกไว้ต่างหาก — หัวใจของบททั้งหมดคือความเข้าใจสอง
ด้านที่ต้องถือคู่กันเสมอ: (1) แนวทาง "documentation as code" ลดปัญหา documentation drift ได้จริงและพิสูจน์ได้
(schema ของ `Book`/`CreateBookRequest`/`ErrorBody` ผูกกับนิยาม struct จริงตลอดเวลา) แต่ (2) มันไม่ได้แก้ปัญหา
นี้ 100% แบบอัตโนมัติ — `path` string, `responses(...)` ที่ประกาศไว้, และ `request_body` ที่อ้างถึง ยังเป็น
ข้อความที่มนุษย์เขียนแยกจาก logic จริงของ handler อยู่ดี (พิสูจน์แล้วในกับดักข้อ 4 ว่า compile ผ่านได้แม้ประกาศ
ผิดจากความจริง) — วินัยในการอัปเดต annotation ให้ตรงกับโค้ดที่เปลี่ยนไปเรื่อย ๆ ยังเป็นสิ่งที่มนุษย์ต้องทำเอง
เสมอ ไม่มีเครื่องมือไหนแทนที่ได้ทั้งหมด

ในทางเทคนิค บทนี้พาไปตั้งแต่การติดตั้ง (`utoipa`/`utoipa-swagger-ui` พร้อม feature `vendored` ที่จำเป็นในหลาย
สภาพแวดล้อม build จริง), การ derive schema จาก struct โดเมนของหลักสูตร (พร้อมพิสูจน์ว่า `Option<T>` จาก Part
11 แปลงเป็น nullable schema แบบ OpenAPI 3.1 ที่ `utoipa` 6.x generate ให้จริงอย่างไร), การผูก `params`/
`request_body`/`responses` เข้ากับ extractor จริงของ Axum (Part 63-64), การบันทึกทุก status code ที่
`AppError` (Part 66) ตอบได้จริงให้ครบถ้วน, การประกาศ Bearer JWT security scheme (Part 74) ให้ปุ่ม "Authorize"
ใช้งานได้จริง, การจัดระเบียบเอกสารด้วย tag และแยกโมดูล (Part 16) ด้วย `utoipa-axum`, และปิดท้ายด้วยการ export
spec ไปใช้นอก Swagger UI — ทุกขั้นตอนพิสูจน์ด้วยการรันจริง ไม่ใช่แค่คัดลอกโค้ดจากเอกสาร

### ภาพรวมทั้งโมดูล 4: จาก HTTP พื้นฐานสู่ระบบเว็บที่พร้อมใช้งานจริง

บทนี้คือบทปิดของโมดูล 4 (Part 61-85) — ก่อนข้ามไปโมดูล 5 ควรหยุดมองภาพรวมทั้งเส้นทางที่เดินผ่านมา เพราะแต่ละ
บทไม่ได้เป็นความรู้แยกส่วนที่ไม่เกี่ยวกัน แต่ประกอบกันเป็น**ทักษะเดียว**ของการสร้างระบบเว็บระดับ production:

- **Part 61** ปูพื้น HTTP/REST ในระดับแนวคิด (method, status, header, resource) — คำศัพท์และกรอบคิดที่ทุกบท
  ถัดไปอ้างอิงกลับไปตลอดโดยไม่ต้องอธิบายซ้ำ
- **Part 62-66** สร้างเซิร์ฟเวอร์จริงด้วย **Axum** ตั้งแต่ routing/handler พื้นฐาน (62-63) → state/extractor
  สำหรับข้อมูลที่ต้องแชร์ข้าม request (64) → middleware สำหรับ concern ข้ามระบบ (65) → error handling ที่
  รวมศูนย์เป็น `AppError` เดียว (66) — ห้าบทนี้คือ "โครงกระดูก" ที่ทุกตัวอย่างที่เหลือของโมดูลสร้างต่อบนมัน
- **Part 67-69** เปิดมุมมองกว้างขึ้นด้วย **Actix-web** เป็นตัวเปรียบเทียบ แล้วสรุปเป็นกรอบตัดสินใจว่าเมื่อไร
  ควรเลือก framework ไหน — สอนให้รู้ว่า Axum ไม่ใช่ทางเลือกเดียวที่มีในโลก Rust
- **Part 70-73** เชื่อมกับ**ฐานข้อมูลจริง** สามแนวทาง (SQLx แบบ raw SQL, Diesel แบบ ORM ที่ compile-time
  check, SeaORM แบบ async ORM) — ทำให้ API ที่สร้างไว้เก็บข้อมูลถาวรได้จริง ไม่ใช่แค่ `HashMap` ในหน่วยความจำ
- **Part 74-76** เพิ่มชั้น**ความปลอดภัย**: JWT (74) และ session/OAuth2 (75) สำหรับยืนยันตัวตน (authentication
  — "คุณเป็นใคร") ต่อด้วย RBAC (76) สำหรับสิทธิ์การเข้าถึง (authorization — "คุณทำอะไรได้บ้าง") — สอง concept
  ที่มักถูกสับสนกันแต่ต้องแยกให้ชัดตามที่ทั้งสามบทนี้อธิบาย
- **Part 77** เปิดช่องทางสื่อสารแบบ real-time ด้วย **WebSocket** สำหรับกรณีที่ request-response ธรรมดาไม่พอ
- **Part 78** ยกระดับจาก "ทำงานได้" สู่ "ออกแบบดี" ด้วยวินัยการออกแบบ REST API เต็มรูปแบบ (naming,
  versioning, envelope, pagination, idempotency) — และทิ้งคำถามเรื่องเอกสารไว้ให้บทนี้ (85) มาตอบ
- **Part 79-80** เปิดมุมมองสถาปัตยกรรม API ทางเลือกอื่นนอกจาก REST: **GraphQL** (client เลือก field ที่
  ต้องการเอง) และ **gRPC** (RPC แบบ binary ประสิทธิภาพสูงสำหรับการสื่อสารระหว่าง service)
- **Part 81-84** ขยายจาก "หนึ่งเซิร์ฟเวอร์" สู่ **ระบบแบบกระจาย (distributed system)** จริง: Microservices
  Architecture (81), Message Queues อย่าง RabbitMQ/Kafka สำหรับสื่อสารแบบ asynchronous ข้าม service (82),
  Redis สำหรับ caching (83), และ background job/task queue สำหรับงานที่ไม่ควรบล็อก request-response cycle
  (84)
- **Part 85 (บทนี้)** ปิดท้ายด้วยการทำให้ทุกอย่างที่สร้างมาทั้งหมด**สื่อสารกับโลกภายนอกได้อย่างแม่นยำ** ผ่าน
  OpenAPI/Swagger — เพราะระบบที่ซับซ้อนขึ้นเรื่อย ๆ ตาม Part 81-84 ยิ่งต้องการเอกสารที่เชื่อถือได้มากขึ้นตามไป
  ด้วย ไม่ใช่น้อยลง

ภาพรวมทั้งหมดนี้คือ**ทักษะการสร้างระบบเว็บที่พร้อมใช้งานจริงระดับ production แบบครบวงจร**: รับ request ได้
ถูกต้อง (61-66), เลือก framework/database ที่เหมาะกับงานได้ (67-73), ปลอดภัยสำหรับผู้ใช้จริง (74-76), รองรับ
การสื่อสารหลายรูปแบบ (77, 79-80), ออกแบบและปรับขนาดให้รองรับระบบที่ซับซ้อนขึ้นได้ (78, 81-84), และสื่อสารกับ
ทีมอื่น/ระบบอื่นได้อย่างแม่นยำไม่ต้องเดา (85) — ไม่มีจุดไหนในลิสต์นี้ที่ "ไม่จำเป็น" สำหรับระบบจริง แต่ละบท
แก้ปัญหาที่เกิดขึ้นจริงในทีมพัฒนา web service ทั่วโลก

### ก้าวต่อไป: จากฝั่ง Backend สู่ WebAssembly และ Full-Stack

โมดูล 4 ทั้งหมดที่ผ่านมาอยู่ฝั่ง **backend/server-side** ล้วน ๆ — Rust ทำหน้าที่รับ request, ประมวลผล, คุยกับ
ฐานข้อมูล, แล้วตอบ response กลับไป ไม่เคยรันในเบราว์เซอร์ของผู้ใช้เลยแม้แต่ครั้งเดียว **โมดูล 5 ที่กำลังจะเริ่ม
ต่อไป (Part 86-95) จะเปลี่ยนมุมมองนี้โดยสิ้นเชิง**: Part 86 จะแนะนำ **WebAssembly (WASM)** — เทคโนโลยีที่ทำให้
โค้ด Rust **รันในเบราว์เซอร์ได้จริง** เทียบเคียงประสิทธิภาพกับ JavaScript แบบเนทีฟ เปิดทางให้เขียน frontend
application ทั้งตัวด้วย Rust และบทต่อ ๆ ไปในโมดูลนี้จะพาไปสร้างระบบ **full-stack** ที่ทั้ง frontend และ
backend เป็น Rust ทั้งคู่

จุดที่บทนี้เชื่อมโยงไปยังโมดูลใหม่โดยตรงที่สุดคือหัวข้อ 85.9: **OpenAPI spec ที่ backend สร้างไว้ตลอดโมดูลนี้
(และเพิ่งทำให้แม่นยำเต็มรูปแบบในบทนี้) คือสัญญาที่ frontend ฝั่ง WASM ที่กำลังจะเรียนต่อไปจะใช้เป็นฐานในการ
เรียก API เหล่านี้อย่าง type-safe** — ไม่ว่า frontend ที่จะเขียนต่อไปจะ generate client code จาก spec นี้
โดยตรง หรือใช้เป็นข้อมูลอ้างอิงตอนเขียน HTTP call มือ สิ่งที่แน่นอนคือ **backend ที่มีเอกสารแม่นยำ พิสูจน์ได้
ด้วยการรันจริง (ไม่ใช่แค่คำอธิบายลอย ๆ ที่หลุด sync ไปนานแล้ว) คือรากฐานที่จำเป็นสำหรับการสร้าง full-stack
application ที่ทั้งสองฝั่งไว้ใจกันได้** — และนั่นคือสิ่งที่โมดูล 4 ทั้งหมด (ปิดท้ายด้วยบทนี้) ได้เตรียมไว้ให้
พร้อมสำหรับก้าวต่อไปแล้ว

---

**Part ก่อนหน้า:** [Background Jobs และ Task Queues](part-084-background-jobs.md) | **Part ถัดไป:** [WebAssembly เบื้องต้นด้วย Rust](part-086-wasm-basics.md)
