# Part 96: Docker และ Containerization สำหรับ Rust

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 270 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- อธิบายได้อย่างตรงไปตรงมาว่า container ให้ประโยชน์อะไรจริง ๆ กับแอป Rust — โดยไม่พูดเกินจริงว่า container
  แก้ปัญหาที่ compiled binary ของ Rust ไม่มีอยู่แล้วตั้งแต่ต้น (ต่างจาก Node.js ที่ต้องแบก `node_modules`
  หรือ Python ที่ต้องแบก virtualenv ไปด้วยเสมอ)
- อธิบายกลไก layer caching ของ Docker ได้อย่างถูกต้องแม่นยำ (ไม่ใช่แค่ "มัน cache ให้" แบบผิวเผิน) และ
  ใช้ความเข้าใจนั้นออกแบบลำดับ instruction ใน `Dockerfile` ให้ build เร็วที่สุด
- เขียน multi-stage `Dockerfile` ที่แยก build stage (มี Rust toolchain เต็ม) จาก runtime stage (เล็กที่สุด
  เท่าที่จำเป็น) ได้จริง พร้อมพิสูจน์ด้วยตัวเลขขนาด image จริงว่าต่างจาก single-stage แบบไม่ระมัดระวังแค่ไหน
- build binary แบบ static ด้วย `musl` target แล้วรันบน `FROM scratch` ได้จริง พร้อมอธิบาย trade-off ของ static
  linking (ทำไม crate ที่พึ่ง OpenSSL ของระบบถึง build แบบนี้ไม่ได้ และทำไมหลักสูตรนี้เลือก `rustls`/pure-Rust
  crypto มาตั้งแต่ Part 70/74 จึงเป็นทางเลือกที่ถูกต้องมาตลอด)
- แก้ปัญหาคลาสสิกของ "Rust + Docker build ช้า" ด้วยสองเทคนิคจริง (การแยก `COPY` manifest ก่อน source, และ
  `cargo-chef`) พร้อมตัวเลขเวลา build จริงที่พิสูจน์ว่าเร็วขึ้นกี่เท่า
- ตั้งค่า environment variable/secret ใน container ตามหลัก twelve-factor app ได้อย่างปลอดภัย และอธิบายได้ว่า
  ทำไม "ลบ secret ในเลเยอร์ถัดไป" ไม่ใช่การลบที่ปลอดภัยจริง พร้อมพิสูจน์ด้วยการกู้ข้อมูลจริงจาก image
- เขียน `docker-compose.yml` ที่รันแอป + PostgreSQL จริง + Redis จริงพร้อมกันด้วยคำสั่งเดียว และนำเทคนิคทั้งบท
  มาประกอบเป็น container image เดียวที่ containerize แอป capstone จาก Part 92-94 อย่างถูกต้องตามหลักการ
  production จริง

## ความรู้ที่ต้องมีมาก่อน

- **Part 92-94 (Full-Stack Capstone Project)** — Part 94 หัวข้อ 94.5 ได้ preview `Dockerfile` แบบ multi-stage
  ไว้แล้วเป็นตัวอย่างที่ใช้งานได้จริง (build backend + frontend รวมเป็น image เดียว) แต่บอกไว้ตรง ๆ ว่า "ไม่ลง
  รายละเอียดกลไกของ Docker เอง เพราะนั่นเป็นหน้าที่ของ Part 96" — บทนี้คือบทนั้น เราจะย้อนกลับไปอธิบายกลไก
  เบื้องหลังของทุกบรรทัดใน Dockerfile ของ Part 94 ให้ลึกกว่าเดิมมาก แล้วต่อยอดด้วยเทคนิคที่ Part 94 ยังไม่ได้ทำ
  (musl static linking, dependency caching อย่างเป็นระบบ)
- **Part 17 (Packages, Crates, Workspaces)** — โครงสร้าง `[workspace] members = [...]`, path dependency,
  และคำสั่ง `-p`/`--workspace` ของ Cargo — บทนี้ใช้ความเข้าใจนี้ตรง ๆ ตอนสอน containerize โปรเจกต์ที่มีหลาย
  crate/binary
- **Part 70 (SQLx และ PostgreSQL) และ Part 74 (JWT Authentication)** — `DATABASE_URL`, `JWT_SECRET`, และหลัก
  "อ่าน config จาก environment variable เสมอ ห้าม hardcode" ที่ทั้งสองบทตั้งไว้ตั้งแต่ต้น — บทนี้ต่อยอดหลักการ
  เดียวกันไปสู่บริบทของ container โดยตรง และอ้างอิงการเลือก TLS backend (`rustls`/`rust_crypto`) ที่ทั้งสองบท
  เลือกไว้ ซึ่งมีผลโดยตรงต่อความเป็นไปได้ของ static linking ในบทนี้
- **Part 82-83 (Message Queues และ Redis Caching)** — แนวคิดเรื่องบริการเสริม (PostgreSQL, Redis) ที่แอปต้อง
  พึ่งพาตอนรันจริง — บทนี้ใช้ `docker-compose` รันบริการเหล่านี้คู่กับแอปจริง
- **Part 81 (Microservices พื้นฐาน)** — แนวคิด health check/readiness ที่ Part 94 หัวข้อ 94.7 implement ไว้
  แล้ว — บทนี้อ้างอิง `/health` endpoint แบบเดียวกันตอนทำ `docker-compose` healthcheck/`depends_on`

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้ตรวจสอบด้วย **Docker daemon จริงที่ใช้งานได้ในสภาพแวดล้อมที่เขียนบทนี้** (`docker --version` รายงาน
`Docker version 29.3.1, build c2be9cc` และ `docker ps` ทำงานได้ปกติ) — **ทุกตัวเลขขนาด image, เวลา build, และ
output ของคำสั่งในบทนี้คัดลอกมาจากการรัน `docker build`/`docker run`/`docker compose` จริงทั้งหมด ไม่มีการ
แต่งขึ้นเองแม้แต่จุดเดียว** โปรเจกต์ทดสอบทั้งหมด (`greet-service`, `fleet-workspace`, `presence-api`,
`library_api_mini`, และ Dockerfile ทดลองต่าง ๆ) ถูกสร้างไว้ใน scratch directory แยกนอก repo ของหลักสูตร
build/run/verify จริงทีละตัว **แล้วลบ image, container, volume, และไฟล์ scratch ทั้งหมดทิ้งหลังตรวจสอบเสร็จ**
(ไม่กระทบไฟล์ใด ๆ ในหลักสูตร) — เหมือนแนวทางที่ Part 92-94 ใช้มาตลอด

สภาพแวดล้อมนี้อยู่หลัง proxy ที่ควบคุม network egress (เหตุผลด้านความปลอดภัยของระบบที่รันหลาย agent พร้อมกัน)
ทำให้บางคำสั่ง (`docker build`) ต้องเติม `--network=host --build-arg https_proxy=...` เพื่อให้ `cargo`/`rustup`
ภายใน container เข้าถึง `crates.io` ได้ — **นี่เป็นเงื่อนไขเฉพาะของสภาพแวดล้อมเขียนบทนี้เท่านั้น** เครื่อง
dev/CI ปกติของผู้อ่านไม่มีข้อจำกัดนี้ ใช้ `docker build -t myimage .` ตรง ๆ ได้เลยโดยไม่ต้องมีเงื่อนไขพิเศษใด ๆ
(รายละเอียดเพิ่มเติมอยู่ในกับดักที่พบบ่อยข้อ 5)

## เนื้อหา

### 96.1 ทำไมต้อง Container สำหรับแอป Rust: มุมมองที่ตรงไปตรงมา

ก่อนเริ่มเขียน `Dockerfile` สักบรรทัดเดียว ต้องตอบคำถามที่ตรงไปตรงมาที่สุดก่อน: **container แก้ปัญหาอะไรให้แอป
Rust จริง ๆ?** คำถามนี้สำคัญเพราะเหตุผลที่คนส่วนใหญ่ใช้อธิบาย containerization (เช่น "รวม dependency ทั้งหมด
ไว้ในที่เดียว", "ไม่ต้องกังวลเรื่อง `node_modules`/virtualenv ขาดหาย") **ไม่ใช่ปัญหาที่ Rust binary ที่ compile
แบบ release มีอยู่แล้วตั้งแต่ต้น**

**เทียบให้เห็นภาพจริง**: ต่อยอดจากที่ Part 94 หัวข้อ 94.2 แสดงไว้ว่า `cargo build --release` ให้ binary ขนาด
14MB ที่รันได้เลยไม่ต้องพึ่งอะไรเพิ่ม ลองตรวจสอบด้วย `ldd` (เครื่องมือมาตรฐานของ Linux ที่แสดง shared library ที่
binary หนึ่งต้องพึ่งตอนรัน) กับ binary จริงที่ compile จาก Rust source เดียวกับที่บทนี้จะใช้ตลอดทั้งบท
(`greet-service` — axum service เล็ก ๆ ที่มีแค่ `axum`, `tokio`, `serde`, `serde_json` เป็น dependency):

```bash
$ ldd greet-service-glibc-binary
	linux-vdso.so.1 (0x00007f3121f9b000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f3121e0a000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f3121d21000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3121a00000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f3121f9d000)
```

**มีแค่ 4 shared library** และทั้ง 4 ตัวนี้ (`libgcc_s`, `libm`, `libc`, dynamic linker) เป็นส่วนหนึ่งของ
glibc/GCC runtime ที่**มีอยู่ในทุก Linux distro สมัยใหม่อยู่แล้วโดย default** ไม่ต้องติดตั้งเพิ่มแม้แต่ตัวเดียว —
เทียบกับแอป Node.js ทั่วไปที่ต้องแบก `node_modules/` (มักหลักสิบ-หลักร้อย MB, หลายพัน dependency ไฟล์) ไปด้วย
เสมอ หรือแอป Python ที่ต้องมี interpreter เวอร์ชันที่ตรงกันบวก virtualenv/package ทั้งหมดถึงจะรันได้ — Rust
binary ที่ compile แบบ release **ไม่มีปัญหา "ผู้ใช้ไม่มี runtime/interpreter ตรงเวอร์ชัน" หรือ "dependency ขาด
หาย" แบบภาษาอื่นเลยตั้งแต่ต้น** เพราะ dependency ทั้งหมดของ crate ที่ใช้ (ไม่ใช่ syscall ของ OS) ถูก compile
รวมเข้าไปในตัว binary เดียวเรียบร้อยแล้วตอน `cargo build`

**ดังนั้นคำถามที่ตรงกว่าคือ: ถ้า Rust ไม่มีปัญหานี้อยู่แล้ว container ยังมีประโยชน์อะไรเหลือ?** คำตอบตรง ๆ คือ
มีอยู่จริง 3 เรื่อง แต่เป็นคนละเรื่องกับที่ภาษาอื่นได้ประโยชน์:

**1. Dependency ระดับ OS ที่ไม่ได้มาคู่กับ binary** — แม้ crate dependency จะถูก compile รวมเข้า binary แต่
บาง crate ยังพึ่ง **native library ของระบบ** อยู่ดี เช่น `openssl-sys` (ถ้าเลือก native-tls แทน rustls),
`libpq` (ถ้าใช้ driver ที่ link กับ PostgreSQL client library โดยตรงแทนการ implement wire protocol เอง แบบที่
Diesel บางโหมดทำ), หรือ `libsqlite3`/font rendering library ต่าง ๆ — ถ้าแอปพึ่งอะไรแบบนี้ container ก็ยังมี
ประโยชน์แน่นอนในการ "แช่แข็ง" เวอร์ชันของ native library เหล่านั้นให้เหมือนกันทุกที่ที่ deploy (dev/staging/
production) ป้องกันปัญหา "รันบนเครื่อง dev ได้ แต่ production error หา `.so` ไม่เจอ/เวอร์ชันไม่ตรง" — สังเกตว่า
หลักสูตรนี้เลือก `sqlx` (implement wire protocol เองแบบ pure-Rust ไม่พึ่ง `libpq`) และ `jsonwebtoken` feature
`rust_crypto`/`argon2` (pure-Rust crypto ไม่พึ่ง OpenSSL ของระบบ) มาตั้งแต่ Part 70/74/92 แล้ว ทำให้แอป capstone
ของหลักสูตรนี้**อยู่ในกรณีที่ container ช่วยเรื่องนี้น้อยกว่าปกติ** — แต่โปรเจกต์ Rust จำนวนมากในโลกจริงยังพึ่ง
native library อยู่ (เช่น crate ที่ bind กับ `libcurl`, image processing library อย่าง `libvips`, หรือ
`native-tls` ที่ผูกกับ OpenSSL ของระบบ) กรณีเหล่านี้ container ช่วยได้เต็มที่

  **ตัวอย่างที่เป็นรูปธรรมที่สุดจากในหลักสูตรนี้เอง**: Part 72 หัวข้อ 72.3 เจอปัญหานี้จริงตอนติดตั้ง Diesel
  (ต่างจาก SQLx ที่ pure-Rust ล้วน Diesel พึ่ง **`libpq`** ผ่าน crate `pq-sys` ที่ link แบบ FFI) — เครื่องที่ใช้
  เขียนบทนั้นมี `libpq5` (runtime library) แต่ไม่มี `libpq-dev` (development package ที่มี unversioned symlink
  `libpq.so` ที่ linker ต้องการ) ทำให้ `cargo install diesel_cli` ล้มเหลวจริงด้วย `error: unable to find
  library -lpq` จนต้องแก้ด้วยการสร้าง symlink มือ (`ln -sf libpq.so.5 libpq.so`) — **นี่คือตัวอย่างจริงของ
  "ปัญหาที่ container แก้ได้เต็มที่"**: ถ้า build image ที่ `apt-get install libpq-dev` ไว้ให้เรียบร้อยตั้งแต่
  ต้น (layer ที่ cache ได้ตามหัวข้อ 96.2) ทุกคนในทีมและทุก CI runner ที่ build จาก image เดียวกันนี้จะไม่เจอ
  ปัญหา `libpq.so` หาไม่เจอแบบที่ Part 72 เจอเลยแม้แต่ครั้งเดียว เพราะ dependency ระดับ OS นี้ถูก "แช่แข็ง" ไว้
  ใน image แล้ว ไม่ต้องพึ่งว่าเครื่องของใครติดตั้ง `libpq-dev` ไว้หรือยัง — ต่างจาก SQLx ของ Part 70 ที่ไม่มี
  ปัญหานี้เลยตั้งแต่ต้นเพราะไม่มี C dependency ให้ต้องจัดการ (สอดคล้องกับตารางเปรียบเทียบ SQLx/Diesel ของ Part
  72 หัวข้อสุดท้ายที่บอกว่า SQLx เหมาะกับ "build/deploy pipeline ที่ต้องการ pure Rust ล้วน ๆ ไม่มี C library
  เป็นภาระ")
- **2. Build environment ที่สม่ำเสมอ** — `rustc`/`cargo` เวอร์ชันที่ต่างกันเพียงเล็กน้อยอาจทำให้ผลลัพธ์ build
  ต่างกัน (edition ใหม่, dependency ที่ต้องการ Cargo feature ที่เพิ่งมี, หรือแค่ optimization ที่เปลี่ยนไป) —
  ถ้าทีมมีคนหลายคน หรือมี CI/CD pipeline หลายจุด (dev build, staging build, production build) การ "build
  ภายใน container ที่ pin เวอร์ชัน Rust ไว้ตายตัว" (`FROM rust:1.82-slim` แทน `FROM rust:latest` ที่เปลี่ยน
  เวอร์ชันไปเรื่อย ๆ) ทำให้ผลลัพธ์ build เหมือนกันทุกที่ 100% ไม่ใช่แค่ "รันได้" แต่ "compile ได้ด้วยเวอร์ชัน
  toolchain เดียวกันเป๊ะ" — นี่คือประโยชน์ที่มีมูลค่าจริงไม่ว่าภาษาโปรแกรมจะเป็นอะไรก็ตาม (แม้ compiled
  language ก็ยังได้ประโยชน์นี้เต็มที่)
- **3. ความสม่ำเสมอของ deployment/orchestration** — นี่คือประโยชน์ที่ใหญ่ที่สุดในทางปฏิบัติ: แพลตฟอร์ม deploy
  สมัยใหม่ (Kubernetes, ECS, Cloud Run, Fly.io ฯลฯ) **ทำงานกับ container image เป็นหน่วยพื้นฐาน** ไม่ว่าแอป
  ข้างในจะเป็นภาษาอะไร — การมี container image ทำให้แอป Rust ของคุณ **deploy ด้วย pipeline เดียวกัน**, **scale
  ด้วยกลไกเดียวกัน**, และ **monitor ด้วยเครื่องมือเดียวกัน** กับแอปภาษาอื่น ๆ ในองค์กรเดียวกัน (เช่น service
  Python/Go/Java ที่ทีมอื่นเขียน) — นี่ไม่ใช่ประโยชน์ที่มาจาก "Rust ต้องการ container" แต่มาจาก "องค์กร/
  แพลตฟอร์มต้องการหน่วยที่เป็นมาตรฐานเดียวกันสำหรับทุกแอป" ซึ่งเป็นความจริงที่ยังคงอยู่ไม่ว่า binary ข้างในจะ
  self-contained แค่ไหน

**ข้อสรุปที่ตรงกับความเป็นจริง**: ถ้าคำถามคือ "container ช่วยแก้ปัญหา dependency ของแอป Rust แบบเดียวกับที่
ช่วย Node.js/Python ไหม" คำตอบคือ **ไม่ค่อยช่วย** เพราะปัญหานั้นเล็กกว่ามากในโลก Rust ตั้งแต่ต้น แต่ถ้าคำถามคือ
"container ยังมีประโยชน์กับแอป Rust ไหม" คำตอบคือ **มีแน่นอน** ด้วยเหตุผลสามข้อข้างบน — เป้าหมายของบทนี้คือสอน
วิธี containerize แอป Rust ให้ **ได้ประโยชน์ทั้งสามข้อเต็มที่ โดยไม่แบก "น้ำหนักส่วนเกิน" ที่ไม่จำเป็น** (เช่น
Rust toolchain เต็มรูปแบบที่ไม่มีใครใช้ตอนรันจริง) ซึ่งเป็นความผิดพลาดคลาสสิกที่หัวข้อ 96.3 จะแสดงให้เห็นด้วย
ตัวเลขจริง

### 96.2 Docker พื้นฐานสำหรับคนที่ยังไม่เคยใช้: Image, Container, Dockerfile, Layer และ Cache

ก่อนลงรายละเอียด Dockerfile ของ Rust ต้องปูพื้นแนวคิดของ Docker เองให้แน่นก่อน เพราะกลไก **layer และ cache**
ที่จะอธิบายในหัวข้อนี้คือสิ่งที่กำหนดรูปแบบของ Dockerfile แทบทุกบรรทัดที่จะเขียนตลอดบทนี้

**Image กับ Container ต่างกันอย่างไร**: เปรียบเทียบง่าย ๆ — **image** คือ "แบบพิมพ์เขียว" (template) ที่อ่านได้
อย่างเดียว ประกอบด้วยไฟล์ระบบ (filesystem) ทั้งหมดที่แอปต้องการ บวก metadata (เช่นจะรัน process ไหนตอน start)
ส่วน **container** คือ "instance ที่รันอยู่จริง" ที่สร้างขึ้นจาก image — เปรียบได้กับความสัมพันธ์ระหว่าง struct
definition (`struct User { ... }`) กับ instance ของมัน (`User { name: "Alice", ... }`) ใน Rust: image คือ
นิยาม, container คือของจริงที่รันอยู่ — จาก image เดียวกันสามารถสร้าง container ได้หลายตัวพร้อมกัน (เหมือน
สร้าง `User` หลาย instance จาก struct เดียวกัน) แต่ละ container มี filesystem layer เขียนได้ของตัวเอง
(ephemeral — หายไปเมื่อ container ถูกลบ) ทับอยู่บน image ที่อ่านได้อย่างเดียวร่วมกัน

