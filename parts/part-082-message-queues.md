# Part 82: Message Queues: RabbitMQ/Kafka Integration

> โมดูล: การพัฒนาเว็บแอปพลิเคชัน (Web Development) | ระดับ: สูง | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างแม่นยำว่า **asynchronous messaging** แก้ปัญหาอะไรที่ synchronous call แบบ gRPC (Part 80) หรือ REST (Part 61-66) แก้ไม่ได้ — โดยเฉพาะการ "decouple" ผู้ส่ง (producer) กับผู้รับ (consumer) ทั้งในมิติของ**เวลา** (consumer ไม่ต้องออนไลน์ตอนที่ส่ง) และมิติของ**ความรู้จัก** (producer ไม่ต้องรู้ว่ามี consumer กี่ตัว หรือเป็นใคร) พร้อมสาธิตด้วย scenario "booking created" ต่อจาก Part 81 ที่ foreshadow ไว้
- อธิบายโมเดล **AMQP** (exchange, queue, binding, routing key) ได้ถูกต้องตามกลไกจริง ไม่ใช่ความเข้าใจผิดที่พบบ่อยว่า "publish ตรงไปที่ queue" — และเลือก exchange type (direct/topic/fanout) ให้เหมาะกับ use case ได้
- เขียน RabbitMQ producer และ consumer ด้วย `lapin` ที่ compile และรันได้จริง ครบวงจร: ประกาศ exchange/queue/binding, publish event `BookingCreated` (serde JSON ต่อยอด Part 57), และ consume ด้วย manual acknowledgment
- เข้าใจ **at-least-once delivery** ที่มาจาก manual ack/nack — รู้ว่าเกิดอะไรขึ้นถ้า consumer ตายกลางทางก่อน ack (message ถูก requeue) และทำไม consumer **ต้อง idempotent** เสมอ (ต่อยอดแนวคิด Idempotency-Key จาก Part 78 หัวข้อ 78.7 มาใช้กับการ consume message แทน HTTP request)
- ตั้งค่า **Dead-Letter Queue (DLQ)** ให้ message ที่ประมวลผลไม่สำเร็จซ้ำ ๆ ไม่วนลูปตายอยู่ใน queue ตลอดไป แต่ถูกส่งไปที่ปลายทางแยกสำหรับตรวจสอบ/แก้ไข
- อธิบายความแตกต่างเชิงสถาปัตยกรรมของ **Kafka** เทียบกับ RabbitMQ (topic/partition/consumer group, log-based retention ที่ไม่ลบ message ตอน consume เทียบกับ queue-based ที่ลบตอน ack) และเขียน producer/consumer ด้วย `rdkafka` ที่ partition ด้วย key จริง พร้อมเข้าใจพฤติกรรม consumer group rebalancing เมื่อ scale ขึ้น/ลง
- เลือกได้อย่างมีเหตุผลระหว่าง RabbitMQ, Kafka, และ PostgreSQL `LISTEN`/`NOTIFY` (Part 71) โดยไม่ไปไขว่คว้า Kafka เป็นค่าเริ่มต้นทั้งที่ requirement จริงต้องการแค่เครื่องมือที่เรียบง่ายกว่า

## ความรู้ที่ต้องมีมาก่อน

- **Part 81 (Microservices Architecture ด้วย Rust)**: บทนั้น foreshadow ไว้ว่า `booking-service` เมื่อสร้าง booking สำเร็จ ควรมีทางให้ service อื่น ๆ (notification, analytics, inventory-sync) รับรู้เหตุการณ์นี้แบบไม่ผูกกันตรง ๆ — บทนี้คือคำตอบเต็มรูปแบบของสิ่งที่ foreshadow ไว้นั้น เราจะ implement `BookingCreated` event ให้เดินจริงผ่าน message queue ในหัวข้อ 82.12 (capstone)
- **Part 80 (gRPC ด้วย Tonic)**: บทนี้ใช้ gRPC เป็นตัวอย่างของ synchronous service-to-service call เพื่อเทียบกับ asynchronous messaging ในหัวข้อ 82.1 — ต้องเข้าใจว่า gRPC call ทำงานแบบ request-response ที่ทั้งสองฝั่งต้องออนไลน์พร้อมกัน (จาก Part 80 หัวข้อ 80.1) ก่อนจึงจะเห็นว่า message queue แก้ปัญหาอะไรที่ gRPC แก้ไม่ได้
- **Part 71 (SQLx: Queries, Migrations, Connection Pooling)**: บทนั้นสอน `sqlx::postgres::PgListener` สำหรับรับ `NOTIFY` จาก PostgreSQL ไว้ในระดับ awareness และบอกไว้ล่วงหน้าว่า Part 82 คือ message queue เต็มรูปแบบที่ "หนักกว่า" — บทนี้จะเทียบทั้งสองแนวทางกันตรง ๆ ในหัวข้อ 82.11 พร้อมอธิบายข้อจำกัดสำคัญของ `LISTEN`/`NOTIFY` ที่ Part 71 ไม่ได้ลงรายละเอียด
- **Part 57 (Serialization: Serde เบื้องต้น)**: `BookingCreated`, `TicketSold` และทุก event struct ในบทนี้ใช้ `#[derive(Serialize, Deserialize)]` แล้ว serialize เป็น JSON ก่อนส่งเข้า queue — ถ้าจุดไหนไม่แน่นเรื่อง serde attribute ให้กลับไปอ่าน Part 57 ก่อน
- **Part 78 (RESTful API Design Best Practices)**: หัวข้อ 78.7 ของบทนั้น implement `Idempotency-Key` สำหรับป้องกัน HTTP request ซ้ำ — บทนี้หัวข้อ 82.5 จะหยิบ**แนวคิดเดียวกัน**มาใช้กับสถานการณ์ที่ต่างออกไปโดยสิ้นเชิง: การป้องกัน**message ซ้ำ**ที่ consumer อาจเห็นมากกว่าหนึ่งครั้งจาก at-least-once delivery
- **Part 48-50 (Tokio Runtime, Networking, Sync)**: `lapin` และ `rdkafka` (ฝั่ง async) ทั้งคู่วิ่งอยู่บน Tokio runtime และใช้ `Stream`/`async fn` ตามแนวทางที่ Part 48-50 สอนไว้ — โค้ด consumer ในบทนี้ใช้ `while let Some(x) = stream.next().await` ตรงตามรูปแบบที่ Part 50 แนะนำ
- **Part 39-40 (Send/Sync, Shared State)**: capstone ในหัวข้อ 82.12 เก็บ `lapin::Channel` ไว้ใน `Arc<AppState>` เพื่อแชร์ข้าม request handler ของ Axum เหมือนที่ Part 39-40 สอนเรื่อง `Arc`/`Mutex` สำหรับ state ที่แชร์ข้าม task
- **Part 62-66 (Axum)**: capstone ใช้ Axum handler, `State` extractor, และ `Router` ตามรูปแบบพื้นฐานที่ Part 62-64 สอนไว้ ไม่สอนพื้นฐาน Axum ซ้ำในบทนี้

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

ก่อนเขียนบทนี้ ผู้เขียนตรวจสอบสภาพแวดล้อมก่อนว่ามีอะไรให้ใช้จริงได้บ้าง เพราะ message queue ต้องมี broker จริงถึงจะทดสอบ end-to-end ได้ ผลตรวจสอบและแนวทางที่เลือก:

- `docker` มีติดตั้งอยู่ในสภาพแวดล้อมนี้ และหลังจากสั่ง `dockerd` ให้ทำงาน (daemon ไม่ได้ start มาให้อัตโนมัติ) ก็สามารถ pull image และรัน container ได้จริงตามปกติ — **นี่คือเส้นทางหลักที่ใช้ตรวจสอบทั้งบท**
- รัน RabbitMQ จริงด้วย image `rabbitmq:3-management-alpine` (container ชื่อ `rmq-scratch`, map port `5673:5672` เพื่อไม่ชนกับ agent อื่นที่อาจใช้ port 5672 มาตรฐานอยู่)
- รัน Kafka จริงด้วย image ทางการ `apache/kafka:3.7.0` ในโหมด **KRaft** (ไม่ต้องพึ่ง ZooKeeper แยก ซึ่งเป็นโหมดที่ Kafka รุ่นใหม่แนะนำ) map port `9092:9092` ตรงกับค่า `advertised.listeners=PLAINTEXT://localhost:9092` ที่ image กำหนดไว้ในตัว (ถ้า map port ไม่ตรงกับค่านี้ client จะต่อ bootstrap ได้ แต่หลุดตอน fetch metadata เพราะ broker แจ้ง address คืนมาไม่ตรงกับที่ client เข้าถึงได้จริง — เป็นกับดักที่เจอเองระหว่างตั้งค่า)
- สร้าง scratch cargo project สามตัวแยกไว้นอก repo ทั้งหมด (ลบทิ้งหลังเขียนบทนี้เสร็จ ตามกฎของ workflow): `rmq_lab` (ทดสอบ `lapin` ทุก scenario), `kafka_lab` (ทดสอบ `rdkafka` ทุก scenario), และ `capstone_booking` (Axum server + consumer แยก process สำหรับ capstone หัวข้อ 82.12)
- **ทุกตัวอย่างโค้ดหลักในบทนี้ compile และรันจริง เชื่อมต่อ broker จริง** ไม่มีการแต่ง output ขึ้นเอง — ผลลัพธ์ที่แสดงในบทนี้ (partition ที่ได้, offset, `redelivered=true`, เนื้อหา `x-death` header, ลำดับ partition assignment ตอน rebalance ฯลฯ) **คัดลอกมาจาก stdout ของโปรแกรมที่รันจริงในสภาพแวดล้อมนี้ทั้งหมด**
- อุปสรรคเดียวที่เจอจริงคือการ compile `rdkafka` (ซึ่งพึ่ง `librdkafka` ที่เป็นไลบรารี C) — sandbox นี้ไม่มี `libcurl4-openssl-dev` ให้ติดตั้งผ่าน `apt` ได้ (mirror ปฏิเสธการดึงแพคเกจนี้โดยเฉพาะ) ทำให้ compile ครั้งแรกพังด้วย `fatal error: curl/curl.h: No such file or directory` เพราะซอร์สของ librdkafka `#include <curl/curl.h>` แบบไม่มีเงื่อนไขไม่ว่าจะเปิดใช้ curl feature หรือไม่ — วิธีแก้คือเปิด cargo feature `curl-static` ของ `rdkafka` ให้มันดึงซอร์ส curl มา build เองผ่าน `curl-sys` แทนที่จะพึ่ง header ของระบบ (รายละเอียดอยู่ในหัวข้อ 82.8) หลังแก้จุดนี้ทุกอย่างก็ compile และรันได้จริงตามปกติ
- ปิดและลบ container/binary/cargo project ทั้งหมดหลังเขียนบทนี้เสร็จแล้ว ไม่มีสิ่งใดหลงเหลือค้างอยู่ในสภาพแวดล้อมถาวร

## เนื้อหา

### 82.1 ทำไมต้อง Asynchronous Messaging: เมื่อ Synchronous Call แบบ gRPC ไม่พอ

#### ทวนสถานการณ์จาก Part 81: "booking created" ต้องไปถึงใครบ้าง

Part 81 วาง scenario ไว้ว่า `booking-service` เมื่อสร้างการจองสำเร็จ ในระบบจริงมักมีงานอื่นที่ "ควรจะเกิดขึ้นตามมา" อีกหลายอย่างที่**ไม่เกี่ยวกับ transaction การจองเลย**:

- `notification-service` ต้องส่งอีเมลยืนยันให้ลูกค้า
- `analytics-service` ต้องบันทึกสถิติยอดจองไว้ทำ dashboard
- `inventory-sync-service` ต้องอัปเดตจำนวนที่นั่งคงเหลือไปยังระบบภายนอกอื่น (เช่น partner ที่ขายตั๋วร่วม)

ถ้าใช้แนวทางเดียวกับ Part 80 (gRPC) ทั้งหมด `booking-service` จะต้องเขียนโค้ดประมาณนี้ (นี่คือโค้ด**สมมติ**เพื่อชี้ปัญหา ไม่ใช่โค้ดที่แนะนำให้ทำจริง):

```rust
// ตัวอย่างแนวทาง "ผิดที่ควรหลีกเลี่ยง" — เพื่อชี้ให้เห็นปัญหาของ synchronous fan-out
async fn create_booking_the_synchronous_way(
    req: CreateBookingRequest,
    notification_client: &mut NotificationClient<Channel>,
    analytics_client: &mut AnalyticsClient<Channel>,
    inventory_client: &mut InventorySyncClient<Channel>,
) -> Result<Booking, Status> {
    let booking = save_booking_to_db(&req).await?; // สร้าง booking สำเร็จแล้ว

    // ต้องเรียกทีละตัว รอทีละตัว และต้องรู้จัก client ของทุก service ที่ "อาจ" สนใจ event นี้
    notification_client.send_confirmation(booking.clone().into()).await?;
    analytics_client.record_booking(booking.clone().into()).await?;
    inventory_client.sync_seats(booking.clone().into()).await?;

    Ok(booking)
}
```

โค้ดนี้มีปัญหาเชิงสถาปัตยกรรมสามชั้นที่ลึกกว่าที่ตาเห็น:

**ปัญหาที่ 1 — temporal coupling (ผูกกันด้วยเวลา):** gRPC call ของ Part 80 เป็น request-response แบบ synchronous เสมอ — ฝั่งเรียกต้อง**รอ**คำตอบก่อนไปทำงานต่อ ซึ่งหมายความว่า `notification-service`, `analytics-service`, และ `inventory-sync-service` **ต้องออนไลน์พร้อมกันทั้งหมดในขณะที่ booking-service เรียก** ถ้า `analytics-service` deploy ใหม่อยู่พอดีแล้วยังไม่ up (downtime สัก 10 วินาที) การจอง booking ทั้งคำขอจะ fail หรือค้างรอ timeout ทั้งที่ analytics ไม่ใช่ส่วนสำคัญของการจองเลย — งานที่ "ไม่จำเป็นต้อง real-time" กลับกลายเป็นจุดที่ทำให้ core flow ล่มไปด้วย

**ปัญหาที่ 2 — knowledge coupling (ผูกกันด้วยความรู้จัก):** `booking-service` ต้อง `use` client type ของทุก service ที่สนใจ event นี้ ต้องรู้ที่อยู่ (endpoint) ของทุกตัว ต้อง handle error ของทุกตัวแยกกัน — ถ้าอนาคตมี `fraud-detection-service` ตัวใหม่อยากรับรู้ event "booking created" ด้วย ก็ต้องกลับไปแก้โค้ด `booking-service` เพิ่ม client ตัวใหม่ ทั้งที่ `booking-service` ไม่ควรต้องรู้จักหรือสนใจการมีอยู่ของ `fraud-detection-service` เลยด้วยซ้ำ

**ปัญหาที่ 3 — latency stacking:** เวลารวมของ request คือผลรวมของเวลาที่ทุก downstream call ใช้ (หรือแย่กว่านั้นถ้า sequential ไม่ parallel) ลูกค้าที่กด "จองตั๋ว" ต้องรอ `analytics-service` เขียน log เสร็จก่อนถึงจะได้รับคำตอบ ทั้งที่ analytics ไม่มีผลอะไรกับการจองของลูกค้าเลย

#### ทางแก้: publish event แล้วให้ทุกคนที่สนใจไปรับเอง

Asynchronous messaging แก้ปัญหาทั้งสามข้อด้วยแนวคิดเดียว: **แยกการ "ทำให้สำเร็จ" (booking ถูกสร้าง) ออกจากการ "แจ้งให้คนอื่นรู้" (notify)** อย่างสิ้นเชิง โดย `booking-service`:

1. สร้าง booking ให้สำเร็จก่อน (เขียนลง PostgreSQL ตามแนวทาง Part 70-71)
2. **publish event เดียว** ชื่อ `booking.created` เข้า message queue แล้วจบหน้าที่ตรงนั้น — ไม่รอใครตอบ ไม่รู้จักใครเลย

ฝั่ง consumer (`notification-service`, `analytics-service`, `inventory-sync-service`, หรือ `fraud-detection-service` ที่จะเพิ่มมาทีหลัง) แต่ละตัวมาสมัครรับ event นี้**ด้วยตัวเอง** โดย `booking-service` ไม่ต้องรู้ด้วยซ้ำว่ามี consumer กี่ตัว หรือมีใครเพิ่มเข้ามาใหม่บ้าง — นี่คือ **decoupling ทั้งในมิติของเวลา (temporal) และมิติของความรู้จัก (spatial/knowledge)**:

- **Decouple ในมิติเวลา**: consumer ไม่จำเป็นต้องออนไลน์ตอนที่ producer ส่ง message — message ถูกเก็บไว้ใน queue รอจนกว่า consumer จะมาต่อ (broker คือ buffer ที่รับผิดชอบเก็บ message แทน) `booking-service` จึงไม่ต้องรอ ไม่ต้องแคร์ว่า downstream พร้อมหรือยัง
- **Decouple ในมิติความรู้จัก**: `booking-service` publish ไปที่ "จุดกลาง" (exchange ใน RabbitMQ หรือ topic ใน Kafka) โดยไม่รู้จักและไม่ต้องระบุ consumer ปลายทางเลย จะมี consumer ใหม่มาสมัครเพิ่มกี่ตัวก็ได้โดยไม่ต้องแก้โค้ด `booking-service` แม้แต่บรรทัดเดียว

ข้อพิสูจน์จริงของการ decouple ในมิติเวลา — ทดลองจริงในหัวข้อ 82.12 (capstone) ของบทนี้: เราจะยิง HTTP request สร้าง booking **ตอนที่ `notification-service` ยังไม่ได้ start เลย** แล้วค่อยไป start `notification-service` ทีหลัง และพิสูจน์ว่า message ยังรอคอยอยู่ใน queue ครบถ้วน ไม่หายไปไหน — ผลลัพธ์จริงจากการรันมีดังนี้ (คัดลอกจาก stdout ตรง ๆ ไม่มีการแต่ง):

```
=== ยิง booking request ตอนที่ notification-service ยังไม่รันเลย ===
{"booking_id":"38e67c61-2dd4-4412-a52c-7332bcc2fbec","status":"confirmed"}

=== message ค้างอยู่ใน queue (ไม่มี consumer ต่ออยู่เลย) ===
notification.booking_created.dlq	0
notification.booking_created	1

=== เพิ่งค่อยสตาร์ท notification-service ตอนนี้ (หลังผ่านไปพักหนึ่ง) ===
[notification-service] พร้อมรอ event 'booking.created' ...
[notification-service] would send confirmation email to 'decouple-demo@example.com' -> งาน 'Decoupling Demo' จำนวน 3 ที่นั่ง (booking_id=38e67c61-2dd4-4412-a52c-7332bcc2fbec)
```

สังเกตว่า `rabbitmqctl list_queues` (คำสั่งดู queue ของ RabbitMQ) รายงานว่ามี **1 message** ค้างอยู่ใน queue `notification.booking_created` ทั้งที่ยังไม่มี consumer ใดต่ออยู่เลย — พอ `notification-service` start ขึ้นมา มันก็รับ message เก่านั้นได้ทันทีโดยไม่มีอะไรสูญหาย นี่คือสิ่งที่ synchronous gRPC call ทำไม่ได้เลย (ถ้า `booking-service` เรียก gRPC ไปยัง `notification-service` ตอนที่มันยังไม่ start request นั้นจะ fail ทันที ไม่มีทางรอได้)

#### สรุปเทียบสองแนวทางแบบตาราง

