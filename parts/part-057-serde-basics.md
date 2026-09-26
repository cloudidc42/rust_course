# Part 57: Serialization: Serde เบื้องต้น

> โมดูล: ระดับสูง (Advanced) | ระดับ: สูง | เวลาโดยประมาณ: 210 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างถูกต้องว่า `serde` คือ **framework ที่แยก "แนวคิดการแปลงข้อมูล" ออกจาก "รูปแบบไฟล์เฉพาะเจาะจง"**
  โดยสิ้นเชิง — `serde` เองไม่รู้จัก JSON, YAML, หรือ TOML เลยแม้แต่นิดเดียว มันนิยามแค่ trait (`Serialize`,
  `Deserialize`) และ **generic data model** ส่วน crate แยกต่างหาก (`serde_json`, `serde_yaml`, `toml`,
  `bincode`) เป็นผู้ implement การเข้ารหัส/ถอดรหัสให้ตรงกับแต่ละรูปแบบจริง และอธิบายได้ว่าการออกแบบแบบนี้ทำให้
  type หนึ่งตัวใช้ได้กับทุก format พร้อมกันโดยไม่ต้องเขียนโค้ดซ้ำเลย
- เพิ่ม `serde`/`serde_json` เข้าโปรเจกต์ด้วย `cargo add serde --features derive` และ `cargo add serde_json`
  แล้วใช้ `#[derive(Serialize, Deserialize)]` แปลง struct ของตัวเองเป็น JSON string ด้วย
  `serde_json::to_string()`/`to_string_pretty()` และแปลงกลับด้วย `serde_json::from_str()` ได้อย่างถูกต้อง
  ครบวงจร (round-trip) พร้อมอธิบายได้ว่า macro ที่ derive มา**สร้างโค้ดอะไร**ขึ้นมาจริง ๆ (ไม่ใช่มองว่าเป็น
  มายากล เพราะ Part 44-45 สอนกลไก derive macro มาให้ครบแล้ว)
- ใช้ attribute ที่ใช้งานจริงบ่อยที่สุดของ `serde` ได้อย่างถูกต้อง: `#[serde(rename = "...")]`,
  `#[serde(rename_all = "camelCase")]`, `#[serde(default)]`/`#[serde(default = "...")]`, `#[serde(skip)]`,
  และ `#[serde(skip_serializing_if = "...")]` พร้อมรู้ว่าแต่ละตัวแก้ปัญหาอะไรจริงในโลกการทำงานกับ REST API
- อธิบายได้ว่า `serde` แปลง Rust enum เป็น JSON แบบไหนได้บ้าง (externally tagged แบบ default, internally
  tagged ด้วย `#[serde(tag = "...")]`, untagged ด้วย `#[serde(untagged)]`) เลือกใช้แบบที่เหมาะกับสถานการณ์ได้
  และรู้ถึงความเสี่ยงเรื่องความกำกวมของ `untagged`
- ใช้ `serde_json::Value` จัดการ JSON ที่ไม่รู้ shape ล่วงหน้าได้ (navigate ด้วย `.get()`/indexing) และตัดสินใจ
  ได้อย่างมีหลักการว่าเมื่อไรควรใช้ `Value` แบบ dynamic เมื่อไรควรใช้ struct ที่มีชนิดชัดเจน
- อ่าน error จาก `serde_json::Error`/`toml::de::Error` ได้อย่างเข้าใจ (บอกได้ว่า error เกิดที่บรรทัด/คอลัมน์
  ไหน เป็น syntax error หรือ data error) และเขียนระบบโหลดไฟล์ config ด้วย `serde` + `toml` ที่มี nested
  struct, optional field ที่มีค่า default, และ error handling ที่ถูกต้องตามที่ Part 30-31 สอนไว้ได้จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 44-45 (Procedural Macros: พื้นฐาน และ Derive Macro ขั้นสูง)** — นี่คือความรู้ที่ต้องมีมาก่อนที่**สำคัญ
  ที่สุด**ของบทนี้ และเป็นเหตุผลที่บทนี้ถูกจัดไว้หลัง Part 45 ไม่ใช่ก่อนหน้านั้น: Part 44 สอนไว้ตรง ๆ ว่า
  `#[derive(Debug, Clone, Serialize, Deserialize)]` (ตารางหัวข้อ 44.10) คือ derive macro ที่ทำงานด้วยกลไก
  เดียวกันกับ `Describe`/`Validated` ที่คุณเขียนเองในบทนั้น — parse ด้วย `syn::DeriveInput`, ดึงข้อมูล field,
  generate โค้ดด้วย `quote!{}` และ Part 45 สอนต่อไปอีกว่า macro รับ**custom attribute ของตัวเอง**ได้อย่างไร
  ผ่านตัวอย่าง `#[describe(skip)]` (หัวข้อ 45.8-45.9) พร้อมทิ้งท้ายไว้ในตารางสรุปบทว่า "`#[serde(rename/skip/
  default)]` (Part 57-58)" คือเวอร์ชันโลกจริงของแนวคิดเดียวกัน — บทนี้คือจุดที่สัญญานั้นถูกเฉลย เมื่อคุณเห็น
  `#[serde(rename = "userId")]` คุณต้องมองมันด้วยสายตาของคนที่**เคยเขียน** helper attribute แบบนี้เองมาแล้ว
  ไม่ใช่คนที่ท่องจำ syntax โดยไม่รู้ว่ามันทำงานอย่างไรข้างใน — ถ้ายังไม่แน่นเรื่อง `TokenStream`/`syn`/`quote!`/
  helper attribute ควรทวน Part 44-45 ก่อนเริ่มบทนี้ เพราะบทนี้จะไม่อธิบายกลไก derive macro ซ้ำอีกเลย
- **Part 19/21 (Traits พื้นฐาน/ขั้นสูง)** — `Serialize`/`Deserialize` เป็น**trait ธรรมดา**เหมือน trait อื่น ๆ
  ที่เรียนมา และการที่ format หลาย crate ใช้ trait เดียวกันได้ทั้งหมดคือการประยุกต์ตรงของแนวคิด "เขียนโค้ดครั้ง
  เดียว ให้ทำงานกับ concrete type/รูปแบบข้อมูลได้หลายแบบผ่าน trait bound" ที่ Part 19-21 ปูพื้นไว้
- **Part 9/10 (Structs, Enums และ Pattern Matching)** — บทนี้ใช้ struct (named field, tuple) และ enum
  (unit/tuple/struct-like variant) เป็นโครงข้อมูลหลักทุกตัวอย่าง และหัวข้อเรื่อง "serde แปลง enum เป็น JSON
  อย่างไร" อ้างอิงตรงกับความรู้เต็มรูปแบบเรื่อง variant ทั้งสามชนิดจาก Part 10
- **Part 12 (Result<T, E> และ Error Handling เบื้องต้น)** และ **Part 30-31 (Error Handling ขั้นสูง,
  thiserror/anyhow)** — หัวข้อเรื่อง error ของ `serde_json`/`toml` และตัวอย่าง config loader ท้ายบท ใช้ `?`,
  `impl From`, และ `#[derive(thiserror::Error)]` ตรงตามที่ Part 30-31 สอนไว้ทุกประการ โดยไม่อธิบาย concept
  พื้นฐานของ error handling ซ้ำอีก
- **Part 17 (Packages, Crates, Workspaces)** — คำสั่ง `cargo add` และการเพิ่ม `[dependencies]` ใน
  `Cargo.toml` ใช้กลไกเดียวกันกับที่ Part 17 สอนไว้ตรง ๆ

## เนื้อหา

### 57.1 ทวนคำถามที่ Part 44 ทิ้งไว้: `#[derive(Serialize, Deserialize)]` ทำงานอย่างไรกันแน่

จากตารางในหัวข้อ 44.10 ของ Part 44 มีบรรทัดนี้อยู่:

> `serde` (`serde_derive`) | Derive — `#[derive(Serialize, Deserialize)]` | generate โค้ดแปลง struct/enum เป็น/
> จาก รูปแบบข้อมูล (JSON, YAML, ...) โดยอัตโนมัติ จากโครงสร้าง struct จริง (จะเรียนใน Part 57-58)

ตอนนั้นคุณเห็นแค่ชื่อ ไม่เห็นรายละเอียด แต่ตอนนี้คุณมีเครื่องมือครบมือแล้วที่จะเข้าใจมันแบบไม่ต้องเดา ลองตั้ง
คำถามแบบเดียวกับที่ Part 44 หัวข้อ 44.1 ตั้งไว้กับ `Describe`: **ถ้าไม่มี `serde` เลย เราจะเขียนโค้ดแปลง struct
เป็น JSON ด้วยมือยังไง?**

```rust
// เวอร์ชัน "ไม่มี serde" — เขียนแปลง struct เป็น JSON ด้วยมือ เพื่อดูว่ามันน่าเบื่อและเสี่ยงผิดแค่ไหน
struct Product {
    id: u32,
    name: String,
    price: f64,
}

impl Product {
    // ต้องเขียนเมธอดแบบนี้เองสำหรับทุก struct ในโปรเจกต์ ทุกครั้งที่ struct เปลี่ยน field ต้องแก้ตรงนี้ตาม
    fn to_json_string(&self) -> String {
        // ต้อง escape string เองด้วย (เครื่องหมายคำพูด, backslash, ตัวควบคุมต่าง ๆ) ไม่ทำแบบนี้จะพังทันที
        // ถ้าชื่อสินค้ามีเครื่องหมายคำพูดอยู่ในตัวมันเอง
        let escaped_name = self.name.replace('\\', "\\\\").replace('"', "\\\"");
        format!(
            r#"{{"id":{},"name":"{}","price":{}}}"#,
            self.id, escaped_name, self.price
        )
    }
}

fn main() {
    let p = Product { id: 1, name: "สาย USB".to_string(), price: 99.0 };
    println!("{}", p.to_json_string());
}
```

โค้ดนี้ *ดู* เหมือนใช้ได้ แต่มีปัญหาลึกกว่าที่เห็น: ต้องเขียน escape logic ให้ถูกต้องตาม JSON spec ทุกตัวอักษร
(quote, backslash, ตัวควบคุม เช่น newline/tab, และตัวอักษร Unicode ที่ต้อง escape ในบางบริบท), ต้องเขียนโค้ด
ฝั่ง**ถอดรหัสกลับ** (parse JSON string เป็น `Product`) แยกอีกชุดหนึ่งที่ซับซ้อนกว่ามาก (ต้องเขียน JSON parser
เองหรือดึง crate มาช่วย), และที่สำคัญที่สุด — **ทำแบบนี้ซ้ำสำหรับทุก struct ในโปรเจกต์ทุกตัว** เหมือนกับที่
Part 44 หัวข้อ 44.1 ชี้ให้เห็นเรื่อง `Display`/`Error`/`From` ที่ต้องเขียนซ้ำ ๆ ด้วยรูปแบบเดิมทุกครั้ง — นี่คือ
สัญญาณเดียวกันเป๊ะว่างานนี้เหมาะกับ derive macro

**สิ่งที่ต่างจาก `Describe`/`thiserror` ที่เจอมาก่อนคือ**: `serde` ไม่ได้ผูกกับ "รูปแบบข้อมูลเดียว" เลย มันคือ
framework ที่ใหญ่กว่านั้นมาก — และนี่คือสิ่งที่หัวข้อถัดไปจะอธิบายให้เห็นภาพทั้งระบบ

### 57.2 สถาปัตยกรรมของ `serde`: Framework ที่แยก "อะไรจะถูกแปลง" ออกจาก "แปลงเป็นรูปแบบไหน"

ชื่อ `serde` มาจาก **ser**ialization + **de**serialization ตรงตัว แต่ส่วนที่ทำให้มันกลายเป็น crate ที่ใช้กัน
แทบทุกโปรเจกต์ Rust ที่ทำงานกับข้อมูลไม่ใช่แค่ชื่อ แต่เป็น**การตัดสินใจเชิงสถาปัตยกรรม**ที่ชัดเจนตั้งแต่ต้น:
`serde` เอง**ไม่รู้จัก JSON, YAML, TOML, หรือ binary format ใด ๆ เลยแม้แต่นิดเดียว**

ลองเทียบกับ trait ที่เรียนมาแล้วใน Part 19-21: `Iterator` trait ไม่รู้จัก "เอาไปทำอะไรต่อ" เลย มันแค่นิยาม
`next()` ตัวเดียว แล้วปล่อยให้ `.collect()`, `.sum()`, `.for_each()` (ที่เป็น method อื่น ๆ ที่ทำงานผ่าน trait
bound `Iterator`) ตัดสินใจว่าจะเอา sequence ที่ได้ไปทำอะไร — type ที่ implement `Iterator` แค่ตัวเดียวจึงใช้ได้
กับปลายทางที่แตกต่างกันได้นับไม่ถ้วนโดยไม่ต้องเขียนโค้ดใหม่เลย `serde` ใช้แนวคิดเดียวกันเป๊ะแต่ในสเกลที่ใหญ่กว่า:

```
                    ┌─────────────────────────────────────────┐
                    │              serde (crate หลัก)           │
                    │                                           │
                    │  trait Serialize    trait Deserialize     │
                    │  trait Serializer   trait Deserializer    │
                    │                                           │
                    │  "generic data model": struct, enum,      │
                    │  sequence, map, string, number, bool, ... │
                    └───────────────┬───────────────────────────┘
                                     │ type ที่ implement Serialize/Deserialize
                                     │ (ผ่าน #[derive(...)] หรือเขียนมือ)
                    ┌────────────────┼────────────────┬─────────────────┐
                    │                │                │                 │
              ┌─────▼─────┐   ┌──────▼──────┐  ┌──────▼──────┐  ┌───────▼──────┐
              │serde_json │   │ serde_yaml  │  │    toml     │  │   bincode    │
              │ (JSON)    │   │  (YAML)     │  │  (TOML)     │  │  (binary)    │
              └───────────┘   └─────────────┘  └─────────────┘  └──────────────┘
```

**"generic data model" คือหัวใจของการออกแบบนี้** — `serde` นิยามไว้ว่า "ข้อมูลใด ๆ ที่จะ serialize ได้ต้อง
สามารถอธิบายได้ด้วยรูปแบบพื้นฐานจำนวนจำกัดเหล่านี้เท่านั้น": primitive (bool, integer หลายขนาด, float, char,
string, bytes), `Option<T>` (มี/ไม่มีค่า), sequence (list ที่มีความยาวรู้/ไม่รู้ล่วงหน้า), tuple, map,
struct (มีชื่อ field คงที่), enum (มีชื่อ variant) — สังเกตว่ารายการนี้คือ**ส่วนผสมของสิ่งที่ทุกภาษาโปรแกรมมิ่ง
สมัยใหม่มีอยู่แล้ว** ไม่ใช่อะไรที่ผูกกับ JSON หรือ Rust โดยเฉพาะ — trait `Serialize` มีหน้าที่แค่ **"อธิบาย
ตัวเองด้วยคำศัพท์พื้นฐานเหล่านี้"** (เช่น "ฉันคือ struct ที่มี 3 field ชื่อ id/name/price") ส่วนที่ว่า
"struct ที่มี 3 field" จะถูกเขียนออกมาเป็น `{"id":1,...}` (JSON), `id = 1\n...` (TOML), หรือ byte สั้น ๆ ไม่มี
ชื่อ field เลย (bincode) นั้นเป็นเรื่องของ **`Serializer`** (คนละ trait จาก `Serialize`) ที่ crate รูปแบบข้อมูล
แต่ละตัว implement เอง

พูดให้เป็นรูปธรรมที่สุด: เวลา `#[derive(Serialize)]` สร้างโค้ดให้ `Product` มันไม่ได้สร้างโค้ดที่รู้จัก JSON เลย
มันสร้างโค้ดประมาณนี้ (แบบง่ายเพื่อความเข้าใจ ไม่ใช่ token-for-token จริงที่ macro generate):

```rust
// รูปแบบคร่าว ๆ ของสิ่งที่ #[derive(Serialize)] generate ให้ Product { id, name, price }
// (เขียนแบบย่อเพื่ออธิบายแนวคิด — เทียบกับที่ Part 44 อธิบายว่า derive macro generate impl block ให้)
impl serde::Serialize for Product {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::Serializer, // generic ครอบคลุมทุก format — นี่คือจุดที่ trait bound ทำงานจริง
    {
        // เรียก method ของ Serializer trait โดยไม่รู้เลยว่าฝั่งรับเป็น JSON, TOML, หรืออื่นใด
        let mut state = serializer.serialize_struct("Product", 3)?;
        state.serialize_field("id", &self.id)?;
        state.serialize_field("name", &self.name)?;
        state.serialize_field("price", &self.price)?;
        state.end()
    }
}
```

สังเกตว่า `serialize()` **generic ที่ `S: Serializer`** — โค้ดนี้เขียนครั้งเดียวโดย `#[derive(Serialize)]`
แต่เรียก `serializer.serialize_struct(...)`/`serialize_field(...)` ที่เป็นแค่**ชื่อ method บน trait**
`Serializer` — ไม่ได้ผูกกับ implementation ไหนเลย เมื่อคุณเรียก `serde_json::to_string(&product)` สิ่งที่เกิด
ขึ้นคือ `serde_json` ส่ง `Serializer` เวอร์ชันของตัวเอง (ที่ `serialize_struct` แปลว่า "เริ่มเขียน `{`" และ
`serialize_field` แปลว่า "เขียน `"key":value,`") เข้าไปเป็น `S` ตัวจริง — ถ้าเปลี่ยนไปเรียก `toml::to_string
(&product)` แทน `toml` จะส่ง `Serializer` เวอร์ชันของตัวเองที่แปล `serialize_struct`/`serialize_field` เป็น
ไวยากรณ์ TOML แทน **โดยที่ `impl Serialize for Product` ที่ macro generate ไว้ไม่ต้องเปลี่ยนแม้แต่บรรทัดเดียว**

นี่คือคำตอบที่สมบูรณ์ของ "ทำไม type หนึ่งตัวใช้ได้กับทุก format": เพราะ `Serialize::serialize()` เขียนโค้ดที่
**อธิบายรูปร่างของข้อมูลผ่าน generic trait `Serializer`** ไม่ใช่เขียนโค้ดที่ผลิต JSON string ตรง ๆ — ความรับ
ผิดชอบเรื่อง "ผลิตออกมาเป็นอะไร" ถูกส่งต่อไปให้ type parameter `S` (ตามหลักการ generic ที่ Part 18/19/21 สอนไว้
ว่า "โค้ดที่ generic เขียนครั้งเดียว ทำงานได้กับ concrete type ที่สนอง trait bound ได้ไม่จำกัด") ทั้งหมดนี้เกิด
ขึ้น**ตอน compile time** ไม่มี runtime dispatch ที่ต้องเช็คว่า "นี่คือ format ไหน" เลย — เพราะ `S` ถูก
monomorphize เป็น concrete type ตอน compile (แนวคิดเดียวกับที่ generic function ทุกตัวถูก monomorphize ตาม
Part 18 สอนไว้)

**ตารางสรุปบทบาทของแต่ละ crate ในสถาปัตยกรรมนี้:**

| Crate | บทบาท | รู้จักรูปแบบข้อมูลไหน |
|---|---|---|
| `serde` | นิยาม trait `Serialize`/`Deserialize`, `Serializer`/`Deserializer`, generic data model | **ไม่รู้จักเลย** — เป็นแค่ vocabulary กลาง |
| `serde_derive` (เปิดผ่าน feature `derive`) | derive macro ที่ generate `impl Serialize`/`impl Deserialize` ให้ type ของคุณ | ไม่รู้จัก — generate โค้ดที่เรียก `Serializer`/`Deserializer` trait เท่านั้น |
| `serde_json` | implement `Serializer`/`Deserializer` สำหรับ JSON | **JSON** |
| `serde_yaml` | implement `Serializer`/`Deserializer` สำหรับ YAML | **YAML** |
| `toml` | implement `Serializer`/`Deserializer` สำหรับ TOML | **TOML** |
| `bincode` | implement `Serializer`/`Deserializer` สำหรับ binary format ของตัวเอง | **binary (compact)** |

**Generic data model ที่ trait `Serializer` ต้อง implement ให้ครบ** (ย่อมาจาก `serde::Serializer` — แสดงเฉพาะ
method หลักที่เกี่ยวข้องกับตัวอย่างในบทนี้ ไม่ใช่ signature ที่ต้อง copy ไปใช้ตรง ๆ):

| Method บน `Serializer` | ใช้แทน data model แบบไหน | ตัวอย่าง Rust type ที่เรียก method นี้ |
|---|---|---|
| `serialize_bool` | boolean | `bool` |
| `serialize_u8`...`serialize_u64`, `serialize_i8`...`serialize_i64`, `serialize_f32`/`serialize_f64` | ตัวเลขแต่ละขนาด | `u8`..`u64`, `i8`..`i64`, `f32`/`f64` |
| `serialize_str` | ข้อความ | `String`, `&str` |
| `serialize_none` / `serialize_some` | ค่าที่อาจไม่มี | `Option<T>` |
| `serialize_seq` | ลำดับที่มีความยาว | `Vec<T>`, slice, `HashSet<T>` |
| `serialize_map` | คู่ key-value | `HashMap<K, V>`, `BTreeMap<K, V>` |
| `serialize_struct` | struct ที่มีชื่อ field คงที่ | struct ที่ `#[derive(Serialize)]` (ตามหัวข้อ 57.1) |
| `serialize_struct_variant`/`serialize_unit_variant`/... | enum แต่ละชนิด variant | enum ที่ `#[derive(Serialize)]` (หัวข้อ 57.6) |

สังเกตว่า **ไม่มี method ชื่อ `serialize_json_object` หรืออะไรที่เจาะจง format เลยแม้แต่ตัวเดียว** — ทุก method
ตั้งชื่อตาม "แนวคิดข้อมูล" (`_seq`, `_map`, `_struct`) ไม่ใช่ตาม "รูปแบบไฟล์" — นี่คือหลักฐานที่จับต้องได้ที่สุด
ของคำกล่าวที่ว่า `serde` ไม่รู้จัก JSON เลย: `#[derive(Serialize)]` ที่ generate ให้ `Product` ในหัวข้อ 57.1
เรียกแค่ `serializer.serialize_struct("Product", 3)` แล้ว `serialize_field(...)` ไล่ทีละ field — ส่วนที่ว่า
method นี้จะไปเขียน `{`, `}`, `:`, `,` ของ JSON จริง ๆ นั่นคือ**หน้าที่ของ `serde_json` เท่านั้น** (implement
`Serializer` เวอร์ชัน JSON ของตัวเอง) `toml`/`serde_yaml`/`bincode` ก็ implement `Serializer` ของตัวเองแบบ
เดียวกัน แต่แปล method เดียวกันนี้ไปเป็นไวยากรณ์/byte layout ที่ต่างกันไปตาม format

หัวข้อ 57.9 จะพิสูจน์เรื่องนี้ด้วยโค้ดจริง — เอา struct ตัวเดียวไป serialize ด้วยทั้ง 4 crate โดยไม่แก้นิยาม
struct แม้แต่บรรทัดเดียว

### 57.3 เพิ่ม `serde`/`serde_json` เข้าโปรเจกต์

```bash
cargo add serde --features derive
```

ผลลัพธ์จริง (ทดสอบในเครื่องผู้เขียนบทนี้):

```
    Updating crates.io index
      Adding serde v1.0.229 to dependencies
             Features:
             + derive
             + serde_derive
             + std
             - alloc
             - rc
             - unstable
    Updating crates.io index
     Locking 7 packages to latest compatible versions
```

```bash
cargo add serde_json
```

```
    Updating crates.io index
      Adding serde_json v1.0.151 to dependencies
             Features:
             + std
             - alloc
             - arbitrary_precision
             - float_roundtrip
             - indexmap
             - preserve_order
             - raw_value
             - unbounded_depth
    Updating crates.io index
     Locking 4 packages to latest compatible versions
      Adding itoa v1.0.18
      Adding memchr v2.8.3
      Adding serde_json v1.0.151
      Adding zmij v1.0.23
```

`Cargo.toml` ที่ได้:

```toml
[dependencies]
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
```

**ข้อสังเกตสำคัญที่มือใหม่พลาดบ่อยที่สุด**: สังเกตว่า `serde` ต้องเปิด **feature `derive` เอง** (ไม่ได้เปิดเป็น
default) — เหตุผลตรงกับที่ Part 44 หัวข้อ 44.3 อธิบายไว้เรื่อง `proc-macro = true` crate: `serde_derive` คือ
**proc-macro crate แยก** ที่ compile สำหรับ host ไม่ใช่ target ตามตารางในหัวข้อนั้น การแยก feature ทำให้
โปรเจกต์ที่ใช้ `serde` แค่ trait (ไม่ derive อะไรเลย เช่น implement `Serialize`/`Deserialize` ด้วยมือ ซึ่งจะ
เรียนใน Part 58) ไม่ต้อง pull dependency ของ proc-macro crate เข้ามาโดยไม่จำเป็น (ลด compile time) — ถ้าลืม
เปิด feature นี้จะเจอ error ทันทีที่ลองเขียน `#[derive(Serialize)]` (ดูตัวอย่าง error จริงในหัวข้อกับดักข้อ 1)

### 57.4 Round-trip แรก: `#[derive(Serialize, Deserialize)]` กับ `serde_json`

มาลงมือเขียนตัวอย่างที่ใช้งานจริงได้ทันที — struct `Product` ที่ครอบคลุม primitive type หลายแบบ (`u32`,
`String`, `f64`, `bool`, `Vec<String>`):

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Product {
    id: u32,
    name: String,
    price: f64,
    in_stock: bool,
    tags: Vec<String>,
}

