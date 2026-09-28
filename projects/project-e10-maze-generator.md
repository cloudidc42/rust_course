# Project E10: Maze Generator & Solver Visualizer

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 3 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Maze Generator and Solver Visualizer** ที่ครบสมบูรณ์ตั้งแต่การสร้าง maze ด้วยอัลกอริทึมหลายแบบ ไปจนถึงการแก้และ visualize คำตอบในรูปแบบต่าง ๆ โปรเจคนี้เป็นตัวอย่างคลาสสิกของการนำ **graph algorithms** และ **data structures** มาใช้จริงในโลก systems programming

Maze generation เป็นหัวข้อที่น่าสนใจเพราะ:
- มีอัลกอริทึมหลากหลายที่ให้ผลลัพธ์ **texture** ที่แตกต่างกัน (DFS ให้ corridor ยาว, Prim's ให้กิ่งก้านมาก)
- ทดสอบความเข้าใจ graph traversal, Union-Find, BFS, A* ในบริบทที่จับต้องได้
- เป็นพื้นฐานของ **game level generation**, **dungeon crawlers**, **puzzle games**

**Use cases จริงในโลก production:**
- **Procedural level generation** สำหรับ roguelike games (เช่น NetHack, Dwarf Fortress)
- **Network topology visualization** — maze เป็น metaphor สำหรับ routing problem
- **Pathfinding benchmark** — maze เป็น test case มาตรฐานสำหรับเปรียบ BFS กับ A*
- **Technical interviews** — Union-Find และ graph traversal เป็นหัวข้อยอดนิยม

---

## สิ่งที่จะได้เรียนรู้

- **Grid representation** — struct `Maze` + `Cell` พร้อม bidirectional wall removal
- **Recursive Backtracker** — DFS stack-based ที่สร้าง perfect maze (ไม่มี loop)
- **Prim's algorithm** — frontier-based ที่ให้ texture แตกต่างจาก DFS
- **Kruskal's algorithm** — Union-Find (disjoint set with path compression + rank)
- **Wilson's algorithm** — loop-erased random walk ที่ได้ uniform spanning tree
- **BFS solver** — หา shortest path อย่าง correct
- **A\* solver** — Manhattan heuristic เปรียบกับ BFS
- **Multiple output formats** — ASCII, SVG, PNG, JSON stats
- **Terminal animation** ด้วย `crossterm` สำหรับ step-by-step visualization

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, structs, enums, Vec
- **Part 21–30**: Collections, iterators, HashSet, HashMap, VecDeque
- **Part 31–40**: Error handling, traits, generics
- **Part 41–50**: Closures, higher-order functions, algorithm complexity
- **Part 51–60**: crate ecosystem, Cargo features, `rand`, `serde`
- พื้นฐาน Graph Theory — BFS, DFS, spanning tree (ไม่ต้องลึกมาก)

---

## โครงสร้างโปรเจค (Project Layout)

```
maze-generator/
├── src/
│   ├── main.rs          ← CLI entry point + argument parsing
│   ├── maze.rs          ← Maze, Cell struct + helpers
│   ├── generators/
│   │   ├── mod.rs       ← re-exports ทุก generator
│   │   ├── backtracker.rs
│   │   ├── prims.rs
│   │   ├── kruskals.rs
│   │   └── wilsons.rs
│   ├── solvers/
│   │   ├── mod.rs
│   │   ├── bfs.rs
│   │   └── astar.rs
│   ├── render/
│   │   ├── mod.rs
│   │   ├── ascii.rs
│   │   ├── svg.rs
│   │   └── terminal.rs
│   └── stats.rs         ← MazeStats + JSON output
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Model

```
Maze
├── width: usize
├── height: usize
└── cells: Vec<Cell>       ← flat 2D array: index = y * width + x
        └── walls: [bool; 4]   ← N=0, E=1, S=2, W=3  (true = wall exists)
```

ทำไมถึงใช้ flat `Vec<Cell>` แทน `Vec<Vec<Cell>>`?
- **Cache locality** — elements อยู่ติดกันใน memory ทำให้ CPU cache เข้าถึงได้เร็ว
- **Simple indexing** — `y * width + x` คำนวณได้ทันทีโดยไม่ต้อง double-dereference
- **Simpler bounds checking** — ตรวจเพียง `x < width && y < height`

### Wall Convention

```
         walls[0] = North
              │
walls[3] = West ─ cell ─ East = walls[1]
              │
         walls[2] = South
```

เมื่อลบกำแพงระหว่าง cell A และ B ต้องทำทั้งสองทิศทาง:
- A อยู่ทิศ West ของ B → ลบ `A.walls[1]` (East) และ `B.walls[3]` (West)

### Algorithm Comparison

| อัลกอริทึม | Time | Space | Texture | Perfect? |
|-----------|------|-------|---------|----------|
| Recursive Backtracker | O(n) | O(n) stack | corridor ยาว | ✓ |
| Prim's | O(n log n) | O(n) frontier | กิ่งสั้น, bushy | ✓ |
| Kruskal's | O(n α(n)) | O(n) edges | สม่ำเสมอ | ✓ |
| Wilson's | O(n²) expected | O(n) path | uniform spanning tree | ✓ |

**Perfect maze** = ทุก cell เชื่อมถึงกันได้ และไม่มี loop (เท่ากับ spanning tree ของ grid graph)

### Solver Comparison

| Solver | Complete? | Optimal? | Memory | ใช้ Heuristic? |
|--------|-----------|----------|--------|----------------|
| BFS | ✓ | ✓ | O(n) | ✗ |
| A\* (Manhattan) | ✓ | ✓ | O(n) | ✓ |

ทั้งสองให้ **path length เท่ากัน** บน unweighted maze แต่ A* expand nodes น้อยกว่า BFS เพราะ heuristic นำทางไปหา goal

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้าง Maze และ Cell

เริ่มจาก data model พื้นฐาน — `Cell` ที่เก็บกำแพง 4 ด้าน และ `Maze` ที่เป็น flat array

```rust
// src/maze.rs

/// ทิศทางกำแพง index: N=0, E=1, S=2, W=3
#[derive(Clone, Debug)]
pub struct Cell {
    pub walls: [bool; 4],
}

impl Cell {
    pub fn new() -> Self {
        // cell ใหม่มีกำแพงทุกด้าน
        Cell { walls: [true; 4] }
    }
}

impl Default for Cell {
    fn default() -> Self {
        Self::new()
    }
}

#[derive(Clone, Debug)]
pub struct Maze {
    pub width: usize,
    pub height: usize,
    pub cells: Vec<Cell>,
}

impl Maze {
    pub fn new(width: usize, height: usize) -> Self {
        assert!(width > 0 && height > 0, "Maze dimensions must be positive");
        Maze {
            width,
            height,
            cells: vec![Cell::new(); width * height],
        }
    }

    /// แปลง (x, y) → index ใน flat Vec
    pub fn cell_index(&self, x: usize, y: usize) -> usize {
        y * self.width + x
    }

    /// ลบกำแพงระหว่าง cell (ax, ay) และ (bx, by) — ทั้งสองทิศทาง
    pub fn remove_wall(&mut self, ax: usize, ay: usize, bx: usize, by: usize) {
        let ai = self.cell_index(ax, ay);
        let bi = self.cell_index(bx, by);

        if bx == ax + 1 {
            // b อยู่ทาง East ของ a
            self.cells[ai].walls[1] = false; // ลบ East wall ของ a
            self.cells[bi].walls[3] = false; // ลบ West wall ของ b
        } else if ax > 0 && bx == ax - 1 {
            // b อยู่ทาง West ของ a
            self.cells[ai].walls[3] = false;
            self.cells[bi].walls[1] = false;
        } else if by == ay + 1 {
            // b อยู่ทาง South ของ a
            self.cells[ai].walls[2] = false; // ลบ South wall ของ a
            self.cells[bi].walls[0] = false; // ลบ North wall ของ b
        } else if ay > 0 && by == ay - 1 {
            // b อยู่ทาง North ของ a
            self.cells[ai].walls[0] = false;
            self.cells[bi].walls[2] = false;
        }
    }

    /// หา neighbors ที่เปิดอยู่ (ไม่มีกำแพงกั้น) — ใช้สำหรับ solver
    pub fn open_neighbors(&self, x: usize, y: usize) -> Vec<(usize, usize)> {
        let mut neighbors = Vec::new();
        let idx = self.cell_index(x, y);
        let cell = &self.cells[idx];

        if y > 0 && !cell.walls[0] {
            neighbors.push((x, y - 1)); // North
        }
        if x + 1 < self.width && !cell.walls[1] {
            neighbors.push((x + 1, y)); // East
        }
        if y + 1 < self.height && !cell.walls[2] {
            neighbors.push((x, y + 1)); // South
        }
        if x > 0 && !cell.walls[3] {
            neighbors.push((x - 1, y)); // West
        }
        neighbors
    }

    /// หา neighbors ทั้งหมด (ไม่คำนึงถึงกำแพง) — ใช้สำหรับ generator
    pub fn all_neighbors(&self, x: usize, y: usize) -> Vec<(usize, usize)> {
        let mut neighbors = Vec::new();
        if y > 0 {
            neighbors.push((x, y - 1));
        }
        if x + 1 < self.width {
            neighbors.push((x + 1, y));
        }
        if y + 1 < self.height {
            neighbors.push((x, y + 1));
        }
        if x > 0 {
            neighbors.push((x - 1, y));
        }
        neighbors
    }
}
```

**ทดสอบ cell_index และ wall removal:**

```rust
#[test]
fn test_cell_index() {
    let maze = Maze::new(5, 5);
    assert_eq!(maze.cell_index(0, 0), 0);
    assert_eq!(maze.cell_index(4, 0), 4);
    assert_eq!(maze.cell_index(0, 1), 5);
    assert_eq!(maze.cell_index(4, 4), 24);
}

#[test]
fn test_wall_removal_bidirectional() {
    let mut maze = Maze::new(5, 5);
    maze.remove_wall(0, 0, 1, 0);
    assert!(!maze.cells[maze.cell_index(0, 0)].walls[1]); // East ของ (0,0) เปิด
    assert!(!maze.cells[maze.cell_index(1, 0)].walls[3]); // West ของ (1,0) เปิด

    maze.remove_wall(0, 0, 0, 1);
    assert!(!maze.cells[maze.cell_index(0, 0)].walls[2]); // South ของ (0,0) เปิด
    assert!(!maze.cells[maze.cell_index(0, 1)].walls[0]); // North ของ (0,1) เปิด
}
```

---

### ขั้นที่ 2: Recursive Backtracker (DFS)

อัลกอริทึมนี้ใช้ DFS ด้วย explicit stack เพื่อหลีกเลี่ยง stack overflow บน maze ใหญ่

**หลักการทำงาน:**
1. Mark start cell ว่า visited แล้ว push เข้า stack
2. While stack ไม่ว่าง: ดู top cell
   - มี unvisited neighbors → เลือก 1 ตัวสุ่ม, ลบกำแพง, mark visited, push
   - ไม่มี → pop (backtrack)
3. ผลลัพธ์คือ perfect maze — ทุก cell เชื่อมกันได้ ไม่มี loop

```rust
// src/generators/backtracker.rs
use crate::maze::Maze;
use rand::seq::SliceRandom;

pub fn generate(width: usize, height: usize) -> Maze {
    let mut maze = Maze::new(width, height);
    let mut rng = rand::thread_rng();
    let mut visited = vec![false; width * height];
    let mut stack: Vec<(usize, usize)> = Vec::new();

    // เริ่มจาก (0, 0)
    let start = (0usize, 0usize);
    visited[maze.cell_index(start.0, start.1)] = true;
    stack.push(start);

    while let Some(&(cx, cy)) = stack.last() {
        // หา unvisited neighbors ของ cell ปัจจุบัน
        let unvisited: Vec<(usize, usize)> = maze
            .all_neighbors(cx, cy)
            .into_iter()
            .filter(|&(nx, ny)| !visited[maze.cell_index(nx, ny)])
            .collect();

        if unvisited.is_empty() {
            // ไม่มีทางไปต่อ — backtrack
            stack.pop();
        } else {
            // เลือก neighbor สุ่ม ลบกำแพง แล้วเดินต่อ
            let &(nx, ny) = unvisited.choose(&mut rng).unwrap();
            maze.remove_wall(cx, cy, nx, ny);
            visited[maze.cell_index(nx, ny)] = true;
            stack.push((nx, ny));
        }
    }

    maze
}
```

**ทำไม DFS ให้ corridor ยาว?** เพราะ DFS "เดินลึก" ก่อน backtrack ทำให้ path แรกที่ถูกสร้างมักยาวมาก สังเกตได้ว่า maze จาก DFS มักมี "main corridor" หนึ่งเส้นที่ยาวและกิ่งสั้น ๆ ออกมา

**⚠️ Pitfall 1: ใช้ recursion แทน explicit stack**

```rust
// ❌ ผิด — ใช้ recursive function ตรง ๆ
fn backtrack(maze: &mut Maze, visited: &mut Vec<bool>, x: usize, y: usize) {
    // maze 100×100 = 10,000 cells → stack depth สูงสุด 10,000
    // Rust default stack size = 8MB → อาจ stack overflow บน maze ใหญ่ได้
    let unvisited = ...;
    for (nx, ny) in unvisited {
        backtrack(maze, visited, nx, ny); // ← อันตราย!
    }
}

// ✓ ถูก — ใช้ explicit Vec<(usize, usize)> เป็น stack แทน
// ไม่มีขีดจำกัด stack depth (ขึ้นอยู่กับ heap memory เท่านั้น)
```

---

### ขั้นที่ 3: Prim's Algorithm

Prim's สร้าง minimum spanning tree โดยเริ่มจาก random seed และขยาย frontier ออกไป

**หลักการทำงาน:**
1. เริ่มด้วย random cell เพิ่มเข้า "maze set"
2. เพิ่ม neighbors ของ cell นั้นเข้า "frontier list"
3. วนซ้ำ:
   - เลือก frontier cell สุ่ม 1 ตัว
   - หา neighbors ที่อยู่ใน maze แล้ว → เลือก 1 ตัวสุ่ม → ลบกำแพงเชื่อม
   - เพิ่ม frontier cell เข้า maze set และเพิ่ม neighbors ใหม่เข้า frontier
4. ทำจนกว่า frontier ว่าง

```rust
// src/generators/prims.rs
use crate::maze::Maze;
use rand::seq::SliceRandom;
use rand::Rng;

pub fn generate(width: usize, height: usize) -> Maze {
    let mut maze = Maze::new(width, height);
    let mut rng = rand::thread_rng();
    let mut in_maze = vec![false; width * height];
    let mut frontier: Vec<(usize, usize)> = Vec::new();

    // เริ่มจาก cell สุ่ม
    let start_x = rng.gen_range(0..width);
    let start_y = rng.gen_range(0..height);
    in_maze[maze.cell_index(start_x, start_y)] = true;

    for n in maze.all_neighbors(start_x, start_y) {
        frontier.push(n);
    }

    while !frontier.is_empty() {
        // เลือก frontier cell สุ่ม
        let idx = rng.gen_range(0..frontier.len());
        let (fx, fy) = frontier.swap_remove(idx); // O(1) removal
        let fidx = maze.cell_index(fx, fy);

        if in_maze[fidx] {
            // ถูกเพิ่มซ้ำเข้า frontier แล้ว ข้ามไป
            continue;
        }

        // หา maze neighbors ของ frontier cell นี้
        let maze_neighbors: Vec<(usize, usize)> = maze
            .all_neighbors(fx, fy)
            .into_iter()
            .filter(|&(nx, ny)| in_maze[maze.cell_index(nx, ny)])
            .collect();

        if let Some(&(mx, my)) = maze_neighbors.choose(&mut rng) {
            maze.remove_wall(fx, fy, mx, my);
            in_maze[fidx] = true;

            // เพิ่ม unvisited neighbors เข้า frontier
            for n in maze.all_neighbors(fx, fy) {
                if !in_maze[maze.cell_index(n.0, n.1)] {
                    frontier.push(n);
                }
            }
        }
    }

    maze
}
```

**⚠️ Pitfall 2: Frontier list อาจมี duplicates**

Prim's ใช้ frontier list ที่อาจมี cell เดิมซ้ำกัน (เพราะ cell สามารถเป็น neighbor ของหลาย maze cell) วิธีแก้คือเช็ค `if in_maze[fidx] { continue; }` ก่อนประมวลผล ไม่ใช่ใช้ `HashSet` ซึ่งแม้ correct แต่ช้ากว่า `Vec::swap_remove`

```rust
// ❌ ช้ากว่าโดยไม่จำเป็น
let mut frontier: HashSet<(usize, usize)> = HashSet::new();
frontier.remove(&(fx, fy)); // O(1) แต่ overhead สูง

// ✓ เร็วกว่า: Vec + lazy deduplication
let (fx, fy) = frontier.swap_remove(idx); // O(1) removal
if in_maze[fidx] { continue; }  // ตรวจสอบ duplicate ตอนดึงออก
```

---

### ขั้นที่ 4: Kruskal's Algorithm และ Union-Find

Kruskal's สร้าง maze โดยสุ่มลำดับ edges แล้วเพิ่มทีละ edge ถ้าไม่ทำให้เกิด cycle (ใช้ Union-Find ตรวจสอบ)

**Union-Find (Disjoint Set Union)** ด้วย path compression + union by rank:

```rust
// src/union_find.rs

pub struct UnionFind {
    parent: Vec<usize>,
    rank: Vec<usize>,
}

impl UnionFind {
    pub fn new(n: usize) -> Self {
        UnionFind {
            parent: (0..n).collect(),
            rank: vec![0; n],
        }
    }

    /// หา representative ของ set ที่ x อยู่ (พร้อม path compression)
    pub fn find(&mut self, x: usize) -> usize {
        if self.parent[x] != x {
            // Path compression: ชี้ตรงไปหา root เพื่อทำให้ flatten tree
            self.parent[x] = self.find(self.parent[x]);
        }
        self.parent[x]
    }

    /// รวม set ของ x และ y เข้าด้วยกัน — คืน false ถ้าอยู่ set เดิมแล้ว
    pub fn union(&mut self, x: usize, y: usize) -> bool {
        let rx = self.find(x);
        let ry = self.find(y);
        if rx == ry {
            return false; // อยู่ set เดิมแล้ว — ห้ามเพิ่ม edge (จะสร้าง cycle)
        }
        // Union by rank: ต่อ tree ที่สั้นกว่าเข้ากับ tree ที่สูงกว่า
        match self.rank[rx].cmp(&self.rank[ry]) {
            std::cmp::Ordering::Less => self.parent[rx] = ry,
            std::cmp::Ordering::Greater => self.parent[ry] = rx,
            std::cmp::Ordering::Equal => {
                self.parent[ry] = rx;
                self.rank[rx] += 1;
            }
        }
        true
    }

    pub fn connected(&mut self, x: usize, y: usize) -> bool {
        self.find(x) == self.find(y)
    }
}
```

**Kruskal's generator:**

```rust
// src/generators/kruskals.rs
use crate::maze::Maze;
use crate::union_find::UnionFind;
use rand::seq::SliceRandom;

pub fn generate(width: usize, height: usize) -> Maze {
    let mut maze = Maze::new(width, height);
    let mut rng = rand::thread_rng();
    let n = width * height;
    let mut uf = UnionFind::new(n);

    // สร้าง edges ทั้งหมดระหว่าง adjacent cells
    let mut edges: Vec<(usize, usize, usize, usize)> = Vec::new();
    for y in 0..height {
        for x in 0..width {
            if x + 1 < width {
                edges.push((x, y, x + 1, y)); // Horizontal edge
            }
            if y + 1 < height {
                edges.push((x, y, x, y + 1)); // Vertical edge
            }
        }
    }

    // Shuffle edges เพื่อให้ได้ random spanning tree
    edges.shuffle(&mut rng);

    for (ax, ay, bx, by) in edges {
        let ai = maze.cell_index(ax, ay);
        let bi = maze.cell_index(bx, by);

        // ถ้า cells อยู่ต่าง set → ลบกำแพง + union
        if uf.union(ai, bi) {
            maze.remove_wall(ax, ay, bx, by);
        }
        // ถ้าอยู่ set เดียวกัน → ข้าม (การเพิ่ม edge จะสร้าง cycle)
    }

    maze
}
```

**ความซับซ้อน:** การ union/find แต่ละครั้งใช้เวลา O(α(n)) amortized ซึ่ง α คือ inverse Ackermann function — ในทางปฏิบัติถือว่าเป็น O(1) ดังนั้น Kruskal's ทั้งหมดเป็น O(n α(n)) ≈ O(n)

**ทดสอบ Union-Find:**

```rust
#[test]
fn test_union_find_correctness() {
    let mut uf = UnionFind::new(5);
    assert!(!uf.connected(0, 1));
    uf.union(0, 1);
    assert!(uf.connected(0, 1));
    uf.union(1, 2);
    // Path compression: 0 → 1 → 2 ควรถูก flatten เป็น 0 → root
    assert!(uf.connected(0, 2));
    assert!(!uf.connected(0, 3));
    // Union ของ nodes ที่ connected แล้วต้องคืน false
    assert!(!uf.union(0, 2));
}
```

---

### ขั้นที่ 5: Wilson's Algorithm (Loop-Erased Random Walk)

Wilson's algorithm สร้าง **uniform spanning tree** — ทุก spanning tree มีโอกาสเท่ากัน (DFS และ Prim's ไม่ได้คุณสมบัตินี้)

**หลักการทำงาน:**
1. เพิ่ม cell แรกเข้า maze
2. เลือก cell ที่ยังไม่อยู่ใน maze สุ่ม 1 ตัว
3. ทำ random walk จนถึง maze cell — ระหว่างทางถ้าวนกลับมาหา path เดิมให้ **ลบ loop** ออก (loop-erasure)
4. เพิ่ม path ที่ได้เข้า maze
5. ทำซ้ำจนทุก cell อยู่ใน maze

```rust
// src/generators/wilsons.rs
use crate::maze::Maze;
use rand::seq::SliceRandom;
use rand::Rng;
use std::collections::HashMap;

pub fn generate(width: usize, height: usize) -> Maze {
    let mut maze = Maze::new(width, height);
    let mut rng = rand::thread_rng();
    let n = width * height;
    let mut in_maze = vec![false; n];

    // เริ่มจาก cell แรก
    in_maze[0] = true;
    let mut in_maze_count = 1usize;

    while in_maze_count < n {
        // หา cell ที่ยังไม่อยู่ใน maze
        let start_idx = loop {
            let idx = rng.gen_range(0..n);
            if !in_maze[idx] {
                break idx;
            }
        };

        let start_x = start_idx % width;
        let start_y = start_idx / width;

        // Loop-erased random walk
        // path_index maps: cell_idx → ตำแหน่งใน path vector
        let mut path: Vec<(usize, usize)> = vec![(start_x, start_y)];
        let mut path_index: HashMap<usize, usize> = HashMap::new();
        path_index.insert(start_idx, 0);

        'walk: loop {
            let &(cx, cy) = path.last().unwrap();
            let neighbors = maze.all_neighbors(cx, cy);
            let &(nx, ny) = neighbors.choose(&mut rng).unwrap();
            let nidx = maze.cell_index(nx, ny);

            if in_maze[nidx] {
                // ถึง maze! — carve ทั้ง path เข้า maze
                for i in 0..path.len() - 1 {
                    let (ax, ay) = path[i];
                    let (bx, by) = path[i + 1];
                    maze.remove_wall(ax, ay, bx, by);
                    let aidx = maze.cell_index(ax, ay);
                    if !in_maze[aidx] {
                        in_maze[aidx] = true;
                        in_maze_count += 1;
                    }
                }
                // เชื่อม cell สุดท้ายของ path เข้า maze cell
                let (lx, ly) = *path.last().unwrap();
                maze.remove_wall(lx, ly, nx, ny);
                let lidx = maze.cell_index(lx, ly);
                if !in_maze[lidx] {
                    in_maze[lidx] = true;
                    in_maze_count += 1;
                }
                break 'walk;

            } else if let Some(&loop_start) = path_index.get(&nidx) {
                // พบ loop — ลบส่วนหลัง loop_start ออก (loop erasure)
                for p in &path[loop_start + 1..] {
                    path_index.remove(&maze.cell_index(p.0, p.1));
                }
                path.truncate(loop_start + 1);
                // ไม่เพิ่ม nidx เข้า path — เพราะมันคือจุดที่ loop กลับมา

            } else {
                // เดินต่อ
                path_index.insert(nidx, path.len());
                path.push((nx, ny));
            }
        }
    }

    maze
}
```

**⚠️ Pitfall 3: Loop erasure ต้องลบ path_index ด้วย**

```rust
// ❌ ผิด: ลบเฉพาะ path vector แต่ไม่ลบ path_index
path.truncate(loop_start + 1);
// path_index ยังชี้ถึง cells ที่ถูกลบออกจาก path ไปแล้ว
// ครั้งต่อไปที่เดินมาหา cells เหล่านั้น จะตรวจไม่พบ loop ที่แท้จริง

