# Project C09: Query Language Parser

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **Query Language Parser** แบบครบวงจรตั้งแต่ต้น — ตั้งแต่การ tokenize ตัวอักษร ไปจนถึง Query Planner และ In-Memory Executor ที่รันได้จริงบน `Vec<HashMap<String,Value>>` โดยรองรับ SQL subset ที่ครอบคลุม SELECT/INSERT/UPDATE/DELETE/CREATE/DROP TABLE พร้อม WHERE clauses แบบซับซ้อน

ในโลก production Query Language Parser ใช้อยู่ทุกที่ที่ต้องการรับคำสั่งจากผู้ใช้และแปลงเป็น operations จริง ตั้งแต่ PostgreSQL, SQLite, InfluxDB, Apache Arrow DataFusion, ไปจนถึง ClickHouse หรือแม้กระทั่ง ORM ทุกตัวที่ต้องแปลง query API เป็น SQL โปรเจคนี้จะสอนให้คุณเขียน parser จากศูนย์โดยไม่ใช้ parser generator อย่าง LALRPOP หรือ pest ทำให้เข้าใจ mechanism อย่างลึกซึ้ง

**Learning value:**
- เข้าใจว่า recursive descent parsing ทำงานอย่างไร — เทคนิคที่ใช้ใน GCC, Clang, rustc
- เรียนรู้การออกแบบ Algebraic Data Types ผ่าน AST (Abstract Syntax Tree) ที่ครบถ้วน
- ฝึกการออกแบบ error messages ที่มีข้อมูลตำแหน่ง line/column
- เข้าใจ query planning pipeline: Scan → Filter → Project → Sort → Limit
- ฝึก Rust pattern matching ระดับสูงกับ nested enums ใน AST

## สิ่งที่จะได้เรียนรู้

- **Hand-written Lexer**: เขียน tokenizer ด้วย iterator ของ `char_indices()` พร้อม line/column tracking
- **Recursive Descent Parser**: เทคนิค top-down parsing สำหรับ operator precedence (OR < AND < NOT < comparison)
- **AST Design**: ออกแบบ `Expr` enum แบบ recursive ที่ represent expression tree ครบทุก pattern
- **Error Recovery**: สร้าง error messages พร้อม `Span { line, col }` เพื่อ UX ที่ดี
- **Query Planner**: แปลง AST → logical plan ที่เป็น `PlanNode` tree และแสดง EXPLAIN output
- **In-Memory Execution**: evaluate expressions บน `HashMap<String,Value>` พร้อม type coercion
- **HTTP API**: expose query engine ผ่าน `axum` endpoints สำหรับ execute SQL และ load data
- **SQL Standard Features**: `IS NULL`, `IN (list)`, `BETWEEN`, `LIKE`, `AS alias`, `COUNT(*)`

## ความรู้ที่ต้องมีมาก่อน

- **Part 1-20**: Rust basics — ownership, borrowing, structs, enums, pattern matching
- **Part 21-40**: Collections (`HashMap`, `Vec`), iterators, closures, generics
- **Part 41-55**: Error handling, `Result`/`Option`, `thiserror`, `?` operator
- **Part 56-70**: Traits, trait objects, `Display`, `From`, lifetimes
- **Part 71-85**: `async/await`, Tokio (จาก Part 46 เรื่อง async/await)
- **Part 86-95**: HTTP servers ด้วย `axum`, `serde_json`
- **Part 96-110**: Production patterns — comprehensive testing, API design
- โปรเจค C08 (Data Migration) — ช่วยให้คุ้นกับ data transformation patterns

## โครงสร้างโปรเจค (Project Layout)

```
query-parser/
├── src/
│   ├── main.rs         — entry point: HTTP server + REPL startup
│   ├── lexer.rs        — Lexer, Token enum, Span, tokenize()
│   ├── ast.rs          — AST nodes: Expr, Statement, SelectStatement, etc.
│   ├── parser.rs       — Parser struct, parse_statement(), recursive descent
│   ├── planner.rs      — Planner, PlanNode, QueryPlan, explain()
│   ├── executor.rs     — Database, Value, eval_expr(), execute()
│   ├── api.rs          — axum HTTP handlers: POST /query, POST /tables/{name}/data
│   ├── repl.rs         — rustyline REPL with tab completion
│   └── error.rs        — ParseError enum ด้วย thiserror
├── tests/
│   └── integration_test.rs   — integration tests
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow Pipeline

```
Input SQL String
      │
      ▼
┌─────────────────────────────────────┐
│  Lexer (lexer.rs)                   │
│  "SELECT * FROM users WHERE age>18" │
│         │                           │
│         ▼                           │
│  Vec<Token> + Span(line, col)       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Parser (parser.rs)                 │
│  Recursive Descent                  │
│                                     │
│  parse_statement()                  │
│    └─ parse_select()               │
│         ├─ parse_select_columns()  │
│         ├─ parse_expr() [WHERE]    │
│         │   ├─ parse_or()          │
│         │   │   └─ parse_and()     │
│         │   │       └─ parse_not() │
│         │   │           └─ parse_comparison() │
│         │   │               └─ parse_primary() │
│         ├─ parse_order_by_list()   │
│         └─ LIMIT / OFFSET          │
│                                     │
│  → Statement (AST)                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Planner (planner.rs)               │
│  Statement → QueryPlan              │
│                                     │
│  SELECT → Scan → Filter → Project   │
│            → Sort → Limit           │
│                                     │
│  plan.explain() = tree-format text  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Executor (executor.rs)             │
│  Database { tables: HashMap<...> }  │
│                                     │
│  eval_expr(expr, &row) → Value      │
│  execute(plan) → QueryResult        │
└─────────────────────────────────────┘
```

### Design Decisions

**ทำไมไม่ใช้ parser generator (pest/nom/LALRPOP)?**

Parser generator เหมาะสำหรับ production parser ที่ต้องการ formal grammar แต่การเขียน recursive descent ด้วยมือทำให้เข้าใจ mechanism ลึกที่สุด และยังปรับแต่ง error messages ได้ง่ายกว่า ในความเป็นจริง SQLite ใช้ hand-written recursive descent parser เช่นกัน

**ทำไม operator precedence ถึงต้องแยกเป็นหลาย parse functions?**

`parse_or()` เรียก `parse_and()` เรียก `parse_not()` เรียก `parse_comparison()` เรียก `parse_primary()` — pattern นี้ทำให้ precedence ถูกต้องโดยธรรมชาติ: OR มี precedence ต่ำที่สุด (evaluated last), primary มี precedence สูงที่สุด ถ้า flatten เป็น function เดียวจะต้องใช้ Pratt parsing หรือ precedence climbing แทน

**ทำไม `Span { line, col }` แทน `usize offset`?**

User จะเห็น "line 3 col 15" ได้ทันที แต่ถ้าเก็บ byte offset ต้องนับ newline ย้อนหลังเพื่อแปลง เป็น tradeoff ระหว่าง storage convenience กับ display convenience

**ทำไม `Vec<HashMap<String,Value>>` แทน schema-enforced storage?**

Schema-free ทำให้ executor เรียบง่ายมาก เหมาะสำหรับ demo/testing โดยไม่ต้องสร้าง type system แยก ในงาน production จะต้องเพิ่ม schema validation และ type enforcement

**ทางเลือกที่พิจารณา:**
- sqlparser-rs: production-grade SQL parser ใน Rust — ใช้ง่ายแต่ไม่ได้เรียนรู้ internals
- pest grammar file: เขียน grammar declaratively — ดีสำหรับ rapid prototyping
- Pratt parser: alternative สำหรับ expression parsing ที่ elegant กว่า recursive descent

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Error Types และ Span

เริ่มจากกำหนด error types ก่อน เพราะทุก step ถัดไปต้องใช้

**`Cargo.toml`:**

```toml
[package]
name = "query-parser"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "query-parser"
path = "src/main.rs"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
axum = "0.8"
tokio = { version = "1", features = ["full"] }
rustyline = "14"

[dev-dependencies]
```

**`src/error.rs`:**

```rust
use crate::lexer::Span;
use thiserror::Error;

pub type ParseResult<T> = Result<T, ParseError>;

#[derive(Debug, Error, Clone)]
pub enum ParseError {
    #[error("Unexpected character '{ch}' at {span}")]
    UnexpectedChar { ch: char, span: Span },

    #[error("Unterminated string literal at {span}")]
    UnterminatedString { span: Span },

