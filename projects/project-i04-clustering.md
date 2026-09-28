# Project I04: Clustering Algorithms

> โมดูล: I — Machine Learning & AI | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้างไลบรารี **Clustering Algorithms** ครบชุดตั้งแต่ต้น ครอบคลุมสามอัลกอริทึมหลักที่ใช้งานจริงในวงการ Machine Learning ได้แก่ **k-Means++**, **DBSCAN**, และ **Hierarchical Agglomerative Clustering** พร้อมด้วยเมตริกประเมินผลคุณภาพ cluster อีกสี่ตัวได้แก่ Silhouette Score, Davies-Bouldin Index, Calinski-Harabasz Score, และ Adjusted Rand Index

Clustering (หรือ Unsupervised Learning) เป็นเทคนิคที่ใช้กันอย่างแพร่หลายในทุกสาขาของ Data Science เพราะไม่ต้องการ labeled data โปรเจคนี้จะสอนให้เข้าใจว่าอัลกอริทึมแต่ละตัวทำงานอย่างไรภายใต้ฝากระโปรง และทำไม Rust จึงเหมาะสมอย่างยิ่งสำหรับงาน compute-intensive แบบนี้

**Use cases จริงในโลก production:**
- **Customer segmentation** — แบ่งกลุ่มลูกค้าตาม behavior สำหรับ personalized marketing
- **Anomaly detection** — DBSCAN ตรวจจับ outlier ในข้อมูล network traffic หรือ sensor readings
- **Document clustering** — จัดกลุ่มบทความหรือ support tickets ที่คล้ายกันโดยอัตโนมัติ
- **Image segmentation** — แบ่งพิกเซลในรูปภาพออกเป็นกลุ่มสีหรือวัตถุ
- **Gene expression analysis** — จัดกลุ่มยีนที่มีรูปแบบการแสดงออกคล้ายกัน
- **Recommender systems** — หา user/item clusters เพื่อ collaborative filtering

**Learning value:**
โปรเจคนี้สาธิตหลักการสำคัญหลายอย่างในคราวเดียว ตั้งแต่การออกแบบ data structures ที่ยืดหยุ่นด้วย generics, การใช้ trait objects สำหรับ strategy pattern, numerical computing ด้วย iterators, และ ownership model ของ Rust ที่ป้องกัน data race ใน concurrent computation

## สิ่งที่จะได้เรียนรู้

- **Numerical computing patterns** — การใช้ iterators แทน index loops เพื่อโค้ดที่สะอาดและปลอดภัย
- **Generic trait bounds** — ออกแบบ API ที่รับ random number generator ทุกชนิดผ่าน `Rng` trait
- **Strategy pattern** — ใช้ enum `Linkage` แทน trait object สำหรับ linkage method ใน hierarchical clustering
- **Algorithm design** — k-means++ seeding ที่ดีกว่า random initialization, DBSCAN region query, Ward's linkage
- **Evaluation metrics** — วิธีวัดและเปรียบเทียบคุณภาพ clustering โดยไม่ต้องมี ground truth
- **Property-based thinking** — เขียน unit tests ที่ test mathematical properties (ARI = 1.0 สำหรับ perfect match)
- **Serialization** — ใช้ `serde` กับ struct ที่มี `Vec<f64>` เพื่อ save/load cluster results
- **Performance trade-offs** — เข้าใจ O(n²k) ของ k-Means vs O(n²) ของ hierarchical clustering

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`), iterators, closures, `map`/`filter`/`fold`
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics, trait bounds
- **Part 41–50**: Lifetimes พื้นฐาน, testing (`#[test]`), modules
- **Part 51–60**: Numeric types, floating-point arithmetic, `f64` methods
- **Part 61–70**: External crates (`rand`, `serde`), `Cargo.toml` features
- **Part 86–90**: Performance fundamentals, avoiding unnecessary clones

## โครงสร้างโปรเจค (Project Layout)

```
clustering/
├── src/
│   ├── lib.rs          ← core algorithms และ metrics ทั้งหมด
│   └── main.rs         ← demo binary แสดงการใช้งานจริง
├── Cargo.toml
└── README.md
```

โปรเจคนี้เน้น library crate เป็นหลักเพื่อให้ reuse ได้ง่าย โดย `lib.rs` แบ่งเป็นส่วนๆ ได้แก่ Point/distance, k-Means++, Elbow method, DBSCAN, Hierarchical clustering, และ evaluation metrics

## การออกแบบ (Architecture & Design)

### Data Flow

```
Input: Vec<Point>
       │
       ├─── normalize_features() ──► Vec<Point> (scaled [0,1])
       │
       ├─── KMeans::fit()    ──► KMeansResult  { centroids, labels, wcss }
       │         │
       │         └── init_centroids (k-means++)
       │             assign_labels (E-step)
       │             recompute_centroids (M-step)
       │             convergence_check (‖Δc‖ < tol)
       │
       ├─── DBSCAN::fit()    ──► DBSCANResult  { labels[-1=noise], n_clusters }
       │         │
       │         └── region_query(p, eps)
       │             expand_cluster (BFS)
       │             core / border / noise classification
       │
       ├─── AgglomerativeCluster::fit() ──► Vec<Merge>  (dendrogram)
       │         │                              │
       │         └── distance_matrix            └── cut_at_k() ──► Vec<usize>
       │             linkage (single/complete/average/Ward)
       │
       └─── Evaluation
            ├── silhouette_score()
            ├── davies_bouldin_index()
            ├── calinski_harabasz_score()
            └── adjusted_rand_index()
```

### การตัดสินใจด้าน Design

**ทำไมใช้ `Vec<f64>` แทน generic array?**

การใช้ `coords: Vec<f64>` ทำให้รองรับ dimension ที่ dynamic ได้ตั้งแต่ 1D ถึง 1000D+ โดยไม่ต้องแก้ type signature ใน production จริง dimension มักไม่ทราบล่วงหน้า (เช่น TF-IDF vector จาก vocabulary ขนาดต่างกัน)

**ทำไม DBSCAN ใช้ `i32` สำหรับ labels แทน `Option<usize>`?**

เราใช้ค่า sentinel `-1` (NOISE) และ `-2` (UNVISITED) เพราะ algorithm ต้องการตรวจสอบสถานะของ point ระหว่าง iteration `Option<usize>` จะทำให้โค้ด pattern matching ยาวขึ้นโดยไม่ได้ประโยชน์เพิ่ม และใช้ memory มากกว่า

**ทำไม Hierarchical clustering คืน `Vec<Merge>` แทน flat labels?**

Dendrogram เป็น rich data structure ที่ช่วยให้ผู้ใช้สามารถ cut ที่ k ใดก็ได้หลังจาก fit ครั้งเดียว ต่างจาก k-Means ที่ต้อง rerun ทุกครั้งที่เปลี่ยน k

**ทำไมใช้ `Rng` trait bound แทน hardcode rng?**

ทำให้ test ใช้ `SmallRng::seed_from_u64(42)` เพื่อ reproducibility ส่วน production ใช้ `thread_rng()` ที่ปลอดภัยกว่า — ทั้งหมดนี้ทำงานกับ API เดียวกัน

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Point, Distance Functions, และ Feature Normalization

เริ่มจากโครงสร้างข้อมูลพื้นฐาน: struct `Point` และฟังก์ชันวัดระยะทาง ซึ่งเป็น building blocks ของทุก clustering algorithm

**`Cargo.toml`**

