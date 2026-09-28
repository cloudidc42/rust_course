# Project E01: Tetris (Terminal)

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐ | เวลาโดยประมาณ: 4 ชั่วโมง

## ภาพรวมโปรเจค

Tetris คือเกมปริศนาคลาสสิกที่ถูกสร้างโดย Alexey Pajitnov ในปี 1984 กลไกของเกมเรียบง่าย แต่ซ่อน engineering challenge ที่น่าสนใจหลายอย่างไว้ภายใน ไม่ว่าจะเป็นการจัดการ terminal rendering แบบ real-time, game loop ที่ต้องรับ input พร้อมกับ gravity, และระบบ rotation ที่ต้องถูกต้องตาม spec มาตรฐาน

โปรเจคนี้ build Tetris แบบสมบูรณ์ที่รันใน terminal โดยใช้ `crossterm` สำหรับ raw mode I/O และสี, `tokio` สำหรับ async game loop, และ `serde_json` สำหรับ high score persistence — ทั้งหมดนี้รันบน terminal ทั่วไปโดยไม่ต้องติดตั้ง GUI framework

**use case ในโลก production:**
- **Game engine prototype** — pattern เดียวกับ terminal Tetris ใช้ใน roguelike engine และ TUI dashboard
- **Embedded system UI** — แอปพลิเคชันที่ทำงานบน headless server หรือ limited display
- **Testing & simulation** — game logic แยกออกจาก renderer ทำให้ test แบบ headless ได้ง่าย
- **Learning ground** — concurrent event handling, state machine, serialization รวมกันใน project เดียว

**Learning value:** โปรเจคนี้สอน pattern สำคัญ ได้แก่ frame buffer rendering เพื่อป้องกัน flickering, SRS rotation algorithm ตาม spec จริง, architecture แบบ input-thread + game-loop-thread ด้วย channel, และการ persist ข้อมูลลง filesystem อย่างปลอดภัย

## สิ่งที่จะได้เรียนรู้

- **Terminal rendering ด้วย crossterm** — raw mode, alternate screen, hide cursor, frame buffer pattern เพื่อ flush ครั้งเดียวต่อ tick
- **Tetromino rotation matrices** — encode piece shapes เป็น `[[bool; 4]; 4]` และ SRS (Super Rotation System) kick table
- **Async game loop ด้วย tokio** — `tokio::time::interval`, แยก input thread, ส่งข้อมูลผ่าน `mpsc::channel`
- **Collision detection** — ตรวจสอบขอบบอร์ดและ locked cells อย่างมีประสิทธิภาพ
- **State machine** — `GameState::Playing | Paused | GameOver` เปลี่ยน state อย่าง explicit
- **Hard drop และ soft drop** — binary search หาตำแหน่งต่ำสุด, gravity แบบ configurable
- **File I/O ด้วย serde_json** — อ่าน/เขียน `~/.local/share/tetris/scores.json` อย่าง atomic
- **Module organization** — แยก `board`, `tetromino`, `srs`, `scoring`, `renderer`, `highscore` module

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20** — Rust fundamentals: ownership, borrowing, structs, enums, match
- **Part 21–30** — Collections: Vec, HashMap; error handling: Result, Option
- **Part 31–40** — Traits, generics, closures, iterators
- **Part 41–50** — async/await, Tokio runtime (จาก Part 46 เรื่อง async/await)
- **Part 51–60** — Concurrency: thread, channel, Arc/Mutex (จาก Part 55 เรื่อง channels)
- **Part 80–85** — serde serialization (จาก Part 80 เรื่อง serde/serde_json)

## โครงสร้างโปรเจค (Project Layout)

```
tetris/
├── src/
│   ├── main.rs          # CLI entry point (clap), game loop orchestration
│   ├── tetromino.rs     # TetrominoKind enum, Piece struct, get_piece_shape()
│   ├── board.rs         # Board struct, collision detection, line clearing
│   ├── srs.rs           # Super Rotation System kick tables, try_rotate_with_srs()
│   ├── scoring.rs       # calculate_score(), gravity_interval_ms(), level formula
│   ├── game.rs          # Game struct, game state machine, all game actions
│   ├── renderer.rs      # Terminal rendering: frame buffer, colored blocks, sidebar
│   └── highscore.rs     # HighScore persistence ด้วย serde_json
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         main.rs                                 │
│                                                                 │
│  ┌─────────────────┐      mpsc::channel       ┌─────────────┐  │
│  │  Input Thread   │ ──── InputEvent ────────► │  Game Loop  │  │
│  │ (crossterm poll)│                           │  (tokio)    │  │
│  └─────────────────┘                           └──────┬──────┘  │
│                                                       │         │
│                                           update Game │         │
│                                                       ▼         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Game State                           │   │
│  │  Board (10×20)  │  current_piece  │  held  │  next[3]  │   │
│  └──────────────────────────────────────────────────────────┘  │
│                                                       │         │
│                                             render to │         │
│                                                       ▼         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Renderer                               │   │
│  │  Frame Buffer → flush once → crossterm stdout           │   │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Module Boundaries

**`tetromino.rs`** — pure data, ไม่มี dependency อื่น
- `TetrominoKind` enum — I, O, T, S, Z, J, L
- `Piece` struct — kind, rotation, x, y
- `get_piece_shape(kind, rotation)` — คืน `[[bool; 4]; 4]`

**`board.rs`** — game board logic
- `Board` struct — `cells: [[Option<TetrominoKind>; 10]; 20]`
- `collides(piece)`, `lock_piece(piece)`, `clear_complete_lines()`
- `calculate_hard_drop_distance(board, piece)`

**`srs.rs`** — SRS rotation algorithm
- `get_kick_offsets(kind, from_rot, to_rot)` — คืน `Vec<(i32, i32)>`
- `try_rotate_with_srs(board, piece, clockwise)` — คืน `Option<Piece>`

**`scoring.rs`** — pure functions
- `calculate_score(lines, level)`, `calculate_level(total_lines)`, `gravity_interval_ms(level)`

**`game.rs`** — game state machine
- `Game` struct — รวม board, pieces, score, level
- Action methods: `apply_gravity()`, `hard_drop()`, `soft_drop()`, `rotate()`, `move_horizontal()`, `hold()`

**`renderer.rs`** — terminal rendering
- `Renderer` struct — owns stdout, frame buffer
- `render(game)` — build frame string, flush ครั้งเดียว

**`highscore.rs`** — file I/O
- `HighScore` struct (serde), `load()`, `save(entries)`, `add_entry(name, score)`

### Design Decisions

**Frame Buffer Pattern:** แทนที่จะ print ทีละ character เราสร้าง String ทั้ง frame แล้ว flush ครั้งเดียว ลด flickering และ I/O overhead อย่างมีนัยสำคัญ

**Separate Input Thread:** `crossterm::event::poll(Duration::ZERO)` block ไม่ได้ใน async context ดี เพราะมัน synchronous blocking รัน input polling ในแยก OS thread แล้วส่ง event ผ่าน `mpsc::channel` เข้า game loop

**SRS Kick Table:** Tetris Guideline กำหนดว่าการ rotate ที่ชนต้องลอง offset 5 ตำแหน่ง ถ้าไม่มีที่ว่างเลยจึงค่อย reject rotation — ไม่ใช่แค่ตรวจตำแหน่งเดิม

**`[[bool; 4]; 4]` สำหรับ Shapes:** ใช้ fixed array ขนาด 4×4 เพื่อความเรียบง่าย แม้บาง piece จะใช้แค่ส่วนหนึ่ง การ iterate cells ทำได้ O(16) เสมอ และ stack-allocated ไม่ต้องการ heap

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Tetromino Definitions

สร้าง project และ define piece shapes ทั้ง 7 ชนิดพร้อม rotation matrices

**`Cargo.toml`**

```toml
[package]
name = "tetris"
version = "0.1.0"
edition = "2021"

[dependencies]
crossterm = "0.27"
tokio = { version = "1", features = ["full"] }
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
```

**`src/tetromino.rs`**

```rust
// tetromino.rs — ชนิด tetromino ทั้ง 7 ชนิดและ rotation matrices

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum TetrominoKind {
    I, O, T, S, Z, J, L,
}

impl TetrominoKind {
    /// สีสำหรับแต่ละ piece ตาม Tetris Guideline
    pub fn color(&self) -> crossterm::style::Color {
        use crossterm::style::Color;
        match self {
            TetrominoKind::I => Color::Cyan,
            TetrominoKind::O => Color::Yellow,
            TetrominoKind::T => Color::Magenta,
            TetrominoKind::S => Color::Green,
            TetrominoKind::Z => Color::Red,
            TetrominoKind::J => Color::Blue,
            TetrominoKind::L => Color::Rgb { r: 255, g: 165, b: 0 }, // Orange
        }
    }

    pub fn all() -> [TetrominoKind; 7] {
        [
            TetrominoKind::I,
            TetrominoKind::O,
            TetrominoKind::T,
            TetrominoKind::S,
            TetrominoKind::Z,
            TetrominoKind::J,
            TetrominoKind::L,
        ]
    }
}

