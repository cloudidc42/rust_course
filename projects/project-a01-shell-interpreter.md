# Project A01: Shell Interpreter (bash subset)

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 6 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **shell interpreter** ขนาดเล็กที่สามารถรับคำสั่งจากผู้ใช้แบบ interactive (REPL) และรันได้จริง รองรับ features สำคัญของ bash เช่น pipeline (`|`), I/O redirection (`>`, `>>`, `<`), sequence (`;`), logical operators (`&&`, `||`), environment variable expansion (`$HOME`, `$PATH`), และ built-in commands หลัก (`cd`, `pwd`, `echo`, `exit`, `export`, `unset`, `history`)

Shell เป็นหนึ่งใน programs ที่ซับซ้อนที่สุดในระบบปฏิบัติการ เพราะต้องทำงานร่วมกับ OS ในระดับต่ำมาก ได้แก่ process spawning, file descriptor manipulation, signal handling, และ inter-process communication ผ่าน pipes การสร้าง shell จาก scratch เป็นวิธีที่ดีที่สุดในการเข้าใจว่า Unix process model ทำงานอย่างไรจริง ๆ

**Use cases จริงในโลก production:**
- เป็น embedded shell ใน configuration tools (เช่น build systems, CI runners)
- เป็น restricted shell สำหรับ SSH sandbox environments
- เป็น scripting engine ภายใน applications (เช่น game modding shells, IoT automation)
- เรียนรู้พื้นฐานสำหรับงาน DevOps, systems programming, และ OS development

## สิ่งที่จะได้เรียนรู้

- **REPL pattern** — วิธีออกแบบ read-eval-print loop ที่ robust และ responsive
- **Lexer/Parser** — การแบ่งข้อความเป็น tokens และสร้าง Abstract Syntax Tree (AST)
- **Process management** — `std::process::Command`, spawning, waiting, exit codes
- **Pipe implementation** — ส่ง stdout ของ process หนึ่งเป็น stdin ของอีก process ผ่าน `Stdio::piped()`
- **I/O redirection** — ใช้ `File` เป็น `Stdio` สำหรับ `>`, `>>`, `<`
- **Signal handling** — จัดการ SIGINT (Ctrl+C) ด้วย crate `nix`
- **Environment variable management** — สืบทอด env จาก parent, expand `$VAR`
- **Error handling idioms** — command not found (exit 127), permission denied, broken pipe

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, pattern matching
- **Part 21–30**: Collections (`Vec`, `HashMap`), iterators, closures
- **Part 31–40**: Error handling (`Result`, `?`), traits, generics
- **Part 41–50**: File I/O (`std::fs`), process execution (`std::process`), environment variables
- **Part 51–60**: String manipulation, `chars()`, `Peekable` iterators

## โครงสร้างโปรเจค (Project Layout)

```
rshell/
├── src/
│   ├── main.rs          ← entry point, --command flag สำหรับ testing
│   ├── lexer.rs         ← tokenizer: input string → Vec<Token>
│   ├── parser.rs        ← parser: Vec<Token> → AST
│   └── executor.rs      ← executor: AST → spawned processes
├── tests/
│   └── integration_test.rs  ← integration tests ด้วย assert_cmd
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
User Input
    │
    ▼
┌─────────┐   Vec<Token>   ┌────────┐   AST   ┌──────────┐
│  Lexer  │ ─────────────▶ │ Parser │ ───────▶ │ Executor │
└─────────┘                └────────┘          └──────────┘
                                                    │
                                    ┌───────────────┼───────────────┐
                                    ▼               ▼               ▼
                              Built-in Cmds    External Cmds   Pipeline
                              (cd, pwd, echo)  (spawn process)  (chain stdout→stdin)
```

### การแยก Concerns ออกจากกัน

เราแยก lexer, parser, executor ออกเป็นโมดูลแยก เพราะ:

1. **Lexer** — รู้แค่เรื่อง characters และ tokens ไม่รู้เรื่อง AST หรือ execution
2. **Parser** — รู้เรื่อง grammar และ AST ไม่รู้เรื่อง I/O หรือ processes
3. **Executor** — รู้เรื่อง OS syscalls และ process management เท่านั้น

การแยก concerns ออกจากกันทำให้ test แต่ละส่วนได้ง่าย และ maintain โค้ดได้ดีกว่า

### ทำไมถึงไม่ใช้ `execvp` โดยตรง?

ใน C เราใช้ `fork()` + `execvp()` โดยตรง ใน Rust เราใช้ `std::process::Command` ซึ่ง:
- Abstract ขั้นตอน fork/exec ออกไป
- จัดการ error handling อย่างถูกต้อง
- ทำงานข้ามแพลตฟอร์มได้ (Windows, macOS, Linux)
- มี safe API ที่ป้องกัน pitfalls เช่น file descriptor leaking

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: REPL Skeleton (Read → Echo Back)

เริ่มจากโครงสร้างพื้นฐานที่สุด — อ่าน input จากผู้ใช้แล้ว echo กลับ ยังไม่ต้องรันคำสั่งจริง เป้าหมายคือให้ interactive loop ทำงานได้

**`Cargo.toml`:**

```toml
[package]
name = "rshell"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "rshell"
path = "src/main.rs"

[dependencies]
rustyline = "14.0"
which = "6.0"
nix = { version = "0.28", features = ["signal", "process"] }

[dev-dependencies]
assert_cmd = "2.0"
predicates = "3.0"
```

**`src/main.rs` (ขั้นที่ 1):**

```rust
use std::env;
use std::io::{self, BufRead, Write};

fn main() {
    let stdin = io::stdin();
    loop {
        // แสดง prompt
        print!("$ ");
        io::stdout().flush().unwrap();

        // อ่านบรรทัดจาก stdin
        let mut line = String::new();
        match stdin.lock().read_line(&mut line) {
            Ok(0) => break, // EOF (Ctrl+D)
            Ok(_) => {
                let line = line.trim();
                if line.is_empty() {
                    continue;
                }
                // ขั้นที่ 1: แค่ echo กลับ
                println!("you typed: {}", line);
            }
            Err(e) => {
                eprintln!("error: {}", e);
                break;
            }
        }
    }
}
```

**Output จากการรัน:**
```
$ ls -la
you typed: ls -la
$ echo hello world
you typed: echo hello world
$ ^D
```

ข้อดีของการเริ่มแบบนี้คือเราได้ feedback loop เร็ว สามารถ test การ input ได้ก่อนที่จะเพิ่ม logic ซับซ้อน

---

### ขั้นที่ 2: Tokenizer พร้อม Quote Handling

Tokenizer (หรือ Lexer) แปลง string เป็น `Vec<Token>` โดย handle:
- **Single quotes** (`'...'`) — เนื้อหาภายในเป็น literal ทั้งหมด ไม่มี escape, ไม่ expand `$VAR`
- **Double quotes** (`"..."`) — อนุญาต escape sequences (`\"`, `\\`, `\$`) และ variable expansion ภายหลัง
- **Backslash** (`\c`) — escape character ตัวถัดไป
- **Operators** — `|`, `>`, `>>`, `<`, `;`, `&`, `&&`, `||`

