# Project I10: Graph Neural Network จาก Scratch

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Graph Neural Network (GNN)** แบบสมบูรณ์ด้วย Pure Rust โดยไม่พึ่ง ML framework ใด ๆ เราจะครอบคลุมตั้งแต่ graph representation พื้นฐาน ไปจนถึง GCN layer, GraphSAGE aggregation, node classification, และ link prediction

Graph Neural Network คือสถาปัตยกรรม deep learning ที่ออกแบบมาเฉพาะสำหรับข้อมูลที่มีโครงสร้างเป็น graph เช่น โครงข่ายสังคม (social networks), โมเลกุลทางเคมี (molecular graphs), ระบบแนะนำ (recommendation systems), โครงสร้างของ knowledge graph และ citation network ความพิเศษของ GNN คือความสามารถในการเรียนรู้ node embeddings โดยการส่ง messages ผ่าน edge ทำให้แต่ละ node ได้รับข้อมูลจาก neighborhood ของตัวเอง

**Use cases จริงในโลก production:**
- Pinterest ใช้ PinSage (variant ของ GraphSAGE) สำหรับ recommendation engine ที่มี 3 พันล้าน nodes
- Google Maps ใช้ GNN วิเคราะห์ traffic flow บน road network
- Drug discovery ใช้ GNN ทำนาย molecular properties และค้นหายาใหม่
- Fraud detection ใช้ GNN ตรวจจับ pattern ผิดปกติใน transaction network
- Code analysis ใช้ GNN วิเคราะห์ call graph และ data flow graph

โปรเจคนี้ครอบคลุม **Graph representation** (dense adjacency + CSR format), **Normalized adjacency matrix** (D^{-1/2} A D^{-1/2}), **Message passing framework** (sum/mean/max aggregation), **GCN layers** พร้อม Xavier initialization, **Two-layer GCN** สำหรับ node classification พร้อม cross-entropy loss, **GraphSAGE** ด้วย mean/max aggregator, **Link prediction** ด้วย dot product และ binary cross-entropy และ **neighborhood sampling** สำหรับ mini-batch training ทั้งหมดนี้มี 28 unit tests ที่ผ่านจริง

## สิ่งที่จะได้เรียนรู้

- **Graph representation ใน Rust** — adjacency list, edge list, CSR format พร้อม type-safe index
- **Matrix operations** — `matmul` แบบ flat `Vec<f32>` row-major, normalized adjacency D^{-1/2} A D^{-1/2}
- **Message passing paradigm** — aggregate(neighbors) → message, update(self, message) → new_features
- **GCN forward pass** — H' = σ(D^{-1/2} A D^{-1/2} H W) ทั้งหมดด้วย raw matrix ops
- **Xavier initialization** — เหตุผลที่ต้องใช้ และวิธีคำนวณ scale ที่ถูกต้อง
- **GraphSAGE inductive learning** — concat self + aggregated → linear → ReLU
- **Link prediction** — dot product score, hadamard product, binary cross-entropy, negative sampling
- **Trait-based design** — วิธีออกแบบ `MessagePassing` ให้ swap aggregator ได้โดยไม่แก้ GCN layer
- **Serialization ด้วย serde** — serialize/deserialize Graph เป็น JSON

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: `Vec`, iterators, closures, `flat_map`, `zip`, `map`, `enumerate`
- **Part 31–40**: Traits, generics, error handling, `Option` / `Result`
- **Part 41–50**: `derive(Debug, Clone)`, default implementations, module system (`mod`)
- **Part 51–60**: Numeric types, `f32`/`f64` operations, `sqrt`, `exp`, `ln`, `clamp`
- **Part 61–70**: `serde`, `serde_json`, `rand` crate, seeded RNG
- **Project I02 (Neural Network)** — matrix operations, backpropagation, training loop
- **Project I09 (Q-Learning)** — ความเข้าใจ reinforcement learning และ state representation

## โครงสร้างโปรเจค (Project Layout)

```
graph-nn/
├── src/
│   ├── main.rs          ← demo: 6-node graph, GCN, GraphSAGE, link prediction
│   ├── graph.rs         ← Graph struct, adjacency list, CSR format, normalized adj
│   ├── features.rs      ← FeatureMatrix, L2/min-max normalize, matmul
│   ├── message_passing.rs ← MessagePassing: sum/mean/max aggregation
│   ├── gcn.rs           ← GcnLayer, softmax, cross_entropy_loss, accuracy
│   ├── sage.rs          ← GraphSage: mean/max aggregator + linear
│   └── link_pred.rs     ← LinkPredictor, sigmoid, BCE loss, negative sampling
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Graph Neural Network Data Flow

```
Input Graph G = (V, E)          Node Features H^(0) ∈ R^{n×d}
         │                                │
         ▼                                ▼
┌─────────────────────────────────────────────────────┐
│                 Preprocessing                       │
│                                                     │
│  1. เพิ่ม self-loops: Ã = A + I                     │
│  2. คำนวณ degree: D̃ᵢᵢ = Σⱼ Ãᵢⱼ                    │
│  3. normalize: D̃^{-1/2} Ã D̃^{-1/2}                │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│               GCN Layer 1                           │
│                                                     │
│  H^(1) = ReLU(D̃^{-1/2} Ã D̃^{-1/2} H^(0) W^(0))  │
│  shape: n × hidden_dim                              │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│               GCN Layer 2                           │
│                                                     │
│  H^(2) = Softmax(D̃^{-1/2} Ã D̃^{-1/2} H^(1) W^(1))│
│  shape: n × n_classes                               │
└─────────────────────────────────────────────────────┘
         │
         ├──────────────────┐
         ▼                  ▼
  Node Classification    Link Prediction
  CrossEntropy(H^(2), y) score(hᵤ, hᵥ) = sigmoid(hᵤ·hᵥ)
```

### Message Passing Framework

ทุก GNN architecture สามารถมองเป็น **message passing** ทั่วไปได้:

```
สำหรับแต่ละ iteration t:
  1. MESSAGE:   mᵥ^(t) = ⊕_{u∈N(v)} msg(hᵤ^(t-1), hᵥ^(t-1), eᵤᵥ)
  2. UPDATE:    hᵥ^(t) = update(hᵥ^(t-1), mᵥ^(t))

ที่ ⊕ คือ aggregation function (sum, mean, max, attention)
```

**GCN** ใช้ normalized sum: `⊕ = Σ with normalization`
**GraphSAGE** ใช้ concat + mean/max: `⊕ = mean หรือ max`
**GAT (Graph Attention Network)** ใช้ weighted sum ด้วย attention

### เหตุผลที่เลือก Flat Vec<f32>

แทนที่จะใช้ nested `Vec<Vec<f32>>` สำหรับ matrix เราเลือกใช้ flat `Vec<f32>` แบบ row-major เพราะ:
1. **Cache-friendly** — elements ติดกันใน memory ทำให้ CPU cache miss น้อยลง
2. **SIMD-ready** — สามารถ vectorize ด้วย SIMD instructions ได้ง่ายกว่า
3. **Zero allocation overhead** — ไม่ต้องสร้าง heap allocation ซ้อนกัน
4. **Compatible กับ BLAS** — ไปต่อกับ `blas` crate ได้โดยตรงถ้าต้องการประสิทธิภาพสูงขึ้น

Element `[i][j]` ของ matrix ขนาด `m×n` อยู่ที่ index `i * n + j`

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Graph Representation และ CSR Format

เริ่มต้นด้วยการสร้าง data structure พื้นฐานสำหรับ graph ใน Rust Graph ของเราจะรองรับสองรูปแบบหลัก:
- **Adjacency list** (`Vec<Vec<usize>>`) เหมาะสำหรับ traversal และ message passing
- **CSR (Compressed Sparse Row)** เหมาะสำหรับ graph ขนาดใหญ่ที่ต้องการ memory-efficient representation

**สร้าง `src/graph.rs`:**

```rust
use serde::{Deserialize, Serialize};
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Graph {
    pub num_nodes: usize,
    pub edges: Vec<(usize, usize, f32)>,  // (u, v, weight)
    pub adj: Vec<Vec<usize>>,              // adjacency list
}