**`Dockerfile` คือสูตรสร้าง image**: ไฟล์ text ธรรมดาที่มี instruction เรียงเป็นลำดับ แต่ละ instruction (เช่น
`FROM`, `RUN`, `COPY`, `ENV`) สร้าง **layer ใหม่หนึ่งชั้นซ้อนบนชั้นก่อนหน้า** — นี่คือกลไกที่สำคัญที่สุดที่ต้อง
เข้าใจให้แม่น: **แต่ละ layer คือ diff ของ filesystem เทียบกับ layer ก่อนหน้า** ไม่ใช่ snapshot เต็มรูปแบบ
(คล้าย git commit ที่เก็บ diff ไม่ใช่ copy ทั้งไฟล์ทุกครั้ง) เมื่อรัน container จาก image, Docker เอา layer
ทั้งหมดมาซ้อนกันด้วย union filesystem (overlay filesystem) ให้เห็นเป็น filesystem เดียวที่สมบูรณ์

**Layer caching — กลไกที่กำหนดว่า Dockerfile ควรเขียนอย่างไร**: ทุกครั้งที่ `docker build` ประมวลผล
instruction หนึ่งบรรทัด มันจะคำนวณ "cache key" ของ layer นั้น (โดยสรุปคร่าว ๆ คือ hash ของ instruction เอง
บวก content ของไฟล์ที่ instruction นั้นอ้างถึง เช่น `COPY src ./src` จะ hash เนื้อหาไฟล์ใน `src/` ทั้งหมด) แล้ว
เทียบกับ cache ของการ build ครั้งก่อน — **ถ้า cache key ตรงกัน Docker จะข้าม (skip) การรัน instruction นั้น
จริง ๆ แล้วใช้ layer ที่ cache ไว้แทนทันที** (แสดงเป็น `CACHED` ใน build output) แต่มีกฎสำคัญที่ทำให้ caching
เป็นแบบ **chain ที่ขาดตอน**: **เมื่อ layer หนึ่ง cache miss (เนื้อหาเปลี่ยน) ทุก layer ที่อยู่หลังจากนั้นใน
Dockerfile จะ cache miss ไปด้วยโดยอัตโนมัติ ไม่ว่า instruction เหล่านั้นจะเปลี่ยนจริงหรือไม่** เพราะ Docker ไม่มี
ทางรู้ว่า layer ที่ตามมาจะได้ผลลัพธ์เหมือนเดิมหรือไม่ถ้า input ของ layer ก่อนหน้าเปลี่ยนไปแล้ว (คิดในมุม
correctness — มันไม่ปลอดภัยที่จะสมมติว่า `RUN cargo build` จะได้ผลลัพธ์เดิมถ้า `COPY src` ก่อนหน้าเปลี่ยนไป)

นี่คือกฎที่**สำคัญที่สุด**ของทั้งบทนี้ และเป็นเหตุผลตรง ๆ ว่าทำไมหัวข้อ 96.6 (dependency caching) ถึงต้องแยก
`COPY Cargo.toml` ออกจาก `COPY src` เป็นคนละบรรทัด — ถ้า `COPY . .` ทั้งโฟลเดอร์ในบรรทัดเดียว **ทุกครั้งที่แก้
โค้ดแม้แค่ตัวอักษรเดียว layer นั้นจะ cache miss ทันที และทุก layer หลังจากนั้น (รวม `RUN cargo build --release`
ที่ compile dependency ทั้งหมดด้วย) จะต้องรันใหม่หมดทุกครั้ง** แม้ dependency (`Cargo.toml`/`Cargo.lock`) จะไม่
เปลี่ยนแปลงเลยก็ตาม — หัวข้อ 96.6 จะพิสูจน์เรื่องนี้ด้วยตัวเลขเวลา build จริงที่ต่างกันอย่างมีนัยสำคัญ

**ทดลองดูกลไกนี้แบบง่ายที่สุดก่อนไปแตะ Rust** (ตัวอย่างจริงที่ build จริง ใช้ `alpine` เปล่า ๆ ไม่มี Rust
เกี่ยวข้องเลย เพื่อแยกกลไกของ Docker เองออกจากความซับซ้อนของ `cargo build`):

```dockerfile
# Dockerfile.cached (จากการ build ครั้งที่สอง ที่แก้ layer C)
FROM rust:1-slim
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src
COPY src ./src
RUN touch src/main.rs && cargo build --release
```

รัน build ครั้งแรก (cold — ยังไม่มี cache ของ Dockerfile นี้เลย) แล้วรันซ้ำโดยไม่แก้อะไรเลย ตามด้วยแก้แค่
`src/main.rs` แล้ว build ครั้งที่สาม — สังเกต output ของ BuildKit ที่บอกสถานะ cache ของแต่ละ layer ตรง ๆ (นี่
คือ output จริงจากการ build `greet-service` ที่หัวข้อ 96.6 จะอธิบายรายละเอียดเต็ม ๆ — เอามาแสดงตรงนี้ก่อนเพื่อ
ให้เห็นกลไก cache ล้วน ๆ):

```
#8 [builder 2/8] WORKDIR /app
#8 CACHED

#9 [builder 3/8] COPY Cargo.toml Cargo.lock ./
#9 CACHED

#10 [builder 4/8] RUN mkdir -p src && echo "fn main() {}" > src/main.rs
#10 CACHED

#11 [builder 5/8] RUN cargo build --release
#11 CACHED

#12 [builder 6/8] RUN rm -rf src
#12 CACHED

#13 [builder 7/8] COPY src ./src
#13 DONE 0.2s          <-- cache miss ที่นี่ (เนื้อหา src/ เปลี่ยน)

#14 [builder 8/8] RUN touch src/main.rs && cargo build --release
#14 0.185    Compiling greet-service v0.1.0 (/app)
#14 6.516     Finished `release` profile [optimized] target(s) in 6.39s
#14 DONE 6.6s          <-- cache miss ต่อเนื่องจาก layer ก่อนหน้า (ตามกฎ chain)
```

สังเกตชัดเจนว่า **layer ที่ 2 ถึง 6 ยัง `CACHED` อยู่ครบทุกอัน** (เพราะ `Cargo.toml`/`Cargo.lock` ไม่เปลี่ยน)
มีแค่ layer ที่ 7 (`COPY src`) ที่ cache miss เพราะเนื้อหาไฟล์เปลี่ยนจริง แล้ว layer ที่ 8 (`cargo build`)
cache miss ตามไปด้วยตามกฎ chain ที่อธิบายไว้ข้างบน — แต่สังเกตว่า **การ compile ครั้งนี้ (6.39 วินาที) เร็วกว่า
การ compile ทั้งโปรเจกต์จากศูนย์มาก** เพราะ `cargo build` ที่รันใน layer 8 ยัง "เห็น" ผลลัพธ์ของ dependency ที่
compile ไว้แล้วจาก layer 5 อยู่ (มันอยู่ใน filesystem เดียวกันที่สืบทอดต่อกันมาเป็นชั้น ๆ ภายใน container
filesystem ของ build stage เดียวกัน — คนละเรื่องกับ "cache ของ Docker layer" แต่เป็นกลไกของ Cargo's incremental
target directory เอง) — นี่คือหลักการที่หัวข้อ 96.6 จะขยายให้เห็นตัวเลขเต็มรูปแบบ

**ทำไม caching นี้ไม่ทำงานข้าม `docker build` คนละ machine โดย default**: cache ที่อธิบายมาทั้งหมดนี้ **อยู่บน
เครื่อง (หรือ CI runner) ที่รัน `docker build` เท่านั้น** โดย default — ถ้า build บน CI runner ที่เป็น container
สดใหม่ทุกครั้ง (ทั่วไปของ GitHub Actions/GitLab CI มาตรฐาน) จะไม่มี cache เดิมให้ใช้เลยในการ build แรกของทุกรัน
(ต้อง build จากศูนย์เสมอ) เว้นแต่จะตั้งค่า **registry cache** หรือ **cache export/import ผ่าน BuildKit**
โดยเฉพาะ (`--cache-from`/`--cache-to`) ซึ่งเป็นหัวข้อของ CI/CD ที่ Part 97 จะสอนต่อ — บทนี้เน้นกลไก cache บน
เครื่องเดียวก่อน เพราะเป็นพื้นฐานที่ต้องเข้าใจก่อนจะไปตั้งค่า cache ข้าม CI runner ได้อย่างถูกต้อง

**Image ประกอบด้วยอะไรจริง ๆ — มองผ่าน `docker inspect`/manifest**: layer ที่อธิบายมาทั้งหมดคือ "filesystem
diff" แต่ image เองมีอีกส่วนที่สำคัญไม่แพ้กันคือ **config** — JSON object ที่บอกว่า layer ไหนต่อจากไหนตามลำดับ,
`ENTRYPOINT`/`CMD` ที่จะรันตอน start container, `ENV` ที่ set ไว้, `EXPOSE` ที่ประกาศไว้ (แค่ documentation ไม่ใช่
การเปิด port จริง — ดูกับดักที่พบบ่อยข้อ 7), และ architecture/OS ที่ image นี้ build มา (`linux/amd64`,
`linux/arm64` ฯลฯ) — ดูได้จริงด้วย:

```bash
$ docker inspect greet-multistage:latest --format '{{json .Config}}' | head -c 300
{"Env":["PATH=/usr/local/sbin:/usr/local/bin:..."],"Cmd":["./greet-service"],
"WorkingDir":"/app","ExposedPorts":{"8080/tcp":{}}, ...}
```

**image ID (hash ที่เห็นในทุกคำสั่ง `docker images`/`docker build` เช่น `2b5b1cea7e45`) คือ hash ของ config
JSON นี้เอง** ไม่ใช่ hash ของไฟล์ทั้งหมดรวมกัน — เปลี่ยน `CMD`/`ENV` แม้ไม่แก้ layer ไหนเลยก็ทำให้ image ID
เปลี่ยนไปด้วย เพราะ config เปลี่ยน — ส่วน layer แต่ละชั้นก็มี hash ของตัวเอง (content-addressable เหมือน git
object) ทำให้ layer เดียวกันที่ถูกใช้ในหลาย image (เช่น layer ของ `debian:bookworm-slim` base) **แชร์กันได้ใน
เครื่องเดียวโดยไม่ต้องเก็บซ้ำ** — นี่คือเหตุผลที่ `docker images` แสดง "Disk Usage" แยกจาก "Content Size" ในบท
นี้ตลอด: Disk Usage คือพื้นที่ที่ image นี้กินจริงบนเครื่อง (นับ layer ที่ share กับ image อื่นแค่ครั้งเดียว)
ส่วน Content Size คือขนาดที่ต้อง push/pull ผ่าน network ถ้าปลายทางยังไม่มี layer ไหนเลย (สถานการณ์แรกสุดตอน
deploy ไปเครื่องใหม่)

**Container กับ Virtual Machine ต่างกันอย่างไร — คำถามที่มักสับสนตอนเริ่มต้น**: VM (เช่น ผ่าน VirtualBox/KVM)
จำลอง**ฮาร์ดแวร์ทั้งเครื่อง**ขึ้นมาใหม่ (virtual CPU, virtual disk, virtual network card) แล้วรัน OS kernel
เต็มรูปแบบของตัวเองข้างในอีกชั้น — ทำให้ VM หนักและช้าต่อการ start (ต้อง boot kernel ใหม่ทุกครั้ง) แต่ได้ isolation
ที่แน่นหนาที่สุด (แยก kernel กันจริง) — **container ไม่จำลองฮาร์ดแวร์หรือ kernel ใหม่เลย** มันรัน process ปกติบน
**kernel เดียวกันกับ host** เพียงแต่ใช้ feature ของ Linux kernel เอง (namespaces สำหรับแยกมุมมองของ process ID/
network/filesystem, cgroups สำหรับจำกัด CPU/memory) มา "หลอก" process ให้คิดว่าตัวเองอยู่ในระบบที่แยกจากกันสมบูรณ์
— ผลคือ container **start เร็วกว่า VM มาก** (เป็น millisecond ไม่ใช่วินาที เพราะไม่ต้อง boot kernel) และเบากว่า
มาก (ไม่ต้องแบก kernel ทั้งชุดไปด้วย) แต่ isolation หย่อนกว่า VM เล็กน้อย (share kernel เดียวกับ host จริง ๆ ถ้ามี
ช่องโหว่ที่ kernel level อาจกระทบ host ได้ ต่างจาก VM ที่ kernel แยกกันสนิท) — สำหรับ deploy แอป web service
ทั่วไป (แบบทุกตัวอย่างในบทนี้) trade-off นี้เอียงไปทาง container ชัดเจน เพราะความเร็วในการ start/scale สำคัญกว่า
ระดับ isolation ที่ VM ให้เพิ่มมา ในสถานการณ์ที่ต้องแยก tenant ที่ไม่เชื่อถือกันเลย 100% (เช่น cloud provider ที่
รัน VM ของลูกค้าคนละคนบนเครื่องเดียวกัน) มักใช้ VM หรือเทคโนโลยีลูกผสมอย่าง Firecracker/gVisor แทน

### 96.3 แนวทางที่ไม่ดี: Single-Stage Dockerfile ด้วย `rust:latest`

มาดูวิธีที่คนเพิ่งเริ่มเขียน Dockerfile สำหรับ Rust มักทำก่อน — ใช้ image `rust:latest` (ที่มี Rust toolchain
เต็มรูปแบบ: `rustc`, `cargo`, และเครื่องมือ build อื่น ๆ) ทั้ง compile และรันแอปในภาพเดียวกัน:

```dockerfile
# Dockerfile.naive — วิธีที่ทำงานได้ แต่ไม่ควรใช้ใน production
FROM rust:latest
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/greet-service"]
```

โปรเจกต์ทดสอบคือ `greet-service` — axum service เล็ก ๆ ที่มีแค่สอง endpoint (`GET /` กับ `GET /health`) ใช้
dependency พื้นฐานที่สุด (`axum`, `tokio`, `serde`, `serde_json`) เพื่อให้เห็นภาพชัดว่าปัญหาที่จะพิสูจน์ต่อไปนี้
เกิดจากตัว Dockerfile เอง ไม่ใช่จาก dependency ที่หนักผิดปกติ:

```rust
// src/main.rs
use axum::{routing::get, Json, Router};
use serde_json::{json, Value};

async fn health() -> Json<Value> {
    Json(json!({ "status": "ok", "service": "greet-service" }))
}

async fn greet() -> Json<Value> {
    Json(json!({ "message": "สวัสดีจาก greet-service" }))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/health", get(health))
        .route("/", get(greet));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    println!("greet-service ฟังอยู่ที่ 0.0.0.0:8080");
    axum::serve(listener, app).await.unwrap();
}
```

**build จริง**:

```bash
$ docker build -t greet-naive:latest .
#8 44.93     Finished `release` profile [optimized] target(s) in 44.79s
#9 exporting to image
#9 naming to docker.io/library/greet-naive:latest done

real	1m17.491s
```

**วัดขนาด image จริง**:

```bash
$ docker images greet-naive:latest
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
greet-naive:latest   0a02d3a7e788       2.56GB          650MB
```

**2.56GB สำหรับ axum service ที่มี route แค่สองเส้น** — นี่คือปัญหาที่แท้จริงของแนวทางนี้ ลองแยกดูว่าอะไรกิน
พื้นที่ขนาดนี้: image `rust:latest` เองมีขนาดหลัก GB อยู่แล้ว (มี `rustc`, `cargo`, standard library ที่
precompile ไว้หลาย target, เครื่องมือ build เสริมอย่าง `git`, `curl` ฯลฯ ที่ Debian base image ของมันติดมาด้วย)
บวกกับ **`cargo build --release` เอง download+compile dependency ทั้งหมดไว้ใน `target/` ที่ยังอยู่ใน image
final** (ไม่ได้ถูกลบทิ้งเลย เพราะ image final กับ image ที่ใช้ compile คือ image เดียวกัน) บวกกับ **source code
ทั้งหมดที่ยัง `COPY . .` อยู่ในนั้นด้วย** — ทั้งสามอย่างนี้ **ไม่มีประโยชน์อะไรเลยตอนรันจริง** (ตอนรัน มีแค่
`./target/release/greet-service` ไฟล์เดียวที่ถูกเรียกจริงจาก `CMD`) แต่ยังกินพื้นที่ทุก byte อยู่ใน image ที่
จะถูก deploy ไปจริง

**ผลกระทบที่จับต้องได้ ไม่ใช่แค่เรื่อง "ดูไม่สวย"**:

1. **Deploy ช้าลงจริง** — image ที่ใหญ่กว่าใช้เวลา push ขึ้น container registry นานกว่า, pull ลง production
   server นานกว่า (คูณด้วยจำนวน instance ที่ scale ออกไปในเวลาเดียวกันถ้าเป็น auto-scaling) — ความต่างระหว่าง
   2.56GB กับ 100MB มีผลจริงเมื่อต้อง scale จาก 3 instance เป็น 30 instance กะทันหันตอน traffic พุ่ง
2. **Attack surface ใหญ่กว่าที่จำเป็น** — Rust toolchain เต็มรูปแบบ (`rustc`, `cargo`, และ dependency ของ
   toolchain เอง) ไม่มีเหตุผลอะไรที่ต้องอยู่ใน production image เลย ถ้ามีช่องโหว่ในเครื่องมือเหล่านี้ในอนาคต
   (CVE ของ `rustc`/`cargo`/library ที่ toolchain พึ่ง) production image ที่ไม่จำเป็นต้องมีมันเลยก็ยัง exposed
   ไปด้วยโดยไม่ได้ประโยชน์อะไรกลับมา — หลักการ **least privilege/least surface** บอกว่าไม่ควรมีอะไรใน image
   ที่ไม่ได้ใช้จริงตอนรัน
3. **source code รั่วไหลเข้า image โดยไม่จำเป็น** — ถ้า image ถูกเก็บใน registry ที่ไม่ปลอดภัยพอ หรือใครมี
   สิทธิ์ `docker pull`/`docker save` image นี้ได้ จะได้ source code ทั้งหมดไปด้วย (แม้ในกรณีนี้ source code
   อาจไม่ใช่ความลับ แต่ในโปรเจกต์จริงจำนวนมาก source code คือทรัพย์สินทางปัญญาที่ไม่ควรอยู่ใน image ที่ deploy
   เลยแม้แต่นิดเดียว)

**ทางแก้คือหัวข้อถัดไป**: multi-stage build — หลักการง่าย ๆ คือ "ใช้ image ใหญ่แค่ตอน build เท่านั้น แล้วเอา
แค่ผลลัพธ์ (compiled binary) ไปไว้ใน image เล็กที่สุดเท่าที่จำเป็นต้องรันจริง"

### 96.4 Multi-Stage Build: แยก Builder กับ Runtime อย่างถูกต้อง

**หลักการของ multi-stage build**: `Dockerfile` หนึ่งไฟล์สามารถมีหลาย `FROM` ได้ — แต่ละ `FROM` เริ่ม "stage"
ใหม่ที่เป็นอิสระจากกัน (filesystem ของแต่ละ stage แยกกันโดยสมบูรณ์) และสามารถ **copy ไฟล์จาก stage ก่อนหน้า
เข้ามาใน stage หลังได้** ด้วย `COPY --from=<stage name>` — image ที่ export ออกมาจริงตอน `docker build` คือ
image ของ **stage สุดท้าย** เท่านั้น (stage ก่อนหน้าถูกใช้แค่ตอน build แล้วไม่ถูกเก็บไว้ใน image final เลย
เว้นแต่จะสั่ง `--target <stage>` ระบุให้หยุดที่ stage นั้น)

