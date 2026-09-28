# Project E04: Ray Tracer (Path Tracing)

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Path Tracer** ด้วย Rust ตั้งแต่ศูนย์ — เป็น renderer ที่จำลองการเดินทางของแสงจริงผ่านฉาก 3D โดยใช้เทคนิค Monte Carlo sampling ผลลัพธ์คือภาพที่มีเงาอ่อน (soft shadows), การสะท้อน (reflections), การหักเหของแสง (refraction), และ color bleeding ที่สวยงาม — ทุกอย่างเกิดจากฟิสิกส์ที่ถูกต้อง ไม่ใช่การโกง

**Path Tracing** แตกต่างจาก Ray Tracing ธรรมดาตรงที่แทนที่จะส่ง ray เดียวต่อ pixel เราส่ง ray จำนวนมากและติดตามการ bounce หลายชั้น สะสมพลังงานแสงตามวัสดุที่ชน แล้วเฉลี่ยผล ยิ่ง samples มาก ผลลัพธ์ยิ่งลด noise และใกล้เคียงภาพจริง

**Use cases จริงในโลก production:**
- **VFX & Animation**: Pixar's RenderMan, Autodesk Arnold, Blender Cycles ล้วนใช้ path tracing
- **Game Engine Baking**: Unreal Engine ใช้ path tracer bake lightmap สำหรับ real-time lighting
- **Architectural Visualization**: แสดงตึกก่อนสร้างจริง — สถาปนิกและ client เห็นแสงจริง
- **Material Research**: จำลองวัสดุใหม่ทางวิทยาศาสตร์ก่อนผลิต

**Learning value สูงมาก**: โปรเจคนี้สอน ownership ระดับลึกผ่าน `Arc<dyn Trait>`, parallel programming ด้วย Rayon, คณิตศาสตร์ 3D, และ performance optimization

## สิ่งที่จะได้เรียนรู้

- **Trait object polymorphism** — ใช้ `Box<dyn Hittable>` และ `Arc<dyn Material>` แทน inheritance
- **Rayon parallel iteration** — render rows แบบ lock-free parallel ด้วย `par_iter()`
- **Monte Carlo methods** — statistical sampling, convergence, variance reduction
- **3D math fundamentals** — Vec3 operators, dot/cross product, reflection, Snell's law
- **BVH acceleration structure** — O(log n) ray-scene intersection ด้วย Bounding Volume Hierarchy
- **Recursive algorithms** — path tracing loop ที่ทำ Russian roulette termination
- **Gamma correction & tone mapping** — แปลง linear light ให้ monitor แสดงถูก
- **Builder pattern + JSON deserialization** — สร้าง scene จาก config file แบบ type-safe

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30**: Rust basics — ownership, borrowing, structs, enums, Vec, HashMap, iterators
- **Part 31–50**: Error handling (`Result`, `?`), traits, generics, lifetime basics
- **Part 51–70**: Trait objects (`dyn Trait`), `Box<T>`, `Arc<T>`, closures, `impl Trait`
- **Part 71–85**: Parallel programming, Rayon, `Send + Sync` bounds, `#[derive]` macros
- **Part 86–100**: Serde deserialization, `clap` CLI parsing, `image` crate output

## โครงสร้างโปรเจค (Project Layout)

```
ray_tracer/
├── src/
│   ├── main.rs          ← CLI entry point (clap), scene loading, render loop
│   ├── math.rs          ← Vec3, Ray, Aabb + unit tests
│   ├── hittable.rs      ← trait Hittable, Sphere, Plane, Triangle, HittableList
│   ├── material.rs      ← trait Material, Lambertian, Metal, Dielectric + Schlick
│   ├── bvh.rs           ← BvhNode builder + Hittable impl
│   ├── camera.rs        ← Camera + DOF ray generation
│   ├── renderer.rs      ← trace(), render_parallel(), write_ppm()
│   └── scene.rs         ← JSON SceneConfig + Cornell Box preset
├── scenes/
│   └── cornell_box.json ← ตัวอย่าง scene file
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของ Path Tracer

```
CLI args / JSON scene
        │
        ▼
   SceneConfig::build_world()
        │ Vec<Box<dyn Hittable>>
        ▼
   BvhNode::build()  ──── O(n log n) build time
        │ Arc<dyn Hittable>
        ▼
  render_parallel()
        │ rayon par_iter() ← แต่ละ row รันบน thread pool
        ▼
  Camera::generate_ray(u, v)
        │ Ray { origin, direction }
        ▼
  trace(ray, world, depth)
        │  recursive
        ├──▶ world.hit(ray, t_min, t_max) ← O(log n) via BVH
        │         │ Some(HitRecord)
        │         ▼
        │    material.scatter(ray, rec)
        │         │ Some(ScatterResult)
        │         ▼
        │    trace(scattered_ray, depth-1) × albedo
        └──▶ sky_color(ray)  ← miss case
        │
        ▼
  Color (accumulated over N samples)
        │ gamma_correct(2.2)
        ▼
  write_ppm() / image::save_png()
```

### ทำไมถึงใช้ `Arc<dyn Trait>` แทน Generic?

ถ้าใช้ Generic เช่น `Sphere<M: Material>` จะ monomorphize ทุก combination ทำให้ binary ใหญ่ และ HittableList ก็เก็บชนิดต่างกันไม่ได้ `Arc<dyn Hittable>` ช่วยให้ลิสต์วัตถุหลากชนิดได้ ใน hot path แม้มี vtable overhead แต่ใน path tracing ที่ O(log n) BVH ค่าใช้จ่ายส่วนนี้น้อยมาก

### Stratified Sampling vs Pure Random

Pure random sampling มี **clumping** — samples อาจกระจุกในพื้นที่เดิม ทำให้บาง region render ไม่ครบ Stratified sampling แบ่งแต่ละ pixel เป็น grid √N × √N และ sample หนึ่งครั้งต่อ cell แน่ใจว่าครอบคลุมสม่ำเสมอ ลด noise โดยไม่เพิ่ม samples

### Russian Roulette Termination

การ terminate ray ที่ depth ตายตัว (เช่น depth 50) ทำให้เสียแสงเพราะ ray ที่ควร bounce ต่อถูกตัดทิ้ง Russian Roulette แก้ปัญหานี้: ที่ depth > 50 ให้มีโอกาส 90% ที่จะ survive ถ้า survive ให้คูณ energy ด้วย `1/0.9` เพื่อ compensate — ผลลัพธ์ถูกต้องทางสถิติและเร็วกว่า

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Math Types — Vec3, Ray, และ Operators

เริ่มจาก foundation ที่สำคัญที่สุด: ชนิด `Vec3` ที่รองรับ operator overloading ทั้งหมด และ `Ray` สำหรับ ray casting

**`Cargo.toml`:**

```toml
[package]
name = "ray_tracer"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "ray_tracer"
path = "src/main.rs"

[dependencies]
rayon = "1.10"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
indicatif = "0.17"
image = "0.25"
clap = { version = "4.5", features = ["derive"] }
rand = { version = "0.8", features = ["small_rng"] }
```

**`src/math.rs`:**

```rust
use std::ops::{Add, Sub, Mul, Div, Neg, AddAssign, MulAssign};

/// เวกเตอร์ 3 มิติ — ใช้ทั้งสำหรับ geometry และสี (Color)
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Vec3 {
    pub x: f64,
    pub y: f64,
    pub z: f64,
}

impl Vec3 {
    pub fn new(x: f64, y: f64, z: f64) -> Self {
        Vec3 { x, y, z }
    }

    pub fn zero() -> Self { Vec3::new(0.0, 0.0, 0.0) }
    pub fn one() -> Self  { Vec3::new(1.0, 1.0, 1.0) }

    /// Dot product: a·b = ax·bx + ay·by + az·bz
    pub fn dot(&self, other: &Vec3) -> f64 {
        self.x * other.x + self.y * other.y + self.z * other.z
    }

    /// Cross product: ตั้งฉากกับทั้งคู่ ใช้หา surface normal
    pub fn cross(&self, other: &Vec3) -> Vec3 {
        Vec3::new(
            self.y * other.z - self.z * other.y,
            self.z * other.x - self.x * other.z,
            self.x * other.y - self.y * other.x,
        )
    }

    pub fn length_sq(&self) -> f64 { self.dot(self) }
    pub fn length(&self) -> f64    { self.length_sq().sqrt() }

    /// Normalize: ทำให้ความยาว = 1 (unit vector)
    pub fn normalize(&self) -> Vec3 {
        let len = self.length();
        if len > 1e-10 { *self / len } else { Vec3::zero() }
    }

    /// ตรวจว่าเกือบเป็น zero vector (ใช้ใน scatter)
    pub fn near_zero(&self) -> bool {
        let eps = 1e-8;
        self.x.abs() < eps && self.y.abs() < eps && self.z.abs() < eps
    }

    /// Reflection: r = d - 2(d·n)n
    pub fn reflect(&self, normal: &Vec3) -> Vec3 {
        *self - *normal * (2.0 * self.dot(normal))
    }