| มิติ | Synchronous call (gRPC, Part 80 / REST, Part 61-66) | Asynchronous messaging (บทนี้) |
|---|---|---|
| ใครต้องออนไลน์ตอนสื่อสาร | ทั้งสองฝั่งต้องออนไลน์พร้อมกัน | แค่ broker ต้องออนไลน์ — consumer มาทีหลังได้ |
| ผู้ส่งรู้จักผู้รับไหม | รู้จักตรง ๆ (ต้องมี client type, endpoint address) | ไม่รู้จักเลย (publish ไปที่ exchange/topic กลาง) |
| จำนวนผู้รับ | คงที่ ต้องแก้โค้ดผู้ส่งถ้าจะเพิ่ม | เพิ่ม/ลดได้อิสระ ไม่กระทบโค้ดผู้ส่ง |
| ผลลัพธ์กลับมาทันทีไหม | ได้ (request-response) | ไม่ได้ (fire-and-forget ที่มีการันตีการส่งถึง) |
| เหมาะกับ | ต้องรู้ผลก่อนไปทำงานต่อ (เช่น เช็คสต๊อกก่อนยืนยันคำสั่งซื้อ) | งานที่ "เกิดแล้ว ใครสนใจไปทำต่อเอง" (เช่น แจ้งเตือน, sync ข้อมูล, analytics) |

> **หมายเหตุสำคัญที่ต้องชัดเจนตั้งแต่ต้นบท**: message queue **ไม่ได้มาแทน gRPC หรือ REST** ในทุกสถานการณ์ Part 80 บอกไว้แล้วว่า gRPC เหมาะกับกรณีที่ต้อง**รอผลลัพธ์กลับมาใช้ทันที** (เช่น "เช็คสต๊อกก่อนยืนยันคำสั่งซื้อ" — ต้องรู้ผลก่อนตอบลูกค้า) ส่วน message queue เหมาะกับงานที่เป็น **"เกิดเหตุการณ์นี้แล้ว ใครสนใจก็ไปทำอะไรต่อเอง"** (fire-and-forget แบบมีการันตีการส่งถึง) ทั้งสองแนวทางมักอยู่ในระบบเดียวกันได้ — `booking-service` อาจเรียก gRPC ไปที่ `payment-service` แบบ synchronous เพื่อตัดเงิน (ต้องรู้ผลก่อนยืนยัน booking) แล้วค่อย publish event แบบ asynchronous ไปที่ `notification-service` (ไม่ต้องรอผล)

### 82.2 AMQP Fundamentals: Exchange, Queue, Binding, Routing Key

RabbitMQ implement โปรโตคอลชื่อ **AMQP (Advanced Message Queuing Protocol)** ก่อนจะเขียนโค้ดสักบรรทัด ต้องเข้าใจโมเดลของ AMQP ให้ถูกก่อน เพราะเป็นจุดที่คนพึ่งเริ่มเข้าใจผิดบ่อยที่สุด

#### ความเข้าใจผิดที่พบบ่อยที่สุด: "publish ตรงไปที่ queue"

คนที่มาจากพื้นฐาน queue แบบง่าย ๆ (เช่น คิดว่า queue คือ "กล่องที่มีชื่อ ใส่ของเข้าไปแล้วมีคนหยิบออก") มักคาดหวังว่าโค้ดจะหน้าตาประมาณ `publish("my_queue", message)` — ตรงไปที่ queue ที่ระบุชื่อไว้เลย **แต่ AMQP ไม่ทำงานแบบนั้น** โมเดลจริงของ AMQP คือ:

```
Producer --publish--> Exchange --routing rule--> Queue(s) <--consume-- Consumer
```

**Producer ไม่เคย publish ตรงไปที่ queue** Producer publish ไปที่ **exchange** เสมอ (ระบุชื่อ exchange + routing key) แล้ว **exchange เป็นคนตัดสินใจ**ว่าจะส่ง message นี้ไปยัง queue ไหนบ้าง ตาม**กฎการ binding** ที่ผูก queue เข้ากับ exchange ไว้ล่วงหน้า ถ้าไม่มี queue ใด bind ไว้ตรงกับ routing key เลย message นั้นจะ**หายไปเงียบ ๆ** (ไม่มี error ใด ๆ บอก เพราะ exchange ทำหน้าที่แค่ route ไม่ใช่เก็บ) — นี่คือกับดักจริงที่จะพูดถึงในหัวข้อ "กับดักที่พบบ่อย" ท้ายบท

องค์ประกอบทั้งสี่ของโมเดลนี้:

- **Exchange**: จุดรับ message เข้ามาจาก producer มีหน้าที่**ตัดสินใจว่าจะ route ไปที่ queue ไหน** ตาม type ของมัน (direct/topic/fanout/headers) — exchange **ไม่เก็บ message ไว้เลย** ถ้า route ไม่ได้ก็หายไปทันที
- **Queue**: ที่เก็บ message จริง ๆ รอให้ consumer มาหยิบไปประมวลผล — เป็นจุดเดียวในโมเดลนี้ที่ message ถูกเก็บไว้จริง (buffer)
- **Binding**: กฎที่บอกว่า "queue นี้สนใจ message จาก exchange นี้ ที่มี routing key ตรงกับรูปแบบนี้" — เป็นความสัมพันธ์ระหว่าง exchange กับ queue ประกาศแยกจากทั้งสองฝั่ง
- **Routing key**: string ที่ producer แนบไปกับ message ตอน publish ใช้เป็นเงื่อนไขให้ exchange ตัดสินใจ route (ความหมายของมันขึ้นกับ exchange type)

#### Exchange types: direct, topic, fanout — เลือกให้ตรงกับ use case

| Exchange type | กลไก routing | Use case ที่เหมาะ |
|---|---|---|
| **direct** | ส่งไปยัง queue ที่ binding key **ตรงกับ routing key แบบเป๊ะ ๆ** เท่านั้น | แยกงานตาม category ที่ตายตัว เช่น `order.region.th` vs `order.region.us` ไปคนละ queue เพื่อให้ worker แต่ละ region จัดการแยกกัน |
| **topic** | ส่งไปยัง queue ที่ binding pattern ตรงกับ routing key แบบมี wildcard: `*` แทนหนึ่งคำ, `#` แทนกี่คำก็ได้ | ระบบ event ที่มีหลาย "ระดับความสนใจ": queue หนึ่งอยากรับ**ทุก** event ของ booking (`booking.#`) อีก queue อยากรับ**แค่** event การยกเลิก (`booking.cancelled`) จาก exchange เดียวกัน |
| **fanout** | **ไม่สนใจ routing key เลย** ส่ง message ไปยัง**ทุก queue** ที่ bind กับ exchange นี้ | Broadcast แบบ "ทุกคนต้องรู้" เช่น audit-log ที่ทุก service ที่สนใจต้องได้ event เดียวกันครบทุกตัว ไม่มีการกรอง |

*(ยังมี **headers exchange** ที่ route ตาม header attribute ของ message แทน routing key — เช่น bind queue ด้วยเงื่อนไข `{"region": "th", "priority": "high"}` แล้ว exchange จะจับคู่ตาม header ที่ message แนบมาแทนที่จะดู routing key เลย มีประโยชน์เมื่อเงื่อนไข routing ซับซ้อนกว่าที่ string pattern ของ topic exchange จะแสดงออกได้ แต่ใช้น้อยกว่า direct/topic/fanout มากในทางปฏิบัติ เพราะ topic exchange มักครอบคลุมเงื่อนไขที่ต้องใช้ได้อยู่แล้ว บทนี้จะไม่ลงรายละเอียดต่อ)*

#### ภาพรวมโมเดล AMQP ที่ใช้ตลอดบทนี้

```
                                    ┌─────────────────────────────┐
                                    │  Exchange: booking.events    │
   booking-service ──publish──────▶│  (type: topic)                │
   (routing key:                   └───────────┬───────────────────┘
    "booking.created")                         │
                                    binding: routing key                binding: routing key
                                    ตรงกับ "booking.created"             ตรงกับ "booking.#" (ทุก event)
                                                │                                    │
                                                ▼                                    ▼
                                  ┌──────────────────────────┐      ┌──────────────────────────────┐
                                  │ Queue:                    │      │ Queue:                        │
                                  │ notification.booking_     │      │ analytics.all_booking_events  │
                                  │ created                   │      │                                │
                                  └────────────┬──────────────┘      └───────────────┬────────────────┘
                                               │ consume                              │ consume
                                               ▼                                      ▼
                                   notification-service                    analytics-service
                                   (ไม่รู้จัก booking-service เลย)          (ไม่รู้จัก booking-service เลย)
```

แผนภาพนี้คือโครงสร้างที่ capstone ในหัวข้อ 82.12 และแบบฝึกหัดข้อ 2 ท้ายบทจะ implement จริง สังเกตว่า `booking-service` เห็นแค่กล่อง "Exchange" กล่องเดียว มันไม่รู้เลยว่ามี queue กี่ใบ bind อยู่ หรือมี consumer กี่ตัวรออยู่ปลายทาง — สอดคล้องกับหลักการ decoupling ที่อธิบายไว้ในหัวข้อ 82.1 ทุกประการ

capstone ของบทนี้ (หัวข้อ 82.12) เลือกใช้ **topic exchange** ชื่อ `booking.events` เพราะเข้ากับสถานการณ์จริงที่สุด: ในระบบจริงอาจมี event หลายแบบ (`booking.created`, `booking.cancelled`, `booking.updated`) ผ่าน exchange เดียวกัน แล้วให้ consumer แต่ละตัวเลือก bind กับ pattern ที่ตัวเองสนใจ — `notification-service` อาจ bind แค่ `booking.created` (สนใจแค่ตอนสร้างใหม่) ในขณะที่ `analytics-service` อาจ bind `booking.#` (สนใจทุกอย่างที่เกิดกับ booking)

### 82.3 lapin: Setup, ประกาศ Exchange/Queue/Binding, และ Publish Event จริง

`lapin` คือ AMQP client library ของ Rust ที่เป็น **pure-Rust implementation ทั้งหมด** (ไม่พึ่งไลบรารี C ใด ๆ) — ประเด็นนี้จะสำคัญตอนเทียบกับ `rdkafka` ในหัวข้อ 82.8 ที่ต้องพึ่งไลบรารี C

#### Cargo.toml

```toml
[dependencies]
lapin = "2.5"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
futures-lite = "2"       # ให้ StreamExt::next() สำหรับวน consume message
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
```

#### กำหนด event struct: `BookingCreated`

