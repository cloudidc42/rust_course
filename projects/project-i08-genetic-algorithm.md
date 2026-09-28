# Project I08: Genetic Algorithm

> โมดูล: I — Machine Learning & AI | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

**Genetic Algorithm (GA)** คืออัลกอริทึมค้นหาและ optimization ที่ได้รับแรงบันดาลใจจากกระบวนการวิวัฒนาการทางธรรมชาติของดาร์วิน ตัวอัลกอริทึมจำลองกลไกทางชีวววิทยา ได้แก่ การคัดเลือกตามความเหมาะสม (selection), การผสมพันธุ์ (crossover), และการกลายพันธุ์ (mutation) เพื่อค้นหาคำตอบที่ดีที่สุดในปัญหาที่มีพื้นที่ค้นหาขนาดใหญ่

ในโปรเจคนี้เราจะสร้างไลบรารี Genetic Algorithm ที่ครบวงจรด้วย Rust ตั้งแต่ระดับ genome encoding พื้นฐาน ไปจนถึงการแก้ปัญหา Travelling Salesman Problem (TSP) และบทนำของ Multi-objective Optimization ด้วย NSGA-II

**Use cases จริงในโลก production:**
- **Engineering design** — ออกแบบโครงสร้างอาคาร, ปีกเครื่องบิน, หรือวงจรไฟฟ้าที่มีหลายพารามิเตอร์
- **Route optimization** — ปัญหา TSP ในการส่งพัสดุ, การจัดเส้นทางรถขนส่ง
- **Neural architecture search** — ค้นหา hyperparameters หรือโครงสร้างของ neural network
- **Game AI** — evolve กลยุทธ์ใน game AI โดยไม่ต้องเขียน heuristic เอง
- **Portfolio optimization** — เลือกน้ำหนักการลงทุนในตราสารหลักทรัพย์หลายตัว
- **Scheduling** — จัดตารางการผลิต, การมอบหมายงาน (job-shop scheduling)
- **Feature selection** — เลือก subset ของ feature ที่ดีที่สุดสำหรับ machine learning model

**Learning value:**
โปรเจคนี้สาธิตทักษะ Rust หลากหลาย ได้แก่ การออกแบบ trait ที่ยืดหยุ่นสำหรับ generic genome types, การใช้ `rand` crate อย่างถูกต้องกับ seeded RNG เพื่อ reproducibility, การ serialize configuration ด้วย `serde_json`, และการคิดเชิง algorithm design เพื่อแก้ปัญหา NP-hard

---

## สิ่งที่จะได้เรียนรู้

- **Trait-based abstraction** — ออกแบบ `Individual` trait ที่ให้ genome หลาย type (binary, permutation) ทำงานร่วมกันใน GA framework เดียวกัน
- **Generic types + associated types** — ใช้ `type Genome` ใน trait เพื่อหลีกเลี่ยง type erasure
- **Evolutionary operators** — implement selection (roulette wheel, tournament, rank), crossover (single-point, two-point, uniform, OX), mutation (bit-flip, swap, inversion) ครบชุด
- **Stochastic algorithms** — จัดการ random number generation ด้วย `StdRng::seed_from_u64` เพื่อให้ผลลัพธ์ทำซ้ำได้
- **Permutation encoding** — แทน TSP route ด้วย `Vec<usize>` และ implement Order Crossover (OX) ที่รักษา constraint ของ permutation
- **Multi-objective optimization** — เข้าใจ Pareto dominance, fast non-dominated sort, และ crowding distance สำหรับ NSGA-II
- **Convergence detection** — วัดความหลากหลายของ population ด้วย Hamming distance และตรวจจับ fitness plateau
- **Serialization** — serialize `GaConfig` ด้วย `serde_json` เพื่อ save/load การตั้งค่า

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`), iterators, closures, `map`/`filter`/`fold`
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics, trait bounds
- **Part 41–50**: Testing (`#[test]`, `#[cfg(test)]`), modules, `use` statements
- **Part 51–60**: Lifetime basics, floating-point arithmetic, `f64` methods
- **Part 61–70**: External crates (`rand`, `serde`), `Cargo.toml` features
- **Part 86–90**: Performance fundamentals — เมื่อใดควร clone เมื่อใดควร borrow

---

## โครงสร้างโปรเจค (Project Layout)

```
genetic_algo/
├── src/
│   └── lib.rs          ← genome types, operators, GA loops, tests
├── Cargo.toml
└── README.md
```

โปรเจคนี้เป็น library crate เพื่อให้ binary หรือโปรเจคอื่น `use genetic_algo::*` ได้โดยตรง โมดูลหลักทั้งหมดอยู่ใน `lib.rs` แบ่งเป็นส่วนๆ ตามหน้าที่ ได้แก่ trait definitions, genome types, selection operators, crossover operators, mutation operators, GA loops, NSGA-II helpers, และ convergence utilities

---

## การออกแบบ (Architecture & Design)

### Data Flow ของ Generational GA

```
สร้าง population เริ่มต้น (random)
          │
          ▼
┌─────────────────────────────────────────┐
│  วนซ้ำ max_generations รอบ             │
│                                         │
│  1. คำนวณ fitness ของทุก individual    │
│  2. Selection — เลือก parents          │
│  3. Crossover — สร้าง offspring        │
│  4. Mutation — กลายพันธุ์ offspring   │
│  5. Elitism — เก็บ best individuals   │
│  6. สร้าง next generation             │
│  7. ตรวจสอบ convergence              │
└─────────────────────────────────────────┘
          │ (หยุดเมื่อ converge หรือครบ gen)
          ▼
   GaResult { best_genome, best_fitness, ... }
```

### Design Decisions

**ทำไมถึงใช้ trait แทน enum สำหรับ genome type?**

Rust enum จะ work ได้ดีถ้า genome types ถูกกำหนดล่วงหน้าและจำนวนน้อย แต่ผู้ใช้อาจต้องการ define genome type ของตัวเอง (เช่น real-valued vector, tree structure) ดังนั้นการใช้ trait ทำให้ extensible มากกว่า

**ทำไม `BinaryIndividual` และ `PermIndividual` ถึงเก็บ `fitness` ไว้ใน struct?**

เพราะการคำนวณ fitness อาจ expensive (เช่น simulate route) เราจึง cache ค่าไว้แทนที่จะคำนวณซ้ำทุกครั้งที่ access การ update fitness จะเกิดขึ้นเฉพาะเมื่อ genome เปลี่ยนแปลง (หลัง mutation หรือ crossover)

**ทำไม TSP fitness ถึงเป็น `1.0 / distance` แทนที่จะเป็น negative distance?**

GA โดยทั่วไปใช้ maximization (fitness ยิ่งสูงยิ่งดี) ดังนั้น fitness = 1/distance จะแปลง minimization problem ให้เป็น maximization โดย individual ที่ route สั้นจะมี fitness สูง — แนวทางอื่นคือใช้ `max_dist - distance` ซึ่งก็ได้ผลเช่นกัน

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Individual Trait และ Binary Genome

เริ่มจากการสร้าง abstraction หลักของ Genetic Algorithm นั่นคือ `Individual` trait ที่กำหนดว่า genome type ใดก็ตามต้องมีความสามารถอะไรบ้างเพื่อเข้าร่วมใน GA framework

สร้างไฟล์ `Cargo.toml`:

```toml
[package]
name = "genetic_algo"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

สร้าง `src/lib.rs` ส่วนแรก:

```rust
use rand::prelude::*;
use serde::{Deserialize, Serialize};

// Individual trait: กำหนด interface ของ genome ทุกชนิด
pub trait Individual: Clone {
    type Genome: Clone;

    fn genome(&self) -> &Self::Genome;
    fn fitness(&self) -> f64;
}

// BinaryIndividual: genome คือ Vec<u8> ที่มีแค่ค่า 0 หรือ 1
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct BinaryIndividual {
    pub genome: Vec<u8>,
    pub fitness: f64,
}