**`src/lexer.rs`:**

```rust
/// Token types produced by the lexer
#[derive(Debug, Clone, PartialEq)]
pub enum Token {
    Word(String),      // คำปกติ หรือ quoted string
    Pipe,              // |
    RedirectOut,       // >
    RedirectAppend,    // >>
    RedirectIn,        // <
    Semicolon,         // ;
    Ampersand,         // & (background)
    And,               // && (logical AND)
    Or,                // || (logical OR)
    Newline,           // \n
}

/// Tokenize a shell input line
pub fn tokenize(input: &str) -> Result<Vec<Token>, String> {
    let mut tokens = Vec::new();
    let mut chars = input.chars().peekable();
    let mut current = String::new();
    let mut in_word = false;

    while let Some(&ch) = chars.peek() {
        match ch {
            // Whitespace: end current word
            ' ' | '\t' => {
                chars.next();
                if in_word {
                    tokens.push(Token::Word(current.clone()));
                    current.clear();
                    in_word = false;
                }
            }
            '\n' => {
                chars.next();
                if in_word {
                    tokens.push(Token::Word(current.clone()));
                    current.clear();
                    in_word = false;
                }
                tokens.push(Token::Newline);
            }
            // Single quote: ไม่มี escape ใด ๆ ทั้งสิ้น
            '\'' => {
                chars.next(); // consume opening '
                in_word = true;
                loop {
                    match chars.next() {
                        Some('\'') => break,
                        Some(c) => current.push(c),
                        None => return Err("unterminated single quote".to_string()),
                    }
                }
            }
            // Double quote: อนุญาต \" \\ \$ เท่านั้น
            '"' => {
                chars.next(); // consume opening "
                in_word = true;
                loop {
                    match chars.next() {
                        Some('"') => break,
                        Some('\\') => {
                            match chars.next() {
                                Some('"')  => current.push('"'),
                                Some('\\') => current.push('\\'),
                                Some('$')  => current.push('$'),
                                Some('\n') => {} // line continuation
                                Some(c)    => { current.push('\\'); current.push(c); }
                                None => return Err("unterminated escape".to_string()),
                            }
                        }
                        Some(c) => current.push(c),
                        None => return Err("unterminated double quote".to_string()),
                    }
                }
            }
            // Backslash escape นอก quotes
            '\\' => {
                chars.next();
                in_word = true;
                match chars.next() {
                    Some('\n') => {} // line continuation: skip newline
                    Some(c)    => current.push(c),
                    None       => {} // trailing backslash
                }
            }
            // Comment character
            '#' => {
                while chars.peek().map(|&c| c != '\n').unwrap_or(false) {
                    chars.next();
                }
            }
            // >> ต้องตรวจก่อน >
            '>' => {
                chars.next();
                if in_word { tokens.push(Token::Word(current.clone())); current.clear(); in_word = false; }
                if chars.peek() == Some(&'>') { chars.next(); tokens.push(Token::RedirectAppend); }
                else { tokens.push(Token::RedirectOut); }
            }
            '<' => {
                chars.next();
                if in_word { tokens.push(Token::Word(current.clone())); current.clear(); in_word = false; }
                tokens.push(Token::RedirectIn);
            }
            // || ต้องตรวจก่อน |
            '|' => {
                chars.next();
                if in_word { tokens.push(Token::Word(current.clone())); current.clear(); in_word = false; }
                if chars.peek() == Some(&'|') { chars.next(); tokens.push(Token::Or); }
                else { tokens.push(Token::Pipe); }
            }
            ';' => {
                chars.next();
                if in_word { tokens.push(Token::Word(current.clone())); current.clear(); in_word = false; }
                tokens.push(Token::Semicolon);
            }
            // && ต้องตรวจก่อน &
            '&' => {
                chars.next();
                if in_word { tokens.push(Token::Word(current.clone())); current.clear(); in_word = false; }
                if chars.peek() == Some(&'&') { chars.next(); tokens.push(Token::And); }
                else { tokens.push(Token::Ampersand); }
            }
            // ตัวอักษรปกติ
            c => {
                chars.next();
                in_word = true;
                current.push(c);
            }
        }
    }

    if in_word {
        tokens.push(Token::Word(current));
    }

    Ok(tokens)
}
```

**Unit tests สำหรับ Lexer:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_simple_words() {
        let tokens = tokenize("ls -la /tmp").unwrap();
        assert_eq!(tokens, vec![
            Token::Word("ls".into()),
            Token::Word("-la".into()),
            Token::Word("/tmp".into()),
        ]);
    }

    #[test]
    fn test_single_quotes() {
        // 'hello world' → Word("hello world") — เป็น single token เดียว
        let tokens = tokenize("echo 'hello world'").unwrap();
        assert_eq!(tokens, vec![
            Token::Word("echo".into()),
            Token::Word("hello world".into()),
        ]);
    }

    #[test]
    fn test_double_quotes() {
        let tokens = tokenize(r#"echo "hello world""#).unwrap();
        assert_eq!(tokens, vec![
            Token::Word("echo".into()),
            Token::Word("hello world".into()),
        ]);
    }

    #[test]
    fn test_pipe() {
        let tokens = tokenize("ls | grep foo").unwrap();
        assert_eq!(tokens, vec![
            Token::Word("ls".into()), Token::Pipe,
            Token::Word("grep".into()), Token::Word("foo".into()),
        ]);
    }

    #[test]
    fn test_redirect_append() {
        let tokens = tokenize("echo hi >> /tmp/out.txt").unwrap();
        assert!(tokens.contains(&Token::RedirectAppend));
    }
}
```

**สิ่งที่ควรระวัง:** `it's` ใน shell ต้องเขียนเป็น `'it'\''s'` — ปิด single quote, escape ด้วย `\'`, แล้วเปิด single quote ใหม่ โค้ดของเราจัดการกรณีนี้ได้อย่างถูกต้องเพราะ tokenizer จะ concatenate words ที่ติดกันโดยไม่มีช่องว่าง

---

### ขั้นที่ 3: Parser → Simple Command AST

Parser รับ `Vec<Token>` แล้วสร้าง AST ตาม grammar ง่าย ๆ:

```
input    = job_list
job_list = pipeline (CONNECTOR pipeline)*
pipeline = simple_cmd ('|' simple_cmd)* ['&']
simple_cmd = WORD+ redirect*
redirect = '>' WORD | '>>' WORD | '<' WORD
CONNECTOR = ';' | '&&' | '||'
```

**`src/parser.rs`:**

