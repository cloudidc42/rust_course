# Project G08: Infrastructure as Code Engine

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 16 ชั่วโมง

## ภาพรวมโปรเจค

ในโลก DevOps สมัยใหม่ ไม่มีใครสร้าง infrastructure ด้วยมือแล้ว เครื่องมืออย่าง Terraform, Pulumi, และ AWS CloudFormation ได้เปลี่ยนแนวคิดการจัดการ server ไปโดยสิ้นเชิง — แทนที่จะ SSH เข้าไปสร้าง VM ทีละเครื่อง เราเขียน *declarations* ว่าต้องการ infrastructure แบบไหน แล้วปล่อยให้ engine คำนวณเองว่าต้องทำอะไรบ้างเพื่อให้โลกจริงตรงกับ declarations นั้น

โปรเจคนี้สร้าง **IaC Engine** แบบเล็กที่ทำงานได้จริง โดยใช้ Rust เป็นภาษาหลัก ครอบคลุมตั้งแต่:

1. **Parser** — อ่าน config language ที่ได้แรงบันดาลใจจาก HCL (HashiCorp Configuration Language) ด้วย recursive descent parser แบบเขียนเองทั้งหมด ไม่ใช้ parser generator
2. **Expression Evaluator** — แปลง `${var.name}`, `${local.x}`, function call อย่าง `format()` และ `join()` เป็นค่าจริง
3. **Resource Graph** — สร้าง DAG (Directed Acyclic Graph) จาก resource references และทำ topological sort เพื่อหา apply order
4. **Plan Engine** — เปรียบเทียบ desired state กับ current state (จาก state file) แล้วสร้าง plan ที่บอกว่าต้อง create/update/delete อะไรบ้าง
5. **Apply Engine** — รัน plan จริงผ่าน `Executor` trait, มี `NullExecutor` สำหรับ testing และ `FileExecutor` สำหรับสร้างไฟล์จริง
6. **State Management** — บันทึก/โหลด JSON state file และตรวจจับ **drift** (เมื่อ infrastructure จริงเบี่ยงเบนจาก state)

**Use case จริงในโลก production:**
- **Internal tooling**: สร้าง IaC engine เฉพาะสำหรับ platform ภายในบริษัทที่ไม่ต้องการ overhead ของ Terraform เต็มรูปแบบ
- **Config management**: ใช้เป็น layer บนสุดของระบบจัดการ configuration ที่ต้องการ declarative syntax
- **Testing harness**: สร้าง test infrastructure ที่ spin up/tear down resources อัตโนมัติใน CI/CD pipeline
- **Education**: เข้าใจว่า Terraform ทำงานอย่างไรจริง ๆ ข้างใต้ hood

**Learning value:**
โปรเจคนี้รวมทักษะ Rust ระดับสูงหลายอย่างไว้ด้วยกัน: recursive descent parsing, trait objects สำหรับ extensibility, graph algorithms, serde สำหรับ serialization, และ design patterns อย่าง strategy/command ที่ทำให้ engine extensible โดยไม่ต้อง modify core code

## สิ่งที่จะได้เรียนรู้

- **Recursive descent parsing** — เขียน parser ด้วยมือโดยไม่ใช้ library เข้าใจ tokenization, lookahead, และ error recovery
- **Enum-based AST** — ออกแบบ `HclValue`, `HclBlock` enum ให้แทนค่า config ทุกชนิดได้อย่างปลอดภัย
- **String interpolation engine** — parse และ evaluate `${...}` expressions ภายใน string โดยรองรับ nested expressions
- **Graph algorithms ใน Rust** — Kahn's algorithm สำหรับ topological sort, cycle detection ด้วย `HashMap` และ `VecDeque`
- **Trait-based executor pattern** — `Box<dyn Executor>` ทำให้เพิ่ม resource type ใหม่โดยไม่ต้อง touch engine
- **serde สำหรับ state serialization** — ใช้ `#[derive(Serialize, Deserialize)]` กับ nested struct ที่มี `HashMap<String, serde_json::Value>`
- **Diffing algorithm** — compare two state representations เพื่อหา minimal change set
- **Drift detection** — ระบุเมื่อ real-world state แตกต่างจาก recorded state

## ความรู้ที่ต้องมีมาก่อน

- **Part 15-20**: Traits, trait objects (`Box<dyn Trait>`), dynamic dispatch, object safety
- **Part 21-24**: Collections — `HashMap`, `HashSet`, `VecDeque`, และ iterator patterns
- **Part 25-30**: Error handling ด้วย `Result<T, E>` และ `?` operator
- **Part 35-40**: Closures, `map`, `filter`, `collect`, iterator adapters
- **Part 55-60**: `serde` ecosystem — Serialize, Deserialize, `serde_json::Value`
- **Part 96-100**: Design patterns — strategy, builder, command
- **Part 101-105**: Graph algorithms, sorting, complexity analysis

## โครงสร้างโปรเจค (Project Layout)

```
infra-as-code/
├── src/
│   ├── lib.rs          # re-export ทุก module
│   ├── parser.rs       # HCL-like parser — HclValue, HclBlock, Parser struct
│   ├── eval.rs         # Expression evaluator — Value enum, EvalContext, interpolation
│   ├── graph.rs        # ResourceGraph — DAG, topological sort, cycle detection
│   ├── plan.rs         # Plan — diff desired vs state, ResourceChange
│   ├── executor.rs     # Executor trait, NullExecutor, FileExecutor, ApplyEngine
│   └── state.rs        # StateStore — save/load JSON, drift detection
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ภาพรวม

```
[HCL Config File]
      │
      ▼ parser::parse()
[HclDocument { blocks: Vec<HclBlock> }]
      │
      ▼ eval::eval_value() + EvalContext
[Vec<DesiredResource> { type, name, attributes: HashMap<String, Value> }]
      │
      ├──▶ graph::build_graph()
      │         │
      │         ▼ topological_sort()
      │    [apply_order: Vec<String>]   ← resources ordered by dependency
      │
      ▼ plan::compute_plan(&desired, &state_file)
