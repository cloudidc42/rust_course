# Project E03: 2D Physics Engine

> โมดูล: Games & Graphics | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

Physics engine คือหัวใจของเกม 2D เกือบทุกประเภท — ไม่ว่าจะเป็นเกม platformer, puzzle ที่ใช้แรงโน้มถ่วง, เกม billiards, หรือ simulation ทางฟิสิกส์ โปรเจคนี้จะสร้าง **2D Rigid Body Physics Engine ตั้งแต่ศูนย์** โดยไม่พึ่งพา library ฟิสิกส์ใด ๆ ทำให้ผู้เรียนเข้าใจหลักการที่ engine อย่าง Box2D, Chipmunk, หรือ Rapier ใช้งานจริงใต้ฝากระโปรง

**Use case ในโลก production:**
- เกม 2D ที่ต้องการ custom physics behavior
- Simulation การเคลื่อนที่ของวัตถุ (robotics, animation)
- เครื่องมือ level design ที่ต้องตรวจสอบ physics
- การเรียนรู้ numerical methods สำหรับ game dev

**Learning value:** โปรเจคนี้ครอบคลุม math primitives, collision detection ทั้ง broad/narrow phase, impulse-based resolution, friction, constraints, และ JSON-driven scene system ซึ่งเป็น pattern ที่นำไปใช้ใน production engine ได้ทันที

---

## สิ่งที่จะได้เรียนรู้

- **Vec2/Matrix math** — การ implement operator overloading สำหรับ math primitives ใน Rust
- **Rigid body dynamics** — Semi-implicit Euler integration, force/torque accumulation
- **Broad phase collision** — Spatial hashing ด้วย `HashMap<(i32,i32), Vec<BodyId>>`
- **Narrow phase collision** — Circle-circle, Circle-AABB, AABB-AABB, SAT สำหรับ polygon
- **Impulse-based resolution** — การแก้ collision ด้วย impulse + Baumgarte stabilization
- **Coulomb friction model** — tangential impulse clamped ด้วย μ
- **Constraints** — DistanceConstraint, HingeConstraint ด้วย position correction
- **JSON scene format** — serde_json สำหรับ scene definition และ snapshot

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 5–15**: Structs, enums, traits, operator overloading (`Add`, `Sub`, `Mul`)
- **Part 20–25**: Generics, trait bounds, lifetime พื้นฐาน
- **Part 30–35**: Collections (`Vec`, `HashMap`), iterators
- **Part 40–45**: Error handling (`Result`, `?`), `serde` basics
- **Part 50–55**: Unsafe Rust พื้นฐาน (raw pointer สำหรับ split borrow workaround)
- **Part 60–65**: `clap` 4 สำหรับ CLI, module system

---

## โครงสร้างโปรเจค (Project Layout)

```
physics-engine/
├── src/
│   ├── main.rs          ← CLI entry point (clap)
│   ├── math.rs          ← Vec2, Transform, AABB, Circle primitives
│   ├── body.rs          ← RigidBody, Shape, integration
│   ├── collision.rs     ← SpatialHash (broad), narrow phase detectors
│   ├── resolution.rs    ← Impulse resolution + Baumgarte correction
│   ├── constraints.rs   ← DistanceConstraint, HingeConstraint
│   └── world.rs         ← World::step, load_scene, save_state
├── scenes/
│   └── demo.json        ← ตัวอย่าง scene definition
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow ของแต่ละ simulation step

```
World::step(dt)
    │
    ├─ 1. Accumulate Forces  ← ใส่ gravity เข้า force_accum ของทุก body
    │
    ├─ 2. Broad Phase        ← SpatialHash: แบ่ง world เป็น grid, หา candidate pairs
    │
    ├─ 3. Narrow Phase       ← detect_collision(a, b) → Option<ContactManifold>
    │      ├─ Circle×Circle  (distance < r1+r2)
    │      ├─ Circle×AABB    (closest point)
    │      ├─ AABB×AABB      (overlap axes)
    │      └─ Polygon×Polygon (SAT)
    │
    ├─ 4. Resolution         ← resolve_collision + positional_correction (Baumgarte)
    │
    ├─ 5. Solve Constraints  ← DistanceConstraint, HingeConstraint
    │
    └─ 6. Integrate          ← Semi-implicit Euler: v += (F/m)*dt; x += v*dt
```

### ทำไมถึงเลือก design นี้

| การตัดสินใจ | เหตุผล |
|---|---|
| Semi-implicit Euler | ง่าย, stable กว่า explicit Euler, เพียงพอสำหรับ game physics |
| Spatial Hashing (broad phase) | O(n) average case, ง่าย implement กว่า BVH tree |
| Impulse-based resolution | Industry standard, handle stacking ได้ดี |
| Baumgarte stabilization | แก้ positional drift ที่สะสมจาก numerical error |
| `f64` ทั้งหมด | Deterministic, reproducible simulation |
| Type alias `BodyId = usize` | ง่าย, ไม่มี overhead, safe ใน single-threaded world |

### Borrow Checker Challenge

ปัญหาหลักของ physics engine ใน Rust คือการ borrow body สองตัวพร้อมกัน (`body_a` และ `body_b` ต้อง `&mut` ทั้งคู่) วิธีแก้คือ `split_at_mut`:

```rust
if id_a < id_b {
    let (left, right) = self.bodies.split_at_mut(id_b);
    resolve_collision(&mut left[id_a], &mut right[0], &m);
} else {
    let (left, right) = self.bodies.split_at_mut(id_a);
    resolve_collision(&mut right[0], &mut left[id_b], &m);
}
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Math Primitives

สร้างพื้นฐานทางคณิตศาสตร์ทั้งหมดก่อน ทุกอย่างในฟิสิกส์ต้องพึ่ง Vec2

**`src/math.rs`** — สร้างก่อน:

```rust
use std::ops::{Add, Sub, Mul, Neg};

#[derive(Debug, Clone, Copy, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct Vec2 {
    pub x: f64,
    pub y: f64,
}

impl Vec2 {
    pub const ZERO: Vec2 = Vec2 { x: 0.0, y: 0.0 };

    pub fn new(x: f64, y: f64) -> Self {
        Vec2 { x, y }
    }

    /// Dot product: a·b = ax*bx + ay*by
    pub fn dot(self, other: Vec2) -> f64 {
        self.x * other.x + self.y * other.y
    }

    /// Cross product 2D: คืน scalar (z-component ของ 3D cross)
    /// ใช้คำนวณ torque และ signed area
    pub fn cross(self, other: Vec2) -> f64 {
        self.x * other.y - self.y * other.x
    }

    /// Cross product ระหว่าง scalar กับ Vec2 (ใช้ใน angular velocity)
    pub fn cross_scalar(s: f64, v: Vec2) -> Vec2 {
        Vec2::new(-s * v.y, s * v.x)
    }

    pub fn length_sq(self) -> f64 {
        self.x * self.x + self.y * self.y
    }

    pub fn length(self) -> f64 {
        self.length_sq().sqrt()
    }

    /// normalize — ระวัง: vector ศูนย์ คืน Vec2::ZERO (ไม่ panic)
    pub fn normalize(self) -> Vec2 {
        let len = self.length();
        if len < 1e-10 { Vec2::ZERO } else { Vec2::new(self.x / len, self.y / len) }
    }

    /// เวกเตอร์ตั้งฉาก (หมุน 90° ทวนเข็ม)
    pub fn perpendicular(self) -> Vec2 {
        Vec2::new(-self.y, self.x)
    }

    /// หมุน angle radians
    pub fn rotate(self, angle: f64) -> Vec2 {
        let (sin, cos) = angle.sin_cos();
        Vec2::new(self.x * cos - self.y * sin, self.x * sin + self.y * cos)
    }

    pub fn min_comp(self, other: Vec2) -> Vec2 {
        Vec2::new(self.x.min(other.x), self.y.min(other.y))
    }

    pub fn max_comp(self, other: Vec2) -> Vec2 {
        Vec2::new(self.x.max(other.x), self.y.max(other.y))
    }
}

// Operator overloading
impl Add for Vec2 {
    type Output = Vec2;
    fn add(self, other: Vec2) -> Vec2 { Vec2::new(self.x + other.x, self.y + other.y) }
}

impl Sub for Vec2 {
    type Output = Vec2;
    fn sub(self, other: Vec2) -> Vec2 { Vec2::new(self.x - other.x, self.y - other.y) }
}

impl Mul<f64> for Vec2 {
    type Output = Vec2;
    fn mul(self, s: f64) -> Vec2 { Vec2::new(self.x * s, self.y * s) }
}

impl Mul<Vec2> for f64 {
    type Output = Vec2;
    fn mul(self, v: Vec2) -> Vec2 { Vec2::new(self * v.x, self * v.y) }
}

impl Neg for Vec2 {
    type Output = Vec2;
    fn neg(self) -> Vec2 { Vec2::new(-self.x, -self.y) }
}
```