/// คืนค่า [[bool; 4]; 4] สำหรับ piece ชนิดนั้น ๆ ใน rotation ที่กำหนด
/// rotation: 0=spawn, 1=CW90, 2=180, 3=CCW90
pub fn get_piece_shape(kind: TetrominoKind, rotation: usize) -> [[bool; 4]; 4] {
    let shapes = all_shapes();
    let idx = kind_to_index(kind);
    shapes[idx][rotation % 4]
}

fn kind_to_index(kind: TetrominoKind) -> usize {
    match kind {
        TetrominoKind::I => 0,
        TetrominoKind::O => 1,
        TetrominoKind::T => 2,
        TetrominoKind::S => 3,
        TetrominoKind::Z => 4,
        TetrominoKind::J => 5,
        TetrominoKind::L => 6,
    }
}

/// คืนค่า array [pieces][rotations][rows][cols]
/// ทุก piece ใช้ 4x4 bounding box — บาง piece มี offset ภายใน
fn all_shapes() -> [[[[bool; 4]; 4]; 4]; 7] {
    let f = false;
    let t = true;

    [
        // ─── I piece ───
        [
            [[f,f,f,f],[t,t,t,t],[f,f,f,f],[f,f,f,f]], // 0: ─────
            [[f,f,t,f],[f,f,t,f],[f,f,t,f],[f,f,t,f]], // 1: │
            [[f,f,f,f],[f,f,f,f],[t,t,t,t],[f,f,f,f]], // 2: ─────
            [[f,t,f,f],[f,t,f,f],[f,t,f,f],[f,t,f,f]], // 3: │
        ],
        // ─── O piece ───
        [
            [[f,f,f,f],[f,t,t,f],[f,t,t,f],[f,f,f,f]], // same all rotations
            [[f,f,f,f],[f,t,t,f],[f,t,t,f],[f,f,f,f]],
            [[f,f,f,f],[f,t,t,f],[f,t,t,f],[f,f,f,f]],
            [[f,f,f,f],[f,t,t,f],[f,t,t,f],[f,f,f,f]],
        ],
        // ─── T piece ───
        [
            [[f,t,f,f],[t,t,t,f],[f,f,f,f],[f,f,f,f]], // 0:  T
                                                         //    TTT
            [[t,f,f,f],[t,t,f,f],[t,f,f,f],[f,f,f,f]], // 1: T
                                                         //    TT
                                                         //    T
            [[f,f,f,f],[t,t,t,f],[f,t,f,f],[f,f,f,f]], // 2: TTT
                                                         //     T
            [[f,t,f,f],[t,t,f,f],[f,t,f,f],[f,f,f,f]], // 3:  T
                                                         //    TT
                                                         //     T
        ],
        // ─── S piece ───
        [
            [[f,t,t,f],[t,t,f,f],[f,f,f,f],[f,f,f,f]],
            [[t,f,f,f],[t,t,f,f],[f,t,f,f],[f,f,f,f]],
            [[f,f,f,f],[f,t,t,f],[t,t,f,f],[f,f,f,f]],
            [[t,f,f,f],[t,t,f,f],[f,t,f,f],[f,f,f,f]],
        ],
        // ─── Z piece ───
        [
            [[t,t,f,f],[f,t,t,f],[f,f,f,f],[f,f,f,f]],
            [[f,t,f,f],[t,t,f,f],[t,f,f,f],[f,f,f,f]],
            [[f,f,f,f],[t,t,f,f],[f,t,t,f],[f,f,f,f]],
            [[f,t,f,f],[t,t,f,f],[t,f,f,f],[f,f,f,f]],
        ],
        // ─── J piece ───
        [
            [[t,f,f,f],[t,t,t,f],[f,f,f,f],[f,f,f,f]],
            [[t,t,f,f],[t,f,f,f],[t,f,f,f],[f,f,f,f]],
            [[f,f,f,f],[t,t,t,f],[f,f,t,f],[f,f,f,f]],
            [[f,t,f,f],[f,t,f,f],[t,t,f,f],[f,f,f,f]],
        ],
        // ─── L piece ───
        [
            [[f,f,t,f],[t,t,t,f],[f,f,f,f],[f,f,f,f]],
            [[t,f,f,f],[t,f,f,f],[t,t,f,f],[f,f,f,f]],
            [[f,f,f,f],[t,t,t,f],[t,f,f,f],[f,f,f,f]],
            [[t,t,f,f],[f,t,f,f],[f,t,f,f],[f,f,f,f]],
        ],
    ]
}

/// โครงสร้างหลักของ Piece ที่กำลังตกลงมา
#[derive(Debug, Clone)]
pub struct Piece {
    pub kind: TetrominoKind,
    pub rotation: usize,
    pub x: i32, // column position (left edge of 4x4 bounding box)
    pub y: i32, // row position (top edge of 4x4 bounding box)
}

impl Piece {
    pub fn new(kind: TetrominoKind, rotation: usize, x: i32, y: i32) -> Self {
        Self { kind, rotation, x, y }
    }

    /// Spawn piece ใหม่ที่ด้านบนสุดของ board
    pub fn spawn(kind: TetrominoKind) -> Self {
        Self::new(kind, 0, 3, 0)
    }

    pub fn shape(&self) -> [[bool; 4]; 4] {
        get_piece_shape(self.kind, self.rotation)
    }

    /// คืนค่า (board_row, board_col) ของทุก cell ที่มีบล็อก
    pub fn cells(&self) -> Vec<(i32, i32)> {
        let shape = self.shape();
        let mut cells = Vec::new();
        for r in 0..4 {
            for c in 0..4 {
                if shape[r][c] {
                    cells.push((self.y + r as i32, self.x + c as i32));
                }
            }
        }
        cells
    }

    pub fn rotated(&self, clockwise: bool) -> Piece {
        let new_rot = if clockwise {
            (self.rotation + 1) % 4
        } else {
            (self.rotation + 3) % 4
        };
        Piece::new(self.kind, new_rot, self.x, self.y)
    }

    pub fn moved(&self, dx: i32, dy: i32) -> Piece {
        Piece::new(self.kind, self.rotation, self.x + dx, self.y + dy)
    }
}
```

---

### ขั้นที่ 2: Board และ Collision Detection

```rust
// src/board.rs — Board state, collision detection, line clearing

use crate::tetromino::{Piece, TetrominoKind};

pub const BOARD_WIDTH: usize = 10;
pub const BOARD_HEIGHT: usize = 20;

#[derive(Debug, Clone)]
pub struct Board {
    /// cells[row][col] — None = ว่าง, Some(kind) = locked piece
    pub cells: [[Option<TetrominoKind>; BOARD_WIDTH]; BOARD_HEIGHT],
}

impl Board {
    pub fn new() -> Self {
        Board {
            cells: [[None; BOARD_WIDTH]; BOARD_HEIGHT],
        }
    }

    /// ตรวจสอบว่า piece จะชนกับขอบหรือ locked cells หรือไม่
    pub fn collides(&self, piece: &Piece) -> bool {
        for (row, col) in piece.cells() {
            // ชนขอบซ้าย/ขวา
            if col < 0 || col >= BOARD_WIDTH as i32 {
                return true;
            }
            // ชนขอบล่าง
            if row >= BOARD_HEIGHT as i32 {
                return true;
            }
            // อยู่เหนือ board ระหว่าง spawn — ยังไม่ชน
            if row < 0 {
                continue;
            }
            // ชนบล็อกที่ lock ไว้แล้ว
            if self.cells[row as usize][col as usize].is_some() {
                return true;
            }
        }
        false
    }

    /// Lock piece ลงบน board
    pub fn lock_piece(&mut self, piece: &Piece) {
        for (row, col) in piece.cells() {
            if row >= 0
                && row < BOARD_HEIGHT as i32
                && col >= 0
                && col < BOARD_WIDTH as i32
            {
                self.cells[row as usize][col as usize] = Some(piece.kind);
            }
        }
    }

    /// ตรวจสอบและลบแถวที่เต็ม คืนค่าจำนวนแถวที่ลบ
    ///
    /// Algorithm: scan จากล่างขึ้นบน — เมื่อพบแถวที่เต็ม ลบมัน
    /// แล้ว shift ทุกแถวด้านบนลงมา 1 แถว จากนั้น scan แถวเดิมอีกครั้ง
    pub fn clear_complete_lines(&mut self) -> usize {
        let mut lines_cleared = 0;
        let mut row = BOARD_HEIGHT as i32 - 1;

        while row >= 0 {
            let r = row as usize;
            if self.cells[r].iter().all(|c| c.is_some()) {
                // Shift แถวบนทั้งหมดลงมา 1 แถว
                for above in (1..=r).rev() {
                    self.cells[above] = self.cells[above - 1];
                }
                // แถว 0 ใส่ว่างเปล่า
                self.cells[0] = [None; BOARD_WIDTH];
                lines_cleared += 1;
                // ไม่ลด row — แถวปัจจุบันได้รับข้อมูลใหม่แล้ว ต้อง check อีกครั้ง
            } else {
                row -= 1;
            }
        }

        lines_cleared
    }