[Plan { to_create, to_update, to_delete }]
      │
      ▼ ApplyEngine::apply()
      │   ├── NullExecutor  (testing)
      │   └── FileExecutor  (real file ops)
      │
[Vec<ApplyResult>]
      │
      ▼ StateStore::record_applied() / record_deleted()
[Updated state.json on disk]
      │
      ▼ StateStore::detect_drift()
[Vec<DriftEntry>]  ← when real world diverges from state
```

### Core Design Decisions

**1. Separate `HclValue` จาก `Value`**

Parser ผลิต `HclValue` ซึ่งรวม `Expr(String)` ไว้สำหรับ interpolation ที่ยังไม่ได้ evaluate ส่วน evaluator ผลิต `Value` ที่ resolve แล้วทั้งหมด การแยกนี้ทำให้ parser ไม่ต้องรู้จัก EvalContext และ evaluator ไม่ต้องรู้จัก parsing

```
HclValue::String("hello-${var.env}")  →  eval_value()  →  Value::String("hello-production")
HclValue::Number(42.0)               →  eval_value()  →  Value::Number(42.0)
HclValue::Expr("format(...)")        →  eval_value()  →  Value::String("...")
```

**2. `serde_json::Value` ใน StateFile**

State file เก็บ attributes เป็น `HashMap<String, serde_json::Value>` ไม่ใช่ `HashMap<String, Value>` เหตุผลคือ state file ต้องอ่าน/เขียนจาก JSON ได้โดยตรง และ `serde_json::Value` เป็น type ที่ serde รองรับ natively โดยไม่ต้องเขียน custom serialization

**3. Executor trait เป็น `dyn` object**

```rust
pub trait Executor: Send + Sync {
    fn resource_type(&self) -> &str;
    fn create(&mut self, name: &str, attrs: &HashMap<String, Value>)
        -> Result<HashMap<String, serde_json::Value>, String>;
    // ...
}
```

การใช้ `Box<dyn Executor>` ทำให้ `ApplyEngine` ไม่รู้จัก concrete type ของ executor เลย สามารถเพิ่ม `DockerExecutor`, `KubernetesExecutor`, `AwsExecutor` ในภายหลังโดยไม่ต้อง modify engine

**4. Kahn's Algorithm สำหรับ Topological Sort**

เลือก Kahn's algorithm แทน DFS-based sort เพราะ cycle detection ทำได้ชัดเจนกว่า — ถ้า nodes ที่เหลือหลัง sort มี in-degree > 0 แสดงว่ามี cycle และ nodes เหล่านั้นคือ cycle participants

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Parser — อ่าน HCL-like Config Language

ก่อนอื่น ออกแบบ AST types:

```rust
// src/parser.rs
#[derive(Debug, Clone, PartialEq)]
pub enum HclValue {
    String(String),
    Number(f64),
    Bool(bool),
    List(Vec<HclValue>),
    Map(Vec<(String, HclValue)>),
    Null,
    Expr(String),  // unresolved interpolation
}

#[derive(Debug, Clone, PartialEq)]
pub struct ResourceBlock {
    pub resource_type: String,
    pub name: String,
    pub attributes: Vec<(String, HclValue)>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum HclBlock {
    Resource(ResourceBlock),
    Variable(VariableBlock),
    Output(OutputBlock),
    Locals(LocalsBlock),
}
```

Parser ใช้ recursive descent — แต่ละ method ทำหน้าที่ parse element หนึ่งชนิด:

```rust
pub struct Parser<'a> {
    input: &'a str,
    pos: usize,
}

impl<'a> Parser<'a> {
    fn skip_whitespace(&mut self) {
        loop {
            while matches!(self.peek(), Some(' ') | Some('\t') | Some('\n') | Some('\r')) {
                self.advance();
            }
            // skip line comments
            if self.remaining().starts_with("//") || self.remaining().starts_with('#') {
                while !matches!(self.peek(), Some('\n') | None) {
                    self.advance();
                }
            } else {
                break;
            }
        }
    }

    fn parse_string(&mut self) -> Result<String, ParseError> {
        self.expect("\"")?;
        let mut result = String::new();
        loop {
            match self.peek() {
                None => return Err(ParseError { message: "unterminated string".into(), position: self.pos }),
                Some('"') => { self.advance(); break; }
                Some('\\') => {
                    self.advance();
                    match self.advance() {
                        Some('n') => result.push('\n'),
                        Some('"') => result.push('"'),
                        Some(c) => result.push(c),
                        None => return Err(ParseError { message: "unterminated escape".into(), position: self.pos }),
                    }
                }
                Some(c) => { result.push(c); self.advance(); }
            }
        }
        Ok(result)
    }

    pub fn parse(&mut self) -> Result<HclDocument, ParseError> {
        let mut blocks = Vec::new();
        loop {
            self.skip_whitespace();
            if self.pos >= self.input.len() { break; }
            let kw = self.parse_ident()?;
            match kw.as_str() {
                "resource" => blocks.push(HclBlock::Resource(self.parse_resource_block()?)),
                "variable" => blocks.push(HclBlock::Variable(self.parse_variable_block()?)),
                "output"   => blocks.push(HclBlock::Output(self.parse_output_block()?)),
                "locals"   => blocks.push(HclBlock::Locals(self.parse_locals_block()?)),
                other => return Err(ParseError {
                    message: format!("unknown block type {:?}", other),
                    position: self.pos,
                }),
            }
        }
        Ok(HclDocument { blocks })
    }
}
```

ตัวอย่าง config ที่ parser รองรับ:

```hcl
// ตัวอย่าง infrastructure config
variable "env" {
  default     = "production"
  description = "Deployment environment"
}

variable "region" {
  default = "ap-southeast-1"
}

locals {
  app_name = "myapp"
  prefix   = "prod"
}

resource "file" "config" {
  path    = "/etc/myapp/config.ini"
  content = "env=production\nregion=ap-southeast-1"
  enabled = true
}

