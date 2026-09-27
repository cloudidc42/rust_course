# Part 104: Blockchain และ Smart Contracts ด้วย Rust

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 260 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า **blockchain คือโครงสร้างข้อมูลอะไรกันแน่** ในระดับ engineering ล้วน ๆ (ไม่ใช่การพูดถึงราคา
  เหรียญหรือการลงทุน) — มันคือ linked list ของ block ที่แต่ละ block เก็บ hash ของ block ก่อนหน้าไว้ ทำให้เกิด
  คุณสมบัติ **tamper-evidence**: แก้ข้อมูลใน block เก่าเพียงจุดเดียว hash ของ block นั้นจะเปลี่ยน และ
  `previous_hash` ที่ block ถัดไปเก็บไว้จะไม่ตรงกันอีกต่อไป ทำให้การปลอมแปลงถูกตรวจจับได้ทันทีโดยไม่ต้องมีใครมา
  ยืนยันแบบ manual
- เขียน `Block` และ `Blockchain` struct ด้วย Rust ตั้งแต่ศูนย์ที่**คำนวณ hash จริงด้วย crate `sha2`** (ไม่ใช่
  mock/placeholder) ต่อยอดจากความรู้เรื่อง serde ใน Part 57 มาใช้ serialize ข้อมูล block ก่อน hash และพิสูจน์
  คุณสมบัติ tamper-evidence ด้วยการรันโค้ดจริงที่แสดงว่า chain กลายเป็น invalid ทันทีที่มีการแก้ไขข้อมูลเก่า
- อธิบายและ implement **Proof of Work** ได้จริง: เข้าใจว่า `nonce` คือตัวเลขที่ปรับค่าไปเรื่อย ๆ เพื่อหา hash ที่
  ขึ้นต้นด้วยเลข 0 ตามจำนวนที่ "difficulty" กำหนด และ**วัดเวลาจริง**ว่า difficulty ที่เพิ่มขึ้นทำให้เวลาค้นหาโตขึ้น
  แบบ exponential ไม่ใช่ linear — ด้วยแนวคิดการวัดจริงแทนการเดา ตามที่ Part 54 (Performance & Benchmarking) สอนไว้
- อธิบายได้ว่าทำไม Rust ถึงเป็นภาษาที่ถูกเลือกใช้จริงในงาน blockchain infrastructure ระดับ production (Solana
  validator client, Polkadot/Substrate) โดยอ้างอิงเหตุผลเชิงเทคนิคที่จับต้องได้ (performance ของ hashing/signature
  verification ที่ต้องทำนับล้านครั้งต่อวินาที และ memory safety ในโค้ดที่ bug หนึ่งจุดอาจแปลว่าเงินหายจริง)
- Sign และ verify transaction จำลองด้วย **digital signature จริง** ผ่าน crate `ed25519-dalek` (private key เซ็น,
  public key ตรวจ) และอธิบายได้ว่ากลไกนี้**เหมือนกันโดยหลักการ**กับ JWT แบบ RS256 asymmetric signing ที่ Part 74
  สอนไว้ทุกประการ ต่างกันแค่ context การใช้งาน
- สร้างและตรวจสอบ **Merkle tree/Merkle proof** จริงด้วยมือ เข้าใจว่าทำไมมันทำให้พิสูจน์ได้ว่า transaction หนึ่งรายการ
  อยู่ใน block จริงโดยไม่ต้องดาวน์โหลดข้อมูลทั้ง block
- อธิบายได้อย่างแม่นยำว่า **smart contract ต้อง deterministic 100%** เพราะทุก node ในเครือข่ายต้องคำนวณผลลัพธ์
  เดียวกันเป๊ะจาก transaction เดียวกัน และเชื่อมโยงได้ว่าทำไม floating point (ที่ Part 3 เตือนไว้เรื่อง
  `0.1 + 0.2 != 0.3`), เวลาจริงของระบบ, และ random number generator ทั่วไป ถึงเป็นสิ่งที่ใช้ใน smart contract
  ไม่ได้
- เขียนและ**ทดสอบ smart contract logic จริงแบบ local** ด้วย CosmWasm (framework สำหรับเขียน smart contract ด้วย
  Rust compile ไปเป็น WebAssembly) ผ่าน unit test ที่รันด้วย `cargo test` ล้วน ๆ ไม่ต้องมี blockchain node จริงรัน
  อยู่เลย พร้อมเข้าใจตรงไปตรงมาว่าอะไรถูกพิสูจน์แล้วจริง (logic ของ contract ผ่านการทดสอบ) และอะไรยังไม่ถูกพิสูจน์
  (การ deploy ขึ้น chain จริงที่ต้องมี account/token จริง)
- เชื่อมโยง WASM ที่ CosmWasm ใช้เป็น target compile เข้ากับความรู้เรื่อง WebAssembly ที่ Part 86 สอนไว้ (sandboxed,
  deterministic, portable execution) และเห็นว่าทักษะจาก Module 5 ถูกใช้ตรงในบริบทใหม่นี้จริง ๆ
- วางขอบเขตความรู้ของตัวเองอย่างมีสติ: บทนี้ให้ "ความรู้กว้างระดับตระหนักรู้ (broad awareness)" เกี่ยวกับสาขาที่ใหญ่
  และเปลี่ยนเร็วมาก ไม่ใช่หลักสูตรที่ทำให้พร้อม deploy สู่ production ได้ทันที และเข้าใจว่าทำไมความปลอดภัยใน
  smart contract ถึงมีความเสี่ยงสูงกว่าโค้ดทั่วไปมาก (เชื่อมกับ Part 100)

## ความรู้ที่ต้องมีมาก่อน

- **Part 57-58 (Serialization: Serde เบื้องต้น/ขั้นสูง)** — ทุกครั้งที่บทนี้ hash ข้อมูลของ block หรือ transaction
  เราต้องแปลงมันเป็น byte sequence ที่แน่นอนก่อนเสมอ และเราใช้ `#[derive(Serialize, Deserialize)]` กับ
  `serde_json::to_string`/`to_vec` ตรงตามที่ Part 57 สอนไว้ทุกประการ — ถ้ายังไม่แน่นเรื่อง serde ควรทวนก่อน เพราะบท
  นี้จะไม่อธิบายกลไก derive macro ของ serde ซ้ำจากศูนย์
- **Part 54 (Performance & Benchmarking)** — หัวข้อ Proof of Work ของบทนี้ใช้แนวคิดเดียวกับที่ Part 54 ปูไว้ตรง ๆ:
  "วัดจริง อย่าเดา" เราจะใช้ `std::time::Instant` วัดเวลาการขุด block จริงที่ difficulty ต่าง ๆ กัน แทนการอธิบาย
  ด้วยทฤษฎีอย่างเดียว
- **Part 51 (Atomics และ Lock-free Programming)** — เนื้อหาเรื่อง hash function และการคำนวณระดับ byte ที่บทนี้ใช้
  (SHA-256 ผ่าน crate `sha2`) อยู่ในตระกูลเดียวกับความรู้เรื่อง byte-level operation และ low-level correctness ที่
  Part 51 ปูพื้นไว้ แม้จะไม่ได้ใช้ atomic โดยตรงในบทนี้ก็ตาม
- **Part 74 (Authentication: JWT)** — หัวข้อ digital signature ของบทนี้อ้างอิงตรงกับสิ่งที่ Part 74 สอนไว้เรื่อง
  RS256 (asymmetric signing: private key เซ็น, public key ตรวจ) — กลไกทางคณิตศาสตร์เบื้องหลังต่างกัน (RSA กับ
  elliptic curve) แต่**หลักการเหมือนกันเป๊ะ** ถ้ายังไม่เข้าใจว่าทำไม JWT RS256 ถึงให้ "ใครก็ตรวจได้ แต่มีแค่คนเดียว
  ที่สร้างได้" ควรทวน Part 74 ก่อน
- **Part 3 (ตัวแปร ชนิดข้อมูลพื้นฐาน และ Mutability)** — บทนี้จะยกตัวอย่าง `0.1_f64 + 0.2_f64` ที่ Part 3 เคยเตือน
  เรื่องความไม่แม่นยำของ floating point ขึ้นมาอีกครั้ง แต่ในบริบทใหม่ที่ผลกระทบร้ายแรงกว่าเดิมมาก: ใน smart
  contract ความไม่แน่นอนแบบนี้ทำให้ node ต่าง ๆ ในเครือข่ายเห็นผลลัพธ์ต่างกัน และ consensus พังทันที
- **Part 86 (WebAssembly เบื้องต้นด้วย Rust)** — CosmWasm ที่บทนี้ใช้ทำ smart contract จริง compile ไปเป็น `.wasm`
  ด้วย target `wasm32-unknown-unknown` ตัวเดียวกันที่ Part 86 สอนไว้ทุกประการ ถ้ายังไม่เข้าใจ linear memory หรือ
  ทำไม WASM ถึง sandboxed/deterministic/portable ควรทวน Part 86 ก่อน เพราะบทนี้จะอ้างอิงความรู้นั้นตรง ๆ ไม่อธิบาย
  ซ้ำ
- **Part 100 (Security Best Practices ใน Rust)** — หัวข้อท้ายบทเรื่องความปลอดภัยของ smart contract อ้างอิงกับ
  หลักการที่ Part 100 สอนไว้ว่า memory safety ของ Rust ป้องกัน bug บางประเภทเท่านั้น ไม่ได้ป้องกัน logic bug — ใน
  บริบท smart contract logic bug คือสิ่งที่อันตรายที่สุด เพราะแก้ไขย้อนหลังไม่ได้เมื่อ deploy ขึ้น chain จริงแล้ว
- **Part 41-43 (Unsafe, Raw Pointers, FFI)** — แนวคิดเรื่อง `extern "C"` function ที่ FFI ใช้ (Part 43) จะช่วยให้
  เข้าใจ `#[entry_point]` ของ CosmWasm ได้เร็วขึ้น เพราะมันทำงานบนหลักการเดียวกัน (export function ให้ระบบภายนอก
  เรียกเข้ามาได้ผ่าน boundary ที่ชัดเจน) แม้จะไม่บังคับว่าต้องรู้มาก่อนก็ตาม

## เนื้อหา

### 104.1 Blockchain คืออะไรกันแน่ — มองทะลุ hype ไปที่โครงสร้างข้อมูล

คำว่า "blockchain" ถูกใช้พร่ำเพรื่อมากจนความหมายทางเทคนิคของมันเลือนไปเยอะ ก่อนเขียนโค้ดสักบรรทัดเดียว เราต้องตั้ง
คำถามให้ตรงจุดก่อน: **ถ้าตัดคำโฆษณาออกให้หมด blockchain คือโครงสร้างข้อมูลอะไรกันแน่?**

คำตอบตรง ๆ คือ: **blockchain คือ linked list ของ block ที่แต่ละ block เก็บ cryptographic hash ของ block ก่อนหน้า
ไว้เป็นส่วนหนึ่งของข้อมูลตัวเอง** นั่นคือทั้งหมด ไม่มีอะไรลึกลับไปกว่านี้ในระดับโครงสร้างข้อมูลพื้นฐาน

ลองเทียบกับ linked list ธรรมดาที่เรียนกันมาตั้งแต่ Part 13 (Vec) และ Part 27-29 (Smart Pointers): ปกติ node หนึ่ง
จะเก็บ "ที่อยู่" (pointer/reference) ของ node ก่อนหน้าหรือถัดไป เพื่อให้ traverse ไปมาได้ blockchain ก็ทำแบบเดียวกัน
ทุกประการ ต่างกันแค่ตรงที่ "ที่อยู่" ที่ block ถัดไปเก็บไว้**ไม่ใช่ memory address** แต่เป็น **hash ของเนื้อหาทั้งหมด
ของ block ก่อนหน้า** — และความต่างจุดนี้แหละที่ทำให้มันมีคุณสมบัติพิเศษที่ linked list ธรรมดาไม่มี

คุณสมบัตินั้นเรียกว่า **tamper-evidence** (การตรวจจับการปลอมแปลงได้) ลองไล่ตรรกะดูทีละขั้น:

1. Hash function แบบ cryptographic (เช่น SHA-256 ที่บทนี้จะใช้จริง) มีคุณสมบัติว่า **input เปลี่ยนแม้แค่ 1 bit
   output จะเปลี่ยนไปแบบคาดเดาไม่ได้เลยโดยสิ้นเชิง** (เรียกว่า avalanche effect) และในทางปฏิบัติ **หา input สอง
   ตัวที่ให้ hash เดียวกันไม่ได้** (collision resistance)
2. ถ้า block #5 เก็บ hash ของ block #4 ไว้ในตัวเอง (สมมติเรียกว่า `previous_hash`) แล้วมีใครไปแก้ข้อมูลใน block #4
   หลังจากมันถูกสร้างไปแล้ว — hash ของ block #4 ที่คำนวณใหม่จะไม่ตรงกับ `previous_hash` ที่ block #5 เก็บไว้อีก
   ต่อไป
3. นั่นแปลว่าแค่เทียบ `previous_hash` ที่ block #5 เก็บไว้ กับ hash ที่คำนวณสด ๆ จากข้อมูลจริงของ block #4 ก็รู้ได้
   ทันทีว่ามีการแก้ไขเกิดขึ้น **โดยไม่ต้องมีใครมา manual review เนื้อหาเลย**
4. และเพราะ block #5 ก็ถูกอ้างอิงโดย `previous_hash` ของ block #6 อีกที การแก้ไข block #4 (ถ้าจะทำให้ chain
   ดูเหมือนถูกต้อง) จะบังคับให้ต้องคำนวณ hash ของ block #4, #5, #6, ... ใหม่หมดทุก block ที่ตามมา — ยิ่ง block
   อยู่ลึกเข้าไปในอดีตมากเท่าไร การปลอมแปลงมันยิ่งต้องทำงานมากขึ้นเท่านั้น

นี่คือเหตุผลทั้งหมดที่คนพูดถึง blockchain ว่า "แก้ไขประวัติไม่ได้" — ไม่ใช่เพราะมีเวทมนตร์อะไร แต่เพราะ**โครงสร้าง
ข้อมูลถูกออกแบบให้การปลอมแปลงถูกตรวจจับได้ในทางคณิตศาสตร์** ล้วน ๆ

ลองเห็นคุณสมบัติ avalanche effect ที่พูดถึงในข้อ 1 ด้วยโค้ดจริงสั้น ๆ ก่อนไปต่อ — เปลี่ยน input แค่ตัวเลขตัวเดียว
(จาก 100 เป็น 101) ดูว่า hash เปลี่ยนไปมากแค่ไหน:

```rust
use sha2::{Digest, Sha256};

fn sha256_hex(data: &[u8]) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data);
    hex::encode(hasher.finalize())
}

fn main() {
    let a = "Alice pays Bob 100";
    let b = "Alice pays Bob 101"; // เปลี่ยนแค่ตัวเลขตัวสุดท้ายตัวเดียว
    println!("hash A : {}", sha256_hex(a.as_bytes()));
    println!("hash B : {}", sha256_hex(b.as_bytes()));
}
```

ผลลัพธ์จริงจากการรัน:

```
hash A : 15fd38c0a310472112be808157dbe38b6a5173b5a9072c62b362a5da9d6b94bf
hash B : 219e14632b486612ab39cb2c54ee44c561f85269fb81160a1b63b8ae903dabba
```

สอง hash นี้**ไม่มีความคล้ายกันเลยแม้แต่นิดเดียว** ทั้งที่ input ต่างกันแค่ 1 character (`0` เทียบ `1`) — นี่แหละคือ
avalanche effect ที่ทำให้การตรวจจับการแก้ไขข้อมูลทำได้ง่ายมาก: **แค่เทียบ hash สองตัวว่าตรงกันหรือไม่ ก็รู้ได้ทันที
ว่าข้อมูลต้นฉบับตรงกันหรือไม่ โดยไม่ต้องเทียบข้อมูลดิบทีละ byte เลย** และสังเกตด้วยว่า hash ทั้งสองมีความยาว
เท่ากันเป๊ะ (64 ตัวอักษร hex = 256 bit) ไม่ว่า input จะยาวแค่ไหนก็ตาม — คุณสมบัติ **fixed-size output** นี้คือ
เหตุผลที่ block header เก็บ hash ของ block ก่อนหน้าได้แบบขนาดคงที่เสมอ ไม่ว่า block นั้นจะมีข้อมูลมากแค่ไหน

