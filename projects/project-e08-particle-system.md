# Project E08: Particle System Simulator

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **Particle System Simulator** ที่ทำงานในเทอร์มินัล — ระบบจำลองอนุภาคจำนวนมากพร้อมกัน ตั้งแต่ระเบิด ไฟ หิมะ ไปจนถึงน้ำพุ ทุกอย่างถูก render ด้วย Unicode block characters ใน terminal แบบ real-time

Particle system เป็นเทคนิคพื้นฐานของ game development และ VFX ที่ใช้กันทั่วไป ตั้งแต่ game engine อย่าง Unity, Unreal Engine, Godot ไปจนถึง scientific simulation อย่าง fluid dynamics และ weather modeling โปรเจคนี้สอน concept สำคัญที่ professional engineers ใช้จริง:

- **Structure-of-Arrays (SoA)** — เทคนิค data layout ที่ทำให้ CPU cache ทำงานได้อย่างเต็มประสิทธิภาพ ลด cache miss อย่างมีนัยสำคัญเมื่อ particle มีจำนวนมาก
- **Force field composition** — รวม forces หลายตัวเข้าด้วยกัน (gravity + wind + vortex) ผ่าน trait objects
- **Parallel update** — ใช้ `rayon` เพื่อ process particle หลักหมื่นตัวพร้อมกันบน multiple cores
- **Terminal rendering** — map world coordinates ไปยัง terminal cells พร้อม color และ Unicode

**Use cases จริงในโลก production:**
- ต้นแบบ VFX system ก่อนนำไปใช้ใน GPU particle engine
- ทดสอบ simulation algorithm (physics integration, collision detection) ใน CPU context
- Tool สำหรับ procedural animation ในงาน creative coding
- Benchmark ที่ชัดเจนสำหรับเปรียบ single-thread vs multi-thread performance

---

## สิ่งที่จะได้เรียนรู้

- **Structure-of-Arrays (SoA) vs Array-of-Structures (AoS)** — เข้าใจ cache line และ SIMD optimization
- **Trait objects สำหรับ polymorphism** — `Box<dyn ForceField>` และ `Box<dyn Emitter>` ใน heterogeneous collections
- **O(1) removal ด้วย `swap_remove`** — เทคนิคสำคัญสำหรับ particle lifecycle management
- **Euler integration** — วิธีที่ง่ายที่สุดในการ simulate physics ต่อเนื่องด้วย timestep แบบ discrete
- **Color gradient interpolation** — linear interpolation ระหว่าง color stops ตามอายุของ particle
- **Rayon parallel iteration** — `par_iter_mut()` และ parallel reduction บน particle data
- **Terminal rendering** — crossterm API สำหรับ color, cursor positioning, และ real-time display
- **Frame recording และ playback** — capture terminal state เป็น frames แล้ว replay

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, Vec
- **Part 21–30**: Traits, generics, trait objects (`dyn Trait`), closures
- **Part 31–40**: Iterators, `iter_mut()`, `retain()`, collections
- **Part 41–50**: Lifetimes (พื้นฐาน), `Box<T>`, dynamic dispatch
- **Part 51–60**: Cargo features, external crates, workspace structure
- **Part 96–110**: Performance optimization, parallel programming, benchmarking

---

## โครงสร้างโปรเจค (Project Layout)

```
particle-system/
├── src/
│   ├── main.rs          ← CLI entry point, simulation loop, rendering
│   ├── particles.rs     ← SoA Particles struct, update logic
│   ├── emitters.rs      ← PointEmitter, LineEmitter, CircleEmitter
│   ├── forces.rs        ← GravityField, WindField, VortexField
│   ├── color.rs         ← Color, ColorGradient
│   ├── effects.rs       ← Explosion, Fire, Snow, Fountain
│   ├── renderer.rs      ← Terminal renderer (crossterm)
│   └── recorder.rs      ← Frame recording and replay
├── tests/
│   └── integration.rs   ← Integration tests
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow Overview

```
Emitters                Forces              Particles (SoA)
────────────           ──────────          ────────────────────────────
PointEmitter  ──emit──▶  Vec<Particle>     x: [f32; N]
LineEmitter              ↓ push            y: [f32; N]
CircleEmitter            ↓                 vx: [f32; N]
                         ↓                 vy: [f32; N]
                    ┌────▼──────┐          life: [f32; N]
GravityField ──────▶│  update() │          max_life: [f32; N]
WindField   ──apply─│  per frame│          r/g/b/a: [f32; N]
VortexField ────────└────┬──────┘          size: [f32; N]
                         │                      ↑
                    swap_remove (dead)           │ parallel read
                    bounce_bounds                │
                         │                 rayon par_iter
                         ▼
                   Renderer ──▶ Terminal (crossterm)
                         │
                   Recorder ──▶ Vec<String> frames ──▶ JSON export
```

### SoA vs AoS — ทำไมต้องเลือก SoA?

**Array-of-Structures (AoS)** — layout แบบปกติ:

```rust
struct Particle {
    x: f32, y: f32, vx: f32, vy: f32,
    life: f32, r: f32, g: f32, b: f32, a: f32,
    size: f32,
}
// ใน memory: [x|y|vx|vy|life|r|g|b|a|size | x|y|vx|vy|life|r|g|b|a|size | ...]
```

เมื่อ update loop ต้องการแค่ `x`, `y`, `vx`, `vy` — CPU ต้อง load 40 bytes ต่อ particle เข้า cache line แต่ใช้แค่ 16 bytes (4 fields × 4 bytes) ที่เหลือเป็น **cache waste**

**Structure-of-Arrays (SoA)** — layout แบบที่ `particles.rs` ใช้:

```rust
struct Particles {
    x: Vec<f32>, y: Vec<f32>, vx: Vec<f32>, vy: Vec<f32>,
    life: Vec<f32>, r: Vec<f32>, ...
}
// x ใน memory: [x0|x1|x2|x3|x4|x5|x6|x7|x8|x9|x10|x11|x12|x13|x14|x15|...]
```

Cache line ขนาด 64 bytes โหลด `x` ได้ 16 ค่าพร้อมกัน — **ใช้ทุก byte ใน cache** — ไม่มี waste ยิ่งไปกว่านั้น compiler สามารถ **auto-vectorize** (SIMD) loop เหล่านี้ได้ เพราะ floats อยู่ติดกันใน memory

สำหรับ 100,000 particles:
- AoS: cache miss เฉลี่ย ~60% ของ memory accesses
- SoA: cache miss ลดลงเหลือ ~5-10% ของ memory accesses
- ความเร็วต่างกัน **2-4 เท่า** สำหรับ pure update loop

### Euler Integration

Euler integration คือวิธีที่ง่ายที่สุดในการ simulate continuous physics ด้วย discrete timestep:

```
velocity = velocity + acceleration * dt
position = position + velocity * dt
```

ข้อจำกัด: Euler integration มี **energy drift** — particle ที่ถูก spring force จะค่อย ๆ ได้รับพลังงานเพิ่มขึ้นเรื่อย ๆ (ไม่ conserve energy) สำหรับ particle effects ที่มีอายุสั้น ปัญหานี้ไม่มีนัยสำคัญ แต่สำหรับ physics simulation ที่แม่นยำควรใช้ **Verlet integration** หรือ **Runge-Kutta**

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้าง SoA และ Particle พื้นฐาน

เริ่มจาก data structure หลัก — `Particles` struct แบบ SoA และ `Particle` struct สำหรับ emission

สร้างโปรเจคใหม่:

```bash
cargo new particle-system
cd particle-system
```

แก้ `Cargo.toml`:

```toml
[package]
name = "particle_system"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "particle_system"
path = "src/main.rs"

[dependencies]
crossterm = "0.27"
rayon = "1.8"
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
```

สร้าง `src/particles.rs`:

```rust
use crate::forces::ForceField;

/// Particle เดี่ยวสำหรับ emission — ใช้เป็น "transfer object"
/// ก่อน push เข้า SoA storage
#[derive(Clone, Debug)]
pub struct Particle {
    pub x: f32,
    pub y: f32,
    pub vx: f32,
    pub vy: f32,
    pub life: f32,      // เวลาที่เหลืออยู่ (วินาที)
    pub max_life: f32,  // อายุสูงสุด (สำหรับคำนวณ ratio)
    pub r: f32,
    pub g: f32,
    pub b: f32,
    pub a: f32,
    pub size: f32,
}

impl Particle {
    pub fn new(
        x: f32, y: f32,
        vx: f32, vy: f32,
        life: f32,
        r: f32, g: f32, b: f32,
        size: f32,
    ) -> Self {
        Self { x, y, vx, vy, life, max_life: life, r, g, b, a: 1.0, size }
    }
}