resource "file" "hosts" {
  path    = "/etc/hosts"
  content = "127.0.0.1 localhost"
}

output "config_path" {
  value = "/etc/myapp/config.ini"
}
```

### ขั้นที่ 2: Expression Evaluator

หลังจาก parse ได้ `HclDocument` แล้ว ต้อง evaluate expressions:

```rust
// src/eval.rs
#[derive(Debug, Clone, PartialEq)]
pub enum Value {
    String(String),
    Number(f64),
    Bool(bool),
    List(Vec<Value>),
    Map(HashMap<String, Value>),
    Null,
}

impl std::fmt::Display for Value {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Value::String(s) => write!(f, "{}", s),
            Value::Number(n) => {
                if n.fract() == 0.0 { write!(f, "{}", *n as i64) }
                else { write!(f, "{}", n) }
            }
            Value::Bool(b) => write!(f, "{}", b),
            Value::List(items) => {
                let s: Vec<String> = items.iter().map(|v| v.to_string()).collect();
                write!(f, "[{}]", s.join(", "))
            }
            Value::Map(_) => write!(f, "{{...}}"),
            Value::Null => write!(f, "null"),
        }
    }
}
```

`EvalContext` เก็บ variables, locals, และ resource outputs ที่ resolve แล้ว:

```rust
pub struct EvalContext {
    pub variables: HashMap<String, Value>,
    pub locals: HashMap<String, Value>,
    pub resource_outputs: HashMap<String, HashMap<String, Value>>,
}
```

Interpolation engine scan หา `${...}` ภายใน string และ evaluate แต่ละ expression:

```rust
pub fn interpolate(template: &str, ctx: &EvalContext) -> Result<String, EvalError> {
    let mut result = String::new();
    let mut chars = template.chars().peekable();
    while let Some(c) = chars.next() {
        if c == '$' && chars.peek() == Some(&'{') {
            chars.next(); // consume '{'
            let mut expr = String::new();
            let mut depth = 1;
            for ec in chars.by_ref() {
                if ec == '{' { depth += 1; expr.push(ec); }
                else if ec == '}' {
                    depth -= 1;
                    if depth == 0 { break; }
                    expr.push(ec);
                } else {
                    expr.push(ec);
                }
            }
            let val = eval_expr(expr.trim(), ctx)?;
            result.push_str(&val.to_string());
        } else {
            result.push(c);
        }
    }
    Ok(result)
}
```

Built-in functions ที่รองรับ:

| Function | ตัวอย่าง | ผลลัพธ์ |
|----------|----------|----------|
| `format(fmt, args...)` | `format("app-%s", var.env)` | `"app-production"` |
| `join(sep, list)` | `join(", ", local.ports)` | `"80, 443"` |
| `length(val)` | `length("hello")` | `5` |
| `tostring(val)` | `tostring(42)` | `"42"` |

### ขั้นที่ 3: Resource Dependency Graph

Resources มักมีการ reference กัน เช่น security group ต้องสร้างก่อน EC2 instance:

```rust
// src/graph.rs
pub struct ResourceGraph {
    /// Adjacency list: node → set of nodes it depends ON
    pub dependencies: HashMap<String, HashSet<String>>,
    pub nodes: HashSet<String>,
}

impl ResourceGraph {
    pub fn add_edge(&mut self, from: &str, to: &str) {
        // from depends on to → to must be applied first
        self.add_node(from);
        self.add_node(to);
        self.dependencies
            .entry(from.to_string())
            .or_default()
            .insert(to.to_string());
    }
}
```

Kahn's Algorithm สำหรับ topological sort:

```rust
pub fn topological_sort(&self) -> Result<Vec<String>, Vec<String>> {
    // คำนวณ in-degree ของแต่ละ node (จำนวน dependencies ที่มี)
    let mut in_degree_map: HashMap<String, usize> =
        self.nodes.iter().map(|n| (n.clone(), 0)).collect();
    let mut forward: HashMap<String, Vec<String>> = HashMap::new();

    for (node, deps) in &self.dependencies {
        for dep in deps {
            *in_degree_map.entry(node.clone()).or_insert(0) += 1;
            forward.entry(dep.clone()).or_default().push(node.clone());
        }
    }

    // เริ่มจาก nodes ที่ไม่มี dependency (in-degree = 0)
    let mut queue: VecDeque<String> = in_degree_map.iter()
        .filter(|(_, &deg)| deg == 0)
        .map(|(n, _)| n.clone())
        .collect();

    let mut result = Vec::new();
    while let Some(node) = queue.pop_front() {
        result.push(node.clone());
        if let Some(dependents) = forward.get(&node) {
            for dep in dependents {
                let deg = in_degree_map.get_mut(dep).unwrap();
                *deg -= 1;
                if *deg == 0 {
                    queue.push_back(dep.clone());
                }
            }
        }
    }

    if result.len() == self.nodes.len() {
        Ok(result)    // topological order
    } else {
        // nodes ที่ยังมี in-degree > 0 คือ cycle participants
        let cycle_nodes: Vec<String> = in_degree_map.iter()
            .filter(|(_, &deg)| deg > 0)
            .map(|(n, _)| n.clone())
            .collect();
        Err(cycle_nodes)
    }
}
```

ตัวอย่าง dependency graph:

```
security_group  ←──── ec2_instance
     │                      │
     ▼                      │
  vpc_subnet  ◄─────────────┘
```

topological order จะเป็น: `vpc_subnet` → `security_group` → `ec2_instance`

### ขั้นที่ 4: Plan Engine — Diff Desired vs Current State

Plan engine เปรียบเทียบ desired state (จาก config) กับ current state (จาก state file):

```rust
// src/plan.rs
#[derive(Debug, Clone, PartialEq)]
pub enum ChangeAction { Create, Update, Delete, NoOp }

#[derive(Debug, Clone)]
pub struct ResourceChange {
    pub key: String,
    pub resource_type: String,
    pub name: String,
    pub action: ChangeAction,
    pub desired: Option<HashMap<String, Value>>,
    pub current: Option<HashMap<String, serde_json::Value>>,
}