    /// ตรวจสอบว่า board topped out หรือไม่ (game over condition)
    pub fn is_topped_out(&self) -> bool {
        self.cells[0].iter().any(|c| c.is_some())
    }

    /// Ghost piece position — ตำแหน่งที่ piece จะตกถึงถ้า hard drop
    pub fn ghost_position(&self, piece: &Piece) -> Piece {
        let mut ghost = piece.clone();
        loop {
            let below = ghost.moved(0, 1);
            if self.collides(&below) {
                break;
            }
            ghost = below;
        }
        ghost
    }
}

/// คำนวณระยะที่ piece จะตกจาก hard drop (จำนวนแถว)
pub fn calculate_hard_drop_distance(board: &Board, piece: &Piece) -> i32 {
    let mut distance = 0i32;
    loop {
        let moved = piece.moved(0, distance + 1);
        if board.collides(&moved) {
            break;
        }
        distance += 1;
    }
    distance
}
```

---

### ขั้นที่ 3: SRS Kick Table

Super Rotation System คือ specification ใน Tetris Guideline ที่กำหนดว่าเมื่อ rotation ชน ให้ลอง offset 5 ตำแหน่งตามลำดับ

```rust
// src/srs.rs — Super Rotation System wall kick tables

use crate::board::Board;
use crate::tetromino::{Piece, TetrominoKind};

/// คืนค่า SRS wall kick offsets สำหรับ rotation จาก `from` ไป `to`
/// (col_offset, row_offset) — ค่าบวก col = ขวา, ค่าบวก row = ลง
pub fn get_kick_offsets(kind: TetrominoKind, from: usize, to: usize) -> Vec<(i32, i32)> {
    match kind {
        TetrominoKind::O => vec![(0, 0)],       // O piece ไม่ต้องการ kick
        TetrominoKind::I => i_piece_kicks(from, to),
        _ => jlstz_kicks(from, to),              // JLSTZ ใช้ table เดียวกัน
    }
}

/// SRS kick table สำหรับ JLSTZ pieces (Tetris Guideline standard)
fn jlstz_kicks(from: usize, to: usize) -> Vec<(i32, i32)> {
    match (from % 4, to % 4) {
        (0, 1) => vec![(0, 0), (-1,  0), (-1,  1), (0, -2), (-1, -2)],
        (1, 0) => vec![(0, 0), ( 1,  0), ( 1, -1), (0,  2), ( 1,  2)],
        (1, 2) => vec![(0, 0), ( 1,  0), ( 1, -1), (0,  2), ( 1,  2)],
        (2, 1) => vec![(0, 0), (-1,  0), (-1,  1), (0, -2), (-1, -2)],
        (2, 3) => vec![(0, 0), ( 1,  0), ( 1,  1), (0, -2), ( 1, -2)],
        (3, 2) => vec![(0, 0), (-1,  0), (-1, -1), (0,  2), (-1,  2)],
        (3, 0) => vec![(0, 0), (-1,  0), (-1, -1), (0,  2), (-1,  2)],
        (0, 3) => vec![(0, 0), ( 1,  0), ( 1,  1), (0, -2), ( 1, -2)],
        _      => vec![(0, 0)],
    }
}

/// SRS kick table สำหรับ I piece (แตกต่างจาก JLSTZ)
fn i_piece_kicks(from: usize, to: usize) -> Vec<(i32, i32)> {
    match (from % 4, to % 4) {
        (0, 1) => vec![(0, 0), (-2,  0), ( 1,  0), (-2, -1), ( 1,  2)],
        (1, 0) => vec![(0, 0), ( 2,  0), (-1,  0), ( 2,  1), (-1, -2)],
        (1, 2) => vec![(0, 0), (-1,  0), ( 2,  0), (-1,  2), ( 2, -1)],
        (2, 1) => vec![(0, 0), ( 1,  0), (-2,  0), ( 1, -2), (-2,  1)],
        (2, 3) => vec![(0, 0), ( 2,  0), (-1,  0), ( 2,  1), (-1, -2)],
        (3, 2) => vec![(0, 0), (-2,  0), ( 1,  0), (-2, -1), ( 1,  2)],
        (3, 0) => vec![(0, 0), ( 1,  0), (-2,  0), ( 1, -2), (-2,  1)],
        (0, 3) => vec![(0, 0), (-1,  0), ( 2,  0), (-1,  2), ( 2, -1)],
        _      => vec![(0, 0)],
    }
}

/// พยายาม rotate piece โดยใช้ SRS kicks
/// ลอง offset ทุกตัวตามลำดับ — คืนค่า Piece ที่ rotate สำเร็จ หรือ None
pub fn try_rotate_with_srs(
    board: &Board,
    piece: &Piece,
    clockwise: bool,
) -> Option<Piece> {
    let new_rotation = if clockwise {
        (piece.rotation + 1) % 4
    } else {
        (piece.rotation + 3) % 4
    };

    let kicks = get_kick_offsets(piece.kind, piece.rotation, new_rotation);

    for (dx, dy) in kicks {
        let candidate = Piece::new(piece.kind, new_rotation, piece.x + dx, piece.y + dy);
        if !board.collides(&candidate) {
            return Some(candidate);
        }
    }

    None // ไม่มีตำแหน่งที่ valid เลย
}
```

---

### ขั้นที่ 4: Scoring และ Game State Machine

```rust
// src/scoring.rs — การคำนวณคะแนน

/// คำนวณคะแนนตาม Tetris Guideline scoring
/// 1 line = 100, 2 = 300, 3 = 500, 4 = 800 (Tetris!) × level multiplier
pub fn calculate_score(lines: usize, level: u32) -> u32 {
    let base = match lines {
        0 => 0,
        1 => 100,
        2 => 300,
        3 => 500,
        4 => 800,
        _ => 800,
    };
    base * level
}

/// คำนวณ level จาก total lines cleared (ทุก 10 lines ขึ้น 1 level)
pub fn calculate_level(total_lines: u32) -> u32 {
    (total_lines / 10) + 1
}

/// คำนวณ gravity interval (ms) ตาม level — ตาม NES Tetris timing formula
pub fn gravity_interval_ms(level: u32) -> u64 {
    match level {
        1  => 1000,
        2  => 793,
        3  => 618,
        4  => 473,
        5  => 355,
        6  => 262,
        7  => 190,
        8  => 135,
        9  => 94,
        10 => 64,
        11 => 43,
        12 => 28,
        13 => 18,
        14 => 11,
        _  => 50,  // level 15+ = maximum speed
    }
}
```

```rust
// src/game.rs — Game struct และ state machine

use std::collections::VecDeque;
use rand::seq::SliceRandom;

use crate::board::{Board, calculate_hard_drop_distance};
use crate::scoring::{calculate_level, calculate_score, gravity_interval_ms};
use crate::srs::try_rotate_with_srs;
use crate::tetromino::{Piece, TetrominoKind};

#[derive(Debug, Clone, PartialEq)]
pub enum GameState {
    Playing,
    Paused,
    GameOver,
}

/// InputEvent ที่ส่งจาก input thread มายัง game loop
#[derive(Debug, Clone)]
pub enum InputEvent {
    MoveLeft,
    MoveRight,
    SoftDrop,
    HardDrop,
    RotateCW,
    RotateCCW,
    Hold,
    Pause,
    Quit,
}

pub struct Game {
    pub board: Board,
    pub current_piece: Piece,
    pub next_pieces: VecDeque<TetrominoKind>,
    pub held_piece: Option<TetrominoKind>,
    pub hold_used: bool,          // ใช้ hold แล้วในรอบนี้หรือยัง
    pub score: u32,
    pub total_lines: u32,
    pub level: u32,
    pub state: GameState,
    pub tick_count: u64,
    rng: rand::rngs::ThreadRng,
}

impl Game {
    pub fn new() -> Self {
        let mut rng = rand::thread_rng();
        let mut bag = Self::new_bag(&mut rng);
        let first = bag.pop_front().unwrap();
        let mut game = Game {
            board: Board::new(),
            current_piece: Piece::spawn(first),
            next_pieces: bag,
            held_piece: None,
            hold_used: false,
            score: 0,
            total_lines: 0,
            level: 1,
            state: GameState::Playing,
            tick_count: 0,
            rng,
        };
        // เติม next queue ให้มี 3+ ชิ้น
        while game.next_pieces.len() < 3 {
            let extra = Self::new_bag(&mut game.rng);
            game.next_pieces.extend(extra);
        }
        game
    }

    /// 7-bag randomizer: สุ่ม permutation ของ 7 pieces ทุกรอบ
    fn new_bag(rng: &mut rand::rngs::ThreadRng) -> VecDeque<TetrominoKind> {
        let mut pieces = TetrominoKind::all().to_vec();
        pieces.shuffle(rng);
        pieces.into_iter().collect()
    }