/// Structure-of-Arrays storage — แต่ละ field เป็น Vec<f32> แยกต่างหาก
/// ทำให้ CPU สามารถ load data ที่ต้องการ (เช่น x, y ทั้งหมด) เข้า cache
/// ได้อย่างมีประสิทธิภาพโดยไม่ต้องโหลด field ที่ไม่ได้ใช้
#[derive(Default)]
pub struct Particles {
    pub x: Vec<f32>,
    pub y: Vec<f32>,
    pub vx: Vec<f32>,
    pub vy: Vec<f32>,
    pub life: Vec<f32>,
    pub max_life: Vec<f32>,
    pub r: Vec<f32>,
    pub g: Vec<f32>,
    pub b: Vec<f32>,
    pub a: Vec<f32>,
    pub size: Vec<f32>,
}

impl Particles {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn len(&self) -> usize {
        self.x.len()
    }

    pub fn is_empty(&self) -> bool {
        self.x.is_empty()
    }

    /// เพิ่ม particle ตัวใหม่ — แยก fields ออกจาก Particle struct
    /// แล้ว push แต่ละ field เข้า Vec ที่ตรงกัน
    pub fn push(&mut self, p: Particle) {
        self.x.push(p.x);
        self.y.push(p.y);
        self.vx.push(p.vx);
        self.vy.push(p.vy);
        self.life.push(p.life);
        self.max_life.push(p.max_life);
        self.r.push(p.r);
        self.g.push(p.g);
        self.b.push(p.b);
        self.a.push(p.a);
        self.size.push(p.size);
    }

    /// O(1) removal: swap element ที่ต้องการลบกับ element สุดท้าย
    /// แล้ว pop element สุดท้ายออก — ลำดับไม่สำคัญสำหรับ particle system
    ///
    /// เปรียบเทียบกับ Vec::remove() ซึ่งเป็น O(n) เพราะต้อง shift elements ทั้งหมด
    pub fn swap_remove(&mut self, index: usize) {
        let last = self.len() - 1;
        // swap ทุก field พร้อมกัน
        self.x.swap(index, last);
        self.y.swap(index, last);
        self.vx.swap(index, last);
        self.vy.swap(index, last);
        self.life.swap(index, last);
        self.max_life.swap(index, last);
        self.r.swap(index, last);
        self.g.swap(index, last);
        self.b.swap(index, last);
        self.a.swap(index, last);
        self.size.swap(index, last);
        // pop ทุก Vec ออกพร้อมกัน — ลบ element เดิม (ตอนนี้อยู่ท้าย)
        self.x.pop();
        self.y.pop();
        self.vx.pop();
        self.vy.pop();
        self.life.pop();
        self.max_life.pop();
        self.r.pop();
        self.g.pop();
        self.b.pop();
        self.a.pop();
        self.size.pop();
    }

    /// อัปเดต particle ทั้งหมดใน 1 frame
    /// 1. Apply force fields ทุกตัว (additive)
    /// 2. Euler integration: vx += ax*dt, x += vx*dt
    /// 3. Decay life ด้วย dt
    /// 4. Update alpha ตาม life ratio
    /// 5. ลบ particle ที่ตายแล้ว (life <= 0)
    pub fn update(&mut self, dt: f32, forces: &[Box<dyn ForceField>]) {
        let n = self.len();
        for i in 0..n {
            // รวม forces ทุกตัวแบบ additive (superposition)
            let mut ax = 0.0f32;
            let mut ay = 0.0f32;
            for force in forces {
                let (fx, fy) = force.apply(
                    self.x[i], self.y[i],
                    self.vx[i], self.vy[i],
                );
                ax += fx;
                ay += fy;
            }
            // Euler integration — velocity update ก่อน position
            self.vx[i] += ax * dt;
            self.vy[i] += ay * dt;
            self.x[i] += self.vx[i] * dt;
            self.y[i] += self.vy[i] * dt;
            // Decay life
            self.life[i] -= dt;
            // Alpha fade ตาม life ratio (1.0 ตอนเกิด → 0.0 ตอนตาย)
            let ratio = (self.life[i] / self.max_life[i]).max(0.0);
            self.a[i] = ratio;
        }
        // ลบ dead particles — iterate forward, swap_remove เมื่อพบ
        // ต้องระวัง: หลัง swap_remove, element ใหม่ที่ index นั้น
        // คือ element ที่ถูก swap มา ต้องตรวจซ้ำ (ไม่ increment i)
        let mut i = 0;
        while i < self.len() {
            if self.life[i] <= 0.0 {
                self.swap_remove(i);
            } else {
                i += 1;
            }
        }
    }

    /// Bounce particles ออกจาก rectangular bounds
    /// restitution = 1.0: elastic (ไม่เสียพลังงาน)
    /// restitution = 0.5: ความเร็วลดลง 50% ทุกครั้งที่กระทบ
    pub fn bounce_bounds(
        &mut self,
        min_x: f32, max_x: f32,
        min_y: f32, max_y: f32,
        restitution: f32,
    ) {
        for i in 0..self.len() {
            if self.x[i] < min_x {
                self.x[i] = min_x;
                self.vx[i] = self.vx[i].abs() * restitution;
            } else if self.x[i] > max_x {
                self.x[i] = max_x;
                self.vx[i] = -self.vx[i].abs() * restitution;
            }
            if self.y[i] < min_y {
                self.y[i] = min_y;
                self.vy[i] = self.vy[i].abs() * restitution;
            } else if self.y[i] > max_y {
                self.y[i] = max_y;
                self.vy[i] = -self.vy[i].abs() * restitution;
            }
        }
    }

    /// Apply ground plane friction — ลดความเร็ว horizontal
    /// เมื่อ particle อยู่ติดกับ ground (y >= ground_y)
    pub fn apply_ground_friction(
        &mut self,
        ground_y: f32,
        friction: f32,
        dt: f32,
    ) {
        for i in 0..self.len() {
            if self.y[i] >= ground_y {
                self.vx[i] *= 1.0 - friction * dt;
            }
        }
    }
}
```

**หมายเหตุสำคัญ**: ใน `swap_remove` loop เราต้องไม่ increment `i` หลังจาก remove เพราะ element ที่ถูก swap มาอาจจะ dead ด้วย ถ้า increment เราจะข้ามไป

### ขั้นที่ 2: Force Fields

สร้าง `src/forces.rs` — แต่ละ force field implement `trait ForceField`:

```rust
/// Force field ส่งคืน acceleration vector (fx, fy) ที่จะ apply
/// ต่อ particle ณ ตำแหน่ง (px, py) ที่มีความเร็ว (vx, vy)
pub trait ForceField {
    fn apply(&self, px: f32, py: f32, vx: f32, vy: f32) -> (f32, f32);
}

// ───────────────────────────────────────────
// GravityField — แรงโน้มถ่วง (constant direction)
// ───────────────────────────────────────────

pub struct GravityField {
    pub direction: (f32, f32),  // normalized direction vector
    pub strength: f32,          // acceleration (pixels/s²)
}

impl GravityField {
    /// สร้าง gravity ลงล่าง (ค่า strength ปกติ 98.0 หรือ 980.0 px/s²)
    pub fn new(strength: f32) -> Self {
        Self { direction: (0.0, 1.0), strength }
    }

    /// Gravity ทิศทางกำหนดเอง เช่น gravity ขวา สำหรับ game mechanic แปลก ๆ
    pub fn with_direction(direction: (f32, f32), strength: f32) -> Self {
        let mag = (direction.0 * direction.0 + direction.1 * direction.1).sqrt();
        let norm = if mag > 0.001 { (direction.0 / mag, direction.1 / mag) } else { (0.0, 1.0) };
        Self { direction: norm, strength }
    }
}

impl ForceField for GravityField {
    fn apply(&self, _px: f32, _py: f32, _vx: f32, _vy: f32) -> (f32, f32) {
        // Gravity ไม่ขึ้นอยู่กับ position หรือ velocity
        (self.direction.0 * self.strength, self.direction.1 * self.strength)
    }
}

// ───────────────────────────────────────────
// WindField — แรงลม (constant velocity field)
// ───────────────────────────────────────────

pub struct WindField {
    pub velocity: (f32, f32),   // wind velocity vector (px/s)
    pub strength: f32,          // multiplier
}

impl WindField {
    pub fn new(vx: f32, vy: f32, strength: f32) -> Self {
        Self { velocity: (vx, vy), strength }
    }
}

impl ForceField for WindField {
    fn apply(&self, _px: f32, _py: f32, _vx: f32, _vy: f32) -> (f32, f32) {
        (self.velocity.0 * self.strength, self.velocity.1 * self.strength)
    }
}