ก่อนไปหัวข้อถัดไป ลองมองภาพรวมของ chain ทั้งเส้นเป็นแผนภาพง่าย ๆ เพื่อให้เห็นว่า field `previous_hash` ผูกทุก
block เข้าด้วยกันเป็นเส้นตรงอย่างไร:

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│ Block #0 (genesis)│      │ Block #1         │      │ Block #2         │
│ data: "..."      │      │ data: "..."      │      │ data: "..."      │
│ previous_hash:   │      │ previous_hash: ──┼─────▶│ previous_hash: ──┼──▶ (block #3 ...)
│   "000...000"    │      │   hash(Block#0)  │      │   hash(Block#1)  │
│ nonce: 12345     │      │ nonce: 67890     │      │ nonce: 24680     │
│ hash: "3f8a..." ─┼──┐   │ hash: "9c1e..." ─┼──┐   │ hash: "0a5b..."  │
└─────────────────┘  │   └─────────────────┘  │   └─────────────────┘
                      └───────────────────────┘
        (hash ของ block #0 ถูกก็อปปี้ไปเป็น previous_hash ของ block #1 ตอนสร้าง — เชื่อมแบบนี้ไปเรื่อย ๆ)
```

จุดที่ต้องเข้าใจให้แม่นจากแผนภาพนี้: ลูกศรที่ชี้จาก `previous_hash` ของ block ขวาไปยัง `hash` ของ block ซ้าย **ไม่ใช่
pointer ในความหมายของ memory address แบบที่ Part 7-8 สอนไว้เรื่อง reference** มันคือ**ค่าตัวเลข (hash) ที่ถูก
copy เข้ามาเก็บไว้ตรง ๆ** ตอนสร้าง block ใหม่ ครั้งเดียว แล้วไม่เปลี่ยนอีก — ถ้า block ซ้ายถูกแก้ไขข้อมูลในอนาคต
`hash` ของมันจะเปลี่ยน แต่ `previous_hash` ที่ block ขวาเก็บไว้แล้ว**ไม่เปลี่ยนตามไปด้วย** ทำให้สองค่านี้ไม่ตรงกัน
อีกต่อไป และนี่คือกลไกการตรวจจับการปลอมแปลงที่อธิบายไว้เป็นข้อ ๆ ด้านบนนั่นเอง

สิ่งที่ยังไม่ได้พูดถึงในหัวข้อนี้ (และมักถูกปนกันจนสับสน) คือ **consensus mechanism** (วิธีที่ node หลายตัวในเครือ
ข่ายตกลงกันว่า block ไหน "ถูกต้อง" เมื่อมีคนพยายามเสนอ block ที่ขัดแย้งกันเข้ามาพร้อมกัน) เช่น Proof of Work
(หัวข้อ 104.3), Proof of Stake, หรือ mechanism อื่น ๆ — consensus เป็นเรื่องของ**เครือข่ายแบบ distributed** ส่วน
โครงสร้างข้อมูล block ที่เชื่อมกันด้วย hash เป็นเรื่องของ**ข้อมูลตัวเดียวในเครื่องเดียว** บทนี้จะสร้างโครงสร้าง
ข้อมูลให้เห็นจริงก่อน แล้วค่อยอธิบาย Proof of Work ในฐานะตัวอย่างของ consensus mechanism ที่ใช้กันจริง แต่จะไม่ลง
รายละเอียดเรื่อง distributed network protocol เพราะนั่นเป็นเรื่องใหญ่อีกเรื่องที่เกินขอบเขตบทเดียว

### 104.2 สร้าง Block และ Blockchain ตัวจริงด้วย Rust

พอเข้าใจแนวคิดแล้ว มาสร้างโครงสร้างข้อมูลนี้ขึ้นจริงด้วย Rust กัน โดยจะ hash ด้วย SHA-256 จริงผ่าน crate `sha2` —
ไม่ใช่ placeholder function ที่แค่คืนค่า string คงที่

เริ่มจากสร้างโปรเจกต์และเพิ่ม dependency (เวอร์ชันที่ตรวจสอบแล้วจริงในบทนี้ ณ เวลาที่เขียน — ให้ตรวจสอบเวอร์ชันล่าสุด
ด้วย `cargo add` เสมอ เพราะ crate เหล่านี้อัปเดตบ่อย):

```toml
# Cargo.toml
[dependencies]
sha2 = "0.11.0"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
hex = "0.4.3"
```

`sha2` คือ crate ที่ implement ตระกูล hash function SHA-2 (SHA-256, SHA-512 ฯลฯ) ตาม trait `Digest` จาก crate
`digest` ที่เป็นมาตรฐานกลางของ ecosystem crypto ใน Rust — เหตุผลที่เลือก SHA-256 เพราะมันคือ hash function ตัวเดียว
กันกับที่ Bitcoin ใช้จริง เป็นตัวเลือกที่ทั้งเร็วและผ่านการตรวจสอบด้านความปลอดภัยมาอย่างยาวนานในโลกจริง `hex` ใช้
แปลง byte array ที่ได้จาก hash ให้เป็น string ที่อ่านง่าย (เช่น `"a1b2c3..."`) เพราะ hash ดิบ ๆ คือ `[u8; 32]` ซึ่ง
พิมพ์ตรง ๆ ไม่มีประโยชน์กับคนอ่าน

```rust
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

// Block เดียวในโครงสร้าง blockchain
// เก็บ hash ของ block ก่อนหน้าไว้ (previous_hash) — นี่คือ "ตัวเชื่อม" ของ linked list แบบพิเศษที่คุยกันไปในหัวข้อ 104.1
#[derive(Debug, Clone, Serialize, Deserialize)]
struct Block {
    index: u64,
    timestamp: u64,
    data: String,
    previous_hash: String,
    nonce: u64,   // ใช้ในหัวข้อ 104.3 (Proof of Work) — ตอนนี้ยังไม่ต้องสนใจ ปล่อยเป็น 0 ไปก่อน
    hash: String,
}

// struct แยกสำหรับ "เนื้อหาที่จะเอาไป hash" โดยเฉพาะ — ไม่รวม field `hash` ของ Block เข้ามาด้วย
// (เหตุผลสำคัญมาก: ถ้าเผลอเอา field `hash` เข้าไป serialize ด้วย จะเกิดปัญหา circular — hash ของตัวเองต้องรู้ค่า
// hash ของตัวเองก่อนถึงจะคำนวณได้ ซึ่งเป็นไปไม่ได้ ต้องแยก struct ออกมาให้ชัดว่า field ไหนเข้าไปในการคำนวณ hash
// บ้าง)
#[derive(Serialize)]
struct BlockContentForHashing<'a> {
    index: u64,
    timestamp: u64,
    data: &'a str,
    previous_hash: &'a str,
    nonce: u64,
}

impl Block {
    fn new(index: u64, timestamp: u64, data: String, previous_hash: String) -> Self {
        let mut block = Block {
            index,
            timestamp,
            data,
            previous_hash,
            nonce: 0,
            hash: String::new(), // ยังไม่รู้ hash ตอนสร้าง ต้องคำนวณทีหลัง
        };
        block.hash = block.calculate_hash();
        block
    }

    // ขั้นตอนคำนวณ hash: serialize เนื้อหาของ block เป็น JSON string ก่อน (ตามที่ Part 57 สอนไว้)
    // แล้วนำ byte ของ string นั้นเข้า SHA-256 คืนค่าเป็น hex string
    fn calculate_hash(&self) -> String {
        let content = BlockContentForHashing {
            index: self.index,
            timestamp: self.timestamp,
            data: &self.data,
            previous_hash: &self.previous_hash,
            nonce: self.nonce,
        };
        let serialized =
            serde_json::to_string(&content).expect("serialize block content ต้องไม่ล้มเหลว");

        let mut hasher = Sha256::new();
        hasher.update(serialized.as_bytes());
        let result = hasher.finalize(); // ได้ [u8; 32] (256 bit)
        hex::encode(result) // แปลงเป็น hex string ยาว 64 ตัวอักษร
    }
}

struct Blockchain {
    blocks: Vec<Block>,
}

impl Blockchain {
    // genesis block คือ block แรกของ chain — ไม่มี block ก่อนหน้าจริง จึงใช้ previous_hash เป็นเลข 0 ล้วน
    // (แทน "ไม่มีที่อยู่ก่อนหน้า" — เทียบได้กับ head ของ linked list ที่ prev เป็น None แต่ในที่นี้ต้องเป็น
    // string คงที่ เพราะ field เป็น String ไม่ใช่ Option)
    fn new() -> Self {
        let genesis = Block::new(0, 0, "genesis block".to_string(), "0".repeat(64));
        Blockchain {
            blocks: vec![genesis],
        }
    }

    fn add_block(&mut self, data: String) {
        let previous = self.blocks.last().expect("ต้องมีอย่างน้อย genesis block");
        let block = Block::new(
            previous.index + 1,
            previous.timestamp + 1,
            data,
            previous.hash.clone(), // <-- นี่คือจุดที่เชื่อม block เข้าด้วยกัน
        );
        self.blocks.push(block);
    }

    // ตรวจสอบความถูกต้องทั้ง chain: ไล่ตรวจทุก block ว่า (1) previous_hash ตรงกับ hash ของ block ก่อนหน้าจริง
    // และ (2) hash ที่เก็บไว้ตรงกับ hash ที่คำนวณสด ๆ จากข้อมูลปัจจุบันจริง
    fn is_valid(&self) -> bool {
        for i in 1..self.blocks.len() {
            let current = &self.blocks[i];
            let previous = &self.blocks[i - 1];
            if current.previous_hash != previous.hash {
                return false; // การเชื่อมโยงขาด — มีคนแก้ previous_hash หรือ hash ของ block ก่อนหน้าไม่ตรง
            }
            if current.hash != current.calculate_hash() {
                return false; // เนื้อหาของ block นี้เองถูกแก้ไขไปแล้วหลังสร้าง
            }
        }
        true
    }
}

fn main() {
    let mut chain = Blockchain::new();
    println!("Genesis hash: {}", chain.blocks[0].hash);

    chain.add_block("Alice ส่ง 10 หน่วยให้ Bob".to_string());
    chain.add_block("Bob ส่ง 5 หน่วยให้ Carol".to_string());

    println!("Chain มี {} blocks, valid = {}", chain.blocks.len(), chain.is_valid());

    // จำลองการปลอมแปลง: แก้ไขข้อมูลของ block #1 ตรง ๆ โดยไม่คำนวณ hash ใหม่
    chain.blocks[1].data = "ข้อมูลที่ถูกแก้ไขโดยไม่ได้รับอนุญาต".to_string();
    println!("หลังแก้ไข block #1: valid = {}", chain.is_valid());
}
```

โค้ดนี้**คอมไพล์และรันได้จริง** (ผลลัพธ์จริงจากการรันโค้ดชุดนี้ในหัวข้อ 104.3 ที่ขยายเพิ่ม Proof of Work เข้าไปด้วย
จะแสดงให้เห็นด้านล่าง) จุดที่ควรสังเกตให้ดี:

- **เราแยก `BlockContentForHashing` ออกจาก `Block`** เพราะ field `hash` ของ `Block` เองต้องไม่ถูกรวมเข้าไปใน
  ข้อมูลที่ใช้คำนวณ hash — ถ้าเผลอ derive `Serialize` ให้ `Block` ทั้งตัวแล้วเอาไป hash ตรง ๆ จะเกิดปัญหา circular
  dependency ทางความหมาย (hash ต้องรู้ค่า hash ของตัวเองก่อนจะคำนวณ hash ได้ ซึ่งเป็นไปไม่ได้ทางตรรกะ) นี่คือรายละ
  เอียดเล็ก ๆ ที่คนเขียน blockchain เดโมมือใหม่มักพลาด
- **`add_block` มีจุดเชื่อมสำคัญหนึ่งบรรทัด**: `previous.hash.clone()` ถูกส่งเข้าไปเป็น `previous_hash` ของ block
  ใหม่ นี่คือบรรทัดเดียวที่ทำให้ทั้งโครงสร้างกลายเป็น "chain" จริง ๆ ถ้าลบบรรทัดนี้ออกไป (เช่น ใส่ string คงที่
  แทน) มันจะกลายเป็นแค่ list ของ block ที่ไม่เกี่ยวข้องกันเลย ไม่มีคุณสมบัติ tamper-evidence อะไรทั้งสิ้น
- `is_valid()` ตรวจสอบสองเงื่อนไขที่แยกจากกันแต่สำคัญทั้งคู่: เงื่อนไขแรกตรวจ**การเชื่อมโยงระหว่าง block**
  เงื่อนไขที่สองตรวจ**ความถูกต้องภายในของแต่ละ block เอง** ทั้งสองต้องผ่านพร้อมกันเสมอ

### 104.3 Proof of Work: nonce, difficulty, และการวัดเวลาจริง

ในโค้ดหัวข้อที่แล้ว `nonce` ถูกตั้งเป็น `0` เฉย ๆ ไม่ได้ใช้ประโยชน์อะไร หัวข้อนี้จะอธิบายว่ามันมีไว้ทำอะไรกันแน่ ผ่าน
กลไกที่เรียกว่า **Proof of Work (PoW)**

ปัญหาที่ PoW แก้คือ: ในเครือข่ายแบบ distributed ที่ไม่มีศูนย์กลาง ใครก็ตามที่เสนอ block ใหม่เข้าเครือข่ายได้ทันที
โดยไม่มีต้นทุนอะไรเลย — ระบบจะเสี่ยงต่อการถูก spam หรือถูกโจมตีด้วยการสร้าง chain ปลอมจำนวนมากอย่างรวดเร็ว PoW แก้
ปัญหานี้ด้วยการ**บังคับให้การสร้าง block ใหม่ต้องใช้"งานคำนวณ"จริง**ก่อนจะถูกเครือข่ายยอมรับ — งานคำนวณนี้ต้องยาก
พอที่จะมีต้นทุน (เวลา/ไฟฟ้า) แต่ต้อง**ตรวจสอบได้เร็วมาก**ว่าใครทำสำเร็จจริง

กลไกที่ใช้กันจริงคือ: **บังคับให้ hash ของ block ต้องขึ้นต้นด้วยเลข 0 จำนวนหนึ่ง** (เรียกจำนวนนี้ว่า "difficulty")
เนื่องจาก SHA-256 มีคุณสมบัติ avalanche effect ตามที่คุยไว้ในหัวข้อ 104.1 — **ไม่มีทางคำนวณย้อนกลับได้ว่าต้องใส่
input อะไรเข้าไปถึงจะได้ hash ที่ขึ้นต้นด้วย 0 ตามต้องการ** วิธีเดียวที่ทำได้คือ**ลองไปเรื่อย ๆ** โดยเปลี่ยนค่าบาง
อย่างในข้อมูล แล้วคำนวณ hash ใหม่ ทำซ้ำจนกว่าจะเจอ — และนี่คือหน้าที่ของ `nonce`: มันเป็น field ที่**ไม่มีความหมาย
ทางธุรกิจอะไรเลย** มีไว้แค่ให้ปรับค่าไปเรื่อย ๆ เพื่อเปลี่ยน hash ที่ได้ โดยไม่ต้องไปแก้ข้อมูล transaction จริง

มาต่อโค้ดจากหัวข้อ 104.2 เพิ่มฟังก์ชัน mining:

```rust
use std::time::Instant;

impl Block {
    // ขุด block: ลองปรับ nonce ไปเรื่อย ๆ จนกว่า hash ที่ได้จะขึ้นต้นด้วยเลข 0 จำนวน `difficulty` ตัว
    // คืนค่าเวลาที่ใช้ (microsecond) เพื่อเอาไปวัดผลจริงตามแนวทาง Part 54
    fn mine(&mut self, difficulty: usize) -> u128 {
        let target_prefix = "0".repeat(difficulty);
        let start = Instant::now();
        loop {
            self.hash = self.calculate_hash();
            if self.hash.starts_with(&target_prefix) {
                break;
            }
            self.nonce += 1; // ปรับค่าแล้วลองใหม่ — นี่คือ "งาน" ที่ PoW บังคับให้ทำ
        }
        start.elapsed().as_micros()
    }
}
```

และเปลี่ยน `add_block` ให้เรียก `mine` แทนการปล่อย `nonce = 0` ไว้เฉย ๆ:

```rust
impl Blockchain {
    fn add_block(&mut self, data: String, difficulty: usize) {
        let previous = self.blocks.last().expect("ต้องมีอย่างน้อย genesis block");
        let mut block = Block::new(
            previous.index + 1,
            previous.timestamp + 1,
            data,
            previous.hash.clone(),
        );
        let micros = block.mine(difficulty);
        println!(
            "  ขุด block #{} สำเร็จ: nonce={}, hash={}, ใช้เวลา {} micro seconds",
            block.index, block.nonce, block.hash, micros
        );
        self.blocks.push(block);
    }
}
```

ทดสอบวัดเวลาจริงที่ difficulty แตกต่างกันสามระดับ:

```rust
fn main() {
    let mut chain = Blockchain::new();
    for (i, difficulty) in [2usize, 3, 4].into_iter().enumerate() {
        println!(
            "กำลังขุด block #{} ด้วย difficulty={}",
            i + 1,
            difficulty
        );
        chain.add_block(format!("transaction batch #{i}"), difficulty);
    }
}
```

นี่คือ**ผลลัพธ์จริงจากการรันโค้ดชุดนี้** (วัดในสภาพแวดล้อมตรวจสอบของบทนี้ — ตัวเลขที่แน่นอนจะต่างกันได้ในการรันครั้ง
อื่นเพราะ Proof of Work เป็นการค้นหาแบบสุ่มโดยธรรมชาติ แต่**แนวโน้ม**ของการเติบโตจะเห็นชัดเสมอ):

```
กำลังขุด block #1 ด้วย difficulty=2 (ต้องการเลขนำหน้า hash เป็นเลข 0 จำนวน 2 ตัว)
  ขุด block #1 สำเร็จ: nonce=340, hash=00acdf7650cb88f6556b29b34f685e3650c2c3e792c5f12d6292ef3336e2248a, ใช้เวลา 177 micro seconds

กำลังขุด block #2 ด้วย difficulty=3 (ต้องการเลขนำหน้า hash เป็นเลข 0 จำนวน 3 ตัว)
  ขุด block #2 สำเร็จ: nonce=3575, hash=0005ae8c5e0d13b2979e1428bf43fa524b8d3880fda82a92c490d88ca317050f, ใช้เวลา 1892 micro seconds

กำลังขุด block #3 ด้วย difficulty=4 (ต้องการเลขนำหน้า hash เป็นเลข 0 จำนวน 4 ตัว)
  ขุด block #3 สำเร็จ: nonce=49313, hash=0000993cf043b3bdbc96885c94e4fcc8ab8d06ef01fad22151aa0ddff0ec2e57, ใช้เวลา 26296 micro seconds
```

ตัวเลขที่ควรอ่านให้เป็น: **nonce ที่ต้องลองก่อนเจอ hash ที่ผ่านเงื่อนไข**: 340 → 3,575 → 49,313 และ**เวลาที่ใช้**:
177 → 1,892 → 26,296 microsecond การขยับ difficulty ทีละ 1 (คือเพิ่มเงื่อนไข "เลข 0 นำหน้า" อีก 1 หลัก hex) ทำให้
งานที่ต้องทำเพิ่มขึ้นประมาณ **10-14 เท่า** ในแต่ละครั้ง ซึ่งใกล้เคียงกับค่าทางทฤษฎีที่ควรจะเป็น **16 เท่า** (เพราะแต่
ละหลัก hex คือ 4 bit และ probability ที่จะสุ่มได้ hash ที่ขึ้นต้นด้วยเลข 0 เพิ่มอีกหนึ่งหลักคือ 1/16 ของโอกาสเดิม)
ความต่างเล็กน้อยจากค่าทางทฤษฎีเกิดจากการที่นี่เป็นการรันครั้งเดียว (sample size เล็ก) ซึ่งมี randomness เจือปนอยู่
เยอะ — **นี่คือเหตุผลที่ต้องวัดจริงเสมอ**ตามที่ Part 54 สอนไว้ ไม่ใช่แค่คำนวณทางทฤษฎีแล้วเชื่อไปเลย เพราะการรัน
จริงมี noise ที่ตัวเลขทฤษฎีล้วน ๆ ไม่บอกให้เรารู้

ข้อสังเกตสำคัญอีกจุดคือ: **การเติบโตแบบ exponential นี้คือกลไกที่ทำให้ Proof of Work มีความหมายจริงในเชิงความ
ปลอดภัย** ถ้าใครต้องการปลอมแปลง block เก่าใน chain ที่มี difficulty สูง (ตามที่คุยไว้ในหัวข้อ 104.1 ว่าต้อง
คำนวณ hash ของทุก block ที่ตามมาใหม่ทั้งหมด) เขาไม่ได้แค่ต้องคำนวณ hash ใหม่เฉย ๆ แต่ต้อง**ขุดหา nonce ที่ถูกต้อง
ใหม่ทุก block** ด้วย ซึ่งที่ difficulty สูง ๆ ในโลกจริง (เช่น Bitcoin) การทำแบบนี้ต้องใช้พลังคำนวณมากกว่าที่เครือข่าย
ทั้งหมดในโลกมีรวมกันในเวลาที่สมเหตุสมผล — นี่คือที่มาของคำว่า "ปลอมแปลงได้ยากในทางเศรษฐศาสตร์/คำนวณ" ไม่ใช่
"ปลอมแปลงไม่ได้ในทางทฤษฎี"

มีอีกประเด็นเชิงวิศวกรรมที่สำคัญมากที่ควรรู้จักไว้ (แม้บทนี้จะไม่ implement เพราะเกินขอบเขตของตัวอย่างเดียว):
เครือข่ายจริงที่ใช้ PoW (เช่น Bitcoin) **ไม่ได้ตั้ง difficulty ไว้คงที่ตลอดไป** — มันมีกลไกที่เรียกว่า
**difficulty retargeting/adjustment** ที่ปรับ difficulty ขึ้นหรือลงเป็นระยะ ๆ (เช่น ทุก ๆ จำนวน block ที่กำหนดไว้
ตายตัว) โดยอิงจาก**เวลาจริงที่ผ่านมาในการขุด block ก่อนหน้า** เทียบกับเวลาที่ต้องการให้เฉลี่ยออกมา — ถ้า node ทั้ง
เครือข่ายรวมกันขุดเร็วกว่าที่ตั้งเป้าไว้ (เพราะมี hardware เข้าร่วมมากขึ้น) difficulty จะถูกปรับขึ้นโดยอัตโนมัติ
เพื่อให้เวลาเฉลี่ยต่อ block กลับมาใกล้เคียงค่าที่ตั้งเป้าไว้เดิม และในทางกลับกันถ้าขุดช้าลง difficulty จะถูกปรับลง
— กลไกนี้คือตัวอย่างที่ดีของ "การวัดจริงแล้วปรับพฤติกรรมของระบบตามข้อมูลจริง" ในสเกลที่ใหญ่กว่าตัวอย่างการวัดเวลา
มือ ๆ ในหัวข้อนี้มาก แต่**หลักการเดียวกันเป๊ะ**: วัดผลจริงจากการทำงานจริง แล้วใช้ตัวเลขนั้นตัดสินใจ ไม่ใช่เดาเอา
เองว่าเครือข่ายจะขุดเร็วหรือช้าแค่ไหน

### 104.4 ทำไม Rust ถึงเหมาะกับงาน Blockchain/Crypto Infrastructure เป็นพิเศษ

ก่อนไปต่อเรื่อง digital signature และ smart contract ควรหยุดตั้งคำถามที่มักถูกมองข้าม: **ทำไมโปรเจกต์ blockchain
infrastructure จริงจำนวนมากในโลกถึงเลือกเขียนด้วย Rust?** คำตอบมีเหตุผลเชิงเทคนิคที่จับต้องได้อยู่สองข้อหลัก ไม่ใช่
แค่ความนิยม

**ข้อที่หนึ่ง: performance ของงานที่ต้องทำซ้ำในสเกลใหญ่มาก** validator node ของเครือข่าย blockchain สาย
production ต้อง**ตรวจสอบ signature และคำนวณ hash เป็นจำนวนมหาศาลต่อวินาที** เครือข่ายที่มี throughput สูง (เช่น
Solana ที่โฆษณาความสามารถประมวลผลหลายพัน transaction ต่อวินาที) ทำแบบนั้นได้เพราะโค้ดที่ตรวจสอบ signature/hash
ทำงานเร็วในระดับเทียบเท่า C/C++ ไม่มี garbage collector มาหยุดโปรแกรมกลางคันแบบที่ภาษาที่มี GC เจอ (ประเด็นเดียว
กับที่ Part 86 อธิบายไว้เรื่อง WASM binary ที่ไม่มี GC runtime แนบมาด้วย) — ที่สเกลระดับพันหรือหมื่น
transaction ต่อวินาที ความต่างของ overhead แม้เพียงเล็กน้อยต่อ operation ก็ขยายเป็นความต่างมหาศาลในภาพรวม

**ข้อที่สอง (สำคัญกว่าข้อแรกมาก): memory safety ในโค้ดที่จัดการมูลค่าทางการเงินโดยตรง** นี่คือจุดที่ Rust ให้คุณค่า
สูงที่สุดในบริบทนี้ ลองนึกถึง Part 42 (Raw Pointers) และ Part 100 (Security) ที่สอนไว้ว่า buffer overflow,
use-after-free, double-free คือ bug class ที่เกิดจาก memory ที่ไม่ปลอดภัย ในโปรแกรมทั่วไป bug แบบนี้แปลว่าโปรแกรม
crash หรือมีช่องโหว่ให้แฮก แต่ใน**โค้ดที่ควบคุมยอดเงินของผู้ใช้นับล้านคนโดยตรง** bug ประเภทเดียวกันนี้อาจแปลว่า
**เงินถูกสร้างขึ้นมาจากอากาศ (double-spend bug), เงินหายไปจริง, หรือมีคนขโมยเงินคนอื่นได้** — ผลกระทบทางการเงิน
โดยตรงและ**แก้ไขย้อนหลังไม่ได้**เมื่อ transaction ถูกบันทึกลง chain ที่ immutable แล้ว (ตามคุณสมบัติที่คุยไว้ใน
104.1) ทำให้ "compiler ปฏิเสธไม่ให้ compile ถ้ามี memory bug ที่ตรวจจับได้แต่ compile time" มีมูลค่าสูงกว่าปกติมาก
ในบริบทนี้ — Rust ไม่ได้ทำให้ smart contract ปลอดภัยจาก**ทุกอย่าง** (หัวข้อ 104.11 จะพูดเรื่องนี้ตรง ๆ) แต่มัน
กำจัด bug class หนึ่งกลุ่มที่สำคัญออกไปได้ตั้งแต่ก่อนโค้ดจะรันด้วยซ้ำ

ในทางปฏิบัติ นี่ไม่ใช่แค่ทฤษฎี — โปรเจกต์ blockchain infrastructure ระดับ production จริงหลายตัวเลือก Rust ด้วย
เหตุผลนี้ตรง ๆ:

| โปรเจกต์ | ใช้ Rust ตรงไหน |
|---|---|
| **Solana** | Validator client (ตัวโปรแกรมหลักที่รันเครือข่าย) และ on-chain program (สิ่งที่เทียบเท่า smart contract) เขียนด้วย Rust ทั้งคู่ — ต่างจาก Ethereum ที่ on-chain logic เขียนด้วยภาษาเฉพาะทาง Solidity |
| **Polkadot / Substrate** | Framework `Substrate` สำหรับสร้าง blockchain แบบกำหนดเองทั้งตัว (custom blockchain framework) เขียนด้วย Rust ทั้งหมด รวมถึง runtime logic ของ chain ที่สร้างด้วย Substrate |
| **CosmWasm** (ecosystem Cosmos) | Smart contract เขียนด้วย Rust compile เป็น WASM — หัวข้อ 104.8-104.9 ของบทนี้จะลงมือทำจริงกับตัวนี้ |
| **Near Protocol** | Smart contract รองรับ Rust เป็นภาษาหลักตัวหนึ่ง compile เป็น WASM เช่นกัน |

ตารางนี้คือ**บริบทอุตสาหกรรมตามข้อเท็จจริง** ไม่ใช่การชักชวนให้ลงทุนหรือประเมินมูลค่าของเครือข่ายเหล่านี้แต่อย่างใด
— ประเด็นที่ต้องการสื่อคือ**เหตุผลเชิงวิศวกรรม**ที่ทำให้ทีมวิศวกรที่สร้างระบบเหล่านี้เลือก Rust ไม่ใช่ภาษาอื่น และ
ทักษะ Rust ที่คุณเรียนมาตลอดหลักสูตรนี้ (ownership, memory safety, performance โดยไม่มี GC) **ถูกใช้งานจริงตรงจุด
นี้** ไม่ใช่ทักษะที่ต้องเรียนเพิ่มจากศูนย์

ตัวอย่างที่จับต้องได้มากขึ้นของ "performance ที่ compile time safety ช่วยได้จริง" คือสถาปัตยกรรม runtime ของ
Solana ที่เรียกว่า **Sealevel** — จุดเด่นของมันคือการรัน transaction ที่ไม่แก้ไข account ที่ทับซ้อนกัน**แบบ
parallel กันจริงบนหลาย CPU core** (ต่างจากเครือข่ายส่วนใหญ่ที่ประมวลผล transaction แบบ sequential ทีละรายการ)
การจะทำแบบนี้ได้อย่างปลอดภัย ระบบต้องรู้แน่ชัดว่า transaction ไหน "แก้ไขข้อมูลชุดเดียวกัน" กับอันไหนบ้าง (เพื่อ
ไม่ให้รันพร้อมกันแล้วเกิด data race ในความหมายเดียวกับที่ Part 39-40 สอนไว้เรื่อง `Mutex`/`Arc`/`Send`/`Sync`) —
Solana บังคับให้ instruction ทุกตัวประกาศ**ล่วงหน้า**ว่าจะอ่าน/เขียน account ตัวไหนบ้าง ทำให้ runtime ตรวจสอบได้
ก่อนรันจริงว่า transaction กลุ่มไหนขนานกันได้อย่างปลอดภัย — นี่คือแนวคิด "ประกาศสิ่งที่จะแก้ไขล่วงหน้าให้ตรวจสอบ
ได้" ที่คล้ายกับหลักการ ownership/borrowing ของ Rust เอง (รู้ล่วงหน้าว่าใครถือ mutable reference ของอะไรอยู่)
แม้จะเป็นกลไกระดับ runtime ของ blockchain ไม่ใช่ compile-time ของภาษาก็ตาม

ในฝั่ง Substrate นอกจากการที่ runtime ทั้งเส้นเขียนด้วย Rust แล้ว มันยังใช้ WASM เป็นรูปแบบสำหรับ**เก็บ runtime
logic ของ chain เอง**ไว้ในตัว blockchain โดยตรง (ไม่ใช่แค่ contract ที่ deploy บน chain แบบ CosmWasm) ทำให้
เครือข่ายที่สร้างด้วย Substrate สามารถ**อัปเกรด logic การทำงานของทั้ง chain ได้ผ่าน on-chain governance** โดยไม่
ต้องบังคับให้ทุก node หยุดแล้วอัปเดต binary กันเองแบบ manual — เป็นตัวอย่างที่ดีอีกอันของ WASM ที่ถูกใช้ประโยชน์จาก
คุณสมบัติ portable/sandboxed ตามที่ Part 86 สอนไว้ ในบริบทที่ต่างจาก CosmWasm แต่หลักการเดียวกัน

### 104.5 Digital Signature: พิสูจน์ตัวตนโดยไม่มีศูนย์กลาง (ed25519-dalek)

กลับมาที่หัวใจของ transaction ในระบบแบบ decentralized: **ถ้าไม่มีธนาคารกลางหรือใครมายืนยันว่า "Alice อนุมัติการโอน
เงินนี้จริง" แล้วเครือข่ายจะเชื่อได้อย่างไรว่า transaction นั้นถูกต้อง?**

คำตอบคือ **digital signature** — กลไกที่คุณเคยเห็นมาแล้วจริงในบทนี้ Part 74 ตอนเรียน JWT แบบ RS256 หลักการเหมือน
กันทุกประการ:

1. ทุกคนมี**คู่กุญแจ** (key pair): **private key** (เก็บเป็นความลับ ไม่มีใครรู้นอกจากเจ้าของ) และ **public key**
   (เผยแพร่ให้ทุกคนรู้ได้อย่างเปิดเผย)
2. เมื่อจะทำ transaction เจ้าของใช้ **private key เซ็นชื่อ (sign)** ข้อมูล transaction — ได้ signature ออกมาเป็น
   ชุดข้อมูลผูกกับทั้ง private key และเนื้อหา transaction นั้น
3. ใครก็ตามที่มี **public key** ของเจ้าของ (ซึ่งเผยแพร่เปิดเผยได้ ไม่มีความเสี่ยง) สามารถ**ตรวจสอบ (verify)**
   ได้ว่า signature นี้ถูกสร้างโดย private key ที่จับคู่กับ public key นี้จริง **โดยไม่ต้องรู้ private key เลย**
4. ถ้าข้อมูล transaction ถูกแก้ไขแม้เพียงนิดเดียวหลังเซ็นแล้ว การ verify จะ**ล้มเหลวทันที** เพราะ signature ผูกกับ
   เนื้อหาที่เซ็น ณ ขณะนั้นเป๊ะ

Part 74 หัวข้อ RS256 อธิบายกลไกนี้ด้วยคณิตศาสตร์ตระกูล RSA ส่วนบทนี้จะใช้คณิตศาสตร์ตระกูล **elliptic curve
cryptography** ผ่าน algorithm ชื่อ **Ed25519** ผ่าน crate `ed25519-dalek` ซึ่งเป็น crate ที่ได้รับการยอมรับและใช้
งานจริงกว้างขวางใน ecosystem crypto ของ Rust (ตระกูล `*-dalek` เป็นชุด crate cryptography ที่ community เชื่อถือสูง)
— **หลักการ public/private key เหมือนกับ RS256 เป๊ะ** ต่างกันแค่คณิตศาสตร์เบื้องหลังและขนาดของ key/signature
(Ed25519 ให้ key และ signature ที่เล็กกว่า RSA มาก ด้วยระดับความปลอดภัยที่เทียบเคียงได้ — เหตุผลหนึ่งที่ระบบ
blockchain จำนวนมากเลือกใช้มันแทน RSA)

เพิ่ม dependency:

```toml
[dependencies]
ed25519-dalek = { version = "3.0.0", features = ["rand_core"] }
rand = "0.10.3"
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
hex = "0.4.3"
```

หมายเหตุสำคัญเรื่อง feature flag: `ed25519-dalek` เวอร์ชัน 3.0 ไม่เปิด feature `rand_core` มาให้โดย default —
ถ้าต้องการสร้าง key คู่ใหม่แบบสุ่มด้วย `SigningKey::generate` ต้องเปิด feature นี้ตรง ๆ (กับดักที่พบบ่อยข้อนี้จะพูด
ถึงอีกครั้งในหัวข้อกับดักท้ายบท)

```rust
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Transaction {
    from: String, // public key ของผู้ส่ง ในรูป hex string
    to: String,
    amount: u64,
    nonce: u64, // ใช้ป้องกัน replay attack (เซ็น transaction เดิมส่งซ้ำ) — ไม่เกี่ยวกับ nonce ของ PoW ในหัวข้อ 104.3
}

fn main() {
    // ขั้นที่ 1: สร้างคู่กุญแจ (ในระบบจริง private key ต้องถูกเก็บอย่างปลอดภัยมาก ไม่ใช่สร้างทิ้งในโค้ดตัวอย่างแบบนี้)
    let mut rng = rand::rng();
    let signing_key = SigningKey::generate(&mut rng); // private key
    let verifying_key: VerifyingKey = signing_key.verifying_key(); // public key ที่จับคู่กัน

    // ขั้นที่ 2: สร้าง transaction แล้ว serialize เป็น byte เพื่อเซ็น (ตามหลักการ serde ที่ Part 57 สอนไว้)
    let tx = Transaction {
        from: hex::encode(verifying_key.to_bytes()),
        to: "recipient_public_key_placeholder".to_string(),
        amount: 250,
        nonce: 1,
    };
    let tx_bytes = serde_json::to_vec(&tx).expect("serialize transaction ต้องไม่ล้มเหลว");

    // ขั้นที่ 3: เซ็น transaction ด้วย private key
    let signature: Signature = signing_key.sign(&tx_bytes);
    println!("Signature (hex): {}", hex::encode(signature.to_bytes()));

    // ขั้นที่ 4: ใครก็ตามที่มี public key ตรวจสอบได้ — ไม่ต้องมี private key เลย
    let verify_result = verifying_key.verify(&tx_bytes, &signature);
    println!("ตรวจสอบลายเซ็นด้วย public key จริง: {:?}", verify_result.is_ok());
}
```

รันจริงได้ผลลัพธ์แบบนี้ (public key/signature เป็นค่าสุ่มใหม่ทุกครั้งที่รัน เพราะสร้างจาก `rand::rng()` แต่โครง
สร้างผลลัพธ์เหมือนกันเสมอ):

```
Signature (hex): 07b578397aa42fe534f459d5a800c219f4dfb18cf80f49fece4f76b987cd8b028ec3fbf93bc600c588b248da99480dd7c2f8c29f94a6f071efd1a9c8524ac800
ตรวจสอบลายเซ็นด้วย public key จริง: true
```

ที่น่าสนใจกว่าคือการพิสูจน์ว่ากลไกนี้**ตรวจจับการปลอมแปลงได้จริง** ลองสองสถานการณ์ต่อ:

```rust
    // สถานการณ์ที่ 1: มีคนแก้ไข amount ของ transaction หลังจากเซ็นไปแล้ว (พยายามขโมยเงินเพิ่ม)
    let mut tampered_tx = tx;
    tampered_tx.amount = 999_999;
    let tampered_bytes = serde_json::to_vec(&tampered_tx).unwrap();
    let verify_tampered = verifying_key.verify(&tampered_bytes, &signature);
    println!(
        "ตรวจสอบลายเซ็นเดิมกับ transaction ที่ถูกแก้ไข amount: {:?}",
        verify_tampered.is_ok()
    );

    // สถานการณ์ที่ 2: มีคนอื่นพยายามเซ็นแทนโดยไม่มี private key ตัวจริงของ Alice
    let mut attacker_rng = rand::rng();
    let attacker_key = SigningKey::generate(&mut attacker_rng);
    let attacker_signature: Signature = attacker_key.sign(&tx_bytes);
    let verify_wrong_key = verifying_key.verify(&tx_bytes, &attacker_signature);
    println!(
        "ตรวจสอบลายเซ็นที่เซ็นด้วย private key ผิดคน: {:?}",
        verify_wrong_key.is_ok()
    );
```

ผลลัพธ์จริง:

```
ตรวจสอบลายเซ็นเดิมกับ transaction ที่ถูกแก้ไข amount: false
ตรวจสอบลายเซ็นที่เซ็นด้วย private key ผิดคน: false
```

ทั้งสองกรณีล้มเหลวตามที่คาด — และนี่คือหัวใจของการทำงานแบบ decentralized ทั้งระบบ: **เครือข่ายไม่ต้องเชื่อใครเลย
เครือข่ายเชื่อแค่คณิตศาสตร์** node ไหนก็ตามที่ได้รับ transaction พร้อม signature เข้ามา ตรวจสอบได้เองทันทีด้วย
public key ที่แนบมาในตัว transaction ว่า signature ถูกต้องหรือไม่ ไม่ต้องถามใคร ไม่ต้องรอศูนย์กลางอนุมัติ — นี่คือ
สิ่งที่คำว่า "decentralized" หมายถึงในทางเทคนิคจริง ๆ

**เรื่องที่สำคัญไม่แพ้กันแต่มักถูกมองข้าม: การจัดการ private key** ในตัวอย่างข้างบน `signing_key` ถูกสร้างขึ้นมา
ใหม่แบบสุ่มในโค้ดตรง ๆ เพื่อความง่ายในการสาธิต แต่**ในระบบจริงห้ามทำแบบนี้เด็ดขาด** เพราะใครก็ตามที่มี private key
ตัวจริงของคุณ**เซ็น transaction แทนคุณได้ทุกอย่างโดยไม่ต้องขออนุญาต** — ไม่มีทาง "ยกเลิก" transaction ที่เซ็นไป
แล้วได้ด้วย เพราะไม่มีศูนย์กลางให้ขอยกเลิก (ย้อนกลับไปหลักการ decentralized ที่พูดไปเมื่อกี้) ในทางปฏิบัติจริง
private key ของระบบ blockchain มักถูกเก็บด้วยวิธีหนึ่งในสามแนวทางนี้ (เรียงจากพื้นฐานไปถึงปลอดภัยสูง): **encrypted
keystore file** ที่เข้ารหัสด้วย passphrase ก่อนเก็บลง disk (ไม่เก็บ raw key เป็น plaintext เด็ดขาด),
**hardware wallet** (อุปกรณ์เฉพาะที่เก็บ private key ไว้ในชิปที่แยกจากคอมพิวเตอร์หลัก เซ็น transaction ภายในชิป
เอง ไม่ส่ง private key ออกมาให้คอมพิวเตอร์เห็นเลย) และ **HSM (Hardware Security Module)** สำหรับระบบระดับองค์กรที่
ต้องจัดการ key จำนวนมากพร้อม audit log — ทั้งสามแนวทางมีหลักการร่วมกันคือ **private key ไม่ควรอยู่ในรูปแบบที่
โปรแกรมทั่วไปอ่านออกมาเป็น plain byte ได้ตรง ๆ โดยไม่มีการป้องกันชั้นใดชั้นหนึ่งเลย**

อีกจุดที่ควรทำความเข้าใจให้ชัด: `to_bytes()`/`from_bytes()` ของทั้ง `SigningKey`, `VerifyingKey`, และ `Signature`
ทำให้แปลงกลับไปมาระหว่าง struct กับ raw byte ได้ (แล้วค่อย `hex::encode`/`hex::decode` เป็น string ที่เก็บ/ส่งต่อ
ได้ง่าย) ลองพิสูจน์ round-trip นี้ให้เห็นจริง:

```rust
let original_key = signing_key.to_bytes(); // [u8; 32]
let hex_string = hex::encode(original_key);
println!("private key (hex, ห้ามเผยแพร่จริง): {hex_string}");

// จำลองการโหลด key กลับมาจาก storage/config (decode จาก hex string ที่เก็บไว้)
let decoded_bytes = hex::decode(&hex_string).expect("hex ต้อง valid");
let restored_key = SigningKey::from_bytes(
    &decoded_bytes.try_into().expect("ต้องมีความยาว 32 byte เป๊ะ"),
);
assert_eq!(signing_key.to_bytes(), restored_key.to_bytes());
println!("โหลด key กลับมาแล้วตรงกับตัวเดิมทุกประการ");
```

รูปแบบนี้ (`to_bytes` → `hex::encode` → เก็บ/ส่ง → `hex::decode` → `from_bytes`) คือ pattern มาตรฐานที่ระบบจริง
ใช้ในการเก็บและโอนย้าย key ระหว่างระบบ — สิ่งที่ต้องระวังเสมอคือ**ขั้นตอน hex::encode ของ private key ต้องเกิดขึ้น
หลังผ่านการเข้ารหัส (encryption) แล้วเท่านั้น** ไม่ใช่ hex encode ตัว raw private key ตรง ๆ แล้วเก็บลง storage
ธรรมดาแบบในตัวอย่างสาธิตข้างบน (ตัวอย่างข้างบนทำแบบนี้เพื่อความง่ายในการสอนเท่านั้น ไม่ใช่แนวทางที่ปลอดภัยสำหรับ
ระบบจริง)

### 104.6 Merkle Tree: พิสูจน์ข้อมูลในชุดใหญ่โดยไม่ต้องมีข้อมูลทั้งหมด

Block หนึ่งใน blockchain จริง ไม่ได้เก็บ transaction แค่รายการเดียว — มันอาจมีเป็นร้อยหรือพันรายการ ปัญหาที่ตามมา
คือ: **ถ้าอยากพิสูจน์ว่า transaction รายการหนึ่งอยู่ใน block จริง จะต้องดาวน์โหลดข้อมูลทุก transaction ใน block นั้น
มาตรวจสอบเองเลยหรือ?** ในทางปฏิบัติทำแบบนั้นไม่ได้ — block อาจมีขนาดหลาย megabyte และมี node หลายพันตัวในเครือข่าย
ที่ต้องการตรวจสอบพร้อมกัน

**Merkle tree** แก้ปัญหานี้ได้อย่างสวยงาม ด้วยแนวคิดที่ต่อยอดจาก hash function ตัวเดียวกันที่ใช้ทำ Proof of Work:

1. นำ transaction ทุกรายการมา hash ทีละตัว (เรียกว่า leaf hash)
2. จับ leaf hash มาเรียงคู่ ๆ นำมา**ต่อกันแล้ว hash รวมเป็นตัวเดียว** ได้ hash ระดับที่สูงขึ้นมา
3. ทำซ้ำขั้นตอนที่ 2 ไปเรื่อย ๆ จนเหลือ hash ตัวเดียว เรียกว่า **Merkle root**
4. Merkle root ตัวเดียวนี้ (ยาวแค่ 64 ตัวอักษร hex) ถูกเก็บไว้ใน block header — เป็นตัวแทนของ transaction ทั้งหมด
   ใน block นั้น**โดยไม่ต้องเก็บ transaction ทั้งหมดไว้ในตัวมันเอง**

ประโยชน์ที่ตามมาคือ **Merkle proof**: ถ้าอยากพิสูจน์ว่า transaction ตัวหนึ่งอยู่ใน block จริง (โดยรู้แค่ Merkle
root ของ block เท่านั้น) เราไม่ต้องมี transaction ทั้งหมด — ต้องการแค่ **sibling hash ที่อยู่ในเส้นทางจาก leaf นั้น
ขึ้นไปถึง root** (จำนวน sibling ที่ต้องใช้คือ log₂(จำนวน transaction) เท่านั้น — ถ้ามี 1,000 transaction ต้องใช้
sibling แค่ประมาณ 10 ตัว ไม่ใช่ 1,000 ตัว) แล้วคำนวณไล่ hash ขึ้นไปเอง ถ้าผลลัพธ์สุดท้ายตรงกับ Merkle root ที่รู้
อยู่แล้ว ก็พิสูจน์ได้ว่า transaction นั้นอยู่ใน block จริง — **นี่คือเหตุผลที่ light client (โปรแกรมที่ไม่อยาก
ดาวน์โหลด blockchain ทั้งเส้น) ตรวจสอบ transaction ได้โดยไม่ต้องมีข้อมูลทั้งหมด**

มา implement จริง:

```rust
use sha2::{Digest, Sha256};

fn sha256_hex(data: &[u8]) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data);
    hex::encode(hasher.finalize())
}