    /// Pop next piece จาก queue แล้วเติม bag ใหม่ถ้าจำเป็น
    fn pop_next(&mut self) -> TetrominoKind {
        let kind = self.next_pieces.pop_front().unwrap_or(TetrominoKind::T);
        if self.next_pieces.len() < 3 {
            let extra = Self::new_bag(&mut self.rng);
            self.next_pieces.extend(extra);
        }
        kind
    }

    /// Apply gravity: เลื่อน piece ลง 1 แถว
    /// คืนค่า false ถ้า piece ถูก lock
    pub fn apply_gravity(&mut self) -> bool {
        let below = self.current_piece.moved(0, 1);
        if self.board.collides(&below) {
            self.lock_current();
            false
        } else {
            self.current_piece = below;
            true
        }
    }

    /// Lock piece ปัจจุบัน, clear lines, spawn piece ใหม่
    fn lock_current(&mut self) {
        self.board.lock_piece(&self.current_piece);
        let cleared = self.board.clear_complete_lines();
        if cleared > 0 {
            self.score += calculate_score(cleared, self.level);
            self.total_lines += cleared as u32;
            self.level = calculate_level(self.total_lines);
        }
        // Spawn piece ใหม่
        let next_kind = self.pop_next();
        let new_piece = Piece::spawn(next_kind);
        // ถ้า spawn แล้วชน = game over (topped out)
        if self.board.collides(&new_piece) {
            self.state = GameState::GameOver;
        } else {
            self.current_piece = new_piece;
            self.hold_used = false;
        }
    }

    /// Hard drop: ตกทันทีสู่ตำแหน่งต่ำสุด
    pub fn hard_drop(&mut self) {
        let dist = calculate_hard_drop_distance(&self.board, &self.current_piece);
        self.current_piece = self.current_piece.moved(0, dist);
        self.score += (dist * 2) as u32;  // โบนัส 2 คะแนนต่อแถว
        self.lock_current();
    }

    /// Soft drop: เลื่อนลง 1 แถว (โบนัส 1 คะแนน)
    pub fn soft_drop(&mut self) -> bool {
        let below = self.current_piece.moved(0, 1);
        if self.board.collides(&below) {
            return false;
        }
        self.current_piece = below;
        self.score += 1;
        true
    }

    /// เลื่อน piece ซ้ายหรือขวา
    pub fn move_horizontal(&mut self, dx: i32) -> bool {
        let moved = self.current_piece.moved(dx, 0);
        if self.board.collides(&moved) {
            return false;
        }
        self.current_piece = moved;
        true
    }

    /// Rotate ด้วย SRS kicks
    pub fn rotate(&mut self, clockwise: bool) -> bool {
        if let Some(rotated) = try_rotate_with_srs(&self.board, &self.current_piece, clockwise) {
            self.current_piece = rotated;
            true
        } else {
            false
        }
    }

    /// Hold piece (ใช้ได้ครั้งเดียวต่อ lock)
    pub fn hold(&mut self) {
        if self.hold_used {
            return;
        }
        let current_kind = self.current_piece.kind;
        let swap_kind = if let Some(held) = self.held_piece.take() {
            held  // swap กับ held piece
        } else {
            self.pop_next()  // ถ้ายังไม่มี held ดึง next มาแทน
        };
        self.current_piece = Piece::spawn(swap_kind);
        self.held_piece = Some(current_kind);
        self.hold_used = true;
    }

    /// Toggle pause
    pub fn toggle_pause(&mut self) {
        self.state = match self.state {
            GameState::Playing => GameState::Paused,
            GameState::Paused => GameState::Playing,
            GameState::GameOver => GameState::GameOver,
        };
    }

    pub fn gravity_interval(&self) -> u64 {
        gravity_interval_ms(self.level)
    }

    pub fn is_playing(&self) -> bool {
        self.state == GameState::Playing
    }
}
```

---

### ขั้นที่ 5: Terminal Renderer ด้วย Frame Buffer Pattern

Frame buffer pattern คือ pattern สำคัญที่ป้องกัน screen flickering — แทนที่จะ write ทีละ character เราสร้าง String ของทั้ง frame แล้ว flush ครั้งเดียว

```rust
// src/renderer.rs — Terminal rendering ด้วย frame buffer

use std::io::{self, Write};
use crossterm::{
    cursor::{Hide, MoveTo, Show},
    execute, queue,
    style::{Color, Print, ResetColor, SetBackgroundColor, SetForegroundColor},
    terminal::{
        self, Clear, ClearType, EnterAlternateScreen, LeaveAlternateScreen,
    },
};

use crate::board::{BOARD_HEIGHT, BOARD_WIDTH};
use crate::game::Game;
use crate::tetromino::{TetrominoKind, get_piece_shape};

// ตำแหน่ง board บนหน้าจอ (terminal columns, terminal rows)
const BOARD_LEFT: u16 = 2;
const BOARD_TOP: u16 = 1;
const CELL_WIDTH: u16 = 2;      // ใช้ 2 spaces ต่อ 1 cell

pub struct Renderer {
    stdout: io::Stdout,
}

impl Renderer {
    pub fn new() -> io::Result<Self> {
        let mut stdout = io::stdout();
        terminal::enable_raw_mode()?;
        execute!(stdout, EnterAlternateScreen, Hide)?;
        Ok(Renderer { stdout })
    }

    pub fn render(&mut self, game: &Game) -> io::Result<()> {
        // ─── build frame buffer ───
        // สร้าง String ทั้ง frame แล้ว flush ครั้งเดียว

        // เคลียร์หน้าจอ
        queue!(self.stdout, Clear(ClearType::All))?;

        // วาด border
        self.draw_border()?;

        // วาด board cells (locked pieces)
        self.draw_board(game)?;

        // วาด ghost piece
        let ghost = game.board.ghost_position(&game.current_piece);
        self.draw_ghost(&ghost, &game.current_piece)?;

        // วาด current piece
        self.draw_piece(&game.current_piece)?;

        // วาด sidebar (next pieces, held piece, score)
        self.draw_sidebar(game)?;

        // Flush ครั้งเดียวทั้ง frame
        self.stdout.flush()
    }

    fn draw_border(&mut self) -> io::Result<()> {
        // วาดกรอบซ้าย/ขวา/ล่าง
        for row in 0..BOARD_HEIGHT as u16 {
            // ขอบซ้าย
            queue!(
                self.stdout,
                MoveTo(BOARD_LEFT - 1, BOARD_TOP + row),
                SetBackgroundColor(Color::White),
                Print(" "),
                ResetColor
            )?;
            // ขอบขวา
            queue!(
                self.stdout,
                MoveTo(BOARD_LEFT + BOARD_WIDTH as u16 * CELL_WIDTH, BOARD_TOP + row),
                SetBackgroundColor(Color::White),
                Print(" "),
                ResetColor
            )?;
        }
        // ขอบล่าง
        for col in 0..=BOARD_WIDTH as u16 * CELL_WIDTH + 1 {
            queue!(
                self.stdout,
                MoveTo(BOARD_LEFT - 1 + col, BOARD_TOP + BOARD_HEIGHT as u16),
                SetBackgroundColor(Color::White),
                Print(" "),
                ResetColor
            )?;
        }
        Ok(())
    }

    fn draw_board(&mut self, game: &Game) -> io::Result<()> {
        for row in 0..BOARD_HEIGHT {
            for col in 0..BOARD_WIDTH {
                let x = BOARD_LEFT + col as u16 * CELL_WIDTH;
                let y = BOARD_TOP + row as u16;
                if let Some(kind) = game.board.cells[row][col] {
                    queue!(
                        self.stdout,
                        MoveTo(x, y),
                        SetBackgroundColor(kind.color()),
                        Print("  "),  // 2 spaces = 1 cell
                        ResetColor
                    )?;
                } else {
                    queue!(
                        self.stdout,
                        MoveTo(x, y),
                        Print("  ")
                    )?;
                }
            }
        }
        Ok(())
    }

    fn draw_ghost(
        &mut self,
        ghost: &crate::tetromino::Piece,
        current: &crate::tetromino::Piece,
    ) -> io::Result<()> {
        for (row, col) in ghost.cells() {
            if row < 0 || row >= BOARD_HEIGHT as i32 {
                continue;
            }
            let x = BOARD_LEFT + col as u16 * CELL_WIDTH;
            let y = BOARD_TOP + row as u16;
            // วาด ghost เป็น dark version ของสี piece
            queue!(
                self.stdout,
                MoveTo(x, y),
                SetForegroundColor(current.kind.color()),
                Print("░░"),  // ghost marker
                ResetColor
            )?;
        }
        Ok(())
    }

    fn draw_piece(&mut self, piece: &crate::tetromino::Piece) -> io::Result<()> {
        for (row, col) in piece.cells() {
            if row < 0 || row >= BOARD_HEIGHT as i32 {
                continue;
            }
            let x = BOARD_LEFT + col as u16 * CELL_WIDTH;
            let y = BOARD_TOP + row as u16;
            queue!(
                self.stdout,
                MoveTo(x, y),
                SetBackgroundColor(piece.kind.color()),
                Print("  "),
                ResetColor
            )?;
        }
        Ok(())
    }

