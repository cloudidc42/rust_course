# Part 105: Contributing to Open Source Rust Projects

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 240 นาที

## เป้าหมายของบทเรียน

- เข้าใจเหตุผลที่แท้จริงในการ contribute ให้ open source — ทั้งด้านการเรียนรู้, ชื่อเสียง, และการตอบแทนเครื่องมือที่ใช้ — โดยไม่หลอกตัวเองด้วยการโฟกัสที่ resume เพียงอย่างเดียว
- ประเมินได้ว่า project ไหน "friendly" ต่อ contributor หน้าใหม่ ก่อนเสียเวลาลงไป โดยดูจาก commit activity, การตอบสนองของ maintainer, และการมี `CONTRIBUTING.md`
- อ่านและทำความเข้าใจ codebase ขนาดใหญ่ที่ไม่คุ้นเคยได้อย่างมีระบบ โดยไม่ต้องพยายามเข้าใจทุกอย่างพร้อมกัน
- ใช้ git workflow แบบ fork + upstream + rebase ได้จริงด้วยคำสั่งที่ถูกต้อง ไม่ใช่แค่จำชื่อคำสั่งลอย ๆ
- เขียน commit message และ PR description ที่ reviewer อ่านแล้วเข้าใจ "อะไร" และ "ทำไม" ได้ทันที
- รัน `cargo fmt`, `cargo clippy`, และ test suite ให้ผ่านทั้งหมด**ก่อน**เปิด PR พร้อมรับมือกับ code review อย่างเป็นมืออาชีพ
- เข้าใจว่า `CONTRIBUTING.md`, Code of Conduct, license, และ CLA มีไว้ทำไม และทำไมการเคารพเอกสารเหล่านี้ทำให้ contribution ราบรื่นขึ้นมาก
- มองเห็นภาพรวมว่า "การ contribute" ไม่ได้จำกัดแค่การเขียนโค้ดใหม่ — เอกสาร, การ triage issue, และการ review PR ของคนอื่นก็เป็น contribution ที่มีค่าไม่แพ้กัน

## ความรู้ที่ต้องมีมาก่อน

- Part 5: `cargo fmt` และ `cargo clippy` — เครื่องมือพื้นฐานที่ทุก contribution ต้องผ่าน
- Part 16/17: Module และโครงสร้าง package/crate/workspace — จำเป็นสำหรับอ่านโครงสร้าง repo ขนาดใหญ่
- Part 34: Rustdoc — `cargo doc --open`, doc comment, doc test — ใช้เป็นเครื่องมือหลักในการอ่านโค้ดคนอื่นในบทนี้
- Part 97: CI/CD ด้วย GitHub Actions — เข้าใจว่า CI ของ project จริงมักตรวจอะไรบ้าง
- Part 100: `cargo audit` และ `cargo deny` — พื้นฐานสำหรับหัวข้อ license awareness ในบทนี้
- ความคุ้นเคยกับคำสั่ง git พื้นฐาน (`clone`, `branch`, `commit`, `push`) ในระดับที่ใช้งานได้จริงมาก่อนแล้ว

## เนื้อหา

### 105.1 ทำไมควร Contribute ให้ Open Source Rust Project

ก่อนจะพูดเรื่อง "วิธีการ" (how) เราต้องซื่อสัตย์กับ "เหตุผล" (why) ก่อน เพราะเหตุผลที่ผิดจะทำให้คุณ burnout เร็ว
และเหตุผลที่ถูกจะทำให้คุณทำต่อได้ในระยะยาวแม้ PR แรกจะถูกปฏิเสธ

**เหตุผลที่มักถูกพูดถึงแต่ไม่ใช่เหตุผลหลักที่ควรใช้ขับเคลื่อนตัวเอง:** "การมี contribution ใน open source ทำให้
resume ดูดี" — ประโยคนี้**เป็นความจริง**บางส่วน บริษัทจำนวนมากดู GitHub profile ของผู้สมัครจริง และการเห็นว่าคุณมี
PR ที่ merge เข้า project ที่มีชื่อ (เช่น `tokio`, `serde`, `ripgrep`) ก็เป็นสัญญาณที่ดีว่าคุณเขียนโค้ดคุณภาพระดับ
production ได้ แต่ถ้านี่เป็นเหตุผล**เดียว**ที่คุณ contribute คุณจะเลือก issue ที่ "ดูดีบน resume" มากกว่า issue ที่
project ต้องการจริง ๆ และจะรู้สึกแย่มากเมื่อ PR ถูก maintainer ขอให้แก้ไขหลายรอบ หรือถูกปิดโดยไม่ merge

เหตุผลที่ควรเป็นแรงขับเคลื่อนหลักมี 4 ข้อ และทุกข้อนี้**ยั่งยืนกว่า**เพราะให้ผลตอบแทนทันทีไม่ว่า PR จะถูก merge
หรือไม่:

**1. การเรียนรู้จากการอ่านโค้ด production-quality จริง** — โค้ดในหนังสือเรียนหรือ tutorial (รวมถึงบทเรียนในหลักสูตร
นี้) ถูกออกแบบมาให้ "เข้าใจง่าย" ซึ่งหมายความว่ามันถูกตัดความซับซ้อนที่ไม่จำเป็นออกไปโดยตั้งใจ แต่โค้ดจริงใน
project อย่าง `tokio` หรือ `axum` ต้องรับมือกับ edge case, backward compatibility, performance ในระดับ nanosecond,
และ API design ที่ต้องรองรับผู้ใช้นับล้านคนที่ใช้งานคนละแบบ การอ่านโค้ดแบบนี้สอนสิ่งที่ tutorial สอนไม่ได้ เช่น
วิธี structure error type ให้ทั้ง ergonomic และครอบคลุม หรือวิธีเขียน unsafe code block เล็ก ๆ ที่ถูก audit อย่าง
ละเอียดพร้อม comment อธิบายเหตุผลทุกบรรทัด (ตามที่เราพูดถึงใน Part 100 หัวข้อ safe abstraction)

**2. การสร้าง portfolio และชื่อเสียงที่พิสูจน์ได้จริง (ไม่ใช่แค่ "อ้างว่าเก่ง")** — ต่างจาก resume ที่เขียนอะไรก็ได้
GitHub contribution history เป็นหลักฐานที่ตรวจสอบได้: ใครก็เปิดดู PR ของคุณ อ่าน code review ที่คุณได้รับ และเห็นว่า
คุณตอบสนอง feedback อย่างไร สิ่งนี้มีค่ามากกว่า "ผมเก่ง Rust" ในตัวเอง เพราะมันคือ**หลักฐานที่ third party ยืนยันแล้ว**
(maintainer ที่ approve และ merge PR ของคุณคือคนยืนยันคุณภาพงานคุณ)

**3. การตอบแทนเครื่องมือที่คุณใช้ทำงานทุกวัน** — ถ้าคุณใช้ `serde` ทุกวันในการทำงาน แต่ไม่เคยคิดจะช่วยอะไรกลับคืน
เลย นั่นคือการใช้ทรัพยากรที่คนอื่นดูแลฟรีโดยไม่ได้ตอบแทน ecosystem ของ Rust แข็งแรงเพราะคนจำนวนมากยอม "จ่ายคืน" ใน
รูปแบบเวลาและความรู้ ไม่ว่าจะเป็น bug report ที่ละเอียด, การตอบคำถามใน issue ของคนอื่น, หรือ PR เล็ก ๆ ที่แก้ typo
ใน error message — ทุกอย่างนี้สะสมกันจนทำให้ ecosystem ดีขึ้นสำหรับทุกคน

**4. การเติบโตของทักษะจากการถูก code review โดย maintainer ที่มีประสบการณ์สูง** — นี่คือข้อที่มีค่ามากที่สุดแต่
มักถูกมองข้าม เมื่อ senior maintainer ของ project ระดับ `tokio` มา review PR ของคุณ คุณได้ feedback ที่ตรงประเด็น
และมาจากคนที่คิดเรื่อง API design, performance, และ correctness ของ Rust มาเป็นสิบปี ค่าของ feedback แบบนี้ (แม้จะ
เจ็บปวดเวลาถูกขอให้แก้ 5 รอบ) สูงกว่า feedback จากเพื่อนร่วมทีมทั่วไปมาก เพราะ maintainer เหล่านี้มองเห็น pattern
ที่ผิดพลาดซ้ำ ๆ จาก contributor นับร้อยคนมาก่อนแล้ว

จุดสำคัญที่ต้องเข้าใจตั้งแต่แรก: **การ contribute ที่ดีที่สุดในช่วงต้นไม่ใช่การเขียน feature ใหญ่** แต่คือการเลือก
งานที่ "เล็กพอที่จะทำสำเร็จ" และ "จริงพอที่จะมีประโยชน์" — สิ่งนี้จะอธิบายในหัวข้อถัดไป

#### เปรียบเทียบ: มุมมองที่ทำให้ Burnout เร็ว กับมุมมองที่ยั่งยืน

ลองเปรียบเทียบความคิดสองแบบของคนที่เพิ่งเริ่ม contribute ให้ open source เพื่อเห็นภาพความต่างที่ชัดเจนขึ้น:

**คนที่คิดแบบ "เพื่อ resume เท่านั้น"** มักมี pattern ความคิดแบบนี้: เลือก project ที่มีชื่อเสียงที่สุดที่หาได้
(ไม่ได้ดูว่าตัวเองใช้งานจริงหรือสนใจจริงหรือไม่) → หา issue ที่ดู "ยิ่งใหญ่" ที่สุดเพื่อให้ resume ดูดี → เขียนโค้ด
โดยไม่สนใจ convention ของ project มากนัก (คิดว่าโค้ดที่ทำงานได้ก็เพียงพอ) → เมื่อ PR ถูกขอให้แก้ไขหลายรอบ รู้สึก
ท้อและมองว่า maintainer "จับผิด" → สุดท้ายอาจปล่อย PR ทิ้งกลางทางเพราะหมดความอดทน

**คนที่คิดแบบยั่งยืน** มี pattern ตรงกันข้าม: เลือก project ที่ตัวเองใช้งานจริงในโปรเจกต์ส่วนตัวหรือที่ทำงาน (มี
motivation ตามธรรมชาติเพราะอยากให้เครื่องมือที่ใช้ดีขึ้น) → เริ่มจาก issue เล็กที่สุดที่หาได้เพื่อเรียนรู้
convention ก่อน → มองทุก feedback จาก code review เป็นโอกาสเรียนรู้ (ไม่ใช่การตัดสิน) → แม้ PR จะถูกปิดโดยไม่
merge ก็ยังได้ความรู้จากการอ่านโค้ดและได้ feedback ติดตัวไปเสมอ

ความต่างที่สำคัญที่สุดคือ **แหล่งที่มาของความพึงพอใจ** — คนแรกพึงพอใจแค่ตอน PR ถูก merge สำเร็จเท่านั้น (ทำให้รู้สึก
แย่มากเมื่อ PR ไม่ถูก merge) ส่วนคนที่สองพึงพอใจจากกระบวนการเรียนรู้ตลอดทาง ไม่ว่าผลลัพธ์สุดท้ายจะเป็นอย่างไร ความ
พึงพอใจแบบที่สองนี้ยั่งยืนกว่ามากในระยะยาว และแปลกที่มันมักนำไปสู่ผลลัพธ์ที่ดีกว่าด้วย (PR ที่ merge สำเร็จมากกว่า)
เพราะคนที่ไม่เร่งรีบมักเขียนโค้ดที่ใส่ใจ convention มากกว่า

### 105.2 หา Project และ Issue แรกที่เหมาะกับตัวเอง

#### ป้ายกำกับ (label) ที่ควรมองหา

GitHub มี convention ที่ maintainer ส่วนใหญ่ในวงการ Rust ยึดถือร่วมกัน คือการติด label สำหรับ issue ที่เหมาะกับ
คนใหม่:

- **`good-first-issue`** — issue ที่ maintainer ตั้งใจทำเครื่องหมายไว้ว่า "เหมาะกับคนที่ไม่คุ้นเคยกับ codebase นี้
  มาก่อน" มักเป็นงานที่ scope ชัดเจน แก้ที่เดียว ไม่ต้องเข้าใจ architecture ทั้งระบบ
- **`help-wanted`** — issue ที่ maintainer อยากได้ความช่วยเหลือ แต่ไม่จำเป็นต้องง่ายสำหรับมือใหม่ — อาจต้องมี
  ความเข้าใจ codebase ระดับหนึ่งมาก่อน
- **`E-easy` / `E-medium` / `E-hard`** — บาง project (เช่น `rust-lang/rust` เอง) ใช้ prefix `E-` (Experience)
  บอกระดับความยากที่คาดหวัง แทนคำว่า good-first-issue ตรง ๆ
- **`documentation`** — งานด้านเอกสาร ซึ่งมักเป็นจุดเริ่มต้นที่ดีมาก (จะพูดถึงในหัวข้อ 105.9)

GitHub เองมีหน้าค้นหาสาธารณะที่รวม issue แบบนี้จากทุก repo คือ `github.com/topics/good-first-issue` และการ search
`is:issue is:open label:"good-first-issue" language:rust` ใน GitHub search ก็ใช้ได้ผลดี

#### การจับคู่ "ขนาดของงาน" กับ "ความคุ้นเคยของคุณกับ codebase" — หลักการที่สำคัญที่สุดของหัวข้อนี้

ข้อผิดพลาดที่ผู้เริ่มต้น contribute เกือบทุกคนทำคือ: เจอ project ที่ชอบ แล้วรีบกระโดดไปทำ issue ที่ใหญ่หรือซับซ้อน
ที่สุดที่หาได้ เพราะคิดว่า "ยิ่งทำงานใหญ่ ยิ่งดูเก่ง" — นี่คือความคิดที่ผิดในทางปฏิบัติ เพราะเหตุผลนี้:

การ contribute ครั้งแรกมีตัวแปรที่ไม่รู้อยู่ 2 อย่างพร้อมกัน คือ (1) คุณไม่รู้ style/convention ของ project นี้
และ (2) คุณไม่รู้ architecture ของ codebase นี้ ถ้าคุณเลือกงานที่ใหญ่และซับซ้อนเป็นครั้งแรก คุณกำลังพยายามแก้ตัวแปร
ที่ไม่รู้ทั้งสองอย่างไปพร้อมกันในคราวเดียว ผลลัพธ์ที่พบบ่อยที่สุดคือ: ใช้เวลาหลายสัปดาห์เขียน PR ขนาดใหญ่ แล้วถูก
maintainer comment กลับมาว่า "เราคิดจะ refactor ส่วนนี้ไปในทิศทางอื่นอยู่แล้ว" หรือ "วิธีนี้ไม่ตรงกับ architecture
ที่เราต้องการ" — เสียเวลาไปมากโดยไม่ได้อะไรกลับมา (และแย่กว่านั้นคือทำให้รู้สึกท้อกับ open source ไปเลย)

หลักการที่ถูกต้องคือ: **ให้ขนาดของ PR แรก ๆ เล็กที่สุดเท่าที่จะทำได้ในขณะที่ยังมีประโยชน์จริง** ตัวอย่างขนาดที่
เหมาะสมสำหรับ PR แรกในโปรเจกต์ที่ไม่คุ้นเคย:

- แก้ typo ในเอกสารหรือ error message
- เพิ่ม test case ที่ยังไม่มีสำหรับ function ที่มีอยู่แล้ว
- แก้ bug เล็ก ๆ ที่ reproduce ได้ชัดเจนและ fix อยู่ในไฟล์เดียว
- ปรับปรุง doc comment ให้มี `# Examples` (ตามรูปแบบที่เรียนใน Part 34)

เมื่อ PR เล็ก ๆ แบบนี้ถูก merge สำเร็จ 2-3 ครั้ง คุณจะ (ก) เข้าใจ style และ workflow ของ project มากขึ้น (ข) เริ่ม
เป็นที่รู้จักกับ maintainer ในระดับหนึ่ง และ (ค) มีความมั่นใจในการอ่าน codebase มากขึ้น — จากนั้นค่อยขยับไปทำงานที่
ใหญ่ขึ้นทีละขั้น นี่คือ pattern ที่ contributor ที่ประสบความสำเร็จใน project ใหญ่ ๆ ใช้กันเกือบทั้งหมด

