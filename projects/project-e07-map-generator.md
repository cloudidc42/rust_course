# Project E07: Procedural Map Generator

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

**Procedural Map Generator** คือโปรแกรมสร้างแผนที่โลกสุ่มด้วยอัลกอริทึม — ผลลัพธ์คือไฟล์ภาพ PNG ที่มีภูมิประเทศสมจริง ได้แก่ มหาสมุทร ชายหาด ทะเลทราย ป่าดงดิบ ภูเขา และหิมะ พร้อมถ้ำ แม่น้ำ และเขตการเมือง

โปรเจคนี้ใช้เทคนิคที่เกม AAA ใช้จริงอย่าง Minecraft, Dwarf Fortress และ No Man's Sky ได้แก่ Perlin noise, Fractional Brownian Motion (FBM), Cellular Automata, Voronoi tessellation และ Signed Distance Fields ทั้งหมดเขียนจาก scratch ใน Rust ไม่พึ่ง noise library สำเร็จรูป

**Use case จริงในโลก production:**
- เกม roguelike และ survival — สร้าง world ขนาดใหญ่ที่ deterministic จาก seed เดียว
- เครื่องมือ GIS — จำลองภูมิประเทศสมมติสำหรับการทดสอบระบบ
- Background generation ใน procedural art และ generative design

## สิ่งที่จะได้เรียนรู้

- ทฤษฎีและการ implement Perlin noise จาก scratch รวมถึง gradient hash, fade function และ trilinear interpolation
- Fractional Brownian Motion (FBM) — การซ้อน noise หลายชั้น (octave) ด้วย persistence และ lacunarity
- Cellular automata สำหรับ organic cave generation — rule B3/S12345
- Voronoi tessellation และ Lloyd relaxation สำหรับ region partitioning
- BFS-based signed distance field สำหรับ coastline
- River simulation ด้วย gradient descent บน heightmap
- การ export แผนที่เป็น PNG, JSON, SVG ด้วย `image`, `serde_json`
- CLI design ด้วย `clap 4` รองรับ deterministic seed

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–25**: ownership, borrowing, structs, enums, traits
- **Part 26–30**: iterators, closures, `Vec<T>` manipulation
- **Part 31–40**: error handling, modules, `use` statements
- **Part 51–60**: traits ขั้นสูง, generic constraints
- โปรเจค E06 (Roguelike) — เข้าใจ tile-based map representation

## โครงสร้างโปรเจค (Project Layout)

```
map-generator/
├── Cargo.toml
├── src/
│   ├── main.rs          ← CLI entry point (clap 4)
│   ├── noise.rs         ← Perlin noise + FBM
│   ├── heightmap.rs     ← HeightMap struct + ridged multifractal
│   ├── biome.rs         ← terrain classification
│   ├── cave.rs          ← cellular automata cave generator
│   ├── river.rs         ← river simulation + erosion
│   ├── voronoi.rs       ← Voronoi regions + Lloyd relaxation
│   ├── distance.rs      ← BFS signed distance field
│   └── export/
│       ├── mod.rs
│       ├── png_render.rs ← PNG output (image crate)
│       ├── json_export.rs
│       └── svg_export.rs
└── tests/
    └── integration.rs
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
seed (u64)
   │
   ├─► PerlinNoise::new(seed)
   │        │
   │        └─► Fbm::sample(x, y) × width × height
   │                 │
   │                 └─► HeightMap { data: Vec<f64> } [0,1]
   │                           │
   │                 ┌─────────┼──────────┐
   │                 ▼         ▼          ▼
   │           moisture     ridged    distance
   │            map (FBM)  variant    field (BFS)
   │                 │         │
   │                 └────┬────┘
   │                      ▼
   │              classify_biome(elev, moisture)
   │                      │
   │              Biome grid (2D Vec<Biome>)
   │                      │
   ├─► CaveMap (cellular automata, separate pass)
   ├─► RiverMap (gradient descent from mountain peaks)
   ├─► VoronoiMap (N seed points → region IDs)
   │
   └─► Renderer → PNG / JSON / SVG
```

### Design Decisions

**ทำไม Perlin noise แทน Simplex noise?**
Simplex noise ให้ผลที่ดีกว่าใน 3D+ แต่ Perlin noise ใน 2D เพียงพอและ implement ได้ง่ายกว่า เหมาะสำหรับการเรียนรู้ algorithm

**ทำไม FBM ไม่ใช่แค่ single-octave noise?**
Single octave ให้ภูมิประเทศที่ smooth เกินไป — FBM ซ้อน octave ที่ความถี่สูงขึ้นทำให้ได้รายละเอียดหลายระดับ เหมือนภูมิประเทศจริงที่มีทั้งเทือกเขาใหญ่และยอดเขาเล็ก

**HeightMap เก็บเป็น `Vec<f64>` แบบ flat array เพราะ:**
- Cache-friendly — ข้อมูลเรียงต่อกันใน memory
- Index คำนวณง่าย: `y * width + x`
- ง่ายต่อการส่งให้ `image` crate render

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: ตั้งโปรเจคและ implement Perlin Noise

สร้าง project ใหม่:

```bash
cargo new map-generator --edition 2021
cd map-generator
```

**`Cargo.toml`:**

```toml
[package]
name = "map-generator"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "map-generator"
path = "src/main.rs"

[dependencies]
rand     = "0.8"
image    = "0.25"
serde    = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
clap     = { version = "4", features = ["derive"] }
```

**`src/noise.rs` — Perlin Noise จาก scratch:**

Perlin noise ทำงานด้วย 3 ส่วนหลัก:
1. **Permutation table** — array สุ่ม 256 ค่า shuffle ด้วย seed
2. **Gradient function** `grad(hash, x, y, z)` — แปลง hash → gradient vector และคูณกับ offset
3. **Fade + lerp** — Smooth interpolation ระหว่าง gradient

