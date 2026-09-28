# Project I06: Recommender System

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Recommender System** ที่สมบูรณ์ตั้งแต่ต้นจนจบ ครอบคลุมอัลกอริทึมหลักทั้งหมดที่ใช้งานจริงในโลก production ตั้งแต่ Collaborative Filtering แบบพื้นฐาน ไปจนถึง Matrix Factorization, ALS, Implicit Feedback และการจัดการ Cold-Start Problem

ระบบแนะนำสินค้า (Recommender System) เป็นหัวใจสำคัญของ platform ดิจิทัลสมัยใหม่ Netflix ใช้มันเพื่อแนะนำภาพยนตร์, Amazon ใช้แนะนำสินค้า, Spotify ใช้แนะนำเพลง ข้อมูลที่ McKinsey รายงานคือ 35% ของยอดซื้อ Amazon และ 75% ของการรับชม Netflix มาจากระบบแนะนำโดยตรง — นี่คือ ML application ที่สร้าง value ทางธุรกิจมากที่สุดในโลก

**ทำไมต้องสร้างด้วย Rust?** ระบบแนะนำในระดับ production ต้องจัดการกับ matrix ขนาดใหญ่มาก (Netflix มีผู้ใช้ 230 ล้านคน, 15,000 ชื่อ) การคำนวณ latent factors ต้องใช้ memory และ CPU อย่างมีประสิทธิภาพ Rust ให้ทั้ง zero-cost abstractions และ memory safety โดยไม่มี garbage collection overhead

**Use cases จริงในโลก production:**
- ระบบแนะนำสินค้า e-commerce (Amazon, Shopee, Lazada)
- Video/Music streaming recommendations (Netflix, YouTube, Spotify)
- News feed ranking (Facebook, Twitter/X)
- Job matching (LinkedIn, Jobtopgun)
- Content discovery (TikTok, Instagram Reels)
- Drug-protein interaction prediction ในงานวิจัย bioinformatics

## สิ่งที่จะได้เรียนรู้

- **Sparse Matrix representation** — ใช้ `HashMap<(u32, u32), f64>` แทน dense matrix เพื่อประหยัด memory
- **Cosine Similarity และ Pearson Correlation** — วิธีวัด "ความเหมือน" ระหว่าง users และ items
- **User-based Collaborative Filtering** — หา k-nearest-neighbor users แล้วทำนายคะแนนด้วย weighted average
- **Item-based Collaborative Filtering** — สร้าง item-item similarity matrix แล้วทำนายจาก items ที่ user เคย rate
- **Matrix Factorization ด้วย SGD** — เรียนรู้ latent factors U และ V โดยใช้ Stochastic Gradient Descent
- **Alternating Least Squares (ALS)** — วิธีแก้สมการ closed-form โดยสลับ fix U/fix V
- **Implicit Feedback** — จัดการ binary preferences พร้อม confidence weights (Hu et al., 2008)
- **Evaluation Metrics** — RMSE, MAE, Precision@k, Recall@k, NDCG@k
- **Cold-Start Problem** — fallback ด้วย content-based features เมื่อ user/item ใหม่ยังไม่มี rating
- **Gaussian Elimination** — เขียน linear solver จาก scratch เพื่อแก้ ALS normal equations

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`, `HashSet`), iterators, closures, `filter_map`
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics, lifetime basics
- **Part 41–50**: Trait objects, closures, functional programming patterns
- **Part 51–60**: Modules, workspace, `Cargo.toml`, crate ecosystem
- **Part 61–70**: Linear algebra basics (dot product, norms) — ใช้เป็นพื้นฐาน
- **Part 96–105**: Serde serialization — สำหรับ save/load model

## โครงสร้างโปรเจค (Project Layout)

```
recommender/
├── src/
│   ├── main.rs          ← integration demo: โหลด CSV, train, recommend
│   ├── lib.rs           ← re-export modules
│   ├── ratings.rs       ← RatingMatrix (sparse HashMap)
│   ├── similarity.rs    ← cosine_similarity, pearson_correlation
│   ├── user_cf.rs       ← UserCF (user-based collaborative filtering)
│   ├── item_cf.rs       ← ItemCF (item-based collaborative filtering)
│   ├── mf.rs            ← MatrixFactorization (SGD), biases
│   ├── als.rs           ← ALS (Alternating Least Squares)
│   ├── implicit.rs      ← ImplicitALS (confidence-weighted)
│   ├── metrics.rs       ← RMSE, MAE, Precision@k, Recall@k, NDCG@k
│   └── cold_start.rs    ← ContentBasedRecommender (cold-start fallback)
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Input: (user_id, item_id, rating) tuples หรือ CSV
         │
         ▼
┌─────────────────────┐
│   RatingMatrix      │  ← sparse HashMap<(u32,u32), f64>
│   (ratings.rs)      │     user_vector(), item_vector()
│                     │     user_mean(), global_mean()
└─────────────────────┘
         │
    ┌────┴────────────────────────────────┐
    │                                     │
    ▼                                     ▼
┌──────────────────┐             ┌──────────────────────┐
│  Similarity      │             │  Model Layer         │
│  (similarity.rs) │             │                      │
│  cosine_sim()    │──────────── │  UserCF              │
│  pearson_corr()  │             │  ItemCF              │
└──────────────────┘             │  MatrixFactorization │
                                 │  ALS                 │
                                 │  ImplicitALS         │
                                 └──────────────────────┘
                                          │
                                          ▼
                                 ┌──────────────────────┐
                                 │  Metrics             │
                                 │  (metrics.rs)        │
                                 │  RMSE, MAE           │
                                 │  Precision@k         │
                                 │  Recall@k, NDCG@k    │
                                 └──────────────────────┘
```

### Design Decisions

**ทำไมใช้ `HashMap<(u32, u32), f64>` แทน `Vec<Vec<f64>>`?**

ใน real-world dataset ความหนาแน่นของ rating matrix มักจะต่ำมาก MovieLens 20M มีผู้ใช้ 138,000 คน, movies 27,000 เรื่อง แต่มี rating เพียง 20 ล้านรายการ ซึ่งหมายความว่า density = 20M / (138K × 27K) ≈ 0.54% เท่านั้น การใช้ dense matrix จะต้องการ memory 138,000 × 27,000 × 8 bytes ≈ 30 GB แต่ sparse HashMap ต้องการเพียง 20M × ~50 bytes ≈ 1 GB

**ทำไม MF มี bias terms?**

Bias-free model: R[u,i] ≈ U[u] · V[i]
Bias model: R[u,i] ≈ μ + b_u + b_i + U[u] · V[i]

ผู้ใช้บางคนให้คะแนนสูงเสมอ (b_u > 0) บางคนให้ต่ำเสมอ (b_u < 0) บาง item ดีจริง (b_i > 0) บาง item แย่จริง (b_i < 0) การ model bias terms แยกออกมาช่วยให้ latent factors เรียนรู้ความชอบ "เหนือค่าเฉลี่ย" ได้ดีกว่ามาก ทำให้ RMSE ลดลงอย่างมีนัยสำคัญ

**Trait Design สำหรับ Recommender**

ในเวอร์ชัน production ควรกำหนด trait กลาง:

```rust
trait Recommender {
    fn fit(&mut self, matrix: &RatingMatrix);
    fn predict(&self, user: u32, item: u32) -> f64;
    fn recommend(&self, user: u32, matrix: &RatingMatrix, n: usize) -> Vec<(u32, f64)>;
}
```

แต่ในโปรเจคนี้เราใช้ concrete structs เพื่อความชัดเจนในการเรียนรู้

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Rating Matrix — โครงสร้างข้อมูล Sparse

**แนวคิดสำคัญ:** ระบบแนะนำทำงานกับข้อมูลที่ sparse (กระจาย) มาก เราต้องเลือกโครงสร้างข้อมูลที่เหมาะสม

สร้างไฟล์ `Cargo.toml`:

```toml
[package]
name = "recommender"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "recommender"
path = "src/main.rs"

[lib]
name = "recommender"
path = "src/lib.rs"

[dependencies]
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

สร้าง `src/lib.rs`:

```rust
pub mod ratings;
pub mod similarity;
pub mod user_cf;
pub mod item_cf;
pub mod mf;
pub mod als;
pub mod implicit;
pub mod metrics;
pub mod cold_start;
```

สร้าง `src/ratings.rs` — โครงสร้างหลักของ rating data:

```rust
use std::collections::HashMap;

/// Sparse rating matrix: (user_id, item_id) -> rating
#[derive(Debug, Clone, Default)]
pub struct RatingMatrix {
    pub data: HashMap<(u32, u32), f64>,
    pub num_users: u32,
    pub num_items: u32,
}

impl RatingMatrix {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn insert(&mut self, user: u32, item: u32, rating: f64) {
        self.data.insert((user, item), rating);
        if user + 1 > self.num_users {
            self.num_users = user + 1;
        }
        if item + 1 > self.num_items {
            self.num_items = item + 1;
        }
    }

    pub fn get(&self, user: u32, item: u32) -> Option<f64> {
        self.data.get(&(user, item)).copied()
    }

    pub fn items_by_user(&self, user: u32) -> Vec<(u32, f64)> {
        self.data
            .iter()
            .filter(|((u, _), _)| *u == user)
            .map(|((_, i), &r)| (*i, r))
            .collect()
    }

    pub fn users_by_item(&self, item: u32) -> Vec<(u32, f64)> {
        self.data
            .iter()
            .filter(|((_, i), _)| *i == item)
            .map(|((u, _), &r)| (*u, r))
            .collect()
    }

    pub fn user_mean(&self, user: u32) -> Option<f64> {
        let items = self.items_by_user(user);
        if items.is_empty() { return None; }
        let sum: f64 = items.iter().map(|(_, r)| r).sum();
        Some(sum / items.len() as f64)
    }

    pub fn global_mean(&self) -> f64 {
        if self.data.is_empty() { return 0.0; }
        let sum: f64 = self.data.values().sum();
        sum / self.data.len() as f64
    }

    pub fn user_vector(&self, user: u32) -> Vec<f64> {
        let mut v = vec![0.0; self.num_items as usize];
        for ((u, i), &r) in &self.data {
            if *u == user { v[*i as usize] = r; }
        }
        v
    }

    pub fn from_tuples(tuples: &[(u32, u32, f64)]) -> Self {
        let mut m = Self::new();
        for &(u, i, r) in tuples { m.insert(u, i, r); }
        m
    }
}
```

**ทำไม tuple `(u32, u32)` เป็น key ของ HashMap ได้?**

Rust implements `Hash` และ `Eq` สำหรับ tuples ของ types ที่ทั้งคู่ implement `Hash + Eq` เพราะ `u32` implement ทั้งสอง จึงใช้ `(u32, u32)` เป็น HashMap key ได้ทันที

**ความแตกต่างระหว่าง HashMap และ BTreeMap:**
- `HashMap`: lookup O(1) average, แต่ไม่มีลำดับ — เหมาะสำหรับ random access
- `BTreeMap`: lookup O(log n), มีลำดับ sorted — เหมาะสำหรับ range queries

สำหรับ rating matrix ที่ต้องการ random access บ่อยๆ `HashMap` เหมาะกว่า

---

### ขั้นที่ 2: Cosine Similarity และ Pearson Correlation

**แนวคิดสำคัญ:** ก่อนจะทำ collaborative filtering เราต้องกำหนดวิธีวัด "ความเหมือน" ระหว่าง users หรือ items

**Cosine Similarity** วัดมุมระหว่าง vectors สองตัว:

```
cos(θ) = (A · B) / (|A| × |B|)
```

ค่าอยู่ระหว่าง -1 (ตรงข้ามกัน) ถึง 1 (เหมือนกันทุกประการ)

**ตัวอย่างรูปธรรม:**
- User A: ให้คะแนน [5, 4, 0, 0, 3] (ชอบ action, sci-fi)
- User B: ให้คะแนน [4, 5, 0, 0, 4] (ชอบ action, sci-fi เหมือนกัน)
- User C: ให้คะแนน [0, 0, 5, 4, 0] (ชอบ drama, romance)
- cos(A,B) ≈ 0.98 — คล้ายกันมาก
- cos(A,C) ≈ 0.0 — ไม่คล้ายกันเลย

สร้าง `src/similarity.rs`:

```rust
pub fn cosine_similarity(a: &[f64], b: &[f64]) -> f64 {
    assert_eq!(a.len(), b.len(), "vectors must have equal length");
    let dot: f64 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
    let norm_a: f64 = a.iter().map(|x| x * x).sum::<f64>().sqrt();
    let norm_b: f64 = b.iter().map(|x| x * x).sum::<f64>().sqrt();
    if norm_a < 1e-12 || norm_b < 1e-12 {
        return 0.0;
    }
    dot / (norm_a * norm_b)
}

pub fn pearson_correlation(a: &[f64], b: &[f64]) -> f64 {
    assert_eq!(a.len(), b.len());
    // เฉพาะตำแหน่งที่ทั้งสองได้ rate (co-rated items)
    let co: Vec<(f64, f64)> = a.iter().zip(b.iter())
        .filter(|(&x, &y)| x != 0.0 && y != 0.0)
        .map(|(&x, &y)| (x, y))
        .collect();
    let n = co.len();
    if n < 2 { return 0.0; }

    let mean_a = co.iter().map(|(x, _)| x).sum::<f64>() / n as f64;
    let mean_b = co.iter().map(|(_, y)| y).sum::<f64>() / n as f64;

    let num: f64 = co.iter().map(|(x, y)| (x - mean_a) * (y - mean_b)).sum();
    let den_a = co.iter().map(|(x, _)| (x - mean_a).powi(2)).sum::<f64>().sqrt();
    let den_b = co.iter().map(|(_, y)| (y - mean_b).powi(2)).sum::<f64>().sqrt();

    if den_a < 1e-12 || den_b < 1e-12 { return 0.0; }
    num / (den_a * den_b)
}
```

**ความแตกต่างระหว่าง Cosine Similarity และ Pearson Correlation:**

| ด้าน | Cosine Similarity | Pearson Correlation |
|------|------------------|---------------------|
| ปัญหาที่แก้ | ทิศทางของ vector | ความสัมพันธ์เชิงเส้น |
| Rating bias | ไม่จัดการ — user ที่ให้คะแนนสูงทุกอย่างดูเหมือน "เหมือน" user อื่น | จัดการได้ — normalizes ด้วย mean |
| Missing values | ใส่ 0 แทน | ใช้เฉพาะ co-rated items |
| ช่วงค่า | [0, 1] สำหรับ positive ratings | [-1, 1] |

---

### ขั้นที่ 3: User-Based Collaborative Filtering

**แนวคิดสำคัญ:** "คนที่ชอบของคล้ายกันในอดีต น่าจะชอบของชิ้นเดียวกันในอนาคต"

อัลกอริทึม:
1. หา k users ที่คล้ายกับ target user ที่สุด (k-nearest neighbors)
2. ทำนายคะแนน: `pred(u, i) = Σ sim(u, v) × r(v, i) / Σ |sim(u, v)|`

สร้าง `src/user_cf.rs`:

```rust
use crate::ratings::RatingMatrix;
use crate::similarity::cosine_similarity;

pub struct UserCF<'a> {
    pub matrix: &'a RatingMatrix,
    pub k: usize,
}

impl<'a> UserCF<'a> {
    pub fn new(matrix: &'a RatingMatrix, k: usize) -> Self {
        UserCF { matrix, k }
    }

    pub fn similarity(&self, user_a: u32, user_b: u32) -> f64 {
        let va = self.matrix.user_vector(user_a);
        let vb = self.matrix.user_vector(user_b);
        cosine_similarity(&va, &vb)
    }

    pub fn top_k_neighbors(&self, user: u32) -> Vec<(u32, f64)> {
        let mut sims: Vec<(u32, f64)> = (0..self.matrix.num_users)
            .filter(|&u| u != user)
            .map(|u| (u, self.similarity(user, u)))
            .filter(|(_, s)| *s > 0.0)
            .collect();
        sims.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        sims.truncate(self.k);
        sims
    }

    pub fn predict(&self, user: u32, item: u32) -> Option<f64> {
        let neighbors = self.top_k_neighbors(user);
        let rated_neighbors: Vec<(u32, f64, f64)> = neighbors
            .into_iter()
            .filter_map(|(u, sim)| {
                self.matrix.get(u, item).map(|r| (u, sim, r))
            })
            .collect();

        if rated_neighbors.is_empty() { return None; }

        let numerator: f64 = rated_neighbors.iter().map(|(_, s, r)| s * r).sum();
        let denominator: f64 = rated_neighbors.iter().map(|(_, s, _)| s.abs()).sum();

        if denominator < 1e-12 { return None; }
        Some(numerator / denominator)
    }

    pub fn recommend(&self, user: u32, n: usize) -> Vec<(u32, f64)> {
        let already_rated: std::collections::HashSet<u32> = self
            .matrix.items_by_user(user)
            .into_iter().map(|(i, _)| i).collect();

        let mut scores: Vec<(u32, f64)> = (0..self.matrix.num_items)
            .filter(|i| !already_rated.contains(i))
            .filter_map(|i| self.predict(user, i).map(|s| (i, s)))
            .collect();

        scores.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        scores.truncate(n);
        scores
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
let matrix = RatingMatrix::from_tuples(&[
    (0, 0, 5.0), (0, 1, 4.0), (0, 2, 1.0),
    (1, 0, 4.0), (1, 1, 5.0), (1, 3, 2.0),
    (2, 1, 2.0), (2, 2, 4.0), (2, 3, 5.0),
]);

let cf = UserCF::new(&matrix, 2);
let pred = cf.predict(0, 3);
// user 0 ยังไม่ได้ rate item 3
// ดูว่า user ที่คล้ายกัน (user 1) ให้คะแนน item 3 ว่าอย่างไร
println!("Predicted: {:?}", pred);
```

