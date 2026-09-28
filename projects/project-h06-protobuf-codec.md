# Project H06: Protocol Buffer Codec (from scratch)

> โมดูล: H — Networking/Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 18 ชั่วโมง

## ภาพรวมโปรเจค

**Protocol Buffers (protobuf)** คือ binary serialization format ที่ Google พัฒนาขึ้น ใช้เป็น default encoding format ของ gRPC และระบบ microservice ขนาดใหญ่ทั่วโลก เช่น Google, Uber, Lyft, Netflix ด้วยคุณสมบัติสำคัญ 3 ประการ:

1. **ขนาดเล็กกว่า JSON** — ไม่มี key ซ้ำในทุก message, ใช้ varint encoding ทำให้ตัวเลขเล็ก ๆ ใช้พื้นที่น้อยมาก
2. **เร็วกว่า JSON** — binary format ที่ decode ได้โดยไม่ต้อง parse text
3. **Schema-first** — type safety ที่ enforce ตั้งแต่ compile time ผ่าน .proto file

ในโปรเจคนี้เราจะ **สร้าง protobuf wire format codec จาก scratch** — ไม่ใช้ library `prost` หรือ `protobuf` — เพื่อทำความเข้าใจกลไกภายในอย่างลึกซึ้ง:

- วิธีที่ integer encode เป็น LEB128 varint ได้ขนาดเล็กมาก
- Zigzag encoding ที่ทำให้ signed integer ลบ ๆ ใช้พื้นที่น้อยแทนที่จะเป็น 10 bytes
- Tag system ที่ทำให้ forward/backward compatibility กับ schema เป็นไปได้
- Message framing ที่ไม่มี separator — decode ได้โดยนับ bytes เอง

### Use Case ใน Production

| ระบบ | ใช้ protobuf อย่างไร |
|------|---------------------|
| **gRPC** | encode/decode request/response ทุก call |
| **Kafka Avro** | serialize events ก่อนส่งเข้า topic |
| **Google Cloud Datastore** | เก็บ entity ใน storage layer |
| **Android Binder IPC** | ส่ง messages ระหว่าง process |
| **Chrome DevTools Protocol** | debug message format |

---

## สิ่งที่จะได้เรียนรู้

- **LEB128 varint encoding** — encode u64 ลงในจำนวน bytes ที่แปรผัน, MSB = continuation bit
- **Zigzag encoding** สำหรับ signed integer — เปลี่ยน -1 → 1, 1 → 2, -2 → 3, 2 → 4 เพื่อให้ varint สั้น
- **Wire type system** — Tag = (field_number << 3) | wire_type ทำให้ decoder รู้ว่าต้องอ่านกี่ bytes
- **FieldValue enum** กับ pattern matching เพื่อ serialize/deserialize ข้อมูลหลายชนิด
- **Fluent builder pattern** — `MessageBuilder::new().add_string(1,"x").build()`
- **Unknown field passthrough** — กฎสำคัญของ protobuf ที่ทำให้ forward-compatible
- **Schema validation** — validate decoded message กับ schema struct ที่กำหนดเอง
- **JSON serialization** จาก protobuf-decoded fields ด้วย `serde_json`

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1-20** — Rust fundamentals: ownership, borrowing, structs, enums, match
- **Part 21-30** — Iterators, closures, `Vec<u8>`, slices, byte manipulation
- **Part 31-40** — Traits, generics, `impl Trait`, error handling, `Option`/`Result`
- **Part 41-50** — การทำงานกับ binary data, bit operations (`<<`, `>>`, `&`, `|`)
- **Part 51-60** — Advanced enums, pattern matching, destructuring
- **Part 96-110** — `serde`, `serde_json`, derive macros
- โปรเจค H05 (Load Balancer) — ทำความเข้าใจ encoding/protocol basics

---

## โครงสร้างโปรเจค (Project Layout)

```
protobuf-codec/
├── src/
│   ├── lib.rs           # re-exports ทุก module
│   ├── main.rs          # demo binary แสดง encode/decode/validate/JSON
│   ├── wire.rs          # WireType enum, Tag struct
│   ├── varint.rs        # LEB128 encode/decode, zigzag encode/decode
│   ├── field.rs         # FieldValue enum, encode_field, decode_field
│   ├── message.rs       # MessageBuilder fluent API, MessageDecoder
│   └── schema.rs        # ProtoMessage schema, FieldDef, validation, to_json
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Wire Format Spec

protobuf message คือ sequence ของ fields ต่อกันโดยตรง ไม่มี separator:

```
[ tag | value ] [ tag | value ] [ tag | value ] ...
```

**Tag** คือ varint เดียวที่encode field_number และ wire_type:

```
tag_value = (field_number << 3) | wire_type
```

ตัวอย่าง: field 1, wire type Varint → `(1 << 3) | 0 = 8`

| wire_type | ค่า | ชนิดข้อมูล |
|-----------|-----|-----------|
| `Varint`  | 0   | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| `I64`     | 1   | fixed64, sfixed64, double (8 bytes ตายตัว) |
| `Len`     | 2   | string, bytes, embedded message, packed repeated |
| `I32`     | 5   | fixed32, sfixed32, float (4 bytes ตายตัว) |

### Data Flow

```
encode path:
MessageBuilder
    │ .add_int32(1, 42)  →  encode_field(1, Varint(42))
    │                    →  encode_varint(tag) + encode_varint(42)
    │ .add_string(2,"x") →  encode_field(2, Bytes(b"x"))
    │                    →  encode_varint(tag) + encode_varint(len) + bytes
    └ .build()           →  Vec<u8>  [08 2A 12 01 78]

decode path:
&[u8]
    │ decode_field() → (tag_varint → Tag { field_num, wire_type })
    │               → read value by wire_type
    └ loop until end → Vec<(u32, FieldValue)>
                           │
                     MessageDecoder::decode()
                           │
                     schema.validate() + schema.to_json()
```

### Varint LEB128 Encoding

LEB128 (Little Endian Base 128) ใช้ 7 bits ต่อ byte สำหรับข้อมูล และ 1 bit (MSB) เป็น continuation flag:

```
300 = 0b1_0010_1100
แบ่งเป็น 7-bit groups (จาก LSB): 010_1100 = 0x2C, 000_0010 = 0x02
byte 1: 0xAC = 1_010_1100 (MSB=1 หมายความว่ามี byte ต่อไป)
byte 2: 0x02 = 0_000_0010 (MSB=0 คือ byte สุดท้าย)
```

ความสำเร็จของ varint: ค่า 0-127 ใช้แค่ 1 byte, 128-16383 ใช้ 2 bytes — เหมาะสำหรับ field number และค่า integer ขนาดเล็กที่พบบ่อย

### Zigzag Encoding

two's complement ของ -1 คือ `0xFFFFFFFFFFFFFFFF` ซึ่งจะกิน 10 bytes ถ้าใช้ varint ธรรมดา zigzag แก้ปัญหานี้โดย map i64 → u64:

```
encode: (n << 1) ^ (n >> 63)
  0 → 0,  -1 → 1,  1 → 2,  -2 → 3,  2 → 4

decode: (n >> 1) ^ -(n & 1)
  0 → 0,  1 → -1,  2 → 1,  3 → -2,  4 → 2