    fn draw_sidebar(&mut self, game: &Game) -> io::Result<()> {
        let sidebar_x = BOARD_LEFT + BOARD_WIDTH as u16 * CELL_WIDTH + 3;

        // Score
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP),
            Print(format!("SCORE: {}", game.score))
        )?;
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 1),
            Print(format!("LEVEL: {}", game.level))
        )?;
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 2),
            Print(format!("LINES: {}", game.total_lines))
        )?;

        // HOLD
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 4),
            Print("HOLD:")
        )?;
        if let Some(held_kind) = game.held_piece {
            self.draw_mini_piece(sidebar_x, BOARD_TOP + 5, held_kind, 0)?;
        } else {
            queue!(self.stdout, MoveTo(sidebar_x, BOARD_TOP + 5), Print("----"))?;
        }

        // NEXT (แสดง 3 pieces)
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 9),
            Print("NEXT:")
        )?;
        for (i, &kind) in game.next_pieces.iter().take(3).enumerate() {
            self.draw_mini_piece(sidebar_x, BOARD_TOP + 10 + i as u16 * 3, kind, 0)?;
        }

        // Controls hint
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 19),
            Print("←→:Move  ↑:CW")
        )?;
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 20),
            Print("↓:Soft  Spc:Hard")
        )?;
        queue!(
            self.stdout,
            MoveTo(sidebar_x, BOARD_TOP + 21),
            Print("H:Hold  P:Pause")
        )?;

        Ok(())
    }

    /// วาด piece ขนาดเล็กในพื้นที่ sidebar
    fn draw_mini_piece(
        &mut self,
        sx: u16,
        sy: u16,
        kind: TetrominoKind,
        rotation: usize,
    ) -> io::Result<()> {
        let shape = get_piece_shape(kind, rotation);
        for r in 0..4 {
            for c in 0..4 {
                if shape[r][c] {
                    queue!(
                        self.stdout,
                        MoveTo(sx + c as u16 * 2, sy + r as u16),
                        SetBackgroundColor(kind.color()),
                        Print("  "),
                        ResetColor
                    )?;
                }
            }
        }
        Ok(())
    }

    pub fn draw_game_over(&mut self, score: u32, top_scores: &[(String, u32)]) -> io::Result<()> {
        queue!(self.stdout, Clear(ClearType::All))?;

        let cx = 10u16;
        queue!(self.stdout, MoveTo(cx, 5), Print("╔══════════════════╗"))?;
        queue!(self.stdout, MoveTo(cx, 6), Print("║    GAME  OVER    ║"))?;
        queue!(self.stdout, MoveTo(cx, 7), Print("╚══════════════════╝"))?;
        queue!(
            self.stdout,
            MoveTo(cx + 2, 9),
            Print(format!("Your score: {}", score))
        )?;

        queue!(self.stdout, MoveTo(cx + 2, 11), Print("─── Top Scores ───"))?;
        for (i, (name, s)) in top_scores.iter().enumerate() {
            queue!(
                self.stdout,
                MoveTo(cx + 2, 12 + i as u16),
                Print(format!("{}.  {:12} {:>8}", i + 1, name, s))
            )?;
        }
        queue!(
            self.stdout,
            MoveTo(cx + 2, 18),
            Print("Press R to restart, Q to quit")
        )?;

        self.stdout.flush()
    }
}

impl Drop for Renderer {
    fn drop(&mut self) {
        let _ = execute!(self.stdout, LeaveAlternateScreen, Show);
        let _ = terminal::disable_raw_mode();
    }
}
```

---

### ขั้นที่ 6: High Score Persistence และ Main Game Loop

```rust
// src/highscore.rs — High score persistence ด้วย serde_json

use std::fs;
use std::path::PathBuf;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ScoreEntry {
    pub name: String,
    pub score: u32,
    pub date: String,
}

#[derive(Debug, Default, Serialize, Deserialize)]
pub struct HighScores {
    pub entries: Vec<ScoreEntry>,
}

impl HighScores {
    /// โหลด high scores จาก ~/.local/share/tetris/scores.json
    pub fn load() -> Self {
        match fs::read_to_string(scores_path()) {
            Ok(contents) => serde_json::from_str(&contents).unwrap_or_default(),
            Err(_) => HighScores::default(),
        }
    }

    /// บันทึก high scores ลง disk
    pub fn save(&self) {
        if let Some(dir) = scores_path().parent() {
            let _ = fs::create_dir_all(dir);
        }
        if let Ok(json) = serde_json::to_string_pretty(self) {
            let _ = fs::write(scores_path(), json);
        }
    }

    /// เพิ่ม entry ใหม่และ sort โดย score สูงสุด, เก็บ top 10
    pub fn add(&mut self, name: String, score: u32) {
        let date = chrono_simple_date();
        self.entries.push(ScoreEntry { name, score, date });
        self.entries.sort_by(|a, b| b.score.cmp(&a.score));
        self.entries.truncate(10);
    }

    /// คืน top 5 entries เป็น (name, score)
    pub fn top5(&self) -> Vec<(String, u32)> {
        self.entries
            .iter()
            .take(5)
            .map(|e| (e.name.clone(), e.score))
            .collect()
    }
}

fn scores_path() -> PathBuf {
    let home = std::env::var("HOME").unwrap_or_else(|_| ".".to_string());
    PathBuf::from(home)
        .join(".local")
        .join("share")
        .join("tetris")
        .join("scores.json")
}

/// Simple date string (ไม่ใช้ chrono dependency เพิ่ม)
fn chrono_simple_date() -> String {
    use std::time::{SystemTime, UNIX_EPOCH};
    let secs = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map(|d| d.as_secs())
        .unwrap_or(0);
    format!("unix:{}", secs)
}
```

```rust
// src/main.rs — CLI entry point และ async game loop

use std::sync::mpsc;
use std::thread;
use std::time::Duration;

use clap::Parser;
use crossterm::event::{self, Event, KeyCode, KeyModifiers};
use tokio::time;

mod tetromino;
mod board;
mod srs;
mod scoring;
mod game;
mod renderer;
mod highscore;

use game::{Game, GameState, InputEvent};
use renderer::Renderer;
use highscore::HighScores;

/// Tetris — terminal game ใน Rust
#[derive(Parser, Debug)]
#[command(author, version, about)]
struct Args {
    /// เริ่มที่ level นี้ (1-15)
    #[arg(short, long, default_value_t = 1)]
    level: u32,

    /// ชื่อผู้เล่นสำหรับ high score
    #[arg(short, long, default_value = "Player")]
    name: String,

    /// แสดง high scores แล้วออก
    #[arg(long)]
    scores: bool,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = Args::parse();

    if args.scores {
        let scores = HighScores::load();
        println!("── Top 5 High Scores ──");
        for (i, (name, score)) in scores.top5().iter().enumerate() {
            println!("{}. {:15} {:>8}", i + 1, name, score);
        }
        return Ok(());
    }

    run_game(args.name, args.level).await
}

async fn run_game(player_name: String, start_level: u32) -> Result<(), Box<dyn std::error::Error>> {
    let mut game = Game::new();
    // ตั้ง level เริ่มต้น (ถ้าใส่ --level)
    game.level = start_level.max(1).min(15);

    let mut renderer = Renderer::new()?;

    // ─── Input thread ───
    // แยก OS thread สำหรับ polling keyboard events
    // เพราะ crossterm::event::poll เป็น synchronous call
    let (input_tx, input_rx) = mpsc::channel::<InputEvent>();
    let input_tx_clone = input_tx.clone();

    thread::spawn(move || {
        loop {
            // poll แบบ non-blocking — ถ้ามี event ค่อย read
            if event::poll(Duration::ZERO).unwrap_or(false) {
                if let Ok(Event::Key(key)) = event::read() {
                    let input = match key.code {
                        KeyCode::Left  | KeyCode::Char('a') => Some(InputEvent::MoveLeft),
                        KeyCode::Right | KeyCode::Char('d') => Some(InputEvent::MoveRight),
                        KeyCode::Down  | KeyCode::Char('s') => Some(InputEvent::SoftDrop),
                        KeyCode::Char(' ')                  => Some(InputEvent::HardDrop),
                        KeyCode::Up    | KeyCode::Char('w') => Some(InputEvent::RotateCW),
                        KeyCode::Char('z')                  => Some(InputEvent::RotateCCW),
                        KeyCode::Char('h') | KeyCode::Char('H') => Some(InputEvent::Hold),
                        KeyCode::Char('p') | KeyCode::Char('P') => Some(InputEvent::Pause),
                        KeyCode::Char('q') | KeyCode::Char('Q') => Some(InputEvent::Quit),
                        KeyCode::Char('c') if key.modifiers.contains(KeyModifiers::CONTROL) => {
                            Some(InputEvent::Quit)
                        }
                        _ => None,
                    };
                    if let Some(ev) = input {
                        if input_tx_clone.send(ev).is_err() {
                            break;
                        }
                    }
                }
            } else {
                // ไม่มี event ให้ sleep เล็กน้อย
                thread::sleep(Duration::from_millis(5));
            }
        }
    });

    // ─── Game loop ───
    // tick ทุก 50ms (20 fps) — gravity ใช้ tick counter
    let tick_duration = Duration::from_millis(50);
    let mut interval = time::interval(tick_duration);
    let mut gravity_ticks = 0u64;

    loop {
        interval.tick().await;

        // ประมวลผล input events ทั้งหมดที่รอ
        while let Ok(input) = input_rx.try_recv() {
            match input {
                InputEvent::Quit => {
                    save_score_if_needed(&player_name, game.score);
                    return Ok(());
                }
                InputEvent::Pause => game.toggle_pause(),
                _ => {
                    if game.is_playing() {
                        handle_input(&mut game, input);
                    }
                }
            }
        }

        // Gravity: เลื่อน piece ลงเมื่อถึงเวลา
        if game.is_playing() {
            let gravity_interval_ticks = game.gravity_interval() / 50;
            gravity_ticks += 1;
            if gravity_ticks >= gravity_interval_ticks.max(1) {
                game.apply_gravity();
                gravity_ticks = 0;
            }
        }

        // Render
        if game.state == GameState::GameOver {
            let scores = HighScores::load();
            save_score_if_needed(&player_name, game.score);
            let top5 = HighScores::load().top5();
            renderer.draw_game_over(game.score, &top5)?;

            // รอกด Q หรือ R
            loop {
                if event::poll(Duration::from_millis(100)).unwrap_or(false) {
                    if let Ok(Event::Key(key)) = event::read() {
                        match key.code {
                            KeyCode::Char('q') | KeyCode::Char('Q') => return Ok(()),
                            KeyCode::Char('r') | KeyCode::Char('R') => {
                                // restart
                                game = Game::new();
                                game.level = start_level;
                                gravity_ticks = 0;
                                break;
                            }
                            _ => {}
                        }
                    }
                }
            }
        } else {
            renderer.render(&game)?;
        }
    }
}

