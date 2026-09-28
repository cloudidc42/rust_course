# Project E06: Roguelike Dungeon Crawler

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 6 ชั่วโมง

## ภาพรวมโปรเจค

เกม **Roguelike Dungeon Crawler** แบบ turn-based ใน terminal — ผู้เล่นสำรวจ dungeon ที่สร้างขึ้นแบบ procedural, ต่อสู้กับ monster, เก็บ item, และลงไปยัง dungeon ชั้นล่างซึ่งยากขึ้นเรื่อย ๆ ระบบทั้งหมดสร้างบน **ECS (Entity Component System)** แบบ lite ที่เขียนเองตั้งแต่ต้น

โปรเจคนี้เป็นต้นแบบของเกม roguelike คลาสสิกอย่าง **NetHack** หรือ **Dungeon Crawl Stone Soup** ในขนาดที่เรียนรู้ได้ภายใน 1 สัปดาห์ เทคนิคที่ได้ฝึกเหมือนกับที่ใช้ใน game engine จริง เช่น การจัดการ state แบบ component-based, procedural content generation, และ pathfinding algorithm

**Use case จริง:**
- ECS pattern ใช้ใน game engine เกือบทุกตัว (Unity DOTS, Bevy, Godot)
- BSP map generation ใช้ใน Diablo, NetHack, และ dungeon crawler ทั่วไป
- A\* pathfinding ใช้ใน AI ทุกประเภท ตั้งแต่เกมถึง robot navigation
- Shadowcasting FOV ใช้ในทุก roguelike ที่มี "fog of war"

## สิ่งที่จะได้เรียนรู้

- **ECS Architecture**: ออกแบบระบบ Entity Component System แบบ minimal ด้วย `HashMap<TypeId, Vec<Option<Box<dyn Any>>>>`
- **Procedural Generation**: สร้าง dungeon ด้วย Binary Space Partitioning (BSP) algorithm แบบ recursive
- **Field of View**: คำนวณ visibility ด้วย Bresenham ray casting พร้อม "fog of war" effect
- **A\* Pathfinding**: ใช้ crate `pathfinding` สำหรับ monster AI navigation ใน grid map
- **Turn-based Game Loop**: จัดการ player input → monster AI → advance turn อย่างถูกต้อง
- **Terminal UI**: สร้าง TUI ด้วย `ratatui` + `crossterm` แสดงแผนที่ ASCII และ sidebar
- **State Machine AI**: ออกแบบ monster AI ที่มี state chase/random-walk
- **Serialization**: บันทึก/โหลด game state ด้วย `serde_json`

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Ownership, borrowing, structs, enums, traits
- **Part 21–30**: Iterators, closures, `Box<dyn Trait>`, `HashMap`
- **Part 31–40**: Error handling, `Option`/`Result`, `std::fs`
- **Part 41–50**: `Any` type, TypeId, dynamic dispatch, lifetimes advanced
- **Part 61–70**: Traits objects, interior mutability (สำหรับ ECS world)
- Crate ecosystem: `rand`, `serde`/`serde_json`, `pathfinding`

## โครงสร้างโปรเจค (Project Layout)

```
roguelike/
├── src/
│   ├── main.rs          ← entry point + game loop หลัก
│   ├── ecs.rs           ← World, EntityBuilder, component storage
│   ├── components.rs    ← component struct definitions ทั้งหมด
│   ├── map.rs           ← Map, Tile, Rect + module declarations
│   ├── map/
│   │   └── bsp.rs       ← BSP dungeon generator
│   ├── fov.rs           ← Field of View (Bresenham ray casting)
│   ├── pathfind.rs      ← A* pathfinding wrapper
│   ├── combat.rs        ← combat resolution system
│   ├── ai.rs            ← monster AI state machine
│   ├── items.rs         ← item types และ effects
│   └── saveload.rs      ← JSON serialization/deserialization
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### ECS Lite Architecture

ECS แบบ full-featured (เช่น `bevy_ecs`) ซับซ้อนมาก สำหรับโปรเจคนี้ใช้ **ECS lite** ที่ออกแบบให้อ่านง่ายและเข้าใจได้ใน 200 บรรทัด:

```
World
├── next_entity: usize          ← counter สำหรับสร้าง entity id
├── alive: Vec<bool>            ← entity ยังมีชีวิตอยู่ไหม
└── components: HashMap<TypeId, Vec<Option<Box<dyn Any>>>>
    ├── TypeId::of::<Position>()   → [Some(pos_0), None, Some(pos_2), ...]
    ├── TypeId::of::<CombatStats>() → [Some(stats_0), None, ...]
    └── ...
```

- **Entity**: แค่ `usize` index — ไม่มี overhead เลย
- **Component storage**: `Vec<Option<Box<dyn Any>>>` indexed by entity id
- **Query**: iterate ผ่าน storage ของ component type นั้น filter entities ที่ `alive`

ทำไมไม่ใช้ `Vec<Option<T>>` โดยตรง? เพราะต้องเก็บ component หลายประเภทใน structure เดียวกัน ต้องใช้ `Box<dyn Any>` เพื่อ type erasure

### Data Flow

```
Game Loop
   │
   ├── Player Input
   │       └── PlayerAction (Move/Attack/PickUp/UseItem/Descend)
   │
   ├── Player System
   │       ├── ตรวจสอบ tile เป้าหมาย
   │       ├── ถ้า monster อยู่ → attack
   │       └── ถ้าว่าง → move
   │
   ├── FOV System
   │       └── compute_fov() → update map visibility
   │
   ├── Monster AI System
   │       ├── สำหรับแต่ละ monster ที่มีชีวิต
   │       ├── ถ้า player ใน FOV และ range ≤ 8 → chase (A*)
   │       └── ไม่งั้น → random walk
   │
   ├── Cleanup System
   │       └── remove dead entities
   │
   └── Render System
           ├── วาด map tiles (ASCII)
           └── วาด sidebar (HP, stats, log)
```

### การเลือก Design Patterns

| ปัญหา | ทางเลือก | ที่เลือก | เหตุผล |
|--------|----------|----------|--------|
| เก็บ game objects | Struct of arrays vs Array of structs vs ECS | ECS | แยก data/logic ได้ชัด เพิ่ม component ไม่ต้องแก้ struct เดิม |
| Map generation | Random rooms vs BSP vs Cellular automata | BSP | ห้องไม่ซ้อนกัน เชื่อมต่อสม่ำเสมอ |
| FOV algorithm | Shadowcasting vs Raycasting vs BFS | Bresenham raycasting | ถูกต้อง symmetric เข้าใจง่าย |
| Pathfinding | DIY A\* vs `pathfinding` crate | `pathfinding` | ผ่าน battle-test แล้ว ไม่ต้อง debug |
| Save format | Binary vs JSON vs RON | JSON via serde_json | human-readable debug ง่าย |

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ECS Core — World, EntityBuilder, Component Storage

เริ่มจากหัวใจของระบบ: `World` struct ที่เก็บ components ของ entities ทั้งหมด

**`Cargo.toml`:**

```toml
[package]
name = "roguelike"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
pathfinding = "4"
ratatui = "0.29"
crossterm = "0.28"
```

**`src/ecs.rs`:**

```rust
use std::any::{Any, TypeId};
use std::collections::HashMap;

pub type Entity = usize;

pub struct World {
    next_entity: Entity,
    alive: Vec<bool>,
    components: HashMap<TypeId, Vec<Option<Box<dyn Any>>>>,
}

impl World {
    pub fn new() -> Self {
        World {
            next_entity: 0,
            alive: Vec::new(),
            components: HashMap::new(),
        }
    }