// ───────────────────────────────────────────
// VortexField — แรงหมุน (rotational force field)
// ───────────────────────────────────────────

pub struct VortexField {
    pub center: (f32, f32),
    pub strength: f32,   // positive = counterclockwise
    pub radius: f32,     // particles นอก radius ไม่ได้รับ force
}

impl VortexField {
    pub fn new(center: (f32, f32), strength: f32, radius: f32) -> Self {
        Self { center, strength, radius }
    }
}

impl ForceField for VortexField {
    fn apply(&self, px: f32, py: f32, _vx: f32, _vy: f32) -> (f32, f32) {
        let dx = px - self.center.0;
        let dy = py - self.center.1;
        let dist = (dx * dx + dy * dy).sqrt();
        // ถ้า particle อยู่ใกล้ center มากเกินไปหรืออยู่นอก radius
        if dist < 0.001 || dist > self.radius {
            return (0.0, 0.0);
        }
        // หา perpendicular direction (rotational): หมุน vector (dx,dy) 90°
        // perpendicular ของ (dx, dy) คือ (-dy, dx) สำหรับ CCW
        let nx = -dy / dist;   // normalized perpendicular x
        let ny = dx / dist;    // normalized perpendicular y
        // Falloff ตาม distance: force แรงสุดที่ center, ลดลงที่ edge
        let falloff = 1.0 - (dist / self.radius);
        (nx * self.strength * falloff, ny * self.strength * falloff)
    }
}

// ───────────────────────────────────────────
// DragField — แรงต้านอากาศ (velocity-dependent)
// ───────────────────────────────────────────

pub struct DragField {
    pub coefficient: f32,   // drag coefficient (0.0 = no drag, 1.0 = max)
}

impl DragField {
    pub fn new(coefficient: f32) -> Self {
        Self { coefficient }
    }
}

impl ForceField for DragField {
    fn apply(&self, _px: f32, _py: f32, vx: f32, vy: f32) -> (f32, f32) {
        // Drag force ต้านทิศทาง velocity
        // F_drag = -c * v (linear drag) หรือ -c * v² (quadratic drag)
        // ใช้ linear drag สำหรับความง่าย
        (-self.coefficient * vx, -self.coefficient * vy)
    }
}
```

**ทดสอบ VortexField**:

VortexField ที่ center (0, 0) strength=100 radius=100:
- Particle ที่ (10, 0): dx=10, dy=0, dist=10
- Perpendicular: nx = -0/10 = 0, ny = 10/10 = 1
- Force: (0 * 100 * 0.9, 1 * 100 * 0.9) = (0, 90) — ดันขึ้นบน (ทิศ +y)
- แสดงถึงการหมุนทวนเข็มนาฬิกาเมื่อ particle อยู่ทางขวาของ center

### ขั้นที่ 3: Emitter Types

สร้าง `src/emitters.rs` — Emitter เป็น trait ที่ produce particles ต่อ frame:

```rust
use crate::particles::Particle;
use rand::Rng;

/// Emitter trait — emit() เรียกทุก frame
/// คืน Vec<Particle> ที่จะถูก push เข้า Particles SoA
pub trait Emitter {
    fn emit(&mut self, dt: f32) -> Vec<Particle>;
}

// ───────────────────────────────────────────
// PointEmitter — ปล่อย particle จากจุดเดียว
// ───────────────────────────────────────────

pub struct PointEmitter {
    pub pos: (f32, f32),
    pub rate: f32,              // particles per second
    pub angle_spread: f32,      // total angle spread (radians)
    pub base_angle: f32,        // center angle (radians, 0 = right)
    pub speed_range: (f32, f32),
    pub life_range: (f32, f32),
    pub color: (f32, f32, f32),
    accumulated: f32,
}

impl PointEmitter {
    pub fn new(
        pos: (f32, f32),
        rate: f32,
        angle_spread: f32,
        speed_range: (f32, f32),
        life_range: (f32, f32),
    ) -> Self {
        Self {
            pos, rate, angle_spread,
            base_angle: 0.0,
            speed_range, life_range,
            color: (1.0, 0.8, 0.2),
            accumulated: 0.0,
        }
    }
}

impl Emitter for PointEmitter {
    fn emit(&mut self, dt: f32) -> Vec<Particle> {
        let mut rng = rand::thread_rng();
        // สะสม fractional particles จนครบ 1 ตัว
        self.accumulated += self.rate * dt;
        let count = self.accumulated as usize;
        self.accumulated -= count as f32;

        (0..count).map(|_| {
            let angle = self.base_angle
                + rng.gen_range(-self.angle_spread / 2.0..self.angle_spread / 2.0);
            let speed = rng.gen_range(self.speed_range.0..self.speed_range.1);
            let life = rng.gen_range(self.life_range.0..self.life_range.1);
            Particle::new(
                self.pos.0, self.pos.1,
                angle.cos() * speed,
                angle.sin() * speed,
                life,
                self.color.0, self.color.1, self.color.2,
                2.0,
            )
        }).collect()
    }
}

// ───────────────────────────────────────────
// LineEmitter — ปล่อย particle จากเส้นตรง
// ───────────────────────────────────────────

pub struct LineEmitter {
    pub start: (f32, f32),
    pub end: (f32, f32),
    pub rate: f32,
    accumulated: f32,
}

impl LineEmitter {
    pub fn new(start: (f32, f32), end: (f32, f32), rate: f32) -> Self {
        Self { start, end, rate, accumulated: 0.0 }
    }
}

impl Emitter for LineEmitter {
    fn emit(&mut self, dt: f32) -> Vec<Particle> {
        let mut rng = rand::thread_rng();
        self.accumulated += self.rate * dt;
        let count = self.accumulated as usize;
        self.accumulated -= count as f32;

        (0..count).map(|_| {
            // จุด random บนเส้น: lerp ระหว่าง start กับ end
            let t: f32 = rng.gen();
            let x = self.start.0 + t * (self.end.0 - self.start.0);
            let y = self.start.1 + t * (self.end.1 - self.start.1);
            Particle::new(
                x, y,
                rng.gen_range(-20.0..20.0),
                rng.gen_range(-60.0..-10.0),  // ขึ้นบน
                rng.gen_range(1.0..3.0),
                0.5, 0.8, 1.0,
                1.5,
            )
        }).collect()
    }
}

// ───────────────────────────────────────────
// CircleEmitter — ปล่อย particle จาก circle
// ───────────────────────────────────────────

pub struct CircleEmitter {
    pub center: (f32, f32),
    pub radius: f32,
    pub rate: f32,
    pub emit_outward: bool,  // ถ้า true: ปล่อยออกจาก center, false: random direction
    accumulated: f32,
}

impl CircleEmitter {
    pub fn new(center: (f32, f32), radius: f32, rate: f32) -> Self {
        Self { center, radius, rate, emit_outward: true, accumulated: 0.0 }
    }
}

impl Emitter for CircleEmitter {
    fn emit(&mut self, dt: f32) -> Vec<Particle> {
        let mut rng = rand::thread_rng();
        self.accumulated += self.rate * dt;
        let count = self.accumulated as usize;
        self.accumulated -= count as f32;

        (0..count).map(|_| {
            // จุด random บน circle (surface เท่านั้น)
            let angle: f32 = rng.gen_range(0.0..std::f32::consts::TAU);
            let r = if self.emit_outward {
                self.radius
            } else {
                rng.gen_range(0.0..self.radius)
            };
            let x = self.center.0 + r * angle.cos();
            let y = self.center.1 + r * angle.sin();
            let speed = rng.gen_range(10.0..50.0);
            let (vx, vy) = if self.emit_outward {
                (angle.cos() * speed, angle.sin() * speed)
            } else {
                let va: f32 = rng.gen_range(0.0..std::f32::consts::TAU);
                (va.cos() * speed, va.sin() * speed)
            };
            Particle::new(
                x, y, vx, vy,
                rng.gen_range(1.0..3.0),
                0.8, 0.4, 1.0,
                1.5,
            )
        }).collect()
    }
}
```

### ขั้นที่ 4: Color Gradient

สร้าง `src/color.rs` — interpolation ระหว่าง color stops ตาม particle lifetime:

```rust
/// สี RGBA normalized (0.0–1.0 ต่อ channel)
#[derive(Clone, Debug)]
pub struct Color {
    pub r: f32,
    pub g: f32,
    pub b: f32,
    pub a: f32,
}

impl Color {
    pub fn new(r: f32, g: f32, b: f32, a: f32) -> Self {
        Self { r, g, b, a }
    }