#[derive(Debug, Clone, Default)]
pub struct Plan {
    pub to_create: Vec<ResourceChange>,
    pub to_update: Vec<ResourceChange>,
    pub to_delete: Vec<ResourceChange>,
}
```

Algorithm สำหรับ compute_plan:

```rust
pub fn compute_plan(desired: &[DesiredResource], state: &StateFile) -> Plan {
    let mut plan = Plan::default();
    let desired_keys: HashMap<String, &DesiredResource> =
        desired.iter().map(|r| (r.key(), r)).collect();

    // Pass 1: ตรวจทุก desired resource ว่าต้อง create หรือ update
    for (key, res) in &desired_keys {
        match state.get(key) {
            None => plan.to_create.push(ResourceChange { action: ChangeAction::Create, .. }),
            Some(current_state) => {
                // เปรียบเทียบ attribute ทีละค่า
                let changed = res.attributes.iter().any(|(attr_key, desired_val)| {
                    let desired_json = value_to_json(desired_val);
                    let current_json = current_state.attributes.get(attr_key)
                        .cloned().unwrap_or(serde_json::Value::Null);
                    desired_json != current_json
                });
                if changed {
                    plan.to_update.push(ResourceChange { action: ChangeAction::Update, .. });
                }
            }
        }
    }

    // Pass 2: resources ที่อยู่ใน state แต่ไม่อยู่ใน desired → delete
    for (key, rs) in &state.resources {
        if !desired_keys.contains_key(key) {
            plan.to_delete.push(ResourceChange { action: ChangeAction::Delete, .. });
        }
    }

    plan
}
```

### ขั้นที่ 5: Apply Engine — รัน Plan จริง

```rust
// src/executor.rs
pub trait Executor: Send + Sync {
    fn resource_type(&self) -> &str;
    fn create(&mut self, name: &str, attrs: &HashMap<String, Value>)
        -> Result<HashMap<String, serde_json::Value>, String>;
    fn update(&mut self, name: &str, attrs: &HashMap<String, Value>,
              current: &HashMap<String, serde_json::Value>)
        -> Result<HashMap<String, serde_json::Value>, String>;
    fn delete(&mut self, name: &str, current: &HashMap<String, serde_json::Value>)
        -> Result<(), String>;
}
```

**NullExecutor** — บันทึก calls ทั้งหมดสำหรับ testing:

```rust
pub struct NullExecutor {
    pub resource_type_name: String,
    pub calls: Vec<String>,  // ["create:myres", "update:other", ...]
}

impl Executor for NullExecutor {
    fn create(&mut self, name: &str, attrs: &HashMap<String, Value>)
        -> Result<HashMap<String, serde_json::Value>, String>
    {
        self.calls.push(format!("create:{}", name));
        // สร้าง result map จาก attrs + auto-generate id
        let mut result = HashMap::new();
        for (k, v) in attrs { result.insert(k.clone(), value_to_json(v)); }
        result.insert("id".to_string(),
            serde_json::json!(format!("{}-id-{}", self.resource_type_name, name)));
        Ok(result)
    }
    // ...
}
```

**FileExecutor** — สร้าง/ลบไฟล์จริงบน disk:

```rust
pub struct FileExecutor {
    pub base_dir: std::path::PathBuf,
    pub calls: Vec<String>,
}

impl Executor for FileExecutor {
    fn create(&mut self, name: &str, attrs: &HashMap<String, Value>)
        -> Result<HashMap<String, serde_json::Value>, String>
    {
        let path_str = attrs.get("path")
            .and_then(|v| if let Value::String(s) = v { Some(s.clone()) } else { None })
            .ok_or("file resource requires 'path' attribute")?;
        let content = attrs.get("content").map(|v| v.to_string()).unwrap_or_default();
        let full_path = self.base_dir.join(path_str.trim_start_matches('/'));
        std::fs::create_dir_all(full_path.parent().unwrap())?;
        std::fs::write(&full_path, &content)?;
        // ...
    }
}
```

**ApplyEngine** orchestrates ทุก executor:

```rust
pub struct ApplyEngine {
    executors: HashMap<String, Box<dyn Executor>>,
    pub apply_log: Vec<ApplyResult>,
}

impl ApplyEngine {
    pub fn register<E: Executor + 'static>(&mut self, executor: E) {
        self.executors.insert(executor.resource_type().to_string(), Box::new(executor));
    }

    pub fn apply(&mut self, change: &ResourceChange) -> ApplyResult {
        let executor = match self.executors.get_mut(&change.resource_type) {
            Some(e) => e,
            None => return ApplyResult { success: false,
                message: format!("no executor for {:?}", change.resource_type), .. },
        };

        match &change.action {
            ChangeAction::Create => {
                match executor.create(&change.name, change.desired.as_ref().unwrap()) {
                    Ok(applied) => ApplyResult { success: true, applied_attributes: applied, .. },
                    Err(e) => ApplyResult { success: false, message: e, .. },
                }
            }
            // ... update, delete
        }
    }
}
```

### ขั้นที่ 6: State Management และ Drift Detection

```rust
// src/state.rs
pub struct StateStore {
    pub state: StateFile,
    path: Option<PathBuf>,
}

impl StateStore {
    pub fn load(&mut self) -> Result<(), String> {
        let path = match &self.path {
            Some(p) => p.clone(),
            None => return Ok(()), // in-memory only
        };
        if !path.exists() { return Ok(()); }
        let content = std::fs::read_to_string(&path)?;
        self.state = serde_json::from_str(&content)?;
        Ok(())
    }

    pub fn save(&self) -> Result<(), String> {
        let path = match &self.path { Some(p) => p.clone(), None => return Ok(()) };
        let content = serde_json::to_string_pretty(&self.state)?;
        std::fs::write(&path, content)?;
        Ok(())
    }