    pub fn spawn(&mut self) -> EntityBuilder {
        let id = self.next_entity;
        self.next_entity += 1;
        self.alive.push(true);
        EntityBuilder { id, components: Vec::new() }
    }

    pub fn remove_entity(&mut self, entity: Entity) {
        if entity < self.alive.len() {
            self.alive[entity] = false;
        }
    }

    pub fn is_alive(&self, entity: Entity) -> bool {
        entity < self.alive.len() && self.alive[entity]
    }

    pub fn insert<T: 'static>(&mut self, entity: Entity, component: T) {
        let type_id = TypeId::of::<T>();
        let storage = self.components.entry(type_id).or_default();
        while storage.len() <= entity {
            storage.push(None);
        }
        storage[entity] = Some(Box::new(component));
    }

    pub fn get<T: 'static>(&self, entity: Entity) -> Option<&T> {
        let type_id = TypeId::of::<T>();
        self.components.get(&type_id)?
            .get(entity)?
            .as_ref()?
            .downcast_ref::<T>()
    }

    pub fn get_mut<T: 'static>(&mut self, entity: Entity) -> Option<&mut T> {
        let type_id = TypeId::of::<T>();
        self.components.get_mut(&type_id)?
            .get_mut(entity)?
            .as_mut()?
            .downcast_mut::<T>()
    }

    /// query entities ทั้งหมดที่มี component ประเภท T
    pub fn query<T: 'static>(&self) -> Vec<(Entity, &T)> {
        let type_id = TypeId::of::<T>();
        let Some(storage) = self.components.get(&type_id) else {
            return Vec::new();
        };
        storage.iter().enumerate()
            .filter(|(e, slot)| slot.is_some() && self.is_alive(*e))
            .filter_map(|(e, slot)| {
                slot.as_ref()?.downcast_ref::<T>().map(|c| (e, c))
            })
            .collect()
    }

    pub fn entity_count(&self) -> usize {
        self.alive.iter().filter(|&&a| a).count()
    }
}
```

**`EntityBuilder` — Fluent API:**

```rust
type DynComponentFn = Box<dyn FnOnce(&mut World, Entity)>;

pub struct EntityBuilder {
    pub id: Entity,
    components: Vec<DynComponentFn>,
}