**Lifetime `'a` ในโค้ด:**

`UserCF<'a>` มี lifetime เพราะ `matrix: &'a RatingMatrix` — เราไม่ copy matrix ทั้งก้อน แต่ยืม reference มา ซึ่งหมายความว่า `UserCF` ต้องไม่ outlive `RatingMatrix` ที่มันอ้างถึง

---

### ขั้นที่ 4: Item-Based Collaborative Filtering

**แนวคิดสำคัญ:** แทนที่จะเปรียบเทียบ users เราเปรียบเทียบ items แทน "items ที่ถูก rate คล้ายๆ กันโดย users กลุ่มเดิม น่าจะเป็น items ที่คล้ายกัน"

**ข้อดีของ Item-based CF เหนือ User-based CF:**
- Item similarity เสถียรกว่า (items ไม่เปลี่ยนพฤติกรรม แต่ users เปลี่ยนได้)
- สามารถ precompute item similarity ล่วงหน้าได้
- Scalable กว่าสำหรับ dataset ที่มี users มากกว่า items

สร้าง `src/item_cf.rs`:

```rust
use crate::ratings::RatingMatrix;
use crate::similarity::cosine_similarity;
use std::collections::HashMap;

pub struct ItemCF<'a> {
    pub matrix: &'a RatingMatrix,
    pub k: usize,
    sim_cache: HashMap<(u32, u32), f64>,
}

impl<'a> ItemCF<'a> {
    pub fn new(matrix: &'a RatingMatrix, k: usize) -> Self {
        ItemCF { matrix, k, sim_cache: HashMap::new() }
    }

    fn item_vector(&self, item: u32) -> Vec<f64> {
        let mut v = vec![0.0; self.matrix.num_users as usize];
        for ((u, i), &r) in &self.matrix.data {
            if *i == item { v[*u as usize] = r; }
        }
        v
    }

    pub fn item_similarity(&mut self, item_a: u32, item_b: u32) -> f64 {
        let key = if item_a <= item_b { (item_a, item_b) } else { (item_b, item_a) };
        if let Some(&s) = self.sim_cache.get(&key) { return s; }
        let va = self.item_vector(item_a);
        let vb = self.item_vector(item_b);
        let s = cosine_similarity(&va, &vb);
        self.sim_cache.insert(key, s);
        s
    }

    pub fn precompute(&mut self) {
        let n = self.matrix.num_items;
        for i in 0..n {
            for j in (i + 1)..n {
                self.item_similarity(i, j);
            }
        }
    }

    pub fn predict(&mut self, user: u32, item: u32) -> Option<f64> {
        let user_items = self.matrix.items_by_user(user);
        if user_items.is_empty() { return None; }

        let mut weighted: Vec<(f64, f64)> = user_items.iter()
            .filter(|(j, _)| *j != item)
            .map(|(j, r)| (self.item_similarity(item, *j), *r))
            .filter(|(s, _)| *s > 0.0)
            .collect();
        weighted.sort_by(|a, b| b.0.partial_cmp(&a.0).unwrap());
        weighted.truncate(self.k);

        let numerator: f64 = weighted.iter().map(|(s, r)| s * r).sum();
        let denominator: f64 = weighted.iter().map(|(s, _)| s.abs()).sum();

        if weighted.is_empty() || denominator < 1e-12 { return None; }
        Some(numerator / denominator)
    }
}
```

**Caching Strategy:** เราใช้ `sim_cache: HashMap<(u32, u32), f64>` เพื่อหลีกเลี่ยงการคำนวณ similarity ซ้ำๆ เพราะ item-item similarity computation เป็น O(n²) ต่อ item pairs และแต่ละ computation เป็น O(num_users) รวมทั้งหมด O(n² × m) ถ้าไม่ cache

**Symmetry optimization:** เราใช้ key แบบ `(min(a,b), max(a,b))` เพื่อให้ sim(i,j) === sim(j,i) ใช้ entry เดียวกัน ลด memory ครึ่งหนึ่ง

---

### ขั้นที่ 5: Matrix Factorization ด้วย SGD

**แนวคิดสำคัญ:** แทนที่จะใช้ rating matrix โดยตรง เราแยก (factorize) มันออกเป็นสอง matrix ขนาดเล็กกว่า:

```
R[u, i] ≈ μ + b_u + b_i + U[u] · V[i]
```

โดยที่:
- `μ` = global mean rating
- `b_u` = user bias (คนนี้ให้คะแนนสูง/ต่ำกว่าค่าเฉลี่ยแค่ไหน)
- `b_i` = item bias (สินค้านี้ได้รับคะแนนสูง/ต่ำกว่าค่าเฉลี่ยแค่ไหน)
- `U[u]` = user latent vector ขนาด k (สิ่งที่ user ชอบ)
- `V[i]` = item latent vector ขนาด k (คุณลักษณะของ item)

**SGD Update Rule:**

สำหรับแต่ละ (user u, item i, rating r):
```
err = r - (μ + b_u + b_i + U[u]·V[i])

b_u ← b_u + lr × (err - λ × b_u)
b_i ← b_i + lr × (err - λ × b_i)

for f in 0..k:
    U[u][f] ← U[u][f] + lr × (err × V[i][f] - λ × U[u][f])
    V[i][f] ← V[i][f] + lr × (err × U[u][f] - λ × V[i][f])
```

สร้าง `src/mf.rs`:

```rust
use crate::ratings::RatingMatrix;
use rand::Rng;

#[derive(Debug, Clone)]
pub struct MatrixFactorization {
    pub num_users: usize,
    pub num_items: usize,
    pub k: usize,
    pub lr: f64,
    pub reg: f64,
    pub epochs: usize,
    pub u: Vec<Vec<f64>>,    // User factor matrix: num_users × k
    pub v: Vec<Vec<f64>>,    // Item factor matrix: num_items × k
    pub bu: Vec<f64>,         // User bias
    pub bi: Vec<f64>,         // Item bias
    pub mu: f64,              // Global mean
}

impl MatrixFactorization {
    pub fn new(num_users: usize, num_items: usize, k: usize,
               lr: f64, reg: f64, epochs: usize) -> Self {
        let mut rng = rand::thread_rng();
        let u: Vec<Vec<f64>> = (0..num_users)
            .map(|_| (0..k).map(|_| rng.gen_range(-0.01..0.01)).collect())
            .collect();
        let v: Vec<Vec<f64>> = (0..num_items)
            .map(|_| (0..k).map(|_| rng.gen_range(-0.01..0.01)).collect())
            .collect();
        MatrixFactorization {
            num_users, num_items, k, lr, reg, epochs,
            u, v,
            bu: vec![0.0; num_users],
            bi: vec![0.0; num_items],
            mu: 0.0,
        }
    }

    pub fn predict(&self, user: usize, item: usize) -> f64 {
        let dot: f64 = self.u[user].iter().zip(self.v[item].iter())
            .map(|(a, b)| a * b).sum();
        (self.mu + self.bu[user] + self.bi[item] + dot)
            .max(1.0).min(5.0)
    }

    pub fn fit(&mut self, matrix: &RatingMatrix) {
        self.mu = matrix.global_mean();
        let mut samples: Vec<(usize, usize, f64)> = matrix.data.iter()
            .map(|((u, i), &r)| (*u as usize, *i as usize, r))
            .collect();
        let mut rng = rand::thread_rng();

        for _epoch in 0..self.epochs {
            // Fisher-Yates shuffle
            let n = samples.len();
            for i in (1..n).rev() {
                let j = rng.gen_range(0..=i);
                samples.swap(i, j);
            }

            for &(u, i, r) in &samples {
                let pred = self.mu + self.bu[u] + self.bi[i]
                    + self.u[u].iter().zip(self.v[i].iter())
                        .map(|(a, b)| a * b).sum::<f64>();
                let err = r - pred;

                self.bu[u] += self.lr * (err - self.reg * self.bu[u]);
                self.bi[i] += self.lr * (err - self.reg * self.bi[i]);

                let pu = self.u[u].clone();
                let qi = self.v[i].clone();
                for f in 0..self.k {
                    self.u[u][f] += self.lr * (err * qi[f] - self.reg * pu[f]);
                    self.v[i][f] += self.lr * (err * pu[f] - self.reg * qi[f]);
                }
            }
        }
    }
}

pub fn rmse(model: &MatrixFactorization, matrix: &RatingMatrix) -> f64 {
    let samples: Vec<(usize, usize, f64)> = matrix.data.iter()
        .map(|((u, i), &r)| (*u as usize, *i as usize, r))
        .collect();
    if samples.is_empty() { return 0.0; }
    let sse: f64 = samples.iter()
        .map(|(u, i, r)| (r - model.predict(*u, *i)).powi(2))
        .sum();
    (sse / samples.len() as f64).sqrt()
}
```