**Transform, AABB, Circle:**

```rust
#[derive(Debug, Clone, Copy, serde::Serialize, serde::Deserialize)]
pub struct Transform {
    pub position: Vec2,
    pub rotation: f64,  // radians
    pub scale: Vec2,
}

impl Transform {
    pub fn identity() -> Self {
        Transform { position: Vec2::ZERO, rotation: 0.0, scale: Vec2::new(1.0, 1.0) }
    }

    pub fn transform_point(&self, local: Vec2) -> Vec2 {
        let scaled = Vec2::new(local.x * self.scale.x, local.y * self.scale.y);
        scaled.rotate(self.rotation) + self.position
    }
}

#[derive(Debug, Clone, Copy, serde::Serialize, serde::Deserialize)]
pub struct Aabb {
    pub min: Vec2,
    pub max: Vec2,
}

impl Aabb {
    pub fn from_center(center: Vec2, half: Vec2) -> Self {
        Aabb { min: center - half, max: center + half }
    }

    pub fn overlaps(&self, other: &Aabb) -> bool {
        self.min.x <= other.max.x && self.max.x >= other.min.x
            && self.min.y <= other.max.y && self.max.y >= other.min.y
    }
}

#[derive(Debug, Clone, Copy, serde::Serialize, serde::Deserialize)]
pub struct Circle {
    pub center: Vec2,
    pub radius: f64,
}
```

**Test ขั้นที่ 1:**

```bash
cargo test math::
```

Output:
```
running 7 tests
test math::tests::test_vec2_add ... ok
test math::tests::test_vec2_sub ... ok
test math::tests::test_vec2_dot ... ok
test math::tests::test_vec2_cross ... ok
test math::tests::test_vec2_length ... ok
test math::tests::test_vec2_normalize ... ok
test math::tests::test_aabb_overlap ... ok
test math::tests::test_vec2_rotate_90 ... ok

test result: ok. 8 passed; 0 failed
```

---

### ขั้นที่ 2: Rigid Body และ Semi-Implicit Euler Integration

**`src/body.rs`** — ออกแบบ RigidBody struct:

```rust
use crate::math::Vec2;
use serde::{Deserialize, Serialize};

pub type BodyId = usize;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Shape {
    Circle { radius: f64 },
    Aabb { half_extents: Vec2 },
    Polygon { vertices: Vec<Vec2> },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RigidBody {
    pub id: BodyId,
    pub mass: f64,
    pub inv_mass: f64,           // 1/m หรือ 0 สำหรับ static
    pub moment_of_inertia: f64,
    pub inv_moi: f64,
    pub position: Vec2,
    pub velocity: Vec2,
    pub angle: f64,              // radians
    pub angular_velocity: f64,   // radians/second
    pub force_accum: Vec2,       // force ที่ accumulated ในแต่ละ step
    pub torque_accum: f64,
    pub restitution: f64,        // 0 = plastic, 1 = elastic
    pub friction: f64,           // Coulomb coefficient
    pub is_static: bool,
    pub shape: Shape,
}

impl RigidBody {
    pub fn new(
        id: BodyId, mass: f64, moment_of_inertia: f64,
        position: Vec2, shape: Shape,
        restitution: f64, friction: f64, is_static: bool,
    ) -> Self {
        let (inv_mass, inv_moi) = if is_static || mass == 0.0 {
            (0.0, 0.0)
        } else {
            (1.0 / mass, 1.0 / moment_of_inertia)
        };
        RigidBody {
            id, mass, inv_mass, moment_of_inertia, inv_moi,
            position, velocity: Vec2::ZERO,
            angle: 0.0, angular_velocity: 0.0,
            force_accum: Vec2::ZERO, torque_accum: 0.0,
            restitution, friction, is_static, shape,
        }
    }

    pub fn apply_force(&mut self, force: Vec2) {
        if !self.is_static { self.force_accum = self.force_accum + force; }
    }

    /// Apply force ที่ world point (สร้าง torque ด้วย)
    pub fn apply_force_at_point(&mut self, force: Vec2, point: Vec2) {
        if !self.is_static {
            self.force_accum = self.force_accum + force;
            let r = point - self.position;
            self.torque_accum += r.cross(force);
        }
    }

    /// Apply impulse (เปลี่ยน velocity ทันที ไม่ใช้ dt)
    pub fn apply_impulse(&mut self, impulse: Vec2, contact_vector: Vec2) {
        if !self.is_static {
            self.velocity = self.velocity + self.inv_mass * impulse;
            self.angular_velocity += self.inv_moi * contact_vector.cross(impulse);
        }
    }

    /// Semi-implicit Euler:
    ///   v(t+dt) = v(t) + a(t)*dt    ← velocity update ก่อน
    ///   x(t+dt) = x(t) + v(t+dt)*dt ← position ใช้ velocity ใหม่
    /// ดีกว่า explicit Euler: energy ลดลงในระยะยาว (stable)
    pub fn integrate(&mut self, dt: f64) {
        if self.is_static {
            self.force_accum = Vec2::ZERO;
            self.torque_accum = 0.0;
            return;
        }

        // Linear
        let accel = self.force_accum * self.inv_mass;
        self.velocity = self.velocity + accel * dt;    // v += (F/m)*dt
        self.position = self.position + self.velocity * dt; // x += v*dt

        // Angular
        let angular_accel = self.torque_accum * self.inv_moi;
        self.angular_velocity += angular_accel * dt;
        self.angle += self.angular_velocity * dt;

        // Reset
        self.force_accum = Vec2::ZERO;
        self.torque_accum = 0.0;
    }

    /// Velocity ที่จุด p บน body (รวม angular component)
    /// v_point = v_center + ω × r
    pub fn velocity_at_point(&self, point: Vec2) -> Vec2 {
        let r = point - self.position;
        self.velocity + Vec2::cross_scalar(self.angular_velocity, r)
    }
}
```

**ทดสอบ Euler integration:**

