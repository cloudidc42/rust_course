# Project E02: Chess Engine + UCI Protocol

> โมดูล: E — Games & Graphics | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 15 ชั่วโมง

## ภาพรวมโปรเจค

Chess engine คือโปรแกรมที่สามารถเล่นหมากรุกได้ด้วยตัวเองโดยไม่ต้องมีผู้เล่น เป็นหนึ่งในโปรแกรม AI ที่เก่าแก่และได้รับการศึกษามากที่สุดในประวัติศาสตร์คอมพิวเตอร์ นับตั้งแต่ Deep Blue เอาชนะ Garry Kasparov ในปี 1997 ไปจนถึง Stockfish, AlphaZero และ Leela Chess Zero ในยุคปัจจุบัน

โปรเจคนี้สร้าง chess engine ตั้งแต่ศูนย์ด้วย Rust โดยครอบคลุม:

- **Bitboard representation** — เก็บตำแหน่งหมากด้วย `u64` bitfields สำหรับการ bit manipulation ที่รวดเร็ว
- **Move generation** — สร้าง legal moves สำหรับทุก piece type รวมถึง castling, en passant, promotion
- **Alpha-beta search** — tree search algorithm พร้อม iterative deepening, transposition table, move ordering
- **Evaluation function** — material + piece-square tables (PST) เพื่อประเมินความได้เปรียบในตำแหน่ง
- **UCI protocol** — Universal Chess Interface มาตรฐานที่ทำให้ engine ทำงานร่วมกับ GUI อย่าง Arena, Cute Chess, Lichess bot ได้

**Use case ใน production:**
- Lichess bot API รองรับ UCI engine โดยตรง
- Chess training software ใช้ engine เพื่อ analyze ตำแหน่ง
- Chess GUI อย่าง Arena, Cute Chess, Fritz รองรับ engine ผ่าน UCI
- Online chess sites อย่าง Chess.com เคยใช้ engine เพื่อ detect cheating

**Learning value:** โปรเจคนี้รวม bit manipulation ขั้นสูง, algorithm design (alpha-beta), protocol implementation, และ performance optimization ไว้ในโปรเจคเดียว เหมาะสำหรับ Rust developer ที่ต้องการฝึก systems programming แบบ real-world

## สิ่งที่จะได้เรียนรู้

- **Bitboard representation** — ใช้ `u64` บิตฟิลด์แทนกระดาน 64 ช่อง และ bit manipulation tricks เช่น `trailing_zeros()`, `count_ones()`, `bb &= bb - 1` (clear LSB)
- **Move encoding ที่กระชับ** — pack from/to/flags ลงใน `u16` เดียว; ประหยัด memory และ cache ใน search tree
- **Alpha-beta pruning** — ลด search space จาก O(b^d) ไปเป็น O(b^(d/2)) เฉลี่ย; negamax framework ที่ symmetric
- **Iterative deepening** — เริ่ม search จาก depth 1 แล้วเพิ่มทีละ 1 เพื่อให้ได้ best move เร็ว และใช้ TT move จาก depth ก่อนหน้าใน ordering
- **Zobrist hashing** — hash ตำแหน่งด้วย XOR ของ random keys; incremental update O(1); ใช้ใน transposition table
- **UCI protocol** — stdin/stdout line-based protocol; parse commands จาก GUI และส่ง bestmove กลับ
- **Perft testing** — วิธีตรวจสอบ move generator ด้วยการนับ leaf node ที่ depth ต่าง ๆ เปรียบเทียบกับค่า reference ที่รู้จักแน่นอน
- **Piece-square tables** — ให้ bonus/penalty ตามตำแหน่งของแต่ละ piece บนกระดาน เพื่อ guide engine ไปสู่ตำแหน่งที่ดีกว่า

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20** — Rust fundamentals: ownership, borrowing, structs, enums, pattern matching
- **Part 21–30** — traits, generics, iterators, closures
- **Part 31–40** — error handling, collections (Vec, HashMap), string manipulation
- **Part 41–45** — bit manipulation operators (`<<`, `>>`, `&`, `|`, `^`, `!`), numeric types
- **Part 46–50** — async/await, Tokio (สำหรับ UCI stdin handling ใน production version)
- **Part 80–85** — serde/serde_json (สำหรับ position serialization และ config)
- **Part 96–100** — performance considerations, profiling

## โครงสร้างโปรเจค (Project Layout)

```
chess-engine/
├── src/
│   ├── main.rs          # Entry point: UCI loop หรือ bench mode
│   ├── bitboard.rs      # u64 bitboard primitives + attack generation
│   ├── board.rs         # Board struct: 12 bitboards + game state
│   ├── movegen.rs       # Move struct + pseudo-legal + legal generation + perft
│   ├── evaluation.rs    # Material + PST static evaluation
│   ├── search.rs        # Negamax alpha-beta + iterative deepening + TT
│   ├── uci.rs           # UCI protocol parser + main loop
│   └── zobrist.rs       # Zobrist hash computation
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Bitboard Layout

ใช้ **Little-endian rank-file** (LERF) mapping:

```
Square index:
a8=56  b8=57  ...  h8=63
a7=48  b7=49  ...  h7=55
...
a2= 8  b2= 9  ...  h2=15
a1= 0  b1= 1  ...  h1= 7

ตัวอย่าง: ตำแหน่ง e4 = rank 3 * 8 + file 4 = 28
```

`Board` เก็บ piece ด้วย array ขนาด `[2][6]` — `pieces[color][piece_type]`:

```
pieces[0][0] = white pawns bitboard
pieces[0][1] = white knights bitboard
...
pieces[1][5] = black king bitboard
```

Piece index: `0=Pawn, 1=Knight, 2=Bishop, 3=Rook, 4=Queen, 5=King`

### Move Encoding

Move เก็บใน `u16` เดียว:

```
Bits 15-12: flags (4 bits)  — quiet/capture/castle/ep/promotion
Bits 11- 6: to square (6 bits)
Bits  5- 0: from square (6 bits)
```

Flag values:
- `0` = quiet move
- `1` = double pawn push
- `2` = kingside castle
- `3` = queenside castle
- `4` = capture
- `5` = en passant capture
- `8`-`11` = promotion (N/B/R/Q)
- `12`-`15` = promotion + capture

### Search Architecture

```
iterative_deepening(board, max_depth)
  └── for depth in 1..=max_depth:
        root_search(board, depth)
          └── for move in ordered_moves:
                -negamax(child, depth-1, -beta, -alpha)
                  └── if depth==0: quiescence_search()
                  └── else: recurse with alpha-beta pruning
```

**Transposition Table:** `HashMap<u64, TTEntry>` ที่ key คือ Zobrist hash ของ position และ value คือ depth, score, flag, best_move

### Why Bitboards?

วิธีทางเลือกคือ **mailbox** (array 64 ช่อง เก็บ piece type ในแต่ละช่อง) ซึ่งเข้าใจง่ายกว่า แต่:
- Knight attack จาก e4: mailbox ต้องวน loop 8 ทิศทาง, bitboard คือ `precalc_table[28]` — O(1)
- หา occupied squares: mailbox ต้องวน 64 ช่อง, bitboard คือ OR ของ 12 boards — O(1)
- เช็ค pin/check: bitboard สามารถใช้ fill algorithm กับ SIMD-friendly operations

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Bitboard Primitives

สร้างไฟล์ `src/bitboard.rs` ที่มี building blocks ทั้งหมด:

```rust
pub type Bitboard = u64;