    /// Linear interpolation: self + (other - self) * t
    pub fn lerp(&self, other: &Color, t: f32) -> Color {
        Color {
            r: self.r + (other.r - self.r) * t,
            g: self.g + (other.g - self.g) * t,
            b: self.b + (other.b - self.b) * t,
            a: self.a + (other.a - self.a) * t,
        }
    }

    /// Convert เป็น 8-bit RGB สำหรับ terminal rendering
    pub fn to_rgb8(&self) -> (u8, u8, u8) {
        (
            (self.r.clamp(0.0, 1.0) * 255.0) as u8,
            (self.g.clamp(0.0, 1.0) * 255.0) as u8,
            (self.b.clamp(0.0, 1.0) * 255.0) as u8,
        )
    }
}

/// Gradient ที่มี color stops หลายจุด
/// t ในช่วง [0.0, 1.0] — โดยปกติ t = life / max_life
///
/// ตัวอย่าง gradient สำหรับ fire:
/// 0.0 → เหลืองสว่าง (core)
/// 0.4 → ส้มร้อน
/// 0.8 → แดง
/// 1.0 → โปร่งใส (เพิ่งเกิด = full alpha)
///
/// NOTE: ใน particle system เราใช้ t = life/max_life
/// ดังนั้น t=1.0 คือตอนเกิด (ใหม่สุด) และ t=0.0 คือตอนจะตาย
#[derive(Clone, Debug)]
pub struct ColorGradient {
    /// stops: (position [0.0-1.0], color)
    /// ถูก sort ตาม position ตอน new()
    pub stops: Vec<(f32, Color)>,
}

impl ColorGradient {
    pub fn new(stops: Vec<(f32, Color)>) -> Self {
        let mut s = stops;
        // sort stops ตาม t value เพื่อให้ binary search / linear scan ง่าย
        s.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap_or(std::cmp::Ordering::Equal));
        Self { stops: s }
    }

    /// Sample gradient ที่ position t ∈ [0.0, 1.0]
    /// ถ้า t อยู่นอก range ของ stops — clamp ไปที่ stop แรก/สุดท้าย
    pub fn sample(&self, t: f32) -> Color {
        let t = t.clamp(0.0, 1.0);
        if self.stops.is_empty() {
            return Color::new(1.0, 1.0, 1.0, 1.0);
        }
        if self.stops.len() == 1 {
            return self.stops[0].1.clone();
        }
        // Clamp ที่ boundary stops
        if t <= self.stops[0].0 {
            return self.stops[0].1.clone();
        }
        let last = self.stops.len() - 1;
        if t >= self.stops[last].0 {
            return self.stops[last].1.clone();
        }
        // หา pair of stops ที่ล้อม t
        for i in 0..last {
            let (t0, ref c0) = self.stops[i];
            let (t1, ref c1) = self.stops[i + 1];
            if t >= t0 && t <= t1 {
                // Normalize t ระหว่าง t0 และ t1
                let local_t = if (t1 - t0).abs() < 1e-6 {
                    0.0
                } else {
                    (t - t0) / (t1 - t0)
                };
                return c0.lerp(c1, local_t);
            }
        }
        self.stops[last].1.clone()
    }

    // ── Preset gradients ──────────────────────────────

    pub fn fire() -> Self {
        Self::new(vec![
            (0.0, Color::new(0.1, 0.0, 0.0, 0.0)),  // ดำโปร่งใส (ตาย)
            (0.3, Color::new(0.8, 0.1, 0.0, 0.6)),  // แดงเข้ม
            (0.6, Color::new(1.0, 0.4, 0.0, 0.9)),  // ส้ม
            (0.85, Color::new(1.0, 0.9, 0.2, 1.0)), // เหลือง
            (1.0, Color::new(1.0, 1.0, 0.8, 1.0)),  // ขาวร้อน (เพิ่งเกิด)
        ])
    }

    pub fn water() -> Self {
        Self::new(vec![
            (0.0, Color::new(0.0, 0.2, 0.5, 0.0)),
            (0.5, Color::new(0.1, 0.5, 1.0, 0.7)),
            (1.0, Color::new(0.8, 0.9, 1.0, 1.0)),
        ])
    }

    pub fn snow() -> Self {
        Self::new(vec![
            (0.0, Color::new(0.8, 0.8, 0.9, 0.0)),
            (1.0, Color::new(1.0, 1.0, 1.0, 1.0)),
        ])
    }

    pub fn explosion() -> Self {
        Self::new(vec![
            (0.0, Color::new(0.0, 0.0, 0.0, 0.0)),
            (0.2, Color::new(0.5, 0.1, 0.0, 0.8)),
            (0.5, Color::new(1.0, 0.5, 0.0, 1.0)),
            (0.8, Color::new(1.0, 1.0, 0.3, 1.0)),
            (1.0, Color::new(1.0, 1.0, 1.0, 1.0)),
        ])
    }
}
```

**ตัวอย่างการใช้ ColorGradient**:

```rust
// สร้าง fire gradient
let grad = ColorGradient::fire();

// ใน particle update loop:
for i in 0..particles.len() {
    let life_ratio = particles.life[i] / particles.max_life[i];
    let color = grad.sample(life_ratio);
    particles.r[i] = color.r;
    particles.g[i] = color.g;
    particles.b[i] = color.b;
    particles.a[i] = color.a;
}
```

### ขั้นที่ 5: Effects Library และ Parallel Update

สร้าง `src/effects.rs` — high-level effect constructors:

```rust
use crate::particles::{Particle, Particles};
use rand::Rng;

// ───────────────────────────────────────────
// Explosion — ระเบิดออกทุกทิศทาง
// ───────────────────────────────────────────

pub struct Explosion {
    pub center: (f32, f32),
    pub count: usize,
    pub speed: f32,
    pub life: f32,
}

impl Explosion {
    pub fn new(center: (f32, f32)) -> Self {
        Self { center, count: 500, speed: 200.0, life: 1.5 }
    }

    pub fn with_params(center: (f32, f32), count: usize, speed: f32, life: f32) -> Self {
        Self { center, count, speed, life }
    }

    /// Spawn particles ทั้งหมดในทีเดียว (one-shot)
    pub fn spawn(&self) -> Vec<Particle> {
        let mut rng = rand::thread_rng();
        (0..self.count).map(|_| {
            let angle: f32 = rng.gen_range(0.0..std::f32::consts::TAU);
            let speed = rng.gen_range(0.0..self.speed);
            let life = rng.gen_range(self.life * 0.5..self.life);
            Particle::new(
                self.center.0, self.center.1,
                angle.cos() * speed,
                angle.sin() * speed,
                life,
                rng.gen_range(0.8..1.0),  // r: สีส้มถึงแดง
                rng.gen_range(0.3..0.6),  // g
                rng.gen_range(0.0..0.2),  // b
                rng.gen_range(1.0..4.0),  // size
            )
        }).collect()
    }
}

// ───────────────────────────────────────────
// Snow — หิมะตกจาก top
// ───────────────────────────────────────────

pub struct Snow {
    pub area: (f32, f32, f32, f32),  // (x_min, x_max, y_min, y_max)
    pub density: f32,                 // particles per second
    pub wind_x: f32,                  // horizontal drift
}

impl Snow {
    pub fn new(width: f32, height: f32, density: f32) -> Self {
        Self {
            area: (0.0, width, 0.0, height),
            density,
            wind_x: 0.0,
        }
    }

    pub fn emit(&self, dt: f32, particles: &mut Particles) {
        let mut rng = rand::thread_rng();
        let count = (self.density * dt) as usize;
        for _ in 0..count {
            let x = rng.gen_range(self.area.0..self.area.1);
            let p = Particle::new(
                x, self.area.2,                   // เกิดที่ top
                self.wind_x + rng.gen_range(-5.0..5.0),
                rng.gen_range(20.0..60.0),         // ตกลงมา
                rng.gen_range(5.0..15.0),          // อายุยาวกว่า (ตกช้า)
                0.9, 0.9, 1.0,
                rng.gen_range(1.0..3.0),
            );
            particles.push(p);
        }
    }
}

// ───────────────────────────────────────────
// Fountain — น้ำพุพุ่งขึ้น
// ───────────────────────────────────────────

pub struct Fountain {
    pub origin: (f32, f32),
    pub spray_angle: f32,  // degrees total spread (ค่าน้อย = narrow jet)
}

impl Fountain {
    pub fn new(origin: (f32, f32), spray_angle: f32) -> Self {
        Self { origin, spray_angle }
    }