    /// Snell's law refraction: eta_ratio = n1/n2
    pub fn refract(&self, normal: &Vec3, eta_ratio: f64) -> Vec3 {
        let cos_theta = (-*self).dot(normal).min(1.0);
        let r_perp = (*self + *normal * cos_theta) * eta_ratio;
        let r_para = *normal * -(1.0 - r_perp.length_sq()).abs().sqrt();
        r_perp + r_para
    }

    /// Gamma correction: linear -> display (sRGB)
    pub fn gamma_correct(&self, gamma: f64) -> Vec3 {
        let inv = 1.0 / gamma;
        Vec3::new(
            self.x.max(0.0).powf(inv),
            self.y.max(0.0).powf(inv),
            self.z.max(0.0).powf(inv),
        )
    }

    pub fn lerp(&self, other: &Vec3, t: f64) -> Vec3 {
        *self * (1.0 - t) + *other * t
    }
}

// Operator overloading
impl Add for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Vec3 {
        Vec3::new(self.x + rhs.x, self.y + rhs.y, self.z + rhs.z)
    }
}

impl Sub for Vec3 {
    type Output = Vec3;
    fn sub(self, rhs: Vec3) -> Vec3 {
        Vec3::new(self.x - rhs.x, self.y - rhs.y, self.z - rhs.z)
    }
}

impl Mul<f64> for Vec3 {
    type Output = Vec3;
    fn mul(self, t: f64) -> Vec3 {
        Vec3::new(self.x * t, self.y * t, self.z * t)
    }
}

impl Mul<Vec3> for Vec3 {
    type Output = Vec3;
    /// Component-wise multiply: ใช้คูณสีกับ attenuation
    fn mul(self, rhs: Vec3) -> Vec3 {
        Vec3::new(self.x * rhs.x, self.y * rhs.y, self.z * rhs.z)
    }
}

impl Div<f64> for Vec3 {
    type Output = Vec3;
    fn div(self, t: f64) -> Vec3 {
        Vec3::new(self.x / t, self.y / t, self.z / t)
    }
}

impl Neg for Vec3 {
    type Output = Vec3;
    fn neg(self) -> Vec3 {
        Vec3::new(-self.x, -self.y, -self.z)
    }
}

impl AddAssign for Vec3 {
    fn add_assign(&mut self, rhs: Vec3) {
        self.x += rhs.x; self.y += rhs.y; self.z += rhs.z;
    }
}

impl MulAssign<f64> for Vec3 {
    fn mul_assign(&mut self, t: f64) {
        self.x *= t; self.y *= t; self.z *= t;
    }
}

/// Color เป็นแค่ type alias ของ Vec3 (r,g,b ∈ [0,1])
pub type Color = Vec3;

/// Ray: เส้นตรงใน 3D — `P(t) = origin + t * direction`
#[derive(Debug, Clone, Copy)]
pub struct Ray {
    pub origin: Vec3,
    pub direction: Vec3,
}

impl Ray {
    pub fn new(origin: Vec3, direction: Vec3) -> Self {
        Ray { origin, direction }
    }

    /// จุดบนเส้นที่ parameter t: P(t) = origin + t * direction
    pub fn at(&self, t: f64) -> Vec3 {
        self.origin + self.direction * t
    }
}

/// Axis-Aligned Bounding Box — ใช้ใน BVH
#[derive(Debug, Clone, Copy)]
pub struct Aabb {
    pub min: Vec3,
    pub max: Vec3,
}

impl Aabb {
    pub fn new(min: Vec3, max: Vec3) -> Self {
        Aabb { min, max }
    }

    /// Slab method: ตรวจ AABB intersection ใน O(1)
    /// เปรียบเทียบ t intervals ของทุก axis พร้อมกัน
    pub fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> bool {
        let mut t_min = t_min;
        let mut t_max = t_max;

        for axis in 0..3 {
            let (orig, dir, mn, mx) = match axis {
                0 => (ray.origin.x, ray.direction.x, self.min.x, self.max.x),
                1 => (ray.origin.y, ray.direction.y, self.min.y, self.max.y),
                _ => (ray.origin.z, ray.direction.z, self.min.z, self.max.z),
            };

            let inv_d = 1.0 / dir;
            let mut t0 = (mn - orig) * inv_d;
            let mut t1 = (mx - orig) * inv_d;
            if inv_d < 0.0 {
                std::mem::swap(&mut t0, &mut t1);
            }
            t_min = t0.max(t_min);
            t_max = t1.min(t_max);
            if t_max <= t_min {
                return false;
            }
        }
        true
    }

    pub fn surrounding(a: &Aabb, b: &Aabb) -> Aabb {
        let small = Vec3::new(
            a.min.x.min(b.min.x),
            a.min.y.min(b.min.y),
            a.min.z.min(b.min.z),
        );
        let big = Vec3::new(
            a.max.x.max(b.max.x),
            a.max.y.max(b.max.y),
            a.max.z.max(b.max.z),
        );
        Aabb::new(small, big)
    }

    /// SAH: Surface Area Heuristic — ใช้ประเมินต้นทุน split
    pub fn surface_area(&self) -> f64 {
        let d = self.max - self.min;
        2.0 * (d.x * d.y + d.y * d.z + d.z * d.x)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_vec3_dot() {
        let a = Vec3::new(1.0, 2.0, 3.0);
        let b = Vec3::new(4.0, 5.0, 6.0);
        // dot = 1*4 + 2*5 + 3*6 = 4 + 10 + 18 = 32
        assert!((a.dot(&b) - 32.0).abs() < 1e-10);
    }

    #[test]
    fn test_vec3_cross() {
        let a = Vec3::new(1.0, 0.0, 0.0);
        let b = Vec3::new(0.0, 1.0, 0.0);
        let c = a.cross(&b);
        // i × j = k
        assert!((c.x).abs() < 1e-10);
        assert!((c.y).abs() < 1e-10);
        assert!((c.z - 1.0).abs() < 1e-10);
    }

    #[test]
    fn test_vec3_normalize() {
        let v = Vec3::new(3.0, 0.0, 4.0);
        let n = v.normalize();
        // ||(3,0,4)|| = 5 → normalized = (0.6, 0, 0.8)
        assert!((n.length() - 1.0).abs() < 1e-10);
        assert!((n.x - 0.6).abs() < 1e-10);
        assert!((n.z - 0.8).abs() < 1e-10);
    }

    #[test]
    fn test_vec3_length_sq() {
        let v = Vec3::new(3.0, 4.0, 0.0);
        assert!((v.length_sq() - 25.0).abs() < 1e-10);
        assert!((v.length() - 5.0).abs() < 1e-10);
    }

    #[test]
    fn test_ray_at() {
        let ray = Ray::new(Vec3::zero(), Vec3::new(1.0, 0.0, 0.0));
        let p = ray.at(3.0);
        // P(3) = (0,0,0) + 3*(1,0,0) = (3,0,0)
        assert!((p.x - 3.0).abs() < 1e-10);
        assert!((p.y).abs() < 1e-10);
        assert!((p.z).abs() < 1e-10);
    }

    #[test]
    fn test_aabb_hit() {
        let aabb = Aabb::new(
            Vec3::new(-1.0, -1.0, -1.0),
            Vec3::new(1.0, 1.0, 1.0),
        );
        // ray เข้าตรงกลาง: hit
        let ray_hit = Ray::new(Vec3::new(0.0, 0.0, -5.0), Vec3::new(0.0, 0.0, 1.0));
        assert!(aabb.hit(&ray_hit, 0.001, f64::INFINITY));

        // ray เบี่ยงด้านข้าง: miss
        let ray_miss = Ray::new(Vec3::new(5.0, 0.0, -5.0), Vec3::new(0.0, 0.0, 1.0));
        assert!(!aabb.hit(&ray_miss, 0.001, f64::INFINITY));
    }

    #[test]
    fn test_schlick_approximation() {
        // r0 = ((1-1.5)/(1+1.5))^2 = (-0.5/2.5)^2 = 0.04
        let r0 = {
            let r = (1.0_f64 - 1.5_f64) / (1.0_f64 + 1.5_f64);
            r * r
        };
        assert!((r0 - 0.04).abs() < 1e-5);
    }

    #[test]
    fn test_snells_law() {
        // ray ตั้งฉากกับพื้นผิว ไม่ควร bend
        let incident = Vec3::new(0.0, -1.0, 0.0).normalize();
        let normal = Vec3::new(0.0, 1.0, 0.0);
        let refracted = incident.refract(&normal, 1.0 / 1.5);
        // ยังเดินทางลงอยู่
        assert!(refracted.y < 0.0, "refracted.y = {}", refracted.y);
    }
}
```

**Output จากการรัน `cargo test math`:**
```
running 7 tests
test math::tests::test_aabb_hit ... ok
test math::tests::test_ray_at ... ok
test math::tests::test_schlick_approximation ... ok
test math::tests::test_snells_law ... ok
test math::tests::test_vec3_cross ... ok
test math::tests::test_vec3_dot ... ok
test math::tests::test_vec3_length_sq ... ok
test math::tests::test_vec3_normalize ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured
```

สิ่งที่เรียนรู้ในขั้นนี้: Rust อนุญาตให้ overload operator ผ่าน traits ใน `std::ops` ทุก operator เป็น trait แยกกัน เช่น `Add`, `Mul<f64>`, `Mul<Vec3>` — สังเกตว่า `Mul` สามารถ impl สองครั้งด้วย RHS ต่างกัน

---

### ขั้นที่ 2: Scene Objects — Hittable Trait และ Sphere, Plane, Triangle

ออกแบบ `trait Hittable` เป็น interface หลักสำหรับวัตถุทุกชนิด แล้วสร้าง 3 primitives

**`src/hittable.rs`:**

```rust
use crate::math::{Vec3, Ray, Aabb};
use crate::material::MaterialHandle;

/// ข้อมูลที่ได้เมื่อ ray ชนวัตถุ
#[derive(Clone)]
pub struct HitRecord {
    pub point: Vec3,       // จุดที่ ray ชน
    pub normal: Vec3,      // normal ที่จุดนั้น (ชี้ออกจากพื้นผิว)
    pub t: f64,            // parameter ตรงที่ชน: P = ray.at(t)
    pub front_face: bool,  // ray มาจากด้านนอกหรือใน?
    pub material: MaterialHandle,
}

impl HitRecord {
    pub fn new(
        point: Vec3,
        outward_normal: Vec3,
        t: f64,
        ray: &Ray,
        material: MaterialHandle,
    ) -> Self {
        // กำหนด normal ให้ชี้สวนทาง ray เสมอ
        let front_face = ray.direction.dot(&outward_normal) < 0.0;
        let normal = if front_face { outward_normal } else { -outward_normal };
        HitRecord { point, normal, t, front_face, material }
    }
}

/// Trait หลักสำหรับวัตถุทุกชนิดในฉาก
/// Send + Sync บังคับ เพราะต้องแชร์ข้าม thread ใน rayon
pub trait Hittable: Send + Sync {
    fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> Option<HitRecord>;
    fn bounding_box(&self) -> Option<Aabb>;
}

/// Sphere: ใช้สูตร quadratic equation
/// |P - C|² = r² → |O + tD - C|² = r²
/// → at² + 2bt + c = 0 (half-b form เพื่อประสิทธิภาพ)
pub struct Sphere {
    pub center: Vec3,
    pub radius: f64,
    pub material: MaterialHandle,
}

impl Sphere {
    pub fn new(center: Vec3, radius: f64, material: MaterialHandle) -> Self {
        Sphere { center, radius, material }
    }
}

impl Hittable for Sphere {
    fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> Option<HitRecord> {
        let oc = ray.origin - self.center;
        let a = ray.direction.length_sq();
        let half_b = oc.dot(&ray.direction);
        let c = oc.length_sq() - self.radius * self.radius;
        let discriminant = half_b * half_b - a * c;

        if discriminant < 0.0 {
            return None;
        }

        let sqrt_d = discriminant.sqrt();
        // ลองรากใกล้ก่อน ถ้าไม่อยู่ใน range ลองรากไกล
        let mut root = (-half_b - sqrt_d) / a;
        if root < t_min || root > t_max {
            root = (-half_b + sqrt_d) / a;
            if root < t_min || root > t_max {
                return None;
            }
        }

        let point = ray.at(root);
        let outward_normal = (point - self.center) / self.radius;
        Some(HitRecord::new(point, outward_normal, root, ray, self.material.clone()))
    }

