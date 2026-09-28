# Project I01: Linear Regression from Scratch

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **ระบบ Machine Learning สำหรับ Regression และ Classification** ตั้งแต่ต้นโดยไม่ใช้ไลบรารี ML ใด ๆ เลย — เขียนทุกอย่างเป็น Rust순수 (pure Rust math) ตั้งแต่ matrix operations ไปจนถึง gradient descent และ logistic regression พร้อม model evaluation framework ครบถ้วน

เป้าหมายหลักไม่ใช่แค่ "รันได้" แต่คือการ **เข้าใจคณิตศาสตร์เบื้องหลัง** ว่าทำไม OLS (Ordinary Least Squares) จึงหา coefficients ที่ดีที่สุด, gradient descent ทำงานอย่างไร, และ logistic regression แตกต่างจาก linear regression อย่างไรในเชิงคณิตศาสตร์ เมื่อเขียนสูตรเหล่านี้เองด้วย Rust คุณจะเข้าใจได้ลึกกว่าการใช้ไลบรารีสำเร็จรูป

**Use cases จริงในโลก production:**
- **Pricing models** — ทำนายราคาบ้าน, ราคาสินค้า ด้วย multiple linear regression
- **Anomaly detection** — ตรวจจับ outliers ในข้อมูล sensor โดยใช้ residuals
- **Binary classification** — spam detection, fraud detection ด้วย logistic regression
- **Feature importance** — วิเคราะห์ว่า feature ไหนมีอิทธิพลต่อ output มากที่สุด
- **Embedded ML** — deploy โมเดลที่ต้องการ compute ต่ำบน embedded systems หรือ edge devices
- **Model serialization** — save/load โมเดลด้วย `serde_json` สำหรับ production API

## สิ่งที่จะได้เรียนรู้

- **Matrix algebra in Rust** — implement Matrix struct ที่มี multiply, transpose, Gauss-Jordan inverse
- **OLS closed-form** — เข้าใจสูตร β = (XᵀX)⁻¹Xᵀy และทำไมมันหาคำตอบที่ดีที่สุด
- **Gradient descent variants** — batch, convergence detection, loss history tracking
- **Regularization** — Ridge (L2) ช่วยแก้ปัญหา multicollinearity อย่างไร
- **Logistic regression** — sigmoid function, binary cross-entropy, gradient update
- **Model evaluation** — R², RMSE, MAE, precision, recall, F1, ROC curve, AUC
- **Train/test split & k-fold CV** — วิธีประเมินโมเดลอย่างถูกต้องโดยไม่ overfit
- **Model serialization** — serialize/deserialize โมเดลด้วย `serde` และ `serde_json`

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`), iterators, closures
- **Part 31–40**: Error handling (`Result`, `Option`, `?`), traits, generics
- **Part 41–50**: File I/O, environment, basic numeric operations
- **Part 51–60**: String manipulation, formatting
- **Part 96–100**: พื้นฐาน linear algebra (matrix multiply, transpose) เป็นประโยชน์มาก

## โครงสร้างโปรเจค (Project Layout)

```
linear-regression/
├── src/
│   ├── lib.rs           ← core library (matrix, regression, evaluation)
│   └── main.rs          ← demo program แสดงผลการใช้งาน
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Input Data (Vec<f64> / Vec<Vec<f64>>)
        │
        ▼
┌──────────────┐   Matrix ops    ┌─────────────────────┐
│  Matrix ops  │ ──────────────▶ │  OLS / Ridge fit    │
│  (multiply,  │                 │  β = (XᵀX+λI)⁻¹Xᵀy │
│  transpose,  │                 └─────────────────────┘
│  inverse)    │
└──────────────┘   Gradient      ┌──────────────────────┐
        │          descent   ──▶ │ GradientDescent::fit │
        │                        │  ∂L/∂w = 2Xᵀ(Xw-y)  │
        │                        └──────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────┐
│             Model Evaluation                         │
│  R², RMSE, MAE  │  Precision/Recall/F1  │  ROC/AUC  │
└──────────────────────────────────────────────────────┘
```

### Design Decisions

**ทำไมถึงไม่ใช้ `ndarray` หรือ `nalgebra`?**

ไลบรารีเช่น `ndarray` หรือ `nalgebra` ดีมากสำหรับ production แต่จุดประสงค์ของโปรเจคนี้คือเรียนรู้คณิตศาสตร์ การเขียน Matrix จากศูนย์ทำให้เข้าใจว่า row-major storage ทำงานอย่างไร, Gauss-Jordan elimination ทำงานอย่างไร, และทำไมการ check pivot ถึงสำคัญ

**ทำไมถึงใช้ `Vec<f64>` แทน arrays?**

`Vec<f64>` รองรับ matrix ขนาดต่าง ๆ ที่ runtime ต่างจาก `[f64; N]` ที่ต้องรู้ขนาดตอน compile time ซึ่ง flexible กว่าสำหรับ ML workloads

**Closed-form OLS vs Gradient Descent**

สำหรับ n ขนาดเล็กถึงกลาง (< 10,000 samples, < 1,000 features), OLS closed-form เร็วและ exact กว่า แต่ต้อง invert matrix ขนาด p×p ซึ่ง O(p³) — ถ้า p ใหญ่มาก gradient descent ประหยัดหน่วยความจำกว่ามาก

**Gauss-Jordan vs LU Decomposition**

Gauss-Jordan เข้าใจง่ายและเหมาะกับ matrix เล็ก (p < 100) แต่สำหรับ production จริง ควรใช้ LU decomposition หรือ Cholesky factorization ซึ่ง numerically stable กว่า

### โครงสร้างของ `Matrix`

```
Matrix {
    data: Vec<f64>,   // row-major storage
    rows: usize,
    cols: usize,
}

element (i, j) อยู่ที่ data[i * cols + j]
```

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Matrix Operations — รากฐานของทุกอย่าง

ก่อนที่จะทำ regression ใด ๆ ได้ เราต้องมี linear algebra primitives ก่อน เริ่มจากสร้าง `Matrix` struct ที่รองรับ element access, multiply, transpose, และ Gauss-Jordan inverse

**`Cargo.toml`:**

```toml
[package]
name = "linear-regression"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**`src/lib.rs` — Part 1: Matrix struct:**

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Matrix {
    pub data: Vec<f64>,
    pub rows: usize,
    pub cols: usize,
}

impl Matrix {
    pub fn new(rows: usize, cols: usize) -> Self {
        Matrix {
            data: vec![0.0; rows * cols],
            rows,
            cols,
        }
    }

    pub fn from_vec(rows: usize, cols: usize, data: Vec<f64>) -> Self {
        assert_eq!(data.len(), rows * cols);
        Matrix { data, rows, cols }
    }

    pub fn get(&self, row: usize, col: usize) -> f64 {
        self.data[row * self.cols + col]
    }

    pub fn set(&mut self, row: usize, col: usize, val: f64) {
        self.data[row * self.cols + col] = val;
    }