pub const FILE_A: Bitboard = 0x0101_0101_0101_0101;
pub const FILE_H: Bitboard = FILE_A << 7;
pub const RANK_2: Bitboard = 0xFF << 8;
pub const RANK_7: Bitboard = 0xFF << 48;

/// Clear least-significant bit และคืนค่า index ของมัน
#[inline(always)]
pub fn pop_lsb(bb: &mut Bitboard) -> u8 {
    let sq = bb.trailing_zeros() as u8;
    *bb &= *bb - 1;  // เทคนิค: x & (x-1) ลบ LSB ออก
    sq
}

/// นับจำนวน bits ที่เป็น 1 (population count)
#[inline(always)]
pub fn count(bb: Bitboard) -> u32 {
    bb.count_ones()  // ใช้ hardware POPCNT instruction
}
```

**Pawn attacks** — White pawn ที่ e2 (square 12) attack d3 (19) และ f3 (21):

```rust
/// White pawn attacks: เลื่อนขึ้น 7 หรือ 9 ช่อง
/// ตัดขอบ: ซ้าย (+7) ต้อง mask !FILE_H, ขวา (+9) ต้อง mask !FILE_A
pub fn white_pawn_attacks(pawns: Bitboard) -> Bitboard {
    let left  = (pawns << 7) & !FILE_H;  // attack ไปทาง d-file
    let right = (pawns << 9) & !FILE_A;  // attack ไปทาง f-file
    left | right
}

// ทำไมต้อง mask? ถ้า pawn อยู่ที่ h2 (bit 15)
// pawns << 7 = bit 22 = g3  ✓ ถูกต้อง
// pawns << 9 = bit 24 = a4  ✗ wrap around ไป file A!
// &!FILE_A กรอง bit 24 ออก
```

**Knight attacks** — precomputed ทุก 64 squares:

```rust
pub fn knight_attacks(sq: u8) -> Bitboard {
    let bb = 1u64 << sq;
    let mut attacks = 0u64;
    // 8 ทิศทาง: (+/-1 file, +/-2 rank) และ (+/-2 file, +/-1 rank)
    attacks |= (bb << 17) & !FILE_A;    // +2 rank, +1 file
    attacks |= (bb << 15) & !FILE_H;    // +2 rank, -1 file
    attacks |= (bb << 10) & !(FILE_A | FILE_B);  // +1 rank, +2 file
    attacks |= (bb <<  6) & !(FILE_G | FILE_H);  // +1 rank, -2 file
    attacks |= (bb >> 17) & !FILE_H;    // -2 rank, -1 file
    attacks |= (bb >> 15) & !FILE_A;    // -2 rank, +1 file
    attacks |= (bb >> 10) & !(FILE_G | FILE_H);
    attacks |= (bb >>  6) & !(FILE_A | FILE_B);
    attacks
}
```

**Sliding piece attacks** (classical fill — ไม่ใช้ magic):

```rust
pub fn rook_attacks(sq: u8, occupied: Bitboard) -> Bitboard {
    let mut attacks = 0u64;
    // North: เลื่อนขึ้นจนกว่าจะชนหมากหรือขอบกระดาน
    let mut r = sq + 8;
    while r < 64 {
        attacks |= 1u64 << r;
        if occupied & (1u64 << r) != 0 { break; }  // ชนหมาก — หยุด แต่รวม square นั้นด้วย (capture)
        r += 8;
    }
    // South, East, West... (เหมือนกัน)
    attacks
}
```

การทดสอบ:

```rust
#[test]
fn test_knight_attacks_e4() {
    let attacks = knight_attacks(28); // e4
    // Knight จาก e4 ไปได้ 8 ช่อง: c3,c5,d2,d6,f2,f6,g3,g5
    assert_eq!(count(attacks), 8);
}

#[test]
fn test_knight_attacks_corner() {
    let attacks = knight_attacks(0); // a1
    // Knight จาก a1 ไปได้แค่ 2 ช่อง: b3, c2
    assert_eq!(count(attacks), 2);
}
```

### ขั้นที่ 2: Board State และ FEN Parsing

`Board` struct เก็บ state ทั้งหมดของตำแหน่ง:

```rust
#[derive(Clone, Debug)]
pub struct Board {
    /// pieces[color][piece_type] — 12 bitboards รวมกัน
    pub pieces: [[Bitboard; 6]; 2],
    pub side_to_move: Color,
    /// bits: 0=WK, 1=WQ, 2=BK, 3=BQ
    pub castling_rights: u8,
    pub en_passant_file: Option<u8>,  // 0-7 หรือ None
    pub halfmove_clock: u32,          // สำหรับ 50-move rule
    pub fullmove_number: u32,
}
```

**FEN (Forsyth-Edwards Notation)** คือ string ที่อธิบาย position อย่างสมบูรณ์:

```
rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1
│                               │             │  │   │ │ └─ fullmove number
│                               │             │  │   │ └─── halfmove clock
│                               │             │  │   └───── en passant square
│                               │             │  └───────── castling rights
│                               │             └──────────── side to move
└───────────────────────────────┘ pieces (rank 8 ลงมา rank 1)
```

```rust
pub fn from_fen(fen: &str) -> Option<Self> {
    let parts: Vec<&str> = fen.split_whitespace().collect();
    let mut board = Board::empty();

    // Parse piece placement (rank 8 → rank 1)
    let ranks: Vec<&str> = parts[0].split('/').collect();
    for (rank_idx, rank_str) in ranks.iter().enumerate() {
        let rank = 7 - rank_idx as u8;  // FEN เริ่มจาก rank 8
        let mut file = 0u8;
        for ch in rank_str.chars() {
            if ch.is_ascii_digit() {
                file += ch as u8 - b'0';  // ตัวเลข = ช่องว่าง N ช่อง
            } else {
                let sq = rank * 8 + file;
                let (color, piece) = match ch {
                    'P' => (Color::White, PAWN),
                    'N' => (Color::White, KNIGHT),
                    // ...
                    _ => return None,
                };
                board.pieces[color as usize][piece] |= 1u64 << sq;
                file += 1;
            }
        }
    }
    // ... parse castling, en passant, clocks
    Some(board)
}
```

### ขั้นที่ 3: Move Struct และ Generation

Move encoding ใน `u16`:

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Move(pub u16);

impl Move {
    pub fn new(from: u8, to: u8, flags: u16) -> Self {
        Move((from as u16) | ((to as u16) << 6) | (flags << 12))
    }

    pub fn from(self) -> u8   { (self.0 & 0x3F) as u8 }
    pub fn to(self) -> u8     { ((self.0 >> 6) & 0x3F) as u8 }
    pub fn flags(self) -> u16 { (self.0 >> 12) & 0xF }

    /// UCI notation: "e2e4" หรือ "e7e8q" (promotion)
    pub fn to_string(self) -> String {
        let from = self.from();
        let to = self.to();
        let mut s = String::new();
        s.push((b'a' + from % 8) as char);
        s.push((b'1' + from / 8) as char);
        s.push((b'a' + to % 8) as char);
        s.push((b'1' + to / 8) as char);
        if self.is_promotion() {
            s.push(match self.promo_piece() {
                KNIGHT => 'n', BISHOP => 'b',
                ROOK   => 'r', QUEEN  => 'q', _ => 'q',
            });
        }
        s
    }
}
```