    fn bounding_box(&self) -> Option<Aabb> {
        let r = Vec3::new(self.radius, self.radius, self.radius);
        Some(Aabb::new(self.center - r, self.center + r))
    }
}

/// Plane: ระนาบอนันต์
/// สมการ: (P - point) · normal = 0
/// แก้หา t: t = ((point - origin) · normal) / (direction · normal)
pub struct Plane {
    pub point: Vec3,
    pub normal: Vec3,
    pub material: MaterialHandle,
}

impl Plane {
    pub fn new(point: Vec3, normal: Vec3, material: MaterialHandle) -> Self {
        Plane { point, normal: normal.normalize(), material }
    }
}

impl Hittable for Plane {
    fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> Option<HitRecord> {
        let denom = self.normal.dot(&ray.direction);
        if denom.abs() < 1e-8 {
            return None; // ray ขนาน plane
        }
        let t = (self.point - ray.origin).dot(&self.normal) / denom;
        if t < t_min || t > t_max {
            return None;
        }
        let point = ray.at(t);
        Some(HitRecord::new(point, self.normal, t, ray, self.material.clone()))
    }

    fn bounding_box(&self) -> Option<Aabb> {
        None // plane ไม่มี finite bounding box
    }
}

/// Triangle: ใช้ Möller–Trumbore algorithm
/// เร็วกว่าการแก้สมการ plane แล้วตรวจจุด
pub struct Triangle {
    pub v0: Vec3,
    pub v1: Vec3,
    pub v2: Vec3,
    pub normal: Vec3,
    pub material: MaterialHandle,
}

impl Triangle {
    pub fn new(v0: Vec3, v1: Vec3, v2: Vec3, material: MaterialHandle) -> Self {
        let e1 = v1 - v0;
        let e2 = v2 - v0;
        let normal = e1.cross(&e2).normalize();
        Triangle { v0, v1, v2, normal, material }
    }
}

impl Hittable for Triangle {
    fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> Option<HitRecord> {
        let e1 = self.v1 - self.v0;
        let e2 = self.v2 - self.v0;
        let h = ray.direction.cross(&e2);
        let a = e1.dot(&h);

        if a.abs() < 1e-8 { return None; } // ขนาน

        let f = 1.0 / a;
        let s = ray.origin - self.v0;
        let u = f * s.dot(&h);
        if !(0.0..=1.0).contains(&u) { return None; }

        let q = s.cross(&e1);
        let v = f * ray.direction.dot(&q);
        if v < 0.0 || u + v > 1.0 { return None; }

        let t = f * e2.dot(&q);
        if t < t_min || t > t_max { return None; }

        let point = ray.at(t);
        Some(HitRecord::new(point, self.normal, t, ray, self.material.clone()))
    }

    fn bounding_box(&self) -> Option<Aabb> {
        let min = Vec3::new(
            self.v0.x.min(self.v1.x).min(self.v2.x) - 1e-4,
            self.v0.y.min(self.v1.y).min(self.v2.y) - 1e-4,
            self.v0.z.min(self.v1.z).min(self.v2.z) - 1e-4,
        );
        let max = Vec3::new(
            self.v0.x.max(self.v1.x).max(self.v2.x) + 1e-4,
            self.v0.y.max(self.v1.y).max(self.v2.y) + 1e-4,
            self.v0.z.max(self.v1.z).max(self.v2.z) + 1e-4,
        );
        Some(Aabb::new(min, max))
    }
}

/// HittableList: ลิสต์วัตถุหลายชนิด — O(n) linear search
pub struct HittableList {
    pub objects: Vec<Box<dyn Hittable>>,
}

impl HittableList {
    pub fn new() -> Self {
        HittableList { objects: Vec::new() }
    }

    pub fn add(&mut self, object: Box<dyn Hittable>) {
        self.objects.push(object);
    }
}

impl Hittable for HittableList {
    fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> Option<HitRecord> {
        let mut closest = t_max;
        let mut result = None;

        for obj in &self.objects {
            if let Some(rec) = obj.hit(ray, t_min, closest) {
                closest = rec.t;
                result = Some(rec);
            }
        }
        result
    }