#### วิธีประเมินว่า project นั้น "เป็นมิตรกับ contributor" ก่อนลงทุนเวลา

ก่อนจะเสียเวลาไปกับ project ไหนก็ตาม ควรใช้เวลา 10-15 นาที "สำรวจ" ก่อน เพื่อประเมินสัญญาณต่อไปนี้:

**1. ดู commit activity ล่าสุด** — เปิดหน้า repo บน GitHub แล้วดูที่ commit ล่าสุดว่าเกิดขึ้นเมื่อไหร่ project ที่
commit ล่าสุดเกิน 6-12 เดือนอาจถูกทิ้ง (abandoned) — PR ที่คุณส่งไปอาจไม่มีใครมา review เลย ให้เช็คที่ tab
"Insights" → "Pulse" ของ repo เพื่อดูภาพรวมกิจกรรมใน 1 เดือนที่ผ่านมาได้เร็วกว่าไล่ดู commit ทีละอัน

**2. ดูว่า maintainer ตอบสนอง issue/PR ที่มีอยู่อย่างไร** — เปิด tab "Issues" และ "Pull requests" แล้วดูสิ่งเหล่านี้:
   - PR ที่เปิดอยู่มีการ comment กลับจาก maintainer ไหม หรือถูกเงียบไปเป็นเดือน?
   - เวลาเฉลี่ยระหว่างการเปิด PR กับการได้ comment แรกกลับมาประมาณเท่าไหร่? (ดูจาก timestamp ของ 5-10 PR ล่าสุด
     ที่ถูก merge หรือ close แล้ว)
   - เมื่อ contributor ถามคำถามใน issue มีคนตอบไหม หรือ issue ค้างอยู่โดยไม่มีคำตอบเป็นสิบ ๆ อัน?

   Project ที่ maintainer ตอบสนองภายใน 1-2 สัปดาห์ถือว่าเป็นสัญญาณที่ดีมาก ส่วน project ที่มี PR ค้างเป็นร้อยอัน
   โดยไม่มีการ comment เลยเกินครึ่งปี บอกว่า maintainer อาจไม่มีเวลาดูแล project นี้อีกแล้ว

**3. ดูว่ามี `CONTRIBUTING.md` หรือไม่ (และคุณภาพของมัน)** — การมีไฟล์นี้บอกว่า maintainer คิดเรื่อง contributor
experience มาก่อนแล้ว ไฟล์ที่ดีจะบอกชัดเจนว่า: ต้องรัน test/lint อะไรก่อน commit, ใช้ branch naming convention
แบบไหน, ต้องการ commit message รูปแบบไหน, และ PR ควรมีรายละเอียดอะไรบ้าง การไม่มีไฟล์นี้เลยไม่ได้แปลว่า project
แย่เสมอไป (project เล็กบางตัวไม่มีเพราะยังไม่มีใครเขียน) แต่การมีไฟล์นี้**และ**ไฟล์นั้นถูกปรับปรุงล่าสุดไม่เกิน 1-2
ปี เป็นสัญญาณบวกที่ชัดเจน

**4. ดูจำนวน maintainer ที่ active** — project ที่มี maintainer เดียวมีความเสี่ยงสูงกว่า project ที่มีทีม (เพราะ
maintainer คนเดียวอาจหมดเวลา/ความสนใจได้ง่ายกว่า) เช็คได้จาก tab "Insights" → "Contributors" ดูว่ามีกี่คนที่ commit
สม่ำเสมอในช่วง 6 เดือนล่าสุด

**5. ทดลองอ่าน 2-3 PR ที่ถูก merge ไปแล้วจริง** — เปิดดู PR ที่ merge สำเร็จ 2-3 อัน อ่าน comment ระหว่าง
maintainer กับ contributor ดูว่าบทสนทนา**ให้ความรู้สึกอย่างไร** — เป็นมิตรและอธิบายเหตุผลชัดเจน หรือดุดันและตัดสิน
คนอื่น สิ่งนี้บอกวัฒนธรรมของ project ได้ตรงกว่าการอ่าน Code of Conduct เสียอีก (เพราะ Code of Conduct คือคำสัญญา
แต่ PR comment คือพฤติกรรมจริง)

ถ้า project ผ่านทั้ง 5 ข้อนี้ในระดับที่ดี ก็มีโอกาสสูงว่าเวลาที่คุณลงทุนจะได้ผลตอบแทนคุ้มค่า — PR ของคุณจะถูก
review และมีโอกาส merge จริง ไม่ใช่ถูกทิ้งไว้เฉย ๆ

#### ค้นหา Issue ด้วย GitHub Search จริง: ตัวอย่าง Query ที่ใช้ได้ผล

GitHub มี search syntax ที่ทรงพลังมากสำหรับกรอง issue ตามเงื่อนไขที่ต้องการ — ตัวอย่าง query ที่ใช้ได้ผลดีเมื่อ
มองหา issue แรกในภาษา Rust:

```
is:issue is:open label:"good-first-issue" language:rust sort:updated-desc
```

การเติม `sort:updated-desc` สำคัญมาก เพราะ label `good-first-issue` มักถูกติดไว้นานแล้วโดยไม่มีใครหยิบไปทำ — การ
เรียงตาม "อัปเดตล่าสุด" ช่วยกรอง issue ที่ maintainer ยังพูดถึงอยู่จริง ๆ ออกจาก issue ที่ถูกลืมไปแล้ว หรือถ้าอยาก
จำกัดเฉพาะ organization ที่มีชื่อในวงการ Rust สามารถเติม `org:` เข้าไปได้ เช่น:

```
is:issue is:open label:"good-first-issue" org:tokio-rs
```

อีกเทคนิคหนึ่งที่มีประโยชน์คือค้นหา issue ที่ถูก comment ล่าสุดโดย**ไม่มีคน assign** (`no:assignee`) เพื่อให้แน่ใจ
ว่ายังไม่มีคนอื่นเริ่มทำอยู่ก่อนแล้ว:

```
is:issue is:open label:"good-first-issue" no:assignee
```

#### ตัวอย่างการประเมิน Project จริงแบบเป็นรูปธรรม (Worked Example)

เพื่อให้เห็นภาพว่าการประเมิน 5 ข้อในหัวข้อก่อนหน้าทำงานอย่างไรในทางปฏิบัติ ลองสมมติว่าคุณกำลังพิจารณา 2 project
สมมติที่มีลักษณะต่างกันชัดเจน:

**Project A:** commit ล่าสุดเมื่อ 2 สัปดาห์ก่อน, มี PR ที่ merge ไปแล้ว 340 อัน โดย PR ล่าสุด 10 อันได้ comment
แรกจาก maintainer ภายใน 3-5 วันเฉลี่ย, มี `CONTRIBUTING.md` ที่อัปเดตเมื่อ 4 เดือนก่อนพร้อมคำสั่ง `cargo test`
และ `cargo clippy -- -D warnings` ระบุไว้ชัดเจน, มี contributor ที่ active สม่ำเสมอ 6 คนในช่วง 6 เดือนล่าสุด

**Project B:** commit ล่าสุดเมื่อ 9 เดือนก่อน, มี PR ที่เปิดค้างอยู่ 45 อันโดยไม่มี comment จาก maintainer เลยเกิน
ครึ่งปี, ไม่มี `CONTRIBUTING.md`, มี contributor ที่ active แค่คนเดียวและ commit ล่าสุดของคนนั้นก็หายไปนานแล้ว
เช่นกัน

ข้อสรุปที่ได้จากการเปรียบเทียบนี้ชัดเจนมาก: Project A ให้สัญญาณบวกครบทุกข้อ — เวลาที่ลงทุนไปมีโอกาสสูงที่จะได้
ผลตอบแทน (PR ถูก review จริง, มีคนดูแลต่อเนื่อง) ในขณะที่ Project B แม้จะมี idea ที่น่าสนใจหรือมี star สูงบน GitHub
ก็ตาม แต่สัญญาณทั้งหมดบอกว่า project นี้อาจถูก "ทิ้ง" ไปแล้วในทางปฏิบัติ — PR ที่คุณส่งไปมีความเสี่ยงสูงที่จะไม่มี
ใครมา review เลย ไม่ว่าโค้ดที่คุณเขียนจะดีแค่ไหนก็ตาม สิ่งที่ควรทำกับ Project B ถ้ายังสนใจอยากช่วยจริง ๆ คือลองเปิด
issue ถามตรง ๆ ก่อนว่า "project นี้ยังรับ contribution อยู่ไหม" แทนการเสียเวลาเขียน PR ทันที

### 105.3 อ่านโค้ดที่ไม่คุ้นเคยขนาดใหญ่อย่างมีประสิทธิภาพ

Codebase ของ project จริงอาจมีไฟล์เป็นร้อยและโค้ดเป็นหมื่นบรรทัด การพยายาม "อ่านทุกไฟล์ให้เข้าใจทั้งหมดก่อนเริ่ม
แก้" เป็นวิธีที่ไม่มีทางทำสำเร็จและทำให้ท้อก่อนได้เริ่มทำงานจริง เทคนิคที่ใช้ได้ผลจริงมี 3 อย่าง ซึ่งควรใช้ร่วมกัน:

#### เทคนิคที่ 1: เริ่มจาก test เพื่อเข้าใจ "พฤติกรรมที่คาดหวัง"

Test คือเอกสารที่ไม่มีวันโกหก (ตามที่เราพูดถึงหลักการเดียวกันนี้กับ doc test ใน Part 34) เพราะมันต้องรันผ่านจริง
ก่อนจะ merge ได้ ในทางกลับกัน comment หรือเอกสารอาจ "เก่า" ตามหลังโค้ดที่เปลี่ยนไปแล้วได้ ถ้าคุณต้องการเข้าใจว่า
function หนึ่งควรทำงานอย่างไรในทุกกรณี ให้เปิดไฟล์ test ของ module นั้นก่อนเปิดไฟล์ implementation

ตัวอย่างการค้นหาแบบ practical: สมมติคุณจะแก้ bug ใน function ชื่อ `parse_header` ในโปรเจกต์ที่ไม่คุ้นเคย ให้ค้นหา
ด้วย:

```bash
# หา test ทั้งหมดที่เกี่ยวข้องกับ parse_header ก่อนเปิดดู implementation
grep -rn "parse_header" --include="*.rs" -l | grep -i test
```

หรือถ้า project แยก integration test ไว้ใน `tests/` ตามโครงสร้างที่เรียนใน Part 17 ให้ไล่ดูไฟล์ใน `tests/` ก่อน
เพราะ integration test มักสะท้อน "การใช้งานจริงจากมุมมอง user ของ crate" ได้ดีกว่า unit test ภายใน `src/`

การอ่าน test ก่อนช่วยตอบคำถาม 3 อย่างที่สำคัญที่สุดก่อนแก้โค้ด: input แบบไหนที่ควรผ่าน, input แบบไหนที่ควรถูก
ปฏิเสธ (return `Err` หรือ panic), และ edge case ไหนที่ maintainer เคยคิดไว้แล้ว (เช่น string ว่าง, ตัวเลขติดลบ,
Unicode ที่ไม่ใช่ ASCII)

#### เทคนิคที่ 2: ใช้ `cargo doc --open` เพื่ออ่าน public API ของ crate

จาก Part 34 เราเรียนวิธีเขียน rustdoc ไปแล้ว — ทักษะเดียวกันนี้กลับมามีประโยชน์ตรงนี้ในมุมของ**ผู้อ่าน**เอกสาร
แทนผู้เขียน เมื่อ clone project ที่ไม่คุ้นเคยมา คำสั่งแรก ๆ ที่ควรรันคือ:

```bash
cargo doc --open --no-deps
```

flag `--no-deps` สำคัญมาก เพราะมันบอก cargo ว่า**ไม่ต้อง**ไป generate เอกสารของ dependency ทั้งหมดด้วย (ซึ่งอาจมี
เป็นร้อย crate และใช้เวลานานมาก) เราต้องการดูแค่เอกสารของ crate ที่เรากำลังจะแก้เท่านั้น

หน้าเอกสารที่เปิดขึ้นมาจะให้ภาพรวมของ **public API** ทั้งหมดของ crate — struct, enum, trait, function ที่ `pub`
พร้อม doc comment ที่ maintainer เขียนไว้ ข้อดีของการเริ่มจากมุมนี้คือ มันกรอง "รายละเอียดภายใน" (private
implementation) ออกไปโดยอัตโนมัติ ทำให้คุณเห็นแค่ "สัญญา" (contract) ที่ crate นี้ให้กับผู้ใช้ ซึ่งมักจะเพียงพอ
สำหรับเข้าใจว่า "ระบบนี้ออกแบบมาให้ใช้งานยังไง" ก่อนที่จะลงไปดู "มันทำงานภายในยังไง"

ถ้า crate ใหญ่มาก ให้เริ่มจากหน้า crate-level doc (`//!` ที่อยู่บนสุดของ `lib.rs`) ซึ่งมักมี "elevator pitch" อธิบาย
ภาพรวมของทั้ง crate ในไม่กี่ประโยค (ตามรูปแบบที่สอนใน Part 34 หัวข้อ 34.11) — นี่คือจุดเริ่มต้นที่ดีที่สุดเสมอ
ก่อนจะไล่ดู module ย่อยทีละตัว

#### เทคนิคที่ 3: ไล่ตาม "หนึ่ง code path" ที่เกี่ยวกับงานที่จะทำ แทนการพยายามเข้าใจทั้งหมด

นี่คือเทคนิคที่สำคัญที่สุดในสามข้อ: **อย่าพยายามเข้าใจทั้ง codebase ก่อนเริ่มแก้** ให้เลือก "หนึ่งเส้นทาง" ที่
เกี่ยวข้องกับ bug หรือ feature ที่คุณจะทำ แล้วไล่ตามมันตั้งแต่จุดเริ่ม (entry point เช่น public function ที่ user
เรียกใช้) ไปจนถึงจุดที่เกิดปัญหา หรือจุดที่ต้องแก้

วิธีปฏิบัติจริง:

1. หา entry point ที่เกี่ยวข้อง (มักมาจาก stack trace ของ bug report, หรือชื่อ function ใน issue)
2. ใช้ "go to definition" ของ editor (หรือ `grep -rn "fn function_name"` ถ้าไม่มี IDE ที่รองรับ rust-analyzer) เพื่อ
   ตามไปดูว่า function นี้เรียก function อะไรต่อ
3. หยุดตามทันทีที่เจอ dependency ภายนอก (เช่น เรียก crate อื่น) ที่ไม่เกี่ยวกับ logic ที่กำลังสนใจ — ไม่ต้องเปิดไป
   อ่าน source ของ dependency นั้นถ้าไม่จำเป็น