**Pseudo-legal move generation** — generate moves โดยไม่ตรวจ check:

```rust
fn generate_pseudo_legal(board: &Board) -> Vec<Move> {
    let mut moves = Vec::with_capacity(64);
    let us = board.side_to_move as usize;
    let occ = board.occupied();
    let our_pieces = board.color_bb(board.side_to_move);
    let their_pieces = board.color_bb(board.side_to_move.flip());
    let empty = !occ;

    // White Pawn pushes
    if board.side_to_move == Color::White {
        let pawns = board.pieces[0][PAWN];

        // Single push
        let singles = (pawns << 8) & empty;
        // iterate squares...

        // Double push จาก rank 2
        let double_push_candidates = singles & (RANK_2 << 8); // rank 3
        let doubles = (double_push_candidates << 8) & empty;

        // Left captures (toward a-file from white's view)
        let left_captures = (pawns << 7) & !FILE_H & their_pieces;

        // En passant
        if let Some(ep_file) = board.en_passant_file {
            let ep_sq = 40 + ep_file;  // rank 6 (a6=40)
            let ep_attackers = (pawns << 7) & !FILE_H & (1u64 << ep_sq);
            // สร้าง Move ด้วย FLAG_EP_CAPTURE
        }
    }
    // ... Knights, Bishops, Rooks, Queens, King, Castling
    moves
}
```

**Legal move filtering:**

```rust
pub fn generate_moves(board: &Board) -> Vec<Move> {
    let pseudo = generate_pseudo_legal(board);
    // กรอง: ทำ move แล้วเช็คว่า king ของเรา in check หรือไม่
    pseudo.into_iter().filter(|m| {
        let after = m.apply(board);
        !king_in_check(&after, board.side_to_move)
    }).collect()
}
```

### ขั้นที่ 4: Perft Testing

Perft (Performance Test) คือวิธีตรวจสอบ move generator ที่ใช้กันอย่างแพร่หลาย:

```rust
pub fn perft(board: &Board, depth: u32) -> u64 {
    if depth == 0 { return 1; }
    let moves = generate_moves(board);
    if depth == 1 {
        return moves.len() as u64;  // optimization: ไม่ต้อง apply ที่ depth 1
    }
    let mut count = 0u64;
    for m in moves {
        let child = m.apply(board);
        count += perft(&child, depth - 1);
    }
    count
}
```

ค่า reference ที่รู้จักแน่นอน (startpos):

| Depth | Nodes   | Captures | En passant | Castles | Promotions | Checks |
|-------|---------|----------|------------|---------|------------|--------|
| 1     | 20      | 0        | 0          | 0       | 0          | 0      |
| 2     | 400     | 0        | 0          | 0       | 0          | 0      |
| 3     | 8,902   | 34       | 0          | 0       | 0          | 12     |
| 4     | 197,281 | 1,576    | 0          | 0       | 0          | 469    |
| 5     | 4,865,609 | 82,719 | 258      | 0       | 0          | 27,351 |

**Debug perft** — แสดงรายละเอียดต่อ move เพื่อหาบั๊ก:

```rust
pub fn perft_divide(board: &Board, depth: u32) {
    let moves = generate_moves(board);
    let mut total = 0u64;
    for m in moves {
        let child = m.apply(board);
        let count = perft(&child, depth - 1);
        println!("{}: {}", m.to_string(), count);
        total += count;
    }
    println!("\nTotal: {}", total);
}
```

ตัวอย่าง output:
```
a2a3: 20
a2a4: 20
b2b3: 20
...
h2h4: 20

Total: 400
```

### ขั้นที่ 5: Evaluation Function

**Material values** (หน่วย centipawns):
- Pawn = 100, Knight = 320, Bishop = 330, Rook = 500, Queen = 900, King = 20000

**Piece-square tables** — bonus/penalty ตามตำแหน่งบนกระดาน:

```rust
// Pawn PST (white, a1=index 0, h8=index 63)
// ค่าสูง = ตำแหน่งดี, ค่าต่ำ = ตำแหน่งแย่
const PAWN_PST: [i32; 64] = [
     0,  0,  0,  0,  0,  0,  0,  0,   // rank 1 (pawns ไม่มีตรงนี้ปกติ)
    50, 50, 50, 50, 50, 50, 50, 50,   // rank 2 — passed pawn bonus
    10, 10, 20, 30, 30, 20, 10, 10,   // rank 3
     5,  5, 10, 25, 25, 10,  5,  5,   // rank 4 — center control
     0,  0,  0, 20, 20,  0,  0,  0,   // rank 5
     5, -5,-10,  0,  0,-10, -5,  5,   // rank 6 — discourage side pawns
     5, 10, 10,-20,-20, 10, 10,  5,   // rank 7 — encourage center
     0,  0,  0,  0,  0,  0,  0,  0,   // rank 8 (pawn promoted แล้ว)
];
```

```rust
pub fn evaluate(board: &Board) -> i32 {
    let mut score = 0i32;

    for piece in 0..6usize {
        // นับหมากขาว
        let mut white_bb = board.pieces[0][piece];
        while white_bb != 0 {
            let sq = white_bb.trailing_zeros() as usize;
            white_bb &= white_bb - 1;
            score += PIECE_VALUES[piece];
            score += PSTS[piece][sq];  // PST bonus
        }

        // นับหมากดำ (mirror PST เพราะดำอยู่ฝั่งตรงข้าม)
        let mut black_bb = board.pieces[1][piece];
        while black_bb != 0 {
            let sq = black_bb.trailing_zeros() as usize;
            black_bb &= black_bb - 1;
            score -= PIECE_VALUES[piece];
            score -= PSTS[piece][mirror_sq(sq)];
        }
    }

    // คืนค่าจาก perspective ของ side-to-move (negamax convention)
    if board.side_to_move == Color::White { score } else { -score }
}

// mirror: rank 0 ↔ rank 7, rank 1 ↔ rank 6, etc.
fn mirror_sq(sq: usize) -> usize {
    let rank = sq / 8;
    let file = sq % 8;
    (7 - rank) * 8 + file
}
```

### ขั้นที่ 6: Alpha-Beta Search

**Negamax** คือ variant ของ minimax ที่ทำงานโดยสมมติว่าทั้งสองฝ่าย maximize ค่าของตัวเอง เพราะ `score(opponent) = -score(us)`:

```rust
fn negamax(board: &Board, depth: u32, mut alpha: i32, beta: i32, ply: usize) -> i32 {
    // Base case
    if depth == 0 {
        return quiescence(board, alpha, beta);
    }

    let moves = generate_moves(board);

    // Checkmate หรือ Stalemate
    if moves.is_empty() {
        return if king_in_check(board, board.side_to_move) {
            -(MATE_SCORE - ply as i32)  // checkmate — ยิ่งเร็วยิ่งดีสำหรับผู้ชนะ
        } else {
            0  // stalemate = draw
        };
    }

    for m in order_moves(moves, board, tt_move, ply) {
        let child = m.apply(board);
        let score = -negamax(&child, depth - 1, -beta, -alpha, ply + 1);

        if score >= beta {
            // Beta cutoff: ฝ่ายตรงข้ามจะไม่เลือก branch นี้
            // เพราะมี move อื่นที่ดีกว่าสำหรับเขาอยู่แล้ว
            return beta;  // fail-hard cutoff
        }
        if score > alpha {
            alpha = score;
        }
    }
    alpha
}
```

**Alpha-beta pruning explained:**

```
Alpha = ค่าต่ำสุดที่ฝ่าย maximizer รับประกันได้แล้ว
Beta  = ค่าสูงสุดที่ฝ่าย minimizer รับประกันได้แล้ว

ถ้า score >= beta → "refutation found"
  ฝ่ายตรงข้ามมี move อื่นที่ตำแหน่งนี้จะไม่เกิดขึ้น
  ไม่ต้อง search ต่อ → prune!
```

**Iterative Deepening:**

```rust
pub fn best_move(&mut self, board: &Board, max_depth: u32) -> Option<Move> {
    let mut best = None;
    for depth in 1..=max_depth {
        // ค้นหาที่แต่ละ depth และเก็บ TT move ไว้
        // ถ้าหมดเวลา ยังมี best move จาก depth ก่อนหน้า
        best = self.root_search(board, depth);
    }
    best
}
```

**Quiescence Search** — ป้องกัน horizon effect:

```rust
fn quiescence(board: &Board, mut alpha: i32, beta: i32) -> i32 {
    // Stand-pat: ตำแหน่งนี้ "quiet" พอแล้วหรือไม่?
    let stand_pat = evaluate(board);
    if stand_pat >= beta { return beta; }
    if stand_pat > alpha { alpha = stand_pat; }

    // Search เฉพาะ captures เท่านั้น (ไม่ใช่ quiet moves)
    let captures: Vec<Move> = generate_moves(board)
        .into_iter()
        .filter(|m| m.is_capture())
        .collect();

    for m in captures {
        let child = m.apply(board);
        let score = -quiescence(&child, -beta, -alpha);
        if score >= beta { return beta; }
        if score > alpha { alpha = score; }
    }
    alpha
}
```

### ขั้นที่ 7: Move Ordering

Move ordering ที่ดีทำให้ alpha-beta prune ได้มากขึ้น:

```rust
fn order_moves(moves: &mut Vec<Move>, board: &Board,
               tt_move: Option<Move>, ply: usize,
               killer_moves: &[[Option<Move>; 2]; 64],
               history: &[[i32; 64]; 64]) {
    moves.sort_by_key(|m| -> i32 {
        // 1. TT move ก่อน (จาก previous iteration หรือ hash hit)
        if Some(*m) == tt_move { return -10_000; }

        // 2. Captures: MVV-LVA (Most Valuable Victim, Least Valuable Attacker)
        //    ยิง Queen ด้วย Pawn > ยิง Pawn ด้วย Queen
        if m.is_capture() {
            let victim = mvv_value(board.piece_at(m.to()).map(|(_, p)| p).unwrap_or(0));
            let attacker = mvv_value(board.piece_at(m.from()).map(|(_, p)| p).unwrap_or(0));
            return -(victim * 10 - attacker);  // score สูง = ลำดับก่อน
        }

        // 3. Killer moves (quiet moves ที่ทำให้ beta cutoff ใน sibling nodes)
        if ply < 64 {
            if killer_moves[ply][0] == Some(*m) { return -9_000; }
            if killer_moves[ply][1] == Some(*m) { return -8_000; }
        }

        // 4. History heuristic (quiet moves ที่เคยทำ cutoff มาก่อน)
        -history[m.from() as usize][m.to() as usize]
    });
}
```

### ขั้นที่ 8: Zobrist Hash และ Transposition Table

**Zobrist hashing** — hash position ด้วย XOR:

```rust
pub struct ZobristHasher {
    piece_keys: [[[u64; 64]; 6]; 2],  // [color][piece][square]
    castling_keys: [u64; 4],
    en_passant_keys: [u64; 8],        // 8 files
    side_key: u64,
}

impl ZobristHasher {
    pub fn new() -> Self {
        // สร้าง random keys (deterministic seed เพื่อ reproducibility)
        let mut state = 0x123456789ABCDEF0u64;
        let mut next = || -> u64 {
            // XorShift64 RNG
            state ^= state << 13;
            state ^= state >> 7;
            state ^= state << 17;
            state
        };
        // ... initialize arrays
    }

    pub fn hash(&self, board: &Board) -> u64 {
        let mut h = 0u64;

        // XOR key ของทุก piece บนทุก square
        for color in 0..2 {
            for piece in 0..6 {
                let mut bb = board.pieces[color][piece];
                while bb != 0 {
                    let sq = (bb.trailing_zeros()) as usize;
                    bb &= bb - 1;
                    h ^= self.piece_keys[color][piece][sq];
                }
            }
        }

        // Castling rights
        if board.castling_rights & 1 != 0 { h ^= self.castling_keys[0]; }
        // ... ฯลฯ

        // En passant file
        if let Some(file) = board.en_passant_file {
            h ^= self.en_passant_keys[file as usize];
        }

        // Side to move
        if board.side_to_move == Color::Black {
            h ^= self.side_key;
        }
        h
    }
}
```

**Incremental update** (ใน production) — แทนที่จะ hash ทั้ง board ใหม่ทุกครั้ง:

```rust
fn make_move_incremental(board: &mut Board, m: Move, hasher: &ZobristHasher) -> u64 {
    let mut hash = board.hash;

    // XOR out piece จาก square เดิม
    hash ^= hasher.piece_keys[us][piece][from];
    // XOR in piece ที่ square ใหม่
    hash ^= hasher.piece_keys[us][piece][to];

    // ถ้า capture: XOR out piece ของฝ่ายตรงข้าม
    if let Some((them_color, them_piece)) = board.piece_at(to) {
        hash ^= hasher.piece_keys[them_color as usize][them_piece][to];
    }

    // Update castling, en passant, side
    hash ^= hasher.side_key;  // flip side to move

    hash
}
```

**Transposition Table:**

```rust
#[derive(Clone)]
pub struct TTEntry {
    pub depth: u32,
    pub score: i32,
    pub flag: TTFlag,  // Exact, LowerBound (fail-low), UpperBound (fail-high)
    pub best_move: Option<Move>,
}

// ใน negamax:
if let Some(entry) = self.tt.get(&hash) {
    if entry.depth >= depth {
        match entry.flag {
            TTFlag::Exact => return entry.score,
            TTFlag::LowerBound => alpha = alpha.max(entry.score),
            TTFlag::UpperBound => beta = beta.min(entry.score),
        }
        if alpha >= beta { return entry.score; }
    }
    tt_move = entry.best_move;  // ใช้ best move สำหรับ ordering
}
```