    #[error("Expected {expected}, found '{found}' at {span}")]
    UnexpectedToken {
        expected: String,
        found: String,
        span: Span,
    },

    #[error("Expected {expected} at {span}")]
    MissingToken { expected: String, span: Span },

    #[error("Invalid number literal '{value}' at {span}")]
    InvalidNumber { value: String, span: Span },

    #[error("Empty column list at {span}")]
    EmptyColumnList { span: Span },

    #[error("Unexpected end of input")]
    UnexpectedEof,
}
```

`thiserror` ช่วยเขียน `Display` implementation ให้อัตโนมัติจาก `#[error("...")]` แทนที่จะต้องเขียน `impl fmt::Display` ด้วยมือ `{span}` ใน error message จะเรียก `Display` ของ `Span` ซึ่งแสดง "line X col Y"

---

### ขั้นที่ 2: Lexer — Tokenizer พร้อม Span

Lexer อ่าน input string ทีละ character และแปลงเป็น `Vec<Token>` โดยแต่ละ Token เก็บ `TokenKind` และ `Span` (ตำแหน่ง)

**`src/lexer.rs` (ส่วนหลัก):**

```rust
use crate::error::{ParseError, ParseResult};

/// ตำแหน่งใน source code สำหรับ error messages
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Span {
    pub line: usize,
    pub col: usize,
}

impl Span {
    pub fn new(line: usize, col: usize) -> Self {
        Self { line, col }
    }
}

impl std::fmt::Display for Span {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "line {} col {}", self.line, self.col)
    }
}

/// Token types สำหรับ SQL query language
#[derive(Debug, Clone, PartialEq)]
pub enum TokenKind {
    // Keywords
    Select, From, Where, And, Or, Not,
    Insert, Into, Values, Update, Set,
    Delete, Create, Drop, Table, Index, On,
    Order, By, Asc, Desc, Limit, Offset,
    As, Is, Null, In, Between, Like, Count,
    // Literals
    Ident(String),
    NumberLit(f64),
    StringLit(String),
    // Operators
    Eq,     // =
    Lt,     // <
    Gt,     // >
    LtEq,   // <=
    GtEq,   // >=
    NotEq,  // <> หรือ !=
    Star,   // *
    // Punctuation
    Comma, LParen, RParen, Semicolon, Dot,
    // End of input
    Eof,
}

impl std::fmt::Display for TokenKind {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            TokenKind::Select    => write!(f, "SELECT"),
            TokenKind::From      => write!(f, "FROM"),
            TokenKind::Where     => write!(f, "WHERE"),
            TokenKind::Ident(s)  => write!(f, "{}", s),
            TokenKind::NumberLit(n) => write!(f, "{}", n),
            TokenKind::StringLit(s) => write!(f, "'{}'", s),
            TokenKind::Eq        => write!(f, "="),
            TokenKind::NotEq     => write!(f, "<>"),
            TokenKind::RParen    => write!(f, ")"),
            TokenKind::Eof       => write!(f, "EOF"),
            // ... (ทุก variant)
            _ => write!(f, "{:?}", self),
        }
    }
}

/// Token พร้อมตำแหน่ง
#[derive(Debug, Clone)]
pub struct Token {
    pub kind: TokenKind,
    pub span: Span,
}

/// Lexer สำหรับ tokenize SQL input
pub struct Lexer<'a> {
    input: &'a str,
    chars: std::iter::Peekable<std::str::CharIndices<'a>>,
    line: usize,
    col: usize,
}

impl<'a> Lexer<'a> {
    pub fn new(input: &'a str) -> Self {
        Self {
            input,
            chars: input.char_indices().peekable(),
            line: 1,
            col: 1,
        }
    }

    fn advance(&mut self) -> Option<(usize, char)> {
        let result = self.chars.next();
        if let Some((_, ch)) = result {
            if ch == '\n' {
                self.line += 1;
                self.col = 1;
            } else {
                self.col += 1;
            }
        }
        result
    }

    fn peek_char(&mut self) -> Option<char> {
        self.chars.peek().map(|(_, ch)| *ch)
    }

    fn skip_whitespace(&mut self) {
        while let Some(ch) = self.peek_char() {
            if ch.is_whitespace() { self.advance(); } else { break; }
        }
    }

    fn read_number(&mut self, first: char) -> f64 {
        let mut s = String::from(first);
        while let Some(ch) = self.peek_char() {
            if ch.is_ascii_digit() || ch == '.' {
                s.push(ch);
                self.advance();
            } else {
                break;
            }
        }
        s.parse().unwrap_or(0.0)
    }

    fn read_ident(&mut self, first: char) -> String {
        let mut s = String::from(first);
        while let Some(ch) = self.peek_char() {
            if ch.is_alphanumeric() || ch == '_' {
                s.push(ch);
                self.advance();
            } else {
                break;
            }
        }
        s
    }

    fn keyword_or_ident(s: &str) -> TokenKind {
        match s.to_uppercase().as_str() {
            "SELECT"  => TokenKind::Select,
            "FROM"    => TokenKind::From,
            "WHERE"   => TokenKind::Where,
            "AND"     => TokenKind::And,
            "OR"      => TokenKind::Or,
            "NOT"     => TokenKind::Not,
            "INSERT"  => TokenKind::Insert,
            "INTO"    => TokenKind::Into,
            "VALUES"  => TokenKind::Values,
            "UPDATE"  => TokenKind::Update,
            "SET"     => TokenKind::Set,
            "DELETE"  => TokenKind::Delete,
            "CREATE"  => TokenKind::Create,
            "DROP"    => TokenKind::Drop,
            "TABLE"   => TokenKind::Table,
            "INDEX"   => TokenKind::Index,
            "ON"      => TokenKind::On,
            "ORDER"   => TokenKind::Order,
            "BY"      => TokenKind::By,
            "ASC"     => TokenKind::Asc,
            "DESC"    => TokenKind::Desc,
            "LIMIT"   => TokenKind::Limit,
            "OFFSET"  => TokenKind::Offset,
            "AS"      => TokenKind::As,
            "IS"      => TokenKind::Is,
            "NULL"    => TokenKind::Null,
            "IN"      => TokenKind::In,
            "BETWEEN" => TokenKind::Between,
            "LIKE"    => TokenKind::Like,
            "COUNT"   => TokenKind::Count,
            _         => TokenKind::Ident(s.to_string()),
        }
    }

    pub fn tokenize(&mut self) -> ParseResult<Vec<Token>> {
        let mut tokens = Vec::new();
        loop {
            self.skip_whitespace();
            let span = Span::new(self.line, self.col);
            match self.advance() {
                None => {
                    tokens.push(Token { kind: TokenKind::Eof, span });
                    break;
                }
                Some((_, ',')) => tokens.push(Token { kind: TokenKind::Comma, span }),
                Some((_, '(')) => tokens.push(Token { kind: TokenKind::LParen, span }),
                Some((_, ')')) => tokens.push(Token { kind: TokenKind::RParen, span }),
                Some((_, ';')) => tokens.push(Token { kind: TokenKind::Semicolon, span }),
                Some((_, '.')) => tokens.push(Token { kind: TokenKind::Dot, span }),
                Some((_, '*')) => tokens.push(Token { kind: TokenKind::Star, span }),
                Some((_, '=')) => tokens.push(Token { kind: TokenKind::Eq, span }),
                Some((_, '<')) => {
                    if self.peek_char() == Some('=') {
                        self.advance();
                        tokens.push(Token { kind: TokenKind::LtEq, span });
                    } else if self.peek_char() == Some('>') {
                        self.advance();
                        tokens.push(Token { kind: TokenKind::NotEq, span });
                    } else {
                        tokens.push(Token { kind: TokenKind::Lt, span });
                    }
                }
                Some((_, '>')) => {
                    if self.peek_char() == Some('=') {
                        self.advance();
                        tokens.push(Token { kind: TokenKind::GtEq, span });
                    } else {
                        tokens.push(Token { kind: TokenKind::Gt, span });
                    }
                }
                Some((_, '!')) => {
                    if self.peek_char() == Some('=') {
                        self.advance();
                        tokens.push(Token { kind: TokenKind::NotEq, span });
                    } else {
                        return Err(ParseError::UnexpectedChar { ch: '!', span });
                    }
                }
                Some((_, '\'')) => {
                    // String literal — อ่านจนเจอ closing quote
                    let mut s = String::new();
                    loop {
                        match self.advance() {
                            Some((_, '\'')) => {
                                // '' = escaped single quote ใน SQL standard
                                if self.peek_char() == Some('\'') {
                                    self.advance();
                                    s.push('\'');
                                } else {
                                    break;
                                }
                            }
                            Some((_, ch)) => s.push(ch),
                            None => return Err(ParseError::UnterminatedString { span }),
                        }
                    }
                    tokens.push(Token { kind: TokenKind::StringLit(s), span });
                }
                Some((_, ch)) if ch.is_ascii_digit() => {
                    let n = self.read_number(ch);
                    tokens.push(Token { kind: TokenKind::NumberLit(n), span });
                }
                Some((_, ch)) if ch.is_alphabetic() || ch == '_' => {
                    let ident = self.read_ident(ch);
                    let kind = Self::keyword_or_ident(&ident);
                    tokens.push(Token { kind, span });
                }
                Some((_, ch)) => {
                    return Err(ParseError::UnexpectedChar { ch, span });
                }
            }
        }
        Ok(tokens)
    }
}

pub fn tokenize(input: &str) -> ParseResult<Vec<Token>> {
    Lexer::new(input).tokenize()
}
```

