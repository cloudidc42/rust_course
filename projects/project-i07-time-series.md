# Project I07: Time Series Forecasting

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 16 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **ไลบรารี Time Series Forecasting แบบครบวงจร** ด้วย pure Rust — ตั้งแต่โครงสร้างข้อมูล Time Series ไปจนถึงอัลกอริทึมการพยากรณ์ขั้นสูง ได้แก่ ARIMA-like models, Exponential Smoothing ทั้ง 3 ระดับ, STL Decomposition, และ Anomaly Detection พร้อม Evaluation Framework ที่ใช้งานได้จริงในระดับ production

**Time Series** คือข้อมูลที่มีลำดับเวลากำกับ เช่น ราคาหุ้นรายวัน, อุณหภูมิรายชั่วโมง, ปริมาณการใช้ไฟฟ้ารายเดือน — ข้อมูลเหล่านี้มีโครงสร้างพิเศษที่ต้องใช้เทคนิคเฉพาะในการวิเคราะห์และพยากรณ์ ต่างจาก Regression ทั่วไปเพราะ **ลำดับของข้อมูลมีความหมาย** และค่าในอดีตมีอิทธิพลต่อค่าในอนาคต

จุดเด่นของโปรเจคนี้คือการเขียนทุกอัลกอริทึมจากศูนย์โดยไม่พึ่งพาไลบรารี ML สำเร็จรูป ทำให้คุณเข้าใจว่า ARIMA ทำงานอย่างไรจริง ๆ ว่า Holt-Winters ต่างจาก SES ตรงไหน และทำไม STL Decomposition ถึงสำคัญสำหรับข้อมูลที่มี seasonality

**Use cases จริงในโลก production:**
- **Financial forecasting** — พยากรณ์ราคาหุ้น, อัตราแลกเปลี่ยน, ดัชนีตลาด
- **Demand forecasting** — คาดการณ์ยอดขาย, ความต้องการสินค้า สำหรับ supply chain management
- **Infrastructure monitoring** — ตรวจจับ anomaly ใน metrics เช่น CPU usage, network traffic, error rate
- **Energy management** — พยากรณ์ความต้องการไฟฟ้าเพื่อ grid balancing
- **IoT sensor analysis** — วิเคราะห์ข้อมูล sensor แบบ real-time, ตรวจจับอุปกรณ์เสีย
- **Business KPI tracking** — วิเคราะห์แนวโน้ม trend, seasonality ใน business metrics

## สิ่งที่จะได้เรียนรู้

- **Time Series data structure** — ออกแบบ struct ที่เก็บ timestamps + values อย่างมีประสิทธิภาพ และตรวจสอบ stationarity
- **Moving average family** — เข้าใจความแตกต่างระหว่าง SMA, EMA, WMA และเลือกใช้ให้เหมาะสม
- **Differencing & stationarity** — ทำไม ARIMA ถึงต้อง difference ข้อมูล และ inverse diff ทำงานอย่างไร
- **AR(p) model via OLS** — fit autoregressive model ด้วย Ordinary Least Squares และ Gauss-Jordan matrix inversion
- **MA(q) model** — Hannan-Rissanen approximation สำหรับ fit moving average of errors
- **Exponential smoothing** — SES, Holt's linear trend, Holt-Winters triple smoothing
- **STL decomposition** — แยก trend, seasonal, residual ด้วย loess-like smoothing
- **Anomaly detection** — Z-score, IQR, rolling statistics สำหรับตรวจจับ outliers
- **Forecast evaluation** — MAE, RMSE, MAPE และ walk-forward validation ที่ถูกต้อง

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, Vec, slices
- **Part 21–30**: Iterators, closures, `map`, `filter`, `zip`, `windows`
- **Part 31–40**: Error handling ด้วย `Result<T, E>`, `?` operator, trait implementations
- **Part 41–50**: File I/O, `serde`/`serde_json` สำหรับ serialization
- **Part 51–60**: Numeric operations, `f64` math functions
- **Part 96–100**: Linear algebra พื้นฐาน (matrix multiply, Gauss-Jordan) — ดูจาก Project I01

## โครงสร้างโปรเจค (Project Layout)

```
time-series/
├── src/
│   ├── lib.rs           ← core library (ทุก algorithm)
│   └── main.rs          ← demo program แสดงผลการทำงาน
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Raw Data (timestamps + values)
         │
         ▼
┌─────────────────────┐
│  TimeSeries struct  │  ← stationarity check, mean, variance
└─────────────────────┘
         │
    ┌────┴──────────────────┬───────────────────┐
    ▼                       ▼                   ▼
┌──────────┐        ┌──────────────┐    ┌────────────────┐
│ Moving   │        │ Differencing │    │  Lag Features  │
│ Averages │        │ (d-th order) │    │  (AR design    │
│ SMA/EMA/ │        │ + inverse    │    │   matrix)      │
│ WMA      │        └──────────────┘    └────────────────┘
└──────────┘                │                   │
                            ▼                   ▼
                   ┌─────────────────────────────────┐
                   │        Model Fitting             │
                   │  AR(p) OLS │ MA(q) H-R │ Holt   │
                   │  SES       │ Holt-W    │ STL    │
                   └─────────────────────────────────┘
                            │
                   ┌────────┴──────────┐
                   ▼                   ▼
           ┌──────────────┐   ┌──────────────────────┐
           │  Forecasting  │   │  Anomaly Detection   │
           │  h steps      │   │  Z-score/IQR/Rolling │
           └──────────────┘   └──────────────────────┘
                   │
                   ▼
          ┌─────────────────┐
          │   Evaluation     │
          │  MAE/RMSE/MAPE   │
          │  Walk-forward CV │
          └─────────────────┘
```

### Design Decisions

**ทำไมถึงใช้ `i64` สำหรับ timestamp?**

Unix timestamp เป็น `i64` (seconds since epoch) ซึ่ง interoperable กับ datetime libraries ส่วนใหญ่ และรองรับทั้งค่า negative (ก่อน 1970) และวันที่ในอนาคตหลายพันปี หากต้องการ nanosecond precision ให้ขยายเป็น `(i64, u32)` หรือใช้ crate `time`

**ทำไมถึง implement OLS ด้วย Gauss-Jordan เอง?**

เช่นเดียวกับ Project I01 — จุดประสงค์คือเรียนรู้ Algorithm ไม่ใช่แค่ใช้ black box สำหรับ production จริง ควรใช้ `nalgebra` หรือ `faer` ซึ่ง numerically stable กว่ามาก

**ทำไม STL decomposition ถึงใช้ centered moving average?**

Centered MA (2×m-MA) เหมาะกับข้อมูลที่มี period เป็นเลขคู่ เช่น 12 เดือน เพราะหลีกเลี่ยง phase shift ที่เกิดจาก even-window MA ทั่วไป

**Stationarity ratio คืออะไร?**

แทนที่จะ implement Augmented Dickey-Fuller test ที่ซับซ้อน เราใช้ proxy อย่างง่าย: คำนวณ variance ของครึ่งแรกและครึ่งหลังของ series แล้วหาอัตราส่วน ถ้า ratio ≈ 1.0 แสดงว่า variance คงที่ (stationary); ถ้า ratio ≫ 1.0 หรือ ≪ 1.0 แสดงว่า non-stationary

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างข้อมูล TimeSeries และ Stationarity Check

ก่อนอื่นต้องสร้าง `Cargo.toml` ที่มี dependencies ที่จำเป็น:

```toml
[package]
name = "time-series"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rand = "0.8"
```

**หัวใจของทุก algorithm คือ `TimeSeries` struct** ซึ่งเก็บ timestamps คู่กับ values:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TimeSeries {
    pub timestamps: Vec<i64>, // Unix timestamp (seconds)
    pub values: Vec<f64>,
}

impl TimeSeries {
    pub fn new(timestamps: Vec<i64>, values: Vec<f64>) -> Result<Self, String> {
        if timestamps.len() != values.len() {
            return Err(format!(
                "timestamps.len() = {} != values.len() = {}",
                timestamps.len(),
                values.len()
            ));
        }
        if timestamps.is_empty() {
            return Err("TimeSeries ต้องมีข้อมูลอย่างน้อย 1 จุด".to_string());
        }
        Ok(TimeSeries { timestamps, values })
    }

    pub fn len(&self) -> usize {
        self.values.len()
    }

    pub fn is_empty(&self) -> bool {
        self.values.is_empty()
    }

    pub fn mean(&self) -> f64 {
        self.values.iter().sum::<f64>() / self.values.len() as f64
    }

    pub fn variance(&self) -> f64 {
        let m = self.mean();
        self.values.iter().map(|v| (v - m).powi(2)).sum::<f64>()
            / self.values.len() as f64
    }

    pub fn std_dev(&self) -> f64 {
        self.variance().sqrt()
    }

    /// Stationarity ratio: var(second_half) / var(first_half)
    /// ≈ 1.0 → stationary, ≫/≪ 1.0 → non-stationary
    pub fn stationarity_ratio(&self) -> f64 {
        let half = self.len() / 2;
        if half < 2 {
            return 1.0;
        }
        let first_half = &self.values[..half];
        let second_half = &self.values[half..];
        let var_first = variance_of(first_half);
        let var_second = variance_of(second_half);
        if var_first == 0.0 {
            return 1.0;
        }
        var_second / var_first
    }
}

fn variance_of(data: &[f64]) -> f64 {
    if data.is_empty() { return 0.0; }
    let m = data.iter().sum::<f64>() / data.len() as f64;
    data.iter().map(|v| (v - m).powi(2)).sum::<f64>() / data.len() as f64
}
```

**แนวคิดสำคัญ:**
- `timestamps` และ `values` มีขนาดเท่ากันเสมอ (enforced ใน constructor)
- `stationarity_ratio` ≈ 1.0 หมายถึงข้อมูลมี variance คงที่ตลอดเวลา (stationary)
- `stationarity_ratio` ≫ 1.0 หมายถึง variance เพิ่มขึ้น (non-stationary — ต้องทำ differencing)

**ตัวอย่างการใช้งาน:**

```rust
use rand::prelude::*;

fn main() {
    let mut rng = StdRng::seed_from_u64(42);
    let n = 60usize;
    let period = 12usize;

    // สร้างข้อมูล synthetic: trend + seasonal + noise
    let values: Vec<f64> = (0..n).map(|i| {
        let trend = i as f64 * 0.5;
        let seasonal = 5.0 * (2.0 * std::f64::consts::PI * i as f64
                              / period as f64).sin();
        let noise: f64 = rng.gen_range(-0.5..0.5);
        trend + seasonal + noise
    }).collect();

    let timestamps: Vec<i64> = (0..n as i64).collect();
    let ts = TimeSeries::new(timestamps, values.clone()).unwrap();

    println!("จำนวนข้อมูล: {}", ts.len());
    println!("ค่าเฉลี่ย:   {:.3}", ts.mean());
    println!("ค่าเบี่ยงเบน: {:.3}", ts.std_dev());
    println!("Stationarity ratio: {:.3}", ts.stationarity_ratio());
    // Stationarity ratio: 0.983 → ใกล้ 1.0 → ค่อนข้าง stationary
}
```

**Output:**
```
จำนวนข้อมูล: 60
ค่าเฉลี่ย:   14.729
ค่าเบี่ยงเบน: 8.855
Stationarity ratio: 0.983
```

---

### ขั้นที่ 2: Moving Averages และ Lag Features

Moving average เป็น building block สำคัญของ time series analysis ใช้ทั้ง smoothing และ feature engineering:

```rust
/// Simple Moving Average: ค่าเฉลี่ยเคลื่อนที่แบบเท่ากัน
pub fn sma(values: &[f64], window: usize) -> Vec<f64> {
    if window == 0 || window > values.len() {
        return vec![];
    }
    values
        .windows(window)
        .map(|w| w.iter().sum::<f64>() / window as f64)
        .collect()
}

/// Exponential Moving Average: ให้น้ำหนักค่าล่าสุดมากกว่า
/// alpha ∈ (0, 1): alpha สูง = ตอบสนองเร็ว, alpha ต่ำ = smooth กว่า
pub fn ema(values: &[f64], alpha: f64) -> Vec<f64> {
    if values.is_empty() { return vec![]; }
    let mut result = Vec::with_capacity(values.len());
    result.push(values[0]);
    for i in 1..values.len() {
        let prev = result[i - 1];
        result.push(alpha * values[i] + (1.0 - alpha) * prev);
    }
    result
}

/// Weighted Moving Average: น้ำหนัก = [1, 2, ..., window]
/// ค่าล่าสุดได้น้ำหนักมากที่สุด
pub fn wma(values: &[f64], window: usize) -> Vec<f64> {
    if window == 0 || window > values.len() { return vec![]; }
    let weights: Vec<f64> = (1..=window).map(|i| i as f64).collect();
    let weight_sum: f64 = weights.iter().sum();
    values
        .windows(window)
        .map(|w| {
            w.iter()
                .zip(weights.iter())
                .map(|(v, wt)| v * wt)
                .sum::<f64>()
                / weight_sum
        })
        .collect()
}

/// Lag features: สร้าง column y_{t-k} สำหรับ k = 1..=max_lag
pub fn lag_features(values: &[f64], max_lag: usize) -> Vec<Vec<f64>> {
    (1..=max_lag)
        .map(|lag| {
            let padding = vec![f64::NAN; lag];
            let lagged: Vec<f64> = values[..values.len() - lag].to_vec();
            [padding, lagged].concat()
        })
        .collect()
}
```

**ความแตกต่างระหว่าง SMA, EMA, WMA:**

| | SMA | EMA | WMA |
|---|---|---|---|
| น้ำหนัก | เท่ากันทุก lag | ลดแบบ exponential | เพิ่มแบบ linear |
| ตอบสนองต่อ spike | ช้า | ปานกลาง | ค่อนข้างเร็ว |
| use case | baseline smoothing | trading signals | short-term trend |

**ตัวอย่างการใช้ lag features สำหรับ AR:**

```rust
fn main() {
    let values = vec![1.0, 2.0, 4.0, 7.0, 11.0, 16.0, 22.0];
    let sma3 = sma(&values, 3);
    let ema3 = ema(&values, 0.3);
    let wma3 = wma(&values, 3);

    println!("SMA(3): {:?}", sma3);
    // [2.333, 4.333, 7.333, 11.333, 16.333]

    println!("EMA(α=0.3) ค่าสุดท้าย: {:.3}", ema3.last().unwrap());
    // EMA(α=0.3) ค่าสุดท้าย: 8.704

    let lags = lag_features(&values, 2);
    println!("Lag-1: {:?}", lags[0]);
    // [NaN, 1.0, 2.0, 4.0, 7.0, 11.0, 16.0]
    println!("Lag-2: {:?}", lags[1]);
    // [NaN, NaN, 1.0, 2.0, 4.0, 7.0, 11.0]
}
```

---

### ขั้นที่ 3: Differencing เพื่อสร้าง Stationarity

ARIMA model ต้องการ stationary series (mean และ variance คงที่) แต่ข้อมูลจริงมักมี trend จึงต้องทำ **differencing**:

```
ΔY_t = Y_t - Y_{t-1}          (1st order diff)
Δ²Y_t = ΔY_t - ΔY_{t-1}       (2nd order diff)
```

```rust
/// d-th order differencing
pub fn difference(values: &[f64], d: usize) -> Vec<f64> {
    let mut result = values.to_vec();
    for _ in 0..d {
        result = result.windows(2).map(|w| w[1] - w[0]).collect();
    }
    result
}