impl EntityBuilder {
    /// เพิ่ม component ด้วย method chaining
    pub fn with<T: 'static>(mut self, component: T) -> Self {
        self.components.push(Box::new(move |world, entity| {
            world.insert(entity, component);
        }));
        self
    }

    /// สร้าง entity จริง ๆ — คืนค่า entity id
    pub fn build(self, world: &mut World) -> Entity {
        let id = self.id;
        for f in self.components {
            f(world, id);
        }
        id
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
let player = world.spawn()
    .with(Position { x: 10, y: 5 })
    .with(CombatStats { hp: 30, max_hp: 30, defense: 2, power: 5 })
    .with(Player)
    .with(Name("Hero".to_string()))
    .build(&mut world);
```

**สิ่งที่น่าสนใจใน ECS Design:**

`Box<dyn FnOnce(&mut World, Entity)>` ใน `EntityBuilder` เป็นเทคนิค "deferred component insertion" — component ถูก capture ใน closure และรอ insert จนกว่าจะ `.build()` เพื่อหลีกเลี่ยงปัญหา borrow checker (ถ้า insert ทันทีจะ conflict กับ `&mut world` ที่ builder ถือไว้)

---

### ขั้นที่ 2: Components และ Map

**`src/components.rs`** — ประกาศ component struct ทั้งหมด:

```rust
use serde::{Deserialize, Serialize};
use crate::items::ItemKind;

pub type Entity = usize;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Position { pub x: i32, pub y: i32 }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Renderable {
    pub glyph: char,
    pub fg: (u8, u8, u8),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CombatStats {
    pub hp: i32,
    pub max_hp: i32,
    pub defense: i32,
    pub power: i32,
}

// Tag components — ไม่มี data เป็นแค่ marker
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Player;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Monster;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Name(pub String);

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Item { pub kind: ItemKind }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Inventory {
    pub items: Vec<Entity>,
    pub capacity: usize,
}

impl Inventory {
    pub fn new(capacity: usize) -> Self {
        Inventory { items: Vec::new(), capacity }
    }

    pub fn add(&mut self, entity: Entity) -> bool {
        if self.items.len() < self.capacity {
            self.items.push(entity);
            true
        } else {
            false // inventory เต็ม
        }
    }

    pub fn remove(&mut self, entity: Entity) -> bool {
        if let Some(pos) = self.items.iter().position(|&e| e == entity) {
            self.items.remove(pos);
            true
        } else {
            false // ไม่มี item นั้น
        }
    }
}

/// FOV component — เก็บ tiles ที่มองเห็น
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Viewshed {
    pub visible_tiles: Vec<(i32, i32)>,
    pub range: i32,
    pub dirty: bool, // ต้อง recompute หรือเปล่า
}
```

**`src/map.rs`** — แผนที่ tile-based:

```rust
pub mod bsp;

use serde::{Deserialize, Serialize};

pub const MAP_WIDTH: usize = 80;
pub const MAP_HEIGHT: usize = 45;

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum TileKind {
    Floor,
    Wall,
    Door,
    StairsDown,
    StairsUp,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tile {
    pub kind: TileKind,
    pub visible: bool,      // มองเห็นใน turn นี้
    pub explored: bool,     // เคยมาแล้ว (fog of war)
    pub blocked: bool,      // กีดขวางการเดิน
    pub block_sight: bool,  // กีดขวางการมองเห็น
}

impl Tile {
    pub fn wall() -> Self {
        Tile { kind: TileKind::Wall, visible: false, explored: false,
               blocked: true, block_sight: true }
    }
    pub fn floor() -> Self {
        Tile { kind: TileKind::Floor, visible: false, explored: false,
               blocked: false, block_sight: false }
    }
    pub fn door() -> Self {
        Tile { kind: TileKind::Door, visible: false, explored: false,
               blocked: false, block_sight: false }
    }
    pub fn stairs_down() -> Self {
        Tile { kind: TileKind::StairsDown, visible: false, explored: false,
               blocked: false, block_sight: false }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Map {
    pub width: usize,
    pub height: usize,
    pub tiles: Vec<Tile>,
}

impl Map {
    pub fn new(width: usize, height: usize) -> Self {
        let tiles = vec![Tile::wall(); width * height];
        Map { width, height, tiles }
    }

    #[inline]
    pub fn idx(&self, x: usize, y: usize) -> usize {
        y * self.width + x
    }

    pub fn in_bounds(&self, x: i32, y: i32) -> bool {
        x >= 0 && y >= 0
            && (x as usize) < self.width
            && (y as usize) < self.height
    }

    pub fn is_passable(&self, x: i32, y: i32) -> bool {
        if !self.in_bounds(x, y) { return false; }
        !self.tiles[self.idx(x as usize, y as usize)].blocked
    }

    pub fn blocks_sight(&self, x: i32, y: i32) -> bool {
        if !self.in_bounds(x, y) { return true; }
        self.tiles[self.idx(x as usize, y as usize)].block_sight
    }

    pub fn tile_mut(&mut self, x: usize, y: usize) -> &mut Tile {
        let idx = self.idx(x, y);
        &mut self.tiles[idx]
    }

    pub fn clear_visibility(&mut self) {
        for tile in &mut self.tiles {
            tile.visible = false;
        }
    }

    /// วาด floor rectangle (ไม่รวม border)
    pub fn carve_room(&mut self, room: &Rect) {
        for y in (room.y1 + 1)..room.y2 {
            for x in (room.x1 + 1)..room.x2 {
                if x < self.width && y < self.height {
                    let idx = self.idx(x, y);
                    self.tiles[idx] = Tile::floor();
                }
            }
        }
    }

    /// เชื่อมต่อด้วย L-shaped corridor (แนวนอนก่อน แล้วแนวตั้ง)
    pub fn carve_corridor_h_then_v(
        &mut self,
        start: (usize, usize),
        end: (usize, usize),
    ) {
        let (x1, y1) = start;
        let (x2, y2) = end;
        let (xs, xe) = if x1 < x2 { (x1, x2) } else { (x2, x1) };
        for x in xs..=xe {
            if x < self.width && y1 < self.height {
                let idx = self.idx(x, y1);
                self.tiles[idx] = Tile::floor();
            }
        }
        let (ys, ye) = if y1 < y2 { (y1, y2) } else { (y2, y1) };
        for y in ys..=ye {
            if x2 < self.width && y < self.height {
                let idx = self.idx(x2, y);
                self.tiles[idx] = Tile::floor();
            }
        }
    }
}

/// สี่เหลี่ยมแทนห้องหรือ partition
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Rect {
    pub x1: usize, pub y1: usize,
    pub x2: usize, pub y2: usize,
}

impl Rect {
    pub fn new(x: usize, y: usize, w: usize, h: usize) -> Self {
        Rect { x1: x, y1: y, x2: x + w, y2: y + h }
    }

    pub fn center(&self) -> (usize, usize) {
        ((self.x1 + self.x2) / 2, (self.y1 + self.y2) / 2)
    }

    pub fn intersects(&self, other: &Rect) -> bool {
        self.x1 <= other.x2 && self.x2 >= other.x1
            && self.y1 <= other.y2 && self.y2 >= other.y1
    }
}
```

---

### ขั้นที่ 3: BSP Dungeon Generator

**BSP (Binary Space Partitioning)** แบ่งพื้นที่ทั้งหมดออกเป็นสองส่วน recursive จนได้ leaf nodes แล้วสร้างห้องใน leaf แต่ละใบ จากนั้นเชื่อมต่อ rooms ระหว่าง sibling nodes

```
┌────────────────────────┐
│         root           │
│  ┌──────┬─────────┐    │
│  │ node │  node   │    │
│  │  A   │    B    │    │
│  ├──────┤  ┌──┬───┤    │
│  │ node │  │C │ D │    │
│  │  E   │  │  │   │    │
│  └──────┴──┴──┴───┘    │
└────────────────────────┘
```

**`src/map/bsp.rs`:**

```rust
use rand::{Rng, SeedableRng};
use rand::rngs::StdRng;
use crate::map::{Map, Rect, Tile};

const MIN_ROOM_SIZE: usize = 5;
const MAX_DEPTH: usize = 5;

struct BspNode {
    region: Rect,
    left: Option<Box<BspNode>>,
    right: Option<Box<BspNode>>,
    room: Option<Rect>,
}

impl BspNode {
    fn new(region: Rect) -> Self {
        BspNode { region, left: None, right: None, room: None }
    }

    fn split(&mut self, rng: &mut StdRng, depth: usize) {
        if depth == 0 {
            self.create_room(rng);
            return;
        }

        let w = self.region.width();
        let h = self.region.height();

        // เลือกทิศทางการแบ่ง
        let split_horizontally = if w > h * 2 { false }
            else if h > w * 2 { true }
            else { rng.gen_bool(0.5) };

        if split_horizontally {
            let min_split = self.region.y1 + MIN_ROOM_SIZE + 2;
            let max_split = self.region.y2.saturating_sub(MIN_ROOM_SIZE + 2);
            if max_split <= min_split {
                self.create_room(rng);
                return;
            }
            let split = rng.gen_range(min_split..max_split);
            // สร้าง child nodes
            let mut left = BspNode::new(Rect::new(
                self.region.x1, self.region.y1,
                w, split - self.region.y1,
            ));
            let mut right = BspNode::new(Rect::new(
                self.region.x1, split,
                w, self.region.y2 - split,
            ));
            left.split(rng, depth - 1);
            right.split(rng, depth - 1);
            self.left = Some(Box::new(left));
            self.right = Some(Box::new(right));
        } else {
            let min_split = self.region.x1 + MIN_ROOM_SIZE + 2;
            let max_split = self.region.x2.saturating_sub(MIN_ROOM_SIZE + 2);
            if max_split <= min_split {
                self.create_room(rng);
                return;
            }
            let split = rng.gen_range(min_split..max_split);
            let mut left = BspNode::new(Rect::new(
                self.region.x1, self.region.y1,
                split - self.region.x1, h,
            ));
            let mut right = BspNode::new(Rect::new(
                split, self.region.y1,
                self.region.x2 - split, h,
            ));
            left.split(rng, depth - 1);
            right.split(rng, depth - 1);
            self.left = Some(Box::new(left));
            self.right = Some(Box::new(right));
        }
    }

    fn create_room(&mut self, rng: &mut StdRng) {
        let w = self.region.width();
        let h = self.region.height();
        if w < MIN_ROOM_SIZE + 2 || h < MIN_ROOM_SIZE + 2 { return; }
        let max_w = w - 2;
        let max_h = h - 2;
        let room_w = rng.gen_range(MIN_ROOM_SIZE..(max_w + 1));
        let room_h = rng.gen_range(MIN_ROOM_SIZE..(max_h + 1));
        let rx = self.region.x1 + 1 + rng.gen_range(0..(max_w - room_w + 1));
        let ry = self.region.y1 + 1 + rng.gen_range(0..(max_h - room_h + 1));
        self.room = Some(Rect::new(rx, ry, room_w, room_h));
    }

    fn collect_rooms(&self, rooms: &mut Vec<Rect>) {
        if let Some(room) = &self.room { rooms.push(room.clone()); }
        if let Some(l) = &self.left { l.collect_rooms(rooms); }
        if let Some(r) = &self.right { r.collect_rooms(rooms); }
    }

    /// เชื่อมต่อ rooms และคืน center ของ room ที่ใกล้ที่สุด
    fn connect_rooms(&self, map: &mut Map) -> Option<(usize, usize)> {
        match (&self.left, &self.right) {
            (Some(left), Some(right)) => {
                let lc = left.connect_rooms(map);
                let rc = right.connect_rooms(map);
                if let (Some(l), Some(r)) = (lc, rc) {
                    map.carve_corridor_h_then_v(l, r);
                    Some(l)
                } else {
                    lc.or(rc)
                }
            }
            _ => self.room.as_ref().map(|r| r.center()),
        }
    }
}

pub fn generate_dungeon(map: &mut Map, seed: u64) -> Vec<Rect> {
    let mut rng = StdRng::seed_from_u64(seed);
    let root_region = Rect::new(1, 1, map.width - 2, map.height - 2);
    let mut root = BspNode::new(root_region);
    root.split(&mut rng, MAX_DEPTH);

    let mut rooms = Vec::new();
    root.collect_rooms(&mut rooms);

    for room in &rooms {
        map.carve_room(room);
    }
    root.connect_rooms(map);

    // วาง stairs down ที่ห้องสุดท้าย
    if let Some(last) = rooms.last() {
        let (cx, cy) = last.center();
        if cx < map.width && cy < map.height {
            let idx = map.idx(cx, cy);
            map.tiles[idx] = Tile::stairs_down();
        }
    }
    rooms
}
```

**Dungeon ที่สร้างออกมา (แผนที่ตัวอย่าง seed=42):**

```
################################################################################
#     #####################################      ##########              ########
#     #####################################      ##########              ########
#  @  #####################################  g   ##########  g           ########
#     #####################################      ##########              ########
##    #####################################      ##########              ########
## ################################### ###########        ###############  ######
## ######  ########  ####  ##########  ###########  g     ###  ########     #####
##        ##      #  ####  ##########  ####   ####        ###  ########  g  #####
##        ##  g   #  ####  ##########  ####  .####  g     ###  ########     #####
##        ##      #  ####  ##########  ####   ####        ###  ########     #####
```

---

### ขั้นที่ 4: Field of View ด้วย Bresenham Ray Casting

**`src/fov.rs`:**

```rust
use std::collections::HashSet;
use crate::map::Map;

/// คำนวณ FOV จาก origin ในรัศมี radius
/// ใช้ Bresenham ray casting ไปยังทุก tile ในรัศมี
pub fn compute_fov(map: &Map, ox: i32, oy: i32, radius: i32)
    -> HashSet<(i32, i32)>
{
    let mut visible = HashSet::new();
    visible.insert((ox, oy));

    for dy in -radius..=radius {
        for dx in -radius..=radius {
            if dx * dx + dy * dy > radius * radius { continue; }
            let tx = ox + dx;
            let ty = oy + dy;
            if !map.in_bounds(tx, ty) { continue; }
            if has_los(map, ox, oy, tx, ty) {
                visible.insert((tx, ty));
            }
        }
    }
    visible
}

/// ตรวจสอบ Line of Sight ด้วย Bresenham's line algorithm
pub fn has_los(map: &Map, x1: i32, y1: i32, x2: i32, y2: i32) -> bool {
    let mut x = x1;
    let mut y = y1;
    let dx = (x2 - x1).abs();
    let dy = (y2 - y1).abs();
    let sx = if x2 > x1 { 1 } else { -1 };
    let sy = if y2 > y1 { 1 } else { -1 };
    let mut err = dx - dy;

    loop {
        if x == x2 && y == y2 { return true; }
        // tile ที่ไม่ใช่ origin และไม่ใช่ goal บัง sight
        if (x != x1 || y != y1) && map.blocks_sight(x, y) {
            return false;
        }
        let e2 = 2 * err;
        if e2 > -dy { err -= dy; x += sx; }
        if e2 < dx  { err += dx; y += sy; }
    }
}

/// อัปเดต map visibility ตาม FOV และ set explored flag
pub fn update_map_visibility(
    map: &mut Map,
    visible: &HashSet<(i32, i32)>,
) {
    map.clear_visibility();
    for &(x, y) in visible {
        if map.in_bounds(x, y) {
            let t = map.tile_mut(x as usize, y as usize);
            t.visible = true;
            t.explored = true; // fog of war: เคยเห็นแล้ว
        }
    }
}
```

**Fog of War rendering logic:**

```rust
// ใน render system — แต่ละ tile แสดงผลต่างกัน
fn render_tile(tile: &Tile) -> (char, Color) {
    if tile.visible {
        // มองเห็นอยู่ — สีเต็ม
        match tile.kind {
            TileKind::Floor => ('.', Color::White),
            TileKind::Wall  => ('#', Color::Gray),
            TileKind::Door  => ('+', Color::Yellow),
            TileKind::StairsDown => ('>', Color::Cyan),
            TileKind::StairsUp   => ('<', Color::Cyan),
        }
    } else if tile.explored {
        // เคยเห็นแล้ว — สีหมอง (dim)
        match tile.kind {
            TileKind::Floor => ('.', Color::DarkGray),
            TileKind::Wall  => ('#', Color::DarkGray),
            _               => (' ', Color::Black),
        }
    } else {
        // ไม่เคยเห็น — ดำสนิท
        (' ', Color::Black)
    }
}
```

---

### ขั้นที่ 5: Combat System และ Monster AI

**`src/combat.rs`:**

```rust
use crate::components::{CombatStats, Entity};
use crate::ecs::World;

/// สูตรความเสียหาย: damage = max(0, power - defense)
pub fn calc_damage(power: i32, defense: i32) -> i32 {
    (power - defense).max(0)
}

/// โจมตี attacker → defender
/// คืนค่า damage จริงที่เกิดขึ้น
pub fn attack(world: &mut World, attacker: Entity, defender: Entity) -> i32 {
    let (power, defense) = {
        let a = world.get::<CombatStats>(attacker);
        let d = world.get::<CombatStats>(defender);
        match (a, d) {
            (Some(a), Some(d)) => (a.power, d.defense),
            _ => return 0,
        }
    };
    let damage = calc_damage(power, defense);
    if damage > 0 {
        if let Some(stats) = world.get_mut::<CombatStats>(defender) {
            stats.hp -= damage;
        }
    }
    damage
}

pub fn is_dead(world: &World, entity: Entity) -> bool {
    world.get::<CombatStats>(entity)
        .map(|s| s.hp <= 0)
        .unwrap_or(false)
}
```

**`src/ai.rs`** — Monster AI State Machine:

```rust
use crate::components::{Entity, Position};
use crate::ecs::World;
use crate::map::Map;
use crate::pathfind;
use rand::Rng;

pub enum AiState { Idle, Chasing, RandomWalk }

pub const CHASE_RANGE: i32 = 8;

/// Manhattan distance สำหรับ range check
pub fn manhattan_distance(ax: i32, ay: i32, bx: i32, by: i32) -> i32 {
    (ax - bx).abs() + (ay - by).abs()
}

/// ให้ monster ตัดสินใจและเคลื่อนที่
/// คืนค่า entity ที่ต้องการโจมตี (ถ้ามี)
pub fn run_monster_ai(
    world: &mut World,
    map: &Map,
    monster: Entity,
    player: Entity,
    rng: &mut impl Rng,
) -> Option<Entity> {
    let (mx, my) = {
        let pos = world.get::<Position>(monster)?;
        (pos.x, pos.y)
    };
    let (px, py) = {
        let pos = world.get::<Position>(player)?;
        (pos.x, pos.y)
    };

    let dist = manhattan_distance(mx, my, px, py);

    if dist <= 1 {
        // ติดกัน → โจมตี
        return Some(player);
    }

    if dist <= CHASE_RANGE {
        // Chase: ใช้ A* หาเส้นทาง
        let next = pathfind::step_toward(
            map,
            (mx as usize, my as usize),
            (px as usize, py as usize),
        );
        let (nx, ny) = (next.0 as i32, next.1 as i32);
        if map.is_passable(nx, ny) && !(nx == px && ny == py) {
            if let Some(pos) = world.get_mut::<Position>(monster) {
                pos.x = nx;
                pos.y = ny;
            }
        }
    } else {
        // Random walk
        let dirs = [(0i32,-1i32),(0,1),(-1,0),(1,0)];
        let idx = rng.gen_range(0..4);
        let (dx, dy) = dirs[idx];
        if map.is_passable(mx + dx, my + dy) {
            if let Some(pos) = world.get_mut::<Position>(monster) {
                pos.x += dx;
                pos.y += dy;
            }
        }
    }
    None
}
```

**Monster Templates — ตารางสถิติ:**

| Monster | Glyph | HP | Power | Defense | ระดับที่พบ |
|---------|-------|-----|-------|---------|-----------|
| Goblin  | `g`   | 6   | 3     | 0       | ทุกระดับ  |
| Orc     | `o`   | 10  | 4     | 1       | ระดับ 2+ |
| Troll   | `t`   | 16  | 6     | 3       | ระดับ 4+ |

---

### ขั้นที่ 6: Items, Inventory และ Dungeon Levels

**`src/items.rs`:**

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum ItemKind {
    HealthPotion,  // ฮีล 15 HP
    MagicScroll,   // ดีลความเสียหาย magic
    Sword,         // +3 power
    Shield,        // +2 defense
}

impl ItemKind {
    pub fn display_name(&self) -> &'static str {
        match self {
            ItemKind::HealthPotion => "Health Potion",
            ItemKind::MagicScroll  => "Magic Scroll",
            ItemKind::Sword        => "Sword",
            ItemKind::Shield       => "Shield",
        }
    }

    pub fn heal_amount(&self) -> i32 {
        match self {
            ItemKind::HealthPotion => 15,
            _ => 0,
        }
    }

    pub fn power_bonus(&self) -> i32 {
        match self { ItemKind::Sword => 3, _ => 0 }
    }

    pub fn defense_bonus(&self) -> i32 {
        match self { ItemKind::Shield => 2, _ => 0 }
    }
}
```

**Dungeon Level Scaling:**

```rust
pub struct LevelConfig {
    pub depth: u32,
}

impl LevelConfig {
    /// scale monster power ตาม depth
    pub fn monster_power_bonus(&self) -> i32 {
        (self.depth as i32 - 1) * 1
    }

    /// scale monster defense ตาม depth
    pub fn monster_defense_bonus(&self) -> i32 {
        (self.depth as i32 / 2)
    }
}

/// สร้าง level ใหม่ — preserve player entity
pub fn next_level(world: &mut World, depth: u32) -> Map {
    // ลบ monsters และ items ทั้งหมด
    let dead: Vec<Entity> = world.query::<Monster>()
        .iter()
        .map(|(e, _)| *e)
        .collect();
    for e in dead {
        world.remove_entity(e);
    }

    // สร้างแผนที่ใหม่
    let mut map = Map::new(MAP_WIDTH, MAP_HEIGHT);
    let rooms = bsp::generate_dungeon(&mut map, depth as u64 * 12345);

    // spawn monsters ที่แรงขึ้น
    let config = LevelConfig { depth };
    for room in rooms.iter().skip(1) {
        let center = room.center();
        let base_power = 3 + config.monster_power_bonus();
        let base_defense = config.monster_defense_bonus();
        world.spawn()
            .with(Position { x: center.0 as i32, y: center.1 as i32 })
            .with(CombatStats {
                hp: 10, max_hp: 10,
                power: base_power,
                defense: base_defense,
            })
            .with(Monster)
            .with(Renderable { glyph: 'o', fg: (255, 100, 0) })
            .build(world);
    }

    map
}
```

**Turn-based Game Loop:**

```rust
pub enum PlayerAction {
    Move(i32, i32),         // delta x, y
    PickUp,                  // เก็บ item ที่พื้น
    UseItem(usize),          // index ใน inventory
    DropItem(usize),         // index ใน inventory
    Descend,                 // ลง stairs
    Ascend,                  // ขึ้น stairs
    Wait,                    // รอ (ให้ monsters เดิน)
    Quit,
}

pub fn run_game_loop(state: &mut GameState) -> bool {
    // 1. อ่าน input
    let action = read_input();
    if matches!(action, PlayerAction::Quit) { return false; }

    // 2. ประมวล player action
    let mut player_acted = false;
    match action {
        PlayerAction::Move(dx, dy) => {
            player_acted = try_move_player(state, dx, dy);
        }
        PlayerAction::PickUp => {
            player_acted = try_pickup(state);
        }
        PlayerAction::UseItem(idx) => {
            player_acted = use_item(state, idx);
        }
        PlayerAction::Descend => {
            player_acted = try_descend(state);
        }
        PlayerAction::Wait => {
            player_acted = true;
        }
        _ => {}
    }

    // 3. ถ้า player ทำอะไรบางอย่าง → monsters ก็ทำ
    if player_acted {
        // อัปเดต FOV
        let fov = compute_fov(&state.map, state.player_x, state.player_y, 8);
        update_map_visibility(&mut state.map, &fov);

        // monster AI
        run_all_monsters(state);

        // ลบ entities ที่ตาย
        cleanup_dead(state);

        state.turn += 1;
    }

    // 4. Render
    render(&state);
    true
}
```

---

### ขั้นที่ 7: Terminal UI ด้วย ratatui

**`src/main.rs`** (ส่วน TUI):

```rust
use ratatui::{
    backend::CrosstermBackend,
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Modifier, Style},
    text::{Line, Span},
    widgets::{Block, Borders, Paragraph},
    Frame, Terminal,
};
use crossterm::{
    event::{self, Event, KeyCode},
    execute,
    terminal::{disable_raw_mode, enable_raw_mode,
               EnterAlternateScreen, LeaveAlternateScreen},
};

pub fn setup_terminal() -> std::io::Result<Terminal<CrosstermBackend<std::io::Stdout>>> {
    enable_raw_mode()?;
    let mut stdout = std::io::stdout();
    execute!(stdout, EnterAlternateScreen)?;
    let backend = CrosstermBackend::new(stdout);
    Terminal::new(backend)
}

pub fn restore_terminal(
    term: &mut Terminal<CrosstermBackend<std::io::Stdout>>
) -> std::io::Result<()> {
    disable_raw_mode()?;
    execute!(term.backend_mut(), LeaveAlternateScreen)?;
    Ok(())
}

pub fn render_game(frame: &mut Frame, state: &GameState) {
    // แบ่ง layout: map (80%) | sidebar (20%)
    let chunks = Layout::default()
        .direction(Direction::Horizontal)
        .constraints([
            Constraint::Min(60),
            Constraint::Length(20),
        ])
        .split(frame.area());

    render_map(frame, state, chunks[0]);
    render_sidebar(frame, state, chunks[1]);
}

fn render_map(frame: &mut Frame, state: &GameState, area: Rect) {
    let mut lines: Vec<Line> = Vec::new();

    for y in 0..(state.map.height.min(area.height as usize)) {
        let mut spans: Vec<Span> = Vec::new();
        for x in 0..(state.map.width.min(area.width as usize)) {
            let tile = &state.map.tiles[state.map.idx(x, y)];

            // ตรวจสอบว่ามี entity อยู่ที่ (x,y) หรือเปล่า
            let (glyph, color) = if tile.visible {
                get_entity_at(state, x as i32, y as i32)
                    .unwrap_or_else(|| tile_glyph(tile))
            } else if tile.explored {
                dim_tile_glyph(tile)
            } else {
                (' ', Color::Black)
            };

            spans.push(Span::styled(
                glyph.to_string(),
                Style::default().fg(color),
            ));
        }
        lines.push(Line::from(spans));
    }

    let para = Paragraph::new(lines)
        .block(Block::default()
            .title(format!(" Dungeon Level {} ", state.depth))
            .borders(Borders::ALL));
    frame.render_widget(para, area);
}

fn render_sidebar(frame: &mut Frame, state: &GameState, area: Rect) {
    let player_stats = state.world.get::<CombatStats>(state.player).unwrap();
    let player_name = state.world.get::<Name>(state.player).unwrap();

    let text = vec![
        Line::from(Span::styled(
            &player_name.0,
            Style::default().add_modifier(Modifier::BOLD),
        )),
        Line::from(""),
        Line::from(format!("HP: {}/{}", player_stats.hp, player_stats.max_hp)),
        Line::from(format!("ATK: {}  DEF: {}", player_stats.power, player_stats.defense)),
        Line::from(""),
        Line::from(format!("Depth: {}", state.depth)),
        Line::from(format!("Turn: {}", state.turn)),
        Line::from(""),
        Line::from("─── Controls ───"),
        Line::from("Arrow/HJKL: move"),
        Line::from("g: pick up"),
        Line::from("i: inventory"),
        Line::from(".: wait"),
        Line::from(">: descend"),
        Line::from("q: quit"),
    ];

    let para = Paragraph::new(text)
        .block(Block::default().title(" Stats ").borders(Borders::ALL));
    frame.render_widget(para, area);
}
```

---

### ขั้นที่ 8: Save/Load System

**`src/saveload.rs`:**

```rust
use serde::{Deserialize, Serialize};
use crate::ecs::World;
use crate::map::Map;

#[derive(Serialize, Deserialize)]
pub struct SaveData {
    pub version: u32,
    pub depth: u32,
    pub map: Map,
    pub entity_count: usize,
}

/// Path บันทึก save file
pub fn save_path() -> std::path::PathBuf {
    let mut path = dirs::data_local_dir()
        .unwrap_or_else(|| std::path::PathBuf::from("."));
    path.push("rogue");
    path.push("save.json");
    path
}

pub fn serialize_world(world: &World, map: &Map, depth: u32) -> String {
    let data = SaveData {
        version: 1,
        depth,
        map: map.clone(),
        entity_count: world.entity_count(),
    };
    serde_json::to_string(&data).unwrap_or_default()
}

pub fn save_game(world: &World, map: &Map, depth: u32) -> std::io::Result<()> {
    let path = save_path();
    std::fs::create_dir_all(path.parent().unwrap())?;
    let json = serialize_world(world, map, depth);
    std::fs::write(&path, json)
}

pub fn load_game() -> Option<SaveData> {
    let path = save_path();
    let json = std::fs::read_to_string(&path).ok()?;
    serde_json::from_str(&json).ok()
}

/// ลบ save file เมื่อ player ตาย
pub fn delete_save() {
    if let Ok(path) = std::env::current_dir() {
        let _ = std::fs::remove_file(save_path());
    }
}
```

**Game startup logic:**

```rust
fn main() -> std::io::Result<()> {
    // ตรวจสอบว่ามี save file หรือเปล่า
    let (world, map, depth) = if let Some(save) = saveload::load_game() {
        println!("กำลังโหลด save...");
        // reconstruct world จาก save data
        (World::new(), save.map, save.depth)
    } else {
        // เกมใหม่
        let mut world = World::new();
        let mut map = Map::new(MAP_WIDTH, MAP_HEIGHT);
        let rooms = bsp::generate_dungeon(&mut map, rand::random());
        // spawn player...
        (world, map, 1)
    };

    // เริ่ม game loop...
    Ok(())
}
```

## การทดสอบ (Testing)

### Unit Tests ทั้งหมด

โปรเจคมี 27 unit tests ครอบคลุมทุก module สำคัญ:

```
cargo test
```

**ผลลัพธ์จริงจากการรัน:**

```
running 27 tests
test ai::tests::test_chase_range_constant ... ok
test ai::tests::test_manhattan_distance ... ok
test combat::tests::test_calc_damage_basic ... ok
test combat::tests::test_attack_fully_blocked ... ok
test combat::tests::test_is_dead ... ok
test combat::tests::test_kill_removes_entity ... ok
test combat::tests::test_attack_reduces_hp ... ok
test ecs::tests::test_query ... ok
test ecs::tests::test_inventory_add_remove ... ok
test ecs::tests::test_remove_entity ... ok
test ecs::tests::test_spawn_and_get_component ... ok
test fov::tests::test_fov_open_area_sees_nearby ... ok
test fov::tests::test_los_blocked_by_wall ... ok
test fov::tests::test_los_clear_line ... ok
test fov::tests::test_fov_radius_limit ... ok
test fov::tests::test_fov_origin_always_visible ... ok
test fov::tests::test_fov_wall_blocks_sight ... ok
test items::tests::test_item_effects ... ok
test map::tests::test_rect_intersects ... ok
test map::tests::test_map_bounds ... ok
test map::tests::test_bsp_floors_exist ... ok
test map::tests::test_bsp_rooms_non_overlapping ... ok
test pathfind::tests::test_astar_no_path_fully_blocked ... ok
test pathfind::tests::test_astar_respects_walls ... ok
test pathfind::tests::test_astar_shortest_path_straight_line ... ok
test pathfind::tests::test_astar_finds_path ... ok
test saveload::tests::test_serialize_deserialize ... ok

test result: ok. 27 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

### Test Details — สิ่งที่ test แต่ละชุดตรวจสอบ

#### BSP Room Generation Tests

```rust
#[test]
fn test_bsp_rooms_non_overlapping() {
    let mut map = Map::new(MAP_WIDTH, MAP_HEIGHT);
    let rooms = generate_dungeon(&mut map, 12345);

    assert!(rooms.len() >= 2, "ต้องมีอย่างน้อย 2 ห้อง");

    // ตรวจสอบว่าห้องไม่ซ้อนกัน
    for i in 0..rooms.len() {
        for j in (i+1)..rooms.len() {
            let a = &rooms[i];
            let b = &rooms[j];
            let overlap = a.x1 < b.x2 && a.x2 > b.x1
                       && a.y1 < b.y2 && a.y2 > b.y1;
            assert!(!overlap,
                "ห้อง {} และ {} ซ้อนกัน: {:?} vs {:?}", i, j, a, b);
        }
    }
}

#[test]
fn test_bsp_floors_exist() {
    let mut map = Map::new(MAP_WIDTH, MAP_HEIGHT);
    generate_dungeon(&mut map, 99);
    let floor_count = map.tiles.iter()
        .filter(|t| t.kind == TileKind::Floor)
        .count();
    assert!(floor_count > 100, "ต้องมี floor tiles เพียงพอ");
}
```

#### FOV Tests

```rust
#[test]
fn test_fov_wall_blocks_sight() {
    let mut map = all_floor_map(30, 30);
    // กำแพงแนวตั้งที่ x=10
    for y in 0..30 {
        *map.tile_mut(10, y) = Tile::wall();
    }
    let visible = compute_fov(&map, 5, 15, 25);

    // กำแพงเองมองเห็นได้
    assert!(visible.contains(&(10, 15)));
    // ข้ามกำแพงไม่มองเห็น
    assert!(!visible.contains(&(20, 15)));
    assert!(!visible.contains(&(15, 15)));
}

#[test]
fn test_los_blocked_by_wall() {
    let mut map = all_floor_map(20, 20);
    *map.tile_mut(5, 5) = Tile::wall(); // กำแพงกั้น
    assert!(!has_los(&map, 2, 5, 8, 5));
}
```

#### A\* Pathfinding Tests

```rust
#[test]
fn test_astar_finds_path() {
    let map = open_map(20, 20);
    let path = find_path(&map, (1, 1), (18, 18));
    assert!(path.is_some());
    let path = path.unwrap();
    assert_eq!(*path.first().unwrap(), (1, 1));
    assert_eq!(*path.last().unwrap(), (18, 18));
}

#[test]
fn test_astar_no_path_fully_blocked() {
    let mut map = open_map(10, 10);
    // ล้อมรอบ goal ด้วยกำแพง
    for dy in -1i32..=1 {
        for dx in -1i32..=1 {
            *map.tile_mut((8+dx) as usize, (5+dy) as usize) = Tile::wall();
        }
    }
    assert!(find_path(&map, (1, 5), (8, 5)).is_none());
}
```

#### Combat Tests

```rust
#[test]
fn test_calc_damage_basic() {
    assert_eq!(calc_damage(5, 2), 3);   // 5-2=3
    assert_eq!(calc_damage(3, 3), 0);   // 3-3=0
    assert_eq!(calc_damage(2, 5), 0);   // defense > power → 0
    assert_eq!(calc_damage(10, 0), 10); // ไม่มี defense
}

#[test]
fn test_attack_reduces_hp() {
    let mut world = World::new();
    let attacker = world.spawn()
        .with(CombatStats { hp: 20, max_hp: 20, defense: 0, power: 5 })
        .build(&mut world);
    let defender = world.spawn()
        .with(CombatStats { hp: 10, max_hp: 10, defense: 2, power: 3 })
        .build(&mut world);

    let dmg = attack(&mut world, attacker, defender);
    assert_eq!(dmg, 3); // 5 - 2 = 3
    assert_eq!(world.get::<CombatStats>(defender).unwrap().hp, 7);
}
```

#### Inventory Tests

```rust
#[test]
fn test_inventory_add_remove() {
    let mut inv = Inventory::new(3);
    assert!(inv.add(0));
    assert!(inv.add(1));
    assert!(inv.add(2));
    assert!(!inv.add(3)); // เต็มแล้ว
    assert_eq!(inv.items.len(), 3);

    assert!(inv.remove(1));
    assert_eq!(inv.items.len(), 2);
    assert!(!inv.remove(99)); // ไม่มี entity นี้
}
```

### ผลลัพธ์ Binary Demo

```
=== Roguelike Dungeon Crawler Demo ===

[Map] สร้างแผนที่ขนาด 80x45 — 17 ห้อง
[ECS] สร้าง player entity #0 ที่ (5, 5)
[ECS] สร้าง 16 monster entities
[FOV] มองเห็น 57 tiles จากจุดเริ่มต้น
[Combat] ความเสียหาย: power=5, defense=2 → 3
[A*] หาเส้นทางจาก (5, 5) ไป (69, 22): 84 ก้าว
[Inventory] เพิ่ม Health Potion เข้า inventory (size=1)
[Save] serialize world สำเร็จ (301695 bytes)

=== Demo เสร็จสมบูรณ์ ===
```

## Pitfalls และข้อควรระวัง

### Pitfall 1: ECS Borrow Checker กับการ Query หลาย Components พร้อมกัน

**ปัญหา:** อยากดึง `Position` และ `CombatStats` ของ entity เดียวกันพร้อมกัน แต่ `get()` กับ `get_mut()` conflict กัน

```rust
// ❌ ERROR: cannot borrow `world` as mutable because it is also borrowed as immutable
let pos = world.get::<Position>(player);       // borrows world immutably
let stats = world.get_mut::<CombatStats>(player); // borrows world mutably!
```

**วิธีแก้:** ดึงค่าออกมาก่อน (copy/clone) หรือใช้ scope แยก:

```rust
// ✓ ดึงค่าออกมาก่อน
let (px, py) = {
    let pos = world.get::<Position>(player).unwrap();
    (pos.x, pos.y)
}; // borrow จบที่ }

// แล้วค่อย borrow mutable
if let Some(stats) = world.get_mut::<CombatStats>(player) {
    stats.hp -= 5;
}
```

**ทำไม Rust ถึงเข้มงวด:** หลักการ "aliasing XOR mutation" ป้องกัน data race ในทุกสถานการณ์ รวมถึง single-threaded ด้วย เพราะ Rust ไม่รู้ว่า `Position` กับ `CombatStats` อยู่ใน `Vec` คนละอันหรือเปล่า

### Pitfall 2: BSP ห้องซ้อนกันเพราะ Off-by-one ใน Rect

**ปัญหา:** `Rect::new(x, y, w, h)` หมายถึงอะไรกันแน่? ถ้า x1=5, x2=10 นั่นคือ width = 5 หรือ 6?

```rust
// ❌ ผิด: ถ้าใช้ inclusive ทั้งสองด้าน
pub fn carve_room(&mut self, room: &Rect) {
    for y in room.y1..=room.y2 { // รวม y2!
        for x in room.x1..=room.x2 { // รวม x2!
```

ปัญหาคือ border ของ partition จะทับกับ border ของ partition ถัดไป ทำให้ห้องชนกัน

**วิธีแก้:** ใช้ convention ชัดเจน — `[x1, x2)` exclusive ที่ปลาย และ carve floor เฉพาะ `(x1+1, x2)` เพื่อเว้น border:

```rust
// ✓ เว้น border wall รอบห้อง
pub fn carve_room(&mut self, room: &Rect) {
    for y in (room.y1 + 1)..room.y2 { // exclusive end
        for x in (room.x1 + 1)..room.x2 {
```

### Pitfall 3: A\* Memory Explosion บน Map ขนาดใหญ่

**ปัญหา:** `pathfinding::astar` ต้องเก็บ open/closed sets ซึ่งอาจกิน RAM มากถ้า map ใหญ่มากและ path ยาวมาก

```rust
// ❌ ปัญหา: ไม่จำกัด path length
pub fn find_path(map: &Map, start: (usize,usize), goal: (usize,usize))
    -> Option<Vec<(usize,usize)>>
{
    astar(&start, |&pos| successors(map, pos), |&pos| heuristic(pos, goal),
          |&pos| pos == goal)
}
```

**วิธีแก้:** จำกัดระยะทางสูงสุด หรือใช้ heuristic ที่ aggressive กว่า:

```rust
// ✓ จำกัดด้วย max cost
const MAX_PATH_COST: i32 = 500; // ≈50 tiles

fn heuristic(pos: (i32,i32), goal: (i32,i32)) -> i32 {
    // admissible: ไม่ overestimate
    ((pos.0 - goal.0).abs() + (pos.1 - goal.1).abs()) * 10
}

// หรือ: ถ้า Manhattan distance เกิน range ไม่ต้อง pathfind
pub fn step_toward(map: &Map, from: (usize,usize), to: (usize,usize))
    -> (usize,usize)
{
    let dist = manhattan_distance(
        from.0 as i32, from.1 as i32,
        to.0 as i32, to.1 as i32
    );
    if dist > CHASE_RANGE * 2 { return from; } // ไม่ pathfind ถ้าไกลเกิน
    // ...
}
```

### Pitfall 4: Serde Serialize กับ `Box<dyn Any>`

**ปัญหา:** `Box<dyn Any>` ไม่ implement `Serialize` โดยตรง ถ้าพยายาม serialize `World` ทั้งก้อนจะ compile error

```rust
// ❌ ไม่ compile
#[derive(Serialize)]
pub struct World {
    components: HashMap<TypeId, Vec<Option<Box<dyn Any>>>>, // ❌ Any ไม่ Serialize
}
```

**วิธีแก้ที่ใช้ในโปรเจคนี้:** สร้าง `SaveData` struct แยกที่เก็บเฉพาะ data ที่ต้องการบันทึก แล้ว serialize เฉพาะ struct นั้น:

```rust
// ✓ แยก save data ออกจาก runtime World
#[derive(Serialize, Deserialize)]
pub struct SaveData {
    pub version: u32,
    pub depth: u32,
    pub map: Map,       // Map implement Serialize ได้
    pub entity_count: usize,
    // เพิ่ม: player_pos, player_stats, inventory, etc.
}
```

สำหรับ full save ที่เก็บ entities ทั้งหมด ต้องเพิ่ม `#[typetag::serde]` หรือใช้ `bevy_reflect` — แต่นั่นซับซ้อนกว่าสำหรับโปรเจคนี้

### Pitfall 5: ratatui กับ Raw Mode Terminal

**ปัญหา:** ถ้าโปรแกรม panic ระหว่างที่ raw mode เปิดอยู่ terminal จะพัง (ไม่แสดงผลปกติ)

```rust
// ❌ ถ้า panic ที่นี่ terminal ค้าง
let terminal = setup_terminal()?;
run_game_loop(&mut game_state); // panic!
restore_terminal(&mut terminal)?; // ไม่ถูกเรียก
```

**วิธีแก้:** ใช้ panic hook เพื่อ restore terminal ก่อนแสดง error:

```rust
// ✓ setup panic handler
let original_hook = std::panic::take_hook();
std::panic::set_hook(Box::new(move |info| {
    // restore terminal ก่อน
    let _ = disable_raw_mode();
    let _ = execute!(std::io::stdout(), LeaveAlternateScreen);
    original_hook(info);
}));
```

หรือใช้ `color_eyre` / `better-panic` crate ที่จัดการให้อัตโนมัติ

## การ Package และ Deploy

### Build Release Binary

```bash
# Build แบบ release (optimized)
cargo build --release

# Binary อยู่ที่
./target/release/roguelike
```

### Cross-compile สำหรับ Windows

```bash
# ติดตั้ง target
rustup target add x86_64-pc-windows-gnu

# Build
cargo build --release --target x86_64-pc-windows-gnu
```

### Cargo.toml — Profile Settings

```toml
[profile.release]
opt-level = 3
lto = true         # Link-time optimization
codegen-units = 1  # Single codegen unit สำหรับ LTO
strip = true       # Strip debug symbols

[profile.dev]
opt-level = 1      # เพิ่ม opt level เล็กน้อยให้ dev build เร็วขึ้น
```

### Distribution

สำหรับ distributing roguelike:

```
roguelike-v1.0-linux-x64.tar.gz
├── roguelike          ← binary
└── README.md
```

ไม่ต้องการ runtime dependency เพิ่มเติม — Rust binary เป็น self-contained

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Equipment System ⭐⭐

ปัจจุบัน Sword และ Shield เป็นแค่ Item เมื่อ "ใช้" แล้วจะหายไป ให้เพิ่มระบบ **equipment** ที่แยกระหว่าง consumables กับ equippable items:

```rust
#[derive(Debug, Clone)]
pub struct Equipment {
    pub main_hand: Option<Entity>,  // อาวุธ
    pub off_hand: Option<Entity>,   // โล่
    pub armor: Option<Entity>,      // เกราะ
}

// ระบบ: ถ้าใส่ Sword แล้ว power เพิ่ม 3
// ระบบ: ใส่ได้แค่ 1 ชิ้นต่อ slot — unequip ชิ้นเก่าก่อน
```

**Hints:**
- เพิ่ม `Equipment` component ให้ player
- `use_item` ตรวจสอบว่า item เป็น consumable หรือ equippable
- ใน combat calculation อย่าลืมรวม equipment bonus

### แบบฝึกหัดที่ 2: Cellular Automata Cave Generator ⭐⭐

นอกจาก BSP ให้เพิ่ม map generator แบบ **cellular automata** สำหรับสร้าง cave ที่ดูเป็นธรรมชาติมากขึ้น:

```rust
pub fn generate_cave(map: &mut Map, seed: u64, iterations: u32) {
    let mut rng = StdRng::seed_from_u64(seed);

    // 1. เติม map แบบ random (45% wall)
    for tile in &mut map.tiles {
        *tile = if rng.gen_bool(0.45) { Tile::wall() } else { Tile::floor() };
    }

    // 2. ทำ cellular automata iterations
    for _ in 0..iterations {
        // rule: ถ้า neighbor walls >= 5 → กลายเป็น wall
        // ถ้า neighbor walls <= 3 → กลายเป็น floor
    }

    // 3. flood fill เพื่อตัด disconnected regions ออก
}
```

### แบบฝึกหัดที่ 3: Particle Effects และ Animation ⭐⭐⭐

เพิ่ม visual feedback สำหรับ combat ด้วย particle system เล็ก ๆ:

```rust
pub struct Particle {
    pub x: f32, pub y: f32,
    pub dx: f32, pub dy: f32,
    pub glyph: char,
    pub color: (u8, u8, u8),
    pub lifetime: f32,  // seconds
}

// เมื่อโจมตีสำเร็จ spawn blood splatter particles
// เมื่อ heal spawn green + symbols
// render particles ทับบน map
```

**Hints:**
- ใช้ `std::time::Instant` สำหรับ frame timing
- ratatui รองรับ color แบบ RGB ผ่าน `Color::Rgb(r, g, b)`
- particle ต้องถูก render หลัง map tiles แต่ก่อน UI

### แบบฝึกหัดที่ 4: High Score Leaderboard ⭐⭐

เพิ่มระบบ leaderboard ที่บันทึก score เมื่อ player ตาย:

```rust
#[derive(Serialize, Deserialize)]
pub struct ScoreEntry {
    pub name: String,
    pub score: u32,
    pub depth: u32,
    pub kills: u32,
    pub date: String,
}

// Score คำนวณจาก: depth * 100 + kills * 10 + gold * 1
// บันทึกลง ~/.local/share/rogue/scores.json
// แสดงใน menu หน้าจอ
```

**Hints:**
- ใช้ `serde_json::from_str::<Vec<ScoreEntry>>()` โหลด leaderboard
- sort ด้วย `scores.sort_by(|a, b| b.score.cmp(&a.score))`
- เก็บเฉพาะ top 10

## สรุป

โปรเจคนี้สร้าง roguelike dungeon crawler ที่มีระบบครบครัน:

**ระบบหลักที่ได้ build:**
- **ECS lite** ด้วย `Box<dyn Any>` storage และ TypeId indexing — เข้าใจว่า game engine ทันสมัยทำงานยังไง
- **BSP dungeon generation** — recursive tree partition สร้างห้องที่ไม่ซ้อนกันรับประกัน
- **Bresenham FOV** — line-of-sight ที่ถูกต้องและ symmetric พร้อม fog of war
- **A\* pathfinding** ผ่าน `pathfinding` crate — monster AI ที่เดินอ้อมกำแพงได้
- **Turn-based combat** ด้วยสูตร `damage = max(0, power - defense)` ที่เรียบง่ายแต่มีกลยุทธ์
- **Item + Inventory** ด้วย capacity limit และ use effects
- **Dungeon levels** ที่ enemy แข็งขึ้นตาม depth
- **JSON save/load** ด้วย serde

**Pattern สำคัญที่ได้เรียน:**
1. **Component-based design** แยก data และ behavior ออกจากกัน ทำให้เพิ่ม feature ใหม่ไม่ต้องแก้ code เก่า
2. **TypeId + Box\<dyn Any\>** เป็นเทคนิค type erasure ที่ใช้กว้างขวางใน Rust ecosystem
3. **Builder pattern** สำหรับ object construction ที่มี optional components หลายตัว
4. **Seed-based random** ทำให้ map reproducible — สำคัญมากสำหรับ testing และ debugging

**เชื่อมโยงไปโปรเจคถัดไป:** Project E07 (Map Generator) จะเจาะลึก procedural generation algorithm มากขึ้น ได้แก่ Perlin noise สำหรับ terrain, room connection algorithms แบบต่าง ๆ, และ biome system — ต่อยอดจาก BSP ที่เรียนในโปรเจคนี้โดยตรง

---

**โปรเจคก่อนหน้า:** [Game of Life](project-e05-game-of-life.md) | **โปรเจคถัดไป:** [Map Generator](project-e07-map-generator.md)