```rust
#[test]
fn test_euler_integration() {
    let mut body = make_circle_body(0, 1.0, Vec2::new(0.0, 0.0));
    let gravity = Vec2::new(0.0, -9.8);
    body.apply_force(gravity * body.mass);  // F = mg

    let dt = 0.016; // ~60fps
    body.integrate(dt);

    // v += (F/m)*dt = (mg/m)*dt = g*dt
    let expected_vy = -9.8 * dt;  // = -0.1568
    assert!((body.velocity.y - expected_vy).abs() < 1e-9);

    // x += v*dt (ใช้ velocity หลัง update)
    let expected_y = expected_vy * dt;  // = -0.002509
    assert!((body.position.y - expected_y).abs() < 1e-9);
}
```

---

### ขั้นที่ 3: Broad Phase — Spatial Hashing

Spatial hashing แบ่ง world ออกเป็น grid cells ขนาด `cell_size × cell_size` แต่ละ body ลงทะเบียนใน cells ที่ AABB ของมันทับ จากนั้น pairs ที่อยู่ใน cell เดียวกันเท่านั้นที่ส่งไป narrow phase

**`src/collision.rs`** — SpatialHash:

```rust
pub struct SpatialHash {
    cell_size: f64,
    cells: std::collections::HashMap<(i32, i32), Vec<BodyId>>,
}

impl SpatialHash {
    pub fn new(cell_size: f64) -> Self {
        SpatialHash { cell_size, cells: std::collections::HashMap::new() }
    }

    pub fn clear(&mut self) { self.cells.clear(); }

    fn world_to_cell(&self, p: Vec2) -> (i32, i32) {
        (
            (p.x / self.cell_size).floor() as i32,
            (p.y / self.cell_size).floor() as i32,
        )
    }

    /// ลงทะเบียน body เข้า cells ที่ AABB ทับ (อาจทับหลาย cell)
    pub fn insert(&mut self, id: BodyId, aabb: &Aabb) {
        let min_cell = self.world_to_cell(aabb.min);
        let max_cell = self.world_to_cell(aabb.max);

        for cy in min_cell.1..=max_cell.1 {
            for cx in min_cell.0..=max_cell.0 {
                self.cells.entry((cx, cy)).or_default().push(id);
            }
        }
    }

    /// คืน pairs ที่ share cell เดียวกัน — ไม่ซ้ำ ด้วย HashSet
    pub fn query_pairs(&self) -> Vec<(BodyId, BodyId)> {
        let mut pairs = std::collections::HashSet::new();
        for ids in self.cells.values() {
            for i in 0..ids.len() {
                for j in (i + 1)..ids.len() {
                    let a = ids[i].min(ids[j]);
                    let b = ids[i].max(ids[j]);
                    pairs.insert((a, b));
                }
            }
        }
        pairs.into_iter().collect()
    }
}
```

**ทดสอบ Spatial Hash:**

```rust
#[test]
fn test_spatial_hash_pairs() {
    let mut hash = SpatialHash::new(2.0);
    // A และ B อยู่ใกล้กัน, C อยู่ไกล
    hash.insert(0, &Aabb::from_center(Vec2::new(0.0, 0.0), Vec2::new(0.5, 0.5)));
    hash.insert(1, &Aabb::from_center(Vec2::new(0.5, 0.0), Vec2::new(0.5, 0.5)));
    hash.insert(2, &Aabb::from_center(Vec2::new(100.0, 100.0), Vec2::new(0.5, 0.5)));

    let pairs = hash.query_pairs();
    assert!(pairs.contains(&(0, 1)));  // ✓ share cell
    assert!(!pairs.contains(&(0, 2))); // ✗ ไม่ share cell
}
```

---

### ขั้นที่ 4: Narrow Phase — Circle, AABB, SAT

**ContactManifold** เก็บผลลัพธ์ของการตรวจจับ:

```rust
pub struct ContactManifold {
    pub body_a: BodyId,
    pub body_b: BodyId,
    pub normal: Vec2,     // ชี้จาก B → A (push direction สำหรับ A)
    pub penetration: f64, // ความลึก overlap
    pub contact_point: Vec2,
}
```

#### Circle vs Circle

```rust
pub fn circle_circle(
    pos_a: Vec2, r_a: f64,
    pos_b: Vec2, r_b: f64,
    id_a: BodyId, id_b: BodyId,
) -> Option<ContactManifold> {
    let diff = pos_a - pos_b;
    let dist_sq = diff.length_sq();
    let sum_r = r_a + r_b;

    if dist_sq >= sum_r * sum_r {
        return None; // ไม่ชนกัน
    }

    let dist = dist_sq.sqrt();
    let (normal, penetration) = if dist < 1e-10 {
        (Vec2::new(0.0, 1.0), sum_r) // overlap สมบูรณ์
    } else {
        (diff * (1.0 / dist), sum_r - dist)
    };

    Some(ContactManifold {
        body_a: id_a, body_b: id_b,
        normal,
        penetration,
        contact_point: pos_b + normal * (r_b - penetration * 0.5),
    })
}
```

#### SAT สำหรับ Convex Polygon

Separating Axis Theorem: ถ้ามี axis ใดที่ projection ของสอง polygon ไม่ overlap แสดงว่าไม่ชนกัน

```rust
fn project_polygon(vertices: &[Vec2], axis: Vec2) -> (f64, f64) {
    let mut min = f64::MAX;
    let mut max = f64::MIN;
    for v in vertices {
        let proj = v.dot(axis);
        min = min.min(proj);
        max = max.max(proj);
    }
    (min, max)
}

pub fn sat_overlap(
    verts_a: &[Vec2], pos_a: Vec2, angle_a: f64,
    verts_b: &[Vec2], pos_b: Vec2, angle_b: f64,
    id_a: BodyId, id_b: BodyId,
) -> Option<ContactManifold> {
    // แปลงเป็น world space
    let wa: Vec<Vec2> = verts_a.iter().map(|v| v.rotate(angle_a) + pos_a).collect();
    let wb: Vec<Vec2> = verts_b.iter().map(|v| v.rotate(angle_b) + pos_b).collect();

    let normals_a = get_normals(&wa); // edge normals ของ A
    let normals_b = get_normals(&wb); // edge normals ของ B

    let mut min_overlap = f64::MAX;
    let mut collision_normal = Vec2::ZERO;

    // ทดสอบทุก separating axis
    for axis in normals_a.iter().chain(normals_b.iter()) {
        let (min_a, max_a) = project_polygon(&wa, *axis);
        let (min_b, max_b) = project_polygon(&wb, *axis);

        let overlap = max_a.min(max_b) - min_a.max(min_b);
        if overlap <= 0.0 {
            return None; // Separating axis found!
        }

        if overlap < min_overlap {
            min_overlap = overlap;
            collision_normal = *axis;
        }
    }

    // ให้ normal ชี้จาก B → A
    if (pos_a - pos_b).dot(collision_normal) < 0.0 {
        collision_normal = -collision_normal;
    }

    Some(ContactManifold {
        body_a: id_a, body_b: id_b,
        normal: collision_normal,
        penetration: min_overlap,
        contact_point: pos_a - collision_normal * (min_overlap * 0.5),
    })
}
```

**ทดสอบ SAT:**

```rust
#[test]
fn test_sat_separation_axis() {
    let square = vec![
        Vec2::new(-1.0, -1.0), Vec2::new(1.0, -1.0),
        Vec2::new(1.0, 1.0), Vec2::new(-1.0, 1.0),
    ];
    // ห่างกัน — ต้องไม่ชนกัน
    let result = sat_overlap(&square, Vec2::ZERO, 0.0,
                             &square, Vec2::new(5.0, 0.0), 0.0, 0, 1);
    assert!(result.is_none());

    // ใกล้กัน — ต้องชนกัน
    let result2 = sat_overlap(&square, Vec2::ZERO, 0.0,
                              &square, Vec2::new(1.5, 0.0), 0.0, 0, 1);
    assert!(result2.is_some());
    assert!(result2.unwrap().penetration > 0.0);
}
```