/// Inverse differencing: คืนค่า original scale จาก diff
/// ต้องการ initial values (d ค่าแรกก่อน diff)
pub fn inverse_difference(diff: &[f64], initial: &[f64], d: usize) -> Vec<f64> {
    let mut result = diff.to_vec();
    for i in (0..d).rev() {
        let start = initial[i];
        let mut reconstructed = vec![start];
        for &delta in &result {
            let last = *reconstructed.last().unwrap();
            reconstructed.push(last + delta);
        }
        result = reconstructed;
    }
    result
}
```

**ทำไม inverse_difference ถึงจำเป็น?**

เมื่อ fit model บน differenced series แล้ว forecast ที่ได้เป็น "ความเปลี่ยนแปลง" ไม่ใช่ค่าจริง ต้อง integrate กลับ:

```rust
fn main() {
    let values = vec![10.0, 13.0, 17.0, 22.0, 28.0, 35.0];
    // trend เพิ่มขึ้นเรื่อย ๆ → non-stationary

    let diff1 = difference(&values, 1);
    println!("diff(1): {:?}", diff1);
    // [3.0, 4.0, 5.0, 6.0, 7.0] — ยังมี trend

    let diff2 = difference(&values, 2);
    println!("diff(2): {:?}", diff2);
    // [1.0, 1.0, 1.0, 1.0] — constant! → stationary

    // Inverse: คืนค่า original จาก diff1
    let recovered = inverse_difference(&diff1, &[values[0]], 1);
    println!("recovered: {:?}", recovered);
    // [10.0, 13.0, 17.0, 22.0, 28.0, 35.0]
}
```

**กฎในการเลือก d:**
- ข้อมูลที่มี linear trend → d=1 มักพอ
- ข้อมูลที่มี quadratic trend (acceleration) → d=2
- ข้อมูลที่มี seasonal pattern → พิจารณา seasonal differencing ด้วย

---

### ขั้นที่ 4: AR(p) Model ด้วย OLS

**Autoregressive Model AR(p)** คาดการณ์ค่าปัจจุบันจาก p ค่าก่อนหน้า:

```
Y_t = φ₁Y_{t-1} + φ₂Y_{t-2} + ... + φₚY_{t-p} + c + ε_t
```

Fit ด้วย OLS เหมือน linear regression:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ARModel {
    pub p: usize,
    pub coefficients: Vec<f64>, // [φ₁, φ₂, ..., φₚ, intercept]
}

impl ARModel {
    pub fn fit(values: &[f64], p: usize) -> Result<Self, String> {
        if values.len() <= p {
            return Err(format!("ต้องการข้อมูลอย่างน้อย {} จุด", p + 1));
        }
        let n = values.len() - p;
        // สร้าง design matrix X (n×(p+1)) และ y (n×1)
        let mut x_mat = vec![vec![0.0f64; p + 1]; n];
        let mut y_vec = vec![0.0f64; n];
        for i in 0..n {
            for j in 0..p {
                x_mat[i][j] = values[i + p - 1 - j]; // lag j+1
            }
            x_mat[i][p] = 1.0; // intercept term
            y_vec[i] = values[i + p]; // target
        }
        // β = (X'X)⁻¹X'y
        let xtx = mat_mul_xt_x(&x_mat, p + 1);
        let xty = mat_mul_xt_y(&x_mat, &y_vec, p + 1);
        let xtx_inv = mat_inv(&xtx, p + 1)?;
        let beta = mat_vec_mul(&xtx_inv, &xty, p + 1);
        Ok(ARModel { p, coefficients: beta })
    }

    /// Forecast h steps ahead (recursive)
    pub fn forecast(&self, history: &[f64], h: usize) -> Vec<f64> {
        let mut extended = history.to_vec();
        let mut forecasts = Vec::with_capacity(h);
        for _ in 0..h {
            let n = extended.len();
            let mut pred = self.coefficients[self.p]; // intercept
            for j in 0..self.p {
                pred += self.coefficients[j] * extended[n - 1 - j];
            }
            forecasts.push(pred);
            extended.push(pred); // ใช้ forecast เป็น input ของ step ถัดไป
        }
        forecasts
    }
}
```

**Design matrix สำหรับ AR(2) เมื่อ p=2:**

```
X = [Y_{t-1}  Y_{t-2}  1]      y = [Y_t]
    [Y_{t}    Y_{t-1}  1]          [Y_{t+1}]
    [Y_{t+1}  Y_{t}    1]          [Y_{t+2}]
    [...]                          [...]
```

**ตัวอย่างการ fit AR(1) กับ process ที่รู้ค่า coefficient จริง:**

```rust
fn main() {
    // สร้าง AR(1) process: Y_t = 0.8 * Y_{t-1} + ε
    let values: Vec<f64> = {
        let mut v = vec![1.0f64];
        for i in 1..50 {
            v.push(0.8 * v[i - 1]);
        }
        v
    };

    let model = ARModel::fit(&values, 1).unwrap();
    println!("φ₁ = {:.4} (จริง = 0.8)", model.coefficients[0]);
    // φ₁ = 0.8000 (จริง = 0.8)

    let forecast = model.forecast(&values, 3);
    println!("Forecast 3 steps: {:.4}, {:.4}, {:.4}",
             forecast[0], forecast[1], forecast[2]);
}
```

**การเลือก p:**
- ดู Partial Autocorrelation Function (PACF) — lag ที่ PACF ตัด off อย่างชัดเจน
- ใช้ Information Criteria: AIC = 2k - 2ln(L), BIC = k·ln(n) - 2ln(L)
- p เล็กที่สุดที่ residuals ดูเป็น white noise

---

### ขั้นที่ 5: MA(q) Model — Moving Average of Errors

**MA(q) Model** แตกต่างจาก SMA/EMA โดยสิ้นเชิง — นี่คือการ model ค่าปัจจุบันจาก **error terms** ในอดีต:

```
Y_t = μ + ε_t + θ₁ε_{t-1} + θ₂ε_{t-2} + ... + θ_qε_{t-q}
```

ปัญหาคือ error terms ε ไม่ observable โดยตรง จึงต้องใช้ **Hannan-Rissanen approximation**:
1. Fit high-order AR เพื่อประมาณ residuals
2. ใช้ lagged residuals เป็น regressors สำหรับ MA

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MAModel {
    pub q: usize,
    pub theta: Vec<f64>,  // [θ₁, θ₂, ..., θ_q]
    pub mu: f64,          // mean ของ series
    pub residuals: Vec<f64>,
}