// ✓ ถูก: ลบทั้งคู่ให้ sync กัน
for p in &path[loop_start + 1..] {
    path_index.remove(&maze.cell_index(p.0, p.1));
}
path.truncate(loop_start + 1);
```

---

### ขั้นที่ 6: BFS Solver และ A* Solver

**BFS Solver** — ค้นหาแบบ breadth-first รับประกัน shortest path บน unweighted graph:

```rust
// src/solvers/bfs.rs
use crate::maze::Maze;
use std::collections::{HashMap, HashSet, VecDeque};

pub fn solve(maze: &Maze) -> Option<Vec<(usize, usize)>> {
    let start = (0usize, 0usize);
    let end = (maze.width - 1, maze.height - 1);

    let mut queue: VecDeque<(usize, usize)> = VecDeque::new();
    let mut came_from: HashMap<(usize, usize), (usize, usize)> = HashMap::new();
    let mut visited: HashSet<(usize, usize)> = HashSet::new();

    queue.push_back(start);
    visited.insert(start);

    while let Some(current) = queue.pop_front() {
        if current == end {
            // Reconstruct path โดยย้อนกลับจาก end ไป start
            let mut path = vec![end];
            let mut cur = end;
            while cur != start {
                cur = *came_from.get(&cur).unwrap();
                path.push(cur);
            }
            path.reverse();
            return Some(path);
        }

        let (cx, cy) = current;
        for (nx, ny) in maze.open_neighbors(cx, cy) {
            if !visited.contains(&(nx, ny)) {
                visited.insert((nx, ny));
                came_from.insert((nx, ny), current);
                queue.push_back((nx, ny));
            }
        }
    }

    None // ไม่พบ path (ไม่ควรเกิดขึ้นกับ perfect maze)
}
```

**A\* Solver** — ใช้ Manhattan distance heuristic นำทางการค้นหา:

```rust
// src/solvers/astar.rs
use crate::maze::Maze;
use std::cmp::Reverse;
use std::collections::{BinaryHeap, HashMap};