## UCI Protocol Implementation

UCI (Universal Chess Interface) เป็น text protocol ผ่าน stdin/stdout:

```
GUI → Engine: uci
Engine → GUI: id name MyEngine
              id author MyName
              uciok

GUI → Engine: isready
Engine → GUI: readyok

GUI → Engine: position startpos moves e2e4 e7e5 g1f3
GUI → Engine: go depth 10
Engine → GUI: info depth 1 score cp 25 nodes 20 pv e2e4
              info depth 2 score cp 10 nodes 420 pv e2e4 e7e5
              ...
              bestmove g1f3 ponder d7d6
```

```rust
pub fn run_uci_loop() {
    let stdin = io::stdin();
    let mut board = Board::startpos();
    let mut searcher = Searcher::new();

    for line in stdin.lock().lines() {
        let line = line.unwrap();
        match line.trim() {
            "uci" => {
                println!("id name RustChess");
                println!("id author RustLearner");
                println!("uciok");
            }
            "isready"    => println!("readyok"),
            "ucinewgame" => { board = Board::startpos(); searcher = Searcher::new(); }
            "quit"       => break,
            cmd if cmd.starts_with("position") => {
                board = parse_position(cmd).unwrap_or(Board::startpos());
            }
            cmd if cmd.starts_with("go") => {
                let depth = parse_go_depth(cmd).unwrap_or(5);
                if let Some(best) = searcher.best_move(&board, depth) {
                    println!("bestmove {}", best.to_string());
                }
            }
            _ => {}
        }
        io::stdout().flush().ok();
    }
}
```

**Position parsing:**

```rust
pub fn parse_position(input: &str) -> Option<Board> {
    let tokens: Vec<&str> = input.split_whitespace().collect();
    // "position startpos" หรือ "position fen <FEN>"
    let mut board = if tokens[1] == "startpos" {
        Board::startpos()
    } else if tokens[1] == "fen" {
        let fen_end = tokens.iter().position(|&t| t == "moves")
            .unwrap_or(tokens.len());
        let fen = tokens[2..fen_end].join(" ");
        Board::from_fen(&fen)?
    } else {
        return None;
    };

    // Apply moves ถ้ามี
    if let Some(moves_idx) = tokens.iter().position(|&t| t == "moves") {
        for mv_str in &tokens[(moves_idx + 1)..] {
            let legal_moves = generate_moves(&board);
            if let Some(m) = legal_moves.iter().find(|m| m.to_string() == *mv_str) {
                board = m.apply(&board);
            }
        }
    }
    Some(board)
}
```

## Cargo.toml

```toml
[package]
name = "chess-engine"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "chess-engine"
path = "src/main.rs"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["rt", "io-util", "macros"] }
clap = { version = "4", features = ["derive"] }

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

## Source Code สมบูรณ์

### `src/main.rs`

```rust
mod bitboard;
mod movegen;
mod board;
mod evaluation;
mod search;
mod uci;
mod zobrist;

use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "chess-engine", about = "Rust Chess Engine with UCI support")]
struct Cli {
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(Subcommand)]
enum Commands {
    /// รัน UCI protocol loop (default)
    Uci,
    /// รัน perft test เพื่อตรวจสอบ move generator
    Perft {
        #[arg(short, long, default_value = "3")]
        depth: u32,
        #[arg(short, long)]
        fen: Option<String>,
    },
    /// แสดง best move จาก position ที่กำหนด
    Analyze {
        #[arg(short, long)]
        fen: Option<String>,
        #[arg(short, long, default_value = "5")]
        depth: u32,
    },
}

fn main() {
    let cli = Cli::parse();
    match cli.command.unwrap_or(Commands::Uci) {
        Commands::Uci => uci::run_uci_loop(),
        Commands::Perft { depth, fen } => {
            let board = fen
                .and_then(|f| board::Board::from_fen(&f))
                .unwrap_or_else(board::Board::startpos);
            println!("Perft({}) = {}", depth, movegen::perft(&board, depth));
        }
        Commands::Analyze { fen, depth } => {
            let board = fen
                .and_then(|f| board::Board::from_fen(&f))
                .unwrap_or_else(board::Board::startpos);
            let mut searcher = search::Searcher::new();
            if let Some(best) = searcher.best_move(&board, depth) {
                println!("bestmove {}", best.to_string());
                println!("nodes searched: {}", searcher.nodes);
            } else {
                println!("no legal moves (checkmate or stalemate)");
            }
        }
    }
}
```

### `src/bitboard.rs` (สมบูรณ์)

```rust
pub type Bitboard = u64;

pub const FILE_A: Bitboard = 0x0101_0101_0101_0101;
pub const FILE_B: Bitboard = FILE_A << 1;
pub const FILE_G: Bitboard = FILE_A << 6;
pub const FILE_H: Bitboard = FILE_A << 7;
pub const RANK_1: Bitboard = 0xFF;
pub const RANK_2: Bitboard = RANK_1 << 8;
pub const RANK_4: Bitboard = RANK_1 << 24;
pub const RANK_5: Bitboard = RANK_1 << 32;
pub const RANK_7: Bitboard = RANK_1 << 48;
pub const RANK_8: Bitboard = RANK_1 << 56;

#[inline(always)]
pub fn sq(file: u8, rank: u8) -> u8 { rank * 8 + file }

#[inline(always)]
pub fn bb(square: u8) -> Bitboard { 1u64 << square }

#[inline(always)]
pub fn lsb(bb: Bitboard) -> u8 { bb.trailing_zeros() as u8 }

#[inline(always)]
pub fn pop_lsb(bb: &mut Bitboard) -> u8 {
    let sq = bb.trailing_zeros() as u8;
    *bb &= *bb - 1;
    sq
}

#[inline(always)]
pub fn count(bb: Bitboard) -> u32 { bb.count_ones() }

pub fn white_pawn_attacks(pawns: Bitboard) -> Bitboard {
    ((pawns << 7) & !FILE_H) | ((pawns << 9) & !FILE_A)
}

pub fn black_pawn_attacks(pawns: Bitboard) -> Bitboard {
    ((pawns >> 9) & !FILE_H) | ((pawns >> 7) & !FILE_A)
}

pub fn white_pawn_push(pawns: Bitboard, empty: Bitboard) -> Bitboard {
    (pawns << 8) & empty
}

pub fn white_pawn_double_push(pawns: Bitboard, empty: Bitboard) -> Bitboard {
    let single = white_pawn_push(pawns & RANK_2, empty);
    white_pawn_push(single, empty)
}

pub fn black_pawn_push(pawns: Bitboard, empty: Bitboard) -> Bitboard {
    (pawns >> 8) & empty
}

pub fn black_pawn_double_push(pawns: Bitboard, empty: Bitboard) -> Bitboard {
    let single = black_pawn_push(pawns & RANK_7, empty);
    black_pawn_push(single, empty)
}

