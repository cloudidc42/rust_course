# Project I09: Q-Learning และ Reinforcement Learning

> โมดูล: I — ML/AI | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Reinforcement Learning (RL) Framework** ที่สมบูรณ์แบบ ครอบคลุมตั้งแต่พื้นฐานของ Grid World Environment ไปจนถึง Q-Learning, SARSA และ Double Q-Learning โดยใช้ Rust เพื่อให้ได้ประสิทธิภาพสูงสุดและความปลอดภัยของหน่วยความจำ

**Reinforcement Learning** คือกระบวนทัศน์การเรียนรู้ที่ Agent เรียนรู้วิธีตัดสินใจโดยการโต้ตอบกับ Environment เป้าหมายคือหา Policy ที่ดีที่สุดที่สะสม Reward สูงสุดในระยะยาว แนวคิดนี้เป็นพื้นฐานของระบบ AI ที่น่าตื่นเต้นที่สุดในปัจจุบัน ไม่ว่าจะเป็น AlphaGo ที่เอาชนะแชมป์โลกหมากล้อม, หุ่นยนต์ที่เรียนรู้การเดิน, หรือระบบควบคุมอัตโนมัติในโรงงาน

**Q-Learning** เป็น model-free RL algorithm ที่เรียนรู้ค่า Q(state, action) — ค่าที่บ่งบอกว่า "ถ้าอยู่ใน state นี้แล้วทำ action นี้ จะได้ reward รวมมากแค่ไหน" Algorithm นี้ guaranteed ว่าจะ converge ไปหา optimal policy ภายใต้เงื่อนไขที่กำหนด และเป็นจุดเริ่มต้นที่ดีในการทำความเข้าใจ Deep Q-Network (DQN) ที่ DeepMind ใช้เล่นเกม Atari

**Use cases จริงในโลก production:**
- ระบบแนะนำสินค้า (Recommendation System) ที่ปรับตัวตามพฤติกรรมผู้ใช้
- Robot navigation ในโกดังสินค้า (คล้ายหุ่นยนต์ Amazon)
- Network routing optimization ที่ปรับเส้นทางตาม traffic แบบ real-time
- Resource scheduling ใน cloud computing
- Trading bot ที่เรียนรู้กลยุทธ์การซื้อขาย

เราจะสร้างระบบที่มี Environment trait ที่ยืดหยุ่น, QTable ที่ serialize/deserialize ได้, training loop ที่วัด convergence ได้, และ visualization ของ policy ที่เรียนรู้มา

## สิ่งที่จะได้เรียนรู้