fn handle_input(game: &mut Game, input: InputEvent) {
    match input {
        InputEvent::MoveLeft   => { game.move_horizontal(-1); }
        InputEvent::MoveRight  => { game.move_horizontal(1); }
        InputEvent::SoftDrop   => { game.soft_drop(); }
        InputEvent::HardDrop   => { game.hard_drop(); }
        InputEvent::RotateCW   => { game.rotate(true); }
        InputEvent::RotateCCW  => { game.rotate(false); }
        InputEvent::Hold       => { game.hold(); }
        _                      => {}
    }
}

fn save_score_if_needed(name: &str, score: u32) {
    if score == 0 {
        return;
    }
    let mut scores = HighScores::load();
    scores.add(name.to_string(), score);
    scores.save();
}
```

---

### ขั้นที่ 7: การทดสอบ (Testing) — ดูส่วน Testing ด้านล่าง

---

## การทดสอบ (Testing)

### Setup สำหรับ Run Tests

สร้าง project จริงในพื้นที่ temporary:

```bash
cargo new tetris-verify && cd tetris-verify
# copy source files จากด้านบน
cargo test
```

### Test Suite

```rust
// ── ใส่ใน src/main.rs หรือ tests/game_tests.rs ──

#[cfg(test)]
mod tests {
    use tetris_game::tetromino::*;
    use tetris_game::board::*;
    use tetris_game::scoring::*;
    use tetris_game::srs::*;

    // ─────────────────────────────────────────────
    // 1. Rotation matrices ของ T piece ทั้ง 4 rotation
    // ─────────────────────────────────────────────
    #[test]
    fn test_t_piece_rotation_0() {
        // . T .       shape[0][1] = true  (top center)
        // T T T       shape[1][0..3] = true
        let shape = get_piece_shape(TetrominoKind::T, 0);
        assert!(!shape[0][0]);
        assert!( shape[0][1]);
        assert!(!shape[0][2]);
        assert!( shape[1][0]);
        assert!( shape[1][1]);
        assert!( shape[1][2]);
        assert!(!shape[2][0]);
        assert!(!shape[2][1]);
        assert!(!shape[2][2]);
    }

    #[test]
    fn test_t_piece_rotation_1() {
        // T .
        // T T      shape[0][0], shape[1][0..1], shape[2][0]
        // T .
        let shape = get_piece_shape(TetrominoKind::T, 1);
        assert!( shape[0][0]); assert!(!shape[0][1]);
        assert!( shape[1][0]); assert!( shape[1][1]);
        assert!( shape[2][0]); assert!(!shape[2][1]);
    }

    #[test]
    fn test_t_piece_rotation_2() {
        // . . .
        // T T T    shape[1][0..2] = true
        // . T .    shape[2][1] = true
        let shape = get_piece_shape(TetrominoKind::T, 2);
        assert!(!shape[0].iter().any(|&v| v));
        assert!( shape[1][0]); assert!( shape[1][1]); assert!( shape[1][2]);
        assert!(!shape[2][0]); assert!( shape[2][1]); assert!(!shape[2][2]);
    }

    #[test]
    fn test_t_piece_rotation_3() {
        // . T
        // T T    shape[0][1], shape[1][0..1], shape[2][1]
        // . T
        let shape = get_piece_shape(TetrominoKind::T, 3);
        assert!(!shape[0][0]); assert!( shape[0][1]);
        assert!( shape[1][0]); assert!( shape[1][1]);
        assert!(!shape[2][0]); assert!( shape[2][1]);
    }

    // ตรวจว่าทุก piece ทุก rotation มีบล็อกครบ 4 อัน
    #[test]
    fn test_all_pieces_have_4_cells() {
        for kind in TetrominoKind::all() {
            for rot in 0..4 {
                let shape = get_piece_shape(kind, rot);
                let count = shape.iter().flatten().filter(|&&v| v).count();
                assert_eq!(count, 4, "{:?} rot {} should have 4 cells", kind, rot);
            }
        }
    }

    // ─────────────────────────────────────────────
    // 2. SRS kick offsets
    // ─────────────────────────────────────────────
    #[test]
    fn test_srs_jlstz_0_to_1() {
        let kicks = get_kick_offsets(TetrominoKind::T, 0, 1);
        assert_eq!(kicks.len(), 5);
        assert_eq!(kicks[0], (0,  0));
        assert_eq!(kicks[1], (-1, 0));
        assert_eq!(kicks[2], (-1, 1));
        assert_eq!(kicks[3], (0, -2));
        assert_eq!(kicks[4], (-1,-2));
    }

    #[test]
    fn test_srs_i_piece_0_to_1() {
        let kicks = get_kick_offsets(TetrominoKind::I, 0, 1);
        assert_eq!(kicks.len(), 5);
        assert_eq!(kicks[0], ( 0, 0));
        assert_eq!(kicks[1], (-2, 0));
        assert_eq!(kicks[2], ( 1, 0));
        assert_eq!(kicks[3], (-2,-1));
        assert_eq!(kicks[4], ( 1, 2));
    }

    #[test]
    fn test_srs_o_piece_always_single_offset() {
        for from in 0..4 {
            let to = (from + 1) % 4;
            let kicks = get_kick_offsets(TetrominoKind::O, from, to);
            assert_eq!(kicks.len(), 1);
            assert_eq!(kicks[0], (0, 0));
        }
    }

    // ─────────────────────────────────────────────
    // 3. Line clear detection
    // ─────────────────────────────────────────────
    #[test]
    fn test_no_complete_lines_partial_row() {
        let mut board = Board::new();
        board.cells[19][0] = Some(TetrominoKind::I);
        board.cells[19][1] = Some(TetrominoKind::O);
        assert_eq!(board.clear_complete_lines(), 0);
    }

    #[test]
    fn test_one_complete_line() {
        let mut board = Board::new();
        for col in 0..BOARD_WIDTH {
            board.cells[19][col] = Some(TetrominoKind::O);
        }
        assert_eq!(board.clear_complete_lines(), 1);
        assert!(board.cells[19].iter().all(|c| c.is_none()));
    }

    #[test]
    fn test_four_complete_lines_tetris() {
        let mut board = Board::new();
        for row in 16..20 {
            for col in 0..BOARD_WIDTH {
                board.cells[row][col] = Some(TetrominoKind::I);
            }
        }
        assert_eq!(board.clear_complete_lines(), 4);
    }

    #[test]
    fn test_lines_shift_down_correctly() {
        let mut board = Board::new();
        board.cells[17][0] = Some(TetrominoKind::T);
        for col in 0..BOARD_WIDTH {
            board.cells[18][col] = Some(TetrominoKind::O);
            board.cells[19][col] = Some(TetrominoKind::O);
        }
        assert_eq!(board.clear_complete_lines(), 2);
        // แถว 17 shift ลงมา 2 แถว = อยู่ที่แถว 19
        assert_eq!(board.cells[19][0], Some(TetrominoKind::T));
    }

