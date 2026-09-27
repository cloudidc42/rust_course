# Part 101: Deployment: Cloud Platforms (AWS/GCP/Fly.io)

> โมดูล: Production, DevOps และระดับมืออาชีพ | ระดับ: มืออาชีพ | เวลาโดยประมาณ: 300 นาที

## เป้าหมายของบทเรียน

หลังจากจบบทนี้ คุณจะสามารถ:

- เลือกแนวทาง deploy ที่เหมาะกับสถานการณ์จริงได้ ระหว่าง VM ดิบ ๆ, container orchestration เต็มรูปแบบ
  (Kubernetes) และ PaaS ที่เรียบง่าย (Fly.io/Railway/Render) โดยใช้ decision framework ที่มีเหตุผลรองรับ
  ชัดเจน ไม่ใช่แค่ "เพราะคนอื่นใช้"
- อ่านและเขียน `fly.toml` ได้ครบทุก section หลัก (`[build]`, `[env]`, `[http_service]`,
  `[http_service.checks]`, `[[vm]]`) พร้อมรู้ว่าแต่ละ field มีผลจริงต่อพฤติกรรมของแอปตอน deploy อย่างไร
  และเชื่อมกับ image ที่ containerize ไว้แล้วจาก Part 96 ได้ถูกต้อง
- ใช้ `cargo lambda` build และ invoke AWS Lambda function ที่เขียนด้วย Rust ได้จริงในเครื่อง (local emulator)
  โดยไม่ต้องมี AWS account หรือ credential ใด ๆ เลย และเข้าใจว่าทำไม `cargo lambda`/`cargo-lambda` ถึงทำสิ่งนี้
  ได้แบบ offline 100%
- เปรียบเทียบตัวเลือก deploy หลักของ AWS (ECS/Fargate, EC2, Lambda) และของ GCP (Cloud Run) ได้อย่างมีเหตุผล
  ว่าแต่ละตัวเหมาะกับ workload แบบไหน พร้อมเชื่อมโยงกับ container image ที่ Part 96 สร้างไว้
- ตั้งค่า config/secret ของแอปที่ deploy อยู่บนคลาวด์ตามหลัก environment-variable-based config (ต่อยอดจาก
  Part 96 หัวข้อ 96.8) และรู้ syntax ของคำสั่ง CLI ของแต่ละแพลตฟอร์มที่ใช้ตั้งค่าเหล่านี้ (`fly secrets set`,
  AWS Systems Manager Parameter Store, Cloud Run `--set-env-vars`) เป็นข้อมูลอ้างอิง
- อธิบายกลไก zero-downtime deployment (rolling deployment, health-check-gated cutover) และการตั้งค่า
  DNS/TLS ผ่านแพลตฟอร์ม managed ได้ พร้อมประกอบ `fly.toml` + `Dockerfile` ฉบับสมบูรณ์สำหรับ deploy แอป
  capstone จาก Part 92-96 จริง โดยรู้ชัดเจนว่าส่วนไหนตรวจสอบจริงในเครื่อง ส่วนไหนอ้างอิงจากเอกสารทางการ

## ความรู้ที่ต้องมีมาก่อน

- **Part 92-96 (Full-Stack Capstone + Docker)** — บทนี้ deploy image ที่ Part 96 containerize ไว้ (`library_api_mini`
  แบบ musl static บน `scratch`, ขนาด 7.84MB ตามที่ Part 96 หัวข้อ 96.10 พิสูจน์ด้วยตัวเลขจริง) ไปสู่คลาวด์จริง
  — ถ้ายังไม่เข้าใจ multi-stage build/musl static linking/environment-variable config ของ Part 96 ควรย้อนกลับไป
  อ่านก่อน เพราะบทนี้ไม่สอนกลไก Docker ซ้ำ แต่ใช้ผลลัพธ์ของมันตรง ๆ
- **Part 81 (Microservices พื้นฐาน)** — แนวคิด health check/readiness endpoint (`/health`) ที่ Part 94 หัวข้อ
  94.7 implement ไว้ — บทนี้ใช้ endpoint เดียวกันเป็น health check ของ `fly.toml`/AWS target group/Cloud Run
  โดยตรงในหัวข้อ 101.10
- **Part 97 (CI/CD Pipeline ด้วย GitHub Actions)** — pipeline ที่ build/push image อัตโนมัติ — บทนี้ต่อยอดว่า
  "หลัง CI build image สำเร็จแล้ว จะเอา image นั้นไป deploy ที่ไหนและอย่างไร" ซึ่งเป็นคำถามที่ Part 97 ยังไม่ได้
  ตอบเต็มรูปแบบ
- **Part 100 (Security Best Practices)** — หัวข้อ TLS/`rustls`/ACME/Let's Encrypt ของ Part 100 หัวข้อ 100.8
  เป็นฐานที่หัวข้อ 101.11 ของบทนี้อ้างอิงตรงเมื่ออธิบายว่าทำไม managed platform ถึง "แทนที่" งาน TLS termination
  ที่ Part 100 สอนให้ทำเองด้วย `rustls`
- **Part 70 (SQLx และ PostgreSQL)** — `DATABASE_URL` และการต่อ PostgreSQL ผ่าน connection string — หัวข้อ
  101.9 (managed database) ใช้รูปแบบ connection string เดียวกันตรง ๆ แค่เปลี่ยนจาก `db:5432` (hostname ภายใน
  docker-compose) เป็น hostname ของ managed database provider จริง

## หมายเหตุเรื่องการตรวจสอบเนื้อหา (สำคัญ — อ่านก่อนเริ่ม)

บทนี้เขียนขึ้นในสภาพแวดล้อมที่ **network egress ถูกจำกัดด้วย allowlist ของ domain** (เหตุผลด้านความปลอดภัย
ของระบบที่รันหลาย agent เขียนหลาย Part พร้อมกัน) — allowlist อนุญาตแค่ `index.crates.io` (สำหรับ `cargo`),
`pypi.org`/`files.pythonhosted.org` (สำหรับ `pip`), `registry.npmjs.org` (สำหรับ `npm`), `proxy.golang.org`
(สำหรับ `go install`) และโดเมนที่เกี่ยวข้องกับ Anthropic เอง — **ไม่รวม `fly.io` (install script/release CDN)
หรือ `github.com` (release binary ของ `flyctl`)** ทำให้ **ไม่สามารถติดตั้ง `flyctl` ตัวจริงในสภาพแวดล้อมนี้ได้**
(ลองทั้งสามทาง: (1) `curl https://fly.io/install.sh` โดนปฏิเสธที่ proxy ด้วย `403`, (2) npm package ชื่อ
`flyctl` เป็นแค่ wrapper library ไม่มี binary จริงติดมาด้วย, (3) `go install github.com/superfly/flyctl@latest`
ผ่าน `proxy.golang.org` ได้จริง แต่ทุกเวอร์ชันที่มีอยู่ต้องการ **Go ≥ 1.26** ขณะที่เครื่องนี้มี Go 1.24.7 และ
การให้ `go` ดาวน์โหลด toolchain ใหม่อัตโนมัติ (`GOTOOLCHAIN=auto`) ทำให้ตัว compiler ที่ดาวน์โหลดมาใหม่รันไม่ได้
ใน sandbox นี้ — ดูรายละเอียดเต็มในกับดักที่พบบ่อยข้อ 5) — **ดังนั้นทุก syntax ของคำสั่ง `flyctl`/schema ของ
`fly.toml` ในบทนี้คือข้อมูลอ้างอิงที่ตรวจทานจากเอกสารทางการของ Fly.io (`fly.io/docs`) ให้ถูกต้องที่สุด
ไม่ใช่ผลจากการรันคำสั่งจริงในสภาพแวดล้อมนี้ — จะระบุไว้ชัดเจนทุกจุดที่เป็นแบบนี้**

ในทางกลับกัน **`cargo-lambda` ติดตั้งได้จริงผ่าน `pip` (แจกจ่ายเป็น Python wheel บน PyPI ที่มี binary native
ฝังอยู่ข้างใน ซึ่ง `pypi.org`/`files.pythonhosted.org` อยู่ใน allowlist) และใช้งานได้แบบ offline 100%**
เพราะ subcommand ที่บทนี้ใช้ (`build`, `watch`, `invoke`) เป็นการ**จำลอง (emulate)** AWS Lambda runtime ทั้งหมด
ในเครื่องเอง ไม่ได้เชื่อมต่อ AWS จริง — บทนี้ install จริง, build จริง, และ invoke local emulator จริงด้วย
`cargo lambda invoke` แล้ว capture output จริงมาแสดงทั้งหมด (ไม่มีจุดใดแต่งขึ้นเอง) ส่วนคำสั่งที่ต้องใช้ AWS
account จริง (`cargo lambda deploy`, `fly deploy`, `aws ecs create-service` ฯลฯ) จะระบุไว้ชัดเจนว่า **ไม่ได้
รันจริง เพราะสภาพแวดล้อมนี้ไม่มี credential ของ cloud provider ใดเลย** และอธิบายจากเอกสารทางการเท่านั้น

สรุปสถานะการตรวจสอบของบทนี้แบบย่อ (รายละเอียดเต็มอยู่ในหัวข้อ 101.13):

| รายการ | สถานะ |
|---|---|
| `cargo-lambda` ติดตั้งจริง, `cargo lambda build`/`watch`/`invoke` | ✅ รันจริง มี output จริงทั้งหมด |
| `fly.toml` schema/field ต่าง ๆ | 📖 อ้างอิงจากเอกสาร Fly.io — TOML syntax ตรวจสอบผ่าน parser จริง (Python `tomllib`) |
| `flyctl` command syntax (`fly launch`, `fly deploy`, `fly secrets set` ฯลฯ) | 📖 อ้างอิงจากเอกสาร Fly.io เท่านั้น (ติดตั้ง binary จริงไม่ได้ในสภาพแวดล้อมนี้) |
| ตาราง AWS/GCP/managed database comparison | 📖 ข้อมูลเชิงแนวคิด ไม่ต้อง login |
| ราคา/cost order-of-magnitude | 📖 ประมาณการจาก public pricing page ณ ช่วงที่เขียนบทนี้ อาจเปลี่ยนแปลงได้ |

## เนื้อหา

### 101.1 ภาพรวมการ Deploy สมัยใหม่: VM, Container Orchestration, และ PaaS

ก่อนจะลงรายละเอียดของแพลตฟอร์มใดแพลตฟอร์มหนึ่ง ต้องเข้าใจภาพใหญ่ก่อนว่า "การ deploy" ในโลกปัจจุบันมีสามระดับของ
abstraction ที่ต่างกันชัดเจน และ Part 96 ได้เตรียมสิ่งที่จำเป็นสำหรับทั้งสามระดับไว้แล้ว (container image) — สิ่งที่
บทนี้ต้องตัดสินใจคือ "จะเอา image นั้นไปรันที่ไหน"

**ระดับที่ 1 — Virtual Machine (VM) ดิบ ๆ**: คุณเช่าเครื่อง virtual (EC2 instance, GCP Compute Engine VM,
DigitalOcean Droplet) ที่มาพร้อม OS เปล่า ๆ แล้วต้องจัดการทุกอย่างเอง: ติดตั้ง Docker (หรือรัน binary ตรง ๆ),
ตั้งค่า reverse proxy (nginx/Caddy), ตั้งค่า TLS certificate เอง (ตามที่ Part 100 หัวข้อ 100.8 สอน),
ตั้งค่า systemd service ให้ auto-restart ถ้า process crash, ตั้งค่า firewall เอง, และถ้าต้อง scale ออกหลายเครื่อง
ต้องตั้ง load balancer เองด้วย — **ข้อดี**: ควบคุมได้ทุกรายละเอียด ไม่มี abstraction layer มาบัง ค่าใช้จ่ายมัก
ต่ำที่สุดต่อหน่วย compute ถ้า workload คงที่ (ไม่ scale ขึ้นลงบ่อย) — **ข้อเสีย**: ทุกอย่างที่กล่าวมาต้องทำเอง
และดูแลเองตลอดชีวิตของระบบ (patch OS, จัดการ certificate renewal, monitor เครื่อง) ซึ่งเป็นงานที่ไม่เกี่ยวกับ
ตัวแอปเลยแต่กินเวลาทีมจริง

**ตัวอย่าง systemd unit สำหรับกรณี deploy บน VM ดิบ**: ถ้าเลือกทางนี้ ส่วนที่ต้องทำเองคือให้ OS (Linux ที่ใช้
`systemd` เป็น init system มาตรฐานของ distro สมัยใหม่ส่วนใหญ่ เช่น Ubuntu/Debian/RHEL) จัดการ auto-restart
ให้ binary ที่ build ไว้แล้ว (จาก Part 96 หรือ compile ตรงบนเครื่องเลยก็ได้ถ้าไม่ใช้ container) — เขียนไฟล์
`/etc/systemd/system/library-api-mini.service`:

```ini
[Unit]
Description=library_api_mini web service
After=network.target

[Service]
Type=simple
ExecStart=/opt/library-api-mini/library_api_mini
Restart=on-failure
RestartSec=5
Environment=BIND_ADDR=0.0.0.0:8080
EnvironmentFile=/etc/library-api-mini/secrets.env
User=appuser

[Install]
WantedBy=multi-user.target
```

`Restart=on-failure` คือกลไก **self-healing ระดับพื้นฐานที่สุด** (เทียบกับที่ Fly.io/ECS/Kubernetes ทำให้
อัตโนมัติในระดับ Machine/Task/Pod) — ถ้า process crash `systemd` จะ start ใหม่ให้เองหลัง `RestartSec` วินาที
— `EnvironmentFile` คือทางที่ใกล้เคียงกับหลัก "แยก secret ออกจาก config ทั่วไป" ของหัวข้อ 101.8 มากที่สุดใน
บริบทของ VM ดิบ (ไฟล์ `secrets.env` ต้องตั้ง permission ให้อ่านได้แค่ `root`/`appuser` เท่านั้น ไม่ commit
เข้า git เด็ดขาด) — เปิดใช้งานด้วย `systemctl enable --now library-api-mini` — สังเกตว่า**ทุกอย่างที่
`systemd` ทำให้ตรงนี้ คือสิ่งที่ Fly.io/ECS/Kubernetes ทำให้อัตโนมัติแบบไม่ต้องเขียนไฟล์ config แยกแบบนี้เลย**
— นี่คือรูปธรรมของ "operational overhead" ที่หัวข้อนี้พูดถึงตอนต้น: งานเดียวกัน (auto-restart, env var
injection) ทำได้ทั้งสองทาง แต่ทางหนึ่งต้องเขียน/ดูแลไฟล์ config ระดับ OS เองทุกเครื่อง อีกทางหนึ่งแพลตฟอร์ม
จัดการให้หมด

**ระดับที่ 2 — Container Orchestration เต็มรูปแบบ (Kubernetes)**: Kubernetes (K8s) คือระบบที่รับ container
image (image เดียวกับที่ Part 96 สร้าง) แล้วจัดการ **scheduling** (จะรัน container ไหนบนเครื่องไหนใน cluster),
**self-healing** (container crash แล้ว restart อัตโนมัติ, เครื่อง (node) ตายแล้วย้าย container ไปเครื่องอื่น),
**scaling** (เพิ่ม/ลดจำนวน replica ตาม CPU/memory หรือ metric ที่กำหนด), **service discovery** (container คุย
กันเองผ่านชื่อ service โดยไม่ต้องรู้ IP จริง), และ **rolling update** (หัวข้อ 101.10 จะอธิบายลึกกว่านี้) —
Kubernetes เป็นมาตรฐานอุตสาหกรรมที่ทุก cloud provider ใหญ่มี managed offering ให้ (EKS ของ AWS, GKE ของ GCP,
AKS ของ Azure) เพราะมันแก้ปัญหาการ orchestrate container ได้ครอบคลุมที่สุดและพกพาข้าม cloud provider ได้
(YAML manifest เดียวกันรันบน EKS/GKE/on-prem cluster ได้เกือบเหมือนกัน) — **ข้อเสียที่ต้องพูดตรง ๆ**: Kubernetes
มี "learning curve" และ "operational overhead" สูงมาก (concept อย่าง Pod, Deployment, Service, Ingress,
ConfigMap, Secret, PersistentVolume, Namespace, RBAC ของ K8s เอง — ไม่ใช่ RBAC ของแอปที่ Part 76 สอน — ต้อง
เรียนรู้ทั้งชุด) และสำหรับทีมเล็กหรือโปรเจกต์ที่ยังไม่ต้องการ scale ซับซ้อน มันคือ "ยกปืนใหญ่มายิงนก" — **บทนี้
จะไม่ลงรายละเอียดการใช้งาน Kubernetes จริง** (นั่นคือเนื้อหาระดับหนังสือแยกทั้งเล่ม) แต่ให้ผู้เรียนรู้จักไว้ในฐานะ
"ตัวเลือกที่มีอยู่และสำคัญมากในอุตสาหกรรม" เพื่อให้ตัดสินใจถูกว่าเมื่อไรควรก้าวไปเรียนมันต่อ

