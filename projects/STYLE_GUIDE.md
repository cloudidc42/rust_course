# แนวทางการเขียนโปรเจค (Project Style Guide)

เอกสารนี้เป็นมาตรฐานสำหรับเขียนทุกโปรเจคใน `projects/` directory

## กฎทั่วไป

- ภาษาเนื้อหา: **ภาษาไทย** เป็นหลัก (คำศัพท์เทคนิค/ชื่อฟังก์ชัน/ชื่อ crate คงเป็นภาษาอังกฤษ)
- ความยาว: แต่ละโปรเจคควรมี **800–3000+ บรรทัด** — เนื้อหาต้องสร้างโปรเจคที่รันได้จริง 100%
- โค้ดทุกตัวอย่าง **ต้อง compile ได้จริง** ด้วย Rust edition 2021 ขึ้นไป
- ต้องมีขั้นตอนการพัฒนาแบบ **Progressive** — เริ่มจากโครงสร้างพื้นฐานแล้วค่อยเพิ่ม feature ทีละขั้น
- โค้ดทุกก้อนต้องมี real output จากการรันจริง (ไม่แต่งขึ้น)
- อ้างอิง Parts จากหลักสูตรหลัก (เช่น "จาก Part 46 เรื่อง async/await...")
- แต่ละโปรเจคต้องมี test ที่รันได้จริง ไม่ใช่แค่ unit test โครงร่าง

## โครงสร้างบังคับของแต่ละโปรเจค

```markdown
# Project {ID}: {ชื่อโปรเจค}

> โมดูล: {ชื่อโมดูล} | ความยาก: {⭐/⭐⭐/⭐⭐⭐/⭐⭐⭐⭐/⭐⭐⭐⭐⭐} | เวลาโดยประมาณ: {X} ชั่วโมง

## ภาพรวมโปรเจค

อธิบาย "ทำอะไร" และ "ทำไมถึงน่าสร้าง" — ไม่ใช่แค่ชื่อโปรเจค
บอก use case จริงในโลก production และ learning value ที่ได้

## สิ่งที่จะได้เรียนรู้

- (bullet list 4-8 ข้อ — ทักษะ/pattern/concept ที่โปรเจคนี้สอน)

## ความรู้ที่ต้องมีมาก่อน

- อ้างอิง Parts จากหลักสูตรหลักที่เกี่ยวข้อง

## โครงสร้างโปรเจค (Project Layout)

```
project-name/
├── src/
│   ├── main.rs
│   └── ...
├── tests/
│   └── ...
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

อธิบาย architecture เชิงลึก — data flow, module boundaries, design decisions
ทำไมถึงเลือก design นี้ ทางเลือกอื่นมีอะไร

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: {ชื่อขั้น}
คำอธิบาย + โค้ดที่ compile และรันได้จริง + output จริง

### ขั้นที่ 2: {ชื่อขั้น}
...

(ต้องมีอย่างน้อย 4-8 ขั้น ค่อย ๆ เพิ่มความซับซ้อน)

## การทดสอบ (Testing)

unit tests, integration tests, หรือ end-to-end tests ที่รันได้จริง
พร้อม real `cargo test` output

## การ Package และ Deploy

วิธี build release binary, Docker image, หรือ publish ตามความเหมาะสมของโปรเจค

## การต่อยอด (Extensions & Exercises)

1. Feature เพิ่มเติมที่น่าลองทำ (4-6 ข้อ, progressive difficulty)

## สรุป

สรุปสิ่งที่สร้าง, pattern สำคัญที่ได้เรียน, และเชื่อมโยงไปโปรเจคถัดไป

---

**โปรเจคก่อนหน้า:** [{ชื่อ}](project-{prev-id}.md) | **โปรเจคถัดไป:** [{ชื่อ}](project-{next-id}.md)
```

## การตั้งชื่อไฟล์

`projects/project-{MODULE_ID}{NN}-{english-kebab-slug}.md`

ตัวอย่าง:
- `projects/project-a01-shell-interpreter.md`
- `projects/project-b03-image-processing-api.md`
- `projects/project-j10-multiplayer-game.md`

Module ID: `a`=CLI/Systems, `b`=Web Services, `c`=Data Processing, `d`=Security/Crypto,
`e`=Games/Graphics, `f`=Distributed Systems, `g`=DevOps/Infra, `h`=Networking/Protocols,
`i`=ML/AI, `j`=Full-Stack/WASM

## ระดับความยาก (Difficulty)

- ⭐ — ใช้ความรู้ Parts 1-40 (พื้นฐาน-กลาง)
- ⭐⭐ — ใช้ความรู้ Parts 41-60 (ระดับสูง)
- ⭐⭐⭐ — ใช้ความรู้ Parts 61-95 (web dev / full-stack)
- ⭐⭐⭐⭐ — ใช้ความรู้ Parts 96-110 (production / DevOps)
- ⭐⭐⭐⭐⭐ — ต้องการความรู้จากหลายโมดูลรวมกัน + research เพิ่มเติม

## ข้อกำหนดเรื่องการ Verify โค้ด

**บังคับ** — ต้องสร้าง cargo project จริงใน scratchpad directory (**ห้ามสร้างใน repo**) แล้ว:
1. `cargo build` ผ่าน — ไม่มี compile error
2. `cargo test` ผ่าน — tests ที่เขียนไว้ต้อง pass จริง
3. รันโปรแกรมจริงด้วย sample input และ capture output จริง
4. ลบ scratch project ออกหลังเสร็จ
5. วาง real output ลงในเนื้อหา (ห้ามแต่งขึ้น)

## Nav Link

ไฟล์โปรเจคทั้งหมดอยู่ใน `projects/` directory เดียวกัน ลิงก์ prev/next
**ต้องเป็นชื่อไฟล์เปล่า ๆ** เช่น `project-a01-shell-interpreter.md`
**ห้ามใส่** `../projects/` หรือ path นำหน้าใด ๆ

## Checklist ก่อนส่ง

- [ ] โครงสร้างตาม template ครบ
- [ ] ≥ 800 บรรทัด (ควร 1200-3000+ สำหรับโปรเจคซับซ้อน)
- [ ] ทุก code block compile ได้จริง + มี real output
- [ ] มี test ที่ผ่านจริง (cargo test output verbatim)
- [ ] มีลิงก์ prev/next ท้ายโปรเจค (bare filename)
- [ ] อัปเดต projects/README.md ถ้าจำเป็น