impl BinaryIndividual {
    /// สร้าง individual แบบสุ่มด้วย RNG ที่กำหนดมา
    pub fn random(length: usize, rng: &mut impl Rng) -> Self {
        let genome: Vec<u8> = (0..length).map(|_| rng.gen_range(0..=1)).collect();
        let fitness = Self::evaluate(&genome);
        BinaryIndividual { genome, fitness }
    }

    /// One-max fitness function: ผลรวมของบิต (เป้าหมาย = all ones)
    pub fn evaluate(genome: &[u8]) -> f64 {
        genome.iter().map(|&b| b as f64).sum()
    }
}

impl Individual for BinaryIndividual {
    type Genome = Vec<u8>;
    fn genome(&self) -> &Self::Genome { &self.genome }
    fn fitness(&self) -> f64 { self.fitness }
}
```

**แนวคิดสำคัญ**: `type Genome: Clone` คือ **associated type** ที่ช่วยให้ compiler รู้ว่า genome ของ individual แต่ละชนิดคืออะไร โดยไม่ต้องใช้ type parameter เพิ่มเติม การ implement `Individual for BinaryIndividual` จะ bind `Genome = Vec<u8>` ให้กับ type นั้นโดยเฉพาะ

ต่างจาก generic parameter `<G: Clone>` ตรงที่ associated type ผูกกับ impl เดียวต่อ type — ป้องกัน ambiguity เมื่อใช้ร่วมกับ trait objects

---

### ขั้นที่ 2: Population Initialization

Population คือ collection ของ individual ที่จะวิวัฒน์ร่วมกัน การ initialize ที่ดีควรให้ความหลากหลาย (diversity) เพียงพอเพื่อไม่ให้ algorithm ติดอยู่ใน local optimum ตั้งแต่ต้น

```rust
/// สร้าง binary population แบบสุ่ม
pub fn init_binary_population(
    size: usize,
    genome_length: usize,
    rng: &mut impl Rng,
) -> Vec<BinaryIndividual> {
    (0..size)
        .map(|_| BinaryIndividual::random(genome_length, rng))
        .collect()
}

/// PermIndividual: genome คือ Vec<usize> ที่เป็น permutation (ใช้สำหรับ TSP)
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct PermIndividual {
    pub genome: Vec<usize>,
    pub fitness: f64,
}

impl PermIndividual {
    /// สร้าง permutation สุ่ม: shuffle [0, 1, ..., n-1]
    pub fn random(n: usize, rng: &mut impl Rng, cities: &[(f64, f64)]) -> Self {
        let mut genome: Vec<usize> = (0..n).collect();
        genome.shuffle(rng);      // <-- rand::seq::SliceRandom::shuffle
        let fitness = Self::route_fitness(&genome, cities);
        PermIndividual { genome, fitness }
    }

    /// fitness = 1 / total_route_distance
    pub fn route_fitness(genome: &[usize], cities: &[(f64, f64)]) -> f64 {
        let dist = route_distance(genome, cities);
        if dist == 0.0 { f64::MAX } else { 1.0 / dist }
    }
}

/// คำนวณระยะทางรวมของ route (รวม leg กลับจุดเริ่มต้น)
pub fn route_distance(genome: &[usize], cities: &[(f64, f64)]) -> f64 {
    let n = genome.len();
    if n == 0 { return 0.0; }
    (0..n)
        .map(|i| euclidean_distance(cities[genome[i]], cities[genome[(i + 1) % n]]))
        .sum()
}

pub fn euclidean_distance(a: (f64, f64), b: (f64, f64)) -> f64 {
    let dx = a.0 - b.0;
    let dy = a.1 - b.1;
    (dx * dx + dy * dy).sqrt()
}
```

**หมายเหตุ**: `genome.shuffle(rng)` ใช้ Fisher-Yates shuffle ซึ่ง guarantee ว่าทุก permutation มีโอกาสออกมาเท่ากัน — ต้อง `use rand::seq::SliceRandom` หรือ `use rand::prelude::*` เพื่อให้ method `shuffle` ถูก bring into scope

**ทำไม binary population สำคัญ?** — binary string เป็น genome encoding ที่ง่ายที่สุดและ well-studied มากที่สุดในทฤษฎี GA ดั้งเดิมของ John Holland (1975) ทุก problem สามารถ encode เป็น binary ได้ แต่ permutation encoding มักดีกว่าสำหรับ combinatorial problems อย่าง TSP

---

### ขั้นที่ 3: Selection Operators

Selection คือกระบวนการเลือก parents เพื่อสร้าง offspring ในรุ่นถัดไป หลักการคือ individual ที่มี fitness สูงกว่าควรมีโอกาสถูกเลือกมากกว่า แต่ไม่ใช่ว่าเฉพาะที่ดีที่สุดเท่านั้น (ไม่งั้น diversity จะหายเร็วเกินไป)

#### 3.1 Roulette Wheel Selection (Fitness-Proportionate)

แนวคิด: โอกาสถูกเลือก = fitness_i / sum(fitness ทั้งหมด) — คล้ายกงล้อที่แต่ละช่องกว้างตามสัดส่วน fitness

```rust
pub fn roulette_select<'a>(
    population: &'a [BinaryIndividual],
    rng: &mut impl Rng,
) -> &'a BinaryIndividual {
    let total: f64 = population.iter().map(|ind| ind.fitness).sum();
    let mut pick = rng.gen::<f64>() * total;  // สุ่มจุดบนกงล้อ
    for ind in population {
        pick -= ind.fitness;
        if pick <= 0.0 {
            return ind;
        }
    }
    population.last().unwrap() // fallback (floating-point rounding)
}
```

**ข้อเสีย**: เมื่อ individual หนึ่งมี fitness สูงมากกว่าคนอื่นมาก (super-individual) จะครอง selection pressure เกือบทั้งหมด ทำให้ diversity พังเร็ว

#### 3.2 Tournament Selection

เลือก k individuals สุ่มแล้วเอาตัวที่ fitness ดีที่สุดจาก k ตัวนั้น ค่า k (tournament size) ควบคุม selection pressure: k ใหญ่ = pressure สูง = converge เร็ว แต่ diversity น้อย

```rust
pub fn tournament_select<'a>(
    population: &'a [BinaryIndividual],
    k: usize,
    rng: &mut impl Rng,
) -> &'a BinaryIndividual {
    (0..k)
        .map(|_| &population[rng.gen_range(0..population.len())])
        .max_by(|a, b| a.fitness.partial_cmp(&b.fitness).unwrap())
        .unwrap()
}
```

เหตุผลที่ใช้ `.partial_cmp().unwrap()` แทน `.total_cmp()`: `f64::NaN` ไม่ควรปรากฏใน fitness value ที่ valid — ถ้า fitness อาจเป็น NaN ควรตรวจสอบก่อน

#### 3.3 Rank Selection

เรียง population ตาม fitness แล้วกำหนด rank 1 (แย่สุด) ถึง n (ดีสุด) โอกาสถูกเลือก = rank / sum(ranks)

```rust
pub fn rank_select<'a>(
    population: &'a [BinaryIndividual],
    rng: &mut impl Rng,
) -> &'a BinaryIndividual {
    let n = population.len();
    let mut indexed: Vec<(usize, f64)> = population
        .iter()
        .enumerate()
        .map(|(i, ind)| (i, ind.fitness))
        .collect();
    indexed.sort_by(|a, b| a.1.partial_cmp(&b.1).unwrap());

    let total = (n * (n + 1) / 2) as f64;
    let mut pick = rng.gen::<f64>() * total;
    for (rank, (idx, _)) in indexed.iter().enumerate() {
        pick -= (rank + 1) as f64;
        if pick <= 0.0 {
            return &population[*idx];
        }
    }
    &population[indexed.last().unwrap().0]
}
```

**เปรียบเทียบ 3 selection methods:**

| วิธี | Selection Pressure | ความ robust | ความซับซ้อน |
|------|-------------------|-------------|------------|
| Roulette | ผันแปรตาม fitness ratio | ต่ำ (super-individual ครอง) | O(n) |
| Tournament | ควบคุมได้ด้วย k | สูง | O(k) |
| Rank | ปานกลาง-สม่ำเสมอ | กลาง | O(n log n) |

---

### ขั้นที่ 4: Crossover Operators

Crossover (หรือ recombination) เป็นกระบวนการรวม genetic material จาก parents สองตัวเพื่อสร้าง offspring ที่อาจดีกว่าทั้งคู่

#### 4.1 Single-Point Crossover

เลือกจุดตัด 1 จุด แล้วสลับส่วนหลัง:

```
parent1: [1 0 1 1 | 0 0 1 0]
parent2: [0 1 0 0 | 1 1 0 1]
              cut=4 ↑