```

ทำให้ค่า `-1` encode เป็น `1` ซึ่งใช้แค่ 1 byte แทนที่จะเป็น 10 bytes

### Unknown Field Passthrough

กฎสำคัญของ protobuf spec: **decoder ต้องเก็บ unknown fields ไว้ ไม่ throw error** ทำให้:

```
ระบบ A (schema v1):  field 1, 2
ระบบ B (schema v2):  field 1, 2, 3  ← ใหม่

ถ้า B ส่งข้อมูลไปให้ A: A decode field 1, 2 ได้ปกติ
field 3 เป็น unknown field → A เก็บไว้ pass-through
ถ้า A forward ข้อมูลนั้นต่อไป field 3 ยังอยู่ครบ
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Wire Type และ Tag

เริ่มจากโครงสร้างพื้นฐานที่สุด — `WireType` enum และ `Tag` struct ที่เข้ารหัส/ถอดรหัส field tag

```toml
# Cargo.toml
[package]
name = "protobuf-codec"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
```

```rust
// src/wire.rs

/// Wire type ของ Protocol Buffer ตาม spec
/// ดู https://protobuf.dev/programming-guides/encoding/#structure
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum WireType {
    /// Varint — int32, int64, uint32, uint64, sint32, sint64, bool, enum
    Varint = 0,
    /// 64-bit — fixed64, sfixed64, double
    I64 = 1,
    /// Length-delimited — string, bytes, embedded messages, packed repeated
    Len = 2,
    /// 32-bit — fixed32, sfixed32, float
    I32 = 5,
}

impl WireType {
    /// แปลง u8 เป็น WireType; คืน None ถ้า wire type ไม่ถูกต้อง
    pub fn from_u8(v: u8) -> Option<Self> {
        match v {
            0 => Some(WireType::Varint),
            1 => Some(WireType::I64),
            2 => Some(WireType::Len),
            5 => Some(WireType::I32),
            _ => None,
        }
    }
}

/// Field tag = (field_number << 3) | wire_type
/// ขนาดสูงสุดของ field number ใน protobuf คือ 2^29 - 1
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Tag {
    pub field_number: u32,
    pub wire_type: WireType,
}

impl Tag {
    pub fn new(field_number: u32, wire_type: WireType) -> Self {
        Tag { field_number, wire_type }
    }

    /// เข้ารหัส tag เป็น u64 (เพื่อส่งผ่าน varint encoder)
    pub fn encode(&self) -> u64 {
        ((self.field_number as u64) << 3) | (self.wire_type as u64)
    }

    /// ถอดรหัส u64 เป็น Tag
    pub fn decode(value: u64) -> Option<Self> {
        let wire_raw = (value & 0x07) as u8;
        let field_number = (value >> 3) as u32;
        if field_number == 0 {
            return None;
        }
        WireType::from_u8(wire_raw).map(|wt| Tag { field_number, wire_type: wt })
    }
}
```

ทดสอบการทำงานขั้นที่ 1:

```rust
// ใน src/wire.rs หัวข้อ #[cfg(test)]
let tag = Tag::new(1, WireType::Varint);
let encoded = tag.encode();
// field 1, wire Varint → (1 << 3) | 0 = 8
assert_eq!(encoded, 8);
let decoded = Tag::decode(8).unwrap();
assert_eq!(decoded.field_number, 1);
assert_eq!(decoded.wire_type, WireType::Varint);
```

---

### ขั้นที่ 2: Varint Encoder และ Decoder

ใจกลางของ protobuf คือ LEB128 varint — ตัวเลขที่ขนาดของ byte ขึ้นอยู่กับค่า ไม่ใช่ type

```rust
// src/varint.rs

/// เข้ารหัส u64 เป็น LEB128 variable-length encoding
/// แต่ละ byte ใช้ 7 bit สำหรับข้อมูล และ 1 bit (MSB) บ่งบอกว่ายังมี byte ต่อไป
pub fn encode_varint(mut value: u64) -> Vec<u8> {
    let mut buf = Vec::new();
    loop {
        let mut byte = (value & 0x7F) as u8;
        value >>= 7;
        if value != 0 {
            byte |= 0x80; // set continuation bit
        }
        buf.push(byte);
        if value == 0 {
            break;
        }
    }
    buf
}

/// ถอดรหัส LEB128 จาก byte slice
/// คืนค่า (decoded_value, bytes_consumed)
/// คืน None ถ้า buffer หมดก่อนครบ varint
pub fn decode_varint(buf: &[u8]) -> Option<(u64, usize)> {
    let mut result: u64 = 0;
    let mut shift = 0u32;
    for (i, &byte) in buf.iter().enumerate() {
        let low7 = (byte & 0x7F) as u64;
        if shift >= 64 {
            return None; // overflow — varint ยาวเกิน 10 bytes
        }
        result |= low7 << shift;
        shift += 7;
        if byte & 0x80 == 0 {
            // MSB = 0 หมายความว่านี่คือ byte สุดท้าย
            return Some((result, i + 1));
        }
    }
    None // buffer หมดก่อน varint จบ
}
```

ตัวอย่างการ encode ค่าต่าง ๆ:

```
encode_varint(0)   → [0x00]          (1 byte)
encode_varint(1)   → [0x01]          (1 byte)
encode_varint(127) → [0x7F]          (1 byte)
encode_varint(128) → [0x80, 0x01]    (2 bytes)
encode_varint(300) → [0xAC, 0x02]    (2 bytes)

ค้นหา encode 300:
  300 = 0b100101100
  7-bit groups: 0101100, 0000010
  byte 1: 10101100 = 0xAC  (MSB=1 = continuation)
  byte 2: 00000010 = 0x02  (MSB=0 = last byte)
```

---

### ขั้นที่ 3: Zigzag Encoding สำหรับ Signed Integers

`sint32` และ `sint64` ใน protobuf ใช้ zigzag encoding แทน two's complement เพื่อทำให้ค่าลบขนาดเล็กมี varint สั้น

```rust
// ต่อใน src/varint.rs

/// Zigzag encoding สำหรับ signed integer (sint32/sint64)
/// แปลง i64 → u64 โดยใช้สูตร: (n << 1) ^ (n >> 63)
pub fn zigzag_encode(n: i64) -> u64 {
    ((n << 1) ^ (n >> 63)) as u64
}

/// Zigzag decoding: u64 → i64
/// สูตร: (n >> 1) ^ -(n & 1)
pub fn zigzag_decode(n: u64) -> i64 {
    ((n >> 1) as i64) ^ -((n & 1) as i64)
}
```

ตารางเปรียบเทียบระหว่าง two's complement และ zigzag:

| ค่า signed | two's complement (varint bytes) | zigzag (varint bytes) |
|-----------|--------------------------------|----------------------|
| 0         | 1 byte (0x00)                  | 1 byte (0x00) |
| -1        | 10 bytes (0xFF x9, 0x01)       | 1 byte (0x01) |
| 1         | 1 byte (0x01)                  | 1 byte (0x02) |
| -2        | 10 bytes                       | 1 byte (0x03) |
| -100      | 10 bytes                       | 1 byte (0xC7) |
| -128      | 10 bytes                       | 2 bytes |

---

### ขั้นที่ 4: FieldValue และ Field Encoder/Decoder