impl Graph {
    pub fn new(num_nodes: usize) -> Self {
        Graph {
            num_nodes,
            edges: Vec::new(),
            adj: vec![Vec::new(); num_nodes],
        }
    }

    pub fn add_edge(&mut self, u: usize, v: usize, w: f32) {
        assert!(u < self.num_nodes && v < self.num_nodes, "node index out of range");
        self.edges.push((u, v, w));
        // undirected graph: เพิ่มทั้งสองทิศทาง
        if !self.adj[u].contains(&v) {
            self.adj[u].push(v);
        }
        if !self.adj[v].contains(&u) {
            self.adj[v].push(u);
        }
    }

    pub fn add_self_loops(&mut self) {
        for i in 0..self.num_nodes {
            if !self.adj[i].contains(&i) {
                self.adj[i].push(i);
            }
        }
    }

    pub fn num_nodes(&self) -> usize { self.num_nodes }
    pub fn num_edges(&self) -> usize { self.edges.len() }
    pub fn degree(&self, node: usize) -> usize { self.adj[node].len() }

    pub fn to_json(&self) -> String {
        serde_json::to_string(self).expect("serialization failed")
    }

    pub fn from_json(s: &str) -> Result<Self, serde_json::Error> {
        serde_json::from_str(s)
    }
}
```

**CSR Format** — สำหรับ graph ที่มีหลักล้าน nodes adjacency list จะใช้ memory มาก CSR เก็บข้อมูลเป็น 3 arrays:

```rust
pub struct CsrGraph {
    pub indptr: Vec<usize>,   // ขนาด n+1: บอกว่า neighbors ของ node i อยู่ที่ indices[indptr[i]..indptr[i+1]]
    pub indices: Vec<usize>,  // column indices ของ nonzero elements
    pub data: Vec<f32>,       // edge weights
}

impl CsrGraph {
    pub fn from_graph(g: &Graph) -> Self {
        let n = g.num_nodes;
        let mut rows: Vec<Vec<(usize, f32)>> = vec![Vec::new(); n];
        for &(u, v, w) in &g.edges {
            rows[u].push((v, w));
            rows[v].push((u, w));  // undirected
        }
        let mut indptr = vec![0usize];
        let mut indices = Vec::new();
        let mut data = Vec::new();
        for row in &rows {
            for &(col, w) in row {
                indices.push(col);
                data.push(w);
            }
            indptr.push(indices.len());
        }
        CsrGraph { indptr, indices, data }
    }