child1:  [1 0 1 1 | 1 1 0 1]
child2:  [0 1 0 0 | 0 0 1 0]
```

```rust
impl BinaryIndividual {
    pub fn crossover_single_point(
        parent1: &Self,
        parent2: &Self,
        rng: &mut impl Rng,
    ) -> (Self, Self) {
        let len = parent1.genome.len();
        let point = rng.gen_range(1..len);  // ≥1 เพื่อให้ได้ exchange จริงๆ
        let mut g1 = parent1.genome[..point].to_vec();
        g1.extend_from_slice(&parent2.genome[point..]);
        let mut g2 = parent2.genome[..point].to_vec();
        g2.extend_from_slice(&parent1.genome[point..]);
        let f1 = Self::evaluate(&g1);
        let f2 = Self::evaluate(&g2);
        (
            BinaryIndividual { genome: g1, fitness: f1 },
            BinaryIndividual { genome: g2, fitness: f2 },
        )
    }
}
```

#### 4.2 Two-Point Crossover

เลือก 2 จุดตัด แล้วสลับส่วนกลาง — ลด positional bias ของ single-point:

```
parent1: [1 0 | 1 1 0 | 0 1 0]
parent2: [0 1 | 0 0 1 | 1 0 1]
              p=2,q=5 ↑   ↑
child1:  [1 0 | 0 0 1 | 0 1 0]
child2:  [0 1 | 1 1 0 | 1 0 1]
```

```rust
pub fn crossover_two_point(
    parent1: &Self,
    parent2: &Self,
    rng: &mut impl Rng,
) -> (Self, Self) {
    let len = parent1.genome.len();
    let mut pts = [rng.gen_range(1..len), rng.gen_range(1..len)];
    pts.sort_unstable();  // รับประกัน pts[0] <= pts[1]
    let (p, q) = (pts[0], pts[1]);
    let mut g1 = parent1.genome.clone();
    let mut g2 = parent2.genome.clone();
    // สลับเฉพาะ segment [p..q]
    g1[p..q].clone_from_slice(&parent2.genome[p..q]);
    g2[p..q].clone_from_slice(&parent1.genome[p..q]);
    let f1 = Self::evaluate(&g1);
    let f2 = Self::evaluate(&g2);
    (
        BinaryIndividual { genome: g1, fitness: f1 },
        BinaryIndividual { genome: g2, fitness: f2 },
    )
}
```

#### 4.3 Uniform Crossover

แต่ละตำแหน่งสุ่มอิสระว่าจะเอาจาก parent1 หรือ parent2 — ให้ diversity สูงสุด เหมาะกับปัญหาที่ epistasis ต่ำ (gene แต่ละตัวทำงานค่อนข้างอิสระ)

```rust
pub fn crossover_uniform(
    parent1: &Self,
    parent2: &Self,
    rng: &mut impl Rng,
) -> (Self, Self) {
    let len = parent1.genome.len();
    let mut g1 = Vec::with_capacity(len);
    let mut g2 = Vec::with_capacity(len);
    for i in 0..len {
        if rng.gen::<bool>() {
            g1.push(parent1.genome[i]);
            g2.push(parent2.genome[i]);
        } else {
            g1.push(parent2.genome[i]);
            g2.push(parent1.genome[i]);
        }
    }
    let f1 = Self::evaluate(&g1);
    let f2 = Self::evaluate(&g2);
    (
        BinaryIndividual { genome: g1, fitness: f1 },
        BinaryIndividual { genome: g2, fitness: f2 },
    )
}
```

#### 4.4 Order Crossover (OX) สำหรับ Permutation

สำหรับ TSP เราไม่สามารถใช้ binary crossover ได้เพราะ child จะไม่ใช่ valid permutation (เมือง A อาจปรากฏสองครั้ง) OX แก้ปัญหานี้โดย:
1. copy segment ตรงกลางจาก parent1
2. เติม gene ที่เหลือจาก parent2 ตามลำดับที่ปรากฏ (ข้ามที่มีแล้ว)

```
parent1: [3 1 | 2 5 4 | 0 6]    segment = [2,5,4]
parent2: [0 3   1 2 6   5 4]
                ↓
child:   [1 6 | 2 5 4 | 0 3]    ← ลำดับจาก p2 คือ 0,3,1,6 แต่ 2,5,4 มีแล้ว → เติม 0,3,1,6
```

```rust
impl PermIndividual {
    pub fn crossover_ox(
        parent1: &Self,
        parent2: &Self,
        rng: &mut impl Rng,
    ) -> (Self, Self) {
        let n = parent1.genome.len();
        let mut pts = [rng.gen_range(0..n), rng.gen_range(0..n)];
        pts.sort_unstable();
        let (a, b) = (pts[0], pts[1]);
        let child1 = ox_child(&parent1.genome, &parent2.genome, a, b);
        let child2 = ox_child(&parent2.genome, &parent1.genome, a, b);
        (
            PermIndividual { genome: child1, fitness: 0.0 },
            PermIndividual { genome: child2, fitness: 0.0 },
        )
        // fitness ต้องคำนวณใหม่หลัง crossover
    }
}