ต่อยอดจาก Part 57 struct นี้ต้อง derive ทั้ง `Serialize` และ `Deserialize` เพราะ producer จะ serialize เป็น JSON ก่อนส่ง และ consumer จะ deserialize กลับมาเป็น struct เดิม:

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct BookingCreated {
    event_id: Uuid,               // id ของ "เหตุการณ์" นี้เอง (คนละอันกับ booking_id)
    booking_id: Uuid,
    customer_email: String,
    event_name: String,
    seats: u32,
    created_at: chrono::DateTime<chrono::Utc>,
}
```

สังเกตว่ามีทั้ง `event_id` และ `booking_id` แยกกัน — `booking_id` คือ id ของ resource (การจอง) ส่วน `event_id` คือ id ของ "การเกิดเหตุการณ์นี้ครั้งนี้" ซึ่งจะสำคัญมากในหัวข้อ 82.5 ตอนพูดถึง idempotent consumer (ใช้ `event_id` เป็น key สำหรับ dedupe การประมวลผลซ้ำ)

#### Producer เต็มรูปแบบ

```rust
use lapin::{
    options::{BasicPublishOptions, ExchangeDeclareOptions, QueueBindOptions, QueueDeclareOptions},
    types::FieldTable,
    BasicProperties, Connection, ConnectionProperties, ExchangeKind,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct BookingCreated {
    event_id: Uuid,
    booking_id: Uuid,
    customer_email: String,
    event_name: String,
    seats: u32,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "amqp://guest:guest@127.0.0.1:5672/%2f";
    let conn = Connection::connect(addr, ConnectionProperties::default()).await?;
    println!("[producer] connected to RabbitMQ");
    let channel = conn.create_channel().await?;

    // 1) ประกาศ exchange แบบ topic ชื่อ "booking.events" — durable=true คือให้ exchange
    //    รอดจาก restart ของ RabbitMQ (ถูกบันทึกลง disk metadata ไม่ใช่แค่อยู่ใน memory)
    channel
        .exchange_declare(
            "booking.events",
            ExchangeKind::Topic,
            ExchangeDeclareOptions { durable: true, ..Default::default() },
            FieldTable::default(),
        )
        .await?;

    // 2) ประกาศ queue ที่ notification-service จะมา consume — ในระบบจริงฝั่ง consumer
    //    ควรเป็นคนประกาศ queue/binding ของตัวเอง (เพื่อไม่ผูก producer กับ consumer)
    //    แต่ในตัวอย่างนี้ประกาศไว้ทั้งสองฝั่งเพื่อความชัดเจน (การประกาศซ้ำด้วยค่าเดียวกันไม่มีผลข้างเคียง)
    channel
        .queue_declare(
            "notification.booking_created",
            QueueDeclareOptions { durable: true, ..Default::default() },
            FieldTable::default(),
        )
        .await?;

    // 3) bind queue เข้ากับ exchange ด้วย routing key "booking.created"
    channel
        .queue_bind(
            "notification.booking_created",
            "booking.events",
            "booking.created",
            QueueBindOptions::default(),
            FieldTable::default(),
        )
        .await?;

    // 4) สร้าง event และ serialize เป็น JSON (ต่อยอด Part 57)
    let event = BookingCreated {
        event_id: Uuid::new_v4(),
        booking_id: Uuid::new_v4(),
        customer_email: "somchai@example.com".to_string(),
        event_name: "Rust Conf Bangkok 2026".to_string(),
        seats: 2,
        created_at: chrono::Utc::now(),
    };
    let payload = serde_json::to_vec(&event)?;

    // 5) publish ไปที่ EXCHANGE (ไม่ใช่ไปที่ queue ตรง ๆ ตามที่อธิบายในหัวข้อ 82.2)
    //    พร้อม routing key "booking.created" ให้ exchange ใช้ตัดสินใจ route
    let confirm = channel
        .basic_publish(
            "booking.events",           // ชื่อ exchange
            "booking.created",          // routing key
            BasicPublishOptions::default(),
            &payload,
            BasicProperties::default().with_content_type("application/json".into()),
        )
        .await?      // ได้ PublisherConfirm กลับมา (future ชั้นที่ 1: broker รับ frame แล้ว)
        .await?;     // await ซ้ำอีกชั้น: รอ ack/nack จาก broker จริง ๆ (publisher confirm)

    println!("[producer] publish confirmed: {:?}", confirm);
    Ok(())
}
```

ผลลัพธ์จริงจากการรัน (broker เดียวกับที่ใช้ตรวจสอบทั้งบท):

```
[producer] connected to RabbitMQ
[producer] publishing: {
  "event_id": "572b27b9-c1bd-4748-b151-2387897e0447",
  "booking_id": "01a65c94-2294-4eb5-b64b-a05d0ccc6416",
  "customer_email": "somchai@example.com",
  "event_name": "Rust Conf Bangkok 2026",
  "seats": 2,
  "created_at": "2026-09-27T01:56:55.008999105Z"
}
[producer] publish confirmed: NotRequested
```

`NotRequested` หมายถึงเรายังไม่ได้เปิดโหมด **publisher confirms** อย่างเป็นทางการด้วย `channel.confirm_select(...)` — ค่าที่ได้กลับมาตอนนี้บอกแค่ว่า broker รับ frame ของ publish ไปแล้ว (ผ่าน TCP ไปถึง broker) แต่ไม่ได้ยืนยันระดับ "เขียนลง disk แล้วแน่ ๆ" หากต้องการการันตีระดับนั้นจริง (สำคัญมากสำหรับข้อมูลการเงิน) ต้องเรียก `channel.confirm_select(ConfirmSelectOptions::default())` ก่อน publish ครั้งแรก แล้วค่าที่ได้กลับมาจะเป็น `Ack`/`Nack` จริงจาก broker — ทดสอบจริงด้วยการเพิ่มบรรทัดนี้ก่อน `basic_publish`:

```rust
// เปิดโหมด publisher confirms อย่างเป็นทางการ -- ต้องเรียกก่อน publish ครั้งแรกของ channel นี้
channel.confirm_select(ConfirmSelectOptions::default()).await?;
```

ผลลัพธ์จริงที่ได้กลับมาเปลี่ยนจาก `NotRequested` เป็นค่า confirm ที่มีความหมายจริง:

```
publish confirm (พร้อม confirm_select): Ack(None)
```

`Ack(None)` แปลว่า broker ยืนยันรับ message นี้เข้าสู่ระบบเรียบร้อยจริง ๆ (`None` ในที่นี้คือไม่มี field เพิ่มเติมแนบมา) ถ้า broker ปฏิเสธ message ด้วยเหตุผลใดก็ตาม (เช่น queue เต็มตาม policy บางแบบ) ค่าที่ได้จะเป็น `Nack` แทน — บทนี้ไม่ได้เปิดโหมดนี้เป็นค่า default ในทุกตัวอย่างเพื่อให้โค้ดหลักอ่านง่าย แต่ระบบ production ที่ข้อมูลสำคัญมาก (เช่น เหตุการณ์ทางการเงิน) ควรพิจารณาเปิด `confirm_select` ไว้เสมอ แล้วเช็คค่า `Ack`/`Nack` ที่ได้กลับมาจริง ๆ ก่อนถือว่า publish สำเร็จ

### 82.4 lapin Consumer: กระบวนการแยก อ่านและ Consume จริง

Consumer ควรเป็น**process แยกจาก producer โดยสิ้นเชิง** (คนละ binary คนละ deployment) เพื่อสะท้อนความเป็นจริงว่ามันคือ service คนละตัวกัน:

```rust
use futures_lite::stream::StreamExt;
use lapin::{
    options::{BasicAckOptions, BasicConsumeOptions, BasicNackOptions},
    types::FieldTable,
    Connection, ConnectionProperties,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct BookingCreated {
    event_id: Uuid,
    booking_id: Uuid,
    customer_email: String,
    event_name: String,
    seats: u32,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "amqp://guest:guest@127.0.0.1:5672/%2f";
    let conn = Connection::connect(addr, ConnectionProperties::default()).await?;
    println!("[consumer] connected to RabbitMQ");
    let channel = conn.create_channel().await?;

    // basic_consume คืน Stream ของ Delivery — แต่ละตัวคือ message หนึ่งชิ้นที่ยังไม่ ack
    let mut consumer = channel
        .basic_consume(
            "notification.booking_created",
            "notification-worker-1",    // consumer tag — ตั้งชื่อเพื่อ debug ง่าย
            BasicConsumeOptions::default(),
            FieldTable::default(),
        )
        .await?;

    println!("[consumer] waiting for messages...");

    while let Some(delivery) = consumer.next().await {
        let delivery = delivery?;
        match serde_json::from_slice::<BookingCreated>(&delivery.data) {
            Ok(event) => {
                println!(
                    "[consumer] would send confirmation email to {} for '{}' ({} seats, booking {})",
                    event.customer_email, event.event_name, event.seats, event.booking_id
                );
                // ack แค่ตอนประมวลผลสำเร็จแล้วเท่านั้น (รายละเอียดใน 82.5)
                delivery.ack(BasicAckOptions::default()).await?;
            }
            Err(e) => {
                eprintln!("[consumer] bad payload, nacking: {e}");
                delivery
                    .nack(BasicNackOptions { requeue: false, ..Default::default() })
                    .await?;
            }
        }
    }
    Ok(())
}
```

ผลลัพธ์จริงจากการรัน producer แล้วตามด้วย consumer (คนละ process คนละครั้งการรัน):

```
=== PRODUCER ===
[producer] connected to RabbitMQ
[producer] publishing: {
  "event_id": "572b27b9-c1bd-4748-b151-2387897e0447",
  ...
}
[producer] publish confirmed: NotRequested

=== CONSUMER ===
[consumer] connected to RabbitMQ
[consumer] waiting for messages...
[consumer] would send confirmation email to somchai@example.com for 'Rust Conf Bangkok 2026' (2 seats, booking 01a65c94-2294-4eb5-b64b-a05d0ccc6416)
[consumer] done, processed 1 message(s)
```

นี่คือ end-to-end จริง: producer publish → RabbitMQ route ผ่าน exchange → เก็บใน queue → consumer (คนละ process) มาต่อและรับ message ที่ producer ส่งไปได้ครบถ้วนถูกต้อง แม้ producer จะปิดตัวไปแล้วตั้งแต่ก่อน consumer จะเริ่มทำงานด้วยซ้ำ (พิสูจน์ temporal decoupling จากหัวข้อ 82.1 อีกครั้งในระดับโค้ดจริง)

#### Prefetch/QoS: กระจายงานอย่างเป็นธรรมเมื่อมี Consumer หลายตัวแข่งกันบน Queue เดียว

ในระบบจริงมักไม่มี consumer แค่ตัวเดียว — เพื่อ scale การประมวลผล เรามักรัน consumer หลาย instance (เรียกว่า "worker") ที่ทั้งหมด consume จาก **queue เดียวกัน** RabbitMQ ค่า default จะส่ง message แบบ **round-robin** ให้ worker ที่ว่างเรียงตามลำดับ แต่ปัญหาคือ: ถ้า worker ตัวหนึ่ง "ช้า" (งานหนัก) กับอีกตัว "เร็ว" (งานเบา) round-robin แบบเดา ๆ อาจส่ง message ให้ worker ช้าไปกองรอเต็มมือ ทั้งที่ worker เร็วว่างอยู่

ทางแก้คือ **`basic_qos`** (เรียกกันทั่วไปว่า "prefetch count") — กำหนดว่า worker แต่ละตัว **รับ message ที่ยังไม่ ack ได้พร้อมกันสูงสุดกี่ตัว** ถ้าตั้ง prefetch เป็น `1` worker จะได้รับ message ใหม่ **ก็ต่อเมื่อ ack message ก่อนหน้าเสร็จแล้วเท่านั้น** ทำให้ RabbitMQ ไม่ยัด message ไปกองไว้ที่ worker ที่กำลังทำงานช้าอยู่ (มันไม่ว่างรับตัวใหม่จนกว่าจะ ack ตัวเดิม) ผลคือ worker ที่เร็วกว่าจะได้รับงานถัดไปแทนโดยธรรมชาติ:

```rust
use lapin::options::BasicQosOptions;

// prefetch_count = 1: consumer ตัวนี้จะมี unacked message ได้สูงสุดทีละ 1 เท่านั้น
channel.basic_qos(1, BasicQosOptions::default()).await?;
```

สาธิตจริง: publish 6 task เข้า queue เดียว แล้วให้ `worker-slow` (จำลองงานหนัก sleep 400ms ก่อน ack) กับ `worker-fast` (จำลองงานเบา sleep 50ms ก่อน ack) แข่งกัน consume โดยทั้งคู่ตั้ง `prefetch_count = 1` เหมือนกัน:

```rust
slow_ch.basic_qos(1, BasicQosOptions::default()).await?;
// ... worker-slow: รับ message มา sleep(400ms) แล้วค่อย ack
fast_ch.basic_qos(1, BasicQosOptions::default()).await?;
// ... worker-fast: รับ message มา sleep(50ms) แล้วค่อย ack
```

ผลลัพธ์จริงจากการรัน (คัดลอกจาก stdout ตรง ๆ):

```
[setup] published 6 tasks
[worker-fast] processed task-1
[worker-fast] processed task-2
[worker-fast] processed task-3
[worker-fast] processed task-4
[worker-fast] processed task-5
[worker-slow] processed task-0
```

เห็นได้ชัดว่า **`worker-slow` ได้รับแค่ 1 task (`task-0`)** ตลอดการทดสอบ ในขณะที่ **`worker-fast` กวาดไปเกือบทั้งหมด (`task-1` ถึง `task-5`)** ทั้งที่ตอนเริ่มต้น RabbitMQ ส่ง `task-0` ให้ `worker-slow` ไปก่อนตามลำดับ round-robin ปกติ (worker ทั้งสองว่างพร้อมกันตอนเริ่ม) — แต่เพราะ `worker-slow` ใช้เวลา 400ms กับ task นั้นและยังไม่ ack, prefetch=1 ทำให้ RabbitMQ **ไม่ส่ง task ใหม่ให้ `worker-slow` อีกจนกว่าจะ ack** ส่วน `worker-fast` ที่ ack เร็วกว่ามากก็วนกลับมารับ task ที่เหลือแทบทั้งหมดไปทำ

ถ้าไม่ตั้ง `basic_qos` เลย (ค่า default ของ RabbitMQ คือไม่จำกัด prefetch เลย — ส่งให้ทุก message ที่มีในกรอบเวลาเดียวกันตาม round-robin แบบไม่ดูว่า worker ไหนว่างจริง) worker ที่ช้าจะสะสม message ค้างอยู่ในมือจำนวนมากโดยไม่จำเป็น สร้าง imbalance ของงานระหว่าง worker ที่ทำให้ throughput โดยรวมแย่ลง — นี่คือเหตุผลที่ **ระบบ production ที่มี worker หลายตัวแทบทุกระบบ ตั้ง `basic_qos` เป็นค่าน้อย ๆ (มักเป็น 1 หรือตัวเลขน้อย ๆ ตามลักษณะงาน) เสมอ** ไม่ปล่อยเป็นค่า default

### 82.5 Message Acknowledgment และ Delivery Guarantees

#### auto-ack เทียบกับ manual ack

`basic_consume` มี option `no_ack` — ถ้าตั้งเป็น `true` (auto-ack) RabbitMQ จะ**ลบ message ออกจาก queue ทันทีที่ส่งให้ consumer** โดยไม่รอให้ consumer ยืนยันว่าประมวลผลสำเร็จเลย นี่คือความเสี่ยงที่ชัดเจน: **ถ้า consumer crash หลังรับ message มาแต่ก่อนประมวลผลเสร็จ message นั้นจะสูญหายไปตลอดกาล** เพราะ broker ลบมันไปแล้วตั้งแต่ตอนส่งออก

ตัวอย่างในบทนี้ทั้งหมดใช้ **manual ack** (`no_ack: false` ซึ่งเป็นค่า default ของ `BasicConsumeOptions::default()`) — RabbitMQ จะ**เก็บ message ไว้ใน "unacked" state** จนกว่า consumer จะเรียก `delivery.ack(...)` อย่างชัดเจน มีสามคำสั่งที่ consumer ใช้ตอบกลับ:

- **`ack`**: ประมวลผลสำเร็จ ลบ message ออกจาก queue ได้เลย
- **`nack` (requeue: true)**: ประมวลผลไม่สำเร็จแต่อยากให้ลองใหม่ → message กลับไปอยู่ใน queue (มักไปต่อคิวด้านหน้าหรือส่งให้ consumer ตัวอื่นในทันที)
- **`nack` (requeue: false)** หรือ **`reject`**: ประมวลผลไม่สำเร็จและไม่อยากให้ลองใหม่แล้ว (จะทิ้งไปเลย หรือถ้ามีตั้ง dead-letter exchange ไว้ก็จะไปที่ DLQ — หัวข้อ 82.6)

#### สาธิตจริง: consumer "ตาย" กลางทางก่อน ack ต้อง requeue

นี่คือ scenario ที่สำคัญที่สุดในหัวข้อนี้ — เขียนโปรแกรมจำลอง "consumer รับ message มาแล้วตาย (crash) ก่อน ack" แล้วพิสูจน์ว่า RabbitMQ requeue message นั้นให้ consumer ตัวอื่นจริงหรือไม่:

```rust
use futures_lite::stream::StreamExt;
use lapin::{
    options::{BasicAckOptions, BasicConsumeOptions},
    types::FieldTable,
    Connection, ConnectionProperties,
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "amqp://guest:guest@127.0.0.1:5672/%2f";

    // --- STEP 1: consumer 'A' รับ message มาแต่ "ไม่ ack" แล้วปิด connection ทิ้ง (จำลอง crash) ---
    {
        let crashing_conn = Connection::connect(addr, ConnectionProperties::default()).await?;
        let crashing_ch = crashing_conn.create_channel().await?;
        let mut consumer = crashing_ch
            .basic_consume(
                "crash_test_queue",
                "crashing-worker",
                BasicConsumeOptions::default(),
                FieldTable::default(),
            )
            .await?;

        let delivery = consumer.next().await.expect("should receive the message")?;
        println!(
            "[crashing-worker] received message (delivery_tag={}) -- แต่ไม่ ack แล้วปิด connection ทิ้งเลย (จำลอง crash)",
            delivery.delivery_tag
        );
        // ไม่เรียก delivery.ack(...) เลย จงใจ drop connection ตรงนี้
        crashing_conn.close(200, "simulated crash").await?;
    }

    tokio::time::sleep(std::time::Duration::from_millis(800)).await;

    // --- STEP 2: consumer 'B' (ตัวใหม่) มาต่อ queue เดิม ต้องเห็น message เดิมกลับมาอีกครั้ง ---
    let good_conn = Connection::connect(addr, ConnectionProperties::default()).await?;
    let good_ch = good_conn.create_channel().await?;
    let mut consumer2 = good_ch
        .basic_consume(
            "crash_test_queue",
            "recovery-worker",
            BasicConsumeOptions::default(),
            FieldTable::default(),
        )
        .await?;

    let delivery2 = consumer2.next().await.expect("stream ended")?;
    println!(
        "[recovery-worker] ได้รับ message เดิมกลับมาอีกครั้ง! redelivered={}",
        delivery2.redelivered
    );
    delivery2.ack(BasicAckOptions::default()).await?;
    Ok(())
}
```

ผลลัพธ์จริงจากการรัน:

```
[setup] published 1 message to crash_test_queue
[crashing-worker] received message (delivery_tag=1), payload="{\"booking_id\":\"crash-test-1\"}" -- แต่ไม่ ack แล้วปิด connection ทิ้งเลย (จำลอง crash)
[recovery-worker] ได้รับ message เดิมกลับมาอีกครั้ง! redelivered=true, payload="{\"booking_id\":\"crash-test-1\"}"
[recovery-worker] ack แล้ว จบการสาธิต
```

สังเกตสองจุดสำคัญ: (1) `recovery-worker` ได้รับ payload เดิม 100% ไม่เปลี่ยนแปลง แม้ `crashing-worker` จะปิด connection ไปแล้วโดยไม่ ack ก็ตาม — RabbitMQ ตรวจจับได้ว่า TCP connection ของ consumer ที่ถือ message นี้อยู่ขาดการเชื่อมต่อ จึงเอา message กลับเข้า queue โดยอัตโนมัติ (2) field `redelivered` เป็น `true` — RabbitMQ บอกไว้ในตัว message เองว่า "นี่ไม่ใช่ครั้งแรกที่ส่งให้ใครสักคน" ซึ่ง consumer สามารถอ่านค่านี้เพื่อ log พิเศษ หรือเพิ่มความระมัดระวังในการประมวลผลได้

#### ทำไมนี่คือ "at-least-once delivery" และทำไม consumer ต้อง idempotent

Scenario ข้างบนพิสูจน์ให้เห็นว่า RabbitMQ (และ message queue ระบบอื่นที่มี manual ack แบบเดียวกัน รวมถึง Kafka ในหัวข้อ 82.7) การันตีแค่ **"message จะถูกส่งให้ consumer อย่างน้อยหนึ่งครั้ง" (at-least-once)** ไม่ใช่ **"ส่งพอดีหนึ่งครั้ง" (exactly-once)** เพราะกลไกการ requeue ที่เห็นข้างบนสามารถทำให้ consumer เห็น message **เดียวกัน** มากกว่าหนึ่งครั้งได้จริง (เช่น ถ้า `crashing-worker` ในตัวอย่างจริง ๆ แล้วประมวลผลสำเร็จไปแล้ว (ส่งอีเมลไปแล้ว) แต่ crash **ก่อน** ที่จะเรียก `ack` สำเร็จ — RabbitMQ จะไม่รู้ว่าประมวลผลสำเร็จไปแล้ว จึง requeue ให้ consumer ตัวใหม่ ซึ่งจะ**ส่งอีเมลซ้ำอีกรอบ**)

นี่คือจุดที่ต้องหยิบแนวคิดเดียวกับ **Idempotency-Key จาก Part 78 หัวข้อ 78.7** มาใช้ — Part 78 แก้ปัญหา "client ยิง HTTP POST ซ้ำ (เพราะ network timeout ไม่รู้ผล) แล้วไม่อยากให้สร้าง resource ซ้ำ" ด้วยการเก็บ key ที่เคยประมวลผลไปแล้วไว้เช็คก่อน ที่นี่ปัญหาคล้ายกันมาก แค่สลับจาก "HTTP request ซ้ำ" เป็น **"message ซ้ำจาก queue"** และ key ที่ใช้เช็คคือ `event_id` (ไม่ใช่ `Idempotency-Key` header อีกต่อไป เพราะไม่มี HTTP มาเกี่ยวข้องแล้ว):

```rust
use std::collections::HashSet;
use std::sync::{Arc, Mutex};

/// จำลอง idempotent consumer ด้วย HashSet ในหน่วยความจำ
/// ในระบบจริงควรเก็บใน PostgreSQL (ตาราง processed_events(event_id PRIMARY KEY))
/// ตามแนวทาง SQLx จาก Part 70-71 เพื่อให้รอดจากการ restart ของ consumer เอง
struct IdempotentEmailSender {
    processed_event_ids: Arc<Mutex<HashSet<Uuid>>>,
}

impl IdempotentEmailSender {
    fn handle(&self, event: &BookingCreated) {
        let mut seen = self.processed_event_ids.lock().unwrap();
        if seen.contains(&event.event_id) {
            println!(
                "[notification-service] event_id={} เคยประมวลผลไปแล้ว ข้ามการส่งอีเมลซ้ำ",
                event.event_id
            );
            return;
        }
        // ประมวลผลจริง (ในที่นี้คือ "ส่งอีเมล") แล้วค่อยจดว่าประมวลผลแล้ว
        println!("[notification-service] ส่งอีเมลยืนยันให้ {}", event.customer_email);
        seen.insert(event.event_id);
    }
}
```

หลักการสำคัญคือ **การ dedupe ต้องผูกกับ "การกระทำที่มีผลข้างเคียงจริง" (ส่งอีเมล, เขียนแถวลง DB) ไม่ใช่ผูกกับแค่การ ack** เพราะลำดับเวลาที่อันตรายที่สุดคือ "ทำงานสำเร็จ (ส่งอีเมลไปแล้ว) → แต่ ack ไม่สำเร็จ (เพราะ crash พอดี)" ซึ่งจะทำให้ message ถูก redeliver แล้วมาถึงจุดตรวจสอบซ้ำอีกครั้ง — ถ้าจุดตรวจสอบ dedupe อยู่**ก่อน**การกระทำจริง (ตรวจ set ก่อนส่งอีเมล) ระบบจะไม่ส่งอีเมลซ้ำ แม้ RabbitMQ จะ deliver message นี้มากกว่าหนึ่งครั้งก็ตาม

### 82.6 Dead-Letter Queue (DLQ): เมื่อ Message ประมวลผลไม่สำเร็จซ้ำ ๆ

ถ้า consumer `nack` message ด้วย `requeue: true` ซ้ำไปเรื่อย ๆ (เช่น message เป็น "poison message" ที่ผิดรูปแบบจนประมวลผลไม่ได้เลยไม่ว่าจะพยายามกี่ครั้ง) message นั้นจะ**วนอยู่ใน queue ตลอดไป** กิน CPU/network ไปเรื่อย ๆ โดยไม่มีวันสำเร็จ ทางแก้คือตั้งค่า **Dead-Letter Exchange (DLX)** ให้กับ queue — เมื่อ message ถูก `nack` แบบ `requeue: false` (หรือ TTL หมดอายุ, หรือ queue เต็มตาม `x-max-length`) RabbitMQ จะส่ง message นั้นไปที่ DLX แทนที่จะทิ้งไปเงียบ ๆ

#### ตั้งค่า DLQ จริง

```rust
use lapin::types::{AMQPValue, FieldTable};

// 1) Dead-letter exchange + DLQ (ปลายทางสำหรับ message ที่ตายแล้ว)
channel.exchange_declare(
    "booking.events.dlx",
    ExchangeKind::Fanout,
    ExchangeDeclareOptions { durable: true, ..Default::default() },
    FieldTable::default(),
).await?;
channel.queue_declare(
    "notification.booking_created.dlq",
    QueueDeclareOptions { durable: true, ..Default::default() },
    FieldTable::default(),
).await?;
channel.queue_bind(
    "notification.booking_created.dlq",
    "booking.events.dlx",
    "",
    QueueBindOptions::default(),
    FieldTable::default(),
).await?;

// 2) Main queue ที่ตั้ง x-dead-letter-exchange ชี้ไปที่ DLX ข้างบน
//    *** ต้องตั้ง argument นี้ "ตอนประกาศ queue ครั้งแรก" เท่านั้น — จะเพิ่มทีหลังไม่ได้ (ดูกับดักท้ายบท) ***
let mut args = FieldTable::default();
args.insert(
    "x-dead-letter-exchange".into(),
    AMQPValue::LongString("booking.events.dlx".into()),
);
channel.queue_declare(
    "poison_test_queue",
    QueueDeclareOptions { durable: true, ..Default::default() },
    args,
).await?;
```

#### สาธิตจริง: poison message ไปโผล่ที่ DLQ

```rust
// publish "poison message" ที่ parse เป็น JSON ไม่ได้เลย
channel.basic_publish(
    "", "poison_test_queue", BasicPublishOptions::default(),
    b"{not valid json at all", BasicProperties::default(),
).await?.await?;

// consumer พยายาม parse ไม่ผ่าน -> nack แบบ requeue=false
let delivery = consumer.next().await.expect("expected message")?;
match serde_json::from_slice::<serde_json::Value>(&delivery.data) {
    Err(e) => {
        println!("parse ล้มเหลว: {e} -> nack(requeue=false) เพื่อส่งเข้า DLQ");
        delivery.nack(BasicNackOptions { requeue: false, ..Default::default() }).await?;
    }
    Ok(_) => unreachable!(),
}
```

ผลลัพธ์จริงจากการรัน (สคริปต์เต็มรันจริง เชื่อมต่อ broker จริง):

```
[setup] ส่ง poison message เข้า poison_test_queue แล้ว
[poison-worker] parse ล้มเหลว: key must be a string at line 1 column 2 -> nack(requeue=false) เพื่อส่งเข้า DLQ
[dlq-checker] เจอ message ใน DLQ แล้ว! payload="{not valid json at all"
```

message ไปโผล่ที่ `notification.booking_created.dlq` จริง — RabbitMQ ยังแนบ **header พิเศษชื่อ `x-death`** มาให้อัตโนมัติทุกครั้งที่ dead-letter เกิดขึ้น บอกรายละเอียดว่า message นี้ตายเพราะอะไร มาจาก queue ไหน กี่ครั้งแล้ว — จาก run จริงข้างต้น header ที่ได้กลับมา (แปลง byte array เป็น string ให้อ่านง่าย) คือ:

```
x-death: [{
  count: 1,
  exchange: "",              // publish ผ่าน default exchange (routing key = ชื่อ queue ตรง ๆ)
  queue: "poison_test_queue",
  reason: "rejected",        // มาจากการ nack/reject แบบ requeue=false
  routing-keys: ["poison_test_queue"],
  time: <timestamp>
}]
```

#### Trigger อื่นของ dead-lettering: Message TTL หมดอายุ

`nack(requeue: false)` ไม่ใช่ trigger เดียวที่ทำให้ message ถูก dead-letter — RabbitMQ ยัง dead-letter message อัตโนมัติเมื่อ **message TTL (`x-message-ttl`) หมดอายุ** ก่อนมีใคร consume มันเลย หรือเมื่อ queue เกินความยาวสูงสุดที่ตั้งไว้ (`x-max-length`) นี่มีประโยชน์มากสำหรับ event ที่ "ถ้าไม่ทันเวลาก็ไม่มีประโยชน์แล้ว" (เช่น การแจ้งเตือน real-time ที่ถ้าผ่านไปเกิน 1 นาทีก็ไม่มีความหมายอีกต่อไป)

ทดสอบจริง: ตั้ง queue ที่มี `x-message-ttl = 1000` (1000ms) พร้อม `x-dead-letter-exchange` ชี้ไปที่ DLX แยก แล้ว publish message เข้าไปโดย**ไม่มี consumer ใดมารับเลย**:

```rust
let mut args = FieldTable::default();
args.insert("x-message-ttl".into(), AMQPValue::LongInt(1000));
args.insert("x-dead-letter-exchange".into(), AMQPValue::LongString("ttl.dlx".into()));
channel.queue_declare("ttl_main_queue", QueueDeclareOptions::default(), args).await?;

channel.basic_publish("", "ttl_main_queue", BasicPublishOptions::default(), b"expires-in-1s", BasicProperties::default())
    .await?.await?;
```

ผลลัพธ์จริงจากการรัน (รอ 1.5 วินาทีแล้วเช็ค DLQ โดยไม่มี consumer ใดต่อกับ `ttl_main_queue` เลยตลอดการทดสอบ):

```
[setup] publish message ที่มี TTL 1 วินาที เข้า ttl_main_queue แล้ว -- ไม่มี consumer มารับเลย
[ttl-checker] message หมดอายุแล้วมาโผล่ที่ DLQ จริง! payload="expires-in-1s"
```

พิสูจน์ว่า RabbitMQ เอง (ไม่ใช่ consumer) เป็นคนตรวจจับว่า message หมดอายุแล้วเอง — ไม่จำเป็นต้องมีใครมา `nack` เลยด้วยซ้ำ นี่คือความต่างสำคัญจาก scenario ในหัวข้อก่อนหน้า (ที่ consumer เป็นคนสั่ง `nack(requeue: false)` เอง) — trigger dead-lettering มีได้หลายทาง (fail จาก consumer, TTL หมดอายุ, queue เกิน max length) แต่ปลายทางเดียวกันคือ DLQ ที่ตั้งไว้

field `count` ใน `x-death` มีประโยชน์มากในระบบจริง: ในทางปฏิบัติมักไม่ dead-letter ตั้งแต่ครั้งแรกที่ fail แต่จะให้ consumer **นับจำนวนครั้งที่ fail เอง** ผ่าน custom header (เช่น `x-retry-count`) เพิ่มค่าทีละ 1 ทุกครั้งที่ `nack(requeue: true)` แล้วเช็คว่าถ้าเกิน N ครั้ง (เช่น 3) ค่อย `nack(requeue: false)` เพื่อส่งเข้า DLQ จริง ๆ — นี่คือ pattern "retry ก่อนค่อย dead-letter" ที่ระบบ production ส่วนใหญ่ใช้ (แบบฝึกหัดข้อ 4 ท้ายบทให้ implement pattern นี้เต็มรูปแบบ)

### 82.7 Kafka Fundamentals: Topics, Partitions, Consumer Groups, และ Log-Based Retention

ถึงตรงนี้เราเข้าใจ RabbitMQ ในระดับที่ใช้งานได้จริงแล้ว แต่ Kafka **ไม่ใช่ "RabbitMQ อีกยี่ห้อ"** — มันมีสถาปัตยกรรมพื้นฐานต่างกันโดยสิ้นเชิง และความต่างนี้สำคัญมากพอที่จะกำหนดว่าคุณควรเลือกตัวไหนสำหรับงานแบบไหน

#### ความต่างพื้นฐาน: Queue-based (ลบตอน ack) เทียบกับ Log-based (retain ตามเวลา)

RabbitMQ คือ **queue-based**: message ถูกเก็บใน queue, พอ consumer `ack` แล้ว **message นั้นถูกลบออกจาก queue ทันที** ไม่มีใครอ่านมันได้อีก — เหมือนจดหมายที่พอเปิดอ่านและยืนยันรับแล้วก็ถูกทิ้งไป

Kafka คือ **log-based**: message (เรียกว่า "record") ถูก append เข้าไปใน **topic** ซึ่งข้างในแบ่งเป็นหลาย **partition** — แต่ละ partition คือ append-only log ที่เรียงลำดับตายตัว (คล้ายไฟล์ log ที่เขียนต่อท้ายเรื่อย ๆ ไม่มีการแทรกหรือลบกลางทาง) เมื่อ consumer "อ่าน" record หนึ่งตัว **record นั้นไม่ถูกลบ** มันยังอยู่ใน log ต่อไปจนกว่าจะครบระยะเวลา retention ที่ตั้งไว้ (เช่น 7 วัน) ไม่ว่าจะมีใคร "อ่าน" ไปแล้วกี่รอบก็ตาม

สิ่งที่เปลี่ยนแทนคือ **แต่ละ consumer group เก็บ "offset" ของตัวเอง** (ตัวเลขบอกว่า "อ่านมาถึงตำแหน่งไหนแล้ว" ในแต่ละ partition) — Kafka broker แค่จำ offset ล่าสุดที่ consumer group นั้น ๆ commit ไว้ ไม่ได้ไปแก้ไข record ในตัว log เลย

ผลลัพธ์ของความต่างนี้คือคุณสมบัติที่สำคัญมาก: **consumer group ที่ต่างกันสามารถอ่าน history ย้อนหลังได้อย่างเป็นอิสระจากกันโดยสิ้นเชิง** ตัวอย่าง: topic `ticket.sold` มี record สะสมมา 7 วัน — `notification-service` (consumer group `notification-group`) อ่านไปเรื่อย ๆ ตาม offset ของตัวเอง แต่ถ้าวันนี้มี `fraud-detection-service` ตัวใหม่ (consumer group ใหม่ `fraud-group`) อยากเริ่ม "อ่านย้อนหลังทั้ง 7 วันตั้งแต่ต้น" เพื่อวิเคราะห์ pattern ก็ทำได้ทันทีโดยตั้ง `auto.offset.reset = earliest` แล้วมันจะอ่านทุก record ที่ยังอยู่ใน retention period ได้เลย — ในขณะที่ RabbitMQ **ทำแบบนี้ไม่ได้เลย** เพราะ message ที่ `notification-service` ack ไปแล้วมันถูกลบไปจาก queue แล้วอย่างถาวร ไม่มีใครอ่านซ้ำได้อีก

#### Topic, Partition, Consumer Group — องค์ประกอบหลัก

```
Topic: ticket.sold (แบ่งเป็น 3 partition)

  Partition 0: [rec0] [rec3] [rec6] [rec9]  ...  <- offset วิ่งอิสระต่อ partition
  Partition 1: [rec1] [rec4] [rec7]         ...
  Partition 2: [rec2] [rec5] [rec8]         ...

Consumer Group "notification-group" (มี 2 consumer instance):
  consumer-A  <- ถูก assign partition 0, 1
  consumer-B  <- ถูก assign partition 2

Consumer Group "analytics-group" (มี 1 consumer instance, อ่านแยกอิสระจาก group บน):
  consumer-X  <- ถูก assign partition 0, 1, 2 (ทั้งหมด เพราะมีตัวเดียวใน group นี้)
```

หลักการสำคัญ:

- **แต่ละ partition** ถูก assign ให้ **consumer instance เดียว**ภายใน consumer group เดียวกันเท่านั้น (ไม่มีสอง instance ใน group เดียวกันอ่าน partition เดียวกันซ้อนกัน) — นี่คือกลไก **scale การประมวลผลแนวนอน**: เพิ่ม consumer instance เข้า group ได้สูงสุดเท่าจำนวน partition (ถ้ามี 3 partition แล้วเพิ่ม instance ตัวที่ 4 เข้ามา ตัวที่ 4 จะไม่ได้รับ partition ใดเลย ไม่มีงานทำ)
- **แต่ละ partition รักษาลำดับ (ordering) ของ record ภายในตัวมันเองเท่านั้น** ไม่มีการการันตี ordering ข้าม partition — นี่คือเหตุผลที่การเลือก **partition key** สำคัญมาก (หัวข้อ 82.8): record ที่ต้องเรียงลำดับกัน (เช่น event ทั้งหมดของ ticket ใบเดียวกัน) ต้องถูกส่งไปยัง partition เดียวกันเสมอ
- **consumer group ต่างกันอ่านเป็นอิสระจากกันสมบูรณ์** — group หนึ่งอ่านไปถึงไหนไม่มีผลกับอีก group เลย เพราะแต่ละ group เก็บ offset ของตัวเองแยกกัน

#### ตั้งค่า Retention จริง และดู Offset/Lag ของ Consumer Group จริง

Retention ของ topic ตั้งได้ต่อ topic ผ่าน `retention.ms` (หน่วยมิลลิวินาที) — ตัวอย่างการตั้งค่าจริงให้ topic เก็บ record ไว้ 7 วัน (604,800,000 ms) ด้วยเครื่องมือ `kafka-configs.sh` ที่มาพร้อม Kafka:

```bash
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name ticket.sold.multi \
  --alter --add-config retention.ms=604800000
```

ผลลัพธ์จริงจากการรันคำสั่งนี้กับ broker ที่ใช้ตรวจสอบทั้งบท:

```
Completed updating config for topic ticket.sold.multi.
Dynamic configs for topic ticket.sold.multi are:
  retention.ms=604800000 sensitive=false synonyms={DYNAMIC_TOPIC_CONFIG:retention.ms=604800000}
```

หลังตั้งค่านี้ record ใน partition ของ topic นี้จะถูกลบทิ้งจริง ๆ ก็ต่อเมื่อผ่านไปแล้ว 7 วันนับจากเวลาที่ record ถูกเขียน (broker มี background process ไล่ลบ segment file ที่หมดอายุเป็นระยะ ไม่ใช่ลบทันทีตอนครบเวลาเป๊ะ ๆ) — นี่คือกลไกที่ทำให้ Kafka "ลืม" ข้อมูลเก่าไปเองในที่สุด **ไม่ใช่เก็บตลอดไปแบบไม่มีที่สิ้นสุด** เพียงแต่ retention period ยาวพอที่จะให้ consumer group ใหม่ ๆ มา replay history ย้อนหลังได้ตามช่วงเวลาที่กำหนด (ต่างจาก RabbitMQ ที่ message หายทันทีที่ ack ไม่ต้องรอ TTL ใด ๆ)

ส่วนฝั่ง offset ของ consumer group ตรวจสอบได้จริงด้วย `kafka-consumer-groups.sh --describe` — นี่คือผลลัพธ์จริงจาก consumer group `notification-group` หลังจากรัน consumer ในหัวข้อ 82.9 เสร็จไปแล้ว (ตรวจสอบตอนที่ไม่มี consumer instance ใด active อยู่):

```
Consumer group 'notification-group' has no active members.

GROUP              TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID  HOST  CLIENT-ID
notification-group ticket.sold     0          3               3               0    -            -     -
```

คอลัมน์เหล่านี้คือหัวใจของโมเดล log-based retention ที่อธิบายไว้ข้างบน: **`LOG-END-OFFSET`** คือตำแหน่งล่าสุดที่มี record อยู่จริงใน partition (broker รู้ค่านี้เสมอ ไม่ว่าจะมี consumer หรือไม่), **`CURRENT-OFFSET`** คือตำแหน่งล่าสุดที่ consumer group นี้ commit ไว้ว่า "อ่านมาถึงตรงนี้แล้ว", และ **`LAG`** คือผลต่างระหว่างสองค่านี้ (`LOG-END-OFFSET - CURRENT-OFFSET`) — ในตัวอย่างนี้ `LAG = 0` แปลว่า `notification-group` อ่านตามทันทุก record ที่มีอยู่แล้ว ถ้า `LAG` เป็นค่าบวกมาก ๆ (เช่น หลักหมื่น) แปลว่า consumer group นี้ตามการผลิต record ไม่ทัน (producer ผลิตเร็วกว่า consumer บริโภค) ซึ่งเป็น metric ที่ระบบ production ต้อง monitor ตลอดเวลาเพื่อรู้ว่าต้อง scale consumer เพิ่มหรือยัง (ผูกกับหัวข้อ 82.10 เรื่อง consumer group rebalancing โดยตรง — เพิ่ม consumer instance เพื่อลด lag ได้จนถึงจำนวน partition สูงสุด)

#### ทำไม Kafka เหมาะกับ high-throughput event streaming/replay, RabbitMQ เหมาะกับ task queue

จากกลไกข้างบนสรุปเป็นแนวทางเลือกได้ตรง ๆ:

- **Kafka เหมาะกับ**: ระบบที่ต้อง**เก็บ event history ไว้ replay ได้** (event sourcing, analytics ที่ต้อง reprocess ข้อมูลเก่าด้วย logic ใหม่, audit trail ที่ต้องเก็บนาน), ระบบที่มี throughput สูงมาก (แสน-ล้าน message ต่อวินาที เพราะ append-only log เขียนเร็วกว่าโครงสร้างข้อมูลที่ต้องรองรับ random delete แบบ queue), หรือมีหลาย consumer group ที่อยากอ่านข้อมูลเดียวกันในมุมที่ต่างกัน
- **RabbitMQ เหมาะกับ**: งานแบบ **task queue** ที่แต่ละ task ต้องถูกประมวลผล**เพียงครั้งเดียวแล้วจบ** (ไม่ต้องมี consumer group ที่สองมาอ่านซ้ำ), ระบบที่ต้องการ **routing ที่ซับซ้อนและยืดหยุ่น** (topic/direct/fanout exchange ที่หัวข้อ 82.2 อธิบายไว้ — Kafka ไม่มีแนวคิด exchange/routing key แบบนี้เลย การกรอง record ต้องทำในโค้ด consumer เอง), และงานที่ throughput ระดับพัน-หมื่นต่อวินาทีก็เพียงพอ (ไม่ต้องการ scale ระดับ Kafka)

### 82.8 rdkafka: Setup และข้อแตกต่างเชิงปฏิบัติจาก lapin

#### `rdkafka` พึ่งไลบรารี C (`librdkafka`) — ต่างจาก `lapin` ที่เป็น pure Rust

นี่คือความต่างเชิงปฏิบัติที่สำคัญมากพอจะกระทบการ deploy จริง: `lapin` (หัวข้อ 82.3) เป็น **pure-Rust implementation** ของ AMQP ทั้งหมด — compile ด้วย `cargo build` ตรงไปตรงมา ไม่ต้องมี toolchain C เพิ่ม ไม่ต้องพึ่ง shared library ตอน runtime

`rdkafka` ตรงกันข้าม: มันเป็น **binding (FFI wrapper)** ครอบไลบรารี **`librdkafka`** ที่เขียนด้วย C ทั้งหมด (project `librdkafka` เป็นของ Confluent ที่ community ยอมรับกันว่าเป็น C client ที่ครบและเสถียรที่สุดสำหรับ Kafka) การ compile `rdkafka` มีสองแนวทาง:

- **`cmake-build` feature** (ที่บทนี้ใช้): ให้ `rdkafka-sys` (crate ระดับล่างที่ `rdkafka` ใช้) ดึงซอร์สของ `librdkafka` มา compile จาก C source เองตอน `cargo build` — ต้องมี `cmake`, C compiler (`gcc`/`clang`), และไลบรารีระดับ C ที่ `librdkafka` ต้องการ (เช่น `zlib`, `libssl` สำหรับ TLS) ติดตั้งอยู่ในเครื่อง build
- **`dynamic-linking` feature**: ให้ link กับ `librdkafka` ที่ติดตั้งไว้ในระบบแล้ว (ผ่าน `apt install librdkafka-dev` หรือเทียบเท่า) — เหมาะกับ production ที่ควบคุม base image ได้ชัดเจน แต่ผูก build environment กับ package ของระบบปฏิบัติการ

ความต่างนี้ไม่ใช่แค่เรื่องทฤษฎี — ระหว่างตรวจสอบเนื้อหาบทนี้ เราเจอปัญหาจริงจาก dependency แบบ C: `librdkafka` source `#include <curl/curl.h>` แบบไม่มีเงื่อนไข (ใช้สำหรับ feature OAUTHBEARER OIDC token refresh) ทำให้ compile fail ทันทีถ้าเครื่อง build ไม่มี `libcurl` development headers ติดตั้งไว้ ด้วย error จริง:

```
fatal error: curl/curl.h: No such file or directory
   60 | #include <curl/curl.h>
      |          ^~~~~~~~~~~~~
```

sandbox ที่ใช้ตรวจสอบบทนี้ไม่มี `libcurl4-openssl-dev` ให้ติดตั้งผ่าน `apt` ได้ (ปัญหาเฉพาะของ mirror ในสภาพแวดล้อมนี้) วิธีแก้ที่ใช้ได้จริงคือเปิด cargo feature เพิ่มอีกตัวคือ **`curl-static`** ซึ่งบอกให้ `rdkafka-sys` ดึงซอร์สของ curl มา build เองผ่าน crate `curl-sys` แทนที่จะพึ่ง header ของระบบปฏิบัติการ:

```toml
[dependencies]
rdkafka = { version = "0.36", features = ["cmake-build", "curl-static"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
```

นี่คือตัวอย่างจริงของ "ต้นทุนแอบแฝง" ของ FFI binding ที่ pure-Rust library อย่าง `lapin` ไม่มีเลย — ไม่ได้แปลว่า `rdkafka` แย่กว่า (มันคือ binding ของ client ที่เสถียรและครบฟีเจอร์ที่สุดสำหรับ Kafka จริง ๆ) แต่เป็นสิ่งที่ต้องรู้ล่วงหน้าตอนวางแผน deployment pipeline (Docker image ต้องมี build toolchain ที่ครบ หรือใช้ dynamic-linking กับ image ที่มี `librdkafka` ติดตั้งไว้แล้ว)

ทำไมต้องพึ่ง `librdkafka` (C library) แทนที่จะเขียน Kafka client เป็น pure Rust แบบ `lapin` ทำกับ AMQP? เหตุผลหลักคือ**ความซับซ้อนของ Kafka wire protocol และ feature set สูงกว่า AMQP มาก** (transactional producer, exactly-once semantics ระดับ producer, compression หลายแบบ, consumer group protocol เต็มรูปแบบ) — `librdkafka` ผ่านการพัฒนาและ battle-test มานานหลายปีโดยทีม Confluent และ community จนกลายเป็น de facto standard ที่ client หลายภาษา (Python's `confluent-kafka`, Go's `confluent-kafka-go`, Node's `node-rdkafka`) ต่างก็ครอบมันเป็น binding เหมือนกัน ไม่ใช่แค่ Rust ที่เลือกทางนี้ — มี pure-Rust Kafka client อื่นอยู่บ้าง (เช่น `kafka-rust`) แต่ยังไม่ครบฟีเจอร์และไม่ active พัฒนาเทียบเท่า `rdkafka` ทำให้ `rdkafka` ยังเป็นตัวเลือกที่ community แนะนำมากที่สุดสำหรับงาน production จริงในตอนนี้ แม้ต้องแบกรับความซับซ้อนของ C dependency ก็ตาม

#### Producer พร้อม Partition Key

จุดสำคัญของ Kafka producer คือการเลือก **key** ให้ record — Kafka ใช้ hash ของ key เพื่อกำหนดว่า record จะไปตก partition ไหน (ถ้าไม่ระบุ key จะกระจายแบบ round-robin ซึ่งไม่รักษาลำดับใด ๆ) สถานการณ์ที่บทนี้ใช้: ระบบขายตั๋ว ต้องการให้ record ของ event/ticket type เดียวกันทั้งหมดตกอยู่ partition เดียวกันเสมอ (รักษาลำดับของ event นั้น) จึงเลือก **`event_id`** เป็น partition key:

```rust
use rdkafka::config::ClientConfig;
use rdkafka::producer::{FutureProducer, FutureRecord};
use serde::{Deserialize, Serialize};
use std::time::Duration;
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct TicketSold {
    event_id: String, // ใช้เป็น partition key
    ticket_id: Uuid,
    buyer_email: String,
    price_cents: u64,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let producer: FutureProducer = ClientConfig::new()
        .set("bootstrap.servers", "127.0.0.1:9092")
        .set("message.timeout.ms", "5000")
        .create()?;

    let events = [
        "rust-conf-2026", "jazz-night", "food-expo-bkk",
        "rust-conf-2026", "jazz-night", "food-expo-bkk",
    ];

    for (i, event_id) in events.iter().copied().enumerate() {
        let ticket = TicketSold {
            event_id: event_id.to_string(),
            ticket_id: Uuid::new_v4(),
            buyer_email: format!("buyer{i}@example.com"),
            price_cents: 150000,
        };
        let payload = serde_json::to_string(&ticket)?;

        // key = event_id -> Kafka ใช้ hash ของ key กำหนด partition
        let record = FutureRecord::to("ticket.sold.multi")
            .payload(&payload)
            .key(event_id);

        match producer.send(record, Duration::from_secs(5)).await {
            Ok((partition, offset)) => println!(
                "event_id={event_id:<16} -> partition={partition}, offset={offset}"
            ),
            Err((e, _)) => eprintln!("ส่งไม่สำเร็จ: {e}"),
        }
    }
    Ok(())
}
```

ผลลัพธ์จริงจากการรันกับ topic ที่มี 3 partition (`ticket.sold.multi` สร้างด้วย `kafka-topics.sh --partitions 3`):

```
event_id=rust-conf-2026   -> partition=1, offset=0
event_id=jazz-night       -> partition=2, offset=0
event_id=food-expo-bkk    -> partition=1, offset=1
event_id=rust-conf-2026   -> partition=1, offset=2
event_id=jazz-night       -> partition=2, offset=1
event_id=food-expo-bkk    -> partition=1, offset=3
```

ยืนยันคุณสมบัติที่สำคัญที่สุดของ partition key ได้ชัดเจนจากข้อมูลจริง: **`rust-conf-2026` ตกที่ partition 1 เสมอทั้งสองครั้ง**, **`jazz-night` ตกที่ partition 2 เสมอทั้งสองครั้ง** (แม้ `food-expo-bkk` จะไปตก partition เดียวกับ `rust-conf-2026` โดยบังเอิญจาก hash แต่ก็ยัง**สม่ำเสมอ**ในตัวมันเอง — ไปตก partition 1 ทั้งสองครั้งเหมือนกัน) นี่คือสิ่งที่การันตี ordering ต่อ event: consumer ที่อ่าน partition 1 จะเห็น record ของ `rust-conf-2026` เรียงตามลำดับที่ publish จริง (offset 0 มาก่อน offset 2) ไม่มีทางสับลำดับกัน

### 82.9 rdkafka Consumer พร้อม Consumer Group

```rust
use rdkafka::config::ClientConfig;
use rdkafka::consumer::{Consumer, StreamConsumer};
use rdkafka::message::Message;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", "127.0.0.1:9092")
        .set("group.id", "notification-group")
        .set("auto.offset.reset", "earliest") // อ่านจากจุดเริ่มต้นของ log ถ้ายังไม่มี offset เก่า
        .set("enable.auto.commit", "true")
        .create()?;

    consumer.subscribe(&["ticket.sold"])?;
    println!("[consumer] subscribed to ticket.sold, group=notification-group");

    loop {
        match tokio::time::timeout(Duration::from_secs(10), consumer.recv()).await {
            Ok(Ok(msg)) => {
                let payload = msg.payload().map(|p| String::from_utf8_lossy(p).to_string());
                let key = msg.key().map(|k| String::from_utf8_lossy(k).to_string());
                println!(
                    "[consumer] partition={} offset={} key={:?} payload={:?}",
                    msg.partition(), msg.offset(), key, payload
                );
            }
            Ok(Err(e)) => { eprintln!("[consumer] kafka error: {e}"); break; }
            Err(_) => { println!("[consumer] timeout -> หยุด"); break; }
        }
    }
    Ok(())
}
```

ผลลัพธ์จริงจากการรัน producer (ส่ง 3 record ไปที่ topic `ticket.sold` แบบ partition เดียว) แล้วตามด้วย consumer แยก process:

```
=== PRODUCER ===
[producer] ส่ง ticket ของ event=rust-conf-2026 สำเร็จ -> partition=0, offset=0
[producer] ส่ง ticket ของ event=rust-conf-2026 สำเร็จ -> partition=0, offset=1
[producer] ส่ง ticket ของ event=jazz-night สำเร็จ -> partition=0, offset=2

=== CONSUMER ===
[consumer] subscribed to ticket.sold, group=notification-group
[consumer] partition=0 offset=0 key=Some("rust-conf-2026") payload=Some("{\"event_id\":\"rust-conf-2026\",\"ticket_id\":\"2eb9e22d-b8f5-4836-b0f6-e3656cd1b114\",\"buyer_email\":\"buyer0@example.com\",\"price_cents\":150000,\"sold_at\":\"2026-09-27T02:05:44.427455810Z\"}")
[consumer] partition=0 offset=1 key=Some("rust-conf-2026") payload=Some("{\"event_id\":\"rust-conf-2026\",\"ticket_id\":\"2d2b6f5e-ea0c-45d1-be13-5808e45fe759\",\"buyer_email\":\"buyer1@example.com\",\"price_cents\":150000,\"sold_at\":\"2026-09-27T02:05:44.543484722Z\"}")
[consumer] partition=0 offset=2 key=Some("jazz-night") payload=Some("{\"event_id\":\"jazz-night\",\"ticket_id\":\"0222ee98-9d5e-46b4-8736-316818d75ba6\",\"buyer_email\":\"buyer2@example.com\",\"price_cents\":150000,\"sold_at\":\"2026-09-27T02:05:44.550098780Z\"}")
[consumer] จบ, ได้รับทั้งหมด 3 message(s)
```

สังเกตว่า `enable.auto.commit = true` — เราให้ `rdkafka` commit offset ให้อัตโนมัติเป็นระยะ (ไม่ใช่ manual ack แบบ RabbitMQ) วิธีนี้เรียบง่ายแต่มีความเสี่ยงคล้ายกับ auto-ack ของ RabbitMQ: ถ้า consumer crash ระหว่างประมวลผล record แต่หลังจาก offset ถูก auto-commit ไปแล้ว record นั้นจะถือว่า "อ่านแล้ว" ทั้งที่ยังประมวลผลไม่เสร็จ — งาน production ที่ต้องการความแม่นยำสูงมักตั้ง `enable.auto.commit = false` แล้วเรียก `consumer.commit_message(...)` เอง**หลังประมวลผลสำเร็จแล้วเท่านั้น** (แนวคิดเดียวกับ manual ack ของ RabbitMQ ในหัวข้อ 82.5 เป๊ะ ๆ):

```rust
let consumer: StreamConsumer = ClientConfig::new()
    .set("bootstrap.servers", "127.0.0.1:9092")
    .set("group.id", "notification-group")
    .set("auto.offset.reset", "earliest")
    .set("enable.auto.commit", "false") // ปิด auto-commit -- เราจะ commit เองหลังประมวลผลสำเร็จ
    .create()?;

while let Ok(msg) = consumer.recv().await {
    // ... ประมวลผล msg (เช่น "would send confirmation email") ...
    // commit offset ของ message นี้ "หลังจาก" ประมวลผลสำเร็จแล้วเท่านั้น
    consumer.commit_message(&msg, rdkafka::consumer::CommitMode::Async)?;
}
```

หลักการ mapping ระหว่างสองระบบตรงกันแบบเป๊ะ ๆ: **`commit_message` ของ Kafka เทียบเท่ากับ `delivery.ack(...)` ของ RabbitMQ** — ทั้งคู่คือการบอก broker ว่า "ประมวลผลชิ้นนี้เสร็จแล้ว จะไม่ขอกลับมาอ่านใหม่อีก (ในเงื่อนไขปกติ)" ต่างกันแค่รายละเอียดว่า RabbitMQ ทำทีละ message ส่วน Kafka commit เป็น "ตำแหน่ง offset" ที่มักครอบคลุมหลาย record รวดเดียว (commit offset ที่ N หมายถึง "อ่านและประมวลผลสำเร็จถึงตำแหน่ง N-1 แล้วทั้งหมด")

#### Idempotent Producer: ป้องกัน Duplicate ที่เกิดจากฝั่ง Producer เอง (ไม่ใช่ Consumer)

หัวข้อ 82.5 พูดถึง idempotency ฝั่ง consumer (ป้องกันประมวลผลซ้ำเมื่อได้รับ message ซ้ำ) แต่ Kafka ยังมีกลไกป้องกัน duplicate ที่เกิด**ฝั่ง producer เอง**ด้วย: ถ้า producer ส่ง record ไปแล้วไม่ได้รับ ack กลับมา (เช่น network กระตุก) แล้ว retry ส่งซ้ำ มีความเป็นไปได้ที่ broker จะได้รับ record นั้น**สองครั้ง**ทั้งที่ครั้งแรกจริง ๆ ก็สำเร็จแล้ว เพียงแต่ ack หายไปกลางทาง เปิด **idempotent producer** ด้วยการตั้งค่าเดียว:

```rust
let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "127.0.0.1:9092")
    .set("enable.idempotence", "true") // broker จะ dedupe record ที่ producer ส่งซ้ำจาก retry เดียวกันให้อัตโนมัติ
    .set("acks", "all")                // รอ ack จากทุก replica ก่อนถือว่าสำเร็จ (ความปลอดภัยสูงสุด)
    .create()?;
```

`enable.idempotence = true` ทำให้ producer แนบ sequence number ภายในไปกับทุก record ทำให้ broker รู้ว่า record ไหนคือการ retry ของ record เดิม (ไม่ใช่ record ใหม่) แล้ว dedupe ให้อัตโนมัติที่ฝั่ง broker เอง — นี่แก้ duplicate ที่เกิด "ระหว่างทางจาก producer ไป broker" เท่านั้น **ไม่ได้แก้ duplicate ที่เกิดจากฝั่ง consumer เห็น record ซ้ำจากการ redeliver/rebalance** (ซึ่งยังต้องพึ่ง idempotent consumer ตามหัวข้อ 82.5 อยู่ดี) ทั้งสองกลไกทำงานคนละชั้นและมักต้องใช้ร่วมกันในระบบที่ต้องการความแม่นยำสูงสุด

### 82.10 Consumer Group Rebalancing

เมื่อ consumer instance ตัวใหม่**เข้าร่วม** consumer group ที่มีอยู่แล้ว หรือ instance เดิม**ออกจาก** group (ปิดตัว/crash) Kafka broker จะสั่ง **rebalance**: กระจาย partition ทั้งหมดของ topic ใหม่ให้กับ instance ที่เหลือ/มีเพิ่มขึ้น กระบวนการนี้เกิดขึ้นโดยอัตโนมัติผ่าน group coordinator protocol ของ Kafka — ผู้พัฒนาไม่ต้องเขียนโค้ดจัดการเอง แต่**ต้องเข้าใจว่ามันเกิดขึ้นได้** เพราะระหว่าง rebalance มีช่วงเวลาสั้น ๆ ที่ partition นั้นไม่มีใคร assign อยู่ (การประมวลผลของ partition นั้นหยุดชั่วคราว)

#### สาธิตจริง: รัน 2 consumer instance ใน group เดียวกัน สังเกต partition assignment

เขียนโปรแกรมที่ join group แล้ว poll ค่า `consumer.assignment()` (partition ที่ตัวเองถูก assign ในตอนนั้น) ทุก ~800ms:

```rust
use rdkafka::config::ClientConfig;
use rdkafka::consumer::{Consumer, StreamConsumer};
use std::env;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let member_name = env::args().nth(1).unwrap_or_else(|| "member".into());

    let consumer: StreamConsumer = ClientConfig::new()
        .set("bootstrap.servers", "127.0.0.1:9092")
        .set("group.id", "rebalance-demo-group")
        .set("auto.offset.reset", "earliest")
        .set("session.timeout.ms", "6000")
        .create()?;

    consumer.subscribe(&["ticket.sold.multi"])?;
    println!("[{member_name}] joined group 'rebalance-demo-group'");

    let started = std::time::Instant::now();
    loop {
        let _ = tokio::time::timeout(Duration::from_millis(500), consumer.recv()).await;
        let assignment = consumer.assignment()?;
        let parts: Vec<String> = assignment
            .elements()
            .iter()
            .map(|e| format!("{}[{}]", e.topic(), e.partition()))
            .collect();
        println!("[{member_name}] t={:>4}ms assigned = {:?}", started.elapsed().as_millis(), parts);
        // ... เงื่อนไขหยุดตาม run_secs ที่กำหนด
    }
}
```

ทดลองจริง: รัน `consumer-A` ตัวเดียวก่อน 5 วินาที แล้วค่อยรัน `consumer-B` เข้ามาสมทบในกลุ่มเดียวกัน (`consumer-B` ทำงานอยู่ 10 วินาทีแล้วออกจากกลุ่ม) — ผลลัพธ์จริงจาก stdout ของทั้งสอง process:

```
=== A ===
[consumer-A] joined group 'rebalance-demo-group'
[consumer-A] t= 501ms assigned = []
[consumer-A] t=1805ms assigned = []
[consumer-A] t=3107ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]", "ticket.sold.multi[2]"]
[consumer-A] t=3908ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]", "ticket.sold.multi[2]"]
[consumer-A] t=5513ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]", "ticket.sold.multi[2]"]
[consumer-A] t=6815ms assigned = ["ticket.sold.multi[2]"]
[consumer-A] t=8117ms assigned = ["ticket.sold.multi[2]"]
[consumer-A] t=10724ms assigned = ["ticket.sold.multi[2]"]
[consumer-A] t=14632ms assigned = ["ticket.sold.multi[2]"]
[consumer-A] t=17239ms assigned = ["ticket.sold.multi[2]"]
[consumer-A] t=18542ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]", "ticket.sold.multi[2]"]
[consumer-A] leaving group

=== B ===
[consumer-B] joined group 'rebalance-demo-group'
[consumer-B] t= 502ms assigned = []
[consumer-B] t=1323ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]"]
[consumer-B] t=2926ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]"]
[consumer-B] t=6833ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]"]
[consumer-B] t=10740ms assigned = ["ticket.sold.multi[0]", "ticket.sold.multi[1]"]
[consumer-B] leaving group
```

อ่านลำดับเวลาให้ตรงกัน (ทั้งสอง log มาจาก run เดียวกัน คนละ process ที่ start ห่างกัน 5 วินาที): ช่วงแรก `consumer-A` เข้ากลุ่มคนเดียว ได้รับ assign **ทั้ง 3 partition** (`[0]`, `[1]`, `[2]`) ไปคนเดียว — พอ `consumer-B` เข้ากลุ่มที่ t≈500ms (นับจากตอน B start) broker สั่ง rebalance: `consumer-B` ได้รับ partition `[0]` และ `[1]` ไป ในขณะที่ `consumer-A` ถูกเหลือแค่ partition `[2]` เท่านั้น (สังเกตว่าที่ t=5513ms ของ A ยังเห็น assignment เก่าอยู่เพราะ rebalance ยังไม่เสร็จสมบูรณ์ ณ ตอนนั้น กว่าจะเห็นผล rebalance จริงต้องรอถึง t=6815ms) — และเมื่อ `consumer-B` ออกจากกลุ่มไปที่ t≈10 วินาที (log ของ B หยุดที่ t=10740ms) `consumer-A` ก็ได้รับ **partition ทั้ง 3 คืนกลับมาทั้งหมด** ที่ t=18542ms (rebalance รอบที่สองใช้เวลานานกว่าเพราะ `session.timeout.ms=6000` ทำให้ broker ต้องรอให้แน่ใจว่า B หายไปจริงก่อนถึงจะ trigger rebalance)

นี่คือพฤติกรรมจริงของ **eager rebalancing** (ค่า default แบบเก่าของ `rdkafka`/`librdkafka`): ตอน rebalance เกิดขึ้น **partition assignment ทั้งหมดถูกเพิกถอนจากทุก instance ก่อน แล้วค่อย assign ใหม่ทั้งหมด** (ไม่ใช่แค่ partition ที่ต้องย้าย) ซึ่งเป็นเหตุผลที่เห็น assignment เป็น `[]` (ว่าง) ชั่วครู่ในช่วงเริ่มต้นของ A ก่อนได้รับ assignment จริง — Kafka รุ่นใหม่มี **cooperative-sticky rebalancing** ที่ฉลาดกว่า (ย้ายเฉพาะ partition ที่จำเป็นต้องย้ายจริง ๆ ลด downtime ของ partition ที่ไม่ต้องย้าย) ซึ่งตั้งได้ผ่าน `partition.assignment.strategy = cooperative-sticky` — บทนี้สาธิตด้วยค่า default เพื่อให้เห็นพฤติกรรม rebalance ชัดที่สุด ส่วนรายละเอียดเชิงลึกของ rebalancing protocol (generation, JoinGroup/SyncGroup RPC) เกินขอบเขตของบทนี้ที่เน้นระดับ awareness ว่า "มันเกิดขึ้นได้ และเกิดขึ้นยังไงในภาพกว้าง"

**ข้อสรุปเชิงปฏิบัติจากการสาธิตนี้**: การ scale consumer ขึ้น (เพิ่ม instance) จะช่วยเพิ่ม throughput ได้จริง **จนถึงจำนวน partition สูงสุด** — เพิ่ม instance เกินจำนวน partition จะไม่ได้อะไรเพิ่ม (instance ส่วนเกินไม่ได้รับ partition ใดเลย ไม่มีงานทำ) การวางแผนจำนวน partition ของ topic ล่วงหน้าจึงสำคัญมาก (เปลี่ยนจำนวน partition ทีหลังทำได้แต่ **ไม่แนะนำ** เพราะกระทบ partition key hashing เดิมที่มีอยู่)

#### เทียบศัพท์ RabbitMQ กับ Kafka แบบตรงตัว

เพราะทั้งสองระบบใช้คำที่ฟังดูคล้ายกันแต่ความหมายไม่ตรงกันเป๊ะ ตารางนี้ช่วยกันสับสนตอนอ่านเอกสารของทั้งสองฝั่งสลับกัน:

| แนวคิด | RabbitMQ | Kafka | หมายเหตุความต่าง |
|---|---|---|---|
| "ที่อยู่" ที่ producer ส่งไป | Exchange | Topic | Kafka **ไม่มี** แนวคิด exchange/routing — producer ส่งตรงไปที่ topic เสมอ |
| ที่เก็บ message จริง | Queue | Partition (ภายใน topic) | Queue ของ RabbitMQ ลบ message ตอน ack, partition ของ Kafka retain ตามเวลา |
| ตัวกำหนดปลายทาง | Routing key + Binding | ไม่มี (topic name คือปลายทางเดียว) | การ "กรอง" ฝั่ง Kafka consumer ต้องทำเองในโค้ด ไม่มี broker ช่วย route |
| ตัวกำหนดว่าไปกลุ่มไหน | (ไม่มีแนวคิดตรงนี้ ทุก queue คือ "กลุ่ม" ของตัวเอง) | Partition key | key ของ Kafka กำหนด "อยู่ partition ไหน" ไม่ใช่ "อยู่ topic ไหน" |
| การยืนยันรับ | `ack`/`nack`/`reject` ต่อ message | Commit offset (มักครอบคลุมหลาย record) | Kafka commit เป็น "ตำแหน่ง" ไม่ใช่ต่อ record แต่ละตัว |
| หน่วยที่ scale งานคู่กัน | Consumer instance ต่อ queue (จำนวนไม่จำกัดตายตัว) | Consumer instance ต่อ consumer group (จำกัดที่จำนวน partition) | นี่คือเหตุผลที่ต้องวางแผนจำนวน partition ล่วงหน้าให้ดีตามหัวข้อ 82.10 |

#### Monitoring และเครื่องมือตรวจสอบสถานะ

ทั้งสองระบบมีเครื่องมือ built-in สำหรับตรวจสอบสถานะที่ควรรู้จักไว้ (ใช้จริงตลอดบทนี้เพื่อตรวจสอบผลลัพธ์):

- **RabbitMQ Management UI**: เปิดใช้งานได้ผ่าน plugin `rabbitmq_management` (มาพร้อม image `rabbitmq:3-management-alpine` ที่ใช้ตรวจสอบทั้งบทนี้อยู่แล้ว) เข้าดูผ่านเว็บที่ port `15672` เห็น queue ทั้งหมด, จำนวน message ที่ค้าง, อัตราการ publish/consume แบบ real-time — หรือใช้ผ่าน command line ด้วย `rabbitmqctl list_queues name messages` (ใช้จริงหลายครั้งในบทนี้เพื่อตรวจสอบว่า message ค้างอยู่ใน queue กี่ตัว)
- **Kafka CLI tools**: `kafka-topics.sh --describe` (ดู partition/replica ของ topic), `kafka-consumer-groups.sh --describe --group <name>` (ดู offset/lag ตามที่สาธิตไว้ในหัวข้อ 82.7), `kafka-configs.sh --describe` (ดู config ปัจจุบันของ topic) — ทั้งหมดมาพร้อม Kafka distribution อยู่แล้ว ไม่ต้องติดตั้งเพิ่ม

ระบบ production จริงมักเสริมด้วย Prometheus exporter (ทั้ง RabbitMQ และ Kafka มี metrics endpoint ให้ scrape ได้) เพื่อ alert อัตโนมัติเมื่อ queue ยาวเกินเกณฑ์หรือ consumer lag สูงเกินเกณฑ์ — รายละเอียดการตั้ง monitoring แบบเต็มรูปแบบเกินขอบเขตของบทนี้ แต่รู้จักคำสั่งพื้นฐานข้างบนก็เพียงพอสำหรับ debug สถานการณ์ทั่วไประหว่างพัฒนา

### 82.11 เลือกอะไรดี: RabbitMQ vs Kafka vs PostgreSQL LISTEN/NOTIFY

ตอนนี้เราเข้าใจทั้งสามแนวทางแล้ว (Part 71 สอน `PgListener`/`LISTEN`/`NOTIFY` ไว้ก่อนหน้า) มาสรุปเป็นตารางตัดสินใจที่ใช้ได้จริง:

| คุณสมบัติ | RabbitMQ | Kafka | PostgreSQL LISTEN/NOTIFY |
|---|---|---|---|
| Infrastructure เพิ่ม | ต้องรัน broker แยก | ต้องรัน broker แยก (มักซับซ้อนกว่า RabbitMQ ในการ operate) | **ไม่มีเลย** — ใช้ PostgreSQL ที่มีอยู่แล้ว |
| Throughput | ปานกลาง-สูง (หลักหมื่น msg/s) | สูงมาก (หลักแสน-ล้าน msg/s) | ต่ำ (จำกัดด้วย connection pool และ PostgreSQL เอง) |
| Routing/Filtering | ยืดหยุ่นมาก (exchange type หลากหลาย, routing key pattern) | ไม่มี routing ในตัว — filter ต้องทำในโค้ด consumer เอง | จำกัดมาก (channel name ธรรมดา) |
| Replay/History | **ไม่ได้** (message ถูกลบตอน ack) | **ได้** (retention period, หลาย consumer group อ่านย้อนหลังอิสระกัน) | **ไม่ได้เลย** — ดูข้อจำกัดสำคัญด้านล่าง |
| Persistence ถ้าไม่มีใครฟัง | message รอใน queue ได้ (จนกว่าจะมีคน consume) | record รอใน log ได้ (ตาม retention) | **หายทันที** ถ้าไม่มี listener ต่ออยู่ตอนนั้น |
| ความซับซ้อนในการ operate | ปานกลาง | สูง (ZooKeeper แบบเก่า/KRaft, partition rebalancing, tuning หลายชั้น) | **ต่ำสุด** (ไม่มีอะไรต้องดูแลเพิ่ม) |
| เหมาะกับ | Task queue, work distribution, routing ที่ซับซ้อน | Event streaming, event sourcing, analytics ที่ต้อง replay, throughput สูงมาก | งานง่าย ๆ ปริมาณน้อย ที่ทนกับการ "พลาดบางครั้ง" ได้ |

#### ตัวอย่างสถานการณ์จริงสามแบบ เพื่อฝึกใช้ตารางข้างบน

การมีตารางไว้เฉย ๆ อาจยังไม่พอ ลองไล่สถานการณ์จริงที่คล้ายระบบที่คุณอาจเจอ เพื่อฝึกกระบวนการตัดสินใจ:

**สถานการณ์ ก — ระบบจัดการสินค้าคงคลังร้านค้าออนไลน์ขนาดเล็ก**: เมื่อสต๊อกสินค้าต่ำกว่าเกณฑ์ ต้องแจ้งเตือนแอดมินทาง email มีร้านค้าแค่ไม่กี่สิบออร์เดอร์ต่อวัน — ไล่ตามคำถามในหัวข้อก่อนหน้า: ทนพลาดได้ไหม (การแจ้งเตือนสต๊อกต่ำพลาดไปบ้างไม่ทำให้ระบบพัง แค่แอดมินอาจรู้ช้าไปหน่อย), ปริมาณน้อยจริงไหม (ไม่กี่สิบครั้งต่อวัน) → **`LISTEN`/`NOTIFY`** เพียงพอ ไม่ต้องเพิ่ม broker เลย

**สถานการณ์ ข — ระบบจองตั๋วเดียวกับที่ใช้ตลอดบทนี้**: booking created ต้องส่งอีเมลยืนยันแน่นอน (ลูกค้าจ่ายเงินไปแล้ว พลาดไม่ได้), ปริมาณระดับพันออร์เดอร์ต่อวัน, ต้อง route ไปหลายปลายทางตามเงื่อนไข (บาง event type ส่งแค่ notification บาง event type ส่งทั้ง notification และ analytics) → **RabbitMQ** ตรงกับทุกเงื่อนไข (task ที่ต้องแน่นอน + routing ที่ยืดหยุ่น) โดยไม่ต้องแบกความซับซ้อนของ Kafka เพราะไม่มีความจำเป็นต้อง replay history เลย

**สถานการณ์ ค — ระบบ analytics ของแพลตฟอร์ม e-commerce ขนาดใหญ่**: ต้องเก็บ event "ทุกคลิกของผู้ใช้" (page view, add-to-cart, purchase) ไว้ให้หลายทีมนำไปประมวลผลต่อ (ทีม recommendation, ทีม fraud detection, ทีม BI dashboard) แต่ละทีมอาจอยากอ่านย้อนหลัง reprocess ด้วย algorithm ใหม่เป็นระยะ ปริมาณ event ระดับหลักแสนต่อวินาทีในช่วง peak → **Kafka** เหมาะที่สุด เพราะทั้ง throughput ระดับนี้และความต้องการ replay/หลาย consumer group อ่านอิสระกันคือจุดแข็งเฉพาะตัวของ Kafka ที่ RabbitMQ ทำไม่ได้เลย (แม้ RabbitMQ จะรองรับปริมาณนี้ได้ในทางเทคนิคระดับหนึ่ง แต่จะไม่มีทาง replay history ให้ทีมใหม่ที่เข้ามาทีหลังได้)

ข้อสังเกตสำคัญจากสามสถานการณ์นี้: **ตัวชี้ขาดที่แท้จริงมักไม่ใช่แค่ "throughput สูงแค่ไหน" แต่คือ "ต้องการ replay/หลาย consumer อ่านอิสระกันหรือไม่"** สถานการณ์ ข มีปริมาณสูงกว่าสถานการณ์ ก มาก แต่ก็ยังไม่ต้อง Kafka เพราะไม่มีความต้องการ replay เลย ในขณะที่สถานการณ์ ค ต้อง Kafka เพราะโครงสร้างการใช้งาน (หลายทีมอ่านอิสระกัน, ต้อง reprocess ได้) ไม่ใช่เพราะตัวเลข throughput เพียงอย่างเดียว

#### ข้อจำกัดสำคัญของ LISTEN/NOTIFY ที่ต้องเข้าใจให้ชัด

Part 71 สอนกลไก `PgListener` ไว้ในระดับที่ใช้งานได้จริง แต่ยังไม่ได้ลงรายละเอียดข้อจำกัดที่**สำคัญที่สุด**ของมัน: **`NOTIFY` ใน PostgreSQL ไม่มี persistence เลยแม้แต่นิดเดียว** ถ้า `pg_notify()` ถูกเรียกในขณะที่**ไม่มี connection ใดเปิด `LISTEN` อยู่บน channel นั้นเลย** notification นั้น**หายไปตลอดกาลทันที** ไม่มีการเก็บไว้รอเหมือน queue ของ RabbitMQ หรือ log ของ Kafka — นี่ต่างจากทั้ง RabbitMQ และ Kafka ที่เราพิสูจน์ไปแล้วในหัวข้อ 82.1 ว่า message/record รอ consumer อยู่ได้แม้ไม่มีใครต่ออยู่ตอนที่ publish

ข้อจำกัดนี้ทำให้ `LISTEN`/`NOTIFY` เหมาะกับสถานการณ์ที่ **"พลาดบางครั้งได้ ไม่ใช่เรื่องคอขาดบาดตาย"** เท่านั้น เช่น การแจ้ง cache invalidation แบบ soft (ถ้าพลาดไปบ้าง cache จะ stale ไปพักหนึ่งแต่ไม่ถึงกับพังระบบ), หรือ dev/staging environment ที่อยากได้ real-time update แบบง่าย ๆ โดยไม่อยากตั้ง broker เพิ่ม — **ไม่เหมาะกับ event ที่ต้องรับประกันว่าถึงปลายทางแน่ ๆ อย่าง `booking.created` ที่ผูกกับการส่งอีเมลยืนยันลูกค้า** (ถ้าพลาดไปคือลูกค้าจ่ายเงินแล้วไม่ได้รับอีเมลยืนยันเลย)

#### แนวทางที่แนะนำ: เลือกตัวที่เรียบง่ายที่สุดที่ตอบโจทย์จริง

หลักการที่ควรยึดถือ (และเป็นหลักการที่คนจำนวนมากทำผิด): **อย่าไปหยิบ Kafka มาใช้เป็นค่าเริ่มต้นเพราะมันดังหรือดูล้ำ** ถามตัวเองตามลำดับนี้ก่อนตัดสินใจ:

1. งานนี้**ทนกับการพลาดบางครั้งได้ไหม** และปริมาณน้อยจริง ๆ (ไม่เกินหลักสิบ-ร้อยต่อวินาที)? → ถ้าใช่และระบบมี PostgreSQL อยู่แล้ว `LISTEN`/`NOTIFY` (Part 71) พอเพียงและ **ไม่ต้องเพิ่ม infrastructure ใหม่เลย**
2. งานนี้เป็น **task ที่ต้องประมวลผลแน่นอนสักครั้งหนึ่ง** (ส่งอีเมล, ประมวลผลคำสั่งซื้อ, sync ข้อมูล) และต้องการ routing ที่ยืดหยุ่น (ส่งไปหลายปลายทางตามเงื่อนไข)? → **RabbitMQ** ตอบโจทย์ตรงตัวและ operate ง่ายกว่า Kafka มาก
3. งานนี้ต้องการ **เก็บ event history ไว้ replay** หรือ throughput สูงมากจนเป็นคอขวดจริง (วัดแล้วจริง ๆ ตามหลัก Part 54 ไม่ใช่แค่เดา) หรือมีหลายทีม/หลาย consumer group ที่ต้องการอ่านข้อมูลสตรีมเดียวกันในมุมต่างกัน? → **Kafka** คุ้มกับความซับซ้อนในการ operate ที่เพิ่มขึ้น

capstone ของบทนี้ (หัวข้อ 82.12) เลือก **RabbitMQ** เพราะ scenario "แจ้งเตือนตอนจองตั๋วสำเร็จ" คือ task queue แบบตรงไปตรงมา ไม่มีความจำเป็นต้อง replay history หรือรับ throughput ระดับ Kafka เลย — ถ้าเลือก Kafka แทน โค้ดฝั่ง producer จะเปลี่ยนจาก `channel.basic_publish(...)` เป็น `producer.send(FutureRecord::to("booking.created").key(&booking_id.to_string())...)` (โครงสร้างเดียวกับหัวข้อ 82.8) และฝั่ง consumer จะเปลี่ยนจาก `basic_consume` เป็น `StreamConsumer` กับ `group.id` (โครงสร้างเดียวกับหัวข้อ 82.9) — แนวคิดเรื่อง "publish event แล้วให้ notification-service ไปสมัครรับเอง" เหมือนกันทุกประการ เปลี่ยนแค่ broker

### 82.12 Capstone: booking-service → RabbitMQ → notification-service (End-to-End จริง)

นี่คือการนำทุกอย่างที่เรียนมาในบทนี้มาต่อกันเป็นระบบเดียว: `booking-service` (Axum HTTP server อย่างง่าย ต่อยอด Part 62-66) รับ HTTP request สร้าง booking แล้ว publish event `booking.created` เข้า RabbitMQ ทันทีหลังสร้างสำเร็จ ส่วน `notification-service` เป็น process แยกที่ consume event นี้แล้ว "would send confirmation email" (จำลองแทนการเชื่อมต่อ email provider จริง)

#### booking-service (Axum server)

```rust
use axum::{extract::State, routing::post, Json, Router};
use lapin::{
    options::{BasicPublishOptions, ExchangeDeclareOptions},
    types::FieldTable,
    BasicProperties, Connection, ConnectionProperties, Channel, ExchangeKind,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct BookingCreated {
    event_id: Uuid,
    booking_id: Uuid,
    customer_email: String,
    event_name: String,
    seats: u32,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Deserialize)]
struct CreateBookingRequest {
    customer_email: String,
    event_name: String,
    seats: u32,
}

#[derive(Debug, Serialize)]
struct CreateBookingResponse {
    booking_id: Uuid,
    status: String,
}

// เก็บ Channel ของ lapin ไว้ใน AppState แชร์ข้าม request ตามแนวทาง Part 39-40/64
struct AppState {
    amqp_channel: Channel,
}

async fn create_booking(
    State(state): State<Arc<AppState>>,
    Json(req): Json<CreateBookingRequest>,
) -> Json<CreateBookingResponse> {
    // ในระบบจริงตรงนี้คือจุดที่ booking-service เขียนแถวใหม่ลง PostgreSQL
    // ภายใน transaction เดียวกัน (ผูกกับแนวคิด Part 70-71) ที่นี่ถือว่า "สร้าง booking สำเร็จ" แล้ว
    let booking_id = Uuid::new_v4();
    println!(
        "[booking-service] สร้าง booking {booking_id} สำเร็จ ({} ที่นั่ง สำหรับ {})",
        req.seats, req.event_name
    );

    let event = BookingCreated {
        event_id: Uuid::new_v4(),
        booking_id,
        customer_email: req.customer_email,
        event_name: req.event_name,
        seats: req.seats,
        created_at: chrono::Utc::now(),
    };
    let payload = serde_json::to_vec(&event).expect("serialize BookingCreated ต้องไม่พัง");

    // publish event หลัง commit สำเร็จ -- booking-service "ไม่รู้จัก" ผู้ฟังรายไหนเลย
    // ไม่ต้องรอให้ notification-service ตอบกลับ ไม่ต้องรู้ว่ามี consumer กี่ตัว
    let publish_result = state
        .amqp_channel
        .basic_publish(
            "booking.events",
            "booking.created",
            BasicPublishOptions::default(),
            &payload,
            BasicProperties::default().with_content_type("application/json".into()),
        )
        .await;

    match publish_result {
        Ok(confirm) => {
            let _ = confirm.await;
            println!("[booking-service] publish 'booking.created' สำเร็จ สำหรับ booking {booking_id}");
        }
        Err(e) => {
            // ในระบบจริงควร log เป็น error เข้าระบบ monitoring และอาจ retry แยกต่างหาก
            // แต่ "ไม่ควร" fail คำขอ HTTP ทั้งอันเพราะ publish ไม่สำเร็จ (booking สำเร็จไปแล้ว)
            eprintln!("[booking-service] publish ล้มเหลว: {e} (booking {booking_id} ยัง valid อยู่)");
        }
    }

    Json(CreateBookingResponse { booking_id, status: "confirmed".to_string() })
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "amqp://guest:guest@127.0.0.1:5672/%2f";
    let conn = Connection::connect(addr, ConnectionProperties::default()).await?;
    let channel = conn.create_channel().await?;
    channel
        .exchange_declare(
            "booking.events",
            ExchangeKind::Topic,
            ExchangeDeclareOptions { durable: true, ..Default::default() },
            FieldTable::default(),
        )
        .await?;

    let state = Arc::new(AppState { amqp_channel: channel });
    let app = Router::new()
        .route("/bookings", post(create_booking))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("127.0.0.1:8089").await?;
    println!("[booking-service] listening on http://127.0.0.1:8089");
    axum::serve(listener, app).await?;
    Ok(())
}
```

#### notification-service (consumer แยก process)

```rust
use futures_lite::stream::StreamExt;
use lapin::{
    options::{
        BasicAckOptions, BasicConsumeOptions, ExchangeDeclareOptions, QueueBindOptions,
        QueueDeclareOptions,
    },
    types::FieldTable,
    Connection, ConnectionProperties, ExchangeKind,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct BookingCreated {
    event_id: Uuid,
    booking_id: Uuid,
    customer_email: String,
    event_name: String,
    seats: u32,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "amqp://guest:guest@127.0.0.1:5672/%2f";
    let conn = Connection::connect(addr, ConnectionProperties::default()).await?;
    let channel = conn.create_channel().await?;

    // notification-service ประกาศ exchange/queue/binding ของตัวเอง
    // -- ไม่ต้องรอ booking-service บอกวิธี หรือรู้จัก booking-service เลยด้วยซ้ำ
    channel
        .exchange_declare(
            "booking.events",
            ExchangeKind::Topic,
            ExchangeDeclareOptions { durable: true, ..Default::default() },
            FieldTable::default(),
        )
        .await?;
    channel
        .queue_declare(
            "notification.booking_created",
            QueueDeclareOptions { durable: true, ..Default::default() },
            FieldTable::default(),
        )
        .await?;
    channel
        .queue_bind(
            "notification.booking_created",
            "booking.events",
            "booking.created",
            QueueBindOptions::default(),
            FieldTable::default(),
        )
        .await?;

    let mut consumer = channel
        .basic_consume(
            "notification.booking_created",
            "notification-service",
            BasicConsumeOptions::default(),
            FieldTable::default(),
        )
        .await?;

    println!("[notification-service] พร้อมรอ event 'booking.created' ...");
    while let Some(delivery) = consumer.next().await {
        let delivery = delivery?;
        match serde_json::from_slice::<BookingCreated>(&delivery.data) {
            Ok(event) => {
                println!(
                    "[notification-service] would send confirmation email to '{}' -> งาน '{}' จำนวน {} ที่นั่ง (booking_id={})",
                    event.customer_email, event.event_name, event.seats, event.booking_id
                );
                delivery.ack(BasicAckOptions::default()).await?;
            }
            Err(e) => eprintln!("[notification-service] parse payload ล้มเหลว: {e}"),
        }
    }
    Ok(())
}
```

#### รันจริงแบบ end-to-end: start consumer → start server → ยิง curl → สังเกตผล

ลำดับการรันจริง (ทุก process รันจริง เชื่อมต่อ RabbitMQ จริงตัวเดียวกันที่ใช้ตรวจสอบทั้งบท):

```bash
# terminal 1: start notification-service (consumer) ก่อน
./target/debug/notification_consumer

# terminal 2: start booking-service (Axum server)
./target/debug/capstone_booking

# terminal 3: ยิง HTTP request จริงสองคำขอ
curl -s -X POST http://127.0.0.1:8089/bookings \
  -H "Content-Type: application/json" \
  -d '{"customer_email":"somchai@example.com","event_name":"Rust Conf Bangkok 2026","seats":2}'

curl -s -X POST http://127.0.0.1:8089/bookings \
  -H "Content-Type: application/json" \
  -d '{"customer_email":"malee@example.com","event_name":"Jazz Night","seats":1}'
```

ผลลัพธ์จาก `curl` (HTTP response จริงจาก Axum server):

```json
{"booking_id":"7f077ab6-0d2f-437b-88f8-544a46473ada","status":"confirmed"}
{"booking_id":"e677a55d-6eaf-421a-be2c-d6c07309f584","status":"confirmed"}
```

log จริงของ `booking-service` (terminal 2):

```
[booking-service] listening on http://127.0.0.1:8089
[booking-service] สร้าง booking 7f077ab6-0d2f-437b-88f8-544a46473ada สำเร็จ (2 ที่นั่ง สำหรับ Rust Conf Bangkok 2026)
[booking-service] publish 'booking.created' สำเร็จ สำหรับ booking 7f077ab6-0d2f-437b-88f8-544a46473ada
[booking-service] สร้าง booking e677a55d-6eaf-421a-be2c-d6c07309f584 สำเร็จ (1 ที่นั่ง สำหรับ Jazz Night)
[booking-service] publish 'booking.created' สำเร็จ สำหรับ booking e677a55d-6eaf-421a-be2c-d6c07309f584
```

log จริงของ `notification-service` (terminal 1) — สังเกตว่า `booking_id` ในทั้งสองข้อความตรงกับที่ `booking-service` สร้างไว้ทุกประการ พิสูจน์ว่า event เดินทางไปถึงปลายทางถูกต้องครบถ้วน:

```
[notification-service] พร้อมรอ event 'booking.created' ...
[notification-service] would send confirmation email to 'somchai@example.com' -> งาน 'Rust Conf Bangkok 2026' จำนวน 2 ที่นั่ง (booking_id=7f077ab6-0d2f-437b-88f8-544a46473ada)
[notification-service] would send confirmation email to 'malee@example.com' -> งาน 'Jazz Night' จำนวน 1 ที่นั่ง (booking_id=e677a55d-6eaf-421a-be2c-d6c07309f584)
```

#### ถ้าเลือก Kafka แทน RabbitMQ หน้าตาโค้ดจะเป็นอย่างไร

หัวข้อ 82.11 บอกไว้ว่า capstone นี้เลือก RabbitMQ เพราะ scenario เป็น task queue ตรงไปตรงมา — ถ้าจะสลับไปใช้ Kafka แทน (เช่น ถ้าในอนาคตต้องการเก็บ history ของ `booking.created` ไว้ replay ให้ analytics ทีมใหม่) ส่วนที่เปลี่ยนมีแค่จุดเชื่อมกับ broker เท่านั้น โครง handler ของ Axum และ struct `BookingCreated` **เหมือนเดิมทุกประการ** — ฝั่ง publish ในตัว handler จะเปลี่ยนจาก

```rust
state.amqp_channel.basic_publish(
    "booking.events", "booking.created", BasicPublishOptions::default(),
    &payload, BasicProperties::default(),
).await
```

เป็น (ใช้ `event_id` เป็น partition key ตามแนวคิดหัวข้อ 82.8 — เพื่อให้ event ทั้งหมดของ booking เดียวกัน ถ้ามีหลาย event ในอนาคต เช่น `booking.created`/`booking.cancelled` เรียงลำดับกันถูกต้องเสมอ):

```rust
state.kafka_producer.send(
    FutureRecord::to("booking.created").payload(&payload).key(&event.booking_id.to_string()),
    Duration::from_secs(5),
).await
```

ส่วน `notification_consumer.rs` จะเปลี่ยนจาก `channel.basic_consume(...)` + `consumer.next().await` เป็น `StreamConsumer` กับ `.set("group.id", "notification-group")` + `consumer.recv().await` ตามโครงที่แสดงไว้เต็มรูปแบบแล้วในหัวข้อ 82.9 — ตรรกะการ deserialize JSON กลับเป็น `BookingCreated` และการพิมพ์ "would send confirmation email" เหมือนเดิมทุกบรรทัด เพราะส่วนนั้นไม่เกี่ยวกับว่า broker เป็นตัวไหนเลย นี่คือข้อดีของการแยก **business logic** ออกจาก **transport code** อย่างชัดเจนตามแนวทางการทดสอบในหัวข้อ 82.13 — สลับ broker ได้โดยกระทบแค่ชั้น transport ไม่กระทบ logic การตัดสินใจใด ๆ เลย

นี่คือคำตอบเต็มรูปแบบของสิ่งที่ Part 81 foreshadow ไว้: `booking-service` ทำหน้าที่ของตัวเองจบ (สร้าง booking, ตอบ HTTP response กลับลูกค้าทันที) โดยไม่ต้องรอ ไม่ต้องรู้จัก `notification-service` เลยแม้แต่นิดเดียว — และอย่างที่พิสูจน์ไว้แล้วในหัวข้อ 82.1 ระบบยังทำงานถูกต้องแม้ `notification-service` จะยังไม่ online ตอนที่ publish event ก็ตาม การเพิ่ม `analytics-service` หรือ `inventory-sync-service` เข้ามาสมัครรับ event เดียวกันในอนาคตทำได้ทันทีโดย**ไม่ต้องแก้โค้ด `booking-service` แม้แต่บรรทัดเดียว** — เพียงแค่เขียน consumer ตัวใหม่ที่ bind queue ของตัวเองเข้ากับ exchange `booking.events` ด้วย routing key ที่สนใจ ตรงตามโมเดล AMQP ที่อธิบายไว้ในหัวข้อ 82.2

### 82.13 การทดสอบโค้ดที่ผูกกับ Message Queue (ต่อยอด Part 32-33)

โค้ดที่คุยกับ message queue มีลักษณะเหมือนโค้ดที่คุยกับ PostgreSQL ใน Part 71: มันมี I/O จริงเข้ามาเกี่ยวข้อง (เชื่อมต่อ network ไปที่ broker) ทำให้เขียนเทสยากกว่าโค้ดล้วน ๆ ทั่วไป แนวทางที่ได้ผลดีที่สุดคือ**แยกสองส่วนออกจากกันให้ชัด** ตามหลักการเดียวกับที่ Part 32-33 สอนเรื่อง unit test เทียบ integration test:

#### ส่วนที่ 1 — Business logic ล้วน ๆ (unit test ธรรมดา ไม่แตะ network)

ฟังก์ชันที่รับ event struct มาแล้ว "ตัดสินใจ" อะไรบางอย่าง (เช่น สร้างหัวเรื่องอีเมลตามจำนวนที่นั่ง) ไม่มี I/O เกี่ยวข้องเลย ควรเขียนเป็นฟังก์ชันแยกที่รับ struct ธรรมดาเข้า-ออก แล้วเทสด้วย `#[test]` ปกติตามที่ Part 32 สอนไว้ — เทสแบบนี้รันเร็วมาก (ไม่ต้องรอ network) และไม่ต้องมี broker รันอยู่เลยตอนรัน `cargo test`:

```rust
fn decide_email_subject(event: &BookingCreated) -> String {
    if event.seats > 1 {
        format!("ยืนยันการจอง {} ที่นั่งสำหรับ {}", event.seats, event.event_name)
    } else {
        format!("ยืนยันการจองสำหรับ {}", event.event_name)
    }
}

#[cfg(test)]
mod unit_tests {
    use super::*;

    #[test]
    fn multiple_seats_uses_plural_subject() {
        let event = BookingCreated {
            event_id: Uuid::new_v4(), booking_id: Uuid::new_v4(),
            customer_email: "a@example.com".into(), event_name: "Rust Conf".into(), seats: 3,
        };
        assert_eq!(decide_email_subject(&event), "ยืนยันการจอง 3 ที่นั่งสำหรับ Rust Conf");
    }
}
```

#### ส่วนที่ 2 — Integration test ที่ต่อ broker จริง (แนวทางเดียวกับ Part 71 ที่ทดสอบ PgListener กับ PostgreSQL จริง)

การ publish/consume จริงต้องทดสอบกับ broker จริงเท่านั้น (mock connection ของ AMQP/Kafka ไม่คุ้มค่าความซับซ้อนเทียบกับการรัน broker จริงในสภาพแวดล้อมทดสอบ) ใช้ `#[tokio::test]` ต่อ broker จริง สร้าง queue ชั่วคราวที่ชื่อไม่ชนกับเทสอื่น (`auto_delete: true` ให้ RabbitMQ ลบ queue ทิ้งเองหลัง connection ปิด) แล้ว publish-consume round trip จริงในเทสเดียว:

```rust
#[tokio::test]
async fn publish_then_consume_round_trip() {
    let addr = "amqp://guest:guest@127.0.0.1:5672/%2f";
    let conn = Connection::connect(addr, ConnectionProperties::default())
        .await
        .expect("ต้องต่อ RabbitMQ ได้ระหว่างเทส");
    let channel = conn.create_channel().await.unwrap();

    // ชื่อ queue ไม่ชนกับเทสอื่นที่รันพร้อมกัน (เทียบกับแนวคิด test isolation ของ Part 32-33)
    let queue_name = format!("test_queue_{}", Uuid::new_v4());
    channel
        .queue_declare(
            &queue_name,
            QueueDeclareOptions { auto_delete: true, ..Default::default() },
            FieldTable::default(),
        )
        .await
        .unwrap();

    let event = BookingCreated {
        event_id: Uuid::new_v4(), booking_id: Uuid::new_v4(),
        customer_email: "test@example.com".into(), event_name: "Integration Test Event".into(), seats: 2,
    };
    let payload = serde_json::to_vec(&event).unwrap();
    channel.basic_publish("", &queue_name, BasicPublishOptions::default(), &payload, BasicProperties::default())
        .await.unwrap().await.unwrap();

    let mut consumer = channel
        .basic_consume(&queue_name, "test-consumer", BasicConsumeOptions::default(), FieldTable::default())
        .await.unwrap();
    let delivery = consumer.next().await.unwrap().unwrap();
    let received: BookingCreated = serde_json::from_slice(&delivery.data).unwrap();
    delivery.ack(BasicAckOptions::default()).await.unwrap();

    assert_eq!(received.event_id, event.event_id);
    assert_eq!(decide_email_subject(&received), "ยืนยันการจอง 2 ที่นั่งสำหรับ Integration Test Event");
}
```

รันจริงด้วย `cargo test` กับ RabbitMQ ที่ใช้ตรวจสอบทั้งบท ได้ผลลัพธ์จริง (ทั้ง unit test และ integration test อยู่ใน binary เดียวกัน):

```
running 3 tests
test unit_tests::single_seat_uses_singular_subject ... ok
test unit_tests::multiple_seats_uses_plural_subject ... ok
test integration_tests::publish_then_consume_round_trip ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.05s
```

จุดสำคัญที่ทำให้ integration test แบบนี้เชื่อถือได้และไม่กวนกันข้ามเทส: **สร้างชื่อ queue ที่ unique ต่อเทส** (`Uuid::new_v4()` ต่อท้ายชื่อ) และตั้ง **`auto_delete: true`** ให้ RabbitMQ เก็บกวาดทิ้งเองหลัง connection ปิด — เทียบเท่ากับแนวคิด "แต่ละ test ใช้ transaction แยกที่ rollback" ที่ Part 71 สอนไว้สำหรับ PostgreSQL เพียงแต่ RabbitMQ ไม่มี transaction rollback แบบนั้น จึงใช้ "queue แยกต่อเทส + auto-delete" แทนเพื่อให้ได้ผลลัพธ์เชิง isolation ที่เทียบเคียงกัน ในระบบ CI จริงมักรัน RabbitMQ/Kafka เป็น service container คู่กับ job ทดสอบ (คล้ายที่ Part 71 แนะนำสำหรับ PostgreSQL) เพื่อให้ integration test เหล่านี้รันได้ทุกครั้งที่ CI ทำงาน

### 82.14 Cheat Sheet: โครงโค้ด lapin เทียบกับ rdkafka แบบเคียงข้างกัน

หลังจากผ่านทั้งบทมาแล้ว หัวข้อนี้สรุปโครงโค้ดที่ใช้บ่อยที่สุดของทั้งสอง crate ไว้เทียบกันในที่เดียว สำหรับเปิดดูอ้างอิงเร็ว ๆ ตอนเขียนโค้ดจริง (ไม่ใช่เนื้อหาใหม่ แต่รวบรวมโครงจากหัวข้อ 82.3-82.9 ให้เห็นภาพเทียบกันชัดเจน):

**เชื่อมต่อและเปิด channel/producer:**

```rust
// lapin (RabbitMQ)
let conn = Connection::connect(addr, ConnectionProperties::default()).await?;
let channel = conn.create_channel().await?;

// rdkafka (Kafka) -- ไม่มีแนวคิด "channel" แยก, ClientConfig สร้าง producer/consumer ได้เลย
let producer: FutureProducer = ClientConfig::new()
    .set("bootstrap.servers", "127.0.0.1:9092")
    .create()?;
```

**Publish/Produce หนึ่ง message:**

```rust
// lapin: publish ไปที่ EXCHANGE + routing key เสมอ
channel.basic_publish(
    "my-exchange", "my.routing.key", BasicPublishOptions::default(),
    &payload, BasicProperties::default(),
).await?.await?;

// rdkafka: ส่งไปที่ TOPIC ตรง ๆ พร้อม partition key ทางเลือก
producer.send(
    FutureRecord::to("my-topic").payload(&payload).key(&partition_key),
    Duration::from_secs(5),
).await?;
```

**Consume/Receive แบบวน loop พร้อม manual ack/commit:**

```rust
// lapin: Stream ของ Delivery แต่ละตัว ack/nack เอง
let mut consumer = channel.basic_consume("my-queue", "tag", BasicConsumeOptions::default(), FieldTable::default()).await?;
while let Some(delivery) = consumer.next().await {
    let delivery = delivery?;
    // ... ประมวลผล delivery.data ...
    delivery.ack(BasicAckOptions::default()).await?;
}

// rdkafka: StreamConsumer.recv() คืนทีละ record, commit offset เอง
loop {
    let msg = consumer.recv().await?;
    // ... ประมวลผล msg.payload() ...
    consumer.commit_message(&msg, CommitMode::Async)?;
}
```

**ตั้งค่าที่ "ต้องรู้ว่ามีอยู่" ก่อนใช้งานจริง:**

| ต้องการ | lapin (RabbitMQ) | rdkafka (Kafka) |
|---|---|---|
| จำกัดจำนวนงานพร้อมกันต่อ consumer | `channel.basic_qos(n, ...)` (หัวข้อ 82.4) | ไม่มีแนวคิดตรงนี้ (จำกัดด้วยจำนวน partition ที่ assign ให้แทน) |
| ยืนยัน publish สำเร็จแน่นอน | `channel.confirm_select(...)` แล้วเช็ค `Ack`/`Nack` (หัวข้อ 82.3) | `.set("acks", "all")` + `.set("enable.idempotence", "true")` (หัวข้อ 82.9) |
| อ่านย้อนหลังทั้งหมดตอนเริ่มใหม่ | (ไม่ต้องตั้ง — message ที่ยังไม่ ack ก็รออยู่ใน queue แล้ว) | `.set("auto.offset.reset", "earliest")` (หัวข้อ 82.9, สำคัญมาก ดูกับดักข้อ 3) |
| ป้องกันงานตกค้างตลอดไปเมื่อ fail ซ้ำ | ตั้ง `x-dead-letter-exchange` ตอน `queue_declare` (หัวข้อ 82.6) | ไม่มีในตัว — ต้อง implement เองด้วย topic แยกสำหรับ "dead" record |

## กับดักที่พบบ่อย (Common Pitfalls)

**1. publish ไปที่ exchange ที่ไม่มี queue ใด bind ไว้เลย — message หายไปเงียบ ๆ ไม่มี error**

นี่คือกับดักที่ตรงกับความเข้าใจผิดในหัวข้อ 82.2 พอดี: ถ้า publish ด้วย routing key ที่ไม่มี queue ใด bind ตรงกับมันเลย (เช่น พิมพ์ routing key ผิดจาก `"booking.created"` เป็น `"booking.create"`) `basic_publish` **จะไม่ error ให้เลย** (จะได้ `PublisherConfirm` กลับมาปกติเหมือน publish สำเร็จ) เพราะ exchange ทำหน้าที่แค่ "พยายาม route" — ถ้าไม่มี queue ตรงกับ routing key มันแค่ทิ้ง message นั้นไปเงียบ ๆ ไม่มีใครแจ้งเตือน วิธีป้องกัน: เปิด flag `mandatory: true` ใน `BasicPublishOptions` — ถ้า publish แบบ mandatory แล้ว route ไม่ได้ RabbitMQ จะส่ง message กลับมาให้ producer ผ่าน `basic_return` แทนที่จะทิ้งไปเงียบ ๆ (ต้องเขียน handler รับ `BasicReturn` เพิ่มเพื่อตรวจจับกรณีนี้)

**2. Type inference ผิดพลาดตอนสร้าง `FutureRecord` ของ `rdkafka` — error จริงที่เจอระหว่างเขียนบทนี้**

ระหว่างตรวจสอบเนื้อหาบทนี้ เขียนโค้ด producer ครั้งแรกด้วย `events.iter().enumerate()` (ได้ `&&str` ไม่ใช่ `&str`) แล้วส่งเข้า `.key(event_id)` ตรง ๆ ทำให้ compiler ฟ้อง error จริงตามนี้:

```
error[E0277]: the trait bound `&str: rdkafka::message::ToBytes` is not satisfied
error[E0277]: the size for values of type `str` cannot be known at compilation time
   |
   = help: the trait `rdkafka::message::ToBytes` is implemented for `str`
```

สาเหตุคือ generic parameter ของ `FutureRecord<K, P>` ต้องการ type ที่ implement `ToBytes` (ซึ่ง implement ให้ `str` โดยตรง ไม่ใช่ `&str` หรือ `&&str`) วิธีแก้คือทำให้ตัวแปรที่ส่งเข้าไปเป็น `&str` เพียงชั้นเดียวจริง ๆ — แก้ด้วย `.iter().copied()` แทน `.iter()` เฉย ๆ เพื่อลบ reference ชั้นเกินออกไปตั้งแต่ต้น บทเรียนจากกับดักนี้: error message ของ generic trait bound ใน Rust บางทีชี้ไปที่ปัญหาปลายเหตุ (ตรงบรรทัด `.await`) มากกว่าต้นเหตุ (ตรงจุดที่สร้าง reference ผิดชั้น) ต้องตามดู type จริงของตัวแปรที่ผ่านเข้าไปในทุกขั้น

**3. `auto.offset.reset = "latest"` (ค่า default ของ Kafka) ทำให้ consumer ที่เพิ่ง start พลาด record ที่ส่งไปก่อนหน้าแบบไม่รู้ตัว**

ถ้าไม่ตั้งค่า `auto.offset.reset` เอง Kafka จะใช้ค่า default เป็น `"latest"` — หมายความว่า consumer group ที่**ยังไม่มี offset ที่ commit ไว้เลย**จะเริ่มอ่านจาก record **ถัดไปที่จะมาถึง** เท่านั้น (ไม่อ่าน record เก่าที่มีอยู่แล้วใน topic ก่อนหน้านี้) นี่ทำให้เกิดอาการที่ทำให้คนสับสนบ่อยมาก: publish record ไปก่อน แล้วค่อย start consumer ทีหลัง (คิดว่า Kafka retain ไว้ให้เหมือนที่หัวข้อ 82.7 อธิบาย) แต่ consumer**ไม่เห็น record เก่าเลย**เพราะ default เป็น `latest` — ทางแก้คือตั้ง `auto.offset.reset = "earliest"` อย่างชัดเจนเสมอสำหรับ consumer group ใหม่ที่ต้องการอ่านย้อนหลังทั้งหมด (ตามที่ตัวอย่างในบทนี้ทำไว้ทุกที่)

**4. เปิด `no_ack: true` (auto-ack) เพื่อความง่าย แล้วสูญเสียการันตี delivery ทั้งหมด**

หัวข้อ 82.5 พิสูจน์ไปแล้วว่า manual ack ทำให้ message ที่ consumer ตายกลางทางถูก requeue กลับมาได้ ถ้าเปลี่ยนไปใช้ `no_ack: true` (auto-ack) — RabbitMQ จะลบ message ออกจาก queue **ทันทีที่ส่งให้ consumer** ก่อนที่ consumer จะได้ประมวลผลด้วยซ้ำ ถ้า scenario เดียวกันในหัวข้อ 82.5 (`crashing-worker` ปิด connection ก่อนประมวลผลเสร็จ) เกิดขึ้นภายใต้ auto-ack message นั้นจะ**สูญหายไปตลอดกาล** ไม่มี `recovery-worker` ตัวไหนได้รับมันอีก เพราะ RabbitMQ ถือว่ามันถูกส่งและ "จบงาน" ไปแล้วตั้งแต่ตอนส่งออกจาก queue คำแนะนำ: ใช้ manual ack เป็นค่าเริ่มต้นเสมอสำหรับงานที่ผลลัพธ์สำคัญ เปิด auto-ack เฉพาะกรณีที่ยอมรับการสูญ message ได้จริง ๆ (เช่น metric/log ที่ไม่ critical)

**5. เปลี่ยนค่า `x-dead-letter-exchange` ของ queue ที่มีอยู่แล้วไม่ได้ — ต้อง declare ใหม่เท่านั้น**

RabbitMQ ถือว่า queue argument อย่าง `x-dead-letter-exchange` เป็นส่วนหนึ่งของ "นิยาม" ของ queue ตั้งแต่ตอนสร้าง — ถ้าพยายาม `queue_declare` queue ที่**มีอยู่แล้ว** (เคยสร้างไว้แบบไม่มี DLX) ด้วย argument ที่ต่างจากเดิม RabbitMQ จะปฏิเสธด้วย error ประมาณนี้ (documented behavior จาก RabbitMQ เอง ไม่ใช่การจำลองในบทนี้ เพราะทุก queue ที่ทดสอบในบทนี้สร้างแบบมี argument ที่ต้องการไว้ตั้งแต่ต้น):

```
PRECONDITION_FAILED - inequivalent arg 'x-dead-letter-exchange' for queue 'poison_test_queue' in vhost '/': received the value 'booking.events.dlx' but current is none
```

ทางแก้เมื่อเจอสถานการณ์นี้ในระบบจริง: ต้องลบ queue เดิมทิ้ง (`queue_delete`) แล้วสร้างใหม่ด้วย argument ที่ต้องการ (มีผลกระทบคือ message ที่ค้างอยู่ใน queue เดิมจะหายไปด้วย ต้องวางแผน migration ให้ดี เช่น ให้ consumer ระบายของออกจาก queue เดิมให้หมดก่อนค่อยลบสร้างใหม่) — บทเรียนคือควรตัดสินใจเรื่อง DLX ให้เรียบร้อย**ตั้งแต่ตอนออกแบบ queue ครั้งแรก** ไม่ใช่ไปเพิ่มทีหลังตอน queue มี data หรือ consumer ใช้งานอยู่แล้ว

**6. Kafka ปฏิเสธ message ที่ใหญ่เกินไปแบบเงียบ ๆ ไม่ crash แต่ producer ได้ error กลับมาแทน**

Kafka broker มีค่า default limit ขนาด message ต่อ record (`message.max.bytes` ฝั่ง broker, ปกติราว ๆ 1MB) ทดสอบจริงด้วยการส่ง payload ขนาด 2MB (เกิน limit ชัดเจน):

```rust
let big_payload = vec![b'x'; 2_000_000]; // 2MB
let record = FutureRecord::to("ticket.sold").payload(&big_payload).key("big-test");
producer.send(record, Duration::from_secs(5)).await
```

ผลลัพธ์จริงจากการรัน:

```
ส่งไม่สำเร็จตามคาด: Message production error: MessageSizeTooLarge (Broker: Message size too large)
```

`.send()` คืน `Err` กลับมาให้จัดการตามปกติ ไม่ crash โปรแกรม — แต่กับดักจริงมักไม่ได้เกิดตอนทดสอบแบบนี้ (ที่รู้ตัวว่าส่งข้อมูลใหญ่) แต่เกิดตอน production ที่ event payload โตขึ้นเรื่อย ๆ ตามเวลา (เช่น แนบ array ของ item ในออร์เดอร์ที่ไม่มีการจำกัดจำนวนไว้ล่วงหน้า) จนวันหนึ่งเกิน limit แบบไม่มีใครคาดคิด ทางแก้ระยะยาวคือกำหนด**ขนาด payload สูงสุดที่ยอมรับได้ตั้งแต่ตอนออกแบบ schema ของ event** (เช่น ไม่แนบ object ก้อนใหญ่เข้าไปในตัว event ตรง ๆ แต่แนบแค่ id แล้วให้ consumer ไป fetch รายละเอียดจาก API/DB เอาเองถ้าจำเป็น) แทนที่จะไปเพิ่ม `message.max.bytes` ที่ฝั่ง broker เรื่อย ๆ ตามขนาดข้อมูลที่โตขึ้น

**7. เปลี่ยน schema ของ event โดยไม่ระวังความเข้ากันได้ระหว่าง producer กับ consumer คนละเวอร์ชัน**

เพราะ producer และ consumer เป็น service คนละตัว deploy แยกกัน (ตามหลัก decoupling ของหัวข้อ 82.1) จึงมีช่วงเวลาที่ **producer เป็นเวอร์ชันใหม่แต่ consumer ยังเป็นเวอร์ชันเก่าอยู่** (deploy ไม่พร้อมกันเป๊ะ) เสมอ ถ้า producer เพิ่ม field ใหม่ที่เป็น**required** (ไม่มี `Option<T>` หรือ `#[serde(default)]`) เข้าไปใน struct โดยตรง — ยังไม่มีปัญหาฝั่ง producer เพราะมันแค่ serialize แต่ปัญหาจะเกิดตรงกันข้าม: ถ้า**ลบ field เก่าออกไปโดยที่ consumer เวอร์ชันเก่ายัง deserialize struct ที่มี field นั้นเป็น non-optional อยู่** consumer จะ deserialize ไม่ผ่านทันทีด้วย error แบบ `missing field` ของ `serde_json` (รูปแบบเดียวกับที่ Part 57 อธิบายไว้เรื่อง struct ที่ไม่ match กับ JSON) ทำให้ event ทุกตัวที่ผลิตจากเวอร์ชันใหม่ถูก parse ไม่ผ่านที่ consumer เวอร์ชันเก่าทั้งหมด (กลายเป็น poison message ไหลเข้า DLQ ตามหัวข้อ 82.6 เป็นจำนวนมากพร้อมกัน) แนวทางป้องกัน: เพิ่ม field ใหม่ให้เป็น `Option<T>` พร้อม `#[serde(default)]` เสมอ (ตาม attribute ที่ Part 57-58 สอนไว้) และห้ามลบ field เก่าออกทันที ให้ deprecate ไว้ก่อนแล้วค่อยลบทีหลังเมื่อ consumer ทุกตัวอัปเดตแล้วแน่ใจแล้วเท่านั้น — เป็นวินัยเดียวกับการทำ API versioning ที่ Part 78 พูดถึงเรื่อง backward compatibility

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** แก้ `consumer.rs` ของหัวข้อ 82.4 ให้ไม่หยุดหลังรับ 1 message (เอา logic ที่ทำให้ loop จบออก) แล้วแก้ `producer.rs` ให้ publish `BookingCreated` 5 event ติดกัน (เปลี่ยน `event_name`/`seats` ให้ต่างกันในแต่ละตัว) รันแล้วสังเกตว่า consumer รับครบทั้ง 5 ข้อความตามลำดับที่ publish หรือไม่

   *Hint*: RabbitMQ queue รักษาลำดับ FIFO ภายใน queue เดียวเสมอ (ไม่เหมือน Kafka ที่ลำดับการันตีแค่ภายใน partition) โครง loop `for i in 0..5 { ... publish ... }` ในฝั่ง producer และเอาเงื่อนไข `if processed >= 1 { break; }` ออกจากฝั่ง consumer ก็เพียงพอแล้ว ผลลัพธ์ที่ควรได้คือ log 5 บรรทัดเรียงตามลำดับ `seats`/`event_name` ที่ publish ไปเป๊ะ ๆ

2. **(กลาง)** เพิ่ม consumer ตัวที่สองชื่อ `analytics-service` ที่ bind queue ของตัวเอง (`analytics.all_booking_events`) เข้ากับ exchange `booking.events` เดียวกัน แต่ใช้ routing key pattern แบบ topic คือ `booking.#` (รับทุก event ที่ขึ้นต้นด้วย `booking.`) แทนที่จะ bind ตรง ๆ ด้วย `booking.created` เท่านั้น แล้วทดสอบ publish ทั้ง `booking.created` และ `booking.cancelled` (routing key ต่างกัน) ยืนยันว่า `analytics-service` ได้รับทั้งสองแบบ ในขณะที่ `notification-service` (ที่ bind แค่ `booking.created`) ได้รับแค่แบบแรก

   *Hint*: ต้องเปลี่ยน exchange declare เป็น `ExchangeKind::Topic` (ถ้ายังไม่ใช่) และใช้ `#` ใน binding pattern ตามที่อธิบายในหัวข้อ 82.2 โครง binding ของ `analytics-service`:
   ```rust
   channel.queue_bind(
       "analytics.all_booking_events", "booking.events", "booking.#",
       QueueBindOptions::default(), FieldTable::default(),
   ).await?;
   ```
   ลองสั่ง publish ด้วย routing key `"booking.cancelled"` (ไม่มี queue ของ `notification-service` bind ตรงกับ key นี้เลย) แล้วยืนยันว่ามีแค่ `analytics-service` เท่านั้นที่ได้รับ log

3. **(กลาง-ยาก)** implement idempotent consumer แบบเต็มรูปแบบตามแนวคิดหัวข้อ 82.5: เก็บ `event_id` ที่ประมวลผลไปแล้วไว้ใน `HashSet<Uuid>` (หรือถ้าอยากต่อยอด SQLx จาก Part 70-71 ให้เก็บในตาราง PostgreSQL `processed_events(event_id UUID PRIMARY KEY, processed_at TIMESTAMPTZ)` แทน เพื่อให้รอดจากการ restart ของ consumer เอง) แล้วจำลอง "at-least-once delivery" ด้วยการส่ง `event_id` เดียวกันเข้า queue สองครั้งซ้อน (publish ซ้ำตรง ๆ) พิสูจน์ว่า consumer ประมวลผล (พิมพ์ "ส่งอีเมล") แค่ครั้งเดียว ไม่ใช่สองครั้ง

   *Hint*: จุดเช็ค `HashSet`/query ตาราง ต้องอยู่**ก่อน**การกระทำที่มีผลข้างเคียงจริงเสมอ (ก่อน `println!("ส่งอีเมล...")` ไม่ใช่หลัง) โครงคร่าว ๆ:
   ```rust
   fn handle(&self, event: &BookingCreated) {
       let mut seen = self.processed_event_ids.lock().unwrap();
       if seen.contains(&event.event_id) {
           println!("event_id={} เคยประมวลผลไปแล้ว ข้าม", event.event_id);
           return; // <-- ต้องเช็คและ return ก่อนถึงบรรทัดที่ "ส่งอีเมล" จริง
       }
       println!("ส่งอีเมลยืนยันให้ {}", event.customer_email);
       seen.insert(event.event_id);
   }
   ```
   ทดสอบด้วยการ publish `BookingCreated` ที่มี `event_id` เดียวกันสองครั้ง (เจตนา — จำลอง redelivery) แล้วนับว่า log "ส่งอีเมลยืนยัน" ปรากฏกี่ครั้ง (ควรได้ 1 ครั้งเท่านั้น)

4. **(ยาก/ประยุกต์ใช้งานจริง)** ต่อยอดหัวข้อ 82.6 ให้เป็น retry-then-dead-letter pattern แบบสมบูรณ์: แทนที่จะ `nack(requeue: false)` ทันทีที่ parse ล้มเหลว ให้ consumer อ่าน custom header `x-retry-count` จาก `delivery.properties.headers()` ก่อน (ถ้าไม่มีให้ถือว่าเป็น 0) ถ้ายังน้อยกว่า 3 ให้ `basic_publish` message เดิมกลับเข้า queue เดิมอีกครั้งพร้อมเพิ่ม `x-retry-count` ขึ้นหนึ่ง แล้ว `ack` message เดิมทิ้ง (เทคนิค "manual requeue with counter" เพราะ `nack(requeue: true)` ธรรมดาไม่ให้เราแก้ header ระหว่างทาง) แต่ถ้าครบ 3 ครั้งแล้วให้ `nack(requeue: false)` เพื่อให้ dead-letter ไปที่ DLQ ที่ตั้งไว้ในหัวข้อ 82.6 จริง ๆ

   *Hint*: ต้องสร้าง `FieldTable` ใหม่ทุกครั้งที่ republish (คัดลอก header เดิมมาแล้วแก้แค่ `x-retry-count`) เพราะ `BasicProperties`/headers เป็น immutable ต่อ message เดิมที่รับมา แก้ในตัวเดิมไม่ได้ โครงตรรกะ:
   ```rust
   let retry_count: i64 = delivery.properties.headers()
       .as_ref()
       .and_then(|h| h.inner().get("x-retry-count"))
       .and_then(|v| v.as_long_long_int())
       .unwrap_or(0);

   if retry_count < 3 {
       let mut new_headers = FieldTable::default();
       new_headers.insert("x-retry-count".into(), AMQPValue::LongLongInt(retry_count + 1));
       let props = BasicProperties::default().with_headers(new_headers);
       channel.basic_publish("", queue_name, BasicPublishOptions::default(), &delivery.data, props)
           .await?.await?;
       delivery.ack(BasicAckOptions::default()).await?; // ack ตัวเดิม เพราะเรา republish เองแล้ว
   } else {
       delivery.nack(BasicNackOptions { requeue: false, ..Default::default() }).await?; // ครบ 3 ครั้ง -> DLQ จริง
   }
   ```
   ทดสอบด้วย poison message เดิมจากหัวข้อ 82.6 แล้วยืนยันว่า retry ครบ 3 รอบก่อนจะไปโผล่ที่ DLQ จริง ๆ (ไม่ใช่ไปตั้งแต่ครั้งแรก)

## สรุป

บทนี้เติมเต็มสิ่งที่ Part 81 foreshadow ไว้ให้สมบูรณ์: **asynchronous messaging** คือคำตอบสำหรับสถานการณ์ที่ synchronous call แบบ gRPC (Part 80) ไม่เหมาะ — โดย decouple ผู้ส่งกับผู้รับทั้งในมิติเวลา (ไม่ต้องออนไลน์พร้อมกัน) และมิติความรู้จัก (ไม่ต้องรู้จักกัน) เราเรียน AMQP model ของ RabbitMQ อย่างละเอียด (exchange/queue/binding/routing key และเหตุผลที่ producer publish ไปที่ exchange ไม่ใช่ queue ตรง ๆ) implement producer/consumer จริงด้วย `lapin` พร้อมพิสูจน์ manual ack, at-least-once delivery, การ requeue ตอน consumer crash, และ dead-letter queue ด้วยการรันจริงทุกขั้นตอน จากนั้นข้ามไปดูสถาปัตยกรรมที่ต่างออกไปโดยสิ้นเชิงของ **Kafka** (topic/partition/consumer group, log-based retention ที่เปิดทางให้ replay ได้ซึ่ง RabbitMQ ทำไม่ได้) พร้อม implement ด้วย `rdkafka` และพิสูจน์พฤติกรรม partition-by-key และ consumer group rebalancing ด้วยข้อมูลจริงจากการรัน ปิดท้ายด้วยตารางตัดสินใจที่ชัดเจนระหว่าง RabbitMQ, Kafka, และ `LISTEN`/`NOTIFY` จาก Part 71 — พร้อมย้ำหลักการสำคัญว่าให้เลือกเครื่องมือที่เรียบง่ายที่สุดที่ตอบโจทย์จริง ไม่ใช่ไล่ตามความล้ำของเทคโนโลยี และปิดด้วย capstone ที่ทำให้ event `booking.created` จาก Part 81 เดินได้จริงแบบ end-to-end ทุกขั้นตอน

จาก Part 82 นี้ไป **Part 83 (Caching ด้วย Redis)** จะพาไปสำรวจอีกเครื่องมือ infrastructure หนึ่งที่ระบบจริงต้องมี — ต่างจาก message queue ที่เน้นการสื่อสารแบบ asynchronous ระหว่าง service, Redis เน้นการเก็บข้อมูลแบบ in-memory ที่เร็วมากสำหรับ caching และ pattern อื่น ๆ ที่ตัดปัญหาโหลด PostgreSQL ซ้ำ ๆ ในงานที่อ่านบ่อยกว่าเขียนมาก

---

**Part ก่อนหน้า:** [Microservices Architecture ด้วย Rust](part-081-microservices.md) | **Part ถัดไป:** [Caching ด้วย Redis](part-083-redis-caching.md)