fn merkle_parent(left: &str, right: &str) -> String {
    let combined = format!("{left}{right}");
    sha256_hex(combined.as_bytes())
}

/// สร้าง Merkle tree จาก leaf data (transaction) แล้วคืนทุก level ของ tree
/// level 0 คือ leaf hash, level สุดท้ายมีสมาชิกตัวเดียวคือ Merkle root
fn build_merkle_tree(leaves: &[String]) -> Vec<Vec<String>> {
    let mut current_level: Vec<String> =
        leaves.iter().map(|l| sha256_hex(l.as_bytes())).collect();
    let mut levels = vec![current_level.clone()];

    while current_level.len() > 1 {
        let mut next_level = Vec::new();
        let mut i = 0;
        while i < current_level.len() {
            let left = &current_level[i];
            // ถ้าจำนวนคี่ ให้ duplicate ตัวสุดท้ายเพื่อจับคู่ (แนวทางเดียวกับที่ Bitcoin ใช้จริง)
            let right = if i + 1 < current_level.len() {
                &current_level[i + 1]
            } else {
                left
            };
            next_level.push(merkle_parent(left, right));
            i += 2;
        }
        levels.push(next_level.clone());
        current_level = next_level;
    }
    levels
}
```

โครงสร้าง proof ต้องเก็บทั้ง sibling hash และ**ตำแหน่ง** (อยู่ซ้ายหรือขวาของเรา) เพราะการต่อ string ก่อน hash
`format!("{left}{right}")` ให้ผลต่างกันถ้าสลับด้าน:

```rust
#[derive(Debug, Clone)]
enum ProofStep {
    Left(String),  // sibling อยู่ทางซ้าย ของเรา (เราอยู่ทางขวา)
    Right(String), // sibling อยู่ทางขวา ของเรา (เราอยู่ทางซ้าย)
}