fn ox_child(p1: &[usize], p2: &[usize], a: usize, b: usize) -> Vec<usize> {
    let n = p1.len();
    let mut child = vec![usize::MAX; n];
    // ขั้น 1: copy segment [a..=b] จาก p1
    child[a..=b].clone_from_slice(&p1[a..=b]);
    let in_segment: std::collections::HashSet<usize> = p1[a..=b].iter().cloned().collect();
    // ขั้น 2: เติมที่เหลือจาก p2 ตามลำดับ circular
    let mut pos = (b + 1) % n;
    let mut src = (b + 1) % n;
    let mut filled = 0;
    let needed = n - (b - a + 1);
    while filled < needed {
        let val = p2[src];
        if !in_segment.contains(&val) {
            child[pos] = val;
            pos = (pos + 1) % n;
            filled += 1;
        }
        src = (src + 1) % n;
    }
    child
}
```

**สาเหตุที่ใช้ `HashSet` แทนการ scan `Vec`**: ตรวจสอบ membership ใน `HashSet` ใช้เวลา O(1) เฉลี่ย ขณะที่ scan `Vec` ใช้ O(n) ทำให้ OX มีความซับซ้อน O(n) แทนที่จะเป็น O(n²)

---

### ขั้นที่ 5: Mutation Operators

Mutation เป็น operator ที่แนะนำ genetic variation แบบสุ่มเพื่อป้องกัน premature convergence มีสองกลุ่มหลักตาม genome type

#### 5.1 Bit-Flip Mutation (Binary Genome)

แต่ละบิตมีโอกาส `mutation_rate` ที่จะถูกกลับค่า โดยทั่วไปใช้ `mutation_rate = 1/L` เมื่อ L คือความยาว genome

```rust
impl BinaryIndividual {
    pub fn mutate(&mut self, mutation_rate: f64, rng: &mut impl Rng) {
        for bit in &mut self.genome {
            if rng.gen::<f64>() < mutation_rate {
                *bit = 1 - *bit;
            }
        }
        // อัปเดต fitness หลัง genome เปลี่ยน
        self.fitness = Self::evaluate(&self.genome);
    }
}
```

**ข้อควรระวัง**: mutation_rate ที่สูงเกินไป (> 0.1 สำหรับ genome ยาว) จะทำให้ algorithm กลายเป็น random search แทนที่จะเป็น guided evolution

#### 5.2 Swap Mutation (Permutation Genome)

สุ่มตำแหน่งสองตำแหน่งแล้วสลับกัน — permutation ยังคง valid

```rust
impl PermIndividual {
    pub fn mutate_swap(
        &mut self,
        mutation_rate: f64,
        rng: &mut impl Rng,
        cities: &[(f64, f64)],
    ) {
        let n = self.genome.len();
        for i in 0..n {
            if rng.gen::<f64>() < mutation_rate {
                let j = rng.gen_range(0..n);
                self.genome.swap(i, j);  // Vec::swap — O(1)
            }
        }
        self.fitness = Self::route_fitness(&self.genome, cities);
    }
}
```

#### 5.3 Inversion Mutation (Permutation Genome)

เลือก sub-sequence สุ่มแล้ว reverse — มักให้ผลดีกว่า swap สำหรับ TSP เพราะ 2-opt neighborhood นั้น strongly connected

```rust
impl PermIndividual {
    pub fn mutate_invert(&mut self, rng: &mut impl Rng, cities: &[(f64, f64)]) {
        let n = self.genome.len();
        let mut pts = [rng.gen_range(0..n), rng.gen_range(0..n)];
        pts.sort_unstable();
        // reverse segment [pts[0]..=pts[1]]
        self.genome[pts[0]..=pts[1]].reverse();
        self.fitness = Self::route_fitness(&self.genome, cities);
    }
}
```

Inversion mutation นั้นสมมูลกับ **2-opt local search** ซึ่งเป็น classical heuristic สำหรับ TSP ดังนั้น GA ที่ใช้ inversion mutation จึงมีองค์ประกอบของ local search ผสมอยู่ด้วย

#### 5.4 Scramble Mutation (Permutation Genome)

เลือก sub-sequence แล้ว shuffle แบบสุ่ม ให้การ perturbation ที่รุนแรงกว่า inversion:

```rust
pub fn mutate_scramble(genome: &mut Vec<usize>, rng: &mut impl Rng) {
    let n = genome.len();
    if n < 2 { return; }
    let mut pts = [rng.gen_range(0..n), rng.gen_range(0..n)];
    pts.sort_unstable();
    let (a, b) = (pts[0], pts[1]);
    genome[a..=b].shuffle(rng);  // ใช้ SliceRandom::shuffle
}
```

---

### ขั้นที่ 6: Generational GA Loop

นี่คือ core loop ที่รวม selection, crossover, mutation, และ elitism เข้าด้วยกัน

#### 6.1 GaConfig และ GaResult

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GaConfig {
    pub population_size: usize,
    pub genome_length: usize,
    pub max_generations: usize,
    pub mutation_rate: f64,
    pub crossover_rate: f64,
    pub tournament_size: usize,
    pub elitism: usize,      // จำนวน top individuals ที่ copy ไป next gen โดยตรง
}

impl Default for GaConfig {
    fn default() -> Self {
        GaConfig {
            population_size: 50,
            genome_length: 20,
            max_generations: 200,
            mutation_rate: 0.01,
            crossover_rate: 0.8,
            tournament_size: 3,
            elitism: 2,
        }
    }
}

#[derive(Debug, Clone)]
pub struct GaResult {
    pub best_fitness: f64,
    pub best_genome: Vec<u8>,
    pub generations_run: usize,
    pub fitness_history: Vec<f64>,   // best fitness ของแต่ละ generation
}
```

**ทำไมต้อง derive `Serialize, Deserialize` สำหรับ `GaConfig`?**
เพราะใน production เราต้องการ save/load config จาก JSON file เพื่อให้การทดลองทำซ้ำได้ และเป็นการ document hyperparameters ที่ใช้ด้วย

#### 6.2 Main GA Loop

```rust
pub fn run_binary_ga(config: &GaConfig, seed: u64) -> GaResult {
    let mut rng = StdRng::seed_from_u64(seed);  // seeded RNG สำหรับ reproducibility

    // 1. Initialize population
    let mut population: Vec<BinaryIndividual> = (0..config.population_size)
        .map(|_| BinaryIndividual::random(config.genome_length, &mut rng))
        .collect();

    let mut fitness_history = Vec::new();
    let target = config.genome_length as f64;  // One-Max target: all bits = 1

    for gen in 0..config.max_generations {
        // 2. Sort by fitness (descending)
        population.sort_by(|a, b| b.fitness.partial_cmp(&a.fitness).unwrap());
        let best = population[0].fitness;
        fitness_history.push(best);

        // 3. Early stopping: solution found
        if best >= target {
            return GaResult {
                best_fitness: best,
                best_genome: population[0].genome.clone(),
                generations_run: gen + 1,
                fitness_history,
            };
        }

        let mut next_gen: Vec<BinaryIndividual> = Vec::with_capacity(config.population_size);

        // 4. Elitism: ส่ง top k ไป next generation โดยตรง (ไม่ mutate)
        for ind in population.iter().take(config.elitism) {
            next_gen.push(ind.clone());
        }

        // 5. Fill rest: crossover + mutation
        while next_gen.len() < config.population_size {
            let p1 = tournament_select(&population, config.tournament_size, &mut rng).clone();
            let p2 = tournament_select(&population, config.tournament_size, &mut rng).clone();

            let (mut c1, mut c2) = if rng.gen::<f64>() < config.crossover_rate {
                BinaryIndividual::crossover_single_point(&p1, &p2, &mut rng)
            } else {
                (p1.clone(), p2.clone())
            };

            c1.mutate(config.mutation_rate, &mut rng);
            c2.mutate(config.mutation_rate, &mut rng);

            next_gen.push(c1);
            if next_gen.len() < config.population_size {
                next_gen.push(c2);
            }
        }

        population = next_gen;
    }

    // Return best found after max_generations
    population.sort_by(|a, b| b.fitness.partial_cmp(&a.fitness).unwrap());
    GaResult {
        best_fitness: population[0].fitness,
        best_genome: population[0].genome.clone(),
        generations_run: config.max_generations,
        fitness_history,
    }
}
```

**แนวคิด Elitism**: ถ้าไม่มี elitism GA อาจสูญเสีย best solution ที่เคยพบไปในรุ่นถัดไป เพราะ selection + crossover + mutation เป็น stochastic operation ที่อาจทำลาย good genome ได้ Elitism = 1-2 ตัวมักเพียงพอ และช่วยให้ convergence curve ไม่ขึ้นๆ ลงๆ

**การใช้งาน:**

```rust
fn main() {
    let config = GaConfig {
        population_size: 100,
        genome_length: 30,
        max_generations: 1000,
        mutation_rate: 1.0 / 30.0,  // rule of thumb: 1/L
        crossover_rate: 0.85,
        tournament_size: 5,
        elitism: 3,
    };

    // Save config to JSON
    let json = serde_json::to_string_pretty(&config).unwrap();
    println!("Config:\n{}", json);

    let result = run_binary_ga(&config, 42);
    println!("Best fitness: {:.1}/{}", result.best_fitness, config.genome_length);
    println!("Converged in {} generations", result.generations_run);
    println!("Best genome: {:?}", result.best_genome);
}
```

---

### ขั้นที่ 7: TSP (Travelling Salesman Problem)

TSP เป็น classic NP-hard combinatorial optimization problem: ให้ n เมืองที่มีพิกัด (x, y) จงหา permutation ของเมืองที่ให้ total tour distance น้อยที่สุด

ขนาดของ search space = (n-1)! / 2 สำหรับ n=20 เมือง = ประมาณ 60 ล้านล้าน combinations — ไม่สามารถ enumerate ทั้งหมดได้

#### 7.1 TSP GA Loop