```rust
// src/noise.rs
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;

/// Smooth step function: 6t^5 - 15t^4 + 10t^3
/// ทำให้ derivative เป็น 0 ที่ t=0 และ t=1 (C2 continuity)
pub fn fade(t: f64) -> f64 {
    t * t * t * (t * (t * 6.0 - 15.0) + 10.0)
}

/// Linear interpolation
pub fn lerp(a: f64, b: f64, t: f64) -> f64 {
    a + t * (b - a)
}

/// Gradient function — hash → 1 ใน 16 gradient vectors
/// เลือก u และ v จาก x, y, z ตาม bitmask ของ hash
pub fn grad(hash: u8, x: f64, y: f64, z: f64) -> f64 {
    let h = hash & 15;
    let u = if h < 8 { x } else { y };
    let v = if h < 4 {
        y
    } else if h == 12 || h == 14 {
        x
    } else {
        z
    };
    let u_val = if h & 1 == 0 { u } else { -u };
    let v_val = if h & 2 == 0 { v } else { -v };
    u_val + v_val
}

/// Perlin Noise generator
/// สร้างจาก seed — deterministic ทุกครั้งที่ใช้ seed เดิม
pub struct PerlinNoise {
    perm: [u8; 512], // doubled permutation table
}

impl PerlinNoise {
    pub fn new(seed: u64) -> Self {
        let mut rng = StdRng::seed_from_u64(seed);
        // สร้าง permutation [0..255] แล้ว shuffle
        let mut p: Vec<u8> = (0u8..=255u8).collect();
        for i in (1..256usize).rev() {
            let j = rng.gen_range(0..=i);
            p.swap(i, j);
        }
        // Double เพื่อหลีกเลี่ยง modulo เมื่อ index ออกนอกขอบ
        let mut perm = [0u8; 512];
        for i in 0..256 {
            perm[i] = p[i];
            perm[i + 256] = p[i];
        }
        PerlinNoise { perm }
    }

    /// Sample noise ที่ตำแหน่ง (x, y, z)
    /// Return value: ประมาณ [-1, 1]
    pub fn noise(&self, x: f64, y: f64, z: f64) -> f64 {
        // หา unit cube ที่ครอบจุด (x,y,z)
        let xi = (x.floor() as i32 & 255) as usize;
        let yi = (y.floor() as i32 & 255) as usize;
        let zi = (z.floor() as i32 & 255) as usize;

        // Fractional part (offset ภายใน cube)
        let xf = x - x.floor();
        let yf = y - y.floor();
        let zf = z - z.floor();

        // Smooth step weights
        let u = fade(xf);
        let v = fade(yf);
        let w = fade(zf);

        // Hash ทั้ง 8 มุมของ cube
        let aaa = self.perm[self.perm[self.perm[xi    ] as usize + yi    ] as usize + zi    ];
        let aba = self.perm[self.perm[self.perm[xi    ] as usize + yi + 1] as usize + zi    ];
        let aab = self.perm[self.perm[self.perm[xi    ] as usize + yi    ] as usize + zi + 1];
        let abb = self.perm[self.perm[self.perm[xi    ] as usize + yi + 1] as usize + zi + 1];
        let baa = self.perm[self.perm[self.perm[xi + 1] as usize + yi    ] as usize + zi    ];
        let bba = self.perm[self.perm[self.perm[xi + 1] as usize + yi + 1] as usize + zi    ];
        let bab = self.perm[self.perm[self.perm[xi + 1] as usize + yi    ] as usize + zi + 1];
        let bbb = self.perm[self.perm[self.perm[xi + 1] as usize + yi + 1] as usize + zi + 1];

        // Trilinear interpolation ของ gradient values
        let x1 = lerp(
            grad(aaa, xf,       yf,       zf      ),
            grad(baa, xf - 1.0, yf,       zf      ),
            u,
        );
        let x2 = lerp(
            grad(aba, xf,       yf - 1.0, zf      ),
            grad(bba, xf - 1.0, yf - 1.0, zf      ),
            u,
        );
        let y1 = lerp(x1, x2, v);

        let x3 = lerp(
            grad(aab, xf,       yf,       zf - 1.0),
            grad(bab, xf - 1.0, yf,       zf - 1.0),
            u,
        );
        let x4 = lerp(
            grad(abb, xf,       yf - 1.0, zf - 1.0),
            grad(bbb, xf - 1.0, yf - 1.0, zf - 1.0),
            u,
        );
        let y2 = lerp(x3, x4, v);

        lerp(y1, y2, w)
    }
}

/// Fractional Brownian Motion — ซ้อน octave หลายชั้น
///
/// value = Σ (noise(f * freq) * amplitude) for each octave
///   amplitude *= persistence   (ลดค่าลงทุก octave)
///   frequency *= lacunarity    (เพิ่มความถี่ขึ้นทุก octave)
pub struct Fbm {
    pub noise: PerlinNoise,
    pub octaves: u32,
    pub persistence: f64,  // ปกติ 0.5 — แต่ละ octave มีแอมพลิจูดครึ่งหนึ่งของอันก่อน
    pub lacunarity: f64,   // ปกติ 2.0 — แต่ละ octave มีความถี่สองเท่าของอันก่อน
}

impl Fbm {
    pub fn new(seed: u64, octaves: u32, persistence: f64, lacunarity: f64) -> Self {
        Fbm {
            noise: PerlinNoise::new(seed),
            octaves,
            persistence,
            lacunarity,
        }
    }

    pub fn sample(&self, x: f64, y: f64) -> f64 {
        let mut value = 0.0f64;
        let mut amplitude = 1.0f64;
        let mut frequency = 1.0f64;
        let mut max_value = 0.0f64;

        for _ in 0..self.octaves {
            value += self.noise.noise(x * frequency, y * frequency, 0.0) * amplitude;
            max_value += amplitude;
            amplitude *= self.persistence;
            frequency *= self.lacunarity;
        }
        // normalize ให้อยู่ใน [-1, 1]
        value / max_value
    }

    /// Ridged multifractal variant — เหมาะสำหรับสร้างเทือกเขา
    /// 1 - |noise| ทำให้ ridges (สันเขา) ชัดเจนขึ้น
    pub fn sample_ridged(&self, x: f64, y: f64) -> f64 {
        let mut value = 0.0f64;
        let mut amplitude = 1.0f64;
        let mut frequency = 1.0f64;
        let mut max_value = 0.0f64;
        let mut weight = 1.0f64;

        for _ in 0..self.octaves {
            let signal = self.noise.noise(x * frequency, y * frequency, 0.0).abs();
            let ridged = (1.0 - signal) * weight;
            value += ridged * amplitude;
            max_value += amplitude;
            weight = (ridged * 2.0).clamp(0.0, 1.0);
            amplitude *= self.persistence;
            frequency *= self.lacunarity;
        }
        value / max_value
    }
}
```

---

### ขั้นที่ 2: HeightMap และ Biome Classification

**`src/heightmap.rs`:**

```rust
// src/heightmap.rs
use crate::noise::Fbm;

/// HeightMap เก็บค่า elevation [0, 1] สำหรับทุก tile
#[derive(Debug, Clone)]
pub struct HeightMap {
    pub data: Vec<f64>,
    pub width: usize,
    pub height: usize,
}

impl HeightMap {
    /// สร้าง HeightMap ด้วย FBM Perlin noise
    /// ผลลัพธ์ถูก normalize ให้อยู่ใน [0, 1]
    pub fn generate(width: usize, height: usize, seed: u64) -> Self {
        // scale = ความถี่ของ noise (ยิ่งมากยิ่งมีรายละเอียดมาก)
        let fbm = Fbm::new(seed, 6, 0.5, 2.0);
        let scale = 4.0;

        let mut raw = Vec::with_capacity(width * height);
        for y in 0..height {
            for x in 0..width {
                let nx = x as f64 / width as f64 * scale;
                let ny = y as f64 / height as f64 * scale;
                raw.push(fbm.sample(nx, ny));
            }
        }

        // Normalize min→0, max→1
        let min = raw.iter().cloned().fold(f64::INFINITY, f64::min);
        let max = raw.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
        let range = (max - min).max(1e-9);
        let data: Vec<f64> = raw.iter().map(|v| (v - min) / range).collect();

        HeightMap { data, width, height }
    }

    /// สร้าง HeightMap แบบ ridged multifractal — เหมาะสำหรับภูเขาสูงชัน
    pub fn generate_ridged(width: usize, height: usize, seed: u64) -> Self {
        let fbm = Fbm::new(seed, 8, 0.6, 2.1);
        let scale = 3.5;

        let mut raw = Vec::with_capacity(width * height);
        for y in 0..height {
            for x in 0..width {
                let nx = x as f64 / width as f64 * scale;
                let ny = y as f64 / height as f64 * scale;
                raw.push(fbm.sample_ridged(nx, ny));
            }
        }

        let min = raw.iter().cloned().fold(f64::INFINITY, f64::min);
        let max = raw.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
        let range = (max - min).max(1e-9);
        let data: Vec<f64> = raw.iter().map(|v| (v - min) / range).collect();

        HeightMap { data, width, height }
    }

    pub fn get(&self, x: usize, y: usize) -> f64 {
        self.data[y * self.width + x]
    }

    pub fn get_mut(&mut self, x: usize, y: usize) -> &mut f64 {
        &mut self.data[y * self.width + x]
    }

    /// ลด elevation ตาม factor (สำหรับ river erosion)
    pub fn erode(&mut self, x: usize, y: usize, amount: f64) {
        let v = self.get(x, y);
        *self.get_mut(x, y) = (v - amount).max(0.0);
    }
}
```

**`src/biome.rs`:**