/// สร้าง Merkle proof สำหรับ leaf ที่ index ที่ระบุ
fn build_merkle_proof(levels: &[Vec<String>], mut index: usize) -> Vec<ProofStep> {
    let mut proof = Vec::new();
    for level in &levels[..levels.len() - 1] {
        let is_right_node = index % 2 == 1;
        let sibling_index = if is_right_node { index - 1 } else { index + 1 };
        let sibling = if sibling_index < level.len() {
            level[sibling_index].clone()
        } else {
            level[index].clone() // กรณีจำนวนคี่ sibling คือตัวมันเอง (duplicate)
        };
        if is_right_node {
            proof.push(ProofStep::Left(sibling));
        } else {
            proof.push(ProofStep::Right(sibling));
        }
        index /= 2;
    }
    proof
}

/// ตรวจสอบ Merkle proof: คำนวณ hash ไล่ขึ้นไปจาก leaf ตาม proof แล้วเทียบกับ root ที่ประกาศไว้
fn verify_merkle_proof(leaf_data: &str, proof: &[ProofStep], expected_root: &str) -> bool {
    let mut current = sha256_hex(leaf_data.as_bytes());
    for step in proof {
        current = match step {
            ProofStep::Left(sibling) => merkle_parent(sibling, &current),
            ProofStep::Right(sibling) => merkle_parent(&current, sibling),
        };
    }
    current == expected_root
}
```

ทดสอบด้วย transaction ห้ารายการจริง:

```rust
fn main() {
    let transactions = vec![
        "tx1: Alice ส่ง 10 ให้ Bob".to_string(),
        "tx2: Bob ส่ง 5 ให้ Carol".to_string(),
        "tx3: Carol ส่ง 2 ให้ Dave".to_string(),
        "tx4: Dave ส่ง 1 ให้ Alice".to_string(),
        "tx5: Alice ส่ง 3 ให้ Eve".to_string(),
    ];
    let levels = build_merkle_tree(&transactions);
    let merkle_root = levels.last().unwrap()[0].clone();
    println!("Merkle root: {merkle_root}");

    let target_index = 2; // tx3
    let proof = build_merkle_proof(&levels, target_index);
    println!("ความยาว proof = {} step", proof.len());

    let is_valid = verify_merkle_proof(&transactions[target_index], &proof, &merkle_root);
    println!("ตรวจสอบ proof กับ root จริง: valid = {is_valid}");

    // จำลองความพยายามปลอมแปลงเนื้อหา transaction
    let is_fake_valid =
        verify_merkle_proof("tx3: Carol ส่ง 999 ให้ Dave (ปลอม)", &proof, &merkle_root);
    println!("ตรวจสอบ proof กับข้อมูลที่ถูกแก้ไข (ปลอม): valid = {is_fake_valid}");
}
```

ผลลัพธ์จริงจากการรัน:

```
จำนวน transaction: 5
จำนวน level ของ tree: 4
  level 0 มี 5 node
  level 1 มี 3 node
  level 2 มี 2 node
  level 3 มี 1 node