```rust
// src/field.rs
use crate::varint::{decode_varint, encode_varint};
use crate::wire::{Tag, WireType};

/// ค่าของ field ใน protobuf message
#[derive(Debug, Clone, PartialEq)]
pub enum FieldValue {
    /// Wire type 0: int32, int64, uint32, uint64, bool, enum
    Varint(u64),
    /// Wire type 1: fixed64, sfixed64, double (8 bytes little-endian)
    Fixed64([u8; 8]),
    /// Wire type 2: string, bytes, embedded messages
    Bytes(Vec<u8>),
    /// Wire type 5: fixed32, sfixed32, float (4 bytes little-endian)
    Fixed32([u8; 4]),
}

impl FieldValue {
    /// ดึง wire type ที่ตรงกับ variant นี้
    pub fn wire_type(&self) -> WireType {
        match self {
            FieldValue::Varint(_) => WireType::Varint,
            FieldValue::Fixed64(_) => WireType::I64,
            FieldValue::Bytes(_) => WireType::Len,
            FieldValue::Fixed32(_) => WireType::I32,
        }
    }

    /// เข้ารหัสส่วน value (ไม่รวม tag) เป็น bytes
    pub fn encode_value(&self) -> Vec<u8> {
        match self {
            FieldValue::Varint(v) => encode_varint(*v),
            FieldValue::Fixed64(b) => b.to_vec(),
            FieldValue::Fixed32(b) => b.to_vec(),
            FieldValue::Bytes(data) => {
                let mut out = encode_varint(data.len() as u64);
                out.extend_from_slice(data);
                out
            }
        }
    }
}

/// เข้ารหัส field เดียว (tag + value) เป็น bytes
pub fn encode_field(field_number: u32, value: &FieldValue) -> Vec<u8> {
    let tag = Tag::new(field_number, value.wire_type());
    let mut out = encode_varint(tag.encode());
    out.extend(value.encode_value());
    out
}

/// ถอดรหัส field จาก byte slice
/// คืน (field_number, FieldValue, bytes_consumed) หรือ None ถ้า parse ไม่ได้
pub fn decode_field(buf: &[u8]) -> Option<(u32, FieldValue, usize)> {
    if buf.is_empty() {
        return None;
    }

    let (tag_raw, tag_len) = decode_varint(buf)?;
    let tag = Tag::decode(tag_raw)?;
    let rest = &buf[tag_len..];

    match tag.wire_type {
        WireType::Varint => {
            let (val, val_len) = decode_varint(rest)?;
            Some((tag.field_number, FieldValue::Varint(val), tag_len + val_len))
        }
        WireType::I64 => {
            if rest.len() < 8 { return None; }
            let mut bytes = [0u8; 8];
            bytes.copy_from_slice(&rest[..8]);
            Some((tag.field_number, FieldValue::Fixed64(bytes), tag_len + 8))
        }
        WireType::Len => {
            let (length, len_len) = decode_varint(rest)?;
            let data_start = len_len;
            let data_end = data_start + length as usize;
            if rest.len() < data_end { return None; }
            let data = rest[data_start..data_end].to_vec();
            Some((tag.field_number, FieldValue::Bytes(data), tag_len + data_end))
        }
        WireType::I32 => {
            if rest.len() < 4 { return None; }
            let mut bytes = [0u8; 4];
            bytes.copy_from_slice(&rest[..4]);
            Some((tag.field_number, FieldValue::Fixed32(bytes), tag_len + 4))
        }
    }
}
```

ตัวอย่างการ encode field ต่าง ๆ และผลลัพธ์ bytes:

```
encode_field(1, Varint(42)):
  tag = (1 << 3) | 0 = 8 → varint = [0x08]
  value 42 → varint = [0x2A]
  result: [0x08, 0x2A]

encode_field(2, Bytes(b"hi")):
  tag = (2 << 3) | 2 = 18 → varint = [0x12]
  length 2 → varint = [0x02]
  data = [0x68, 0x69]
  result: [0x12, 0x02, 0x68, 0x69]

encode_field(3, Fixed32([0x00, 0x00, 0x48, 0x42])):  // 3.14f32
  tag = (3 << 3) | 5 = 29 → varint = [0x1D]
  4 bytes = [0xC3, 0xF5, 0x48, 0x40]
  result: [0x1D, 0xC3, 0xF5, 0x48, 0x40]
```

---

### ขั้นที่ 5: MessageBuilder — Fluent API

`MessageBuilder` เป็น fluent API ที่ทำให้สร้าง message ได้ง่ายโดยต่อ method call แบบ chain

```rust
// src/message.rs
use crate::field::{decode_field, encode_field, FieldValue};
use crate::varint::zigzag_encode;

/// MessageBuilder: Fluent API สำหรับสร้าง protobuf message
///
/// ตัวอย่าง:
/// ```
/// use protobuf_codec::message::MessageBuilder;
/// let bytes = MessageBuilder::new()
///     .add_int32(1, 42)
///     .add_string(2, "Alice")
///     .build();
/// ```
#[derive(Default)]
pub struct MessageBuilder {
    fields: Vec<u8>,
}

impl MessageBuilder {
    pub fn new() -> Self {
        MessageBuilder { fields: Vec::new() }
    }

    pub fn add_int32(mut self, field_number: u32, value: i32) -> Self {
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(value as u64)));
        self
    }

    pub fn add_int64(mut self, field_number: u32, value: i64) -> Self {
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(value as u64)));
        self
    }

    pub fn add_uint32(mut self, field_number: u32, value: u32) -> Self {
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(value as u64)));
        self
    }

    pub fn add_uint64(mut self, field_number: u32, value: u64) -> Self {
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(value)));
        self
    }

    /// Zigzag-encoded signed integer (sint32/sint64)
    pub fn add_sint32(mut self, field_number: u32, value: i32) -> Self {
        let encoded = zigzag_encode(value as i64);
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(encoded)));
        self
    }

    pub fn add_sint64(mut self, field_number: u32, value: i64) -> Self {
        let encoded = zigzag_encode(value);
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(encoded)));
        self
    }

    pub fn add_bool(mut self, field_number: u32, value: bool) -> Self {
        self.fields
            .extend(encode_field(field_number, &FieldValue::Varint(value as u64)));
        self
    }

    /// string field (length-delimited UTF-8 bytes)
    pub fn add_string(mut self, field_number: u32, value: &str) -> Self {
        self.fields.extend(encode_field(
            field_number,
            &FieldValue::Bytes(value.as_bytes().to_vec()),
        ));
        self
    }

    /// bytes field (arbitrary binary data)
    pub fn add_bytes(mut self, field_number: u32, value: Vec<u8>) -> Self {
        self.fields
            .extend(encode_field(field_number, &FieldValue::Bytes(value)));
        self
    }

    /// double field (fixed64, IEEE 754 little-endian)
    pub fn add_double(mut self, field_number: u32, value: f64) -> Self {
        self.fields.extend(encode_field(
            field_number,
            &FieldValue::Fixed64(value.to_le_bytes()),
        ));
        self
    }

    /// float field (fixed32, IEEE 754 little-endian)
    pub fn add_float(mut self, field_number: u32, value: f32) -> Self {
        self.fields.extend(encode_field(
            field_number,
            &FieldValue::Fixed32(value.to_le_bytes()),
        ));
        self
    }

    /// fixed32 field — uint32 เก็บ 4 bytes ตายตัว
    pub fn add_fixed32(mut self, field_number: u32, value: u32) -> Self {
        self.fields.extend(encode_field(
            field_number,
            &FieldValue::Fixed32(value.to_le_bytes()),
        ));
        self
    }

    /// fixed64 field — uint64 เก็บ 8 bytes ตายตัว
    pub fn add_fixed64(mut self, field_number: u32, value: u64) -> Self {
        self.fields.extend(encode_field(
            field_number,
            &FieldValue::Fixed64(value.to_le_bytes()),
        ));
        self
    }

    /// embedded message field: รับ MessageBuilder อีกตัวแล้วห่อด้วย length-delimited field
    pub fn add_embedded(mut self, field_number: u32, sub: MessageBuilder) -> Self {
        let sub_bytes = sub.build();
        self.fields
            .extend(encode_field(field_number, &FieldValue::Bytes(sub_bytes)));
        self
    }

    /// สร้าง raw bytes ของ message
    pub fn build(self) -> Vec<u8> {
        self.fields
    }
}
```

ตัวอย่างการใช้งาน `MessageBuilder`:

```rust
// สร้าง Person message แบบ fluent
let bytes = MessageBuilder::new()
    .add_int32(1, 42)                           // id = 42
    .add_string(2, "Alice")                      // name = "Alice"
    .add_string(3, "alice@example.com")          // email
    .add_bool(4, true)                           // active
    .add_sint32(5, -100)                         // score (zigzag)
    .add_double(6, 98.6)                         // temperature
    .build();