    pub fn detect_drift(
        &self,
        actual: &HashMap<String, HashMap<String, serde_json::Value>>,
    ) -> Vec<DriftEntry> {
        let mut drifts = Vec::new();
        for (key, rs) in &self.state.resources {
            match actual.get(key) {
                Some(actual_attrs) => {
                    for (attr_key, expected_val) in &rs.attributes {
                        let actual_val = actual_attrs.get(attr_key)
                            .cloned().unwrap_or(serde_json::Value::Null);
                        if *expected_val != actual_val {
                            drifts.push(DriftEntry {
                                resource_key: key.clone(),
                                attribute: attr_key.clone(),
                                expected: expected_val.clone(),
                                actual: actual_val,
                            });
                        }
                    }
                }
                None => {
                    // Resource ถูกลบนอก IaC — drift!
                    drifts.push(DriftEntry {
                        resource_key: key.clone(),
                        attribute: "__exists__".to_string(),
                        expected: serde_json::json!(true),
                        actual: serde_json::json!(false),
                    });
                }
            }
        }
        drifts
    }
}
```

State file รูปแบบ JSON:

```json
{
  "resources": {
    "file.config": {
      "resource_type": "file",
      "name": "config",
      "attributes": {
        "path": "/etc/myapp/config.ini",
        "content": "env=production",
        "size": 15
      }
    },
    "file.hosts": {
      "resource_type": "file",
      "name": "hosts",
      "attributes": {
        "path": "/etc/hosts",
        "content": "127.0.0.1 localhost",
        "size": 19
      }
    }
  }
}
```

### ขั้นที่ 7: ประกอบร่างทุกส่วนเข้าด้วยกัน

ตัวอย่าง workflow ครบ:

```rust
use infra_as_code::{
    parser::{parse, HclBlock, HclValue},
    eval::{EvalContext, Value, eval_value},
    graph::ResourceGraph,
    plan::{DesiredResource, compute_plan, StateFile},
    executor::{ApplyEngine, NullExecutor},
    state::StateStore,
};
use std::collections::HashMap;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. Parse config
    let config = r#"
variable "env" { default = "production" }
locals { prefix = "myapp" }
resource "file" "config" {
  path    = "/etc/myapp/config.ini"
  content = "env=production"
}
resource "file" "hosts" {
  path    = "/etc/hosts"
  content = "127.0.0.1 localhost"
}
"#;

    let doc = parse(config)?;

    // 2. สร้าง EvalContext จาก variable blocks
    let mut ctx = EvalContext::new();
    for block in &doc.blocks {
        if let HclBlock::Variable(v) = block {
            if let Some(default) = &v.default {
                let val = eval_value(default, &ctx)?;
                ctx.set_var(&v.name, val);
            }
        }
        if let HclBlock::Locals(l) = block {
            for (k, v) in &l.assignments {
                let val = eval_value(v, &ctx)?;
                ctx.set_local(k, val);
            }
        }
    }

    // 3. Evaluate resource blocks → DesiredResource
    let mut desired = Vec::new();
    for block in &doc.blocks {
        if let HclBlock::Resource(r) = block {
            let mut attrs = HashMap::new();
            for (k, v) in &r.attributes {
                attrs.insert(k.clone(), eval_value(v, &ctx)?);
            }
            desired.push(DesiredResource {
                resource_type: r.resource_type.clone(),
                name: r.name.clone(),
                attributes: attrs,
            });
        }
    }

    // 4. Build dependency graph
    let resource_ids: Vec<String> = desired.iter()
        .map(|r| format!("{}.{}", r.resource_type, r.name))
        .collect();
    let graph = infra_as_code::graph::build_graph(&resource_ids, |_| vec![]);
    let apply_order = graph.topological_sort()
        .map_err(|cycle| format!("cycle detected: {:?}", cycle))?;
    println!("Apply order: {:?}", apply_order);

    // 5. Compute plan
    let state = StateFile::new(); // empty state → all creates
    let plan = compute_plan(&desired, &state);
    println!("Plan: {} creates, {} updates, {} deletes",
        plan.to_create.len(), plan.to_update.len(), plan.to_delete.len());

    // 6. Apply
    let mut engine = ApplyEngine::new();
    engine.register(NullExecutor::new("file"));

    let mut store = StateStore::new();
    for change in &plan.to_create {
        let result = engine.apply(change);
        if result.success {
            store.record_applied(
                result.key.clone(),
                change.resource_type.clone(),
                change.name.clone(),
                result.applied_attributes.clone(),
            );
            println!("✓ {}: {}", result.action, result.key);
        }
    }

    println!("State has {} resources", store.state.resources.len());
    Ok(())
}
```

Output:
```
Apply order: ["file.config", "file.hosts"]
Plan: 2 creates, 0 updates, 0 deletes
✓ create: file.config
✓ create: file.hosts
State has 2 resources
```

## การทดสอบ (Testing)

### Unit Tests — 36 tests ครอบคลุมทุก module

ตัวอย่าง test สำคัญในแต่ละ module:

```rust
// parser tests
#[test]
fn test_parse_resource_block() {
    let src = r#"resource "aws_instance" "web" {
  ami           = "ami-12345"
  instance_type = "t3.micro"
  count         = 2
  enabled       = true
}"#;
    let doc = parse(src).expect("parse failed");
    if let HclBlock::Resource(r) = &doc.blocks[0] {
        assert_eq!(r.resource_type, "aws_instance");
        assert_eq!(r.name, "web");
        assert_eq!(r.attributes[2], ("count".to_string(), HclValue::Number(2.0)));
    }
}

// eval tests
#[test]
fn test_interpolate_var() {
    let ctx = ctx_with_var("env", Value::String("production".to_string()));
    let result = interpolate("Hello ${var.env}!", &ctx).unwrap();
    assert_eq!(result, "Hello production!");
}