```dockerfile
# Dockerfile.multistage
# --- Stage 1: builder — มี Rust toolchain เต็ม ใช้แค่ตอน build ---
FROM rust:1-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# --- Stage 2: runtime — เอาแค่ binary ที่ compile แล้วมาไว้ใน image เล็ก ---
FROM debian:bookworm-slim AS runtime
WORKDIR /app
COPY --from=builder /app/target/release/greet-service ./greet-service
EXPOSE 8080
CMD ["./greet-service"]
```

**อธิบายทีละจุด**:

- **`FROM rust:1-slim AS builder`** — ใช้ `rust:1-slim` (Debian slim แทน Debian ธรรมดาที่ `rust:latest` ใช้)
  เพราะ stage นี้ไม่ต้องมี tool เสริมอะไรมากกว่าที่ `cargo build` ต้องใช้จริง `1-slim` หมายถึง "Rust เวอร์ชัน
  major 1.x ล่าสุด บน Debian slim" — pin ที่ major version พอ (ไม่ pin ถึง patch version เป๊ะ) เพื่อได้ security
  patch ของ toolchain อัตโนมัติโดยไม่เสี่ยง breaking change ข้าม major version (ซึ่ง Rust รักษา backward
  compatibility ข้าม minor version อย่างเข้มงวดมาตลอดอยู่แล้ว)
- **`FROM debian:bookworm-slim AS runtime`** — stage ที่สองเริ่มจาก image คนละตัวโดยสิ้นเชิง (ไม่มี Rust
  toolchain ติดมาด้วยเลย) — `debian:bookworm-slim` มีแค่ libc/libgcc/libm พื้นฐานที่ทุก dynamically-linked
  Rust binary ต้องการ (ตามที่หัวข้อ 96.1 พิสูจน์ด้วย `ldd` ไว้แล้ว) ไม่มี `apt`/`curl`/compiler ใด ๆ ที่ไม่ได้
  ใช้ตอนรันจริง
- **`COPY --from=builder /app/target/release/greet-service ./greet-service`** — จุดสำคัญที่สุด: copy แค่
  **ไฟล์ binary เดียว** จาก stage แรกเข้า stage ที่สอง ไม่มี source code, ไม่มี `target/` ที่เหลือ (build
  artifact กลาง ๆ), ไม่มี Rust toolchain ติดมาด้วยเลยแม้แต่ byte เดียว

**build จริงและวัดขนาด**:

```bash
$ docker build -t greet-multistage:latest .
#11 44.01    Compiling greet-service v0.1.0 (/app)
#11 49.88     Finished `release` profile [optimized] target(s) in 49.70s
#13 naming to docker.io/library/greet-multistage:latest done

real	0m52.768s

$ docker images greet-multistage:latest
IMAGE                     ID             DISK USAGE   CONTENT SIZE   EXTRA
greet-multistage:latest   2b5b1cea7e45        116MB         28.9MB
```

**จาก 2.56GB (disk usage) เหลือ 116MB — เล็กลงกว่า 22 เท่า** และถ้าเทียบที่ content size (ขนาดจริงที่ต้อง
push/pull ผ่าน network หลังหัก layer ที่ share กับ image base อื่นที่อาจมีอยู่แล้วในเครื่อง) คือ **650MB เทียบ
28.9MB — เล็กลงกว่า 22 เท่าเช่นกัน** ตัวเลขทั้งสองชุดสอดคล้องกันเพราะ `rust:latest` (ที่ single-stage ใช้เป็น
runtime ด้วย) มีขนาดใหญ่กว่า `debian:bookworm-slim` มาก และ multi-stage ตัด Rust toolchain ทั้งหมดทิ้งไปจาก
image final โดยสิ้นเชิง

**พิสูจน์ว่า image ที่เล็กลงยังรันได้ถูกต้องทุกอย่าง**:

```bash
$ docker run -d --name greet-ms-test -p 18080:8080 greet-multistage:latest
$ curl -sS http://127.0.0.1:18080/health
{"service":"greet-service","status":"ok"}
$ docker logs greet-ms-test
greet-service ฟังอยู่ที่ 0.0.0.0:8080
```

**ตรวจสอบ dynamic linking ของ binary ที่ได้จาก multi-stage นี้ด้วย `ldd`** (extract binary ออกมาจาก container
ด้วย `docker cp` มาตรวจนอก container):

```bash
$ file greet-service-glibc-binary
greet-service-glibc-binary: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0,
BuildID[sha1]=115172481f787c7a9f382677de653a1cc556aa45, stripped
```

binary นี้ยัง **dynamically linked** อยู่ (ผูกกับ `libc.so.6`/`libgcc_s.so.1`/`libm.so.6` ของระบบตอนรัน ตามที่
หัวข้อ 96.1 แสดงไว้) — นี่คือเหตุผลที่ runtime stage **ต้อง**ใช้ image ที่มี glibc ตรงกัน (`debian:bookworm-slim`
ที่มี glibc เวอร์ชันที่รองรับ binary ที่ compile จาก `rust:1-slim` ซึ่งใช้ base OS เดียวกัน — Debian bookworm)
— ถ้าใช้ runtime image ที่เป็น distro/libc คนละตัวกัน (เช่น compile บน Debian แล้วรันบน Alpine ที่ใช้ musl
libc) binary จะรันไม่ได้เลย เพราะ dynamic linker ที่ binary คาดหวัง (`/lib64/ld-linux-x86-64.so.2`) ไม่มีอยู่ใน
Alpine — **นี่คือสิ่งที่หัวข้อถัดไปจะแก้ด้วยวิธีที่ต่างออกไปโดยสิ้นเชิง**: compile ให้เป็น static binary ที่ไม่
ต้องพึ่ง dynamic linker ของระบบเลยแม้แต่ตัวเดียว

**ก่อนไปหัวข้อถัดไป — เหตุผลที่ Dockerfile นี้ยังไม่ใช่เวอร์ชันที่ดีที่สุด**: สังเกตว่า `COPY . .` ในบรรทัดเดียว
ของ builder stage **ยังมีปัญหา cache invalidation แบบที่หัวข้อ 96.2 อธิบายไว้อยู่** — ทุกครั้งที่แก้โค้ดแม้แค่
ตัวอักษรเดียว `cargo build --release` ต้อง compile dependency ทั้งหมดใหม่จากศูนย์ (หัวข้อ 96.6 จะพิสูจน์ด้วย
ตัวเลขเวลาจริง) — Dockerfile ในหัวข้อนี้ตั้งใจเขียนแบบง่ายที่สุดก่อนเพื่อโฟกัสที่หลักการ multi-stage อย่างเดียว
ส่วนการแก้ปัญหา caching จะอยู่ในหัวข้อ 96.6

**ทำไมเลือก `rust:1-slim` เป็น builder ไม่ใช่ `rust:1-alpine` ในหัวข้อนี้**: ทั้งสอง image มี Rust toolchain
เต็มรูปแบบเหมือนกัน ต่างกันที่ base OS (`rust:1-slim` เป็น Debian, `rust:1-alpine` เป็น Alpine ที่ใช้ musl
libc เป็นค่าเริ่มต้น) — เลือก `rust:1-slim` ในหัวข้อนี้เพราะ runtime stage เป็น `debian:bookworm-slim` (glibc)
ให้ builder/runtime เป็น **ตระกูล libc เดียวกัน** ลดความเสี่ยงเรื่อง compatibility ระหว่าง build-time กับ
run-time ที่ตัวเองไม่ได้ตั้งใจ (แม้ในทางเทคนิค `rust:1-alpine` ก็ compile binary แบบ dynamic-linked-with-musl
ให้ได้เหมือนกัน แต่ต้องจับคู่กับ runtime ที่เป็น musl-based เท่านั้น เช่น `alpine` เปล่า ๆ ไม่ใช่
`debian:bookworm-slim`) — หัวข้อถัดไปจะสลับไปใช้ `rust:1-alpine` เป็น builder โดยเจตนา เพราะเป้าหมายเปลี่ยนเป็น
"compile ให้เป็น static binary ด้วย musl target โดยเฉพาะ" ซึ่ง Alpine เอื้อให้ทำได้โดยไม่ต้อง `apt-get install
musl-tools` เพิ่มบน Debian (ตามที่อธิบายไว้ในหัวข้อถัดไป) — **หลักการเลือก**: ถ้า runtime เป็น glibc-based
(`debian:bookworm-slim`/distroless-cc) ให้ builder เป็น glibc-based ตระกูลเดียวกันเพื่อความสอดคล้อง ถ้า
target สุดท้ายคือ static musl binary ให้ builder เป็น Alpine (หรือ Debian ที่เพิ่ม musl target ผ่าน `rustup
target add` ก็ทำได้เหมือนกัน เพียงแต่ Alpine สะดวกกว่าเพราะ musl คือ libc พื้นฐานของมันอยู่แล้ว)

### 96.5 Static Linking ด้วย `musl`: Image เล็กที่สุดด้วย `scratch`

Multi-stage build ในหัวข้อก่อนลด image เหลือ 116MB ซึ่งดีมากแล้วเทียบกับ 2.56GB ของ single-stage — แต่ยังมี
"เพดาน" อยู่ที่ขนาดของ `debian:bookworm-slim` เอง (ประมาณ 80MB) ที่ต้องมีเพราะ binary ยัง **dynamically
linked** และต้องพึ่ง glibc ของระบบ — คำถามคือ: ถ้าตัด dynamic linking ออกไปเลยได้ไหม แล้ว runtime image จะเล็ก
ลงไปถึงระดับไหน

**Static linking คืออะไร**: แทนที่จะให้ binary "อ้างถึง" library ที่ต้องมีอยู่ในระบบตอนรัน (dynamic/shared
linking แบบที่หัวข้อก่อนแสดงด้วย `ldd`) static linking คือการ **ฝัง code ของ library ทั้งหมดที่ต้องใช้ลงไปใน
ตัว binary เองตั้งแต่ตอน compile** — ผลคือ binary ที่ได้ไม่ต้องพึ่ง shared library ของระบบเลยแม้แต่ตัวเดียว
(ไม่ต้องมี dynamic linker, ไม่ต้องมี libc ของระบบ) รันได้บน Linux kernel เปล่า ๆ โดยตรง

**ทำไมต้องเปลี่ยน target เป็น `musl`**: Linux แจกจ่าย glibc (GNU libc) เป็น libc มาตรฐานของ distro ส่วนใหญ่
(Debian, Ubuntu, Fedora) แต่ glibc **ออกแบบมาให้ dynamic link เป็นหลัก** — การ static link กับ glibc ทำได้ใน
ทางเทคนิคแต่มีข้อจำกัดและ warning จำนวนมาก (glibc เอกสารเองก็ไม่ค่อยสนับสนุนโหมดนี้เต็มที่) — **musl libc** เป็น
libc implementation อีกตัวที่ออกแบบมาให้ static link ได้สะอาดกว่ามาก (เล็กกว่า, เขียนใหม่ทั้งหมดโดยไม่มีปัญหา
legacy ของ glibc) Rust รองรับ target `x86_64-unknown-linux-musl` มาตั้งนานแล้วสำหรับกรณีนี้โดยเฉพาะ

**ติดตั้ง target และ build**:

```dockerfile
# Dockerfile.musl
# --- Stage 1: builder — ใช้ rust:1-alpine (Alpine ใช้ musl เป็น libc พื้นฐานอยู่แล้ว
# ไม่ต้อง apt-get install musl-tools เพิ่มเลย ต่างจากการเพิ่ม musl target บน Debian-based image
# ที่ต้องพึ่ง apt-get install musl-tools ซึ่งอาจติดปัญหาถ้าเครือข่ายจำกัดการเข้าถึง deb.debian.org
# (ดูกับดักที่พบบ่อยข้อ 5) ---
FROM rust:1-alpine AS builder
WORKDIR /app
RUN rustup target add x86_64-unknown-linux-musl
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

# --- Stage 2: runtime — scratch คือ image ที่ "ว่างเปล่าที่สุด" ที่ Docker มีให้ ---
FROM scratch AS runtime
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/greet-service /greet-service
EXPOSE 8080
ENTRYPOINT ["/greet-service"]
```

**`FROM scratch` คืออะไร**: `scratch` ไม่ใช่ image จริง ๆ แต่เป็น **image พิเศษที่ Docker built-in ไว้ให้ที่ไม่
มี layer อะไรเลยแม้แต่ layer เดียว** (ไม่มี OS, ไม่มี shell, ไม่มี `/bin`, ไม่มีอะไรทั้งสิ้น) — เป็นจุดเริ่มต้นที่
"ว่างที่สุด" ที่จะ `COPY` ไฟล์อะไรเข้าไปก็ได้ตามต้องการ เหมาะกับ static binary ที่ไม่ต้องพึ่งอะไรจากระบบเลย
(ถ้า binary ยัง dynamically linked อยู่ การรันบน `scratch` จะ error ทันทีเพราะไม่มี dynamic linker/libc ให้
binary หา)

**build จริง**:

```bash
$ docker build -f Dockerfile.musl -t greet-musl:latest .
#9 74.43    Compiling greet-service v0.1.0 (/app)
#9 89.02     Finished `release` profile [optimized] target(s) in 1m 28s
#11 naming to docker.io/library/greet-musl:latest done

real	1m44.250s
```

**วัดขนาด image — นี่คือตัวเลขที่น่าตกใจที่สุดของบทนี้**:

```bash
$ docker images greet-musl:latest
IMAGE               ID             DISK USAGE   CONTENT SIZE   EXTRA
greet-musl:latest   2638a65c3408       2.18MB          692kB
```

เทียบ 3 แนวทางเต็ม ๆ ด้วยตัวเลขจริงทั้งหมดจากบทนี้:

| แนวทาง | Disk Usage | Content Size | ลดจาก naive |
|---|---|---|---|
| Single-stage (`rust:latest`) | 2.56GB | 650MB | — |
| Multi-stage (`debian:bookworm-slim`) | 116MB | 28.9MB | ~22 เท่า |
| Multi-stage + musl (`scratch`) | **2.18MB** | **692kB** | **~940 เท่า** |

**พิสูจน์ว่า binary นี้ static จริง — ไม่ใช่แค่คำกล่าวลอย ๆ** (extract binary ออกมาจาก image `scratch` แล้วรัน
`file`):

```bash
$ file greet-service-musl-binary
greet-service-musl-binary: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
static-pie linked, BuildID[sha1]=7f1709a2bca2be6ff3b929645dd82d084e24f303, stripped
	statically linked
```

`file` รายงานตรง ๆ ว่า **"statically linked"** (เทียบกับ binary จาก glibc build ในหัวข้อก่อนที่รายงาน
"dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2") — และการรัน `ldd` กับ binary นี้จะไม่ได้ output
เป็นรายการ shared library เลย (แสดง error แบบ "not a dynamic executable" เพราะไม่มี dynamic dependency ให้
แสดง) ยืนยันว่า image `scratch` ขนาด 2.18MB นี้**มีแค่ตัว binary เดียว ไม่มีอะไรอื่นเลยจริง ๆ** และมันยังรัน
ได้ปกติทุกอย่าง:

```bash
$ docker run -d --name greet-musl-test -p 18081:8080 greet-musl:latest
$ curl -sS http://127.0.0.1:18081/health
{"service":"greet-service","status":"ok"}
```

**Trade-off ของ static linking — ต้องเข้าใจให้ครบก่อนใช้จริง**: static linking ไม่ใช่ทางเลือกที่ "ดีกว่าเสมอ"
มันมีข้อแลกเปลี่ยนจริงที่ต้องรู้:

- **ไม่รองรับ crate ที่พึ่ง native library ของระบบผ่าน dynamic linking โดยตรง** — ที่ชัดที่สุดคือ crate ที่ใช้
  `native-tls`/OpenSSL แบบ dynamic (feature default ของหลาย crate ที่ทำ HTTPS) จะ **build ไม่ผ่านสำหรับ musl
  target** เพราะ OpenSSL ของระบบไม่มีให้ link แบบ static ได้ง่าย ๆ (ต้อง cross-compile OpenSSL แบบ static เอง
  ซึ่งยุ่งยากและเสี่ยงบั๊กมาก) — **นี่คือเหตุผลที่หลักสูตรนี้เลือก `rustls` เป็น TLS backend มาตั้งแต่ Part 70
  (feature `rustls` ของ `sqlx-cli`) และเลือก `jsonwebtoken` feature `rust_crypto` กับ `argon2` (pure-Rust
  implementation ไม่พึ่ง OpenSSL เลย) มาตั้งแต่ Part 74/92** — การเลือก dependency ที่ implement ด้วย pure
  Rust ทั้งหมด (ไม่มี FFI ไปเรียก C library ของระบบ) ทำให้ static linking แบบนี้**เป็นไปได้โดยไม่ต้องแก้โค้ด
  อะไรเลยแม้แต่บรรทัดเดียว** — หัวข้อ 96.10 (capstone) จะพิสูจน์เรื่องนี้ด้วยแอปจริงที่มี dependency set
  เดียวกับ Part 92
- **Binary ใหญ่ขึ้นเล็กน้อยเทียบกับ dynamic** (เพราะต้องฝัง code ของ library ทั้งหมดเข้าไปเอง ไม่ได้ share
  code กับ process อื่นผ่าน shared library ของระบบ) — จากตัวอย่างนี้ static binary มีขนาด 1,479,688 byte
  เทียบกับ dynamic binary ที่ 1,371,592 byte (ใหญ่ขึ้นประมาณ 8%) — แลกมากับการไม่ต้องมี libc ของระบบใน image
  เลยซึ่งประหยัดพื้นที่มากกว่าหลายสิบเท่า (ดังที่ตารางข้างบนแสดง)
- **Debug ยากขึ้นเล็กน้อยตอน production** — `scratch` ไม่มี shell, ไม่มี `curl`/`wget`, ไม่มีเครื่องมือ debug
  อะไรเลย ถ้าต้อง `docker exec` เข้าไปดูอะไรข้างใน container ตอนมีปัญหา จะทำไม่ได้เลย (ต่างจาก
  `debian:bookworm-slim` ที่ยังมี shell พื้นฐานให้ `docker exec -it ... sh` เข้าไปดูได้) — นี่คือเหตุผลที่บาง
  ทีมเลือกใช้ `gcr.io/distroless/cc` หรือ `gcr.io/distroless/static` (ของ Google) แทน `scratch` ตรง ๆ เพราะ
  distroless ยังมี certificate bundle/timezone data ให้ (ที่ `scratch` ไม่มีเลย) แต่ยังไม่มี shell/package
  manager ที่ไม่จำเป็น — เป็นจุดกึ่งกลางระหว่าง `scratch` กับ `debian:bookworm-slim`

**เมื่อไรควรใช้ musl+scratch เต็มที่**: เมื่อ dependency ทั้งหมดของโปรเจกต์เป็น pure-Rust (ไม่มี FFI ไปเรียก
native library ที่ static link ไม่ได้) และต้องการ image ที่เล็กและปลอดภัยที่สุดเท่าที่จะทำได้ (เช่น serverless
function, CLI tool ที่แจกจ่ายเป็น container, หรือ microservice ที่ scale บ่อยมากจนความเร็ว pull image มีผลจริง)
— เมื่อไรควรใช้ multi-stage + glibc แบบหัวข้อ 96.4 ธรรมดาแทน: เมื่อโปรเจกต์พึ่ง native library ที่ static link
ยาก (เช่น ต้องใช้ `native-tls`/OpenSSL จริง ๆ ด้วยเหตุผลบางอย่าง, หรือพึ่ง C library ภายนอกที่ไม่มี pure-Rust
alternative) หรือเมื่อต้องการความสะดวกตอน debug production มากกว่าความเล็กของ image