impl MAModel {
    pub fn fit(values: &[f64], q: usize) -> Result<Self, String> {
        if values.len() <= q {
            return Err(format!("ต้องการข้อมูลอย่างน้อย {} จุด", q + 1));
        }
        let mu = values.iter().sum::<f64>() / values.len() as f64;
        let demeaned: Vec<f64> = values.iter().map(|v| v - mu).collect();

        // Step 1: fit high-order AR เพื่อ estimate residuals
        let ar_order = (2 * q).min(values.len() / 4).max(q + 1);
        let ar = ARModel::fit(&demeaned, ar_order.min(values.len() - 1))?;
        let mut residuals = vec![0.0f64; values.len()];
        for i in ar.p..values.len() {
            let mut pred = ar.coefficients[ar.p];
            for j in 0..ar.p {
                pred += ar.coefficients[j] * demeaned[i - 1 - j];
            }
            residuals[i] = demeaned[i] - pred;
        }

        // Step 2: regress demeaned บน lagged residuals
        let n = values.len() - q;
        let mut x_mat = vec![vec![0.0f64; q]; n];
        let mut y_vec = vec![0.0f64; n];
        for i in 0..n {
            for j in 0..q {
                x_mat[i][j] = residuals[i + q - 1 - j];
            }
            y_vec[i] = demeaned[i + q];
        }
        let xtx = mat_mul_xt_x(&x_mat, q);
        let xty = mat_mul_xt_y(&x_mat, &y_vec, q);
        let xtx_inv = mat_inv(&xtx, q)?;
        let theta = mat_vec_mul(&xtx_inv, &xty, q);

        Ok(MAModel { q, theta, mu, residuals })
    }

    /// Forecast: errors ในอนาคต = 0 (expected value)
    pub fn forecast(&self, h: usize) -> Vec<f64> {
        let n = self.residuals.len();
        let mut forecasts = Vec::with_capacity(h);
        for step in 0..h {
            let mut pred = self.mu;
            if step == 0 {
                for j in 0..self.q {
                    let idx = n as isize - 1 - j as isize;
                    if idx >= 0 {
                        pred += self.theta[j] * self.residuals[idx as usize];
                    }
                }
            }
            // step > 0: future errors = 0 → pred = mu
            forecasts.push(pred);
        }
        forecasts
    }
}
```

**ข้อสังเกต:** MA(q) forecast ทุก step ที่เกิน q จะเท่ากับ μ เสมอ เพราะ error terms ในอนาคตมีค่าคาดหวัง = 0 ซึ่งต่างจาก AR ที่ forecast converge ช้ากว่า

---

### ขั้นที่ 6: Exponential Smoothing — SES, Holt, Holt-Winters

Exponential smoothing เป็น approach ที่ใช้กว้างมากในการพยากรณ์ มีความ interpretable สูง และ parameter ไม่เยอะ

#### Simple Exponential Smoothing (SES)

เหมาะกับข้อมูลที่ไม่มี trend และ seasonality:

```
S_t = α·Y_t + (1-α)·S_{t-1}
```

```rust
pub fn simple_exponential_smoothing(values: &[f64], alpha: f64) -> Vec<f64> {
    // ใช้ EMA เดียวกัน (เป็น alias)
    ema(values, alpha)
}
```

#### Holt's Linear Trend Method

รองรับ linear trend โดยเพิ่ม trend component β:

```
Level:  L_t = α·Y_t + (1-α)·(L_{t-1} + T_{t-1})
Trend:  T_t = β·(L_t - L_{t-1}) + (1-β)·T_{t-1}
Forecast: Ŷ_{t+h} = L_t + h·T_t
```

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HoltLinear {
    pub alpha: f64,
    pub beta: f64,
    pub level: Vec<f64>,
    pub trend: Vec<f64>,
}

impl HoltLinear {
    pub fn fit(values: &[f64], alpha: f64, beta: f64) -> Self {
        let n = values.len();
        let mut level = vec![0.0f64; n];
        let mut trend = vec![0.0f64; n];
        // Initialize: level = first value, trend = difference of first two
        level[0] = values[0];
        trend[0] = if n > 1 { values[1] - values[0] } else { 0.0 };
        for i in 1..n {
            let prev_l = level[i - 1];
            let prev_t = trend[i - 1];
            level[i] = alpha * values[i] + (1.0 - alpha) * (prev_l + prev_t);
            trend[i] = beta * (level[i] - prev_l) + (1.0 - beta) * prev_t;
        }
        HoltLinear { alpha, beta, level, trend }
    }

    pub fn forecast(&self, h: usize) -> Vec<f64> {
        let n = self.level.len();
        let l = self.level[n - 1];
        let t = self.trend[n - 1];
        (1..=h).map(|k| l + k as f64 * t).collect()
    }
}
```

#### Holt-Winters Triple Exponential Smoothing

รองรับ trend และ seasonality พร้อมกัน — เหมาะกับข้อมูล seasonal เช่น ยอดขายรายเดือน:

```
Level:    L_t = α·(Y_t - S_{t-m}) + (1-α)·(L_{t-1} + T_{t-1})
Trend:    T_t = β·(L_t - L_{t-1}) + (1-β)·T_{t-1}
Seasonal: S_t = γ·(Y_t - L_t) + (1-γ)·S_{t-m}
Forecast: Ŷ_{t+h} = L_t + h·T_t + S_{t-m+((h-1) mod m)+1}
```

โดย m = season length (เช่น 12 สำหรับรายเดือน)

**การ initialize Holt-Winters:**

```rust
// Initial level = average of first season
level[0] = values[..season_len].iter().sum::<f64>() / season_len as f64;

// Initial trend = slope between first and second season average
let second_avg = values[season_len..2*season_len].iter().sum::<f64>()
                 / season_len as f64;
trend[0] = (second_avg - level[0]) / season_len as f64;

// Initial seasonal = deviation from initial level
for i in 0..season_len {
    seasonal[i] = values[i] - level[0];
}
```

**ตัวอย่างการ fit Holt-Winters กับข้อมูล seasonal:**

```rust
fn main() {
    // ข้อมูล 24 เดือน: trend + sine seasonality
    let values: Vec<f64> = (0..24).map(|i| {
        (i as f64) + 5.0 * (2.0 * std::f64::consts::PI * i as f64 / 12.0).sin()
    }).collect();

    let model = HoltWinters::fit(&values, 0.3, 0.1, 0.2, 12).unwrap();
    let forecast = model.forecast(6);
    println!("Forecast 6 months: {:?}", forecast);
    // แต่ละค่าจะมี seasonal component แตกต่างกันตามเดือน
}
```

---

### ขั้นที่ 7: STL Decomposition

**STL (Seasonal and Trend decomposition using Loess)** แยก time series ออกเป็น 3 ส่วน:

```
Y_t = T_t + S_t + R_t
(Trend + Seasonal + Residual)
```

การ implement simplified STL ใช้ centered moving average สำหรับ trend และค่าเฉลี่ยตาม phase สำหรับ seasonal:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct STLDecomposition {
    pub trend: Vec<f64>,
    pub seasonal: Vec<f64>,
    pub residual: Vec<f64>,
}

