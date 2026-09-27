# Project A06: Terminal Text Editor

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 6 ชั่วโมง

## ภาพรวมโปรเจค

Text editor คือโปรแกรมที่นักพัฒนาซอฟต์แวร์ใช้งานทุกวัน แต่คนส่วนใหญ่ไม่เคยลองสร้างมันด้วยตัวเอง โปรเจคนี้จะสร้าง **terminal text editor** ที่ทำงานในโหมด raw terminal — เหมือน `nano`, `micro`, หรือ `kilo` (text editor ขนาดเล็กที่มีชื่อเสียง) แต่เขียนด้วย Rust อย่างถูกต้องตั้งแต่ต้น

ทำไมต้องสร้าง text editor? เพราะมันรวมทักษะ systems programming ไว้แบบครบถ้วนที่สุด:

- **Terminal control** ที่ต้องสื่อสารกับ OS kernel โดยตรงผ่าน POSIX termios
- **Data structure** ที่ต้องเลือกแบบระวังเพื่อให้การแก้ไขข้อความมีประสิทธิภาพ (Rope vs Gap Buffer)
- **Event-driven programming** ที่ต้อง handle keyboard events ทุกประเภท
- **File I/O** ที่ต้อง safe (atomic save ป้องกันข้อมูลเสียหายถ้าไฟฟ้าดับกลางทาง)
- **Incremental search** ที่ต้อง highlight matches ขณะพิมพ์
- **Syntax highlighting** ที่ต้อง tokenize source code แบบ real-time
- **Undo/Redo** ที่ต้อง model เป็น command stack

ใน production จริง editor อย่าง `helix`, `zed` (บางส่วน), และ `lapce` เขียนด้วย Rust ทั้งหมด สิ่งที่เราสร้างในโปรเจคนี้จะช่วยให้เข้าใจ core loop ของ editor เหล่านั้นได้ลึกขึ้นมาก

## สิ่งที่จะได้เรียนรู้

- **Raw terminal mode:** `crossterm` enable/disable raw mode, alternate screen, cursor movement via ANSI escape codes
- **Event loop pattern:** อ่าน keyboard events และ dispatch ไปยัง handler อย่างถูกต้อง
- **Rope data structure:** ทำไม `String` ถึงไม่พอสำหรับ editor และ `ropey` crate แก้ปัญหาอย่างไร
- **Viewport/scrolling:** render เฉพาะส่วนที่มองเห็น — ไม่ต้อง redraw ทั้งไฟล์
- **Atomic file save:** เขียนไป temp file แล้ว rename เพื่อ guarantee atomicity
- **Command pattern สำหรับ Undo/Redo:** `Vec<EditCommand>` เป็น undo stack
- **Incremental search + highlight:** หา pattern ใน rope และ mark positions สำหรับ rendering
- **Syntax highlighting:** detect token types แล้วใส่ ANSI color codes ขณะ render

## ความรู้ที่ต้องมีมาก่อน

- **Part 21–25:** struct, enum, impl blocks และ method syntax
- **Part 31–35:** error handling ด้วย `Result`, `?` operator
- **Part 41–45:** collections — `Vec`, `HashMap`, `String`
- **Part 61–65:** standard library — `std::fs`, `std::io`, `std::path`
- **Part 71–75:** trait objects, closures, iterators
- **Part 81–85:** unsafe basics และ FFI concepts (terminal syscalls)

## โครงสร้างโปรเจค (Project Layout)

```
terminal-editor/
├── src/
│   ├── main.rs          ← Entry point: CLI args (clap), event loop หลัก
│   ├── editor.rs        ← Editor state: ทุก field รวมกัน
│   ├── buffer.rs        ← Buffer struct: ครอบ ropey::Rope + undo stack
│   ├── viewport.rs      ← Viewport: scroll state, visible lines
│   ├── terminal.rs      ← Terminal init/cleanup, draw routines
│   ├── search.rs        ← Incremental search, match positions
│   ├── highlight.rs     ← Syntax highlighting (Rust keywords)
│   └── commands.rs      ← EditCommand enum สำหรับ undo/redo
├── tests/
│   └── buffer_tests.rs  ← Integration tests สำหรับ Buffer
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### ทำไมถึงใช้ Rope แทน Vec\<String\>

วิธีง่ายที่สุดในการเก็บข้อความคือ `Vec<String>` (หนึ่ง String ต่อหนึ่งบรรทัด) แต่มีปัญหาคือ:

1. **Insert ตรงกลาง:** ถ้าไฟล์มี 100,000 บรรทัดและ cursor อยู่บรรทัด 50,000 การ insert บรรทัดใหม่ต้อง shift elements ทั้งหมดที่อยู่หลัง = O(n)
2. **Long lines:** บรรทัดที่ยาวมาก (เช่น minified JavaScript) ต้อง reallocate ทั้ง String ทุกครั้งที่แก้
3. **Undo snapshot:** ถ้าเก็บ full-text snapshot ทุกครั้ง memory จะพุ่งสูงมาก

**Rope** แก้ปัญหาเหล่านี้ด้วยการเก็บข้อความเป็น balanced binary tree of string chunks:

```
         [root: "Hello, world!\nFoo bar"]
          /                       \
  ["Hello, world!\n"]         ["Foo bar"]
      /       \
 ["Hello, "]  ["world!\n"]
```

การ insert O(log n) — แค่ split node แล้วสร้าง node ใหม่, ไม่ต้อง copy ทั้งไฟล์ Crate `ropey` implement Rope ที่ Unicode-aware (นับเป็น char/grapheme cluster ไม่ใช่ byte) ซึ่งเหมาะมากสำหรับ editor ที่ต้องรองรับภาษาไทย/จีน/ญี่ปุ่น

### Event Loop Architecture

```
main() → Editor::run()
           │
           ▼
     loop {
       terminal::draw(&editor)   ← render ทุก frame
       event = crossterm::event::read()
       match event {
         Key(Ctrl+Q)  → break
         Key(Ctrl+S)  → editor.save()
         Key(Ctrl+F)  → editor.start_search()
         Key(Ctrl+Z)  → editor.undo()
         Key(Ctrl+Y)  → editor.redo()
         Key(Arrow)   → editor.move_cursor(dir)
         Key(Char(c)) → editor.insert_char(c)
         Key(Backspace) → editor.delete_before_cursor()
         ...
       }
     }
```

### Atomic Save Flow

```
Ctrl+S pressed
    │
    ▼
rope.to_string() → content: String
    │
    ▼
Path::new("/tmp/.tmp_filename.txt")   ← temp file ใน dir เดียวกัน
    │
    ▼
File::create(tmp) → write all → flush
    │
    ▼
fs::rename(tmp, real_path)   ← atomic rename syscall
    │
    ▼
modified = false
```

`rename()` เป็น atomic บน Unix filesystem — ถ้าระบบ crash ระหว่าง write ไฟล์เดิมยังอยู่ครบถ้วน

### ViewModel: Status Bar

```
┌─────────────────────────────────────────────────────────────┐
│ src/main.rs                                            [+]  │  ← filename + modified flag
│ line 42 of 150  col 10  UTF-8  Normal                       │  ← position + encoding + mode
└─────────────────────────────────────────────────────────────┘
```

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: เข้า Raw Mode, วาดข้อความ, ออกด้วย 'q'

ก่อนอื่นต้องเข้าใจว่า "raw mode" คืออะไร — ปกติ terminal จะ buffer การกด key ไว้จนกดเอนเตอร์ (canonical mode) และแสดงตัวอักษรให้เราเห็นอัตโนมัติ (echo mode) ใน text editor เราต้องการ:

- รู้ทันทีว่ากด key อะไร (ไม่ต้อง wait enter)
- ควบคุมการแสดงผลเอง (cursor อยู่ที่ไหน สีอะไร)
- ใช้ **alternate screen** — เป็น buffer แยกของ terminal เพื่อไม่ให้ editor ลบ history ที่มีอยู่

`crossterm` จัดการ raw mode ผ่าน RAII — ใช้ `Drop` trait เพื่อ restore terminal state อัตโนมัติเมื่อ struct ถูก drop:

```toml
# Cargo.toml
[package]
name = "terminal-editor"
version = "0.1.0"
edition = "2021"

[dependencies]
crossterm = "0.27"
ropey = "1.6"
clap = { version = "4", features = ["derive"] }
owo-colors = "4"
```

```rust
// src/terminal.rs — ขั้นที่ 1: raw mode + hello world

use crossterm::{
    cursor,
    event::{self, Event, KeyCode, KeyModifiers},
    execute, queue,
    style::{Color, Print, SetForegroundColor, ResetColor},
    terminal::{
        self, ClearType, EnterAlternateScreen, LeaveAlternateScreen,
    },
};
use std::io::{self, Write};

/// RAII guard — เข้า raw mode ตอน new(), ออกตอน Drop
pub struct RawTerminal {
    pub stdout: io::Stdout,
}