```rust
pub fn run_tsp_ga(
    cities: &[(f64, f64)],
    population_size: usize,
    max_generations: usize,
    mutation_rate: f64,
    elitism: usize,
    seed: u64,
) -> TspResult {
    let mut rng = StdRng::seed_from_u64(seed);
    let n = cities.len();

    // Initialize: random permutations
    let mut population: Vec<PermIndividual> = (0..population_size)
        .map(|_| PermIndividual::random(n, &mut rng, cities))
        .collect();

    let mut best_dist = f64::MAX;
    let mut best_route = population[0].genome.clone();

    for _gen in 0..max_generations {
        // Sort by fitness (1/dist) descending = sort by dist ascending
        population.sort_by(|a, b| b.fitness.partial_cmp(&a.fitness).unwrap());

        let current_best_dist = route_distance(&population[0].genome, cities);
        if current_best_dist < best_dist {
            best_dist = current_best_dist;
            best_route = population[0].genome.clone();
        }

        // Elitism
        let mut next_gen: Vec<PermIndividual> = population[..elitism].to_vec();

        // Crossover + mutation
        while next_gen.len() < population_size {
            // Tournament from top 10 (pressure สูง)
            let idx1 = rng.gen_range(0..population_size.min(10));
            let idx2 = rng.gen_range(0..population_size.min(10));
            let p1 = &population[idx1];
            let p2 = &population[idx2];

            let (mut c1, mut c2) = PermIndividual::crossover_ox(p1, p2, &mut rng);
            // คำนวณ fitness หลัง OX
            c1.fitness = PermIndividual::route_fitness(&c1.genome, cities);
            c2.fitness = PermIndividual::route_fitness(&c2.genome, cities);
            // Mutation: inversion สำหรับ c1, swap สำหรับ c2
            c1.mutate_invert(&mut rng, cities);
            c2.mutate_swap(mutation_rate, &mut rng, cities);

            next_gen.push(c1);
            if next_gen.len() < population_size {
                next_gen.push(c2);
            }
        }

        population = next_gen;
    }

    TspResult {
        best_distance: best_dist,
        best_route,
        generations_run: max_generations,
    }
}

#[derive(Debug, Clone)]
pub struct TspResult {
    pub best_distance: f64,
    pub best_route: Vec<usize>,
    pub generations_run: usize,
}
```

#### 7.2 ตัวอย่างการแก้ TSP 10 เมือง

```rust
fn demo_tsp() {
    let cities: Vec<(f64, f64)> = vec![
        (0.0, 0.0), (1.0, 2.0), (3.0, 1.0), (5.0, 3.0), (4.0, 5.0),
        (2.0, 6.0), (0.0, 4.0), (1.5, 1.0), (3.5, 4.5), (2.5, 2.5),
    ];

    let result = run_tsp_ga(&cities, 100, 500, 0.05, 2, 42);
    println!("Best distance: {:.4}", result.best_distance);
    println!("Best route: {:?}", result.best_route);
}
```

**ทำไม TSP ถึงเป็นตัวอย่างที่ดีสำหรับ GA?**
1. Search space ใหญ่มาก (NP-hard) จน exact algorithm ไม่ feasible สำหรับ n ใหญ่
2. Permutation encoding สาธิต constraint ที่ binary encoding ไม่มี
3. OX crossover เป็น technique ที่ elegant และ transferable ไปยัง scheduling problems อื่นๆ

---

### ขั้นที่ 8: Multi-Objective NSGA-II Basics

ปัญหา optimization จริงในโลกส่วนใหญ่มีหลาย objectives ที่ขัดแย้งกัน เช่น minimize cost AND maximize quality ซึ่งไม่มีคำตอบเดียวที่ดีที่สุด มีแต่ **Pareto front** — ชุดคำตอบที่ไม่มีคำตอบไหนดีกว่าในทุก objective พร้อมกัน

#### 8.1 MoIndividual และ Pareto Dominance

```rust
#[derive(Clone, Debug)]
pub struct MoIndividual {
    pub genome: Vec<u8>,
    pub objectives: Vec<f64>, // ค่าที่ต้องการ minimize
    pub rank: usize,           // Pareto front rank (1 = best)
    pub crowding_distance: f64,
}

/// a dominates b ถ้า: a ไม่แย่กว่า b ในทุก objective
/// และ a ดีกว่า b อย่างน้อย 1 objective
pub fn dominates(a: &MoIndividual, b: &MoIndividual) -> bool {
    let no_worse = a.objectives.iter()
        .zip(b.objectives.iter())
        .all(|(ai, bi)| ai <= bi);
    let strictly_better = a.objectives.iter()
        .zip(b.objectives.iter())
        .any(|(ai, bi)| ai < bi);
    no_worse && strictly_better
}
```

#### 8.2 Fast Non-Dominated Sort

แบ่ง population ออกเป็น Pareto fronts: Front 1 คือชุดที่ไม่ถูก dominate โดยใคร, Front 2 คือชุดที่ถูก dominate แค่โดย Front 1, และต่อไปเรื่อยๆ

```rust
pub fn non_dominated_sort(population: &mut Vec<MoIndividual>) -> Vec<Vec<usize>> {
    let n = population.len();
    let mut domination_count = vec![0usize; n];
    let mut dominated_set: Vec<Vec<usize>> = vec![Vec::new(); n];
    let mut fronts: Vec<Vec<usize>> = vec![Vec::new()];

    for i in 0..n {
        for j in 0..n {
            if i == j { continue; }
            if dominates(&population[i], &population[j]) {
                dominated_set[i].push(j);
            } else if dominates(&population[j], &population[i]) {
                domination_count[i] += 1;
            }
        }
        if domination_count[i] == 0 {
            population[i].rank = 1;
            fronts[0].push(i);  // Front 1
        }
    }

    let mut current_front = 0;
    while !fronts[current_front].is_empty() {
        let mut next_front = Vec::new();
        for &i in &fronts[current_front].clone() {
            for &j in &dominated_set[i] {
                domination_count[j] -= 1;
                if domination_count[j] == 0 {
                    population[j].rank = current_front + 2;
                    next_front.push(j);
                }
            }
        }
        fronts.push(next_front);
        current_front += 1;
    }
    fronts.pop(); // ลบ empty last front
    fronts
}
```

ความซับซ้อน: O(M × N²) เมื่อ M = จำนวน objectives, N = population size

#### 8.3 Crowding Distance

ใน NSGA-II เราใช้ crowding distance เพื่อเลือกระหว่าง individuals ที่อยู่ใน front เดียวกัน — individual ที่อยู่ในพื้นที่ที่หนาแน่นน้อยกว่า (crowding distance สูง) จะถูกต้องการมากกว่าเพื่อรักษา diversity บน Pareto front

```rust
pub fn assign_crowding_distance(
    population: &mut Vec<MoIndividual>,
    front: &[usize],
) {
    let l = front.len();
    if l == 0 { return; }

    for &i in front {
        population[i].crowding_distance = 0.0;
    }

    let m = population[front[0]].objectives.len();
    for obj_idx in 0..m {
        let mut sorted_front = front.to_vec();
        sorted_front.sort_by(|&a, &b| {
            population[a].objectives[obj_idx]
                .partial_cmp(&population[b].objectives[obj_idx])
                .unwrap()
        });

        // Extreme points ได้ infinity
        population[sorted_front[0]].crowding_distance = f64::INFINITY;
        population[sorted_front[l - 1]].crowding_distance = f64::INFINITY;

        let f_min = population[sorted_front[0]].objectives[obj_idx];
        let f_max = population[sorted_front[l - 1]].objectives[obj_idx];
        let range = f_max - f_min;

        if range > 1e-10 {
            for k in 1..(l - 1) {
                let prev = population[sorted_front[k - 1]].objectives[obj_idx];
                let next = population[sorted_front[k + 1]].objectives[obj_idx];
                population[sorted_front[k]].crowding_distance += (next - prev) / range;
            }
        }
    }
}
```

**NSGA-II Tournament**: เลือกระหว่าง individual `a` กับ `b`:
- ถ้า `a.rank < b.rank` → เลือก `a`
- ถ้า `a.rank == b.rank` และ `a.crowding_distance > b.crowding_distance` → เลือก `a`
- ไม่งั้น → เลือก `b`

---

### ขั้นที่ 9: Convergence Detection และ Diversity Measure

การรู้ว่า GA converge แล้วช่วยหยุด computation ได้เร็วกว่า และช่วยวินิจฉัยปัญหา premature convergence

#### 9.1 Fitness Plateau Detection

```rust
/// ตรวจสอบว่า fitness ไม่เปลี่ยนแปลงใน window รุ่นที่ผ่านมา
pub fn is_converged(history: &[f64], window: usize, tolerance: f64) -> bool {
    if history.len() < window { return false; }
    let recent = &history[history.len() - window..];
    let max = recent.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    let min = recent.iter().cloned().fold(f64::INFINITY, f64::min);
    (max - min) <= tolerance
}
```