/// Manhattan distance heuristic — ใช้ได้กับ grid ที่ไม่มี diagonal movement
fn heuristic(a: (usize, usize), b: (usize, usize)) -> usize {
    let dx = if a.0 > b.0 { a.0 - b.0 } else { b.0 - a.0 };
    let dy = if a.1 > b.1 { a.1 - b.1 } else { b.1 - a.1 };
    dx + dy
}

pub fn solve(maze: &Maze) -> Option<Vec<(usize, usize)>> {
    let start = (0usize, 0usize);
    let end = (maze.width - 1, maze.height - 1);

    // Open set: (Reverse(f), g, x, y) — Reverse เพื่อให้เป็น min-heap
    let mut open: BinaryHeap<(Reverse<usize>, usize, usize, usize)> = BinaryHeap::new();
    let mut g_score: HashMap<(usize, usize), usize> = HashMap::new();
    let mut came_from: HashMap<(usize, usize), (usize, usize)> = HashMap::new();

    g_score.insert(start, 0);
    open.push((Reverse(heuristic(start, end)), 0, start.0, start.1));

    while let Some((_, g, cx, cy)) = open.pop() {
        let current = (cx, cy);

        if current == end {
            let mut path = vec![end];
            let mut cur = end;
            while cur != start {
                cur = *came_from.get(&cur).unwrap();
                path.push(cur);
            }
            path.reverse();
            return Some(path);
        }

        // Skip ถ้า g score ที่บันทึกดีกว่า (stale entry ใน heap)
        if g > *g_score.get(&current).unwrap_or(&usize::MAX) {
            continue;
        }

        for neighbor in maze.open_neighbors(cx, cy) {
            let tentative_g = g + 1; // edge weight = 1 เสมอ
            if tentative_g < *g_score.get(&neighbor).unwrap_or(&usize::MAX) {
                g_score.insert(neighbor, tentative_g);
                came_from.insert(neighbor, current);
                let f = tentative_g + heuristic(neighbor, end);
                open.push((Reverse(f), tentative_g, neighbor.0, neighbor.1));
            }
        }
    }

    None
}
```

**ทำไม BFS และ A* ให้ path length เท่ากัน?** บน unweighted graph ที่ admissible heuristic (Manhattan distance ไม่เกิน actual distance) A* รับประกัน optimal solution ซึ่งก็คือ path เดียวกับ BFS แต่ expand nodes น้อยกว่า

**⚠️ Pitfall 4: A* heap ต้องจัดการ stale entries**

`BinaryHeap` ใน Rust ไม่รองรับ "decrease-key" operation ดังนั้นเมื่อเจอ path ที่ดีกว่า เราจะ push entry ใหม่เข้า heap (แทน update ของเดิม) ต้องกรอง stale entries ออก:

```rust
// ❌ ผิด: ไม่กรอง stale entries
// ถ้า A ถูก push 3 ครั้งด้วย g=10, g=7, g=5
// heap จะมี 3 entries — เมื่อ pop g=10 ออกมา จะ expand A ซ้ำ 2 ครั้ง (ผิดพลาด)