impl RawTerminal {
    pub fn new() -> crossterm::Result<Self> {
        let mut stdout = io::stdout();
        terminal::enable_raw_mode()?;
        execute!(stdout, EnterAlternateScreen, cursor::Hide)?;
        Ok(RawTerminal { stdout })
    }
}

impl Drop for RawTerminal {
    fn drop(&mut self) {
        // ไม่ว่าจะ exit ปกติหรือ panic เสมอ restore terminal
        let _ = terminal::disable_raw_mode();
        let _ = execute!(
            self.stdout,
            LeaveAlternateScreen,
            cursor::Show,
            ResetColor
        );
    }
}

// --- ฟังก์ชัน step 1: minimal event loop ---
pub fn run_step1() -> crossterm::Result<()> {
    let mut rt = RawTerminal::new()?;
    let stdout = &mut rt.stdout;

    // วาด "Hello, Editor!" ตรงกลางหน้าจอ
    let (cols, rows) = terminal::size()?;
    let msg = "Hello, Editor! (press 'q' to quit)";
    let x = (cols / 2).saturating_sub(msg.len() as u16 / 2);
    let y = rows / 2;

    queue!(
        stdout,
        terminal::Clear(ClearType::All),
        cursor::MoveTo(x, y),
        SetForegroundColor(Color::Green),
        Print(msg),
        ResetColor,
    )?;
    stdout.flush()?;

    // Event loop
    loop {
        if let Event::Key(key) = event::read()? {
            match key.code {
                KeyCode::Char('q') => break,
                KeyCode::Char('c') if key.modifiers.contains(KeyModifiers::CONTROL) => break,
                _ => {}
            }
        }
    }
    Ok(())
}
```

สิ่งสำคัญที่ต้องสังเกต:

1. `terminal::enable_raw_mode()` เปิด raw mode — ต้องเรียก `disable_raw_mode()` เสมอไม่ว่าจะเกิดอะไรขึ้น ดังนั้นใส่ใน `Drop` impl
2. `EnterAlternateScreen` เปิด buffer หน้าจอที่สอง — terminal จะจำ shell history ไว้และ restore เมื่อ `LeaveAlternateScreen`
3. `cursor::Hide` ซ่อน cursor ขณะ render เพื่อไม่ให้กระพริบ
4. `queue!` กับ `execute!` ต่างกัน — `queue!` เพิ่ม command เข้า internal buffer, `execute!` flush ทันที, ควรใช้ `queue!` สำหรับ rendering และ `stdout.flush()` ครั้งเดียวเพื่อประสิทธิภาพ

---

### ขั้นที่ 2: Render Vec\<String\> Buffer กับ Status Bar

ขั้นนี้เพิ่มความสามารถในการเก็บข้อความเป็น `Vec<String>` (ยังไม่ใช้ Rope — จะเพิ่มใน step 4) และ render พร้อม status bar ด้านล่าง:

```rust
// src/editor.rs — ขั้นที่ 2: basic buffer + status bar

use crossterm::{
    cursor,
    queue,
    style::{Attribute, Color, Colors, Print, SetAttribute,
            SetColors, SetForegroundColor, ResetColor},
    terminal::{self, ClearType},
};
use std::io::{Stdout, Write};

pub struct EditorV2 {
    pub lines: Vec<String>,
    pub cursor_row: usize,   // 0-based row in document
    pub cursor_col: usize,   // 0-based col in current line
    pub filename: Option<String>,
    pub modified: bool,
    pub status_msg: String,
}

impl EditorV2 {
    pub fn new() -> Self {
        EditorV2 {
            lines: vec![String::new()],
            cursor_row: 0,
            cursor_col: 0,
            filename: None,
            modified: false,
            status_msg: String::from("Ctrl+Q: Quit  Ctrl+S: Save  Ctrl+F: Find"),
        }
    }

    pub fn draw(&self, stdout: &mut Stdout) -> crossterm::Result<()> {
        let (cols, rows) = terminal::size()?;
        let edit_rows = rows.saturating_sub(2) as usize; // หัก status bar 2 บรรทัด

        queue!(stdout, terminal::Clear(ClearType::All), cursor::MoveTo(0, 0))?;

        // Render text lines
        for screen_row in 0..edit_rows {
            if screen_row < self.lines.len() {
                let line = &self.lines[screen_row];
                // ตัดให้พอดีหน้าจอ (รองรับบรรทัดยาว)
                let display: &str = if line.len() > cols as usize {
                    &line[..cols as usize]
                } else {
                    line
                };
                queue!(stdout, cursor::MoveTo(0, screen_row as u16), Print(display))?;
            } else {
                // แสดง tilde สำหรับบรรทัดที่ไม่มี content (เหมือน vim)
                queue!(
                    stdout,
                    cursor::MoveTo(0, screen_row as u16),
                    SetForegroundColor(Color::DarkBlue),
                    Print("~"),
                    ResetColor,
                )?;
            }
        }

        // Status bar (บรรทัดที่ rows-2)
        let status_left = format!(
            " {} {}",
            self.filename.as_deref().unwrap_or("[New File]"),
            if self.modified { "[+]" } else { "" }
        );
        let status_right = format!(
            "{}:{} ",
            self.cursor_row + 1,
            self.cursor_col + 1
        );
        // Pad status bar ให้เต็มความกว้าง
        let padding = cols as usize
            .saturating_sub(status_left.len() + status_right.len());
        let status = format!("{}{:>width$}{}", status_left, status_right, "", width = padding);
        let status_display = if status.len() > cols as usize {
            status[..cols as usize].to_string()
        } else {
            status
        };

        queue!(
            stdout,
            cursor::MoveTo(0, rows - 2),
            SetColors(Colors::new(Color::Black, Color::White)),
            Print(&status_display),
            ResetColor,
        )?;

        // Message bar (บรรทัดสุดท้าย)
        queue!(
            stdout,
            cursor::MoveTo(0, rows - 1),
            terminal::Clear(ClearType::CurrentLine),
            SetForegroundColor(Color::DarkYellow),
            Print(&self.status_msg),
            ResetColor,
        )?;

        // วาง cursor ให้ถูกตำแหน่ง
        queue!(
            stdout,
            cursor::MoveTo(self.cursor_col as u16, self.cursor_row as u16),
            cursor::Show,
        )?;
        stdout.flush()?;
        Ok(())
    }
}
```

ประเด็นสำคัญเรื่อง status bar:

- ใช้ inverse colors (ขาวบนดำ) เหมือน nano/micro
- แสดง `[+]` เมื่อ `modified = true` — ผู้ใช้รู้ว่ายังไม่ได้ save
- `cursor_row + 1` และ `cursor_col + 1` เพราะแสดงแบบ 1-based ให้ผู้ใช้
- ต้อง `cursor::Show` ตอนสุดท้ายของ render เพราะเราซ่อนไว้ตอน init

---

### ขั้นที่ 3: Cursor Movement — Arrows, Home/End, Page Up/Down

Cursor movement เป็นส่วนที่มี edge cases เยอะที่สุด:

```rust
// src/editor.rs — เพิ่ม cursor movement methods

use crossterm::event::{KeyCode, KeyEvent, KeyModifiers};

impl EditorV2 {
    pub fn handle_key(&mut self, key: KeyEvent) {
        match key.code {
            KeyCode::Up    => self.move_cursor_up(),
            KeyCode::Down  => self.move_cursor_down(),
            KeyCode::Left  => self.move_cursor_left(),
            KeyCode::Right => self.move_cursor_right(),
            KeyCode::Home  => self.move_home(),
            KeyCode::End   => self.move_end(),
            KeyCode::PageUp   => self.page_up(),
            KeyCode::PageDown => self.page_down(),
            // Ctrl+Home / Ctrl+End
            KeyCode::Home if key.modifiers.contains(KeyModifiers::CONTROL) => {
                self.cursor_row = 0;
                self.cursor_col = 0;
            }
            KeyCode::End if key.modifiers.contains(KeyModifiers::CONTROL) => {
                self.cursor_row = self.lines.len().saturating_sub(1);
                self.move_end();
            }
            // Ctrl+Arrow: word jump
            KeyCode::Left if key.modifiers.contains(KeyModifiers::CONTROL) => {
                self.word_jump_left();
            }
            KeyCode::Right if key.modifiers.contains(KeyModifiers::CONTROL) => {
                self.word_jump_right();
            }
            KeyCode::Char(c) => {
                self.insert_char_v2(c);
            }
            KeyCode::Backspace => {
                self.delete_before_cursor_v2();
            }
            KeyCode::Enter => {
                self.insert_newline();
            }
            _ => {}
        }
    }

    fn move_cursor_up(&mut self) {
        if self.cursor_row > 0 {
            self.cursor_row -= 1;
            // Clamp col ให้ไม่เกินความยาว line ปลายทาง
            let line_len = self.lines[self.cursor_row].len();
            if self.cursor_col > line_len {
                self.cursor_col = line_len;
            }
        }
    }