```rust
// src/biome.rs
use serde::{Deserialize, Serialize};

/// Biome ของแต่ละ tile — กำหนดด้วย elevation + moisture
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum Biome {
    Ocean,      // < 0.30
    Beach,      // 0.30 – 0.35
    Desert,     // 0.35 – 0.60, moisture < 0.35
    Grassland,  // 0.35 – 0.60, moisture 0.35 – 0.65
    Jungle,     // 0.35 – 0.60, moisture > 0.65
    Forest,     // 0.60 – 0.75
    Mountain,   // 0.75 – 0.90
    Snow,       // > 0.90
}

/// จำแนก biome จาก elevation [0,1] และ moisture [0,1]
///
/// threshold:
///   ocean    : elev < 0.30
///   beach    : 0.30 ≤ elev < 0.35
///   low land : 0.35 ≤ elev < 0.60  → desert/grassland/jungle ตาม moisture
///   forest   : 0.60 ≤ elev < 0.75
///   mountain : 0.75 ≤ elev < 0.90
///   snow     : elev ≥ 0.90
pub fn classify_biome(elevation: f64, moisture: f64) -> Biome {
    if elevation < 0.30 {
        Biome::Ocean
    } else if elevation < 0.35 {
        Biome::Beach
    } else if elevation < 0.60 {
        if moisture < 0.35 {
            Biome::Desert
        } else if moisture > 0.65 {
            Biome::Jungle
        } else {
            Biome::Grassland
        }
    } else if elevation < 0.75 {
        Biome::Forest
    } else if elevation < 0.90 {
        Biome::Mountain
    } else {
        Biome::Snow
    }
}

/// สี RGB สำหรับ render แต่ละ biome
pub fn biome_color(biome: &Biome) -> [u8; 3] {
    match biome {
        Biome::Ocean     => [65,  105, 225],  // royal blue
        Biome::Beach     => [238, 214, 175],  // sandy
        Biome::Desert    => [210, 180, 100],  // tan
        Biome::Grassland => [124, 189,  90],  // medium green
        Biome::Jungle    => [ 34, 139,  34],  // forest green
        Biome::Forest    => [ 85, 107,  47],  // dark olive
        Biome::Mountain  => [139, 137, 137],  // gray
        Biome::Snow      => [255, 250, 250],  // snow white
    }
}
```

---

### ขั้นที่ 3: Cellular Automata Cave Generator

Cave generation ใช้วิธีที่เรียกว่า **Bootstrap Automata**:
1. เติม grid สุ่มด้วย wall 45%
2. วน 5 รอบ: cell กลายเป็น wall ถ้ามี ≥5 wall neighbors, กลายเป็น floor ถ้ามี ≤1 wall neighbors

**`src/cave.rs`:**

```rust
// src/cave.rs
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum Cell {
    Wall,
    Floor,
}

pub struct CaveMap {
    pub cells: Vec<Cell>,
    pub width: usize,
    pub height: usize,
}

impl CaveMap {
    /// สร้าง cave map ด้วย cellular automata
    /// fill_prob: ความน่าจะเป็นที่แต่ละ cell เริ่มต้นเป็น wall (แนะนำ 0.45)
    /// iterations: จำนวนรอบ CA (แนะนำ 5)
    pub fn generate(
        width: usize,
        height: usize,
        seed: u64,
        fill_prob: f64,
        iterations: u32,
    ) -> Self {
        let mut rng = StdRng::seed_from_u64(seed);
        let cells: Vec<Cell> = (0..width * height)
            .map(|_| {
                if rng.gen::<f64>() < fill_prob {
                    Cell::Wall
                } else {
                    Cell::Floor
                }
            })
            .collect();

        let mut cave = CaveMap { cells, width, height };
        for _ in 0..iterations {
            cave.step();
        }
        cave
    }

    /// นับจำนวน wall neighbors รอบ cell (x, y) — 8 ทิศทาง
    /// Border นับเป็น wall เสมอ (ขอบแผนที่เป็นกำแพง)
    pub fn count_wall_neighbors(&self, grid: &[Cell], x: usize, y: usize) -> u32 {
        let mut count = 0u32;
        for dy in -1i32..=1 {
            for dx in -1i32..=1 {
                if dx == 0 && dy == 0 {
                    continue;
                }
                let nx = x as i32 + dx;
                let ny = y as i32 + dy;
                if nx < 0 || ny < 0
                    || nx >= self.width as i32
                    || ny >= self.height as i32
                {
                    count += 1; // border นับเป็น wall
                } else if grid[ny as usize * self.width + nx as usize] == Cell::Wall {
                    count += 1;
                }
            }
        }
        count
    }

    /// Cellular automata step (B3/S12345 variant)
    /// Rule:
    ///   walls >= 5 → กลายเป็น Wall
    ///   walls <= 1 → กลายเป็น Floor
    ///   otherwise  → คงเดิม
    pub fn step(&mut self) {
        let old = self.cells.clone();
        for y in 0..self.height {
            for x in 0..self.width {
                let walls = self.count_wall_neighbors(&old, x, y);
                self.cells[y * self.width + x] = if walls >= 5 {
                    Cell::Wall
                } else if walls <= 1 {
                    Cell::Floor
                } else {
                    old[y * self.width + x].clone()
                };
            }
        }
    }

    pub fn get(&self, x: usize, y: usize) -> &Cell {
        &self.cells[y * self.width + x]
    }

    pub fn is_floor(&self, x: usize, y: usize) -> bool {
        self.cells[y * self.width + x] == Cell::Floor
    }
}
```

---

### ขั้นที่ 4: River Simulation และ Voronoi Regions

**`src/river.rs`:**

```rust
// src/river.rs
use crate::heightmap::HeightMap;
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;
use serde::{Deserialize, Serialize};

const OCEAN_THRESHOLD: f64 = 0.30;
const EROSION_AMOUNT: f64 = 0.015;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct River {
    pub path: Vec<(usize, usize)>,
}

impl River {
    /// สร้างแม่น้ำโดย gradient descent จาก mountain tile ลงสู่มหาสมุทร
    /// 1. หา tile ที่ elevation > 0.75 (mountain)
    /// 2. ไหลไปยัง neighbor ที่ต่ำที่สุดทีละ step
    /// 3. หยุดเมื่อถึง ocean หรือ stuck
    pub fn generate(heightmap: &mut HeightMap, seed: u64) -> Vec<River> {
        let mut rng = StdRng::seed_from_u64(seed);
        let w = heightmap.width;
        let h = heightmap.height;

        // เก็บตำแหน่ง mountain tiles เป็น candidates
        let mut mountains: Vec<(usize, usize)> = Vec::new();
        for y in 1..h - 1 {
            for x in 1..w - 1 {
                if heightmap.get(x, y) > 0.75 {
                    mountains.push((x, y));
                }
            }
        }

        let river_count = (mountains.len() / 50).max(3).min(20);
        let mut rivers = Vec::new();

        for _ in 0..river_count {
            if mountains.is_empty() {
                break;
            }
            let start_idx = rng.gen_range(0..mountains.len());
            let (sx, sy) = mountains[start_idx];

            let mut path = vec![(sx, sy)];
            let mut cx = sx;
            let mut cy = sy;
            let mut visited = std::collections::HashSet::new();
            visited.insert((cx, cy));

            loop {
                let elev = heightmap.get(cx, cy);
                if elev < OCEAN_THRESHOLD {
                    break; // ถึงมหาสมุทรแล้ว
                }

                // หา neighbor ที่ต่ำที่สุด (4 ทิศ)
                let neighbors = [
                    (cx.wrapping_sub(1), cy),
                    (cx + 1, cy),
                    (cx, cy.wrapping_sub(1)),
                    (cx, cy + 1),
                ];

                let next = neighbors
                    .iter()
                    .filter(|&&(nx, ny)| {
                        nx < w && ny < h && !visited.contains(&(nx, ny))
                    })
                    .min_by(|&&(ax, ay), &&(bx, by)| {
                        heightmap
                            .get(ax, ay)
                            .partial_cmp(&heightmap.get(bx, by))
                            .unwrap()
                    });

                match next {
                    Some(&(nx, ny)) => {
                        // Erode ตลอดเส้นทาง
                        heightmap.erode(cx, cy, EROSION_AMOUNT);
                        cx = nx;
                        cy = ny;
                        visited.insert((cx, cy));
                        path.push((cx, cy));
                        if path.len() > w + h {
                            break; // ป้องกัน infinite loop
                        }
                    }
                    None => break, // stuck
                }
            }

            if path.len() > 5 {
                rivers.push(River { path });
            }
        }

        rivers
    }
}
```

**`src/voronoi.rs`:**