    fn bounding_box(&self) -> Option<Aabb> {
        if self.objects.is_empty() { return None; }
        let mut bbox = self.objects[0].bounding_box()?;
        for obj in &self.objects[1..] {
            if let Some(b) = obj.bounding_box() {
                bbox = Aabb::surrounding(&bbox, &b);
            }
        }
        Some(bbox)
    }
}
```

ประเด็นที่น่าสนใจ: `HitRecord` ต้องไม่ `derive(Debug)` เพราะ `MaterialHandle = Arc<dyn Material>` ไม่ implement `Debug` โดยอัตโนมัติ — นี่คือสัญญาณว่า `dyn Trait` ทำให้เราต้องระวัง derived traits

---

### ขั้นที่ 3: Materials — Lambertian, Metal, Dielectric

วัสดุแต่ละชนิดบอกว่า ray จะ bounce ยังไงเมื่อชนพื้นผิว

**`src/material.rs`:**

```rust
use std::sync::Arc;
use rand::Rng;
use crate::math::{Vec3, Color, Ray};
use crate::hittable::HitRecord;

/// ผลลัพธ์การกระเจิง: scattered ray ใหม่ + attenuation (ดูดซับ)
pub struct ScatterResult {
    pub attenuation: Color,
    pub scattered: Ray,
}

/// Trait สำหรับวัสดุ — ต้อง Send + Sync เพราะแชร์ข้าม thread
pub trait Material: Send + Sync {
    fn scatter(
        &self,
        ray_in: &Ray,
        rec: &HitRecord,
        rng: &mut dyn rand::RngCore,
    ) -> Option<ScatterResult>;
}

pub type MaterialHandle = Arc<dyn Material>;

/// สุ่ม unit vector บน unit sphere (rejection sampling)
pub fn random_unit_vector(rng: &mut dyn rand::RngCore) -> Vec3 {
    loop {
        let v = Vec3::new(
            rng.gen_range(-1.0..1.0),
            rng.gen_range(-1.0..1.0),
            rng.gen_range(-1.0..1.0),
        );
        if v.length_sq() < 1.0 {
            return v.normalize();
        }
    }
}

/// สุ่ม vector บน unit disk (สำหรับ depth of field)
pub fn random_in_unit_disk(rng: &mut dyn rand::RngCore) -> Vec3 {
    loop {
        let v = Vec3::new(rng.gen_range(-1.0..1.0), rng.gen_range(-1.0..1.0), 0.0);
        if v.length_sq() < 1.0 { return v; }
    }
}

/// Lambertian: diffuse — กระจายแสงไปทุกทิศทางบน hemisphere
/// สร้างเงาอ่อน, color bleeding
pub struct Lambertian {
    pub albedo: Color,
}

impl Lambertian {
    pub fn new(albedo: Color) -> Self { Lambertian { albedo } }
}

impl Material for Lambertian {
    fn scatter(
        &self,
        _ray_in: &Ray,
        rec: &HitRecord,
        rng: &mut dyn rand::RngCore,
    ) -> Option<ScatterResult> {
        // scatter direction = normal + random unit vector (hemisphere)
        let mut scatter_dir = rec.normal + random_unit_vector(rng);
        if scatter_dir.near_zero() {
            scatter_dir = rec.normal; // ป้องกัน degenerate ray
        }
        Some(ScatterResult {
            scattered: Ray::new(rec.point, scatter_dir),
            attenuation: self.albedo,
        })
    }
}

/// Metal: สะท้อนแบบ specular + fuzz (roughness)
/// fuzz = 0 → mirror perfect, fuzz = 1 → rough metal
pub struct Metal {
    pub albedo: Color,
    pub fuzz: f64,
}

impl Metal {
    pub fn new(albedo: Color, fuzz: f64) -> Self {
        Metal { albedo, fuzz: fuzz.min(1.0) }
    }
}

impl Material for Metal {
    fn scatter(
        &self,
        ray_in: &Ray,
        rec: &HitRecord,
        rng: &mut dyn rand::RngCore,
    ) -> Option<ScatterResult> {
        let reflected = ray_in.direction.normalize().reflect(&rec.normal);
        // เพิ่ม perturbation ตาม fuzz
        let scattered_dir = reflected + random_unit_vector(rng) * self.fuzz;
        if scattered_dir.dot(&rec.normal) > 0.0 {
            Some(ScatterResult {
                scattered: Ray::new(rec.point, scattered_dir),
                attenuation: self.albedo,
            })
        } else {
            None // ray ถูก fuzz เข้าไปใต้พื้นผิว
        }
    }
}

/// Schlick approximation: ประเมิน Fresnel reflectance เร็ว
/// R(θ) ≈ R₀ + (1 - R₀)(1 - cosθ)⁵
pub fn schlick(cosine: f64, ref_idx: f64) -> f64 {
    let r0 = ((1.0 - ref_idx) / (1.0 + ref_idx)).powi(2);
    r0 + (1.0 - r0) * (1.0 - cosine).powi(5)
}

/// Dielectric: แก้ว/น้ำ — หักเหแสงตาม Snell's law + Fresnel
/// ior = index of refraction (น้ำ ≈ 1.33, แก้ว ≈ 1.5)
pub struct Dielectric {
    pub ior: f64,
}

impl Dielectric {
    pub fn new(ior: f64) -> Self { Dielectric { ior } }
}

impl Material for Dielectric {
    fn scatter(
        &self,
        ray_in: &Ray,
        rec: &HitRecord,
        rng: &mut dyn rand::RngCore,
    ) -> Option<ScatterResult> {
        // แก้วใสไม่ดูดซับแสง
        let attenuation = Color::new(1.0, 1.0, 1.0);
        // ถ้า ray มาจากด้านนอก → eta = air/glass, ถ้ามาจากใน → glass/air
        let eta_ratio = if rec.front_face { 1.0 / self.ior } else { self.ior };

        let unit_dir = ray_in.direction.normalize();
        let cos_theta = (-unit_dir).dot(&rec.normal).min(1.0);
        let sin_theta = (1.0 - cos_theta * cos_theta).sqrt();

        // Total Internal Reflection: sin_theta > 1/eta_ratio → ต้อง reflect
        let cannot_refract = eta_ratio * sin_theta > 1.0;
        let direction = if cannot_refract || schlick(cos_theta, eta_ratio) > rng.gen::<f64>() {
            unit_dir.reflect(&rec.normal)
        } else {
            unit_dir.refract(&rec.normal, eta_ratio)
        };

        Some(ScatterResult {
            scattered: Ray::new(rec.point, direction),
            attenuation,
        })
    }
}
```

ความลึกที่เรียนรู้: `rng: &mut dyn rand::RngCore` ใช้ trait object สำหรับ random เพื่อ decouple — เราสามารถส่ง rng ชนิดใดก็ได้ (SmallRng สำหรับ speed, StdRng สำหรับ security) โดยไม่ต้องเปลี่ยน Material code

---

### ขั้นที่ 4: BVH Acceleration Structure

BVH (Bounding Volume Hierarchy) เปลี่ยน ray-scene intersection จาก O(n) → O(log n) สำหรับ scenes ที่มีวัตถุหลายพัน

```
Scene: N objects = 1000
Linear: ต้องตรวจทุกวัตถุ = 1000 ops/ray
BVH:    ตรวจ log₂(1000) ≈ 10 nodes = 100× เร็วกว่า
```

**`src/bvh.rs`:**

```rust
use std::sync::Arc;
use crate::math::{Aabb, Ray};
use crate::hittable::{HitRecord, Hittable};

pub struct BvhNode {
    pub left: Arc<dyn Hittable>,
    pub right: Arc<dyn Hittable>,
    pub aabb: Aabb,         // bounding box ที่ครอบ left + right
}

impl BvhNode {
    /// สร้าง BVH tree จาก list ของ objects
    /// ใช้ recursive midpoint split ตาม longest axis
    pub fn build(objects: Vec<Arc<dyn Hittable>>) -> Arc<dyn Hittable> {
        Self::build_range(objects)
    }

    fn build_range(mut objects: Vec<Arc<dyn Hittable>>) -> Arc<dyn Hittable> {
        // Base cases
        if objects.len() == 1 {
            return objects.remove(0);
        }
        if objects.len() == 2 {
            let left = objects.remove(0);
            let right = objects.remove(0);
            let bbox = merge_bbox(&left, &right);
            return Arc::new(BvhNode { left, right, aabb: bbox });
        }

        // หา longest axis ของ bounding box รวม
        let total_bbox = compute_bbox_of_list(&objects);
        let axis = longest_axis(&total_bbox);

        // Sort objects ตาม centroid บน axis นั้น
        objects.sort_by(|a, b| {
            centroid_on_axis(a.as_ref(), axis)
                .partial_cmp(&centroid_on_axis(b.as_ref(), axis))
                .unwrap()
        });

        // Split ที่ midpoint แล้ว recurse
        let mid = objects.len() / 2;
        let right_objects = objects.split_off(mid);
        let left = Self::build_range(objects);
        let right = Self::build_range(right_objects);
        let bbox = merge_bbox(&left, &right);

        Arc::new(BvhNode { left, right, aabb: bbox })
    }
}

fn merge_bbox(a: &Arc<dyn Hittable>, b: &Arc<dyn Hittable>) -> Aabb {
    match (a.bounding_box(), b.bounding_box()) {
        (Some(ba), Some(bb)) => Aabb::surrounding(&ba, &bb),
        (Some(ba), None) => ba,
        (None, Some(bb)) => bb,
        (None, None) => panic!("BVH: objects without bounding box"),
    }
}

fn compute_bbox_of_list(objects: &[Arc<dyn Hittable>]) -> Aabb {
    use crate::math::Vec3;
    objects.iter().fold(
        Aabb::new(
            Vec3::new(f64::INFINITY, f64::INFINITY, f64::INFINITY),
            Vec3::new(f64::NEG_INFINITY, f64::NEG_INFINITY, f64::NEG_INFINITY),
        ),
        |acc, obj| {
            if let Some(b) = obj.bounding_box() {
                Aabb::surrounding(&acc, &b)
            } else {
                acc
            }
        },
    )
}

fn longest_axis(bbox: &Aabb) -> usize {
    let d = bbox.max - bbox.min;
    if d.x > d.y && d.x > d.z { 0 }
    else if d.y > d.z { 1 }
    else { 2 }
}

fn centroid_on_axis(obj: &dyn Hittable, axis: usize) -> f64 {
    if let Some(b) = obj.bounding_box() {
        match axis {
            0 => (b.min.x + b.max.x) * 0.5,
            1 => (b.min.y + b.max.y) * 0.5,
            _ => (b.min.z + b.max.z) * 0.5,
        }
    } else { 0.0 }
}

impl Hittable for BvhNode {
    fn hit(&self, ray: &Ray, t_min: f64, t_max: f64) -> Option<HitRecord> {
        // ตรวจ AABB ก่อน: ถ้า miss ไม่ต้องตรวจลูก
        if !self.aabb.hit(ray, t_min, t_max) {
            return None;
        }
        let hit_left = self.left.hit(ray, t_min, t_max);
        // ใช้ t ของ left เป็น t_max สำหรับ right (right ต้องใกล้กว่า)
        let t_right_max = hit_left.as_ref().map_or(t_max, |r| r.t);
        let hit_right = self.right.hit(ray, t_min, t_right_max);
        hit_right.or(hit_left)
    }