println!("Encoded {} bytes: {:02X?}", bytes.len(), bytes);
// output: Encoded 49 bytes: [08, 2A, 12, 05, 41, 6C, ...]
```

---

### ขั้นที่ 6: MessageDecoder

```rust
// ต่อใน src/message.rs

/// MessageDecoder: ถอดรหัส protobuf binary message เป็น list ของ (field_number, FieldValue)
pub struct MessageDecoder;

impl MessageDecoder {
    /// ถอดรหัส message bytes เป็น field list
    /// รองรับ unknown fields (เก็บไว้ as-is โดยไม่ throw error)
    /// รองรับ repeated fields (field_number เดียวกันปรากฏหลายครั้ง)
    pub fn decode(buf: &[u8]) -> Result<Vec<(u32, FieldValue)>, String> {
        let mut fields = Vec::new();
        let mut pos = 0;

        while pos < buf.len() {
            match decode_field(&buf[pos..]) {
                Some((field_num, value, consumed)) => {
                    fields.push((field_num, value));
                    pos += consumed;
                }
                None => {
                    return Err(format!(
                        "failed to decode field at byte offset {}",
                        pos
                    ));
                }
            }
        }

        Ok(fields)
    }

    /// ถอดรหัส embedded message ที่อยู่ใน FieldValue::Bytes
    pub fn decode_embedded(fv: &FieldValue) -> Result<Vec<(u32, FieldValue)>, String> {
        match fv {
            FieldValue::Bytes(data) => Self::decode(data),
            _ => Err("expected Bytes field value for embedded message".to_string()),
        }
    }
}
```

ตัวอย่าง nested message (Address อยู่ใน Person):

```rust
// encode nested message
let address = MessageBuilder::new()
    .add_string(1, "123 Rust Street")    // street
    .add_string(2, "Bangkok")            // city
    .add_int32(3, 10110);                // postal_code

let person = MessageBuilder::new()
    .add_int32(1, 1)
    .add_string(2, "Alice")
    .add_embedded(3, address)            // field 3 คือ Address message
    .build();

// decode
let fields = MessageDecoder::decode(&person).unwrap();
// fields[2] = (3, Bytes([...address bytes...]))
let addr_fields = MessageDecoder::decode_embedded(&fields[2].1).unwrap();
// addr_fields[0] = (1, Bytes(b"123 Rust Street"))
```

---

### ขั้นที่ 7: Schema Definition และ Validation

สร้าง schema struct ที่จำลอง `.proto` definition ใน Rust เพื่อ validate และ serialize เป็น JSON

```rust
// src/schema.rs
use std::collections::HashMap;
use crate::field::FieldValue;
use serde::{Deserialize, Serialize};
use serde_json::Value as JsonValue;

/// ชนิดของ field ใน schema (คล้าย .proto types)
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum FieldType {
    Int32, Int64, Uint32, Uint64,
    Sint32, Sint64,  // zigzag-encoded
    Bool,
    Fixed32, Fixed64,
    Float, Double,
    String, Bytes,
    Message,         // embedded message
}

/// นิยาม field หนึ่งตัวใน schema
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldDef {
    pub number: u32,           // field number (1 ถึง 536870911)
    pub name: String,          // ชื่อ field
    pub ftype: FieldType,      // ชนิดข้อมูล
    pub required: bool,        // true = validation error ถ้าไม่มี field นี้
}

/// Schema ของ protobuf message (คล้าย `message Foo { ... }` ใน .proto)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ProtoMessage {
    pub name: String,
    pub fields: Vec<FieldDef>,
}

impl ProtoMessage {
    pub fn validate(&self, fields: &[(u32, FieldValue)]) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        let field_map: HashMap<u32, &FieldDef> =
            self.fields.iter().map(|f| (f.number, f)).collect();

        // ตรวจว่า required fields ครบ
        for field_def in &self.fields {
            if field_def.required {
                let present = fields.iter().any(|(n, _)| *n == field_def.number);
                if !present {
                    errors.push(format!(
                        "required field '{}' (#{}) is missing",
                        field_def.name, field_def.number
                    ));
                }
            }
        }

        // ตรวจ wire type ของแต่ละ field ที่รู้จัก
        for (field_num, value) in fields {
            if let Some(def) = field_map.get(field_num) {
                let expected_wire = expected_wire_type(&def.ftype);
                let actual_wire = value.wire_type();
                if expected_wire != actual_wire {
                    errors.push(format!(
                        "field '{}' (#{}) has wrong wire type: expected {:?}, got {:?}",
                        def.name, field_num, expected_wire, actual_wire
                    ));
                }
            }
            // unknown fields → ปล่อยผ่านตาม protobuf spec
        }

        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }

    /// แปลง decoded fields เป็น JSON object ตาม schema
    pub fn to_json(&self, fields: &[(u32, FieldValue)]) -> JsonValue {
        // ... (ดู schema.rs สำหรับ implementation เต็ม)
    }
}
```

ตัวอย่างการนิยาม schema และใช้งาน:

```rust
let schema = ProtoMessage {
    name: "Person".to_string(),
    fields: vec![
        FieldDef { number: 1, name: "id".into(), ftype: FieldType::Int32, required: true },
        FieldDef { number: 2, name: "name".into(), ftype: FieldType::String, required: true },
        FieldDef { number: 3, name: "email".into(), ftype: FieldType::String, required: false },
        FieldDef { number: 4, name: "active".into(), ftype: FieldType::Bool, required: false },
    ],
};

let bytes = MessageBuilder::new()
    .add_int32(1, 42)
    .add_string(2, "Alice")
    .build();

let fields = MessageDecoder::decode(&bytes)?;

// Validate
schema.validate(&fields)?;

// Serialize to JSON
let json = schema.to_json(&fields);
println!("{}", serde_json::to_string_pretty(&json).unwrap());
```

---

### ขั้นที่ 8: Demo Binary

```rust
// src/main.rs
use protobuf_codec::message::{MessageBuilder, MessageDecoder};
use protobuf_codec::schema::{FieldDef, FieldType, ProtoMessage};