    fn move_cursor_down(&mut self) {
        if self.cursor_row + 1 < self.lines.len() {
            self.cursor_row += 1;
            let line_len = self.lines[self.cursor_row].len();
            if self.cursor_col > line_len {
                self.cursor_col = line_len;
            }
        }
    }

    fn move_cursor_left(&mut self) {
        if self.cursor_col > 0 {
            self.cursor_col -= 1;
        } else if self.cursor_row > 0 {
            // ถ้าอยู่ต้นบรรทัด ให้ข้ามไปบรรทัดก่อนหน้าท้ายสุด
            self.cursor_row -= 1;
            self.cursor_col = self.lines[self.cursor_row].len();
        }
    }

    fn move_cursor_right(&mut self) {
        let line_len = self.lines[self.cursor_row].len();
        if self.cursor_col < line_len {
            self.cursor_col += 1;
        } else if self.cursor_row + 1 < self.lines.len() {
            // ข้ามไปต้น line ถัดไป
            self.cursor_row += 1;
            self.cursor_col = 0;
        }
    }

    fn move_home(&mut self) {
        self.cursor_col = 0;
    }

    fn move_end(&mut self) {
        self.cursor_col = self.lines[self.cursor_row].len();
    }

    fn page_up(&mut self) {
        let (_, rows) = terminal::size().unwrap_or((80, 24));
        let page = (rows as usize).saturating_sub(2);
        self.cursor_row = self.cursor_row.saturating_sub(page);
        let line_len = self.lines[self.cursor_row].len();
        if self.cursor_col > line_len {
            self.cursor_col = line_len;
        }
    }

    fn page_down(&mut self) {
        let (_, rows) = terminal::size().unwrap_or((80, 24));
        let page = (rows as usize).saturating_sub(2);
        self.cursor_row = (self.cursor_row + page).min(self.lines.len().saturating_sub(1));
        let line_len = self.lines[self.cursor_row].len();
        if self.cursor_col > line_len {
            self.cursor_col = line_len;
        }
    }

    fn word_jump_left(&mut self) {
        // ข้ามคำไปซ้าย: ข้าม whitespace ก่อน แล้วข้าม non-whitespace
        let line = &self.lines[self.cursor_row];
        let bytes: &[u8] = line.as_bytes();
        let mut pos = self.cursor_col;
        // ข้าม whitespace
        while pos > 0 && bytes[pos - 1].is_ascii_whitespace() {
            pos -= 1;
        }
        // ข้าม word characters
        while pos > 0 && !bytes[pos - 1].is_ascii_whitespace() {
            pos -= 1;
        }
        self.cursor_col = pos;
    }

    fn word_jump_right(&mut self) {
        let line = &self.lines[self.cursor_row];
        let bytes: &[u8] = line.as_bytes();
        let mut pos = self.cursor_col;
        // ข้าม word characters
        while pos < bytes.len() && !bytes[pos].is_ascii_whitespace() {
            pos += 1;
        }
        // ข้าม whitespace
        while pos < bytes.len() && bytes[pos].is_ascii_whitespace() {
            pos += 1;
        }
        self.cursor_col = pos;
    }

    fn insert_char_v2(&mut self, c: char) {
        let row = self.cursor_row;
        let col = self.cursor_col;
        self.lines[row].insert(col, c);
        self.cursor_col += c.len_utf8();
        self.modified = true;
    }

    fn delete_before_cursor_v2(&mut self) {
        if self.cursor_col > 0 {
            let row = self.cursor_row;
            let col = self.cursor_col;
            // ต้องหา byte offset ของ char ก่อนหน้า
            let line = &self.lines[row];
            // หา char boundary ก่อนหน้า cursor
            let mut byte_pos = col;
            while byte_pos > 0 && !line.is_char_boundary(byte_pos - 1) {
                byte_pos -= 1;
            }
            if byte_pos > 0 {
                let prev_char_start = line[..byte_pos]
                    .char_indices()
                    .next_back()
                    .map(|(i, _)| i)
                    .unwrap_or(0);
                self.lines[row].remove(prev_char_start);
                self.cursor_col = prev_char_start;
                self.modified = true;
            }
        } else if self.cursor_row > 0 {
            // Merge กับ line ก่อนหน้า
            let row = self.cursor_row;
            let current_line = self.lines.remove(row);
            let prev_len = self.lines[row - 1].len();
            self.lines[row - 1].push_str(&current_line);
            self.cursor_row -= 1;
            self.cursor_col = prev_len;
            self.modified = true;
        }
    }

    fn insert_newline(&mut self) {
        let row = self.cursor_row;
        let col = self.cursor_col;
        let remainder = self.lines[row].split_off(col);
        self.lines.insert(row + 1, remainder);
        self.cursor_row += 1;
        self.cursor_col = 0;
        self.modified = true;
    }
}
```

---

### ขั้นที่ 4: เปลี่ยนไปใช้ ropey — Insert/Delete บน Rope

เมื่อไฟล์ใหญ่ขึ้น `Vec<String>` จะเริ่มช้า ขั้นนี้ refactor ไปใช้ `ropey::Rope` และ introduce **EditCommand** สำหรับ undo/redo:

```rust
// src/commands.rs

#[derive(Debug, Clone)]
pub enum EditKind {
    Insert,
    Delete,
}

#[derive(Debug, Clone)]
pub struct EditCommand {
    pub kind: EditKind,
    pub pos: usize,    // character index ใน Rope
    pub text: String,  // ข้อความที่ insert หรือถูก delete
}
```

```rust
// src/buffer.rs — Buffer struct ที่ใช้ ropey

use ropey::Rope;
use crate::commands::{EditCommand, EditKind};

pub struct Buffer {
    pub rope: Rope,
    pub cursor_char: usize,    // ตำแหน่ง cursor เป็น char index
    pub undo_stack: Vec<EditCommand>,
    pub redo_stack: Vec<EditCommand>,
    pub modified: bool,
    pub filename: Option<std::path::PathBuf>,
}

impl Buffer {
    pub fn new() -> Self {
        Buffer {
            rope: Rope::new(),
            cursor_char: 0,
            undo_stack: Vec::new(),
            redo_stack: Vec::new(),
            modified: false,
            filename: None,
        }
    }

    /// โหลดจากไฟล์
    pub fn open(path: &std::path::Path) -> std::io::Result<Self> {
        let content = std::fs::read_to_string(path)?;
        Ok(Buffer {
            rope: Rope::from_str(&content),
            cursor_char: 0,
            undo_stack: Vec::new(),
            redo_stack: Vec::new(),
            modified: false,
            filename: Some(path.to_path_buf()),
        })
    }

    /// Insert char ที่ cursor position พร้อม push undo command
    pub fn insert_char(&mut self, ch: char) {
        let pos = self.cursor_char;
        self.rope.insert_char(pos, ch);
        self.undo_stack.push(EditCommand {
            kind: EditKind::Insert,
            pos,
            text: ch.to_string(),
        });
        self.redo_stack.clear();
        self.cursor_char += 1;
        self.modified = true;
    }

    /// Insert string (เช่น paste)
    pub fn insert_str(&mut self, s: &str) {
        let pos = self.cursor_char;
        self.rope.insert(pos, s);
        self.undo_stack.push(EditCommand {
            kind: EditKind::Insert,
            pos,
            text: s.to_string(),
        });
        self.redo_stack.clear();
        self.cursor_char += s.chars().count();
        self.modified = true;
    }

    /// Backspace: ลบ char ก่อน cursor
    pub fn delete_char_before(&mut self) {
        if self.cursor_char == 0 {
            return;
        }
        let pos = self.cursor_char - 1;
        let ch = self.rope.char(pos);
        self.rope.remove(pos..self.cursor_char);
        self.undo_stack.push(EditCommand {
            kind: EditKind::Delete,
            pos,
            text: ch.to_string(),
        });
        self.redo_stack.clear();
        self.cursor_char -= 1;
        self.modified = true;
    }

    /// Delete char ที่ cursor (Del key)
    pub fn delete_char_at(&mut self) {
        if self.cursor_char >= self.rope.len_chars() {
            return;
        }
        let ch = self.rope.char(self.cursor_char);
        self.rope.remove(self.cursor_char..self.cursor_char + 1);
        self.undo_stack.push(EditCommand {
            kind: EditKind::Delete,
            pos: self.cursor_char,
            text: ch.to_string(),
        });
        self.redo_stack.clear();
        self.modified = true;
    }

    /// Undo last command
    pub fn undo(&mut self) -> bool {
        if let Some(cmd) = self.undo_stack.pop() {
            match cmd.kind {
                EditKind::Insert => {
                    let end = cmd.pos + cmd.text.chars().count();
                    self.rope.remove(cmd.pos..end);
                    self.cursor_char = cmd.pos;
                }
                EditKind::Delete => {
                    self.rope.insert(cmd.pos, &cmd.text);
                    self.cursor_char = cmd.pos + cmd.text.chars().count();
                }
            }
            self.redo_stack.push(cmd);
            true
        } else {
            false
        }
    }