- **Temporal Difference (TD) Learning** — การอัปเดต value estimate ทีละ step โดยไม่รอจบ episode
- **Epsilon-Greedy Exploration** — balance ระหว่าง explore (ลองสิ่งใหม่) และ exploit (ทำสิ่งที่รู้ว่าดี)
- **Bellman Optimality Equation** — Q(s,a) += α × (r + γ × max_Q(s') − Q(s,a))
- **Off-policy vs On-policy** — ความแตกต่างระหว่าง Q-Learning (off) และ SARSA (on)
- **Double Q-Learning** — วิธีลด overestimation bias ด้วย 2 networks
- **Trait-based Environment Design** — pattern ที่นำไปใช้กับ environment ได้หลายประเภท
- **Convergence Tracking** — วัด running average และ success rate เพื่อติดตามการเรียนรู้
- **Policy Extraction & Visualization** — แปลง Q-values เป็น policy และแสดงผลบน grid

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`, `HashSet`), iterators, closures
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics, trait objects
- **Part 41–50**: Lifetimes, advanced traits, `impl Trait`
- **Part 51–60**: Modules, crate ecosystem, `Cargo.toml` dependencies
- **Part 96–105**: Serde serialization/deserialization, JSON format
- **Project I01**: Linear Regression (เพื่อทำความเข้าใจ gradient-based learning)
- **Project I08**: Genetic Algorithm (เพื่อเปรียบเทียบ optimization approaches)

## โครงสร้างโปรเจค (Project Layout)

```
q-learning-rl/
├── src/
│   ├── main.rs          ← demo และ integration
│   └── lib.rs           ← Environment trait, GridWorld, QTable, training
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของ Reinforcement Learning

```
┌─────────────────────────────────────────────────────────────┐
│                     Training Loop                           │
│                                                             │
│   Episode 1, 2, ..., N                                      │
│                                                             │
│   ┌─────────┐   action    ┌─────────────┐                  │
│   │  Agent  │ ──────────► │ Environment │                  │
│   │         │             │  (GridWorld) │                  │
│   │ QTable  │ ◄────────── │             │                  │
│   │ epsilon │  (s', r, d) └─────────────┘                  │
│   └─────────┘                                               │
│        │                                                    │
│        ▼                                                    │
│   Bellman Update:                                           │
│   Q(s,a) += α(r + γ·max_Q(s') − Q(s,a))                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Component Diagram

```
Environment Trait
    │
    └── GridWorld
          ├── rows, cols
          ├── start, goal
          ├── walls: Vec<State>
          └── impl step(), reset()

QTable
    ├── table: HashMap<String, f64>
    ├── get(s, a) → f64
    ├── set(s, a, v)
    ├── max_q(s) → f64
    ├── best_action(s) → Action
    ├── epsilon_greedy(s, ε) → Action
    └── update(s, a, α, γ, r, s')   ← Bellman update

Training Functions
    ├── train_q_learning()     ← off-policy TD
    ├── train_sarsa()          ← on-policy TD
    └── train_double_q_learning()  ← reduces overestimation

TrainingResult
    ├── q_table: QTable
    ├── episode_stats: Vec<EpisodeStats>
    ├── running_average(window)
    └── success_rate(window)
```

### ทำไมต้องใช้ Rust สำหรับ RL?

ใน Python ที่ใช้กันทั่วไปสำหรับ RL, training loop ที่ทำงาน 1 ล้าน episode อาจใช้เวลาหลายนาที Rust ช่วยให้:

1. **Zero-cost abstractions** — trait `Environment` ไม่มี overhead ตอน runtime
2. **Cache-friendly data structures** — `HashMap` ที่ optimized ใน Rust มี lookup เร็วมาก
3. **No GC pauses** — ไม่มี garbage collector pause ที่จะรบกวน training loop
4. **Safe concurrency** — สามารถ parallelize training ด้วย `rayon` ได้ในภายหลัง

### การออกแบบ QTable Key

ปัญหาสำคัญคือ `HashMap` ใน serde_json ต้องการ key เป็น `String` เราจึงใช้ format `"row,col,action_id"` แทน tuple key:

```
(state=(2,3), action=Right) → key = "2,3,3"
```

วิธีนี้ทำให้ serialize/deserialize ได้โดยตรงโดยไม่ต้องเขียน custom serializer

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: นิยาม Environment Trait และ Types พื้นฐาน

ขั้นแรกเราจะวางโครงสร้างพื้นฐานของระบบ RI ทั้งหมด ออกแบบ `Environment` trait ให้ทั่วไพอที่จะรองรับ environment ประเภทต่าง ๆ ไม่ใช่แค่ Grid World

**Cargo.toml:**

```toml
[package]
name = "q_learning_rl"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "q_learning_rl"
path = "src/main.rs"

[dependencies]
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[lib]
name = "q_learning_rl"
path = "src/lib.rs"
```

เริ่มต้นด้วยการนิยาม types พื้นฐาน:

```rust
use std::collections::HashMap;
use rand::Rng;
use serde::{Serialize, Deserialize};

// State เป็น (row, col) tuple ใน grid
pub type State = (usize, usize);

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum Action {
    Up,
    Down,
    Left,
    Right,
}

impl Action {
    pub fn all() -> Vec<Action> {
        vec![Action::Up, Action::Down, Action::Left, Action::Right]
    }

    pub fn to_arrow(&self) -> &str {
        match self {
            Action::Up    => "↑",
            Action::Down  => "↓",
            Action::Left  => "←",
            Action::Right => "→",
        }
    }
}
```

ต่อมานิยาม `Environment` trait:

```rust
pub trait Environment {
    fn reset(&mut self) -> State;
    fn step(&mut self, action: Action) -> (State, f64, bool);
    fn current_state(&self) -> State;
    fn is_terminal(&self, state: State) -> bool;
}
```

signature ของ `step()` คือ `(Action) -> (State, f64, bool)` ซึ่งเป็นมาตรฐาน RL:
- `State` — next state หลัง action
- `f64` — reward ที่ได้รับ
- `bool` — done flag (true = episode จบ)

ส่วน `reset()` คืน initial state และ reset environment กลับสู่สถานะเริ่มต้น ทำให้ทดสอบได้หลาย episode

**หลักการออกแบบ**: เราใช้ `&mut self` สำหรับทั้ง `reset()` และ `step()` เพราะ environment มีสถานะที่เปลี่ยนได้ (current position ของ agent)

---

### ขั้นที่ 2: สร้าง GridWorld Environment

GridWorld เป็น environment แบบง่ายที่สุดสำหรับทดสอบ RL algorithm มีตาราง grid ขนาด M×N ที่ agent เคลื่อนที่ได้ 4 ทิศทาง มีกำแพง และ goal

```
+---+---+---+---+
| S |   | # | # |   S = Start, G = Goal
+---+---+---+---+   # = Wall
|   | # | # |   |
+---+---+---+---+
|   | # |   |   |
+---+---+---+---+
|   |   |   | G |
+---+---+---+---+
```

```rust
#[derive(Debug, Clone)]
pub struct GridWorld {
    pub rows: usize,
    pub cols: usize,
    pub start: State,
    pub goal: State,
    pub walls: Vec<State>,
    pub current: State,
    pub step_penalty: f64,
    pub goal_reward: f64,
    pub wall_penalty: f64,
}

impl GridWorld {
    pub fn new(rows: usize, cols: usize, start: State, goal: State, walls: Vec<State>) -> Self {
        GridWorld {
            rows, cols, start, goal, walls,
            current: start,
            step_penalty: -0.01,   // ค่าปรับสำหรับแต่ละ step ที่ไม่ถึง goal
            goal_reward: 1.0,      // reward เมื่อถึง goal
            wall_penalty: -0.1,    // ค่าปรับเมื่อชนกำแพงหรือขอบ
        }
    }

    pub fn is_wall(&self, state: State) -> bool {
        self.walls.contains(&state)
    }

    pub fn is_valid(&self, row: i64, col: i64) -> bool {
        row >= 0 && row < self.rows as i64
            && col >= 0 && col < self.cols as i64
    }

    // คืน (next_state, did_move)
    pub fn apply_action(&self, state: State, action: Action) -> (State, bool) {
        let (r, c) = state;
        let (nr, nc) = match action {
            Action::Up    => (r as i64 - 1, c as i64),
            Action::Down  => (r as i64 + 1, c as i64),
            Action::Left  => (r as i64, c as i64 - 1),
            Action::Right => (r as i64, c as i64 + 1),
        };

        if !self.is_valid(nr, nc) {
            return (state, false);   // ชนขอบ → อยู่กับที่
        }

        let next = (nr as usize, nc as usize);
        if self.is_wall(next) {
            return (state, false);   // ชนกำแพง → อยู่กับที่
        }

        (next, true)
    }
}
```

**Reward Design** มีความสำคัญมาก:
- `step_penalty = -0.01` บังคับให้ agent หา path ที่สั้นที่สุด ถ้าไม่มี penalty agent อาจเดินวนไปเรื่อย ๆ ได้ total reward เท่ากัน
- `goal_reward = 1.0` บอก agent ว่าเป้าหมายสำคัญ
- `wall_penalty = -0.1` ทำให้ชนกำแพงน่าเจ็บปวด ช่วยให้ agent เรียนรู้หลีกเลี่ยงกำแพงเร็วขึ้น

จากนั้น implement `Environment` trait:

```rust
impl Environment for GridWorld {
    fn reset(&mut self) -> State {
        self.current = self.start;
        self.current
    }

    fn step(&mut self, action: Action) -> (State, f64, bool) {
        let (next, moved) = self.apply_action(self.current, action);

        let reward = if next == self.goal {
            self.goal_reward
        } else if !moved {
            self.wall_penalty
        } else {
            self.step_penalty
        };

        self.current = next;
        let done = next == self.goal;
        (next, reward, done)
    }

    fn current_state(&self) -> State { self.current }

    fn is_terminal(&self, state: State) -> bool {
        state == self.goal
    }
}
```

ทดสอบเบื้องต้น:

```rust
fn main() {
    let mut env = GridWorld::new(3, 3, (0, 0), (2, 2), vec![(1, 1)]);

    let state = env.reset();
    println!("Start: {:?}", state);         // (0, 0)

    let (next, reward, done) = env.step(Action::Right);
    println!("→ next={:?}, r={:.2}, done={}", next, reward, done);
    // next=(0,1), r=-0.01, done=false

    // ลองชนขอบ
    env.reset();
    let (next, reward, done) = env.step(Action::Up);
    println!("↑ next={:?}, r={:.2}, done={}", next, reward, done);
    // next=(0,0), r=-0.10, done=false  ← ชนขอบ ไม่ขยับ
}
```

**Output:**
```
Start: (0, 0)
→ next=(0, 1), r=-0.01, done=false
↑ next=(0, 0), r=-0.10, done=false
```

---

### ขั้นที่ 3: สร้าง QTable และ Epsilon-Greedy Policy

QTable เก็บ Q-values สำหรับทุก (state, action) pair ซึ่งเป็น "ความรู้" ที่ agent สะสมขณะ training

```rust
fn action_to_u8(a: Action) -> u8 {
    match a {
        Action::Up    => 0,
        Action::Down  => 1,
        Action::Left  => 2,
        Action::Right => 3,
    }
}

fn u8_to_action(v: u8) -> Action {
    match v {
        0 => Action::Up,
        1 => Action::Down,
        2 => Action::Left,
        _ => Action::Right,
    }
}

// ใช้ String เป็น key เพื่อให้ serde_json serialize ได้
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QTable {
    pub table: HashMap<String, f64>,
    pub rows: usize,
    pub cols: usize,
}

impl QTable {
    pub fn new(rows: usize, cols: usize) -> Self {
        QTable { table: HashMap::new(), rows, cols }
    }

    fn key(state: State, action: Action) -> String {
        format!("{},{},{}", state.0, state.1, action_to_u8(action))
    }

    pub fn get(&self, state: State, action: Action) -> f64 {
        *self.table.get(&Self::key(state, action)).unwrap_or(&0.0)
    }

    pub fn set(&mut self, state: State, action: Action, value: f64) {
        self.table.insert(Self::key(state, action), value);
    }

    // หา max Q-value ใน state นี้ (ใช้ใน Bellman equation)
    pub fn max_q(&self, state: State) -> f64 {
        Action::all()
            .iter()
            .map(|a| self.get(state, *a))
            .fold(f64::NEG_INFINITY, f64::max)
    }

    // หา action ที่ดีที่สุดตาม Q-values ปัจจุบัน
    pub fn best_action(&self, state: State) -> Action {
        Action::all()
            .iter()
            .max_by(|a, b| {
                self.get(state, **a)
                    .partial_cmp(&self.get(state, **b))
                    .unwrap()
            })
            .copied()
            .unwrap_or(Action::Up)
    }

    // Epsilon-greedy: ด้วยความน่าจะเป็น ε เลือก action แบบสุ่ม
    // ไม่งั้นเลือก action ที่ดีที่สุด
    pub fn epsilon_greedy<R: Rng>(&self, state: State, epsilon: f64, rng: &mut R) -> Action {
        if rng.gen::<f64>() < epsilon {
            let idx = rng.gen_range(0..4usize);
            u8_to_action(idx as u8)   // explore
        } else {
            self.best_action(state)   // exploit
        }
    }
}
```

**ทำไม Epsilon-Greedy ถึงสำคัญ?**

ตอนเริ่ม training ทุก Q-value เป็น 0 ทั้งหมด ถ้าเลือกแต่ best action agent จะติดอยู่กับ action แรกที่เจอตลอด (exploitation trap) Epsilon-greedy แก้ปัญหานี้:

- ช่วงต้น (ε ≈ 1.0): explore เกือบทั้งหมด — ลองทุก action สุ่ม
- ช่วงกลาง (ε ≈ 0.3): ผสม explore/exploit — เรียนรู้ไปด้วย
- ช่วงท้าย (ε ≈ 0.01): exploit เกือบทั้งหมด — ใช้สิ่งที่เรียนรู้มา

---

### ขั้นที่ 4: Q-Learning Update Rule

หัวใจของ Q-Learning คือ **Bellman Optimality Equation**:

```
Q(s, a) ← Q(s, a) + α × [r + γ × max_a' Q(s', a') − Q(s, a)]
```

แต่ละ term มีความหมาย:
- `Q(s, a)` — ค่า Q เดิม (สิ่งที่เรารู้อยู่แล้ว)
- `r` — reward จริงที่ได้รับขณะนี้
- `γ × max_a' Q(s', a')` — ค่าในอนาคตที่คาดหวัง (discounted)
- `α` — learning rate (0 = ไม่เรียนรู้เลย, 1 = ลืมทุกอย่าง เชื่อข้อมูลใหม่อย่างเดียว)
- `γ` — discount factor (0 = สนแค่ reward ทันที, 1 = สนทุก future reward เท่ากัน)

```rust
pub fn update(
    &mut self,
    state: State,
    action: Action,
    alpha: f64,
    gamma: f64,
    reward: f64,
    next_state: State,
) {
    let old_q = self.get(state, action);
    let max_next = self.max_q(next_state);
    // TD target = r + γ × max_Q(s')
    // TD error  = TD target - old_Q
    let new_q = old_q + alpha * (reward + gamma * max_next - old_q);
    self.set(state, action, new_q);
}
```

**ตัวอย่างการคำนวณ:**

สมมติว่า:
- `Q((0,1), Right) = 0.0` ตอนเริ่มต้น
- `r = -0.01` (step penalty)
- `max_Q((0,2)) = 0.0` (ยังไม่รู้อะไร)
- `α = 0.1`, `γ = 0.99`

```
new_Q = 0.0 + 0.1 × (−0.01 + 0.99 × 0.0 − 0.0)
      = 0.0 + 0.1 × (−0.01)
      = −0.001
```

หลังจาก agent เดินถึง goal แล้ว (reward = 1.0) และ backpropagate กลับมา Q-value ใน path ที่นำไปสู่ goal จะค่อย ๆ เพิ่มขึ้น

**Serialize/Deserialize:**

```rust
pub fn to_json(&self) -> String {
    serde_json::to_string_pretty(self).unwrap_or_default()
}

pub fn from_json(s: &str) -> Option<Self> {
    serde_json::from_str(s).ok()
}
```

ตัวอย่าง JSON output:
```json
{
  "table": {
    "2,1,3": 0.891,
    "2,2,0": 0.234,
    "1,0,1": 0.756
  },
  "rows": 3,
  "cols": 3
}
```

---

### ขั้นที่ 5: Training Loop พร้อม Epsilon Decay Schedule

ตอนนี้รวมทุกส่วนเข้าด้วยกันใน training loop:

```rust
#[derive(Debug, Clone)]
pub struct TrainingConfig {
    pub episodes: usize,     // จำนวน episode ทั้งหมด
    pub max_steps: usize,    // steps สูงสุดต่อ episode
    pub alpha: f64,          // learning rate
    pub gamma: f64,          // discount factor
    pub epsilon_start: f64,  // ε ตอนเริ่มต้น
    pub epsilon_end: f64,    // ε ขั้นต่ำ
    pub epsilon_decay: f64,  // อัตรา decay ต่อ episode
}

impl Default for TrainingConfig {
    fn default() -> Self {
        TrainingConfig {
            episodes: 1000,
            max_steps: 200,
            alpha: 0.1,
            gamma: 0.99,
            epsilon_start: 1.0,
            epsilon_end: 0.01,
            epsilon_decay: 0.995,
        }
    }
}
```

**Epsilon Decay Schedule:**

`epsilon_decay = 0.995` หมายความว่าทุก episode:
```
ε_new = max(ε_end, ε_old × 0.995)
```

หลัง 1000 episodes: `1.0 × 0.995^1000 ≈ 0.0067 → clamp ที่ 0.01`
หลัง 500 episodes:  `1.0 × 0.995^500 ≈ 0.082`

นี่คือ "linear in log space" decay — ให้เวลา explore เพียงพอช่วงต้น:

```rust
#[derive(Debug, Clone)]
pub struct EpisodeStats {
    pub episode: usize,
    pub total_reward: f64,
    pub steps: usize,
    pub epsilon: f64,
}

pub struct TrainingResult {
    pub q_table: QTable,
    pub episode_stats: Vec<EpisodeStats>,
}

pub fn train_q_learning(env: &mut GridWorld, config: &TrainingConfig) -> TrainingResult {
    let mut rng = rand::thread_rng();
    let mut q_table = QTable::new(env.rows, env.cols);
    let mut stats = Vec::new();
    let mut epsilon = config.epsilon_start;

    for ep in 0..config.episodes {
        let mut state = env.reset();
        let mut total_reward = 0.0;
        let mut steps = 0;

        for _ in 0..config.max_steps {
            // 1. เลือก action ด้วย epsilon-greedy
            let action = q_table.epsilon_greedy(state, epsilon, &mut rng);

            // 2. ทำ action ใน environment
            let (next_state, reward, done) = env.step(action);

            // 3. อัปเดต Q-table ด้วย Bellman equation
            q_table.update(state, action, config.alpha, config.gamma, reward, next_state);

            total_reward += reward;
            state = next_state;
            steps += 1;

            if done { break; }
        }

        // 4. Decay epsilon หลัง episode จบ
        epsilon = (epsilon * config.epsilon_decay).max(config.epsilon_end);

        stats.push(EpisodeStats { episode: ep, total_reward, steps, epsilon });
    }

    TrainingResult { q_table, episode_stats: stats }
}
```

**ทำไมต้อง `max_steps`?**

ถ้าไม่มี limit agent อาจวนอยู่ใน environment ตลอดไปได้ (infinite loop) โดยเฉพาะตอนต้น training ที่ policy ยังไม่ดี `max_steps = 200` บังคับให้ episode จบแม้ไม่ถึง goal

---

### ขั้นที่ 6: Convergence Tracking และ Policy Extraction

เพิ่มเครื่องมือวัด convergence:

```rust
impl TrainingResult {
    // คำนวณ running average ของ rewards (เหมือน sliding window mean)
    pub fn running_average(&self, window: usize) -> Vec<f64> {
        let rewards: Vec<f64> = self.episode_stats
            .iter()
            .map(|s| s.total_reward)
            .collect();
        rewards.windows(window)
            .map(|w| w.iter().sum::<f64>() / w.len() as f64)
            .collect()
    }

    // success rate = สัดส่วน episode ที่ reward > 0.5 (ถึง goal)
    // ดู window สุดท้ายเพื่อวัด "ตอนนี้ converge แค่ไหน"
    pub fn success_rate(&self, window: usize) -> f64 {
        let n = self.episode_stats.len();
        if n == 0 { return 0.0; }
        let start = if n > window { n - window } else { 0 };
        let successes = self.episode_stats[start..]
            .iter()
            .filter(|s| s.total_reward > 0.5)
            .count();
        successes as f64 / (n - start) as f64
    }
}
```

**Policy Visualization:**

```rust
pub fn print_policy(env: &GridWorld, q_table: &QTable) {
    println!("Policy (best action per cell):");
    for r in 0..env.rows {
        for c in 0..env.cols {
            let s = (r, c);
            if s == env.goal {
                print!("  G ");
            } else if env.is_wall(s) {
                print!("  # ");
            } else {
                let best = q_table.best_action(s);
                print!("  {} ", best.to_arrow());
            }
        }
        println!();
    }
}
```

ตัวอย่าง policy หลัง training บน 4×4 grid:

```
Policy (best action per cell):
  →   →   →   ↓
  ↓   #   #   ↓
  ↓   #   →   ↓
  →   →   →   G
```

จะเห็นว่า agent เรียนรู้ path ที่หลีกเลี่ยงกำแพง (`#`) และมุ่งสู่ goal (`G`)

**ตัวอย่างการติดตาม convergence:**

```rust
fn main() {
    let mut env = GridWorld::new(4, 4, (0,0), (3,3), vec![(1,1),(1,2),(2,1)]);
    let config = TrainingConfig { episodes: 2000, ..Default::default() };
    let result = train_q_learning(&mut env, &config);

    // แสดง running average ทุก 200 episode
    let avg = result.running_average(100);
    for (i, &v) in avg.iter().enumerate().step_by(200) {
        println!("Episode {:4}: avg reward = {:.4}", i + 100, v);
    }

    println!("Final success rate: {:.1}%", result.success_rate(200) * 100.0);
}
```

**Output ตัวอย่าง:**

```
Episode  100: avg reward = -0.7234
Episode  300: avg reward = -0.2156
Episode  500: avg reward =  0.1843
Episode  700: avg reward =  0.6921
Episode  900: avg reward =  0.8344
Episode 1100: avg reward =  0.8901
Final success rate: 93.5%
```

reward เพิ่มขึ้นเรื่อย ๆ แสดงว่า agent เรียนรู้ได้ดีขึ้นตลอด

---

### ขั้นที่ 7: SARSA — On-Policy TD Learning

SARSA (State-Action-Reward-State-Action) คือ on-policy variant ของ Q-learning ความแตกต่างสำคัญ:

| | Q-Learning | SARSA |
|---|---|---|
| **Policy** | Off-policy | On-policy |
| **Update** | ใช้ `max_Q(s')` | ใช้ `Q(s', a')` จริง |
| **เสถียรภาพ** | อาจ overestimate | ระมัดระวังกว่า |
| **ประสิทธิภาพ** | เร็วกว่า | ช้ากว่าเล็กน้อย |

**Off-policy**: Q-Learning เรียนรู้ optimal policy โดยไม่ว่า agent จะใช้ policy ใดตอน training
**On-policy**: SARSA เรียนรู้ policy ที่ agent กำลังใช้จริง (รวม exploration)

Update rule ของ SARSA:
```
Q(s, a) ← Q(s, a) + α × [r + γ × Q(s', a') − Q(s, a)]
```

ต่างจาก Q-learning ตรง `Q(s', a')` แทน `max_Q(s')` — ใช้ Q-value ของ action ที่จะทำจริงใน s'

```rust
pub fn train_sarsa(env: &mut GridWorld, config: &TrainingConfig) -> TrainingResult {
    let mut rng = rand::thread_rng();
    let mut q_table = QTable::new(env.rows, env.cols);
    let mut stats = Vec::new();
    let mut epsilon = config.epsilon_start;

    for ep in 0..config.episodes {
        let mut state = env.reset();
        // เลือก action แรกก่อน loop (ต่างจาก Q-learning)
        let mut action = q_table.epsilon_greedy(state, epsilon, &mut rng);
        let mut total_reward = 0.0;
        let mut steps = 0;

        for _ in 0..config.max_steps {
            let (next_state, reward, done) = env.step(action);
            // เลือก next action ด้วย epsilon-greedy
            let next_action = q_table.epsilon_greedy(next_state, epsilon, &mut rng);

            // SARSA update: ใช้ Q(s', a') ไม่ใช่ max_Q(s')
            let old_q = q_table.get(state, action);
            let next_q = q_table.get(next_state, next_action);
            let new_q = old_q + config.alpha * (reward + config.gamma * next_q - old_q);
            q_table.set(state, action, new_q);

            total_reward += reward;
            state = next_state;
            action = next_action;  // ← ส่งต่อ action ไปใช้รอบถัดไป
            steps += 1;

            if done { break; }
        }

        epsilon = (epsilon * config.epsilon_decay).max(config.epsilon_end);
        stats.push(EpisodeStats { episode: ep, total_reward, steps, epsilon });
    }

    TrainingResult { q_table, episode_stats: stats }
}
```

**เมื่อไหร่ SARSA ดีกว่า Q-learning?**

ใน "cliff walking" environment ที่ทางสั้นที่สุดเดินริมหน้าผา Q-learning จะเรียนรู้ path ริมหน้าผา (optimal แต่เสี่ยง) เพราะใช้ max-Q ที่ไม่คำนึงถึง exploration SARSA จะเรียนรู้ path ที่ปลอดภัยกว่า (ห่างจากหน้าผา) เพราะ exploration บางครั้งทำให้ตกหน้าผา ทำให้ Q(s', a') สะท้อนความเสี่ยงนั้น

---

### ขั้นที่ 8: Double Q-Learning — ลด Overestimation Bias

**ปัญหา Overestimation ใน Q-Learning:**

Q-Learning มี systematic bias — มันมักประเมิน Q-values สูงเกินจริง เหตุผลคือเราใช้ `max_Q(s')` ซึ่งเลือก noisy estimate ที่สูงที่สุด ในทางสถิติ:

```
E[max(Q̂)] ≥ max(E[Q̂])
```

คือ expected value ของ max estimate สูงกว่า max ของ expected values เสมอ

**Double Q-Learning** แก้ปัญหาด้วยการแยก "การเลือก action" และ "การประเมิน value" ออกจากกัน:

```
# Standard Q-Learning:
target = r + γ × max_a' Q(s', a')   ← ใช้ Q table เดียวทั้งเลือกและประเมิน

# Double Q-Learning:
best_a = argmax_a' Q_A(s', a')       ← Q_A เลือก action
target = r + γ × Q_B(s', best_a)     ← Q_B ประเมิน value
```

โดยสลับบทบาทระหว่าง Q_A และ Q_B แบบสุ่มใน training:

```rust
pub fn train_double_q_learning(env: &mut GridWorld, config: &TrainingConfig) -> TrainingResult {
    let mut rng = rand::thread_rng();
    let mut q_a = QTable::new(env.rows, env.cols);
    let mut q_b = QTable::new(env.rows, env.cols);
    let mut stats = Vec::new();
    let mut epsilon = config.epsilon_start;

    for ep in 0..config.episodes {
        let mut state = env.reset();
        let mut total_reward = 0.0;
        let mut steps = 0;

        for _ in 0..config.max_steps {
            // epsilon-greedy ใช้ผลรวม Q_A + Q_B
            let action = if rng.gen::<f64>() < epsilon {
                u8_to_action(rng.gen_range(0..4u8))
            } else {
                Action::all()
                    .iter()
                    .max_by(|a, b| {
                        let sum_a = q_a.get(state, **a) + q_b.get(state, **a);
                        let sum_b = q_a.get(state, **b) + q_b.get(state, **b);
                        sum_a.partial_cmp(&sum_b).unwrap()
                    })
                    .copied()
                    .unwrap_or(Action::Up)
            };

            let (next_state, reward, done) = env.step(action);

            // สลับอัปเดต Q_A และ Q_B แบบสุ่ม
            if rng.gen::<f64>() < 0.5 {
                // อัปเดต Q_A: Q_A เลือก action, Q_B ประเมิน value
                let best_action_a = q_a.best_action(next_state);
                let next_value = q_b.get(next_state, best_action_a);
                let old_q = q_a.get(state, action);
                let new_q = old_q + config.alpha * (reward + config.gamma * next_value - old_q);
                q_a.set(state, action, new_q);
            } else {
                // อัปเดต Q_B: Q_B เลือก action, Q_A ประเมิน value
                let best_action_b = q_b.best_action(next_state);
                let next_value = q_a.get(next_state, best_action_b);
                let old_q = q_b.get(state, action);
                let new_q = old_q + config.alpha * (reward + config.gamma * next_value - old_q);
                q_b.set(state, action, new_q);
            }

            total_reward += reward;
            state = next_state;
            steps += 1;

            if done { break; }
        }

        epsilon = (epsilon * config.epsilon_decay).max(config.epsilon_end);
        stats.push(EpisodeStats { episode: ep, total_reward, steps, epsilon });
    }

    // รวม Q_A + Q_B เป็น Q table เดียว (หาร 2 สำหรับ policy extraction)
    let mut merged = QTable::new(env.rows, env.cols);
    for r in 0..env.rows {
        for c in 0..env.cols {
            let s = (r, c);
            for a in Action::all() {
                let avg = (q_a.get(s, a) + q_b.get(s, a)) / 2.0;
                merged.set(s, a, avg);
            }
        }
    }

    TrainingResult { q_table: merged, episode_stats: stats }
}
```

Double Q-Learning มีประโยชน์เด่นชัดขึ้นเมื่อ:
- Reward มี noise สูง
- Environment มีหลาย path ที่ดูดีพอกัน
- ต้องการ Q-values ที่ calibrate ได้ดีสำหรับ policy comparison

---

### ขั้นที่ 9: Policy Visualization และ Q-Value Heatmap

สุดท้ายเพิ่ม visualization ที่ช่วยให้เข้าใจสิ่งที่ agent เรียนรู้:

```rust
pub fn print_policy(env: &GridWorld, q_table: &QTable) {
    println!("Policy (best action per cell):");
    for r in 0..env.rows {
        for c in 0..env.cols {
            let s = (r, c);
            if s == env.goal {
                print!("  G ");
            } else if env.is_wall(s) {
                print!("  # ");
            } else {
                let best = q_table.best_action(s);
                print!("  {} ", best.to_arrow());
            }
        }
        println!();
    }
}

pub fn print_q_values(env: &GridWorld, q_table: &QTable, action: Action) {
    println!("Q-values for action {:?}:", action);
    for r in 0..env.rows {
        for c in 0..env.cols {
            let s = (r, c);
            if env.is_wall(s) {
                print!("  ### ");
            } else {
                print!("{:6.3} ", q_table.get(s, action));
            }
        }
        println!();
    }
}
```

ตัวอย่าง main.rs ที่รวมทุกส่วน:

```rust
use q_learning_rl::*;

fn main() {
    println!("=== Q-Learning Grid World Demo ===\n");

    let walls = vec![(1, 1), (1, 2), (2, 1), (3, 3)];
    let mut env = GridWorld::new(5, 5, (0, 0), (4, 4), walls);

    let config = TrainingConfig {
        episodes: 2000,
        max_steps: 200,
        alpha: 0.1,
        gamma: 0.99,
        epsilon_start: 1.0,
        epsilon_end: 0.01,
        epsilon_decay: 0.995,
    };

    println!("Training Q-Learning for {} episodes...", config.episodes);
    let result = train_q_learning(&mut env, &config);

    println!("Training complete!");
    println!("Success rate (last 200 eps): {:.1}%",
        result.success_rate(200) * 100.0);

    println!("\n--- Learned Policy ---");
    print_policy(&env, &result.q_table);

    // Save Q-table to JSON
    let json = result.q_table.to_json();
    println!("\nQ-table serialized ({} entries)", result.q_table.table.len());

    // Reload and verify
    let reloaded = QTable::from_json(&json).expect("failed to deserialize");
    println!("Reloaded Q-table: {} entries", reloaded.table.len());
}
```

**Output ตัวอย่าง:**

```
=== Q-Learning Grid World Demo ===

Training Q-Learning for 2000 episodes...
Training complete!
Success rate (last 200 eps): 96.5%

--- Learned Policy ---
Policy (best action per cell):
  →   →   ↓   ↓   ↓
  ↓   #   #   ↓   ↓
  ↓   #   →   →   ↓
  ↓   →   →   #   ↓
  →   →   →   →   G

Q-table serialized (84 entries)
Reloaded Q-table: 84 entries
```

---

## การทดสอบ (Testing)

ชุด test ครอบคลุมทุก component สำคัญ:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn make_simple_env() -> GridWorld {
        // 3x3 grid, start (0,0), goal (2,2), ไม่มีกำแพง
        GridWorld::new(3, 3, (0, 0), (2, 2), vec![])
    }

    fn make_wall_env() -> GridWorld {
        // 4x4 grid มีกำแพง
        GridWorld::new(4, 4, (0, 0), (3, 3), vec![(1, 1), (1, 2), (2, 1)])
    }

    #[test]
    fn test_gridworld_reset() {
        let mut env = make_simple_env();
        env.step(Action::Right);
        let state = env.reset();
        assert_eq!(state, (0, 0));
    }

    #[test]
    fn test_qtable_update_bellman() {
        let mut q = QTable::new(3, 3);
        // Q(s,a)=0, r=1.0, max_Q(s')=0, α=0.1, γ=0.99
        // new_Q = 0 + 0.1*(1.0 + 0.99*0 - 0) = 0.1
        q.update((0, 0), Action::Right, 0.1, 0.99, 1.0, (0, 1));
        assert!((q.get((0, 0), Action::Right) - 0.1).abs() < 1e-9);
    }

    // ... (ดู src/lib.rs สำหรับ tests ทั้งหมด 23 test)
}
```

รัน tests จริง:

```
$ cargo test
```

**Output จริงจาก `cargo test`:**

```
   Compiling q_learning_rl v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.05s
     Running unittests src/lib.rs (target/debug/deps/q_learning_rl-a4dfe29a715823ad)

running 23 tests
test tests::test_action_arrows ... ok
test tests::test_all_actions ... ok
test tests::test_episode_stats_count ... ok
test tests::test_get_all_states ... ok
test tests::test_gridworld_boundary ... ok
test tests::test_epsilon_decay ... ok
test tests::test_get_all_states_with_walls ... ok
test tests::test_gridworld_reset ... ok
test tests::test_gridworld_step_basic ... ok
test tests::test_gridworld_step_goal ... ok
test tests::test_gridworld_is_terminal ... ok
test tests::test_gridworld_wall_collision ... ok
test tests::test_qtable_best_action ... ok
test tests::test_qtable_initial_zero ... ok
test tests::test_qtable_max_q ... ok
test tests::test_qtable_serde ... ok
test tests::test_qtable_set_get ... ok
test tests::test_qtable_update_bellman ... ok
test tests::test_q_learning_converges ... ok
test tests::test_training_result_running_average ... ok
test tests::test_sarsa_converges ... ok
test tests::test_q_learning_wall_env ... ok
test tests::test_double_q_learning_converges ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.39s

     Running unittests src/main.rs (target/debug/deps/q_learning_rl-5f3b0b5143f27c08)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests q_learning_rl

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

tests ทั้ง 23 ผ่าน 100% ครอบคลุม:
- **Environment tests (7 tests)**: reset, step, goal detection, boundary, wall collision, terminal state
- **QTable tests (6 tests)**: initial values, set/get, max_q, best_action, Bellman update, serialization
- **Training tests (8 tests)**: Q-learning convergence, SARSA convergence, Double Q-learning convergence, wall navigation, running average, episode count, epsilon decay
- **Utility tests (2 tests)**: action arrows, all actions

---

## ข้อผิดพลาดที่พบบ่อย

### 1. Reward Design ผิดพลาด — ไม่มี Step Penalty

**ปัญหา:** กำหนด reward เป็น `0` สำหรับทุก step แล้วให้ `1.0` เฉพาะที่ goal

```rust
// ❌ ผิด: ไม่มี step penalty
let reward = if next == self.goal { 1.0 } else { 0.0 };
```

**ผลที่ตามมา:** Agent เรียนรู้ว่า "อยู่เฉย ๆ ก็ได้ 0 เหมือนกัน" หรืออาจเดินวนโดยไม่ได้ผลอะไร เพราะทุก path ที่ถึง goal มี total reward เท่ากัน (ไม่สนใจความยาว)

**วิธีแก้:**
```rust
// ✓ ถูก: ใช้ step penalty ให้ agent หา shortest path
let reward = if next == self.goal {
    1.0
} else if !moved {
    -0.1  // wall penalty
} else {
    -0.01 // step penalty
};
```

**บทเรียน:** Reward shaping ส่งผลโดยตรงต่อ behavior ที่ agent จะเรียนรู้ ต้องออกแบบ reward ให้สอดคล้องกับเป้าหมายจริง ๆ

---

### 2. Epsilon ไม่ Decay — ติดใน Exploration ตลอด

**ปัญหา:** ลืม decay epsilon หลัง training ทำให้ agent explore แบบสุ่มตลอดเวลา

```rust
// ❌ ผิด: epsilon คงที่ตลอด
for ep in 0..episodes {
    let action = q_table.epsilon_greedy(state, 0.5, &mut rng);
    // ... (ไม่อัปเดต epsilon)
}
```

**ผลที่ตามมา:** ช่วงท้าย training agent ยังสุ่ม action 50% ของเวลา ทำให้ success rate ต่ำแม้ Q-table จะดีแล้ว

**วิธีแก้:**
```rust
// ✓ ถูก: decay epsilon หลังทุก episode
epsilon = (epsilon * epsilon_decay).max(epsilon_end);
```

**การตั้งค่า epsilon ที่ดี:**
- `epsilon_start = 1.0` — explore ทั้งหมดตอนเริ่ม
- `epsilon_end = 0.01` — ยังมี 1% exploration ตลอด (ป้องกัน stuck)
- `epsilon_decay = 0.995` — decay ช้าพอให้ explore เพียงพอ

---

### 3. Gamma = 1.0 บน Infinite Horizon — Q-value ไม่ Converge

**ปัญหา:** ตั้ง `gamma = 1.0` บน environment ที่ไม่รับประกันว่า episode จะจบ

```rust
// ❌ อันตราย: gamma = 1.0 ทำให้ Q-values diverge
let config = TrainingConfig {
    gamma: 1.0,
    // ...
};
```

**ผลที่ตามมา:** Q-values อาจบวกหรือลบอย่างไม่มีขอบเขต เพราะ Q(s,a) = r + Q(s', a') + Q(s'', a'') + ... โดยไม่มี discounting ที่ทำให้ sum converge

**วิธีแก้:**
```rust
// ✓ ถูก: gamma < 1 เสมอสำหรับ Q-learning แบบ tabular
let config = TrainingConfig {
    gamma: 0.99,  // หรือ 0.95 ถ้าต้องการ focus ระยะสั้น
    // ...
};
```

**กฎทั่วไป:** `gamma = 0.99` เป็นค่า default ที่ดีสำหรับงาน navigation; ลดลงถ้า episode สั้น

---

### 4. HashMap Key ไม่ Compatible กับ serde_json

**ปัญหา:** ใช้ tuple เป็น key ของ `HashMap` และพยายาม serialize ด้วย serde_json

```rust
// ❌ ผิด: tuple key ไม่ serialize ด้วย serde_json ได้
#[derive(Serialize, Deserialize)]
pub struct QTable {
    pub table: HashMap<(usize, usize, u8), f64>,
}
// จะ panic ตอน runtime หรือ compile error
```

**สาเหตุ:** JSON ต้องการ key เป็น string เท่านั้น serde_json ไม่ serialize tuple เป็น JSON object key ได้

**วิธีแก้:**
```rust
// ✓ ถูก: ใช้ String เป็น key
#[derive(Serialize, Deserialize)]
pub struct QTable {
    pub table: HashMap<String, f64>,
}

fn key(state: State, action: Action) -> String {
    format!("{},{},{}", state.0, state.1, action_to_u8(action))
}
```

---

### 5. Off-by-One ใน Episode Loop — ข้อมูลหาย

**ปัญหา:** เก็บ stats นอก loop ทำให้ขาด episode สุดท้าย

```rust
// ❌ ผิด: stats เก็บแค่ตอนถึง goal
for ep in 0..episodes {
    // ...
    if done {
        stats.push(EpisodeStats { ... }); // ← เก็บแค่เมื่อ done
    }
}
```

**ผลที่ตามมา:** `running_average()` และ `success_rate()` คำนวณผิดเพราะข้อมูลไม่ครบ

**วิธีแก้:**
```rust
// ✓ ถูก: เก็บ stats ทุก episode ไม่ว่าจะ done หรือไม่
for ep in 0..episodes {
    // ...
    stats.push(EpisodeStats { episode: ep, total_reward, steps, epsilon });
    // เก็บหลัง loop inner เสมอ
}
```

---

### 6. Double Q-Learning — อัปเดตแค่ Table เดียว

**ปัญหา:** ลืมสลับระหว่าง Q_A และ Q_B ทำให้กลายเป็น Q-learning ธรรมดา

```rust
// ❌ ผิด: อัปเดตแค่ q_a เสมอ ไม่ได้ double
let next_value = q_b.get(next_state, q_a.best_action(next_state));
let old_q = q_a.get(state, action);
let new_q = old_q + alpha * (reward + gamma * next_value - old_q);
q_a.set(state, action, new_q);  // ← ไม่มีการอัปเดต q_b เลย
```

**วิธีแก้:**
```rust
// ✓ ถูก: สลับสุ่ม 50/50 ระหว่าง q_a และ q_b
if rng.gen::<f64>() < 0.5 {
    // อัปเดต Q_A ด้วย Q_B ประเมิน
    let best_a = q_a.best_action(next_state);
    let next_v = q_b.get(next_state, best_a);
    // update q_a ...
} else {
    // อัปเดต Q_B ด้วย Q_A ประเมิน
    let best_b = q_b.best_action(next_state);
    let next_v = q_a.get(next_state, best_b);
    // update q_b ...
}
```

---

## การ Package และ Deploy

### Build สำหรับ Release

```bash
# Build optimized binary
cargo build --release

# ไฟล์จะอยู่ที่ target/release/q_learning_rl
ls -lh target/release/q_learning_rl
```

### Run พร้อม Custom Config

```bash
# รันด้วย environment variable สำหรับ config
EPISODES=5000 ALPHA=0.05 cargo run --release
```

### Export Q-Table เป็น JSON

```rust
// บันทึก Q-table ที่ train แล้ว
use std::fs;

let json = result.q_table.to_json();
fs::write("qtable_trained.json", &json)?;
println!("Q-table saved to qtable_trained.json");

// โหลดกลับมาใช้งาน (เช่น inference)
let loaded_json = fs::read_to_string("qtable_trained.json")?;
let q_table = QTable::from_json(&loaded_json).expect("invalid Q-table JSON");
```

### Benchmark เปรียบเทียบ Algorithms

```rust
use std::time::Instant;

let mut env = GridWorld::new(10, 10, (0,0), (9,9), walls);
let config = TrainingConfig { episodes: 10000, ..Default::default() };

let t1 = Instant::now();
let r_ql = train_q_learning(&mut env, &config);
println!("Q-Learning: {:.2}s, SR={:.1}%",
    t1.elapsed().as_secs_f64(), r_ql.success_rate(500)*100.0);

let t2 = Instant::now();
let r_sarsa = train_sarsa(&mut env, &config);
println!("SARSA:      {:.2}s, SR={:.1}%",
    t2.elapsed().as_secs_f64(), r_sarsa.success_rate(500)*100.0);

let t3 = Instant::now();
let r_dql = train_double_q_learning(&mut env, &config);
println!("Double Q:   {:.2}s, SR={:.1}%",
    t3.elapsed().as_secs_f64(), r_dql.success_rate(500)*100.0);
```

ตัวอย่าง benchmark output บน 10×10 grid:
```
Q-Learning: 0.12s, SR=91.4%
SARSA:      0.13s, SR=88.7%
Double Q:   0.18s, SR=89.2%
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Stochastic Environment (ระดับง่าย)

ดัดแปลง `GridWorld` ให้มี action noise — บางครั้ง action ที่เลือกอาจไม่เกิดขึ้นจริง เช่น สั่ง "ขวา" แต่มี 10% โอกาสเดินผิดทิศ:

```rust
pub struct StochasticGridWorld {
    inner: GridWorld,
    noise: f64,   // 0.0 = deterministic, 0.3 = 30% random action
}

impl Environment for StochasticGridWorld {
    fn step(&mut self, action: Action) -> (State, f64, bool) {
        let actual_action = if self.rng.gen::<f64>() < self.noise {
            // เลือก action สุ่มอื่น
            let idx = self.rng.gen_range(0..4usize);
            u8_to_action(idx as u8)
        } else {
            action
        };
        self.inner.step(actual_action)
    }
    // ...
}
```

เปรียบเทียบว่า Q-Learning vs SARSA แตกต่างกันแค่ไหนบน stochastic environment และอธิบายว่าทำไม

---

### แบบฝึกหัดที่ 2: Prioritized Experience Replay (ระดับกลาง)

แทนที่จะ update ทันทีหลัง step ให้เก็บ transitions ลงใน replay buffer และ sample แบบ prioritized — transitions ที่มี TD error สูงควรได้รับเลือกบ่อยกว่า:

```rust
pub struct ReplayBuffer {
    buffer: Vec<Transition>,
    priorities: Vec<f64>,
    capacity: usize,
}

pub struct Transition {
    state: State,
    action: Action,
    reward: f64,
    next_state: State,
    done: bool,
}

impl ReplayBuffer {
    pub fn push(&mut self, t: Transition, priority: f64) { /* ... */ }

    pub fn sample(&self, n: usize, rng: &mut impl Rng) -> Vec<&Transition> {
        // Sample โดยให้น้ำหนักตาม priority
        // ...
    }
}
```

วัดว่า sample efficiency ดีขึ้นแค่ไหนเมื่อเทียบกับ uniform sampling

---

### แบบฝึกหัดที่ 3: Multi-Goal Environment (ระดับกลาง)

ดัดแปลง environment ให้มี goals หลาย goals ที่ให้ reward ต่างกัน และ agent ต้องตัดสินใจว่าจะไปที่ไหนก่อน:

```rust
pub struct MultiGoalWorld {
    inner: GridWorld,
    sub_goals: Vec<(State, f64)>,   // (position, reward)
    collected: HashSet<State>,
}
```

ทดสอบว่า Q-Learning เรียนรู้ optimal ordering ของ sub-goals ได้หรือไม่ และเปรียบเทียบกับ greedy strategy

---

### แบบฝึกหัดที่ 4: Function Approximation ด้วย Tile Coding (ระดับยาก)

Grid World ขนาดใหญ่ (100×100) ทำให้ Q-table มี entries ถึง 40,000 ค่า แทนที่จะเก็บทุก entry ให้ใช้ tile coding แบบง่าย:

```rust
pub struct TiledQTable {
    // แบ่ง grid เป็น tiles ขนาด k×k
    // cell ใน tile เดียวกันใช้ Q-value ร่วมกัน
    tile_size: usize,
    table: HashMap<(usize, usize, u8), f64>,
}

impl TiledQTable {
    fn state_to_tile(&self, state: State) -> (usize, usize) {
        (state.0 / self.tile_size, state.1 / self.tile_size)
    }
    // Q(s, a) ≈ Q(tile(s), a)
}
```

วัด trade-off ระหว่าง memory usage และ policy quality

---

### แบบฝึกหัดที่ 5: Parallel Training ด้วย Rayon (ระดับยาก)

ใช้ `rayon` crate เพื่อ train หลาย agent พร้อมกันแล้ว merge Q-tables:

```toml
[dependencies]
rayon = "1"
```

```rust
use rayon::prelude::*;

pub fn train_parallel(env_factory: impl Fn() -> GridWorld + Sync, config: &TrainingConfig, workers: usize) -> TrainingResult {
    let results: Vec<TrainingResult> = (0..workers)
        .into_par_iter()
        .map(|_| {
            let mut env = env_factory();
            train_q_learning(&mut env, config)
        })
        .collect();

    // Merge Q-tables โดยเฉลี่ย Q-values
    // ...
}
```

วัด speedup เมื่อเปรียบเทียบกับ single-threaded training

---

### แบบฝึกหัดที่ 6: Curriculum Learning (ระดับยาก)

เริ่ม train บน environment ง่าย (grid เล็ก ไม่มีกำแพง) แล้วค่อย ๆ เพิ่มความยากขึ้น — เทคนิคนี้ใช้ใน AlphaGo และ OpenAI Five:

```rust
pub struct CurriculumTrainer {
    stages: Vec<(GridWorld, usize)>,  // (environment, episodes)
}

impl CurriculumTrainer {
    pub fn train(&self) -> QTable {
        let mut q_table = QTable::new(/* max size */);

        for (env, episodes) in &self.stages {
            // Transfer learning: เริ่มจาก Q-table ที่ train มาแล้ว
            let result = train_with_qtable(env, &config, q_table.clone());
            q_table = result.q_table;
        }

        q_table
    }
}
```

เปรียบเทียบจำนวน episodes ที่ใช้ถึง convergence ระหว่าง curriculum training กับ direct training

---

## สรุป

ในโปรเจคนี้เราสร้าง Reinforcement Learning Framework ที่สมบูรณ์แบบตั้งแต่ต้น ประกอบด้วย:

**สิ่งที่สร้าง:**
- `Environment` trait ที่ยืดหยุ่น รองรับ environment ประเภทต่าง ๆ
- `GridWorld` environment ที่มี walls, rewards, และ terminal states
- `QTable` ที่ serialize/deserialize ด้วย serde_json ได้
- Q-Learning (off-policy TD) — converge เร็ว, optimal
- SARSA (on-policy TD) — ระมัดระวังกว่า, เหมาะกับ risky environments
- Double Q-Learning — ลด overestimation bias ได้ดี
- Training infrastructure: epsilon decay, convergence tracking, policy extraction
- Policy visualization ด้วย arrows บน grid

**Pattern สำคัญที่ได้เรียน:**
- **Trait-based design** ทำให้ swap environment ได้โดยไม่ต้องแก้ training code
- **TD Learning** เรียนรู้ step-by-step แทนที่จะรอจบ episode (ต่างจาก Monte Carlo)
- **Exploration vs Exploitation** เป็น fundamental trade-off ใน RL ทุกระบบ
- **Off-policy vs On-policy** มีผลต่อ stability และ behavior ของ agent
- **Serde + String keys** แก้ปัญหา HashMap serialization ที่พบบ่อย

**เชื่อมโยงไปโปรเจคถัดไป:**

โปรเจคนี้ใช้ tabular Q-table ซึ่งได้ผลดีบน discrete, small state space แต่เมื่อ state space ใหญ่ขึ้น (เช่น pixel input จากเกม) Q-table จะไม่ scalable **Project I10: Graph Neural Networks** จะสำรวจวิธีที่ neural network ช่วยประมาณ Q-function ได้บน continuous, high-dimensional state spaces ซึ่งเป็นพื้นฐานของ DQN ที่ DeepMind ใช้เล่น Atari

ความรู้เรื่อง trait design และ training loop structure ที่ได้จากโปรเจคนี้จะนำไปใช้ได้โดยตรงเมื่อเขียน deep learning framework

---

**โปรเจคก่อนหน้า:** [Project I08: Genetic Algorithm](project-i08-genetic-algorithm.md) | **โปรเจคถัดไป:** [Project I10: Graph Neural Network](project-i10-graph-nn.md)