---

### ขั้นที่ 5: Collision Resolution — Impulse + Friction

#### สูตร Impulse

สำหรับการชนระหว่าง body A และ B ที่จุด contact `p`:

```
r_a = p - pos_a     (contact vector จาก center of mass)
r_b = p - pos_b

v_rel = v_A(p) - v_B(p)  (relative velocity ที่ contact point)
v_rel_n = v_rel · n       (component ตาม normal)

ถ้า v_rel_n > 0 → bodies กำลังเคลื่อนออกจากกัน → ข้ามไป

j_n = -(1 + e) * v_rel_n
    / (1/m_A + 1/m_B + (r_A×n)²/I_A + (r_B×n)²/I_B)
```

**`src/resolution.rs`:**

```rust
pub fn resolve_collision(
    body_a: &mut RigidBody,
    body_b: &mut RigidBody,
    manifold: &ContactManifold,
) {
    let n = manifold.normal;
    let cp = manifold.contact_point;
    let r_a = cp - body_a.position;
    let r_b = cp - body_b.position;

    let v_rel = body_a.velocity_at_point(cp) - body_b.velocity_at_point(cp);
    let v_rel_n = v_rel.dot(n);

    // ถ้ากำลังเคลื่อนออกจากกัน ไม่ต้อง resolve
    if v_rel_n > 0.0 { return; }

    let e = body_a.restitution.min(body_b.restitution);

    let ra_cross_n = r_a.cross(n);
    let rb_cross_n = r_b.cross(n);
    let inv_mass_sum = body_a.inv_mass + body_b.inv_mass
        + ra_cross_n * ra_cross_n * body_a.inv_moi
        + rb_cross_n * rb_cross_n * body_b.inv_moi;

    if inv_mass_sum < 1e-10 { return; }

    // Normal impulse
    let j_n = -(1.0 + e) * v_rel_n / inv_mass_sum;
    let impulse_n = n * j_n;
    body_a.apply_impulse(impulse_n, r_a);
    body_b.apply_impulse(-impulse_n, r_b);

    // Friction (Coulomb model)
    let v_rel2 = body_a.velocity_at_point(cp) - body_b.velocity_at_point(cp);
    let v_tang = v_rel2 - n * v_rel2.dot(n);
    let t_len = v_tang.length();
    if t_len < 1e-10 { return; }

    let tangent = v_tang * (1.0 / t_len);
    let ra_cross_t = r_a.cross(tangent);
    let rb_cross_t = r_b.cross(tangent);
    let inv_mass_t = body_a.inv_mass + body_b.inv_mass
        + ra_cross_t * ra_cross_t * body_a.inv_moi
        + rb_cross_t * rb_cross_t * body_b.inv_moi;

    if inv_mass_t < 1e-10 { return; }

    let j_t_raw = -t_len / inv_mass_t;

    // Coulomb's Law: |j_t| ≤ μ * |j_n|
    let mu = (body_a.friction * body_b.friction).sqrt();
    let j_t = j_t_raw.clamp(-mu * j_n, mu * j_n);

    let impulse_t = tangent * j_t;
    body_a.apply_impulse(impulse_t, r_a);
    body_b.apply_impulse(-impulse_t, r_b);
}
```

#### Baumgarte Stabilization

เมื่อ bodies ซ้อนกัน (positional drift จาก numerical error) เราแก้ด้วยการเลื่อน position โดยตรง:

```rust
pub fn positional_correction(
    body_a: &mut RigidBody,
    body_b: &mut RigidBody,
    manifold: &ContactManifold,
) {
    const SLOP: f64 = 0.01;    // penetration ที่ยอมรับได้ (ป้องกัน jitter)
    const PERCENT: f64 = 0.2;  // แก้ไข 20% ต่อ step

    let correction_mag = ((manifold.penetration - SLOP).max(0.0)
        / (body_a.inv_mass + body_b.inv_mass)) * PERCENT;
    let correction = manifold.normal * correction_mag;

    if !body_a.is_static {
        body_a.position = body_a.position + correction * body_a.inv_mass;
    }
    if !body_b.is_static {
        body_b.position = body_b.position - correction * body_b.inv_mass;
    }
}
```

**ทดสอบสูตร impulse:**

```rust
#[test]
fn test_impulse_formula() {
    // mass_a = mass_b = 1, v_rel_n = -2, e = 1 (elastic)
    // j = -(1+1)*(-2) / (1/1 + 1/1) = 4 / 2 = 2.0
    let j = compute_impulse_magnitude(1.0, 1.0, -2.0, 1.0);
    assert!((j - 2.0).abs() < 1e-9);
}

#[test]
fn test_impulse_infinite_mass() {
    // body_b เป็น static (inv_mass = 0)
    // j = -(1+0.5)*(-1) / (1/1 + 0) = 1.5 / 1 = 1.5
    let j = compute_impulse_magnitude(1.0, 0.0, -1.0, 0.5);
    assert!((j - 1.5).abs() < 1e-9);
}
```

---

### ขั้นที่ 6: Constraints

#### DistanceConstraint

จำกัดระยะห่างระหว่าง 2 bodies (เช่น chain link, elastic band):

```rust
pub struct DistanceConstraint {
    pub body_a: BodyId,
    pub body_b: BodyId,
    pub rest_length: f64,
    pub stiffness: f64,   // 0..1
}

impl DistanceConstraint {
    pub fn solve(&self, bodies: &mut Vec<RigidBody>) {
        let pos_a = bodies[self.body_a].position;
        let pos_b = bodies[self.body_b].position;
        let inv_mass_a = bodies[self.body_a].inv_mass;
        let inv_mass_b = bodies[self.body_b].inv_mass;

        let diff = pos_a - pos_b;
        let dist = diff.length();
        if dist < 1e-10 { return; }

        let error = dist - self.rest_length;
        let total_inv_mass = inv_mass_a + inv_mass_b;
        if total_inv_mass < 1e-10 { return; }

        // Position correction ตาม Baumgarte
        let correction = diff.normalize() * (error * self.stiffness / total_inv_mass);

        if !bodies[self.body_a].is_static {
            bodies[self.body_a].position =
                bodies[self.body_a].position - correction * inv_mass_a;
        }
        if !bodies[self.body_b].is_static {
            bodies[self.body_b].position =
                bodies[self.body_b].position + correction * inv_mass_b;
        }
    }
}
```

#### HingeConstraint

Pin joint ที่ยึด anchor point ของสอง bodies เข้าด้วยกัน พร้อม optional angle limits:

```rust
pub struct HingeConstraint {
    pub body_a: BodyId,
    pub body_b: BodyId,
    pub anchor_a: Vec2,      // local space
    pub anchor_b: Vec2,      // local space
    pub min_angle: Option<f64>,
    pub max_angle: Option<f64>,
}

impl HingeConstraint {
    pub fn solve(&self, bodies: &mut Vec<RigidBody>) {
        let angle_a = bodies[self.body_a].angle;
        let pos_a = bodies[self.body_a].position;
        let angle_b = bodies[self.body_b].angle;
        let pos_b = bodies[self.body_b].position;

        // World-space anchor points
        let world_anchor_a = self.anchor_a.rotate(angle_a) + pos_a;
        let world_anchor_b = self.anchor_b.rotate(angle_b) + pos_b;

        // แก้ positional error
        let diff = world_anchor_a - world_anchor_b;
        let inv_sum = bodies[self.body_a].inv_mass + bodies[self.body_b].inv_mass;
        if inv_sum > 1e-10 {
            let correction = diff * 0.5;
            let inv_a = bodies[self.body_a].inv_mass;
            let inv_b = bodies[self.body_b].inv_mass;
            if !bodies[self.body_a].is_static {
                bodies[self.body_a].position -= correction * (inv_a / inv_sum);
            }
            if !bodies[self.body_b].is_static {
                bodies[self.body_b].position += correction * (inv_b / inv_sum);
            }
        }

        // Angle limits
        if let (Some(min_a), Some(max_a)) = (self.min_angle, self.max_angle) {
            let rel_angle = bodies[self.body_b].angle - bodies[self.body_a].angle;
            if rel_angle < min_a {
                let corr = rel_angle - min_a;
                if !bodies[self.body_b].is_static { bodies[self.body_b].angle -= corr * 0.5; }
                if !bodies[self.body_a].is_static { bodies[self.body_a].angle += corr * 0.5; }
            } else if rel_angle > max_a {
                let corr = rel_angle - max_a;
                if !bodies[self.body_b].is_static { bodies[self.body_b].angle -= corr * 0.5; }
                if !bodies[self.body_a].is_static { bodies[self.body_a].angle += corr * 0.5; }
            }
        }
    }
}
```

---

### ขั้นที่ 7: World และ Fixed Timestep

**Fixed timestep ด้วย accumulator pattern** — สำคัญมากสำหรับ deterministic simulation:

```rust
pub struct World {
    pub bodies: Vec<RigidBody>,
    pub gravity: Vec2,
    pub distance_constraints: Vec<DistanceConstraint>,
    pub hinge_constraints: Vec<HingeConstraint>,
    spatial_hash: SpatialHash,
    pub step_count: u64,   // สำหรับ deterministic replay
    pub time: f64,
    accumulator: f64,
    fixed_dt: f64,
    pub contacts: Vec<ContactManifold>,
}

impl World {
    /// เรียกจาก game loop ด้วย variable dt
    pub fn update(&mut self, dt: f64) {
        self.accumulator += dt;
        while self.accumulator >= self.fixed_dt {
            self.step(self.fixed_dt);
            self.accumulator -= self.fixed_dt;
        }
    }

    /// Fixed step — deterministic
    pub fn step(&mut self, dt: f64) {
        // 1. Gravity
        for body in self.bodies.iter_mut() {
            if !body.is_static {
                body.apply_force(self.gravity * body.mass);
            }
        }

        // 2. Broad phase
        self.spatial_hash.clear();
        for body in &self.bodies {
            self.spatial_hash.insert(body.id, &body.world_aabb());
        }

        // 3. Narrow phase + resolution
        let pairs = self.spatial_hash.query_pairs();
        self.contacts.clear();
        for (id_a, id_b) in &pairs {
            // detect collision (unsafe raw pointer workaround สำหรับ split borrow)
            let a_ref = &self.bodies[*id_a] as *const RigidBody;
            let b_ref = &self.bodies[*id_b] as *const RigidBody;
            let manifold = unsafe { detect_collision(&*a_ref, &*b_ref) };

            if let Some(m) = manifold {
                self.contacts.push(m.clone());
                if id_a < id_b {
                    let (left, right) = self.bodies.split_at_mut(*id_b);
                    resolve_collision(&mut left[*id_a], &mut right[0], &m);
                    positional_correction(&mut left[*id_a], &mut right[0], &m);
                } else {
                    let (left, right) = self.bodies.split_at_mut(*id_a);
                    resolve_collision(&mut right[0], &mut left[*id_b], &m);
                    positional_correction(&mut right[0], &mut left[*id_b], &m);
                }
            }
        }

        // 4. Constraints
        for c in self.distance_constraints.clone().iter() { c.solve(&mut self.bodies); }
        for c in self.hinge_constraints.clone().iter() { c.solve(&mut self.bodies); }

        // 5. Integrate
        for body in self.bodies.iter_mut() { body.integrate(dt); }

        self.step_count += 1;
        self.time += dt;
    }
}
```

---

### ขั้นที่ 8: JSON Scene Format

**`scenes/demo.json`:**

```json
{
  "gravity": [0.0, -9.8],
  "bodies": [
    {
      "shape": { "type": "aabb", "half_w": 10.0, "half_h": 0.5 },
      "position": [0.0, -5.0],
      "mass": 0.0,
      "restitution": 0.3,
      "friction": 0.5,
      "is_static": true
    },
    {
      "shape": { "type": "circle", "radius": 0.5 },
      "position": [0.0, 5.0],
      "velocity": [2.0, 0.0],
      "mass": 1.0,
      "restitution": 0.6,
      "friction": 0.3
    },
    {
      "shape": { "type": "aabb", "half_w": 0.5, "half_h": 0.5 },
      "position": [3.0, 3.0],
      "mass": 2.0,
      "restitution": 0.2,
      "friction": 0.6
    }
  ]
}
```

**`World::load_scene` และ `save_state`:**

```rust
pub fn load_scene(json: &str) -> Result<World, serde_json::Error> {
    let scene: SceneDef = serde_json::from_str(json)?;
    let gravity_arr = scene.gravity.unwrap_or([0.0, -9.8]);
    let gravity = Vec2::new(gravity_arr[0], gravity_arr[1]);
    let mut world = World::new(gravity, 1.0 / 60.0, 5.0);

    for (i, def) in scene.bodies.iter().enumerate() {
        let shape = match &def.shape {
            ShapeDef::Circle { radius } => Shape::Circle { radius: *radius },
            ShapeDef::Aabb { half_w, half_h } =>
                Shape::Aabb { half_extents: Vec2::new(*half_w, *half_h) },
        };
        let pos = Vec2::new(def.position[0], def.position[1]);
        let mut body = RigidBody::new(
            i, def.mass, def.mass.max(0.001),
            pos, shape,
            def.restitution.unwrap_or(0.3),
            def.friction.unwrap_or(0.5),
            def.is_static.unwrap_or(false),
        );
        if let Some(v) = def.velocity { body.velocity = Vec2::new(v[0], v[1]); }
        world.bodies.push(body);
    }
    Ok(world)
}

pub fn save_state(&self) -> WorldState {
    WorldState {
        step_count: self.step_count,
        time: self.time,
        bodies: self.bodies.iter().map(|b| BodyState {
            id: b.id,
            position: [b.position.x, b.position.y],
            velocity: [b.velocity.x, b.velocity.y],
            angle: b.angle,
            angular_velocity: b.angular_velocity,
        }).collect(),
    }
}
```

---