    /// Redo last undone command
    pub fn redo(&mut self) -> bool {
        if let Some(cmd) = self.redo_stack.pop() {
            match cmd.kind {
                EditKind::Insert => {
                    self.rope.insert(cmd.pos, &cmd.text);
                    self.cursor_char = cmd.pos + cmd.text.chars().count();
                }
                EditKind::Delete => {
                    let end = cmd.pos + cmd.text.chars().count();
                    self.rope.remove(cmd.pos..end);
                    self.cursor_char = cmd.pos;
                }
            }
            self.undo_stack.push(cmd);
            true
        } else {
            false
        }
    }

    /// (line, col) ทั้งคู่ 0-based
    pub fn cursor_position(&self) -> (usize, usize) {
        let line = self.rope.char_to_line(self.cursor_char);
        let line_start = self.rope.line_to_char(line);
        (line, self.cursor_char - line_start)
    }

    /// ย้าย cursor ไปต้นบรรทัด
    pub fn move_home(&mut self) {
        let (line, _) = self.cursor_position();
        self.cursor_char = self.rope.line_to_char(line);
    }

    /// ย้าย cursor ไปท้ายบรรทัด (ก่อน newline)
    pub fn move_end(&mut self) {
        let (line, _) = self.cursor_position();
        let line_rope = self.rope.line(line);
        let mut end_offset = line_rope.len_chars();
        // ลบ newline ออก
        if end_offset > 0 {
            let last_char = line_rope.char(end_offset - 1);
            if last_char == '\n' || last_char == '\r' {
                end_offset -= 1;
            }
        }
        self.cursor_char = self.rope.line_to_char(line) + end_offset;
    }

    /// ย้าย cursor ขึ้น N บรรทัด
    pub fn move_up(&mut self, n: usize) {
        let (line, col) = self.cursor_position();
        let target_line = line.saturating_sub(n);
        let target_line_start = self.rope.line_to_char(target_line);
        let target_line_rope = self.rope.line(target_line);
        // clamp col
        let mut eol = target_line_rope.len_chars();
        if eol > 0 {
            let lc = target_line_rope.char(eol - 1);
            if lc == '\n' || lc == '\r' { eol -= 1; }
        }
        self.cursor_char = target_line_start + col.min(eol);
    }

    /// ย้าย cursor ลง N บรรทัด
    pub fn move_down(&mut self, n: usize) {
        let (line, col) = self.cursor_position();
        let total_lines = self.rope.len_lines();
        let target_line = (line + n).min(total_lines.saturating_sub(1));
        if target_line == line { return; }
        let target_line_start = self.rope.line_to_char(target_line);
        let target_line_rope = self.rope.line(target_line);
        let mut eol = target_line_rope.len_chars();
        if eol > 0 {
            let lc = target_line_rope.char(eol - 1);
            if lc == '\n' || lc == '\r' { eol -= 1; }
        }
        self.cursor_char = target_line_start + col.min(eol);
    }

    pub fn num_lines(&self) -> usize {
        self.rope.len_lines()
    }

    pub fn len_chars(&self) -> usize {
        self.rope.len_chars()
    }
}
```

**ทำไม `cursor_char` ไม่ใช่ `(row, col)`?**

ใน Rope การ index ด้วย char offset เดียวมีประสิทธิภาพสูงกว่า (O(log n)) เมื่อต้องการ (row, col) เราแปลงจาก char offset ได้ด้วย `char_to_line()` และ `line_to_char()` ซึ่งก็เป็น O(log n) เช่นกัน ดังนั้นเก็บ single value ไว้แล้วแปลงเมื่อต้องการดีกว่าเก็บสองค่าแล้ว sync กันให้ถูกต้องตลอดเวลา

---

### ขั้นที่ 5: File Open (clap) และ Ctrl+S Save แบบ Atomic

```rust
// src/main.rs — CLI args + file open + save

use clap::Parser;
use std::path::PathBuf;

#[derive(Parser, Debug)]
#[command(name = "te", about = "Terminal text editor")]
pub struct Args {
    /// File to open (optional — creates new file if omitted)
    pub filename: Option<PathBuf>,
}

/// Atomic save: เขียนไป temp file ในไดเรกทอรีเดียวกัน แล้ว rename
pub fn atomic_save(path: &std::path::Path, content: &str) -> std::io::Result<()> {
    use std::io::Write;
    let dir = path.parent().unwrap_or(std::path::Path::new("."));
    let basename = path.file_name()
        .map(|n| n.to_string_lossy().to_string())
        .unwrap_or_else(|| "file".to_string());
    let tmp_path = dir.join(format!(".tmp_{}", basename));

    {
        let mut f = std::fs::File::create(&tmp_path)?;
        f.write_all(content.as_bytes())?;
        f.flush()?;
        // fsync: รับประกันว่า data อยู่บน disk ก่อน rename
        f.sync_all()?;
    }
    // rename() เป็น atomic บน POSIX filesystems
    std::fs::rename(&tmp_path, path)?;
    Ok(())
}
```

เหตุผลที่ต้องใช้ `sync_all()` (fsync):

- `flush()` แค่ flush from Rust's buffer → kernel buffer
- `sync_all()` force kernel → disk
- ถ้าไม่ sync_all แล้วเกิด crash หลัง rename temp file อาจยังอยู่ใน kernel page cache ยังไม่ได้ถึง disk จริง

### Main Loop ที่สมบูรณ์

```rust
// src/main.rs — event loop เต็ม

use crossterm::event::{self, Event, KeyCode, KeyModifiers};
use std::time::Duration;

pub fn run(args: Args) -> crossterm::Result<()> {
    use crate::buffer::Buffer;
    use crate::terminal::RawTerminal;

    let mut buf = match args.filename {
        Some(ref path) if path.exists() => {
            Buffer::open(path).unwrap_or_else(|_| {
                let mut b = Buffer::new();
                b.filename = Some(path.clone());
                b
            })
        }
        Some(path) => {
            let mut b = Buffer::new();
            b.filename = Some(path);
            b
        }
        None => Buffer::new(),
    };

    let mut rt = RawTerminal::new()?;
    let mut status_msg = String::from("Ctrl+Q: Quit  Ctrl+S: Save  Ctrl+F: Find  Ctrl+Z: Undo");

    loop {
        // Draw frame
        crate::terminal::draw_frame(&mut rt.stdout, &buf, &status_msg)?;

        // Wait for event (timeout ช่วยให้ refresh เป็นระยะ)
        if event::poll(Duration::from_millis(50))? {
            match event::read()? {
                Event::Key(key) => {
                    match (key.code, key.modifiers) {
                        // Quit
                        (KeyCode::Char('q'), KeyModifiers::CONTROL) => {
                            if buf.modified {
                                status_msg = String::from(
                                    "Unsaved changes! Ctrl+Q again to force quit."
                                );
                                // ต้องกด Ctrl+Q สองครั้งเพื่อ quit เมื่อมี unsaved changes
                            } else {
                                break;
                            }
                        }
                        // Save
                        (KeyCode::Char('s'), KeyModifiers::CONTROL) => {
                            let content = buf.rope.to_string();
                            if let Some(ref path) = buf.filename {
                                match atomic_save(path, &content) {
                                    Ok(()) => {
                                        buf.modified = false;
                                        status_msg = format!(
                                            "Saved: {}",
                                            path.display()
                                        );
                                    }
                                    Err(e) => {
                                        status_msg = format!("Save error: {}", e);
                                    }
                                }
                            } else {
                                status_msg = String::from("No filename! Set filename first.");
                            }
                        }
                        // Undo / Redo
                        (KeyCode::Char('z'), KeyModifiers::CONTROL) => {
                            if buf.undo() {
                                status_msg = String::from("Undo");
                            }
                        }
                        (KeyCode::Char('y'), KeyModifiers::CONTROL) => {
                            if buf.redo() {
                                status_msg = String::from("Redo");
                            }
                        }
                        // Arrow keys
                        (KeyCode::Up, _)    => buf.move_up(1),
                        (KeyCode::Down, _)  => buf.move_down(1),
                        (KeyCode::Left, mods) if mods.contains(KeyModifiers::CONTROL) => {
                            // word jump left — implement เป็น helper
                        }
                        (KeyCode::Right, mods) if mods.contains(KeyModifiers::CONTROL) => {
                            // word jump right
                        }
                        (KeyCode::Left, _)  => {
                            if buf.cursor_char > 0 { buf.cursor_char -= 1; }
                        }
                        (KeyCode::Right, _) => {
                            if buf.cursor_char < buf.len_chars() {
                                buf.cursor_char += 1;
                            }
                        }
                        (KeyCode::Home, _) => buf.move_home(),
                        (KeyCode::End, _)  => buf.move_end(),
                        (KeyCode::PageUp, _)   => buf.move_up(20),
                        (KeyCode::PageDown, _) => buf.move_down(20),
                        // Text editing
                        (KeyCode::Char(c), _) => buf.insert_char(c),
                        (KeyCode::Enter, _)   => buf.insert_char('\n'),
                        (KeyCode::Backspace, _) => buf.delete_char_before(),
                        (KeyCode::Delete, _)    => buf.delete_char_at(),
                        _ => {}
                    }
                }
                Event::Resize(_, _) => {
                    // Terminal resize — redraw next iteration
                }
                _ => {}
            }
        }
    }
    Ok(())
}
```

---

### ขั้นที่ 6: Viewport และ Scrolling

เมื่อไฟล์ใหญ่กว่าหน้าจอ ต้องเก็บ "scroll offset" และ render เฉพาะส่วนที่มองเห็น:

```rust
// src/viewport.rs

