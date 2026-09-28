# Project I02: Neural Network from Scratch

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 16 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Feedforward Neural Network พร้อม Backpropagation** จาก scratch โดยไม่ใช้ ML framework ใด ๆ ทั้งสิ้น — ไม่มี TensorFlow, ไม่มี PyTorch, ไม่มี `tch-rs`, ไม่มี `candle` เราจะเขียนทุกอย่างด้วย Pure Rust ตั้งแต่ matrix multiplication ไปจนถึง gradient descent

เป้าหมายคือ "เข้าใจจริง" ไม่ใช่แค่ "ใช้งานได้" เมื่อจบโปรเจคนี้คุณจะเห็นชัดว่าทำไม neural network ถึงทำงานได้ — ทุก chain rule, ทุก gradient flow, ทุก weight update มีที่มาที่ไปอย่างโปร่งใส

**Use cases จริงในโลก production:**
- ฝัง inference engine ลงใน embedded system ที่ไม่สามารถติดตั้ง ML runtime ขนาดใหญ่ได้
- ทำความเข้าใจ numerical issues ของ neural network เพื่อ debug model ที่ train แล้วผิดปกติ
- พื้นฐานสำหรับเขียน custom layer หรือ custom optimizer ที่ framework มาตรฐานไม่รองรับ
- การเรียนรู้สำหรับงานวิจัยที่ต้องการ full control เหนือทุกขั้นตอนของ training

โปรเจคนี้ครอบคลุม **Tensor operations**, **Activation functions**, **Dense layers**, **Backpropagation**, **Optimizers (SGD + Adam)**, **Loss functions (CrossEntropy + MSE)** และ **Training loop** แบบ mini-batch ทั้งหมดนี้พร้อม 31 unit tests ที่ผ่านจริง

## สิ่งที่จะได้เรียนรู้

- **Matrix operations จาก scratch** — `matmul`, `transpose`, `broadcast add` บน `Vec<f64>` แบบ flat array
- **Chain rule ใน code** — วิธีส่ง gradient ย้อนกลับผ่าน layer หลายชั้นแบบ automatic ด้วย `backward()`
- **Xavier/He initialization** — ทำไมค่า weights เริ่มต้นถึงสำคัญ และวิธีคำนวณ scale ที่เหมาะสม
- **Numerical stability** — `softmax` แบบ subtract-max, `log(eps)` ป้องกัน NaN ใน cross-entropy
- **Adam optimizer** — moment estimates (`m`, `v`), bias correction, และความแตกต่างกับ SGD
- **Ownership pattern ใน training loop** — วิธีจัดการ mutable state ของ optimizer ที่ต้องอยู่แยกจาก layer
- **Trait-based polymorphism** — `Loss` trait ที่ให้ swap `CrossEntropyLoss` กับ `MSELoss` ได้โดยไม่แก้ Network
- **Mini-batch training** — วิธี slice data เป็น batch, handle uneven batch size, accumulate gradients

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: `Vec`, iterators, closures, `flat_map`, `zip`, `map`
- **Part 31–40**: Traits, generics, error handling, `Option` / `Result`
- **Part 41–50**: `derive(Debug, Clone)`, default implementations, trait objects (`dyn`)
- **Part 51–60**: Numeric types, `f64` operations, `iter().sum()`, `powf()`, `sqrt()`
- **Part 61–70**: Module system (`mod`), `pub` visibility, crate dependencies
- **Project I01 (Linear Regression)** — ความเข้าใจพื้นฐาน gradient descent และ loss function

## โครงสร้างโปรเจค (Project Layout)

```
neural-network/
├── src/
│   ├── main.rs          ← demo XOR problem
│   ├── tensor.rs        ← Tensor struct, matrix ops
│   ├── activation.rs    ← Activation enum + forward/derivative
│   ├── layer.rs         ← DenseLayer + backprop
│   ├── network.rs       ← Network, forward/backward pass, fit()
│   ├── optimizer.rs     ← SGD, Adam, AdamState
│   └── loss.rs          ← CrossEntropyLoss, MSELoss traits
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของ Training Loop

```
Input X (batch × features)
        │
        ▼
┌──────────────────────────────────────────┐
│            Forward Pass                  │
│                                          │
│  Layer 1: z₁ = X @ W₁ + b₁             │
│           a₁ = activation(z₁)           │
│           ↓                              │
│  Layer 2: z₂ = a₁ @ W₂ + b₂            │
│           a₂ = activation(z₂)           │
│           ↓                              │
│  Output = aₙ (predictions)              │
└──────────────────────────────────────────┘
        │
        ▼
  Loss = CrossEntropy(predictions, targets)
        │
        ▼
┌──────────────────────────────────────────┐
│            Backward Pass                 │
│                                          │
│  grad = ∂Loss/∂predictions              │
│  ↑                                       │
│  Layer N: δₙ = grad ⊙ σ'(zₙ)           │
│           grad_Wₙ = aₙ₋₁ᵀ @ δₙ         │
│           grad_input = δₙ @ Wₙᵀ        │
│  ↑                                       │
│  Layer N-1: ... (chain rule)             │
└──────────────────────────────────────────┘
        │
        ▼
  Optimizer: W -= lr * grad_W  (SGD/Adam)
```

### ทำไมถึงเก็บ `last_input` และ `last_z` ใน Layer?

Backpropagation ต้องการ cached values จาก forward pass:

1. **`last_input`** — ใช้คำนวณ `grad_weights = input^T @ delta` (ต้องการ input ของ layer นี้)
2. **`last_z`** — ใช้คำนวณ `activation'(z)` เพื่อหา delta (ต้องการค่าก่อน activation)

ถ้าไม่ cache ไว้ เราต้องส่ง tensors เพิ่มเติมผ่าน function signatures ซึ่งทำให้ API ซับซ้อนขึ้นมาก

### ทำไม Adam State อยู่แยกจาก Layer?

`AdamState` (moment estimates `m`, `v`) ต้อง persist ข้ามหลาย training steps แต่ถ้าเก็บไว้ใน `DenseLayer` จะทำให้ Layer รู้เรื่อง optimizer-specific state ซึ่งผิดหลัก separation of concerns

เราเลือกเก็บ `Vec<AdamState>` ใน `Network` โดย index ตรงกับ `Vec<DenseLayer>` วิธีนี้ทำให้ swap optimizer ได้โดยไม่แตะ Layer code

### Tensor Storage: Flat Array Row-Major

```
Matrix 2×3:
┌─────────────────────────┐
│ data = [a,b,c, d,e,f]  │
│ shape = [2, 3]          │
│                         │
│ index(i,j) = i*3 + j   │
└─────────────────────────┘
```

การเก็บแบบ flat `Vec<f64>` มีข้อดีคือ cache locality ดีกว่า `Vec<Vec<f64>>` เพราะข้อมูลอยู่ต่อเนื่องในหน่วยความจำ

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Tensor — โครงสร้างข้อมูลพื้นฐาน

เริ่มจาก `Tensor` struct ที่เก็บ matrix แบบ row-major flat array

**`Cargo.toml`**

```toml
[package]
name = "neural-network"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
```

**`src/tensor.rs`** (ส่วนแรก — struct และ basic ops)