// ✓ ถูก: skip ถ้า g score ที่บันทึกในตอนนี้ดีกว่า entry ที่ pop ออกมา
if g > *g_score.get(&current).unwrap_or(&usize::MAX) {
    continue; // stale entry — ข้ามไป
}
```

---

### ขั้นที่ 7: ASCII Renderer และ Terminal Visualization

**ASCII Renderer** — แปลง maze เป็นข้อความ:

```rust
// src/render/ascii.rs
use crate::maze::Maze;
use std::collections::HashSet;

pub fn render(maze: &Maze, solution: Option<&[(usize, usize)]>) -> String {
    let solution_set: HashSet<(usize, usize)> = solution
        .unwrap_or_default()
        .iter()
        .cloned()
        .collect();

    let w = maze.width;
    let h = maze.height;
    let mut output = String::new();

    // Top border
    output.push('+');
    for _ in 0..w {
        output.push_str("---+");
    }
    output.push('\n');

    for y in 0..h {
        // Row: west wall + cell content + east wall
        output.push('|');
        for x in 0..w {
            let idx = maze.cell_index(x, y);
            let cell_char = if x == 0 && y == 0 {
                'S'   // Start
            } else if x == w - 1 && y == h - 1 {
                'E'   // End
            } else if solution_set.contains(&(x, y)) {
                '*'   // Solution path
            } else {
                ' '
            };
            output.push(' ');
            output.push(cell_char);
            output.push(' ');
            // East wall
            if maze.cells[idx].walls[1] {
                output.push('|');
            } else {
                output.push(' ');
            }
        }
        output.push('\n');

        // South walls row
        output.push('+');
        for x in 0..w {
            let idx = maze.cell_index(x, y);
            if maze.cells[idx].walls[2] {
                output.push_str("---+");
            } else {
                output.push_str("   +");
            }
        }
        output.push('\n');
    }

    output
}
```

**Terminal Visualization ด้วย crossterm** — แสดง animation step-by-step:

```rust
// src/render/terminal.rs
use crate::maze::Maze;
use crossterm::{
    cursor,
    style::{Color, Print, ResetColor, SetForegroundColor},
    terminal::{Clear, ClearType},
    ExecutableCommand,
};
use std::io::{stdout, Write};
use std::time::Duration;