Merkle root: 5ebb3086b0a3db24239bc997476612206784920c2947fdda039562ea299b6543

สร้าง Merkle proof สำหรับ 'tx3: Carol ส่ง 2 ให้ Dave' (index 2), ความยาว proof = 3 step
ตรวจสอบ proof กับ root จริง: valid = true
ตรวจสอบ proof กับข้อมูลที่ถูกแก้ไข (ปลอม): valid = false
```

สังเกตว่า transaction 5 รายการใช้ proof แค่ **3 step** เท่านั้นในการพิสูจน์ (แทนที่จะต้องมีข้อมูลทั้ง 5 รายการ) และ
ถ้าเนื้อหาของ transaction ที่ต้องการพิสูจน์ถูกเปลี่ยนแม้แค่ตัวเลขเดียว (จาก "2" เป็น "999") ผลการตรวจสอบจะกลายเป็น
`false` ทันที เพราะ hash ของ leaf ที่คำนวณใหม่ไม่ตรงกับที่ใช้สร้าง proof ไว้แต่แรก — คุณสมบัติ avalanche effect
ของ hash function (จากหัวข้อ 104.1) ทำงานที่นี่เหมือนกันทุกประการ

ในบริบทของ blockchain จริง Merkle root ของ transaction ทั้งหมดใน block คือหนึ่งใน field ที่ block header เก็บไว้
(นอกจาก `previous_hash`) — ทำให้ block header ทั้งก้อนมีขนาดเล็กคงที่ ไม่ว่า block นั้นจะมี transaction กี่พัน
รายการก็ตาม นี่คือเหตุผลที่ light client ตรวจสอบ blockchain ได้โดยดาวน์โหลดแค่ header กับ proof ที่เกี่ยวข้อง ไม่
ต้องดาวน์โหลดข้อมูลทั้งเส้น

### 104.7 Smart Contract คืออะไร และทำไม Determinism ถึงเป็นเรื่องตายตัว

ทุกอย่างที่ทำมาจนถึงตอนนี้ (block, PoW, signature, Merkle tree) คือโครงสร้างพื้นฐานของ "บันทึกข้อมูลแบบไม่มีศูนย์
กลางที่แก้ไขย้อนหลังไม่ได้" — แต่ยังไม่มี "โค้ดที่รันอยู่บน blockchain" เลย **smart contract คือส่วนที่เติมเข้ามา
ตรงนี้**

นิยามที่แม่นยำ: **smart contract คือโค้ดที่ถูก deploy ขึ้นไปอยู่บน blockchain และรันบน virtual machine ของ chain
นั้น ถูกกระตุ้นให้รันโดย transaction ที่ส่งเข้ามา และผลลัพธ์การรัน (state change) ต้องได้รับการยอมรับจากทุก node
ในเครือข่ายว่าถูกต้องเหมือนกัน** — คำว่า "สัญญาอัจฉริยะ" เป็นชื่อที่ทำให้เข้าใจผิดได้ง่าย มันไม่ใช่สัญญาทางกฎหมายและ
ไม่ได้ "ฉลาด" ในความหมายของ AI มันคือ**โค้ดโปรแกรมธรรมดา** ที่มีข้อจำกัดพิเศษหนึ่งข้อที่โค้ดทั่วไปไม่มี — และข้อ
จำกัดนั้นสำคัญพอที่จะเป็นหัวใจของหัวข้อนี้ทั้งหมด:

**Smart contract ต้อง deterministic 100%** หมายความว่า: **ให้ input เดียวกัน (transaction เดียวกัน + state ของ
contract ก่อนหน้าเดียวกัน) ทุก node ในเครือข่ายที่รันโค้ดเดียวกันต้องได้ผลลัพธ์เหมือนกันเป๊ะทุกครั้ง ไม่มีข้อยกเว้น**

ทำไมถึงต้องเข้มงวดขนาดนี้? เพราะกลไก consensus ของ blockchain (Proof of Work หรือแบบอื่น) ทำงานได้ก็เพราะสมมติฐาน
ว่า **"ทุก node ที่ตรวจสอบ transaction เดียวกัน จะเห็นด้วยกันว่า state หลัง transaction นั้นควรเป็นอะไร"** ถ้าโค้ด
ของ smart contract ให้ผลลัพธ์ต่างกันได้ในสอง node ที่รันโค้ดเดียวกันกับ input เดียวกัน เครือข่ายจะ**แตกความเห็น
ทันที** — บาง node เห็นว่า Alice มีเงิน 100 บาง node เห็นว่ามี 99 — และไม่มีทางตัดสินได้ว่าใครถูก เพราะไม่มีศูนย์
กลางที่จะชี้ขาด นี่คือหายนะที่เรียกว่า **consensus failure** และเป็นสิ่งที่ระบบ smart contract ทุกแพลตฟอร์มต้อง
ป้องกันให้ได้ 100%

ย้อนกลับไปดู Part 3 ที่เตือนไว้ว่า floating point ไม่แม่นยำ — ลองพิสูจน์ด้วยโค้ดจริงอีกครั้งในบริบทใหม่นี้:

```rust
fn main() {
    let a: f64 = 0.1;
    let b: f64 = 0.2;
    println!("{:.20}", a + b);
    println!("{}", a + b == 0.3);
}
```

ผลลัพธ์จริง:

```
0.30000000000000004441
false
```

ใน Part 3 ปัญหานี้ถูกนำเสนอในฐานะ "ต้องรู้ไว้เพื่อไม่ตกใจตอน debug" แต่ในบริบท smart contract ผลกระทบร้ายแรงกว่า
เดิมมาก: ถึงแม้ผลลัพธ์ `0.30000000000000004441` จะ**คงที่**บน CPU/compiler เดียวกันเสมอ (IEEE 754 กำหนดผลลัพธ์การ
คำนวณ floating point พื้นฐานไว้แน่นอน) ความเสี่ยงตัวจริงคือ:

1. **CPU/compiler ต่างกันอาจใช้ optimization ที่ต่างกัน** เช่น fused-multiply-add (FMA) ที่มีบน CPU บางรุ่นแต่ไม่มี
   บนบางรุ่น หรือ intermediate precision ที่ต่างกันระหว่าง architecture — ทำให้การคำนวณ floating point แบบซับซ้อน
   (ไม่ใช่แค่บวกสองตัวเลขตรง ๆ) มีโอกาสให้ผลลัพธ์ต่างกันเล็กน้อยระหว่างเครื่องที่ต่างกัน ซึ่งในโปรแกรมทั่วไปต่างกัน
   ระดับ bit สุดท้ายไม่มีผลอะไร แต่ใน smart contract ต่างกัน 1 bit ก็ทำให้ hash ของผลลัพธ์ต่างกันทันที และเครือข่าย
   เห็นไม่ตรงกันทันที
2. Rounding mode และการปัดเศษที่สะสมไปเรื่อย ๆ ในการคำนวณหลายขั้นตอนอาจนำไปสู่ผลต่างที่ขยายขึ้นเมื่อมีการคำนวณซ้ำ
   จำนวนมาก (เช่น คำนวณดอกเบี้ยทบต้นด้วย floating point) — สิ่งที่ยอมรับได้ในแอปทั่วไปกลายเป็นความเสี่ยงต่อ
   consensus ในบริบทนี้

นี่คือเหตุผลที่**แพลตฟอร์ม smart contract ที่ออกแบบมาอย่างรอบคอบ หลีกเลี่ยง floating point ในการคำนวณค่าที่เป็น
ส่วนหนึ่งของ consensus โดยสิ้นเชิง** และใช้ integer arithmetic (เช่น เก็บมูลค่าเป็นจำนวนเต็มระดับ "หน่วยย่อยที่
สุด" อย่าง cent หรือ satoshi แทนทศนิยม) ซึ่งมีผลลัพธ์แน่นอนตายตัว 100% ในทุก CPU/architecture เสมอ

ด้วยตรรกะเดียวกัน มีอีกสองสิ่งที่ smart contract ห้ามใช้โดยสิ้นเชิง:

- **เวลาจริงของระบบ (real wall-clock time)**: `SystemTime::now()` บนเครื่อง node A กับเครื่อง node B ไม่มีทางตรง
  กันเป๊ะเสมอไป (นาฬิกาของเครื่องต่างกัน, network latency ต่างกัน) ถ้า contract ใช้เวลาจริงในการตัดสินใจ (เช่น
  "ถ้าเวลาปัจจุบันเลย deadline ให้ทำ X") สอง node จะได้ผลลัพธ์ต่างกันได้ทันทีถ้า transaction มาถึงคาบเกี่ยวกับ
  deadline พอดี — แพลตฟอร์มจริงแก้ปัญหานี้ด้วยการให้ contract ใช้ **"เวลาของ block" ที่ตกลงกันไว้แล้วในเครือข่าย**
  (เช่น block height หรือ block timestamp ที่ผ่าน consensus มาแล้ว) แทนเวลาจริงของเครื่องตัวเอง
- **Random number generator ทั่วไป**: `rand::rng()` ที่ใช้ในหัวข้อ 104.5 ของบทนี้ (สำหรับสร้าง key pair ในโค้ด
  ตัวอย่างที่รันแค่เครื่องเดียว) **ใช้ใน smart contract logic ไม่ได้เด็ดขาด** เพราะแต่ละ node สุ่มค่าต่างกันแน่นอน
  100% แพลตฟอร์มที่ต้องการ randomness ใน smart contract จริง (เช่น สุ่มผู้ชนะ) ต้องใช้กลไกพิเศษที่ทุก node เห็น
  ค่าเดียวกัน (เช่น commit-reveal scheme หรือ verifiable random function ที่ผลลัพธ์พิสูจน์ได้และเหมือนกันทุก node)
  ไม่ใช่ RNG ทั่วไปที่แต่ละเครื่องสุ่มเอง

ยังมีอีกตัวอย่างที่จับต้องได้มากในบริบทของ Rust โดยเฉพาะ ซึ่งเชื่อมกับ Part 57 ตรง ๆ: การ serialize `HashMap`
ธรรมดาเป็น JSON แล้วนำไป hash ทดสอบจริงในสภาพแวดล้อมตรวจสอบของบทนี้ (โค้ดและผลลัพธ์เต็มอยู่ในกับดักที่พบบ่อย ข้อ 4
ท้ายบทนี้) แสดงผลลัพธ์ที่**ลำดับ key ต่างกันในแต่ละครั้งที่รัน แม้เนื้อหาเหมือนกันทุกประการ** — เพราะ `HashMap`
มาตรฐานของ Rust ใช้ random seed ต่อ process เพื่อป้องกัน HashDoS attack (ผลข้างเคียงของฟีเจอร์ด้านความปลอดภัยที่ดี
ในบริบททั่วไป) ผลคือ JSON string ที่ได้ต่างกัน → byte ที่เข้า hash function ต่างกัน → **hash ต่างกัน** ทั้งที่ข้อมูล
เชิงตรรกะเหมือนกัน 100% — ถ้าโค้ด smart contract เผลอ serialize `HashMap` ก่อน hash แบบนี้ node สอง node ที่รันโค้ด
เดียวกันเป๊ะจะได้ hash ต่างกัน และ consensus พังทันที วิธีแก้คือใช้โครงสร้างข้อมูลที่มี**ลำดับแน่นอนเสมอ**
(`BTreeMap` ที่เรียง key ตามลำดับคงที่, หรือ `Vec` ที่ sort ก่อนเสมอ) แทน `HashMap` ในทุกจุดที่ผลลัพธ์จะถูกนำไป hash
หรือมีผลต่อ consensus

### 104.8 แพลตฟอร์ม Smart Contract ที่เขียนด้วย Rust จริงในโลกจริง

เข้าใจ concept ของ determinism แล้ว มาดูว่าในโลกจริงมีแพลตฟอร์มไหนที่ให้เขียน smart contract ด้วย Rust ได้จริง
บทนี้จะพูดถึงสองตัวหลัก แล้วเลือกตัวหนึ่งมาลงมือทำจริงในหัวข้อ 104.9

**Solana — on-chain program เขียนด้วย Rust โดยตรง** Solana ออกแบบมาให้ on-chain program (สิ่งที่ platform อื่น
เรียกว่า smart contract) **เขียนด้วย Rust เป็นภาษาแรกโดยตรง** ไม่ใช่ผ่านภาษาตัวกลางแบบ Solidity ของ Ethereum — มี
สองทางเลือกในการเขียน: ใช้ crate `solana-program` ตรง ๆ (ระดับ low-level ควบคุมทุกอย่างเอง) หรือใช้ framework
**Anchor** ที่ห่อ boilerplate จำนวนมาก (การตรวจสอบ account, การ serialize/deserialize instruction data) ให้เขียน
ง่ายขึ้นมาก ในสภาพแวดล้อมตรวจสอบของบทนี้ crate `solana-program` (เวอร์ชัน 5.1.0 ณ เวลาที่เขียน) **ถูกเพิ่มเป็น
dependency และ compile ผ่านได้จริง** ยืนยันว่ามันเป็น crate ที่เผยแพร่จริงบน crates.io ใช้งานได้จริงในฐานะ
dependency ของ Rust project ทั่วไป

**CosmWasm — smart contract สำหรับ ecosystem Cosmos** CosmWasm คือ framework ที่ให้เขียน smart contract ด้วย
Rust ธรรมดา แล้ว **compile เป็น WebAssembly (.wasm)** ก่อนนำไป deploy บน blockchain ที่รองรับ (เครือข่ายในตระกูล
Cosmos จำนวนมากรองรับ CosmWasm) จุดเด่นที่สำคัญมากคือ **CosmWasm มีเครื่องมือทดสอบ contract logic แบบ local ทั้ง
หมดผ่าน crate `cosmwasm-std` เอง** ไม่ต้องพึ่ง CLI พิเศษ ไม่ต้องต่อ chain จริงหรือ test validator เลย — สามารถเขียน
unit test ด้วย `cargo test` ธรรมดาได้ทันที

**เหตุผลที่บทนี้เลือก CosmWasm เป็นตัวอย่างที่ลงมือทำจริง (หัวข้อ 104.9):** ในสภาพแวดล้อมตรวจสอบของบทนี้ ตรวจสอบ
พบว่า **ไม่มี `solana` CLI หรือ `anchor` CLI ติดตั้งอยู่** (คำสั่ง `which solana` และ `which anchor` ไม่พบทั้งคู่) —
การทำงานกับ Solana program แบบเต็มรูปแบบต้องมี `solana-test-validator` รันเป็น local blockchain จำลอง ซึ่งต้อง
ติดตั้ง Solana CLI toolchain ที่ไม่มีอยู่ในสภาพแวดล้อมนี้ ในทางกลับกัน CosmWasm ออกแบบมาให้ **การทดสอบ logic ของ
contract ไม่ต้องพึ่ง blockchain จำลองเลยแม้แต่นิดเดียว** — ใช้แค่ `cosmwasm-std::testing` module ที่เป็น pure Rust
ธรรมดา รันผ่าน `cargo test` ปกติ ทำให้เป็นตัวเลือกที่**verify ได้จริงและครบวงจรที่สุด**ในสภาพแวดล้อมนี้ — นี่คือ
การตัดสินใจที่อิงจาก**ความสามารถในการตรวจสอบจริง** ไม่ใช่การประเมินว่าแพลตฟอร์มใดดีกว่ากันในภาพรวม (ทั้ง Solana
และ CosmWasm ต่างมีจุดแข็งของตัวเองที่เหมาะกับสถานการณ์ต่างกัน อ่านเพิ่มได้จากหัวข้อ 104.11)

ยังมีอีกโปรเจกต์หนึ่งที่ควรรู้จักในบริบทนี้: **Substrate** (framework จาก Parity Technologies ที่ใช้สร้าง
Polkadot) — ต่างจาก Solana/CosmWasm ที่เขียน "contract" มารันบน chain ที่มีอยู่แล้ว Substrate ให้คุณสร้าง
**blockchain ทั้งเส้นของตัวเอง** (custom runtime) ด้วย Rust ทั้งหมด เหมาะกับกรณีที่ต้องการควบคุมพฤติกรรมของ
เครือข่ายระดับลึกกว่าการเขียน contract ธรรมดา — เกินขอบเขตของตัวอย่างที่ลงมือทำได้ในบทเดียว แต่ควรรู้จักไว้ในฐานะ
ข้อเท็จจริงของอุตสาหกรรม

### 104.9 ลงมือจริง: เขียนและทดสอบ CosmWasm Smart Contract แบบ Local ทั้งหมด

มาสร้าง smart contract จริงด้วย CosmWasm: **counter contract** — เก็บตัวเลขหนึ่งตัวไว้ใน state ของ contract,
มีคำสั่งเพิ่มค่า (`Increment`) ที่ใครก็เรียกได้ และคำสั่ง reset (`Reset`) ที่มีแค่ owner (คนที่ deploy contract)
เท่านั้นที่เรียกได้ — ตัวอย่างเล็กแต่ครบทุกองค์ประกอบสำคัญของ smart contract จริง: state, การเขียน state
(`execute`), การอ่าน state (`query`), และการตรวจสอบสิทธิ์ (authorization)

สร้างโปรเจกต์และเพิ่ม dependency (เวอร์ชันที่ตรวจสอบจริงแล้วในบทนี้):

```toml
# Cargo.toml
[package]
name = "counter_contract"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[features]
library = []