**สรุปทางเลือกของ runtime base image ทั้งหมดที่บทนี้พูดถึง เทียบกันเป็นตาราง** (ตัวเลขขนาดฐาน image เอง — ไม่
รวม binary ของแอป — อ้างอิงจากขนาดที่วัดได้จริงตลอดบทนี้และข้อมูลทั่วไปของแต่ละ image ที่เป็นที่รู้จัก):

| Runtime base | มี shell/debug tool ไหม | รองรับ dynamic-linked binary | เหมาะกับ |
|---|---|---|---|
| `debian:bookworm-slim` | มี (`sh`, coreutils พื้นฐาน) | ใช่ (glibc) | โปรเจกต์ทั่วไปที่ยังต้อง debug เข้า container บ่อย, มี native dependency ที่ static link ไม่ได้ |
| `gcr.io/distroless/cc` | ไม่มี shell แต่มี libc/certificate bundle | ใช่ (glibc) | ต้องการเล็กกว่า debian-slim แต่ยัง dynamic link อยู่ (เช่นพึ่ง `ca-certificates` สำหรับเรียก HTTPS ออก) |
| `alpine` (ใช้เป็น runtime ตรง ๆ ไม่ใช่ builder) | มี shell (`ash`) | ต้อง compile ด้วย musl target ให้ตรง libc | ต้องการ shell เล็ก ๆ ไว้ debug แต่ไม่ต้องการ `scratch` ที่ไม่มีอะไรเลย |
| `scratch` | ไม่มีอะไรเลยแม้แต่ byte เดียว | ต้องเป็น static binary เท่านั้น (เช่น musl target) | เล็กที่สุด/ปลอดภัยที่สุดเท่าที่เป็นไปได้ เมื่อ dependency ทั้งหมดเป็น pure-Rust |

**ข้อควรระวังเรื่อง `scratch`/distroless กับ HTTPS ออกไปข้างนอก**: ถ้าแอปต้องเรียก HTTPS ไปยัง service ภายนอก
(ไม่ใช่แค่ต่อ PostgreSQL/Redis แบบในบทนี้ที่เป็น plain TCP) ต้องมี **certificate bundle** (`ca-certificates`)
อยู่ใน image ด้วยเพื่อ verify ใบรับรอง TLS ของปลายทาง — `scratch` **ไม่มี certificate bundle ให้เลย** (ไม่มี
`/etc/ssl/certs/` เพราะไม่มี filesystem อะไรเลยนอกจาก binary ที่ copy เข้าไป) วิธีแก้คือ copy certificate
bundle จาก builder stage เข้ามาด้วยตรง ๆ (`COPY --from=builder /etc/ssl/certs/ca-certificates.crt
/etc/ssl/certs/ca-certificates.crt`) หรือใช้ `gcr.io/distroless/cc`/`gcr.io/distroless/static` ที่มี bundle
นี้ให้อยู่แล้วโดยไม่ต้อง copy เอง — แอปทุกตัวในบทนี้ (`greet-service`, `library_api_mini`) ไม่เรียก HTTPS ออก
ไปข้างนอกเลย (เชื่อม PostgreSQL/Redis แบบ plain TCP ในสภาพแวดล้อมทดสอบ) จึงไม่ต้องมี certificate bundle — ถ้า
โปรเจกต์ของคุณต้องเรียก third-party API ผ่าน HTTPS (เช่น payment gateway, external webhook) ต้องเพิ่มขั้นตอน
นี้เข้าไปเสมอไม่ว่าจะเลือก `scratch` หรือ distroless

### 96.6 Dependency Caching: ปัญหาคลาสสิกของ Rust + Docker และวิธีแก้

หัวข้อ 96.2 อธิบายกลไก layer caching ไว้แล้วว่า **เมื่อ layer หนึ่ง cache miss ทุก layer ที่ตามมาจะ cache miss
ไปด้วย** — Dockerfile ทุกตัวที่เขียนมาในหัวข้อ 96.3-96.5 ใช้ `COPY . .` ในบรรทัดเดียว ซึ่งหมายความว่า **ทุกครั้ง
ที่แก้โค้ดแอปเองแม้แค่ตัวอักษรเดียว (ไม่แก้ dependency เลย) `cargo build --release` ต้อง compile dependency
ทั้งหมดใหม่จากศูนย์** — นี่คือปัญหาคลาสสิกที่สุดของการทำ Docker กับโปรเจกต์ Rust (และภาษา compiled อื่น ๆ ที่มี
dependency จำนวนมากคล้ายกัน) เพราะ **การ compile dependency มักกินเวลามากกว่าการ compile โค้ดของเราเองมาก** —
มาพิสูจน์ด้วยตัวเลขจริงว่าปัญหานี้ใหญ่แค่ไหน แล้วดูสองวิธีแก้ที่ใช้จริงในโลกทำงาน

**การทดลอง**: ใช้ `greet-service` เดิม (Dockerfile ที่มี `COPY . .` บรรทัดเดียวแบบหัวข้อ 96.4) เทียบกับ
Dockerfile ที่แยก `COPY Cargo.toml`/`Cargo.lock` ออกจาก `COPY src` — ทำ 4 การทดลองตามลำดับ:

**T1 — Dockerfile เดิม (`COPY . .`), cold build** (build ครั้งแรกโดยไม่มี cache อะไรเลย):

```bash
$ time docker build -f Dockerfile.multistage -t greet-naive-cache:latest .
real	1m25.956s
```

**T2 — Dockerfile เดิม, rebuild หลังแก้ src/main.rs เพียงบรรทัดเดียว** (แก้แค่ string ข้อความ ไม่แตะ
`Cargo.toml`/`Cargo.lock` เลย):

```bash
$ sed -i 's/สวัสดีจาก greet-service/สวัสดีจาก greet-service (v2)/' src/main.rs
$ time docker build -f Dockerfile.multistage -t greet-naive-cache:latest .
real	1m6.664s

$ grep -c "Compiling" build.log
57
```

**สังเกตให้ดี**: แก้แค่ 1 บรรทัดใน `main.rs` แต่ build log แสดง **"Compiling" 57 ครั้ง** — นี่คือ dependency
ทุกตัว (axum, tokio, serde, และ transitive dependency ทั้งหมดของทั้งสามตัว) ที่ต้อง compile ใหม่หมด ทั้งที่ไม่มี
อะไรในตัว dependency เหล่านี้เปลี่ยนแปลงเลยแม้แต่ byte เดียว — สาเหตุตรงกับที่หัวข้อ 96.2 อธิบายไว้เป๊ะ: `COPY .
.` ทำให้ layer นั้น cache miss เมื่อไฟล์ไหนในโฟลเดอร์เปลี่ยน (ไม่ว่าจะเป็น dependency file หรือ source file ก็
ตาม) แล้ว `RUN cargo build --release` ที่ตามมาก็ cache miss ไปด้วยตามกฎ chain — และเพราะ `cargo build` รันใน
container ที่เพิ่งถูกสร้างใหม่จาก layer ที่ cache miss นั้น (ไม่มี `target/` เดิมให้ใช้ต่อ) มันจึงต้อง compile
ทุกอย่างใหม่จริง ๆ

**T3 — Dockerfile ที่แยก manifest ออกจาก source (เทคนิคที่ถูกต้อง), cold build**:

```dockerfile
# Dockerfile.cached
FROM rust:1-slim AS builder
WORKDIR /app
# ขั้นที่ 1: copy แค่ manifest ก่อน — layer นี้ invalidate เฉพาะเมื่อ dependency เปลี่ยนจริง ๆ
COPY Cargo.toml Cargo.lock ./
# ขั้นที่ 2: สร้าง src ปลอมที่ compile ผ่านแน่นอน เพื่อ "warm" การ compile dependency ทั้งหมดไว้ก่อน
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
# ขั้นที่ 3: ลบ src ปลอมทิ้ง แล้ว copy source code จริงเข้ามาทับ (layer นี้เปลี่ยนทุกครั้งที่แก้โค้ด
# แต่ layer ก่อนหน้า (dependency build) ยังถูก cache ไว้เหมือนเดิมถ้า Cargo.toml/Cargo.lock ไม่เปลี่ยน)
RUN rm -rf src
COPY src ./src
RUN touch src/main.rs && cargo build --release

FROM debian:bookworm-slim AS runtime
WORKDIR /app
COPY --from=builder /app/target/release/greet-service ./greet-service
EXPOSE 8080
CMD ["./greet-service"]
```

```bash
$ time docker build -f Dockerfile.cached -t greet-cached:latest .
real	1m2.861s

$ grep -c "Compiling" build.log
58
```

cold build ของ Dockerfile.cached ใช้เวลาใกล้เคียง T1 (สมเหตุสมผล เพราะยังไม่มี cache อะไรให้ใช้เลยทั้งคู่ ต้อง
compile ทุกอย่างเหมือนกัน) — ตัวเลขที่น่าสนใจจริง ๆ อยู่ที่การทดลองถัดไป

**T4 — Dockerfile ที่แยก manifest, rebuild หลังแก้ src/main.rs เพียงบรรทัดเดียว** (การทดลองที่ตรงกับสถานการณ์
จริงที่สุด — แก้โค้ดแอปแต่ dependency ไม่เปลี่ยน):

```bash
$ sed -i 's/(v2)/(v3 -- app-only change)/' src/main.rs
$ time docker build -f Dockerfile.cached -t greet-cached:latest .
```

```
#8 [builder 2/8] WORKDIR /app
#8 CACHED

#9 [builder 3/8] COPY Cargo.toml Cargo.lock ./
#9 CACHED

#10 [builder 4/8] RUN mkdir -p src && echo "fn main() {}" > src/main.rs
#10 CACHED

#11 [builder 5/8] RUN cargo build --release
#11 CACHED

#12 [builder 6/8] RUN rm -rf src
#12 CACHED

#13 [builder 7/8] COPY src ./src
#13 DONE 0.2s

#14 [builder 8/8] RUN touch src/main.rs && cargo build --release
#14 0.185    Compiling greet-service v0.1.0 (/app)
#14 6.516     Finished `release` profile [optimized] target(s) in 6.39s
#14 DONE 6.6s

real	0m7.820s
```

**สรุปผลการทดลองทั้ง 4 ครั้งเทียบกันเป็นตาราง**:

| การทดลอง | Dockerfile | สถานการณ์ | เวลา build จริง | จำนวน "Compiling" |
|---|---|---|---|---|
| T1 | `COPY . .` | cold build | 1m25.956s | (compile ทุกอย่าง) |
| T2 | `COPY . .` | แก้แค่โค้ดแอป | **1m6.664s** | **57** (dependency ทั้งหมด) |
| T3 | แยก manifest | cold build | 1m2.861s | (compile ทุกอย่าง) |
| T4 | แยก manifest | แก้แค่โค้ดแอป | **7.820s** | **1** (แค่ crate ของเราเอง) |

**T2 เทียบ T4 คือตัวเลขที่สำคัญที่สุด**: สถานการณ์เดียวกันเป๊ะ (แก้โค้ดแอปแค่บรรทัดเดียว ไม่แตะ dependency) แต่
ใช้เวลาต่างกัน **1m6.664s เทียบ 7.820s — เร็วขึ้นประมาณ 8.5 เท่า** และในทางปฏิบัติ ยิ่ง dependency tree ของ
โปรเจกต์ใหญ่ขึ้น (โปรเจกต์จริงที่มี `sqlx`/`utoipa`/`leptos` แบบ Part 92-94 อาจมี dependency นับร้อย) ความต่างนี้
จะยิ่งมากขึ้นเรื่อย ๆ เพราะ T2 ต้อง compile dependency ทั้งหมดใหม่เสมอไม่ว่าจะมีกี่ตัว ในขณะที่ T4 ยังคง compile
แค่ crate ของเราเองเท่านั้นไม่ว่า dependency tree จะใหญ่แค่ไหน — สำหรับทีมที่ build image ทุกครั้งที่ push commit
(ปกติของ CI/CD ที่ Part 97 จะสอน) ความต่างนี้สะสมเป็นเวลารอที่มีผลจริงต่อความเร็วของ feedback loop ทั้งทีม

**ทางเลือกที่สอง: `cargo-chef`** — เทคนิคการแยก manifest ข้างบนใช้ได้ดี แต่มีข้อจำกัดหนึ่ง: มันต้อง "ปลอม"
`src/main.rs` เปล่า ๆ ขึ้นมาเพื่อ compile dependency ก่อน ซึ่งใช้ได้ดีกับโปรเจกต์ที่มี binary เดียว แต่ซับซ้อนขึ้น
เมื่อโปรเจกต์มีหลาย binary/crate (ต้องปลอมทุกไฟล์ entry point ให้ตรงกับที่ `Cargo.toml` ประกาศไว้) —
[`cargo-chef`](https://github.com/LukeMathWalker/cargo-chef) คือเครื่องมือที่เขียนมาเพื่อแก้ปัญหานี้โดยเฉพาะ:
มันวิเคราะห์ dependency graph ของโปรเจกต์ (ไม่ว่าจะมีกี่ crate/binary) แล้วสร้างไฟล์ "recipe" ที่พอสำหรับ
compile dependency ล่วงหน้าได้ โดยไม่ต้องปลอมไฟล์ source เองเลย:

```dockerfile
# Dockerfile.chef
FROM rust:1-slim AS chef
RUN cargo install cargo-chef --locked
WORKDIR /app

# ขั้นที่ 1: วิเคราะห์ dependency graph ของโปรเจกต์ แล้วสร้าง "recipe.json" (ไม่มีซอร์สโค้ดจริงอยู่ในนั้น
# มีแค่ข้อมูล dependency ที่จำเป็นต่อการ compile deps ล่วงหน้า)
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# ขั้นที่ 2: cook — compile เฉพาะ dependency ตาม recipe.json (layer นี้ cache ได้ตราบใดที่
# dependency graph ไม่เปลี่ยน แม้จะยังไม่เห็นซอร์สโค้ดจริงของเราเลยก็ตาม)
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
# ขั้นที่ 3: ค่อย copy ซอร์สโค้ดจริงเข้ามา แล้ว build ต่อจาก dependency ที่ cook ไว้แล้ว
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim AS runtime
WORKDIR /app
COPY --from=builder /app/target/release/greet-service ./greet-service
EXPOSE 8080
CMD ["./greet-service"]
```

**build จริง** (ครั้งแรก — รวมเวลาติดตั้ง `cargo-chef` เองด้วย ซึ่งเป็นต้นทุนครั้งเดียวที่ Docker cache ไว้ได้
ในการ build ครั้งต่อ ๆ ไป):

```bash
$ time docker build -f Dockerfile.chef -t greet-chef:latest .
#13 [builder 2/4] RUN cargo chef cook --release --recipe-path recipe.json
#13 21.02    Compiling axum v0.8.9
#13 28.65    Compiling greet-service v0.0.1 (/app)     <-- package หลอกจาก recipe.json (ยังไม่มี src จริง)
#13 28.79     Finished `release` profile [optimized] target(s) in 28.65s

#14 [builder 3/4] COPY . .
#14 DONE 0.1s

#15 [builder 4/4] RUN cargo build --release
#15 0.190    Compiling greet-service v0.1.0 (/app)      <-- คราวนี้คือ package จริง (v0.1.0 ไม่ใช่ v0.0.1)
#15 3.588     Finished `release` profile [optimized] target(s) in 3.45s

real	2m28.462s
```

สังเกตว่า `cargo chef cook` compile dependency ทั้งหมดเสร็จใน layer ของมันเอง (28.65s) ภายใต้ package หลอกชื่อ
`greet-service v0.0.1` (เวอร์ชันปลอมที่ `cargo-chef` สร้างจาก `recipe.json` โดยไม่มี source code จริงของเรา
เลย) จากนั้น `COPY . .` ตามด้วย `cargo build --release` (layer ที่แยกออกมาต่างหาก) **compile แค่ครั้งเดียวคือ
`greet-service v0.1.0` (package จริงของเรา) ใช้เวลาเพียง 3.45 วินาที** — หลักการเดียวกับเทคนิคแยก manifest
เป๊ะ แต่ `cargo-chef` จัดการความซับซ้อนของหลาย crate/binary ให้อัตโนมัติโดยไม่ต้องเขียน dummy source เอง —
ข้อเสียคือต้องเสียเวลา `cargo install cargo-chef` ครั้งแรก (ซึ่งใน production มักแก้ด้วยการใช้ image
`lukemathwalker/cargo-chef` ที่มี `cargo-chef` ติดตั้งไว้แล้วสำเร็จรูปแทนการ `cargo install` เอง ถ้าเครือข่ายเข้า
ถึง Docker Hub ได้)

**เลือกอย่างไรระหว่างสองเทคนิค**: สำหรับโปรเจกต์ binary เดียว (แบบ `greet-service`) เทคนิคแยก manifest ธรรมดา
(T3/T4) เพียงพอและไม่ต้องพึ่ง tool เพิ่ม — สำหรับ workspace ที่มีหลาย crate/binary (แบบหัวข้อ 96.7 ถัดไป) หรือ
โปรเจกต์ที่ dependency graph ซับซ้อนมาก `cargo-chef` ช่วยลดความยุ่งยากของการปลอม dummy source ในหลายที่พร้อมกัน
ได้มาก — ทั้งสองเทคนิคแก้ปัญหาเดียวกัน (แยก layer ของ dependency ออกจาก layer ของ source code) เพียงแค่ระดับ
ความสะดวกต่างกัน

**ทางเลือกที่สาม: BuildKit cache mount (`--mount=type=cache`) — ไม่ต้องแยก manifest หรือปลอม dummy source
เลยแม้แต่ไฟล์เดียว**: ทั้งสองเทคนิคข้างบนแก้ปัญหาด้วยการ "จัดลำดับ layer ให้ dependency อยู่ก่อน source" แต่
Docker BuildKit (builder ที่ Docker เวอร์ชันปัจจุบันใช้เป็น default) มีกลไกอีกแบบที่แก้ปัญหาเดียวกันจากอีกมุม:
**cache mount** คือ persistent volume ที่ BuildKit จัดการให้ ผูกกับ `RUN` instruction หนึ่งบรรทัด — เนื้อหาใน
mount นี้ **อยู่ข้ามการ build แต่ละครั้งได้โดยไม่ขึ้นกับ Docker layer cache ปกติเลย** (แม้ layer ที่ `RUN` นั้น
อยู่จะ cache miss จาก `COPY` ก่อนหน้าที่เปลี่ยนไป cache mount ก็ยังมีเนื้อหาเดิมให้ใช้ต่อ):

```dockerfile
# Dockerfile.cachemount — ไม่มีการแยก manifest หรือปลอม src เลย ใช้ COPY . . ตรง ๆ แบบเดียวกับ
# Dockerfile.naive ของหัวข้อ 96.3 ทุกประการ ต่างกันแค่บรรทัด RUN บรรทัดเดียว
FROM rust:1-slim AS builder
WORKDIR /app
COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release && cp target/release/greet-service /greet-service-out

FROM debian:bookworm-slim AS runtime
WORKDIR /app
COPY --from=builder /greet-service-out ./greet-service
EXPOSE 8080
CMD ["./greet-service"]
```

**อธิบายสองบรรทัด `--mount`**: `target=/usr/local/cargo/registry` ผูก cache mount เข้ากับตำแหน่งที่ `cargo`
เก็บ crate ที่ download มาแล้ว (ไม่ต้อง download ซ้ำทุกครั้งที่ build แม้ layer ของ `COPY` จะเปลี่ยนไปก็ตาม) และ
`target=/app/target` ผูกเข้ากับ Cargo's build cache directory เอง (ผลลัพธ์ compile ของ dependency ที่ไม่
เปลี่ยน ยังถูกเก็บไว้ให้ใช้ต่อได้ — นี่คือกลไกเดียวกับที่ทำให้ Cargo's incremental compilation เร็วขึ้นตอนพัฒนา
บนเครื่องปกติ เพียงแต่ตอนนี้ persist ข้าม `docker build` ได้ด้วย) — สังเกตว่า `cp target/release/greet-service
/greet-service-out` ในบรรทัดเดียวกันจำเป็น**ต้องมี**เพราะไฟล์ที่อยู่ใต้ cache mount (`/app/target`) จะ**หายไป
หลัง `RUN` นั้นจบ** (cache mount ถูก unmount ทันทีที่ instruction เสร็จ ไม่ได้กลายเป็นส่วนของ layer image เลย)
ถ้าไม่ copy binary ออกมาไว้ที่อื่นก่อน stage runtime จะหา `target/release/greet-service` ไม่เจอ