    pub fn identity(n: usize) -> Matrix {
        let mut m = Matrix::new(n, n);
        for i in 0..n {
            m.set(i, i, 1.0);
        }
        m
    }
}
```

**Transpose:**

```rust
impl Matrix {
    pub fn transpose(&self) -> Matrix {
        let mut result = Matrix::new(self.cols, self.rows);
        for i in 0..self.rows {
            for j in 0..self.cols {
                result.set(j, i, self.get(i, j));
            }
        }
        result
    }
}
```

**Matrix Multiply (naive O(n³)):**

```rust
impl Matrix {
    pub fn multiply(&self, other: &Matrix) -> Matrix {
        assert_eq!(self.cols, other.rows, "Matrix dimension mismatch");
        let mut result = Matrix::new(self.rows, other.cols);
        for i in 0..self.rows {
            for j in 0..other.cols {
                let mut sum = 0.0;
                for k in 0..self.cols {
                    sum += self.get(i, k) * other.get(k, j);
                }
                result.set(i, j, sum);
            }
        }
        result
    }
}
```

**Gauss-Jordan Inverse:**

สิ่งที่ต้องทำคือสร้าง augmented matrix `[A | I]` แล้วทำ row operations จนได้ `[I | A⁻¹]`

```rust
impl Matrix {
    pub fn inverse(&self) -> Option<Matrix> {
        assert_eq!(self.rows, self.cols, "Only square matrices can be inverted");
        let n = self.rows;
        // สร้าง augmented matrix [A | I] ขนาด n × 2n
        let mut aug = vec![0.0f64; n * n * 2];
        for i in 0..n {
            for j in 0..n {
                aug[i * 2 * n + j] = self.get(i, j);
            }
            aug[i * 2 * n + n + i] = 1.0; // identity
        }

        for col in 0..n {
            // หา pivot ที่ใหญ่สุด (partial pivoting)
            let mut pivot_row = None;
            let mut max_val = 0.0_f64;
            for row in col..n {
                let val = aug[row * 2 * n + col].abs();
                if val > max_val {
                    max_val = val;
                    pivot_row = Some(row);
                }
            }
            let pivot_row = pivot_row?;
            if aug[pivot_row * 2 * n + col].abs() < 1e-12 {
                return None; // Singular matrix
            }
            // สลับแถว
            if pivot_row != col {
                for j in 0..2 * n {
                    aug.swap(col * 2 * n + j, pivot_row * 2 * n + j);
                }
            }
            // Scale แถว pivot ให้ diagonal เป็น 1
            let scale = aug[col * 2 * n + col];
            for j in 0..2 * n {
                aug[col * 2 * n + j] /= scale;
            }
            // Eliminate column ทุกแถว
            for row in 0..n {
                if row != col {
                    let factor = aug[row * 2 * n + col];
                    for j in 0..2 * n {
                        let tmp = aug[col * 2 * n + j] * factor;
                        aug[row * 2 * n + j] -= tmp;
                    }
                }
            }
        }

        // Extract ส่วนขวา = A⁻¹
        let mut result = Matrix::new(n, n);
        for i in 0..n {
            for j in 0..n {
                result.set(i, j, aug[i * 2 * n + n + j]);
            }
        }
        Some(result)
    }
}
```

**dot product:**

```rust
pub fn dot_product(a: &[f64], b: &[f64]) -> f64 {
    assert_eq!(a.len(), b.len());
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}
```

**ทดสอบขั้นที่ 1:**

```bash
$ cargo test tests::test_matrix
```

```
test tests::test_matrix_multiply_2x2 ... ok
test tests::test_matrix_multiply_2x3_3x2 ... ok
test tests::test_matrix_transpose ... ok
test tests::test_matrix_inverse_2x2 ... ok
test tests::test_matrix_inverse_identity ... ok
test tests::test_matrix_singular_returns_none ... ok
test tests::test_dot_product ... ok
```

---

### ขั้นที่ 2: Simple Linear Regression — OLS สูตรปิด

Simple Linear Regression กับตัวแปรเดียวสามารถหา slope และ intercept ได้ด้วยสูตร closed-form โดยตรง ไม่ต้องใช้ matrix:

```
β₁ = Σ(xᵢ - x̄)(yᵢ - ȳ) / Σ(xᵢ - x̄)²
β₀ = ȳ - β₁x̄
```

```rust
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct SimpleLinearRegression {
    pub slope: f64,
    pub intercept: f64,
}

impl SimpleLinearRegression {
    pub fn fit(x: &[f64], y: &[f64]) -> Self {
        assert_eq!(x.len(), y.len());
        let n = x.len() as f64;
        let x_mean = x.iter().sum::<f64>() / n;
        let y_mean = y.iter().sum::<f64>() / n;
        let numerator: f64 = x.iter().zip(y.iter())
            .map(|(xi, yi)| (xi - x_mean) * (yi - y_mean))
            .sum();
        let denominator: f64 = x.iter()
            .map(|xi| (xi - x_mean).powi(2))
            .sum();
        let slope = if denominator.abs() < 1e-12 { 0.0 } else { numerator / denominator };
        let intercept = y_mean - slope * x_mean;
        SimpleLinearRegression { slope, intercept }
    }

    pub fn predict(&self, x: f64) -> f64 {
        self.slope * x + self.intercept
    }

    pub fn residuals(&self, x: &[f64], y: &[f64]) -> Vec<f64> {
        x.iter().zip(y.iter())
            .map(|(xi, yi)| yi - self.predict(*xi))
            .collect()
    }

    pub fn r_squared(&self, x: &[f64], y: &[f64]) -> f64 {
        let y_mean = y.iter().sum::<f64>() / y.len() as f64;
        let ss_res: f64 = self.residuals(x, y).iter().map(|r| r * r).sum();
        let ss_tot: f64 = y.iter().map(|yi| (yi - y_mean).powi(2)).sum();
        if ss_tot < 1e-12 { return 1.0; }
        1.0 - ss_res / ss_tot
    }

    pub fn rmse(&self, x: &[f64], y: &[f64]) -> f64 {
        let res = self.residuals(x, y);
        let mse: f64 = res.iter().map(|r| r * r).sum::<f64>() / res.len() as f64;
        mse.sqrt()
    }

    pub fn mae(&self, x: &[f64], y: &[f64]) -> f64 {
        let res = self.residuals(x, y);
        res.iter().map(|r| r.abs()).sum::<f64>() / res.len() as f64
    }
}
```

**ทดสอบ Simple Linear Regression:**

```rust
#[test]
fn test_simple_lr_fit_perfect_line() {
    // y = 2x + 1
    let x = vec![1.0, 2.0, 3.0, 4.0, 5.0];
    let y: Vec<f64> = x.iter().map(|xi| 2.0 * xi + 1.0).collect();
    let model = SimpleLinearRegression::fit(&x, &y);
    assert!((model.slope - 2.0).abs() < 1e-9);
    assert!((model.intercept - 1.0).abs() < 1e-9);
}