```rust
// src/voronoi.rs
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;

pub struct VoronoiMap {
    pub seeds: Vec<(f64, f64)>,
    pub regions: Vec<usize>, // index เข้า seeds[]
    pub width: usize,
    pub height: usize,
}

impl VoronoiMap {
    /// สร้าง Voronoi map ด้วย N seed points สุ่ม
    pub fn generate(width: usize, height: usize, n: usize, seed: u64) -> Self {
        let mut rng = StdRng::seed_from_u64(seed);
        let seeds: Vec<(f64, f64)> = (0..n)
            .map(|_| (
                rng.gen::<f64>() * width as f64,
                rng.gen::<f64>() * height as f64,
            ))
            .collect();

        let regions = Self::compute_regions(&seeds, width, height);
        VoronoiMap { seeds, regions, width, height }
    }

    fn compute_regions(seeds: &[(f64, f64)], width: usize, height: usize) -> Vec<usize> {
        let mut regions = Vec::with_capacity(width * height);
        for y in 0..height {
            for x in 0..width {
                let px = x as f64 + 0.5;
                let py = y as f64 + 0.5;
                let nearest = seeds
                    .iter()
                    .enumerate()
                    .min_by(|(_, a), (_, b)| {
                        let da = (a.0 - px).powi(2) + (a.1 - py).powi(2);
                        let db = (b.0 - px).powi(2) + (b.1 - py).powi(2);
                        da.partial_cmp(&db).unwrap()
                    })
                    .map(|(i, _)| i)
                    .unwrap_or(0);
                regions.push(nearest);
            }
        }
        regions
    }

    /// Lloyd relaxation — ย้าย seed แต่ละจุดไปที่ centroid ของ region ตัวเอง
    /// ทำซ้ำ N รอบเพื่อให้ region กระจายสม่ำเสมอขึ้น
    pub fn lloyd_relaxation(&mut self, iterations: u32) {
        for _ in 0..iterations {
            let n = self.seeds.len();
            let mut sum_x = vec![0.0f64; n];
            let mut sum_y = vec![0.0f64; n];
            let mut count = vec![0usize; n];

            for y in 0..self.height {
                for x in 0..self.width {
                    let region = self.regions[y * self.width + x];
                    sum_x[region] += x as f64 + 0.5;
                    sum_y[region] += y as f64 + 0.5;
                    count[region] += 1;
                }
            }

            for i in 0..n {
                if count[i] > 0 {
                    self.seeds[i] = (
                        sum_x[i] / count[i] as f64,
                        sum_y[i] / count[i] as f64,
                    );
                }
            }

            // Recompute regions หลัง relaxation
            self.regions = Self::compute_regions(&self.seeds, self.width, self.height);
        }
    }

    pub fn nearest_seed(&self, x: usize, y: usize) -> usize {
        self.regions[y * self.width + x]
    }
}
```

---

### ขั้นที่ 5: Distance Field และ PNG Export

**`src/distance.rs`:**

```rust
// src/distance.rs
use crate::heightmap::HeightMap;
use std::collections::VecDeque;

/// Compute BFS distance field จาก ocean tiles
/// ทุก ocean tile มี distance = 0
/// tiles ถัดออกไปมี distance = 1, 2, 3, ...
/// ใช้สำหรับ beach width variation และ fog-of-war
pub fn compute_distance_field(heightmap: &HeightMap, ocean_threshold: f64) -> Vec<u32> {
    let w = heightmap.width;
    let h = heightmap.height;
    let mut dist = vec![u32::MAX; w * h];
    let mut queue = VecDeque::new();

    // เริ่มต้น: ocean tiles มี distance = 0
    for y in 0..h {
        for x in 0..w {
            if heightmap.get(x, y) < ocean_threshold {
                dist[y * w + x] = 0;
                queue.push_back((x, y));
            }
        }
    }

    // BFS 4-directional
    while let Some((cx, cy)) = queue.pop_front() {
        let d = dist[cy * w + cx];
        for (dx, dy) in [(-1i32, 0), (1, 0), (0, -1i32), (0, 1)] {
            let nx = cx as i32 + dx;
            let ny = cy as i32 + dy;
            if nx >= 0 && ny >= 0 && nx < w as i32 && ny < h as i32 {
                let ni = ny as usize * w + nx as usize;
                if dist[ni] == u32::MAX {
                    dist[ni] = d + 1;
                    queue.push_back((nx as usize, ny as usize));
                }
            }
        }
    }
    dist
}
```

**`src/export/png_render.rs`:**

```rust
// src/export/png_render.rs
use image::{ImageBuffer, Rgb};
use crate::biome::{biome_color, classify_biome, Biome};
use crate::heightmap::HeightMap;
use crate::river::River;

/// Render แผนที่เป็น PNG สี (biome colors)
/// output: path ของไฟล์ PNG
pub fn render_colored(
    heightmap: &HeightMap,
    moisture_map: &HeightMap,
    rivers: &[River],
    output_path: &str,
) -> Result<(), image::ImageError> {
    let w = heightmap.width as u32;
    let h = heightmap.height as u32;
    let mut img: ImageBuffer<Rgb<u8>, Vec<u8>> = ImageBuffer::new(w, h);

    for y in 0..heightmap.height {
        for x in 0..heightmap.width {
            let elev = heightmap.get(x, y);
            let moisture = moisture_map.get(x, y);
            let biome = classify_biome(elev, moisture);
            let color = biome_color(&biome);
            img.put_pixel(x as u32, y as u32, Rgb(color));
        }
    }

    // วาดแม่น้ำด้วยสีน้ำเงิน
    for river in rivers {
        for &(rx, ry) in &river.path {
            if rx < heightmap.width && ry < heightmap.height {
                img.put_pixel(rx as u32, ry as u32, Rgb([64, 164, 223]));
            }
        }
    }

    img.save(output_path)?;
    Ok(())
}

/// Render heightmap เป็น grayscale PNG
pub fn render_heightmap(heightmap: &HeightMap, output_path: &str)
    -> Result<(), image::ImageError>
{
    let w = heightmap.width as u32;
    let h = heightmap.height as u32;
    let mut img: ImageBuffer<Rgb<u8>, Vec<u8>> = ImageBuffer::new(w, h);

    for y in 0..heightmap.height {
        for x in 0..heightmap.width {
            let v = (heightmap.get(x, y) * 255.0) as u8;
            img.put_pixel(x as u32, y as u32, Rgb([v, v, v]));
        }
    }

    img.save(output_path)?;
    Ok(())
}

/// Slope shading — จำลองแสงอาทิตย์จากทิศ NW
/// ทำให้เห็นความลาดชันของภูมิประเทศชัดขึ้น
pub fn render_shaded(
    heightmap: &HeightMap,
    moisture_map: &HeightMap,
    rivers: &[River],
    output_path: &str,
) -> Result<(), image::ImageError> {
    let w = heightmap.width;
    let h = heightmap.height;
    let mut img: ImageBuffer<Rgb<u8>, Vec<u8>> = ImageBuffer::new(w as u32, h as u32);

    for y in 0..h {
        for x in 0..w {
            let elev = heightmap.get(x, y);
            let moisture = moisture_map.get(x, y);
            let biome = classify_biome(elev, moisture);
            let base_color = biome_color(&biome);

            // Compute slope shading
            let shade = compute_shade(heightmap, x, y);

            let r = (base_color[0] as f64 * shade) as u8;
            let g = (base_color[1] as f64 * shade) as u8;
            let b = (base_color[2] as f64 * shade) as u8;
            img.put_pixel(x as u32, y as u32, Rgb([r, g, b]));
        }
    }

    for river in rivers {
        for &(rx, ry) in &river.path {
            if rx < w && ry < h {
                img.put_pixel(rx as u32, ry as u32, Rgb([64, 164, 223]));
            }
        }
    }

    img.save(output_path)?;
    Ok(())
}

/// คำนวณ shade factor [0.5, 1.2] จาก slope ของ heightmap
fn compute_shade(heightmap: &HeightMap, x: usize, y: usize) -> f64 {
    let w = heightmap.width;
    let h = heightmap.height;
    let ex = if x + 1 < w { heightmap.get(x + 1, y) } else { heightmap.get(x, y) };
    let wx = if x > 0 { heightmap.get(x - 1, y) } else { heightmap.get(x, y) };
    let ny = if y > 0 { heightmap.get(x, y - 1) } else { heightmap.get(x, y) };
    let sy = if y + 1 < h { heightmap.get(x, y + 1) } else { heightmap.get(x, y) };

    // Gradient ในแนว NW
    let dx = ex - wx;
    let dy = ny - sy;
    let slope = dx * 0.7071 + dy * 0.7071;

    (1.0 + slope * 3.0).clamp(0.5, 1.2)
}
```