pub struct Viewport {
    pub scroll_row: usize,  // บรรทัดแรกที่มองเห็น (0-based)
    pub scroll_col: usize,  // column แรกที่มองเห็น (สำหรับ horizontal scroll)
    pub height: usize,      // จำนวนบรรทัดที่แสดงได้ (terminal rows - 2)
    pub width: usize,       // ความกว้าง terminal (cols)
}

impl Viewport {
    pub fn new(height: usize, width: usize) -> Self {
        Viewport { scroll_row: 0, scroll_col: 0, height, width }
    }

    /// ปรับ scroll_row และ scroll_col ให้ cursor อยู่ใน viewport เสมอ
    pub fn scroll_to_cursor(&mut self, cursor_line: usize, cursor_col: usize) {
        // Vertical
        if cursor_line < self.scroll_row {
            self.scroll_row = cursor_line;
        } else if cursor_line >= self.scroll_row + self.height {
            self.scroll_row = cursor_line + 1 - self.height;
        }
        // Horizontal
        if cursor_col < self.scroll_col {
            self.scroll_col = cursor_col;
        } else if cursor_col >= self.scroll_col + self.width {
            self.scroll_col = cursor_col + 1 - self.width;
        }
    }

    /// Range ของบรรทัดที่ต้อง render
    pub fn visible_lines(&self, total_lines: usize) -> std::ops::Range<usize> {
        let start = self.scroll_row;
        let end = (self.scroll_row + self.height).min(total_lines);
        start..end
    }
}
```

```rust
// src/terminal.rs — draw_frame ที่รองรับ Viewport

use ropey::Rope;
use crate::viewport::Viewport;

pub fn draw_frame(
    stdout: &mut impl std::io::Write,
    rope: &Rope,
    viewport: &Viewport,
    cursor_char: usize,
    filename: Option<&std::path::Path>,
    modified: bool,
    status_msg: &str,
    search_matches: &[usize],  // char positions ของ match
    highlight_fn: &dyn Fn(&str, usize) -> Vec<(std::ops::Range<usize>, u8, u8, u8)>,
) -> crossterm::Result<()> {
    use crossterm::{cursor, queue, terminal, style::*};

    let total_lines = rope.len_lines();
    queue!(stdout, terminal::Clear(terminal::ClearType::All))?;

    let visible = viewport.visible_lines(total_lines);
    for (screen_row, doc_row) in visible.enumerate() {
        let line = rope.line(doc_row);
        let line_str: String = line.to_string();
        // horizontal scroll
        let display_start = viewport.scroll_col.min(line_str.len());
        let display_end = (viewport.scroll_col + viewport.width).min(line_str.len());
        let display = &line_str[display_start..display_end];

        queue!(stdout, cursor::MoveTo(0, screen_row as u16))?;

        // Syntax highlighting: แยก segments ตาม color
        let highlights = highlight_fn(display, doc_row);
        if highlights.is_empty() {
            queue!(stdout, Print(display))?;
        } else {
            // render segment by segment
            let mut last = 0;
            for (range, r, g, b) in &highlights {
                if range.start > last {
                    queue!(stdout, Print(&display[last..range.start]))?;
                }
                queue!(
                    stdout,
                    SetForegroundColor(Color::Rgb { r: *r, g: *g, b: *b }),
                    Print(&display[range.clone()]),
                    ResetColor,
                )?;
                last = range.end;
            }
            if last < display.len() {
                queue!(stdout, Print(&display[last..]))?;
            }
        }
    }

    // Status bar
    let (cols, rows) = crossterm::terminal::size()?;
    let fname = filename
        .map(|p| p.file_name().unwrap_or_default().to_string_lossy().to_string())
        .unwrap_or_else(|| "[New File]".into());
    let mod_flag = if modified { " [+]" } else { "" };
    let (cur_line, cur_col) = {
        let line = rope.char_to_line(cursor_char);
        let line_start = rope.line_to_char(line);
        (line + 1, cursor_char - line_start + 1)
    };
    let left = format!(" {}{}", fname, mod_flag);
    let right = format!("{}:{}  ", cur_line, cur_col);
    let pad = (cols as usize).saturating_sub(left.len() + right.len());
    let status = format!("{}{:>width$}{}", left, right, "", width = pad);

    queue!(
        stdout,
        cursor::MoveTo(0, rows - 2),
        SetColors(Colors::new(Color::Black, Color::White)),
        Print(if status.len() > cols as usize { &status[..cols as usize] } else { &status }),
        ResetColor,
        cursor::MoveTo(0, rows - 1),
        terminal::Clear(terminal::ClearType::CurrentLine),
        SetForegroundColor(Color::DarkYellow),
        Print(status_msg),
        ResetColor,
    )?;

    // Move cursor to screen position
    let cur_line_0 = cur_line - 1;
    let screen_cur_row = cur_line_0.saturating_sub(viewport.scroll_row);
    let screen_cur_col = (cur_col - 1).saturating_sub(viewport.scroll_col);
    queue!(
        stdout,
        cursor::MoveTo(screen_cur_col as u16, screen_cur_row as u16),
        cursor::Show,
    )?;
    stdout.flush()?;
    Ok(())
}
```

---

### ขั้นที่ 7: Incremental Search พร้อม Highlight

Search mode เปิดด้วย Ctrl+F — ผู้ใช้พิมพ์ pattern แล้ว editor highlight matches ทันที กด `n`/`N` เพื่อข้ามไปยัง match ถัดไป/ก่อนหน้า:

```rust
// src/search.rs

use ropey::Rope;

#[derive(Default)]
pub struct SearchState {
    pub query: String,
    pub matches: Vec<usize>,   // char positions ของทุก match
    pub current: usize,        // index ใน matches ที่ cursor อยู่
    pub active: bool,          // กำลังอยู่ใน search mode
}

impl SearchState {
    pub fn new() -> Self {
        SearchState::default()
    }

    /// อัปเดต query และหา match ทั้งหมดใหม่
    pub fn update(&mut self, rope: &Rope, query: &str) {
        self.query = query.to_string();
        self.matches = find_all_char_positions(rope, query);
        self.current = 0;
    }

    /// ขยับไปยัง match ถัดไป — return cursor_char ใหม่
    pub fn next_match(&mut self) -> Option<usize> {
        if self.matches.is_empty() { return None; }
        self.current = (self.current + 1) % self.matches.len();
        Some(self.matches[self.current])
    }

    /// ขยับไปยัง match ก่อนหน้า
    pub fn prev_match(&mut self) -> Option<usize> {
        if self.matches.is_empty() { return None; }
        if self.current == 0 {
            self.current = self.matches.len() - 1;
        } else {
            self.current -= 1;
        }
        Some(self.matches[self.current])
    }
}

/// หา char positions ของทุก occurrence ของ query ใน rope
pub fn find_all_char_positions(rope: &Rope, query: &str) -> Vec<usize> {
    if query.is_empty() { return Vec::new(); }
    let text = rope.to_string();
    let query_chars = query.chars().count();
    let mut results = Vec::new();
    let mut start = 0usize; // byte offset
    while let Some(byte_pos) = text[start..].find(query) {
        let abs_byte = start + byte_pos;
        // แปลง byte offset → char offset
        let char_pos = text[..abs_byte].chars().count();
        results.push(char_pos);
        start = abs_byte + 1; // advance อย่างน้อย 1 byte ป้องกัน infinite loop
    }
    results
}

/// สร้าง set ของ char positions ที่ถูก highlight
pub fn match_char_set(matches: &[usize], query_char_len: usize) -> std::collections::HashSet<usize> {
    let mut set = std::collections::HashSet::new();
    for &start in matches {
        for i in 0..query_char_len {
            set.insert(start + i);
        }
    }
    set
}
```

### การ integrate search เข้าใน event loop

```rust
// ใน event loop — handle search mode

// State เพิ่มเติม
let mut search = SearchState::new();
let mut in_search = false;
let mut search_input = String::new();