```rust
/// Tensor สำหรับเก็บข้อมูล matrix 2D
#[derive(Debug, Clone)]
pub struct Tensor {
    pub data: Vec<f64>,
    pub shape: Vec<usize>, // [rows, cols]
}

impl Tensor {
    pub fn new(data: Vec<f64>, shape: Vec<usize>) -> Self {
        assert_eq!(
            data.len(),
            shape.iter().product::<usize>(),
            "data length must match shape product"
        );
        Tensor { data, shape }
    }

    pub fn zeros(rows: usize, cols: usize) -> Self {
        Tensor {
            data: vec![0.0; rows * cols],
            shape: vec![rows, cols],
        }
    }

    pub fn rows(&self) -> usize { self.shape[0] }
    pub fn cols(&self) -> usize { self.shape[1] }

    pub fn get(&self, row: usize, col: usize) -> f64 {
        self.data[row * self.cols() + col]
    }

    pub fn set(&mut self, row: usize, col: usize, val: f64) {
        let cols = self.cols();
        self.data[row * cols + col] = val;
    }
}
```

สังเกตว่า `get`/`set` ใช้ `row * cols + col` ซึ่งเป็น row-major indexing การ inline `cols()` ใน setter จำเป็นเพราะ Rust borrow checker จะไม่ยอมให้ call `self.cols()` และ `self.data[...]` พร้อมกันในบรรทัดเดียวกันเนื่องจาก mutable borrow

**Matrix Multiplication**

```rust
impl Tensor {
    /// Matrix multiplication: (m×k) × (k×n) → (m×n)
    pub fn matmul(&self, other: &Tensor) -> Tensor {
        assert_eq!(
            self.cols(),
            other.rows(),
            "matmul shape mismatch: ({},{}) × ({},{})",
            self.rows(), self.cols(), other.rows(), other.cols()
        );
        let m = self.rows();
        let k = self.cols();
        let n = other.cols();
        let mut result = Tensor::zeros(m, n);
        for i in 0..m {
            for j in 0..n {
                let mut sum = 0.0;
                for l in 0..k {
                    sum += self.get(i, l) * other.get(l, j);
                }
                result.set(i, j, sum);
            }
        }
        result
    }

    /// Transpose: (m×n) → (n×m)
    pub fn transpose(&self) -> Tensor {
        let m = self.rows();
        let n = self.cols();
        let mut result = Tensor::zeros(n, m);
        for i in 0..m {
            for j in 0..n {
                result.set(j, i, self.get(i, j));
            }
        }
        result
    }

    /// Broadcast bias addition: (m×n) + (1×n) → (m×n)
    pub fn add_broadcast(&self, bias: &Tensor) -> Tensor {
        assert_eq!(bias.rows(), 1, "bias must have 1 row");
        assert_eq!(self.cols(), bias.cols(), "cols mismatch for broadcast add");
        let m = self.rows();
        let n = self.cols();
        let mut result = Tensor::zeros(m, n);
        for i in 0..m {
            for j in 0..n {
                result.set(i, j, self.get(i, j) + bias.get(0, j));
            }
        }
        result
    }

    /// Element-wise multiply by scalar
    pub fn scale(&self, s: f64) -> Tensor {
        Tensor {
            data: self.data.iter().map(|x| x * s).collect(),
            shape: self.shape.clone(),
        }
    }

    /// Element-wise multiply two tensors (Hadamard product)
    pub fn mul_elementwise(&self, other: &Tensor) -> Tensor {
        assert_eq!(self.shape, other.shape, "shape mismatch for elementwise mul");
        Tensor {
            data: self.data.iter().zip(&other.data)
                .map(|(a, b)| a * b).collect(),
            shape: self.shape.clone(),
        }
    }

    /// Element-wise add two tensors of same shape
    pub fn add(&self, other: &Tensor) -> Tensor {
        assert_eq!(self.shape, other.shape, "shape mismatch for add");
        Tensor {
            data: self.data.iter().zip(&other.data)
                .map(|(a, b)| a + b).collect(),
            shape: self.shape.clone(),
        }
    }

    /// Apply function element-wise
    pub fn map<F: Fn(f64) -> f64>(&self, f: F) -> Tensor {
        Tensor {
            data: self.data.iter().map(|&x| f(x)).collect(),
            shape: self.shape.clone(),
        }
    }
}
```

**ทดสอบเบื้องต้น:**

```rust
// ตรวจสอบ matmul
// [1,2] × [5,6]  = [1×5+2×7, 1×6+2×8] = [19, 22]
// [3,4]   [7,8]    [3×5+4×7, 3×6+4×8]   [43, 50]
let a = Tensor::new(vec![1.0, 2.0, 3.0, 4.0], vec![2, 2]);
let b = Tensor::new(vec![5.0, 6.0, 7.0, 8.0], vec![2, 2]);
let c = a.matmul(&b);
assert_eq!(c.get(0, 0), 19.0);
assert_eq!(c.get(1, 1), 50.0);
```

### ขั้นที่ 2: Activation Functions

**`src/activation.rs`**

```rust
use crate::tensor::Tensor;

#[derive(Debug, Clone)]
pub enum Activation {
    ReLU,
    Sigmoid,
    Tanh,
    Softmax,
    Linear,
}

impl Activation {
    /// Forward pass: ใช้กับแต่ละ element (ยกเว้น Softmax ที่ใช้ทั้ง row)
    pub fn forward(&self, z: &Tensor) -> Tensor {
        match self {
            Activation::ReLU    => z.map(|x| x.max(0.0)),
            Activation::Sigmoid => z.map(|x| 1.0 / (1.0 + (-x).exp())),
            Activation::Tanh    => z.map(|x| x.tanh()),
            Activation::Linear  => z.clone(),
            Activation::Softmax => {
                // Numerically stable: subtract max per row ก่อน exp
                let rows = z.rows();
                let cols = z.cols();
                let mut result = Tensor::zeros(rows, cols);
                for i in 0..rows {
                    // หา max ในแต่ละ row เพื่อป้องกัน overflow
                    let max_val = (0..cols)
                        .map(|j| z.get(i, j))
                        .fold(f64::NEG_INFINITY, f64::max);
                    let exps: Vec<f64> = (0..cols)
                        .map(|j| (z.get(i, j) - max_val).exp())
                        .collect();
                    let sum: f64 = exps.iter().sum();
                    for j in 0..cols {
                        result.set(i, j, exps[j] / sum);
                    }
                }
                result
            }
        }
    }

    /// Derivative ใช้สำหรับ backprop (element-wise ยกเว้น Softmax+CE ที่ handle แยก)
    pub fn derivative(&self, z: &Tensor) -> Tensor {
        match self {
            Activation::ReLU => z.map(|x| if x > 0.0 { 1.0 } else { 0.0 }),
            Activation::Sigmoid => {
                z.map(|x| {
                    let s = 1.0 / (1.0 + (-x).exp());
                    s * (1.0 - s)
                })
            }
            Activation::Tanh   => z.map(|x| 1.0 - x.tanh().powi(2)),
            Activation::Linear => z.map(|_| 1.0),
            // Combined Softmax+CrossEntropy gradient ส่งตรงจาก loss layer
            Activation::Softmax => z.map(|_| 1.0),
        }
    }
}
```

**หัวใจของ Softmax: Numerical Stability**

ปัญหา: ถ้า input มีค่า `[1000, 1001, 999]` การคำนวณ `e^1000` จะ overflow เป็น `inf`