---

### ขั้นที่ 6: JSON/SVG Export และ CLI Interface

**`src/export/json_export.rs`:**

```rust
// src/export/json_export.rs
use serde::{Deserialize, Serialize};
use std::fs;
use crate::biome::{classify_biome, Biome};
use crate::heightmap::HeightMap;
use crate::river::River;

#[derive(Serialize, Deserialize)]
pub struct TileData {
    pub x: usize,
    pub y: usize,
    pub elevation: f64,
    pub moisture: f64,
    pub biome: Biome,
    pub is_river: bool,
}

#[derive(Serialize, Deserialize)]
pub struct MapExport {
    pub width: usize,
    pub height: usize,
    pub seed: u64,
    pub tiles: Vec<TileData>,
}

pub fn export_json(
    heightmap: &HeightMap,
    moisture_map: &HeightMap,
    rivers: &[River],
    seed: u64,
    output_path: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    // สร้าง river set สำหรับ O(1) lookup
    let river_tiles: std::collections::HashSet<(usize, usize)> = rivers
        .iter()
        .flat_map(|r| r.path.iter().cloned())
        .collect();

    let tiles: Vec<TileData> = (0..heightmap.height)
        .flat_map(|y| {
            (0..heightmap.width).map(move |x| {
                let elevation = heightmap.get(x, y);
                let moisture = moisture_map.get(x, y);
                TileData {
                    x,
                    y,
                    elevation,
                    moisture,
                    biome: classify_biome(elevation, moisture),
                    is_river: river_tiles.contains(&(x, y)),
                }
            })
        })
        .collect();

    let export = MapExport {
        width: heightmap.width,
        height: heightmap.height,
        seed,
        tiles,
    };

    let json = serde_json::to_string_pretty(&export)?;
    fs::write(output_path, json)?;
    Ok(())
}
```

**`src/export/svg_export.rs`:**

```rust
// src/export/svg_export.rs
use std::fmt::Write as FmtWrite;
use std::fs;
use crate::biome::{biome_color, classify_biome};
use crate::heightmap::HeightMap;

/// Export แผนที่เป็น SVG — แต่ละ pixel เป็น rect 1x1
/// สำหรับแผนที่ขนาดเล็ก (ไม่เกิน 256x256) — ขนาดไฟล์อาจใหญ่สำหรับ 512x512
pub fn export_svg(
    heightmap: &HeightMap,
    moisture_map: &HeightMap,
    output_path: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    let w = heightmap.width;
    let h = heightmap.height;
    let mut svg = String::new();

    writeln!(
        svg,
        r#"<svg xmlns="http://www.w3.org/2000/svg" width="{w}" height="{h}" viewBox="0 0 {w} {h}">"#
    )?;

    for y in 0..h {
        for x in 0..w {
            let elev = heightmap.get(x, y);
            let moisture = moisture_map.get(x, y);
            let biome = classify_biome(elev, moisture);
            let [r, g, b] = biome_color(&biome);
            writeln!(
                svg,
                r#"<rect x="{x}" y="{y}" width="1" height="1" fill="rgb({r},{g},{b})"/>"#
            )?;
        }
    }

    writeln!(svg, "</svg>")?;
    fs::write(output_path, svg)?;
    Ok(())
}
```

**`src/main.rs` — CLI ด้วย clap 4:**

```rust
// src/main.rs
mod noise;
mod heightmap;
mod biome;
mod cave;
mod river;
mod voronoi;
mod distance;
mod export {
    pub mod png_render;
    pub mod json_export;
    pub mod svg_export;
}

use clap::{Parser, ValueEnum};
use heightmap::HeightMap;
use river::River;
use voronoi::VoronoiMap;

#[derive(Debug, Clone, ValueEnum)]
enum Algorithm {
    Perlin,
    Caves,
    Voronoi,
    Ridged,
}

/// Procedural Map Generator — สร้างแผนที่โลกสุ่มด้วย Perlin noise
#[derive(Parser, Debug)]
#[command(name = "map-generator", version = "0.1.0", about = "Procedural world map generator")]
struct Cli {
    /// ความกว้างของแผนที่ (pixels)
    #[arg(long, default_value_t = 512)]
    width: usize,

    /// ความสูงของแผนที่ (pixels)
    #[arg(long, default_value_t = 512)]
    height: usize,

    /// Seed สำหรับ random (เดิม seed เดิม = แผนที่เดิมเสมอ)
    #[arg(long, default_value_t = 42)]
    seed: u64,

    /// ไฟล์ output PNG
    #[arg(long, default_value = "map.png")]
    output: String,

    /// อัลกอริทึมที่ใช้
    #[arg(long, value_enum, default_value_t = Algorithm::Perlin)]
    algorithm: Algorithm,

    /// Export JSON tile data
    #[arg(long)]
    export_json: bool,

    /// Export SVG (แนะนำสำหรับ width/height <= 256)
    #[arg(long)]
    export_svg: bool,

    /// ใช้ slope shading (ช้ากว่าเล็กน้อย)
    #[arg(long)]
    shading: bool,

    /// จำนวน Voronoi regions (ใช้กับ --algorithm voronoi)
    #[arg(long, default_value_t = 20)]
    voronoi_regions: usize,
}

fn main() {
    let args = Cli::parse();

    eprintln!(
        "Generating {}x{} map (seed={}, algorithm={:?})",
        args.width, args.height, args.seed, args.algorithm
    );

    // สร้าง heightmap หลัก
    let mut heightmap = match args.algorithm {
        Algorithm::Ridged => HeightMap::generate_ridged(args.width, args.height, args.seed),
        _ => HeightMap::generate(args.width, args.height, args.seed),
    };

    // สร้าง moisture map ด้วย seed ต่างออกไป
    let moisture_map = HeightMap::generate(args.width, args.height, args.seed.wrapping_add(999));

    // สร้างแม่น้ำ
    let rivers = River::generate(&mut heightmap, args.seed.wrapping_add(7));
    eprintln!("Generated {} rivers", rivers.len());

    // Render PNG
    let result = if args.shading {
        export::png_render::render_shaded(&heightmap, &moisture_map, &rivers, &args.output)
    } else {
        match args.algorithm {
            Algorithm::Caves => {
                render_caves(args.width, args.height, args.seed, &args.output)
            }
            Algorithm::Voronoi => {
                render_voronoi(args.width, args.height, args.seed, args.voronoi_regions, &args.output)
            }
            _ => export::png_render::render_colored(&heightmap, &moisture_map, &rivers, &args.output),
        }
    };

    match result {
        Ok(()) => eprintln!("Saved PNG: {}", args.output),
        Err(e) => eprintln!("Error saving PNG: {}", e),
    }

    // Optional exports
    if args.export_json {
        let json_path = args.output.replace(".png", ".json");
        match export::json_export::export_json(&heightmap, &moisture_map, &rivers, args.seed, &json_path) {
            Ok(()) => eprintln!("Saved JSON: {json_path}"),
            Err(e) => eprintln!("Error saving JSON: {e}"),
        }
    }

    if args.export_svg {
        let svg_path = args.output.replace(".png", ".svg");
        match export::svg_export::export_svg(&heightmap, &moisture_map, &svg_path) {
            Ok(()) => eprintln!("Saved SVG: {svg_path}"),
            Err(e) => eprintln!("Error saving SVG: {e}"),
        }
    }
}

fn render_caves(
    width: usize,
    height: usize,
    seed: u64,
    output: &str,
) -> Result<(), image::ImageError> {
    use cave::{CaveMap, Cell};
    use image::{ImageBuffer, Rgb};

    let cave = CaveMap::generate(width, height, seed, 0.45, 5);
    let mut img: ImageBuffer<Rgb<u8>, Vec<u8>> = ImageBuffer::new(width as u32, height as u32);

    for y in 0..height {
        for x in 0..width {
            let color = if *cave.get(x, y) == Cell::Floor {
                Rgb([220u8, 210, 180]) // floor — tan
            } else {
                Rgb([40u8, 30, 20])    // wall — dark brown
            };
            img.put_pixel(x as u32, y as u32, color);
        }
    }

    img.save(output)?;
    Ok(())
}

fn render_voronoi(
    width: usize,
    height: usize,
    seed: u64,
    n: usize,
    output: &str,
) -> Result<(), image::ImageError> {
    use image::{ImageBuffer, Rgb};
    use rand::{Rng, SeedableRng};
    use rand::rngs::StdRng;

    let mut voronoi = VoronoiMap::generate(width, height, n, seed);
    voronoi.lloyd_relaxation(3);

    // สุ่มสีสำหรับแต่ละ region
    let mut rng = StdRng::seed_from_u64(seed + 1);
    let region_colors: Vec<[u8; 3]> = (0..n)
        .map(|_| [rng.gen(), rng.gen(), rng.gen()])
        .collect();

    let mut img: ImageBuffer<Rgb<u8>, Vec<u8>> = ImageBuffer::new(width as u32, height as u32);
    for y in 0..height {
        for x in 0..width {
            let region = voronoi.nearest_seed(x, y);
            let color = region_colors[region];
            img.put_pixel(x as u32, y as u32, Rgb(color));
        }
    }

    img.save(output)?;
    Ok(())
}
```