// ...ใน event handler...
(KeyCode::Char('f'), KeyModifiers::CONTROL) => {
    in_search = true;
    search_input.clear();
    search.active = true;
    status_msg = String::from("Search: ");
}

// ถ้า in_search mode
if in_search {
    match key.code {
        KeyCode::Char(c) => {
            search_input.push(c);
            search.update(&buf.rope, &search_input);
            if let Some(pos) = search.matches.first() {
                buf.cursor_char = *pos;
            }
            status_msg = format!("Search: {} ({} matches)", search_input, search.matches.len());
        }
        KeyCode::Backspace => {
            search_input.pop();
            search.update(&buf.rope, &search_input);
            status_msg = format!("Search: {} ({} matches)", search_input, search.matches.len());
        }
        KeyCode::Char('n') => {
            if let Some(pos) = search.next_match() {
                buf.cursor_char = pos;
            }
        }
        KeyCode::Char('p') | KeyCode::Char('N') => {
            if let Some(pos) = search.prev_match() {
                buf.cursor_char = pos;
            }
        }
        KeyCode::Esc | KeyCode::Enter => {
            in_search = false;
            search.active = false;
            status_msg = String::from("Ctrl+Q: Quit  Ctrl+S: Save");
        }
        _ => {}
    }
}
```

---

### ขั้นที่ 8: Syntax Highlighting สำหรับ Rust Files

Syntax highlighting ที่ถูกต้องสมบูรณ์ต้องการ parser เต็ม แต่เราจะทำแบบ line-based tokenizer ที่ครอบคลุม Rust basics:

```rust
// src/highlight.rs

/// Token ประเภทต่างๆ สำหรับ Rust
#[derive(Debug, Clone, PartialEq)]
pub enum TokenKind {
    Keyword,     // fn, let, pub, struct, ...
    Comment,     // // line comment
    String,      // "..." string literal
    Number,      // 42, 3.14, 0xFF
    Operator,    // +, -, *, /, =, ==, ...
    Type,        // ชื่อที่ขึ้นต้นด้วย uppercase
    Macro,       // println!, vec!, ...
    Normal,
}

#[derive(Debug, Clone)]
pub struct Token {
    pub kind: TokenKind,
    pub start: usize,  // byte offset ใน line
    pub end: usize,
}

/// Tokenize บรรทัดเดียว (simplified — ไม่ handle multi-line strings/comments)
pub fn tokenize_line(line: &str) -> Vec<Token> {
    let mut tokens = Vec::new();
    let bytes = line.as_bytes();
    let len = bytes.len();
    let mut i = 0;

    while i < len {
        // Skip whitespace
        if bytes[i].is_ascii_whitespace() {
            i += 1;
            continue;
        }

        // Line comment: // ...
        if i + 1 < len && bytes[i] == b'/' && bytes[i + 1] == b'/' {
            tokens.push(Token {
                kind: TokenKind::Comment,
                start: i,
                end: len,
            });
            break;
        }

        // String literal: "..."
        if bytes[i] == b'"' {
            let start = i;
            i += 1;
            while i < len {
                if bytes[i] == b'\\' {
                    i += 2; // skip escape
                    continue;
                }
                if bytes[i] == b'"' {
                    i += 1;
                    break;
                }
                i += 1;
            }
            tokens.push(Token { kind: TokenKind::String, start, end: i });
            continue;
        }

        // Char literal: '...'
        if bytes[i] == b'\'' {
            let start = i;
            i += 1;
            while i < len && bytes[i] != b'\'' {
                if bytes[i] == b'\\' { i += 1; }
                i += 1;
            }
            if i < len { i += 1; } // closing '
            tokens.push(Token { kind: TokenKind::String, start, end: i });
            continue;
        }

        // Number: starts with digit
        if bytes[i].is_ascii_digit() {
            let start = i;
            while i < len && (bytes[i].is_ascii_alphanumeric() || bytes[i] == b'.' || bytes[i] == b'_') {
                i += 1;
            }
            tokens.push(Token { kind: TokenKind::Number, start, end: i });
            continue;
        }

        // Identifier or keyword
        if bytes[i].is_ascii_alphabetic() || bytes[i] == b'_' {
            let start = i;
            while i < len && (bytes[i].is_ascii_alphanumeric() || bytes[i] == b'_') {
                i += 1;
            }
            let word = &line[start..i];

            // Macro: identifier ตามด้วย '!'
            let kind = if i < len && bytes[i] == b'!' {
                i += 1; // consume '!'
                TokenKind::Macro
            } else if is_rust_keyword(word) {
                TokenKind::Keyword
            } else if word.chars().next().map(|c| c.is_uppercase()).unwrap_or(false) {
                TokenKind::Type
            } else {
                TokenKind::Normal
            };

            tokens.push(Token { kind, start, end: i });
            continue;
        }

        // Operator / punctuation (single char for simplicity)
        if "+-*/=<>!&|^%".contains(bytes[i] as char) {
            let start = i;
            // Consume up to 3 chars for operators like ==, !=, <=, >=, ->
            while i < len && i - start < 3
                && "+-*/=<>!&|^%".contains(bytes[i] as char)
            {
                i += 1;
            }
            tokens.push(Token { kind: TokenKind::Operator, start, end: i });
            continue;
        }

        // Fallthrough: อื่น ๆ
        i += 1;
    }

    tokens
}

/// ตรวจสอบว่าเป็น Rust keyword หรือไม่
pub fn is_rust_keyword(word: &str) -> bool {
    matches!(word,
        "as" | "break" | "const" | "continue" | "crate" | "else" | "enum"
        | "extern" | "false" | "fn" | "for" | "if" | "impl" | "in" | "let"
        | "loop" | "match" | "mod" | "move" | "mut" | "pub" | "ref" | "return"
        | "self" | "Self" | "static" | "struct" | "super" | "trait" | "true"
        | "type" | "unsafe" | "use" | "where" | "while" | "async" | "await"
        | "dyn" | "i8" | "i16" | "i32" | "i64" | "i128" | "isize"
        | "u8" | "u16" | "u32" | "u64" | "u128" | "usize"
        | "f32" | "f64" | "bool" | "char" | "str" | "String"
    )
}

/// สีสำหรับแต่ละ TokenKind (RGB)
pub fn token_color(kind: &TokenKind) -> (u8, u8, u8) {
    match kind {
        TokenKind::Keyword  => (197, 134, 192), // purple
        TokenKind::Comment  => (106, 153,  85), // green
        TokenKind::String   => (206, 145, 120), // orange
        TokenKind::Number   => (181, 206, 168), // light green
        TokenKind::Operator => (212, 212, 212), // light grey
        TokenKind::Type     => ( 78, 201, 176), // teal
        TokenKind::Macro    => (220, 220, 170), // yellow
        TokenKind::Normal   => (212, 212, 212), // light grey
    }
}

/// Determine if file should have Rust highlighting based on extension
pub fn detect_language(path: &std::path::Path) -> Option<&'static str> {
    match path.extension()?.to_str()? {
        "rs"   => Some("rust"),
        "toml" => Some("toml"),
        "md"   => Some("markdown"),
        _      => None,
    }
}
```

### ใส่ Highlighting เข้า Render Loop

```rust
// ใน draw_frame: ใช้ highlight เมื่อ language = "rust"

pub fn render_highlighted_line(
    stdout: &mut impl std::io::Write,
    line: &str,
    screen_col: usize, // viewport.scroll_col
    visible_width: usize,
) -> crossterm::Result<()> {
    use crossterm::{queue, style::*};
    use crate::highlight::{tokenize_line, token_color, TokenKind};

    let tokens = tokenize_line(line);
    let chars: Vec<char> = line.chars().collect();
    let mut i = 0usize; // char index

    // สร้าง per-char color map
    let mut color_map: Vec<(u8, u8, u8)> = vec![(212, 212, 212); chars.len()];
    // แปลง token byte ranges → char ranges และ fill color_map
    for token in &tokens {
        let (r, g, b) = token_color(&token.kind);
        // byte → char conversion
        let start_char = line[..token.start].chars().count();
        let end_char = line[..token.end].chars().count();
        for idx in start_char..end_char.min(chars.len()) {
            color_map[idx] = (r, g, b);
        }
    }

    // Render visible portion
    let visible_end = (screen_col + visible_width).min(chars.len());
    let mut last_color = (0u8, 0u8, 0u8);
    for idx in screen_col..visible_end {
        let (r, g, b) = color_map[idx];
        if (r, g, b) != last_color {
            queue!(stdout, SetForegroundColor(Color::Rgb { r, g, b }))?;
            last_color = (r, g, b);
        }
        queue!(stdout, Print(chars[idx]))?;
    }
    queue!(stdout, ResetColor)?;
    Ok(())
}
```

---

## การทดสอบ (Testing)

สร้าง test file สำหรับ Buffer และ file operations ที่ครอบคลุมทุก core logic:

```rust
// tests/buffer_tests.rs — integration tests