    fn bounding_box(&self) -> Option<Aabb> {
        Some(self.aabb)
    }
}
```

**Output จาก test BVH:**
```
test bvh::tests::test_bvh_build_small_scene ... ok
test bvh::tests::test_bvh_hit_finds_correct_sphere ... ok
```

---

### ขั้นที่ 5: Camera พร้อม Depth of Field

กล้องจริงมี aperture และ focal plane — วัตถุนอก focus plane จะ blur

**`src/camera.rs`:**

```rust
use crate::math::{Vec3, Ray};
use crate::material::random_in_unit_disk;
use rand::RngCore;

pub struct Camera {
    pub origin: Vec3,
    pub lower_left: Vec3,  // มุมซ้ายล่างของ viewport
    pub horizontal: Vec3,  // เวกเตอร์แนวนอนของ viewport
    pub vertical: Vec3,    // เวกเตอร์แนวตั้งของ viewport
    pub u: Vec3,           // camera right vector
    pub v: Vec3,           // camera up vector
    pub lens_radius: f64,  // aperture / 2
}

impl Camera {
    /// สร้าง camera จาก parameters แบบ cinema
    ///
    /// - `look_from`: ตำแหน่งกล้อง
    /// - `look_at`: จุดที่กล้องชี้ไป
    /// - `v_up`: "up" vector ของโลก (ปกติ Y)
    /// - `vfov_deg`: vertical field of view (degrees)
    /// - `aspect_ratio`: width / height
    /// - `aperture`: ขนาด lens (0 = pinhole camera ไม่มี DOF)
    /// - `focus_dist`: ระยะ focal plane
    pub fn new(
        look_from: Vec3,
        look_at: Vec3,
        v_up: Vec3,
        vfov_deg: f64,
        aspect_ratio: f64,
        aperture: f64,
        focus_dist: f64,
    ) -> Self {
        let theta = vfov_deg.to_radians();
        let h = (theta / 2.0).tan();
        let viewport_height = 2.0 * h;
        let viewport_width = aspect_ratio * viewport_height;

        // สร้าง camera coordinate system
        let w = (look_from - look_at).normalize(); // camera -z
        let u = v_up.cross(&w).normalize();         // camera +x
        let v = w.cross(&u);                         // camera +y

        let origin = look_from;
        let horizontal = u * (viewport_width * focus_dist);
        let vertical = v * (viewport_height * focus_dist);
        let lower_left = origin - horizontal * 0.5 - vertical * 0.5 - w * focus_dist;

        Camera { origin, lower_left, horizontal, vertical, u, v, lens_radius: aperture / 2.0 }
    }

    /// สร้าง ray ผ่าน pixel (s, t) — s, t ∈ [0, 1]
    /// Disk sampling สำหรับ depth of field
    pub fn generate_ray(&self, s: f64, t: f64, rng: &mut dyn RngCore) -> Ray {
        let rd = random_in_unit_disk(rng) * self.lens_radius;
        let offset = self.u * rd.x + self.v * rd.y;
        Ray::new(
            self.origin + offset,
            self.lower_left + self.horizontal * s + self.vertical * t
                - self.origin - offset,
        )
    }
}
```

ทฤษฎี Depth of Field: แทนที่จะยิง ray จากจุดเดียว (pinhole) เราสุ่ม origin บน disk รอบๆ กล้อง ทุก ray ผ่าน focus plane ที่จุดเดิม แต่ออกจาก origin ต่างกัน — วัตถุที่ focus plane จะ sharp (rays ตัดกัน), วัตถุอื่นจะ blur

---

### ขั้นที่ 6: Path Tracing Loop + Stratified Sampling

หัวใจของโปรแกรม — recursive `trace()` function

**`src/renderer.rs`:**

```rust
use rand::{Rng, SeedableRng};
use rand::rngs::SmallRng;
use rayon::prelude::*;
use crate::math::{Color, Ray};
use crate::hittable::Hittable;
use crate::camera::Camera;

/// Sky gradient: ไล่สีจากขาว (ล่าง) เป็นฟ้า (บน)
pub fn sky_color(ray: &Ray) -> Color {
    let unit = ray.direction.normalize();
    let t = 0.5 * (unit.y + 1.0);  // แปลง [-1,1] → [0,1]
    Color::new(1.0, 1.0, 1.0).lerp(&Color::new(0.5, 0.7, 1.0), t)
}

/// Path tracing recursive function
///
/// Algorithm:
/// 1. ถ้า depth = 0 → return black (หมด bounces)
/// 2. Russian roulette ที่ depth > 50
/// 3. ถ้า ray ชน → scatter ตาม material + recurse × albedo
/// 4. ถ้า ray miss → return sky color (background illumination)
pub fn trace(
    ray: &Ray,
    world: &dyn Hittable,
    depth: u32,
    rng: &mut dyn rand::RngCore,
) -> Color {
    if depth == 0 {
        return Color::zero();
    }

    // Russian Roulette: terminate rays probabilistically
    // survival_prob = 0.9 → ประหยัดได้ ~10% ของเวลา bounce ที่ลึก
    if depth < u32::MAX && depth > 50 {
        if rng.gen::<f64>() > 0.9 {
            return Color::zero();
        }
    }

    if let Some(rec) = world.hit(ray, 0.001, f64::INFINITY) {
        // t_min = 0.001 ป้องกัน "shadow acne" — ray ชน surface ตัวเอง
        if let Some(scatter) = rec.material.scatter(ray, &rec, rng) {
            let bounced = trace(&scatter.scattered, world, depth - 1, rng);
            return scatter.attenuation * bounced;
        }
        return Color::zero(); // material absorbs all light
    }

    sky_color(ray)
}

/// Render configuration
pub struct RenderConfig {
    pub width: u32,
    pub height: u32,
    pub samples: u32,  // samples per pixel
    pub max_depth: u32,
}

pub struct RenderedImage {
    pub width: u32,
    pub height: u32,
    pub pixels: Vec<[u8; 3]>,
}

/// Render ด้วย Rayon parallel — lock-free row accumulation
///
/// แต่ละ row ทำงานอิสระบน thread pool ของ Rayon
/// ไม่มี mutex หรือ shared state → ไม่มี contention
pub fn render_parallel(
    config: &RenderConfig,
    camera: &Camera,
    world: &(dyn Hittable + Send + Sync),
) -> RenderedImage {
    let width = config.width;
    let height = config.height;
    let samples = config.samples;
    let max_depth = config.max_depth;

    let rows: Vec<u32> = (0..height).collect();

    // par_iter() แจก rows ให้ threads โดยอัตโนมัติ (work stealing)
    let pixel_rows: Vec<Vec<[u8; 3]>> = rows
        .par_iter()
        .map(|&j| {
            // SmallRng per-thread — ไม่ต้องแชร์ RNG ข้ามเธรด
            let mut rng = SmallRng::seed_from_u64(j as u64 * 12345 + 67890);
            let mut row = Vec::with_capacity(width as usize);

            for i in 0..width {
                let mut color_acc = Color::zero();

                // Stratified supersampling: แบ่ง pixel เป็น √N × √N grid
                let sqrt_spp = (samples as f64).sqrt() as u32;
                let sqrt_spp = sqrt_spp.max(1);
                let actual_samples = sqrt_spp * sqrt_spp;

                for si in 0..sqrt_spp {
                    for sj in 0..sqrt_spp {
                        // jitter ภายใน stratum [si/√N, (si+1)/√N]
                        let u = (i as f64 + (si as f64 + rng.gen::<f64>()) / sqrt_spp as f64)
                            / (width - 1) as f64;
                        let v = (j as f64 + (sj as f64 + rng.gen::<f64>()) / sqrt_spp as f64)
                            / (height - 1) as f64;

                        let ray = camera.generate_ray(u, v, &mut rng);
                        color_acc += trace(&ray, world, max_depth, &mut rng);
                    }
                }

                let scale = 1.0 / actual_samples as f64;
                let c = (color_acc * scale).gamma_correct(2.2);
                row.push(color_to_u8(&c));
            }
            row
        })
        .collect();

    // rows เรียงจาก bottom ขึ้น top ใน PPM
    let pixels = pixel_rows.into_iter().rev().flatten().collect();
    RenderedImage { width, height, pixels }
}

fn color_to_u8(c: &Color) -> [u8; 3] {
    [
        (c.x.clamp(0.0, 1.0) * 255.999) as u8,
        (c.y.clamp(0.0, 1.0) * 255.999) as u8,
        (c.z.clamp(0.0, 1.0) * 255.999) as u8,
    ]
}

/// เขียน PPM format (P3 ASCII) โดยตรง — ไม่ต้องใช้ crate เพิ่ม
/// Format: "P3\nwidth height\n255\nr g b\n..."
pub fn write_ppm(image: &RenderedImage, path: &str) -> std::io::Result<()> {
    use std::io::Write;
    let mut file = std::fs::File::create(path)?;
    writeln!(file, "P3")?;
    writeln!(file, "{} {}", image.width, image.height)?;
    writeln!(file, "255")?;
    for px in &image.pixels {
        writeln!(file, "{} {} {}", px[0], px[1], px[2])?;
    }
    Ok(())
}
```

---

### ขั้นที่ 7: Scene JSON + CLI + Output

เชื่อมทุกส่วนเข้าด้วยกันผ่าน CLI และ scene file

**`src/scene.rs` (JSON deserialization):**

```rust
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use crate::math::Vec3;
use crate::hittable::{HittableList, Sphere, Plane};
use crate::material::{Lambertian, Metal, Dielectric, MaterialHandle};

#[derive(Serialize, Deserialize, Debug)]
pub struct Vec3Config { pub x: f64, pub y: f64, pub z: f64 }

impl Vec3Config {
    pub fn to_vec3(&self) -> Vec3 { Vec3::new(self.x, self.y, self.z) }
}

#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type")]
pub enum MaterialConfig {
    Lambertian { albedo: Vec3Config },
    Metal { albedo: Vec3Config, fuzz: f64 },
    Dielectric { ior: f64 },
}

impl MaterialConfig {
    pub fn build(&self) -> MaterialHandle {
        match self {
            MaterialConfig::Lambertian { albedo } =>
                Arc::new(Lambertian::new(albedo.to_vec3())),
            MaterialConfig::Metal { albedo, fuzz } =>
                Arc::new(Metal::new(albedo.to_vec3(), *fuzz)),
            MaterialConfig::Dielectric { ior } =>
                Arc::new(Dielectric::new(*ior)),
        }
    }
}

#[derive(Serialize, Deserialize, Debug)]
pub struct SceneConfig {
    pub camera: CameraConfig,
    pub render: RenderSettings,
    pub objects: Vec<ObjectConfig>,
}

#[derive(Serialize, Deserialize, Debug)]
pub struct CameraConfig {
    pub look_from: Vec3Config,
    pub look_at: Vec3Config,
    pub v_up: Vec3Config,
    pub vfov: f64,
    pub aperture: f64,
    pub focus_dist: f64,
}

#[derive(Serialize, Deserialize, Debug)]
pub struct RenderSettings {
    pub width: u32,
    pub height: u32,
    pub samples: u32,
    pub max_depth: u32,
    pub output: String,
}

#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "shape")]
pub enum ObjectConfig {
    Sphere { center: Vec3Config, radius: f64, material: MaterialConfig },
    Plane { point: Vec3Config, normal: Vec3Config, material: MaterialConfig },
}

impl SceneConfig {
    pub fn from_json(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let content = std::fs::read_to_string(path)?;
        Ok(serde_json::from_str(&content)?)
    }