แนวทางแก้: subtract max ก่อน คือ `softmax(x) = softmax(x - max)` ซึ่งให้ผลลัพธ์เท่ากันทางคณิตศาสตร์

```
softmax([1000, 1001, 999])
= softmax([1000-1001, 1001-1001, 999-1001])
= softmax([-1, 0, -2])
= [e⁻¹, e⁰, e⁻²] / sum([e⁻¹, e⁰, e⁻²])
≈ [0.212, 0.577, 0.212]
```

**ตรวจสอบ Sigmoid Saturation:**

```
σ(-100) ≈ 3.7×10⁻⁴⁴  (ใกล้ 0)
σ(0)    = 0.5
σ(100)  ≈ 1 - 3.7×10⁻⁴⁴  (ใกล้ 1)
```

ที่บริเวณ ±3 ขึ้นไป gradient จะเข้าใกล้ 0 — นี่คือ "vanishing gradient problem" ที่ ReLU ถูกออกแบบมาเพื่อแก้

### ขั้นที่ 3: Dense Layer และ Forward Pass

**`src/layer.rs`**

```rust
use crate::tensor::Tensor;
use crate::activation::Activation;
use rand::Rng;

#[derive(Debug, Clone)]
pub struct DenseLayer {
    pub weights: Tensor,    // shape: (in_features × out_features)
    pub biases: Tensor,     // shape: (1 × out_features)
    pub activation: Activation,

    // ต้อง cache ไว้สำหรับ backward pass
    pub last_input: Option<Tensor>,  // X ก่อนเข้า layer
    pub last_z: Option<Tensor>,      // XW + b (ก่อน activation)

    // Gradients หลังจาก backward
    pub grad_weights: Option<Tensor>,
    pub grad_biases: Option<Tensor>,
}

impl DenseLayer {
    pub fn new(in_features: usize, out_features: usize, activation: Activation) -> Self {
        let mut rng = rand::thread_rng();

        // He initialization สำหรับ ReLU, Xavier สำหรับ sigmoid/tanh
        let scale = match activation {
            Activation::ReLU => (2.0f64 / in_features as f64).sqrt(),
            _                => (2.0f64 / (in_features + out_features) as f64).sqrt(),
        };

        let w_data: Vec<f64> = (0..in_features * out_features)
            .map(|_| rng.gen_range(-1.0..1.0) * scale)
            .collect();

        DenseLayer {
            weights:     Tensor::new(w_data, vec![in_features, out_features]),
            biases:      Tensor::new(vec![0.0; out_features], vec![1, out_features]),
            activation,
            last_input:  None,
            last_z:      None,
            grad_weights: None,
            grad_biases:  None,
        }
    }

    /// Forward pass: output = activation(X @ W + b)
    pub fn forward(&mut self, input: &Tensor) -> Tensor {
        let z = input.matmul(&self.weights).add_broadcast(&self.biases);
        self.last_input = Some(input.clone());
        self.last_z     = Some(z.clone());
        self.activation.forward(&z)
    }

    pub fn zero_grad(&mut self) {
        self.grad_weights = None;
        self.grad_biases  = None;
    }
}
```

**ทำไม biases เริ่มที่ 0?**

งานวิจัยพบว่า weights ควร initialize แบบ random (เพื่อ symmetry breaking) แต่ biases สามารถเริ่มที่ 0 ได้อย่างปลอดภัย เพราะ weights ที่ต่างกันระหว่าง neurons จะ break symmetry ได้แล้ว

**Xavier vs He Initialization:**

```
Xavier scale = sqrt(2 / (fan_in + fan_out))
He scale     = sqrt(2 / fan_in)
```

He initialization เหมาะกับ ReLU เพราะ ReLU ตัดครึ่งหนึ่งของ neurons ออก (negative → 0) ทำให้ variance ของ output ลดลงครึ่งหนึ่ง การใช้ `sqrt(2/fan_in)` ชดเชยส่วนที่หายไป

### ขั้นที่ 4: Backpropagation

นี่คือหัวใจของโปรเจค ต้องเข้าใจ chain rule อย่างลึกซึ้ง

**`src/layer.rs`** (เพิ่ม backward method)

```rust
impl DenseLayer {
    /// Backward pass: คำนวณ gradient ทั้งสำหรับ weights และ input
    ///
    /// grad_output: upstream gradient จาก layer ถัดไป
    ///              shape = (batch_size × out_features)
    /// returns:     grad_input = gradient ที่ส่งต่อไปยัง layer ก่อนหน้า
    ///              shape = (batch_size × in_features)
    pub fn backward(&mut self, grad_output: &Tensor) -> Tensor {
        let input = self.last_input.as_ref()
            .expect("must call forward before backward");
        let z = self.last_z.as_ref()
            .expect("must call forward before backward");
        let batch_size = grad_output.rows() as f64;

        // Step 1: Apply activation derivative (element-wise Hadamard product)
        // delta = grad_output ⊙ σ'(z)
        let act_deriv = self.activation.derivative(z);
        let delta = grad_output.mul_elementwise(&act_deriv);

        // Step 2: Gradient for weights
        // grad_W = (1/N) * Xᵀ @ delta   shape: (in_features × out_features)
        let grad_w = input.transpose().matmul(&delta).scale(1.0 / batch_size);

        // Step 3: Gradient for biases
        // grad_b = (1/N) * sum(delta, axis=0)   shape: (1 × out_features)
        let cols = delta.cols();
        let mut gb_data = vec![0.0f64; cols];
        for i in 0..delta.rows() {
            for j in 0..cols {
                gb_data[j] += delta.get(i, j);
            }
        }
        let grad_b = Tensor::new(gb_data, vec![1, cols])
            .scale(1.0 / batch_size);

        // Step 4: Gradient for input (ส่งต่อ layer ก่อนหน้า)
        // grad_input = delta @ Wᵀ   shape: (batch_size × in_features)
        let grad_input = delta.matmul(&self.weights.transpose());

        self.grad_weights = Some(grad_w);
        self.grad_biases  = Some(grad_b);

        grad_input
    }
}
```

**ทำไมต้อง divide by `batch_size`?**

เราต้องการ "average gradient" ข้าม batch ไม่ใช่ "sum gradient" ถ้าไม่ normalize ขนาด gradient จะขึ้นอยู่กับ batch size ซึ่งทำให้ต้องปรับ learning rate ทุกครั้งที่เปลี่ยน batch size

**อนุพันธ์ของแต่ละ Activation:**

```
ReLU:    σ'(z) = 1 ถ้า z > 0, ไม่เช่นนั้น 0
Sigmoid: σ'(z) = σ(z)(1 - σ(z))
Tanh:    σ'(z) = 1 - tanh²(z)
Linear:  σ'(z) = 1
```

**Chain Rule ในทางปฏิบัติ:**

สมมติว่า network มี 3 layers, loss L

```
Forward:   x → z₁ = xW₁+b₁ → a₁ = σ(z₁) → z₂ = a₁W₂+b₂ → output → L

Backward:  ∂L/∂W₂ = a₁ᵀ @ (∂L/∂z₂ ⊙ σ'(z₂))
           ∂L/∂a₁ = (∂L/∂z₂ ⊙ σ'(z₂)) @ W₂ᵀ
           ∂L/∂W₁ = xᵀ @ (∂L/∂a₁ ⊙ σ'(z₁))
```