#[test]
fn test_simple_lr_r_squared_perfect() {
    let x = vec![1.0, 2.0, 3.0, 4.0, 5.0];
    let y: Vec<f64> = x.iter().map(|xi| 3.0 * xi - 2.0).collect();
    let model = SimpleLinearRegression::fit(&x, &y);
    let r2 = model.r_squared(&x, &y);
    assert!((r2 - 1.0).abs() < 1e-9);
}
```

**ทำความเข้าใจ R²:**

- R² = 1.0 หมายความว่าโมเดลอธิบาย variance ได้ 100% (fit สมบูรณ์แบบ)
- R² = 0.0 หมายความว่าโมเดลไม่ดีกว่า predicting ค่าเฉลี่ยเลย
- R² < 0 หมายความว่าโมเดลแย่กว่า baseline!

---

### ขั้นที่ 3: OLS ด้วย Matrix Form — สำหรับ Multiple Features

สำหรับ multiple linear regression เราต้องใช้ matrix form:

```
β = (XᵀX)⁻¹Xᵀy
```

โดย X คือ feature matrix ขนาด n×(p+1) ที่มี bias column (column แรกทั้งหมดเป็น 1) เพิ่มเข้ามา

```rust
pub fn ols_matrix(x_matrix: &Matrix, y: &[f64]) -> Vec<f64> {
    let xt = x_matrix.transpose();               // (p+1) × n
    let xtx = xt.multiply(x_matrix);             // (p+1) × (p+1)
    let xty: Vec<f64> = (0..xt.rows).map(|i| {
        (0..xt.cols).map(|j| xt.get(i, j) * y[j]).sum::<f64>()
    }).collect();                                 // (p+1) × 1
    let xtx_inv = xtx.inverse().expect("XtX is singular");
    (0..xtx_inv.rows).map(|i| {
        (0..xtx_inv.cols).map(|j| xtx_inv.get(i, j) * xty[j]).sum::<f64>()
    }).collect()
}
```

**`LinearRegressor` สำหรับ Multiple Regression:**

```rust
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct LinearRegressor {
    pub coefficients: Vec<f64>,
    pub intercept: f64,
}

impl LinearRegressor {
    pub fn fit(x_data: &[Vec<f64>], y: &[f64]) -> Self {
        let n = x_data.len();
        let p = x_data[0].len();
        // เพิ่ม bias column (column ของ 1s) ที่ตำแหน่งแรก
        let mut x_aug = Matrix::new(n, p + 1);
        for i in 0..n {
            x_aug.set(i, 0, 1.0);       // bias
            for j in 0..p {
                x_aug.set(i, j + 1, x_data[i][j]);
            }
        }
        let beta = ols_matrix(&x_aug, y);
        LinearRegressor {
            intercept: beta[0],
            coefficients: beta[1..].to_vec(),
        }
    }