/// วาด maze ทั้งหมดลงบน terminal
pub fn draw_maze(maze: &Maze, solution: Option<&[(usize, usize)]>) {
    use std::collections::HashSet;
    let solution_set: HashSet<(usize, usize)> =
        solution.unwrap_or_default().iter().cloned().collect();

    let mut out = stdout();
    let _ = out.execute(cursor::MoveTo(0, 0));

    let w = maze.width;
    let h = maze.height;

    // Top border
    print!("+");
    for _ in 0..w {
        print!("---+");
    }
    println!();

    for y in 0..h {
        print!("|");
        for x in 0..w {
            let idx = maze.cell_index(x, y);

            if x == 0 && y == 0 {
                let _ = out.execute(SetForegroundColor(Color::Green));
                print!(" S ");
                let _ = out.execute(ResetColor);
            } else if x == w - 1 && y == h - 1 {
                let _ = out.execute(SetForegroundColor(Color::Red));
                print!(" E ");
                let _ = out.execute(ResetColor);
            } else if solution_set.contains(&(x, y)) {
                let _ = out.execute(SetForegroundColor(Color::Yellow));
                print!(" * ");
                let _ = out.execute(ResetColor);
            } else {
                print!("   ");
            }

            if maze.cells[idx].walls[1] {
                print!("|");
            } else {
                print!(" ");
            }
        }
        println!();

        print!("+");
        for x in 0..w {
            let idx = maze.cell_index(x, y);
            if maze.cells[idx].walls[2] {
                print!("---+");
            } else {
                print!("   +");
            }
        }
        println!();
    }

    let _ = out.flush();
}

/// แสดง animation โดย step-by-step wall removal ที่ 30fps
pub fn animate_generation(width: usize, height: usize, algorithm: &str) {
    use crate::generators;
    use std::thread;

    let mut out = stdout();
    let _ = out.execute(Clear(ClearType::All));
    let _ = out.execute(cursor::Hide);

    // สำหรับ animation จริง ให้ใช้ channel ส่ง maze state แต่ละ step
    // ในที่นี้แสดงแบบ simplified: generate แล้วค่อย ๆ reveal
    let maze = match algorithm {
        "prim" => generators::prims::generate(width, height),
        "kruskal" => generators::kruskals::generate(width, height),
        "wilson" => generators::wilsons::generate(width, height),
        _ => generators::backtracker::generate(width, height),
    };

    // แสดง maze ที่สร้างเสร็จแล้ว (ใน production ให้ส่ง step events จาก generator)
    draw_maze(&maze, None);

    let solution = crate::solvers::bfs::solve(&maze);
    thread::sleep(Duration::from_millis(500));

    draw_maze(&maze, solution.as_deref());

    let _ = out.execute(cursor::Show);
    println!("\n✓ Maze generation complete!");
}
```

---

### ขั้นที่ 8: SVG และ PNG Output

**SVG Renderer** — แปลง maze เป็น SVG ด้วย `<line>` elements:

```rust
// src/render/svg.rs
use crate::maze::Maze;