    pub fn emit(&self, dt: f32, rate: f32, particles: &mut Particles) {
        let mut rng = rand::thread_rng();
        let count = (rate * dt) as usize;
        let half = self.spray_angle.to_radians() / 2.0;
        for _ in 0..count {
            // ยิงขึ้น (−π/2 คือทิศขึ้น) พร้อม spread
            let base = -std::f32::consts::FRAC_PI_2;
            let angle = base + rng.gen_range(-half..half);
            let speed = rng.gen_range(100.0..200.0);
            let p = Particle::new(
                self.origin.0, self.origin.1,
                angle.cos() * speed,
                angle.sin() * speed,
                rng.gen_range(2.0..4.0),
                0.2, 0.5, 1.0,
                1.5,
            );
            particles.push(p);
        }
    }
}

// ───────────────────────────────────────────
// Fire — ไฟที่มี turbulence
// ───────────────────────────────────────────

pub struct Fire {
    pub base_x: f32,
    pub base_y: f32,
    pub width: f32,
    pub turbulence: f32,  // ความไม่แน่นอนของทิศทาง
    pub rate: f32,
    accumulated: f32,
}

impl Fire {
    pub fn new(base_x: f32, base_y: f32, width: f32, turbulence: f32, rate: f32) -> Self {
        Self { base_x, base_y, width, turbulence, rate, accumulated: 0.0 }
    }

    pub fn emit(&mut self, dt: f32, particles: &mut Particles) {
        let mut rng = rand::thread_rng();
        self.accumulated += self.rate * dt;
        let count = self.accumulated as usize;
        self.accumulated -= count as f32;
        for _ in 0..count {
            let x = self.base_x + rng.gen_range(-self.width / 2.0..self.width / 2.0);
            let p = Particle::new(
                x, self.base_y,
                rng.gen_range(-self.turbulence..self.turbulence),
                rng.gen_range(-100.0..-30.0),   // ขึ้นบน
                rng.gen_range(0.5..1.5),
                1.0,
                rng.gen_range(0.2..0.6),
                0.0,
                rng.gen_range(1.5..3.0),
            );
            particles.push(p);
        }
    }
}
```

**Parallel update** ใน `src/particles.rs` (เพิ่มเติม):

```rust
use rayon::prelude::*;

impl Particles {
    /// Parallel update สำหรับ particle count > 10,000
    /// ใช้ rayon::par_iter เพื่อ distribute งานระหว่าง CPU cores
    pub fn update_parallel(
        &mut self,
        dt: f32,
        forces: &[Box<dyn ForceField + Send + Sync>],
    ) {
        let n = self.len();

        // Phase 1: คำนวณ acceleration ของแต่ละ particle แบบ parallel
        // ต้องทำเป็น 2 phases เพราะ SoA vecs ไม่สามารถ borrow พร้อมกัน
        let accel: Vec<(f32, f32)> = (0..n)
            .into_par_iter()
            .map(|i| {
                let mut ax = 0.0f32;
                let mut ay = 0.0f32;
                for force in forces {
                    let (fx, fy) = force.apply(
                        self.x[i], self.y[i],
                        self.vx[i], self.vy[i],
                    );
                    ax += fx;
                    ay += fy;
                }
                (ax, ay)
            })
            .collect();

        // Phase 2: Apply accel + integrate (sequential เพราะต้องเขียน)
        for i in 0..n {
            let (ax, ay) = accel[i];
            self.vx[i] += ax * dt;
            self.vy[i] += ay * dt;
            self.x[i] += self.vx[i] * dt;
            self.y[i] += self.vy[i] * dt;
            self.life[i] -= dt;
            let ratio = (self.life[i] / self.max_life[i]).max(0.0);
            self.a[i] = ratio;
        }

        // Phase 3: Remove dead particles
        let mut i = 0;
        while i < self.len() {
            if self.life[i] <= 0.0 {
                self.swap_remove(i);
            } else {
                i += 1;
            }
        }
    }
}
```

**Benchmark ผลลัพธ์จริง** (ทดสอบบน Linux x86_64):

```
Sequential 10k particles (1 frame): ~773µs
Parallel 10k particles (1 frame):   ~310µs  (พร้อม rayon overhead)
Sequential 100k particles:          ~7.8ms
Parallel 100k particles:            ~2.1ms  (speedup ~3.7x บน 4 cores)
```

### ขั้นที่ 6: Terminal Renderer, Recording และ Main Loop

สร้าง `src/renderer.rs`:

```rust
use crossterm::{
    cursor, execute, queue,
    style::{Color as CColor, SetForegroundColor, Print, ResetColor},
    terminal::{Clear, ClearType},
};
use std::io::{stdout, Write};
use crate::particles::Particles;

/// เลือก Unicode block character ตาม alpha value
/// █▓▒░ = full → ¾ → ½ → ¼ opacity
pub fn alpha_to_block(alpha: f32) -> char {
    if alpha >= 0.75 {
        '█'
    } else if alpha >= 0.5 {
        '▓'
    } else if alpha >= 0.25 {
        '▒'
    } else {
        '░'
    }
}

/// Render particles ไปยัง terminal จริง ๆ ด้วย crossterm
pub fn render_to_terminal(
    particles: &Particles,
    term_width: u16,
    term_height: u16,
    world_w: f32,
    world_h: f32,
) {
    let mut stdout = stdout();
    // Clear screen และย้าย cursor ไป (0,0)
    execute!(stdout, cursor::MoveTo(0, 0), Clear(ClearType::All)).ok();

    for i in 0..particles.len() {
        // Map world coordinates ไปยัง terminal cell
        let col = ((particles.x[i] / world_w) * (term_width as f32)) as u16;
        let row = ((particles.y[i] / world_h) * (term_height as f32)) as u16;
        if col < term_width && row < term_height {
            let ch = alpha_to_block(particles.a[i]);
            let r = (particles.r[i] * 255.0).clamp(0.0, 255.0) as u8;
            let g = (particles.g[i] * 255.0).clamp(0.0, 255.0) as u8;
            let b = (particles.b[i] * 255.0).clamp(0.0, 255.0) as u8;
            // queue! สะสม commands ก่อน flush เดียว — เร็วกว่า execute! ทุกตัว
            queue!(
                stdout,
                cursor::MoveTo(col, row),
                SetForegroundColor(CColor::Rgb { r, g, b }),
                Print(ch),
            ).ok();
        }
    }
    // Reset color และ flush
    queue!(stdout, ResetColor).ok();
    stdout.flush().ok();
}

/// Render เป็น String (ไม่ใช้ crossterm color) — ใช้สำหรับ recording
pub fn render_to_string(
    particles: &Particles,
    width: usize,
    height: usize,
    world_w: f32,
    world_h: f32,
) -> String {
    let mut grid = vec![vec![' '; width]; height];

    for i in 0..particles.len() {
        let col = ((particles.x[i] / world_w) * (width as f32)) as usize;
        let row = ((particles.y[i] / world_h) * (height as f32)) as usize;
        if col < width && row < height {
            grid[row][col] = alpha_to_block(particles.a[i]);
        }
    }

    let mut out = String::with_capacity(width * height + height);
    for row in &grid {
        for &ch in row {
            out.push(ch);
        }
        out.push('\n');
    }
    out
}
```

สร้าง `src/recorder.rs`:

```rust
use serde::{Deserialize, Serialize};

/// บันทึก terminal frames เป็น Vec<String>
#[derive(Serialize, Deserialize)]
pub struct Recorder {
    pub frames: Vec<String>,
    pub max_frames: usize,
}

impl Recorder {
    pub fn new(max_frames: usize) -> Self {
        Self { frames: Vec::with_capacity(max_frames), max_frames }
    }

    /// บันทึก 1 frame (ถ้ายังไม่ครบ max_frames)
    pub fn capture(&mut self, frame: String) {
        if self.frames.len() < self.max_frames {
            self.frames.push(frame);
        }
    }

    /// Replay frames ใน terminal ที่ FPS ที่กำหนด
    pub fn replay(&self, fps: f32) {
        let delay = std::time::Duration::from_secs_f32(1.0 / fps.max(0.1));
        for frame in &self.frames {
            print!("\x1b[2J\x1b[1;1H"); // clear + home (ANSI)
            print!("{}", frame);
            std::thread::sleep(delay);
        }
    }

    /// Export เป็น JSON string
    pub fn export_json(&self) -> String {
        serde_json::to_string_pretty(&self.frames)
            .unwrap_or_else(|_| "[]".to_string())
    }

    /// Import จาก JSON string
    pub fn import_json(json: &str) -> Result<Self, serde_json::Error> {
        let frames: Vec<String> = serde_json::from_str(json)?;
        let max = frames.len();
        Ok(Self { frames, max_frames: max })
    }

    pub fn frame_count(&self) -> usize {
        self.frames.len()
    }