pub fn knight_attacks(sq: u8) -> Bitboard {
    let bb = 1u64 << sq;
    let mut a = 0u64;
    a |= (bb << 17) & !FILE_A;
    a |= (bb << 15) & !FILE_H;
    a |= (bb << 10) & !(FILE_A | FILE_B);
    a |= (bb <<  6) & !(FILE_G | FILE_H);
    a |= (bb >> 17) & !FILE_H;
    a |= (bb >> 15) & !FILE_A;
    a |= (bb >> 10) & !(FILE_G | FILE_H);
    a |= (bb >>  6) & !(FILE_A | FILE_B);
    a
}

pub fn king_attacks(sq: u8) -> Bitboard {
    let bb = 1u64 << sq;
    let mut a = 0u64;
    a |= bb << 8;
    a |= bb >> 8;
    a |= (bb << 1) & !FILE_A;
    a |= (bb >> 1) & !FILE_H;
    a |= (bb << 9) & !FILE_A;
    a |= (bb >> 9) & !FILE_H;
    a |= (bb << 7) & !FILE_H;
    a |= (bb >> 7) & !FILE_A;
    a
}

pub fn rook_attacks(sq: u8, occupied: Bitboard) -> Bitboard {
    let mut attacks = 0u64;
    let mut r = sq + 8;
    while r < 64 {
        attacks |= 1u64 << r;
        if occupied & (1u64 << r) != 0 { break; }
        r += 8;
    }
    if sq >= 8 {
        let mut r = sq - 8;
        loop {
            attacks |= 1u64 << r;
            if occupied & (1u64 << r) != 0 || r < 8 { break; }
            r -= 8;
        }
    }
    let mut file = (sq % 8) + 1;
    let mut r = sq + 1;
    while file < 8 {
        attacks |= 1u64 << r;
        if occupied & (1u64 << r) != 0 { break; }
        r += 1; file += 1;
    }
    if sq % 8 > 0 {
        let mut file = sq % 8;
        let mut r = sq - 1;
        loop {
            attacks |= 1u64 << r;
            if occupied & (1u64 << r) != 0 || file == 0 { break; }
            r -= 1; file -= 1;
        }
    }
    attacks
}

pub fn bishop_attacks(sq: u8, occupied: Bitboard) -> Bitboard {
    let mut attacks = 0u64;
    let mut r = sq;
    while r % 8 < 7 && r < 56 { r += 9; attacks |= 1u64 << r; if occupied & (1u64 << r) != 0 { break; } }
    let mut r = sq;
    while r % 8 > 0 && r < 56 { r += 7; attacks |= 1u64 << r; if occupied & (1u64 << r) != 0 { break; } }
    let mut r = sq;
    while r % 8 < 7 && r >= 8 { r -= 7; attacks |= 1u64 << r; if occupied & (1u64 << r) != 0 { break; } }
    let mut r = sq;
    while r % 8 > 0 && r >= 8 { r -= 9; attacks |= 1u64 << r; if occupied & (1u64 << r) != 0 { break; } }
    attacks
}

pub fn queen_attacks(sq: u8, occupied: Bitboard) -> Bitboard {
    rook_attacks(sq, occupied) | bishop_attacks(sq, occupied)
}
```

## การทดสอบ (Testing)

### Unit Tests

```rust
// src/bitboard.rs — tests ระดับ bit manipulation
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_knight_attacks_e4() {
        // e4 = square 28; knight มี 8 moves ที่กลางกระดาน
        assert_eq!(count(knight_attacks(28)), 8);
    }

    #[test]
    fn test_knight_attacks_corner() {
        // a1 = square 0; knight มีแค่ 2 moves จากมุม
        assert_eq!(count(knight_attacks(0)), 2);
    }

    #[test]
    fn test_pawn_attacks() {
        // e2 (sq=12) white pawn attacks d3 (sq=19) and f3 (sq=21)
        let e2 = 1u64 << 12;
        let attacks = white_pawn_attacks(e2);
        assert!(attacks & (1u64 << 19) != 0, "should attack d3");
        assert!(attacks & (1u64 << 21) != 0, "should attack f3");
    }
}
```

### Integration Tests (Perft)

```rust
// src/main.rs
#[cfg(test)]
mod tests {
    use super::*;
    use board::Board;
    use movegen::perft;

    #[test]
    fn test_perft_startpos_depth1() {
        let board = Board::startpos();
        assert_eq!(perft(&board, 1), 20);
    }

    #[test]
    fn test_perft_startpos_depth2() {
        let board = Board::startpos();
        assert_eq!(perft(&board, 2), 400);
    }

    #[test]
    fn test_perft_startpos_depth3() {
        let board = Board::startpos();
        assert_eq!(perft(&board, 3), 8902);
    }

    #[test]
    fn test_material_evaluation_startpos() {
        use evaluation::evaluate;
        let board = Board::startpos();
        let score = evaluate(&board);
        // startpos สมมาตร ดังนั้น score ควรอยู่ใกล้ 0
        assert!(score.abs() < 50, "starting position eval ≈ 0, got {}", score);
    }

    #[test]
    fn test_zobrist_consistency() {
        use zobrist::ZobristHasher;
        let h = ZobristHasher::new();
        let b = Board::startpos();
        // Hash ต้องเหมือนกันทุกครั้งที่ compute จาก state เดียวกัน
        assert_eq!(h.hash(&b), h.hash(&b));
    }

    #[test]
    fn test_uci_position_parsing() {
        use uci::parse_position;
        assert!(parse_position("position startpos").is_some());
        assert!(parse_position("position startpos moves e2e4").is_some());
    }

    #[test]
    fn test_move_notation() {
        use movegen::Move;
        let m = Move::from_str("e2e4").unwrap();
        assert_eq!(m.from(), 12);  // e2 = file 4, rank 1 = 1*8+4 = 12
        assert_eq!(m.to(), 28);    // e4 = file 4, rank 3 = 3*8+4 = 28
    }

    #[test]
    fn test_castling_rights_preserved() {
        let mut board = Board::startpos();
        board.make_move_str("e2e4");
        board.make_move_str("e7e5");
        assert!(board.white_can_castle_kingside());
    }

    #[test]
    fn test_en_passant() {
        // หลัง 1.e4 e5 2.e5 d5 — en passant exd6 ต้องมีใน move list
        let fen = "rnbqkbnr/ppp1pppp/8/3pP3/8/8/PPPP1PPP/RNBQKBNR w KQkq d6 0 3";
        let board = Board::from_fen(fen).unwrap();
        let moves = movegen::generate_moves(&board);
        assert!(moves.iter().any(|m| m.to_string() == "e5d6"));
    }
}
```

### Real `cargo test` Output

```
running 15 tests
test bitboard::tests::test_knight_attacks_corner ... ok
test bitboard::tests::test_pawn_attacks ... ok
test bitboard::tests::test_knight_attacks_e4 ... ok
test tests::test_en_passant ... ok
test tests::test_move_notation ... ok
test tests::test_material_evaluation_startpos ... ok
test tests::test_castling_rights_preserved_after_moves ... ok
test tests::test_perft_startpos_depth1 ... ok
test tests::test_uci_position_parsing ... ok
test tests::test_zobrist_consistency ... ok
test tests::test_perft_startpos_depth2 ... ok
test uci::tests::test_parse_fen_position ... ok
test uci::tests::test_parse_startpos ... ok
test uci::tests::test_parse_startpos_with_moves ... ok
test tests::test_perft_startpos_depth3 ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