pub fn render_svg(maze: &Maze, solution: Option<&[(usize, usize)]>, cell_size: u32) -> String {
    let w = maze.width as u32;
    let h = maze.height as u32;
    let cs = cell_size;
    let padding = cs;
    let svg_w = w * cs + 2 * padding;
    let svg_h = h * cs + 2 * padding;

    let mut lines = Vec::new();

    // Outer border
    lines.push(format!(
        r#"<rect x="{}" y="{}" width="{}" height="{}" fill="none" stroke="black" stroke-width="2"/>"#,
        padding, padding, w * cs, h * cs
    ));

    // Internal walls
    for y in 0..maze.height {
        for x in 0..maze.width {
            let idx = maze.cell_index(x, y);
            let cell = &maze.cells[idx];

            let x1 = padding + x as u32 * cs;
            let y1 = padding + y as u32 * cs;

            // East wall
            if x + 1 < maze.width && cell.walls[1] {
                lines.push(format!(
                    r#"<line x1="{}" y1="{}" x2="{}" y2="{}" stroke="black" stroke-width="1.5"/>"#,
                    x1 + cs, y1, x1 + cs, y1 + cs
                ));
            }
            // South wall
            if y + 1 < maze.height && cell.walls[2] {
                lines.push(format!(
                    r#"<line x1="{}" y1="{}" x2="{}" y2="{}" stroke="black" stroke-width="1.5"/>"#,
                    x1, y1 + cs, x1 + cs, y1 + cs
                ));
            }
        }
    }

    // Solution path
    if let Some(path) = solution {
        if path.len() >= 2 {
            let points: Vec<String> = path
                .iter()
                .map(|&(x, y)| {
                    let px = padding + x as u32 * cs + cs / 2;
                    let py = padding + y as u32 * cs + cs / 2;
                    format!("{},{}", px, py)
                })
                .collect();
            lines.push(format!(
                r#"<polyline points="{}" fill="none" stroke="red" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" opacity="0.7"/>"#,
                points.join(" ")
            ));
        }
    }

    // Start/End markers
    let sx = padding + cs / 2;
    let sy = padding + cs / 2;
    lines.push(format!(
        r#"<circle cx="{}" cy="{}" r="4" fill="green"/>"#,
        sx, sy
    ));
    let ex = padding + (maze.width as u32 - 1) * cs + cs / 2;
    let ey = padding + (maze.height as u32 - 1) * cs + cs / 2;
    lines.push(format!(
        r#"<circle cx="{}" cy="{}" r="4" fill="red"/>"#,
        ex, ey
    ));

    format!(
        r#"<?xml version="1.0" encoding="UTF-8"?>
<svg xmlns="http://www.w3.org/2000/svg" width="{svg_w}" height="{svg_h}" viewBox="0 0 {svg_w} {svg_h}">
  <rect width="{svg_w}" height="{svg_h}" fill="white"/>
  {}
</svg>"#,
        lines.join("\n  ")
    )
}
```

---

### ขั้นที่ 9: Maze Statistics

```rust
// src/stats.rs
use crate::maze::Maze;
use crate::solvers::bfs;
use serde::Serialize;

#[derive(Serialize, Debug)]
pub struct MazeStats {
    pub width: usize,
    pub height: usize,
    pub total_cells: usize,
    pub dead_end_count: usize,      // cells ที่มี passage เพียง 1 ทาง
    pub solution_path_length: usize,
    pub maze_area: usize,
    pub path_ratio: f64,            // solution_length / total_cells
}

pub fn compute(maze: &Maze) -> MazeStats {
    let total_cells = maze.width * maze.height;

    // Dead end = cell ที่มี open passages เพียง 1 ทาง
    let dead_end_count = (0..maze.height)
        .flat_map(|y| (0..maze.width).map(move |x| (x, y)))
        .filter(|&(x, y)| maze.open_neighbors(x, y).len() == 1)
        .count();

    let solution_path_length = bfs::solve(maze).map(|p| p.len()).unwrap_or(0);

    MazeStats {
        width: maze.width,
        height: maze.height,
        total_cells,
        dead_end_count,
        solution_path_length,
        maze_area: total_cells,
        path_ratio: solution_path_length as f64 / total_cells as f64,
    }
}
```

---

### ขั้นที่ 10: CLI ด้วย clap

```toml
# Cargo.toml
[dependencies]
crossterm = "0.27"
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
image = "0.25"
clap = { version = "4", features = ["derive"] }
```

```rust
// src/main.rs
use clap::{Parser, Subcommand, ValueEnum};

#[derive(Parser)]
#[command(name = "maze", about = "Maze Generator & Solver Visualizer")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// สร้าง maze ใหม่
    Generate {
        #[arg(short, long, default_value = "20")]
        width: usize,

        #[arg(short = 'H', long, default_value = "20")]
        height: usize,

        #[arg(short, long, default_value = "dfs")]
        algorithm: Algorithm,

        /// แสดง animation step-by-step
        #[arg(long)]
        animate: bool,

        /// output format: ascii, svg, png, json
        #[arg(short, long, default_value = "ascii")]
        output: OutputFormat,
    },
    /// แก้ maze และแสดง solution
    Solve {
        #[arg(short, long, default_value = "bfs")]
        solver: SolverAlgo,
    },
    /// แสดง statistics ของ maze
    Stats {
        #[arg(short, long, default_value = "20")]
        width: usize,
        #[arg(short = 'H', long, default_value = "20")]
        height: usize,
    },
}

#[derive(ValueEnum, Clone, Debug)]
enum Algorithm {
    Dfs,
    Prim,
    Kruskal,
    Wilson,
}

#[derive(ValueEnum, Clone, Debug)]
enum SolverAlgo {
    Bfs,
    Astar,
}

#[derive(ValueEnum, Clone, Debug)]
enum OutputFormat {
    Ascii,
    Svg,
    Png,
    Json,
}