ตัวอย่างการใช้ใน GA loop:

```rust
if is_converged(&fitness_history, 50, 1e-4) {
    println!("Converged at generation {}", gen);
    break;
}
```

#### 9.2 Hamming Distance สำหรับวัด Diversity

Hamming distance ระหว่าง binary genomes วัดว่า population มีความหลากหลายมากน้อยแค่ไหน

```rust
pub fn hamming_distance(a: &[u8], b: &[u8]) -> usize {
    a.iter().zip(b.iter()).filter(|(x, y)| x != y).count()
}

/// คำนวณ average Hamming distance ของทุกคู่ใน population
pub fn population_diversity(population: &[BinaryIndividual]) -> f64 {
    let n = population.len();
    if n < 2 { return 0.0; }
    let mut total_dist = 0usize;
    let mut pairs = 0usize;
    for i in 0..n {
        for j in (i + 1)..n {
            total_dist += hamming_distance(&population[i].genome, &population[j].genome);
            pairs += 1;
        }
    }
    total_dist as f64 / pairs as f64
}
```

ตัวอย่างการ monitor diversity ระหว่าง GA:

```rust
let diversity = population_diversity(&population);
if diversity < 1.0 {
    println!("Warning: low diversity ({:.2}) at gen {} — consider restart", diversity, gen);
}
```

**Diversity ใกล้ 0** หมายถึง population เกือบ identical — GA กำลัง premature convergence
**Diversity สูง** ในช่วงแรกและค่อยๆ ลดลงตามรุ่น คือสัญญาณดีว่า algorithm กำลัง exploit knowledge ที่ค้นพบ

---

## การทดสอบ (Testing)

โปรเจคมี unit tests ครอบคลุมทุก component หลัก ตั้งแต่ genome initialization ไปจนถึง multi-objective sort:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_binary_individual_random() {
        let mut rng = StdRng::seed_from_u64(42);
        let ind = BinaryIndividual::random(10, &mut rng);
        assert_eq!(ind.genome.len(), 10);
        assert!(ind.genome.iter().all(|&b| b == 0 || b == 1));
    }

    #[test]
    fn test_one_max_fitness() {
        let genome = vec![1u8, 1, 0, 1, 0];
        let f = BinaryIndividual::evaluate(&genome);
        assert_eq!(f, 3.0);
    }

    #[test]
    fn test_binary_ga_solves_one_max() {
        let config = GaConfig {
            population_size: 50,
            genome_length: 20,
            max_generations: 500,
            mutation_rate: 0.02,
            crossover_rate: 0.8,
            tournament_size: 3,
            elitism: 2,
        };
        let result = run_binary_ga(&config, 42);
        assert_eq!(result.best_fitness, 20.0);
        assert!(result.generations_run <= 500);
    }

    #[test]
    fn test_route_distance_square() {
        let cities = vec![(0.0, 0.0), (1.0, 0.0), (1.0, 1.0), (0.0, 1.0)];
        let route = vec![0, 1, 2, 3];
        let dist = route_distance(&route, &cities);
        assert!((dist - 4.0).abs() < 1e-9);
    }

    // ... tests อื่นๆ
}
```

รัน `cargo test` แล้วได้ผลลัพธ์จริง:

```
running 21 tests
test tests::test_bit_flip_mutation_changes_genome ... ok
test tests::test_convergence_detection ... ok
test tests::test_binary_individual_random ... ok
test tests::test_crossover_single_point_preserves_length ... ok
test tests::test_crossover_two_point_preserves_length ... ok
test tests::test_crowding_distance_extreme_points_infinite ... ok
test tests::test_ga_config_serialization ... ok
test tests::test_hamming_distance ... ok
test tests::test_crossover_uniform_preserves_length ... ok
test tests::test_non_dominated_sort_two_fronts ... ok
test tests::test_one_max_fitness ... ok
test tests::test_pareto_dominance ... ok
test tests::test_ox_crossover_is_valid_permutation ... ok
test tests::test_population_diversity_homogeneous ... ok
test tests::test_permutation_individual_random ... ok
test tests::test_rank_selection_returns_valid ... ok
test tests::test_roulette_selection_returns_valid ... ok
test tests::test_route_distance_square ... ok
test tests::test_tournament_selection_returns_valid ... ok
test tests::test_binary_ga_solves_one_max ... ok
test tests::test_tsp_ga_improves ... ok

test result: ok. 21 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.05s

   Doc-tests genetic_algo

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก 21 tests ผ่าน ครอบคลุม:

| กลุ่ม Test | จำนวน | สิ่งที่ตรวจสอบ |
|-----------|-------|---------------|
| Individual initialization | 2 | genome length, bit values, permutation validity |
| Crossover operators | 4 | single-point, two-point, uniform, OX preserve length |
| Mutation | 1 | bit-flip เปลี่ยน genome จริง |
| Selection | 3 | roulette, tournament, rank ส่งคืน valid individual |
| GA end-to-end | 2 | one-max solves ≤ 500 gen, TSP improves distance |
| TSP helpers | 2 | route_distance (unit square = 4), OX valid permutation |
| Diversity & Convergence | 3 | hamming distance, diversity=0 สำหรับ identical pop, convergence detection |
| NSGA-II | 3 | Pareto dominance, two-front sort, crowding distance infinity |
| Serialization | 1 | GaConfig round-trip JSON |

---

## ข้อผิดพลาดที่พบบ่อย

### ข้อผิดพลาดที่ 1: ลืม update fitness หลัง mutation/crossover

**อาการ**: fitness ที่ stored ใน Individual ไม่ตรงกับ genome จริง ทำให้ selection เลือกตัวที่ผิด

```rust
// ❌ ผิด: mutate genome แต่ไม่ update fitness
fn bad_mutate(ind: &mut BinaryIndividual, rate: f64, rng: &mut impl Rng) {
    for bit in &mut ind.genome {
        if rng.gen::<f64>() < rate {
            *bit = 1 - *bit;
        }
    }
    // ลืม! ind.fitness = BinaryIndividual::evaluate(&ind.genome);
}

// ✅ ถูก: update fitness ทุกครั้งที่ genome เปลี่ยน
pub fn mutate(ind: &mut BinaryIndividual, rate: f64, rng: &mut impl Rng) {
    for bit in &mut ind.genome {
        if rng.gen::<f64>() < rate {
            *bit = 1 - *bit;
        }
    }
    ind.fitness = BinaryIndividual::evaluate(&ind.genome); // ← สำคัญมาก
}
```

**บทเรียน**: ใน Rust ไม่มี getter/setter magic เหมือน Python property เราต้องดูแล cache invalidation เอง แนวทางอีกอย่างคือ lazy evaluation: เก็บ `fitness: Option<f64>` แล้วคำนวณเมื่อ access ครั้งแรก แต่ต้องระวังความซับซ้อนของ code

---

### ข้อผิดพลาดที่ 2: OX Crossover สร้าง invalid permutation

**อาการ**: child มีเมืองซ้ำหรือขาดเมือง ทำให้ route_distance ผิดและ fitness ผิดเพี้ยน

```rust
// ❌ ผิด: naive copy โดยไม่ check membership
fn bad_ox_child(p1: &[usize], p2: &[usize], a: usize, b: usize) -> Vec<usize> {
    let n = p1.len();
    let mut child = vec![usize::MAX; n];
    child[a..=b].clone_from_slice(&p1[a..=b]);

    // BUG: ไม่ check ว่า val ซ้ำกับที่มีแล้วหรือเปล่า
    let mut pos = (b + 1) % n;
    for k in 0..n {
        let src = (b + 1 + k) % n;
        child[pos] = p2[src]; // อาจเขียนทับหรือซ้ำ
        pos = (pos + 1) % n;
    }
    child
}
```

**สาเหตุที่แท้จริง**: ลืมข้ามค่าที่อยู่ใน segment ที่ copy มาจาก p1 แล้ว