4. จดสิ่งที่พบไว้เป็น comment ชั่วคราวหรือ note แยก (เช่น "flow: `handle_request` → `validate_input` →
   `parse_header` ← bug อยู่ที่นี่ เพราะไม่ trim whitespace ก่อน parse")

เทคนิคนี้ใช้ประโยชน์จากข้อเท็จจริงว่า **คุณไม่จำเป็นต้องเข้าใจทุกส่วนของระบบเพื่อแก้ปัญหาหนึ่งจุด** เหมือนกับที่
วิศวกร production ไม่ต้องเข้าใจทุกบรรทัดของ Linux kernel เพื่อแก้ driver บั๊กหนึ่งตัว — เขาแค่ต้องเข้าใจ "เส้นทาง"
ที่เกี่ยวข้องกับปัญหานั้นให้ลึกพอ

รวมสามเทคนิคนี้เข้าด้วยกัน ลำดับที่แนะนำคือ: **test ก่อน (เข้าใจพฤติกรรมที่คาดหวัง) → doc ต่อ (เข้าใจภาพรวม public
API) → ไล่ code path เฉพาะจุดที่เกี่ยวกับงาน (เข้าใจ implementation เท่าที่จำเป็น)**

#### ตัวอย่างการไล่ Code Path แบบเป็นรูปธรรม

ลองจำลองสถานการณ์ที่เป็นรูปธรรมมากขึ้น: สมมติมี issue บอกว่า `tiny_calc::sub(5, 3)` ควร return `2` แต่ผู้ report
สงสัยว่า function นี้อาจมี edge case ที่ผิดตอน argument ติดลบ (สมมติเหตุการณ์นี้ขึ้นมาเพื่อสาธิตกระบวนการคิด ไม่ใช่
bug จริงใน `tiny_calc` ที่ใช้สาธิตในบทนี้) วิธีไล่ code path ตามเทคนิคที่ 3 มีลำดับดังนี้:

1. **เริ่มจาก entry point ที่ issue พูดถึง**: เปิด `src/lib.rs` หา `pub fn sub`
2. **ตรวจว่า `sub` เรียก function อื่นต่อหรือไม่**: ในกรณีนี้ `sub` แค่ทำ `a - b` ตรง ๆ ไม่มีการเรียก function อื่น
   ต่อเลย — code path สิ้นสุดตรงนี้ ไม่ต้องไล่ต่อ
3. **ตรวจสอบว่า type `i32` รองรับค่าติดลบอยู่แล้วโดย design**: จาก signature `pub fn sub(a: i32, b: i32) -> i32`
   ชนิดข้อมูล `i32` เป็น signed integer ที่รองรับค่าติดลบตามธรรมชาติ (`i32::MIN` ถึง `i32::MAX`) การลบเลขติดลบจึง
   ไม่ใช่ปัญหาในเชิง type — ถ้ามีปัญหาจริงต้องเป็นเรื่อง overflow ที่ขอบของ `i32::MIN`/`i32::MAX` เท่านั้น (ซึ่งใน
   `debug` build จะ panic อัตโนมัติ แต่ใน `release` build จะ wrap around แบบ two's complement)
4. **สรุปผลจากการไล่ code path**: bug ที่ผู้ report สงสัยไม่มีจริงสำหรับ input ปกติ — มีแต่ edge case ที่ขอบเขต
   ของ `i32` ซึ่งควรมี test เพิ่มเพื่อ document พฤติกรรมนี้ให้ชัดเจน (เช่น `sub(i32::MIN, 1)` ควร panic ใน debug
   build ตามที่คาดหวัง)

สังเกตว่าทั้งกระบวนการนี้ใช้เวลาไม่กี่นาที เพราะเราไม่ได้พยายามเข้าใจทั้ง crate ก่อน — เราไล่ตามแค่เส้นทางที่
เกี่ยวข้องกับคำถามเฉพาะเจาะจงเท่านั้น ถ้า crate มีขนาดใหญ่กว่านี้มาก (เช่น `sub` เรียก validation function อีก 3-4
ชั้นก่อนจะคำนวณจริง) หลักการเดียวกันนี้ก็ใช้ได้ — ไล่ทีละชั้นจนกว่าจะเจอจุดที่เป็นสาเหตุ แล้วหยุดที่นั่น ไม่ต้องไล่
ต่อไปยัง module อื่นที่ไม่เกี่ยวข้อง

### 105.4 Git Workflow สำหรับ Open Source Contribution: Fork, Upstream, Rebase

ในการทำงานแบบทีมภายในบริษัท คุณมักมี **write access** เข้า repo หลักโดยตรง — สร้าง branch ใหม่ใน repo เดียวกันได้
เลย แต่การ contribute ให้ open source project ที่ไม่ใช่ของคุณ (และคุณไม่ได้เป็น maintainer) คุณจะ**ไม่มี** write
access เข้า repo ต้นทาง ดังนั้นต้องใช้ workflow แบบ **fork** ซึ่งมีแนวคิดต่างจากการทำงานในทีมเล็กน้อย

ข้อสังเกตที่ควรรู้ไว้ก่อนเริ่ม: บาง project เล็ก ๆ ที่คุณคุ้นเคยกับ maintainer เป็นอย่างดีแล้ว (หรือ project ที่
maintainer เชิญคุณเข้าทีมหลังจาก contribute สำเร็จหลายครั้ง) อาจให้ **write access แบบจำกัด** เข้า repo หลักได้
โดยตรง — ในกรณีนี้ workflow จะง่ายขึ้นเหลือแค่สร้าง branch ใน repo เดียวกันแล้วเปิด PR จาก branch นั้นตรง ๆ โดยไม่
ต้อง fork เลย (คล้ายกับการทำงานในทีมภายในบริษัท) แต่สำหรับ project ส่วนใหญ่ที่คุณเพิ่งเริ่ม contribute เป็นครั้งแรก
ให้สมมติว่าต้อง fork เสมอ เพราะเป็นสถานการณ์ที่พบบ่อยที่สุดในทางปฏิบัติ

#### แนวคิดของ Fork

Fork คือการสร้าง "สำเนาที่คุณเป็นเจ้าของ" ของ repo ต้นทาง (เรียกว่า **upstream**) บน GitHub เอง (กดปุ่ม "Fork" บน
หน้า repo) คุณ push การเปลี่ยนแปลงไปที่ fork ของคุณ (ซึ่งคุณมี write access เต็มที่) แล้วเปิด Pull Request จาก
fork ของคุณไปยัง upstream — maintainer ของ upstream จะเป็นคนตัดสินใจว่าจะ merge เข้า repo หลักหรือไม่

ปัญหาที่เกิดขึ้นตามมาคือ: **fork ของคุณจะค่อย ๆ ล้าหลัง (out of date) เมื่อเทียบกับ upstream** เพราะคนอื่นยังคง
push การเปลี่ยนแปลงเข้า upstream ต่อไปเรื่อย ๆ ในขณะที่คุณทำงาน ดังนั้นเราต้องมี remote 2 ตัวใน local repo ของเรา:
`origin` (ชี้ไปที่ fork ของเรา) และ `upstream` (ชี้ไปที่ repo ต้นทาง) เพื่อให้ดึงความเปลี่ยนแปลงล่าสุดจาก upstream
มา sync กับงานของเราได้ตลอดเวลา

#### สาธิตด้วยคำสั่งจริง: จำลอง Fork/Upstream ด้วย Local Repository สองตัว

เพื่อแสดงคำสั่งจริงโดยไม่ต้องพึ่งพา GitHub จริง (และไม่กระทบ repo จริงใด ๆ) เราจะจำลองความสัมพันธ์นี้ด้วย local
git repository สองตัวในเครื่อง: `upstream/` แทน repo ต้นทางที่เราไม่มี write access และ `fork/` แทน fork ของเรา
(ในโลกจริง `fork/` จะถูก clone มาจาก URL ของ fork บน GitHub เช่น `git@github.com:yourname/project.git` แต่
mechanics ของคำสั่งเหมือนกันทุกอย่าง ไม่ว่า remote จะเป็น local path หรือ URL จริงบน GitHub)

เริ่มจากสร้าง "upstream" repo ที่มี crate เล็ก ๆ ชื่อ `tiny_calc`:

```bash
mkdir upstream && cd upstream
git init -b main
git config user.email "maintainer@example.com"
git config user.name "Demo Maintainer"
```

สร้างไฟล์ crate ขั้นต่ำ (`Cargo.toml` และ `src/lib.rs` ที่มี `add` และ `sub`) แล้ว commit:

```bash
git add -A
git commit -m "Initial commit: tiny_calc v0.1.0 with add and sub"
```

ผลลัพธ์ที่ได้จริงจากการรันคำสั่งนี้:

```
== upstream log ==
534579f Initial commit: tiny_calc v0.1.0 with add and sub
```

ตอนนี้เรามี "upstream" ที่มี 1 commit ต่อไปเราจำลองการ "fork" ด้วยการ `git clone` จาก upstream (ในโลกจริงคือกดปุ่ม
Fork บน GitHub แล้ว `git clone` URL ของ fork ที่ได้):

```bash
git clone upstream fork
cd fork
git remote add upstream ../upstream
```

หลังจากนี้ `git remote -v` แสดงผลลัพธ์จริงดังนี้:

```
origin    /path/to/upstream (fetch)
origin    /path/to/upstream (push)
upstream  ../upstream (fetch)
upstream  ../upstream (push)
```

สังเกตว่า **`origin` ชี้ไปที่ fork ของเรา** (เพราะเรา clone มาจากที่นั่น — ในโลกจริง `origin` จะเป็น URL ของ fork
บน GitHub ของบัญชีเรา) ส่วน **`upstream` เป็น remote เพิ่มเติม**ที่เราเพิ่มเข้ามาเองด้วย `git remote add` เพื่อให้
`git fetch upstream` ดึงความเปลี่ยนแปลงล่าสุดจาก repo ต้นทางได้โดยตรง — นี่คือคำสั่งที่ใช้เป๊ะ ๆ ในการทำงานกับ
GitHub จริง เพียงแค่แทน `../upstream` ด้วย URL จริงเช่น `git@github.com:original-owner/project.git`

#### สร้าง Branch สำหรับงานของเรา

ตาม convention ที่ project ส่วนใหญ่ใช้ ไม่ควรทำงานบน `main` โดยตรง (เพราะ `main` ต้องสะอาดพร้อม sync กับ upstream
ตลอด) ให้สร้าง branch ใหม่ที่ชื่อสื่อความหมายของงานที่ทำ:

```bash
git checkout -b fix/sub-doc-typo-and-test
```

ชื่อ branch ที่ดีมักมี prefix บอกประเภทของงาน (`fix/`, `feat/`, `docs/`, `chore/`) ตามด้วยคำอธิบายสั้น ๆ ของสิ่งที่
ทำ — นี่ไม่ใช่กฎบังคับตายตัวของ git แต่เป็น convention ที่หลาย project ระบุไว้ใน `CONTRIBUTING.md`

#### จำลองสถานการณ์จริง: Upstream เปลี่ยนไปในขณะที่เรากำลังทำงาน

ในโลกจริง ระหว่างที่เราทำงานบน branch ของเรา คนอื่นก็ยังคง merge PR เข้า upstream ต่อไปเรื่อย ๆ เราจำลองสิ่งนี้
โดยกลับไปที่ `upstream/` แล้วเพิ่ม commit ใหม่ (เสมือนมีคนอื่น merge PR เข้าไปแล้ว):

```bash
# ที่ upstream/ (จำลองว่ามี PR อื่นถูก merge ไปแล้ว)
git commit -am "Add mul() function for multiplication support"
```

```
== upstream log after simulated new merge ==
8629594 Add mul() function for multiplication support
534579f Initial commit: tiny_calc v0.1.0 with add and sub
```

ในขณะที่ fork ของเรายังไม่รู้เรื่องนี้เลย — `main` ของ fork ยังชี้อยู่ที่ commit เดิม (`534579f`)

#### แก้ไขงานจริงบน branch ของเรา แล้ว Commit

กลับมาที่ `fork/` บน branch `fix/sub-doc-typo-and-test` เราแก้ไขจริง 2 อย่าง: (1) เติม `.` ท้าย doc comment ของ
`sub()` ที่ขาดไปเมื่อเทียบกับ `add()` และ (2) เพิ่ม unit test `test_sub` ที่ยังไม่มีมาก่อน — เป็นการแก้ไขเล็ก ๆ
ที่จริง มีประโยชน์จริง และ scope ชัดเจนตามหลักการในหัวข้อ 105.2:

```diff
-/// Subtracts `b` from `a`
+/// Subtracts `b` from `a`.
 pub fn sub(a: i32, b: i32) -> i32 {
     a - b
 }
```

```diff
     fn test_add() {
         assert_eq!(add(2, 3), 5);
     }
+
+    #[test]
+    fn test_sub() {
+        assert_eq!(sub(5, 3), 2);
+    }
 }
```

(รายละเอียดของการรัน `cargo fmt`/`cargo clippy`/`cargo test` ก่อน commit จะอธิบายในหัวข้อ 105.6 — ในที่นี้ทุกคำสั่ง
ผ่านหมดจริง ก่อนที่จะ commit)

Commit ด้วยข้อความที่อธิบาย "อะไร" และ "ทำไม" (รายละเอียดเรื่องการเขียน commit message ที่ดีอยู่ในหัวข้อ 105.5):

```bash
git commit -m "Fix missing period in sub() doc comment and add test_sub

sub() was the only public function without a matching unit test and
its doc comment was missing the trailing period used elsewhere in the
file. Add test_sub to bring it to parity with add(), and fix the doc
comment punctuation.

Closes #42"
```

#### Sync Fork กับ Upstream ล่าสุด แล้ว Rebase Branch ของเราทับ

ตอนนี้ upstream มี commit ใหม่ (`mul()`) ที่ fork ของเรายังไม่รู้จัก ก่อนเปิด PR เราควร sync ให้ branch ของเรา
build อยู่บนโค้ดล่าสุดของ upstream เสมอ — วิธีทำคือ `git fetch upstream` (ดึงข้อมูลมาโดยไม่แก้ branch ปัจจุบัน)
ตามด้วยการ update `main` ของ fork ให้ตรงกับ upstream แล้ว rebase branch งานของเราทับ `main` ที่ sync แล้ว:

```bash
git fetch upstream
```

ผลลัพธ์จริง:

```
From ../upstream
 * [new branch]      main       -> upstream/main
```

ตรวจสอบก่อน rebase ว่าประวัติ diverge กันอย่างไร:

```
== สถานะ branch เรา ก่อน rebase ==
* 9932763 Fix missing period in sub() doc comment and add test_sub
| * 8629594 Add mul() function for multiplication support
|/
* 534579f Initial commit: tiny_calc v0.1.0 with add and sub
```

เห็นได้ชัดว่า commit ของเรา (`9932763`) และ commit `mul()` ของ upstream (`8629594`) แยกออกจากกันคนละสาย
(diverge) จาก commit ร่วมกันตัวเดียวกัน (`534579f`) — นี่คือสถานการณ์ปกติมากในการ contribute จริง

ขั้นแรก sync `main` ของ fork ให้ตรงกับ `upstream/main` ด้วย fast-forward merge (ปลอดภัยเพราะ `main` ของเราไม่มี
commit ของตัวเอง ไม่มีทาง conflict):

```bash
git checkout main
git merge --ff-only upstream/main
```

```
Updating 534579f..8629594
Fast-forward
 src/lib.rs | 5 +++++
 1 file changed, 5 insertions(+)
```

จากนั้น rebase branch งานของเราทับ `main` ที่ sync แล้ว:

```bash
git checkout fix/sub-doc-typo-and-test
git rebase main
```

ผลลัพธ์จริง:

```
Rebasing (1/1)
Successfully rebased and updated refs/heads/fix/sub-doc-typo-and-test.
```

ตรวจสอบประวัติหลัง rebase — ตอนนี้เป็น**เส้นตรง** (linear history) ไม่มีการแยกสาย:

```
== log หลัง rebase ==
* 8062ca5 Fix missing period in sub() doc comment and add test_sub
* 8629594 Add mul() function for multiplication support
* 534579f Initial commit: tiny_calc v0.1.0 with add and sub
```

สังเกตว่า commit hash ของเราเปลี่ยนจาก `9932763` เป็น `8062ca5` — นี่เป็นเรื่องปกติของ rebase (rebase สร้าง commit
ใหม่ที่มี parent ต่างจากเดิม แม้เนื้อหา diff จะเหมือนกัน) และเป็นเหตุผลสำคัญที่ **ไม่ควร rebase branch ที่คนอื่น
กำลังใช้งานอยู่ร่วมกับเรา** เพราะ hash ที่เปลี่ยนจะทำให้ประวัติของคนอื่นไม่ตรงกับของเรา (rebase ปลอดภัยกับ branch
ส่วนตัวของเราเองที่ยังไม่แชร์กับใคร หรือ branch PR ที่เรา force-push ทับได้เองเท่านั้น)

ทำไมต้อง rebase แทนที่จะ merge upstream/main เข้า branch ของเราตรง ๆ? Project จำนวนมากในวงการ Rust นิยม **linear
history** (ประวัติเป็นเส้นตรง ไม่มี merge commit พร่ำเพรื่อ) เพราะทำให้ `git log`, `git bisect`, และการอ่าน
history ย้อนหลังทำได้ง่ายกว่ามาก — แต่ก็มี project จำนวนไม่น้อยที่ใช้ merge commit ตามปกติ (โดยเฉพาะ project ที่มี
GitHub "merge" button แบบ default) ดังนั้น**ควรเช็ค `CONTRIBUTING.md` ของ project นั้นก่อนเสมอ**ว่าต้องการแบบไหน

หลัง rebase สำเร็จ ให้รัน `cargo fmt`, `cargo clippy`, และ `cargo test` อีกครั้งเพื่อยืนยันว่าทุกอย่างยังผ่านอยู่
(เพราะ rebase อาจทำให้เกิด conflict ที่ resolve ผิดได้) — ในตัวอย่างนี้ทุกคำสั่งผ่านเหมือนก่อน rebase ทุกประการ

สุดท้าย push branch ของเราไปที่ `origin` (fork ของเรา) ด้วย:

```bash
git push origin fix/sub-doc-typo-and-test
```

(คำสั่งนี้ไม่ได้รันจริงในสาธิตนี้เพราะ `origin` เป็น local path ไม่ใช่ GitHub จริง — แต่ mechanics เหมือนกันทั้งหมด
เมื่อ `origin` เป็น URL ของ fork บน GitHub จริง จากนั้นเปิดหน้า fork บน GitHub จะเห็นปุ่ม "Compare & pull request"
ปรากฏขึ้นให้กดเปิด PR ได้ทันที)

ถ้า branch ของเราถูก push ไปแล้วครั้งหนึ่ง แล้วเรา rebase ใหม่อีกครั้งภายหลัง (เช่น sync กับ upstream รอบใหม่ หรือ
แก้ commit ตาม review feedback) การ push ครั้งต่อไปต้องใช้ `--force-with-lease` แทน `--force` เปล่า ๆ:

```bash
git push --force-with-lease origin fix/sub-doc-typo-and-test
```

`--force-with-lease` ปลอดภัยกว่า `--force` ตรงที่มันจะ**ปฏิเสธ push** ถ้ามีคนอื่น push อะไรเข้า branch เดียวกันบน
remote ไปแล้วโดยที่เราไม่รู้ (เช่น maintainer แก้บางอย่างให้เราตรงบน branch PR ของเรา ซึ่งบางโปรเจกต์ทำแบบนี้เพื่อ
ประหยัดเวลา) — ในขณะที่ `--force` เปล่า ๆ จะทับทุกอย่างโดยไม่เตือนอะไรเลย

#### เมื่อ Rebase เจอ Conflict จริง ๆ: วิธี Resolve อย่างเป็นระบบ

ในตัวอย่างของหัวข้อก่อนหน้า rebase ผ่านไปโดยไม่มี conflict เพราะการเปลี่ยนแปลงของเราอยู่ใน function คนละตัวกับที่
upstream เปลี่ยน แต่ในโลกจริง conflict เกิดขึ้นได้บ่อยกว่านั้นมาก โดยเฉพาะเมื่อทั้งสองฝั่งแก้ไฟล์บรรทัดใกล้เคียงกัน
เราจำลองสถานการณ์นี้ด้วย local repository อีกคู่หนึ่ง (แยกจากตัวอย่าง `tiny_calc`) เพื่อแสดง conflict marker และ
วิธี resolve จริง

สมมติ branch `main` และ branch `feature` ของเราต่าง fork ออกจาก commit เดียวกันที่มี:

```rust
pub fn greet(name: &str) -> String {
    format!("Hello, {name}")
}
```

บน `feature` เราแก้ข้อความทักทาย:

```rust
pub fn greet(name: &str) -> String {
    format!("Hi there, {name}")
}
```

ในขณะเดียวกัน upstream (`main`) ก็ถูกคนอื่นแก้ไฟล์**เดียวกัน บรรทัดเดียวกัน**ไปในทางอื่น (เติม `!`):

```rust
pub fn greet(name: &str) -> String {
    format!("Hello, {name}!")
}
```

เมื่อรัน `git rebase main` จาก branch `feature` ผลลัพธ์จริงที่ได้คือ:

```
Rebasing (1/1)
Auto-merging file.rs
CONFLICT (content): Merge conflict in file.rs
error: could not apply 831dee1... feature: change greeting to Hi there
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
```

Git บอกตรง ๆ ว่ามี conflict ที่ต้อง resolve เอง เปิดไฟล์ที่ conflict ดูจะเห็น **conflict marker** จริงแบบนี้:

```
pub fn greet(name: &str) -> String {
<<<<<<< HEAD
    format!("Hello, {name}!")
=======
    format!("Hi there, {name}")
>>>>>>> 831dee1 (feature: change greeting to Hi there)
}
```

การอ่าน marker เหล่านี้ต้องเข้าใจว่า **ระหว่าง rebase, `HEAD` หมายถึงฝั่ง `main` (ปลายทางที่เรากำลัง rebase ทับไป)**
ไม่ใช่ branch ของเราเอง — ส่วนที่อยู่ระหว่าง `<<<<<<< HEAD` กับ `=======` คือเนื้อหาจาก `main`, และส่วนระหว่าง
`=======` กับ `>>>>>>> <commit>` คือเนื้อหาจาก commit ของเราที่กำลังพยายาม apply ทับ (นี่เป็นจุดที่สร้างความสับสน
ให้มือใหม่บ่อยมาก เพราะระหว่าง `git merge` ธรรมดา `HEAD` หมายถึง branch ปัจจุบันของเรา แต่ระหว่าง `git rebase`
ความหมายกลับกัน — ต้องระมัดระวังเป็นพิเศษ)

วิธี resolve คือแก้ไฟล์ให้เหลือเนื้อหาที่ถูกต้องที่รวมทั้งสองการเปลี่ยนแปลงเข้าด้วยกัน (ลบ conflict marker ทั้งหมด
ออก) ในกรณีนี้เราต้องการทั้ง "Hi there" ของเราและ "!" ของ upstream:

```rust
pub fn greet(name: &str) -> String {
    format!("Hi there, {name}!")
}
```

จากนั้น `git add` ไฟล์ที่ resolve แล้ว และสั่งให้ rebase ทำงานต่อ:

```bash
git add file.rs
git rebase --continue
```

ผลลัพธ์จริง:

```
[detached HEAD d01fb1f] feature: change greeting to Hi there
1 file changed, 1 insertion(+), 1 deletion(-)
Successfully rebased and updated refs/heads/feature.
```

ตรวจสอบ log สุดท้าย — ประวัติกลับมาเป็นเส้นตรงเรียบร้อย:

```
d01fb1f feature: change greeting to Hi there
ce8e68a main: add exclamation mark to greeting
5bd996f Initial greet function
```

ข้อสังเกตสำคัญ 3 อย่างจากการ resolve conflict:

**1. Rebase อาจมีหลายรอบถ้ามีหลาย commit ที่ conflict** — ถ้า branch ของเรามีหลาย commit และแต่ละ commit conflict
กับ upstream คนละจุด `git rebase --continue` จะหยุดให้ resolve ทีละ commit ตามลำดับ ไม่ใช่ resolve ทีเดียวจบหมด

**2. ถ้า resolve ผิดพลาดหรือสับสน ให้ `git rebase --abort` กลับไปที่จุดเริ่มต้นได้เสมอ** — คำสั่งนี้ปลอดภัย 100%
เพราะแค่คืน branch กลับไปเป็นสถานะก่อนเริ่ม rebase โดยไม่เสียงานอะไรเลย เหมาะมากถ้ารู้สึกว่า resolve ไปผิดทางและ
อยากเริ่มใหม่อย่างใจเย็น

**3. หลัง resolve conflict เสร็จ ต้องรัน `cargo fmt`/`cargo clippy`/`cargo test` อีกครั้งเสมอ** — เพราะการ resolve
conflict ด้วยมืออาจทำให้โค้ด format ผิดหรือมี logic ที่ผิดพลาดโดยไม่ตั้งใจ (เช่น ลบ conflict marker ไม่ครบ หรือลืม
ปิดวงเล็บ) การรัน check ทั้งหมดอีกครั้งช่วยจับความผิดพลาดแบบนี้ได้ก่อนที่จะ push

### 105.5 เขียน Commit Message และ PR Description ที่ดี

Commit message และ PR description คือ**เอกสารที่คนอ่านมากที่สุด**ในกระบวนการ code review — reviewer อ่านมันก่อน
จะอ่านโค้ดจริงเสียอีก ดังนั้นคุณภาพของข้อความเหล่านี้ส่งผลต่อความเร็วในการได้รับการ review อย่างมาก

#### Convention ที่ project ในวงการ Rust ส่วนใหญ่ยึดถือ

**1. Imperative mood (รูปคำสั่ง) สำหรับบรรทัดแรก** — เขียนเหมือนกำลังสั่งให้ codebase ทำอะไร ไม่ใช่บรรยายว่า
"ฉันได้ทำอะไรไปแล้ว" ตัวอย่าง: `Fix off-by-one error in slice bounds check` (ถูก) เทียบกับ `Fixed off-by-one
error` หรือ `Fixes off-by-one error` (ทั้งสองแบบหลังนี้ใช้กันแพร่หลายเหมือนกันในหลาย project แต่ Linux kernel และ
project จำนวนมากในวงการ Rust อย่าง `rust-lang/rust` ยึดถือ imperative mood อย่างเคร่งครัด) เหตุผลเชิงเทคนิคที่มัก
ถูกอ้างถึงคือ: บรรทัดแรกของ commit message ควรเติมประโยค "If applied, this commit will ___" ได้อย่างเป็นธรรมชาติ
— `If applied, this commit will fix off-by-one error` อ่านลื่นกว่า `If applied, this commit will fixed off-by-one
error` อย่างชัดเจน

**2. บรรทัดแรกสั้น กระชับ (ไม่เกิน ~72 ตัวอักษร) ตามด้วยบรรทัดเปล่า แล้วค่อยอธิบายรายละเอียด** — GitHub, `git log
--oneline`, และเครื่องมืออีกหลายตัวตัดบรรทัดแรกมาแสดงแบบ summary ถ้ายาวเกินจะถูกตัดจนอ่านไม่รู้เรื่อง

**3. อธิบาย "ทำไม" ไม่ใช่แค่ "อะไร" ในส่วน body** — โค้ด diff เองก็บอก "อะไร" อยู่แล้ว (คนอ่าน diff เห็นว่าบรรทัด
ไหนถูกลบ บรรทัดไหนถูกเพิ่ม) สิ่งที่ diff บอกไม่ได้คือ "ทำไมถึงต้องเปลี่ยนแบบนี้" — เหตุผลเชิง business, bug ที่
พบ, หรือ trade-off ที่พิจารณาแล้ว

**4. อ้างอิง issue ที่เกี่ยวข้องด้วย keyword ที่ GitHub รู้จัก** — คำว่า `Closes #42`, `Fixes #42`, หรือ `Resolves
#42` ที่อยู่ใน commit message หรือ PR description ทำให้ GitHub **ปิด issue นั้นให้อัตโนมัติ**ทันทีที่ PR ถูก
merge — ประหยัดเวลา maintainer ไม่ต้องมาปิด issue มือ ถ้าแค่ "เกี่ยวข้องกับ" issue แต่ไม่ได้แก้ทั้งหมด ให้ใช้คำว่า
`Refs #42` หรือ `Related to #42` แทน (คำเหล่านี้จะ link ไปที่ issue แต่ไม่ปิดให้อัตโนมัติ)

#### ตัวอย่างเปรียบเทียบ: Commit Message ที่ดี vs ไม่ดี

**ตัวอย่างที่ไม่ดี:**

```
fix bug
```

ปัญหา: ไม่บอกว่า bug อะไร, ไม่บอกว่าแก้ยังไง, ไม่มี context ใด ๆ เลย เมื่อ maintainer เปิด `git log` ย้อนดูใน
อนาคต (เช่นตอน bisect หา commit ที่ทำให้เกิด regression) ข้อความแบบนี้ไม่มีประโยชน์อะไรเลย

**ตัวอย่างที่ไม่ดี (อีกแบบ):**

```
Updated the parser code to handle the case where the input string
contains unicode characters that were causing issues before because
the byte indexing was wrong and I had to change several things
around the tokenizer module to make everything work correctly again
```

ปัญหา: เป็นประโยคยาวเดียวไม่มีการแบ่งบรรทัดแรก/body อย่างเหมาะสม ปนกันทั้ง "อะไร" และ "ทำไม" แบบไม่มีโครงสร้าง
อ่านยากมากเมื่อแสดงแบบ `--oneline`

**ตัวอย่างที่ดี:**

```
Fix panic on multi-byte UTF-8 boundary in tokenizer

The tokenizer indexed into the input &str using raw byte offsets
computed from `.len()`, which can land in the middle of a multi-byte
UTF-8 character and cause `&str` slicing to panic.

Switch to `char_indices()` so offsets always align to a valid char
boundary. Added a regression test with Thai text ("สวัสดี") which
previously triggered the panic.

Fixes #128
```

เหตุผลที่ดีกว่า: บรรทัดแรกสั้นและเป็น imperative mood, body อธิบาย root cause ("ทำไม" ถึง panic) ก่อนอธิบายวิธี
แก้ ("อะไร" ที่เปลี่ยน) แล้วปิดท้ายด้วยการอ้างอิง issue ที่ GitHub จะปิดให้อัตโนมัติ — reviewer อ่านครั้งเดียวเข้าใจ
ทั้งปัญหาและวิธีแก้โดยไม่ต้องเปิด diff ก่อน

#### PR Description: โครงสร้างที่ project ส่วนใหญ่คาดหวัง

Pull Request description ควรตอบ 3 คำถามหลักเสมอ ไม่ว่า project จะมี template บังคับหรือไม่:

1. **What** — เปลี่ยนอะไรบ้าง (สรุปสั้น ๆ อาจซ้ำกับ commit message ก็ได้)
2. **Why** — ทำไมถึงต้องเปลี่ยน (แก้ bug ไหน? ตอบสนอง feature request ไหน?)
3. **How to verify** — reviewer จะทดสอบว่า PR นี้ทำงานถูกต้องได้อย่างไร (รัน test ไหน? หรือ reproduce ด้วยขั้นตอน
   อะไร?)

ตัวอย่าง PR description ที่ดี (สมมติสำหรับ PR ในหัวข้อ 105.10):

```markdown
## What

Fixes a small documentation inconsistency in `sub()`'s doc comment
(missing trailing period compared to `add()`) and adds a matching
unit test `test_sub`, which was missing even though every other
public function has one.

## Why

Noticed while reading through the crate for an unrelated task that
`sub()` was the only exported function without test coverage. Small,
low-risk fix — good first contribution to get familiar with the
project's workflow.

## How to verify

- `cargo test` — new `test_sub` passes alongside existing tests
- `cargo fmt --check` and `cargo clippy -- -D warnings` — both clean

Closes #42
```

PR นี้อ่านง่าย ไม่ต้องเดาอะไร reviewer เห็นทันทีว่า scope เล็กแค่ไหน มีความเสี่ยงต่ำแค่ไหน และจะ verify ได้อย่างไร
— PR แบบนี้มักได้รับการ review และ merge เร็วกว่า PR ที่เปิดมาโดยไม่มีคำอธิบายอะไรเลยมาก

เทียบกับตัวอย่าง PR description ที่ไม่ดี ซึ่งพบได้บ่อยจาก contributor ที่รีบเปิด PR โดยไม่คิดจากมุมมองของ
reviewer:

```markdown
fixed the thing we talked about in the issue, let me know if
anything else needed
```

ปัญหาของ description แบบนี้ชัดเจนมาก: ไม่บอกว่า "the thing" คืออะไร (reviewer ต้องเปิด issue ไปอ่านเองก่อนจะเข้าใจ
ว่า PR นี้แก้อะไร), ไม่มีข้อมูลว่าจะ verify การแก้ไขนี้ได้อย่างไร, และไม่ได้อ้างอิง issue number ด้วย keyword ที่
GitHub รู้จัก (`Closes #...`) ทำให้ issue ไม่ถูกปิดอัตโนมัติแม้ PR จะ merge ไปแล้ว reviewer ที่เจอ PR แบบนี้ต้องเสีย
เวลาเปิดหลายแท็บเพื่อปะติดปะต่อ context เอง ก่อนจะเริ่ม review โค้ดจริงได้เสียอีก — ในขณะที่ PR ที่มี description
ครบ What/Why/How-to-verify ทำให้ reviewer เริ่ม review โค้ดได้ทันทีโดยไม่ต้องเสียเวลาถามคำถามพื้นฐานก่อน

### 105.6 ผ่าน CI และ Code Review อย่างมืออาชีพ

#### ทำไมต้องรัน check ทุกตัวบนเครื่องตัวเอง**ก่อน**เปิด PR

จาก Part 97 เราเรียนไปแล้วว่า CI pipeline ของ project Rust จริงส่วนใหญ่ (ที่ใช้ GitHub Actions) มักตรวจ 4 อย่างเป็น
ขั้นต่ำ: `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo build`, และ `cargo test` — ทุกอย่างนี้**สามารถ
รันบนเครื่องของคุณเองได้ก่อน**โดยไม่ต้องรอ CI เลย

การเปิด PR ที่ CI แดง (fail) ทันทีที่เปิดสร้างความรำคาญให้ maintainer 2 แบบ: (1) เขาต้องเสียเวลามาเปิดดู log ของ
CI ที่ fail เพื่อบอกคุณว่าต้องแก้อะไร (ซึ่งคุณควรรู้เองได้ก่อนแล้ว) และ (2) มันบอกเป็นนัยว่าคุณยังไม่ได้ตรวจสอบ
งานของตัวเองก่อนส่งมา — สร้างความรู้สึกว่า PR นี้ "ยังไม่พร้อม" ให้ maintainer ตั้งแต่แรกเห็น

Checklist ที่ควรรันครบก่อนเปิด PR ทุกครั้ง (ตรงกับสิ่งที่ CI ของ project ส่วนใหญ่รันจริงตามที่เรียนใน Part 97
หัวข้อ 97.5):

```bash
cargo fmt --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test
cargo doc --no-deps   # เผื่อ doc comment มี syntax ผิดจนสร้างเอกสารไม่ได้
```

จากการรันจริงในสาธิตของบทนี้ (หัวข้อ 105.10) ผลลัพธ์ที่ได้คือ:

```
=== cargo fmt --check ===
OK: no formatting issues

=== cargo clippy --all-targets --all-features -- -D warnings ===
    Checking tiny_calc v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s

=== cargo test ===
running 2 tests
test tests::test_add ... ok
test tests::test_sub ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests tiny_calc
running 1 test
test src/lib.rs - add (line 7) ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured
```

ทุกอย่างผ่านหมดก่อนเปิด PR จริง — นี่คือมาตรฐานขั้นต่ำที่ควรทำทุกครั้ง ไม่ว่า project จะมี CI ที่เข้มงวดแค่ไหนก็ตาม

#### สาธิตจริง: เมื่อ Codebase มี Lint ค้างอยู่ก่อนที่เราจะเริ่มแก้

เพื่อให้เห็นภาพที่ตรงกับสถานการณ์จริงมากขึ้น (project ที่ไม่คุ้นเคยอาจมี lint ค้างอยู่ก่อนแล้วที่ไม่เกี่ยวกับงาน
ที่เรากำลังจะทำ) เราสร้าง crate สาธิตเล็ก ๆ อีกตัวชื่อ `clippy_demo` ที่จงใจมีโค้ดผิด lint 3 ชนิดที่ตรงกับตัวอย่าง
ในหัวข้อ 5.12/5.13 ของ Part 5 (`needless_return`, `needless_bool`, `len_zero`) แล้วรัน `cargo clippy` จริง:

```rust
fn is_even(n: i32) -> bool {
    if n % 2 == 0 {
        return true;
    } else {
        return false;
    }
}

fn main() {
    let v: Vec<i32> = vec![1, 2, 3];
    if v.len() == 0 {
        println!("empty");
    }
    println!("{}", is_even(4));
}
```

ผลลัพธ์จริงจาก `cargo clippy`:

```
warning: unneeded `return` statement
 --> src/main.rs:3:9
  |
3 |         return true;
  |         ^^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.98.0/index.html#needless_return
  = note: `#[warn(clippy::needless_return)]` on by default
help: remove `return`
  |
3 -         return true;
3 +         true
  |

warning: unneeded `return` statement
 --> src/main.rs:5:9
  |
5 |         return false;
...

warning: this if-then-else expression returns a bool literal
 --> src/main.rs:2:5
  |
2 | /     if n % 2 == 0 {
3 | |         return true;
4 | |     } else {
5 | |         return false;
6 | |     }
  | |_____^ help: you can reduce it to: `return n % 2 == 0`
  |
  = note: `#[warn(clippy::needless_bool)]` on by default

warning: length comparison to zero
  --> src/main.rs:11:8
   |
11 |     if v.len() == 0 {
   |        ^^^^^^^^^^^^ help: using `is_empty` is clearer and more explicit: `v.is_empty()`
   |
   = note: `#[warn(clippy::len_zero)]` on by default

warning: `clippy_demo` (bin "clippy_demo") generated 4 warnings (run `cargo clippy --fix --bin "clippy_demo" -p clippy_demo -- ` to apply 4 suggestions)
```

สังเกตว่า clippy ไม่ได้แค่บอกว่า "ผิด" — มันบอก**ตำแหน่งบรรทัดที่แน่นอน**, **ลิงก์ไปอ่านเหตุผลเชิงลึก**, และที่
สำคัญที่สุดคือ **เสนอ diff ที่แก้ให้พร้อมทันที** (บรรทัดที่มี `help: remove \`return\`` พร้อม `-`/`+` เหมือน git
diff) นี่คือเหตุผลที่ตรงกับที่ Part 5 อธิบายไว้ว่า `cargo clippy --fix` แก้ lint กลุ่ม `style`/`complexity` ให้
อัตโนมัติได้อย่างปลอดภัย — ลองรันจริง:

```bash
cargo clippy --fix --allow-dirty
```

```
    Checking clippy_demo v0.1.0
       Fixed src/main.rs (3 fixes)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.07s
```

โค้ดหลังแก้อัตโนมัติ:

```rust
fn is_even(n: i32) -> bool {
    n % 2 == 0
}

fn main() {
    let v: Vec<i32> = vec![1, 2, 3];
    if v.is_empty() {
        println!("empty");
    }
    println!("{}", is_even(4));
}
```

รัน `cargo clippy -- -D warnings` อีกครั้งเพื่อยืนยันว่าสะอาดจริง:

```
    Checking clippy_demo v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.03s
```

ไม่มี warning เหลือเลย — นี่คือ workflow ที่ควรทำจริงเมื่อเข้าไปทำงานกับ codebase ที่ไม่คุ้นเคยและพบว่ามี lint
ค้างอยู่ก่อนแล้ว **ข้อควรระวังสำคัญ:** ถ้า lint ที่ค้างอยู่ไม่เกี่ยวกับไฟล์ที่คุณกำลังจะแก้ในงานของคุณ **อย่ารีบไป
แก้ทั้งหมดในคราวเดียว** — ให้แก้เฉพาะไฟล์ที่เกี่ยวกับ PR ของคุณเท่านั้น เพราะการแก้ lint ทั่ว codebase จะทำให้ diff
ของ PR ใหญ่เกินความจำเป็นและปนกับการเปลี่ยนแปลงที่ตั้งใจทำจริง (ตรงกับกับดักข้อ 3 ในหัวข้อถัดไป) — ถ้าอยากช่วยแก้
lint ทั่ว codebase จริง ๆ ให้เปิดเป็น PR แยกต่างหากที่มี scope ชัดเจนว่า "chore: fix clippy lints across
codebase" เท่านั้น

ถ้า project มี `Cargo.lock` หรือ `target/` อยู่ใน working directory ระหว่างทำงาน ให้ตรวจสอบ `.gitignore` ของ
project ก่อน commit เสมอด้วย `git status` — ไฟล์ที่ build ขึ้นมา (เช่น `target/`) ไม่ควรถูก commit เข้าไปโดย
บังเอิญ เพราะจะทำให้ diff ของ PR ใหญ่และไม่เกี่ยวข้องกับการเปลี่ยนแปลงจริงเลย (ปัญหานี้พบได้บ่อยเวลา repo ใหม่ที่
ยังไม่มี `.gitignore` สมบูรณ์ — ควรเพิ่ม `target/` และ `Cargo.lock` ตาม convention ของ project นั้น ๆ ก่อน commit
ไฟล์ที่ไม่เกี่ยวข้อง)

#### สิ่งที่ CI ของ project ใหญ่มักตรวจเพิ่มเติมจาก 4 อย่างพื้นฐาน

ต่อจาก Part 97 ที่สอน pipeline แบบครบวงจร project จริงระดับ production มักมี CI job เพิ่มเติมที่ควรรู้จักไว้ก่อน
เปิด PR:

- **Matrix build ข้าม Rust channel** (`stable`, `beta`, `nightly`) และข้าม OS (Linux/macOS/Windows) — โค้ดที่ผ่าน
  บน Linux stable อาจไม่ผ่านบน Windows เพราะ path separator ต่างกัน (ตามที่เรียนใน Part 97 หัวข้อ 97.6 และกับดัก
  ข้อ 6)
- **MSRV (Minimum Supported Rust Version) check** — บาง project ประกาศว่ารองรับ Rust version เก่าสุดเท่าไหร่ และ
  มี CI job รันด้วย toolchain version นั้นโดยเฉพาะ เพื่อป้องกันไม่ให้ PR ใช้ feature ของ Rust version ใหม่โดยไม่
  ตั้งใจ
- **`cargo audit` / `cargo deny`** — ตรวจ dependency ใหม่ที่ PR เพิ่มเข้ามาว่าไม่มี known vulnerability และไม่ผิด
  license policy ของ project (ตามที่เรียนใน Part 100 และจะกล่าวถึงอีกครั้งในหัวข้อ 105.8)
- **Doc test และ `cargo doc` ต้อง build ผ่านโดยไม่มี broken link** — ถ้า PR แก้ doc comment ควรรัน `cargo test
  --doc` และ `cargo doc` ก่อนเสมอ (ตามที่เรียนใน Part 34)

ถ้า CI ของ project ใด job หนึ่ง fail หลังเปิด PR แล้ว (เช่น matrix build บน Windows fail แต่ผ่านบน Linux) ให้เปิด
ดู log ของ job ที่ fail อย่างละเอียดก่อนเดา — GitHub Actions แสดง log แบบ step-by-step และมักบอกบรรทัดที่ error
ชัดเจนอยู่แล้ว

ตารางสรุปสิ่งที่ควรตรวจสอบบนเครื่องตัวเองก่อนเปิด PR เทียบกับ CI job ที่มักตรวจสิ่งเดียวกันฝั่ง project (อ้างอิงจาก
รูปแบบ workflow ที่เรียนใน Part 97):

| ตรวจบนเครื่องตัวเองด้วย | ตรงกับ CI job ชื่อประมาณนี้ | ถ้า fail มักแปลว่า |
|---|---|---|
| `cargo fmt --check` | `fmt` / `rustfmt` | มีโค้ดที่ format ไม่ตรง `rustfmt.toml` |
| `cargo clippy -- -D warnings` | `clippy` / `lint` | มี lint ที่ project ตั้งเป็น hard error |
| `cargo test` | `test` (มักมี matrix ตาม OS) | test fail จริง หรือ fail เฉพาะบาง OS |
| `cargo build --all-features` | `build` | โค้ดใช้ feature flag ผิดหรือลืม `cfg` guard |
| `cargo doc --no-deps` | `docs` | doc comment มี syntax ผิดหรือ broken intra-doc link |
| `cargo audit` / `cargo deny check` | `security` / `deny` | dependency ใหม่มี vulnerability หรือ license ที่ผิด policy |

การรัน checklist ฝั่งซ้ายให้ครบก่อนเปิด PR ทุกครั้งเท่ากับการ "รัน CI ล่วงหน้า" บนเครื่องตัวเอง — ลด surprise
เกือบทั้งหมดที่อาจเกิดขึ้นตอน CI จริงรันบน GitHub Actions

#### Etiquette ของการรับ Code Review

Code review คือส่วนที่มือใหม่หลายคนรู้สึกกดดันที่สุด แต่ถ้าเข้าใจ norm ที่ถูกต้องจะรู้สึกดีขึ้นมาก:

**1. อย่าเอาความเห็นเชิงเทคนิคมาเป็นเรื่องส่วนตัว** — เมื่อ reviewer comment ว่า "ควรใช้ `Vec::with_capacity`
แทนเพราะเรารู้ขนาดล่วงหน้าอยู่แล้ว" นั่นคือ comment เกี่ยวกับ**โค้ด** ไม่ใช่การตัดสินความสามารถของคุณในฐานะคน
เขียนโค้ด maintainer ที่ดีจะ comment แบบนี้กับ PR ของทุกคนรวมถึง contributor ที่มีประสบการณ์สูงเช่นกัน — การ
review อย่างละเอียดคือสัญญาณว่า maintainer**เอาจริง**กับ PR ของคุณ ไม่ใช่สัญญาณว่าคุณเขียนโค้ดแย่

**2. ถ้า feedback ไม่ชัดเจน ให้ถามกลับอย่างสุภาพแทนการเดา** — ตัวอย่างการถามที่ดี: "ขอบคุณสำหรับ feedback ครับ/ค่ะ
ผม/ฉันไม่แน่ใจว่าหมายถึงให้เปลี่ยนที่ function `parse_input` เท่านั้น หรือรวม `parse_header` ด้วย ช่วยยืนยันได้
ไหมครับ/คะ" การถามแบบนี้ดีกว่าการเดาแล้วแก้ผิดทาง เพราะทำให้ maintainer ไม่ต้อง review ซ้ำสองรอบจากการเข้าใจผิด

ตัวอย่างบทสนทนา review ที่เป็นแบบอย่างที่ดี (สมมติขึ้นเพื่อแสดง pattern การสื่อสารที่เหมาะสม):

```
Reviewer: This clones the Vec on every call, which seems unnecessary
since we only read from it. Could we take a reference instead?

Contributor: Good catch — I clone it because the caller in
`process_batch` mutates the original Vec right after this call
returns, and I wasn't sure if that could invalidate a borrowed
reference. Would `&[T]` work here, or is there a lifetime concern
I'm missing?

Reviewer: Ah, that context helps. Since `process_batch` doesn't
mutate until after this function fully returns, a `&[T]` reference
is safe — the borrow ends when this function's scope ends. Let's
use that instead of cloning.

Contributor: Makes sense, updated in the latest commit. Thanks for
walking through it!
```

บทสนทนานี้เป็นตัวอย่างที่ดีเพราะทั้งสองฝั่งอธิบายเหตุผลของตัวเองอย่างชัดเจน (ไม่ใช่แค่ "ทำตามที่บอก" หรือ "ไม่เห็น
ด้วย" แบบไม่มีเหตุผล) — contributor ไม่ได้แก้ตามคำสั่งทันทีโดยไม่เข้าใจ แต่อธิบาย concern ของตัวเองก่อน (เรื่อง
lifetime) ทำให้ reviewer เห็นภาพและอธิบายกลับได้ตรงจุด สุดท้ายทั้งสองฝั่งเรียนรู้อะไรบางอย่างจากกันและกัน — นี่คือ
เป้าหมายที่แท้จริงของ code review ไม่ใช่แค่การ "ผ่านการตรวจ" เท่านั้น

**3. เข้าใจ norm ของ "squash and re-push" vs "add new commits" — ต้องเช็คกับ project แต่ละที่** — มี 2 pattern
ที่ project ใช้กันในการตอบ review feedback:
   - **Squash and force-push** — แก้ไขแล้ว amend หรือ squash เข้ากับ commit เดิม แล้ว force-push ทับ branch เดิม
     (`git push --force-with-lease`) ทำให้ PR มี commit history ที่สะอาด เหมาะกับ project ที่ต้องการ 1 PR = 1
     commit สุดท้ายที่ merge (squash merge)
   - **Add new commits ทับไปเรื่อย ๆ** — แก้ไขแล้ว commit ใหม่ต่อท้าย (ไม่ force-push) ทำให้ reviewer เห็นได้ว่า
     "อะไรถูกแก้ไปตาม feedback รอบไหน" ง่ายกว่า เหมาะกับ project ที่ reviewer ต้องการ track ว่า comment ไหนถูก
     resolve แล้วบ้าง

   วิธีรู้ว่า project ไหนใช้ pattern ไหน: อ่าน `CONTRIBUTING.md` (มักระบุตรง ๆ) หรือดูจาก PR อื่นที่ merge ไปแล้ว
   ว่า reviewer ขอให้ contributor ทำแบบไหน — ถ้าไม่แน่ใจเลย **ให้ถามใน PR ตรง ๆ** ว่า "อยากให้ผม squash commit
   ทั้งหมดหรือเพิ่ม commit ใหม่ต่อไปครับ/คะ" — คำถามแบบนี้ไม่มีใครมองว่าเสียเวลา ตรงกันข้ามคือมองว่าเป็น
   contributor ที่ใส่ใจ convention ของ project

**4. Resolve conversation thread เมื่อแก้ตามที่ขอแล้ว** — GitHub มีปุ่ม "Resolve conversation" สำหรับทุก comment
thread ใน PR หลังแก้โค้ดตาม feedback แล้ว ควรกดปุ่มนี้ (หรือถ้าไม่แน่ใจว่าแก้ถูกทางหรือยัง ให้ comment กลับถามก่อน
resolve) — การปล่อย thread ค้างไว้เยอะ ๆ โดยไม่จัดการทำให้ maintainer หลง track ว่าอะไร resolve แล้วบ้าง

### 105.7 การเคารพ `CONTRIBUTING.md` และ Code of Conduct

#### `CONTRIBUTING.md` มีไว้ทำไม

`CONTRIBUTING.md` คือเอกสารที่ maintainer เขียนขึ้นเพื่อตอบคำถามที่ contributor ใหม่ทุกคนถามซ้ำ ๆ ล่วงหน้า — การ
มีไฟล์นี้ช่วยประหยัดเวลา maintainer ได้มาก (ไม่ต้องตอบคำถามเดียวกันซ้ำ ๆ ใน issue ทุกครั้ง) และช่วย contributor
ประหยัดเวลาเช่นกัน (ไม่ต้องเดาว่า project คาดหวังอะไร) เนื้อหาที่ `CONTRIBUTING.md` ของ project Rust ทั่วไปมักมี:

- คำสั่งสำหรับ build/test project (บาง project ต้องมี dependency ระบบเพิ่มเติม เช่น `libssl-dev` หรือ Docker)
- Convention เรื่อง branch naming และ commit message ตามที่พูดถึงในหัวข้อ 105.5
- ขั้นตอนการรัน `cargo fmt`/`cargo clippy` ก่อน submit (บาง project มี pre-commit hook สคริปต์ให้ใช้)
- นโยบายเรื่อง test coverage สำหรับ feature ใหม่

ตัวอย่างโครงสร้างของ `CONTRIBUTING.md` ที่พบได้บ่อยใน crate ระดับกลาง (ผสมผสานจากรูปแบบที่ crate จริงในวงการ Rust
หลายตัวใช้ ไม่ใช่ copy จาก project เดียว):

```markdown
# Contributing to this project

Thanks for your interest in contributing! Here's how to get started.

## Development setup

1. Clone the repo and run `cargo build` to make sure everything compiles.
2. Run `cargo test` before making any changes to confirm the baseline
   is green on your machine.

## Before submitting a PR

- Run `cargo fmt` and `cargo clippy --all-targets -- -D warnings`.
  CI will reject PRs that don't pass both.
- Add tests for any new behavior. PRs without tests for new
  functionality will be asked to add them before review continues.
- Add an entry to `CHANGELOG.md` under `## [Unreleased]` if your
  change affects the public API.

## Commit messages

We follow imperative-mood commit messages (see CONTRIBUTING style in
most Rust projects). Reference the issue number with `Closes #123`
where applicable.

## Code of Conduct

This project follows the Contributor Covenant. See CODE_OF_CONDUCT.md.
```

การอ่านไฟล์แบบนี้ให้ครบก่อนเริ่มเขียนโค้ดสักบรรทัดเดียวช่วยป้องกันปัญหาที่พบบ่อยที่สุดของมือใหม่ — การเปิด PR ที่
CI แดงทันทีเพราะไม่รู้ว่า project ต้องการอะไรก่อน (ตรงกับกับดักข้อ 4 ในหัวข้อท้ายบท)

#### ตัวอย่าง convention จริงที่พบบ่อยใน Rust ecosystem

**1. บังคับให้เพิ่ม entry ใน `CHANGELOG.md`** — หลาย crate ที่ publish บน crates.io (เช่นที่ตามแนวทาง
[Keep a Changelog](https://keepachangelog.com/)) กำหนดให้ทุก PR ที่เปลี่ยนพฤติกรรม public API ต้องเพิ่มบรรทัดใน
`CHANGELOG.md` ใต้หัวข้อ `## [Unreleased]` เช่น:

```markdown
## [Unreleased]

### Fixed
- Fix panic on multi-byte UTF-8 boundary in tokenizer (#128)
```

เหตุผล: `CHANGELOG.md` เป็นสิ่งที่ user ของ crate อ่านตอน upgrade version เพื่อรู้ว่ามีอะไรเปลี่ยนไป การให้
contributor เขียนตอนเปิด PR สะดวกกว่าให้ maintainer ต้องนั่งเขียนย้อนหลังทุก release

**2. บังคับต้องมี test สำหรับ functionality ใหม่ทุกครั้ง** — นี่คือ convention ที่พบเกือบทุก project ที่มีคุณภาพ
สูง — PR ที่เพิ่ม public function ใหม่โดยไม่มี test ประกอบมักถูกขอให้เพิ่ม test ก่อน merge เสมอ ไม่มีข้อยกเว้น

**3. ข้อกำหนดเรื่อง commit signing (`Signed-off-by` หรือ GPG signature)** — บาง project ขนาดใหญ่ (โดยเฉพาะที่อยู่
ภายใต้ Linux Foundation หรือองค์กรใหญ่) กำหนดให้ทุก commit ต้องมี `Signed-off-by: Your Name <email>` ท้าย commit
message ซึ่งเป็นส่วนหนึ่งของ **Developer Certificate of Origin (DCO)** — คุณยืนยันว่าคุณมีสิทธิ์ทาง legal ที่จะ
submit โค้ดนี้ (เช่น มันเป็นงานที่คุณเขียนเอง ไม่ได้ copy จากที่อื่นที่มี license ขัดกัน) เพิ่ม sign-off ได้ง่าย ๆ
ด้วย flag `-s`:

```bash
git commit -s -m "Fix panic on multi-byte UTF-8 boundary in tokenizer"
```

คำสั่งนี้จะเติมบรรทัด `Signed-off-by: Your Name <your.email@example.com>` ต่อท้าย commit message อัตโนมัติ (ดึงชื่อ
และ email จาก `git config user.name` / `git config user.email`) บาง project ตั้ง CI ให้ตรวจสอบว่าทุก commit มี
บรรทัดนี้หรือไม่ ถ้าไม่มีจะ fail CI ทันที (มักเรียก job นี้ว่า "DCO check")

#### Code of Conduct มีไว้ทำไม

Code of Conduct (มักอยู่ในไฟล์ `CODE_OF_CONDUCT.md`) กำหนดมาตรฐานพฤติกรรมที่คาดหวังในการสื่อสารภายใน community
ของ project — ส่วนใหญ่อ้างอิงจากแม่แบบที่ใช้กันแพร่หลายอย่าง [Contributor Covenant](https://www.contributor-covenant.org/)
สิ่งที่ระบุทั่วไปคือ ห้ามการคุกคาม (harassment), ห้ามการเหยียด, และกำหนดวิธีร้องเรียนถ้าเจอพฤติกรรมที่ผิด (มักมี
email ของทีมที่ดูแลเรื่องนี้แยกจาก maintainer ทั่วไป)

การเคารพ Code of Conduct สำคัญไม่ใช่แค่เพราะ "กฎ" แต่เพราะมันสร้าง**สภาพแวดล้อมที่ทำให้คนกล้าถามคำถามและกล้าทำ
ผิดพลาด** — ถ้า community ของ project ใดตอบคำถามมือใหม่ด้วยความหยาบคายบ่อย ๆ contributor ใหม่จะไม่กล้าถามและจะ
หายไปเงียบ ๆ ในที่สุด (สังเกตสัญญาณนี้ได้จากการอ่าน PR/issue เก่า ๆ ตามที่แนะนำในหัวข้อ 105.2 ข้อ 5)

### 105.8 Licensing Awareness

#### ทำไมต้องรู้ license ของ project ก่อน contribute

ก่อนจะ contribute โค้ดให้ project ไหน ควรเปิดดูไฟล์ `LICENSE` หรือ `LICENSE-MIT` / `LICENSE-APACHE` ที่ root ของ
repo ก่อนเสมอ — Rust ecosystem นิยม dual-license แบบ **MIT OR Apache-2.0** เป็นค่าเริ่มต้น (crate ที่สร้างด้วย
`cargo new` ไม่ได้ตั้ง license มาให้อัตโนมัติ แต่ convention ของ community ส่วนใหญ่ยึดถือแบบนี้) ความหมายเชิง
practical คือ: เมื่อคุณ submit PR เข้า project ที่ใช้ license แบบนี้ **คุณกำลังยินยอมให้โค้ดที่คุณเขียนถูกแจกจ่าย
ภายใต้ license เดียวกันของ project นั้นโดยอัตโนมัติ** (เว้นแต่ project มีข้อตกลงอื่นเป็นลายลักษณ์อักษร)

สิ่งนี้สำคัญเป็นพิเศษถ้าคุณกำลังจะ copy โค้ดบางส่วนมาจากที่อื่น (เช่น จาก Stack Overflow หรือ project อื่น) —
ต้องตรวจสอบว่า license ของแหล่งที่มาเข้ากันได้กับ license ปลายทางหรือไม่ การ copy โค้ดที่มี license แบบ copyleft
เข้มงวด (เช่น GPL) เข้าไปใน project ที่ใช้ MIT/Apache-2.0 อาจสร้างปัญหาทาง legal ให้ project นั้นได้จริง — ถ้าไม่
แน่ใจ ให้เขียนโค้ดขึ้นใหม่เองแทนการ copy โดยตรงเสมอ

จาก Part 100 เราเรียนเรื่อง `cargo deny` ไปแล้วในมุมของการตรวจสอบ dependency ทั้งหมดของ project ว่าไม่มี license
ที่ผิดนโยบาย (เช่น project อาจ deny license ประเภท copyleft ทั้งหมด) — หลักการเดียวกันนี้ใช้ได้กับตัวคุณเองในฐานะ
contributor เช่นกัน: ถ้า PR ของคุณเพิ่ม dependency ใหม่ ควรตรวจสอบ license ของ dependency นั้นก่อนด้วย `cargo
deny check licenses` (หรืออย่างน้อย `cargo license` ถ้า project ไม่ได้ตั้ง `cargo deny` ไว้) เพราะการเพิ่ม
dependency ที่มี license ขัดกับ policy ของ project อาจทำให้ PR ถูก reject ทันทีไม่ว่าโค้ดจะดีแค่ไหนก็ตาม

ในทางปฏิบัติ ถ้า project มีไฟล์ `deny.toml` อยู่แล้ว (ตามที่เรียนใน Part 100 หัวข้อ 100.3) วิธีตรวจสอบก่อนเปิด PR
ที่เพิ่ม dependency ใหม่คือ:

```bash
cargo install cargo-deny --locked   # ติดตั้งครั้งแรกถ้ายังไม่มี
cargo deny check licenses
```

ถ้า dependency ใหม่ที่เพิ่มเข้ามามี license ที่ project ไม่อนุญาต (เช่น project ตั้ง `deny = ["GPL-3.0"]` ไว้ใน
`deny.toml` แต่ dependency ใหม่ใช้ license นั้น) `cargo deny` จะรายงาน error ทันทีพร้อมชื่อ crate และ license ที่
เป็นปัญหา ทำให้เรารู้ตัวก่อนเปิด PR แทนที่จะให้ CI ของ project มาบอกเราทีหลัง (เสียเวลา review รอบหนึ่งไปเปล่า ๆ)
ถ้า project ไม่มี `cargo deny` ตั้งไว้เลย อย่างน้อยควรเปิดดู license ของ dependency ใหม่ด้วยตาเปล่าที่หน้า
crates.io ของมัน (ทุก crate หน้า crates.io จะแสดง license ไว้ชัดเจนตรงกล่องข้อมูลด้านขวา) ก่อน commit `Cargo.toml`
ที่เพิ่ม dependency นั้นเข้ามา

#### Contributor License Agreement (CLA) — ระดับที่ต้องรู้จักไว้ (Awareness)

บาง project ขนาดใหญ่ (โดยเฉพาะที่อยู่ภายใต้บริษัทหรือมูลนิธิ เช่น project บางตัวของ Google, Microsoft, หรือ
Apache Software Foundation) กำหนดให้ contributor ต้องเซ็น **Contributor License Agreement (CLA)** ก่อน PR แรกจะ
ถูก merge ได้ CLA คือเอกสารทาง legal ที่ contributor ยืนยันให้สิทธิ์กับองค์กรที่ดูแล project ในการใช้งานโค้ดที่
contribute ไป (มักครอบคลุมกว้างกว่า license ปกติของ project เล็กน้อย เพื่อป้องกันปัญหา legal ในอนาคตขององค์กร)

ในทางปฏิบัติ กระบวนการนี้มักถูกทำให้เป็น automation แล้ว — เมื่อคุณเปิด PR แรกกับ project ที่ต้องการ CLA, GitHub
bot (เช่น CLA Assistant) จะ comment อัตโนมัติในหน้า PR พร้อมลิงก์ให้ไปเซ็น CLA แบบ online (มักแค่ login ด้วย
GitHub account แล้วกดยอมรับ) — หลังเซ็นแล้ว bot จะ mark PR ว่า "CLA signed" และไม่ต้องทำซ้ำสำหรับ PR อื่นในอนาคต
กับ project เดียวกัน

ข้อสังเกตที่สำคัญ (**ระดับความรู้ทั่วไป ไม่ใช่คำแนะนำทาง legal**): CLA เป็นเรื่องที่ contributor ควรอ่านเอกสารจริง
ก่อนเซ็นเสมอ โดยเฉพาะถ้าโค้ดที่คุณจะ contribute เกี่ยวข้องกับงานที่คุณทำให้บริษัทที่คุณทำงานอยู่ (บางบริษัทมี
นโยบายภายในเกี่ยวกับ IP ที่พนักงานสร้างขึ้น ซึ่งอาจขัดกับการเซ็น CLA ให้บุคคลที่สามโดยไม่ได้รับอนุญาตจากบริษัทก่อน)
— ถ้าไม่แน่ใจเรื่อง legal ใด ๆ ควรปรึกษาผู้เชี่ยวชาญด้านกฎหมายจริง บทเรียนนี้ให้ความรู้พื้นฐานเพื่อให้คุณรู้ว่า
"ควรระวังเรื่องอะไร" เท่านั้น ไม่ใช่คำแนะนำทาง legal ที่ใช้อ้างอิงได้จริง

### 105.9 Contributing Beyond Code: เอกสาร, การ Triage Issue, และการ Review PR

การ contribute ไม่ได้จำกัดแค่การเขียนโค้ดใหม่ ในความเป็นจริง project ที่แข็งแรงต้องการความช่วยเหลือในหลายมิติ
พร้อมกัน และงานที่ไม่ใช่โค้ดหลายอย่างมี "friction" ต่ำกว่าการเขียนโค้ดใหม่มาก เหมาะเป็นจุดเริ่มต้นที่ดีมาก

#### เอกสาร: จุดเริ่มต้นที่มี Friction ต่ำที่สุด

จาก Part 34 เราเรียนรูปแบบมาตรฐานของ rustdoc ไปแล้ว (`///`, `//!`, `# Examples`, `# Panics`, `# Errors`) —
ความรู้นี้ทำให้คุณสามารถ contribute เอกสารได้ทันทีแม้จะยังไม่คุ้นเคยกับ logic ภายในของ codebase เลยก็ตาม ตัวอย่าง
การ contribute ด้านเอกสารที่มีค่าจริงและทำได้โดยไม่ต้องเข้าใจ codebase ทั้งหมด:

- เพิ่ม `# Examples` block ให้ public function ที่มีแต่คำอธิบายสั้น ๆ แต่ไม่มีตัวอย่างการใช้งาน (แล้วรัน `cargo
  test --doc` เพื่อยืนยันว่าตัวอย่างนั้น compile และรันได้จริงตามที่เรียนใน Part 34 หัวข้อ 34.7)
- แก้ประโยคที่กำกวมหรือ typo ใน `README.md` (โดยเฉพาะ project ที่ maintainer ไม่ได้ใช้ภาษาอังกฤษเป็นภาษาแรก
  มักมีจุดที่ประโยคแปลกเล็กน้อยที่ native speaker ช่วยแก้ได้)
- เพิ่ม intra-doc link (ตามที่เรียนใน Part 34 หัวข้อ 34.10) เชื่อมโยงระหว่าง type ที่เกี่ยวข้องกัน ทำให้ผู้อ่าน
  เอกสารบน docs.rs กระโดดไปมาระหว่าง type ได้สะดวกขึ้น
- เขียน crate-level doc (`//!` แบบ elevator pitch) ให้ crate ที่มี public API ครบแล้วแต่ไม่มีคำอธิบายภาพรวมที่
  หน้าแรกของเอกสารเลย

งานเหล่านี้มีค่าจริงเพราะเอกสารที่ดีลดจำนวนคำถามซ้ำ ๆ ที่ maintainer ต้องตอบใน issue — เอกสาร 1 ประโยคที่ชัดเจน
อาจประหยัดเวลา maintainer ได้หลายชั่วโมงจากการตอบคำถามเดียวกันซ้ำ ๆ ในอนาคต

#### Triage Issue: งานที่มีค่ามากแต่มักถูกมองข้าม

**Triage** คือกระบวนการจัดระเบียบ issue ที่เข้ามาใหม่ให้ maintainer จัดการง่ายขึ้น งาน triage ที่ contributor
ทั่วไป (ไม่จำเป็นต้องเป็น maintainer) ทำได้จริงมี 3 อย่าง:

**1. Reproduce bug ที่ถูก report** — เมื่อมีคน report bug โดยไม่ได้ให้ minimal reproducible example คุณช่วยได้
โดยพยายาม reproduce ตามคำอธิบาย แล้ว comment กลับว่า reproduce ได้จริงหรือไม่ พร้อม environment ที่ใช้ทดสอบ (OS,
Rust version จาก `rustc --version`) — comment แบบนี้มีค่ามากเพราะยืนยันว่า bug นี้ "จริง" ไม่ใช่แค่ปัญหาเฉพาะ
เครื่องคนที่ report

**2. ถามคำถามที่ชัดเจนขึ้นเมื่อ issue ขาดรายละเอียด** — issue จำนวนมากถูกปิดโดย "ไม่มีคำตอบ" เพราะ maintainer
ไม่มีเวลาถามคำถามตามกลับ การช่วยถามคำถามที่ตรงจุด (เช่น "คุณใช้ Rust version ไหน?" หรือ "ขอ minimal code ที่
reproduce ปัญหานี้ได้ไหม?") ช่วยให้ issue เคลื่อนไปข้างหน้าได้เร็วขึ้นมาก

**3. ติด label ที่เหมาะสม (ถ้ามี permission)** — contributor ที่ทำงานกับ project มาสักระยะอาจได้รับสิทธิ์ triage
(GitHub มี role "Triage" แยกจาก "Write" ที่ให้สิทธิ์ติด label/assign ได้โดยไม่ต้องมีสิทธิ์ merge code) — การติด
label เช่น `bug`, `needs-reproduction`, หรือ `duplicate` ช่วยให้ maintainer กรอง issue ที่ต้อง priority สูงได้เร็ว
ขึ้นมาก

ตัวอย่าง comment triage ที่มีคุณภาพ (สมมติสำหรับ issue ที่รายงาน panic แต่ไม่มี minimal example):

```markdown
Thanks for the report! I tried to reproduce this with the code you
shared but couldn't get it to panic on my end.

Environment I tested on:
- rustc 1.82.0 (stable)
- OS: Ubuntu 22.04

Could you share:
1. The exact input that triggers the panic (a minimal `fn main()`
   that reproduces it would be ideal)
2. The output of `cargo tree` for this project, in case a specific
   dependency version matters

Tagging as `needs-reproduction` until we can confirm.
```

Comment แบบนี้มีค่ามากเพราะทำ 3 อย่างพร้อมกัน: (1) แสดงว่าได้ลอง reproduce จริงแล้วไม่ใช่แค่พูดลอย ๆ (2) ระบุ
environment ที่ทดสอบอย่างชัดเจนเพื่อตัดตัวแปรเรื่อง version ออกไปก่อน (3) ถามคำถามที่เจาะจงพอที่ผู้ report จะตอบ
ได้ง่าย ไม่ใช่คำถามกว้าง ๆ ที่ทำให้ผู้ report ไม่รู้จะตอบอะไร — comment แบบนี้ทำได้แม้ไม่มีสิทธิ์ maintainer เลย
แค่ต้องมี GitHub account และความตั้งใจช่วยเหลือเท่านั้น

#### Review PR ของคนอื่น: ขั้นถัดไปเมื่อคุณคุ้นเคยกับ Project แล้ว

เมื่อคุณ contribute ให้ project หนึ่งมาสักระยะจนคุ้นเคยกับ codebase และ convention แล้ว การ review PR ของคนอื่น
(แม้ไม่มีสิทธิ์ merge เอง) ก็เป็น contribution ที่มีค่ามาก — GitHub อนุญาตให้ใครก็ตาม comment บน PR ของ project
public ได้ ไม่จำเป็นต้องมีสิทธิ์ maintainer เลย

การ review ที่มีประโยชน์ไม่จำเป็นต้องเป็น "approve/request changes" อย่างเป็นทางการ (สิทธิ์นี้มักจำกัดไว้ให้
maintainer) แต่คือ comment ที่ช่วยลดงานของ maintainer เช่น: ทดสอบ PR บนเครื่องตัวเองว่าทำงานถูกต้องจริง, ชี้ edge
case ที่ยังไม่ถูก cover ใน test, หรือถาม contributor คนอื่นเกี่ยวกับ design decision ที่ไม่ชัดเจน — การ review
แบบนี้สร้างชื่อเสียงในฐานะ "คนที่เข้าใจ project นี้จริง" ซึ่งมักนำไปสู่การถูกชักชวนให้เป็น maintainer ในอนาคต

### 105.10 Walkthrough แบบครบวงจร: จำลอง Contribution จริงตั้งแต่ต้นจนจบ

หัวข้อนี้รวบรวมทุกอย่างที่เรียนมาในบทนี้เข้าเป็น flow เดียว โดยใช้ตัวอย่างเดียวกันกับที่สาธิตคำสั่ง git ในหัวข้อ
105.4 — เพื่อความชัดเจน เราจะไล่ทวนทั้ง flow ตั้งแต่ต้นอีกครั้งแบบสรุป พร้อมเชื่อมทุกขั้นตอนเข้ากับหัวข้อที่
เกี่ยวข้อง

**สถานการณ์จำลอง:** สมมติมี crate เล็ก ๆ ชื่อ `tiny_calc` (เป็น open source project ที่มี "upstream" ซึ่งเรา**ไม่มี**
write access) เราต้องการ contribute แก้ documentation typo เล็ก ๆ ใน `sub()` และเพิ่ม test ที่ยังไม่มี — งานที่
ตรงตามหลักการในหัวข้อ 105.2 (เล็ก, scope ชัดเจน, มีประโยชน์จริง)

**ขั้นที่ 1 — ประเมิน project ก่อน (105.2):** ในสถานการณ์จริง ก่อนเริ่มงานเราจะเช็ค commit activity, การตอบสนอง
ของ maintainer ใน issue เก่า ๆ, และดูว่ามี `CONTRIBUTING.md` ไหม (สำหรับ `tiny_calc` ในตัวอย่างนี้ เราสมมติว่าผ่าน
เกณฑ์ทั้งหมดแล้ว)

**ขั้นที่ 2 — Fork และ clone (105.4):**

```bash
# ในโลกจริง: กดปุ่ม "Fork" บน GitHub ก่อน แล้ว clone จาก URL ของ fork
git clone <fork-url> tiny_calc && cd tiny_calc
git remote add upstream <upstream-url>
```

**ขั้นที่ 3 — สร้าง branch (105.4):**

```bash
git checkout -b fix/sub-doc-typo-and-test
```

**ขั้นที่ 4 — อ่านโค้ดที่เกี่ยวข้องก่อนแก้ (105.3):** เปิด `cargo doc --open --no-deps` ดูภาพรวม public API ก่อน
แล้วเปิด `src/lib.rs` อ่าน `sub()` และ test module ที่มีอยู่ พบว่า `add()` มี `test_add` แต่ `sub()` ไม่มี test
คู่กันเลย และ doc comment ของ `sub()` ขาด `.` ท้ายประโยคเมื่อเทียบกับ `add()`

**ขั้นที่ 5 — แก้ไขจริง:**

```diff
-/// Subtracts `b` from `a`
+/// Subtracts `b` from `a`.
 pub fn sub(a: i32, b: i32) -> i32 {
     a - b
 }
```

```diff
     fn test_add() {
         assert_eq!(add(2, 3), 5);
     }
+
+    #[test]
+    fn test_sub() {
+        assert_eq!(sub(5, 3), 2);
+    }
 }
```

**ขั้นที่ 6 — รัน check ทั้งหมดก่อน commit (105.6):**

```bash
cargo fmt --check   # → OK: no formatting issues
cargo clippy --all-targets --all-features -- -D warnings   # → Finished, ไม่มี warning
cargo test           # → 2 passed (test_add, test_sub) + 1 doc-test passed
```

ผลลัพธ์จริงจากการรันคำสั่งเหล่านี้ในสภาพแวดล้อมสาธิตของบทนี้ผ่านครบทุกตัว ตามที่แสดงไว้แล้วในหัวข้อ 105.6

**ขั้นที่ 7 — Commit ด้วยข้อความที่ดี (105.5):**

```bash
git add -A
git commit -m "Fix missing period in sub() doc comment and add test_sub

sub() was the only public function without a matching unit test and
its doc comment was missing the trailing period used elsewhere in the
file. Add test_sub to bring it to parity with add(), and fix the doc
comment punctuation.

Closes #42"
```

**ขั้นที่ 8 — Sync กับ upstream ล่าสุด แล้ว rebase (105.4):** ระหว่างที่เราทำงาน สมมติว่ามี PR อื่นถูก merge เข้า
upstream ไปแล้ว (เพิ่ม `mul()`) เราจึงต้อง sync ก่อนเปิด PR:

```bash
git fetch upstream
git checkout main
git merge --ff-only upstream/main   # → Fast-forward, ได้ mul() มาด้วย
git checkout fix/sub-doc-typo-and-test
git rebase main                      # → Successfully rebased
```

ผลลัพธ์จริงตามที่แสดงไว้ในหัวข้อ 105.4: rebase สำเร็จโดยไม่มี conflict เพราะการเปลี่ยนแปลงของเราอยู่ใน function
คนละตัว (`sub`) กับที่ upstream เพิ่มมาใหม่ (`mul`)

**ขั้นที่ 9 — รัน check อีกครั้งหลัง rebase (สำคัญมาก เพราะ rebase อาจทำให้เกิด conflict ที่ resolve ผิด):**

```bash
cargo fmt --check && cargo clippy --all-targets --all-features -- -D warnings && cargo test
```

ทุกอย่างผ่านเหมือนก่อน rebase — ยืนยันว่า commit ของเรายัง compatible กับโค้ดล่าสุดของ upstream

**ขั้นที่ 10 — Push ไปที่ fork:**

```bash
git push origin fix/sub-doc-typo-and-test
```

**ขั้นที่ 11 — เปิด Pull Request (คำอธิบายเชิงข้อเท็จจริงของ UI บน GitHub — ไม่ได้ทำจริงในสาธิตนี้เพราะเป็น
external service):** หลัง push สำเร็จ ถ้า `origin` เป็น fork จริงบน GitHub หน้า repo ของ fork จะแสดง banner
"Compare & pull request" ปรากฏขึ้นให้กดโดยอัตโนมัติ (พฤติกรรมนี้เป็นฟีเจอร์มาตรฐานของ GitHub UI ที่รู้จักกันดี) กด
เข้าไปจะเจอฟอร์มให้กรอก PR title และ description — GitHub เติม title จาก commit message แรกให้อัตโนมัติ และถ้า
repo มี PR template (ไฟล์ `.github/PULL_REQUEST_TEMPLATE.md`) จะถูกใส่ในกล่อง description ให้กรอกตามแบบที่ project
กำหนด เราจะกรอก description ตามโครงสร้าง What/Why/How-to-verify ที่อธิบายไว้ในหัวข้อ 105.5 พร้อม `Closes #42`
ก่อนกดปุ่ม "Create pull request"

**ขั้นที่ 12 — รอ CI และ code review:** หลังเปิด PR, GitHub Actions ของ upstream จะรัน CI pipeline อัตโนมัติตาม
ที่ตั้งไว้ (ตาม Part 97) — ถ้า CI เขียวหมดและ maintainer review แล้วขอแก้ไขอะไร ให้ทำตาม etiquette ที่อธิบายไว้ใน
หัวข้อ 105.6 (ถามถ้าไม่ชัดเจน, squash หรือ add commit ตาม convention ของ project, resolve thread เมื่อแก้แล้ว)

**สรุป flow ทั้งหมด:** ประเมิน project → fork/clone → สร้าง branch → อ่านโค้ดที่เกี่ยวข้อง → แก้ไขเล็ก ๆ ที่มี
scope ชัดเจน → รัน check ทั้งหมดก่อน commit → commit ด้วยข้อความที่ดี → sync/rebase กับ upstream ล่าสุด → รัน check
อีกครั้งหลัง rebase → push → เปิด PR ที่มี description ครบ What/Why/How-to-verify → รับมือ code review อย่างเป็น
มืออาชีพ — นี่คือ flow เดียวกันที่ใช้ได้กับ project ทุกขนาดตั้งแต่ crate เล็ก ๆ ไปจนถึง project ระดับ `tokio` หรือ
`rust-lang/rust` เอง ต่างกันแค่รายละเอียดของ convention เฉพาะที่ระบุใน `CONTRIBUTING.md` ของแต่ละ project เท่านั้น

ข้อสังเกตปิดท้ายที่สำคัญ: ทุกคำสั่ง git ที่แสดงใน walkthrough นี้ (`fetch`, `merge --ff-only`, `rebase`,
`push --force-with-lease`) และผลลัพธ์ของ `cargo fmt`/`cargo clippy`/`cargo test` ที่แสดงไว้ **รันจริงบน local
repository ที่สร้างขึ้นเพื่อสาธิตในบทเรียนนี้โดยเฉพาะ** ไม่ใช่การจำลองข้อความขึ้นมาลอย ๆ — เหตุผลที่ทำแบบนี้คือ
เพื่อให้คุณเห็น output ที่ตรงกับที่จะเจอจริงเป๊ะ ๆ เมื่อรันคำสั่งเดียวกันกับ fork/upstream จริงบน GitHub ในอนาคต
ส่วนขั้นตอนสุดท้าย (การเปิด PR จริงบน GitHub UI) ไม่ได้ทำจริงในบทเรียนนี้เพราะไม่มี external repository ที่เหมาะสม
จะใช้สาธิตอย่างปลอดภัย แต่คำอธิบายเรื่อง UI ที่ให้ไว้อ้างอิงจากพฤติกรรมจริงของ GitHub ที่เป็นที่รู้จักกันดี ไม่ใช่
การคาดเดา

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. เปิด PR ขนาดใหญ่เป็นครั้งแรกกับ project ที่ไม่คุ้นเคย

อาการ: ใช้เวลาหลายสัปดาห์เขียน refactor ใหญ่หรือ feature ใหญ่เป็น PR แรก แล้วถูก maintainer comment กลับว่า
"เราไม่ได้วางแผนจะไปทางนี้" หรือ PR ถูกปล่อยไว้เงียบ ๆ เป็นเดือนโดยไม่มีใคร review เพราะ reviewer ไม่มีเวลาพอจะ
review การเปลี่ยนแปลงขนาดใหญ่จากคนที่ยังไม่รู้จัก

วิธีแก้: ทำตามหลักการในหัวข้อ 105.2 — เริ่มจาก PR เล็กที่สุดที่มีประโยชน์จริงก่อนเสมอ (แก้ typo, เพิ่ม test เดียว,
แก้ bug เล็กที่ scope ชัดเจน) สร้างความคุ้นเคยและความน่าเชื่อถือก่อนจะขยับไปทำงานใหญ่ขึ้น

### 2. Force-push ทับ branch โดยไม่ใช้ `--force-with-lease` แล้วทับงานของคนอื่นโดยไม่ตั้งใจ

อาการ: หลัง rebase แล้ว push ด้วย `git push --force origin my-branch` ซึ่งอาจทับ commit ที่ maintainer เพิ่งแก้
ให้เราตรงบน branch เดียวกัน (บาง project ทำแบบนี้เพื่อประหยัดเวลา) ทำให้งานของ maintainer หายไปโดยไม่มีใครรู้ตัว
จนกว่าจะสังเกตเห็นทีหลัง

Error/สัญญาณที่ควรสังเกต: ถ้าใช้ `--force-with-lease` แทน จะได้ error ป้องกันไว้ก่อนแบบนี้:

```
! [rejected]        my-branch -> my-branch (stale info)
error: failed to push some refs to 'origin'
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref.
```

วิธีแก้: ใช้ `git push --force-with-lease origin my-branch` เสมอแทน `--force` เปล่า ๆ ตามที่อธิบายไว้ในหัวข้อ
105.4 — ถ้าเจอ error แบบข้างต้น ให้ `git fetch` แล้วดูว่ามีอะไรเปลี่ยนบน remote ก่อนตัดสินใจว่าจะ merge หรือ
rebase เอาการเปลี่ยนแปลงนั้นเข้ามาก่อน push ต่อ

### 3. เปิด PR โดยไม่รัน `cargo fmt`/`cargo clippy` ก่อน ทำให้ CI แดงทันที และ diff ปนกับการเปลี่ยน format

อาการ: PR ถูกเปิดขึ้นมาพร้อม CI status เป็นสีแดงทันที เพราะลืมรัน `cargo fmt --check` ก่อน หรือ diff ของ PR มีการ
เปลี่ยน indentation/whitespace ปนกับการเปลี่ยน logic จริง ทำให้ reviewer แยกไม่ออกว่าอะไรคือการเปลี่ยนแปลงที่
ตั้งใจทำจริง ๆ

Error ตัวอย่างจาก `cargo fmt --check` เมื่อโค้ดยังไม่ถูก format:

```
Diff in /path/to/src/lib.rs:12:
-pub fn sub(a:i32,b:i32)->i32{
+pub fn sub(a: i32, b: i32) -> i32 {
     a - b
 }
```

วิธีแก้: รัน checklist เต็มตามหัวข้อ 105.6 ก่อนเปิด PR ทุกครั้งไม่มีข้อยกเว้น (`cargo fmt --check`, `cargo clippy
--all-targets --all-features -- -D warnings`, `cargo test`) ถ้าโปรเจกต์เดิมไม่ได้ format มาก่อน ให้แยก
`cargo fmt` เป็น commit เดี่ยว ๆ ที่ไม่ปนกับการเปลี่ยน logic (ตามที่แนะนำใน Part 5)

### 4. ไม่อ่าน `CONTRIBUTING.md` ก่อนเริ่มทำงาน แล้วทำผิด convention ที่ระบุไว้ชัดเจนแล้ว

อาการ: ส่ง PR ที่ commit message ไม่มี `Signed-off-by` ทั้งที่ project กำหนด DCO ไว้ชัดเจนใน `CONTRIBUTING.md`
หรือลืมเพิ่ม entry ใน `CHANGELOG.md` ทั้งที่ project กำหนดไว้ว่าทุก PR ต้องมี ทำให้ maintainer ต้อง comment ขอให้
แก้ไขเพิ่มก่อนจะ review ต่อ เสียเวลาไปกลับไปกลับมาโดยไม่จำเป็น

ตัวอย่าง CI check ที่ fail เพราะไม่มี sign-off (จาก DCO bot):

```
❌ DCO check failed
1 commit(s) are missing Signed-off-by:
  - 8062ca5 Fix missing period in sub() doc comment and add test_sub

Please amend your commit(s) to add "Signed-off-by: Your Name <email>"
```

วิธีแก้: อ่าน `CONTRIBUTING.md` ให้ครบ**ก่อน**เริ่มเขียนโค้ดสักตัวเดียว ไม่ใช่หลังเปิด PR แล้ว — ถ้าพลาดไปแล้ว
สามารถแก้ commit เดิมด้วย `git commit --amend -s` (ถ้ายังไม่ push) หรือ `git rebase --exec 'git commit --amend
--no-edit -s' <base>` (ถ้ามีหลาย commit ที่ต้องเติม sign-off ทั้งหมด) แล้ว force-push ทับด้วย `--force-with-lease`

### 5. เปิด PR ซ้ำกับ issue ที่มีคนกำลังทำอยู่แล้ว (Duplicate Work)

อาการ: เห็น issue ที่ดูน่าสนใจ รีบเริ่มเขียนโค้ดทันทีโดยไม่เช็คก่อนว่ามีใคร comment ใน issue นั้นแล้วว่า "I'm
working on this" หรือมี PR อื่นที่เปิดค้างอยู่แล้วซึ่งแก้ปัญหาเดียวกัน สุดท้ายเสียเวลาไปกับงานที่ถูกทำไปแล้ว (หรือ
กำลังจะถูกทำ) และอาจสร้างความอึดอัดใจระหว่าง contributor สองคนที่บังเอิญมาชนกัน

สัญญาณที่ควรเช็คก่อนเริ่ม: เปิด tab "Pull requests" ของ repo แล้ว filter ด้วยเลข issue (`is:pr #42`) เพื่อดูว่ามี
PR ที่ reference issue นี้อยู่แล้วหรือไม่ และอ่าน comment ทั้งหมดใน issue เองก่อนเริ่มเขียนโค้ดเสมอ

วิธีแก้: ถ้า issue ยังไม่มีใคร assign หรือ comment ว่ากำลังทำอยู่ ให้ comment สั้น ๆ ก่อนเริ่มเขียนโค้ดว่า "I'd
like to take this on — will open a PR shortly" เพื่อบอก contributor คนอื่นว่ามีคนเริ่มทำแล้ว (การ comment แบบนี้
เป็น norm ที่พบทั่วไปในวงการ ไม่ใช่การผูกมัดทาง legal อะไร แค่เป็นมารยาทป้องกันการทำงานซ้ำกัน) ถ้าเผลอเริ่มไปแล้ว
พบว่ามีคนทำไปก่อน ก็ยังเป็นโอกาสเรียนรู้ที่ดี — ลองเปรียบเทียบวิธีแก้ของตัวเองกับ PR ที่ merge ไปแล้วเพื่อดูว่า
แนวทางต่างกันอย่างไรและทำไม

## แบบฝึกหัด (Exercises)

1. **[ง่าย]** เลือก crate สาธารณะที่คุณใช้งานอยู่บ่อย ๆ ใน Rust (เช่น crate ที่อยู่ใน `Cargo.toml` ของโปรเจกต์คุณ
   เอง) เปิดหน้า GitHub ของ crate นั้น แล้วประเมินตามเกณฑ์ 5 ข้อในหัวข้อ 105.2 (commit activity, การตอบสนอง
   maintainer, การมี `CONTRIBUTING.md`, จำนวน active maintainer, และบทสนทนาใน PR ที่ merge แล้ว) เขียนสรุปสั้น ๆ
   ว่า project นี้ "friendly" ต่อ contributor ใหม่ระดับไหน พร้อมหลักฐานที่พบจริงประกอบการประเมินแต่ละข้อ

2. **[กลาง]** สร้าง local git repository สองตัวในเครื่องตัวเอง (`upstream/` และ `fork/`) จำลองความสัมพันธ์แบบใน
   หัวข้อ 105.4 ด้วย crate เล็ก ๆ ของคุณเอง จากนั้นจำลองสถานการณ์ที่ upstream มี commit ใหม่เข้ามาระหว่างที่คุณ
   ทำงานอยู่บน feature branch แล้วฝึกคำสั่ง `git fetch upstream` → sync `main` → `git rebase main` จนสำเร็จโดยไม่มี
   conflict (hint: ให้การเปลี่ยนแปลงของคุณอยู่ใน function คนละตัวกับที่ upstream เปลี่ยน เพื่อหลีกเลี่ยง conflict
   ในรอบแรก แล้วลองทำซ้ำอีกครั้งโดยตั้งใจให้ทั้งสองฝั่งแก้ไฟล์บรรทัดเดียวกันเพื่อดู conflict marker ที่เกิดขึ้นจริง)

3. **[กลาง]** เขียน commit message 3 แบบสำหรับการแก้ bug เดียวกัน (สมมติ bug อะไรก็ได้ที่คุณเจอมาก่อน) — แบบแรก
   แย่มาก (สั้นเกินไป ไม่มี context), แบบที่สองยาวเกินไปแบบไม่มีโครงสร้าง, และแบบที่สามที่ดีตามหลักการในหัวข้อ
   105.5 (imperative mood, อธิบายทำไม, อ้างอิง issue) จากนั้นเขียนเหตุผลสั้น ๆ ว่าทำไมแบบที่สามดีกว่าอีกสองแบบ
   อย่างเป็นรูปธรรม (hint: ลองนึกภาพว่าอีก 6 เดือนข้างหน้ามีคนมา `git log` ย้อนหาว่า bug นี้ถูกแก้ตอนไหนและทำไม —
   ข้อความไหนช่วยคนนั้นได้จริง)

4. **[ยาก/ประยุกต์]** ทำ walkthrough แบบเต็มตามหัวข้อ 105.10 ด้วย crate ทดลองของคุณเอง (ไม่ใช่ `tiny_calc` ในบทเรียน
   — สร้าง crate ใหม่ที่มี function อย่างน้อย 3 ตัว) ให้ครบทุกขั้นตอนจริง: ตั้ง upstream/fork แบบ local, สร้าง
   branch, แก้ไขจริง (เช่น เพิ่ม doc example ที่ยังไม่มี, หรือแก้ bug เล็ก ๆ ที่คุณจงใจใส่ไว้ก่อน), รัน `cargo fmt`/
   `cargo clippy -- -D warnings`/`cargo test` ให้ผ่านทั้งหมด, commit ด้วยข้อความที่ดีตามหลักการ, จำลอง upstream
   เปลี่ยนแปลงแล้ว rebase, และเขียน PR description แบบ What/Why/How-to-verify ให้ครบ (ไม่ต้องเปิด PR จริงบน GitHub
   — แค่เขียน description เก็บไว้เป็นไฟล์ markdown ก็พอ) — hint: ลองตั้งใจให้ rebase รอบที่สองเจอ conflict จริง
   (แก้ไฟล์เดียวกันทั้งสองฝั่ง) แล้วฝึก resolve conflict ด้วยตัวเองจนจบ จากนั้นรัน check ทั้งหมดอีกครั้งหลัง resolve
   เพื่อยืนยันว่าไม่มีอะไรพังจากการ resolve conflict ผิดพลาด

## สรุป

ในบทนี้เราเปลี่ยนมุมจาก "การเขียนโค้ด Rust" ไปเป็น "การเข้าร่วม community ของ Rust ในฐานะ contributor จริง" ซึ่งเป็น
ทักษะที่แยกจากความรู้ด้านภาษาโดยสิ้นเชิง เราเริ่มจากเหตุผลที่แท้จริงในการ contribute — การเรียนรู้จากโค้ด
production-quality, การสร้าง portfolio ที่พิสูจน์ได้, การตอบแทนเครื่องมือที่ใช้ทุกวัน, และการเติบโตจาก code review
โดยผู้เชี่ยวชาญ — ก่อนจะลงรายละเอียดเชิงปฏิบัติ: วิธีหา project และ issue ที่เหมาะกับระดับความคุ้นเคยของตัวเอง,
เทคนิคอ่านโค้ดขนาดใหญ่อย่างมีระบบด้วยการเริ่มจาก test และ `cargo doc --open` (ต่อยอดจาก Part 34), git workflow
แบบ fork/upstream/rebase ที่สาธิตด้วยคำสั่งจริงและ output จริงจาก local repository สองตัว, การเขียน commit message
และ PR description ที่ดี, การผ่าน CI (ต่อยอดจาก Part 97) และรับมือ code review อย่างเป็นมืออาชีพ, ความสำคัญของการ
เคารพ `CONTRIBUTING.md` และ Code of Conduct, ความตระหนักเรื่อง license และ CLA (ต่อยอดจาก Part 100), และมุมมองที่
กว้างขึ้นว่า "การ contribute" ครอบคลุมทั้งเอกสาร, การ triage issue, และการ review PR ของคนอื่น ไม่ใช่แค่การเขียน
feature ใหม่

เราปิดท้ายด้วย walkthrough ที่จำลองการ contribute ที่สมบูรณ์ตั้งแต่ fork จนถึงจุดที่พร้อมเปิด PR จริง โดยใช้ local
git repository จำลอง upstream/fork อย่างปลอดภัย (ไม่แตะ repo จริงใด ๆ) — flow นี้เป็น flow เดียวกันที่ใช้ได้กับ
project ทุกขนาดใน Rust ecosystem ตั้งแต่ crate เล็ก ๆ ไปจนถึง `tokio` หรือ `rust-lang/rust` เอง

ใน **Part 106** เราจะเปลี่ยนมุมกลับมาที่เนื้อหาเชิงเทคนิคเข้มข้นอีกครั้ง ด้วยหัวข้อ **Rust Design Patterns สำหรับ
Enterprise Applications** — การนำ design pattern คลาสสิกจากโลก object-oriented (เช่น Builder, Strategy, Observer,
Repository) มาปรับใช้ให้เข้ากับ ownership model และ trait system ของ Rust อย่างเป็น idiomatic ซึ่งเป็นทักษะที่
สำคัญมากเมื่อคุณต้องทำงานกับ codebase ระดับ enterprise ที่ต้อง maintain ในระยะยาว — และทักษะการอ่านโค้ดคนอื่นที่
เราฝึกในบทนี้จะมีประโยชน์อย่างมากตอนอ่านตัวอย่าง pattern ที่ซับซ้อนขึ้นใน Part หน้า

---

**Part ก่อนหน้า:** [Blockchain และ Smart Contracts ด้วย Rust](part-104-blockchain-smart-contracts.md) | **Part ถัดไป:** [Rust Design Patterns สำหรับ Enterprise Applications](part-106-enterprise-design-patterns.md)