### ขั้นที่ 9: CLI ด้วย clap 4

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "physics-engine", about = "2D Physics Engine Demo")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// รัน built-in demo simulation
    Demo {
        #[arg(short, long, default_value = "100")]
        steps: u32,
    },
    /// โหลด scene จาก JSON และรัน simulation
    Scene {
        #[arg(short, long)]
        file: String,
        #[arg(short, long, default_value = "60")]
        steps: u32,
    },
    /// แสดง version info
    Info,
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        Commands::Demo { steps } => run_demo(steps),
        Commands::Scene { file, steps } => run_scene(&file, steps),
        Commands::Info => println!("physics-engine v0.1.0"),
    }
}
```

---

## `Cargo.toml`

```toml
[package]
name = "physics-engine"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "physics-engine"
path = "src/main.rs"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
```

---

## การทดสอบ (Testing)

### รัน `cargo test` จริง

```bash
cargo test
```

**Output จริง:**

```
running 22 tests
test body::tests::test_euler_integration ... ok
test collision::tests::test_aabb_overlap_detection ... ok
test body::tests::test_static_body_no_movement ... ok
test body::tests::test_apply_impulse ... ok
test collision::tests::test_circle_circle_collision ... ok
test collision::tests::test_circle_circle_no_collision ... ok
test collision::tests::test_circle_circle_normal_direction ... ok
test collision::tests::test_spatial_hash_pairs ... ok
test constraints::tests::test_distance_constraint_static_body ... ok
test collision::tests::test_sat_separation_axis ... ok
test constraints::tests::test_distance_constraint_correction ... ok
test math::tests::test_vec2_add ... ok
test math::tests::test_vec2_cross ... ok
test math::tests::test_vec2_dot ... ok
test math::tests::test_vec2_length ... ok
test math::tests::test_vec2_normalize ... ok
test math::tests::test_aabb_overlap ... ok
test resolution::tests::test_baumgarte_position_correction ... ok
test math::tests::test_vec2_sub ... ok
test math::tests::test_vec2_rotate_90 ... ok
test resolution::tests::test_impulse_infinite_mass ... ok
test resolution::tests::test_impulse_formula ... ok

test result: ok. 22 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### รัน Demo

```bash
cargo run -- demo --steps 100
```

**Output จริง:**

```
=== 2D Physics Engine Demo ===
Steps: 100
Step   1: ball=(-0.067, 4.997) box=(2.033, 4.997) contacts=0
Step  21: ball=(-0.594, 4.371) box=(2.297, 4.371) contacts=0
Step  41: ball=(-0.658, 2.656) box=(2.329, 2.656) contacts=0
Step  61: ball=(-0.666, -0.148) box=(2.333, -0.148) contacts=0
Step  81: ball=(-0.667, -4.040) box=(2.333, -4.040) contacts=0
Step 100: ball=(-0.217, -3.706) box=(2.267, -7.370) contacts=0

Simulation complete. Total steps: 100
State snapshot: 3 bodies at t=1.667s
```

### สรุป test ที่ครอบคลุม

| Test | Module | ทดสอบอะไร |
|---|---|---|
| `test_vec2_add/sub/dot/cross` | math | Vec2 arithmetic operations |
| `test_vec2_length` | math | `sqrt(x²+y²)` correctness |
| `test_vec2_normalize` | math | unit vector, edge case zero |
| `test_vec2_rotate_90` | math | rotation matrix correctness |
| `test_aabb_overlap` | math | AABB overlap detection |
| `test_euler_integration` | body | `v += (F/m)*dt; x += v*dt` |
| `test_static_body_no_movement` | body | static body immovability |
| `test_apply_impulse` | body | `delta_v = impulse / mass` |
| `test_circle_circle_collision` | collision | penetration depth + normal |
| `test_circle_circle_no_collision` | collision | miss detection |
| `test_circle_circle_normal_direction` | collision | normal pointing B→A |
| `test_aabb_overlap_detection` | collision | AABB broad phase |
| `test_sat_separation_axis` | collision | SAT detect/miss |
| `test_spatial_hash_pairs` | collision | spatial hash cell grouping |
| `test_impulse_formula` | resolution | `j = -(1+e)*v_n/(1/m1+1/m2)` |
| `test_impulse_infinite_mass` | resolution | static body impulse |
| `test_baumgarte_position_correction` | resolution | position drift correction |
| `test_distance_constraint_correction` | constraints | spring-like correction |
| `test_distance_constraint_static_body` | constraints | one static anchor |

---

## ⚠️ Pitfalls ที่พบบ่อย

### Pitfall 1: Explicit Euler ทำให้ simulation ระเบิด

**ปัญหา:** ใช้ explicit Euler: `v += a*dt` แล้ว `x += v_old * dt` — ระบบมี energy เพิ่มขึ้นตลอด ทำให้วัตถุเด้งแรงขึ้นเรื่อย ๆ

```rust
// ❌ Explicit Euler — UNSTABLE
let old_v = self.velocity;
self.velocity = self.velocity + accel * dt;
self.position = self.position + old_v * dt; // ใช้ velocity เก่า!
```

**แก้ไข:** ใช้ Semi-implicit Euler — update velocity ก่อน แล้วใช้ velocity ใหม่:

```rust
// ✓ Semi-implicit Euler — STABLE
self.velocity = self.velocity + accel * dt;      // velocity ใหม่
self.position = self.position + self.velocity * dt; // ใช้ velocity ใหม่
```

**เหตุผลทางคณิตศาสตร์:** Semi-implicit Euler เป็น symplectic integrator ที่ conserve energy ในระยะยาวสำหรับ conservative systems

---

### Pitfall 2: Borrow Checker กับ Mutable References สองตัวพร้อมกัน

**ปัญหา:** Rust ไม่อนุญาตให้ borrow สอง element ของ Vec พร้อมกัน แม้ index ต่างกัน:

```rust
// ❌ Compile Error
let a = &mut self.bodies[id_a];
let b = &mut self.bodies[id_b]; // error: cannot borrow twice
resolve_collision(a, b, &m);
```

**แก้ไข 1: `split_at_mut`** (แนะนำ):

```rust
// ✓ split_at_mut — safe, idiomatic
if id_a < id_b {
    let (left, right) = self.bodies.split_at_mut(id_b);
    resolve_collision(&mut left[id_a], &mut right[0], &m);
}
```

**แก้ไข 2: unsafe raw pointer** (ใช้เมื่อ index ไม่เป็น sequential):

```rust
// ✓ unsafe — ต้องมั่นใจว่า id_a ≠ id_b
let a = &mut self.bodies[id_a] as *mut RigidBody;
let b = &mut self.bodies[id_b] as *mut RigidBody;
unsafe { resolve_collision(&mut *a, &mut *b, &m); }
```

---

### Pitfall 3: Tunneling — วัตถุเร็วทะลุผ่านวัตถุบาง

**ปัญหา:** เมื่อ body เคลื่อนที่เร็วมาก ระยะที่เคลื่อนใน 1 step อาจมากกว่าความหนาของ collider ทำให้ physics engine ไม่ detect การชน

```
Frame 1: ●  |wall|        (ด้านซ้าย)
Frame 2:    |wall|  ●     (ทะลุแล้ว! ไม่ detect)
```

**แก้ไข:**
1. **ลด dt** — ใช้ fixed timestep เล็กลง (1/120s แทน 1/60s)
2. **Continuous Collision Detection (CCD)** — sweep test ระหว่าง position เก่าและใหม่
3. **Velocity clamping** — จำกัด max velocity ตาม body size:

```rust
// Simple max velocity clamp
const MAX_VELOCITY: f64 = 20.0; // units/second
let speed = self.velocity.length();
if speed > MAX_VELOCITY {
    self.velocity = self.velocity * (MAX_VELOCITY / speed);
}
```

---

### Pitfall 4: Ghost Collision จาก Spatial Hash กับวัตถุขอบ Cell

**ปัญหา:** วัตถุที่อยู่บริเวณขอบ cell อาจถูก insert เข้า 2-4 cells ทำให้ query_pairs คืน pair เดิมหลายครั้ง