**build จริง — cold build (ไม่มี cache mount อะไรอยู่เลย)**:

```bash
$ docker build --no-cache -f Dockerfile.cachemount -t greet-cachemount:latest .
#11 15.05    Compiling greet-service v0.1.0 (/app)
#11 16.83     Finished `release` profile [optimized] target(s) in 16.74s
#11 DONE 17.1s

real	0m27.130s
```

**rebuild หลังแก้ `src/main.rs` เพียงบรรทัดเดียว (Dockerfile เดิมเป๊ะ, ยังมี `COPY . .` บรรทัดเดียวไม่เปลี่ยน)**:

```bash
$ sed -i 's/ok", "service/ok-v2", "service/' src/main.rs
$ time docker build -f Dockerfile.cachemount -t greet-cachemount:latest .
#9 [builder 3/4] COPY . .
#9 DONE 0.0s

#10 [builder 4/4] RUN --mount=type=cache,... cargo build --release && cp ...
#10 0.122    Compiling greet-service v0.1.0 (/app)
#10 1.973     Finished `release` profile [optimized] target(s) in 1.89s
#10 DONE 2.0s

real	0m3.277s
```

**สังเกตให้ดี: layer `COPY . .` (`#9`) แสดง `DONE` ไม่ใช่ `CACHED`** — หมายความว่า layer นี้ cache miss จริง
ตามที่คาด (เนื้อหา `src/` เปลี่ยน) แต่ `RUN` ที่ตามมากลับ **compile แค่ `greet-service` ตัวเดียว (1.89 วินาที)
ไม่ต้อง compile dependency ใหม่เลยแม้ layer ก่อนหน้าจะ cache miss ไปแล้วก็ตาม** — นี่คือความต่างสำคัญจากกฎ
"cache miss แล้ว invalidate ทุก layer ที่ตามมา" ของหัวข้อ 96.2: **กฎนั้นใช้กับ Docker layer cache ปกติเท่านั้น
ไม่ใช้กับเนื้อหาที่อยู่ใน cache mount** เพราะ cache mount ไม่ใช่ "layer" ในความหมายที่ image เก็บไว้ มันเป็น
persistent storage ที่ BuildKit ดูแลแยกต่างหากโดยสิ้นเชิง

**ข้อดี/ข้อเสียเทียบกับสองเทคนิคก่อนหน้า**: cache mount **เขียนง่ายที่สุด** (ไม่ต้องแยก manifest, ไม่ต้องปลอม
dummy source, ไม่ต้องติดตั้ง tool เพิ่มแบบ `cargo-chef`) และใช้ได้กับ `COPY . .` ตรง ๆ แบบเดียวกับ Dockerfile
ที่ "ไม่ดี" ของหัวข้อ 96.3 ทุกประการ — แต่มีข้อจำกัดที่ต้องรู้: (1) **cache mount อยู่บนเครื่อง/CI runner ที่รัน
`docker build` เท่านั้น** เหมือนกับ Docker layer cache ปกติ (ไม่ export/import ข้าม runner โดย default เหมือน
กับข้อจำกัดที่หัวข้อ 96.2 อธิบายไว้ท้ายหัวข้อ) — CI runner ที่เป็น container สดใหม่ทุกครั้งจะไม่เห็นประโยชน์นี้
เลยเว้นแต่ตั้งค่า persistent cache volume ให้ CI โดยเฉพาะ (2) **cache mount ไม่ได้ผูกกับ project/Dockerfile
ใดโดยเฉพาะ** — ถ้ามีหลายโปรเจกต์ที่ build บนเครื่องเดียวกันและใช้ `target=/usr/local/cargo/registry` เดียวกัน
เนื้อหาจะถูก share ข้ามโปรเจกต์ (ซึ่งมักเป็นประโยชน์สำหรับ registry cache แต่ต้องตั้ง `id=` ให้ชัดเจนถ้าต้องการ
แยก cache ของ `target/` ระหว่างโปรเจกต์ที่ไม่เกี่ยวข้องกัน เพื่อป้องกัน cache ขนาดใหญ่ผิดปกติจากการสะสมของหลาย
โปรเจกต์ปนกัน)

**สรุปสามเทคนิคของหัวข้อนี้**: ทั้งแยก manifest ด้วยมือ, `cargo-chef`, และ `--mount=type=cache` แก้ปัญหาเดียวกัน
(compile dependency ซ้ำโดยไม่จำเป็น) ด้วยกลไกที่ต่างกัน — โปรเจกต์จริงจำนวนมากใช้ **ผสมกัน** ได้ด้วย (เช่น
`cargo-chef` สำหรับจัดการ dummy source ของหลาย crate ให้อัตโนมัติ ร่วมกับ cache mount สำหรับ registry cache
เพื่อไม่ต้อง download crate ซ้ำแม้ `recipe.json` จะเปลี่ยน) — ไม่มีทางเลือกที่ "ถูกที่สุด" ตายตัว ขึ้นอยู่กับ
ว่าทีมต้องการความง่ายในการอ่าน Dockerfile (cache mount ชนะ เพราะ diff จาก naive Dockerfile มีแค่บรรทัดเดียว)
หรือความสามารถในการย้าย cache ข้าม CI runner ผ่าน registry cache export (`cargo-chef`/แยก manifest ทำงานร่วม
กับกลไกนั้นได้ตรงไปตรงมากว่า เพราะผลลัพธ์อยู่ใน layer image จริงที่ export/import ได้ ไม่ใช่ cache mount ที่
ผูกกับเครื่อง build เท่านั้น)

### 96.7 Multi-Crate Workspace: Build เฉพาะ Binary ที่ต้องการ

Part 17 หัวข้อ 17.5-17.9 สอนไว้ว่าโปรเจกต์ที่โตขึ้นมักแตกเป็นหลาย crate ภายใน workspace เดียว (เช่น
`fleet-core` เป็น shared library, `api` และ `worker` เป็น binary คนละตัวที่ใช้ library เดียวกัน) — คำถามที่
เกิดขึ้นตามมาตอน containerize คือ: **ถ้า workspace มีหลาย binary แต่ image หนึ่งใบต้องรันแค่ binary เดียว
(เช่น container ของ API ไม่ควรมี binary ของ worker ติดไปด้วย) จะเขียน Dockerfile อย่างไร**

**โครงสร้าง workspace ตัวอย่าง** (ต่อยอด pattern จาก Part 17 หัวข้อ 17.6 ตรง ๆ):

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = ["fleet-core", "api", "worker"]
```

```toml
# api/Cargo.toml
[package]
name = "api"
version = "0.1.0"
edition = "2021"

[dependencies]
fleet-core = { path = "../fleet-core" }
```

`worker/Cargo.toml` มีโครงสร้างเดียวกัน (path dependency ไปที่ `fleet-core` เหมือนกัน) — ทั้ง `api` และ
`worker` ใช้ library กลาง (`fleet-core`) ร่วมกัน แต่เป็นคนละ binary ที่ทำหน้าที่ต่างกันโดยสิ้นเชิง (API server
เทียบกับ background worker)

**Dockerfile ที่ build เฉพาะ `api` โดยไม่แตะ `worker` เลย**:

```dockerfile
FROM rust:1-slim AS builder
WORKDIR /app
# copy ทั้ง workspace เข้ามา (ต้องมี Cargo.toml ของทุก member เพื่อให้ cargo resolve dependency graph
# ของทั้ง workspace ได้ถูกต้อง แม้ image สุดท้ายจะมีแค่ binary เดียวก็ตาม)
COPY Cargo.toml Cargo.lock ./
COPY fleet-core ./fleet-core
COPY api ./api
COPY worker ./worker
# build เฉพาะ crate "api" ตัวเดียวจากทั้ง workspace ด้วย -p (ต่อยอด Part 17 หัวข้อ 17.9)
# ไม่ build "worker" เลย -- ประหยัดเวลา build และไม่มี worker binary หลงเหลือใน image นี้
RUN cargo build --release -p api

FROM debian:bookworm-slim AS runtime
WORKDIR /app
COPY --from=builder /app/target/release/api ./api
CMD ["./api"]
```

**จุดสำคัญคือ flag `-p api`** (ตัวย่อของ `--package api`) — บอก Cargo ว่า "build แค่ crate ชื่อ `api` และสิ่งที่
`api` ต้องใช้ (คือ `fleet-core`) เท่านั้น ไม่ต้อง build `worker`" แม้ทั้งสาม crate จะอยู่ใน `Cargo.lock` ไฟล์
เดียวกันและถูก `COPY` เข้ามาทั้งหมดก็ตาม (การ `COPY` ทั้ง workspace เข้ามาจำเป็น เพราะ Cargo ต้อง "เห็น" โครงสร้าง
ทั้ง workspace เพื่อ resolve dependency graph ให้ถูกต้อง แม้จะสั่ง build แค่ crate เดียวก็ตาม — แต่ **การ compile
จริง** เกิดขึ้นแค่กับ `api`/`fleet-core` เท่านั้น)

**build จริงและพิสูจน์ว่า `worker` ไม่ถูก compile เลย**:

```bash
$ docker build -t fleet-api:latest .
#14 [builder 7/7] RUN cargo build --release -p api
#14 0.302    Compiling fleet-core v0.1.0 (/app/fleet-core)
#14 0.352    Compiling api v0.1.0 (/app/api)
#14 0.489     Finished `release` profile [optimized] target(s) in 0.32s
```

**build log แสดง "Compiling" แค่สองครั้ง — `fleet-core` และ `api` — ไม่มี "Compiling worker" เลยแม้แต่ครั้ง
เดียว** (ตัวเลขเวลา build เร็วมาก 0.32 วินาที เพราะทั้งสาม crate นี้เป็น binary เปล่า ๆ ไม่มี external
dependency เลย ในโปรเจกต์จริงตัวเลขนี้จะสูงกว่านี้มาก แต่หลักการเดียวกัน — สิ่งที่ต้องดูคือ "Compiling" ไม่มี
`worker` ปรากฏ ไม่ใช่ตัวเลขเวลาที่ต่ำผิดปกติ) — ตรวจสอบ image final ว่ามีแค่ `api` binary จริง:

```bash
$ docker run --rm --entrypoint sh fleet-api:latest -c "ls -la /app"
total 456
-rwxr-xr-x 1 root root 458192 Sep 27 05:32 api

$ docker run --rm fleet-api:latest
fleet api binary กำลังฟัง HTTP request

$ docker images fleet-api:latest
IMAGE              ID             DISK USAGE   CONTENT SIZE   EXTRA
fleet-api:latest   4efb9a309556        114MB         28.4MB
```

**มีแค่ `api` ไฟล์เดียวใน `/app` — ไม่มี `worker` หลงเหลืออยู่เลย** ตรงกับที่คาดไว้ — สำหรับ `worker` ต้องเขียน
Dockerfile คนละไฟล์ (เช่น `Dockerfile.worker`) ที่สั่ง `cargo build --release -p worker` แทน แล้ว copy
`target/release/worker` เข้า runtime stage ของมันเอง — ผลคือได้ **สอง image ที่แยกกันสมบูรณ์** จาก workspace
เดียวกัน แต่ละ image มีแค่ binary ที่จำเป็นต่อบทบาทของมันเท่านั้น ตรงกับหลักการ "deploy หน่วยที่เล็กที่สุดที่
จำเป็น" — สำหรับระบบที่ต้อง scale `api` กับ `worker` แยกกันตามภาระงาน (เช่น `api` ต้อง scale ตาม traffic
ในขณะที่ `worker` ต้อง scale ตามจำนวน job ในคิว) การแยก image แบบนี้เป็นสิ่งจำเป็น ไม่ใช่แค่ทางเลือก

### 96.8 Environment Variables, Config และ Secrets ใน Container

Part 70/74/94 ตั้งหลักการไว้แล้วว่า config (เช่น `DATABASE_URL`, `JWT_SECRET`) ต้องอ่านจาก environment
variable เสมอ ไม่ hardcode ในโค้ด — หลักการนี้มาจาก **twelve-factor app** (แนวปฏิบัติมาตรฐานสำหรับแอปที่
deploy บน cloud/container ที่เขียนไว้ตั้งแต่ยุคแรกของ Heroku) ข้อที่สามซึ่งบอกว่า **"Store config in the
environment"** — เหตุผลเชิงลึกคือ **config ต่างกันไปตาม environment (dev/staging/production) แต่ code ควรเป็น
artifact เดียวกันเป๊ะที่ deploy ไปทุกที่** ถ้า config ฝังอยู่ในโค้ด (หรือแย่กว่านั้นคือ built เข้าไปใน image
ตอน build) จะต้อง build image คนละตัวสำหรับแต่ละ environment ซึ่งขัดกับหลักการที่ว่า "image ตัวเดียวกันเป๊ะ
ต้อง deploy ได้ทุก environment โดยเปลี่ยนแค่ environment variable ตอน run"

**`ARG` เทียบ `ENV` ใน Dockerfile — สองอย่างที่คนสับสนบ่อย**:

- **`ARG`** — ตัวแปรที่มีค่าแค่**ตอน build** เท่านั้น (ผ่าน `--build-arg` ตอน `docker build`) ไม่ persist เข้า
  image final และไม่มีอยู่ตอน container รันจริงเลย — ใช้สำหรับค่าที่ต้องรู้แค่ตอน compile (เช่น
  `API_BASE` ที่ Part 94 หัวข้อ 94.5 ใช้ตอน `trunk build --release` เพื่อฝัง URL เข้าไปใน WASM bundle)
- **`ENV`** — ตัวแปรที่ set เข้า image และ**persist ไปจนถึง container ที่รันจริง** (เว้นแต่จะถูก override
  ตอน `docker run -e`/`docker-compose environment:`) — ใช้สำหรับค่า default ที่ไม่ใช่ความลับ (เช่น
  `ENV BIND_ADDR=0.0.0.0:8080` ที่หัวข้อ 96.10 จะใช้)

**กฎเหล็กที่ต้องจำ: ห้ามใส่ secret ผ่าน `ARG`/`ENV` ที่ build เข้า image เด็ดขาด** — ไม่ใช่แค่เพราะมัน "ดูไม่ดี"
แต่เพราะ**มันกู้กลับมาได้จริงแม้จะพยายาม "ลบ" ทิ้งในเลเยอร์ถัดไปแล้วก็ตาม** ต่อยอดจากหลักการ layer/diff ที่หัวข้อ
96.2 อธิบายไว้ (แต่ละ layer เก็บ diff ของ filesystem ไม่ใช่ snapshot ที่เขียนทับของเก่า) — มาพิสูจน์ด้วยการ
ทดลองจริง

**การทดลอง: bake secret เข้า layer แล้ว "ลบ" ในเลเยอร์ถัดไป**:

```dockerfile
FROM alpine:3.20
ARG DATABASE_PASSWORD
# ผิดพลาด: bake secret เข้าไปเป็นไฟล์ในเลเยอร์นี้โดยตรง
RUN echo "DATABASE_PASSWORD=${DATABASE_PASSWORD}" > /app-secret.env
RUN cat /app-secret.env
# พยายาม "ลบ" ทิ้งในเลเยอร์ถัดไป -- ทำให้ไฟล์มองไม่เห็นในระบบไฟล์สุดท้าย แต่ "ไม่ได้ลบ" ข้อมูลจากเลเยอร์เดิม
RUN rm /app-secret.env
CMD ["sh", "-c", "echo container started; sleep 3600"]
```

```bash
$ docker build --build-arg DATABASE_PASSWORD="SuperSecretPass123" -t secret-leak-demo:latest .
#5 [2/4] RUN echo "DATABASE_PASSWORD=SuperSecretPass123" > /app-secret.env
#6 [3/4] RUN cat /app-secret.env
#6 0.106 DATABASE_PASSWORD=SuperSecretPass123
#7 [4/4] RUN rm /app-secret.env

 1 warning found (use docker --debug to expand):
 - SecretsUsedInArgOrEnv: Do not use ARG or ENV instructions for sensitive data (ARG "DATABASE_PASSWORD") (line 2)
```

**สังเกตว่า Docker เองเตือนไว้แล้วโดยอัตโนมัติ** ("SecretsUsedInArgOrEnv") — นี่ไม่ใช่แค่คำแนะนำเชิงทฤษฎี แต่
Docker's build linter ตรวจจับ pattern นี้และเตือนจริงตอน build แต่หลายทีมยังมองข้าม warning นี้ไปเพราะ build
"สำเร็จ" ตามปกติ — มาดูว่าทำไมมันสำคัญ

**ขั้นที่ 1: ตรวจสอบ final container filesystem — ดูเหมือนปลอดภัย (ไฟล์ไม่อยู่แล้ว)**:

```bash
$ docker run --rm secret-leak-demo:latest cat /app-secret.env
cat: can't open '/app-secret.env': No such file or directory
```

ถ้าดูแค่นี้ อาจสรุปผิดว่า "ไฟล์ถูกลบไปแล้ว ปลอดภัยดี" — แต่ **`rm` ใน layer ที่ 4 แค่เพิ่ม diff ใหม่ที่บอกว่า
"ไฟล์นี้หายไปในมุมมองของ layer นี้" มันไม่ได้ไปแก้ไข/ลบเนื้อหาของ layer ที่ 2 ที่สร้างไฟล์นี้ขึ้นมาเลยแม้แต่
นิดเดียว** — layer ที่ 2 (ที่มีไฟล์ `/app-secret.env` พร้อม secret เต็ม ๆ) ยังถูกเก็บอยู่ใน image จริง ๆ

**ขั้นที่ 2: `docker history --no-trunc` — เห็น secret ตรง ๆ ใน build metadata**:

```bash
$ docker history --no-trunc secret-leak-demo:latest
IMAGE          CREATED BY                                                                                     SIZE
<missing>      RUN |1 DATABASE_PASSWORD=SuperSecretPass123 /bin/sh -c rm /app-secret.env # buildkit           4.1kB
<missing>      RUN |1 DATABASE_PASSWORD=SuperSecretPass123 /bin/sh -c cat /app-secret.env # buildkit           4.1kB
<missing>      RUN |1 DATABASE_PASSWORD=SuperSecretPass123 /bin/sh -c echo "DATABASE_PASSWORD=${DATABASE_PASSWORD}" > /app-secret.env # buildkit   8.19kB
<missing>      ARG DATABASE_PASSWORD=SuperSecretPass123                                                       0B
```

**secret ปรากฏเป็น plaintext ตรง ๆ ในทุกบรรทัดของ `docker history`** — ใครก็ตามที่มีสิทธิ์ `docker pull`
image นี้ (หรือแค่เห็น registry ที่เก็บมันไว้) รัน `docker history` เพียงคำสั่งเดียวก็เห็น secret ทั้งหมดทันที
โดยไม่ต้องทำอะไรซับซ้อนเลย

**ขั้นที่ 3: กู้ไฟล์จริงกลับมาจาก layer ที่ "ลบ" ไปแล้ว** (พิสูจน์ให้ลึกกว่า metadata — ดึงเนื้อหาไฟล์จริงจาก
layer blob):

```bash
$ docker save secret-leak-demo:latest -o image.tar
$ mkdir extracted && cd extracted && tar xf ../image.tar