fn main() {
    println!("=== Protocol Buffer Codec Demo ===\n");

    let schema = ProtoMessage {
        name: "Person".to_string(),
        fields: vec![
            FieldDef { number: 1, name: "id".to_string(), ftype: FieldType::Int32, required: true },
            FieldDef { number: 2, name: "name".to_string(), ftype: FieldType::String, required: true },
            FieldDef { number: 3, name: "email".to_string(), ftype: FieldType::String, required: false },
            FieldDef { number: 4, name: "age".to_string(), ftype: FieldType::Int32, required: false },
        ],
    };

    let bytes = MessageBuilder::new()
        .add_int32(1, 42)
        .add_string(2, "Alice Rust")
        .add_string(3, "alice@rust.dev")
        .add_int32(4, 30)
        .build();

    println!("Encoded bytes ({} bytes): {:02X?}", bytes.len(), bytes);

    let fields = MessageDecoder::decode(&bytes).expect("decode failed");
    println!("\nDecoded fields:");
    for (n, v) in &fields {
        println!("  field #{}: {:?}", n, v);
    }

    match schema.validate(&fields) {
        Ok(()) => println!("\nValidation: PASS"),
        Err(errs) => {
            println!("\nValidation FAILED:");
            for e in errs { println!("  - {}", e); }
        }
    }

    let json = schema.to_json(&fields);
    println!("\nJSON output:\n{}", serde_json::to_string_pretty(&json).unwrap());
}
```

Output ของ demo binary:

```
=== Protocol Buffer Codec Demo ===

Encoded bytes (32 bytes): [08, 2A, 12, 0A, 41, 6C, 69, 63, 65, 20, 52, 75, 73, 74, 1A, 0E, 61, 6C, 69, 63, 65, 40, 72, 75, 73, 74, 2E, 64, 65, 76, 20, 1E]

Decoded fields:
  field #1: Varint(42)
  field #2: Bytes([65, 108, 105, 99, 101, 32, 82, 117, 115, 116])
  field #3: Bytes([97, 108, 105, 99, 101, 64, 114, 117, 115, 116, 46, 100, 101, 118])
  field #4: Varint(30)

Validation: PASS

JSON output:
{
  "age": 30,
  "email": "alice@rust.dev",
  "id": 42,
  "name": "Alice Rust"
}
```

---

## การทดสอบ (Testing)

ทดสอบระบบทั้งหมดด้วย 31 unit tests ครอบคลุมทุก module:

```rust
// ตัวอย่าง tests สำคัญ

// --- varint tests ---
#[test]
fn test_varint_roundtrip() {
    for v in [0u64, 1, 127, 128, 255, 300, 65535, 1 << 20, u64::MAX / 2] {
        let encoded = encode_varint(v);
        let (decoded, _) = decode_varint(&encoded).expect("decode failed");
        assert_eq!(decoded, v, "roundtrip failed for {v}");
    }
}

#[test]
fn test_varint_decode_partial_buffer() {
    let incomplete = vec![0x80u8]; // continuation bit set แต่ไม่มี byte ต่อ
    assert!(decode_varint(&incomplete).is_none());
}

// --- zigzag tests ---
#[test]
fn test_zigzag_roundtrip() {
    for v in [0i64, -1, 1, -100, 100, -32768, 32767, i64::MIN / 2, i64::MAX / 2] {
        let encoded = zigzag_encode(v);
        let decoded = zigzag_decode(encoded);
        assert_eq!(decoded, v, "zigzag roundtrip failed for {v}");
    }
}

// --- field encode/decode tests ---
#[test]
fn test_encode_decode_string_field() {
    let s = "สวัสดี Rust!";
    let bytes = encode_string_field(3, s);
    let (field_num, value, _) = decode_field(&bytes).unwrap();
    assert_eq!(field_num, 3);
    if let FieldValue::Bytes(b) = value {
        assert_eq!(String::from_utf8(b).unwrap(), s);
    }
}

// --- message tests ---
#[test]
fn test_builder_embedded_message() {
    let inner = MessageBuilder::new()
        .add_string(1, "inner_name")
        .add_int32(2, 99);
    let outer = MessageBuilder::new()
        .add_int32(1, 1)
        .add_embedded(2, inner)
        .build();

    let fields = MessageDecoder::decode(&outer).unwrap();
    let inner_fields = MessageDecoder::decode_embedded(&fields[1].1).unwrap();
    assert_eq!(inner_fields.len(), 2);
}

#[test]
fn test_builder_repeated_fields() {
    let bytes = MessageBuilder::new()
        .add_int32(1, 1)
        .add_int32(1, 2)
        .add_int32(1, 3)
        .build();
    let fields = MessageDecoder::decode(&bytes).unwrap();
    assert_eq!(fields.len(), 3);
    assert!(fields.iter().all(|(n, _)| *n == 1));
}

// --- schema tests ---
#[test]
fn test_schema_unknown_field_passthrough() {
    let schema = make_person_schema();
    let bytes = MessageBuilder::new()
        .add_int32(1, 7)
        .add_string(2, "Charlie")
        .add_int32(99, 9999)  // unknown field
        .build();
    let fields = MessageDecoder::decode(&bytes).unwrap();
    assert_eq!(fields.len(), 3);            // decode ครบ 3 fields
    assert!(schema.validate(&fields).is_ok()); // unknown ไม่ทำให้ fail
}
```

### ผลลัพธ์จาก `cargo test`

```
running 31 tests
test field::tests::test_decode_field_partial_returns_none ... ok
test field::tests::test_encode_decode_fixed64 ... ok
test field::tests::test_encode_decode_bytes_field ... ok
test field::tests::test_encode_decode_string_field ... ok
test field::tests::test_encode_decode_fixed32 ... ok
test message::tests::test_builder_int32_roundtrip ... ok
test message::tests::test_builder_embedded_message ... ok
test field::tests::test_encode_decode_varint_field ... ok
test message::tests::test_builder_multiple_fields ... ok
test message::tests::test_builder_repeated_fields ... ok
test message::tests::test_builder_string_roundtrip ... ok
test message::tests::test_fixed32_roundtrip ... ok
test message::tests::test_sint32_zigzag_roundtrip ... ok
test schema::tests::test_schema_to_json ... ok
test schema::tests::test_schema_json_serialization ... ok
test message::tests::test_fixed64_roundtrip ... ok
test schema::tests::test_schema_unknown_field_passthrough ... ok
test schema::tests::test_schema_validate_missing_required ... ok
test schema::tests::test_schema_validate_ok ... ok
test varint::tests::test_varint_decode_consumes_correct_bytes ... ok
test varint::tests::test_varint_decode_partial_buffer ... ok
test varint::tests::test_varint_encode_multibyte ... ok
test varint::tests::test_varint_encode_small ... ok
test varint::tests::test_varint_roundtrip ... ok
test varint::tests::test_zigzag_decode ... ok
test varint::tests::test_zigzag_roundtrip ... ok
test wire::tests::test_tag_encode_decode ... ok
test wire::tests::test_tag_field0_invalid ... ok
test wire::tests::test_tag_field2_len ... ok
test wire::tests::test_wire_type_from_u8 ... ok
test varint::tests::test_zigzag_encode ... ok

test result: ok. 31 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/protobuf_codec-5db4a1ca331b47e6)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests protobuf_codec

running 1 test
test src/message.rs - message::MessageBuilder (line 7) ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.11s
```

---

## จุดระวัง (Common Pitfalls)

### Pitfall 1: ใช้ `int32` แทน `sint32` สำหรับค่าลบ

```rust
// ❌ ผิด: ใช้ int32 เก็บค่าลบ — เปลือง 10 bytes
let bytes = MessageBuilder::new()
    .add_int32(1, -1)  // -1 as u64 = 0xFFFFFFFFFFFFFFFF → varint 10 bytes!
    .build();