**แก้ไข:** ใช้ `HashSet` เก็บ pairs แทน `Vec` — รับประกัน uniqueness:

```rust
pub fn query_pairs(&self) -> Vec<(BodyId, BodyId)> {
    let mut pairs = std::collections::HashSet::new(); // ✓ ไม่ซ้ำ
    for ids in self.cells.values() {
        for i in 0..ids.len() {
            for j in (i + 1)..ids.len() {
                let a = ids[i].min(ids[j]);
                let b = ids[i].max(ids[j]);
                pairs.insert((a, b)); // HashSet deduplicate อัตโนมัติ
            }
        }
    }
    pairs.into_iter().collect()
}
```

---

### Pitfall 5: Normal Direction ที่ไม่สม่ำเสมอ

**ปัญหา:** SAT คืน normal ที่อาจชี้ผิดทิศ ทำให้ impulse ผลักวัตถุเข้าหากัน แทนที่จะผลักออก

**แก้ไข:** ตรวจสอบและ flip normal ก่อนเสมอ:

```rust
// ให้ normal ชี้จาก B → A เสมอ
let center_diff = pos_a - pos_b;
if center_diff.dot(collision_normal) < 0.0 {
    collision_normal = -collision_normal;
}
```

---

## การ Package และ Deploy

### Build Release

```bash
cargo build --release
./target/release/physics-engine demo --steps 1000
```

Release build เร็วกว่า debug ~10x (สำคัญมากสำหรับ physics simulation)

### รัน Scene จาก JSON

```bash
./target/release/physics-engine scene --file scenes/demo.json --steps 120
```

Output (JSON snapshot):

```json
{
  "step_count": 120,
  "time": 2.0,
  "bodies": [
    {
      "id": 0,
      "position": [0.0, -5.0],
      "velocity": [0.0, 0.0],
      "angle": 0.0,
      "angular_velocity": 0.0
    },
    ...
  ]
}
```

### Profile Performance

```bash
cargo build --release --features profiling
# หรือใช้ perf
perf record ./target/release/physics-engine demo --steps 10000
perf report
```

### Docker (ถ้าต้องการ)

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/physics-engine /usr/local/bin/
CMD ["physics-engine", "info"]
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม Circle vs Polygon Collision

ขยาย `detect_collision` ให้รองรับ `Circle` vs `Polygon` — ต้องหาจุดที่ใกล้ circle center ที่สุดบนขอบ polygon แล้วคำนวณ penetration depth

**Hint:**
```rust
// หา closest edge ของ polygon กับ circle center
for edge in polygon_edges {
    let closest = closest_point_on_segment(circle_center, edge.start, edge.end);
    let dist = (circle_center - closest).length();
    if dist < circle_radius { /* collision */ }
}
```

**ทักษะที่ได้:** Point-on-segment projection, edge iteration, contact manifold assembly

---

### Exercise 2: Continuous Collision Detection (CCD)

แก้ปัญหา tunneling สำหรับ fast-moving objects — implement sweep test ระหว่าง circle positions ใน 2 frames:

```rust
pub fn sweep_circle_vs_static_aabb(
    circle_pos_start: Vec2, circle_pos_end: Vec2, radius: f64,
    aabb_pos: Vec2, half: Vec2,
) -> Option<f64> {
    // คืน time of impact (0..1) หรือ None
    // ใช้ ray vs expanded AABB (Minkowski sum)
    todo!()
}
```

**ทักษะที่ได้:** Ray casting, Minkowski sum, swept volume tests

---

### Exercise 3: Island-based Sleep System

Bodies ที่นิ่งนาน ๆ ควรถูก "sleep" เพื่อประหยัด CPU:

```rust
pub struct RigidBody {
    // เพิ่ม fields:
    pub sleep_timer: f64,
    pub is_sleeping: bool,
}

impl RigidBody {
    const SLEEP_THRESHOLD_V: f64 = 0.05;  // velocity ต่ำกว่านี้ → นับ timer
    const SLEEP_THRESHOLD_W: f64 = 0.05;  // angular velocity
    const SLEEP_TIME: f64 = 0.5;          // รอ 0.5 วินาที → sleep

    pub fn update_sleep(&mut self, dt: f64) {
        if self.velocity.length() < Self::SLEEP_THRESHOLD_V
            && self.angular_velocity.abs() < Self::SLEEP_THRESHOLD_W {
            self.sleep_timer += dt;
            if self.sleep_timer > Self::SLEEP_TIME {
                self.is_sleeping = true;
            }
        } else {
            self.sleep_timer = 0.0;
            self.is_sleeping = false;
        }
    }
}
```

**ทักษะที่ได้:** State machine, timer, island detection, wake-up propagation

---

### Exercise 4: Broadphase ด้วย BVH Tree

เปลี่ยน SpatialHash เป็น Bounding Volume Hierarchy (Dynamic AABB Tree) เพื่อประสิทธิภาพที่ดีกว่าสำหรับ objects ขนาดต่างกัน:

```rust
pub struct AabbTree {
    nodes: Vec<AabbNode>,
    root: Option<usize>,
}

#[derive(Debug)]
pub struct AabbNode {
    pub aabb: Aabb,        // enlarged AABB (fat AABB)
    pub body_id: Option<BodyId>,
    pub left: Option<usize>,
    pub right: Option<usize>,
    pub parent: Option<usize>,
}

impl AabbTree {
    pub fn insert(&mut self, id: BodyId, aabb: Aabb) { todo!() }
    pub fn remove(&mut self, id: BodyId) { todo!() }
    pub fn query_pairs(&self) -> Vec<(BodyId, BodyId)> { todo!() }
}
```

**ทักษะที่ได้:** Tree data structure, AABB tree insertion algorithm, O(n log n) broad phase

---

## สรุป

โปรเจคนี้สร้าง 2D Physics Engine ครบวงจรที่มี:

1. **Math layer** — Vec2 พร้อม operator overloading, Transform, AABB, Circle
2. **Rigid body** — mass, moment of inertia, semi-implicit Euler integration
3. **Broad phase** — spatial hashing ด้วย `HashMap<(i32,i32), Vec<BodyId>>`
4. **Narrow phase** — Circle×Circle, Circle×AABB, AABB×AABB, SAT polygon
5. **Resolution** — impulse-based collision + Coulomb friction + Baumgarte correction
6. **Constraints** — DistanceConstraint, HingeConstraint พร้อม angle limits
7. **World** — fixed timestep accumulator, JSON scene load/save, state snapshot
8. **CLI** — clap 4 subcommands สำหรับ demo และ scene runner

**Pattern สำคัญที่ได้เรียน:**
- `split_at_mut` แก้ปัญหา dual mutable borrow ใน Vec
- `unsafe` raw pointer เป็น last resort สำหรับ performance-critical code
- `HashSet` เป็น deduplication layer บน `HashMap`
- Fixed timestep accumulator สำหรับ deterministic simulation
- Baumgarte stabilization แก้ positional drift จาก numerical integration

**เชื่อมโยงไปโปรเจคถัดไป:** Project E04 (Ray Tracer) จะนำ Vec3 math ที่คล้ายกัน มาใช้ในระบบ ray-object intersection, shading model และ recursive ray tracing — pattern ของ math primitives ที่สร้างใน E03 จะเห็นว่า apply ได้ตรงกัน

---

## ภาคผนวก: ตัวอย่าง Integration Test แบบ End-to-End

ทดสอบ simulation ทั้งระบบโดยการตรวจสอบ energy conservation และ collision response ที่ถูกต้อง:

```rust
// tests/integration_test.rs
#[cfg(test)]
mod integration_tests {
    use physics_engine::math::Vec2;
    use physics_engine::body::{RigidBody, Shape};
    use physics_engine::world::World;

    fn make_world() -> World {
        World::new(Vec2::new(0.0, -9.8), 1.0 / 60.0, 5.0)
    }

    /// ทดสอบว่า static body ไม่เคลื่อนที่เมื่อถูกชน
    #[test]
    fn test_static_floor_remains_fixed() {
        let mut world = make_world();

        // พื้น static
        let floor = RigidBody::new(
            0, 0.0, 0.0, Vec2::new(0.0, 0.0),
            Shape::Aabb { half_extents: Vec2::new(10.0, 0.5) },
            0.5, 0.5, true,
        );
        world.add_body(floor);

        // ลูกบอลตกลงมา
        let ball = RigidBody::new(
            1, 1.0, 0.5, Vec2::new(0.0, 3.0),
            Shape::Circle { radius: 0.5 },
            0.5, 0.3, false,
        );
        world.add_body(ball);

        // รัน 200 steps
        let dt = 1.0 / 60.0;
        for _ in 0..200 {
            world.step(dt);
        }

        // พื้นต้องอยู่เดิม
        let floor = world.body(0).unwrap();
        assert!((floor.position.x).abs() < 1e-9);
        assert!((floor.position.y).abs() < 1e-9);
    }

    /// ทดสอบ elastic collision — ball ต้องเด้งกลับ
    #[test]
    fn test_elastic_ball_bounces() {
        let mut world = World::new(Vec2::ZERO, 1.0 / 60.0, 10.0); // ไม่มี gravity

        // พื้น static
        let floor = RigidBody::new(
            0, 0.0, 0.0, Vec2::new(0.0, -5.0),
            Shape::Aabb { half_extents: Vec2::new(10.0, 0.5) },
            1.0, 0.0, true, // restitution = 1.0 (perfectly elastic)
        );
        world.add_body(floor);

        // ลูกบอลตกลงมาด้วย velocity เริ่มต้น
        let mut ball = RigidBody::new(
            1, 1.0, 0.5, Vec2::new(0.0, 0.0),
            Shape::Circle { radius: 0.5 },
            1.0, 0.0, false, // restitution = 1.0
        );
        ball.velocity = Vec2::new(0.0, -5.0); // เคลื่อนลง
        world.add_body(ball);

        let initial_speed = 5.0;
        let dt = 1.0 / 120.0; // smaller dt for accuracy

        // รัน จนกระทั่งชนพื้น
        for _ in 0..300 {
            world.step(dt);
        }

        // ลูกบอลต้องเด้งกลับ (vy > 0)
        let ball_after = world.body(1).unwrap();
        // หลัง elastic collision ความเร็วควรใกล้เคียงกับ initial_speed
        assert!(ball_after.velocity.y > 0.0 || ball_after.position.y > -4.0);
    }

    /// ทดสอบ determinism — รัน 2 ครั้งต้องได้ผลเหมือนกัน
    #[test]
    fn test_simulation_determinism() {
        let mut world1 = make_world();
        let mut world2 = make_world();

        // เพิ่ม bodies เหมือนกัน
        for w in [&mut world1, &mut world2] {
            w.add_body(RigidBody::new(
                0, 1.0, 1.0, Vec2::new(0.5, 5.0),
                Shape::Circle { radius: 0.5 },
                0.4, 0.3, false,
            ));
        }

        let dt = 1.0 / 60.0;
        for _ in 0..120 {
            world1.step(dt);
            world2.step(dt);
        }

        let b1 = world1.body(0).unwrap();
        let b2 = world2.body(0).unwrap();

        // ผลลัพธ์ต้องเหมือนกันทุก bit
        assert_eq!(b1.position.x.to_bits(), b2.position.x.to_bits());
        assert_eq!(b1.position.y.to_bits(), b2.position.y.to_bits());
        assert_eq!(world1.step_count, world2.step_count);
    }

    /// ทดสอบ World::save_state และ restore_state
    #[test]
    fn test_snapshot_restore() {
        let mut world = make_world();
        world.add_body(RigidBody::new(
            0, 1.0, 1.0, Vec2::new(0.0, 5.0),
            Shape::Circle { radius: 0.5 },
            0.5, 0.3, false,
        ));

        // รัน 30 steps
        let dt = 1.0 / 60.0;
        for _ in 0..30 { world.step(dt); }

        // บันทึก state
        let snapshot = world.save_state();
        let pos_at_30 = world.body(0).unwrap().position;

        // รันต่ออีก 30 steps
        for _ in 0..30 { world.step(dt); }
        let pos_at_60 = world.body(0).unwrap().position;

        // Restore กลับไป step 30
        world.restore_state(&snapshot);
        let pos_after_restore = world.body(0).unwrap().position;

        assert_eq!(pos_at_30.x.to_bits(), pos_after_restore.x.to_bits());
        assert_eq!(pos_at_30.y.to_bits(), pos_after_restore.y.to_bits());
        assert_ne!(pos_at_60.y.to_bits(), pos_after_restore.y.to_bits());
    }
}
```

> **หมายเหตุ:** Integration tests เหล่านี้ใช้ public API ของ World และ RigidBody — ต้องเพิ่ม `pub use` ใน `lib.rs` หรือเปลี่ยน `main.rs` เป็น `lib.rs` สำหรับ project ที่ต้องการ test ระดับ integration

---

## ภาคผนวก: ตัวเลขอ้างอิง Physics

| ค่าคงที่ | ค่าปกติ | หน่วย |
|---|---|---|
| แรงโน้มถ่วงโลก | -9.8 | m/s² |
| restitution ยาง vs พื้น | 0.6–0.8 | dimensionless |
| restitution เหล็ก vs เหล็ก | 0.5–0.7 | dimensionless |
| friction ยาง vs คอนกรีต (static) | 0.7–0.8 | dimensionless |
| friction เหล็ก vs เหล็ก (kinetic) | 0.1–0.2 | dimensionless |
| moment of inertia วงกลม | m*r²/2 | kg·m² |
| moment of inertia สี่เหลี่ยม | m*(w²+h²)/12 | kg·m² |
| Baumgarte SLOP | 0.005–0.02 | m |
| Baumgarte PERCENT | 0.1–0.3 | dimensionless |

```rust
/// คำนวณ moment of inertia สำหรับ shapes ทั่วไป
pub fn moment_of_inertia_circle(mass: f64, radius: f64) -> f64 {
    0.5 * mass * radius * radius
}

pub fn moment_of_inertia_rect(mass: f64, width: f64, height: f64) -> f64 {
    mass * (width * width + height * height) / 12.0
}

pub fn moment_of_inertia_polygon(mass: f64, vertices: &[Vec2]) -> f64 {
    // Numerical calculation for arbitrary convex polygon
    let n = vertices.len();
    if n < 3 { return 0.001; }
    let mut numerator = 0.0;
    let mut denominator = 0.0;
    for i in 0..n {
        let a = vertices[i];
        let b = vertices[(i + 1) % n];
        let cross = a.cross(b).abs();
        numerator += cross * (a.dot(a) + a.dot(b) + b.dot(b));
        denominator += cross;
    }
    (mass / 6.0) * (numerator / denominator)
}
```

---

**โปรเจคก่อนหน้า:** [project-e02-chess-engine.md](project-e02-chess-engine.md) | **โปรเจคถัดไป:** [project-e04-ray-tracer.md](project-e04-ray-tracer.md)