$ for f in $(find blobs -type f); do
    tar tf "$f" 2>/dev/null | grep -q "app-secret.env" && echo "พบใน: $f" && tar xOf "$f" app-secret.env
  done
พบใน: blobs/sha256/e2eecbd93efa89b88b0135befbb14cdbc8432d660df61c8b47d7ec82da3b0de6
DATABASE_PASSWORD=SuperSecretPass123
```

**กู้เนื้อหาไฟล์ `app-secret.env` ที่ "ถูกลบไปแล้ว" กลับมาได้ 100% ตรงเป๊ะ** — นี่คือหลักฐานที่ชัดที่สุดว่า
"ลบไฟล์ใน layer ถัดไป" **ไม่ใช่การลบข้อมูลที่ปลอดภัย** เพราะ layer เดิมที่มีข้อมูลนั้นยังอยู่ใน image เต็ม ๆ ไม่
ว่า layer ที่ตามมาจะบอกว่าไฟล์นั้น "ไม่อยู่แล้ว" ในมุมมองสุดท้ายก็ตาม (union filesystem แค่ซ่อนมันจากมุมมองที่
merge แล้ว ไม่ได้ทำลายข้อมูลจริง)

**วิธีที่ถูกต้อง — สอดคล้องกับหลักการที่บทนี้สอนมาตลอด**:

1. **secret ต้องมาจาก environment variable ตอน `docker run`/`docker-compose` เท่านั้น ไม่ใช่ตอน `docker
   build`** — `-e DATABASE_URL=...`/`-e JWT_SECRET=...` (หรือ `environment:` ใน compose แบบหัวข้อ 96.9) ค่า
   เหล่านี้อยู่แค่ใน memory ของ container ที่รันอยู่ ไม่ถูก bake เข้า image เลยแม้แต่ byte เดียว — image
   เดียวกันเป๊ะ deploy ได้ทุก environment โดยแค่เปลี่ยนค่า env ตอน run (ตรงกับหลักการ twelve-factor app ที่
   อธิบายไว้ต้นหัวข้อ)
2. **multi-stage build ช่วยเสริมอีกชั้น** — ถ้ามีขั้นตอนที่ต้อง "เห็น" secret ชั่วคราวตอน build จริง ๆ (เช่น
   private registry credential สำหรับ `cargo` ที่ต้องดึง private crate) ควรใช้ **BuildKit secret mount**
   (`RUN --mount=type=secret,id=mytoken ...`) ที่ออกแบบมาเฉพาะสำหรับกรณีนี้ — ค่าจาก secret mount นี้ **ไม่ถูก
   เก็บใน layer เลยแม้แต่นิดเดียว** (ต่างจาก `ARG`/`ENV` โดยสิ้นเชิง) เพราะ BuildKit จัดการให้ secret มาถึงแค่
   ตอน `RUN` นั้นทำงานอยู่ ไม่ persist ลง layer diff — นี่คือ feature ของ BuildKit โดยเฉพาะ (Docker เวอร์ชัน
   ปัจจุบันใช้ BuildKit เป็น builder default อยู่แล้ว)
3. **runtime stage ของ multi-stage build ไม่ควรมี `ARG`/`ENV` ของ secret ค้างอยู่เลย** — สังเกตว่า Dockerfile
   ของหัวข้อ 96.4-96.7 ทั้งหมดไม่มี `ARG`/`ENV` ที่เป็น secret เลยแม้แต่ตัวเดียว มีแค่ `ENV BIND_ADDR=...` ที่
   ไม่ใช่ความลับ — ทุก secret ที่แอปต้องใช้จริง (`DATABASE_URL`, `JWT_SECRET`) จะถูก inject ตอน `docker run`/
   `docker-compose up` เท่านั้น ตามที่หัวข้อ 96.9-96.10 จะแสดงให้เห็นจริง

**`.env` ไฟล์และ `--env-file` — ความสะดวกตอน dev ที่ต้องระวังไม่ให้หลุดเข้า git/image**: การพิมพ์
`-e DATABASE_URL=... -e JWT_SECRET=...` ยาว ๆ ทุกครั้งที่ `docker run` นั้นน่าเบื่อ Docker (และ Docker Compose)
รองรับการอ่านค่าจากไฟล์ `.env` ให้อัตโนมัติแทน:

```bash
# .env (อยู่ระดับเดียวกับ docker-compose.yml — Compose อ่านไฟล์นี้ให้อัตโนมัติโดยไม่ต้องตั้งค่าเพิ่ม)
DATABASE_URL=postgres://postgres:postgres@db:5432/library_mini
JWT_SECRET=dev-only-secret-do-not-use-in-prod
```

```bash
# หรือระบุไฟล์ตรง ๆ กับ docker run
$ docker run --env-file .env library-api-mini-musl:latest
```

**กฎเดียวกับที่ Part 66/70/74 สอนไว้เรื่อง `.env` ของแอป Rust เองใช้ได้ตรงกันเป๊ะที่นี่**: `.env` ไฟล์นี้
**ต้องอยู่ใน `.gitignore` เสมอ** (ไม่ commit เข้า git — ต่างจาก `.env.example` ที่ commit ได้เพราะไม่มีค่าจริง)
และ **ต้องอยู่ใน `.dockerignore` ด้วย** (กันไม่ให้ `COPY . .` ตอน build ดึงมันเข้า image โดยไม่ตั้งใจ — ถ้าเผลอ
`COPY` เข้าไปจะเจอปัญหาเดียวกับหัวข้อที่แล้วทั้งหมด คือ secret ถูก bake เข้า layer แม้จะไม่ได้ตั้งใจก็ตาม) —
`.env` เหมาะกับความสะดวกตอน dev บนเครื่องตัวเอง ส่วน production ควรมาจาก secret manager ของแพลตฟอร์ม deploy
โดยตรง (Kubernetes Secret, AWS Secrets Manager, Docker Swarm secret ฯลฯ) ไม่ใช่ไฟล์ `.env` ที่วางไว้บนเครื่อง
เซิร์ฟเวอร์ตรง ๆ

### 96.9 Docker Compose: รัน App + PostgreSQL จริง + Redis จริงด้วยคำสั่งเดียว

ตอน dev บนเครื่องคนเดียว การรัน `docker run` แยกทีละ container (แอป, PostgreSQL, Redis) แล้วต่อ network เอง
ทำได้แต่ยุ่งยากและลืมง่าย — `docker-compose.yml` แก้ปัญหานี้โดยประกาศทุก service ที่ต้องรันพร้อมกันไว้ในไฟล์
เดียว แล้วสั่ง `docker compose up` ครั้งเดียวได้ทั้งระบบ

**โปรเจกต์ทดสอบ**: `presence-api` — axum service ที่มี `/health` เช็คทั้ง PostgreSQL (ต่อยอด Part 70) และ
Redis (ต่อยอด Part 83) จริง ๆ (ไม่ใช่แค่ตอบ `200` เสมอแบบ mock):

```rust
// src/main.rs (ส่วนสำคัญ)
async fn health(State(state): State<Arc<AppState>>) -> (StatusCode, Json<Value>) {
    let db_ok = sqlx::query_scalar::<_, i32>("SELECT 1")
        .fetch_one(&state.db)
        .await
        .is_ok();

    let redis_ok = match state.redis.get_multiplexed_async_connection().await {
        Ok(mut conn) => conn.ping::<String>().await.is_ok(),
        Err(_) => false,
    };

    let status = if db_ok && redis_ok { "ok" } else { "error" };
    let code = if db_ok && redis_ok { StatusCode::OK } else { StatusCode::SERVICE_UNAVAILABLE };

    (code, Json(json!({
        "status": status,
        "database": if db_ok { "ok" } else { "unreachable" },
        "redis": if redis_ok { "ok" } else { "unreachable" },
    })))
}
```

image ของ `presence-api` build ด้วยเทคนิคแยก manifest จากหัวข้อ 96.6 (ไม่แสดงซ้ำเพราะโครงสร้างเหมือนเดิมทุก
ประการ ต่างแค่ dependency ที่เพิ่ม `sqlx`/`redis` เข้ามา)

**`docker-compose.yml`**:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: presence
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      timeout: 2s
      retries: 15

  cache:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 2s
      timeout: 2s
      retries: 15

  app:
    image: presence-api:latest
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/presence
      REDIS_URL: redis://cache:6379
    ports:
      - "18090:8080"
```

**อธิบายจุดสำคัญ**:

- **`DATABASE_URL`/`REDIS_URL` ใช้ hostname `db`/`cache` (ไม่ใช่ `127.0.0.1`)** — Docker Compose สร้าง network
  ภายในให้อัตโนมัติที่ทุก service คุยกันผ่าน **ชื่อ service เป็น DNS name ได้เลย** (`app` resolve `db` เป็น IP
  ของ container `db` ให้อัตโนมัติผ่าน internal DNS ของ Docker) — นี่คือสิ่งที่ทำให้ `environment:` ของหัวข้อ
  96.8 ทำงานได้จริง (secret/config ทั้งหมดยัง inject ผ่าน environment variable ตอน compose up เท่านั้น ไม่มี
  อะไร bake เข้า image เลย)