// ✓ ถูก: ใช้ sint32 ถ้าค่ามักเป็นลบ
let bytes = MessageBuilder::new()
    .add_sint32(1, -1)  // zigzag(-1) = 1 → varint 1 byte
    .build();
```

`add_int32(-1)` ทำ `value as u64 = 0xFFFFFFFFFFFFFFFF` ซึ่งใช้ varint 10 bytes เต็มที่ ถ้าต้องการ encode ค่าลบแบบ compact ต้องใช้ `add_sint32()` เสมอ

### Pitfall 2: Default Value ไม่ถูก Encode

ตาม protobuf spec (proto3), default value จะ **ไม่ถูก encode** ลงใน message:

```rust
// ❌ อันตราย: ถ้า id = 0, field 1 จะไม่อยู่ใน encoded bytes
let bytes = MessageBuilder::new()
    .add_int32(1, 0)    // ถ้าเป็น proto3: field นี้อาจถูก strip
    .add_string(2, "")  // empty string ก็เช่นกัน
    .build();

// ในการ decode คุณจะได้ fields ว่าง และต้องตีความว่า field ไม่มี = ค่า default
// แต่ใน implementation ของเราจะ encode ทุก field ที่ถูก add() เสมอ
// (proto2 behaviour: ยังคง encode fields ทั้งหมด)
```

ระวัง: codec ของเรา encode ทุก field ที่ add() ซึ่งคล้าย proto2 แต่ protobuf library จริง (proto3) จะ skip fields ที่มีค่า default

### Pitfall 3: ลืม length prefix สำหรับ Len wire type

```rust
// ❌ ผิด: encode bytes โดยไม่มี length prefix
fn encode_bytes_wrong(data: &[u8]) -> Vec<u8> {
    data.to_vec()  // decoder จะไม่รู้ว่า bytes จบที่ไหน!
}

// ✓ ถูก: ต้องนำหน้าด้วย varint ของ length
fn encode_bytes_correct(data: &[u8]) -> Vec<u8> {
    let mut out = encode_varint(data.len() as u64);
    out.extend_from_slice(data);
    out
}
```

Len wire type ต้องนำหน้าด้วย varint บอกความยาว มิเช่นนั้น decoder จะ overread หรือ panic เนื่องจาก slice index out of bounds

### Pitfall 4: Field Number 0 ไม่ถูกต้อง

```rust
// ❌ ผิด: field number 0 ไม่ถูกต้องใน protobuf spec
let tag = Tag::new(0, WireType::Varint);
// encode ออกมาเป็น (0 << 3) | 0 = 0
// decode คืน None เพราะ field_number 0 เป็น reserved

// ✓ ถูก: field number เริ่มที่ 1 เสมอ
let tag = Tag::new(1, WireType::Varint);
```

Tag::decode() ของเราจะคืน `None` ถ้า field_number == 0 ป้องกัน panic แต่ถ้า field_number 0 ถูกส่งผ่าน encode_field โดยตรงจะทำให้ message ที่ encode ออกมา decode ไม่ได้

### Pitfall 5: Repeated Fields ต้องอ่านจาก field ที่ซ้ำกัน

```rust
// ❌ ผิด: สมมติว่า field number ใน Vec<(u32, FieldValue)> ไม่ซ้ำ
let fields = MessageDecoder::decode(&bytes).unwrap();
let id = fields.iter().find(|(n, _)| *n == 1).unwrap();
// ถ้าเป็น repeated field การใช้ find() จะได้แค่ occurrence แรก

// ✓ ถูก: ใช้ filter() เพื่อเก็บทุก occurrence
let ids: Vec<i32> = fields.iter()
    .filter(|(n, _)| *n == 1)
    .filter_map(|(_, v)| if let FieldValue::Varint(x) = v { Some(*x as i32) } else { None })
    .collect();
```

MessageDecoder::decode() คืน `Vec<(u32, FieldValue)>` ที่อาจมี field number เดิมหลายครั้ง (repeated fields) ต้องใช้ `filter()` ไม่ใช่ `find()`

### Pitfall 6: Unicode String กับ UTF-8 Safety

```rust
// ❌ อันตราย: ถ้า bytes ใน field ไม่ใช่ valid UTF-8
let raw = b"\xFF\xFE hello"; // invalid UTF-8
let bytes = encode_field(1, &FieldValue::Bytes(raw.to_vec()));
let fields = MessageDecoder::decode(&bytes).unwrap();
if let FieldValue::Bytes(b) = &fields[0].1 {
    let s = String::from_utf8(b.clone()).unwrap(); // PANIC! invalid UTF-8
}

// ✓ ถูก: ใช้ from_utf8_lossy() หรือ check ก่อน
let s = String::from_utf8_lossy(&b).into_owned();
// หรือ
let s = String::from_utf8(b.clone()).unwrap_or_else(|_| "<invalid utf-8>".to_string());
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# สร้าง optimized binary
cargo build --release

# ขนาดของ binary
ls -lh target/release/protobuf-codec
# ประมาณ 2-4 MB (รวม debug symbols)

# Strip symbols เพื่อลดขนาด
strip target/release/protobuf-codec
ls -lh target/release/protobuf-codec
# ลงไปประมาณ 300-500 KB
```

### ใช้เป็น Library

สำหรับใช้เป็น library ใน project อื่น ให้ตั้งค่าใน `Cargo.toml`:

```toml
# ใน project ที่จะใช้ codec นี้
[dependencies]
protobuf-codec = { path = "../protobuf-codec" }
# หรือถ้า publish ไปที่ crates.io
protobuf-codec = "0.1"
```

### Benchmark

เปรียบเทียบกับ `prost` (production protobuf library):

```bash
# ติดตั้ง cargo-criterion
cargo install cargo-criterion

# เพิ่มใน Cargo.toml
[dev-dependencies]
criterion = "0.5"
prost = "0.13"

# รัน benchmark
cargo criterion
```

ผลลัพธ์โดยประมาณ (message 100 fields):

| operation | codec ของเรา | prost |
|-----------|--------------|-------|
| encode    | ~800 ns      | ~300 ns |
| decode    | ~1.2 µs      | ~400 ns |
| memory    | ใกล้เคียงกัน  | ใกล้เคียงกัน |

codec ของเรา allocate `Vec<u8>` บ่อยกว่า ซึ่งเป็นสาเหตุหลักที่ช้ากว่า

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Packed Repeated Fields

ใน protobuf, repeated numeric fields สามารถใช้ "packed encoding" ที่มีประสิทธิภาพกว่า:

```protobuf
// .proto definition
message Scores {
  repeated int32 values = 1 [packed = true];
}
```

แทนที่จะ encode `(tag)(value)(tag)(value)(tag)(value)` สามารถทำเป็น `(tag)(len)(value)(value)(value)` ได้:

```
ปกติ 3 ค่า [1,2,3]:   [0x08, 0x01, 0x08, 0x02, 0x08, 0x03]  (6 bytes)
packed  3 ค่า [1,2,3]: [0x0A, 0x03, 0x01, 0x02, 0x03]          (5 bytes)
```

**งาน**: เพิ่ม method `add_packed_int32(field_number: u32, values: &[i32])` ใน `MessageBuilder` และ method `decode_packed_varint(fv: &FieldValue) -> Vec<u64>` ใน `MessageDecoder`

### แบบฝึกหัดที่ 2: Oneof Fields

`oneof` ใน protobuf บังคับให้มีได้แค่ field เดียวจาก group:

```protobuf
message Contact {
  oneof contact_type {
    string phone = 1;
    string email = 2;
  }
}
```

**งาน**: เพิ่ม `OneofGroup { fields: Vec<u32> }` ใน `ProtoMessage` schema และ validate ว่า oneof group มีค่าไม่เกิน 1 field ใน decoded message

### แบบฝึกหัดที่ 3: Proto Text Format Parser

นอกจาก binary format แล้ว protobuf มี text format สำหรับ debug:

```
id: 42
name: "Alice"
address {
  street: "123 Main St"
  city: "Bangkok"
}
```

**งาน**: เขียน `TextFormatParser` ที่ parse string แบบนี้ตาม schema แล้วแปลงเป็น `Vec<(u32, FieldValue)>` เพื่อส่ง encode ต่อ

### แบบฝึกหัดที่ 4: Proto File Parser (`.proto` file)

```protobuf
// person.proto
syntax = "proto3";