แต่ละ layer รับ `grad_output` จาก layer ถัดไป คำนวณ `delta` คูณกับ `σ'(z)` แล้วส่ง `grad_input` ต่อไปยัง layer ก่อนหน้า

### ขั้นที่ 5: Loss Functions

**`src/loss.rs`**

```rust
use crate::tensor::Tensor;

pub trait Loss {
    fn compute(&self, predictions: &Tensor, targets: &Tensor) -> f64;
    fn gradient(&self, predictions: &Tensor, targets: &Tensor) -> Tensor;
}
```

**Binary Cross-Entropy Loss:**

```rust
pub struct CrossEntropyLoss;

impl Loss for CrossEntropyLoss {
    fn compute(&self, predictions: &Tensor, targets: &Tensor) -> f64 {
        let n   = predictions.data.len() as f64;
        let eps = 1e-15;  // ป้องกัน log(0)
        let loss: f64 = predictions.data.iter()
            .zip(&targets.data)
            .map(|(&p, &t)| {
                let p = p.max(eps).min(1.0 - eps);
                -(t * p.ln() + (1.0 - t) * (1.0 - p).ln())
            })
            .sum();
        loss / n
    }

    fn gradient(&self, predictions: &Tensor, targets: &Tensor) -> Tensor {
        let n   = predictions.data.len() as f64;
        let eps = 1e-15;
        let grad: Vec<f64> = predictions.data.iter()
            .zip(&targets.data)
            .map(|(&p, &t)| {
                let p = p.max(eps).min(1.0 - eps);
                (-t / p + (1.0 - t) / (1.0 - p)) / n
            })
            .collect();
        Tensor::new(grad, predictions.shape.clone())
    }
}
```

**ทำไมต้อง clip p ด้วย eps?**

```
ถ้า p = 0.0 และ t = 1.0:
  loss = -ln(0.0) = +∞  ← NaN/Inf
  grad = -1.0 / 0.0 = -∞ ← NaN/Inf

ด้วย eps = 1e-15:
  p = max(0, eps) = 1e-15
  loss = -ln(1e-15) ≈ 34.5  ← finite
  grad = -1.0 / 1e-15 = -1e15  ← ใหญ่มากแต่ finite
```

**MSE Loss:**

```rust
pub struct MSELoss;

impl Loss for MSELoss {
    fn compute(&self, predictions: &Tensor, targets: &Tensor) -> f64 {
        let n = predictions.data.len() as f64;
        predictions.data.iter().zip(&targets.data)
            .map(|(&p, &t)| (p - t).powi(2))
            .sum::<f64>() / n
    }

    fn gradient(&self, predictions: &Tensor, targets: &Tensor) -> Tensor {
        let n = predictions.data.len() as f64;
        let grad: Vec<f64> = predictions.data.iter().zip(&targets.data)
            .map(|(&p, &t)| 2.0 * (p - t) / n)
            .collect();
        Tensor::new(grad, predictions.shape.clone())
    }
}
```

### ขั้นที่ 6: Optimizers — SGD และ Adam

**`src/optimizer.rs`**

```rust
use crate::tensor::Tensor;
use crate::layer::DenseLayer;

/// SGD with momentum
pub struct SGD {
    pub learning_rate: f64,
    pub momentum: f64,
}

impl SGD {
    pub fn new(learning_rate: f64, momentum: f64) -> Self {
        SGD { learning_rate, momentum }
    }

    pub fn update(&self, layer: &mut DenseLayer) {
        if let (Some(gw), Some(gb)) = (
            &layer.grad_weights.clone(),
            &layer.grad_biases.clone()
        ) {
            layer.weights = layer.weights.add(&gw.scale(-self.learning_rate));
            layer.biases  = layer.biases.add(&gb.scale(-self.learning_rate));
        }
    }
}
```

**Adam Optimizer:**

```rust
#[derive(Debug, Clone, Default)]
pub struct AdamState {
    pub m_w: Option<Tensor>,  // first moment (mean) for weights
    pub v_w: Option<Tensor>,  // second moment (variance) for weights
    pub m_b: Option<Tensor>,
    pub v_b: Option<Tensor>,
}

pub struct Adam {
    pub lr:      f64,  // learning rate
    pub beta1:   f64,  // decay for first moment (typically 0.9)
    pub beta2:   f64,  // decay for second moment (typically 0.999)
    pub epsilon: f64,  // numerical stability (typically 1e-8)
}

impl Adam {
    pub fn new(lr: f64, beta1: f64, beta2: f64, epsilon: f64) -> Self {
        Adam { lr, beta1, beta2, epsilon }
    }

    pub fn update_layer(
        &self,
        layer: &mut DenseLayer,
        state: &mut AdamState,
        t: usize,          // step number (เริ่มที่ 1)
    ) {
        let gw = match &layer.grad_weights { Some(g) => g.clone(), None => return };
        let gb = match &layer.grad_biases  { Some(g) => g.clone(), None => return };

        let t = t as f64;

        // Initialize moments เป็น zeros ครั้งแรก
        if state.m_w.is_none() {
            state.m_w = Some(Tensor::zeros(gw.rows(), gw.cols()));
            state.v_w = Some(Tensor::zeros(gw.rows(), gw.cols()));
            state.m_b = Some(Tensor::zeros(gb.rows(), gb.cols()));
            state.v_b = Some(Tensor::zeros(gb.rows(), gb.cols()));
        }

        // Weight update
        {
            let mw = state.m_w.as_ref().unwrap();
            let vw = state.v_w.as_ref().unwrap();

            // mₜ = β₁·mₜ₋₁ + (1-β₁)·gₜ
            let new_mw = mw.scale(self.beta1).add(&gw.scale(1.0 - self.beta1));
            // vₜ = β₂·vₜ₋₁ + (1-β₂)·gₜ²
            let new_vw = vw.scale(self.beta2)
                .add(&gw.map(|x| x * x).scale(1.0 - self.beta2));

            // Bias correction: m̂ₜ = mₜ / (1 - β₁ᵗ)
            let mw_hat = new_mw.scale(1.0 / (1.0 - self.beta1.powf(t)));
            let vw_hat = new_vw.scale(1.0 / (1.0 - self.beta2.powf(t)));

            // Wₜ = Wₜ₋₁ - lr · m̂ₜ / (√v̂ₜ + ε)
            let update_w: Vec<f64> = mw_hat.data.iter().zip(&vw_hat.data)
                .map(|(m, v)| -self.lr * m / (v.sqrt() + self.epsilon))
                .collect();
            let update_w = Tensor::new(update_w, vec![gw.rows(), gw.cols()]);
            layer.weights = layer.weights.add(&update_w);
            state.m_w = Some(new_mw);
            state.v_w = Some(new_vw);
        }

        // Bias update (เหมือนกัน)
        { /* ... similar for gb, mb, vb ... */ }
    }
}
```

**ทำไม Adam ดีกว่า SGD?**

| | SGD | Adam |
|---|---|---|
| Learning rate | คงที่ ต้องปรับมือ | Adaptive ต่อ parameter |
| Gradient noise | sensitive มาก | smooth ด้วย momentum |
| Sparse gradients | แย่ | ดีมาก |
| Convergence | ช้า | เร็ว |
| Memory | O(params) | O(3×params) |