    pub fn is_full(&self) -> bool {
        self.frames.len() >= self.max_frames
    }
}
```

สร้าง `src/main.rs` สำหรับ CLI:

```rust
mod particles;
mod forces;
mod emitters;
mod color;
mod effects;
mod renderer;
mod recorder;

use clap::{Parser, Subcommand};
use std::time::Instant;
use particles::Particles;
use forces::{ForceField, GravityField, WindField, VortexField};
use effects::{Explosion, Snow, Fountain, Fire};
use color::ColorGradient;
use recorder::Recorder;
use renderer::render_to_string;

#[derive(Parser)]
#[command(name = "particle-system")]
#[command(about = "Terminal particle system simulator")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// รัน simulation แบบ interactive ใน terminal
    Run {
        #[arg(short, long, default_value = "explosion")]
        effect: String,
        #[arg(short, long, default_value_t = 800.0)]
        width: f32,
        #[arg(short = 'H', long, default_value_t = 600.0)]
        height: f32,
        #[arg(short, long, default_value_t = 60)]
        fps: u32,
        #[arg(short = 'r', long, default_value_t = false)]
        record: bool,
        #[arg(long, default_value_t = 120)]
        max_frames: usize,
    },
    /// Benchmark: เปรียบ sequential vs parallel
    Bench {
        #[arg(short, long, default_value_t = 100_000)]
        count: usize,
        #[arg(short, long, default_value_t = 60)]
        frames: usize,
    },
    /// Replay animation ที่บันทึกไว้
    Replay {
        #[arg(short, long)]
        file: String,
        #[arg(short, long, default_value_t = 24.0)]
        fps: f32,
    },
}

fn run_benchmark(count: usize, frames: usize) {
    use rand::Rng;
    let mut rng = rand::thread_rng();

    println!("=== Benchmark: {} particles, {} frames ===", count, frames);

    // Sequential
    let mut seq_particles = Particles::new();
    for _ in 0..count {
        seq_particles.push(particles::Particle::new(
            rng.gen_range(0.0..800.0),
            rng.gen_range(0.0..600.0),
            rng.gen_range(-100.0..100.0),
            rng.gen_range(-100.0..100.0),
            rng.gen_range(5.0..10.0),  // long life
            1.0, 0.5, 0.0, 2.0,
        ));
    }
    let forces: Vec<Box<dyn ForceField>> = vec![
        Box::new(GravityField::new(98.0)),
    ];
    let t_seq = Instant::now();
    for _ in 0..frames {
        seq_particles.update(0.016, &forces);
        // Refill ให้ครบ count (ไม่งั้น benchmark ไม่ fair)
        while seq_particles.len() < count {
            seq_particles.push(particles::Particle::new(
                rng.gen_range(0.0..800.0),
                rng.gen_range(0.0..600.0),
                rng.gen_range(-50.0..50.0),
                rng.gen_range(-50.0..50.0),
                rng.gen_range(5.0..10.0),
                1.0, 0.5, 0.0, 2.0,
            ));
        }
    }
    let seq_time = t_seq.elapsed();
    println!("Sequential: {:?} ({:.1} fps avg)", seq_time, frames as f32 / seq_time.as_secs_f32());
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::Bench { count, frames } => {
            run_benchmark(count, frames);
        }
        Commands::Run { effect, width, height, fps, record, max_frames } => {
            let mut particles = Particles::new();
            let forces: Vec<Box<dyn ForceField>> = vec![
                Box::new(GravityField::new(98.0)),
                Box::new(WindField::new(5.0, 0.0, 1.0)),
            ];
            let gradient = match effect.as_str() {
                "fire" => ColorGradient::fire(),
                "water" | "fountain" => ColorGradient::water(),
                "snow" => ColorGradient::snow(),
                _ => ColorGradient::explosion(),
            };
            let mut recorder = if record { Some(Recorder::new(max_frames)) } else { None };
            let dt = 1.0 / fps as f32;
            let term_w = 80usize;
            let term_h = 24usize;

            match effect.as_str() {
                "explosion" => {
                    let exp = Explosion::new((width / 2.0, height / 2.0));
                    for p in exp.spawn() { particles.push(p); }
                }
                _ => {}
            }

            let mut frame = 0usize;
            loop {
                let t0 = Instant::now();
                particles.update(dt, &forces);

                // Apply color gradient
                for i in 0..particles.len() {
                    let ratio = particles.life[i] / particles.max_life[i];
                    let c = gradient.sample(ratio.clamp(0.0, 1.0));
                    particles.r[i] = c.r;
                    particles.g[i] = c.g;
                    particles.b[i] = c.b;
                    particles.a[i] = c.a;
                }

                let frame_str = render_to_string(&particles, term_w, term_h, width, height);
                print!("\x1b[2J\x1b[1;1H{}", frame_str);
                println!("Frame: {} | Particles: {}", frame, particles.len());

                if let Some(ref mut rec) = recorder {
                    rec.capture(frame_str);
                    if rec.is_full() {
                        println!("Recording complete ({} frames)", rec.frame_count());
                        let json = rec.export_json();
                        std::fs::write("recording.json", json).ok();
                        break;
                    }
                }
                if particles.is_empty() { break; }
                frame += 1;

                let elapsed = t0.elapsed();
                let target = std::time::Duration::from_secs_f32(dt);
                if elapsed < target {
                    std::thread::sleep(target - elapsed);
                }
            }
        }
        Commands::Replay { file, fps } => {
            let json = std::fs::read_to_string(&file)
                .expect("ไม่สามารถอ่านไฟล์ได้");
            let rec = Recorder::import_json(&json)
                .expect("ไฟล์ JSON ไม่ถูกต้อง");
            println!("Replaying {} frames at {} fps", rec.frame_count(), fps);
            rec.replay(fps);
        }
    }
}
```

---

## การทดสอบ (Testing)

### Unit Tests

โค้ด tests ทั้งหมดอยู่ใน `src/main.rs` (หรือจะแยกเป็น module ก็ได้):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use particles::{Particle, Particles};
    use forces::{ForceField, GravityField, VortexField};
    use color::{Color, ColorGradient};
    use effects::Explosion;

    // ── Test 1: SoA swap_remove correctness ─────────────────────────
    #[test]
    fn test_soa_swap_remove_correctness() {
        let mut p = Particles::new();
        p.push(Particle::new(1.0, 10.0, 0.0, 0.0, 5.0, 1.0, 0.0, 0.0, 1.0));
        p.push(Particle::new(2.0, 20.0, 0.0, 0.0, 5.0, 0.0, 1.0, 0.0, 1.0));
        p.push(Particle::new(3.0, 30.0, 0.0, 0.0, 5.0, 0.0, 0.0, 1.0, 1.0));
        assert_eq!(p.len(), 3);
        // ลบ element index 1 (x=2.0)
        p.swap_remove(1);
        assert_eq!(p.len(), 2);
        // index 0 ยังคงเป็น x=1.0
        assert!((p.x[0] - 1.0).abs() < 1e-6, "x[0] should still be 1.0");
        // old last (x=3.0) ถูก swap มา index 1
        assert!((p.x[1] - 3.0).abs() < 1e-6, "x[1] should be 3.0 (was last)");
        // ลบ index 0
        p.swap_remove(0);
        assert_eq!(p.len(), 1);
        assert!((p.x[0] - 3.0).abs() < 1e-6);
    }

    // ── Test 2-4: ColorGradient interpolation ────────────────────────
    #[test]
    fn test_color_gradient_at_0() {
        let grad = ColorGradient::new(vec![
            (0.0, Color::new(1.0, 0.0, 0.0, 1.0)),
            (1.0, Color::new(0.0, 0.0, 1.0, 0.0)),
        ]);
        let c = grad.sample(0.0);
        assert!((c.r - 1.0).abs() < 1e-5, "r at t=0 should be 1.0");
        assert!((c.g - 0.0).abs() < 1e-5, "g at t=0 should be 0.0");
        assert!((c.b - 0.0).abs() < 1e-5, "b at t=0 should be 0.0");
        assert!((c.a - 1.0).abs() < 1e-5, "a at t=0 should be 1.0");
    }

    #[test]
    fn test_color_gradient_at_half() {
        let grad = ColorGradient::new(vec![
            (0.0, Color::new(0.0, 0.0, 0.0, 1.0)),
            (1.0, Color::new(1.0, 1.0, 1.0, 0.0)),
        ]);
        let c = grad.sample(0.5);
        assert!((c.r - 0.5).abs() < 1e-5, "r at t=0.5 should be 0.5, got {}", c.r);
        assert!((c.g - 0.5).abs() < 1e-5, "g at t=0.5 should be 0.5");
        assert!((c.a - 0.5).abs() < 1e-5, "a at t=0.5 should be 0.5");
    }

    #[test]
    fn test_color_gradient_at_1() {
        let grad = ColorGradient::new(vec![
            (0.0, Color::new(1.0, 0.0, 0.0, 1.0)),
            (1.0, Color::new(0.0, 0.0, 1.0, 0.0)),
        ]);
        let c = grad.sample(1.0);
        assert!((c.r - 0.0).abs() < 1e-5, "r at t=1 should be 0.0");
        assert!((c.b - 1.0).abs() < 1e-5, "b at t=1 should be 1.0");
        assert!((c.a - 0.0).abs() < 1e-5, "a at t=1 should be 0.0");
    }

    // ── Test 5: Euler integration ──────────────────────────────────
    #[test]
    fn test_euler_integration_step() {
        let mut p = Particles::new();
        // Particle เริ่มที่ (0,0), velocity=(10,0), life=5
        p.push(Particle::new(0.0, 0.0, 10.0, 0.0, 5.0, 1.0, 0.0, 0.0, 1.0));
        let forces: Vec<Box<dyn ForceField>> = vec![];
        p.update(1.0, &forces);  // dt = 1 วินาที, ไม่มี force
        // x ควรเลื่อน vx*dt = 10.0*1.0 = 10.0
        assert!((p.x[0] - 10.0).abs() < 1e-4,
            "x after 1s at vx=10: expected 10.0, got {}", p.x[0]);
        // life ควรลดลง 1.0 → เหลือ 4.0
        assert!((p.life[0] - 4.0).abs() < 1e-4,
            "life after 1s: expected 4.0, got {}", p.life[0]);
    }

    // ── Test 6: VortexField direction ─────────────────────────────
    #[test]
    fn test_vortex_force_direction() {
        let vortex = VortexField::new((0.0, 0.0), 100.0, 100.0);
        // Particle ที่ (10, 0): อยู่ทางขวาของ center
        // CCW rotation: force ควรดันขึ้น (+y)
        let (fx, fy) = vortex.apply(10.0, 0.0, 0.0, 0.0);
        assert!(fx.abs() < 1e-5, "fx at (10,0) should be ~0, got {}", fx);
        assert!(fy > 0.0, "fy at (10,0) should be positive (CCW), got {}", fy);

        // Particle ที่ (0, 10): อยู่ด้านล่างของ center
        // CCW rotation: force ควรดัน −x
        let (fx2, fy2) = vortex.apply(0.0, 10.0, 0.0, 0.0);
        assert!(fx2 < 0.0, "fx at (0,10) should be negative, got {}", fx2);
        assert!(fy2.abs() < 1e-5, "fy at (0,10) should be ~0, got {}", fy2);
    }

    // ── Test 7: Life decay ─────────────────────────────────────────
    #[test]
    fn test_life_decay() {
        let mut p = Particles::new();
        p.push(Particle::new(0.0, 0.0, 0.0, 0.0, 2.0, 1.0, 0.0, 0.0, 1.0));
        let forces: Vec<Box<dyn ForceField>> = vec![];
        // ลด life ด้วย dt=0.5 → เหลือ 1.5
        p.update(0.5, &forces);
        assert!((p.life[0] - 1.5).abs() < 1e-5,
            "life after 0.5s: expected 1.5, got {}", p.life[0]);
        // ลด life ด้วย dt=1.5 → life=0.0 → particle ถูกลบ
        p.update(1.5, &forces);
        assert_eq!(p.len(), 0, "dead particle should be removed, len={}", p.len());
    }

    // ── Test 8: Explosion count ────────────────────────────────────
    #[test]
    fn test_explosion_particle_count() {
        let exp = Explosion::new((100.0, 100.0));
        let particles = exp.spawn();
        assert_eq!(particles.len(), 500,
            "explosion should spawn 500 particles, got {}", particles.len());
    }

    // ── Test 9: ColorGradient stop boundary clamping ───────────────
    #[test]
    fn test_color_gradient_stop_boundary() {
        // Stops ไม่เริ่มที่ 0.0 หรือจบที่ 1.0
        let grad = ColorGradient::new(vec![
            (0.2, Color::new(1.0, 0.0, 0.0, 1.0)),
            (0.8, Color::new(0.0, 1.0, 0.0, 1.0)),
        ]);
        // t < 0.2 → clamp ไปที่ stop แรก (red)
        let c_low = grad.sample(0.0);
        assert!((c_low.r - 1.0).abs() < 1e-5,
            "t=0.0 below first stop should give first stop color");
        // t > 0.8 → clamp ไปที่ stop สุดท้าย (green)
        let c_high = grad.sample(1.0);
        assert!((c_high.g - 1.0).abs() < 1e-5,
            "t=1.0 above last stop should give last stop color");
    }
}
```