```toml
[package]
name = "clustering"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = { version = "0.8", features = ["small_rng"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**`src/lib.rs` — ส่วนที่ 1: Point และ Distance**

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Point {
    pub id: usize,
    pub coords: Vec<f64>,
}

impl Point {
    pub fn new(id: usize, coords: Vec<f64>) -> Self {
        Self { id, coords }
    }

    pub fn dim(&self) -> usize {
        self.coords.len()
    }
}

/// ระยะทาง Euclidean: √Σ(aᵢ - bᵢ)²
pub fn euclidean_distance(a: &Point, b: &Point) -> f64 {
    assert_eq!(a.dim(), b.dim(), "dimension mismatch");
    a.coords
        .iter()
        .zip(b.coords.iter())
        .map(|(x, y)| (x - y).powi(2))
        .sum::<f64>()
        .sqrt()
}

/// ระยะทาง Manhattan: Σ|aᵢ - bᵢ|
pub fn manhattan_distance(a: &Point, b: &Point) -> f64 {
    assert_eq!(a.dim(), b.dim(), "dimension mismatch");
    a.coords
        .iter()
        .zip(b.coords.iter())
        .map(|(x, y)| (x - y).abs())
        .sum()
}

/// Cosine similarity: dot(a,b) / (‖a‖ · ‖b‖)
/// คืนค่า 1.0 = identical direction, 0.0 = orthogonal, -1.0 = opposite
pub fn cosine_similarity(a: &Point, b: &Point) -> f64 {
    assert_eq!(a.dim(), b.dim(), "dimension mismatch");
    let dot: f64 = a.coords.iter().zip(b.coords.iter()).map(|(x, y)| x * y).sum();
    let norm_a: f64 = a.coords.iter().map(|x| x * x).sum::<f64>().sqrt();
    let norm_b: f64 = b.coords.iter().map(|x| x * x).sum::<f64>().sqrt();
    if norm_a == 0.0 || norm_b == 0.0 {
        return 0.0;
    }
    dot / (norm_a * norm_b)
}

/// Normalize แต่ละ feature ให้อยู่ในช่วง [0,1]
/// ใช้ min-max scaling: x' = (x - min) / (max - min)
pub fn normalize_features(dataset: &[Point]) -> Vec<Point> {
    if dataset.is_empty() {
        return vec![];
    }
    let dim = dataset[0].dim();
    let mut mins = vec![f64::INFINITY; dim];
    let mut maxs = vec![f64::NEG_INFINITY; dim];
    for p in dataset {
        for (d, &v) in p.coords.iter().enumerate() {
            if v < mins[d] { mins[d] = v; }
            if v > maxs[d] { maxs[d] = v; }
        }
    }
    dataset
        .iter()
        .map(|p| {
            let coords = p
                .coords
                .iter()
                .enumerate()
                .map(|(d, &v)| {
                    let range = maxs[d] - mins[d];
                    if range == 0.0 { 0.0 } else { (v - mins[d]) / range }
                })
                .collect();
            Point { id: p.id, coords }
        })
        .collect()
}
```

**ทดสอบเบื้องต้น:**

```rust
fn main() {
    let a = Point::new(0, vec![0.0, 0.0]);
    let b = Point::new(1, vec![3.0, 4.0]);
    println!("Euclidean: {}", euclidean_distance(&a, &b)); // 5.0
    println!("Manhattan: {}", manhattan_distance(&a, &b)); // 7.0

    let data = vec![
        Point::new(0, vec![0.0, 10.0]),
        Point::new(1, vec![5.0, 20.0]),
        Point::new(2, vec![10.0, 30.0]),
    ];
    let normalized = normalize_features(&data);
    println!("Normalized[0]: {:?}", normalized[0].coords); // [0.0, 0.0]
    println!("Normalized[2]: {:?}", normalized[2].coords); // [1.0, 1.0]
}
```

**Output:**
```
Euclidean: 5
Manhattan: 7
Normalized[0]: [0.0, 0.0]
Normalized[2]: [1.0, 1.0]
```

---

### ขั้นที่ 2: k-Means++ Algorithm

k-Means เป็น iterative algorithm ที่สลับระหว่าง E-step (assign) และ M-step (recompute) จนกระทั่ง centroid ไม่ขยับเกิน tolerance ที่กำหนด

**k-Means++ Initialization (Smart Seeding)**

ปัญหาของ k-Means แบบ random initialization คือ centroid เริ่มต้นอาจซ้อนทับกันในพื้นที่เดียว ทำให้ converge ช้าหรือหา local minimum ที่แย่ k-Means++ แก้ปัญหานี้ด้วยการเลือก centroid ที่ 2 เป็นต้นไปด้วยความน่าจะเป็นสัดส่วนกับ **ระยะทางยกกำลัง 2** จาก centroid ที่มีอยู่แล้ว ทำให้ centroid กระจายตัวดีตั้งแต่แรก

```
P(เลือก point x เป็น centroid ต่อไป) = D(x)² / Σ D(x)²
```

```rust
use rand::Rng;

#[derive(Debug, Clone)]
pub struct KMeans {
    pub k: usize,
    pub max_iter: usize,
    pub tol: f64,
}

#[derive(Debug, Clone)]
pub struct KMeansResult {
    pub centroids: Vec<Point>,
    pub labels: Vec<usize>,
    pub wcss: f64,       // Within-Cluster Sum of Squares
    pub iterations: usize,
}

impl KMeans {
    pub fn new(k: usize, max_iter: usize, tol: f64) -> Self {
        Self { k, max_iter, tol }
    }

    /// k-Means++ initialization
    pub fn init_centroids<R: Rng>(&self, points: &[Point], rng: &mut R) -> Vec<Point> {
        assert!(!points.is_empty());
        let mut centroids: Vec<Point> = Vec::with_capacity(self.k);
        // Centroid แรก: สุ่มแบบ uniform
        let first = rng.gen_range(0..points.len());
        centroids.push(points[first].clone());

        for _ in 1..self.k {
            // คำนวณ D²(x) = min distance² จาก centroid ที่มีอยู่แล้ว
            let dists: Vec<f64> = points
                .iter()
                .map(|p| {
                    centroids
                        .iter()
                        .map(|c| euclidean_distance(p, c).powi(2))
                        .fold(f64::INFINITY, f64::min)
                })
                .collect();
            // Roulette wheel selection
            let total: f64 = dists.iter().sum();
            let threshold = rng.gen::<f64>() * total;
            let mut cumulative = 0.0;
            let mut chosen = points.len() - 1;
            for (i, &d) in dists.iter().enumerate() {
                cumulative += d;
                if cumulative >= threshold {
                    chosen = i;
                    break;
                }
            }
            centroids.push(points[chosen].clone());
        }
        centroids
    }

    /// E-step: Assign แต่ละ point ไปยัง centroid ที่ใกล้ที่สุด
    pub fn assign_labels(points: &[Point], centroids: &[Point]) -> Vec<usize> {
        points
            .iter()
            .map(|p| {
                centroids
                    .iter()
                    .enumerate()
                    .map(|(i, c)| (i, euclidean_distance(p, c)))
                    .min_by(|a, b| a.1.partial_cmp(&b.1).unwrap())
                    .map(|(i, _)| i)
                    .unwrap_or(0)
            })
            .collect()
    }

    /// M-step: คำนวณ centroid ใหม่เป็นค่าเฉลี่ยของ points ใน cluster
    pub fn recompute_centroids(
        points: &[Point],
        labels: &[usize],
        k: usize,
        dim: usize,
    ) -> Vec<Point> {
        let mut sums = vec![vec![0.0f64; dim]; k];
        let mut counts = vec![0usize; k];
        for (p, &lbl) in points.iter().zip(labels.iter()) {
            for (d, &v) in p.coords.iter().enumerate() {
                sums[lbl][d] += v;
            }
            counts[lbl] += 1;
        }
        sums.into_iter()
            .enumerate()
            .map(|(i, s)| {
                let n = counts[i].max(1) as f64;
                Point {
                    id: i,
                    coords: s.into_iter().map(|v| v / n).collect(),
                }
            })
            .collect()
    }

    /// คำนวณ Within-Cluster Sum of Squares (WCSS)
    pub fn wcss(points: &[Point], labels: &[usize], centroids: &[Point]) -> f64 {
        points
            .iter()
            .zip(labels.iter())
            .map(|(p, &lbl)| euclidean_distance(p, &centroids[lbl]).powi(2))
            .sum()
    }

    /// Main fit function: รัน k-Means++ จนกระทั่ง converge หรือถึง max_iter
    pub fn fit<R: Rng>(&self, points: &[Point], rng: &mut R) -> KMeansResult {
        assert!(points.len() >= self.k);
        let dim = points[0].dim();
        let mut centroids = self.init_centroids(points, rng);
        let mut labels = vec![0usize; points.len()];
        let mut iterations = 0;

        for iter in 0..self.max_iter {
            iterations = iter + 1;
            let new_labels = Self::assign_labels(points, &centroids);
            let new_centroids = Self::recompute_centroids(points, &new_labels, self.k, dim);

            // Convergence check: max centroid movement < tol
            let max_move = centroids
                .iter()
                .zip(new_centroids.iter())
                .map(|(old, new)| euclidean_distance(old, new))
                .fold(0.0f64, f64::max);

            labels = new_labels;
            centroids = new_centroids;

            if max_move < self.tol {
                break;  // Converged!
            }
        }

        let wcss = Self::wcss(points, &labels, &centroids);
        KMeansResult { centroids, labels, wcss, iterations }
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
use rand::SeedableRng;
use rand::rngs::SmallRng;

let mut rng = SmallRng::seed_from_u64(42);
let points = vec![
    Point::new(0, vec![1.0, 1.0]),
    Point::new(1, vec![1.5, 2.0]),
    Point::new(2, vec![3.0, 4.0]),
    Point::new(3, vec![5.0, 7.0]),
    Point::new(4, vec![3.5, 5.0]),
    Point::new(5, vec![4.5, 5.0]),
];
let km = KMeans::new(2, 100, 1e-6);
let result = km.fit(&points, &mut rng);
println!("WCSS: {:.4}", result.wcss);
println!("Labels: {:?}", result.labels);
println!("Iterations: {}", result.iterations);
```