```rust
// ✅ ถูก: ใช้ HashSet เพื่อ track ค่าที่มีแล้ว
fn ox_child(p1: &[usize], p2: &[usize], a: usize, b: usize) -> Vec<usize> {
    let n = p1.len();
    let mut child = vec![usize::MAX; n];
    child[a..=b].clone_from_slice(&p1[a..=b]);
    let in_segment: std::collections::HashSet<usize> =
        p1[a..=b].iter().cloned().collect();
    let mut pos = (b + 1) % n;
    let mut src = (b + 1) % n;
    let mut filled = 0;
    let needed = n - (b - a + 1);
    while filled < needed {
        let val = p2[src];
        if !in_segment.contains(&val) {  // ← check ก่อนเสมอ
            child[pos] = val;
            pos = (pos + 1) % n;
            filled += 1;
        }
        src = (src + 1) % n;
    }
    child
}
```

**วิธีตรวจจับ bug**: เขียน test ที่ sort child genome แล้วเปรียบเทียบกับ sorted parent:

```rust
#[test]
fn test_ox_always_valid_permutation() {
    let mut rng = StdRng::seed_from_u64(0);
    for _ in 0..1000 {
        let p1 = PermIndividual::random(10, &mut rng, &[]);
        let p2 = PermIndividual::random(10, &mut rng, &[]);
        let (c1, _) = PermIndividual::crossover_ox(&p1, &p2, &mut rng);
        let mut sorted = c1.genome.clone();
        sorted.sort();
        assert_eq!(sorted, (0..10).collect::<Vec<_>>());
    }
}
```

---

### ข้อผิดพลาดที่ 3: Roulette Wheel Selection ล้มเหลวเมื่อ fitness เป็น 0

**อาการ**: division by zero หรือ infinite loop เมื่อ population มี fitness รวมเป็น 0

```rust
// ❌ ปัญหา: ถ้า total = 0 จะ pick = 0.0 * 0 = 0 หรือ NaN
pub fn bad_roulette(population: &[BinaryIndividual], rng: &mut impl Rng)
    -> &BinaryIndividual
{
    let total: f64 = population.iter().map(|ind| ind.fitness).sum();
    let mut pick = rng.gen::<f64>() * total;  // total = 0 → pick = 0
    for ind in population {
        pick -= ind.fitness;
        if pick <= 0.0 { return ind; }
    }
    population.last().unwrap()
    // ปัญหา: ถ้า total = 0 ทุก individual จะผ่าน check ได้เพราะ pick ลดไม่ได้
}
```

**วิธีแก้**: handle edge case และใช้ fitness shifting ถ้าจำเป็น

```rust
// ✅ ถูก: ตรวจสอบ total ก่อน + เพิ่ม epsilon ถ้าทุกตัวมี fitness 0
pub fn safe_roulette<'a>(
    population: &'a [BinaryIndividual],
    rng: &mut impl Rng,
) -> &'a BinaryIndividual {
    let total: f64 = population.iter().map(|ind| ind.fitness).sum();
    if total <= 0.0 {
        // Fallback: uniform random selection
        return &population[rng.gen_range(0..population.len())];
    }
    let mut pick = rng.gen::<f64>() * total;
    for ind in population {
        pick -= ind.fitness;
        if pick <= 0.0 { return ind; }
    }
    population.last().unwrap()
}
```

**ข้อคิด**: ในปัญหา real-world fitness อาจเป็น negative หรือ zero ได้ ควร normalize fitness ให้อยู่ใน range [0, ∞) ก่อน selection หรือใช้ tournament selection ซึ่ง robust กว่าในกรณีเช่นนี้

---

### ข้อผิดพลาดที่ 4: ใช้ `thread_rng()` โดยตรงทำให้ผลลัพธ์ไม่ reproducible

**อาการ**: รัน program สองครั้งได้ผลต่างกัน ยากต่อการ debug และ compare experiments

```rust
// ❌ ปัญหา: ผลลัพธ์ random ทุกครั้ง
fn bad_init(size: usize, length: usize) -> Vec<BinaryIndividual> {
    let mut rng = thread_rng(); // ← ใช้ OS random seed
    (0..size).map(|_| BinaryIndividual::random(length, &mut rng)).collect()
}
```

```rust
// ✅ ถูก: ใช้ seeded RNG เสมอสำหรับ algorithms
use rand::SeedableRng;

fn good_init(size: usize, length: usize, seed: u64) -> Vec<BinaryIndividual> {
    let mut rng = StdRng::seed_from_u64(seed); // ← deterministic
    (0..size).map(|_| BinaryIndividual::random(length, &mut rng)).collect()
}
```

**แนวปฏิบัติ**: เก็บ seed ลงใน config file ไปด้วย:

```rust
#[derive(Serialize, Deserialize)]
struct ExperimentConfig {
    ga: GaConfig,
    seed: u64,       // บันทึกไว้เพื่อ reproducibility
    run_id: String,
}
```

---

### ข้อผิดพลาดที่ 5: Elitism ทำงานผิดเพราะ population ยังไม่ได้ sort

**อาการ**: "elite" individuals ที่เลือกไว้ไม่ใช่ตัวที่ fitness ดีที่สุดจริงๆ

```rust
// ❌ ผิด: copy elites ก่อน sort
let mut next_gen: Vec<BinaryIndividual> = Vec::new();
for ind in population.iter().take(config.elitism) {
    next_gen.push(ind.clone()); // อาจเป็น individual สุ่ม ไม่ใช่ top
}
population.sort_by(|a, b| b.fitness.partial_cmp(&a.fitness).unwrap());
```

```rust
// ✅ ถูก: sort ก่อนเสมอ แล้วค่อย copy
population.sort_by(|a, b| b.fitness.partial_cmp(&a.fitness).unwrap());
let mut next_gen: Vec<BinaryIndividual> = Vec::new();
for ind in population.iter().take(config.elitism) {
    next_gen.push(ind.clone()); // ตอนนี้ index 0..elitism คือ top จริงๆ
}
```

---

### ข้อผิดพลาดที่ 6: Crowding Distance ผิดเมื่อทุก individual มี objective เดียวกัน

**อาการ**: range = 0 → division by zero → NaN propagation → sort ล้มเหลว

```rust
// ❌ ปัญหา: ลืม check range
if range > 0.0 { // ← ใช้ 0.0 จะไม่ catch floating point near-zero
    population[sorted_front[k]].crowding_distance += (next - prev) / range;
}
```

```rust
// ✅ ถูก: ใช้ epsilon threshold
if range > 1e-10 {
    population[sorted_front[k]].crowding_distance += (next - prev) / range;
}
// ถ้า range ≈ 0 หมายความว่าทุก individual ใน front นี้มี objective นี้เท่ากัน
// → ไม่ต้อง update crowding distance สำหรับ objective นี้
```

---

## การ Package และ Deploy

### Build Release Binary

สำหรับ binary crate ให้เพิ่ม `src/main.rs`:

```bash
cargo build --release
# binary อยู่ที่ target/release/genetic_algo
```

### Benchmark ด้วย Criterion

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "ga_bench"
harness = false
```

```rust
// benches/ga_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use genetic_algo::{run_binary_ga, GaConfig};

fn bench_ga(c: &mut Criterion) {
    let config = GaConfig { genome_length: 50, max_generations: 100, ..GaConfig::default() };
    c.bench_function("binary_ga_50bit", |b| {
        b.iter(|| run_binary_ga(black_box(&config), 42))
    });
}

criterion_group!(benches, bench_ga);
criterion_main!(benches);
```

```bash
cargo bench
```

### Save/Load Config

```rust
use std::fs;

// บันทึก config
let config = GaConfig::default();
let json = serde_json::to_string_pretty(&config).unwrap();
fs::write("config.json", &json).unwrap();

// โหลด config
let loaded: GaConfig = serde_json::from_str(
    &fs::read_to_string("config.json").unwrap()
).unwrap();
```

### Docker Deployment

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/genetic_algo /usr/local/bin/
CMD ["genetic_algo"]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Real-Valued GA สำหรับ Function Optimization ⭐⭐

implement `RealIndividual` ที่ใช้ `Vec<f64>` เป็น genome สำหรับ optimize ฟังก์ชัน Rastrigin:

```
f(x) = 10n + Σ[xᵢ² - 10cos(2πxᵢ)]  โดย xᵢ ∈ [-5.12, 5.12]
```

ค่า minimum global = 0.0 ที่ x = (0, 0, ..., 0)

mutation operator ที่ควรใช้: Gaussian mutation `xᵢ' = xᵢ + N(0, σ)` โดยมี adaptive σ ที่ลดลงตาม generation