impl STLDecomposition {
    pub fn decompose(values: &[f64], period: usize) -> Result<Self, String> {
        let n = values.len();
        if n < period * 2 {
            return Err(format!("ต้องการข้อมูลอย่างน้อย {} จุด", period * 2));
        }

        // Step 1: ประมาณ trend ด้วย centered moving average
        let trend = centered_moving_average(values, period);

        // Step 2: detrend = original - trend
        let detrended: Vec<f64> = values.iter()
            .zip(trend.iter())
            .map(|(v, t)| v - t)
            .collect();

        // Step 3: seasonal component = average per phase
        let mut phase_sums = vec![0.0f64; period];
        let mut phase_counts = vec![0usize; period];
        for (i, &v) in detrended.iter().enumerate() {
            if v.is_finite() {
                phase_sums[i % period] += v;
                phase_counts[i % period] += 1;
            }
        }
        let seasonal_avg: Vec<f64> = phase_sums.iter()
            .zip(phase_counts.iter())
            .map(|(s, c)| if *c > 0 { s / *c as f64 } else { 0.0 })
            .collect();
        let seasonal: Vec<f64> = (0..n).map(|i| seasonal_avg[i % period]).collect();

        // Step 4: residual = original - trend - seasonal
        let residual: Vec<f64> = values.iter()
            .zip(trend.iter())
            .zip(seasonal.iter())
            .map(|((v, t), s)| v - t - s)
            .collect();

        Ok(STLDecomposition { trend, seasonal, residual })
    }
}

fn centered_moving_average(values: &[f64], window: usize) -> Vec<f64> {
    let n = values.len();
    let half = window / 2;
    let mut result = vec![f64::NAN; n];
    // คำนวณ CMA ตรงกลาง (สำหรับ position ที่มีข้อมูลครบ)
    for i in half..n - half {
        let sum: f64 = values[i - half..=i + half].iter().sum();
        result[i] = sum / (2 * half + 1) as f64;
    }
    // เติม edge ด้วย nearest valid value
    if half < n {
        let first_valid = result[half];
        for i in 0..half { result[i] = first_valid; }
        let last_valid = result[n - half - 1];
        for i in (n - half)..n { result[i] = last_valid; }
    }
    result
}
```

**ประโยชน์ของ STL:**

1. **Trend analysis**: แยก trend ออกมาเพื่อ forecast แนวโน้มระยะยาว
2. **Seasonality removal**: deseasonalized data สำหรับ short-term forecasting
3. **Anomaly detection**: residual ที่ผิดปกติบ่งชี้ outlier หรือ structural break
4. **Visualization**: เข้าใจว่า pattern ไหนมาจากอะไร

**ตัวอย่าง:**

```rust
fn main() {
    let n = 48;
    let period = 12;
    // data = trend + seasonal (clean)
    let values: Vec<f64> = (0..n).map(|i| {
        (i as f64 * 0.5)   // trend เพิ่มขึ้น
        + 3.0 * (2.0 * std::f64::consts::PI * i as f64 / period as f64).sin()
    }).collect();

    let decomp = STLDecomposition::decompose(&values, period).unwrap();
    println!("Trend ที่ t=24:   {:.3}", decomp.trend[24]);
    println!("Seasonal ที่ t=24: {:.3}", decomp.seasonal[24]);
    println!("Residual ที่ t=24: {:.6}", decomp.residual[24]);
    // Residual ควร ≈ 0 สำหรับข้อมูล clean
}
```

---

### ขั้นที่ 8: Anomaly Detection

Anomaly detection ใน time series มีหลาย approach ขึ้นอยู่กับ context:

#### Z-score Method (Global)

```rust
/// ตรวจจับ anomaly จาก global mean/std
/// threshold = 2.0 → 95% CI, threshold = 3.0 → 99.7% CI
pub fn detect_anomalies_zscore(values: &[f64], threshold: f64) -> Vec<AnomalyResult> {
    let n = values.len() as f64;
    let mean = values.iter().sum::<f64>() / n;
    let std = (values.iter().map(|v| (v - mean).powi(2)).sum::<f64>() / n).sqrt();
    if std == 0.0 { return vec![]; }
    values.iter().enumerate()
        .filter_map(|(i, &v)| {
            let z = (v - mean).abs() / std;
            if z > threshold {
                Some(AnomalyResult {
                    index: i, value: v, score: z,
                    method: "z-score".to_string()
                })
            } else { None }
        })
        .collect()
}
```

#### IQR Method (Robust)

Z-score sensitive ต่อ outliers เพราะ mean/std ถูกกระทบ IQR robust กว่า:

```rust
pub fn detect_anomalies_iqr(values: &[f64], multiplier: f64) -> Vec<AnomalyResult> {
    let mut sorted = values.to_vec();
    sorted.sort_by(|a, b| a.partial_cmp(b).unwrap());
    let n = sorted.len();
    let q1 = sorted[n / 4];
    let q3 = sorted[3 * n / 4];
    let iqr = q3 - q1;
    let lower = q1 - multiplier * iqr;
    let upper = q3 + multiplier * iqr;
    values.iter().enumerate()
        .filter_map(|(i, &v)| {
            if v < lower || v > upper {
                let score = if v < lower {
                    (lower - v) / iqr
                } else {
                    (v - upper) / iqr
                };
                Some(AnomalyResult {
                    index: i, value: v, score,
                    method: "iqr".to_string()
                })
            } else { None }
        })
        .collect()
}
```

#### Rolling Z-score (Local Context)

เหมาะกับข้อมูล non-stationary ที่ mean/variance เปลี่ยนแปลงตามเวลา:

```rust
pub fn detect_anomalies_rolling(
    values: &[f64], window: usize, threshold: f64
) -> Vec<AnomalyResult> {
    let mut results = vec![];
    for i in window..values.len() {
        let win = &values[i - window..i]; // window ก่อนหน้า (ไม่รวม i)
        let m = win.iter().sum::<f64>() / window as f64;
        let s = (win.iter().map(|v| (v - m).powi(2)).sum::<f64>()
                 / window as f64).sqrt();
        if s > 0.0 {
            let z = (values[i] - m).abs() / s;
            if z > threshold {
                results.push(AnomalyResult {
                    index: i, value: values[i], score: z,
                    method: "rolling-z".to_string()
                });
            }
        }
    }
    results
}
```

**เมื่อใช้วิธีไหน:**

| วิธี | เหมาะกับ | ข้อระวัง |
|---|---|---|
| Z-score | Stationary series, น้อย outliers | ถูกกระทบถ้ามี outlier เยอะ |
| IQR | Non-normal distribution, robust | ไม่คำนึงถึงลำดับเวลา |
| Rolling Z | Non-stationary, streaming data | ต้องเลือก window ให้เหมาะสม |

---

### ขั้นที่ 9: Forecast Evaluation และ Walk-Forward Validation

ข้อผิดพลาดที่พบบ่อยที่สุดใน time series evaluation คือ **data leakage** — ใช้ข้อมูลอนาคตในการประเมินโมเดล Walk-forward validation แก้ปัญหานี้:

```rust
pub fn mae(actual: &[f64], forecast: &[f64]) -> f64 {
    assert_eq!(actual.len(), forecast.len());
    actual.iter().zip(forecast.iter())
        .map(|(a, f)| (a - f).abs())
        .sum::<f64>() / actual.len() as f64
}

pub fn rmse(actual: &[f64], forecast: &[f64]) -> f64 {
    assert_eq!(actual.len(), forecast.len());
    let mse = actual.iter().zip(forecast.iter())
        .map(|(a, f)| (a - f).powi(2))
        .sum::<f64>() / actual.len() as f64;
    mse.sqrt()
}

pub fn mape(actual: &[f64], forecast: &[f64]) -> f64 {
    assert_eq!(actual.len(), forecast.len());
    let valid: Vec<_> = actual.iter().zip(forecast.iter())
        .filter(|(a, _)| **a != 0.0)
        .collect();
    if valid.is_empty() { return f64::NAN; }
    valid.iter()
        .map(|(a, f)| (*a - *f).abs() / a.abs())
        .sum::<f64>() / valid.len() as f64 * 100.0
}