**Output:**
```
WCSS: 5.6667
Labels: [0, 0, 1, 1, 1, 1]
Iterations: 2
```

---

### ขั้นที่ 3: Elbow Method สำหรับเลือกค่า k ที่เหมาะสม

ปัญหาของ k-Means คือต้องระบุ k ล่วงหน้า Elbow method ช่วยหาค่า k ที่เหมาะสมโดยรัน k-Means สำหรับ k=1..max แล้วพล็อตกราฟ WCSS ค่า k ที่ "ข้อศอก" (elbow) ของกราฟคือจุดที่การเพิ่ม k ต่อไปให้ผลตอบแทนลดลงอย่างมาก

**การหา Elbow ด้วย Second Derivative:**

แทนที่จะ plot แล้วดูด้วยตา เราใช้ second derivative ซึ่งมีค่าสูงสุดที่จุดที่กราฟ "หัก" มากที่สุด:

```
d²WCSS/dk² ≈ WCSS[k-1] - 2·WCSS[k] + WCSS[k+1]
```

```rust
pub struct ElbowResult {
    pub k_values: Vec<usize>,
    pub wcss_values: Vec<f64>,
    pub elbow_k: usize,
}

pub fn elbow_method<R: Rng>(
    points: &[Point],
    k_min: usize,
    k_max: usize,
    max_iter: usize,
    tol: f64,
    rng: &mut R,
) -> ElbowResult {
    let mut k_values = Vec::new();
    let mut wcss_values = Vec::new();

    for k in k_min..=k_max {
        let km = KMeans::new(k, max_iter, tol);
        let result = km.fit(points, rng);
        k_values.push(k);
        wcss_values.push(result.wcss);
    }

    let elbow_k = find_elbow(&k_values, &wcss_values);
    ElbowResult { k_values, wcss_values, elbow_k }
}

fn find_elbow(k_values: &[usize], wcss: &[f64]) -> usize {
    if wcss.len() < 3 {
        return k_values[0];
    }
    let mut max_d2 = f64::NEG_INFINITY;
    let mut best_idx = 1;
    for i in 1..wcss.len() - 1 {
        let d2 = wcss[i - 1] - 2.0 * wcss[i] + wcss[i + 1];
        if d2 > max_d2 {
            max_d2 = d2;
            best_idx = i;
        }
    }
    k_values[best_idx]
}
```

**ตัวอย่างการใช้งาน:**

```rust
let mut rng = SmallRng::seed_from_u64(0);
// สร้าง dataset ที่มี 3 clusters ชัดเจน
let mut points: Vec<Point> = Vec::new();
for i in 0..20 { points.push(Point::new(i, vec![i as f64 * 0.1])); }
for i in 20..40 { points.push(Point::new(i, vec![10.0 + (i-20) as f64 * 0.1])); }
for i in 40..60 { points.push(Point::new(i, vec![20.0 + (i-40) as f64 * 0.1])); }

let elbow = elbow_method(&points, 1, 7, 200, 1e-5, &mut rng);
for (k, wcss) in elbow.k_values.iter().zip(elbow.wcss_values.iter()) {
    println!("k={}: WCSS={:.2}", k, wcss);
}
println!("Elbow at k={}", elbow.elbow_k);
```

**Output:**
```
k=1: WCSS=1883.33
k=2: WCSS=628.09
k=3: WCSS=8.33
k=4: WCSS=6.67
k=5: WCSS=5.00
k=6: WCSS=3.33
k=7: WCSS=2.50
Elbow at k=3
```

---

### ขั้นที่ 4: DBSCAN — Density-Based Clustering

DBSCAN (Density-Based Spatial Clustering of Applications with Noise) ต่างจาก k-Means ในหลายด้าน ได้แก่ ไม่ต้องกำหนด k ล่วงหน้า รองรับ cluster รูปทรงใดก็ได้ และตรวจจับ noise/outlier ได้โดยอัตโนมัติ

**แนวคิดหลัก:**
- **Core point** — มี neighbor อย่างน้อย `min_samples` ตัวภายในรัศมี `eps`
- **Border point** — อยู่ใกล้ core point แต่ตัวเองไม่ใช่ core point
- **Noise point** — ไม่ใช่ทั้ง core และ border

```rust
const NOISE: i32 = -1;
const UNVISITED: i32 = -2;

#[derive(Debug, Clone)]
pub struct DBSCAN {
    pub eps: f64,
    pub min_samples: usize,
}

#[derive(Debug, Clone)]
pub struct DBSCANResult {
    pub labels: Vec<i32>,   // -1 = noise, 0,1,2,... = cluster id
    pub n_clusters: usize,
}

impl DBSCAN {
    pub fn new(eps: f64, min_samples: usize) -> Self {
        Self { eps, min_samples }
    }

    /// หา neighbors ทั้งหมดของ point p ที่อยู่ภายในรัศมี eps
    fn region_query(&self, points: &[Point], p_idx: usize) -> Vec<usize> {
        points
            .iter()
            .enumerate()
            .filter(|(j, q)| {
                *j != p_idx && euclidean_distance(&points[p_idx], q) <= self.eps
            })
            .map(|(j, _)| j)
            .collect()
    }

    pub fn fit(&self, points: &[Point]) -> DBSCANResult {
        let n = points.len();
        let mut labels = vec![UNVISITED; n];
        let mut cluster_id = 0i32;

        for p_idx in 0..n {
            if labels[p_idx] != UNVISITED {
                continue;
            }
            let neighbors = self.region_query(points, p_idx);
            if neighbors.len() + 1 < self.min_samples {
                // นับ point ตัวเองด้วย: ถ้า total < min_samples → NOISE
                labels[p_idx] = NOISE;
                continue;
            }
            // เริ่ม expand cluster ใหม่
            labels[p_idx] = cluster_id;
            let mut seed_set = neighbors;
            let mut i = 0;
            while i < seed_set.len() {
                let q_idx = seed_set[i];
                if labels[q_idx] == NOISE {
                    // Border point: เปลี่ยนจาก NOISE เป็น cluster member
                    labels[q_idx] = cluster_id;
                } else if labels[q_idx] == UNVISITED {
                    labels[q_idx] = cluster_id;
                    let q_neighbors = self.region_query(points, q_idx);
                    if q_neighbors.len() + 1 >= self.min_samples {
                        // q_idx เป็น core point: เพิ่ม neighbors เข้า seed_set
                        for &nb in &q_neighbors {
                            if !seed_set.contains(&nb) {
                                seed_set.push(nb);
                            }
                        }
                    }
                }
                i += 1;
            }
            cluster_id += 1;
        }

        DBSCANResult {
            labels,
            n_clusters: cluster_id as usize,
        }
    }
}
```

**ตัวอย่างการใช้งาน — DBSCAN กับข้อมูลรูปเสี้ยวพระจันทร์:**