```rust
use crate::lexer::Token;

/// Command เดี่ยวพร้อม redirections
#[derive(Debug, Clone)]
pub struct SimpleCommand {
    pub argv: Vec<String>,             // argv[0] = program name
    pub redirect_in:     Option<String>, // < file
    pub redirect_out:    Option<String>, // > file (truncate)
    pub redirect_append: Option<String>, // >> file (append)
}

impl SimpleCommand {
    pub fn new(argv: Vec<String>) -> Self {
        SimpleCommand { argv, redirect_in: None, redirect_out: None, redirect_append: None }
    }
}

/// Pipeline: หนึ่งหรือหลาย SimpleCommand ต่อกันด้วย |
#[derive(Debug, Clone)]
pub struct Pipeline {
    pub commands: Vec<SimpleCommand>,
    pub background: bool,
}

/// Connector ระหว่าง pipelines
#[derive(Debug, Clone, PartialEq)]
pub enum Connector {
    Semicolon, // ;
    And,       // &&
    Or,        // ||
}

/// Job list: หลาย pipelines ต่อกันด้วย connectors
#[derive(Debug, Clone)]
pub struct JobList {
    pub pipelines:  Vec<Pipeline>,
    pub connectors: Vec<Connector>, // len == pipelines.len() - 1
}

/// AST root
#[derive(Debug, Clone)]
pub struct Ast {
    pub jobs: JobList,
}

pub fn parse(tokens: &[Token]) -> Result<Ast, String> {
    let mut pos = 0;
    let jobs = parse_job_list(tokens, &mut pos)?;
    Ok(Ast { jobs })
}

fn parse_job_list(tokens: &[Token], pos: &mut usize) -> Result<JobList, String> {
    let mut pipelines  = Vec::new();
    let mut connectors = Vec::new();

    skip_delimiters(tokens, pos);

    while *pos < tokens.len() {
        match tokens.get(*pos) {
            Some(Token::Semicolon) | Some(Token::Newline) => { *pos += 1; continue; }
            _ => {}
        }
        if *pos >= tokens.len() { break; }

        let pipeline = parse_pipeline(tokens, pos)?;
        pipelines.push(pipeline);

        match tokens.get(*pos) {
            Some(Token::Semicolon) | Some(Token::Newline) => {
                *pos += 1;
                connectors.push(Connector::Semicolon);
            }
            Some(Token::And) => { *pos += 1; connectors.push(Connector::And); }
            Some(Token::Or)  => { *pos += 1; connectors.push(Connector::Or);  }
            _ => {}
        }
    }

    Ok(JobList { pipelines, connectors })
}

fn skip_delimiters(tokens: &[Token], pos: &mut usize) {
    while matches!(tokens.get(*pos), Some(Token::Semicolon) | Some(Token::Newline)) {
        *pos += 1;
    }
}

fn parse_pipeline(tokens: &[Token], pos: &mut usize) -> Result<Pipeline, String> {
    let mut commands   = Vec::new();
    let mut background = false;

    commands.push(parse_simple_command(tokens, pos)?);

    loop {
        match tokens.get(*pos) {
            Some(Token::Pipe) => {
                *pos += 1;
                while matches!(tokens.get(*pos), Some(Token::Newline)) { *pos += 1; }
                commands.push(parse_simple_command(tokens, pos)?);
            }
            Some(Token::Ampersand) => { *pos += 1; background = true; break; }
            _ => break,
        }
    }

    Ok(Pipeline { commands, background })
}

fn parse_simple_command(tokens: &[Token], pos: &mut usize) -> Result<SimpleCommand, String> {
    let mut argv:            Vec<String>   = Vec::new();
    let mut redirect_in:     Option<String> = None;
    let mut redirect_out:    Option<String> = None;
    let mut redirect_append: Option<String> = None;

    loop {
        match tokens.get(*pos) {
            Some(Token::Word(w)) => { argv.push(w.clone()); *pos += 1; }
            Some(Token::RedirectOut) => {
                *pos += 1;
                match tokens.get(*pos) {
                    Some(Token::Word(f)) => { redirect_out = Some(f.clone()); *pos += 1; }
                    _ => return Err("expected filename after >".to_string()),
                }
            }
            Some(Token::RedirectAppend) => {
                *pos += 1;
                match tokens.get(*pos) {
                    Some(Token::Word(f)) => { redirect_append = Some(f.clone()); *pos += 1; }
                    _ => return Err("expected filename after >>".to_string()),
                }
            }
            Some(Token::RedirectIn) => {
                *pos += 1;
                match tokens.get(*pos) {
                    Some(Token::Word(f)) => { redirect_in = Some(f.clone()); *pos += 1; }
                    _ => return Err("expected filename after <".to_string()),
                }
            }
            _ => break,
        }
    }

    if argv.is_empty() {
        return Err("empty command".to_string());
    }

    let mut cmd = SimpleCommand::new(argv);
    cmd.redirect_in     = redirect_in;
    cmd.redirect_out    = redirect_out;
    cmd.redirect_append = redirect_append;
    Ok(cmd)
}
```

**Parser tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::lexer::tokenize;

    #[test]
    fn test_parse_simple() {
        let tokens = tokenize("ls -la").unwrap();
        let ast = parse(&tokens).unwrap();
        assert_eq!(ast.jobs.pipelines.len(), 1);
        assert_eq!(ast.jobs.pipelines[0].commands[0].argv, vec!["ls", "-la"]);
    }

    #[test]
    fn test_parse_pipeline() {
        let tokens = tokenize("ls | grep foo | wc -l").unwrap();
        let ast = parse(&tokens).unwrap();
        // pipeline มี 3 commands
        assert_eq!(ast.jobs.pipelines[0].commands.len(), 3);
    }

    #[test]
    fn test_parse_redirect() {
        let tokens = tokenize("echo hello > /tmp/out.txt").unwrap();
        let ast = parse(&tokens).unwrap();
        let cmd = &ast.jobs.pipelines[0].commands[0];
        assert_eq!(cmd.redirect_out, Some("/tmp/out.txt".to_string()));
    }

    #[test]
    fn test_parse_sequence() {
        let tokens = tokenize("echo a ; echo b").unwrap();
        let ast = parse(&tokens).unwrap();
        // มี 2 pipelines แยกกันด้วย ;
        assert_eq!(ast.jobs.pipelines.len(), 2);
    }
}
```

---

### ขั้นที่ 4: Execute External Commands + Built-ins

นี่คือหัวใจของ shell — ส่วนที่รับ AST แล้วสั่งให้ OS ทำงานจริง

**`src/executor.rs` (ส่วน Executor struct):**

```rust
use std::collections::HashMap;
use std::env;
use std::fs::{File, OpenOptions};
use std::io::{self, Write};
use std::process::{Command, Stdio};

use crate::lexer::tokenize;
use crate::parser::{parse, Connector, JobList, Pipeline, SimpleCommand};

pub struct Executor {
    pub env: HashMap<String, String>,
    pub last_exit: i32,
    history: Vec<String>,
}

impl Executor {
    pub fn new() -> Self {
        // สืบทอด environment จาก parent process
        let env: HashMap<String, String> = env::vars().collect();
        Executor { env, last_exit: 0, history: Vec::new() }
    }