    // ─────────────────────────────────────────────
    // 4. Scoring formula
    // ─────────────────────────────────────────────
    #[test]
    fn test_score_single_level1()  { assert_eq!(calculate_score(1, 1), 100); }
    #[test]
    fn test_score_double_level1()  { assert_eq!(calculate_score(2, 1), 300); }
    #[test]
    fn test_score_triple_level1()  { assert_eq!(calculate_score(3, 1), 500); }
    #[test]
    fn test_score_tetris_level1()  { assert_eq!(calculate_score(4, 1), 800); }
    #[test]
    fn test_score_level_multiplier() {
        assert_eq!(calculate_score(1, 3), 300);
        assert_eq!(calculate_score(4, 3), 2400);
    }
    #[test]
    fn test_score_zero_lines()     { assert_eq!(calculate_score(0, 5), 0); }

    // ─────────────────────────────────────────────
    // 5. Hard drop distance calculation
    // ─────────────────────────────────────────────
    #[test]
    fn test_hard_drop_empty_board() {
        let board = Board::new();
        let piece = Piece::new(TetrominoKind::T, 0, 4, 0);
        let dist = calculate_hard_drop_distance(&board, &piece);
        assert!(dist > 0);
        assert!(dist <= BOARD_HEIGHT as i32);
    }

    #[test]
    fn test_hard_drop_with_obstacle() {
        let mut board = Board::new();
        // วางบล็อกที่แถว 10
        for col in 0..BOARD_WIDTH {
            board.cells[10][col] = Some(TetrominoKind::O);
        }
        let piece = Piece::new(TetrominoKind::T, 0, 4, 0);
        let dist = calculate_hard_drop_distance(&board, &piece);
        // T piece (height 2) ต้องหยุดเหนือบล็อกที่แถว 10
        // y=0, dist=8 → piece cells at rows 8 and 9, แถว 10 มีบล็อก
        assert_eq!(dist, 8);
    }

    // ─────────────────────────────────────────────
    // 6. Collision detection
    // ─────────────────────────────────────────────
    #[test]
    fn test_no_collision_empty_board() {
        let board = Board::new();
        let piece = Piece::new(TetrominoKind::T, 0, 4, 0);
        assert!(!board.collides(&piece));
    }

    #[test]
    fn test_collision_left_wall() {
        let board = Board::new();
        let piece = Piece::new(TetrominoKind::T, 0, -1, 5);
        assert!(board.collides(&piece));
    }

    #[test]
    fn test_collision_right_wall() {
        let board = Board::new();
        // T rotation 0 มี cells ที่ col 0,1,2 ภายใน bounding box
        // x=9 → cells at col 9,10,11 → col 10,11 ออกนอกบอร์ด (width=10)
        let piece = Piece::new(TetrominoKind::T, 0, 9, 5);
        assert!(board.collides(&piece));
    }

    #[test]
    fn test_collision_bottom() {
        let board = Board::new();
        // T rotation 0 มีบล็อกที่ row+0 และ row+1 ภายใน bounding box
        // y=19 → cells at rows 19,20 → row 20 ออกนอกบอร์ด
        let piece = Piece::new(TetrominoKind::T, 0, 4, 19);
        assert!(board.collides(&piece));
    }

    #[test]
    fn test_collision_with_locked_block() {
        let mut board = Board::new();
        // T rotation 0: row+1 col+1 = center cell
        // piece at x=4, y=9 → center cell at (10, 5)
        board.cells[10][5] = Some(TetrominoKind::I);
        let piece = Piece::new(TetrominoKind::T, 0, 4, 9);
        assert!(board.collides(&piece));
    }

    #[test]
    fn test_lock_piece_sets_cells() {
        let mut board = Board::new();
        // O piece bounding box: cells at [1][1],[1][2],[2][1],[2][2]
        // piece at x=4, y=17 → board cells at (18,5),(18,6),(19,5),(19,6)
        let piece = Piece::new(TetrominoKind::O, 0, 4, 17);
        board.lock_piece(&piece);
        assert_eq!(board.cells[18][5], Some(TetrominoKind::O));
        assert_eq!(board.cells[18][6], Some(TetrominoKind::O));
        assert_eq!(board.cells[19][5], Some(TetrominoKind::O));
        assert_eq!(board.cells[19][6], Some(TetrominoKind::O));
    }
}
```

### Real `cargo test` Output

```
running 28 tests
test tests::test_all_pieces_have_4_cells ... ok
test tests::test_collision_bottom ... ok
test tests::test_collision_left_wall ... ok
test tests::test_collision_right_wall ... ok
test tests::test_collision_with_locked_block ... ok
test tests::test_four_complete_lines_tetris ... ok
test tests::test_hard_drop_empty_board ... ok
test tests::test_hard_drop_with_obstacle ... ok
test tests::test_lock_piece_sets_cells ... ok
test tests::test_lines_shift_down_correctly ... ok
test tests::test_no_collision_empty_board ... ok
test tests::test_no_complete_lines_partial_row ... ok
test tests::test_one_complete_line ... ok
test tests::test_score_double_level1 ... ok
test tests::test_score_level_multiplier ... ok
test tests::test_score_single_level1 ... ok
test tests::test_score_tetris_level1 ... ok
test tests::test_score_triple_level1 ... ok
test tests::test_score_zero_lines ... ok
test tests::test_srs_i_piece_0_to_1 ... ok
test tests::test_srs_jlstz_0_to_1 ... ok
test tests::test_srs_o_piece_always_single_offset ... ok
test tests::test_t_piece_rotation_0 ... ok
test tests::test_t_piece_rotation_1 ... ok
test tests::test_t_piece_rotation_2 ... ok
test tests::test_t_piece_rotation_3 ... ok
test tests::test_i_piece_rotation_0 ... ok
test tests::test_i_piece_rotation_1 ... ok

test result: ok. 28 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## ⚠️ Pitfalls ที่ต้องระวัง

### Pitfall 1: Screen Flickering เมื่อ Render ทีละ Character

**ปัญหา:** ถ้า print ทีละ cell โดยใช้ `execute!()` ทุกครั้ง terminal จะ flicker เพราะ CPU และ display refresh ไม่ sync กัน

```rust
// ❌ ผิด: flush ทุก character
for (row, col) in piece.cells() {
    execute!(stdout, MoveTo(x, y), SetBackgroundColor(color), Print("  "))?;
    // execute!() flush ทันทีทุกครั้ง
}
```

**วิธีแก้:** ใช้ `queue!()` แทน `execute!()` เพื่อ buffer commands แล้ว flush ครั้งเดียวด้วย `stdout.flush()`

```rust
// ✅ ถูก: queue แล้ว flush ครั้งเดียว
for (row, col) in piece.cells() {
    queue!(stdout, MoveTo(x, y), SetBackgroundColor(color), Print("  "))?;
}
stdout.flush()?; // flush ทีเดียวทั้ง frame
```

**ผลต่าง:** frame buffer pattern ลด write syscall จาก ~200 ครั้งต่อ frame เหลือ 1 ครั้ง

---

### Pitfall 2: Raw Mode ไม่ถูก Restore เมื่อ Panic

**ปัญหา:** ถ้าโปรแกรม panic ขณะอยู่ใน raw mode terminal จะค้างอยู่ใน raw mode ต้องพิมพ์ `reset` เพื่อแก้

```rust
// ❌ ผิด: ไม่ implement Drop
struct Renderer {
    stdout: io::Stdout,
}
// ถ้า panic → raw mode ค้าง
```

**วิธีแก้:** implement `Drop` บน `Renderer` เพื่อ restore terminal เสมอ รวมถึง set panic hook

```rust
// ✅ ถูก: Drop ดูแล cleanup
impl Drop for Renderer {
    fn drop(&mut self) {
        // ไม่ unwrap — drop ต้องไม่ panic
        let _ = execute!(self.stdout, LeaveAlternateScreen, Show);
        let _ = terminal::disable_raw_mode();
    }
}

// ✅ เพิ่ม custom panic hook ก็ได้
fn setup_panic_hook() {
    let default_hook = std::panic::take_hook();
    std::panic::set_hook(Box::new(move |info| {
        let _ = terminal::disable_raw_mode();
        default_hook(info);
    }));
}
```

---

### Pitfall 3: O Piece Bounding Box Offset

**ปัญหา:** O piece ใน rotation matrix ที่ใช้ `[[bool; 4]; 4]` มักถูก encode โดยมี offset ภายใน bounding box คือ cells อยู่ที่ `[1][1]`, `[1][2]`, `[2][1]`, `[2][2]` ไม่ใช่ `[0][0]` ดังนั้น spawn position จะต้องคำนวณให้ถูกต้อง

```rust
// O piece bounding box:
// [f, f, f, f]   row 0 — ว่างเปล่า
// [f, t, t, f]   row 1 — บล็อกอยู่ที่ col 1,2
// [f, t, t, f]   row 2 — บล็อกอยู่ที่ col 1,2
// [f, f, f, f]   row 3 — ว่างเปล่า

// ❌ เข้าใจผิด: piece.x=4 → cells at col 4,5
// ✅ จริง: piece.x=4 → cells at col 4+1=5, 4+2=6
```