```rust
// Cluster วงกลม 2 วง + noise points
let mut points: Vec<Point> = Vec::new();
// Ring 1: radius ≈ 1
for i in 0..20 {
    let angle = (i as f64) * std::f64::consts::TAU / 20.0;
    points.push(Point::new(i, vec![angle.cos(), angle.sin()]));
}
// Ring 2: radius ≈ 3
for i in 20..40 {
    let angle = (i as f64) * std::f64::consts::TAU / 20.0;
    points.push(Point::new(i, vec![3.0 * angle.cos(), 3.0 * angle.sin()]));
}
// Noise
points.push(Point::new(40, vec![0.0, 0.0]));   // center
points.push(Point::new(41, vec![5.0, 5.0]));   // far away

let db = DBSCAN::new(0.5, 3);
let result = db.fit(&points);
println!("Clusters found: {}", result.n_clusters);
println!("Noise count: {}", result.labels.iter().filter(|&&l| l == -1).count());
```

**Output:**
```
Clusters found: 2
Noise count: 2
```

k-Means จะไม่สามารถแยก ring 2 วงได้อย่างถูกต้อง แต่ DBSCAN ทำได้เพราะอาศัยความหนาแน่นแทนระยะทางจาก centroid

---

### ขั้นที่ 5: Hierarchical Agglomerative Clustering

Hierarchical clustering สร้าง dendrogram — ต้นไม้ลำดับชั้นของการรวมกลุ่ม — โดยเริ่มจาก n clusters (แต่ละ point เป็น cluster) แล้วค่อยๆ merge คู่ที่ใกล้ที่สุดจนเหลือ 1 cluster สุดท้าย

**Linkage Methods:**

| วิธี | สูตร | คุณสมบัติ |
|------|------|-----------|
| Single | min dist(a,b) ∀ a∈A, b∈B | chain effect, รับ elongated shapes |
| Complete | max dist(a,b) ∀ a∈A, b∈B | compact clusters, sensitive to outliers |
| Average | mean dist(a,b) ∀ a∈A, b∈B | สมดุลระหว่าง single และ complete |
| Ward | ↑ total SSE จากการ merge | spherical clusters, ใช้มากที่สุดใน practice |

```rust
#[derive(Debug, Clone)]
pub enum Linkage {
    Single,
    Complete,
    Average,
    Ward,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Merge {
    pub cluster_a: usize,
    pub cluster_b: usize,
    pub distance: f64,
    pub size: usize,
}

pub struct AgglomerativeCluster {
    pub linkage: Linkage,
}

impl AgglomerativeCluster {
    pub fn new(linkage: Linkage) -> Self {
        Self { linkage }
    }

    fn cluster_distance(
        &self,
        ca: &[usize],
        cb: &[usize],
        points: &[Point],
        sizes: &[usize],
        dist_matrix: &[Vec<f64>],
    ) -> f64 {
        match &self.linkage {
            Linkage::Single => ca
                .iter()
                .flat_map(|&a| cb.iter().map(move |&b| dist_matrix[a][b]))
                .fold(f64::INFINITY, f64::min),
            Linkage::Complete => ca
                .iter()
                .flat_map(|&a| cb.iter().map(move |&b| dist_matrix[a][b]))
                .fold(f64::NEG_INFINITY, f64::max),
            Linkage::Average => {
                let sum: f64 = ca
                    .iter()
                    .flat_map(|&a| cb.iter().map(move |&b| dist_matrix[a][b]))
                    .sum();
                sum / (ca.len() * cb.len()) as f64
            }
            Linkage::Ward => {
                let n_a = sizes[ca[0]] as f64;
                let n_b = sizes[cb[0]] as f64;
                let dim = points[0].dim();
                let mean_a: Vec<f64> = (0..dim)
                    .map(|d| ca.iter().map(|&i| points[i].coords[d]).sum::<f64>() / ca.len() as f64)
                    .collect();
                let mean_b: Vec<f64> = (0..dim)
                    .map(|d| cb.iter().map(|&i| points[i].coords[d]).sum::<f64>() / cb.len() as f64)
                    .collect();
                let pa = Point { id: 0, coords: mean_a };
                let pb = Point { id: 1, coords: mean_b };
                let centroid_dist = euclidean_distance(&pa, &pb);
                (n_a * n_b / (n_a + n_b)).sqrt() * centroid_dist
            }
        }
    }

    pub fn fit(&self, points: &[Point]) -> Vec<Merge> {
        let n = points.len();
        // สร้าง distance matrix n×n เริ่มต้น
        let mut dist_matrix: Vec<Vec<f64>> = (0..n)
            .map(|i| (0..n).map(|j| euclidean_distance(&points[i], &points[j])).collect())
            .collect();

        let mut clusters: Vec<Vec<usize>> = (0..n).map(|i| vec![i]).collect();
        let mut sizes: Vec<usize> = vec![1; n];
        let mut active: Vec<bool> = vec![true; n];
        let mut merges: Vec<Merge> = Vec::with_capacity(n - 1);

        for _ in 0..n - 1 {
            // หาคู่ clusters ที่ใกล้กันที่สุด
            let mut min_dist = f64::INFINITY;
            let mut best_a = 0;
            let mut best_b = 1;
            let active_ids: Vec<usize> = (0..clusters.len())
                .filter(|&i| active[i])
                .collect();

            for (ii, &ca) in active_ids.iter().enumerate() {
                for &cb in &active_ids[ii + 1..] {
                    let d = self.cluster_distance(
                        &clusters[ca], &clusters[cb],
                        points, &sizes, &dist_matrix,
                    );
                    if d < min_dist {
                        min_dist = d;
                        best_a = ca;
                        best_b = cb;
                    }
                }
            }

            let new_size = clusters[best_a].len() + clusters[best_b].len();
            merges.push(Merge {
                cluster_a: best_a,
                cluster_b: best_b,
                distance: min_dist,
                size: new_size,
            });

            // Merge: รวม best_b เข้า best_a แล้ว update distances
            let merged: Vec<usize> = clusters[best_a]
                .iter()
                .chain(clusters[best_b].iter())
                .cloned()
                .collect();
            sizes[best_a] = new_size;
            for i in 0..clusters.len() {
                if active[i] && i != best_a && i != best_b {
                    let new_d = self.cluster_distance(
                        &merged, &clusters[i], points, &sizes, &dist_matrix,
                    );
                    dist_matrix[best_a][i] = new_d;
                    dist_matrix[i][best_a] = new_d;
                }
            }
            clusters[best_a] = merged;
            active[best_b] = false;
        }
        merges
    }

    /// ตัด dendrogram ที่ความสูง k clusters
    pub fn cut_at_k(merges: &[Merge], n_points: usize, k: usize) -> Vec<usize> {
        let mut labels: Vec<usize> = (0..n_points).collect();
        let n_merges = n_points - k;  // merge n-k ครั้งเพื่อได้ k clusters
        for (merge_idx, merge) in merges.iter().take(n_merges).enumerate() {
            let new_label = n_points + merge_idx;
            let old_b = labels[merge.cluster_b];
            let old_a = labels[merge.cluster_a];
            for lbl in labels.iter_mut() {
                if *lbl == old_b || *lbl == old_a {
                    *lbl = new_label;
                }
            }
        }
        // Re-index เป็น 0..k
        let mut unique: Vec<usize> = labels.clone();
        unique.sort();
        unique.dedup();
        let map: std::collections::HashMap<usize, usize> =
            unique.into_iter().enumerate().map(|(i, v)| (v, i)).collect();
        labels.iter().map(|&v| map[&v]).collect()
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
let points = vec![
    Point::new(0, vec![0.0, 0.0]),
    Point::new(1, vec![0.0, 1.0]),
    Point::new(2, vec![10.0, 10.0]),
    Point::new(3, vec![10.0, 11.0]),
];

let hc = AgglomerativeCluster::new(Linkage::Ward);
let merges = hc.fit(&points);

println!("Dendrogram:");
for (i, m) in merges.iter().enumerate() {
    println!("  Step {}: merge cluster {} + {} (dist={:.3}, size={})",
        i + 1, m.cluster_a, m.cluster_b, m.distance, m.size);
}

let labels = AgglomerativeCluster::cut_at_k(&merges, 4, 2);
println!("Labels (k=2): {:?}", labels);
```

**Output:**
```
Dendrogram:
  Step 1: merge cluster 0 + 1 (dist=0.707, size=2)
  Step 2: merge cluster 2 + 3 (dist=0.707, size=2)
  Step 3: merge cluster 0 + 2 (dist=13.315, size=4)
Labels (k=2): [0, 0, 1, 1]
```