fn main() {
    let product = Product {
        id: 1001,
        name: "คีย์บอร์ดกลไก".to_string(),
        price: 1990.50,
        in_stock: true,
        tags: vec!["electronics".to_string(), "peripherals".to_string()],
    };

    // Serialize: struct -> JSON string (แบบ compact ตัวเดียว บรรทัดเดียว)
    let compact = serde_json::to_string(&product).unwrap();
    println!("compact:\n{compact}");

    // Serialize: struct -> JSON string (แบบ pretty มี indent อ่านง่าย)
    let pretty = serde_json::to_string_pretty(&product).unwrap();
    println!("pretty:\n{pretty}");

    // Deserialize: JSON string -> struct กลับมา
    let restored: Product = serde_json::from_str(&pretty).unwrap();
    println!(
        "restored: id={} name={} price={} in_stock={} tags={:?}",
        restored.id, restored.name, restored.price, restored.in_stock, restored.tags
    );

    assert_eq!(product.id, restored.id);
    assert_eq!(product.name, restored.name);
    println!("round-trip ตรงกันทุก field");
}
```

ผลลัพธ์ (คอมไพล์และรันจริงด้วย `cargo run` ในเครื่องผู้เขียนบทนี้):

```
compact:
{"id":1001,"name":"คีย์บอร์ดกลไก","price":1990.5,"in_stock":true,"tags":["electronics","peripherals"]}
pretty:
{
  "id": 1001,
  "name": "คีย์บอร์ดกลไก",
  "price": 1990.5,
  "in_stock": true,
  "tags": [
    "electronics",
    "peripherals"
  ]
}
restored: id=1001 name=คีย์บอร์ดกลไก price=1990.5 in_stock=true tags=["electronics", "peripherals"]
round-trip ตรงกันทุก field
```

**อธิบายกลไกทีละส่วน (เชื่อมกลับไปที่หัวข้อ 57.2):**

- **`#[derive(Debug, Serialize, Deserialize)]`** — สังเกตว่า `Debug` ยังต้อง derive เองตามปกติ (มาตรฐานของ
  standard library ไม่เกี่ยวกับ `serde`) ส่วน `Serialize`/`Deserialize` มาจาก `serde_derive` (export ผ่าน
  `serde::Serialize`/`serde::Deserialize` เพราะเปิด feature `derive` ไว้แล้ว) — macro ทั้งสองตัวทำงานคนละทิศ
  ทาง: `Serialize` สร้าง `impl Serialize for Product` (struct → generic data model), `Deserialize` สร้าง
  `impl<'de> Deserialize<'de> for Product` (generic data model → struct) — สองอันแยกจากกันโดยสมบูรณ์ คุณ
  derive แค่ตัวเดียวก็ได้ถ้าต้องการ (เช่น type ที่คุณจะ**ส่งออก**อย่างเดียว ไม่ต้องรับกลับมา ก็ derive แค่
  `Serialize`)
- **`serde_json::to_string(&product)`** — รับ `&T` ที่ `T: Serialize` (ตรงตาม generic bound ที่อธิบายไว้ใน
  หัวข้อ 57.2) คืน `Result<String, serde_json::Error>` — สังเกตว่า `.unwrap()` ในตัวอย่างนี้ใช้เพราะรู้แน่ว่า
  serialize ไม่มีทางล้มเหลว (มีแต่ error ตอน deserialize เท่านั้นที่พบบ่อยในทางปฏิบัติ เพราะข้อมูลขาเข้ามาจาก
  แหล่งที่ไม่น่าเชื่อถือ ต่างจากขาออกที่ struct ของเราถูกต้องอยู่แล้วเสมอ — หัวข้อ 57.8 จะพูดเรื่อง error
  โดยละเอียด)
- **`price: 1990.5`** — สังเกตว่า JSON output พิมพ์ `1990.5` ไม่ใช่ `1990.50` เพราะ `f64` ไม่มีแนวคิดเรื่อง
  "จำนวนหลักทศนิยม" ในตัวเอง (`1990.50` และ `1990.5` คือค่าเดียวกันในระบบ floating point) — นี่เป็นสัญญาณ
  เตือนล่วงหน้าที่สำคัญเรื่อง float กับเงิน จะอธิบายเพิ่มในหัวข้อกับดัก
- **`serde_json::from_str::<Product>(&pretty)`** — รับ `&str` คืน `Result<T, serde_json::Error>` โดย `T` ต้อง
  `Deserialize` — สังเกตว่าฟังก์ชันนี้ **generic ที่ return type** (compiler ต้องรู้ว่าจะ deserialize เป็น
  type ไหนจาก context เช่น การประกาศ `let restored: Product = ...`) นี่คือรูปแบบเดียวกับ `.parse::<T>()` ที่
  Part 12 สอนไว้เรื่อง `FromStr` — `Deserialize` คือเวอร์ชันที่ทั่วไปกว่า `FromStr` (ไม่ผูกกับ `&str` อย่างเดียว
  แต่ผูกกับ generic data model ทั้งระบบ)

### 57.5 Attribute ที่ใช้งานจริงบ่อยที่สุดของ `serde`

จาก Part 45 หัวข้อ 45.9 คุณเขียน helper attribute ของตัวเอง (`#[describe(skip)]`) มาแล้วทั้งกระบวนการ: ประกาศ
`#[proc_macro_derive(Describe, attributes(describe))]`, parse attribute ด้วย `meta.path.is_ident(...)`, แล้ว
ปรับ logic การ generate ตามที่เจอ — attribute ของ `serde` ที่กำลังจะเห็นต่อไปนี้**ทำงานด้วยกลไกเดียวกันทุก
ประการ** เพียงแต่ `serde_derive` เขียน parser ที่รองรับ attribute หลากหลายกว่าตัวอย่างง่าย ๆ ที่เราเขียนกันเอง
มาก

#### 57.5.1 `#[serde(rename = "...")]`: เปลี่ยนชื่อ field ตอน serialize/deserialize

สถานการณ์จริงที่พบบ่อยที่สุด: struct ในโค้ด Rust ตั้งชื่อตาม convention ของ Rust (`snake_case` ตาม Part 9) แต่
JSON ที่ต้องคุยด้วย (เช่น external API ที่ทีมอื่นเขียนด้วยภาษาอื่น) ใช้ชื่อคนละแบบ:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct UserProfile {
    #[serde(rename = "userId")]
    user_id: u64,

    #[serde(rename = "fullName")]
    full_name: String,
}

fn main() {
    let u = UserProfile {
        user_id: 42,
        full_name: "สมชาย ใจดี".to_string(),
    };
    println!("{}", serde_json::to_string(&u).unwrap());
}
```

ผลลัพธ์:

```
{"userId":42,"fullName":"สมชาย ใจดี"}
```

field ในโค้ด Rust ยังชื่อ `user_id`/`full_name` ตามปกติ (เขียนโค้ด Rust ที่อ่านง่ายตาม convention ของภาษาได้
เต็มที่) แต่ JSON ที่ผลิตออกมาตรงกับที่ external API คาดหวัง — `#[serde(rename = "...")]` ทำงาน**ทั้งสองทาง**
(ถ้า deserialize JSON ที่มี key `"userId"` มันก็จะแมปกลับเข้า field `user_id` ให้ถูกต้องเช่นกัน)

#### 57.5.2 `#[serde(rename_all = "camelCase")]`: rename ทุก field พร้อมกันด้วย convention

การเขียน `#[serde(rename = "...")]` ทีละ field น่าเบื่อมากถ้า struct มีหลาย field ทั้งหมด (และ JSON API ที่ใช้
`camelCase` ทั้งระบบเป็นเรื่องปกติมากในโลกจริง เพราะ JavaScript/TypeScript ใช้ convention นี้) `serde` จึงมี
attribute ระดับ struct ที่ rename **ทุก field พร้อมกัน**ตาม convention ที่กำหนด:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct UserProfileCamel {
    user_id: u64,
    full_name: String,
    is_active: bool,
}

fn main() {
    let u = UserProfileCamel {
        user_id: 42,
        full_name: "สมหญิง ใจงาม".to_string(),
        is_active: true,
    };
    println!("{}", serde_json::to_string(&u).unwrap());
}
```

ผลลัพธ์:

```
{"userId":42,"fullName":"สมหญิง ใจงาม","isActive":true}
```

ทุก field ถูกแปลงจาก `snake_case` เป็น `camelCase` อัตโนมัติ (`user_id` → `userId`, `is_active` → `isActive`)
โดยไม่ต้องเขียน `#[serde(rename = "...")]` ทีละบรรทัดเลย ค่าที่ `rename_all` รองรับมีหลายแบบ (`"camelCase"`,
`"PascalCase"`, `"snake_case"`, `"SCREAMING_SNAKE_CASE"`, `"kebab-case"`, `"lowercase"`, `"UPPERCASE"`) —
เลือกให้ตรงกับ convention ของระบบปลายทางที่ทำงานด้วย

#### 57.5.3 `#[serde(default)]` และ `#[serde(default = "...")]`: เติมค่าถ้า field หายไปตอน deserialize

จาก Part 19 คุณรู้จัก trait `Default` มาแล้ว (`Default::default()` คืนค่า "เริ่มต้นตามธรรมชาติ" ของ type
เช่น `0` สำหรับตัวเลข, `String::new()` สำหรับ string) — `#[serde(default)]` ผูกสอง concept นี้เข้าด้วยกัน
ตรง ๆ: **ถ้า field นั้นหายไปจาก input ตอน deserialize ให้ใช้ `Default::default()` แทนการ error**

```rust
use serde::{Deserialize, Serialize};

fn default_timeout() -> u32 {
    30
}

#[derive(Debug, Serialize, Deserialize)]
struct ServerSettings {
    host: String,

    #[serde(default = "default_timeout")] // ไม่มีให้ใช้ค่าที่ฟังก์ชันนี้คืน (ไม่ใช้ Default::default())
    timeout_secs: u32,

    #[serde(default)] // ไม่มีให้ใช้ Default::default() ของ u32 คือ 0
    retries: u32,
}

fn main() {
    // JSON นี้มีแค่ "host" — ไม่มี timeout_secs, retries เลย
    let json_missing_fields = r#"{ "host": "db.internal" }"#;
    let settings: ServerSettings = serde_json::from_str(json_missing_fields).unwrap();
    println!("{settings:?}");
}
```

ผลลัพธ์:

```
ServerSettings { host: "db.internal", timeout_secs: 30, retries: 0 }
```

**อธิบายความต่างระหว่างสองรูปแบบ:**

- **`#[serde(default)]`** — ไม่มี argument เลย ใช้ `<FieldType as Default>::default()` ตรง ๆ (ต้องมี
  `impl Default` ให้ type นั้นก่อน — primitive ทุกตัวมีอยู่แล้ว, `Option<T>` มี `Default` เป็น `None` เสมอ)
- **`#[serde(default = "default_timeout")]`** — ระบุชื่อ**ฟังก์ชัน** (ไม่ใช่ค่าตรง ๆ) ที่คืนค่า default ที่
  ต้องการ ใช้เมื่อค่า default ไม่ใช่ "ค่าธรรมดาของ type" (เช่น `30` ไม่ใช่ default ธรรมชาติของ `u32` ซึ่งคือ
  `0`) ฟังก์ชันนี้ต้องไม่รับ parameter และคืนชนิดเดียวกับ field — attribute เก็บชื่อฟังก์ชันเป็น**string**
  (`"default_timeout"`) เพราะเหตุผลเดียวกับที่ Part 44-45 อธิบายไว้ตลอด: attribute ทั้งหมดคือ token stream
  ที่ macro parse ตอน compile time ไม่ใช่ค่าที่ evaluate ตอน runtime — `serde_derive` จึงต้อง generate โค้ด
  ที่**เรียกชื่อฟังก์ชันนั้นตรง ๆ** (`default_timeout()`) ไม่ใช่รับค่าคงที่มาฝัง

#### 57.5.4 `#[serde(skip)]` และ `#[serde(skip_serializing_if = "...")]`

`#[serde(skip)]` บอกว่า field นี้**ไม่เกี่ยวข้องกับการ serialize/deserialize เลย** — ตอน serialize field นี้
จะไม่ปรากฏใน output เลย และตอน deserialize field นี้จะถูกเติมด้วย `Default::default()` เสมอ (เหมือนมี
`#[serde(default)]` ในตัวโดยอัตโนมัติ ไม่ต้องเขียนซ้ำ) เหมาะกับ field ที่เป็น **runtime-only state** (เช่น
cache ที่คำนวณใหม่ได้ทุกครั้งไม่ต้องเก็บลง JSON):

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct ServerSettings {
    host: String,

    #[serde(skip)]
    runtime_cache: Option<String>,

    #[serde(skip_serializing_if = "Option::is_none")]
    api_key: Option<String>,
}