crossover ที่เหมาะสม: Arithmetic crossover `c = αp₁ + (1-α)p₂` เมื่อ α สุ่มจาก [0,1]

**hint**: ต้อง clamp ค่าให้อยู่ใน bounds หลัง mutation:
```rust
let mutated = (x + rng.sample(Normal::new(0.0, sigma).unwrap())).clamp(-5.12, 5.12);
```

---

### แบบฝึกหัดที่ 2: Adaptive Mutation Rate ⭐⭐⭐

สังเกตว่าค่า `mutation_rate` ที่ดีที่สุดอาจเปลี่ยนไปตาม generation — ในช่วงแรกควร explore มาก ในช่วงหลังควร exploit fine-grained

implement **self-adaptive mutation** โดยให้แต่ละ `Individual` เก็บ `strategy_parameter: f64` ที่ evolve ไปพร้อมกับ genome:

```rust
pub struct SelfAdaptiveBinaryIndividual {
    pub genome: Vec<u8>,
    pub mutation_rate: f64,   // strategy parameter
    pub fitness: f64,
}
```

mutation rule (Evolution Strategy style):
```
mutation_rate' = mutation_rate × exp(τ × N(0,1))
mutation_rate' = mutation_rate'.max(min_rate)
```

โดย `τ = 1.0 / (genome_length as f64).sqrt()`

เปรียบเทียบ convergence กับ fixed mutation_rate ใน plot

---

### แบบฝึกหัดที่ 3: Island Model GA ⭐⭐⭐

ในปัญหาใหญ่ เราอาจแบ่ง population เป็น "เกาะ" หลายเกาะที่ evolve แยกกัน แล้วแลกเปลี่ยน individuals เป็นระยะ (migration)

implement Island Model:
```rust
pub struct Island {
    pub population: Vec<BinaryIndividual>,
    pub config: GaConfig,
}

pub struct IslandModel {
    pub islands: Vec<Island>,
    pub migration_interval: usize,   // ทุก N generations ถึง migrate
    pub migration_rate: f64,         // สัดส่วน population ที่ migrate
}

impl IslandModel {
    pub fn run(&mut self, generations: usize, seed: u64) -> GaResult;
    fn migrate(&mut self, rng: &mut impl Rng);  // ring topology
}
```

เปรียบเทียบ:
- ไม่มี migration (isolated islands)
- migration ทุก 10 generations
- panmixia (population เดียวขนาดเท่ากัน)

---

### แบบฝึกหัดที่ 4: TSP Visualization ด้วย SVG ⭐⭐

สร้างฟังก์ชัน `route_to_svg(cities, route, filename)` ที่ output SVG file แสดง:
- จุดวงกลมสำหรับแต่ละเมือง พร้อมหมายเลข
- เส้น route ที่ต่อเชื่อมกัน
- title บอก total distance

```rust
pub fn route_to_svg(
    cities: &[(f64, f64)],
    route: &[usize],
    filename: &str,
) -> std::io::Result<()> {
    let width = 600u32;
    let height = 600u32;
    // normalize coordinates ให้อยู่ใน [padding, width-padding]
    // สร้าง SVG string และ write ลงไฟล์
    todo!()
}
```

**bonus**: animate convergence โดยบันทึก best route ของแต่ละ generation แล้ว output animated SVG หรือ HTML+JavaScript

---

### แบบฝึกหัดที่ 5: Parallel GA ด้วย Rayon ⭐⭐⭐⭐

fitness evaluation เป็น bottleneck ที่ใช้เวลามากที่สุดใน GA สำหรับปัญหาที่ evaluation expensive (เช่น simulation)

เพิ่ม `rayon` เป็น dependency แล้ว parallelize fitness evaluation:

```toml
[dependencies]
rayon = "1"
```

```rust
use rayon::prelude::*;

// ❌ Sequential
for ind in &mut population {
    ind.fitness = expensive_evaluate(&ind.genome);
}

// ✅ Parallel
population.par_iter_mut().for_each(|ind| {
    ind.fitness = expensive_evaluate(&ind.genome);
});
```

**ข้อควรระวัง**: RNG ไม่ thread-safe! แต่ fitness evaluation ส่วนใหญ่ไม่ต้องการ RNG ถ้าต้องการ RNG ใน evaluation ให้ใช้ `thread_local!` RNG หรือสร้าง RNG แยกต่างหากต่อ thread

---

### แบบฝึกหัดที่ 6: NSGA-II Full Implementation ⭐⭐⭐⭐⭐

โปรเจคนี้ implement เฉพาะ helpers ของ NSGA-II ลองต่อยอด implement GA loop เต็มรูปแบบ:

```rust
pub fn run_nsga2(
    problem: &impl MultiObjectiveProblem,
    population_size: usize,
    max_generations: usize,
    seed: u64,
) -> Vec<MoIndividual>;

pub trait MultiObjectiveProblem {
    fn evaluate(&self, genome: &[u8]) -> Vec<f64>;
    fn genome_length(&self) -> usize;
}
```

ทดสอบกับ ZDT1 benchmark problem:
```
minimize f₁(x) = x₁
minimize f₂(x) = g(x) × [1 - sqrt(x₁/g(x))]
where g(x) = 1 + 9/(n-1) × Σxᵢ  (i from 2 to n)
```

Pareto front ที่ถูกต้องของ ZDT1 คือ f₂ = 1 - sqrt(f₁) สำหรับ f₁ ∈ [0, 1]

---

## สรุป

ในโปรเจคนี้เราสร้าง Genetic Algorithm framework ครบชุดด้วย Rust ครอบคลุม:

**Genome & Fitness**
- `Individual` trait พร้อม associated type สำหรับ generic genome
- `BinaryIndividual` (One-Max) และ `PermIndividual` (TSP)
- Fitness caching เพื่อหลีกเลี่ยงการคำนวณซ้ำ

**Evolutionary Operators**
- Selection: roulette wheel, tournament (k-way), rank selection
- Crossover binary: single-point, two-point, uniform
- Crossover permutation: Order Crossover (OX) ที่รักษา validity
- Mutation binary: bit-flip; Mutation permutation: swap, inversion, scramble

**GA Loops**
- Generational GA สำหรับ One-Max problem
- TSP GA พร้อม OX crossover และ inversion mutation

**Multi-Objective**
- Pareto dominance check
- Fast non-dominated sort (O(MN²))
- Crowding distance assignment

**Diagnostics**
- Hamming distance และ population diversity
- Fitness plateau / convergence detection

**Pattern สำคัญที่ได้เรียน:**
1. **Associated types ใน trait** ให้ความยืดหยุ่นในการ define genome types โดยไม่สูญเสีย type safety
2. **Seeded RNG** ด้วย `StdRng::seed_from_u64` เป็น best practice สำหรับ reproducible experiments
3. **Elitism** เป็น simple technique ที่ significantly ช่วยให้ convergence เสถียรขึ้น
4. **HashSet membership check** ใน OX crossover คือตัวอย่างที่ดีของการเลือก data structure ที่เหมาะสม
5. **Fitness caching** ใน struct เป็น trade-off ระหว่าง memory กับ compute ที่ต้องจัดการ manually ใน Rust

**โปรเจคถัดไป [project-i09-q-learning.md]** จะเจาะลึก Reinforcement Learning ด้วย Q-Learning ซึ่งเป็น optimization approach อีกแนวที่เรียนรู้จาก interaction กับ environment โดยตรง แทนที่จะวิวัฒน์ population ของคำตอบ — สองแนวทางนี้ complement กันได้ดีในระบบ neuroevolution และ evolutionary reinforcement learning

---

**โปรเจคก่อนหน้า:** [project-i07-time-series.md](project-i07-time-series.md) | **โปรเจคถัดไป:** [project-i09-q-learning.md](project-i09-q-learning.md)