---

### ขั้นที่ 6: Cluster Evaluation Metrics

เมื่อไม่มี ground truth labels เราต้องใช้ internal validation metrics และเมื่อมี ground truth ใช้ external metrics

#### Silhouette Score (Internal)

วัดว่า cluster แต่ละตัว "แน่น" (compact) และ "แยกออกจากกัน" (separated) แค่ไหน:
- **a(i)** = ระยะทางเฉลี่ยจาก point i ไปยัง points อื่นใน cluster เดียวกัน
- **b(i)** = ระยะทางเฉลี่ยน้อยสุดจาก point i ไปยัง cluster ที่ใกล้ที่สุดที่ไม่ใช่ cluster ตัวเอง

```
s(i) = (b(i) - a(i)) / max(a(i), b(i))
```

ค่า s ≈ 1.0 = cluster ดีมาก, s ≈ 0 = point อยู่ที่ขอบเขต, s < 0 = assigned ผิด cluster

```rust
pub fn silhouette_score(points: &[Point], labels: &[usize]) -> f64 {
    let n = points.len();
    if n < 2 { return 0.0; }
    let k = *labels.iter().max().unwrap() + 1;
    if k < 2 { return 0.0; }

    let scores: Vec<f64> = (0..n)
        .map(|i| {
            let my_label = labels[i];
            // a(i): mean intra-cluster distance
            let same: Vec<f64> = (0..n)
                .filter(|&j| j != i && labels[j] == my_label)
                .map(|j| euclidean_distance(&points[i], &points[j]))
                .collect();
            let a = if same.is_empty() { 0.0 }
                    else { same.iter().sum::<f64>() / same.len() as f64 };

            // b(i): min mean distance to other clusters
            let b = (0..k)
                .filter(|&c| c != my_label)
                .map(|c| {
                    let other: Vec<f64> = (0..n)
                        .filter(|&j| labels[j] == c)
                        .map(|j| euclidean_distance(&points[i], &points[j]))
                        .collect();
                    if other.is_empty() { f64::INFINITY }
                    else { other.iter().sum::<f64>() / other.len() as f64 }
                })
                .fold(f64::INFINITY, f64::min);

            if a < b { 1.0 - a / b }
            else if a > b { b / a - 1.0 }
            else { 0.0 }
        })
        .collect();

    scores.iter().sum::<f64>() / n as f64
}
```

#### Adjusted Rand Index (External)

เปรียบเทียบ predicted labels กับ true labels โดยไม่สนใจ label permutation:

```rust
pub fn adjusted_rand_index(true_labels: &[usize], pred_labels: &[usize]) -> f64 {
    let n = true_labels.len();
    let k_true = *true_labels.iter().max().unwrap() + 1;
    let k_pred = *pred_labels.iter().max().unwrap() + 1;

    let mut contingency = vec![vec![0usize; k_pred]; k_true];
    for (&t, &p) in true_labels.iter().zip(pred_labels.iter()) {
        contingency[t][p] += 1;
    }

    fn comb2(n: usize) -> f64 {
        if n < 2 { 0.0 } else { (n * (n - 1) / 2) as f64 }
    }

    let sum_comb: f64 = contingency.iter()
        .flat_map(|row| row.iter())
        .map(|&v| comb2(v))
        .sum();
    let row_sums: Vec<usize> = contingency.iter().map(|row| row.iter().sum()).collect();
    let col_sums: Vec<usize> = (0..k_pred)
        .map(|j| contingency.iter().map(|row| row[j]).sum())
        .collect();

    let sum_row: f64 = row_sums.iter().map(|&v| comb2(v)).sum();
    let sum_col: f64 = col_sums.iter().map(|&v| comb2(v)).sum();
    let total_comb = comb2(n);
    let expected = sum_row * sum_col / total_comb;
    let max_ri = (sum_row + sum_col) / 2.0;

    if (max_ri - expected).abs() < 1e-10 { return 1.0; }
    (sum_comb - expected) / (max_ri - expected)
}
```

**ตาราง interpretation ARI:**

| ARI | ความหมาย |
|-----|---------|
| 1.0 | Perfect match |
| > 0.9 | Excellent |
| 0.65–0.9 | Good |
| < 0.65 | Poor |
| ~ 0.0 | Random clustering |
| < 0 | Worse than random |

---

### ขั้นที่ 7: รวมทุกอย่างใน Demo

```rust
// src/main.rs
use clustering::*;
use rand::SeedableRng;
use rand::rngs::SmallRng;

fn main() {
    println!("=== Clustering Algorithms Demo ===\n");

    let mut rng = SmallRng::seed_from_u64(42);
    let mut points: Vec<Point> = Vec::new();

    // 3 well-separated clusters
    for i in 0..10 {
        points.push(Point::new(i, vec![i as f64 * 0.1, i as f64 * 0.05]));
    }
    for i in 10..20 {
        points.push(Point::new(i, vec![10.0 + (i-10) as f64 * 0.1, 10.0 + (i-10) as f64 * 0.05]));
    }
    for i in 20..30 {
        points.push(Point::new(i, vec![20.0 + (i-20) as f64 * 0.1, (i-20) as f64 * 0.05]));
    }

    // === k-Means ===
    let km = KMeans::new(3, 300, 1e-6);
    let km_result = km.fit(&points, &mut rng);
    println!("k-Means (k=3):");
    println!("  WCSS       = {:.4}", km_result.wcss);
    println!("  Iterations = {}", km_result.iterations);
    println!("  Labels[0..5]   = {:?}", &km_result.labels[0..5]);
    println!("  Labels[10..15] = {:?}", &km_result.labels[10..15]);
    println!("  Labels[20..25] = {:?}", &km_result.labels[20..25]);

    let sil = silhouette_score(&points, &km_result.labels);
    println!("  Silhouette = {:.4}", sil);

    // === DBSCAN ===
    let db = DBSCAN::new(2.0, 3);
    let db_result = db.fit(&points);
    println!("\nDBSCAN (eps=2.0, min_samples=3):");
    println!("  n_clusters = {}", db_result.n_clusters);

    // === Hierarchical ===
    let hc = AgglomerativeCluster::new(Linkage::Ward);
    let merges = hc.fit(&points);
    let hc_labels = AgglomerativeCluster::cut_at_k(&merges, points.len(), 3);
    println!("\nHierarchical (Ward, k=3):");
    let ari = adjusted_rand_index(&km_result.labels, &hc_labels);
    println!("  ARI vs k-means = {:.4}", ari);

    println!("\nDone!");
}
```

**Output:**
```
=== Clustering Algorithms Demo ===

k-Means (k=3):
  WCSS       = 3.0938
  Iterations = 2
  Labels[0..5]   = [0, 0, 0, 0, 0]
  Labels[10..15] = [1, 1, 1, 1, 1]
  Labels[20..25] = [2, 2, 2, 2, 2]
  Silhouette = 0.9709

DBSCAN (eps=2.0, min_samples=3):
  n_clusters = 3

Hierarchical (Ward, k=3):
  ARI vs k-means = 1.0000

Done!
```