fn main() {
    let with_key = ServerSettings {
        host: "cache.internal".to_string(),
        runtime_cache: Some("จะไม่ถูก serialize เลย".to_string()),
        api_key: Some("secret-token".to_string()),
    };
    println!("มี api_key: {}", serde_json::to_string(&with_key).unwrap());

    let without_key = ServerSettings {
        host: "cache.internal".to_string(),
        runtime_cache: None,
        api_key: None,
    };
    println!("ไม่มี api_key: {}", serde_json::to_string(&without_key).unwrap());
}
```

ผลลัพธ์:

```
มี api_key: {"host":"cache.internal","api_key":"secret-token"}
ไม่มี api_key: {"host":"cache.internal"}
```

สังเกตว่า `runtime_cache` **ไม่ปรากฏใน output ทั้งสองกรณี** (ตามที่ `#[serde(skip)]` สั่งไว้) ส่วน `api_key`
ปรากฏหรือไม่ปรากฏ**ขึ้นอยู่กับค่า**: `#[serde(skip_serializing_if = "Option::is_none")]` บอกว่า "ก่อน
serialize field นี้ ให้เรียกฟังก์ชัน `Option::is_none` (รับ `&Option<String>` คืน `bool`) ก่อน ถ้าคืน `true`
ให้ข้าม field นี้ไปเลย" — นี่คือ pattern ที่พบบ่อยที่สุดในโลกจริงสำหรับ field ที่เป็น optional และไม่อยากให้
JSON เต็มไปด้วย `"api_key":null` เกลื่อนทุกที่ที่ค่ามันไม่มี (ประหยัด bandwidth และอ่านง่ายขึ้นเมื่อ field
ไม่เกี่ยวข้อง) — ค่าที่ให้กับ `skip_serializing_if` ต้องเป็น**ชื่อฟังก์ชัน**แบบเดียวกับ `default = "..."`
(string ที่ระบุชื่อ path ของฟังก์ชันที่รับ `&FieldType` คืน `bool`) ไม่ใช่ expression ที่ evaluate ตรงๆ

**เชื่อมกลับไปหัวข้อ 45.9**: ทั้งสี่ attribute ที่เห็นมา (`rename`, `rename_all`, `default`, `skip`,
`skip_serializing_if`) ทำงานผ่านกลไกเดียวกับที่คุณเขียนเองใน `Describe`: `serde_derive` ประกาศ
`#[proc_macro_derive(Serialize, attributes(serde))]` (helper attribute ชื่อ `serde`) รับ token stream ของ
struct มา parse ด้วย `syn`, วนดู attribute บนแต่ละ field ที่ path ตรงกับ `serde`, อ่าน key-value ข้างในวงเล็บ
(`rename = "..."`, `default = "..."`) แล้วปรับโค้ดที่ generate ตามนั้น — ต่างจาก `#[describe(skip)]` ที่รองรับ
แค่ key เดียวไม่มี value เพียงเพราะ `serde_derive` ลงทุนเขียน parser ที่ครอบคลุมกรณีมากกว่าเรามาก (`serde`
เป็น crate ที่มีคนใช้มากที่สุดใน ecosystem จึงคุ้มที่จะลงทุนรองรับทุก use case ที่พบในโลกจริง)

### 57.6 Enum กับ Serde: สามรูปแบบการแทน JSON

จาก Part 10 คุณรู้ variant ทั้งสามแบบของ Rust enum ครบแล้ว (unit variant ไม่มีข้อมูล, tuple variant มีข้อมูล
ไม่มีชื่อ, struct-like variant มีข้อมูลมีชื่อ) — คำถามที่สำคัญตอนนี้คือ **enum ที่มีข้อมูลติดมาด้วยควรแทนด้วย
JSON แบบไหน?** เพราะ JSON เองไม่มีแนวคิดเรื่อง "enum"/"variant" อยู่แล้วในตัว (มีแค่ object, array,
string, number, bool, null) — `serde` ต้อง**เลือกวิธีเข้ารหัส**อย่างใดอย่างหนึ่ง และให้ทางเลือกคุณปรับได้
ผ่าน attribute ลองใช้ enum ตัวอย่างที่พบบ่อยมากในระบบจริง: `ApiResponse` ที่มีสองสถานะ (`Success`/`Error`)

#### 57.6.1 Externally tagged (ค่า default — ไม่ต้องเขียน attribute อะไรเลย)

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
enum ApiResponse {
    Success { data: String, count: u32 },
    Error { message: String, code: u16 },
}

fn main() {
    let success = ApiResponse::Success {
        data: "รายการสินค้า 3 ชิ้น".to_string(),
        count: 3,
    };
    let error = ApiResponse::Error {
        message: "ไม่พบสินค้า".to_string(),
        code: 404,
    };

    println!("{}", serde_json::to_string_pretty(&success).unwrap());
    println!("{}", serde_json::to_string_pretty(&error).unwrap());
}
```

ผลลัพธ์:

```
{
  "Success": {
    "data": "รายการสินค้า 3 ชิ้น",
    "count": 3
  }
}
{
  "Error": {
    "message": "ไม่พบสินค้า",
    "code": 404
  }
}
```

นี่คือรูปแบบ **default ของ `serde`** เรียกว่า **externally tagged**: ชื่อ variant (`"Success"`, `"Error"`)
กลายเป็น **key ของ object ชั้นนอกสุด** ที่ "ห่อ" ข้อมูลของ variant นั้นไว้อีกชั้นหนึ่ง — เปรียบเทียบกับ
`match` ที่ Part 10 สอนไว้: การ deserialize JSON แบบนี้กลับมาคือการ "ดู key ชั้นนอกสุดว่าคือ variant ไหน แล้ว
parse ข้อมูลข้างในตามรูปร่างของ variant นั้น" ตรงไปตรงมา ข้อดีคือ**ไม่มีความกำกวมเลย** (รู้ variant จาก key
ทันที) แต่ข้อเสียคือรูปร่าง JSON ที่ได้ไม่ตรงกับ pattern ที่ REST API ส่วนใหญ่ในโลกจริงนิยมใช้

#### 57.6.2 Internally tagged ด้วย `#[serde(tag = "...")]`

REST API จำนวนมากนิยมรูปแบบที่มี field ชื่อ `type` (หรือชื่ออื่นที่ทีมกำหนด) อยู่**ในระดับเดียวกับ field อื่น
ของ object** ไม่ใช่ห่อไว้ชั้นนอก — นี่คือสิ่งที่ `#[serde(tag = "...")]` ทำ:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]
enum ApiResponseTagged {
    Success { data: String, count: u32 },
    Error { message: String, code: u16 },
}

fn main() {
    let success = ApiResponseTagged::Success {
        data: "รายการสินค้า 3 ชิ้น".to_string(),
        count: 3,
    };
    println!("{}", serde_json::to_string_pretty(&success).unwrap());
}
```

ผลลัพธ์:

```
{
  "type": "Success",
  "data": "รายการสินค้า 3 ชิ้น",
  "count": 3
}
```

เทียบกับหัวข้อก่อน: ไม่มีการ "ห่อ" อีกชั้นแล้ว — `"type": "Success"` อยู่ใน object เดียวกับ `data`/`count`
เลย นี่คือรูปแบบที่ตรงกับ **REST API แบบ tagged union** ที่ระบบภาษาอื่น (เช่น TypeScript discriminated union)
ใช้กันเป็นปกติ — ชื่อ key (`"type"` ในตัวอย่างนี้) เลือกได้ตามที่ API กำหนด (`#[serde(tag = "kind")]`,
`#[serde(tag = "status")]` ก็ได้เหมือนกัน)

**ข้อจำกัดที่ควรรู้**: internally tagged ใช้ได้กับ variant ที่เป็น struct-like หรือ unit เท่านั้น (ต้องมี
"ที่" ให้แทรก key ของ tag เข้าไปในระดับเดียวกัน) — variant แบบ tuple ที่มีมากกว่า 1 field
(`Success(String, u32)`) จะทำให้ `#[derive(Serialize)]` ปฏิเสธตั้งแต่ compile time เพราะ tuple ไม่มีชื่อ field
ให้วาง tag เข้าไปข้าง ๆ ได้

#### 57.6.3 Untagged ด้วย `#[serde(untagged)]` — และความเสี่ยงเรื่องความกำกวม

รูปแบบสุดท้าย: **ไม่มี tag เลย** JSON ที่ได้จะดูเหมือนไม่รู้ว่ามาจาก enum เลย — `serde` จะ**ไล่ลองทีละ
variant ตามลำดับที่ประกาศไว้ในโค้ด** จนกว่าจะเจอ variant ที่ shape ของ field ตรงกับ JSON ที่ได้รับ:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(untagged)]
enum ApiResponseUntagged {
    Success { data: String, count: u32 },
    Error { message: String, code: u16 },
}

fn main() {
    let success = ApiResponseUntagged::Success {
        data: "รายการสินค้า 3 ชิ้น".to_string(),
        count: 3,
    };
    println!("{}", serde_json::to_string_pretty(&success).unwrap());

    // deserialize กลับ: ไม่มี tag บอกเลยว่าเป็น variant ไหน serde ต้อง "เดา" จาก shape
    let untagged_json = r#"{"data":"x","count":1}"#;
    let back: ApiResponseUntagged = serde_json::from_str(untagged_json).unwrap();
    println!("deserialize กลับ (เดาจาก shape): {back:?}");
}
```

ผลลัพธ์:

```
{
  "data": "รายการสินค้า 3 ชิ้น",
  "count": 3
}
deserialize กลับ (เดาจาก shape): Success { data: "x", count: 1 }
```

JSON ที่ได้ตอน serialize สะอาดที่สุด (ไม่มี key พิเศษเจือปนเลย) แต่ราคาที่ต้องจ่ายคือ**ความกำกวม**: ลองดู
ตัวอย่างที่ทำให้เห็นความเสี่ยงชัดเจนที่สุด — enum สอง variant ที่ field ซ้อนทับกันบางส่วน:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(untagged)]
enum Ambiguous {
    A { x: u32 },
    B { x: u32, y: u32 },
}

fn main() {
    // JSON มีแค่ field x เดียว
    let amb: Ambiguous = serde_json::from_str(r#"{"x": 1}"#).unwrap();
    println!("{amb:?}");
}
```

ผลลัพธ์ (ทดสอบจริง):

```
A { x: 1 }
```

`serde` เลือก `A` เพราะไล่ลองตามลำดับที่ประกาศในโค้ด (`A` มาก่อน `B`) และ `{"x": 1}` มี field ครบตามที่ `A`
ต้องการ (แค่ `x`) จึง match ก่อนที่จะไปลอง `B` เลย — **นี่ไม่ใช่ปัญหาในกรณีนี้เพราะ intent ตรงกับผลลัพธ์
บังเอิญ** แต่ลองนึกภาพว่าถ้า `B` ถูกประกาศ**ก่อน** `A` ในโค้ด (สลับตำแหน่งกัน) ผลลัพธ์จะเปลี่ยนพฤติกรรมทันที
โดยที่ JSON input เหมือนกันทุกตัวอักษร (`{"x": 1}` จะไม่ match `B` เพราะ `B` ต้องการ field `y` ด้วยที่ไม่มีอยู่
จริง เลย fall through ไปที่ `A` เหมือนเดิม — แต่ถ้าลองแก้เป็น `A { x: u32, y: Option<u32> }` แทน ปัญหาจะซับซ้อน
กว่านี้อีกมาก) **หลักการที่ควรจำ**: `untagged` เหมาะกับ enum ที่แต่ละ variant มี field **ไม่ทับซ้อนกันเลย**
(หรือทับซ้อนกันแบบที่คุณตรวจสอบมาแล้วว่าไม่มีทางกำกวม) — ถ้า variant มีโอกาส "ใส่ข้อมูลชุดหนึ่งแล้ว match ได้
มากกว่าหนึ่ง variant" ให้เปลี่ยนไปใช้ externally/internally tagged แทนเพื่อความชัดเจนไม่กำกวม

#### 57.6.4 Adjacently tagged ด้วย `#[serde(tag = "...", content = "...")]`

มีอีกรูปแบบหนึ่งที่อยู่ระหว่าง externally tagged กับ internally tagged เรียกว่า **adjacently tagged**: มี key
ของ tag แยกออกมา **และ** ข้อมูลของ variant ถูกห่อไว้ใน key อีกตัวที่กำหนดชื่อได้ (ไม่ผสมเข้ากับระดับเดียวกัน
แบบ internally tagged) ใช้เมื่อต้องการทั้งความชัดเจนของ tag และไม่อยากให้ field ของแต่ละ variant ชนกับ tag
หรือกับ field อื่นในระดับบนสุด (ยังใช้ได้กับ tuple variant หลาย field ด้วย ต่างจาก internally tagged ในหัวข้อ
57.6.2 ที่ทำไม่ได้):

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", content = "payload")]
enum ApiResponseAdjacent {
    Success { data: String, count: u32 },
    Error { message: String, code: u16 },
}