[dependencies]
cosmwasm-std = "3.0.9"
cosmwasm-schema = "3.0.9"
cw-storage-plus = "3.0.1"
serde = { version = "1.0.229", features = ["derive"] }
thiserror = "2.0.21"
```

จุดที่ควรสังเกต: `crate-type = ["cdylib", "rlib"]` — `cdylib` คือรูปแบบที่จำเป็นสำหรับ compile ไปเป็น `.wasm`
(เหมือนกับที่ Part 86 อธิบายไว้เรื่อง `wasm32-unknown-unknown` target) ส่วน `rlib` ทำให้ crate ตัวเองยังใช้เป็น
library ธรรมดาได้ด้วย (จำเป็นสำหรับรัน `cargo test` แบบปกติในเครื่องพัฒนา ไม่ใช่ target wasm) `feature = "library"`
เป็น pattern มาตรฐานของ CosmWasm ที่ให้ปิดการ export entry point ตอนที่ contract ถูกใช้เป็น library โดย contract
อื่น (ไม่เกี่ยวกับตัวอย่างนี้โดยตรง แต่เป็น boilerplate มาตรฐานที่ควรมีไว้)

```rust
use cosmwasm_std::{
    entry_point, to_json_binary, Binary, Deps, DepsMut, Env, MessageInfo, Response, StdResult,
};
use cw_storage_plus::Item;
use serde::{Deserialize, Serialize};
use thiserror::Error;

// ---------- State ----------
// เก็บค่าตัวนับตัวเดียวไว้ใน contract storage ด้วย key คงที่ "count"
const COUNT: Item<i64> = Item::new("count");
// เก็บ address ของคนที่มีสิทธิ์ reset ตัวนับ (ตั้งตอน instantiate)
const OWNER: Item<String> = Item::new("owner");

// ---------- Messages (เข้า-ออก contract ผ่าน JSON เสมอ) ----------

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct InstantiateMsg {
    pub initial_count: i64,
}

#[derive(Serialize, Deserialize, Clone, Debug)]
#[serde(rename_all = "snake_case")]
pub enum ExecuteMsg {
    Increment {},
    Reset { count: i64 },
}

#[derive(Serialize, Deserialize, Clone, Debug)]
#[serde(rename_all = "snake_case")]
pub enum QueryMsg {
    GetCount {},
}

#[derive(Serialize, Deserialize, Debug)]
pub struct CountResponse {
    pub count: i64,
}

#[derive(Error, Debug)]
pub enum ContractError {
    #[error("Std error: {0}")]
    Std(#[from] cosmwasm_std::StdError),

    #[error("Unauthorized: มีแค่ owner เท่านั้นที่ reset ตัวนับได้")]
    Unauthorized {},
}
```

โครงสร้าง message สามชนิดนี้คือ pattern มาตรฐานของ CosmWasm: `InstantiateMsg` (ข้อมูลตอน deploy contract ครั้งแรก
ครั้งเดียว), `ExecuteMsg` (คำสั่งที่เปลี่ยน state — ต้องมาพร้อม transaction เสมอ), `QueryMsg` (คำสั่งอ่าน state
อย่างเดียว ไม่เปลี่ยนอะไร ไม่ต้องมี transaction) — การแยกชัดขนาดนี้คือส่วนหนึ่งของการันตี determinism: `query`
รับประกันไม่มีทางแก้ state ได้เลยแม้จะเผลอเขียนโค้ดผิดในนั้น เพราะ signature ของฟังก์ชัน (`Deps` แบบ read-only
ไม่ใช่ `DepsMut`) บังคับไว้ตั้งแต่ระดับ type system

ตอนนี้มาเขียน entry point จริง:

```rust
// #[entry_point] ทำให้ฟังก์ชันเหล่านี้ถูก export ออกไปเป็นจุดที่ WASM runtime ของ chain เรียกเข้ามาได้จาก
// ภายนอก (เทียบได้กับ #[no_mangle] extern "C" ใน FFI แบบดิบที่ Part 43 สอนไว้ แต่ macro นี้ generate wrapper ที่
// (de)serialize JSON ให้ครบ ไม่ต้องเขียน FFI boundary ด้วยมือ)

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: InstantiateMsg,
) -> Result<Response, ContractError> {
    COUNT.save(deps.storage, &msg.initial_count)?;
    OWNER.save(deps.storage, &info.sender.to_string())?;
    Ok(Response::new()
        .add_attribute("action", "instantiate")
        .add_attribute("initial_count", msg.initial_count.to_string())
        .add_attribute("owner", info.sender.to_string()))
}

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    _env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::Increment {} => execute_increment(deps),
        ExecuteMsg::Reset { count } => execute_reset(deps, info, count),
    }
}

fn execute_increment(deps: DepsMut) -> Result<Response, ContractError> {
    // ทุก node ที่รัน transaction นี้ต้องได้ผลลัพธ์เดียวกันเป๊ะ: บวก 1 เข้าค่าที่อ่านจาก storage
    // deterministic 100% ตามหลักการหัวข้อ 104.7 — ไม่มี floating point, ไม่มีเวลาจริง, ไม่มี randomness เจือปนเลย
    let new_count = COUNT.update(deps.storage, |count| -> StdResult<_> { Ok(count + 1) })?;
    Ok(Response::new()
        .add_attribute("action", "increment")
        .add_attribute("new_count", new_count.to_string()))
}

fn execute_reset(deps: DepsMut, info: MessageInfo, count: i64) -> Result<Response, ContractError> {
    let owner = OWNER.load(deps.storage)?;
    if info.sender.to_string() != owner {
        return Err(ContractError::Unauthorized {});
    }
    COUNT.save(deps.storage, &count)?;
    Ok(Response::new()
        .add_attribute("action", "reset")
        .add_attribute("new_count", count.to_string()))
}

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::GetCount {} => {
            let count = COUNT.load(deps.storage)?;
            to_json_binary(&CountResponse { count })
        }
    }
}
```

สังเกตว่า `execute_reset` ตรวจสอบ `info.sender` (address ของคนที่ส่ง transaction เข้ามา — ระบุตัวได้ด้วยกลไก
signature เดียวกับหัวข้อ 104.5 แต่ระดับ chain runtime ตรวจให้ก่อนที่ transaction จะมาถึง contract แล้ว) เทียบกับ
`owner` ที่บันทึกไว้ตอน `instantiate` — นี่คือ pattern การตรวจสอบสิทธิ์มาตรฐานของ smart contract ทุกแพลตฟอร์ม: **ไม่
เชื่อ input ใด ๆ ที่มาจากผู้เรียก ต้องตรวจสอบสิทธิ์ก่อนแก้ state ทุกครั้ง** (เชื่อมกับหลักการ "ห้ามเชื่อ input"
ที่ Part 100 สอนไว้ในบริบท security ทั่วไป — ในบริบท smart contract การตรวจสอบสิทธิ์พลาดหนึ่งจุดแปลว่าใครก็ตาม
ปลอมตัวเป็น owner ได้ทันที)

**ส่วนที่สำคัญที่สุดของหัวข้อนี้: การทดสอบ logic แบบ local ทั้งหมด** CosmWasm มี `cosmwasm_std::testing` module
ให้ mock ทุกอย่างที่ contract ต้องการโดยไม่ต้องมี blockchain จริงรันอยู่เลย:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use cosmwasm_std::testing::{message_info, mock_dependencies, mock_env};
    use cosmwasm_std::{coins, from_json};

    #[test]
    fn instantiate_and_query_initial_count() {
        let mut deps = mock_dependencies();
        let creator = deps.api.addr_make("creator");
        let info = message_info(&creator, &coins(0, "token"));
        let msg = InstantiateMsg { initial_count: 17 };

        let res = instantiate(deps.as_mut(), mock_env(), info, msg).unwrap();
        assert_eq!(res.attributes[1].value, "17");

        let bin = query(deps.as_ref(), mock_env(), QueryMsg::GetCount {}).unwrap();
        let value: CountResponse = from_json(&bin).unwrap();
        assert_eq!(value.count, 17);
    }

    #[test]
    fn increment_adds_one_each_time_deterministically() {
        let mut deps = mock_dependencies();
        let creator = deps.api.addr_make("creator");
        let info = message_info(&creator, &[]);
        instantiate(
            deps.as_mut(),
            mock_env(),
            info.clone(),
            InstantiateMsg { initial_count: 0 },
        )
        .unwrap();

        for expected in 1..=5 {
            execute(deps.as_mut(), mock_env(), info.clone(), ExecuteMsg::Increment {}).unwrap();
            let bin = query(deps.as_ref(), mock_env(), QueryMsg::GetCount {}).unwrap();
            let value: CountResponse = from_json(&bin).unwrap();
            assert_eq!(value.count, expected);
        }
    }

    #[test]
    fn reset_by_owner_succeeds() {
        let mut deps = mock_dependencies();
        let owner = deps.api.addr_make("owner");
        let owner_info = message_info(&owner, &[]);
        instantiate(
            deps.as_mut(),
            mock_env(),
            owner_info.clone(),
            InstantiateMsg { initial_count: 100 },
        )
        .unwrap();

        execute(deps.as_mut(), mock_env(), owner_info, ExecuteMsg::Reset { count: 0 }).unwrap();

        let bin = query(deps.as_ref(), mock_env(), QueryMsg::GetCount {}).unwrap();
        let value: CountResponse = from_json(&bin).unwrap();
        assert_eq!(value.count, 0);
    }

    #[test]
    fn reset_by_non_owner_fails_with_unauthorized() {
        let mut deps = mock_dependencies();
        let owner = deps.api.addr_make("owner");
        let attacker = deps.api.addr_make("attacker");
        let owner_info = message_info(&owner, &[]);
        let attacker_info = message_info(&attacker, &[]);

        instantiate(
            deps.as_mut(),
            mock_env(),
            owner_info,
            InstantiateMsg { initial_count: 50 },
        )
        .unwrap();

        let err = execute(
            deps.as_mut(),
            mock_env(),
            attacker_info,
            ExecuteMsg::Reset { count: 0 },
        )
        .unwrap_err();

        match err {
            ContractError::Unauthorized {} => {}
            other => panic!("คาดว่าจะได้ Unauthorized แต่ได้ {other:?}"),
        }
    }
}
```