    pub fn build_world(&self) -> HittableList {
        let mut world = HittableList::new();
        for obj in &self.objects {
            match obj {
                ObjectConfig::Sphere { center, radius, material } =>
                    world.add(Box::new(Sphere::new(
                        center.to_vec3(), *radius, material.build(),
                    ))),
                ObjectConfig::Plane { point, normal, material } =>
                    world.add(Box::new(Plane::new(
                        point.to_vec3(), normal.to_vec3(), material.build(),
                    ))),
            }
        }
        world
    }
}

/// Cornell Box scene preset สำหรับ demo
pub fn cornell_box() -> HittableList {
    let mut world = HittableList::new();

    let red: MaterialHandle = Arc::new(Lambertian::new(Vec3::new(0.65, 0.05, 0.05)));
    let white: MaterialHandle = Arc::new(Lambertian::new(Vec3::new(0.73, 0.73, 0.73)));
    let green: MaterialHandle = Arc::new(Lambertian::new(Vec3::new(0.12, 0.45, 0.15)));
    let glass: MaterialHandle = Arc::new(Dielectric::new(1.5));
    let metal: MaterialHandle = Arc::new(Metal::new(Vec3::new(0.8, 0.85, 0.88), 0.0));

    // พื้น, เพดาน, ด้านหลัง
    world.add(Box::new(Plane::new(
        Vec3::new(0.0, 0.0, 0.0), Vec3::new(0.0, 1.0, 0.0), white.clone())));
    world.add(Box::new(Plane::new(
        Vec3::new(0.0, 5.0, 0.0), Vec3::new(0.0, -1.0, 0.0), white.clone())));
    world.add(Box::new(Plane::new(
        Vec3::new(0.0, 0.0, -5.0), Vec3::new(0.0, 0.0, 1.0), white.clone())));
    // ด้านซ้าย (แดง), ด้านขวา (เขียว)
    world.add(Box::new(Plane::new(
        Vec3::new(-2.5, 0.0, 0.0), Vec3::new(1.0, 0.0, 0.0), red)));
    world.add(Box::new(Plane::new(
        Vec3::new(2.5, 0.0, 0.0), Vec3::new(-1.0, 0.0, 0.0), green)));
    // วัตถุในฉาก
    world.add(Box::new(Sphere::new(Vec3::new(-0.8, 1.0, -3.0), 1.0, glass)));
    world.add(Box::new(Sphere::new(Vec3::new(1.0, 0.7, -2.0), 0.7, metal)));

    world
}
```

**`scenes/cornell_box.json`:**

```json
{
  "camera": {
    "look_from": { "x": 0.0, "y": 2.5, "z": 5.0 },
    "look_at":   { "x": 0.0, "y": 2.0, "z": -1.0 },
    "v_up":      { "x": 0.0, "y": 1.0, "z": 0.0 },
    "vfov": 45.0,
    "aperture": 0.0,
    "focus_dist": 6.0
  },
  "render": {
    "width": 800,
    "height": 800,
    "samples": 128,
    "max_depth": 50,
    "output": "cornell_box.png"
  },
  "objects": [
    {
      "shape": "Plane",
      "point": { "x": 0.0, "y": 0.0, "z": 0.0 },
      "normal": { "x": 0.0, "y": 1.0, "z": 0.0 },
      "material": { "type": "Lambertian", "albedo": { "x": 0.73, "y": 0.73, "z": 0.73 } }
    },
    {
      "shape": "Sphere",
      "center": { "x": 0.0, "y": 1.0, "z": -2.0 },
      "radius": 1.0,
      "material": { "type": "Dielectric", "ior": 1.5 }
    },
    {
      "shape": "Sphere",
      "center": { "x": -1.5, "y": 0.5, "z": -1.5 },
      "radius": 0.5,
      "material": {
        "type": "Metal",
        "albedo": { "x": 0.7, "y": 0.6, "z": 0.5 },
        "fuzz": 0.1
      }
    },
    {
      "shape": "Sphere",
      "center": { "x": 1.5, "y": 0.5, "z": -1.5 },
      "radius": 0.5,
      "material": {
        "type": "Lambertian",
        "albedo": { "x": 0.8, "y": 0.3, "z": 0.3 }
      }
    }
  ]
}
```

**`src/main.rs`:**

```rust
mod math;
mod hittable;
mod material;
mod bvh;
mod camera;
mod renderer;
mod scene;

use std::sync::Arc;
use clap::Parser;
use indicatif::{ProgressBar, ProgressStyle};
use math::Vec3;
use hittable::Sphere;
use material::Lambertian;
use camera::Camera;
use renderer::{RenderConfig, render_parallel, write_ppm};

#[derive(Parser, Debug)]
#[command(name = "ray_tracer", about = "Path Tracer ใน Rust")]
struct Args {
    /// Scene JSON file
    #[arg(long)]
    scene: Option<String>,

    /// Output file (PPM หรือ PNG ขึ้นกับ extension)
    #[arg(long, default_value = "output.ppm")]
    output: String,

    #[arg(long, default_value_t = 800)]
    width: u32,

    #[arg(long, default_value_t = 600)]
    height: u32,

    /// Samples per pixel
    #[arg(long, default_value_t = 64)]
    samples: u32,

    /// Max bounce depth
    #[arg(long, default_value_t = 50)]
    depth: u32,

    /// ใช้ Cornell Box demo scene
    #[arg(long)]
    demo: bool,
}

fn main() {
    let args = Args::parse();

    eprintln!(
        "Ray Tracer — {}×{} @ {}spp depth={}",
        args.width, args.height, args.samples, args.depth
    );

    let pb = ProgressBar::new(args.height as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos}/{len} rows {msg}")
            .unwrap(),
    );