/// Walk-forward validation: train ขยายทีละ 1 step
/// ทุก forecast ใช้เฉพาะข้อมูลที่ "available at that time"
pub fn walk_forward_validation(
    values: &[f64],
    train_size: usize,
    ar_p: usize,
) -> Result<(f64, f64, f64), String> {
    let n = values.len();
    if train_size >= n {
        return Err("train_size ต้องน้อยกว่า n".to_string());
    }
    let mut actuals = vec![];
    let mut forecasts = vec![];
    for i in train_size..n {
        let train = &values[..i];
        // กำหนด p ให้ไม่เกิน train.len() / 2
        let p = ar_p.min(train.len() / 2).max(1);
        let model = ARModel::fit(train, p)?;
        let pred = model.forecast(train, 1);
        actuals.push(values[i]);
        forecasts.push(pred[0]);
    }
    Ok((
        mae(&actuals, &forecasts),
        rmse(&actuals, &forecasts),
        mape(&actuals, &forecasts),
    ))
}
```

**ความแตกต่างของ metrics:**

| Metric | สูตร | ข้อดี | ข้อเสีย |
|---|---|---|---|
| MAE | mean(|actual - forecast|) | interpretable, robust | ไม่ penalize ขนาด error |
| RMSE | sqrt(mean((actual-forecast)²)) | penalize large errors | sensitive ต่อ outliers |
| MAPE | mean(|error/actual|)×100% | scale-independent (%) | ปัญหาเมื่อ actual ≈ 0 |

**Visualization ของ walk-forward:**

```
Time:   1  2  3  4  5  6  7  8  9  10
Train:  ████████████████
Test:                   ↑  ↑  ↑  ↑  ↑  (แต่ละ step expand train เพิ่ม 1)

Fold 1: Train=[1..5], Forecast=6
Fold 2: Train=[1..6], Forecast=7
Fold 3: Train=[1..7], Forecast=8
...
```

---

## การทดสอบ (Testing)

โปรเจคมี unit tests ครอบคลุมทุก algorithm:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // --- TimeSeries ---
    #[test]
    fn test_time_series_creation() {
        let ts = TimeSeries::new(vec![0, 1, 2], vec![1.0, 2.0, 3.0]).unwrap();
        assert_eq!(ts.len(), 3);
        assert!((ts.mean() - 2.0).abs() < 1e-10);
    }

    #[test]
    fn test_time_series_mismatched_lengths() {
        let result = TimeSeries::new(vec![0, 1], vec![1.0, 2.0, 3.0]);
        assert!(result.is_err());
    }

    #[test]
    fn test_time_series_empty() {
        let result = TimeSeries::new(vec![], vec![]);
        assert!(result.is_err());
    }

    #[test]
    fn test_time_series_variance() {
        let ts = TimeSeries::new(vec![0, 1, 2, 3],
                                 vec![2.0, 4.0, 4.0, 6.0]).unwrap();
        // mean=4, var=((4+0+0+4)/4)=2
        assert!((ts.variance() - 2.0).abs() < 1e-10);
    }

    // --- SMA ---
    #[test]
    fn test_sma_basic() {
        let values = vec![1.0, 2.0, 3.0, 4.0, 5.0];
        let result = sma(&values, 3);
        assert_eq!(result.len(), 3);
        assert!((result[0] - 2.0).abs() < 1e-10); // (1+2+3)/3
        assert!((result[1] - 3.0).abs() < 1e-10); // (2+3+4)/3
        assert!((result[2] - 4.0).abs() < 1e-10); // (3+4+5)/3
    }

    // --- EMA ---
    #[test]
    fn test_ema_converges_to_constant() {
        let values = vec![5.0; 100];
        let result = ema(&values, 0.5);
        assert!((result[99] - 5.0).abs() < 1e-10);
    }

    // --- Differencing ---
    #[test]
    fn test_difference_second_order() {
        let values = vec![1.0, 3.0, 6.0, 10.0];
        let diff2 = difference(&values, 2);
        // 1st: [2,3,4], 2nd: [1,1]
        assert_eq!(diff2, vec![1.0, 1.0]);
    }

    #[test]
    fn test_inverse_difference_roundtrip() {
        let values = vec![1.0, 3.0, 6.0, 10.0, 15.0];
        let diff = difference(&values, 1);
        let recovered = inverse_difference(&diff, &[values[0]], 1);
        for (a, b) in values.iter().zip(recovered.iter()) {
            assert!((a - b).abs() < 1e-10);
        }
    }

    // --- AR ---
    #[test]
    fn test_ar_fit_and_forecast() {
        let values: Vec<f64> = {
            let mut v = vec![1.0f64];
            for i in 1..50 { v.push(0.8 * v[i - 1]); }
            v
        };
        let model = ARModel::fit(&values, 1).unwrap();
        assert!((model.coefficients[0] - 0.8).abs() < 0.01);
    }

    // ... (test อื่น ๆ ทั้งหมด 28 tests)
}
```

**Real `cargo test` output:**

```
running 28 tests
test tests::test_difference_first_order ... ok
test tests::test_difference_second_order ... ok
test tests::test_ema_converges_to_constant ... ok
test tests::test_ema_length ... ok
test tests::test_holt_forecast_length ... ok
test tests::test_holt_linear_trending ... ok
test tests::test_holt_winters_fit ... ok
test tests::test_ar_fit_and_forecast ... ok
test tests::test_ar_forecast_length ... ok
test tests::test_iqr_detects_outlier ... ok
test tests::test_inverse_difference_roundtrip ... ok
test tests::test_lag_features ... ok
test tests::test_mae_perfect ... ok
test tests::test_ma_fit_and_forecast ... ok
test tests::test_mape_basic ... ok
test tests::test_rmse_basic ... ok
test tests::test_rolling_anomaly ... ok
test tests::test_sma_window_larger_than_data ... ok
test tests::test_stl_decomposition ... ok
test tests::test_stl_needs_enough_data ... ok
test tests::test_sma_basic ... ok
test tests::test_time_series_empty ... ok
test tests::test_time_series_mismatched_lengths ... ok
test tests::test_time_series_creation ... ok
test tests::test_time_series_variance ... ok
test tests::test_zscore_detects_spike ... ok
test tests::test_walk_forward_validation ... ok
test tests::test_wma_basic ... ok

test result: ok. 28 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### ข้อผิดพลาดที่ 1: ใช้ข้อมูลอนาคตใน Evaluation (Data Leakage)

นี่คือ bug ที่ร้ายแรงที่สุดใน time series — ทำให้ metrics ดูดีเกินจริง

**ผิด:**
```rust
// ใช้ train/test split ธรรมดา (shuffle data ก่อน)
let model = ARModel::fit(&all_data, p).unwrap();
let test_pred = model.forecast(&all_data, test_size);
// BUG: model เห็น future data ตอน training แล้ว
```

**ถูก:**
```rust
// Walk-forward validation: train ขยายทีละ step
for i in train_size..n {
    let train = &values[..i];  // ใช้แค่ข้อมูลถึงเวลา i
    let model = ARModel::fit(train, p)?;
    let pred = model.forecast(train, 1);  // forecast 1 step ahead
    // ...
}
```

**ผลกระทบ:** Data leakage ทำให้ AR(2) บน linear data ดู RMSE = 0 ทั้งที่จริง ๆ อาจ RMSE = 5+

---

### ข้อผิดพลาดที่ 2: ลืม Differencing บน Non-Stationary Series

AR model สมมติว่าข้อมูล stationary ถ้าใช้กับ non-stationary series จะได้ spurious forecast:

**ผิด:**
```rust
// ข้อมูลมี trend แต่ fit AR โดยตรง
let prices = load_stock_prices(); // trend upward
let model = ARModel::fit(&prices, 2).unwrap();
let forecast = model.forecast(&prices, 10);
// BUG: forecast จะ extrapolate trend ผิดมาก
```

**ถูก:**
```rust
let ratio = ts.stationarity_ratio();
let (d, series_to_fit) = if ratio > 2.0 || ratio < 0.5 {
    // non-stationary → difference
    let diff = difference(&prices, 1);
    (1, diff)
} else {
    (0, prices.clone())
};