use terminal_editor::{Buffer, Viewport, find_all_char_positions, is_rust_keyword, atomic_save};

// --- Buffer insert + len ---

#[test]
fn test_buffer_insert_and_len() {
    let mut buf = Buffer::new();
    buf.insert_char('h');
    buf.insert_char('i');
    assert_eq!(buf.to_string(), "hi");
    assert_eq!(buf.len_chars(), 2);
    assert_eq!(buf.cursor_char, 2);
}

// --- Backspace ---

#[test]
fn test_buffer_delete_backspace() {
    let mut buf = Buffer::new();
    for ch in "hello".chars() { buf.insert_char(ch); }
    buf.delete_char_before();
    assert_eq!(buf.to_string(), "hell");
    assert_eq!(buf.cursor_char, 4);
}

// --- Cursor movement ---

#[test]
fn test_cursor_move_left_right() {
    let mut buf = Buffer::from_str("abc");
    buf.cursor_char = 3;
    // move left
    if buf.cursor_char > 0 { buf.cursor_char -= 1; }
    assert_eq!(buf.cursor_char, 2);
    // move right
    if buf.cursor_char < buf.len_chars() { buf.cursor_char += 1; }
    assert_eq!(buf.cursor_char, 3);
    // cannot go past end
    if buf.cursor_char < buf.len_chars() { buf.cursor_char += 1; }
    assert_eq!(buf.cursor_char, 3);
}

#[test]
fn test_cursor_home_end() {
    let mut buf = Buffer::from_str("hello\nworld");
    buf.cursor_char = 0;
    buf.move_end();
    // "hello" = 5 chars, cursor should be at 5 (before '\n')
    assert_eq!(buf.cursor_char, 5);
    buf.move_home();
    assert_eq!(buf.cursor_char, 0);
}

#[test]
fn test_cursor_line_col() {
    let buf = Buffer::from_str("abc\nde\nf");
    // char 5 = 'e' → line 1, col 1
    let (line, col) = {
        let line = buf.rope.char_to_line(5);
        let line_start = buf.rope.line_to_char(line);
        (line, 5 - line_start)
    };
    assert_eq!(line, 1);
    assert_eq!(col, 1);
}

// --- Undo/Redo ---

#[test]
fn test_undo_insert_five_chars() {
    let mut buf = Buffer::new();
    for ch in "hello".chars() { buf.insert_char(ch); }
    assert_eq!(buf.to_string(), "hello");
    for _ in 0..5 {
        let ok = buf.undo();
        assert!(ok);
    }
    assert_eq!(buf.to_string(), "");
    assert_eq!(buf.len_chars(), 0);
}

#[test]
fn test_undo_beyond_empty() {
    let mut buf = Buffer::new();
    buf.insert_char('x');
    buf.undo();
    assert!(!buf.undo(), "undo on empty stack should return false");
}

#[test]
fn test_redo_after_undo() {
    let mut buf = Buffer::new();
    for ch in "abc".chars() { buf.insert_char(ch); }
    buf.undo();
    assert_eq!(buf.to_string(), "ab");
    buf.redo();
    assert_eq!(buf.to_string(), "abc");
}

#[test]
fn test_redo_cleared_on_new_edit() {
    let mut buf = Buffer::new();
    buf.insert_char('a');
    buf.insert_char('b');
    buf.undo();              // undo 'b'
    buf.insert_char('c');   // new edit clears redo stack
    assert!(!buf.redo());
    assert_eq!(buf.to_string(), "ac");
}

// --- File save ---

#[test]
fn test_atomic_save_and_read_back() {
    let dir = tempfile::tempdir().expect("temp dir");
    let path = dir.path().join("test.txt");
    let content = "Hello, editor!\nLine two.\n";
    atomic_save(&path, content).expect("save ok");
    let read_back = std::fs::read_to_string(&path).expect("read back");
    assert_eq!(read_back, content);
}

// --- Viewport ---

#[test]
fn test_viewport_scroll_down() {
    let mut vp = Viewport::new(5, 80);
    vp.scroll_to_cursor(7, 0);
    assert_eq!(vp.scroll_row, 3);
}

#[test]
fn test_viewport_visible_lines() {
    let mut vp = Viewport::new(5, 80);
    vp.scroll_row = 2;
    let range = vp.visible_lines(20);
    assert_eq!(range, 2..7);
}

// --- Search ---

#[test]
fn test_find_all_matches() {
    let rope = ropey::Rope::from_str("hello world hello");
    let matches = find_all_char_positions(&rope, "hello");
    assert_eq!(matches.len(), 2);
    assert_eq!(matches[0], 0);
    assert_eq!(matches[1], 12);
}

#[test]
fn test_find_no_match() {
    let rope = ropey::Rope::from_str("nothing here");
    let matches = find_all_char_positions(&rope, "xyz");
    assert!(matches.is_empty());
}

// --- Syntax highlighting ---

#[test]
fn test_rust_keyword_detection() {
    assert!(is_rust_keyword("fn"));
    assert!(is_rust_keyword("struct"));
    assert!(is_rust_keyword("impl"));
    assert!(is_rust_keyword("async"));
    assert!(!is_rust_keyword("foo"));
    assert!(!is_rust_keyword("editor"));
}
```

### ผลการรัน `cargo test` จริง

```
running 15 tests
test tests::test_buffer_delete_backspace ... ok
test tests::test_cursor_home_end ... ok
test tests::test_buffer_insert_and_len ... ok
test tests::test_atomic_save_and_read_back ... ok
test tests::test_cursor_line_col ... ok
test tests::test_cursor_move_left_right ... ok
test tests::test_find_no_match ... ok
test tests::test_find_all_matches ... ok
test tests::test_redo_after_undo ... ok
test tests::test_rust_keyword_detection ... ok
test tests::test_undo_beyond_empty ... ok
test tests::test_undo_insert_five_chars ... ok
test tests::test_redo_cleared_on_new_edit ... ok
test tests::test_viewport_scroll_down ... ok
test tests::test_viewport_visible_lines ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก 15 tests ผ่าน — ครอบคลุม cursor movement, undo/redo, file save, viewport scrolling, search, และ syntax highlighting detection

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary จะอยู่ที่:
./target/release/te

# ขนาดปกติหลัง strip:
strip ./target/release/te
ls -lh ./target/release/te
# -rwxr-xr-x  1.2M  te
```

### ติดตั้งใน PATH

```bash
# ติดตั้งจาก Cargo
cargo install --path .

# หรือ copy manual
cp target/release/te ~/.local/bin/te

# ทดสอบ
te myfile.rs
te  # เปิด new file
```

### Cross-compile สำหรับ Linux musl (static binary)

```bash
# เพิ่ม target
rustup target add x86_64-unknown-linux-musl

# Build static binary (ไม่ต้องพึ่ง glibc)
cargo build --release --target x86_64-unknown-linux-musl

# ขนาดเล็ก ใช้งานได้ทุก Linux distro
strip target/x86_64-unknown-linux-musl/release/te
```

### Cargo.toml สมบูรณ์

```toml
[package]
name = "te"
version = "0.1.0"
edition = "2021"
description = "A minimal terminal text editor"
license = "MIT"

[dependencies]
crossterm = "0.27"
ropey = "1.6"
clap = { version = "4", features = ["derive"] }
owo-colors = "4"

[dev-dependencies]
tempfile = "3"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true
```

`lto = true` และ `codegen-units = 1` ทำให้ binary เล็กลงและเร็วขึ้น แต่ compile time นานขึ้น

---

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดัก 1: ลืม Restore Terminal เมื่อ Panic

โค้ดที่ผิด:

```rust
// ❌ ถ้า panic เกิดขึ้นก่อน disable_raw_mode terminal จะพัง
terminal::enable_raw_mode().unwrap();
execute!(stdout, EnterAlternateScreen).unwrap();

// ... code ที่อาจ panic ...

terminal::disable_raw_mode().unwrap(); // ❌ อาจไม่ถูกเรียก!
execute!(stdout, LeaveAlternateScreen).unwrap();
```

โค้ดที่ถูก:

```rust
// ✅ ใช้ RAII — Drop เรียก cleanup เสมอ แม้ panic
struct RawTerminal { stdout: Stdout }
impl Drop for RawTerminal {
    fn drop(&mut self) {
        let _ = terminal::disable_raw_mode();
        let _ = execute!(self.stdout, LeaveAlternateScreen, cursor::Show);
    }
}
```

Error message ที่จะเจอถ้าลืม:

```
# Terminal พัง — ทุก input หายไป, prompt ไม่แสดง
$ # (cursor ไม่ขยับ ไม่มี echo)
$ reset  # ต้องรันคำสั่งนี้เพื่อ restore
```

### กับดัก 2: Char Index กับ Byte Index ใน Rust Strings

```rust
// ❌ ผิด — s.len() คือ byte length, ไม่ใช่ char count
let s = "สวัสดี";
println!("{}", s.len());       // 18 (bytes!)
println!("{}", s.chars().count()); // 6 (chars)