**Regularization — ป้องกัน Overfitting:**

ถ้าไม่มี regularization (`reg = 0`) model จะพยายาม minimize training error จนกระทั่ง latent vectors มีค่า magnitude ใหญ่มากๆ ทำให้ทำนาย training data ได้ดีแต่ generalize ไปยัง test data ไม่ได้ — นี่คือ overfitting

regularization term `λ × ||U||² + λ × ||V||²` บังคับให้ weights อยู่ใกล้ 0 ป้องกันไม่ให้ model fit noise ใน training data

---

### ขั้นที่ 6: Alternating Least Squares (ALS)

**แนวคิดสำคัญ:** แทนที่จะใช้ gradient descent ที่ต้องรอ converge อย่างช้าๆ ALS แก้สมการ closed-form โดยสลับ:
1. Fix V, แก้หา U อย่างเหมาะสมที่สุด (linear regression)
2. Fix U, แก้หา V อย่างเหมาะสมที่สุด (linear regression)
3. ทำซ้ำจนกว่าจะ converge

**สมการ ALS Normal Equations:**

เมื่อ Fix V แล้วแก้หา user factor p_u:
```
(V_I^T × C^u × V_I + λI) × p_u = V_I^T × C^u × r^u
```

โดยที่ `V_I` คือ item factors ของ items ที่ user u ได้ rate และ `C^u` คือ diagonal matrix ของ confidence weights

สร้าง `src/als.rs` (เฉพาะ key parts):

```rust
use rand::Rng;

#[derive(Debug, Clone)]
pub struct ALS {
    pub num_users: usize,
    pub num_items: usize,
    pub k: usize,
    pub reg: f64,
    pub epochs: usize,
    pub u: Vec<Vec<f64>>,
    pub v: Vec<Vec<f64>>,
}

impl ALS {
    pub fn fit(&mut self, matrix: &RatingMatrix) {
        for _epoch in 0..self.epochs {
            // Fix V, แก้หา U แต่ละ user
            for u in 0..self.num_users {
                let user_items = matrix.items_by_user(u as u32);
                if user_items.is_empty() { continue; }
                let indices: Vec<usize> = user_items.iter()
                    .map(|(i, _)| *i as usize).collect();
                let ratings: Vec<f64> = user_items.iter()
                    .map(|(_, r)| *r).collect();
                self.u[u] = Self::solve_factor(&self.v, &indices, &ratings,
                                               self.reg, self.k);
            }
            // Fix U, แก้หา V แต่ละ item
            for i in 0..self.num_items {
                let item_users = matrix.users_by_item(i as u32);
                if item_users.is_empty() { continue; }
                let indices: Vec<usize> = item_users.iter()
                    .map(|(u, _)| *u as usize).collect();
                let ratings: Vec<f64> = item_users.iter()
                    .map(|(_, r)| *r).collect();
                self.v[i] = Self::solve_factor(&self.u, &indices, &ratings,
                                               self.reg, self.k);
            }
        }
    }
}
```

**เปรียบเทียบ SGD vs ALS:**

| ด้าน | SGD | ALS |
|------|-----|-----|
| Speed | ช้ากว่าต่อ iteration | เร็วกว่าใน distributed setting |
| Parallelism | ยาก (shared state) | ง่ายมาก — users independent |
| Convergence | ต้องปรับ learning rate | Guaranteed ลดลงทุก iteration |
| Memory | ต่ำ | ต้องการ solve linear systems |
| Implicit feedback | ยาก | ง่าย — เพิ่ม confidence weights |

ALS เป็นที่นิยมมากใน distributed frameworks อย่าง Apache Spark MLlib เพราะ user updates เป็น independent กัน สามารถ parallelize ข้าม cluster ได้ทันที

---

### ขั้นที่ 7: Implicit Feedback

**แนวคิดสำคัญ:** ในชีวิตจริง explicit ratings (1-5 ดาว) มีน้อยมาก แต่ implicit feedback (clicks, purchases, play history) มีมหาศาล

**ความต่างระหว่าง Explicit และ Implicit:**

| ด้าน | Explicit | Implicit |
|------|----------|----------|
| ตัวอย่าง | คะแนน 1-5 ดาว | click, purchase, play time |
| ความหมาย | ชัดเจน — "ชอบ/ไม่ชอบ" | ไม่ชัดเจน — click ≠ ชอบเสมอไป |
| ปริมาณ | น้อย | มหาศาล |
| Negative feedback | มี (คะแนน 1-2) | ไม่มี — ไม่ interact ≠ ไม่ชอบ |

**Hu et al. (2008) framework:**
- `p_{ui} = 1` ถ้า user u ได้ interact กับ item i (binary preference)
- `c_{ui} = 1 + α × count_{ui}` (confidence weight — interaction บ่อยๆ = เชื่อมั่นมากกว่า)
- เมื่อ `α = 40`, `count = 5`: `c = 1 + 40 × 5 = 201` (เชื่อมั่นสูงมาก)
- สำหรับ items ที่ไม่เคย interact: `p = 0`, `c = 1` (unobserved แต่ยังนับใน objective)

สร้าง `src/implicit.rs` (key parts):

```rust
pub struct ImplicitALS {
    pub num_users: usize,
    pub num_items: usize,
    pub k: usize,
    pub reg: f64,
    pub alpha: f64,
    pub epochs: usize,
    pub u: Vec<Vec<f64>>,
    pub v: Vec<Vec<f64>>,
}

impl ImplicitALS {
    pub fn fit_from_counts(&mut self, counts: &HashMap<(u32, u32), u32>) {
        for _epoch in 0..self.epochs {
            for user in 0..self.num_users {
                let mut a = vec![vec![0.0_f64; self.k]; self.k];
                let mut b = vec![0.0_f64; self.k];

                for item in 0..self.num_items {
                    let c = if let Some(&cnt) = counts.get(&(user as u32, item as u32)) {
                        1.0 + self.alpha * cnt as f64
                    } else { 1.0 };

                    let p = if counts.contains_key(&(user as u32, item as u32)) {
                        1.0
                    } else { 0.0 };

                    let v = &self.v[item];
                    for row in 0..self.k {
                        b[row] += c * p * v[row];
                        for col in 0..self.k {
                            a[row][col] += c * v[row] * v[col];
                        }
                    }
                }
                for f in 0..self.k { a[f][f] += self.reg; }
                self.u[user] = solve_linear(&a, &b);
            }
            // ทำเหมือนกันสำหรับ items...
        }
    }
}
```

**ทำไม unobserved items ต้องถูกนำมาคำนวณด้วย?**

นี่คือความแตกต่างหลักจาก explicit feedback ใน explicit model เราคำนวณ loss เฉพาะ observed ratings แต่ใน implicit model เราต้อง penalize การทำนาย high score ให้กับ items ที่ user ไม่เคย interact — เพราะเราไม่รู้ว่า user "ไม่ชอบ" หรือแค่ "ยังไม่รู้จัก" confidence weight ต่ำ (c=1) จึงทำให้ unobserved items มี weight น้อยในการ optimize

---

### ขั้นที่ 8: Evaluation Metrics

**แนวคิดสำคัญ:** จะรู้ได้อย่างไรว่า model ดีแค่ไหน? ต้องมี metrics ที่วัดได้

สร้าง `src/metrics.rs`:

```rust
/// Root Mean Square Error
pub fn rmse(predictions: &[(f64, f64)]) -> f64 {
    if predictions.is_empty() { return 0.0; }
    let sse: f64 = predictions.iter()
        .map(|(pred, actual)| (pred - actual).powi(2)).sum();
    (sse / predictions.len() as f64).sqrt()
}

/// Mean Absolute Error
pub fn mae(predictions: &[(f64, f64)]) -> f64 {
    if predictions.is_empty() { return 0.0; }
    let sum: f64 = predictions.iter()
        .map(|(pred, actual)| (pred - actual).abs()).sum();
    sum / predictions.len() as f64
}

/// Precision@k: fraction of top-k recs that are relevant
pub fn precision_at_k(recommended: &[u32], relevant: &[u32], k: usize) -> f64 {
    let top_k: Vec<u32> = recommended.iter().take(k).copied().collect();
    if top_k.is_empty() { return 0.0; }
    let hits = top_k.iter().filter(|item| relevant.contains(item)).count();
    hits as f64 / k as f64
}

/// Recall@k: fraction of relevant items in top-k
pub fn recall_at_k(recommended: &[u32], relevant: &[u32], k: usize) -> f64 {
    if relevant.is_empty() { return 0.0; }
    let top_k: Vec<u32> = recommended.iter().take(k).copied().collect();
    let hits = top_k.iter().filter(|item| relevant.contains(item)).count();
    hits as f64 / relevant.len() as f64
}

/// Normalized Discounted Cumulative Gain at k
pub fn ndcg_at_k(recommended: &[u32], relevant: &[u32], k: usize) -> f64 {
    let idcg = idcg_at_k(relevant, k);
    if idcg < 1e-12 { return 0.0; }
    dcg_at_k(recommended, relevant, k) / idcg
}

fn dcg_at_k(recommended: &[u32], relevant: &[u32], k: usize) -> f64 {
    recommended.iter().take(k).enumerate()
        .map(|(i, item)| {
            let rel = if relevant.contains(item) { 1.0 } else { 0.0 };
            rel / (2.0_f64 + i as f64).log2()
        })
        .sum()
}

fn idcg_at_k(relevant: &[u32], k: usize) -> f64 {
    let n = relevant.len().min(k);
    (0..n).map(|i| 1.0 / (2.0_f64 + i as f64).log2()).sum()
}
```

**ทำความเข้าใจ NDCG:**

DCG (Discounted Cumulative Gain) ให้ weight กับ position ในรายการแนะนำ — item ที่แนะนำอยู่อันดับ 1 มีค่ามากกว่าอันดับ 5 อย่างมาก

```
DCG@k = Σ_{i=1}^{k} rel_i / log2(i+1)
```

NDCG normalize ด้วย IDCG (ideal DCG ที่ items ที่ relevant ทั้งหมดอยู่อันดับต้นๆ):
- NDCG = 1.0 → perfect ranking
- NDCG = 0.0 → ไม่มี relevant item ใน top-k

**ตัวอย่างเปรียบเทียบ metrics:**

| Scenario | RMSE | Precision@5 | Recall@5 | NDCG@5 |
|----------|------|-------------|----------|--------|
| Perfect | 0.0 | 1.0 | 1.0 | 1.0 |
| Good | 0.5 | 0.6 | 0.75 | 0.82 |
| Poor | 1.2 | 0.2 | 0.25 | 0.31 |
| Random | 1.5 | 0.1 | 0.12 | 0.14 |

**ข้อควรระวัง:** RMSE วัดความแม่นยำของการทำนาย rating แต่ไม่วัดว่า top-N recommendations ดีแค่ไหน สำหรับ production recommendation เราสนใจ ranking metrics (Precision@k, NDCG@k) มากกว่า RMSE

---

### ขั้นที่ 9: Cold-Start Problem และ Content-Based Fallback

**แนวคิดสำคัญ:** Collaborative Filtering ทำงานได้ดีเมื่อมีข้อมูล rating แต่ล้มเหลวในสองสถานการณ์:
1. **User cold start:** New user ที่ยังไม่มี rating ประวัติ — CF ไม่รู้ว่าใครเป็น neighbor
2. **Item cold start:** New item ที่ยังไม่มีใคร rate — CF ไม่รู้จะใส่ใน latent space ไหน

**วิธีแก้ปัญหา:**
- Content-Based Filtering — ใช้ features ของ item (genres, tags, description)
- Demographic filtering — ใช้ข้อมูล user (อายุ, เพศ, location)
- Hybrid approach — ผสม CF + Content-Based

สร้าง `src/cold_start.rs`:

```rust
use std::collections::HashMap;
use crate::similarity::cosine_similarity;

#[derive(Debug, Clone)]
pub struct ItemProfile {
    pub item_id: u32,
    pub features: Vec<f64>,
}

#[derive(Debug, Default)]
pub struct ContentBasedRecommender {
    pub profiles: HashMap<u32, ItemProfile>,
}

impl ContentBasedRecommender {
    pub fn new() -> Self { Self::default() }

    pub fn add_item(&mut self, item_id: u32, features: Vec<f64>) {
        self.profiles.insert(item_id, ItemProfile { item_id, features });
    }

    pub fn similar_to_items(&self, liked_items: &[u32], exclude: &[u32],
                             n: usize) -> Vec<(u32, f64)> {
        if liked_items.is_empty() {
            return self.popular_items(exclude, n);
        }

        // สร้าง average feature vector จาก items ที่ user ชอบ
        let dim = self.profiles.values().next()
            .map(|p| p.features.len()).unwrap_or(0);
        let mut avg = vec![0.0_f64; dim];
        let mut count = 0usize;
        for &item in liked_items {
            if let Some(p) = self.profiles.get(&item) {
                for (i, &f) in p.features.iter().enumerate() { avg[i] += f; }
                count += 1;
            }
        }
        for v in &mut avg { *v /= count as f64; }

        // Score items ด้วย cosine similarity กับ average profile
        let mut scores: Vec<(u32, f64)> = self.profiles.iter()
            .filter(|(id, _)| !exclude.contains(id) && !liked_items.contains(id))
            .map(|(id, profile)| (*id, cosine_similarity(&avg, &profile.features)))
            .collect();
        scores.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
        scores.truncate(n);
        scores
    }

    fn popular_items(&self, exclude: &[u32], n: usize) -> Vec<(u32, f64)> {
        let mut items: Vec<u32> = self.profiles.keys()
            .filter(|id| !exclude.contains(id)).copied().collect();
        items.sort();
        items.into_iter().take(n).map(|id| (id, 1.0)).collect()
    }
}
```

**Hybrid Strategy ใน Production:**

```
ถ้า user มี ratings ≥ threshold → ใช้ CF (MF/ALS)
ถ้า user มี ratings น้อย → ใช้ Content-Based หรือ hybrid
ถ้า user ใหม่ → ถามถึง preferences (onboarding) หรือ popular items
ถ้า item ใหม่ → ใช้ Content-Based เพื่อ bootstrap ก่อน CF จะมีข้อมูลเพียงพอ
```

---

### ขั้นที่ 10: Integration Demo และ CSV Loading

**แนวคิดสำคัญ:** รวมทุกอย่างเข้าด้วยกันใน `main.rs` และดู end-to-end flow

สร้าง `src/main.rs`:

```rust
use recommender::ratings::RatingMatrix;
use recommender::user_cf::UserCF;
use recommender::item_cf::ItemCF;
use recommender::mf::{MatrixFactorization, rmse};
use recommender::metrics::{precision_at_k, recall_at_k, ndcg_at_k};
use recommender::cold_start::ContentBasedRecommender;

fn main() {
    println!("=== Recommender System Demo ===\n");

    let ratings_data = vec![
        (0, 0, 5.0), (0, 1, 4.0), (0, 2, 1.0), (0, 4, 3.0),
        (1, 0, 4.0), (1, 1, 5.0), (1, 3, 2.0), (1, 4, 4.0),
        (2, 1, 2.0), (2, 2, 4.0), (2, 3, 5.0), (2, 4, 1.0),
        (3, 0, 1.0), (3, 2, 5.0), (3, 3, 4.0), (3, 5, 3.0),
        (4, 0, 3.0), (4, 1, 3.0), (4, 4, 5.0), (4, 5, 4.0),
        (5, 2, 4.0), (5, 3, 3.0), (5, 5, 5.0), (5, 0, 2.0),
    ];

    let matrix = RatingMatrix::from_tuples(&ratings_data);
    println!("Rating Matrix: {} users, {} items, {} ratings",
        matrix.num_users, matrix.num_items, matrix.data.len());
    println!("Global mean: {:.2}\n", matrix.global_mean());

    // User-Based CF
    let ucf = UserCF::new(&matrix, 3);
    let top_neighbors = ucf.top_k_neighbors(0);
    println!("Top neighbors for user 0: {:?}", top_neighbors);

    // Matrix Factorization
    let mut mf = MatrixFactorization::new(
        matrix.num_users as usize, matrix.num_items as usize,
        10, 0.01, 0.02, 200);
    mf.fit(&matrix);
    let train_rmse = rmse(&mf, &matrix);
    println!("Training RMSE after 200 epochs: {:.4}", train_rmse);

    // Cold-start
    let mut cb = ContentBasedRecommender::new();
    cb.add_item(0, vec![1.0, 0.0, 0.0, 0.0, 1.0]);
    cb.add_item(1, vec![1.0, 0.0, 0.0, 0.0, 1.0]);
    cb.add_item(2, vec![0.0, 1.0, 1.0, 0.0, 0.0]);
    let cb_recs = cb.similar_to_items(&[0], &[0], 3);
    println!("Cold-start recs: {:?}", cb_recs);
}
```

**Output จากการรันจริง:**

```
=== Recommender System Demo ===

Rating Matrix: 6 users, 6 items, 24 ratings
Global mean: 3.42

--- User-Based Collaborative Filtering ---
Top neighbors for user 0: [(1, 0.9322949635538228), (4, 0.7656639446725211), (2, 0.30969005213075024)]
Predicted rating for user=0, item=3: 2.75
Top-3 recommendations for user 0: [(5, 4.0), (3, 2.748052629185831)]

--- Item-Based Collaborative Filtering ---
Item-CF predicted rating for user=0, item=3: 2.49
Item-CF top-3 for user 0: [(5, 2.7725098054324455), (3, 2.4923501444275633)]

--- Matrix Factorization (SGD) ---
Training RMSE after 200 epochs: 0.3824
MF top-3 for user 0: [(5, 4.112159417468007), (3, 1.2324215013455853)]

--- Evaluation Metrics ---
RMSE: 0.3824
MAE:  0.2996

--- Cold-Start: Content-Based Fallback ---
Cold-start recs for new user who liked item 0: [(1, 0.9999999999999998), (4, 0.4999999999999999), (5, 0.0)]

=== Demo Complete ===
```

**สังเกตผล:**
- User-CF พบว่า user 1 คล้ายกับ user 0 มากที่สุด (similarity ≈ 0.93) ซึ่งสมเหตุสมผลเพราะทั้งคู่ให้คะแนนสูงให้ item 0 และ 1
- MF สามารถ fit training data ได้ดีมาก (RMSE = 0.38) หลัง 200 epochs
- Cold-start แนะนำ item 1 (cosine ≈ 1.0) เพราะมี feature vector เหมือน item 0 ทุกประการ

---

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ครบทุก module รันด้วย `cargo test` แล้วได้ผล:

```
   Compiling recommender v0.1.0 (/tmp/.../scratchpad/recommender)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 2.98s
     Running unittests src/lib.rs (target/debug/deps/recommender-70b6ec1bbb5771a5)

running 35 tests
test cold_start::tests::test_cold_start_no_ratings ... ok
test cold_start::tests::test_exclude_already_seen ... ok
test als::tests::test_als_predict_in_range ... ok
test implicit::tests::test_confidence_weight ... ok
test implicit::tests::test_implicit_recommend ... ok
test cold_start::tests::test_similar_items_basic ... ok
test item_cf::tests::test_item_similarity_self ... ok
test implicit::tests::test_implicit_als_trains ... ok
test item_cf::tests::test_item_similarity_range ... ok
test item_cf::tests::test_predict_item_cf ... ok
test metrics::tests::test_f1 ... ok
test metrics::tests::test_mae_known ... ok
test metrics::tests::test_ndcg_perfect ... ok
test metrics::tests::test_ndcg_zero ... ok
test metrics::tests::test_precision_at_k ... ok
test metrics::tests::test_rmse_known ... ok
test metrics::tests::test_rmse_perfect ... ok
test als::tests::test_als_trains ... ok
test metrics::tests::test_recall_at_k ... ok
test mf::tests::test_mf_predict_in_range ... ok
test mf::tests::test_mf_trains_and_rmse_decreases ... ok
test ratings::tests::test_insert_and_get ... ok
test ratings::tests::test_user_mean ... ok
test mf::tests::test_mf_recommend_not_already_rated ... ok
test ratings::tests::test_global_mean ... ok
test similarity::tests::test_cosine_known_value ... ok
test similarity::tests::test_cosine_orthogonal ... ok
test similarity::tests::test_pearson_perfect ... ok
test ratings::tests::test_user_vector ... ok
test similarity::tests::test_cosine_identical ... ok
test similarity::tests::test_pearson_with_zeros ... ok
test user_cf::tests::test_top_k_neighbors ... ok
test user_cf::tests::test_predict_returns_some ... ok
test user_cf::tests::test_recommend_not_already_rated ... ok
test user_cf::tests::test_user_similarity_self ... ok

test result: ok. 35 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s

     Running unittests src/main.rs (target/debug/deps/recommender-2d3f928aa4d98e40)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests recommender

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**สรุป Tests ที่ครอบคลุม:**

| Module | Tests | ครอบคลุม |
|--------|-------|---------|
| `ratings` | 4 | insert/get, user_mean, global_mean, user_vector |
| `similarity` | 5 | cosine identical/orthogonal/known, pearson perfect/zeros |
| `user_cf` | 4 | similarity self, top_k_neighbors, predict, recommend |
| `item_cf` | 3 | similarity self, range, predict |
| `mf` | 3 | trains (RMSE check), predict in range, recommend |
| `als` | 2 | trains, predict in range |
| `implicit` | 3 | trains, recommend, confidence weight |
| `metrics` | 8 | RMSE, MAE, Precision@k, Recall@k, NDCG perfect/zero, F1 |
| `cold_start` | 3 | similar items, no ratings, exclude seen |
| **รวม** | **35** | |

**Property-Based Testing (แนวคิดเพิ่มเติม):**

สำหรับ production คุณอาจต้องการ property tests เช่น:
- `cosine_similarity(v, v) == 1.0` สำหรับทุก vector v ≠ 0
- `cosine_similarity(a, b) == cosine_similarity(b, a)` (symmetry)
- `ndcg_at_k ∈ [0, 1]` สำหรับทุก input
- ใช้ `proptest` crate สำหรับ property-based testing อัตโนมัติ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### ข้อผิดพลาดที่ 1: Division by Zero ใน Cosine Similarity

**ปัญหา:** ผู้เรียนหลายคนลืม handle กรณีที่ vector มี norm เท่ากับ 0

```rust
// ❌ ผิด — panic! เมื่อ norm = 0
fn cosine_bad(a: &[f64], b: &[f64]) -> f64 {
    let dot: f64 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
    let norm_a = a.iter().map(|x| x * x).sum::<f64>().sqrt();
    let norm_b = b.iter().map(|x| x * x).sum::<f64>().sqrt();
    dot / (norm_a * norm_b)  // division by zero!
}
```

**สถานการณ์ที่เกิด:** User ที่ยังไม่เคย rate อะไรเลย จะมี user vector เป็น `[0, 0, 0, ...]` ทำให้ norm = 0

**วิธีแก้:**
```rust
// ✅ ถูก — check ก่อน divide
fn cosine_good(a: &[f64], b: &[f64]) -> f64 {
    let dot: f64 = a.iter().zip(b.iter()).map(|(x, y)| x * y).sum();
    let norm_a = a.iter().map(|x| x * x).sum::<f64>().sqrt();
    let norm_b = b.iter().map(|x| x * x).sum::<f64>().sqrt();
    if norm_a < 1e-12 || norm_b < 1e-12 {
        return 0.0;  // no similarity with empty vector
    }
    dot / (norm_a * norm_b)
}
```

---

### ข้อผิดพลาดที่ 2: Recommendation ซ้ำกับ Items ที่ User Rate แล้ว

**ปัญหา:** ลืม filter items ที่ user เคย rate ออกจาก recommendations

```rust
// ❌ ผิด — อาจแนะนำ item ที่ user เคย rate แล้ว
fn recommend_bad(user: u32, matrix: &RatingMatrix, n: usize) -> Vec<(u32, f64)> {
    let mut scores: Vec<(u32, f64)> = (0..matrix.num_items)
        .filter_map(|i| predict(user, i).map(|s| (i, s)))
        .collect();
    scores.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    scores.truncate(n);
    scores
}
```

**วิธีแก้:**
```rust
// ✅ ถูก — filter out already-rated items
fn recommend_good(user: u32, matrix: &RatingMatrix, n: usize) -> Vec<(u32, f64)> {
    let already_rated: HashSet<u32> = matrix.items_by_user(user)
        .into_iter().map(|(i, _)| i).collect();

    let mut scores: Vec<(u32, f64)> = (0..matrix.num_items)
        .filter(|i| !already_rated.contains(i))  // ← สำคัญมาก!
        .filter_map(|i| predict(user, i).map(|s| (i, s)))
        .collect();
    scores.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    scores.truncate(n);
    scores
}
```

---

### ข้อผิดพลาดที่ 3: Borrow Checker กับ `ItemCF::item_similarity` ที่ต้องการ `&mut self`

**ปัญหา:** เมื่อพยายามเรียก `predict` ซึ่ง call `item_similarity` ภายใน loop ที่ iterate ผ่าน `user_items`

```rust
// ❌ ผิด — cannot borrow `self.sim_cache` as mutable
// while it's also borrowed through `self.v[i]` chain
pub fn predict_bad(&mut self, user: u32, item: u32) -> Option<f64> {
    let user_items = self.matrix.items_by_user(user);
    let mut result = 0.0;
    for (j, r) in user_items.iter() {
        let sim = self.item_similarity(item, *j);  // ← mutable borrow
        result += sim * r;
    }
    // ...
}
```

**วิธีแก้:** Clone ข้อมูลออกมาก่อน หรือ collect ก่อน iterate:
```rust
// ✅ ถูก — collect user_items ก่อน แล้วค่อย borrow cache
pub fn predict(&mut self, user: u32, item: u32) -> Option<f64> {
    // collect ออกมาเป็น Vec ก่อน — ตัดการ borrow ของ `self.matrix`
    let user_items: Vec<(u32, f64)> = self.matrix.items_by_user(user);
    let mut weighted: Vec<(f64, f64)> = user_items.iter()
        .filter(|(j, _)| *j != item)
        .map(|(j, r)| (self.item_similarity(item, *j), *r))  // ← ตอนนี้ OK
        .filter(|(s, _)| *s > 0.0)
        .collect();
    // ...
}
```

---

### ข้อผิดพลาดที่ 4: SGD Learning Rate ที่ไม่เหมาะสม

**ปัญหา:** เลือก learning rate ผิด ทำให้ model ไม่ converge หรือ diverge

```rust
// ❌ ผิด — learning rate สูงเกินไป → loss oscillates หรือ diverge
let mut mf = MatrixFactorization::new(
    num_users, num_items, 10,
    0.1,  // ← lr สูงเกิน → SGD diverges
    0.02, 200
);