let model = ARModel::fit(&series_to_fit, 2).unwrap();
let diff_forecast = model.forecast(&series_to_fit, 10);

// Invert differencing to get original scale
let final_forecast = if d > 0 {
    // ต้องรู้ last value ก่อน difference
    inverse_difference(&diff_forecast, &[*prices.last().unwrap()], d)
} else {
    diff_forecast
};
```

**กฎ:** ถ้า `stationarity_ratio` > 2.0 หรือ < 0.5 ให้ difference ก่อนเสมอ

---

### ข้อผิดพลาดที่ 3: Singular Matrix ใน OLS

เกิดเมื่อ design matrix มี columns ที่ linear dependent เช่น ข้อมูลที่เป็น perfect linear trend:

**สาเหตุ:**
```rust
let values = vec![0.0, 1.0, 2.0, 3.0, 4.0]; // perfect linear
// AR(2) design matrix:
// [1, 0, 1]   → row ที่ 3 = row ที่ 1 + row ที่ 2 (linear dependent!)
// [2, 1, 1]
// [3, 2, 1]
let model = ARModel::fit(&values, 2); // Error: "Matrix is singular"
```

**วิธีแก้:**

1. **เพิ่ม noise** ให้ข้อมูล
2. **Regularization** — เพิ่ม λI ใน X'X ก่อน invert (Ridge regression):
   ```rust
   // เพิ่ม regularization term
   let lambda = 1e-6;
   for i in 0..n_cols {
       xtx[i][i] += lambda;
   }
   ```
3. **Reduce p** — ลด AR order ลง
4. **Difference ข้อมูลก่อน** — linear series หลัง diff(1) จะเป็น constant ไม่ singular

---

### ข้อผิดพลาดที่ 4: เลือก season_len ผิดใน Holt-Winters / STL

**ผิด:**
```rust
// ข้อมูลรายวัน 2 ปี (730 วัน) ใช้ period ผิด
let decomp = STLDecomposition::decompose(&daily_data, 7).unwrap();
// สมมติ period=7 (weekly) แต่ข้อมูลมี yearly seasonality (period=365)
// → seasonal component จะจับได้แค่ weekly pattern ไม่ใช่ annual
```

**ถูก:**
```rust
// วิเคราะห์ ACF (Autocorrelation Function) เพื่อหา period ที่แท้จริง
// หรือถ้ารู้จาก domain knowledge
let period = match data_frequency {
    "hourly" => 24,      // daily seasonality
    "daily" => 7,        // weekly (หรือ 365 สำหรับ annual)
    "monthly" => 12,     // annual
    "quarterly" => 4,    // annual
    _ => 1,
};
let decomp = STLDecomposition::decompose(&data, period).unwrap();
```

**ผลกระทบ:** period ผิดทำให้ seasonal component ไม่ถูกต้อง residual ใหญ่ผิดปกติ และ forecast มีคุณภาพต่ำ

---

### ข้อผิดพลาดที่ 5: ใช้ MAPE กับข้อมูลที่มีค่าใกล้ศูนย์

```rust
let actual = vec![0.01, 100.0, 200.0];
let forecast = vec![0.5, 98.0, 205.0];
let mape_val = mape(&actual, &forecast);
// MAPE = (|0.01-0.5|/0.01 + |100-98|/100 + |200-205|/200) / 3 * 100
// = (49 + 0.02 + 0.025) / 3 * 100 = 1634%
// ค่าเดียวที่ ≈ 0 ทำให้ MAPE พุ่งสูงมากผิดปกติ
```

**วิธีแก้:** ใช้ MASE (Mean Absolute Scaled Error) หรือ sMAPE แทน หรือ filter ค่าที่ `|actual| < epsilon`:

```rust
pub fn smape(actual: &[f64], forecast: &[f64]) -> f64 {
    let n = actual.len() as f64;
    actual.iter().zip(forecast.iter())
        .map(|(a, f)| {
            let denom = (a.abs() + f.abs()) / 2.0;
            if denom < 1e-10 { 0.0 } else { (a - f).abs() / denom }
        })
        .sum::<f64>() / n * 100.0
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# Binary อยู่ที่ target/release/time-series

# รัน demo
./target/release/time-series
```

### Serialize โมเดลสำหรับ Production API

```rust
use std::fs;

fn save_model(model: &ARModel, path: &str) -> Result<(), Box<dyn std::error::Error>> {
    let json = serde_json::to_string_pretty(model)?;
    fs::write(path, json)?;
    Ok(())
}

fn load_model(path: &str) -> Result<ARModel, Box<dyn std::error::Error>> {
    let json = fs::read_to_string(path)?;
    let model: ARModel = serde_json::from_str(&json)?;
    Ok(model)
}

fn main() {
    // Train และบันทึก
    let model = ARModel::fit(&train_data, 3).unwrap();
    save_model(&model, "ar_model.json").unwrap();

    // Load กลับมาใช้
    let loaded = load_model("ar_model.json").unwrap();
    let forecast = loaded.forecast(&recent_data, 5);
}
```

### Benchmark ด้วย Criterion

เพิ่มใน `Cargo.toml`:

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "forecasting"
harness = false
```

```rust
// benches/forecasting.rs
use criterion::{criterion_group, criterion_main, Criterion};
use time_series::*;

fn bench_ar_fit(c: &mut Criterion) {
    let values: Vec<f64> = (0..1000).map(|i| (i as f64 * 0.01).sin()).collect();
    c.bench_function("AR(5) fit 1000 points", |b| {
        b.iter(|| ARModel::fit(&values, 5).unwrap())
    });
}

criterion_group!(benches, bench_ar_fit);
criterion_main!(benches);
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Implement ARIMA(p,d,q) แบบครบวงจร

ในตอนนี้ AR(p) และ MA(q) แยกกัน จงเขียน `ARIMAModel` ที่รวมทั้ง 3 component:

```rust
pub struct ARIMAModel {
    pub p: usize,
    pub d: usize,
    pub q: usize,
    // hint: fit บน differenced series แล้ว inverse transform forecast
}

impl ARIMAModel {
    pub fn fit(values: &[f64], p: usize, d: usize, q: usize)
        -> Result<Self, String> {
        // 1. difference ข้อมูล d ครั้ง
        // 2. fit AR(p) บน differenced series
        // 3. คำนวณ residuals แล้ว fit MA(q)
        // 4. combined coefficients
        todo!()
    }

    pub fn forecast(&self, history: &[f64], h: usize)
        -> Vec<f64> {
        // 1. forecast บน stationary series
        // 2. inverse difference h ครั้ง
        todo!()
    }
}
```

**Hint:** ตอน inverse diff ต้องเก็บ "seed values" (ค่า original ก่อน diff) ไว้ใน struct

---

### แบบฝึกหัดที่ 2: Auto-selection ของ ARIMA Order ด้วย AIC/BIC

เขียน grid search เพื่อหา (p, d, q) ที่ดีที่สุด:

```rust
pub fn auto_arima(values: &[f64], max_p: usize, max_q: usize)
    -> (usize, usize, usize) {
    // Grid search ทุก combination (p, d, q) โดย:
    // - d: 0 หรือ 1 (ตรวจจากค่า stationarity)
    // - p: 0..=max_p
    // - q: 0..=max_q
    // เลือก combination ที่ให้ AIC ต่ำที่สุด
    // AIC = 2*(p+q+1) - 2*log_likelihood
    // log_likelihood ≈ -n/2 * ln(sigma^2_residual)
    todo!()
}
```

**Tips:**
- ขอบเขตทั่วไป: max_p = max_q = 5
- เพิ่ม Tikhonov regularization เพื่อป้องกัน singular matrix ตอน search

---

### แบบฝึกหัดที่ 3: Autocorrelation Function (ACF) และ PACF

เขียน function ที่คำนวณ ACF และ PACF เพื่อช่วย identify p และ q:

```rust
/// Autocorrelation at lag k
pub fn acf(values: &[f64], max_lag: usize) -> Vec<f64> {
    let n = values.len();
    let mean = values.iter().sum::<f64>() / n as f64;
    let variance = values.iter().map(|v| (v - mean).powi(2)).sum::<f64>() / n as f64;
    (0..=max_lag).map(|lag| {
        if lag == 0 { return 1.0; }
        let cov: f64 = values[..n-lag].iter()
            .zip(values[lag..].iter())
            .map(|(a, b)| (a - mean) * (b - mean))
            .sum::<f64>() / n as f64;
        cov / variance
    }).collect()
}

/// Partial Autocorrelation at lag k (Yule-Walker approximation)
pub fn pacf(values: &[f64], max_lag: usize) -> Vec<f64> {
    // Hint: ใช้ Durbin-Levinson recursion
    todo!()
}
```

**การแปลผล ACF/PACF:**

| Model | ACF | PACF |
|---|---|---|
| AR(p) | Tails off (damped) | Cuts off at lag p |
| MA(q) | Cuts off at lag q | Tails off |
| ARMA | Tails off | Tails off |

---

### แบบฝึกหัดที่ 4: Seasonal ARIMA (SARIMA)

ขยาย ARIMA ให้รองรับ seasonal component ด้วย (P, D, Q)[m]:

```
SARIMA(p,d,q)(P,D,Q)[m]:
φ(B)·Φ(B^m)·(1-B)^d·(1-B^m)^D·Y_t = θ(B)·Θ(B^m)·ε_t
```

โดย:
- `B` คือ backshift operator: `B·Y_t = Y_{t-1}`
- `(1-B)^d` คือ differencing d ครั้ง
- `(1-B^m)^D` คือ seasonal differencing D ครั้ง ด้วย period m

```rust
pub struct SARIMAModel {
    pub p: usize, pub d: usize, pub q: usize,  // non-seasonal
    pub big_p: usize, pub big_d: usize, pub big_q: usize,  // seasonal
    pub m: usize,  // season length
}
```

**Hint:** Seasonal diff: `Y_t - Y_{t-m}` แทนที่จะเป็น `Y_t - Y_{t-1}`

---

### แบบฝึกหัดที่ 5: Real-time Streaming Anomaly Detection

สร้าง detector ที่อัปเดตแบบ online ไม่ต้อง reprocess ข้อมูลทั้งหมด:

```rust
pub struct OnlineAnomalyDetector {
    window: usize,
    threshold: f64,
    buffer: VecDeque<f64>,
    running_mean: f64,
    running_m2: f64,  // สำหรับ Welford's online algorithm
    count: usize,
}

impl OnlineAnomalyDetector {
    pub fn new(window: usize, threshold: f64) -> Self { todo!() }

    /// ประมวลผลค่าใหม่ทีละตัว O(1) per point
    pub fn update(&mut self, value: f64) -> Option<AnomalyResult> {
        // Welford's algorithm สำหรับ running mean/variance
        // ไม่ต้องเก็บ history ทั้งหมด
        todo!()
    }
}
```

---

### แบบฝึกหัดที่ 6: Confidence Intervals สำหรับ AR Forecast

Forecast โดยไม่มี confidence interval ไม่บอกอะไรเกี่ยวกับความไม่แน่นอน เขียน function ที่คำนวณ prediction interval:

```rust
pub struct ForecastWithCI {
    pub point: f64,
    pub lower_95: f64,
    pub upper_95: f64,
    pub lower_80: f64,
    pub upper_80: f64,
}

impl ARModel {
    pub fn forecast_with_ci(
        &self, history: &[f64], h: usize
    ) -> Vec<ForecastWithCI> {
        // 1. คำนวณ residual variance จาก in-sample fit
        // 2. AR(p) h-step forecast error variance:
        //    Var(ε_{t+h}) = σ² * (1 + Σψ_k²) สำหรับ k=1..h-1
        //    โดย ψ coefficients คำนวณจาก φ coefficients
        // 3. 95% CI: forecast ± 1.96 * sqrt(Var)
        todo!()
    }
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **Time Series Forecasting Library** ที่ครอบคลุม algorithm หลักทั้งหมด:

**สิ่งที่สร้างได้:**
- `TimeSeries` struct พร้อม stationarity check
- Moving averages ทั้ง 3 แบบ (SMA, EMA, WMA) และ lag features
- Differencing (d-th order) และ inverse differencing พร้อม roundtrip guarantee
- AR(p) model ด้วย OLS + Gauss-Jordan matrix inversion
- MA(q) model ด้วย Hannan-Rissanen approximation
- Exponential smoothing ทั้ง 3 ระดับ: SES, Holt, Holt-Winters
- STL decomposition: trend, seasonal, residual
- Anomaly detection: Z-score, IQR, rolling Z-score
- Evaluation framework: MAE, RMSE, MAPE, walk-forward validation

**Patterns สำคัญที่ได้เรียน:**
1. **Design matrix pattern**: การแปลง sequence เป็น regression matrix สำหรับ OLS
2. **Iterator chaining**: ใช้ `.windows()`, `.zip()`, `.enumerate()` สำหรับ time series operations
3. **Error propagation**: `Result<T, String>` ใน algorithm chain ที่ยาว
4. **Numerical stability**: ทำไม matrix inversion ถึงต้องระวัง และ regularization ช่วยได้
5. **Validation discipline**: walk-forward validation ป้องกัน data leakage

**เชื่อมโยงกับโปรเจคถัดไป:**

โปรเจค I08: Genetic Algorithm จะนำ optimization technique ไปใช้กับปัญหาที่ gradient descent ทำไม่ได้ — เช่น hyper-parameter tuning สำหรับ Holt-Winters (เลือก α, β, γ ที่ minimize RMSE) หรือ feature selection ใน AR model นี่เป็นการเชื่อม classical statistics เข้ากับ evolutionary computation

---

**โปรเจคก่อนหน้า:** [Project I06: Recommender System](project-i06-recommender.md) | **โปรเจคถัดไป:** [Project I08: Genetic Algorithm](project-i08-genetic-algorithm.md)