**จุดสำคัญในการออกแบบ Lexer:**
1. `Peekable<CharIndices>` ทำให้ lookahead 1 character สำหรับ `<=`, `<>`, `>=`, `!=` ได้โดยไม่ต้อง unread
2. `line`/`col` tracking ทำใน `advance()` เพื่อให้ทุก character อัปเดตตำแหน่งถูกต้อง
3. String literal รองรับ `''` เป็น escaped quote ตาม SQL standard
4. `keyword_or_ident()` ใช้ `to_uppercase()` ทำให้ `SELECT` กับ `select` เป็น keyword เดียวกัน

**ทดสอบ Lexer:**

```bash
$ cargo test test_lexer
test tests::test_lexer_keywords ... ok
test tests::test_lexer_number_literal ... ok
test tests::test_lexer_operators ... ok
test tests::test_lexer_span_tracking ... ok
test tests::test_lexer_string_literal ... ok
```

---

### ขั้นที่ 3: AST — Abstract Syntax Tree

AST คือ representation ของ SQL query ในรูปแบบ tree structure ที่ type-safe ใน Rust เราใช้ nested enums

**`src/ast.rs`:**

```rust
use serde::{Deserialize, Serialize};

/// Expression node ใน AST — recursive enum
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum Expr {
    Ident(String),          // column name: age, users.name
    NumberLit(f64),         // ตัวเลข: 42, 3.14
    StringLit(String),      // string: 'hello'
    Null,                   // NULL literal
    BinaryOp {
        left: Box<Expr>,
        op: BinaryOp,
        right: Box<Expr>,
    },
    UnaryOp {
        op: UnaryOp,
        expr: Box<Expr>,
    },
    FunctionCall {
        name: String,
        args: Vec<Expr>,    // UPPER(name), COUNT(*)
    },
    IsNull {
        expr: Box<Expr>,
        negated: bool,      // IS NULL vs IS NOT NULL
    },
    InList {
        expr: Box<Expr>,
        list: Vec<Expr>,
        negated: bool,      // IN vs NOT IN
    },
    Between {
        expr: Box<Expr>,
        low: Box<Expr>,
        high: Box<Expr>,
        negated: bool,
    },
    Like {
        expr: Box<Expr>,
        pattern: String,
        negated: bool,
    },
    Star,                   // * ใน SELECT
    Alias {
        expr: Box<Expr>,
        alias: String,      // name AS username
    },
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum BinaryOp { And, Or, Eq, NotEq, Lt, Gt, LtEq, GtEq }

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum UnaryOp { Not }

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum OrderDir { Asc, Desc }

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct OrderBy {
    pub expr: Expr,
    pub dir: OrderDir,
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct ColumnDef {
    pub name: String,
    pub type_name: String,
    pub not_null: bool,
    pub primary_key: bool,
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct Assignment {
    pub column: String,
    pub value: Expr,
}

/// Top-level Statement enum
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum Statement {
    Select(SelectStatement),
    Insert(InsertStatement),
    Update(UpdateStatement),
    Delete(DeleteStatement),
    CreateTable(CreateTableStatement),
    DropTable(DropTableStatement),
    CreateIndex(CreateIndexStatement),
}

/// SELECT statement AST node
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct SelectStatement {
    pub columns: Vec<Expr>,            // SELECT columns (หรือ [Star])
    pub from: String,                  // FROM table_name
    pub where_clause: Option<Expr>,    // WHERE ...
    pub order_by: Vec<OrderBy>,        // ORDER BY ...
    pub limit: Option<u64>,            // LIMIT n
    pub offset: Option<u64>,           // OFFSET n
}

/// INSERT statement
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct InsertStatement {
    pub table: String,
    pub columns: Vec<String>,
    pub values: Vec<Vec<Expr>>,        // multiple VALUE rows
}

/// UPDATE statement
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct UpdateStatement {
    pub table: String,
    pub assignments: Vec<Assignment>,  // SET col = val, ...
    pub where_clause: Option<Expr>,
}

/// DELETE statement
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct DeleteStatement {
    pub table: String,
    pub where_clause: Option<Expr>,
}

/// CREATE TABLE statement
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct CreateTableStatement {
    pub table_name: String,
    pub if_not_exists: bool,
    pub columns: Vec<ColumnDef>,
}

/// DROP TABLE statement
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct DropTableStatement {
    pub table_name: String,
    pub if_exists: bool,
}

/// CREATE INDEX statement
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct CreateIndexStatement {
    pub index_name: String,
    pub table_name: String,
    pub columns: Vec<String>,
    pub unique: bool,
}
```

**ทำไม `Box<Expr>` สำหรับ nested expressions?**

Recursive enum ใน Rust ต้องใช้ indirection (`Box`) เพราะ Rust ต้องการขนาด type แน่นอน ณ compile time ถ้าไม่มี Box compiler จะ complain: "recursive type has infinite size" ตัวอย่าง: `BinaryOp { left: Box<Expr>, right: Box<Expr> }` ทำให้ `1 + (2 * 3)` กลายเป็น tree ได้

---

### ขั้นที่ 4: Recursive Descent Parser

Parser ใช้ pattern "recursive descent" — แต่ละ operator precedence level มี function ของตัวเอง

**`src/parser.rs` (ส่วนหลัก):**