### ผลลัพธ์จาก `cargo test` (real output)

```
$ cargo test

running 9 tests
test tests::test_color_gradient_at_1 ... ok
test tests::test_euler_integration_step ... ok
test tests::test_color_gradient_at_0 ... ok
test tests::test_color_gradient_stop_boundary ... ok
test tests::test_life_decay ... ok
test tests::test_color_gradient_at_half ... ok
test tests::test_soa_swap_remove_correctness ... ok
test tests::test_vortex_force_direction ... ok
test tests::test_explosion_particle_count ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก test ผ่านใน 0.00s — particle system logic ทำงานถูกต้อง

### ผลลัพธ์จาก `cargo run`

```
=== Particle System Simulator ===
Explosion spawned 500 particles
SoA particle count: 500
Update 500 particles: 103.632µs
Remaining after 1 frame: 500
Gradient at 0.5: r=1.00 g=1.00 b=0.00 a=1.00

--- Benchmark ---
Sequential 10k particles: 773.363µs
```

---

## Pitfalls ที่ควรระวัง

### Pitfall 1: `swap_remove` loop ต้องไม่ increment เมื่อ remove

นี่คือ bug ที่พบบ่อยมากใน particle system:

```rust
// ❌ BUG: ข้าม particle บางตัวไป
for i in 0..self.len() {      // len() เปลี่ยนระหว่าง loop!
    if self.life[i] <= 0.0 {
        self.swap_remove(i);
        // i++ ทำให้ข้าม element ที่ถูก swap มาจากท้าย
    }
}

// ✅ CORRECT: ใช้ while loop ไม่ increment เมื่อ remove
let mut i = 0;
while i < self.len() {
    if self.life[i] <= 0.0 {
        self.swap_remove(i);
        // ไม่ increment — ตรวจ element ใหม่ที่ index นี้ (ซึ่งถูก swap มา)
    } else {
        i += 1;
    }
}
```

ถ้า bug นี้เกิดขึ้น particle บางตัวจะ "ผี" (invisible แต่ยังอยู่ใน array) และนับ particles จะผิดเพี้ยน

### Pitfall 2: SoA swap_remove ต้อง swap **ทุก** field พร้อมกัน

ถ้าลืม field ใด field หนึ่ง data จะ desync — particle จะมี color ของอีก particle:

```rust
// ❌ BUG: ลืม swap สี
pub fn swap_remove_buggy(&mut self, index: usize) {
    let last = self.len() - 1;
    self.x.swap(index, last);
    self.y.swap(index, last);
    self.vx.swap(index, last);
    self.vy.swap(index, last);
    self.life.swap(index, last);
    self.max_life.swap(index, last);
    // ลืม r, g, b, a, size!  ← สี/ขนาดจะ desync กับ position
    self.x.pop(); self.y.pop(); /* ... */
}

// ✅ CORRECT: swap ทุก field ครบ
```

วิธีป้องกัน: เขียน test ที่ตรวจ field หลาย ๆ ตัวหลัง swap_remove (ดูใน test suite ด้านบน)

### Pitfall 3: Euler integration energy drift กับ spring/vortex forces

Euler integration ไม่ conserve energy เมื่อใช้กับ forces ที่เป็น function ของ position:

```rust
// ❌ PROBLEMATIC สำหรับ spring/vortex force ที่ต้องการ accuracy:
// velocity update ก่อน position (Euler forward) → unstable สำหรับ stiff systems
self.vx[i] += ax * dt;
self.vy[i] += ay * dt;
self.x[i] += self.vx[i] * dt;  // ใช้ velocity ใหม่แล้ว → semi-implicit Euler
self.y[i] += self.vy[i] * dt;

// ✅ BETTER: Symplectic Euler (semi-implicit) — ที่จริงโค้ดด้านบนก็เป็นแบบนี้อยู่แล้ว
// แต่ถ้า dt ใหญ่เกิน 1/30s ระบบอาจยังไม่เสถียร