// ❌ ผิด — learning rate ต่ำเกินไป → converge ช้ามาก
let mut mf = MatrixFactorization::new(
    num_users, num_items, 10,
    0.0001,  // ← lr ต่ำเกิน → ต้องใช้ epochs เยอะมาก
    0.02, 200
);
```

**วิธีแก้ — ใช้ค่า default ที่ได้รับการทดสอบแล้ว:**
```rust
// ✅ ถูก — learning rate ที่เหมาะสม
let mut mf = MatrixFactorization::new(
    num_users, num_items,
    k: 10,
    lr: 0.01,   // ← learning rate สำหรับ SGD MF
    reg: 0.02,  // ← L2 regularization
    epochs: 100,
);
```

**หลักการเลือก hyperparameters:**
- `lr = 0.005–0.02` สำหรับ MF SGD
- `reg = 0.01–0.1` ขึ้นอยู่กับ dataset size
- `k = 10–50` สำหรับ latent factors (ใหญ่กว่า = ซับซ้อนกว่า แต่เสี่ยง overfit)
- ใช้ cross-validation เพื่อเลือก hyperparameters ที่เหมาะสมที่สุด

---

### ข้อผิดพลาดที่ 5: คำนวณ Pearson Correlation กับ vectors ที่มีแค่ 1 co-rated item

**ปัญหา:** ถ้า users สองคน rate ร่วมกันแค่ 1 item ค่า standard deviation จะเป็น 0

```rust
// ❌ ปัญหา — n = 1 ทำให้ standard deviation = 0 → division by zero
// หรือ Pearson = NaN
```

**วิธีแก้:**
```rust
// ✅ ถูก — require อย่างน้อย 2 co-rated items
let n = co.len();
if n < 2 { return 0.0; }  // ← minimum threshold
```

---

### ข้อผิดพลาดที่ 6: Gaussian Elimination ไม่มี Partial Pivoting

**ปัญหา:** ใน ALS การแก้ normal equations อาจเจอ near-zero pivot ที่ทำให้ numerical instability

```rust
// ❌ ผิด — ไม่มี partial pivoting → numerically unstable
fn solve_bad(a: &[Vec<f64>], b: &[f64]) -> Vec<f64> {
    let n = b.len();
    // ... elimination โดยไม่ swap rows ...
    for row in (col + 1)..n {
        let factor = m[row][col] / m[col][col];  // อาจเป็น division by near-zero!
        // ...
    }
}
```

**วิธีแก้:**
```rust
// ✅ ถูก — มี partial pivoting
fn solve_good(a: &[Vec<f64>], b: &[f64]) -> Vec<f64> {
    // ... augmented matrix ...
    for col in 0..n {
        // หา row ที่มีค่า absolute ใหญ่สุดใน column col
        let mut max_row = col;
        let mut max_val = m[col][col].abs();
        for row in (col + 1)..n {
            if m[row][col].abs() > max_val {
                max_val = m[row][col].abs();
                max_row = row;
            }
        }
        m.swap(col, max_row);  // ← swap rows → stable

        let pivot = m[col][col];
        if pivot.abs() < 1e-12 { continue; }  // ← skip near-zero
        // ...
    }
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/recommender
```

### Serialize และ Save Model ด้วย Serde

เพิ่ม `serde` derive ใน struct:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MatrixFactorization {
    pub num_users: usize,
    pub num_items: usize,
    pub k: usize,
    pub lr: f64,
    pub reg: f64,
    pub epochs: usize,
    pub u: Vec<Vec<f64>>,
    pub v: Vec<Vec<f64>>,
    pub bu: Vec<f64>,
    pub bi: Vec<f64>,
    pub mu: f64,
}
```

จากนั้น save/load:

```rust
use std::fs;

// Save model
fn save_model(model: &MatrixFactorization, path: &str) -> std::io::Result<()> {
    let json = serde_json::to_string_pretty(model)
        .expect("serialization failed");
    fs::write(path, json)
}

// Load model
fn load_model(path: &str) -> MatrixFactorization {
    let json = fs::read_to_string(path)
        .expect("cannot read model file");
    serde_json::from_str(&json)
        .expect("deserialization failed")
}
```

### Load Ratings จาก CSV

ตัวอย่าง format: `user_id,item_id,rating`

```rust
use std::fs::File;
use std::io::{BufRead, BufReader};

fn load_csv(path: &str) -> RatingMatrix {
    let file = File::open(path).expect("cannot open file");
    let reader = BufReader::new(file);
    let mut matrix = RatingMatrix::new();

    for line in reader.lines().skip(1) {  // skip header
        let line = line.unwrap();
        let parts: Vec<&str> = line.split(',').collect();
        if parts.len() < 3 { continue; }

        let user: u32 = parts[0].trim().parse().unwrap_or(0);
        let item: u32 = parts[1].trim().parse().unwrap_or(0);
        let rating: f64 = parts[2].trim().parse().unwrap_or(0.0);

        if rating > 0.0 {
            matrix.insert(user, item, rating);
        }
    }
    matrix
}
```

### Docker Deployment

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /app/target/release/recommender .
COPY ratings.csv .
CMD ["./recommender"]
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Bias-Adjusted Collaborative Filtering ⭐⭐

**เป้าหมาย:** ปรับปรุง User-based CF โดยใช้ bias-adjusted ratings แทน raw ratings

**พื้นหลัง:** User บางคนมีนิสัยให้คะแนนสูงทุกอย่าง (generous rater) บางคนให้ต่ำทุกอย่าง (critical rater) การเปรียบเทียบ raw ratings โดยตรงอาจทำให้ similarity ผิดเพี้ยน

**งานที่ต้องทำ:**
1. คำนวณ `r̄_u` = mean rating ของ user u สำหรับทุก user
2. แก้ไขฟังก์ชัน `similarity()` ให้ใช้ normalized ratings: `r'(u,i) = r(u,i) - r̄_u`
3. แก้ไขฟังก์ชัน `predict()` ให้ทำนายแบบ mean-centered:
   ```
   pred(u,i) = r̄_u + Σ sim(u,v) × (r(v,i) - r̄_v) / Σ |sim(u,v)|
   ```
4. เปรียบเทียบ RMSE กับวิธีเดิม

**힌트:** สร้าง method `normalized_user_vector(user: u32) -> Vec<f64>` ใน `RatingMatrix`

---

### แบบฝึกหัดที่ 2: Train/Test Split และ Cross-Validation ⭐⭐⭐

**เป้าหมาย:** implement train/test split และ k-fold cross-validation สำหรับ evaluation ที่น่าเชื่อถือ

**งานที่ต้องทำ:**
1. สร้าง function `train_test_split(matrix: &RatingMatrix, test_ratio: f64) -> (RatingMatrix, RatingMatrix)` โดยใช้ `rand::thread_rng()` เพื่อ shuffle ก่อน split
2. สร้าง function `k_fold_cv(matrix: &RatingMatrix, k: usize) -> Vec<f64>` ที่:
   - แบ่ง ratings ออกเป็น k folds
   - train บน k-1 folds, evaluate บน 1 fold
   - return RMSE สำหรับแต่ละ fold
3. เปรียบเทียบ training RMSE vs. test RMSE ของ MF เพื่อดู overfitting

**หมายเหตุ:** ระวัง data leakage — อย่าให้ user/item ที่อยู่เฉพาะใน test set

---

### แบบฝึกหัดที่ 3: Learning Rate Schedule ใน SGD ⭐⭐⭐

**เป้าหมาย:** implement learning rate decay เพื่อ improve convergence

**งานที่ต้องทำ:**
1. เพิ่ม field `lr_decay: f64` ใน `MatrixFactorization` struct
2. แก้ `fit()` ให้ลด learning rate ทุก epoch:
   ```rust
   // Option A: Exponential decay
   current_lr = initial_lr * lr_decay.powi(epoch as i32)

   // Option B: Inverse decay
   current_lr = initial_lr / (1.0 + lr_decay * epoch as f64)
   ```
3. plot RMSE vs epoch สำหรับ lr=0.01 ไม่มี decay vs. decay=0.99 (พิมพ์เป็น text chart ใน terminal)
4. สังเกตว่า decay ช่วย converge เร็วขึ้นและ stable ขึ้นแค่ไหน

---

### แบบฝึกหัดที่ 4: Top-N Evaluation Pipeline ⭐⭐⭐⭐

**เป้าหมาย:** สร้าง full evaluation pipeline สำหรับ ranking quality

**งานที่ต้องทำ:**
1. ใช้ train/test split จาก Exercise 2
2. สำหรับแต่ละ user ใน test set:
   - กำหนด "relevant items" = test items ที่มี rating ≥ 4.0
   - generate top-N recommendations จาก model (trained บน train set)
   - คำนวณ Precision@5, Recall@5, NDCG@5
3. Aggregate ด้วย mean ทุก users
4. เปรียบเทียบ UserCF, ItemCF, และ MF บน metrics เดียวกัน
5. สร้าง function `evaluate_model` ที่รับ model ใดๆ และ return `EvaluationReport`

---

### แบบฝึกหัดที่ 5: Hybrid Recommender ⭐⭐⭐⭐⭐

**เป้าหมาย:** รวม CF และ Content-Based เข้าด้วยกันเป็น hybrid system

**งานที่ต้องทำ:**
1. สร้าง struct `HybridRecommender`:
   ```rust
   pub struct HybridRecommender {
       pub mf: MatrixFactorization,
       pub cb: ContentBasedRecommender,
       pub cf_weight: f64,    // น้ำหนักของ CF score
       pub cb_weight: f64,    // น้ำหนักของ content score
       pub cold_start_threshold: usize,  // ถ้า user มี ratings น้อยกว่านี้ → ใช้ CB มากขึ้น
   }
   ```
2. Implement method `predict_hybrid()` ที่ blend scores:
   ```
   score = α × score_cf + (1-α) × score_cb
   ```
   โดย α = `min(1.0, num_user_ratings / cold_start_threshold)`
3. Test ว่า hybrid ช่วย cold-start users ได้จริงหรือไม่ เปรียบเทียบกับ pure CF

---

### แบบฝึกหัดที่ 6: MovieLens Dataset Integration ⭐⭐⭐⭐⭐

**เป้าหมาย:** load และ train บน MovieLens 100K dataset จริง

**งานที่ต้องทำ:**
1. Download MovieLens 100K จาก https://grouplens.org/datasets/movielens/100k/
2. Parse `u.data` file (tab-separated: user_id, movie_id, rating, timestamp)
3. Build `RatingMatrix` จาก dataset (943 users, 1682 movies, 100,000 ratings)
4. Train MF model ด้วย hyperparameters ที่เหมาะสมสำหรับ dataset ขนาดนี้
5. Evaluate ด้วย 80/20 train-test split
6. เปรียบเทียบ training time ระหว่าง SGD และ ALS

**ตัวอย่าง code สำหรับ parse:**
```rust
fn load_movielens(path: &str) -> RatingMatrix {
    let content = std::fs::read_to_string(path).unwrap();
    let tuples: Vec<(u32, u32, f64)> = content.lines()
        .filter_map(|line| {
            let cols: Vec<&str> = line.split('\t').collect();
            if cols.len() < 3 { return None; }
            let user: u32 = cols[0].parse().ok()?;
            let item: u32 = cols[1].parse().ok()?;
            let rating: f64 = cols[2].parse().ok()?;
            Some((user - 1, item - 1, rating))  // 0-indexed
        })
        .collect();
    RatingMatrix::from_tuples(&tuples)
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง Recommender System ที่ครบสมบูรณ์ใน Rust ตั้งแต่ต้นจนจบ ครอบคลุมอัลกอริทึมหลักทั้งหมดที่ใช้งานจริงใน industry:

**สิ่งที่สร้าง:**
- **`RatingMatrix`** — sparse representation ด้วย `HashMap<(u32, u32), f64>` ที่ประหยัด memory สำหรับ real-world datasets
- **`UserCF` / `ItemCF`** — Collaborative Filtering สองแบบพร้อม cosine similarity และ k-nearest-neighbor prediction
- **`MatrixFactorization`** — SGD-based latent factor model พร้อม user/item biases และ L2 regularization
- **`ALS`** — Alternating Least Squares ด้วย Gaussian elimination แบบ from scratch
- **`ImplicitALS`** — Implicit feedback model ตาม Hu et al. (2008) พร้อม confidence weights
- **`metrics`** — evaluation suite ครบถ้วน: RMSE, MAE, Precision@k, Recall@k, NDCG@k, F1
- **`ContentBasedRecommender`** — cold-start fallback ด้วย cosine similarity บน content features

**Pattern สำคัญที่ได้เรียนรู้:**

1. **Sparse vs Dense Trade-off** — เมื่อ data density ต่ำกว่า 1-5% HashMap ประหยัด memory กว่า Vec&lt;Vec&lt;f64&gt;&gt; อย่างมาก
2. **Lifetime Annotations** — `UserCF<'a>` ที่ hold reference ไปยัง `RatingMatrix` แทนการ clone ข้อมูลทั้งก้อน
3. **Numerical Stability** — ต้อง check near-zero denominators, ใช้ partial pivoting ใน linear solver
4. **Borrow Checker Navigation** — `collect()` ก่อน iterate เมื่อต้องการ `&mut self` ภายใน loop
5. **Caching Pattern** — `sim_cache: HashMap<(u32, u32), f64>` เพื่อ memoize ผลการคำนวณที่แพง

**เชื่อมโยงกับโปรเจคถัดไป:**

โปรเจค **Project I07: Time Series Analysis** จะนำ pattern ของ iterative training ที่เรียนรู้ในโปรเจคนี้ (SGD loop, convergence monitoring) ไปใช้กับ sequence data ต่อเนื่องตามเวลา โดยเพิ่มเรื่อง ARIMA models, seasonal decomposition, และ LSTM-inspired recurrent structures ใน Rust

---

**โปรเจคก่อนหน้า:** [Project I05: NLP Tokenizer & BPE](project-i05-nlp-tokenizer.md) | **โปรเจคถัดไป:** [Project I07: Time Series Analysis](project-i07-time-series.md)