- **`depends_on` พร้อม `condition: service_healthy`** — ทำให้ `app` container **รอ** จนกว่า `db`/`cache` จะ
  ผ่าน `healthcheck` ก่อนเริ่มทำงาน ป้องกันปัญหา `app` พยายามต่อ database/Redis ที่ยังไม่พร้อมรับ connection
  (image ต้องใช้เวลาเริ่มต้นตัวเองเล็กน้อยก่อนพร้อมรับ connection จริง แม้ container จะ "start" แล้วก็ตาม —
  ปัญหาคลาสสิกที่ `depends_on` แบบไม่มี condition แก้ไม่ได้ เพราะมันแค่รอให้ container "start" ไม่ใช่ "พร้อมใช้
  งานจริง")

**รันจริง**:

```bash
$ docker compose up -d
 Container presence-api-cache-1 Starting
 Container presence-api-db-1 Starting
 Container presence-api-db-1 Started
 Container presence-api-cache-1 Started
 Container presence-api-cache-1 Waiting
 Container presence-api-db-1 Waiting
 Container presence-api-cache-1 Healthy
 Container presence-api-db-1 Healthy
 Container presence-api-app-1 Starting
 Container presence-api-app-1 Started

$ docker compose ps
NAME                   IMAGE                 SERVICE   STATUS                   PORTS
presence-api-app-1     presence-api:latest   app       Up 3 seconds             0.0.0.0:18090->8080/tcp
presence-api-cache-1   redis:7-alpine        cache     Up 5 seconds (healthy)   6379/tcp
presence-api-db-1      postgres:16           db        Up 5 seconds (healthy)   5432/tcp
```

สังเกตว่า `cache`/`db` ขึ้นสถานะ **`(healthy)`** ก่อนที่ `app` จะเริ่ม start เลย ตรงกับที่ `depends_on:
condition: service_healthy` กำหนดไว้ — พิสูจน์ผลลัพธ์จริงด้วย `curl`:

```bash
$ curl -sS -i http://127.0.0.1:18090/health
HTTP/1.1 200 OK
content-type: application/json
content-length: 44

{"database":"ok","redis":"ok","status":"ok"}

$ docker logs presence-api-app-1
presence-api ฟังอยู่ที่ 0.0.0.0:8080
```

**`{"database":"ok","redis":"ok","status":"ok"}` — health check เชื่อมต่อทั้ง PostgreSQL 16 จริงและ Redis 7
จริงสำเร็จทั้งคู่** ผ่าน container ที่สร้างขึ้นใหม่ทั้งหมดจากคำสั่งเดียวคือ `docker compose up -d` — ปิดงานด้วย
`docker compose down -v` ที่ลบทั้ง container และ network/volume ที่สร้างไว้ทั้งหมดในคำสั่งเดียวเช่นกัน:

```bash
$ docker compose down -v
 Container presence-api-db-1 Removed
 Container presence-api-cache-1 Removed
 Network presence-api_default Removed
```

**สิ่งที่เกิดขึ้นเบื้องหลัง `docker compose up` — network และ volume ที่ถูกสร้างอัตโนมัติ**: สังเกตข้อความ
`Network presence-api_default Removed` ในผลลัพธ์ข้างบน — Compose สร้าง **network แบบ bridge ของตัวเอง** ให้
ทุกครั้งที่ `up` (ชื่อ default คือ `<ชื่อโปรเจกต์>_default` โดย "ชื่อโปรเจกต์" มาจากชื่อโฟลเดอร์ที่มี
`docker-compose.yml` อยู่ ถ้าไม่ตั้งชื่อเอง) — นี่คือสิ่งที่ทำให้ hostname อย่าง `db`/`cache` resolve กันได้
ตามที่อธิบายไว้ข้างต้น: **container ทุกตัวที่ประกาศใน `services:` เดียวกันจะถูกใส่เข้า network นี้โดยอัตโนมัติ**
ต่างจาก `docker run` เดี่ยว ๆ ที่ default จะอยู่บน `bridge` network มาตรฐานของ Docker เองที่**ไม่มี** DNS
resolution ข้าม container ให้ (ต้องสร้าง custom network ด้วยมือผ่าน `docker network create` ถ้าจะทำ `docker
run` หลายตัวให้คุยกันด้วยชื่อ) — Compose จัดการส่วนนี้ให้อัตโนมัติ **นี่คือเหตุผลหนึ่งที่ compose สะดวกกว่า
`docker run` แยกทีละตัวมากสำหรับ multi-service setup**

**เรื่อง volume ที่ `-v` ในคำสั่ง `down -v` ลบ**: สำหรับ `presence-api` ในหัวข้อนี้ไม่มี volume ที่ตั้งชื่อไว้
(`volumes:` ระดับบนสุดของไฟล์ไม่มีเลย) ต่างจาก `library_api_mini` ของหัวข้อ 96.10 ที่มี `library_mini_pgdata`
เพื่อให้ข้อมูล PostgreSQL อยู่รอดข้าม `docker compose down`/`up` ธรรมดา (ไม่มี `-v`) — การไม่ใส่ `-v` ตอน
`down` จะทำให้ volume ที่ตั้งชื่อไว้ **ยังอยู่** พร้อมข้อมูลเดิมสำหรับ `up` ครั้งต่อไป (มีประโยชน์ตอน dev ที่
อยากเก็บข้อมูลทดสอบไว้ข้าม session) ส่วน `-v` บอกให้ลบ volume ที่ตั้งชื่อไว้ทิ้งไปด้วย (ใช้ตอนอยากเริ่มจาก
database เปล่าสนิททุกครั้ง แบบที่บทนี้ตั้งใจทำตลอดเพื่อพิสูจน์ว่า migration ทำงานถูกต้องกับข้อมูลใหม่จริง ๆ)

### 96.10 Capstone: Containerize แอป Part 92-94 อย่างถูกต้องด้วยทุกเทคนิคในบทนี้

ถึงเวลานำทุกเทคนิคของบทนี้มารวมกัน — Part 94 หัวข้อ 94.5 ได้ทำ multi-stage Dockerfile สำหรับ backend+frontend
รวมกันไว้แล้ว และพิสูจน์ด้วย end-to-end test เต็มรูปแบบว่า deploy ได้จริง (headless Chromium ครบ flow สมัคร →
login → ยืม → คืน) — สิ่งที่หัวข้อนี้จะทำเพิ่มคือ **นำเทคนิคใหม่ของบทนี้ที่ Part 94 ยังไม่ได้ทำ** (musl static
linking, dependency caching อย่างเป็นระบบ) มาประยุกต์กับ **dependency set จริงเดียวกันกับ backend ของ Part
92-94** เพื่อพิสูจน์ข้อกล่าวอ้างที่ Part 94 หัวข้อ 94.5 ทิ้งไว้ให้บทนี้พิสูจน์ต่อ: *"binary ไม่ผูกกับ `libssl`
เลย เพราะทุก crate ที่ใช้ crypto ในบทนี้ใช้ pure-Rust implementation ทั้งหมด"* — ถ้าข้อกล่าวอ้างนี้จริง มันควร
หมายความว่า **musl static linking ต้องทำได้จริงกับ dependency set นี้โดยไม่ต้องแก้โค้ดอะไรเลย**

**ขอบเขตของหัวข้อนี้ — พูดตรง ๆ ก่อนเริ่ม**: หัวข้อนี้สร้างใหม่เฉพาะส่วน **backend** ที่จำลอง Cargo.toml
dependency ชุดจริงของ Part 92 (`axum`, `tokio`, `tower-http`, `serde`, `sqlx` แบบ `postgres`+`migrate`,
`tracing`, `jsonwebtoken` แบบ `rust_crypto`, `argon2`, `chrono` — ตรงตัวกับ Part 92 หัวข้อที่ตั้ง Cargo.toml
ไว้) พร้อม endpoint หลักสามตัว (`/health`, `POST /api/v1/auth/register`, `POST /api/v1/auth/login`) ที่ใช้
`argon2`/`jsonwebtoken` จริงตาม logic เดียวกับ Part 92 หัวข้อที่สอน `hash_password`/`register`/`login` — **ไม่
ได้คัดลอกทั้ง 10 endpoint หรือรวม frontend เข้ามาด้วย** เพราะ Part 94 หัวข้อ 94.9 พิสูจน์ครบไปแล้วว่า full-stack
combo ทั้งก้อน build/run/e2e ผ่านจริง 100% — สิ่งที่ยังไม่ถูกพิสูจน์คือเทคนิคใหม่ของบทนี้เองกับ dependency set
เดียวกัน ซึ่งคือสิ่งที่หัวข้อนี้พิสูจน์

**migration** (ตรงตัวกับ Part 92 หัวข้อที่สร้างตาราง `users`):

```sql
-- migrations/20260101000000_create_users.up.sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    username      TEXT NOT NULL UNIQUE,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('member', 'admin')),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_username ON users (username);
```

**`Cargo.toml`** (dependency set ตรงกับ Part 92 — ตัดแค่ `utoipa`/`utoipa-swagger-ui` ออกเพราะไม่เกี่ยวกับ
จุดที่หัวข้อนี้ต้องพิสูจน์):

```toml
[dependencies]
axum = { version = "0.8.9", features = ["macros"] }
tokio = { version = "1.53.1", features = ["full"] }
tower-http = { version = "0.7.1", features = ["trace"] }
serde = { version = "1.0.229", features = ["derive"] }
serde_json = "1.0.151"
sqlx = { version = "0.9.0", features = ["runtime-tokio", "postgres", "chrono", "migrate"] }
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter"] }
jsonwebtoken = { version = "11.1.0", features = ["rust_crypto"] }
argon2 = "0.6.0"
chrono = { version = "0.4.45", features = ["serde"] }
```

**`src/main.rs`** (ส่วนสำคัญที่ใช้ argon2/jsonwebtoken ตรงตาม logic ของ Part 92 หัวข้อ `hash_password`/
`register`/`login` — ต่างจาก Part 92 แค่จุดที่ query ผ่าน `sqlx::query_as` แบบ runtime-checked แทน macro
`query_as!` แบบ compile-time เพื่อไม่ต้องพึ่ง PostgreSQL จริงตอน `docker build` เอง ซึ่งเป็นรายละเอียดของ Part
70/94 ที่พิสูจน์แยกไปแล้วผ่าน SQLx offline mode — ไม่ใช่จุดโฟกัสของบทนี้):

```rust
use argon2::{
    password_hash::{phc::PasswordHash, PasswordHasher, PasswordVerifier},
    Argon2,
};
use jsonwebtoken::{encode, EncodingKey, Header};

fn hash_password(password: &str) -> Result<String, argon2::password_hash::Error> {
    let argon2 = Argon2::default();
    Ok(argon2.hash_password(password.as_bytes())?.to_string())
}

fn verify_password(password: &str, hash: &str) -> bool {
    let parsed_hash = match PasswordHash::new(hash) {
        Ok(h) => h,
        Err(_) => return false,
    };
    Argon2::default()
        .verify_password(password.as_bytes(), &parsed_hash)
        .is_ok()
}

async fn health(State(state): State<Arc<AppState>>) -> (StatusCode, Json<Value>) {
    match sqlx::query_scalar::<_, i32>("SELECT 1").fetch_one(&state.db).await {
        Ok(_) => (StatusCode::OK, Json(json!({"status": "ok", "database": "ok"}))),
        Err(e) => {
            tracing::error!(error = %e, "health check: database ไม่พร้อม");
            (StatusCode::SERVICE_UNAVAILABLE, Json(json!({"status": "error", "database": "unreachable"})))
        }
    }
}
```

**ขั้นที่ 1 — multi-stage + dependency caching (debian runtime, ตาม 96.4+96.6)**:

```dockerfile
FROM rust:1-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src
COPY src ./src
COPY migrations ./migrations
RUN touch src/main.rs && cargo build --release

FROM debian:bookworm-slim AS runtime
RUN useradd --system --create-home --uid 10001 appuser
WORKDIR /app
COPY --from=builder /app/target/release/library_api_mini ./library_api_mini
COPY --from=builder /app/migrations ./migrations
ENV BIND_ADDR=0.0.0.0:8080
USER appuser
EXPOSE 8080
ENTRYPOINT ["/app/library_api_mini"]
```

```bash
$ docker build -t library-api-mini:latest .
real	1m16.432s

$ docker images library-api-mini:latest
IMAGE                     ID             DISK USAGE   CONTENT SIZE   EXTRA
library-api-mini:latest   f9e9631af68b        121MB         30.6MB
```

**ตรวจสอบ `ldd` เพื่อยืนยันข้อกล่าวอ้างของ Part 94 ก่อนลองทำ musl build**:

```bash
$ ldd library_api_mini_glibc
	linux-vdso.so.1 (0x00007f260d88d000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f260d84e000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f260d765000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f260ce00000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f260d88f000)
```

**ไม่มี `libssl.so` อยู่ในรายการเลย** แม้ dependency set นี้จะมี `jsonwebtoken`/`argon2`/`sqlx` เข้ามาเต็มรูปแบบ
— ยืนยันข้อกล่าวอ้างของ Part 94 ด้วยหลักฐานจริง (เหมือนกันเป๊ะกับ binary ของ `greet-service` ในหัวข้อ 96.1/96.4
ที่ไม่มี dependency ที่ไม่มาตรฐานอะไรเลย) เพราะทั้ง `jsonwebtoken` feature `rust_crypto` และ `argon2` เป็น
pure-Rust implementation จริงตามที่ Part 92/94 เลือกไว้

**ขั้นที่ 2 — musl static linking (ตาม 96.5) กับ dependency set เดียวกันเป๊ะ**:

```dockerfile
FROM rust:1-alpine AS builder
WORKDIR /app
RUN rustup target add x86_64-unknown-linux-musl
COPY Cargo.toml Cargo.lock ./
RUN mkdir -p src && echo "fn main() {}" > src/main.rs
RUN cargo build --release --target x86_64-unknown-linux-musl
RUN rm -rf src
COPY src ./src
COPY migrations ./migrations
RUN touch src/main.rs && cargo build --release --target x86_64-unknown-linux-musl

FROM scratch AS runtime
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/library_api_mini /library_api_mini
COPY --from=builder /app/migrations /migrations
ENV BIND_ADDR=0.0.0.0:8080
EXPOSE 8080
ENTRYPOINT ["/library_api_mini"]
```

```bash
$ docker build -f Dockerfile.musl -t library-api-mini-musl:latest .
#10 129.3    Compiling sqlx v0.9.0
#10 136.3    Compiling jsonwebtoken v11.1.0
#10 137.0    Compiling tower-http v0.7.1
#10 151.8    Compiling library_api_mini v0.1.0 (/app)
#10 152.0     Finished `release` profile [optimized] target(s) in 2m 31s
#14 [builder 10/10] RUN touch src/main.rs && cargo build --release --target x86_64-unknown-linux-musl
#14 0.318    Compiling library_api_mini v0.1.0 (/app)
#14 19.40     Finished `release` profile [optimized] target(s) in 19.32s

real	2m53.884s
```

**build ผ่านสำเร็จ 100% โดยไม่ต้องแก้โค้ดหรือ dependency แม้แต่บรรทัดเดียว** — dependency set เต็มรูปแบบของ
Part 92 (รวม `sqlx-macros`, `rsa`, `ed25519-dalek` ที่เป็น transitive dependency ของ `jsonwebtoken`) compile
ผ่านสำหรับ `x86_64-unknown-linux-musl` target ทั้งหมด ยืนยันว่าการเลือก `rust_crypto`/`argon2` (pure-Rust
crypto) มาตั้งแต่ Part 74/92 **ทำให้ static linking เป็นไปได้จริงโดยไม่เสียอะไรเลย** วัดขนาด image:

```bash
$ docker images library-api-mini-musl:latest
IMAGE                          ID             DISK USAGE   CONTENT SIZE   EXTRA
library-api-mini-musl:latest   3d1a062fba76       7.84MB          2.42MB

$ file library_api_mini_musl_binary
library_api_mini_musl_binary: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
static-pie linked, ..., stripped
	statically linked
```

**จาก 121MB (multi-stage+debian) เหลือ 7.84MB (musl+scratch) — เล็กลงกว่า 15 เท่า** สำหรับแอปที่มี auth
เต็มรูปแบบ (JWT + argon2 password hashing) ต่อ PostgreSQL จริง ไม่ใช่แค่ hello-world เปล่า ๆ แบบหัวข้อ 96.5

**ขั้นที่ 3 — `docker-compose` กับ PostgreSQL จริง (ตาม 96.9) รัน image musl/scratch ตัวนี้ตรง ๆ**:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: library_mini
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      timeout: 2s
      retries: 15
    volumes:
      - library_mini_pgdata:/var/lib/postgresql/data

  app:
    image: library-api-mini-musl:latest
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/library_mini
      JWT_SECRET: compose-capstone-secret-change-me
      BIND_ADDR: 0.0.0.0:8080
      RUST_LOG: info
    ports:
      - "18099:8080"

volumes:
  library_mini_pgdata:
```

**รันจริงและพิสูจน์ full flow (migration → health → register → login) กับ PostgreSQL ที่สร้างขึ้นใหม่ล้วน ๆ**:

```bash
$ docker compose up -d
 Container library_api_mini-db-1 Healthy
 Container library_api_mini-app-1 Started

$ docker logs library_api_mini-app-1
รัน migration สำเร็จ
library_api_mini ฟังอยู่ที่ 0.0.0.0:8080

$ curl -sS -i http://127.0.0.1:18099/health
HTTP/1.1 200 OK
content-type: application/json

{"database":"ok","status":"ok"}

$ curl -sS -X POST http://127.0.0.1:18099/api/v1/auth/register -H 'Content-Type: application/json' \
    -d '{"username":"musluser","email":"musluser@example.com","password":"password123"}'
{"email":"musluser@example.com","id":1,"username":"musluser"}

$ curl -sS -X POST http://127.0.0.1:18099/api/v1/auth/login -H 'Content-Type: application/json' \
    -d '{"username":"musluser","password":"password123"}'
{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...","token_type":"Bearer","user":{"id":1,"username":"musluser"}}
```

**ทุกขั้นตอนสำเร็จจริง 100%**: migration รันอัตโนมัติตอน container start (embed ผ่าน `sqlx::migrate!` ตาม
Part 71), health check ยืนยันต่อ PostgreSQL จริงสำเร็จ, register สร้าง user จริงพร้อม hash password ด้วย
argon2, login verify password แล้วออก JWT จริงด้วย `jsonwebtoken` — ทั้งหมดนี้รันจาก **image ขนาด 7.84MB
เพียงตัวเดียวที่ไม่มี OS ใด ๆ อยู่ข้างในเลย (`scratch`)** ต่อกับ PostgreSQL 16 จริงผ่าน `docker-compose`

**สรุปสิ่งที่ capstone นี้พิสูจน์**: ทุกเทคนิคของบทนี้ (multi-stage build, dependency caching ผ่านการแยก
manifest, musl static linking, environment-variable config, docker-compose กับ real database) **ใช้งานร่วม
กันได้จริงกับ dependency set เดียวกับ Part 92** โดยไม่ต้องแก้ไข business logic แม้แต่บรรทัดเดียว — สิ่งที่ต้อง
ปรับมีแค่ **Dockerfile และวิธี deploy** ซึ่งเป็นสิ่งที่ดีจริง ๆ เพราะหมายความว่าทีมสามารถปรับปรุง
containerization strategy (เช่น ย้ายจาก glibc ไป musl ตอนไหนก็ได้ในอนาคต) โดยไม่ต้องแตะ business logic ของแอป
เลย — ปิดท้ายด้วยการ cleanup ทุกอย่างให้เรียบร้อยเหมือนเดิม:

```bash
$ docker compose down -v
$ docker rmi library-api-mini-musl:latest library-api-mini:latest
```

### 96.11 เส้นทางจาก Image บนเครื่องสู่ Production จริง: Tagging และการเตรียมพร้อมสำหรับ Part 97

image ทุกตัวที่บทนี้ build มาทั้งหมดอยู่แค่บนเครื่องเดียว (`docker images` เห็นได้แค่ในเครื่องนั้น) — ก่อนจะ
deploy จริงต้อง **push image ขึ้น container registry** (Docker Hub, GitHub Container Registry, AWS ECR
ฯลฯ) ที่ production server ดึงมาใช้ได้ — หัวข้อนี้ปิดท้ายด้วยเรื่องพื้นฐานที่ Part 97 (CI/CD ด้วย GitHub
Actions) จะใช้ต่อทันที: **การตั้งชื่อ/tag image ให้ถูกต้อง**

**โครงสร้างชื่อ image เต็มรูปแบบ**: `<registry>/<namespace>/<repository>:<tag>` — เช่น
`ghcr.io/myorg/library-api:v1.2.0` (`ghcr.io` คือ GitHub Container Registry, `myorg` คือ namespace/organization,
`library-api` คือชื่อ repository ของ image, `v1.2.0` คือ tag) — ถ้าไม่ระบุ registry เลย (แบบทุกตัวอย่างในบทนี้
ที่ใช้ชื่อสั้น ๆ อย่าง `greet-multistage:latest`) Docker จะสมมติว่าเป็น Docker Hub โดย default

**กฎสำคัญที่สุดเรื่อง tag: อย่าพึ่ง `latest` เป็น mechanism หลักในการ deploy production** — `latest` เป็นแค่
ชื่อ tag ธรรมดา (ไม่มีความหมายพิเศษทาง technical ใด ๆ นอกจากเป็นชื่อ default ที่ Docker ใช้เมื่อไม่ระบุ tag)
แต่ปัญหาคือ **`latest` เป็น mutable tag** — ถ้า build image ใหม่แล้ว push ทับ `latest` เดิม ไม่มีทางรู้จาก
ชื่อ tag อย่างเดียวว่า image ที่รันอยู่บน production ตอนนี้คือ commit ไหนกันแน่ (ต่างจาก tag ที่ผูกกับ
เวอร์ชัน/commit ที่ตายตัว) ทำให้ **rollback ยากมาก** (ไม่รู้ว่า "เวอร์ชันก่อนหน้า" คือ image ไหน) และทำให้
สอง environment ที่ต่างเวลากัน pull "latest" ได้ image คนละตัวกันโดยไม่รู้ตัว — แนวทางที่ใช้จริงในทีม
production คือ tag ด้วย **ค่าที่ระบุตัวตนได้แน่นอน** เช่น:

- **Git commit SHA** — `myapp:a1b2c3d` (สั้น, immutable แน่นอน, ตรวจสอบย้อนกลับไปยัง commit ที่แน่ชัดได้เสมอ —
  Part 97 จะใช้ `${{ github.sha }}` ของ GitHub Actions สร้าง tag แบบนี้อัตโนมัติทุก build)
- **Semantic version** — `myapp:v1.2.0` (สำหรับ release ที่ตั้งใจ tag เป็นเวอร์ชันที่ผู้ใช้มองเห็น ตาม
  semantic versioning ที่ Part 17 หัวข้อ publish เกริ่นไว้)
- **`latest` ยังมีประโยชน์อยู่ — แค่ไม่ใช่ mechanism หลัก**: ใช้เป็น "ป้ายชี้" ไปยัง tag ที่เสถียรล่าสุดสำหรับ
  คนที่แค่อยาก `docker pull myapp` แบบไม่ต้องรู้เวอร์ชันเป๊ะ (เช่นคนที่มาลองรันครั้งแรก) แต่ deployment
  pipeline จริงควร deploy ด้วย tag ที่ immutable เสมอ ไม่ใช่ `latest`

**`docker tag`/`docker push` — คำสั่งพื้นฐานที่ Part 97 จะเรียกอัตโนมัติ**:

```dockerfile
# ตั้ง tag เพิ่มให้ image ที่ build ไว้แล้ว (ไม่ build ใหม่ แค่ตั้งชื่ออื่นให้ image ID เดียวกัน)
docker tag library-api-mini:latest ghcr.io/myorg/library-api:a1b2c3d
docker tag library-api-mini:latest ghcr.io/myorg/library-api:latest

# login เข้า registry ก่อน push เสมอ (credential ต้อง inject ผ่าน secret ของ CI ไม่ hardcode ตามหลักการ
# ของหัวข้อ 96.8)
docker login ghcr.io -u <username> --password-stdin

# push ทั้งสอง tag ขึ้น registry
docker push ghcr.io/myorg/library-api:a1b2c3d
docker push ghcr.io/myorg/library-api:latest
```

**Multi-platform build — ประเด็นที่ Rust ต้องระวังเป็นพิเศษเทียบกับภาษา interpreted**: ถ้าทีมมีทั้งเครื่อง
`x86_64` (Intel/AMD, เซิร์ฟเวอร์ cloud ส่วนใหญ่) และ `arm64` (Apple Silicon ตอน dev, หรือ AWS Graviton ตอน
production) **image ที่ build บนเครื่องหนึ่งจะรันไม่ได้บนอีก architecture หนึ่งเลย** เพราะ binary ที่ compile
แล้วผูกกับ CPU architecture ตรง ๆ (ต่างจากภาษาที่ interpret ตอนรันที่ image เดียวใช้ได้ทุก architecture เพราะ
interpreter จัดการความต่างให้) — `docker buildx build --platform linux/amd64,linux/arm64` แก้ปัญหานี้ด้วยการ
build binary สอง architecture แล้ว push เข้า **manifest list เดียว** ที่ Docker เลือก architecture ที่ตรงกับ
เครื่องที่ `docker pull` เองอัตโนมัติ — สำหรับ Rust หมายความว่า builder stage ต้อง cross-compile จริง (เพิ่ม
`rustup target add aarch64-unknown-linux-gnu` เป็นต้น) ซึ่งเป็นหัวข้อที่ลึกกว่าที่บทนี้จะครอบคลุม (Part 97 หรือ
เนื้อหา cross-compilation ในโมดูลถัดไปจะพูดถึงรายละเอียดนี้เพิ่ม) — ระบุไว้ตรงนี้เพื่อให้รู้ว่ามันมีอยู่ ไม่ใช่
สิ่งที่มองข้ามได้ถ้าทีมมีเครื่อง dev/production ต่าง architecture กัน

ทั้งหมดนี้คือจุดที่บทนี้ "ส่งต่อ" ให้ Part 97 — image ที่ build/tag ถูกต้องตามหลักการข้างบน คือสิ่งที่ CI/CD
pipeline จะ build/tag/push ให้อัตโนมัติทุกครั้งที่มีคน push commit เข้า repository โดยไม่ต้องมีใครรัน
`docker build`/`docker push` ด้วยมือเองอีกต่อไป

## กับดักที่พบบ่อย (Common Pitfalls)

**1. Bind ที่ `127.0.0.1` แทน `0.0.0.0` — ปัญหาคลาสสิกที่สุดของ Rust web service ใน container**

โค้ดที่ทำงานปกติตอน dev บนเครื่องตัวเอง (`bind("127.0.0.1:8080")`) จะ **รันได้ปกติภายใน container แต่ต่อจาก
host ไม่ได้เลย** แม้จะ `docker run -p 8080:8080` ถูกต้องแล้วก็ตาม — พิสูจน์ด้วยการทดลองจริง (ใช้ HTTP server
เปล่า ๆ เพื่อแยกปัญหาออกจากความซับซ้อนของ axum):

```bash
$ docker run -d -p 19999:8000 python:3-alpine python3 -m http.server 8000 --bind 127.0.0.1
$ curl -m 3 http://127.0.0.1:19999/
curl: (56) Recv failure: Connection reset by peer
```

```bash
$ docker run -d -p 19999:8000 python:3-alpine python3 -m http.server 8000 --bind 0.0.0.0
$ curl -m 3 -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1:19999/
HTTP 200
```

**เหตุผล**: `127.0.0.1` คือ loopback address ที่หมายถึง "เครื่องนี้เอง" — ใน container, "เครื่องนี้เอง" คือ
network namespace ของ container นั้น **ไม่ใช่ host** ที่ `docker run -p` map port มาจากข้างนอก การ bind ที่
`127.0.0.1` หมายความว่า process จะรับ connection แค่จาก**ภายใน network namespace เดียวกัน**เท่านั้น (เช่น จาก
process อื่นใน container เดียวกัน) connection ที่มาจาก host ผ่าน port mapping ถือเป็น "จากข้างนอก" ในมุมมองของ
network namespace จึงถูกปฏิเสธ — **ทางแก้คือ bind ที่ `0.0.0.0` เสมอสำหรับ service ที่รันใน container** (รับ
connection จากทุก network interface ภายใน container รวมถึง interface ที่ port mapping ใช้) ทุก Dockerfile ใน
บทนี้ตั้งใจใช้ `axum::serve(TcpListener::bind("0.0.0.0:8080")...)` มาตั้งแต่ต้นด้วยเหตุผลนี้โดยตรง

**2. `.dockerignore` หายไป ทำให้ `docker build` ช้าและอาจดึงไฟล์ที่ไม่ต้องการเข้า context**

ถ้าไม่มี `.dockerignore` และรัน `docker build .` ใน directory ที่มี `target/` ของ Cargo อยู่ (จากการ `cargo
build` บนเครื่องมาก่อน) **Docker จะส่งทั้ง `target/` (อาจหลาย GB) เข้า build context ก่อนเริ่ม build ทุกครั้ง**
แม้ Dockerfile จะไม่ได้ `COPY target` เลยก็ตาม เพราะ Docker ต้อง "อ่าน" ทั้ง context เข้าไปก่อนเสมอเพื่อให้พร้อม
สำหรับ `COPY`/`ADD` ที่อาจอ้างถึงไฟล์ไหนก็ได้ — วิธีแก้ง่ายมาก:

```
# .dockerignore
**/target
**/dist
**/*.log
.git
```

**3. Docker Hub registry rate limit (`429 Too Many Requests`) ตอน build ซ้ำหลายครั้งในเวลาสั้น ๆ**

ระหว่างทดลองบทนี้ พบ error นี้จริงหลายครั้งตอน build Dockerfile ที่อ้าง `FROM rust:1-slim`/`rust:1-alpine`
ซ้ำ ๆ ในเวลาสั้น ๆ:

```
ERROR: unexpected status from HEAD request to https://registry-1.docker.io/v2/library/rust/manifests/1-slim:
429 Too Many Requests
```

**เหตุผล**: Docker Hub จำกัดจำนวน pull ต่อ IP ต่อช่วงเวลา (rate limit สำหรับ anonymous pull) — ทุกครั้งที่
`docker build` เจอ `FROM image:tag` ที่เป็น mutable tag (เช่น `1-slim` ที่อาจมีเวอร์ชันใหม่กว่า) มันจะยิง `HEAD`
request ไปเช็ค manifest ล่าสุดแม้ image นั้นจะมีอยู่ในเครื่องแล้วก็ตาม (เพื่อเช็คว่ามีเวอร์ชันใหม่กว่าไหม) — ถ้า
build ซ้ำถี่เกินไปจะโดน rate limit **วิธีแก้**: (1) รอสักพักแล้ว retry, (2) ใช้ `--pull=false` เพื่อบอก
BuildKit ให้ใช้ image ที่มีอยู่ในเครื่องโดยไม่เช็ค registry เลย (ใช้ได้เมื่อรู้แน่ชัดว่า image ในเครื่องคือ
เวอร์ชันที่ต้องการแล้ว), หรือ (3) ใน CI/production ที่ build บ่อยมาก ควร login เข้า Docker Hub account (หรือใช้
registry mirror ขององค์กร) เพื่อได้ rate limit ที่สูงกว่า anonymous pull มาก

**4. `argon2`/`password-hash` crate เปลี่ยน API ระหว่างเวอร์ชัน — `cargo check` ผ่านไม่ได้ถ้า pin เวอร์ชันผิด**

ระหว่างเขียนโค้ดสำหรับหัวข้อ 96.10 (ที่จำลอง dependency set ของ Part 92 ให้ตรงเป๊ะ) เจอ compiler error จริงตอน
ลองเขียนตาม API เก่าที่ต้องสร้าง `SaltString` เองแล้วส่งเข้า `hash_password` สองพารามิเตอร์:

```
error[E0432]: unresolved imports `argon2::password_hash::rand_core`, `argon2::password_hash::SaltString`
error[E0061]: this method takes 1 argument but 2 arguments were supplied
   |
 = help: remove the extra argument
```

**เหตุผล**: `argon2`/`password-hash` เวอร์ชัน 0.6.x เปลี่ยน API ของ `hash_password` ให้ **สุ่ม salt ให้เองภายใน
โดยอัตโนมัติ** (ผ่าน feature `getrandom` ที่เปิดเป็น default) ไม่ต้องสร้าง `SaltString`/`OsRng` เองแบบ API รุ่น
เก่าอีกต่อไป — วิธีแก้คือเรียกแบบพารามิเตอร์เดียวตรง ๆ (`argon2.hash_password(password.as_bytes())`) ตรงตามที่
Part 92 หัวข้อที่สอน `hash_password` เขียนไว้แล้ว — **บทเรียนที่กว้างกว่าเรื่อง Docker**: เมื่อ containerize
โปรเจกต์เก่าด้วย `Cargo.lock` ที่ pin เวอร์ชันไว้แน่นอน (`Cargo.lock` ที่ commit เข้า git ตามที่ Part 17/70
สอนไว้) ปัญหานี้จะไม่เกิดเลยเพราะ build จะได้ dependency เวอร์ชันเดียวกันเป๊ะทุกครั้ง — ปัญหาจะเกิดเฉพาะตอนสร้าง
โปรเจกต์ใหม่โดยไม่มี `Cargo.lock` เดิมมาก่อน (แบบที่หัวข้อ 96.10 ทำเพื่อทดลอง) ซึ่งเป็นอีกเหตุผลหนึ่งที่ต้อง
`COPY Cargo.lock` เข้า Docker build context เสมอ ไม่ปล่อยให้ `.dockerignore` กันมันออกไปโดยไม่ตั้งใจ

**5. `apt-get install` ล้มเหลวในสภาพแวดล้อมที่ network ถูกจำกัด (CI runner บางประเภท, sandbox ที่ควบคุม egress)**

ทดสอบจริงในสภาพแวดล้อมเขียนบทนี้ (ที่ egress ถูกควบคุมด้วย policy proxy) พบว่า `apt-get update` ภายใน container
ล้มเหลวจริง:

```
$ docker run debian:bookworm-slim bash -c "apt-get update"
Err:1 http://deb.debian.org/debian bookworm InRelease
  405 Method Not Allowed [IP: 127.0.0.1 33177]
E: The repository 'http://deb.debian.org/debian bookworm InRelease' is not signed.
E: Failed to fetch http://deb.debian.org/debian/dists/bookworm/InRelease  405 Method Not Allowed
```

**เหตุผล**: environment บางประเภท (sandbox ที่ควบคุม network egress อย่างเข้มงวด, บาง corporate proxy) อนุญาต
แค่โดเมนที่อยู่ใน allowlist (เช่น `index.crates.io` สำหรับ `cargo`) แต่ปฏิเสธโดเมนอื่น (เช่น `deb.debian.org`
สำหรับ `apt-get`) — **นี่ไม่ใช่ปัญหาของ Docker เอง** เครื่อง dev/CI runner มาตรฐานทั่วไป (GitHub Actions
runner ของ GitHub เอง, GitLab shared runner) ไม่มีข้อจำกัดแบบนี้ — แต่ก็เป็นเหตุผลที่ดีอีกข้อที่สนับสนุนแนวทาง
ของบทนี้: Dockerfile ทุกไฟล์ในบทนี้**หลีกเลี่ยง `apt-get install` โดยสิ้นเชิง** (ติดตั้ง `trunk`/`cargo-chef`
ผ่าน `cargo install` จาก crates.io แทน, runtime image ไม่ต้อง package เพิ่มเลยเพราะพึ่ง pure-Rust crypto
เท่านั้น) ทำให้ Dockerfile เหล่านี้ **build ผ่านได้ในสภาพแวดล้อมที่จำกัดกว่าปกติด้วย** โดยไม่ได้เสียอะไรไปเลย
(image เล็กกว่า, build เร็วกว่า, attack surface น้อยกว่าตามที่หัวข้อ 96.3-96.4 อธิบายไว้)

**6. ลืมว่า `depends_on` แบบไม่มี `condition` ใน Docker Compose รอแค่ "container start" ไม่ใช่ "พร้อมใช้งาน"**

```yaml
# ผิด -- ไม่มี condition
app:
  depends_on:
    - db
```

รูปแบบนี้ทำให้ Compose รอแค่ให้ container `db` เข้าสถานะ "Running" (process เริ่มทำงานแล้ว) ก่อนจะ start `app`
— แต่ PostgreSQL/Redis ยังต้องใช้เวลาเริ่มต้นตัวเองอีกเล็กน้อยหลัง process เริ่ม (initialize data directory,
เปิด listener) ก่อนจะพร้อมรับ connection จริง — ถ้า `app` เชื่อมต่อเร็วเกินไปจะได้ connection error ตอน startup
(อาจ crash หรือ retry loop ที่ไม่จำเป็น) **วิธีแก้ที่ถูกต้องคือ `condition: service_healthy` คู่กับ
`healthcheck`** ตามที่หัวข้อ 96.9-96.10 ใช้ตลอด — `depends_on` เฉย ๆ เหมาะกับกรณีที่ไม่สนใจลำดับความพร้อมจริง
(เช่น service ที่ retry connection เองอยู่แล้วอย่างทนทาน) แต่สำหรับ demo/dev ที่ต้องการความแน่นอนของลำดับ
`service_healthy` ปลอดภัยกว่าเสมอ

**7. เข้าใจผิดว่า `EXPOSE` ใน Dockerfile "เปิด port" ให้เข้าถึงจากนอก container ได้**

```dockerfile
FROM debian:bookworm-slim
COPY app /app
EXPOSE 8080
CMD ["/app"]
```

หลายคนเห็น `EXPOSE 8080` แล้วเข้าใจว่าต้องมีบรรทัดนี้ถึงจะ "เปิด" ให้เข้าถึง port 8080 จาก host ได้ — ความจริง
**`EXPOSE` เป็นแค่ documentation/metadata** ที่บอกว่า image นี้ตั้งใจให้ process ข้างในฟังที่ port ไหน (เห็นได้
ผ่าน `docker inspect` ตามที่หัวข้อ 96.2 แสดงไว้) **ไม่มีผลต่อการเข้าถึงจริงจาก host เลยแม้แต่นิดเดียว** — ทุก
Dockerfile ในบทนี้ยังต้องพึ่ง `-p host_port:container_port` ตอน `docker run` (หรือ `ports:` ใน compose) เพื่อ
ทำ port mapping จริง ไม่ว่าจะมี `EXPOSE` หรือไม่ก็ตาม — ลบ `EXPOSE` ทิ้งไปเลยก็ยัง `docker run -p
18080:8080 image` ได้ผลเหมือนกันทุกประการ (ลองเทียบได้จริงกับ Dockerfile ของหัวข้อ 96.4 ที่ลบ `EXPOSE 8080`
ออก แล้ว build ใหม่ — `curl` ยัง 200 เหมือนเดิม) ประโยชน์ที่แท้จริงของ `EXPOSE` มีแค่สองอย่าง: (1) เป็น
documentation ที่อ่านง่ายสำหรับคนอื่นที่มาดู Dockerfile ทีหลัง และ (2) `docker run -P` (ตัวใหญ่ ไม่มี port
ระบุ) จะ map port แบบสุ่มให้เฉพาะ port ที่ประกาศด้วย `EXPOSE` เท่านั้น — ไม่ใช่กลไก security ใด ๆ ทั้งสิ้น

**8. อ่านค่า SIZE ของ `docker images` แล้วสรุปผิดว่า "ต้อง push/pull เท่านี้เสมอ"**

ตลอดบทนี้แสดงตัวเลขสองคอลัมน์เสมอคือ **Disk Usage** และ **Content Size** (เช่นของ `greet-multistage:latest` คือ
116MB เทียบ 28.9MB) — กับดักที่พบบ่อยคือมองแค่ตัวเลขเดียว (มักเป็น Disk Usage เพราะมาก่อน) แล้วสรุปว่า "image
นี้กิน bandwidth เท่านี้ทุกครั้งที่ deploy" ซึ่ง**ไม่ถูกต้องเสมอไป** — ตามที่หัวข้อ 96.2 อธิบายไว้ Disk Usage
นับ layer ที่อาจ **share กับ image อื่นที่มีอยู่ในเครื่องนั้นแล้ว** เข้าไปด้วย (เช่น layer ของ
`debian:bookworm-slim` base ที่หลาย image ใน registry เดียวกันอาจใช้ร่วมกัน) ในขณะที่ Content Size คือขนาดที่
ต้อง transfer จริงถ้าปลายทาง**ไม่มี layer ไหนอยู่ก่อนเลย** (worst case) — ในสถานการณ์จริงที่ deploy ไป server
เดิมซ้ำ ๆ (เช่น CI/CD ที่ Part 97 จะสอน) หลัง deploy ครั้งแรกไปแล้ว **server มักมี base image layer อยู่แล้ว**
ทำให้ deploy ครั้งต่อ ๆ ไปโอนแค่ layer ที่เปลี่ยนจริง (มักเป็น layer สุดท้ายที่มี binary ของแอปเท่านั้น) — ตัวเลข
ทั้งสองแบบมีประโยชน์คนละสถานการณ์: Disk Usage สำหรับวางแผนพื้นที่ disk ของเครื่อง build/runtime, Content Size
สำหรับประเมิน bandwidth ของการ deploy ครั้งแรกสุด (cold deploy ไป environment ใหม่)

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียน `Dockerfile` แบบ multi-stage สำหรับโปรเจกต์ Rust ง่าย ๆ ที่มีแค่ `println!("Hello,
   Docker!")` (ไม่ต้องมี `axum`/network เลย) ใช้ `rust:1-slim` เป็น builder และ `debian:bookworm-slim` เป็น
   runtime แล้ว build+run จริง วัดขนาด image ที่ได้ — เทียบกับขนาดของ `rust:1-slim` เองว่าเล็กลงเท่าไร (hint:
   ใช้ `docker images` เทียบทั้งสอง image)
2. **(กลาง)** เอา Dockerfile จากข้อ 1 มาลองเปลี่ยนเป็น static build ด้วย `x86_64-unknown-linux-musl` แล้วรันบน
   `FROM scratch` ให้สำเร็จ — จากนั้นลองเพิ่ม dependency ที่ใช้ `reqwest` แบบ default feature (ที่ใช้
   `native-tls`/OpenSSL) เข้าไปแล้วลอง build ใหม่ด้วย musl target อีกครั้ง สังเกตว่าเกิด error อะไร (hint:
   เปลี่ยนไปใช้ `reqwest = { version = "...", default-features = false, features = ["rustls-tls"] }` แทนแล้ว
   ลองใหม่ ควร build ผ่าน — นี่คือการพิสูจน์ trade-off ของหัวข้อ 96.5 ด้วยมือตัวเอง)
3. **(ยาก)** ต่อยอดจากหัวข้อ 96.9 (`presence-api`) เพิ่ม service ที่สามคือ `nginx` ที่ทำหน้าที่ reverse proxy
   หน้า `app` (แทนการ expose port ของ `app` ตรง ๆ) พร้อมตั้ง healthcheck ของ `nginx` เองที่เช็คว่า proxy ไปถึง
   `app` ได้จริง (ไม่ใช่แค่เช็คว่า nginx process รันอยู่) — เขียน `depends_on` ให้ถูกลำดับทั้งสาม service (db/
   cache ต้องพร้อมก่อน app, app ต้องพร้อมก่อน nginx) (hint: nginx healthcheck ใช้ `curl` เช็ค endpoint ของ
   ตัวเองที่ proxy ผ่านไปหา `app:8080/health` ได้ ถ้า `nginx:alpine` ไม่มี `curl` ให้ลองใช้ image
   `nginx:alpine-slim` หรือติดตั้งเพิ่มด้วยวิธีที่ไม่ต้องพึ่ง `apt-get`)
4. **(ยาก/ประยุกต์)** เอา capstone ของหัวข้อ 96.10 (`library_api_mini`) มาปรับ Dockerfile ให้ใช้ `cargo-chef`
   แทนเทคนิคแยก manifest แบบมือ (ตามที่หัวข้อ 96.6 สอนไว้ทั้งสองวิธี) แล้ว build+run จริงให้ผ่าน end-to-end
   (health → register → login) เหมือนหัวข้อ 96.10 ทุกประการ จากนั้นวัดเวลา build เทียบกันสามกรณี: cold build,
   rebuild หลังแก้ `src/main.rs` เพียงบรรทัดเดียว, และ rebuild หลังเพิ่ม dependency ใหม่ใน `Cargo.toml` — อธิบาย
   ว่าทำไมกรณีที่สามช้ากว่ากรณีที่สองเสมอไม่ว่าจะใช้เทคนิค caching แบบไหนก็ตาม (hint: dependency graph
   เปลี่ยนแปลง invalidate cache ของ layer "cook"/"build deps" เองโดยตรง ไม่ใช่ปัญหาของเทคนิคที่เลือก)

**ตารางสรุปการตัดสินใจของบททั้งหมด** (อ้างอิงกลับเวลาต้องเลือกทางระหว่างทำโปรเจกต์จริง):

| ต้องตัดสินใจ | เลือกอย่างไร |
|---|---|
| Runtime base image | ไม่มี native dependency เลย → musl+`scratch` (96.5); มี native dependency ที่ static link ไม่ได้ หรือต้อง debug บ่อย → `debian:bookworm-slim` (96.4)/distroless (96.5) |
| ลด build time ที่ dependency ไม่เปลี่ยน | โปรเจกต์เดียว/binary เดียว → แยก manifest ด้วยมือ (96.6); workspace หลาย crate/binary → `cargo-chef` (96.6); ต้องการ diff น้อยที่สุดจาก naive Dockerfile → BuildKit `--mount=type=cache` (96.6) |
| Workspace ที่มีหลาย binary | build ทีละ image ด้วย `-p <crate>` แยกตามบทบาท ไม่รวมทุก binary ไว้ image เดียว (96.7) |
| Secret/config | environment variable ตอน `docker run`/compose เท่านั้น ไม่มีใน `ARG`/`ENV` ที่ build เข้า image เด็ดขาด (96.8) |
| Dev environment หลายบริการ | `docker-compose.yml` + `healthcheck`/`depends_on: condition: service_healthy` (96.9) |
| Tag สำหรับ deploy จริง | commit SHA/semantic version ที่ immutable ไม่พึ่ง `latest` เป็น mechanism หลัก (96.11) |

## สรุป

บทนี้พาไปดู containerization สำหรับ Rust อย่างลึกและตรงไปตรงมา เริ่มจากคำถามที่คนมักมองข้าม — **Rust binary
ที่ compile แบบ release ไม่มีปัญหา dependency แบบที่ Node.js/Python มี** ดังนั้นเหตุผลที่ container มีประโยชน์
จริงกับแอป Rust คือเรื่อง native dependency ระดับ OS, ความสม่ำเสมอของ build environment, และมาตรฐานเดียวกันของ
deployment/orchestration — ไม่ใช่การแก้ปัญหาที่ Rust ไม่มีอยู่แล้ว จากนั้นลงรายละเอียดกลไก Docker เอง (image,
container, layer, cache) ที่เป็นพื้นฐานของทุกเทคนิคที่ตามมา แล้วพิสูจน์ด้วยตัวเลขจริงตลอดทั้งบท: single-stage
`rust:latest` ให้ image 2.56GB, multi-stage ลดเหลือ 116MB (เล็กลง 22 เท่า), และ musl static linking บน
`scratch` ลดเหลือเพียง 2.18MB (เล็กลงเกือบ 940 เท่าจากจุดเริ่มต้น) — ปัญหา dependency caching ที่ทำให้ build
ช้าถูกแก้ด้วยสองเทคนิค (แยก manifest ด้วยมือ, หรือ `cargo-chef`) พิสูจน์ด้วยตัวเลขเวลาจริงที่เร็วขึ้นกว่า 8 เท่า
สำหรับการแก้โค้ดแอปเพียงเล็กน้อย — ปิดท้ายด้วย environment/secret handling ที่ถูกต้อง, `docker-compose` สำหรับ
dev environment ที่มีหลายบริการ, และ capstone ที่นำทุกเทคนิครวมกันพิสูจน์กับ dependency set จริงของ Part 92-94
ว่า static linking กับ dependency caching ใช้ได้จริงกับแอป production-grade ที่มี auth เต็มรูปแบบ ไม่ใช่แค่
hello-world

Part ถัดไป (Part 97) จะพา container image ที่บทนี้สร้างไว้ไปสู่ **CI/CD pipeline อัตโนมัติด้วย GitHub
Actions** — build/test/push image ทุกครั้งที่ push commit โดยอัตโนมัติ พร้อมใช้ประโยชน์จาก layer caching ข้าม
CI runner ที่บทนี้เกริ่นไว้ในหัวข้อ 96.2 ว่าต้องตั้งค่าเพิ่มเติม (registry cache/BuildKit cache export) ให้ทำงาน
ได้จริงในสภาพแวดล้อม CI ที่เป็น container สดใหม่ทุกครั้ง

---

**Part ก่อนหน้า:** [Testing Web Applications แบบครบวงจร (unit/integration/e2e)](part-095-testing-web-apps.md) | **Part ถัดไป:** [CI/CD Pipeline ด้วย GitHub Actions](part-097-cicd-github-actions.md)