`mock_dependencies()`, `mock_env()`, และ `message_info()` คือฟังก์ชันจาก `cosmwasm_std::testing` ที่จำลอง
สภาพแวดล้อมของ chain ทั้งหมด (storage, environment info, ผู้ส่ง transaction) ให้เป็น pure Rust struct ธรรมดาที่รัน
ในเครื่องพัฒนาได้ทันที — `deps.api.addr_make("creator")` สร้าง address จำลองที่ valid ตามรูปแบบของ chain (ไม่ใช่
address จริงบน mainnet ใด ๆ)

รันด้วย `cargo test` จริง ได้ผลลัพธ์นี้ (รันจริงในสภาพแวดล้อมตรวจสอบของบทนี้):

```
running 4 tests
test tests::reset_by_non_owner_fails_with_unauthorized ... ok
test tests::instantiate_and_query_initial_count ... ok
test tests::reset_by_owner_succeeds ... ok
test tests::increment_adds_one_each_time_deterministically ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทั้งสี่ test ผ่านจริง — และยังทดลอง compile contract นี้ไปเป็น `.wasm` จริงด้วย target `wasm32-unknown-unknown`
(target ตัวเดียวกันที่ Part 86 สอนไว้) สำเร็จด้วย ได้ไฟล์ `counter_contract.wasm` ขนาดจริง 276,272 byte (ยังไม่ผ่าน
การ optimize ขนาดแบบที่ workflow การ deploy จริงต้องทำ ซึ่งใช้เครื่องมือ `cosmwasm/optimizer` แบบ Docker ที่ไม่ได้
ติดตั้งในสภาพแวดล้อมตรวจสอบนี้)

**สรุปให้ชัดว่าอะไรถูกพิสูจน์แล้วจริง และอะไรยังไม่ถูกพิสูจน์ในบทนี้:**

| สิ่งที่ทำ | สถานะ |
|---|---|
| Contract logic (instantiate/execute/query, การตรวจสอบสิทธิ์ owner) compile ผ่านจริง | ✅ พิสูจน์แล้วจริง — compile สำเร็จในสภาพแวดล้อมตรวจสอบ |
| Unit test 4 เคส (instantiate, increment, reset โดย owner, reset โดยไม่ใช่ owner) ผ่านทั้งหมด | ✅ พิสูจน์แล้วจริง — รันด้วย `cargo test` จริง ได้ผลลัพธ์ตรงตามที่แสดงด้านบน |
| Compile เป็น `.wasm` จริงด้วย target `wasm32-unknown-unknown` | ✅ พิสูจน์แล้วจริง — ได้ไฟล์ `.wasm` ขนาด 276,272 byte จริง |
| การ optimize ขนาด `.wasm` ด้วย `cosmwasm/optimizer` (Docker) ตามมาตรฐานก่อน deploy จริง | ❌ ไม่ได้ทำ — Docker workflow นี้ไม่ได้ติดตั้งในสภาพแวดล้อมตรวจสอบ |
| การ deploy ขึ้น testnet/mainnet จริงของเครือข่ายในตระกูล Cosmos | ❌ ไม่ได้ทำ — ต้องมี wallet, token จริงสำหรับค่า gas, และการเชื่อมต่อเครือข่ายจริงที่ไม่มีในสภาพแวดล้อมนี้ |
| การทดสอบ interaction ข้าม contract หลายตัว หรือกับ chain module อื่น ๆ (เช่น bank module) | ❌ ไม่ได้ทำ — ต้องใช้เครื่องมือเพิ่มเติมอย่าง `cw-multi-test` ซึ่งเกินขอบเขตตัวอย่างนี้ |

ความแตกต่างระหว่างสองกลุ่มนี้คือประเด็นสำคัญที่สุดของหัวข้อ 104.11: **"contract logic ถูกต้องและผ่าน test ในเครื่อง
พัฒนา" กับ "contract พร้อม deploy สู่ production จริง" เป็นคนละเรื่องกันโดยสิ้นเชิง**

### 104.10 WebAssembly กับ Blockchain: จุดที่ Module 5 เชื่อมกับบทนี้ตรง ๆ

หัวข้อที่แล้วน่าจะทำให้สังเกตเห็นแล้วว่า CosmWasm compile contract เป็น **WebAssembly (.wasm)** ก่อนนำไป deploy —
นี่ไม่ใช่เรื่องบังเอิญ Part 86 เคยอธิบายไว้ว่า WASM ถูกออกแบบมาให้มีคุณสมบัติสามข้อหลัก: **sandboxed** (โค้ดที่รันใน
WASM เข้าถึงได้แค่สิ่งที่ host โดยตั้งใจเปิดให้เข้าถึงเท่านั้น ผ่าน linear memory ที่แยกจากระบบจริงโดยสิ้นเชิง),
**deterministic** (WASM instruction set ถูกกำหนดไว้แน่นอน ไม่มีพฤติกรรมที่ผลลัพธ์ขึ้นกับ hardware แบบที่ native
code บางแบบมี), และ **portable** (binary รูปแบบเดียวรันได้ในทุก environment ที่มี WASM runtime โดยไม่ต้อง compile
ใหม่)

สามคุณสมบัตินี้แหละที่ทำให้ WASM เหมาะกับ smart contract เป็นพิเศษ — และเหตุผลตรงกับปัญหาที่หัวข้อ 104.7 อธิบายไว้
พอดี:

- **Sandboxed** แก้ปัญหาความปลอดภัยของ smart contract โดยตรง: contract ที่ deploy โดยใครก็ได้ (รวมถึงคนไม่รู้จัก
  ทั่วโลก) ต้อง**ไม่มีทางเข้าถึงระบบไฟล์, เครือข่าย, หรือ memory ของ contract อื่นได้เลย** ไม่ว่า logic ภายในจะ
  เขียนผิดพลาดแค่ไหนก็ตาม — WASM การันตีสิ่งนี้ในระดับ execution model ไม่ต้องพึ่งการตรวจโค้ดด้วยมือ
- **Deterministic** คือคำตอบตรงต่อความต้องการของหัวข้อ 104.7 ทั้งหมด — WASM instruction ทุกตัวถูกกำหนดผลลัพธ์ไว้
  แน่นอนตายตัว (ต่างจาก native machine code ที่อาจมีพฤติกรรมต่างกันเล็กน้อยระหว่าง CPU รุ่นต่าง ๆ ตามที่คุยไว้เรื่อง
  floating point) ทำให้ node ทุกตัวในเครือข่ายที่รัน WASM bytecode เดียวกัน**การันตีได้ในระดับ execution model**
  ว่าจะได้ผลลัพธ์เหมือนกัน — นี่คือเหตุผลเชิงเทคนิคที่แท้จริงว่าทำไม CosmWasm (และแพลตฟอร์มอื่นอย่าง Near) เลือก
  WASM เป็น target แทนการรัน native code ตรง ๆ
- **Portable** ทำให้ validator node ที่รันบน hardware/OS ต่างกันทั่วโลก (Linux, ARM, x86 ฯลฯ) รัน `.wasm` ไฟล์
  เดียวกันได้โดยไม่ต้อง compile ใหม่สำหรับแต่ละ architecture — สำคัญมากสำหรับเครือข่ายแบบ decentralized ที่ node
  ควบคุมโดยคนละกลุ่มคนละที่ทั่วโลก ไม่มีทางบังคับให้ทุกคนใช้ hardware เดียวกัน

สิ่งที่ Part 86 สอนไว้เรื่อง **linear memory** (memory ทั้งหมดของ WASM คือ byte array ต่อเนื่องผืนเดียว pointer
เป็นแค่ตัวเลข offset) ก็นำมาใช้ตรงในบริบทนี้เช่นกัน: เมื่อ chain runtime เรียก entry point ของ contract (เช่น
`execute` ที่เขียนในหัวข้อ 104.9) มันส่งข้อมูล instruction (JSON ที่ serialize มาแล้ว) เข้าไปโดยเขียนลง linear
memory ของ WASM instance นั้น แล้ว contract อ่านออกมา deserialize เป็น Rust struct — นี่คือกลไกเดียวกับที่ Part 86
อธิบายไว้เรื่องปัญหาการส่งค่าที่ซับซ้อนกว่าตัวเลขข้ามพรมแดน WASM/host ไม่ได้อย่างตรง ๆ ต้องมีชั้นแปลงข้อมูลกลาง —
เพียงแต่ในบริบทของ browser นั้นชั้นแปลงข้อมูลคือ `wasm-bindgen` (ที่ Part 87 จะสอน) ส่วนในบริบทของ blockchain
ชั้นแปลงข้อมูลคือ CosmWasm runtime layer ที่ macro `#[entry_point]` generate ให้ — **หลักการเดียวกันเป๊ะ
implementation ต่างกันตามบริบท**

จุดที่ชวนให้เห็นภาพรวมสำคัญ: **ทักษะเรื่อง WASM ที่เรียนมาใน Module 5 ไม่ได้ผูกติดอยู่กับ "การรันโค้ด Rust ใน
เบราว์เซอร์" เท่านั้น** มันคือความรู้เรื่อง **execution model แบบ sandboxed/deterministic/portable** ที่ถูกนำไปใช้
ในบริบทไหนก็ได้ที่ต้องการคุณสมบัติแบบนี้ — เบราว์เซอร์เป็นบริบทแรกที่ WASM ถูกออกแบบมาให้ใช้ แต่ blockchain คือ
บริบทที่สองที่ต้องการคุณสมบัติชุดเดียวกันนี้พอดิบพอดี และนำ WASM ไปใช้ต่อโดยตรง

### 104.11 ขอบเขตของบทนี้ ความปลอดภัย และแหล่งเรียนรู้ต่อ

ก่อนปิดบท ต้องพูดตรง ๆ อย่างชัดเจนถึงขอบเขตของสิ่งที่เรียนมา: **บทนี้ให้ความรู้กว้างระดับตระหนักรู้
(broad awareness) เกี่ยวกับสาขา blockchain/smart contract ที่ใหญ่และเปลี่ยนแปลงเร็วมาก มันไม่ใช่เส้นทางที่ทำให้
พร้อม deploy ระบบ blockchain สู่ production ได้ทันทีหลังจบบทนี้** การไปถึงจุดนั้นได้จริงต้องมีการศึกษาลึกเพิ่มอีก
มากในแพลตฟอร์ม/consensus mechanism ที่เลือกใช้จริงโดยเฉพาะ — สิ่งที่บทนี้ให้คือ**พื้นฐานที่ถูกต้องและพิสูจน์ได้จริง
ทุกจุด**ให้ไปต่อยอดศึกษาลึกได้เร็วขึ้น ไม่ใช่ทางลัดที่ทำให้ข้ามการศึกษาลึกไปได้

**ความปลอดภัยคือประเด็นที่ต้องเน้นย้ำเป็นพิเศษในบริบทนี้** Part 100 สอนไว้ว่า memory safety ของ Rust ป้องกัน bug
class หนึ่งกลุ่ม (buffer overflow, use-after-free, data race) แต่**ไม่ป้องกัน logic bug** — ในโปรแกรมทั่วไป
logic bug แปลว่าต้องแก้แล้ว deploy เวอร์ชันใหม่ แต่ **ใน smart contract ที่ deploy ขึ้น chain แล้ว หลาย
แพลตฟอร์มไม่อนุญาตให้แก้โค้ดที่ deploy ไปแล้วโดยตรง** (บาง platform รองรับ upgradeable contract pattern แต่ก็มี
ความเสี่ยงและความซับซ้อนของตัวมันเองเพิ่มเข้ามา) — ผลคือ **logic bug ใน smart contract อาจกลายเป็นความเสียหายทาง
การเงินที่แก้ไขย้อนหลังไม่ได้จริง ๆ** ต่างจาก bug ในเว็บแอปทั่วไปที่ patch แล้วจบ นี่คือเหตุผลที่วงการ smart
contract ให้ความสำคัญกับ **security audit โดยทีมผู้เชี่ยวชาญเฉพาะทาง** ก่อน deploy ขึ้น mainnet จริงเสมอ ไม่ว่า
โค้ดจะดูเรียบง่ายแค่ไหนก็ตาม — ตัวอย่าง counter contract ในหัวข้อ 104.9 ที่ดูเรียบง่ายมาก ก็ยังมีจุดที่ผิดพลาดได้
จริง เช่น integer overflow ถ้าเรียก `Increment` มากพอ (ใน Rust แบบ debug build จะ panic ให้เห็นทันที แต่แบบ
release build ที่ compile แบบ default จะ wrap around แบบเงียบ ๆ ตามพฤติกรรมมาตรฐานของ Rust ที่ Part 3 อธิบายไว้ —
เป็นเหตุผลหนึ่งที่ smart contract จริงมักเลือกใช้ arithmetic ที่ checked/saturating อย่างชัดเจนเสมอ ไม่พึ่งพฤติกรรม
default)

สำหรับใครที่ต้องการศึกษาต่อจริงจัง แหล่งข้อมูลที่ควรไปอ่านต่อ (เอกสารทางการของแต่ละแพลตฟอร์ม ไม่ใช่บล็อกหรือ
คอร์สออนไลน์ที่ไม่มีการดูแลคุณภาพ):

- **Solana**: เอกสารทางการที่ solana.com/docs และ Anchor framework ที่มีหนังสือประกอบชื่อ "The Anchor Book"
  อธิบาย pattern การเขียน on-chain program แบบละเอียด รวมถึงเรื่อง account model ที่ Solana ออกแบบไว้ต่างจาก
  แพลตฟอร์มอื่นค่อนข้างมาก (ต้องศึกษาแยกเพราะเป็นแนวคิดที่เฉพาะตัวของ Solana)
- **CosmWasm**: เอกสารทางการที่ docs.cosmwasm.com ครอบคลุมทั้งการเขียน contract ขั้นสูงกว่าตัวอย่างในบทนี้ (การ
  ทำงานข้าม contract, IBC สำหรับสื่อสารข้าม chain ใน ecosystem Cosmos) และ workflow การ deploy จริง
- **Substrate/Polkadot**: เอกสารทางการที่ docs.substrate.io สำหรับผู้ที่สนใจสร้าง custom blockchain runtime ทั้ง
  เส้นด้วย Rust ไม่ใช่แค่เขียน contract บน chain ที่มีอยู่แล้ว

สุดท้าย ควรมองภาพรวมให้ถูกต้อง: สิ่งที่เรียนมาในบทนี้ทั้งหมด — data structure ที่เชื่อมด้วย hash, Proof of Work,
digital signature, Merkle tree, determinism ของ smart contract, WASM execution model — **ล้วนเป็นการประยุกต์
ทักษะ Rust พื้นฐานที่เรียนมาตลอดทั้งหลักสูตรนี้** (structs, traits, serde, hashing, error handling, testing) เข้า
กับปัญหาทางวิศวกรรมที่มีข้อจำกัดเฉพาะตัว (ต้อง deterministic, ต้อง tamper-evident, ต้อง verifiable โดยไม่มีศูนย์
กลาง) — ไม่ใช่สาขาความรู้ที่แยกออกจาก Rust ทั่วไปโดยสิ้นเชิงอย่างที่มักถูกมองกัน

## กับดักที่พบบ่อย (Common Pitfalls)