#[test]
fn test_function_join() {
    let mut ctx = EvalContext::new();
    ctx.set_local("ports", Value::List(vec![
        Value::String("80".to_string()),
        Value::String("443".to_string()),
    ]));
    let result = eval_expr(r#"join(", ", local.ports)"#, &ctx).unwrap();
    assert_eq!(result, Value::String("80, 443".to_string()));
}

// graph tests
#[test]
fn test_graph_cycle_detection() {
    let mut g = ResourceGraph::new();
    g.add_edge("a", "b");
    g.add_edge("b", "c");
    g.add_edge("c", "a"); // cycle!
    let result = g.topological_sort();
    assert!(result.is_err(), "should detect cycle");
}

// plan tests
#[test]
fn test_plan_mixed_changes() {
    let desired = vec![
        make_desired("file", "new", &[("path", Value::String("/new".into()))]),
        make_desired("file", "existing", &[("path", Value::String("/updated".into()))]),
    ];
    // state has "existing" (old path) and "orphan" (to delete)
    let plan = compute_plan(&desired, &state);
    assert_eq!(plan.to_create.len(), 1);  // "new"
    assert_eq!(plan.to_update.len(), 1);  // "existing" (path changed)
    assert_eq!(plan.to_delete.len(), 1);  // "orphan"
}
```

### Real `cargo test` Output

```
running 36 tests
test eval::tests::test_function_format ... ok
test eval::tests::test_function_join ... ok
test eval::tests::test_eval_hcl_value_string_with_interpolation ... ok
test eval::tests::test_function_length_string ... ok
test eval::tests::test_interpolate_missing_var ... ok
test eval::tests::test_function_tostring ... ok
test eval::tests::test_interpolate_var ... ok
test eval::tests::test_interpolate_local ... ok
test executor::tests::test_apply_engine_no_executor ... ok
test executor::tests::test_apply_engine_create ... ok
test executor::tests::test_null_executor_records_create ... ok
test executor::tests::test_null_executor_records_delete ... ok
test executor::tests::test_null_executor_records_update ... ok
test graph::tests::test_dependency_edges ... ok
test graph::tests::test_graph_linear_chain ... ok
test graph::tests::test_graph_cycle_detection ... ok
test graph::tests::test_graph_diamond ... ok
test graph::tests::test_graph_no_deps ... ok
test parser::tests::test_parse_error_unknown_block ... ok
test graph::tests::test_graph_self_loop_cycle ... ok
test parser::tests::test_parse_list_value ... ok
test parser::tests::test_parse_locals_block ... ok
test parser::tests::test_parse_output_block ... ok
test parser::tests::test_parse_multiple_blocks ... ok
test parser::tests::test_parse_resource_block ... ok
test parser::tests::test_parse_variable_block ... ok
test plan::tests::test_plan_create_new_resource ... ok
test plan::tests::test_plan_delete_removed_resource ... ok
test plan::tests::test_plan_mixed_changes ... ok
test plan::tests::test_plan_no_change ... ok
test state::tests::test_drift_detection_attribute_changed ... ok
test state::tests::test_drift_detection_no_drift ... ok
test plan::tests::test_plan_update_changed_attribute ... ok
test state::tests::test_drift_detection_resource_missing ... ok
test state::tests::test_state_store_in_memory ... ok
test state::tests::test_state_save_and_load ... ok

test result: ok. 36 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests infra_as_code

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### Cargo.toml

```toml
[package]
name = "infra-as-code"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

## Pitfalls และข้อผิดพลาดที่พบบ่อย

### Pitfall 1: Parser Position ไม่อัพเดตถูกต้องเมื่อ multi-byte characters

เมื่อ input มี Unicode characters (เช่น ภาษาไทยใน string values) การ advance position ด้วย `+= 1` จะผิด:

```rust
// ❌ WRONG — breaks on multi-byte chars
fn advance(&mut self) {
    self.pos += 1;  // อาจชี้กลาง byte ของ UTF-8 char
}

// ✅ CORRECT — ใช้ char's byte length
fn advance(&mut self) -> Option<char> {
    let ch = self.remaining().chars().next()?;
    self.pos += ch.len_utf8();  // 1–4 bytes depending on char
    Some(ch)
}
```

ใช้ `str.chars()` เสมอเมื่อ iterate over characters — อย่า index โดยตรงด้วย `self.input[self.pos]` เพราะ Rust จะ panic ถ้า index อยู่กลาง multi-byte character

### Pitfall 2: Lifetime ของ Parser ทำให้ borrow checker ร้องเรียน

```rust
// ❌ ปัญหา — เก็บ &str reference แต่ self.input ก็ borrow อยู่
fn parse_current_token(&self) -> &str {
    &self.input[self.start..self.pos]  // lifetime ของ &str ผูกกับ self.input
}

// เมื่อเรียก self.advance() ข้างหน้าก็ยังถือ reference อยู่ → borrow conflict
let token = self.parse_current_token();
self.advance();  // ERROR: cannot borrow `*self` as mutable
println!("{}", token);
```

แก้โดยใช้ lifetime parameter ชัดเจน หรือ return `String` แทน `&str`:

```rust
// ✅ CORRECT — clone เพื่อหลีกเลี่ยง borrow conflict
fn parse_ident(&mut self) -> Result<String, ParseError> {
    let start = self.pos;
    while matches!(self.peek(), Some('a'..='z') | ...) {
        self.advance();
    }
    Ok(self.input[start..self.pos].to_string())  // clone → no lifetime issue
}
```

### Pitfall 3: Cycle Detection ใน Graph ที่ใช้ DFS approach อาจ false positive

DFS-based cycle detection ที่ไม่ track visited nodes อย่างถูกต้องอาจ report false cycle:

```rust
// ❌ Naive DFS — false positive ในกรณี diamond dependencies
fn has_cycle_dfs(node: &str, graph: &Graph, visited: &mut HashSet<String>) -> bool {
    if visited.contains(node) { return true; }  // ❌ ผิด! อาจเคย visit ในสาย path อื่น
    visited.insert(node.to_string());
    // ...
}
```

ใช้ Kahn's algorithm แทน หรือถ้าต้องใช้ DFS ต้องแยก `visited` (เคยดูแล้ว) กับ `in_stack` (อยู่ใน current path):

```rust
// ✅ CORRECT DFS — ใช้ 2 sets
fn has_cycle_dfs(node: &str, graph: &Graph,
    visited: &mut HashSet<String>,   // เคย process แล้ว (complete)
    in_stack: &mut HashSet<String>,  // อยู่ใน current DFS path
) -> bool {
    in_stack.insert(node.to_string());
    for neighbor in graph.neighbors(node) {
        if in_stack.contains(neighbor) { return true; }  // cycle!
        if !visited.contains(neighbor) {
            if has_cycle_dfs(neighbor, graph, visited, in_stack) { return true; }
        }
    }
    in_stack.remove(node);
    visited.insert(node.to_string());
    false
}
```

Kahn's algorithm (ที่ใช้ในโปรเจคนี้) หลีกเลี่ยงปัญหานี้ได้ทั้งหมด

### Pitfall 4: serde_json::Value กับ f64 Comparison

เมื่อ compare ค่า Number ระหว่าง `desired` และ `current` state อาจเจอปัญหา floating-point equality:

```rust
// ❌ ปัญหา: desired = 2.0 (f64), current = 2 (JSON integer)
let desired_json = serde_json::json!(2.0_f64);  // → Number(2.0)
let current_json = serde_json::json!(2_i64);    // → Number(2)
assert_eq!(desired_json, current_json);          // ✅ serde_json ทำ OK

// แต่ถ้า desired = 2.000000001 (floating point เล็กน้อย)
let desired_json = serde_json::json!(2.000000001_f64);
let current_json = serde_json::json!(2.0_f64);
// != เพราะ serde_json เก็บทศนิยมตรง ๆ
```

วิธีแก้: normalize ก่อน compare — ถ้าทั้งคู่เป็น Number ใช้ `abs_diff_eq` หรือ round ให้ precision เดียวกัน:

```rust
fn json_values_equal(a: &serde_json::Value, b: &serde_json::Value) -> bool {
    match (a, b) {
        (serde_json::Value::Number(na), serde_json::Value::Number(nb)) => {
            match (na.as_f64(), nb.as_f64()) {
                (Some(fa), Some(fb)) => (fa - fb).abs() < 1e-9,
                _ => a == b,
            }
        }
        _ => a == b,
    }
}
```

### Pitfall 5: Executor trait ต้อง `Send + Sync` สำหรับ async apply

ถ้าต้องการ apply resources แบบ parallel (async/await), executor ต้องเป็น `Send + Sync`:

```rust
// ❌ ปัญหา: ถ้า Executor ไม่ใช่ Send+Sync
pub trait Executor {  // ขาด Send + Sync
    fn create(&mut self, ...) -> Result<...>;
}

// Box<dyn Executor> ไม่สามารถส่งข้าม thread ได้
// tokio::spawn(async move { engine.apply(change) })  ← ERROR

// ✅ CORRECT: เพิ่ม Send + Sync bound
pub trait Executor: Send + Sync {
    fn create(&mut self, ...) -> Result<...>;
}
```

NullExecutor และ FileExecutor ที่เราเขียนไม่มี raw pointer หรือ non-Send type จึง auto-implement `Send + Sync` โดยอัตโนมัติ

### Pitfall 6: String Interpolation กับ Nested Braces

Expression อย่าง `"${format("prefix-%s", var.env)}"` มี `{` และ `}` ซ้อนกัน ถ้า track depth ไม่ถูก:

```rust
// ❌ ผิด: หยุดที่ } แรกที่เจอ ไม่ track depth
while let Some(c) = chars.next() {
    if c == '}' { break; }  // ❌ จะหยุดที่ } ของ "prefix-%s" แทน
    expr.push(c);
}

// ✅ ถูก: track depth
let mut depth = 1;
for ec in chars.by_ref() {
    if ec == '{' { depth += 1; expr.push(ec); }
    else if ec == '}' {
        depth -= 1;
        if depth == 0 { break; }  // หยุดเมื่อปิด brace ของ ${...} จริง
        expr.push(ec);
    } else {
        expr.push(ec);
    }
}
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# ขนาด binary
ls -lh target/release/infra-as-code
# ประมาณ 2-5 MB (ขึ้นกับ features)

# Strip symbols เพื่อลดขนาด
strip target/release/infra-as-code
```

### Docker Image

```dockerfile
# Multi-stage build
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src/ ./src/

# Cache dependencies
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/infra-as-code /usr/local/bin/iac

ENTRYPOINT ["iac"]
CMD ["--help"]
```

```bash
docker build -t iac-engine:latest .
docker run -v $(pwd)/config:/config iac-engine plan /config/main.hcl
```

### CLI Interface (ขยายจาก library)

```toml
# Cargo.toml
[[bin]]
name = "iac"
path = "src/main.rs"
```

```rust
// src/main.rs
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    match args.get(1).map(|s| s.as_str()) {
        Some("plan")  => cmd_plan(&args[2..]),
        Some("apply") => cmd_apply(&args[2..]),
        Some("show")  => cmd_show(&args[2..]),
        _ => {
            eprintln!("Usage: iac <plan|apply|show> [config.hcl]");
            std::process::exit(1);
        }
    }
}
```

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม `depends_on` Meta-Argument

ใน Terraform เราสามารถระบุ dependencies ชัดเจนด้วย `depends_on`:

```hcl
resource "file" "app_config" {
  path       = "/etc/app.conf"
  content    = "version=1"
  depends_on = ["file.base_dir"]
}
```

งาน:
1. เพิ่ม `depends_on` เป็น special attribute ที่ parser รู้จัก
2. ใน resource evaluation loop ดึง `depends_on` ออกจาก attributes ก่อน evaluate
3. Build graph edges จาก `depends_on` list
4. เขียน test: resource ที่มี `depends_on` ต้องมี edge ใน graph และอยู่ลำดับหลัง dependency

### Exercise 2: รองรับ `count` Meta-Argument

Terraform รองรับ `count` เพื่อสร้าง resource หลายชิ้นจาก block เดียว:

```hcl
resource "file" "log" {
  count   = 3
  path    = "/var/log/app-${count.index}.log"
  content = ""
}
# สร้าง: file.log[0], file.log[1], file.log[2]
```

งาน:
1. เพิ่ม `count.index` ใน `EvalContext` ให้สามารถ interpolate ได้
2. ใน resource expansion loop clone ResourceBlock ตาม `count` value
3. Key format: `"type.name[index]"` แทน `"type.name"`
4. เขียน tests: `count = 3` ผลิต 3 `DesiredResource` ที่มี unique keys

### Exercise 3: เพิ่ม `data` Source Blocks

Data sources อ่านข้อมูลจากโลกจริง (ไม่สร้าง resource ใหม่):

```hcl
data "file" "existing_config" {
  path = "/etc/os-release"
}