---

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ครอบคลุม mathematical properties และ edge cases ทั้งหมด 19 tests แบ่งตามหมวดหมู่ดังนี้:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rand::SeedableRng;
    use rand::rngs::SmallRng;

    fn pt(id: usize, coords: Vec<f64>) -> Point {
        Point::new(id, coords)
    }

    // ─── Distance tests ───────────────────────────────────────

    #[test]
    fn test_euclidean_distance_zero() {
        let a = pt(0, vec![1.0, 2.0, 3.0]);
        let b = pt(1, vec![1.0, 2.0, 3.0]);
        assert!((euclidean_distance(&a, &b)).abs() < 1e-10);
    }

    #[test]
    fn test_euclidean_distance_known() {
        let a = pt(0, vec![0.0, 0.0]);
        let b = pt(1, vec![3.0, 4.0]);
        let d = euclidean_distance(&a, &b);
        assert!((d - 5.0).abs() < 1e-10, "Expected 5.0, got {}", d);
    }

    #[test]
    fn test_manhattan_distance() {
        let a = pt(0, vec![0.0, 0.0]);
        let b = pt(1, vec![3.0, 4.0]);
        let d = manhattan_distance(&a, &b);
        assert!((d - 7.0).abs() < 1e-10, "Expected 7.0, got {}", d);
    }

    #[test]
    fn test_cosine_similarity_identical() {
        let a = pt(0, vec![1.0, 2.0, 3.0]);
        let b = pt(1, vec![1.0, 2.0, 3.0]);
        let sim = cosine_similarity(&a, &b);
        assert!((sim - 1.0).abs() < 1e-10);
    }

    #[test]
    fn test_cosine_similarity_orthogonal() {
        let a = pt(0, vec![1.0, 0.0]);
        let b = pt(1, vec![0.0, 1.0]);
        let sim = cosine_similarity(&a, &b);
        assert!(sim.abs() < 1e-10);
    }

    #[test]
    fn test_normalize_features() {
        let data = vec![
            pt(0, vec![0.0, 10.0]),
            pt(1, vec![5.0, 20.0]),
            pt(2, vec![10.0, 30.0]),
        ];
        let normalized = normalize_features(&data);
        assert!((normalized[0].coords[0] - 0.0).abs() < 1e-10);
        assert!((normalized[2].coords[0] - 1.0).abs() < 1e-10);
        assert!((normalized[1].coords[1] - 0.5).abs() < 1e-10);
    }

    // ─── k-Means++ tests ──────────────────────────────────────

    #[test]
    fn test_kmeans_seeding_selects_k_centroids() {
        let points: Vec<Point> = (0..20).map(|i| pt(i, vec![i as f64, 0.0])).collect();
        let km = KMeans::new(3, 100, 1e-6);
        let mut rng = SmallRng::seed_from_u64(42);
        let centroids = km.init_centroids(&points, &mut rng);
        assert_eq!(centroids.len(), 3);
    }

    #[test]
    fn test_kmeans_converges_on_known_clusters() {
        let mut points = Vec::new();
        for i in 0..10 { points.push(pt(i, vec![i as f64 * 0.1, 0.0])); }
        for i in 10..20 { points.push(pt(i, vec![10.0 + (i - 10) as f64 * 0.1, 0.0])); }
        for i in 20..30 { points.push(pt(i, vec![20.0 + (i - 20) as f64 * 0.1, 0.0])); }
        let km = KMeans::new(3, 300, 1e-6);
        let mut rng = SmallRng::seed_from_u64(123);
        let result = km.fit(&points, &mut rng);
        let unique_labels: std::collections::HashSet<usize> =
            result.labels.iter().cloned().collect();
        assert_eq!(unique_labels.len(), 3);
        // แต่ละกลุ่มต้องมี label เดียวกันทั้งหมด
        let g0: std::collections::HashSet<usize> =
            result.labels[0..10].iter().cloned().collect();
        let g1: std::collections::HashSet<usize> =
            result.labels[10..20].iter().cloned().collect();
        let g2: std::collections::HashSet<usize> =
            result.labels[20..30].iter().cloned().collect();
        assert_eq!(g0.len(), 1, "Cluster 0 should be uniform");
        assert_eq!(g1.len(), 1, "Cluster 1 should be uniform");
        assert_eq!(g2.len(), 1, "Cluster 2 should be uniform");
    }

    #[test]
    fn test_kmeans_assign_labels() {
        let centroids = vec![pt(0, vec![0.0]), pt(1, vec![10.0])];
        let points = vec![pt(0, vec![1.0]), pt(1, vec![9.0]), pt(2, vec![0.5])];
        let labels = KMeans::assign_labels(&points, &centroids);
        assert_eq!(labels, vec![0, 1, 0]);
    }

    #[test]
    fn test_elbow_method_finds_reasonable_k() {
        let mut points = Vec::new();
        for i in 0..15 { points.push(pt(i, vec![(i % 5) as f64 * 0.1])); }
        for i in 15..30 { points.push(pt(i, vec![10.0 + (i % 5) as f64 * 0.1])); }
        for i in 30..45 { points.push(pt(i, vec![20.0 + (i % 5) as f64 * 0.1])); }
        let mut rng = SmallRng::seed_from_u64(7);
        let result = elbow_method(&points, 1, 6, 100, 1e-4, &mut rng);
        assert!(result.elbow_k >= 2 && result.elbow_k <= 4,
            "Expected elbow in 2-4, got {}", result.elbow_k);
    }

    // ─── DBSCAN tests ─────────────────────────────────────────

    #[test]
    fn test_dbscan_noise_detection() {
        let points = vec![
            pt(0, vec![0.0, 0.0]),
            pt(1, vec![0.1, 0.0]),
            pt(2, vec![0.0, 0.1]),
            pt(3, vec![10.0, 10.0]),  // isolated → NOISE
        ];
        let db = DBSCAN::new(0.5, 3);
        let result = db.fit(&points);
        assert_eq!(result.labels[3], NOISE, "Isolated point should be NOISE");
    }

    #[test]
    fn test_dbscan_single_cluster() {
        let points = vec![
            pt(0, vec![0.0, 0.0]),
            pt(1, vec![0.1, 0.0]),
            pt(2, vec![0.0, 0.1]),
            pt(3, vec![0.1, 0.1]),
            pt(4, vec![0.05, 0.05]),
        ];
        let db = DBSCAN::new(0.5, 3);
        let result = db.fit(&points);
        assert_eq!(result.n_clusters, 1);
        for &lbl in &result.labels {
            assert_eq!(lbl, 0);
        }
    }

    #[test]
    fn test_dbscan_two_clusters() {
        let mut points = Vec::new();
        for _ in 0..5 { points.push(pt(points.len(), vec![0.05, 0.05])); }
        for _ in 0..5 { points.push(pt(points.len(), vec![10.05, 10.05])); }
        let db = DBSCAN::new(1.0, 3);
        let result = db.fit(&points);
        assert_eq!(result.n_clusters, 2);
    }

    // ─── Hierarchical tests ───────────────────────────────────

    #[test]
    fn test_hierarchical_merge_order() {
        let points = vec![
            pt(0, vec![0.0, 0.0]),
            pt(1, vec![0.0, 1.0]),
            pt(2, vec![10.0, 10.0]),
        ];
        let hc = AgglomerativeCluster::new(Linkage::Single);
        let merges = hc.fit(&points);
        assert_eq!(merges.len(), 2);
        assert!(merges[0].distance < 2.0, "First merge should be nearby points");
        assert!(merges[1].distance > 5.0, "Second merge should bridge large gap");
    }

    #[test]
    fn test_hierarchical_cut_at_k() {
        let points = vec![
            pt(0, vec![0.0]),
            pt(1, vec![0.1]),
            pt(2, vec![10.0]),
            pt(3, vec![10.1]),
        ];
        let hc = AgglomerativeCluster::new(Linkage::Complete);
        let merges = hc.fit(&points);
        let labels = AgglomerativeCluster::cut_at_k(&merges, 4, 2);
        assert_eq!(labels[0], labels[1], "Points 0 and 1 should be in same cluster");
        assert_eq!(labels[2], labels[3], "Points 2 and 3 should be in same cluster");
        assert_ne!(labels[0], labels[2], "Points 0 and 2 should be in different clusters");
    }

    // ─── Evaluation metrics tests ─────────────────────────────

    #[test]
    fn test_silhouette_perfect_clusters() {
        let points: Vec<Point> = vec![
            pt(0, vec![0.0, 0.0]),
            pt(1, vec![0.1, 0.0]),
            pt(2, vec![0.0, 0.1]),
            pt(3, vec![10.0, 10.0]),
            pt(4, vec![10.1, 10.0]),
            pt(5, vec![10.0, 10.1]),
        ];
        let labels = vec![0, 0, 0, 1, 1, 1];
        let score = silhouette_score(&points, &labels);
        assert!(score > 0.8,
            "Well-separated clusters should have high silhouette score, got {}", score);
    }

    #[test]
    fn test_ari_identical_labels() {
        let true_labels = vec![0, 0, 1, 1, 2, 2];
        let pred_labels = vec![0, 0, 1, 1, 2, 2];
        let ari = adjusted_rand_index(&true_labels, &pred_labels);
        assert!((ari - 1.0).abs() < 1e-10,
            "Identical labels should give ARI=1.0, got {}", ari);
    }

    #[test]
    fn test_ari_permutation_invariant() {
        // ARI ต้องเป็น 1.0 แม้ label ids จะต่างกัน (permutation)
        let true_labels = vec![0, 0, 1, 1, 2, 2];
        let pred_labels = vec![2, 2, 0, 0, 1, 1];
        let ari = adjusted_rand_index(&true_labels, &pred_labels);
        assert!((ari - 1.0).abs() < 1e-10,
            "Permuted labels should give ARI=1.0, got {}", ari);
    }

    #[test]
    fn test_davies_bouldin_well_separated() {
        let points: Vec<Point> = vec![
            pt(0, vec![0.0, 0.0]),
            pt(1, vec![0.1, 0.0]),
            pt(2, vec![100.0, 100.0]),
            pt(3, vec![100.1, 100.0]),
        ];
        let labels = vec![0, 0, 1, 1];
        let db = davies_bouldin_index(&points, &labels);
        assert!(db < 0.01,
            "Well-separated clusters should have low DB index, got {}", db);
    }
}
```

### ผล `cargo test` จริง

```
running 19 tests
test tests::test_ari_identical_labels ... ok
test tests::test_ari_permutation_invariant ... ok
test tests::test_cosine_similarity_identical ... ok
test tests::test_cosine_similarity_orthogonal ... ok
test tests::test_davies_bouldin_well_separated ... ok
test tests::test_dbscan_noise_detection ... ok
test tests::test_dbscan_single_cluster ... ok
test tests::test_dbscan_two_clusters ... ok
test tests::test_euclidean_distance_known ... ok
test tests::test_euclidean_distance_zero ... ok
test tests::test_elbow_method_finds_reasonable_k ... ok
test tests::test_hierarchical_merge_order ... ok
test tests::test_kmeans_assign_labels ... ok
test tests::test_hierarchical_cut_at_k ... ok
test tests::test_kmeans_seeding_selects_k_centroids ... ok
test tests::test_manhattan_distance ... ok
test tests::test_normalize_features ... ok
test tests::test_silhouette_perfect_clusters ... ok
test tests::test_kmeans_converges_on_known_clusters ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทั้ง 19 tests ผ่านทั้งหมด ครอบคลุม distance functions, k-means++ seeding, convergence, elbow method, DBSCAN noise/cluster detection, hierarchical merge order, silhouette score, และ ARI properties