message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;
  bool active = 4;
}
```

**งาน**: เขียน parser ที่อ่าน `.proto` file แบบ subset ง่าย ๆ และสร้าง `ProtoMessage` struct จากไฟล์นั้น เพื่อให้สามารถ validate message โดยไม่ต้องเขียน schema ด้วยมือ

hint: ใช้ `nom` crate หรือ hand-rolled line parser

### แบบฝึกหัดที่ 5: gRPC Wire Format

gRPC ห่อ protobuf message ด้วย 5-byte frame header ก่อนส่ง:

```
[compression_flag: 1 byte][message_length: 4 bytes big-endian][protobuf_payload]
```

**งาน**: เพิ่ม `GrpcFrame` struct พร้อม `encode(compressed: bool, payload: &[u8]) -> Vec<u8>` และ `decode(buf: &[u8]) -> (bool, Vec<u8>)` ซึ่งจะเป็น building block สำหรับโปรเจค H07 (gRPC Service)

### แบบฝึกหัดที่ 6: Performance Optimization

ปัจจุบัน codec allocate `Vec<u8>` ใหม่ทุก field call ซึ่ง expensive:

```rust
// ปัจจุบัน: allocate Vec ใหม่ทุก call
pub fn encode_varint(mut value: u64) -> Vec<u8> { ... }

// ดีกว่า: เขียนลง buffer ที่รับมา
pub fn encode_varint_into(mut value: u64, buf: &mut Vec<u8>) { ... }
```

**งาน**:
1. เปลี่ยน `encode_varint` ให้รับ `&mut Vec<u8>` แทนคืน `Vec<u8>`
2. เปลี่ยน `encode_field` ให้รับ `&mut Vec<u8>`
3. Benchmark ว่าเร็วขึ้นเท่าไหร่ด้วย `criterion`

---

## แนวคิด Architecture เพิ่มเติม

### Comparison: Binary Formats

| Format | ขนาด (100 fields) | Decode speed | Schema required | Human-readable |
|--------|-------------------|--------------|-----------------|----------------|
| JSON   | ~2.5 KB           | ช้า (text parse)  | ไม่จำเป็น | ✓ |
| CBOR   | ~1.2 KB           | ปานกลาง | ไม่จำเป็น | ✗ |
| MessagePack | ~1.0 KB    | ปานกลาง | ไม่จำเป็น | ✗ |
| **Protobuf** | ~0.4 KB  | เร็ว | ✓ | ✗ |
| FlatBuffers | ~0.6 KB  | เร็วมาก (zero-copy) | ✓ | ✗ |

### Wire Format ของ Real Protobuf Message

สำหรับ message `{ id: 42, name: "Alice", active: true }` bytes จริง:

```
08          → tag: field 1 (id), wire type 0 (Varint)
2A          → value: 42
12          → tag: field 2 (name), wire type 2 (Len)
05          → length: 5 bytes
41 6C 69 63 65  → "Alice"
18          → tag: field 3 (active), wire type 0 (Varint)
01          → value: 1 (true)
total: 10 bytes
```

เทียบกับ JSON `{"id":42,"name":"Alice","active":true}` = 38 bytes — **เล็กกว่า 3.8 เท่า**

### การจัดการ Backward/Forward Compatibility

```
ผู้ส่ง (schema v2)     ผู้รับ (schema v1)
field 1: id = 1         field 1: id = 1         ✓ รองรับ
field 2: name = "A"     field 2: name = "A"      ✓ รองรับ
field 3: email = "a@b"  (unknown field)          ✓ เก็บไว้ pass-through

ผู้ส่ง (schema v1)     ผู้รับ (schema v2)
field 1: id = 1         field 1: id = 1          ✓ รองรับ
field 2: name = "A"     field 2: name = "A"      ✓ รองรับ
(ไม่มี email)           field 3: email = ""      ✓ ใช้ default value
```

นี่คือเหตุผลหลักที่ protobuf เป็น standard choice สำหรับ long-lived API

---

## สรุป

ในโปรเจคนี้เราสร้าง Protocol Buffer wire format codec ครบชุดจาก scratch โดยไม่พึ่ง library ภายนอก สิ่งที่ implement:

1. **`WireType` + `Tag`** — field tag system ที่เป็นหัวใจของ binary format
2. **LEB128 varint** — variable-length integer encoding ที่ทำให้ small values กิน byte น้อย
3. **Zigzag encoding** — optimization พิเศษสำหรับ signed integers ที่มักมีค่าเป็นลบ
4. **`FieldValue` enum** — type-safe representation ของ 4 wire types
5. **`MessageBuilder`** — fluent API สำหรับสร้าง message ด้วย method chaining
6. **`MessageDecoder`** — decoder ที่รองรับ repeated fields, nested messages, และ unknown field passthrough
7. **`ProtoMessage` schema** — validate decoded fields และ serialize เป็น JSON

### Pattern สำคัญที่ได้เรียน

- **Bit manipulation** — `<<`, `>>`, `&`, `|` สำหรับ pack/unpack ข้อมูลใน byte
- **Builder pattern** ที่คืน `Self` เพื่อทำ method chaining
- **Enum as discriminated union** — `FieldValue` เป็น tagged union ที่ type-safe
- **Protocol compatibility** — unknown field passthrough เป็น pattern ที่ทำให้ API evolution เป็นไปได้
- **Property-based testing mindset** — roundtrip tests ที่ทดสอบ property "decode(encode(x)) == x" แทนที่จะ hardcode expected values

### เชื่อมโยงไปโปรเจคถัดไป

โปรเจค H07 (gRPC Service) จะใช้ codec ที่เราสร้างนี้เป็น transport layer:

```
gRPC = HTTP/2 + protobuf framing + codec

HTTP/2 connection
    │
    ├── stream 1: POST /helloworld.Greeter/SayHello
    │      ├── [5-byte gRPC frame header]
    │      └── [protobuf encoded HelloRequest]
    │
    └── stream 1 response:
           ├── [5-byte gRPC frame header]
           └── [protobuf encoded HelloReply]
```

ทุก field ที่เรา encode/decode ใน codec นี้คือข้อมูลจริงที่วิ่งอยู่ใน production gRPC call ทั่วโลกทุกวัน

---

## ความเข้าใจลึกเรื่อง Binary Format

### การแยกวิเคราะห์ Bytes ด้วยมือ

ฝึก decode bytes ด้วยมือเพื่อทำความเข้าใจ format:

```
ข้อมูล: [0x08, 0x96, 0x01]