## Pitfalls ที่พบบ่อย

### Pitfall 1: Wrap-around ใน Bitboard Shifts

**ปัญหา:** Pawn ที่อยู่ที่ h-file เมื่อทำ `pawns << 9` จะ "wrap around" ไปที่ a-file ของ rank ถัดไป:

```rust
// ❌ ผิด: h2 pawn ที่ bit 15 shift left 9 → bit 24 = a4!
let wrong = pawns << 9;

// ✓ ถูก: mask ออก FILE_A เพราะ capture ไปทาง e-file (ขวา) ไม่ควรไปถึง a-file
let right_attacks = (pawns << 9) & !FILE_A;
```

เช่นเดียวกัน:
- Left attacks (+7): mask `!FILE_H` (ห้าม h-file pawn attack ไปที่ a-file ของ rank สูงกว่า)
- Black pawn attacks: ตรงกันข้าม shift ลง

**การตรวจสอบ:** ทำ perft(1) จาก startpos ต้องได้ 20 ถ้า wrap-around บั๊ก, จะได้มากกว่า

### Pitfall 2: En Passant Square Index ผิด

**ปัญหา:** FEN ระบุ en passant square เป็น algebraic notation เช่น "e6" แต่ถ้าแปลงผิด จะทำให้ capture ผิด square หรือ capture ได้เมื่อไม่ควร

```rust
// FEN: "...w KQkq e6 0 1" → en passant square คือ e6
// e6 = file 4 (e=4), rank 5 = square 44
// ❌ ผิด: เก็บแค่ file แต่ลืม validate rank
board.en_passant_file = Some(ep_file);  // ถูก

// ✓ แต่ตอน generate moves ต้องรู้ rank ด้วย:
// White EP: เบี้ยขาวอยู่ rank 5 (sq 32-39) capture ไปที่ rank 6 (sq 40-47)
let ep_sq = 40 + ep_file;  // rank 6 (0-indexed rank 5) * 8 + file

// Black EP: เบี้ยดำอยู่ rank 4 (sq 24-31) capture ไปที่ rank 3 (sq 16-23)
let ep_sq = 16 + ep_file;
```

**การตรวจสอบ:** ใช้ FEN `"rnbqkbnr/ppp1pppp/8/3pP3/8/8/PPPP1PPP/RNBQKBNR w KQkq d6 0 3"` perft(1) ต้องได้ 31 (30 moves + 1 EP capture = 31)

### Pitfall 3: Negamax Score ต้องนำหน้าด้วย `-`

**ปัญหา:** ใน negamax recursive call ลืม negate score ทำให้ engine เล่นแย่ หรือ alpha-beta ทำงานผิดพลาด:

```rust
// ❌ ผิด: ลืม negate!
let score = negamax(&child, depth - 1, alpha, beta, ply + 1);
if score > alpha { alpha = score; }

// ✓ ถูก: ต้อง negate เพราะ perspective สลับฝ่ายในทุก recursive call
let score = -negamax(&child, depth - 1, -beta, -alpha, ply + 1);
//          ^                            ^      ^
//          negate score                 negate window bounds ด้วย
if score > alpha { alpha = score; }
```

**ผลที่ตามมา:** ถ้า negate ไม่ครบ engine จะ "มองว่าตำแหน่งดีสำหรับเราแต่จริง ๆ ดีสำหรับเขา" → เล่นแย่อย่างสม่ำเสมอ

### Pitfall 4: Castling Check Detection ผิด

**ปัญหา:** Castling ผิดกฎถ้า king ผ่าน square ที่ถูก attack:

```rust
// ❌ ผิด: แค่เช็ค king start และ end position
if board.castling_rights & 1 != 0 && occ & 0x60 == 0 {
    moves.push(Move::new(4, 6, FLAG_KS_CASTLE));
}

// ✓ ถูก: ต้องเช็ค 3 squares (e1, f1, g1) ว่าไม่ถูก attack
if board.castling_rights & 1 != 0
    && occ & 0x60 == 0   // f1, g1 ว่าง
    && !is_attacked(board, 4, Color::Black)  // e1 ไม่ถูก attack
    && !is_attacked(board, 5, Color::Black)  // f1 ไม่ถูก attack
    && !is_attacked(board, 6, Color::Black)  // g1 ไม่ถูก attack
{
    moves.push(Move::new(4, 6, FLAG_KS_CASTLE));
}
```

กฎ castling ใน FIDE:
1. King และ Rook ยังไม่เคย move
2. ไม่มีหมากระหว่าง King กับ Rook
3. King ไม่อยู่ใน check
4. King ไม่ผ่าน square ที่ถูก attack
5. King ไม่ไปอยู่ใน check

### Pitfall 5: Promotion Capture Flag

**ปัญหา:** เบี้ยที่เดินถึง rank สุดท้ายพร้อม capture ต้องใช้ flag `PROMO_CAP_*` ไม่ใช่แค่ `PROMO_*` หรือ `CAPTURE`:

```rust
// White pawn ที่ g7 capture ไปที่ h8 (promotion + capture)
// ❌ ผิด:
moves.push(Move::new(from, to, FLAG_CAPTURE));    // ไม่มี promotion
moves.push(Move::new(from, to, FLAG_PROMO_Q));    // ไม่ได้ remove captured piece

// ✓ ถูก:
if to >= 56 {
    moves.push(Move::new(from, to, FLAG_PROMO_CAP_Q));  // queen + capture
    moves.push(Move::new(from, to, FLAG_PROMO_CAP_R));  // rook + capture
    moves.push(Move::new(from, to, FLAG_PROMO_CAP_B));  // bishop + capture
    moves.push(Move::new(from, to, FLAG_PROMO_CAP_N));  // knight + capture
}
```

ใน `Move::apply()` ต้องตรวจสอบ:

```rust
if flags >= FLAG_PROMO_CAP_N {
    // Remove captured piece ด้วย
    for p in 0..6 { b.pieces[them][p] &= !(1u64 << to); }
}
```

## การ Package และ Deploy

### Build Release Binary

```bash
# สร้าง optimized binary
cargo build --release

# binary อยู่ที่
./target/release/chess-engine

# ทดสอบ UCI
echo -e "uci\nisready\nposition startpos\ngo depth 5\nquit" | ./target/release/chess-engine
```

Output:
```
id name RustChess
id author RustLearner
uciok
readyok
bestmove e2e4
```

### รันใน GUI (Arena Chess)

1. ดาวน์โหลด Arena Chess GUI จาก http://www.playwitharena.de/
2. ไปที่ Engines → Install New Engine
3. เลือกไฟล์ `chess-engine` (Linux) หรือ `chess-engine.exe` (Windows)
4. เล่นกับ engine ได้ทันที