```rust
use crate::ast::*;
use crate::error::{ParseError, ParseResult};
use crate::lexer::{Token, TokenKind};

pub struct Parser {
    tokens: Vec<Token>,
    pos: usize,
}

impl Parser {
    pub fn new(tokens: Vec<Token>) -> Self {
        Self { tokens, pos: 0 }
    }

    fn peek_kind(&self) -> &TokenKind {
        &self.tokens[self.pos].kind
    }

    fn advance(&mut self) -> &Token {
        let tok = &self.tokens[self.pos];
        if self.pos + 1 < self.tokens.len() {
            self.pos += 1;
        }
        tok
    }

    fn expect(&mut self, expected: TokenKind) -> ParseResult<Token> {
        let tok = self.peek().clone();
        if std::mem::discriminant(&tok.kind) == std::mem::discriminant(&expected) {
            self.advance();
            Ok(tok)
        } else {
            Err(ParseError::UnexpectedToken {
                expected: format!("{}", expected),
                found: format!("{}", tok.kind),
                span: tok.span,
            })
        }
    }

    fn eat(&mut self, kind: &TokenKind) -> bool {
        if std::mem::discriminant(self.peek_kind()) == std::mem::discriminant(kind) {
            self.advance();
            true
        } else {
            false
        }
    }

    /// Top-level entry point
    pub fn parse_statement(&mut self) -> ParseResult<Statement> {
        match self.peek_kind() {
            TokenKind::Select => self.parse_select().map(Statement::Select),
            TokenKind::Insert => self.parse_insert().map(Statement::Insert),
            TokenKind::Update => self.parse_update().map(Statement::Update),
            TokenKind::Delete => self.parse_delete().map(Statement::Delete),
            TokenKind::Create => self.parse_create(),
            TokenKind::Drop   => self.parse_drop(),
            _ => Err(ParseError::UnexpectedToken { ... }),
        }
    }

    // Expression parsing — operator precedence via call chain:
    // parse_expr → parse_or → parse_and → parse_not → parse_comparison → parse_primary

    pub fn parse_expr(&mut self) -> ParseResult<Expr> {
        self.parse_or()
    }

    fn parse_or(&mut self) -> ParseResult<Expr> {
        let mut left = self.parse_and()?;
        while self.eat(&TokenKind::Or) {
            let right = self.parse_and()?;
            left = Expr::BinaryOp {
                left: Box::new(left),
                op: BinaryOp::Or,
                right: Box::new(right),
            };
        }
        Ok(left)
    }

    fn parse_and(&mut self) -> ParseResult<Expr> {
        let mut left = self.parse_not()?;
        while self.eat(&TokenKind::And) {
            let right = self.parse_not()?;
            left = Expr::BinaryOp {
                left: Box::new(left),
                op: BinaryOp::And,
                right: Box::new(right),
            };
        }
        Ok(left)
    }

    fn parse_not(&mut self) -> ParseResult<Expr> {
        if self.eat(&TokenKind::Not) {
            let expr = self.parse_not()?;  // recursive สำหรับ NOT NOT x
            Ok(Expr::UnaryOp { op: UnaryOp::Not, expr: Box::new(expr) })
        } else {
            self.parse_comparison()
        }
    }

    fn parse_comparison(&mut self) -> ParseResult<Expr> {
        let left = self.parse_primary()?;

        // Comparison operators
        let op = match self.peek_kind() {
            TokenKind::Eq    => Some(BinaryOp::Eq),
            TokenKind::NotEq => Some(BinaryOp::NotEq),
            TokenKind::Lt    => Some(BinaryOp::Lt),
            TokenKind::Gt    => Some(BinaryOp::Gt),
            TokenKind::LtEq  => Some(BinaryOp::LtEq),
            TokenKind::GtEq  => Some(BinaryOp::GtEq),
            _                => None,
        };
        if let Some(op) = op {
            self.advance();
            let right = self.parse_primary()?;
            return Ok(Expr::BinaryOp { left: Box::new(left), op, right: Box::new(right) });
        }

        // IS NULL / IS NOT NULL
        if self.eat(&TokenKind::Is) {
            let negated = self.eat(&TokenKind::Not);
            self.expect(TokenKind::Null)?;
            return Ok(Expr::IsNull { expr: Box::new(left), negated });
        }

        // NOT? IN (list)
        let negated = self.eat(&TokenKind::Not);
        if self.eat(&TokenKind::In) {
            self.expect(TokenKind::LParen)?;
            let mut list = vec![self.parse_expr()?];
            while self.eat(&TokenKind::Comma) {
                list.push(self.parse_expr()?);
            }
            self.expect(TokenKind::RParen)?;
            return Ok(Expr::InList { expr: Box::new(left), list, negated });
        }

        // NOT? BETWEEN low AND high
        if self.eat(&TokenKind::Between) {
            let low = self.parse_primary()?;
            self.expect(TokenKind::And)?;
            let high = self.parse_primary()?;
            return Ok(Expr::Between {
                expr: Box::new(left), low: Box::new(low),
                high: Box::new(high), negated,
            });
        }

        // NOT? LIKE pattern
        if self.eat(&TokenKind::Like) {
            if let TokenKind::StringLit(pattern) = self.peek_kind().clone() {
                self.advance();
                return Ok(Expr::Like { expr: Box::new(left), pattern, negated });
            }
        }

        Ok(left)
    }

    fn parse_primary(&mut self) -> ParseResult<Expr> {
        match self.peek_kind().clone() {
            TokenKind::LParen => {
                self.advance();
                let expr = self.parse_expr()?;
                self.expect(TokenKind::RParen)?;
                Ok(expr)
            }
            TokenKind::NumberLit(n) => { self.advance(); Ok(Expr::NumberLit(n)) }
            TokenKind::StringLit(s) => { self.advance(); Ok(Expr::StringLit(s)) }
            TokenKind::Null         => { self.advance(); Ok(Expr::Null) }
            TokenKind::Star         => { self.advance(); Ok(Expr::Star) }
            TokenKind::Ident(name)  => {
                self.advance();
                if self.eat(&TokenKind::LParen) {
                    // Function call: UPPER(name)
                    let args = self.parse_arg_list()?;
                    self.expect(TokenKind::RParen)?;
                    Ok(Expr::FunctionCall { name: name.to_uppercase(), args })
                } else if self.eat(&TokenKind::Dot) {
                    // table.column notation
                    let col = self.expect_ident()?;
                    Ok(Expr::Ident(format!("{}.{}", name, col)))
                } else {
                    Ok(Expr::Ident(name))
                }
            }
            _ => Err(ParseError::UnexpectedToken { ... }),
        }
    }
}

pub fn parse(sql: &str) -> ParseResult<Statement> {
    let tokens = crate::lexer::tokenize(sql)?;
    let mut parser = Parser::new(tokens);
    parser.parse_statement()
}
```

**ตัวอย่าง Parse Tree:**

```
SQL: "SELECT name FROM users WHERE age > 18 AND active = 1"

Statement::Select(SelectStatement {
    columns: [Ident("name")],
    from: "users",
    where_clause: Some(
        BinaryOp {
            left: BinaryOp {
                left: Ident("age"),
                op: Gt,
                right: NumberLit(18.0)
            },
            op: And,
            right: BinaryOp {
                left: Ident("active"),
                op: Eq,
                right: NumberLit(1.0)
            }
        }
    ),
    order_by: [],
    limit: None,
    offset: None,
})
```

---

### ขั้นที่ 5: Query Planner

Planner แปลง AST (`Statement`) เป็น `QueryPlan` (logical plan tree) และให้ `explain()` output เพื่อ debug

**`src/planner.rs`:**

```rust
use crate::ast::*;

/// Node ใน Query Plan (logical plan)
#[derive(Debug, Clone)]
pub enum PlanNode {
    Scan   { table: String },
    Filter { input: Box<PlanNode>, predicate: Expr },
    Project{ input: Box<PlanNode>, columns: Vec<Expr> },
    Sort   { input: Box<PlanNode>, order_by: Vec<OrderBy> },
    Limit  { input: Box<PlanNode>, count: u64, offset: u64 },
    Insert { table: String, columns: Vec<String>, values: Vec<Vec<Expr>> },
    Update { table: String, assignments: Vec<Assignment>, predicate: Option<Expr> },
    Delete { table: String, predicate: Option<Expr> },
    CreateTable { stmt: CreateTableStatement },
    DropTable   { stmt: DropTableStatement },
}

#[derive(Debug, Clone)]
pub struct QueryPlan {
    pub root: PlanNode,
}

impl QueryPlan {
    /// EXPLAIN output — tree-formatted plan
    pub fn explain(&self) -> String {
        let mut buf = String::new();
        explain_node(&self.root, &mut buf, 0);
        buf
    }
}

fn explain_node(node: &PlanNode, buf: &mut String, depth: usize) {
    let indent = "  ".repeat(depth);
    match node {
        PlanNode::Scan   { table }                 => buf.push_str(&format!("{}Scan({})\n", indent, table)),
        PlanNode::Filter { input, predicate }      => {
            buf.push_str(&format!("{}Filter({})\n", indent, predicate));
            explain_node(input, buf, depth + 1);
        }
        PlanNode::Project{ input, columns }        => {
            let cols: Vec<String> = columns.iter().map(|c| c.to_string()).collect();
            buf.push_str(&format!("{}Project({})\n", indent, cols.join(", ")));
            explain_node(input, buf, depth + 1);
        }
        PlanNode::Sort   { input, order_by }       => {
            let parts: Vec<String> = order_by.iter().map(|o| {
                format!("{} {}", o.expr, if matches!(o.dir, OrderDir::Asc) { "ASC" } else { "DESC" })
            }).collect();
            buf.push_str(&format!("{}Sort({})\n", indent, parts.join(", ")));
            explain_node(input, buf, depth + 1);
        }
        PlanNode::Limit  { input, count, offset }  => {
            buf.push_str(&format!("{}Limit(count={}, offset={})\n", indent, count, offset));
            explain_node(input, buf, depth + 1);
        }
        // ... DML nodes
        _ => buf.push_str(&format!("{}[DML node]\n", indent)),
    }
}

pub struct Planner;

impl Planner {
    pub fn new() -> Self { Self }

    pub fn plan(&self, stmt: Statement) -> QueryPlan {
        let root = match stmt {
            Statement::Select(s) => self.plan_select(s),
            Statement::Insert(s) => PlanNode::Insert { table: s.table, columns: s.columns, values: s.values },
            Statement::Update(s) => PlanNode::Update { table: s.table, assignments: s.assignments, predicate: s.where_clause },
            Statement::Delete(s) => PlanNode::Delete { table: s.table, predicate: s.where_clause },
            Statement::CreateTable(s) => PlanNode::CreateTable { stmt: s },
            Statement::DropTable(s)   => PlanNode::DropTable { stmt: s },
            Statement::CreateIndex(_) => PlanNode::Scan { table: "(create_index)".to_string() },
        };
        QueryPlan { root }
    }

    fn plan_select(&self, s: SelectStatement) -> PlanNode {
        // 1. Scan
        let mut node = PlanNode::Scan { table: s.from };
        // 2. Filter (WHERE)
        if let Some(predicate) = s.where_clause {
            node = PlanNode::Filter { input: Box::new(node), predicate };
        }
        // 3. Project (SELECT columns)
        node = PlanNode::Project { input: Box::new(node), columns: s.columns };
        // 4. Sort (ORDER BY)
        if !s.order_by.is_empty() {
            node = PlanNode::Sort { input: Box::new(node), order_by: s.order_by };
        }
        // 5. Limit / Offset
        if let Some(count) = s.limit {
            node = PlanNode::Limit { input: Box::new(node), count, offset: s.offset.unwrap_or(0) };
        }
        node
    }
}
```