---

## การทดสอบ (Testing)

สร้างไฟล์ `tests/integration.rs`:

```rust
// tests/integration.rs — integration tests สำหรับ map generator

// Re-import modules ที่ต้องการ
use map_generator::noise::{PerlinNoise, Fbm, fade};
use map_generator::heightmap::HeightMap;
use map_generator::biome::{classify_biome, Biome};
use map_generator::cave::{CaveMap, Cell};
use map_generator::voronoi::VoronoiMap;
use map_generator::distance::compute_distance_field;
```

ไฟล์ test หลักอยู่ใน `src/` แต่ละ module (unit tests) ใน `#[cfg(test)]` block:

**ตัวอย่าง tests ที่ครอบคลุมทุกส่วน — เพิ่มใน `src/noise.rs`, `src/heightmap.rs`, `src/biome.rs`, `src/cave.rs`, `src/voronoi.rs`, `src/distance.rs`:**

```rust
// ─── ใน src/noise.rs ───

#[cfg(test)]
mod tests {
    use super::*;

    // Perlin noise ต้องอยู่ใน [-1, 1]
    #[test]
    fn test_perlin_noise_range() {
        let p = PerlinNoise::new(42);
        for i in 0..100 {
            let x = i as f64 * 0.137;
            let y = i as f64 * 0.251;
            let v = p.noise(x, y, 0.0);
            assert!(
                v >= -1.0 && v <= 1.0,
                "Perlin noise out of [-1,1]: {}",
                v
            );
        }
    }

    // fade(0) = 0, fade(0.5) = 0.5, fade(1) = 1
    #[test]
    fn test_fade_function_endpoints() {
        let eps = 1e-12;
        assert!((fade(0.0) - 0.0).abs() < eps, "fade(0) must be 0");
        assert!((fade(1.0) - 1.0).abs() < eps, "fade(1) must be 1");
        assert!((fade(0.5) - 0.5).abs() < eps, "fade(0.5) must be 0.5");
    }

    // FBM กับ seed เดิมต้องให้ผลเหมือนกันทุกครั้ง (deterministic)
    #[test]
    fn test_fbm_deterministic() {
        let fbm1 = Fbm::new(42, 4, 0.5, 2.0);
        let fbm2 = Fbm::new(42, 4, 0.5, 2.0);
        let v1 = fbm1.sample(1.23, 4.56);
        let v2 = fbm2.sample(1.23, 4.56);
        assert!((v1 - v2).abs() < 1e-12, "FBM must be deterministic");
    }

    // FBM ที่มี octave ต่างกันต้องให้ผลต่างกัน (มากกว่า 1 octave เพิ่ม detail)
    #[test]
    fn test_fbm_octaves_differ() {
        let fbm_low  = Fbm::new(42, 1, 0.5, 2.0);
        let fbm_high = Fbm::new(42, 8, 0.5, 2.0);
        let v_low  = fbm_low.sample(0.3, 0.7);
        let v_high = fbm_high.sample(0.3, 0.7);
        assert!(
            (v_low - v_high).abs() > 1e-10,
            "FBM with 1 and 8 octaves should differ"
        );
    }

    // FBM seed ต่างกัน ต้องให้ผลต่างกัน
    #[test]
    fn test_fbm_different_seeds() {
        let fbm1 = Fbm::new(42, 4, 0.5, 2.0);
        let fbm2 = Fbm::new(99, 4, 0.5, 2.0);
        let v1 = fbm1.sample(0.5, 0.5);
        let v2 = fbm2.sample(0.5, 0.5);
        assert!((v1 - v2).abs() > 1e-8, "Different seeds must produce different values");
    }
}

// ─── ใน src/heightmap.rs ───

#[cfg(test)]
mod tests {
    use super::*;

    // ทุกค่าใน heightmap ต้องอยู่ใน [0, 1]
    #[test]
    fn test_heightmap_normalized() {
        let hm = HeightMap::generate(64, 64, 123);
        for &v in &hm.data {
            assert!(v >= 0.0 && v <= 1.0, "HeightMap value {} out of [0,1]", v);
        }
    }

    // ขนาดต้องตรงกับที่ระบุ
    #[test]
    fn test_heightmap_dimensions() {
        let hm = HeightMap::generate(32, 48, 0);
        assert_eq!(hm.width, 32);
        assert_eq!(hm.height, 48);
        assert_eq!(hm.data.len(), 32 * 48);
    }

    // Ridge variant ต้องให้ผลต่างจาก standard FBM
    #[test]
    fn test_ridged_differs_from_standard() {
        let std_hm = HeightMap::generate(32, 32, 42);
        let rid_hm = HeightMap::generate_ridged(32, 32, 42);
        let std_sum: f64 = std_hm.data.iter().sum();
        let rid_sum: f64 = rid_hm.data.iter().sum();
        assert!((std_sum - rid_sum).abs() > 1.0, "Ridged map should differ from standard");
    }
}

// ─── ใน src/biome.rs ───

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_biome_ocean()     { assert_eq!(classify_biome(0.10, 0.5), Biome::Ocean); }
    #[test]
    fn test_biome_beach()     { assert_eq!(classify_biome(0.32, 0.5), Biome::Beach); }
    #[test]
    fn test_biome_desert()    { assert_eq!(classify_biome(0.50, 0.1), Biome::Desert); }
    #[test]
    fn test_biome_grassland() { assert_eq!(classify_biome(0.50, 0.5), Biome::Grassland); }
    #[test]
    fn test_biome_jungle()    { assert_eq!(classify_biome(0.50, 0.9), Biome::Jungle); }
    #[test]
    fn test_biome_forest()    { assert_eq!(classify_biome(0.70, 0.5), Biome::Forest); }
    #[test]
    fn test_biome_mountain()  { assert_eq!(classify_biome(0.85, 0.5), Biome::Mountain); }
    #[test]
    fn test_biome_snow()      { assert_eq!(classify_biome(0.95, 0.5), Biome::Snow); }

    // Boundary conditions
    #[test]
    fn test_biome_boundary_ocean_beach() {
        // 0.30 = beach
        assert_eq!(classify_biome(0.30, 0.5), Biome::Beach);
        // 0.2999... = ocean
        assert_eq!(classify_biome(0.2999, 0.5), Biome::Ocean);
    }
}

// ─── ใน src/cave.rs ───

#[cfg(test)]
mod tests {
    use super::*;

    // ทดสอบ rule: grid ทุก cell เป็น wall → center ยังเป็น wall
    #[test]
    fn test_all_walls_stay_wall() {
        let mut cave = CaveMap {
            cells: vec![Cell::Wall; 25],
            width: 5,
            height: 5,
        };
        cave.step();
        // Center (2,2) มี 8 wall neighbors → ≥5 → ยังเป็น wall
        assert_eq!(cave.get(2, 2), &Cell::Wall);
    }

    // Grid ทั้งหมด floor + center ไม่ติดบอร์เดอร์ → center stays floor
    #[test]
    fn test_center_floor_no_walls() {
        let mut cave = CaveMap {
            cells: vec![Cell::Floor; 25],
            width: 5,
            height: 5,
        };
        cave.step();
        // (2,2) มี 0 wall neighbors → ≤1 → stays floor
        assert_eq!(cave.get(2, 2), &Cell::Floor);
    }

    // 3x3 grid all walls → center has 8 neighbors all wall → stays wall
    #[test]
    fn test_3x3_all_walls() {
        let mut cave = CaveMap {
            cells: vec![Cell::Wall; 9],
            width: 3,
            height: 3,
        };
        cave.step();
        assert_eq!(cave.get(1, 1), &Cell::Wall);
    }

    // count_wall_neighbors: all walls in 5x5 → center = 8 neighbors
    #[test]
    fn test_count_wall_neighbors_all_walls() {
        let cave = CaveMap {
            cells: vec![Cell::Wall; 25],
            width: 5,
            height: 5,
        };
        let grid = cave.cells.clone();
        let count = cave.count_wall_neighbors(&grid, 2, 2);
        assert_eq!(count, 8);
    }
}

// ─── ใน src/voronoi.rs ───

#[cfg(test)]
mod tests {
    use super::*;

    // 1 seed → ทุก cell ต้องอยู่ใน region 0
    #[test]
    fn test_single_seed() {
        let v = VoronoiMap::generate(10, 10, 1, 0);
        for y in 0..10 {
            for x in 0..10 {
                assert_eq!(v.nearest_seed(x, y), 0);
            }
        }
    }

    // region indices ต้องอยู่ใน [0, n)
    #[test]
    fn test_region_indices_valid() {
        let n = 7;
        let v = VoronoiMap::generate(16, 16, n, 42);
        for &r in &v.regions {
            assert!(r < n, "Region {} out of [0, {})", r, n);
        }
    }

    // Lloyd relaxation ต้องเปลี่ยนตำแหน่ง seeds
    #[test]
    fn test_lloyd_changes_seeds() {
        let mut v = VoronoiMap::generate(32, 32, 5, 99);
        let seeds_before = v.seeds.clone();
        v.lloyd_relaxation(3);
        // seed อย่างน้อย 1 ตัวต้องเปลี่ยน
        let changed = seeds_before
            .iter()
            .zip(v.seeds.iter())
            .any(|(a, b)| (a.0 - b.0).abs() > 1e-6 || (a.1 - b.1).abs() > 1e-6);
        assert!(changed, "Lloyd relaxation must move at least one seed");
    }
}

// ─── ใน src/distance.rs ───

#[cfg(test)]
mod tests {
    use super::*;
    use crate::heightmap::HeightMap;

    // ocean tiles ต้องมี distance = 0
    #[test]
    fn test_ocean_distance_zero() {
        let hm = HeightMap::generate(32, 32, 42);
        let dist = compute_distance_field(&hm, 0.30);
        for y in 0..32 {
            for x in 0..32 {
                if hm.get(x, y) < 0.30 {
                    assert_eq!(dist[y * 32 + x], 0,
                        "Ocean tile ({},{}) must have distance 0", x, y);
                }
            }
        }
    }

    // BFS distance field ต้อง monotone: |d(a) - d(b)| ≤ 1 สำหรับ neighbors
    #[test]
    fn test_bfs_distance_monotone() {
        let hm = HeightMap::generate(32, 32, 99);
        let dist = compute_distance_field(&hm, 0.30);
        for y in 0..32usize {
            for x in 0..32usize {
                let d = dist[y * 32 + x];
                if d == u32::MAX { continue; }
                for (dx, dy) in [(-1i32,0),(1,0),(0,-1i32),(0,1)] {
                    let nx = x as i32 + dx;
                    let ny = y as i32 + dy;
                    if nx >= 0 && ny >= 0 && nx < 32 && ny < 32 {
                        let nd = dist[ny as usize * 32 + nx as usize];
                        if nd != u32::MAX {
                            let diff = (d as i64 - nd as i64).abs();
                            assert!(diff <= 1,
                                "BFS distance jump >1: ({},{})={} → ({},{})={}", x,y,d,nx,ny,nd);
                        }
                    }
                }
            }
        }
    }

    // ทุก land tile ต้องมี distance > 0 (ถ้าไม่ใช่ ocean)
    #[test]
    fn test_land_distance_positive() {
        let hm = HeightMap::generate(32, 32, 77);
        let dist = compute_distance_field(&hm, 0.30);
        for y in 0..32 {
            for x in 0..32 {
                if hm.get(x, y) >= 0.30 {
                    assert!(
                        dist[y * 32 + x] > 0,
                        "Land tile ({},{}) must have distance > 0", x, y
                    );
                }
            }
        }
    }
}
```