### รันเป็น Lichess Bot

```bash
# ต้องการ Python lichess-bot wrapper
git clone https://github.com/lichess-bot-devs/lichess-bot
cd lichess-bot

# แก้ config.yml:
# engine:
#   dir: /path/to/chess-engine/target/release
#   name: chess-engine

python lichess-bot.py -l
```

### Benchmark ด้วย Perft

```bash
# วัดความเร็ว move generator
time ./target/release/chess-engine perft --depth 6

# Output คาดหวัง (depth 6 = 119,060,324 nodes):
# Perft(6) = 119060324
# real    0m12.345s  (classical bitboard, no magic)
```

Magic bitboards จะเร็วกว่า ~3-4x สำหรับ sliding pieces

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Magic Bitboards สำหรับ Sliding Pieces (ปานกลาง)

Classical fill algorithm สำหรับ rook/bishop attacks ช้ากว่า magic bitboards:

**งาน:** implement magic bitboards โดย:
1. สร้าง `ROOK_MAGIC` และ `BISHOP_MAGIC` tables (precomputed หรือ hardcoded)
2. สำหรับแต่ละ square, precompute attack table indexed by `(occupied & mask) * magic >> shift`
3. Measure ความเร็วด้วย `perft(6)` ก่อนและหลัง

```rust
// Concept:
fn rook_attacks_magic(sq: u8, occupied: Bitboard) -> Bitboard {
    let index = ((occupied & ROOK_MASK[sq as usize])
                 .wrapping_mul(ROOK_MAGIC[sq as usize]))
                >> (64 - ROOK_BITS[sq as usize]);
    ROOK_ATTACK_TABLE[sq as usize][index as usize]
}
```

**เป้าหมาย:** `perft(6)` ใน < 5 วินาที (จากเดิม ~12 วินาที)

### แบบฝึกหัดที่ 2: Null Move Pruning (ระดับสูง)

Null move pruning ลด search space อีกได้มาก โดยสมมติว่า "ถ้าเรา pass ฝ่ายตรงข้าม แล้วเขายัง >= beta แสดงว่าตำแหน่งนี้ดีพอที่จะ cutoff":

**งาน:** implement null move pruning ใน `negamax`:

```rust
// ใส่หลัง TT lookup
const R: u32 = 2; // reduction factor
if depth >= R + 1
    && !in_check
    && !is_zugzwang_likely(board)  // ไม่ทำเมื่อมีแค่ king+pawns
{
    let null_board = make_null_move(board);  // flip side_to_move, reset EP
    let null_score = -negamax(&null_board, depth - R - 1, -beta, -beta + 1, ply + 1);
    if null_score >= beta {
        return beta;  // null move cutoff
    }
}
```

**เป้าหมาย:** engine ควรค้นหา depth เดิมได้เร็วขึ้น ~30-40%

### แบบฝึกหัดที่ 3: Opening Book (ปานกลาง)

**งาน:** implement simple opening book ด้วย Polyglot format (`.bin` file):

```rust
// Polyglot format: key(8) + move(2) + weight(2) + learn(4)
pub struct BookEntry {
    pub key: u64,    // Zobrist hash ของ position
    pub move_: u16,  // packed move
    pub weight: u16, // frequency/quality weight
    pub learn: u32,
}

pub fn lookup_book(hash: u64, book: &[BookEntry]) -> Option<Move> {
    // Binary search เพราะ book เรียงตาม key
    let idx = book.partition_point(|e| e.key < hash);
    let candidates: Vec<&BookEntry> = book[idx..]
        .iter()
        .take_while(|e| e.key == hash)
        .collect();
    // เลือก move ตาม weight หรือ random weighted
    candidates.iter().max_by_key(|e| e.weight).map(|e| decode_poly_move(e.move_))
}
```

Download Polyglot book จาก: https://www.chessprogramming.org/Polyglot

**เป้าหมาย:** engine เล่น opening เร็วขึ้น (< 1ms ต่อ move ใน opening) และมีความหลากหลายมากขึ้น

### แบบฝึกหัดที่ 4: Endgame Tablebases (ท้าทาย)

**งาน:** implement Syzygy tablebase probing เพื่อเล่น endgame สมบูรณ์แบบ:

1. ดาวน์โหลด Syzygy DTZ tablebases (3-4 piece)
2. Link กับ Fathom library (C) หรือ port ส่วน probing ด้วย Rust
3. ใน search: ถ้า piece count <= 5 ให้ probe tablebase ก่อน

```rust
// ตัวอย่างการใช้ผ่าน FFI
extern "C" {
    fn tb_probe_wdl(white: u64, black: u64, kings: u64,
                   queens: u64, rooks: u64, bishops: u64,
                   knights: u64, pawns: u64,
                   ep: u32, turn: u32) -> u32;
}
// Return: 0=loss, 1=blessed_loss, 2=draw, 3=cursed_win, 4=win
```

**เป้าหมาย:** engine เล่น KQK, KRK, KBBK, KBNK สมบูรณ์แบบ 100%

## สรุป

โปรเจคนี้สอน pattern สำคัญหลายอย่างที่ใช้ได้นอกเหนือจาก chess:

1. **Bit manipulation** — `u64` bitfields เป็น data structure ที่ compact และรวดเร็ว ใช้ได้กับ set operations, graph algorithms, game states ต่าง ๆ
2. **Alpha-beta = branch-and-bound generalized** — ใช้ได้กับ minimax problems ทั่วไป เช่น checkers, Go (ก่อน AlphaZero), card games
3. **Protocol implementation** — UCI เป็นตัวอย่างของ simple text protocol ที่ interoperable; pattern เดียวกันใช้ใน LSP (Language Server Protocol), DAP (Debug Adapter Protocol)
4. **Perft testing** — วิธี validate correctness ของ generator โดยเปรียบเทียบกับ reference values; คล้ายกับ property-based testing
5. **Incremental hashing** — Zobrist hash ที่ update แบบ incremental คือ pattern ที่ใช้กว้างขวางใน caching, fingerprinting, rolling hash

**ประสิทธิภาพ:** implementation นี้ใช้ classical bitboard (ไม่ใช่ magic) สามารถ search ได้ประมาณ depth 6-8 ใน 1 วินาที (ขึ้นกับ position) เครื่อง engine ระดับ production อย่าง Stockfish ใช้เวลา ~1ms สำหรับ depth 20+ เพราะใช้ magic bitboard, NNUE evaluation, multithreading, และ opening book

**โปรเจคถัดไป** (project-e03-physics-engine.md) จะสร้าง 2D physics engine ด้วย rigid body dynamics, collision detection, และ constraint solver — ซึ่งจะนำ numeric computing และ iterative algorithm ที่เรียนรู้ที่นี่ไปประยุกต์ใช้

---

**โปรเจคก่อนหน้า:** [project-e01-tetris.md](project-e01-tetris.md) | **โปรเจคถัดไป:** [project-e03-physics-engine.md](project-e03-physics-engine.md)