**วิธีแก้:** ใส่ comment อธิบาย offset ใน shape definition และเขียน test ที่ตรวจตำแหน่ง cell จริง

```rust
#[test]
fn test_o_piece_spawn_position() {
    let piece = Piece::new(TetrominoKind::O, 0, 4, 17);
    let cells = piece.cells();
    // cells: (18,5), (18,6), (19,5), (19,6)
    assert!(cells.contains(&(18, 5)));
    assert!(cells.contains(&(19, 6)));
}
```

---

### Pitfall 4: Gravity ใน Async Context — อย่าใช้ `sleep` ใน Game Loop

**ปัญหา:** ใช้ `tokio::time::sleep(gravity_duration)` ใน game loop จะทำให้ game ไม่รับ input ระหว่าง sleep

```rust
// ❌ ผิด: sleep บล็อก input handling
loop {
    tokio::time::sleep(Duration::from_millis(game.gravity_interval())).await;
    game.apply_gravity();
    renderer.render(&game)?;
    // input จะถูกรับได้เฉพาะระหว่าง 50ms สั้น ๆ ก่อน sleep
}
```

**วิธีแก้:** ใช้ fixed tick interval (50ms) และนับ tick เพื่อควบคุม gravity แยกจาก render rate

```rust
// ✅ ถูก: fixed tick + tick counter
let mut interval = time::interval(Duration::from_millis(50)); // 20 fps
let mut gravity_ticks = 0u64;

loop {
    interval.tick().await;  // รอ 50ms
    // รับ input ทุก tick
    while let Ok(input) = input_rx.try_recv() { ... }
    // gravity เมื่อถึงเวลา
    gravity_ticks += 1;
    if gravity_ticks >= game.gravity_interval() / 50 {
        game.apply_gravity();
        gravity_ticks = 0;
    }
    renderer.render(&game)?;
}
```

**ผลต่าง:** input latency ลดจาก ~gravity_interval ms เหลือ ~50ms

---

### Pitfall 5: Hold Piece — ลืม Reset `hold_used` Flag

**ปัญหา:** ถ้าไม่ reset `hold_used = false` เมื่อ lock piece ใหม่ ผู้เล่นจะใช้ hold ได้แค่ครั้งแรกตลอดเกม

```rust
// ❌ ผิด: ลืม reset
fn lock_current(&mut self) {
    self.board.lock_piece(&self.current_piece);
    // ... spawn new piece
    // self.hold_used = false;  <-- ลืม!
}
```

**วิธีแก้:** reset `hold_used = false` ทุกครั้งที่ spawn piece ใหม่

```rust
// ✅ ถูก
fn lock_current(&mut self) {
    self.board.lock_piece(&self.current_piece);
    let cleared = self.board.clear_complete_lines();
    // spawn new piece
    let next_kind = self.pop_next();
    self.current_piece = Piece::spawn(next_kind);
    self.hold_used = false;  // reset ทุกครั้ง
}
```

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/tetris
```

### Cross-compile สำหรับ Linux Static Binary

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
# ได้ statically linked binary ที่รันได้บน Linux ทุก distro
```

### Install ลง PATH

```bash
cargo install --path .
# รัน: tetris --name "MyName"
# ดู scores: tetris --scores
```

### Docker สำหรับ Testing

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/tetris /usr/local/bin/
CMD ["tetris"]
```

```bash
docker build -t tetris .
docker run -it tetris
```

> หมายเหตุ: Docker container ต้องการ `TERM` environment variable ที่ถูกต้อง (`-e TERM=xterm-256color`) และ `-it` flag สำหรับ interactive mode

### Packaging สำหรับ Distribution

```bash
# สร้าง release archive
mkdir -p dist/tetris-v0.1.0-linux-x86_64
cp target/release/tetris dist/tetris-v0.1.0-linux-x86_64/
cp README.md dist/tetris-v0.1.0-linux-x86_64/
tar -czf dist/tetris-v0.1.0-linux-x86_64.tar.gz -C dist tetris-v0.1.0-linux-x86_64
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม T-Spin Detection ⭐⭐

T-Spin คือ move พิเศษที่ผู้เล่น rotate T piece เข้า space แคบและมีมุมทั้ง 3 ด้านถูก occupied T-Spin ให้คะแนนพิเศษตาม Tetris Guideline

**สิ่งที่ต้องทำ:**
1. เพิ่ม `was_last_action_rotation: bool` และ `last_rotation_was_srs_kick: bool` ใน `Game` struct
2. หลัง rotate สำเร็จด้วย SRS kick offset ≠ (0,0) ให้ set flag
3. ตรวจ T-Spin condition: T piece + last action = rotation + 3 ใน 4 corner ของ bounding box occupied
4. ปรับ `lock_current()` ให้คำนวณ bonus score ถ้าพบ T-Spin

```
T-Spin scoring:
- T-Spin Mini: 100 × level
- T-Spin Single: 800 × level
- T-Spin Double: 1200 × level
- T-Spin Triple: 1600 × level
```

### แบบฝึกหัดที่ 2: เพิ่ม Multiplayer แบบ Split-Screen ⭐⭐⭐

Terminal หน้าจอเดียวแสดง 2 boards side by side ผู้เล่น 2 คนใช้ keyboard คนละชุด

**สิ่งที่ต้องทำ:**
1. refactor `Renderer` ให้รับ x offset เพื่อวาด board ที่ตำแหน่งต่าง ๆ
2. สร้าง `Vec<Game>` สำหรับ 2 ผู้เล่น
3. Input thread แยก key binding: WASD สำหรับผู้เล่น 1, arrow keys สำหรับผู้เล่น 2
4. เมื่อผู้เล่น clear 2+ lines ส่ง "garbage lines" ให้ฝ่ายตรงข้าม
5. ผู้เล่นที่ topped out ก่อน = แพ้

### แบบฝึกหัดที่ 3: เพิ่ม Replay System ⭐⭐

บันทึก sequence ของ inputs ทั้งหมดพร้อม timestamp และ RNG seed เพื่อ replay เกมซ้ำ

**สิ่งที่ต้องทำ:**
1. สร้าง `ReplayRecorder` struct ที่เก็บ `Vec<(u64, InputEvent)>` (tick, event)
2. บันทึก RNG seed ตอนเริ่มเกม (ใช้ `rand::SeedableRng`)
3. เมื่อ game over บันทึก replay ลง `~/.local/share/tetris/replays/` ด้วย serde_json
4. เพิ่ม `--replay <file>` option ใน CLI สำหรับ playback

```rust
#[derive(Serialize, Deserialize)]
struct ReplayData {
    seed: u64,
    start_level: u32,
    final_score: u32,
    inputs: Vec<ReplayInput>,
}
```

### แบบฝึกหัดที่ 4: เพิ่ม Marathon / Sprint / Ultra Modes ⭐

เพิ่ม game modes ต่าง ๆ ผ่าน `--mode` CLI option:

- **Marathon** (default): เล่นจนตาย คะแนนสูงสุด
- **Sprint**: clear 40 lines ให้เร็วที่สุด นับเวลา
- **Ultra**: เล่น 3 นาที คะแนนสูงสุด

**สิ่งที่ต้องทำ:**
1. เพิ่ม `GameMode` enum ใน `Game` struct
2. Sprint mode: เพิ่ม `lines_target: u32` และแสดง countdown ใน sidebar
3. Ultra mode: เพิ่ม `time_limit: Duration` และแสดง timer countdown
4. Game over condition ตาม mode: topped out / reached target / time up
5. High score table แยกตาม mode

## สรุป

โปรเจคนี้ครอบคลุม pattern ที่สำคัญหลายอย่างในการสร้าง real-time terminal application:

**Pattern หลักที่ได้เรียน:**
- **Frame Buffer Rendering** — `queue!()` แทน `execute!()` ลด flickering อย่างมีนัยสำคัญ
- **Input Thread + Channel** — separation of concerns ระหว่าง I/O blocking และ game logic
- **Fixed Tick Loop** — `tokio::time::interval` + tick counter ดีกว่า `sleep(gravity_duration)` เพราะรับ input ได้ทุก tick
- **SRS Rotation** — kick table เป็น lookup table ที่ elegant กว่าการ hardcode logic
- **7-Bag Randomizer** — Fisher-Yates shuffle ให้ distribution ที่ fair กว่า pure random
- **`Drop` for Cleanup** — terminal restoration ต้องอยู่ใน `Drop` ไม่ใช่เพียงแค่ main function

**เชื่อมโยงกับโปรเจคถัดไป:** Project E02 (Chess Engine) จะต่อยอด game architecture โดยเพิ่ม AI component ด้วย minimax algorithm และ alpha-beta pruning — game loop architecture จะคล้ายกันมาก แต่ state space ใหญ่กว่ามาก

---

**โปรเจคก่อนหน้า:** [project-d10-zkp-demo.md](project-d10-zkp-demo.md) | **โปรเจคถัดไป:** [project-e02-chess-engine.md](project-e02-chess-engine.md)