resource "file" "app" {
  path    = "/etc/app.conf"
  content = "os=${data.file.existing_config.content}"
}
```

งาน:
1. เพิ่ม `DataBlock` ใน `HclBlock` enum
2. สร้าง `DataSource` trait คล้าย `Executor` แต่มีแค่ `read()` method
3. `FileDataSource` อ่านไฟล์จริงและ return attributes
4. Populate `EvalContext.resource_outputs` ด้วยผลลัพธ์จาก data sources
5. เพิ่ม `data.type.name.attr` resolution ใน `eval_expr()`

### Exercise 4: Parallel Apply ด้วย Rayon

Resources ที่ไม่มี dependency กันสามารถ apply พร้อมกันได้:

```rust
// ปัจจุบัน: sequential
for change in &plan.to_create {
    engine.apply(change);
}

// เป้าหมาย: parallel apply สำหรับ independent resources
use rayon::prelude::*;
let results: Vec<ApplyResult> = plan.to_create
    .par_iter()  // parallel iterator
    .map(|change| {
        // ต้องมี thread-safe executor
        executor.apply(change)
    })
    .collect();
```

งาน:
1. เพิ่ม `rayon` dependency
2. ปรับ `Executor::create()` ให้ `&self` แทน `&mut self` (หรือใช้ `Mutex<Executor>`)
3. Group resources ตาม topological "level" — resources ที่อยู่ level เดียวกัน apply พร้อมกัน
4. Benchmark sequential vs parallel apply ด้วย 100 file resources

### Exercise 5: Import Existing Resources เข้า State

ใน Terraform มีคำสั่ง `import` เพื่อนำ resource ที่มีอยู่แล้วเข้า state file:

```bash
iac import file.config /etc/app.conf
```

งาน:
1. สร้าง `import` command ใน CLI
2. รับ resource key และ resource ID เป็น arguments
3. ใช้ executor ที่ตรงกัน read current attributes ของ resource
4. เขียน ResourceState เข้า state file
5. หลัง import รัน `plan` — ต้องไม่มี changes ถ้า config ตรงกับ resource จริง

### Exercise 6: HCL Modules

Modules ทำให้ reuse config ได้ระหว่างโปรเจค:

```hcl
module "app_config" {
  source = "./modules/app"