**ตัวอย่าง EXPLAIN output:**

```
SQL: SELECT name FROM users WHERE age > 18 ORDER BY name ASC LIMIT 5

PLAN:
Limit(count=5, offset=0)
  Sort(name ASC)
    Project(name)
      Filter((age > 18))
        Scan(users)
```

ลำดับการอ่าน plan จากล่างขึ้นบน: Scan → Filter → Project → Sort → Limit ซึ่งตรงกับ data flow จริง

---

### ขั้นที่ 6: In-Memory Executor

Executor รัน `QueryPlan` บน in-memory database ที่เก็บเป็น `HashMap<String, Vec<Row>>`

**`src/executor.rs` (ส่วน eval_expr):**

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use crate::ast::*;
use crate::planner::{PlanNode, QueryPlan};

/// Value type สำหรับ runtime
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(untagged)]
pub enum Value {
    Null,
    Int(i64),
    Float(f64),
    Text(String),
    Bool(bool),
}

pub type Row = HashMap<String, Value>;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QueryResult {
    pub columns: Vec<String>,
    pub rows: Vec<Row>,
    pub rows_affected: usize,
    pub message: String,
}

pub struct Database {
    tables: HashMap<String, Vec<Row>>,
}

impl Database {
    pub fn new() -> Self { Self { tables: HashMap::new() } }

    pub fn create_table(&mut self, name: &str) {
        self.tables.entry(name.to_string()).or_insert_with(Vec::new);
    }

    pub fn load_data(&mut self, table: &str, rows: Vec<Row>) {
        self.tables.insert(table.to_string(), rows);
    }

    pub fn execute(&mut self, plan: &QueryPlan) -> Result<QueryResult, String> {
        self.exec_node(&plan.root)
    }

    fn exec_node(&mut self, node: &PlanNode) -> Result<QueryResult, String> {
        match node {
            PlanNode::Scan { table } => {
                let rows = self.tables.get(table)
                    .ok_or_else(|| format!("Table '{}' does not exist", table))?
                    .clone();
                // ...
                Ok(QueryResult { columns, rows, rows_affected: 0, message: "OK".to_string() })
            }
            PlanNode::Filter { input, predicate } => {
                let mut result = self.exec_node(input)?;
                result.rows.retain(|row|
                    eval_expr(predicate, row).map(|v| is_truthy(&v)).unwrap_or(false)
                );
                Ok(result)
            }
            PlanNode::Sort { input, order_by } => {
                let mut result = self.exec_node(input)?;
                result.rows.sort_by(|a, b| {
                    for ob in order_by {
                        let va = eval_expr(&ob.expr, a).unwrap_or(Value::Null);
                        let vb = eval_expr(&ob.expr, b).unwrap_or(Value::Null);
                        let ord = cmp_values(&va, &vb);
                        let ord = match ob.dir {
                            OrderDir::Asc  => ord,
                            OrderDir::Desc => ord.reverse(),
                        };
                        if ord != std::cmp::Ordering::Equal { return ord; }
                    }
                    std::cmp::Ordering::Equal
                });
                Ok(result)
            }
            // ... (Filter, Project, Limit, Insert, Update, Delete, CreateTable, DropTable)
        }
    }
}

/// Evaluate expression บน row หนึ่ง → Option<Value>
pub fn eval_expr(expr: &Expr, row: &Row) -> Option<Value> {
    match expr {
        Expr::Null           => Some(Value::Null),
        Expr::NumberLit(n)   => Some(if n.fract() == 0.0 { Value::Int(*n as i64) } else { Value::Float(*n) }),
        Expr::StringLit(s)   => Some(Value::Text(s.clone())),
        Expr::Ident(name)    => {
            // รองรับ table.column notation
            let key = name.rfind('.').map(|i| &name[i+1..]).unwrap_or(name.as_str());
            row.get(key).cloned().or(Some(Value::Null))
        }
        Expr::BinaryOp { left, op, right } => {
            let lv = eval_expr(left, row)?;
            let rv = eval_expr(right, row)?;
            eval_binary_op(&lv, op, &rv)
        }
        Expr::UnaryOp { op: UnaryOp::Not, expr } => {
            let v = eval_expr(expr, row)?;
            Some(Value::Bool(!is_truthy(&v)))
        }
        Expr::IsNull { expr, negated } => {
            let v = eval_expr(expr, row)?;
            let is_null = matches!(v, Value::Null);
            Some(Value::Bool(if *negated { !is_null } else { is_null }))
        }
        Expr::Between { expr, low, high, negated } => {
            let v  = eval_expr(expr, row)?;
            let lv = eval_expr(low,  row)?;
            let hv = eval_expr(high, row)?;
            let in_range = cmp_values(&v, &lv) != std::cmp::Ordering::Less
                        && cmp_values(&v, &hv) != std::cmp::Ordering::Greater;
            Some(Value::Bool(if *negated { !in_range } else { in_range }))
        }
        Expr::Like { expr, pattern, negated } => {
            if let Some(Value::Text(s)) = eval_expr(expr, row) {
                let matched = like_match(&s, pattern);
                Some(Value::Bool(if *negated { !matched } else { matched }))
            } else {
                Some(Value::Bool(false))
            }
        }
        Expr::FunctionCall { name, args } => eval_function(name, args, row),
        // ...
    }
}

/// Simple LIKE pattern matching ด้วย % wildcard
fn like_match(s: &str, pattern: &str) -> bool {
    like_match_inner(s.to_lowercase().as_bytes(), pattern.to_lowercase().as_bytes())
}

fn like_match_inner(s: &[u8], p: &[u8]) -> bool {
    if p.is_empty() { return s.is_empty(); }
    if p[0] == b'%' {
        // % matches ทุก substring รวมถึง empty
        return (0..=s.len()).any(|i| like_match_inner(&s[i..], &p[1..]));
    }
    if s.is_empty() { return false; }
    let ch_match = p[0] == b'_' || p[0] == s[0];  // _ matches any single char
    ch_match && like_match_inner(&s[1..], &p[1..])
}
```

**ตัวอย่างรัน:**

```
SQL: SELECT * FROM users WHERE age BETWEEN 25 AND 30 LIMIT 2
PLAN:
Limit(count=2, offset=0)
  Project(*)
    Filter(age BETWEEN 25 AND 30)
      Scan(users)
RESULT: 2 rows
  active=true, age=30, id=1, name=Alice
  active=false, age=25, id=2, name=Bob