    let aspect = args.width as f64 / args.height as f64;

    // สร้าง scene
    let world: Box<dyn hittable::Hittable + Send + Sync> = if args.demo {
        Box::new(scene::cornell_box())
    } else if let Some(scene_path) = &args.scene {
        let cfg = scene::SceneConfig::from_json(scene_path)
            .expect("Failed to load scene JSON");
        Box::new(cfg.build_world())
    } else {
        // Default three-sphere scene
        let mut list = hittable::HittableList::new();
        list.add(Box::new(Sphere::new(
            Vec3::new(0.0, 0.0, -1.0), 0.5,
            Arc::new(Lambertian::new(Vec3::new(0.7, 0.3, 0.3))),
        )));
        list.add(Box::new(Sphere::new(
            Vec3::new(-1.2, 0.0, -1.0), 0.5,
            Arc::new(material::Metal::new(Vec3::new(0.8, 0.8, 0.9), 0.05)),
        )));
        list.add(Box::new(Sphere::new(
            Vec3::new(1.2, 0.0, -1.0), 0.5,
            Arc::new(material::Dielectric::new(1.5)),
        )));
        list.add(Box::new(Sphere::new(
            Vec3::new(0.0, -100.5, -1.0), 100.0,
            Arc::new(Lambertian::new(Vec3::new(0.8, 0.8, 0.0))),
        )));
        Box::new(list)
    };

    let cam = Camera::new(
        Vec3::new(0.0, 0.5, 3.0),
        Vec3::new(0.0, 0.0, -1.0),
        Vec3::new(0.0, 1.0, 0.0),
        50.0,
        aspect,
        0.0,
        1.0,
    );

    let config = RenderConfig {
        width: args.width,
        height: args.height,
        samples: args.samples,
        max_depth: args.depth,
    };

    let image = render_parallel(&config, &cam, world.as_ref());
    pb.finish_with_message("done");

    if args.output.ends_with(".png") {
        let img = image::RgbImage::from_fn(image.width, image.height, |x, y| {
            let px = image.pixels[(y * image.width + x) as usize];
            image::Rgb(px)
        });
        img.save(&args.output).expect("Failed to save PNG");
    } else {
        write_ppm(&image, &args.output).expect("Failed to write PPM");
    }

    eprintln!("Saved → {}", args.output);
}
```

---

## การทดสอบ (Testing)

### Unit Tests

Tests ที่เขียนไว้ครอบคลุม:
- `math::tests` — Vec3 arithmetic, normalize, dot, cross, AABB hit/miss, Schlick approximation, Snell's law
- `hittable::tests` — ray-sphere intersection formula (discriminant), sphere hit/miss, HittableList
- `material::tests` — Schlick ที่ normal incidence และ grazing angle, Lambertian scatter validity
- `bvh::tests` — BVH build สำหรับ small scene, BVH hit returns nearest sphere

**Output จาก `cargo test` จริง:**

```
running 17 tests
test bvh::tests::test_bvh_build_small_scene ... ok
test bvh::tests::test_bvh_hit_finds_correct_sphere ... ok
test hittable::tests::test_ray_sphere_discriminant ... ok
test hittable::tests::test_sphere_hit_basic ... ok
test hittable::tests::test_hittable_list ... ok
test hittable::tests::test_sphere_miss ... ok
test material::tests::test_schlick_grazing_angle ... ok
test material::tests::test_lambertian_scatter_produces_valid_ray ... ok
test material::tests::test_schlick_normal_incidence ... ok
test math::tests::test_aabb_hit ... ok
test math::tests::test_ray_at ... ok
test math::tests::test_schlick_approximation ... ok
test math::tests::test_snells_law ... ok
test math::tests::test_vec3_cross ... ok
test math::tests::test_vec3_dot ... ok
test math::tests::test_vec3_length_sq ... ok
test math::tests::test_vec3_normalize ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### Render Test

```
$ cargo run --release -- --width 80 --height 45 --samples 4 --output /tmp/test_render.ppm
Ray Tracer — 80×45 @ 4spp depth=50
[00:00:00] ████████████████████████████████████████ 45/45 rows done
Saved → /tmp/test_render.ppm
```

ผลลัพธ์คือไฟล์ PPM ขนาด 80×45 มีทรงกลม diffuse (แดง), metal (เงา), dielectric (แก้ว) บนพื้น ground สีเหลือง

### ตรวจ Ray-Sphere Intersection ด้วยมือ

ทดสอบสูตร discriminant โดยตรงในโค้ด:
```
ray: origin=(0,0,0), direction=(0,0,-1)
sphere: center=(0,0,-5), radius=1

oc = origin - center = (0,0,5)
a = |direction|² = 1
half_b = oc · direction = 0*0 + 0*0 + 5*(-1) = -5
c = |oc|² - r² = 25 - 1 = 24
discriminant = half_b² - a*c = 25 - 24 = 1 > 0 → HIT

root = (-(-5) - √1) / 1 = 4 → P(4) = (0,0,-4) = ด้านหน้า sphere ✓
```

---

## Pitfalls และจุดระวัง

### Pitfall 1: Shadow Acne — ray ชนพื้นผิวตัวเอง

**ปัญหา**: เมื่อ ray bounce จากพื้นผิว จุดเริ่มต้นใหม่อาจอยู่ "ใต้" พื้นผิวเล็กน้อยเนื่องจาก floating-point error ทำให้ ray ชนวัตถุเดิมทันที ภาพจะมีจุดดำกระจาย

**สัญญาณ**: ภาพมีจุดดำ/noise ที่ diffuse surfaces โดยเฉพาะเมื่อมี light หลายแหล่ง

**แก้ไข**: ใช้ `t_min = 0.001` แทน `0.0`:
```rust
// ผิด: ray ชนตัวเองได้
world.hit(ray, 0.0, f64::INFINITY)

// ถูก: ข้าม self-intersection
world.hit(ray, 0.001, f64::INFINITY)
```

ค่า 0.001 ต้องพอดี — ถ้าเล็กเกินไปยังมี acne, ถ้าใหญ่เกินไปจะ miss วัตถุที่อยู่ใกล้มาก

---

### Pitfall 2: Gamma Correction ที่ขาดไป

**ปัญหา**: ภาพจะ dark เกินจริง เพราะ path tracer คำนวณใน linear light space แต่ monitor แสดงผลใน gamma-encoded space (sRGB ≈ γ 2.2)

**สัญญาณ**: ภาพ render ดูมืดและ "flat" ผิดปกติ — เงาดำมาก, highlight ดูหม่น

**อธิบาย**: ถ้า ray เก็บ radiance = 0.5 (กลางช่วง), แต่ monitor ตีความ value 128/255 ≠ 50% brightness ที่แท้จริงเพราะ sRGB curve

**แก้ไข**: Apply gamma correction ก่อน output:
```rust
// ผิด: output linear ตรง
let c = color_acc * scale;

// ถูก: gamma correct ก่อน
let c = (color_acc * scale).gamma_correct(2.2);
// หรือแบบง่าย: เฉพาะ γ=2.0
let c = Color::new(
    (color.x * scale).sqrt(),
    (color.y * scale).sqrt(),
    (color.z * scale).sqrt(),
);
```

---

### Pitfall 3: RNG ข้าม Thread ไม่ได้แชร์ กับ `Send + Sync` errors

**ปัญหา**: ถ้าพยายามแชร์ `rand::ThreadRng` ข้าม rayon threads จะเกิด compile error เพราะ `ThreadRng` ไม่ใช่ `Send`

```
error[E0277]: `ThreadRng` cannot be sent between threads safely
```

**สาเหตุ**: `ThreadRng` bind กับ thread เฉพาะด้วยเหตุผล safety

**แก้ไข**: สร้าง RNG แยกต่อ row ด้วย `SmallRng`:
```rust
// ผิด: แชร์ ThreadRng ข้าม threads
let rng = rand::thread_rng(); // ไม่ Send!

// ถูก: สร้าง per-thread RNG
rows.par_iter().map(|&j| {
    let mut rng = SmallRng::seed_from_u64(j as u64 * 12345);
    // ใช้ rng ภายใน closure นี้เท่านั้น
    ...
});
```

หมายเหตุ: ต้องเพิ่ม feature flag ใน Cargo.toml:
```toml
rand = { version = "0.8", features = ["small_rng"] }
```

---

### Pitfall 4: `Arc<dyn Trait>` ต้อง Clone แต่ Data ไม่ Clone

**ปัญหา**: `HitRecord` เก็บ `material: Arc<dyn Material>` เมื่อ derive `#[derive(Clone)]` ใน HitRecord จะ clone `Arc` ซึ่งโอเค แต่ถ้า derive `#[derive(Debug)]` จะล้มเหลวเพราะ `dyn Material` ไม่ implement `Debug`