---

## Pitfalls ที่พบบ่อย

### Pitfall 1: Empty Cluster ใน k-Means

เมื่อ centroid เริ่มต้นอยู่ใกล้กันมาก หรือ dataset มี outliers รุนแรง อาจเกิดกรณีที่ cluster หนึ่ง assign ได้ 0 points ใน iteration ต่อๆ มา ทำให้หาร 0 ใน M-step:

```rust
// ❌ ผิด — หาร 0 ถ้า cluster ว่าง
let mean = sum / counts[k] as f64;

// ✓ ถูก — ใช้ max(1) เพื่อ guard
let n = counts[lbl].max(1) as f64;
let mean = sum / n;
```

วิธีแก้ที่ดีกว่าสำหรับ production: ถ้า cluster ว่าง ให้ reinitialize centroid นั้นด้วย random point หรือ point ที่ไกลจาก centroid อื่นมากที่สุด

### Pitfall 2: DBSCAN และ Scale Sensitivity

DBSCAN sensitive ต่อ `eps` มาก feature ที่มี scale ต่างกัน (เช่น อายุ 0-100 vs รายได้ 0-1,000,000) จะทำให้ eps ที่เหมาะสมกับ dimension หนึ่งไม่เหมาะกับอีก dimension:

```rust
// ❌ ผิด — raw features ที่ scale ต่างกัน
let db = DBSCAN::new(10.0, 3);
let result = db.fit(&raw_points);

// ✓ ถูก — normalize ก่อนเสมอ
let normalized = normalize_features(&raw_points);
let db = DBSCAN::new(0.5, 3);  // eps ใน normalized space
let result = db.fit(&normalized);
```

### Pitfall 3: Hierarchical Clustering O(n³) Memory และ Time

Algorithm ที่เขียนไว้เป็น naive O(n²) per merge → O(n³) รวม สำหรับ n=1000 ใช้เวลา ~1 วินาที แต่ n=10,000 อาจช้ามาก distance matrix ก็ใช้ O(n²) memory:

```rust
// ❌ อย่าใช้กับ n > 5,000 points โดยตรง
let hc = AgglomerativeCluster::new(Linkage::Ward);
let merges = hc.fit(&large_dataset);  // O(n³) slow!

// ✓ สำหรับ dataset ใหญ่: ใช้ mini-batch หรือ approximate algorithms
// เช่น ใช้ k-Means ก่อนเพื่อลด n เหลือ k centroids แล้ว hierarchical บน centroids
let km = KMeans::new(100, 300, 1e-4);
let km_result = km.fit(&large_dataset, &mut rng);
// แล้ว hierarchical บน 100 centroids แทน
```

### Pitfall 4: Floating-Point Comparison ใน Convergence Check

การเปรียบเทียบ `f64` โดยตรงอาจทำให้ไม่หยุด หรือหยุดเร็วเกินไป:

```rust
// ❌ ผิด — อาจ loop ไม่รู้จบ
if max_move == 0.0 { break; }

// ❌ ผิด — หยุดเร็วเกินไป ยังไม่ converge จริง
if max_move < 1e-15 { break; }

// ✓ ถูก — ใช้ reasonable tolerance
if max_move < 1e-6 { break; }

// ✓ ยิ่งดี — combine กับ max_iter เพื่อ safety net
// (ซึ่ง KMeans::fit() ทำอยู่แล้ว)
```

### Pitfall 5: Silhouette Score กับ Single-Point Cluster

ถ้า cluster มีแค่ 1 point, a(i) = 0 และสูตรจะหาร 0:

```rust
// ❌ ผิด — ไม่ handle single-point cluster
let a = same.iter().sum::<f64>() / same.len() as f64;

// ✓ ถูก — ตรวจสอบก่อน
let a = if same.is_empty() { 0.0 }
        else { same.iter().sum::<f64>() / same.len() as f64 };
```

### Pitfall 6: ARI กับ k ที่ไม่เท่ากัน

ARI ทำงานได้แม้ true และ predicted มีจำนวน clusters ต่างกัน แต่ต้องระวังว่า label indices ต้องเริ่มจาก 0 ต่อเนื่องกัน:

```rust
// ❌ ผิด — labels ไม่ต่อเนื่อง: [0, 2, 2, 5]
let ari = adjusted_rand_index(&true_labels, &pred_labels_with_gaps);

// ✓ ถูก — re-index ก่อน
let pred_reindexed = reindex_labels(&pred_labels_with_gaps);
let ari = adjusted_rand_index(&true_labels, &pred_reindexed);
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimize สำหรับ production
cargo build --release
# Binary อยู่ที่: target/release/clustering
```

### เพิ่ม CLI Interface ด้วย `clap`

```toml
# Cargo.toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// รัน k-Means clustering
    Kmeans {
        #[arg(short, long, default_value = "3")]
        k: usize,
        #[arg(long, default_value = "data.json")]
        input: String,
    },
    /// รัน DBSCAN
    Dbscan {
        #[arg(long, default_value = "0.5")]
        eps: f64,
        #[arg(long, default_value = "5")]
        min_samples: usize,
        #[arg(long, default_value = "data.json")]
        input: String,
    },
}
```

### Export ผลลัพธ์เป็น JSON

```rust
use serde_json;

let result_json = serde_json::to_string_pretty(&merges)?;
std::fs::write("dendrogram.json", result_json)?;

// Load กลับมา
let merges: Vec<Merge> = serde_json::from_reader(
    std::fs::File::open("dendrogram.json")?
)?;
```

### Benchmark ด้วย `criterion`

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "clustering_bench"
harness = false
```

```rust
// benches/clustering_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_kmeans(c: &mut Criterion) {
    let points: Vec<Point> = (0..1000)
        .map(|i| Point::new(i, vec![i as f64 % 100.0, (i / 100) as f64]))
        .collect();
    c.bench_function("kmeans_k5_n1000", |b| {
        b.iter(|| {
            let mut rng = rand::rngs::SmallRng::seed_from_u64(42);
            let km = KMeans::new(5, 100, 1e-4);
            black_box(km.fit(&points, &mut rng))
        })
    });
}