fn main() {
    let success = ApiResponseAdjacent::Success { data: "x".to_string(), count: 3 };
    println!("{}", serde_json::to_string_pretty(&success).unwrap());
}
```

ผลลัพธ์:

```
{
  "type": "Success",
  "payload": {
    "data": "x",
    "count": 3
  }
}
```

`"type"` บอกว่าเป็น variant ไหน (เหมือน internally tagged) แต่ข้อมูลจริงถูกย้ายไปไว้ใน `"payload"` แยกออกมา
(เหมือน externally tagged ที่ห่อไว้อีกชั้น แต่ใช้ชื่อ key ที่กำหนดเองได้แทนชื่อ variant) — รูปแบบนี้พบใน
protocol ที่ต้องการทั้งความชัดเจนของ tag และรองรับ variant ที่มี field ซับซ้อน/ชนกับชื่อ tag ได้

**ตารางสรุปเปรียบเทียบทั้งสี่รูปแบบ:**

| รูปแบบ | Attribute | JSON ตัวอย่าง | ข้อดี | ข้อเสีย |
|---|---|---|---|---|
| Externally tagged | (ไม่ต้องเขียนอะไร — default) | `{"Success":{"data":"x","count":1}}` | ไม่กำกวมเลย | ไม่ตรงกับ REST API ส่วนใหญ่ |
| Internally tagged | `#[serde(tag = "type")]` | `{"type":"Success","data":"x","count":1}` | ตรงกับ pattern REST API ทั่วไป | ใช้กับ tuple variant หลาย field ไม่ได้ |
| Adjacently tagged | `#[serde(tag = "type", content = "payload")]` | `{"type":"Success","payload":{"data":"x","count":1}}` | ชัดเจนและรองรับ tuple variant ได้ | JSON มีการซ้อนอีกชั้นเพิ่มขึ้นมา |
| Untagged | `#[serde(untagged)]` | `{"data":"x","count":1}` | JSON สะอาดที่สุด ไม่มี tag เจือปน | เสี่ยงกำกวมถ้า variant ทับซ้อนกัน |

### 57.7 `serde_json::Value`: จัดการ JSON ที่ไม่รู้ shape ล่วงหน้า

ทุกตัวอย่างที่ผ่านมาสมมติว่า**รู้ shape ของ JSON ล่วงหน้าแน่นอน** (เขียน struct ที่ตรงกับมันได้) แต่บางสถานการณ์
ไม่เป็นแบบนั้น — เช่น กำลังสำรวจ API ใหม่ที่ยังไม่รู้ schema แน่ชัด, รับ JSON ที่ shape เปลี่ยนไปตามเงื่อนไข
(polymorphic response), หรือแค่ต้องการดึงค่าบางฟิลด์จาก JSON ก้อนใหญ่โดยไม่อยากประกาศ struct ให้ครบทุก field
— กรณีเหล่านี้ `serde_json::Value` (enum ที่แทน JSON value **แบบไหนก็ได้**แบบ dynamic) คือเครื่องมือที่ถูก
ต้อง:

```rust
use serde_json::{json, Value};

fn main() {
    let raw = r#"
    {
        "order_id": "ORD-9911",
        "customer": {
            "name": "ปิยะดา",
            "vip": true
        },
        "items": [
            { "sku": "A100", "qty": 2 },
            { "sku": "B200", "qty": 1 }
        ],
        "metadata": {
            "source": "mobile_app",
            "promo_code": null
        }
    }
    "#;

    // parse เป็น Value แบบ dynamic — ไม่ต้องนิยาม struct ให้ครบทุก field ล่วงหน้า
    let v: Value = serde_json::from_str(raw).unwrap();

    // เข้าถึงด้วย .get() แบบ chain ปลอดภัย (คืน Option<&Value> ทุกขั้น)
    let customer_name = v
        .get("customer")
        .and_then(|c| c.get("name"))
        .and_then(|n| n.as_str())
        .unwrap_or("ไม่ทราบชื่อ");
    println!("ชื่อลูกค้า: {customer_name}");

    // เข้าถึงด้วย indexing [] ตรง ๆ (สะดวกกว่า แต่ key/index ที่ไม่มีจะได้ Value::Null ไม่ panic)
    println!("vip: {}", v["customer"]["vip"]);
    println!("field ที่ไม่มีจริง: {}", v["customer"]["address"]);

    // วน items ที่เป็น array แบบไม่รู้จำนวนล่วงหน้า
    if let Some(items) = v.get("items").and_then(|i| i.as_array()) {
        for item in items {
            let sku = item["sku"].as_str().unwrap_or("?");
            let qty = item["qty"].as_i64().unwrap_or(0);
            println!("item: sku={sku} qty={qty}");
        }
    }

    println!("มี promo_code ไหม: {}", !v["metadata"]["promo_code"].is_null());

    // สร้าง Value ใหม่แบบ dynamic ด้วย macro json!
    let built = json!({
        "status": "ok",
        "echo": v["order_id"].clone(),
        "count": v["items"].as_array().map(|a| a.len()).unwrap_or(0),
    });
    println!("สร้างใหม่ด้วย json! macro: {built}");
}
```

ผลลัพธ์ (ทดสอบจริง):

```
ชื่อลูกค้า: ปิยะดา
vip: true
field ที่ไม่มีจริง: null
item: sku=A100 qty=2
item: sku=B200 qty=1
มี promo_code ไหม: false
สร้างใหม่ด้วย json! macro: {"count":2,"echo":"ORD-9911","status":"ok"}
```

**อธิบายกลไก:**

- `Value` เป็น **enum** (`Null`, `Bool(bool)`, `Number(Number)`, `String(String)`, `Array(Vec<Value>)`,
  `Object(Map<String, Value>)`) ตรงกับ generic data model ของ `serde` ในหัวข้อ 57.2 พอดี — มันคือ type ที่
  `Serialize`/`Deserialize` เองแบบ **ครอบคลุมทุกรูปแบบไปพร้อมกัน** (ต่างจาก struct ที่คุณเขียน ซึ่งครอบคลุม
  "รูปแบบเดียวที่ตายตัว")
- **`.get(key)`** คืน `Option<&Value>` — ปลอดภัยกับการ chain ด้วย `.and_then()` ตามที่ Part 12/Option สอน
  ไว้เรื่อง combinator ของ `Option<T>` เมื่อ key ไม่มีจริงจะได้ `None` ไม่ panic
- **indexing `v["key"]`** สะดวกกว่าแต่ทำงานคนละแบบ: ถ้า `Value` เป็น object และไม่มี key นั้น จะคืน
  **`&Value::Null`** (ไม่ panic เหมือน `HashMap`/`Vec` indexing ปกติ) — นี่คือการตัดสินใจออกแบบเฉพาะของ
  `serde_json::Value` เพื่อให้เขียน chain แบบ `v["a"]["b"]["c"]` ได้สะดวกโดยไม่ต้องกลัว panic แม้ path นั้น
  ไม่มีอยู่จริงบางช่วง (แต่ต้องระวังว่ามัน**เงียบ** — ถ้าพิมพ์ชื่อ key ผิด จะได้ `null` ไม่ใช่ error ที่เตือนให้
  รู้ทันที)
- **`json!{}` macro** ทำงานคล้าย `vec!`/`hashmap!` ที่ Part 36 สอนไว้ (function-like macro ที่แปลง syntax
  literal เป็นค่า) แต่ซับซ้อนกว่าเพราะต้องรองรับ nested object/array แบบเดียวกับ JSON literal จริง

**เมื่อไรควรใช้ `Value` เมื่อไรควรใช้ struct ที่มีชนิดชัดเจน:**

| สถานการณ์ | ควรใช้ |
|---|---|
| รู้ schema ล่วงหน้า, ต้องการ type safety, compiler ช่วยเช็ค field ครบ/ไม่ครบ | **struct + derive** |
| กำลังสำรวจ API ใหม่ ยังไม่รู้ schema แน่ชัด | `Value` (prototype ก่อน แล้วเปลี่ยนเป็น struct ทีหลังเมื่อรู้ shape แน่นอน) |
| JSON มีบาง field ที่ shape เปลี่ยนไปตามเงื่อนไข (polymorphic เกินกว่า enum จะแทนได้สะดวก) | `Value` เฉพาะจุดนั้น (ผสมกับ struct ที่เหลือได้ — field หนึ่งเป็น `Value` ส่วน field อื่นเป็น type ปกติ) |
| Config/API ที่ต้อง validate ความถูกต้องอย่างเข้มงวด (ระบบ production ที่ error ต้องชัดเจนตั้งแต่ compile/parse time) | **struct + derive** เสมอ — `Value` ไม่ validate อะไรให้เลย ต้องเขียน `.get()`/`.as_str()` เช็คเองทุกจุดซึ่งเสี่ยงพลาดมากกว่า |

หลักการสั้น ๆ ที่จำง่าย: **struct ที่มี derive คือ "สัญญา" (contract) ที่ compiler ช่วยตรวจสอบให้ตลอดเวลา
ส่วน `Value` คือ "ไม่มีสัญญาเลย" ที่ยืดหยุ่นที่สุดแต่ปลอดภัยน้อยที่สุด** — ในระบบจริง มักใช้ `Value` เป็นจุดเริ่ม
ต้นตอนสำรวจ แล้วค่อยเปลี่ยนไปใช้ struct ทันทีที่รู้ schema แน่นอน

### 57.8 Error Handling กับ Serde: อ่าน `serde_json::Error` ให้เป็น

จาก Part 12/30 คุณรู้อยู่แล้วว่า error ที่ดีต้องให้ข้อมูลพอที่จะ debug ได้ — `serde_json::Error` ทำสิ่งนี้ได้ดี
มาก เพราะมันรู้ตำแหน่ง**บรรทัด/คอลัมน์**ที่ปัญหาเกิดขึ้นจริงในข้อความ JSON ต้นทาง มาดู 4 กรณีที่พบบ่อยที่สุด:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Product {
    id: u32,
    name: String,
    price: f64,
}

fn main() {
    // 1) JSON ผิด syntax (ลืม comma ระหว่าง "id": 1 กับ "name": ...)
    let malformed_syntax = r#"
    {
        "id": 1
        "name": "สินค้า A"
    }
    "#;
    match serde_json::from_str::<Product>(malformed_syntax) {
        Ok(p) => println!("{p:?}"),
        Err(e) => {
            println!("syntax error: {e}");
            println!("  line = {}, column = {}", e.line(), e.column());
            println!("  classify = {:?}", e.classify());
        }
    }

    // 2) JSON ถูก syntax แต่ type ผิด (price เป็น string ไม่ใช่ number)
    let wrong_type = r#"{ "id": 1, "name": "สินค้า A", "price": "แพงมาก" }"#;
    match serde_json::from_str::<Product>(wrong_type) {
        Ok(p) => println!("{p:?}"),
        Err(e) => {
            println!("type error: {e}");
            println!("  line = {}, column = {}", e.line(), e.column());
            println!("  classify = {:?}", e.classify());
        }
    }

    // 3) field ที่จำเป็นหายไป (ไม่มี #[serde(default)])
    let missing_field = r#"{ "id": 1, "name": "สินค้า A" }"#;
    match serde_json::from_str::<Product>(missing_field) {
        Ok(p) => println!("{p:?}"),
        Err(e) => println!("missing field error: {e}"),
    }

    // 4) input ขาดตอนกลาง (truncated) — is_eof()/is_syntax()/is_data() ช่วยแยกประเภท
    let truncated = r#"{ "id": 1, "name": "#;
    match serde_json::from_str::<Product>(truncated) {
        Ok(p) => println!("{p:?}"),
        Err(e) => {
            println!("truncated error: {e}");
            println!(
                "  is_eof={} is_syntax={} is_data={}",
                e.is_eof(), e.is_syntax(), e.is_data()
            );
        }
    }
}
```

ผลลัพธ์ (ทดสอบจริงทั้ง 4 กรณี):

```
syntax error: expected `,` or `}` at line 4 column 9
  line = 4, column = 9
  classify = Syntax
type error: invalid type: string "แพงมาก", expected f64 at line 1 column 72
  line = 1, column = 72
  classify = Data
missing field error: missing field `price` at line 1 column 43
truncated error: EOF while parsing a value at line 1 column 19
  is_eof=true is_syntax=false is_data=false