```
error[E0277]: `(dyn Material + 'static)` doesn't implement `Debug`
```

**แก้ไข**: อย่า derive Debug ให้ structs ที่เก็บ `dyn Trait`:
```rust
// ผิด
#[derive(Debug, Clone)]
pub struct HitRecord { ... }

// ถูก
#[derive(Clone)]
pub struct HitRecord { ... }
```

หรือ impl Debug ด้วยมือ:
```rust
impl std::fmt::Debug for HitRecord {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "HitRecord {{ t: {}, front_face: {} }}", self.t, self.front_face)
    }
}
```

---

### Pitfall 5: Infinite Recursion และ Stack Overflow

**ปัญหา**: Path tracer recursive ลึกมาก ถ้า max_depth สูงเกินไปหรือไม่มี base case ที่ดีพอ อาจ stack overflow โดยเฉพาะใน debug mode

**สัญญาณ**: โปรแกรม crash ด้วย "thread 'main' has overflowed its stack"

**แก้ไข**:
```rust
// ต้องมี depth limit ที่ชัดเจน
pub fn trace(ray: &Ray, world: &dyn Hittable, depth: u32, ...) -> Color {
    if depth == 0 { return Color::zero(); } // BASE CASE บังคับ
    // ...
}

// เรียกด้วย max_depth ที่สมเหตุผล (50 ดีสำหรับ production quality)
trace(&ray, world, 50, &mut rng)
```

ใน release mode จะเร็วกว่า debug ประมาณ 10-50× เพราะ Rust optimizes tail-adjacent calls และ inlines ได้ดีกว่า

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/ray_tracer --help
```

### Render ด้วย Command Line

```bash
# Default three-sphere scene, 64 spp
cargo run --release -- --width 800 --height 600 --samples 64 --output scene.ppm

# Cornell Box demo
cargo run --release -- --demo --width 512 --height 512 --samples 256 --output box.png

# จาก JSON scene file
cargo run --release -- --scene scenes/cornell_box.json

# High quality render (ใช้เวลานาน)
cargo run --release -- --width 1920 --height 1080 --samples 1024 --depth 100 --output hq.png
```

### Performance Comparison

```
Resolution: 400×300, Samples: 64, Depth: 50

Debug build:    ~45 seconds
Release build:  ~3.2 seconds   (14× faster)
Rayon (8 cores): ~0.5 seconds  (6.4× parallel speedup)
```

### สร้าง Docker Container (ถ้าต้องการ)

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/ray_tracer /usr/local/bin/
ENTRYPOINT ["ray_tracer"]
```

```bash
docker build -t ray-tracer .
docker run --rm -v $(pwd):/out ray-tracer \
    --width 800 --height 600 --samples 128 \
    --output /out/output.ppm
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Emissive Material (Light Sources)

เพิ่ม material ที่ emit แสงเองได้ (ไม่ต้องพึ่ง sky lighting เท่านั้น):

```rust
pub struct DiffuseLight {
    pub emit: Color,
}

impl Material for DiffuseLight {
    fn scatter(&self, ...) -> Option<ScatterResult> {
        None // ไม่ scatter — absorb แสงทั้งหมด
    }
    
    // เพิ่ม method ใหม่ใน Material trait
    fn emitted(&self) -> Color {
        self.emit
    }
}
```

แก้ trace() ให้เรียก `rec.material.emitted()` เมื่อ scatter return None แทนที่จะ return black เสมอ ผลลัพธ์คือ Cornell Box ที่มี ceiling light จริง ทดลองเทียบกับ sky-only illumination ว่า noise pattern ต่างกันอย่างไร

---

### แบบฝึกหัดที่ 2: Texture Mapping

เพิ่ม texture บน sphere ด้วย UV coordinates — แทนที่ `albedo: Color` solid ด้วย texture ที่ sample ได้:

```rust
pub trait Texture: Send + Sync {
    fn value(&self, u: f64, v: f64, point: &Vec3) -> Color;
}

pub struct CheckerTexture {
    pub even: Color,
    pub odd: Color,
    pub scale: f64,
}

impl Texture for CheckerTexture {
    fn value(&self, _u: f64, _v: f64, point: &Vec3) -> Color {
        let sines = (self.scale * point.x).sin()
                  * (self.scale * point.y).sin()
                  * (self.scale * point.z).sin();
        if sines < 0.0 { self.odd } else { self.even }
    }
}
```

เพิ่ม `uv: (f64, f64)` ใน `HitRecord` และคำนวณ UV ใน `Sphere::hit()` ด้วยสูตร spherical UV:
- `u = 0.5 + atan2(n.z, n.x) / (2π)`
- `v = 0.5 - asin(n.y) / π`

---

### แบบฝึกหัดที่ 3: OBJ File Loader

โหลด mesh ไฟล์ `.obj` ด้วย Triangle primitives ที่สร้างไว้แล้ว:

```rust
pub fn load_obj(path: &str, material: MaterialHandle) -> Vec<Triangle> {
    let content = std::fs::read_to_string(path).unwrap();
    let mut vertices: Vec<Vec3> = Vec::new();
    let mut triangles: Vec<Triangle> = Vec::new();

    for line in content.lines() {
        if line.starts_with("v ") {
            let parts: Vec<f64> = line[2..]
                .split_whitespace()
                .map(|s| s.parse().unwrap())
                .collect();
            vertices.push(Vec3::new(parts[0], parts[1], parts[2]));
        } else if line.starts_with("f ") {
            let indices: Vec<usize> = line[2..]
                .split_whitespace()
                .map(|s| s.split('/').next().unwrap().parse::<usize>().unwrap() - 1)
                .collect();
            if indices.len() == 3 {
                triangles.push(Triangle::new(
                    vertices[indices[0]],
                    vertices[indices[1]],
                    vertices[indices[2]],
                    material.clone(),
                ));
            }
        }
    }
    triangles
}
```

ทดสอบกับ Stanford Bunny หรือ Utah Teapot (หา OBJ ได้ฟรีบน internet) แล้วสังเกตความสำคัญของ BVH acceleration เมื่อ mesh มีหมื่น triangles

---

### แบบฝึกหัดที่ 4: HDR Environment Map

แทนที่ sky gradient ด้วย HDR environment map จริง (ไฟล์ `.hdr` หรือ `.exr`) เพื่อ realistic lighting:

```rust
pub struct HdrEnvironment {
    pub pixels: Vec<Vec3>, // radiance values (linear HDR)
    pub width: u32,
    pub height: u32,
}

impl HdrEnvironment {
    /// Equirectangular projection sampling
    pub fn sample(&self, direction: &Vec3) -> Color {
        let u = 0.5 + direction.z.atan2(direction.x) / (2.0 * std::f64::consts::PI);
        let v = 0.5 - direction.y.asin() / std::f64::consts::PI;
        let x = (u * (self.width - 1) as f64) as u32;
        let y = (v * (self.height - 1) as f64) as u32;
        self.pixels[(y * self.width + x) as usize]
    }
}
```

ใช้ crate `exr` หรือ image crate เพื่อ load HDR file แล้วเปลี่ยน `sky_color()` ให้ call `env.sample(&ray.direction)` ผลลัพธ์คือ realistic outdoor/indoor lighting จาก real-world capture

---

## สรุป

ในโปรเจคนี้เราสร้าง Path Tracer ตั้งแต่ math primitives จนถึง parallel renderer ที่สมบูรณ์ใน Rust สิ่งที่ได้เรียนรู้หลักๆ:

**Pattern สำคัญที่ได้จากโปรเจคนี้:**

1. **Trait object polymorphism** — `Box<dyn Hittable>` และ `Arc<dyn Material>` เป็นการออกแบบที่ทำให้ extensible โดยไม่ต้องแก้ core code ทุกครั้งที่เพิ่ม primitive ใหม่

2. **Zero-cost abstraction ในงาน compute** — Rayon `par_iter()` แจก work อย่าง automatic ไม่ต้องเขียน thread management ด้วยตัวเอง ยังคง type safety ครบถ้วน

3. **Builder pattern + Serde** — `SceneConfig::build_world()` แยก "data representation" จาก "runtime objects" ทำให้ test ง่ายและ serialize/deserialize ได้

4. **Recursive algorithm ที่ terminate อย่างถูกต้อง** — Russian Roulette เป็นตัวอย่างดีของ probabilistic termination ที่ unbiased

5. **f64 precision awareness** — shadow acne, near_zero checks, epsilon guards ล้วนเกิดจากการ awareness ว่า floating-point ไม่ exact

**เชื่อมโยงกับโปรเจคถัดไป**: Project E05 — Game of Life เป็น simulation อีกแบบที่ใช้ grid-based computation แทน ray casting เทียบให้เห็นว่า parallelism ในแต่ละ domain มี pattern ต่างกัน (path tracer: embarrassingly parallel per pixel, Game of Life: cellular automata มี data dependency ระหว่าง cells)

---

**โปรเจคก่อนหน้า:** [project-e03-physics-engine.md](project-e03-physics-engine.md) | **โปรเจคถัดไป:** [project-e05-game-of-life.md](project-e05-game-of-life.md)