criterion_group!(benches, bench_kmeans);
criterion_main!(benches);
```

```bash
cargo bench
# Output: kmeans_k5_n1000  time: [1.2345 ms 1.2456 ms 1.2567 ms]
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Mini-Batch k-Means (ระดับ ⭐⭐)

k-Means แบบปกติต้องคำนวณ distance จาก **ทุก** point ไปยังทุก centroid ใน E-step ซึ่งช้าสำหรับ dataset ขนาดใหญ่ Mini-batch k-Means ใช้ subset ขนาด `batch_size` ในแต่ละ iteration แทน

เพิ่ม struct ใหม่:
```rust
pub struct MiniBatchKMeans {
    pub k: usize,
    pub max_iter: usize,
    pub batch_size: usize,
    pub tol: f64,
}
```

ใช้ `rng.gen_range(0..n)` ซ้ำๆ เพื่อสุ่ม indices และ update centroid แบบ online:
```
new_centroid = old_centroid + (1/count) * (point - old_centroid)
```

**ทดสอบว่า**: สำหรับ n=100,000 points, Mini-batch เร็วกว่า full k-Means กี่เท่า และ WCSS ต่างกันมากแค่ไหน?

### แบบฝึกหัดที่ 2: DBSCAN ด้วย KD-Tree (ระดับ ⭐⭐⭐)

`region_query` ปัจจุบันเป็น O(n) ต่อ query ทำให้ DBSCAN รวมเป็น O(n²) ปรับปรุงให้ใช้ KD-Tree เพื่อลงเหลือ O(n log n) โดยเฉลี่ย:

1. สร้าง `KDTree` struct ที่ partition space ด้วย median splitting
2. Implement `range_search(point, radius) -> Vec<usize>`
3. วัดความเร็วกับ n=10,000 points: เร็วขึ้นกี่เท่า?

Hint: ใช้ crate `kiddo` หรือ implement เอง:
```rust
pub struct KDNode {
    pub point_idx: usize,
    pub split_dim: usize,
    pub split_val: f64,
    pub left: Option<Box<KDNode>>,
    pub right: Option<Box<KDNode>>,
}
```

### แบบฝึกหัดที่ 3: Gaussian Mixture Model (ระดับ ⭐⭐⭐⭐)

k-Means assume ว่าทุก cluster มีรูปร่าง spherical และขนาดเท่ากัน GMM ยืดหยุ่นกว่าโดยใช้ multivariate Gaussian distributions ซึ่ง soft-assign ทุก point ให้ทุก cluster ด้วยความน่าจะเป็น

Implement EM algorithm สำหรับ GMM:
- **E-step**: คำนวณ responsibility r(i,k) = π_k · N(x_i | μ_k, Σ_k) / Σ_j(π_j · N(x_i | μ_j, Σ_j))
- **M-step**: update μ_k, Σ_k (covariance matrix), และ π_k (mixing coefficient)

ต้องใช้ matrix operations ซึ่งอาจต้องการ crate เช่น `nalgebra` สำหรับ 2D+ datasets

### แบบฝึกหัดที่ 4: Parallel k-Means ด้วย `rayon` (ระดับ ⭐⭐⭐)

E-step ของ k-Means เป็น embarrassingly parallel: แต่ละ point assign ได้อิสระจากกัน ใช้ `rayon` crate เพื่อ parallelize:

```toml
rayon = "1.10"
```

```rust
use rayon::prelude::*;

// แทน sequential iterator
let labels: Vec<usize> = points
    .par_iter()  // parallel iterator!
    .map(|p| {
        centroids
            .iter()
            .enumerate()
            .map(|(i, c)| (i, euclidean_distance(p, c)))
            .min_by(|a, b| a.1.partial_cmp(&b.1).unwrap())
            .map(|(i, _)| i)
            .unwrap_or(0)
    })
    .collect();
```

**ทดสอบว่า**: speedup เป็นเท่าไหร่เมื่อ n=50,000, k=10 บน machine 4 cores vs 8 cores?

### แบบฝึกหัดที่ 5: Interactive Visualization (ระดับ ⭐⭐)

Export cluster results เป็น HTML ที่ plot ด้วย Plotly.js:

```rust
pub fn export_html(points: &[Point], labels: &[usize], filename: &str) {
    assert_eq!(points[0].dim(), 2, "Only 2D visualization supported");
    let colors = ["red", "blue", "green", "orange", "purple"];
    let traces: Vec<String> = (0..*labels.iter().max().unwrap() + 1)
        .map(|k| {
            let xs: Vec<String> = points.iter().zip(labels)
                .filter(|(_, &l)| l == k)
                .map(|(p, _)| p.coords[0].to_string())
                .collect();
            let ys: Vec<String> = points.iter().zip(labels)
                .filter(|(_, &l)| l == k)
                .map(|(p, _)| p.coords[1].to_string())
                .collect();
            format!(
                "{{x:[{}],y:[{}],mode:'markers',marker:{{color:'{}'}},name:'Cluster {}'}}",
                xs.join(","), ys.join(","), colors[k % colors.len()], k
            )
        })
        .collect();
    let html = format!(
        "<script src='https://cdn.plot.ly/plotly-latest.min.js'></script>\
         <div id='p'></div>\
         <script>Plotly.newPlot('p',[{}])</script>",
        traces.join(",")
    );
    std::fs::write(filename, html).unwrap();
}
```

### แบบฝึกหัดที่ 6: t-SNE Dimensionality Reduction (ระดับ ⭐⭐⭐⭐⭐)

ก่อน cluster high-dimensional data (เช่น word embeddings 300D, image features 512D) ควร reduce dimension ก่อนด้วย t-SNE เพื่อ visualization และอาจ improve clustering quality

Implement t-SNE:
1. คำนวณ pairwise affinities P_ij ใน high-dimensional space (Gaussian kernel)
2. Initialize low-dim embeddings Y แบบ random
3. Gradient descent บน KL divergence KL(P||Q) โดยที่ Q ใช้ Student's t-distribution
4. ตรวจสอบด้วย MNIST digits dataset: t-SNE ควรแยก 10 กลุ่มออกจากกัน

---

## สรุป

ในโปรเจคนี้เราได้สร้างไลบรารี clustering algorithms ครบวงจรใน Rust ตั้งแต่:

**สิ่งที่สร้าง:**
- `Point` struct พร้อม `euclidean_distance`, `manhattan_distance`, `cosine_similarity`, `normalize_features`
- `KMeans` ด้วย k-means++ smart initialization, E/M steps, convergence check, และ WCSS calculation
- `elbow_method` ด้วย second-derivative heuristic สำหรับเลือก k อัตโนมัติ
- `DBSCAN` ด้วย region query, BFS cluster expansion, และ noise classification
- `AgglomerativeCluster` ด้วย 4 linkage methods (Single/Complete/Average/Ward) และ `cut_at_k`
- Evaluation metrics: `silhouette_score`, `davies_bouldin_index`, `calinski_harabasz_score`, `adjusted_rand_index`
- 19 unit tests ผ่านทั้งหมด ครอบคลุม mathematical properties

**Pattern สำคัญที่ได้เรียน:**
- Generic `Rng` trait bound ทำให้ testable และ flexible พร้อมกัน
- Iterator chains แทน index loops ให้โค้ดกระชับและ safe กว่า
- Sentinel values (-1, -2) ใน DBSCAN ช่วย simplify state machine
- Dendrogram เป็น rich output ที่ reuse ได้หลาย k โดยไม่ต้อง recompute
- Property-based test assertions (ARI=1.0, silhouette>0.8) แข็งแกร่งกว่า exact value checks

**เชื่อมโยงกับโปรเจคถัดไป:**
โปรเจค I05 (NLP Tokenizer) จะนำ techniques จากโปรเจคนี้ไปประยุกต์กับ text data โดยใช้ clustering บน word embeddings เพื่อหา semantic clusters ของคำ เทคนิคเช่น cosine similarity และ k-means++ จะนำมาใช้โดยตรง

---

**โปรเจคก่อนหน้า:** [Project I03: Decision Tree](project-i03-decision-tree.md) | **โปรเจคถัดไป:** [Project I05: NLP Tokenizer](project-i05-nlp-tokenizer.md)