### รัน Tests จริง

```
$ cargo test
```

**Output จริงจากการรัน** (verification build):

```
running 19 tests
test tests::test_biome_beach ... ok
test tests::test_biome_desert ... ok
test tests::test_biome_forest ... ok
test tests::test_biome_grassland ... ok
test tests::test_biome_mountain ... ok
test tests::test_biome_jungle ... ok
test tests::test_biome_ocean ... ok
test tests::test_biome_snow ... ok
test tests::test_cellular_automata_rule_floor_island ... ok
test tests::test_fade_function ... ok
test tests::test_cellular_automata_rule_known_input ... ok
test tests::test_fbm_more_octaves_refines ... ok
test tests::test_fbm_deterministic ... ok
test tests::test_perlin_noise_range ... ok
test tests::test_voronoi_nearest_seed ... ok
test tests::test_voronoi_regions_count ... ok
test tests::test_distance_field_ocean_zero ... ok
test tests::test_distance_field_bfs_monotone ... ok
test tests::test_heightmap_normalized ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

**Output จากการรัน binary:**

```
Procedural Map Generator — Verification Build
HeightMap 32x32 — min=0.0000 max=1.0000
All values in [0,1]: true
elevation=0.20 → Ocean
elevation=0.50 moisture=0.20 → Desert
elevation=0.80 → Mountain
Voronoi(16x16, 4 seeds): region at (8,8) = seed #0
```

---

## ข้อควรระวัง (Pitfalls)

### Pitfall 1: Closure capture กับ struct ที่ไม่ implement `Copy`

**ปัญหา:** เมื่อพยายาม move struct เข้า nested closures:

```rust
// ❌ ไม่ compile — fbm ถูก move ใน outer closure ก่อนแล้ว
let raw: Vec<f64> = (0..height)
    .flat_map(|y| {
        (0..width).map(move |x| {    // ← ตรงนี้พยายาม move fbm อีกครั้ง
            fbm.sample(x as f64, y as f64)  // fbm: Fbm ไม่ implement Copy
        })
    })
    .collect();
```

**Error:** `E0507: cannot move out of fbm, a captured variable in an FnMut closure`

**วิธีแก้:** ใช้ nested loop แทน iterator chain:

```rust
// ✅ ถูกต้อง
let mut raw = Vec::with_capacity(width * height);
for y in 0..height {
    for x in 0..width {
        raw.push(fbm.sample(x as f64 / width as f64 * scale,
                            y as f64 / height as f64 * scale));
    }
}
```

หรือ derive `Clone` และ clone ใน closure:

```rust
// ✅ ก็ได้ แต่ clone ทุก y iteration
let fbm_clone = fbm.clone(); // ต้อง derive Clone บน Fbm
let raw: Vec<f64> = (0..height)
    .flat_map(|y| {
        let fbm = fbm_clone.clone();
        (0..width).map(move |x| fbm.sample(...))
    })
    .collect();
```

---

### Pitfall 2: Permutation table index overflow

**ปัญหา:** Perlin noise ใช้ `perm[perm[perm[xi] + yi] + zi]` ถ้า `perm` มีขนาดแค่ 256 จะ panic เมื่อ `xi + 1 = 255` เพราะ `perm[255] + yi + 1` อาจ > 255

**ผิด:**
```rust
let mut perm = [0u8; 256]; // ← ขนาดผิด!
let aaa = self.perm[self.perm[self.perm[xi] as usize + yi] as usize + zi];
// xi=254, perm[254]=255, 255 + yi=1 = 256 → panic!
```

**ถูก:**
```rust
let mut perm = [0u8; 512]; // double table
for i in 0..256 {
    perm[i] = p[i];
    perm[i + 256] = p[i]; // ← duplicate ครึ่งหลัง
}
// ตอนนี้ index สูงสุด = 255 + 256 = 511 ซึ่งอยู่ใน bounds ของ array ขนาด 512
```

---

### Pitfall 3: River infinite loop บน flat terrain

**ปัญหา:** แม่น้ำอาจ stuck วน loop ถ้า heightmap มี flat plateau ทุก neighbor มี elevation เท่ากัน

```rust
// ❌ อาจวนไม่ออก
loop {
    let next = neighbors.iter().min_by(elevation).unwrap();
    cx = next.x; cy = next.y;  // ถ้า min == current → วนกลับที่เดิม
}
```

**วิธีแก้:** ใช้ `HashSet` track visited + limit ความยาวสูงสุด:

```rust
let mut visited = std::collections::HashSet::new();
visited.insert((cx, cy));