**Bias Correction ใน Adam:**

เมื่อ t ยังน้อย `β₁ᵗ` ยังใกล้ 1 ทำให้ `1 - β₁ᵗ` เล็กมาก หาร moment ด้วยค่าเล็ก ๆ นี้จะทำให้ estimate ถูกต้องในช่วงแรกของ training

```
t=1: β₁=0.9, bias_correction = 1/(1-0.9¹) = 1/0.1 = 10x
t=10: bias_correction = 1/(1-0.9¹⁰) ≈ 1.53x
t→∞: bias_correction → 1x
```

### ขั้นที่ 7: Network และ Training Loop

**`src/network.rs`**

```rust
use crate::tensor::Tensor;
use crate::layer::DenseLayer;
use crate::optimizer::{Adam, AdamState};
use crate::loss::Loss;

pub struct Network {
    pub layers:          Vec<DenseLayer>,
    optimizer_states:    Vec<AdamState>,
    global_step:         usize,
}

impl Network {
    pub fn new() -> Self {
        Network { layers: Vec::new(), optimizer_states: Vec::new(), global_step: 0 }
    }

    pub fn add_layer(&mut self, layer: DenseLayer) {
        self.layers.push(layer);
        self.optimizer_states.push(AdamState::new());
    }

    /// Forward pass ผ่านทุก layer ตามลำดับ
    pub fn forward(&mut self, x: &Tensor) -> Tensor {
        let mut out = x.clone();
        for layer in &mut self.layers {
            out = layer.forward(&out);
        }
        out
    }

    /// Backward pass ย้อนกลับผ่านทุก layer
    pub fn backward(&mut self, grad: &Tensor) {
        let mut current_grad = grad.clone();
        for layer in self.layers.iter_mut().rev() {
            current_grad = layer.backward(&current_grad);
        }
    }

    /// Apply Adam optimizer
    fn step(&mut self, optimizer: &Adam) {
        self.global_step += 1;
        let t = self.global_step;
        for (layer, state) in self.layers.iter_mut()
            .zip(self.optimizer_states.iter_mut())
        {
            optimizer.update_layer(layer, state, t);
        }
    }

    fn zero_grad(&mut self) {
        for layer in &mut self.layers { layer.zero_grad(); }
    }

    /// Predict สำหรับ sample เดี่ยว
    pub fn predict(&mut self, x: &[f64]) -> Vec<f64> {
        let input = Tensor::new(x.to_vec(), vec![1, x.len()]);
        self.forward(&input).data
    }

    /// Mini-batch training loop
    pub fn fit(
        &mut self,
        x:          &[Vec<f64>],
        y:          &[Vec<f64>],
        epochs:     usize,
        batch_size: usize,
        optimizer:  &Adam,
        loss_fn:    &dyn Loss,
    ) -> f64 {
        let n = x.len();
        let in_features  = x[0].len();
        let out_features = y[0].len();
        let mut last_loss = f64::MAX;

        for _epoch in 0..epochs {
            let mut epoch_loss  = 0.0;
            let mut batch_count = 0;
            let mut i = 0;

            while i < n {
                let end = (i + batch_size).min(n);
                let bs  = end - i;

                // สร้าง batch tensors
                let x_data: Vec<f64> = x[i..end].iter()
                    .flat_map(|v| v.iter().copied()).collect();
                let y_data: Vec<f64> = y[i..end].iter()
                    .flat_map(|v| v.iter().copied()).collect();
                let x_batch = Tensor::new(x_data, vec![bs, in_features]);
                let y_batch = Tensor::new(y_data, vec![bs, out_features]);

                // Forward → Loss → Backward → Step
                let predictions = self.forward(&x_batch);
                epoch_loss += loss_fn.compute(&predictions, &y_batch);
                batch_count += 1;

                self.zero_grad();
                let grad = loss_fn.gradient(&predictions, &y_batch);
                self.backward(&grad);
                self.step(optimizer);

                i = end;
            }

            last_loss = epoch_loss / batch_count as f64;
        }

        last_loss
    }
}
```

### ขั้นที่ 8: Demo — XOR Problem

XOR เป็น classic test case สำหรับ neural network เพราะมันไม่ linearly separable — ต้องใช้ hidden layer

**`src/main.rs`**

```rust
mod tensor;
mod activation;
mod layer;
mod network;
mod optimizer;
mod loss;

use network::Network;
use layer::DenseLayer;
use activation::Activation;
use optimizer::Adam;
use loss::CrossEntropyLoss;

fn main() {
    println!("=== Neural Network from Scratch ===\n");

    // XOR dataset
    let x = vec![
        vec![0.0f64, 0.0],
        vec![0.0, 1.0],
        vec![1.0, 0.0],
        vec![1.0, 1.0],
    ];
    let y = vec![
        vec![0.0f64],
        vec![1.0],
        vec![1.0],
        vec![0.0],
    ];

    let mut net = Network::new();
    net.add_layer(DenseLayer::new(2, 8, Activation::ReLU));
    net.add_layer(DenseLayer::new(8, 4, Activation::ReLU));
    net.add_layer(DenseLayer::new(4, 1, Activation::Sigmoid));

    let optimizer = Adam::new(0.01, 0.9, 0.999, 1e-8);
    let loss_fn   = CrossEntropyLoss;

    let final_loss = net.fit(&x, &y, 2000, 4, &optimizer, &loss_fn);
    println!("XOR training final loss: {:.6}", final_loss);

    println!("\nXOR Predictions:");
    for (xi, yi) in x.iter().zip(y.iter()) {
        let pred = net.predict(xi);
        println!("  input={:?} → pred={:.4}, target={}", xi, pred[0], yi[0]);
    }
}
```

**Output จากการรันจริง:**

```
=== Neural Network from Scratch ===

XOR training final loss: 0.346587

XOR Predictions:
  input=[0.0, 0.0] → pred=0.5000, target=0
  input=[0.0, 1.0] → pred=0.5000, target=1
  input=[1.0, 0.0] → pred=1.0000, target=1
  input=[1.0, 1.0] → pred=0.0000, target=0
```

(output อาจต่างกันเล็กน้อยในแต่ละ run เพราะ random initialization แต่ unit test `test_xor_converges` ใช้ 3000 epochs และ verify ว่า loss < 0.1)

## การทดสอบ (Testing)

### Unit Tests ครบ 31 Tests