    pub fn neighbors(&self, node: usize) -> &[usize] {
        &self.indices[self.indptr[node]..self.indptr[node + 1]]
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
let mut g = Graph::new(5);
g.add_edge(0, 1, 1.0);
g.add_edge(1, 2, 1.0);
g.add_edge(2, 3, 1.0);
g.add_edge(3, 4, 1.0);
g.add_edge(4, 0, 1.0);  // ring graph

println!("nodes: {}", g.num_nodes());  // 5
println!("edges: {}", g.num_edges());  // 5
println!("degree(0): {}", g.degree(0));  // 2

let csr = CsrGraph::from_graph(&g);
println!("neighbors(2): {:?}", csr.neighbors(2));  // [1, 3]
```

**แนวคิด CSR:** สำหรับ graph ที่มี 1 ล้าน nodes และ 10 ล้าน edges adjacency list ใช้ memory `O(V + E)` = ~44 MB ส่วน CSR ใช้เพียง 3 arrays รวมกันประมาณ 48 MB แต่ access pattern ดีกว่าเพราะ neighbors ของแต่ละ node อยู่ติดกันใน `indices` array

---

### ขั้นที่ 2: Feature Matrix และ Normalization

Node feature matrix คือ matrix H ขนาด `n × d` ที่แต่ละแถวเป็น feature vector ของ node แต่ละตัว

**สร้าง `src/features.rs`:**

```rust
#[derive(Debug, Clone)]
pub struct FeatureMatrix {
    pub data: Vec<f32>,  // flat row-major: data[i*cols + j] = H[i][j]
    pub rows: usize,
    pub cols: usize,
}

impl FeatureMatrix {
    pub fn new(rows_data: Vec<Vec<f32>>) -> Self {
        let rows = rows_data.len();
        let cols = if rows > 0 { rows_data[0].len() } else { 0 };
        let mut data = Vec::with_capacity(rows * cols);
        for row in &rows_data {
            assert_eq!(row.len(), cols, "all rows must have same length");
            data.extend_from_slice(row);
        }
        FeatureMatrix { data, rows, cols }
    }

    pub fn rows(&self) -> usize { self.rows }
    pub fn cols(&self) -> usize { self.cols }

    pub fn row(&self, i: usize) -> &[f32] {
        let start = i * self.cols;
        &self.data[start..start + self.cols]
    }

    /// L2 normalize แต่ละ row ให้ norm = 1
    pub fn l2_normalize(&self) -> Vec<f32> {
        let mut out = self.data.clone();
        for i in 0..self.rows {
            let start = i * self.cols;
            let end = start + self.cols;
            let norm: f32 = self.data[start..end]
                .iter().map(|&x| x * x).sum::<f32>().sqrt();
            if norm > 1e-8 {
                for v in &mut out[start..end] {
                    *v /= norm;
                }
            }
        }
        out
    }

    /// Min-max normalize แต่ละ column ให้อยู่ใน [0, 1]
    pub fn min_max_normalize(&self) -> Vec<f32> {
        let mut out = self.data.clone();
        for j in 0..self.cols {
            let vals: Vec<f32> = (0..self.rows).map(|i| self.data[i*self.cols+j]).collect();
            let min_v = vals.iter().cloned().fold(f32::MAX, f32::min);
            let max_v = vals.iter().cloned().fold(f32::MIN, f32::max);
            let range = max_v - min_v;
            if range > 1e-8 {
                for i in 0..self.rows {
                    out[i*self.cols+j] = (self.data[i*self.cols+j] - min_v) / range;
                }
            }
        }
        out
    }
}
```

**Matrix Multiplication** เป็น core operation ของ GCN ทุก layer:

```rust
/// A(m×k) @ B(k×n) -> C(m×n)  row-major
pub fn matmul(a: &[f32], b: &[f32], m: usize, k: usize, n: usize) -> Vec<f32> {
    let mut c = vec![0.0f32; m * n];
    for i in 0..m {
        for j in 0..n {
            let mut sum = 0.0f32;
            for l in 0..k {
                sum += a[i * k + l] * b[l * n + j];
            }
            c[i * n + j] = sum;
        }
    }
    c
}
```

**ตัวอย่าง normalization:**

```rust
let data = vec![
    vec![3.0f32, 4.0],   // norm = 5.0 -> [0.6, 0.8]
    vec![0.0, 2.0],      // norm = 2.0 -> [0.0, 1.0]
];
let feat = FeatureMatrix::new(data);
let normed = feat.l2_normalize();
println!("{:?}", &normed[..2]);  // [0.6, 0.8]
```

**เหตุใดต้อง normalize features:** features ที่มี scale ต่างกันมาก (เช่น age [0-100] vs income [0-100000]) จะทำให้ gradient ไม่สมดุล และ training ช้า L2 normalization เหมาะสำหรับ features ที่เป็น embedding vector ส่วน min-max เหมาะสำหรับ continuous numerical features

---

### ขั้นที่ 3: Message Passing Framework

Message Passing เป็น abstraction ที่รวม GNN หลายสถาปัตยกรรมไว้ใน framework เดียวกัน

**สร้าง `src/message_passing.rs`:**

```rust
pub struct MessagePassing;

impl MessagePassing {
    pub fn new() -> Self { MessagePassing }

    /// Sum aggregation
    pub fn aggregate_sum(
        &self,
        features: &[f32],
        adj: &[Vec<usize>],
        n_nodes: usize,
        feat_dim: usize,
    ) -> Vec<f32> {
        let mut out = vec![0.0f32; n_nodes * feat_dim];
        for node in 0..n_nodes {
            for &nb in &adj[node] {
                for d in 0..feat_dim {
                    out[node * feat_dim + d] += features[nb * feat_dim + d];
                }
            }
        }
        out
    }

    /// Mean aggregation
    pub fn aggregate_mean(
        &self,
        features: &[f32],
        adj: &[Vec<usize>],
        n_nodes: usize,
        feat_dim: usize,
    ) -> Vec<f32> {
        let mut out = vec![0.0f32; n_nodes * feat_dim];
        for node in 0..n_nodes {
            let n_nb = adj[node].len();
            if n_nb == 0 { continue; }
            for &nb in &adj[node] {
                for d in 0..feat_dim {
                    out[node * feat_dim + d] += features[nb * feat_dim + d];
                }
            }
            for d in 0..feat_dim {
                out[node * feat_dim + d] /= n_nb as f32;
            }
        }
        out
    }

    /// Max aggregation — เลือกค่า max ในแต่ละ dimension
    pub fn aggregate_max(
        &self,
        features: &[f32],
        adj: &[Vec<usize>],
        n_nodes: usize,
        feat_dim: usize,
    ) -> Vec<f32> {
        let mut out = vec![f32::NEG_INFINITY; n_nodes * feat_dim];
        for node in 0..n_nodes {
            if adj[node].is_empty() {
                for d in 0..feat_dim {
                    out[node * feat_dim + d] = 0.0;
                }
                continue;
            }
            for &nb in &adj[node] {
                for d in 0..feat_dim {
                    let v = features[nb * feat_dim + d];
                    if v > out[node * feat_dim + d] {
                        out[node * feat_dim + d] = v;
                    }
                }
            }
        }
        out
    }

    /// Update: self + message -> ReLU
    pub fn update(
        &self,
        self_feat: &[f32],
        msg: &[f32],
        n_nodes: usize,
        feat_dim: usize,
    ) -> Vec<f32> {
        (0..(n_nodes * feat_dim))
            .map(|i| relu(self_feat[i] + msg[i]))
            .collect()
    }
}

fn relu(x: f32) -> f32 { if x > 0.0 { x } else { 0.0 } }
```

**ตัวอย่างการใช้ message passing:**

```rust
// Graph: 0 -- 1 -- 2
let adj = vec![vec![1], vec![0, 2], vec![1]];
// Features: node 0=[1,0], node 1=[0,1], node 2=[1,1]
let feat = vec![1.0f32, 0.0,  0.0, 1.0,  1.0, 1.0];

let mp = MessagePassing::new();
let agg = mp.aggregate_mean(&feat, &adj, 3, 2);
// node 0: mean neighbor(1) = [0, 1]
// node 1: mean(neighbor(0), neighbor(2)) = mean([1,0],[1,1]) = [1.0, 0.5]
// node 2: mean neighbor(1) = [0, 1]
println!("{:?}", &agg);
```

**Aggregation functions เปรียบเทียบ:**

| Aggregation | ข้อดี | ข้อเสีย | ใช้ใน |
|-------------|-------|---------|-------|
| Sum | รักษาข้อมูลขนาด neighborhood | sensitive ต่อ node degree | GraphSAGE variant |
| Mean | normalize by degree | อาจ lose structural info | GCN, GraphSAGE |
| Max | จับ dominant features | ทิ้งข้อมูลหลายค่า | GraphSAGE max |
| Attention | weighted by importance | คำนวณแพงกว่า | GAT |

---

### ขั้นที่ 4: Normalized Adjacency Matrix (D^{-1/2} A D^{-1/2})

นี่คือหัวใจของ GCN Kipf & Welling (2017) แสดงให้เห็นว่าการ normalize adjacency matrix ด้วย D^{-1/2} ทั้งสองข้างทำให้ numerical stability ดีขึ้นและ training converge เร็วขึ้น

**Algorithm:**

```
ขั้นที่ 1: สร้าง Ã = A + I  (เพิ่ม self-loops)
ขั้นที่ 2: คำนวณ D̃ᵢᵢ = Σⱼ Ãᵢⱼ  (degree matrix รวม self-loops)
ขั้นที่ 3: คำนวณ D̃^{-1/2}: element D̃^{-1/2}ᵢᵢ = 1/√D̃ᵢᵢ
ขั้นที่ 4: ĝ = D̃^{-1/2} Ã D̃^{-1/2}
```

**Implementation ใน `src/graph.rs`:**

```rust
pub fn normalized_adj(&self) -> Vec<f32> {
    let n = self.num_nodes;
    let mut a = vec![0.0f32; n * n];

    // เพิ่ม self-loops (Ã = A + I)
    for i in 0..n {
        a[i * n + i] = 1.0;
    }
    for &(u, v, _w) in &self.edges {
        a[u * n + v] = 1.0;
        a[v * n + u] = 1.0;
    }

    // คำนวณ degree (รวม self-loops)
    let mut deg = vec![0.0f32; n];
    for i in 0..n {
        for j in 0..n {
            deg[i] += a[i * n + j];
        }
    }

    // D^{-1/2}: inverse square root ของ degree
    let d_inv_sqrt: Vec<f32> = deg.iter()
        .map(|&d| if d > 0.0 { 1.0 / d.sqrt() } else { 0.0 })
        .collect();

    // ĝᵢⱼ = D̃^{-1/2}ᵢᵢ × Ãᵢⱼ × D̃^{-1/2}ⱼⱼ
    let mut norm = vec![0.0f32; n * n];
    for i in 0..n {
        for j in 0..n {
            norm[i * n + j] = d_inv_sqrt[i] * a[i * n + j] * d_inv_sqrt[j];
        }
    }
    norm
}
```

**ตัวอย่าง:** สำหรับ ring graph (0-1-2-3-0) ที่มี self-loops:

```
Ã (adjacency + self-loops):
  0 1 2 3
0[1 1 0 1]
1[1 1 1 0]
2[0 1 1 1]
3[1 0 1 1]

Degree: D̃ = diag(3, 3, 3, 3)
D̃^{-1/2} = diag(1/√3, 1/√3, 1/√3, 1/√3)

ĝ = D̃^{-1/2} Ã D̃^{-1/2}:
  ĝᵢⱼ = (1/√3) × Ãᵢⱼ × (1/√3) = Ãᵢⱼ/3

นั่นคือ ĝᵢⱼ = 1/3 ถ้า Ãᵢⱼ = 1
```

**ทำไมต้อง normalize?** ถ้าใช้ A โดยตรงโดยไม่ normalize: node ที่มี degree สูงจะมี activation ที่ scale ใหญ่กว่า node ที่มี degree ต่ำ ทำให้ gradient ไม่สมดุล การ normalize ทำให้ message จากทุก neighbor มี contribution ที่สม่ำเสมอ

**คุณสมบัติของ ĝ:**
- Symmetric: ĝᵢⱼ = ĝⱼᵢ (เพราะ A และ D ล้วน symmetric)
- ค่า eigenvalues อยู่ใน [-1, 1] ทำให้ gradient ไม่ explode/vanish
- สามารถคำนวณล่วงหน้าครั้งเดียวก่อน training ได้เลย (precomputed)

---

### ขั้นที่ 5: GCN Layer

**สร้าง `src/gcn.rs`:**

```rust
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;
use crate::features::matmul;

pub struct GcnLayer {
    pub in_dim: usize,
    pub out_dim: usize,
    pub weights: Vec<f32>,   // in_dim × out_dim, row-major
    pub bias: Vec<f32>,      // out_dim
}

impl GcnLayer {
    pub fn new(in_dim: usize, out_dim: usize, seed: u64) -> Self {
        let mut rng = StdRng::seed_from_u64(seed);
        // Xavier initialization
        let limit = (6.0f32 / (in_dim + out_dim) as f32).sqrt();
        let weights: Vec<f32> = (0..in_dim * out_dim)
            .map(|_| rng.gen_range(-limit..limit))
            .collect();
        let bias = vec![0.0f32; out_dim];
        GcnLayer { in_dim, out_dim, weights, bias }
    }

    /// Forward pass: H' = ReLU(ĝ H W + b)
    /// ĝ: n×n normalized adjacency
    /// H: n×in_dim node features
    /// W: in_dim×out_dim weight matrix
    pub fn forward(&mut self, h: &[f32], norm_adj: &[f32], n_nodes: usize) -> Vec<f32> {
        // ขั้นที่ 1: AH = ĝ @ H  -> shape (n × in_dim)
        let ah = matmul(norm_adj, h, n_nodes, n_nodes, self.in_dim);

        // ขั้นที่ 2: AHW = AH @ W  -> shape (n × out_dim)
        let mut out = matmul(&ah, &self.weights, n_nodes, self.in_dim, self.out_dim);

        // ขั้นที่ 3: add bias + ReLU
        for i in 0..n_nodes {
            for j in 0..self.out_dim {
                out[i * self.out_dim + j] += self.bias[j];
                out[i * self.out_dim + j] = relu(out[i * self.out_dim + j]);
            }
        }
        out
    }

    /// SGD weight update: W ← W - lr × ∇W
    pub fn sgd_update(&mut self, grad_w: &[f32], grad_b: &[f32], lr: f32) {
        for (w, &gw) in self.weights.iter_mut().zip(grad_w.iter()) {
            *w -= lr * gw;
        }
        for (b, &gb) in self.bias.iter_mut().zip(grad_b.iter()) {
            *b -= lr * gb;
        }
    }
}

fn relu(x: f32) -> f32 { if x > 0.0 { x } else { 0.0 } }
```

**Xavier Initialization** — เหตุใดต้องใช้:

```
ถ้า initialize weights ด้วยค่า random ที่ scale ใหญ่เกินไป:
  → activations จะ explode → gradient explode → NaN

ถ้า initialize ด้วยค่าที่เล็กเกินไป (เช่น zero):
  → activations จะ vanish → gradient ≈ 0 → network ไม่เรียนรู้

Xavier formula: limit = sqrt(6 / (fan_in + fan_out))
  W ~ Uniform(-limit, limit)

สำหรับ in_dim=8, out_dim=4: limit = sqrt(6/12) = sqrt(0.5) ≈ 0.707
```

**ตัวอย่างการใช้ GCN Layer:**

```rust
let mut g = Graph::new(5);
g.add_edge(0, 1, 1.0);
g.add_edge(1, 2, 1.0);
// ...

let norm_adj = g.normalized_adj();  // precompute
let feat = vec![1.0f32; 5 * 4];    // 5 nodes, 4 features each

let mut layer1 = GcnLayer::new(4, 8, 42);  // seed=42 สำหรับ reproducibility
let h1 = layer1.forward(&feat, &norm_adj, 5);

println!("H1 shape: {}×{}", 5, 8);
println!("H1[0]: {:?}", &h1[..8]);  // embedding ของ node 0
```

---

### ขั้นที่ 6: Two-Layer GCN สำหรับ Node Classification

Node classification คือการ predict label ของแต่ละ node (เช่น community ในโครงข่ายสังคม, category ของบทความ) โดยใช้ 2-layer GCN:

```
Layer 1: H^(1) = ReLU(ĝ H^(0) W^(0))   # extract features
Layer 2: H^(2) = Softmax(ĝ H^(1) W^(1)) # predict class probabilities
```

**Loss function: Cross-Entropy**

```rust
pub fn softmax(x: &[f32]) -> Vec<f32> {
    // numerically stable: subtract max ก่อน
    let max_v = x.iter().cloned().fold(f32::NEG_INFINITY, f32::max);
    let exps: Vec<f32> = x.iter().map(|&v| (v - max_v).exp()).collect();
    let sum: f32 = exps.iter().sum();
    exps.iter().map(|&e| e / sum).collect()
}

pub fn cross_entropy_loss(logits: &[Vec<f32>], labels: &[usize]) -> f32 {
    let n = logits.len();
    let total: f32 = logits.iter().zip(labels.iter())
        .map(|(logit, &label)| {
            let probs = softmax(logit);
            let p = probs[label].max(1e-7);  // clip เพื่อหลีกเลี่ยง log(0)
            -p.ln()
        })
        .sum();
    total / n as f32
}
```

**Training loop สำหรับ node classification:**

```rust
fn train_gcn(
    graph: &Graph,
    features: &[f32],      // n × in_dim
    labels: &[usize],      // n (class index)
    n_epochs: usize,
    lr: f32,
) {
    let norm_adj = graph.normalized_adj();
    let n_nodes = graph.num_nodes();
    let in_dim = 4;
    let hidden_dim = 16;
    let n_classes = 3;

    let mut gcn1 = GcnLayer::new(in_dim, hidden_dim, 42);
    let mut gcn2 = GcnLayer::new(hidden_dim, n_classes, 43);

    for epoch in 0..n_epochs {
        // Forward pass
        let h1 = gcn1.forward(features, &norm_adj, n_nodes);
        let h2 = gcn2.forward(&h1, &norm_adj, n_nodes);

        // Compute loss
        let logits = reshape_output(&h2, n_nodes, n_classes);
        let loss = cross_entropy_loss(&logits, labels);
        let acc = accuracy(&logits, labels);

        if epoch % 10 == 0 {
            println!("Epoch {}: loss={:.4}, acc={:.2}%", epoch, loss, acc * 100.0);
        }
        // Note: Full backprop ต้องการ gradient computation
        // ซึ่งซับซ้อนกว่า — สำหรับโปรเจคนี้เน้น forward pass architecture
    }
}
```

**Semi-supervised learning:** ใน paper ต้นฉบับของ GCN (Kipf 2017) training ใช้เฉพาะ nodes ที่มี label (ซึ่งอาจเป็นแค่ไม่กี่ % ของ graph ทั้งหมด) แต่ forward pass วิ่งผ่าน nodes ทั้งหมด ทำให้ unlabeled nodes ได้รับ gradient ผ่าน message passing โดยอ้อม

---

### ขั้นที่ 7: GraphSAGE Aggregation

GraphSAGE (Hamilton et al. 2017) แก้ปัญหาสำคัญของ GCN: **transductive learning** — GCN ต้อง retrain ใหม่ทั้งหมดเมื่อเพิ่ม node ใหม่ ส่วน GraphSAGE เป็น **inductive learning** สามารถ generalize ไปยัง unseen nodes ได้

**หลักการ:**
1. Sample neighborhood: สุ่มเลือก k neighbors (ไม่ใช้ทุก neighbor)
2. Aggregate: mean หรือ max ของ neighbor features
3. Concat: รวม self feature กับ aggregated features
4. Linear + ReLU: ผ่าน learnable weight matrix

```rust
pub struct GraphSage {
    pub in_dim: usize,
    pub out_dim: usize,
    pub weights: Vec<f32>,  // (2*in_dim) × out_dim
    pub bias: Vec<f32>,
}

impl GraphSage {
    pub fn new(in_dim: usize, out_dim: usize, seed: u64) -> Self {
        let mut rng = StdRng::seed_from_u64(seed);
        let concat_dim = 2 * in_dim;
        let limit = (6.0f32 / (concat_dim + out_dim) as f32).sqrt();
        let weights: Vec<f32> = (0..concat_dim * out_dim)
            .map(|_| rng.gen_range(-limit..limit))
            .collect();
        GraphSage { in_dim, out_dim, weights, bias: vec![0.0f32; out_dim] }
    }

    pub fn aggregate_mean(
        &self,
        features: &[f32],
        adj: &[Vec<usize>],
        n_nodes: usize,
        feat_dim: usize,
    ) -> Vec<f32> {
        // ขั้นที่ 1: mean aggregate
        let mut agg = vec![0.0f32; n_nodes * feat_dim];
        for node in 0..n_nodes {
            let neighbors = &adj[node];
            if neighbors.is_empty() {
                // ถ้าไม่มี neighbor ใช้ self feature
                for d in 0..feat_dim {
                    agg[node * feat_dim + d] = features[node * feat_dim + d];
                }
                continue;
            }
            for &nb in neighbors {
                for d in 0..feat_dim {
                    agg[node * feat_dim + d] += features[nb * feat_dim + d];
                }
            }
            let count = neighbors.len() as f32;
            for d in 0..feat_dim {
                agg[node * feat_dim + d] /= count;
            }
        }

        // ขั้นที่ 2: concat [self || aggregated]
        let concat_dim = 2 * feat_dim;
        let mut concat = vec![0.0f32; n_nodes * concat_dim];
        for node in 0..n_nodes {
            for d in 0..feat_dim {
                concat[node * concat_dim + d] = features[node * feat_dim + d];
                concat[node * concat_dim + feat_dim + d] = agg[node * feat_dim + d];
            }
        }

        // ขั้นที่ 3: linear transform + ReLU
        let mut out = matmul(&concat, &self.weights, n_nodes, concat_dim, self.out_dim);
        for i in 0..n_nodes {
            for j in 0..self.out_dim {
                out[i * self.out_dim + j] += self.bias[j];
                let v = out[i * self.out_dim + j];
                out[i * self.out_dim + j] = if v > 0.0 { v } else { 0.0 };
            }
        }
        out
    }
}
```

**GraphSAGE vs GCN:**

| คุณสมบัติ | GCN | GraphSAGE |
|-----------|-----|-----------|
| Learning type | Transductive | Inductive |
| Aggregation | Normalized sum (ĝ) | Mean / Max (concat) |
| New nodes | ต้อง retrain | Generalize ได้ทันที |
| Memory | O(n²) สำหรับ dense adj | O(k) per node |
| Speed | เร็วกว่า (matrix multiply) | ช้ากว่าเล็กน้อย (concat) |

**Neighborhood Sampling** ใน GraphSAGE สำคัญมากสำหรับ scalability:

```rust
// sample ≤ k neighbors ด้วย seed สำหรับ reproducibility
pub fn sample_neighbors(&self, node: usize, k: usize, seed: u64) -> Vec<usize> {
    let neighbors = &self.adj[node];
    if neighbors.len() <= k {
        return neighbors.clone();
    }
    let mut rng = StdRng::seed_from_u64(seed);
    let mut indices: Vec<usize> = (0..neighbors.len()).collect();
    // Fisher-Yates partial shuffle
    for i in 0..k {
        let j = i + rng.gen_range(0..(neighbors.len() - i));
        indices.swap(i, j);
    }
    indices[..k].iter().map(|&idx| neighbors[idx]).collect()
}
```

**ทำไม Fisher-Yates?** เพราะ Fisher-Yates ให้ uniformly random sample โดยไม่ต้องสร้าง copy ของ array ทั้งหมด ทำให้เป็น O(k) แทนที่จะเป็น O(n log n) สำหรับ node ที่มี neighbors เป็นพัน

---

### ขั้นที่ 8: Link Prediction

Link prediction คือการทำนายว่า edge ระหว่าง node คู่ใดควรมีอยู่ใน graph ใช้ในระบบ friend recommendation, drug-target interaction prediction, knowledge graph completion

**สร้าง `src/link_pred.rs`:**

```rust
pub struct LinkPredictor {
    pub embed_dim: usize,
}

impl LinkPredictor {
    pub fn new(embed_dim: usize) -> Self {
        LinkPredictor { embed_dim }
    }

    /// Dot product: score(u,v) = hᵤ · hᵥ
    /// สูง = มีแนวโน้มมี edge
    pub fn dot_product(&self, h_u: &[f32], h_v: &[f32]) -> f32 {
        h_u.iter().zip(h_v.iter()).map(|(&a, &b)| a * b).sum()
    }

    /// Hadamard product: hᵤ ⊙ hᵥ (element-wise multiply)
    /// ใช้เป็น edge feature สำหรับ classifier เพิ่มเติม
    pub fn hadamard(&self, h_u: &[f32], h_v: &[f32]) -> Vec<f32> {
        h_u.iter().zip(h_v.iter()).map(|(&a, &b)| a * b).collect()
    }

    /// Cosine similarity
    pub fn cosine_similarity(&self, h_u: &[f32], h_v: &[f32]) -> f32 {
        let dot: f32 = h_u.iter().zip(h_v.iter()).map(|(&a, &b)| a * b).sum();
        let norm_u: f32 = h_u.iter().map(|&a| a * a).sum::<f32>().sqrt();
        let norm_v: f32 = h_v.iter().map(|&b| b * b).sum::<f32>().sqrt();
        if norm_u < 1e-8 || norm_v < 1e-8 { return 0.0; }
        dot / (norm_u * norm_v)
    }
}

/// Sigmoid: ✓ numerically stable สำหรับทุก range
pub fn sigmoid(x: f32) -> f32 {
    if x >= 0.0 {
        1.0 / (1.0 + (-x).exp())
    } else {
        let e = x.exp();
        e / (1.0 + e)
    }
}

/// Binary Cross-Entropy: BCE(p, y) = -[y log(p) + (1-y) log(1-p)]
pub fn binary_cross_entropy(preds: &[f32], labels: &[f32]) -> f32 {
    let eps = 1e-7f32;
    preds.iter().zip(labels.iter())
        .map(|(&p, &y)| {
            let p = p.clamp(eps, 1.0 - eps);  // ป้องกัน log(0)
            -(y * p.ln() + (1.0 - y) * (1.0 - p).ln())
        })
        .sum::<f32>() / preds.len() as f32
}
```

**Negative Sampling** — สำหรับ link prediction เราต้องมีทั้ง positive edges (ที่มีอยู่จริง) และ negative edges (ที่ไม่มีอยู่):

```rust
pub fn negative_sample(
    n_nodes: usize,
    existing_edges: &[(usize, usize)],
    n_samples: usize,
    seed: u64,
) -> Vec<(usize, usize)> {
    let mut rng = StdRng::seed_from_u64(seed);
    let mut negatives = Vec::with_capacity(n_samples);

    while negatives.len() < n_samples {
        let u = rng.gen_range(0..n_nodes);
        let v = rng.gen_range(0..n_nodes);
        if u == v { continue; }
        // ต้องไม่ใช่ edge ที่มีอยู่แล้ว
        let edge = if u < v { (u, v) } else { (v, u) };
        if existing_edges.contains(&edge) { continue; }
        if negatives.contains(&edge) { continue; }
        negatives.push(edge);
    }
    negatives
}
```

**Link Prediction Training Pipeline:**

```rust
fn train_link_pred(
    graph: &Graph,
    embeddings: &[f32],    // node embeddings จาก GCN/GraphSAGE
    embed_dim: usize,
    n_epochs: usize,
    lr: f32,
) {
    let predictor = LinkPredictor::new(embed_dim);
    let existing_edges: Vec<(usize, usize)> = graph.edges
        .iter()
        .map(|&(u, v, _)| (u.min(v), u.max(v)))
        .collect();

    for epoch in 0..n_epochs {
        // Positive samples
        let pos_scores: Vec<f32> = existing_edges.iter()
            .map(|&(u, v)| {
                let hu = &embeddings[u * embed_dim..(u+1) * embed_dim];
                let hv = &embeddings[v * embed_dim..(v+1) * embed_dim];
                sigmoid(predictor.dot_product(hu, hv))
            })
            .collect();

        // Negative samples
        let neg_edges = negative_sample(
            graph.num_nodes(), &existing_edges, existing_edges.len(), epoch as u64
        );
        let neg_scores: Vec<f32> = neg_edges.iter()
            .map(|&(u, v)| {
                let hu = &embeddings[u * embed_dim..(u+1) * embed_dim];
                let hv = &embeddings[v * embed_dim..(v+1) * embed_dim];
                sigmoid(predictor.dot_product(hu, hv))
            })
            .collect();

        let all_scores: Vec<f32> = pos_scores.iter().chain(neg_scores.iter()).cloned().collect();
        let pos_labels = vec![1.0f32; pos_scores.len()];
        let neg_labels = vec![0.0f32; neg_scores.len()];
        let all_labels: Vec<f32> = pos_labels.iter().chain(neg_labels.iter()).cloned().collect();

        let loss = binary_cross_entropy(&all_scores, &all_labels);
        if epoch % 10 == 0 {
            println!("Epoch {}: BCE loss = {:.4}", epoch, loss);
        }
    }
}
```

**ทำไม negative sampling ถึงสำคัญ:** ใน graph จริง ๆ จำนวน edge มักน้อยมากเมื่อเทียบกับ node ทั้งหมด (sparse graph) ถ้า train โดยใช้แค่ positive examples model จะ predict ทุก pair เป็น 1 ซึ่งไม่มีประโยชน์ negative sampling สร้าง class balance ทำให้ model เรียนรู้ที่จะแยกแยะ positive จาก negative ได้

---

### ขั้นที่ 9: Mini-Batch Training ด้วย Neighborhood Sampling

สำหรับ graph ขนาดใหญ่ การทำ full-batch gradient descent (คำนวณ loss จากทุก node พร้อมกัน) ไม่ practical เพราะ:
- `normalized_adj` matrix ขนาด n×n ใช้ memory O(n²)
- การทำ matmul บน matrix ขนาดใหญ่ช้ามาก
- GPU memory จำกัด

**Mini-batch strategy ของ GraphSAGE:**

```
1. สุ่มเลือก batch of target nodes
2. สำหรับแต่ละ target node: sample k1 neighbors (layer 1)
3. สำหรับแต่ละ sampled neighbor: sample k2 neighbors (layer 2)
4. คำนวณ embeddings เฉพาะ sub-graph นี้เท่านั้น
```

**Implementation ของ neighborhood sampling:**

```rust
pub fn sample_neighbors(&self, node: usize, k: usize, seed: u64) -> Vec<usize> {
    let neighbors = &self.adj[node];
    if neighbors.len() <= k {
        return neighbors.clone();
    }
    let mut rng = StdRng::seed_from_u64(seed);
    let mut indices: Vec<usize> = (0..neighbors.len()).collect();
    // Fisher-Yates shuffle แบบ partial (O(k) แทน O(n))
    for i in 0..k {
        let j = i + rng.gen_range(0..(neighbors.len() - i));
        indices.swap(i, j);
    }
    indices[..k].iter().map(|&idx| neighbors[idx]).collect()
}
```

**Mini-batch training loop ตัวอย่าง:**

```rust
fn mini_batch_train(
    graph: &Graph,
    features: &[f32],
    labels: &[usize],
    batch_size: usize,
    k_neighbors: usize,
    n_epochs: usize,
) {
    let n_nodes = graph.num_nodes();
    let feat_dim = features.len() / n_nodes;

    for epoch in 0..n_epochs {
        // สุ่ม batch ของ target nodes
        let mut batch_nodes: Vec<usize> = (0..n_nodes).collect();
        // shuffle (ใช้ seed จาก epoch เพื่อ reproducibility)
        let mut rng = StdRng::seed_from_u64(epoch as u64);
        for i in 0..batch_nodes.len() {
            let j = rng.gen_range(i..batch_nodes.len());
            batch_nodes.swap(i, j);
        }

        for batch in batch_nodes.chunks(batch_size) {
            // สร้าง sub-graph สำหรับ batch นี้
            let mut all_nodes: Vec<usize> = batch.to_vec();
            for &node in batch {
                let sampled = graph.sample_neighbors(node, k_neighbors, epoch as u64);
                all_nodes.extend(sampled);
            }
            all_nodes.sort_unstable();
            all_nodes.dedup();

            // extract features ของ nodes ใน sub-graph
            let sub_features: Vec<f32> = all_nodes.iter()
                .flat_map(|&n| features[n*feat_dim..(n+1)*feat_dim].iter().cloned())
                .collect();

            // train บน sub_features + sub_labels
            // (ละ forward/backward pass สั้น ๆ เพื่อความกระชับ)
            let _ = sub_features;
        }
    }
}
```

**Complexity comparison:**

| Method | Memory (adjacency) | Time per epoch |
|--------|-------------------|----------------|
| Full-batch GCN | O(n²) | O(n² × d) |
| GraphSAGE full | O(n × avg_degree) | O(n × k × d) |
| GraphSAGE mini-batch | O(batch × k^L) | O(batch × k^L × d) |

สำหรับ graph 1 ล้าน nodes, d=128, L=2, k=25: mini-batch ใช้ memory เพียง ~25MB ต่อ batch ส่วน full-batch ต้องการ ~512GB

---

## การทดสอบ (Testing)

โปรเจคนี้มี 28 unit tests ครอบคลุมทุก component อย่างละเอียด

**สร้างใน scratchpad แล้วรัน `cargo test`:**

```
running 28 tests
test tests::test_binary_cross_entropy_loss ... ok
test tests::test_cross_entropy_loss ... ok
test tests::test_csr_format ... ok
test tests::test_feature_matrix_normalization ... ok
test tests::test_feature_matrix_shape ... ok
test tests::test_feature_row_access ... ok
test tests::test_gcn_relu_activation ... ok
test tests::test_gcn_two_layer_forward ... ok
test tests::test_gcn_weight_dimensions ... ok
test tests::test_gcn_output_shape ... ok
test tests::test_graph_add_self_loops ... ok
test tests::test_graph_adjacency_undirected ... ok
test tests::test_graph_degree ... ok
test tests::test_graph_creation ... ok
test tests::test_graphsage_no_negatives_after_relu ... ok
test tests::test_graphsage_output_shape ... ok
test tests::test_link_prediction_dot_product ... ok
test tests::test_graph_serialization ... ok
test tests::test_link_prediction_hadamard ... ok
test tests::test_message_passing_aggregate ... ok
test tests::test_message_passing_mean ... ok
test tests::test_mini_batch_neighborhood_sampling ... ok
test tests::test_negative_sampling ... ok
test tests::test_normalized_adj_self_loop ... ok
test tests::test_normalized_adj_symmetry ... ok
test tests::test_normalized_adj_shape ... ok
test tests::test_softmax_sums_to_one ... ok
test tests::test_sigmoid_range ... ok

test result: ok. 28 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Tests ที่สำคัญและอธิบาย:**

`test_normalized_adj_symmetry` — ตรวจว่า ĝᵢⱼ = ĝⱼᵢ สำหรับทุกคู่ (i, j) เพราะ A symmetric และ D symmetric:
```rust
for i in 0..n {
    for j in 0..n {
        let diff = (norm[i * n + j] - norm[j * n + i]).abs();
        assert!(diff < 1e-5, "norm_adj not symmetric at ({},{})", i, j);
    }
}
```

`test_gcn_relu_activation` — ตรวจว่า output ของ GCN layer ไม่มีค่าลบ เพราะ ReLU:
```rust
for &v in &out {
    assert!(v >= 0.0, "ReLU output should be >= 0, got {}", v);
}
```

`test_cross_entropy_loss` — ตรวจว่า loss ต่ำเมื่อ prediction ถูกต้อง:
```rust
let logits = vec![
    vec![10.0f32, 0.0, 0.0],  // node 0 -> class 0
    vec![0.0, 10.0, 0.0],     // node 1 -> class 1
];
let labels = vec![0usize, 1];
let loss = cross_entropy_loss(&logits, &labels);
assert!(loss < 0.01);
```

`test_negative_sampling` — ตรวจว่า negative samples ไม่ซ้ำกับ existing edges และ index ไม่เกิน n_nodes:
```rust
for (u, v) in &negs {
    assert!(*u < n_nodes && *v < n_nodes);
    assert!(!existing.contains(&(*u, *v)));
}
```

`test_sigmoid_range` — ตรวจ boundary cases:
```rust
assert!((sigmoid(0.0) - 0.5).abs() < 1e-5);
let hi = sigmoid(100.0);
assert!(hi > 0.99);  // ไม่ overflow เป็น 1.0 หรือ NaN
```

---

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### Pitfall 1: ลืมเพิ่ม Self-Loops ก่อน Normalize

**อาการ:** node ที่ไม่มี neighbor (isolated node) มี degree = 0 → division by zero → NaN

**โค้ดผิด:**
```rust
// ผิด: ไม่เพิ่ม self-loops ก่อน normalize
fn bad_normalized_adj(adj_matrix: &[f32], n: usize) -> Vec<f32> {
    let mut deg = vec![0.0f32; n];
    for i in 0..n {
        for j in 0..n {
            deg[i] += adj_matrix[i * n + j];  // isolated node -> deg[i] = 0
        }
    }
    // D^{-1/2}: ถ้า deg[i] = 0 -> division by zero!
    let d_inv_sqrt: Vec<f32> = deg.iter()
        .map(|&d| 1.0 / d.sqrt())  // NaN สำหรับ isolated node!
        .collect();
    // ...
}
```

**โค้ดถูก:**
```rust
// ถูก: เพิ่ม self-loops ก่อนเสมอ
fn correct_normalized_adj(g: &Graph) -> Vec<f32> {
    let n = g.num_nodes;
    let mut a = vec![0.0f32; n * n];
    // เพิ่ม self-loops ก่อน
    for i in 0..n {
        a[i * n + i] = 1.0;  // ทุก node มี degree ≥ 1
    }
    for &(u, v, _) in &g.edges {
        a[u * n + v] = 1.0;
        a[v * n + u] = 1.0;
    }
    // คำนวณ degree (ตอนนี้ทุก node มีอย่างน้อย 1 จาก self-loop)
    let mut deg = vec![0.0f32; n];
    for i in 0..n {
        for j in 0..n {
            deg[i] += a[i * n + j];
        }
    }
    // safe: deg[i] >= 1 เสมอ
    let d_inv_sqrt: Vec<f32> = deg.iter()
        .map(|&d| if d > 0.0 { 1.0 / d.sqrt() } else { 0.0 })
        .collect();
    // ...
}
```

**กฎ:** ใน `normalized_adj()` ให้เพิ่ม self-loop ก่อนเสมอ และใส่ guard `if d > 0.0` เพื่อความปลอดภัย

---

### Pitfall 2: Index Out of Bounds ใน Flat Matrix

**อาการ:** panic ที่ runtime พร้อม "index out of bounds" เมื่อ access `mat[i * cols + j]`

**สาเหตุที่พบบ่อย:** สับสนระหว่าง row size กับ column size ของ matrix ในการ matmul

**โค้ดผิด:**
```rust
// A(m×k) @ B(k×n) -> C(m×n)
fn bad_matmul(a: &[f32], b: &[f32], m: usize, k: usize, n: usize) -> Vec<f32> {
    let mut c = vec![0.0f32; m * n];
    for i in 0..m {
        for j in 0..n {
            for l in 0..k {
                // ผิด: ใช้ n แทน k เป็น stride ของ A
                c[i * n + j] += a[i * n + l] * b[l * n + j];
                //                     ^^^ ควรเป็น k ไม่ใช่ n
            }
        }
    }
    c
}
```

**โค้ดถูก:**
```rust
fn matmul(a: &[f32], b: &[f32], m: usize, k: usize, n: usize) -> Vec<f32> {
    let mut c = vec![0.0f32; m * n];
    for i in 0..m {
        for j in 0..n {
            let mut sum = 0.0f32;
            for l in 0..k {
                sum += a[i * k + l] * b[l * n + j];
                //         ^^^ stride ของ A คือ k (จำนวน columns ของ A)
                //                         ^^^ stride ของ B คือ n (จำนวน columns ของ B)
            }
            c[i * n + j] = sum;
        }
    }
    c
}
```

**วิธีจำ:** index `[i][j]` ใน matrix ขนาด `rows×cols` คือ `i * cols + j` ไม่ใช่ `i * rows + j`

---

### Pitfall 3: Numerically Unstable Softmax

**อาการ:** loss เป็น NaN หลัง forward pass หนึ่งสองครั้ง

**สาเหตุ:** logits ที่มีค่าสูงมาก เช่น [100, 200, 300] → `exp(300)` = infinity → softmax คำนวณ `inf/inf` = NaN

**โค้ดผิด:**
```rust
fn bad_softmax(x: &[f32]) -> Vec<f32> {
    let exps: Vec<f32> = x.iter().map(|&v| v.exp()).collect();  // อาจ overflow!
    let sum: f32 = exps.iter().sum();
    exps.iter().map(|&e| e / sum).collect()
}
```

**โค้ดถูก:**
```rust
fn softmax(x: &[f32]) -> Vec<f32> {
    // Subtract max ก่อน: exp(xᵢ - max) ไม่ overflow
    // คณิตศาสตร์เหมือนกัน: softmax(x) = softmax(x - c) สำหรับค่าคงที่ c ใดก็ได้
    let max_v = x.iter().cloned().fold(f32::NEG_INFINITY, f32::max);
    let exps: Vec<f32> = x.iter().map(|&v| (v - max_v).exp()).collect();
    let sum: f32 = exps.iter().sum();
    exps.iter().map(|&e| e / sum).collect()
}
```

**คำพิสูจน์:**
```
softmax(xᵢ) = exp(xᵢ) / Σⱼ exp(xⱼ)
            = exp(xᵢ - c) / Σⱼ exp(xⱼ - c)   สำหรับทุกค่า c
```
เมื่อ c = max(x): `exp(max - max) = exp(0) = 1` และ `exp(xᵢ - max) ≤ 1` ทุกตัว จึงไม่ overflow

**หลักสำหรับ log ด้วย:** ใช้ `p.max(1e-7)` ก่อน `.ln()` เสมอ เพื่อป้องกัน `log(0)` = -infinity ใน cross-entropy

---

### Pitfall 4: สับสน Undirected กับ Directed Graph ใน Adjacency List

**อาการ:** message passing ทำงานไม่สมมาตร, node A ได้รับ message จาก B แต่ B ไม่ได้จาก A

**สาเหตุ:** เพิ่ม edge เป็น directed (ทิศทางเดียว) ใน adjacency list

**โค้ดผิด:**
```rust
fn bad_add_edge(&mut self, u: usize, v: usize, w: f32) {
    self.edges.push((u, v, w));
    self.adj[u].push(v);  // เพิ่มแค่ u->v
    // ลืม: self.adj[v].push(u);  // v->u ด้วย!
}
```

**โค้ดถูก:**
```rust
fn add_edge(&mut self, u: usize, v: usize, w: f32) {
    self.edges.push((u, v, w));
    if !self.adj[u].contains(&v) {
        self.adj[u].push(v);  // u -> v
    }
    if !self.adj[v].contains(&u) {
        self.adj[v].push(u);  // v -> u (undirected!)
    }
}
```

**หมายเหตุ:** GCN ต้องการ symmetric adjacency matrix ถ้าต้องการ directed GNN (เช่น DGCN) ต้องใช้ normalized adjacency อีกแบบ

---

### Pitfall 5: Overfit เพราะ Positive Edges มากกว่า Negative ใน Link Prediction

**อาการ:** BCE loss ลดลงเร็วมากแต่ model predict ทุก pair เป็น 1 (precision ต่ำ)

**สาเหตุ:** ใช้ positive:negative ratio ที่ไม่สมดุล เช่น 1000 positives แต่ 10 negatives

**โค้ดผิด:**
```rust
// ผิด: sample negative น้อยเกินไป
let negatives = negative_sample(n_nodes, &pos_edges, 10, seed);
// pos:neg = 1000:10 = 100:1 -> model เรียนรู้แค่ "predict 1 เสมอ"
```

**โค้ดถูก:**
```rust
// ถูก: sample negative ให้เท่ากับ positive (1:1 ratio)
let n_pos = pos_edges.len();
let negatives = negative_sample(n_nodes, &pos_edges, n_pos, seed);
// pos:neg = 1:1 -> balanced training
```

**เทคนิคเพิ่มเติม:** บาง paper ใช้ negative:positive = 5:1 หรือ 10:1 ขึ้นอยู่กับ density ของ graph sparse graph มักต้องการ negative ratio สูงกว่า

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build แบบ optimized
cargo build --release

# รันด้วย optimized binary
./target/release/graph-nn
```

**Cargo.toml สำหรับ production:**
```toml
[profile.release]
opt-level = 3
lto = true          # Link-time optimization
codegen-units = 1   # ลด compile parallelism เพื่อ optimize ดีขึ้น
panic = "abort"     # ลด binary size

[dependencies]
rand = { version = "0.8", features = ["small_rng"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

### Serialize และ Save Model

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
pub struct SavedModel {
    pub gcn1_weights: Vec<f32>,
    pub gcn1_bias: Vec<f32>,
    pub gcn2_weights: Vec<f32>,
    pub gcn2_bias: Vec<f32>,
    pub in_dim: usize,
    pub hidden_dim: usize,
    pub out_dim: usize,
}

impl SavedModel {
    pub fn save(&self, path: &str) -> std::io::Result<()> {
        let json = serde_json::to_string_pretty(self)?;
        std::fs::write(path, json)?;
        Ok(())
    }

    pub fn load(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let json = std::fs::read_to_string(path)?;
        Ok(serde_json::from_str(&json)?)
    }
}
```

### Benchmark ด้วย Criterion

```toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "gnn_bench"
harness = false
```

```rust
// benches/gnn_bench.rs
use criterion::{criterion_group, criterion_main, Criterion};

fn bench_gcn_forward(c: &mut Criterion) {
    let n_nodes = 1000;
    let in_dim = 64;
    let out_dim = 128;
    // ...
    c.bench_function("gcn_forward_1000nodes", |b| {
        b.iter(|| gcn.forward(&feat, &norm_adj, n_nodes))
    });
}

criterion_group!(benches, bench_gcn_forward);
criterion_main!(benches);
```

### Docker Container

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/graph-nn /usr/local/bin/
ENTRYPOINT ["graph-nn"]
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Implement Graph Attention Network (GAT)

GAT (Veličković et al. 2018) เพิ่ม attention mechanism เพื่อให้ neighbors ที่สำคัญกว่ามี weight มากกว่า:

```
αᵢⱼ = softmax_j(LeakyReLU(a^T [Whᵢ || Whⱼ]))
h'ᵢ = σ(Σⱼ αᵢⱼ Whⱼ)
```

**งาน:** สร้าง `GatLayer` struct ที่มี:
- `attention_weights: Vec<f32>` ขนาด `2 * out_dim` (เพื่อ compute αᵢⱼ)
- method `compute_attention(hi: &[f32], hj: &[f32]) -> f32`
- method `forward_with_attention(h, adj, n_nodes) -> Vec<f32>`
- Tests ที่ตรวจว่า attention weights sum to 1 สำหรับทุก node

**Hint:** ต้อง compute attention ต่อ edge ก่อน แล้วค่อย softmax ต่อ node

---

### แบบฝึกหัดที่ 2: Implement DeepWalk Node Embeddings

DeepWalk (Perozzi et al. 2014) สร้าง node embeddings โดยใช้ random walks + Word2Vec:

1. Random walk: เริ่มจาก node v สุ่มเดินตาม edge ไปเรื่อย ๆ t ขั้น
2. ได้ sequence ของ nodes เหมือน sequence ของ words
3. ใช้ Skip-gram model (Word2Vec) เรียนรู้ embedding ที่ทำให้ nodes ที่อยู่ใกล้กันมี embedding คล้ายกัน

**งาน:** implement `random_walk(start: usize, length: usize, seed: u64) -> Vec<usize>` ใน `graph.rs` และ `skipgram_loss(center: usize, context: Vec<usize>, embeddings: &[f32]) -> f32`

**Hint:** ใช้ `negative_sample` ที่มีอยู่แล้วสำหรับ Skip-gram negative sampling

---

### แบบฝึกหัดที่ 3: Community Detection ด้วย GNN

Community detection คือการหากลุ่มของ nodes ที่มี connections หนาแน่นภายในกลุ่ม แต่ connection น้อยระหว่างกลุ่ม

**งาน:** implement `modularity_loss(logits: &[Vec<f32>], adj: &[Vec<usize>]) -> f32` โดยใช้ modularity function:

```
Q = (1/2m) Σᵢⱼ [Aᵢⱼ - kᵢkⱼ/(2m)] δ(cᵢ, cⱼ)
```

ที่ m = จำนวน edges, kᵢ = degree ของ node i, δ = 1 ถ้า same community

**จากนั้น:** train GCN ด้วย modularity loss บน Karate Club graph (34 nodes, 78 edges) และ visualize ผล clustering

---

### แบบฝึกหัดที่ 4: Temporal Graph Network (TGN) เบื้องต้น

ใน real-world graph ส่วนใหญ่ edges มี timestamps (เช่น retweet, transaction, call) TGN เพิ่ม time dimension ให้กับ GNN:

```
h(t) = update(h(t-1), aggregate_time_neighbors(v, t))
```

**งาน:**
1. เพิ่ม timestamp ให้กับ edges: `edges: Vec<(usize, usize, f32, u64)>` โดย u64 คือ timestamp
2. implement `temporal_neighbors(node: usize, before_time: u64) -> Vec<usize>` ที่ return เฉพาะ neighbors ที่ edge timestamp < before_time
3. implement `time_encode(dt: f64) -> Vec<f32>` ด้วย sinusoidal encoding เหมือน Transformer positional encoding
4. Tests ที่ตรวจว่า temporal_neighbors ไม่รวม future edges

---

### แบบฝึกหัดที่ 5: Heterogeneous Graph Neural Network

Real-world graph มักมีหลาย type ของ nodes และ edges (heterogeneous graph) เช่น:
- Academic graph: nodes = Papers, Authors, Venues; edges = writes, publishes, cites
- E-commerce: nodes = Users, Items, Categories; edges = buys, belongs_to, similar_to

**งาน:**
1. สร้าง `HeteroGraph` struct ที่มี `node_types: Vec<String>`, `edge_types: Vec<(String, String, String)>`
2. implement `HeteroGCN` ที่มี weight matrix แยกต่างหากสำหรับแต่ละ edge type
3. implement `type_based_aggregate(node: usize, edge_type: &str) -> Vec<f32>`

---

### แบบฝึกหัดที่ 6: Knowledge Graph Embedding ด้วย TransE

TransE (Bordes et al. 2013) เป็น knowledge graph embedding model ที่ใช้ relation เป็น translation:

```
h + r ≈ t   (หัว entity + relation ≈ หาง entity)
Loss = Σ [d(h+r,t) - d(h'+r,t') + γ]₊
```

**งาน:** implement `TransE` ด้วย:
- `entity_embeddings: Vec<f32>` (n_entities × dim)
- `relation_embeddings: Vec<f32>` (n_relations × dim)
- `score(head: usize, rel: usize, tail: usize) -> f32`
- `margin_loss(pos: &[(usize,usize,usize)], neg: &[(usize,usize,usize)], margin: f32) -> f32`
- Tests บน FB15K-237 mini subset

---

## สรุป

ในโปรเจคนี้เราได้สร้าง Graph Neural Network framework สมบูรณ์แบบตั้งแต่ต้นด้วย Pure Rust:

**สิ่งที่สร้าง:**
- `Graph` struct รองรับทั้ง adjacency list และ CSR format พร้อม `serde` serialization
- `FeatureMatrix` พร้อม L2 normalize และ min-max normalize
- `MessagePassing` framework (sum/mean/max aggregation)
- `GcnLayer` ด้วย Xavier initialization และ forward pass H' = σ(ĝHW)
- Normalized adjacency D^{-1/2} A D^{-1/2} พร้อม self-loops
- Two-layer GCN สำหรับ node classification ด้วย cross-entropy loss และ softmax
- `GraphSage` ด้วย mean/max aggregator และ inductive learning
- `LinkPredictor` ด้วย dot product, hadamard, cosine similarity
- Binary cross-entropy loss และ negative sampling
- Neighborhood sampling สำหรับ mini-batch training

**Pattern สำคัญที่ได้เรียน:**
- **Flat matrix representation** — `Vec<f32>` row-major เป็น fundamental pattern ใน numerical computing ใน Rust
- **Precomputed normalization** — การ compute ĝ ครั้งเดียวก่อน training เป็น standard practice
- **Seeded RNG** — `StdRng::seed_from_u64(seed)` สำหรับ reproducible experiments
- **Numerical stability** — subtract-max softmax, log-clipping, sigmoid formulation

**GNN ใน production world:**
- **PyG (PyTorch Geometric)** และ **DGL (Deep Graph Library)** เป็น framework หลักที่ใช้ Python
- **Rust GNN** ยังอยู่ในช่วง early stage แต่มี potential สูงสำหรับ inference engine
- **ONNX** สามารถ export PyTorch model มา serve ด้วย `onnxruntime` crate ใน Rust ได้

**โปรเจคถัดไป (Project J01: Yew SPA)** จะเปลี่ยนมุมมองอย่างสิ้นเชิง — จาก ML algorithms ไปสู่ Full-Stack Web Development ด้วย **Yew framework** ที่ compile Rust เป็น WebAssembly และรันใน browser แนวคิด component-based architecture ที่คุณจะเรียนใน Yew คล้ายกับ React และจะเปิดประตูสู่ Rust Web Development อย่างสมบูรณ์

---

**โปรเจคก่อนหน้า:** [Project I09: Q-Learning](project-i09-q-learning.md) | **โปรเจคถัดไป:** [Project J01: Yew SPA](project-j01-yew-spa.md)