  app_name = "myapp"
  env      = var.env
}
```

งาน:
1. เพิ่ม `ModuleBlock` ใน parser
2. Load และ parse `source` path เป็น HclDocument แยก
3. Module variables รับค่าจาก call site แทน `variable.default`
4. Namespace resources: `module.app_config.file.config` แทน `file.config`
5. เพิ่ม integration test ที่ใช้ module จากไดเรกทอรีอื่น

## สรุป

โปรเจคนี้สร้าง IaC engine ที่มีองค์ประกอบครบถ้วน:

**Parser** ที่เขียน recursive descent ด้วยมือสอนให้เข้าใจว่า config language ทำงานอย่างไร — ไม่ใช่แค่ regex แต่เป็น proper grammar parsing ที่จัดการ nesting, escaping, และ error reporting

**Expression Evaluator** สอน string template interpolation ซึ่งเป็นพื้นฐานของ templating engine ทุกชนิด ตั้งแต่ HCL ไปจนถึง Jinja2 และ Handlebars

**Resource Graph** สอน graph algorithms จริงในบริบทที่ใช้งานได้จริง — topological sort ไม่ใช่แค่ leetcode problem แต่เป็น core algorithm ที่ build systems ทุกตัวใช้

**Plan Engine** สอน diffing algorithm ที่เรียบง่ายแต่ทรงพลัง — เปรียบเทียบสองสถานะแล้วหา minimal change set เป็น pattern ที่ใช้ใน React (Virtual DOM diff), Kubernetes (desired vs actual state), และ Git

**Executor trait** สอน strategy pattern ที่ทำให้ engine extensible โดยไม่ต้อง modify core — เพิ่ม AWS/GCP/Kubernetes executor ได้ง่ายโดย implement trait เดียว

**State Management** สอนการ serialize/deserialize complex state และ detect drift ซึ่งเป็นปัญหาที่ทุก production system เจอเมื่อ real world ไม่ตรงกับ expected state

Pattern สำคัญที่ได้เรียน:
- **Recursive descent parsing** — อ่านและ parse structured text แบบ hand-written
- **Trait objects สำหรับ extensibility** — `Box<dyn Executor>` ทำให้ engine pluggable
- **DAG + topological sort** — จัดการ dependencies อย่างถูกต้อง
- **Declarative diff** — เปรียบเทียบ desired vs actual เพื่อหา minimal changes
- **serde สำหรับ state persistence** — serialize/deserialize state ลง JSON

โปรเจคถัดไป G09 จะสร้าง **Rate Limiter** ซึ่งเน้นไปที่ concurrent access patterns, atomic operations, และ time-window algorithms — ทักษะที่สำคัญสำหรับ production-grade middleware

---

**โปรเจคก่อนหน้า:** [Project G07: Secret Scanner](project-g07-secret-scanner.md) | **โปรเจคถัดไป:** [Project G09: Rate Limiter](project-g09-rate-limiter.md)