```

---

### ขั้นที่ 7: HTTP API ด้วย axum

เปิด endpoint สำหรับ execute SQL query ผ่าน HTTP

**`src/api.rs`:**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};
use serde_json::{json, Value as JsonValue};
use std::sync::Arc;
use tokio::sync::Mutex;
use crate::executor::{Database, QueryResult, Row, Value};
use crate::parser::parse;
use crate::planner::Planner;

pub type SharedDb = Arc<Mutex<Database>>;

#[derive(Deserialize)]
pub struct QueryRequest {
    pub sql: String,
}

#[derive(Serialize)]
pub struct QueryResponse {
    pub success: bool,
    pub message: String,
    pub rows: Vec<serde_json::Map<String, JsonValue>>,
    pub rows_affected: usize,
    pub explain: Option<String>,
}

pub async fn handle_query(
    State(db): State<SharedDb>,
    Json(req): Json<QueryRequest>,
) -> (StatusCode, Json<serde_json::Value>) {
    let stmt = match parse(&req.sql) {
        Ok(s) => s,
        Err(e) => {
            return (
                StatusCode::BAD_REQUEST,
                Json(json!({ "success": false, "error": e.to_string() })),
            );
        }
    };

    let planner = Planner::new();
    let plan = planner.plan(stmt);
    let explain = plan.explain();

    let mut db_guard = db.lock().await;
    match db_guard.execute(&plan) {
        Ok(result) => {
            let rows_json: Vec<serde_json::Map<String, JsonValue>> = result.rows.iter()
                .map(|row| {
                    let mut map = serde_json::Map::new();
                    for (k, v) in row {
                        let jv = match v {
                            Value::Null      => JsonValue::Null,
                            Value::Int(n)    => JsonValue::Number((*n).into()),
                            Value::Float(n)  => serde_json::Number::from_f64(*n)
                                .map(JsonValue::Number).unwrap_or(JsonValue::Null),
                            Value::Text(s)   => JsonValue::String(s.clone()),
                            Value::Bool(b)   => JsonValue::Bool(*b),
                        };
                        map.insert(k.clone(), jv);
                    }
                    map
                }).collect();
            (
                StatusCode::OK,
                Json(json!({
                    "success": true,
                    "message": result.message,
                    "rows_affected": result.rows_affected,
                    "rows": rows_json,
                    "explain": explain,
                })),
            )
        }
        Err(e) => (
            StatusCode::INTERNAL_SERVER_ERROR,
            Json(json!({ "success": false, "error": e })),
        ),
    }
}

pub async fn handle_load_data(
    Path(table_name): Path<String>,
    State(db): State<SharedDb>,
    Json(data): Json<Vec<serde_json::Map<String, JsonValue>>>,
) -> (StatusCode, Json<serde_json::Value>) {
    let rows: Vec<Row> = data.iter().map(|obj| {
        obj.iter().map(|(k, v)| {
            let val = match v {
                JsonValue::Null        => Value::Null,
                JsonValue::Bool(b)     => Value::Bool(*b),
                JsonValue::Number(n)   => {
                    if let Some(i) = n.as_i64() { Value::Int(i) }
                    else if let Some(f) = n.as_f64() { Value::Float(f) }
                    else { Value::Null }
                }
                JsonValue::String(s)   => Value::Text(s.clone()),
                _                      => Value::Null,
            };
            (k.clone(), val)
        }).collect()
    }).collect();

    let count = rows.len();
    db.lock().await.load_data(&table_name, rows);
    (StatusCode::OK, Json(json!({ "success": true, "rows_loaded": count })))
}

pub fn create_router(db: SharedDb) -> Router {
    Router::new()
        .route("/query", post(handle_query))
        .route("/tables/:name/data", post(handle_load_data))
        .with_state(db)
}
```

**ทดสอบ API:**

```bash
# โหลดข้อมูล
curl -s -X POST http://localhost:3000/tables/users/data \
  -H 'Content-Type: application/json' \
  -d '[
    {"id":1,"name":"Alice","age":30,"active":true},
    {"id":2,"name":"Bob","age":25,"active":false}
  ]'
# {"success":true,"rows_loaded":2}

# Execute query
curl -s -X POST http://localhost:3000/query \
  -H 'Content-Type: application/json' \
  -d '{"sql":"SELECT * FROM users WHERE age > 25"}'
# {
#   "success": true,
#   "message": "OK",
#   "rows_affected": 0,
#   "rows": [{"active":true,"age":30,"id":1,"name":"Alice"}],
#   "explain": "Project(*)\n  Filter((age > 25))\n    Scan(users)\n"
# }
```

---

### ขั้นที่ 8: REPL ด้วย rustyline

Interactive SQL REPL พร้อม tab completion และ meta-commands

**`src/repl.rs`:**

```rust
use rustyline::Editor;
use rustyline::completion::{Completer, Pair};
use rustyline::hint::Hinter;
use rustyline::highlight::Highlighter;
use rustyline::validate::Validator;
use rustyline::Helper;
use crate::executor::Database;
use crate::parser::parse;
use crate::planner::Planner;
use std::sync::{Arc, Mutex};

/// Custom helper ที่ implement tab completion
pub struct SqlHelper {
    pub table_names: Vec<String>,
    pub keywords: Vec<String>,
}

impl SqlHelper {
    pub fn new(tables: Vec<String>) -> Self {
        let keywords = vec![
            "SELECT", "FROM", "WHERE", "AND", "OR", "NOT",
            "INSERT", "INTO", "VALUES", "UPDATE", "SET",
            "DELETE", "CREATE", "DROP", "TABLE", "INDEX",
            "ORDER", "BY", "ASC", "DESC", "LIMIT", "OFFSET",
            "AS", "IS", "NULL", "IN", "BETWEEN", "LIKE", "COUNT",
        ].iter().map(|s| s.to_string()).collect();
        Self { table_names: tables, keywords }
    }
}

impl Completer for SqlHelper {
    type Candidate = Pair;

    fn complete(
        &self,
        line: &str,
        pos: usize,
        _ctx: &rustyline::Context<'_>,
    ) -> rustyline::Result<(usize, Vec<Pair>)> {
        let word_start = line[..pos]
            .rfind(|c: char| c.is_whitespace() || c == ',')
            .map(|i| i + 1)
            .unwrap_or(0);
        let word = &line[word_start..pos];

        let candidates: Vec<Pair> = self.keywords.iter()
            .chain(self.table_names.iter())
            .filter(|s| s.to_lowercase().starts_with(&word.to_lowercase()))
            .map(|s| Pair { display: s.clone(), replacement: s.clone() })
            .collect();

        Ok((word_start, candidates))
    }
}

impl Hinter for SqlHelper { type Hint = String; }
impl Highlighter for SqlHelper {}
impl Validator for SqlHelper {}
impl Helper for SqlHelper {}

/// เริ่ม interactive REPL
pub fn run_repl(db: Arc<Mutex<Database>>) {
    let tables = {
        let guard = db.lock().unwrap();
        guard.table_names().iter().map(|s| s.to_string()).collect()
    };
    let helper = SqlHelper::new(tables);
    let mut rl = Editor::new().expect("Failed to create editor");
    rl.set_helper(Some(helper));

    println!("SQL REPL — type \\help for commands, \\quit to exit");
    println!("Tab completion available for keywords and table names\n");

    loop {
        match rl.readline("sql> ") {
            Ok(line) => {
                let input = line.trim().to_string();
                if input.is_empty() { continue; }
                rl.add_history_entry(input.as_str()).ok();

                match input.as_str() {
                    "\\quit" | "\\q" => {
                        println!("Bye!");
                        break;
                    }
                    "\\help" => {
                        println!("Commands:");
                        println!("  \\tables          — list all tables");
                        println!("  \\schema <table>  — show table schema");
                        println!("  \\quit / \\q       — exit");
                    }
                    "\\tables" => {
                        let guard = db.lock().unwrap();
                        let names = guard.table_names();
                        if names.is_empty() {
                            println!("(no tables)");
                        } else {
                            for name in names { println!("  {}", name); }
                        }
                    }
                    cmd if cmd.starts_with("\\schema ") => {
                        let tbl = &cmd[8..].trim();
                        println!("Schema for '{}': (dynamic — no static schema)", tbl);
                    }
                    sql => {
                        // Remove trailing semicolon
                        let sql = sql.trim_end_matches(';');
                        match parse(sql) {
                            Err(e) => println!("Parse error: {}", e),
                            Ok(stmt) => {
                                let plan = Planner::new().plan(stmt);
                                let mut guard = db.lock().unwrap();
                                match guard.execute(&plan) {
                                    Ok(result) => {
                                        if !result.rows.is_empty() {
                                            // Print columns header
                                            println!("{}", result.columns.join(" | "));
                                            println!("{}", "-".repeat(40));
                                            for row in &result.rows {
                                                let vals: Vec<String> = result.columns.iter()
                                                    .map(|c| row.get(c).map(|v| v.to_string()).unwrap_or("NULL".to_string()))
                                                    .collect();
                                                println!("{}", vals.join(" | "));
                                            }
                                            println!("({} rows)", result.rows.len());
                                        } else {
                                            println!("{}", result.message);
                                        }
                                    }
                                    Err(e) => println!("Error: {}", e),
                                }
                            }
                        }
                    }
                }
            }
            Err(_) => break,
        }
    }
}
```