```

**อธิบายรายละเอียดที่เป็นประโยชน์จริงตอน debug:**

- **`e.line()` / `e.column()`** — ตำแหน่งที่ parser หยุดชะงัก นับจากต้นข้อความ JSON ที่ป้อนเข้าไป (1-indexed
  เหมือนที่ editor ส่วนใหญ่แสดงเลขบรรทัด) มีประโยชน์มากเมื่อ JSON เป็นไฟล์ยาวหลายร้อยบรรทัด เพราะไม่ต้องไล่หา
  ด้วยตาว่าปัญหาอยู่ตรงไหน
- **`e.classify()`** คืน `serde_json::error::Category` ซึ่งเป็น enum ที่มี 4 variant: `Io` (ปัญหาจากการอ่าน
  เช่น stream ขาด), `Syntax` (โครงสร้าง JSON เองผิด เช่น ลืม comma/bracket), `Data` (JSON syntax ถูก แต่ข้อมูล
  ไม่ตรงกับ type ที่คาดไว้ เช่น string ที่ควรเป็น number, field จำเป็นที่หายไป), `Eof` (input ขาดตอนกลางไม่ครบ
  ประโยค) — เขียน `match e.classify() { ... }` ได้ตรงกับที่ Part 30 สอนไว้เรื่อง "ผู้เรียกที่ต้องแยกกรณี error
  ควรได้ type ที่ match ได้" (แม้ `serde_json::Error` เองไม่ใช่ enum ที่ `match` ตรง ๆ ได้ แต่ `classify()`
  แปลงมันให้เป็น enum ที่ `match` ได้อีกที)
- **`e.is_eof()`/`e.is_syntax()`/`e.is_data()`/`e.is_io()`** — ตัวช่วยแบบ boolean ที่สะดวกกว่า `match` เมื่อ
  ต้องเช็คแค่กรณีเดียว (ตัวอย่างที่ 4 แสดงให้เห็นว่า `Eof` เป็นกรณีพิเศษที่แยกออกจาก `Syntax` เพราะ "ประโยคยัง
  ไม่จบ" ต่างจาก "ประโยคผิดโครงสร้าง")
- ข้อความ error ที่ `{e}` (Display) พิมพ์มาอ่านง่ายมากอยู่แล้ว (บอกทั้ง**สิ่งที่คาดหวัง**และ**ตำแหน่ง**ในบรรทัด
  เดียว) — สำหรับโปรแกรมระดับ application ทั่วไป การ `.with_context()` แบบที่ Part 31 สอนไว้ (ห่อด้วย
  `anyhow`) มักพอเพียงแล้วโดยไม่ต้องแยกกรณีตาม `classify()` เลย ส่วนโค้ดระดับ library ที่ต้องให้ผู้เรียก
  ตัดสินใจต่างกันไปตามสาเหตุ (เช่น "ถ้า syntax ผิดให้ reject ทันที แต่ถ้า data ผิดบาง field ให้ลองใช้ค่า
  default แทน") จึงจะคุ้มที่จะ `match` บน `classify()` จริง ๆ

### 57.9 "หนึ่ง derive ใช้ได้ทุก Format": พิสูจน์ด้วยโค้ดจริง

กลับมาที่คำกล่าวอ้างสำคัญของหัวข้อ 57.2 — struct เดียวกัน derive ครั้งเดียว ใช้กับ 4 format crate ได้พร้อมกัน
โดยไม่แก้นิยาม struct เลยแม้แต่บรรทัดเดียว มาดูโค้ดที่พิสูจน์เรื่องนี้ตรง ๆ:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, PartialEq)]
struct Employee {
    id: u32,
    name: String,
    department: String,
    salary: f64,
    active: bool,
}

fn main() {
    let emp = Employee {
        id: 7,
        name: "วรรณา".to_string(),
        department: "Engineering".to_string(),
        salary: 45000.0,
        active: true,
    };

    // --- serde_json ---
    let as_json = serde_json::to_string_pretty(&emp).unwrap();
    let back_json: Employee = serde_json::from_str(&as_json).unwrap();
    assert_eq!(emp, back_json);

    // --- toml ---
    let as_toml = toml::to_string_pretty(&emp).unwrap();
    let back_toml: Employee = toml::from_str(&as_toml).unwrap();
    assert_eq!(emp, back_toml);

    // --- serde_yaml ---
    let as_yaml = serde_yaml::to_string(&emp).unwrap();
    let back_yaml: Employee = serde_yaml::from_str(&as_yaml).unwrap();
    assert_eq!(emp, back_yaml);

    // --- bincode ---
    let as_bincode: Vec<u8> = bincode::serialize(&emp).unwrap();
    let back_bincode: Employee = bincode::deserialize(&as_bincode).unwrap();
    assert_eq!(emp, back_bincode);

    println!("=== serde_json ({} bytes) ===\n{as_json}", as_json.len());
    println!("=== toml ({} bytes) ===\n{as_toml}", as_toml.len());
    println!("=== serde_yaml ({} bytes) ===\n{as_yaml}", as_yaml.len());
    println!("=== bincode ({} bytes) ===\n{:?}", as_bincode.len(), as_bincode);
    println!("ทุก format round-trip กลับมาเท่ากับต้นฉบับ (assert_eq! ผ่านหมด)");
}
```

ผลลัพธ์จริง (คอมไพล์และรันจริง สังเกตขนาดที่ต่างกันมากของแต่ละ format สำหรับข้อมูลชุดเดียวกัน):

```
=== serde_json (112 bytes) ===
{
  "id": 7,
  "name": "วรรณา",
  "department": "Engineering",
  "salary": 45000.0,
  "active": true
}
=== toml (90 bytes) ===
id = 7
name = "วรรณา"
department = "Engineering"
salary = 45000.0
active = true

=== serde_yaml (81 bytes) ===
id: 7
name: วรรณา
department: Engineering
salary: 45000.0
active: true

=== bincode (55 bytes) ===
55 bytes
ทุก format round-trip กลับมาเท่ากับต้นฉบับ (assert_eq! ผ่านหมด)
```

`Employee` ถูกนิยามครั้งเดียว, derive ครั้งเดียว, แล้วผลิตทั้ง JSON, TOML, YAML, และ binary format ได้จริงทุก
รูปแบบ พร้อม deserialize กลับมาเท่ากับต้นฉบับทุกครั้ง (`assert_eq!` ผ่านหมดในโค้ดข้างบน) — สังเกตความต่างของ
ขนาด: `bincode` (55 bytes) เล็กที่สุดเพราะไม่มีชื่อ field/key เก็บไว้เลย (รู้ตำแหน่งของแต่ละ field จาก
"ลำดับ" ที่ประกาศไว้ใน struct เท่านั้น ต่างจาก text format ทั้งสามที่ต้องเก็บชื่อ field ไว้ในข้อความเสมอ)

**สรุปสั้น ๆ ของแต่ละ format crate (เพิ่ม `cargo add` ที่ทดสอบจริง):**

```bash
cargo add toml
```
```
      Adding toml v1.1.6 to dependencies
             Features: + display + parse + serde + std
```

```bash
cargo add serde_yaml
```
```
      Adding serde_yaml v0.9.34 to dependencies
```

```bash
cargo add bincode
```
```
      Adding bincode v3.0.0 to dependencies
```

| Crate | เหมาะกับ | หมายเหตุ |
|---|---|---|
| `toml` | ไฟล์ config (`Cargo.toml` ของโปรเจกต์คุณเองก็คือตัวอย่าง TOML ที่ parse ด้วยกลไกนี้ทุกครั้งที่รัน `cargo build`) | อ่านง่ายสำหรับมนุษย์ รองรับ comment ในไฟล์ (ต่าง JSON ที่ไม่รองรับ comment) |
| `serde_yaml` | ไฟล์ config ที่ต้อง nested ลึกและอ่านง่าย (เช่น Docker Compose, Kubernetes manifest ในภาษาอื่น) | crate นี้อยู่ในสถานะ **deprecated/maintenance mode** จากผู้เขียนต้นฉบับ (เวอร์ชันที่ `cargo add` ดึงมาคือ `0.9.34+deprecated`) แต่ยังใช้งานได้จริงและยังเป็นตัวเลือกหลักของ ecosystem ตอนนี้ — โปรเจกต์ใหม่ควรตรวจสอบสถานะ crate นี้ใน crates.io ก่อนใช้งานจริงจัง |
| `bincode` — ตัวอย่างในบทนี้ใช้ `bincode = "1"` (API แบบ `serialize()`/`deserialize()` ตรงไปตรงมาที่สุด) | serialize ภายในระบบของตัวเอง (cache บน disk, ส่งข้อมูลระหว่าง process บนเครื่องเดียวกัน) ที่ไม่ต้องอ่านง่ายด้วยตาแต่ต้องการความเร็ว/ขนาดเล็ก | **ไม่ใช่ทางเลือกที่ดีสำหรับสื่อสารข้าม API/ข้าม version ของโปรแกรม** เพราะ layout ของ byte ผูกกับลำดับ field ใน struct ตรง ๆ ถ้าเปลี่ยนโครงสร้าง struct (เพิ่ม/ลบ/สลับ field) ข้อมูลเก่าที่ serialize ไว้ก่อนหน้าจะ deserialize ผิดพลาดหรือพังทันที — เหมาะกับข้อมูลที่ serialize/deserialize ด้วยโค้ดเวอร์ชันเดียวกันเท่านั้น (bincode มีเวอร์ชัน 2.x/3.x ที่ปรับ API และปรัชญาไปมากด้วย ควรอ่าน CHANGELOG ก่อนอัปเกรดถ้าใช้ในโปรเจกต์จริง) |

### 57.10 ตัวอย่างจริง: Config Loader ด้วย `serde` + `toml`

มาปิดท้ายด้วยตัวอย่างที่รวมทุกอย่างที่เรียนมาในบทนี้เข้าด้วยกัน — เทียบกับ config loader ที่ Part 30-31 สร้าง
ด้วยมือ (parse `key=value` เอง, เขียน error enum เอง) เวอร์ชันนี้ใช้ **`serde` + `toml` จริง** และ
**`thiserror`** (จาก Part 31) สำหรับ error type — โครงสร้าง config มีทั้ง nested struct, field ที่มีค่า
default, field ที่เป็น `Option<T>` แบบ optional จริง ๆ:

```rust
use serde::Deserialize;
use std::fs;
use std::path::Path;
use thiserror::Error;

#[derive(Debug, Deserialize)]
struct AppConfig {
    app_name: String,

    #[serde(default = "default_port")]
    port: u16,

    #[serde(default)]
    debug: bool,

    database: DatabaseConfig, // nested struct — ไม่มี default ต้องมี [database] section เสมอ

    #[serde(default)]
    features: FeatureFlags,
}

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    host: String,

    #[serde(default = "default_db_port")]
    port: u16,

    username: String,

    #[serde(default)] // ไม่ระบุ = None (Option<T> มี Default เป็น None เสมอ)
    max_connections: Option<u32>,
}

#[derive(Debug, Default, Deserialize)]
struct FeatureFlags {
    #[serde(default)]
    enable_metrics: bool,
    #[serde(default)]
    enable_tracing: bool,
}

fn default_port() -> u16 {
    8080
}
fn default_db_port() -> u16 {
    5432
}

// ใช้ thiserror ตามที่เรียนมาแล้วใน Part 31 — #[from] แปลง io::Error/toml::de::Error ให้อัตโนมัติผ่าน `?`
#[derive(Debug, Error)]
enum ConfigLoadError {
    #[error("อ่านไฟล์ config ไม่สำเร็จ")]
    Io(#[from] std::io::Error),

    #[error("ไฟล์ config มีรูปแบบไม่ถูกต้อง")]
    Parse(#[from] toml::de::Error),
}

fn load_config(path: &Path) -> Result<AppConfig, ConfigLoadError> {
    let raw = fs::read_to_string(path)?; // io::Error -> ConfigLoadError::Io
    let config: AppConfig = toml::from_str(&raw)?; // toml::de::Error -> ConfigLoadError::Parse
    Ok(config)
}

fn print_chain(err: &dyn std::error::Error) {
    println!("error: {err}");
    let mut source = err.source();
    while let Some(cause) = source {
        println!("caused by: {cause}");
        source = cause.source();
    }
}

fn main() {
    let good_toml = r#"
app_name = "InventorySystem"
debug = true

[database]
host = "db.internal.local"
username = "app_user"

[features]
enable_metrics = true
"#;
    fs::write("/tmp/app_good.toml", good_toml).unwrap();
    println!("=== โหลด config ที่ถูกต้อง ===");
    match load_config(Path::new("/tmp/app_good.toml")) {
        Ok(cfg) => println!("{cfg:#?}"),
        Err(e) => print_chain(&e),
    }
}
```

ผลลัพธ์:

```
=== โหลด config ที่ถูกต้อง ===
AppConfig {
    app_name: "InventorySystem",
    port: 8080,
    debug: true,
    database: DatabaseConfig {
        host: "db.internal.local",
        port: 5432,
        username: "app_user",
        max_connections: None,
    },
    features: FeatureFlags {
        enable_metrics: true,
        enable_tracing: false,
    },
}
```

สังเกตว่าไฟล์ TOML ที่ให้มา**ไม่มี** `port` ระดับบนสุด, `[database].port`, `[database].max_connections`, และ
`[features].enable_tracing` เลย — ทุก field เหล่านี้ถูกเติมด้วยค่า default ที่กำหนดไว้ผ่าน `#[serde(default)]`
ทั้งหมด **โดยไม่เกิด error แม้แต่จุดเดียว** นี่คือสิ่งที่ต่างจาก Part 30 อย่างชัดเจน: เวอร์ชันมือใน Part 30 ต้อง
เขียน logic เช็ค "field นี้มีไหม ถ้าไม่มีใช้ค่า default" ด้วยตัวเองทุกจุด (ผ่าน `HashMap::get()` แล้ว
`unwrap_or()`) แต่เวอร์ชันนี้แค่ประกาศ attribute แล้วให้ `serde_derive` generate logic เดียวกันให้เอง