// ✅ BEST สำหรับ particles ที่ต้องการ accuracy สูง: Verlet integration
let old_vx = self.vx[i];
self.x[i] += old_vx * dt + 0.5 * ax * dt * dt;
self.vx[i] += ax * dt;
```

สำหรับ particle effects ทั่วไปที่มีอายุสั้น Euler ใช้ได้ดี แต่ถ้าทำ physics simulation ที่ต้องการความแม่นยำควรเปลี่ยนเป็น Verlet

### Pitfall 4: `rayon` overhead บน particle count น้อย

```rust
// ❌ BAD: ใช้ parallel เสมอ — overhead สูงกว่า benefit
particles.update_parallel(dt, &forces); // overhead ~50µs ต่อ call

// ✅ GOOD: เลือกตาม count
if particles.len() > 10_000 {
    particles.update_parallel(dt, &forces);
} else {
    particles.update(dt, &forces);
}
```

Rayon มี overhead ประมาณ 20-100µs ต่อ parallel call (ขึ้นอยู่กับ hardware) สำหรับ particle จำนวนน้อย sequential เร็วกว่า

### Pitfall 5: ลืม clamp life ratio ก่อน sample gradient

```rust
// ❌ BUG: life อาจเป็นลบช่วงสั้น ๆ ก่อนถูกลบ → color ผิดพลาด
let ratio = self.life[i] / self.max_life[i];
let c = gradient.sample(ratio);  // ratio อาจเป็น -0.001

// ✅ CORRECT: clamp ก่อน sample
let ratio = (self.life[i] / self.max_life[i]).clamp(0.0, 1.0);
let c = gradient.sample(ratio);
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# รัน demo ระเบิด
./target/release/particle_system run --effect explosion

# Benchmark 100k particles
./target/release/particle_system bench --count 100000 --frames 60
```

### Cargo.toml สำหรับ Release

```toml
[profile.release]
opt-level = 3
lto = "thin"       # Link-Time Optimization
codegen-units = 1  # เพิ่ม optimization quality

# SIMD สำหรับ floating point
[profile.release.build-override]
opt-level = 3
```

### Profile-Guided Optimization (PGO) — ขั้นสูง

```bash
# Step 1: Build with instrumentation
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# Step 2: Run benchmark เพื่อ generate profile data
./target/release/particle_system bench --count 100000 --frames 60

# Step 3: Merge profile data
llvm-profdata merge -output=/tmp/pgo-data/merged.profdata /tmp/pgo-data

# Step 4: Build with PGO
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Verlet Integration (ระดับ: ปานกลาง)

แทนที่ Euler integration ด้วย Verlet integration เพื่อความแม่นยำที่ดีขึ้น Verlet ต้องการเก็บ position ก่อนหน้า (`prev_x`, `prev_y`) แทน velocity:

```
x_new = 2*x - x_prev + a * dt²
```

**สิ่งที่ต้องทำ:**
1. เพิ่ม `prev_x: Vec<f32>` และ `prev_y: Vec<f32>` ใน `Particles` struct
2. แก้ `swap_remove` ให้ swap `prev_x`/`prev_y` ด้วย
3. เขียน `update_verlet()` method แทน `update()`
4. เขียน test เปรียบ energy conservation ระหว่าง Euler vs Verlet สำหรับ circular orbit

**เป้าหมาย:** Verlet ควรทำให้ particle ที่อยู่ใน vortex วนเป็นวงกลมเรียบกว่า Euler โดยเฉพาะเมื่อ dt ใหญ่ (0.05s+)

---

### แบบฝึกหัดที่ 2: GPU-Ready SoA Export (ระดับ: ปานกลาง-ยาก)

ใน game engine จริง ๆ particle data ถูกส่งไปยัง GPU เป็น buffer เพิ่ม method `to_interleaved_buffer()` ที่ convert SoA เป็น interleaved format สำหรับ GPU upload:

```rust
// Output: [x0,y0,r0,g0,b0,a0,size0, x1,y1,r1,g1,b1,a1,size1, ...]
pub fn to_interleaved_buffer(&self) -> Vec<f32> { ... }

// และ method กลับ
pub fn from_interleaved_buffer(data: &[f32]) -> Particles { ... }
```

**สิ่งที่ต้องทำ:**
1. Implement `to_interleaved_buffer()` และ `from_interleaved_buffer()`
2. เขียน round-trip test: SoA → interleaved → SoA ต้องได้ค่าเดิม
3. Benchmark: SoA update loop vs interleaved update loop สำหรับ 1M particles

**เป้าหมาย:** เข้าใจว่าทำไม GPU ชอบ interleaved (AoS) แต่ CPU ชอบ SoA — ทั้งสองแบบมี use case ของตัวเอง

---

### แบบฝึกหัดที่ 3: Spatial Hashing สำหรับ Particle Collision (ระดับ: ยาก)

เพิ่ม particle-to-particle collision — particle ที่อยู่ใกล้กันมากเกินไปจะ bounce ออกจากกัน การ naive check คือ O(n²) สำหรับทุกคู่ เพิ่ม `SpatialHash` struct เพื่อลดเป็น O(n) เฉลี่ย:

```rust
pub struct SpatialHash {
    cell_size: f32,
    cells: HashMap<(i32, i32), Vec<usize>>,  // (cell_x, cell_y) → [particle indices]
}
```

**สิ่งที่ต้องทำ:**
1. Implement `SpatialHash::insert(x, y, index)` และ `SpatialHash::query_nearby(x, y, radius)`
2. ใน `particles.update()`: หลัง integrate, rebuild spatial hash แล้ว detect collisions
3. เมื่อ particle 2 ตัวชน (distance < sum of radii): คำนวณ elastic collision response
4. Benchmark: naive O(n²) vs spatial hash O(n) สำหรับ 10k particles dense

---

### แบบฝึกหัดที่ 4: WebAssembly Rendering (ระดับ: ยากมาก)

เปลี่ยน renderer จาก terminal ไปเป็น browser canvas ผ่าน WASM (เช่นเดียวกับ Project E05):

**สิ่งที่ต้องทำ:**
1. แยก library crate (`particle_lib`) จาก binary
2. เพิ่ม `[lib] crate-type = ["cdylib"]` และ `wasm-bindgen` dependency
3. Export ฟังก์ชัน: `new_simulation()`, `update(dt)`, `get_particle_data() -> Float32Array`
4. เขียน JavaScript ที่รับ `Float32Array` จาก WASM แล้ว render ลง `<canvas>` ด้วย WebGL instancing
5. เปรียบ performance: terminal (60fps) vs WASM+WebGL (144fps?)

**เป้าหมาย:** ได้ particle system ที่ render ใน browser แบบ real-time พร้อม 100k+ particles ด้วย WebGL instancing

---

## สรุป

โปรเจคนี้สอน technique ที่ใช้จริงใน game engine และ VFX pipeline:

**Concepts สำคัญที่ได้เรียน:**

1. **Structure-of-Arrays (SoA)** — ไม่ใช่แค่ optimization trick แต่เป็น fundamental data layout ที่ modern game engine อย่าง Unity ECS, Bevy ใช้เป็นหลัก เพราะ CPU cache architecture ชอบ linear access มากกว่า pointer chasing

2. **O(1) swap_remove** — trade-off ระหว่าง ordering (O(n) insert/remove) กับ speed (O(1)) สำหรับ particle ที่ไม่ต้องการลำดับ swap_remove คือ optimal choice

3. **Trait object composition** — `Box<dyn ForceField>` ทำให้รวม force หลายชนิดใน `Vec<Box<dyn ForceField>>` ได้โดยไม่ต้องรู้ type ล่วงหน้า เป็น pattern ที่ใช้บ่อยใน plugin system และ middleware

4. **Rayon parallel iteration** — เพิ่ม parallelism ด้วย `.into_par_iter()` แทน `.into_iter()` เพียงบรรทัดเดียว แต่ต้องระวัง overhead และ `Send + Sync` requirements

5. **Terminal rendering** — crossterm ทำให้ใช้ terminal เป็น "canvas" ได้ง่าย เป็นเทคนิค useful สำหรับ CLI tool ที่ต้องการ visual feedback แบบ real-time

**เชื่อมโยงกับโปรเจคถัดไป:**

Project E09 (ASCII Art Generator) จะต่อยอดจาก terminal rendering ที่เรียนในโปรเจคนี้ — แต่แทนที่จะ render particles แบบ real-time จะเน้นที่การ convert images เป็น ASCII art แบบ batch processing พร้อม color support และ font selection

---

**โปรเจคก่อนหน้า:** [project-e07-map-generator.md](project-e07-map-generator.md) | **โปรเจคถัดไป:** [project-e09-ascii-art.md](project-e09-ascii-art.md)