byte 0: 0x08 = 0000_1000
  3 bits ล่าง: 000 = wire type 0 (Varint)
  bits สูงกว่า: 00001 = field number 1
  → Tag { field: 1, wire: Varint }

byte 1: 0x96 = 1001_0110
  MSB = 1 → มี byte ต่อไป
  7 bits ล่าง = 001_0110 = 0x16 = 22 (bits 0-6 ของ value)

byte 2: 0x01 = 0000_0001
  MSB = 0 → byte สุดท้าย
  7 bits ล่าง = 000_0001 = 1 (bits 7-13 ของ value)

value = (1 << 7) | 22 = 128 + 22 = 150

ผลลัพธ์: field 1, Varint(150)
```

### การอ่าน Embedded Message

```
ข้อมูล: [0x1A, 0x05, 0x08, 0x01, 0x12, 0x01, 0x61]

byte 0: 0x1A = 0001_1010
  wire type: 010 = 2 (Len)
  field number: 00011 = 3
  → Tag { field: 3, wire: Len }

byte 1: 0x05 = length = 5 bytes
bytes 2-6: [0x08, 0x01, 0x12, 0x01, 0x61] = embedded message

  decode inner bytes:
  [0x08, 0x01] → field 1, Varint(1)
  [0x12, 0x01, 0x61] → field 2, Bytes([0x61]) = "a"
```

### Field Number Range ที่ควรรู้จัก

| Range | ความหมาย |
|-------|---------|
| 1 - 15 | encode tag ด้วย varint 1 byte (ใช้สำหรับ hot fields ที่ใช้บ่อย) |
| 16 - 2047 | encode tag ด้วย varint 2 bytes |
| 2048+ | encode tag ด้วย varint 3+ bytes |
| 19000 - 19999 | reserved สำหรับ Google internal |
| 536870912 (2^29) | สูงสุด, ไม่อนุญาตเกินนี้ |

ดังนั้นควรให้ fields ที่ใช้บ่อยที่สุดมี field number ต่ำ (1-15) เพื่อประหยัด bandwidth

---

## Integration Test: End-to-End

ทดสอบ encode/decode/validate แบบ end-to-end กับ message ที่ซับซ้อน:

```rust
// tests/integration.rs
use protobuf_codec::message::{MessageBuilder, MessageDecoder};
use protobuf_codec::schema::{FieldDef, FieldType, ProtoMessage};
use protobuf_codec::varint::{zigzag_decode, zigzag_encode};

#[test]
fn test_full_roundtrip_complex_message() {
    // สร้าง nested schema: Order มี LineItem[]
    let line_item_schema = ProtoMessage {
        name: "LineItem".to_string(),
        fields: vec![
            FieldDef { number: 1, name: "product_id".into(), ftype: FieldType::Uint32, required: true },
            FieldDef { number: 2, name: "quantity".into(), ftype: FieldType::Int32, required: true },
            FieldDef { number: 3, name: "price_cents".into(), ftype: FieldType::Int64, required: true },
        ],
    };

    let order_schema = ProtoMessage {
        name: "Order".to_string(),
        fields: vec![
            FieldDef { number: 1, name: "order_id".into(), ftype: FieldType::Uint64, required: true },
            FieldDef { number: 2, name: "customer".into(), ftype: FieldType::String, required: true },
            FieldDef { number: 3, name: "item".into(), ftype: FieldType::Message, required: false },
            FieldDef { number: 4, name: "discount_pct".into(), ftype: FieldType::Sint32, required: false },
        ],
    };

    // encode line item
    let item = MessageBuilder::new()
        .add_uint32(1, 1001)       // product_id
        .add_int32(2, 5)           // quantity
        .add_int64(3, 99_00)       // price 99.00 บาท (สตางค์)
        .build();

    // encode order ที่มี embedded item
    let order = MessageBuilder::new()
        .add_uint64(1, 20241201_001)    // order_id
        .add_string(2, "Alice")          // customer
        .add_bytes(3, item)              // embedded line_item (ใช้ add_bytes เพราะเรามี bytes แล้ว)
        .add_sint32(4, -10)              // discount 10% (zigzag)
        .build();

    println!("Order encoded: {} bytes", order.len());

    // decode
    let fields = MessageDecoder::decode(&order).unwrap();
    assert_eq!(fields.len(), 4);

    // validate order schema
    order_schema.validate(&fields).unwrap();

    // decode embedded item
    let item_fields = MessageDecoder::decode_embedded(&fields[2].1).unwrap();
    line_item_schema.validate(&item_fields).unwrap();

    // ตรวจค่า discount (zigzag)
    if let protobuf_codec::field::FieldValue::Varint(encoded_discount) = fields[3].1 {
        let discount = zigzag_decode(encoded_discount) as i32;
        assert_eq!(discount, -10);
    } else {
        panic!("expected discount field");
    }

    // serialize to JSON
    let order_json = order_schema.to_json(&fields);
    println!("Order JSON:\n{}", serde_json::to_string_pretty(&order_json).unwrap());
    assert_eq!(order_json["customer"], "Alice");
}

#[test]
fn test_binary_compatibility_with_known_bytes() {
    // hardcode expected bytes จาก reference implementation
    // message { field1: int32 = 150 }
    // ตรวจสอบกับ https://protobuf.dev/programming-guides/encoding/#simple
    let expected = vec![0x08u8, 0x96, 0x01];
    let actual = MessageBuilder::new().add_int32(1, 150).build();
    assert_eq!(actual, expected,
        "wire format ไม่ตรงกับ protobuf spec: expected {:02X?}, got {:02X?}",
        expected, actual);
}
```

---

## Debugging Tips

### วิธี Inspect Encoded Bytes

```rust
fn hexdump(bytes: &[u8]) {
    for (i, chunk) in bytes.chunks(16).enumerate() {
        print!("{:04x}  ", i * 16);
        for b in chunk {
            print!("{:02x} ", b);
        }
        // padding
        for _ in chunk.len()..16 {
            print!("   ");
        }
        print!(" |");
        for b in chunk {
            let c = if b.is_ascii_graphic() { *b as char } else { '.' };
            print!("{}", c);
        }
        println!("|");
    }
}

// ใช้งาน:
let bytes = MessageBuilder::new()
    .add_string(1, "hello")
    .add_int32(2, 42)
    .build();
hexdump(&bytes);

// output:
// 0000  0a 05 68 65 6c 6c 6f 10 2a              |..hello.*|
```

### ทดสอบ Wire Compatibility กับ Python

```python
# ติดตั้ง: pip install protobuf
import struct

def decode_varint(data, pos):
    result, shift = 0, 0
    while True:
        b = data[pos]
        pos += 1
        result |= (b & 0x7f) << shift
        shift += 7
        if not (b & 0x80):
            return result, pos

# decode bytes จาก Rust codec ของเรา
data = bytes([0x08, 0x96, 0x01])
tag_raw, pos = decode_varint(data, 0)
field_number = tag_raw >> 3
wire_type = tag_raw & 0x7
value, pos = decode_varint(data, pos)
print(f"field={field_number}, wire={wire_type}, value={value}")
# output: field=1, wire=0, value=150
```

---

**โปรเจคก่อนหน้า:** [Project H05: Layer 7 Load Balancer](project-h05-load-balancer.md) | **โปรเจคถัดไป:** [Project H07: gRPC Service](project-h07-grpc-service.md)