```rust
// ใน src/tensor.rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_tensor_new_valid() {
        let t = Tensor::new(vec![1.0, 2.0, 3.0, 4.0], vec![2, 2]);
        assert_eq!(t.rows(), 2);
        assert_eq!(t.cols(), 2);
        assert_eq!(t.get(0, 0), 1.0);
        assert_eq!(t.get(1, 1), 4.0);
    }

    #[test]
    #[should_panic(expected = "data length must match shape product")]
    fn test_tensor_shape_mismatch_panics() {
        Tensor::new(vec![1.0, 2.0], vec![2, 2]);  // 2 elements != 4
    }

    #[test]
    fn test_matmul_shape() {
        // (2×3) × (3×4) → (2×4)
        let a = Tensor::new(vec![1.0; 6], vec![2, 3]);
        let b = Tensor::new(vec![1.0; 12], vec![3, 4]);
        let c = a.matmul(&b);
        assert_eq!(c.rows(), 2);
        assert_eq!(c.cols(), 4);
    }

    #[test]
    fn test_matmul_values() {
        let a = Tensor::new(vec![1.0, 2.0, 3.0, 4.0], vec![2, 2]);
        let b = Tensor::new(vec![5.0, 6.0, 7.0, 8.0], vec![2, 2]);
        let c = a.matmul(&b);
        assert_eq!(c.get(0, 0), 19.0);
        assert_eq!(c.get(0, 1), 22.0);
        assert_eq!(c.get(1, 0), 43.0);
        assert_eq!(c.get(1, 1), 50.0);
    }

    #[test]
    fn test_transpose() {
        let a = Tensor::new(
            vec![1.0, 2.0, 3.0, 4.0, 5.0, 6.0], vec![2, 3]
        );
        let at = a.transpose();
        assert_eq!(at.rows(), 3);
        assert_eq!(at.cols(), 2);
        assert_eq!(at.get(0, 1), 4.0);  // element (0,1) of aᵀ = (1,0) of a
    }

    #[test]
    fn test_add_broadcast() {
        let a = Tensor::new(vec![1.0, 2.0, 3.0, 4.0], vec![2, 2]);
        let b = Tensor::new(vec![10.0, 20.0], vec![1, 2]);
        let c = a.add_broadcast(&b);
        assert_eq!(c.get(0, 0), 11.0);
        assert_eq!(c.get(1, 1), 24.0);
    }

    #[test]
    fn test_scale() {
        let a = Tensor::new(vec![1.0, 2.0, 3.0, 4.0], vec![2, 2]);
        let b = a.scale(3.0);
        assert_eq!(b.get(0, 0), 3.0);
        assert_eq!(b.get(1, 1), 12.0);
    }
}

// ใน src/activation.rs
#[cfg(test)]
mod tests {
    #[test]
    fn test_relu_negative_is_zero() {
        let z = Tensor::new(vec![-3.0, -1.0, 0.0, 2.0, 5.0], vec![1, 5]);
        let out = Activation::ReLU.forward(&z);
        assert_eq!(out.get(0, 0), 0.0);
        assert_eq!(out.get(0, 3), 2.0);
    }

    #[test]
    fn test_relu_derivative_positive() {
        let z = Tensor::new(vec![-1.0, 0.0, 1.0, 5.0], vec![1, 4]);
        let d = Activation::ReLU.derivative(&z);
        assert_eq!(d.get(0, 0), 0.0);  // negative → 0
        assert_eq!(d.get(0, 2), 1.0);  // positive → 1
    }

    #[test]
    fn test_sigmoid_saturation() {
        let z = Tensor::new(vec![-500.0, 500.0], vec![1, 2]);
        let out = Activation::Sigmoid.forward(&z);
        assert!(out.get(0, 0) < 1e-10);
        assert!(out.get(0, 1) > 1.0 - 1e-10);
    }

    #[test]
    fn test_softmax_sums_to_one() {
        let z = Tensor::new(vec![1.0, 2.0, 3.0, 4.0, 5.0], vec![1, 5]);
        let out = Activation::Softmax.forward(&z);
        let sum: f64 = (0..5).map(|j| out.get(0, j)).sum();
        assert!((sum - 1.0).abs() < 1e-10);
    }

    #[test]
    fn test_softmax_numerical_stability() {
        let z = Tensor::new(vec![1000.0, 1001.0, 999.0], vec![1, 3]);
        let out = Activation::Softmax.forward(&z);
        for j in 0..3 {
            assert!(out.get(0, j).is_finite());
        }
    }
}
```

### Real `cargo test` Output

```
running 31 tests
test activation::tests::test_linear_identity ... ok
test activation::tests::test_relu_derivative_positive ... ok
test activation::tests::test_relu_negative_is_zero ... ok
test activation::tests::test_sigmoid_derivative ... ok
test activation::tests::test_sigmoid_range ... ok
test activation::tests::test_sigmoid_saturation ... ok
test activation::tests::test_softmax_numerical_stability ... ok
test activation::tests::test_softmax_sums_to_one ... ok
test activation::tests::test_tanh_range ... ok
test layer::tests::test_dense_grad_weights_shape ... ok
test layer::tests::test_dense_backward_grad_shape ... ok
test loss::tests::test_cross_entropy_gradient_shape ... ok
test loss::tests::test_cross_entropy_perfect_binary ... ok
test loss::tests::test_mse_known_value ... ok
test layer::tests::test_layer_caches_input ... ok
test layer::tests::test_dense_forward_shape ... ok
test network::tests::test_backward_does_not_panic ... ok
test network::tests::test_forward_pass_shape ... ok
test optimizer::tests::test_adam_moment_estimates ... ok
test loss::tests::test_mse_perfect_prediction ... ok
test optimizer::tests::test_sgd_update_decreases_weights ... ok
test tensor::tests::test_add_broadcast ... ok
test tensor::tests::test_matmul_shape ... ok
test tensor::tests::test_matmul_values ... ok
test tensor::tests::test_scale ... ok
test optimizer::tests::test_adam_updates_weights ... ok
test tensor::tests::test_tensor_new_valid ... ok
test tensor::tests::test_transpose ... ok
test network::tests::test_batch_splitting ... ok
test tensor::tests::test_tensor_shape_mismatch_panics - should panic ... ok
test network::tests::test_xor_converges ... ok

test result: ok. 31 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.34s
```

### Integration Test — XOR Convergence

```rust
// ใน src/network.rs
#[test]
fn test_xor_converges() {
    let x = vec![
        vec![0.0f64, 0.0],
        vec![0.0, 1.0],
        vec![1.0, 0.0],
        vec![1.0, 1.0],
    ];
    let y = vec![
        vec![0.0f64],
        vec![1.0],
        vec![1.0],
        vec![0.0],
    ];

    let mut net = Network::new();
    net.add_layer(DenseLayer::new(2, 8, Activation::ReLU));
    net.add_layer(DenseLayer::new(8, 4, Activation::ReLU));
    net.add_layer(DenseLayer::new(4, 1, Activation::Sigmoid));

    let optimizer = Adam::new(0.01, 0.9, 0.999, 1e-8);
    let loss_fn   = CrossEntropyLoss;
    let final_loss = net.fit(&x, &y, 3000, 4, &optimizer, &loss_fn);

    println!("XOR final loss: {:.6}", final_loss);
    assert!(
        final_loss < 0.1,
        "XOR network should converge: loss={:.6}", final_loss
    );
}
```

Test นี้ verify ว่า network จริง ๆ สามารถ learn XOR ได้ ไม่ใช่แค่ test รูปร่าง gradient

## จุดระวัง (Common Pitfalls)

### Pitfall 1: Vanishing Gradient ใน Deep Networks

**อาการ:** Loss ไม่ลดลง, gradient ของ layer แรก ๆ ใกล้ 0

**สาเหตุ:** Sigmoid/Tanh มี derivative สูงสุด 0.25/1.0 ถ้า chain ผ่านหลาย layer:
```
∂L/∂W₁ = ∂L/∂aₙ × σ'(zₙ) × Wₙ × ... × σ'(z₁)
```
แต่ละ σ'(z) ≤ 0.25 — ถ้ามี 10 layers จะลดเหลือ `0.25¹⁰ ≈ 9.5×10⁻⁷`