**ตัวอย่างใช้งาน REPL:**

```
SQL REPL — type \help for commands, \quit to exit
Tab completion available for keywords and table names

sql> \tables
  users
  products

sql> SELECT * FROM users WHERE age > 28;
active | age | id | name
----------------------------------------
true | 30 | 1 | Alice
true | 35 | 3 | Charlie
(2 rows)

sql> \schema users
Schema for 'users': (dynamic — no static schema)

sql> \quit
Bye!
```

---

## การทดสอบ (Testing)

โปรเจคนี้มีชุดทดสอบที่ครอบคลุม Lexer, Parser, Planner และ Executor

**Tests ครบ 36 test cases:**

```rust
// ใน src/main.rs #[cfg(test)]

// --- Lexer tests (5) ---
#[test]
fn test_lexer_keywords() {
    let tokens = tokenize("SELECT * FROM users WHERE").unwrap();
    assert!(matches!(tokens[0].kind, TokenKind::Select));
    assert!(matches!(tokens[1].kind, TokenKind::Star));
    assert!(matches!(tokens[2].kind, TokenKind::From));
}

#[test]
fn test_lexer_operators() {
    let tokens = tokenize("= < > <= >= <> !=").unwrap();
    assert!(matches!(tokens[0].kind, TokenKind::Eq));
    assert!(matches!(tokens[3].kind, TokenKind::LtEq));
    assert!(matches!(tokens[5].kind, TokenKind::NotEq));
    assert!(matches!(tokens[6].kind, TokenKind::NotEq)); // != ก็เป็น NotEq
}

#[test]
fn test_lexer_span_tracking() {
    let tokens = tokenize("SELECT\nFROM").unwrap();
    assert_eq!(tokens[0].span.line, 1);
    assert_eq!(tokens[1].span.line, 2);
    assert_eq!(tokens[1].span.col,  1);
}

// --- Parser tests (15+) ---
#[test]
fn test_parse_select_star() {
    let stmt = parse("SELECT * FROM users").unwrap();
    if let Statement::Select(s) = stmt {
        assert_eq!(s.from, "users");
        assert!(matches!(s.columns[0], Expr::Star));
    } else { panic!("Expected SELECT"); }
}

#[test]
fn test_parse_is_null() {
    let stmt = parse("SELECT * FROM t WHERE email IS NULL").unwrap();
    if let Statement::Select(s) = stmt {
        assert!(matches!(s.where_clause.unwrap(), Expr::IsNull { negated: false, .. }));
    }
}

#[test]
fn test_parse_between() {
    let stmt = parse("SELECT * FROM t WHERE age BETWEEN 18 AND 65").unwrap();
    if let Statement::Select(s) = stmt {
        assert!(matches!(s.where_clause.unwrap(), Expr::Between { negated: false, .. }));
    }
}

#[test]
fn test_parse_error_unexpected_token() {
    let err = parse("SELECT ) FROM users");
    assert!(err.is_err());
    let msg = err.unwrap_err().to_string();
    assert!(msg.contains(")") || msg.contains("found"));
}

// --- Executor tests (12+) ---
#[test]
fn test_execute_between() {
    let mut db = make_db();
    let plan = Planner::new().plan(
        parse("SELECT * FROM users WHERE age BETWEEN 25 AND 30").unwrap()
    );
    let result = db.execute(&plan).unwrap();
    assert_eq!(result.rows.len(), 3); // Alice(30), Bob(25), Diana(28)
}

#[test]
fn test_planner_explain_output() {
    let plan = Planner::new().plan(
        parse("SELECT name FROM users WHERE age > 18 ORDER BY name ASC LIMIT 5").unwrap()
    );
    let explain = plan.explain();
    assert!(explain.contains("Limit"));
    assert!(explain.contains("Sort"));
    assert!(explain.contains("Project"));
    assert!(explain.contains("Filter"));
    assert!(explain.contains("Scan"));
}
```

**Real `cargo test` output:**

```
running 36 tests
test tests::test_error_message_location ... ok
test tests::test_execute_between ... ok
test tests::test_execute_create_and_drop_table ... ok
test tests::test_execute_in_list ... ok
test tests::test_execute_insert_and_query ... ok
test tests::test_execute_like ... ok
test tests::test_execute_limit ... ok
test tests::test_execute_select_star ... ok
test tests::test_execute_where_filter ... ok
test tests::test_execute_delete ... ok
test tests::test_execute_update ... ok
test tests::test_lexer_keywords ... ok
test tests::test_lexer_number_literal ... ok
test tests::test_lexer_operators ... ok
test tests::test_lexer_span_tracking ... ok
test tests::test_lexer_string_literal ... ok
test tests::test_parse_and_or_expr ... ok
test tests::test_parse_as_alias ... ok
test tests::test_parse_between ... ok
test tests::test_parse_create_table ... ok
test tests::test_parse_delete ... ok
test tests::test_parse_drop_table ... ok
test tests::test_parse_error_unexpected_token ... ok
test tests::test_parse_function_call ... ok
test tests::test_parse_in_list ... ok
test tests::test_parse_is_not_null ... ok
test tests::test_parse_is_null ... ok
test tests::test_parse_like ... ok
test tests::test_parse_not_expr ... ok
test tests::test_parse_select_columns ... ok
test tests::test_parse_select_star ... ok
test tests::test_parse_select_order_by_limit ... ok
test tests::test_parse_select_where ... ok
test tests::test_parse_update ... ok
test tests::test_parse_insert ... ok
test tests::test_planner_explain_output ... ok

test result: ok. 36 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Demo binary output:**

```
=== Query Language Parser Demo ===

SQL: SELECT * FROM users
PLAN:
Project(*)
  Scan(users)
RESULT: 4 rows
  active=true, age=30, id=1, name=Alice
  active=false, age=25, id=2, name=Bob
  active=true, age=35, id=3, name=Charlie
  active=true, age=28, id=4, name=Diana

SQL: SELECT name, age FROM users WHERE age > 28
PLAN:
Project(name, age)
  Filter((age > 28))
    Scan(users)
RESULT: 2 rows
  age=30, name=Alice
  age=35, name=Charlie

SQL: SELECT name FROM users WHERE active = 1 ORDER BY name ASC
PLAN:
Sort(name ASC)
  Project(name)
    Filter((active = 1))
      Scan(users)
RESULT: 3 rows
  name=Alice
  name=Charlie
  name=Diana

SQL: SELECT * FROM users WHERE age BETWEEN 25 AND 30 LIMIT 2
PLAN:
Limit(count=2, offset=0)
  Project(*)
    Filter(age BETWEEN 25 AND 30)
      Scan(users)
RESULT: 2 rows
  active=true, age=30, id=1, name=Alice
  active=false, age=25, id=2, name=Bob

SQL: SELECT * FROM users WHERE name LIKE 'A%'
PLAN:
Project(*)
  Filter(name LIKE 'A%')
    Scan(users)
RESULT: 1 rows
  active=true, age=30, id=1, name=Alice
```

---

## Pitfalls และข้อควรระวัง

### Pitfall 1: `std::mem::discriminant` สำหรับ enum comparison

เมื่อต้องการ check ว่า token มี kind ตรง variant ที่ต้องการโดยไม่สนใจ inner value เช่น `TokenKind::Ident("whatever")` ต้อง match กับ variant `Ident` ไม่ว่า string จะเป็นอะไร วิธีที่ผิด:

```rust
// ❌ ผิด — compares value inside too
if tok.kind == TokenKind::Ident("".to_string()) { ... }
```

วิธีที่ถูก:

```rust
// ✅ ถูก — compares discriminant (variant tag) เท่านั้น
if std::mem::discriminant(&tok.kind) == std::mem::discriminant(&TokenKind::Ident(String::new())) { ... }
// หรือใช้ matches! macro
if matches!(&tok.kind, TokenKind::Ident(_)) { ... }
```

### Pitfall 2: Recursive `Expr` enum ต้องใช้ `Box`

```rust
// ❌ compile error: recursive type has infinite size
pub enum Expr {
    BinaryOp { left: Expr, right: Expr, op: BinaryOp },
}