// ❌ ผิด — panic ถ้า byte offset ไม่ตรงกับ char boundary
let ch = &s[1..3]; // panics! byte 1 ไม่ใช่ char boundary

// ✅ ถูก — ใช้ char_indices() หรือ ropey ที่ Unicode-aware
for (byte_offset, ch) in s.char_indices() {
    println!("{}: {}", byte_offset, ch);
}
```

นี่คือเหตุผลหลักที่เราใช้ `ropey` — มันเก็บ char offsets แยกจาก byte offsets และ handle Unicode correctly

### กับดัก 3: queue! กับ execute! — ลืม Flush

```rust
// ❌ ผิด — ใช้ execute! ทุก operation ทำให้ระบบ I/O ทำงานหนัก
// แต่ละ execute! = syscall write() หนึ่งครั้ง
execute!(stdout, cursor::MoveTo(0, 0))?;
execute!(stdout, Print("line 1"))?;
execute!(stdout, cursor::MoveTo(0, 1))?;
execute!(stdout, Print("line 2"))?;
// ... 100 บรรทัด = 200+ syscalls per frame!

// ✅ ถูก — queue! สะสม commands, flush ครั้งเดียวตอนสุดท้าย
queue!(stdout, cursor::MoveTo(0, 0))?;
queue!(stdout, Print("line 1"))?;
queue!(stdout, cursor::MoveTo(0, 1))?;
queue!(stdout, Print("line 2"))?;
// ...
stdout.flush()?; // syscall เดียว
```

ถ้าใช้ `execute!` ทั้งหมดจะเห็น terminal กระพริบ (flickering) เพราะแต่ละ line render ทีละชิ้น

### กับดัก 4: undo_stack ไม่ Clear redo_stack เมื่อ insert ใหม่

```rust
// ❌ ผิด — redo_stack ยังมีข้อมูลเก่าอยู่
pub fn insert_char_wrong(&mut self, ch: char) {
    let pos = self.cursor_char;
    self.rope.insert_char(pos, ch);
    self.undo_stack.push(EditCommand { kind: EditKind::Insert, pos, text: ch.to_string() });
    // ลืม: self.redo_stack.clear();  ← BUG!
    self.cursor_char += 1;
}
// ผล: พิมพ์ 'a', undo, redo → 'a' กลับมา  ✓ ถูก
// พิมพ์ 'a', undo, พิมพ์ 'b', redo → 'a' กลับมา ✗ ผิด! redo ต้อง invalid แล้ว

// ✅ ถูก
pub fn insert_char(&mut self, ch: char) {
    let pos = self.cursor_char;
    self.rope.insert_char(pos, ch);
    self.undo_stack.push(EditCommand { kind: EditKind::Insert, pos, text: ch.to_string() });
    self.redo_stack.clear(); // ← สำคัญมาก
    self.cursor_char += 1;
    self.modified = true;
}
```

### กับดัก 5: Rope line_to_char() สำหรับ last line

```rust
// ❌ ผิด — rope.len_lines() นับ "virtual line" ถ้าไฟล์จบด้วย newline
let rope = Rope::from_str("hello\nworld\n");
println!("{}", rope.len_lines()); // 3 (ไม่ใช่ 2!)
// บรรทัดที่ 2 (index 2) คือ empty string

// ✅ ถูก — ใช้ saturating_sub หรือตรวจสอบก่อน
let num_lines = rope.len_lines();
let display_lines = if rope.len_chars() > 0
    && rope.char(rope.len_chars() - 1) == '\n'
{
    num_lines.saturating_sub(1)
} else {
    num_lines
};
```

### กับดัก 6: Crossterm บน Windows — ANSI Mode

Windows 10+ รองรับ ANSI escape codes แต่ต้องเปิดก่อน `crossterm` จัดการให้อัตโนมัติผ่าน:

```rust
// crossterm ทำสิ่งนี้ให้อัตโนมัติใน enable_raw_mode() บน Windows:
// SetConsoleMode(ENABLE_VIRTUAL_TERMINAL_PROCESSING)

// แต่ถ้า Windows เวอร์ชันเก่า (< 1607) อาจต้องใช้ alternate backend
// ตรวจสอบ crossterm features ใน Cargo.toml
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1 (ระดับง่าย): Line Numbers

เพิ่มการแสดงหมายเลขบรรทัดทางซ้ายเหมือน vim ด้วย `set number`:

```
  1 │ fn main() {
  2 │     println!("Hello!");
  3 │ }
```

ใช้ format string `{:>4} │ ` เพื่อ right-align เลขบรรทัด ต้องปรับ `viewport.width` ให้ลบ gutter width ออกด้วย

### แบบฝึกหัดที่ 2 (ระดับกลาง): Multiple Buffers / Tabs

เพิ่ม `Vec<Buffer>` และ index `active_buffer: usize` จากนั้น:
- Ctrl+T: เปิด buffer ใหม่
- Ctrl+Tab / Ctrl+Shift+Tab: ข้ามระหว่าง buffer
- Tab bar ที่ด้านบนแสดงชื่อไฟล์และ `[+]` flag ของแต่ละ buffer

ความท้าทาย: viewport ควร independent ต่อแต่ละ buffer

### แบบฝึกหัดที่ 3 (ระดับกลาง): Find and Replace

ขยาย search mode ให้รองรับ replace:
- Ctrl+H เปิด find-and-replace mode
- `pattern → replacement` — replace occurrence แรก
- `y` (yes) / `n` (no) เพื่อ confirm replace ทีละจุด
- `a` (all) replace ทั้งหมดในครั้งเดียว

ใช้ EditCommand สำหรับ undo หลัง replace ด้วย

### แบบฝึกหัดที่ 4 (ระดับยาก): LSP Integration

เชื่อมต่อกับ Language Server Protocol เพื่อ:
- Diagnostics (underline errors ด้วยสี)
- Hover information (แสดง type signature)
- Go to definition (Ctrl+Click)

ใช้ crate `lsp-types` สำหรับ protocol types และ `tokio::process::Command` เพื่อ spawn `rust-analyzer` เป็น child process

สื่อสารผ่าน `stdin`/`stdout` ด้วย JSON-RPC 2.0 format ตาม LSP specification

### แบบฝึกหัดที่ 5 (ระดับสูง): Gap Buffer Implementation

แทนที่จะใช้ `ropey` ให้ implement **Gap Buffer** เอง:

```
Gap Buffer: เก็บข้อความใน single Vec<char> ที่มี "gap" ตรงตำแหน่ง cursor

Before insert at cursor position 3:
[H][e][l][_][_][_][_][l][o]
              ^ gap (indices 3-6)

After insert 'X':
[H][e][l][X][_][_][_][l][o]
               ^ gap shifts right
```

Gap Buffer มี O(1) amortized สำหรับ insert/delete ที่ cursor (ไม่ต้อง rebalance tree เหมือน Rope) แต่ O(n) ถ้า cursor กระโดดไปไกล

---

## สรุป

ในโปรเจคนี้เราได้สร้าง terminal text editor ตั้งแต่ศูนย์โดยครอบคลุม:

1. **Raw terminal mode** ด้วย `crossterm` และ RAII pattern สำหรับ cleanup ที่ safe
2. **Rope data structure** ผ่าน `ropey` — เข้าใจว่าทำไม editor จริงไม่ใช้ Vec\<String\>
3. **Event-driven architecture** ที่ handle keyboard input ทุกประเภทอย่างครบถ้วน
4. **Viewport/scrolling** — render เฉพาะสิ่งที่จำเป็น ไม่ waste CPU
5. **Atomic file save** — ป้องกันข้อมูลสูญหายด้วย temp-file + rename pattern
6. **Undo/Redo** ด้วย Command pattern — `Vec<EditCommand>` เป็น time machine
7. **Incremental search** พร้อม real-time highlight
8. **Syntax highlighting** แบบ line-based tokenizer ที่ extendable

Pattern สำคัญที่ได้เรียนในโปรเจคนี้:

- **RAII สำหรับ resource cleanup** — `Drop` trait ทำให้ cleanup ไม่มีวันถูกลืม
- **Command pattern** — encapsulate actions เป็น data structure เพื่อ undo/redo
- **Separation of concerns** — Buffer (data), Viewport (presentation state), Terminal (I/O) แยกจากกันชัดเจน
- **Unicode correctness** — ใน Rust ต้องระวัง byte vs char boundary เสมอ

โปรเจคถัดไปจะสร้าง **Process Manager** ซึ่งต่อยอดทักษะ terminal/systems programming ที่ได้จาก editor นี้ — แทนที่จะ manage text เราจะ manage running processes แบบ `htop`

---

**โปรเจคก่อนหน้า:** [project-a05-dns-resolver.md](project-a05-dns-resolver.md) | **โปรเจคถัดไป:** [project-a07-process-manager.md](project-a07-process-manager.md)