**ระดับที่ 3 — PaaS ที่เรียบง่าย (Fly.io, Railway, Render, Heroku)**: แพลตฟอร์มกลุ่มนี้รับ container image
(หรือ source code ที่จะ build image ให้อัตโนมัติ) แล้ว **ซ่อน** งาน orchestration ทั้งหมดของ Kubernetes ไว้หลัง
CLI/UI ที่เรียบง่ายมาก — สั่ง `fly deploy` คำสั่งเดียว แพลตฟอร์มจัดการ scheduling, health check, TLS
certificate (auto-provision ผ่าน Let's Encrypt ให้อัตโนมัติ — เชื่อมกับหัวข้อ 101.11), rolling deployment,
และ DNS ให้ทั้งหมด — **ข้อดี**: เริ่มต้นเร็วที่สุด ทีมเล็กหรือ solo developer deploy แอป production-grade ได้
ภายในนาทีเดียวโดยไม่ต้องรู้จัก Kubernetes เลย — **ข้อเสีย**: ควบคุมรายละเอียดได้น้อยกว่า Kubernetes ดิบ (เช่น
custom scheduling policy ที่ซับซ้อนมาก ๆ อาจทำไม่ได้), และแพลตฟอร์มเหล่านี้บางตัว "ผูก" คุณไว้กับ ecosystem
ของเขา (vendor lock-in ระดับหนึ่ง แม้ Fly.io จะยังคงใช้ container image มาตรฐานอยู่ ทำให้ portability สูงกว่า
Heroku แบบเดิมที่ใช้ buildpack เฉพาะตัว)

**Decision Framework — ตัดสินใจอย่างไรในสถานการณ์จริง**:

| สถานการณ์ | เลือกอย่างไร | เหตุผล |
|---|---|---|
| Solo developer/startup เล็ก ที่ต้องการ ship เร็ว, ทีม DevOps ไม่มีหรือมีน้อย | PaaS (Fly.io/Railway/Render) | ไม่มี operational overhead ของ K8s, deploy ได้ภายในนาทีเดียว, TLS/DNS อัตโนมัติ |
| องค์กรกลาง-ใหญ่ ที่มีหลาย service, ทีม DevOps เฉพาะทาง, ต้องการ portability ข้าม cloud | Kubernetes (EKS/GKE/self-managed) | ควบคุมละเอียด, มาตรฐานที่ทุกคนในอุตสาหกรรมรู้จัก, ไม่ผูกกับ provider เดียว |
| Workload ที่ traffic ไม่สม่ำเสมอมาก (event-driven, cron, webhook handler ที่นาน ๆ เรียกที) | Serverless (AWS Lambda/Cloud Run) | จ่ายตามการใช้งานจริง ไม่ต้องจ่ายค่าเครื่องที่ idle ตลอดเวลา |
| ต้องการควบคุมทุกรายละเอียดของ OS/network, มี compliance requirement เฉพาะที่ managed platform ทำไม่ได้ | VM ดิบ | ควบคุมได้ 100% แต่ต้องดูแลเองทั้งหมด |
| Workload คงที่มาก (ไม่ scale ขึ้นลง), ต้องการต้นทุนต่อหน่วย compute ต่ำที่สุด | VM แบบ reserved/spot instance | จ่ายล่วงหน้าได้ราคาถูกกว่า on-demand/serverless มาก สำหรับ workload ที่รันตลอด 24/7 |

บทนี้เลือกโฟกัสที่ **PaaS (Fly.io)** และ **Serverless (AWS Lambda)** เป็นหลัก เพราะทั้งสองแบบเหมาะกับสถานการณ์
ที่ผู้เรียนหลักสูตรนี้ (นักพัฒนา Rust ที่กำลังเรียนรู้ deploy แอป web service เดี่ยว ๆ) เจอบ่อยที่สุด — พร้อม
เกริ่น AWS ECS/EC2/GCP Cloud Run ในเชิงแนวคิดให้เห็นภาพครบ

**ลองประยุกต์ decision framework กับสถานการณ์จริงสามแบบ** (เพื่อให้เห็นว่าตารางด้านบนใช้งานจริงอย่างไร ไม่ใช่
แค่ท่องจำ):

- **สถานการณ์ที่ 1**: ทีม 2 คนกำลังสร้าง MVP ของแอป booking ระบบห้องประชุม (คล้าย capstone ของ Part 92-94)
  ต้องการ demo ให้ลูกค้าดูภายในสัปดาห์นี้ ยังไม่รู้ว่าจะมีผู้ใช้จริงกี่คน — **คำตอบ**: Fly.io หรือ Cloud Run
  ทั้งสองใช้ container image เดียวกับที่ Part 96 สร้างได้ตรง ๆ, `auto_stop_machines`/scale-to-zero ทำให้
  ค่าใช้จ่ายช่วง demo ต่ำมาก (ตามหัวข้อ 101.12), และไม่ต้องมีทีม DevOps แยกมาดูแล Kubernetes ที่ทีมเล็กขนาดนี้
  ไม่มีกำลังคนพอ
- **สถานการณ์ที่ 2**: บริษัทมี 15 microservice (ภาษาหลากหลาย: Rust, Go, Python) ทีม Platform Engineering
  8 คนดูแล infrastructure ให้ทุกทีม ต้องการมาตรฐานเดียวที่ scale ได้ถึงหลักร้อย instance และ portable ถ้า
  ต้องย้าย cloud provider ในอนาคต — **คำตอบ**: Kubernetes (EKS/GKE) เหมาะกว่า เพราะจำนวน service และทีมที่
  มากพอจะคุ้มกับ operational overhead ที่ต้องแบกรับ, และ manifest เดียวกันใช้กับทุกภาษาได้ (ไม่ต้องมี
  Fly.io/Cloud Run ต่างระบบสำหรับแต่ละภาษา)
- **สถานการณ์ที่ 3**: ระบบประมวลผลภาพที่ผู้ใช้อัปโหลดรูปแล้วต้อง resize/แปลง format — เกิดขึ้นไม่สม่ำเสมอมาก
  (บางวันไม่มีเลย บางวันมีหลักพันรูป) แต่ละงานใช้เวลาไม่กี่วินาที — **คำตอบ**: AWS Lambda (trigger จาก S3
  event ตอนมีไฟล์ใหม่) เหมาะที่สุด เพราะ scale-to-zero ธรรมชาติของ FaaS ตรงกับ traffic pattern แบบ "burst
  ไม่สม่ำเสมอ" เป๊ะ ไม่ต้องจ่ายค่าเครื่องช่วงที่ไม่มีงานเข้ามาเลย ต่างจาก Fly.io/ECS ที่ต้องมี Machine/task
  อย่างน้อย 1 ตัวพร้อมรับงานตลอดเวลาถ้าไม่อยากมี cold start (แม้ตั้ง scale-to-zero ได้เหมือนกัน แต่ Lambda
  ออกแบบมาเพื่อ pattern นี้โดยเฉพาะตั้งแต่ต้น)

**รู้จัก Kubernetes ไว้ในระดับแนวคิด (awareness เท่านั้น — บทนี้ไม่ลงมือใช้งานจริง)**: เพราะ Kubernetes เป็น
มาตรฐานที่พบได้ทั่วไปมากในสายงาน DevOps/Platform Engineering ผู้เรียนควรรู้จักคำศัพท์หลักไว้อย่างน้อยเพื่อ
อ่านเอกสาร/สื่อสารกับทีมอื่นได้ แม้บทนี้จะไม่ลงรายละเอียดการใช้งานจริง:

- **Pod** — หน่วยที่เล็กที่สุดที่ Kubernetes จัดการ ประกอบด้วย container หนึ่งตัวหรือมากกว่า (ปกติแอป Rust
  หนึ่งตัวคือหนึ่ง container ในหนึ่ง Pod) ที่ share network namespace เดียวกัน (คุยกันผ่าน `localhost` ได้)
  — เทียบได้กับ "Machine" หนึ่งตัวของ Fly.io ในความหมายที่ใกล้เคียงกัน
- **Deployment** — object ที่บอกว่าต้องการ Pod แบบไหนกี่ตัว (`replicas: 3`) พร้อมกลยุทธ์ rolling update
  (`maxUnavailable`/`maxSurge` ตามที่หัวข้อ 101.10 พูดถึง) — คล้าย `[[http_service]]`+`min_machines_running`
  ของ Fly.io รวมกัน
- **Service** — ให้ endpoint เครือข่ายที่คงที่สำหรับกลุ่ม Pod ที่อาจถูกสร้าง/ทำลายอยู่เรื่อย ๆ (Pod มี IP
  เปลี่ยนได้ทุกครั้งที่ restart แต่ Service มี IP/DNS name คงที่เสมอ) — Pod อื่นเรียก Service นี้ผ่านชื่อ
  DNS ภายใน cluster ได้เลย (`my-service.my-namespace.svc.cluster.local`)
- **Ingress** — ตัวกำหนดว่า HTTP traffic จาก public internet จะ route เข้า Service ไหนตาม hostname/path
  (ต้องมี Ingress Controller เช่น nginx-ingress/Traefik ติดตั้งแยกใน cluster — ไม่ได้มาให้ในตัวเหมือน
  `[http_service]` ของ Fly.io)
- **ConfigMap/Secret** — เก็บ config/secret แบบเดียวกับที่หัวข้อ 101.8 อธิบาย แต่เป็น object ของ Kubernetes
  เอง ที่ Pod mount เข้ามาเป็น environment variable หรือไฟล์ได้

สังเกตว่าแนวคิดทั้งหมดนี้ **มีคู่เทียบใน Fly.io/ECS อยู่แล้วเกือบทุกตัว** เพียงแต่ Kubernetes แยกเป็น object
ชัดเจนกว่ามาก (ยืดหยุ่นกว่าแต่ก็ซับซ้อนกว่าตามไปด้วย) — นี่คือเหตุผลที่ concept การ deploy ที่เรียนในบทนี้ (
container image มาตรฐาน, health check, environment variable config, rolling deployment) **ถ่ายทอดข้ามไปสู่
Kubernetes ได้ตรง ๆ** ถ้าผู้เรียนต้องเรียน Kubernetes ต่อในอนาคต แค่ต้องเรียนรู้ syntax/object ใหม่ ไม่ต้อง
เรียน concept ใหม่ทั้งหมด

### 101.2 Fly.io และ `flyctl`: แนวคิดของ Fly.io ในฐานะ PaaS ที่ใช้ Container Image มาตรฐาน

**Fly.io ทำงานต่างจาก Heroku แบบเดิมอย่างไร**: Heroku (PaaS รุ่นบุกเบิก) ใช้ "buildpack" ที่ตรวจจับภาษา
โปรแกรมมิ่งจาก source code แล้ว build image ด้วยกลไกเฉพาะของ Heroku เอง — ทำให้ image ที่ได้ **ใช้นอก Heroku
ไม่ได้เลย** (ผูกกับ ecosystem ของ Heroku 100%) — **Fly.io เลือกใช้ container image มาตรฐานของ Docker/OCI**
เป็นหน่วย deploy หลัก หมายความว่า image เดียวกันที่ Part 96 สร้างด้วย `docker build` (ไม่ต้องมี Fly.io-specific
อะไรเลยแม้แต่นิดเดียว) **สามารถรันบน Fly.io ได้ตรง ๆ** และถ้าวันหนึ่งต้องย้ายออกจาก Fly.io ไปแพลตฟอร์มอื่น
(หรือ self-host ด้วย Kubernetes) ก็เอา image เดิมไปรันที่อื่นได้เลยโดยไม่ต้องเขียนอะไรใหม่ — นี่คือเหตุผลที่
Fly.io เหมาะกับหลักสูตรนี้เป็นพิเศษ เพราะทุกเทคนิคที่ Part 96 สอน (multi-stage build, musl static linking,
environment-variable config) **ใช้ได้ตรงกับ Fly.io โดยไม่ต้องปรับอะไรเลย**

**หน่วยการทำงานของ Fly.io — App, Machine, Region**: Fly.io จัดโครงสร้างเป็นสามระดับ: **App** คือหน่วยบนสุด
(เทียบได้กับ "โปรเจกต์" หนึ่งตัว มีชื่อ unique ทั้งระบบ เช่น `library-api-mini`) แต่ละ App มี **Machine**
(เครื่อง virtual ที่เบาที่สุดของ Fly.io เอง สร้างจาก image ของ App นั้น — คล้าย container แต่มี isolation
ระดับ micro-VM ผ่าน Firecracker ตามที่ Part 96 หัวข้อ 96.2 เกริ่นไว้ว่าเป็นเทคโนโลยีลูกผสมระหว่าง container
กับ VM) หนึ่ง App สามารถมีหลาย Machine กระจายอยู่คนละ **Region** (ศูนย์ข้อมูลทั่วโลก เช่น `sin` = สิงคโปร์,
`nrt` = โตเกียว, `iad` = Virginia สหรัฐฯ) เพื่อให้ traffic จากผู้ใช้ในภูมิภาคต่าง ๆ ได้ latency ต่ำ — Fly.io
มี load balancer ระดับ network (Anycast) ที่ route request ไปยัง Machine ที่ใกล้ที่สุดโดยอัตโนมัติ

**`flyctl` คือ CLI หลักในการควบคุมทุกอย่าง**: คำสั่งทุกตัวที่จะเห็นในหัวข้อถัดไป (`fly launch`, `fly deploy`,
`fly secrets set` ฯลฯ) มาจาก binary ชื่อ `flyctl` (บางที่เรียกสั้น ๆ ว่า `fly`) — ตามที่อธิบายไว้ใน "หมายเหตุ
เรื่องการตรวจสอบเนื้อหา" ด้านบน **สภาพแวดล้อมที่เขียนบทนี้ไม่สามารถติดตั้ง `flyctl` ตัวจริงได้** เพราะ network
allowlist ไม่ครอบคลุมโดเมนที่ใช้แจกจ่าย binary นี้ — ในเครื่อง dev ปกติของผู้อ่าน การติดตั้งทำได้ง่ายมากตาม
เอกสารทางการของ Fly.io (`https://fly.io/docs/flyctl/install/`):

```bash
# macOS / Linux — วิธีที่เอกสารทางการแนะนำ (ดาวน์โหลด binary จริงจาก Fly.io CDN)
curl -L https://fly.io/install.sh | sh

# macOS ผ่าน Homebrew
brew install flyctl

# Windows ผ่าน PowerShell
pwsh -Command "iwr https://fly.io/install.ps1 -useb | iex"
```

หลังติดตั้งเสร็จ ตรวจสอบเวอร์ชันด้วย `flyctl version` (คำสั่งนี้เป็น local-only ไม่ต้อง login) — ตามเอกสาร
ทางการ output ควรมีรูปแบบประมาณนี้ (**ตัวเลขเวอร์ชันด้านล่างเป็นตัวอย่างจากเอกสาร ไม่ใช่ output จากการรันจริง
ในสภาพแวดล้อมนี้** เพราะไม่มี binary ให้รัน):

```
flyctl v0.x.xxx linux/amd64 Commit: xxxxxxx BuildDate: 2026-xx-xxTxx:xx:xxZ
```

**ขั้นตอนที่ต้อง login (ต้องมี account จริง — ไม่ได้ทำในบทนี้)**: หลังติดตั้ง ต้องรัน `fly auth login` ซึ่ง
จะเปิด browser ให้ authenticate กับ Fly.io account จริง (หรือ `fly auth signup` ถ้ายังไม่มี account) — คำสั่งนี้
**ต้องมี account จริงและเชื่อมต่อ internet ไปยัง fly.io เท่านั้น ไม่มีทางเลี่ยงแบบ offline ได้เลย** เพราะเป็น
ขั้นตอนยืนยันตัวตนที่ตั้งใจให้ต้องผ่าน server ของ Fly.io จริง — บทนี้จะไม่รันคำสั่งนี้และคำสั่งอื่นใดที่ต้องพึ่ง
การ login เด็ดขาด ตามเงื่อนไขของบทนี้ที่ห้าม authenticate กับ cloud provider จริงใด ๆ

### 101.3 `fly.toml`: อ่านและเขียน Configuration Schema อย่างละเอียด

`fly.toml` คือไฟล์ configuration หลักของแต่ละ App บน Fly.io — เขียนด้วย [TOML](https://toml.io) (ภาษา
configuration ที่ Rust เองก็ใช้เป็นฟอร์แมตของ `Cargo.toml` ตามที่ Part 2/17 สอนไว้แล้ว ทำให้ syntax นี้คุ้นเคย
อยู่แล้วโดยไม่ต้องเรียนรู้ภาษาใหม่) — ต่างจาก `Dockerfile` ที่บอกวิธี **สร้าง** image, `fly.toml` บอกวิธี
**รัน** image นั้นบน Fly.io: จะ expose port ไหน, ตั้ง environment variable อะไรบ้าง (ที่ไม่ใช่ความลับ), จะ
health check อย่างไร, จะ scale กี่ Machine ฯลฯ

มาดู `fly.toml` ตัวอย่างที่เขียนสำหรับ capstone `library_api_mini` ของ Part 92-96 ทีละ section:

```toml
# fly.toml — สำหรับ deploy library_api_mini (capstone จาก Part 92-96) ขึ้น Fly.io
app = "library-api-mini"
primary_region = "sin"
kill_signal = "SIGINT"
kill_timeout = "5s"

[build]
  dockerfile = "Dockerfile.musl"

[env]
  RUST_LOG = "info"
  BIND_ADDR = "0.0.0.0:8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 1
  processes = ["app"]

  [[http_service.checks]]
    grace_period = "10s"
    interval = "15s"
    method = "GET"
    timeout = "3s"
    path = "/health"

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
```

(ไฟล์นี้ถูกตรวจสอบ **syntax TOML จริง** ด้วย Python `tomllib` — parser มาตรฐานของ TOML — ในสภาพแวดล้อมที่
เขียนบทนี้ ยืนยันว่า parse ผ่านโดยไม่มี syntax error ก่อนนำมาแสดงในบทนี้ แต่ **ความถูกต้องของ field/value
ตาม schema ของ Fly.io เอง** (เช่น ค่าที่ `size`/`auto_stop_machines` รับได้จริงมีอะไรบ้าง) มาจากเอกสารทางการ
ของ Fly.io เพราะไม่มี `flyctl` ตัวจริงให้รัน `fly config validate` ยืนยันในสภาพแวดล้อมนี้)

**อธิบายทีละ field**:

- **`app = "library-api-mini"`** — ชื่อ App ที่ unique ทั้งระบบ Fly.io (เหมือน username บน platform ที่ต้อง
  ไม่ซ้ำกับใคร) ชื่อนี้กลายเป็นส่วนหนึ่งของ default hostname ด้วย (`library-api-mini.fly.dev`) จนกว่าจะผูก
  custom domain (หัวข้อ 101.11)
- **`primary_region = "sin"`** — region หลักที่ Machine แรกจะถูกสร้าง (`sin` = สิงคโปร์ ใกล้ผู้ใช้ในเอเชีย
  ตะวันออกเฉียงใต้/ไทยที่สุดในรายการ region ของ Fly.io) — เลือก region ให้ใกล้ผู้ใช้ส่วนใหญ่หรือใกล้ managed
  database ที่แอปต่ออยู่มากที่สุด (latency ระหว่าง app กับ database มีผลต่อ response time ของทุก request ที่
  ต้อง query — ยิ่งใกล้กันยิ่งเร็ว)
- **`kill_signal = "SIGINT"` / `kill_timeout = "5s"`** — ตอน Fly.io ต้องหยุด Machine (deploy ใหม่, scale ลง,
  หรือ auto-stop ตาม `auto_stop_machines`) มันจะส่ง signal นี้ไปให้ process หลักก่อน (ให้เวลา cleanup เช่น
  ปิด database connection pool ให้เรียบร้อยตามที่ SQLx ทำอัตโนมัติตอน `Pool` ถูก drop) แล้วรอไม่เกิน
  `kill_timeout` ก่อนจะส่ง `SIGKILL` บังคับปิดถ้า process ยังไม่ยอมหยุด — ตรงกับหลัก graceful shutdown ที่ทุก
  service ควรรองรับ (axum เองจัดการ `SIGINT` ได้ผ่าน `tokio::signal::ctrl_c()` ถ้าตั้งค่า graceful shutdown
  handler ไว้)
- **`[build]` / `dockerfile = "Dockerfile.musl"`** — บอก Fly.io ว่าจะ build image จาก `Dockerfile.musl`
  (ไฟล์เดียวกับที่ Part 96 หัวข้อ 96.10 เขียนไว้สำหรับ capstone — musl + `scratch`) ที่อยู่ใน build context
  เดียวกับ `fly.toml` — Fly.io จะรัน `docker build` เทียบเท่ากันบนเครื่อง remote builder ของเขาเอง (หรือ
  local Docker daemon ถ้ามี ผ่าน flag `--local-only`) แล้ว push image ที่ได้ขึ้น registry ภายในของ Fly.io เอง
  โดยอัตโนมัติ ผู้ใช้ไม่ต้องจัดการ registry เอง — ถ้าไม่ระบุ `[build]` เลย Fly.io จะมองหา `Dockerfile` (ชื่อ
  default) ในโฟลเดอร์เดียวกันแทน
- **`[env]`** — environment variable ที่ **ไม่ใช่ความลับ** จะถูกฝังใน image config ตรง ๆ (คล้ายกับ `ENV` ใน
  Dockerfile แต่ตั้งแยกจากตัว image ทำให้เปลี่ยนได้โดยไม่ต้อง build ใหม่) — ในตัวอย่างนี้ `RUST_LOG` และ
  `BIND_ADDR` ไม่เป็นความลับเลย เหมาะกับ section นี้ตรงตามหลักการของ Part 96 หัวข้อ 96.8 ที่แยก **config
  ทั่วไป** (ใส่ `[env]` ได้) ออกจาก **secret** (`DATABASE_URL`/`JWT_SECRET` ที่ต้องใช้ `fly secrets set` แทน
  ตามหัวข้อ 101.8 — **ห้ามใส่ secret ใน `[env]` ของ `fly.toml` เด็ดขาด เพราะไฟล์นี้มักถูก commit เข้า git**)
- **`[http_service]`** — section ที่บอกว่า App นี้เป็น HTTP service (มี HTTP traffic เข้ามาจาก public
  internet) พร้อมพฤติกรรมที่เกี่ยวข้อง:
  - **`internal_port = 8080`** — port ที่ process **ภายใน** container ฟังอยู่จริง (ต้องตรงกับ `BIND_ADDR`
    ที่ Dockerfile/`[env]` ตั้งไว้ — ถ้าไม่ตรงกัน health check จะ fail ด้วย connection refused ทันที ตามที่
    กับดักที่พบบ่อยข้อ 4 ของบทนี้จะอธิบาย) — **นี่คนละเรื่องกับ `EXPOSE` ใน Dockerfile** ที่ Part 96 หัวข้อ
    96 กับดักข้อ 7 บอกไว้ว่าเป็นแค่ documentation — `internal_port` ของ Fly.io นี้มีผลจริงต่อการ route
    traffic เข้า container
  - **`force_https = true`** — บังคับ redirect ทุก HTTP request เป็น HTTPS อัตโนมัติที่ระดับ Fly.io proxy
    ก่อนถึงแอปเลย (แอปไม่ต้องรู้เรื่อง TLS อะไรเลย — Fly.io จัดการ TLS termination ให้ทั้งหมด ตามที่หัวข้อ
    101.11 จะอธิบายลึกกว่านี้)
  - **`auto_stop_machines = "stop"` / `auto_start_machines = true`** — ถ้าไม่มี traffic เข้ามาเลยช่วงหนึ่ง
    Fly.io จะ **หยุด** Machine ให้อัตโนมัติ (ประหยัดค่าใช้จ่าย เพราะ Machine ที่หยุดอยู่คิดเงินน้อยกว่ามาก —
    ดูหัวข้อ 101.12) แล้วพอมี request ใหม่เข้ามา จะ **start** Machine ขึ้นมาใหม่อัตโนมัติ (มี latency
    เพิ่มขึ้นเล็กน้อยตอน "cold start" ครั้งแรกหลังหยุด แต่ Machine ของ Fly.io เบามาก start เร็วกว่า VM ปกติ
    มาก) — เหมาะกับแอปที่ traffic ไม่สม่ำเสมอ ถ้าต้องการให้พร้อมรับ traffic ตลอดเวลาไม่มี cold start เลย
    ตั้ง `auto_stop_machines = "off"` แทน (แต่จะเสียค่าใช้จ่ายเต็มเวลาที่ Machine รันอยู่)
  - **`min_machines_running = 1`** — จำนวน Machine ต่ำสุดที่ต้องมีรันอยู่เสมอแม้ไม่มี traffic (ตั้งเป็น 0
    ได้ถ้ายอมรับ cold start ทุกครั้งที่ไม่มี traffic นานพอ — ตั้งเป็น 1 ขึ้นไปถ้าต้องการ Machine หนึ่งตัวพร้อม
    เสมอเพื่อไม่ให้ request แรกช้าเพราะต้อง cold start)
  - **`[[http_service.checks]]`** — HTTP health check ที่ Fly.io proxy จะยิงไปที่ Machine เป็นระยะ (ทุก
    `interval`) เพื่อรู้ว่า Machine นั้น**พร้อมรับ traffic จริงหรือไม่** — `path = "/health"` คือ endpoint
    เดียวกับที่ Part 94 หัวข้อ 94.7 implement ไว้แล้ว (คืน `{"database":"ok","status":"ok"}` ตามที่ Part 96
    หัวข้อ 96.10 พิสูจน์ไว้จริง) — `grace_period = "10s"` ให้เวลา Machine ที่เพิ่ง start ไม่ต้องผ่าน health
    check ทันที (รอ 10 วินาทีแรกก่อนเริ่มเช็ค เผื่อแอปยัง initialize ไม่เสร็จ เช่นกำลังรัน migration) —
    ความสัมพันธ์ระหว่าง health check นี้กับ zero-downtime deployment คือหัวใจของหัวข้อ 101.10
- **`[[vm]]`** — ขนาดของ Machine ที่จะสร้าง: `size = "shared-cpu-1x"` คือ tier เล็กที่สุดของ Fly.io (แชร์
  CPU core กับ Machine อื่นบนเครื่องเดียวกัน ไม่ได้ dedicated เต็ม core) พร้อม `memory = "512mb"` — เพียงพอ
  สำหรับ axum service ขนาดเล็กแบบ capstone ของหลักสูตรนี้ (binary เพียง ~7.84MB ตามที่ Part 96 วัดไว้ ใช้
  memory ตอนรันจริงน้อยมากเพราะไม่มี Rust runtime ที่หนักแบบภาษาที่มี garbage collector) — เขียน `[[vm]]`
  แบบ array-of-table (สังเกต `[[ ]]` สองชั้น) เพราะ Fly.io รองรับการตั้ง process หลายกลุ่มให้ VM ขนาดต่างกันได้
  ถ้าแอปมีหลาย process (เช่น web process กับ background worker process แยกกัน)

**Field เพิ่มเติมที่ควรรู้จัก แม้ capstone ของบทนี้ยังไม่ต้องใช้ทั้งหมด** (อ้างอิงจากเอกสาร Fly.io):

- **`[http_service.concurrency]`** — ควบคุมว่าแต่ละ Machine รับ connection พร้อมกันได้กี่ตัวก่อนที่ Fly.io
  proxy จะเริ่ม route ไปที่ Machine อื่นแทน (หรือ start Machine ใหม่ถ้า `auto_start_machines = true`):

  ```toml
  [http_service.concurrency]
    type = "connections"   # หรือ "requests"
    hard_limit = 25
    soft_limit = 20
  ```

  `soft_limit` คือจุดที่ proxy **เริ่ม**พิจารณาส่ง traffic ไปที่ Machine อื่น (ถ้ามี) ส่วน `hard_limit` คือ
  เพดานที่ Machine นั้น**ปฏิเสธ**ไม่รับ connection ใหม่เพิ่มเลย — field นี้สำคัญมากสำหรับแอปที่ใช้
  `tokio`/async runtime อย่าง axum เพราะ connection หนึ่งไม่ได้แปลว่า thread หนึ่ง (axum จัดการ concurrent
  connection ได้เยอะมากด้วย async I/O ตามที่ Part 30+ สอน) แต่ก็ยังมีเพดานจริงที่ database connection pool
  (ขนาดที่ Part 70 ตั้งไว้ตอนสร้าง `PgPool`) จะรับได้ — ตั้ง `hard_limit` ให้สัมพันธ์กับขนาด connection pool
  จริงของแอป ไม่ใช่ตั้งสูงลอย ๆ โดยไม่คิดถึงข้อจำกัดของ database ด้านหลัง
- **`[[mounts]]`** — ผูก persistent volume เข้ากับ Machine (สำหรับข้อมูลที่ต้องอยู่ถาวรข้าม deployment เช่น
  ไฟล์ที่ผู้ใช้อัปโหลด) — **capstone ของบทนี้ไม่ต้องใช้เลย** เพราะข้อมูลทั้งหมดอยู่ใน managed PostgreSQL
  (ตามหัวข้อ 101.9) ซึ่งเป็นแนวทางที่แนะนำอยู่แล้ว (เก็บ state ไว้ที่ database แยกออกจาก Machine ที่รันแอป
  ทำให้ Machine เป็น "stateless" — ทดแทน/ย้าย region ได้ทุกเมื่อโดยไม่เสียข้อมูล ตรงกับหลัก twelve-factor
  app ข้อที่ว่า process ควร stateless) แต่ถ้าแอปจำเป็นต้องเก็บไฟล์ไว้ในเครื่องจริง ๆ (เช่น cache ขนาดใหญ่ที่
  ไม่ควรโหลดใหม่ทุกครั้ง) จะเขียนแบบนี้:

  ```toml
  [[mounts]]
    source = "library_mini_data"
    destination = "/data"
  ```

  ข้อจำกัดสำคัญที่ต้องรู้: volume แบบนี้ **ผูกกับ Machine ตัวเดียวในตำแหน่งเดียว** ไม่ใช่ shared storage
  ข้าม Machine หลายตัว (ต่างจาก S3/Cloud Storage ที่หลาย instance เข้าถึงพร้อมกันได้) — ถ้าต้องการ scale
  หลาย Machine พร้อม shared file storage ต้องใช้ object storage (S3-compatible) แทน ไม่ใช่ `[[mounts]]`
- **`[processes]`** (top-level table, คนละอันกับ `processes` ใน `[http_service]`) — ถ้าโปรเจกต์เป็น
  workspace ที่มีหลาย binary (ตามที่ Part 17/96 หัวข้อ 96.7 สอนเรื่อง workspace หลาย crate) กำหนดได้ว่า
  process group ไหนรันคำสั่งอะไร เช่น:

  ```toml
  [processes]
    app = "./library_api_mini"
    worker = "./library_api_mini --mode=background-worker"
  ```

  แล้วอ้างชื่อ group (`app`) กลับใน `[http_service] processes = ["app"]` (ตามที่ตัวอย่างหัวข้อนี้ตั้งไว้แล้ว)
  — ทำให้ Fly.io รู้ว่า process group ไหนควรรับ HTTP traffic (มี `[[http_service.checks]]`) และ process group
  ไหนไม่ควร (เช่น `worker` ที่ประมวลผล background job อย่างเดียว ไม่ต้องมี health check HTTP เลย)

**Database Migration ตอน Deploy — `[deploy] release_command` เทียบกับการรัน migration ใน process ตอน
start**: capstone `library_api_mini` ของ Part 96 หัวข้อ 96.10 เลือกรัน migration ผ่าน `sqlx::migrate!`
**ใน process หลักตอน start ทุกครั้ง** (log ยืนยันจริงว่า "รัน migration สำเร็จ" ปรากฏก่อน "library_api_mini
ฟังอยู่ที่ 0.0.0.0:8080") — วิธีนี้ใช้งานได้ดีเพราะ `sqlx::migrate!` ออกแบบมาให้ **idempotent** (รันซ้ำกี่ครั้ง
ก็ปลอดภัย เพราะเช็ค migration ที่รันไปแล้วจากตาราง `_sqlx_migrations` ก่อนเสมอ ไม่รัน migration เดิมซ้ำ) แต่
เมื่อ scale เป็นหลาย Machine พร้อมกัน (ตามหัวข้อ 101.10) **ทุก Machine ที่ start จะพยายามรัน migration พร้อม
กันได้ในทางทฤษฎี** ซึ่ง `sqlx::migrate!` จัดการด้วย advisory lock ของ PostgreSQL เองให้ปลอดภัย (Machine อื่น
จะรอ Machine แรกที่ได้ lock ทำ migration เสร็จก่อน) แต่ก็ยังเพิ่มความซับซ้อนเล็กน้อยที่ไม่จำเป็นถ้าไม่ต้องการ
ให้ทุก Machine มีหน้าที่นี้

Fly.io มีทางเลือกอื่นผ่าน **`[deploy] release_command`** — คำสั่งที่ Fly.io รันแค่ **ครั้งเดียว** บน Machine
ชั่วคราวพิเศษ **ก่อน**ที่จะเริ่ม deploy เวอร์ชันใหม่ไปที่ Machine จริงเลย (ถ้า `release_command` fail
deployment ทั้งหมดจะถูกยกเลิกทันที ไม่มี Machine ใหม่ตัวไหนถูกสร้างขึ้นมาเลย — เป็นอีกระดับของ
health-check-gated cutover ที่หัวข้อ 101.10 อธิบาย แต่เช็คก่อนแม้แต่จะสร้าง Machine ใหม่ด้วยซ้ำ):

```toml
[deploy]
  release_command = "/library_api_mini --migrate-only"
```

(ตัวอย่างนี้ต้องเพิ่ม flag `--migrate-only` เข้าไปใน binary จริงเอง ให้รัน migration แล้ว exit ทันทีโดยไม่
เปิด HTTP listener — เป็นการปรับโค้ดแอปเพิ่มเติมที่ Part 96/94 ยังไม่ได้ทำไว้ จึงไม่ใช่ส่วนที่ capstone ของ
บทนี้ใช้จริง แต่เป็นแนวทางที่ควรรู้จักไว้สำหรับโปรเจกต์ที่ต้องการแยก "งาน migration" ออกจาก "งานรับ traffic"
อย่างเด็ดขาด) — **ข้อดี**: migration รันแค่ครั้งเดียวจริง ๆ ไม่ต้องพึ่ง advisory lock ของ PostgreSQL มาช่วย
กันชนกัน, และถ้า migration fail deployment จะไม่ไปถึงขั้นสร้าง Machine ใหม่เลยด้วยซ้ำ (ปลอดภัยกว่าการให้
Machine ใหม่ start ไม่สำเร็จเพราะ migration fail แล้วเข้า restart loop ตามกับดักข้อ 3) — **ข้อเสีย**: ต้องเขียน
โค้ดเพิ่ม (flag แยกสำหรับโหมด migrate-only) และเพิ่มความซับซ้อนของ binary หนึ่งตัวให้ทำได้สองโหมด — สำหรับ
โปรเจกต์เล็ก/capstone ระดับหลักสูตรนี้ วิธีเดิม (migrate ใน process ตอน start) ยังคงเป็นทางเลือกที่เรียบง่าย
พอและปลอดภัยเพียงพอ แต่โปรเจกต์ที่ scale ใหญ่ขึ้นมากในอนาคตควรพิจารณาย้ายไปทาง `release_command`

### 101.4 คำสั่ง `flyctl` หลักที่ต้องรู้ (อ้างอิงจากเอกสารทางการ)

หัวข้อนี้สรุป syntax ของคำสั่งที่ใช้บ่อยที่สุด — **ทุกคำสั่งในหัวข้อนี้อ้างอิงจากเอกสารทางการของ Fly.io
(`fly.io/docs/flyctl/`) เท่านั้น ไม่ได้รันจริงในสภาพแวดล้อมนี้** ตามที่อธิบายไว้ในหมายเหตุต้นบท

**`fly launch`** — คำสั่งแรกที่ใช้ตอนเริ่ม App ใหม่ ต้องรันในโฟลเดอร์ที่มี `Dockerfile` (หรือ source code ที่
Fly.io ตรวจจับภาษาได้อัตโนมัติผ่าน buildpack ของเขาเองถ้าไม่มี Dockerfile) มันจะ**ถามคำถามแบบ interactive**
(ชื่อ App, region, ต้องการ Postgres/Redis แนบมาด้วยไหม) แล้วสร้าง `fly.toml` ตัวแรกให้อัตโนมัติจากคำตอบ พร้อม
สร้าง App บน Fly.io จริง (ขั้นตอนนี้ **ต้อง login แล้ว** และสร้างทรัพยากรจริงบน account จริง) — ถ้ามี
`fly.toml` อยู่แล้ว (แบบที่หัวข้อ 101.3 เขียนไว้) ใช้ `fly launch --no-deploy` เพื่อสร้าง App ตาม config ที่
เขียนไว้แล้วโดยไม่ deploy ทันที หรือข้ามไปใช้ `fly apps create <name>` ตรง ๆ ถ้าต้องการควบคุมทุก field ของ
`fly.toml` เองตั้งแต่ต้นแบบที่หัวข้อ 101.3 ทำ

**`fly deploy`** — คำสั่งหลักที่ใช้ทุกครั้งที่ต้องการ deploy โค้ดใหม่ (หลัง `git push`/CI build เสร็จ หรือแก้
`fly.toml`) มันจะ: (1) build image ตาม `[build]` section, (2) push image ขึ้น internal registry ของ Fly.io,
(3) สร้าง Machine ใหม่จาก image ใหม่, (4) รอให้ Machine ใหม่ผ่าน health check ตาม `[[http_service.checks]]`,
(5) ค่อย route traffic มาที่ Machine ใหม่แล้วปิด Machine เก่า (นี่คือกลไก rolling deployment ที่หัวข้อ 101.10
อธิบายลึก) — flag ที่ควรรู้: `--strategy rolling` (default อยู่แล้ว, ทีละ Machine), `--strategy immediate`
(เปลี่ยนทุก Machine พร้อมกัน เร็วกว่าแต่มี downtime ช่วงสั้น ๆ ถ้า image ใหม่มีปัญหา), `--local-only` (build
ด้วย Docker daemon ในเครื่องผู้ใช้เองแทนที่จะส่งไป remote builder ของ Fly.io — มีประโยชน์ถ้า network ไป Fly.io
builder ช้าหรือมี Docker layer cache ในเครื่องอยู่แล้ว)

**`fly status`** — แสดงสถานะของทุก Machine ใน App (กำลังรัน, กำลัง deploy, health check ผ่านหรือไม่) — คำสั่ง
แรกที่ควรรันเมื่อสงสัยว่า deploy สำเร็จจริงหรือไม่

**`fly logs`** — stream log จาก Machine ที่กำลังรันอยู่แบบ real-time (เทียบเท่า `docker logs -f` ของ Part 96
แต่สำหรับ Machine บนคลาวด์) มีประโยชน์มากตอน debug ปัญหาหลัง deploy

**`fly secrets set KEY=value`** — ตั้งค่า secret (จะอธิบายลึกในหัวข้อ 101.8) syntax คือ `fly secrets set
DATABASE_URL="postgres://..." JWT_SECRET="..."` (ตั้งได้หลายตัวในคำสั่งเดียว คั่นด้วย space) — **การตั้ง
secret ใหม่จะ trigger deployment ใหม่โดยอัตโนมัติ** เพื่อให้ Machine ที่รันอยู่ได้ environment variable ใหม่
(เพราะ Machine ที่รันอยู่แล้วจะไม่เห็น secret ที่ตั้งทีหลังจนกว่าจะ restart) — ดู secret ที่ตั้งไว้ (ชื่อ
เท่านั้น ไม่แสดงค่า เพื่อความปลอดภัย) ด้วย `fly secrets list`

**`fly scale count <N>`** — ปรับจำนวน Machine ทั้งหมดของ App เป็น N ตัว (กระจายตาม region ที่ตั้งไว้) —
ต่างจาก `min_machines_running` ใน `fly.toml` ที่เป็นแค่ "ขั้นต่ำ" ตอนไม่มี traffic คำสั่งนี้ตั้งจำนวนที่
ต้องการโดยตรง

**`fly ssh console`** — เปิด shell เข้าไปใน Machine ที่รันอยู่จริงโดยตรง (มีประโยชน์ตอน debug) — **ข้อควร
ระวังที่ต่อเนื่องจาก Part 96 หัวข้อ 96.5**: ถ้า runtime image เป็น `scratch` (แบบ capstone ของบทนี้ที่ไม่มี
shell อยู่ข้างในเลย) คำสั่งนี้จะใช้ไม่ได้ — ต้อง `fly ssh console` เข้าเครื่อง host ที่รัน Machine นั้นแทน
(Fly.io มี debug image พิเศษให้ผ่าน flag เพิ่มเติม) หรือพึ่ง `fly logs` เป็นหลักแทนสำหรับ image ที่ไม่มี shell

**`fly machine list` / `fly machine status <id>`** — ละเอียดกว่า `fly status` ตรงที่แสดงข้อมูลระดับ Machine
เดี่ยว ๆ (region, ขนาด CPU/memory จริงที่รันอยู่, เวลาที่ start) มีประโยชน์เมื่อ App มีหลาย Machine กระจาย
หลาย region และต้องการรู้ว่า Machine ตัวใดตัวหนึ่งเจาะจงมีปัญหาหรือไม่

**Deployment strategy — ทางเลือกที่มากกว่า `rolling` แบบ default**: `fly deploy --strategy <ชื่อ>` รองรับ
หลายกลยุทธ์ที่ให้ trade-off ต่างกันระหว่างความเร็วกับความปลอดภัยของการ cutover (ตามเอกสาร Fly.io):

- **`rolling`** (default) — ทดแทน Machine ทีละตัวตามที่หัวข้อ 101.10 อธิบาย ปลอดภัยที่สุดแต่ใช้เวลานานสุด
  ถ้ามี Machine หลายตัว
- **`immediate`** — เปลี่ยนทุก Machine พร้อมกันทันที ไม่รอ health check ก่อน เร็วที่สุดแต่เสี่ยงมีช่วง
  downtime สั้น ๆ ถ้า image ใหม่มีปัญหา (เหมาะกับ dev/staging environment ที่ downtime สั้น ๆ ไม่กระทบใคร
  ไม่ใช่ production ที่มีผู้ใช้จริง)
- **`bluegreen`** — สร้าง Machine ชุดใหม่ **ทั้งหมด** ขึ้นมาคู่กับชุดเก่า (ไม่ได้ทดแทนทีละตัวแบบ rolling)
  รอให้ชุดใหม่ผ่าน health check ครบทุกตัวก่อน แล้วสลับ traffic ทั้งหมดไปที่ชุดใหม่ในทีเดียว พร้อมคง Machine
  ชุดเก่าไว้ช่วงหนึ่งเผื่อต้อง rollback ทันที (เร็วกว่า rollback ผ่าน `fly deploy` เวอร์ชันก่อนหน้าใหม่ทั้ง
  หมด) — ใช้ resource มากกว่า rolling ชั่วคราว (มี Machine สองชุดอยู่พร้อมกันช่วงสั้น ๆ) แต่ปลอดภัยกว่าและ
  rollback ไวกว่า เหมาะกับ production ที่ critical มาก
- **`canary`** — ทดสอบ Machine ใหม่แค่**บางส่วน**ก่อน (รับ traffic เปอร์เซ็นต์น้อย ๆ) แล้วค่อยขยายทีละขั้น
  ถ้าไม่มีปัญหา — ให้ความปลอดภัยสูงสุดสำหรับการเปลี่ยนแปลงที่มีความเสี่ยง แลกกับเวลาที่ deploy เต็มรูปแบบใช้
  นานที่สุดในทุกกลยุทธ์

**ผูกกับ CI/CD (ต่อยอดจาก Part 97)**: Part 97 สอนให้ GitHub Actions build/test/push container image
อัตโนมัติทุกครั้งที่ push — ขั้นตอนที่ยังไม่ได้ตอบคือ "แล้วใครสั่ง deploy image ใหม่นั้นไปที่ Fly.io" คำตอบคือ
เพิ่ม step ท้าย workflow ที่เรียก `flyctl deploy` โดยใช้ Fly.io API token ที่เก็บเป็น GitHub Actions secret
(สร้างผ่าน `fly tokens create deploy` ตามเอกสารทางการ แล้วเก็บด้วยชื่อ `FLY_API_TOKEN`) ตัวอย่าง workflow
(YAML นี้ตรวจสอบ **syntax จริง** ด้วย `PyYAML` — parser มาตรฐานสำหรับ YAML — ในสภาพแวดล้อมที่เขียนบทนี้
ส่วนพฤติกรรมจริงของ `superfly/flyctl-actions` เป็นข้อมูลอ้างอิงจากเอกสารของ Fly.io เพราะไม่มี account จริง
ให้ทดสอบ):

```yaml
name: Deploy to Fly.io

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    concurrency: deploy-group
    steps:
      - uses: actions/checkout@v4
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
```

> **ข้อสังเกตที่พบจริงตอนตรวจสอบ syntax ของไฟล์นี้**: เมื่อ parse ด้วย `PyYAML` (`yaml.safe_load`) key
> `on:` (ที่บรรทัดสองของไฟล์) **ถูกแปลงเป็น boolean `true` แทนที่จะเป็น string `"on"`** ตามที่คาดหวัง! นี่ไม่
> ใช่บั๊กของไฟล์ — เป็นพฤติกรรมของ **YAML 1.1 spec เอง** ที่กำหนดให้คำอย่าง `on`/`off`/`yes`/`no`/`true`/
> `false` (แม้ไม่ได้ใส่ quote) ถูกตีความเป็น boolean เสมอ ซึ่งเป็นเรื่องที่ชุมชน GitHub Actions รู้จักกันดีใน
> ชื่อ "the Norway problem" (เพราะ `NO` ซึ่งเป็น ISO code ของประเทศนอร์เวย์ก็โดนแปลงเป็น `false` ไปด้วยถ้าไม่
> ใส่ quote ในบริบทอื่น) — **GitHub Actions เองมี parser พิเศษที่รู้จัก key `on` ของ workflow file
> โดยเฉพาะและตีความเป็น string เสมอ ไม่ตกกับดักนี้** แต่ถ้าเขียนเครื่องมือ/script ของทีมเองที่ใช้ YAML parser
> มาตรฐานตรวจสอบ workflow file (เช่นตอนเขียน test เอง) ต้องรู้ตัวว่าอาจเจอพฤติกรรมนี้ และเลือกใส่ quote
> (`"on":`) หรือใช้ YAML 1.2 parser ที่ไม่มีปัญหานี้ถ้าต้องการผลลัพธ์ที่ตรงไปตรงมากว่า

### 101.5 ตัวเลือก Deployment ของ AWS: ECS/Fargate, EC2, และ Lambda — เปรียบเทียบ

AWS มีตัวเลือกในการรัน container/binary จำนวนมากกว่า Fly.io มาก เพราะ AWS เป็น cloud provider ที่ครอบคลุม
ทุกระดับของ infrastructure (ตั้งแต่ VM ดิบไปจนถึง managed service เฉพาะทาง) — หัวข้อนี้เปรียบเทียบสามตัวเลือก
หลักที่เกี่ยวกับการรันแอป Rust แบบ web service/function:

| | **EC2** (VM ดิบ) | **ECS/Fargate** (Container) | **Lambda** (Function-as-a-Service) |
|---|---|---|---|
| หน่วยที่ deploy | VM instance เต็มตัว | Container image (Task/Service) | ZIP หรือ container image ของ function เดียว |
| ต้องจัดการ OS เองไหม | ต้อง (patch, security update) | Fargate ไม่ต้อง (serverless container), EC2 launch type ต้อง | ไม่ต้องเลย (AWS จัดการทั้งหมด) |
| Scale อัตโนมัติ | ต้องตั้ง Auto Scaling Group เอง | มี (ECS Service Auto Scaling) | มีในตัว 100% (scale เป็นศูนย์ได้ตอนไม่มี traffic) |
| จ่ายเงินตอนไม่มี traffic ไหม | จ่าย (เครื่องรันตลอดไม่ว่ามี traffic หรือไม่) | Fargate จ่ายตาม task ที่รันอยู่ (ตั้ง 0 task ได้ถ้าไม่ต้องการ) | ไม่จ่ายเลยถ้าไม่มี invocation (billing ตาม invocation+เวลาที่ใช้จริง) |
| เหมาะกับ workload แบบไหน | ต้องการควบคุมเต็มที่, workload คงที่ 24/7 | Web service ที่ต้องการ container standard, traffic สม่ำเสมอปานกลาง-สูง | Event-driven, traffic ไม่สม่ำเสมอมาก, งานที่ทำเสร็จเร็ว (นาทีเดียวหรือน้อยกว่า) |
| Cold start | ไม่มี (เครื่องรันตลอด) | น้อย (task ที่ scale ขึ้นใหม่ใช้เวลาหลักวินาที) | มีจริง โดยเฉพาะครั้งแรกหลัง idle (หัวข้อ 101.6 อธิบายลึก) |
| ใช้ image เดียวกับ Part 96 ได้ตรงไหม | ได้ (รัน `docker run` บน EC2 ตรง ๆ) | ได้ตรง 100% (ECS ใช้ container image มาตรฐาน) | ต้องปรับ (ดูหัวข้อ 101.6 — Lambda ต้องการ runtime interface พิเศษ) |

**ECS (Elastic Container Service) กับ Fargate คืออะไรกันแน่**: ECS คือ **ตัวจัดการ (orchestrator)** ของ AWS
สำหรับรัน container (คล้าย Kubernetes แต่เป็นระบบของ AWS เอง ง่ายกว่า K8s แต่ผูกกับ AWS 100% ต่างจาก K8s ที่
portable ข้าม cloud ได้) — ECS มีสอง "launch type": **EC2 launch type** (ECS จัดการ container แต่ยังต้องมี
EC2 instance ของคุณเองเป็นที่รัน — ต้องจัดการ patch OS ของ EC2 เอง) และ **Fargate launch type** (AWS จัดการ
เครื่องที่รันให้ทั้งหมด คุณส่งแค่ container image กับจำนวน CPU/memory ที่ต้องการ ไม่เห็น EC2 instance เลย —
นี่คือ "serverless container" ที่คนส่วนใหญ่หมายถึงเมื่อพูดถึง ECS ในปัจจุบัน) — สำหรับแอป web service แบบ
capstone ของหลักสูตรนี้ **Fargate คือตัวเลือกที่ใกล้เคียงกับ Fly.io ที่สุดในฝั่ง AWS**: รับ container image
มาตรฐานตรง ๆ, ไม่ต้องจัดการ OS, scale ตาม metric ได้ — ต่างกันหลักที่ ECS/Fargate **ไม่มี PaaS-level
convenience** แบบ Fly.io ให้ (ต้องตั้ง VPC, Application Load Balancer, Target Group, Security Group เองผ่าน
AWS Console/Terraform/CDK — ซับซ้อนกว่า `fly deploy` คำสั่งเดียวมาก แต่ควบคุมได้ละเอียดกว่าและ integrate กับ
บริการอื่นของ AWS ได้แน่นกว่า เช่น IAM role-based access ไป S3/DynamoDB โดยไม่ต้องมี credential แยก)

**EC2 ดิบ ๆ**: เหมือน VM ทั่วไปตามที่หัวข้อ 101.1 อธิบาย — เหมาะกับกรณีที่ต้องการควบคุมทุกอย่างเอง หรือ
workload ที่คงที่มากจนการใช้ Reserved Instance/Savings Plan (จ่ายล่วงหน้าแลกราคาถูกกว่า on-demand มาก) คุ้ม
กว่า Fargate ในระยะยาว

**ตัวเลือกที่ใกล้เคียง Fly.io มากกว่า ECS/Fargate ดิบ — AWS App Runner และ Lightsail Containers**: AWS เอง
รู้ตัวว่า ECS/Fargate ดิบมี operational overhead สูงกว่า PaaS แบบ Fly.io มาก จึงมีบริการเสริมสองตัวที่ **ลด**
ความซับซ้อนนั้นลงโดยยังใช้ container image มาตรฐานอยู่:

- **AWS App Runner** — รับ container image (หรือ source code ที่ build ให้อัตโนมัติ) แล้วจัดการ Load
  Balancer/Auto Scaling/HTTPS certificate ให้ **ทั้งหมดในคำสั่งเดียว** ใกล้เคียงความเรียบง่ายของ `fly deploy`
  มากที่สุดในฝั่ง AWS — เหมาะกับทีมที่ต้องการอยู่ใน AWS ecosystem (integrate กับ IAM/VPC/CloudWatch ของ AWS
  แน่นกว่า Fly.io) แต่ไม่ต้องการความซับซ้อนของการตั้ง ECS/ALB/Target Group เอง
- **Amazon Lightsail Containers** — ง่ายกว่า App Runner อีกขั้น (ตัดสินใจเรื่อง scaling/network ให้น้อยลง
  ไปอีก) เหมาะกับโปรเจกต์เล็กมาก/demo ที่ต้องการความเรียบง่ายสูงสุดในฝั่ง AWS แลกกับความยืดหยุ่นที่น้อยกว่า
  App Runner/ECS

ทั้งสองตัวนี้แสดงให้เห็นว่า **"container image มาตรฐาน" กับ "ความเรียบง่ายระดับ PaaS" ไม่ใช่สิ่งที่แยกจากกัน
เสมอไป** — AWS/GCP ก็มีชั้น PaaS ของตัวเองที่ใช้ container image เดียวกับที่ deploy บน Fly.io/ECS/Kubernetes
ได้ ต่างกันแค่ว่าอยู่ใน ecosystem ไหนและควบคุมรายละเอียดได้มาก-น้อยแค่ไหน

### 101.6 AWS Lambda สำหรับ Rust: `cargo lambda` — ตรวจสอบจริงแบบ Local 100%

Lambda คือบริการ **Function-as-a-Service (FaaS)** ของ AWS — อัปโหลด code (หรือ container image) ของ
function เดียว แล้ว AWS จัดการทุกอย่างที่เหลือ: จะรัน instance กี่ตัว, scale อย่างไรตาม traffic, ปิด
instance ตอนไม่มีงานเข้า (ไม่มีค่าใช้จ่ายเลยตอนนั้น) — โมเดลนี้ต่างจาก ECS/EC2/Fly.io ตรงที่ **คุณไม่ควบคุม
process ที่รันอยู่ตลอดเวลาเหมือน web server ปกติ** แต่เขียน "handler function" หนึ่งตัวที่ AWS เรียกทุกครั้ง
ที่มี event เข้ามา (HTTP request ผ่าน API Gateway, message จาก SQS queue, ไฟล์ใหม่ใน S3 ฯลฯ)

**ทำไม Rust เหมาะกับ Lambda เป็นพิเศษ**: Lambda คิดค่าใช้จ่ายตาม **เวลาที่ function ทำงานจริง** (หน่วยเป็น
millisecond) คูณด้วย memory ที่จอง — ภาษาที่ cold start ช้า (JVM ที่ต้อง warm up, Python ที่ต้อง import
library หนัก ๆ) เสียเวลา (และเงิน) ในช่วง cold start มากกว่า **Rust compile เป็น native binary ที่ start
เร็วมาก (มักต่ำกว่า 10ms สำหรับ binary เล็ก ๆ ไม่มี runtime ที่ต้อง initialize แบบ garbage-collected language)**
— นี่คือเหตุผลที่ AWS เองเขียน [Firecracker](https://firecracker-microvm.github.io/) (micro-VM ที่ Lambda ใช้
รัน function อยู่ข้างใน — เทคโนโลยีเดียวกับที่ Fly.io ใช้ทำ Machine ตามหัวข้อ 101.2) ด้วย Rust และผลักดัน
`cargo lambda`/`lambda_runtime` crate อย่างจริงจังในฐานะทางเลือก first-class สำหรับเขียน Lambda function

**`cargo lambda` คืออะไร และทำไมใช้งานแบบ offline ได้ 100%**: `cargo lambda` เป็น cargo subcommand (ติดตั้ง
ผ่าน `cargo install cargo-lambda` จาก crates.io ได้ตรง ๆ หรือผ่าน `pip install cargo-lambda`/`brew install
cargo-lambda` ที่แจกจ่าย binary เดียวกันในฟอร์แมตที่ต่าง ecosystem — บทนี้ใช้ทาง `pip` เพราะ `pypi.org` อยู่ใน
network allowlist ของสภาพแวดล้อมนี้) ที่ทำสามอย่างหลัก: **`build`** (cross-compile Rust ให้เป็น binary ที่
ตรงกับ Lambda runtime target โดยอัตโนมัติ ผ่าน [Zig](https://ziglang.org/) เป็น linker backend สำหรับ
cross-compilation แบบไม่ต้องติดตั้ง toolchain ของแต่ละ target เอง), **`watch`** (บูต **emulator ของ AWS
Lambda Runtime API ขึ้นมาในเครื่องเอง 100%** — จำลอง HTTP endpoint ที่ Lambda runtime ปกติจะคุยกับ AWS จริง
แต่ตอนนี้คุยกับตัวเองในเครื่อง ไม่ติดต่อ AWS เลยแม้แต่ byte เดียว), และ **`invoke`** (ส่ง event ปลอมไปยัง
emulator นั้น เหมือนกำลังเรียก Lambda จริงบน AWS แต่ทั้งหมดเกิดขึ้นใน localhost) — ด้วยสถาปัตยกรรมนี้
**การพัฒนาและทดสอบ Lambda function ด้วย Rust ทำได้ครบวงจรโดยไม่ต้องมี AWS account เลยจนกว่าจะพร้อม deploy
จริง** ซึ่งตรงกับเงื่อนไขของบทนี้เป๊ะ — ทุกคำสั่งในหัวข้อนี้ **รันจริงในสภาพแวดล้อมที่เขียนบทนี้ทั้งหมด**

**ติดตั้งจริง**:

```bash
$ pip install cargo-lambda
Collecting cargo-lambda
  Downloading cargo_lambda-1.9.2-py3-none-manylinux_2_5_x86_64.manylinux1_x86_64.whl (14.4 MB)
Collecting ziglang>=0.10.0 (from cargo-lambda)
  Downloading ziglang-0.16.0-py3-none-manylinux_2_12_x86_64.manylinux2010_x86_64.musllinux_1_1_x86_64.whl (97.9 MB)
Successfully installed cargo-lambda-1.9.2 ziglang-0.16.0

$ cargo-lambda lambda --version
cargo-lambda 1.9.2 (67129fc 2026-08-21Z)
```

สังเกตว่า `pip install cargo-lambda` ดึง `ziglang` (แจกจ่าย Zig compiler เป็น Python wheel) มาด้วยเป็น
dependency โดยอัตโนมัติ — นี่คือ Zig ที่ `cargo lambda build` จะใช้เป็น linker ตอน cross-compile ให้ target
Lambda โดยไม่ต้องให้ผู้ใช้ติดตั้ง Zig เองแยก ยืนยันด้วยคำสั่ง diagnostic ของ `cargo lambda` เอง (local-only,
ไม่ต้อง AWS):

```bash
$ cargo lambda system
zig:
  command: <python venv path>/bin/python3 -m ziglang
  version: '0.16.0'
```

**เขียน Lambda function ตัวอย่าง**: สร้างโปรเจกต์เล็ก ๆ ชื่อ `greet-lambda` ที่รับ JSON `{"name": "..."}`
แล้วตอบข้อความทักทาย — ใช้ crate `lambda_runtime` ตรง ๆ (ไม่ใช่ `lambda_http` เพราะตัวอย่างนี้ไม่ได้จำลอง
API Gateway event เต็มรูปแบบ แค่ event ธรรมดาที่สุดเพื่อให้เห็นกลไกของ `cargo lambda` ชัดที่สุด):

```toml
# Cargo.toml
[package]
name = "greet-lambda"
version = "0.1.0"
edition = "2021"

[dependencies]
lambda_runtime = "0.13"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["macros"] }
```

```rust
// src/main.rs
use lambda_runtime::{run, service_fn, Error, LambdaEvent};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Request {
    name: String,
}

#[derive(Serialize)]
struct Response {
    message: String,
}

// handler function — AWS Lambda เรียกฟังก์ชันนี้ทุกครั้งที่มี event เข้ามา
// (ต่างจาก axum handler ของ Part 60+ ที่ Router เรียกตาม route — ที่นี่ event เดียวเรียก
// function เดียวเสมอ ไม่มี routing ในตัว Lambda เอง เพราะ Lambda คือ "หนึ่ง function ต่อหนึ่งงาน")
async fn handler(event: LambdaEvent<Request>) -> Result<Response, Error> {
    let name = event.payload.name;
    Ok(Response {
        message: format!("สวัสดีจาก Lambda, {name}"),
    })
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    run(service_fn(handler)).await
}
```

โครงสร้างนี้ดูคล้าย `#[tokio::main] async fn main()` ของ axum ที่ Part 60 สอน แต่ `run(service_fn(handler))`
แทนที่ `axum::serve(listener, app)` — เหตุผลคือ Lambda ไม่ได้ "ฟัง" TCP port เหมือน web server ปกติ (ไม่มี
`TcpListener::bind` เลยในโค้ดนี้) แต่ `lambda_runtime::run` จะไปคุยกับ **Lambda Runtime API** (ที่ AWS จัดเตรียม
ไว้จริงตอน deploy บน AWS, หรือ emulator ของ `cargo lambda watch` ตอน dev) ผ่าน environment variable
`AWS_LAMBDA_RUNTIME_API` เพื่อ **poll หา event ใหม่** ทีละตัว ประมวลผล แล้วส่งผลลัพธ์กลับไปที่ Runtime API
วนซ้ำไปเรื่อย ๆ — นี่คือกลไกที่ทำให้ handler function เดียวรับ event ได้หลายตัวต่อเนื่องในกรณีที่ AWS นำ
execution environment (container ที่ Lambda เตรียมไว้) กลับมาใช้ซ้ำสำหรับ invocation ถัดไป (เรียกว่า "warm
start" — ตรงข้ามกับ "cold start" ที่ต้องสร้าง execution environment ใหม่)

**Build จริงด้วย `cargo lambda build`**:

```bash
$ cargo lambda build --release
   Compiling lambda_runtime v0.13.0
   Compiling greet-lambda v0.1.0 (/.../lambda-demo)
warning: linker stderr: ignoring deprecated linker optimization setting '1'
    Finished `release` profile [optimized] target(s) in 25.98s

real	0m26.293s
```

`cargo lambda build` cross-compile ให้เป็น binary ที่ AWS Lambda ต้องการโดยอัตโนมัติ — ผลลัพธ์ที่ได้ไม่ใช่
`target/release/greet-lambda` แบบปกติ แต่เป็นไฟล์ชื่อ **`bootstrap`** (ไม่มี extension) ในโฟลเดอร์
`target/lambda/<ชื่อ binary>/`:

```bash
$ ls -la target/lambda/greet-lambda/bootstrap
-rwxr-xr-x 1 root root 1213880 ... target/lambda/greet-lambda/bootstrap
```

ชื่อ `bootstrap` ไม่ใช่ชื่อที่ตั้งเอง — เป็นชื่อไฟล์ **บังคับ** ตามข้อกำหนดของ Lambda **custom runtime**
(`provided.al2`/`provided.al2023`) ที่ Rust ใช้ (เพราะ AWS ไม่มี "native Rust runtime" ให้เลือกโดยตรงแบบ
Node.js/Python — Rust ต้องใช้ custom runtime ที่ AWS เตรียม execution environment เปล่า ๆ แล้วรอ process ชื่อ
`bootstrap` ให้ทำหน้าที่ทุกอย่างเอง ซึ่งก็คือ binary ที่ compile จาก `lambda_runtime` crate นี่เอง) — ไฟล์
`bootstrap` ขนาด ~1.2MB นี้คือทุกอย่างที่ต้องอัปโหลดไป AWS (ผ่าน `cargo lambda deploy` — ไม่ได้รันในบทนี้
เพราะต้องมี AWS credential จริง) ไม่ต้องมี container image ก็ได้ (แม้ `cargo lambda build --output-format
Zip` จะแพ็ก `bootstrap` เป็น ZIP ให้พร้อมอัปโหลดตรง ๆ ก็ตาม — Lambda รองรับทั้งฟอร์แมต ZIP ดั้งเดิมและ
container image)

**Local emulator จริงด้วย `cargo lambda watch`**: คำสั่งนี้ build (โหมด `dev` ไม่ optimize) แล้วบูต Runtime
API emulator พร้อมรัน `bootstrap` (จริง ๆ คือ `cargo run` ธรรมดา ไม่ใช่ cross-compile เพราะรันในเครื่องเดียว
ไม่ต้อง target Lambda จริง) ให้เชื่อมกับ emulator นั้น:

```bash
$ cargo lambda watch
 INFO starting Runtime server runtime_addr=[::]:9000
```

> **หมายเหตุสำคัญที่พบจริงระหว่างตรวจสอบเนื้อหาบทนี้**: ค่า default ของ `cargo lambda watch` ผูก (bind)
> emulator กับ `[::]:9000` (IPv6 "any address") — สภาพแวดล้อมบางประเภท (รวมถึง sandbox ที่เขียนบทนี้) **ไม่มี
> IPv6 stack เลย** ทำให้ bind ล้มเหลวแบบเงียบ ๆ (log ไม่มี ERROR แต่ process จบการทำงานทันทีด้วย exit code
> 0 หลัง log "terminating lambda scheduler") ต้องเพิ่ม flag `-A 127.0.0.1` บังคับให้ bind ด้วย IPv4 แทน — ดู
> รายละเอียดเต็มในกับดักที่พบบ่อยข้อ 1-2 ของบทนี้ (เป็นปัญหาที่พบได้จริงในหลาย container/CI runner ที่ปิด IPv6
> ไว้ ไม่ใช่ปัญหาเฉพาะของสภาพแวดล้อมนี้)

รันใหม่โดยระบุ address ให้ตรง:

```bash
$ cargo lambda watch -A 127.0.0.1 -P 9000
 INFO starting Runtime server runtime_addr=127.0.0.1:9000
 INFO spawning function instance function="_" instance_id=f5e0d0ad-d21e-45a4-ae46-2e9f5d70ef6a
 INFO starting lambda function instance function="_" instance_id=f5e0d0ad-d21e-45a4-ae46-2e9f5d70ef6a
   Compiling greet-lambda v0.1.0 (/.../lambda-demo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 12.62s
     Running `target/debug/greet-lambda`
```

emulator บูตสำเร็จ, compile function สำเร็จ, และรัน binary ที่ได้เชื่อมกับ Runtime API emulator ที่
`127.0.0.1:9000` — ยังไม่มี event ใดเข้ามาเลยตอนนี้ (function กำลัง poll รอ event อยู่เงียบ ๆ เหมือน Lambda
จริงตอนไม่มี invocation)

**Invoke จริงด้วย `cargo lambda invoke`**: ส่ง JSON payload ปลอมไปทดสอบ handler โดยตรง:

```bash
$ cargo lambda invoke --invoke-address 127.0.0.1 --invoke-port 9000 --data-ascii '{"name":"Rust Course"}'
{"message":"สวัสดีจาก Lambda, Rust Course"}

$ cargo lambda invoke --invoke-address 127.0.0.1 --invoke-port 9000 --data-ascii '{"name":"Ferris"}'
{"message":"สวัสดีจาก Lambda, Ferris"}
```

**ทั้งสอง invocation สำเร็จจริง 100% ในเครื่องเดียว ไม่มีการเชื่อมต่อ AWS แม้แต่ครั้งเดียว** — `event.payload.name`
ถูก deserialize ถูกต้องจาก JSON ที่ส่งเข้าไป, handler ประมวลผลและคืนค่าตาม struct `Response` ที่ serialize
กลับมาเป็น JSON ตรงตามที่ AWS Lambda จริงจะทำ (แค่ AWS จริงจะมี API Gateway/event source อื่นเป็นผู้ส่ง event
เข้ามาแทนคำสั่ง `cargo lambda invoke` ที่ทำหน้าที่นี้แทนตอน dev) — นี่คือหลักฐานที่แสดงว่า flow การพัฒนา Lambda
function ด้วย Rust **ทดสอบได้ครบวงจรแบบ offline สมบูรณ์** ตรงตามที่โจทย์ของบทนี้ต้องการ

**Cold Start ในรายละเอียด — ทำไม memory ที่จองมีผลต่อความเร็ว CPU ด้วย**: จุดที่มักเข้าใจผิดคือคิดว่า memory
ที่จองให้ Lambda function มีผลแค่เรื่อง "จะ out-of-memory หรือไม่" — จริง ๆ แล้ว **AWS จัดสรร CPU ให้ Lambda
function ตามสัดส่วนของ memory ที่จอง** (จองน้อย ได้ CPU น้อยตามไปด้วย ไม่ใช่ CPU เต็มความเร็วเสมอ) ทำให้
บางครั้ง**การเพิ่ม memory กลับทำให้ค่าใช้จ่ายรวมถูกลง** แม้ราคาต่อ GB-second จะสูงขึ้นตามสัดส่วน memory เพราะ
function ทำงานเสร็จเร็วขึ้นมากจนเวลารวมที่ต้องจ่ายลดลงมากกว่าที่ราคาต่อหน่วยเพิ่มขึ้น (ต้องวัดจริงเป็นกรณี ๆ
ไป ไม่มีค่าที่ดีที่สุดตายตัว) — สำหรับ Rust ที่ binary เบาและ start เร็วอยู่แล้วตามที่อธิบายไว้ต้นหัวข้อ ผล
ต่างจากการปรับ memory มักเห็นได้น้อยกว่าภาษาที่ CPU-bound หนัก ๆ ตอน cold start (เช่นภาษาที่ต้อง JIT compile
ตอน start) แต่ยังคุ้มที่จะทดลองปรับดูสำหรับ workload ที่มีการคำนวณหนักจริง ๆ ใน handler

**Provisioned Concurrency — ทางเลือกสำหรับกำจัด cold start ให้เหลือศูนย์ (แลกกับเสียประโยชน์ scale-to-zero
ไปบางส่วน)**: ถ้า cold start (แม้จะเร็วมากสำหรับ Rust) ยังไม่เป็นที่ยอมรับได้เลยสำหรับบางกรณี (เช่น API ที่
ต้องการ latency ต่ำสุดเสมอไม่มีข้อยกเว้น) AWS มี **Provisioned Concurrency** ให้ตั้งจำนวน execution
environment ที่ **"อุ่นไว้ล่วงหน้าตลอดเวลา"** (คล้ายกับตั้ง `min_machines_running` ของ Fly.io ในหัวข้อ 101.3)
— แลกกับการต้อง**จ่ายค่าใช้จ่ายคงที่สำหรับ environment ที่อุ่นไว้นั้นตลอดเวลาไม่ว่าจะมี invocation จริงหรือ
ไม่** ทำให้เสียคุณสมบัติ "จ่ายตามการใช้งานจริง 100%" ของ Lambda ไปบางส่วนแลกกับ latency ที่แน่นอนกว่า — เป็น
ตัวอย่างที่ดีว่าทุก serverless platform มักมี "ทางออก" สำหรับ trade-off ของตัวเองเสมอ แต่ทางออกนั้นมักต้อง
แลกกับคุณสมบัติที่เป็นจุดขายหลักของ platform นั้นบางส่วนเสมอเช่นกัน

**สถาปัตยกรรม CPU — ARM (Graviton) เทียบกับ x86_64**: `cargo lambda build` รองรับ cross-compile ให้ทั้ง
`x86_64-unknown-linux-gnu` (default) และ `aarch64-unknown-linux-gnu` (ARM) ผ่าน flag `--arm64` โดยไม่ต้อง
ติดตั้ง toolchain เพิ่มเอง (Zig backend ที่ `cargo lambda` ใช้จัดการ cross-compilation ให้ทั้งหมด) — ทดสอบ
จริงในสภาพแวดล้อมที่เขียนบทนี้:

```bash
$ cargo lambda build --release --arm64
    Finished `release` profile [optimized] target(s) in 42.22s
```

build ผ่านสำเร็จจริงโดยไม่ต้องแก้โค้ดแม้แต่บรรทัดเดียว (Rust cross-compile ข้าม architecture ได้ตรงไปตรงมา
กว่าหลายภาษาที่ต้องพึ่ง native extension/FFI ที่ผูกกับ architecture เฉพาะ) — เหตุผลที่ควรสนใจตัวเลือกนี้:
AWS Lambda บน **AWS Graviton (ARM)** มีราคาต่อ GB-second **ถูกกว่า x86_64 ประมาณ 20% ตามที่ AWS ประกาศไว้**
(ตัวเลขนี้เป็นสัดส่วนที่ AWS สื่อสารต่อสาธารณะ อาจเปลี่ยนแปลงได้) โดยประสิทธิภาพต่อ core ใกล้เคียงกันหรือดีกว่า
สำหรับ workload จำนวนมาก — เพราะ Rust compile เป็น native machine code ให้ทั้งสอง architecture ได้เท่าเทียม
กัน (ไม่มี "penalty" แบบภาษาที่ต้องพึ่ง JIT/interpreter ที่อาจ optimize ให้ x86 ดีกว่า ARM หรือกลับกัน) **การ
เลือก ARM สำหรับ Lambda function ที่เขียนด้วย Rust จึงมักเป็นตัวเลือกที่ "ได้เปล่า" ในแง่ต้นทุน** แทบไม่มี
downside ให้ต้องแลก ต่างจากภาษาอื่นที่บางครั้ง library dependency ยังไม่รองรับ ARM สมบูรณ์ (ต้องเช็ค
compatibility เพิ่มเติม)

**Packaging เป็น Container Image แทน ZIP/`bootstrap`**: `cargo lambda build --output-format Zip` (ตามที่
เกริ่นไว้ก่อนหน้า) ให้ ZIP ที่มี `bootstrap` อยู่ข้างใน แต่ Lambda ยังรองรับการ deploy เป็น **container image**
โดยตรงด้วย (ขนาดสูงสุดถึง 10GB ต่าง จาก ZIP ที่จำกัดเล็กกว่ามาก) — ถ้าเลือกทางนี้ Dockerfile ต้อง `FROM`
base image พิเศษที่ AWS เตรียมไว้ (`public.ecr.aws/lambda/provided:al2023` สำหรับ custom runtime อย่าง Rust)
แทน `scratch`/`debian:bookworm-slim` แบบที่ Part 96 สอน เพราะ Lambda container image ต้องมี Lambda Runtime
Interface Client ฝังอยู่ (หรือ compile เข้ากับ binary เองแบบที่ `lambda_runtime` crate ทำให้แล้ว) — สำหรับ
capstone ของหลักสูตรนี้ที่เน้น musl static binary บน `scratch` (Part 96 หัวข้อ 96.5) เส้นทาง ZIP/`bootstrap`
ตรงไปตรงมากว่ามาก เพราะไม่ต้องเปลี่ยน base image ของ Dockerfile เลย — เส้นทาง container image เหมาะกับกรณีที่
โปรเจกต์มี dependency ระดับ OS ที่ซับซ้อนเกินกว่าจะฝังใน ZIP function ธรรมดา (เช่นต้องมี native library ขนาด
ใหญ่ที่ base Lambda runtime ไม่มีให้)

**ขั้นตอนที่ต้องมี AWS account จริง (ไม่ได้ทำในบทนี้ — อธิบายจากเอกสารทางการ)**: หลัง build/test ในเครื่อง
พอใจแล้ว การ deploy จริงใช้ `cargo lambda deploy` (ต้องตั้งค่า AWS credential ผ่าน `aws configure` หรือ
environment variable `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` ก่อน) คำสั่งนี้จะอัปโหลด `bootstrap`
(หรือสร้าง container image แล้ว push ขึ้น ECR) ไปสร้าง/อัปเดต Lambda function บน AWS จริง แล้วมักต่อกับ
**API Gateway** หรือ **Lambda Function URL** (ทางลัดใหม่กว่าที่ AWS เพิ่มมา ไม่ต้องตั้ง API Gateway แยกถ้า
ไม่ต้องการ feature เพิ่มเติมของมัน) เพื่อให้ Lambda function รับ HTTP request จาก public internet ได้ — ทั้ง
สองขั้นตอนนี้ (`cargo lambda deploy` และการต่อ API Gateway/Function URL) เขียนจากเอกสารทางการของ
`cargo-lambda`/AWS เท่านั้น ไม่ได้รันจริงในสภาพแวดล้อมนี้ เพราะไม่มี AWS credential ให้ authenticate

**ต่อยอด — Lambda ที่รับ event จาก API Gateway จริงด้วย `lambda_http`**: ตัวอย่างข้างบนใช้ `lambda_runtime`
ตรง ๆ กับ event ที่กำหนดโครงสร้างเอง — แต่สถานการณ์ที่พบบ่อยกว่ามากในโลกจริงคือ Lambda ที่อยู่หลัง **API
Gateway** (หรือ Function URL) เพื่อรับ HTTP request จาก public internet — สำหรับกรณีนี้ crate `lambda_http`
(แยกจาก `lambda_runtime` แต่ใช้ร่วมกัน) ให้ type `Request`/`Response<Body>` ที่มี**หน้าตาคล้าย `http` crate
มาตรฐานที่ Part 60 ใช้กับ axum มาก** ทำให้ปรับโค้ดจาก axum handler มาเป็น Lambda handler ได้ไม่ยาก:

```toml
# Cargo.toml
[dependencies]
lambda_http = "0.14"
lambda_runtime = "0.13"
tokio = { version = "1", features = ["macros"] }
```

```rust
// src/main.rs
use lambda_http::{run, service_fn, Body, Error, Request, RequestExt, Response};

async fn handler(event: Request) -> Result<Response<Body>, Error> {
    // RequestExt::query_string_parameters_ref() ให้ query parameter จาก URL
    // เหมือนกับ Query<T> extractor ของ axum ที่ Part 63 สอน แต่ดึงค่าด้วยมือ
    // ตรง ๆ เพราะ Lambda ไม่มี extractor pattern แบบ axum ในตัว
    let name = event
        .query_string_parameters_ref()
        .and_then(|params| params.first("name"))
        .unwrap_or("world");

    let body = format!("สวัสดี, {name} (path: {})", event.uri().path());

    Ok(Response::builder()
        .status(200)
        .header("content-type", "text/plain; charset=utf-8")
        .body(body.into())
        .map_err(Box::new)?)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    run(service_fn(handler)).await
}
```

Build จริงสำเร็จ (`cargo lambda build --release` ใช้เวลา 33.4 วินาที รวม dependency ของ `lambda_http`/
`aws_lambda_events` ที่หนักกว่าตัวอย่างแรกเล็กน้อยเพราะต้อง deserialize event schema ของ API Gateway เต็ม
รูปแบบ) แล้วทดสอบด้วย `cargo lambda watch -A 127.0.0.1 -P 9001` (คนละ port จากตัวอย่างแรกเพื่อรันคู่กันได้)
— คราวนี้ต้อง**ส่ง event ให้ตรงกับรูปแบบที่ API Gateway HTTP API (payload format version 2.0) จริงจะส่งมา**
(ไม่ใช่ JSON ธรรมดาแบบตัวอย่างแรก) — `cargo lambda invoke` มี flag `--data-example` ที่ดึง example event
สำเร็จรูปจาก `event-examples.cargo-lambda.info` มาให้ตรง ๆ แต่โดเมนนี้**ไม่อยู่ใน network allowlist ของ
สภาพแวดล้อมนี้** ทำให้ต้อง fail แบบนี้ (บันทึกไว้จริงตามที่พบ):

```bash
$ cargo lambda invoke --invoke-address 127.0.0.1 --invoke-port 9001 --data-example apigw-request
Error:   × error dowloading example data
  ├─▶ error sending request for url (https://event-examples.cargo-lambda.info/example-apigw-request.json)
  ├─▶ client error (Connect)
  ╰─▶ unsuccessful tunnel
```

จึงเขียน event payload ของ API Gateway HTTP API (v2.0) เองแทน (`apigw_v2_event.json` — โครงสร้าง JSON ตรง
ตาม schema ที่ AWS เอกสารไว้ ประกอบด้วย `requestContext.http`, `rawPath`, `queryStringParameters` ฯลฯ) แล้ว
invoke ด้วย `--data-file`:

```bash
$ cargo lambda invoke --invoke-address 127.0.0.1 --invoke-port 9001 --data-file apigw_v2_event.json
{"statusCode":200,"headers":{"content-type":"text/plain; charset=utf-8"},"multiValueHeaders":{},"body":"สวัสดี, Ferris (path: /greet)","isBase64Encoded":false,"cookies":[]}
```

**สำเร็จจริง** — สังเกตว่า response ที่ `cargo lambda invoke` แสดงกลับมา **ไม่ใช่** `Response<Body>` ดิบ ๆ
ที่ handler คืนตรง ๆ แต่เป็น **JSON ที่ `lambda_http` ห่อให้ตรงตาม schema ที่ API Gateway คาดหวัง**
(`statusCode`/`headers`/`multiValueHeaders`/`body`/`isBase64Encoded`/`cookies`) — นี่คือสิ่งที่ `lambda_http`
ทำให้อัตโนมัติ: แปลง `Response<Body>` แบบมาตรฐานของ Rust ให้เป็น "API Gateway proxy response" ที่ AWS ต้องการ
พอดี ทำให้นักพัฒนาเขียนโค้ดด้วย type ที่คุ้นเคย (เหมือนเขียน axum handler) โดยไม่ต้องรู้จัก JSON schema ที่ AWS
กำหนดไว้เองเลยแม้แต่บรรทัดเดียว และ `event.query_string_parameters_ref()` ก็ดึงค่า `name=Ferris` จาก
`queryStringParameters` ของ event ได้ถูกต้อง ยืนยันว่า deserialization ทำงานถูกต้องสมบูรณ์ทั้ง input และ
output

**Cleanup ของหัวข้อนี้**: หลังตรวจสอบเสร็จ ลบโปรเจกต์ทั้งสอง (`greet-lambda`, `http-lambda-demo`), หยุด
`cargo lambda watch` process ทั้งหมด, และลบ virtual environment ของ `pip` ที่สร้างไว้ทั้งหมดออกจาก scratch
directory (ไม่กระทบไฟล์ใด ๆ ในหลักสูตร) ตามที่ระบุไว้ในหมายเหตุต้นบท

### 101.7 GCP Cloud Run: ภาพรวมเชิงแนวคิด

Cloud Run คือบริการของ Google Cloud Platform ที่**อยู่กึ่งกลางระหว่าง ECS/Fargate กับ Lambda**: รับ container
image มาตรฐาน (เหมือน ECS/Fargate) แต่มีโมเดล billing และ scale-to-zero แบบ serverless (เหมือน Lambda) —
เขียนแบบสรุปเปรียบเทียบ:

| | Cloud Run | ECS/Fargate | Lambda |
|---|---|---|---|
| หน่วยที่ deploy | Container image มาตรฐาน (เหมือน Fly.io) | Container image มาตรฐาน | ZIP หรือ container image ของ **function เดียว** (ต้องเข้ากับ Lambda Runtime API) |
| Scale-to-zero | มี (ไม่มี traffic = ไม่มีค่าใช้จ่าย, คล้าย `auto_stop_machines` ของ Fly.io) | ไม่มีในตัว (ต้องตั้ง Service Auto Scaling ขั้นต่ำ 0 task เองได้แต่ไม่ใช่ default) | มีเสมอ (ธรรมชาติของ FaaS) |
| รับ HTTP request ตรง ๆ ได้ไหม | ได้ตรง (Cloud Run มี HTTPS endpoint ให้อัตโนมัติทันที เหมือน Fly.io) | ต้องตั้ง Load Balancer เอง | ต้องผ่าน API Gateway/Function URL เพิ่ม |
| เขียนโค้ดต้องปรับจาก web service ปกติไหม | **ไม่ต้องเลย** — แค่ฟัง port ที่กำหนดผ่าน `$PORT` env var, เหมือน axum service ปกติทุกประการ | ไม่ต้องปรับ | ต้องปรับ (ใช้ `lambda_runtime`/`lambda_http` แทน `axum::serve` ตรง ๆ ตามหัวข้อ 101.6) |

**จุดที่ Cloud Run น่าสนใจที่สุดสำหรับแอป Rust**: เพราะรับ container image มาตรฐานและแอปแค่ต้องฟัง HTTP บน
port ที่กำหนดผ่าน environment variable `PORT` (ค่า default คือ 8080) **แอป axum ปกติที่เขียนตาม Part 60-65
โดยไม่ต้องรู้จัก Cloud Run มาก่อนเลยก็ deploy บน Cloud Run ได้ตรง ๆ** แค่เปลี่ยนจาก `TcpListener::bind
("0.0.0.0:8080")` ที่ hardcode เป็น `TcpListener::bind(format!("0.0.0.0:{}", env::var("PORT")
.unwrap_or_else(|_| "8080".into())))` — สอดคล้องกับหลัก twelve-factor app (อ่าน config จาก environment
variable) ที่ Part 96 หัวข้อ 96.8 สอนไว้แล้วพอดี ทำให้ image เดียวกับ capstone ของ Part 96 (ที่อ่าน
`BIND_ADDR` จาก environment variable อยู่แล้ว) **ปรับให้ deploy บน Cloud Run ได้ง่ายมาก** เพียงตั้งค่า
`BIND_ADDR=0.0.0.0:$PORT` ผ่าน `gcloud run deploy --set-env-vars` (documented syntax — ไม่ได้รันจริงในบทนี้
เพราะต้องมี GCP account) แทนการ hardcode 8080 ตรง ๆ

**คำสั่งหลักที่ใช้ (เอกสารอ้างอิงเท่านั้น ไม่ได้รันจริง)**: `gcloud run deploy <service-name> --image
<image-url> --region <region> --platform managed` เป็นคำสั่งเดียวที่ deploy image ไปเป็น Cloud Run service
พร้อม HTTPS endpoint อัตโนมัติ — ใกล้เคียงความเรียบง่ายของ `fly deploy` มากกว่า ECS/Fargate ที่ต้องตั้ง
infrastructure หลายชิ้นเอง

### 101.8 Config และ Secrets ในบริบท Cloud Deployment

Part 96 หัวข้อ 96.8 สอนหลักการไว้แล้วว่า **config ที่ไม่ใช่ความลับใส่ environment variable ธรรมดาได้ (เช่น
ผ่าน `[env]` ใน `fly.toml`) ส่วน secret (`DATABASE_URL`, `JWT_SECRET`) ต้องแยกเก็บต่างหากและห้าม commit เข้า
git เด็ดขาด** — หัวข้อนี้ขยายหลักการเดียวกันไปสู่บริบทที่มีหลายแพลตฟอร์ม โดยแต่ละแพลตฟอร์มมี**กลไกเก็บ secret
ของตัวเอง** ที่ทำหน้าที่เดียวกัน (encrypt at rest, inject เป็น environment variable ตอน container start,
ไม่แสดงค่าใน log/UI ปกติ) แต่ syntax คำสั่งต่างกัน:

**Fly.io** — `fly secrets set KEY=value` (documented syntax, ไม่ได้รันจริงตามที่อธิบายไว้ต้นบท):

```bash
# ตัวอย่าง syntax ตามเอกสาร Fly.io — ตั้ง secret ของ capstone ก่อน deploy ครั้งแรก
fly secrets set \
  DATABASE_URL="postgres://user:password@host.example.com:5432/library_mini" \
  JWT_SECRET="production-secret-สร้างด้วย openssl-rand-อย่างน้อย-32-bytes"
```

secret ที่ตั้งด้วยคำสั่งนี้ถูก encrypt เก็บไว้ที่ Fly.io และ inject เป็น environment variable ให้ process
ภายใน Machine ตอน start เท่านั้น (`fly secrets list` แสดงแค่**ชื่อ** ไม่แสดงค่า ป้องกันไม่ให้ค่าจริงหลุดผ่าน
การแสดงผลโดยไม่ตั้งใจ) — จุดสำคัญที่ต่างจาก `[env]` คือ **secret ไม่ถูกเก็บใน `fly.toml`** เลย ทำให้ commit
`fly.toml` เข้า git ได้อย่างปลอดภัยโดยไม่ต้องกลัวรั่ว credential

**AWS** — ไม่มีคำสั่งเดียวที่ครอบคลุมทุกกรณีแบบ `fly secrets set` เพราะ AWS แยกบริการ secret เป็นหลายตัว:
**AWS Systems Manager (SSM) Parameter Store** (เก็บ config/secret แบบ key-value เรียบง่าย ค่าใช้จ่ายต่ำ/ฟรี
สำหรับ standard tier) กับ **AWS Secrets Manager** (มี feature เพิ่มเช่น auto-rotation ของ credential
database แต่มีค่าใช้จ่ายต่อ secret) — syntax ตัวอย่าง (documented, ไม่ได้รันจริง):

```bash
# SSM Parameter Store — ตั้งค่าแบบ SecureString (encrypt ด้วย KMS อัตโนมัติ)
aws ssm put-parameter --name "/library-api/DATABASE_URL" --value "postgres://..." \
  --type SecureString

# ECS Task Definition อ่านค่ากลับมาผ่าน "secrets" field (ไม่ใช่ "environment" field ธรรมดา)
# ระบุ ARN ของ parameter แทนค่าตรง ๆ — ECS จะ resolve และ inject เป็น env var ตอน container start
```

ความต่างสำคัญ: ECS Task Definition แยก field `environment` (ค่าธรรมดา, เห็นได้ผ่าน `describe-task-definition`)
ออกจาก field `secrets` (ระบุ ARN ของ SSM parameter/Secrets Manager secret แทนค่าตรง ๆ) อย่างชัดเจนในระดับ
schema เอง — บังคับให้แยก concept สองอย่างนี้ตั้งแต่ตอนเขียน infrastructure-as-code เลย ไม่ต้องพึ่งความ
ระมัดระวังของ developer อย่างเดียว

**Lambda** — ตั้ง environment variable ของ function ผ่าน `--environment` flag ของ `aws lambda
update-function-configuration` หรือผ่าน `cargo lambda deploy --env-var KEY=value` ตรง ๆ (สำหรับ config
ทั่วไป) — สำหรับ secret จริงจัง AWS แนะนำให้ function **ดึงค่าจาก Secrets Manager ณ runtime ตอน cold start**
(ผ่าน AWS SDK เรียก Secrets Manager API) แทนการฝัง secret เป็น environment variable ตรง ๆ เพื่อได้ประโยชน์
จาก auto-rotation (ถ้า rotate credential ใน Secrets Manager ไม่ต้อง redeploy function ใหม่เลย function ดึง
ค่าใหม่เองตอน cold start ครั้งถัดไป)

**GCP Cloud Run** — `gcloud run services update <name> --set-env-vars KEY=value` สำหรับ config ทั่วไป, หรือ
`--set-secrets KEY=SECRET_NAME:VERSION` ที่เชื่อมกับ **Secret Manager** ของ GCP (เทียบเท่า AWS Secrets
Manager) สำหรับ secret จริงจัง — แนวคิดเหมือนกันกับ AWS ทุกประการ: แยก "ค่าธรรมดา" ออกจาก "secret ที่มา
จากบริการเก็บ secret เฉพาะทาง" อย่างชัดเจนในระดับ CLI flag เอง

**สรุปหลักการที่เหมือนกันทุกแพลตฟอร์ม (จุดที่สำคัญกว่า syntax ที่ต่างกัน)**: ไม่ว่าจะใช้แพลตฟอร์มไหน หลักการ
ของ twelve-factor app ที่ Part 96 สอนไว้ยังคงเป็นจริงเสมอ — **แอปอ่าน config จาก environment variable
เท่านั้น ไม่ hardcode ค่าใด ๆ ในโค้ด** ส่วน "environment variable นั้นมาจากไหน" (ไฟล์ `.env` ตอน dev,
`docker-compose environment:` ตอน Part 96, `fly secrets`/SSM Parameter Store/Secret Manager ตอน production)
คือรายละเอียดของแต่ละ deployment target ที่**ไม่ควรมีผลต่อโค้ดแอปเลยแม้แต่บรรทัดเดียว** — นี่คือเหตุผลที่
แอปเดียวกัน (image เดียวกัน) deploy ได้ทั้ง Fly.io/ECS/Cloud Run โดยไม่ต้องแก้ business logic เลย ตรงกับที่
Part 96 หัวข้อ 96.10 พิสูจน์ไว้แล้วว่า capstone ทำงานถูกต้องทั้งบน `docker run` ตรง ๆ และ `docker-compose`
เพราะมันอ่านทุกอย่างจาก environment variable ตั้งแต่ต้น

**สรุปรวม — environment variable ของ capstone `library_api_mini` ตั้งที่ไหนในแต่ละ target**: เพื่อให้เห็น
ภาพรวมทั้งหมดในตารางเดียว (สังเคราะห์จากทุกหัวข้อของบทนี้และ Part 96):

| Environment Variable | เป็น secret ไหม | Fly.io | ECS Task Definition | Lambda |
|---|---|---|---|---|
| `BIND_ADDR` | ไม่ (config ทั่วไป) | `[env]` ใน `fly.toml` | field `environment` | ไม่เกี่ยว (Lambda ไม่ bind TCP port เอง ตามหัวข้อ 101.6) |
| `RUST_LOG` | ไม่ | `[env]` ใน `fly.toml` | field `environment` | `--env-var RUST_LOG=info` ตอน `cargo lambda deploy` |
| `DATABASE_URL` | **ใช่** | `fly secrets set` | field `secrets` (อ้าง ARN ของ SSM/Secrets Manager) | ดึงจาก Secrets Manager ที่ runtime (แนะนำ) หรือ `--env-var` (ไม่แนะนำสำหรับ production) |
| `JWT_SECRET` | **ใช่** | `fly secrets set` | field `secrets` | ดึงจาก Secrets Manager ที่ runtime (แนะนำ) |

ตารางนี้ตอกย้ำหลักการของหัวข้อ 101.8 อีกครั้งในรูปแบบที่นำไปใช้ตรงได้ทันที: **ไม่ว่า deploy target จะเป็น
อะไร ค่าที่เป็นความลับต้องผ่านกลไก secret เฉพาะของ platform นั้นเสมอ ไม่เคยใส่ในช่องของ config ทั่วไป** — และ
โค้ดแอป (`env::var("DATABASE_URL")`) **เหมือนกันทุกประการไม่ว่าจะ deploy ที่ไหน** เพราะสุดท้ายทุก target ก็
inject เป็น environment variable ธรรมดาให้ process อ่านผ่าน `std::env` เหมือนกันหมด ต่างกันแค่ "ที่มา" ของค่า
นั้นเบื้องหลังเท่านั้น

### 101.9 Managed Database Hosting: RDS, Cloud SQL, Fly Postgres, Neon, Supabase

Part 70 สอนการต่อ PostgreSQL ผ่าน `DATABASE_URL` (connection string) และ Part 96 รัน PostgreSQL ผ่าน
`docker-compose` สำหรับ dev/demo — ตอน deploy จริงขึ้นคลาวด์ แทบไม่มีใครรัน PostgreSQL เองบน VM/container
ที่ตัวเองดูแล (ต้องจัดการ backup, patch security, tuning เอง ซึ่งเป็นงานเฉพาะทางมาก) แต่ใช้ **managed
database service** ที่ provider จัดการ operational work ทั้งหมดให้ แล้วให้ connection string มาต่อจากแอป
ตรง ๆ — connection string ที่ได้มีรูปแบบเดียวกับที่ Part 70 สอนทุกประการ (`postgres://user:pass@host:port/db`)
เปลี่ยนแค่ hostname/credential เท่านั้น โค้ด `sqlx::PgPool::connect(&database_url)` ไม่ต้องแก้อะไรเลย

| Provider | ผูกกับ Platform ไหน | จุดเด่น | Free tier |
|---|---|---|---|
| **AWS RDS for PostgreSQL** | AWS (ใช้กับ EC2/ECS/Lambda ได้หมด) | Feature ครบที่สุด (Multi-AZ failover, read replica, automated backup, snapshot), integrate แน่นกับ IAM | มี free tier จำกัดขนาด/เวลาสำหรับบัญชีใหม่ |
| **GCP Cloud SQL for PostgreSQL** | GCP (ใช้กับ Cloud Run/GCE ได้) | ใกล้เคียง RDS ในแง่ feature, integrate แน่นกับ Cloud Run ผ่าน Cloud SQL Auth Proxy | มี free tier จำกัดคล้าย RDS |
| **Fly Postgres** (Managed Postgres ของ Fly.io เอง) | Fly.io | ตั้งง่ายที่สุดถ้าแอป deploy บน Fly.io อยู่แล้ว (region เดียวกัน, latency ต่ำสุด, ผ่าน `fly postgres create`) | ไม่มี tier ฟรีถาวร แต่ราคาเริ่มต้นต่ำ |
| **Neon** | Platform-agnostic (ใช้กับที่ไหนก็ได้ที่ต่อ internet ถึง) | **Serverless Postgres** — scale-to-zero ได้แบบ compute (เก็บข้อมูลถาวรแต่ปิด compute ตอนไม่มี query ประหยัดเงินคล้ายหลักการ auto-stop ของ Fly.io Machine), branching ของ database คล้าย git branch | มี free tier ใช้งานได้จริงสำหรับโปรเจกต์เล็ก |
| **Supabase** | Platform-agnostic | Postgres + ชุด feature เพิ่ม (auth, realtime subscription, storage, auto-generated REST/GraphQL API) — เหมาะถ้าต้องการ backend-as-a-service ไม่ใช่แค่ database ดิบ | มี free tier ใช้งานได้จริงสำหรับโปรเจกต์เล็ก |

**รูปแบบ connection string จริงของแต่ละ provider (documented — เพื่อเทียบให้เห็นว่าต่างจาก Part 70 แค่ตรง
hostname/พารามิเตอร์เสริมเท่านั้น)**:

```bash
# AWS RDS — hostname เป็น endpoint ที่ RDS สร้างให้อัตโนมัติตอนสร้าง instance
DATABASE_URL="postgres://appuser:password@library-mini.xxxxxxxx.ap-southeast-1.rds.amazonaws.com:5432/library_mini"

# GCP Cloud SQL — ถ้าต่อผ่าน public IP ตรง ๆ (ต้องเปิด Authorized Network ก่อน) หรือผ่าน Cloud SQL Auth Proxy
# ที่รันเป็น sidecar แล้วต่อ localhost แทน (แนะนำกว่าเพราะไม่ต้องเปิด public IP เลย)
DATABASE_URL="postgres://appuser:password@127.0.0.1:5432/library_mini"   # ผ่าน Cloud SQL Auth Proxy

# Fly Postgres — hostname เป็นชื่อ internal ของ Fly.io private network (.internal)
# ทำงานได้เฉพาะจาก Machine อื่นใน organization เดียวกันบน Fly.io เท่านั้น ไม่ expose สู่ public internet เลย
DATABASE_URL="postgres://appuser:password@library-api-mini-db.internal:5432/library_mini"

# Neon — มี query parameter "sslmode=require" เสมอ (บังคับเชื่อมต่อผ่าน TLS เท่านั้น ไม่มีโหมดไม่เข้ารหัสให้เลือก)
DATABASE_URL="postgres://appuser:password@ep-cool-name-123456.ap-southeast-1.aws.neon.tech/library_mini?sslmode=require"
```

สังเกตว่า **`sqlx::PgPool::connect(&database_url)` ที่ Part 70 สอนไม่ต้องแก้โค้ดแม้แต่บรรทัดเดียวสำหรับทั้ง
สี่ provider นี้** เพราะทั้งหมดพูด wire protocol ของ PostgreSQL มาตรฐานเดียวกัน — ความต่างมีแค่ hostname/
query parameter ที่อยู่ใน connection string เท่านั้น ซึ่งเป็นสิ่งที่ environment-variable-based config
(ตามหลักการของ Part 96 หัวข้อ 96.8 และหัวข้อ 101.8 ของบทนี้) ถูกออกแบบมาให้จัดการเรื่องนี้อยู่แล้วโดยธรรมชาติ

**หลักการเลือก**: ถ้า deploy อยู่บน AWS/GCP อยู่แล้ว (ECS/Lambda หรือ Cloud Run) การใช้ RDS/Cloud SQL ตามลำดับ
มักคุ้มที่สุดเพราะ **latency ต่ำสุด** (database อยู่ VPC/network เดียวกับแอป ไม่ต้องออกไปนอก network) และ
**integrate กับระบบสิทธิ์ (IAM) ของ platform เดียวกัน** ได้แน่นกว่า — ถ้า deploy บน Fly.io การใช้ Fly Postgres
ให้ประโยชน์คล้ายกัน (region เดียวกับแอป) — ถ้าต้องการความ flexible สูงสุด (ไม่ผูกกับ deployment platform
ไหนเลย, ยากต่อการ lock-in) หรือต้องการ feature พิเศษอย่าง database branching สำหรับ testing (สร้าง copy ของ
database ทั้งชุดแบบ copy-on-write ในไม่กี่วินาทีสำหรับแต่ละ PR/feature branch) Neon เป็นตัวเลือกที่ได้รับ
ความนิยมสูงมากในช่วงหลัง — ส่วน Supabase เหมาะกับโปรเจกต์ที่ต้องการ feature เสริมรอบ ๆ database (auth,
realtime) มากกว่าแค่ PostgreSQL ดิบ ๆ

**Connection Pooling — ทำไม managed database มักมี "proxy" ตัวเพิ่มเติมเสมอ**: Part 70 สอนให้สร้าง
`PgPool` ด้วย `PgPoolOptions::new().max_connections(N)` — บนเครื่อง dev ที่มี PostgreSQL instance เดียวและ
แอปเดียวต่ออยู่ N connection ไม่เคยเป็นปัญหา แต่พอ deploy จริงที่ **scale เป็นหลาย Machine/Task/instance
พร้อมกัน** (ตามที่ rolling deployment ของหัวข้อ 101.10 อาจมี Machine เก่า+ใหม่วิ่งซ้อนกันชั่วขณะ หรือแค่
ตั้ง `min_machines_running = 3` ปกติ) **จำนวน connection รวมของทุก instance คูณกันเข้า อาจชนเพดานที่
PostgreSQL รับได้จริง** (PostgreSQL เพดาน default คือ `max_connections = 100` และแต่ละ connection ใช้ memory
ของ server ฝั่ง database เองด้วย ไม่ใช่ทรัพยากรที่ไม่มีเพดาน) — ถ้า 5 Machine ตั้ง pool ละ 20 connection คือ
100 connection พอดีเต็มเพดาน ไม่มีเหลือให้ connection อื่น (เช่น admin tool ที่ต้องต่อเข้าไปดูข้อมูล) เลย

ทางแก้ที่ทุก managed provider หลักมีให้คือ **connection pooler แยกชั้น** ที่อยู่ระหว่างแอปกับ database จริง
ทำหน้าที่ "รวม" connection จากแอปหลาย instance เข้าเป็น connection จำนวนน้อยกว่าไปยัง database จริงอีกที (คล้าย
หลักการ multiplexing):

- **RDS Proxy** (AWS) — proxy ที่ตั้งแยกจาก RDS instance เอง แอปต่อไปที่ RDS Proxy endpoint แทนต่อ RDS
  ตรง ๆ
- **Cloud SQL Auth Proxy** (GCP) — รันเป็น sidecar/local process ข้าง ๆ แอป (หรือใน container เดียวกัน) ทำ
  หน้าที่ authenticate ผ่าน IAM แทน password ตรง ๆ ด้วย พร้อม pooling
- **Fly Proxy สำหรับ Fly Postgres** — Fly Postgres มี built-in connection pooler (PgBouncer ภายใน) ให้เลือก
  ต่อผ่าน port ที่ pool ไว้แล้วแทนต่อ PostgreSQL ตรง ๆ
- **PgBouncer** — ตัวเลือกแบบ self-hosted ที่ใช้ได้กับ PostgreSQL ทุกที่ (ไม่ผูกกับ cloud provider ใด) นิยม
  ตั้งเป็น sidecar container คู่กับแอปหรือแยกเป็น service กลางที่หลายแอปต่อร่วมกัน — Neon ก็ built-in
  PgBouncer-compatible pooling ไว้ในโหมด "pooled connection string" ที่ให้เลือกใช้แทน direct connection
  string ได้เลย

**ผลต่อโค้ดแอป**: connection string ที่เปลี่ยนไปต่อผ่าน pooler มักมี**รูปแบบเดียวกันเป๊ะ** กับที่ Part 70 สอน
(`postgres://user:pass@pooler-host:port/db`) เปลี่ยนแค่ hostname/port — โค้ด `sqlx::PgPool::connect` ไม่ต้อง
แก้เลย แต่ต้อง**ปรับขนาด `max_connections` ของ `PgPoolOptions` ในแอปให้เหมาะกับสถานการณ์ที่มี pooler อยู่แล้ว**
(บางครั้งตั้ง pool ในแอปให้เล็กลง เพราะ pooler ด้านหลังทำหน้าที่ "กันชน" ให้แล้ว ไม่ต้องให้แอปเปิด connection
มากเกินจำเป็นซ้ำอีกชั้น) — นี่คือรายละเอียดที่ต้องอ่านเอกสารของ pooler แต่ละตัวเพิ่มเติมตอน tune production
จริง ไม่ใช่ค่าที่ตั้งครั้งเดียวแล้วใช้ได้ตลอดไปโดยไม่ต้องคิดอะไรเพิ่ม

**เกี่ยวกับการตรวจสอบจริง**: สภาพแวดล้อมที่เขียนบทนี้ไม่มี connection string ของ managed database ใด ๆ ที่
verified มาจาก Part ก่อนหน้า (Part 70/96 ใช้ PostgreSQL ที่รันเองผ่าน `docker-compose` ในเครื่อง ไม่ใช่ managed
service บนคลาวด์จริง) ตามเงื่อนไขของบทนี้ที่ระบุว่า **ให้ตรวจสอบการเชื่อมต่อจริงเฉพาะกรณีที่มี connection
string อยู่แล้วจาก Part ก่อนหน้าเท่านั้น** — เนื้อหาของหัวข้อนี้จึงเป็นข้อมูลอ้างอิงเชิงเปรียบเทียบ (feature/
free tier ตามที่แต่ละ provider ประกาศไว้บนเว็บไซต์ทางการ) ไม่ใช่ผลจากการต่อจริง

### 101.10 Zero-Downtime Deployment: Rolling Deployment และ Health-Check-Gated Cutover

**ปัญหาที่ zero-downtime deployment แก้**: ถ้า deploy โค้ดใหม่ด้วยวิธีง่ายที่สุด (หยุด instance เก่าทั้งหมด
→ start instance ใหม่ทั้งหมด) จะมี**ช่วงเวลาที่ไม่มี instance ไหนพร้อมรับ traffic เลย** (ตั้งแต่ instance
เก่าหยุดจนกว่า instance ใหม่จะ start+initialize เสร็จ) — request ที่เข้ามาช่วงนั้นจะ fail หมด ผู้ใช้เห็น error
ทันที — สำหรับ production service ที่มีผู้ใช้จริงตลอดเวลา นี่คือ downtime ที่ยอมรับไม่ได้แม้แค่ไม่กี่วินาที
ก็ตาม (ยิ่งถ้า deploy บ่อย เช่นหลาย ๆ ครั้งต่อวันตาม CI/CD ที่ Part 97 สอน downtime สะสมจะมากขึ้นเรื่อย ๆ)

**Rolling Deployment คือทางแก้**: แทนที่จะหยุด instance ทั้งหมดพร้อมกัน ให้ **ทดแทนทีละตัว**: (1) start
instance ใหม่ (ยังไม่รับ traffic จริง), (2) รอให้ instance ใหม่ผ่าน **health check** (พิสูจน์ว่าพร้อมรับ
traffic จริง ไม่ใช่แค่ process start สำเร็จ — เชื่อมกับ Part 96 หัวข้อ 96 กับดักข้อ 6 ที่อธิบายความต่างระหว่าง
"container running" กับ "service พร้อมใช้งานจริง"), (3) เริ่ม route traffic บางส่วนไปที่ instance ใหม่ พร้อม
กับ instance เก่ายังรับ traffic อยู่ (ไม่มีช่วงที่ไม่มี instance ไหนพร้อมเลย), (4) หยุด instance เก่าตัวหนึ่ง
(ที่ traffic ไม่ไปแล้ว), (5) ทำซ้ำขั้น 1-4 จนครบทุก instance ถูกทดแทนด้วยเวอร์ชันใหม่ — ตลอดกระบวนการนี้
**มี instance อย่างน้อยหนึ่งตัวพร้อมรับ traffic เสมอ** ทำให้ผู้ใช้ไม่เห็น downtime เลย (อาจเห็น response ที่มา
จากเวอร์ชันเก่าปนกับเวอร์ชันใหม่ช่วงสั้น ๆ ระหว่าง rolling ผ่านไป — เป็นเหตุผลที่ database migration ต้อง
backward-compatible ในช่วง rolling ตามที่ Part 71 พูดถึงหลักการ migration ที่ปลอดภัย)

**Health-check-gated cutover คือหัวใจที่ทำให้ rolling deployment ปลอดภัยจริง**: ขั้นตอนที่ 2 ข้างบน ("รอให้
instance ใหม่ผ่าน health check") คือจุดที่ป้องกัน **การ deploy โค้ดที่มีบั๊กร้ายแรงไปทับ instance ที่ทำงานได้
อยู่แล้ว** — ถ้า instance ใหม่ start ไม่สำเร็จ (crash, connect database ไม่ได้เพราะลืมตั้ง `DATABASE_URL`
secret ตามที่กับดักข้อ 5 จะอธิบาย) health check จะ**ไม่ผ่านตลอดไป** ทำให้ platform (Fly.io/ECS/Kubernetes)
**หยุด rollout อัตโนมัติ**และคง instance เก่าที่ยังทำงานได้ไว้ต่อไป — นี่คือความต่างสำคัญระหว่าง `depends_on`
ธรรมดากับ `condition: service_healthy` ที่ Part 96 หัวข้อ 96 กับดักข้อ 6 อธิบายไว้ ขยายมาสู่ระดับ production
deployment: **"process กำลังรัน" ไม่เท่ากับ "พร้อมรับ traffic"** และการ cutover traffic ต้องเงื่อนไขจากเงื่อนไข
หลังไม่ใช่เงื่อนไขแรก

**ในทางปฏิบัติแต่ละแพลตฟอร์มทำอย่างไร**:

- **Fly.io** — `fly deploy` (default strategy คือ `rolling`) ใช้ `[[http_service.checks]]` ใน `fly.toml`
  (ตามที่หัวข้อ 101.3 อธิบายไว้) เป็นเกณฑ์ตัดสิน — Machine ใหม่ต้องผ่าน check ตาม `path`/`interval`/`timeout`
  ที่กำหนดก่อนที่ Fly.io proxy จะเริ่ม route traffic มาให้ ถ้าไม่ผ่านภายในเวลาที่กำหนด deployment จะ**ถูก
  rollback อัตโนมัติ** กลับไปเวอร์ชันก่อนหน้า
- **ECS** — Application Load Balancer (ALB) มี **Target Group Health Check** (คล้ายกันมาก — ยิง HTTP request
  ไปที่ path ที่กำหนดเป็นระยะ) ECS Service ใช้ผลจาก health check นี้ตัดสินว่า task ใหม่ "healthy" แล้วหรือยัง
  ก่อนเริ่ม deregister task เก่าออกจาก target group
- **Kubernetes** — concept เดียวกันเรียกว่า **readiness probe** (ต่างจาก **liveness probe** ที่ใช้ตัดสินว่า
  ต้อง restart container หรือไม่ — readiness probe ใช้ตัดสินว่าจะ route traffic เข้าหรือไม่ เป็นสอง concept
  ที่แยกกันชัดเจนใน Kubernetes แม้ Fly.io/ECS จะรวมสองอย่างนี้เป็น health check เดียวก็ตาม) `Deployment`
  resource ของ Kubernetes มี field `maxUnavailable`/`maxSurge` ควบคุมว่า rolling update จะทดแทนทีละกี่ตัว
  พร้อมกัน (คล้ายกับที่ Fly.io ทดแทนทีละ Machine ตาม default)
- **Cloud Run** — จัดการ rolling deployment ให้อัตโนมัติทั้งหมดโดยไม่ต้องตั้งค่าอะไรเพิ่ม (เพราะ Cloud Run
  เป็น managed service ระดับสูงกว่า ECS/Fly.io ในแง่นี้ — เทรดออฟคือควบคุมรายละเอียดได้น้อยกว่า)

**Connection Draining — รายละเอียดที่มักถูกมองข้ามของขั้นตอน "หยุด instance เก่า"**: กลับไปที่ขั้นตอนที่ 4
ของ rolling deployment ("หยุด instance เก่าตัวหนึ่ง") — ถ้า request ที่กำลังประมวลผลอยู่ ณ ขณะนั้น (เช่น
query ที่ query ช้าไปที่ database, หรือ transaction ที่ยังไม่ commit ตามหลัก `sqlx::Transaction` ที่ Part 70
สอน) ถูก "ตัดตอน" ทันทีที่ instance เก่าถูก `SIGKILL` ผู้ใช้ที่กำลังรอ response นั้นจะได้ error แม้ระบบ
โดยรวมจะดู "zero-downtime" แล้วก็ตาม (เพราะ instance ใหม่พร้อมรับ request ใหม่แล้วจริง แต่ request เก่าที่
กำลังทำอยู่ค้างอยู่ตรงกลาง) — **connection draining** คือขั้นตอนเสริมที่แก้ปัญหานี้: ก่อนส่ง `SIGKILL` ให้
instance เก่า ระบบจะ **หยุดส่ง traffic ใหม่ไปที่ instance นั้น (ถอนออกจาก load balancer/target group ก่อน)
แต่ยังปล่อยให้ request ที่กำลังทำอยู่ทำต่อจนเสร็จ** ภายในเวลาที่กำหนด (grace period) ก่อนค่อยส่ง kill signal
จริง — ตรงกับ `kill_signal`/`kill_timeout` ของ `fly.toml` ที่หัวข้อ 101.3 อธิบายไว้แล้ว (`SIGINT` ให้เวลา
process จัดการ cleanup เอง เช่นรอ request ที่ทำอยู่เสร็จก่อนค่อยยอม exit จริง) — ฝั่งแอปเองก็ต้องรองรับด้วย
graceful shutdown handler (axum ที่ Part 60+ สอนรองรับผ่าน `axum::serve(...).with_graceful_shutdown(...)`)
ไม่ใช่แค่แพลตฟอร์ม deploy ให้เวลามาเฉย ๆ แล้วแอปไม่สนใจ signal ที่ส่งมาเลย

### 101.11 DNS, Custom Domain, และ TLS ในบริบท Managed Platform

Part 100 หัวข้อ 100.8 สอนวิธีตั้งค่า TLS **เองด้วยมือ** ผ่าน `rustls`/`axum-server` (โหลด certificate/key จาก
ไฟล์ PEM เอง) และอธิบายว่า production ต้องใช้ certificate จาก CA จริงผ่าน ACME/Let's Encrypt — หัวข้อนี้
อธิบายว่า **บนแพลตฟอร์ม managed (Fly.io/ECS+ALB/Cloud Run) งานส่วนนี้ถูกทำให้อัตโนมัติเกือบทั้งหมด** แอปไม่ต้อง
รู้จัก `rustls` เลยแม้แต่นิดเดียวสำหรับ TLS ของ public traffic (แต่ยังต้องรู้จัก `rustls` สำหรับกรณีอื่น เช่น
`sqlx` ต่อ managed database ผ่าน TLS ตามที่ Part 70/100 สอน — TLS ของ inbound traffic กับ TLS ของ outbound
connection เป็นคนละเรื่องกัน)

**TLS Termination คืออะไร และทำไม managed platform รับหน้าที่นี้แทนแอปได้**: "TLS termination" คือจุดที่การ
เชื่อมต่อแบบเข้ารหัส (HTTPS) ถูก "ถอดรหัส" กลายเป็น HTTP ธรรมดา — ถ้าทำเองแบบ Part 100 (`axum_server::
bind_rustls`) TLS termination เกิดขึ้น **ในตัวแอปเอง** แต่บน managed platform ส่วนใหญ่ TLS termination เกิดขึ้น
ที่ **load balancer/proxy ของแพลตฟอร์ม** (Fly.io edge proxy, AWS ALB, Google Front End ของ Cloud Run) ก่อน
traffic จะไปถึงแอป — แอปที่อยู่ข้างในจึงรับแค่ HTTP ธรรมดา (ไม่ต้องมี certificate/key ในตัวแอปเลย) การเชื่อมต่อ
ระหว่าง load balancer กับแอป (ภายใน network เดียวกันของแพลตฟอร์ม ไม่ได้เดินทางผ่าน public internet) มักไม่
encrypt ซ้ำอีกชั้น (หรือ encrypt ด้วยกลไกภายในของแพลตฟอร์มเองที่แอปไม่ต้องรู้) — **นี่คือเหตุผลที่ `fly.toml`
หัวข้อ 101.3 ตั้ง `force_https = true` ที่ระดับ `[http_service]` ได้เลยโดยไม่ต้องมี certificate อะไรในโค้ดแอป
หรือใน Dockerfile แม้แต่นิดเดียว**

**Custom domain + certificate อัตโนมัติผ่าน ACME**: เมื่อผูก custom domain (เช่น `api.example.com` แทน
`library-api-mini.fly.dev` default) แพลตฟอร์มเหล่านี้ทำ ACME challenge (protocol เดียวกับที่ Part 100 อธิบาย
ว่า Let's Encrypt ใช้ยืนยันว่าคุณเป็นเจ้าของ domain จริงก่อนออก certificate ให้) **ให้อัตโนมัติทั้งหมด**
ขั้นตอนที่ผู้ใช้ต้องทำมีแค่: (1) เพิ่ม DNS record (`CNAME`/`A` record ชี้ไปที่ endpoint ของแพลตฟอร์ม) ที่ DNS
provider ของตัวเอง (Cloudflare, Route 53, หรือที่ registrar จด domain), (2) รอให้แพลตฟอร์มตรวจ DNS แล้วออก
certificate ให้อัตโนมัติ (มักใช้เวลาไม่กี่นาทีถึงไม่กี่ชั่วโมง) — ตัวอย่าง syntax (documented, ไม่ได้รันจริง):

```bash
# Fly.io — เพิ่ม custom domain (ตาม fly.io/docs)
fly certs add api.example.com
fly certs show api.example.com   # ตรวจสถานะการ provision certificate

# หลังจากนั้นต้องไปเพิ่ม DNS record ที่ DNS provider ของ domain นั้นเอง ตามที่ fly certs show บอกไว้
```

Cloud Run มีกลไกคล้ายกันผ่าน `gcloud run domain-mappings create` ส่วน ECS/ALB ต้องขอ certificate ผ่าน
**AWS Certificate Manager (ACM)** ก่อน (ACM ก็ใช้กลไก ACME-like validation เหมือนกัน แต่เป็นระบบของ AWS เอง
ไม่ใช่ Let's Encrypt ตรง ๆ) แล้วผูก certificate นั้นเข้ากับ ALB listener — แนวคิดเหมือนกันทุกแพลตฟอร์ม: **แยก
ความรับผิดชอบเรื่อง certificate ออกจากแอปโดยสิ้นเชิง** ต่างจาก Part 100 ที่สอนให้แอปรับผิดชอบเองทั้งหมด
(เหมาะกับกรณี deploy บน VM ดิบที่ไม่มี managed proxy ชั้นหน้า)

**DNS record type ที่ต้องรู้จักตอนผูก custom domain**: DNS (Domain Name System) แปลงชื่อ domain ที่มนุษย์
อ่านง่าย (`api.example.com`) เป็น IP address ที่เครื่องใช้จริง — record type หลักที่เจอบ่อยตอนผูก domain
เข้ากับ managed platform:

| Record type | ใช้ทำอะไร | ใช้กับ managed platform อย่างไร |
|---|---|---|
| **A** | ชี้ hostname ไปยัง IPv4 address ตรง ๆ | Fly.io ให้ IPv4 address คงที่มาผูกกับ apex domain (`example.com` ไม่มี subdomain) ได้ |
| **AAAA** | เหมือน A แต่เป็น IPv6 address | เช่นเดียวกับ A แต่สำหรับ IPv6 — Fly.io/Cloudflare ให้ทั้งคู่เสมอเพื่อรองรับ dual-stack |
| **CNAME** | ชี้ hostname ไปยัง hostname อื่น (ไม่ใช่ IP ตรง ๆ) | ใช้กับ subdomain (`api.example.com` → `library-api-mini.fly.dev`) — **ใช้กับ apex domain ไม่ได้ตาม RFC ของ DNS เอง** (ข้อจำกัดที่ DNS provider บางตัวแก้ด้วย record พิเศษของตัวเอง เช่น `ALIAS`/`ANAME` ของ Cloudflare/Route 53) |
| **TXT** | เก็บข้อมูล text ใด ๆ | ใช้ยืนยันความเป็นเจ้าของ domain ตอน ACME challenge บางรูปแบบ (`dns-01` challenge ของ Let's Encrypt ที่ Part 100 พูดถึง ต้องพิสูจน์ผ่าน DNS ไม่ใช่ผ่าน HTTP) |

**DNS propagation delay**: การเปลี่ยน DNS record ไม่มีผลทันทีทั่วโลก เพราะ DNS resolver (ของ ISP/browser/OS)
**cache** ผลลัพธ์ไว้ตามค่า **TTL (Time To Live)** ที่ record นั้นตั้งไว้ (หน่วยเป็นวินาที เช่น TTL 300 วินาที
= cache ไว้ 5 นาทีก่อน resolver จะ query ใหม่) — ถ้าเพิ่งเปลี่ยน record ผลลัพธ์อาจ**ไม่ตรงกันในแต่ละที่**
ชั่วขณะ (บางคนเห็นค่าใหม่ บางคนยังเห็นค่าเก่าที่ resolver ของเขา cache ไว้) จนกว่า TTL เดิมจะหมดอายุทั่วโลก —
เป็นเหตุผลที่คำแนะนำทั่วไปคือ **ลด TTL ลงล่วงหน้าก่อนวันที่วางแผนเปลี่ยน record สำคัญ** (เช่นเปลี่ยนจาก 3600
เป็น 300 ก่อนสองสามวัน) เพื่อให้การเปลี่ยนแปลงจริงมีผลเร็วขึ้นตอนถึงเวลาจริง — ไม่เกี่ยวกับ managed platform
ที่เลือกเลย เป็นธรรมชาติของ DNS protocol เองที่ทุกแพลตฟอร์มต้องเจอเหมือนกัน

### 101.12 Cost Awareness: ลำดับขนาดของค่าใช้จ่าย (Order of Magnitude)

ตัวเลขในหัวข้อนี้เป็น**การประมาณระดับลำดับขนาด** (order of magnitude) จาก public pricing page ของแต่ละ
provider ณ ช่วงที่เขียนบทนี้ **ไม่ใช่ราคาที่ยืนยันแล้วและมีโอกาสเปลี่ยนแปลงได้เสมอ** (ราคา cloud เปลี่ยนบ่อย
และต่างกันตาม region/currency) — เป้าหมายของหัวข้อนี้คือให้เห็น**สัดส่วน**ระหว่างตัวเลือกต่าง ๆ เพื่อ
ประกอบการตัดสินใจ ไม่ใช่ตัวเลขที่เอาไปคำนวณ budget จริงได้เป๊ะ ๆ (ต้องเปิด pricing calculator ของ provider
นั้น ๆ เองเสมอก่อนตัดสินใจจริง):

| ตัวเลือก | ลำดับขนาดค่าใช้จ่าย/เดือน (แอปเล็ก, traffic ต่ำ-ปานกลาง) | หมายเหตุ |
|---|---|---|
| Fly.io `shared-cpu-1x` + 512MB, 1 Machine รันตลอด | **หลักดอลลาร์ (~$2-5)** | ถ้าใช้ `auto_stop_machines` และ traffic น้อย อาจถูกกว่านี้อีกมาก |
| AWS Lambda (traffic ต่ำ, invocation ไม่บ่อย) | **เกือบ $0 ถึงหลักดอลลาร์** | AWS มี free tier ถาวร 1 ล้าน request/เดือนสำหรับ Lambda — งาน demo/side project มักอยู่ในฟรี tier ตลอด |
| ECS Fargate (0.25 vCPU/0.5GB, รันตลอด) | **หลักสิบดอลลาร์ (~$10-20)** | จ่ายตามเวลาที่ task รันจริง คูณด้วยจำนวน task — ถ้าตั้ง desired count 2+ (สำหรับ high availability) ราคาคูณตามจำนวน |
| EC2 `t3.micro`/`t3.small` รันตลอด 24/7 | **หลักสิบดอลลาร์ (~$8-15)** | ยังไม่รวมค่า EBS storage/data transfer/load balancer แยก |
| Managed PostgreSQL ระดับเล็กสุด (RDS/Cloud SQL) | **หลักสิบดอลลาร์ (~$15-30)** | Fly Postgres/Neon free tier มักถูกกว่านี้มากสำหรับโปรเจกต์เล็ก (Neon free tier ใช้งานได้จริงที่ $0) |
| Kubernetes managed control plane (EKS/GKE) | **หลักสิบดอลลาร์ (~$70-75) แค่ค่า control plane** | ยังไม่รวมค่า worker node (EC2/GCE instance) ที่ต้องจ่ายเพิ่มแยกทั้งหมด — เป็นเหตุผลว่าทำไม K8s ไม่คุ้มสำหรับโปรเจกต์เล็ก |

**ตัวอย่างการคำนวณแบบง่าย — เปรียบเทียบ Lambda กับเครื่องที่รันตลอด**: เพื่อให้เห็นภาพลำดับขนาดชัดขึ้น ลอง
ประมาณการแบบคร่าว ๆ (ตัวเลขนี้เป็นการประมาณเพื่อการศึกษา ไม่ใช่ราคาที่ยืนยันแล้ว) สำหรับ Lambda function ที่
ทำงานเฉลี่ย 50ms ต่อ invocation, memory 256MB, เรียก 100,000 ครั้ง/เดือน — เทียบราคาต่อ GB-second ระดับที่
AWS ประกาศไว้บน pricing page ณ ช่วงที่เขียนบทนี้ (หลักการคำนวณคือ `จำนวน invocation × ระยะเวลาเฉลี่ย ×
memory ที่จอง (แปลงเป็น GB) × ราคาต่อ GB-second` บวกค่า request แยกอีกเล็กน้อย) ผลลัพธ์ที่ได้อยู่ใน**ระดับ
เกือบ $0** เพราะ AWS มี free tier ถาวรที่ครอบคลุมปริมาณการใช้งานระดับนี้ทั้งหมด (ไม่ต้องคำนวณละเอียดเพราะยัง
อยู่ในฟรี tier) — ในขณะที่ EC2 `t3.micro` ที่รันตลอด 24 ชั่วโมง × 30 วัน (720 ชั่วโมง) คูณด้วยราคาต่อชั่วโมง
ระดับ $0.01-0.02 ต่อชั่วโมง จะอยู่ที่**ประมาณ $8-15 ไม่ว่าจะมี traffic เข้ามาจริง 100,000 ครั้งหรือ 0 ครั้ง
เลยก็ตาม** เพราะรูปแบบ billing ของ EC2 ไม่สนใจว่ามี traffic จริงหรือไม่ — **นี่คือตัวอย่างที่ชัดที่สุดว่าทำไม
traffic pattern ที่ไม่สม่ำเสมอ (โดยเฉพาะระดับ MVP/side project ที่ traffic ยังน้อย) เหมาะกับ Serverless
มากกว่า VM ที่รันตลอด** แม้ราคาต่อหน่วยงานของ Lambda (คิดเป็น $ ต่อ GB-second) อาจดูสูงกว่าราคาต่อหน่วยงาน
ของ EC2 (คิดเป็น $ ต่อ vCPU-hour) เมื่อเทียบตรง ๆ ก็ตาม เพราะจุดสำคัญไม่ใช่ "ราคาต่อหน่วยงาน" แต่คือ **"จำนวน
หน่วยงานที่เกิดขึ้นจริง" กับ "ค่าใช้จ่ายตอนไม่มีงานเกิดขึ้นเลย"**

**ข้อสังเกตที่สำคัญกว่าตัวเลข**: รูปแบบ billing มีสองแบบหลักที่ต่างกันโดยพื้นฐาน — **"จ่ายตามเวลาที่เครื่อง
รันอยู่"** (EC2, Fargate ถ้าไม่ scale เป็น 0, RDS/Cloud SQL) กับ **"จ่ายตามการใช้งานจริง"** (Lambda ตาม
invocation, Fly.io Machine ที่ตั้ง `auto_stop_machines`, Neon ตาม compute-second ที่ query จริง) — สำหรับ
โปรเจกต์ที่ traffic ไม่แน่นอนหรือยังอยู่ช่วงทดลอง (side project, MVP, demo) **รูปแบบที่สองมักถูกกว่ามาก**
เพราะไม่จ่ายเงินตอน idle เลย ในขณะที่รูปแบบแรกเหมาะกับ workload ที่ traffic สม่ำเสมอสูงพอจนค่าเฉลี่ยต่อชั่วโมง
ของรูปแบบที่สองแพงกว่าค่าเช่าคงที่ของรูปแบบแรก — นี่คือเหตุผลเชิงเศรษฐศาสตร์ (ไม่ใช่แค่เชิงเทคนิค) ที่ทำให้
Serverless/PaaS เหมาะกับสถานการณ์ที่ตาราง decision framework ของหัวข้อ 101.1 ระบุไว้

### 101.13 Capstone: ประกอบ `fly.toml` + `Dockerfile` ฉบับสมบูรณ์สำหรับ `library_api_mini`

หัวข้อนี้รวมทุกอย่างของบทนี้เข้าด้วยกัน เป็น artifact ที่ครบสมบูรณ์สำหรับ deploy capstone จาก Part 92-96 ขึ้น
Fly.io จริง (ถ้าผู้เรียนมี Fly.io account ของตัวเองก็เอาไฟล์เหล่านี้ไปใช้ deploy ได้ตรง ๆ ทันที) — พร้อมระบุ
ให้ชัดเจนที่สุดว่าส่วนไหน**ตรวจสอบจริง**ในสภาพแวดล้อมที่เขียนบทนี้ และส่วนไหน**อ้างอิงจากเอกสารทางการ**

**`Dockerfile.musl`** (ไฟล์เดียวกับที่ Part 96 หัวข้อ 96.10 build/run จริงสำเร็จแล้ว — วัดขนาด image ได้
7.84MB, พิสูจน์ full flow migration→health→register→login ผ่าน `docker-compose` จริงมาแล้ว — บทนี้ไม่ build
ซ้ำ เพราะ Part 96 ทำไปแล้วครบถ้วน):

```dockerfile
# Dockerfile.musl — ตรวจสอบจริงแล้วใน Part 96 หัวข้อ 96.10 (image 7.84MB, ผ่าน full flow จริง)
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

**`fly.toml`** (schema ตรวจสอบ TOML syntax จริงด้วย Python `tomllib` ในสภาพแวดล้อมที่เขียนบทนี้ — field
value อ้างอิงจากเอกสาร Fly.io เพราะไม่มี `flyctl` ตัวจริงให้ยืนยันด้วย `fly config validate`):

```toml
app = "library-api-mini"
primary_region = "sin"
kill_signal = "SIGINT"
kill_timeout = "5s"

[build]
  dockerfile = "Dockerfile.musl"

[env]
  RUST_LOG = "info"
  BIND_ADDR = "0.0.0.0:8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 1
  processes = ["app"]

  [[http_service.checks]]
    grace_period = "10s"
    interval = "15s"
    method = "GET"
    timeout = "3s"
    path = "/health"

[[vm]]
  size = "shared-cpu-1x"
  memory = "512mb"
```

**ขั้นตอน deploy เต็มรูปแบบ (documented — ไม่ได้รันจริงในบทนี้ เพราะต้องมี Fly.io account จริงที่สภาพแวดล้อม
นี้ไม่มี)**:

```bash
# 1. สมัคร/login (ต้อง account จริง — ข้ามไม่ได้)
fly auth login

# 2. สร้าง App ตาม fly.toml ที่เขียนไว้แล้ว โดยยังไม่ deploy ทันที
fly launch --no-deploy

# 3. ตั้ง secret ที่จำเป็นก่อน deploy ครั้งแรก (DATABASE_URL/JWT_SECRET ต้องมีก่อน
#    ไม่งั้น Machine แรกจะ crash วนตั้งแต่ start เพราะต่อ database ไม่ได้ —
#    ดูกับดักที่พบบ่อยข้อ 5 ของบทนี้)
fly secrets set \
  DATABASE_URL="postgres://user:pass@<managed-db-host>:5432/library_mini" \
  JWT_SECRET="$(openssl rand -base64 32)"

# 4. (ทางเลือก) ถ้าใช้ Fly Postgres เป็น managed database — สร้างและผูกอัตโนมัติ
#    fly postgres create จะสร้าง DATABASE_URL secret ให้เองโดยไม่ต้องทำขั้นตอนที่ 3 เอง
# fly postgres create --name library-api-mini-db
# fly postgres attach library-api-mini-db -a library-api-mini

# 5. Deploy จริง — build image จาก Dockerfile.musl, rolling deployment ตามหัวข้อ 101.10
fly deploy

# 6. ตรวจสอบสถานะและ log
fly status
fly logs
```

**สิ่งที่ capstone นี้พิสูจน์และสิ่งที่ยังไม่ได้พิสูจน์ — สรุปให้ชัดที่สุด**:

- ✅ **พิสูจน์จริงแล้ว (Part 96)**: `Dockerfile.musl` build สำเร็จ, image รันได้จริง, full flow
  (migration→health→register→login) ทำงานถูกต้อง 100% ผ่าน `docker-compose` กับ PostgreSQL จริง
- ✅ **พิสูจน์จริงแล้ว (บทนี้)**: `fly.toml` TOML syntax ถูกต้องตาม parser มาตรฐาน, `cargo lambda`
  build+watch+invoke ทำงานได้จริงแบบ offline สมบูรณ์กับ Lambda function แยกตัวอย่าง
- 📖 **อ้างอิงจากเอกสารทางการเท่านั้น**: field value ทั้งหมดของ `fly.toml` ตรงตาม schema จริงของ Fly.io
  หรือไม่ (ต้องมี `flyctl`+account จริงยืนยันด้วย `fly config validate`/`fly deploy`), พฤติกรรมจริงของ
  `fly deploy`/rolling deployment/health check cutover บน infrastructure จริงของ Fly.io, ราคาที่เกิดขึ้น
  จริงหลัง deploy

**Checklist หลัง deploy ครั้งแรก — ตรวจอะไรก่อนบอกว่า "deploy สำเร็จแล้ว"**: อย่าถือว่า deployment สำเร็จ
แค่เพราะคำสั่ง `fly deploy`/`aws ecs update-service`/`gcloud run deploy` คืน exit code 0 — ต้องตรวจต่อไปนี้
ให้ครบก่อนสรุปว่าระบบพร้อมรับผู้ใช้จริง (สังเคราะห์จากทุกหัวข้อของบทนี้):

1. **Health check ผ่านจริง** — `fly status`/AWS Console (ECS Service Events tab)/`gcloud run services
   describe` ต้องแสดงสถานะ "healthy"/"steady state" ไม่ใช่แค่ "running" (ตามหลัก health-check-gated cutover
   ของหัวข้อ 101.10 — "process รันอยู่" ไม่เท่ากับ "พร้อมรับ traffic")
2. **Log ไม่มี error ที่ผิดปกติ** — `fly logs`/CloudWatch Logs (สำหรับ ECS/Lambda)/Cloud Logging (สำหรับ
   Cloud Run) — ทุกแพลตฟอร์มมีระบบรวบรวม log จาก stdout/stderr ของ container ให้อัตโนมัติ (ไม่ต้องตั้งค่า
   log shipping เองเหมือน VM ดิบที่ต้องพึ่งเครื่องมือแยก เช่น Fluentd/Vector) — ตรวจว่าไม่มี panic/connection
   error ซ้ำ ๆ ที่บ่งบอกว่า Machine "healthy" แค่ผ่าน HTTP check แต่ยังมีปัญหาซ่อนอยู่ (เช่น database
   connection pool เต็มบ่อย ๆ)
3. **Secret ครบและถูกต้อง** — ยืนยันว่า endpoint ที่ต้องพึ่ง database (เช่น `/health` ที่เช็ค `database:
   "ok"` ตามที่ Part 96 หัวข้อ 96.10 ออกแบบไว้) คืนค่าถูกต้องจริง ไม่ใช่แค่ HTTP 200 เปล่า ๆ ที่ไม่ได้เช็ค
   dependency ภายในเลย (ย้อนกลับไปหลักการ "readiness ต้องเช็คสิ่งที่แอปพึ่งพาจริง" ของ Part 81)
4. **TLS certificate ออกสำเร็จแล้ว** (ถ้าผูก custom domain ตามหัวข้อ 101.11) — เช็คด้วย `curl -vI
   https://<custom-domain>` ว่า handshake สำเร็จและ certificate ไม่ใช่ self-signed/expired
5. **Rollback plan พร้อมใช้** — รู้ว่าคำสั่ง rollback ของแพลตฟอร์มนั้นคืออะไร (`fly deploy` เวอร์ชันก่อนหน้า
   ซ้ำ, ECS "roll back to previous task definition revision", Lambda "point alias ไปเวอร์ชันก่อนหน้า") ก่อน
   ที่จะต้องใช้จริงตอนมีปัญหาฉุกเฉิน ไม่ใช่มาหาวิธีตอนเกิดปัญหาแล้ว

การแยกสถานะแบบนี้คือสิ่งที่บทนี้ต้องการให้ผู้เรียนติดเป็นนิสัย: **เมื่อไรก็ตามที่ทำงานในสภาพแวดล้อมที่ไม่มี
credential จริงของ cloud provider (เช่น CI runner ที่ยังไม่ผูก secret, sandbox สำหรับพัฒนา, เครื่องส่วนตัวที่
ยังไม่สมัคร account) ต้องรู้ตัวเสมอว่าขั้นไหน "ตรวจสอบได้จริงแบบ offline" (syntax, local emulator, unit
test) และขั้นไหน "ต้องพึ่งเอกสารทางการ/ต้องทดสอบกับ account จริงก่อน production" — การปนสองอย่างนี้เข้าด้วย
กันโดยไม่แยกแยะคือที่มาของ incident จำนวนมากในโลกจริงที่ "ทดสอบผ่านในเครื่อง" แต่ไม่เคยพิสูจน์กับ
infrastructure จริงเลยก่อน deploy ครั้งแรก

## กับดักที่พบบ่อย (Common Pitfalls)

**1. `cargo lambda watch` เงียบ ๆ จบการทำงานทันทีโดยไม่มี ERROR log ในสภาพแวดล้อมที่ไม่มี IPv6**

พบจริงระหว่างเขียนบทนี้ — รัน `cargo lambda watch` เฉย ๆ (ไม่ระบุ address) แล้ว process จบทำงานภายในไม่ถึง
วินาที:

```
 INFO starting Runtime server runtime_addr=[::]:9000
 INFO terminating lambda scheduler
```

ไม่มี log ระดับ ERROR ปรากฏเลย แม้เปิด `-vv`/`RUST_LOG=debug` — ตรวจสอบด้วยการลอง bind socket คู่กันด้วย
Python เปล่า ๆ พบสาเหตุจริง:

```python
>>> import socket
>>> s = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
OSError: [Errno 97] Address family not supported by protocol
```

**เหตุผล**: `cargo lambda watch` ค่า default bind ที่ `[::]:9000` (IPv6 "any address") — สภาพแวดล้อมที่ไม่มี
IPv6 stack เลย (container บางประเภทที่ปิด IPv6 ไว้โดย default, บาง CI runner, sandbox ที่เขียนบทนี้) จะ bind
ล้มเหลวทันที และดูเหมือน `cargo lambda` จัดการ error นี้แบบ "จบการทำงานเงียบ ๆ" มากกว่าจะ panic/print error
ชัดเจน — **วิธีแก้**: ระบุ `-A 127.0.0.1` (หรือ `-A 0.0.0.0`) บังคับให้ bind ด้วย IPv4 เสมอ:

```bash
cargo lambda watch -A 127.0.0.1 -P 9000
```

บทเรียนที่กว้างกว่าเรื่อง Lambda: **เครื่องมือ dev-server จำนวนมากเลือก bind ที่ IPv6 any-address เป็น
default** (เพราะปกติ dual-stack socket รับทั้ง IPv4 และ IPv6 พร้อมกันในเครื่องที่มี IPv6 ตามปกติ) — ถ้าเจอ
เครื่องมือใดที่ "เงียบ ๆ ไม่ทำงาน" โดยไม่มี error ชัดเจน ให้สงสัย IPv6 เป็นอันดับแรกในสภาพแวดล้อมที่ควบคุม
network เข้มงวด (container/sandbox/CI ที่ปิด IPv6)

**2. `cargo lambda invoke` connect ไม่ติดด้วย error เดียวกัน แม้ตั้งค่า `watch` ให้ bind IPv4 ถูกแล้ว**

หลังแก้กับดักข้อ 1 แล้ว (start `watch` ด้วย `-A 127.0.0.1`) ลองรัน `cargo lambda invoke` แบบไม่ระบุ address
ยัง error อยู่:

```
$ cargo lambda invoke --data-ascii '{"name":"test"}'
Error:   × error sending request to the runtime emulator
  ├─▶ error sending request for url (http://[::1]:9000/2015-03-31/functions/_/invocations)
  ├─▶ client error (Connect)
  ├─▶ tcp open error
  ╰─▶ Address family not supported by protocol (os error 97)
```

**เหตุผล**: `cargo lambda invoke` มีค่า default address **ของตัวเอง แยกจาก `watch`** เป็น `::1` (IPv6
loopback) — ต้องระบุ `--invoke-address 127.0.0.1` ให้ตรงกับที่ `watch` bind ไว้ด้วยเช่นกัน (คนละ flag กับที่
แก้ใน `watch` แต่ปัญหาเดียวกัน) แก้แล้วเรียกสำเร็จจริง:

```bash
$ cargo lambda invoke --invoke-address 127.0.0.1 --invoke-port 9000 --data-ascii '{"name":"Rust Course"}'
{"message":"สวัสดีจาก Lambda, Rust Course"}
```

**บทเรียนที่กว้างกว่า**: เมื่อเครื่องมือหนึ่งมีทั้ง "server side" (`watch`) และ "client side" (`invoke`) ที่
ตั้ง address/port แยกกันคนละ flag ต้องเช็คว่าทั้งสองด้านตรงกันเสมอ ไม่ใช่แก้แค่ด้านเดียวแล้วคาดว่าอีกด้านจะรู้
ค่าที่แก้ไปโดยอัตโนมัติ

**3. ลืมตั้ง `fly secrets set` ก่อน deploy ครั้งแรก — Machine เข้า restart loop ตลอดไป**

นี่คือกับดักที่ **เอกสารทางการของ Fly.io เตือนไว้ชัดเจนมาก** (documented — ไม่ได้จำลองจริงในสภาพแวดล้อมนี้
เพราะต้องมี account จริง) แต่สำคัญพอที่ต้องรู้ก่อน deploy ครั้งแรกเสมอ: ถ้า deploy image ของ `library_api_mini`
ที่ต้องการ `DATABASE_URL` ตอน start (เพื่อสร้าง connection pool และรัน migration ตาม Part 71) โดยยังไม่ได้
`fly secrets set DATABASE_URL=...` มาก่อน Machine ใหม่จะ **panic ทันทีตอน start** (เพราะ `sqlx::PgPool::
connect` ล้มเหลว หรือ `.env`/environment variable ที่ไม่มีค่าทำให้ `env::var("DATABASE_URL")` return `Err`)
— Fly.io เห็นว่า process จบการทำงานแบบ error แล้ว**จะพยายาม restart Machine ให้ใหม่อัตโนมัติ** (ตามหลัก
self-healing) แต่ก็ crash อีกด้วยเหตุผลเดิม วนแบบนี้ไปเรื่อย ๆ (restart loop) — **ไม่มี Machine ไหนผ่าน health
check ได้เลย** ทำให้ deployment ค้างอยู่สถานะ "unhealthy" ตลอด **วิธีแก้**: ตั้ง secret ที่จำเป็นทั้งหมด
(`DATABASE_URL`, `JWT_SECRET`) ด้วย `fly secrets set` **ก่อน** รัน `fly deploy` ครั้งแรกเสมอ (ตามลำดับขั้นตอน
ในหัวข้อ 101.13) — บทเรียนที่กว้างกว่า: ตรงกับหลักการ health-check-gated cutover ของหัวข้อ 101.10 — ระบบ
ป้องกันไม่ให้ traffic ไปโดน Machine ที่พังได้ถูกต้องตามที่ออกแบบไว้ (นี่ไม่ใช่บั๊กของ Fly.io แต่เป็นพฤติกรรม
ที่ถูกต้อง — ปัญหาอยู่ที่ลำดับขั้นตอนการตั้งค่าของผู้ deploy เอง)

**4. `internal_port` ใน `fly.toml` ไม่ตรงกับ port ที่แอปฟังจริงตาม `BIND_ADDR` — health check fail ด้วย connection refused**

อีกกับดักที่เอกสาร Fly.io เตือนไว้ (documented): ถ้าตั้ง `internal_port = 8080` ใน `[http_service]` แต่
Dockerfile/environment variable ตั้ง `BIND_ADDR=0.0.0.0:3000` (พลาดจาก config เก่าที่เคยใช้ตอน dev) —
Fly.io proxy จะพยายามต่อ TCP ไปที่ port 8080 ของ Machine แต่ไม่มี process ไหนฟังอยู่จริงที่ port นั้น ทำให้
health check fail ด้วย connection refused ตลอดไป (คล้ายกันกับกับดักข้อ 3 ในผลลัพธ์ที่เห็น คือ deployment
ค้างสถานะ unhealthy แต่**สาเหตุคนละเรื่องกัน**: ข้อ 3 คือแอป crash ตั้งแต่ต้น, ข้อนี้คือแอป**รันอยู่ปกติดี**
แต่ฟัง port ผิดที่ Fly.io มองหา) — **วิธีแก้**: ตรวจให้ `internal_port` ใน `fly.toml` ตรงกับ port จริงที่
แอปฟังเสมอ (ในตัวอย่างของบทนี้คือ 8080 ตรงกับ `BIND_ADDR=0.0.0.0:8080` ทั้งใน `[env]` และ `ENV` ของ
Dockerfile) — บทเรียนที่กว้างกว่า: เชื่อมกับ Part 96 หัวข้อ 96 กับดักข้อ 7 ที่บอกว่า `EXPOSE` ใน Dockerfile
เป็นแค่ documentation ไม่มีผลจริง — **`internal_port` ของ `fly.toml` คนละเรื่องกัน มีผลจริงต่อการ route
traffic** อย่าสับสนสอง concept นี้ว่าเหมือนกัน

**5. พยายามติดตั้ง `flyctl` ในสภาพแวดล้อมที่ network egress ถูกจำกัดด้วย allowlist — ทุกทางเลือกมาตรฐานล้มเหลว
คนละแบบ**

ระหว่างเขียนบทนี้ ลองทั้งสามทางติดตั้ง `flyctl` จริง แล้วเจอปัญหาคนละแบบทั้งสามทาง (บันทึกไว้เพื่อเป็นบทเรียน
เรื่อง "ติดตั้ง CLI tool ในสภาพแวดล้อมที่ network ถูกควบคุมเข้มงวด" ซึ่งเป็นสถานการณ์ที่พบได้จริงใน CI runner
บางประเภท/corporate proxy — ต่อยอดจาก Part 96 หัวข้อ 96 กับดักข้อ 5 ที่เจอปัญหาแนวเดียวกันกับ `apt-get`):

```
$ curl -L https://fly.io/install.sh | sh
curl: (56) CONNECT tunnel failed, response 403
```

ทางที่สอง (`npm install flyctl`) ติดตั้งสำเร็จแบบไม่มี error ใด ๆ เลย **แต่กลับใช้งานไม่ได้** เพราะ package นี้
เป็นแค่ Node.js wrapper library (`exports: { ".": "./src/index.js" }` ไม่มี `bin` field) ที่คาดหวังให้มี
`flyctl` binary จริงติดตั้งแยกอยู่แล้วในระบบให้มันเรียกใช้ — ไม่ได้ดาวน์โหลด binary ให้เองจริง ๆ:

```
$ npx flyctl version
npm error could not determine executable to run
```

ทางที่สาม (`go install github.com/superfly/flyctl@latest` ผ่าน `proxy.golang.org` ที่อยู่ใน allowlist)
ดาวน์โหลด source code ผ่าน Go module proxy สำเร็จ แต่ compile ไม่ผ่าน เพราะ **ทุกเวอร์ชันที่มีต้องการ Go
เวอร์ชันใหม่กว่าที่มีในเครื่อง**:

```
$ go install github.com/superfly/flyctl@latest
go: github.com/superfly/flyctl@latest: github.com/superfly/flyctl@v0.4.108 requires go >= 1.26.3
(running go 1.24.7; GOTOOLCHAIN=local)
```

**เหตุผลรวม**: allowlist ของสภาพแวดล้อมนี้อนุญาตแค่ `index.crates.io`/`pypi.org`/`registry.npmjs.org`/
`proxy.golang.org` (โดเมนที่เกี่ยวกับ package registry มาตรฐาน) แต่**ไม่อนุญาต `fly.io` หรือ `github.com`**
(โดเมนที่แจกจ่าย binary/release โดยตรง) — `flyctl` เป็น Go binary ที่แจกจ่ายผ่าน GitHub Releases/สคริปต์
ของ Fly.io เอง ไม่ได้ publish เป็น package บน registry มาตรฐานที่ allowlist ครอบคลุม (ต่างจาก `cargo-lambda`
ที่มีคนแพ็กเป็น Python wheel บน PyPI ให้ตามหัวข้อ 101.6) ทำให้ไม่มีทางติดตั้งได้ในสภาพแวดล้อมนี้เลยแม้จะลอง
ครบทุกช่องทางมาตรฐานแล้วก็ตาม — **บทเรียนที่กว้างกว่าเรื่อง Fly.io**: เมื่อเจอสภาพแวดล้อมที่ network ถูก
ควบคุมด้วย allowlist (CI runner บางประเภท, corporate network, sandbox เพื่อความปลอดภัย) ต้องรู้ตัวว่า
"เครื่องมือที่ใช้ package registry มาตรฐานเป็นช่องทางแจกจ่าย" (เช่น `cargo install`, `pip install`, บาง
เครื่องมือ Go ที่ publish source ให้ build จาก module proxy ได้) มีโอกาสติดตั้งสำเร็จสูงกว่า "เครื่องมือที่
แจกจ่าย binary ผ่าน CDN/release page ของตัวเอง" มาก — เป็นเกณฑ์หนึ่งที่ควรพิจารณาตอนเลือกเครื่องมือสำหรับทีม
ที่ทำงานในสภาพแวดล้อมที่ควบคุม network เข้มงวดแบบนี้

**6. `cargo lambda invoke --data-example` ใช้ไม่ได้ในสภาพแวดล้อมที่ network ถูกจำกัด เพราะดึง example
payload จาก server ภายนอก**

พบจริงตอนทดสอบ `lambda_http` ในหัวข้อ 101.6 — flag ที่ดูสะดวกที่สุดสำหรับทดสอบ event จริงของ AWS
(`--data-example apigw-request`) กลับใช้ไม่ได้ในสภาพแวดล้อมที่เขียนบทนี้:

```
$ cargo lambda invoke --data-example apigw-request
Error:   × error dowloading example data
  ├─▶ error sending request for url (https://event-examples.cargo-lambda.info/example-apigw-request.json)
  ├─▶ client error (Connect)
  ╰─▶ unsuccessful tunnel
```

**เหตุผล**: `--data-example` ไม่ได้ฝัง example payload มาในตัว `cargo-lambda` เอง แต่**ดึงจาก server ของ
โปรเจกต์ `cargo-lambda` เองทาง `event-examples.cargo-lambda.info`** ทุกครั้งที่เรียกใช้ — เป็น domain ที่ไม่
อยู่ใน allowlist ของสภาพแวดล้อมนี้ (ต่างจากตัว `cargo-lambda` binary เองที่ pip ดึงจาก `pypi.org` ที่อยู่ใน
allowlist) **วิธีแก้**: เขียน event payload เองเป็นไฟล์ JSON ตรงตาม schema ที่ AWS เอกสารไว้ (เช่น
API Gateway HTTP API payload format version 2.0 ตามตัวอย่างในหัวข้อ 101.6) แล้วใช้ `--data-file` แทน — ใช้
งานได้แน่นอนกว่าเพราะไม่ต้องพึ่ง network ภายนอกเลย และยังเป็นวิธีที่แนะนำสำหรับ CI pipeline ที่ทดสอบ Lambda
function อัตโนมัติ (เก็บ fixture event ไว้ใน repository เอง ไม่ต้องดึงจากที่ไหนทุกครั้งที่รัน test) —
บทเรียนที่กว้างกว่า: **ต้องรู้ว่า flag/feature ไหนของเครื่องมือ "ทำงานแบบ offline จริง" กับไหน "ยังต้องพึ่ง
network เพิ่มเติมที่ไม่ใช่ตัวติดตั้งเครื่องมือเอง"** เพราะสองอย่างนี้มีพฤติกรรมต่างกันมากในสภาพแวดล้อมที่
ควบคุม network เข้มงวด แม้จะมาจากเครื่องมือตัวเดียวกันก็ตาม

## แบบฝึกหัด (Exercises)

1. **(ง่าย)** เขียน `fly.toml` สำหรับแอป axum เล็ก ๆ (แบบ `greet-service` ของ Part 96 หัวข้อ 96.3 ที่มีแค่
   `/`, `/health`) ที่ฟังที่ port 3000 (ไม่ใช่ 8080 แบบตัวอย่างในบทนี้) — ต้องตั้ง `internal_port` ให้ตรงกับ
   port จริงที่แอปฟัง และเพิ่ม `[[http_service.checks]]` ที่เช็ค `/health` (hint: ตรวจสอบ syntax TOML ของ
   ไฟล์ที่เขียนด้วย Python `python3 -c "import tomllib; tomllib.load(open('fly.toml','rb'))"` แบบเดียวกับที่
   บทนี้ทำ ก่อนเชื่อว่าไฟล์ถูกต้อง)
2. **(กลาง)** เขียน Lambda function ด้วย `lambda_runtime` ที่รับ event เป็น `{"a": i32, "b": i32,
   "operation": "add"|"multiply"}` แล้วคืนผลลัพธ์การคำนวณ พร้อม error handling ที่ถูกต้อง (ถ้า `operation`
   เป็นค่าอื่นที่ไม่รู้จัก ต้องคืน `Err` ไม่ใช่ panic) — build และ invoke ผ่าน `cargo lambda build`/`watch`/
   `invoke` ให้สำเร็จจริงในเครื่องแบบเดียวกับหัวข้อ 101.6 (hint: ระวังปัญหา IPv6 ตามกับดักข้อ 1-2 ถ้าเครื่อง
   ทดสอบไม่มี IPv6 ให้ระบุ `-A 127.0.0.1` ทั้งฝั่ง `watch` และ `invoke`)
3. **(ยาก)** เขียนตารางเปรียบเทียบต้นทุนแบบละเอียดกว่าหัวข้อ 101.12 สำหรับสถานการณ์เจาะจง: แอป capstone ของ
   Part 92-96 ที่มี traffic เฉลี่ย 10 request/วินาที ตลอด 24 ชั่วโมง เทียบสามตัวเลือก (Fly.io 2 Machine,
   ECS Fargate desired count 2, Lambda+API Gateway) — เปิด pricing calculator ของแต่ละ provider จริง (Fly.io
   pricing page, AWS Pricing Calculator, GCP Pricing Calculator) แล้วกรอกตัวเลขให้ตรงสถานการณ์ สรุปว่าตัวเลือก
   ไหนถูกที่สุดสำหรับ traffic pattern นี้โดยเฉพาะ และอธิบายว่าทำไมคำตอบจะเปลี่ยนไปถ้า traffic pattern เป็นแบบ
   spike สั้น ๆ (เช่น 1000 req/s แค่ 1 นาทีต่อวัน) แทน (hint: จุดตัดสำคัญคือ traffic ที่ค่าเฉลี่ยของรูปแบบ
   "จ่ายตามการใช้งานจริง" แพงเกินรูปแบบ "จ่ายตามเวลาที่เครื่องรัน" อยู่ที่ไหน)
4. **(ยาก/ประยุกต์)** ประกอบ CI/CD pipeline (ต่อยอดจาก Part 97) ที่ build image จาก `Dockerfile.musl` ของ
   หัวข้อ 101.13 แล้ว deploy ขึ้น Fly.io อัตโนมัติทุกครั้งที่ push เข้า branch `main` โดยใช้ GitHub Actions
   step `flyctl deploy` (ต้องสร้าง Fly.io API token แล้วเก็บเป็น GitHub Actions secret ชื่อ `FLY_API_TOKEN`
   ตามเอกสารทางการของ Fly.io) — ถ้าไม่มี Fly.io account จริงให้ทำแค่เขียน YAML workflow file ให้ถูกต้องตาม
   syntax (ตรวจสอบด้วย YAML parser เหมือนที่บทนี้ตรวจ TOML) และอธิบายเป็นลายลักษณ์อักษรว่าแต่ละ step ทำอะไร
   พร้อมระบุชัดเจนว่าส่วนไหนของคำตอบ "ตรวจสอบ syntax จริงแล้ว" กับส่วนไหน "อ้างอิงจากเอกสารเท่านั้นเพราะไม่มี
   account จริงให้ทดสอบ" (hint: ใช้ `secrets.FLY_API_TOKEN` context ของ GitHub Actions และ Fly.io GitHub
   Action ที่ชื่อ `superfly/flyctl-actions` ตามเอกสารทางการ)

## สรุป

บทนี้พาแอป Rust ที่ containerize ไว้แล้วตั้งแต่ Part 96 ไปสู่คลาวด์จริง ผ่านสามแนวทางหลัก: **PaaS ที่เรียบง่าย
(Fly.io)** สำหรับทีมที่ต้องการ ship เร็วโดยไม่ต้องดูแล infrastructure เอง, **Serverless (AWS Lambda)** สำหรับ
workload แบบ event-driven ที่ traffic ไม่สม่ำเสมอ (พิสูจน์ด้วยการ install `cargo-lambda` จริง, build จริง,
และ invoke local emulator จริงสำเร็จ 100% แบบ offline สมบูรณ์ ไม่ต้องมี AWS account เลยตลอดกระบวนการพัฒนา),
และ **Container orchestration ระดับ enterprise (ECS/Fargate, Kubernetes)** ในเชิงแนวคิดสำหรับทีมที่ต้องการ
ควบคุมละเอียดกว่าและมี operational capacity รองรับความซับซ้อนนั้น — อธิบาย `fly.toml` schema ครบทุก field
หลักพร้อมเหตุผลของแต่ละค่า, เปรียบเทียบ managed database (RDS/Cloud SQL/Fly Postgres/Neon/Supabase),
อธิบายกลไก zero-downtime deployment ที่ผูก rolling deployment เข้ากับ health-check-gated cutover อย่างแนบ
สนิท (ต่อยอดจากหลัก `service_healthy` ของ Part 96), และปิดท้ายด้วย capstone ที่ประกอบ `fly.toml` +
`Dockerfile.musl` ฉบับสมบูรณ์สำหรับ `library_api_mini` — ที่สำคัญที่สุด บทนี้ตั้งใจแสดงให้เห็น**ความแตกต่าง
ระหว่างสิ่งที่ตรวจสอบได้จริงแบบ offline (syntax validation, local emulator) กับสิ่งที่ต้องพึ่งเอกสารทางการ/
ต้องมี credential จริง** อย่างตรงไปตรงมาตลอดทั้งบท (รวมถึงบันทึกความล้มเหลวจริงสามแบบตอนพยายามติดตั้ง
`flyctl` ในสภาพแวดล้อมที่ network ถูกจำกัด) เพื่อให้ผู้เรียนติดเป็นนิสัยในการแยกแยะสองสิ่งนี้เมื่อทำงานกับ
cloud provider ในสถานการณ์จริงที่หลากหลาย

Part ถัดไป (Part 102) จะเปลี่ยนทิศทางไปสู่โลกที่ต่างจาก web service/cloud โดยสิ้นเชิง — **Embedded Rust**
การเขียน Rust สำหรับ microcontroller ที่ไม่มี OS รองรับเลย (`no_std`), หน่วยความจำวัดเป็น KB ไม่ใช่ GB, และ
ไม่มี cloud หรือ container ใด ๆ ให้ deploy ไปหา — เป็นอีกด้านสุดขั้วของ spectrum การ deploy ที่บทนี้เปิดไว้ที่
ปลายด้าน "คลาวด์เต็มรูปแบบ"

---

**Part ก่อนหน้า:** [Security Best Practices ใน Rust](part-100-security-best-practices.md) | **Part ถัดไป:** [Embedded Rust เบื้องต้น](part-102-embedded-rust.md)