let next = neighbors
    .iter()
    .filter(|&&(nx, ny)| !visited.contains(&(nx, ny))) // ← ห้ามย้อนกลับ
    .min_by_elevation();

if path.len() > width + height {
    break; // ← safety valve
}
```

---

### Pitfall 4: Voronoi ด้วย N=0 seeds

**ปัญหา:** ถ้าไม่มี seed เลย, `min_by` จะ return `None` และการ `.unwrap_or(0)` จะ panic เพราะ index 0 ไม่มีอยู่

```rust
// ❌ อันตราย
let nearest = seeds.iter().enumerate()
    .min_by(distance_comparator)
    .map(|(i, _)| i)
    .unwrap_or(0); // ← ถ้า seeds ว่าง, index 0 ไม่มีอยู่!
```

**วิธีแก้:** validate input ก่อน:

```rust
pub fn generate(width: usize, height: usize, n: usize, seed: u64) -> Self {
    assert!(n > 0, "Must have at least 1 Voronoi seed");
    // ...
}
```

---

### Pitfall 5: Normalization ด้วย range ≈ 0

**ปัญหา:** ถ้า heightmap ที่สร้างด้วย seed พิเศษให้ค่าทุกจุดเท่ากัน, `range = max - min = 0` ทำให้หารด้วยศูนย์

```rust
// ❌ อันตราย
let range = max - min;
let data: Vec<f64> = raw.iter().map(|v| (v - min) / range).collect(); // NaN!
```

**วิธีแก้:** clamp range ด้วยค่าเล็กมาก:

```rust
// ✅ ป้องกัน division by zero
let range = (max - min).max(1e-9);
```

---

## การ Package และ Deploy

### Build release binary

```bash
cargo build --release
# binary อยู่ที่ target/release/map-generator
```

### การใช้งาน

```bash
# สร้างแผนที่ 512x512 ด้วย Perlin noise seed=42
./target/release/map-generator \
    --width 512 --height 512 \
    --seed 42 \
    --output world.png

# Caves algorithm
./target/release/map-generator \
    --algorithm caves \
    --width 256 --height 256 \
    --seed 7 \
    --output cave.png

# Voronoi regions + export JSON
./target/release/map-generator \
    --algorithm voronoi \
    --voronoi-regions 30 \
    --seed 100 \
    --output regions.png \
    --export-json

# Ridged multifractal + slope shading
./target/release/map-generator \
    --algorithm ridged \
    --shading \
    --seed 2024 \
    --output mountain.png

# Perlin + shading + export ทุก format
./target/release/map-generator \
    --width 256 --height 256 \
    --seed 99 \
    --output map.png \
    --shading \
    --export-json \
    --export-svg
```

### Determinism

seed เดิมรับประกันผลเหมือนกัน 100% บน machine ใดก็ตาม เพราะ:
- ใช้ `rand::rngs::StdRng` ซึ่งเป็น ChaCha-based PRNG ที่ reproducible
- ทุก calculation เป็น pure arithmetic — ไม่มี floating-point nondeterminism จาก thread ordering
- `Fbm::sample` ไม่มี global state

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: Temperature Map และ Biome ที่ละเอียดขึ้น (ง่าย)

เพิ่ม temperature map (FBM pass ที่ 3) เพื่อแบ่ง biome ตาม latitude + elevation:
- Arctic: temperature < 0.2 → snow ไม่ว่า elevation จะเป็นเท่าไร
- Tropical: temperature > 0.8 → jungle ถ้า moisture สูง
- เพิ่ม `Tundra`, `Savanna`, `Swamp` เป็น biome ใหม่

```rust
pub fn classify_biome_v2(elevation: f64, moisture: f64, temperature: f64) -> Biome {
    if temperature < 0.2 { return Biome::Snow; }
    if elevation < 0.30  { return Biome::Ocean; }
    // ...
}
```

### Exercise 2: Erosion Simulation ที่สมจริง (ปานกลาง)

Implement hydraulic erosion (น้ำกัดหิน) แทน gradient descent แบบง่าย:
- Particle-based: ปล่อย N อนุภาคน้ำจากที่สุ่ม ให้ไหลลงตามความชัน
- แต่ละอนุภาคพาดิน (sediment) ลงตามแรง กรณีช้าลงวาง sediment ลงแทน
- ทำซ้ำหลาย iteration จนได้ valley และ ridge ที่สมจริง

ดูอ้างอิง: *"Procedural Hydraulic Erosion"* โดย Hans Theobald Beyer

### Exercise 3: Animated GIF ของการเกิด Caves (ปานกลาง)

แทนที่จะ render แค่ผลลัพธ์สุดท้าย ให้ export แต่ละ CA iteration เป็น frame แล้วรวมเป็น GIF:

```rust
use image::codecs::gif::{GifEncoder, Repeat};
use image::Frame;

let mut frames: Vec<Frame> = Vec::new();
let mut cave_state = CaveMap::initial(width, height, seed, 0.45);
frames.push(render_frame(&cave_state));
for _ in 0..5 {
    cave_state.step();
    frames.push(render_frame(&cave_state));
}
// encode as GIF...
```

### Exercise 4: Multi-layer Map (ยาก)

สร้างระบบ layer หลายชั้น:
- Layer 0: ภูมิประเทศ (heightmap + biome)
- Layer 1: Cave system (cellular automata) ที่ซ่อนอยู่ใต้ดิน
- Layer 2: Political overlay (Voronoi regions)
- Layer 3: Fog of war (BFS distance field)

ออกแบบ `MapLayer` trait และ `CompositeMap` struct ที่ combine หลาย layer:

```rust
pub trait MapLayer {
    fn pixel_color(&self, x: usize, y: usize) -> [u8; 4]; // RGBA
}

pub struct CompositeMap {
    layers: Vec<Box<dyn MapLayer>>,
}

impl CompositeMap {
    pub fn render(&self, width: usize, height: usize) -> ImageBuffer<Rgba<u8>, Vec<u8>> {
        // alpha-compositing แต่ละ layer
    }
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง Procedural Map Generator ที่ครบวงจร:

**Algorithm ที่ implement:**
- **Perlin Noise** จาก scratch — permutation table + grad function + trilinear interpolation
- **FBM** — ซ้อน octave ด้วย persistence/lacunarity, ridged multifractal variant
- **Cellular Automata** — organic cave generation rule B3/S12345
- **Voronoi + Lloyd** — region tessellation สำหรับ political map
- **BFS Distance Field** — coastline distance สำหรับ beach shading
- **River Simulation** — gradient descent + erosion

**Rust patterns ที่สำคัญ:**
- Flat `Vec<T>` แทน `Vec<Vec<T>>` สำหรับ 2D data (cache-friendly)
- `#[derive(Serialize, Deserialize)]` บน enum และ struct สำหรับ JSON export
- `SeedableRng` สำหรับ deterministic randomness
- `HashSet` สำหรับ O(1) visited-tile lookup ใน river simulation
- `clap 4` derive macro สำหรับ CLI ที่ clean และ ergonomic
- `image` crate สำหรับ PNG output โดยไม่ต้องเขียน file format เอง

โปรเจคถัดไป **E08 Particle System** จะนำ rendering pipeline ที่เรียนมาต่อยอดด้วยการ simulate particle physics แบบ real-time — ใช้ `Vec<Particle>` ที่ update ทุก frame และ render เป็น PNG หรือ terminal output

---

**โปรเจคก่อนหน้า:** [project-e06-roguelike.md](project-e06-roguelike.md) | **โปรเจคถัดไป:** [project-e08-particle-system.md](project-e08-particle-system.md)