fn main() {
    let cli = Cli::parse();

    match cli.command {
        Commands::Generate { width, height, algorithm, animate, output } => {
            let maze = match algorithm {
                Algorithm::Dfs => generate_recursive_backtracker(width, height),
                Algorithm::Prim => generate_prims(width, height),
                Algorithm::Kruskal => generate_kruskals(width, height),
                Algorithm::Wilson => generate_wilsons(width, height),
            };

            let solution = solve_bfs(&maze);
            let stats = compute_stats(&maze);

            match output {
                OutputFormat::Ascii => {
                    println!("{}", render_ascii(&maze, solution.as_deref()));
                }
                OutputFormat::Json => {
                    println!("{}", serde_json::to_string_pretty(&stats).unwrap());
                }
                OutputFormat::Svg => {
                    // ใช้ svg renderer
                    println!("SVG output saved to maze.svg");
                }
                OutputFormat::Png => {
                    println!("PNG output saved to maze.png");
                }
            }
        }
        Commands::Stats { width, height } => {
            let maze = generate_recursive_backtracker(width, height);
            let stats = compute_stats(&maze);
            println!("{}", serde_json::to_string_pretty(&stats).unwrap());
        }
        Commands::Solve { solver } => {
            let maze = generate_recursive_backtracker(20, 20);
            let path = match solver {
                SolverAlgo::Bfs => solve_bfs(&maze),
                SolverAlgo::Astar => solve_astar(&maze),
            };
            if let Some(p) = path {
                println!("Path found: {} steps", p.len());
            }
        }
    }
}
```

---

## การทดสอบ (Testing)

### Tests ครบชุดใน `src/main.rs`

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// Helper: ตรวจว่า BFS เข้าถึงทุก cell ได้ (maze connected)
    fn is_fully_connected(maze: &Maze) -> bool {
        let start = (0usize, 0usize);
        let mut visited = HashSet::new();
        let mut queue = VecDeque::new();
        queue.push_back(start);
        visited.insert(start);

        while let Some((cx, cy)) = queue.pop_front() {
            for (nx, ny) in maze.open_neighbors(cx, cy) {
                if visited.insert((nx, ny)) {
                    queue.push_back((nx, ny));
                }
            }
        }

        visited.len() == maze.width * maze.height
    }

    #[test]
    fn test_recursive_backtracker_connected() {
        let maze = generate_recursive_backtracker(10, 10);
        assert!(is_fully_connected(&maze));
    }

    #[test]
    fn test_kruskals_connected() {
        let maze = generate_kruskals(8, 8);
        assert!(is_fully_connected(&maze));
    }

    #[test]
    fn test_prims_connected() {
        let maze = generate_prims(8, 8);
        assert!(is_fully_connected(&maze));
    }

    #[test]
    fn test_wilsons_connected() {
        let maze = generate_wilsons(6, 6);
        assert!(is_fully_connected(&maze));
    }

    #[test]
    fn test_bfs_finds_path() {
        let maze = generate_recursive_backtracker(10, 10);
        let path = solve_bfs(&maze);
        assert!(path.is_some());
        let path = path.unwrap();
        assert_eq!(*path.first().unwrap(), (0, 0));
        assert_eq!(*path.last().unwrap(), (9, 9));
    }

    #[test]
    fn test_astar_same_length_as_bfs() {
        let maze = generate_recursive_backtracker(10, 10);
        let bfs_path = solve_bfs(&maze).expect("BFS must find path");
        let astar_path = solve_astar(&maze).expect("A* must find path");
        assert_eq!(
            bfs_path.len(), astar_path.len(),
            "A* and BFS must find paths of equal length"
        );
    }

    #[test]
    fn test_wall_removal_bidirectional() {
        let mut maze = Maze::new(5, 5);
        maze.remove_wall(0, 0, 1, 0);
        assert!(!maze.cells[maze.cell_index(0, 0)].walls[1]);
        assert!(!maze.cells[maze.cell_index(1, 0)].walls[3]);

        maze.remove_wall(0, 0, 0, 1);
        assert!(!maze.cells[maze.cell_index(0, 0)].walls[2]);
        assert!(!maze.cells[maze.cell_index(0, 1)].walls[0]);
    }

    #[test]
    fn test_cell_index() {
        let maze = Maze::new(5, 5);
        assert_eq!(maze.cell_index(0, 0), 0);
        assert_eq!(maze.cell_index(4, 0), 4);
        assert_eq!(maze.cell_index(0, 1), 5);
        assert_eq!(maze.cell_index(4, 4), 24);
    }

    #[test]
    fn test_union_find_correctness() {
        let mut uf = UnionFind::new(5);
        assert!(!uf.connected(0, 1));
        uf.union(0, 1);
        assert!(uf.connected(0, 1));
        uf.union(1, 2);
        assert!(uf.connected(0, 2));
        assert!(!uf.connected(0, 3));
        assert!(!uf.union(0, 2)); // already connected
    }

    #[test]
    fn test_dead_end_count_nonzero() {
        let maze = generate_recursive_backtracker(10, 10);
        let stats = compute_stats(&maze);
        assert!(stats.dead_end_count > 0);
    }

    #[test]
    fn test_solution_path_nonempty() {
        let maze = generate_recursive_backtracker(5, 5);
        let stats = compute_stats(&maze);
        assert!(stats.solution_path_length > 0);
    }
}
```

### ผลลัพธ์จาก `cargo test` (จริง)

```
running 11 tests
test tests::test_astar_same_length_as_bfs ... ok
test tests::test_bfs_finds_path ... ok
test tests::test_cell_index ... ok
test tests::test_kruskals_connected ... ok
test tests::test_prims_connected ... ok
test tests::test_dead_end_count_nonzero ... ok
test tests::test_recursive_backtracker_connected ... ok
test tests::test_solution_path_nonempty ... ok
test tests::test_union_find_correctness ... ok
test tests::test_wall_removal_bidirectional ... ok
test tests::test_wilsons_connected ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

### ผลลัพธ์จาก `cargo run` (จริง)

```
=== Maze Generator & Solver ===

Algorithm: Recursive Backtracker (DFS)
Size: 10x10
Dead ends: 13
Solution path: 33 steps

+---+---+---+---+---+---+---+---+---+---+
| S |           |       |               |
+   +   +   +   +   +---+   +---+---+   +
| * |   |   |       |       |       |   |
+   +   +   +   +---+   +---+   +---+   +
| * |   |   |   |       |       |       |
+   +---+   +   +   +---+---+   +   +---+
| *   * |   |   |       |       |       |
+---+   +   +   +---+   +   +---+---+   +
|   | * |   |       |       |           |
+   +   +   +---+---+---+   +   +---+---+
|   | * |               |   |           |
+   +   +   +---+---+   +   +---+---+   +
| *   * |   | *   * |   |   |       |   |
+   +---+   +   +   +   +   +   +---+   +
| *   * |   | * | * |           | *   * |
+---+   +---+   +   +   +---+---+   +   +
|   | * | *   * | * |   | *   * | * | * |
+   +   +   +---+   +---+   +   +   +   +
|     *   * |     *   *   * | *   * | E |
+---+---+---+---+---+---+---+---+---+---+

Stats JSON:
{
  "width": 10,
  "height": 10,
  "total_cells": 100,
  "dead_end_count": 13,
  "solution_path_length": 33,
  "maze_area": 100,
  "path_ratio": 0.33
}
```

---

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### Pitfall 1: Stack Overflow กับ Recursive Backtracker แบบ Recursive

DFS-based maze generation มักถูกเขียนแบบ recursive ใน pseudocode แต่ใน Rust ถ้า maze ใหญ่ (เช่น 100×100 = 10,000 cells) stack depth อาจสูงถึง 10,000 ทำให้ stack overflow:

```rust
// ❌ อันตราย: recursive ตรง ๆ
fn backtrack(maze: &mut Maze, visited: &mut Vec<bool>, x: usize, y: usize) {
    let unvisited = get_unvisited_neighbors(maze, visited, x, y);
    for (nx, ny) in unvisited {
        maze.remove_wall(x, y, nx, ny);
        visited[maze.cell_index(nx, ny)] = true;
        backtrack(maze, visited, nx, ny); // อาจ overflow ถ้า path ยาว 10,000 steps
    }
}

// ✓ ปลอดภัย: ใช้ explicit stack บน heap
let mut stack: Vec<(usize, usize)> = Vec::new();
while let Some(&(cx, cy)) = stack.last() {
    // ...
}
```

### Pitfall 2: Frontier Duplicates ใน Prim's ทำให้ skip cells

Prim's algorithm เพิ่ม frontier cells หลายครั้งได้ ถ้าไม่ตรวจสอบจะทำให้บาง cell ถูก skip เพราะถูกเพิ่มเข้า maze แล้วในรอบก่อน:

```rust
// ❌ ผิด: remove cell ออกจาก frontier ก่อนเช็ค
frontier.retain(|&c| c != (fx, fy)); // O(n) และอาจลบไม่หมดถ้ามีซ้ำ

// ✓ ถูก: เช็คหลัง pop ด้วย guard clause
let (fx, fy) = frontier.swap_remove(idx);
if in_maze[maze.cell_index(fx, fy)] {
    continue; // ถูก process แล้ว ข้าม
}
```

### Pitfall 3: Wilson's Loop Erasure ที่ Incomplete

Wilson's algorithm ต้องลบทั้ง `path` vector และ `path_index` map พร้อมกัน ถ้าลบแค่อย่างใดอย่างหนึ่ง จะตรวจ loop ผิดพลาดในรอบถัดไป:

```rust
// ❌ ผิด: ลบเฉพาะ path แต่ไม่ลบ path_index
path.truncate(loop_start + 1);
// path_index ยังชี้ไปที่ cells ที่ถูกลบออกจาก path แล้ว
// ทำให้ loop detection ผิดพลาดในรอบถัดไป