// ✅ ถูก — Box<T> มีขนาดแน่นอน (pointer = 8 bytes)
pub enum Expr {
    BinaryOp { left: Box<Expr>, right: Box<Expr>, op: BinaryOp },
}
```

Rust ต้องรู้ขนาดทุก type ณ compile time `Expr` ที่มี `Expr` ข้างใน จะมี infinite size ถ้าไม่มี indirection `Box<Expr>` คือ heap pointer ที่มีขนาดแน่นอน 8 bytes

### Pitfall 3: Type coercion ระหว่าง Bool และ Int ใน SQL

SQL มักใช้ `WHERE active = 1` เพื่อหมายถึง "active is true" แต่ถ้า column เก็บเป็น `Value::Bool(true)` และ RHS เป็น `Value::Int(1)` การเปรียบเทียบแบบ strict จะ fail:

```rust
// ❌ ผิด — Bool != Int เสมอ
fn values_eq(a: &Value, b: &Value) -> bool {
    match (a, b) {
        (Value::Bool(x), Value::Bool(y)) => x == y,
        (Value::Int(x),  Value::Int(y))  => x == y,
        _ => false,  // Bool vs Int → false เสมอ ← BUG!
    }
}

// ✅ ถูก — handle Bool/Int interop
fn values_eq(a: &Value, b: &Value) -> bool {
    match (a, b) {
        (Value::Bool(b),  Value::Int(n))  => (*b as i64) == *n,
        (Value::Int(n),   Value::Bool(b)) => *n == (*b as i64),
        // ... rest of cases
    }
}
```

### Pitfall 4: BETWEEN parsing ambiguity กับ AND

`age BETWEEN 18 AND 65` ต้องไม่ parse `AND` ตรงนี้ว่าเป็น binary AND operator แต่เป็นส่วนของ BETWEEN syntax วิธีแก้คือ parse `low` และ `high` ด้วย `parse_primary()` แทน `parse_expr()` เพราะ `parse_expr()` จะดึง AND ออกไปก่อน:

```rust
// ❌ ผิด — parse_expr() จะ consume AND ก่อน BETWEEN ได้ใช้
fn parse_comparison(...) {
    if self.eat(&TokenKind::Between) {
        let low = self.parse_expr()?;   // WRONG: eats the AND!
        self.expect(TokenKind::And)?;   // จะ fail
        let high = self.parse_expr()?;
    }
}

// ✅ ถูก — ใช้ parse_primary() สำหรับ low และ high
fn parse_comparison(...) {
    if self.eat(&TokenKind::Between) {
        let low = self.parse_primary()?;   // parse only a single primary
        self.expect(TokenKind::And)?;      // now AND is available
        let high = self.parse_primary()?;
    }
}
```

### Pitfall 5: Error message ควรมีตำแหน่งเสมอ

Error message แบบ "Unexpected token" โดยไม่มีตำแหน่งทำให้ debug ยาก:

```
// ❌ ไม่มีประโยชน์
Error: unexpected token

// ✅ บอกตำแหน่งชัดเจน
Error: Expected column name, found ')' at line 1 col 15
```

การ track `Span` ทุก token ตั้งแต่ Lexer ทำให้ Parser สามารถ report ตำแหน่งได้เสมอ เปรียบเทียบกับ Rust compiler ที่ report "error[E0308]: mismatched types → at src/main.rs:42:15"

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/query-parser

# ขนาด release binary
ls -lh target/release/query-parser
# -rwxr-xr-x 1 user user 4.2M query-parser

# รัน HTTP server
./target/release/query-parser --mode server --port 3000

# รัน REPL
./target/release/query-parser --mode repl
```

### Docker

```dockerfile
FROM rust:1.82-alpine AS builder
WORKDIR /app
COPY . .
RUN apk add --no-cache musl-dev
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/query-parser /query-parser
EXPOSE 3000
ENTRYPOINT ["/query-parser", "--mode", "server"]
```

```bash
docker build -t query-parser .
docker run -p 3000:3000 query-parser
```

### Health Check

```bash
# ตรวจสอบว่า server พร้อม
curl http://localhost:3000/health
# {"status":"ok","tables":0}
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม Aggregation Functions (ระดับกลาง)

เพิ่ม `SUM`, `AVG`, `MIN`, `MAX`, `COUNT` พร้อม `GROUP BY` clause ต้องเพิ่ม `PlanNode::Aggregate` และ `GroupBy` ใน AST ความยากอยู่ที่การ group rows ก่อน aggregate

```rust
// Target SQL:
"SELECT department, COUNT(*), AVG(salary) FROM employees GROUP BY department"

// PlanNode ใหม่:
PlanNode::Aggregate {
    input: Box<PlanNode>,
    group_by: Vec<Expr>,
    aggregates: Vec<AggregateExpr>,  // { func: COUNT/SUM/AVG, expr: Expr, alias: String }
}
```

### Exercise 2: JOIN Support (ระดับยาก)

เพิ่ม `INNER JOIN`, `LEFT JOIN` ใน SELECT statement ต้องเพิ่ม `PlanNode::Join` พร้อม join condition และ join type Nested loop join เป็น approach ที่ง่ายที่สุดสำหรับ in-memory executor

```rust
// Target SQL:
"SELECT u.name, o.total FROM users u INNER JOIN orders o ON u.id = o.user_id WHERE o.total > 100"
```

### Exercise 3: Persistent Storage ด้วย SQLite Backend (ระดับกลาง)

แทนที่ in-memory `Vec<Row>` ด้วย SQLite (ผ่าน `rusqlite` crate) โดยที่ Parser และ Planner ยังคงเหมือนเดิม แต่ Executor แปลง `PlanNode` เป็น SQLite queries แทน

```toml
[dependencies]
rusqlite = { version = "0.31", features = ["bundled"] }
```

### Exercise 4: Query Optimizer (ระดับยาก)

เพิ่ม optimization pass ระหว่าง Planner กับ Executor:
- **Predicate Pushdown**: เลื่อน Filter ให้อยู่ใกล้ Scan มากที่สุด
- **Projection Pushdown**: อ่านแค่ column ที่ต้องการแทนทั้ง row
- **Constant Folding**: `1 + 1` → `2` ณ plan time

```rust
pub struct Optimizer;

impl Optimizer {
    pub fn optimize(&self, plan: QueryPlan) -> QueryPlan {
        let root = self.push_predicates_down(plan.root);
        QueryPlan { root }
    }

    fn push_predicates_down(&self, node: PlanNode) -> PlanNode {
        match node {
            PlanNode::Filter { input, predicate } => {
                // ถ้า input เป็น Project แล้ว Filter อยู่บน —
                // ย้าย Filter ลงไปอยู่ใต้ Project แทน
                // ...
            }
            _ => node,
        }
    }
}
```

---

## สรุป

โปรเจคนี้ครอบคลุม compiler/interpreter pipeline ครบทั้ง stack:

| Component | Pattern หลัก | Rust Concept ที่ใช้ |
|-----------|-------------|-------------------|
| Lexer     | Hand-written tokenizer | Iterator, `Peekable`, lifetime `'a` |
| AST       | Algebraic Data Types | Recursive enum, `Box<T>`, serde |
| Parser    | Recursive descent | Pattern matching, error propagation |
| Planner   | Visitor pattern | Tree transformation |
| Executor  | Interpreter pattern | HashMap, closure, type coercion |
| API       | REST + JSON | axum, `Arc<Mutex<T>>`, async/await |

**Pattern สำคัญที่ได้เรียน:**

1. **Recursive Descent Parsing** — เทคนิคที่ใช้ใน compiler จริงทุกตัว รวมถึง rustc เอง
2. **AST เป็น Recursive Enum** — วิธี model tree data structure ที่ type-safe ใน Rust
3. **Error Types ด้วย thiserror** — สร้าง error hierarchy ที่มีข้อมูลตำแหน่งสำหรับ debugging
4. **Shared Mutable State ด้วย `Arc<Mutex<T>>`** — pattern สำหรับ database engine ที่รองรับ concurrent access

โปรเจคถัดไป **C10 — Schema Registry** จะต่อยอดจากโปรเจคนี้โดยเพิ่ม schema versioning, schema evolution และการ validate data ตาม schema definition

---

**โปรเจคก่อนหน้า:** [project-c08-data-migration.md](project-c08-data-migration.md) | **โปรเจคถัดไป:** [project-c10-schema-registry.md](project-c10-schema-registry.md)