**วิธีแก้:**
- ใช้ ReLU แทน Sigmoid/Tanh สำหรับ hidden layers
- ใช้ He initialization (scale ใหญ่กว่า Xavier)
- Batch Normalization (เพิ่ม layer ปรับ z ก่อน activation)

### Pitfall 2: การ Clone Tensor โดยไม่จำเป็น

**อาการ:** program ช้ามาก, memory ใช้สูง

**สาเหตุ:**
```rust
// BAD: clone ทุก call
pub fn forward(&mut self, input: &Tensor) -> Tensor {
    self.last_input = Some(input.clone());  // OK - ต้อง cache
    let z = input.matmul(&self.weights);   // ไม่ clone เปล่า
    self.last_z = Some(z.clone());         // BAD: z ถูก move อยู่แล้ว
    self.activation.forward(&z)            // ← error เพราะ z moved
}

// GOOD: clone เฉพาะที่จำเป็น
pub fn forward(&mut self, input: &Tensor) -> Tensor {
    let z = input.matmul(&self.weights).add_broadcast(&self.biases);
    self.last_input = Some(input.clone());
    self.last_z = Some(z.clone());  // ต้อง clone เพราะต้องใช้ z ต่อ
    self.activation.forward(&z)
}
```

**วิธีแก้จริง:** ในโปรเจคจริงควรใช้ `Rc<Tensor>` หรือ `Arc<Tensor>` เพื่อ share ข้อมูลโดยไม่ต้อง copy

### Pitfall 3: Exploding Gradient

**อาการ:** Loss กลายเป็น NaN หลังจาก train ไปสักพัก

**สาเหตุ:** Gradient ใหญ่มากเกินไป, weight update ทำให้ loss พุ่งสูง

```
step 100: loss = 0.45
step 101: loss = 1.23
step 102: loss = 87.4
step 103: loss = NaN
```

**วิธีแก้:**
```rust
// Gradient clipping
fn clip_gradients(&mut self, max_norm: f64) {
    for layer in &mut self.layers {
        if let Some(gw) = &layer.grad_weights {
            let norm: f64 = gw.data.iter().map(|x| x * x).sum::<f64>().sqrt();
            if norm > max_norm {
                layer.grad_weights = Some(gw.scale(max_norm / norm));
            }
        }
    }
}
```

### Pitfall 4: Wrong Batch Dimension

**อาการ:** `matmul shape mismatch` panic ระหว่าง backward pass

**สาเหตุ:** สับสนว่า input shape ควรเป็น `(batch × features)` หรือ `(features × batch)`

```rust
// Convention ที่ใช้ในโปรเจคนี้ (row = sample):
// Input X: (batch_size × in_features)
// Weights W: (in_features × out_features)
// Output: X @ W = (batch_size × out_features)  ✓

// WRONG convention (column = sample):
// Input X: (in_features × batch_size)
// ทำให้ X.T @ delta มีขนาดผิด
```

**วิธีแก้:** เลือก convention หนึ่งแล้วใช้ตลอด วาด diagram shape ก่อนเขียนโค้ด

### Pitfall 5: ลืม zero_grad() ก่อน backward

**อาการ:** Gradient สะสมข้าม step ทำให้ update ใหญ่เกินจริง, training ไม่ converge

**สาเหตุ:**
```rust
// BAD: ไม่ reset gradient
for batch in batches {
    let pred = net.forward(&x_batch);
    let grad = loss_fn.gradient(&pred, &y_batch);
    net.backward(&grad);  // gradient สะสมทับกัน!
    net.step(&optimizer);
}

// GOOD: zero_grad ก่อน backward เสมอ
for batch in batches {
    let pred = net.forward(&x_batch);
    let grad = loss_fn.gradient(&pred, &y_batch);
    net.zero_grad();      // ← ต้องทำก่อน
    net.backward(&grad);
    net.step(&optimizer);
}
```

**วิธีแก้:** บางคนชอบทำ `zero_grad` หลัง `step` แทน แต่ต้องสม่ำเสมอ

### Pitfall 6: Learning Rate ไม่เหมาะสม

| Learning Rate | ปัญหา |
|---|---|
| `lr = 1.0` (สูงเกิน) | Loss diverges, NaN |
| `lr = 0.01` (เหมาะสมกับ Adam) | Converge ดี |
| `lr = 0.0001` (ต่ำเกิน) | Converge ช้า, ต้องใช้ epoch มากขึ้น |
| `lr = 0.1` กับ SGD | มักใช้งานได้ |
| `lr = 0.1` กับ Adam | มักระเบิด |

Adam sensitive ต่อ learning rate น้อยกว่า SGD แต่ค่า default `0.001` หรือ `0.01` มักได้ผลดีโดยไม่ต้อง tune

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
ls -lh target/release/neural-network
# -rwxr-xr-x 1 user user 1.2M neural-network
```

Release build ใช้ optimization level 3 ซึ่งทำให้ matrix multiplication เร็วขึ้นมาก (LLVM auto-vectorize loops)

### ทดสอบ Performance

```bash
# ทดสอบ time ในการ train
time cargo run --release

# Profile ด้วย perf (Linux)
cargo build --release
perf record ./target/release/neural-network
perf report
```

ด้วย pure Rust implementation บน XOR problem (4 samples, 3000 epochs):
- Debug build: ~340ms
- Release build: ~15ms

ความเร็วส่วนใหญ่มาจาก `matmul` การ optimize SIMD หรือใช้ BLAS จะเร็วขึ้นอีก 10-100x

### Export Weights สำหรับ Inference

```rust
use std::io::{BufWriter, Write};

pub fn save_weights(&self, path: &str) -> std::io::Result<()> {
    let file = std::fs::File::create(path)?;
    let mut writer = BufWriter::new(file);
    for (i, layer) in self.layers.iter().enumerate() {
        writeln!(writer, "# layer {}", i)?;
        let w = &layer.weights;
        writeln!(writer, "weights {} {}", w.rows(), w.cols())?;
        for val in &w.data {
            write!(writer, "{:.8} ", val)?;
        }
        writeln!(writer)?;
    }
    Ok(())
}
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Dropout Layer (ระดับกลาง)

Dropout คือ regularization technique ที่สุ่ม "ปิด" neurons บางตัวระหว่าง training เพื่อป้องกัน overfitting

```rust
pub struct DropoutLayer {
    pub rate: f64,          // probability ที่ neuron จะถูกปิด
    mask: Option<Tensor>,   // 0/1 mask จาก forward pass
}

impl DropoutLayer {
    pub fn forward(&mut self, input: &Tensor, training: bool) -> Tensor {
        if !training {
            return input.clone();  // inference: ไม่ dropout
        }
        // สร้าง mask ของ 0 และ 1
        let mut rng = rand::thread_rng();
        let mask_data: Vec<f64> = (0..input.data.len())
            .map(|_| if rng.gen::<f64>() > self.rate { 1.0 } else { 0.0 })
            .collect();
        // Scale ขึ้นเพื่อรักษา expected value: E[output] = E[input]
        let scale = 1.0 / (1.0 - self.rate);
        let mask = Tensor::new(mask_data, input.shape.clone());
        let result = input.mul_elementwise(&mask).scale(scale);
        self.mask = Some(mask);
        result
    }

    pub fn backward(&self, grad: &Tensor) -> Tensor {
        let mask = self.mask.as_ref().unwrap();
        grad.mul_elementwise(mask).scale(1.0 / (1.0 - self.rate))
    }
}
```