**1. แก้ไขข้อมูลใน struct ของ block โดยตรง แล้วลืมว่า `hash` เดิมไม่ valid อีกต่อไป**

นี่ไม่ใช่ compiler error แต่เป็น logic bug ที่พบบ่อยที่สุดของคนเขียน blockchain เดโมมือใหม่ — โค้ดคอมไพล์ผ่าน
รันได้ ไม่ panic แต่ chain กลายเป็นข้อมูลที่ไม่สมเหตุสมผลอย่างเงียบ ๆ พิสูจน์ได้จริงด้วยการรันโค้ดจากหัวข้อ 104.2:

```
Blockchain ทั้งหมดมี 4 blocks, valid = true

--- ทดสอบ tamper-evidence: แก้ไขข้อมูลใน block #1 โดยไม่ mine ใหม่ ---
หลังแก้ไข data ของ block #1 (ไม่คำนวณ hash ใหม่): valid = false
```

วิธีป้องกัน: **ห้าม expose field ของ `Block` ให้แก้ไขได้ตรง ๆ จากนอก module** ในระบบจริงควรทำให้ field เป็น
private และเปิดแค่ constructor ที่คำนวณ hash ให้ครบเสมอ ไม่เปิดทางให้ mutate field ที่มีผลต่อ hash ได้เลยหลังจาก
สร้าง block แล้ว

**2. เรียก `sign()`/`verify()` ของ `ed25519-dalek` โดยไม่ import trait `Signer`/`Verifier` เข้ามา**

```
error[E0599]: no method named `sign` found for struct `SigningKey` in the current scope
   --> src/main.rs:269:44
    |
269 |     let signature: Signature = signing_key.sign(&tx_bytes);
    |                                            ^^^^
    |
    = help: items from traits can only be used if the trait is in scope
help: trait `Signer` which provides `sign` is implemented but not in scope; perhaps you want to import it
    |
  1 + use ed25519_dalek::Signer;
    |
```

`sign` และ `verify` ไม่ใช่ method ปกติของ `SigningKey`/`VerifyingKey` — มันมาจาก trait `Signer`/`Verifier` (จาก
crate `signature` ที่ re-export ผ่าน `ed25519_dalek`) ตามที่ Part 19-21 สอนไว้เรื่อง trait method ที่ต้องมี trait
อยู่ใน scope ก่อนเรียกใช้ได้ วิธีแก้ตรงตามที่ compiler แนะนำ: `use ed25519_dalek::{Signer, Verifier};`

**3. ลืมเปิด feature `rand_core` ของ `ed25519-dalek` แล้วเรียก `SigningKey::generate`**

`ed25519-dalek` เวอร์ชัน 3.0 ไม่เปิด feature สำหรับสร้าง key แบบสุ่มมาให้ default (เพื่อให้ dependency tree เล็ก
ที่สุดสำหรับคนที่ไม่ต้องการ generate key ในโค้ด เช่น โค้ดที่แค่ verify signature ที่มีอยู่แล้ว) ถ้าเพิ่ม
dependency ด้วย `cargo add ed25519-dalek` เฉย ๆ โดยไม่ระบุ feature จะเจอ error ว่าไม่พบ method `generate` เลย
วิธีแก้: ต้องเปิด feature ตรง ๆ ใน `Cargo.toml`:

```toml
ed25519-dalek = { version = "3.0.0", features = ["rand_core"] }
```

กับดักนี้เป็นตัวอย่างที่ดีของสิ่งที่ Part 17 สอนไว้เรื่อง feature flag ของ Cargo: crate สาย cryptography จำนวนมาก
เปิด functionality บางส่วนไว้หลัง feature flag เพื่อลด attack surface และขนาด binary ของคนที่ไม่ต้องการฟีเจอร์นั้น
— ควรอ่าน documentation ของ crate cryptography ทุกตัวให้ละเอียดเรื่อง feature ที่เปิด/ปิด ก่อนใช้งานจริงเสมอ

**4. Serialize `HashMap` ก่อนนำไป hash — ได้ hash ต่างกันทุกครั้งที่รัน ทั้งที่ข้อมูลเหมือนกัน**

นี่คือกับดักที่อันตรายที่สุดในบรรดาทั้งหมด เพราะโค้ดคอมไพล์ผ่าน รันได้ ไม่มี error ใด ๆ แต่ผิดพลาดในเชิง
determinism ซึ่งเป็นหัวใจของ smart contract ตามหัวข้อ 104.7 พิสูจน์ได้จริงด้วยโค้ดสั้น ๆ:

```rust
use std::collections::HashMap;

fn main() {
    let mut m: HashMap<String, String> = HashMap::new();
    m.insert("alice".to_string(), "100".to_string());
    m.insert("bob".to_string(), "50".to_string());
    m.insert("carol".to_string(), "25".to_string());
    m.insert("dave".to_string(), "10".to_string());
    println!("{}", serde_json::to_string(&m).unwrap());
}
```

รันสามครั้ง (คนละ process) ได้สาม string ที่ต่างกันจริง (ทดสอบจริงในสภาพแวดล้อมตรวจสอบของบทนี้):

```
{"alice":"100","dave":"10","carol":"25","bob":"50"}
{"dave":"10","carol":"25","alice":"100","bob":"50"}
{"alice":"100","bob":"50","carol":"25","dave":"10"}
```

สาเหตุคือ `HashMap` มาตรฐานของ Rust ใช้ random seed ต่อ process เพื่อป้องกัน HashDoS attack ตามที่ Part 15
(HashMap and Sets) เคยกล่าวถึงไว้ — ฟีเจอร์ด้านความปลอดภัยที่ดีในบริบททั่วไป แต่เป็นหายนะถ้าเผลอนำผลลัพธ์การ
serialize ไปเข้า hash function ที่ต้องได้ผลเดียวกันทุกครั้ง วิธีแก้: ใช้ `BTreeMap` (เรียง key ตามลำดับคงที่เสมอ)
แทน `HashMap` ในทุกจุดที่ผลลัพธ์จะถูกนำไป hash หรือมีผลต่อ consensus โดยตรง

**5. ตรวจสอบ Merkle proof แต่ลืมว่าตำแหน่ง (ซ้าย/ขวา) ของ sibling มีผลต่อผลลัพธ์**

เพราะการรวม hash สองตัวคือการต่อ string ก่อน hash (`format!("{left}{right}")`) การสลับด้านซ้าย/ขวาทำให้ได้
hash คนละตัวแม้ sibling เป็นค่าเดียวกัน ถ้าโครงสร้าง proof ไม่เก็บข้อมูลตำแหน่งไว้ (เก็บแค่ `Vec<String>` ของ
sibling hash เฉย ๆ โดยไม่รู้ว่าแต่ละตัวอยู่ซ้ายหรือขวา) การ verify จะได้ผลลัพธ์ผิดแบบเงียบ ๆ โดยไม่มี error ใด ๆ
เตือน — วิธีแก้ตามที่ทำในหัวข้อ 104.6: ให้ proof step เป็น enum ที่บอกตำแหน่งชัดเจนเสมอ (`ProofStep::Left`/
`ProofStep::Right`) ไม่ใช่แค่ `String` เปล่า ๆ

**6. เขียน `message_info("sender_string", &[])` แบบ tutorial เก่าของ CosmWasm แล้วเจอ type mismatch**

Tutorial และตัวอย่างเก่าของ CosmWasm จำนวนมากในอินเทอร์เน็ตใช้ฟังก์ชันชื่อ `mock_info(sender: &str, funds: &[Coin])`
ซึ่งถูกแทนที่ใน `cosmwasm-std` เวอร์ชัน 3.x ด้วย `message_info(sender: &Addr, funds: &[Coin])` — ถ้าลองเขียนตาม
tutorial เก่าโดยส่ง `&str` ตรง ๆ จะเจอ type mismatch error ทันที เพราะ parameter คือ `&Addr` ไม่ใช่ `&str` วิธีแก้
คือสร้าง `Addr` ก่อนด้วย `deps.api.addr_make("sender_name")` (ตามที่ทำในหัวข้อ 104.9) แล้วส่ง reference ของมันเข้า
`message_info` — กับดักนี้เป็นตัวเตือนที่ดีว่า crate ที่เปลี่ยนเวอร์ชัน major บ่อย (เช่น `cosmwasm-std` ที่ขยับจาก
2.x ไป 3.x) อาจเปลี่ยน API ของ testing utility ได้เหมือนกัน ควรเช็ค changelog/documentation เวอร์ชันที่ใช้จริง
เสมอ ไม่ใช่ก็อปโค้ดจาก tutorial เก่าโดยไม่ตรวจสอบ

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** จากโค้ด `Blockchain` ในหัวข้อ 104.2 เพิ่ม method `tamper_check_report(&self) -> Vec<String>` ที่คืน
   list ของข้อความอธิบายว่า block ไหนบ้าง (ระบุ index) ที่ทำให้ chain invalid แทนที่จะคืนแค่ `bool` เดียวเหมือน
   `is_valid()` — hint: วน loop เหมือน `is_valid()` เดิม แต่แทนที่จะ `return false` ทันที ให้ push ข้อความอธิบาย
   เข้า `Vec` แล้ววนต่อไปให้ครบทุก block ก่อนคืนค่า

2. **(กลาง)** เพิ่ม field `transactions: Vec<String>` ให้กับ `Block` (แทนที่ `data: String` เดิม) แล้วปรับ
   `calculate_hash` ให้ใช้ **Merkle root** ของ `transactions` (จากฟังก์ชัน `build_merkle_tree` ในหัวข้อ 104.6)
   เป็นส่วนหนึ่งของข้อมูลที่ hash แทนการ hash ข้อความทั้งหมดตรง ๆ — hint: `calculate_hash` ต้องเรียก
   `build_merkle_tree(&self.transactions)` แล้วดึง Merkle root (element เดียวใน level สุดท้าย) มาใส่ใน
   `BlockContentForHashing` แทน field `data`

3. **(ยาก)** เขียนฟังก์ชัน `sign_transaction(signing_key: &SigningKey, tx: &Transaction) -> Signature` และ
   `verify_transaction(verifying_key: &VerifyingKey, tx: &Transaction, signature: &Signature) -> bool` แล้วผสาน
   เข้ากับโครงสร้าง `Block`/`Blockchain` จากข้อ 2: ทำให้ `add_block` รับ `Vec<(Transaction, Signature,
   VerifyingKey)>` เข้ามา และ**ปฏิเสธ** transaction ใดก็ตามที่ signature ไม่ถูกต้อง (ไม่ใส่เข้า block เลย พร้อม
   print คำเตือนออกมา) ก่อนคำนวณ Merkle root และ mine block — hint: ทำเหมือน `execute_reset` ในหัวข้อ 104.9 ที่
   ตรวจสอบสิทธิ์ก่อนแก้ state เสมอ ในที่นี้คือ "ตรวจสอบ signature ก่อนรวม transaction เข้า block เสมอ"

4. **(ยาก / ประยุกต์ใช้งานจริง)** ต่อยอด counter contract จากหัวข้อ 104.9: เพิ่ม `ExecuteMsg::IncrementBy { amount:
   u64 }` ที่เพิ่มค่าตัวนับตามจำนวนที่ระบุ (ไม่ใช่แค่ +1) โดยต้องใช้ `checked_add` (ไม่ใช่ operator `+` ตรง ๆ)
   เพื่อป้องกัน integer overflow แบบที่หัวข้อ 104.11 เตือนไว้ — ถ้า overflow เกิดขึ้น ให้คืน
   `Err(ContractError::Std(...))` ที่มีข้อความอธิบายชัดเจน แทนการปล่อยให้ wrap around แบบเงียบ ๆ แล้วเขียน unit
   test เพิ่มอย่างน้อย 2 เคส: (ก) `IncrementBy` ทำงานถูกต้องในกรณีปกติ (ข) `IncrementBy` คืน error เมื่อค่าที่ได้
   จะเกิน `i64::MAX` — hint: `i64::checked_add` คืน `Option<i64>`, ใช้ `.ok_or_else(|| ...)` แปลงเป็น
   `Result` ตามที่ Part 12/30 สอนไว้เรื่อง `Option`↔`Result` conversion

## สรุป

บทนี้เริ่มจากคำถามที่ตัดคำโฆษณาทั้งหมดออก: **blockchain คือโครงสร้างข้อมูลอะไรกันแน่** — และตอบด้วยการสร้าง
`Block`/`Blockchain` จริงที่ hash ด้วย SHA-256 จริงผ่าน `sha2` ต่อยอดจาก serde ที่ Part 57 สอนไว้ พิสูจน์คุณสมบัติ
tamper-evidence ด้วยการรันโค้ดจริงที่แสดงให้เห็นว่าการแก้ไขข้อมูลเก่าทำให้ chain invalid ทันที จากนั้นต่อยอดด้วย
Proof of Work ที่วัดเวลาจริงตามแนวทาง Part 54 แสดงให้เห็นการเติบโตแบบ exponential ของความยากในการขุดเมื่อ
difficulty เพิ่มขึ้น อธิบายเหตุผลเชิงเทคนิคที่ทำให้ Rust ถูกเลือกใช้จริงในงาน blockchain infrastructure
(Solana, Substrate) จากทั้งมุม performance และ memory safety ในโค้ดที่จัดการมูลค่าทางการเงิน

จากนั้นบทนี้ implement digital signature จริงด้วย `ed25519-dalek` เชื่อมกับหลักการ RS256 asymmetric signing ที่
Part 74 สอนไว้ตรง ๆ และสร้าง Merkle tree/Merkle proof จริงที่พิสูจน์ข้อมูลในชุดใหญ่ได้โดยไม่ต้องมีข้อมูลทั้งหมด
ก่อนเข้าสู่หัวใจของ smart contract: **determinism** — เหตุผลที่ floating point (ย้อนกลับไปที่ Part 3), เวลาจริง,
และ randomness ทั่วไป ใช้ในบริบทที่ทุก node ต้องเห็นผลลัพธ์เดียวกันเป๊ะไม่ได้ ปิดท้ายด้วยการเขียนและทดสอบ CosmWasm
smart contract จริงแบบ local ทั้งหมด (4 unit test ผ่านจริง, compile เป็น `.wasm` จริงสำเร็จ) พร้อมความชัดเจนว่า
อะไรถูกพิสูจน์แล้วจริงและอะไรยังไม่ถูกพิสูจน์ และเชื่อมโยง WASM execution model กลับไปยัง Part 86 ให้เห็นว่าทักษะ
จาก Module 5 ถูกใช้ตรงในบริบทใหม่นี้จริง ๆ

สิ่งสำคัญที่สุดที่ควรพกติดตัวไปจากบทนี้คือ**ขอบเขตของความรู้**: นี่คือความรู้กว้างระดับตระหนักรู้เกี่ยวกับสาขาที่
ใหญ่และเปลี่ยนเร็วมาก ไม่ใช่ทางลัดสู่ production — และความปลอดภัยในบริบทนี้มีความเสี่ยงสูงกว่าปกติมาก เพราะ bug
ที่นี่อาจแปลว่าความเสียหายทางการเงินที่แก้ไขย้อนหลังไม่ได้จริง (เชื่อมกับ Part 100) Part ถัดไปจะเปลี่ยนโฟกัสไปที่
การมีส่วนร่วมในโปรเจกต์ Rust แบบ open source จริง — ทักษะที่จำเป็นไม่ว่าคุณจะไปทำงานในสาขาใดของ Rust ก็ตาม
รวมถึงสาขา blockchain ที่พูดถึงในบทนี้ด้วย เพราะโปรเจกต์อย่าง Substrate, Solana validator client, และ CosmWasm
เองล้วนเป็น open source ที่เปิดรับ contribution จริงทั้งสิ้น

---

**Part ก่อนหน้า:** [Game Development ด้วย Bevy Engine เบื้องต้น](part-103-bevy-game-dev.md) | **Part ถัดไป:** [Contributing to Open Source Rust Projects](part-105-open-source-contributing.md)