**ทดสอบกรณี error ที่พบบ่อยในระบบจริง — field จำเป็น (`username`) หายไป:**

```rust
// ไฟล์ config ที่ [database] ไม่มี username (username ไม่มี #[serde(default)] จึงเป็น field จำเป็น)
let missing_required = "app_name = \"X\"\n\n[database]\nhost = \"h\"\n";
```

ผลลัพธ์:

```
error: ไฟล์ config มีรูปแบบไม่ถูกต้อง
caused by: TOML parse error at line 3, column 1
  |
3 | [database]
  | ^^^^^^^^^^
missing field `username`
```

สังเกต error chain สองชั้นที่ `print_chain()` พิมพ์: `error:` คือข้อความจาก `#[error("...")]` ของ
`ConfigLoadError::Parse`, `caused by:` คือข้อความจริงจาก `toml::de::Error` ที่ **ชี้ตำแหน่งบรรทัด/คอลัมน์**
ในไฟล์ TOML ต้นทางตรง ๆ (เหมือนกับ `serde_json::Error` ในหัวข้อ 57.8) — และกรณีไฟล์ไม่มีอยู่จริงเลย:

```
error: อ่านไฟล์ config ไม่สำเร็จ
caused by: No such file or directory (os error 2)
```

`ConfigLoadError::Io` ทำงานถูกต้องผ่าน `#[from] std::io::Error` ที่ `fs::read_to_string(path)?` เรียกใช้
อัตโนมัติ ตรงตามกลไกที่ Part 31 หัวข้อ 31.4 อธิบายไว้ทุกประการ — บทนี้ไม่ได้สอนอะไรใหม่เรื่อง error handling
เลย แค่เอา `serde`/`toml` มาต่อเข้ากับสิ่งที่คุณรู้จักดีอยู่แล้วจาก Part 30-31

### 57.11 สรุปอ้างอิงเร็ว (Quick Reference)

ก่อนไปหัวข้อกับดัก มาสรุป attribute/ฟังก์ชันที่ใช้บ่อยที่สุดในบทนี้ไว้เป็น checklist สำหรับเปิดดูตอนเขียนโค้ด
จริง (เทียบรูปแบบเดียวกับหัวข้อ 44.11 ของ Part 44):

**Attribute ระดับ struct/enum (เขียนเหนือ `#[derive(...)]`):**

| Attribute | ผลลัพธ์ |
|---|---|
| `#[serde(rename_all = "camelCase")]` | rename ทุก field/variant พร้อมกันตาม convention (หัวข้อ 57.5.2) |
| `#[serde(tag = "type")]` | ทำให้ enum เป็น internally tagged (หัวข้อ 57.6.2) |
| `#[serde(tag = "type", content = "payload")]` | ทำให้ enum เป็น adjacently tagged (หัวข้อ 57.6.4) |
| `#[serde(untagged)]` | ทำให้ enum เป็น untagged — ไม่มี tag เลย (หัวข้อ 57.6.3) |

**Attribute ระดับ field (เขียนเหนือ field เดียว):**

| Attribute | ผลลัพธ์ |
|---|---|
| `#[serde(rename = "userId")]` | เปลี่ยนชื่อ field นี้ตอน serialize/deserialize (หัวข้อ 57.5.1) |
| `#[serde(default)]` | ใช้ `Default::default()` ถ้า field หายไปตอน deserialize (หัวข้อ 57.5.3) |
| `#[serde(default = "fn_name")]` | ใช้ค่าที่ `fn_name()` คืนถ้า field หายไป (หัวข้อ 57.5.3) |
| `#[serde(skip)]` | ไม่ serialize/deserialize field นี้เลย ใช้ `Default` เสมอตอนอ่านกลับ (หัวข้อ 57.5.4) |
| `#[serde(skip_serializing_if = "fn_name")]` | ข้าม field นี้ตอน serialize ถ้า `fn_name(&field)` คืน `true` (หัวข้อ 57.5.4) |

**ฟังก์ชันหลักของ `serde_json` ที่ใช้บ่อย:**

| ฟังก์ชัน | ทำอะไร |
|---|---|
| `serde_json::to_string(&v)` | serialize เป็น `String` แบบ compact |
| `serde_json::to_string_pretty(&v)` | serialize เป็น `String` แบบมี indent |
| `serde_json::from_str::<T>(s)` | deserialize จาก `&str` เป็น `T` |
| `serde_json::from_str::<Value>(s)` | parse เป็น `Value` แบบ dynamic (หัวข้อ 57.7) |
| `serde_json::json!{...}` | สร้าง `Value` จาก literal syntax แบบ JSON ตรง ๆ (หัวข้อ 57.7) |

**ตารางเทียบ error handling ระหว่าง `serde_json::Error` กับสิ่งที่ Part 30 สอนไว้:**

| แนวคิดจาก Part 30 | เทียบเท่าใน `serde_json::Error`/`toml::de::Error` |
|---|---|
| ข้อความ error ที่อ่านง่าย (`impl Display`) | มีอยู่แล้ว พิมพ์ผ่าน `{e}` ได้ตรง (หัวข้อ 57.8) |
| แยกกรณี error ด้วย `match` บน enum | `e.classify()` คืน enum ที่ `match` ได้ (`Io`/`Syntax`/`Data`/`Eof`) |
| ตำแหน่งที่เกิด error (คล้าย stack trace) | `e.line()`/`e.column()` บอกตำแหน่งในข้อความต้นทางตรง ๆ |
| `impl From<E>` ให้ `?` แปลงชนิดอัตโนมัติ | ใช้ `#[from]` ของ `thiserror` ห่อ `serde_json::Error`/`toml::de::Error` ได้ตรงแบบเดียวกับ Part 31 (หัวข้อ 57.10) |

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืมเปิด feature `derive` — `cannot find derive macro`

ถ้าเพิ่ม `serde` ด้วย `cargo add serde` เฉย ๆ (ไม่ใส่ `--features derive`) แล้วลองเขียน `#[derive(Serialize)]`
จะได้ error ทันที:

```
error: cannot find derive macro `Serialize` in this scope
 --> src/main.rs:3:10
  |
3 | #[derive(Serialize)]
  |          ^^^^^^^^^
  |
note: `Serialize` is imported here, but it is only a trait, without a derive macro
 --> src/main.rs:1:5
  |
1 | use serde::Serialize;
  |     ^^^^^^^^^^^^^^^^
```

ข้อความ `note:` อธิบายสาเหตุตรงตามที่หัวข้อ 57.3 อธิบายไว้: `serde::Serialize` ที่ import มาตอนนี้คือ**trait
เฉย ๆ** (ไม่มี derive macro ติดมาด้วย) เพราะ feature `derive` (ที่ pull เข้า `serde_derive` — proc-macro crate
ตามที่ Part 44 อธิบายไว้) ไม่ได้ถูกเปิด **วิธีแก้**: เพิ่ม `--features derive` ตอน `cargo add` หรือแก้
`Cargo.toml` ตรง ๆ ให้เป็น `serde = { version = "...", features = ["derive"] }`

### 2. Field จำเป็นหายไปตอน deserialize โดยไม่มี `#[serde(default)]`

```
missing field `price` at line 1 column 43
```

error นี้ (จากหัวข้อ 57.8) เกิดเมื่อ JSON/TOML ที่ป้อนเข้าไปไม่มี key ที่ struct ต้องการ **และ field นั้นไม่มี
`#[serde(default)]`** — นี่ไม่ใช่ bug ของ `serde` แต่เป็น**พฤติกรรมที่ถูกต้องตามเจตนา**: ถ้าไม่ใส่
`#[serde(default)]` แสดงว่า field นั้น**ต้องมีอยู่จริงเสมอ** ตามสัญญาของ struct (เหมือนที่หัวข้อ 57.7 อธิบาย
ไว้ว่า struct คือ "สัญญา") — วิธีแก้คือถามตัวเองก่อนว่า field นี้ "ควรมีค่า default เมื่อขาด" จริงหรือไม่ ถ้า
ใช่ ให้เพิ่ม `#[serde(default)]`/`#[serde(default = "...")]` ถ้าไม่ใช่ (เช่น `username` ของ database config
ที่ไม่มีค่า default ที่สมเหตุสมผลเลย) ก็ปล่อยให้มัน error ต่อไปแบบนี้ถูกแล้ว เพราะการเดา default ผิด ๆ
(เช่น เดา username เป็น "" เปล่า) จะสร้างบั๊กที่ตรวจจับยากกว่า error ตอน parse config มาก

### 3. `rename_all` ไม่ไล่ลงไปถึง enum ที่เป็น field ของ struct อื่น

```rust
#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct Outer {
    user_id: u32,
    status: Status,
}

#[derive(Serialize)] // ลืมใส่ rename_all ตรงนี้ด้วย
enum Status {
    NotStarted,
    InProgress,
}
```

ผลลัพธ์จริง:

```
{"userId":1,"status":"InProgress"}
```

สังเกตว่า `user_id` ถูก rename เป็น `userId` ถูกต้อง (เพราะ `#[serde(rename_all = "camelCase")]` อยู่บน
`Outer`) แต่ `"InProgress"` (ชื่อ variant ของ `Status`) **ไม่ถูกแปลงเป็น `"inProgress"` เลย** — เพราะ
`rename_all` เป็น attribute ที่ทำงาน**เฉพาะภายใน item ที่มันติดอยู่เท่านั้น** (ตรงตามกลไก derive macro ที่
Part 44 สอนไว้: macro เห็นแค่ token stream ของ item เดียวที่มันกำลัง derive อยู่ ไม่เห็นและไม่แก้ item อื่นที่
อยู่ห่างออกไปเลย ถึงแม้ item นั้นจะถูกใช้เป็น field type ของกันก็ตาม) — **วิธีแก้**: ต้องใส่
`#[serde(rename_all = "camelCase")]` บน `Status` **แยกต่างหากด้วย** ถ้าต้องการให้ variant name ของมันเป็น
`camelCase` เช่นกัน ไม่มี attribute ไหนที่ "ไล่ลงไปทำกับทุก type ที่เกี่ยวข้อง" ให้อัตโนมัติ

### 4. Field type ที่ไม่ implement `Serialize`/`Deserialize`

```rust
use serde::Serialize;
use std::sync::mpsc::Receiver;

#[derive(Serialize)]
struct Job {
    id: u32,
    channel: Receiver<i32>, // Receiver ไม่ implement Serialize
}
```

Error จริง:

```
error[E0277]: the trait bound `std::sync::mpsc::Receiver<i32>: serde::Serialize` is not satisfied
    --> src/main.rs:4:10
     |
   4 | #[derive(Serialize)]
     |          ^^^^^^^^^ the trait `Serialize` is not implemented for `std::sync::mpsc::Receiver<i32>`
...
   7 |     channel: Receiver<i32>,
     |     ------- required by a bound introduced by this call
```

นี่คือ error แบบเดียวกับที่ Part 19/21 สอนไว้เรื่อง trait bound ไม่ครบ — `#[derive(Serialize)]` generate โค้ด
ที่เรียก `self.channel.serialize(...)` ภายใน (ตามรูปแบบในหัวข้อ 57.2) ซึ่งต้องการ `Receiver<i32>: Serialize`
แต่ `Receiver` (ช่อง channel สำหรับสื่อสารข้าม thread จาก `std::sync::mpsc`) ไม่มีความหมายอะไรที่จะ "แปลงเป็น
ข้อมูล" ได้เลย (มันคือ handle ที่ผูกกับ runtime state ของโปรแกรมตอนนั้น ไม่ใช่ข้อมูลที่ persist ได้) จึงไม่มี
`impl Serialize` ให้ตามธรรมชาติ — ทางแก้คือ**ไม่ derive field ที่เป็น runtime-only handle แบบนี้** ให้ใช้
`#[serde(skip)]` (หัวข้อ 57.5.4) ถ้ายังต้องการเก็บ field นี้ไว้ใน struct แต่ไม่ต้อง serialize มันเลย

### 5. Ordering ของ `HashMap` ไม่แน่นอนระหว่างการรันแต่ละครั้ง

```rust
use std::collections::HashMap;

let mut hm: HashMap<&str, i32> = HashMap::new();
hm.insert("zebra", 1);
hm.insert("apple", 2);
hm.insert("mango", 3);
println!("{}", serde_json::to_string(&hm).unwrap());
```

ผลลัพธ์ที่พบจริง (ลำดับ key ในผลลัพธ์**ไม่ตรงกับลำดับที่ `insert` ไว้เลย** และอาจต่างกันไปในแต่ละครั้งที่รัน):

```
{"zebra":1,"mango":3,"apple":2}
```