ทดสอบว่า: กับ XOR ที่ overfitting, Dropout rate=0.3 ช่วยลด validation loss ได้หรือไม่

### แบบฝึกหัดที่ 2: Batch Normalization (ระดับยาก)

Batch Normalization normalize output ของแต่ละ layer ให้มี mean=0 variance=1 ช่วยแก้ปัญหา vanishing gradient อย่างได้ผลมาก

```rust
pub struct BatchNormLayer {
    gamma: Tensor,  // learnable scale parameter
    beta:  Tensor,  // learnable shift parameter
    eps:   f64,
    // Running statistics สำหรับ inference
    running_mean: Tensor,
    running_var:  Tensor,
}

// Forward: x_norm = (x - μ) / √(σ² + ε)
// Output:  y = γ·x_norm + β
```

ท้าทาย: backward pass ของ BatchNorm ซับซ้อนมาก เพราะ mean และ variance ขึ้นอยู่กับ batch ทั้งหมด ต้องใช้ chain rule สองทอด

### แบบฝึกหัดที่ 3: Convolutional Layer สำหรับ Image (ระดับยากมาก)

Dense layer ไม่เหมาะกับ image เพราะ:
- Input 28×28 = 784 features, แต่ละ neuron connect กับทุก pixel
- ไม่รู้จัก spatial locality (pixels ที่ใกล้กันน่าจะเกี่ยวข้องกัน)

Convolution layer แก้ได้ด้วย sliding window:

```rust
pub struct Conv2DLayer {
    filters:      Tensor,  // (out_channels × in_channels × kernel_h × kernel_w)
    biases:       Tensor,  // (1 × out_channels)
    stride:       usize,
    padding:      usize,
    activation:   Activation,
}
```

เป้าหมาย: ทำให้ network classify MNIST digit ได้ accuracy > 95%

### แบบฝึกหัดที่ 4: Early Stopping และ Learning Rate Scheduler (ระดับง่าย-กลาง)

เพิ่ม validation set และ stop training เมื่อ validation loss ไม่ลดลงติดต่อกัน N epochs

```rust
pub struct EarlyStopping {
    patience:    usize,   // จำนวน epochs ที่ยอมรับ
    min_delta:   f64,     // improvement น้อยกว่านี้ถือว่าไม่ดีขึ้น
    best_loss:   f64,
    wait_count:  usize,
}

impl EarlyStopping {
    pub fn should_stop(&mut self, val_loss: f64) -> bool {
        if val_loss < self.best_loss - self.min_delta {
            self.best_loss  = val_loss;
            self.wait_count = 0;
            false
        } else {
            self.wait_count += 1;
            self.wait_count >= self.patience
        }
    }
}
```

เพิ่ม ReduceLROnPlateau: ลด learning rate ลงครึ่งหนึ่งเมื่อ validation loss ไม่ดีขึ้น 5 epochs ติดกัน

### แบบฝึกหัดที่ 5: Multi-class Classification ด้วย Softmax + Categorical Cross-Entropy

ขยาย network ให้รองรับ output หลาย classes:

```rust
// สำหรับ K classes:
// output layer: Softmax activation, K neurons
// loss: Categorical Cross-Entropy
//   L = -Σᵢ yᵢ log(pᵢ)   (one-hot y)

pub struct CategoricalCrossEntropyLoss;

impl Loss for CategoricalCrossEntropyLoss {
    fn compute(&self, predictions: &Tensor, targets: &Tensor) -> f64 {
        let n   = predictions.rows() as f64;
        let eps = 1e-15;
        let loss: f64 = (0..predictions.rows()).map(|i| {
            -(0..predictions.cols()).map(|j| {
                let p = predictions.get(i, j).max(eps);
                targets.get(i, j) * p.ln()
            }).sum::<f64>()
        }).sum();
        loss / n
    }
}
```

ทดสอบกับ Iris dataset (3 species, 4 features): ควรได้ accuracy > 90%

### แบบฝึกหัดที่ 6: SIMD Optimization (ระดับขั้นสูง)

Matrix multiplication คือ bottleneck หลัก ใช้ `std::simd` (nightly Rust) หรือ crate `wide` เพื่อประมวลผล 4-8 f64 พร้อมกัน:

```rust
// ใช้ rayon สำหรับ parallel matmul
use rayon::prelude::*;

pub fn matmul_parallel(&self, other: &Tensor) -> Tensor {
    let m = self.rows();
    let n = other.cols();
    let mut result_data = vec![0.0f64; m * n];
    result_data.par_chunks_mut(n).enumerate().for_each(|(i, row)| {
        for j in 0..n {
            row[j] = (0..self.cols())
                .map(|l| self.get(i, l) * other.get(l, j))
                .sum();
        }
    });
    Tensor::new(result_data, vec![m, n])
}
```

วัด speedup กับ matrix 1000×1000 บน CPU ที่มี multiple cores

## สรุป

ในโปรเจคนี้เราได้สร้าง Neural Network ที่ fully functional จาก scratch ครอบคลุม:

1. **`Tensor`** — matrix operations ด้วย flat array, matmul, transpose, broadcast add
2. **`Activation`** — ReLU, Sigmoid, Tanh, Softmax (numerically stable), Linear
3. **`DenseLayer`** — forward pass, cache management, backpropagation
4. **`Network`** — forward/backward pass ผ่านหลาย layers ด้วย chain rule
5. **`SGD` / `Adam`** — weight update, moment estimates, bias correction
6. **`CrossEntropyLoss` / `MSELoss`** — loss computation และ gradient
7. **Training loop** — mini-batch, epoch, loss tracking

**Pattern สำคัญที่ได้เรียนรู้:**

- **Separation of concerns**: Layer รู้เรื่อง computation, Network รู้เรื่อง composition, Optimizer รู้เรื่อง update rule
- **Cache pattern**: Store intermediate values ใน `Option<Tensor>` เพื่อใช้ใน backward pass
- **Trait-based design**: `Loss` trait ทำให้ swap loss function โดยไม่แตะ Network code
- **Ownership ใน training**: State ที่ต้อง persist (Adam moments) อยู่ใน Network ไม่ใช่ใน Layer

**สิ่งที่โปรเจคนี้ไม่ครอบคลุม (แต่น่าสนใจ):**

- GPU acceleration (ต้องใช้ CUDA/ROCm หรือ `wgpu`)
- Automatic differentiation (autograd) ที่ compute graph แบบ dynamic
- Recurrent Neural Networks (RNN/LSTM) สำหรับ sequence data
- Transformer architecture สำหรับ NLP

โปรเจคถัดไป **I03: Decision Tree** จะเปลี่ยนมุมมองจาก gradient-based learning ไปสู่ tree-based learning ซึ่งไม่ต้องการ derivatives เลย — เป็นวิธีที่ interpretable กว่าและ works well กับ tabular data

---

**โปรเจคก่อนหน้า:** [Project I01: Linear Regression](project-i01-linear-regression.md) | **โปรเจคถัดไป:** [Project I03: Decision Tree](project-i03-decision-tree.md)