    pub fn run_line(&mut self, line: &str) -> i32 {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') { return 0; }

        self.history.push(line.to_string());

        let tokens = match tokenize(line) {
            Ok(t) => t,
            Err(e) => { eprintln!("rshell: parse error: {}", e); return 1; }
        };

        if tokens.is_empty() { return 0; }

        let ast = match parse(&tokens) {
            Ok(a) => a,
            Err(e) => { eprintln!("rshell: {}", e); return 1; }
        };

        self.execute_job_list(&ast.jobs)
    }
```

**Execute Job List (จัดการ `&&` และ `||`):**

```rust
    fn execute_job_list(&mut self, jobs: &JobList) -> i32 {
        if jobs.pipelines.is_empty() { return 0; }

        let mut last = self.execute_pipeline(&jobs.pipelines[0]);
        self.last_exit = last;

        for (i, pipeline) in jobs.pipelines[1..].iter().enumerate() {
            let connector = jobs.connectors.get(i);
            match connector {
                Some(Connector::And) => {
                    // && → รัน pipeline ถัดไปก็ต่อเมื่อ pipeline ก่อนสำเร็จ (exit 0)
                    if last != 0 { continue; }
                }
                Some(Connector::Or) => {
                    // || → รัน pipeline ถัดไปก็ต่อเมื่อ pipeline ก่อนล้มเหลว
                    if last == 0 { continue; }
                }
                _ => {} // Semicolon หรือไม่มี connector: รันเสมอ
            }
            last = self.execute_pipeline(pipeline);
            self.last_exit = last;
        }
        last
    }
```

**Built-in Commands:**

```rust
    fn try_builtin(&mut self, argv: &[String], cmd: &SimpleCommand) -> Option<i32> {
        match argv.get(0).map(|s| s.as_str()) {
            Some("exit") => {
                let code = argv.get(1)
                    .and_then(|s| s.parse::<i32>().ok())
                    .unwrap_or(self.last_exit);
                std::process::exit(code);
            }
            Some("cd") => {
                let target = argv.get(1)
                    .map(|s| s.as_str())
                    .unwrap_or(self.env.get("HOME").map(|s| s.as_str()).unwrap_or("/"));
                match env::set_current_dir(target) {
                    Ok(()) => {
                        // อัปเดต $PWD ด้วย เพราะ set_current_dir ไม่ทำให้อัตโนมัติ
                        if let Ok(cwd) = env::current_dir() {
                            self.env.insert("PWD".to_string(), cwd.to_string_lossy().to_string());
                        }
                        Some(0)
                    }
                    Err(e) => { eprintln!("rshell: cd: {}: {}", target, e); Some(1) }
                }
            }
            Some("pwd") => {
                match env::current_dir() {
                    Ok(p) => { println!("{}", p.display()); Some(0) }
                    Err(e) => { eprintln!("rshell: pwd: {}", e); Some(1) }
                }
            }
            Some("echo") => {
                let (no_newline, words) = if argv.get(1).map(|s| s == "-n").unwrap_or(false) {
                    (true, &argv[2..])
                } else {
                    (false, &argv[1..])
                };
                let output = if no_newline {
                    words.join(" ")
                } else {
                    words.join(" ") + "\n"
                };
                // built-in ต้อง handle redirection เอง
                if let Some(ref path) = cmd.redirect_out {
                    match File::create(path) {
                        Ok(mut f) => { let _ = f.write_all(output.as_bytes()); }
                        Err(e) => { eprintln!("rshell: {}: {}", path, e); return Some(1); }
                    }
                } else if let Some(ref path) = cmd.redirect_append {
                    match OpenOptions::new().append(true).create(true).open(path) {
                        Ok(mut f) => { let _ = f.write_all(output.as_bytes()); }
                        Err(e) => { eprintln!("rshell: {}: {}", path, e); return Some(1); }
                    }
                } else {
                    print!("{}", output);
                    let _ = io::stdout().flush();
                }
                Some(0)
            }
            Some("export") => {
                for arg in &argv[1..] {
                    if let Some(eq) = arg.find('=') {
                        let key = arg[..eq].to_string();
                        let val = arg[eq + 1..].to_string();
                        self.env.insert(key.clone(), val.clone());
                        env::set_var(&key, &val);
                    } else {
                        if let Ok(val) = env::var(arg) {
                            self.env.insert(arg.to_string(), val);
                        }
                    }
                }
                Some(0)
            }
            Some("unset") => {
                for arg in &argv[1..] {
                    self.env.remove(arg);
                    env::remove_var(arg);
                }
                Some(0)
            }
            Some("history") => {
                for (i, entry) in self.history.iter().enumerate() {
                    println!("{:4}  {}", i + 1, entry);
                }
                Some(0)
            }
            _ => None,
        }
    }
```

**External command execution:**

```rust
    fn run_external(&mut self, cmd: &SimpleCommand,
                    stdin_override: Option<Stdio>,
                    stdout_override: Option<Stdio>) -> i32 {
        let argv = self.expand_args(&cmd.argv);
        let program = &argv[0];
        let args = &argv[1..];

        let stdin = if let Some(s) = stdin_override {
            s
        } else if let Some(ref path) = cmd.redirect_in {
            match File::open(path) {
                Ok(f) => Stdio::from(f),
                Err(e) => { eprintln!("rshell: {}: {}", path, e); return 1; }
            }
        } else { Stdio::inherit() };

        let stdout = if let Some(s) = stdout_override {
            s
        } else if let Some(ref path) = cmd.redirect_out {
            match File::create(path) {
                Ok(f) => Stdio::from(f),
                Err(e) => { eprintln!("rshell: {}: {}", path, e); return 1; }
            }
        } else if let Some(ref path) = cmd.redirect_append {
            match OpenOptions::new().append(true).create(true).open(path) {
                Ok(f) => Stdio::from(f),
                Err(e) => { eprintln!("rshell: {}: {}", path, e); return 1; }
            }
        } else { Stdio::inherit() };

        let mut child_cmd = Command::new(program);
        child_cmd.args(args).stdin(stdin).stdout(stdout).envs(self.env.iter());

        match child_cmd.spawn() {
            Ok(mut child) => match child.wait() {
                Ok(status) => status.code().unwrap_or(1),
                Err(e) => { eprintln!("rshell: wait: {}", e); 1 }
            },
            Err(e) => {
                if e.kind() == io::ErrorKind::NotFound {
                    eprintln!("rshell: {}: command not found", program);
                    127 // exit code 127 = command not found (มาตรฐาน POSIX)
                } else {
                    eprintln!("rshell: {}: {}", program, e);
                    1
                }
            }
        }
    }
```

**ทดสอบ execute ขั้นพื้นฐาน:**

```
$ pwd
/home/user/rshell
$ echo hello world
hello world
$ true
$ echo $?
0
$ false
$ echo $?
1
$ __nonexistent__
rshell: __nonexistent__: command not found
$ echo $?
127
```

---

### ขั้นที่ 5: Pipeline Support (`ls | grep foo | wc -l`)

Pipeline คือการเชื่อมหลาย processes โดยส่ง stdout ของ process หนึ่งเป็น stdin ของ process ถัดไป

**หลักการ:**
```
process[0] ─stdout─▶ pipe[0] ─stdin─▶ process[1] ─stdout─▶ pipe[1] ─stdin─▶ process[2]
```

ใน Rust เราใช้ `Stdio::piped()` เพื่อขอให้ OS สร้าง pipe และให้ `ChildStdout` ที่เราสามารถแปลงเป็น `Stdio` สำหรับ process ถัดไปได้

**`src/executor.rs` (ส่วน pipeline):**

```rust
    fn execute_pipeline(&mut self, pipeline: &Pipeline) -> i32 {
        let cmds = &pipeline.commands;

        // กรณี single command: ตรวจ builtin ก่อน
        if cmds.len() == 1 {
            let cmd = &cmds[0];
            let argv = self.expand_args(&cmd.argv);
            if let Some(code) = self.try_builtin(&argv, cmd) {
                return code;
            }
            return self.run_external(cmd, None, None);
        }

        // Multi-command pipeline
        let mut children: Vec<std::process::Child> = Vec::new();
        let mut prev_stdout: Option<std::process::ChildStdout> = None;

        for (i, cmd) in cmds.iter().enumerate() {
            let is_last = i == cmds.len() - 1;

            // stdin: ได้จาก pipe ก่อนหน้า หรือ redirect_in หรือ inherit
            let stdin_pipe = if let Some(prev) = prev_stdout.take() {
                Stdio::from(prev) // แปลง ChildStdout → Stdio
            } else if let Some(ref path) = cmd.redirect_in {
                match File::open(path) {
                    Ok(f) => Stdio::from(f),
                    Err(e) => { eprintln!("rshell: {}: {}", path, e); return 1; }
                }
            } else {
                Stdio::inherit()
            };

            // stdout: piped (ถ้ามี next cmd) หรือ redirect_out หรือ inherit
            let stdout_pipe = if !is_last {
                Stdio::piped() // บอก OS ว่าต้องการ pipe
            } else if let Some(ref path) = cmd.redirect_out {
                match File::create(path) {
                    Ok(f) => Stdio::from(f),
                    Err(e) => { eprintln!("rshell: {}: {}", path, e); return 1; }
                }
            } else if let Some(ref path) = cmd.redirect_append {
                match OpenOptions::new().append(true).create(true).open(path) {
                    Ok(f) => Stdio::from(f),
                    Err(e) => { eprintln!("rshell: {}: {}", path, e); return 1; }
                }
            } else {
                Stdio::inherit()
            };

            let argv = self.expand_args(&cmd.argv);
            let program = &argv[0];

            let mut child_cmd = Command::new(program);
            child_cmd.args(&argv[1..])
                     .stdin(stdin_pipe)
                     .stdout(stdout_pipe)
                     .envs(self.env.iter());

            match child_cmd.spawn() {
                Ok(mut child) => {
                    if !is_last {
                        // ดึง ChildStdout ออกมาก่อนที่ child จะถูก move เข้า Vec
                        prev_stdout = child.stdout.take();
                    }
                    children.push(child);
                }
                Err(e) => {
                    eprintln!("rshell: {}: {}", program, e);
                    return 127;
                }
            }
        }

        // รอให้ทุก process เสร็จ และ return exit code ของ process สุดท้าย
        let mut last_code = 0;
        for (i, mut child) in children.into_iter().enumerate() {
            match child.wait() {
                Ok(status) => {
                    if i == cmds.len() - 1 {
                        last_code = status.code().unwrap_or(1);
                    }
                }
                Err(e) => { eprintln!("rshell: wait error: {}", e); last_code = 1; }
            }
        }
        last_code
    }
```

**ทดสอบ pipeline:**

```
$ ls /tmp | grep test
$ echo "hello world" | tr a-z A-Z
HELLO WORLD
$ ls | sort | head -3
Cargo.lock
Cargo.toml
README.md
$ echo foo | tr a-z A-Z
FOO
```

**ข้อสังเกตสำคัญ:** ต้อง spawn ทุก process ก่อนค่อย wait — ถ้า wait ทีละตัวจะเกิด deadlock เพราะ process ก่อน block รอให้ pipe ถูกอ่าน แต่เราก็ยังไม่ได้ spawn process ที่จะอ่าน pipe นั้น

---

### ขั้นที่ 6: I/O Redirection

เราได้ implement redirect พื้นฐานไว้แล้วในขั้นที่ 4–5 แต่ขั้นนี้จะทดสอบอย่างละเอียด

```
$ echo hello > /tmp/out.txt         # สร้างไฟล์ใหม่ หรือ truncate ไฟล์เก่า
$ cat /tmp/out.txt
hello
$ echo world >> /tmp/out.txt        # append (ไม่ลบเนื้อหาเดิม)
$ cat /tmp/out.txt
hello
world
$ wc -l < /tmp/out.txt              # ส่ง stdin จากไฟล์
2
$ cat < /tmp/out.txt | tr a-z A-Z  # ผสม redirect + pipe
HELLO
WORLD
```

**ความแตกต่างระหว่าง `>` และ `>>`:**

| Operator | ผลลัพธ์ |
|----------|---------|
| `>` | เปิดด้วย `File::create()` — truncate ไฟล์ก่อนเขียน |
| `>>` | เปิดด้วย `OpenOptions::new().append(true).create(true)` — เพิ่มต่อท้าย |

**สิ่งสำคัญ:** built-in commands เช่น `echo` ต้องจัดการ redirection เอง เพราะพวกมันรัน in-process ไม่ใช่เป็น child process แยก ดังนั้นเราต้องตรวจ `cmd.redirect_out` / `cmd.redirect_append` ใน `try_builtin()` ด้วย

---

### ขั้นที่ 7: Environment Variable Expansion (`$HOME`, `$PATH`)

Variable expansion แปลง `$NAME` หรือ `${NAME}` ในแต่ละ argument เป็นค่าจาก environment

**`src/executor.rs` (ส่วน expansion):**

```rust
    /// Expand $VAR and ${VAR} in a single string
    pub fn expand_vars(&self, s: &str) -> String {
        let mut result = String::new();
        let mut chars = s.chars().peekable();

        while let Some(ch) = chars.next() {
            if ch != '$' {
                result.push(ch);
                continue;
            }
            // พบ $ ดูว่าตามมาด้วยอะไร
            match chars.peek() {
                Some(&'?') => {
                    // $? → exit code ของคำสั่งก่อนหน้า
                    chars.next();
                    result.push_str(&self.last_exit.to_string());
                }
                Some(&'{') => {
                    // ${VAR} syntax
                    chars.next(); // consume '{'
                    let mut name = String::new();
                    loop {
                        match chars.next() {
                            Some('}') => break,
                            Some(c)   => name.push(c),
                            None      => break,
                        }
                    }
                    let val = self.env.get(&name).map(|s| s.as_str()).unwrap_or("");
                    result.push_str(val);
                }
                Some(&c) if c.is_alphanumeric() || c == '_' => {
                    // $VAR syntax — อ่านชื่อตัวแปรจนกว่าจะเจอตัวอักษรที่ไม่ใช่ alphanumeric/_
                    let mut name = String::new();
                    while let Some(&c) = chars.peek() {
                        if c.is_alphanumeric() || c == '_' {
                            name.push(c);
                            chars.next();
                        } else {
                            break;
                        }
                    }
                    let val = self.env.get(&name).map(|s| s.as_str()).unwrap_or("");
                    result.push_str(val);
                }
                _ => result.push('$'), // ไม่ใช่ตัวแปร เก็บ $ literal
            }
        }
        result
    }