    pub fn predict(&self, x: &[f64]) -> f64 {
        self.intercept + dot_product(&self.coefficients, x)
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
// y = 1 + 2*x1 + 3*x2
let x_data = vec![
    vec![1.0, 1.0],
    vec![2.0, 1.0],
    vec![1.0, 2.0],
    vec![3.0, 2.0],
    vec![2.0, 3.0],
    vec![4.0, 1.0],
];
let y: Vec<f64> = x_data.iter()
    .map(|x| 1.0 + 2.0 * x[0] + 3.0 * x[1])
    .collect();

let model = LinearRegressor::fit(&x_data, &y);
println!("intercept = {:.4}", model.intercept);    // 1.0000
println!("coef[0]   = {:.4}", model.coefficients[0]); // 2.0000
println!("coef[1]   = {:.4}", model.coefficients[1]); // 3.0000
```

---

### ขั้นที่ 4: Gradient Descent — อีกทางเลือกในการ fit โมเดล

Gradient descent เป็น iterative algorithm ที่เหมาะกับ dataset ขนาดใหญ่ที่ matrix inversion แพงเกินไป แนวคิดคือค่อย ๆ ปรับ weights ไปในทิศทางที่ลด loss ลงทีละนิด

```
w ← w - α · (2/n) · Xᵀ(Xw - y)
```

โดย α คือ learning rate

```rust
#[derive(Debug, Clone)]
pub struct GradientDescent {
    pub learning_rate: f64,
    pub max_iter: usize,
    pub tolerance: f64,
}

#[derive(Debug, Clone)]
pub struct LossHistory {
    pub losses: Vec<f64>,
}

impl GradientDescent {
    pub fn new(learning_rate: f64, max_iter: usize, tolerance: f64) -> Self {
        GradientDescent { learning_rate, max_iter, tolerance }
    }

    pub fn fit_linear(&self, x: &Matrix, y: &[f64]) -> (Vec<f64>, LossHistory) {
        let n = x.rows;
        let p = x.cols;
        let mut weights = vec![0.0f64; p];
        let mut history = LossHistory { losses: Vec::new() };

        for _ in 0..self.max_iter {
            // คำนวณ predictions: ŷ = Xw
            let preds: Vec<f64> = (0..n).map(|i| {
                (0..p).map(|j| x.get(i, j) * weights[j]).sum::<f64>()
            }).collect();

            // MSE Loss = (1/n) Σ(ŷᵢ - yᵢ)²
            let loss: f64 = preds.iter().zip(y.iter())
                .map(|(p, yi)| (p - yi).powi(2))
                .sum::<f64>() / n as f64;
            history.losses.push(loss);

            // Gradient = (2/n) Xᵀ(ŷ - y)
            let mut grad = vec![0.0f64; p];
            for i in 0..n {
                let err = preds[i] - y[i];
                for j in 0..p {
                    grad[j] += err * x.get(i, j);
                }
            }

            // Update weights
            for j in 0..p {
                weights[j] -= self.learning_rate * 2.0 * grad[j] / n as f64;
            }

            // Convergence check
            if history.losses.len() > 1 {
                let prev = history.losses[history.losses.len() - 2];
                let curr = *history.losses.last().unwrap();
                if (prev - curr).abs() < self.tolerance {
                    break;
                }
            }
        }
        (weights, history)
    }
}
```

**ทดสอบ convergence:**

```rust
#[test]
fn test_gradient_descent_convergence() {
    // y = 3x (ไม่มี intercept)
    let n = 20;
    let x_vals: Vec<f64> = (1..=n).map(|i| i as f64).collect();
    let y_vals: Vec<f64> = x_vals.iter().map(|xi| 3.0 * xi).collect();
    let x_mat = Matrix::from_vec(n, 1, x_vals.clone());
    let gd = GradientDescent::new(0.001, 5000, 1e-8);
    let (weights, history) = gd.fit_linear(&x_mat, &y_vals);
    // slope ควรใกล้ 3
    assert!((weights[0] - 3.0).abs() < 0.05);
    // Loss ควรลดลงเรื่อย ๆ
    assert!(history.losses.first().unwrap() > history.losses.last().unwrap());
}
```

**สิ่งที่ต้องระวัง — Learning Rate:**

| Learning Rate | ผลลัพธ์ |
|---------------|---------|
| ใหญ่เกินไป (>0.1 สำหรับข้อมูลที่ไม่ได้ normalize) | Loss diverges, weights → ±∞ |
| เล็กเกินไป (<0.00001) | converge ช้ามาก ต้อง iter เพิ่มมาก |
| พอดี (0.001–0.01 สำหรับ normalized data) | converge ได้ดี |

---

### ขั้นที่ 5: Ridge Regression — L2 Regularization

Ridge regression (L2) แก้ปัญหา multicollinearity และ overfitting โดยเพิ่ม penalty term λΣwᵢ² เข้าไปใน loss function:

```
L(w) = (y - Xw)ᵀ(y - Xw) + λwᵀw
```

สูตร closed-form ของ Ridge:

```
β = (XᵀX + λI)⁻¹Xᵀy
```

โดย λ คือ regularization strength — ยิ่งมาก coefficients ยิ่งถูก shrink เข้าหาศูนย์

```rust
impl LinearRegressor {
    pub fn fit_ridge(x_data: &[Vec<f64>], y: &[f64], lambda: f64) -> Self {
        let n = x_data.len();
        let p = x_data[0].len();
        let mut x_aug = Matrix::new(n, p + 1);
        for i in 0..n {
            x_aug.set(i, 0, 1.0);
            for j in 0..p {
                x_aug.set(i, j + 1, x_data[i][j]);
            }
        }
        let xt = x_aug.transpose();
        let xtx = xt.multiply(&x_aug);
        // เพิ่ม λI เข้าไปที่ diagonal ของ XᵀX
        let xtx_reg = xtx.add_scalar_diag(lambda);
        let xty: Vec<f64> = (0..xt.rows).map(|i| {
            (0..xt.cols).map(|j| xt.get(i, j) * y[j]).sum::<f64>()
        }).collect();
        let inv = xtx_reg.inverse().expect("Regularized XtX is singular");
        let beta: Vec<f64> = (0..inv.rows).map(|i| {
            (0..inv.cols).map(|j| inv.get(i, j) * xty[j]).sum::<f64>()
        }).collect();
        LinearRegressor {
            intercept: beta[0],
            coefficients: beta[1..].to_vec(),
        }
    }
}

impl Matrix {
    pub fn add_scalar_diag(&self, lambda: f64) -> Matrix {
        assert_eq!(self.rows, self.cols);
        let mut result = self.clone();
        for i in 0..self.rows {
            let v = result.get(i, i) + lambda;
            result.set(i, i, v);
        }
        result
    }
}
```

**เมื่อไรควรใช้ Ridge:**

1. เมื่อมี **multicollinearity** — features ที่มี correlation สูงกัน (เช่น x₁ ≈ 2x₂)
2. เมื่อ **p ≈ n** หรือ p > n — จำนวน features ใกล้เคียงหรือมากกว่า samples
3. เมื่อต้องการ **stable predictions** แม้จะเสียความ interpretability เล็กน้อย

---

### ขั้นที่ 6: Logistic Regression — Binary Classification

Logistic regression ใช้ sigmoid function เพื่อ map output ของ linear function ให้อยู่ในช่วง [0, 1]:

```
σ(z) = 1 / (1 + e^{-z})
P(y=1|x) = σ(wᵀx + b)
```

Loss function คือ binary cross-entropy:

```
L = -(1/n) Σ [yᵢ ln(p̂ᵢ) + (1-yᵢ) ln(1-p̂ᵢ)]
```

```rust
pub fn sigmoid(z: f64) -> f64 {
    1.0 / (1.0 + (-z).exp())
}

pub fn binary_cross_entropy(y_true: &[f64], y_prob: &[f64]) -> f64 {
    let n = y_true.len() as f64;
    y_true.iter().zip(y_prob.iter()).map(|(yi, pi)| {
        // clamp เพื่อป้องกัน ln(0)
        let pi = pi.clamp(1e-12, 1.0 - 1e-12);
        -yi * pi.ln() - (1.0 - yi) * (1.0 - pi).ln()
    }).sum::<f64>() / n
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct LogisticRegressor {
    pub coefficients: Vec<f64>,
    pub intercept: f64,
}

impl LogisticRegressor {
    pub fn fit(x_data: &[Vec<f64>], y: &[f64], learning_rate: f64, max_iter: usize) -> Self {
        let n = x_data.len();
        let p = x_data[0].len();
        let mut w = vec![0.0f64; p];
        let mut b = 0.0f64;

        for _ in 0..max_iter {
            // Forward pass: คำนวณ probabilities
            let probs: Vec<f64> = (0..n).map(|i| {
                sigmoid(b + dot_product(&w, &x_data[i]))
            }).collect();

            // Gradient: ∂L/∂w = (1/n) Xᵀ(σ(Xw+b) - y)
            let errors: Vec<f64> = probs.iter().zip(y.iter())
                .map(|(p, yi)| p - yi)
                .collect();
            let db: f64 = errors.iter().sum::<f64>() / n as f64;
            let mut dw = vec![0.0f64; p];
            for i in 0..n {
                for j in 0..p {
                    dw[j] += errors[i] * x_data[i][j];
                }
            }

            // Update parameters
            b -= learning_rate * db;
            for j in 0..p {
                w[j] -= learning_rate * dw[j] / n as f64;
            }
        }
        LogisticRegressor { coefficients: w, intercept: b }
    }

    pub fn predict_proba(&self, x: &[f64]) -> f64 {
        sigmoid(self.intercept + dot_product(&self.coefficients, x))
    }

    pub fn predict(&self, x: &[f64]) -> u8 {
        if self.predict_proba(x) >= 0.5 { 1 } else { 0 }
    }
}
```

**ตัวอย่างการใช้งาน Logistic Regression:**

```rust
fn main() {
    // ข้อมูล linearly separable: x < 0 → class 0, x > 0 → class 1
    let x_data: Vec<Vec<f64>> = (0..20).map(|i| vec![i as f64 - 10.0]).collect();
    let y: Vec<f64> = (0..20).map(|i| if i < 10 { 0.0 } else { 1.0 }).collect();

    let model = LogisticRegressor::fit(&x_data, &y, 0.1, 1000);

    println!("predict(-5) = {}", model.predict(&[-5.0]));  // 0
    println!("predict( 0) = {}", model.predict(&[0.0]));   // 1 (borderline)
    println!("predict( 5) = {}", model.predict(&[5.0]));   // 1
}
```

---

### ขั้นที่ 7: Model Evaluation — วัดผลโมเดลอย่างถูกต้อง

**Confusion Matrix:**

```rust
#[derive(Debug, Clone)]
pub struct ConfusionMatrix {
    pub tp: usize,   // True Positive
    pub tn: usize,   // True Negative
    pub fp: usize,   // False Positive
    pub fn_: usize,  // False Negative
}

impl ConfusionMatrix {
    pub fn from_predictions(y_true: &[u8], y_pred: &[u8]) -> Self {
        let mut tp = 0; let mut tn = 0; let mut fp = 0; let mut fn_ = 0;
        for (t, p) in y_true.iter().zip(y_pred.iter()) {
            match (t, p) {
                (1, 1) => tp += 1,
                (0, 0) => tn += 1,
                (0, 1) => fp += 1,
                (1, 0) => fn_ += 1,
                _ => {}
            }
        }
        ConfusionMatrix { tp, tn, fp, fn_ }
    }
}
```

**Classification Report:**

```rust
#[derive(Debug, Clone)]
pub struct ClassificationReport {
    pub precision: f64,  // TP / (TP + FP)
    pub recall: f64,     // TP / (TP + FN)
    pub f1: f64,         // 2 * P * R / (P + R)
    pub accuracy: f64,   // (TP + TN) / total
}

impl ClassificationReport {
    pub fn from_confusion(cm: &ConfusionMatrix) -> Self {
        let precision = if cm.tp + cm.fp == 0 { 0.0 }
            else { cm.tp as f64 / (cm.tp + cm.fp) as f64 };
        let recall = if cm.tp + cm.fn_ == 0 { 0.0 }
            else { cm.tp as f64 / (cm.tp + cm.fn_) as f64 };
        let f1 = if precision + recall < 1e-12 { 0.0 }
            else { 2.0 * precision * recall / (precision + recall) };
        let total = cm.tp + cm.tn + cm.fp + cm.fn_;
        let accuracy = if total == 0 { 0.0 }
            else { (cm.tp + cm.tn) as f64 / total as f64 };
        ClassificationReport { precision, recall, f1, accuracy }
    }
}
```

**Train/Test Split ด้วย LCG Shuffle:**

```rust
pub struct TrainTestSplit;

impl TrainTestSplit {
    pub fn split(n: usize, ratio: f64, seed: u64) -> (Vec<usize>, Vec<usize>) {
        let mut indices: Vec<usize> = (0..n).collect();
        // LCG (Linear Congruential Generator) สำหรับ reproducible shuffle
        let mut state = seed;
        for i in (1..n).rev() {
            state = state
                .wrapping_mul(6364136223846793005)
                .wrapping_add(1442695040888963407);
            let j = (state >> 33) as usize % (i + 1);
            indices.swap(i, j);
        }
        let train_size = (n as f64 * ratio).round() as usize;
        let train = indices[..train_size].to_vec();
        let test = indices[train_size..].to_vec();
        (train, test)
    }
}
```

**K-Fold Cross-Validation:**

```rust
pub fn k_fold_indices(n: usize, k: usize) -> Vec<(Vec<usize>, Vec<usize>)> {
    let fold_size = n / k;
    let indices: Vec<usize> = (0..n).collect();
    let folds: Vec<Vec<usize>> = (0..k).map(|i| {
        let start = i * fold_size;
        let end = if i == k - 1 { n } else { start + fold_size };
        indices[start..end].to_vec()
    }).collect();

    (0..k).map(|i| {
        let test = folds[i].clone();
        let train: Vec<usize> = folds.iter().enumerate()
            .filter(|(fi, _)| *fi != i)
            .flat_map(|(_, f)| f.iter().cloned())
            .collect();
        (train, test)
    }).collect()
}
```

---

### ขั้นที่ 8: ROC Curve และ AUC

ROC curve แสดงความสัมพันธ์ระหว่าง **True Positive Rate** (sensitivity) และ **False Positive Rate** (1-specificity) เมื่อเปลี่ยน threshold ต่าง ๆ

AUC (Area Under the Curve) คือพื้นที่ใต้ ROC curve:
- AUC = 1.0 → perfect classifier
- AUC = 0.5 → random guessing

```rust
pub fn roc_curve(y_true: &[u8], y_scores: &[f64]) -> Vec<(f64, f64)> {
    let mut thresholds: Vec<f64> = y_scores.to_vec();
    thresholds.sort_by(|a, b| b.partial_cmp(a).unwrap()); // sort descending
    thresholds.dedup();
    thresholds.push(0.0); // threshold ต่ำสุด

    let pos = y_true.iter().filter(|&&v| v == 1).count() as f64;
    let neg = y_true.iter().filter(|&&v| v == 0).count() as f64;

    let mut points = vec![(0.0f64, 0.0f64)]; // origin
    for &thresh in &thresholds {
        let tp = y_true.iter().zip(y_scores.iter())
            .filter(|(&t, &s)| t == 1 && s >= thresh).count();
        let fp = y_true.iter().zip(y_scores.iter())
            .filter(|(&t, &s)| t == 0 && s >= thresh).count();
        let tpr = if pos > 0.0 { tp as f64 / pos } else { 0.0 };
        let fpr = if neg > 0.0 { fp as f64 / neg } else { 0.0 };
        points.push((fpr, tpr));
    }
    points
}

pub fn auc(roc_points: &[(f64, f64)]) -> f64 {
    let mut pts = roc_points.to_vec();
    pts.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap());
    // Trapezoidal rule
    let mut area = 0.0;
    for i in 1..pts.len() {
        let dx = pts[i].0 - pts[i - 1].0;
        let avg_y = (pts[i].1 + pts[i - 1].1) / 2.0;
        area += dx * avg_y;
    }
    area
}
```

**ทำความเข้าใจ Trapezoidal Rule:**

เราประมาณ area ใต้กราฟโดยแบ่งเป็นสี่เหลี่ยมคางหมู ๆ ละ `dx × avg_height` แล้วบวกทั้งหมด

---

### ขั้นที่ 9: Model Serialization ด้วย serde_json

การ save/load โมเดลสำคัญมากสำหรับ production — เราไม่ต้อง retrain ทุกครั้งที่ start application

```rust
use std::fs;

fn save_model(model: &LinearRegressor, path: &str) -> Result<(), Box<dyn std::error::Error>> {
    let json = serde_json::to_string_pretty(model)?;
    fs::write(path, json)?;
    Ok(())
}

fn load_model(path: &str) -> Result<LinearRegressor, Box<dyn std::error::Error>> {
    let json = fs::read_to_string(path)?;
    let model: LinearRegressor = serde_json::from_str(&json)?;
    Ok(model)
}
```

**ตัวอย่าง JSON output:**

```json
{
  "coefficients": [
    1.999999999999996,
    3.0
  ],
  "intercept": 1.0000000000000568
}
```

**Demo Program Output จริง:**

```
=== Linear Regression Demo ===

Simple Linear Regression (y ≈ 2x):
  slope     = 1.9976
  intercept = 0.0357
  R²        = 0.9993
  RMSE      = 0.1198
  MAE       = 0.1006
  predict(9) = 18.0143

Multiple Linear Regression:
  intercept    = 1.0000
  coefficients = ["2.0000", "3.0000"]

Logistic Regression:
  predict(-5) = 0
  predict( 0) = 1
  predict( 5) = 1

Train/Test Split (100 samples, 80/20):
  train size = 80
  test size  = 20

5-Fold Cross-Validation:
  Fold 1: train=80, test=20
  Fold 2: train=80, test=20
  Fold 3: train=80, test=20
  Fold 4: train=80, test=20
  Fold 5: train=80, test=20

Serialized model:
{
  "coefficients": [
    1.999999999999996,
    3.0
  ],
  "intercept": 1.0000000000000568
}
```

---

## การทดสอบ (Testing)

โปรเจคนี้มี **22 unit tests** ครอบคลุมทุก component สำคัญ:

```rust
// ตัวอย่าง unit tests สำคัญ

#[cfg(test)]
mod tests {
    use super::*;

    // --- Matrix tests ---

    #[test]
    fn test_matrix_multiply_2x2() {
        let a = Matrix::from_vec(2, 2, vec![1.0, 2.0, 3.0, 4.0]);
        let b = Matrix::from_vec(2, 2, vec![5.0, 6.0, 7.0, 8.0]);
        let c = a.multiply(&b);
        // [1*5+2*7, 1*6+2*8, 3*5+4*7, 3*6+4*8] = [19, 22, 43, 50]
        assert_eq!(c.get(0, 0), 19.0);
        assert_eq!(c.get(0, 1), 22.0);
        assert_eq!(c.get(1, 0), 43.0);
        assert_eq!(c.get(1, 1), 50.0);
    }

    #[test]
    fn test_matrix_inverse_2x2() {
        // [2, 1; 1, 1]^{-1} = [1, -1; -1, 2]
        let a = Matrix::from_vec(2, 2, vec![2.0, 1.0, 1.0, 1.0]);
        let inv = a.inverse().expect("Should be invertible");
        assert!((inv.get(0, 0) - 1.0).abs() < 1e-9);
        assert!((inv.get(0, 1) - (-1.0)).abs() < 1e-9);
        assert!((inv.get(1, 0) - (-1.0)).abs() < 1e-9);
        assert!((inv.get(1, 1) - 2.0).abs() < 1e-9);
    }

    #[test]
    fn test_matrix_singular_returns_none() {
        // [1, 2; 2, 4] — singular (row 2 = 2 × row 1)
        let a = Matrix::from_vec(2, 2, vec![1.0, 2.0, 2.0, 4.0]);
        assert!(a.inverse().is_none());
    }

    #[test]
    fn test_ols_matrix_simple_case() {
        // y = 1 + 2*x  =>  beta = [1, 2]
        let x_data = vec![
            vec![1.0, 1.0],  // [bias, x]
            vec![1.0, 2.0],
            vec![1.0, 3.0],
            vec![1.0, 4.0],
        ];
        let y = vec![3.0, 5.0, 7.0, 9.0];
        let x_matrix = Matrix::from_vec(4, 2,
            x_data.iter().flat_map(|r| r.iter().cloned()).collect());
        let beta = ols_matrix(&x_matrix, &y);
        assert!((beta[0] - 1.0).abs() < 1e-6);
        assert!((beta[1] - 2.0).abs() < 1e-6);
    }

    #[test]
    fn test_gradient_descent_convergence() {
        let n = 20;
        let x_vals: Vec<f64> = (1..=n).map(|i| i as f64).collect();
        let y_vals: Vec<f64> = x_vals.iter().map(|xi| 3.0 * xi).collect();
        let x_mat = Matrix::from_vec(n, 1, x_vals);
        let gd = GradientDescent::new(0.001, 5000, 1e-8);
        let (weights, history) = gd.fit_linear(&x_mat, &y_vals);
        assert!((weights[0] - 3.0).abs() < 0.05);
        assert!(history.losses.first().unwrap() > history.losses.last().unwrap());
    }

    #[test]
    fn test_sigmoid_values() {
        assert!((sigmoid(0.0) - 0.5).abs() < 1e-9);
        assert!(sigmoid(100.0) > 0.99);
        assert!(sigmoid(-100.0) < 0.01);
    }

    #[test]
    fn test_binary_cross_entropy() {
        let y_true = vec![1.0, 0.0, 1.0, 0.0];
        let y_pred_perfect = vec![1.0 - 1e-10, 1e-10, 1.0 - 1e-10, 1e-10];
        let loss = binary_cross_entropy(&y_true, &y_pred_perfect);
        assert!(loss < 0.0001);
    }

    #[test]
    fn test_train_test_split_proportions() {
        let (train, test) = TrainTestSplit::split(100, 0.8, 42);
        assert_eq!(train.len(), 80);
        assert_eq!(test.len(), 20);
    }

    #[test]
    fn test_k_fold_sizes() {
        let folds = k_fold_indices(100, 5);
        assert_eq!(folds.len(), 5);
        for (train, test) in &folds {
            assert_eq!(test.len(), 20);
            assert_eq!(train.len(), 80);
        }
    }
}
```

**`cargo test` output จริง:**

```
$ cargo test
   Compiling linear-regression v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.87s
     Running unittests src/lib.rs (target/debug/deps/linear_regression-62bdf8a3c2747e5a)

running 22 tests
test tests::test_auc_perfect_classifier ... ok
test tests::test_binary_cross_entropy ... ok
test tests::test_classification_report ... ok
test tests::test_dot_product ... ok
test tests::test_k_fold_all_indices_covered ... ok
test tests::test_gradient_descent_convergence ... ok
test tests::test_k_fold_sizes ... ok
test tests::test_matrix_inverse_2x2 ... ok
test tests::test_matrix_inverse_identity ... ok
test tests::test_matrix_multiply_2x3_3x2 ... ok
test tests::test_matrix_multiply_2x2 ... ok
test tests::test_matrix_singular_returns_none ... ok
test tests::test_matrix_transpose ... ok
test tests::test_ols_matrix_simple_case ... ok
test tests::test_sigmoid_values ... ok
test tests::test_ridge_vs_ols_on_collinear_data ... ok
test tests::test_simple_lr_r_squared_perfect ... ok
test tests::test_simple_lr_fit_perfect_line ... ok
test tests::test_train_test_split_proportions ... ok
test tests::test_train_test_split_no_overlap ... ok
test tests::test_simple_lr_rmse_zero_on_perfect ... ok
test tests::test_logistic_regressor_linearly_separable ... ok

test result: ok. 22 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/linear_regression-8094515a7adec602)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests linear_regression

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**coverage ของ tests:**

| Test | ครอบคลุม |
|------|----------|
| `test_matrix_multiply_2x2` | Matrix::multiply กับ 2×2 |
| `test_matrix_multiply_2x3_3x2` | Matrix::multiply กับ non-square |
| `test_matrix_transpose` | Matrix::transpose |
| `test_matrix_inverse_2x2` | Gauss-Jordan inverse |
| `test_matrix_inverse_identity` | inverse(I) = I |
| `test_matrix_singular_returns_none` | singular matrix → None |
| `test_dot_product` | dot_product function |
| `test_simple_lr_fit_perfect_line` | OLS slope/intercept |
| `test_simple_lr_r_squared_perfect` | R² = 1.0 on perfect data |
| `test_simple_lr_rmse_zero_on_perfect` | RMSE = 0 on perfect data |
| `test_ols_matrix_simple_case` | matrix OLS β = [1, 2] |
| `test_gradient_descent_convergence` | GD converges to slope 3 |
| `test_ridge_vs_ols_on_collinear_data` | Ridge on collinear features |
| `test_sigmoid_values` | sigmoid(0)=0.5, limits |
| `test_binary_cross_entropy` | BCE loss on perfect/random |
| `test_logistic_regressor_linearly_separable` | LR predicts class correctly |
| `test_train_test_split_proportions` | 80/20 split sizes |
| `test_train_test_split_no_overlap` | no overlap + full coverage |
| `test_k_fold_sizes` | 5-fold: 80 train, 20 test each |
| `test_k_fold_all_indices_covered` | all indices appear in test |
| `test_auc_perfect_classifier` | AUC ≈ 1.0 for perfect scores |
| `test_classification_report` | precision/recall/f1/accuracy |

---

## ข้อผิดพลาดที่มักเจอ (Common Pitfalls)

### Pitfall 1: Multicollinearity ทำให้ XᵀX Singular

**ปัญหา:** เมื่อ features มี linear relationship กัน (เช่น x₂ = 2x₁) matrix XᵀX จะ singular และหา inverse ไม่ได้

```rust
// ❌ ปัญหา: x_data[1] = 2 * x_data[0] เสมอ
let x_data = vec![
    vec![1.0, 2.0],
    vec![2.0, 4.0],
    vec![3.0, 6.0],
];
let model = LinearRegressor::fit(&x_data, &y);
// → panics: "XtX is singular"
```

**วิธีแก้:** ใช้ Ridge regression ที่เพิ่ม λI เข้าไป หรือทำ feature selection ก่อน

```rust
// ✅ แก้ไข: ใช้ Ridge แทน
let model = LinearRegressor::fit_ridge(&x_data, &y, 0.01);
```

### Pitfall 2: Learning Rate ที่ไม่เหมาะสมทำให้ Gradient Explode

**ปัญหา:** learning rate ใหญ่เกินไปทำให้ weights update แล้วข้ามจุด minimum ไป จากนั้น loss diverge ไปอนันต์

```rust
// ❌ learning_rate ใหญ่เกินไปสำหรับข้อมูลที่ไม่ได้ normalize
let gd = GradientDescent::new(1.0, 1000, 1e-8);
// → weights → NaN ภายใน 10 iterations แรก
```

**วิธีแก้:** normalize features ก่อน หรือเริ่มจาก learning rate เล็ก ๆ แล้วค่อยเพิ่ม

```rust
// ✅ แก้ไข: normalize data และใช้ learning rate เล็กกว่า
fn normalize(data: &[f64]) -> Vec<f64> {
    let mean = data.iter().sum::<f64>() / data.len() as f64;
    let std = (data.iter().map(|x| (x - mean).powi(2)).sum::<f64>()
        / data.len() as f64).sqrt();
    data.iter().map(|x| (x - mean) / (std + 1e-8)).collect()
}
let gd = GradientDescent::new(0.01, 1000, 1e-8);
```

### Pitfall 3: Binary Cross-Entropy กับ log(0)

**ปัญหา:** ถ้า prediction เป็น 0.0 หรือ 1.0 พอดี การคำนวณ ln(0) จะได้ -∞ ทำให้ loss เป็น NaN

```rust
// ❌ ปัญหา: sigmoid(1000) ≈ 1.0 อย่างแน่นอน
let y_true = vec![0.0];
let y_prob = vec![1.0];  // ln(1-1.0) = ln(0) = -∞
let loss = binary_cross_entropy(&y_true, &y_prob); // NaN!
```

**วิธีแก้:** clamp probabilities ให้อยู่ในช่วง [ε, 1-ε]

```rust
// ✅ แก้ไข
let pi = pi.clamp(1e-12, 1.0 - 1e-12);
```

### Pitfall 4: Train/Test Split ด้วย index เดียวกันกับ Training Leads to Data Leakage

**ปัญหา:** ถ้าเราใช้ข้อมูล test set ระหว่าง training (เช่น normalize ด้วย mean/std ของข้อมูลทั้งหมด) จะเกิด **data leakage** ทำให้ metrics ดูดีกว่าความเป็นจริง

```rust
// ❌ ปัญหา: normalize ด้วย statistics ของ test set ด้วย
let all_mean = all_data.iter().sum::<f64>() / all_data.len() as f64;
let normalized = all_data.iter().map(|x| x - all_mean).collect();
// แล้ว split → test set รู้ mean ของตัวเองไปแล้ว!
```

**วิธีแก้:** คำนวณ normalization statistics จาก **training set เท่านั้น** แล้ว apply กับ test set

```rust
// ✅ แก้ไข
let (train_idx, test_idx) = TrainTestSplit::split(n, 0.8, 42);
let train_data: Vec<f64> = train_idx.iter().map(|&i| data[i]).collect();
let train_mean = train_data.iter().sum::<f64>() / train_data.len() as f64;
// normalize ทั้ง train และ test ด้วย train_mean เท่านั้น
```

### Pitfall 5: Floating-Point Precision ใน Matrix Operations

**ปัญหา:** การคำนวณ matrix หลายชั้นสะสม floating-point error ทำให้ผลลัพธ์ไม่แน่นอน

```rust
// อาจได้ 1.9999999999999996 แทน 2.0 — ปกติมาก
let model = LinearRegressor::fit(&x_data, &y);
println!("{}", model.coefficients[0]); // 1.9999999999999996
```

**วิธีแก้:** ใช้ threshold เมื่อ compare ไม่ใช่ `==`

```rust
// ✅ แก้ไข
assert!((model.coefficients[0] - 2.0).abs() < 1e-9);
```

### Pitfall 6: K-Fold สุดท้ายอาจมีขนาดไม่เท่ากัน

**ปัญหา:** ถ้า n ไม่หารด้วย k ลงตัว fold สุดท้ายจะมีขนาดใหญ่กว่า

```rust
// n=101, k=5 → folds ขนาด [20, 20, 20, 20, 21]
let folds = k_fold_indices(101, 5);
```

**วิธีแก้:** เราแก้ด้วยการให้ fold สุดท้ายรับ remainder ทั้งหมด หรือใช้ stratified k-fold

```rust
let end = if i == k - 1 { n } else { start + fold_size };
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/linear-regression
```

### Model Persistence สำหรับ Production API

```rust
use std::path::Path;

// Train ครั้งเดียว บันทึก
if !Path::new("model.json").exists() {
    let model = LinearRegressor::fit(&training_data, &labels);
    let json = serde_json::to_string_pretty(&model).unwrap();
    std::fs::write("model.json", json).unwrap();
    println!("Model saved.");
} else {
    // Load ทุกครั้งที่ start
    let json = std::fs::read_to_string("model.json").unwrap();
    let model: LinearRegressor = serde_json::from_str(&json).unwrap();
    println!("Model loaded. intercept={:.4}", model.intercept);
}
```

### Integration กับ Web Service

สำหรับ production API ใช้ร่วมกับ `axum` หรือ `actix-web`:

```rust
// Cargo.toml เพิ่ม:
// axum = "0.7"
// tokio = { version = "1", features = ["full"] }

use axum::{extract::Json, routing::post, Router};

#[derive(serde::Deserialize)]
struct PredictRequest {
    features: Vec<f64>,
}

async fn predict(
    Json(req): Json<PredictRequest>,
) -> Json<f64> {
    // โหลด model จาก state หรือ file
    let model = load_model("model.json").unwrap();
    Json(model.predict(&req.features))
}
```

### Docker (ถ้าต้องการ containerize)

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/linear-regression /usr/local/bin/
CMD ["linear-regression"]
```

---

## ความรู้เสริม: คณิตศาสตร์เบื้องหลัง

### ทำไม OLS ถึง minimize MSE?

สูตร β = (XᵀX)⁻¹Xᵀy มาจากการ differentiate loss function MSE = ||y - Xβ||² แล้วตั้งเท่ากับศูนย์:

```
∂/∂β ||y - Xβ||² = -2Xᵀ(y - Xβ) = 0
Xᵀy = XᵀXβ
β = (XᵀX)⁻¹Xᵀy
```

นี่คือ **Normal Equations** — จุดที่ gradient เป็นศูนย์คือ global minimum เพราะ MSE เป็น convex function

### ทำไม Ridge ถึงช่วยแก้ Multicollinearity?

XᵀX อาจ nearly singular เมื่อ features มี correlation สูง ทำให้ inverse ไม่ stable การเพิ่ม λI ทำให้ eigenvalues ทุกตัวเพิ่มขึ้น λ ทำให้ matrix invertible เสมอ:

```
(XᵀX + λI)⁻¹ มีอยู่เสมอถ้า λ > 0
```

### Logistic Regression ไม่ใช่ Regression จริงๆ

แม้ชื่อจะมีคำว่า "Regression" แต่ Logistic Regression เป็น **classification algorithm** ที่ model probability P(y=1|x) แล้วใช้ threshold (ปกติ 0.5) ตัดสินใจ class

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Lasso Regression (L1 Regularization) ⭐⭐⭐

Lasso ใช้ penalty ||w||₁ แทน ||w||₂² ทำให้บาง coefficients เป็น 0 พอดี (sparse solution) ซึ่งมีประโยชน์สำหรับ feature selection แต่ไม่มี closed-form จึงต้อง fit ด้วย **coordinate descent**:

สำหรับ feature j เดียว ในขณะที่ features อื่น fixed:

```
wⱼ ← soft_threshold(rⱼ / ||xⱼ||², λ / ||xⱼ||²)

soft_threshold(z, γ) = sign(z) × max(|z| - γ, 0)
```

ลอง implement `LassoRegressor::fit(x, y, lambda, max_iter)` และทดสอบว่า coefficients เล็กน้อยถูก set เป็น 0 จริง

### แบบฝึกหัดที่ 2: Polynomial Regression ⭐⭐

Linear regression สามารถ fit non-linear patterns ได้โดยเพิ่ม polynomial features:

```rust
fn polynomial_features(x: &[f64], degree: usize) -> Vec<Vec<f64>> {
    x.iter().map(|&xi| {
        (1..=degree).map(|d| xi.powi(d as i32)).collect()
    }).collect()
}
```

จากนั้น fit `LinearRegressor` กับ polynomial features แทน ลองดูว่า degree เท่าไรที่ fit curve y = x² + 2x + 1 ได้

### แบบฝึกหัดที่ 3: Stochastic Gradient Descent (SGD) ⭐⭐

Batch GD ใช้ทุก sample ต่อ iteration ซึ่งช้าสำหรับ dataset ใหญ่ SGD ใช้ **1 sample ต่อ iteration** แต่ noisy กว่า:

```rust
pub fn fit_sgd(&self, x: &Matrix, y: &[f64], seed: u64) -> (Vec<f64>, LossHistory) {
    // สุ่ม sample 1 ตัวต่อ iteration
    // คำนวณ gradient จาก sample เดียว
    // อัปเดต weights
}
```

เปรียบเทียบ convergence speed ระหว่าง batch GD, mini-batch (32 samples), และ SGD

### แบบฝึกหัดที่ 4: Multiclass Logistic Regression (Softmax) ⭐⭐⭐

Logistic regression ที่เราสร้างรองรับแค่ binary classification ขยายเป็น **Softmax Regression** สำหรับ k classes:

```
P(y=k|x) = exp(wₖᵀx) / Σⱼ exp(wⱼᵀx)
```

implement `SoftmaxRegressor::fit(x, y, k_classes, lr, iter)` และทดสอบกับ 3-class dataset

### แบบฝึกหัดที่ 5: Feature Importance via Permutation ⭐⭐⭐

วัดความสำคัญของแต่ละ feature โดย:
1. วัด baseline score (R² หรือ accuracy)
2. สุ่ม shuffle feature j
3. วัด score ใหม่ — ถ้าลดลงมาก แสดงว่า feature j สำคัญ

```rust
fn permutation_importance(
    model: &LinearRegressor,
    x: &[Vec<f64>],
    y: &[f64],
    n_repeat: usize,
    seed: u64,
) -> Vec<f64> {
    // คืน importance score ต่อ feature
}
```

### แบบฝึกหัดที่ 6: Learning Curves ⭐⭐

เขียนฟังก์ชันที่ train โมเดลด้วย training set ขนาดต่าง ๆ แล้ว plot training score vs validation score:

```rust
fn learning_curve(
    x: &[Vec<f64>],
    y: &[f64],
    sizes: &[usize],  // เช่น [10, 20, 50, 100, 200]
) -> Vec<(usize, f64, f64)>  // (size, train_r2, val_r2)
```

Learning curves ช่วยวิเคราะห์ว่าโมเดล **underfitting** (ทั้ง train/val score ต่ำ) หรือ **overfitting** (train สูง, val ต่ำ)

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **ML library สำหรับ regression และ classification** ครบชุดตั้งแต่รากฐาน:

1. **Matrix struct** พร้อม Gauss-Jordan inverse — รากฐานของทุกอย่าง
2. **Simple Linear Regression** — OLS closed-form ด้วยสูตรตรง
3. **OLS Matrix Form** — β = (XᵀX)⁻¹Xᵀy สำหรับ multiple features
4. **Gradient Descent** — iterative optimization พร้อม loss tracking
5. **Ridge Regression** — L2 regularization แก้ multicollinearity
6. **Logistic Regression** — sigmoid + cross-entropy สำหรับ classification
7. **Model Evaluation** — confusion matrix, precision/recall/F1, ROC/AUC
8. **Train/Test Split & K-Fold** — proper model evaluation framework
9. **Model Serialization** — save/load ด้วย serde_json

**Pattern สำคัญที่ได้เรียน:**

- **Ownership ใน numerical computing** — การ clone Matrix ต้องระวัง memory ใช้ `&self` เมื่อทำได้
- **Option สำหรับ failure cases** — `inverse()` คืน `Option<Matrix>` แทน panic
- **Iterator idioms** — ใช้ `.map().sum()` แทน for-loop ที่ verbose กว่า
- **Numeric stability** — clamp, epsilon comparison, partial pivoting

**เชื่อมโยงกับโปรเจคถัดไป:**

โปรเจคถัดไป [Project I02: Neural Network](project-i02-neural-network.md) จะต่อยอดจากพื้นฐานที่สร้างในโปรเจคนี้ — matrix operations เดิมจะถูกนำมาใช้ใน forward/backward pass ของ neural network, gradient descent จะกลายเป็น backpropagation, และ logistic regression จะเป็น single neuron layer เมื่อเข้าใจ regression อย่างลึกซึ้งแล้ว neural network จะไม่ดู mysterious อีกต่อไป

---

**โปรเจคก่อนหน้า:** [Project H10: Network Proxy](project-h10-network-proxy.md) | **โปรเจคถัดไป:** [Project I02: Neural Network](project-i02-neural-network.md)