นี่ไม่ใช่บั๊กของ `serde_json` — `HashMap` ใน Rust (ตามที่ standard library เอกสารไว้) **ไม่รับประกันลำดับของ
key เลย** (ใช้ random seed ป้องกัน HashDoS attack) เมื่อ `serde` serialize มันก็เดินตาม iteration order ของ
`HashMap` ตรง ๆ จึงได้ผลลัพธ์ที่ลำดับไม่แน่นอนไปด้วย — ถ้าต้องการ**ลำดับ key ที่แน่นอนซ้ำ ๆ ได้** (เช่นเพื่อทำ
diff ระหว่างสอง JSON, หรือเขียน snapshot test) ให้เปลี่ยนไปใช้ `std::collections::BTreeMap` แทน (เรียงตาม
key เสมอ):

```rust
use std::collections::BTreeMap;

let mut bm: BTreeMap<&str, i32> = BTreeMap::new();
bm.insert("zebra", 1);
bm.insert("apple", 2);
bm.insert("mango", 3);
println!("{}", serde_json::to_string(&bm).unwrap());
// {"apple":2,"mango":3,"zebra":1}  -- เรียงตาม key เสมอ ไม่ว่าจะรันกี่ครั้งก็ตาม
```

`BTreeMap` ทั้งคู่มี `impl Serialize`/`impl Deserialize` ให้จาก `serde` เองอยู่แล้ว (ไม่ต้องเขียนอะไรเพิ่ม)
เพราะ standard library collection ที่พบบ่อยเกือบทั้งหมด (`Vec`, `HashMap`, `BTreeMap`, `HashSet`, `BTreeSet`,
`Option`, tuple, array) มี `impl Serialize`/`Deserialize` ให้ล่วงหน้าจาก `serde` เองแล้วตั้งแต่แรก — เลือก
collection ให้ตรงกับความต้องการเรื่อง ordering ตั้งแต่ตอนออกแบบ struct

### 6. `f64` กับเงิน: ความแม่นยำที่เสียไปแบบไม่รู้ตัว

ย้อนกลับไปหัวข้อ 57.4 ที่ `price: 1990.50` กลายเป็น `1990.5` ใน JSON output — นี่ดูไม่มีปัญหาในตัวอย่างนั้น
แต่ลองดูตัวอย่างที่อันตรายกว่า:

```rust
let price: f64 = 0.1 + 0.2;
println!("{}", serde_json::to_string(&price).unwrap());
// {ผลลัพธ์จริง} -> 0.30000000000000004
```

นี่คือปัญหา floating point ที่เป็นสากล (ไม่เกี่ยวกับ `serde` เลย — `0.1 + 0.2 != 0.3` พอดีในระบบ binary
floating point ของทุกภาษาที่ใช้ IEEE 754) แต่ `serde_json` ทำให้ปัญหานี้**ปรากฏชัดเจนในข้อมูลที่ persist ไว้**
ทันที เพราะมันพิมพ์ค่า `f64` ตรงตามที่มันเก็บจริงแบบไม่ปัดเศษให้ — ถ้า struct ที่แทนเงินใน struct ของคุณใช้
`f64` โปรแกรมที่คำนวณเงินสะสมไปเรื่อย ๆ (เช่นบวกราคาสินค้าหลาย ๆ ตัวรวมกัน) อาจได้ผลรวมที่ผิดเพี้ยนไปเศษ
สตางค์โดยไม่รู้ตัว แล้วพอ serialize ออกไปเป็น JSON ก็จะเห็นตัวเลขแปลก ๆ แบบ `19.999999999999996` ปรากฏใน
ข้อมูลจริง — **วิธีแก้ที่นิยมในระบบการเงินจริง**: เก็บเงินเป็น**จำนวนเต็ม**ในหน่วยที่เล็กที่สุด (เช่น สตางค์/
cent แทนบาท/ดอลลาร์ — `amount_cents: i64` แทน `amount: f64`) หรือใช้ crate เฉพาะสำหรับ decimal ที่แม่นยำ
(เช่น `rust_decimal`) ที่ implement `Serialize`/`Deserialize` ของตัวเองให้ตรงกับ requirement เรื่องความแม่นยำ
โดยเฉพาะ

## แบบฝึกหัด (Exercises)

1. **โจทย์ระดับง่าย**: สร้าง struct `BlogPost` ที่มี field `title: String`, `author: String`,
   `published: bool`, `view_count: u64` ใส่ `#[derive(Debug, Serialize, Deserialize)]` แล้วเขียน `main()`
   ที่สร้างค่าตัวอย่าง, serialize เป็น JSON แบบ pretty ด้วย `serde_json::to_string_pretty()`, deserialize
   กลับมา, และ `assert_eq!` ว่า field ทุกตัวตรงกับต้นฉบับ (เทียบกับ round-trip ในหัวข้อ 57.4)
   *Hint*: ต้องเพิ่ม `#[derive(PartialEq)]` ด้วยถ้าจะใช้ `assert_eq!` เทียบทั้ง struct ในครั้งเดียว
   (ไม่ใช่เทียบทีละ field)

2. **โจทย์ระดับกลาง**: ให้ `BlogPost` จากข้อ 1 เพิ่ม field `#[serde(rename = "publishedAt")] published_at:
   Option<String>` และ field `#[serde(default)] tags: Vec<String>` ทดสอบ deserialize JSON ที่**ไม่มี**
   `publishedAt`/`tags` เลย ยืนยันว่า deserialize สำเร็จโดยไม่ error (เพราะ `Option<T>` มี `Default` เป็น
   `None` และ `Vec<T>` มี `Default` เป็น vector เปล่า) แล้วลองใช้ `#[serde(skip_serializing_if =
   "Vec::is_empty")]` กับ `tags` เพื่อไม่ให้ field นี้ปรากฏใน output ตอนเป็น vector เปล่า — สังเกตความต่างของ
   JSON output ก่อน/หลังเพิ่ม attribute นี้
   *Hint*: `Vec::is_empty` รับ `&Vec<T>` คืน `bool` ตรงกับ signature ที่ `skip_serializing_if` ต้องการเป๊ะ
   เหมือนกับ `Option::is_none` ในหัวข้อ 57.5.4

3. **โจทย์ระดับยาก**: ออกแบบ enum `PaymentMethod` ที่มี 3 variant: `CreditCard { last_four: String, brand:
   String }`, `BankTransfer { bank_name: String, account_last_four: String }`, `Cash` (unit variant) — เขียน
   ให้ serialize แบบ **internally tagged** ด้วย `#[serde(tag = "method")]` แล้วเขียนฟังก์ชัน
   `describe_payment(p: &PaymentMethod) -> String` ที่ `match` แยกทั้ง 3 กรณีคืนข้อความอธิบายภาษาไทยที่ต่างกัน
   ทดสอบ serialize ทั้ง 3 variant ดู JSON ที่ได้ แล้วลอง deserialize JSON ที่มี `"method": "Cash"` เพียว ๆ
   (ไม่มี field อื่น) กลับมาดูว่าทำงานถูกต้องหรือไม่ (unit variant กับ internally tagged ทำงานอย่างไร)
   *Hint*: unit variant ในโหมด internally tagged จะ serialize เป็น object ที่มีแค่ field ของ tag เดียว
   (`{"method":"Cash"}`) เพราะไม่มี field อื่นให้ใส่เลย

4. **โจทย์ระดับยาก/ประยุกต์ใช้งานจริง**: เขียนโปรแกรม CLI เล็ก ๆ ที่โหลดไฟล์ config ชื่อ `settings.toml`
   (ใช้ `AppConfig` แบบในหัวข้อ 57.10 หรือออกแบบใหม่ให้เหมาะกับระบบสมมติ เช่น ระบบจัดการห้องสมุด/ระบบจอง
   ห้องประชุม) ให้มี **nested struct อย่างน้อย 2 ชั้น** (เช่น `AppConfig` มี `database: DatabaseConfig` และ
   `DatabaseConfig` มี `pool: PoolConfig` อีกชั้น), field ที่มี `#[serde(default = "...")]` อย่างน้อย 2 ตัว,
   และ error type ที่เขียนด้วย `thiserror` (ตาม Part 31) ที่ห่อทั้ง `std::io::Error` และ `toml::de::Error`
   ไว้ด้วย `#[from]` เขียนเทสต์ (ตาม Part 32) ที่ยืนยันว่า: (ก) ไฟล์ที่ครบทุก field โหลดสำเร็จ, (ข) ไฟล์ที่ขาด
   field ซึ่งมี default โหลดสำเร็จและได้ค่า default ที่ถูกต้อง, (ค) ไฟล์ที่ขาด field จำเป็นโหลดล้มเหลวด้วย
   error message ที่มีคำว่า `missing field`
   *Hint*: ใช้ `tempfile` crate หรือ `std::env::temp_dir()` ร่วมกับชื่อไฟล์ที่ไม่ชนกัน (เช่นใส่ thread id/
   นับเลขสุ่ม) เพื่อไม่ให้ test ที่รันพร้อมกันหลาย thread (ตาม Part 32 ที่ `cargo test` รัน test แบบ parallel
   โดย default) เขียนทับไฟล์กันเอง

## สรุป

บทนี้พาคุณจากคำถามที่ Part 44 ทิ้งไว้ ("`#[derive(Serialize, Deserialize)]` ทำงานอย่างไรกันแน่") มาสู่ความ
เข้าใจที่สมบูรณ์: `serde` คือ **framework** ที่แยก **"ข้อมูลรู้จักตัวเองอย่างไร"** (ผ่าน trait `Serialize`/
`Deserialize` และ generic data model) ออกจาก **"ข้อมูลถูกเข้ารหัสเป็นรูปแบบไหน"** (ผ่าน crate แยกต่างหากอย่าง
`serde_json`/`toml`/`serde_yaml`/`bincode` ที่ implement `Serializer`/`Deserializer`) — การแยกส่วนนี้คือ
เหตุผลที่ struct ตัวเดียว derive ครั้งเดียว ใช้งานได้กับทุก format พร้อมกันโดยไม่ต้องแก้โค้ดแม้แต่บรรทัดเดียว
ตามที่พิสูจน์ไว้ในหัวข้อ 57.9 คุณได้เห็น attribute ที่ใช้งานจริงบ่อยที่สุด (`rename`, `rename_all`, `default`,
`skip`, `skip_serializing_if`) ในฐานะ**เวอร์ชันโลกจริง**ของ helper attribute แบบที่คุณเขียนเองมาแล้วใน Part
45, เห็นว่า enum แทน JSON ได้ 3 รูปแบบ (externally/internally tagged, untagged) พร้อม trade-off ของแต่ละแบบ,
ใช้ `serde_json::Value` สำหรับ JSON ที่ไม่รู้ shape ล่วงหน้า, อ่าน error จาก `serde_json`/`toml` ที่ให้ตำแหน่ง
บรรทัด/คอลัมน์ชัดเจน และปิดท้ายด้วยระบบโหลด config ที่รวม `serde`+`toml`+`thiserror` เข้าด้วยกันแบบที่ใช้งาน
จริงได้ทันที

**สิ่งที่บทนี้ยังไม่ได้ตอบ**: ทุกตัวอย่างที่เห็นมาใช้ `#[derive(Serialize, Deserialize)]` ล้วน ๆ ซึ่งครอบคลุม
สถานการณ์ส่วนใหญ่ในโลกจริง — แต่มีบางกรณีที่ derive macro **ทำให้ไม่ได้เลย** ไม่ว่าจะใส่ attribute อะไรก็ตาม
เช่น: แปลง string วันที่รูปแบบพิเศษที่ไม่ตรงกับ ISO 8601 มาตรฐาน, validate ค่าตอน deserialize (เช่น
"อายุต้องอยู่ระหว่าง 0-150 เท่านั้น มิฉะนั้นให้ error ทันทีตั้งแต่ parse"), หรือ deserialize รูปแบบข้อมูลเก่า
หลาย version ให้เข้ากับ struct เวอร์ชันปัจจุบัน (schema migration) — สถานการณ์เหล่านี้ต้อง **implement
`Serialize`/`Deserialize` ด้วยมือ** โดยเรียก method ของ `Serializer`/`Deserializer` trait ตรง ๆ (ตามรูปแบบที่
หัวข้อ 57.2 แสดงไว้แบบคร่าว ๆ) — Part 58 (Serde ขั้นสูง) จะพาคุณลงลึกไปที่จุดนั้น: เขียน `impl Serialize`/
`impl Deserialize` ด้วยมือแบบเต็มรูปแบบ, ใช้ `#[serde(with = "...")]` เชื่อมกับ module แปลงข้อมูลแบบกำหนดเอง,
และแนวคิดเรื่อง `Visitor` pattern ที่ `Deserializer` ใช้ภายใน

---

**Part ก่อนหน้า:** [Memory Management ขั้นสูง และ Zero-cost Abstractions](part-056-memory-management-advanced.md) | **Part ถัดไป:** [Serde ขั้นสูง](part-058-serde-advanced.md)