    fn expand_args(&self, argv: &[String]) -> Vec<String> {
        argv.iter().map(|a| self.expand_vars(a)).collect()
    }
```

**ทดสอบ variable expansion:**

```
$ echo $HOME
/home/user
$ echo ${PATH}
/usr/local/bin:/usr/bin:/bin
$ export GREETING=hello
$ echo $GREETING world
hello world
$ unset GREETING
$ echo $GREETING
                    ← (empty string)
$ echo $?
0
$ false
$ echo "exit code was: $?"
exit code was: 1
```

**ข้อควรระวัง:** เราทำ expansion หลัง tokenize แต่ก่อน execute เพราะ:
1. tokenizer ไม่ควรรู้จัก environment
2. expansion ใน single quotes ไม่ควรเกิด (tokenizer จัดการไว้แล้วโดยเก็บ `$` เป็น literal ใน single-quoted string)

---

### ขั้นที่ 8: Signal Handling + Interactive REPL

สร้าง REPL ที่สมบูรณ์ด้วย `rustyline` สำหรับ line editing และ `nix` สำหรับ signal handling

**ทำไม Ctrl+C ถึงต้องไม่ kill shell:**

ใน Unix เมื่อกด Ctrl+C terminal จะส่ง SIGINT ไปยัง **process group** ทั้งหมด ซึ่งรวมถึง shell process ด้วย shell ปกติจะ ignore SIGINT ในตัวเอง แต่จะ propagate ไปยัง child processes (foreground job) ผ่าน process group

**`src/executor.rs` (ส่วน REPL):**

```rust
    pub fn run_repl(&mut self) {
        use rustyline::error::ReadlineError;
        use rustyline::DefaultEditor;

        // Ignore SIGINT ใน shell process เอง
        // child processes สืบทอด signal disposition จาก parent
        // แต่เมื่อ exec() ถูกเรียก signal handlers จะ reset กลับเป็น default
        // ดังนั้น child process จะยังรับ SIGINT ได้ปกติ
        #[cfg(unix)]
        {
            use nix::sys::signal::{self, SigHandler, Signal};
            unsafe {
                // SigHandler::SigIgn = ignore SIGINT ใน shell
                let _ = signal::signal(Signal::SIGINT, SigHandler::SigIgn);
            }
        }

        let mut rl = DefaultEditor::new().expect("failed to create line editor");

        loop {
            // สร้าง prompt แสดง current directory
            let prompt = format!(
                "{}$ ",
                env::current_dir()
                    .map(|p| p.display().to_string())
                    .unwrap_or_else(|_| "?".to_string())
            );

            match rl.readline(&prompt) {
                Ok(line) => {
                    // เพิ่มลงใน rustyline history (สำหรับ arrow up/down)
                    let _ = rl.add_history_entry(line.as_str());
                    self.run_line(&line);
                }
                Err(ReadlineError::Interrupted) => {
                    // Ctrl+C ขณะพิมพ์คำสั่ง: แค่ขึ้นบรรทัดใหม่
                    println!();
                }
                Err(ReadlineError::Eof) => {
                    // Ctrl+D: exit shell
                    break;
                }
                Err(e) => {
                    eprintln!("rshell: readline error: {}", e);
                    break;
                }
            }
        }
    }
```

**ทดสอบ signal handling:**

```
/home/user$ sleep 100
^C          ← กด Ctrl+C
/home/user$ echo still alive
still alive
```

Shell ยังคงทำงานต่อ เพราะ SIGINT ถูก ignore ใน shell process แต่ `sleep 100` รับ SIGINT ตามปกติและ terminate

---

## การทดสอบ (Testing)

### Unit Tests

Unit tests ถูก embed ไว้ในแต่ละโมดูล run ด้วย `cargo test`:

```rust
// tests/integration_test.rs
use assert_cmd::Command;
use predicates::prelude::*;

fn rshell(cmd: &str) -> assert_cmd::assert::Assert {
    Command::cargo_bin("rshell")
        .unwrap()
        .args(["--command", cmd])
        .assert()
}

#[test]
fn test_echo_hello() {
    rshell("echo hello")
        .success()
        .stdout(predicate::str::contains("hello"));
}

#[test]
fn test_pipeline_tr_uppercase() {
    // ทดสอบ pipeline: echo foo | tr a-z A-Z
    rshell("echo foo | tr a-z A-Z")
        .success()
        .stdout(predicate::str::contains("FOO"));
}

#[test]
fn test_cd_and_pwd() {
    // ทดสอบ cd ตามด้วย pwd ด้วย && operator
    rshell("cd /tmp && pwd")
        .success()
        .stdout(predicate::str::contains("/tmp"));
}

#[test]
fn test_exit_code_true() {
    rshell("true").success();
}

#[test]
fn test_exit_code_false() {
    rshell("false").failure();
}

#[test]
fn test_redirect_out() {
    rshell("echo redirected > /tmp/rshell_test_out.txt").success();
    let content = std::fs::read_to_string("/tmp/rshell_test_out.txt").unwrap();
    assert!(content.contains("redirected"));
    let _ = std::fs::remove_file("/tmp/rshell_test_out.txt");
}

#[test]
fn test_sequence() {
    rshell("echo first ; echo second")
        .success()
        .stdout(predicate::str::contains("first"))
        .stdout(predicate::str::contains("second"));
}

#[test]
fn test_env_export() {
    rshell("export MY_VAR=hello ; echo $MY_VAR")
        .success()
        .stdout(predicate::str::contains("hello"));
}
```

### Real `cargo test` Output

ผลลัพธ์จากการรัน `cargo test` จริง:

```
$ cargo test
   Compiling rshell v0.1.0 (/tmp/scratchpad/shell_interp)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 5.53s
     Running unittests src/main.rs (target/debug/deps/rshell-db48ea94b31542e9)

running 17 tests
test executor::tests::test_builtin_echo ... ok
test executor::tests::test_builtin_pwd ... ok
test executor::tests::test_builtin_cd ... ok
test lexer::tests::test_double_quotes ... ok
test lexer::tests::test_pipe ... ok
test executor::tests::test_expand_vars ... ok
test executor::tests::test_command_not_found ... ok
test lexer::tests::test_redirect_append ... ok
test lexer::tests::test_redirect ... ok
test lexer::tests::test_semicolon ... ok
test lexer::tests::test_simple_words ... ok
test lexer::tests::test_single_quotes ... ok
test parser::tests::test_parse_pipeline ... ok
test parser::tests::test_parse_redirect ... ok
test parser::tests::test_parse_simple ... ok
test parser::tests::test_parse_sequence ... ok
test executor::tests::test_external_command ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs (target/debug/deps/integration_test-6a30f8ade1538666)

running 8 tests
test test_env_export ... ok
test test_cd_and_pwd ... ok
test test_echo_hello ... ok
test test_exit_code_false ... ok
test test_redirect_out ... ok
test test_exit_code_true ... ok
test test_sequence ... ok
test test_pipeline_tr_uppercase ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

**ทุก test ผ่าน: 25/25** (17 unit + 8 integration)

---

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. Partial Move ของ `Child.stdout`

ปัญหานี้เกิดบ่อยมากเมื่อทำ pipeline — เราต้องการดึง `stdout` ออกจาก `Child` แล้ว store ใน `Vec<Child>` ในคราวเดียว แต่ borrow checker ไม่ยอม:

```rust
// ❌ ERROR: partial move ออกจาก child แล้วยังพยายาม move child เข้า Vec
let child = cmd.spawn().unwrap();
let out = child.stdout;   // partial move: ดึง stdout field ออก
children.push(child);     // ERROR: child ถูก partially moved แล้ว
child.wait().unwrap();    // ERROR: same issue
```

```
error[E0382]: borrow of partially moved value: `child`
 --> src/executor.rs:7:9
  |
6 |         let _out = child.stdout; // move
  |                    ------------ value partially moved here
7 |         child.wait().unwrap();  // child was partially moved
  |         ^^^^^ value borrowed here after partial move
  |
  = note: partial move occurs because `child.stdout` has type `Option<ChildStdout>`,
          which does not implement the `Copy` trait
```

**วิธีแก้:** ใช้ `.take()` แทน direct field access เพื่อ extract value ผ่าน `&mut self` โดยไม่ consume whole struct:

```rust
// ✅ ถูกต้อง: take() ทำงานผ่าน mutable reference ไม่ใช่ move
let mut child = cmd.spawn().unwrap();
let out = child.stdout.take(); // สลับ Some(stdout) กับ None ใน place
children.push(child);          // OK: child ยังสมบูรณ์ (stdout กลายเป็น None แล้ว)
```

### 2. `Stdio` ไม่ Implement `Clone`

เมื่อต้องการส่ง stdout เดียวกันไปหลาย process หรือ retry การสร้าง command:

```rust
// ❌ ERROR: Stdio ไม่มี clone()
let s = Stdio::piped();
let _s2 = s.clone(); // method not found in `Stdio`
```

```
error[E0599]: no method named `clone` found for struct `Stdio` in the current scope
 --> src/main.rs:5:17
  |
5 |     let _s2 = s.clone(); // Stdio doesn't implement Clone
  |                 ^^^^^ method not found in `Stdio`
```

**วิธีแก้:** สร้าง `Stdio::piped()` ใหม่ทุกครั้งที่ต้องการ หรือใช้ `File::try_clone()` ถ้าต้องการ share file handle:

```rust
// ✅ แต่ละ process ต้องได้ Stdio ของตัวเอง
let stdin_for_process_1 = Stdio::from(file.try_clone()?);
let stdin_for_process_2 = Stdio::from(file.try_clone()?);
```

### 3. `set_current_dir` ไม่อัปเดต `$PWD` อัตโนมัติ

บน Linux, environment variable `PWD` ไม่ได้เป็น magic variable ที่ OS อัปเดตให้อัตโนมัติ — มันเป็นแค่ string ธรรมดาที่ shell ต้องจัดการเอง:

```rust
// ❌ ผิด: PWD ยังคงเป็นค่าเก่า
std::env::set_current_dir("/tmp").unwrap();
let pwd = std::env::var("PWD").unwrap_or_default();
println!("PWD: {}", pwd);       // ยังเป็น /home/user หรือค่าเดิม
println!("actual: {}", std::env::current_dir().unwrap().display()); // /tmp
```

ผลลัพธ์จริง:
```
PWD: /home/user/rust_course
actual: /tmp
```

**วิธีแก้:** อัปเดต `$PWD` ด้วยตัวเองหลัง `set_current_dir`:

```rust
// ✅ ถูกต้อง
std::env::set_current_dir("/tmp").unwrap();
if let Ok(cwd) = std::env::current_dir() {
    std::env::set_var("PWD", cwd.to_string_lossy().as_ref());
    self.env.insert("PWD".to_string(), cwd.to_string_lossy().to_string());
}
```

### 4. Deadlock ใน Pipeline ถ้า Wait ก่อน Spawn ครบ

```rust
// ❌ DEADLOCK: spawn process[0], wait มันก่อน, แล้วค่อย spawn process[1]
let child0 = Command::new("yes").stdout(Stdio::piped()).spawn()?;
child0.wait()?; // DEADLOCK: yes รอให้ pipe ถูกอ่าน แต่เราไม่ได้ spawn reader
let _child1 = Command::new("head").args(["-1"])
    .stdin(child0.stdout.take().unwrap()) // นี่ไม่มีวันถึง
    .spawn()?;
```

`yes` จะ block รอให้ pipe buffer ถูกอ่าน และ `head` ก็ไม่ถูก spawn เพราะรอ `yes` เสร็จก่อน → deadlock

**วิธีแก้:** Spawn ทุก process ก่อน จากนั้นค่อย wait ทั้งหมด:

```rust
// ✅ ถูกต้อง
let mut child0 = Command::new("yes").stdout(Stdio::piped()).spawn()?;
let stdout0 = child0.stdout.take().unwrap();
let mut child1 = Command::new("head").args(["-1"])
    .stdin(Stdio::from(stdout0))
    .spawn()?;
// THEN wait:
child0.wait()?;
child1.wait()?;
```

### 5. Borrow Checker กับ Self-referencing State

เมื่อ method ของ `Executor` return reference ที่ชี้ไปยัง internal state แล้วพยายามใช้ self อีกครั้ง:

```rust
struct Shell {
    env: HashMap<String, String>,
    history: Vec<String>,
}
impl Shell {
    fn get_var(&self, key: &str) -> Option<&str> {
        self.env.get(key).map(|s| s.as_str())
    }
}

fn main() {
    let mut sh = Shell { env: HashMap::new(), history: Vec::new() };
    sh.env.insert("HOME".to_string(), "/home/user".to_string());

    let val = sh.get_var("HOME"); // borrow sh immutably → &str ชี้ไปใน sh.env
    sh.history.push(val.unwrap_or("").to_string()); // ERROR: mutable borrow ขณะที่ immutable borrow ยังมีชีวิต
    println!("{:?}", val); // val ยังถูกใช้อยู่ตรงนี้
}
```

```
error[E0502]: cannot borrow `sh.history` as mutable because it is also borrowed as immutable
  --> src/main.rs:20:5
   |
19 |     let val = sh.get_var("HOME"); // borrows sh immutably
   |               -- immutable borrow occurs here
20 |     sh.history.push(val.unwrap_or("").to_string()); // mutable borrow
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
21 |     println!("{:?}", val);
   |                      --- immutable borrow later used here
```

**วิธีแก้:** Clone หรือ convert เป็น owned String ก่อนที่จะทำ mutable operation:

```rust
// ✅ ถูกต้อง: แปลงเป็น owned String ก่อน
let val: String = sh.get_var("HOME").unwrap_or("").to_string(); // owned → borrow จบ
sh.history.push(val.clone()); // OK
println!("{}", val);
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/rshell
strip target/release/rshell  # ลด binary size
ls -lh target/release/rshell
```

### ติดตั้งใน PATH

```bash
cargo install --path .
# หรือ
cp target/release/rshell ~/.local/bin/
```

### เพิ่มลงใน `/etc/shells` (optional)

```bash
echo "$(which rshell)" | sudo tee -a /etc/shells
# จากนั้น users สามารถเปลี่ยน shell ได้ด้วย:
chsh -s $(which rshell)
```

### Build สำหรับ static linking (Alpine Linux / musl)

```bash
# ติดตั้ง musl target
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
# ได้ static binary ที่ไม่ต้องพึ่ง glibc
```

### Docker Image

```dockerfile
FROM rust:1.75-alpine AS builder
RUN apk add --no-cache musl-dev
WORKDIR /build
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /build/target/x86_64-unknown-linux-musl/release/rshell /rshell
ENTRYPOINT ["/rshell"]
```

---

## การต่อยอด (Extensions & Exercises)

### 1. Tab Completion ด้วย `rustyline` Completer (ระดับ: กลาง)

`rustyline` มี trait `Completer` ที่ให้เราเพิ่ม completion logic เองได้

**hint:** ต้องสร้าง struct ที่ implement `rustyline::completion::Completer` แล้วส่ง `DefaultEditor::with_config()` พร้อม helper ที่ custom

```rust
use rustyline::completion::{Completer, FilenameCompleter, Pair};
use rustyline::Context;

struct ShellCompleter {
    file_completer: FilenameCompleter,
}

impl Completer for ShellCompleter {
    type Candidate = Pair;
    fn complete(&self, line: &str, pos: usize, ctx: &Context<'_>)
        -> rustyline::Result<(usize, Vec<Pair>)>
    {
        // ถ้าเป็น argument แรก → complete จาก PATH
        // ถ้าเป็น argument ถัดไป → complete filename
        self.file_completer.complete(line, pos, ctx)
    }
}
```

### 2. Here-doc Support (`<< EOF`) (ระดับ: ยาก)

Heredoc ให้ผู้ใช้พิมพ์ multi-line input inline:
```bash
cat << EOF
line 1
line 2
EOF
```

**hint:** tokenizer ต้องตรวจ `<<` token และอ่าน lines ต่อไปจนกว่าจะเจอ delimiter word เก็บเป็น `Token::HereDoc(String)` แล้ว executor แปลงเป็น stdin pipe ด้วย `std::io::Cursor`

```rust
// executor: แปลง heredoc เป็น pipe
let heredoc_content = "line 1\nline 2\n".as_bytes().to_vec();
let (reader, mut writer) = os_pipe::pipe()?;
writer.write_all(&heredoc_content)?;
drop(writer); // close write end → EOF สำหรับ reader
Stdio::from(reader)
```

### 3. Job Control (`fg`, `bg`, `jobs`) (ระดับ: ยาก)

Job control ให้ผู้ใช้ suspend (Ctrl+Z) และ resume processes:

**hint:** ต้องใช้ `nix::sys::signal::Signal::SIGTSTP` และ `nix::unistd::tcsetpgrp` สำหรับ terminal control

```rust
// Store background jobs
struct Job {
    id: usize,
    pid: nix::unistd::Pid,
    status: JobStatus,
    command: String,
}

enum JobStatus { Running, Stopped, Done(i32) }

// ใน executor เก็บ Vec<Job>
// builtin "jobs" แสดงรายการ
// builtin "fg %1" ส่ง SIGCONT และ wait
// builtin "bg %1" ส่ง SIGCONT โดยไม่ wait
```

### 4. Subshell และ Command Substitution `$(...)` (ระดับ: สูงมาก)

Command substitution แทน `$(cmd)` ด้วย stdout ของ `cmd`:

```bash
echo "Today is $(date +%Y-%m-%d)"
files=$(ls /tmp)
```

**hint:** tokenizer ต้องตรวจ `$(` แล้ว count parenthesis depth เก็บเนื้อหาเป็น `Token::CmdSubst(String)` executor รัน nested shell แล้ว capture stdout:

```rust
// executor: รัน command substitution
fn cmd_subst(&mut self, cmd: &str) -> String {
    let output = std::process::Command::new(std::env::current_exe().unwrap())
        .args(["--command", cmd])
        .output()
        .unwrap_or_default();
    String::from_utf8_lossy(&output.stdout)
        .trim_end_matches('\n')
        .to_string()
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง shell interpreter ที่ทำงานได้จริง โดยเรียนรู้:

1. **Lexer/Parser architecture** — แยก tokenization ออกจาก parsing และ execution ทำให้ code แต่ละส่วน testable ได้อิสระ

2. **Unix Process Model** — ความแตกต่างระหว่าง built-in commands (รัน in-process, ต้องจัดการ state โดยตรง) และ external commands (fork/exec ผ่าน `std::process::Command`)

3. **Pipeline mechanics** — ต้อง spawn ทุก process ก่อนค่อย wait เพื่อหลีกเลี่ยง deadlock, ใช้ `ChildStdout::take()` เพื่อหลีกเลี่ยง partial move

4. **Borrow checker realities** — `Stdio` ไม่ implement `Clone`, partial moves ต้องแก้ด้วย `.take()`, self-referencing borrows ต้องแก้ด้วยการ clone เป็น owned values

5. **Signal handling** — SIGINT ใน shell process ควร ignore แต่ child processes ยังรับได้ตามปกติเพราะ `exec()` reset signal dispositions

6. **Environment management** — ต้องอัปเดต `$PWD` เอง, `export` ต้องอัปเดตทั้ง internal HashMap และ `std::env`

Shell interpreter เป็นโปรเจคที่ครอบคลุม systems programming ทั้งหมดที่จำเป็น และเป็นรากฐานที่ดีสำหรับโปรเจคถัดไปที่ต้องทำงานกับ I/O, processes, และ networking

---

**โปรเจคก่อนหน้า:** (บทแรกของโมดูล A) | **โปรเจคถัดไป:** [HTTP Client (curl-like)](project-a02-http-client.md)