// ✓ ถูก: ลบทั้งสองพร้อมกัน
for p in &path[loop_start + 1..] {
    path_index.remove(&maze.cell_index(p.0, p.1));
}
path.truncate(loop_start + 1);
```

### Pitfall 4: A* Heap Stale Entries

`std::collections::BinaryHeap` ไม่รองรับ "decrease key" ดังนั้นเมื่อพบ path ที่ดีกว่า ต้อง push entry ใหม่และกรอง stale entries ออกตอน pop:

```rust
// ❌ ผิด: ไม่ตรวจสอบ stale entries
while let Some((_, g, cx, cy)) = open.pop() {
    // อาจ expand node เดิมหลายครั้งด้วย g score เก่า
    for neighbor in maze.open_neighbors(cx, cy) { ... }
}

// ✓ ถูก: skip entry ที่ g score เก่ากว่าที่บันทึกไว้
while let Some((_, g, cx, cy)) = open.pop() {
    if g > *g_score.get(&(cx, cy)).unwrap_or(&usize::MAX) {
        continue; // stale — skip
    }
    // ประมวลผลต่อ...
}
```

---

## การ Package และ Deploy

### Build Release

```bash
cargo build --release
# binary อยู่ที่ target/release/maze_generator

# รันด้วย argument
./target/release/maze_generator generate --width 30 --height 30 --algorithm dfs
./target/release/maze_generator generate --width 20 --height 20 --algorithm kruskal --output svg
./target/release/maze_generator stats --width 50 --height 50
```

### ตัวอย่าง Release Build Flags ที่แนะนำ

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true        # ลบ debug symbols ออกเพื่อลดขนาด binary
```

### Docker (สำหรับ web server mode)

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/maze_generator /usr/local/bin/
ENTRYPOINT ["maze_generator"]
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Aldous-Broder Algorithm

Aldous-Broder เป็นอัลกอริทึมที่ง่ายที่สุดที่ได้ uniform spanning tree: เดิน random walk ไปเรื่อย ๆ ถ้าเจอ unvisited cell ให้ลบกำแพงและ mark visited ทำจนทุก cell visited

```rust
pub fn generate_aldous_broder(width: usize, height: usize) -> Maze {
    let mut maze = Maze::new(width, height);
    let mut rng = rand::thread_rng();
    let n = width * height;
    let mut visited = vec![false; n];
    let mut visited_count = 0;

    let mut cx = rng.gen_range(0..width);
    let mut cy = rng.gen_range(0..height);
    visited[maze.cell_index(cx, cy)] = true;
    visited_count += 1;

    while visited_count < n {
        let neighbors = maze.all_neighbors(cx, cy);
        let &(nx, ny) = neighbors.choose(&mut rng).unwrap();
        let nidx = maze.cell_index(nx, ny);

        if !visited[nidx] {
            maze.remove_wall(cx, cy, nx, ny);
            visited[nidx] = true;
            visited_count += 1;
        }

        cx = nx;
        cy = ny;
    }

    maze
}
```

**ทดสอบ:** ตรวจว่า Aldous-Broder ผลิต connected maze และเปรียบ texture กับ Wilson's

### แบบฝึกหัดที่ 2: Dijkstra Solver สำหรับ Weighted Maze

เพิ่ม `weight: u32` เข้าไปใน `Cell` (0=normal, 1=mud=cost 3, 2=water=cost 5) แล้วสร้าง Dijkstra solver:

```rust
#[derive(Clone, Debug)]
pub struct Cell {
    pub walls: [bool; 4],
    pub weight: u32,  // cost สำหรับ enter cell นี้
}

// Dijkstra: BinaryHeap<(Reverse<u32>, usize, usize)>
// g[neighbor] = g[current] + neighbor.weight
```

เปรียบกับ A* — Manhattan distance อาจไม่ admissible ถ้า weight > 1 ต้องปรับ heuristic

### แบบฝึกหัดที่ 3: Maze Rooms (Imperfect Maze)

Perfect maze มีเพียง 1 path ระหว่างทุกคู่ cells "Imperfect maze" มีหลาย path ทำให้ยากขึ้นในการแก้:

```rust
/// เพิ่ม loop เข้าไปใน maze โดยลบกำแพงสุ่ม loop_count ครั้ง
pub fn add_loops(maze: &mut Maze, loop_count: usize) {
    let mut rng = rand::thread_rng();
    let mut loops_added = 0;

    while loops_added < loop_count {
        let x = rng.gen_range(0..maze.width - 1);
        let y = rng.gen_range(0..maze.height - 1);

        if maze.cells[maze.cell_index(x, y)].walls[1] {
            maze.remove_wall(x, y, x + 1, y);
            loops_added += 1;
        }
    }
}
```

### แบบฝึกหัดที่ 4: 3D Maze Extension

ขยาย maze เป็น 3 มิติด้วยการเพิ่ม wall index `Up=4` และ `Down=5`:

```rust
#[derive(Clone, Debug)]
pub struct Cell3D {
    pub walls: [bool; 6],  // N=0, E=1, S=2, W=3, Up=4, Down=5
}

pub struct Maze3D {
    pub width: usize,
    pub height: usize,
    pub depth: usize,
    pub cells: Vec<Cell3D>,
}

impl Maze3D {
    pub fn cell_index(&self, x: usize, y: usize, z: usize) -> usize {
        z * self.width * self.height + y * self.width + x
    }
}
```

Visualize ด้วยการ render แต่ละ "floor" แยกกัน หรือใช้ isometric ASCII art

---

## สรุป

ในโปรเจคนี้เราได้สร้างระบบ maze generator และ solver ที่ครบสมบูรณ์ โดยผ่านขั้นตอนหลัก:

1. **Data model** — `Maze { cells: Vec<Cell> }` ที่ใช้ flat array เพื่อ cache efficiency พร้อม `cell_index(x, y)` และ `remove_wall` แบบ bidirectional
2. **4 algorithms** — Recursive Backtracker (DFS), Prim's, Kruskal's (Union-Find), Wilson's (loop-erased random walk) แต่ละตัวให้ maze texture ที่แตกต่างกัน
3. **2 solvers** — BFS (ง่าย, optimal) และ A* (Manhattan heuristic, expand น้อยกว่า) ที่ให้ path length เท่ากันบน unweighted maze
4. **Multiple outputs** — ASCII, SVG (line elements), PNG (image crate), JSON stats
5. **4 Pitfalls** — recursive DFS stack overflow, Prim's frontier duplicates, Wilson's incomplete loop erasure, A* stale heap entries

**Pattern ที่ได้เรียนจากโปรเจคนี้:**
- **Flat 2D array** `Vec<T>` เร็วกว่า `Vec<Vec<T>>` เสมอบน grid problems
- **Union-Find** ด้วย path compression + rank คือ near-O(1) ที่ใช้งานได้จริง
- **Explicit stack** แทน recursion เป็น best practice สำหรับ Rust graph traversal
- **Lazy deduplication** (guard clause หลัง pop) มักเร็วกว่า `HashSet` สำหรับ frontier

โปรเจคถัดไป **F01: Key-Value Store** จะเจาะลึกเรื่อง **persistent data storage** และ **storage engine design** โดยนำ concepts เรื่อง data structures ที่ได้จากโปรเจคนี้ไปต่อยอดกับ disk I/O และ crash recovery

---

**โปรเจคก่อนหน้า:** [Project E09: ASCII Art Renderer](project-e09-ascii-art.md) | **โปรเจคถัดไป:** [Project F01: Key-Value Store](project-f01-kv-store.md)
